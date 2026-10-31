# 14: Rollout Plan, Auto-Close Criteria, and Governance

Author: Surya

Read `00-MAIN-DESIGN.md` first. Cross-cutting.

## 1. Principles
- Humans decide until the numbers prove otherwise.
- The cost of a wrongly dismissed true positive is far higher than the cost of an extra review. Every threshold below favors caution.
- Expand by segment (language x vulnerability family x severity tier), not all at once.
- Define the success thresholds **now**, before seeing results, so the decision to auto-close rests on pre-agreed criteria and not on a gut feeling.
- Every phase has an explicit exit criterion and a sign-off owner.

## 2. Phases

### Phase 0: Spike and foundations (weeks 0 to 4, adjust to team size)
Goals: remove the unknowns that would change the design.
- CxSAST API spike (checklist in 09): result states, predicate update endpoint and payload, notes overwrite versus append, assignment support, notes length, mandatory comments, Recurrent inheritance, concurrency.
- **Pattern-result survey:** pull a sample of Checkmarx pattern-only results (hardcoded usernames, suspicious files, suspicious URLs) and record what location data they carry (file only, file and line, snippet); confirm 15 section 3.1 assumptions.
- Scan comment survey: sample at least 200 scan comments across the 1,500 projects; record the **build URL format(s)** that download the zip, auth needed, how long artifacts stay available, and failures; derive the host allowlist. (Source is archive-first; Git is phase 2, see 17.)
- Source fetch prototype for each format; measure revision confidence distribution and the `source_match_score` distribution (02 section 5.3); check scan-start-time resolution against known scans; measure incremental-scan share.
- **Joern evaluation** on 5 to 10 real repos per language (sizes from small to large): build success, time, memory, node-mapping success (including tabs and multibyte columns), caller recall on a labeled sample, **no-CPG rate by language, size tier and reason**. Decide where per-language supplements are required (03 section 13), the size-tier thresholds (06), and fill the capability matrix (03 section 11).
- Choose workflow engine (Temporal or Celery) and confirm OpenShift constraints (13 open questions), including `restricted-v2` admission of a Joern pod.
- Golden set kickoff: obtain the sample Checkmarx XML report (Reports UI export) and matching zips for the coding agent; verify on the 9.7 server that `XmlReportSourceLinesRange` and `AddQueryMetaDataToXmlReport` give the fields needed; plan the stratified sample and second-analyst re-review.
- Phase 2 pointer: build-to-repository mapping, introducing commit and owning team are specified in 17 and are not built in phase 1.
- Reviewers (decided): the central CISO security analyst team works one queue. The agent only sets Proposed Not Exploitable; TP and NEEDS_HUMAN get a comment only. Nothing left to decide here.
- Define governance: owner, risk sign-off path, who can approve auto-close (section 5).
Exit criteria: spike report approved; design open assumptions in `00-MAIN-DESIGN.md` section 9 resolved or consciously accepted; pilot application list chosen.

**Decision gate (end of phase 0 and again in phase 2): tree-sitter fallback.** If the measured no-CPG rate for any language or tier exceeds the agreed threshold (suggest 10% of findings), add a lightweight tree-sitter fallback for node mapping and enclosing-function lookup (an L1 enhancement). Otherwise stay Joern-only.

### Phase 1: Shadow and assist (target 8 to 12 weeks after phase 0)
Phase 1a: Shadow (dry-run):
- Pipeline runs on pilot applications (start with roughly 5 to 10 applications covering all five languages and a mix of repo sizes). Writeback in **dry-run**: payloads computed and stored, nothing written.
- Dashboard live (10). Evaluation harness running on golden set and on shadow outputs compared against analysts' independent decisions where available.
- Pattern findings (15, Checkmarx results) run in shadow with masking checks.
Exit: pipeline stable (no stuck findings, failure reasons understood), node-mapping and fetch success rates acceptable, no unexplained validator anomalies, security test suite green, **zero masking violations**, spend within budget.

Phase 1b: Assist (live writeback, proposals only):
- Enable live writeback for the pilot allowlist: proposed states, appended comments, reviewer assignment or comment fallback. Kill switch tested. Reviewers accept or overturn in CxSAST.
- Expand the allowlist in steps (for example 10 -> 50 -> 200 applications -> all) with a stability check at each step (writeback failure rate, reviewer complaints, overturn rate).
- Weekly error-analysis review of overturned FP proposals (11 section 9) feeding fixes to playbooks and decision tables, KB, framework catalog and the context builder.
- Reviewer feedback channel and training: short guide on reading the AI block, how to overturn, how to give feedback.
Exit criteria (initial defaults, to be agreed with stakeholders): at least 4 weeks of live proposals on the expanded allowlist; reviewed outcomes available for a statistically meaningful sample per priority segment; no unresolved systemic issues; operational SLOs met (13).

### Phase 2: Calibration and tuning
- Fit calibration tables from reviewer outcomes plus golden data (11 section 7); update the validator caps and the strength-to-confidence map.
- Tune decision tables, per-segment thresholds, second-opinion rates, and routing to reduce cost without raising FN risk; measure prompt cache and batch savings.
- Grow the framework catalog, sanitizer KB and pattern-family decision tables from observed unknowns (04, 05, 15).
- Re-run the tree-sitter and per-language supplement decision gates with production data.
- Measure `would_have_auto_closed` populations and their eventual reviewer outcomes for each segment.
Exit: at least one segment meets the auto-close readiness thresholds (section 3) and has human sign-off, **or** the decision is made to stay in assist mode (a valid outcome).

### Phase 3: Selective automation
- Enable auto-close **only** for allowlisted segments that passed readiness review (section 3). Pattern findings (15) are excluded in v1.
- Auto-close sets Not Exploitable with the standard AI block and an `AUTO-CLOSED` marker, via the separate feature-flagged path (09). Rate-limited, auditable, reversible by the revert command (12).
- **Random audit sampling continues permanently** (default: 10% of auto-closable findings are diverted to human review, adjustable but never below an agreed floor).
- Continuous monitoring: if any audited auto-close is found to be a true positive, an automatic alert fires and the segment is **suspended** pending review (circuit breaker).
- High and Critical remain excluded unless governance explicitly approves a change with its own evidence bar.

### Phase 4: Expansion
- More languages, families, and frameworks; more applications.
- Optional capabilities (not committed now): fix suggestions or draft PRs, ticketing integrations (explicitly out of scope until requested), deeper cross-repo analysis for flows that span services.

## 3. Auto-close readiness criteria (propose defaults; stakeholders to confirm)

A segment (language x vulnerability family, optionally with severity tier Medium or lower) becomes auto-close eligible only when **all** hold over a defined window of reviewed proposals:
1. **Volume:** at least 300 reviewed FP proposals in the segment (adjust by risk appetite; smaller segments may never qualify and that is acceptable).
2. **Precision:** the lower bound of the 95% confidence interval for FP precision (proposed FPs that reviewers accepted) is at least 99% for Medium/Low. Report the observed overturn count; any overturn requires a documented root-cause analysis and a fix before eligibility proceeds.
3. **No High/Critical:** segment limited to Medium and lower.
4. **Calibration:** expected calibration error below an agreed value, and eligible findings drawn only from the confidence range where precision was verified.
5. **Stability:** the thresholds hold across at least 4 consecutive weeks and across at least a minimum number of distinct applications (for example 20) so results are not driven by one codebase.
6. **Evidence quality:** evidence validity rate near 100%; no `injection_suspected` or `source_mismatch` in eligible findings; revision confidence `EXACT` or `HIGH`; `source_match_score` at least 0.95; `context_level` L0.
7. **Second opinion:** agreement required for all auto-closed FPs in the segment (initially), with the possibility to relax for very high-precision segments by explicit decision.
8. **Sign-off:** recorded approval from the AppSec lead and the risk/governance owner.
These numbers are starting proposals. The key requirement is that they are written down, approved, and applied mechanically (11 section 8 report) rather than adjusted after seeing results.

## 4. Metrics tracked through the rollout
Coverage, agreement, FN proxy (overturned FP proposals), FP precision with confidence intervals, calibration, reviewer acceptance time, backlog and SLO attainment, cost per finding and per application (including cache hit ratio and single-pass share), failure reasons, no-CPG rate, `source_match_score` distribution, injection and downgrade rates (by rule, including `table_disagreement`), pattern findings by family and origin, masking violations, would-have-auto-closed outcomes. All visible on the dashboard (10).

## 5. Governance and roles
- **Product/engineering owner:** Surya (and delegate) for the system.
- **AppSec lead:** owns playbooks and decision tables and allowlists, sanitizer KB, auto-close allowlist proposals.
- **Risk/governance sign-off:** approves moving any segment to auto-close and any change affecting severity handling.
- **Reviewers:** analysts receiving proposals; their feedback is a first-class input.
- **Platform/operations:** runs the OpenShift platform and on-call.
- Change control: prompts, playbooks and decision tables, KB entries, validator rules, thresholds, allowlists, and model versions all change via reviewed pull requests with evaluation results attached (11 release gates).
- Periodic review (monthly): quality report, incidents, allowlist status, cost, backlog, roadmap.

## 6. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Wrongly dismissed true positive | Asymmetry rule, ID-based evidence verification, decision-table derivation, source-match gate, second opinion on facts, High/Critical excluded, human review phases, random audit, segment circuit breaker, revert command |
| CxSAST API cannot do assignment or append | Client-side append; comment-based reviewer fallback; dashboard reviewer queues (09) |
| Scan comments unparseable for many projects | Pluggable parsers; `NEEDS_HUMAN_NO_ANALYSIS` with reason; remediation campaign to standardize scan comments (separate effort) |
| Source revision mismatch | Scan-start anchor, revision confidence, per-finding `source_match_score`, origin cross-check; approximate cases excluded from auto-close |
| Joern fails or is weak (large repos, C#, Go) | L1/L2 ladder with restricted verdicts; no-CPG rate tracking; supplements and optional tree-sitter fallback by data; conservative asymmetry rule; per-language readiness gating |
| Secret exposure in pattern findings | Dedupe by value fingerprint, masking before any prompt/log/comment, masking check blocks writeback |
| Cost overruns at volume | Dedupe and reuse (pack hash in fingerprint), tiering, prompt caching, batch APIs, caps, cost brake mode (13) |
| Reviewer fatigue or distrust | Concise comments, accurate evidence, fast feedback loop, visible quality metrics, gradual expansion |
| Prompt injection from hostile code | Controls in 07 and 12; injection findings forced to humans |
| Model or provider changes shifting behavior | Pinned model IDs, regression gates (11), per-segment re-validation after changes |
| OpenShift constraints block JVM-heavy or sandboxed workers | Phase 0 admission and sizing tests; dedicated namespace and node pool; narrow custom SCC only with review |
| Scope creep (SCA, IaC, Jira) | Explicitly out of scope for phase 1; revisit after phase 2 |

## 7. Definition of success
- Phase 1: reviewers report the AI blocks save time; overturn rate for FP proposals known per segment and trending down; no safety incidents; no masking violations; dashboard is the shared source of truth.
- Phase 3: a meaningful share of Medium/Low injection-family false positives are closed automatically with audited accuracy at or above the agreed bar, freeing analyst time for High/Critical and complex findings.

## 8. Implementation order for the coding agent (suggested build sequence)
1. 01 Ingestion (with CxSAST client wrapper, read-only first) and the Postgres schema; support `finding_kind` and empty paths.
2. 02 Source acquisition (parser chain, safe fetch, filtering, snapshots, scan-start anchor, `source_match_score`).
3. 03 Code intelligence: plain-file source verification, file classification and ripgrep tools first; then the Joern adapter for one language (suggest Java); then node mapping, slice builder, dual flow check, and Evidence Pack assembly; add other languages incrementally.
4. 06 Orchestrator skeleton (workflow, state machine, events), then dedupe (fingerprint with `pack_hash`) and budgets.
5. 10 Dashboard minimal (scan/project triaged and pending) as soon as events exist, so progress is visible from the start.
6. 07 Agent: facts-first schema, single-pass run, decision tables for SQL injection and command injection, plus 05 sanitizer KB seed for Java.
7. 08 Validator and policy engine (ID-based evidence, decision-table derivation, asymmetry rule); 11 harness with fixtures first, golden set when available.
8. 09 Writeback in dry-run, then live behind the allowlist and kill switch.
9. 15 Pattern findings (Checkmarx results) with masking.
10. 04 Framework detection and extractors for the highest-volume frameworks; expand languages (Node, Python, C#, Go).
11. 12 and 13 hardening items in parallel from day one (sandboxing, SCC admission and isolation must exist before processing untrusted repositories at scale).
