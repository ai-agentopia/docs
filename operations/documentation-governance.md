---
title: "Documentation Governance"
---

# Documentation Governance

## Where docs live

| Type | Location | Examples |
|------|----------|---------|
| Architecture, design, cross-repo | `agentopia-docs` | System overview, A2A design, multi-bot architecture |
| ADRs | `agentopia-docs/adrs/` | Integration abstraction, relay deprecation |
| Milestone trace | `agentopia-docs/milestones/` | M6.2 delivery, rollout status |
| Runbooks | `agentopia-docs/runbooks/` | Secret bootstrap, A2A ops |
| Env/CI/CD contracts | `agentopia-docs/operations/` | CI/CD contract, environment contract |
| Product behavior | `agentopia-docs/product/` | API reference, workflow docs |
| Service README, local setup | Code repo | `agentopia-protocol/README.md` |
| Component implementation notes | Code repo | Tightly coupled to code |
| Generated API docs | Code repo | If auto-generated from source |

## Rules

1. **No duplication.** A canonical doc exists in exactly one place.
2. **Code repos link, not copy.** If a doc is canonical, code repos add a pointer: `See [Canonical Doc](https://github.com/ai-agentopia/agentopia-docs/blob/main/docs/...)`.
3. **Milestone docs are always canonical.** Created in `agentopia-docs/milestones/`, not in code branches.
4. **Stale docs are removed or marked.** If a doc is outdated, either update it, mark it `[DEPRECATED]`, or delete with a commit message explaining why.

## Workflow integration

### PR checklist
Every PR should consider:
- Does this change require a docs repo update?
- Is there a new ADR needed?
- Does an existing canonical doc need updating?

### Milestone close
Before closing a milestone:
- Milestone trace doc created/updated in `agentopia-docs/milestones/`
- Relevant canonical docs updated
- Stale docs cleaned up

### Release close
Before releasing:
- Operations docs current (CI/CD, environment contracts)
- Runbooks current
- Product docs reflect released behavior
