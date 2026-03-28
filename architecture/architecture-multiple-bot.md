---
title: "Agentopia Multi-Bot System — Architecture Baseline"
---

# Agentopia Multi-Bot System — Architecture Baseline

> **Version:** 2.2
> **Last updated:** 2026-03-08
> **Purpose:** Baseline architecture for all Agentopia multi-bot deployments. Follow this document when setting up a new environment, adding bots, or onboarding new groups.

---

## 1. Overview

Agentopia is a self-hosted AI agent gateway that routes Telegram messages to multiple AI bots. Each bot has its own identity, model, and role, but shares a common knowledge base and memory infrastructure.

**Design goals:**
- Multiple bots, multiple groups, single user
- Cross-bot context sharing without token bloat
- New bots inherit all shared knowledge from day 1
- Memory survives session compaction
- Simple enough to operate without deep technical knowledge

---

## 2. Infrastructure

### Server requirements

| Component | Requirement |
|-----------|-------------|
| OS | Linux (Ubuntu 20.04+ recommended) |
| RAM | 8GB minimum (16GB recommended for running Qdrant + Neo4j) |
| Disk | 20GB+ for models, vector DB, and session data |
| Network | Outbound access to Telegram, Anthropic, OpenRouter APIs |
| Access | SSH (with optional ProxyJump if behind NAT) |

> **Deployment note:** All services run on the same host. For production, Qdrant and Neo4j can be separated onto dedicated instances.

### Services

| Service | Default port | Runtime | Managed by | Purpose |
|---------|-------------|---------|------------|---------|
| `agentopia-gateway` | 18789 | Node.js | systemd / K8s pod | Main AI orchestrator — routes messages to agents |
| `mem0-api` | 8000 | Python FastAPI | systemd / K8s pod | Fact extraction + semantic memory API |
| `bot-config-api` | 8001 | Python FastAPI | K8s pod | LLM-powered bot config generator (K8s only) |
| `agentopia-llm-proxy` | 18789 | Rust (Axum) | K8s pod | LLM proxy → Codex OAuth, A2A + generation calls (K8s only) |
| `Qdrant` | 6333 | Qdrant v1.16.1 | Docker / K8s StatefulSet | Vector database for semantic facts |
| `Neo4j` | 7474 / 7687 | Neo4j 5.26 | Docker / K8s StatefulSet | Graph database (entity relations, used by mem0) |
| `OpenRouter` | — | External API | — | Embeddings (`text-embedding-3-small`, 1536 dims) + mem0 fact extraction — no local server |

> All ports are localhost-only by default. The gateway (`18789`) is the only service that may be exposed for dashboard access — via SSH tunnel, not direct exposure.

### External APIs

| API | Used for | Auth method |
|-----|---------|------------|
| Telegram Bot API | Message channel | Bot token per bot (from @BotFather) |
| Anthropic API | Claude models (Opus, Sonnet) | API key |
| OpenAI Codex | gpt-5.x models | OAuth (~10 day expiry, re-auth via `openclaw onboard`) |
| OpenRouter API | LLM for mem0 fact extraction | API key |

### Infrastructure Diagram

```
┌──────────────────────────── Linux Host / K8s Pod ──────────────────────────┐
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    agentopia-gateway  :18789                         │  │
│  │                                                                      │  │
│  │   Agent: {bot-A} ── workspace-{bot-A}/                              │  │
│  │   Agent: {bot-B} ── workspace-{bot-B}/                              │  │
│  │   Agent: {bot-N} ── workspace-{bot-N}/    [scalable]                │  │
│  └───────────────────────────┬──────────────────────────────────────────┘  │
│                              │                                              │
│          ┌───────────────────┼──────────────┐                              │
│          │                   │              │                              │
│  ┌───────▼──────┐  ┌─────────▼──────┐  ┌───▼─────────────┐               │
│  │  mem0-api    │  │ Qdrant  :6333  │  │ Neo4j :7474/7687│               │
│  │  :8000       │◄─►│ (vector DB)    │  │ (graph / rels)  │               │
│  │  (FastAPI)   │  │                │  │                 │               │
│  └──────────────┘  └────────────────┘  └─────────────────┘               │
│                                                                             │
│  Embeddings: OpenRouter API (text-embedding-3-small, 1536 dims) — external │
└─────────────────────────────────────────────────────────────────────────────┘
          │                     │                    │
    Telegram API          Anthropic API        OpenRouter API
    (1 token per bot)     Claude models        embeddings + mem0 LLM
                               │
                         OpenAI Codex OAuth
                         (via agentopia-llm-proxy, for gpt-5.x models)
```

> **K8s deployment:** In Kubernetes, each bot runs as a separate pod with its own EBS PVC (StorageClass `ebs-gp3`, RWO). Scope-based shared-memory PVCs are currently disabled (no `efs.fileSystemId` configured). `bot-config-api` is deployed as a K8s pod and is the only way to generate bot config files in K8s (it calls Codex via `agentopia-llm-proxy`, no local CLI needed). See `docs/openclaw-k8s.md` for K8s architecture.

---

## 3. File System Layout

### Root structure

```
~/.openclaw/
├── openclaw.json                      # Master config (gateway, agents, plugins)
├── extensions/
│   └── mem0-api/
│       ├── index.ts                   # Custom memory plugin (patched v1.1)
│       ├── package.json
│       └── openclaw.plugin.json
├── shared-memory/                     # L2 Knowledge: cross-bot shared docs
│   ├── USER.md                        # Canonical user profile (single source of truth)
│   ├── architecture.md                # Technical architecture decisions
│   ├── decisions.md                   # ADR (Architecture Decision Records)
│   ├── tasks.md                       # Project status board (project-level only)
│   ├── PROTOCOLS.md                   # Team rules + memory architecture
│   └── groups/                        # Per-group and per-topic role/context definitions
│       ├── {chatId}.md                # Group-wide role/context
│       └── {chatId}_{topicId}.md      # Topic-specific role/context (overrides group file)
├── workspace-{bot-A}/                 # Bot A workspace
├── workspace-{bot-B}/                 # Bot B workspace
├── workspace-{bot-N}/                 # Additional bots
└── agents/
    ├── {bot-A}/sessions/*.jsonl       # Session transcripts
    ├── {bot-B}/sessions/*.jsonl
    └── {bot-N}/sessions/*.jsonl
```

### Per-bot workspace structure

```
workspace-{bot}/
├── SOUL.md              # L1: Bot identity, role, communication rules, memory rules
├── IDENTITY.md          # L1: Bot name, emoji, vibe (set at bootstrap, never reset)
├── AGENTS.md            # Team directory (same content across all bots)
├── USER.md              # L2: User quick card (auto-loaded at every session start)
├── TOOLS.md             # Tool conventions
├── HEARTBEAT.md         # Periodic background tasks (optional)
├── MEMORY.md            # L3: Curated long-term memory (DM sessions only)
└── memory/
    └── YYYY-MM-DD.md    # L3: Daily log (today + yesterday auto-read)
```

---

## 4. Memory Architecture (3 Layers)

```
┌─────────────────────────────────────────────────────────────────┐
│  L1: IDENTITY  (per-bot, static)                                │
│                                                                 │
│  SOUL.md     → Role, personality, team rules, memory rules      │
│  IDENTITY.md → Name, emoji (set once at bootstrap)              │
│                                                                 │
│  Loaded: every session start (auto, bootstrapMaxChars: 50,000)  │
│  Updated by: Admin only — bots do not self-modify these files   │
├─────────────────────────────────────────────────────────────────┤
│  L2: KNOWLEDGE  (shared across all bots, persistent)            │
│                                                                 │
│  [AUTO] mem0 Qdrant                                             │
│  ├── userId: {deployment_id}  (single scope, all bots)          │
│  ├── autoRecall: top 5 facts injected BEFORE each response      │
│  │   → timeout: 3500ms (skip if slow, never block response)     │
│  ├── autoCapture: extract facts AFTER each response             │
│  │   → captureOnFailure: true (capture even on error/timeout)   │
│  └── Pipeline: messages → OpenRouter LLM → facts → Qdrant      │
│                                                                 │
│  [ON-DEMAND] shared-memory/                                     │
│  ├── Indexed by: built-in memorySearch (OpenRouter → SQLite)     │
│  ├── Accessed via: memory_search() tool call by bot             │
│  ├── Token cost: 0 unless bot explicitly searches               │
│  └── Contains: USER.md, architecture, decisions, tasks          │
│                                                                 │
│  USER.md (bootstrap anchor — per-bot workspace)                 │
│  ├── Minimal quick card: name, timezone, language, key prefs    │
│  ├── Auto-loaded at session start (no search needed)            │
│  └── Full profile: shared-memory/USER.md (via memory_search)   │
├─────────────────────────────────────────────────────────────────┤
│  L3: SESSION  (per-bot, ephemeral, rolling 30 days)             │
│                                                                 │
│  memory/YYYY-MM-DD.md                                           │
│  ├── Auto-read: today + yesterday at every session start        │
│  ├── Written by bot: BEFORE compact (memoryFlush trigger)       │
│  └── Format: topics as headings, not raw timestamp log          │
│                                                                 │
│  MEMORY.md  (curated, DM only)                                  │
│  ├── Distilled long-term facts from private conversations       │
│  ├── NOT loaded in group chat (OpenClaw security recommendation)│
│  └── Bot writes here in DM session for high-value durable facts │
└─────────────────────────────────────────────────────────────────┘
```

### What survives session compaction

| Data | Survives | Source layer |
|------|----------|-------------|
| User name, timezone, language | ✅ Always | L1 (USER.md workspace) |
| User preferences, active projects | ✅ Always | L2 (shared-memory/USER.md) |
| Bot identity and role | ✅ Always | L1 (SOUL.md) |
| Current project status | ✅ if bot reads at session start | L2 (shared-memory/tasks.md) |
| Project architecture / decisions | ✅ Always | L2 (shared-memory/) |
| Conversation facts (auto-extracted) | ✅ Usually | L2 (mem0 Qdrant) |
| Daily summaries | ✅ Always | L3 (memory/YYYY-MM-DD.md) |
| Curated long-term facts | ✅ Always (DM) | L3 (MEMORY.md) |
| In-progress discussion context | ❌ Lost on compact | Must flush to L3 before compact |

---

## 5. Request Data Flow

```
User sends message in Telegram
            │
            ▼
    Telegram Bot API
            │
            ▼
    agentopia-gateway
    → match mention pattern → route to correct agent
            │
            ▼
┌─── BEFORE RESPONSE ─────────────────────────────────────────┐
│                                                             │
│  1. Load workspace files (L1 bootstrap)                     │
│     SOUL.md + IDENTITY.md + AGENTS.md + USER.md             │
│                                                             │
│  2. Load L3 daily logs                                      │
│     memory/today.md + memory/yesterday.md                   │
│                                                             │
│  3. autoRecall — L2 mem0  (max 3.5s, non-blocking)          │
│     user message → OpenRouter embed → Qdrant search         │
│     → inject top 5 relevant facts into context              │
│     → if timeout: skip silently, proceed without memories   │
│                                                             │
│  4. memorySearch — L2 shared-memory  (on-demand only)       │
│     only if bot explicitly calls memory_search() tool       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
    LLM API call (Anthropic / OpenAI / OpenRouter)
            │
            ▼
    Streaming response → Telegram
            │
            ▼
┌─── AFTER RESPONSE ──────────────────────────────────────────┐
│                                                             │
│  5. autoCapture — L2 mem0                                   │
│     last 10 messages → OpenRouter LLM extract facts         │
│     → store in Qdrant                                       │
│     → runs even on failure (captureOnFailure: true)         │
│                                                             │
│  6. Near compaction — L3 memoryFlush                        │
│     silent agentic turn triggers automatically              │
│     → bot writes key context to memory/YYYY-MM-DD.md        │
│     → session is then compacted                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Bot Configuration

### Example bot roster (customize per deployment)

| Bot name | Agent ID | Recommended model | Fallback | Role |
|---------|---------|------------------|---------|------|
| Architect | `architect` | `anthropic/claude-opus-4-6` | `claude-sonnet-4-6` | System design & review |
| Developer | `developer` | `openai-codex/gpt-5.2` | `claude-sonnet-4-6` | Implementation |
| DevOps | `devops` | `anthropic/claude-sonnet-4-6` | — | Infrastructure & ops |
| Analyst | `analyst` | `anthropic/claude-sonnet-4-6` | — | Research & data |

> Model choice is flexible. Any model supported by openclaw can be assigned. Recommended: use `claude-sonnet-4-6` as the global fallback for all bots.

### Model fallback chain

```
Agent primary model fails
        │
        ▼
Agent explicit fallbacks (if configured in openclaw.json)
        │
        ▼
Global default (agents.defaults.model.primary)
Recommended: anthropic/claude-sonnet-4-6
```

> **OpenAI Codex OAuth note:** If using gpt-5.x via Codex OAuth, the token expires approximately every 10 days. Re-authenticate with: `openclaw onboard --auth-choice openai-codex` → Manual mode → No (skip channels).

### Topic-level requireMention override

By default, all bots require `@mention` in groups. To make a bot auto-respond in a specific Telegram topic (forum thread) without mention:

```json
// channels.telegram.accounts.{bot-account}.groups
"groups": {
  "*": {
    "requireMention": true          // default: all groups require mention
  },
  "{groupId}": {
    "requireMention": true,         // group-level still requires mention
    "topics": {
      "{topicId}": {
        "requireMention": false     // topic-specific: no mention needed
      }
    }
  }
}
```

> `topicId` = `message_thread_id` from Telegram API, matches the `_N` suffix in the Telegram Web URL (`#{groupId}_{topicId}`).

### openclaw.json — agents.defaults (applied to all bots automatically)

```json
"agents": {
  "defaults": {
    "model": { "primary": "anthropic/claude-sonnet-4-6" },
    "bootstrapMaxChars": 50000,
    "contextTokens": 200000,
    "memorySearch": {
      "extraPaths": ["~/.openclaw/shared-memory"],
      "provider": "openai",
      "remote": { "baseUrl": "https://openrouter.ai/api/v1", "apiKey": "${OPENROUTER_API_KEY}" },
      "model": "openai/text-embedding-3-small",
      "query": { "hybrid": { "enabled": true, "vectorWeight": 0.7, "textWeight": 0.3 } }
    },
    "contextPruning": {
      "mode": "cache-ttl",
      "ttl": "1h",
      "keepLastAssistants": 3,
      "softTrimRatio": 0.3,
      "hardClearRatio": 0.5
    },
    "compaction": {
      "mode": "safeguard",
      "reserveTokensFloor": 24000,
      "memoryFlush": { "enabled": true, "softThresholdTokens": 6000 }
    },
    "heartbeat": { "every": "1h" }
  }
}
```

---

## 7. mem0-api Plugin (Custom)

**Location:** `~/.openclaw/extensions/mem0-api/index.ts`
**Version:** 1.1.0 (patched from default)

### Plugin config (`openclaw.json` → `plugins.entries.mem0-api.config`)

```json
{
  "apiUrl": "http://localhost:8000",
  "userId": "{deployment_id}",
  "autoCapture": true,
  "autoRecall": true,
  "topK": 5,
  "searchThreshold": 0.3
}
```

> `{deployment_id}` — choose a unique identifier for your deployment (e.g., `openclaw_myteam`). All bots share this single userId. Changing it creates a fresh memory store.

### Patched behaviors (v1.1.0)

These are hardcoded as defaults in `index.ts` — **not** in `openclaw.json` (config schema rejects unknown properties):

| Behavior | Value | Reason |
|----------|-------|--------|
| `recallTimeoutMs` | 3500ms (K8s: 8000ms) | Prevent blocking response if OpenRouter is slow |
| `captureOnFailure` | true | Capture conversation context even when agent times out or errors |

### mem0 pipeline

```
autoCapture (after each response):
  last 10 messages
  → mem0-api :8000 /v1/memories
  → Python mem0 library
  → OpenRouter LLM  ── extract facts
  → OpenRouter text-embedding-3-small  ── embed facts (1536 dims)
  → Qdrant store

autoRecall (before each response, max 3.5s):
  user message
  → mem0-api :8000 /v1/memories/search
  → OpenRouter embed query (text-embedding-3-small)
  → Qdrant hybrid search (vector 70% + text 30%)
  → return top 5 facts (score ≥ 0.3)
  → inject as <relevant-memories> block into context
```

### Extraction model recommendation

| Model | Quality | Cost (input/output per M) | Notes |
|-------|---------|--------------------------|-------|
| `meta-llama/llama-3.1-8b-instruct` | ⭐⭐ | $0.02 / $0.05 | Cheapest. Fast, extracts surface-level facts |
| `google/gemini-2.0-flash-001` | ⭐⭐⭐⭐ | $0.10 / $0.40 | **Recommended.** Excellent at structured extraction, reliable, no rate limits |
| `meta-llama/llama-3.3-70b-instruct` | ⭐⭐⭐⭐ | $0.10 / $0.32 | Good alternative to Gemini Flash |

> **⚠️ DO NOT use `:free` variants** (e.g. `meta-llama/llama-3.3-70b-instruct:free`) — free tier is limited to 8 RPM on a shared pool. Under any real usage load, both `autoCapture` and `autoRecall` will hit 429 errors, completely disabling the mem0 pipeline.

To change model: edit `~/.openclaw/openclaw-mem0/.env` → `MEM0_LLM_MODEL=<model-id>` → `systemctl restart mem0-api`.

---

## 8. SOUL.md — Required Memory Rules Section

Every bot's `SOUL.md` **must** include a `## Memory Rules` section. This ensures consistent behavior across all bots and prevents context loss after compaction.

**Template to include in every SOUL.md:**

```markdown
## Memory Rules — CRITICAL

### User Profile
- Canonical source: ~/.openclaw/shared-memory/USER.md
- Local copy: USER.md (this workspace) — quick card only
- ❌ NEVER guess user info — check USER.md FIRST, then memory_search("user profile")
- ✅ If you learn new info about the user → update shared-memory/USER.md + workspace USER.md

### Project Status Rule — CRITICAL
- **`shared-memory/tasks.md` is the single source of truth for all project status**
- At the START of each new session (especially after `/compact`): **read `shared-memory/tasks.md`** to get current project context — do NOT rely on compact summary which is per-bot and may be stale
- When project status changes (start / pause / resume / complete) → **update `shared-memory/tasks.md` IMMEDIATELY**, before executing

### Group / Topic Context Rule
Context files loaded in priority order — more specific overrides more general:
1. `shared-memory/groups/{chatId}_{topicId}.md` — **topic-specific** (highest priority)
2. `shared-memory/groups/{chatId}.md` — **group-wide** fallback

**At session start:**
- Check topic-specific file first → if exists, use it
- Else check group file → if exists, use it
- Else: operate from SOUL.md default role → ask user to confirm scope → write the appropriate file

**When user assigns role/scope in a topic:** write `shared-memory/groups/{chatId}_{topicId}.md` immediately
**When user assigns role for whole group:** write `shared-memory/groups/{chatId}.md`
- tasks.md is for **project-level status only** — group/topic role lives in `groups/` files

### Project Context
- Architecture decisions → shared-memory/architecture.md
- Task assignments → shared-memory/tasks.md (project-level)
- Group roles → shared-memory/groups/{chatId}.md (group-wide)
- Topic roles → shared-memory/groups/{chatId}_{topicId}.md (topic-specific)
- ADR log → shared-memory/decisions.md
- Update BEFORE executing — other bots read these files

### Before Compact / End of Session — Flush Aggressively
The compact summary size = your ongoing token cost. Write more now → smaller summary → cheaper every session.

**Flush to `memory/YYYY-MM-DD.md` BEFORE `/compact`:**
- Every decision made (architecture, tool choice, approach)
- Any new facts about the user or their projects
- Current task status + next steps
- Any context the next session will need

**Rule of thumb:** If the compact summary will need to mention it, write it to memory first. Then compact can just say "see memory/YYYY-MM-DD.md".

### What Survives Compact

| Data | Survives | Source |
|------|----------|--------|
| User identity | ✅ | USER.md |
| Bot identity | ✅ | SOUL.md, IDENTITY.md |
| Current project status | ✅ if read at session start | shared-memory/tasks.md |
| Project context | ✅ | shared-memory/ |
| Conversation facts | ✅ auto | mem0 Qdrant |
| Daily summaries | ✅ | memory/YYYY-MM-DD.md |
| In-progress discussion | ❌ | Write to memory/ before compact |
```

---

## 9. New Bot Pipeline

### Prerequisites
- Telegram bot token (create via @BotFather)
- Model ID (e.g. `anthropic/claude-sonnet-4-6`, `openai-codex/gpt-5.3-codex`)
- Free-text requirements (what the bot does, who uses it — no template editing needed)

### Step 1 — Generate bot config files (bot-config-api)

End-users write plain-text requirements and call the LLM-powered generator to produce SOUL.md, USER.md, and bot.yaml:

```bash
# Option A: Use the templates endpoint to get a starting point
curl http://localhost:8001/api/v1/bots/templates | jq '.templates[] | select(.name=="devops") | .content'

# Option B: Write requirements directly
curl -X POST http://localhost:8001/api/v1/bots/generate \
  -H "Content-Type: application/json" \
  -d '{
    "requirements": "I am a DevOps engineer at a financial company.\nBot purpose: review K8s manifests.\nLanguage: Vietnamese with English technical terms.",
    "bot_name": "sonnet-devops",
    "provider": "anthropic/claude-sonnet-4-6",
    "memory_scope": "personal",
    "role_hint": "devops"
  }' > generated.json
```

Returns: `soul_md`, `user_md`, `bot_yaml` — ready to review.

**Review before provisioning:**
```bash
cat generated.json | jq -r '.soul_md'
cat generated.json | jq -r '.user_md'
cat generated.json | jq -r '.bot_yaml'
```

**Save to bots/ directory:**
```bash
BOT_NAME=$(cat generated.json | jq -r '.bot_name')
mkdir -p bots/$BOT_NAME
cat generated.json | jq -r '.soul_md' > bots/$BOT_NAME/SOUL.md
cat generated.json | jq -r '.user_md' > bots/$BOT_NAME/USER.md
cat generated.json | jq -r '.bot_yaml' > bots/$BOT_NAME/bot.yaml
```

> See [docs/bot-config-api.md](bot-config-api.md) for full API reference, templates, and local dev setup.

### Step 2 — Provision (K8s)

```bash
./scripts/provision-bot.sh bots/$BOT_NAME/
```

For on-premise (server36), see steps 3–7 below.

---

### On-Premise Steps (server36 only)

### Step 3 — Add agent (OpenClaw built-in)

```bash
openclaw agents add {bot-name} \
  --model {model-id} \
  --workspace ~/.openclaw/workspace-{bot-name}
```

This creates:
- Agent entry in `openclaw.json`
- `workspace-{bot-name}/` seeded with blank templates
- Automatically inherits all `agents.defaults` (mem0, extraPaths, compaction, heartbeat)

### Step 4 — Copy generated config files

```bash
cp bots/{bot-name}/SOUL.md ~/.openclaw/workspace-{bot-name}/SOUL.md
cp bots/{bot-name}/USER.md ~/.openclaw/workspace-{bot-name}/USER.md
```

### Step 5 — Bind Telegram token

Edit `openclaw.json` → `channels.telegram.accounts`:

```json
"{bot-account-id}": {
  "enabled": true,
  "dmPolicy": "pairing",
  "botToken": "{telegram-bot-token}",
  "groups": { "*": { "requireMention": true } },
  "groupAllowFrom": ["*"],
  "groupPolicy": "allowlist",
  "streamMode": "partial"
}
```

Edit `openclaw.json` → `bindings`:

```json
{
  "agentId": "{bot-name}",
  "match": { "channel": "telegram", "accountId": "{bot-account-id}" }
}
```

### Step 6 — Restart gateway

```bash
XDG_RUNTIME_DIR=/run/user/$(id -u) systemctl --user restart openclaw-gateway
```

### Step 7 — Bootstrap

Send a first message to the new bot in Telegram (DM).
Bot runs through `BOOTSTRAP.md` ritual:
1. Picks identity, names itself → writes `IDENTITY.md`
2. Reviews user info (already pre-filled from USER.md)
3. Completes → **delete `BOOTSTRAP.md`**

**Result:** New bot is live. L2 knowledge (mem0 shared facts + shared-memory) available from day 1 — no manual onboarding of past context needed.

---

## 10. Operational Commands

### Gateway management

```bash
# Status
XDG_RUNTIME_DIR=/run/user/$(id -u) systemctl --user status openclaw-gateway

# Restart
XDG_RUNTIME_DIR=/run/user/$(id -u) systemctl --user restart openclaw-gateway

# Logs (live)
XDG_RUNTIME_DIR=/run/user/$(id -u) journalctl --user -u openclaw-gateway -f
```

### mem0 API

```bash
# Service status
systemctl status mem0-api

# Get all stored memories
curl -s -X POST http://localhost:8000/v1/memories/get_all \
  -H 'Content-Type: application/json' \
  -d '{"user_id": "{deployment_id}"}'

# Search memories
curl -s -X POST http://localhost:8000/v1/memories/search \
  -H 'Content-Type: application/json' \
  -d '{"query": "user preferences", "user_id": "{deployment_id}", "limit": 5}'
```

### OpenClaw CLI

```bash
# List configured agents
openclaw agents list

# Memory index status
openclaw memory status

# Force re-index memory files
openclaw memory index

# Open dashboard (requires SSH tunnel: ssh -fNL 18789:127.0.0.1:18789 {host})
openclaw dashboard
```

### Session management

```bash
# Check session sizes per agent
ls -lh ~/.openclaw/agents/*/sessions/*.jsonl

# Manual compact (send in Telegram chat)
/compact

# Monitor session size (if a bot feels slow, check this first)
du -sh ~/.openclaw/agents/*/sessions/
```

---

## 11. Multi-Group Scalability

Current architecture uses a single `userId` for mem0, sufficient for one user across multiple groups. Group-specific role and context is handled via per-group context files (implemented).

### shared-memory directory structure (current)

```
shared-memory/
├── USER.md                          # Cross-group: canonical user profile
├── architecture.md
├── decisions.md
├── tasks.md                         # Project-level status (not group/topic-specific)
├── PROTOCOLS.md
└── groups/                          # Per-group and per-topic role/context (implemented)
    ├── {chatId}.md                  # Group-wide role/context for each Telegram group
    └── {chatId}_{topicId}.md        # Topic-specific role/context (overrides group file)
```

### Context file priority

```
{chatId}_{topicId}.md   ← highest priority (topic-specific role)
        │ overrides
{chatId}.md             ← group-wide fallback
        │ falls back to
SOUL.md default         ← if neither file exists
```

### Adding a bot to a new group or topic

**New group (no topic):**
1. Bot receives first message → checks `shared-memory/groups/{chatId}.md`
2. If missing: operates from SOUL.md default → asks user to confirm group role
3. User confirms → bot writes `shared-memory/groups/{chatId}.md`
4. All subsequent sessions in that group auto-load the group context

**New topic within existing group:**
1. Bot receives first message in topic → checks `shared-memory/groups/{chatId}_{topicId}.md`
2. If missing: falls back to `{chatId}.md` → then SOUL.md default
3. User assigns topic-specific role → bot writes `shared-memory/groups/{chatId}_{topicId}.md`
4. Topic context overrides group context for all sessions in that topic

**Pre-creating a context file (admin shortcut):**
- Admin can create the file directly before the bot's first message
- Example: create `{chatId}_{topicId}.md` with role/rules → bot loads it on first message in that topic

### Scope-Based Sharing (Design — K8s only, NOT currently active)

> **Note:** Scope-based sharing is currently disabled. No `efs.fileSystemId` is set in values.yaml. All bots are isolated — each bot has its own EBS PVC (`ebs-gp3`, RWO) and its own mem0 userId. The scope PVC design below describes the intended architecture for when sharing is re-enabled.

In K8s deployments, shared-memory would use **named scopes** as the isolation boundary:

- Bots in the SAME scope share one shared PVC + one mem0 userId
- Bots in DIFFERENT scopes have fully isolated PVCs and mem0 stores
- Bots WITHOUT a scope have no shared memory at all

| Scenario | Shared PVC | mem0 userId | Sees |
|---|---|---|---|
| Bot A + Bot B in scope "alpha" | agentopia-shared-alpha | scope-alpha | Same files, same memories |
| Bot C in scope "beta" | agentopia-shared-beta | scope-beta | Only its own scope |
| Bot D (no scope) | (none) | bot-bot-d | Only its own workspace |

Scripts:
- `scripts/create-scope.sh --scope <name>` — create scope PVC
- `scripts/seed-scope.sh --scope <name>` — seed with initial content
- `scripts/add-group.sh --scope <name> --group-id <id>` — add group context
- `scripts/deploy-bot.sh --memory-scope <name>` — deploy bot into scope

> **On-premise (server36):** Scope-based isolation is not yet implemented on-premise. The original flat `shared-memory/` directory structure still applies. Per-group project isolation (separate `architecture.md`, `tasks.md` per group via subdirectories) remains a future enhancement for on-premise.

### mem0 multi-scope

Scope-based userId is now implemented at the K8s deployment level. Each scope gets its own mem0 userId (`scope-{name}`), configured via `openclaw.json` config variants per deployment. This provides full memory isolation between scopes without any plugin patch.

The per-request dynamic userId patch (setting userId dynamically in `index.ts` based on the incoming group/scope context) is still not implemented but is less critical now that scope isolation is handled at the deployment layer.

---

## 12. Known Constraints

| Constraint | Impact | Mitigation |
|-----------|--------|-----------|
| OpenAI Codex OAuth expiry | gpt-5.x bots stop working after ~10 days | Re-auth: `openclaw onboard --auth-choice openai-codex` |
| mem0 recall latency | OpenRouter embedding API round-trip ~0.3-0.5s — well within 3.5s timeout | Plugin capped at 3.5s (K8s: 8s); bot proceeds without memories if exceeded. **Timeout is rare in practice.** |
| OpenRouter `:free` model rate limit | Free tier capped at 8 RPM shared pool → 429 errors → mem0 pipeline fully disabled | **Never use `:free` models for mem0 extraction.** Use a paid model (e.g. `google/gemini-2.0-flash-001`) |
| MEMORY.md not loaded in groups | Curated memory unavailable in group chat | mem0 Qdrant (L2) compensates — same facts available via autoRecall |
| Large sessions slow responses | Sessions >1MB cause noticeable latency | Send `/compact` in chat when bot feels slow; memoryFlush auto-triggers near limit |
| `/compact` perceived as memory loss | Bot may not recall in-session context after compact | Architecture is designed for this: L1 reloads from files, L2 mem0 persists, L3 daily log written before compact |
| Single mem0 scope | All bots and groups share one memory store | Mitigated in K8s via scope-based userId (`scope-{name}`). On-premise (server36) still uses single userId. |
| Per-bot compact summary divergence | After `/compact`, each bot recalls different project context because compact summary is per-bot and per-conversation | SOUL.md rule: at session start, read `shared-memory/tasks.md` (project status) and `shared-memory/groups/{chatId}[_{topicId}].md` (group/topic role) — do NOT rely on compact summary for project state |
| 409 Conflict on gateway restart | When gateway restarts, old polling session and new session compete briefly → some messages near the restart window may be missed | Expected behavior; conflict auto-resolves within ~60s. Avoid sending critical messages immediately after a restart. |
| Gateway runs as root | AI agent has full filesystem read access including credential files (`~/.aws/credentials`, SSH keys, etc.) | See Section 13 — Security for mitigation options. |

---

## 13. Security Considerations

### Current risk profile

OpenClaw gateway runs as `root` by default. This means the AI agent can read any file on the system — including credential files — even if the bot's SOUL.md instructs it not to share them in chat. The LLM model itself still processes the secret values in its context window.

**Attack surface:**
- Direct file read via `exec`/`bash` tools (`cat ~/.aws/credentials`)
- `read_file` tool on any system path
- Environment variable inspection (`env`, `printenv`)

### Security layers — from verified OpenClaw source

The following options are verified from `types.tools.d.ts`:

#### Layer 1 — OpenClaw native (no external dependency)

**`tools.fs.workspaceOnly: true`** — Blocks `read_file`, `write_file`, `edit_file` tools outside the agent's workspace directory:

```json
// In agent config or agents.defaults
"tools": {
  "fs": { "workspaceOnly": true }
}
```

> Does NOT block `exec`/`bash` — the agent can still run shell commands.

**`exec.security: "allowlist"` + `safeBins`** — Restricts which commands the exec/bash tool can run:

```json
"tools": {
  "exec": {
    "security": "allowlist",
    "safeBins": ["aws", "kubectl", "terraform", "docker", "systemctl", "helm", "git"]
  }
}
```

This blocks `cat`, `less`, `grep` on arbitrary paths — the agent can only run explicitly approved DevOps binaries.

**`logging.redactPatterns`** — Regex patterns to redact from logs (not from AI context):

```json
"logging": {
  "redactSensitive": "tools",
  "redactPatterns": [
    "AKIA[A-Z0-9]{16}",
    "(?i)(secret|password|token)\\s*=\\s*\\S+"
  ]
}
```

#### Layer 2 — Eliminate credential files (AWS)

**IAM Instance Profile** — Attach an IAM role to the EC2 instance. AWS CLI reads credentials from the instance metadata service (`169.254.169.254`) — no `~/.aws/credentials` file exists at all:

```bash
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxx \
  --iam-instance-profile Name=devops-restricted-role
```

The agent can run `aws` commands normally; there is nothing to read.

#### Layer 3 — Non-root process user (highest impact)

Run OpenClaw gateway as a dedicated restricted user instead of `root`:

```bash
useradd -m -s /bin/bash openclaw

# Selective sudo for DevOps tools only
# /etc/sudoers.d/openclaw:
openclaw ALL=(ALL) NOPASSWD: /usr/bin/docker, /usr/bin/kubectl, /usr/bin/aws

# Migrate workspace and config ownership
chown -R openclaw:openclaw /root/.openclaw
# Update systemd service User=openclaw
```

After this, the agent cannot read `/root/.aws/credentials`, other users' SSH keys, or system files owned by root.

#### Layer 4 — CLI wrappers for secrets (Vault / secrets manager)

For non-AWS secrets (DB passwords, API keys), wrap credentials behind a CLI script:

```bash
#!/bin/bash
# /usr/local/bin/db-connect — agent calls this, never sees the password
export DB_PASSWORD=$(vault kv get -field=password secret/prod/db)
psql -h "$DB_HOST" -U "$DB_USER" "$@"
unset DB_PASSWORD
```

The agent calls the wrapper; credential values never appear in the agent's context.

### Recommended implementation order

| Priority | Action | Blocks what | Effort |
|----------|--------|-------------|--------|
| 🔴 P1 | AWS IAM Instance Profile | AWS credential file exposure | Low |
| 🔴 P1 | Run gateway as non-root user | Full filesystem read as root | Medium |
| 🟡 P2 | `exec.security: "allowlist"` + `safeBins` | Arbitrary shell commands | Low |
| 🟡 P2 | `tools.fs.workspaceOnly: true` | `read_file` outside workspace | Low |
| 🟢 P3 | `logging.redactPatterns` | Secret leakage in logs | Low |
| 🟢 P4 | HashiCorp Vault + CLI wrappers | Non-AWS secret exposure | High |

> P1 + P2 covers ~80% of risk with minimal effort. P4 is recommended when managing more than a few external secrets across multiple projects.
