---
title: "Class-Driven tools.allow Rendering"
---

# Class-Driven `tools.allow` Rendering

**Status:** Landed (H2.5 / WP-H2.5-02, WP-H2.5-06).
**Scope:** `agentopia-infra/charts/agentopia-bot/templates/_helpers.tpl`
**Audience:** Operators inspecting or validating a bot's rendered tool surface
**Related:** [capability-class-rollout-guide](./capability-class-rollout-guide), [runtime-facts-capability-classes-baseline §8](./runtime-facts-capability-classes-baseline)

---

## Overview

Prior to H2.5, the chart contained a hand-written `tools.allow` block that granted a broad tool set to every bot regardless of role. H2.5 replaces that block with a single Helm helper call — `agentopia-bot.toolsAllow` — that renders the correct surface from the bot's `capabilityClass` value.

No operator edits the allowlist directly. The chart is the renderer; bot-config-api is the policy source.

---

## 1. How `agentopia-bot.toolsAllow` Works

The helper reads `.Values.capabilityClass`. When the class matches one of the four known values (`conversant`, `worker`, `orchestrator`, `admin`), it renders the corresponding pre-defined tool list. When `capabilityClass` is empty, it falls back to the legacy path — see §3.

---

## 2. Rendered Surface per Class

The tables below list the exact tools rendered for each class. Each class is cumulative — it includes everything from the classes below it. Tool names are exact string values as they appear in the chart.

### `conversant`

| Tools |
|---|
| `group:fs-read` |
| `knowledge_retrieve` |
| `mem_recall`, `mem_search`, `mem_list` |

`maxToolCalls`: **8**

### `worker` (adds to conversant)

| Tools added |
|---|
| `gov_list_milestones`, `gov_get_milestone` |
| `gov_list_issues`, `gov_get_issue` |
| `gov_get_file_contents`, `gov_search_code`, `gov_list_commits` |
| `wf_status` |

`maxToolCalls`: **12**

### `orchestrator` (adds to worker)

| Tools added |
|---|
| `group:fs` (replaces `group:fs-read` — includes write tools, sandboxed to bot workspace) |
| `gov_audit_milestone`, `gov_audit_repo` |
| `wf_command`, `start_delivery` |
| `sessions_send`, `sessions_list`, `sessions_history`, `sessions_spawn`, `subagents` |
| `relay_send`, `relay_list_bots` |
| `relay_create_thread`, `relay_send_turn`, `relay_read_thread` |
| `relay_checkpoint`, `relay_conclude`, `relay_list_threads` |
| `a2a_discover_agents`, `a2a_send_task`, `a2a_create_thread`, `a2a_send_turn` |
| `gov_create_milestone`, `gov_update_milestone`, `gov_close_milestone` |
| `gov_create_issue`, `gov_update_issue`, `gov_add_issue_comment`, `gov_close_issue` |
| `gov_create_branch`, `gov_create_or_update_file`, `gov_push_files` |
| `gov_create_pull_request`, `gov_update_pr_branch` |
| `gov_get_pull_request`, `gov_get_pr_files`, `gov_get_pr_status` |
| `gov_get_pr_reviews`, `gov_get_pr_comments`, `gov_create_pr_review` |
| `gov_merge_pull_request` |
| `mcp_*` (when `mcpBridge` is enabled — rendered per allowed_tools / denied_tools) |

`maxToolCalls`: **40**

### `admin` (adds to orchestrator)

| Tools added |
|---|
| `admin_inspect` |
| `admin_mutate` |

`maxToolCalls`: **40**

**Note:** `session_status` is absent from every class surface. `group:sessions` and `group:runtime` are also absent — they appear only in the legacy fallback path (§3), not in any class-driven surface.

---

## 3. The Empty `capabilityClass` Fallback Path

When `.Values.capabilityClass` is empty (a bot provisioned before H2.5 that has not yet been re-deployed through bot-config-api), the helper renders the **legacy `wfBridgeRoleKey`-based surface**. This is not a conversant default — it is the full pre-H2.5 hand-written surface.

The legacy surface includes `group:sessions`, `group:fs`, `group:runtime`, the full relay/A2A/gov write surface, and conditionally `wf_command`/`start_delivery` when `wfBridgeRoleKey=orchestrator`. This surface is wider than any single class-driven surface and intentionally preserves existing bot behaviour until the bot is re-deployed with a `capabilityClass` set.

**This fallback path is eliminated by `strictCapabilityClass: true`.** See §4.

---

## 4. `strictCapabilityClass` Toggle

| `strictCapabilityClass` | Behaviour |
|---|---|
| `false` (default) | If `capabilityClass` is set, renders the class surface. If empty, renders the legacy `wfBridgeRoleKey` surface. No enforcement change for existing bots. |
| `true` | If `capabilityClass` is set, renders only the class surface. If `capabilityClass` is empty, the Helm render **fails** with an explicit error — there is no silent fallback. |

The error message when `strictCapabilityClass=true` and `capabilityClass` is empty:

> `strictCapabilityClass=true but capabilityClass is empty for this bot. Either set capabilityClass explicitly (conversant|worker|orchestrator|admin) or disable strictCapabilityClass.`

Flip to `strictCapabilityClass: true` only after running `scripts/diff-tools-allow.sh` and confirming surface parity for every deployed bot (see [capability-class-rollout-guide §3](./capability-class-rollout-guide#3-validating-the-rendered-surface-before-enforcing)).

---

## 5. Validating with `diff-tools-allow.sh`

The infra repo ships `scripts/diff-tools-allow.sh` to compare what the class-driven helper renders against the bot's current hand-written surface.

**Local mode:**

```bash
./scripts/diff-tools-allow.sh --bot <bot-name> --class <class>
```

**Cluster mode** (reads live ArgoCD Application values):

```bash
./scripts/diff-tools-allow.sh --bot <bot-name> --cluster
```

`ADDED` — tools the class grants that the old list did not include. Safe to accept.

`REMOVED` — tools present in the old list that the class does not grant. Review before cutover.

---

## 6. Helm Unit Tests

The chart ships 61 Helm unit tests that assert the exact tool surface rendered for each class. If you modify `_helpers.tpl`, run:

```bash
helm unittest charts/agentopia-bot
```

All 61 tests must pass. A test failure means the rendered surface diverges from the approved baseline. Do not ship a rendering change that breaks these tests.

---

## 7. What the Operator Does Not Own

| Concern | Owner |
|---|---|
| Which tools are in each class | `agentopia-core` (`capability-classes.ts`) |
| Per-bot `capabilityClass` value | `bot-config-api` (set during deploy) |
| `maxToolCalls` per class | `agentopia-core` defaults, emitted into SOUL via bot-config-api |
| R1/R2/R3 runaway-prevention rails | `agentopia-core` and chart values (`loopDetection.enabled`) |

The chart renders policy. It does not define it. If a class surface is wrong, the fix is in `agentopia-core` or `bot-config-api`, not in `_helpers.tpl`.
