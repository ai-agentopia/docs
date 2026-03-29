---
title: "Multi-Provider Model Selection"
---

# Multi-Provider Model Selection

> Each Agentopia bot can use a different LLM provider and model.
> Provider is selected at deploy time and honored at gateway bootstrap.
> Last updated: 2026-03-29

---

## 1. Overview

Agentopia supports running bots on different LLM providers simultaneously. A team can include:
- An orchestrator on Claude Opus 4.6 (deep reasoning)
- Workers on Claude Sonnet 4.6 (fast, cost-effective)
- A code reviewer on Fireworks Kimi K2.5 Turbo (alternative provider)

The provider is selected per-bot at deploy time. The gateway honors this selection at bootstrap and uses configured fallbacks at runtime.

---

## 2. Supported Providers

| Provider | Config Prefix | Models | Key Source |
|---|---|---|---|
| **Anthropic** | `anthropic/` | claude-opus-4-6, claude-sonnet-4-6 | Vault |
| **OpenAI** | `openai/` | gpt-5.2, gpt-4.1 | Vault |
| **OpenRouter** | `openrouter/` | google/gemini-2.0-flash-001, etc. | Vault |
| **Fireworks AI** | `fireworks/` | accounts/fireworks/routers/kimi-k2p5-turbo | Vault |

All API keys are stored in HashiCorp Vault. Gateway pods fetch keys at startup via the entrypoint script.

---

## 3. How Model Selection Works

### 3.1 Deploy Time

When a bot is created through the UI or API, the operator selects a provider. The deploy pipeline writes this into the bot's OpenClaw configuration:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-6",
        "fallbacks": ["anthropic/claude-sonnet-4-6"]
      }
    }
  }
}
```

For non-Anthropic providers (e.g. Fireworks), an additional `models.providers` block is injected to register the provider with the gateway runtime.

### 3.2 Gateway Bootstrap

At startup, the gateway resolves the configured model through a deterministic chain:

1. Read `agents.defaults.model.primary` from config
2. If found → parse `provider/model` format → use it
3. If not found → fall back to hardcoded default (`anthropic/claude-opus-4-6`)

The bootstrap model is logged:
```
[gateway] agent model: fireworks/accounts/fireworks/routers/kimi-k2p5-turbo
```

### 3.3 Runtime Failover

When the primary model fails (rate limit, auth error, network timeout), the gateway tries the configured fallback chain:

1. Try each model in `agents.defaults.model.fallbacks[]`
2. Try the configured primary as a last resort (if session overrode the model)

The fallback chain is explicit — only models listed in `fallbacks` are attempted. The hardcoded Opus 4.6 default only enters when no configuration exists at all.

### 3.4 Per-Agent Override

Individual agents in `agents.list[]` can override the global model:

```json
{
  "agents": {
    "list": [{
      "id": "code-reviewer-agent",
      "model": {
        "primary": "fireworks/accounts/fireworks/routers/kimi-k2p5-turbo"
      }
    }]
  }
}
```

Agent-specific `model.primary` takes precedence over `agents.defaults.model.primary`.

### 3.5 Session Override

Users can switch models mid-conversation using the `/model` command. This creates a per-session override that reverts to the configured model when the session ends.

---

## 4. Provider Catalog API

The platform exposes a provider catalog for UI consumption:

```
GET /api/v1/providers
```

Returns all supported providers with their models, display names, and configuration requirements. The UI uses this to populate the provider dropdown during bot creation.

---

## 5. Key Management

All LLM API keys are managed through HashiCorp Vault:

- **Single source of truth** — one `vault kv put` command to add/rotate a key
- **Auto-injection** — gateway pods fetch keys at startup via entrypoint script
- **No K8s Secrets** for LLM keys — Vault provides audit trail and rotation capability

Adding a new provider requires:
1. Add the API key to Vault
2. Update the Helm chart template (if provider needs a `models.providers` block)
3. Add to the provider catalog in `services/providers.py`

---

## 6. Verification

After deploying a bot with a specific provider, verify the selection:

**Check gateway log:**
```
[gateway] agent model: <expected provider/model>
```

**Check ConfigMap:**
```bash
kubectl get configmap agentopia-config-<bot> -o jsonpath='{.data.openclaw\.json}' | jq '.agents.defaults.model'
```

---

## 7. Limitations

- **Model availability**: Not all models support all features (vision, tool use, etc.). The platform does not validate capability compatibility at deploy time.
- **Fallback cross-provider**: Runtime failover works best within the same provider. Cross-provider fallbacks require both providers to be configured with valid keys.
- **Cost visibility**: Token usage is tracked per-session but per-provider cost dashboards are not yet implemented.
