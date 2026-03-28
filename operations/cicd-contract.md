---
title: "CI/CD Contract"
---

# CI/CD Contract

## Branch Model

| Branch | Purpose | Builds images | Tag format |
|--------|---------|--------------|-----------|
| `dev` | Active development | Yes | `dev-{7-char SHA}` |
| `uat` | Pre-production validation | Yes | `uat-{7-char SHA}` |
| `main` | No-op (workflow_dispatch only) | No | — |

## Image Build Pipeline

### agentopia-protocol (`build-images.yml`)
Builds 5 images on push to `dev` or `uat` (path-filtered per service):
- `agentopia-gateway` (depends on `agentopia-runtime`)
- `bot-config-api`
- `mem0-api`
- `agentopia-llm-proxy`
- `a2a-sidecar`

Runtime resolution: CI queries GHCR API for latest `{env}-{sha}` tagged `agentopia-runtime` image.

### agentopia-ui (`build-image.yml`)
Builds `agentopia-ui` on push to `dev` or `uat` (no path filter).

### agentopia-core (`build-runtime.yml`)
Builds `agentopia-runtime` on push to `dev` or `uat` (path-filtered: `src/`, `ui/`, `package.json`, `Dockerfile.runtime`).

## Image Updater Policy

| Environment | Strategy | Allow-tags | Write-back |
|-------------|----------|-----------|------------|
| DEV | `newest-build` | `regexp:^dev-[a-f0-9]{7,9}$` | ArgoCD |
| UAT | `newest-build` | `regexp:^uat-[a-f0-9]{7,9}$` | ArgoCD |

All images (base services, UI, bot gateway, sidecar) follow the same policy per environment.

## Promotion Model

```
dev branch → CI builds dev-{sha} → Image Updater → DEV cluster
    ↓ (merge dev→uat)
uat branch → CI builds uat-{sha} → Image Updater → UAT cluster
```

No manual image patching. Promotion = branch merge + CI + Image Updater auto-update.

## Registry

All images: `ghcr.io/ai-agentopia/`

No ECR. No DockerHub. No `thanhth2813/` org references.
