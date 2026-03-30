---
title: "Agentopia Architecture Overview"
---

# Agentopia Architecture Overview

> High-level architecture for the Agentopia autonomous multi-bot delivery platform.
> Aligned with: `docs/milestones/p1-web-app-primary-dual-lane-mvp.md`
> Last updated: 2026-03-30

---

## 1. P1 Target Architecture (Web-App Primary)

> **Note:** P1 implementation is in progress (2026-03-30). The unified web app, auth/session layer, Communication API, and Workflow API all exist in code. Workflow-scoped conversation is being added. The `start_delivery` model tool has been removed (Boundary 1 enforced). See P1 milestone doc for remaining scope.

```mermaid
graph TB
    subgraph User["User (Browser)"]
        APP["Unified Web App<br/>(Communication | Workflow | Admin)"]
    end

    subgraph External["External Services"]
        TG["Telegram API<br/>(optional secondary)"]
        GH["GitHub API"]
        LLM["Anthropic Claude API"]
    end

    subgraph K8s["Kubernetes Cluster"]
        subgraph ControlPlane["Control Plane + BFF"]
            API["bot-config-api<br/>(FastAPI)"]
            subgraph APIServices["API Surfaces"]
                AUTH["Auth/Session<br/>(/api/v1/auth/*)"]
                CHAT["Communication API<br/>(/api/v1/chat/*)"]
                WFAPI["Workflow API<br/>(/api/v1/workflows/*<br/>/api/v1/delivery/start)"]
                ADMIN["Admin API<br/>(/api/v1/bots/*<br/>/api/v1/deploy/*<br/>/api/v1/governance/*)"]
            end
            subgraph Services["Domain Services"]
                WFS["Workflow Service"]
                GOVS["Governance Service"]
                A2AS["A2A Dispatch"]
            end
        end

        subgraph BotPods["Bot Pods (1 per bot)"]
            subgraph Bot1["Orchestrator Bot"]
                GW1["Agent Gateway"]
                SC1["A2A Sidecar"]
                GOV1["governance-bridge"]
                WF1["wf-bridge<br/>(wf_status + wf_command)"]
                MEM1["mem0-api"]
            end
            subgraph Bot2["Worker Bot"]
                GW2["Agent Gateway<br/>+ dispatch port 18790"]
                SC2["A2A Sidecar"]
                GOV2["governance-bridge"]
                WF2["wf-bridge<br/>(wf_status only)"]
                MEM2["mem0-api"]
            end
            subgraph Bot3["Reviewer Bot"]
                GW3["Agent Gateway<br/>+ dispatch port 18790"]
                SC3["A2A Sidecar"]
                GOV3["governance-bridge"]
                WF3["wf-bridge<br/>(wf_status only)"]
                MEM3["mem0-api"]
            end
        end

        subgraph Data["Persistent Storage"]
            PG[("PostgreSQL<br/>w0_runs, w0_actor_bindings,<br/>w0_work_packets, a2a_tasks,<br/>w0_gate_verdicts,<br/>conversations, messages")]
        end

        subgraph GitOps["GitOps"]
            ARGO["ArgoCD"]
        end
    end

    APP -->|"Communication<br/>(chat proxy)"| CHAT
    APP -->|"Workflow start<br/>(typed form)"| WFAPI
    APP -->|"Admin UI"| ADMIN

    CHAT -->|"Proxy to gateway<br/>(general_chat)"| GW1 & GW2 & GW3
    GW1 & GW2 & GW3 -->|"LLM requests"| LLM

    GOV1 & GOV2 & GOV3 -->|"Governance API<br/>(execution_class matrix)"| API

    SC2 -->|"localhost:18790<br/>(workflow_dispatch)"| GW2
    SC3 -->|"localhost:18790<br/>(workflow_dispatch)"| GW3

    API -->|"GitHub operations<br/>(per-bot PAT)"| GH
    API -->|"A2A/sidecar dispatch"| SC1 & SC2 & SC3
    API -->|"CRUD"| PG

    ARGO -->|"Sync"| BotPods & ControlPlane

    style External fill:#f5f5f5,stroke:#999
    style K8s fill:#e8f4fd,stroke:#2196F3
    style ControlPlane fill:#fff3e0,stroke:#FF9800
    style BotPods fill:#e8f5e9,stroke:#4CAF50
    style Data fill:#fce4ec,stroke:#E91E63
    style User fill:#e8eaf6,stroke:#3F51B5
```

### Dual-lane product model

The system supports two interaction modes, real at every layer:

| Mode | UX | API surface | Backend path | Governance |
|---|---|---|---|---|
| **Communication** | Chat with selected bot | `/api/v1/chat/{bot_id}/*` | bot-config-api → gateway (general_chat) | consultation_read only |
| **Workflow** | Start delivery, track progress, workflow-scoped conversation | `/api/v1/workflows/*`, `/api/v1/delivery/start` | bot-config-api → Temporal → sidecar → gateway (workflow_dispatch) | execution_write, review_write via sidecar |

Communication and Workflow are not cosmetic tabs — they hit different API surfaces, follow different backend paths, and enforce different governance rules.

### 3 execution boundaries

| Boundary | Control point | Runtime dependency |
|---|---|---|
| **1. Orchestrator delivery start** | Workflow UI form → typed API. `start_delivery` tool removed from model surface. | NO |
| **2. Worker/reviewer normal chat** | Governance defaults to general_chat → denies writes. | NO |
| **3. Worker/reviewer sidecar execution** | Dispatch port 18790 + executionClass propagation → governance allows writes. | YES |

### What is implemented

- **Typed delivery start**: `POST /api/v1/delivery/start` with `DeliveryTargetRef`. Called by Workflow UI form (web app). `start_delivery` model tool removed (Boundary 1).
- **Temporal workflow**: DeliveryWorkflow with 9 signals, 6 updates, 3 queries. State machine: PLANNED → IN_DEV → DEV_DONE → QA_APPROVED → MERGE_READY → DONE
- **Sidecar dispatch**: Sidecar → dispatch port 18790 (localhost-only) → gateway with `executionClass: workflow_dispatch`. Task lifecycle API with correlation tracking
- **Governance**: 28 API tools classified into 5 action classes. `execution_class × action_class` authorization matrix. Fail-closed for unknown.
- **LangGraph**: Planner, reviewer, routing graphs. Run inside Temporal activities (ephemeral, no I/O)
- **A2A consultation**: Bot-to-bot relay for questions, brainstorming. Separate from delivery dispatch
- **Role contracts**: Orchestrator (`wf_status` + `wf_command`), worker (`wf_status` only), reviewer (`wf_status` only). `start_delivery` removed from model surface.
- **Persistence**: Runs, bindings, packets, A2A tasks, gate verdicts, objectives, activation records, conversations, workflow messages in Postgres. Auto-migration on startup.
- **Bot activation**: 4 gates (SOUL, binding, MCP, tools). Deployment validation. Only ACTIVE bots eligible for delivery routing
- **Auth/session layer**: `admin_user` principal, BFF session, httpOnly cookie, route guards. `require_auth`, `require_dual_auth` dependencies.
- **Communication API** (`/api/v1/chat/*`, WS): Chat proxy, conversation persistence. WebSocket for real-time streaming.
- **Workflow API** (`/api/v1/workflows/*`): List, detail, artifacts, workflow-scoped conversation endpoints.
- **Unified web app**: agentopia-ui with Communication lane + Workflow lane + Admin sections. React 19 + Vite.
- **Review system**: SCM-bound and workflow-scoped reviewer coexistence. Metadata-based dispatch filtering. GitHub webhook integration.
- **Telegram integration**: Optional secondary interaction surface (web app is primary).

### Known system gaps

- **Execution authorization (Boundary 3)**: `executionClass` propagation implemented in code across `agentopia-core` and governance-bridge. Runtime verification pending (P1 #259).
- **Delivery completion semantics**: Sidecar marks "completed" on any LLM text response, not artifact verification.
- **No auto-escalation**: `wait_dev` blocks indefinitely if worker fails. Manual cancel required.
- **Delivery start API trust**: Product entrypoint, not hardened boundary. Dual-auth guard accepts session cookie or service token.

---

## 2. Target Architecture (Hybrid Stack)

```mermaid
graph TB
    subgraph External["External Services"]
        TG["Telegram API"]
        GH["GitHub API<br/>(Webhooks + REST)"]
        LLM["Anthropic Claude API"]
    end

    subgraph Operator["Operator"]
        OP["CTO / Operator<br/>(Web App — primary)"]
    end

    subgraph K8s["Kubernetes Cluster"]

        subgraph Agentopia["Agentopia — Governed Execution Substrate"]
            API["bot-config-api<br/>(FastAPI)"]
            subgraph Services["Domain Services"]
                WFS["Workflow Service<br/>(domain state owner)"]
                OES["Orchestration Event Service<br/>(webhook + signal routing)"]
                GOVS["Governance Service<br/>(GitHub operations)"]
            end
            subgraph Infra["Infrastructure"]
                TC["temporal_client.py<br/>(outbound signals)"]
            end
        end

        subgraph Temporal["Temporal — Durable Orchestration Runtime"]
            TS["Temporal Server<br/>(event sourcing, signals,<br/>retry, timeout)"]
            subgraph Workers["Temporal Workers"]
                DW["DeliveryWorkflow<br/>(state machine:<br/>PLANNED→IN_DEV→DEV_DONE<br/>→QA_APPROVED→MERGE_READY→DONE)"]
                subgraph Activities["Activities (anti-corruption boundary)"]
                    A_PLAN["plan_delivery"]
                    A_DISPATCH["dispatch_work"]
                    A_VERIFY["verify_dev_completion<br/>(read-only)"]
                    A_PKT["transition_packet_state"]
                    A_ADV["advance_workflow_state"]
                    A_REVIEW["request_review"]
                    A_MERGE["merge_and_close"]
                    A_GATE["evaluate_gate"]
                end
            end
        end

        subgraph LangGraph["LangGraph — Cognitive Layer (Ephemeral)"]
            PG_PLAN["planner_graph<br/>(decompose→validate→structure)"]
            PG_REVIEW["reviewer_graph<br/>(analyze→decide)"]
            PG_ROUTE["routing_graph<br/>(actor pool selection)"]
        end

        subgraph BotPods["Bot Pods"]
            Bot1["Orchestrator Bot<br/>wf_status + wf_command<br/>(start_delivery removed from model surface)"]
            Bot2["Worker Bot<br/>wf_status only<br/>+ relay + governance (role-scoped)"]
            Bot3["Reviewer Bot<br/>wf_status only<br/>+ relay + governance (role-scoped)"]
        end

        subgraph Data["PostgreSQL (Managed)"]
            PG_AGT[("Agentopia DB<br/>w0_runs, w0_work_packets,<br/>w0_actor_bindings, a2a_tasks,<br/>w0_objectives, w0_evidence,<br/>w0_gate_verdicts,<br/>webhook_delivery_log,<br/>workflow_artifact_mapping")]
            PG_TMP[("Temporal DB<br/>(internal persistence)")]
        end

        subgraph Observability["Observability"]
            PROM["Prometheus"]
            GRAF["Grafana"]
        end

    end

    OP -->|"Workflow UI form /<br/>Communication chat"| API
    TG <-.->|"Messages<br/>(optional secondary)"| Bot1 & Bot2 & Bot3

    Bot1 & Bot2 & Bot3 -->|"Governance API<br/>(execution_class matrix)"| API
    Bot1 -->|"wf_command<br/>(status/cancel)"| API
    Bot1 & Bot2 & Bot3 -->|"LLM"| LLM

    GH -->|"Webhooks"| API
    API -->|"validate + dedup"| OES
    OES -->|"signal/update"| TC
    TC -->|"signal_workflow /<br/>execute_update"| TS

    TS -->|"schedule"| DW
    DW -->|"execute"| Activities
    Activities -->|"call services"| WFS & GOVS
    A_PLAN -->|"invoke<br/>(ephemeral, no I/O)"| PG_PLAN
    A_DISPATCH -->|"invoke"| PG_ROUTE
    GOVS -->|"GitHub API<br/>(per-bot PAT)"| GH
    WFS -->|"A2A dispatch"| Bot1 & Bot2 & Bot3

    WFS & GOVS & OES -->|"read/write"| PG_AGT
    TS -->|"event history"| PG_TMP

    API & TS -->|"metrics"| PROM
    PROM --> GRAF

    style External fill:#f5f5f5,stroke:#999
    style K8s fill:#e8f4fd,stroke:#2196F3
    style Agentopia fill:#fff3e0,stroke:#FF9800
    style Temporal fill:#e1f5fe,stroke:#03A9F4
    style LangGraph fill:#f3e5f5,stroke:#9C27B0
    style BotPods fill:#e8f5e9,stroke:#4CAF50
    style Data fill:#fce4ec,stroke:#E91E63
    style Observability fill:#f5f5f5,stroke:#607D8B
```

### Hybrid stack decision

| Layer | Owns | Does NOT own |
|---|---|---|
| **Agentopia** | Domain state (Postgres), role contracts, governance auth, GitHub execution | Orchestration durability, planning, long-running coordination |
| **Temporal** | Durable orchestration lifecycle, signals, retry/timeout, activity scheduling | Business rules, domain state, LLM reasoning |
| **LangGraph** | Planning, review analysis, actor routing (ephemeral cognitive) | Durable execution, persistence, orchestration lifecycle |

---

## 3. Autonomous Delivery Lifecycle

```mermaid
sequenceDiagram
    participant OP as Operator
    participant API as Agentopia API
    participant TW as Temporal Workflow
    participant ACT as Activities
    participant LG as LangGraph
    participant GH as GitHub
    participant BOT as Worker/Reviewer Bot

    OP->>API: Submit objective
    API->>API: Persist Objective (Postgres)
    API->>TW: Start DeliveryWorkflow

    Note over TW: PLANNED

    TW->>ACT: plan_delivery
    ACT->>LG: planner_graph (cognitive, no I/O)
    LG-->>ACT: DeliveryPlan
    ACT->>API: Create milestone + issue + packet
    ACT-->>TW: PlanDeliveryResult

    TW->>ACT: dispatch_work
    ACT->>API: Resolve actor from pool
    ACT->>BOT: A2A task dispatch
    ACT-->>TW: DispatchResult
    TW->>ACT: advance_workflow_state(IN_DEV)

    Note over TW: IN_DEV

    BOT->>GH: Create branch + PR
    GH-->>API: Webhook: PR opened
    API->>TW: Signal: on_pr_opened

    BOT->>API: A2A task completed
    API->>TW: Signal: on_a2a_task_completed

    TW->>ACT: verify_dev_completion (read-only)
    ACT-->>TW: {verified: true}
    TW->>ACT: transition_packet_state(review)
    TW->>ACT: advance_workflow_state(DEV_DONE)

    Note over TW: DEV_DONE

    TW->>ACT: request_review
    ACT->>BOT: Dispatch review to reviewer

    BOT->>GH: Submit APPROVE review
    GH-->>API: Webhook: review submitted
    API->>TW: Signal: on_review_submitted(APPROVE)
    TW->>ACT: advance_workflow_state(QA_APPROVED)

    Note over TW: QA_APPROVED

    TW->>ACT: merge_and_close
    ACT->>GH: Merge PR + close issue
    GH-->>API: Webhook: PR merged
    API->>TW: Signal: on_pr_merged
    TW->>ACT: advance_workflow_state(MERGE_READY)

    Note over TW: MERGE_READY

    TW->>ACT: evaluate_gate
    ACT-->>TW: {passed: true}
    TW->>ACT: advance_workflow_state(DONE)

    Note over TW: DONE
```

---

## 4. Layer Ownership

```
┌─────────────────────────────────────────────────────────────┐
│                    OWNERSHIP MODEL                           │
├──────────────┬──────────────────────────────────────────────┤
│              │                                              │
│  AGENTOPIA   │  Domain state (canonical):                   │
│  (Postgres)  │  - Objectives, Workflows, Packets            │
│              │  - Bindings, Evidence, Gate Verdicts          │
│              │  - A2A Tasks, Artifact Mapping                │
│              │  Role contracts + governance enforcement      │
│              │  GitHub API execution (per-bot PAT)           │
│              │                                              │
├──────────────┼──────────────────────────────────────────────┤
│              │                                              │
│  TEMPORAL    │  Orchestration lifecycle (durable):           │
│  (Event      │  - Workflow state machine execution           │
│   History)   │  - Signal/update handling                     │
│              │  - Timer/timeout management                   │
│              │  - Activity scheduling + retry                │
│              │  - Event sourcing (replay-safe)               │
│              │                                              │
├──────────────┼──────────────────────────────────────────────┤
│              │                                              │
│  LANGGRAPH   │  Cognitive decisions (ephemeral):             │
│  (In-memory, │  - Planning decomposition                    │
│   no persist)│  - Review analysis                           │
│              │  - Actor routing selection                    │
│              │  - Runs ONLY inside Temporal activities       │
│              │  - NO I/O, NO service imports                 │
│              │                                              │
├──────────────┼──────────────────────────────────────────────┤
│              │                                              │
│  GITHUB      │  External artifacts (source of truth):       │
│  (External)  │  - Issues, PRs, Reviews, Milestones          │
│              │  - CI/CD check results                        │
│              │  - Webhook events → Agentopia → Temporal      │
│              │                                              │
└──────────────┴──────────────────────────────────────────────┘
```

---

## 5. Implementation Status

| Milestone | Scope | Status |
|---|---|---|
| **Rounds 1-5** | Persistence, refactor, Temporal, activities, workflow, LangGraph, pool, monitoring | **DONE** |
| **Wave E L2** | Gateway-native A2A: sidecar, task lifecycle API, dispatch transport | **DONE** |
| **Wave E LangGraph** | LangGraph standalone service planning | **DONE** |
| **P0 #26** | Production contract foundation: typed start, review fallback, activation | **IN PROGRESS** — Phase 1+2 code merged. Permanent dual-lane model adopted. |
| **P0.5 #28** | Deterministic delivery start front door | **CLOSED/DEFERRED** — C2 rejected. Permanent dual-lane model adopted. |
| **P1 #27 (old)** | Execution authorization enforcement (runtime-centric) | **SUPERSEDED** — rebased to P1 #30 |
| **P1 #30 (new)** | Web-App Primary Dual-Lane MVP | **ACTIVE** — 11 issues. See [p1-rebase-web-primary-dual-lane.md](./p1-rebase-web-primary-dual-lane.md) |
| **#25** | Agent Runtime — Fork & Customize OpenClaw | Epic — umbrella for runtime fork work |
| **Wave F #22** | Multi-step planning, human-in-loop, streaming | 1/6 closed |
| **Wave G #23** | Multi-packet, collaborative conversation, graph branching | 0/4 |

### Current Active Work — P1 #30 Phases
1. **Phase 1**: Auth session layer + Communication API + Workflow API extensions + Workflow conversation API
2. **Phase 2**: Workflow start surface (Boundary 1) — remove `start_delivery` from model, SOUL update
3. **Phase 3**: Sidecar propagation fix (Boundary 3) — re-verify + fix runtime executionClass chain
4. **Phase 4**: Unified frontend (app shell, Communication UI, Workflow UI, admin link)
5. **Phase 5**: E2E proof (5 scenarios through web UI)

### Delivery start path resolution
- **Old gap**: LLM may not call `start_delivery` tool — depended on model compliance
- **Resolved by rebase**: `start_delivery` removed from model surface. Delivery start is now Workflow UI form → typed API. Model-mediated start eliminated. Start-path integrity is product-surface-controlled.

---

## 6. Related Documents

| Document | Purpose |
|---|---|
| [p1-rebase-web-primary-dual-lane.md](./p1-rebase-web-primary-dual-lane.md) | **CANONICAL** — P1/P2 milestone rebase, dual-lane product model, boundaries, release gate |
| [p1-execution-authorization.md](./p1-execution-authorization.md) | Execution authorization architecture — sections 2.1–2.5 remain canonical (matrix, action/execution classes) |
| [canonical-object-model.md](./canonical-object-model.md) | 13-object delivery model with ownership |
| [state-ownership-matrix.md](./state-ownership-matrix.md) | 4-layer state ownership (Agentopia/Temporal/LangGraph/GitHub) |
| [hybrid-integration.md](./hybrid-integration.md) | Agentopia + Temporal + LangGraph integration contract |
| [../contracts/delivery-start-contract.md](../contracts/delivery-start-contract.md) | Typed delivery start API — updated for web-app-primary model |
| [../contracts/review-verification.md](../contracts/review-verification.md) | Review fallback verification semantics |
| [../contracts/bot-activation.md](../contracts/bot-activation.md) | Activation states, gates, routing enforcement |
| [../a2a-protocol/a2a-solution-protocol.md](../a2a-protocol/a2a-solution-protocol.md) | A2A protocol design |
| [P0.5-deterministic-delivery-start-front-door.md](./P0.5-deterministic-delivery-start-front-door.md) | CLOSED/DEFERRED — superseded by web-app-primary Workflow start surface |
| [../design-waves/](../design-waves/) | Historical wave deliverables (A through E) |
| [../operations/](../operations/) | Runbooks, UAT guide, memory hygiene |
