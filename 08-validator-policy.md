# 08: Verdict Validator and Policy Engine

Author: Surya

Read `00-MAIN-DESIGN.md` first. Depends on 07. Feeds 09.

## 1. Goal
A **deterministic** (non-LLM) gate between the agent's verdict and anything written to Checkmarx. It verifies evidence by snippet ID, derives the verdict the facts allow from the playbook decision table, enforces the asymmetry rule, recalibrates confidence, applies policy gates, triggers second opinions, and decides the final proposed state. This is the main defense against confident-but-wrong false-positive verdicts.

Rule of thumb: the validator can downgrade a verdict (FP -> NEEDS_HUMAN, lower strength) but never upgrade an FP on its own.

## 2. Inputs
Agent verdict (07), finding (01), Evidence Pack (03: snippets, facts, precision, unresolved edges, flow checks, `context_level`, `capabilities`, `source_match_score`), source snapshot (02: revision confidence, origin conflict), framework facts (04), sanitizer assessments (05), playbook decision table and `fp_requires` (07), historical calibration data (11).

## 3. Validation pipeline (in order)

### 3.1 Schema and integrity
- JSON schema validation; required fields per verdict type.
- `FP` must include `fp_reason_category` and at least one evidence item.
- Verdict, strength and enum values valid. Fail -> `NEEDS_HUMAN` (reason `invalid_output`).
- `pack_hash` in the verdict equals the hash of the pack on record (guards against stale packs).

### 3.2 Evidence verification (mechanical, by ID)
For each evidence item:
- `snippet_id` exists in the Evidence Pack (or was added by a recorded gap-fill tool call).
- `quote` is a **verbatim substring** of that snippet's text after whitespace normalization. Mismatch -> evidence invalid.
- The snippet's content hash (`sha`) matches the file content in the pinned snapshot at the recorded lines (detects pack tampering or drift).
- Evidence must be **executable code or config**, not only comments or string literals describing intent. Use the language's comment and string-literal detection on the snippet range (comment-only ranges are rejected as FP support). Joern's AST classification is used where a CPG exists; otherwise a per-language lexical check.
- Count valid evidence items. If an FP verdict has zero valid evidence items -> downgrade to `NEEDS_HUMAN` (`fp_evidence_unverified`).
- Also verify every snippet ID cited inside `facts.*.cites` exists.

### 3.3 Rule-derived verdict (decision table)
- Load the playbook decision table for the finding's family (07 section 5) and evaluate it over the agent's `facts` (plus pack facts for cross-checking).
- Compute `derived_verdict` (the most permissive verdict the table allows).
- If the agent's verdict is *more FP-leaning* than `derived_verdict` -> downgrade to `derived_verdict` or `NEEDS_HUMAN` (`table_disagreement`, record both).
- If the agent's verdict is *less FP-leaning* (for example agent says `TP`, table allows FP) -> keep the agent's verdict (more cautious); record `table_disagreement_conservative` for tuning.
- Cross-check agent facts against pack facts: if the agent contradicts a tool-verified pack fact (for example agent says source `constant` but pack fact `source_class = http_param`), the pack fact wins; recompute the table.
- Check `fp_requires` and `never_fp_if` mechanically. Unmet -> downgrade (`playbook_requirements_unmet`).

### 3.4 Asymmetry rule (enforced)
An FP whose justification category is `unreachable`, or whose facts rely on `reachability.status = unreachable_sound` or "no callers found" or "no flow", is **rejected** (downgrade to `NEEDS_HUMAN`) unless **all** hold:
- `unresolved_edges` for the relevant path and callers are empty (no reflection, DI, callback, dynamic dispatch ambiguity touching the path);
- The call graph precision for the relevant results is `exact` or a sound over-approximation with the dispatch constructs for that language and framework modeled (read from `capabilities.dispatch_modeled`);
- Framework detection recognized the framework for that module (`capabilities.framework_recognized = true`, not unknown indicators relevant to routing).
- `name_based` (ripgrep) results never satisfy this rule.
Also: `flow_check.raw = NO_FLOW_FOUND` alone never supports FP. Use `with_kb_sanitizers` versus `raw` as a hint to which sanitizer to inspect, not as proof.

### 3.5 Source and mapping integrity
- `revision_confidence = UNKNOWN` -> FP not allowed (downgrade).
- `revision_confidence = APPROXIMATE` -> FP allowed only if all path nodes were verified (`mapping = verified`), strength capped, and never auto-close eligible.
- `source_match_score < source_match_min` (default 0.8) -> `NEEDS_HUMAN` (`source_mismatch`).
- Any FP requires `source_match_score >= source_match_fp_min` (default 0.95).
- `origin_conflict = true` on the snapshot -> FP not allowed unless the finding's `source_match_score` is 1.0; never auto-close eligible.
- `path_mapping_failed` -> `NEEDS_HUMAN`.
- `injection_suspected` flag set -> FP not allowed; TP or `NEEDS_HUMAN` only.
- `context_level` L1: FP only for categories listed as allowed (07 row 9, test or generated); L2: no FP.

### 3.6 Contradiction checks
- Compare with `flow_check`: if the agent says FP but `flow_check.raw = FLOW_CONFIRMED` with the same sink context and no effective sanitizer assessed -> downgrade.
- Compare with sanitizer assessments: FP citing a sanitizer requires `assessment = effective`.
- If reasoning mentions a protection from framework facts, verify that fact exists (04) and `enabled = true` with no `disabled_by`.

### 3.7 Confidence calibration
The agent reports ordinal `evidence_strength` (`strong | moderate | weak`). The validator maps it to a number:
`calibrated_confidence = f(evidence_strength, family, language, tier, precision, revision_confidence, source_match_score)` using per-segment calibration curves fitted on reviewer outcomes (11).
Before data exists, use a starting table (strength -> value) and caps: FP confidence capped at 0.7 when precision is `approximate`, 0.5 when `name_based`, 0.6 when revision is `APPROXIMATE`, and multiplied by `source_match_score`. Store the raw ordinal and the calibrated value.

## 4. Policy gates

### 4.1 Phase 1 behavior (all verdicts are proposals)
Every finding gets a proposed state; nothing is closed. Mapping to Checkmarx proposed states (confirm exact state names via the result states list, see 09):
- `FP` (validated) -> Proposed Not Exploitable. This is the **only** state the agent changes.
- `TP` and `NEEDS_HUMAN` -> state left unchanged; the comment block is added and the finding goes to the central CISO analyst queue (dashboard shows the queue split by verdict).
The comment block always carries verdict, calibrated confidence, reason, evidence, `source_match_score` and revision confidence. A `proposed_by_agent` marker allows measuring.

### 4.2 Severity handling
- High and Critical FP proposals: always require a second opinion (4.3) and are flagged `high_risk_fp` on the dashboard.
- Severity adjustments are written only as a recommendation in the comment in phase 1; the Checkmarx severity field is not changed by the agent unless config enables it.

### 4.3 Second opinion (fact-level)
Triggered for: FP on High/Critical; FP with calibrated confidence below `second_opinion_threshold`; random audit sample of other FPs (default 5%); any case where the primary model escalated tiers.
- A different provider/model gets the **same Evidence Pack**, playbook and prompt, without the first verdict, and fills the same facts schema. Run it through the same validator.
- Compare **facts**, not only verdicts. Record every differing fact (for example `sanitization: effective vs partial`).
- Both FP, both pass validation, and no differing fact that feeds the decision table -> FP stands (strength = the weaker of the two).
- Any differing decision-table fact, a different verdict, or one failing validation -> `NEEDS_HUMAN` with `second_opinion_disagreement`, both fact sets stored and shown to the reviewer.

### 4.4 Auto-close eligibility (phase 3; compute from day one, act on it later)
A finding is `auto_close_eligible` only if **all** hold:
- Verdict FP, validated, second opinion agrees (where required);
- Severity Medium or lower (High and Critical never eligible without explicit governance change);
- Family and language segment is on the **auto-close allowlist** (config) because it met the thresholds in 14 and 11;
- Calibrated confidence at or above the segment threshold;
- Revision confidence `EXACT` or `HIGH`, `source_match_score >= 0.95`, all nodes verified, no `origin_conflict`;
- `context_level = L0_full`;
- Call graph precision sufficient per 3.4 or the FP justification does not rely on reachability;
- No `injection_suspected`, no `source_mismatch`;
- Not overturned previously for this fingerprint;
- `finding_kind = path_based` (pattern findings, 15, are not auto-closed in v1).
In phase 1 and 2 the flag is stored but ignored; the dashboard shows how many would have been eligible.

### 4.5 Pattern-only findings (feature 15)
Pattern findings use their own decision tables (15 section 6). The same pipeline stages apply: schema, ID-based evidence checks, table derivation, source-match on the matched line, injection check, second opinion for High, and masked-value rules. Additional gates:
- Any comment, log, dashboard field or Checkmarx note produced for a pattern finding must pass the **secret-masking check**: the matched secret value must not appear in cleartext. A failure blocks writeback (`masking_violation`).
- A `pattern_only` FP is allowed only when the decision table's `benign` rows are satisfied with verified evidence (for example the value is in a test fixture and flows to no production sink).

## 5. Output of the validator
```json
{
  "finding_id": "uuid",
  "final_verdict": "TP | FP | NEEDS_HUMAN",
  "final_proposed_state": "string",
  "derived_verdict": "TP | FP | NEEDS_HUMAN",
  "calibrated_confidence": 0.0,
  "downgrades": [{"from": "FP", "to": "NEEDS_HUMAN", "rule": "asymmetry_rule", "detail": ""}],
  "evidence_validation": [{"snippet_id": "S1", "valid": true, "reason": ""}],
  "second_opinion": {"required": true, "performed": true, "agreement": true, "model": "", "differing_facts": []},
  "auto_close_eligible": false,
  "flags": ["high_risk_fp", "approximate_revision", "injection_suspected", "table_disagreement"],
  "validator_version": "string"
}
```
Every downgrade and flag is recorded as an event (dashboard shows the downgrade reasons).

## 6. Human-outcome feedback
Reviewer outcomes (accept or overturn, from 09/10) are joined back to verdicts and feed:
- calibration curves per segment (family, language, tier, precision);
- decision-table tuning (rows whose FP outcomes are overturned);
- sanitizer KB entry effectiveness (05);
- per-segment overturn rates driving auto-close allowlist decisions (14).
Maintain a `reviewer_outcomes` table keyed by finding_id with outcome and optional reviewer comment.

## 7. Data model
```sql
CREATE TABLE validated_verdicts (
  finding_id UUID PRIMARY KEY,
  verdict_id UUID,
  final_verdict TEXT, derived_verdict TEXT, final_proposed_state TEXT,
  evidence_strength TEXT, calibrated_confidence REAL,
  downgrades JSONB, flags JSONB,
  second_opinion JSONB,
  auto_close_eligible BOOLEAN,
  pack_hash TEXT,
  validator_version TEXT,
  created_at TIMESTAMPTZ
);
CREATE TABLE reviewer_outcomes (
  finding_id UUID PRIMARY KEY,
  outcome TEXT,             -- accepted | overturned | modified | pending
  reviewer TEXT, reviewer_comment TEXT,
  decided_at TIMESTAMPTZ
);
```

## 8. Acceptance criteria
- Each rule has unit tests with positive and negative cases: fabricated snippet ID, quote not a substring, snippet hash drift, comment-only evidence, unmet `fp_requires`, asymmetry rule with and without unresolved edges and with `name_based` precision, flow-check contradiction, sanitizer assessed not effective, approximate revision, low `source_match_score`, origin conflict, injection flag, table disagreement in both directions, agent fact contradicting a pack fact.
- **Property test:** for any input, the validator output verdict is never "more FP" than the agent verdict (it may only downgrade).
- **Table test:** every row of every shipped decision table has a fixture; derived verdicts match.
- Second-opinion flow tested: agreement, verdict disagreement, differing decision-table fact, non-decision fact difference, second model failing schema.
- Confidence caps applied per precision, revision and `source_match_score` segment (table-driven tests).
- `auto_close_eligible` truth table tests covering every condition; High/Critical and `pattern_only` never eligible.
- Masking test: a pattern finding whose comment contains the secret value is blocked from writeback.
- Replaying a recorded set of agent outputs produces identical validator outputs (determinism).
- Validator version recorded; a version bump forces re-validation of non-written findings only.
