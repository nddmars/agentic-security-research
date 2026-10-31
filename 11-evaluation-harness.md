# 11: Evaluation Harness

Author: Surya

Read `00-MAIN-DESIGN.md` first. Depends on 07, 08, and reviewer feedback from 09/10.

v2 changes: evaluates the Evidence Pack, fact extraction and decision tables separately from verdicts; adds source-match and no-CPG metrics; adds a pattern-finding golden set and masking tests (15).

## 1. Goal
Measure the system's accuracy honestly, per segment, before and during rollout, and use the numbers to decide when auto-close is safe. The harness evaluates components independently (code-intelligence layer, fact extraction, decision tables, validator) so a bad verdict can be traced to bad context versus bad reasoning.

Priority order of metrics:
1. **False-negative rate**: true positives that the system proposed as FP, especially High/Critical. Target near zero.
2. Agreement with analysts.
3. Confidence calibration.
4. Coverage: share of findings the system can decide without `NEEDS_HUMAN`.
5. Analyst time saved, cost per finding, latency.

## 2. Datasets

### 2.1 Golden set (build in phase 0/1)
- Source (decided): a **Checkmarx XML report** exported through the Reports UI. It carries the true/false-positive state, analyst comments (Remark field) and a Path element per result. Source lines appear only when `XmlReportSourceLinesRange` is above 0, and query details need `AddQueryMetaDataToXmlReport=true` (documented for 9.0 to 9.2; verify on the 9.7 server). The sample export and matching zips are not available yet; the coding agent reads the sample first and builds `load_xml_report()` against it. Items need source at the scanned revision (the zip).
- Sampling: stratified by language, vulnerability family and query, severity, framework (as detected), repo size tier, and original verdict (TP and FP). Include hard cases: custom sanitizers, DI-heavy code, framework protections, wrapper functions, second-order flows, test and generated code.
- Size targets (initial): at least 100 per language per priority family where available, at least 30 per rare segment; total roughly 2,000 to 5,000. Adjust to what exists.
- Label quality: each item has the original analyst verdict, plus a **re-review** by a second analyst for a sample (20%) to estimate label noise. Disputed labels flagged and excluded from headline metrics (kept in a separate "ambiguous" set).
- Store: `golden_items(item_id, project, scan_id, snapshot ref, finding record, label, label_source, reviewer_notes, segment tags, ambiguity flag)`. The dataset is versioned (immutable releases) and access-controlled (it contains source references).
- **Contamination control:** never put golden-set items or their labels into prompts, KB entries, or decision-table examples. Track item IDs used in any fixtures; CI checks for overlap.
- **Golden fact labels (subset):** for a sample, analysts also label the intermediate facts (source class, sink context, sanitization assessment, reachability) so fact extraction can be scored directly (3.2).

### 2.2 Pattern-finding golden set (15)
- A separate labeled set of Checkmarx pattern results (hardcoded username, suspicious file, suspicious URL) with analyst verdicts and usage labels, taken from the same XML report.
- Values in this set are stored masked or as fingerprints; the set must contain no live secrets.

### 2.3 Synthetic and fixture suites
- Hand-built small repos per language and family with known TP and FP variants (parameterized versus concatenated, effective versus bypassable sanitizer, DI-hidden caller, comment-injection, `ORDER BY` with bind parameters, etc.). Used for fast regression in CI.
- Decision-table fixtures: facts -> expected allowed verdict, with no LLM involved.
- Adversarial suite: prompt-injection fixtures (see 07, 12), path-mapping edge cases (tabs, CRLF, BOM, Unicode), archive and SSRF cases for 02, source-mismatch cases (exact SHA with rewritten files), planted secrets for masking tests.

### 2.4 Live shadow data (from phase 1)
- Reviewer outcomes on live proposals (accepted or overturned) join to verdicts. This is the long-term evaluation stream and the primary basis for the auto-close decision.
- **Random audit sample:** randomly route a configurable percentage of FP proposals (including would-be-eligible ones) to mandatory human review and track outcomes, even after auto-close is on.

## 3. Component-level evaluations

### 3.1 Code-intelligence layer (03, independent of the LLM)
- Node-mapping success rate and failure categories (including tabs and multibyte columns, per Joern frontend).
- `source_match_score` distribution; share of findings below the mismatch threshold; correlation with analyst-reported "wrong code version".
- Caller recall and precision versus hand-labeled call chains (sample per language and framework), for Joern alone and for each supplement; ripgrep fallback precision.
- Entry-point resolution rate; share with unresolved edges.
- Slice sufficiency: analysts rate "contains everything needed to decide" (yes/partly/no) on a sample; track token size distribution.
- Dual flow-check agreement with Checkmarx paths and with analyst labels (raw versus KB-excluded).
- Index build time, memory, failure rate by size tier; **no-CPG rate by language, tier and reason**.
- Pre-classification accuracy (source class, sink context hint, code class) against golden fact labels.

### 3.2 Agent (07)
Run on the golden set with fixed Evidence Packs (cached) to isolate model reasoning. Report per model tier and provider:
- **Fact accuracy** per fact type (source class, sink context, sanitization assessment, reachability) against golden fact labels; this is the primary agent metric because verdicts derive from facts.
- Confusion matrix (TP/FP/NEEDS_HUMAN versus label).
- **FN rate** (label TP, predicted FP) with confidence intervals; separately for High/Critical.
- FP recall (labeled FP correctly predicted FP) and the share sent to NEEDS_HUMAN.
- Evidence validity rate (snippet ID exists and quote is a substring); schema validity rate; gap-fill call counts; single-pass share; cost, cache hit ratio and latency.

### 3.3 Decision tables and validator (08)
- Table coverage: every row exercised by fixtures; derived-versus-agent disagreement rates on the golden set (which rows cause losses or catches).
- Re-run the validator over agent outputs: how many wrong FPs are caught by each rule; how many correct FPs are lost (cost of caution). Per-rule precision and recall of downgrades.
- Second-opinion value: FNs caught by fact-level disagreement versus extra NEEDS_HUMAN volume; which facts disagree most.

### 3.4 Pattern pipeline (15)
- Fact accuracy (literal nature, usage sink) and verdict accuracy against the pattern golden set, per family.
- **Masking test:** planted secrets never appear in prompts, logs, comments or dashboards.

### 3.5 End to end
Full pipeline on golden set with real context building. Compare to analyst labels; this is the headline number. Also report ablations: without sanitizer KB, without the KB-excluded flow check, without second opinion, cheap tier only, Joern-only versus Joern plus supplement, L0 versus forced L1, to know what each part contributes.

## 4. Metrics definitions
- **FN rate** = (# labeled TP predicted final FP) / (# labeled TP). Report overall and by segment; Wilson 95% intervals. Because TPs may be rare in some segments, report counts alongside rates and mark segments with insufficient data.
- **FP precision** = (# predicted FP that are labeled FP) / (# predicted FP). This is the quantity auto-close relies on.
- **Coverage** = (# decided TP or FP) / (# all).
- **Agreement** = (# matching labels) / (# all decided).
- **Fact accuracy** = (# facts matching golden fact labels) / (# labeled facts), per fact type.
- **Calibration**: bucket calibrated confidence (for example deciles); compare to observed FP precision; report expected calibration error and reliability curves per segment.
- **Cost per finding** and **analyst minutes saved** (estimated from review time samples).

## 5. Segmenting
Metrics sliced by: language, family, query name, severity, framework (detected), repo size tier, precision level (`exact/approximate/name_based`), revision confidence, `source_match_score` band, context level (L0/L1/L2), finding kind, model tier. Minimum sample size per slice before it influences decisions (config, default 100 reviewed FP proposals).

## 6. Regression and release gates
- CI suite: fixtures (including decision-table fixtures, which need no model) plus a golden-set subset (smoke set, a few hundred items) on every change to prompts, playbooks and decision tables, KB entries, validator rules, model versions, or context-builder logic.
- Full golden run before any release to production and weekly in production.
- **Gates** (initial defaults; set precisely with stakeholders): no increase in FN rate beyond the confidence interval on High/Critical; no drop in fact accuracy, schema or evidence validity; zero masking violations; cost and latency within budget. A failed gate blocks release.
- Every run records versions: model IDs, prompt versions, playbook and decision-table versions, KB versions, validator version, code-intelligence engine versions, golden set release.

## 7. Calibration pipeline
- The agent reports ordinal evidence strength (`strong/moderate/weak`). Fit per-segment mapping from strength (plus precision, revision, source-match, context level) to observed FP precision from reviewer-labeled outcomes plus golden data (isotonic or Platt, with a held-out validation split).
- Publish calibration tables consumed by 08 (`calibrated_confidence`). Re-fit on a schedule (monthly) and when versions change; keep prior versions for audit.
- Before enough data exists, use the conservative caps in 08.

## 8. Auto-close readiness (feeds 14)
For each segment (family x language, optionally x severity tier), compute: number of reviewed FP proposals, overturn count, FP precision with lower confidence bound, calibration error, `would_have_auto_closed` counts and outcomes. A segment is **ready** only if it meets the pre-agreed thresholds in 14 (for example minimum reviewed count and a lower confidence bound on FP precision above the agreed level), and a human sign-off is recorded. Output a readiness report page (also surfaced on the dashboard quality page, 10). Pattern findings are not eligible in v1.

## 9. Labeling and reviewer workflow support
- Reviewer outcomes are captured automatically (09). Provide a light feedback mechanism: reviewers can add a structured reason when overturning (choose from a list: sanitizer ineffective, input attacker-controlled, reachable, wrong code version, other) to speed error analysis.
- Error analysis tooling: for each overturned FP, a report with the Evidence Pack, agent facts, decision-table derivation, validator output, and a categorization (context error, fact-extraction error, table error, KB gap, tooling gap, label noise). Weekly review meeting artifacts generated from this.

## 10. Data model
```sql
CREATE TABLE eval_runs (
  run_id UUID PRIMARY KEY,
  kind TEXT,           -- ci | golden_full | shadow_weekly | ablation | pattern
  versions JSONB,      -- models, prompts, playbooks, tables, rules, kb, validator, engines, dataset release
  started_at TIMESTAMPTZ, finished_at TIMESTAMPTZ,
  summary JSONB
);
CREATE TABLE eval_results (
  run_id UUID, item_id TEXT,
  predicted TEXT, label TEXT, evidence_strength TEXT, calibrated_confidence REAL,
  facts_predicted JSONB, facts_label JSONB,
  segment JSONB, downgrades JSONB, cost REAL, latency_ms INT,
  PRIMARY KEY (run_id, item_id)
);
```

## 11. Acceptance criteria
- Golden set loader enforces schema and immutability by release; contamination check passes in CI (no golden item or label text found in prompts, KB, tables, rules or fixtures).
- Metrics code verified against hand-computed examples (small tables with known confusion matrices, Wilson intervals, calibration error, fact accuracy).
- A full golden run produces a report with all metrics sliced by the segments in section 5, with sample sizes shown, and marks under-sampled segments.
- Ablation runs work and report deltas.
- Release gates implemented and demonstrated to block a deliberately degraded prompt (test: a prompt change that increases FNs, or lowers fact accuracy, fails the gate).
- Calibration tables are produced, versioned, and consumed by the validator in a test.
- Readiness report computes per-segment status from seeded reviewer outcomes and matches an independent calculation.
- Masking test suite runs in CI and blocks release on any leak.
