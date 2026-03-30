---
title: "Worker Pool Routing Improvement Plan"
---


**Status**: Improvement plan — not yet implemented end-to-end
**Date**: 2026-03-30
**Type**: Architecture improvement plan
**Scope**: Evolve generic worker dispatch toward capability-aware pool routing

---

## 1. Summary

Agentopia's workflow orchestration can dispatch tasks to workers and reviewers through a role-based system. Today, the primary dispatch path resolves a role (e.g., "worker") to a single bound agent. This works well for small teams with one worker per role, but does not scale to multi-specialist pools where tasks should route to the most appropriate worker based on capability.

This document describes how Agentopia should evolve from generic role-based dispatch toward capability-aware worker pool routing — enabling frontend tasks to reach frontend-capable workers, backend tasks to reach backend-capable workers, and rework to return to the assigned owner.

---

## 2. What Agentopia Supports Today

### Workflow-driven task dispatch

Agentopia orchestrates delivery workflows through a durable workflow engine. When a workflow reaches the development phase, the system dispatches a work packet to a worker agent. The dispatch path resolves a role key (e.g., `worker`) to a bound agent, validates that the agent is active, and delivers the task via A2A protocol or sidecar transport.

### Role-based agent binding

Agents are bound to roles through a persistent registry. Three canonical roles exist:

- **Orchestrator** — decomposes objectives, manages workflow lifecycle, gates releases
- **Worker** — creates branches, writes code, submits pull requests
- **Reviewer** — inspects code, submits reviews, approves or requests changes

Each role carries an explicit authority contract that governs which tools and actions the agent may perform. Role bindings are stored persistently and enforced at dispatch time.

### Work packet model

Work packets carry structured metadata including objective, scope boundaries, acceptance criteria, and dependencies. The primary routing key is `assigned_role` — packets are role-addressed, not actor-addressed. This separation keeps work definitions independent of specific agent identity.

Packets also carry an extensible metadata field that can store routing-relevant context such as repository, owner, and issue references.

### Agent capability metadata

Each agent has metadata including `a2a_role` and `a2a_specialty` (e.g., "backend-api", "frontend"). This metadata is set during agent creation and is available through the bot list API. Today, this metadata is used for display and discovery — it is not yet consumed by the primary dispatch path.

### Foundation for pool routing

An optional actor pool system exists behind a feature flag. When enabled, it provides:

- A pool of routable actors per role, each with capability tags and load state
- A routing algorithm with two strategies: `best_available` (lowest load with skill match) and `sticky` (prefer previous actor for rework continuity)
- Atomic claim management to prevent concurrent dispatch conflicts

This foundation is functional but not yet wired into the main delivery workflow dispatch path end-to-end.

### Reviewer routing

Reviewer dispatch includes filtering by operating mode (workflow-scoped vs. repository-bound), ensuring that workflow reviews route to appropriately configured reviewer agents.

---

## 3. Current Limitation

The primary dispatch path resolves a role to a **single** bound agent. If exactly one worker is bound, dispatch succeeds deterministically. If zero or multiple workers are bound, dispatch fails with an explicit error.

This model has two gaps for teams with multiple specialist workers:

1. **No automatic specialization matching.** If both a frontend worker and a backend worker are bound to the `worker` role, the system cannot automatically select the right one based on the task's nature. The dispatch path does not yet consult work packet content or agent capability metadata to make this decision.

2. **Generic routing is insufficient at scale.** As teams grow beyond a single worker per discipline, the system needs to select from a pool of eligible candidates rather than requiring exactly one binding per role.

The actor pool foundation addresses these gaps architecturally, but the connection between the pool routing logic and the main delivery workflow dispatch is not yet fully integrated.

---

## 4. Why This Matters

### Better specialist matching

When a delivery objective involves frontend changes, routing to a frontend-capable worker produces better results than routing to a generic or backend-focused worker. The same applies across other specializations — infrastructure, data engineering, mobile, and more.

### Less ambiguity in multi-worker teams

Without capability-aware routing, operators must either maintain a single worker per role (limiting team size) or manually direct work to the right agent. Automatic routing reduces this operational burden.

### Continuity across rework

When a reviewer requests changes on a pull request, the rework should return to the worker who originally built the feature. That worker has context on the codebase area, the design decisions, and the reviewer's feedback. Routing rework to a different worker wastes context and increases error rates.

### Clearer scaling model

Capability-aware routing provides a natural scaling path: add more specialist workers to the pool, tag them with capabilities, and the system routes work appropriately. This is more sustainable than manually managing one-to-one role bindings as the team grows.

---

## 5. Design Principles

1. **Explicit capability modeling.** Worker capabilities must be declared as structured data, not inferred from names or prompts. The routing system should match against declared capabilities, not heuristics.

2. **Deterministic routing when required.** For workflows where a specific worker must handle the task (e.g., operator-directed assignment), the system must support explicit actor targeting that bypasses pool selection.

3. **Safe fallback behavior.** If no capable worker is available for a specialized task, the system should fall back to a generic worker (if one exists) or surface a clear routing failure — never silently drop the task or route to an inappropriate agent.

4. **Sticky ownership for rework.** Once a worker is assigned to a work packet, rework iterations on that packet should return to the same worker unless the operator explicitly reassigns. Context continuity is more valuable than load balancing for rework.

5. **Observability of routing decisions.** Operators should be able to see why a specific worker was selected, what alternatives were considered, and what fallback was applied. Routing decisions should be auditable.

6. **Incremental rollout.** Each phase should be deployable independently. Teams using a single worker per role should see no behavior change. Teams with multiple workers should be able to opt into pool routing progressively.

---

## 6. Proposed Evolution

### Phase 1 — Capability-aware packet metadata

Add structured routing hints to work packets during the planning phase.

When the orchestrator decomposes an objective into work packets, each packet should carry a classification signal — for example, `frontend`, `backend`, `infrastructure`, `full-stack`, or `unknown`. This classification should be derived from the objective content and the target repository context, not hard-coded.

The classification is a routing hint, not a hard constraint. It informs the routing layer without over-constraining early.

**Outcome**: Work packets carry enough metadata for the routing layer to make informed decisions.

### Phase 2 — Worker capability profiles

Extend agent metadata to include a structured capability declaration.

Each worker agent should declare its capabilities at creation time — for example, `["frontend", "react", "css"]` or `["backend", "python", "api"]`. Capabilities should be stored alongside existing agent metadata and be queryable by the routing layer.

Multiple workers may share the same capabilities. A single worker may have multiple capabilities (e.g., a full-stack worker with both frontend and backend tags).

**Outcome**: The system has a structured representation of what each worker can do, separate from what role they hold.

### Phase 3 — Eligible-subset routing

Connect packet classification to worker capabilities in the dispatch path.

When dispatching a work packet, the routing layer should:
1. Identify all workers bound to the required role
2. Filter to the subset whose capabilities match the packet's classification
3. Select from the eligible subset using the existing load-balanced or sticky strategy

If no workers match the specific classification, fall back to workers tagged as `general` or `full-stack` before failing.

**Outcome**: Frontend tasks route to frontend-capable workers. Backend tasks route to backend-capable workers. Ambiguous generic routing is reduced.

### Phase 4 — Sticky continuity and rework ownership

Ensure assignment continuity across the lifecycle of a work packet.

Once a worker is assigned to a packet:
- Rework iterations on that packet should return to the same worker
- The assignment should be preserved even if the worker's load increases
- Reassignment should only happen through explicit operator action

The existing `parent_packet_id` linkage and sticky routing strategy provide the foundation for this. The gap is connecting these primitives to the delivery workflow's rework path.

**Outcome**: Rework returns to the assigned worker. Context is preserved. Operators can override when needed.

### Phase 5 — Observability and operator controls

Make routing decisions visible and overridable.

- **Routing trace**: Each dispatch should record the routing decision — which workers were eligible, which was selected, and why.
- **Fallback transparency**: If a fallback was applied (e.g., no frontend worker available, fell back to general), the trace should show this.
- **Operator override**: Allow operators to redirect a work packet to a specific worker before or during execution, bypassing pool selection.
- **Dashboard visibility**: Surface routing decisions in the workflow detail view so operators can understand and audit the dispatch history.

**Outcome**: Operators can understand, audit, and control routing decisions. The system is transparent, not opaque.

---

## 7. What This Plan Does Not Yet Promise

This document is an improvement plan. It describes the direction Agentopia is evolving toward, not capabilities that exist today.

Specifically:
- **Automatic task classification** based on objective content is a planned capability, not a shipped feature. The quality of classification will improve incrementally.
- **Workflow-context-aware specialization** (e.g., routing based on the specific files or modules being changed) is a future refinement beyond the initial phases.
- **Advanced planning decomposition** that produces multiple parallel packets with different specialization requirements is tracked separately and is not a prerequisite for basic pool routing.

The phased approach allows each improvement to deliver value independently while building toward the full vision.

---

## 8. Success Criteria

The routing improvement plan is successful when:

- Frontend-classified tasks route to frontend-capable workers when available
- Backend-classified tasks route to backend-capable workers when available
- Ambiguous tasks fall back to general-capability workers with a visible trace
- Rework iterations return to the originally assigned worker
- Operators can see why a worker was selected for any given task
- Teams with a single worker per role experience no behavior change
- Teams with multiple specialist workers see measurably better task-to-worker matching

---

## 9. Conclusion

Agentopia already has the orchestration foundation — durable workflows, role-based dispatch, authority contracts, agent metadata, and an optional pool routing system. What remains is connecting these primitives into a coherent, end-to-end capability-aware routing path that serves multi-specialist teams.

This plan evolves the system incrementally: first enriching packet metadata, then structuring worker capabilities, then connecting them through eligible-subset routing, then ensuring rework continuity, and finally making the whole process observable and operator-controllable. Each phase delivers standalone value while building toward scalable specialist routing.

---

## Appendix: Current Technical Foundations

The following platform primitives already exist and provide the foundation for this improvement plan:

| Foundation | Status | Role in Routing |
|---|---|---|
| Role-based agent binding | Active | Maps roles to agents; primary dispatch path today |
| Role authority contracts | Active | Defines what each role can do; not routing, but enforcement |
| Agent capability metadata (`a2a_specialty`) | Active (metadata only) | Stored per-agent; not yet consumed by dispatch |
| Work packet extensible metadata | Active | Can carry routing hints; not yet populated by planner |
| Actor pool with load tracking | Feature-flagged | Pool selection with skill match and load balancing |
| Sticky routing strategy | Feature-flagged | Prefers previous actor for rework; requires pool to be active |
| Packet claim management | Feature-flagged | Atomic actor claiming to prevent double-dispatch |
| Activation enforcement | Active | Blocks dispatch to inactive or degraded agents |
| Parent packet linkage | Active | Tracks rework lineage; available for sticky resolution |
