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

H2.5 replaces hand-written `tools.allow` blocks in the Helm chart with a class-driven rendering model. Each bot is assigned a `capabilityClass` value on the control plane; the chart resolves the full tool surface from that single field. No Helm values editing is needed per bot.

This guide covers the four-step operator workflow:

1. Understand the four-rung class ladder
2. Determine each bot's class from its current role
3. Run shadow-diff to validate the rendered surface before enforcement
4. Flip `strictCapabilityClass` to enforce

---

## 1. The Four-Rung Class Ladder

Classes are strictly cumulative. Each class adds tools on top of the class below it.

| Class | `maxToolCalls` / turn | Intent |
|---|---|---|
| `conversant` | 8 | Answer questions using read access, retrieval, memory |
| `worker` | 12 | Conversant + read-only domain operations (code review, QA, SA) |
| `orchestrator` | 40 | Worker + cross-session coordination, write tools, relay/A2A/MCP |
| `admin` | 40 | Operator surface: `admin_inspect` and `admin_mutate` only |

**Default for any new bot:** `conversant`. No bot auto-promotes. `admin` is opt-in by explicit control-plane annotation.

Full tool surface per class is defined in the baseline §8 and rendered by `agentopia-bot.toolsAllow` in the chart (see [tools-allow-rendering](./tools-allow-rendering)).

---

## 2. Determining a Bot's Class

bot-config-api computes `capabilityClass` automatically from existing bot metadata during reconcile. The migration function `resolve_capability_class()` in `bot_config_api/domain/capability_class/migration.py` applies this mapping:

| Existing `wfBridgeRoleKey` | Resolved `capabilityClass` |
|---|---|
| `orchestrator` | `orchestrator` |
| `worker` | `worker` |
| `reviewer` | `worker` |
| *(unset / anything else)* | `conversant` |

`admin` is never derived automatically. It must be declared explicitly in the bot's config payload.

You can inspect the resolved class for any bot via the diagnostics endpoint:

```
GET /api/bots/{bot_id}/capability-diagnostics
```

The response includes the resolved class, the source field (`wfBridgeRoleKey` or explicit), and the list of tools the bot will gain or lose relative to its current chart rendering.

---

## 3. Shadow-Diff Window

Before `strictCapabilityClass` is enabled, the chart runs in shadow mode: both the hand-written allowlist and the class-driven allowlist are computed, and a diff is emitted to the bot's ArgoCD sync log. No enforcement change occurs.

To inspect the diff for a specific bot without waiting for a sync cycle, use the shadow-diff script from the infra repo:

**Local mode** (reads your local chart + current values):

```bash
./scripts/diff-tools-allow.sh --bot <bot-name> --class <class>
```

**Cluster mode** (reads live ArgoCD Application values):

```bash
./scripts/diff-tools-allow.sh --bot <bot-name> --cluster
```

Output format:

```
BOT: <bot-name>   CLASS: worker   MODE: shadow
ADDED (class-driven adds these):
  + gov_list_issues
  + gov_get_pr_files
REMOVED (class-driven removes these):
  - group:sessions
  - relay_send
OK:  tools present in both surfaces: 14
```

Tools in `REMOVED` represent capabilities the bot had via the hand-written allowlist that its class does not grant. Review each one before enforcing. If the removal is wrong, the bot's declared class may need to be raised.

---

## 4. The `reconcile-capability` Endpoint

`bot-config-api` exposes a `reconcile-capability` endpoint modelled on the existing `reconcile-routing` endpoint. Calling it:

1. Re-runs `resolve_capability_class()` for the bot
2. Patches the bot's ArgoCD Application `valuesObject.capabilityClass`
3. Patches the `agentopia/capability-class` annotation on the Application for read-back
4. Emits a diagnostic listing tools added or dropped vs the previous render
5. Returns the new class and the delta

```
POST /api/bots/{bot_id}/reconcile-capability
```

You do not need to call this manually on a normal deploy — bot-config-api calls it automatically during bot creation and update flows. Call it explicitly when:
- You have changed a bot's declared role and want the chart to reflect the new class immediately
- You want to inspect the delta without waiting for the next ArgoCD sync
- You are validating a migration batch

---

## 5. Cutover: Enabling `strictCapabilityClass`

When the shadow-diff window shows no unexpected removals, enable enforcement:

```yaml
# In the bot's Helm values (via bot-config-api reconcile or direct ArgoCD patch):
strictCapabilityClass: true
```

With `strictCapabilityClass: true`:
- The class-driven `tools.allow` surface is the only surface rendered
- The hand-written allowlist fallback path is disabled
- Any tool not in the class surface is blocked at the Helm rendering level, before the runtime sees it

**Recommended rollout order:** `conversant` bots first (smallest surface change), then `worker`, then `orchestrator`. `admin` bots should be migrated last and verified with the full Admin tool contract (see [admin-tool-contract](./admin-tool-contract)).

---

## 6. Diagnostics Output Reference

The `capability-diagnostics` endpoint and the `reconcile-capability` response both emit the same diagnostic record shape:

```json
{
  "bot_id": "my-bot",
  "capability_class": "worker",
  "source": "wfBridgeRoleKey",
  "strict_mode": false,
  "tools_added": ["gov_list_issues", "gov_get_pr_files", "wf_status"],
  "tools_removed": ["group:sessions", "relay_send"],
  "tools_retained": 14,
  "shadow_diff_available": true
}
```

`tools_removed` lists tools the bot had in its previous allowlist that the class does not grant. These are the items to review before cutover.

---

## 7. Rollout Checklist

```
[ ] Run capability-diagnostics for each bot in the migration batch
[ ] Review tools_removed entries — confirm no legitimate capability loss
[ ] Raise bot's capabilityClass if a removal is incorrect
[ ] Run diff-tools-allow.sh --cluster to confirm shadow render matches expectations
[ ] POST /reconcile-capability for each bot to land the capabilityClass annotation
[ ] Enable strictCapabilityClass: true per bot (or per wave)
[ ] Verify bot responds normally on a smoke turn after cutover
[ ] Monitor tool_calls_per_turn metric for the class during the first 24 hours
```

---

## 8. Known Constraints and Pending Items

| Item | Status |
|---|---|
| `wfBridgeRoleKey` removal from codebase | Deferred — removed after `capabilityClass` migration is stable for two release cycles (Phase 4) |
| `tool_calls_per_turn{capability_class}` histogram metric | Deferred (Phase 4 telemetry) |
| Conversant and Worker `maxToolCalls` budget review | Pending two weeks of p95 telemetry after full fleet migration |
| Additive capability bundles orthogonal to the ladder | Explicitly deferred — requires a dedicated ADR with a motivating case |
