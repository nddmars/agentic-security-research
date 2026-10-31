# 16: Modules, Functions, and Implementation Libraries

Author: Surya

Read `00-MAIN-DESIGN.md` first. This file maps the design onto a code layout: one Python package (`cxtriage`) with the modules below, each module's public functions, and the libraries to implement it. Feature files (01 to 15) remain the specification for behavior; this file is the implementation index.

Library choices are proposals, not mandates. Before adopting any library, check its current version, license and internal approval status (artifact proxy, SBOM policy). Items marked **(verify)** are the ones most likely to need a decision in phase 0.

## 1. Package layout

```
cxtriage/
  common/            shared contracts, config, logging, errors, events
  cxclient/          CxSAST API wrapper (read + restricted write)
  ingestion/         scans, results, normalization          [01]
  sourceacq/         scan comment parsing, fetch, filter    [02]
  codeintel/         Joern adapter, mapping, slice, packs   [03]
  frameworks/        detection + facts extraction           [04]
  sanitizers/        sanitizer KB + lookup                  [05]
  orchestrator/      workflows, dedupe, budgets, routing    [06]
  llm/               model gateway clients, prompts, caching [07]
  agent/             facts-first triage runs                [07]
  playbooks/         decision tables + engine               [07, 08]
  validator/         deterministic gate                     [08]
  writeback/         comment blocks + predicate updates     [09]
  patterns/          Checkmarx pattern-finding triage       [15]
  redaction/         secret redaction + masking             [07, 12, 15]
  dashboard_api/     read-only REST API                     [10]
  dashboard_ui/      React front end                        [10]
  evalharness/       golden sets, metrics, gates            [11]
  platform/          observability, admin CLI, security helpers [12, 13]
  tests/             unit, fixtures, integration, security
  deploy/            Containerfiles, OpenShift manifests, GitOps [13]
```
Dependency rule: lower layers never import higher layers. Order from bottom: `common` -> `redaction`, `cxclient` -> `ingestion`, `sourceacq` -> `codeintel` (+`frameworks`, `sanitizers`) -> `playbooks` -> `llm`, `agent` -> `validator` -> `writeback` -> `orchestrator` (wires everything). `dashboard_api` and `evalharness` read from the database only.

## 2. Cross-cutting library choices (used by all modules)

| Concern | Library | Notes |
|---|---|---|
| Language | Python 3.12 | Pinned in Containerfiles, CI and lockfiles; Joern scripts are Scala invoked from Python |
| Data models and validation | `pydantic` v2 | All contracts from 00 section 7 are Pydantic models; JSON Schema is generated from them |
| JSON schema checks | `jsonschema` | Validate model outputs and pack documents independently of Pydantic |
| Config | `pydantic-settings` + `PyYAML` | Versioned YAML plus environment overrides |
| HTTP | `httpx` (async) | CxSAST, source hosts, LLM gateways |
| Retry/backoff | `tenacity` | Jittered exponential backoff on 429/5xx |
| Database | PostgreSQL via `SQLAlchemy` 2.x + `psycopg` 3; migrations with `alembic` | Core (not ORM-heavy) for bulk upserts |
| Object storage | `boto3` (S3-compatible) | CPGs, packs, snapshots, audit archives |
| Fast JSON | `orjson` | Pack hashing uses canonical JSON (sorted keys) |
| Hashing | `hashlib` (stdlib) | SHA-256 for packs, files, fingerprints |
| Logging | `structlog` | JSON logs with correlation IDs; redaction processor |
| Metrics/tracing | `prometheus-client`, `opentelemetry-sdk` | OpenShift monitoring scrape + OTLP traces |
| Testing | `pytest`, `pytest-asyncio`, `hypothesis`, `testcontainers` | Hypothesis for the validator "never more FP" property test |
| Lint/types | `ruff`, `mypy` | CI gates |
| Packaging | `uv` or `pip-tools` with hash-pinned lockfile | Pinned, scanned, SBOM via `cyclonedx-py` |

## 3. Modules

### 3.1 `common`
Purpose: shared contracts and utilities. No business logic.
Functions/classes:
- `models.Finding`, `PathNode`, `EvidencePack`, `Snippet`, `Fact`, `Hint`, `Gap`, `Verdict`, `ValidatedVerdict`, `PrecisionEnvelope` (Pydantic).
- `enums`: `Verdict`, `Severity`, `FindingKind`, `ContextLevel`, `RevisionConfidence`, `SinkContext`, `LifecycleState`.
- `config.load_settings(env)`; `config.feature_flags()` (kill switch, dry-run, allowlists, read at runtime).
- `canonical_json(obj) -> bytes`; `sha256_hex(bytes)`; `pack_hash(pack)`.
- `events.emit(event)`; `lifecycle.transition(finding_id, to_state, reason, detail)`.
- `errors`: typed exceptions mapped to failure reasons (02 section 10, 03 section 16).
- `paths.normalize_path()`, `paths.safe_join(root, rel)` (rejects traversal).
Libraries: `pydantic`, `PyYAML`, `orjson`, stdlib.

### 3.2 `redaction`
Purpose: remove or mask secrets before anything reaches a prompt, log, comment or dashboard (07 section 9, 12 section 7, 15).
Functions:
- `redact_text(text, mode) -> RedactedText` (modes: `sast` placeholder `<REDACTED:type>`, `pattern` mask `****`).
- `detect_secrets(text) -> list[Finding]` (private key blocks, cloud credential formats, token prefixes, password-like assignments, connection strings, high-entropy strings).
- `value_fingerprint(value, salt) -> str` (HMAC hash used for dedupe; never the value).
- `assert_no_secret(text, known_fingerprints)` (the masking check used by the validator and writeback).
Libraries: `re`, `math` (Shannon entropy), optional `detect-secrets` **(verify)** as a detector source, `hmac`. Keep own regex set in YAML so analysts can extend it.

### 3.3 `cxclient`
Purpose: the only code allowed to talk to CxSAST (01, 09). Exposes no method that writes the scan comment.
Functions:
- `authenticate()` with token refresh; `list_scans(from, status)`; `get_scan(scan_id)`; `iter_results(scan_id)` (paged/streamed); `get_result_paths(scan_id, path_id)`.
- `list_result_states()`; `get_result_predicate(scan_id, path_id)`; `update_result_predicate(...)` with `expected_notes_hash`.
- `get_project_source_settings(project_id)` (origin cross-check, if exposed).
- Rate limiter, circuit breaker, request audit logging.
Libraries: `httpx`, `tenacity`, `aiolimiter`, `pybreaker` (or a small hand-written breaker). No official Python SDK is assumed **(verify)**; endpoints and payloads come from the phase 0 API spike.

### 3.4 `ingestion` (feature 01)
Functions:
- `discover_scans()`; `ingest_scan(scan_id)`; `backfill(from, to, project)`.
- `normalize_result(raw) -> Finding` incl. `classify_kind()` (path_based vs pattern_only), `detect_language()`, `assign_roles()`, `compute_path_hash()`.
- `parse_project_name(name) -> (application_id, component)` (configurable regex).
- `mask_pattern_snippets()` for secrets-like queries.
- Admin CLI: `ingest scan|backfill|status`.
Libraries: `cxclient`, `SQLAlchemy`/`psycopg` (bulk `INSERT ... ON CONFLICT`), `regex` or `re`.

### 3.5 `sourceacq` (feature 02)
Submodules and functions:
- `comment_parsers/`: `ArchiveUrlParser` (primary), `GitUrlParser`, `WebUiUrlParser`, `KeyValueParser`, `parse_chain(comment) -> Candidate | None`; `redact_url_credentials(url)`.
- `ssrf.py`: `validate_host(url)`, `resolve_and_check(host)`, `check_redirect_chain()`.
- `git_fetch.py`: phase 2 only, off by default (see 17): `fetch_git(url, ref, sha, anchor_time)`, `resolve_revision()` using `scan_started_at`.
- `archive_fetch.py`: `download_archive()`, `safe_extract()` (traversal, symlink, bomb limits).
- `filtering.py`: `filter_workspace()`, `classify_file()` (source/test/generated/vendored/config).
- `modules.py`: `discover_modules(workspace)` (Maven/Gradle, `.sln/.csproj`, workspaces, `go.mod`).
- `verify.py`: `compute_source_match(finding, snapshot) -> score, node_results`; `cross_check_origin()`; `per_file_revision()` for incremental scans.
- `cache.py`: snapshot cache with LRU and in-flight pinning.
Libraries: system `git` CLI invoked through `subprocess` with argument arrays only (no shell, hooks disabled) rather than a Python git library; `httpx` (streaming download); `ipaddress`, `socket`; stdlib `zipfile`/`tarfile` wrapped by own safe extractor; `pathspec` (gitignore-style patterns for filters); `charset-normalizer` (encoding detection); `rapidfuzz` (snippet matching tolerance, if needed) **(verify)**.

### 3.6 `codeintel` (feature 03)
Submodules and functions:
- `joern/`: `JoernServer` (start/stop, warm pool, TTL), `build_cpg(snapshot, language, module_scope)`, `load_cpg()`, `run_query(script, params, timeout)`; per-language adapters `JavaAdapter`, `CSharpAdapter`, `PythonAdapter`, `JsTsAdapter`, `GoAdapter` implementing `LanguageAdapter`.
- `mapping.py`: `map_node(idx, node) -> MappedNode | MappingFailure` (BOM/CRLF/tab/multibyte handling, name verification, plus/minus 3 line search); text-only fallback `text_verify()` that needs no CPG.
- `slicing.py`: `get_slice(idx, finding, budget_tokens)` with tiers (must/should/nice/summarize); `trim_to_statements()`; `elided` markers.
- `flowcheck.py`: `flow_check(idx, finding, exclude_kb_sanitizers)` run twice (raw and KB-excluded); result enum.
- `callers.py`: `find_callers()`, `find_callees()`, `find_implementations()`, `entrypoints_reaching()` merging Joern edges with `frameworks` synthetic edges; `unresolved_edges` tracking.
- `rg_tools.py`: `search_usages(name, kind)`, `search_config(pattern)`, name-based fallback mapped to enclosing methods.
- `preclass.py`: `classify_source()`, `hint_sink_context()`, `classify_code()` using rule tables in YAML.
- `pack_builder.py`: `build_pack(finding, snapshot, idx) -> EvidencePack` (assigns snippet IDs `S1..`, facts/hints, gaps, degradation level L0/L1/L2, `pack_hash`).
- `usage.py`: `literal_usage(idx, file, line, col)` for pattern findings.
- `supplements/`: optional `RoslynSupplement`, `GoCallgraphSupplement`, `JdtlsSupplement`/`SootUpSupplement`, `PyrightSupplement`, `TsServerSupplement` (enabled by phase 0 data).
- `tools_api.py`: read-only tool endpoints exposed to the agent (03 section 14) with limits and audit digests.
Libraries and tools: **Joern** (`joern-cli`, run as a server; Python talks to it with `cpgqls-client` or plain `httpx` against its query endpoint **(verify)**, queries written in Scala/CPGQL); **ripgrep** (`rg --json`, parsed with `orjson`); `pathspec`; `tiktoken`-free token budgeting: use the provider's token counting API or a conservative character-based estimator **(verify)** because counts differ by model. Supplements use their own toolchains inside sandboxed images (Roslyn via .NET, `golang.org/x/tools`, jdtls, pyright, TypeScript compiler API).

### 3.7 `frameworks` (feature 04)
Functions:
- `detect(snapshot, module) -> FrameworkReport` from manifests, text markers and config files, with `catalog.yml`.
- Extractors (plugins): `SpringExtractor`, `JaxRsServletExtractor`, `AspNetExtractor`, `ExpressNestExtractor`, `DjangoFlaskFastApiExtractor`, `GoHttpGinEchoExtractor`.
- `extract_entry_points()`, `extract_auth()`, `extract_di_edges()`, `extract_protections()`, `unknown_fallback()`.
- `capability_matrix.lookup(language, framework) -> dispatch_modeled`.
- Weekly report job: `framework_volume_report()`.
Libraries: manifest parsing with `tomllib` (stdlib), `lxml` (pom.xml/csproj, hardened against XXE: `resolve_entities=False`), `json`, `PyYAML`; ripgrep for text markers; Joern queries for cross-file resolution.

### 3.8 `sanitizers` (feature 05)
Functions:
- `load_kb(path) -> KnowledgeBase` (JSON Schema validated); `kb_version()`.
- `sanitizer_lookup(symbol, sink_context, language) -> LookupResult`.
- `assess_custom_sanitizer(candidate, slice) -> SanitizerAssessment` (context match, mechanism, bypassability, coverage, ordering, return handling).
- `kb_excluded_nodes(language)` for the second flow check.
- `record_usage()` and `effectiveness_report()` (overturn tracking).
Libraries: `PyYAML`, `jsonschema`; Joern for symbol resolution.

### 3.9 `playbooks` (feature 07 section 5, 08 section 3.3)
Functions:
- `load_tables(path)`; `evaluate(family, facts) -> DerivedVerdict` (first-match rule evaluation, `never_fp_if`, `fp_requires`).
- `family_for(query_name, cwe)` mapping.
- `table_coverage_report()` (every row has a fixture).
Libraries: `PyYAML`, `pydantic`. Rules are data, not code; no rules-engine dependency.

### 3.10 `llm` (feature 07)
Functions:
- `LLMClient` interface with `AnthropicClient` and `GeminiClient`; `complete(prompt, schema, profile)`; structured output/JSON-schema mode where supported; retries and provider failover.
- `PromptBuilder`: static prefix (system, playbook, schema) then finding-specific pack; nonce tags; delimiter escaping; `render_pack(pack, nonce)`.
- `tokens.estimate()`; cache-control markers; batch submission helpers (`submit_batch`, `poll_batch`).
- `profiles.yml`: model names, temperature 0, max tokens (configuration only).
- Usage/cost accounting.
Libraries: `anthropic` SDK and `google-genai` SDK **(verify against the approved gateway)**; if the organization exposes a gateway, call it with `httpx` and keep SDKs optional. `tenacity` for retries.

### 3.11 `agent` (feature 07)
Functions:
- `triage(finding, pack, profile) -> Verdict` (single pass; gap-fill loop; escalation).
- `fill_gaps(gap_ids, budget)` calling `codeintel.tools_api`; new snippets appended with IDs.
- `parse_and_validate_output()` with one retry on schema error.
- `detect_injection_indicators(pack)` (phrases, hidden Unicode, bidi, base64 blobs).
- `second_opinion(pack, other_profile) -> Verdict` (independent facts).
Libraries: `pydantic`, `jsonschema`, `llm`, `playbooks`; `regex` (Unicode-category checks for hidden/bidi characters).

### 3.12 `validator` (feature 08)
Functions:
- `validate(verdict, pack, snapshot, finding) -> ValidatedVerdict` running the pipeline: schema, evidence by snippet ID (substring + `sha` check), decision-table derivation, asymmetry rule, source/mapping integrity (`source_match_score`), contradictions, calibration.
- `is_comment_only(snippet)`, `calibrate(strength, segment)`, `second_opinion_required()`, `auto_close_eligible()`, `compare_facts(a, b)`.
- `masking_check(text, fingerprints)` for pattern outputs.
Libraries: `pydantic`, `jsonschema`, `playbooks`, `redaction`; `hypothesis` in tests. No LLM calls except via the `agent.second_opinion` hook.

### 3.13 `writeback` (feature 09)
Functions:
- `compose_block(validated, pack) -> str` (deterministic; sanitized; length-aware; v2 format).
- `append_notes(existing, block)`; `plan_write(finding)` -> `WritePlan`; `execute(plan)` with read-then-append, retry on conflict, verify by re-read.
- `map_state(verdict) -> state_id`; `resolve_reviewer(application_id)`; dry-run recorder; kill-switch check; rate limiter.
- `collect_reviewer_outcomes()` polling job; `revert(window | ids)` admin operation.
Libraries: `cxclient`, `SQLAlchemy`, `aiolimiter`, `tenacity`.

### 3.14 `patterns` (feature 15)
Functions (Checkmarx pattern results only; no own scan, no rule library):
- `locate_match(finding, snapshot)` (when only a file is given: family search with ripgrep inside that file; one, many or none -> `no_match_location`).
- `build_pattern_pack(finding, idx)` (masked snippet, usage context via `codeintel.usage`, header-only classification for key/cert files, URL host classification without fetching).
- `classify_url_host(url)`; `classify_file_header(path)` (reads only the first bytes; never sends content); `value_fingerprint(literal)` (HMAC).
- Decision tables per family in `playbooks` (`hardcoded_credential_username`, `suspicious_file`, `suspicious_url`, `generic_pattern`).
Libraries: ripgrep (`rg --json`), `re`, `hmac`, `ipaddress`, `tldextract` for host classification **(verify; needs offline suffix list)**.

### 3.15 `orchestrator` (feature 06)
Functions/workflows:
- `ScanWorkflow`, `FindingWorkflow`, `PatternFindingWorkflow`; activities for each stage; idempotency keys `(finding_id, stage, input_hash)`.
- `fingerprint(finding, pack)`; `dedupe_decide()` (reuse, in-flight subscribe, invalidate, human-outcome wins).
- `size_tier(snapshot)`; `budgets(tier)`; `route_model(finding, signals)`; `priority_score()`; per-project concurrency limits.
- Watchdog for stuck findings; poison quarantine; dead-letter replay.
- Backpressure controls (CxSAST latency, LLM quota pauses).
Libraries: `temporalio` (Python SDK) preferred; fallback `celery` + Redis/RabbitMQ with a Postgres state machine **(decision in phase 0)**; `SQLAlchemy`.

### 3.16 `dashboard_api` (feature 10)
Functions: endpoints listed in 10 section 5; `refresh_materialized_views()` scheduled job; export (CSV); RBAC and row-level scoping; "data as of" timestamps.
Libraries: `FastAPI` + `uvicorn`; `SQLAlchemy` (read-only role); `python-jose` or `authlib` for OIDC token validation **(verify against the SSO standard)**; `fastapi-pagination` (optional); `slowapi` for rate limits.

### 3.17 `dashboard_ui` (feature 10)
Pages: home, scan list, scan detail, finding detail, quality, patterns, reviewer queue, ops.
Libraries: React + TypeScript, `Vite`, `TanStack Table` and `TanStack Query`, `Recharts` (or `visx`), a component kit already approved internally **(verify)**; `react-router`; Playwright for end-to-end tests.

### 3.18 `evalharness` (feature 11)
Functions:
- `load_xml_report(path)` (Checkmarx XML export: state, Remark, Path; parser hardened against XXE with `lxml` `resolve_entities=False` or `defusedxml` **(verify)**; built against the real sample first); `load_golden(release)` (immutable, schema checked); `contamination_check()`.
- `run_component_eval(kind)` for codeintel, fact extraction, decision tables, validator, pattern pipeline; `run_end_to_end()`; `run_ablation(config)`.
- Metrics: `fn_rate()`, `fp_precision()`, `fact_accuracy()`, `coverage()`, `wilson_interval()`, `calibration_error()`; `fit_calibration()`; `readiness_report()`; `gate_check()` for CI.
Libraries: `pandas`, `numpy`, `statsmodels` (`proportion_confint` for Wilson intervals), `scikit-learn` (isotonic regression, calibration curves), `matplotlib` for reports. Hand-computed fixture tests verify the metrics code.

### 3.19 `platform`
Functions:
- `observability`: metric definitions, trace helpers, log processors (secret scrubbing).
- `admin_cli`: `kill-switch`, `dry-run`, `replay-dlq`, `reindex`, `revert`, `ingest`, `config show`.
- `security`: `egress_allowlist`, `sandbox_checks` (self-tests used in CI and smoke tests), audit-log writer.
Libraries: `typer` (CLI), `prometheus-client`, `opentelemetry-sdk`, `structlog`.

### 3.20 `deploy`
Contents: Containerfiles (UBI-based images, non-root, GID 0 writable paths, random UID safe), OpenShift manifests/Kustomize (Deployments, `NetworkPolicy`, `EgressFirewall`, `Route`, `ResourceQuota`, HPA/KEDA `ScaledObject`), GitOps definitions, Temporal deployment values, Joern image with pinned version and JVM flags.
Tools: `kustomize` or Helm, Argo CD/OpenShift GitOps, `hadolint`, `trivy` or the organization's scanner, `cyclonedx` SBOM, `cosign` or the organization's signer **(verify)**.

## 4. Module-to-feature-to-test map

| Module | Feature | Key test suites |
|---|---|---|
| common | 00 | model round-trip, canonical hash stability |
| redaction | 07, 12, 15 | planted-secret tests across prompts/logs/comments/dashboard |
| cxclient | 01, 09 | mock server: paging, 429, conflicts; no scan-comment write method |
| ingestion | 01 | idempotent upsert, kind classification, 50k-result memory test |
| sourceacq | 02 | 30+ comment samples, SSRF, archive bombs, anchor-time, source-match |
| codeintel | 03 | mapping (tabs/CRLF/BOM/multibyte), recall, dual flow check, pack schema |
| frameworks | 04 | fixture apps, order-sensitive auth, unknown-framework fallback |
| sanitizers | 05 | positive/negative fixture per entry, custom-sanitizer cases |
| playbooks | 07, 08 | every table row has a fixture |
| llm / agent | 07 | schema validity, injection suite, asymmetry, fact accuracy, cost |
| validator | 08 | per-rule unit tests, "never more FP" property, determinism |
| writeback | 09 | byte-exact append, concurrency retry, human state never overwritten, masking block |
| patterns | 15 | locate-match, decision-table fixtures, masking |
| orchestrator | 06 | forced-retry idempotency, dedupe, routing, load test |
| dashboard_api / ui | 10 | count consistency vs SQL, RBAC, performance |
| evalharness | 11 | metrics vs hand-computed, gates block degraded prompt |
| platform / deploy | 12, 13 | isolation self-tests, `restricted-v2` admission, kill switch |

## 5. Suggested implementation sequence by module
1. `common`, `redaction`, `cxclient`, `ingestion` (database schema first).
2. `sourceacq` (parsers, SSRF, fetch, filtering, source match).
3. `codeintel` text-only parts (mapping verification, rg tools, preclass), then one Joern adapter (Java), slicing, flow check, `pack_builder`.
4. `playbooks`, `llm`, `agent` (SQL injection and command injection first), `validator`.
5. `orchestrator` skeleton, then dedupe/budgets; `dashboard_api` minimal.
6. `writeback` in dry-run, then live behind allowlist.
7. `patterns` (Checkmarx results).
8. `frameworks`, `sanitizers` expansion, remaining language adapters and supplements by data.
9. `evalharness` fixtures first, golden set when available; `dashboard_ui`.
10. `platform` and `deploy` hardening in parallel from the start.

## 6. Decisions to confirm in phase 0
- Temporal versus Celery (3.15).
- Joern interface: server mode client versus script runs (3.6).
- Token counting method per provider (3.6, 3.10).
- LLM access path: direct SDKs versus the organization's gateway (3.10).
- SSO/OIDC library and the approved UI component kit (3.16, 3.17).
- Whether `detect-secrets` or `tldextract` pass the internal approval process (3.2, 3.14).
