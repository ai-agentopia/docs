---
title: "P1: Web-App Primary Dual-Lane MVP"
---

# P1: Web-App Primary Dual-Lane MVP

**Status**: In execution — Wave 1 (#258, #259) + Wave 2 (#257, #290) active
**Date**: 2026-03-30
**Type**: Milestone trace document — canonical execution reference
**Primary repos**: `ai-agentopia/agentopia-protocol`, `ai-agentopia/agentopia-ui`

---

## 1. Traceability

### GitHub artifacts

**Milestone**: [#15 — P1: Web-App Primary Dual-Lane MVP](https://github.com/ai-agentopia/agentopia-protocol/milestone/15)

**Open issues** (remaining work):
| Issue | Title | Blocking? |
|---|---|---|
| [#257](https://github.com/ai-agentopia/agentopia-protocol/issues/257) | Workflow conversation API | Yes — net-new backend |
| [#258](https://github.com/ai-agentopia/agentopia-protocol/issues/258) | Boundary 1: remove start_delivery from model | Yes — security |
| [#259](https://github.com/ai-agentopia/agentopia-protocol/issues/259) | Boundary 3: sidecar executionClass propagation | Yes — cross-repo runtime |
| [#263](https://github.com/ai-agentopia/agentopia-protocol/issues/263) | Stale doc alignment | No — parallel |
| [#264](https://github.com/ai-agentopia/agentopia-protocol/issues/264) | E2E proof (5 scenarios) | Yes — release gate |
| [#290](https://github.com/ai-agentopia/agentopia-protocol/issues/290) | Workflow conversation frontend panel | Yes — depends on #257 |

**Closed issues** (cleanup 2026-03-29):
| Issue | Title | Closure reason |
|---|---|---|
| [#254](https://github.com/ai-agentopia/agentopia-protocol/issues/254) | Auth session layer | Implemented in code |
| [#255](https://github.com/ai-agentopia/agentopia-protocol/issues/255) | Communication API surface | Implemented in code |
| [#256](https://github.com/ai-agentopia/agentopia-protocol/issues/256) | Workflow API surface | Implemented in code |
| [#260](https://github.com/ai-agentopia/agentopia-protocol/issues/260) | Frontend: app shell + Communication UI | Implemented in code, partially validated |
| [#261](https://github.com/ai-agentopia/agentopia-protocol/issues/261) | Frontend: Workflow UI | Implemented in code; conversation panel split to #290 |
| [#262](https://github.com/ai-agentopia/agentopia-protocol/issues/262) | Admin link posture | Stale — `/ui` endpoint removed |

### Related docs
- [P1 Execution Authorization](../architecture/p1-execution-authorization.md) — boundary design
- [Architecture Overview](../architecture/overview.md) — system context
- [Chatbot Architecture](../architecture/chatbot-architecture.md) — component reference

### Deferred milestones
- **Wave F** ([#11](https://github.com/ai-agentopia/agentopia-protocol/milestone/11)) — post-P1 enhancement
- **Wave G** ([#12](https://github.com/ai-agentopia/agentopia-protocol/milestone/12)) — post-MVP / future

---

## 2. Purpose

P1 establishes Agentopia as a **web-app-primary product** with two distinct interaction lanes:

- **Communication lane**: Human-to-bot conversation through the browser. Free-form chat, SSE/WebSocket streaming, conversation persistence.
- **Workflow lane**: Structured delivery orchestration through the browser. Start workflows, track phases, view artifacts, converse with orchestrator in workflow context.

P1 also enforces three execution boundaries:
1. **Boundary 1** — Only the Workflow UI form can start delivery. The LLM model surface must not include `start_delivery`.
2. **Boundary 2** — Worker bots in Communication mode cannot write code (governance enforces 403 on write tools without `workflow_dispatch`).
3. **Boundary 3** — Worker/reviewer bots executing through sidecar dispatch receive `executionClass: workflow_dispatch`, enabling write operations.

This document is the **canonical P1 tracking reference**. Future milestone or issue updates should reference it.

---

## 3. Current Status Snapshot

### Why P1 was rescoped (2026-03-29)

P1 originally had 11 open issues. Assessment revealed that 6 of the 11 issues described work that already existed in code — the issues were created without accounting for existing implementation in `agentopia-protocol` and `agentopia-ui`.

After cleanup:
- **6 issues closed** as implemented in code or stale
- **1 new issue created** (#290, split from #261 for workflow conversation frontend)
- **6 issues remain open** — representing the actual remaining work
- **Milestone description updated** to reflect rescoped reality

### What this means
P1 is now clean enough for implementation to continue. The remaining 6 issues are well-scoped, with clear dependencies and an identifiable critical path. The release gate is #264 (E2E proof of 5 scenarios through the web UI).

---

## 4. Evidence Model

This document and all P1 issue closures use three evidence levels:

| Level | Definition | What qualifies |
|---|---|---|
| **Implemented in code** | Source code exists in the correct module, with expected function signatures and data models | File paths, function names, route definitions |
| **Partially validated** | Some tests or frontend wiring confirm individual behaviors | Unit tests, component tests, frontend hooks wired to backend endpoints |
| **Runtime/E2E proven** | Actual evidence from a running system confirms the full behavior chain | Test output, screenshots, logs, workflow completion records |

**Hard rule**: Do not claim runtime/E2E proof without attaching explicit evidence. "Implemented in code" is not proof that a feature works end-to-end.

---

## 5. Delivered Reality

### Auth session layer (closed #254)
**Evidence level**: Implemented in code, partially validated

The auth session layer exists:
- Backend: `auth/router.py` provides `POST /login`, `POST /logout`, `GET /me`
- Backend: `auth/session.py` provides cookie-based session store with expiry
- Backend: `auth/guards.py` provides `require_auth` FastAPI dependency
- Frontend: `AuthContext` + `LoginPage` wired to auth endpoints
- Frontend test: `auth.test.tsx` validates login form, authenticated content, anonymous fallback

No dedicated backend integration tests exist for auth router endpoints.

### Communication lane (closed #255)
**Evidence level**: Implemented in code, partially validated

The Communication API surface exists:
- Backend: `routers/chat.py` provides `POST /messages`, `POST /messages/stream` (SSE), conversation CRUD
- Backend: `routers/ws_proxy.py` provides bidirectional WebSocket proxy to gateway
- Frontend: `CommunicationPage` with conversation list, chat input, streaming display
- Frontend test: `stream.test.ts` validates SSE client-side parsing

**Important caveat**: The current Communication UI primarily uses the **WebSocket/gateway path** for real-time chat. The SSE endpoint exists in backend code and the frontend has a tested SSE client utility, but the SSE path is not proven by current UI behavior. Both transports exist in code.

### Workflow lane (closed #256, #261)
**Evidence level**: Implemented in code, partially validated

The Workflow API surface and frontend exist:
- Backend: `routers/workflows.py` provides `GET /workflows`, `GET /workflows/{id}`, `GET /workflows/{id}/artifacts`
- Frontend: `WorkflowList`, `WorkflowComposer` (start form), `WorkflowDetail` with `PhaseTimeline`, `ArtifactPanel`
- Frontend: role-based gating via `canStartWorkflow()` — only orchestrator bots see the start button
- Frontend tests: `intent.test.ts`, `workflow-live-status.test.ts`

**Important caveat**: The workflow-scoped conversation panel does not exist. It was split to #290 and depends on #257 (backend API).

### Frontend app shell (closed #260)
**Evidence level**: Implemented in code, partially validated

The unified web app shell exists:
- `AppShell.tsx` with persistent sidebar, workspace header, tab navigation
- `BotSidebar` with role badges, status indicators
- Responsive design with mobile drawer
- Login flow with session cookie management
- Component tests: `auth.test.tsx`, `sidebar.test.tsx`, `gateway-events.test.ts`

---

## 6. Remaining Scope

### #257 — Workflow conversation API
**What**: Backend API for workflow-scoped messages: `POST /api/v1/workflows/{id}/messages`, `GET /api/v1/workflows/{id}/messages`. Includes `workflow_messages` table. REST transport with cursor-based pagination.
**Implemented scope**: Persisted message thread (create + list). No context injection, no SSE streaming, no LLM orchestrator prompting. Those are deferred.
**Dependencies**: None.
**Blocking**: Yes — #290 (frontend) and #264 (E2E proof) depend on this.

### #258 — Boundary 1: remove start_delivery from model
**What**: Remove `start_delivery` tool from the gateway wf-bridge extension. Update orchestrator SOUL templates to reference Workflow UI instead. Verify model cannot initiate delivery from Communication mode.
**Status**: Implemented. `start_delivery` tool removed from `wf-bridge/index.ts`. Orchestrator SOUL template and A2A section updated. Stale references in activation.py, tests, and comments cleaned.
**Dependencies**: None.
**Blocking**: Yes — security boundary required for #264.

### #259 — Boundary 3: sidecar executionClass propagation
**What**: Ensure `executionClass: workflow_dispatch` propagates through the sidecar dispatch chain in the gateway runtime so that governance-bridge can distinguish workflow execution from general chat.
**Why open**: Cross-repo work (code changes land in the private `agentopia` runtime repo). Current propagation status is unverified.
**Dependencies**: None (independent workstream in separate repo).
**Blocking**: Yes — without this, worker/reviewer bots cannot execute delivery tasks.

### #263 — Stale doc alignment
**What**: Update `delivery-start-contract.md` and `overview.md` to reflect the P1 web-app-primary model. Remove references to model-driven delivery start and Telegram-primary assumptions.
**Why open**: These docs still contain pre-P1 architecture assumptions.
**Dependencies**: None.
**Blocking**: No — can proceed in parallel with all implementation work.

### #290 — Workflow conversation frontend panel
**What**: Message panel within `WorkflowDetail` for workflow-scoped messages. REST + React Query polling. Not a real-time LLM conversation surface — that is deferred.
**Implemented scope**: Message list + composer, 10s polling, user/assistant/system role display. Separate from Communication lane (no gateway-ws, no conversation-store).
**Dependencies**: #257 backend API.
**Blocking**: Yes — required for #264 E2E scenario coverage.

### #264 — E2E proof (5 scenarios)
**What**: Execute and document 5 integrated scenarios through the web UI proving all 3 boundaries work together.
**Why open**: This is the release gate. No scenarios have been executed.
**Dependencies**: ALL other remaining issues (#257, #258, #259, #290).
**Blocking**: This IS the acceptance gate. P1 is not complete until #264 passes.

---

## 7. Boundary Tracking

### Boundary 1 — Workflow start surface
**Intent**: Only the Workflow UI form can start delivery. The model cannot call `start_delivery`.
**Current status**: Enforced. `start_delivery` tool removed from `wf-bridge/index.ts`. SOUL templates and A2A section updated. `/wf start` returns "use Workflow UI" hint. Stale references cleaned from activation.py, tests, comments.
**Evidence**: Structurally validated — tests assert tool is absent, prompt contains no positive instruction to use it.

### Boundary 2 — Worker read-only in Communication
**Intent**: Worker bots chatting in Communication mode cannot create delivery artifacts (branches, PRs).
**Current status**: Governance router enforces `execution_class` check. This boundary works IF the gateway correctly stamps `general_chat` for Communication mode requests. This depends on Boundary 3 being implemented.

### Boundary 3 — Sidecar dispatch execution class
**Intent**: Worker/reviewer bots executing via sidecar dispatch receive `executionClass: workflow_dispatch`, enabling write operations.
**Current status**: Unverified. Code changes are in the private `agentopia` runtime repo. #259 tracks verification and fix.

---

## 8. Critical Path

Implementation order, grounded in dependencies:

```
Phase 1 — Parallel starts (no dependencies):
  #258  Boundary 1 (remove start_delivery)
  #259  Boundary 3 (sidecar propagation — cross-repo)
  #257  Workflow conversation API (backend)
  #263  Doc alignment (parallel, non-blocking)

Phase 2 — After #257 completes:
  #290  Workflow conversation frontend panel

Phase 3 — After #258, #259, #257, #290 all complete:
  #264  E2E proof (5 scenarios — release gate)
```

**Shortest critical path**: #257 → #290 → #264
**Risk path**: #259 (cross-repo, unverified status — could surface unexpected complexity)

---

## 9. Deferred Scope

### Wave F ([#11](https://github.com/ai-agentopia/agentopia-protocol/milestone/11)) — Post-P1 enhancement
Multi-step planning approval, streaming output to Telegram, long-running agent loops, workflow notifications, worker clarification channel. These improve delivery quality and operator experience but are not required for the P1 web-app MVP to function. Prioritize after #264 passes.

### Wave G ([#12](https://github.com/ai-agentopia/agentopia-protocol/milestone/12)) — Post-MVP / future
Complex graph branching, LLM-driven routing, multi-packet parallel delivery, multi-agent collaborative conversation. Zero implementation exists. Defer until single-packet MVP is proven in production.

---

## 10. Change Discipline

### For future P1 updates
- Issue closures must state the evidence level (implemented in code / partially validated / runtime/E2E proven)
- Do not close an issue that requires runtime proof based on "implemented in code" alone
- Split ambiguous issues rather than leaving milestone accounting unclear
- Update this document when P1 scope or status changes

### For runtime proof (#264)
- Each of the 5 E2E scenarios must produce documented evidence (screenshots, logs, or test output)
- "It should work" is not evidence
- Partial scenario pass is partial — do not claim full E2E proof

---

## 11. Execution Status (2026-03-30)

### Wave 1 — Boundary blockers

#### #258 Boundary 1 — Remove start_delivery
- `gateway/extensions/wf-bridge/index.ts`: `start_delivery` tool removed; `before_agent_start` hint updated to route `/wf start` → `wf_command` or inform user to use Workflow UI
- `bot-config-api/src/prompts/bot_prompts.py`: `DELIVERY_TEMPLATES["orchestrator"]` updated — no start_delivery reference; SYSTEM_PROMPT Hard Rules updated; `build_a2a_section()` updated
- **Evidence level**: Implemented in code (PR pending)

#### #259 Boundary 3 — executionClass propagation
- **Code-traced propagation chain** (verified 2026-03-30):
  1. `agentopia-core/src/gateway/server-runtime-state.ts:201` — dispatch port server created with `executionClass: "workflow_dispatch"`
  2. `agentopia-core/src/gateway/server-http.ts:551` — stamps `req.__executionClass` (immutable, server-owned)
  3. `agentopia-core/src/gateway/openai-http.ts:245-247` — reads stamp, defaults `"general_chat"`
  4. `agentopia-core/src/agents/pi-tools.ts:503,547` — flows through to `OpenClawPluginToolContext.executionClass`
  5. `governance-bridge/index.ts:608` — reads `toolContext?.executionClass` → sends as `execution_class`
  6. `governance/policy.py:141-237` — `check_execution_authorization()` enforces full matrix
- Debug logs in place: `[P1-DEBUG]` in openai-http.ts, `[P1-DEBUG-GOV]` in governance-bridge
- **Evidence level**: Implemented in code, propagation traced across both repos. **NOT runtime-proven.** Runtime verification requires: deploy → trigger sidecar dispatch → check debug logs.

### Wave 2 — Workflow conversation

#### #257 Workflow conversation API
- `bot-config-api/db/021_workflow_messages.sql`: new `workflow_messages` table
- `bot-config-api/src/routers/workflow_conversations.py`: `POST /api/v1/workflows/{id}/messages`, `GET /api/v1/workflows/{id}/messages`
- `bot-config-api/src/main.py`: router registered
- `bot-config-api/src/tests/test_workflow_conversations.py`: model validation, route registration, scope boundary tests
- **Scope**: Persisted message thread only. No context injection, no SSE, no LLM prompting.
- **Evidence level**: Implemented in code, partially validated (tests pass)

#### #290 Workflow conversation frontend
- `agentopia-ui/src/components/workflow/WorkflowConversation.tsx`: new component — message list + composer. REST + React Query polling (10s), NOT WebSocket.
- `agentopia-ui/src/components/workflow/WorkflowDetail.tsx`: Conversation section added between Artifacts and Trello.
- `agentopia-ui/src/hooks/useWorkflows.ts`: `useWorkflowMessages` hook added (React Query, GET /workflows/{id}/messages).
- **Scope**: Persisted workflow-scoped message thread only. NOT a real-time conversation loop with LLM orchestrator behavior. No bot is prompted via this surface.
- **Evidence level**: Implemented in code, partially validated (scope tests pass)

### Wave 3 — Doc alignment

#### #263 Stale doc alignment
- `docs/architecture/overview.md`: updated — delivery-start contract = Workflow UI only
- `docs/architecture/p1-execution-authorization.md`: updated — current implementation status
- `docs/operations/delivery-workflow-gaps.md`: updated — Boundary 1 fix noted
- **Evidence level**: Implemented in code

### Wave 4 — E2E proof (#264)

Not yet started. Requires all Wave 1–3 work deployed to dev. Gate criteria:
1. Communication lane works (chat, streaming)
2. Workflow start via Workflow UI → delivery workflow created
3. No model-start delivery possible (start_delivery tool absent from tool list)
4. Workflow conversation panel works (send message, receive response, separate from comm lane)
5. Worker/reviewer bot governance tool call during sidecar dispatch shows `executionClass=workflow_dispatch` in logs

## 12. Milestone Decision

P1 is **in progress**. Boundary 1 (#258) is cleaned, workflow conversation (#257/#290) delivers a persisted message thread (minimal viable scope), executionClass (#259) is code-traced but not runtime-proven.

**Not code-complete yet.** #257/#290 delivers only a persisted message thread — the full workflow conversation behavior (context injection, LLM orchestrator prompting, SSE streaming) is deferred scope. #259 requires runtime verification.

P1 is **not release-ready** until #264 evidence is attached.
