# 13: Infrastructure, Deployment, Scaling, and Observability (OpenShift)

Author: Surya

Read `00-MAIN-DESIGN.md` first. Cross-cutting. The deployment platform is **OpenShift**. Names below use OpenShift and Kubernetes terms; confirm versions and cluster policies in phase 0.

## 1. Open questions to resolve in phase 0
1. OpenShift version, cluster topology (shared or dedicated), namespace/project model, and whether memory-heavy JVM workers with large heaps and ephemeral disks are allowed (node pools, `LimitRange`, `ResourceQuota`).
2. Network connectivity: from the cluster to CxSAST on-prem, to source hosts, and to the LLM gateway; egress controls (EgressFirewall, EgressIP, or proxy).
3. Available managed services: PostgreSQL, object storage (S3 compatible), secret manager (and how it integrates with OpenShift), queue/workflow (Temporal), identity provider for SSO (OpenShift OAuth or corporate SSO for the dashboard).
4. Artifact proxy availability for dependencies (Maven, NuGet, npm, PyPI, Go) if supplements need restores.
5. Environments: dev, staging, production, and who operates each.
6. Capacity and cost limits for compute and LLM spend.
7. Image registry, image signing and scanning policy, and GitOps tooling (OpenShift GitOps/Argo CD or equivalent).

## 2. OpenShift-specific requirements (apply to every workload)

- **Random non-root UID.** Under the default `restricted-v2` SCC, OpenShift assigns each pod a random UID from the namespace range, with GID 0. Images must **not** rely on a fixed user:
  - Do not use `USER <fixed uid>` that the app depends on; do not write to paths owned by a specific UID.
  - Make writable directories (`/workspace`, `/tmp`, CPG cache, Joern home, git config dir) group-writable by GID 0: `chgrp -R 0 <dir> && chmod -R g=u <dir>`, and set `HOME` to a writable path (for Joern, git, and JVM temp files).
  - Set `umask 002` in entrypoints that create files shared across containers.
- **Security Context Constraints.** Run everything under `restricted-v2` (no privileged, no host mounts, dropped capabilities, seccomp `RuntimeDefault`, no privilege escalation). If any component needs more (for example user namespaces for sandboxing tools), it gets a dedicated service account with a narrowly scoped, reviewed custom SCC; document the reason. **Do not** grant `anyuid` or `privileged`.
- **Untrusted-code workers (fetch, index, Joern, supplements)** run in their own namespace/project with: default-deny NetworkPolicy, per-class egress allowances, read-only root filesystem where the tool allows it, `emptyDir` scratch with `sizeLimit`, CPU/memory limits, no service account token mounted (`automountServiceAccountToken: false`), and a dedicated node pool/taint if the cluster policy allows. Consider RuntimeClass with stronger isolation (for example Kata) if available; validate in phase 0.
- **Networking.** Use `NetworkPolicy` (default deny, explicit allow). External egress to CxSAST, source hosts and the LLM gateway through approved egress paths (EgressFirewall rules by DNS name or CIDR, or an egress proxy). The dashboard is exposed by a `Route` (TLS edge or re-encrypt) behind SSO; no other component gets a Route.
- **Storage.** `emptyDir` for clone and CPG scratch (sized per tier); `PersistentVolumeClaim` only where required (Temporal DB if self-hosted, local CPG cache SSD via local storage class if available). Remember that scratch is lost when a pod is rescheduled; builds must be retryable.
- **Config and secrets.** `ConfigMap` for versioned YAML; secrets from the organization's secret manager (External Secrets Operator or Vault injector) rather than long-lived `Secret` objects where policy allows.
- **Resources and JVM.** Set `requests` = `limits` for memory on Joern workers. Set the JVM heap explicitly (`-Xmx` around 70 to 75% of the pod memory limit, leaving headroom for off-heap, metaspace and tooling) and `-XX:+ExitOnOutOfMemoryError` so an OOM fails the pod cleanly; map that to `index_failed_oom`. Do not rely on container-aware defaults alone.
- **Quotas and priority.** `ResourceQuota` and `PriorityClass` so the index pool cannot starve other tenants; protect the cluster from a runaway repo with limits at both pod and namespace level.
- **Autoscaling.** HPA on custom metrics (queue depth) via KEDA or the Custom Metrics Adapter; cluster autoscaling by machine sets if available.
- **Image supply chain.** Build in the pipeline, scan, generate SBOM, sign, and pull only from the approved registry. Pin base images by digest.
- **Day-2.** Use OpenShift monitoring (Prometheus, Alertmanager) and logging (Loki or the corporate stack) rather than self-installed copies where possible; service monitors for each component.

## 3. Logical deployment

```
                          +------------------+
                          |  Dashboard UI    |  (Route -> SSO)
                          +--------+---------+
                                   |
                          +--------v---------+      +------------------+
                          | Read-only API    +----->+ PostgreSQL       |
                          +------------------+      | (+ read replica) |
                                                    +--------+---------+
  +-----------+     +-------------------+                    |
  |  CxSAST   |<--->| Ingestion service |--------------------+
  |  on-prem  |     +-------------------+                    |
  +-----------+     +-------------------+                    |
        ^           |   Orchestrator    |<-------------------+
        |           |   (Temporal)      |
        |           +----+------+-----+-+
        |                |      |     |
        |        +-------v-+  +-v-----v----+   +----------------+
        |        | fetch   |  | index      |   | agent workers  |--> LLM gateway
        |        | workers |  | workers    |   +--------+-------+    (Claude, Gemini)
        |        +---------+  | (Joern)    |            |
        |                     +-----+------+            |
        |                           | code-intel service (internal)
        |                     +-----v------+
        +---------------------+ writeback  |
                              | workers    |
                              +------------+
  Object storage: snapshots cache, CPGs, evidence packs, prompt/response archive
  Secret manager, identity provider, metrics/logging stack
  Namespaces: platform (api, orchestrator, agents, writeback) | untrusted-compute (fetch, index, code-intel)
```

## 4. Components and sizing guidance
Initial sizing is a starting point; validate with load tests (06 acceptance criteria).

- **Ingestion service:** small, stateless, 2 replicas; leader election or workflow-scheduled polling to avoid duplicates.
- **Orchestrator (Temporal):** managed or self-hosted cluster with its own database; worker processes per task queue (`fetch`, `index`, `context`, `llm`, `writeback`, `pattern`).
- **Fetch workers:** I/O bound, modest CPU, `emptyDir` sized for clones (for example 50 to 100 GB scratch) with cleanup; autoscale on queue depth.
- **Index workers (Joern):** memory-heavy. Node pools by tier: S/M (8 to 16 GB), L (32 GB), XL (64 GB), CPU 4 to 8 cores, scratch for CPGs. Autoscale on queue depth with a max cap to protect cost. Spot or preemptible capacity only with checkpoint-tolerant jobs (builds are retryable).
- **Code-intelligence service:** loads CPGs on demand; warm pool with TTL; co-locate with the CPG cache (local SSD) to avoid repeated downloads; horizontal scale by consistent hashing on snapshot ID to maximize cache hits. Also builds Evidence Packs and runs ripgrep gap-fill tools.
- **Agent workers:** light CPU, outbound to the LLM gateway; concurrency set by provider quotas, not CPU.
- **Writeback workers:** few replicas, strict rate limiting.
- **PostgreSQL:** primary plus replica; partitioned event tables; materialized views refreshed by a scheduled job; connection pooling; backups with point-in-time recovery.
- **Object storage:** buckets for `snapshots` (short TTL), `cpg` (medium TTL), `evidence` and `llm-archive` (policy TTL), with lifecycle rules and encryption.
- **Cache layers:** snapshot cache and CPG cache with LRU and size budgets (02, 03); Redis optional for rate limiting and short-lived locks.

## 5. Scaling model (back-of-envelope; validate)
- Volume: about 1 million findings per quarter, roughly 80,000 per week on average (about 11,000 per day if spread evenly over 90 days); plan for bursts at 3x.
- Dedupe and reuse (fingerprint includes the pack hash) will reduce LLM calls substantially (Recurrent findings and cross-project duplicates); measure the reuse rate in the first weeks and size the `llm` queue from the measured **unique** fingerprint rate, not the raw finding rate.
- Heavy work is per **snapshot** (index build), not per finding; group findings by snapshot to amortize builds. Scans per day and distinct snapshots per day drive index worker capacity.
- LLM throughput: size by (unique fingerprints per day) x (average tokens per run) against provider rate limits; keep the queue buffered; configure per-provider concurrency and token-per-minute budgets. Prompt caching and batch APIs (07 section 4) change the effective cost; check provider features and measure.
- Budget guardrails: daily and monthly spend caps per provider with alerts and an automatic pause on the `llm` queue when the cap is reached.

## 6. Environments and release
- Environments: `dev` (mock CxSAST and mock LLM), `staging` (real CxSAST test projects, real LLM with low quotas, dry-run writeback), `prod` (feature flags, project allowlist, dry-run until enabled). Separate projects/namespaces per environment.
- CI/CD: build, test (unit, integration with mocks, fixtures, security suite), image scan, SBOM, sign, deploy via GitOps. Database migrations via a migration tool with backward-compatible steps.
- Versioned artifacts: services, prompts, playbooks and decision tables, sanitizer KB, framework catalog, calibration tables, Joern version. All pinned and recorded in verdicts.
- Config: versioned YAML per environment plus runtime flags (kill switch, dry-run, allowlist, sample rates) that can change without redeploy.
- Rollback: previous image and config versions retained; KB, rule and prompt changes gated by the eval harness (11) and reversible.

## 7. Observability
- **Metrics (Prometheus-style):** queue depth, task latency and failure rate per activity, index build duration and memory peak by tier, cache hit rates (snapshot, CPG, prompt cache), LLM tokens and cost by provider/tier, validator downgrade counts by rule, writeback success and latency, CxSAST API error rate, ingestion lag, findings by lifecycle state, `source_match_score` distribution, no-CPG rate by language and reason, masking violations.
- **Logging:** structured JSON logs with correlation IDs (`scan_id`, `finding_id`, `run_id`); redacted of secrets and code bodies; centralized.
- **Tracing:** distributed traces across workflow activities, with spans for fetch, index, context, agent, validate, writeback.
- **Dashboards (ops):** pipeline health, backlog, cost, errors; the product dashboard is the separate feature in 10.
- **Alerts:** ingestion lag over threshold; backlog age; failure spikes by reason; worker OOM kills; LLM quota exhaustion or error spikes; writeback failure rate; CxSAST latency; spend cap near; kill switch toggled; injection-suspected spike; stuck findings; masking violation (page immediately).
- **SLOs (initial targets to agree):** High/Critical findings reach a written proposal within X hours of scan completion (suggest 4 hours); 95% of other findings within 24 hours; writeback verification success above 99%; dashboard data freshness under 5 minutes.

## 8. Reliability
- Idempotent activities and durable workflows (06); retries with backoff; circuit breakers on CxSAST, source hosts, and LLM providers.
- Provider failover: if one LLM provider is down or throttled, route to the other where policy allows (configurable per tier; second-opinion independence rule still requires a different model than the primary for that finding).
- Backups and recovery objectives: database RPO under 15 minutes, RTO under 4 hours (agree actual values); object storage versioning for evidence and audit; periodic restore tests.
- Graceful degradation order when under stress: pause low-priority findings, stop reuse invalidations, keep High/Critical flowing; never drop findings.
- Capacity limits protect CxSAST: ingestion and writeback rate limits adjustable at runtime.

## 9. Cost controls
- Dedupe and reuse (06); model tiering (06, 07); prompt caching and batch APIs where supported; snapshot-level index amortization; short TTLs on large caches; scale-to-zero for idle worker pools; spot capacity for index builds where safe.
- Per-finding and per-application cost accounting on the dashboard (10).
- Hard caps and alerts per day and month; a "cost brake" mode that routes only High/Critical to the strong tier and sends the rest to the cheap tier or defers them.

## 10. Operations runbooks (to write during phase 1)
Kill switch and dry-run toggling; draining queues; replaying dead-letter items; reindexing a snapshot; rotating credentials; handling CxSAST outages; handling LLM provider outages; reverting writebacks (12); onboarding a new application or reviewer mapping; adding a framework to the catalog; adding a sanitizer KB entry or pattern rule; updating calibration tables; handling a pod stuck on SCC admission.

## 11. Acceptance criteria
- Infrastructure defined as code; a fresh environment can be created from scratch in staging via pipeline.
- **All workloads admit under `restricted-v2`** with a random UID: fetch, index (Joern), code-intel, agent and writeback pods start, write to their scratch paths and run a full fixture finding end to end. No `anyuid` or `privileged` grants exist.
- Network policy tests confirm default-deny and per-class egress allowances (12).
- Load test results documented: sustained 3x average daily volume with mocked LLM/CxSAST; autoscaling behavior and queue drain times recorded.
- Index worker memory limits and JVM heap settings verified with the largest allowed tier and an oversized repo (fails cleanly with `index_failed_oom`, no node-level impact).
- Metrics, logs, traces, and alerts exist for each item in section 7; synthetic alert drills performed (for example kill switch toggle, LLM outage simulation, CxSAST outage simulation).
- Backup and restore drill completed for the database; object storage lifecycle policies verified.
- Cost report available per day by provider and tier; spend cap pause demonstrated in staging.
- Runbooks reviewed by an operator who did not write them.
