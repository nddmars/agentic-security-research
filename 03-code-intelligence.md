# 03: Code Intelligence Service

Author: Surya

Read `00-MAIN-DESIGN.md` first. Depends on `02-source-acquisition.md`.

## 1. Goal
A standalone service that, given a finding and its source snapshot, builds an **Evidence Pack** (section 6): mapped path nodes, enclosing functions, a path-guided slice, guards, callers up to entry points, sanitizer candidates, pre-classified facts, and an independent data-flow check. It also exposes a small read-only tool API for bounded gap-filling by the agent. Every response declares its precision.

This is the most important component for triage quality. The LLM can only be as good as the context built here.

## 2. Engines (v1 decision)

| Tool | Used for | Precision tag |
|---|---|---|
| **Joern** (code property graph: AST, CFG, PDG, call graph, dataflow) | The only code-analysis engine: node mapping, enclosing functions, slicing, guards, callers to entry points, flow check | `exact` or `approximate` (depends on frontend and construct) |
| **ripgrep** | Config search, string-wired dispatch (reflection targets, bean XML, route tables), name-based caller fallback | always `name_based` |
| **Path and naming rules** | File classification (test, generated, vendored, config), module discovery | n/a |
| Per-language supplements (section 9) | Added **only** where measured caller recall is too low | tagged with their engine |

Decisions:
- **No tree-sitter and no separate AST or call-graph stack in v1.** Checkmarx already parses the code for its own findings, but it does not expose its AST, call graph, enclosing functions, guards or callers. Joern fills that gap and builds its graph internally, so no AST code is written by us.
- **No Semgrep.** Checkmarx does pattern detection.
- When no CPG can be built, the finding goes to L1 or L2 of the degradation ladder (section 7) or to `NEEDS_HUMAN_NO_ANALYSIS`, with the reason recorded. Track the "no CPG" rate per language and size tier. If it is high enough to hurt, add a lightweight tree-sitter fallback in phase 2 (decision gate in 14).

Joern frontends (confirm names against the installed Joern version): Java `javasrc2cpg` (source) or `jimple2cpg` (bytecode); JS/TS `jssrc2cpg`; Python `pysrc2cpg`; C# `csharpsrc2cpg`; Go `gosrc2cpg`. Maturity is uneven: Java and JS strongest, C# and Go newer. Verify in phase 0.

## 3. Language adapter interface

Each language implements the same interface. The agent never sees which engine answered.

```python
class LanguageAdapter(Protocol):
    language: str
    def build_index(self, snapshot: Snapshot, module_scope: list[str]) -> IndexHandle: ...
    def map_node(self, idx: IndexHandle, node: PathNode) -> MappedNode | MappingFailure: ...
    def get_function(self, idx, file: str, line: int | None, symbol: str | None) -> FunctionContext: ...
    def find_callers(self, idx, symbol: str, depth: int) -> CallersResult: ...
    def find_callees(self, idx, symbol: str, depth: int) -> CallersResult: ...
    def find_implementations(self, idx, symbol: str) -> list[SymbolRef]: ...
    def entrypoints_reaching(self, idx, symbol: str, max_depth: int) -> EntrypointResult: ...
    def get_guards(self, idx, mapped_path: list[MappedNode]) -> list[Guard]: ...
    def get_slice(self, idx, finding: Finding, budget_tokens: int) -> SliceBundle: ...
    def flow_check(self, idx, finding: Finding, exclude_kb_sanitizers: bool) -> FlowCheckResult: ...
    def sanitizer_candidates(self, idx, mapped_path) -> list[SanitizerCandidate]: ...
    def literal_usage(self, idx, file: str, line: int, col: int | None) -> UsageContext: ...  # used by 15
    def supplement(self) -> list[SupplementEngine]: ...  # optional precision engines
```

All results include the precision envelope from `00-MAIN-DESIGN.md` section 7.3.

## 4. Node mapping (critical, verify always; done in Joern)

Map each Checkmarx path node (file, line, column, length, name) to a CPG node.

Procedure:
1. Resolve the file in the snapshot. If missing: `MappingFailure(file_not_found)`. Try case-insensitive and suffix match (Checkmarx paths may be relative to a different root, or an archive may add a top-level folder); record how it matched.
2. Read the file as bytes. Detect and strip a UTF-8 BOM, remember the offset. Normalize CRLF handling consistently: compute line start offsets from the raw bytes.
3. Convert Checkmarx (line, column) to the position model Joern uses. **Pitfall:** Checkmarx columns count characters; Joern node line and column values come from its frontends, and tabs and multibyte characters can shift them. Test early with tabs and non-ASCII. Where a frontend's columns are unreliable, match by line plus node name and use column only as a tiebreaker.
4. Select the CPG node on that line whose code or name matches the Checkmarx node `name`; then walk up to the enclosing method, constructor, lambda, or class.
5. **Verify** that the source text at the mapped location contains the node `name` (case-sensitive, allowing for qualified names). If not, search within plus or minus 3 lines. If still no match: `MappingFailure(text_mismatch)`.
6. Output `MappedNode` with: file, line range, enclosing function symbol, enclosing class, annotations, and `mapping_confidence` (`verified` or `searched`).

The per-node results feed `source_match_score` (02 section 5.3).

If more than 20% of a finding's nodes fail to map (configurable), the finding is flagged `path_mapping_failed` and goes to `NEEDS_HUMAN_NO_ANALYSIS` with the failure detail. A `source_mismatch` here also downgrades the snapshot's revision confidence (see 02).

Text-only checks (does this snippet appear in this file) do not need the CPG: do them with plain file reads so a failed CPG build still allows source verification.

## 5. Pre-classification (deterministic facts and hints)

Rule tables per language, stored in the pack as `facts` (verified) or `hints` (heuristic):

- **Source class:** HTTP query, path, body, header, or cookie; message queue; file upload; database read (possible second-order); environment or config; constant. Derived from framework annotations and API signatures at the path's source node. Rule ids are recorded in `basis`.
- **Sink context:** for example `sql_value` versus `sql_order_by`, derived from how the tainted argument is built into the query string (placeholder, concatenation, template). Shared taxonomy in 05 section 3. Stored as a **hint** unless the structure is unambiguous.
- **Code class:** production, test, generated, or vendored (from 02 classification).
- **Framework facts:** routes, auth annotations, middleware (from 04).

This saves tokens and gives the validator something to cross-check the model against.

## 6. The Evidence Pack (contract with 07 and 08)

One typed pack per finding. The agent must cite snippet IDs; the validator checks that each ID exists and each quote is a verbatim substring of the cited snippet (after whitespace normalization).

```json
{
  "pack_version": "1",
  "context_level": "L0_full | L1_name_based | L2_mapping_only",
  "capabilities": {
    "cpg_built": true,
    "dispatch_modeled": {"di": false, "reflection": false, "callbacks": "partial"},
    "framework_recognized": true
  },
  "snippets": [
    {"id": "S1", "file": "", "start": 0, "end": 0, "sha": "", "kind": "path_node|function|guard|callee|config|match", "text": ""}
  ],
  "path": [{"node": 1, "snippet": "S1", "role": "source", "mapping": "verified"}],
  "facts": [{"id": "F1", "type": "source_class", "value": "http_param", "basis": "rule:spring.requestparam", "cites": ["S1"]}],
  "hints": [{"id": "H1", "type": "sink_context", "value": "sql_value", "confidence": "medium"}],
  "reachability": {"entrypoints": [], "auth": "unknown", "precision": "approximate"},
  "flow_check": {"raw": "NO_FLOW_FOUND", "with_kb_sanitizers": "NO_FLOW_FOUND"},
  "sanitizer_assessments": [],
  "gaps": [{"id": "G1", "kind": "unresolved_edge", "detail": ""}],
  "elided": [{"id": "E1", "file": "", "lines": "", "reason": "not data-dependent"}],
  "source_match_score": 0.0,
  "revision_confidence": "EXACT|HIGH|APPROXIMATE|UNKNOWN",
  "pack_hash": "sha256 of the canonical JSON above"
}
```

Rules:
- **Facts vs hints.** Facts are deterministic, tool-verified (for example "annotation `@PreAuthorize` exists at line 40"). Hints are heuristic (for example a guessed sink context). The agent treats facts as given and verifies hints against the snippets.
- **Pack hash.** The canonical-JSON hash goes into the fingerprint in 06, so identical context yields verdict reuse and changed context forces re-triage.
- **Elided regions** carry IDs and reasons. In the rendered prompt, omitted code is replaced by a marker such as `// ... 14 lines omitted (E3), not data-dependent`, so the model knows what it has not seen.
- Secrets in snippet text are redacted before the pack is stored for prompting (07 section 9). Redaction preserves variable names and structure.

## 7. Degradation ladder (stored in the pack; chosen deterministically)

| Level | What exists | What 07 may conclude |
|---|---|---|
| L0 | CPG built; mapping verified; slice, flow check, framework facts | Any verdict, subject to the validator |
| L1 | No CPG. Plain file reads, ripgrep (name-based callers and config), path rules | `TP` or `NEEDS_HUMAN`; `FP` only for narrow categories that need no flow or reachability reasoning (for example test or generated file, with verified text) |
| L2 | Mapping by line only; enclosing function by simple text or none | `NEEDS_HUMAN`, or `TP` for obvious cases |

The level selects the prompt variant and the allowed verdicts, so the model never decides how much to trust its own context. Every "no CPG" outcome records a reason (`index_failed_timeout`, `index_failed_oom`, `index_failed_parse`, `unsupported_language`, `size_tier_exceeded`) for the phase-2 decision on a fallback parser.

## 8. Path-guided slicing (token budget)

Goal: a bounded, structured slice that contains what decides TP versus FP. Budget in **tokens**, not lines (default about 6,000 tokens of code; configurable).

Priority tiers:
1. **Must include:** path-node statements and the data-dependent statements of each enclosing function (backward data slice from the sink argument using the PDG).
2. **Should include:** guards (control-dependent `if`, `switch`, `try/catch`, early `return`/`throw`, null checks, authorization and validation checks), sanitizer candidate bodies, and the entry-point handler signature with its annotations.
3. **Nice to have:** callers beyond the first hop; callee bodies under 40 lines that propagate taint.
4. **Summarize:** everything else, as a one-line deterministic summary (parameters, return type, calls made) or an `elided` marker.

Algorithm:
1. **Seed:** mapped path nodes; criterion is the sink argument expression.
2. **Backward data slice per function** on the path using Joern's data dependence.
3. **Guards:** record each as a `Guard` with kind (`auth`, `validation`, `feature_flag`, `other`).
4. **Inter-procedural expansion** for each call on the path: include the callee body if it is (a) a sanitizer or validator candidate, (b) a wrapper that propagates taint, or (c) short (under 40 lines, configurable).
5. **Upward expansion** to entry points: for the first function on the path, include up to N callers per hop (default 3) until reaching an entry point (route, controller action, main, scheduled job, message consumer) or max depth (default 6). Include the entry point's auth annotations and middleware.
6. **Trim at statement level.** Never silently truncate; every omission is an `elided` entry.

## 9. Independent flow check (Joern), run twice

`flow_check` runs a Joern data-flow query from the mapped source to the mapped sink and compares it to the Checkmarx path. Run it **twice**:
- `raw`: no sanitizer exclusions.
- `with_kb_sanitizers`: excluding flows that pass through sanitizers recognized by the KB (05).

Report both in the pack. If the raw run finds a flow and the KB run does not, a KB-listed sanitizer is on the path: a strong signal that the sanitizer matters and the agent knows which one to inspect.

```scala
importCode(inputPath = "/workspace/repo", projectName = "p") // or load a cached CPG
def sinkArgs = cpg.call.nameExact("<sink>").lineNumber(<line>).argument
def sources = cpg.call.nameExact("<source>").lineNumber(<line>)
sinkArgs.reachableByFlows(sources).p
```
Result values: `FLOW_CONFIRMED` (a flow exists and overlaps the Checkmarx path), `FLOW_DIFFERENT` (a flow exists but through different nodes), `NO_FLOW_FOUND`, `INCONCLUSIVE` (engine error, timeout, or unmapped nodes).

**Important:** `NO_FLOW_FOUND` is **not** proof of a false positive. Engines model frameworks, libraries, and sanitizers differently. Check how your Joern version treats unresolved external methods by default, since this affects how often it agrees with Checkmarx. The validator (08) must downgrade FP verdicts that rest solely on `NO_FLOW_FOUND` unless the asymmetry rule is satisfied.

Run with a timeout (config, default 120 seconds) and memory cap. On timeout return `INCONCLUSIVE`.

## 10. Call graph, ripgrep fallback and entry points

Joern's call graph is the only call graph in v1. It is weak for: dependency injection, interface dispatch with multiple implementations, reflection, callbacks, event buses, queues, ORMs, `eval`/dynamic `require`/`importlib`, dynamic languages (Python and Node are mostly name-resolved, so `approximate`), and the newer C# and Go frontends.

How the design handles it:
- **Synthetic edges** from framework facts (04): route-to-handler, interface-to-implementation via DI config, middleware chains.
- Every result carries `precision` and `unresolved_edges`. Every blind spot touching the path is listed.
- The **asymmetry rule** applies: "no callers" never justifies FP unless dispatch constructs on the path were modeled (capability matrix, section 11).

**ripgrep fallback for missing callers** (a defined step, not ad hoc):
1. Search for the method or class name as a call pattern per language, and as a plain string (catches reflection targets, XML or YAML bean definitions, route tables, config-driven dispatch).
2. **Tag everything `name_based`.** A name match can hit unrelated methods with the same name.
3. **Map each hit to its enclosing method through Joern** (or by file and line if there is no CPG) so the agent receives functions, not bare lines.
4. **Never treat an empty result as proof of "no callers".**
5. **Bound it:** cap hits per query and files searched; exclude vendored and build folders; return a count when truncated.

`entrypoints_reaching` returns chains with route or trigger, HTTP method, auth annotations, and middleware. If none are found, return `entrypoints: []` **plus** the unresolved edges that could explain it. Entry point detection patterns (extend in 04): Java `@RequestMapping/@GetMapping/@PostMapping`, JAX-RS `@Path`, servlets `doGet/doPost`; C# `[HttpGet]/[Route]` controllers, minimal APIs `MapGet`; Python `@app.route`, Django `urls.py`, FastAPI decorators; Node `app.get/post`, `router.*`, NestJS decorators; Go `http.HandleFunc`, `mux.Handle`, gin/echo route registrations; plus `main`, scheduled jobs, message consumers.

## 11. Capability matrix

`dispatch_modeled` in the pack comes from a per-language and per-framework table in config (for example Java + Spring DI modeled: yes or no; C# + ASP.NET DI: partial), set from phase 0 measurements, **not** guessed at runtime. The validator's asymmetry rule reads it directly. Unknown framework means all dispatch flags false.

## 12. Index lifecycle and scaling

Work is split by snapshot and by finding:
- **Per snapshot (heavy, once):** CPG build, module discovery, framework facts, entry-point table, and a per-function summary index (parameters, calls made, annotations, return type).
- **Per finding (light):** mapping, slicing, flow check, pack assembly.

At about 1,500 projects this split keeps cost sane.

- Key every index by (snapshot revision, module scope hash, engine version). Cache CPGs in object storage; load on demand. Invalidate on revision or engine version change.
- **Size tiers** (thresholds are config; see 06 for budgets): small and medium repos get a full CPG; large repos get a CPG scoped to `module_scope`; very large repos are filtered (02) then scoped; if still too large or the build exceeds budget, fall back to L1 and the finding leans `NEEDS_HUMAN`.
- Joern workers: dedicated pods with explicit JVM heap (`-Xmx` sized per tier, below the pod memory limit with headroom for off-heap). Set a hard build timeout; on timeout mark `index_failed(timeout)`.
- Build one CPG per language per snapshot (mixed-language repos build multiple CPGs). Record build time, memory peak, node counts for metrics.
- Concurrency: limit concurrent Joern builds per node. Joern server instances can serve multiple queries on one loaded CPG; keep a warm pool keyed by CPG, with TTL eviction.

## 13. Per-language notes and supplements

Verify each during phase 0 on real repos. Supplements are enabled only where measured caller recall or node-mapping success is insufficient.

- **Java:** Joern source frontend has weaker type resolution than a bytecode CPG. If the repo builds in a sandbox (dependencies from the internal artifact proxy), consider a bytecode CPG. Supplements: jdtls or SootUp for exact callers. Watch: Spring DI (`@Autowired`, constructor injection), interface dispatch, reflection, Struts action mappings, JAX-RS.
- **C#:** `csharpsrc2cpg` is newer; expect gaps in type resolution, DI (`AddScoped/AddTransient/AddSingleton`), LINQ, async. Supplement: Roslyn `SymbolFinder` (requires project load; sandboxed restore from internal NuGet proxy). Likely the first language needing a supplement.
- **Python:** dynamic typing, decorators, duck typing. Supplements: pyright for types; approximate callers via PyCG-style analysis. Mark call graph precision `approximate` by default.
- **Node.js (JS/TS):** callbacks, middleware chains, dynamic dispatch. Supplement: TypeScript compiler API or tsserver for exact references on TS; JS stays `approximate`. Model Express/Nest middleware order in 04.
- **Go:** `gosrc2cpg` is newer. Supplement: `golang.org/x/tools/go/callgraph` (RTA or VTA) plus gopls; needs module download from the internal proxy. Watch: interface dispatch, goroutines and channels, generics.

Supplement interface: each supplement implements a subset of the adapter methods (typically `find_callers`, `find_implementations`, `entrypoints_reaching`) and merges edges into the Joern results, tagging merged edges with their engine and precision.

## 14. Tool API (what the agent sees for gap-filling; see 07)

```
resolve_symbol(file, line, col) -> symbol, kind, definition
get_function(file, symbol | line) -> code, annotations, precision
find_callers(symbol, depth) -> edges[], precision, unresolved[]
find_callees(symbol, depth) -> edges[], precision, unresolved[]
find_implementations(interface_method) -> symbols[], precision
entrypoints_reaching(symbol, max_depth) -> chains, routes, auth, middleware
get_guards(finding_id) -> guards[]
flow_check(finding_id, exclude_kb_sanitizers) -> FlowCheckResult
sanitizer_lookup(symbol, sink_context) -> see 05
search_config(key | pattern) -> matches[] (ripgrep over config files, name_based)
search_usages(name, kind) -> matches[] (ripgrep, mapped to enclosing methods, name_based)
read_file(file, start_line, end_line) -> text (bounded, max lines per call)
```
All tools are **read-only**. Every result is returned as **new snippets with IDs** appended to the pack, so citations keep working. All return the precision envelope. All enforce limits (max lines returned, max edges, per-call timeout). Calls are logged for audit with a digest.

## 15. Data model

```sql
CREATE TABLE code_indexes (
  index_id UUID PRIMARY KEY,
  snapshot_id UUID,
  language TEXT,
  engine TEXT, engine_version TEXT,
  module_scope_hash TEXT,
  storage_uri TEXT,
  status TEXT,                -- READY | FAILED
  failure_reason TEXT,
  build_seconds INT, peak_memory_mb INT, node_count BIGINT,
  created_at TIMESTAMPTZ,
  UNIQUE (snapshot_id, language, engine, engine_version, module_scope_hash)
);

CREATE TABLE finding_context (
  finding_id UUID PRIMARY KEY,
  index_id UUID,
  context_level TEXT,         -- L0_full | L1_name_based | L2_mapping_only
  mapping_failures JSONB,
  pack_uri TEXT,              -- Evidence Pack in object storage
  pack_hash TEXT,
  flow_check_raw TEXT,
  flow_check_kb TEXT,
  flow_check_detail JSONB,
  precision TEXT,
  unresolved_edges JSONB,
  no_cpg_reason TEXT,         -- for the phase-2 fallback decision
  built_at TIMESTAMPTZ
);
```

## 16. Failure modes (all map to `NEEDS_HUMAN_NO_ANALYSIS` unless a lower ladder level applies)
`path_mapping_failed`, `index_failed_timeout`, `index_failed_oom`, `index_failed_parse`, `unsupported_language`, `slice_over_budget_unreducible`, `flow_check_inconclusive` (this one alone does not block triage; it just lowers evidence strength).

## 17. Acceptance criteria
- **Node-mapping success rate** measured on a sample of at least 200 real findings per language; target 95% or better verified mappings; failures categorized.
- Tests for column conversion with tabs, CRLF, BOM, and multibyte characters, **through Joern**, per frontend.
- **Caller recall** against a hand-labeled sample (at least 50 findings per language); report per language and per framework; decide supplements from the numbers.
- **ripgrep fallback:** results are always tagged `name_based`, mapped to enclosing methods, bounded; an empty result never produces an unqualified "no callers".
- Slice size distribution reported; 90% of slices within token budget; analysts rate slice sufficiency on a sample.
- Dual flow check implemented; agreement rate with Checkmarx paths reported per language and query family; `NO_FLOW_FOUND` never used alone to justify FP (enforced in 08).
- **No-CPG rate** reported per language and size tier with reasons (input to the phase-2 tree-sitter decision).
- Build time and memory per size tier reported; hard timeouts enforced and tested with an oversized repo.
- All tools verified read-only (no write, exec, or network calls from tool code); tests for path traversal in `read_file` (cannot read outside the snapshot).
- Precision envelope present on 100% of tool responses; Evidence Pack validates against its JSON schema on 100% of findings; `pack_hash` is stable across rebuilds of identical inputs.
