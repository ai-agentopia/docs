---
title: "mem0-api — Memory Service"
---

# mem0-api — Memory Service

> **Version:** 1.0.0
> **Last updated:** 2026-03-06

---

## 1. Overview

`mem0-api` is a stateless Python FastAPI server that wraps the [`mem0ai`](https://github.com/mem0ai/mem0) library. It provides a simple HTTP API for storing and retrieving semantic memories, used by the `agentopia-gateway` via the `mem0-api` plugin.

**Role in the stack:**

```
agentopia-gateway
  └── mem0-api plugin (index.ts)
        └── HTTP → mem0-api:8000
                      ├── OpenRouter API (LLM — extracts memory facts from conversations)
                      ├── OpenRouter API (embedder — text-embedding-3-small, 1536 dims)
                      ├── Qdrant (vector store — stores/searches embeddings)
                      └── Neo4j (graph store — stores entity relationships)
```

**What it does:**
- On `agent_end`: receives the conversation, calls OpenRouter LLM to extract facts (e.g. "Prefers dark mode"), stores as vectors in Qdrant + entity graph in Neo4j
- On `before_agent_start`: searches memories by the user's prompt, injects relevant memories as `<relevant-memories>` context

---

## 2. Source Layout

```
docker/mem0-api/
├── Dockerfile
├── .dockerignore
└── src/
    ├── requirements.txt
    ├── mem0_api_server.py       # FastAPI app — all endpoints
    └── config/
        ├── __init__.py
        └── mem0_config.py       # MEM0_CONFIG dict — all backends configured via env vars
```

---

## 3. API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Health check — returns `{"status": "healthy"}` |
| `GET` | `/` | Version info |
| `POST` | `/v1/memories` | Add memories from a conversation |
| `POST` | `/v1/memories/search` | Search memories by semantic query |
| `GET` | `/v1/memories/{user_id}` | Get all memories for a user |
| `POST` | `/v1/memories/get_all` | Get all memories for a user (POST variant) |
| `DELETE` | `/v1/memories/{memory_id}` | Delete a specific memory by ID |

### POST `/v1/memories`

```json
{
  "messages": [
    { "role": "user", "content": "I prefer dark mode" },
    { "role": "assistant", "content": "Got it!" }
  ],
  "user_id": "openclaw_shared_memory",
  "metadata": {}
}
```

Response:
```json
{
  "results": [
    { "id": "uuid", "memory": "Prefers dark mode", "event": "ADD" }
  ],
  "relations": {}
}
```

### POST `/v1/memories/search`

```json
{
  "query": "user interface preferences",
  "user_id": "openclaw_shared_memory",
  "limit": 5
}
```

Response:
```json
{
  "results": [
    { "id": "uuid", "memory": "Prefers dark mode", "score": 0.476 }
  ]
}
```

---

## 4. Configuration (Environment Variables)

All configuration is via environment variables — no config files inside the container.

### Embedder

The embedder is configured statically in `mem0_config.py` using OpenRouter. No env vars are needed — the `OPENROUTER_API_KEY` already required for the LLM is reused.

| Provider | Model | Dims | Endpoint |
|----------|-------|------|----------|
| OpenRouter | `openai/text-embedding-3-small` | 1536 | `https://openrouter.ai/api/v1` |

> **No local embedding server required.** The old `EMBED_URL`, `EMBED_MODEL`, and `EMBED_API_KEY` env vars are removed. Embeddings are computed via OpenRouter API at ~$0.02/1M tokens (effectively free for personal scale).

### LLM (OpenRouter)

| Var | Value | Notes |
|-----|-------|-------|
| `OPENROUTER_API_KEY` | `sk-or-v1-...` | From K8s Secret `mem0-env` / Vault in local test |
| `MEM0_LLM_MODEL` | `google/gemini-2.0-flash-001` | OpenRouter model ID for memory extraction |

### Vector Store (Qdrant)

| Var | Default | Notes |
|-----|---------|-------|
| `QDRANT_HOST` | `qdrant` | Qdrant service hostname |
| `QDRANT_PORT` | `6333` | |
| `MEM0_COLLECTION` | `agentopia_memory` | Qdrant collection name |

### Graph Store (Neo4j)

| Var | Default | Notes |
|-----|---------|-------|
| `NEO4J_URL` | `bolt://neo4j:7687` | |
| `NEO4J_USERNAME` | `neo4j` | |
| `NEO4J_PASSWORD` | `openclaw` | From K8s Secret `neo4j-auth` |

---

## 5. Python Dependencies

```
mem0ai==1.0.4
fastapi==0.128.8
uvicorn[standard]==0.39.0
qdrant-client==1.16.1
langchain-neo4j         # required for Neo4j graph store (not bundled with mem0ai)
rank-bm25               # required for hybrid search in graph_memory (not bundled with mem0ai)
```

> **Note:** `langchain-neo4j` and `rank-bm25` are NOT included in `mem0ai`'s default install.
> They must be listed explicitly in `requirements.txt` or the server crashes on startup with `ImportError`.

---

## 6. OpenClaw Plugin (Gateway Extension)

The JavaScript extension in `docker/gateway/extensions/mem0-api/` connects the gateway to this server.

### Files

| File | Purpose |
|------|---------|
| `index.ts` | Plugin logic — auto-recall on `before_agent_start`, auto-capture on `agent_end` |
| `openclaw.plugin.json` | Plugin manifest — declares `kind: memory`, defines config schema |
| `package.json` | Plugin package descriptor |

### Plugin Config (in `openclaw.json`)

```json
"plugins": {
  "allow": ["mem0-api"],
  "slots": {
    "memory": "mem0-api"
  },
  "entries": {
    "mem0-api": {
      "enabled": true,
      "config": {
        "apiUrl": "http://mem0-api:8000",
        "userId": "openclaw_shared_memory",
        "autoCapture": true,
        "autoRecall": true,
        "recallTimeoutMs": 3500,
        "captureOnFailure": true,
        "topK": 5,
        "searchThreshold": 0.3
      }
    }
  }
}
```

### Config Schema

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `apiUrl` | string | `http://localhost:8000` | mem0-api service URL |
| `userId` | string | `openclaw_shared_memory` | Memory namespace — use scope-based IDs for multi-bot setups |
| `autoCapture` | boolean | `true` | Capture memories after each `agent_end` event |
| `autoRecall` | boolean | `true` | Inject memories before each `before_agent_start` event |
| `recallTimeoutMs` | number | `3500` | Abort recall if embedder/search is slow — never blocks agent startup |
| `captureOnFailure` | boolean | `true` | Also capture memories from failed/cancelled agent runs |
| `topK` | number | `5` | Max number of memories to inject |
| `searchThreshold` | number | `0.3` | Min similarity score to include a memory (0.0–1.0) |

### userId Strategy

| Deployment | Recommended `userId` |
|-----------|---------------------|
| Single bot, isolated | `bot-{botName}` |
| Multiple bots sharing memory (same scope) | `scope-{scopeName}` |
| All bots share one global memory | `openclaw_shared_memory` |

---

## 7. K8s Deployment (Helm)

`mem0-api` is deployed as part of the `agentopia-base` chart.

```yaml
# charts/agentopia-base/values.yaml (mem0-api section)
mem0api:
  image:
    repository: ghcr.io/ai-agentopia/mem0-api
    tag: "1.0.0"
  env:
    MEM0_LLM_MODEL: "google/gemini-2.0-flash-001"
    MEM0_COLLECTION: "agentopia_memory"
  # OPENROUTER_API_KEY and NEO4J_PASSWORD come from K8s Secrets (mem0-env, neo4j-auth)
  # Embedder is OpenRouter text-embedding-3-small — no extra env vars needed
```

**Secrets needed before deploy (via `create-secrets.sh`):**

```bash
kubectl create secret generic mem0-env \
  --namespace agentopia \
  --from-literal=OPENROUTER_API_KEY=sk-or-v1-... \
  --from-literal=MEM0_LLM_MODEL=google/gemini-2.0-flash-001 \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## 8. Local Dev Setup

**Prerequisites:** `openclaw-net` Podman network exists, Qdrant + Neo4j running on the network. No local embedding server required — embeddings go through OpenRouter API.

```bash
# Start mem0-api
podman run -d --name mem0-api --network openclaw-net \
  -e QDRANT_HOST=qdrant \
  -e NEO4J_URL=bolt://neo4j:7687 \
  -e NEO4J_PASSWORD=openclaw \
  -e MEM0_LLM_MODEL=google/gemini-2.0-flash-001 \
  -e OPENROUTER_API_KEY=sk-or-v1-... \
  localhost/mem0-api:1.0.0

# Verify
podman exec mem0-api python3 -c "
import urllib.request, json
res = urllib.request.urlopen('http://localhost:8000/health')
print(json.loads(res.read()))
"
```

**Test memory write/search:**

```bash
podman exec mem0-api python3 -c "
import urllib.request, json

# Write
data = json.dumps({
  'messages': [
    {'role': 'user', 'content': 'I prefer dark mode'},
    {'role': 'assistant', 'content': 'Got it!'}
  ],
  'user_id': 'test'
}).encode()
req = urllib.request.Request('http://localhost:8000/v1/memories',
  data=data, headers={'Content-Type': 'application/json'})
r = json.loads(urllib.request.urlopen(req).read())
print('Stored:', [x['memory'] for x in r.get('results',[])])

# Search
data = json.dumps({'query': 'UI preferences', 'user_id': 'test', 'limit': 3}).encode()
req = urllib.request.Request('http://localhost:8000/v1/memories/search',
  data=data, headers={'Content-Type': 'application/json'})
r = json.loads(urllib.request.urlopen(req).read())
print('Found:', [(x['memory'], round(x['score'],3)) for x in r.get('results',[])])
"
```

---

## 9. Troubleshooting

### `ImportError: langchain_neo4j is not installed`

```
ImportError: Could not import MemoryGraph for provider 'neo4j': langchain_neo4j is not installed
```

**Fix:** Add `langchain-neo4j` to `requirements.txt` and rebuild the image. This package is not bundled with `mem0ai`.

### `ImportError: rank_bm25 is not installed`

```
ImportError: rank_bm25 is not installed. Please install it using pip install rank-bm25
```

**Fix:** Add `rank-bm25` to `requirements.txt` and rebuild. Required for hybrid search in the graph memory module.

### mem0-api slow to initialize

On first startup, mem0ai checks/creates the Qdrant collection and Neo4j schema. This takes 5–15 seconds. The `start-period: 15s` in the HEALTHCHECK accounts for this.

### Memory capture not working

Check gateway logs for `mem0-api: capture failed`. Common causes:
- mem0-api container not reachable (wrong `apiUrl` in plugin config)
- OpenRouter API key missing or invalid (LLM extraction fails)
- Neo4j not ready (graph store connection error)
