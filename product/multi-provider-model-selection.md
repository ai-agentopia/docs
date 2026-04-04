---
title: "Multi-Provider Model Selection"
description: "Run different AI models per agent — mix providers for cost, capability, and resilience"
---


Each Agentopia agent can use a different LLM provider and model. A team can include an orchestrator on Claude Opus for deep reasoning, workers on Claude Sonnet for cost efficiency, and a reviewer on a different provider entirely.

---

## Supported Providers

| Provider | Example Models |
|---|---|
| **Anthropic** | Claude Opus 4.6, Claude Sonnet 4.6 |
| **OpenAI** | GPT-5.2, GPT-4.1 |
| **Google** (via OpenRouter) | Gemini 2.0 Flash |
| **Fireworks AI** | Kimi K2.5 Turbo |

Additional providers can be added by registering their API key and model configuration.

---

## How It Works

### Deploy-Time Selection

When creating a bot, the operator selects a provider and model. This selection is written into the agent's configuration and honored at startup. Each bot runs on exactly one primary model.

### Post-Deploy Provider/Model Change

Operators can change a bot's provider and model after deployment from the Bot Settings page. The change patches the bot's ArgoCD Application configuration and triggers an automatic pod rollout via GitOps.

<Warning>
  Changing provider/model restarts the bot pod. Expect brief downtime (~30 seconds). Ensure the API key for the target provider is configured in Vault before switching.
</Warning>

The provider/model selector in the UI loads from the platform's provider catalog API. The operator sees human-readable labels (e.g. "Claude Opus 4.6") and the platform submits the full catalog value (e.g. `anthropic/claude-opus-4-6`) to the backend.

### Runtime Failover

If the primary model fails (rate limit, auth error, timeout), the agent automatically tries configured fallback models in order. Fallbacks are explicit — only pre-configured models are attempted.

### Per-Agent Override

Individual agents within a bot can override the global model selection. For example, a code review agent within a bot can use a different model than the main conversation agent.

### Session Override

Users can switch models mid-conversation using a command. The override applies to the current session only and reverts to the configured model when the session ends.

---

## Provider Catalog

The platform exposes a provider catalog API that returns all supported providers with their models and display names. The web app uses this to populate the provider dropdown during bot creation.

---

## Key Management

All LLM API keys are managed through a centralized secret store:

- **Single source of truth** — one place to add or rotate keys
- **Auto-injection** — agent pods fetch keys at startup
- **Audit trail** — key access is logged and auditable

---

## Limitations

- **Model capability differences** — not all models support all features (vision, tool use, etc.). The platform does not validate capability compatibility at deploy time.
- **Cross-provider fallback** — runtime failover works best within the same provider. Cross-provider fallbacks require both providers to have valid keys configured.
- **Cost visibility** — token usage is tracked per session, but per-provider cost dashboards are not yet available.
