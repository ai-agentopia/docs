---
title: "Workflow Cancel with Cleanup"
description: "How operators cancel active workflows, what gets cleaned up, and what is preserved."
---

Operators can cancel an active workflow from the workflow detail view. Cancellation stops the active workflow, terminates live sub-resources, and transitions the workflow to a clear terminal state. Audit and history records are always preserved.

## The Problem

Without a cancel action, an operator who needs to stop an in-progress delivery has no product-safe path. The only options are waiting for natural completion, or manual backend intervention — neither is acceptable for an operator-facing product.

A canceled workflow must:

1. Stop all active workflow progression
2. Show a clear terminal canceled state in the UI
3. Clean up live, external workflow resources
4. Preserve all auditable workflow records in the database

## Behavior

### Initiating Cancel

The **Cancel Workflow** button appears in the workflow detail view for any workflow that is not already in a terminal state (`DONE` or `CANCELED`). Clicking it opens a confirmation dialog. Confirming sends the cancel request.

During cancellation, the button shows a loading state. The page reflects the outcome once the API responds.

### After Cancel

On success, the workflow status badge updates to **Canceled**. All active controls (including the cancel button itself) disappear. The workflow no longer polls for updates.

If cleanup is only partially successful — for example, if the Temporal runtime was temporarily unavailable — the workflow is still marked **Canceled** (Postgres is authoritative), and a warning note appears below the status badge.

## Cleanup Contract

### Live Resource Cleanup

These resources are cleaned up as part of every successful cancel:

| Resource | Action |
|----------|--------|
| Workflow run state | Transitions to `CANCELED` in the database — this is the authoritative first step |
| Orchestration monitor loop | Stopped immediately after DB transition (synchronous, idempotent) |
| Temporal delivery workflow | Sent a cancel signal (best-effort — see Temporal Divergence below) |
| Work packets in active states | Transitioned to `cancelled` (see Packet Policy below) |
| Objectives in `active` state | Transitioned to `cancelled` |
| Trello planning card | Archived (fire-and-forget — best-effort) |

### Work Packet Policy

| Packet Status | On Cancel | Reason |
|---------------|-----------|--------|
| `draft` | → `cancelled` | Not started, no work product |
| `pending` | → `cancelled` | No work product |
| `assigned` | → `cancelled` | Not yet dispatched |
| `in_progress` | → `cancelled` | Worker work abandoned by operator decision |
| `review` | → `cancelled` | Active review abandoned by operator decision |
| `completed` | Preserved | Meaningful success evidence |
| `failed` | Preserved | Meaningful failure evidence — erasing misrepresents history |
| `cancelled` | Preserved | Already terminal |

### Preserved Audit State

The following are **never deleted** by workflow cancel:

- Workflow run records (`w0_runs`)
- Delivery start requests
- Workflow messages and conversation history
- Evidence and artifact records
- Command audit rows

This is by design. Cancel is a product operation, not a data purge. If historical records need to be removed, that requires a separate admin-level operation with explicit scope and authorization.

## Partial Cleanup Behavior

Cancel is modeled as a two-tier operation:

- **Canonical step** (DB run-state transition): must succeed. If it fails, the workflow is not canceled and the API returns an error. No cleanup runs.
- **Subordinate cleanup steps** (monitor stop, Temporal, packets, objectives, Trello): run after the canonical step. Failures in these steps are captured as warnings in the result — they do not prevent the workflow from being marked `CANCELED`.

When the workflow is `CANCELED` in the database but some cleanup step failed, the API returns `cleanup_status: "partial"` and includes a `warnings` list describing what was not completed.

### Temporal Divergence

If the Temporal runtime cancel signal fails (e.g. Temporal is temporarily unavailable):

- The workflow is still `CANCELED` in the database (Postgres is authoritative)
- The API response includes `temporal_cancelled: false` and a warning
- A structured divergence log is emitted: `cancel_temporal_divergence: workflow_id={id} db_state=CANCELED error={msg}`
- The UI shows the CANCELED state with a partial cleanup warning

**Manual reconciliation**: An admin can terminate the Temporal workflow directly using the Temporal CLI or Temporal Web UI, targeting workflow ID `delivery-{workflow_id}`.

## Objective Status After Cancel

Active objectives are transitioned to `cancelled` (not `inactive`). The term "cancelled" is semantically precise for a workflow-level termination and mirrors the terminal vocabulary used by both the workflow state (`CANCELED`) and work packets (`cancelled`). Cancelled objectives are non-actionable and displayed as dimmed in any reporting surface.

## Non-Goals

This feature does not:

- Hard-delete `w0_runs`, `workflow_messages`, `delivery_start_requests`, or any audit/evidence rows
- Provide an admin purge mechanism — that is a separate admin follow-up action
- Guarantee Temporal cleanup if the Temporal runtime is unavailable

## Acceptance Criteria

- Cancel requires session authentication — 401 without a valid session cookie
- Cancel transitions the workflow run state to `CANCELED` in the database
- Orchestration monitor is stopped for the workflow
- Non-terminal packets (draft, pending, assigned, in_progress, review) are cancelled
- Completed and failed packets are preserved
- Active objectives are transitioned to `cancelled`
- Trello card is archived if present
- Already-terminal workflows return a 200 with `ALREADY_TERMINAL` status (idempotent)
- Temporal divergence returns 200 with `cleanup_status: "partial"` and warning
- All audit rows exist after cancel — no hard deletes
- UI cancel button is visible only on non-terminal workflows
- UI requires confirmation before cancel
- UI shows CANCELED badge after successful cancel
- UI shows a partial cleanup warning when `cleanup_status: "partial"`
