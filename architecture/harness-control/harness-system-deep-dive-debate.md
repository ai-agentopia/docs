---
title: "Harness System — Deep-Dive Debate (Research + Target Architecture)"
---

# Harness System — Deep-Dive Debate (Research + Target Architecture)

**Status:** Research document — input to an architecture review board decision round
**Owner:** Platform Architecture
**Audience:** ARB + engineering leads across `agentopia-core`, `agentopia-protocol`, `agentopia-infra`
**Canonical home:** this document, in the shared docs repo
**Sister documents:**
- [Runtime Facts, Capability Classes, and Runaway Tool Prevention](./runtime-facts-capability-classes-baseline.md)
- [Orchestrated Multi-Agent Platform with Partial Harness Control](./orchestrated-multi-agent-platform-partial-harness-control.md)
- [Agent Harness Control Plane & Bounded Autonomy](../../milestones/agent-harness-control-plane.md)

---

## 1. Executive Summary

Agentopia today is an orchestrated multi-agent platform with partial harness control. Two adjacent baselines have been approved: the *runtime-facts / capability-classes / runaway-prevention* safety model at the tool-surface layer, and the *partial-harness-control* assessment at the platform layer. The next step is neither another safety patch nor a greenfield rewrite; it is a deliberate decision, supported by primary-source research, about what the production harness should actually be, which parts we build because they are domain-specific, and which parts we integrate because mature open-source or primary-vendor options already exist.

The research in §3–§4 shows that the external agent ecosystem has converged on a small set of load-bearing patterns: a deterministic orchestrator with non-deterministic execution units (Temporal), a model-driven tool-use loop with per-subagent isolation and mechanical stop conditions (Claude Agent SDK, OpenAI Agents SDK), OpenTelemetry/OpenInference for traces, and MCP as the context-exchange protocol. No single framework offers a production harness that is both provider-agnostic and opinionated enough to be adopted whole. The strongest OSS and vendor pieces are unambiguous: Temporal for durable orchestration; OTEL+OpenInference+Phoenix/Langfuse for traces and evals; MCP as a tool protocol. The weakest pieces — and the ones Agentopia must own — are the intent router, the run contract, the capability-class enforcement point, the artifact taxonomy, and the checkpoint matrix.

The recommendation in §10 is therefore a hybrid target architecture: keep Temporal as the backbone of the *deterministic front door* (it already is, for delivery); keep LangGraph as a planner-only component inside Temporal activities (it already is); integrate OTEL+OpenInference and a Phoenix or Langfuse backend for the trace/eval layer; do not adopt any agent framework as the primary harness — the primary harness is a thin, provider-agnostic Agentopia-owned layer that materialises run contracts, routes lanes, and enforces capability classes and the R1/R2/R3 rails. §11 describes a four-phase migration that preserves what is already strong and retires soft prompt compliance one lane at a time.

The document ends with a build-vs-buy matrix (§8), an OSS candidate comparison (§9), and a set of explicitly rejected shortcuts (§12). The verdict is `HARNESS_DEEP_DIVE_READY` and the document is suitable as input to the ARB decision that selects the final harness architecture and funds Phase H1.

## 2. Problem Framing

Two separate documents already frame the problem at their own layer. The *runtime-facts* baseline closed the tool-surface class of failure exposed by the `session_status` runaway loop. The *partial-harness* baseline locked the current-state assessment: the platform has several strong control islands (`bot-config-api` as control plane, A2A orchestration with turn limits, delivery workflows on Temporal, `sessions_spawn` and ACP harness, governance-bridge execution-class enforcement, same-turn dedupe), but these islands do not yet add up to one coherent bounded-autonomy harness.

This document answers a different question: given that the next milestone is the Agent Harness Control Plane, what should its architecture actually be — grounded in primary-source research about how production agent systems are built today, and grounded in evidence from the current Agentopia code and docs? Three decisions must come out of the answer: which subsystems Agentopia builds because they are domain-specific, which subsystems it integrates because mature solutions exist, and which shortcuts it explicitly rejects.

## 3. What a Production Harness System Includes Today

Primary-source research across the current frontier agent systems converges on seven load-bearing components, regardless of vendor:

1. **A run loop with a run contract.** The loop has a typed input (goal or message), a typed termination (final output or handoff), and mechanical stop conditions. OpenAI's Agents SDK states the rule explicitly: "If the LLM returns a `final_output`, the loop ends… If we exceed the `max_turns` passed, we raise a `MaxTurnsExceeded` exception" ([OpenAI Agents SDK — running agents](https://openai.github.io/openai-agents-python/running_agents/)). The Claude Agent SDK has the same structure at the per-subagent level with per-agent `maxTurns` ([Claude Agent SDK — subagents](https://code.claude.com/docs/en/agent-sdk/subagents)).
2. **Deterministic orchestrator with non-deterministic execution units.** Temporal's canonical pattern — "The Workflow is the orchestration layer… and must be deterministic… while Activities are where the actual work happens (calling LLMs, invoking tools, making API requests) and can be as unpredictable and non-deterministic as needed" ([Temporal — Durable execution meets AI](https://temporal.io/blog/durable-execution-meets-ai-why-temporal-is-the-perfect-foundation-for-ai)) — is the most durable model in the research set.
3. **Tool-call boundary enforcement.** Per-tool allow/deny lists (OpenAI Guardrails, Claude SDK `allowedTools`/`disallowedTools` per subagent), plus mechanical loop caps. MCP is explicit that it does **not** solve this: "MCP itself cannot enforce these security principles at the protocol level, implementors SHOULD… build robust consent and authorization flows into their applications" ([MCP specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25)).
4. **Sub-agent and delegation primitives.** The frontier split is between *handoff-as-tool-call* (OpenAI: a handoff is represented to the LLM as `transfer_to_X` tool call, [OpenAI — handoffs](https://openai.github.io/openai-agents-python/handoffs/)), *isolated subagent with one-way input* (Claude: "Each subagent runs in its own fresh conversation… only its final message returns to the parent", [Claude — subagents](https://code.claude.com/docs/en/agent-sdk/subagents); note: subagents **cannot** spawn subagents), and *manager-driven delegation* (CrewAI hierarchical process with a dedicated `manager_llm`, [CrewAI hierarchical](https://docs.crewai.com/en/learn/hierarchical-process)).
5. **Observability as part of the run contract, not after.** Traces are emitted by the runtime, not bolted on in production. OpenAI's Agents SDK tracing is on by default with wrappers for `agent_span`, `generation_span`, `function_span`, `handoff_span`, `guardrail_span` ([OpenAI — tracing](https://openai.github.io/openai-agents-python/tracing/)). The OSS side has converged on **OpenInference** as the instrumentation spec ([OpenInference spec](https://arize-ai.github.io/openinference/spec/)) and OpenTelemetry as the wire format.
6. **Structured artifacts for long-running work.** Plans, task packets, checkpoint summaries, evaluator findings, final outcomes — persisted out-of-band from the conversation. Anthropic's harness-design guidance for long-running coding and OpenAI's "AGENTS.md as a map, not an encyclopedia" guidance both make this explicit.
7. **Human-in-the-loop checkpoints at irreversible boundaries.** LangGraph makes this a first-class primitive through interrupt/resume + a checkpointer ([LangGraph — HIL](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)); every other serious harness has it as a pattern rather than a built-in, and usually requires the workflow layer (Temporal signals, LangGraph checkpointer) to implement it durably.

What is *not* a harness component, despite frequent confusion in informal discussion: MCP is a context protocol, not an orchestrator. AGENTS.md is a documentation convention, not a runtime protocol — "AGENTS.md has no JSON schema, no YAML, no special syntax — just Markdown" ([agents.md](https://agents.md/)). A2A (Agent2Agent) is a multi-vendor open protocol for agent↔agent RPC ([A2A specification](https://github.com/a2aproject/A2A/blob/main/docs/specification.md)), not a ratified standard and not an orchestration layer. ACP (Agent Client Protocol) is an editor↔agent protocol from Zed ([Agent Client Protocol](https://agentclientprotocol.com/)); it is complementary to MCP, not a harness.

These distinctions matter because they are where Agentopia's roadmap discussions most often drift — conflating a protocol with a harness produces phantom coverage.

## 4. External Research Findings

Summarised from the primary-source research compiled for this document; individual citations live in §3 and §9. The research findings relevant to the target architecture are:

- **OpenAI Agents SDK** provides a clean reference design for the agent-loop contract and the tracing model. It is Python/TS-centric, OpenAI-first, and not a fit as the primary harness for a multi-provider platform — but the run-loop semantics (final-output rule + max-turns cap) are the pattern Agentopia's harness must also implement. Tracing is OpenAI-dashboard-first and is not a general ingestion endpoint ([OpenAI Agents SDK](https://openai.github.io/openai-agents-python/), [tracing](https://openai.github.io/openai-agents-python/tracing/)).
- **Claude Agent SDK** is the same engine as Claude Code, now exposed as a library. Its strongest contributions to the discourse are (a) the explicit rename from "Claude Code SDK" to "Claude Agent SDK" formalising that this is a library for building harnessed agents, (b) the isolation contract for subagents ("only its final message returns to the parent"), (c) the hard rule that subagents cannot spawn subagents ([Claude Agent SDK — overview](https://code.claude.com/docs/en/agent-sdk/overview), [subagents](https://code.claude.com/docs/en/agent-sdk/subagents)). Agentopia's ACP runtime already implements a similar boundary in spirit; the Claude guidance confirms the shape is correct.
- **AutoGen** is in maintenance mode per its own README, with Microsoft directing new work to Microsoft Agent Framework (MAF) ([microsoft/autogen README](https://github.com/microsoft/autogen)). Any Agentopia plan that references "AutoGen = Microsoft's agent framework" is out of date.
- **LangGraph** is the strongest OSS choice for graph-structured workflow-with-LLM, with first-class checkpointing and HIL ([langgraph README](https://github.com/langchain-ai/langgraph)). Agentopia already uses it as the *planner* inside Temporal activities per the current roadmap entry. It is not a fit as the primary harness outside of that role — promoting it to the platform orchestrator would put durability into LangGraph's checkpointer instead of Temporal's event history, which already owns durability for Agentopia's delivery work.
- **CrewAI** is opinionated and well-documented, with a manager-LLM hierarchical model ([CrewAI hierarchical process](https://docs.crewai.com/en/learn/hierarchical-process)). Not a fit — Agentopia's orchestrator pattern is already richer than CrewAI (A2A threads + ACP + subagents + governance bridge), and adopting CrewAI would be a step backward in capability.
- **OpenHands** is a coding-agent runtime with a documented controller/agent contract. Its V0 `controller/agent.py` is explicitly deprecated (scheduled for removal April 1, 2026); the supported surface is the V1 software-agent-sdk, which separates agent-local logic from remote sandboxed tool execution over WebSocket. Useful as an architectural reference for sandbox boundaries; not a replacement for Agentopia's gateway/runtime split.
- **Goose (Block)** is an MCP-first general-purpose agent ([block/goose](https://github.com/block/goose)). Not a platform fit — Goose is an end-user agent, not a platform runtime for fleets of specialist bots.
- **Temporal** is the single most-aligned piece of infrastructure in the research set. Its Workflow/Activity split is exactly the deterministic-orchestrator / non-deterministic-execution boundary Agentopia's partial-harness baseline describes at §5.2. Temporal's own AI content ([Of course you can build dynamic AI agents with Temporal](https://temporal.io/blog/of-course-you-can-build-dynamic-ai-agents-with-temporal), [Durable execution meets AI](https://temporal.io/blog/durable-execution-meets-ai-why-temporal-is-the-perfect-foundation-for-ai)) is the clearest primary-source articulation of the pattern. Agentopia already uses Temporal for delivery; the remaining work is to formalise Temporal as the deterministic front-door lane for *every* typed intent.
- **MCP** is what it is: a tool/context exchange protocol ([MCP 2025-11-25 spec](https://modelcontextprotocol.io/specification/2025-11-25)). Agentopia already has an MCP bridge extension; continuing to expand MCP surface is good engineering and has nothing to do with the harness problem.
- **OpenTelemetry + OpenInference** are the de facto open instrumentation pair for the agent layer. **Arize Phoenix** (OSS, OTEL-native, OpenInference-based) and **Langfuse** (OSS, OTLP endpoint + LLM observability UI) are the two most credible OSS trace/eval backends; **LangSmith** is a credible commercial option with framework-agnostic OTEL ingest ([blog.langchain.com — end-to-end OpenTelemetry LangSmith](https://blog.langchain.com/end-to-end-opentelemetry-langsmith/)); **Braintrust** is eval-first.
- **Policy-as-config for tool allowlists** does not have a turnkey OSS flagship. The pattern in production is OPA ([Open Policy Agent](https://www.openpolicyagent.org/)) or Cedar ([permitio/cedar-agent](https://github.com/permitio/cedar-agent)) placed at the tool-call boundary; Permit.io packages this for MCP commercially. Agentopia's R1/R2/R3 safety-rail baseline already covers the runtime side; a policy-as-config layer for capability-class → tool-set mapping is the component that might optionally borrow Cedar or OPA, but does not require it.

A surprising state-of-reality finding from the research: despite the apparent crowdedness of the agent-framework space, no single project is a drop-in production harness for a multi-provider, multi-bot, workflow-grounded platform like Agentopia. This is the key reason the §10 recommendation is hybrid rather than adoption.

## 5. Current Agentopia Architecture Inventory

Grounded in the current code and design docs:

**Control plane.** `bot-config-api` owns bot lifecycle, relay configuration, workflows, credentials, governance signalling, and deployment reconcile endpoints. The recent `reconcile-routing` pattern (proxy-routed provider refactor round) is the canonical example of control-plane-driven reconciliation against the Helm chart: the control plane emits policy values, the chart renders them, existing bots are migrated via an explicit endpoint call. Evidence: `agentopia-protocol/bot-config-api/src/services/providers.py`, `services/k8s_service.py::reconcile_routing_values`, the merged `reconcile-routing` endpoint in `routers/deploy.py`.

**Execution plane.** `agentopia-core` runs the per-bot gateway binary, which hosts the LLM run loop, the tool catalog, the plugin hooks (`knowledge-retrieval`, `mem0-api`, `relay`, `governance-bridge`, `wf-bridge`, `mcp-bridge`), the session manager, and the subagent/ACP spawn primitives. Evidence: the before-tool-call hook structure in `src/agents/pi-tools.before-tool-call.ts`, the loop-detection module in `src/agents/tool-loop-detection.ts`, the subagent spawn path in `src/agents/subagent-announce.ts` and `src/agents/acp-spawn.ts`.

**Orchestration runtime — durable side.** Temporal workflows power delivery (`DeliveryWorkflow`, plan-dispatch-develop-review-merge). Per the roadmap: "LangGraph planner decomposes objectives into structured work packets" — LangGraph is used *inside* a Temporal activity, not as a parallel orchestrator. This is already the correct layering.

**Orchestration runtime — ephemeral side.** A2A JSON-RPC threads support debate / bridge / epoch patterns with turn ordering, max-turn control, checkpoints, idempotency, and recovery semantics. Evidence: `architecture/a2a-full-design.md`. A2A is ephemeral (in-memory/Redis-backed thread state), complementary to Temporal's durable workflows.

**Delegated runtimes.** `sessions_spawn` launches OpenClaw subagents with explicit parent/child relationships and lifecycle constraints. ACP harness sessions (`runtime: "acp"`) launch advanced external coding/runtime contexts with structured handoff. Both exist as real primitives — referenced and documented in `agentopia-core/docs/tools/subagents.md` and `docs/tools/acp-agents.md` — but neither is governed by a platform-wide run contract yet.

**Safety rails.** Partial: loop detection code exists but default-off; `genericRepeat` warn-only; `knownPollNoProgress` excludes tools with volatile output. The runtime-facts baseline introduced the R1/R2/R3 contract but implementation is still a pending phase. Same-turn dedupe exists inside `governance-bridge` for specific read-only actions — a harness behaviour encoded in an extension rather than platform-wide.

**Execution-class authorization.** `governance-bridge` propagates a trusted `execution_class` boundary (`general_chat` vs `workflow_dispatch`) and binds action classes to execution classes before tool execution. This is the strongest real production-grade enforcement surface on the platform today and proves the approach scales; it is scoped to governance, not universal.

**Internal-prompt visibility.** The `<agentopia-internal>` sentinel boundary (recent round) gives chat.history authoritative display normalization and removes the prompt-layer leakage class.

**Capability class work.** Approved baseline, not yet implemented. `wfBridgeRoleKey` (orchestrator gate) is the only currently-live precedent.

**Tool protocol.** MCP is integrated through the `mcp-bridge` extension; per-bot MCP credentials are managed by `bot-config-api`. This is a production-grade MCP host.

**Provider routing.** The completed proxy-routed-provider abstraction (round that introduced `routing_mode`, `build_routing_values`, `modelsProviders`/`extraEnvFromSecrets`) is a genuinely elegant control-plane-to-chart pattern. It is the *shape* Phase H1 (intent router / run-contract builder) should mirror: control-plane declares intent, chart renders policy, reconcile endpoint migrates existing state.

## 6. Current Agentopia Harness Maturity Assessment

| Harness capability | Current state | Evidence | Maturity | Notes |
|---|---|---|---|---|
| Run loop with mechanical stop conditions | Partial — maxToolCalls universal 40; no per-tool-per-turn cap | `src/agents/tool-loop-detection.ts` default-off; `pi-tools.before-tool-call.ts` wires the hook | 🟡 L2 | R1 new detector is the Phase-1 delta the runtime-facts baseline locked |
| Deterministic orchestrator with non-det activities | Strong for delivery; weak elsewhere | Temporal `DeliveryWorkflow`; nothing equivalent for workflow cancel, admin mutation, approval transitions | 🟢 L4 in one domain; 🟡 L2 platform-wide | Phase H1's deterministic front doors extend this to every typed intent |
| Tool-call boundary enforcement | Partial — in-runtime per-bot allowlist from chart `tools.allow`; no policy-as-config per role | `charts/agentopia-bot/templates/configmap-config.yaml` line 183 `group:sessions` unconditional | 🟡 L2 | Capability classes + per-class render (baselined) is the Phase-2 delta |
| Subagent / delegation primitives | Strong primitives; not governed by unified contract | `subagent-announce.ts`, `acp-spawn.ts`; docs `tools/subagents.md`, `tools/acp-agents.md` | 🟢 L3 capability; 🟡 L2 governance | Phase H4 hardens ACP into an explicitly-governed lane |
| Run contract per autonomous run | Missing | No typed run metadata carries across A2A / subagent / ACP; no stop-condition contract | 🔴 L0 | Phase H2 delta |
| Structured artifacts for long-running work | Partial — delivery workflow has work packets; everything else uses chat continuity | LangGraph work-packet schema inside delivery; no artifact schema in A2A threads or subagent handoff | 🟡 L2 | Phase H2 delta |
| Human-in-the-loop checkpoints | Partial — workflow governance has controlled transitions; no platform-wide matrix | `execution-authorization.md`; no generic checkpoint primitive | 🟡 L2 | Phase H3 delta |
| Traces | Missing at harness layer — logs exist, OTEL-native tracing does not | gateway logs; no OTEL/OpenInference emission | 🔴 L0 | Phase H3 delta; integrate OTEL+OpenInference |
| Evals | Missing at platform | No task-class eval coverage; no regression test for run outcomes | 🔴 L0 | Phase H3 delta |
| Execution-class authorization | Strong in governance | `governance-bridge/index.ts` `execution_class` propagation | 🟢 L4 in governance | Pattern worth generalising |
| Tool-protocol integration (MCP) | Strong | `mcp-bridge` extension; per-bot credentials; Permit.io-style auth patterns unused | 🟢 L4 | Continue expanding MCP surface |
| Provider routing | Strong (recent refactor) | `build_routing_values`, `modelsProviders`, capability-class precursor | 🟢 L5 reference pattern | Shape to copy for Phase H1 |
| Internal-prompt visibility boundary | Strong (recent round) | `<agentopia-internal>` sentinel + chat.history authoritative display | 🟢 L4 | Done |

Maturity legend: **L0** absent; **L1** ad-hoc; **L2** exists in some subsystems; **L3** exists platform-wide but inconsistent; **L4** unified contract, some gaps; **L5** fully shipped and tested.

The assessment reconfirms the partial-harness-control baseline: the platform is mid-way on the ladder, strong in control-plane patterns and in specific domain enforcement, weak in universal harness contracts. The Phase H1–H4 plan in the milestone doc is correctly scoped against this assessment.

## 7. Gap Analysis

Five gaps separate the current platform from a production bounded-autonomy harness. Each is a specific, concrete delta, not a vague aspiration.

1. **No intent router.** There is no single place where "this user/system intent requires a deterministic front door" vs "this intent enters an autonomous run" is decided. Delivery-start is deterministic because p0.5 typed it that way; almost nothing else is. The router itself is ~300 LoC of Agentopia-owned code and a registry of typed intents.
2. **No universal run contract.** A2A threads have their own turn/checkpoint model; subagents have parent/child lifecycle; ACP has its own spawn contract; direct specialist chats have none. A shared schema (`{run_id, goal_type, runtime, capability_class, wall_clock_budget, tool_budget, stop_conditions, expected_artifact_shape, escalation_policy, trace_id}`) must be the input to every autonomous run and the output of every checkpoint/evaluator step. This is domain-specific; no OSS primitive replaces it.
3. **No trace/eval infrastructure at the harness layer.** The platform has logs, not structured traces. Every L4 harness in the research has OTEL emission as a baseline. Building this from scratch is unnecessary: the OpenInference + OTEL + Phoenix (or Langfuse) stack is mature, OSS, and agnostic. Integration, not building.
4. **No artifact taxonomy.** Delivery has a work-packet shape. Nothing else does. Structured handoff between sessions, subagents, and ACP runs currently relies on chat continuity. This must be defined by Agentopia (domain-specific) and persisted by Agentopia (can use Postgres or the existing workflow state; does not require a dedicated artifact store).
5. **No platform-wide checkpoint policy matrix.** Irreversible/high-impact actions must be declaratively mapped to their checkpoint requirements. This is config-as-data and lives in the control plane. Temporal signals are the right durable primitive for implementing a pause; LangGraph's interrupt-and-resume pattern is a useful reference but not required since Temporal already owns the workflow layer.

Gaps *not* in the list because they are either already strong or out-of-scope: MCP integration (strong; ongoing expansion), provider routing (strong; recently refactored), internal-prompt visibility (shipped), agent-to-agent RPC (A2A is sufficient), tool-surface safety (baselined, Phase-1-pending).

## 8. Build vs Buy vs Integrate Evaluation

| Subsystem | Current Agentopia owner | Candidate external | Recommended decision | Rationale | Migration difficulty | Lock-in / risk |
|---|---|---|---|---|---|---|
| Run loop (per-turn) | `agentopia-core` gateway (pi-embedded-runner + plugin hooks) | OpenAI Agents SDK / Claude Agent SDK | **Keep current. Adopt patterns only.** | Both SDKs are provider-centric; replacing the Agentopia gateway would reduce a multi-provider platform to single-provider. The run-loop *pattern* (final-output rule + max-turns + tool-call boundary) is what we adopt. | N/A | None — pattern adoption |
| Deterministic orchestrator | Temporal (delivery) | Temporal (extended to more lanes) | **Keep and extend.** | Temporal's Workflow/Activity split is the canonical durable pattern and Agentopia already runs production delivery on it. Extending it to cover workflow cancel, admin mutation, approval transitions is incremental. | Low | Existing dependency |
| Graph planner | LangGraph inside Temporal activity | LangGraph | **Keep. Do not promote.** | LangGraph's checkpointer is scoped to its own state; Temporal is the durability authority. Keep LangGraph as a planner component, not as the platform orchestrator. | N/A | Minor — bounded |
| Agent-to-agent protocol | In-house A2A (JSON-RPC) | a2aproject spec / SDKs | **Keep in-house.** | The a2aproject spec is not ratified and offers no operational benefit over the existing in-house protocol, which is already productionised. Re-evaluate in 12 months if standardisation matures. | N/A | None |
| Tool/context protocol | In-house MCP bridge | Continue MCP expansion | **Continue.** | MCP is the correct protocol at this layer; Agentopia already integrates it cleanly. No action beyond ongoing expansion. | N/A | None (open standard) |
| Editor/external-agent protocol | In-house ACP harness | ACP spec (Zed) | **Keep, align to spec.** | ACP is complementary to MCP and ACP harness is already Agentopia-owned; aligning the in-house surface to Zed's ACP spec where feasible removes integration friction. | Medium (one-off) | None (open protocol) |
| Tool-call boundary (per-role allowlist) | Helm chart `tools.allow` → control-plane `capabilityClass` (baselined) | OPA / Cedar-agent / Permit.io | **Build. Defer OPA/Cedar until Phase H3+.** | Capability classes as an enum + control-plane emission is domain-specific and cheap. OPA/Cedar is an optional future extraction if policy-as-rego/cedar ever adds value; today Python + typed enums are sufficient. | Low | Low |
| Runaway-prevention rails (R1/R2/R3) | Agentopia (baselined; Phase 1 pending) | None credible OSS | **Build.** | No OSS project provides per-tool-per-turn caps with turn-id awareness. This is domain-specific and small (~200 LoC + test). | Low | None |
| Sub-agent / delegation primitive | `sessions_spawn`, ACP harness | Claude Agent SDK patterns (reference only) | **Keep in-house. Adopt Claude's one-way input pattern as a contract.** | Claude SDK's "subagents cannot spawn subagents; only final message returns" is the right invariant. Codify it in Agentopia's subagent contract. | None (contract-level) | None |
| Human-in-the-loop checkpoints | Governance transitions (partial) | LangGraph interrupt/resume / Temporal signals | **Build on Temporal signals.** | Temporal already owns durability. LangGraph's pattern is useful as a reference; reimplementing it on Temporal signals is trivial and avoids dual-durability stores. | Medium | None |
| Run contract schema | None | None | **Build.** | Domain-specific; no external answer. TypeScript interface + JSON schema shared across core and control plane. | Low | None |
| Artifact taxonomy | Delivery work-packet (partial) | None | **Build.** | Domain-specific. Postgres table + typed schemas. No external product provides semantic artifact types for a multi-agent platform. | Low–Medium | None |
| Tracing (wire format + emit) | Logs only | **OpenTelemetry + OpenInference** | **Integrate.** | OTEL is the industry wire format; OpenInference is the de facto agent-specific semantic-conventions layer. Instrument in the gateway plugin hook surface; emit via OTLP. Provider-agnostic, portable. | Medium | Very low (open standards) |
| Trace backend / UI | None | **Arize Phoenix** (OSS) or **Langfuse** (OSS) or LangSmith (commercial) | **Integrate Phoenix or Langfuse. ARB choice.** | Both are OTEL-native and OpenInference-compatible. Phoenix leans trace/eval-heavy; Langfuse leans LLM-observability + prompt management. Either is adoptable; pick one based on ops preference. LangSmith is a valid commercial alternative if a managed service is preferred. | Medium | Low (OTEL portability) |
| Eval harness | Ad-hoc | Braintrust (commercial) or Phoenix Evals (OSS) | **Integrate Phoenix Evals first; revisit Braintrust if eval complexity grows.** | Phoenix covers trace + eval in one OSS stack with OpenInference. Braintrust is stronger for dedicated eval workflows but adds a commercial vendor. Start with Phoenix; escalate if evidence demands. | Medium | Low |
| Execution-class authorization | `governance-bridge` | None credible OSS | **Keep. Generalise pattern.** | The `execution_class` propagation pattern is the strongest existing harness enforcement on the platform. Lift it into the generic run contract rather than leaving it extension-local. | Low | None |
| Internal-prompt visibility | `<agentopia-internal>` sentinel (shipped) | None | **Keep. Done.** | Already production-grade. | N/A | None |
| Control-plane reconcile pattern | `reconcile-routing` endpoint + Helm values | None | **Keep. Replicate for capability-class.** | Best existing shape for platform-wide migrations of live bots. The pattern is `reconcile-capability` for Phase H2. | N/A | None |

Summary: **build the domain-specific primitives** (intent router, run contract, artifact taxonomy, capability-class enforcement, R1/R2/R3), **extend what already works** (Temporal, A2A, ACP, MCP, governance `execution_class`, control-plane reconcile), and **integrate the mature OSS stack for the observability layer** (OTEL + OpenInference + Phoenix or Langfuse). Explicitly reject adopting any agent framework as the primary harness.

## 9. Open-Source / Existing Solutions Comparison

| Candidate | What it solves | Fit for Agentopia | Non-fit / limitations | Recommended role |
|---|---|---|---|---|
| OpenAI Agents SDK ([docs](https://openai.github.io/openai-agents-python/)) | Single-agent run loop, handoffs, built-in tracing for OpenAI models | Useful as pattern reference for run-loop and stop-condition shape | Python/TS-first; tracing tied to OpenAI dashboard; handoff-as-tool-call is a single pattern of many | Reference only |
| Claude Agent SDK ([docs](https://code.claude.com/docs/en/agent-sdk/overview)) | Claude Code harness repackaged as a library; tools, hooks, subagents, MCP, permissions | Strong pattern reference for subagent isolation invariants; strong overlap with Agentopia's ACP runtime concept | Anthropic-only; subagents cannot spawn subagents; not a multi-provider platform harness | Reference only |
| AutoGen ([repo](https://github.com/microsoft/autogen)) | Actor-model multi-agent chat + runtime | Maintenance mode; no forward-compat story | Do not adopt as the primary platform harness; Microsoft Agent Framework is the successor | Rejected |
| LangGraph ([repo](https://github.com/langchain-ai/langgraph)) | Stateful graph-based workflow with LLM nodes, checkpointer, HIL | Already used as planner inside Temporal activity in delivery | Promoting LangGraph to platform orchestrator would fragment durability between Temporal and LangGraph checkpointer; not justified | Partial integration (current role) |
| CrewAI ([hierarchical](https://docs.crewai.com/en/learn/hierarchical-process)) | Manager-LLM hierarchical delegation | Weaker than Agentopia's A2A + Orchestrator class model | Too opinionated; would regress existing capability | Rejected |
| OpenHands ([repo](https://github.com/All-Hands-AI/OpenHands)) | Coding-agent runtime + sandboxed tool execution | Architecture reference for V1 sandbox boundary | V0 controller is deprecated; V1 SDK is narrow-scope (coding agent) | Reference only |
| Goose ([repo](https://github.com/block/goose)) | General-purpose local agent with MCP extensions | End-user product, not a platform runtime | Not a platform fit | Rejected |
| Temporal ([docs](https://docs.temporal.io/workflow-definition), [AI blog](https://temporal.io/blog/of-course-you-can-build-dynamic-ai-agents-with-temporal)) | Durable Workflow/Activity orchestration | Already in production for delivery; the canonical durable-orchestrator primitive | None material | **Keep and extend** — deterministic front door for all typed intents |
| Model Context Protocol ([spec](https://modelcontextprotocol.io/specification/2025-11-25)) | Host↔tool/context protocol | Already integrated via `mcp-bridge` | MCP explicitly does not enforce auth, orchestration, policy | **Keep and expand** — not a harness, but a good tool surface |
| AGENTS.md ([site](https://agents.md/)) | Multi-vendor repo-level agent guidance file | Trivially adoptable | No runtime semantics | Adopt as convention |
| ACP ([site](https://agentclientprotocol.com/)) | Editor↔agent protocol (from Zed) | Complements Agentopia's ACP harness; align where feasible | Editor-centric scope | Align in-house surface to spec |
| A2A ([spec](https://github.com/a2aproject/A2A/blob/main/docs/specification.md)) | Multi-vendor agent↔agent RPC protocol | In-house A2A already production | Not ratified; no operational upside to migrating | Keep in-house |
| OpenTelemetry + OpenInference ([spec](https://arize-ai.github.io/openinference/spec/)) | Wire-format + semantic-conventions for agent traces | Industry default for observability | Instrumentation work required at gateway plugin layer | **Integrate — primary tracing** |
| Arize Phoenix ([docs](https://arize.com/docs/phoenix)) | OSS trace + eval backend, OTEL-native, OpenInference-native | Strong fit for trace UI + eval in one product | Operational overhead of self-hosting if that matters | **Integrate — candidate trace/eval backend** |
| Langfuse ([OTEL docs](https://langfuse.com/integrations/native/opentelemetry)) | OSS LLM-observability with OTLP endpoint + prompt management | Strong fit; stronger on the prompt-management side | Trace UI less agent-specific than Phoenix | **Integrate — alternative trace backend; ARB decision** |
| LangSmith ([OTEL](https://docs.langchain.com/langsmith/trace-with-opentelemetry)) | Commercial trace/eval, framework-agnostic, OTEL ingest | Plausible if a commercial vendor is preferred | Vendor lock on UI; OTEL portability mitigates | Commercial alternative only |
| Braintrust ([docs](https://www.braintrust.dev/docs/evaluate)) | Commercial eval-first platform | Adopt later if eval complexity exceeds Phoenix | Commercial vendor; adds cost | Defer |
| OPA ([site](https://www.openpolicyagent.org/)) | Policy-as-config engine | Could host capability-class rules as Rego | Overkill at Phase 1; Python + enum is sufficient | Defer; possible future extraction |
| Cedar / cedar-agent ([repo](https://github.com/permitio/cedar-agent)) | Policy-as-config engine (HTTP) | Same role as OPA; slightly AWS-flavoured | Defer | Defer |
| Permit.io ([AI access control](https://www.permit.io/ai-access-control)) | Commercial AI auth layer (OPA-based) | Would replace Agentopia's in-house capability-class enforcement | Commercial vendor; unnecessary coupling | Rejected for now |

## 10. Recommended Target Architecture for Agentopia

The target architecture is a **hybrid harness**: Agentopia owns the domain-specific control primitives; the durable orchestrator is Temporal; observability is OTEL + OpenInference + Phoenix-or-Langfuse; tool/context is MCP; agent-to-agent is in-house A2A; advanced external runtime is ACP.

### 10.1 Layers

```
┌──────────────────────────────────────────────────────────────────┐
│  Control plane        bot-config-api                              │
│  (declares policy)    • capabilityClass per bot                   │
│                       • intent registry (typed front doors)       │
│                       • run-contract templates                    │
│                       • checkpoint policy matrix                  │
│                       • reconcile endpoints                       │
└──────────────────────────────────────────────────────────────────┘
                              │ renders into
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  Deployment           Helm chart (agentopia-bot)                  │
│  (renders policy)     • tools.allow from capabilityClass          │
│                       • env from routing values                   │
└──────────────────────────────────────────────────────────────────┘
                              │ runs
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  Intent router        Temporal (deterministic front door)         │
│  + durable lane       • workflow start/cancel, admin mutate,      │
│                         approval transitions                      │
│                       • Workflow = deterministic orchestrator     │
│                       • Activity = LLM calls, tool calls, etc.    │
└──────────────────────────────────────────────────────────────────┘
                              │ dispatches to
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  Harnessed runs       agentopia-core gateway                      │
│  (autonomous lane)    • per-turn run loop                         │
│                       • R1 per-tool-per-turn / R2 maxToolCalls /  │
│                         R3 loop detection                         │
│                       • capability-class tool filtering           │
│                       • subagent / ACP spawn with run contract    │
│                       • A2A threads for multi-agent debate        │
│                       • plugins: knowledge, mem0, governance-bridge, │
│                         wf-bridge, mcp-bridge, relay              │
└──────────────────────────────────────────────────────────────────┘
                              │ emits
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  Observability        OpenTelemetry + OpenInference               │
│  (integrated)         • OTLP exporter                             │
│                       • Backend: Phoenix or Langfuse (ARB choice) │
│                       • Trace = run record; spans = tool calls,   │
│                         LLM calls, handoffs, checkpoints          │
└──────────────────────────────────────────────────────────────────┘
                              │ persists
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  Artifact store       Postgres (existing)                         │
│                       • plans, task packets, checkpoint summaries, │
│                         evaluator findings, final outcomes        │
└──────────────────────────────────────────────────────────────────┘
```

### 10.2 What owns what

- **Run contract schema** — Agentopia (new, shared across core + control plane).
- **Intent router + typed front-door registry** — Agentopia control plane.
- **Deterministic lane** — Temporal workflows (extend existing).
- **Autonomous lane** — Agentopia gateway run loop (the thing being hardened).
- **Tool-call boundary** — Agentopia R1/R2/R3 + capability-class filter (baselined; Phase 1 implementation).
- **Sub-agent invariants** — Agentopia; adopt Claude-style "only final message returns; children cannot spawn children".
- **HIL checkpoints** — Temporal signals (extend); control plane owns the policy matrix.
- **Traces + evals** — OTEL + OpenInference + Phoenix/Langfuse (integrate).
- **Tool protocol** — MCP (keep, expand).
- **Agent↔agent** — A2A in-house (keep).
- **External/editor runtime** — ACP harness (keep, align to Zed spec where feasible).

### 10.3 What the target architecture explicitly does not include

- No greenfield agent framework as the primary harness.
- No second durability store (LangGraph checkpointer stays inside Temporal activity scope only).
- No custom trace/eval backend (OTEL + OSS backend is sufficient).
- No commercial agent platform dependency (LangSmith / Braintrust / Permit.io are deferred alternatives, not core).
- No Rego/Cedar at Phase 1 (capability-class enum + reconcile is enough until scale demands more).

## 11. Migration Strategy

Migration is phased, matches the Agent Harness Control Plane milestone (Phases H1–H4), and preserves the separately-scoped containment workstream for the `session_status` incident.

**Phase H0 — Containment (parallel; already scoped).**
Ship the `session_status` containment per the runtime-facts baseline §12 (system-prompt edit + chart `tools.allow` + loopDetection.enabled + maxToolCalls=12). Required before any bot is recreated from scratch. Not a harness phase; listed here for completeness.

**Phase H1 — Intent router + deterministic front doors.**
Build the typed-intent registry in `bot-config-api`. Move `workflow start/cancel`, `admin mutate`, `approval transitions` into Temporal workflows with typed inputs. Classify every production entry path as deterministic or autonomous; waive with explicit reason or convert. No runtime change in `agentopia-core`. Deliverable: registry + routing rules + per-path classification report.

**Phase H2 — Run contract + artifact handoff.**
Publish the run-contract schema (shared TS + Python). Attach it to every autonomous run path: direct specialist chat, A2A thread turn, subagent spawn, ACP spawn. Persist artifacts (plans, task packets, checkpoint summaries, outcomes) in Postgres. Stop conditions become mandatory. Deliverable: schema + persistence + `reconcile-capability` endpoint modelled on `reconcile-routing`.

**Phase H2.5 — Capability class implementation.**
Execute the runtime-facts baseline Phase 2: `capability-classes.ts`, chart `tools.allow` from `.Values.capabilityClass`, `admin_inspect` / `admin_mutate` split (with audit sink as the Phase 2.5 gate), `session_status` removed from agent-callable registration. This sits between H2 and H3 because the run contract references `capabilityClass` as a field.

**Phase H3 — Trace + eval + checkpoints.**
Instrument the gateway with OTEL + OpenInference. Stand up the chosen backend (Phoenix or Langfuse; ARB decision). Emit spans for LLM calls, tool calls, subagent spawn, A2A turns, checkpoint events. Build the checkpoint policy matrix in `bot-config-api` and back it by Temporal signals. First evaluator-enabled lane per Phase H3 milestone acceptance. Deliverable: traced runs visible in the backend, checkpoint matrix live for irreversible actions, one task class with eval coverage.

**Phase H4 — ACP hardening.**
Align the in-house ACP surface with the Zed ACP spec where feasible. Restrict ACP to Orchestrator/Admin classes (already baselined). Ensure ACP runs emit the same run-contract and traces as other harnessed runs. Deliverable: ACP runs instrumented identically to direct specialist runs; class gating enforced.

**Ordering rationale.** H1 before H2 because intent classification is the input to the run-contract. H2.5 between H2 and H3 because the run-contract schema needs `capabilityClass` as a field before traces start referencing it. H3 last among the three core phases because traces should cover the finished contract, not a moving target. H4 after H3 because ACP hardening benefits from being instrumentable.

**Migration compatibility contract.** Existing live bots must survive every phase. The `reconcile-capability` endpoint mirrors `reconcile-routing` semantics: patch `valuesObject` atomically; ArgoCD selfHeal picks up the change; the ConfigMap checksum rolls the pod. No manual ConfigMap patches. No destructive migrations. The explicit `wfBridgeRoleKey` → `capabilityClass` derivation is documented and tested.

## 12. Risks / Rejected Shortcuts

Explicitly rejected:

1. **"Rewrite the gateway on Claude Agent SDK."** Locks the platform to Anthropic. Agentopia's multi-provider routing (`openai-codex`, `anthropic`, `openrouter`, `fireworks`, future providers) is a first-class capability and must not regress. Claude Agent SDK contributes patterns, not the harness.
2. **"Adopt LangGraph as the platform orchestrator."** Would fragment durability between Temporal and LangGraph's checkpointer. LangGraph stays inside Temporal activities as a planner only.
3. **"Replace Temporal with an agent framework."** Temporal's own AI content ([Durable execution meets AI](https://temporal.io/blog/durable-execution-meets-ai-why-temporal-is-the-perfect-foundation-for-ai)) is the clearest articulation of the deterministic-orchestrator pattern. Replacing Temporal is a step backward.
4. **"MCP solves tool security."** The MCP spec explicitly disclaims this: "MCP itself cannot enforce these security principles at the protocol level" ([MCP spec](https://modelcontextprotocol.io/specification/2025-11-25)). Agentopia's R1/R2/R3 + capability-class + audit log is the authoritative policy surface.
5. **"AutoGen is Microsoft's framework."** Out of date. AutoGen is in maintenance mode ([microsoft/autogen README](https://github.com/microsoft/autogen)). Any roadmap that references AutoGen as a go-forward option is stale.
6. **"Build our own trace/eval backend."** OpenInference + OTEL + Phoenix/Langfuse is mature, OSS, and specific to agent workloads. Building a bespoke backend is cost without differentiation.
7. **"Promote A2A to a ratified standard."** It isn't one. A2A is a multi-vendor protocol ([A2A spec](https://github.com/a2aproject/A2A/blob/main/docs/specification.md)). Agentopia's in-house A2A is already production; migration has no operational upside.
8. **"Collapse containment into Phase H1."** Containment and final architecture are separate workstreams per the partial-harness baseline. Blocking containment on H1 delays a user-visible regression fix for weeks.
9. **"Defer capability classes until Phase H4."** Capability-class enforcement is the tool-surface safety layer the runtime-facts baseline already locked. It is H2.5, not a post-H4 follow-up.
10. **"Use OPA/Cedar for capability-class enforcement now."** Premature. An enum + typed emission from `bot-config-api` is sufficient at the current scale and ships faster. Keep OPA/Cedar as a future extraction option if policy expressivity becomes a real constraint.

## 13. Open Questions

1. **Trace backend — Phoenix or Langfuse?** Both are OSS and OTEL-native. Phoenix has a stronger trace/eval UI specifically for agents; Langfuse has a stronger prompt-management side. ARB decision, not a blocker for Phase H1.
2. **Eval tool — Phoenix Evals only, or Braintrust later?** Phoenix first; escalate to Braintrust only if evaluator complexity exceeds OSS capabilities.
3. **ACP alignment depth.** Align to Zed's ACP spec fully (including event shapes), or keep a permissive superset in-house? Phase H4 decision.
4. **Turn-id propagation through A2A and subagent/ACP paths.** Baselined as a Phase 1 gate; the exact shape (flat turn-id vs parent-child-aware) is an implementation detail that should land with R1. Not an architectural open question but worth flagging.
5. **Whether to replicate the `execution_class` pattern from `governance-bridge` into a generic platform field.** Likely yes, as part of the run-contract schema in Phase H2. Open for ARB confirmation.
6. **AGENTS.md adoption scope.** Agentopia should add an `AGENTS.md` at each repo root for human + agent onboarding. Trivial. Non-blocking.

## 14. Final Recommendations

1. **Accept the hybrid architecture in §10.** Build the domain-specific primitives; extend Temporal; integrate OTEL + OpenInference + OSS backend; reject the "adopt a framework" shortcut.
2. **Fund Phase H1 (intent router + typed front doors) as the next architectural investment** after the runtime-facts baseline Phase 1 (R1/R2/R3 rails) lands.
3. **Pick a trace backend (Phoenix vs Langfuse) in a separate ARB follow-up.** Do not block H1 on it.
4. **Codify the run-contract schema in Phase H2** as the authoritative handoff object across specialist, orchestrator, subagent, ACP, and A2A surfaces.
5. **Keep containment on its parallel track.** Containment ships before any bot is recreated. Final architecture ships over the phases.
6. **Adopt AGENTS.md at every repo root** as a cheap documentation-layer win.
7. **Re-evaluate LangSmith / Braintrust / Permit.io in 6 months** only if evidence from Phase H3 shows the OSS stack hits a real limit.
8. **Do not expand LangGraph's role beyond planner-inside-activity.** Revisit only if a concrete multi-node-with-HIL problem emerges that Temporal signals cannot address.
9. **Adopt Claude Agent SDK's subagent invariants** ("only final message returns; children cannot spawn children") as a contract on Agentopia's subagent runtime, not an adoption of the SDK itself.
10. **Keep the `<agentopia-internal>` sentinel, the proxy-routed-provider abstraction, and the `governance-bridge` execution-class pattern** as reference shapes for the rest of the harness work; they are the strongest existing examples of what "right" looks like on this platform.

---

## Verdict

`HARNESS_DEEP_DIVE_READY`

The document is written in the shared docs repo, is standalone, cites primary sources throughout §3–§9 and inline in §12, assesses Agentopia's current architecture with repo evidence in §5–§7, resolves build-vs-buy explicitly in §8, and states a concrete target architecture (§10) and phased migration (§11). The remaining open items in §13 are appropriately scoped to ARB decisions that do not block the next phase of implementation planning.
