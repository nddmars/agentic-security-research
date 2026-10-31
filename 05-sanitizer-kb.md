# 05: Sanitizer and Safe-Pattern Knowledge Base

Author: Surya

Read `00-MAIN-DESIGN.md` first. Depends on `03-code-intelligence.md`; feeds `07-triage-agent.md`.

## 1. Goal
The largest source of SAST false positives is a path through a sanitizer, validator, or framework protection that Checkmarx does not recognize. This feature provides (a) a curated, versioned knowledge base of known-good and known-insufficient sanitizers per language and sink context, and (b) a procedure for evaluating **in-repo custom sanitizers**. It is the key tool for reducing false positives **safely**.

Core rule: a sanitizer only counts if its protection **matches the sink context** and is applied on **all** relevant branches. HTML encoding does not stop SQL injection. URL encoding does not stop command injection. A regex blocklist is not a sanitizer.

## 2. Knowledge base format

Plain YAML files under `sanitizers/<language>/<family>.yml`, versioned in git, owned by AppSec analysts, loaded by the service. No rule engine required; matching is by **symbol resolution through the Joern call graph** (03) and, where no CPG exists, by fully qualified name lookup on the call text (tagged `name_based`).

Entry schema:
```yaml
- id: java.sqli.preparedstatement-params
  language: java
  family: sql_injection        # vulnerability family
  kind: safe_api               # safe_api | encoder | validator | allowlist | framework_protection | insufficient
  match:
    symbols:                   # fully qualified, resolved via the call graph
      - java.sql.PreparedStatement.setString
      - java.sql.PreparedStatement.setInt
  requires:                    # conditions that must hold for the protection to apply
    - "query string passed to prepareStatement is a compile-time constant or contains only ? placeholders"
  protects_sink_contexts: [sql_value]
  does_not_protect: [sql_identifier, sql_order_by, sql_fragment]
  bypass_conditions:
    - "user input concatenated into the SQL string before prepareStatement"
  confidence: high             # high | medium | low
  references: ["https://cheatsheetseries.owasp.org/..."]
  notes: "Parameterization protects values only, not table/column names or ORDER BY."
  added_by: analyst-id
  reviewed_at: 2026-10-30
  version: 1
```

`kind: insufficient` entries are important: they list APIs that **look** like sanitizers but do not protect (for example `String.replace("'", "")` for SQLi, `HttpUtility.UrlEncode` for HTML-attribute contexts, `escape()` used in the wrong context). The agent must be told when a claimed sanitizer is known-insufficient.

The KB is also the input to the **second flow check** in 03 section 9 (`with_kb_sanitizers`): entries of kind other than `insufficient` mark sanitizer nodes that the check excludes.

## 3. Sink contexts (taxonomy)
Define a shared enum used by the KB, playbook decision tables (07), pattern families (15) and validator (08):
- SQL: `sql_value`, `sql_identifier`, `sql_order_by`, `sql_fragment`
- OS command: `os_command_arg`, `os_command_string`, `shell_interpreted`
- Path: `file_path_read`, `file_path_write`, `archive_extract_path`
- XSS: `html_text`, `html_attribute`, `html_attribute_url`, `js_string`, `js_code`, `css`, `dom_sink`
- Others: `ldap_filter`, `ldap_dn`, `xpath`, `xml_parser` (XXE), `deserialization`, `ssrf_url`, `open_redirect`, `log_message`, `regex`, `nosql_query`, `template_engine`, `header_value`
New families append to the taxonomy; the generic playbook uses `other`. Pre-classification in 03 section 5 proposes the sink context as a **hint**; the agent confirms it against the code and the decision table (07 section 5) consumes it.

## 4. Starting content (seed; analysts extend)

Seed at least these per language, marking confidence and bypass conditions; **verify each entry against current official docs before enabling** (do not copy blindly).
- **Java:** `PreparedStatement` and `NamedParameterJdbcTemplate` bind parameters; JPA named/positional parameters; MyBatis `#{}` (safe) versus `${}` (unsafe); OWASP Java Encoder (`Encode.forHtml*`, `forJavaScript`, `forUriComponent`) by context; Spring `HtmlUtils.htmlEscape` (HTML text only); `ProcessBuilder` with argument list and no shell versus `Runtime.exec(String)`; `Path.normalize` plus `startsWith(base)` check (with caveats: must be applied after resolve and on real paths); XML parsers with secure-processing features for XXE.
- **C#:** parameterized `SqlCommand.Parameters`, EF LINQ and `FromSqlInterpolated` (safe) versus `FromSqlRaw` with concatenation (unsafe); `HtmlEncoder`, Razor auto-encoding versus `Html.Raw`; `Process.Start` with `ArgumentList` and `UseShellExecute=false`; `Path.GetFullPath` plus prefix check; `XmlReaderSettings` with `DtdProcessing.Prohibit`.
- **Python:** DB-API parameterized queries (`cursor.execute(sql, params)`), SQLAlchemy bound parameters versus `text(f"...")`; Django ORM versus `raw()`/`extra()` with interpolation; `subprocess.run([...], shell=False)` versus `shell=True`; `shlex.quote` (context-limited); `markupsafe.escape`, Jinja2 autoescape; `os.path.realpath` plus `commonpath` check; `defusedxml`.
- **Node.js:** parameterized queries (`pg` `$1`, `mysql2` `?`, Sequelize replacements/bind), Prisma tagged templates versus `$queryRawUnsafe`; `child_process.execFile`/`spawn` with args and no shell versus `exec`; `path.resolve` plus `startsWith` check; `DOMPurify`, framework escaping versus `innerHTML`/`dangerouslySetInnerHTML`; `escape-html`.
- **Go:** `database/sql` with `?`/`$1` placeholders versus `fmt.Sprintf` queries; `exec.Command` with args (no shell) versus `sh -c`; `html/template` (contextual autoescape) versus `text/template`; `filepath.Clean` plus `strings.HasPrefix` check (with caveats) or `filepath.IsLocal`.

## 5. Lookup tool: `sanitizer_lookup`

```
sanitizer_lookup(symbol, sink_context, language) -> {
  "matches": [{"id": "", "kind": "", "protects": true|false, "reason": "", "requires": [], "bypass_conditions": [], "confidence": ""}],
  "in_repo_body": "code if symbol is in repo (returned as a snippet with an ID)",
  "origin": "kb | in_repo | library_unknown",
  "decompiled_available": true|false,
  "precision": "exact | approximate | name_based"
}
```
Matching steps: (1) resolve the symbol via 03; (2) look up by fully qualified name in the KB; (3) if the symbol is in the repo, fetch the body and run the custom sanitizer procedure; (4) if it is a library symbol not in the KB, return `origin: library_unknown`.

Pre-computation: 03 may run `sanitizer_lookup` for every sanitizer candidate on the path while building the pack, so assessments arrive as `sanitizer_assessments` and the agent rarely needs the tool.

## 6. Custom (in-repo) sanitizer evaluation procedure

When a sanitizer candidate is defined in the repo, the agent plus tools check:
1. **Context match:** which sink context does it claim to protect? Does it match the finding's sink context?
2. **Mechanism:** allowlist or strict-validation (strong) versus blocklist or regex replace (weak) versus encoding (strong if context-correct).
3. **Bypassability:** common bypass classes per family (case, encoding layers, double decode, Unicode normalization, null bytes, path traversal variants such as `..\`, URL-encoded separators, quote handling, comment injection).
4. **Coverage:** is it applied on **every** branch reaching the sink? Look at the slice guards; early returns or conditional application make it partial.
5. **Ordering:** sanitization before concatenation or transformation? Post-sanitize decoding or concatenation can undo protection.
6. **Return handling:** is the sanitized value the one actually used at the sink, or is the original variable used?

Output an assessment object (consumed by the verdict as `facts.sanitization`):
```json
{"symbol": "", "claimed_context": "", "matches_sink_context": true, "mechanism": "allowlist|blocklist|encoding|unknown", "covers_all_paths": true, "bypass_risks": [], "assessment": "effective | partial | ineffective | unknown", "evidence": ["S4"]}
```
Evidence cites snippet IDs. **Only `effective` with high evidence may support an FP verdict** (the decision tables in 07 encode this). `partial`, `ineffective`, `unknown` push toward TP or `NEEDS_HUMAN`.

## 7. Library sanitizers without source
- Prefer KB match. If the symbol is a third-party library not in the KB, do not assume it is safe: return `library_unknown`, and the agent leans `NEEDS_HUMAN` unless other evidence is decisive.
- Optional enhancement: decompile on demand (CFR or Fernflower for JVM, ILSpy for .NET) inside the sandbox to inspect the implementation; results are tagged `decompiled` and treated as lower confidence.

## 8. Governance of the KB
- Changes by pull request; each entry requires `added_by`, a reference, and a reviewer. Entries start `confidence: medium` until verified against the evaluation harness.
- **Entry-level effectiveness tracking:** record each time an entry supports a verdict and whether a reviewer later overturned that verdict. An entry with overturn rate above a threshold is auto-flagged for review and demoted.
- Tests: each entry has at least one positive fixture (safe usage) and one negative fixture (bypass condition) as small code snippets in the test suite; CI verifies the lookup matches or rejects them correctly.
- The KB version is recorded in every verdict (`playbook_version` plus `kb_version`) and in the Evidence Pack hash inputs, so a KB change invalidates reuse where it matters.
- Seed entries are starting material written from general knowledge. **Analysts must verify each against official documentation before enabling it.**

## 9. Data model
```sql
CREATE TABLE sanitizer_entries (
  id TEXT, version INT, language TEXT, family TEXT, kind TEXT,
  definition JSONB, confidence TEXT, status TEXT,   -- active | demoted | retired
  PRIMARY KEY (id, version)
);
CREATE TABLE sanitizer_usage (
  finding_id UUID, entry_id TEXT, entry_version INT,
  supported_verdict TEXT, reviewer_outcome TEXT,    -- accepted | overturned | pending
  created_at TIMESTAMPTZ
);
```
(The YAML in git is the source of truth; the DB is a loaded cache for usage tracking.)

## 10. Acceptance criteria
- KB loads and validates against a JSON schema; invalid entries fail CI.
- For every seeded entry: positive and negative fixtures pass.
- `sanitizer_lookup` correctly distinguishes: parameterized query (protects `sql_value`), same API used with concatenated query string (bypass), HTML encoder used for SQL sink (does not protect), `${}` versus `#{}` in MyBatis, `exec` versus `execFile`.
- Custom sanitizer procedure unit-tested on fixtures: effective allowlist, blocklist replace, sanitizer applied on only one branch, sanitize-then-decode, returned value unused.
- Library symbol not in KB returns `library_unknown` and never `protects: true`.
- The KB-excluded flow check (03 section 9) removes exactly the flows through non-`insufficient` entries in fixtures.
- Usage and overturn tracking works end to end with a mocked reviewer outcome.
