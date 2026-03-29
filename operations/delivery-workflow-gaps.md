---
title: "Delivery Workflow — Known Gaps"
---

# Delivery Workflow — Known Gaps

> Internal tracking document for known delivery workflow gaps.
> Not public — excluded via .mintignore.
> Last updated: 2026-03-29

---

## GAP-1: Merge Phase Requires Webhook or Manual Signal

### Status: OPEN — workaround available

### Problem

When the delivery workflow reaches `wait_merge` phase (QA approved, PR ready), it blocks on Temporal signal `on_pr_merged`. This signal only arrives via:

1. **`gov_merge_pull_request` governance tool** — bot calls merge → signal auto-sent (governance.py:1050-1055)
2. **GitHub webhook** — `POST /api/v1/webhooks/github` receives `pull_request.closed+merged` event (webhook_api.py)
3. **`/wf accept` operator command** — SA bot or CTO sends `/wf accept wf-xxx` → sets `pr_merged=True` directly (delivery_workflow.py:751-760)

If the operator merges on GitHub UI without webhook configured, no signal is sent and the workflow sticks at `wait_merge` indefinitely.

### Current Workaround

- Setup GitHub webhook per repo pointing to `https://dev.agentopia.vn/api/v1/webhooks/github`
- Or use `/wf accept <workflow_id>` command via SA bot after manual merge

### Repos with Webhook

| Repo | Webhook | Status |
|---|---|---|
| `ai-agentopia/agentopia-demo-app` | Needs setup | Pending |
| `thanhth2813/a2a-flow-testing` | Needs setup | Pending |

### Future Options (Not Yet Decided)

**Option A — Auto-merge after QA approved**: Workflow calls `merge_and_close` activity automatically. Zero manual steps. `merge_and_close` activity already exists (activities.py:771-809) but is not wired into the workflow. Concern: removes human gate before merge.

**Option B — Webhook (current plan)**: Per-repo webhook setup. Works but requires manual setup per repo. Could be automated during bot deploy or repo onboarding.

**Option C — Hybrid**: Auto-merge default with `auto_merge: false` config override per-workflow or per-repo. Best of both but most effort.

### Decision

Deferred. Current MVP uses webhook + `/wf accept` fallback. Revisit when delivery workflow is used in production-critical repos.

---

## GAP-2: ArgoCD Image Updater Skipping agentopia-base

### Status: FIXED (2026-03-29)

### Problem

ArgoCD Image Updater was skipping `agentopia-base` Application with warning:
```
skipping app 'argocd/agentopia-base' of type '' because it's not of supported source type
```

### Root Cause

`agentopia-base` Application had `targetRevision: dev` pointing to a deleted branch on infra repo. After dev→main merge, the `dev` branch no longer existed → ArgoCD couldn't resolve → `status.sourceType` was empty → Image Updater skipped.

### Fix

Patched `targetRevision` from `dev` to `main`:
```bash
kubectl patch application agentopia-base -n argocd --type merge \
  -p '{"spec":{"source":{"targetRevision":"main"}}}'
```

After fix: `sourceType: Helm`, `sync: Synced`, Image Updater processing normally.
