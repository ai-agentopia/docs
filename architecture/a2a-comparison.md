---
title: "A2A Implementation Comparison: BeeAI Framework vs Agentopia"
---

# A2A Implementation Comparison: BeeAI Framework vs Agentopia

> Date: 2026-03-19
> Purpose: Honest technical comparison of A2A (Agent-to-Agent) implementations
> Scope: A2A features only — not full platform comparison

---

## 1. Overview

Both projects implement the A2A protocol using the official Linux Foundation SDK (`a2a` Python package). They serve different purposes:

- **BeeAI Framework** — A general-purpose multi-agent framework optimized for developer experience and rapid agent prototyping
- **Agentopia** — A governed delivery orchestration platform optimized for durable, role-enforced autonomous software delivery

This document compares their A2A implementations specifically, with code evidence from both repositories.

---

## 2. Protocol Compliance

### 2.1 SDK Usage

Both projects use the official A2A SDK from the Linux Foundation.

**BeeAI:**
```python
# beeai-framework/python/beeai_framework/adapters/a2a/
from a2a.types import AgentCard, AgentCapabilities, AgentSkill
```

**Agentopia:**
```python
# bot-config-api/src/a2a_protocol/card_generator.py:14
from a2a.types import AgentCard, AgentCapabilities, AgentSkill

# bot-config-api/src/a2a_protocol/server.py:19
from a2a.server.agent_execution import AgentExecutor, RequestContext
from a2a.server.apps import A2AStarletteApplication
```

Both use the same official types and server components.

### 2.2 Agent Card

| Aspect | BeeAI | Agentopia |
|---|---|---|
| Agent Card generation | Manual in code | Auto-generated from bot.yaml + SOUL.md metadata |
| `.well-known/agent-card.json` | Served per A2A spec | Served at `/a2a/{bot_name}/.well-known/agent-card.json` |
| Multi-agent per server | One agent per server | Multiple bots on single server via mount registry |

**Agentopia evidence** — auto-generation from bot metadata:
```python
# bot-config-api/src/a2a_protocol/card_generator.py:23-42
def generate_agent_card(bot_name, bot_yaml=None, soul_md=None):
    return AgentCard(
        name=bot_yaml.get("name", bot_name.replace("-", " ").title()),
        url=f"{SERVICE_BASE_URL}/a2a/{bot_name}",
        description=_extract_description(soul_md),  # Extracts from SOUL.md Identity section
        capabilities=AgentCapabilities(streaming=True),
        skills=_generate_skills(bot_yaml),  # Standard + custom + role-specialist skills
    )
```

**Agentopia evidence** — multi-agent mount registry:
```python
# bot-config-api/src/a2a_protocol/mount_registry.py:37-51
def mount_bot_a2a(app, bot_name, bot_yaml=None, soul_md=None):
    card = generate_agent_card(bot_name, bot_yaml, soul_md)
    a2a_sub_app = create_a2a_app(bot_name, card)
    app.mount(f"/a2a/{bot_name}", a2a_sub_app)  # Each bot gets its own A2A endpoint
```

### 2.3 Transport Protocols

| Transport | BeeAI | Agentopia |
|---|---|---|
| JSON-RPC (Starlette) | ✅ | ✅ (via A2A SDK `A2AStarletteApplication`) |
| gRPC | ✅ (with health check + reflection) | ❌ |
| HTTP/JSON | ✅ (via FastAPI) | ✅ (via sidecar HTTP adapter) |
| Streaming | ✅ Event streaming | ✅ `EventQueue` + `InMemoryQueueManager` |

**BeeAI advantage:** gRPC transport is available, which Agentopia does not implement.

### 2.4 Multi-Turn Conversations

| Aspect | BeeAI | Agentopia |
|---|---|---|
| Context chaining | Task ID / context ID across turns | Thread system with `a2a_create_thread`, `a2a_send_turn` |
| Turn dedup | Not implemented | turn_id dedup at executor level |
| Conversation memory | LRU cache on server (`LRUMemoryManager`) | Postgres + mem0 + session files |

---

## 3. Agent Discovery

### 3.1 Discovery Service

**BeeAI:** No dedicated discovery service. Clients must know the agent URL upfront and fetch the agent card directly.

**Agentopia:** Full discovery service with search, ranking, caching, and security controls.

```python
# bot-config-api/src/a2a_protocol/discovery.py:71-122
class AgentDiscovery:
    async def resolve(self, agent_url: str) -> AgentCard:
        # HTTPS-only enforcement
        # Allowlist check
        # Cache check (TTL-based)
        # SSRF protection (blocks private IPs)
        # Fetch + validate + cache

    def discover(self, skill=None, tag=None, name=None) -> list[dict]:
        # Search internal + external agents
        # Score by relevance (skill match, tag match, name match)
        # Return ranked results
```

### 3.2 Security Controls

| Control | BeeAI | Agentopia |
|---|---|---|
| HTTPS enforcement | Not enforced | External agents must use HTTPS |
| SSRF protection | None | Blocks private IP ranges (10.x, 172.16.x, 192.168.x, 127.x, link-local, IPv6 private) |
| Allowlist | None | `A2A_EXTERNAL_ALLOWLIST` — only allowed URLs can be discovered |
| Size limit | None | 64KB max agent card size |
| Cache TTL | None | Configurable (`A2A_CARD_CACHE_TTL`, default 5 min) |

**Agentopia evidence** — SSRF protection:
```python
# bot-config-api/src/a2a_protocol/discovery.py:37-45
_BLOCKED_NETWORKS = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("127.0.0.0/8"),
    ipaddress.ip_network("169.254.0.0/16"),  # link-local + AWS IMDS
    ipaddress.ip_network("::1/128"),
    ipaddress.ip_network("fc00::/7"),
]
```

### 3.3 Skill-Based Search

**BeeAI:** Not available. Clients connect to known agents by URL.

**Agentopia:** Relevance-scored search across internal and external agents.

```python
# bot-config-api/src/a2a_protocol/discovery.py:221-256
@staticmethod
def _match_score(card, skill, tag, name) -> float:
    # skill exact match: +1.0
    # skill in description: +0.5
    # tag match in skill tags: +0.8
    # tag in description: +0.3
    # name match: +1.0
```

### 3.4 Auto-Generated Skills

**BeeAI:** Skills are declared manually in code.

**Agentopia:** Skills are auto-generated from three sources:

```python
# bot-config-api/src/a2a_protocol/card_generator.py:61-123
def _generate_skills(bot_yaml):
    skills = [
        # 1. Standard skills (every bot gets these)
        AgentSkill(id="debate", ...),
        AgentSkill(id="bridge-respond", ...),
        AgentSkill(id="summarize", ...),
    ]
    # 2. Custom skills from bot.yaml
    for s in bot_yaml.get("skills", []):
        skills.append(AgentSkill(id=s["id"], ...))

    # 3. Auto-specialist from role_hint
    role = bot_yaml.get("role_hint", "")
    if role:
        skills.append(AgentSkill(id="specialist", name=f"{role} Specialist", ...))
```

---

## 4. Dispatch and Transport

### 4.1 Dispatch Model

**BeeAI:** Direct client-to-server dispatch. Client sends A2A message, server executor wraps agent, returns result.

**Agentopia:** Multi-layer dispatch with tracking and coordination.

```
Temporal activity
  → SidecarDispatchTransport
  → sidecar (port 18791)
  → OpenClawRuntimeAdapter (anti-corruption boundary)
  → gateway /v1/chat/completions (LLM + tools)
  → response
  → sidecar reports to bot-config-api task lifecycle API
  → bot-config-api emits Temporal signal
```

### 4.2 Coordination Guards

**BeeAI:** No coordination guards. Any agent can call any other agent freely, including recursively.

**Agentopia:** Explicit 10-tool deny list prevents A2A recursion. Guards are task_kind-specific.

```python
# bot-config-api/src/infrastructure/runtime/openclaw_adapter.py
# 10 tools denied in ALL supported task kinds:
#   a2a_send_task, a2a_create_thread, a2a_send_turn,
#   a2a_discover_agents, relay_send, relay_create_thread,
#   relay_send_turn, relay_read_thread, relay_checkpoint,
#   relay_conclude
```

### 4.3 Typed Dispatch Request

**BeeAI:** Standard A2A task message.

**Agentopia:** Typed request with full tracking context.

```python
# bot-config-api/src/w0_protocol/a2a_dispatch.py
@dataclass
class DispatchTransportRequest:
    target_agent: str
    message: str
    packet_id: str
    workflow_id: str
    correlation_id: str
    task_kind: str    # delivery_task, review, operator_command, a2a_one_shot
    metadata: dict
```

### 4.4 Transport Call Site Classification

**BeeAI:** All dispatch calls are equivalent — no distinction between tracked and untracked.

**Agentopia:** 7 call sites classified as tracked (full attempt lifecycle) or untracked (fire-and-forget):

| Site | Type | Tracked? |
|---|---|---|
| `activities.py::dispatch_work` | Delivery dispatch | Yes — full attempt lifecycle |
| `workflow_service.py /wf assign` | Manual assign | Yes |
| `orchestration_loop.py downstream` | Downstream dispatch | Yes |
| `activities.py::request_review` | Review request | No — correlation_id="" |
| `activities.py::trigger_rework` | Rework instruction | No |
| `activities.py::escalate_rework_limit` | Escalation notification | No |
| `orchestration_loop.py notification` | Status notification | No |

---

## 5. Task Lifecycle Management

### 5.1 Task States

| State | BeeAI | Agentopia |
|---|---|---|
| submitted | ✅ | ✅ |
| working | ✅ | ✅ |
| completed | ✅ | ✅ |
| failed | ✅ | ✅ |
| canceled | ❌ | ✅ |

### 5.2 Task Lifecycle API

**BeeAI:** No dedicated task lifecycle API. The executor handles task state internally.

**Agentopia:** Four dedicated endpoints with auth.

```python
# bot-config-api/src/routers/a2a_tasks.py
# Auth: Bearer A2A_INTERNAL_TOKEN on all endpoints

POST /api/v1/a2a/tasks              # Create task (idempotent on correlation_id)
POST /api/v1/a2a/tasks/{id}/status  # Update terminal status (with decision matrix)
GET  /api/v1/a2a/tasks/{id}         # Read task state
POST /api/v1/a2a/tasks/{id}/cancel  # Cancel task (forward to sidecar)
```

### 5.3 Decision Matrix on Status Update

**BeeAI:** Not applicable — no external status update mechanism.

**Agentopia:** Explicit decision matrix for every status update:

| Condition | Result |
|---|---|
| Already terminal | 409 — reject |
| Dispatch found + active attempt | Write status + cascade attempt_status + emit signal |
| Dispatch found + superseded attempt | Audit write only, no cascade, no signal |
| Dispatch found + canceled attempt | Audit write only, no cascade, no signal |
| No dispatch linkage | 409 — invariant violation |

### 5.4 Attempt Tracking

**BeeAI:** No concept of attempts. Each dispatch is independent.

**Agentopia:** Full attempt lifecycle with database enforcement.

```sql
-- bot-config-api/db/009_dispatch_attempt_fields.sql
ALTER w0_dispatch_records ADD attempt_status TEXT NOT NULL DEFAULT 'active';
ALTER w0_dispatch_records ADD correlation_id TEXT NOT NULL DEFAULT '';
ALTER w0_dispatch_records ADD source_event_id TEXT NOT NULL DEFAULT '';

-- Only one active attempt per packet (database-enforced invariant)
CREATE UNIQUE INDEX idx_one_active_attempt_per_packet
    ON w0_dispatch_records(packet_id) WHERE attempt_status = 'active';

-- Prevent duplicate reassign from same event
CREATE UNIQUE INDEX idx_dispatch_source_event
    ON w0_dispatch_records(packet_id, source_event_id)
    WHERE source_event_id != '';
```

### 5.5 Dispatch Result Tracking

**BeeAI:** Fire and forget — no result tracking.

**Agentopia:** Every tracked dispatch records its outcome.

| Result | Packet Impact | Rationale |
|---|---|---|
| DISPATCH_OK | Stays in_progress | Delivered successfully |
| DISPATCH_TIMEOUT | Stays in_progress | Transport timeout — Temporal can retry |
| DISPATCH_RETRYABLE | Stays in_progress | Transient error — Temporal can retry |
| DISPATCH_CIRCUIT_OPEN | Stays in_progress | Target unavailable — Temporal can retry |
| DISPATCH_TERMINAL | Packet → failed | Unrecoverable transport failure |

---

## 6. Orchestration Integration

### 6.1 Workflow Integration

**BeeAI:** A2A operates independently. The in-memory `Workflow` class has no integration with A2A task lifecycle. Workflow steps are in-process function calls, not durable activities.

```python
# beeai-framework/python/beeai_framework/workflows/workflow.py
# In-memory state machine — no Temporal, no durable execution
workflow = Workflow(State)
workflow.add_step("research", research_step)
workflow.add_step("podcast", podcast_step)
# Process crash → all state lost
```

**Agentopia:** A2A completion triggers Temporal signals that advance the delivery workflow automatically.

```python
# bot-config-api/src/temporal_workflows/delivery_workflow.py:466-495
@workflow.signal
async def on_a2a_task_completed(self, event: A2ATaskEvent):
    # Verify dev completion (READ-ONLY activity)
    verify_result = await workflow.execute_activity(verify_dev_completion, ...)
    if verify_result.verified:
        self._state.dev_verified = True
        # Transition packet state (dedicated mutation activity)
        await workflow.execute_activity(transition_packet_state, ...)
    # Workflow's wait_condition(dev_verified) unblocks → advances to DEV_DONE
```

### 6.2 Delivery Lifecycle

**BeeAI:** No delivery lifecycle concept.

**Agentopia:** Full lifecycle with 9 signals, 6 updates, 3 queries:

```
PLANNED → IN_DEV → DEV_DONE → QA_APPROVED → MERGE_READY → DONE
                ↑                    |
                └── rework ──────────┘
```

Each transition is driven by A2A signals or governance API signals — no manual operator intervention required for the happy path.

### 6.3 Key Invariant

**Agentopia explicitly separates transport truth from delivery truth:**

> A2A task completed ≠ workflow advancement. The workflow must verify (PR exists, CI passes) before advancing state.

This invariant is enforced by the `verify_dev_completion` activity (read-only) and the `advance_workflow_state` activity (dedicated lifecycle mutation).

BeeAI has no equivalent concept — A2A task completion is the final state.

---

## 7. Persistence and Crash Recovery

| Aspect | BeeAI | Agentopia |
|---|---|---|
| Task state storage | In-memory only | Postgres (`a2a_tasks` table) |
| Dispatch records | None | Postgres (`w0_dispatch_records` with attempt tracking) |
| Workflow state | In-memory `Workflow` class | Temporal event-sourced (survives restart) |
| Conversation memory | LRU cache (`LRUMemoryManager(maxsize=100)`) | Postgres + mem0 + session files |
| Crash recovery | All state lost — must restart from beginning | Temporal replays workflow. Idempotent task creation recovers sidecar state via correlation_id |

**Agentopia crash recovery evidence:**

When sidecar crashes and restarts, it calls `POST /api/v1/a2a/tasks` with the same `correlation_id`. The endpoint returns the existing task (200) instead of creating a duplicate (201). The sidecar resumes from the known state.

---

## 8. Security

| Control | BeeAI | Agentopia |
|---|---|---|
| Auth on A2A endpoints | None by default | Bearer `A2A_INTERNAL_TOKEN` on all endpoints |
| Role-based access control | None — any agent can use any tool | 28 governance tools enforced by role (orchestrator/worker/reviewer) |
| Anti-spoof | None | `_verify_actor()` — cannot claim a role different from binding |
| Input validation | Pydantic tool input schemas | Input size limits + rate limiting on A2A server |
| SSRF protection for discovery | None | Blocks all private IP ranges + link-local + AWS IMDS |
| HTTPS for external agents | Not enforced | Required for all external agent URLs |
| Audit trail | In-memory event emitter | Postgres: dispatch records, command audit, evidence store, gate verdicts |
| Replay prevention | None | turn_id dedup at executor + correlation_id dedup at task level |

---

## 9. Deployment and Operations

| Aspect | BeeAI | Agentopia |
|---|---|---|
| Deployment model | `python main.py` — standard process | Helm charts + ArgoCD GitOps + per-bot K8s Deployment |
| Multi-agent deployment | One process per agent | Multiple bots in one cluster, each with gateway + sidecar |
| Configuration | Code-level — redeploy to change | UI + ConfigMap + values.yaml — change via UI or GitOps |
| Monitoring | In-process event emitter, no Prometheus | 8 Prometheus metrics + 2 Grafana dashboards + 4 alert rules |
| Health checks | gRPC health (gRPC transport only) | Kubernetes liveness/readiness probes for all pods |
| Scaling | Stateless server behind load balancer | Per-bot K8s replicas + separate Temporal worker scaling |

---

## 10. Summary Scorecard

| Area | BeeAI | Agentopia | Winner |
|---|---|---|---|
| A2A SDK usage | ✅ Official | ✅ Official | Tie |
| Agent Card auto-generation | Manual | Auto from metadata | Agentopia |
| Discovery service | None | Full (search, rank, cache, security) | Agentopia |
| Transport protocols | 3 (JSON-RPC, gRPC, HTTP) | 2 (JSON-RPC, HTTP) | BeeAI |
| Streaming | Event streaming | EventQueue + InMemoryQueueManager | Tie |
| Dispatch sophistication | Basic client-server | Typed request + coordination guards + attempt tracking | Agentopia |
| Task lifecycle API | None (internal executor) | 4 endpoints with decision matrix | Agentopia |
| Attempt tracking | None | Full (active/superseded, DB-enforced invariants) | Agentopia |
| Orchestration integration | None (standalone) | Temporal signals + delivery workflow | Agentopia |
| Persistence | In-memory only | Postgres (tasks, dispatches, workflows) | Agentopia |
| Crash recovery | All state lost | Temporal replay + idempotent task creation | Agentopia |
| Security | Minimal | Auth + RBAC + anti-spoof + SSRF protection + audit | Agentopia |
| Deployment | Manual process | Helm + ArgoCD + K8s | Agentopia |
| Monitoring | Event emitter | Prometheus + Grafana + alerts | Agentopia |
| Conversation memory types | 5 types (unconstrained, token, sliding, summarize, readonly) | 3 types (mem0, session, relay) | BeeAI |
| Developer experience | pip install + 5 lines | K8s cluster + Temporal + Postgres + ArgoCD | BeeAI |

---

## 11. Conclusion

**BeeAI strengths in A2A:**
- gRPC transport for high-performance inter-agent communication
- Simpler developer experience — rapid prototyping
- More conversation memory types for flexible agent design

**Agentopia strengths in A2A:**
- Auto-generated Agent Cards from bot deployment metadata
- Discovery service with security controls (SSRF, HTTPS, allowlist)
- Typed dispatch with coordination guards preventing recursive A2A calls
- Full task lifecycle API with attempt tracking and database-enforced invariants
- Temporal integration — A2A completion drives durable workflow advancement
- Crash recovery via Temporal replay and idempotent task creation
- Role-based governance enforcement on every dispatch
- Production deployment model (Helm + ArgoCD + monitoring)

**Key architectural difference:**

BeeAI treats A2A as a **communication protocol** — agents talk to agents. The framework provides excellent tooling for building and connecting agents.

Agentopia treats A2A as a **governed execution layer** — agents talk to agents within a durable, role-enforced delivery pipeline. A2A task completion is not the end state; it is an input to a verification and advancement process that ensures delivery quality.

Neither approach is universally better. They serve different use cases:
- **BeeAI A2A** is ideal for general-purpose multi-agent applications where agents need flexible, spec-compliant communication
- **Agentopia A2A** is ideal for production software delivery automation where durability, governance, and orchestration are requirements
