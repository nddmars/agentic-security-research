# AI SAST Triage Agent: Main Design

Author: Surya
Status: Draft v2 (updated after design review: Joern-only code analysis, Evidence Pack contract, facts-first agent, pattern findings, OpenShift)
Audience: Coding agent and engineering team. Each feature has its own file; this file is the index, the contract, and the source of shared decisions. Read this first, then the feature file you are implementing.

---

## 1. Purpose

Build an AI agent that triages Checkmarx CxSAST results using the scanned source code. For every finding the agent produces a verdict with cited evidence:

- `TP` (true positive)
- `FP` (false positive)
- `NEEDS_HUMAN`

The agent writes its verdict back to Checkmarx as a **proposed** state with a reason appended to the per-result comment. Humans review. After metrics prove accuracy, selected low-risk classes can be auto-closed (phase 3).

**Primary design principle: minimize false negatives.** A true positive wrongly dismissed is the costly failure. The agent may be unsure; it must never be confidently wrong. Every design decision below favors "send to a human" over "dismiss with weak evidence".

## 2. Scope

In scope:
- SAST findings from on-prem Checkmarx CxSAST (believed to be version 9.7; verify).
- Two finding kinds (see section 7.2):
  - `path_based`: taint-style findings with a source-to-sink path (SQL injection, command injection, path traversal, XSS, and similar).
  - `pattern_only`: findings that come from pattern or regex queries and have **no** source-to-sink path (for example hardcoded usernames, suspicious files, suspicious URLs). Handled by feature 15.
- Pattern-only Checkmarx results (hardcoded usernames, suspicious files and URLs) are triaged in context (feature 15). This system runs no pattern scan of its own; Checkmarx already holds the rules.
- Languages: Java, C#, Python, Node.js (JavaScript and TypeScript), Go.
- Phase 1 output: Checkmarx writeback only, plus a dashboard.

Out of scope for now:
- SCA, IaC, containers.
- Jira or ServiceNow integration.
- Auto-closure (phase 3, gated by metrics).
- Fix-PR generation (later).
- Semgrep. Decision: Checkmarx already does pattern detection; Joern provides the independent flow check.
- tree-sitter and a separate AST or call-graph stack in v1. Decision: Joern is the only code-analysis engine; ripgrep and path rules cover the gaps (see 03). tree-sitter is an optional phase-2 fallback, added only if measured "no CPG" rates justify it.

## 3. Key facts and constraints (from stakeholder)

| Item | Value |
|---|---|
| Checkmarx | On-prem CxSAST, believed 9.7 (verify exact version) |
| Scale | About 1 million findings per quarter, about 600 applications, about 1,500 Checkmarx projects |
| Weekly volume | About 80,000 findings per week on average, many recurrent |
| Naming | Each project is prefixed with the application ID, then the component name |
| Repo shapes | Monoliths and microservice repos; size ranges from 100 KB to 12 GB (12 GB is rare) |
| Frameworks | Unknown and varied; must be auto-detected per repo |
| Finding mix | Mostly input validation and injection; also pattern findings with no path |
| Source location | The **scan comment** contains a network/HTTP location of the scanned source. The agent **only reads** the scan comment, never modifies it |
| Writeback | Per-result predicate: state, severity, notes. **Append** to existing notes; never overwrite |
| Phase 1 writeback | The agent only ever proposes Not Exploitable (for FP). For TP and NEEDS_HUMAN the Checkmarx state is left unchanged and only a comment is added. Reviewers are the central CISO security analyst team (one queue) |
| Models approved | Claude (including Opus-class) and Gemini |
| Code to models | No restriction on sending code; still redact secrets as a safeguard |
| Dashboard | Per scan ID and project ID: findings triaged, triages pending (plus extras in feature 10) |
| Tickets | No Jira in phase 1 |
| Platform | OpenShift (see 13) |

## 4. Architecture overview

```
CxSAST (on-prem)
   | REST/OData
   v
[F01 Ingestion] --> finding store (Postgres)
   |  scan comment -> source URL
   v
[F02 Source Acquisition] -> workspace (repo@revision, filtered, source_match_score)
   |
   v
[F03 Code Intelligence] (Joern CPG + ripgrep + path rules; builds the Evidence Pack)
   |   +--> [F04 Framework Detection + Facts]
   |   +--> [F05 Sanitizer Knowledge Base]
   v
[F06 Orchestrator] (queue, delta/dedupe, size tiers, budgets)
   |
   v
[F07 Triage Agent] (facts-first LLM, decision-table playbooks, bounded gap-fill tools)
   |
   v
[F08 Validator & Policy Engine] (deterministic; rule-derived verdict check; second opinion)
   |
   v
[F09 Checkmarx Writeback] ---> CxSAST per-result predicate

[F15 Pattern Findings Triage (Checkmarx results only)] reuses F02, F03 (pack), F07 (agent runtime), F08, F09
[F10 Dashboard & Metrics] <--- all stages emit events
[F11 Evaluation Harness] <--- golden set + reviewer feedback
[F12 Security & Isolation] cross-cutting
[F13 Infrastructure & Deployment] cross-cutting (OpenShift)
```

## 5. Feature file index

Implement in roughly this order. Each file is self-contained but depends on the shared contracts in section 7 of this file.

| File | Feature | Depends on |
|---|---|---|
| `01-ingestion.md` | CxSAST ingestion and normalization | none |
| `02-source-acquisition.md` | Scan-comment parsing, source fetch, filtering, source verification | 01 |
| `03-code-intelligence.md` | Joern, ripgrep, node mapping, slicing, Evidence Pack | 02 |
| `04-framework-detection.md` | Per-repo framework detection and facts extraction | 02, 03 |
| `05-sanitizer-kb.md` | Sanitizer and safe-pattern knowledge base | 03 |
| `06-orchestrator.md` | Queueing, delta triage, dedupe, size tiers, budgets | 01, 02, 03 |
| `07-triage-agent.md` | Facts-first LLM agent, decision-table playbooks, model tiering | 03, 04, 05, 06 |
| `08-validator-policy.md` | Deterministic validator, rule-derived verdicts, second opinion | 07 |
| `09-checkmarx-writeback.md` | Predicate update, append-only comments, assignment | 08 |
| `10-dashboard-metrics.md` | Dashboard and metrics | 01, 06, 08, 09 |
| `11-evaluation-harness.md` | Golden set, metrics, regression evals | 07, 08 |
| `12-security-isolation.md` | Threat model, sandboxing, prompt injection defense | all |
| `13-infrastructure.md` | OpenShift deployment, scaling, observability | all |
| `14-rollout-plan.md` | Phases, auto-close criteria, governance | all |
| `15-pattern-scan-triage.md` | Pattern-only Checkmarx findings (no path), triaged in context | 02, 03, 07, 08, 09 |
| `17-phase2-git-history.md` | Phase 2: build-to-repository mapping, introducing commit, owning team (individual names policy gated) | 02, 09, 10, 12 |
| `16-modules-and-libraries.md` | Code layout: modules, functions, implementation libraries, test map | all |

All fifteen feature files are v2 and live in this folder. Section 11 lists what changed in each.

## 6. Technology decisions

- Language for services: Python 3.12 (orchestrator, agent, validator, API, ingestion). Pin 3.12 in images, CI and lockfiles; confirm that the base image and every pinned dependency support 3.12.
- Code analysis: **Joern is the only code-analysis engine in v1** (node mapping, slicing, call graph, flow check). ripgrep for config search, string-wired dispatch and name-based fallback; plain path and naming rules for file classification. No separate AST or call-graph tool. Per-language supplements (Roslyn for C#, `go/callgraph` plus gopls for Go, jdtls or SootUp for Java, pyright for Python, TypeScript compiler API for Node) are added **only** where phase 0 measurements show insufficient caller recall (see 03 section 9).
- Queue and workflow: Temporal (preferred) or Celery with Redis/RabbitMQ. Choose one; see 06.
- Database: PostgreSQL for findings, verdicts, fingerprints, events, metrics. Object storage (S3 compatible) for CPGs, evidence packs, and prompt/response audit logs.
- Models: Claude (cheap tier plus Opus-class) and Gemini via the approved gateway. Model names are configuration, never hard-coded.
- Dashboard: web UI (React or similar) over a read-only API backed by Postgres. Tool-agnostic; see 10.
- Platform: OpenShift (see 13).
- All config in versioned YAML or environment variables; secrets in the organization's secret manager.

## 7. Shared contracts (all features must conform)

### 7.1 Identifiers
- `scan_id`: CxSAST scan ID (integer from CxSAST, stored as string).
- `project_id`: CxSAST project ID.
- `project_name`: full Checkmarx project name. Parse `application_id` and `component` from it using the naming convention (prefix = application ID, then component). The parsing rule must be configurable (regex in config) and must not fail the pipeline if it does not match; store `application_id = UNKNOWN` and log.
- `finding_id`: internal UUID.
- `result_id`: CxSAST path ID/result identifier within a scan (store both `scan_id` and the CxSAST identifiers needed to write back).
- `similarity_id`: CxSAST similarity ID, stable across scans for the same flow.
- `fingerprint`: our own stable hash (defined in 06) used for dedupe and verdict reuse. It includes the Evidence Pack hash (7.7).

### 7.2 Finding record (normalized)
```json
{
  "finding_id": "uuid",
  "finding_kind": "path_based | pattern_only",
  "scan_id": "string",
  "project_id": "string",
  "project_name": "string",
  "application_id": "string",
  "component": "string",
  "cx_result_ref": {"scan_id": "", "path_id": "", "similarity_id": ""},
  "query_name": "string",
  "cwe": "string",
  "language": "java|csharp|python|javascript|typescript|go|other",
  "severity": "Critical|High|Medium|Low|Info",
  "cx_state": "string",
  "cx_status": "New|Recurrent|Fixed",
  "existing_notes": "string",
  "path": [
    {
      "node_index": 0,
      "file": "string",
      "line": 0,
      "column": 0,
      "length": 0,
      "name": "string",
      "snippet": "string",
      "role": "source|propagator|sanitizer_candidate|sink|match"
    }
  ],
  "first_seen_scan_id": "string",
  "ingested_at": "timestamp"
}
```
For `pattern_only` findings the `path` holds one or a few nodes with `role: match` (the matched location). Never assume `path` has a source and a sink. The router (06) sends `pattern_only` findings to the pattern pipeline in 15.

### 7.3 Evidence precision tags
Every tool result from the Code Intelligence Service carries:
```json
{
  "precision": "exact | approximate | name_based",
  "unresolved_edges": [{"kind": "reflection|di|dynamic_dispatch|callback|unknown", "where": "file:line"}],
  "engine": "joern|ripgrep|lsp|roslyn|gocallgraph|other"
}
```
The validator uses these (see 08). **Asymmetry rule:** concluding "reachable" requires precise evidence; concluding "unreachable, dead, or no flow" requires a sound over-approximation with all relevant dispatch constructs modeled. Otherwise the FP verdict is downgraded to `NEEDS_HUMAN`. ripgrep results are always `name_based`.

### 7.4 Verdict schema (output of the agent, validated by 08): facts first
The model fills structured **facts** (each citing Evidence Pack IDs) before it concludes. The verdict is checked against a deterministic decision table (07 section 5, 08 section 3.3).
```json
{
  "finding_id": "uuid",
  "facts": {
    "source": {"class": "attacker_controlled|internal|constant|second_order|unknown", "cites": ["S1"]},
    "sink_context": {"value": "string (shared taxonomy, see 05)", "cites": ["S3"]},
    "sanitization": {"assessment": "effective|partial|ineffective|none|unknown", "cites": ["S4"]},
    "reachability": {"status": "reachable|unreachable_sound|unknown", "auth": "none|authn|role|unknown", "cites": []},
    "code_class": "prod|test|generated|vendored",
    "injection_indicators": []
  },
  "gap_requests": [{"gap_id": "G1", "resolved": true, "via": "tool_name", "new_snippets": ["S9"]}],
  "verdict": "TP | FP | NEEDS_HUMAN",
  "evidence_strength": "strong | moderate | weak",
  "adjusted_severity": "Critical|High|Medium|Low|Info|null",
  "fp_reason_category": "sanitized | not_attacker_controlled | unreachable | test_or_generated_code | safe_api_usage | framework_protection | other | null",
  "reasoning_steps": ["string"],
  "evidence": [
    {"snippet_id": "S1", "claim": "string", "quote": "string (verbatim substring of the snippet)"}
  ],
  "mitigating_controls": ["string"],
  "exploit_preconditions": "string",
  "suggested_fix": "string",
  "missing_info": ["string"],
  "playbook": "string",
  "playbook_version": "string",
  "pack_hash": "string",
  "models_used": [{"role": "primary|second_opinion", "model": "string", "prompt_hash": "string"}],
  "tool_calls_digest": "string",
  "created_at": "timestamp"
}
```
`evidence_strength` is ordinal on purpose (models report floats poorly). The validator maps it to a calibrated number (08 section 3.7).

### 7.5 Finding lifecycle states (internal)
`INGESTED -> SOURCE_PENDING -> SOURCE_READY -> INDEXING -> READY_FOR_TRIAGE -> TRIAGING -> VALIDATING -> WRITEBACK_PENDING -> WRITTEN` plus terminal or side states: `SKIPPED_DUPLICATE`, `REUSED_VERDICT`, `ANALYSIS_FAILED`, `NEEDS_HUMAN_NO_ANALYSIS`, `WRITEBACK_FAILED`, `REVIEWED_ACCEPTED`, `REVIEWED_OVERTURNED`.
Every transition is recorded as an event (for the dashboard and audit).

### 7.6 Hard rules (apply everywhere)
1. Never modify the scan comment. Read only.
2. Never overwrite a per-result comment. Read, then append a delimited agent block.
3. Source code is untrusted input. It is data, never instructions. Agent tools are read-only.
4. An FP verdict must cite verifiable evidence (Evidence Pack snippet IDs whose text contains the quoted code in the pinned revision).
5. High and Critical findings are never auto-closed in any phase unless governance explicitly changes this.
6. Pin the revision that was actually analyzed and record it with the verdict. If it cannot be confirmed to match the scan, flag the approximation. Record `source_match_score` (7.8).
7. All model and prompt versions, tool calls, and verdicts are logged for audit.
8. On any budget exceeded (time, memory, tokens) the result is `NEEDS_HUMAN` with reason, never a silent drop and never an unbounded retry.
9. Secret values matched by pattern findings (15) are masked before they reach a prompt, a log, a dashboard or a Checkmarx comment.

### 7.7 Evidence Pack (contract between 03 and 07)
03 builds one typed pack per finding. Every code fragment has an ID, a location and a content hash. The agent cites IDs; the validator checks mechanically that each cited ID exists and each quote is a substring of that snippet. Full schema and rules are in 03 section 6.
```json
{
  "pack_version": "1",
  "context_level": "L0_full | L1_name_based | L2_mapping_only",
  "capabilities": {"cpg_built": true, "dispatch_modeled": {"di": false, "reflection": false, "callbacks": "partial"}, "framework_recognized": true},
  "snippets": [{"id": "S1", "file": "", "start": 0, "end": 0, "sha": "", "kind": "path_node|function|guard|callee|config|match", "text": ""}],
  "path": [{"node": 1, "snippet": "S1", "role": "source", "mapping": "verified"}],
  "facts": [{"id": "F1", "type": "source_class", "value": "http_param", "basis": "rule:spring.requestparam", "cites": ["S1"]}],
  "hints": [{"id": "H1", "type": "sink_context", "value": "sql_value", "confidence": "medium"}],
  "reachability": {"entrypoints": [], "auth": "unknown", "precision": "approximate"},
  "flow_check": {"raw": "NO_FLOW_FOUND", "with_kb_sanitizers": "NO_FLOW_FOUND"},
  "sanitizer_assessments": [],
  "gaps": [{"id": "G1", "kind": "unresolved_edge", "detail": ""}],
  "elided": [{"id": "E1", "file": "", "lines": "", "reason": "not data-dependent"}],
  "source_match_score": 0.0,
  "pack_hash": "sha256"
}
```
**Facts** are deterministic and tool-verified. **Hints** are heuristic and must be verified by the agent against the snippets.

### 7.8 Source verification fields (from 02)
- `revision_confidence`: `EXACT | HIGH | APPROXIMATE | UNKNOWN`.
- `source_match_score` (0.0 to 1.0, per finding): fraction of Checkmarx path-node snippets whose text is found at the mapped location in the fetched file. Computed by 02/03. Low scores cap confidence and forbid FP (08 section 3.5).

## 8. Phased rollout (summary; details in 14)

- Phase 0, spike: confirm CxSAST 9.7 APIs (states, predicate update, assignment, notes append and length), scan-comment formats, source fetch, Joern frontend maturity, "no CPG" rates.
- Phase 1, shadow and assist: agent writes proposed states and comments, assigned to a reviewer; dashboard live; humans decide; collect metrics.
- Phase 2, calibration: tune per vulnerability family and language using reviewer feedback; decide on supplements and the optional tree-sitter fallback from measured data.
- Phase 3, selective automation: auto-close only allowlisted low and medium severity families that meet pre-defined thresholds; random audit sampling continues.
- Phase 4, expansion: more languages and families, optional fix suggestions.

## 9. Open assumptions (verify during phase 0)

1. Exact CxSAST version and API surface (believed 9.7). Confirm result states list via `GET /sast/resultStates`, predicate update endpoint, and per-result comment behavior (append is done client-side by read then write).
2. Whether assignment of a result to a user is possible via API in 9.7. If not, fall back to putting the reviewer name in the comment (see 09).
3. Source acquisition (decided): Checkmarx runs in the pipeline and the scan comment holds a **build URL that downloads a zip of the scanned code**. The archive path in 02 is the primary route in phase 1. Git is not used in phase 1; build-to-repository mapping and Git history are phase 2 (17). The build-URL format still needs a survey (14 phase 0).
4. Maximum notes length in the predicate. See 09.
5. Joern frontend maturity for C# and Go on your code; column offset handling for tabs and multibyte characters. See 03.
6. Peak-week behavior for ~80,000 findings per week is unknown.
7. Dashboard authentication and hosting standards (SSO, OpenShift route policy).
8. Reviewers (decided): the central CISO security analyst team works one queue; no per-application champions.
9. Checkmarx states (decided): the agent only sets Proposed Not Exploitable for FP. TP and NEEDS_HUMAN keep the existing state and get a comment only. No other states need to be chosen.
10. Golden set (decided source): a Checkmarx XML report exported through the Reports UI (true/false-positive state, analyst comments in the Remark field, and Path). Settings to verify on the 9.7 server: `XmlReportSourceLinesRange` above 0 for source lines and `AddQueryMetaDataToXmlReport=true` for query details (documented for 9.0 to 9.2). The sample XML export and the matching zips are not available yet and will be given to the coding agent.
11. Whether Checkmarx pattern-only findings carry any location data beyond file and line (15). If only a file is given, the matching line is located by a family search.
12. **Coding agent starts here:** read the sample XML export and the matching zips first. Build the XML parser against the real sample, and verify the two report settings on the 9.7 server.

## 10. Definition of done for phase 1

- Ingests new scans from CxSAST automatically and incrementally.
- Fetches source for the scanned revision for all five languages, with filtering, size tiering and per-finding source verification.
- Produces a verdict, validated, for every in-scope finding, or an explicit `NEEDS_HUMAN`/`ANALYSIS_FAILED` outcome with reason.
- Handles `pattern_only` findings from Checkmarx through feature 15 (no independent pattern scan).
- Writes proposed state plus appended comment (and assignment where supported) without destroying existing notes.
- Dashboard shows per scan ID and project ID: total, triaged, pending, plus verdict and failure breakdowns.
- Evaluation harness runs against a golden set (when available) and reports false-negative rate, agreement, and calibration.
- Full audit trail for every verdict.

## 11. Change log from v1 (impact list)

| File | Change |
|---|---|
| 00 | Joern-only; OpenShift; finding kinds; Evidence Pack; facts-first verdict schema; source_match_score; feature 15 added |
| 02 | Scan-start-time rule (was scan_finished_at); source_match_score; incremental scans; metadata cross-check; upload path flagged as open |
| 03 | tree-sitter removed; ripgrep fallback; Evidence Pack; pre-classification; token-based slicing; dual flow check; degradation ladder; capability matrix |
| 05 | Matching is by Joern symbol resolution; decision-table link; sink context taxonomy extended |
| 07 | Facts-first output, run modes, decision-table playbooks, fact-level second opinion, failure table |
| 08 | Rule-derived verdict check, ID-based evidence validation, source_match_score gates, pattern-finding rules |
| 13 | OpenShift (SCCs, non-root UIDs, routes, network policy) |
| 14 | Phase and build-order updates; tree-sitter optional; feature 15 |
| 15 | New |
| 01 | Stores `finding_kind`, scan start time and source origin; handles pattern results with no path |
| 04 | No tree-sitter: ripgrep/regex plus Joern queries; facts feed the capability matrix and source-class rules |
| 06 | Fingerprint includes `pack_hash`; routes `pattern_only` to 15; single-pass plus bounded gap-fill budgets; L1 fallback instead of tree-sitter |
| 09 | Comment block v2 (evidence strength, source match, context level); masking check blocks pattern writes; agent sets only Proposed Not Exploitable, otherwise comment only |
| 02 | v2.1: archive-first (build URL zip), Git moved to phase 2 (17); pattern-scan exception removed |
| 11 | v2.1: golden set from the Checkmarx XML report |
| 15 | v2.1: Checkmarx pattern results only; own scan, pre-filter and filtered inventory removed |
| 17 | New: phase 2 Git mapping, introducing commit, ownership |
| 10 | Pattern-finding views, source-match and context-level breakdowns, no-CPG rate, fact-level second-opinion stats |
| 11 | Fact accuracy, decision-table fixtures, pattern golden set, masking gate |
| 12 | OpenShift controls, no tree-sitter, pattern masking, never fetch suspicious URLs |
