---
title: "Migration: UAT Environment on ai-agentopia"
---

# Migration: UAT Environment on ai-agentopia

## Overview

Create a UAT environment under GitHub org `ai-agentopia`, running alongside the existing DEV environment on the same k3s cluster (server36).

| | DEV (unchanged) | UAT (new) |
|---|---|---|
| **GitHub Org** | `thanhth2813` | `ai-agentopia` |
| **Registry** | `ghcr.io/thanhth2813/` | `ghcr.io/ai-agentopia/` |
| **Cluster** | k3s server36 | k3s server36 (same) |
| **Namespace** | `agentopia` | `agentopia-uat` |
| **ArgoCD** | Shared | Shared |
| **Infra** | Existing | All new, isolated |
| **Bots** | Existing | Created fresh from UI |

**Goal**: Prove 90-95% automation — empty namespace to working platform with minimal manual steps.

**After UAT stable**: `ai-agentopia` becomes primary, Agentopia repos removed from `thanhth2813` (account kept for other projects).

**Estimated time**: 3-4 hours smooth, 1 day with troubleshooting.

---

## Impact Summary

| Repo | Files to change | Occurrences |
|---|---|---|
| `openclaw-infra` | 25 files | 103 |
| `agentopia-protocol` | 10 files | 17 |
| `agentopia-ui` | 2 files | 6 |
| `agentopia` | 2 files | 2 |
| **Total** | **39 files** | **~128** |

Most changes are mechanical find-and-replace. Helm templates are already parameterized via `.Values.namespace` — no template changes needed.

---

## Phase 0 — GitHub Org Setup

**Who**: CTO
**Time**: 10 minutes

### 0.1 Create org

1. Go to https://github.com/organizations/plan
2. Create org: `ai-agentopia`
3. Add yourself as owner

### 0.2 Fork repos

Fork these 4 repos into the new org:

| Source | Fork target | Purpose |
|---|---|---|
| `thanhth2813/openclaw-infra` | `ai-agentopia/openclaw-infra` | Helm charts, ArgoCD apps, scripts |
| `thanhth2813/agentopia-protocol` | `ai-agentopia/agentopia-protocol` | Backend: bot-config-api, gateway extensions, a2a-sidecar |
| `thanhth2813/agentopia-ui` | `ai-agentopia/agentopia-ui` | Frontend: React web UI |
| `thanhth2813/agentopia` | `ai-agentopia/agentopia` | Gateway core (openclaw fork) + agentopia-runtime base image |

### 0.3 Org secrets for CI/CD

Go to `ai-agentopia` org → Settings → Secrets and variables → Actions:

| Secret | Value | Used by |
|---|---|---|
| `GHCR_TOKEN` | GitHub PAT with `write:packages`, `read:packages` | CI image push |

Also enable GitHub Packages for the org:
- Org Settings → Packages → Enable improved container support

### 0.4 GHCR visibility

After first image push, set package visibility:
- Go to org → Packages → each package → Settings → Change visibility to "Public" (or add org read access if private)

---

## Phase 1 — Update Forked Repos

**Who**: Dev
**Time**: 1-2 hours

### 1.1 Clone forked repos

```bash
git clone git@github.com:ai-agentopia/openclaw-infra.git /tmp/uat-openclaw-infra
git clone git@github.com:ai-agentopia/agentopia-protocol.git /tmp/uat-agentopia-protocol
git clone git@github.com:ai-agentopia/agentopia-ui.git /tmp/uat-agentopia-ui
git clone git@github.com:ai-agentopia/agentopia.git /tmp/uat-agentopia
```

### 1.2 String replacements — openclaw-infra

**Registry references** (all files):
```
Find:    ghcr.io/thanhth2813/
Replace: ghcr.io/ai-agentopia/
```

**GitHub repo URLs** (ArgoCD apps, templates, scripts):
```
Find:    https://github.com/thanhth2813/openclaw-infra
Replace: https://github.com/ai-agentopia/openclaw-infra
```

**Repo slug** (secrets template, Python defaults, scripts):
```
Find:    thanhth2813/openclaw-infra
Replace: ai-agentopia/openclaw-infra
```

**Protocol repo references** (docs):
```
Find:    thanhth2813/agentopia-protocol
Replace: ai-agentopia/agentopia-protocol
```

**GHCR user** (scripts, docs):
```
Find:    GHCR_USER (references to "thanhth2813" as username)
Replace: "ai-agentopia"
```

**Files (critical — verify each)**:

| File | What changes |
|---|---|
| `argocd/agentopia-root.yaml` | `repoURL` |
| `argocd/agentopia-root-k3s.yaml` | `repoURL` |
| `argocd/agentopia-base.yaml` | `repoURL` + image repos |
| `argocd/agentopia-base-k3s.yaml` | `repoURL` + image repos + image-updater annotations |
| `argocd/agentopia-base-uat.yaml` | `repoURL` + image repos + image-updater annotations |
| `argocd/agentopia-bots.yaml` | `repoURL` + image repo |
| `argocd/agentopia-bots-k3s.yaml` | `repoURL` + image repo |
| `argocd/agentopia-ui-k3s.yaml` | `repoURL` + image repo + image-updater annotation |
| `charts/agentopia-base/values.yaml` | image repositories |
| `charts/agentopia-base/templates/secrets.yaml` | default `GITHUB_REPO` |
| `charts/agentopia-base/templates/bot-config-api.yaml` | default `ARGOCD_REPO_URL` |
| `charts/agentopia-bot/values.yaml` | `a2a-sidecar` image repo |
| `charts/agentopia-ui/values.yaml` | `agentopia-ui` image repo |
| `bots/kevin-sa/bot.yaml` | gateway image repo |
| `bots/mia-dev/bot.yaml` | gateway image repo |
| `bots/viola-qa/bot.yaml` | gateway image repo |
| `scripts/create-secrets.sh` | GHCR_USER hint, GITHUB_REPO default |
| `.github/workflows/pr-check.yml` | 6 image repo refs in helm lint |
| `.github/workflows/deploy.yml` | 2 image repo refs in helm lint |
| `CLAUDE.md` | registry + repo refs (docs) |
| `DEPLOY.md` | GHCR_USER + repo refs (docs) |
| `docs/Chatbot-architecture.md` | ~33 refs (registry, repo, issue links) |
| `docs/bot-config-api.md` | registry + repo refs |
| `docs/mem0-api.md` | registry ref |
| `docs/A2A.md` | issue link |

### 1.3 String replacements — agentopia-protocol

**Registry** (Dockerfile):
```
Find:    ghcr.io/thanhth2813/
Replace: ghcr.io/ai-agentopia/
```

**Python runtime defaults**:
```
Find:    thanhth2813/openclaw-infra
Replace: ai-agentopia/openclaw-infra

Find:    ghcr.io/thanhth2813/
Replace: ghcr.io/ai-agentopia/
```

**Files (critical)**:

| File | What changes |
|---|---|
| `gateway/Dockerfile` | base image `FROM ghcr.io/ai-agentopia/agentopia-runtime:slim-latest` |
| `bot-config-api/src/services/k8s_service.py` | `BOT_IMAGE_REPOSITORY`, `SIDECAR_IMAGE_REPOSITORY`, `ARGOCD_REPO_URL` defaults |
| `bot-config-api/src/services/llm_generator.py` | `HTTP-Referer` header, image in generated config |
| `bot-config-api/src/services/github_service.py` | `GITHUB_REPO` default |

**Files (low priority — docs/tests)**:

| File | What changes |
|---|---|
| `bot-config-api/src/tests/test_delivery_start.py` | test fixture owner |
| `bot-config-api/src/tests/test_w2_workflow_api.py` | test command string |
| `gateway/extensions/wf-bridge/index.ts` | description example |
| `docs/` (6 files) | various references |

### 1.4 String replacements — agentopia-ui

| File | What changes |
|---|---|
| `.github/workflows/build-image.yml` | image push target `ghcr.io/ai-agentopia/agentopia-ui` |
| `src/__tests__/intent.test.ts` | test fixture owner (low priority) |

### 1.5 String replacements — agentopia (gateway core)

| File | What changes |
|---|---|
| `Dockerfile.runtime` | OCI label `org.opencontainers.image.source` → `https://github.com/ai-agentopia/agentopia` |
| `.github/workflows/build-runtime.yml` | `IMAGE_NAME: ai-agentopia/agentopia-runtime` (was `thanhth2813/agentopia-runtime`) |

**Build order matters**: `agentopia-runtime` must be pushed to `ghcr.io/ai-agentopia/` BEFORE `agentopia-protocol` CI runs, because `gateway/Dockerfile` uses `FROM ghcr.io/ai-agentopia/agentopia-runtime:slim-latest`.

### 1.6 Create UAT ArgoCD root application

Create new file `argocd/agentopia-root-uat.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: agentopia-root-uat
  namespace: argocd
spec:
  project: infra
  source:
    repoURL: https://github.com/ai-agentopia/openclaw-infra
    targetRevision: main
    path: argocd
    directory:
      include: '*-k3s.yaml'   # reuse k3s variants
  destination:
    server: https://kubernetes.default.svc
    namespace: agentopia-uat
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

> **Note**: The k3s ArgoCD app files (`*-k3s.yaml`) already use `local-path` StorageClass and Traefik ingress. These will be reused for UAT. Update the `destination.namespace` in each `-k3s.yaml` file to use `agentopia-uat` — OR create separate `-uat.yaml` variants if you want to keep k3s DEV files unchanged.

**Recommended approach**: Update the `-k3s.yaml` files to use the namespace from the forked repo (since the fork IS the UAT repo). The DEV environment on `thanhth2813` remains untouched.

### 1.7 Pin Helm values to UAT tags

All image tags in UAT must use `uat-1.0.0` instead of `latest`.

**`charts/agentopia-base/values.yaml`**:
```yaml
images:
  mem0Api:
    repository: ghcr.io/ai-agentopia/mem0-api
    tag: "uat-1.0.0"            # was "latest"

botConfigApi:
  image:
    repository: ghcr.io/ai-agentopia/bot-config-api
    tag: "uat-1.0.0"            # was "2.0.0"

llmProxy:
  image:
    repository: ghcr.io/ai-agentopia/agentopia-llm-proxy
    tag: "uat-1.0.0"            # was "latest"
```

**`charts/agentopia-bot/values.yaml`**:
```yaml
image:
  repository: ghcr.io/ai-agentopia/agentopia-gateway
  tag: "uat-1.0.0"              # was "latest"

sidecar:
  image:
    repository: ghcr.io/ai-agentopia/a2a-sidecar
    tag: "uat-1.0.0"            # was "latest"
```

**`charts/agentopia-ui/values.yaml`**:
```yaml
image:
  repository: ghcr.io/ai-agentopia/agentopia-ui
  tag: "uat-1.0.0"              # was "latest"
```

Also update ArgoCD Image Updater annotations in `-k3s.yaml` files to track `uat-*` semver tags (see Phase 2.2).

### 1.8 Parameterize create-secrets.sh namespace

The script currently hardcodes `NAMESPACE="agentopia"` at line 36.

Change to:
```bash
NAMESPACE="${K8S_NAMESPACE:-agentopia}"
```

Also update the DNS references in the script (lines ~203-232) to use `$NAMESPACE`:
```bash
# Before:
DATABASE_URL="postgresql://agentopia:${POSTGRES_PASSWORD}@postgres.agentopia.svc.cluster.local:5432/agentopia"

# After:
DATABASE_URL="postgresql://agentopia:${POSTGRES_PASSWORD}@postgres.${NAMESPACE}.svc.cluster.local:5432/agentopia"
```

Same for Redis URL references.

### 1.9 Commit and push

**Build order**: Push `agentopia` first (runtime base image), then `agentopia-protocol` (depends on runtime), then the rest.

```bash
# 1. agentopia (runtime base image — must be first)
cd /tmp/uat-agentopia
git add -A
git commit -m "chore: migrate references from thanhth2813 to ai-agentopia"
git push origin main
# Wait for agentopia-runtime image to build before proceeding

# 2. agentopia-protocol (depends on runtime base image)
cd /tmp/uat-agentopia-protocol
git add -A
git commit -m "chore: migrate references from thanhth2813 to ai-agentopia"
git push origin main

# 3. agentopia-ui (independent)
cd /tmp/uat-agentopia-ui
git add -A
git commit -m "chore: migrate references from thanhth2813 to ai-agentopia"
git push origin main

# 4. openclaw-infra (Helm charts, ArgoCD apps)
cd /tmp/uat-openclaw-infra
git add -A
git commit -m "chore: migrate references from thanhth2813 to ai-agentopia for UAT"
git push origin main
```

---

## Phase 2 — CI Build First Images

**Who**: Automatic (CI) + manual tag
**Time**: 15-20 minutes

After pushing to `ai-agentopia` repos, GitHub Actions should trigger and build images.

### 2.1 Image Tagging Strategy

UAT only picks **stable versioned images**, not `latest` or `dev-{sha}`.

| Environment | Tag format | Example | Tracking |
|---|---|---|---|
| DEV (`thanhth2813`) | `dev-{sha}` | `dev-a1b2c3d4` | Auto (Image Updater, newest-build) |
| **UAT (`ai-agentopia`)** | **`uat-{semver}`** | **`uat-1.0.0`** | **Manual promotion only** |

**CI workflow must tag images with both `latest` AND `uat-x.y.z`.**

Update CI workflows to add the UAT tag:

**`agentopia-protocol/.github/workflows/build-images.yml`** — add step after build:
```yaml
- name: Tag UAT version
  if: github.ref == 'refs/heads/main'
  env:
    UAT_VERSION: "uat-1.0.0"   # bump manually per stable release
  run: |
    for IMAGE in bot-config-api mem0-api agentopia-gateway agentopia-llm-proxy a2a-sidecar; do
      docker tag ghcr.io/ai-agentopia/${IMAGE}:latest ghcr.io/ai-agentopia/${IMAGE}:${UAT_VERSION}
      docker push ghcr.io/ai-agentopia/${IMAGE}:${UAT_VERSION}
    done
```

**`agentopia-ui/.github/workflows/build-image.yml`** — same pattern:
```yaml
- name: Tag UAT version
  if: github.ref == 'refs/heads/main'
  env:
    UAT_VERSION: "uat-1.0.0"
  run: |
    docker tag ghcr.io/ai-agentopia/agentopia-ui:latest ghcr.io/ai-agentopia/agentopia-ui:${UAT_VERSION}
    docker push ghcr.io/ai-agentopia/agentopia-ui:${UAT_VERSION}
```

### 2.2 ArgoCD Image Updater — UAT config

UAT ArgoCD apps must track `uat-*` tags instead of `dev-*`:

```yaml
# In UAT ArgoCD Application annotations:
argocd-image-updater.argoproj.io/botconfigapi.allow-tags: regexp:^uat-[0-9]+\.[0-9]+\.[0-9]+$
argocd-image-updater.argoproj.io/botconfigapi.update-strategy: semver
```

This means:
- CI pushes `uat-1.0.0` → Image Updater picks it up
- New push with `uat-1.0.1` → Image Updater auto-updates (semver strategy)
- Push without `uat-*` tag → ignored by UAT, only DEV picks up `dev-{sha}`

### 2.3 Promotion flow

```
Dev codes → push to ai-agentopia/main
  → CI builds → pushes :latest and :dev-{sha}
  → DEV auto-picks up dev-{sha} (if DEV still running)

When stable:
  → Bump UAT_VERSION in workflow (e.g., uat-1.0.0 → uat-1.0.1)
  → Push → CI tags :uat-1.0.1
  → UAT Image Updater picks up uat-1.0.1 → pods restart
```

### 2.4 Verify CI triggers

Check each repo's Actions tab:
- `ai-agentopia/agentopia-protocol` → should build: `bot-config-api`, `mem0-api`, `agentopia-gateway`, `agentopia-llm-proxy`, `a2a-sidecar`
- `ai-agentopia/agentopia-ui` → should build: `agentopia-ui`

### 2.5 Troubleshooting CI on forked repos

GitHub Actions on forks may be disabled by default. If workflows don't trigger:

1. Go to repo → Actions tab → "I understand my workflows, go ahead and enable them"
2. Check that org secrets (`GHCR_TOKEN`) are accessible to the repo
3. Verify workflow trigger conditions (push to `main`)

`agentopia-runtime` base image:
- Built from `ai-agentopia/agentopia` repo (`.github/workflows/build-runtime.yml`)
- Update `IMAGE_NAME` in workflow from `thanhth2813/agentopia-runtime` to `ai-agentopia/agentopia-runtime`
- Must be built BEFORE `agentopia-protocol` images (gateway Dockerfile depends on it)

### 2.6 Verify images exist

```bash
docker pull ghcr.io/ai-agentopia/bot-config-api:uat-1.0.0
docker pull ghcr.io/ai-agentopia/mem0-api:uat-1.0.0
docker pull ghcr.io/ai-agentopia/agentopia-gateway:uat-1.0.0
docker pull ghcr.io/ai-agentopia/agentopia-llm-proxy:uat-1.0.0
docker pull ghcr.io/ai-agentopia/a2a-sidecar:uat-1.0.0
docker pull ghcr.io/ai-agentopia/agentopia-ui:uat-1.0.0
```

---

## Phase 3 — Bootstrap UAT Namespace

**Who**: Dev or CTO
**Time**: 15 minutes

### 3.1 SSH to server36

```bash
ssh server36
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### 3.2 Update ArgoCD AppProject

Allow the `agentopia-uat` namespace and the new source repo:

```bash
# Add namespace permission
kubectl patch appproject infra -n argocd --type=json \
  -p='[{"op":"add","path":"/spec/destinations/-","value":{"namespace":"agentopia-uat","server":"https://kubernetes.default.svc"}}]'

# Add source repo permission
kubectl patch appproject infra -n argocd --type=json \
  -p='[{"op":"add","path":"/spec/sourceRepos/-","value":"https://github.com/ai-agentopia/openclaw-infra"}]'
```

### 3.3 Add ArgoCD repo credentials (if repos are private)

```bash
# Skip this if repos are public
kubectl create secret generic argocd-repo-ai-agentopia -n argocd \
  --from-literal=url=https://github.com/ai-agentopia/openclaw-infra \
  --from-literal=username=git \
  --from-literal=password=ghp_YOUR_PAT \
  -o yaml --dry-run=client | kubectl apply -f -

kubectl label secret argocd-repo-ai-agentopia -n argocd \
  argocd.argoproj.io/secret-type=repository
```

### 3.4 Create K8s secrets

```bash
export K8S_NAMESPACE="agentopia-uat"
export VAULT_TOKEN="agentopia-uat-vault-token"
export GHCR_PAT="ghp_..."          # PAT with read:packages for ai-agentopia
export GHCR_USER="ai-agentopia"
export NEO4J_PASSWORD="<choose-new>"
export OPENROUTER_API_KEY="sk-or-..."
export MEM0_LLM_MODEL="google/gemini-2.0-flash-001"

# Optional but recommended:
export GITHUB_TOKEN_DEPLOY="ghp_..."   # PAT with repo scope for ai-agentopia/openclaw-infra
export POSTGRES_PASSWORD="<choose-new>"
export REDIS_PASSWORD="<choose-new>"

./scripts/create-secrets.sh
```

### 3.5 Apply ArgoCD root Application

```bash
kubectl apply -f argocd/agentopia-root-uat.yaml
```

### 3.6 Wait for base infra

```bash
# Watch pods come up (~3-5 minutes)
kubectl get pods -n agentopia-uat -w
```

Expected: Qdrant, Neo4j, Vault, mem0-api, bot-config-api, llm-proxy, Postgres, Redis all `Running`.

### 3.7 Seed Vault with API keys

```bash
# Port-forward Vault (use different local port to avoid conflict with DEV)
kubectl port-forward svc/vault -n agentopia-uat 8201:8200 &

export VAULT_ADDR="http://127.0.0.1:8201"
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
export OPENROUTER_API_KEY="sk-or-..."
export AGENTOPIA_GATEWAY_TOKEN="<choose-new>"

./scripts/init-vault.sh
```

---

## Phase 4 — Verify Base Infra

**Who**: Dev or CTO
**Time**: 15 minutes

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# All pods Running
kubectl get pods -n agentopia-uat

# bot-config-api health
kubectl exec -n agentopia-uat deploy/bot-config-api -- curl -s localhost:8001/health
# Expected: {"status": "ok", "version": "2.0.0"}

# Vault unsealed
kubectl exec -n agentopia-uat vault-0 -c vault -- vault status
# Expected: Sealed = false

# Qdrant healthy
kubectl exec -n agentopia-uat qdrant-0 -- curl -s localhost:6333/healthz

# Neo4j reachable
kubectl exec -n agentopia-uat neo4j-0 -- cypher-shell -u neo4j -p "$NEO4J_PASSWORD" "RETURN 1"

# Postgres reachable
kubectl exec -n agentopia-uat deploy/bot-config-api -- python3 -c "from db.db_migrate import check_connection; check_connection()"
```

### Access bot-config-api UI

```bash
# From local machine (use port 8002 to avoid conflict with DEV on 8001):
ssh -L 8002:localhost:8002 server36 \
  "KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl port-forward -n agentopia-uat svc/bot-config-api 8002:80"

# Open http://localhost:8002/ui
```

---

## Phase 5 — Create Bots from UI

**Who**: CTO
**Time**: 15 minutes

This phase proves the automation. No kubectl, no scripts — just the UI.

1. Open `http://localhost:8002/ui`
2. Click "Deploy Bot"
3. Fill form:
   - Role template: pick any (e.g., "backend")
   - Bot name: choose new name
   - Provider: Claude Sonnet 4.6
   - Telegram Token: from @BotFather (or leave empty for web-only)
4. Click Deploy
5. Watch pipeline:
   - Step 1: LLM generates SOUL.md + USER.md ✅
   - Step 2: K8s ConfigMap + Secret created ✅
   - Step 3: ArgoCD Application CRD created ✅
   - Step 4: ArgoCD syncs → bot pod starts ✅
6. Verify bot pod is Running
7. Open Communication tab → send message → get response

Repeat for 2-3 bots to prove consistency.

---

## Verification Checklist

| Gate | Check | Pass criteria |
|---|---|---|
| **CI** | Images on `ghcr.io/ai-agentopia/` | `docker pull` succeeds for all 6 images |
| **Infra** | All base pods Running | `kubectl get pods -n agentopia-uat` shows all healthy |
| **Vault** | Unsealed + keys accessible | `vault status` shows Sealed=false |
| **UI** | bot-config-api UI accessible | `http://localhost:8002/ui` loads |
| **Bot Deploy** | Create bot from UI → pod Running | Within 5 min, no manual kubectl |
| **Chat** | Send message via web → get response | Bot responds coherently |
| **A2A** | Bot discovers peers + relays | `a2a_discover_agents` returns results |
| **Automation** | Manual steps count | ≤ 6 manual commands total |

---

## Resource Usage (server36)

| Component | DEV (existing) | UAT (new) | Total |
|---|---|---|---|
| CPU | ~3.5 cores | ~3.5 cores | ~7 / 32 |
| RAM | ~8 Gi | ~8 Gi | ~16 / 47 Gi |
| Disk | ~30 Gi | ~30 Gi | ~60 / 135 Gi |

Server36 has plenty of headroom for both environments.

---

## After UAT Stable

1. `ai-agentopia` becomes the primary org for all Agentopia repos
2. Update local git remotes:
   ```bash
   cd personal/openclaw-infra && git remote set-url origin git@github.com:ai-agentopia/openclaw-infra.git
   cd personal/agentopia-protocol && git remote set-url origin git@github.com:ai-agentopia/agentopia-protocol.git
   cd personal/agentopia-ui && git remote set-url origin git@github.com:ai-agentopia/agentopia-ui.git
   cd personal/agentopia && git remote set-url origin git@github.com:ai-agentopia/agentopia.git
   ```
3. Remove Agentopia repos from `thanhth2813` (account kept for other projects)
4. DEV namespace `agentopia` on server36: can be deleted or kept for testing
5. Update Claude memory + CLAUDE.md references

---

## Rollback

If UAT fails:
- DEV environment is completely untouched
- Delete UAT namespace: `kubectl delete namespace agentopia-uat`
- Delete ArgoCD app: `kubectl delete application agentopia-root-uat -n argocd`
- Forked repos can be deleted from org
- Zero impact on existing development
