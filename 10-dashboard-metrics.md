# 10: Dashboard and Metrics

Author: Surya

Read `00-MAIN-DESIGN.md` first. Depends on 01, 06, 08, 09 (data sources).

v2 changes: adds pattern-finding views (15), source-match and context-level breakdowns, no-CPG tracking, evidence strength, and OpenShift Route/SSO notes. The primary requirement is unchanged.

## 1. Goal
A web dashboard that shows, **per scan ID and per project ID**, how many findings have been triaged and how many are pending, plus the breakdowns needed to decide when auto-close is safe. Tool-agnostic: a web UI backed by a read-only API over Postgres. (If the organization prefers Grafana, Power BI, or Splunk later, the same API and SQL views can feed them.)

## 2. Primary requirement (must have)
For every **scan ID** and **project ID**:
- Total findings in the scan.
- In-scope findings (supported languages and filters).
- **Triaged** count (findings with a terminal outcome).
- **Pending** count (in scope minus terminal), and where they are in the pipeline.
- Last updated time.

Definitions (shared with 06):
- `triaged` = terminal states: `WRITTEN`, `REUSED_VERDICT` (written), `NEEDS_HUMAN_NO_ANALYSIS` (written), `ANALYSIS_FAILED`, `SKIPPED_DUPLICATE`.
- `pending` = in-scope findings not in a terminal state.
- Show `ANALYSIS_FAILED` separately so "triaged" is not mistaken for "successfully analyzed".
- Counts are shown for all findings and split by `finding_kind` (`path_based`, `pattern_only`).

## 3. Additional views and metrics

### 3.1 Per scan and per project detail
- Verdict breakdown: FP, TP, NEEDS_HUMAN (proposed), no-analysis, failed.
- Pipeline stage funnel for pending findings (source pending, indexing, triaging, validating, writeback pending).
- Failure reasons breakdown (`no_source_location`, `source_mismatch`, `origin_conflict`, `index_failed_timeout`, `writeback conflict`, `masking_violation`, and so on).
- Flags: `high_risk_fp`, `approximate_revision`, `injection_suspected`, `second_opinion_disagreement`, `table_disagreement`.
- Reuse: count reused from dedupe versus freshly triaged.
- Revision confidence distribution (EXACT, HIGH, APPROXIMATE, UNKNOWN), `source_match_score` distribution, and context level mix (L0, L1, L2).
- Drill-down to a finding list with filters; each finding shows the verdict, evidence strength, reason, evidence, validator downgrades, and a deep link to the Checkmarx result.

### 3.2 Rollups
- By application ID (aggregating its projects), by team if available, by language, by vulnerability family and query name, by severity.
- Time series: findings ingested, triaged, pending per day; backlog age; time from ingestion to writeback (p50, p90, p99).

### 3.3 Reviewer view
- Per reviewer: assigned items (from assignment or the comment fallback), pending count, aging buckets, accepted versus overturned.
- Overturn rate by segment (family, language, tier, severity, precision level): **the key metric for auto-close decisions**.

### 3.4 Quality metrics (feed the evaluation harness)
- False-negative proxy: confirmed TPs among findings the agent proposed FP (overturned FP proposals), by segment, with counts and confidence intervals.
- Agreement rate with reviewers.
- Calibration plot data: confidence bucket versus actual acceptance rate.
- Second-opinion disagreement rate, with the most frequent differing facts.
- Decision-table disagreement rate (agent versus derived verdict) by family.
- `would_have_auto_closed` counts: findings that satisfied `auto_close_eligible` and their eventual reviewer outcomes (the metric that justifies enabling auto-close).

### 3.5 Pattern findings (15)
- Pattern findings by family, verdict, file class and usage-context coverage.
- Masking check failures (should be zero); values are never displayed.

### 3.6 Operations
- Queue depths and throughput per stage, worker health, index build times and failures by size tier, Joern memory failures, **no-CPG rate by language, tier and reason** (input to the tree-sitter decision gate in 14).
- LLM usage: tokens, cost per finding by tier and provider, prompt cache hit ratio, single-pass share, escalation share, rate-limit pauses.
- CxSAST API health: latency, error rate, writeback success rate, dry-run versus live counts.
- Ingestion lag: time between scan finish and ingestion.
- Alerts surfaced in the UI (kill switch state, quota exhaustion, failure spikes).

## 4. Data architecture
- Source of truth: Postgres tables from 01, 06, 08, 09, 15 (`scans`, `findings`, `finding_events`, `verdicts`, `validated_verdicts`, `writebacks`, `reviewer_outcomes`, `finding_context`).
- **Reporting layer:** materialized views (refreshed incrementally, for example every 1 to 5 minutes) so the dashboard stays fast at about 1 million findings per quarter. Suggested views:
  - `mv_scan_progress` (scan_id, project_id, application_id, total, in_scope, triaged, pending, failed, verdict counts, by finding_kind, last_event_at)
  - `mv_project_rollup`, `mv_application_rollup`
  - `mv_daily_throughput`
  - `mv_reviewer_queue`
  - `mv_segment_quality` (family x language x tier x severity x precision: counts, overturned, accepted)
  - `mv_failure_reasons`
  - `mv_pattern_summary`
  - `mv_context_levels` (context level, no-CPG reasons, by language and tier)
- Example core query:
```sql
SELECT f.scan_id, f.project_id, f.application_id, f.finding_kind,
  COUNT(*) FILTER (WHERE f.in_scope) AS in_scope,
  COUNT(*) FILTER (WHERE f.in_scope AND f.lifecycle_state IN
    ('WRITTEN','REUSED_VERDICT','NEEDS_HUMAN_NO_ANALYSIS','ANALYSIS_FAILED','SKIPPED_DUPLICATE')) AS triaged,
  COUNT(*) FILTER (WHERE f.in_scope AND f.lifecycle_state NOT IN
    ('WRITTEN','REUSED_VERDICT','NEEDS_HUMAN_NO_ANALYSIS','ANALYSIS_FAILED','SKIPPED_DUPLICATE')) AS pending
FROM findings f
GROUP BY f.scan_id, f.project_id, f.application_id, f.finding_kind;
```
- Partition large event tables by month; index by (scan_id, project_id, occurred_at).
- Retention: events 18 months (config); prompt and response bodies per audit policy.

## 5. Backend API (read-only)
REST JSON, authenticated; the API only reads the reporting layer (no write endpoints in v1 other than optional saved filters).
```
GET /api/scans?project_id=&application_id=&from=&to=&status=&page=
GET /api/scans/{scan_id}                 -> progress, breakdowns
GET /api/projects/{project_id}/scans     -> scan list with triaged/pending
GET /api/applications/{application_id}/summary
GET /api/findings?scan_id=&project_id=&verdict=&state=&severity=&family=&kind=&flag=&page=
GET /api/findings/{finding_id}           -> verdict, evidence, downgrades, writeback status
GET /api/reviewers/{reviewer}/queue
GET /api/quality/segments?window=30d     -> overturn and acceptance by segment
GET /api/patterns/summary?family=&origin=&window=
GET /api/ops/queues | /api/ops/llm | /api/ops/cxsast | /api/ops/context
GET /api/export/scans.csv?...            -> CSV export
```
Pagination, sorting, filter validation, and rate limits. Responses must be consistent with the definitions of triaged and pending in section 2.

## 6. UI requirements
- Home: search by scan ID, project ID, or application ID; global tiles (today's ingested, triaged, pending, failed, would-have-auto-closed); alerts banner.
- Scan list/table: scan ID, project ID, project name, application ID, finished time, in-scope, triaged, pending (with progress bar), verdict mix, failures; sortable and filterable; auto-refresh (poll every 30 to 60 seconds, or server-sent events).
- Scan detail: progress header, stage funnel, breakdown charts, failure reasons, flags, finding table with drill-down.
- Finding detail: path summary (or matched location for pattern findings), verdict, evidence strength, reason, evidence with file and line, validator downgrades, second opinion result with differing facts, revision confidence, source match, writeback status, reviewer outcome, link to Checkmarx.
- Quality page: segment table with overturn rate and confidence intervals, calibration chart, auto-close readiness indicator per segment (meets thresholds yes/no, with the thresholds from 14 shown).
- Patterns page: families, origins, review queue (section 3.5).
- Reviewer page: queue and aging.
- Ops page: queues, throughput, costs, API health, no-CPG rates, kill switch state (read-only display; the kill switch itself is operated through config or an admin CLI, not the UI, in v1).
- Accessibility basics, responsive layout, CSV export on tables, deep-linkable URLs for filters.
- Tech suggestion: React with a table/chart library and a thin API service; any equivalent stack is fine. Served from OpenShift behind a Route with SSO (13).

## 7. Security and access
- SSO via the organization's identity provider; role-based access: viewer (all), reviewer (their queue plus global), admin (ops). SSO integration with OpenShift OAuth or corporate SSO to be confirmed in phase 0.
- Row-level scoping by application where the organization requires it (config).
- No source code or secrets rendered beyond the short evidence quotes already in verdicts; **matched secret values from pattern findings are never stored in the reporting layer**, so they cannot be displayed or exported.
- Audit log of dashboard exports.
- Read-only database credentials for the API.

## 8. Acceptance criteria
- The scan table shows, for any scan ID and project ID, correct `in_scope`, `triaged`, and `pending` counts that match a direct SQL count on the findings table (automated consistency test on seeded data covering every lifecycle state and both finding kinds).
- Pending plus triaged equals in-scope for 100% of scans in tests; failed findings are visible separately.
- Materialized views refresh within the configured interval and the UI shows a "data as of" timestamp.
- Performance: scan list page loads in under 2 seconds at 1,500 projects and 5 million finding rows (synthetic data); detail pages under 3 seconds.
- Filters by application, project, scan, verdict, severity, family, kind, and flag work and are covered by API tests.
- Segment quality page reproduces overturn rates computed independently in a test notebook or SQL script.
- CSV export matches on-screen counts.
- Access control tests: viewer cannot see admin pages; row-level scoping works when enabled.
- A planted secret value in a pattern fixture appears nowhere in API responses, exports or UI.
- Dry-run versus live writebacks are clearly distinguished in all counts.
