---
title: "ADR-009: SA Knowledge Base — Governance, Security, and Compliance"
status: accepted
date: 2026-03-30
decision-makers: CTO
debate: "D2 (#295)"
milestone: "#33 — SA Knowledge Base"
depends-on: "ADR-008 (D1)"
---

# ADR-009: SA Knowledge Base — Governance, Security, and Compliance

## Context

D1 (ADR-008) established:
- Client boundary is the hard isolation wall
- Within-client access is subscription-based, not blanket
- Scope identity: `{client_id}/{scope_name}`
- SA bots are the only first-release consumer

The knowledge system today has **zero authentication** on all routes — any HTTP caller can ingest, search, delete, or list knowledge across any scope. This is not production-safe.

D2 must define the governance model that makes the knowledge system production-ready while preserving D1 constraints.

### Current auth system (code-verified)

- **Principal model**: `Principal(type, user_id, roles)` — types: `admin_user`, `service`
- **Guards**: `require_auth` (session), `require_admin` (type check), `require_dual_auth` (session OR bearer)
- **Session**: Cookie-based, 24h TTL, in-memory store
- **Service auth**: `A2A_INTERNAL_TOKEN` bearer, shared across all services
- **Knowledge routes**: All 10 endpoints mounted without any auth dependency
- **Protected routes**: Only `generate` and `deploy` use `require_auth`
- **Audit infra**: CommandAuditStore (Postgres), AuthZDecision log, structured JSON logging — none covers knowledge ops

## Decision

### 1. Authentication Boundary

All knowledge routes must be authenticated in first release. No exceptions.

| Operation | Auth Method | Allowed Principal |
|---|---|---|
| Document ingestion (upload) | Session (`require_auth`) | Operator (admin_user) |
| Document deletion | Session (`require_auth`) | Operator (admin_user) |
| Scope creation | Session (`require_auth`) | Operator (admin_user) |
| Scope listing (operator) | Session (`require_auth`) | Operator — returns all scopes within requested client |
| Scope listing (bot runtime) | Bearer with bot identity | Service — returns ONLY the bot's explicitly subscribed scopes |
| Scope deletion | Session (`require_auth`) | Operator (admin_user) |
| Knowledge search (operator) | Session (`require_auth`) | Operator (admin_user) |
| Knowledge search (bot runtime) | Bearer with bot identity | Service (with bot_name) |
| Reindex | Session (`require_auth`) | Operator (admin_user) |

**Principle**: Write operations (ingest, delete, reindex, scope CRUD) are operator-only. Read operations (search, list) are available to both operators and bots through their respective auth paths.

### 2. Authorization Model

First release uses a **config-driven subscription model**:

- Bot is assigned `client_id` at deploy time (new field)
- Bot is assigned `knowledge_scopes: list[str]` at deploy time (analogous to `memory_scopes`)
- At query time, the requesting bot's identity is resolved to its registered `client_id` + `knowledge_scopes`
- Access is granted only to subscribed scopes within the bot's client
- Cross-client access: blocked (403)
- Unsubscribed scope within same client: blocked (403)
- Operator (admin_user): can manage all clients and scopes in first release (single-operator model)
- Multi-operator RBAC: deferred beyond first release

### 3. Bot Identity and Client Binding

- `client_id` is a deploy-time config field on the bot (new field in DeployRequest)
- `knowledge_scopes` is a deploy-time config field (list of scope names within the client)
- Fully-qualified scope identity: `{client_id}/{scope_name}` (from D1)
- At runtime, bot proves identity via `bot_name` in request (header or token claim)
- bot-config-api resolves `bot_name` → `client_id` + `knowledge_scopes` from registration
- Operator-managed assignment only — no self-service scope subscription in first release

### 4. Audit Requirements

Minimum auditable event set for first release (structured logging):

| Event | Required Identifiers |
|---|---|
| Document ingested | operator_id, client_id, scope_name, source_filename, chunk_count |
| Document deleted | operator_id, client_id, scope_name, source_filename |
| Scope created | operator_id, client_id, scope_name |
| Scope deleted | operator_id, client_id, scope_name |
| Reindex triggered | operator_id, client_id, scope_name |
| Knowledge search (bot) | bot_name, client_id, scopes_queried, result_count |
| Knowledge search (operator) | operator_id, client_id, scopes_queried, result_count |
| Access denied (wrong client) | requester_id, requested_client, requester_client |
| Access denied (unsubscribed scope) | bot_name, client_id, requested_scope |
| Scope subscription changed | operator_id, bot_name, old_scopes, new_scopes |

Implementation: Structured JSON logs (existing `_JSONFormatter`). Denial events at WARN level. Full DB-persisted audit trail deferred to post-first-release.

### 5. Retention and Deletion

| Aspect | First Release Decision |
|---|---|
| Document deletion | Immediate, permanent. Operator-initiated. |
| Right-to-delete | Supported — delete any document or scope on request |
| Retention window | No mandatory retention. Exists until explicitly deleted. |
| Audit retention | **Provisional** — environment-dependent. Production deployment must define and verify minimum retention before go-live. No infrastructure-backed retention guarantee exists today. Added to #307 rollout verification. |
| Stale scope handling | Surfaced via `list_stale_scopes()`. Operator acts — no auto-purge. |
| Client offboarding | Operator deletes all client scopes. Collections removed. Bot subscriptions cleared. |

Not committed: soft-delete, automated retention, compliance certifications, data residency.

### 6. Provisional Decisions (Pending D3)

| Item | Status | Why Provisional |
|---|---|---|
| Exact enforcement point for bot runtime search | Provisional | D3 decides whether check is in bot-config-api endpoint, gateway plugin, or both |
| Query-level audit shape for runtime calls | Provisional | D3 decides the injection flow — audit wraps whatever path D3 chooses |
| Bot identity propagation mechanism | Provisional | D3 decides whether identity travels in header, token claim, or plugin config |

These items are architecturally constrained (must authenticate, must enforce subscription, must audit) but the exact implementation point depends on D3.

## Alternatives Considered

### Auth posture

| Option | Verdict | Reason |
|---|---|---|
| Fully open routes (current) | Rejected | Not production-safe. Any caller can read/write/delete any scope. |
| Operator-only auth (session) | Rejected | Bot runtime retrieval cannot use session cookies. |
| Mixed human + service auth | Accepted | Operator uses session for write ops. Bot uses bearer for read ops. Matches existing `require_dual_auth` pattern. |

### Authorization style

| Option | Verdict | Reason |
|---|---|---|
| Simple role-based (admin can do everything) | Rejected | Violates D1 subscription-based access. Bearer token would grant access to all scopes. |
| Config-driven scope subscription | Accepted | Mirrors `memory_scopes` pattern. Explicit, operator-managed. Enforces D1 constraints. |
| Dynamic RBAC with permission matrix | Rejected for first release | Over-engineering. Single operator. No multi-operator model needed yet. |

### Search exposure

| Option | Verdict | Reason |
|---|---|---|
| Operators can search directly | Accepted | Operators need to verify knowledge quality, debug retrieval. |
| Bots only | Rejected | Operator needs direct search for management and QA. |
| Both with different permissions | Accepted | Operator searches any scope in any client (admin). Bot searches only subscribed scopes. |

### Audit posture

| Option | Verdict | Reason |
|---|---|---|
| Write-only audit | Rejected | Cannot trace unauthorized read attempts or debug retrieval issues. |
| Query + write audit | Accepted | Captures all mutation and access events. Denial events surfaced. |
| Full request/response audit | Rejected for first release | Excessive for first release. Structured event logging is sufficient. |

## Consequences

### Downstream constraints

- **D3 (#296)**: Must respect auth boundary — runtime retrieval must authenticate and carry bot identity. Enforcement point and identity mechanism are provisional but the constraint is final. Bot can only retrieve from subscribed scopes.
- **#301**: Must implement `client_id` and `knowledge_scopes` on bot registration. Scope model must enforce client-bounded subscriptions. Knowledge routes must add auth guards.
- **#305**: Operator UI must show client → scope navigation. Scope subscription assignment is part of bot deploy/update flow. Operator can search directly for QA purposes.
- **#307**: Production verification must prove: auth on all knowledge routes, cross-client access blocked, unsubscribed-scope access blocked, audit events captured for all operations.

### What D2 does NOT decide

- Runtime retrieval mechanism (D3)
- Document versioning model (D4)
- Ingestion paths and lifecycle (D5)
- Quality evaluation (D7)
- Multi-operator RBAC (deferred beyond first release)
- Compliance certifications (deferred)
