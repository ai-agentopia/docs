---
title: "Environment Migration Guide"
---

# Environment Migration Guide

How to deploy Agentopia on a new Kubernetes cluster (k3s, EKS, GKE, etc.).

---

## Architecture Overview

```
┌─────────────┐     git push      ┌──────────────┐     sync      ┌─────────────┐
│ bot-config-  │ ──────────────►  │   GitHub      │ ◄──────────  │   ArgoCD    │
│ api (API-    │                  │   (GitOps)    │              │   (cluster) │
│ only)        │                  └──────────────┘              └──────┬──────┘
       │ K8s API                                                       │ Helm
       ▼                                                               ▼
┌──────────────┐                                              ┌──────────────┐
│ ConfigMap +  │                                              │  Bot Pods    │
│ Secret       │                                              │  (gateway)   │
└──────────────┘                                              └──────────────┘
```

Bot deploy/delete goes through **two paths simultaneously**:
- **K8s direct**: ConfigMap + Secret created/deleted immediately
- **Git push**: Application CRD created by bot-config-api → ArgoCD syncs → Helm deploys/prunes pod

---

## Per-Environment Files

Each environment needs **2 ArgoCD manifest files** in `argocd/`:

| File | Kind | Purpose |
|---|---|---|
| `agentopia-base-<env>.yaml` | Application | Shared infra (Qdrant, Neo4j, Vault, mem0, bot-config-api, llm-proxy, postgres, redis) |
| `agentopia-bots-<env>.yaml` | Application CRDs | Bot list — one Application CRD per bot, created directly by bot-config-api |

### Critical: `botsYamlPath`

In the base Application, set `botConfigApi.botsYamlPath` to point to the **correct bots file** for this environment:

```yaml
# argocd/agentopia-base-<env>.yaml
helm:
  valuesObject:
    botConfigApi:
      botsYamlPath: "argocd/agentopia-bots-<env>.yaml"   # ← MUST match
```

If missing or wrong → bot-config-api writes to the **EKS default** (`argocd/agentopia-bots.yaml`), so bots deployed via the API will not appear in the target env.

### Environment-Specific Overrides

Typical values that change per environment:

```yaml
# Storage
qdrant:
  storage:
    storageClass: local-path     # k3s: local-path, EKS: ebs-gp3
neo4j:
  storage:
    storageClass: local-path
vault:
  storage:
    storageClass: local-path
postgres:
  storage:
    storageClass: local-path
redis:
  storage:
    storageClass: local-path

# Ingress
botConfigApi:
  ingress:
    ingressClassName: traefik    # k3s: traefik, EKS: alb
    host: ""                     # empty = match all, or specific domain
    annotations: {}              # EKS needs ALB annotations
```

For bot Application CRDs, override persistence in the Helm template values:

```yaml
# argocd/agentopia-bots-<env>.yaml → template.spec.source.helm.valuesObject
persistence:
  storageClass: local-path       # k3s: local-path, EKS: ebs-gp3
```

---

## Migration Checklist

### Phase 1: Cluster Prep

```
□ Kubernetes running (k3s/EKS/GKE)
□ kubectl access configured
□ StorageClass available (check: kubectl get sc)
□ Ingress controller running (Traefik/ALB/nginx)
```

### Phase 2: ArgoCD

```
□ Install ArgoCD
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

□ Create AppProject 'infra'
    kubectl apply -f - <<'EOF'
    apiVersion: argoproj.io/v1alpha1
    kind: AppProject
    metadata:
      name: infra
      namespace: argocd
    spec:
      sourceRepos: ['*']
      destinations:
        - namespace: agentopia
          server: https://kubernetes.default.svc
        - namespace: argocd
          server: https://kubernetes.default.svc
      clusterResourceWhitelist:
        - group: '*'
          kind: '*'
    EOF
```

### Phase 3: Namespace + Secrets

```
□ kubectl create namespace agentopia

□ Run scripts/create-secrets.sh
    Required env vars:
      VAULT_TOKEN, GHCR_PAT, GHCR_USER, NEO4J_PASSWORD,
      OPENROUTER_API_KEY, GITHUB_TOKEN_DEPLOY

□ Create agentopia-llm-proxy-env secret
    kubectl create secret generic agentopia-llm-proxy-env \
      --namespace agentopia \
      --from-literal=AGENTOPIA_RELAY_TOKEN="<token>"
```

### Phase 4: Create ArgoCD Manifests

```
□ Copy from existing env as template:
    cp argocd/agentopia-base-k3s.yaml argocd/agentopia-base-<env>.yaml
    cp argocd/agentopia-bots-k3s.yaml argocd/agentopia-bots-<env>.yaml

□ Edit base: update storageClass, ingress, botsYamlPath
□ Edit bots: update storageClass, start with elements: []
□ Commit + push to GitHub
```

### Phase 5: Deploy

```
□ kubectl apply -f argocd/agentopia-base-<env>.yaml
□ Wait for all base pods healthy:
    kubectl get pods -n agentopia -w
□ Seed Vault (first time only):
    kubectl port-forward svc/vault -n agentopia 8200:8200
    ./scripts/init-vault.sh
□ kubectl apply -f argocd/agentopia-bots-<env>.yaml
```

### Phase 6: Verify

```
□ bot-config-api health:
    kubectl exec deploy/bot-config-api -n agentopia -- curl -s localhost:8001/health

□ Access health endpoint (SSH tunnel or ingress):
    ssh -L 8001:localhost:8001 <host> \
      "KUBECONFIG=... kubectl port-forward -n agentopia svc/bot-config-api 8001:80"
    → http://localhost:8001/health

□ Deploy a test bot from agentopia-ui → verify pod starts
□ Delete test bot from agentopia-ui → verify pod removed + git cleaned up
```

---

## Bot Deploy Flow (detailed)

```
User clicks "Deploy Bot" on UI
  │
  ├─ Step 1: LLM Generation
  │    bot-config-api → llm-proxy → Codex gpt-5.1
  │    Output: SOUL.md, USER.md, bot.yaml
  │
  ├─ Step 2: K8s Resources (direct API)
  │    Create: agentopia-soul-<bot> ConfigMap
  │    Create: agentopia-gateway-env-<bot> Secret
  │
  ├─ Step 3: Git Push (atomic commit, 4 files) + Application CRD
  │    bots/<bot>/SOUL.md
  │    bots/<bot>/USER.md
  │    bots/<bot>/bot.yaml
  │    argocd/agentopia-bots-<env>.yaml  ← Application CRD created by bot-config-api
  │    (path determined by BOTS_YAML_PATH env var)
  │
  └─ Step 4: ArgoCD Sync (~30s)
       Detects Application CRD → Helm renders agentopia-bot chart
       → Deployment + Service + PVC + token Secret created
       → Pod starts, reads SOUL from ConfigMap, token from Secret
```

## Bot Delete Flow (detailed)

```
User clicks "Delete" on UI
  │
  ├─ Step 1: Git Push (atomic commit)
  │    argocd/agentopia-bots-<env>.yaml  ← element removed
  │    bots/<bot>/ (3 files deleted)
  │
  ├─ Step 2: K8s Cleanup (direct API)
  │    Delete: agentopia-soul-<bot> ConfigMap
  │    Delete: agentopia-gateway-env-<bot> Secret
  │
  └─ Step 3: ArgoCD Sync
       Application CRD removed → ArgoCD cascade prune
       → prune: true → Deployment, Service, Secret deleted
       → PVC retained (Helm reclaim policy: keep)
```

---

## Common Gotchas

| Issue | Cause | Fix |
|---|---|---|
| Bot created but no pod appears | `botsYamlPath` wrong or missing → writes to EKS file | Set correct `botsYamlPath` in base Application |
| Bot deleted but pod stays | Same as above — delete modifies wrong file | Same fix |
| `ImagePullBackOff` | Missing `ghcr-pull-secret` | Run `scripts/create-secrets.sh` with `GHCR_PAT` |
| Application error "project not found" | ArgoCD project `infra` not created | Create AppProject (see Phase 2) |
| Git push step skipped silently | `bot-config-api-github` Secret missing | Create secret with `GITHUB_TOKEN`, `GITHUB_REPO`, `GITHUB_BRANCH` |
| Bot pod starts but Telegram 401 | Token Secret has `REPLACE_ME` placeholder | UI should inject real token; check `agentopia-bot-token-<bot>` secret |
| Vault sealed after restart | Auto-unsealer not running | Check vault-unsealer sidecar logs |
| DNS label overflow (pod crash) | Bot name > 25 chars | bot-config-api enforces 25-char limit (v2.0.0+) |

---

## Existing Environments

| Env | Base File | Bots File | StorageClass | Ingress |
|---|---|---|---|---|
| EKS (nbo-dev-apps) | `agentopia-base.yaml` | `agentopia-bots.yaml` | ebs-gp3 | ALB | *(legacy/decommissioned)* |
| k3s (server36) | `agentopia-base-k3s.yaml` | `agentopia-bots-k3s.yaml` | local-path | Traefik |
