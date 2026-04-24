---
title: "H2.5 Capability-Class Rollout Guide"
---

# H2.5 Capability-Class Rollout Guide

**Status:** Implementation complete (H2.5). Operator-facing runbook.
**Scope:** `agentopia-protocol` (bot-config-api), `agentopia-infra` (Helm chart)
**Audience:** Operators and platform engineers responsible for bot provisioning and migration
**Canonical architecture:** [runtime-facts-capability-classes-baseline](./runtime-facts-capability-classes-baseline)

---

## Overview

H2.5 replaces hand-written `tools.allow` blocks in the Helm chart with a class-driven rendering model. Each bot is assigned a `capabilityClass` value on the control plane; the chart resolves the full tool surface from that single field.

This guide covers the four-step operator workflow:

1. Understand the four-rung class ladder
2. Understand how `capabilityClass` is assigned during deploy
3. Validate the rendered surface with `diff-tools-allow.sh` before enforcing
4. Flip `strictCapabilityClass` to enforce

---

## 1. The Four-Rung Class Ladder

Classes are strictly cumulative. Each class adds tools on top of the class below it.

| Class | `maxToolCalls` / turn | Intent |
|---|---|---|
| `conversant` | 8 | Answer questions using read access, retrieval, memory |
| `worker` | 12 | Conversant + read-only domain operations (code review, QA, SA) |
| `orchestrator` | 40 | Worker + cross-session coordination, write tools, relay/A2A/MCP |
| `admin` | 40 | Operator surface: `admin_inspect` and `admin_mutate` only (on top of orchestrator) |

**Default for any new bot:** `conversant`. No bot auto-promotes. `admin` is opt-in by explicit control-plane annotation.

Full tool surface per class is defined in the baseline §8 and rendered by `agentopia-bot.toolsAllow` in the chart (see [tools-allow-rendering](./tools-allow-rendering)).

---

## 2. How `capabilityClass` Is Assigned

`resolve_capability_class()` in `bot-config-api/src/domain/capability_class/migration.py` runs automatically inside the bot deploy flow (`POST /api/v1/bots/deploy`). Operators do not call it directly.

The function applies this mapping:

| Explicit `capability_class` in deploy request | Resolved class |
|---|---|
| `"admin"` | `admin` |
| `"orchestrator"` | `orchestrator` |
| `"worker"` | `worker` |
| `"conversant"` | `conversant` |
| *(not set)* | Derived from `wfBridgeRoleKey` (see table below) |

When `capability_class` is not set explicitly in the deploy request, the function falls back to `wfBridgeRoleKey`:

| Existing `wfBridgeRoleKey` | Derived `capabilityClass` |
|---|---|
| `orchestrator` | `orchestrator` |
| `worker` | `worker` |
| `reviewer` | `worker` |
| *(unset / anything else)* | `conversant` |

`admin` is never derived automatically. It must be declared explicitly in the deploy request.

The resolved class is written to:
- `spec.source.helm.valuesObject.capabilityClass` in the bot's ArgoCD Application (controls chart rendering)
- `metadata.annotations["agentopia/capability-class"]` on the Application (read-back mirror)

**When does a bot get its class set?** On next deploy or update through bot-config-api. Bots provisioned before H2.5 that have not been re-deployed still have an empty `capabilityClass` in their Application and fall back to the legacy chart path (see §3 and [tools-allow-rendering §3](./tools-allow-rendering#3-the-empty-capabilityclass-fallback-path)).

---

## 3. Validating the Rendered Surface Before Enforcing

Before setting `strictCapabilityClass: true`, validate that the class-driven surface matches expectations using `scripts/diff-tools-allow.sh` from the infra repo.

**Local mode** (reads your local chart and a specified class):

```bash
./scripts/diff-tools-allow.sh --bot <bot-name> --class <class>
```

**Cluster mode** (reads the live ArgoCD Application values):

```bash
./scripts/diff-tools-allow.sh --bot <bot-name> --cluster
```

Example output:

```
BOT: delivery-bot   CLASS: orchestrator   MODE: diff
ADDED (class-driven adds these):
  + relay_broadcast
REMOVED (class-driven removes these):
  - group:sessions   ← covered by named sessions_* tools in the class surface
OK:  tools present in both surfaces: 28
```

`ADDED` — tools the class grants that were not in the old hand-written list. Safe by definition.

`REMOVED` — tools the old hand-written list had that the class does not grant. Review each one:
- If the tool is legitimately required by the bot's role, raise the bot's `capabilityClass`
- If the tool was an artifact of the flat allowlist, the removal is correct

Run this for every bot before enabling `strictCapabilityClass`.

---

## 4. Cutover: Enabling `strictCapabilityClass`

When `diff-tools-allow.sh` confirms no unexpected removals for a bot, enable enforcement by setting `strictCapabilityClass: true` in the bot's Helm values. This happens through the bot-config-api deploy flow or a targeted ArgoCD Application patch.

With `strictCapabilityClass: true`:
- Only the class-driven surface from `agentopia-bot.toolsAllow` is rendered
- The legacy `wfBridgeRoleKey` fallback path is eliminated
- If `capabilityClass` is empty when `strictCapabilityClass=true`, the Helm render fails with an explicit error — it does not fall back to any default

**Recommended cutover order:** `conversant` bots first (smallest surface change), then `worker`, then `orchestrator`. `admin` bots last; verify with the full Admin tool contract (see [admin-tool-contract](./admin-tool-contract)).

---

## 5. Rollout Checklist

```
[ ] For each bot in the migration batch:
    [ ] Confirm capabilityClass annotation on the ArgoCD Application
        (re-deploy via bot-config-api if not yet set)
    [ ] Run: ./scripts/diff-tools-allow.sh --bot <name> --cluster
    [ ] Review REMOVED entries — no legitimate capability loss
    [ ] Raise capabilityClass if a removal is incorrect, then re-check
    [ ] Set strictCapabilityClass: true (via next deploy or Application patch)
    [ ] Verify bot responds normally on a smoke turn after cutover
[ ] Monitor first 24 hours for unexpected tool-block events
```

---

## 6. Known Constraints and Pending Items

| Item | Status |
|---|---|
| `wfBridgeRoleKey` removal from codebase | Deferred — removed after `capabilityClass` migration is stable for two release cycles (Phase 4) |
| `tool_calls_per_turn{capability_class}` histogram metric | Deferred (Phase 4 telemetry) |
| Conversant and Worker `maxToolCalls` budget review | Pending two weeks of p95 telemetry after full fleet migration |
| Additive capability bundles orthogonal to the ladder | Explicitly deferred — requires a dedicated ADR with a motivating case |
