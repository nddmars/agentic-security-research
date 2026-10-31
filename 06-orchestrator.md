# 06: Orchestrator, Dedupe, Size Tiers, and Budgets

Author: Surya

Read `00-MAIN-DESIGN.md` first. Depends on 01, 02, 03. Feeds 07.

v2 changes: the fingerprint includes the Evidence Pack hash; `pattern_only` findings route to the pattern pipeline (15); per-finding agent budgets reflect single-pass plus bounded gap-fill; size-tier fallback is the L1/L2 ladder (no tree-sitter); OpenShift queues.

## 1. Goal
Run the pipeline at scale: roughly 1 million findings per quarter (about 80,000 per week on average, bursty), 600 applications, about 1,500 projects, repos from 100 KB to 12 GB. The orchestrator schedules work, avoids redundant work (dedupe and verdict reuse), applies size tiers and budgets, prioritizes by severity, and guarantees that nothing silently disappears: every finding ends in an explicit outcome.

## 2. Workflow engine
Preferred: **Temporal** (durable workflows, retries, timeouts, visibility). Alternative: Celery plus Redis/RabbitMQ with a Postgres state machine. Pick one in phase 0; the contracts below are engine-agnostic.

Workflow hierarchy:
```
ScanWorkflow(scan_id)
  -> SourceAcquisitionActivity(scan_id)            [02]
  -> ModuleDiscoveryActivity                       [02]
  -> FrameworkDetectionActivity per module         [04]
  -> for each language group:
       IndexBuildActivity (cached by snapshot+scope)   [03]
  -> fan-out: FindingWorkflow(finding_id) for each in-scope, non-duplicate finding

FindingWorkflow(finding_id)
  -> Route by finding_kind: path_based -> below; pattern_only -> PatternFindingWorkflow [15]
  -> SourceMatchActivity (source_match_score)      [02]
  -> ContextBuildActivity (Evidence Pack: slice, guards, dual flow check, pack_hash) [03]
  -> DedupeCheck (needs pack_hash) -> (reuse | continue)
  -> TriageActivity (single pass, bounded gap-fill) [07]
  -> ValidateActivity (+ second opinion when required) [08]
  -> WritebackActivity                             [09]
  -> EmitMetrics
```
All activities are idempotent and keyed by (finding_id, stage, input hash) so retries never duplicate writes.

## 3. Fingerprint and dedupe (essential at this volume)

**Fingerprint** = SHA-256 over: `query_name`, `cwe`, `language`, `sink_file` path, ordered `path_hash` (from 01), the Evidence Pack **`pack_hash`** (03 section 6: canonical hash of the snippets, facts, flow check results, gaps and capabilities, with whitespace and comments normalized in code), plus `playbook_version` (includes the decision-table version), `kb_version`, `prompt_version`.

The pack hash is computed from code content and tool-derived facts, not project identity, so identical flaws in forked or copied repos across projects dedupe. Identical context yields verdict reuse; changed context (code, framework facts, KB, capability matrix, or flow check) forces re-triage.

For `pattern_only` findings the fingerprint uses `rule_id`, the normalized value fingerprint (hash of the literal, never the value), file class, and the usage pack hash (15).

Dedupe rules, in order:
1. **Same finding, new scan (Recurrent):** same `similarity_id` + project, and `pack_hash` unchanged -> reuse the previous verdict (`REUSED_VERDICT`). Re-apply the writeback only if the Checkmarx predicate lacks our block (CxSAST normally carries predicate state forward for Recurrent results; verify before writing again).
2. **Same flaw across projects:** identical fingerprint in a different project -> reuse the verdict content, but **write it separately per project** (CxSAST predicates are project-scoped). Mark `reuse_source_finding_id`.
3. **In-flight dedupe:** if an identical fingerprint is currently being triaged, subscribe to its result instead of starting a new run.
4. **Invalidate** reuse when: pack hash changed, playbook or KB version changed materially (config: `reuse_across_kb_versions=false` default), source revision confidence dropped or `source_match_score` fell below its threshold, or the previous verdict was `NEEDS_HUMAN` caused by a transient failure (budget exceeded or tool timeout).
5. **Reviewer overrides win:** if a human reviewer has overturned or accepted a verdict for this fingerprint, reuse the human outcome as the verdict basis and flag it as `human_confirmed`. Never re-propose FP for a fingerprint a human marked TP.

## 4. Size tiers and budgets

Thresholds are configuration (defaults are starting points to tune in phase 0). Measure after filtering (02).

| Tier | Filtered source size | Index strategy | Budgets (defaults) |
|---|---|---|---|
| S | under 50 MB | Full Joern CPG per language | build 10 min, heap 4 GB |
| M | 50 to 300 MB | Full CPG per language, may shard by module | build 25 min, heap 8 GB |
| L | 300 MB to 2 GB | CPG scoped to modules containing findings plus local deps, sharded | build 45 min per shard, heap 16 GB |
| XL | over 2 GB after filtering | Module-scoped shards only; if a shard exceeds budget, degrade to L1 (plain-file and ripgrep context, 03 section 7) | build 60 min per shard, heap 24 GB; fallback allowed |

Per-finding budgets (defaults): context build and Evidence Pack assembly 5 min, flow check 2 min (run twice, so 4 min total), agent run 10 min wall clock, **one prefetched pass plus at most 5 gap-fill tool calls on the cheap tier and 8 on the strong tier**, max input and output tokens per run (configured per model tier, rough targets: pack 6,000 to 10,000 input tokens, 800 to 1,500 output), max 2 retries for transient errors.

On any budget exceeded: record the reason and send the finding to `NEEDS_HUMAN_NO_ANALYSIS` (or `NEEDS_HUMAN` if the agent produced a partial analysis). No unbounded retry loops. Transient infrastructure failures (network, worker crash) retry with exponential backoff and jitter, up to the configured max; after that, mark `ANALYSIS_FAILED` with reason.

Record every "no CPG" outcome (`no_cpg_reason`) for the phase-2 tree-sitter decision gate (14).

## 5. Prioritization and fairness
- Priority score = severity weight + recurrence (New before Recurrent) + age + small bonus for small repos (cheap to finish). Critical and High first, but cap a single project's concurrent findings so one huge scan does not starve others (per-project concurrency limit, default 20).
- Separate queues: `index-build` (heavy), `context` (medium), `llm` (rate-limited by provider quotas), `pattern` (cheap, high volume, 15), `writeback` (rate-limited to protect CxSAST).
- Backpressure: if CxSAST latency or error rate rises, slow the writeback queue automatically; if LLM quotas are hit, pause the `llm` queue with a visible state on the dashboard.

## 6. Model routing (cost control at 80,000 findings per week)
- Pre-routing without LLM: classify trivially benign cases deterministically where safe, for example sink file classified `test` or `generated` (still verified by the validator, never auto-closed in phase 1), then send them to the **cheap tier**. Pattern placeholders are classified deterministically in 15 and never reach an LLM.
- Tiering: cheap tier handles clear-cut, low and medium severity; strong tier (Opus-class) handles High and Critical, ambiguous cases, and any case where the cheap tier returns `weak` evidence strength or `NEEDS_HUMAN` (escalation).
- Cost levers: group findings from the same function or snapshot so they share context and the cached prompt prefix; send non-urgent Medium and Low findings through a provider batch API where available; reuse verdicts by fingerprint. Verify provider features before relying on them.
- Second opinion (different provider/model) only for: FP verdicts on High/Critical, FP verdicts with calibrated confidence below the threshold, and a random audit sample (configurable percentage, default 5%) of other FP verdicts. It compares **facts**, not only verdicts (07 section 8).
- Model names, thresholds, and sample rates are configuration. Track cost per finding and per tier on the dashboard.

## 7. State machine and events
Use the lifecycle states in `00-MAIN-DESIGN.md` section 7.5. Each transition inserts an event row:

```sql
CREATE TABLE finding_events (
  event_id BIGSERIAL PRIMARY KEY,
  finding_id UUID, scan_id TEXT, project_id TEXT, application_id TEXT,
  from_state TEXT, to_state TEXT,
  reason TEXT, detail JSONB,
  occurred_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX ON finding_events (scan_id, occurred_at);
CREATE INDEX ON finding_events (project_id, occurred_at);

CREATE TABLE fingerprints (
  fingerprint TEXT PRIMARY KEY,
  first_finding_id UUID,
  latest_verdict_id UUID,
  pack_hash TEXT,
  human_outcome TEXT,       -- none | confirmed_tp | confirmed_fp
  updated_at TIMESTAMPTZ
);

CREATE TABLE verdicts (
  verdict_id UUID PRIMARY KEY,
  finding_id UUID, fingerprint TEXT,
  verdict TEXT, evidence_strength TEXT, confidence REAL, payload JSONB,
  source TEXT,              -- agent | reused | human
  reuse_source_finding_id UUID,
  created_at TIMESTAMPTZ
);
```

## 8. Scan completion semantics
A scan is `COMPLETE` when every in-scope finding is in a terminal state: `WRITTEN`, `REUSED_VERDICT` (written), `NEEDS_HUMAN_NO_ANALYSIS` (written as proposed-needs-human), `ANALYSIS_FAILED`, `SKIPPED_DUPLICATE`. The dashboard shows `pending = in_scope - terminal`. A watchdog finds findings stuck in a non-terminal state beyond a timeout and raises them (and requeues once).

## 9. Failure handling
- Worker crash mid-index: workflow retries on a different worker; partial CPG discarded.
- Poison findings (crash the engine repeatedly): after N crashes, quarantine with `ANALYSIS_FAILED(poison)` and alert.
- Repeated failure for the same snapshot: circuit-break the snapshot (all its findings fail fast with the same reason, or drop to L1 where allowed) instead of rebuilding per finding.
- Dead-letter queue for unexpected exceptions, with an admin command to replay.
- OpenShift: a worker pod killed for exceeding its memory limit maps to `index_failed_oom`; a pod evicted or rescheduled retries on another node.

## 10. Configuration (all in versioned YAML)
Thresholds for size tiers, budgets, concurrency limits, model routing, second-opinion rules and sample rates, reuse policy flags, priority weights, timeouts, retry policy.

## 11. Acceptance criteria
- Idempotency: re-running a `FindingWorkflow` produces no duplicate verdicts or writebacks (test with forced retries at each activity).
- Dedupe: same flaw in two projects triages once and writes twice; Recurrent finding with unchanged pack hash reuses the verdict; changed context triggers re-triage; human TP is never overridden by a reused FP.
- Routing: `pattern_only` findings never enter the SAST path; `path_based` findings never enter the pattern path.
- Size tiers: simulated S, M, L, XL repos route to the right strategy; XL with an oversized shard degrades to L1 and ends `NEEDS_HUMAN` (or a restricted verdict) with a reason.
- Backpressure: injected CxSAST latency slows writeback; LLM quota exhaustion pauses `llm` queue without losing findings.
- Every in-scope finding in a test scan reaches a terminal state; a stuck-finding watchdog test passes.
- Load test: sustain at least 3x the average daily volume (about 33,000 findings per day equivalent at the 80,000 per week average) in a staging environment with mocked LLM and CxSAST, with queue depth and latency reported.
