---
title: "Semantic Memory System"
description: "Agentopia's built-in memory layer that lets agents learn, remember, and personalize conversations over time."
---

# Semantic Memory System

Agentopia includes a semantic memory layer that gives every agent the ability to remember facts, preferences, and context across conversations. Memories are extracted automatically from conversations and recalled when relevant, making each interaction more personalized.

---

## How It Works

The memory system operates in two phases during every conversation:

1. **Recall** -- Before the agent responds, relevant memories are retrieved by semantic similarity to the user's message and injected as context. The agent sees what it has learned about this user without any manual prompting.

2. **Capture** -- After the agent finishes a response, the conversation is analyzed by an LLM to extract factual memories (e.g., "User prefers dark mode", "User's project uses Python 3.12"). These are stored as vector embeddings and entity relationships.

This creates a feedback loop: every conversation enriches the agent's understanding, and every future conversation benefits from that understanding.

---

## Storage Architecture

Memories are stored in two complementary backends:

- **Vector store** -- Embeddings of extracted facts, enabling fast semantic search (e.g., searching "UI preferences" retrieves "Prefers dark mode" even though the words differ).
- **Graph store** -- Entity relationships between people, projects, tools, and concepts, enabling structured knowledge queries.

Both stores are queried together during recall to provide the most relevant context.

---

## Memory API

The memory service exposes a simple REST API used by the gateway plugin:

| Operation | Description |
|-----------|-------------|
| **Add memories** | Submit a conversation; the system extracts and stores facts automatically |
| **Search memories** | Query by natural language; returns ranked results by semantic similarity |
| **List memories** | Retrieve all stored memories for a given user or scope |
| **Delete memory** | Remove a specific memory by ID |

---

## Gateway Integration

The memory system integrates with the Agentopia gateway as a plugin with configurable behavior:

| Setting | Description |
|---------|-------------|
| **Auto-capture** | Automatically extract and store memories after each conversation turn |
| **Auto-recall** | Automatically inject relevant memories before each agent response |
| **Recall timeout** | Maximum time to wait for memory retrieval, ensuring the agent is never blocked |
| **Top-K results** | Maximum number of memories injected per turn |
| **Similarity threshold** | Minimum relevance score for a memory to be included |
| **Capture on failure** | Whether to capture memories even from incomplete or failed conversations |

---

## Memory Scoping

Memories can be scoped to control what each agent remembers:

| Scope | Use case |
|-------|----------|
| **Per-bot** | Each agent has its own isolated memory -- nothing is shared |
| **Per-scope** | A group of agents share memory within a defined scope |
| **Global** | All agents share a single memory pool |

This allows flexible architectures: a customer support bot can have private memory per user, while a team of research agents can share findings across the group.

---

## Key Design Decisions

- **No blocking** -- Memory recall has a strict timeout. If the memory service is slow or unreachable, the agent proceeds without memories rather than hanging.
- **LLM-powered extraction** -- Facts are extracted by an LLM, not by keyword matching. This means the system captures semantic meaning, not just surface-level text.
- **Hybrid retrieval** -- Combining vector similarity with graph relationships produces more accurate recall than either approach alone.
- **Stateless service** -- The memory API server is stateless and horizontally scalable. All persistence lives in the vector and graph stores.
