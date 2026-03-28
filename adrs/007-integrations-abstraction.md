---
title: "ADR-007: Integrations Abstraction for Bot Management"
---

# ADR-007: Integrations Abstraction for Bot Management

**Status**: Accepted
**Date**: 2026-03-25
**Context**: Milestone #2 — Bot Management Native Migration

## Decision

Use **Integrations** as the top-level information architecture for bot-connected external services.

V1 implements **GitHub MCP** only. The abstraction must accommodate future integrations (Slack, custom MCP servers, Skills, Tools) without requiring IA rewrite.

## Vocabulary

| Term | Meaning | Used in |
|---|---|---|
| **Integration** | An external service connected to a bot via MCP or other protocol | UI sections, API naming |
| **GitHub MCP** | The specific MCP server providing GitHub tools (PR, branch, file ops) | V1 integration |
| **Credential** | A per-bot secret (PAT, token) stored as K8s Secret | Settings, create flow |
| **Credential status** | `present` or `missing` — never echoed | UI display |
| **Enabled** | Integration is configured for a bot (MCP config exists) | Settings toggle |
| **Partial success** | Bot deployed but integration setup failed | Create flow result |

## Component Naming

### Routes
- `/bots/create` — Step 2 includes "Integrations" section
- `/bots/:botId/settings` — contains "Integrations" tab/section

### React Components
- `IntegrationsSection` — shared container for both create and settings
- `GitHubMcpToggle` — create-time opt-in (checkbox + PAT input)
- `GitHubMcpSettings` — settings-time config (enable/disable, credential save/delete, status)

### API Endpoints (existing, no changes needed)
- `GET /api/v1/bots/{name}/mcp` — per-bot MCP config
- `GET /api/v1/bots/{name}/mcp/credentials` — credential statuses
- `PUT /api/v1/bots/{name}/mcp/credentials/{server}` — save credential
- `DELETE /api/v1/bots/{name}/mcp/credentials/{server}` — delete credential

### MCP Server Name
- `github` — the server name used in API calls for GitHub MCP
- This is the value passed to `/mcp/credentials/github`

## State Model

### Create Flow
```
User toggles GitHub MCP ON
  → enters PAT
  → deploy starts
  → deploy succeeds
  → POST /api/v1/bots/{name}/mcp (configure GitHub MCP server)
  → PUT /api/v1/bots/{name}/mcp/credentials/github (save PAT)
  → if both succeed: full success
  → if deploy OK but MCP fails: partial success
     - bot is deployed and running
     - user sees "Bot deployed. GitHub integration setup failed — configure in Settings."
     - no retry in create flow; user goes to Settings
```

### Settings Flow
```
Settings page loads:
  → GET /api/v1/bots/{name}/mcp → shows enabled integrations
  → GET /api/v1/bots/{name}/mcp/credentials → shows credential status

User saves credential:
  → PUT /api/v1/bots/{name}/mcp/credentials/github
  → on success: status updates to "present"
  → on error: show backend error message

User deletes credential:
  → DELETE /api/v1/bots/{name}/mcp/credentials/github
  → on success: status updates to "missing"
```

## UI Sections Layout

### Create Flow (Step 2)
```
[Bot Name]  [Provider]
[Team]      [Workflow Role]
[Display Name]  [Telegram Token]

── Integrations ──────────────────
☐ GitHub MCP — enable GitHub tools (PR, branch, file operations)
  [GitHub PAT: ••••••••••]  (shown only when checked)

── Bot Requirements ──────────────
[textarea]
```

### Settings Page
```
── General ───────────────────────
Bot Name: thomas-sa
Provider: claude-sonnet-4-6
Team: personal
...

── Integrations ──────────────────
GitHub MCP: Enabled | Credential: ●● Present
  [Update Token]  [Remove]

── Lifecycle ─────────────────────
[Stop] [Start] [Restart] [Delete]

── Validation ────────────────────
[Validate] (only for workflow-bound bots)
```

## What This ADR Does NOT Cover

- Generic MCP server browser/registry (admin surface, not bot-facing)
- Skills UI (future milestone)
- Tools UI (future milestone)
- Global MCP configuration (admin surface)
- Knowledge scope management (admin surface)

## Extensibility Path

When a new integration arrives (e.g. Slack MCP):
1. Add `SlackMcpToggle` component for create flow
2. Add `SlackMcpSettings` component for settings
3. Both render inside existing `IntegrationsSection`
4. No IA rewrite needed — just new children

## References

- Milestone #2: [Frontend] Bot Management Native Migration
- Issues: #14, #15, #16, #19
- Backend MCP APIs: `bot-config-api/src/routers/mcp.py`
