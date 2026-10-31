# 02: Source Acquisition

Author: Surya

Read `00-MAIN-DESIGN.md` first. Depends on `01-ingestion.md`.

## 1. Goal
For each scan, determine where the scanned source lives from the **scan comment** (read only), fetch exactly the scanned revision when possible, filter it down to analyzable source, **verify it against what Checkmarx saw**, and produce a workspace the Code Intelligence Service can index. Record exactly what was fetched so every verdict is auditable.

The quality of every downstream verdict is capped by what this feature can prove about the code.

## 2. Inputs
- `scans.scan_comment_raw` (read only)
- `scans.scan_started_at`, `scans.scan_finished_at`, scan type (full or incremental) from CxSAST scan metadata
- CxSAST project source-origin settings (Git URL and branch, if exposed by the API), used for cross-checking (section 5.4)
- Credentials for source hosts (read-only, from the secret manager)
- Config: host allowlist, size limits, filter rules, workspace root, cache policy

## 3. Outputs
- A `source_snapshots` record per (project, scan) with: resolved URL, host, revision type, revision value (commit SHA if known), content hash, size before and after filtering, `revision_confidence`, and `origin_conflict` flag.
- A per-finding `source_match_score` (section 5.3), written to the finding context and into the Evidence Pack (03).
- A workspace directory (or object store prefix) with filtered source, mounted read-only for analysis workers.

## 4. Scan comment parsing

The scan comment contains a network/HTTP-based location of the scanned source. Formats will vary across 1,500 projects. Build a **pluggable parser chain**; each parser returns a candidate or nothing, and the first confident match wins. Log which parser matched.

Parsers to implement:
1. Git URL (https or ssh form) with optional branch and optional commit SHA, in free text. Extract with regexes; accept `branch`, `ref`, `commit`, `sha`, `rev` keywords. A 40-hex (or 64-hex) token near a git URL is a candidate SHA.
2. Web UI URLs (GitHub, GitLab, Bitbucket, Azure DevOps style) including `/tree/<ref>`, `/commit/<sha>`, `/-/tree/`, and similar; convert to clone URL plus ref.
3. **Build URL / archive (primary in phase 1):** the scan comment holds a build URL that downloads a zip of the scanned code (also `.tar.gz`, `.tgz`). Formats vary and are surveyed in 14 phase 0.
4. Key-value or JSON in the comment (for example `{"repo": "...", "branch": "...", "commit": "..."}` or `repo=... commit=...`).
5. Fallback: no parseable location. Result: `source_unavailable`, findings go to `NEEDS_HUMAN_NO_ANALYSIS` with reason `no_source_location`.

**Decision (v2.1):** archive-first. Checkmarx runs in the pipeline and the comment gives a build URL for the zip. Parser 3 and section 6.2 are the primary path. Git parsers 1, 2 and 4 stay implemented but Git fetch is off in phase 1; build-to-repository mapping and Git history are phase 2 (17). Revision pinning relies on the archive hash and `source_match_score`, not commit SHAs.

Rules:
- Treat the comment as **untrusted free text**. Never execute or shell-interpolate any part of it. Use argument arrays for subprocess calls and never build shell strings.
- **Host allowlist:** only fetch from hosts in config. A URL on any other host is rejected with reason `host_not_allowed`. Reject IP-address hosts, `file://`, `localhost`, and link-local ranges (SSRF defense). Resolve DNS and re-check the resolved IP against private-range rules unless the internal host is explicitly allowlisted.
- Redact credentials embedded in URLs before logging or storing (`https://user:token@host/...`). Store only the sanitized URL.
- Support multiple candidate URLs in one comment: pick the first that is allowlisted; record alternates.

## 5. Revision resolution, confidence and verification

### 5.1 Anchor time (corrected from v1)
CxSAST takes the source at **scan start**, not at scan finish. All time-based branch resolution therefore uses `scan_started_at` (if CxSAST exposes the source-pull time, use that). `scan_finished_at` is **not** used. In phase 1 (archive-first) no branch resolution happens; this anchor applies to the phase 2 Git path (17).

### 5.2 Revision confidence
`revision_confidence`:
- `EXACT`: a commit SHA from the comment was checked out and exists; or, for an archive, the per-snapshot source-match score is at least 0.99 or a scan-time content hash is known.
- `HIGH`: a tag, or a branch whose HEAD commit date is on or before the anchor time (phase 2); **or an archive from the build URL by default** (the pipeline produced it for this scan). The artifact may change or expire, so `source_match_score` stays the verifier.
- `APPROXIMATE`: branch HEAD fetched with no SHA. Pick the latest commit on that branch at or before the anchor time using `git rev-list -1 --before=<anchor_time> <ref>` (requires enough history; deepen the clone as needed). Record the chosen SHA.
- `UNKNOWN`: could not determine the relation (including archives with unknown provenance).

Every verdict stores the `source_snapshot_id`. If confidence is `APPROXIMATE` or `UNKNOWN`:
- The comment block written to Checkmarx states that the match is approximate.
- The validator (08) caps confidence and forbids auto-close eligibility.

### 5.3 Per-finding `source_match_score`
Even with an `EXACT` SHA, something can be wrong (different branch, a pre-build step that rewrote files, line-ending conversion). So every finding is verified against the code, not only the revision.

Procedure, per finding:
1. For each Checkmarx path node (file, line, column, name, snippet), locate the file in the snapshot (03 section 4 handles path matching and column conversion).
2. Compare the node snippet and name with the text at the mapped location (whitespace-normalized). Match types: `verified` (exact at position), `searched` (found within plus or minus 3 lines), `missing`.
3. `source_match_score = (verified + 0.5 * searched) / total_nodes`.
4. Store the score and the per-node results.

Effects (enforced in 08 section 3.5):
- Score below `source_match_min` (default 0.8): flag `source_mismatch`, route to `NEEDS_HUMAN`, and downgrade the snapshot's revision confidence if many findings in the same snapshot fail.
- Score at least 0.8 with `APPROXIMATE` revision: the finding may be upgraded to `HIGH` for that finding only, when all nodes are `verified`.
- Any FP verdict requires `source_match_score >= source_match_fp_min` (default 0.95).

For `pattern_only` findings (15) the check is the matched line text instead of path nodes.

### 5.4 Cross-check against CxSAST metadata
If the CxSAST project settings expose a source origin (Git URL and branch), compare it with the URL and branch parsed from the scan comment:
- Agree: record `origin_conflict = false`.
- Conflict (different repo or branch): set `origin_conflict = true`, flag the snapshot, and cap `revision_confidence` at `APPROXIMATE`. Do not choose one silently; record both.
- Metadata unavailable: record `origin_check = unavailable`.

### 5.5 Incremental scans
Detect scan type from CxSAST scan metadata. For an incremental scan, results for unchanged files may have been carried over from an earlier scan, so a finding's file can differ from the revision of the current scan.
- For findings in files not changed in this scan, resolve the revision **per file** from the scan in which the result was last produced (`first_seen_scan_id` or the previous full scan), not from the current scan.
- If the per-file revision cannot be determined, mark the finding `APPROXIMATE`.
- `source_match_score` is the final arbiter: a mismatch always wins over an assumed revision.

## 6. Fetching

### 6.1 Git (phase 2, off by default)
- Prefer shallow, partial clones: `git clone --filter=blob:none --no-checkout` then `git fetch --depth=N` of the target SHA, then checkout. If the SHA is not reachable at shallow depth, incrementally deepen (for example 50, 500, then full) up to a config limit.
- Use sparse checkout for large repos and monorepos: first fetch the tree, then check out only directories containing finding files plus their module roots and dependency manifests (see section 7).
- Do not fetch LFS objects by default (`GIT_LFS_SKIP_SMUDGE=1`).
- Submodules: off by default; config flag per host. If findings reference files under a submodule path, fetch that submodule shallowly if its host is allowlisted; otherwise mark `submodule_unavailable`.
- Timeouts and size ceilings per clone; on exceed, abort and mark `source_fetch_failed` with reason.

### 6.2 Archive over HTTP(S) (primary in phase 1)
- Download with size ceiling and timeout; stream to disk; compute SHA-256 while downloading.
- Safe extraction: reject path traversal entries (`..`, absolute paths), symlinks pointing outside the extraction root, device files, and zip/tar bombs (limit total uncompressed size and file count; limit compression ratio).
- Record the content hash as the revision value.
- Archives may contain build output or a different root folder than the Checkmarx paths; 03 section 4 handles root-prefix matching, and `source_match_score` verifies the result.

### 6.3 Caching
- Cache key: (host, repo path, revision SHA) for git; (content hash) for archives. A snapshot is reusable by any scan that resolves to the same key.
- Evict by LRU with a size budget. Never evict a snapshot with in-flight analysis jobs.

## 7. Filtering (critical for 12 GB repos)

Most of a very large repo is not source. Filter before indexing.

- **Exclude directories by default:** `node_modules`, `vendor` (but keep Go `vendor` if findings point into it; make this config), `bin`, `obj`, `target`, `build`, `dist`, `out`, `.git`, `.gradle`, `.m2`, `__pycache__`, `.venv`, `venv`, `site-packages`, `.idea`, `.vs`, `packages` (NuGet), `coverage`, generated asset folders. Make the list configurable per language.
- **Exclude by file type:** images, video, audio, fonts, archives, binaries, `.class`, `.jar`, `.dll`, `.exe`, `.so`, large data files (size cutoff, for example above 2 MB for non-source extensions).
- **Always include (never filtered):** every file referenced by a finding's path nodes; build and dependency manifests (`pom.xml`, `build.gradle*`, `*.csproj`, `*.sln`, `packages.config`, `package.json`, lockfiles, `requirements*.txt`, `pyproject.toml`, `setup.py`, `go.mod`, `go.sum`); application config (`application*.yml`, `application*.properties`, `web.config`, `appsettings*.json`, `*.env.example`, framework config); route/DI/security config files; the **module directories** containing finding files and their direct local dependencies.
- File classification (store per file): `source`, `test`, `generated`, `vendored`, `config`, `other`. Heuristics: path segments (`test`, `tests`, `__tests__`, `spec`, `src/test`), file suffixes (`_test.go`, `.spec.ts`, `Test.java`, `Tests.cs`), generated markers (`// <auto-generated>`, `@Generated`, `DO NOT EDIT`, `.g.cs`, `.pb.go`, `_pb2.py`), and vendored paths. This is plain path and header logic; no parser is needed.
- Record `size_before_bytes`, `size_after_bytes`, `excluded_summary` (counts per reason).
- If after filtering the source still exceeds the tier limits from 06, do not fail: pass a `module_scope` (the set of module roots to index) to 03.

## 8. Module discovery (for sharded indexing)
- Java: Maven modules (`pom.xml` `<modules>`), Gradle `settings.gradle*` includes.
- C#: `.sln` projects, `.csproj` references.
- Node: `package.json` workspaces, nearest `package.json` ancestor.
- Python: nearest ancestor with `pyproject.toml`, `setup.py`, or `requirements.txt`; fall back to top-level package directories.
- Go: `go.mod` roots and `go.work`.
- Output a `modules` table: module root, language, manifest path, and local dependency edges. 03 uses it to scope CPG builds.

## 9. Data model

```sql
CREATE TABLE source_snapshots (
  snapshot_id UUID PRIMARY KEY,
  scan_id TEXT NOT NULL,
  project_id TEXT NOT NULL,
  parser_used TEXT,
  resolved_url_sanitized TEXT,
  host TEXT,
  fetch_kind TEXT,            -- git | archive
  revision_value TEXT,        -- commit SHA or content hash
  revision_confidence TEXT,   -- EXACT | HIGH | APPROXIMATE | UNKNOWN
  anchor_time TIMESTAMPTZ,    -- scan start (or source-pull time if known)
  scan_type TEXT,             -- full | incremental | unknown
  origin_check TEXT,          -- agree | conflict | unavailable
  origin_conflict BOOLEAN,
  alternates JSONB,
  workspace_path TEXT,
  size_before_bytes BIGINT,
  size_after_bytes BIGINT,
  excluded_summary JSONB,
  status TEXT,                -- READY | FAILED | UNAVAILABLE
  failure_reason TEXT,
  created_at TIMESTAMPTZ
);

CREATE TABLE source_files (
  snapshot_id UUID REFERENCES source_snapshots(snapshot_id),
  path TEXT,
  classification TEXT,
  size_bytes BIGINT,
  sha256 TEXT,
  indexed BOOLEAN,            -- false for inventory-only files
  PRIMARY KEY (snapshot_id, path)
);

CREATE TABLE modules (
  snapshot_id UUID REFERENCES source_snapshots(snapshot_id),
  module_root TEXT,
  language TEXT,
  manifest_path TEXT,
  local_deps JSONB,
  PRIMARY KEY (snapshot_id, module_root)
);

CREATE TABLE finding_source_match (
  finding_id UUID PRIMARY KEY,
  snapshot_id UUID,
  source_match_score REAL,
  node_results JSONB,         -- per node: verified | searched | missing
  per_file_revision JSONB,    -- for incremental scans
  computed_at TIMESTAMPTZ
);
```

## 10. Failure reasons (enumerated; shown on dashboard)
`no_source_location`, `host_not_allowed`, `auth_failed`, `repo_not_found`, `revision_not_found`, `fetch_timeout`, `size_limit_exceeded`, `archive_unsafe`, `artifact_expired`, `submodule_unavailable`, `source_mismatch`, `origin_conflict`, `unknown_error`.
All map findings to `NEEDS_HUMAN_NO_ANALYSIS` with the reason attached. (`origin_conflict` alone does not block analysis; it caps confidence.)

## 11. Security requirements
- Fetch workers run in isolated pods with network egress limited to allowlisted source hosts and no access to internal services beyond those (OpenShift NetworkPolicy, see 13).
- Read-only credentials, short-lived tokens where the host supports them.
- Never run repository scripts, hooks, build commands, or package installs in this stage. `git` config: `core.hooksPath=/dev/null`, `protocol.file.allow=never`, disable `core.fsmonitor`, no `--recurse-submodules` by default.
- Images must not assume a fixed UID: OpenShift runs pods with a random non-root UID. Workspace directories must be group-writable for GID 0 and created with `umask 002`.
- Mount workspaces read-only into downstream workers.

## 12. Acceptance criteria
- Parser chain unit tests with at least 30 realistic comment samples, including: URL only, URL plus branch, URL plus SHA, web-UI links, archive links, multi-URL comments, credentials embedded, garbage text, and hostile inputs (`;` and backticks and `$()`), each producing the expected outcome and never executing anything.
- SSRF tests: `http://169.254.169.254/`, `http://localhost`, `http://10.0.0.5`, DNS names that resolve to private IPs, and redirect chains to disallowed hosts are all rejected.
- A 12 GB synthetic repo (mostly `node_modules`, binaries, and media) filters to a small source set without exceeding memory limits, and all files referenced by findings are retained.
- Archive bomb and path-traversal archives are rejected safely.
- Revision confidence is computed and stored for all four classes; tests cover each.
- **Anchor-time test:** a branch with a commit made after scan start and before scan finish resolves to the commit **before** scan start.
- **Source-match tests:** exact SHA with matching snippets scores 1.0; exact SHA with rewritten files scores low and flags `source_mismatch`; CRLF and tab/multibyte fixtures still match.
- **Incremental scan test:** a result carried over from an earlier scan resolves its file to the earlier revision.
- **Origin conflict test:** comment URL differs from CxSAST metadata, so the conflict flag is set and confidence is capped.
- Snapshot cache hit for two scans resolving to the same SHA performs no second fetch.
