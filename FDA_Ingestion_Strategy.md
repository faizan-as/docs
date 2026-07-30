# FDA Guidance Ingestion Strategy

## AI-Powered FDA Regulatory Intelligence Platform

> **Data source:** [Search for FDA Guidance Documents](https://www.fda.gov/regulatory-information/search-fda-guidance-documents) — the search UI is backed by a **JSON feed** (Drupal/DataTables-style) that returns one record per guidance document with structured metadata, filters, and links.
>
> **Core recommendation:** Adopt a **two-tier, catalog-first ingestion architecture**. Treat the JSON feed as the authoritative **catalog / registry**, and treat PDF/landing-page fetching as a separate, incremental **content pipeline** driven by that registry.

---

## 1. What the JSON Feed Actually Is (and Isn't)

Analysis of your uploaded sample (`search-for-guidance-sample.json`) shows each record is a **metadata row**, not document content. Fields are HTML-escaped with embedded anchors and HTML entities that must be parsed.

### 1.1 Field Map (raw → normalized)

| Raw JSON field | Normalized field | Notes / transform |
|---|---|---|
| `title` | `title`, `landing_url`, `slug` | HTML anchor → extract text + relative href; slug = last path segment |
| `field_associated_media_2` | `pdf_url`, `has_pdf` | HTML anchor → PDF `/media/<id>/download`; **often empty** |
| `field_issue_datetime` | `issue_date` | `MM/DD/YYYY` → ISO date |
| `field_final_guidance_1` | `status` | `Draft` / `Final` — **critical PRD field** |
| `field_center` / `field_issuing_office_taxonomy` | `center` | e.g. "Human Foods Program", CDER, CBER, CDRH |
| `field_communication_type` | `comm_type` | e.g. "Guidance Document", "Small Entity Compliance Guide" |
| `topics-product` / `term_node_tid` | `topics[]` | Comma-separated → array |
| `field_regulated_product_field` | `regulated_product` | HTML entity decode (`&amp;` → `&`) |
| `field_docket_number` | `docket_id`, `docket_url` | Anchor → `FDA-YYYY-D-NNNN` + regulations.gov URL |
| `field_comment_close_date` | `comment_close` | Draft comment window |
| `open-comment` | `open_comment` | " No " / "Yes" → boolean |
| `changed` | `last_changed` | `<time datetime="…">` → ISO timestamp — **drives delta detection** |

### 1.2 Three Findings That Shape the Strategy

1. **PDF links are frequently missing from the feed.** ~75% of the sample had no direct PDF; the PDF URL must be **resolved from the landing page HTML** as a fallback. Never assume `field_associated_media_2` is populated.
2. **`changed` timestamp is present on every row.** This is your free, reliable **change-detection signal** — no need to hash every PDF just to know *whether* something changed.
3. **`docket_id` links to regulations.gov.** This is a stable cross-reference key, valuable now for dedup/versioning and later for the V1 `CITES` / `SUPERSEDES` graph relationships.

---

## 2. Recommended Architecture: Two-Tier, Catalog-First

```
┌───────────────────────── TIER 1: CATALOG SYNC (cheap, complete, daily) ────────────────────────┐
│  FDA JSON feed  →  parse/normalize  →  Document Registry (Postgres)                              │
│  • full metadata for all ~2,788 docs   • status, dates, docket, center, topics                  │
│  • no downloads   • diff against last sync using `changed` + slug                               │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
                                              │  emits change events: NEW / UPDATED / WITHDRAWN
                                              ▼
┌────────────────────── TIER 2: CONTENT INGESTION (expensive, incremental, queued) ───────────────┐
│  resolve PDF URL (feed → landing page fallback)  →  download  →  SHA-256 hash                    │
│  →  PyMuPDF + Unstructured parse  →  parent-child chunk  →  BGE-M3 embed                          │
│  →  index: Qdrant (dense) + OpenSearch (BM25)  →  snapshot to object store                       │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

**Why decouple the two tiers?**

- **Completeness without cost.** Tier 1 gives you a 100% metadata catalog (filters, status, counts) immediately — powering the search filters in the PRD — without downloading anything.
- **Incremental, prioritized content.** Tier 2 only fetches/parses documents that are *new or changed*, and can be prioritized (e.g., Final before Draft, most-requested centers first).
- **Idempotency & resumability.** If content ingestion fails on doc #900, the registry is intact and the job resumes; nothing is re-scraped.
- **Clean audit trail.** The registry records *what FDA published and when*; the content pipeline records *what we parsed and indexed* — two separate, auditable ledgers, which matters for an audit-grade product.

---

## 3. Tier 1 — Catalog Sync (Metadata Registry)

### 3.1 Acquiring the Feed

The guidance search UI calls a JSON endpoint (visible in your browser DevTools → Network tab, which is how you found it). Recommended approach:

- **Capture the endpoint + query params** the UI uses, including the page-size / offset (or `page`) parameters.
- **Paginate to completion** — request pages until an empty/short page is returned; aggregate all rows.
- **Be a good citizen:** identify a realistic `User-Agent`, add polite rate limiting (e.g., 1–2 req/sec), retry with backoff, and cache the raw feed response per run for reproducibility.
- **Snapshot the raw feed** (date-stamped) to object storage before parsing — this is your immutable evidence of "what FDA showed on day X."

> Prefer the JSON feed over HTML scraping of the results table — it is more stable, structured, and complete. Keep HTML parsing only as the *landing-page* fallback in Tier 2.

### 3.2 Document Identity & Change Detection

| Concern | Recommendation |
|---|---|
| **Stable primary key** | Use the **landing-page slug** (e.g. `guidance-industry-voluntary-labeling…`) as the natural key. It is stable across re-issues. Store `docket_id` as a secondary index. |
| **New vs. updated** | Compare incoming `(slug, last_changed, status)` against the registry. New slug → **NEW**; same slug + newer `changed` or status flip → **UPDATED**. |
| **Withdrawn** | A slug present in the previous sync but **absent** from the current feed → mark **WITHDRAWN** (soft-delete; keep for audit + freshness alerts per PRD). |
| **Draft → Final transition** | Detect `status` change on the same slug/docket → record a lifecycle event (feeds the "draft vs final awareness" differentiator). |

### 3.3 Registry Schema (PostgreSQL)

```sql
CREATE TABLE guidance_registry (
    slug              TEXT PRIMARY KEY,
    title             TEXT NOT NULL,
    landing_url       TEXT NOT NULL,
    pdf_url           TEXT,                 -- may be NULL; resolved in Tier 2
    status            TEXT,                 -- Draft | Final
    center            TEXT,
    comm_type         TEXT,
    topics            TEXT[],
    regulated_product TEXT,
    docket_id         TEXT,
    docket_url        TEXT,
    issue_date        DATE,
    comment_close     DATE,
    open_comment      BOOLEAN,
    fda_last_changed  TIMESTAMPTZ,          -- from feed `changed`
    first_seen_at     TIMESTAMPTZ DEFAULT now(),
    last_synced_at    TIMESTAMPTZ,
    lifecycle_state   TEXT DEFAULT 'active' -- active | withdrawn
);
CREATE INDEX ix_reg_status  ON guidance_registry(status);
CREATE INDEX ix_reg_center  ON guidance_registry(center);
CREATE INDEX ix_reg_docket  ON guidance_registry(docket_id);
CREATE INDEX ix_reg_changed ON guidance_registry(fda_last_changed);
```

---

## 4. Tier 2 — Content Ingestion Pipeline

Runs only for documents flagged NEW or UPDATED by Tier 1.

### 4.1 Step-by-Step

| # | Step | Detail |
|---|---|---|
| 1 | **Resolve PDF URL** | If `pdf_url` present in registry, use it. Else fetch the **landing page** and extract the `/media/<id>/download` link (BeautifulSoup). If no PDF exists, ingest the landing-page HTML body as the content source. |
| 2 | **Download** | Fetch the PDF/HTML; store raw file to S3/MinIO under `raw/{slug}/{sha256}.pdf`. |
| 3 | **Content hash** | Compute **SHA-256** of the downloaded bytes. If it matches the last stored hash for that slug, skip re-parsing (metadata may have changed without content changing). |
| 4 | **Parse** | **PyMuPDF** for text + layout + page numbers; **Unstructured** for tables and section structure. Preserve page and section coordinates for citation. |
| 5 | **Chunk** | **Parent-child chunking**: parent = section/heading block, child = paragraph. Carry section title, page number, and char offsets on every chunk. |
| 6 | **Embed** | **BGE-M3** dense embeddings on child chunks. |
| 7 | **Index** | Upsert child vectors + metadata payload to **Qdrant**; index full text + metadata to **OpenSearch** (BM25). Use deterministic point IDs (`{slug}:{version_hash}:{chunk_no}`) for idempotent upserts. |
| 8 | **Snapshot** | Write a dated snapshot record linking `slug → version_hash → parsed artifacts` for version history and diffing. |

### 4.2 Chunk Payload (Qdrant / OpenSearch metadata)

Every chunk carries the metadata needed to build an **evidence card** at answer time:

```json
{
  "slug": "guidance-industry-voluntary-labeling...",
  "doc_title": "Guidance for Industry: Voluntary Labeling...",
  "version_hash": "sha256:ab12...",
  "status": "Final",
  "center": "Human Foods Program",
  "comm_type": "Guidance Document",
  "topics": ["Bioengineering / GMOs", "Labeling"],
  "docket_id": "FDA-2000-D-0075",
  "issue_date": "2019-03-11",
  "source_url": "https://www.fda.gov/media/120958/download",
  "section_title": "III. Labeling Recommendations",
  "page": 7,
  "char_start": 1423,
  "char_end": 2011,
  "parent_id": "…"
}
```

> This payload is what closes the loop on your **one non-negotiable**: every generated claim can be bound to `document + section + page + exact passage + version + source URL`.

### 4.3 Versioning & Draft/Final Lifecycle

- **Content versions** are keyed by SHA-256; each new hash for a slug becomes a new version, old vectors are retained (or soft-archived) for diffing.
- **Status transitions** (Draft→Final, or withdrawal) are recorded as lifecycle events, powering PRD features: *update monitoring*, *draft vs final diff (Phase 2)*, and V1 `SUPERSEDES` graph edges.

---

## 5. Orchestration & Scheduling

| Aspect | MVP recommendation |
|---|---|
| **Scheduler** | **APScheduler** for the daily catalog sync (simple, in-process). Move to **Prefect** when you need retries, observability, and backfills as a DAG. |
| **Cadence** | Tier 1 catalog sync **daily**; Tier 2 content ingestion triggered by change events, with an initial **one-time full backfill** of all ~2,788 docs. |
| **Concurrency** | Bounded worker pool for Tier 2 (e.g., 4–8 concurrent fetch/parse) with polite rate limiting toward fda.gov. |
| **Idempotency** | Deterministic IDs + SHA-256 dedup make every run safe to re-run. |
| **Observability** | Log per-document outcomes (fetched / parsed / indexed / skipped / failed) and emit counts to Langfuse/metrics; a nightly reconciliation confirms registry count == indexed count. |

This maps directly onto your existing repo layout (`src/ingestion/`): `fda_scraper.py` (Tier 1 feed + Tier 2 fetch), `pdf_parser.py`, `metadata_extractor.py`, `chunking.py`, `embedder.py`, `version_tracker.py`, `pipeline.py`.

---

## 6. Edge Cases to Handle Explicitly

| Edge case | Handling |
|---|---|
| **No PDF in feed** (~75% of sample) | Resolve from landing page; if still none, ingest landing-page HTML body. |
| **HTML-escaped fields** (`\u003C`, `&amp;`) | Unicode-unescape + `html.unescape` + strip tags during normalization (validated on your sample). |
| **Multi-value topics** | Split on comma into arrays; normalize whitespace. |
| **Scanned/image PDFs** | Detect low extracted-text ratio → route to **OCR** (future; PRD OCR is out of MVP scope but flag for later). |
| **Very large PDFs / tables** | Unstructured for table extraction; keep tables as distinct chunk type. |
| **Withdrawn documents** | Soft-delete in registry; suppress from retrieval but keep for audit + freshness alerts. |
| **Duplicate docket, different slug** | Group by `docket_id` for supersession detection (V1 graph). |
| **fda.gov rate limits / errors** | Exponential backoff, resumable queue, dated raw-feed snapshot for replay. |

---

## 7. Recommended Ingestion Sequence (Build Order)

1. **Feed parser + normalizer** — turn the JSON into clean registry rows (working prototype already validated on your sample).
2. **Registry + change detection** — Postgres table + NEW/UPDATED/WITHDRAWN diff on `changed` + slug.
3. **One-time full metadata backfill** — populate all ~2,788 rows (no downloads).
4. **PDF resolver + downloader** — feed link, else landing-page fallback, with SHA-256.
5. **Parse → parent-child chunk → BGE-M3 embed.**
6. **Qdrant + OpenSearch indexing** with the evidence-card payload.
7. **Daily scheduler + snapshotting + reconciliation checks.**

---

## 8. Bottom Line

**Use the JSON feed as a catalog, not as content.** Build a **two-tier pipeline**: a cheap daily **catalog sync** that maintains a complete, change-tracked metadata registry of all ~2,788 documents, and an incremental **content pipeline** that fetches, parses, chunks, embeds, and indexes only what changed — always resolving the PDF from the landing page when the feed omits it.

This gives you:

- **Complete, filterable metadata from day one** (powers PRD search filters and Draft/Final awareness).
- **Free, reliable change detection** via the `changed` timestamp (powers update monitoring).
- **Idempotent, resumable, auditable ingestion** with SHA-256 versioning (powers the citation-first moat and version history).

It slots cleanly into your existing `src/ingestion/` module and your LangGraph MVP without adding new infrastructure.
