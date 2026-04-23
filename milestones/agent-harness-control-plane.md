---
title: "Agent Harness Control Plane & Bounded Autonomy"
---

**Status**: Approved — planning baseline
**Date**: 2026-04-22
**Type**: Cross-repo milestone design
**Primary repos**: `ai-agentopia/docs`, `ai-agentopia/agentopia-core`, `ai-agentopia/agentopia-protocol`, `ai-agentopia/agentopia-infra`

## 1. Purpose

Define the next milestone after the runtime-facts / capability-class baseline: a production harness for agentic execution.

This milestone exists because the current baseline closes one class of failure — unsafe runtime self-introspection and runaway tool loops — but it does **not yet** define the full control plane for when an LLM is allowed to dynamically drive execution, how long it may do so, what contract it must satisfy while running, and where deterministic front doors must replace prompt-compliance.

The milestone sets the implementation target for **bounded autonomy**:
- humans steer, agents execute
- deterministic paths are used when the business contract is typed and known in advance
- autonomous loops are used only where model-driven flexibility is actually required
- every autonomous run has explicit budgets, stop conditions, artifacts, and observability

## 2. Why This Milestone Exists

Recent failures and existing design docs point to the same gap:

1. The `session_status` incident showed that a bot can be given a broad tool surface and a soft prompt contract without a sufficiently strong execution harness.
2. The deterministic delivery-start workstream already established that some user intents cannot safely rely on prompt compliance alone; a typed front door is required when the contract is known in advance.
3. Existing A2A and ACP-harness guidance describes orchestration patterns, but not a unified control boundary for specialist bots, orchestrators, ACP sessions, and workflow-triggering paths.

The missing piece is not “more prompt engineering.” It is a **harness/control-plane layer** that decides:
- when the system uses deterministic code paths vs agentic loops
- what runtime an autonomous task is allowed to use
- how context is handed off across long-running or multi-agent work
- which checkpoints require human judgment
- how traces, evals, and recovery loops are attached to every run

## 3. External Research Synthesis

Current official guidance across frontier vendors converges on a small set of ideas:

### 3.1 Simplicity first, then autonomy

Anthropic explicitly distinguishes **workflows** from **agents**:
- workflows = predefined code paths with LLM/tool steps
- agents = model-directed loops with dynamic tool use

Their recommendation is to start with the simplest solution and increase complexity only when needed. Open-ended agent loops are appropriate only where a fixed path cannot be predicted in advance.

### 3.2 Humans steer; agents execute

OpenAI’s current harness guidance frames the human role as:
- specify intent
- design environments
- define acceptance criteria
- validate outcomes
- improve the scaffolding when the agent fails

This is not prompt-only autonomy. It is harnessed execution with feedback loops.

### 3.3 Repository-local knowledge and structured artifacts matter more than giant prompts

OpenAI’s harness engineering work emphasizes:
- repository knowledge as the system of record
- `AGENTS.md` as a map, not an encyclopedia
- structured documents and execution plans as durable context

Anthropic’s recent harness design for long-running autonomous coding emphasizes:
- decomposing work into tractable chunks
- structured artifacts to hand off context between sessions
- planner / generator / evaluator separation where it measurably improves outcomes

### 3.4 Guardrails are mechanical, not aspirational

Across OpenAI and Anthropic guidance, the pattern is consistent:
- explicit stop conditions / max turns / max iterations
- tool design with tight schemas and clear descriptions
- traces, evals, and human checkpoints
- sandboxing and bounded environments for open-ended execution

### 3.5 Tool and capability surfaces must match role intent

Tool overload and overlapping tools materially degrade model behavior. Current best practice is not “give every agent everything,” but rather:
- expose a narrow tool surface
- split tools by responsibility
- use specialization and orchestration when complexity grows

## 4. Relationship To The Approved Runtime Safety Baseline

The approved baseline remains authoritative for:
- ambient runtime facts
- `session_status` removal from agent-callable registration
- capability classes
- R1 / R2 / R3 runaway-prevention rails

This milestone builds on that baseline and adds the next layer:

| Already locked by baseline | Added by this milestone |
|---|---|
| What tools a class may call | When an agentic loop may exist at all |
| How tool loops are mechanically capped | How entire runs are typed, budgeted, observed, and checkpointed |
| Runtime facts and self-knowledge contract | Deterministic front doors vs autonomous harnessed runs |
| Specialist vs Orchestrator vs Admin capability surfaces | Runtime selection, artifact handoff, evaluator/checkpoint model |

In short:
- the baseline answers **tool-surface safety**
- this milestone answers **execution-harness control**

## 5. Final Product Contract

### In scope

1. Define which user/system intents must enter through deterministic typed front doors.
2. Define which intents are allowed to enter an autonomous agent loop.
3. Define a run contract for every agentic execution path.
4. Define runtime selection rules across direct specialist chat, orchestrator coordination, subagents, and ACP harness sessions.
5. Define structured handoff artifacts for multi-session and long-running work.
6. Define human checkpoints for irreversible or high-judgment actions.
7. Define trace/eval requirements for harnessed runs.
8. Define migration steps from today’s soft prompt-compliance paths.

### Out of scope

1. Full removal of all LLM-driven orchestration from the platform.
2. A general-purpose computer-use runtime for every bot.
3. Product-specific workflow UX beyond what is required to expose checkpoints and run state.
4. Replacing existing capability-class work.
5. Reopening the runtime-facts baseline.

## 6. Architecture Decisions

### 6.1 Distinguish deterministic workflows from autonomous harness runs

The platform will explicitly separate two execution modes:

1. **Deterministic front door**
   - Typed user/system intents with known business contracts
   - Example: start a delivery workflow, request a workflow cancel, inspect bot runtime policy
   - Entry path is code-owned, not model-decided

2. **Harnessed autonomous run**
   - Open-ended work where step count and exact action sequence cannot be predicted upfront
   - Example: code investigation, multi-file patching, exploratory research, orchestrated specialist debate
   - Entry path is explicit and instrumented; the agent is not left to improvise whether a hard contract should exist

This distinction becomes a first-class platform rule.

### 6.2 Typed front doors are mandatory when the contract is already known

If the system can name the user intent and its allowed outcomes in advance, the LLM is not the front door.

This follows the earlier deterministic delivery-start diagnosis and extends it into a general rule.

Examples that should be deterministic or explicitly typed:
- delivery/workflow start
- workflow cancel / retry / close
- policy or approval transitions
- any other path whose success states are finite, typed, and auditable

**Admin mutation is NOT in this list under the currently approved baseline.** Per the [Runtime Facts, Capability Classes, and Runaway Tool Prevention baseline](../architecture/harness-control/runtime-facts-capability-classes-baseline.md) and the concise [Harness Architecture](../architecture/harness-control/harness-architecture.md) verdict, admin mutation is an **Admin-class tool (`admin_mutate`)** with a closed action enum, cap=1/turn, and an audit-log sink. It is not a deterministic workflow ingress. A deterministic-workflow alternative is discussed in the [Harness System — Deep-Dive Debate §10.4](../architecture/harness-control/harness-system-deep-dive-debate.md) as a possible future ADR topic; it is not part of this milestone. This milestone implements the approved tool shape, not the debated alternative.

### 6.3 Every autonomous run must satisfy a run contract

A harnessed run is not just “let the model keep going.” It must carry:
- a typed goal
- an allowed runtime
- a capability class
- a maximum wall-clock duration
- a maximum number of turns / tool calls
- explicit stop conditions
- an expected final artifact shape
- a trace id / run id
- escalation rules for blocker or checkpoint states

No production run may be launched without this contract.

### 6.4 Long-running work uses structured artifacts, not raw conversational continuity

Long-running or multi-session work will hand off through durable artifacts, not by assuming the next agent/session can reconstruct state from raw chat history.

Artifact types include:
- execution plan
- sprint/phase contract
- task packet
- evaluator findings
- checkpoint summary
- final delivery summary

This aligns with both OpenAI’s repository-as-system-of-record guidance and Anthropic’s structured-handoff pattern.

### 6.5 Evaluator/checkpoint patterns are optional by task class, not universal

The platform will not blindly use a planner-generator-evaluator trio everywhere. Instead:
- use a **solo harness** for bounded, verifiable tasks
- add an **evaluator** when quality judgment or verification materially improves outcomes
- require a **human checkpoint** when the action is irreversible, high-impact, or policy-bearing

This keeps the harness simple where possible and richer only where the gains justify the cost.

Human checkpoints are a **policy layer with multiple insertion points**, not a single mandatory hop. Under the current architecture, a checkpoint may live:
- at the workflow/work-item boundary when the object under review is already a workflow or packet transition
- at the A2A / relay thread boundary when the object under review is a thread epoch or debate continuation
- at the autonomous tool boundary when a sensitive action has no natural workflow/thread home

The harness milestone must unify decision semantics, policy ownership, and observability across these insertion points. It does **not** require every approval to be migrated into one workflow type before implementation can begin.

### 6.6 Runtime selection is policy, not prompt folklore

The platform must decide which runtime classes are available for which work:
- direct specialist run
- orchestrator thread
- OpenClaw subagent
- ACP harness session

ACP harness sessions remain an advanced runtime reserved for explicit orchestrator/admin use cases. They are not a generic escape hatch for specialist bots.

### 6.7 Trace and eval are part of the harness, not afterthoughts

Every harnessed run must emit:
- structured run metadata
- tool timeline
- stop reason
- final artifact pointers
- checkpoint/escalation events

Task classes that matter operationally must also get eval coverage. This mirrors current official practice: production agents require visibility and reproducible evaluation, not only prompt iteration.

## 7. Target Harness Model

### 7.1 Execution lanes

| Lane | Owner | Examples | Model autonomy |
|---|---|---|---|
| Deterministic front door | Platform code | workflow start/cancel, approval transition (non-chat typed ingress) | None at ingress |
| Conversational specialist lane | Specialist bot | answer questions, read workspace/knowledge, bounded domain analysis | Low |
| Worker harness lane | Worker bot | bounded execution against typed task packet | Medium |
| Orchestrator harness lane | Orchestrator bot | delegate, coordinate sessions, manage checkpoints | Medium-high |
| ACP harness lane | Orchestrator/Admin only | advanced external coding/runtime sessions with structured handoff | High but explicitly bounded |

### 7.2 Core harness components

1. **Intent router**
   - selects deterministic front door vs harnessed run
2. **Run contract builder**
   - creates typed run metadata and budgets
3. **Runtime selector**
   - chooses allowed runtime(s) by lane and class
4. **Artifact store**
   - persists plans, task packets, checkpoint summaries, evaluator results
5. **Checkpoint manager**
   - pauses for human approval when policy requires it
6. **Trace/eval layer**
   - records what happened and grades what matters

## 8. Milestone Scope

### Phase H1 — Intent & Lane Control

Goal: reduce the reliance on soft prompt-compliance for high-authority entry paths, to the extent that Agentopia's current ingress surface allows.

H1 is **not a single phase that can remove soft prompt-compliance from all high-authority paths in one move**. The [Deterministic Delivery Start Front Door](../architecture/p0.5-deterministic-delivery-start.md) workstream has already proven that chat-originated deterministic ingress cannot be implemented in the control plane alone: the gateway owns the Telegram connection via the OpenClaw framework as an opaque binary with no intercept point, so deterministic ingress for chat-originated typed intents requires the gateway fork (milestone #25) and was deferred on 2026-03-22. H1 is therefore split into two sub-phases that reflect that constraint honestly.

**Phase H1a — Control-plane classifier and typed-intent registry (control-plane-only).**

Goal: classify every entry path and wire the typed front doors whose ingress already exists outside the chat surface.

Deliver:
- lane taxonomy (deterministic vs harnessed)
- typed front-door registry in `bot-config-api`
- per-path classification report (deterministic-today / should-be-deterministic-but-chat-ingress-pending-fork / autonomous-by-design)
- initial routing rules for existing product flows whose ingress is already HTTP / API / workflow-signal (operator dashboard-triggered workflow cancel, workflow-internal approval transitions, etc.)
- explicit mapping of chat-originated paths that remain soft under the interim dual-lane model, with the reason (gateway-fork dependency)

Definition of done:
- every production entry path is classified
- every typed intent whose ingress is non-chat has an owner and a Temporal-lane implementation or a waiver
- the soft chat-originated paths are documented as H1b-pending, not hidden

**Phase H1b — Gateway-fork ingress for chat-originated typed intents (runtime/gateway lane, fork-dependent).**

Goal: once the gateway fork (milestone #25) lands, realise deterministic ingress for chat-originated typed intents that H1a marked as should-be-deterministic.

Depends on: milestone #25 (gateway fork) closing. Until then, H1b cannot close.

Canonical example: **delivery-start.** It is the paradigmatic case of a chat-originated typed intent whose classifier verdict is "should-be-deterministic" but whose ingress is currently soft, covered by the SOUL-guided `start_delivery` tool under the permanent dual-lane model. Any additional `/command`-style typed intent introduced on a messaging surface joins this queue.

Deliver:
- deterministic Telegram/chat ingress for the set of H1a-classified should-be-deterministic intents
- migration from the interim dual-lane model (for those specific intents) to a hard deterministic ingress
- continued-compatibility behaviour for intents that remain soft

Definition of done:
- delivery-start has a deterministic ingress that does not rely on LLM prompt compliance
- the interim dual-lane model remains the fallback only for intents explicitly waived
- every intent that H1a marked as should-be-deterministic has a deterministic ingress

**H1 admin scope (explicit).** Admin mutation is **not** part of H1. Per the currently approved baseline, admin mutation is implemented as the Admin-class `admin_mutate` tool in the capability-class work (H2.5 gate; audit sink is Phase 2.5). A deterministic-workflow ingress for admin mutation is a possible future ADR topic and is outside this milestone.

### Phase H2 — Run Contract & Artifact Handoff

Goal: make every autonomous run explicit and resumable.

Deliver:
- run contract schema
- artifact schema for plans / packets / checkpoints / outcomes
- structured handoff between sessions and runtimes
- stop-condition contract

Definition of done:
- autonomous runs are created with typed budgets and stop conditions
- long-running work hands off through artifacts, not raw chat dependence

### Phase H3 — Checkpoints, Traces, and Evals

Goal: attach feedback and verification loops to the harness.

Deliver:
- checkpoint policy matrix
- run trace schema and storage (backend selection resolved: [ADR-015](../adrs/015-h3-02-trace-backend-langfuse); production service design drafted in [h3-observability-production-design](../architecture/harness-control/h3-observability-production-design) — phase-α shape specified, production steady-state target specified with open architecture gaps, document still Draft)
- task-class eval plan
- first evaluator-enabled lane where evidence shows real value
- insertion-point mapping for workflow-level, thread-level, and tool-level checkpoints under one policy contract

Definition of done:
- irreversible/high-impact actions have a checkpoint rule
- harnessed runs are inspectable after the fact
- at least one important task class has repeatable eval coverage
- checkpoint semantics are aligned across workflow/work-item approvals, A2A / relay thread checkpoints, and autonomous tool-boundary approvals

### Phase H4 — ACP / Advanced Runtime Hardening

Goal: treat ACP as an explicitly-governed advanced runtime, not a loophole.

Deliver:
- ACP runtime policy by class and lane
- structured spawn/handoff contract
- recovery/cleanup rules
- guidance for when ACP is warranted vs when OpenClaw subagents or deterministic flows are better

Definition of done:
- ACP use is policy-governed
- no specialist bot gets ACP access by accident
- ACP runs emit the same run-contract and trace primitives as other harnessed runs

## 9. Dependencies And Adjacent Work

### Depends on
- [Runtime Facts, Capability Classes, and Runaway Tool Prevention](../architecture/harness-control/runtime-facts-capability-classes-baseline.md)
- [Deterministic Delivery Start Front Door](../architecture/p0.5-deterministic-delivery-start.md)
- [Agent-to-Agent Protocol](../architecture/a2a-full-design.md)

### Adjacent to
- worker pool routing improvement
- multi-step planning with human approval
- advanced multi-agent collaboration
- observability & cost management

This milestone is the control-plane layer that makes those roadmap items coherent instead of incremental feature pile-on.

## 10. What Changes In The Roadmap

This milestone should be tracked as its own planned workstream:

**Agent Harness Control Plane**
- deterministic front doors where business contracts are known
- bounded autonomous runs where flexibility is actually needed
- structured artifacts and handoffs across sessions/runtimes
- checkpoints, traces, and evals as first-class harness components
- ACP as an explicitly governed advanced runtime

## 11. Acceptance Criteria For Planning Completion

Planning for this milestone is complete only when all of the following are documented:

1. inventory of current soft vs deterministic entry paths
2. run-contract schema draft
3. lane/runtime policy matrix
4. artifact taxonomy and ownership
5. checkpoint policy matrix
6. trace/eval minimum contract
7. ACP policy and access boundary

## 12. Final Verdict

This milestone is **approved for planning**.

The platform already has a runtime-safety baseline. What is still missing is the harness that controls when autonomy begins, how it is bounded, and how it is observed. Current official practice across OpenAI and Anthropic is aligned on this direction: simple where possible, autonomous only where needed, repository-local artifacts as durable context, and mechanical guardrails plus feedback loops around every serious run.

Agentopia should implement this as a distinct control-plane milestone rather than letting harness behavior emerge piecemeal from prompts, tool lists, and ad hoc orchestration docs.
