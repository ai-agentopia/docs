---
title: "UI Requirement: Knowledge Search Result → Create Workflow Brief"
status: requirement
---

# UI Requirement: Knowledge Search Result → Create Workflow Brief

## Context

The Agentopia operator UI has two separate surfaces for knowledge and workflow:

- **Knowledge UI** (`KnowledgePage.tsx`): operators can search ingested documents across client scopes. Search results show source, section, score, and text snippet. Currently **no action can be taken** on a search result — it is display-only.

- **Workflow UI** (`WorkflowPage.tsx`): operators can compose and start delivery workflows for bots. The workflow composer accepts a natural language objective, a target git repo (`owner/repo`), and an actor bot. Workflows are started via `POST /delivery/start`.

These two surfaces are disconnected. An operator who finds relevant knowledge must manually copy context into a workflow brief, losing grounding and wasting time.

## Problem

1. Operator searches Knowledge UI, finds a relevant document chunk about a requirement or architecture decision.
2. To act on it, operator must switch to Workflow UI, manually retype the context, and start a workflow.
3. This breaks the knowledge-to-action story: the knowledge is right there, but there's no path from "I found this" to "let's work on this."

## Desired UX

1. Operator searches in Knowledge UI.
2. Search results display as today (source, section, score, snippet).
3. Each result has a **"Create Workflow"** action button.
4. Clicking the button navigates to the Workflow UI with a **knowledge-grounded brief prefilled**.
5. Operator reviews/edits the prefilled objective and repo, then confirms to start the workflow.

## Required Behavior

### CTA on Search Results

- Each search result card displays a "Create Workflow" button.
- The button is only active when at least one workflow-capable bot exists.

### Workflow-Capable Bot Resolution

The system determines workflow-capable bots from the bot list where `can_start_workflow === true`.

| Scenario | Behavior |
|---|---|
| Zero workflow-capable bots | CTA disabled with helper text: "No workflow-capable bots available" |
| Exactly one workflow-capable bot | Click navigates directly to `/bots/:botId/workflow` with prefilled state |
| Multiple workflow-capable bots | Lightweight inline picker appears before navigation — operator selects which bot |

### Prefilled Brief Content

The workflow brief objective is prefilled from the selected knowledge result:

```
Based on knowledge from [source] (section: [section], scope: [scope]):

"[selected text snippet]"

Objective: [operator edits this]
```

The prefilled text is **editable** — the operator can modify before starting.

### Workflow Start Path

- Reuses existing `POST /delivery/start` endpoint.
- Request shape unchanged: `{ objective, target: { kind: "git_repo", owner, repo }, actor_id }`.
- The knowledge context is encoded into the `objective` field text, not a new API field.
- No backend API contract change required.

### Navigation

- After clicking "Create Workflow", operator lands on `/bots/:botId/workflow`.
- The `WorkflowPage` receives prefilled intent via navigation state (React Router state or URL params).
- The `WorkflowComposer` shows the prefilled objective.
- Operator can still edit objective and target repo before confirming.

## Acceptance Criteria

1. Search result cards in Knowledge UI show a "Create Workflow" action.
2. The action is disabled/hidden when no workflow-capable bots exist.
3. With exactly one workflow-capable bot, clicking navigates directly to workflow page.
4. With multiple workflow-capable bots, a lightweight bot picker appears first.
5. Workflow brief objective is prefilled with knowledge context (source, section, scope, snippet).
6. Operator can edit the prefilled objective before starting.
7. Workflow starts via existing `POST /delivery/start` — no backend change.
8. Existing free-text workflow composition path is not broken.

## Non-Goals

- No backend `/delivery/start` API redesign.
- No automatic repo detection from KB scope.
- No automatic workflow start without operator confirmation.
- No KB backend/schema change.
- No changes to knowledge-api or bot-config-api in this feature.

## Technical Notes

### Relevant Types

```typescript
// Search result (useKnowledge.ts)
interface SearchResult {
  text: string;
  score: number;
  scope: string;
  citation: {
    source: string;
    section: string;
    page: number | null;
    chunk_index: number;
    score: number;
    ingested_at: number;
    document_hash: string;
  };
}

// Workflow intent (WorkflowPage.tsx)
interface WorkflowIntent {
  objective: string;
  targetOwner?: string;
  targetRepo?: string;
}

// Bot workflow capability (useBots.ts)
interface Bot {
  name: string;
  can_start_workflow: boolean;
  can_view_workflows: boolean;
  // ...
}
```

### Expected Files Touched

- `agentopia-ui/src/pages/KnowledgePage.tsx` — add CTA + bot picker
- `agentopia-ui/src/pages/WorkflowPage.tsx` — accept prefilled intent from navigation state
- `agentopia-ui/src/__tests__/knowledge-workflow.test.tsx` — new test file

## Shipped Behavior (2026-04-03)

### Implementation Summary

Feature implemented in `agentopia-ui` branch `feat/ui-knowledge-workflow-trigger`:

- **KnowledgePage.tsx**: Added "Create Workflow" button on each search result card using the `Play` icon from lucide-react. The button is disabled with tooltip when no workflow-capable bots exist. For single workflow-capable bot: direct navigation. For multiple: lightweight modal picker with bot name/slug.
- **WorkflowPage.tsx**: Added `useEffect` that reads `location.state.prefilled.objective` from React Router navigation state. Prefilled intent is set as `currentIntent`, which renders the `WorkflowIntentCard` with editable objective and target repo fields. Navigation state is cleared after consumption via `window.history.replaceState`.
- **Tests**: 6 tests in `knowledge-workflow.test.tsx` covering objective text generation (source/section/scope/snippet inclusion, empty section handling, 300-char truncation) and bot resolution logic (single/zero/multiple workflow-capable bots).

### Actual Limitations

- Prefilled objective encodes knowledge context as text in the `objective` field — no structured metadata field was added to avoid backend API changes.
- Bot picker is a simple modal overlay, not a dropdown — sufficient for typical 1-3 bot deployments.
- The `objective` text includes the snippet truncated to 300 chars — longer chunks are cut.

### Final Acceptance Outcome

| Criterion | Result |
|---|---|
| CTA appears on search results | PASS — Play icon + "Create Workflow" on every result |
| Disabled when no workflow-capable bots | PASS — `disabled` attribute + tooltip |
| Single bot → direct navigation | PASS — navigates to `/bots/:botId/workflow` |
| Multiple bots → picker | PASS — modal with bot list |
| Prefilled objective with knowledge context | PASS — source, section, scope, snippet included |
| Operator can edit before starting | PASS — WorkflowIntentCard objective is editable |
| No backend API change | PASS — same `POST /delivery/start` payload |
| Existing workflow path not broken | PASS — knowledge-smoke tests still green |

## Evaluation Queries

These queries will be used for before/after KB evaluation:

1. "How does an operator create a workflow from a knowledge search result?"
2. "What happens when a user clicks Create Workflow on a search result?"
3. "How does the system handle multiple workflow-capable bots when starting from Knowledge UI?"
4. "Does the Create Workflow action change the backend delivery/start API?"
5. "What knowledge context is prefilled into the workflow brief?"
