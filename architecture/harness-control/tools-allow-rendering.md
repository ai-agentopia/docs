---
title: "Class-Driven tools.allow Rendering"
---

# Class-Driven `tools.allow` Rendering

**Status:** Landed (H2.5 / WP-H2.5-02, WP-H2.5-06).
**Scope:** `agentopia-infra/charts/agentopia-bot`
**Audience:** Operators inspecting or validating a bot's rendered tool surface
**Related:** [capability-class-rollout-guide](./capability-class-rollout-guide), [runtime-facts-capability-classes-baseline §8](./runtime-facts-capability-classes-baseline)

---

## Overview

Prior to H2.5, the chart contained a 37-line hand-written `tools.allow` block that granted the same tool set to every bot regardless of role. H2.5 replaces that block with a single Helm helper call — `agentopia-bot.toolsAllow` — that renders the correct surface from the bot's `capabilityClass` value.

No operator edits the allowlist directly. The chart is the renderer; bot-config-api is the policy source.

---

## 1. How `agentopia-bot.toolsAllow` Works

The helper is defined in `agentopia-infra/charts/agentopia-bot/templates/_helpers.tpl`. When the chart renders:

1. It reads `.Values.capabilityClass` (set by bot-config-api during reconcile)
2. It selects the pre-defined tool set for that class
3. It emits the YAML `tools.allow` block in the ConfigMap

The helper contains no `hasPrefix` logic and no conditional hand-written lists. The class value is the sole selector.

If `.Values.capabilityClass` is empty or unrecognised, the helper renders the `conversant` surface as a safe fallback. This matches the baseline rule that the minimum default for any bot is Conversant.

---

## 2. Rendered Surface per Class

The tables below list the tool groups and named tools rendered for each class. Each class is cumulative — it includes everything from the classes below it.

### `conversant`

| Tools |
|---|
| `group:fs-read` (`read`, `list_dir`, `find`, `grep`) |
| `knowledge_retrieve` |
| `mem_recall` and other read-only memory tools |

`maxToolCalls`: **8**

### `worker` (adds to conversant)

| Tools added |
|---|
| `gov_list_*` |
| `gov_get_*` |
| `gov_search_code` |
| `gov_get_file_contents` |
| `wf_status` |

`maxToolCalls`: **12**

### `orchestrator` (adds to worker)

| Tools added |
|---|
| `group:fs` write tools (`write`, `edit`, `move`, `delete`, sandboxed to bot workspace) |
| `sessions_send`, `sessions_list`, `sessions_history`, `sessions_spawn` |
| `subagents` |
| `relay_*` suite |
| `a2a_*` suite |
| `wf_command`, `start_delivery` |
| `gov_create_*`, `gov_update_*`, `gov_create_branch`, `gov_create_or_update_file` |
| `gov_push_files`, `gov_create_pull_request`, `gov_add_issue_comment`, `gov_close_issue` |
| `gov_merge_pull_request`, `gov_create_pr_review`, `gov_get_pr_*` |
| `mcp_*` (when `mcpBridge` is enabled for the bot) |

`maxToolCalls`: **40**

### `admin` (adds to orchestrator)

| Tools added |
|---|
| `admin_inspect` |
| `admin_mutate` |

`maxToolCalls`: **40**

**Note:** `session_status` is absent from every class surface. It is removed from agent-callable registration entirely. Only the operator `/status` UI path uses the internal card formatter.

---

## 3. `strictCapabilityClass` Toggle

The chart supports a shadow mode for migration safety.

| `strictCapabilityClass` | Behaviour |
|---|---|
| `false` (default) | Both the hand-written allowlist and the class-driven surface are computed. A diff is emitted to the sync log. The hand-written surface remains active. |
| `true` | Only the class-driven surface is rendered. The hand-written fallback path is disabled. |

Set `strictCapabilityClass: true` after validating the shadow diff (see §4).

---

## 4. Validating the Rendered Surface with `diff-tools-allow.sh`

The infra repo ships `scripts/diff-tools-allow.sh` to inspect what the class-driven helper renders and compare it to the bot's current hand-written allowlist.

**Local mode** — runs against your local chart and specified class:

```bash
./scripts/diff-tools-allow.sh --bot <bot-name> --class <class>
```

**Cluster mode** — reads the live ArgoCD Application values and runs the same diff:

```bash
./scripts/diff-tools-allow.sh --bot <bot-name> --cluster
```

Example output:

```
BOT: delivery-bot   CLASS: orchestrator   MODE: shadow
ADDED (class-driven adds these):
  + relay_broadcast
REMOVED (class-driven removes these):
  - group:sessions   ← already covered by sessions_* named tools
OK:  tools present in both surfaces: 28
```

`ADDED` entries are tools the class grants that the hand-written list did not include. These are safe additions by definition — the class contract authorises them.

`REMOVED` entries are tools present in the old hand-written list that the class does not grant. Review each one:
- If the tool is legitimately needed, the bot's class may need to be raised
- If the tool was mistakenly included in the old list, the removal is correct

---

## 5. Interpreting the 61-Test Rendering Suite

The chart ships 61 Helm unit tests that assert the exact tool surface rendered for each class. If you modify the `_helpers.tpl` helper, run:

```bash
helm unittest charts/agentopia-bot
```

All 61 tests must pass. A test failure means the rendered surface diverges from the approved baseline. Do not ship a rendering change that breaks these tests.

---

## 6. What the Operator Does Not Own

| Concern | Owner |
|---|---|
| Which tools are in each class | `agentopia-core` (`capability-classes.ts`) |
| Per-bot `capabilityClass` value | `bot-config-api` |
| `maxToolCalls` per class | `agentopia-core` defaults, emitted into SOUL via bot-config-api |
| R1/R2/R3 runaway-prevention rails | `agentopia-core` and chart values (`loopDetection.enabled`) |

The chart renders policy. It does not define it. If a class surface is wrong, the fix is in `agentopia-core` or `bot-config-api`, not in `_helpers.tpl`.
