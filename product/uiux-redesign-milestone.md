---
title: "[Frontend] UI/UX Redesign Milestone"
---

# [Frontend] UI/UX Redesign Milestone

## Companion Docs

- Issue pack: `docs/frontend-uiux-issues.md`
- Layout spec: `docs/frontend-uiux-layout-spec.md`

## Objective

Raise the visual quality of the standalone bot-facing frontend so it feels intentional, premium, and operationally clear without expanding product scope.

This milestone tracks the UI/UX redesign pass for:
- Communication
- Workflow
- Shared visual system
- Motion and state feedback
- Transcript robustness and machine-output presentation
- Empty/loading/error states

This is a frontend quality milestone, not a product redesign milestone.

## Visual Direction

Working theme: `Mission Console`

Principles:
- Operator-grade, not generic chat app
- Strong information hierarchy
- Dark surfaces with clearer depth separation
- Bright but controlled accent system
- Purposeful motion, not decorative motion
- Mobile-safe and desktop-first

## Non-Goals

This milestone does NOT include:
- workflow/business logic redesign
- A2A protocol redesign
- session/auth architecture changes
- multi-conversation product expansion
- backend API redesign

Ad-hoc correctness fixes may land in parallel, but they are not part of this milestone's visual acceptance.

## Scope

Primary surfaces:
- `CommunicationPage`
- `WorkflowPage`
- `WorkflowDetail`
- sidebar / bot identity shell
- shared status / state / feedback components

Primary repo:
- `personal/agentopia-ui`

## Tracking Model

Use one milestone with issue-sized tickets.

Milestone name:
- `[Frontend] UI/UX Redesign`

Label set:
- `frontend`
- `uiux`
- `design-system`
- `communication`
- `workflow`
- `polish`

Issue naming format:
- `FE-UI-01 ...`
- `FE-UI-02 ...`
- etc.

## Wave Plan

### Wave A — Visual System Foundation

#### FE-UI-01 — Shared Theme Refresh
Goal:
- Refine color tokens, surface depth, border contrast, and accent usage.

Files likely touched:
- `src/index.css`

Acceptance:
- shell / sidebar / panels are visually separated at a glance
- accent color usage is deliberate and limited
- success/error/warning states feel consistent
- no readability regressions on dark background

#### FE-UI-02 — Typography and Density Pass
Goal:
- Improve text hierarchy, spacing rhythm, and component density.

Files likely touched:
- `src/index.css`
- shared page/component surfaces

Acceptance:
- headers, labels, timestamps, badges, and body copy have clear hierarchy
- dense areas remain readable on desktop and mobile

### Wave B — Communication Redesign

#### FE-UI-03 — Communication Header and Bot Identity
Goal:
- Make the Communication surface feel like a live operator console.

Files likely touched:
- `src/pages/CommunicationPage.tsx`
- `src/components/WorkspaceHeader.tsx`

Acceptance:
- bot identity is clearer
- connection/session state is visible and well-placed
- page top section feels premium, not placeholder-level

#### FE-UI-04 — Message Bubble and Transcript Redesign
Goal:
- Upgrade transcript presentation, bubble styling, spacing, avatar treatment, and timestamps.

Files likely touched:
- `src/components/communication/MessageBubble.tsx`
- `src/components/communication/MessageList.tsx`
- `src/components/communication/StreamingMessage.tsx`

Acceptance:
- assistant and user messages are distinguishable instantly
- transcript has depth and rhythm
- long responses remain comfortable to read
- empty/invalid history messages do not render visual artifacts

#### FE-UI-05 — Composer and Input UX
Goal:
- Redesign composer to feel more deliberate and production-grade.

Files likely touched:
- `src/components/communication/CommunicationComposer.tsx`

Acceptance:
- input focus state is stronger
- send/stop actions feel clear
- disabled/streaming states are obvious
- mobile interaction remains usable

#### FE-UI-06 — Communication State Feedback
Goal:
- Improve loading, reconnect, error, and streaming/thinking states.

Files likely touched:
- `src/pages/CommunicationPage.tsx`
- `src/components/ErrorBanner.tsx`
- communication components

Acceptance:
- reconnect and failure states feel intentional
- streaming state has meaningful feedback
- no generic spinner-only UX where richer feedback is needed

#### FE-UI-13 — Transcript Robustness and Machine Output Handling
Goal:
- Prevent raw JSON, tool payloads, and other machine-like intermediary output from degrading the conversation UX.

Files likely touched:
- `src/pages/CommunicationPage.tsx`
- `src/lib/gateway-events.ts`
- `src/components/communication/MessageList.tsx`
- `src/components/communication/MessageBubble.tsx`

Acceptance:
- raw machine payloads do not appear as normal assistant replies by accident
- intermediary structured output is either filtered, downgraded, or rendered intentionally
- transcript remains readable even when backend/tool output is irregular

### Wave C — Workflow Redesign

#### FE-UI-07 — Workflow List Card Refresh
Goal:
- Redesign workflow cards so objective, state, and progress are easier to scan.

Files likely touched:
- `src/components/workflow/WorkflowCard.tsx`
- `src/components/workflow/WorkflowList.tsx`

Acceptance:
- active workflow stands out clearly
- progress indicator feels modern and legible
- card layout has stronger hierarchy

#### FE-UI-08 — Workflow Detail Information Architecture
Goal:
- Improve workflow detail layout for objective, status, timeline, artifacts, and activity.

Files likely touched:
- `src/components/workflow/WorkflowDetail.tsx`
- `src/components/workflow/PhaseTimeline.tsx`
- `src/components/workflow/ArtifactPanel.tsx`
- `src/components/workflow/ActivityFeed.tsx`

Acceptance:
- key workflow status is visible without scanning the whole page
- timeline, artifacts, and activity read as distinct sections
- detail page feels deliberate, not stacked boxes

#### FE-UI-09 — Workflow Composer and Intent UX
Goal:
- Improve the workflow creation / intent-entry surface.

Files likely touched:
- `src/components/workflow/WorkflowComposer.tsx`
- `src/components/workflow/WorkflowIntentCard.tsx`
- `src/components/workflow/WorkflowPlaceholder.tsx`

Acceptance:
- entry path is clearer
- empty state is more persuasive and less dead-looking
- create-flow surface matches redesigned Communication quality

### Wave D — Shell, Motion, and Final Polish

#### FE-UI-10 — Sidebar and Shell Cohesion
Goal:
- Make sidebar, tabs, and shell surfaces feel like one product.

Files likely touched:
- `src/layouts/AppShell.tsx`
- `src/components/sidebar/BotSidebar.tsx`
- `src/components/sidebar/BotListItem.tsx`

Acceptance:
- shell feels cohesive across Communication and Workflow
- selected bot and tab states are visually strong
- navigation is clearer on mobile and desktop

#### FE-UI-11 — Motion and Transition Pass
Goal:
- Add restrained motion for page load, list reveal, state transitions, and streaming feedback.

Acceptance:
- motion helps orientation
- motion does not feel generic or noisy
- no regressions in responsiveness

#### FE-UI-12 — Responsive and Final QA Pass
Goal:
- Close layout edge cases and visual regressions.

Acceptance:
- desktop and mobile both feel intentional
- no clipped transcript, broken buttons, or awkward empty states
- core pages are visually consistent

## Suggested Execution Order

1. Wave A
2. Wave B
3. Wave C
4. Wave D

Recommended implementation priority:
1. Communication first
2. Communication robustness second
3. Workflow third
4. Shell polish fourth

This gives the biggest visible product lift fastest.

## Issue Template

Use this template for each milestone issue.

### Title
- `[Frontend][UI/UX] <short task name>`

### Problem
- What feels weak or visually unclear today?

### Goal
- What user-facing improvement should be visible after this issue?

### In Scope
- exact components/files

### Out of Scope
- what should not expand in this task

### Acceptance Criteria
- concrete visual/interaction outcomes

### Validation
- screenshots before/after
- desktop + mobile check
- no regression in core flows

## Acceptance for the Milestone

The milestone is complete when:
- Communication feels premium and intentional
- Workflow surfaces match the same quality bar
- shell, sidebar, states, and motion feel cohesive
- desktop/mobile are both acceptable
- no obvious placeholder-level UI remains on primary bot surfaces

## Delivery Notes

- Keep changes incremental and reviewable.
- Prefer issue-sized PRs grouped by the tickets above.
- Do not mix major backend logic changes into this milestone.
- After ad-hoc correctness fixes are closed, continue on the main P1 / Wave F+G track with this milestone tracked separately.
