---
title: "Agentopia Chatbot Platform — Architecture & Roadmap"
---

# Agentopia Chatbot Platform — Architecture & Roadmap

> **Version:** 1.1
> **Last updated:** 2026-03-08
> **Status:** Living document — updated as features are implemented
> **Baseline reference:** [Architecture-multiple-bot.md](Architecture-multiple-bot.md) — detailed multi-bot deployment architecture

---

## 1. Executive Summary

Agentopia is a self-hosted AI chatbot platform that enables organizations to deploy multiple specialized AI bots with shared knowledge, long-term memory, and enterprise-grade infrastructure. The platform runs on Kubernetes (EKS) with full GitOps automation via ArgoCD.

**Current state:** Production-ready single-user multi-bot platform with Telegram integration, automated bot provisioning via web UI, and persistent semantic memory.

**Vision:** Enterprise AI chatbot platform with multi-channel support, MCP tool integration, LLM cost management, knowledge base RAG, and multi-tenant SaaS capabilities.

### Key Differentiators

- **Zero-code bot creation:** Natural language requirements → LLM generates complete bot config → auto-deploys via GitOps
- **Shared knowledge across bots:** All bots in a scope share the same memory store and knowledge base
- **Memory survives context compaction:** 3-layer memory architecture ensures no context loss
- **Extensible without forking:** Plugin SDK + LLM Proxy pattern enables 90%+ features without modifying gateway source
- **GitOps-native:** Every change tracked in Git, deployed via ArgoCD ApplicationSet

---

## 2. Current Architecture (Implemented & Production-Ready)

### 2.1 System Overview

```
                                    ┌─────────────────────────────┐
                                    │     bot-config-api          │
                                    │     (FastAPI :8001)         │
                                    │                             │
                                    │  Web UI → LLM Generator    │
                                    │  → K8s Resources           │
                                    │  → Git Push                │
                                    │  → ApplicationSet Patch    │
                                    │  → Pod Monitor             │
                                    └──────────┬──────────────────┘
                                               │ creates
                                               ▼
┌──────────────────────────── EKS Cluster (agentopia namespace) ────────────────────────────┐
│                                                                                            │
│  ┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────────────────────┐   │
│  │  agentopia-gateway      │  │  agentopia-gateway      │  │  agentopia-gateway      │   │
│  │  Bot A (pod)            │  │  Bot B (pod)            │  │  Bot N (pod)            │   │
│  │  :18789                 │  │  :18789                 │  │  :18789                 │   │
│  │                         │  │                         │  │                         │   │
│  │  Agent: {bot-A}         │  │  Agent: {bot-B}         │  │  Agent: {bot-N}         │   │
│  │  Model: claude-sonnet   │  │  Model: gpt-5.2         │  │  Model: claude-opus     │   │
│  │  Channel: Telegram      │  │  Channel: Telegram      │  │  Channel: Telegram      │   │
│  │  Plugin: mem0-api       │  │  Plugin: mem0-api       │  │  Plugin: mem0-api       │   │
│  └────────────┬────────────┘  └────────────┬────────────┘  └────────────┬────────────┘   │
│               │                            │                            │                 │
│               └────────────────┬───────────┘────────────────────────────┘                 │
│                                │                                                          │
│               ┌────────────────┼────────────────┐                                        │
│               │                │                │                                        │
│  ┌────────────▼───┐  ┌────────▼────────┐  ┌────▼─────────────┐                           │
│  │  mem0-api      │  │  Qdrant         │  │  Neo4j           │                           │
│  │  :8000         │◄─►│  :6333          │  │  :7474/:7687     │                           │
│  │  (FastAPI)     │  │  (Vector DB)    │  │  (Graph DB)      │                           │
│  └────────────────┘  └─────────────────┘  └──────────────────┘                           │
│                                                                                            │
│  ┌──────────────────────────────┐  ┌────────────────────────────────────────────────┐    │
│  │  agentopia-llm-proxy  :18789 │  │  AWS EBS (ebs-gp3 StorageClass)                │    │
│  │  (Rust, OpenAI-compat API)   │  │  Per-bot state PVC (RWO, /openclaw-state/)      │    │
│  │  Routes: Codex OAuth →       │  │  Scope-based sharing: NOT active               │    │
│  │  openai-codex/gpt-5.1        │  └────────────────────────────────────────────────┘    │
│  └──────────────────────────────┘                                                         │
│                                                                                            │
└────────────────────────────────────────────────────────────────────────────────────────────┘
          │                        │                        │                   │
    Telegram API             Anthropic API            OpenRouter API       OpenAI Codex
    (1 token/bot)            (Claude models)          (mem0 + embed)      (gpt-5.1, OAuth)
```

### 2.2 Core Services

| Service | Port | Runtime | Purpose | Deployed By |
|---------|------|---------|---------|-------------|
| `agentopia-gateway` | 18789 | Node.js (npm `openclaw@2026.2.26`) | AI agent orchestrator — routes messages to agents | Helm (`agentopia-bot`) |
| `bot-config-api` | 8001 | Python FastAPI | LLM-powered bot provisioning via web UI + A2A orchestrator | Helm (`agentopia-base`) |
| `mem0-api` | 8000 | Python FastAPI | Fact extraction + semantic memory API (OpenRouter LLM + embed) | Helm (`agentopia-base`) |
| `agentopia-llm-proxy` | 18789 (K8s svc) | Rust (Axum) | OpenAI-compatible proxy → Codex OAuth (gpt-5.1); auth via `AGENTOPIA_RELAY_TOKEN` | Helm (`agentopia-base`) |
| `Qdrant` | 6333 | Qdrant v1.16.1 (StatefulSet) | Vector database for semantic memory | Helm (`agentopia-base`) |
| `Neo4j` | 7474/7687 | Neo4j 5.26 (StatefulSet) | Graph database for entity relations | Helm (`agentopia-base`) |
| `Vault` | 8200 | HashiCorp Vault (dev mode) | API key store (ANTHROPIC_API_KEY, OPENROUTER_API_KEY, AGENTOPIA_GATEWAY_TOKEN) | Helm (`agentopia-base`) |

### 2.3 Gateway Architecture (Closed Source, Extensible)

The openclaw-gateway is an npm package (`openclaw@2026.2.26`) that runs as a Node.js process. While the core is closed-source, it provides extensive extensibility through:

**Configuration-driven features:**
- Multi-model support (Anthropic, OpenAI, OpenRouter, etc.)
- Multi-channel support (15+ channels — see Section 3.1)
- MCP server integration (via `mcp-bridge` Plugin SDK extension — see Section 3.2)
- Memory search with hybrid vector+text retrieval
- Context pruning and compaction with automatic memory flush
- Per-agent workspace isolation
- Group/topic-level routing and mention patterns

**Plugin SDK (`openclaw/plugin-sdk`):**
- TypeScript interface `OpenClawPluginApi`
- Event-driven: `before_agent_start`, `agent_end`
- Inject context before LLM call (`prependContext`)
- Capture messages after response
- Register background services
- Full plugin lifecycle management

**Current custom plugin:** `mem0-api` — see Section 2.6 for Plugin SDK details.

### 2.4 Bot Provisioning Pipeline (5-Step Automated Deploy)

```
Web UI (http://<ip>/ui)
      │
      │ POST /api/v1/bots/deploy
      ▼
  bot-config-api (FastAPI)
      │
      ├── Step 1: generate
      │     LLMGenerator → SOUL.md + USER.md + bot.yaml
      │     (agentopia-llm-proxy → openai-codex/gpt-5.1, tool calling)
      │     Cost: ~$0 token burn (Codex OAuth subscription)
      │
      ├── Step 2: k8s_resources
      │     K8sService →
      │       agentopia-soul-<bot>          (ConfigMap: SOUL.md, USER.md)
      │       agentopia-gateway-env-<bot>   (Secret: AGENTOPIA_GATEWAY_TOKEN, AGENTOPIA_RELAY_TOKEN)
      │
      ├── Step 3: scope_pvc
      │     Ensure agentopia-shared-<scope> PVC exists (currently disabled — skipped)
      │
      ├── Step 4: git_push
      │     GitHubService → atomic 4-file commit:
      │       bots/<bot>/SOUL.md, USER.md, bot.yaml, argocd/agentopia-bots.yaml
      │     K8sService → patch ApplicationSet with REAL telegram token
      │       (GitHub stores "REPLACE_ME" — real token only in K8s)
      │           │
      │           ▼
      │      ArgoCD detects ApplicationSet change (~30s)
      │           │
      │           ▼
      │      Helm renders all K8s resources for the new bot
      │
      └── Step 5: pod_monitor
            Poll until pod reaches Running state (timeout 120s)
```

### 2.5 Token Flow & Security

```
UI telegram_token
      │
      ▼
bot-config-api.add_bot_to_applicationset(telegram_token=real)
      │
      ▼  (ApplicationSet element — K8s object, standalone, NOT synced from Git)
ApplicationSet.spec.generators[0].list.elements[].telegramToken = "<real_token>"
      │
      ▼  (ArgoCD Helm sync)
Helm chart → Secret/agentopia-bot-token-<bot>.data.token = base64(<real_token>)
      │
      ▼  (pod volume mount)
/secrets/telegram/token → openclaw.json tokenFile reads this
      │
      ▼
Bot connects to Telegram
```

**Security principle:** GitHub **always** stores `REPLACE_ME` — the real token exists only in the K8s ApplicationSet object (in-cluster). ArgoCD `ignoreDifferences` on Secret prevents `selfHeal` from reverting the patched value.

### 2.6 K8s Resources Per Bot

| Resource | Name Pattern | Created By | Lifecycle |
|----------|-------------|------------|-----------|
| ConfigMap | `agentopia-soul-<bot>` | bot-config-api (step 2) | Recreated on redeploy |
| Secret | `agentopia-gateway-env-<bot>` | bot-config-api (step 2) | Contains `AGENTOPIA_GATEWAY_TOKEN` + `AGENTOPIA_RELAY_TOKEN` |
| Secret | `agentopia-bot-token-<bot>` | Helm (ArgoCD sync) | Real token from ApplicationSet |
| PVC | `agentopia-state-<bot>` | Helm (`resource-policy: keep`) | Persists across redeploys (ebs-gp3, RWO) |
| PVC | `agentopia-shared-<scope>` | bot-config-api (step 3) | Shared across bots in scope (currently disabled — no EFS configured) |
| Deployment | `agentopia-<bot>` | Helm (ArgoCD sync) | Managed by ArgoCD |
| ConfigMap | `agentopia-config-<bot>` | Helm (ArgoCD sync) | openclaw.json with Helm values |

### 2.7 Memory Architecture (3 Layers)

> Detailed in [Architecture-multiple-bot.md § 4](Architecture-multiple-bot.md#4-memory-architecture-3-layers)

```
┌──────────────────────────────────────────────────────┐
│  L1: IDENTITY (per-bot, static, auto-loaded)         │
│  SOUL.md → Role, personality, behavioral rules       │
│  USER.md → Quick card (name, timezone, language)     │
│  Loaded: every session start (bootstrapMaxChars: 50k)│
├──────────────────────────────────────────────────────┤
│  L2: KNOWLEDGE (shared across bots, persistent)      │
│                                                      │
│  [AUTO] mem0 → Qdrant                                │
│  ├── autoRecall: top 5 facts before each response    │
│  │   (timeout: 3.5s, non-blocking)                   │
│  ├── autoCapture: extract facts after each response  │
│  │   (captureOnFailure: true)                        │
│  └── Pipeline: messages → OpenRouter LLM → Qdrant   │
│                                                      │
│  [ON-DEMAND] shared-memory/ (scope PVC — NOT active)  │
│  ├── USER.md, architecture.md, decisions.md          │
│  ├── groups/{chatId}.md, {chatId}_{topicId}.md       │
│  └── Accessed via memory_search() tool call          │
├──────────────────────────────────────────────────────┤
│  L3: SESSION (per-bot, rolling)                      │
│  memory/YYYY-MM-DD.md → Daily summaries              │
│  MEMORY.md → Curated long-term facts (DM only)       │
│  Written by bot before context compaction            │
└──────────────────────────────────────────────────────┘
```

**Scope-based isolation (K8s — NOT currently active, all bots are isolated):**

| Scenario | Shared PVC | mem0 userId | Isolation |
|----------|-----------|-------------|-----------|
| Bot A + B in scope "alpha" | `agentopia-shared-alpha` | `scope-alpha` | Shared memory |
| Bot C in scope "beta" | `agentopia-shared-beta` | `scope-beta` | Isolated |
| Bot D (no scope) | none | `bot-{botName}` | Fully isolated |

### 2.8 Plugin SDK Architecture

The mem0-api plugin demonstrates the extensibility pattern available for all future features:

```typescript
// Extension structure
extensions/
└── mem0-api/
    ├── index.ts                 // Plugin implementation
    ├── package.json
    └── openclaw.plugin.json     // Plugin manifest (kind, configSchema)

// Plugin interface
const plugin = {
  id: "mem0-api",
  kind: "memory" as const,

  register(api: OpenClawPluginApi) {
    // Event: before LLM call — inject context
    api.on("before_agent_start", async (event, ctx) => {
      // event.prompt = user message
      // return { prependContext: "..." } to inject into context
    });

    // Event: after LLM response — capture data
    api.on("agent_end", async (event, ctx) => {
      // event.messages = full conversation
      // event.success = whether agent completed
    });

    // Register background service
    api.registerService({ id: "...", start: () => {}, stop: () => {} });
  },
};
```

**Reusable for future features:**
| Feature | Event | Action |
|---------|-------|--------|
| Audit logging | `agent_end` | Write conversation to audit store |
| Knowledge base RAG | `before_agent_start` | Inject relevant documents from vector DB |
| Content filtering (input) | `before_agent_start` | Scan user message for PII/harmful content |
| Metrics collection | `agent_end` | Count tokens, track response time |
| Multi-agent routing | `before_agent_start` | Route to specialist agent based on query |

### 2.9 GitOps & ArgoCD

```
GitHub: ai-agentopia/agentopia-infra  (infra only)
      │
      ├── charts/agentopia-base/    → ArgoCD App: agentopia-base (sync-wave -1)
      │     Infrastructure: mem0-api, Qdrant, Neo4j, Vault, agentopia-llm-proxy, bot-config-api
      │     Storage: ebs-gp3 StorageClass (scope PVC disabled — efs.fileSystemId not set)
      │
      ├── charts/agentopia-bot/     → ArgoCD ApplicationSet: agentopia-bots
      │     Per-bot: gateway pod, config, secrets, PVC
      │
      └── argocd/agentopia-bots.yaml → ApplicationSet definition (standalone K8s object)
            elements: [] (bots added by bot-config-api, not manually)

GitHub: ai-agentopia/agentopia-protocol  (application code)
      │
      └── bot-config-api/, mem0-api/, gateway/, llm-proxy/
            → built by build-images.yml → pushed to ghcr.io/ai-agentopia/
```

**ApplicationSet management rules:**
- `agentopia-bots.yaml` in git = source of truth for structure
- ApplicationSet K8s object = standalone (not synced from git by ArgoCD)
- bot-config-api patches K8s object directly (adds elements with real tokens)
- `argocd/agentopia-bots.yaml` in git tracks bot additions for auditability
- **Never** patch K8s ApplicationSet manually without updating git file

### 2.10 Operational Health Checks

| Probe | Type | Target | Notes |
|-------|------|--------|-------|
| Readiness | `tcpSocket` | port 18789 | NOT httpGet (canvas endpoint returns 401) |
| Liveness | `tcpSocket` | port 18789 | Same reason |
| Startup | `tcpSocket` | port 18789 | `failureThreshold: 30`, `periodSeconds: 10` |

**Known fixes applied (do not revert):**
- `tcpSocket` probes instead of `httpGet /__openclaw__/canvas/` (returns 401)
- `dmPolicy: "open"` + `allowFrom: ["*"]` (no pairing required)
- `AGENTOPIA_GATEWAY_TOKEN` injected from `agentopia-gateway-env-<bot>` Secret
- Gateway memorySearch: OpenRouter `openai/text-embedding-3-small` via `${OPENROUTER_API_KEY}` (infinity REMOVED)
- `${OPENROUTER_API_KEY}` in `openclaw.json` is resolved from Vault at pod startup (not K8s Secret directly)

---

## 3. Platform Capabilities Discovery

> Discovered during deep-dive review (2026-03-03). These native capabilities significantly reduce implementation effort for planned features.

### 3.1 Native Multi-Channel Support

The openclaw-gateway npm package **natively supports 15+ messaging channels**. Adding a new channel requires only configuration changes — no code development.

**Supported channels:**

| Channel | Config Key | Auth Method | Status |
|---------|-----------|-------------|--------|
| Telegram | `channels.telegram` | Bot token (BotFather) | **In use** |
| Slack | `channels.slack` | Bot token (Slack App) | Available |
| Discord | `channels.discord` | Bot token (Discord Dev) | Available |
| WhatsApp | `channels.whatsapp` | Business API | Available |
| Microsoft Teams | `channels.teams` | Azure Bot Service | Available |
| Signal | `channels.signal` | Signal CLI | Available |
| Matrix | `channels.matrix` | Homeserver token | Available |
| WebChat | `channels.webchat` | Built-in UI | Available |
| Google Chat | `channels.googlechat` | Service account | Available |
| LINE | `channels.line` | Channel token | Available |
| Facebook Messenger | `channels.messenger` | Page token | Available |
| Twilio SMS | `channels.twilio` | Account SID | Available |
| REST API | `channels.rest` | API key | Available |
| SSE (Server-Sent Events) | `channels.sse` | Token | Available |

**Adding a channel (example: Slack):**
```json
// openclaw.json — add alongside existing telegram config
"channels": {
  "telegram": { ... },
  "slack": {
    "accounts": {
      "my-slack-bot": {
        "enabled": true,
        "botToken": "xoxb-...",
        "appToken": "xapp-..."
      }
    }
  }
},
"bindings": [
  { "agentId": "bot-a", "match": { "channel": "telegram", "accountId": "..." } },
  { "agentId": "bot-a", "match": { "channel": "slack", "accountId": "my-slack-bot" } }
]
```

**Impact on roadmap:** F3 (Multi-Channel) effort reduced from months to **days per channel** — mostly channel-specific app setup, not development.

### 3.2 MCP Integration via Plugin Bridge

> **Correction (2026-03-14):** Earlier versions of this document claimed openclaw supports `mcpServers` as a native top-level config key. E2E testing disproved this — openclaw v2026.3.8 uses strict config validation (`additionalProperties: false`) and rejects `mcpServers` as "Unrecognized key". openclaw VISION.md confirms MCP integration is via bridge model, not native config.

MCP tool integration uses the **Plugin SDK** — a `mcp-bridge` extension (kind: `tool`) that:
1. Reads MCP server definitions from `plugins.entries.mcp-bridge.config`
2. Connects to configured MCP servers (stdio transport only; SSE planned)
3. Discovers available tools from each MCP server
4. Registers tools via `api.registerTool()` — same pattern as relay and wf-bridge extensions
5. Enforces per-bot allowed/denied tool policy at runtime

```json5
// openclaw.json — MCP via plugin bridge (NOT top-level mcpServers)
"plugins": {
  "allow": ["mem0-api", "telegram", "relay", "wf-bridge", "mcp-bridge"],
  "entries": {
    "mcp-bridge": {
      "enabled": true,
      "config": {
        "servers": {
          "github": {
            "transport": "stdio",
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${MCP_GITHUB_TOKEN}" }
          }
        },
        "permissions": {
          "allowed_tools": ["create_issue", "search_code", "get_file"],
          "denied_tools": []
        }
      }
    }
  }
}
```

**Architecture layers:**
- **UI/Control plane** (bot-config-api): MCP registry, per-bot permissions, credential CRUD via UI
- **Runtime** (gateway plugin): `mcp-bridge` extension connects to MCP servers, registers tools
- **Auth**: Per-bot K8s Secrets (`mcp-{server}-token-{bot_name}`) → pod env vars → plugin config env references (no plaintext in config)

**Per-bot MCP credential flow (#221):**
```
UI: Save Credential (PAT)
    │
    ▼
PUT /api/v1/bots/{bot}/mcp/credentials/{server}
    │
    ├── K8s Secret created: mcp-{server}-token-{bot}
    │     Labels: agentopia/managed-by, agentopia/bot-name, agentopia/mcp-server
    │
    └── _reapply_bot_mcp_runtime()
          │
          ├── Re-reads per-bot MCP config (registry + per-bot secret names)
          ├── Patches Application CRD with updated mcpBridge + mcpSecrets
          └── ArgoCD selfHeal → pod rollout with new credentials
```

- Precedence: per-bot secret > shared secret > none
- Raw tokens never returned by API (redacted responses only)
- Two bots can use different credentials independently
- Credential delete removes the K8s Secret and triggers runtime re-apply

**Impact on roadmap:** F2 (MCP Integration) requires `mcp-bridge` extension development + existing control-plane infrastructure. Not config-only as previously assumed.

### 3.3 Plugin SDK Extensibility

As demonstrated by the mem0-api plugin, the Plugin SDK enables:
- **Input processing** via `before_agent_start` event
- **Output processing** via `agent_end` event
- **Context injection** via `prependContext` return value
- **Background services** via `registerService`
- **Configurable per-bot** via `plugins.entries.<id>.config`

This pattern covers audit logging, knowledge base RAG, content filtering, metrics collection — all without modifying gateway source code.

---

## 4. Roadmap — Platform Foundation (P0–P4)

### P0: Security & Authentication (Critical)

| # | Task | Approach | Effort |
|---|------|----------|--------|
| 1 | Vault integration for secrets | Replace env vars with Vault KV mount | 3-5 days |
| 2 | API authentication (bot-config-api) | JWT/API key middleware | 2-3 days |
| 3 | RBAC for bot management | Role-based access control | 3-5 days |
| 4 | Gateway security hardening | `tools.fs.workspaceOnly`, `exec.security: allowlist` | 1-2 days |

**GitHub:** [#11](https://github.com/ai-agentopia/agentopia-infra/issues/11)

### P1: Bot Lifecycle Management

| # | Task | Approach | Effort |
|---|------|----------|--------|
| 1 | Bot update/redeploy (SOUL.md changes) | bot-config-api endpoint + UI | 3-5 days |
| 2 | Bot pause/resume | Scale deployment to 0/1 | 1-2 days |
| 3 | Bot deletion with cleanup | Delete ApplicationSet element + K8s resources | 2-3 days |
| 4 | Bot health monitoring UI | Real-time pod status + Telegram connection | 3-5 days |

**GitHub:** [#12](https://github.com/ai-agentopia/agentopia-infra/issues/12)

### P2: Observability & Monitoring

| # | Task | Approach | Effort |
|---|------|----------|--------|
| 1 | Prometheus + Grafana stack | Helm chart, ServiceMonitor | 2-3 days |
| 2 | K8s metrics dashboards | Pod CPU/memory, restart count | 1-2 days |
| 3 | Alerting (PagerDuty/Slack) | AlertManager rules | 2-3 days |
| 4 | Log aggregation | Loki or EFK stack | 3-5 days |

> **Scope boundary:** P2 = infrastructure health (pods, nodes, databases). LLM cost analytics → F5 (depends on F9 LLM Proxy).

**GitHub:** [#13](https://github.com/ai-agentopia/agentopia-infra/issues/13)

### P3: Multi-tenancy & Scaling

| # | Task | Approach | Effort |
|---|------|----------|--------|
| 1 | Namespace-per-tenant isolation | Helm + ArgoCD per namespace | 5-7 days |
| 2 | Resource quotas & LimitRange | Per-namespace resource control | 1-2 days |
| 3 | Network policies | Inter-namespace isolation | 2-3 days |
| 4 | Horizontal pod autoscaling | HPA based on message queue depth | 3-5 days |

> **Scope boundary:** P3 = infrastructure isolation. SaaS business logic (billing, onboarding) → F8.

**GitHub:** [#14](https://github.com/ai-agentopia/agentopia-infra/issues/14)

### P4: Developer Experience & Extensibility

| # | Task | Approach | Effort |
|---|------|----------|--------|
| 1 | Plugin SDK documentation | API reference, examples, cookbook | 3-5 days |
| 2 | Extension scaffolding CLI | `openclaw plugin create <name>` equivalent | 2-3 days |
| 3 | Local dev environment | docker-compose for full stack | 3-5 days |
| 4 | CI/CD for custom plugins | Build + deploy plugin extensions | 2-3 days |

> **Scope boundary:** P4 = developer tools. Bot templates gallery → F7.

**GitHub:** [#15](https://github.com/ai-agentopia/agentopia-infra/issues/15)

---

## 5. Roadmap — Feature Epics (F1–F9)

### F1: Multi-Agent Orchestration

**Vision:** Bots collaborate on complex tasks. User asks one bot, it delegates subtasks to specialist bots.

| # | Issue | Description | Effort | Fork? |
|---|-------|-------------|--------|-------|
| #24 | Agent-to-agent messaging bus | Redis/NATS pub/sub for inter-bot communication | 5-7 days | No |
| #25 | Orchestrator agent pattern | Meta-agent that routes queries to specialists | 5-7 days | No |
| #26 | A2A protocol adapter | Google Agent-to-Agent protocol support | 7-10 days | No |

**Dependencies:** None (standalone feature).
**GitHub:** [#16](https://github.com/ai-agentopia/agentopia-infra/issues/16)

### F2: MCP Tool Integration

**Vision:** Bots use external tools (GitHub, databases, file systems) via MCP servers.

> **Correction (2026-03-14):** "Native MCP support" claim was incorrect. openclaw runtime rejects top-level `mcpServers` config. MCP requires a `mcp-bridge` gateway extension using the Plugin SDK. Control-plane infrastructure (registry, permissions, auth) is built; runtime bridge is the remaining work.

| # | Issue | Description | Effort | Fork? |
|---|-------|-------------|--------|-------|
| #219 | Build mcp-bridge gateway extension | Plugin SDK extension to connect MCP servers and register tools | 5-7 days | No |
| #27 | Deploy core MCP server (GitHub) | Start with 1 server, expand incrementally | 2-3 days | No |
| #28 | Per-bot tool permissions & config UI | UI to manage which tools each bot can access | 3-5 days | No |
| #29 | Custom MCP server builder | Template for building domain-specific MCP servers | 5-7 days | No |

**Dependencies:** #219 (bridge) must be completed before #27 (deploy) can deliver runtime tool access.
**GitHub:** [#17](https://github.com/ai-agentopia/agentopia-infra/issues/17), [#219](https://github.com/ai-agentopia/agentopia-protocol/issues/219)

### F3: Multi-Channel Support

**Vision:** Same bot accessible via Telegram, Slack, WebChat, Discord, etc.

> **Key discovery:** Gateway natively supports 15+ channels. Adding a channel = config change, not development.

| # | Issue | Description | Effort | Fork? |
|---|-------|-------------|--------|-------|
| #30 | Slack integration | Config-only: create Slack app, add account config | 1-2 days | No |
| #31 | Embeddable web chat widget | Built-in to gateway, enable in config | < 1 day | No |
| #32 | REST/SSE API channel | Enable REST API channel for custom integrations | 1-2 days | No |

**Dependencies:** Helm chart update to support multi-channel config (bot-config-api changes).
**GitHub:** [#18](https://github.com/ai-agentopia/agentopia-infra/issues/18)

### F4: Knowledge Base & RAG Pipeline

**Vision:** Bots answer questions from organizational documents, code repos, and structured data.

| # | Issue | Description | Effort | Fork? |
|---|-------|-------------|--------|-------|
| #33 | Document ingestion API | PDF, Markdown, HTML → chunk → embed → Qdrant | 5-7 days | No |
| #34 | Code repository indexing | GitHub/GitLab repo → code-aware chunking → embed | 5-7 days | No |
| #35 | Graph RAG | Combine Neo4j entities + Qdrant vectors for richer retrieval | 7-10 days | No |

**Approach:** New microservice (`knowledge-api`) + Plugin SDK integration (`before_agent_start` → inject relevant docs).
**Dependencies:** P4 (Plugin SDK docs) helpful but not blocking.
**GitHub:** [#19](https://github.com/ai-agentopia/agentopia-infra/issues/19)

### F5: Analytics & Cost Management

**Vision:** Track LLM costs per bot, per model, per user. Optimize spending with smart routing and caching.

| # | Issue | Description | Effort | Fork? |
|---|-------|-------------|--------|-------|
| #36 | Per-bot LLM token tracking | LiteLLM metrics → Prometheus → per-bot cost | 3-5 days | No |
| #37 | Grafana dashboards | Cluster health + per-bot cost breakdown | 2-3 days | No |
| #38 | Smart model routing & semantic cache | LiteLLM routing rules + Redis cache | 3-5 days | No |

> **Dependency:** All F5 items require **F9 (LLM Proxy)** to be deployed first. Without proxy, no LLM traffic visibility.

**GitHub:** [#20](https://github.com/ai-agentopia/agentopia-infra/issues/20)

### F6: Governance, Audit & Compliance

**Vision:** Enterprise compliance with conversation audit trails, content filtering, and approval workflows.

| # | Issue | Description | Effort | Fork? |
|---|-------|-------------|--------|-------|
| #39 | Conversation audit log (immutable) | Plugin SDK `agent_end` → write to audit store | 3-5 days | No |
| #40 | Content filtering & PII detection | Input: Plugin SDK. Output: LLM Proxy or **fork** | 5-7 days | Maybe |
| #41 | Human-in-the-loop approval | Approval workflow before executing actions | 7-10 days | Maybe |

**Fork analysis:**
- #39: No fork — Plugin SDK `agent_end` captures all messages
- #40: Input filtering via Plugin SDK (`before_agent_start`). Output filtering options: (a) LLM Proxy output filter (no fork), (b) gateway fork for response interception
- #41: Options: (a) MCP tool that pauses execution (no fork), (b) gateway fork for blocking approval. MCP approach preferred.

**GitHub:** [#21](https://github.com/ai-agentopia/agentopia-infra/issues/21)

### F7: Bot Marketplace & Templates

**Vision:** Library of pre-built bot templates that users can deploy with one click.

| # | Issue | Description | Effort | Fork? |
|---|-------|-------------|--------|-------|
| #42 | Template schema v2 & gallery UI | Browse, preview, deploy templates | 5-7 days | No |
| #43 | Fork, customize & version templates | Template versioning, user customization | 5-7 days | No |

**Dependencies:** bot-config-api already has template system. This extends it with UI and versioning.
**GitHub:** [#22](https://github.com/ai-agentopia/agentopia-infra/issues/22)

### F8: SaaS Platform & Monetization

**Vision:** Multi-tenant SaaS offering with self-service onboarding and usage-based billing.

| # | Issue | Description | Effort | Fork? |
|---|-------|-------------|--------|-------|
| #44 | Multi-tenant data model | Namespace isolation, tenant config | 7-10 days | No |
| #45 | Self-service onboarding flow | Registration, Telegram token setup, first bot | 5-7 days | No |
| #46 | Usage metering & Stripe billing | Token-based pricing, Stripe integration | 7-10 days | No |

**Dependencies:** P3 (multi-tenancy infra), F9 (LLM Proxy for usage metering).
**GitHub:** [#23](https://github.com/ai-agentopia/agentopia-infra/issues/23)

### F9: LLM Proxy — Custom Rust Proxy (IMPLEMENTED ✓)

**Status: IMPLEMENTED** — `agentopia-llm-proxy` deployed in cluster, NOT LiteLLM.

**What was built:**
```
bot-config-api ──→ agentopia-llm-proxy :18789 ──→ OpenAI Codex OAuth ──→ gpt-5.1
A2A debate/bridge/epoch ──→ same proxy
                          │
                     ┌────┴────────────────────────────┐
                     │ Rust (Axum), OpenAI-compat API   │
                     │ Provider routing (by model prefix)│
                     │ Bearer auth (AGENTOPIA_RELAY_TOKEN)│
                     │ Codex OAuth token management      │
                     └─────────────────────────────────┘
```

**Why custom Rust proxy instead of LiteLLM:**
- LiteLLM requires x-api-key header — incompatible with Codex OAuth token format
- Codex OAuth requires special header injection for Microsoft/OpenAI enterprise auth
- Custom proxy is ~200 lines Rust, zero maintenance overhead
- Source: `agentopia-protocol/llm-proxy/`

**What's NOT implemented yet (from original F9 plan):**
- Cost tracking dashboard (Codex API returns 0 tokens — limitation of Codex API)
- Semantic cache (Redis)
- Rate limiting & budget alerts per-bot

**Deployed:**
- Image: `ghcr.io/ai-agentopia/agentopia-llm-proxy:latest`
- Secret: `agentopia-llm-proxy-env` → `AGENTOPIA_RELAY_TOKEN`
- K8s Service: `agentopia-llm-proxy.agentopia.svc.cluster.local:18789`

**GitHub:** [#47](https://github.com/ai-agentopia/agentopia-infra/issues/47) (partially done)

---

## 6. Dependency Graph

```
                    ┌─────────┐
                    │   P0    │ Security & Auth
                    │(Critical)│
                    └────┬────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
         ┌────▼────┐ ┌──▼───┐ ┌───▼───┐
         │   P1    │ │  P2  │ │  F9   │ ← LLM Proxy (NEW)
         │Lifecycle│ │ Obs  │ │Proxy  │
         └────┬────┘ └──┬───┘ └───┬───┘
              │         │         │
              │         │    ┌────┼────────────────┐
              │         │    │    │                 │
         ┌────▼────┐    │ ┌──▼───▼──┐  ┌──────────▼──┐
         │   F3    │    │ │   F5    │  │     F6      │
         │MultiCh  │    │ │Analytics│  │  Governance  │
         │(config) │    │ │(needs F9)│  │(audit+filter)│
         └─────────┘    │ └─────────┘  └─────────────┘
                        │
              ┌─────────┼─────────┐
              │         │         │
         ┌────▼────┐ ┌──▼───┐ ┌──▼────┐
         │   F2    │ │  F4  │ │  F1   │
         │  MCP    │ │ RAG  │ │Multi  │
         │(config) │ │ KB   │ │Agent  │
         └─────────┘ └──────┘ └───────┘

              ┌─────────┐
              │   P3    │──→ ┌──────┐
              │MultiTen │    │  F8  │ SaaS
              └────┬────┘    │(needs│
                   │         │P3+F9)│
              ┌────▼────┐    └──────┘
              │   P4    │──→ ┌──────┐
              │   DX    │    │  F7  │ Marketplace
              └─────────┘    └──────┘
```

### Sprint Roadmap

Each sprint builds on the previous one. Issues cannot start until their dependencies from prior sprints are complete.

| Sprint | Label | Epics | Issues | Why This Order |
|--------|-------|-------|--------|----------------|
| **Sprint 1** | `sprint:1` | **P0** Security, **F9** LLM Proxy | #11, #47, #48-51 | Foundation layer. P0 secures the platform (no auth = no production). F9 deploys LiteLLM proxy — prerequisite for cost tracking, audit, and model routing in later sprints. |
| **Sprint 2** | `sprint:2` | **P1** Bot Lifecycle, **F3** Multi-Channel | #12, #18, #30-32 | Complete the CRUD cycle: currently can create bots, now add update/pause/delete. F3 is config-only (gateway supports 15+ channels natively), high business value, very low effort. |
| **Sprint 3** | `sprint:3` | **F2** MCP Tools, **F4** Knowledge Base | #17, #219, #27-29, #19, #33-35 | Make bots smarter. F2 adds external tools (GitHub, DB) via `mcp-bridge` Plugin SDK extension (#219). F4 adds document/code knowledge base (new microservice + Plugin SDK RAG injection). Both are independent and can parallelize. |
| **Sprint 4** | `sprint:4` | **P2** Observability, **F5** Analytics | #13, #20, #36-38 | Full visibility. P2 deploys Prometheus+Grafana for infra monitoring. F5 builds on F9 (Sprint 1) for LLM cost dashboards. #38 (model routing) uses LiteLLM routing rules from Sprint 1. |
| **Sprint 5** | `sprint:5` | **F6** Governance, **P4** DX | #21, #39-41, #15 | Enterprise compliance. F6 adds audit logs (Plugin SDK), content filtering (Plugin SDK + LLM Proxy from Sprint 1), HITL approval. P4 documents Plugin SDK and creates dev tooling for ecosystem growth. |
| **Sprint 6** | `sprint:6` | **F1** Multi-Agent, **P3** Multi-tenancy | #16, #24-26, #14 | Advanced features. F1 requires stable platform (Sprints 1-5) for agent collaboration. P3 adds namespace isolation — prerequisite for SaaS in Sprint 7. |
| **Sprint 7** | `sprint:7` | **F7** Marketplace, **F8** SaaS | #22, #42-43, #23, #44-46 | Monetization layer. F7 builds template gallery (needs P4 from Sprint 5). F8 adds billing + onboarding (needs P3 from Sprint 6, F9+F5 for usage metering). |

### Sprint Dependency Chain

```
Sprint 1 ──→ Sprint 2 ──→ Sprint 3 ──→ Sprint 4 ──→ Sprint 5 ──→ Sprint 6 ──→ Sprint 7
(Foundation)  (Platform)   (Intelligence) (Visibility)  (Enterprise)  (Advanced)   (Monetize)
P0 + F9       P1 + F3      F2 + F4        P2 + F5       F6 + P4       F1 + P3      F7 + F8
   │                                         │              │              │
   └─── F9 unblocks ────────────────────── F5 cost ─── F6 audit ──────────┘
                                              │
                                         F5 needs P2
```

**Cross-sprint dependencies:**
- F5 (Sprint 4) depends on F9 (Sprint 1): LLM Proxy provides cost data
- F5 (Sprint 4) depends on P2 (Sprint 4): Prometheus/Grafana for dashboards
- F6 (Sprint 5) depends on F9 (Sprint 1): LLM Proxy for output filtering
- F8 (Sprint 7) depends on P3 (Sprint 6): Multi-tenant infra
- F8 (Sprint 7) depends on F5 (Sprint 4): Usage metering for billing
- F7 (Sprint 7) depends on P4 (Sprint 5): Plugin SDK ecosystem

---

## 7. Fork Analysis

### Do We Need to Fork openclaw-gateway?

**Short answer: No, for 90%+ of planned features.**

The gateway is an npm package (`openclaw@2026.2.26`). Forking means maintaining a custom build, losing upstream updates, and significantly increasing maintenance burden.

### Feature Feasibility Without Fork

| Approach | Coverage | Features |
|----------|----------|----------|
| **Config-only** (openclaw.json, Helm) | ~35% | Multi-channel (F3), security hardening (P0), memory scope |
| **Plugin SDK** (custom extensions) | ~30% | Audit logging (F6/#39), knowledge base RAG (F4), metrics, content filter (input) |
| **LLM Proxy** (LiteLLM) | ~20% | Cost tracking (F5), model routing (#38), semantic cache, output filtering (#40), budget alerts |
| **New microservices** | ~8% | bot-config-api (done), knowledge-api (F4), orchestrator (F1), tenant-api (F8) |
| **Possible fork** | ~2% | Output content filter (#40 if proxy insufficient), HITL blocking approval (#41) |

### Alternatives to Forking for Edge Cases

**#40 Content filtering (output):**
1. **LLM Proxy output hook** (preferred) — LiteLLM supports custom callbacks on response
2. **Post-processing plugin** — if Plugin SDK adds `before_response_send` event in future
3. **Fork** — last resort if neither works

**#41 Human-in-the-loop approval:**
1. **MCP tool** (preferred) — bot calls `approval_request` MCP tool that pauses and waits for human approval via webhook
2. **Plugin with external approval service** — `before_agent_start` checks pending approvals
3. **Fork** — only if blocking approval in agent execution loop is required

---

## 8. Technology Decisions

### Why LiteLLM for LLM Proxy?

| Criteria | LiteLLM | Custom Proxy | Direct Integration |
|----------|---------|-------------|-------------------|
| OpenAI-compatible API | Yes | Build from scratch | N/A |
| Per-key cost tracking | Built-in | Custom | Not possible |
| 100+ LLM providers | Yes | Manual | N/A |
| Semantic caching | Built-in (Redis) | Custom | Not possible |
| Rate limiting | Built-in | Custom | Not possible |
| K8s deployment | Helm chart available | Custom | N/A |
| Maintenance burden | Low (OSS community) | High | Zero |

### Why ArgoCD ApplicationSet (not Flux, not raw ArgoCD)?

- **ApplicationSet:** One template → N bots. Bot additions are element patches, not new YAML files.
- **ArgoCD vs Flux:** ArgoCD has ApplicationSet, better UI, stronger K8s adoption (CNCF graduated).
- **Standalone ApplicationSet:** Not synced from git → allows runtime token patching without git conflicts.

### Why Plugin SDK over Gateway Fork?

- **Upstream compatibility:** npm package updates don't break custom plugins
- **Hot-reload:** Plugins can be updated without gateway restart
- **Isolation:** Plugin crash doesn't take down the gateway
- **Community:** Plugin pattern is standard in the openclaw ecosystem

---

## 9. Current Production Topology

### EKS Cluster Details

| Property | Value |
|----------|-------|
| Cluster | `arn:aws:eks:eu-central-1:198698840116:cluster/nbo-dev-apps` |
| Namespace | `agentopia` (only namespace we operate in) |
| Node count | 7 schedulable nodes |
| Storage | EBS gp3 (per-bot state PVC, RWO); scope shared-memory PVC not active |
| Images | `ghcr.io/ai-agentopia/agentopia-gateway`, `mem0-api`, `bot-config-api`, `agentopia-llm-proxy` |
| CI/CD | GitHub Actions in `agentopia-protocol` repo → build + push on merge to main |
| GitOps | ArgoCD (apps: `agentopia-base`, `agentopia-bots` ApplicationSet) |

### Resource Budget Per Bot

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 250m | 1000m |
| Memory | 768Mi | 2Gi |
| Storage (state PVC) | 2Gi | — |

> With 7 nodes, current cluster supports ~20-25 concurrent bots before hitting CPU limits. Scale by adding nodes or reducing per-bot requests.

---

## 10. References

| Document | Description |
|----------|-------------|
| [Architecture-multiple-bot.md](Architecture-multiple-bot.md) | Detailed multi-bot deployment architecture (memory layers, file layout, on-premise setup) |
| [bot-config-api.md](bot-config-api.md) | bot-config-api v2.0.0 API reference, deploy pipeline, token flow |
| [mem0-api.md](mem0-api.md) | mem0-api service documentation |
| [openclaw-k8s.md](openclaw-k8s.md) | K8s migration guide (from on-premise to EKS) |

### GitHub Issue Tracking (by Sprint)

| Sprint | Epic | Issue # | Sub-issues | Dependencies |
|--------|------|---------|------------|--------------|
| **1** | P0 Security & Auth | [#11](https://github.com/ai-agentopia/agentopia-infra/issues/11) | — | None |
| **1** | F9 LLM Proxy | [#47](https://github.com/ai-agentopia/agentopia-infra/issues/47) | #48, #49, #50, #51 | None |
| **2** | P1 Bot Lifecycle | [#12](https://github.com/ai-agentopia/agentopia-infra/issues/12) | — | Sprint 1 |
| **2** | F3 Multi-Channel | [#18](https://github.com/ai-agentopia/agentopia-infra/issues/18) | #30, #31, #32 | Sprint 1 |
| **3** | F2 MCP Integration | [#17](https://github.com/ai-agentopia/agentopia-infra/issues/17) | #27, #28, #29 | Sprint 2 |
| **3** | F4 Knowledge Base | [#19](https://github.com/ai-agentopia/agentopia-infra/issues/19) | #33, #34, #35 | Sprint 2 |
| **4** | P2 Observability | [#13](https://github.com/ai-agentopia/agentopia-infra/issues/13) | — | Sprint 3 |
| **4** | F5 Analytics & Cost | [#20](https://github.com/ai-agentopia/agentopia-infra/issues/20) | #36, #37, #38 | F9 (Sprint 1), P2 (Sprint 4) |
| **5** | F6 Governance | [#21](https://github.com/ai-agentopia/agentopia-infra/issues/21) | #39, #40, #41 | F9 (Sprint 1) |
| **5** | P4 Developer Experience | [#15](https://github.com/ai-agentopia/agentopia-infra/issues/15) | — | Sprint 4 |
| **6** | F1 Multi-Agent | [#16](https://github.com/ai-agentopia/agentopia-infra/issues/16) | #24, #25, #26 | Sprint 5 |
| **6** | P3 Multi-tenancy | [#14](https://github.com/ai-agentopia/agentopia-infra/issues/14) | — | Sprint 5 |
| **7** | F7 Marketplace | [#22](https://github.com/ai-agentopia/agentopia-infra/issues/22) | #42, #43 | P4 (Sprint 5) |
| **7** | F8 SaaS Platform | [#23](https://github.com/ai-agentopia/agentopia-infra/issues/23) | #44, #45, #46 | P3 (Sprint 6), F5 (Sprint 4) |
