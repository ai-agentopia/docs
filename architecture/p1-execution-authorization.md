---
title: "Execution Authorization Architecture"
---

# Execution Authorization Architecture

> Status: FINAL (CTO-approved 2026-03-20)
> Scope: Dual-lane orchestration — consultation vs delivery enforcement

---

## 1. Problem Statement

Agentopia's relay/A2A mechanism is semantically overloaded. The same tools can be used for:
- **Consultation:** "What do you think about this approach?"
- **Delivery delegation:** "Create a branch and implement this feature"

This makes delivery correctness depend on LLM prompt compliance rather than deterministic policy enforcement. A weaker model (or even a strong model on a bad day) can bypass the workflow by directly delegating delivery work via relay instead of using the canonical `start_delivery` path.

The architecture must separate consultation from delivery enforcement by **execution-level authorization**, not by prompt guidance alone.

---

## 2. Architecture Contract (Runtime-Agnostic)

This section defines the authorization model. It does NOT reference any specific runtime (OpenClaw, LangGraph, etc.).

### 2.1 Action Classes

Every tool invocation is classified into exactly one action class:

| Class | Description | Examples |
|---|---|---|
| `consultation_read` | Read-only queries about code, issues, PRs | get_file_contents, search_code, list_issues, get_pr_files, get_pr_status, get_pr_reviews, get_pr_comments, list_commits |
| `execution_write` | Code delivery actions that create artifacts | create_branch, push_files, create_or_update_file, create_pull_request, update_pr_branch |
| `review_write` | Review actions that affect PR state | create_pr_review |
| `admin_write` | Planning/governance actions by orchestrator | create_milestone, create_issue, update_milestone, close_milestone, update_issue, close_issue, add_issue_comment, merge_pull_request |
| `audit_read` | Audit/inspection by orchestrator | audit_milestone, audit_repo |

### 2.2 Execution Classes

Every request to the agent execution engine is classified into exactly one execution class:

| Class | Meaning |
|---|---|
| `workflow_dispatch` | Request originates from a trusted workflow execution channel. Verified by the runtime, not caller-asserted |
| `general_chat` | All other requests: consultation, chat, relay, direct API, Telegram messages |

### 2.3 Authorization Matrix

| Action Class | Execution Class Required | Role Required |
|---|---|---|
| `consultation_read` | any | any bound role |
| `execution_write` | `workflow_dispatch` only | worker |
| `review_write` | `workflow_dispatch` only | reviewer |
| `admin_write` | any | orchestrator |
| `audit_read` | any | orchestrator |

### 2.4 Hard Rules

1. `execution_write` and `review_write` MUST require `workflow_dispatch`. No exceptions.
2. `admin_write` is role-authorized only. No workflow dispatch context required.
3. `consultation_read` is always allowed for any bound actor.
4. `execution_class` is determined by the runtime, not by request content or caller assertion.
5. `general_chat` callers can read and discuss but CANNOT create delivery artifacts.
6. Prompt/SOUL guidance may help UX but is NOT the security boundary.

### 2.5 Dual-Lane Semantic Model

The system supports two semantic lanes:

- **Lane A — Consultation:** Bot-to-bot questions, brainstorming, status discussion, specialist consultation. No delivery artifacts created. Uses `consultation_read` tools.
- **Lane B — Delivery:** Build/fix/change code in repository. Canonical entrypoint: `start_delivery`. Workflow/state machine owns sequencing. Uses `execution_write` and `review_write` tools.

Lane selection is a UX/semantic concern. Lane enforcement is an execution authorization concern. The authorization matrix (section 2.3) is the hard boundary, not lane detection.

---

## 3. Runtime Contract

Any agent runtime that implements this architecture MUST provide these capabilities:

### RC-1: Trusted Workflow-Dispatch Channel

The runtime must provide a communication channel between the workflow dispatch component and the agent execution engine that is:
- Distinguishable from general chat at the **trusted runtime boundary**
- Not spoofable by general chat callers
- Stamped by the runtime itself, not by request body content

### RC-2: Execution Class Stamping

The runtime must stamp every request with a trusted `execution_class` before tool execution begins:
- Requests arriving via the trusted workflow-dispatch channel: `workflow_dispatch`
- All other requests: `general_chat`

The stamp must be immutable once set. Tool handlers must not be able to override it.

### RC-3: Tool Execution Context Propagation

The runtime must propagate `execution_class` to the tool execution boundary so that:
- Tool handlers (or the authorization layer they call) can read the trusted `execution_class`
- Authorization decisions are made with this context before the tool action executes

### RC-4: General Chat Cannot Reach Workflow-Dispatch Channel

The trusted workflow-dispatch channel must not be reachable by:
- External network callers
- Relay/A2A message forwarding paths
- Direct user/Telegram messages

Only the co-located workflow dispatch component (sidecar) may use it.

### RC-5: Consultation Preservation

General chat callers must retain access to `consultation_read` tools. The runtime must not block all tool access for general chat — only `execution_write` and `review_write` tools.

---

## 4. Implementation Status (OpenClaw Runtime)

**Chosen approach:** Option A — dedicated loopback port (dispatch port).

### Propagation chain (verified in code, 2026-03-30)

| Layer | File | Mechanism |
|---|---|---|
| 1. Dispatch server created | `agentopia-core/src/gateway/server-runtime-state.ts:201` | `executionClass: "workflow_dispatch"` passed to server constructor |
| 2. Request stamp | `agentopia-core/src/gateway/server-http.ts:551` | `req.__executionClass = opts.executionClass` (immutable, server-owned) |
| 3. Agent input builder | `agentopia-core/src/gateway/openai-http.ts:245-247` | Reads stamp, defaults `"general_chat"` for non-dispatch |
| 4. Tool context propagation | `agentopia-core/src/agents/pi-tools.ts:503,547` | `executionClass` flows through tool factory → `OpenClawPluginToolContext` |
| 5. governance-bridge reads | `agentopia-protocol/gateway/extensions/governance-bridge/index.ts:608` | `toolContext?.executionClass \|\| "general_chat"` → sends as `execution_class` in body |
| 6. Backend enforcement | `agentopia-protocol/bot-config-api/src/governance/policy.py:141-237` | `check_execution_authorization()` evaluates full matrix |

### Runtime verification

Debug logs are in place:
- `[P1-DEBUG] executionClass=...` in `openai-http.ts:250-252`
- `[P1-DEBUG-GOV] tool=... toolContext.executionClass=...` in `governance-bridge/index.ts:609`

Runtime verification requires: deploy to dev → trigger sidecar dispatch → check logs for `executionClass=workflow_dispatch`. Then trigger Communication chat → check logs for `executionClass=general_chat`.

---

## 5. Migration Status

| Phase | Status | Notes |
|---|---|---|
| **Tactical** | Complete | Actor-level dispatch check in governance router |
| **P1** | Implemented in code | RC-1 through RC-5 implemented. Option A (dispatch port). Runtime verification pending. |
| **P2 (hardening)** | Deferred | Dispatch capability token (Biscuit). Per-tool cryptographic authorization. |

---

## 6. Relationship to Permanent Dual-Lane Model

P1 enables the **execution-safe dual-lane model**:
- After P1: A2A consultation reads are OK, delivery writes denied without active workflow dispatch
- After P1: orchestrator can freely use relay for consultation while delivery is enforced through `start_delivery`
- **Start-path integrity NOT solved by P1** — orchestrator can still be prompted to bypass `start_delivery` via creative user input. This is an accepted gap until gateway fork (milestone #25) enables deterministic Telegram routing
- P1 is the **active enforcement milestone**. It is NOT "production-grade start-path architecture" — that requires gateway fork

---

## 7. References

- [Delivery Start Contract](../contracts/../contracts/delivery-start-contract.md)
- [Review Verification](../contracts/review-verification.md)
- [Bot Activation](../contracts/bot-activation.md)
- [Runtime Boundary Plan](../contracts/runtime-boundary/fully-swappable-runtime-boundary-plan.md)
- [Architecture Overview](./overview.md)
