---
title: "Documentation Governance"
---

# Documentation Governance

## Where docs live

| Type | Location | Examples |
|------|----------|---------|
| Architecture, design, cross-repo | `ai-agentopia/docs` | System overview, A2A design, multi-bot architecture |
| ADRs | `ai-agentopia/docs/adrs/` | Integration abstraction, relay deprecation |
| Milestone trace | `ai-agentopia/docs/milestones/` | M6.2 delivery, rollout status |
| Runbooks | `ai-agentopia/docs/runbooks/` | Secret bootstrap, A2A ops |
| Env/CI/CD contracts | `ai-agentopia/docs/operations/` | CI/CD contract, environment contract |
| Product behavior | `ai-agentopia/docs/product/` | API reference, workflow docs |
| Service README, local setup | Code repo | `agentopia-protocol/README.md` |
| Component implementation notes | Code repo | Tightly coupled to code |
| Generated API docs | Code repo | If auto-generated from source |

## Rules

1. **No duplication.** A canonical doc exists in exactly one place.
2. **Code repos link, not copy.** If a doc is canonical, code repos add a pointer: `See [Canonical Doc](https://github.com/ai-agentopia/docs/blob/main/docs/...)`.
3. **Milestone docs are always canonical.** Created in `ai-agentopia/docs/milestones/`, not in code branches.
4. **Stale docs are removed or marked.** If a doc is outdated, either update it, mark it `[DEPRECATED]`, or delete with a commit message explaining why.

## Workflow integration

### PR checklist
Every PR should consider:
- Does this change require a docs repo update?
- Is there a new ADR needed?
- Does an existing canonical doc need updating?

### Milestone close
Before closing a milestone:
- Milestone trace doc created/updated in `ai-agentopia/docs/milestones/`
- Relevant canonical docs updated
- Stale docs cleaned up

### Release close
Before releasing:
- Operations docs current (CI/CD, environment contracts)
- Runbooks current
- Product docs reflect released behavior
