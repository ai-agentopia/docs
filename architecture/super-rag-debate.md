---
title: "Super RAG Debate — CTO Architecture Decision Session"
description: "Formal debate on production-ready Super RAG for Agentopia. Grounded in current implementation, not theory."
---

# Super RAG — CTO Debate Session

> **Date**: 2026-03-31 (revised after CTO pushback)
> **Participants**: CTO, Platform/DevOps
> **Status**: READY_FOR_CTO_REVIEW
> **Assumption**: English-first production. Vietnamese/multilingual is optional enhancement.
> **Revision**: v2 — fixes evaluation logic, demotes Contextual Retrieval, honest current-state labels, separates evidence types, adds service architecture assessment

---

## A. Current State Assessment

### What Exists Today

| Component | Evidence (repo) | Readiness |
|---|---|---|
| **Scoped knowledge model** | `{client_id}/{scope_name}`, collection-per-scope in Qdrant, server-side bot→scope resolution. 18 auth tests. | **Implemented** — code complete, tested |
| **File-upload ingestion** | PDF/HTML/Markdown/code parsing, 3 chunking strategies, SHA-256 dedup, two-phase atomic replace | **Implemented** — production-grade patterns |
| **Postgres document lifecycle** | `document_records` table: active/superseded/deleted states, tombstone ledger | **Implemented** — tested |
| **Provenance & citations** | Source, section, page, chunk_index, score, ingested_at, document_hash — 35 provenance tests | **Implemented** — tested |
| **Runtime retrieval plugin** | Gateway extension (priority 10), 5s timeout, non-blocking, XML injection with D7 answer contract | **Production-pending** — Helm config gap silently disables plugin (F0.1) |
| **Vector search** | Qdrant v1.16.1, `qwen/qwen3-embedding-8b` (1024d), cosine similarity, topK=5 | **Operationally present** — works, but no retry/circuit breaker on embedding API |
| **Memory system** | mem0-api: Qdrant (semantic) + Neo4j (graph), separate from knowledge | **Operationally present** — running, patched for Neo4j edge cases |
| **Operator Knowledge UI** | Client-first navigation, scope browser, upload/delete/search | **Implemented** — tested |
| **Evaluation framework** | ADR-014: 8 criteria (3 hard at 100%), 6 scenario types, 7 evaluation artifacts, 22 contract tests | **Design-complete, automation-pending** — no retrieval quality metrics yet |
| **Test suite** | 150+ knowledge-specific tests, 2,494 total bot-config-api tests | **Implemented** — covers contracts, not retrieval quality |
| **Live pilot gate** | #307 OPEN — requires real client documents + manual evaluation | **Blocked** — business dependency on first client |

### What Works Well (repo-proven)

1. **Ingestion pipeline** — Two-phase atomic replace, hash-based dedup, tombstone lifecycle. Enterprise-grade data management. Tested.
2. **Scope isolation** — Server-side resolution, zero cross-scope leakage. ADR-008/009 locked. Tested.
3. **Non-blocking retrieval** — Gateway plugin times out gracefully. Bot always answers. Correct architecture.
4. **Evaluation design** — ADR-014 defines 8 concrete criteria with thresholds. 3 hard requirements automated. ~60% complete (design + contract tests), not 0%.

### What Is Weak (honest)

1. **Retrieval quality is unmeasured** — Pure vector search, no hybrid, no reranking. No data on whether top-5 results are actually relevant.
2. **No quality metrics** — Cannot answer "how good is our retrieval?" No nDCG, MRR, or faithfulness measurement.
3. **Configuration fragility** — Plugin Helm config gap, hardcoded embedding timeout (30s), no retry/circuit breaker.
4. **No observability** — Zero retrieval-specific metrics. Cannot diagnose production issues.
5. **Monolithic knowledge service** — All RAG logic (ingestion, embedding, search, lifecycle) lives inside `bot-config-api`. See Section J for analysis.

---

## B. Critical Review of Existing Blueprint

### Problems Identified

**1. Overstates gaps**: Blueprint scored 37% by counting 60 items from a generic RAG wishlist including non-RAG items (Reasoning Engine, Tooling Layer, RLHF). Production-relevant coverage is ~55-60%.

**2. Underweights strengths**: Ingestion pipeline is production-grade. Evaluation framework is ~60% complete, not 0%.

**3. Graph RAG too early**: Should be explicitly deferred, not in production path.

**4. Missing Phase 0**: Known fragility must be fixed before quality measurement.

**5. Evaluation logic was inconsistent**: Mixed reference-free metrics (RAGAS) with labeled ranking metrics (nDCG/MRR) as if interchangeable. They are not. Fixed in this revision (see Section E).

**6. Contextual Retrieval was over-promoted**: Promoted to default Phase 3 based on Anthropic benchmarks alone, without Agentopia-specific validation. Demoted to conditional candidate in this revision.

---

## C. Candidate Solution Options — Debate

### C1. Retrieval Strategy

#### Option A: Vector-only baseline (current)
- **Repo evidence**: Works today. `knowledge.py` uses `QdrantBackend.search_scope()` with cosine similarity, topK=5.
- **Gap**: Fails on exact keyword matches (error codes, API names). No empirical quality data.
- **Verdict**: Sufficient for MVP. Not optimal.

#### Option B: Vector + keyword hybrid (Qdrant native)
- **Repo evidence**: Qdrant v1.16.1 deployed. Current code uses only `client.search(query_vector)` — does not use Qdrant's native text indexing or sparse vectors.
- **External evidence**: Qdrant v1.15+ supports [native BM25 with server-side IDF](https://qdrant.tech/articles/sparse-vectors/). [Hybrid search in one request via prefetch + RRF](https://qdrant.tech/articles/hybrid-search/). [Hybrid improves recall 1-9%](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking) over vector-only.
- **Cost**: $0 — uses existing Qdrant instance.
- **Verdict**: Best ROI. Uses deployed capability we're not using.

#### Option C: Vector + keyword + reranker
- **External evidence**: [+28% nDCG@10](https://www.zeroentropy.dev/articles/ultimate-guide-to-choosing-the-best-reranking-model-in-2025) (ZeroEntropy). [48% quality improvement](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking) (Pinecone, hybrid+reranker vs single-method). But: ["A reranker can only reorder what retrieval already found"](https://docs.bswen.com/blog/2026-02-25-hybrid-search-vs-reranker/).
- **Verdict**: High value, but recall (hybrid) must come before precision (reranker).

**Recommendation: Option B first. Option C only if labeled eval data from Phase 1 shows hybrid is insufficient.**

### C2. Reranker Choice (if needed — conditional)

- **Local ONNX**: [BAAI/bge-reranker-v2-m3 — 278M params, 51.8 nDCG@10 on BEIR, Apache 2.0](https://docs.bswen.com/blog/2026-02-25-best-reranker-models/). Runs on CPU for <100 pairs.
- **API**: [Cohere Rerank ~$1/1K queries, ~8-11% improvement](https://lancedb.com/blog/benchmarking-cohere-reranker-with-lancedb/). Lower ops burden.
- **Alternative**: [Jina Reranker v3 — 81.3% Hit@1, 188ms, Apache 2.0](https://docs.bswen.com/blog/2026-02-25-best-reranker-models/).

**Decision deferred until Phase 2 eval data exists.**

### C3. Search Backend

- **Qdrant native** (recommended): Already deployed. [Native BM25 since v1.15](https://qdrant.tech/articles/sparse-vectors/). Zero new services.
- **Elasticsearch**: Overkill for <10K documents. [80.5% of enterprise RAG uses FAISS/ES](https://www.mdpi.com/2076-3417/16/1/368), but Qdrant native hybrid is adequate.

**Recommendation: Qdrant native. External search engine REJECTED for current scale.**

### C4. Ingestion Evolution

**Repo evidence**: Current 3 strategies (fixed-size, paragraph, code-aware) are working and tested. Semantic chunking defined in enum but intentionally deferred.

**All ingestion improvements are conditional on Phase 1 eval data showing chunk quality is a bottleneck.**

### C5. Graph RAG — DEFERRED

**Repo evidence**: Neo4j exists for mem0 graph memory. Knowledge system is separate.
**External evidence**: [Neo4j advocates Graph RAG](https://neo4j.com/blog/genai/advanced-rag-techniques/) but industry consensus: ["RAG remains the grounding mechanism"](https://datanucleus.dev/rag-and-agentic-ai/what-is-rag-enterprise-guide-2025) — ground basics first.

**Explicitly deferred. No client demand. High complexity. Needs retrieval foundation first.**

### C6. Evaluation Strategy

#### Two distinct evaluation layers (CRITICAL DISTINCTION)

**Layer 1 — Reference-free quality signals (RAGAS)**:
- [Faithfulness, Context Precision, Answer Relevancy](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/) via LLM-as-judge
- No golden dataset needed. Can start immediately after Phase 0.
- Useful as **early signal** and **directional regression check**
- Limitation: LLM-as-judge has biases. NOT a substitute for labeled metrics.

**Layer 2 — Labeled retrieval ranking metrics**:
- nDCG@5, MRR, Precision@5 — require **human-labeled relevance judgments**
- Need golden dataset: query + expected relevant chunks with graded relevance
- This is the **authoritative quality gate** for Phase 2 improvement claims
- Cannot be shortcut. RAGAS does NOT produce these.

**Recommendation**: Start with RAGAS (Layer 1) as early signal in Phase 1a. Build labeled dataset from #307 pilot + client collaboration for Layer 2 in Phase 1b. Phase 2 eval gate MUST use labeled nDCG@5, not RAGAS alone.

---

## D. Recommended Target Architecture

### Target Retrieval Stack

```
Query
  ↓
Qdrant vector search (cosine, top-20)     ← existing, increase limit
  +
Qdrant BM25 search (sparse, top-20)       ← NEW: enable native BM25
  ↓
Reciprocal Rank Fusion (RRF)              ← NEW: merge + deduplicate
  ↓
Top-5 by fused score
  ↓
Confidence threshold (configurable)        ← existing, expose via Helm
  ↓
Token budget enforcement                   ← existing
  ↓
XML injection into LLM context             ← existing
```

**No reranker in V1.** Added only if labeled eval data proves hybrid is insufficient.

### Target Ingestion Stack

**No changes in production V1.** Current pipeline is production-grade (repo-proven).

### Target Evaluation Approach

| Layer | What | When | Gate Type |
|---|---|---|---|
| **1a. RAGAS** | Faithfulness + Context Precision (reference-free) | Phase 1 immediate — no golden dataset needed | Early signal, directional regression |
| **1b. Labeled baseline** | nDCG@5, MRR, Precision@5 against golden queries | Phase 1 after #307 pilot produces labeled data | **Authoritative quality gate** |
| **2. Eval gate** | nDCG@5 improvement ≥ 10% over labeled baseline | Phase 2 hybrid search | Hard gate — must use labeled metrics, not RAGAS |

### Target Observability

| Metric | Type | Purpose |
|---|---|---|
| `knowledge_search_latency_seconds` | Histogram | p50/p95/p99 retrieval latency |
| `knowledge_search_results_count` | Histogram | Result count distribution per query |
| `knowledge_search_top_score` | Histogram | Best result confidence |
| `knowledge_ingest_chunks_total` | Counter | Indexed data volume |
| `knowledge_embedding_latency_seconds` | Histogram | OpenRouter API latency |
| `knowledge_search_errors_total` | Counter (by type) | Qdrant/embedding/timeout failures |

### Explicit Deferrals

| Item | Reason | Revisit When |
|---|---|---|
| **Graph RAG** | No demand. High complexity. Needs retrieval foundation. | Multi-hop query failures in production |
| **Reranker** | Adds latency + complexity. Hybrid may suffice. | Phase 2 labeled eval shows hybrid insufficient |
| **Contextual Retrieval** | Promising ([Anthropic: -49% failure rate](https://www.anthropic.com/news/contextual-retrieval)) but unvalidated on Agentopia data. | Phase 2 eval data + cost-benefit analysis |
| **Semantic chunking** | Current chunking untested against quality metrics. | Phase 1 eval shows chunk quality is bottleneck |
| **Agentic RAG** | [Adds unpredictability](https://towardsdatascience.com/agentic-rag-vs-classic-rag-from-a-pipeline-to-a-control-loop/). Injection-based is correct for our use case. | Multi-hop demand proven |
| **Auto-ingestion** | Manual upload sufficient for first clients. | Scale demand |
| **Multilingual optimization** | English-first assumption. | Non-English client demand |

---

## E. Execution Phases

### Phase 0: Foundation Hardening (prerequisite)

**Why**: Known fragility that will cause production incidents. Must fix before quality measurement is meaningful.

| Task | Description | Risk if Skipped |
|---|---|---|
| **F0.1** Fix knowledge plugin Helm config | Add `knowledgeRetrieval` section to `configmap-config.yaml` | Knowledge injection silently disabled for new bots |
| **F0.2** Retry + exponential backoff on embedding API | `tenacity.retry`, 3 attempts, 1s→2s→4s | Transient OpenRouter failures crash ingest |
| **F0.3** Circuit breaker on OpenRouter | 5 failures → fast-fail 5 minutes | Cascading failures during API outage |
| **F0.4** Expose embedding timeout as env var | Replace hardcoded 30s | Cannot tune without redeploy |
| **F0.5** Expose vector dimension as env var | Replace hardcoded 1024 | Model change requires code change |
| **F0.6** Qdrant health check endpoint | `/health` returns reachability | Silent search failures |
| **F0.7** `score_threshold` in Qdrant search | Native param vs post-filter | Slightly more efficient |

**Scope**: Code changes only. 1-2 PRs.

### Phase 1: Evaluation Baseline

**Two sub-phases reflecting the two evaluation layers:**

#### Phase 1a: RAGAS early signal (no golden dataset needed)

| Task | Description |
|---|---|
| **E1a.1** Integrate RAGAS Python library | Add to bot-config-api test dependencies |
| **E1a.2** Implement reference-free checks | Faithfulness + Context Precision on sample queries |
| **E1a.3** CI regression gate (RAGAS) | Flag if faithfulness drops >15% between PRs — **directional signal only** |

#### Phase 1b: Labeled retrieval baseline (requires client data)

| Task | Description |
|---|---|
| **E1b.1** Complete #307 live pilot | First client, 20+ queries, 6 scenarios |
| **E1b.2** Build golden dataset from pilot | Query + expected relevant chunks + graded relevance (0/1/2) |
| **E1b.3** Implement labeled metrics | nDCG@5, MRR, Precision@5 computed against golden data |
| **E1b.4** Baseline metrics report | Document: current nDCG@5, MRR, citation accuracy, latency p95 |
| **E1b.5** CI eval gate (labeled) | Fail if nDCG@5 drops >10% — **authoritative gate** |

**Dependency**: Phase 1b requires first client documents (#307). Phase 1a can start immediately after Phase 0.

### Phase 2: Retrieval Quality (Hybrid Search)

| Task | Description |
|---|---|
| **R2.1** Enable Qdrant native BM25 | Sparse vector config + text tokenization per collection |
| **R2.2** Implement hybrid search | Vector + BM25 parallel query via Qdrant prefetch API |
| **R2.3** Implement RRF fusion | `score = 1/(k + rank_vector) + 1/(k + rank_text)` |
| **R2.4** Collection migration | Re-create collections with BM25 support (migration script) |
| **R2.5** Observability metrics | Prometheus counters/histograms for retrieval stack |
| **R2.6** **Labeled eval gate** | Require nDCG@5 ≥ 10% improvement over Phase 1b baseline. **Must use labeled metrics, not RAGAS.** |

**Cost**: $0 — Qdrant native features.

### Phase 3+: Conditional Experiments (only if Phase 1/2 data justifies)

**No default recommendation for Phase 3.** All options are candidates, not commitments:

| Candidate | Trigger Condition | External Evidence |
|---|---|---|
| **Contextual Retrieval** | Phase 2 eval shows ingestion-side chunk quality limits recall | [Anthropic: -49% failure rate](https://www.anthropic.com/news/contextual-retrieval), [$1.02/1M tokens one-time](https://www.anthropic.com/news/contextual-retrieval) |
| **Semantic chunking** | Phase 1 eval shows fixed-size chunking breaks semantic boundaries | Defined in code enum, not yet implemented |
| **Reranker** | Phase 2 labeled nDCG@5 plateaus below target | [+28% nDCG@10 possible](https://www.zeroentropy.dev/articles/ultimate-guide-to-choosing-the-best-reranking-model-in-2025) |
| **Query expansion** | Phase 1 eval shows short/ambiguous queries have low recall | External directional evidence only |

**Each candidate requires Agentopia-specific eval data before promotion to committed path.**

### Phase 4+: Post-Production

| Track | Trigger |
|---|---|
| **Graph RAG** | Multi-hop query failures identified in production |
| **Agentic RAG** | Injection-based consistently misses relevant context |
| **Caching** | Observability shows >30% duplicate queries within 5min |
| **Auto-ingestion** | Client requests automated document sync |

---

## F. Decision Points — Recommendations

### 1. Approve Phase 0 foundation hardening before evaluation?

**YES.** Mandatory. Known fragility (Helm config gap, no retry, hardcoded values) causes production incidents and unreliable eval metrics. Small scope (1-2 PRs).

### 2. Approve RAGAS as Phase 1 evaluation framework?

**YES, as Layer 1 early signal only.** RAGAS provides reference-free faithfulness/context precision checks without golden data. But it does NOT replace labeled retrieval metrics (nDCG/MRR). Phase 1 has two sub-phases: 1a (RAGAS, immediate) and 1b (labeled baseline, after #307 client data).

### 3. Approve Qdrant native hybrid as Phase 2?

**YES.** Zero infrastructure cost. Uses Qdrant features already deployed but unused. Eval gate uses **labeled nDCG@5**, not RAGAS.

### 4. Approve Graph RAG deferral?

**YES.** No client demand. High complexity. No quality metrics to measure impact yet.

### 5. Minimum production-ready eval gate?

**Phase 1a** (RAGAS directional signal) + **#307 manual pilot** (CTO sign-off) for initial production.
**Phase 1b** (labeled nDCG@5 baseline) required before Phase 2 claims can be validated.

### 6. Required vs optional scope?

| Level | Items |
|---|---|
| **REQUIRED for production** | Phase 0, #307 completion, RAGAS early signal, basic observability |
| **REQUIRED for quality claims** | Phase 1b labeled baseline, Phase 2 with labeled eval gate |
| **CONDITIONAL** | All Phase 3+ candidates — evidence-driven only |
| **DEFERRED** | Graph RAG, Agentic RAG, auto-ingestion, multilingual |

---

## G. Decision Summary

| Decision | Verdict | Evidence Type |
|---|---|---|
| Phase 0 before evaluation | **RECOMMENDED** | Repo: known fragility in code |
| RAGAS as early signal | **RECOMMENDED** | External: [RAGAS framework](https://docs.ragas.io/en/stable/concepts/metrics/). Limitation: not a substitute for labeled metrics |
| Labeled nDCG/MRR as authoritative gate | **RECOMMENDED** | Industry standard: [nDCG correlates with RAG quality](https://labelyourdata.com/articles/llm-fine-tuning/rag-evaluation) |
| Hybrid search via Qdrant native | **RECOMMENDED** | Repo: Qdrant v1.16.1 deployed. External: [native BM25 since v1.15](https://qdrant.tech/articles/sparse-vectors/) |
| Reranker | **DEFERRED** | Conditional on Phase 2 labeled eval |
| Graph RAG | **DEFERRED** | No demand, high complexity |
| Contextual Retrieval | **CONDITIONAL candidate** | External: [Anthropic benchmarks](https://www.anthropic.com/news/contextual-retrieval). Needs Agentopia-specific validation |
| Semantic chunking | **CONDITIONAL candidate** | Repo: enum defined, not implemented. Needs eval data |
| External search engine | **REJECTED** | Qdrant native sufficient. <10K docs year 1 |
| Knowledge service extraction | **RECOMMENDED for Phase 2** | Repo: monolithic bot-config-api. See Section J |

---

## H. Market Research — Industry Context

> **Evidence separation**: This section provides **external directional evidence**. It informs but does not settle architecture decisions. Repo-specific evidence is in Sections A-E.

### H1. Industry Standard Pipeline

Production RAG has converged on three stages: recall → fusion → precision.
- [Hybrid improves recall 1-9%](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking) over vector-only
- [Reranker adds +28% nDCG@10](https://www.zeroentropy.dev/articles/ultimate-guide-to-choosing-the-best-reranking-model-in-2025) on top of hybrid
- [Full pipeline: 48% quality improvement](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking) vs single-method (Pinecone)
- Critical: ["Reranker can only reorder what retrieval already found"](https://docs.bswen.com/blog/2026-02-25-hybrid-search-vs-reranker/) — recall first, precision second

### H2. Qdrant Native Hybrid

- [Native BM25 with server-side IDF since v1.15](https://qdrant.tech/articles/sparse-vectors/)
- [Hybrid in one request via prefetch + RRF](https://qdrant.tech/articles/hybrid-search/)
- [Rust + SIMD inverted index](https://qdrant.tech/articles/sparse-embeddings-ecommerce-part-1/) — production-grade performance

### H3. Contextual Retrieval (Anthropic)

- [Contextual Embeddings + BM25: -49% retrieval failure rate](https://www.anthropic.com/news/contextual-retrieval)
- [With reranking: -67%](https://www.anthropic.com/news/contextual-retrieval)
- [Cost: $1.02/1M doc tokens one-time](https://www.anthropic.com/news/contextual-retrieval)
- [AWS Bedrock has native support](https://aws.amazon.com/blogs/machine-learning/contextual-retrieval-in-anthropic-using-amazon-bedrock-knowledge-bases/)
- **Note**: These are Anthropic's benchmarks, not Agentopia-validated. Remains conditional candidate.

### H4. Evaluation Frameworks

- [RAGAS: reference-free, LLM-as-judge, Python-native](https://docs.ragas.io/en/stable/concepts/metrics/)
- [nDCG correlates most strongly with end-to-end RAG quality](https://labelyourdata.com/articles/llm-fine-tuning/rag-evaluation) — requires labeled data
- Industry: ["test sets combine golden data, synthetic queries, and human review"](https://www.evidentlyai.com/llm-guide/rag-evaluation)

### H5. Enterprise Stats

- [63.6% use GPT-based models; 80.5% use FAISS/ES](https://www.mdpi.com/2076-3417/16/1/368)
- [Retrieval reduces hallucinations by 47%](https://www.aplyca.com/en/blog/ultimate-guide-to-rag-for-enterprise-use-cases-platforms-and-production-best)
- [Prototype-to-production gap spans months](https://introl.com/blog/rag-infrastructure-production-retrieval-augmented-generation-guide)

### H6. Agentic vs Injection-Based RAG

- [Agentic: control loop, multi-hop, tool-based](https://arxiv.org/abs/2501.09136)
- [Classic RAG better when "predictable cost and latency are priorities"](https://towardsdatascience.com/agentic-rag-vs-classic-rag-from-a-pipeline-to-a-control-loop/)
- [Enterprises blend both — "agents orchestrate, RAG grounds"](https://datanucleus.dev/rag-and-agentic-ai/what-is-rag-enterprise-guide-2025)
- **Agentopia alignment**: Injection-based is correct for current use case. Agentic is Phase 4+.

---

## J. Service Architecture Assessment — Microservice Readiness

> **Client requirement**: RAG components should be extractable to independent services, not locked into a monolithic bot-config-api or gateway.

### Current State (repo-proven)

All knowledge/RAG logic currently lives inside **`bot-config-api`**:

| Module | Location | Responsibility | Lines |
|---|---|---|---|
| `knowledge.py` | `bot-config-api/src/services/` | KnowledgeService, QdrantBackend, chunking, embedding, search | ~780 |
| `knowledge router` | `bot-config-api/src/routers/` | REST API: ingest, search, delete, lifecycle | ~350 |
| `document_store.py` | `bot-config-api/src/services/` | PostgresDocumentStore, lifecycle management | ~270 |
| `bot_knowledge_index.py` | `bot-config-api/src/services/` | Bot→scope binding resolution | ~135 |
| `knowledge models` | `bot-config-api/src/models/` | Pydantic schemas | ~170 |
| **Total** | | | **~1,700 lines** |

Gateway side:
| Module | Location | Responsibility | Lines |
|---|---|---|---|
| `knowledge-retrieval/index.ts` | `gateway/extensions/` | Plugin: query → API call → XML injection | ~230 |

### Problem Analysis

**bot-config-api has too many responsibilities**:
1. Bot CRUD + deployment (Application CRD management)
2. SOUL/USER prompt generation
3. Knowledge ingestion + search + lifecycle
4. A2A protocol server
5. Workflow/Temporal integration
6. Usage metrics

Knowledge is **~1,700 lines** of domain-specific logic (embedding, chunking, vector search, document lifecycle) that shares nothing with bot deployment except the Qdrant connection and Postgres instance.

**Risks of keeping it monolithic**:
- Embedding API circuit breaker affects bot deployment availability
- Knowledge ingest load (CPU-intensive chunking + API calls) competes with bot-config-api request handling
- Cannot scale knowledge processing independently
- Cannot deploy knowledge service updates without redeploying bot-config-api
- Harder to test knowledge in isolation

### Extraction Plan — `knowledge-api` Service

**Recommended for Phase 2** (when hybrid search adds complexity):

```
Current:
  bot-config-api
    ├── bot CRUD / deploy
    ├── knowledge (ingest, search, lifecycle)  ← EXTRACT
    ├── A2A protocol
    └── workflow integration

Target:
  bot-config-api                  knowledge-api (NEW)
    ├── bot CRUD / deploy           ├── ingest (chunking, embedding, Qdrant write)
    ├── A2A protocol                ├── search (vector, BM25, RRF, token budget)
    └── workflow integration        ├── lifecycle (Postgres document_records)
                                    ├── scope resolution (bot→scope index)
                                    └── health / metrics

  gateway
    └── knowledge-retrieval plugin → calls knowledge-api (not bot-config-api)
```

**What changes**:
- New `knowledge-api` FastAPI service with its own Dockerfile, Helm chart, ArgoCD app
- Gateway plugin `apiUrl` points to `knowledge-api` instead of `bot-config-api`
- `bot-config-api` no longer imports knowledge modules
- Shared Postgres (same DB, different tables) and shared Qdrant (same instance, scoped collections)

**What stays shared**:
- Qdrant instance (collection-per-scope isolation is sufficient)
- Postgres instance (knowledge tables are already independent)
- Vault secrets (same OPENROUTER_API_KEY)

**Benefits**:
- Independent scaling: knowledge ingest can burst without affecting bot API
- Independent deployment: knowledge code changes don't require bot-config-api redeploy
- Clear domain boundary: knowledge service owns its data model entirely
- Easier to add hybrid search, reranker, caching as internal concerns
- Testing in isolation: knowledge service has its own test suite

**Migration approach** (non-breaking):
1. Extract code into new service, keep API contract identical
2. Deploy knowledge-api alongside bot-config-api
3. Update gateway plugin + bot-config-api proxy to route to knowledge-api
4. Remove knowledge code from bot-config-api
5. Each step is a separate PR, rollback-safe

**Timing**: Start extraction in Phase 2 alongside hybrid search. The hybrid search code naturally goes into the new service.

---

## K. Final Recommended Production Path

```
Phase 0: Foundation Hardening
  → Helm config fix, retry/circuit breaker, health checks, env vars
  → Scope: 1-2 PRs, code-only

Phase 1a: RAGAS Early Signal (immediate after Phase 0)
  → Reference-free faithfulness + context precision
  → Directional regression check in CI

Phase 1b: Labeled Evaluation Baseline (after #307 pilot)
  → Golden dataset from client data
  → nDCG@5, MRR, Precision@5 — authoritative quality gate

Phase 2: Hybrid Search + Service Extraction
  → Qdrant native BM25 + RRF fusion ($0 infra)
  → Extract knowledge-api from bot-config-api
  → MUST pass labeled nDCG@5 ≥ 10% improvement gate

Phase 3+: CONDITIONAL — evidence-driven only
  → Contextual Retrieval, reranker, semantic chunking — all candidates
  → Each requires Agentopia-specific eval data before commitment

DEFERRED: Graph RAG, Agentic RAG, auto-ingestion, multilingual
```

---

> All claims are linked inline to sources. Evidence is separated into repo-proven (Sections A-E, J) and external directional (Section H).

*This debate document is READY_FOR_CTO_REVIEW.*
