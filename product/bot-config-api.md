---
title: "bot-config-api (v2.0.0)"
---

# bot-config-api (v2.0.0)

LLM-powered bot provisioning service for Agentopia. Accepts free-text requirements via a web UI, generates structured bot config files (SOUL.md, USER.md, bot.yaml), and automatically deploys the bot through the full GitOps pipeline. Also serves as the A2A thread orchestrator (debate/bridge/epoch coordination via direct LLM calls through agentopia-llm-proxy).

## Architecture

```
Web UI (http://<ip>/ui)
      |
      | POST /api/v1/bots/deploy
      v
  FastAPI (deploy.py)
      |
      ├── Step 1: generate
      │     LLMGenerator → SOUL.md + USER.md + bot.yaml
      │     (OpenRouter, google/gemini-2.0-flash-001, tool calling)
      │
      ├── Step 2: k8s_resources
      │     K8sService →
      │       agentopia-soul-<bot>          (ConfigMap: SOUL.md, USER.md)
      │       agentopia-gateway-env-<bot>   (Secret: AGENTOPIA_GATEWAY_TOKEN + AGENTOPIA_RELAY_TOKEN — random 64-char hex each)
      │
      ├── Step 3: scope_pvc
      │     Ensure agentopia-shared-<scope> PVC exists (currently disabled — skipped unless scope sharing is re-enabled)
      │
      ├── Step 4: git_push
      │     GitHubService → atomic 4-file commit (SOUL.md, USER.md, bot.yaml, agentopia-bots.yaml)
      │       GitHub: always stores telegramToken=REPLACE_ME (never commit real tokens)
      │     K8sService → patch ApplicationSet element with REAL telegram token
      │       ApplicationSet is standalone (not synced from Git) → token persists permanently
      │             |
      │             v
      │        ArgoCD detects ApplicationSet change (~30s)
      │             |
      │             v
      │        Helm renders: agentopia-bot-token-<bot> Secret with REAL token
      │
      └── Step 5: pod_monitor
            Poll until pod reaches Running state (timeout 120s)
```

### Token Flow

```
UI telegram_token
      │
      ▼
add_bot_to_applicationset(telegram_token=real_token)
      │
      ▼  (ApplicationSet element)
ApplicationSet.spec.generators[0].list.elements[].telegramToken = "<real_token>"
      │
      ▼  (ArgoCD Helm sync)
Helm chart → Secret/agentopia-bot-token-<bot>.data.token = base64(<real_token>)
      │
      ▼  (pod volume mount)
/secrets/telegram/token  →  openclaw.json tokenFile reads this
      │
      ▼
Bot connects to Telegram ✓
```

**GitHub always stores `REPLACE_ME`** — the ApplicationSet K8s object holds the real token in-cluster only.

## API Reference

### POST /api/v1/bots/deploy

Start an async bot provisioning job. Returns immediately with `deploy_id` for polling.

**Request:**
```json
{
  "requirements": "I am a DevOps engineer...\n- Review K8s manifests\n- Debug CI/CD pipelines",
  "telegram_token": "1234567890:AABBccDDee...",
  "bot_name": "sonnet-devops",
  "provider": "anthropic/claude-sonnet-4-6",
  "memory_scope": "personal",
  "team": "devops",
  "role_hint": "devops"
}
```

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `requirements` | string | yes | — | Min 20 chars. The more detail, the better. |
| `telegram_token` | string | yes | — | BotFather format: `1234567890:AABBcc...` |
| `bot_name` | string | no | auto | URL-safe slug. Auto-generated from requirements if empty. |
| `provider` | string | no | `anthropic/claude-sonnet-4-6` | LLM provider for the bot |
| `memory_scope` | string | no | `personal` | Shared scope name. Empty = isolated. PVC must exist. |
| `team` | string | no | `memory_scope` | ArgoCD label and team grouping |
| `role_hint` | string | no | — | `devops`, `backend`, `frontend`, `code-assistant`, `research`, `general` |

**Response (202 Accepted):**
```json
{
  "deploy_id": "abc123...",
  "bot_name": "pending",
  "status": "pending",
  "steps": [
    {"name": "generate",      "label": "Generating bot config via LLM",                     "status": "pending"},
    {"name": "k8s_resources", "label": "Creating K8s resources (soul ConfigMap)",            "status": "pending"},
    {"name": "scope_pvc",     "label": "Ensuring shared memory PVC exists",                  "status": "pending"},
    {"name": "git_push",      "label": "Committing to GitHub + ApplicationSet (with real token)", "status": "pending"},
    {"name": "pod_monitor",   "label": "Waiting for pod to reach Running state",             "status": "pending"}
  ]
}
```

---

### GET /api/v1/bots/deploy/{deploy_id}

Poll job status. Poll every 2-3 seconds until `status` is `done` or `failed`.

**Response:**
```json
{
  "deploy_id": "abc123...",
  "bot_name": "sonnet-devops",
  "status": "done",
  "commit_sha": "a1b2c3d4...",
  "pod_status": "Running",
  "steps": [
    {"name": "generate",      "status": "done", "message": "Generated config for Sonnet DevOps"},
    {"name": "k8s_resources", "status": "done", "message": "Soul ConfigMap + gateway env Secret created"},
    {"name": "scope_pvc",     "status": "done", "message": "PVC agentopia-shared-personal: exists"},
    {"name": "git_push",      "status": "done", "message": "Committed a1b2c3d4"},
    {"name": "pod_monitor",   "status": "done", "message": "Running"}
  ]
}
```

Step statuses: `pending` | `running` | `done` | `failed` | `skipped`

Steps are skipped (not failed) when services are unavailable:
- `k8s_resources` → skipped if K8s not configured (local dev)
- `scope_pvc` → skipped if no `memory_scope` or K8s unavailable
- `git_push` → skipped if `GITHUB_TOKEN` not set

---

### GET /api/v1/bots/

List all deployed bots from ApplicationSet, augmented with live K8s pod status.

**Response:**
```json
{
  "bots": [
    {
      "bot_name": "sonnet-devops",
      "agent_name": "Sonnet DevOps Assistant",
      "provider": "anthropic/claude-sonnet-4-6",
      "memory_scope": "personal",
      "team": "personal",
      "pod_status": "Running"
    }
  ],
  "total": 1
}
```

---

### GET /api/v1/bots/{bot_name}/status

Live K8s pod status for a single bot.

```json
{"bot_name": "sonnet-devops", "pod_status": "Running", "k8s_available": true}
```

---

### GET /api/v1/bots/{bot_name}/mcp

Per-bot MCP configuration (enabled servers, allowed/denied tools, rate limits).

```json
{
  "enabled": true,
  "servers": [
    {
      "server_name": "github",
      "allowed_tools": ["search_code", "get_file_contents", "list_commits"],
      "denied_tools": ["create_issue", "create_pull_request"],
      "rate_limit": 0
    }
  ]
}
```

---

### GET /api/v1/bots/{bot_name}/mcp/credentials

List all MCP credential statuses for a bot. Never returns raw tokens.

```json
{
  "credentials": [
    {
      "bot_name": "max-sa",
      "server_name": "github",
      "secret_name": "mcp-github-token-max-sa",
      "credential_present": true,
      "created_at": "2026-03-14T15:58:11+00:00"
    }
  ]
}
```

---

### GET /api/v1/bots/{bot_name}/mcp/credentials/{server_name}

Redacted credential status for a specific bot/server pair.

```json
{
  "bot_name": "max-sa",
  "server_name": "github",
  "secret_name": "mcp-github-token-max-sa",
  "credential_present": true,
  "created_at": "2026-03-14T15:58:11+00:00"
}
```

---

### PUT /api/v1/bots/{bot_name}/mcp/credentials/{server_name}

Create or update a per-bot MCP credential. Stores as K8s Secret `mcp-{server}-token-{bot_name}`. Triggers runtime re-apply (patches Application CRD → ArgoCD selfHeal → pod rollout).

**Request:**
```json
{"token": "ghp_..."}
```

**Response:**
```json
{
  "status": "saved",
  "bot_name": "max-sa",
  "server_name": "github",
  "secret_name": "mcp-github-token-max-sa",
  "credential_present": true
}
```

---

### DELETE /api/v1/bots/{bot_name}/mcp/credentials/{server_name}

Delete a per-bot MCP credential. Removes the K8s Secret and triggers runtime re-apply.

```json
{"status": "deleted", "bot_name": "max-sa", "server_name": "github"}
```

---

### POST /api/v1/bots/generate

Generate SOUL.md/USER.md/bot.yaml only (no K8s/GitHub steps). For manual review before deploy.

**Request:** Same as `/deploy` but without `telegram_token`.

**Response:**
```json
{
  "bot_name": "sonnet-devops",
  "agent_name": "Sonnet DevOps Assistant",
  "provider": "anthropic/claude-sonnet-4-6",
  "soul_md": "# SOUL — Sonnet DevOps Assistant\n...",
  "user_md": "# USER — Canonical Profile\n...",
  "bot_yaml": "name: sonnet-devops\n...",
  "generation_model": "openai-codex/gpt-5.1",
  "warnings": [],
  "saved_to": "/output/sonnet-devops"
}
```

---

### GET /api/v1/bots/templates

List role requirement templates.

```json
{
  "templates": [
    {"name": "devops",         "description": "DevOps / SRE...", "content": "..."},
    {"name": "backend",        "description": "Backend Dev...",  "content": "..."},
    {"name": "frontend",       "description": "Frontend Dev...", "content": "..."},
    {"name": "code-assistant", "description": "Code review...",  "content": "..."},
    {"name": "research",       "description": "Research...",     "content": "..."},
    {"name": "general",        "description": "General...",      "content": "..."}
  ]
}
```

---

### GET /health

```json
{"status": "ok", "version": "2.0.0"}
```

---

### POST /api/v1/bots/{bot_name}/telegram-username

Backfill or update the Telegram @username for an existing bot (for relay name resolution).

**Request:**
```json
{"telegram_username": "openclaw01thanhbot_bot"}
```

**Response:**
```json
{"bot_name": "tech-lead-architect", "telegram_username": "openclaw01thanhbot_bot", "status": "updated"}
```

---

## Relay API

### POST /api/v1/relay/{to_bot}

Send a message from one bot to another. `to_bot` can be a K8s bot name OR a Telegram @username.

**Request:**
```json
{
  "from_bot": "sonnet-research-analyst",
  "message": "What are the security implications of blue-green deployment?",
  "context": {}
}
```

**Query params:** `?async=true` — fire-and-forget mode (skip direct delivery, write to PVC inbox queue immediately).

**Response:**
```json
{"status": "delivered", "delivered_via": "direct", "response": "<bot reply>"}
// or {"status": "queued", "delivered_via": "queue", "response": null}
```

**Delivery chain:**
1. Validate `Authorization: Bearer` token against caller bot's `AGENTOPIA_RELAY_TOKEN`
2. Resolve `to_bot` name (Telegram @username → K8s canonical name via `resolve_bot_name()`)
3. `?async=true` → write immediately to PVC inbox, return `queued`
4. Otherwise: POST to `http://agentopia-{to_bot}.agentopia.svc.cluster.local:18789/v1/chat/completions`
5. On timeout: retry up to `RELAY_MAX_RETRIES` times (default 1), then fallback to PVC inbox queue

**Env vars:**
| Variable | Default | Description |
|---|---|---|
| `RELAY_DIRECT_ENABLED` | `true` | Set `false` to always queue |
| `RELAY_MAX_RETRIES` | `1` | Retry attempts on timeout before queue fallback |
| `RELAY_INBOX_DIR` | `/openclaw-state/shared-memory/inbox` | PVC inbox queue path |

---

### GET /api/v1/relay/bots

Return all registered bots with relay-relevant metadata for peer discovery.

**Response:**
```json
[
  {"botName": "tech-lead-architect", "agentId": "agentopia-tech-lead-architect", "telegramUsername": "openclaw01thanhbot_bot"},
  {"botName": "sonnet-backend-engineer", "agentId": "agentopia-sonnet-backend-engineer", "telegramUsername": "openclawthanhbot_bot"}
]
```

---

## Thread API

Multi-turn bot brainstorm sessions. The calling bot (SA bot / orchestrator) manages the thread;
specialist bots receive turns and reply synchronously.

### POST /api/v1/threads

Create a new brainstorm thread.

**Request:**
```json
{
  "topic": "Evaluate blue-green vs canary deployment strategy",
  "participants": ["sonnet-backend-engineer", "sonnet-research-analyst"],
  "initiator": "tech-lead-architect",
  "max_turns": 20,
  "auto_checkpoint_every": 10
}
```

**Response (201):**
```json
{
  "id": "thr_3f8a2b1c...",
  "status": "active",
  "topic": "Evaluate blue-green vs canary deployment strategy",
  "initiator": "tech-lead-architect",
  "participants": ["sonnet-backend-engineer", "sonnet-research-analyst"],
  "turn_count": 0,
  "current_epoch": 1,
  "session_key": "thr_3f8a2b1c..._epoch-1",
  "max_turns": 20,
  "auto_checkpoint_every": 10,
  "created_at": "2026-03-05T10:00:00Z"
}
```

---

### POST /api/v1/threads/{thread_id}/turn

Send one turn to a target bot and receive its response synchronously.

**Request:**
```json
{
  "from_bot": "tech-lead-architect",
  "to_bot": "sonnet-backend-engineer",
  "message": "What are the infra considerations for blue-green?",
  "turn_id": "custom-turn-1"   // optional — omit to auto-generate
}
```

**Response (200):**
```json
{
  "turn_id": "turn-0001",
  "turn_number": 1,
  "status": "done",
  "delivered_via": "direct",
  "response": "<bot's reply text>"
}
```

**Error responses:**
- `401` — Bearer token does not match `from_bot`'s RELAY_TOKEN
- `404` — thread not found
- `409` — thread not active (pending_approval / closed / rejected)
- `409` — Max turns reached (auto-conclude fires in background)

**Idempotency:** sending the same `turn_id` twice returns the cached first response without re-delivering.

---

### GET /api/v1/threads/{thread_id}

Read thread metadata and all turns.

**Response (200):**
```json
{
  "id": "thr_3f8a2b1c...",
  "status": "active",
  "turn_count": 3,
  "current_epoch": 1,
  "max_turns": 20
}
```

---

### GET /api/v1/threads/{thread_id}/transcript

Full thread transcript: turns + epoch summaries.

**Response (200):**
```json
{
  "thread_id": "thr_3f8a2b1c...",
  "topic": "...",
  "turns": [
    {
      "turn_id": "turn-0001",
      "from_bot": "tech-lead-architect",
      "to_bot": "sonnet-backend-engineer",
      "message": "...",
      "response": "...",
      "status": "done",
      "turn_number": 1
    }
  ],
  "epoch_summaries": [
    {"epoch": 1, "summary": "...", "created_at": "..."}
  ]
}
```

---

### GET /api/v1/threads

List all threads (active and concluded).

---

### POST /api/v1/threads/{thread_id}/checkpoint

Manually trigger an epoch checkpoint. Generates LLM summary, rotates session key, sets status to `pending_approval`.

**Request:**
```json
{"from_bot": "tech-lead-architect"}
```

**Response (200):**
```json
{
  "thread_id": "thr_3f8a2b1c...",
  "status": "pending_approval",
  "current_epoch": 2,
  "summary": "Turn 1-10 key points: ..."
}
```

---

### POST /api/v1/threads/{thread_id}/approve

Human or SA bot approves or rejects a checkpoint.

**Request:**
```json
{"action": "approve"}          // or "reject"
```

**Response (200):**
```json
{"thread_id": "...", "status": "active"}   // or "rejected"
```

- `409` if thread is not in `pending_approval` state.

---

### POST /api/v1/threads/{thread_id}/conclude

Close the thread, generate final summary, persist to mem0 for all participants, and send Telegram bridge notification to each participant.

Only the thread `initiator` can call conclude.

**Request:**
```json
{"from_bot": "tech-lead-architect", "final_summary": "Optional override summary"}
```

**Response (200):**
```json
{
  "thread_id": "thr_3f8a2b1c...",
  "status": "closed",
  "summary": "Final agreed design: blue-green with feature flags..."
}
```

---

## Configuration

| Env Variable | Required | Default | Description |
|---|---|---|---|
| `OPENROUTER_API_KEY` | **yes** | — | OpenRouter API key for mem0 fact extraction (K8s Secret: `bot-config-api-env`) |
| `LLM_BASE_URL` | no | `http://agentopia-llm-proxy.agentopia.svc.cluster.local:18789/v1` | llm-proxy base URL for generation + A2A calls |
| `LLM_API_KEY` | no | (from `agentopia-llm-proxy-env`) | Bearer token for agentopia-llm-proxy |
| `GENERATION_MODEL` | no | `openai-codex/gpt-5.1` | LLM model for bot generation (via llm-proxy) |
| `GITHUB_TOKEN` | no | — | GitHub PAT with `repo` scope (K8s Secret: `bot-config-api-github`) |
| `GITHUB_REPO` | no | `ai-agentopia/agentopia-infra` | Target repository |
| `GITHUB_BRANCH` | no | `main` | Target branch |
| `K8S_NAMESPACE` | no | `agentopia` | Namespace for ConfigMap/Secret |
| `BOT_OUTPUT_DIR` | no | `/output` | Where to save generated files |
| `PORT` | no | `8001` | Server port |
| `CORS_ORIGINS` | no | `*` | CORS allowed origins |
| `MEM0_API_URL` | no | `http://mem0-api.agentopia.svc.cluster.local:8000` | mem0-api endpoint for memory persistence |
| **Relay** | | | |
| `RELAY_INBOX_DIR` | no | `/openclaw-state/shared-memory/inbox` | PVC inbox queue fallback directory |
| `RELAY_DIRECT_ENABLED` | no | `true` | Set `false` to always queue (bypass direct delivery) |
| `RELAY_MAX_RETRIES` | no | `1` | Retry attempts on timeout before queue fallback |
| **Thread** | | | |
| `THREADS_BASE_DIR` | no | `/openclaw-state/shared-memory/threads` | PVC thread storage root |
| `THREAD_DELIVER_RETRIES` | no | `1` | Retry attempts on turn delivery timeout |

**Feature flags** (auto-detected at startup):
- **GITHUB_ENABLED**: both `GITHUB_TOKEN` and `GITHUB_REPO` set → git push active
- **K8S_ENABLED**: `KUBERNETES_SERVICE_HOST` (in-cluster) or `KUBECONFIG` set → K8s active

---

## K8s Deployment

Included in `charts/agentopia-base`. Gets:
- **ServiceAccount** `bot-config-api` with RBAC (configmaps/secrets CRUD, pods/deployments read)
- **Service** `LoadBalancer` port 80 → pod 8001 (exposes UI externally via AWS ELB)
- Env from Secrets: `bot-config-api-env` (OPENROUTER_API_KEY), `bot-config-api-github` (GITHUB_* vars)

**Prerequisites:**
```bash
# Create K8s secrets (includes bot-config-api-env and bot-config-api-github)
OPENROUTER_API_KEY=sk-or-v1-... \
GITHUB_TOKEN_DEPLOY=ghp_... \
GITHUB_REPO_DEPLOY=ai-agentopia/agentopia-infra \
GITHUB_BRANCH_DEPLOY=main \
./scripts/create-secrets.sh

# Get UI URL after ArgoCD syncs
kubectl get svc bot-config-api -n agentopia \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

**RBAC:**
- `Role` in `agentopia` ns: ConfigMaps, Secrets, PVCs, Pods (CRUD)
- `ClusterRole` + `ClusterRoleBinding`: `get`/`patch` on `argoproj.io/applicationsets` in `argocd` ns (required for patching the agentopia-bots ApplicationSet with real telegram tokens)

**K8s resources created per bot (by bot-config-api, NOT by Helm):**
| Resource | Name | Contents |
|---|---|---|
| ConfigMap | `agentopia-soul-<bot>` | SOUL.md, USER.md |
| Secret | `agentopia-gateway-env-<bot>` | `AGENTOPIA_GATEWAY_TOKEN` (random 64-char hex), `AGENTOPIA_RELAY_TOKEN` (random 64-char hex) |

> `AGENTOPIA_RELAY_TOKEN` is K8s-stable (never touched by Vault) and serves dual purpose:
> (1) `gateway.auth.token` for `/v1/chat/completions` auth — readable by bot-config-api via `get_relay_token()`
> (2) `hooks.token` fallback (via `AGENTOPIA_GATEWAY_TOKEN` which Vault overrides at runtime)
> See [A2A-architecture.md — Token Architecture](./A2A-architecture.md) for full token flow.

**Resources:** 256Mi/512Mi memory, 100m/500m CPU.

---

## Local Development

```bash
cd bot-config-api  # in agentopia-protocol repo

# Create venv and install deps
python3 -m venv .venv
source .venv/bin/activate
pip install -r src/requirements.txt

# Run server (K8s and GitHub steps will be skipped without credentials)
OPENROUTER_API_KEY=sk-or-v1-... \
BOT_OUTPUT_DIR=$(git rev-parse --show-toplevel)/bots \
  .venv/bin/python src/main.py

# Open UI
open http://localhost:8001/ui
```

---

## Testing

```bash
cd bot-config-api  # in agentopia-protocol repo

# All 47 tests (no real API calls — all external services mocked)
.venv/bin/pytest src/tests/ -v

# Smoke test against real OpenRouter
OPENROUTER_API_KEY=sk-or-v1-... bash test-local.sh
```

**Test modules (162 tests):**
- `test_routes.py` — health, info, generate, templates
- `test_llm_generator.py` — validation, retry, enrichment
- `test_deploy_route.py` — deploy job lifecycle, all step combinations
- `test_github_service.py` — atomic commits, AppSet idempotency
- `test_k8s_service.py` — ConfigMap/Secret CRUD, gateway env secret, SSA token patch, pod status
- `test_relay_route.py` — direct delivery, queue fallback, auth, name resolution, async mode
- `test_thread_route.py` — create, turn, idempotency, checkpoint, approve/reject, conclude, transcript
- `test_thread_compaction.py` — epoch summary generation, PVC write, epoch advance
- `test_a2a_integration.py` — full thread lifecycle (turn → checkpoint → approve → conclude)

**CI gate (runs on every push affecting `bot-config-api/`):**
```bash
pytest tests/test_relay_route.py tests/test_thread_route.py \
       tests/test_thread_compaction.py tests/test_a2a_integration.py -v
```

---

## Docker Build

```bash
cd bot-config-api  # in agentopia-protocol repo
docker build --platform linux/amd64 -t ghcr.io/ai-agentopia/bot-config-api:2.0.0 .
docker push ghcr.io/ai-agentopia/bot-config-api:2.0.0
```

CI/CD: push to `main` in `agentopia-protocol` repo → `.github/workflows/build-images.yml` auto-builds + pushes `bot-config-api:latest` to ghcr.io.

---

## Workflow Bridge (wf-bridge) — Gateway Plugin

The `wf-bridge` gateway extension bridges Telegram `/wf` commands to bot-config-api's workflow API (`POST /api/v1/workflow/command`).

### Helm Chart Config (agentopia-bot)

Three sections in the bot ConfigMap (`openclaw.json`) must include wf-bridge:

1. **`plugins.allow`** — add `"wf-bridge"` to the allowlist
2. **`tools.allow`** — add `"wf_execute"` to the tool allowlist
3. **`plugins.entries.wf-bridge`** — plugin config entry:
   ```json
   {
     "enabled": true,
     "config": {
       "serviceUrl": "http://bot-config-api.<namespace>.svc.cluster.local",
       "roleKey": "",
       "timeoutMs": 15000
     }
   }
   ```

**values.yaml:**
- `wfBridgeRoleKey` (optional, default `""`) — **leave empty for normal operation.** When empty, the gateway sends empty `role_key` and the backend resolves the actor's bound role from the `role_registry` (Postgres-backed). Only set this for explicit override (e.g. manual GitOps deploy without a backend binding). **Source-of-truth for runtime role is the backend binding, NOT this Helm value.** Post-deploy role changes: call `POST /bind-actor` — no Helm/ArgoCD update needed.

### Telegram Command Registration

**Automated (default since #203):** The deploy pipeline calls Telegram Bot API `setMyCommands` after pod readiness, registering `/wf`, `/start`, and `/help` automatically. No manual BotFather step needed.

**Manual fallback (if automation fails):**

1. Open Telegram → @BotFather → `/mybots` → select bot
2. Edit Bot → Edit Commands
3. Add: `wf - Execute workflow commands (status, start, gate, assign, report)`
4. Save

**Note:** BotFather/setMyCommands registration is cosmetic (adds `/wf` to autocomplete). The actual handling is tool-based — the gateway LLM decides to call `wf_execute` when it sees `/wf` in user input.

### Rollout Verification

After merging a Helm chart update with wf-bridge config:

1. Trigger ArgoCD hard refresh (or wait for poll interval)
2. Verify ConfigMap contains `wf-bridge` and `wf_execute`: `kubectl get configmap agentopia-config-<bot> -n agentopia -o jsonpath="{.data.openclaw\.json}" | grep wf-bridge`
3. Rollout restart deployments (ConfigMap changes don't auto-restart pods): `kubectl rollout restart deployment agentopia-<bot> -n agentopia`
4. Check startup logs for `wf-bridge: registered`: `kubectl logs <pod> -n agentopia | grep wf-bridge`
5. No errors related to plugin loading

### Smoke Test (before UAT)

- Send `/wf status` to a bot in Telegram DM
- Bot should invoke `wf_execute` tool and return a response (success or "no active workflow" error from backend)
- If bot ignores `/wf` entirely, check that `wf-bridge` appears in startup logs

---

## Fresh-Bot Workflow Setup Path

### Automated Path (default since #203)

When deploying via the web UI with a **Workflow Role** selected (orchestrator/worker/reviewer), the deploy pipeline handles everything automatically:

1. **Deploy** — bot pod created via ArgoCD Application CRD with `wfBridgeRoleKey` in Helm values
2. **Actor binding** — after pod readiness, `bind_actor(bot_name, role_key)` is called automatically
3. **Telegram commands** — `setMyCommands` registers `/wf`, `/start`, `/help` via Telegram Bot API

**Identity mapping:** The runtime `actor_id` is `bot_name` (env var `AGENTOPIA_BOT_NAME`), **NOT** `agentId` (`{bot_name}-agent`). The `agentId` is only used for A2A routing.

No manual steps needed. The deploy job reports progress for each step.

### Manual Path (fallback or custom roles)

For bots deployed via manual GitOps, or when using custom roles not available in the UI dropdown:

#### Step 1 — Register a Workflow Role (if not using built-in templates)

Built-in roles available on startup: `orchestrator`, `worker`, `reviewer` (plus legacy aliases `sa-bot`, `backend-bot`, `qa-bot`).

To register a custom role:

```bash
curl -X POST http://bot-config-api:80/api/v1/workflow/register-role \
  -H 'Content-Type: application/json' \
  -d '{
    "role_key": "deploy-engineer",
    "display_name": "Deploy Engineer",
    "description": "Handles deployment and rollback tasks",
    "allowed_commands": ["/wf status", "/wf gate", "/wf report"],
    "gate_eligible": true,
    "capabilities": ["implement", "test"]
  }'
```

#### Step 2 — Bind the Bot's Actor Identity to a Role

```bash
curl -X POST http://bot-config-api:80/api/v1/workflow/bind-actor \
  -H 'Content-Type: application/json' \
  -d '{
    "actor_id": "<bot-name>",
    "role_key": "orchestrator"
  }'
```

**The `actor_id` must match `bot_name` (AGENTOPIA_BOT_NAME), NOT `agentId`.**

#### Step 3 — Telegram Command Registration (cosmetic)

See "Telegram Command Registration" section above. If deploy automation ran, this is already done.

### Durability

**Actor-role bindings are persisted to PostgreSQL** (`w0_actor_bindings` table). Bindings survive pod restarts — no re-binding required.

Built-in roles (`orchestrator`, `worker`, `reviewer`, `sa-bot`, `backend-bot`, `qa-bot`) are re-registered by `_bootstrap_roles()` on init. Custom roles are persisted to `w0_custom_roles` table.

Resolved in: [agentopia-protocol #199](https://github.com/ai-agentopia/agentopia-protocol/issues/199) (PR #200 + #201), extended by [#203](https://github.com/ai-agentopia/agentopia-protocol/issues/203)

### End-to-End Fresh-Bot Checklist

**Automated (UI deploy with workflow role):**
1. Deploy bot via web UI with Workflow Role selected → all steps automatic
2. Verify deploy job shows all 6 steps done (including `workflow_bind` + `telegram_commands`)
3. Test: send `/wf status` in Telegram DM → should get a response

**Manual (GitOps deploy):**
1. Deploy bot via GitOps (normal provisioning flow)
2. Verify wf-bridge loaded: `kubectl logs <pod> -n agentopia | grep wf-bridge`
3. Bind actor to role: `POST /bind-actor` with `actor_id=<bot_name>` + `role_key`
4. Register `/wf` via BotFather (optional, cosmetic)
5. Test: send `/wf status` in Telegram DM → should get a response
6. After pod restart: bindings are auto-restored from PostgreSQL — no action needed

---

## Cost

~$0 per bot generation — `openai-codex/gpt-5.1` via `agentopia-llm-proxy` routes to Codex OAuth (billed under OpenAI Codex subscription, not per-token API cost).
