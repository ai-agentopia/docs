---
title: "Repo Split Notes"
---

# Repo Split Notes

## Context

The Agentopia frontend was originally embedded in `bot-config-api` as static files served by FastAPI at `/static` and `/ui`.

## Split Decision

A standalone `agentopia-ui` repository was created to decouple the frontend from the API backend.

Prerequisites before completing the split:
1. Backend API contracts stabilize
2. Communication and Workflow endpoints are in place
3. Image/deploy strategy is agreed

## Current State (completed 2026-03-28)

- `agentopia-ui` builds its own frontend image (nginx-based)
- UI deploys independently from `bot-config-api` via ArgoCD
- `bot-config-api` is now **API-only** — legacy `/ui` surface removed
- Root `/` returns JSON API info instead of redirecting to `/ui`
- All static assets (`index.html`, `app.js`, `style.css`) deleted from `bot-config-api`

## Repos

| Repo | Role |
|---|---|
| `agentopia-ui` | Standalone frontend (React + Vite + nginx) |
| `agentopia-protocol/bot-config-api` | API-only control plane (FastAPI) |
