---
title: "[Frontend] UI/UX Redesign — Layout Spec"
---

# [Frontend] UI/UX Redesign — Layout Spec

This is a Figma-ready layout brief for the `Mission Console` redesign.

If this is moved into Figma, use this document as the canonical handoff spec.

## Design Intent

The UI should feel like:
- a live operator console
- a premium internal tool
- a focused mission workspace for one selected bot

It should NOT feel like:
- a generic consumer chat app
- a flat admin CRUD surface
- a random dark dashboard

## Figma File Structure

Create one Figma file with 4 pages:

1. `00 Foundations`
2. `01 Communication`
3. `02 Workflow`
4. `03 Components & States`

## Frame Set

Use these base frames:
- Desktop: `1440 x 1024`
- Laptop: `1280 x 900`
- Mobile: `390 x 844`

Grid:
- Desktop main canvas: 12 columns, 24px margin, 24px gutter
- Mobile: 4 columns, 16px margin, 12px gutter

## Foundation Spec

### Color Tokens

Base direction:
- shell: near-black graphite
- sidebar: slightly warmer dark surface
- primary panel: deep charcoal
- elevated cards: cool graphite with higher separation
- accent: electric blue / cyan range
- warm accent: restrained amber/orange for warnings or emphasis
- success: vivid but not neon green
- error: controlled red, not oversaturated

### Surface Hierarchy

Use 5 layers max:
- shell background
- sidebar/header background
- primary content surface
- elevated card surface
- hover/interactive surface

### Typography

Fonts already present:
- sans: `Geist Sans`
- mono: `IBM Plex Mono`

Use hierarchy:
- page title / bot identity
- section title
- card title
- body text
- metadata
- micro labels

## Shell Layout

### Desktop Shell

Structure:
- left sidebar: `280px`
- top workspace header: `72px`
- main content area: remaining width
- content padding: `24px`

ASCII wireframe:

```text
+------------------+--------------------------------------------------+
| Sidebar          | Workspace Header                                 |
| brand            | bot name | role chip | status | tabs            |
| bot list         +--------------------------------------------------+
|                  |                                                  |
|                  | Main Content                                      |
|                  |                                                  |
|                  |                                                  |
+------------------+--------------------------------------------------+
```

### Sidebar Spec

Sections:
- brand block
- bot list
- utility footer

Bot row anatomy:
- avatar / initial
- bot name
- role/subtitle
- subtle status marker
- selected state with strong active surface

Sidebar should feel like a navigation rail, not a plain list.

## Communication Layout

### Desktop Communication Page

Structure:
- top hero panel
- transcript pane
- bottom composer dock

ASCII wireframe:

```text
+--------------------------------------------------------------+
| Hero Panel                                                   |
| bot identity | role | connection | session | quick status    |
| short descriptor / operational note                          |
+--------------------------------------------------------------+
|                                                              |
| Transcript Surface                                           |
|                                                              |
|  A  assistant bubble                                         |
|         assistant text...                                    |
|                                                              |
|                               user bubble                    |
|                               user text...                   |
|                                                              |
|  streaming row / thinking indicator                          |
|                                                              |
+--------------------------------------------------------------+
| Composer Dock                                                |
| [ multiline input............................... ] [send]    |
| helper row: enter to send / shift+enter newline / stop state |
+--------------------------------------------------------------+
```

### Hero Panel Spec

Content blocks:
- bot title: large, high-contrast
- role chip
- connection health chip
- session chip
- optional descriptor line

Visual style:
- layered surface with subtle gradient or glow edge
- not a flat rectangle
- slight border highlight

### Transcript Surface Spec

Rules:
- transcript width max around `880px` desktop
- generous vertical rhythm
- timestamp de-emphasized but readable
- user and assistant message styles clearly different

Assistant bubble:
- elevated dark surface
- subtle border/highlight
- avatar disk anchored left

User bubble:
- stronger accent-filled bubble
- slightly sharper/brighter feel than assistant bubble

Streaming state:
- same lane as assistant
- include subtle live indicator
- avoid generic bouncing dots unless styled intentionally

Machine-output guardrail:
- raw JSON or tool/status payloads must not be rendered as a normal assistant reply by default
- if surfaced at all, they should appear as a clearly secondary machine/status card
- primary conversational reply should stay visually dominant

### Composer Dock Spec

Behavior:
- sticky/docked bottom feel
- slightly separated from transcript
- strong focus ring when active

Controls:
- multiline input
- primary send action
- stop action during streaming
- helper text line optional on desktop only

### Mobile Communication Layout

Changes:
- hero panel becomes compact stacked card
- transcript keeps avatar + bubble pattern but tighter spacing
- composer remains docked and thumb-friendly
- header/tabs remain readable without crowding

## Workflow Layout

### Workflow List Page

Structure:
- top summary strip
- filter tabs
- stacked workflow cards
- composer dock at bottom when supported

ASCII wireframe:

```text
+--------------------------------------------------------------+
| Summary Strip                                                |
| Workflows | active count | recent state | quick hint         |
+--------------------------------------------------------------+
| Tabs: All | Active | Completed | Failed                     |
+--------------------------------------------------------------+
| Workflow Card                                                |
| objective                                                    |
| phase rail / progress                                        |
| status badge | owner/repo/meta | timestamp                   |
+--------------------------------------------------------------+
| Workflow Card                                                |
+--------------------------------------------------------------+
| Workflow Composer / Intent entry                             |
+--------------------------------------------------------------+
```

### Workflow Card Spec

Card anatomy:
- objective title
- state badge
- progress rail / phase indicator
- metadata row

Visual rules:
- active cards get stronger border/accent treatment
- completed cards feel resolved, not merely dimmed
- blocked/failed cards should read immediately as attention states

### Workflow Detail Page

Structure:
- back link + hero header
- summary row
- phase timeline section
- artifacts section
- activity section

ASCII wireframe:

```text
+--------------------------------------------------------------+
| Back                                                         |
| Workflow Objective                                           |
| status badge | created time | optional owner/repo meta       |
+--------------------------------------------------------------+
| Phase Timeline                                               |
+--------------------------------------------------------------+
| Artifacts                                                    |
+--------------------------------------------------------------+
| Activity                                                     |
+--------------------------------------------------------------+
```

### Workflow Detail Hero

Needs:
- stronger objective presentation
- status badge with good contrast
- metadata grouped cleanly

### Timeline Spec

Timeline should show:
- current active phase clearly
- completed phases as resolved
- blocked/failure variants intentionally

Do not visually depend on stale backend names.

## Motion Spec

Use motion sparingly:
- transcript entry reveal
- active status pulse or glow
- list card stagger on first load
- subtle composer focus transition

Avoid:
- heavy spring animations everywhere
- oversized shimmer effects
- decorative motion without meaning

## Component States to Mock in Figma

Communication:
- connected idle
- streaming
- error/reconnect
- empty state
- machine payload / raw JSON leak fallback

Workflow:
- active list
- completed list
- failed/blocked card
- detail active
- detail done
- empty workflow state

Shell:
- sidebar selected state
- mobile drawer open

## Implementation Boundaries

This layout spec should preserve:
- current route structure
- current product scope
- current one-bot communication model
- current workflow surfaces
- readable conversation UX even when backend output is imperfect

This layout spec should NOT assume:
- multi-thread chat redesign
- new backend APIs
- a separate right-side inspector unless explicitly approved later

## Handoff Checklist

A design pass is ready when it includes:
- desktop communication frame
- mobile communication frame
- workflow list frame
- workflow detail frame
- shell/sidebar frame
- component states
- token notes for colors, spacing, radius, and shadows

## Build Order

Recommended implementation order from this layout:
1. theme tokens
2. shell/header/sidebar
3. communication page
4. transcript + composer
5. workflow list
6. workflow detail
7. motion + responsive cleanup
