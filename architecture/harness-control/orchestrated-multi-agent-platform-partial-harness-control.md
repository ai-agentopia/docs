---
title: "Baseline: Orchestrated Multi-Agent Platform with Partial Harness Control"
---

# Baseline: Orchestrated Multi-Agent Platform with Partial Harness Control

**Status:** Approved baseline for implementation planning  
**Owner:** Platform Architecture  
**Scope:** `ai-agentopia/docs`, `agentopia-core`, `agentopia-protocol`, `agentopia-infra`  
**Purpose:** Describe what Agentopia already is today, what harness-control properties it already has, what remains missing, and why the next harness milestone exists.

---

## 1. Executive Summary

Agentopia today is no longer a simple chatbot gateway. It already operates as an **orchestrated multi-agent platform** with several real harness-control properties:
- a central control plane in `bot-config-api`
- orchestrated A2A threads with turn limits, checkpoints, and recovery semantics
- runtime execution authorization for delivery/governance actions
- delegated runtimes through `sessions_spawn` and ACP harness sessions
- lane separation in the product between communication and workflow execution

At the same time, Agentopia does **not yet** have a unified production harness that governs all agentic execution through a single contract. Control currently exists in several strong but separate islands: workflow authorization, A2A orchestration, subagent/ACP spawning, and the newly-approved runtime-safety baseline. Those pieces are valuable, but they do not yet amount to one coherent bounded-autonomy system.

The correct baseline description for the platform is therefore:

> **Agentopia is an orchestrated multi-agent platform with partial harness control.**

This document locks that assessment as the current-state baseline. It also explains why the next milestone is not “more prompt tuning,” but a harness-control-plane milestone that unifies deterministic front doors, autonomous run contracts, artifacts, checkpoints, traces, and evaluation.

---

## 2. Baseline Question

This document answers one precise question:

**How far has Agentopia already gone toward a true harnessed agentic system, and what is still missing before it can be called a bounded-autonomy production platform?**

It is intentionally different from the runtime-facts baseline.

- The [Runtime Facts, Capability Classes, and Runaway Tool Prevention baseline](./runtime-facts-capability-classes-baseline.md) defines the approved safety model for tool surfaces, runtime facts, and runaway-loop prevention.
- This document defines the current platform-level harness maturity around orchestration, execution control, runtime selection, and autonomy boundaries.

---

## 3. External Research Basis

This baseline is informed by current official guidance from OpenAI and Anthropic.

### 3.1 OpenAI: harness engineering and agent scaffolding

OpenAI’s recent engineering guidance emphasizes that production agent systems are not just prompts plus tools. They rely on:
- repository knowledge as a system of record
- explicit scaffolding and feedback loops
- humans steering while agents execute
- control systems that keep autonomous work coherent at scale

Primary sources:
- [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- [New tools for building agents](https://openai.com/index/new-tools-for-building-agents/)
- [Agents guide](https://developers.openai.com/api/docs/guides/agents)
- [A practical guide to building agents (PDF)](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf)

### 3.2 Anthropic: workflows vs agents, and harness design for long-running work

Anthropic’s guidance makes two points especially relevant here:
- there is a fundamental distinction between **workflows** (predefined code paths) and **agents** (model-directed loops)
- the most effective harnesses use structured artifacts, decomposed tasks, mechanical guardrails, and evaluator/checkpoint loops only where they add measurable value

Primary sources:
- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)

### 3.3 Converged external pattern

Across these sources, the converged production pattern is:

1. Start simple; do not default to open-ended autonomy.
2. Use deterministic code paths when the contract is already known.
3. Use agentic loops only where flexibility is actually needed.
4. Narrow tools and capabilities by role and task.
5. Attach mechanical stop conditions, traces, and evals.
6. Use structured artifacts rather than relying on raw chat continuity.

That converged pattern is the reference frame for the assessment below.

---

## 4. Internal Evidence Base

This assessment is grounded in current Agentopia design docs and code-level behavior.

### Core internal references

- [Multi-Bot Architecture](../architecture-multiple-bot.md)
- [Agent-to-Agent Protocol](../a2a-full-design.md)
- [Execution Authorization](../p1-execution-authorization.md)
- [Deterministic Delivery Start Front Door](../p0.5-deterministic-delivery-start.md)
- [Runtime Facts, Capability Classes, and Runaway Tool Prevention](./runtime-facts-capability-classes-baseline.md)
- [Agent Harness Control Plane & Bounded Autonomy](../../milestones/agent-harness-control-plane.md)

### Key code signals

- `agentopia-protocol/bot-config-api` already acts as a control-plane service for bots, relay, workflows, credentials, and governance.
- `governance-bridge` already propagates a trusted `execution_class` boundary and includes same-turn dedupe for repeated read-only governance calls.
- A2A threads already have turn ordering, max-turn control, checkpoints, idempotency, and recovery semantics.
- `sessions_spawn` and `runtime: "acp"` already exist as advanced delegated runtimes with lifecycle and sandbox constraints.
- The newly approved runtime-facts baseline is still an implementation baseline, not yet a fully shipped platform-wide harness.

---

## 5. Baseline Assessment

### 5.1 What Agentopia already is

Agentopia is already a **multi-agent orchestration platform** with real execution boundaries in some domains.

It has moved beyond:
- single-bot chat
- prompt-only coordination
- uniform tool access with no policy

It already has:
- role-aware bot provisioning
- workflow-orchestrated delivery
- structured inter-agent communication
- domain-specific runtime authorization
- delegated subagent and ACP runtime primitives

### 5.2 What Agentopia is not yet

Agentopia is **not yet** a unified bounded-autonomy harness platform.

Specifically, it does not yet have one coherent system that answers, for every serious execution path:
- whether this path must be deterministic or may be autonomous
- which runtime is allowed
- what the run contract is
- what artifacts it must emit
- where human checkpoints apply
- what trace and eval contract must exist

---

## 6. Current Harness Maturity Matrix

| Layer | Current state | Assessment |
|---|---|---|
| L1. Agent runtime primitives | Strong | Agentopia already has tools, relay, A2A, subagents, ACP runtime, memory, workflow integration |
| L2. Domain-specific execution control | Strong in pockets | Governance/delivery already has runtime authorization boundaries and workflow semantics |
| L3. Cross-agent orchestration | Strong | A2A thread model, checkpoints, max turns, idempotency, recovery are real harness behaviors |
| L4. Unified role/capability safety model | Approved but not fully shipped | Runtime-facts baseline defines the right target, but it is still a baseline-to-implementation layer |
| L5. Unified autonomy harness | Missing | No platform-wide lane router, run contract, artifact taxonomy, checkpoint policy matrix, or unified trace/eval contract |
| L6. Bounded-autonomy production platform | Not yet | The system is close in ingredients, but not yet coherent at the harness level |

### Locked baseline statement

The platform baseline is therefore:

> **Agentopia is an orchestrated multi-agent platform with partial harness control.**

---

## 7. What Is Already Harnessed Well

### 7.1 Control plane exists for the fleet

`bot-config-api` is already more than a CRUD service. It acts as a real platform control plane for:
- bot lifecycle
- inter-agent communication
- collaborative threads
- workflow orchestration
- governance signaling
- credential governance

That is a real prerequisite for a harnessed platform, and Agentopia already has it.

### 7.2 A2A already behaves like a harnessed collaboration runtime

The A2A design already encodes several harness properties:
- turn-based coordination
- thread lifecycle state
- maximum turns
- checkpointing
- idempotent turns
- recovery and queue fallback
- compaction/epoch summaries

This is not a naive “agents can just talk to each other” model. It is already orchestrated and bounded.

### 7.3 Delivery/governance already has a runtime authorization boundary

The execution-authorization model is one of Agentopia’s strongest harnessed subsystems.

It already separates:
- `general_chat`
- `workflow_dispatch`

And binds action classes to execution classes and roles before tool execution. That is the right shape of production enforcement: the runtime boundary, not the prompt, is authoritative.

### 7.4 Delegated runtimes already exist

`sessions_spawn` and ACP harness support mean Agentopia already has:
- nested execution
- delegated work
- runtime specialization
- child lifecycle controls
- explicit ACP-specific guidance and restrictions

This means the platform already owns the idea of “one agent can launch another bounded execution context.”

### 7.5 Some local anti-loop and anti-redundancy logic already exists

The governance bridge’s same-turn dedupe is an example of local harness logic:
- repeated read-only retrievals with identical args are cached in-run
- the model is explicitly told to stop calling the same tool again

This is not yet a platform-wide harness rail, but it shows the system is already encoding control in runtime behavior, not only in prompt text.

---

## 8. Where Harness Control Is Still Partial

### 8.1 No unified deterministic-vs-agentic lane router

Agentopia has several good boundaries, but it does not yet apply one platform rule for all entry paths.

Today, the platform has a mixture of:
- deterministic workflow semantics in some places
- soft prompt-compliance in others
- subsystem-specific orchestration decisions

The system still lacks a single answer to:

**Which intents must enter through deterministic code-owned front doors, and which may enter autonomous loops?**

This is the most important missing harness boundary.

### 8.2 No universal run contract

Autonomous work does not yet flow through one shared run contract that always carries:
- goal type
- runtime
- capability class
- wall-clock budget
- turn/tool budget
- stop conditions
- expected artifact shape
- escalation policy

Without that, autonomy is still bounded locally rather than uniformly.

### 8.3 No structured artifact taxonomy across all autonomous work

Agentopia has summaries, workflow records, and thread context. But it does not yet have one explicit artifact model for:
- task packets
- execution plans
- checkpoint summaries
- evaluator findings
- final run outcomes

That means some context handoff still depends on subsystem-specific memory or raw conversational continuity.

### 8.4 No unified checkpoint policy matrix

A2A supports checkpoints, and workflow governance supports controlled transitions, but the platform still lacks one policy matrix that says:
- which run classes require human approval
- at what boundary
- with what artifact
- before which irreversible action

### 8.5 No first-class trace/eval contract for every serious autonomous run

Agentopia has platform observability, but not yet a universal harness-level run trace and eval contract.

The missing layer is a standard record of:
- run metadata
- stop reason
- tool timeline
- artifact outputs
- checkpoint events
- eval outcome by task class

### 8.6 ACP remains powerful but not yet fully governed as a platform lane

ACP support already exists, but it is still closer to a powerful runtime primitive than to a fully-governed lane in the platform-wide autonomy model.

That is acceptable at the current maturity stage, but it should not be mistaken for a complete harness.

---

## 9. Relationship To The Runtime-Facts Baseline

The runtime-facts baseline remains correct and necessary. It fixed a concrete class of failure:
- unsafe self-introspection
- overly broad session-tool access
- missing generic anti-runaway rails

But that baseline operates at the **tool-surface safety** layer.

This document defines the platform baseline at the **execution-harness maturity** layer.

The distinction is:

| Baseline | Main question |
|---|---|
| Runtime facts / capability classes / runaway prevention | What may the agent call, and how do we stop bad tool loops? |
| Partial harness control baseline (this doc) | How much of execution is already truly harnessed, and what must be unified next? |

---

## 10. Baseline Conclusions

### 10.1 What can be claimed today

Agentopia can defensibly claim all of the following today:

1. It is a multi-agent platform, not a single-agent chatbot.
2. It already has real orchestrator-driven collaboration semantics.
3. It already has a real control plane.
4. It already has at least one strong runtime authorization boundary for high-value actions.
5. It already has delegated runtimes and bounded subagent primitives.
6. It already recognizes that prompt compliance is not an adequate security or correctness boundary.

### 10.2 What cannot yet be claimed

Agentopia cannot yet claim that it has:

1. a unified autonomy harness across all execution lanes
2. deterministic front doors as a general platform rule
3. a universal run contract for autonomous execution
4. platform-wide artifact handoff semantics
5. a unified checkpoint matrix
6. a first-class run trace/eval contract for all serious autonomous work

That is why the next milestone is necessary.

---

## 11. Platform Baseline Decision

The current platform baseline is locked as:

> **Agentopia is an orchestrated multi-agent platform with partial harness control.**

This wording is deliberate.

- **orchestrated**: because the platform already coordinates bots, threads, workflows, and delegated runtimes
- **multi-agent**: because collaboration and delegation are built-in, not bolted on
- **partial harness control**: because some critical domains are already governed at runtime, but autonomy is not yet unified under one control-plane contract

This baseline should be used in planning, architecture discussion, and milestone scoping.

---

## 12. What The Next Milestone Must Do

The next milestone is correctly framed as **Agent Harness Control Plane & Bounded Autonomy**.

Its job is not to reinvent the system. Its job is to unify what already exists into one coherent harness model:
- deterministic front doors where contracts are already known
- autonomous lanes where flexibility is truly needed
- run contracts for all harnessed execution
- artifact-based handoff for long-running and multi-session work
- checkpoint policy matrix
- run traces and task-class evals
- ACP as an explicitly governed advanced runtime lane

That milestone is already tracked here:
- [Agent Harness Control Plane & Bounded Autonomy](../../milestones/agent-harness-control-plane.md)

---

## 13. Final Verdict

Agentopia has already crossed the line from “prompted bot system” into “orchestrated multi-agent platform.” That is a material achievement.

However, the platform’s harness properties are still unevenly distributed across subsystems. The next architectural step is not another isolated safety patch. It is the unification of those existing control islands into a bounded-autonomy harness model.

Until that milestone lands, the correct and defensible baseline description remains:

> **Agentopia is an orchestrated multi-agent platform with partial harness control.**
