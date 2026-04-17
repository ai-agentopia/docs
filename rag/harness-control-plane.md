---
title: "Agentopia Harness and Control-Plane Architecture"
status: "ACTIVE"
decision-date: "2026-04-17"
---

# Harness and Control-Plane Architecture

## 1. What a Harness Control Plane Is

Modern production agent systems separate two distinct concerns that are often conflated in early implementations:

- **Configuration and policy authority** — who declares what a bot is, what tools it may use, what routing rules apply, what sources of truth are authoritative, and what the lifecycle of a run looks like. This is the harness or host layer.
- **Runtime execution** — the actual per-request work: receiving a user message, assembling context, invoking tools, calling the model, returning a response. This is the runtime or data plane.

Industry reference points:

**MCP Architecture** defines a three-tier model with Host, Client, and Server layers. The host is described as "the container and coordinator that manages security policies, user authorization, context aggregation, and AI coordination." Servers "expose focused capabilities and should be trivial to build." The host bears the orchestration complexity, not the servers. Ref: [MCP Architecture Specification](https://modelcontextprotocol.io/specification/2025-06-18/architecture)

**Google Vertex AI Agent Engine** separates the managed platform (runtime, sessions, memory bank, observability, security) from the application logic that developers own. The platform "handles production infrastructure so developers can focus on creating applications." Ref: [Vertex AI Agent Engine Overview](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)

**OpenAI Agents SDK** delegates orchestration to "manager agents" or host-level logic that coordinates specialists. Agents are first-class tools; the SDK provides composition primitives but the orchestration policy is developer-owned at the host layer. Ref: [OpenAI Agents SDK — Multi-Agent Orchestration](https://openai.github.io/openai-agents-python/multi_agent/)

**Anthropic Context Engineering** positions context assembly as a shared-system concern — not purely agent logic, not purely platform logic — requiring explicit curation of system prompts, tool definitions, retrieved context, and memory across the full attention budget. Ref: [Anthropic — Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents/)

**The consensus pattern**: A harness control plane declares policy, manages lifecycle, and enforces authorization. A runtime execution plane executes per-request within the policy constraints the harness declared. These two layers must not collapse into one service or become coupled.

---

## 2. Agentopia Five-Plane System Decomposition

The Agentopia system operates across five planes. The earlier architecture documents identified four planes (Control, Operational State, Knowledge, Memory). This document adds a fifth — **Runtime Execution** — which was implicit in the gateway but not explicitly named. Naming it matters because it clarifies the ownership boundary between the harness and the runtime.

| Plane | Role | Primary Owner | Sources of Truth |
|---|---|---|---|
| **Control Plane** | Bot identity, policy, lifecycle, tool registry, scope/auth authority | `bot-config-api` | Postgres, K8s CRDs (argocd ns), K8s Secrets/ConfigMaps |
| **Runtime Execution Plane** | Per-request: context assembly, tool invocation, model call, response | `gateway` + OpenClaw embedded runner | Stateless — reads policy from Control Plane |
| **Operational State Plane** | Live system state: workflow status, deployment state, active config | `governance-bridge` (gateway plugin) → Temporal, K8s API, GitHub API, `bot-config-api` | Live APIs at query time |
| **Knowledge Plane** | Versioned domain knowledge from source documents | `agentopia-super-rag` (Qdrant), `agentopia-knowledge-ingest` (ingestion, migrating) | Source systems → ingest pipeline → Qdrant `kb-{scope_hash}` collections |
| **Memory Plane** | User-session episodic facts and entity graph | `mem0-api` | Qdrant `agentopia_memory` collection, Neo4j |

**Isolation invariant across all planes**: Each plane has a single authoritative source of truth. The Runtime Execution Plane does not persist state — it reads from the Control Plane's policy declarations and writes results to the Memory Plane. The Knowledge Plane and Memory Plane do not overlap. The Operational State Plane does not index into Qdrant.

---

## 3. Responsibility Taxonomy

The fifteen responsibilities listed below span the full system. Each is assigned to its recommended owner, with the current owner noted where it differs from the target.

| Responsibility | Recommended Owner | Current Owner | Gap? |
|---|---|---|---|
| Bot identity / spec | Control Plane (`bot-config-api`) | `bot-config-api` ✓ | None — provisioned via `/api/v1/bots/deploy`, stored in ArgoCD CRD + K8s |
| Route policy | Control Plane (`bot-config-api`) | Gateway (`configmap-config.yaml`, hardcoded) | **Gap** — route policy is currently a hardcoded plugin list in gateway; it should be declared in the control plane and consumed by the runtime |
| Source-of-truth policy | Control Plane (`bot-config-api`) | `bot-config-api` (partial: `knowledge-scopes`, `scope_ingest_mode`) | Partial — formalization needed; not yet a first-class registry |
| Tool registry / bindings | Control Plane (`bot-config-api`) | Gateway extensions (implicit `ALL_TOOLS` constant in `governance.py`) | **Gap** — tool bindings are implicit code constants; no configurable per-bot tool registry |
| Scope / tenant auth policy | Control Plane (`bot-config-api`) | `bot-config-api` ✓ | None — enforced via `check_governance_access()` and role registry (`w0_actor_bindings`) |
| Run / session lifecycle metadata | Control Plane (`bot-config-api`) | Not owned | **Gap** — no service tracks session lifecycle (started, budget, ended) |
| Rollout flags | Control Plane (`bot-config-api`) | Partial (`scope_ingest_mode` per scope only) | **Gap** — no general per-bot feature flag / rollout flag mechanism |
| Eval policy | Control Plane (`bot-config-api`) | Not owned | **Gap** — eval thresholds (nDCG@5 floor, misroute rate target) are global, not per-bot |
| Deployment metadata | Control Plane (`bot-config-api`) | `bot-config-api` ✓ | None — ArgoCD Application CRD spec fully owned |
| Actual tool execution | Runtime Execution Plane (`gateway`) | `gateway` ✓ | None |
| Context assembly | Runtime Execution Plane (`gateway`) | `gateway` `before_agent_start` hooks ✓ | None — knowledge-retrieval, mem0-api, governance-bridge, wf-bridge hooks inject context |
| Model invocation | Runtime Execution Plane (`gateway`) | `gateway` → Anthropic API (Vault key) ✓ | None |
| Knowledge ingestion | Knowledge Plane (`agentopia-knowledge-ingest`, migrating to Pathway/mixed) | `agentopia-knowledge-ingest` | In progress — Pathway migration (Phases 3–7) |
| Retrieval serving | Knowledge Plane (`agentopia-super-rag` + Qdrant) | `agentopia-super-rag` ✓ | None |
| Memory capture / retrieval | Memory Plane (`mem0-api`) | `mem0-api` ✓ | None |

**Three gaps are architectural**: route policy, tool registry, and session lifecycle. The remaining gaps (rollout flags, eval policy) are product maturity gaps that do not block the core architecture.

---

## 4. bot-config-api: Current Role

The following is based on direct code audit. Claims below are grounded in specific files and line numbers.

**Self-identification**: `bot-config-api` defines itself as `control-plane-api` at `agentopia-protocol/bot-config-api/src/main.py:570`. This is accurate to its current function — it is already a harness control plane in structure, though some roles are not yet formalized.

**What it currently owns:**

- **Bot provisioning and lifecycle**: `POST /api/v1/bots/deploy` runs an 8-step deploy job (SOUL generation → K8s Secrets + ConfigMaps → ArgoCD Application CRD → role binding → Telegram registration → pod readiness poll). Implementation: `routers/deploy.py`.
- **ArgoCD Application CRD management**: Creates CRDs directly via K8s API in the `argocd` namespace. Sets image, Helm values, sync policy, and ArgoCD Image Updater directives. Implementation: `services/k8s_service.py:215–390`.
- **Knowledge scope bindings**: Stores `client_id` + `knowledge_scopes` per bot in Postgres (`bot_knowledge_bindings`) and syncs to K8s annotation. Implementation: `services/kb_bindings_service.py`.
- **Workflow actor→role registry**: Maintains `w0_actor_bindings` table (Postgres) mapping actor IDs to role keys. Implementation: `services/workflow_service.py`, `services/role_registry.py`.
- **Governance policy enforcement**: `check_governance_access()` enforces role-based access per tool. 28 governance tools registered; access depends on role (orchestrator → all, reviewer → read-only, worker → code-only). Implementation: `routers/governance.py`.
- **MCP configuration**: Generates `mcpBridge` / `mcpSecrets` config per bot; stored in K8s Secrets. Implementation: `routers/deploy.py`.
- **Governance API** (GitHub operations): Executes GitHub REST API calls (issues, milestones, PRs, branches). `bot-config-api` is the caller — not gateway. Implementation: `routers/governance.py`.

**What it currently does NOT own:**

- **Route policy per query family** — the gateway's plugin chain (`configmap-config.yaml:123–182`) is hardcoded. `bot-config-api` does not declare which plugins fire for which query families.
- **Per-request routing decisions** — all per-request routing is done by gateway's `before_agent_start` hooks and the LLM's own tool-use decisions.
- **Session/run lifecycle tracking** — no service currently tracks session-started, context-budget, or session-ended metadata.
- **Formal tool registry** — `ALL_TOOLS` in `governance.py` is a code constant, not a per-bot configurable binding.
- **Eval policy per bot** — eval thresholds are not stored in bot-config-api or any service; they are defined only in eval scripts and this documentation.

---

## 5. Runtime Execution Plane: Gateway's Role

The gateway is a **stateless runtime execution plane**. It has no authoritative state of its own. Its behavior on any request is determined by:

1. Policy it reads from the Control Plane (plugin configuration in bot Helm values, sourced from bot-config-api deploy)
2. Context it assembles at request time (from Knowledge, Memory, and Operational State planes)
3. Model invocation (Anthropic API, Vault key per bot)

**Plugin chain** (from `configmap-config.yaml:123–182`): `knowledge-retrieval`, `mem0-api`, `relay`, `wf-bridge`, `governance-bridge`, `mcp-bridge`, `telegram`. The chain is currently hardcoded per bot type. Bot-specific plugin selection is set at deploy time via Helm values.

**Context assembly hooks** run before the LLM call (`before_agent_start`):
- `knowledge-retrieval`: calls `POST /api/v1/knowledge/search`, injects `<domain-knowledge>` block
- `mem0-api`: calls memory search, injects `<memory>` block
- `governance-bridge`: queries role, injects role-appropriate tool hints
- `wf-bridge`: detects `/wf` prefix, injects workflow role context

**What gateway does not do**:
- Does not classify query intent — no intent router exists in gateway today
- Does not enforce authorization — calls backend, returns 403 to LLM on denial
- Does not persist any state — all state lives in mem0-api, Postgres, or Qdrant

The gateway's stateless nature is a design property that should be preserved. Pushing state or policy decision logic into gateway would couple the runtime to the control plane, violating the separation.

---

## 6. Recommended Ownership Split

Three options for bot-config-api's evolution are evaluated below. The recommendation follows.

### Option A — Remain a narrow config CRUD service

bot-config-api continues to own only bot provisioning and K8s lifecycle, with no harness policy role.

**Assessment**: Not recommended. `bot-config-api` already owns governance policy enforcement, role registry, and knowledge scope bindings. Treating it as pure CRUD would require moving those responsibilities elsewhere. The current architecture is already past this option.

### Option B — Expand as harness control plane (Recommended)

bot-config-api is formally positioned as the harness control plane authority. It retains all current responsibilities and adds:

1. **Route policy registry** — bot-config-api declares a per-bot route policy: which plugin chain is active, which query families map to which plugins. Gateway reads this at bot startup (embedded in Helm values or fetched at init). This replaces hardcoded plugin list in `configmap-config.yaml`. **This is the most important gap to close.**

2. **Formal tool registry** — tool bindings per bot are declared as config (which tools are enabled, which are restricted to which roles) rather than implied by code constants. The 28 governance tools, MCP tools, and relay tools should be representable as a per-bot binding set.

3. **Eval policy per bot** — bot-config-api stores eval thresholds (nDCG@5 floor, misroute rate target, freshness SLO) per bot or per scope. This enables per-tenant eval configuration and audit trails.

4. **Session lifecycle metadata** (lower priority) — bot-config-api or a lightweight sidecar tracks session lifecycle events. This does not mean conversation history (that is mem0-api) — it means: session-started, context-budget-remaining, session-ended, tools-invoked-in-session. Used for observability and billing, not for routing decisions.

**What Option B does NOT change**:
- Context assembly remains in gateway
- Model invocation remains in gateway
- Memory capture/retrieval remains in mem0-api
- Retrieval serving remains in super-rag + Qdrant
- Knowledge ingestion remains in knowledge-ingest (migrating to Pathway/mixed)

### Option C — Split bot-config-api into harness + governance (Future consideration, not recommended now)

As bot-config-api grows, it now owns provisioning, governance (28 GitHub tools + GitHub REST calls), relay API, workflow management, knowledge bindings, and MCP config. These are distinct concerns.

A future split could separate:
- `agentopia-harness` — pure harness: bot spec, route policy, tool bindings, auth policy, scope management, eval policy, session lifecycle
- `agentopia-governance` — GitHub operations, workflow management, relay — essentially the operational state service API

**Assessment**: Architecturally sound as a future direction, but premature now. The boundary is not causing problems in the current system. This should be revisited when bot-config-api's surface area makes its ownership model unclear to new contributors or when governance/relay SLA requirements differ materially from provisioning SLAs.

---

## 7. Critical Gap: Route Policy

The most important architectural gap is that **route policy is not owned by the control plane**. This is the root cause of the routing misclassification problem described in README.md (Gap 1).

**Current state**: Gateway's plugin list (`configmap-config.yaml:123–182`) is a static configuration deployed with the bot Helm chart. There is no per-query intent classification. Plugins fire based on keyword patterns (`NON_KB_PATTERNS`, `NON_MEMORY_META_PATTERNS`) that have incomplete coverage.

**Target state after Phase 1**:
1. The intent router (Phase 1 implementation) classifies each query into a family at request time. This is a runtime concern — it lives in the gateway execution plane.
2. bot-config-api declares the **route policy**: which plugin is the primary handler for each query family, and which plugins are suppressed. This is a config concern — it lives in the control plane.
3. The intent router at runtime reads the query family, looks up the route policy (embedded in bot config at startup), and enforces it via `event.coordination`.

**Why this separation matters**: An intent router without a route policy registry is just a classifier with no declared consequence. The policy (operational_state → governance-bridge, knowledge_query → knowledge-retrieval, etc.) must be declarative and auditable — not hardcoded in multiple plugin implementations.

---

## 8. Interaction with RAG Architecture

The harness control plane and the knowledge/retrieval planes interact at three points:

**1. Scope binding** — bot-config-api declares which knowledge scopes a bot may access (`bot_knowledge_bindings` in Postgres). This is a harness-level concern. The knowledge retrieval plugin reads this binding at query time to scope its Qdrant search.

**2. Route policy for knowledge_query** — bot-config-api's route policy (when formalized) will declare that `knowledge_query` → `knowledge-retrieval`. This is distinct from how knowledge-retrieval operates internally (Pathway ingestion, Qdrant HNSW index, chunk scoring). The harness declares routing; the knowledge plane owns retrieval quality.

**3. Ingest mode flag** — `scope_ingest_mode` (currently partial — see `migration-plan.md`) is a harness-level flag: it declares that a scope's knowledge data plane is managed by Pathway (single-publisher invariant). This is correctly owned by the control plane, not by the knowledge service itself.

**The harness does not own retrieval quality**. nDCG@5, chunk scoring, freshness SLOs — these are knowledge-plane concerns evaluated by `agentopia-super-rag`. The harness declares who may query and which scopes; the knowledge plane guarantees quality within those constraints.

---

## 9. Open Questions

The following questions are unresolved and must be answered before the harness architecture can be considered final.

1. **Route policy format**: What is the schema for a per-bot route policy declaration? Should it be a structured object in the bot Helm values, a Postgres table, or a K8s CRD? The format affects how gateway reads it at startup and how it is versioned.

2. **Tool registry schema**: What is the minimum viable schema for a per-bot tool binding? 28 governance tools + N MCP tools + relay tools must be representable. Should this be additive to the current role-based check model or replace it?

3. **Intent router ownership**: The Phase 1 intent router runs at request time — is it a gateway component (stateless, per-request) or a control-plane component (policy-driven, configured per bot)? The recommendation is: classifier logic lives in gateway; classification policy (confidence thresholds, family definitions) lives in bot-config-api. This split is not yet confirmed.

4. **Session lifecycle metadata ownership**: Is this bot-config-api, gateway sidecar, or a separate telemetry service? The answer depends on whether session lifecycle is a control-plane concern (affects policy decisions) or purely an observability concern (affects billing and monitoring).

5. **Option C trigger conditions**: At what growth point does bot-config-api warrant a harness/governance split? The current recommendation (Option B) should specify a revisit condition — e.g., governance endpoint count exceeds N, or the SLA requirements for provisioning vs governance diverge.

6. **Eval policy per bot**: Should eval policy (nDCG@5 floor, misroute rate target) be per-bot or per-scope? The answer affects whether it belongs in the bot's config or the scope's config.

---

## Relationship to Other Documents

| Document | Relationship |
|---|---|
| `architecture.md` | Defines the 5-plane system, query family model, and per-plane source-of-truth rules |
| `implementation-plan.md` | Phase 1 (intent router) + Phase 2 (operational state plane) directly address route policy gap |
| `README.md` | Initiative overview; harness workstream added alongside RAG workstream |
| `evals-and-slos.md` | Route correctness SLOs interact with harness route policy; contamination SLOs interact with plane isolation invariant |
| `migration-plan.md` | Knowledge-plane migration; `scope_ingest_mode` flag is a harness-level control |
