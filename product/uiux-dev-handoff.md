---
title: "[Frontend] UI/UX Redesign — Dev Handoff"
---

# [Frontend] UI/UX Redesign — Dev Handoff

This is the direct implementation brief for dev.

Use with:
- [frontend-uiux-milestone.md](/Users/thtn/Documents/Working/Codebase/personal/agentopia-ui/docs/frontend-uiux-milestone.md)
- [frontend-uiux-issues.md](/Users/thtn/Documents/Working/Codebase/personal/agentopia-ui/docs/frontend-uiux-issues.md)
- [frontend-uiux-layout-spec.md](/Users/thtn/Documents/Working/Codebase/personal/agentopia-ui/docs/frontend-uiux-layout-spec.md)

## Decision

Workflow correctness is considered stable enough to proceed with the UI improvement track.

SA is occupied elsewhere.
Dev should work directly from this brief.

Do not wait for a broader design phase before starting implementation.

## Implementation Strategy

Do this in two waves.

### Wave 1 — Communication and Visual Foundation

Implement first:
- `FE-UI-01` Shared Theme Refresh
- `FE-UI-02` Typography and Density Pass
- `FE-UI-03` Communication Header and Bot Identity
- `FE-UI-04` Message Bubble and Transcript Redesign
- `FE-UI-05` Composer and Input UX
- `FE-UI-06` Communication State Feedback
- `FE-UI-13` Transcript Robustness and Machine Output Handling

This wave has the highest visible product impact and should land first.

#### Wave 1 Gates

##### Gate 0 — Baseline Capture
Before changing UI:
- capture current desktop screenshots for:
  - Communication idle
  - Communication active transcript
  - Communication composer
- capture at least one mobile screenshot or responsive viewport snapshot
- note any existing weird transcript behavior still visible

Pass only if:
- before-state screenshots are attached
- current UX issues are documented briefly

##### Gate 1 — Theme and Shell Foundation
Implement:
- `FE-UI-01`
- `FE-UI-02`

Pass only if:
- shell, sidebar, header, and surfaces show stronger hierarchy
- typography hierarchy is visibly improved
- no readability regression on dark surfaces

##### Gate 2 — Communication Surface Redesign
Implement:
- `FE-UI-03`
- `FE-UI-04`
- `FE-UI-05`

Pass only if:
- Communication header feels materially redesigned
- transcript bubbles and spacing are upgraded
- composer feels docked, deliberate, and clearer to use
- desktop and mobile layouts are both usable

##### Gate 3 — State Feedback Quality
Implement:
- `FE-UI-06`

Pass only if:
- loading, reconnect, and error states look intentional
- streaming/thinking state is visually meaningful
- stop/send state transitions are clear

##### Gate 4 — Transcript Robustness
Implement:
- `FE-UI-13`

Pass only if:
- raw JSON / machine-like intermediary payloads do not appear as ordinary assistant replies by accident
- empty or malformed history entries do not produce ugly transcript artifacts
- normal assistant text still renders correctly

##### Gate 5 — Final Wave 1 Smoke
Run the full Wave 1 smoke test checklist below.

Pass only if:
- all required smoke checks pass
- screenshots are attached
- no obvious regression is introduced in Communication core flow

### Wave 2 — Workflow and Shell Polish

After Wave 1 is reviewed:
- `FE-UI-07` Workflow List Card Refresh
- `FE-UI-08` Workflow Detail Information Architecture
- `FE-UI-09` Workflow Composer and Intent UX
- `FE-UI-10` Sidebar and Shell Cohesion
- `FE-UI-11` Motion and Transition Pass
- `FE-UI-12` Responsive and Final QA Pass

## Scope Lock

This is a frontend implementation wave.

Do NOT expand into:
- backend API changes
- A2A protocol changes
- workflow state-machine changes
- auth/session architecture changes
- multi-thread chat redesign

If a backend bug is discovered, stop and report it separately.
Do not silently fold backend work into this milestone.

## Wave 1 Design Direction

Theme:
- `Mission Console`

The Communication page should feel like:
- a live bot workspace
- a premium internal operator surface
- more deliberate, less generic

It should NOT feel like:
- a default chat template
- a flat admin page
- a generic dark dashboard

## Files Expected in Wave 1

Core likely files:
- `src/index.css`
- `src/layouts/AppShell.tsx`
- `src/components/WorkspaceHeader.tsx`
- `src/pages/CommunicationPage.tsx`
- `src/components/communication/MessageList.tsx`
- `src/components/communication/MessageBubble.tsx`
- `src/components/communication/StreamingMessage.tsx`
- `src/components/communication/CommunicationComposer.tsx`
- `src/components/ErrorBanner.tsx`

Possible shell alignment:
- `src/components/sidebar/BotSidebar.tsx`
- `src/components/sidebar/BotListItem.tsx`

## Required Outcomes for Wave 1

### 1. Theme and Surface Quality
- stronger shell/surface depth
- cleaner hierarchy between shell, sidebar, header, transcript, and composer
- more intentional use of accent color

### 2. Communication Header
- stronger bot identity presentation
- visible operational context
- cleaner tab/header treatment

### 3. Transcript Redesign
- user and assistant messages visually distinct at a glance
- transcript spacing and rhythm improved
- timestamps de-emphasized but readable
- streaming state looks intentional

### 4. Composer Redesign
- stronger focus state
- clearer send/stop actions
- better docked composition feel

### 5. State Feedback
- loading, reconnect, and error states improved
- no generic “just spinner and text” where richer feedback is needed

### 6. Transcript Robustness
- raw JSON / machine payload / intermediary tool-like output should not show as a normal assistant reply by accident
- empty or malformed history items should not create ugly transcript artifacts
- if machine output must appear, render it in a visibly secondary way

## FE-UI-13 Specific Guardrails

The screenshot bug class to avoid:
- assistant emits structured payload or raw JSON
- UI renders it as a normal conversational bubble
- transcript quality collapses

Minimum acceptable handling:
- detect obviously machine-oriented payloads
- filter or downgrade them before they become normal transcript content
- preserve normal assistant replies without regression

Do NOT overengineer this into a debug console in this wave.

## Implementation Boundaries

Keep PRs reviewable.

Recommended split:
1. PR A — theme + shell + header
2. PR B — transcript + composer
3. PR C — state feedback + transcript robustness

If dev can deliver cleanly in one PR, that is acceptable only if the diff remains reviewable.

## Validation Requirements

For each PR, return:
- files changed
- screenshots before/after
- desktop verification
- mobile verification
- any known gaps

For Wave 1 final review, provide:
- Communication idle screenshot
- Communication streaming screenshot
- Communication reconnect/error screenshot
- example showing malformed/raw machine output no longer degrades the transcript

## Wave 1 Smoke Test Checklist

Dev should run this after Wave 1 implementation.

### Smoke A — App Shell
Verify:
- sidebar renders correctly on desktop
- mobile drawer opens/closes correctly
- header tabs remain usable
- selected bot state is visually clear

### Smoke B — Communication Idle
Verify:
- Communication page loads without layout break
- history renders with improved spacing/styling
- empty state still works if history is empty

### Smoke C — Send and Stream
Verify:
- send a normal message
- streaming reply appears in the redesigned transcript correctly
- final assistant message is readable
- stop button still works during streaming

### Smoke D — Error and Reconnect
Verify:
- connection failure / reconnect state is visually coherent
- user can recover without layout corruption

### Smoke E — Transcript Robustness
Verify at least one of:
- a known raw/malformed/machine-oriented payload case from local/dev environment
- or a targeted fixture/test case that simulates it

Pass criteria:
- raw machine payload is filtered or downgraded intentionally
- it does not appear as a normal polished assistant reply
- transcript remains readable

### Smoke F — Responsive Check
Verify:
- desktop viewport
- laptop viewport
- mobile viewport

At minimum check:
- header
- transcript
- composer
- sidebar/drawer

## Recommended Test Commands

Do not install new packages in this wave unless separately approved.

Run the minimum practical set already available in the repo:
- targeted frontend tests for touched components
- existing routing/sidebar/auth tests if impacted

If additional rendering tests are added for transcript robustness, include them in the report.

If a full build is needed for confidence, call it out explicitly rather than assuming it is in scope.

## Required Return Package

For Wave 1 completion, dev must return:
- changed files
- screenshots before/after
- which wave gates passed
- smoke test results for Smoke A-F
- tests run
- known gaps or follow-up items for Wave 2

## Testing Expectations

Do not rely only on visual inspection.

At minimum:
- existing frontend tests must still pass if touched
- add targeted rendering tests where practical for transcript robustness
- verify no regression in send/stop flow

## Readiness Gate

Wave 1 is ready for review only when:
- Communication visually feels upgraded, not just recolored
- transcript readability is materially better
- machine-output weirdness is handled intentionally
- desktop and mobile both remain usable

## Direct Dev Prompt

Implement Wave 1 of the frontend UI/UX redesign in `agentopia-ui`.

Use these docs as source of truth:
- `docs/frontend-uiux-milestone.md`
- `docs/frontend-uiux-issues.md`
- `docs/frontend-uiux-layout-spec.md`
- `docs/frontend-uiux-dev-handoff.md`

Scope for this implementation:
- FE-UI-01
- FE-UI-02
- FE-UI-03
- FE-UI-04
- FE-UI-05
- FE-UI-06
- FE-UI-13

Priorities:
1. improve overall visual hierarchy and theme quality
2. redesign Communication header/transcript/composer
3. improve state feedback
4. harden transcript rendering against raw machine output and malformed history entries

Constraints:
- frontend only
- no backend changes unless separately approved
- keep route structure and product scope intact
- do not redesign multi-conversation behavior

Return:
- changed files
- screenshots before/after
- wave gate results
- smoke test results
- tests run
- any follow-up issues for Wave 2
