---
title: "Reviewer Remediation: Suggestions, Fix Commands, and Verified Closure"
---

# Reviewer Remediation: Suggestions, Fix Commands, and Verified Closure

**Milestone #31, Issue #287**

## Overview

The governed PR review workflow extends beyond finding-only review into **remediation**: the reviewer bot can suggest concrete fixes and, for well-understood issue classes, apply fixes directly to the PR branch via governed commands.

This document covers the full architecture, command reference, guardrails, operator guide, and implementation status.

---

## 1. Architecture — Where Fix Commands Fit

```
PR opened
  → SCM intake → review run created → reviewer bot dispatched
  → bot reads files + policy → produces findings WITH remediation suggestions
  → bot posts GitHub review

Developer reads review
  → posts /agentopia fix command as PR comment
  → webhook intake → parse command → validate finding + safety
  → dispatch fixer bot → bot reads file → applies fix → pushes commit
  → new head SHA triggers re-review
  → bot verifies fix → finding status updated → audit trail complete
```

### Key Concepts

| Term | Definition |
|------|-----------|
| **Finding** | A specific issue identified in a PR |
| **Suggested Remediation** | A concrete fix suggestion (code snippet, explanation) attached to a finding |
| **Safe-Fix Class** | A well-understood, mechanically safe issue category eligible for auto-fix |
| **Fix Command** | A `/agentopia fix ...` PR comment that triggers automated remediation |
| **Fix Execution** | The fixer bot reads the file, applies the transformation, pushes a commit |
| **Verified Closure** | Re-review confirms the fix addresses the finding; status → `resolved` |

### Finding Lifecycle

```
open → fix_suggested → fix_applied → resolved
                                   → unresolved (fix insufficient)
```

---

## 2. Command Reference

Commands are issued as **GitHub PR comments**. The webhook receives `issue_comment.created` events and routes recognized commands.

### Supported Commands

#### `/agentopia fix finding <finding_id>`
Fix a single finding by ID. Only works for findings with a `safe` fix class.

**Example:**
```
/agentopia fix finding 3e0d13a9-abc1-4def-...
```

**Behavior:**
- Resolves the finding from the latest review run
- Validates the finding's category has a `safe` fix class
- Dispatches the fixer bot to apply the fix
- Bot pushes a commit: `fix(review): <description> [reviewer-bot]`

#### `/agentopia fix safe findings`
Fix ALL open findings that have a `safe` fix class in the latest review run.

**Example:**
```
/agentopia fix safe findings
```

**Behavior:**
- Collects all open findings with safe-class categories
- Dispatches the fixer bot with all eligible findings
- Each finding gets one atomic commit

#### `/agentopia propose patch <finding_id>`
Show the proposed fix without auto-applying. The bot posts a PR comment with the before/after code.

**Example:**
```
/agentopia propose patch 3e0d13a9-abc1-4def-...
```

**Behavior:**
- Works for ANY finding (not restricted to safe classes)
- Bot reads the file and generates the fix
- Bot posts the patch as a PR comment
- Does NOT push any commits

### Rejection Cases

| Condition | Response |
|-----------|----------|
| Command on non-onboarded repo | Rejected: "Repository has no bound reviewer bot" |
| No review run for this PR | Rejected: "No review run found for PR #N" |
| Finding ID not found | Rejected: "Finding 'X' not found in latest run" |
| Finding not safe-class eligible | Rejected: "Finding 'X' is not eligible for auto-fix" |
| PR targets protected branch | Rejected: "Cannot apply fixes to protected branch 'main'" |
| Malformed command | Rejected with supported command list |

---

## 3. Safe-Fix Guardrails

### Only Safe Classes Are Auto-Applied

| Class ID | Safety | Auto-fix? | Description |
|----------|--------|-----------|-------------|
| `hardcoded-secret` | **safe** | Yes | Replace string literal with `os.environ[]` |
| `timing-unsafe-comparison` | **safe** | Yes | Replace `==` with `hmac.compare_digest()` |
| `sql-injection` | needs_review | No | Placeholder syntax varies by DB driver |
| `stack-trace-leak` | needs_review | No | Requires judgment about what to log vs return |

### Why SQL Injection Is Not Auto-Fixable

Parameterized query placeholder syntax varies by driver:
- `?` for sqlite3
- `%s` for psycopg/MySQLdb
- `:name` for sqlalchemy
- `$1` for asyncpg

The f-string → parameterized transformation requires knowing which driver is in use. This is not a single-expression mechanical replacement. The reviewer **suggests** the fix, but auto-apply is deferred until tightly bounded driver-specific patterns are implemented.

### Protected Branch Safety

Fix commits are **NEVER** pushed to:
- `main`, `master`
- `release/*`, `production`, `prod`

The branch check runs before dispatch. If the PR's head branch matches a protected pattern, the command is rejected.

### Fix Commit Provenance

Every fix commit includes:
- Message format: `fix(review): <description> [reviewer-bot]`
- One atomic commit per finding
- Traceable back to: finding ID → review run → fix command → commit SHA

### No Auto-Merge

The bot **never** merges the PR. After fixes are applied, the developer reviews the fix commits and decides whether to merge.

---

## 4. Operator / Developer Guide

### Triggering Fix Commands

1. Open a PR that has been reviewed by the Agentopia reviewer bot
2. Find a finding you want to fix in the review
3. Post a PR comment with the fix command:
   ```
   /agentopia fix finding <finding_id>
   ```
4. The system validates the command and dispatches the fixer bot
5. The bot pushes a fix commit to the PR branch

### Inspecting Remediation History

**Finding detail with remediation status:**
```
GET /api/v1/review/runs/{run_id}
```

Each finding includes:
- `suggested_fix`: the remediation suggestion (description, diff snippet, fix class, confidence)
- `resolution_status`: `open` / `fix_suggested` / `fix_applied` / `resolved` / `unresolved`
- `fix_commit_sha`: the commit that attempted to fix this finding

**Safe-fix classes:**
```
GET /api/v1/review/safe-fix-classes
```

Returns all registered fix classes with safety ratings.

### What Evidence to Expect

After a fix command:
1. A fix commit appears on the PR branch with `[reviewer-bot]` in the message
2. The new head SHA triggers a re-review automatically
3. The re-review checks if the finding is resolved
4. Finding status updates to `resolved` or `unresolved`

### Resolution Status Meanings

| Status | Meaning |
|--------|---------|
| `open` | Finding reported, no fix action taken |
| `fix_suggested` | Fix command issued, dispatch in progress |
| `fix_applied` | Fix commit pushed to PR branch |
| `resolved` | Re-review confirmed fix addresses the finding |
| `unresolved` | Re-review found fix insufficient |

---

## 5. Implementation Status

### Included Now

- Command parser: `/agentopia fix finding`, `/agentopia fix safe findings`, `/agentopia propose patch`
- Webhook intake: `issue_comment.created` → command routing with PR branch resolution via GitHub API
- Validation: onboarding, finding lookup, safe-class check, protected branch guard
- Fix dispatch: A2A sidecar transport to reviewer/fixer bot
- Dispatch message: includes fix class, template, evidence, safety rules
- Finding status updates: `update_finding_status()` with resolution lifecycle
- API: fix-command endpoint (internal-only, requires `X-Internal-Token`), finding responses with remediation fields
- Safe-fix registry: 2 safe + 2 needs_review classes
- Documentation: this document

### Audit Trail — What Is Actually Persisted

| Field | Persisted? | How |
|-------|-----------|-----|
| `resolution_status` | **Yes** | Updated on fix command (`fix_suggested`) and after closure (`resolved`/`unresolved`) |
| `fix_commit_sha` | **Modeled** | Column exists; not yet written by execution path (requires bot to report SHA back) |
| `suggested_fix` (JSONB) | **Yes** | Written when reviewer produces suggestions |
| Command source / provenance | **Yes** | `review_finding_audit` table: action=`fix_command`, detail includes command text, correlation ID, repo, PR, branch |
| Closure decision | **Yes** | `review_finding_audit` table: action=`closure_resolved` or `closure_unresolved`, detail includes prior/new run IDs and reason |
| Task/dispatch provenance | **Partial** | A2A task ID + correlation ID logged, but not yet linked in audit table |

**Audit query endpoint:**
```
GET /api/v1/review/findings/{finding_id}/audit
```
Returns the full audit trail: fix commands, closure events, timestamps.

### Finding Closure — How It Works

1. Fix command updates finding to `fix_suggested`
2. Bot dispatched → pushes fix commit → PR head SHA changes
3. GitHub Action triggers new intake with `event_type=synchronize`
4. New review run links to prior run via `prior_run_id`
5. Orchestrator detects prior run had fix-targeted findings
6. `reconcile_findings()` compares: same file+category still present?
   - **Absent in re-review** → `resolved` + audit event
   - **Still present** → `unresolved` + audit event

### Deferred

- **Commit SHA capture**: Bot pushes fix commit but does not yet report SHA back to the finding record.
- **SQL injection auto-fix**: Deferred until driver-specific patterns are bounded.
- **UI for fix commands**: Commands are PR-comment-based. No UI trigger yet.
- **Fix confirmation feedback**: Bot does not yet post a confirmation comment after pushing the fix commit.
- **Async closure**: Currently reconciliation runs synchronously during re-review dispatch. Full async (after bot completes review) is future work.

---

## 6. Review Completion — Current Status

### Current Stable Path (as of 2026-03-29)

1. Reviewer bot posts GitHub PR review via `gov_create_pr_review`
2. A2A sidecar marks task as completed
3. Review run status remains DISPATCHED (no automatic completion callback)

### What Works

- Bot reliably posts GitHub reviews (APPROVE / REQUEST_CHANGES)
- Reviews are visible on GitHub PR
- Review runs are created and tracked in the database

### What Is Partial / Deferred

| Capability | Status | Reason |
|------------|--------|--------|
| Structured finding persistence | **Deferred** | Requires completion callback or atomic tool — both blocked by platform issues |
| Run status → COMPLETED | **Partial** | Watchdog can recover stale DISPATCHED runs, but no real-time completion signal |
| Finding-level audit trail | **Deferred** | Depends on findings being persisted first |

### Experimental Tools (not active)

Two experimental tools exist in the codebase but are **not part of the active feature path**:

- `gov_report_review_completion` — completion callback tool. Works when bot calls it, but A2A sidecar marks task complete before bot can make a second tool call.
- `gov_submit_structured_pr_review` — atomic review + persistence tool. Backend endpoint works, but the OpenClaw gateway SDK does not expose newly registered tools to the LLM function set. Bot cannot see this tool at runtime.

Both are deferred to **milestone #32** ([Platform] Governance Tool Contract, Exposure, and Runtime Introspection).

---

## 7. Platform Blocker — Tool Exposure (#32)

### Problem
The OpenClaw gateway SDK registers tools via `api.registerTool()`, but newly added tools are not reliably exposed to the LLM's function set. The bot explicitly reported: *"gov_submit_structured_pr_review is not available in my function set"* despite the gateway logging 29 registered tools.

### Impact
- Cannot introduce new governance tools to bots without platform-layer investigation
- Only tools that existed in the original tool set are visible to the LLM
- Multiple schema iterations (nested object, simplified string) made no difference

### Resolution Path
Tracked under **milestone #32**: tool contract, runtime exposure verification, introspection, and release gates for tool changes.

### Current Workaround
The stable feature path uses `gov_create_pr_review` — a tool that predates this issue and is reliably visible to the LLM. Finding persistence and `/agentopia fix` commands remain partial until the platform blocker is resolved.
