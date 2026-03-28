---
title: "A2A Operational Runbook"
---

# A2A Operational Runbook

On-call reference for the Agentopia A2A subsystem. Everything you need at 3am.

## Quick Reference

| Component | Health Check | Logs |
|---|---|---|
| bot-config-api | `curl :8001/health` | `kubectl logs deploy/bot-config-api -n agentopia` |
| A2A endpoint | `curl :8001/a2a/{bot}/.well-known/agent-card.json` | same as bot-config-api |
| Postgres | `kubectl exec sts/postgres -n agentopia -- pg_isready` | `kubectl logs sts/postgres -n agentopia` |
| Redis | `kubectl exec sts/redis -n agentopia -- redis-cli ping` | `kubectl logs sts/redis -n agentopia` |
| LLM proxy | `curl :8001/api/v1/metrics/usage` | check `a2a_llm_latency_seconds` metric |

### Key API Endpoints

| Endpoint | Purpose |
|---|---|
| `GET /health` | Liveness + readiness (checks Postgres, Redis) |
| `GET /api/v1/metrics/usage` | LLM token usage aggregates |
| `GET /api/v1/metrics/a2a` | A2A task/error metrics |
| `GET /api/v1/a2a/bots` | List mounted A2A bots |
| `GET /api/v1/a2a/agents` | All known agents (internal + external) |
| `POST /a2a/{bot_name}` | A2A JSON-RPC endpoint (per bot) |
| `GET /api/v1/threads` | List active threads |
| `GET /api/v1/threads/{id}` | Thread detail + status |
| `GET /api/v1/threads/{id}/transcript` | Full thread transcript |
| `POST /api/v1/threads/{id}/conclude` | Force-conclude a thread |

### Valid A2A Skills

`debate`, `bridge-respond`, `summarize`, `generic` — requests with unknown skills are rejected via JSON-RPC error response (TaskState.failed), not HTTP status codes. A2A uses JSON-RPC over HTTP 200.

---

## Alert Response Procedures

### A2AHighBurnRate (14.4x burn rate — fast burn)

**Severity:** Critical

1. **Identify scope** — which bot(s) and skill(s) are failing:
   ```bash
   # Check circuit breaker and mounted bots
   curl -s :8001/api/v1/metrics/a2a | jq .
   # Returns: {"mounted_bots": [...], "circuit_breaker": {"open": bool, ...}, "task_store_type": "..."}
   # For error breakdown by bot and type, query Prometheus:
   # rate(a2a_errors_total[5m]) by (bot, error_type)
   ```
2. **Check error_type distribution:**
   - `rate_limited` — rate limit hit (see Rate Limit Tuning below)
   - `auth_failed` — A2A_INTERNAL_TOKEN mismatch (see Auth Token Rotation)
   - `timeout` — LLM or downstream bot too slow (see A2AHighLatency)
   - `llm_error` — LLM provider returning errors (check LLM proxy)
   - `validation_error` — bad input (message too large, unknown skill)
   - `internal_error` — bug, check stack traces in logs
3. **Common causes:**
   - LLM provider down or rate-limiting — check `a2a_llm_latency_seconds` for spikes
   - A2A_INTERNAL_TOKEN rotated on one side but not the other
   - Postgres connection pool exhausted (if `A2A_TASK_STORE=postgres`)
4. **Action:**
   ```bash
   # Check bot-config-api logs for error patterns
   kubectl logs deploy/bot-config-api -n agentopia --since=10m | grep -i "error\|failed\|429\|503"
   # Check circuit breaker state
   # a2a_circuit_breaker_state{bot="X"} == 1 means open
   ```

### A2AHighLatency (p95 >30s)

**Severity:** Warning

1. **Distinguish LLM latency vs infra latency:**
   ```bash
   # LLM latency: a2a_llm_latency_seconds (per bot, per provider)
   # Task latency: a2a_task_duration_seconds (includes LLM + overhead)
   # If LLM latency ~ task duration → LLM is the bottleneck
   # If task duration >> LLM latency → infra issue (DB, lock contention)
   ```
2. **LLM bottleneck:**
   - Check LLM proxy health and provider status page
   - Check if circuit breaker already flipped to fallback model
   - Verify `MODEL_DEBATE`, `MODEL_BRIDGE`, `MODEL_EPOCH` env vars point to available models
3. **Infra bottleneck:**
   - Postgres connections: check pool exhaustion (`PG_POOL_MAX` default: 10)
   - Lock contention: check if distributed lock (`A2A_LOCK_BACKEND`) is blocking
   - Pod resources: `kubectl top pod -n agentopia -l app=bot-config-api`
4. **Action:**
   ```bash
   # Check Postgres connection count
   kubectl exec sts/postgres -n agentopia -- psql -c "SELECT count(*) FROM pg_stat_activity"
   # Check for slow queries
   kubectl exec sts/postgres -n agentopia -- psql -c "SELECT pid, state, query_start, query FROM pg_stat_activity WHERE state = 'active'"
   ```

### A2ACircuitBreakerOpen (>5min)

**Severity:** Warning

The A2A client circuit breaker opens when failure rate >20% over 5min window (minimum 5 calls). It auto-resets after 30s (`A2A_CB_RESET`). If it stays open >5min, failures are persistent.

1. **Identify which bot's circuit breaker is open:**
   ```bash
   # Prometheus: a2a_circuit_breaker_state{bot="X"} == 1
   kubectl logs deploy/bot-config-api -n agentopia --since=10m | grep "circuit breaker"
   ```
2. **Check failure pattern in logs:**
   ```bash
   kubectl logs deploy/bot-config-api -n agentopia --since=15m | grep -E "A2A (send|task) failed"
   ```
3. **Common causes:**
   - Target bot's A2A endpoint unreachable (pod crashed, not mounted)
   - Persistent LLM errors (provider outage)
   - Auth token mismatch between caller and callee
4. **Action:**
   - Verify target bot pod is running: `kubectl get pods -n agentopia -l bot={name}`
   - Verify bot is mounted: `curl :8001/api/v1/a2a/bots`
   - Manual reset: restart bot-config-api pod (resets in-memory circuit breaker state)
   ```bash
   kubectl rollout restart deploy/bot-config-api -n agentopia
   ```

**Note:** The LLM client has a separate circuit breaker — if fallback rate >20% in 5min, it holds on the quality/fallback model (`MODEL_FALLBACK`). This is self-healing and not alertable.

### A2ASlowBurnRate (>0.5% error rate over 6h)

**Severity:** Info (low-urgency)

1. **Check error rate trend:**
   ```bash
   # Prometheus: use the recording rule
   # a2a:error_ratio_6h  (= rate(a2a_request_failures_total[6h]) / rate(a2a_requests_total[6h]))
   ```
2. **Pattern analysis:**
   - Steadily increasing → resource leak, connection pool drain, memory growth
   - Periodic spikes → correlate with cron jobs, traffic patterns
   - Flat elevated → chronic misconfiguration
3. **Action:**
   - Check pod memory/CPU trend over 24h
   - Check Postgres connection pool usage trend
   - Check LLM token usage for unexpected cost growth: `curl :8001/api/v1/metrics/usage`
   - Investigate root cause before error budget is exhausted

---

## Common Operations

### Circuit Breaker Reset

**A2A Client circuit breaker** (in `a2a_client.py`):
- Auto-resets after `A2A_CB_RESET` seconds (default: 30s)
- Opens when failure rate >20% (`A2A_CB_THRESHOLD`) over 5min window (`A2A_CB_WINDOW`)
- Minimum 5 calls before evaluation
- Manual reset: restart bot-config-api pod (clears in-memory state)
- Verify: `a2a_circuit_breaker_state` metric (0=closed, 1=open)

**LLM Client circuit breaker** (in `llm_client.py`):
- Same 20% threshold over 5min window
- When open: routes directly to fallback model (`MODEL_FALLBACK`)
- Self-healing: closes when fallback rate drops below threshold
- No manual reset needed — restart pod if stuck

```bash
# Check state
kubectl logs deploy/bot-config-api -n agentopia | grep "circuit breaker"
# Force reset (restarts pod)
kubectl rollout restart deploy/bot-config-api -n agentopia
```

### Rate Limit Tuning

| Setting | Default | Env Var |
|---|---|---|
| Requests per bot per minute | 120 | `A2A_RATE_LIMIT_RPM` |
| Redis failure mode | closed (reject) | `A2A_RATE_LIMIT_FAIL_MODE` |
| Cache backend | memory | `A2A_CACHE_BACKEND` |

**To change:**
1. Update Helm values (env vars in bot-config-api section)
2. Commit and push to Git
3. ArgoCD auto-syncs the change
4. Rolling restart applies new env vars

**Backend behavior:**
- `A2A_CACHE_BACKEND=memory` — per-pod rate limit (each replica has its own counter)
- `A2A_CACHE_BACKEND=redis` — global rate limit across all replicas (shared sliding window via Redis sorted sets)
- Redis failure with `fail_mode=closed` — requests rejected (safe default)
- Redis failure with `fail_mode=degraded` — falls back to per-pod in-memory limit

### Auth Token Rotation (A2A_INTERNAL_TOKEN)

**When A2A_INTERNAL_TOKEN is empty:** auth is disabled (dev/test mode). The `/.well-known/agent-card.json` endpoint is always public (A2A spec).

**Rotation steps:**
1. Generate new token:
   ```bash
   openssl rand -hex 32
   ```
2. Update the Kubernetes secret that provides `A2A_INTERNAL_TOKEN` to bot-config-api
3. Rolling restart to pick up the new secret:
   ```bash
   kubectl rollout restart deploy/bot-config-api -n agentopia
   ```
4. Zero-downtime: K8s rolling update ensures at least 1 pod is always running
5. **Both caller and callee must have the same token** — if the A2A client (`a2a_client.py`) sends requests with the old token, it will get 401s until restarted

### Postgres Operations

**Health check:**
```bash
kubectl exec sts/postgres -n agentopia -- pg_isready
kubectl exec sts/postgres -n agentopia -- psql -c "SELECT 1"
```

**Connection pool exhausted** (symptom: tasks hang, `a2a_task_duration_seconds` spikes):
```bash
# Check active connections
kubectl exec sts/postgres -n agentopia -- psql -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state"
# Current pool settings
# PG_POOL_MIN=2, PG_POOL_MAX=10 (defaults in task_store.py)
# Increase: update PG_POOL_MAX env var in Helm values
```

**Schema migration (SQLite to Postgres):**
```bash
# Check current state
kubectl exec deploy/bot-config-api -n agentopia -- python -m a2a_protocol.migration --check
# Run migration (idempotent — skips existing rows)
kubectl exec deploy/bot-config-api -n agentopia -- python -m a2a_protocol.migration --migrate
# Verify
kubectl exec deploy/bot-config-api -n agentopia -- python -m a2a_protocol.migration --verify
```

**Backup:**
```bash
kubectl exec sts/postgres -n agentopia -- pg_dump -U agentopia agentopia > backup-$(date +%Y%m%d).sql
```

**Health impact by task store backend:**
- `A2A_TASK_STORE=memory` — Postgres down has no impact on A2A tasks
- `A2A_TASK_STORE=postgres` — Postgres down degrades health to `degraded`, tasks fail
- `A2A_TASK_STORE=sqlite` — Postgres not used for tasks at all

### Redis Operations

**Health check:**
```bash
kubectl exec sts/redis -n agentopia -- redis-cli ping
# Check memory usage
kubectl exec sts/redis -n agentopia -- redis-cli info memory | grep used_memory_human
```

**Key prefixes used by A2A:**
- `a2a:rate:{bot_name}` — sliding window rate limit counters (sorted sets, TTL 120s)
- `a2a:idemp:{turn_id}` — idempotency cache entries (TTL: `A2A_IDEMPOTENCY_TTL`, default 300s)
- `thread_lock:{thread_id}` — distributed locks (if `A2A_LOCK_BACKEND=redis`)

**Memory full:**
```bash
# Check maxmemory policy
kubectl exec sts/redis -n agentopia -- redis-cli config get maxmemory-policy
# A2A keys are all TTL'd, so allkeys-lru is safe
```

**Connection lost:**
- Rate limiter: behavior depends on `A2A_RATE_LIMIT_FAIL_MODE`
  - `closed` — all A2A requests rejected until Redis recovers
  - `degraded` — falls back to in-memory per-pod rate limiting
- Idempotency cache: `has()` returns false, falls through to Postgres unique constraint as ultimate dedup
- Pod does NOT crash on Redis unavailability (startup or runtime)

**Health endpoint behavior:** Redis down shows `"status": "degraded"` in `/health` but does NOT fail readiness. Pod stays ready.

### Recency Query Source of Truth

**Symptom:** Bot answers "latest topic" correctly but gets "2nd latest" wrong — names a stale topic from memory instead of the actual 2nd-most-recent thread.

**Root cause:** mem0 stores relative temporal claims ("Second latest session is X") that are correct at write time but become stale once new sessions start. When a user asks "what's the 2nd latest topic?", mem0 semantic search returns the stale claim at high similarity score, bypassing the live thread API entirely.

**Canonical source of truth for recency questions:**

| Question type | Correct source | Wrong source |
|---|---|---|
| "What is the latest topic?" | `GET /api/v1/threads` (sorted by `created_at` desc) | mem0 recall |
| "What is the 2nd latest topic?" | `GET /api/v1/threads` index 1 | mem0 recall |
| "What did we discuss in [topic]?" | `GET /api/v1/threads/{id}/transcript` | mem0 recall |
| "What are the ongoing threads?" | `GET /api/v1/threads` (filter `status=working`) | mem0 recall |

**Fix applied (W1):** The relay plugin now detects recency intent in user prompts (EN + VI) and auto-injects live thread state via `before_agent_start` hook **before** the LLM sees the request. The injected `<live-thread-state>` block includes a PRECEDENCE RULE: live state overrides any mem0 memory about session ordering.

**Verify recency injection is working:**
```bash
# Check relay plugin logs — should see "recency-inject" entries
kubectl logs deploy/agentopia-llm-proxy -n agentopia --since=5m | grep "recency-inject"
# Expected: "relay: recency-inject: injecting N threads for prompt: ..."
```

**If recency inject is NOT firing:**
1. Check `recencyAutoInject: true` in relay plugin config (bot-config-api ConfigMap or Vault)
2. Check `AGENTOPIA_RELAY_TOKEN` env var is set in gateway pod
3. Check gateway can reach bot-config-api: `curl <bot-config-api>:8001/api/v1/threads`
4. Verify the prompt contains a recency keyword (EN: latest/recent/ongoing; VI: gần nhất/mới nhất)

**If bot still answers with stale mem0 data despite inject:**
1. Check for stale temporal memories — search mem0 for "latest session is" or "second latest"
2. Delete stale entries via mem0 admin API: `DELETE /v1/memories/{id}`
3. The `filterRelativeTemporal` flag (W2, default `true`) prevents NEW stale entries from being written, but does not clean up existing ones

**Related config flags:**

| Plugin | Flag | Default | Purpose |
|---|---|---|---|
| relay | `recencyAutoInject` | `true` | Enable live thread injection on recency intent |
| relay | `recencyTopN` | `5` | How many threads to include in injected context |
| relay | `recencyTimeoutMs` | `3000` | Timeout for live thread fetch (fail-open) |
| mem0-api | `filterRelativeTemporal` | `true` | Strip relative ordering claims from assistant messages before mem0 capture |

---

### Thread Stuck in "working"

1. **Check thread state:**
   ```bash
   curl :8001/api/v1/threads/{thread_id}
   ```
2. **Look for active A2A tasks in logs:**
   ```bash
   kubectl logs deploy/bot-config-api -n agentopia --since=10m | grep {thread_id}
   ```
3. **If stuck >5min — force conclude:**
   ```bash
   curl -X POST :8001/api/v1/threads/{thread_id}/conclude \
     -H "Content-Type: application/json" \
     -d '{"from_bot": "operator", "final_summary": "Force-concluded by on-call (stuck thread)"}'
   ```
4. **Root cause (most common):**
   - LLM timeout — check `A2A_TIMEOUT` (default: 120s) and `a2a_llm_latency_seconds`
   - Unhandled exception in turn delivery — check logs for stack trace
   - Lock contention — another turn holding the thread lock (max TTL: 60s)
   - Auto-conclude should trigger at `max_turns` — check if it fired

### LLM Provider Issues

**Model configuration (env vars):**

| Env Var | Default | Purpose |
|---|---|---|
| `A2A_LLM_BASE_URL` | `https://openrouter.ai/api/v1` | LLM API base URL |
| `A2A_LLM_API_KEY` | (from `OPENROUTER_API_KEY`) | LLM API key |
| `MODEL_DEBATE` | `google/gemini-2.0-flash-001` | Debate turn model (via OpenRouter) |
| `MODEL_BRIDGE` | `google/gemini-2.0-flash-001` | Bridge notification model |
| `MODEL_EPOCH` | `google/gemini-2.0-flash-001` | Epoch summary model |
| `MODEL_FALLBACK` | `openai/gpt-4.1-mini` | Fallback model on 429/503 |

**Token budgets (hardcoded):**
- debate_turn: 1024 max output tokens
- bridge_notify: 512
- epoch_summary: 500

**Fallback chain:**
1. Primary model gets 429 or 503 → automatic retry with `MODEL_FALLBACK`
2. If fallback rate >20% over 5min → LLM circuit breaker opens, all calls go to fallback
3. Circuit breaker self-heals when fallback rate normalizes

**Check token spend:**
```bash
curl :8001/api/v1/metrics/usage
# Returns: total_tokens, total_cost_usd, breakdown by bot and call_type
```

---

## Game-Day Drill Scenarios

### Scenario 1: Postgres Pod Kill

**Command:**
```bash
kubectl delete pod postgres-0 -n agentopia
```
**Expected behavior:**
- StatefulSet controller recreates the pod
- If `A2A_TASK_STORE=postgres`: bot-config-api `/health` reports `degraded`, new A2A tasks fail until Postgres recovers
- If `A2A_TASK_STORE=memory`: no impact on A2A task processing
- Connection pool reconnects automatically (psycopg pool lazy-opens)

**Verify after recovery:**
```bash
# Health returns ok
curl :8001/health
# New tasks succeed
curl -X POST :8001/a2a/{bot_name} -H "Content-Type: application/json" -d '...'
# No data loss (PVC retains data)
kubectl exec sts/postgres -n agentopia -- psql -c "SELECT count(*) FROM a2a_tasks"
```

### Scenario 2: Redis Unavailability

**Simulate:**
```bash
# Option A: Kill Redis pod
kubectl delete pod redis-0 -n agentopia
# Option B: Network partition (apply NetworkPolicy blocking Redis)
```
**Expected behavior (A2A_RATE_LIMIT_FAIL_MODE=closed):**
- All A2A requests rejected with rate limit error
- `a2a_errors_total{error_type="rate_limited"}` spikes
- `/health` shows Redis as `degraded` but pod stays ready

**Expected behavior (A2A_RATE_LIMIT_FAIL_MODE=degraded):**
- Falls back to in-memory per-pod rate limiting
- Warning logs: `"Redis rate-limiter error"`
- A2A requests continue to work (with per-pod limits)

**Verify after recovery:**
```bash
kubectl exec sts/redis -n agentopia -- redis-cli ping
# Rate limiting returns to global mode
# Idempotency cache rebuilds (old entries expired via TTL)
```

### Scenario 3: Traffic Spike (10x)

**Generate load:**
```bash
# Using hey or k6 against A2A endpoint
hey -n 1000 -c 50 -m POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"message/send","params":{"message":{"role":"user","parts":[{"text":"test"}]}},"id":"1"}' \
  http://bot-config-api:8001/a2a/{bot_name}
```
**Expected behavior:**
- Rate limiter kicks in at 120 RPM per bot (default)
- Excess requests get 429 / rate_limited error
- `a2a_rate_limit_remaining` gauge drops to 0
- Pod should NOT OOM — LLM calls are bounded by rate limit
- Latency degrades gracefully for allowed requests

**Verify:**
```bash
# Pod still healthy
kubectl get pods -n agentopia -l app=bot-config-api
# No OOM kills
kubectl describe pod -n agentopia -l app=bot-config-api | grep -i oom
# Rate limit metric reflects rejection
# a2a_errors_total{error_type="rate_limited"} increased
```

### Scenario 4: LLM Provider Outage

**Simulate:** Set `A2A_LLM_BASE_URL` to an unreachable endpoint, or use a revoked API key.

**Expected behavior:**
- First call to primary model fails
- Automatic fallback to `MODEL_FALLBACK`
- If fallback also fails: circuit breaker opens after 5+ failures at >20% rate
- A2A client returns `"circuit_breaker_open"` error text
- All errors recorded in `a2a_errors_total{error_type="llm_error"}`

**Recovery:** Fix API key or endpoint, circuit breaker auto-resets after 30s.

### Scenario 5: A2A Delivery Fault Matrix (#141)

Four fault injection scenarios for the relay→A2A delivery path. Each validates that `agentopia_fallback_total{reason=...}` increments correctly.

**5a. a2a_error (invalid API key):**
```bash
kubectl set env deploy/bot-config-api -n agentopia A2A_LLM_API_KEY=sk-invalid-key-for-testing
# Send a turn → LLM returns 401 → agentopia_fallback_total{reason="a2a_error"} increments
```

**5b. a2a_timeout (unreachable endpoint):**
```bash
kubectl set env deploy/bot-config-api -n agentopia \
  A2A_BASE_URL=http://192.168.254.254:9999 A2A_TIMEOUT=3 A2A_MAX_RETRIES=0
# Send a turn → 3s timeout → agentopia_fallback_total{reason="a2a_timeout"} increments
```

**5c. a2a_circuit_breaker (5+ timeouts trip CB):**
```bash
# With A2A_BASE_URL still pointing to unreachable host and A2A_MAX_RETRIES=0:
# Send 5+ turns → CB opens → agentopia_fallback_total{reason="a2a_circuit_breaker"} increments
```

**5d. bot_not_mounted (non-existent bot):**
```bash
# Send turn with to_bot="ghost-bot-999" (not deployed in K8s)
# k8s.resolve_bot_name() returns None → agentopia_fallback_total{reason="bot_not_mounted"} increments
# Response: {"fallback_reason":"bot_not_mounted","delivered_via":"failed"}
```

**Cleanup after all scenarios:**
```bash
kubectl set env deploy/bot-config-api -n agentopia A2A_BASE_URL- A2A_TIMEOUT- A2A_MAX_RETRIES- A2A_LLM_API_KEY-
# Re-set A2A_LLM_API_KEY from secret if needed
```

---

## Environment Configuration

| Env Var | Default | Purpose |
|---|---|---|
| **Task Store** | | |
| `A2A_TASK_STORE` | `memory` | Backend: `memory`, `sqlite`, `postgres` |
| `A2A_TASK_DB` | `/tmp/a2a-tasks.db` | SQLite path (only for sqlite mode) |
| `DATABASE_URL` | — | Postgres DSN (required for postgres mode) |
| `PG_POOL_MIN` | `2` | Postgres pool minimum connections |
| `PG_POOL_MAX` | `10` | Postgres pool maximum connections |
| **Cache & Rate Limiting** | | |
| `A2A_CACHE_BACKEND` | `memory` | Rate limit + idempotency backend: `memory`, `redis` |
| `REDIS_URL` | — | Redis connection string |
| `A2A_RATE_LIMIT_RPM` | `120` | Rate limit per bot per minute |
| `A2A_RATE_LIMIT_FAIL_MODE` | `closed` | Redis failure: `closed` (reject) or `degraded` (in-memory) |
| `A2A_IDEMPOTENCY_TTL` | `300` | Redis idempotency cache TTL (seconds) |
| `A2A_IDEMPOTENCY_CACHE_SIZE` | `1000` | In-memory LRU cache size (turn_ids) |
| **Distributed Locking** | | |
| `A2A_LOCK_BACKEND` | `local` | Lock backend: `local`, `postgres`, `redis` |
| **A2A Client** | | |
| `A2A_BASE_URL` | `http://localhost:8001` | Internal A2A endpoint base URL |
| `A2A_TIMEOUT` | `120` | A2A request timeout (seconds) |
| `A2A_CB_WINDOW` | `300` | Circuit breaker evaluation window (seconds) |
| `A2A_CB_THRESHOLD` | `0.20` | Circuit breaker failure rate threshold |
| `A2A_CB_RESET` | `30` | Circuit breaker auto-reset delay (seconds) |
| `A2A_MAX_RETRIES` | `2` | Max retries on timeout/503 |
| `A2A_INTERNAL_TOKEN` | (empty) | Bearer token for A2A auth (empty = auth disabled) |
| **LLM** | | |
| `A2A_LLM_BASE_URL` | `https://openrouter.ai/api/v1` | LLM provider base URL |
| `A2A_LLM_API_KEY` | (from `OPENROUTER_API_KEY`) | LLM API key |
| `MODEL_DEBATE` | `google/gemini-2.0-flash-001` | Debate turn model (via OpenRouter) |
| `MODEL_BRIDGE` | `google/gemini-2.0-flash-001` | Bridge notification model |
| `MODEL_EPOCH` | `google/gemini-2.0-flash-001` | Epoch summary model |
| `MODEL_FALLBACK` | `openai/gpt-4.1-mini` | Fallback model on 429/503 |
| **Security** | | |
| `A2A_MAX_MESSAGE_SIZE` | `32768` | Max input message size (bytes) |
| `A2A_EXTERNAL_AGENTS` | `false` | Enable external agent discovery |
| `A2A_EXTERNAL_ALLOWLIST` | (empty) | Comma-separated allowed external agent URLs |
| `A2A_CARD_CACHE_TTL` | `300` | External agent card cache TTL (seconds) |
| `A2A_CARD_FETCH_TIMEOUT` | `5.0` | External agent card fetch timeout (seconds) |
| `A2A_CARD_MAX_SIZE` | `65536` | Max agent card size (bytes) |
| **Usage Logging** | | |
| `A2A_USAGE_DB` | `/openclaw-state/a2a-usage.db` | SQLite usage log path |

---

## Prometheus Metrics Reference

| Metric | Type | Labels | Purpose |
|---|---|---|---|
| `a2a_tasks_total` | Counter | bot, skill, status | Total tasks (completed/failed/canceled) |
| `a2a_errors_total` | Counter | bot, error_type | Errors by type |
| `a2a_tokens_total` | Counter | bot, type | LLM tokens (input/output) |
| `a2a_task_duration_seconds` | Histogram | bot, skill | Task execution time |
| `a2a_llm_latency_seconds` | Histogram | bot, provider | LLM call latency |
| `a2a_rate_limit_remaining` | Gauge | bot | Remaining rate limit capacity |
| `a2a_circuit_breaker_state` | Gauge | bot | 0=closed, 1=open |
| `a2a_build` | Info | version, commit | Build metadata |
| `a2a_requests_total` | Counter | bot | Total requests (SLO denominator) |
| `a2a_request_failures_total` | Counter | bot | Failed requests (SLO numerator) |
| `a2a_idempotency_hits_total` | Counter | bot, layer | Cache/Postgres dedup hits |

**Recording rules** (computed by Prometheus):
| Rule | Window | Purpose |
|---|---|---|
| `a2a:error_ratio_5m` | 5m | Fast burn detection |
| `a2a:error_ratio_30m` | 30m | Medium burn detection |
| `a2a:error_ratio_1h` | 1h | Fast burn long-window |
| `a2a:error_ratio_6h` | 6h | Slow burn detection |

**M1.9 Delivery metrics** (relay→A2A migration tracking):

| Metric | Type | Labels | Purpose |
|---|---|---|---|
| `agentopia_delivery_total` | Counter | bot, path, skill | Deliveries by path (`a2a`/`relay_direct`/`relay_queue`) |
| `agentopia_fallback_total` | Counter | bot, reason | A2A failures (`timeout`/`circuit_breaker`/`error`/`not_mounted`/`disabled`) |
| `agentopia_delivery_duration_seconds` | Histogram | bot, path | Delivery latency by path |

**Cardinality:** bounded at ~300 series max. No unbounded labels (no task_id, user_id, etc.).

**Histogram buckets:**
- Task duration: 0.5, 1, 2, 5, 10, 20, 30, 60, 120s
- LLM latency: 0.5, 1, 2, 5, 10, 20, 30, 60s
- Delivery duration: 0.1, 0.5, 1, 2, 5, 10, 30, 60s

---

## M1.9 Adoption Alert Procedures

### A2AHighRelayFallbackRate (relay >20% for 5min)

**Severity:** Warning
**Source:** PrometheusRule `a2a-adoption-alerts`
**Fires when:** `rate(agentopia_delivery_total{path=~"relay_.*"}[5m]) / rate(agentopia_delivery_total[5m]) > 0.2`

1. **Check which bots are falling back:**
   ```bash
   curl -s :8001/api/v1/metrics/a2a-adoption | jq '.bots[] | select(.a2a_ratio < 0.8)'
   ```
2. **Identify fallback reasons:**
   ```bash
   curl -s :8001/api/v1/metrics/a2a-adoption | jq '.bots[].fallback_counts'
   # Common reasons:
   #   a2a_timeout      — A2A endpoint too slow (check LLM proxy)
   #   circuit_breaker  — CB open, persistent failures
   #   a2a_error        — A2A endpoint returning errors
   #   not_mounted      — Bot A2A endpoint not mounted
   #   disabled         — RELAY_A2A_ENABLED=false
   ```
3. **Triage by reason:**
   - `not_mounted` → Mount bot: `curl -X POST :8001/api/v1/a2a/mount/{bot_name}`
   - `disabled` → Set `RELAY_A2A_ENABLED=true` in bot-config-api env
   - `a2a_timeout` → Check LLM proxy health (see A2ADeliveryHighLatency below)
   - `circuit_breaker` → See A2ADeliveryCircuitBreakerOpen below
   - `a2a_error` → Check bot-config-api logs: `kubectl logs deploy/bot-config-api -n agentopia --since=10m | grep "A2A.*error"`
4. **Verify recovery:**
   ```bash
   # After fix, ratio should climb back above 0.8 within 5min
   curl -s :8001/api/v1/metrics/a2a-adoption | jq '.bots[] | {bot_name, a2a_ratio}'
   ```

### A2ADeliveryCircuitBreakerOpen (CB open >5min)

**Severity:** Warning
**Source:** PrometheusRule `a2a-adoption-alerts`
**Fires when:** `agentopia_fallback_total{reason="circuit_breaker"}` incrementing continuously for 5min

1. **Identify which bot's delivery circuit breaker is open:**
   ```bash
   curl -s :8001/api/v1/metrics/a2a-adoption | jq '.bots[] | select(.fallback_counts.circuit_breaker > 0)'
   ```
2. **Distinguish from A2A client CB:** This alert tracks the relay→A2A delivery path CB, not the per-bot A2A client CB (which is `a2a_circuit_breaker_state`).
3. **Root cause — same as persistent A2A failures:**
   - A2A endpoint unreachable (bot-config-api down or bot not mounted)
   - LLM proxy returning errors (check `kubectl logs deploy/agentopia-llm-proxy -n agentopia --since=10m`)
   - Network issues between gateway and bot-config-api
4. **Reset:** CB auto-resets after `A2A_CB_RESET` (30s). If staying open:
   ```bash
   # Restart bot-config-api to clear in-memory CB state
   kubectl rollout restart deploy/bot-config-api -n agentopia
   # Verify A2A is accessible
   curl -s :8001/a2a/{bot_name}/.well-known/agent-card.json | jq .name
   ```

### A2ADeliveryHighLatency (p95 >60s for 10min)

**Severity:** Warning
**Source:** PrometheusRule `a2a-adoption-alerts`
**Fires when:** `histogram_quantile(0.95, rate(agentopia_delivery_duration_seconds_bucket[5m])) > 60`

1. **Compare A2A vs relay latency:**
   ```bash
   # Prometheus queries:
   # A2A p95: histogram_quantile(0.95, rate(agentopia_delivery_duration_seconds_bucket{path="a2a"}[5m]))
   # Relay p95: histogram_quantile(0.95, rate(agentopia_delivery_duration_seconds_bucket{path="relay_direct"}[5m]))
   ```
2. **If A2A is the slow path:**
   - Check LLM latency (A2A calls go directly to OpenRouter, NOT through llm-proxy)
   - Check bot-config-api CPU/memory: `kubectl top pod -n agentopia -l app=bot-config-api`
   - Check for thread lock contention in logs
3. **If both paths are slow:**
   - Network issue between gateway pods and bot-config-api
   - Check DNS resolution: `kubectl exec deploy/bot-config-api -n agentopia -- nslookup agentopia-llm-proxy`
4. **Performance baseline (from E2E testing):**
   - Normal A2A latency: **1.1–3.2s** (includes LLM round-trip)
   - Alert threshold (60s) is 20x normal — indicates severe degradation
   - Expected p95 under load: <10s

---

## M1.9 A2A Cutover Deployment

### Overview

The relay→A2A migration is controlled by a single env var `RELAY_A2A_ENABLED` and per-bot `a2aRole` in soul ConfigMaps. This enables gradual rollout with instant rollback.

### Pre-cutover Checklist

```bash
# 1. Verify bot-config-api is running with A2A support
curl -s :8001/health | jq .

# 2. Verify A2A endpoints are mounted for target bots
curl -s :8001/api/v1/a2a/bots | jq .

# 3. Verify OpenRouter connectivity (A2A calls LLM directly, not through llm-proxy)
curl -s :8001/a2a/{any-mounted-bot}/.well-known/agent-card.json | jq .name

# 4. Check current adoption ratio (should be 0% before cutover)
curl -s :8001/api/v1/metrics/a2a-adoption | jq '.bots[] | {bot_name, a2a_ratio}'

# 5. Verify Grafana dashboard "A2A Adoption (M1.9)" is accessible
# Panels: delivery ratio pie, fallback reasons, latency timeseries, per-bot table
```

### Cutover Steps (per bot)

1. **Enable A2A for a single bot** (canary):
   ```bash
   # Set a2aRole in soul ConfigMap
   kubectl patch configmap soul-{bot-name} -n agentopia --type merge \
     -p '{"data": {"a2aRole": "participant"}}'
   # Mount A2A endpoint
   curl -X POST :8001/api/v1/a2a/mount/{bot_name}
   ```

2. **Monitor canary** (15min minimum):
   ```bash
   # Watch adoption ratio climb
   curl -s :8001/api/v1/metrics/a2a-adoption | jq '.bots[] | select(.bot_name == "{bot_name}")'
   # Expected: a2a_ratio approaching 1.0, no fallback_counts
   # Check latency is within baseline (1-3s)
   ```

3. **Roll out to remaining bots** if canary healthy:
   ```bash
   # Repeat step 1 for each bot
   # Or batch-enable via: RELAY_A2A_ENABLED=true (already default)
   ```

4. **Verify full adoption:**
   ```bash
   curl -s :8001/api/v1/metrics/a2a-adoption | jq '.bots[] | {bot_name, a2a_ratio, total_deliveries}'
   # All bots should show a2a_ratio >= 0.95
   ```

### Rollback Playbook

**Instant rollback** — disable A2A delivery, revert to pure relay:

```bash
# Option A: Disable A2A globally (fastest)
# Set RELAY_A2A_ENABLED=false — triggers automatic rolling restart
kubectl set env deploy/bot-config-api -n agentopia RELAY_A2A_ENABLED=false
# IMPORTANT: RELAY_A2A_ENABLED is read at module load (relay.py:57), NOT per-request.
# kubectl set env changes the Deployment spec → triggers rolling restart automatically.
# Wait for rollout to complete before verifying:
kubectl rollout status deploy/bot-config-api -n agentopia --timeout=90s

# Option B: Disable per-bot (surgical)
kubectl patch configmap soul-{bot-name} -n agentopia --type merge \
  -p '{"data": {"a2aRole": "none"}}'
# Restart bot-config-api to pick up ConfigMap change
kubectl rollout restart deploy/bot-config-api -n agentopia

# Option C: Unmount A2A endpoint (no config change)
# Currently no unmount API — restart bot-config-api with a2aRole=none
```

**Rollback verification:**
```bash
# Confirm relay delivery resumes
curl -s :8001/api/v1/metrics/a2a-adoption | jq '.bots[] | {bot_name, a2a_ratio}'
# a2a_ratio should drop toward 0.0 for new deliveries
# Old counter values persist until pod restart (in-process counters)
```

**Rollback triggers** (when to immediately rollback):
- A2A ratio drops below 50% with active fallback reasons
- Delivery latency p95 exceeds 30s sustained for >5min
- Circuit breaker stays open for >10min across multiple bots
- LLM proxy outage (all A2A calls fail)

---

## Performance Baseline (M1.9 E2E Results)

Baseline established from Wave 5 E2E testing on k3s single-node cluster:

| Metric | Value | Notes |
|---|---|---|
| A2A delivery latency | 1.1–3.2s | Includes LLM round-trip via OpenRouter (direct, not through llm-proxy) |
| Relay direct latency | <1s | No LLM call in relay path |
| A2A success rate | 100% (5/5) | Cross-bot: max→anna, anna→max |
| Adoption ratio | 1.0 | All deliveries via A2A when enabled |
| Task ID returned | Yes | All deliveries return valid task_id |

**Expected degradation under load:**
- 2-3x latency increase at 50+ concurrent deliveries (LLM-bound)
- Rate limiter caps at 120 RPM per bot (configurable)
- Circuit breaker opens at >20% failure rate over 5min window

**Known limitations:**
- In-process counters reset on pod restart — use Prometheus for historical data
- Single bot-config-api replica means A2A is SPOF (scale to 2+ for HA)
- LLM round-trip (OpenRouter) is the primary latency contributor (~90% of A2A delivery time)
- `a2a_timeout` fallback reason requires the A2A endpoint to be unreachable (e.g. `A2A_BASE_URL` pointing to a dead host). For internal bots (localhost), timeouts are rare since transport is in-process. The string-based timeout detection in `a2a_client.py` catches SDK-wrapped `A2AClientTimeoutError` that bypasses typed exception handlers via async generator propagation.
