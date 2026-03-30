---
title: "Agent-to-Agent Protocol"
description: "How Agentopia agents discover each other, communicate through structured threads, and collaborate on multi-turn tasks with human oversight."
---

# Agent-to-Agent Protocol

Agentopia's Agent-to-Agent (A2A) protocol enables autonomous agents to collaborate through structured, multi-turn conversations. Agents can debate topics, share knowledge, and produce joint outcomes — all under human oversight.

---

## How Agents Communicate

In Agentopia, each agent is an independent unit with its own identity, personality, and skills. Agents communicate through a central orchestrator that manages threads, enforces turn order, and handles authentication between participants.

Communication flows fall into three categories:

- **Direct chat** — a human talks to one agent through a messaging interface. This is unrelated to A2A.
- **Threaded collaboration** — multiple agents take turns contributing to a shared topic. The orchestrator coordinates the conversation.
- **Bridge notifications** — short, one-way messages between agents (status updates, alerts, handoffs).

The protocol ensures that agents participating in a thread respond only with their own perspective. An agent cannot invoke other agents during its turn — the orchestrator handles all routing.

---

## Agent Discovery

Every agent publishes an **Agent Card** — a lightweight descriptor that advertises its identity, capabilities, and available skills.

```json
{
  "name": "Aria",
  "description": "Creative writing specialist",
  "version": "1.0",
  "capabilities": { "streaming": true },
  "skills": [
    { "id": "debate", "name": "Structured Debate" },
    { "id": "bridge-respond", "name": "Bridge Response" },
    { "id": "summarize", "name": "Epoch Summary" }
  ]
}
```

Agent Cards are served at a well-known path and generated automatically from each agent's configuration. Other agents and external systems use these cards to understand what an agent can do before initiating communication.

---

## Thread Model

A **thread** is a structured, multi-turn conversation between two or more agents on a defined topic. Threads have a lifecycle managed by a state machine:

```
  create
    │
    ▼
┌─────────┐
│ ACTIVE  │ ── turn (agent responds) ──► ACTIVE
└────┬────┘ ── auto-checkpoint after N turns
     │
     │ explicit checkpoint
     ▼
┌─────────────────┐
│ PENDING_APPROVAL│  (human review required)
└────────┬────────┘
         │ approve        │ reject       │ approve with guidance
         ▼                ▼              ▼
    ┌────────┐      ┌──────────┐   ┌────────────────┐
    │ ACTIVE │      │ REJECTED │   │ ACTIVE         │
    └────┬───┘      └──────────┘   │ (guidance      │
         │                         │  injected)     │
         │ conclude or max turns   └────────────────┘
         ▼
    ┌────────┐
    │ CLOSED │  (summary persisted, participants notified)
    └────────┘
```

Key behaviors:

- **Auto-conclude** — when a thread reaches its maximum turn count, the orchestrator automatically generates a summary and closes the thread.
- **Epoch summaries** — after every N turns, the conversation history is compacted into a summary. This keeps each turn's context small and focused.
- **Write-once turns** — each turn is recorded as an immutable entry. Status updates happen at the thread level, never by modifying past turns.

---

## Turn Protocol

Each turn is a synchronous request from the orchestrator to one agent. The agent receives:

1. **Epoch summaries** — condensed history from earlier phases of the conversation.
2. **Thread constraints** — the topic, current turn number, and instructions to respond independently.
3. **Recent turns** — the last few raw exchanges for immediate conversational context.
4. **The current prompt** — the message the agent must respond to.

This incremental context strategy means each turn carries only the information the agent needs, rather than rebuilding the full conversation from scratch.

### Session Isolation

Every turn uses a unique session key. This prevents collisions when multiple turns run in sequence or when retries occur. Session keys rotate at epoch boundaries to keep context fresh.

---

## Collaboration Patterns

### Human-Initiated Collaboration

A human sends a request to a coordinating agent, which then orchestrates a thread among specialist agents:

1. Human asks the coordinator to brainstorm a topic.
2. Coordinator creates a thread with selected participants.
3. Coordinator sends turns to each specialist, building on previous responses.
4. At a checkpoint, the coordinator summarizes progress and presents it to the human.
5. Human approves, rejects, or provides guidance.
6. Thread resumes or concludes based on human input.

### Autonomous Orchestration

A coordinating agent can manage the full conversation flow independently — selecting which agent to address, building on responses, and deciding when to pause for human review.

### Direct Relay

For simple, one-off messages between agents, the protocol supports fire-and-forget delivery. The sender does not wait for a response. This is useful for notifications and handoffs.

---

## Human-in-the-Loop

Checkpoints are the primary mechanism for human oversight. They can be triggered explicitly by the coordinating agent or automatically after a configured number of turns.

When a checkpoint fires:

1. The thread pauses and enters a pending approval state.
2. The orchestrator generates a summary of the conversation so far.
3. The summary is delivered to the human.
4. The human can approve (resume), reject (archive), or approve with guidance (inject new direction).

Final decisions — deployments, architecture choices, irreversible actions — must always pass through a checkpoint.

---

## Concurrency and Reliability

### Limits

| Dimension | Behavior |
|-----------|----------|
| Concurrent threads per agent | Bounded to prevent context saturation |
| Maximum turns per thread | Configurable cap to prevent runaway conversations |
| Concurrent turns to one agent | Serialized — additional calls are queued |
| Thread timeout | Idle threads auto-checkpoint after a configured duration |

### Fault Tolerance

- **Queue fallback** — if an agent is unreachable, the message is written to a persistent queue for later delivery.
- **Crash recovery** — on restart, the orchestrator scans for non-closed threads and resumes from the last completed turn.
- **Circuit breaker** — after repeated failures to reach an agent, the orchestrator marks it unavailable and notifies the coordinator.
- **Turn deadlines** — each turn has a timeout. On expiry, the system retries once before falling back to the queue.
- **Idempotency** — completed turns are never re-sent. The orchestrator checks turn status before dispatching.

---

## Context Compaction

Conversations grow over time. To keep each turn efficient, the protocol uses rolling summaries:

1. After every N turns, a summarizer condenses the conversation into key points.
2. The summary is stored as an epoch record.
3. The session rotates to a fresh context window.
4. Subsequent turns receive epoch summaries plus only the most recent raw exchanges.

This keeps per-turn context small while preserving the full arc of the conversation.

---

## Memory Persistence

Conversation knowledge is persisted to long-term memory at three points:

- **After each turn** — partial context is captured incrementally.
- **At checkpoints** — epoch summaries are stored for all participants.
- **At thread close** — a final summary is persisted for all participants.

Memory persistence is non-blocking. A failure to store does not affect the thread itself. This ensures that insights from agent collaboration are available in future conversations, even across different threads.

---

## A2A and Tool Integration

The A2A protocol is complementary to each agent's tool capabilities. An agent can use its own tools (memory, file access, web search) independently during its turn. A2A governs how agents talk to each other; tools govern how agents interact with external systems.

```
              A2A (agent ↔ agent)
Agent A ◄──────────────────────────► Agent B
  │                                    │
  │  Tools (agent → services)          │  Tools (agent → services)
  ├──► Memory                          ├──► Memory
  ├──► File storage                    ├──► File storage
  └──► Web search                      └──► Web search
```

Both layers operate independently and do not interfere with each other.
