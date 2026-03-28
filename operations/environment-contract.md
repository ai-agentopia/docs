---
title: "Environment Contract"
---

# Environment Contract

## Environments

| Environment | Namespace | Domain | Infra branch | Image prefix |
|-------------|-----------|--------|-------------|-------------|
| DEV | `agentopia-dev` | `dev.agentopia.vn` | `dev` | `dev-` |
| UAT | `agentopia-uat` | `uat.agentopia.vn` | `uat` | `uat-` |

## Cluster

- Provider: k3s on server36
- StorageClass: `local-path`
- Ingress: Traefik
- GitOps: ArgoCD (auto-sync, self-heal, prune)

## ArgoCD App Structure

Each environment has:
- `root.yaml` — App-of-Apps bootstrap
- `base.yaml` — Base infrastructure (postgres, redis, neo4j, vault, qdrant, bot-config-api, mem0-api, llm-proxy)
- `ui.yaml` — Frontend
- `bots.yaml` — ApplicationSet for per-bot deployments

All apps target their environment's infra branch (`dev` or `uat`).

## Bot Deploy Contract

New bots created via bot-config-api:
- Initial image tags seeded from `BOT_IMAGE_TAG` env var (environment-correct)
- Image Updater manages subsequent updates via `helm.parameters`
- `valuesObject` contains config, `helm.parameters` contains image tags (no dual-source)

## Secret Management

- API keys in Vault (`secret/agentopia/gateway`)
- DB passwords pinned in Helm values (no auto-generation drift)
- Bootstrap: `scripts/bootstrap-secrets.sh {namespace}`

## Isolation

- Bot list scoped by `agentopia/env` label
- No cross-env contamination
- Each env has independent Vault, Postgres, Redis instances
