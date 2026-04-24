---
title: "Admin Tool Contract (H2.5)"
---

# Admin Tool Contract (H2.5)

**Status:** Landed — `admin_inspect` and `admin_mutate` are live in `agentopia-core`. Two `admin_mutate` actions are pending Phase 2.5 (see §5).
**Scope:** `agentopia-core/src/agents/tools/admin-tools.ts`
**Audience:** Platform engineers and operators working with Admin-class bots
**Canonical architecture:** [runtime-facts-capability-classes-baseline §7](./runtime-facts-capability-classes-baseline)

---

## Overview

H2.5 removes `session_status` from agent-callable registration and replaces the admin surface with two purpose-built tools:

- `admin_inspect` — read-only runtime and session introspection
- `admin_mutate` — explicit privileged writes under a closed action enum

Both tools are available **only** to bots whose `capabilityClass` is `admin`. No other class has access. Neither tool is registered unless the bot's capability class is exactly `"admin"`.

---

## 1. `admin_inspect`

### Purpose

Read-only runtime and session introspection. Use this tool to retrieve current session state, account identity, or platform runtime details without triggering any write.

### Guarantee

`admin_inspect` is **side-effect-free**. It never writes session state, never persists overrides, and never invalidates caches. An Admin bot may call it repeatedly without risk of state mutation.

### Schema

```typescript
{
  target: "session" | "provider" | "account" | "cache" | "quota",
  filter?: string   // optional — session key, provider ID, or other context
}
```

### Targets

| Target | What it returns | Phase 2.5 status |
|---|---|---|
| `session` | Current session store entries (redacted to model/provider fields). With `filter`: looks up a specific session key. | Live |
| `account` | Bot identity (`agentId`, `sessionKey`). | Live |
| `provider` | Agent ID plus a note pointing to bot-config-api for the full model catalog. | Live (minimal) — full catalog in Phase 2.5 |
| `cache` | Provider usage cache state. | Stub — returns a note. Full data in Phase 2.5. |
| `quota` | Per-provider usage window summary. | Stub — returns a note. Full data in Phase 2.5. |

### R1 Cap

Default R1 cap: **3 calls per turn per unique `{target, filter}` args hash**. The cap is args-hash-based, not name-only. This means inspecting three different targets in a single turn is permitted; calling the same target three times with the same filter is not.

### Example call

```json
{
  "target": "session",
  "filter": "my-session-key-123"
}
```

### Example response

```json
{
  "sessionKey": "my-session-key-123",
  "found": true,
  "entry": {
    "model": "claude-sonnet-4-6",
    "provider": "anthropic"
  }
}
```

---

## 2. `admin_mutate`

### Purpose

Explicit privileged write actions. Every action is a member of a closed enum; free-form strings never reach execution. Every call is audited.

### Schema

```typescript
{
  action: "set_session_model" | "reset_session_model"
        | "clear_provider_usage_cache" | "invalidate_session_memory_cache",

  // Action-specific fields (only the fields required by the action need be provided):
  sessionKey?:       string,  // required for session-scoped actions
  providerModelRef?: string,  // required for set_session_model ("provider/model" format)
  provider?:         string,  // required for clear_provider_usage_cache
  accountId?:        string,  // optional for clear_provider_usage_cache
}
```

### R1 Cap

**1 call per turn (name-only cap, any action).** This cap applies regardless of which action is called or what arguments are supplied. An Admin bot may not call `admin_mutate` more than once in a single user turn.

### Closed Action Enum

#### `set_session_model`

Sets the provider and model for a specific session, overriding the default for subsequent turns in that session.

Required fields: `sessionKey`, `providerModelRef`

`providerModelRef` must be in `"provider/model"` format (e.g. `"anthropic/claude-sonnet-4-6"`). The handler validates the format and rejects free-form strings.

**Status: Live.**

#### `reset_session_model`

Clears any model override for a session, restoring the bot's default routing.

Required fields: `sessionKey`

Deletes `provider`, `model`, and `providerOverride` from the session entry.

**Status: Live.**

#### `clear_provider_usage_cache`

Invalidates the provider's usage cache for the given provider and optional account.

Required fields: `provider`. Optional: `accountId`.

**Status: Pending Phase 2.5.** Calling this action currently throws a `ToolInputError` with message:
> `clear_provider_usage_cache is not yet implemented. Durable cache invalidation API lands in Phase 2.5.`

The enum shape is final. The handler is a stub.

#### `invalidate_session_memory_cache`

Evicts the gateway-owned session memory cache entry for a session.

Required fields: `sessionKey`.

**Status: Pending Phase 2.5.** Calling this action currently throws a `ToolInputError` with message:
> `invalidate_session_memory_cache is not yet implemented. Gateway-owned cache eviction API lands in Phase 2.5.`

The enum shape is final. The handler is a stub.

---

## 3. Audit Record

Every `admin_mutate` call emits a structured audit record, regardless of outcome (success or failure). The record shape is stable.

```json
{
  "actor":     "<session key of the Admin bot>",
  "action":    "<action enum value>",
  "argsHash":  "<16-char SHA-256 prefix of canonicalised args>",
  "before":    { "<state snapshot before mutation>" },
  "after":     { "<state snapshot after mutation>" },
  "outcome":   "success" | "failure",
  "timestamp": "<ISO 8601 UTC>",
  "error":     "<error message, only present on failure>"
}
```

**Current emit path:** Records are emitted to the agentopia-core subsystem log via `log.info("ADMIN_MUTATE_AUDIT ...")`. The JSON shape is stable and is the same shape the durable sink will ingest.

**Durable sink (protocol side — landed):** `bot-config-api` exposes an append-only audit endpoint at `POST /api/v1/admin/audit/records` (router: `admin_audit.py`, mounted in H2.5). The endpoint enforces actor-binding rules (actor must have an active RoleRegistry binding and `actor_capability_class` must be `"admin"`). Read access is at `GET /api/v1/admin/audit/records`. Liveness is at `GET /api/v1/admin/audit/status`.

**Gap tracked in `#477`:** The agentopia-core `admin_mutate` handler does not yet call the protocol-side sink — it only logs locally. `#477` wires the core handler to `POST /api/v1/admin/audit/records`. Until `#477` closes, the audit trail is subsystem-log-only, and `admin_mutate` must not be used in any production environment where a durable audit trail is required.

A new mutation shape is added by introducing a new enum value and a dedicated audit event type — never by relaxing the cap or extending the flat schema with free-form fields.

---

## 4. Adding a New `admin_mutate` Action

Adding a new action requires:

1. Add the new action string to the `ADMIN_MUTATE_ACTIONS` tuple in `admin-tools.ts`
2. Add the required parameters to `AdminMutateSchema` (as `Type.Optional` fields)
3. Add a `case` branch to the `switch` block with handler logic
4. Add a dedicated audit event type for the new action
5. Update this contract doc to reflect the new action

Do not extend the R1 cap or change the flat schema shape (no `anyOf`/`oneOf`). The schema guardrail is explicit: the tool input schema must remain a flat `Type.Object`.

---

## 5. Phase 2.5 Status Summary

| Item | Landed | Pending |
|---|---|---|
| `admin_inspect` — `session` target | ✅ | — |
| `admin_inspect` — `account` target | ✅ | — |
| `admin_inspect` — `provider` target (minimal) | ✅ | Full model catalog (#477) |
| `admin_inspect` — `cache` target | ⏳ | Full data in Phase 2.5 (#477) |
| `admin_inspect` — `quota` target | ⏳ | Full data in Phase 2.5 (#477) |
| `admin_mutate` — `set_session_model` | ✅ | — |
| `admin_mutate` — `reset_session_model` | ✅ | — |
| `admin_mutate` — `clear_provider_usage_cache` | ⏳ | Handler + durable cache API (#477) |
| `admin_mutate` — `invalidate_session_memory_cache` | ⏳ | Handler + gateway cache eviction (#477) |
| Audit record shape | ✅ (stable) | — |
| Durable audit sink — protocol side (`POST /api/v1/admin/audit/records`) | ✅ landed in H2.5 | — |
| Core → protocol sink wiring (`admin_mutate` calls the endpoint) | ⏳ | #477 — gates production use of `admin_mutate` |

---

## 6. What Was Removed

`session_status` is **not re-exposed** through the Admin class or any other class. Its status-card formatter is retained as an internal function called by the operator `/status` UI path only. No capability class grants an agent-callable path to `session_status`.
