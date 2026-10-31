# 17: Phase 2 Requirements: Git Mapping, Introducing Commit and Ownership

Author: Surya

Read `00-MAIN-DESIGN.md` and `02-source-acquisition.md` first. This file is a **requirements document for phase 2**. Nothing here is built in phase 1. Phase 1 is archive-first: the scan comment holds a build URL that downloads a zip of the scanned code, and triage needs no Git history.

## 1. Goal
After a finding is triaged (especially confirmed or likely true positives), help remediation by answering:
1. Which repository and commit did the scanned build come from?
2. Which commit introduced the vulnerable code?
3. Which team owns that code, and (only if policy allows) who committed it?

This also lets the system keep the history of when vulnerable code entered a codebase, for metrics such as time-to-introduction and time-to-fix.

## 2. Why this is phase 2
- Triage works from the zip alone: the zip is the code Checkmarx scanned, so no commit-guessing is needed for verdicts (02).
- The build-to-repository mapping does not exist yet and must be found or built.
- Attributing code to individuals has privacy, HR and works-council implications that need a leadership decision, not only an engineering one.

## 3. Requirements

### 3.1 Build-to-repository mapping
- Given a build URL (the one in the scan comment), resolve: repository URL, branch or ref, commit SHA, and build ID.
- Sources to evaluate (phase 2 discovery): the CI system's build API or build metadata (most reliable), artifact metadata or manifest files inside the zip (for example a build-info file, `git.properties`, SBOM, version stamps), naming conventions that link a Checkmarx project to a repository, a maintained mapping table (application ID to repository), and CI webhook records.
- Output a `build_repo_map` record with `confidence` (`EXACT` when the CI system reports the commit SHA; `INFERRED` when derived from naming or a table; `UNKNOWN`), the evidence used, and the retrieval time.
- Validate the mapping by checking that the files in the zip match the repository at that commit (reuse the `source_match` idea from 02 with a git tree comparison). A mapping that fails validation is discarded.
- Read-only credentials, host allowlist, SSRF defenses and sandboxing as in 02 and 12.

### 3.2 Introducing commit
- For each triaged finding (start with `TP` and `NEEDS_HUMAN` in High and Critical), locate the commit that **introduced** the vulnerable lines:
  - `git blame` on the sink (and source) lines at the mapped commit, following renames and moves (`-M -C`), then walk back with `git log -L` or a pickaxe search (`-S` / `-G`) to find where the vulnerable construct first appeared.
  - Handle large formatting-only commits, merges, squashes and vendored-code imports (record `introduced_by = import|reformat|merge` and flag low confidence instead of blaming the wrong commit).
- Output: introducing commit SHA, commit date, merge or pull request reference if available, and a `confidence` plus caveats.
- Cap work per finding (time and history depth); on exceed record `history_unavailable` with the reason.

### 3.3 Ownership and attribution (policy gated)
Default mode is **team-level**:
- Resolve the owning team through CODEOWNERS, repository ownership metadata, or the application-to-team mapping, and route remediation to that team.
Individual-level attribution is **off by default** and sits behind an explicit policy gate:
- Gate: `attribution.individual_names = false` unless HR, Legal, Privacy and CISO leadership approve in writing, including purpose limits, access control, retention and how developers are informed.
- If enabled: store the committer identifier as a pseudonymous ID by default, resolve to a name only for authorized roles on request, log every lookup, and exclude it from exports and the shared dashboard.
- Never use the data to rank, score or evaluate individuals. Metrics are aggregated at team or application level.
- Respect local regulation (for example data protection and employment law in each jurisdiction where developers work).

### 3.4 Developer or team history views (also policy gated)
- Team-level history: how often vulnerability classes are introduced per team over time, mean time from introduction to detection and to fix.
- Individual-level history is **not** part of phase 2 unless the policy gate in 3.3 approves it separately.

### 3.5 Where it plugs into the design
- 02: a new fetch path `git_fetch` (already described there and disabled by default) used only for mapping validation and history, not for triage input.
- 03/07/08: no change to verdicts; Git data is never fed into the verdict decision in phase 2, to avoid bias and extra attack surface. It enriches the comment and the dashboard only.
- 09: the comment block may add a line such as `Introduced: <short sha>, <date>, owning team: <team>` (no individual names unless the gate is on).
- 10: dashboard adds team-level introduction and fix-time views.
- 12: Git hosts join the egress allowlist; commit messages and author fields are treated as untrusted text (prompt injection) and are never sent to models in phase 2.
- 13: history jobs run as a separate low-priority queue with disk-bounded scratch.

## 4. Data model (draft)
```sql
CREATE TABLE build_repo_map (
  build_id TEXT, scan_id TEXT, project_id TEXT,
  repo_url_sanitized TEXT, ref TEXT, commit_sha TEXT,
  confidence TEXT,                 -- EXACT | INFERRED | UNKNOWN
  evidence JSONB, validated BOOLEAN, retrieved_at TIMESTAMPTZ,
  PRIMARY KEY (build_id)
);
CREATE TABLE finding_history (
  finding_id UUID PRIMARY KEY,
  introducing_commit TEXT, introduced_at TIMESTAMPTZ,
  introduced_by TEXT,              -- change | import | reformat | merge | unknown
  confidence TEXT, caveats JSONB,
  owning_team TEXT,
  committer_pseudonym TEXT,        -- null unless the policy gate is enabled
  status TEXT, failure_reason TEXT, computed_at TIMESTAMPTZ
);
```

## 5. Libraries (proposals; verify)
System `git` through `subprocess` with argument arrays (hooks disabled, no shell); `pathspec` for CODEOWNERS parsing or a small dedicated parser; `httpx` for CI system APIs; the existing `SQLAlchemy`/`psycopg` stack. Avoid in-process Git libraries to keep the sandbox surface small.

## 6. Open questions for phase 2
1. Which CI system(s) trigger the Checkmarx scans, and do they expose build metadata (repository, ref, commit) by API?
2. Does the zip contain build-info files that carry the commit SHA?
3. Is there an existing application-to-repository mapping, or does one need to be built?
4. Who decides the attribution policy (HR, Legal, Privacy, CISO), and what approval is required?
5. Is CODEOWNERS (or equivalent ownership metadata) maintained across repositories?
6. How long are build artifacts and Git history retained?

## 7. Acceptance criteria (phase 2)
- Mapping validated on a sample of at least 100 builds: share EXACT, share INFERRED, share unresolved; mapping errors detected by the tree comparison are 0 in accepted records.
- Introducing-commit accuracy measured against a hand-labeled sample; formatting-only, import and merge cases are flagged, not mis-attributed.
- With the policy gate off, no individual identifier is stored, displayed, exported or logged; verified by tests.
- With the gate on (test environment only), lookups are role-restricted and audited.
- No Git-derived text (commit messages, author names) reaches a model prompt.
- History jobs respect time, depth and disk limits and fail with `history_unavailable`, never blocking triage or writeback.
