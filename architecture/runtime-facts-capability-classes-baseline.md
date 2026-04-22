---
title: "Architecture Baseline: Runtime Facts, Capability Classes, and Runaway Tool Prevention"
---

# Architecture Baseline: Runtime Facts, Capability Classes, and Runaway Tool Prevention

**Status:** Approved implementation baseline
**Owner:** Platform Architecture
**Scope:** `agentopia-core`, `agentopia-infra`, `agentopia-protocol` (bot-config-api)
**Audience:** Implementation owners (engineering)
**Canonical source:** this document. Product repos may link here; they must not duplicate the decisions.

---

## 1. Executive Summary

The agentopia bot platform must guarantee that a benign user turn cannot be consumed by a single-tool runaway loop, that agents never call a tool to answer questions the platform already knows before inference, and that a bot's tool surface is determined by its intent rather than by a uniform allowlist.

This baseline establishes four hard rules. First, self-knowledge facts about the agent — model, provider, agent identity, channel, capabilities, and the current date/time — are injected as ambient context into every turn and are the only canonical path for specialist bots. Second, the `session_status` tool is removed from agent-callable registration; the admin surface is re-expressed as two tools, `admin_inspect` (read-only) and `admin_mutate` (explicit write actions under a closed enum). Third, the bot tool surface is governed by a strict, cumulative capability-class ladder with Conversant as the minimum default and Admin as the privileged maximum. Fourth, three independent runaway-prevention rails — R1 per-tool-per-turn cap, R2 per-class `maxToolCalls` budget, and R3 loop detection on by default — are mandatory, with R1 as the generic guard that does not depend on tool output stability or prompt compliance.

Policy ownership is centralised: `agentopia-core` owns the capability-class enum, tool schemas, ambient-fact injection, and the safety rails; `bot-config-api` owns the per-bot `capabilityClass` declaration; the Helm chart renders policy from values without a hand-written allowlist. A separate containment workstream fixes the current user-visible regression without waiting for the full migration.

The document is standalone and sufficient to hand to implementation owners.

## 2. Problem Statement

Specialist workflow bots are deployed as Telegram-fronted agents shaped by their system prompt, their tool catalog, and the model they route through (currently `openai-codex/gpt-5.4` via `agentopia-llm-proxy`). On production traffic, a benign Vietnamese greeting that asked the bot about its current model produced a run in which the bot never replied: it called the `session_status` tool forty times, hit the universal `maxToolCalls = 40` limit, and aborted. The user saw a perpetual "Thinking…".

The failure is not an instance bug. Every bot in the fleet shares the same tool catalog, the same system prompt instructions, and the same 40-call budget. Any future model or provider that weighs tool calls more aggressively than the current Claude baseline will trigger the same class of failure, often on a different tool. This baseline therefore solves a class, not an instance.

## 3. Proven Failure Chain

```
user turn → system prompt inlines:
              · Runtime: agent=… model=openai-codex/gpt-5.4 …
              · "For questions about your current model, provider, runtime
                 config… use session_status first when needed."
            ↑ self-contradiction: the same fact appears as ambient context
              AND as an instruction to call a tool. Frontier models resolve
              the ambiguity toward the tool call.
→ session_status returns a volatile status card (timestamp + usage + queue depth)
→ before_tool_call hook:
    – tool-loop detection defaults to disabled
    – even when enabled:
        · generic-repeat detector is warn-only by design
        · known-poll detector does not include session_status
        · the no-progress circuit breaker requires ≥30 consecutive identical
          result hashes; session_status results change each call, so the
          streak never accumulates
→ the model re-plans; every fresh status card looks like new information;
  the model issues another call
→ 40 calls × ~1s each consumed; maxToolCalls aborts the run
→ no assistant message is produced; the user sees a spinner
```

Five enforcement layers (system prompt · tool catalog · chart allowlist · loop detector · `maxToolCalls`) let the request through. Only the last — the most expensive — actually stops it.

Evidence (verified at the time of authoring):

- `agentopia-core/src/agents/system-prompt.ts`: runtime info is emitted ambiently (`buildRuntimeLine`, lines 695–738) *and* the prompt instructs the model to call `session_status` for the same questions (lines 65–77).
- `agentopia-core/src/agents/tools/session-status-tool.ts`: the tool persists a model override via `updateSessionStore(…)` when a `model` argument is present (lines 271–302).
- `agentopia-core/src/agents/tool-loop-detection.ts`: `enabled: false` by default (line 32); generic-repeat is warn-only (line 473); no-progress streak requires stable result hashes (lines 232–259).
- `agentopia-infra/charts/agentopia-bot/templates/configmap-config.yaml`: `group:sessions` is allowed for every bot unconditionally (line 183).

## 4. Final Architecture Decisions

1. **Agents never call tools to answer questions whose answers are knowable before inference begins.** Model, provider, agent identity, channel, capabilities, and the current date/time are in this class. They are injected as ambient context.
2. **`session_status` is removed from agent-callable tool registration.** Its status-card formatter is retained as an internal gateway function and used only by the operator `/status` UI path. Its mutation responsibility is moved out of the tool entirely.
3. **The admin surface is two tools.** `admin_inspect` is read-only and side-effect-free. `admin_mutate` is explicit write actions under a closed enum, audited, and capped at one call per turn.
4. **A strict cumulative capability-class ladder governs the bot tool surface.** Conversant ⊂ Worker ⊂ Orchestrator ⊂ Admin. Class is declared by the control plane per bot; the Helm chart renders `tools.allow` from class; there are no hand-written allowlists.
5. **Three runaway-prevention rails are mandatory.** R1 (per-tool-per-turn hard cap) is the primary generic guard and does not depend on tool output stability or prompt compliance. R2 (per-class `maxToolCalls` budget) is the worst-case cost bound. R3 (loop detection on by default) is the secondary pattern-detection guard.
6. **Cross-session tools are Orchestrator-class or higher.** `sessions_send`, `sessions_list`, `sessions_history`, `sessions_spawn`, `subagents`, the `relay_*` suite, and the `a2a_*` suite are not Conversant or Worker defaults.
7. **Policy ownership is centralised.** `agentopia-core` owns classes, schemas, ambient-fact injection, and rails. `bot-config-api` owns the per-bot `capabilityClass` declaration. The Helm chart renders policy from values; it does not define policy.

## 5. Canonical Runtime-Facts Model

Runtime facts are classified by volatility and by consumer class. The classification is authoritative for implementation.

| Fact | Volatility | Conversant / Worker / Orchestrator | Admin |
|---|---|---|---|
| Agent identity, bot name, workspace path | Stable per deploy | Ambient (`buildRuntimeLine`) | Ambient |
| Primary model, provider, default model | Stable per session | Ambient | Ambient |
| Current channel, channel capabilities | Stable per session | Ambient | Ambient |
| Current date / time / timezone | Volatile (advances) | **Ambient, per-turn injected (§6)** | Ambient, per-turn injected |
| Available tools, capability class | Stable per deploy | Ambient (system prompt enumerates) | Ambient |
| Session usage quota, provider cache state, cooldown | Volatile | Not exposed | `admin_inspect` |
| Session store contents beyond current session | Stable until mutated | Not exposed | `admin_inspect` |
| Model override, cache invalidation, session mutation | N/A (write) | Not exposed | `admin_mutate` |

The rule is: if a fact is knowable before inference, it is ambient; if a fact is deep runtime state that an operator bot legitimately needs programmatically, it is `admin_inspect`; mutations are `admin_mutate`. Specialists, Workers, and Orchestrators have no tool path to any runtime fact.

## 6. Canonical Time/Date Path

Time and date are unique among runtime facts in that they advance between turns. The canonical path for specialists is ambient per-turn injection into the system prompt at the moment the turn is assembled.

`buildTimeSection` in `agentopia-core/src/agents/system-prompt.ts` is extended from its current "timezone only" form to:

```
## Current Date & Time
Now (UTC):              2026-04-22T03:48:12Z
Now (user timezone):    Tuesday 22 April 2026 10:48 (Asia/Ho_Chi_Minh)
Time zone:              Asia/Ho_Chi_Minh
```

Justification:

1. The semantics match the user's expectation: "what time is it?" means the time at which the user asked. The ambient-at-turn-build time is exactly that reference.
2. No tool exists for the question; no model can loop on what is not callable.
3. The design is model-agnostic: it makes no assumption about any provider's tool-call priors.
4. Imprecision is bounded by product expectation: a reply that lands twenty seconds after turn build reports the turn-build time. That matches how users already reason about chatbot timing.
5. The code path already exists; this is an extension of `buildTimeSection`, not a new primitive.

No specialist-facing `current_time` or `runtime_info` tool is introduced. If a future operational workflow needs second-precision runtime clock access (timers, schedules), that capability is delivered through `admin_inspect` and governed by the Admin class; it does not become a specialist tool.

If a bot has no declared user timezone, the user-timezone line is omitted and the UTC line remains authoritative. No fallback tool is introduced.

## 7. Tool Surface Redesign

**Removed from agent-callable registration.**
`session_status`. Its card formatter is retained as an internal function in the gateway and is called only by the operator `/status` UI path. No capability class is granted agent-level access to this tool.

**Introduced: `admin_inspect`.**
Read-only runtime / session introspection. Signature: `{ target: "session" | "provider" | "account" | "cache" | "quota", filter?: string }`. Returns JSON. The tool is side-effect-free: it never writes session state, never persists overrides, and never invalidates caches.

Several of its targets are volatile by nature (quota counters, usage statistics, provider cooldown state). **Safety does not depend on output stability.** Primary protection is R1 (default three calls per turn for this tool) and R2 (Admin class budget). An optional per-tool `fingerprint: "args-only"` flag is available for targets whose output is genuinely invariant (e.g., `target: "provider"`); this optional mode feeds R3's existing no-progress breaker but is not required for safety.

**Introduced: `admin_mutate`.**
Explicit privileged write actions under a closed enum. Signature: `{ action: ActionEnum, …action-specific fields }`. The enum is exhaustive:

```
"set_session_model"               — { sessionKey, providerModelRef }
"reset_session_model"             — { sessionKey }
"clear_provider_usage_cache"      — { provider, accountId? }
"invalidate_session_memory_cache" — { sessionKey }
```

The handler validates `action` against the enum at the hook layer before dispatch; free-form strings never reach execution. Every call is written to an append-only audit log owned by the gateway (not by the agent) with `{actor, action, argsHash, before, after, outcome, timestamp}`. R1 cap for this tool is **one call per turn**. A new mutation shape is added by introducing a new enum value with its own audit event, never by increasing the cap.

**Unchanged.**
`sessions_send`, `sessions_list`, `sessions_history`, `sessions_spawn`, `subagents`, `relay_*`, `a2a_*`, `wf_command`, `start_delivery`, `gov_*` write tools, `knowledge_retrieve`, `mem_*` read tools. Their schemas are unchanged; only the capability class that may reach them changes (§8).

**New sub-group.**
`group:fs-read` is introduced as a subset of `group:fs` containing only `read`, `list_dir`, `find`, `grep`. The upstream OpenClaw tool catalog is untouched; the sub-group is fork-local.

## 8. Capability-Class Model

Classes are strictly cumulative. Each row lists the tools the class adds on top of the preceding class.

| Class | Intent | Default Tool Surface (additive) |
|---|---|---|
| **Conversant** | Answer user questions using workspace read, retrieval, and memory. No outbound communication beyond the originating session. | `group:fs-read`; `knowledge_retrieve`; `mem_recall` and other read-only memory tools. |
| **Worker** | Conversant + act within a declared domain role (QA, BE, SA). Read-only domain surface only. | `gov_list_*`, `gov_get_*`, `gov_search_code`, `gov_get_file_contents`; `wf_status`. |
| **Orchestrator** | Worker + coordinate work across sessions and agents; write to shared state. | `group:fs` write tools (`write`, `edit`, `move`, `delete`, sandboxed to the bot workspace); `sessions_send`, `sessions_list`, `sessions_history`, `sessions_spawn`, `subagents`; `relay_*` suite; `a2a_*` suite; `wf_command`, `start_delivery`; `gov_*` write surface (`gov_create_*`, `gov_update_*`, `gov_create_branch`, `gov_create_or_update_file`, `gov_push_files`, `gov_create_pull_request`, `gov_add_issue_comment`, `gov_close_issue`, `gov_merge_pull_request`, `gov_create_pr_review`, `gov_get_pr_*`); `mcp_*` when `mcpBridge` is enabled for that bot. |
| **Admin** | Operate the platform itself. | `admin_inspect`, `admin_mutate`. |

The **minimum default for any new bot is Conversant.** A bot acquires a higher class only when its declared role demonstrably requires it. Admin is opt-in by explicit control-plane annotation; no bot auto-promotes.

**R2 `maxToolCalls` budgets:**

| Class | `maxToolCalls` per turn |
|---|---|
| Conversant | 8 |
| Worker | 12 |
| Orchestrator | 40 |
| Admin | 40 |

The Conversant and Worker defaults are initial values to be reviewed after two weeks of `tool_calls_per_turn` p95 telemetry from production. The Orchestrator value retains the prior universal budget.

Additive capability bundles on top of the class ladder are explicitly deferred. If a future orthogonal capability emerges, it is introduced through a dedicated ADR with a specific motivating case.

## 9. Ownership Boundaries

| Concern | Owner | Notes |
|---|---|---|
| Ambient self-knowledge (`Runtime:` line, `Current Date & Time` section) | `agentopia-core/src/agents/system-prompt.ts` | A stability test asserts that no runtime/meta fact requires a tool call when ambient content is present. |
| Tool schemas, profile metadata, `group:fs-read` sub-group | `agentopia-core/src/agents/tool-catalog.ts` | Fork-local; upstream catalog untouched. |
| Capability-class enum, class → tool-set mapping | `agentopia-core/src/agents/capability-classes.ts` (new) | Source of truth for the class ladder. |
| `admin_inspect` and `admin_mutate` implementations and schemas | `agentopia-core/src/agents/tools/admin.ts` (new) | `admin_mutate` action enum is exhaustive and tested. |
| Audit-log sink for `admin_mutate` | Gateway-owned append-only log | Not disableable by the bot. |
| Per-bot `capabilityClass` declaration | `bot-config-api` (extending `providers.py`) | Emitted into Helm `valuesObject.capabilityClass`. |
| Chart renders `tools.allow` from `.Values.capabilityClass` | `agentopia-infra/charts/agentopia-bot` | Helper template; no `hasPrefix` on class; no hand-written list. |
| Per-class `maxToolCalls` | `agentopia-core` defaults + `bot-config-api` emits into SOUL `agents.defaults.maxToolCalls` | |
| R1 per-tool-per-turn cap | `agentopia-core/src/agents/tool-loop-detection.ts` + `pi-tools.before-tool-call.ts` | |
| R3 default-on wiring | `agentopia-infra` Helm chart values (`loopDetection.enabled: true`) | |
| UI `/status` rendering | Gateway chat-surface handler | Calls the internal card formatter; does not call an agent tool. |

**Not owners of capability policy:** the UI, the Helm chart, and the system prompt text. Each of these renders policy; none defines it.

## 10. Runaway-Prevention Contract

Three independent rails. Each is mandatory. **No claim of safety depends on any single rail being perfect or on the LLM complying with a prompt hint.**

**R1 — Per-tool-per-turn hard cap (primary generic guard).**
A new detector in `tool-loop-detection.ts`. For each user turn, the detector counts identical-args calls of the same tool using `{turn_id, tool_name, argsHash}`. On reaching the per-tool cap, `before_tool_call` returns `{ blocked: true, reason: … }` and the tool handler does not execute. The cap operates purely on tool name + args hash + turn identifier. It does not depend on tool output stability, on model-specific priors, or on prompt compliance. Default caps: three for most tools, one for `admin_mutate`. Individual tools may declare a lower cap in their schema.

R1 is the primary guard because it works for every tool, regardless of output volatility or future provider behaviour. Its absence is what allowed the production incident.

**R2 — Per-class `maxToolCalls` budget.**
Each capability class declares a total tool-call budget per turn (§8). When the budget is exhausted, the embedded runner aborts the run with a capability-appropriate error. R2 is the worst-case cost bound: even if R1 failed for reasons not anticipated, R2 caps damage.

**R3 — Loop detection default-on.**
The existing `tool-loop-detection` subsystem is enabled by default via chart values. Its `genericRepeat`, `knownPollNoProgress`, and `pingPong` detectors operate as a secondary net. The `globalCircuitBreakerThreshold` behaviour is retained for tools with stable outputs.

**Prompt hints are not a rail.** Instructions such as "do not poll this tool" or "you already know the answer" may appear in prompt text where they improve UX, but they are never counted as safety. No future tool-addition review may assert prompt text as justification for omitting R1 or R2 coverage.

## 11. Migration Plan

**Phase 1 — Safety rails.**
- Implement R1 in `tool-loop-detection.ts` (new `maxCallsPerToolPerTurn` detector) and wire into `pi-tools.before-tool-call.ts`.
- Wire per-class `maxToolCalls` through `agents.defaults` and chart values.
- Enable R3 by default in chart values (`loopDetection.enabled: true`).
- **Turn-id plumbing gate (§13):** R1 requires a well-defined turn identifier in `SessionState`. If the existing plumbing is insufficient for nested-turn paths (subagent, A2A), the plumbing is added and tested in Phase 1 before R1 lands.
- Adversarial test: iterate every tool in `tool-catalog.ts`; call each fifty times with identical args in one turn; assert `before_tool_call` blocks at the configured cap.

**Phase 2 — Capability model and tool surface.**
- Introduce `capability-classes.ts` with the enum and class → tool-set mapping.
- Declare `capabilityClass` as a control-plane field in `bot-config-api`. Derive initial values for existing bots from their current `wfBridgeRoleKey`: orchestrator → Orchestrator; worker/reviewer → Worker; unset → Conversant. Admin is opt-in by explicit annotation.
- Chart renders `tools.allow` from `.Values.capabilityClass` via a helper template.
- Add `group:fs-read` sub-group.
- Implement `admin_inspect` and `admin_mutate` per §7.
- Remove `session_status` from agent-callable profiles. Retain its internal formatter for `/status`.
- Add `reconcile-capability` API endpoint on `bot-config-api`, modelled on the existing `reconcile-routing` endpoint. It recomputes a bot's `capabilityClass`-derived Helm values and patches the Application.
- Emit a per-bot diagnostic during reconcile listing tools that were dropped, for operator review.

**Phase 2.5 — `admin_mutate` audit sink.**
Must land before `admin_mutate` is callable in any environment. Append-only gateway-owned log; retention and signing policy delivered in a follow-up ADR. Phase 2 does not close until Phase 2.5 lands.

**Phase 3 — System prompt surgery.**
- Delete the "use `session_status` first" block and supporting mentions from `system-prompt.ts`.
- Extend `buildTimeSection` to emit the canonical time/date lines defined in §6.
- Add a stability test: given `Runtime:` and `Current Date & Time` are ambient, a canned turn "what model are you using and what time is it?" must produce a reply with zero tool calls across both Claude and `openai-codex/gpt-5.4`.

**Phase 4 — Cleanup and telemetry.**
- Remove `wfBridgeRoleKey` from code after `capabilityClass` migration has been stable for two release cycles.
- Add `tool_calls_per_turn{capability_class}` histogram metric. Alert on Conversant p95 > 5, Worker p95 > 8.
- Revisit Conversant and Worker budgets with two weeks of telemetry.

## 12. Immediate Containment vs Final Architecture

Containment and the final architecture are parallel workstreams, not sequential phases.

| | Containment | Final Architecture |
|---|---|---|
| Trigger | Observed user-visible "Thinking…" on benign greeting | Any tool-loop class, present or future |
| Scope | System-prompt edit (remove "use `session_status` first"); chart `tools.allow` drops `session_status`; `loopDetection.enabled: true`; `maxToolCalls: 12` for existing bots | §4–§11 |
| Repos touched | `agentopia-core` (system-prompt text), `agentopia-infra` (chart values) | All three |
| Build required | **Runtime / gateway rebuild is required for the system-prompt half of containment.** The chart half rolls via ArgoCD without rebuild. | Standard CI per repo |
| Reversibility | Additive; revertible | Reversible per phase |
| Acceptance | The original greeting replies in under three seconds; a five-turn smoke across three bots shows zero `session_status` calls | CI adversarial test passes; live Conversant p95 tool-calls-per-turn ≤ 3 |

**Containment must ship before any bot is recreated from scratch.** Bot recreation with the current tool catalog would reproduce the loop on first use.

## 13. Implementation Gates

The following gates apply to Phase 1 and must be explicitly cleared before their respective deliverables land.

1. **Turn-id plumbing gate (Phase 1).** R1 depends on a well-defined turn identifier in `SessionState`. The subagent and A2A paths introduce nested turn semantics. Before R1 is merged, the turn-id propagation is verified through these paths; any gap is closed in the same Phase 1 work. This gate is a blocker, not a loose future note.
2. **Containment build gate.** The system-prompt half of containment modifies `agentopia-core`. A runtime image rebuild and a gateway image rebuild on top of it are required for that half to take effect in production. The chart half of containment rolls out via ArgoCD without a rebuild. This document states the requirement so it is not discovered during rollout.
3. **Phase 2.5 gate.** The `admin_mutate` audit sink (append-only, gateway-owned) must be live before `admin_mutate` is callable in any environment. Phase 2 does not close until this gate passes.
4. **Adversarial-loop CI gate (Phase 1).** A CI test iterates every tool in `tool-catalog.ts`, calls each fifty times with identical args in one turn, and asserts `before_tool_call` blocks at the per-tool cap. This test must pass before Phase 1 is declared complete.
5. **System-prompt stability test (Phase 3).** A canned turn "what model are you using and what time is it?" against ambient context must produce a reply with zero tool calls across both Claude and `openai-codex/gpt-5.4`. This test must pass before Phase 3 is declared complete.

## 14. Follow-up ADRs

The following items are acknowledged and will be resolved in dedicated ADRs after the baseline is accepted. None of them blocks Phase 1 or Phase 2 planning.

| # | Title | Blocking baseline? |
|---|---|---|
| ADR-1 | Worker class comment authority (`gov_add_issue_comment` placement) | No |
| ADR-2 | Approval / multi-factor flow for high-impact `admin_mutate` actions; audit-log retention and signing policy | No for baseline; closes before or during Phase 2.5 |
| ADR-3 | Lifecycle policy for `admin_inspect` if Admin population remains zero after six months | No |
| ADR-4 | Additive capability bundles on top of the class ladder | No, deferred |
| ADR-5 | Cross-repo consumer sweep for `session_status` before Phase 2 removes it from agent-callable profiles | Operational prerequisite for Phase 2, not an architecture blocker |

## 15. Final Baseline Verdict

This document is the approved implementation baseline. It is standalone and complete: the canonical runtime-facts model, the canonical time/date path, the tool surface redesign, the capability-class ladder, the ownership boundaries, the runaway-prevention contract, the migration plan, the separately-scoped containment workstream, and the implementation gates are all defined without unresolved core ambiguity. Follow-up ADRs are enumerated and are non-blocking for Phase 1 or Phase 2 planning.

Engineering owners proceed on this basis.
