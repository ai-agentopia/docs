---
title: "Agentopia Fresh Deploy Guide"
---

# Agentopia Fresh Deploy Guide

Full deploy from zero (after cluster rebuild or namespace wipe).

**Total time**: ~7 minutes (~3 min manual, rest automated)

## Prerequisites

- `kubectl` configured and pointing to EKS cluster `nbo-dev-apps`
- ArgoCD installed with AppProject `infra` allowing namespace `agentopia`
- EFS file system `fs-02681554d212df736` exists (eu-central-1)
- EFS CSI driver add-on enabled on EKS
- GitHub PATs ready (see Secrets section)

## Step 1: Create namespace

```bash
kubectl create namespace agentopia
```

## Step 2: Create K8s Secrets

All secrets are created idempotently by `create-secrets.sh`.

### Required env vars

| Variable | Description | Where to get |
|----------|-------------|--------------|
| `VAULT_TOKEN` | Vault root token for gateway pods | Use `agentopia-vault-token` (convention) |
| `GHCR_PAT` | GitHub PAT with `read:packages` scope | github.com/settings/tokens |
| `GHCR_USER` | GitHub username | `ai-agentopia` |
| `NEO4J_PASSWORD` | Neo4j database password | Choose a strong password |
| `OPENROUTER_API_KEY` | OpenRouter API key | openrouter.ai |
| `MEM0_LLM_MODEL` | LLM model for mem0 | e.g. `openrouter/google/gemini-2.0-flash-001` |

### Optional env vars

| Variable | Description |
|----------|-------------|
| `GITHUB_TOKEN_DEPLOY` | GitHub PAT with `repo` scope (enables git push from UI) |
| `CODEX_AUTH_PROFILES_PATH` | Path to Codex OAuth auth-profiles.json |

### About GitHub PATs — two different tokens

The script uses **two separate GitHub PATs** for different purposes:

| Env var | Secret created | PAT scope needed | Purpose |
|---------|---------------|------------------|---------|
| `GHCR_PAT` | `ghcr-pull-secret` | `read:packages` | Pull Docker images from ghcr.io |
| `GITHUB_TOKEN_DEPLOY` | `bot-config-api-github` | `repo` (read+write) | Push bot configs to git repo |

You **can** use the same PAT for both if it has both scopes (`read:packages` + `repo`),
but best practice is to use separate tokens with minimal permissions.

### Run

```bash
export VAULT_TOKEN="agentopia-vault-token"
export GHCR_PAT="ghp_..."
export GHCR_USER="ai-agentopia"
export NEO4J_PASSWORD="..."
export OPENROUTER_API_KEY="sk-or-..."
export MEM0_LLM_MODEL="openrouter/google/gemini-2.0-flash-001"
export GITHUB_TOKEN_DEPLOY="ghp_..."   # optional but recommended

./scripts/create-secrets.sh
```

### Secrets created

| # | Secret | Used by |
|---|--------|---------|
| 1 | `agentopia-vault-token` | Gateway pods → Vault auth |
| 2 | `ghcr-pull-secret` | imagePullSecret for ghcr.io |
| 3 | `neo4j-auth` | Neo4j + mem0-api |
| 4 | `mem0-env` | mem0-api (OPENROUTER_API_KEY, MEM0_LLM_MODEL) |
| 5 | `agentopia-auth-codex` | (Optional) Codex OAuth |
| 6 | `bot-config-api-env` | bot-config-api (OpenRouter for LLM generation) |
| 7 | `bot-config-api-github` | bot-config-api (GitHub push for deploy automation) |

> **Important**: If `GITHUB_TOKEN_DEPLOY` is not set, secret #7 is skipped.
> Bot deploy from UI will show "GITHUB_TOKEN not set" and skip the git push step.
> Bots still work but configs won't be persisted in git (lost on cluster rebuild).

## Step 3: Apply ArgoCD Application (base infra)

```bash
kubectl apply -f argocd/agentopia-base.yaml
```

This auto-deploys all base infrastructure via ArgoCD:
- Namespace labels
- EFS StorageClass `efs-sc`
- Shared memory PVC `agentopia-shared-shared`
- Qdrant (StatefulSet + PVC)
- Neo4j (StatefulSet + PVC)
- Vault (StatefulSet + auto-init/unseal sidecar)
- mem0-api (Deployment)
- bot-config-api (Deployment + Ingress + RBAC)

### Wait for pods

```bash
kubectl get pods -n agentopia -w
```

All 5 pods should reach `Running` + `Ready` within ~3-5 minutes:
- `bot-config-api` (1/1)
- `mem0-api` (1/1) — may restart 2-3 times waiting for deps
- `neo4j-0` (1/1)
- `qdrant-0` (1/1)
- `vault-0` (2/2) — vault + auto-unsealer sidecar

## Step 4: Seed Vault with API keys

Once Vault pod is `2/2 Running`:

```bash
# Terminal 1: port-forward
kubectl port-forward svc/vault -n agentopia 8200:8200 &

# Terminal 2: seed keys
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
export OPENROUTER_API_KEY="sk-or-..."
export AGENTOPIA_GATEWAY_TOKEN="<generate-or-reuse>"

./scripts/init-vault.sh
```

## Step 5: Apply bots ApplicationSet

```bash
kubectl apply -f argocd/agentopia-bots.yaml
```

Starts empty — no bots deployed yet. Provision bots from the web UI.

## Step 6: Deploy bots from UI

Open: `https://demo-chat.dev.nbo.blx-demo.com/ui`

Each bot deploy:
1. Generates config via LLM
2. Creates K8s ConfigMap + Secret
3. Ensures shared memory PVC
4. Commits to GitHub + patches ApplicationSet (requires `bot-config-api-github` secret)
5. ArgoCD syncs → bot pod starts

### Per-bot Telegram token

After bot is deployed, inject the real Telegram token:

```bash
BOT_NAME=<name> TELEGRAM_BOT_TOKEN=<token> ./scripts/set-bot-token.sh
```

## Verification

```bash
# All pods Running
kubectl get pods -n agentopia

# bot-config-api health
curl -s https://demo-chat.dev.nbo.blx-demo.com/health

# Vault unsealed
kubectl exec vault-0 -n agentopia -c vault -- vault status

# Ingress ALB active
kubectl get ingress -n agentopia
```

## Troubleshooting

### ArgoCD Application shows Unknown
- Check `repoURL` in ArgoCD YAML matches actual repo name (`openclaw-infra`, not `agentopia-infra`)
- Check AppProject `infra` allows namespace `agentopia`:
  ```bash
  kubectl get appproject infra -n argocd -o jsonpath='{.spec.destinations}'
  ```
  If missing, add:
  ```bash
  kubectl patch appproject infra -n argocd --type=json \
    -p='[{"op":"add","path":"/spec/destinations/-","value":{"namespace":"agentopia","server":"https://kubernetes.default.svc"}}]'
  ```

### bot-config-api Pending (PVC not found)
PVC `agentopia-shared-shared` is created by the chart when `efs.fileSystemId` is set.
Check EFS StorageClass exists:
```bash
kubectl get sc efs-sc
```

### Bot pod CrashLoopBackOff (config not found)
Mount paths must be `/openclaw-config/` and `/openclaw-state/` (upstream binary paths).
K8s resource names use `agentopia-*` but mount paths stay `/openclaw-*`.

### "GITHUB_TOKEN not set" during bot deploy
Secret `bot-config-api-github` missing. Create it:
```bash
kubectl create secret generic bot-config-api-github \
  -n agentopia \
  --from-literal=GITHUB_TOKEN="ghp_..." \
  --from-literal=GITHUB_REPO="ai-agentopia/agentopia-infra" \
  --from-literal=GITHUB_BRANCH="main"
kubectl rollout restart deployment/bot-config-api -n agentopia
```
Or re-run `create-secrets.sh` with `GITHUB_TOKEN_DEPLOY` set.
