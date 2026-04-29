---
title: Entry-Path Inventory (H1a)
description: Canonical inventory of every production ingress path mapped to typed intent and execution lane. Required baseline for H1b deterministic chat ingress.
---

# Entry-Path Inventory — H1a

> **Status**: Shadow mode (WP-H1a-03)
> **Canonical source**: [`agentopia-protocol/docs/architecture/entry-path-inventory.md`](https://github.com/ai-agentopia/agentopia-protocol/blob/dev/docs/architecture/entry-path-inventory.md)

This page is the docs-site reference for the H1a entry-path inventory. The
authoritative file lives next to the classifier code in `agentopia-protocol`
so the inventory and the deterministic matching logic stay aligned without
content duplication. Per `WP-H1a-03` non-goals, this site does not maintain
a parallel copy.

## What the inventory covers

- Every classified `POST` ingress path on `bot-config-api`, with current
  execution lane and target lane.
- Excluded admin/lifecycle prefixes (mutating endpoints intentionally not
  classified) so unresolved-rate metrics stay actionable.
- Unresolved-path handling rule and the `harness_intent_classified_total`
  metric contract.
- The five-file implementation map: `registry.py`, `classifier.py`,
  `shadow.py`, `middleware.py`, plus the registry/classifier tests.

## Why it is published

H1b (deterministic chat ingress) is gated on this inventory being
reviewable and binding. Routing decisions that move chat into Temporal
must trace back to a typed intent named here.

## How to update

Treat the canonical file as the working surface. When a new ingress is
added or an existing one changes lane:

1. Edit `agentopia-protocol/docs/architecture/entry-path-inventory.md` in
   the same PR that touches `registry.py` / `classifier.py`.
2. Bump the registry version (`registry.py` `REGISTRY_VERSION`) per the
   semver rule documented at the top of the registry module.
3. Re-run the shadow-classifier tests.

This site updates by reference only — no edits here unless the canonical
location moves.

## Related

- [Agent Harness Implementation Plan](/milestones/agent-harness-implementation-plan)
  — `WP-H1a-03` definition and acceptance criteria.
- [Harness Architecture](/architecture/harness-control/harness-architecture)
  — control-plane / deterministic-spine / autonomous-plane separation that
  the inventory's lane assignments key off.
