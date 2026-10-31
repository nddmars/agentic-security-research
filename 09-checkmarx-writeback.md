# 09: Checkmarx Writeback

Author: Surya

Read `00-MAIN-DESIGN.md` first. Depends on 08.

v2 changes: comment block carries evidence strength, `source_match_score` and context level; pattern-finding comments (15) must pass the secret-masking check. v2.1: the agent only ever sets Proposed Not Exploitable; TP and NEEDS_HUMAN get a comment only; reviewers are the central CISO analyst team.

## 1. Goal
Write the validated verdict back to on-prem CxSAST (believed 9.7) as a **proposed** triage state with a reason, appended to the existing per-result comment, and (where supported) assigned to a designated reviewer. Never destroy analyst content. Never touch the scan comment.

## 2. Verified facts and open items (from docs research; confirm in phase 0 against your server)
Confirmed in Checkmarx documentation:
- Each vulnerability instance has a **predicate** with `state`, `severity`, and `notes`; triage edits these.
- Recurrent results inherit predicate changes in later scans of the same project.
- Predicate changes apply only within that project.
- CxSAST can enforce **mandatory comments** when changing a state to Not Exploitable or Proposed Not Exploitable.
- `GET /sast/resultStates` lists result states (Bearer token with `sast_api` scope).
Not confirmed (phase 0 spike must settle):
- The exact REST endpoint and payload to update a result's state, severity, and notes in your 9.7 server (the public docs mention SOAP and REST variants; find the one your installation exposes).
- Whether the per-result notes API **overwrites** or appends (assume overwrite; implement append client-side).
- Whether **assignment** of a result to a user is possible through the API in 9.7. If not, use the fallback (section 6).
- Maximum length of notes.
- Behavior when state names are customized (custom states exist in some installs).

**Phase 0 spike checklist** (script and record results in `docs/cxsast-api-findings.md`):
1. Authenticate; list result states and store the name-to-ID map.
2. On a test project: read a result's predicate (state, severity, notes, assignee).
3. Update notes only; read back; confirm overwrite vs append behavior.
4. Update state to Proposed Not Exploitable with a comment; confirm mandatory comment enforcement.
5. Attempt assignment; record success or failure and the exact API.
6. Probe notes length limits (test 1k, 4k, 16k characters).
7. Confirm Recurrent inheritance by rescanning the test project.
8. Confirm concurrency behavior: two near-simultaneous updates to the same result.
9. Pull a sample of pattern-only results and confirm they accept the same predicate update (15).

## 3. Client wrapper
Implement a `CxSastClient` class exposing **only** the operations this system is allowed to perform:
- `list_result_states()`
- `get_result_predicate(scan_id, path_id)` -> state, severity, notes, assignee, last-modified if available
- `update_result_predicate(scan_id, path_id, state_id=None, severity=None, notes=None, assignee=None, expected_notes_hash=None)`
- read-only scan metadata methods (used by 01)
The wrapper must **not** expose any method that writes the scan comment (enforced structurally and by a CI test). Authentication via OAuth2 token with refresh; rate limiting and retry with backoff (honor 429/5xx); timeouts.

## 4. Append-only comment protocol

### 4.1 Agent block format
Each write appends one delimited block. Stable markers let the system recognize its own blocks:
```
<existing notes unchanged>

----- AI-TRIAGE v2 | 2026-10-31T15:04:00Z | finding <short-id> -----
Verdict: FALSE POSITIVE (proposed) | Confidence: 0.82 (calibrated, evidence: strong) | Severity: High (unchanged)
Reason: [category] User input is bound via PreparedStatement parameters and never concatenated; value context only.
Evidence:
 - src/main/java/.../OrderRepo.java:88-92 "ps.setString(1, name)"
 - src/main/java/.../OrderRepo.java:84 "prepareStatement(\"SELECT ... WHERE name = ?\")"
Checks: second opinion agreed (facts matched) | revision EXACT (a1b2c3d) | source match 1.00 | context L0 | flow check: NO_FLOW_FOUND (not relied upon)
Review: accept or overturn in CxSAST; reply here if overturning.
Run: <run-id> | models: <ids> | validator <v>
----- END AI-TRIAGE -----
```
Rules:
- Keep it concise and factual; the full reasoning lives in the internal store, with a run ID for lookup.
- Include revision confidence, `source_match_score` and context level; state clearly when the match is approximate.
- Never include secrets, raw prompts, or large code. Quote at most a few short lines.
- **Pattern findings (15):** the block describes the rule and the usage context in words (for example "value flows into DriverManager.getConnection in DbConfig.java"); the matched secret value is **never** written. Before any write, the masking check (08 section 4.5) must pass; a failure blocks the write (`masking_violation`).
- Plain text only; escape or strip characters that could break the notes field or be interpreted as markup by downstream tooling. The block is composed by deterministic code from validated fields; model text is limited to the short reason and sanitized quotes.
- Every block starts with `----- AI-TRIAGE` and ends with `----- END AI-TRIAGE -----` for reliable parsing. Version `v1` blocks written earlier remain valid for recognition.

### 4.2 Read-then-append procedure
1. `get_result_predicate` -> current notes, state, assignee. Compute `notes_hash`.
2. If a previous AI-TRIAGE block exists with the **same fingerprint and same verdict**, do not add a duplicate (idempotency); at most update nothing.
3. If a previous AI-TRIAGE block exists with a **different** verdict (re-triage after code change): append a new block marked `SUPERSEDES previous block`; do not edit earlier text.
4. Compose `new_notes = existing_notes + "\n\n" + block`. If it exceeds the maximum notes length: shorten the block (drop the Checks and Run lines first), then if still too long, write a minimal block and store the full text internally. Never truncate existing analyst notes.
5. Immediately before writing, **re-read** the predicate; if `notes_hash` changed (analyst edited meanwhile), recompute and retry (up to 3 times, with jitter). On repeated conflict: mark `WRITEBACK_FAILED(conflict)` and requeue later.
6. Write state, notes, and assignment in the fewest calls the API allows. If the API requires separate calls and one fails, record the partial outcome and reconcile on retry (idempotent: state and notes verified by re-read).
7. After writing, **verify by re-read**: notes end with our block, state equals intended. Record `writeback_verified = true/false`.

## 5. State mapping
Configurable mapping from validated verdict to Checkmarx state **name**, resolved to ID at runtime via the result states list (never hard-code IDs):

| Validated verdict | Phase 1 proposed state (default) |
|---|---|
| FP | Proposed Not Exploitable |
| TP | unchanged state, comment only |
| NEEDS_HUMAN | unchanged state, comment only |
| NEEDS_HUMAN_NO_ANALYSIS | unchanged state, comment only with the failure reason |

Rules:
- **Never overwrite a human-decided state.** If the current state is a human-set final state (Not Exploitable, Confirmed, Urgent, etc. set by a non-agent user), do not change the state; optionally append a comment only if the agent's verdict **disagrees** and policy says to notify (config `comment_on_disagreement`).
- Propose Not Exploitable only on findings still in an initial (untriaged) state (for example To Verify). The agent never sets any other state.
- If mandatory comments are enforced, the notes block satisfies it; verify that the call succeeds.
- Do not change severity in phase 1 (recommendation stays in the comment).
- Phase 3 auto-close (future): a separate, explicitly enabled code path that sets Not Exploitable only for `auto_close_eligible` findings; guarded by a feature flag, allowlist, and rate limits; includes an audit event. Pattern findings are excluded in v1.

## 6. Assignment
- Reviewers: the central CISO security analyst team works a single queue (no per-application mapping). The dashboard shows the queue split by verdict (TP, NEEDS_HUMAN, FP proposals).
- If the CxSAST API supports assignment: set it in the same update.
- **Fallback if assignment is unsupported:** include `Assigned reviewer: <name>` in the block, and expose per-reviewer queues in the dashboard (10). Optionally output a daily reviewer digest file or email feed in a later phase (not Jira, out of scope now).


## 7. Reviewer feedback capture
Determine reviewer outcomes by polling predicates for findings we proposed:
- State changed to Not Exploitable (by a human) after our Proposed Not Exploitable -> `accepted`.
- State changed to Confirmed, Urgent, or other TP-like state -> `overturned` (for an FP proposal).
- For TP proposals: reviewer set Not Exploitable -> `overturned`.
- Notes containing a reviewer reply marker (for example `[AI-TRIAGE-FEEDBACK] <text>`) -> store the text as reviewer comment, with an optional structured reason (sanitizer ineffective, input attacker-controlled, reachable, wrong code version, other).
- No decision after N days -> `pending` (dashboard shows aging).
Write these to `reviewer_outcomes` (08). Poll on a schedule with rate limits; also detect on subsequent scans (Recurrent inherits the human state).

## 8. Data model
```sql
CREATE TABLE writebacks (
  writeback_id UUID PRIMARY KEY,
  finding_id UUID, scan_id TEXT, project_id TEXT,
  intended_state TEXT, applied_state TEXT,
  assignee TEXT, assignment_method TEXT,   -- api | comment_only | none
  notes_block_hash TEXT, notes_length INT,
  status TEXT,   -- PENDING | WRITTEN | SKIPPED_HUMAN_DECIDED | FAILED
  failure_reason TEXT,
  verified BOOLEAN,
  attempts INT,
  written_at TIMESTAMPTZ
);
```
Failure reasons: `auth_failed`, `rate_limited`, `conflict`, `state_not_found`, `result_not_found` (scan deleted), `notes_too_long`, `assignment_unsupported`, `human_decided`, `masking_violation`, `api_error`.

## 9. Safety controls
- **Dry-run mode** (default on in lower environments and for the first production week): compute and store the exact payload and diff without calling write endpoints; dashboard shows dry-run counts.
- **Kill switch:** config flag halts all writes within seconds.
- **Rate limit** writes (config, default modest, for example 5 per second) with an adaptive slowdown on CxSAST latency.
- **Allowlist of projects** for early rollout (start with a few pilot applications), expanded gradually.
- Idempotent retries; all writes logged with before and after snapshots (notes hash, state).
- Alert when verification fails or when more than X% of writebacks fail within a window; page immediately on any `masking_violation`.

## 10. Acceptance criteria
- Append test: existing analyst notes are preserved byte for byte before the AI block, across 100 randomized note contents (including unicode, newlines, very long notes).
- Concurrency test: simulated analyst edit between read and write results in a retry that preserves both texts; no lost updates.
- Idempotency: re-running writeback for the same verdict adds no second block; changed verdict adds a `SUPERSEDES` block.
- Human-decided states are never changed (table-driven tests across the states).
- Mandatory-comment mode: state change succeeds with the block; fails gracefully with a recorded reason otherwise.
- Notes length handling: block shortening and minimal block fallback; existing text never truncated.
- Structural test: no code path calls any scan-comment write endpoint (client wrapper exposes none; CI grep guard).
- Dry-run produces payloads identical to real mode and makes zero write calls.
- Kill switch halts writes within the configured time in an integration test.
- Masking test: a pattern finding whose block would contain the matched value is blocked (`masking_violation`) and nothing is written.
- Reviewer outcome detection: accepted, overturned, and feedback-marker cases covered by tests against a mock server.
