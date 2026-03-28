---
title: "Agentopia A2A: Thread Protocol & Cost Strategy"
---

# Agentopia A2A: Thread Protocol & Cost Strategy

> Merged reference doc covering the full A2A implementation in Agentopia:
> thread-based multi-turn bot communication, token architecture, cost optimization,
> A2A SDK integration, and operational runbook.
>
> **Status**: All 4 phases implemented (✅ IMPLEMENTED).

---

## 1. Problem Statement

### Root Cause

Every interaction — direct chat, thread debate turn, bridge notification, epoch summary — goes through gateway, which reloads the full bootstrap context unconditionally:

```
User (Telegram) → Gateway → Load bootstrap (50K+ tokens) → LLM → Response
Thread turn     → Gateway → Load bootstrap (50K+ tokens) → LLM → Response
Bridge notify   → Gateway → Load bootstrap (50K+ tokens) → LLM → Response
Epoch summary   → Gateway → Load bootstrap (50K+ tokens) → LLM → Response
```

Three compounding factors:
1. **Stateless API**: `/v1/chat/completions` has no session persistence — every request starts from zero.
2. **Architectural coupling**: All flows route through the same gateway path, unconditionally loading SOUL.md, USER.md, IDENTITY.md, BOOTSTRAP.md, AGENTS.md, TOOLS.md, mem0 context, transcript history (~50K tokens total).
3. **No task abstraction**: Thread turns are modeled as independent chat completions rather than stateful multi-turn tasks.

And additionally: `/hooks/agent` is always **async** — gateway returns HTTP 200 immediately; agent processes in background with no response body. This makes synchronous bot-to-bot relay via hooks fundamentally impossible.

### Cost Impact

- 1 debate: 6 turns × 3 bots = 18 requests × 50K = ~900K tokens wasted on bootstrap
- 2 debates/day + chat = **~$1.50–2.50/day** (mostly bootstrap overhead)
- Thread turns only need ~2–3K tokens of actual context

---

## 2. Solution Architecture

### What Changes

| Flow | Before | After |
|------|--------|-------|
| Direct chat (user → bot) | Gateway → LLM (primary model, 50K) | **No change** — gateway handles |
| Thread/debate turn | Gateway → LLM (primary model, 50K) | **bot-config-api → LLM direct (cheap model, 2–3K)** |
| Bridge notification | Gateway → LLM (primary model, 50K) | **bot-config-api → LLM direct (cheap model, 500)** |
| Epoch summary | Gateway → LLM (primary model, 50K) | **bot-config-api → LLM direct (cheap model, 2K)** |

### What Stays the Same

- Gateway binary untouched (upstream, no fork)
- Telegram polling via gateway
- mem0 integration unchanged
- File memory unchanged (local-path PVC)
- ArgoCD GitOps unchanged

### Synchronous Relay Solution

Openclaw gateway exposes `/v1/chat/completions` — an OpenAI-compatible endpoint that is **fully synchronous**:
- Waits for agent to complete its turn
- Returns agent's generated message in the HTTP response body
- Supports streaming and blocking modes
- Enabled via `gateway.http.endpoints.chatCompletions.enabled: true`

**Authentication:** `Authorization: Bearer <gateway.auth.token>`
**Agent targeting:** `model: "openclaw:<agentId>"`
**Session isolation:** `user` field — each unique value creates an isolated context in the gateway

---

## 3. Token Architecture

### Current State (✅ IMPLEMENTED)

From `charts/agentopia-bot/templates/configmap-config.yaml`:

```json
"gateway": {
  "auth": { "token": "${AGENTOPIA_RELAY_TOKEN}" }
},
"hooks": {
  "token": "${AGENTOPIA_GATEWAY_TOKEN}"
}
```

| Token | Source | Runtime value |
|-------|--------|---------------|
| `AGENTOPIA_GATEWAY_TOKEN` | K8s Secret `agentopia-gateway-env-{bot}` | **Vault overrides at pod startup** — serves as hooks ingress protection token |
| `AGENTOPIA_RELAY_TOKEN` | K8s Secret `agentopia-gateway-env-{bot}` | NOT Vault-overridden — K8s-stable, readable by bot-config-api via `get_relay_token()` |

### Rationale

`/v1/chat/completions` requires `Bearer <gateway.auth.token>`. By setting `gateway.auth.token = ${AGENTOPIA_RELAY_TOKEN}` (K8s-stable), bot-config-api can read the token from K8s Secret and authenticate directly — no Vault access needed.

**Constraint (verified from source):** `hooks.token` MUST NOT equal `gateway.auth.token`. Gateway refuses to start with `Error: hooks.token must not match gateway auth token`. This forces the two tokens to be distinct.

**Security boundary:** RELAY_TOKEN is per-bot and only held by bot-config-api. A bot knows its own RELAY_TOKEN but cannot use it to call other bots' gateways. Acceptable for private cluster threat model.

Result:
- `gateway.auth.token` = `${AGENTOPIA_RELAY_TOKEN}` — K8s-stable, known to bot-config-api
- `hooks.token` = `${AGENTOPIA_GATEWAY_TOKEN}` — Vault-controlled, distinct value at runtime
- bot-config-api calls `/v1/chat/completions` with `Bearer <get_relay_token(to_bot)>` — no new Secret needed
- `/hooks/agent` is no longer used by bot-config-api; hooks remain available but protected by Vault-controlled GATEWAY_TOKEN

---

## 4. Architecture Overview

```
┌────────────────────────────────────────────────────────────┐
│                   Human (Telegram)                          │
│                   SA Bot (trigger)                          │
└────────────────────┬───────────────────────────────────────┘
                     │ Start brainstorm / assign topic
                     ▼
┌────────────────────────────────────────────────────────────┐
│                  bot-config-api                             │
│             (Thread Coordinator + A2A Orchestrator)         │
│                                                             │
│  POST /api/v1/threads               → create thread         │
│  POST /api/v1/threads/{id}/turn     → send one turn        │
│  GET  /api/v1/threads/{id}          → read thread state    │
│  POST /api/v1/threads/{id}/checkpoint → pause for human    │
│  POST /api/v1/threads/{id}/approve    → human approves     │
│  POST /api/v1/threads/{id}/conclude   → summarise + archive│
└────────┬──────────────────────────────┬─────────────────────┘
         │                              │
         │ /v1/chat/completions         │ /v1/chat/completions
         │ Bearer RELAY_TOKEN(bot-b)    │ Bearer RELAY_TOKEN(bot-c)
         ▼                              ▼
┌────────────────────┐        ┌────────────────────┐
│  agentopia-bot-b   │        │  agentopia-bot-c   │
│  (Infra specialist)│        │  (Security expert) │
│  port 18789        │        │  port 18789        │
└────────────────────┘        └────────────────────┘

SA Bot (bot-a) calls relay plugin tools → relay plugin POSTs to bot-config-api Thread API
SA Bot gateway (bot-a) also reachable at agentopia-bot-a:18789 for summarizer calls

NOTE: debate_turn / bridge_notify / epoch_summary LLM calls go via agentopia-llm-proxy
      NOT through gateway — direct call: bot-config-api → llm-proxy:18789 → Codex gpt-5.1

                     ▼
┌────────────────────────────────────────────────────────────┐
│          Shared Memory (local-path PVC, per bot-config-api pod)    │
│  /openclaw-state/shared-memory/threads/{thread_id}/         │
│    ├── meta.json              (state + lease lock)          │
│    ├── turn-0001.json         (write-once, status+response) │
│    ├── turn-0002.json                                       │
│    ├── summary-epoch-1.json   (rolling summary after N turns│
│    └── summary-final.json     (written at CLOSE)           │
│                                                             │
│  /openclaw-state/shared-memory/inbox/{bot_name}/           │
│    └── {timestamp}-{uuid}.json  (async fallback queue)     │
└────────────────────────────────────────────────────────────┘
                     │
                     ▼ at checkpoint + at close (non-blocking)
┌────────────────────────────────────────────────────────────┐
│              mem0-api (per-bot memory)                      │
│  POST http://mem0-api.agentopia.svc.cluster.local:8000/v1/memories
│  userId: bot-{bot_name}   (isolated) or scope-{scope} (shared)
│  Summary searchable via autoRecall on next conversation     │
└────────────────────────────────────────────────────────────┘
```

---

## 5. Auth Flow End-to-End

```
┌──────────────┐  Bearer RELAY_TOKEN(bot-a)   ┌──────────────────┐
│  relay plugin│ ──────────────────────────►  │  bot-config-api  │
│  (SA Bot)    │                              │  Thread API      │
└──────────────┘                              │                  │
                                              │  validates:      │
                                              │  from_bot token  │
                                              │  == RELAY_TOKEN  │
                                              └────────┬─────────┘
                                                       │ Bearer RELAY_TOKEN(bot-b)
                                                       ▼
                                              ┌──────────────────┐
                                              │  agentopia-bot-b │
                                              │  /v1/chat/compl. │
                                              └──────────────────┘
```

Each hop:
1. relay plugin → bot-config-api: `Bearer ${AGENTOPIA_RELAY_TOKEN}` (calling bot's own token)
2. bot-config-api → target gateway: `Bearer get_relay_token(to_bot)` (target bot's token, read from K8s Secret)
3. Thread API validates step 1 via `_authenticate_caller(from_bot, bearer)` — same pattern as relay router

---

## 6. Thread State Machine

```
         create
            │
            ▼
       ┌─────────┐
       │ ACTIVE  │ ──── turn (bot responds) ────────────────▶ ACTIVE
       └────┬────┘ ──── turn_count % N == 0 ──▶ auto-checkpoint
            │
            │ relay_checkpoint called (explicit)
            ▼
   ┌─────────────────┐
   │ PENDING_APPROVAL│  (human must respond)
   └────────┬────────┘
            │ approve          │ reject       │ approve + guidance
            ▼                  ▼              ▼
       ┌────────┐       ┌──────────┐   ┌────────────────┐
       │ ACTIVE │       │ REJECTED │   │ ACTIVE         │
       └────┬───┘       └──────────┘   │ (guidance turn │
            │                          │  injected)     │
            │ relay_conclude OR        └────────────────┘
            │ turn_count >= max_turns
            ▼
       ┌────────┐
       │ CLOSED │  (summary → mem0, turn files deleted, bridge notify)
       └────────┘
```

**Auto-conclude at max_turns:** when `turn_count >= max_turns`, bot-config-api fires `_auto_conclude_thread()` as a background task — generates epoch summary, persists to mem0 for all participants, sends bridge notifications — then returns HTTP 409 to the caller.

---

## 7. Thread Turn Protocol

Each turn is a synchronous call from bot-config-api to a target bot:

```
POST http://agentopia-{bot_name}.agentopia.svc.cluster.local:18789/v1/chat/completions
Authorization: Bearer {RELAY_TOKEN of target bot}
Content-Type: application/json

{
  "model": "openclaw:{agentId}",
  "messages": [
    // 1. Epoch summaries — system role
    {"role": "system", "content": "Prior brainstorm summary:\n{epoch-1 summary}\n---\n{epoch-2 summary}"},

    // 2. Thread constraint — system role (prevents nested relay tool calls)
    {"role": "system", "content": "Thread: {topic}. Turn {n}/{max}. Reply with your own perspective directly — do NOT use relay_send, relay_create_thread, or call other bots during this response. The thread orchestrator manages all participant coordination."},

    // 3. Recent raw turns — alternating user/assistant pairs (last 3 turns only)
    {"role": "user",      "content": "[bot-a]: {turn-N-2 message}"},
    {"role": "assistant", "content": "{turn-N-2 response}"},
    {"role": "user",      "content": "[bot-b]: {turn-N-1 message}"},
    {"role": "assistant", "content": "{turn-N-1 response}"},

    // 4. Current turn — user role
    {"role": "user", "content": "[{from_bot}]: {message}"}
  ],
  "user": "thr_{uuidv4}_epoch-{k}-t{turn_number}",  // unique per turn — prevents lane collision
  "stream": false
}
```

### Context Strategy (Incremental, No Rebuild)

**What's NOT included** (vs. gateway bootstrap):
- SOUL.md (~15K tokens) — not needed; debate has its own system prompt
- USER.md (~5K tokens) — not needed for bot-to-bot debate
- IDENTITY.md, BOOTSTRAP.md, AGENTS.md, TOOLS.md (~10K tokens) — not needed
- mem0 context (~5K tokens) — not needed per turn
- Full transcript history (~15K+ tokens) — only last 3 turns sent

Total per turn: ~1–2K tokens vs. 50K+ through gateway.

### Session Keys

| Purpose | Key format | Lifetime |
|---------|-----------|----------|
| Brainstorm thread turn | `thr_{uuid32}_epoch-{k}-t{turn_number}` | Unique per turn (rotates at each new turn and at epoch boundary) |
| Rolling summarizer call | `summarizer-{thread_id}` | Isolated — never pollutes brainstorm context |
| Single-turn relay (`relay_send`) | `relay-{uuid4().hex[:12]}` | Per-call, stateless |

> **Why unique per turn?** Re-using the same `user` key across Turn 1 and its retry caused lane collision in the gateway — Turn 2 queued behind Turn 1's still-active session for 15s. Using `{epoch}-t{n}` guarantees every call lands in a fresh session, eliminating the collision while still rotating at epoch boundaries.

---

## 8. Trigger Patterns

### Pattern 1: Human → SA Bot → Specialists

```
Human sends to SA Bot via Telegram:
  "Brainstorm architecture for X"
    │
    └─► SA Bot calls relay plugin tools:
          relay_create_thread(topic="...", participants=["bot-b", "bot-c"])
          relay_send_turn(thread_id, to_bot="bot-b", message="...")
          relay_send_turn(thread_id, to_bot="bot-c", message="...")
          relay_checkpoint(thread_id)   ← pauses + sends summary to human
          relay_send(to_bot="human-relay", message=summary)  ← notify human
```

### Pattern 2: Direct API trigger

```
POST /api/v1/threads
Authorization: Bearer {SA Bot RELAY_TOKEN}
{
  "topic": "Evaluate database migration strategy",
  "participants": ["bot-b", "bot-c"],
  "initiator": "bot-a",
  "max_turns": 20
}
```

### Pattern 3: SA Bot autonomous orchestration

```
Turn 1: SA (bot-a) → Bot-B: "What are the infra considerations?"
Turn 2: SA (bot-a) → Bot-C: "What are the security risks?"
Turn 3: SA (bot-a) → Bot-B: "Bot-C flagged X, your response?"
Turn 4: SA (bot-a) → Bot-C: "Bot-B suggests Y, agree?"
[Auto-checkpoint at turn 10 OR SA calls relay_checkpoint]
Checkpoint: SA sends epoch summary to Human via Telegram
Human: "approve" → thread resumes
Turn 5: SA → Bot-B: "Approved. Draft final implementation plan."
relay_conclude(thread_id) → summary saved to mem0 for all participants
```

---

## 9. Man-in-the-Loop: Checkpoint Pattern

```
1. Thread reaches checkpoint:
   - SA bot calls relay_checkpoint (explicit)
   - OR turn_count % auto_checkpoint_every == 0 (automatic, default: every 10 turns)

2. bot-config-api:
   - Sets thread.status = "PENDING_APPROVAL"
   - Calls SA bot /v1/chat/completions with user="summarizer-{thread_id}" to generate summary
   - Writes summary-epoch-{k}.json to PVC
   - POSTs summary to mem0 for all participants (non-blocking)
   - Returns 200 — SA bot then relay_sends the summary to human via Telegram

3. Human responds via Telegram (processed by SA bot's /api/v1/threads/{id}/approve):
   - "approve"          → status=active, epoch already advanced
   - "reject"           → status=rejected, archived
   - "approve: guidance" → status=active, guidance injected as next system turn

4. FINAL decisions (deploy, architecture choice, irreversible action) MUST hit a checkpoint.
```

---

## 10. Concurrency Model

| Dimension | Limit | Reason |
|-----------|-------|--------|
| Max threads | 9 (3 bots × 3 threads) | Protect bot context windows from saturation |
| Max turns per thread | 20 (configurable) | Prevent infinite loops |
| Concurrent threads per bot | 3 | 3 parallel sessions via distinct session keys |
| Auto-compact every N turns | 10 (configurable) | Keep context within window |
| Thread timeout | 30 min (auto-checkpoint) | Prevent stalled threads from blocking |
| Max concurrent turns per `to_bot` | 1 | Queue second call — return 429 with Retry-After |

---

## 11. Context Compaction: Rolling Summary

Openclaw HTTP gateway has no API to reset or compact a session. Sessions grow unbounded without mitigation.

```
Every N turns (or at relay_checkpoint):

  1. Call SA bot /v1/chat/completions:
     - system: "You are a summarizer."
     - user:   "Summarize these brainstorm turns into 3-5 key points: {turns}"
     - user:   "summarizer-{thread_id}"   ← isolated session, never pollutes brainstorm

  2. Write summary-epoch-{k}.json to PVC

  3. advance_epoch():
     - Increment current_epoch
     - Rotate session key: thr_{prev_uuid}_epoch-{k} → thr_{new_uuid}_epoch-{k+1}
       (old gateway session abandoned; new calls start a fresh session)

  4. Next turns inject epoch summaries as system messages
     + last 3 raw turns as user/assistant pairs
```

---

## 12. Shared Memory Layout

```
/openclaw-state/shared-memory/
├── threads/
│   └── {thread_id}/
│       ├── meta.json              # thread state + lease lock
│       ├── turn-0001.json         # write-once: {turn_id, from, to, message, response, status, response_hash, ts}
│       ├── turn-0002.json
│       ├── summary-epoch-1.json   # compact summary of turns 1-10
│       ├── turn-0011.json         # epoch 2 turns start here
│       └── summary-final.json     # written at CLOSE → fed to mem0
│
└── inbox/
    └── {bot_name}/
        └── {ts}-{uuid}.json       # async fallback queue
```

---

## 13. PVC Locking & Atomic Writes

### meta.json — lease-based mutex

```json
{
  "id": "thr_abc123",
  "status": "active",
  "turn_count": 7,
  "current_epoch": 1,
  "session_key": "thr_f3a8...b2c1_epoch-1",
  "lock_owner": "bot-config-api-pod-xyz",
  "lock_until": "2026-03-04T10:01:30Z"
}
```

- Writer: acquire lease only if `lock_until` is expired or `lock_owner == self`
- Lock TTL: 30s with heartbeat renewal during long turns
- On crash: lock expires, next writer takes over

### Turn files — atomic rename

```python
# Write temp → fsync → atomic rename (POSIX safe)
tmp = thread_dir / f".tmp-{uuid4().hex[:8]}"
tmp.write_text(json.dumps(data), encoding="utf-8")
tmp.rename(thread_dir / filename)  # atomic
```

Turn files are **write-once**. Status updates go into meta.json (with lease), never by modifying turn files in-place.

---

## 14. Idempotency

```python
# Before sending a turn:
turn_file = thread_dir / f"turn-{turn_id}.json"
if turn_file.exists():
    data = json.loads(turn_file.read_text())
    if data["status"] == "done":
        return data["response"]   # cached — do NOT re-send

# Turn statuses:
# pending → turn file created, not yet sent
# sent    → HTTP in-flight (crash? retry after deadline)
# done    → response persisted; never re-send
# timeout → deadline passed; one retry, then queue fallback
# queued  → written to PVC inbox fallback
```

Session key is unique per turn (`thr_{uuid}_epoch-{k}-t{n}`). Each call — including retries — uses a fresh key to avoid lane collision. Thread context is explicitly rebuilt in `messages[]` from PVC turn files on every call.

---

## 15. relay Plugin: Tools Exposed to Bots

Plugin config in `configmap-config.yaml`:

```json
"relay": {
  "enabled": true,
  "config": {
    "serviceUrl": "http://bot-config-api.agentopia.svc.cluster.local"
  }
}
```

Plugin reads `${AGENTOPIA_RELAY_TOKEN}` from bot env and passes it as Bearer on all Thread API calls.

| Tool | HTTP call | Auth |
|------|-----------|------|
| `relay_send` | `POST /api/v1/relay/{to_bot}` | Bearer = calling bot's RELAY_TOKEN |
| `relay_create_thread` | `POST /api/v1/threads` | Bearer = calling bot's RELAY_TOKEN |
| `relay_send_turn` | `POST /api/v1/threads/{id}/turn` | Bearer = calling bot's RELAY_TOKEN |
| `relay_read_thread` | `GET /api/v1/threads/{id}` | Bearer = calling bot's RELAY_TOKEN |
| `relay_checkpoint` | `POST /api/v1/threads/{id}/checkpoint` | Bearer = calling bot's RELAY_TOKEN |
| `relay_conclude` | `POST /api/v1/threads/{id}/conclude` | Bearer = calling bot's RELAY_TOKEN |

### Bot Name Resolution

Bots may address each other using their Telegram @username rather than K8s bot name. The relay router resolves both:

```python
# K8sService.resolve_bot_name(name)
def normalize(s): return s.lstrip("@").lower().replace("-", "_")

for bot in get_bots_from_applicationset():
    if normalize(bot["botName"])          == normalize(name): return bot["botName"]
    if normalize(bot["telegramUsername"]) == normalize(name): return bot["botName"]
return None  # → 404
```

### Async Fire-and-Forget Mode

```
POST /api/v1/relay/{to_bot}?async=true
→ {"status": "queued", "delivered_via": "queue"}  (immediate, ~1ms)
```

Use `?async=true` for heavy orchestration tasks where the caller does not need to wait (e.g. "start a debate thread"). Writes to PVC inbox immediately and returns.

| Mode | Use case |
|------|----------|
| Default (sync) | Short requests expecting a direct reply |
| `?async=true` | Fire-and-forget delegation (orchestration tasks that take 120s+) |

---

## 16. Message Envelope (Turn File on PVC)

```json
{
  "thread_id": "thr_abc123",
  "turn_id": "turn-0003",
  "from_bot": "bot-a",
  "to_bot": "bot-b",
  "message": "Given bot-c's concern about downtime, what's your view on blue-green migration?",
  "status": "done",
  "response": "Blue-green makes sense given the 99.9% SLA requirement...",
  "response_hash": "sha256:abc123...",
  "epoch": 1,
  "turn_number": 3,
  "sent_at": "2026-03-04T10:00:00Z",
  "responded_at": "2026-03-04T10:00:45Z"
}
```

---

## 17. mem0 Persistence

Called at three points (all non-blocking — mem0 failure is logged, does not affect thread state):

1. **After every turn** (intermediate persist): captures partial context even if thread never reaches `/conclude`
2. **At checkpoint** (per epoch): persist epoch summary for all participants
3. **At conclude / auto-conclude** (final): persist full summary for all participants

```python
async def _persist_to_mem0(bot_name: str, text: str, thread_id: str) -> None:
    mem0_url = os.getenv("MEM0_API_URL", "http://mem0-api.agentopia.svc.cluster.local:8000")
    async with httpx.AsyncClient(timeout=10.0) as client:
        try:
            await client.post(f"{mem0_url}/v1/memories", json={
                "messages": [{"role": "user", "content": text}],
                "user_id": f"openclaw-{bot_name}",
                "metadata": {"thread_id": thread_id, "type": "brainstorm_summary"},
            })
        except Exception as exc:
            logger.warning("mem0 persist failed for %s (non-fatal): %s", bot_name, exc)
```

---

## 18. Reliability Patterns

### PVC inbox queue fallback
If `/v1/chat/completions` fails (pod down, timeout), fall back to PVC inbox queue. Turn status → `queued`. SA bot notified via response.

### Thread recovery on restart
On bot-config-api restart, in-memory state is lost. `recover_threads()` at startup:
1. Scan `threads/*/meta.json` for non-CLOSED threads
2. For each: find last `status=done` turn
3. Resume from next pending turn
4. If `status=sent` with `sent_at` > 75s → assume lost, mark `timeout`, retry

### Circuit breaker per bot
After 3 consecutive failures per bot in a thread: mark bot unavailable, notify SA bot via turn response.

### Turn deadline
Default 125s (120s gateway read timeout + 5s buffer). On timeout: retry once (with 2s backoff) → queue fallback.
`THREAD_DELIVER_RETRIES` env (default: 1) controls number of retries.

---

## 19. Model Tiering & Cost

### Provider Allocation

| Workload | Provider | Key Source | Rationale |
|----------|----------|-----------|-----------|
| **Direct chat** (user → bot) | Anthropic (native API) | Vault (`ANTHROPIC_API_KEY`) | Full Claude features, prompt caching, personality-rich |
| **Debate turns** (A2A) | agentopia-llm-proxy → Codex | K8s Secret (`agentopia-llm-proxy-env`) | No per-token cost, OpenAI-compatible |
| **Bridge notifications** (A2A) | agentopia-llm-proxy → Codex | K8s Secret (`agentopia-llm-proxy-env`) | Short notification, no per-token cost |
| **Epoch summaries** (A2A) | agentopia-llm-proxy → Codex | K8s Secret (`agentopia-llm-proxy-env`) | Summarization, no per-token cost |
| **mem0 / extract** | OpenRouter | K8s Secret (`mem0-env` / `bot-config-api-env`) | Existing integration |

> **Key insight**: Direct chat stays on Anthropic native API (via gateway/Vault) because it needs full personality + Claude features. All internal A2A utility tasks go through `agentopia-llm-proxy` → Codex OAuth — cost is $0 per token.

### Configuration

```python
MODEL_CONFIG = {
    "direct_chat": os.getenv("MODEL_DIRECT_CHAT", "anthropic/claude-sonnet-4-6"),
    "debate_turn": os.getenv("MODEL_DEBATE", "openai-codex/gpt-5.1"),
    "bridge_notify": os.getenv("MODEL_BRIDGE", "openai-codex/gpt-5.1"),
    "epoch_summary": os.getenv("MODEL_EPOCH", "openai-codex/gpt-5.1"),
}

TOKEN_BUDGET = {
    "debate_turn": 1024,
    "bridge_notify": 512,
    "epoch_summary": 500,
}
```

Provider env vars (swap without code change):

| Provider | `LLM_BASE_URL` | Model format |
|----------|----------------|-------------|
| **agentopia-llm-proxy** (default) | `http://agentopia-llm-proxy.agentopia.svc.cluster.local:18789/v1` | `openai-codex/gpt-5.1` |
| OpenRouter (fallback) | `https://openrouter.ai/api/v1` | `anthropic/claude-haiku-4-5` |
| OpenAI | `https://api.openai.com/v1` | `gpt-4o-mini` |

### Cost Projection

| Call Type | Before (Sonnet × 50K) | After (Codex × 2–3K) | Reduction |
|-----------|----------------------|-----------------------|-----------|
| Debate turn | ~$0.075 | ~$0 (Codex subscription) | ~100% |
| Bridge notify | ~$0.075 | ~$0 | ~100% |
| Epoch summary | ~$0.075 | ~$0 | ~100% |
| 1 debate (18 turns) | ~$1.35 | ~$0 | ~100% |
| Daily (2 debates + chat) | ~$3.00 | chat cost only | ~95% |

---

## 20. A2A Protocol Mapping

### Roles

| Agentopia Component | A2A Role | Responsibility |
|---------------------|----------|----------------|
| **bot-config-api** | A2A Client + Server (Orchestrator) | Creates tasks, sends messages, manages lifecycle. Hosts A2A endpoints on behalf of all bots (single physical endpoint, multiple logical agents) |
| **Each bot** | Logical A2A Agent | Has its own Agent Card and identity; A2A requests handled by bot-config-api |
| **Gateway** | Internal (not A2A) | Handles Telegram ↔ LLM for direct chat only |

### Task Mapping

| Agentopia Flow | A2A Task | contextId | Lifecycle |
|----------------|----------|-----------|-----------|
| Debate session | 1 Task per debate | `debate-{thread_id}` | submitted → working → (turns) → completed |
| Bridge notification | 1 Task per notification | `bridge-{source_bot}-{target_bot}` | submitted → working → completed |
| Epoch summary | 1 Task per epoch | `epoch-{bot_id}-{date}` | submitted → working → completed (artifact: summary) |

### Agent Card per Bot

Served by bot-config-api at `/a2a/{bot_name}/.well-known/agent-card.json`. Generated from `bot.yaml` + `SOUL.md` metadata:

```json
{
  "name": "Aria",
  "url": "http://bot-config-api.agentopia.svc.cluster.local:8001/a2a/aria",
  "description": "Creative writing specialist",
  "version": "1.0",
  "capabilities": {"streaming": true},
  "skills": [
    {"id": "debate", "name": "Structured Debate"},
    {"id": "bridge-respond", "name": "Bridge Response"},
    {"id": "summarize", "name": "Epoch Summary"}
  ]
}
```

### A2A + MCP Integration

```
                A2A (agent ↔ agent)
Bot A ◄────────────────────────────────────► Bot B
  │                                            │
  │  MCP (agent → tools)                       │  MCP (agent → tools)
  ├──► mem0-api (memory)                       ├──► mem0-api (memory)
  ├──► file system (local-path PVC)             ├──► file system (local-path PVC)
  └──► web search                              └──► web search
```

A2A and MCP are complementary. No change needed to existing MCP integrations.

---

## 21. A2A SDK Integration

### Implementation in bot-config-api

```python
# src/a2a/server.py
from a2a.server.apps import A2AStarletteApplication
from a2a.server.request_handlers import DefaultRequestHandler
from a2a.server.agent_execution import AgentExecutor, RequestContext
from a2a.server.events.event_queue import EventQueue
from a2a.server.tasks import InMemoryTaskStore

class AgentopiaExecutor(AgentExecutor):
    async def execute(self, context: RequestContext, event_queue: EventQueue) -> None:
        skill_id = self._identify_skill(context)
        if skill_id == "debate":
            await self._handle_debate_turn(context, event_queue)
        elif skill_id == "bridge-respond":
            await self._handle_bridge(context, event_queue)
        elif skill_id == "summarize":
            await self._handle_epoch_summary(context, event_queue)

    async def cancel(self, context: RequestContext, event_queue: EventQueue) -> None:
        pass
```

### FastAPI Mount

```python
# src/main.py
def create_a2a_app(bot_name: str, agent_card: AgentCard) -> Starlette:
    executor = AgentopiaExecutor(bot_name=bot_name)
    handler = DefaultRequestHandler(
        agent_executor=executor,
        task_store=InMemoryTaskStore(),
        queue_manager=InMemoryQueueManager(),
    )
    a2a_app = A2AStarletteApplication(
        agent_card=agent_card,
        http_handler=handler,  # NOTE: param is "http_handler", NOT "request_handler"
    )
    return a2a_app.build()

for bot_name, card in bot_cards.items():
    app.mount(f"/a2a/{bot_name}", create_a2a_app(bot_name, card))
```

### Key SDK API Facts (verified against a2a-sdk 0.3.24)

| | |
|---|---|
| **PyPI package** | `a2a-sdk` (extras: `a2a-sdk[http-server]`) |
| **Import root** | `from a2a...` (NOT `from a2a_sdk...`) |
| **Server app** | `a2a.server.apps.A2AStarletteApplication` |
| **Request handler** | `a2a.server.request_handlers.DefaultRequestHandler` |
| **Executor ABC** | `a2a.server.agent_execution.AgentExecutor` |
| **Event queue** | `a2a.server.events.event_queue.EventQueue` — use `enqueue_event()` |
| **Task store** | `a2a.server.tasks.InMemoryTaskStore` (Tier 1), `PostgresTaskStore` (Tier 2 — M1.8) |
| **Types** | `a2a.types.AgentCard`, `AgentSkill`, `AgentCapabilities`, `TransportProtocol`, `TaskState` |
| **Server constructor** | `A2AStarletteApplication(agent_card=..., http_handler=...)` |
| **TaskState values** | `submitted`, `working`, `input-required`, `completed`, `canceled`, `failed`, `rejected`, `auth-required`, `unknown` |

> **Rule**: When coding A2A integration, always cross-reference the [a2a-python SDK source](https://github.com/a2aproject/a2a-python) for correct import paths. Do NOT assume from memory.

---

## 22. Observability

### Usage Logging (SQLite)

Every direct LLM call logs usage to SQLite in bot-config-api:

```python
@dataclass
class LLMUsageLog:
    timestamp: datetime
    bot_id: str
    call_type: str   # debate_turn, bridge_notify, epoch_summary
    provider: str
    model: str
    input_tokens: int
    output_tokens: int
    task_id: str
    context_id: str
    cost_usd: float
```

Usage API:
```
GET /api/v1/metrics/usage?period=7d&bot_id=aria
→ { total_tokens, total_cost, breakdown_by_type, breakdown_by_bot, breakdown_by_provider }
```

### Prometheus Metrics (M1.8)

Prometheus counters exported at `/metrics`:

| Metric | Labels | Purpose |
|--------|--------|---------|
| `a2a_requests_total` | `bot`, `success` | Request throughput |
| `a2a_request_failures_total` | `bot` | Error rate for SLO |
| `a2a_idempotency_hits_total` | `bot`, `layer` (cache/postgres) | Dedup effectiveness |
| `a2a_circuit_breaker_state` | `client`, `is_open` | Circuit breaker health |
| `a2a_rate_limit_remaining` | `bot` | Rate limit headroom |

PrometheusRule (`monitoring.yaml`) implements Google SRE multi-window burn-rate model:
- Recording rules: `a2a:error_ratio_5m`, `a2a:error_ratio_30m`, `a2a:error_ratio_1h`, `a2a:error_ratio_6h`
- Alerts: `A2AHighBurnRate` (14.4x, critical), `A2AMediumBurnRate` (6x, warning), `A2ASlowBurnRate` (1x, info)

---

## 23. Implementation Plan

### Tier 1 — Fix synchronous relay (issues #59, #60) ✅ IMPLEMENTED

| File | Change |
|------|--------|
| `charts/agentopia-bot/templates/configmap-config.yaml` | `gateway.auth.token` → `${AGENTOPIA_RELAY_TOKEN}`. Add `chatCompletions.enabled: true`. |
| `charts/agentopia-base/templates/bot-config-api.yaml` | Add `MEM0_API_URL` env var. |
| `bot-config-api/src/routers/relay.py` | Use `/v1/chat/completions` with `get_relay_token(to_bot)`. Per-call UUID session key. |

### Tier 2 — Thread API (issues #61, #62) ✅ IMPLEMENTED

| File | Change |
|------|--------|
| `bot-config-api/src/models/thread.py` | `ThreadMeta`, `ThreadTurn`, `TurnStatus`, `CreateThreadRequest`, etc. |
| `bot-config-api/src/services/thread_service.py` | State machine, PVC persistence (atomic writes + lease locking), `recover_threads()` |
| `bot-config-api/src/routers/threads.py` | All endpoints with Bearer auth; context builder; auto-checkpoint logic |

### Tier 3 — relay plugin tools (issue #63) ✅ IMPLEMENTED

| File | Change |
|------|--------|
| `gateway/extensions/relay/` | Add 5 thread tools. Bearer `${AGENTOPIA_RELAY_TOKEN}` on all Thread API calls. |
| `charts/agentopia-bot/templates/configmap-config.yaml` | Add thread tools to `tools.allow`. |

### Tier 4 — Rolling summary + mem0 (issue #64) ✅ IMPLEMENTED

| File | Change |
|------|--------|
| `bot-config-api/src/services/thread_service.py` | `advance_epoch()`, `load_epoch_summaries()`, `load_recent_turns()`, `conclude_thread()`, `_persist_to_mem0()` |
| `bot-config-api/src/routers/threads.py` | Trigger `advance_epoch()` at checkpoint; call `_persist_to_mem0()` at checkpoint + conclude |

### Phase mapping (A2A SDK)

| Phase | Scope | Status |
|-------|-------|--------|
| Phase 1 (#77) | A2A SDK integration, Agent Card, Task store | ✅ IMPLEMENTED |
| Phase 2 (#78) | Thread bypass, direct LLM calls, context minimization, Redis task store | ✅ IMPLEMENTED |
| Phase 3 (#79) | Model tiering, Codex via llm-proxy, fallback chain, benchmarks | ✅ IMPLEMENTED |
| Phase 4 (#80) | Token tracking, cost visibility, usage API | ✅ IMPLEMENTED |

---

## 24. What We Are NOT Doing

| Approach | Reason |
|----------|--------|
| **Fork openclaw gateway** | Unnecessary — `/v1/chat/completions` exists natively |
| **kubectl exec** | O(n) overhead per exchange, too heavy for brainstorm turns |
| **NATS/Kafka** | Bots can't subscribe directly (closed system); PVC inbox queue covers async |
| **mTLS/SPIFFE** | Enterprise zero-trust — out of scope for MVP; K8s NetworkPolicy sufficient |
| **openclaw-p2p (Nostr)** | Decentralized pub/sub — async, no sync response |
| **sessions_send / agentToAgent** | Same-gateway process only — won't work cross-pod |
| **Relay proxy (auth translator)** | Post-MVP concern — add when per-capability allowlists are needed |

---

## 25. Success Criteria

| Metric | Target |
|--------|--------|
| Thread turn tokens | < 3K (from 50K+) |
| 1 debate cost | ~$0 (Codex subscription) |
| Daily cost | Chat cost only for automated calls |
| Thread turn latency | < 5s |
| Zero gateway changes | ✅ Confirmed |
| Provider-swappable | Switch via `LLM_BASE_URL` + `LLM_API_KEY` + model env vars |
| A2A spec compliance | Agent Card + Task lifecycle + Messages |

---

## 26. Verification Steps

### Post Tier 1: Relay smoke test

```bash
# 1. Verify configmap
kubectl exec -n agentopia deploy/agentopia-<bot> -- \
  cat /openclaw-config/openclaw.json | jq '.gateway.auth, .gateway.http, .hooks.token'

# 2. Verify chatCompletions endpoint
RELAY_TOKEN=$(kubectl get secret agentopia-gateway-env-<bot> -n agentopia \
  -o jsonpath='{.data.AGENTOPIA_RELAY_TOKEN}' | base64 -d)
AGENT_ID=$(kubectl exec -n agentopia deploy/agentopia-<bot> -- \
  jq -r '.agents.list[0].id' /openclaw-config/openclaw.json)

kubectl exec -n agentopia deploy/agentopia-<bot> -- \
  curl -s http://localhost:18789/v1/chat/completions \
    -H "Authorization: Bearer $RELAY_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"openclaw:$AGENT_ID\",\"messages\":[{\"role\":\"user\",\"content\":\"respond with one word: pong\"}],\"user\":\"thr_smoke_$(date +%s)\"}"

# 3. End-to-end relay via bot-config-api
FROM_RELAY_TOKEN=$(kubectl get secret agentopia-gateway-env-<from_bot> -n agentopia \
  -o jsonpath='{.data.AGENTOPIA_RELAY_TOKEN}' | base64 -d)
kubectl exec -n agentopia deploy/bot-config-api -- \
  curl -s -X POST http://localhost/api/v1/relay/<to_bot> \
    -H "Authorization: Bearer $FROM_RELAY_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"from_bot":"<from_bot>","message":"respond with one word: pong"}'

# 4. Verify MEM0_API_URL
kubectl exec -n agentopia deploy/bot-config-api -- env | grep MEM0
```

### Post Tier 3: Thread API smoke test

```bash
FROM_BOT="<from_bot>"
TO_BOT="<to_bot>"
FROM_TOKEN=$(kubectl get secret agentopia-gateway-env-${FROM_BOT} -n agentopia \
  -o jsonpath='{.data.AGENTOPIA_RELAY_TOKEN}' | base64 -d)

# 1. Create thread
THREAD_ID=$(kubectl exec -n agentopia deploy/bot-config-api -- \
  curl -s -X POST http://localhost/api/v1/threads \
    -H "Authorization: Bearer ${FROM_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{\"topic\":\"Smoke test\",\"participants\":[\"${FROM_BOT}\",\"${TO_BOT}\"],\"initiator\":\"${FROM_BOT}\",\"max_turns\":10}" \
  | jq -r '.id')
echo "Thread: ${THREAD_ID}"

# 2. Send turn
kubectl exec -n agentopia deploy/bot-config-api -- \
  curl -s -X POST http://localhost/api/v1/threads/${THREAD_ID}/turn \
    -H "Authorization: Bearer ${FROM_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{\"from_bot\":\"${FROM_BOT}\",\"to_bot\":\"${TO_BOT}\",\"message\":\"respond with one word: pong\"}"

# 3. Checkpoint
kubectl exec -n agentopia deploy/bot-config-api -- \
  curl -s -X POST http://localhost/api/v1/threads/${THREAD_ID}/checkpoint \
    -H "Authorization: Bearer ${FROM_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{\"from_bot\":\"${FROM_BOT}\"}"

# 4. Verify epoch summary on PVC
kubectl exec -n agentopia deploy/bot-config-api -- \
  cat /openclaw-state/shared-memory/threads/${THREAD_ID}/summary-epoch-1.json | jq '.epoch, .summary'

# 5. Approve
kubectl exec -n agentopia deploy/bot-config-api -- \
  curl -s -X POST http://localhost/api/v1/threads/${THREAD_ID}/approve \
    -H "Content-Type: application/json" \
    -d '{"action":"approve"}'

# 6. Conclude
kubectl exec -n agentopia deploy/bot-config-api -- \
  curl -s -X POST http://localhost/api/v1/threads/${THREAD_ID}/conclude \
    -H "Authorization: Bearer ${FROM_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{\"from_bot\":\"${FROM_BOT}\",\"final_summary\":\"Smoke test complete.\"}"

# 7. Verify PVC state (should only have meta + summaries, no turn-*.json)
kubectl exec -n agentopia deploy/bot-config-api -- sh -c \
  "ls /openclaw-state/shared-memory/threads/${THREAD_ID}/"
```

---

## 27. References

| Resource | URL | Purpose |
|----------|-----|---------|
| A2A Protocol Spec | https://a2a-protocol.org/latest/specification/ | Official specification (~0.3.0) |
| A2A Project | https://github.com/a2aproject/A2A | Main protocol repo |
| A2A Python SDK | https://github.com/a2aproject/a2a-python | Official Python SDK (`a2a-sdk` on PyPI) |
| A2A Samples | https://github.com/a2aproject/a2a-samples | Reference implementations |
| GitHub Issues | https://github.com/ai-agentopia/agentopia-infra/issues | #59–#65, #77–#80 |
