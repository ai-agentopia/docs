---
title: "ADR-010: SA Knowledge Base — Runtime Retrieval Architecture"
status: accepted
date: 2026-03-30
decision-makers: CTO
debate: "D3 (#296)"
milestone: "#33 — SA Knowledge Base"
depends-on: "ADR-008 (D1), ADR-009 (D2)"
---

# ADR-010: SA Knowledge Base — Runtime Retrieval Architecture

## Context

D1 (ADR-008) established client-scoped knowledge with subscription-based access. D2 (ADR-009) established authenticated, subscription-enforced governance. The critical architecture gap remains: knowledge system and gateway inference path are completely disconnected. A bot with ingested domain documents today cannot use them during inference.

D3 must decide HOW knowledge reaches the SA bot at inference time.

### Current system reality (code-verified)

- **Gateway plugin hooks**: `before_agent_start` receives `{prompt, messages?}`, returns `{prependContext?}`. Priority ordering via `api.on(hook, handler, {priority: N})`.
- **mem0 plugin pattern**: Recall via POST to mem0-api → inject XML context `<relevant-memories>` before LLM → capture after response. Timeout: 3500-8000ms. Non-blocking.
- **Knowledge search API**: `GET /api/v1/knowledge/search?query=Q&scopes=[...]&limit=5`. Multi-scope sequential fan-out. Returns `SearchResult{text, score, scope, citation}`.
- **Latency profile**: Embedding (~200-500ms) + Qdrant (~50-200ms) per scope. Multi-scope (3): ~750-2100ms typical.
- **Bot identity at runtime**: `AGENTOPIA_BOT_NAME` env var available in gateway pod. `AGENTOPIA_RELAY_TOKEN` (per-bot unique, 64-char hex) for service auth. `AGENTOPIA_GATEWAY_TOKEN` is shared (NOT suitable for identity binding).

## Decision

### 1. Retrieval Mechanism

**Gateway plugin using `before_agent_start` hook** — auto-inject knowledge context on every inference turn.

- New plugin: `knowledge-retrieval` registered in gateway extensions
- Uses `before_agent_start` hook (same pattern as mem0 recall)
- Searches bot's subscribed knowledge scopes and injects results as prepended context
- Non-blocking: timeout → skip injection, log warning, continue

Tool-based retrieval (bot decides when to search) rejected for first release — the production requirement is that SA bots MUST reliably use client knowledge. Auto-inject guarantees this. Tool-based search is the natural upgrade path for a future release.

### 2. Scope Resolution

**Server-side resolution** — bot-config-api resolves scopes from bot registration.

- Plugin sends `bot_name` + `query` to bot-config-api (no scopes parameter)
- bot-config-api looks up bot's `client_id` + `knowledge_scopes` from registration
- bot-config-api builds Qdrant collection names: `kb-{client_id}-{scope_name}`
- bot-config-api searches only registered scopes, returns results
- Bot cannot request scopes outside its registration

Search endpoint behavior:
- **Operator** (session auth): specifies `scopes` explicitly — can search any scope
- **Bot** (bearer + X-Bot-Name): scopes resolved from registration — parameter ignored

### 3. Precedence Order

| Priority | Source | Hook Priority |
|---|---|---|
| 1 | System prompt (SOUL.md) | N/A (built-in) |
| 2 | Domain knowledge (knowledge plugin) | `priority: 10` |
| 3 | Conversation memories (mem0 plugin) | `priority: 20` |
| 4 | Conversation history | N/A (built-in) |
| 5 | User message | N/A (built-in) |

Knowledge before mem0 because: domain knowledge is authoritative (client documents with citations), mem0 is conversational fact extraction (no citations, less authoritative).

### 4. Token Budget

| Parameter | Value |
|---|---|
| `topK` (max results) | 5 |
| `maxContextTokens` | 3000 |
| Chunk size | 512 tokens (existing) |
| Budget enforcement | Drop lowest-scoring chunks if total exceeds budget |
| Token estimation | chars / 4 (approximate, first release) |

Total plugin overhead per turn: ~4000 tokens (knowledge + mem0). Against 200K context: <2%.

### 5. Timeout and Fallback

| Parameter | Value |
|---|---|
| `retrieveTimeoutMs` | 5000 |
| On timeout | Skip injection, log warning, continue |
| On error | Skip injection, log warning, continue |
| On empty results | No injection, no error |

**Non-negotiable**: Knowledge retrieval never blocks bot response.

### 6. Injection Format

```
<domain-knowledge>
The following knowledge from client documents may be relevant.
Cite sources when using this information.

[1] (source: api-reference.md, section: Authentication)
API authentication uses OAuth2 bearer tokens with 1-hour expiry...

[2] (source: coding-standards.md, section: Error Handling)
All API errors must return structured JSON with error_code and message fields...
</domain-knowledge>
```

- XML tags: consistent with mem0's `<relevant-memories>` pattern
- Numbered citations: LLM can reference `[1]`, `[2]` in responses
- Source + section inline: attribution without separate lookup
- Instruction line: explicit directive to cite sources

### 7. Bot Identity Propagation

- **Mechanism**: Per-bot `AGENTOPIA_RELAY_TOKEN` as bearer + `X-Bot-Name` header
- **Token**: Unique per bot — cryptographically random 64-char hex (`secrets.token_hex(32)`), stored in per-bot K8s Secret `agentopia-gateway-env-{bot_name}`
- **Auth flow**: `Authorization: Bearer {AGENTOPIA_RELAY_TOKEN}` + `X-Bot-Name: {bot_name}`
- **Validation**: bot-config-api reads `X-Bot-Name`, fetches expected relay token from K8s Secret `agentopia-gateway-env-{bot_name}`, validates with constant-time comparison (`secrets.compare_digest`). Match proves identity.
- **Why relay token, not gateway token**: `AGENTOPIA_GATEWAY_TOKEN` is shared across all bots (same Vault value). `AGENTOPIA_RELAY_TOKEN` is per-bot unique. A shared token + caller-supplied header does not bind identity — any service with the shared token could claim any bot. Per-bot token binds identity to the specific bot's K8s Secret.
- **Proven pattern**: Identical to `relay.py:_authenticate_caller()` — caller provides bot_name + bearer token, API does forward lookup + `secrets.compare_digest`.

### 8. Plugin Config Shape

```typescript
type KnowledgeRetrievalConfig = {
  apiUrl: string;              // bot-config-api URL
  autoRetrieve: boolean;       // Enable auto-injection (default: true)
  retrieveTimeoutMs: number;   // Timeout (default: 5000)
  topK: number;                // Max results (default: 5)
  confidenceThreshold: number; // Min score (default: 0.3)
  maxContextTokens: number;    // Token budget (default: 3000)
};
```

Bot identity sourced from environment variables (not plugin config):
- `AGENTOPIA_BOT_NAME` → `X-Bot-Name` header
- `AGENTOPIA_RELAY_TOKEN` → `Authorization: Bearer` header

## Alternatives Considered

| Decision | Chosen | Rejected | Reason |
|---|---|---|---|
| Mechanism | Plugin auto-inject | Tool-based (LLM decides) | Unreliable — LLM might skip search; production requires guaranteed knowledge use |
| Mechanism | Plugin auto-inject | Hybrid (plugin + tool) | Over-engineering for first release; tool is the upgrade path |
| Scope resolution | Server-side | Plugin config (Helm scopes) | Config drift risk; bot could request unauthorized scopes; server is single source of truth |
| Bot identity | Per-bot relay token + X-Bot-Name | Shared gateway token + header | Shared token doesn't bind identity — any service could claim any bot |
| Bot identity | Per-bot relay token + X-Bot-Name | JWT token claim | No JWT infra; relay token is already per-bot unique and proven |
| Precedence | Knowledge before mem0 | mem0 before knowledge | Knowledge is authoritative (client docs with citations) |
| Timeout | 5000ms | 3500ms (mem0 default) | Knowledge path includes embedding + Qdrant; needs more headroom |

## Consequences

### D2 Provisional Items Resolved

| Item | Resolution |
|---|---|
| Exact enforcement point | bot-config-api search endpoint — server-side scope resolution from registration |
| Query-level audit shape | `knowledge_search: bot={bot_name} client={client_id} scopes=[...] results={count} latency={ms}` |
| Bot identity propagation | Per-bot `AGENTOPIA_RELAY_TOKEN` bearer + `X-Bot-Name` header. Token is unique per bot (K8s Secret). Validated via forward lookup + `secrets.compare_digest` (same pattern as relay auth). |

### Downstream constraints

- **D4 (#297)**: Citation format is defined — `[N] (source: file, section: heading)`. D4 must define provenance rules that work within this injection format.
- **#302**: Must implement the knowledge-retrieval gateway plugin following this architecture. Plugin config via Helm. Server-side scope resolution. XML injection format.
- **#304**: Citation implementation must match the `[N] (source, section)` injection format. SearchResult.citation fields are the data source.
- **#306**: Quality evaluation must measure retrieval quality within this runtime path — auto-inject relevance, citation accuracy, token budget efficiency.

### What D3 does NOT decide

- Document versioning / supersession (D4)
- What happens when sources conflict (D4)
- Ingestion paths and lifecycle (D5)
- External vector import (D6)
- Quality evaluation framework and answer contract (D7)
