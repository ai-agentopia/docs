---
title: "ADR-008: SA Knowledge Base — Product Boundary, Tenancy, and Isolation"
status: accepted
date: 2026-03-30
decision-makers: CTO
debate: "D1 (#294)"
milestone: "#33 — SA Knowledge Base"
---

# ADR-008: SA Knowledge Base — Product Boundary, Tenancy, and Isolation

## Context

Production SA bots need access to client-provided domain knowledge (architecture specs, API docs, project standards). The knowledge system exists in code (KnowledgeService: ingestion, Qdrant vector search, scope isolation) but has no authentication, no client/project entity model, and is completely disconnected from gateway inference.

Before building production features, we must decide: what is this capability, who owns the data, and what are the isolation guarantees?

### Current system reality (code-verified)

- **Scope model**: Flat strings (`scope: str`), auto-created on first ingest, no hierarchy
- **Qdrant isolation**: Collection-per-scope (1:1), query isolation proven
- **Auth**: Zero — all knowledge routes are public, no `require_auth`
- **Bot identity**: `bot_name` only — no project, client, or org concept in DB
- **Memory scopes**: Proven N:M pattern — bots subscribe to multiple scopes via `memory_scopes`
- **Gateway plugin**: `before_agent_start` hook proven via mem0 plugin — injects context before LLM inference
- **Deployment**: Single k3s cluster, single namespace, one shared Qdrant instance

## Decision

### 1. Product Boundary

**Knowledge is a client/project-scoped service consumed by SA bots** — not a per-bot feature, not a platform-wide layer.

- Knowledge is organized by client, not by bot
- SA bots subscribe to knowledge scopes within their assigned client (same pattern as `memory_scopes`)
- One document upload serves all SA bots working that client's project
- Operator manages knowledge per client/scope, not per bot
- SA bots are the only first-release consumer

### 2. Tenancy Model

**Hybrid: shared deployment with logical isolation for first release; physical isolation as upgrade path.**

- First release runs on the existing shared cluster with logical isolation at scope + auth level
- Qdrant collection-per-scope provides storage and query isolation
- Auth enforcement (D2) will add the missing access control layer
- Dedicated deployment (separate cluster/namespace per client) is an operational upgrade path — not required for first release but not architecturally blocked
- Scope naming contract is designed to be portable across deployment models

### 3. Isolation Hierarchy

**Two-level: client -> knowledge scope.**

- `client_id` is the hard isolation boundary — the non-negotiable wall
- `scope_name` is the knowledge partition within a client
- Cross-client access is blocked by design at auth layer
- Within a client, bots can subscribe to multiple scopes
- "Project" as a formal third level is deferred — can be introduced later between client and scope without breaking the contract

### 4. Sharing Model

Client boundary is the hard outer wall. Within a client, access is **subscription/authorization-based** — a bot only accesses scopes it has been explicitly assigned, not all scopes in the client.

| Scenario | Decision |
|---|---|
| Same bot, multiple scopes | **Allowed** — bot accesses only its explicitly subscribed knowledge scopes |
| Multiple bots, same scope | **Allowed** — multiple bots can subscribe to the same scope within a client |
| Unsubscribed scopes within same client | **Not accessible** — subscription is required, not implicit. A bot does not gain access to a client scope it was not assigned. |
| Across clients | **Blocked** — hard enforcement, non-negotiable |

The exact authorization model for scope subscription (who can assign, how it is validated, whether it is operator-managed or config-driven) is owned by **D2 (Governance)**.

### 5. Scope Identity Contract

Knowledge scope identity carries two dimensions:

| Dimension | Purpose | Example |
|---|---|---|
| `client_id` | Hard isolation boundary | `acme` |
| `scope_name` | Knowledge partition within client | `api-docs`, `domain-standards` |

- **Composite identity**: `{client_id}/{scope_name}`
- **Qdrant collection**: `kb-{client_id}-{scope_name}` (prefixed to avoid collision with memory collections)
- **Extensible**: A future `project_id` level can be inserted as `{client_id}/{project_id}/{scope_name}` without breaking first-release format

## Alternatives Considered

### Product boundary alternatives

| Option | Verdict | Reason |
|---|---|---|
| Per-bot feature | Rejected | No sharing; client re-uploads docs per bot; contradicts operator workflow |
| Platform-wide layer | Rejected | Over-engineering; SA bots are the only first-release consumer |

### Tenancy alternatives

| Option | Verdict | Reason |
|---|---|---|
| Dedicated-per-client | Rejected for first release | Delays delivery; requires infra per client; no current multi-cluster setup |
| Shared-only (no upgrade path) | Rejected | Blocks future high-security clients who require physical separation |

### Isolation alternatives

| Option | Verdict | Reason |
|---|---|---|
| Bot-only | Rejected | No sharing; contradicts product boundary decision |
| Flat strings (current) | Rejected for production | No enforcement of client boundaries; typo/misconfig causes data leakage |
| Three-level (client/project/scope) | Deferred | Over-engineering for first release; can be added later |

## Consequences

### Downstream constraints

- **D2 (Governance)**: Must define how `client_id` is assigned to bots, how scope subscriptions are authorized (who assigns, how validated), and how auth layer enforces client-scope-bot authorization. Cross-client access attempts must be audited. Within-client access is subscription-based, not blanket — D2 owns the authorization model.
- **D3 (Runtime Retrieval)**: Bot's subscribed knowledge scopes use `{client_id}/{scope_name}` format. Plugin resolves scopes from bot's client assignment.
- **#301 (Scoped model impl)**: Scope model must carry `client_id`. Collection naming uses `kb-{client_id}-{scope_name}`. Bot-to-scope subscription is client-bounded.
- **#305 (Operator workflow)**: UX organized by client then scope, not by bot. Upload flow: select client -> select/create scope -> upload docs.

### What is explicitly NOT decided

- How `client_id` is stored/managed in the system (D2)
- Runtime retrieval mechanism — plugin vs tool vs hybrid (D3)
- Document versioning and supersession model (D4)
- Ingestion paths and lifecycle (D5)
- External vector import scope (D6)
- Quality evaluation framework (D7)

### Open questions resolved

- "What is the client deployment model?" -> Shared with logical isolation for first release, physical isolation as upgrade path
- "Client deployment model decision" external dependency is partially resolved — first release does not require dedicated deployment

### Open questions remaining

- How is `client_id` represented in the database? (New table? Field on existing tables?)
- Is `client_id` an internal identifier or does it map to an external system?
- How does the operator authenticate and prove client membership?
