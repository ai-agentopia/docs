---
title: "A2A Improvement Plan: Production-Ready A2A-First Architecture"
---

# A2A Improvement Plan: Production-Ready A2A-First Architecture

> **Status**: Phase 1 + Phase 2 COMPLETE — all production hardening implemented
> **Date**: 2026-03-09 (updated: Phase 2 verified complete)
> **Scope**: `bot-config-api` A2A module (`a2a_protocol/` + `routers/threads.py`)
> **Prerequisite**: [A2A Solution Protocol](./a2a-solution-protocol.md) (current implementation)

---

## 1. Problem Statement

The A2A implementation is now **A2A-first** with production hardening:

| Layer | Status | Implementation |
|-------|--------|----------------|
| Agent Card + Discovery | **Live** | `card_generator.py` + `/.well-known/agent-card.json` |
| JSON-RPC Server | **Live** | `server.py` — `AgentExecutor`, task lifecycle, skill allowlist |
| Internal Orchestration | **Live** | `threads.py` → `AgentopiaA2AClient` → A2A JSON-RPC → `AgentopiaExecutor` |
| External Interop | **Live** | Separate httpx clients (no token leak), SSRF protection, allowlist |
| Security Hardening | **Live** | Participant validation, lease locking, auth isolation, threading.Lock |

**Phase 1 (A2A-first)**: Complete. `threads.py` routes through `AgentopiaA2AClient.send_task()` → A2A JSON-RPC server when `A2A_DIRECT_LLM=true`. Cost optimization preserved (~1-2K tokens per turn).

**Phase 2 (Production Hardening — M1.8)**: Complete. Postgres task store (`a2a_tasks`), Postgres idempotency (`a2a_idempotency`), Redis cache backend, distributed locking (local/postgres/redis), Prometheus metrics (burn-rate SLO), TLS toggle, chaos test manifests, migration tool (SQLite→Postgres), rolling update strategy with startupProbe, operational runbook.

---

## 2. Architecture: Before vs After

```
CURRENT (A2A-first, live):
  threads.py ──→ AgentopiaA2AClient.send_task() ──→ POST /a2a/{bot} [JSON-RPC] ──→ AgentopiaExecutor ──→ LLMClient ──→ OpenRouter
                 (a2a-sdk ClientFactory)             (localhost, JSON-RPC)          (skill routing)       (unchanged)

PRODUCTION-HARDENED (completed):
  + SQLiteTaskStore (opt-in via A2A_TASK_STORE=sqlite)
  + Correlation: thread_id ↔ a2a_task_id (end-to-end tracing)
  + Cancel semantics (conclude/reject → cancel in-flight tasks)
  + Auth per A2A endpoint
  + External agent discovery + routing
  + Threading.Lock per thread for same-process concurrency
  + Circuit breaker with graceful degradation
```

**Token cost: unchanged** — context building stays minimal (topic + last 3 turns). Only the transport layer changes.

**Latency overhead**: +1 localhost hop (~<1ms), negligible.

---

## 3. Phase 1: A2A-First Transport

### 3.1 New File: `a2a_protocol/mount_registry.py`

> **Why separate module**: `main.py` imports from `a2a_protocol/` (server.py, card_generator.py).
> If `a2a_client.py` (in `a2a_protocol/`) imports back from `main.py` for `mount_bot_a2a()`,
> this creates a **circular import**: `main → routers.threads → a2a_client → main`.
> Solution: extract mount state and logic into `mount_registry.py` — both `main.py` and
> `a2a_client.py` import from this module, no cycle.

```python
# a2a_protocol/mount_registry.py
# Owns: _mounted_bots dict, mount_bot_a2a(), ensure_bot_mounted()
# Imported by: main.py (to wire routes), a2a_client.py (to auto-mount before send)
# Does NOT import from main.py — breaks the cycle
```

| Function | Description |
|----------|-------------|
| `mount_bot_a2a(app, bot_name, bot_yaml?, soul_md?)` | Mount A2A sub-app on FastAPI app |
| `ensure_bot_mounted(app, bot_name)` | Idempotent mount with `asyncio.Lock` |
| `is_mounted(bot_name) → bool` | Check mount status |
| `list_mounted() → list[str]` | List all mounted bots |

`main.py` passes `app` reference once at startup. `a2a_client.py` calls `ensure_bot_mounted()`.

### 3.2 New File: `a2a_protocol/a2a_client.py`

A2A client wrapper using the official SDK (`a2a.client.ClientFactory`, v0.3.24+).

**Class: `AgentopiaA2AClient`**

| Method | Signature | Description |
|--------|-----------|-------------|
| `__init__` | `(base_url, timeout)` | Shared `httpx.AsyncClient`, connection pooling |
| `send_task` | `(bot_name, message, skill_id, context_id?) → (bool, str, str?)` | Send A2A task, return `(ok, response_text, task_id)` |
| `cancel_task` | `(task_id) → bool` | Cancel in-flight task (Phase 2 wiring) |

**`send_task` flow**:

1. Call `ensure_bot_mounted()` from `mount_registry` (no import from `main.py`)
2. Build `SendMessageRequest`:
   - `Message(role=user, parts=[TextPart(text=message)], task_id=None)`
   - `Message.metadata` = full thread metadata (schema v1)
   - **DO NOT set `message.task_id`** — SDK generates it (see below)
3. Call `factory.create(agent_card)` → `client.send_message(request)` (via `ClientFactory`)
4. Parse `SendMessageResponse` with fallback chain:
   ```python
   def extract_text_from_result(result) -> str:
       """Extract response text from A2A result (Task or Message)."""
       # Path 1: Task with status.message
       if isinstance(result, Task) and result.status.message:
           for part in result.status.message.parts:
               if hasattr(part.root, 'text'):
                   return part.root.text
       # Path 2: Task with artifacts
       if isinstance(result, Task) and result.artifacts:
           for artifact in result.artifacts:
               for part in artifact.parts:
                   if hasattr(part.root, 'text'):
                       return part.root.text
       # Path 3: Direct Message response
       if isinstance(result, Message):
           for part in result.parts:
               if hasattr(part.root, 'text'):
                   return part.root.text
       return ""  # empty fallback
   ```
5. Return `(ok, response_text, task_id)` — `task_id` from response for correlation

> **CRITICAL — task_id behavior (verified against a2a-sdk 0.3.24)**:
>
> The SDK's `DefaultRequestHandler._setup_message_execution()` (line 167) raises
> `TaskNotFoundError` if `message.task_id` is set but the task doesn't exist in the store.
> This means: **we CANNOT set a deterministic task_id on first-time messages**.
>
> Correct approach:
> - `message.task_id = None` → SDK auto-generates UUID via `_check_or_generate_task_id()`
> - Store returned `task.id` in `ThreadTurn.a2a_task_id` for correlation
> - Use `turn_id` in `message.metadata` as the idempotency key (thread-layer dedup)
> - `task_id` is for tracing only, not for idempotency

**Why SDK `ClientFactory` (not raw httpx)**:
- JSON-RPC request/response handling built-in
- Separate `_internal_factory` and `_external_factory` — auth token isolation (internal has token, external never)
- Future-proof: when A2A endpoints move to different pods, only change `base_url`
- Protocol-compliant — no manual JSON-RPC parsing

**Singleton**: `get_a2a_client()` → cached instance (same pattern as current `_get_a2a_llm_client()`).

### 3.2 Modified: `a2a_protocol/server.py`

Fix skill routing — accept `skill_id` from message metadata instead of keyword guessing.

**Current problem** (line 91-103):
```python
def _identify_skill(self, text: str) -> str:
    text_lower = text.lower()
    if "debate" in text_lower or "stance:" in text_lower:
        return "debate"
    # ... keyword matching — fragile if payload is minimal
```

**Fix**: Priority-based routing:
1. Check `message.metadata["skill_id"]` (explicit, from orchestrator)
2. Fall back to keyword matching (backward compat for external callers)

**New method**:
```python
def _get_skill_from_context(self, context: RequestContext) -> str | None:
    """Extract skill_id from A2A message metadata."""
    # Access message metadata from the request context
    # Return skill_id if present, None otherwise
```

**Backward compatible**: External A2A callers without metadata still work via keyword fallback.

### 3.3 Modified: `routers/threads.py`

Replace 3 functions — swap `LLMClient` for `AgentopiaA2AClient`.

#### 3.3a: `_deliver_turn_a2a()` (line 264)

**Before**:
```python
client = _get_a2a_llm_client()
response = await client.create_message(
    model=MODEL_CONFIG["debate_turn"],
    system=f"You are {to_bot}...",
    messages=[{"role": "user", "content": context_text}],
    max_tokens=TOKEN_BUDGET["debate_turn"],
)
return True, response["content"]
```

**After**:
```python
a2a = get_a2a_client()
formatted_message = _format_debate_context(thread, to_bot, message, turn_number, svc)
ok, text, task_id = await a2a.send_task(
    bot_name=to_bot,
    message=formatted_message,
    skill_id="debate",
    context_id=thread_id,
)
# task_id is SDK-generated UUID — store in ThreadTurn for correlation
return ok, text
```

**Context format** (sent as message text — system prompt managed by `AgentopiaExecutor`):
```
Topic: {thread.topic}
You are responding as: {to_bot}
Turn {turn_number} of {thread.max_turns}

Recent conversation:
[Turn N-2] bot-a: ...
[Turn N-1] bot-b: ...
[Turn N] bot-a: ...

Current message from {from_bot}:
{message}
```

#### 3.3b: `_generate_epoch_summary_a2a()` (line ~380)

```python
# Before: client.create_message(model=MODEL_CONFIG["epoch_summary"], ...)
# After:  a2a.send_task(bot_name=thread.initiator, message=turns_text, skill_id="summarize")
```

#### 3.3c: `_notify_participant_bridge_a2a()` (line ~400)

```python
# Before: client.create_message(model=MODEL_CONFIG["bridge_notify"], ...)
# After:  a2a.send_task(bot_name=bot_name, message=summary, skill_id="bridge-respond")
```

#### 3.3d: Import changes

```python
# REMOVE (no longer needed in threads.py):
from a2a_protocol.llm_client import LLMClient, MODEL_CONFIG, TOKEN_BUDGET

# ADD:
from a2a_protocol.a2a_client import get_a2a_client
```

Note: `LLMClient` is still used by `AgentopiaExecutor` in `server.py` — it just moves one layer deeper (threads.py no longer calls it directly).

### 3.4 Modified: `main.py`

Auto-mount bots when thread operations require them.

**Current issue**: `mount_bot_a2a()` only called via explicit `POST /api/v1/a2a/mount/{bot_name}`. Thread operations may hit unmounted bots.

**Fix**: `AgentopiaA2AClient.send_task()` calls `ensure_bot_mounted(app, bot_name)` before every request.

```python
async def ensure_bot_mounted(app: Starlette, bot_name: str) -> None:
    """Mount bot A2A endpoint if not already mounted. Thread-safe."""
    if bot_name in _mounted_bots:
        return
    async with _mount_lock:
        if bot_name in _mounted_bots:
            return
        mount_bot_a2a(app, bot_name)
```

### 3.5 Usage Logging Update

**Before**: `log_usage()` called in `threads.py` after each `LLMClient` call.

**After**: `log_usage()` called in `AgentopiaExecutor` (server.py) after each LLM call.

Rationale: single logging point at the executor level. `threads.py` is now purely transport — it doesn't know token counts.

### 3.6 `A2A_DIRECT_LLM` Flag Semantics

| Flag Value | Before (current) | After (Phase 1) |
|-----------|-------------------|------------------|
| `true` | `LLMClient` direct call | `A2AClient` → A2A server → `LLMClient` |
| `false` | Gateway `/v1/chat/completions` | Gateway `/v1/chat/completions` (unchanged) |

Flag name kept for backward compatibility. Meaning shifts from "bypass everything" to "bypass gateway, use A2A".

### 3.7 Phase 1 File Summary

| File | Action | Key Changes |
|------|--------|-------------|
| `a2a_protocol/mount_registry.py` | **NEW** | Mount state + `ensure_bot_mounted()` — breaks circular import |
| `a2a_protocol/a2a_client.py` | **NEW** | SDK client wrapper, `send_task()` / `cancel_task()`, response extraction |
| `a2a_protocol/server.py` | MODIFY | `_get_skill_from_context()`, metadata-based skill routing |
| `routers/threads.py` | MODIFY | 3 functions swap `LLMClient` → `A2AClient` |
| `main.py` | MODIFY | Delegate mount logic to `mount_registry.py` |
| `a2a_protocol/card_generator.py` | MINOR | Skill metadata enrichment |
| `a2a_protocol/usage_logger.py` | MINOR | Move `log_usage()` calls to server.py |

---

## 4. Phase 2: Production Hardening

### 4.1 Task Store — Multi-Backend (DONE — `task_store.py`)

**File**: `a2a_protocol/task_store.py`

Three backends, selected via `A2A_TASK_STORE` env var:

| Backend | Env Value | Use Case | Table |
|---------|-----------|----------|-------|
| `InMemoryTaskStore` | `memory` (default) | Tier 1 single-pod, testing | — |
| `SQLiteTaskStore` | `sqlite` | Tier 1 persistent, local dev | file-based |
| `PostgresTaskStore` | `postgres` | **Tier 2 multi-pod production** | `a2a_tasks` |

- **Factory**: `get_task_store()` returns appropriate store based on env config
- **Postgres**: requires `DATABASE_URL` env var, uses `psycopg` async + connection pool
- **SQL table**: `a2a_tasks` (matches Helm `init.sql` schema — NOT `tasks`)
- **Migration**: `migration.py` provides SQLite→Postgres with MD5 checksum verification

### 4.2 Correlation: Thread ↔ A2A Task (DONE — `models/thread.py:60`, `thread_service.py:293`)

Map each thread turn to an A2A task_id for end-to-end tracing.

> **CRITICAL — SDK constraint (verified against a2a-sdk 0.3.24)**:
>
> `DefaultRequestHandler._setup_message_execution()` raises `TaskNotFoundError` if
> `message.task_id` is set but the task doesn't exist in the TaskStore.
> Therefore: **DO NOT set deterministic task_id in `message.task_id`**.
> Let SDK generate task_id (UUID), store the returned value for correlation.

**Model change** — `models/thread.py`:
```python
class ThreadTurn(BaseModel):
    # ... existing fields ...
    a2a_task_id: Optional[str] = None       # SDK-generated, stored after response
    a2a_context_id: Optional[str] = None    # SDK-generated context ID
```

**Correlation flow** — `threads.py`:
```python
# 1. Send task (no task_id set — SDK generates)
ok, text, task_id = await a2a.send_task(bot_name=to_bot, message=msg, skill_id="debate")

# 2. Store SDK-generated task_id in ThreadTurn for correlation
svc.record_turn_done(thread_id, turn_number, response=text)
# + update turn with a2a_task_id=task_id
```

**Idempotency key**: `turn_id` in `message.metadata` (thread-layer), NOT `task_id` (A2A-layer).

**Benefits**:
- Debug trace: thread turn 5 → stored `a2a_task_id` → LLM call in usage log
- Query: "show all A2A tasks for thread X" via ThreadTurn scan
- Cancel: stored `a2a_task_id` used for `tasks/cancel` on conclude

### 4.3 Idempotency (DONE — 2 Layers: `threads.py:800`, `server.py:188`)

> **Design change**: Originally 3 layers with deterministic task_id.
> Removed Layer 3 because SDK doesn't support client-set task_id on first message.
> Idempotency is enforced at the **thread layer** (turn_id), not A2A layer (task_id).

| Layer | Where | Mechanism |
|-------|-------|-----------|
| 1 — Thread (primary) | `ThreadService.record_turn_sent()` | If turn file exists with `status=done` → return cached response |
| 2 — Metadata | `message.metadata.turn_id` | Executor can log/check for duplicate turn_id in metadata |

Layer 1 is the **authoritative idempotency gate**. Same turn resent → turn file has `status=done` → cached response returned, no A2A call made.

Layer 2 is defense-in-depth: if executor receives a message with a `turn_id` it has already processed (visible in usage log), it can short-circuit. This is best-effort, not guaranteed.

**Why not A2A-layer idempotency**: SDK generates random UUID per `message/send` call. There is no way to force a deterministic task_id for dedup at the A2A protocol level without conflicting with the SDK's task lifecycle management.

### 4.4 Cancel Semantics (DONE — `threads.py:654`, `server.py:236`)

**Trigger scenarios**:
1. Thread conclude → cancel all in-flight tasks
2. Thread reject → cancel all pending tasks
3. Turn timeout → cancel specific task

**`threads.py` changes**:
```python
# POST /{thread_id}/conclude:
#   1. Collect active a2a_task_ids from ThreadTurns with status=sent
#   2. For each: await a2a_client.cancel_task(task_id)
#   3. Then proceed with conclusion logic

# POST /{thread_id}/approve (action=reject):
#   Same: cancel all in-flight tasks
```

**`server.py` — `AgentopiaExecutor.cancel()`**:
```python
# Before: logger.info(...) only (no-op)
# After:
#   1. Set task status to canceled
#   2. Cancel in-progress LLM call (asyncio.Task.cancel())
#   3. Emit TaskStatusUpdateEvent(state=canceled, final=True)
```

### 4.5 Observability (DONE — `usage_logger.py`, `server.py:304`)

#### 4.5a: Persistent Usage Logging

```python
# Usage DB: /openclaw-state/a2a-usage.db (persistent on local-path PVC)
# Override: A2A_USAGE_DB env var
# Old path /tmp/ was lost on pod restart — migrated to persistent storage
```

#### 4.5b: Structured Correlation Logging

Every log line in the A2A path includes:
```
thread_id | turn_number | task_id | bot_name | skill_id | tokens
```

Example:
```
A2A task completed: thread=thr_abc123 turn=5 task=a1b2c3d4-uuid bot=my-bot skill=debate tokens=1847
```

#### 4.5c: Metrics Endpoint Enhancement

```
GET /api/v1/metrics/usage — add:
  + tasks_total (by state, by bot, by skill)
  + task_duration_avg (by skill)
  + a2a_error_rate (by bot)
  Source: TaskStore + usage_logger
```

### 4.6 Security (DONE — auth `server.py:64`, rate limiting `server.py:118`)

#### 4.6a: A2A Endpoint Auth

- Internal calls (from `threads.py`): include internal service token via `AuthInterceptor`
- External calls: validate against bot's relay token
- Implementation: a2a-sdk middleware (`ClientCallInterceptor`)

#### 4.6b: Input Validation

- Max message size: 10KB (configurable via env)
- Reject messages with >5000 chars in any single part
- Validate `skill_id` against known skills list

#### 4.6c: Rate Limiting

- Per bot: max 120 A2A tasks/minute (configurable via `A2A_RATE_LIMIT_RPM`, default 120)
- Prevents runaway thread loops
- Sliding window in-memory counter (`_RateLimiter` in `server.py:118`), single pod, no Redis needed
- On limit exceeded: emits task-level error via `_emit_error()` (not HTTP 429 — A2A JSON-RPC semantic)

### 4.7 Error Handling & Graceful Degradation (DONE — `a2a_client.py`)

#### 4.7a: A2A → Turn Status Mapping

| A2A TaskState | Thread TurnStatus |
|---------------|-------------------|
| `completed` | `done` |
| `failed` | `failed` |
| `canceled` | `failed` (with reason) |
| `rejected` | `failed` (with reason) |
| `auth_required` | `failed` (config error) |

#### 4.7b: Circuit Breaker

If A2A server returns 5xx for >20% of requests in a 5-minute window:
1. Open circuit → fallback to direct `LLMClient` (current behavior)
2. Log warning: `"A2A circuit breaker open, falling back to direct LLM"`
3. Auto-reset after 30 seconds
4. Reuse circuit breaker pattern from `llm_client.py`

#### 4.7c: Timeout & Retry

- `A2AClient` timeout: 120s (matches `GATEWAY_TIMEOUT`)
- Retry on: `A2AClientTimeoutError`, `A2AClientHTTPError` (503)
- Max retries: 2 (matches `THREAD_DELIVER_RETRIES`)
- No retry on: 400, 401, 404 (client errors)

### 4.8 External Agent Support (DONE — `discovery.py`)

#### 4.8a: Agent Card Discovery

**New file**: `a2a_protocol/discovery.py`

```python
class AgentDiscovery:
    """Discover and cache external A2A agent cards."""

    resolve(agent_url) -> AgentCard
        # Fetch /.well-known/agent-card.json
        # Cache with 5-minute TTL
        # Validate card schema

    list_known_agents() -> list[AgentCard]
        # Internal bots (auto-discovered from _mounted_bots)
        # External agents (configured via env/config)
```

#### 4.8b: Routing to External Agents

Thread participants can be:
- `"my-bot"` → internal, routed to `/a2a/my-bot` on localhost
- `"https://external.agent/a2a"` → external, fetch card + send via A2A

#### 4.8c: Trust Model

- Internal bots: trusted (same cluster, service token)
- External agents: require explicit allowlist in thread config
- `CreateThreadRequest` addition: `external_agents: list[str] = []`

### 4.9 Phase 2 File Summary

| File | Status | Key Changes |
|------|--------|-------------|
| `a2a_protocol/task_store.py` | **DONE** | SQLiteTaskStore (opt-in) + InMemoryTaskStore (default) |
| `a2a_protocol/discovery.py` | **DONE** | Agent Card discovery + cache, SSRF guard, HTTPS-only, allowlist |
| `a2a_protocol/server.py` | **DONE** | Auth exact path, cancel impl, rate limiter, Layer 2 idempotency (`_turn_cache`) |
| `a2a_protocol/a2a_client.py` | **DONE** | Circuit breaker, fallback chain, cancel_task, retry, auth isolation |
| `a2a_protocol/usage_logger.py` | **DONE** | Persistent SQLite + correlation fields (`thread_id`, `turn_number`, `skill_id`) |
| `routers/threads.py` | **DONE** | `_cancel_inflight_tasks()` on conclude/reject, correlation IDs persisted |
| `models/thread.py` | **DONE** | `+a2a_task_id`, `+a2a_context_id` on `ThreadTurn` |
| `main.py` | **DONE** | Wire `get_task_store()`, enhanced metrics |

---

## 5. Implementation Sequence

### Phase 1 (A2A-first transport)

```
Step 1.0  a2a_protocol/mount_registry.py   — extract mount logic (breaks circular import)
Step 1.1  a2a_protocol/a2a_client.py       — new client wrapper (imports mount_registry, NOT main)
Step 1.2  a2a_protocol/server.py           — skill routing fix (metadata-based)
Step 1.3  routers/threads.py               — swap 3 functions (LLMClient → A2AClient)
Step 1.4  main.py                          — delegate mount to mount_registry
Step 1.5  a2a_protocol/card_generator.py   — skill metadata
Step 1.6  Test: thread turn via A2A path, verify same token cost
```

### Phase 2 (production hardening) — ALL DONE

```
Step 2.1  a2a_protocol/task_store.py       — persistent task store              ✅
Step 2.2  models/thread.py                 — correlation fields                 ✅
Step 2.3  threads.py + server.py           — idempotency wiring                 ✅
Step 2.4  server.py + a2a_client.py        — cancel semantics                   ✅
Step 2.5  usage_logger.py + logging        — observability                      ✅
Step 2.6  server.py middleware             — auth + rate limiting               ✅
Step 2.7  a2a_client.py                    — circuit breaker + retry            ✅
Step 2.8  a2a_protocol/discovery.py        — external agent foundation          ✅
```

---

## 6. Rollback Strategy

| Scope | Rollback Mechanism |
|-------|-------------------|
| Phase 1 (full) | `A2A_DIRECT_LLM=false` → reverts to gateway path (unchanged code) |
| Phase 2 TaskStore | `A2A_TASK_STORE=memory` env var → falls back to `InMemoryTaskStore` |
| Phase 2 Auth | Unset `A2A_INTERNAL_TOKEN` → auth middleware skipped (dev/test mode) |
| Phase 2 External | `A2A_EXTERNAL_AGENTS=false` → disables external agent routing |
| Phase 2 Circuit Breaker | Auto-fallback: if A2A fails → direct LLMClient (transparent) |

---

## 7. Verification Checklist

### Phase 1 Complete: ✅

- [x] Thread turn with `A2A_DIRECT_LLM=true` routes through A2A server
- [x] `AgentopiaExecutor` receives `skill_id` from metadata (not keyword guessing)
- [x] Token usage per turn unchanged (~1-2K for debate, ~500 for bridge/epoch)
- [x] External A2A callers (without metadata) still work via keyword fallback
- [x] Bot auto-mounted on first thread operation
- [x] Usage logging moved to executor level (single logging point)

### Phase 2 Complete: ✅

- [x] Tasks persist via opt-in SQLiteTaskStore (`A2A_TASK_STORE=sqlite`) — `task_store.py`
- [x] Thread turn → A2A task correlation visible in logs and metrics — `models/thread.py:60`, `thread_service.py:293`, `usage_logger.py`
- [x] Idempotent: same turn resent → cached response (no duplicate LLM calls) — Thread layer (`threads.py:800`) + executor `_turn_cache` (`server.py:188`)
- [x] Thread conclude/reject → in-flight A2A tasks canceled — `_cancel_inflight_tasks()` (`threads.py:654`), called at reject (`:1004`) + conclude (`:1026`)
- [x] Usage data persists across restarts — SQLite `usage_logger.py` with correlation fields
- [x] A2A endpoints authenticated (exact path match `server.py:64`, internal token isolation)
- [x] Rate limiting prevents runaway loops — `_RateLimiter` (`server.py:118`), checked in `execute()` (`:169`)
- [x] Circuit breaker: A2A failure → automatic fallback to direct LLM — `a2a_client.py`
- [x] External agent cards discoverable and cacheable — `discovery.py` (HTTPS-only, SSRF guard, allowlist)

### Test Gate: ✅

- [x] 154/154 passed across 7 test files (0 failures)
- [x] Test isolation: module-level constants patched in all fixtures
- [x] Security: exact path match, concurrent threading, singleton reset

---

## 8. Dependencies

| Dependency | Version | Usage |
|------------|---------|-------|
| `a2a-sdk` | >=0.3.24 | Client (`ClientFactory`), Server, TaskStore interface |
| `httpx` | existing | Shared async client for A2A calls |
| `sqlite3` | stdlib | Sync SQLite for `SQLiteTaskStore` (opt-in) |

---

## 9. Key Design Decisions

### Why A2A protocol doesn't increase token cost

A2A is a transport/task-lifecycle protocol (JSON-RPC). It does not dictate:
- How much context to include in messages
- Which LLM model to call
- What system prompt to use

The current minimal context strategy (topic + last 3 turns = ~1-2K tokens) is preserved exactly as-is. Only the delivery mechanism changes: `LLMClient.create_message()` → `ClientFactory.create()` → `client.send_message()` → `AgentopiaExecutor` → `LLMClient.create_message()`.

### Why localhost hop is acceptable

Internal A2A calls go to `http://localhost:8001/a2a/{bot}` — same process, same pod. Overhead is:
- 1 HTTP roundtrip on loopback (~<1ms)
- JSON-RPC serialize/deserialize (~<1ms)
- Total: ~2ms additional latency on a 10-30s LLM call

The benefit (proper A2A compliance, single executor logging, clean separation) far outweighs the cost.

### Why not A2A for gateway path

When `A2A_DIRECT_LLM=false`, turns go through the gateway (`/v1/chat/completions`). This path provides full personality (SOUL.md, tool plugins, 50K token context) — it's a fundamentally different interaction model. A2A wrapping here adds no value since the gateway is not an A2A agent.

---

## 10. Production Readiness Tiers

This plan targets **two distinct production tiers**. Phase 1 + Phase 2 (2.1-2.5) achieves Tier 1. Phase 2 (2.6-2.8) achieves Tier 2.

### Tier 1: Single-Pod Production

Sufficient for: current Agentopia deployment (1 replica bot-config-api, controlled environment).

| Requirement | Solution | Phase |
|-------------|----------|-------|
| A2A-compliant transport | `ClientFactory` → JSON-RPC `message/send` | 1 |
| Task persistence | SQLiteTaskStore (opt-in, local disk) | 2.1 |
| Idempotency | Thread-layer turn_id dedup (2 layers) | 2.3 |
| Cancel semantics | Best-effort cancel + idempotent re-check | 2.4 |
| Observability | Persistent SQLite + correlation logging | 2.5 |
| Auth | Internal service token (single pod = trusted) | 2.6 (basic) |

**Limitations accepted**:
- SQLite single-writer — no concurrent multi-pod writes
- In-memory rate limiting — resets on restart
- No distributed lock — lease-based on local disk + threading.Lock (sufficient for single pod)

**Hard guards (must enforce)**:
- Helm values: `replicaCount: 1` for bot-config-api (document in chart README)
- Startup check: log warning if >1 replica detected via K8s downward API
- SQLite mode: verify directory exists and writable at startup (fail-fast)

### Tier 2: Multi-Pod Production (M1.8 — DONE)

Required for: horizontal scaling (>1 replica), external agent federation.

| Requirement | Solution | Status |
|-------------|----------|--------|
| Persistent task store | PostgresTaskStore (`a2a_tasks` table) | **DONE** (#74) |
| Distributed rate limiting | Redis-backed counter (`redis_backend.py`) | **DONE** (#76) |
| Cross-pod idempotency | Postgres `a2a_idempotency` table + cache write-through | **DONE** (#75) |
| Distributed locking | `ThreadLock` Protocol — local/postgres/redis backends | **DONE** (#81) |
| Prometheus metrics | Burn-rate SLO model (14.4x/6x/1x multi-window) | **DONE** (#76, #83) |
| TLS-capable deployment | Conditional TLS for Redis + Postgres via Helm values | **DONE** (#82) |
| Chaos test infrastructure | 4 K8s Job manifests (pod kill, network partition) | **DONE** (#80) |
| Migration tool | SQLite→Postgres with MD5 checksum verify + rollback | **DONE** (#79) |
| Rolling update | `maxUnavailable: 0, maxSurge: 1` + startupProbe | **DONE** (#77) |
| Operational runbook | `docs/runbook-a2a.md` with alert playbook | **DONE** (#78) |
| External agent auth | mTLS or OAuth2 + allowlist | **Deferred** (Security Hardening milestone) |
| SSRF protection | DNS/IP guard on discovery URLs | **Deferred** |

---

## 11. A2A Message Metadata Schema

All internal A2A messages from thread orchestrator MUST include metadata conforming to this schema.

### Schema v1 (`agentopia/thread-metadata/v1`)

```json
{
  "schema_version": "agentopia/thread-metadata/v1",
  "skill_id": "debate | bridge-respond | summarize",
  "source": "thread-orchestrator",
  "thread_id": "thr_abc123def456",
  "turn_number": 5,
  "turn_id": "turn-0005",
  "from_bot": "bot-alpha",
  "to_bot": "bot-beta"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schema_version` | string | yes | Schema identifier for forward compatibility |
| `skill_id` | enum | yes | Explicit skill routing — `debate`, `bridge-respond`, `summarize` |
| `source` | string | yes | Caller identity — `thread-orchestrator` for internal calls |
| `thread_id` | string | yes | Thread correlation ID |
| `turn_number` | int | conditional | Required for debate turns |
| `turn_id` | string | conditional | Required for debate turns (idempotency key) |
| `from_bot` | string | conditional | Sender bot name |
| `to_bot` | string | conditional | Target bot name |

### Placement

- **`MessageSendParams.metadata`** — params-level metadata (visible to request handler)
- **`Message.metadata`** — message-level metadata (visible to executor via `RequestContext`)

Both are set. Executor reads from `Message.metadata` (more accessible in `RequestContext`).

### Backward Compatibility

External A2A callers (without metadata) are handled by keyword-based skill fallback in `AgentopiaExecutor._identify_skill()`. No metadata = no schema enforcement.

### Schema Evolution

New fields can be added without breaking existing consumers. `schema_version` allows executor to handle different metadata formats. Unknown fields are ignored.

---

## 12. JSON-RPC Protocol Contract

### Endpoint

All A2A communication uses a **single JSON-RPC endpoint** per bot:

```
POST /a2a/{bot_name}
Content-Type: application/json
```

There is **no** `/tasks/send` or `/tasks/get` REST path. All methods are dispatched via the `method` field in the JSON-RPC request body. This is a key distinction from REST — do not "REST-ify" the protocol.

### Request: `message/send`

```json
{
  "jsonrpc": "2.0",
  "id": "thr-abc123-t5",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "messageId": "msg-uuid-here",
      "parts": [
        {
          "kind": "text",
          "text": "Topic: AI regulation\nYou are responding as: bot-beta\n..."
        }
      ],
      "metadata": {
        "schema_version": "agentopia/thread-metadata/v1",
        "skill_id": "debate",
        "source": "thread-orchestrator",
        "thread_id": "thr_abc123def456",
        "turn_number": 5
      }
    },
    "metadata": {
      "skill_id": "debate"
    }
  }
}
```

### Response: Success

```json
{
  "jsonrpc": "2.0",
  "id": "thr-abc123-t5",
  "result": {
    "kind": "task",
    "id": "task-uuid",
    "contextId": "ctx-uuid",
    "status": {
      "state": "completed",
      "message": {
        "role": "agent",
        "messageId": "resp-uuid",
        "parts": [
          {
            "kind": "text",
            "text": "AI regulation is necessary because..."
          }
        ]
      }
    },
    "artifacts": [
      {
        "artifactId": "artifact-uuid",
        "parts": [
          {
            "kind": "text",
            "text": "AI regulation is necessary because..."
          }
        ]
      }
    ]
  }
}
```

### Response: Error

```json
{
  "jsonrpc": "2.0",
  "id": "thr-abc123-t5",
  "error": {
    "code": -32603,
    "message": "Internal error: LLM call failed"
  }
}
```

### Other Methods

| Method | Usage |
|--------|-------|
| `message/send` | Primary — send task, get response |
| `message/stream` | Future — SSE streaming for long responses |
| `tasks/get` | Query task status by ID |
| `tasks/cancel` | Cancel in-flight task |

### Error Code Mapping

| JSON-RPC Code | Meaning | Thread Action |
|---------------|---------|---------------|
| `-32600` | Invalid request | `TurnStatus.failed` — bad payload |
| `-32601` | Method not found | `TurnStatus.failed` — protocol error |
| `-32602` | Invalid params | `TurnStatus.failed` — bad message format |
| `-32603` | Internal error | `TurnStatus.failed` — LLM/server error |
| `-32000` to `-32099` | Server-defined errors | Map per error code |

---

## 13. External Agent Discovery Policy

### Phase 2.8 — Security Controls

#### HTTPS-Only Requirement

External agent URLs MUST use HTTPS. HTTP URLs are rejected at discovery time.

```python
def validate_agent_url(url: str) -> bool:
    parsed = urlparse(url)
    if parsed.scheme != "https":
        raise ValueError("External agent URLs must use HTTPS")
    return True
```

#### Allowlist Enforcement

External agents must be explicitly allowlisted before they can participate in threads.

```python
# Allowlist source: environment variable or ConfigMap
A2A_EXTERNAL_ALLOWLIST = os.getenv("A2A_EXTERNAL_ALLOWLIST", "").split(",")
# Example: "https://agent-a.example.com,https://agent-b.example.com"
```

- Thread creation with external participants validates against allowlist
- Allowlist is empty by default — no external agents allowed unless explicitly configured

#### SSRF Protection

Discovery fetches (`GET /.well-known/agent-card.json`) are guarded against SSRF:

- Block private IP ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`
- Block metadata endpoints: `169.254.169.254` (AWS IMDS)
- DNS resolution validated before connection
- Timeout: 5s for card fetch (prevent slow-loris)
- Max response size: 64KB for agent card

#### Agent Card Validation

Fetched cards are validated before caching:
- Required fields: `name`, `url`, `version`, `capabilities`, `skills`
- `url` in card must match the URL used to fetch it (prevent redirect attacks)
- Card cached with 5-minute TTL (configurable via `A2A_CARD_CACHE_TTL`)

---

## 14. Conformance Testing

### Internal Conformance (Phase 1)

Verify Agentopia's A2A server handles standard A2A client requests correctly.

| Test | Method | Validates |
|------|--------|-----------|
| Send debate task | `message/send` | Skill routing, response format, artifact structure |
| Send bridge notification | `message/send` | Different skill_id, correct system prompt |
| Send epoch summary | `message/send` | Summary generation, token budget |
| Metadata routing | `message/send` + metadata | `skill_id` from metadata takes priority over keyword |
| Keyword fallback | `message/send` without metadata | Backward compat for external callers |
| Agent Card fetch | `GET /.well-known/agent-card.json` | Card schema, skills, capabilities |
| Invalid request | malformed JSON-RPC | Proper error response (-32600) |
| Unknown method | `tasks/foo` | Method not found error (-32601) |

### External Interop Conformance (Phase 2)

Verify Agentopia can communicate with a standard external A2A agent.

| Test | Description |
|------|-------------|
| Send to external agent | `AgentopiaA2AClient` → external A2A server → response parsed correctly |
| Receive from external | External client → Agentopia A2A endpoint → task processed correctly |
| Agent Card exchange | Fetch external card, validate schema, cache |
| Auth handshake | Bearer token accepted/rejected correctly |
| Error handling | External agent returns error → graceful degradation |

### Test Implementation

- Use `a2a-sdk` test utilities if available
- Minimal external A2A echo server for interop tests (responds with input text)
- CI: run conformance suite on every PR touching `a2a_protocol/`

---

## 15. Document Sync Policy

### Single Source of Truth

| Topic | Source of Truth | Other Docs Reference |
|-------|----------------|---------------------|
| A2A design & rationale | `a2a-solution-protocol.md` | Improvement plan links to it |
| Implementation status | `a2a-solution-protocol.md` Phase status | MEMORY.md mirrors |
| Improvement roadmap | `A2A-improvement-plan.md` | Solution protocol links to it |
| Runtime config (models, URLs) | Code (`llm_client.py`, Helm values) | Docs note "see code for current defaults" |
| Architecture overview | `Chatbot-architecture.md` | A2A docs reference |

### Sync Rules

1. **Code change** → update the source-of-truth doc in the same commit
2. **Status change** (phase complete, feature shipped) → update `a2a-solution-protocol.md` Phase status
3. **No contradictions**: if two docs disagree, the source-of-truth wins — fix the other doc
4. **As-built notes**: when implementation diverges from design, add `> As-built note` callout (not silent divergence)
