# 04: Framework Detection and Facts Extraction

Author: Surya

Read `00-MAIN-DESIGN.md` first. Depends on `02-source-acquisition.md` and `03-code-intelligence.md`.

v2 changes: no tree-sitter. Code markers are found with Joern CPG queries (where a CPG exists) and ripgrep/regex over files (always available). Facts are emitted as Evidence Pack `facts` with `basis` rule ids.

## 1. Goal
The organization has hundreds of applications with unknown and varied frameworks. This feature **detects frameworks per repository** and extracts **framework facts**: entry points (routes), authentication and authorization attachment, DI wiring, middleware order, and framework-provided protections (auto-escaping, parameterized ORM APIs, built-in CSRF or XSS defenses). These facts add synthetic edges to the call graph and feed the agent evidence.

Principle: build extractors for the top two or three frameworks per language, chosen from observed volume. For anything unrecognized, degrade gracefully: the agent gets generic evidence, precision is tagged lower, and verdicts lean toward `NEEDS_HUMAN`. Never guess.

## 2. Detection approach (no reliance on Checkmarx detection)

Checkmarx may hint at frameworks, but do not depend on it. Detect from:
1. **Manifests and dependencies** (highest signal): `pom.xml`, `build.gradle*`, `*.csproj`, `packages.config`, `package.json`, lockfiles, `requirements*.txt`, `pyproject.toml`, `Pipfile`, `go.mod`.
2. **Code markers** via ripgrep/regex over source files (annotations and attributes, imports, route registrations, base classes), refined with **Joern CPG queries** where a CPG exists and cross-file resolution is needed.
3. **Config files**: `application*.yml/properties`, `web.config`, `appsettings*.json`, `struts.xml`, `web.xml`, `settings.py`, `urls.py`.

Output per snapshot and module:
```json
{
  "module_root": "services/orders",
  "language": "java",
  "frameworks": [
    {"name": "spring-boot", "version": "2.7.x", "confidence": 0.95, "evidence": ["pom.xml: spring-boot-starter-web", "@RestController in OrdersController.java"]}
  ],
  "unknown_indicators": ["custom servlet filter chain in web.xml"]
}
```
Detection rules live in a **versioned YAML catalog** (`frameworks/catalog.yml`) so analysts can add frameworks without code changes. Each entry: name, language, dependency signatures (group/artifact, package names, NuGet IDs, npm packages, PyPI names, Go modules), code markers (regex or Joern query), config file hints, and the extractor to run (if any).

## 3. Framework catalog (starting set; extend from observed data)

- **Java:** Spring Boot/MVC/WebFlux, Spring Security, JAX-RS (Jersey, RESTEasy), Jakarta/Java EE servlets and JSF, Struts 1 and 2, Micronaut, Quarkus, Hibernate/JPA, MyBatis, JDBC templates.
- **C#:** ASP.NET Core (MVC, Web API, minimal APIs), classic ASP.NET MVC/Web API/WebForms, Entity Framework, Dapper.
- **Python:** Django, Flask, FastAPI, SQLAlchemy, Django ORM.
- **Node.js:** Express, NestJS, Koa, Fastify, Next.js API routes, Sequelize, TypeORM, Prisma, Mongoose.
- **Go:** net/http, Gin, Echo, Chi, Gorilla mux, GORM, sqlx, database/sql.

Initial extractors to build (priority; confirm against observed data after a few weeks of ingestion): Spring (Boot/MVC/Security), JAX-RS and servlets, ASP.NET Core and classic MVC/Web API, Express and NestJS, Django and Flask/FastAPI, net/http plus Gin/Echo. Everything else is detection-only (framework named, no extractor), which tags precision lower.

## 4. Facts to extract (per extractor)

### 4.1 Entry points
For each route or handler: HTTP method(s), path pattern, handler symbol (file and function), parameters bound from the request (query, path, body, header, cookie, form), declared content type.
Examples: Spring `@RequestMapping/@GetMapping/@PostMapping` plus `@RequestParam/@PathVariable/@RequestBody`; ASP.NET `[HttpGet]`, `[Route]`, `[FromBody]`, `[FromQuery]`; Express `app.get/post/use`, `router.*`; NestJS `@Controller/@Get/@Post/@Body/@Query`; Django `urlpatterns`; Flask `@app.route`; FastAPI decorators; Go `http.HandleFunc`, `mux.HandleFunc`, `r.GET`.
Also non-HTTP entry points: `main`, schedulers (`@Scheduled`, cron, hosted services), message consumers (`@KafkaListener`, `@RabbitListener`, queue triggers), CLI commands, serverless handlers.

The request-parameter bindings above are also the **source class** rules used by 03 pre-classification (`http_param`, `http_body`, `http_header`, and so on); keep one rule table and reuse it.

### 4.2 Authentication and authorization
Per entry point: required auth (none, authenticated, role or permission), source (annotation, attribute, middleware, filter, config).
Examples: `@PreAuthorize`, `@Secured`, `@RolesAllowed`, Spring Security config (`authorizeHttpRequests`, `antMatchers`); `[Authorize]`, `[AllowAnonymous]`, authorization policies; Express middleware ordering (`app.use(auth)` before routes); NestJS guards; Django `@login_required`, permission classes; Flask-Login; Go middleware wrapping handlers.
Record **order-sensitive** wiring: middleware registered after a route does not protect it. If order cannot be determined, mark `auth: unknown`.

### 4.3 Dependency injection and dispatch
Interface-to-implementation mappings: Spring `@Component/@Service/@Repository/@Bean`, `@Qualifier`, `@Primary`; .NET `services.AddScoped<I, Impl>()` and friends; NestJS providers; Python and Go explicit wiring. Emit **synthetic call edges** (tagged `engine: framework-facts`, `precision: approximate` when multiple implementations are possible) into the call graph in 03.

The set of constructs an extractor models feeds the **capability matrix** in 03 section 11 (`dispatch_modeled`). An extractor that is detection-only models nothing, so all dispatch flags stay false for that framework.

### 4.4 Framework-provided protections
Facts the agent can cite, each with the sink contexts it protects and conditions that disable it:
- Template auto-escaping (Thymeleaf `th:text` versus `th:utext`, Razor `@` versus `Html.Raw`, Jinja2 autoescape, Django templates versus `|safe`, React JSX escaping versus `dangerouslySetInnerHTML`, Go `html/template` versus `text/template`).
- Parameterized data access (JPA named parameters, EF LINQ and `FromSqlInterpolated`, SQLAlchemy bound parameters, Sequelize replacements) versus string-built variants (`createNativeQuery(concat)`, `FromSqlRaw(concat)`, `text(f"...")`, ``sequelize.query(`...${x}`)``).
- CSRF and CORS configuration, header security middleware (Helmet, Spring Security headers), input validation annotations (`@Valid`, `[Required]`, Pydantic, class-validator).
These feed 05 (sanitizer KB) as context-aware protections. A protection that is configured off, or bypassed on this path, must be recorded as `protection_bypassed` with evidence.

## 5. Implementation approach

- Extractors are plugins: `extract(snapshot, module, catalog_entry) -> FrameworkFacts`.
- Implementation uses ripgrep/regex over source files first (cheap, no build, always available, so detection works even when no CPG could be built), and **Joern CPG queries** where cross-file resolution is needed (for example resolving the target of `app.use(router)`). Facts found only by text search carry `precision: name_based`.
- Facts stored as structured rows plus a per-module JSON document; the call-graph builder in 03 consumes `entrypoints` and `synthetic_edges`; the Evidence Pack builder copies relevant facts with `basis` rule ids and snippet citations.
- Each fact carries `evidence` (file, line) and `confidence`. Facts derived from inference rather than literal code are tagged `inferred`.
- **Unknown-framework fallback:** run generic entry-point heuristics (functions named `main`, handlers passed to known web-library registration calls, files under `controllers/routes/handlers`), tag all results `precision: name_based`, and record `unknown_indicators`.

## 6. Data model

```sql
CREATE TABLE module_frameworks (
  snapshot_id UUID, module_root TEXT, language TEXT,
  framework TEXT, version TEXT, confidence REAL,
  evidence JSONB, extractor TEXT,
  PRIMARY KEY (snapshot_id, module_root, framework)
);

CREATE TABLE entry_points (
  snapshot_id UUID, module_root TEXT,
  entry_id UUID PRIMARY KEY,
  kind TEXT,                -- route | main | job | consumer | cli | serverless
  http_method TEXT, route TEXT,
  handler_symbol TEXT, handler_file TEXT, handler_line INT,
  params JSONB,
  auth JSONB,               -- {"required":"none|authenticated|role|unknown","details":[...],"source":""}
  middleware JSONB,
  confidence REAL, inferred BOOLEAN
);

CREATE TABLE synthetic_edges (
  snapshot_id UUID, from_symbol TEXT, to_symbol TEXT,
  reason TEXT,              -- di | route_binding | middleware | interface_impl
  precision TEXT, evidence JSONB
);

CREATE TABLE framework_protections (
  snapshot_id UUID, module_root TEXT,
  protection TEXT, sink_contexts JSONB,
  enabled BOOLEAN, disabled_by JSONB, evidence JSONB
);
```

## 7. Observed-volume feedback loop
Log every detected framework and every `unknown_indicators` entry with counts per application. Produce a weekly report: top frameworks by finding volume, top unrecognized patterns. This tells the team which extractor to build next. Do not hard-code the list of priorities; use the data.

## 8. Edge cases
- Multiple frameworks in one module (Spring plus JAX-RS): run all matching extractors; merge entry points; dedupe by handler.
- Monorepo with different frameworks per module: detection is per module root.
- Framework version differences (Struts 1 versus 2, classic ASP.NET versus Core): detect version from manifests; catalog entries are version-aware.
- Generated or proxy-based routing (annotation processors, code generation): mark `inferred`.
- Programmatic route registration in loops or from config: record `route: dynamic` with low confidence rather than guessing.
- Auth implemented in a custom filter or interceptor not recognized: `auth: unknown` plus a note listing the suspected custom component.

## 9. Acceptance criteria
- Detection precision and recall measured on a labeled sample of at least 100 repos across the five languages; report per framework. Target 90% or better on the starting catalog.
- Detection works with and without a CPG (text-only mode produces the same framework list on fixtures; extracted facts are tagged `name_based` in text-only mode).
- Entry-point extraction tested per extractor against fixture apps (small sample apps checked into the test suite, at least two per priority extractor) with expected route lists.
- Auth extraction tests include order-sensitive middleware (route registered before versus after the auth middleware) and `[AllowAnonymous]` overrides.
- DI synthetic edges tested with interface having one and multiple implementations (multiple => `approximate`).
- Unknown-framework repos produce `name_based` precision, all `dispatch_modeled` flags false, and non-empty `unknown_indicators` without errors.
- Catalog is loaded from YAML; adding a new entry requires no code change (test with a dummy framework).
