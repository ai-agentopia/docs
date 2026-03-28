---
title: "[Frontend] UI/UX Redesign — Issue Pack"
---

# [Frontend] UI/UX Redesign — Issue Pack

This document turns the milestone into issue-ready tickets.

Use with:
- Milestone: `[Frontend] UI/UX Redesign`
- Repo: `agentopia-ui`
- Labels: `frontend`, `uiux`, `design-system`, `communication`, `workflow`, `polish`

## FE-UI-01 — Shared Theme Refresh

### Title
`[Frontend][UI/UX] Refresh shared theme tokens and surface hierarchy`

### Problem
The current UI is functional but visually flat. Shell, sidebar, content panels, and interaction surfaces do not have enough depth separation.

### Goal
Create a stronger visual system so the product feels intentional and premium before page-level redesign begins.

### In Scope
- `src/index.css`
- shared color, border, surface, and accent variables

### Out of Scope
- page-level layout redesign
- new backend behavior

### Acceptance Criteria
- shell, sidebar, surface, elevated panel, and hover surfaces are clearly distinct
- accent, success, warning, and error colors are consistent
- contrast improves without harming readability
- no component becomes visually washed out on dark background

### Validation
- screenshots of shell, sidebar, communication, and workflow surfaces
- desktop + mobile visual sanity check

## FE-UI-02 — Typography and Density Pass

### Title
`[Frontend][UI/UX] Improve typography hierarchy and spacing rhythm`

### Problem
Headers, labels, timestamps, badges, and body text are readable but not well tiered.

### Goal
Improve information hierarchy and density so heavy bot/workflow views stay readable.

### In Scope
- `src/index.css`
- shared text scales and spacing patterns

### Out of Scope
- content/model changes
- new navigation structure

### Acceptance Criteria
- headers and subheaders feel distinct
- metadata text does not compete with primary content
- spacing rhythm is consistent across cards, lists, and details

### Validation
- before/after screenshots for Communication and Workflow

## FE-UI-03 — Communication Header and Bot Identity

### Title
`[Frontend][UI/UX] Redesign Communication header as operator console`

### Problem
The current Communication header is too light and does not establish bot identity, connection state, or context strongly enough.

### Goal
Make Communication feel like a live mission console for a selected bot.

### In Scope
- `src/pages/CommunicationPage.tsx`
- `src/components/WorkspaceHeader.tsx`

### Out of Scope
- transcript redesign
- composer redesign

### Acceptance Criteria
- bot identity is visually strong
- connection/session state has a clear position
- header feels premium and not placeholder-level
- layout works on desktop and mobile

### Validation
- desktop and mobile screenshots

## FE-UI-04 — Message Bubble and Transcript Redesign

### Title
`[Frontend][UI/UX] Redesign transcript bubbles, spacing, and streaming presentation`

### Problem
Current transcript presentation is correct but visually generic.

### Goal
Upgrade transcript readability and visual quality without changing the core chat model.

### In Scope
- `src/components/communication/MessageBubble.tsx`
- `src/components/communication/MessageList.tsx`
- `src/components/communication/StreamingMessage.tsx`

### Out of Scope
- WS protocol changes
- multi-conversation redesign

### Acceptance Criteria
- assistant and user messages are instantly distinguishable
- transcript has stronger rhythm and depth
- long assistant replies remain readable
- empty or invalid history messages do not render visual artifacts

### Validation
- short chat screenshot
- long reply screenshot
- streaming screenshot

## FE-UI-05 — Composer and Input UX

### Title
`[Frontend][UI/UX] Redesign Communication composer and action states`

### Problem
The current composer works but looks basic and does not communicate send/stop states strongly enough.

### Goal
Make the composer feel production-grade and more deliberate.

### In Scope
- `src/components/communication/CommunicationComposer.tsx`

### Out of Scope
- attachments feature expansion
- command parsing changes

### Acceptance Criteria
- input area has strong focus state
- send and stop actions are obvious
- disabled and streaming states are visually clear
- mobile interaction remains comfortable

### Validation
- keyboard/input screenshots
- send/stop state screenshots

## FE-UI-06 — Communication State Feedback

### Title
`[Frontend][UI/UX] Improve loading, reconnect, error, and thinking feedback`

### Problem
Several communication states still rely on generic loading/error presentation.

### Goal
Make state feedback intentional and informative.

### In Scope
- `src/pages/CommunicationPage.tsx`
- `src/components/ErrorBanner.tsx`
- communication state components

### Out of Scope
- backend reconnection policy changes

### Acceptance Criteria
- loading state feels designed
- reconnect/error state feels actionable
- streaming/thinking state gives confidence without noise

### Validation
- screenshots of loading, reconnect, and error states

## FE-UI-13 — Transcript Robustness and Machine Output Handling

### Title
`[Frontend][UI/UX] Harden transcript rendering against raw machine output and intermediary payloads`

### Problem
Conversation UI can still degrade when structured backend/tool output leaks directly into the transcript, such as raw JSON blocks or intermediary status payloads that were not meant to be the primary assistant reply.

### Goal
Make transcript rendering resilient so the UI remains readable even when backend output is irregular or partially machine-oriented.

### In Scope
- `src/pages/CommunicationPage.tsx`
- `src/lib/gateway-events.ts`
- `src/components/communication/MessageList.tsx`
- `src/components/communication/MessageBubble.tsx`

### Out of Scope
- backend protocol redesign
- full debug-console feature expansion

### Acceptance Criteria
- raw JSON/tool payloads do not appear as ordinary assistant chat bubbles by accident
- intermediary machine output is filtered, downgraded, or rendered intentionally
- conversation transcript remains readable under imperfect backend output
- normal assistant text still renders without regression

### Validation
- screenshot of normal reply flow
- screenshot or test case covering raw machine payload / structured output
- no empty or malformed visual artifacts in transcript

## FE-UI-07 — Workflow List Card Refresh

### Title
`[Frontend][UI/UX] Redesign workflow list cards and progress scanning`

### Problem
Workflow cards communicate information but do not scan strongly.

### Goal
Make active workflows easier to identify and compare quickly.

### In Scope
- `src/components/workflow/WorkflowCard.tsx`
- `src/components/workflow/WorkflowList.tsx`

### Out of Scope
- workflow business logic changes

### Acceptance Criteria
- objective, status, and progress have stronger hierarchy
- active items stand out clearly
- card layout is modern and legible

### Validation
- list screenshots with mixed states

## FE-UI-08 — Workflow Detail Information Architecture

### Title
`[Frontend][UI/UX] Redesign workflow detail layout and section hierarchy`

### Problem
Workflow detail is currently readable but too much like stacked utility panels.

### Goal
Make the detail view feel deliberate and high-signal.

### In Scope
- `src/components/workflow/WorkflowDetail.tsx`
- `src/components/workflow/PhaseTimeline.tsx`
- `src/components/workflow/ArtifactPanel.tsx`
- `src/components/workflow/ActivityFeed.tsx`

### Out of Scope
- workflow state-machine redesign

### Acceptance Criteria
- objective/status visible immediately
- timeline, artifacts, and activity feel like distinct sections
- layout works on desktop and mobile

### Validation
- before/after detail screenshots

## FE-UI-09 — Workflow Composer and Intent UX

### Title
`[Frontend][UI/UX] Improve workflow composer, intent card, and empty states`

### Problem
The workflow entry path is functional but not persuasive or polished.

### Goal
Make workflow creation feel intentional and easier to trust.

### In Scope
- `src/components/workflow/WorkflowComposer.tsx`
- `src/components/workflow/WorkflowIntentCard.tsx`
- `src/components/workflow/WorkflowPlaceholder.tsx`

### Out of Scope
- workflow parsing logic changes

### Acceptance Criteria
- entry path is clearer
- intent confirmation looks stronger
- empty state feels designed rather than dead

### Validation
- screenshots of empty, draft, and intent-confirmation states

## FE-UI-10 — Sidebar and Shell Cohesion

### Title
`[Frontend][UI/UX] Redesign sidebar and shell for stronger product cohesion`

### Problem
Sidebar and shell work but feel visually detached from Communication and Workflow surfaces.

### Goal
Make the app shell feel like one product.

### In Scope
- `src/layouts/AppShell.tsx`
- `src/components/sidebar/BotSidebar.tsx`
- `src/components/sidebar/BotListItem.tsx`
- `src/components/WorkspaceHeader.tsx`

### Out of Scope
- route structure changes

### Acceptance Criteria
- selected bot state is stronger
- tabs feel integrated into the shell
- mobile drawer feels deliberate, not temporary

### Validation
- full-shell screenshots desktop and mobile

## FE-UI-11 — Motion and Transition Pass

### Title
`[Frontend][UI/UX] Add restrained motion for orientation and polish`

### Problem
Current UI has little motion guidance and risks feeling static.

### Goal
Use motion to improve orientation and quality perception.

### In Scope
- transcript reveal
- list reveal
- page transitions
- state transitions

### Out of Scope
- animation-heavy redesign

### Acceptance Criteria
- motion helps orientation
- motion remains restrained
- no jank on desktop/mobile

### Validation
- short screen capture or manual QA notes

## FE-UI-12 — Responsive and Final QA Pass

### Title
`[Frontend][UI/UX] Close responsive edge cases and run final polish QA`

### Problem
Even after redesign work, final responsive and consistency issues can linger.

### Goal
Close the last visible rough edges before sign-off.

### In Scope
- desktop/mobile pass across Communication and Workflow
- final visual consistency review

### Out of Scope
- new features

### Acceptance Criteria
- no clipped or awkward primary interactions
- transcript, cards, buttons, and headers remain intentional on smaller screens
- primary pages feel visually consistent

### Validation
- final screenshot set
- manual responsive checklist

## Recommended Delivery Order

1. FE-UI-01
2. FE-UI-02
3. FE-UI-03
4. FE-UI-04
5. FE-UI-05
6. FE-UI-06
7. FE-UI-13
8. FE-UI-07
9. FE-UI-08
10. FE-UI-09
11. FE-UI-10
12. FE-UI-11
13. FE-UI-12

## Notes

- Run ad-hoc correctness fixes separately from these tickets.
- Communication redesign should land before Workflow polish.
- FE-UI-13 is the robustness lane for weird transcript cases that still slip through during implementation.
- This milestone should stay frontend-scoped and not expand into backend/product redesign.
