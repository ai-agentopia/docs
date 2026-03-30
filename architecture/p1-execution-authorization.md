---
title: "Execution Authorization"
description: "Dual-lane security model that separates consultation from delivery through deterministic execution authorization, ensuring delivery actions only occur within trusted workflow contexts."
---


## Problem

In a multi-agent system where bots communicate freely, the same tools can serve two very different purposes:

- **Consultation:** "What do you think about this approach?"
- **Delivery:** "Create a branch and implement this feature"

If both paths use the same authorization, delivery correctness depends on LLM prompt compliance rather than deterministic policy. A model can bypass workflow controls by delegating delivery work through a consultation channel.

Execution authorization solves this by enforcing delivery boundaries at the runtime level, not the prompt level.

## Action Classes

Every tool invocation is classified into exactly one action class:

| Class | Description | Examples |
|---|---|---|
| `consultation_read` | Read-only queries about code, issues, PRs | `get_file_contents`, `search_code`, `list_issues` |
| `execution_write` | Delivery actions that create artifacts | `create_branch`, `push_files`, `create_pull_request` |
| `review_write` | Review actions that affect PR state | `create_pr_review` |
| `admin_write` | Planning and governance actions | `create_issue`, `merge_pull_request` |
| `audit_read` | Audit and inspection queries | `audit_milestone`, `audit_repo` |

## Execution Classes

Every request to the agent execution engine is classified into one of two classes:

| Class | Meaning |
|---|---|
| `workflow_dispatch` | Request originates from a trusted workflow channel. Verified by the runtime, not caller-asserted. |
| `general_chat` | All other requests: consultation, relay, direct API, messaging integrations. |

## Authorization Matrix

| Action Class | Execution Class Required | Role Required |
|---|---|---|
| `consultation_read` | any | any bound role |
| `execution_write` | `workflow_dispatch` only | worker |
| `review_write` | `workflow_dispatch` only | reviewer |
| `admin_write` | any | orchestrator |
| `audit_read` | any | orchestrator |

## Hard Rules

1. `execution_write` and `review_write` require `workflow_dispatch`. No exceptions.
2. `admin_write` is role-authorized only. No workflow dispatch context required.
3. `consultation_read` is always allowed for any bound actor.
4. Execution class is determined by the runtime, not by request content or caller assertion.
5. General chat callers can read and discuss but cannot create delivery artifacts.
6. Prompt guidance may improve UX but is not the security boundary.

## Dual-Lane Model

The system operates in two semantic lanes:

**Lane A -- Consultation.** Bot-to-bot questions, brainstorming, status discussion, specialist input. No delivery artifacts are created. Uses `consultation_read` tools only.

**Lane B -- Delivery.** Build, fix, or change code in a repository. Entry point is the workflow UI, which triggers delivery through a trusted dispatch channel. The workflow engine owns sequencing. Uses `execution_write` and `review_write` tools. The delivery-start action is not exposed on the bot's tool surface -- it can only be initiated through the workflow UI.

Lane selection is a UX concern. Lane enforcement is an execution authorization concern. The authorization matrix above is the hard boundary.

## Runtime Contract

Any agent runtime implementing this model must satisfy five requirements:

### RC-1: Trusted Workflow-Dispatch Channel

The runtime must provide a communication channel between the workflow dispatch component and the agent execution engine that is distinguishable from general chat at a trusted boundary, not spoofable by general chat callers, and stamped by the runtime itself.

### RC-2: Execution Class Stamping

Every request must be stamped with a trusted execution class before tool execution begins. The stamp is immutable once set. Tool handlers cannot override it.

### RC-3: Tool Execution Context Propagation

The runtime must propagate execution class to the tool execution boundary so that authorization decisions occur before the tool action executes.

### RC-4: Channel Isolation

The trusted workflow-dispatch channel must not be reachable by external network callers, relay or agent-to-agent message forwarding paths, or direct user messages. Only the co-located workflow dispatch component may use it.

### RC-5: Consultation Preservation

General chat callers must retain access to `consultation_read` tools. The runtime must not block all tool access for general chat -- only `execution_write` and `review_write`.

## Design Rationale

This model achieves three goals:

1. **Deterministic enforcement.** Delivery actions are gated by runtime-verified execution class, not LLM behavior. A misbehaving model cannot bypass the boundary.

2. **Consultation remains open.** Agents can freely read code, discuss approaches, and consult specialists without workflow overhead. Only artifact-creating actions are restricted.

3. **Runtime-agnostic contract.** The authorization model is defined independently of any specific agent runtime. Any runtime that satisfies RC-1 through RC-5 can implement it.
