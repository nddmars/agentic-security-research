# 15: Pattern Findings Triage (Checkmarx results with no source-to-sink path)

Author: Surya

Read `00-MAIN-DESIGN.md` first. Depends on 02, 03, 07, 08, 09. Reuses the Evidence Pack, agent runtime, validator and writeback.

v2.1 scope decision: this system **does not run its own pattern scan**. Checkmarx already runs the regex and filename rules (hardcoded usernames, suspicious files, suspicious URLs, and similar). This feature only triages what Checkmarx reports: it reads the code around each hit and judges whether the match is a real problem or harmless. The independent scanner, its rule library, the pre-filter and the filename inventory were removed from the design.

## 1. Goal
Triage Checkmarx findings that have **no stack trace or source-to-sink path** because they come from pattern or regex queries. The question is not "does tainted data reach a sink" but "is this match suspicious in its usage context, or benign".

These findings enter through the normal ingestion (01) with `finding_kind = pattern_only`, travel through the same source acquisition (02), and are written back with the same rules as every other finding (09): the agent proposes Not Exploitable for benign matches and adds a comment only for everything else.

## 2. Scope (families)
Families are derived from Checkmarx query names and are configuration:
- `hardcoded_credential_username`: hardcoded usernames and credential-like literals (passwords, tokens and keys fall in the same family and are always masked).
- `suspicious_file`: Checkmarx results that flag a file (for example key stores, `.env`, backups, dumps).
- `suspicious_url`: hardcoded URLs and IP addresses.
Add a family by adding the Checkmarx query names that map to it and a decision table (section 6). Unknown pattern queries use a `generic_pattern` table that leans `NEEDS_HUMAN`.

Out of scope: running regex scans of our own, live-key validation, and any action that contacts a suspicious host (never fetch or resolve a URL found in code).

## 3. Inputs: what Checkmarx gives us
- The result has a file and usually a line (and column/length when available), the matched snippet, query name, CWE, severity, state and existing remark. There is no path.
- **If Checkmarx gives only a file (no line):** locate the match by re-applying a family-specific search (for example ripgrep for username-like assignments) inside that file. If one candidate line is found, use it; if several, keep them all as `match` nodes; if none, end as `NEEDS_HUMAN_NO_ANALYSIS` with reason `no_match_location`.
- Verify the match text against the file from the zip (the same source-match check as 02 section 5.3, applied to the matched line).
- The matched snippet is **masked at ingestion** when the query is in a secrets-like family (01 section 4.4). Only a value fingerprint (hash) is kept for dedupe.

## 4. Light pre-processing (kept deliberately small)
- **Dedupe:** identical normalized literal (by value fingerprint) in the same project and rule are judged once; occurrences are listed (06 fingerprint handles reuse).
- **Placeholders:** values on a short config allowlist (`changeme`, `example.com`, documentation IP ranges, `YOUR_API_KEY`) are labeled `placeholder_or_example` as a **hint**, not a final classification; the decision table and validator still apply. There is no separate pre-filter stage and no filtered inventory.
- Test, generated and vendored files are classified by path rules (02) as for every other finding.

## 5. Building the Evidence Pack for a pattern finding
Same pack schema as 03 section 6, with `snippets[kind = match]` and these additions.

1. **Match snippet:** about 20 lines around the match (or the enclosing function if it fits), with the secret value **masked** (`****`), keeping variable names and structure.
2. **File class and location:** production, test, generated, vendored, docs, config; directory context; whether the file looks deployed (for example under `src/main`) when cheaply known.
3. **Usage context (Joern, when a CPG exists):** `literal_usage` (03 section 3) answers:
   - Which identifier or literal node sits at the file and line?
   - Where does the value flow? Into an auth or login call, a database connection string, an HTTP client, `exec`, a file write, a log statement, a returned value, or nowhere.
   - Is it a constant, a config default, a test fixture, or a build script?
   - Who calls the enclosing method, and is it reachable from an entry point (asymmetry rule applies as usual)?
   Result: `usage.sinks[]` with sink kind, snippet IDs and precision. Without a CPG (`context_level` L1): ripgrep for other occurrences of the identifier in the same file or module (`name_based`), and `usage.sinks` is `unknown`.
4. **Neighboring facts:** config file origin (is the value also in a deployed config?), comments, nearby variable names, other occurrences of the same literal (count and file classes).
5. **File findings:** path, size, extension, directory, whether other files reference it (ripgrep for the file name), and for key or certificate formats a **header-only** check (first bytes such as a private-key header versus a public key or certificate header). The file content is never sent to a model.
6. **URL findings:** scheme, host class (internal pattern, public domain, raw IP, private range, documentation range), path shape, and usage (HTTP client call, config default, comment, test, documentation). **The URL is never fetched or resolved.**

Keep the Checkmarx hit as a **fact** (query, matched span, family) and the model's judgment as a separate field.

## 6. Decision tables (pattern families)
The model fills facts; a deterministic table derives the allowed verdict (same mechanism as 07 section 5). Verdict labels reuse `TP` (confirmed issue), `FP` (benign), `NEEDS_HUMAN`. Remember the writeback rule: only `FP` produces a state change (Proposed Not Exploitable); `TP` and `NEEDS_HUMAN` get a comment only.

Facts for pattern families:
```json
{
  "facts": {
    "literal_nature": "real_credential_likely | placeholder_or_example | identifier_not_credential | unknown",
    "usage": {"sink": "auth_call|db_connection|http_client_auth|exec|file_write|log|return_value|none_found|unknown", "cites": ["S2"]},
    "code_class": "prod|test|generated|vendored|docs",
    "deployed_context": "yes|no|unknown",
    "reachability": {"status": "reachable|unreachable_sound|unknown", "cites": []},
    "injection_indicators": []
  },
  "verdict": "TP | FP | NEEDS_HUMAN",
  "evidence_strength": "strong | moderate | weak"
}
```

### 6.1 `hardcoded_credential_username`

| # | literal_nature | usage.sink | code_class | deployed_context | Allowed verdict |
|---|---|---|---|---|---|
| 1 | real_credential_likely | auth_call / db_connection / http_client_auth | prod | yes / unknown | `TP` |
| 2 | real_credential_likely | any | prod | * | `TP` or `NEEDS_HUMAN` (never FP) |
| 3 | placeholder_or_example | none_found / log / return_value | any | * | `FP` allowed, evidence = the placeholder token in code |
| 4 | real_credential_likely | auth_call / db_connection | test | no | `NEEDS_HUMAN` (check it is not reused in prod) |
| 5 | identifier_not_credential (role name, display label) | none_found | prod | * | `FP` allowed only with the sink verified as non-auth |
| 6 | any | unknown | * | * | `NEEDS_HUMAN` |
| 7 | any | any | * | * | `injection_indicators` non-empty -> no FP |

`never_fp_if`: usage.sink is `auth_call`, `db_connection` or `http_client_auth` and `deployed_context != no`; the same literal also appears in a deployed config; the literal looks like a real credential and is in production code.

### 6.2 `suspicious_file`

| # | file type | header/metadata check | tracked in repo | Allowed verdict |
|---|---|---|---|---|
| 1 | private key / keystore (`id_rsa`, `.pfx`, `.p12`, `.jks`, `.kdbx`) | private key header or unknown | yes | `TP` |
| 2 | `.pem` | public key / certificate header only | yes | `FP` allowed (public material), evidence = header type |
| 3 | `.env` / credentials file | not a template | yes | `TP` or `NEEDS_HUMAN` |
| 4 | `.env.example` / sample | template | yes | `FP` allowed |
| 5 | database dump / backup (`*.sql`, `*.bak`) | large or contains data statements | yes | `NEEDS_HUMAN` or `TP` |
| 6 | any | test fixture directory and tiny size | yes | `NEEDS_HUMAN` unless the header shows non-secret |
| 7 | any | unknown | * | `NEEDS_HUMAN` |

The model sees names, sizes, directory context and the header classification, never the file content.

### 6.3 `suspicious_url`

| # | host class | usage | code_class | Allowed verdict |
|---|---|---|---|---|
| 1 | raw IP / paste or file-sharing / shortener / non-allowlisted external | http_client call or config default | prod | `TP` |
| 2 | non-allowlisted external | comment / docs / test only | docs / test | `FP` allowed |
| 3 | documentation or placeholder (`example.com`, RFC 5737) | any | any | `FP` allowed |
| 4 | internal allowlisted host | any | any | `FP` allowed (`allowlisted`) |
| 5 | external, `http://` (not TLS) | http_client call | prod | `TP` or `NEEDS_HUMAN` |
| 6 | any | unknown usage | * | `NEEDS_HUMAN` |

The host allowlist (approved internal and vendor hosts) is configuration owned by AppSec; changes go through pull requests.

## 7. Agent run (reuse 07)
- Same runtime and prompt layout (static prefix first), single-pass default, bounded gap-fill (max 3 calls for pattern families), Evidence Pack citation rules, injection indicators, failure handling.
- Playbook = the family's decision table. Fact-extraction accuracy is what to test, not free-form judgment.
- **Masking:** the matched value is replaced with `****` before the pack is built; models judge usage context, not the secret. If judging needs a property of the value (for example generic username versus real account name), pass a deterministically computed classification (`looks_generic`, length, character classes), not the value.
- Cheap tier handles most pattern hits (short context). Escalate on uncertainty or for `TP` candidates in production code.

## 8. Validator rules (08 section 4.5)
All of 08 applies (schema, snippet-ID evidence, decision-table derivation, injection check, source verification on the matched line). Extra:
- **Masking check** on every produced text (comment, log, dashboard field): the matched value must not appear. A violation blocks writeback (`masking_violation`).
- `FP` requires the table's benign rows with verified evidence; `literal_nature = unknown` or `usage.sink = unknown` -> `NEEDS_HUMAN`.
- Pattern findings are never `auto_close_eligible` in v1.
- Second opinion applies to High severity as for SAST (08 section 4.3).

## 9. Writeback and reporting
- Standard writeback (09): `FP` -> Proposed Not Exploitable with the comment block; `TP` and `NEEDS_HUMAN` -> comment only, state unchanged. The comment names the Checkmarx query, the usage context in words (for example "value flows into DriverManager.getConnection in DbConfig.java"), the verdict and the reason; never the secret.
- Reviewers are the central CISO security analyst team (09 section 6).
- Dashboard (10): pattern findings by family, verdict, file class and usage-context coverage (known sink versus unknown); masking-check failures (should be zero).

## 10. Data model
No new tables beyond the common finding tables (01, 06, 08, 09). Pattern-specific data lives in `findings` (with `finding_kind = pattern_only`), `finding_path_nodes` (role `match`), `finding_context` (pack URI and hash), and a small table for the dedupe fingerprint:
```sql
CREATE TABLE pattern_value_fingerprints (
  fingerprint TEXT PRIMARY KEY,            -- HMAC of the normalized literal; never the value
  family TEXT, first_finding_id UUID,
  occurrence_count INT, updated_at TIMESTAMPTZ
);
```

## 11. Cautions
- **Secrets in prompts and logs:** mask at ingestion and again in the pack builder; store only value fingerprints.
- **Precision:** a ripgrep hit is name-based. The flow from a literal in Joern is only as good as the language frontend; keep precision tags, and "no sink found" never proves benign (asymmetry rule).
- **Never fetch** a suspicious URL, open a suspicious file, or decode embedded blobs outside the sandbox.

## 12. Acceptance criteria
- Fact-extraction fixtures: test credential versus production credential flowing into a login call, generic role name versus account name, public key versus private key header, documentation URL versus HTTP client URL, each yields the expected facts and allowed verdict.
- Masking tests: planted secrets never appear in prompts, logs, dashboard or Checkmarx comments; a masking violation blocks writeback.
- A Checkmarx result with only a file (no line) is located by the family search, or ends `NEEDS_HUMAN_NO_ANALYSIS` with reason `no_match_location`.
- Placeholder hints do not bypass the decision table or the validator.
- Dedupe: the same normalized literal in the same project and family is triaged once.
- Writeback behavior: only `FP` changes state; `TP` and `NEEDS_HUMAN` leave state unchanged and add a comment.
