---
title: "Mem0 OSS Production Setup - Summary"
---

# Mem0 OSS Production Setup - Summary

> **Note:** This document describes the initial standalone deployment (2026-02-21).
> mem0-api now runs as a Kubernetes pod in the agentopia namespace on the k3s cluster.

**Date:** 2026-02-21
**Status:** Production Ready

## Infrastructure

**Server:** server36 (Ubuntu 20.04, 47GB RAM, 32 CPUs)

**Stack:**
- Docker Compose v2.x
- Qdrant v1.13.0 (Vector DB)
- Neo4j v5.26 (Graph DB)
- Ollama latest (LLM Runtime)

**Models:**
- qwen2.5:1.5b (986MB) - LLM for fact extraction
- nomic-embed-text (274MB) - 768-dim embeddings

## Performance

**Benchmarks:**
- Fact extraction: 11.6s (complex content)
- Embedding generation: 200-500ms
- Search relevance: 0.93 score
- Processing time: ~60-80s (async mode)

**Model Selection:**
- Tested: 0.5b, 1.5b, 3b, llama3.2:1b, phi3, gemma2
- Winner: qwen2.5:1.5b
- Reason: 40% faster than 3b, same quality, no hallucinations

## Configuration

**Shared Memory:**
- User ID: agentopia_shared_memory
- All 3 bots share same memory pool
- AutoCapture: Enabled
- AutoRecall: Enabled

**API:**
- Endpoint: http://server36:8000
- Health: http://server36:8000/health
- Service: systemd (mem0-api.service)

## Services Status

Check with:
```bash
systemctl status mem0-api
docker-compose ps
```

All services should show "active (running)" or "Up".

## Cleanup Applied

- Removed test scripts
- Removed unused models (0.5b, 3b, llama3.2, etc.)
- Cleaned Qdrant (0 test memories)
- Production-ready state

## Next Steps

1. Agentopia plugin captures conversations automatically
2. Memories stored in Qdrant + Neo4j
3. All 3 bots can search and recall shared memory
4. $0 cost, completely local
