---
title: "ADR-015: Trace Backend Selection — Langfuse (WP-H3-02)"
---

# ADR-015: Trace Backend Selection — Langfuse (WP-H3-02)

**Status:** Accepted
**Date:** 2026-04-23
**Context:** Harness phase H3, work package `WP-H3-02` — select a self-hosted trace/eval backend for Agentopia's OpenInference-tagged OTLP spans. Gates all subsequent H3 span-wiring PRs.

**Owning issues:**

- Umbrella: [ai-agentopia/agentopia-protocol#467](https://github.com/ai-agentopia/agentopia-protocol/issues/467) (`WP-H3-01` + `WP-H3-02`).
- Infra deployment half: [ai-agentopia/agentopia-infra#163](https://github.com/ai-agentopia/agentopia-infra/issues/163).
- Protocol-side groundwork shipped in PR [ai-agentopia/agentopia-protocol#487](https://github.com/ai-agentopia/agentopia-protocol/pull/487) (merged 2026-04-23).

---

## Decision

**Use Langfuse (MIT core) as Agentopia's self-hosted trace and eval backend.**

- Deploy via the community Helm chart `langfuse/langfuse-k8s` on the existing k3s cluster.
- Ingest spans over OTLP/HTTP at Langfuse's `/api/public/otel` endpoint.
- Depend only on the MIT-licensed root of the repository and the MIT Helm chart. The `ee/`, `web/src/ee/`, and `worker/src/ee/` modules — covered by the Langfuse Enterprise License and requiring a license key — are **out of scope** and will not be enabled.

Arize Phoenix was evaluated and rejected. Reasons below.

## Context

The Agent Harness emits OpenTelemetry spans tagged with OpenInference semantic conventions (`openinference.span.kind` ∈ `AGENT / CHAIN / LLM / TOOL`) and Agentopia-native attributes (`agentopia.run_id`, `agentopia.intent`, `agentopia.lane`, `agentopia.execution_class`, `agentopia.capability_class`). The protocol-side groundwork locked the emission contract in `bot-config-api/src/observability/`.

We need a backend that:

1. Runs self-hosted on our k3s cluster (bare-metal `server36`, `local-path` storage, ArgoCD + Helm-driven).
2. Ingests OTLP without custom adapters.
3. Does not pull the project into a non-OSS license. Our saved rule `feedback_oss_k8s_native.md` is explicit: "All deps must be OSS-licensed + K8s-native deployable. No vendor lock-in."
4. Supports our per-bot tenancy model without requiring a per-tenant deployment.
5. Offers first-class evaluations so the WP-H3-03 eval harness can land on the same surface.

Two candidates exist in this space: **Arize Phoenix** (the OpenInference reference UI) and **Langfuse** (agent-observability platform recently acquired by ClickHouse). Both accept OTLP; both self-host.

## Evaluation

### License (decisive constraint)

| | Phoenix | Langfuse |
|---|---|---|
| Root license | **Elastic License 2.0** (source-available, not OSI-approved) | **MIT** (core) + separate `ee/` Enterprise License |
| Forbids hosted service | Yes — ELv2 §3.2 | No (core) |
| License-key circumvention clause | Yes — ELv2 §3.3 | No (core) |
| Governance risk | Single vendor; ELv2 reserves feature-gating right | Vendor changed hands (ClickHouse acquired 2026-01-16), license unchanged to date |

Phoenix's ELv2 status materially conflicts with our OSS-only rule. The `no circumvention of license-key functionality` clause is a latent vendor-lock surface even though today's build has no license key. Langfuse's `ee/` modules are the inverse pattern: commercial features are cleanly isolated on disk, and the MIT-licensed core is a strict superset of what H3 needs.

**Source:** [Phoenix LICENSE](https://github.com/Arize-ai/phoenix/blob/main/LICENSE), [Langfuse LICENSE](https://github.com/langfuse/langfuse/blob/main/LICENSE), [Langfuse ee/LICENSE](https://github.com/langfuse/langfuse/blob/main/ee/LICENSE).

### OpenInference semantic convention support

| | Phoenix | Langfuse |
|---|---|---|
| Native OpenInference UI | Yes — reference implementation | No — consumes OTLP and re-buckets into Langfuse's own `trace`/`observation`/`generation` schema |
| `openinference.span.kind` as first-class UI facet | Yes | No (arrives as a custom attribute, visible but not a group-by dimension) |

This is the dimension on which Phoenix is stronger. We accept the trade-off because Agentopia's primary grouping concern is **by run** (`agentopia.run_id`) and **by capability class** — both custom attributes that Langfuse displays and filters on regardless. The `openinference.span.kind` attribute is still emitted (per WP-H3-01) and remains useful for exports to any future backend.

**Source:** [OpenInference spec](https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md), [Langfuse OTel docs](https://langfuse.com/docs/opentelemetry/get-started).

### Multi-tenancy in the OSS build

| | Phoenix | Langfuse |
|---|---|---|
| Multiple tenants / projects / orgs in free build | **No** — open backlog issue #10504 confirms "the underlying data layer is global" | **Yes** — multi-org + multi-project, no hard caps |
| Fine-grained project RBAC | N/A | Enterprise-only (out of scope) |
| Coarse project-level isolation | N/A | MIT core |

For the per-bot tenancy model this is decisive in Langfuse's favor. Using Phoenix would force us to either (a) run one Phoenix instance per tenant — erasing its single-pod operational advantage — or (b) accept that any VIEWER sees everyone's traces.

**Source:** [Phoenix issue #10504 (open, backlog, 2025-12-08)](https://github.com/Arize-ai/phoenix/issues/10504), [Langfuse self-hosting administration](https://langfuse.com/self-hosting/administration).

### Built-in evals

Both ship OSS evaluation surfaces. Phoenix has `arize-phoenix-evals` (MIT, in-tree). Langfuse has managed evaluators, LLM-as-a-judge, human annotations, and ragas integrations in the MIT core. For WP-H3-03 keyed off `RunContract`, Langfuse's native score API bound to trace IDs is the closer fit — scores attach directly to the trace surface we already emit.

**Source:** [arize-phoenix-evals](https://pypi.org/project/arize-phoenix-evals/), [Langfuse scores](https://langfuse.com/docs/scores/overview).

### Operational cost on k3s

| | Phoenix | Langfuse |
|---|---|---|
| Required services | 1 pod (app) + Postgres (bundled or external) | 6: web, worker, Postgres, ClickHouse, Redis/Valkey, S3/MinIO |
| Storage-heaviest component | Postgres | ClickHouse (default 3 replicas in the Helm chart) |
| Official Helm chart | Yes (`helm/` in repo) | Community chart `langfuse/langfuse-k8s` (MIT), actively maintained |

Phoenix is materially cheaper to run. Langfuse's six-service footprint is the principal operational cost of this decision. ClickHouse is the largest concern on `local-path` storage.

**Source:** [Phoenix Helm README](https://github.com/Arize-ai/phoenix/blob/main/helm/README.md), [langfuse-k8s](https://github.com/langfuse/langfuse-k8s).

---

## Rationale

The license constraint is non-negotiable per our saved rule. Phoenix's ELv2 status is disqualifying on that axis alone. Once license is fixed, Langfuse's multi-tenancy advantage compounds — our per-bot model needs tenant isolation in the default deploy, and that is not on Phoenix's roadmap. The remaining Phoenix advantages (OpenInference UI native, single-pod ops) are mitigatable in ways the license and tenancy gaps are not:

- **OpenInference UI** — our primary grouping is `agentopia.run_id`, which both backends expose. `openinference.span.kind` stays on the wire for any future export.
- **Single-pod ops** — addressed by explicit non-HA Helm values for our dev/staging tier (see constraints below). Production HA sizing is out of scope for this ADR.

## Deployment constraints (infra half, issue #163)

These are binding on the infra rollout PR and must be reflected in the ArgoCD Application that deploys Langfuse:

1. **Chart repo:** `langfuse/langfuse-k8s`. Pin to a specific chart version; do not track `HEAD`.
2. **MIT-only:** `langfuse.licenseKey` MUST remain unset. Enterprise features MUST NOT be enabled in any environment by default.
3. **Right-sized subcharts for k3s:**
   - `clickhouse.replicaCount: 1` (default is 3; will not schedule on our cluster otherwise).
   - `redis.architecture: standalone` (no sentinel).
   - ~~`postgresql.architecture: standalone`~~ — **superseded 2026-04-24**: `postgresql.enabled: false`. Langfuse uses the cluster shared system Postgres with a dedicated `langfuse` database and `langfuse_app` role. DSN from Vault at `secret/langfuse/postgres-dsn`. See [H3 Observability Production Design §5.4](../architecture/harness-control/h3-observability-production-design.md) for the full boundary spec.
   - `s3.deploy: true` with MinIO standalone (no distributed mode).
   - Storage: ClickHouse, Redis, MinIO PVCs use the cluster default `local-path`. No Postgres PVC (external).
4. **Namespace:** deploy into the existing `agentopia` namespace (per CLAUDE.md rule: only touch this namespace).
5. **Ingress:** Traefik (no ALB). Internal-only at first; no public ingress.
6. **Ingest URL:** expose `/api/public/otel` via cluster-internal Service. Do NOT expose externally in this phase.
7. **Secrets:** Postgres DSN is Vault-managed (`secret/langfuse/postgres-dsn`) — not chart-generated. ClickHouse + MinIO passwords may be chart-generated on first install for non-prod; Langfuse encryption key and NEXTAUTH secret are Vault-managed. See [production design §10.4](../architecture/harness-control/h3-observability-production-design.md) for the full secret surface.

## Protocol-side implications

`bot-config-api`'s observability module already emits OTLP/HTTP and reads `OTEL_EXPORTER_OTLP_ENDPOINT` per WP-H3-01. Once Langfuse is deployed, configuration is a single env-var set pointing at the cluster-internal Langfuse ingest URL — **no code change is required to start ingesting**. This was the entire point of the backend-agnostic groundwork round.

## Consequences

- Agentopia gains a self-hosted, OTLP-native, multi-tenant, MIT-licensed trace surface with built-in evals that the WP-H3-03 harness can target directly.
- We accept a six-service footprint in the cluster — materially heavier than Phoenix's single pod. ClickHouse will be the dominant operational surface.
- We lose `openinference.span.kind` as a native UI group-by dimension. Spans still carry the attribute; downstream backends that honor it remain compatible.
- License reciprocity risk is minimized: MIT core has no copyleft or circumvention clauses; `ee/` modules are opt-in and isolated.
- **Reversibility:** because protocol-side emission is OTLP with OpenInference attributes, switching to Phoenix or any other OTLP-compatible backend in the future is a deployment change, not a code rewrite. This ADR is therefore a **deployment-level commitment**, not a protocol-level one.

## Explicit non-goals (this ADR)

- Picking chart versions — the infra PR on issue #163 pins exact versions.
- Span wiring at gateway / Temporal / A2A / relay / llm-proxy boundaries — those are subsequent PRs under the #467 umbrella.
- Production HA sizing — this ADR ships with dev/staging-tier values only.
- Eval harness design — **WP-H3-03**, separate issue [#469](https://github.com/ai-agentopia/agentopia-protocol/issues/469).
- Checkpoint-policy / ApprovalSidecarWorkflow — **WP-H3-04..07**, separate issue [#468](https://github.com/ai-agentopia/agentopia-protocol/issues/468).
- `agentopia-core` instrumentation — tracked under the core-side work package, not here.
- A public ingress for the Langfuse UI — internal-only for now.

## References

- Harness implementation plan: `milestones/agent-harness-implementation-plan.md` §6 Phase H3.
- Umbrella issue: [ai-agentopia/agentopia-protocol#467](https://github.com/ai-agentopia/agentopia-protocol/issues/467).
- Infra deployment issue: [ai-agentopia/agentopia-infra#163](https://github.com/ai-agentopia/agentopia-infra/issues/163).
- Groundwork PR: [ai-agentopia/agentopia-protocol#487](https://github.com/ai-agentopia/agentopia-protocol/pull/487).
- OpenInference semantic conventions: <https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md>.
- Langfuse docs: <https://langfuse.com/self-hosting>.
