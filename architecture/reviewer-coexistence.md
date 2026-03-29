---
title: "Reviewer Coexistence Model"
---

# Reviewer Coexistence Model

> How Agentopia separates workflow reviewers from repo-bound SCM reviewers.
> Both can coexist without dispatch ambiguity.
> Last updated: 2026-03-29

---

## 1. Two Review Concepts

Agentopia supports two distinct reviewer roles that serve different purposes:

| Concept | Purpose | Trigger | Resolution |
|---|---|---|---|
| **Workflow Reviewer** | QA reviewer in the delivery pipeline | Temporal DeliveryWorkflow reaches Review phase | RoleRegistry `reviewer` role (whitelist filter) |
| **Repo-Bound Reviewer** | Dedicated code reviewer for a specific GitHub repo | GitHub PR event via GitHub Actions | OnboardedRepo table (direct lookup) |

Both roles use the same governance tools (`gov_create_pr_review`, `gov_get_pr_files`, etc.) and share the `reviewer` role key for authorization. The separation is in the **dispatch path**, not in permissions.

---

## 2. Separation Model

### 2.1 Data Layer

Every reviewer binding in the RoleRegistry carries explicit `reviewer_mode` metadata:

```
kia-qa:              role=reviewer, metadata={"reviewer_mode": "workflow"}
code-reviewer:       role=reviewer, metadata={"reviewer_mode": "repo_bound"}
```

Repo-bound reviewers additionally have an `OnboardedRepo` record:

```
repo: ai-agentopia/agentopia-demo-app → reviewer: demo-apa-code-review, mode: repo_bound
```

### 2.2 Deploy Layer

When a reviewer bot is deployed:

- **Step 5 (Workflow Bind)**: Binds actor to `reviewer` role with explicit `reviewer_mode` metadata
  - Workflow reviewer → `{"reviewer_mode": "workflow"}`
  - Repo-bound reviewer → `{"reviewer_mode": "repo_bound"}`
- **Step 5b (Repo Onboarding)**: Only for repo-bound mode — creates `OnboardedRepo` record linking the bot to a specific GitHub repository

### 2.3 Workflow Dispatch (Delivery Pipeline)

When the delivery workflow reaches the Review phase:

1. `request_review()` activity calls `_resolve_dispatch_target("reviewer")`
2. Dispatch target resolution uses **whitelist filter**: only actors with `metadata.reviewer_mode == "workflow"` are included
3. Repo-bound reviewers are excluded from the dispatch pool
4. Single workflow reviewer → resolved cleanly, no ambiguity

### 2.4 SCM Dispatch (GitHub Action)

When a GitHub PR event arrives:

1. GitHub Action calls `POST /api/v1/review/intake`
2. `review_orchestrator` looks up `OnboardedRepo` table — **not** the RoleRegistry
3. Resolves the bound reviewer by repo coordinates
4. Dispatches only if `reviewer_mode == "repo_bound"`
5. Workflow-mode reviewers are NOT auto-dispatched from SCM intake

### 2.5 API Binding

The `POST /bind-actor` endpoint enforces reviewer mode:

```json
{
  "actor_id": "my-reviewer",
  "role_key": "reviewer",
  "reviewer_mode": "workflow"    // required for reviewer role
}
```

- `reviewer_mode` is **required** when `role_key == "reviewer"`
- Must be `"workflow"` or `"repo_bound"`
- Prevents legacy bindings without metadata from entering the dispatch pool

---

## 3. Coexistence Guarantee

When both reviewer types are active:

```
RoleRegistry:
  kia-qa           → reviewer (reviewer_mode: workflow)
  code-reviewer    → reviewer (reviewer_mode: repo_bound)

Workflow dispatch query:
  get_actors_for_role("reviewer") → [kia-qa, code-reviewer]
  whitelist filter (== "workflow") → [kia-qa]
  → RESOLVED: kia-qa

SCM dispatch query:
  get_onboarded_repo("org", "repo") → code-reviewer
  → RESOLVED: code-reviewer
```

No `AMBIGUOUS_BOUND_AGENT`. No cross-contamination.

---

## 4. Authorization

Both reviewer types share the same `reviewer` role key in the RoleRegistry. This means:
- Same governance tool access (review PRs, read code, post findings)
- Same `/wf approve` command access
- Metadata is invisible to the authorization layer — it only affects dispatch routing

---

## 5. Operational Notes

### Adding a Workflow Reviewer

Deploy a bot with `workflow_role_key: "reviewer"` and no `reviewer_binding` (or `reviewer_binding.mode: "workflow"`). The bot joins the workflow dispatch pool.

### Adding a Repo-Bound Reviewer

Deploy a bot with `workflow_role_key: "reviewer"` and `reviewer_binding: { mode: "repo_bound", repo_owner: "org", repo_name: "repo" }`. The bot is onboarded to the specified repo and excluded from workflow dispatch.

### Migration

Existing reviewer bindings without metadata must be re-bound with explicit `reviewer_mode`:
- Redeploy the bot, OR
- Call `POST /bind-actor` with `reviewer_mode` field, OR
- Manual DB update via `store.save_binding()`

Bindings without metadata are excluded from workflow dispatch by the whitelist filter — they do not cause errors, but they will not participate in workflow review.
