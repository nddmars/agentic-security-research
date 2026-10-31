# 01: CxSAST Ingestion and Normalization

Author: Surya

Read `00-MAIN-DESIGN.md` first (shared contracts in section 7).

v2 changes: stores `finding_kind` (`path_based` or `pattern_only`) and handles results with no source-to-sink path; stores scan start time (needed by 02); no tree-sitter dependency.

## 1. Goal
Continuously discover completed CxSAST scans, pull their results with the full attack-vector path (where one exists) and existing triage state, normalize them into the internal finding schema, and persist them. Emit events that downstream stages and the dashboard consume.

## 2. Inputs
- CxSAST on-prem REST API (and OData where useful). Authentication: OAuth2 token endpoint with a service account; scope `sast_api` (verify scopes for your version).
- Configuration: base URL, client credentials (from the secret manager), poll interval, project include/exclude lists, naming regex for application ID parsing, list of **pattern-query names** (queries known to be pattern or regex based and to return no path; see 4.4).
- Checkmarx **XML report export** (Reports UI) used as the test/golden input: state, Remark and Path per result. The parser is built against the real sample (see 11 section 2.1).

## 3. Outputs
- Rows in `scans`, `findings`, `finding_path_nodes`, `ingestion_events` tables.
- A `FindingIngested` event per finding, and a `ScanIngested` event per scan.

## 4. Functional requirements

### 4.1 Scan discovery
- Poll for scans with status Finished (and not previously ingested). Use a high-water-mark per project (last ingested scan finish time) plus idempotent upserts keyed by `scan_id`.
- Support backfill mode: ingest scans in a date range, with rate limiting.
- Handle scans that finish while polling (re-check status; ignore in-progress or failed scans, but record failed scans in `scans` with status).
- Respect a configurable max scans per cycle to avoid overloading the CxSAST server. Use exponential backoff on 5xx and 429.

### 4.2 Project metadata
- For each scan, capture `project_id`, `project_name`, team, preset, scan type (full or incremental), **scan start time**, scan finish time, languages, lines of code if available, and the **scan comment**. Scan start time is required by 02 (source is taken at scan start). If the API exposes the source-pull time or the project's source origin settings (repository URL and branch), store them too (`source_pull_at`, `cx_source_origin`).
- Parse `application_id` and `component` from `project_name` using the configured regex. Never fail on mismatch (use `UNKNOWN`).
- Store the raw scan comment verbatim in `scans.scan_comment_raw`. **Read only. No code in this system may write to the scan comment.**

### 4.3 Results retrieval
- Fetch all results for the scan, including: query name, CWE, severity, state, status (New, Recurrent, Fixed), similarity ID, path ID, first scan ID where found (if available), and the full path nodes (file, line, column, length, name, snippet).
- Fetch the **existing per-result comment/notes** and existing state/assignment for each result. Store as `existing_notes`, `cx_state`, `cx_assigned_to`.
- Retrieve result states list once per run (`GET /sast/resultStates` or equivalent) and cache it; store the mapping from state name to ID.
- Filtering: ingest all results but mark scope. Set `in_scope = true` only for languages in the supported set and for non-excluded severities (config). Out-of-scope results are stored (so the dashboard can count them) but not queued.
- Skip results already in a terminal human-decided state (for example already Not Exploitable or Confirmed by a human) by default. Config flag `triage_already_decided` default false. Always still ingest them for completeness.

### 4.4 Normalization
- Map Checkmarx severity names to the internal enum.
- Determine `language` from the result's language field if present, else from file extension of the sink file. Mapping: `.java`->java, `.cs`->csharp, `.py`->python, `.js/.jsx/.mjs/.cjs`->javascript, `.ts/.tsx`->typescript, `.go`->go.
- **Determine `finding_kind`:** `pattern_only` if the query name is in the configured pattern-query list, or the result has no path nodes or a single node and the query is not known to be a taint query. Otherwise `path_based`. Log the rule used. Unknown queries default to `path_based` and are reported in a weekly "unclassified queries" list for analysts.
- Assign `role` to path nodes: for `path_based`, first node = `source`, last node = `sink`, middle = `propagator` (refined in 03 once code is parsed). For `pattern_only`, every node has `role = match`.
- Normalize file paths: strip leading slashes and backslashes, convert to POSIX separators, keep the original in `file_raw`.
- Compute `path_hash` = SHA-256 of the ordered list of (file, line, name) tuples. Used by the fingerprint in 06.
- Never log or store a matched secret value from a pattern-only result beyond what Checkmarx itself holds: snippets of `pattern_only` findings are masked at ingestion when the query is in a secrets-like family (config list), keeping only a value fingerprint (hash) for dedupe (see 15).

### 4.5 Idempotency and recurrence
- Upsert by (`scan_id`, `path_id`). Re-ingesting the same scan must not duplicate rows.
- For Recurrent findings, link to the prior finding via `similarity_id` and project. Do not delete or merge; downstream fingerprint logic handles reuse.

### 4.6 Rate limiting and resilience
- Concurrency limit for CxSAST calls (config, default 4).
- Per-call timeout and retry with jitter. Persist partial progress so a restart resumes.
- Large scans (tens of thousands of results): stream or page, do not hold the full result set in memory.

## 5. Data model (Postgres)

```sql
CREATE TABLE scans (
  scan_id TEXT PRIMARY KEY,
  project_id TEXT NOT NULL,
  project_name TEXT NOT NULL,
  application_id TEXT NOT NULL,
  component TEXT,
  scan_comment_raw TEXT,
  scan_type TEXT,                 -- full | incremental | unknown
  scan_started_at TIMESTAMPTZ,    -- source anchor time (02)
  scan_finished_at TIMESTAMPTZ,
  source_pull_at TIMESTAMPTZ,     -- if exposed by the API
  cx_source_origin JSONB,         -- repo URL/branch from project settings, if exposed
  loc BIGINT,
  status TEXT NOT NULL,           -- INGESTING | INGESTED | FAILED
  ingested_at TIMESTAMPTZ,
  total_results INT,
  in_scope_results INT
);

CREATE TABLE findings (
  finding_id UUID PRIMARY KEY,
  finding_kind TEXT NOT NULL,     -- path_based | pattern_only
  origin TEXT NOT NULL DEFAULT 'checkmarx',   -- always 'checkmarx' in v1
  scan_id TEXT NOT NULL REFERENCES scans(scan_id),
  project_id TEXT NOT NULL,
  application_id TEXT NOT NULL,
  cx_path_id TEXT NOT NULL,
  similarity_id TEXT,
  query_name TEXT NOT NULL,
  cwe TEXT,
  language TEXT,
  severity TEXT NOT NULL,
  cx_state TEXT,
  cx_status TEXT,
  cx_assigned_to TEXT,
  existing_notes TEXT,
  path_hash TEXT NOT NULL,
  path_incomplete BOOLEAN NOT NULL DEFAULT false,
  in_scope BOOLEAN NOT NULL,
  lifecycle_state TEXT NOT NULL,
  first_seen_scan_id TEXT,
  ingested_at TIMESTAMPTZ NOT NULL,
  UNIQUE (scan_id, cx_path_id)
);

CREATE TABLE finding_path_nodes (
  finding_id UUID REFERENCES findings(finding_id),
  node_index INT,
  file TEXT, file_raw TEXT,
  line INT, col INT, length INT,
  name TEXT, snippet TEXT, role TEXT,    -- source | propagator | sanitizer_candidate | sink | match
  PRIMARY KEY (finding_id, node_index)
);

CREATE INDEX ON findings (project_id, scan_id);
CREATE INDEX ON findings (application_id);
CREATE INDEX ON findings (lifecycle_state);
CREATE INDEX ON findings (similarity_id, project_id);
CREATE INDEX ON findings (finding_kind, lifecycle_state);
```

## 6. API of this component
Internal Python module and a small admin CLI:
- `ingest scan <scan_id>`
- `ingest backfill --from <date> --to <date> [--project <id>]`
- `ingest status`

## 7. Events emitted
`ScanDiscovered`, `ScanIngested(scan_id, total, in_scope)`, `FindingIngested(finding_id)`, `IngestionFailed(scan_id, reason)`.

## 8. Edge cases
- Scan with zero results: record scan, emit `ScanIngested` with 0.
- `path_based` result with missing path nodes or a single-node path: ingest, mark `path_incomplete = true`, route to `NEEDS_HUMAN_NO_ANALYSIS` later. (A `pattern_only` result with one node is normal and is **not** incomplete.)
- Pattern-only result with only a file and no line: ingest with `line = null`; 15 section 3.1 locates the match.
- File path in a result refers to a file outside the source root or using Windows drive paths: normalize and keep raw.
- Same query and path in two projects: treated as separate findings (CxSAST predicates are project-scoped); dedupe of agent work happens via fingerprint in 06.
- Results whose state changed between ingestion and writeback: handled in 09 by re-reading before writing.
- Clock skew and timezone: store all times in UTC.

## 9. Acceptance criteria
- Re-running ingestion for the same scan produces zero new rows.
- A scan with 50,000 results ingests without exceeding a configured memory ceiling (for example 1 GB).
- All `path_based` findings have a non-empty `path` or `path_incomplete = true`; all `pattern_only` findings have at least one `match` node or a file reference.
- Scan comment is stored verbatim and no code path writes to it (enforced by a test that greps the codebase for write calls to the scan comment endpoint, and by using a client wrapper that exposes no such method).
- `scan_started_at` is stored for every scan where the API provides it, and is distinct from `scan_finished_at`.
- Unit tests for: severity mapping, language detection, `finding_kind` classification (pattern query list, single-node results, unknown query), path normalization, naming regex parsing (match, no match, multiple components), idempotent upsert, secret-like pattern snippets masked at ingestion.
- Integration test against a CxSAST mock server covering paging, 429 backoff, and partial failure resume.
