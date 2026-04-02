---
title: "Super RAG Debate — CTO Architecture Decision Session"
description: "Formal debate on production-ready Super RAG for Agentopia. Grounded in current implementation, not theory."
---

# Super RAG — CTO Debate Session

> **Date**: 2026-03-31 (revised after CTO pushback)
> **Participants**: CTO, Platform/DevOps
> **Status**: READY_FOR_CTO_REVIEW
> **Assumption**: English-first production. Vietnamese/multilingual is optional enhancement.
> **Revision**: v6 — adds Section L (always-updated planning knowledge debate) addressing code/feature/freshness requirements

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
| **Vector search** | Qdrant v1.16.1, `qwen/qwen3-embedding-8b` (4096d), cosine similarity, topK=5 | **Operationally present** — works, but no retry/circuit breaker on embedding API |
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
Qdrant sparse search (TF, top-20)         ← NEW: sparse term-frequency vectors
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
| **F0.5** Expose vector dimension as env var | Replace hardcoded dimension | Model change requires code change |
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

### Phase 2a: Hybrid Retrieval (within current boundary)

**Why within current boundary**: If retrieval quality or latency changes, we know it's the retrieval logic, not topology.

| Task | Description |
|---|---|
| **R2a.1** Enable Qdrant sparse vectors | Sparse TF vector config + text tokenization per collection (NOTE: TF-only, not full BM25 — no IDF component. Upgrade path documented.) |
| **R2a.2** Implement hybrid search | Dense + sparse TF parallel query via Qdrant prefetch API + RRF fusion |
| **R2a.3** Implement RRF fusion | `score = 1/(k + rank_vector) + 1/(k + rank_text)` |
| **R2a.4** Collection migration | Re-create collections with sparse vector support (migration script) |
| **R2a.5** Observability metrics | Prometheus counters/histograms for retrieval stack |
| **R2a.6** **Labeled eval gate** | Require nDCG@5 ≥ 10% improvement over Phase 1b baseline. **Must use labeled metrics, not RAGAS.** |

**Cost**: $0 — Qdrant native features. Code changes only.

### Phase 2b: Knowledge-API Extraction (separate track, after 2a stabilizes)

**Prerequisite**: Phase 2a hybrid search is stable and baselined.

**Locked constraint**: knowledge-api is a **new service inside the existing `agentopia-protocol` monorepo** — NOT a new repository. New app directory, Dockerfile, Helm chart, and ArgoCD app within the monorepo. Repo extraction only reconsidered if later evidence shows independent release cadence, distinct team ownership, low shared-code coupling, or monorepo CI/CD pain materially blocking delivery.

| Task | Description |
|---|---|
| **S2b.1** Create knowledge-api service | New app directory in `agentopia-protocol`, FastAPI service, Dockerfile, Helm chart, ArgoCD app |
| **S2b.2** Implement binding sync + reconcile | `POST /internal/bindings/sync` webhook, cache-miss CRD fallback, periodic 5min reconcile |
| **S2b.3** Implement auth layer | Bot bearer via K8s Secret read (RBAC scoped to `agentopia-gateway-env-*`) + internal service token for proxied operator requests. NO independent session middleware. (see Section J3) |
| **S2b.4** Deploy alongside bot-config-api (proxy model) | bot-config-api terminates operator auth, proxies `/api/v1/knowledge/*` to knowledge-api with `X-Internal-Service` + `INTERNAL_SERVICE_TOKEN` |
| **S2b.5** Switch gateway plugin target | Update Helm `apiUrl` to knowledge-api directly (bot bearer auth, no session) |
| **S2b.6** Remove knowledge code from bot-config-api | After stable operation period |
| **S2b.7** **Topology gate** | Retrieval latency p95 must not increase >200ms vs Phase 2a baseline. Rollback tested. |

**Why separate from 2a**: Cannot diagnose regression root cause if retrieval logic and service topology change simultaneously.

### Phase 3+: Conditional Experiments (only if Phase 1/2 data justifies)

**No default recommendation.** All options are candidates, not commitments:

| Candidate | Trigger Condition | External Evidence |
|---|---|---|
| **Contextual Retrieval** | Phase 2a eval shows ingestion-side chunk quality limits recall | [Anthropic: -49% failure rate](https://www.anthropic.com/news/contextual-retrieval), [$1.02/1M tokens one-time](https://www.anthropic.com/news/contextual-retrieval) |
| **Semantic chunking** | Phase 1 eval shows fixed-size chunking breaks semantic boundaries | Defined in code enum, not yet implemented |
| **Reranker** | Phase 2a labeled nDCG@5 plateaus below target | [+28% nDCG@10 possible](https://www.zeroentropy.dev/articles/ultimate-guide-to-choosing-the-best-reranking-model-in-2025) |
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

### 3. Approve Qdrant native hybrid as Phase 2a?

**YES.** Zero infrastructure cost. Uses Qdrant features already deployed but unused. Eval gate uses **labeled nDCG@5**, not RAGAS. Service extraction (Phase 2b) follows after hybrid baseline stabilizes — **not bundled**.

### 4. Approve Graph RAG deferral?

**YES.** No client demand. High complexity. No quality metrics to measure impact yet.

### 5. Minimum production-ready eval gate?

**Phase 1a** (RAGAS directional signal) + **#307 manual pilot** (CTO sign-off) for initial production.
**Phase 1b** (labeled nDCG@5 baseline) required before Phase 2 claims can be validated.

### 6. Required vs optional scope?

| Level | Items |
|---|---|
| **REQUIRED for production** | Phase 0, #307 completion, RAGAS early signal, basic observability |
| **REQUIRED for quality claims** | Phase 1b labeled baseline, Phase 2a hybrid with labeled eval gate |
| **CONDITIONAL** | All Phase 3+ candidates — evidence-driven only |
| **DEFERRED** | Graph RAG, Agentic RAG, auto-ingestion, multilingual |

---

## G. Decision Summary

| Decision | Verdict | Evidence Type |
|---|---|---|
| Phase 0 before evaluation | **RECOMMENDED** | Repo: known fragility in code |
| RAGAS as early signal | **RECOMMENDED** | External: [RAGAS framework](https://docs.ragas.io/en/stable/concepts/metrics/). Limitation: not a substitute for labeled metrics |
| Labeled nDCG/MRR as authoritative gate | **RECOMMENDED** | Industry standard: [nDCG correlates with RAG quality](https://labelyourdata.com/articles/llm-fine-tuning/rag-evaluation) |
| Hybrid search via Qdrant native (Phase 2a) | **RECOMMENDED** | Repo: Qdrant v1.16.1 deployed. External: [native BM25 since v1.15](https://qdrant.tech/articles/sparse-vectors/) |
| Knowledge-api extraction (Phase 2b) | **RECOMMENDED — separate track** | Repo: coupled but extractable. After 2a stabilizes. See Section J |
| Reranker | **DEFERRED** | Conditional on Phase 2a labeled eval |
| Graph RAG | **DEFERRED** | No demand, high complexity |
| Contextual Retrieval | **CONDITIONAL candidate** | External: [Anthropic benchmarks](https://www.anthropic.com/news/contextual-retrieval). Needs Agentopia-specific validation |
| Semantic chunking | **CONDITIONAL candidate** | Repo: enum defined, not implemented. Needs eval data |
| External search engine | **REJECTED** | Qdrant native sufficient. <10K docs year 1 |
| Knowledge service extraction | **RECOMMENDED — separate track from retrieval** | Repo: coupled to bot deploy flow. Proxy-first auth, K8s Secret bearer verification, 3-strategy reconcile. See Section J |

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

## J. Service Architecture Assessment — Knowledge-API Extraction

> **Client requirement**: RAG components should be extractable to independent services, not locked into a monolithic bot-config-api.
>
> **Locked constraint**: knowledge-api is a new service inside the `agentopia-protocol` monorepo. Service extraction only (independent deploy/scale boundary). Repo extraction deferred — reconsidered only with evidence of independent release cadence, distinct team ownership, or monorepo CI/CD pain.

### J1. Current State (repo-proven)

All knowledge/RAG logic lives inside **`bot-config-api`** (~1,700 lines):

| Module | Location | Lines | Responsibility |
|---|---|---|---|
| `knowledge.py` | `services/` | ~780 | KnowledgeService, QdrantBackend, chunking, embedding, search |
| `knowledge router` | `routers/` | ~350 | REST API: ingest, search, delete, lifecycle |
| `document_store.py` | `services/` | ~270 | PostgresDocumentStore, lifecycle management |
| `bot_knowledge_index.py` | `services/` | ~135 | Bot→scope binding resolution |
| `knowledge models` | `models/` | ~170 | Pydantic schemas |

`bot-config-api` also owns: bot CRUD/deploy, SOUL/USER generation, A2A protocol, Temporal workflow integration, usage metrics.

### J2. Coupling Analysis — Honest Assessment

Knowledge is a **strong candidate for extraction, but not a trivial decoupling**. It remains coupled to:

#### Coupling 1: Bot→Scope Binding Lifecycle

**Source of truth**: K8s Application CRD annotations.

```
Application CRD "agentopia-{bot}" (argocd namespace)
  metadata.annotations:
    agentopia/client-id: "{client_id}"
    agentopia/knowledge-scopes: '["{scope1}", "{scope2}"]'
```

**Who writes**: `bot-config-api` deploy router — `persist_to_k8s()` patches annotation during bot deploy/PATCH.
**Who reads**: `BotKnowledgeIndex` singleton — `rebuild_from_k8s()` on startup, `resolve()` on every search.
**Runtime cache**: In-memory dict `bot_name → BotKnowledgeBinding`, O(1) lookup on hot path.

**Problem for extraction**: If knowledge-api is separate, who owns this binding?
- Deploy endpoint (bot-config-api) creates the CRD and writes the annotation
- Knowledge search endpoint (knowledge-api) needs to resolve bot→scopes
- Two services reading/writing the same CRD annotation = ownership conflict

#### Coupling 2: Auth/Token Verification

**Bot bearer auth** on knowledge search:
1. Gateway sends `Authorization: Bearer {AGENTOPIA_RELAY_TOKEN}` + `X-Bot-Name` header
2. Knowledge router calls `_verify_bot_bearer()` → reads K8s Secret `agentopia-gateway-env-{bot}`
3. Time-constant comparison of token values

**Operator auth**: Cookie session middleware on `bot-config-api`.

**Problem for extraction**: Knowledge-api must verify bot relay tokens. Today this reads K8s Secrets created by `bot-config-api` during deploy. Separate service needs either:
- Direct K8s Secret read access (elevated permissions)
- A shared token store (Redis/Postgres)
- A verification endpoint on bot-config-api (HTTP round-trip on hot path)

#### Coupling 3: Helm Value Patching

When knowledge binding changes (PATCH endpoint), `bot-config-api` calls `K8sService.patch_bot_knowledge_values(enabled=True/False)` → patches Application CRD Helm values → ArgoCD selfHeal triggers pod rollout with plugin enabled/disabled.

**Problem for extraction**: Knowledge-api would need K8s API access to patch Application CRDs, or delegate back to bot-config-api.

#### Coupling 4: Gateway Plugin Contract

Plugin calls `GET /api/v1/knowledge/search` with headers:
- `Authorization: Bearer {AGENTOPIA_RELAY_TOKEN}`
- `X-Bot-Name: {bot_name}`

Response: `{ results: SearchResult[], count: number }`

**This contract is URL-agnostic** — plugin's `apiUrl` is configurable via Helm. Changing the target from `bot-config-api` to `knowledge-api` only requires updating the Helm value. **This coupling is clean.**

### J3. Recommended Ownership Model After Extraction

**Bot-config-api retains**:
- CRD creation/deletion (bot deploy/delete lifecycle)
- Knowledge binding writes: `persist_to_k8s()` on deploy/PATCH → writes annotation
- Helm value patching: `knowledgeRetrieval.enabled` toggle
- Relay token creation (K8s Secret management)
- Notify knowledge-api of binding changes via internal webhook

**Knowledge-api owns**:
- Ingestion: chunking, embedding, Qdrant write, document lifecycle
- Search: vector search, hybrid (dense + sparse TF + RRF fusion), token budget
- Scope resolution: maintains its own in-memory index, synced from bot-config-api
- Auth: verifies bot bearer tokens (own verification, not delegating to bot-config-api)
- Observability: retrieval-specific Prometheus metrics

**Sync mechanism for bot→scope bindings**:

```
Bot deploy/PATCH (bot-config-api)
  ├── Write annotation to CRD (existing)
  └── POST knowledge-api /internal/bindings/sync
        body: { bot: "max-sa", client_id: "c1", scopes: ["docs", "api"] }
        ← knowledge-api updates its in-memory index
        ← if knowledge-api unavailable: deploy succeeds with warning
                                        reconcile handles eventual consistency (see below)
```

**Required reconcile strategies for Phase 2b** (must be implemented during knowledge-api extraction):

1. **Cache-miss CRD fallback** (proposed) — when `resolve(bot_name)` returns empty (cache miss), knowledge-api must perform a live CRD annotation lookup before returning "no scopes". This closes the window for newly deployed bots whose sync webhook was missed. Cost: one K8s API call per cache miss (rare on hot path).

2. **Periodic background reconciliation** (proposed) — background task must run `rebuild_from_k8s()` every 5 minutes (configurable via `BINDING_RECONCILE_INTERVAL_SECONDS`). Catches any drift from missed webhooks, manual CRD edits, or partial failures. Bounded staleness: max 5 minutes.

3. **Startup full rebuild** (existing pattern) — `rebuild_from_k8s()` on service start. Already implemented in `bot_knowledge_index.py` within bot-config-api. Must be carried over to knowledge-api.

**Expected stale binding window**: With cache-miss fallback, a newly deployed bot's first search triggers a live lookup — **zero stale window for new bindings**. For binding updates (scope change on existing bot), periodic reconcile bounds staleness to the reconcile interval. Acceptable for scope changes which are operator-initiated and infrequent.

**Auth after extraction**:

> **Critical constraint**: Operator sessions use in-memory `SessionStore` (Python dict, single-replica). A cookie issued by `bot-config-api` is meaningless to `knowledge-api` — there is no shared session backend. "Same ADMIN_PASSWORD" allows independent login but does NOT create shared sessions.

**Migration phase (proxy model — recommended)**:

| Auth path | Termination | How |
|---|---|---|
| **Operator session** | **bot-config-api terminates auth** (existing session middleware) | UI continues calling bot-config-api; bot-config-api proxies `/api/v1/knowledge/*` to knowledge-api with internal service header |
| **Bot bearer** | knowledge-api verifies relay token directly via K8s Secret read | Reads `agentopia-gateway-env-{bot}` Secret. Requires RBAC `get` on Secrets in `agentopia` namespace. Same pattern as current `_verify_bot_bearer()` in bot-config-api. |
| **Gateway plugin** | Calls knowledge-api directly (Helm `apiUrl` updated) | Clean — URL-agnostic contract. Bot bearer auth, no session cookie. |
| **UI** | Calls **bot-config-api** (proxied to knowledge-api) | Operator session stays on bot-config-api; knowledge-api receives pre-authenticated requests via internal header |

**Bot bearer verification decision**:
- **Recommended**: Direct K8s Secret read. knowledge-api reads `agentopia-gateway-env-{bot}` to verify relay tokens, identical to the current `_verify_bot_bearer()` implementation in bot-config-api.
- **Why preferred**: Zero runtime dependency on bot-config-api for the gateway hot path. No additional network hop. Same code pattern — lift and shift from bot-config-api. K8s RBAC (`get` on Secrets) is a standard, auditable permission.
- **Tradeoff**: knowledge-api ServiceAccount needs RBAC read access to bot Secrets. This is a narrow permission (read-only, scoped to `agentopia-gateway-env-*` naming convention), not a broad cluster privilege.
- **Alternative rejected**: Passing token hash during binding sync was considered. Rejected because it couples auth verification to the sync path — a missed sync webhook would leave knowledge-api unable to verify tokens for that bot until the next reconcile cycle. Direct Secret read is always current.

**Internal service auth for proxied requests**: bot-config-api adds `X-Internal-Service: bot-config-api` + shared `INTERNAL_SERVICE_TOKEN` header. knowledge-api trusts requests with valid internal token — no session cookie needed on the knowledge-api side.

**Post-migration option (if services fully decoupled later)**:

Migrate session store from in-memory to shared backend (Redis or Postgres). Both services validate the same session state. This is already planned as P2 in the current `session.py` (`"MVP1: in-memory store. Single replica required. P2: migrate to Redis/Postgres."`). Only pursue this when multi-replica or independent UI access to knowledge-api is required.

**Why proxy-first**: No new infrastructure (Redis), no new auth protocol. Uses existing session middleware in bot-config-api. knowledge-api only needs to verify two auth types: bot bearer (existing) and internal service token (simple shared secret). Rollback = remove proxy, knowledge code still in bot-config-api.

**Rollback path**:
1. Revert gateway plugin `apiUrl` in Helm to point back to bot-config-api
2. ArgoCD selfHeal redeploys gateway pods with old target
3. Knowledge-api pods can remain running (no harm) or be scaled to 0
4. Bot-config-api still has all knowledge code (removed only after extraction proven stable)

**Compatibility during migration**:
- Both services run simultaneously
- bot-config-api terminates operator session auth, then proxies `/api/v1/knowledge/*` to knowledge-api with internal service token
- **UI** continues calling bot-config-api URL — operator sessions work unchanged
- **Gateway plugin** can switch to knowledge-api directly via Helm `apiUrl` (uses bot bearer, not session)
- Once stable: UI migration requires either shared session store (P2) or UI repointing to knowledge-api with its own login
- Remove proxy + knowledge code from bot-config-api only after all clients migrated

### J4. Why Not Extract Now (Phase 0-1)

1. **Phase 0-1 must stabilize retrieval quality first** — extracting during foundation hardening creates two moving targets
2. **Hybrid search (Phase 2a) should be proven within current boundary** — if we change topology and retrieval logic simultaneously, regression source is ambiguous
3. **Service extraction is an infrastructure change** — requires new Dockerfile, Helm chart, ArgoCD app, CI pipeline, health checks. This is a separate workstream from retrieval quality.

---

## K. Final Recommended Production Path

### Two Parallel Tracks

```
TRACK A: Retrieval Quality (sequential phases)
  Phase 0  → Foundation hardening (code-only, 1-2 PRs)
  Phase 1a → RAGAS early signal (immediate)
  Phase 1b → Labeled eval baseline (after #307)
  Phase 2a → Hybrid search within current boundary (dense + sparse TF + RRF)
           → MUST pass labeled nDCG@5 ≥ 10% improvement gate
  Phase 3+ → Conditional experiments (evidence-driven)

TRACK B: Service Architecture (independent track)
  Phase 2b → knowledge-api extraction
           → Starts AFTER Phase 2a hybrid search is stable and baselined
           → Does NOT change retrieval logic — only topology
           → Must preserve: operator auth, bot bearer auth, gateway plugin contract, UI contract
           → Rollback: revert Helm apiUrl to bot-config-api
```

**Why separated**: If Phase 2a hybrid search causes quality/latency regression, we know it's retrieval logic. If Phase 2b extraction causes issues, we know it's topology. Cannot diagnose root cause if both change simultaneously.

**Phase 2b gate**: Extraction is complete when:
1. Knowledge-api runs independently with own health check
2. Gateway plugin calls knowledge-api directly (Helm apiUrl updated, bot bearer auth)
3. UI access works via bot-config-api proxy (operator session terminated at proxy, internal token forwarded)
4. Binding sync webhook works + cache-miss fallback verified + periodic reconcile running
5. All existing knowledge tests pass against knowledge-api
6. Retrieval latency p95 does not increase >200ms vs Phase 2a baseline
7. Rollback to bot-config-api proxy is tested and documented

---

## L. Always-Updated Planning Knowledge — Re-Debate

> **Trigger**: Client requirement refinement. SA bots must use always-updated knowledge for planning, solution design, and workflow generation. Knowledge sources include business documents, code repositories, and feature artifacts — not only uploaded files.

### L1. Current State (repo-verified)

| Component | Reality | Evidence |
|---|---|---|
| **Planner knowledge input** | **ZERO** — planner receives only `objective_text`, `workflow_id`, `owner`, `repo`. No knowledge search, no codebase context, no document context. | `planner_graph.py`: `ObjectivePlanInput` has no knowledge fields. `plan_delivery` activity reads objective text only. |
| **Knowledge ingestion** | **Manual file upload only** — `POST /{scope}/ingest` with multipart file. Webhook endpoint retired (410 Gone). No git sync, no API ingest. | `routers/knowledge.py`: single ingest endpoint. Webhook returns 410. |
| **Source type awareness** | **None** — all chunks treated identically (text + embedding + metadata). No distinction between business doc, code file, feature spec. | `knowledge.py`: `DocumentFormat` enum has CODE but chunking strategy is not selected by source type. |
| **Freshness tracking** | **Static** — `ingested_at` timestamp only. No commit SHA, no branch reference, no staleness detection. | `document_records` table: `ingested_at`, `document_hash`, `status`. No git metadata. |
| **Code awareness at chat time** | **MCP tools only** — bots can read files via GitHub MCP during conversation. Not persisted to knowledge. | `gateway/mcp-servers/github/`: standard MCP server, not connected to knowledge service. |
| **Dead code hint** | `RepoIndexConfig` model exists in `models/knowledge.py` but is never instantiated anywhere. | Dead artifact from planned feature #24. |

**Bottom line**: The planner is knowledge-blind. Knowledge only flows into chat (gateway plugin injection). Planning and workflow generation operate on raw objective text alone.

### L2. Requirement Analysis — What Actually Needs to Change

The client requirement has three distinct dimensions. They have **different implementation complexity, different value, and different prerequisites**:

| Dimension | What | Complexity | Prerequisites |
|---|---|---|---|
| **Business doc knowledge for planning** | Planner queries existing knowledge scopes before decomposing objective | Medium | Phase 0 (config fix) + working knowledge scopes |
| **Code repository knowledge** | Git-based ingestion: clone → index → incremental re-index on push | High | Knowledge service must handle large volumes + freshness tracking |
| **Feature artifact knowledge** | GitHub issues, PRs, specs → knowledge | Very High | GitHub webhook pipeline, volatile data, state tracking |

### L3. Pushback: Scope Discipline Required

**Concern**: Adding code ingestion, feature sync, and planner integration to Super RAG triples the scope of milestone #34. The approved plan (Phase 0→1→2a→2b) addresses **retrieval quality and service architecture** — necessary prerequisites for any of the new capabilities.

**Argument**:

1. **You cannot evaluate code knowledge quality without evaluation infrastructure.** If we ingest a 10K-file repo and the planner uses it, how do we know the retrieved code chunks are relevant? Phase 1a/1b (evaluation) must come first.

2. **You cannot build reliable code ingestion inside the current monolithic knowledge service.** Code ingestion is a background pipeline (clone → diff → chunk → embed → store). This is exactly the kind of workload that justifies knowledge-api extraction (Phase 2b). Building it inside bot-config-api creates more technical debt to extract later.

3. **Hybrid search is essential for code.** Developers search code by exact identifiers (function names, error codes, config keys). Pure vector search fails here. Phase 2a (sparse TF + RRF hybrid) directly enables better code retrieval.

4. **The planner consuming knowledge is a planner feature, not a RAG feature.** The planner today is a minimal stub (`planner_graph.py` returns hardcoded templates). Making it knowledge-aware requires planner evolution regardless of RAG quality.

**Recommendation**: Phase 0→2b remains the foundation. New requirements become **Phase 3 committed work** (not conditional), sequenced AFTER the foundation is proven. This is not deferral — it is correct ordering.

### L4. Knowledge Source Model — Recommended

#### Source Type 1: Business Documents

| Attribute | Recommendation |
|---|---|
| **Source of truth** | Uploaded files (PDF, HTML, Markdown, text, code) — operator-managed |
| **What to ingest** | Client-provided domain docs, architecture specs, API references, project standards |
| **What NOT to ingest** | Credentials, PII, internal communications, draft documents without explicit approval |
| **Scope identity** | `{client_id}/{scope_name}` — existing model, unchanged |
| **Update model** | Manual upload (existing). Source-based sync (e.g., Confluence/GDrive webhook) is **deferred** — no client has requested it |
| **Freshness** | `ingested_at` timestamp + `document_hash`. Operator controls when to re-upload. Acceptable for business docs which change infrequently. |

**Verdict**: Current model is sufficient for business docs. No change needed in Phase 0-2b.

#### Source Type 2: Code Repositories

| Attribute | Recommendation |
|---|---|
| **Source of truth** | Git repository — specific branch (default: main/default branch) |
| **What to ingest** | Source code files (.py, .ts, .js, .go, .rs, .java, etc.), README, doc files within repo, configuration files |
| **What NOT to ingest** | Binary files, node_modules, build artifacts, .git directory, files matching .gitignore, secrets/env files |
| **Scope identity** | `{client_id}/repo:{owner}/{repo}` — new scope convention, prefixed with `repo:` to distinguish from business doc scopes |
| **Update model** | Git webhook (push event on tracked branch) → incremental re-index (diff against last indexed commit SHA). Fallback: scheduled full reconcile every N hours. |
| **Freshness** | Per-document: `commit_sha`, `branch`, `indexed_at`. Per-scope: `last_indexed_sha`, `last_indexed_at`. |

**Verdict**: Committed as Phase 3a. Requires Phase 2a (hybrid search for code identifiers) and Phase 2b (knowledge-api owns the pipeline independently).

#### Source Type 3: Feature Artifacts (Issues, PRs, Specs)

| Attribute | Recommendation |
|---|---|
| **Source of truth** | GitHub API — issues, pull requests, project specs |
| **What to ingest** | Issue title + body + labels, PR title + body + diff summary, milestone descriptions |
| **What NOT to ingest** | Comment threads (too volatile, too noisy), review comments (handled by governed PR review), CI logs |
| **Scope identity** | `{client_id}/features:{owner}/{repo}` — prefixed with `features:` |
| **Update model** | GitHub webhook (issues/PR events) → upsert. Issue closed → mark superseded. |
| **Freshness** | `github_updated_at`, `issue_state` (open/closed), `pr_state` (open/merged/closed). |

**Verdict**: **Conditional, not committed.** Pushback rationale:
1. Feature artifacts are the most volatile knowledge source — issues change state constantly, PR descriptions are updated, labels shift
2. The signal-to-noise ratio is low — most issue content is conversation, not structured knowledge
3. Code knowledge (Source Type 2) provides more actionable planning context than issue metadata
4. **Reconsider when**: Phase 3a (code knowledge) is stable AND planner demonstrates it needs project management context beyond what's in SOUL.md and objective text

### L5. Update / Sync Model — Recommended

| Source Type | Primary Sync | Fallback | Staleness Bound |
|---|---|---|---|
| **Business docs** | Manual upload (existing) | None needed — operator-controlled | Operator responsibility |
| **Code repos** | GitHub webhook on push → incremental re-index | Scheduled full reconcile every 6h | Max 6h (scheduled) or near-real-time (webhook) |
| **Feature artifacts** | GitHub webhook on issue/PR events | Scheduled reconcile every 1h | Max 1h |

**Incremental re-index strategy for code**:

```
Webhook payload (push event)
  ├── Extract: commits[].added, commits[].modified, commits[].removed
  ├── Clone: sparse checkout of changed files only (NOT full clone)
  ├── Diff: compare file hashes against last indexed state
  │   ├── New file → ingest (code-aware chunking)
  │   ├── Modified file → replace (two-phase atomic, existing pattern)
  │   └── Deleted file → tombstone (existing lifecycle)
  ├── Update scope metadata: last_indexed_sha = head_sha
  └── Emit: knowledge_repo_sync_files_total (Prometheus counter)
```

**Same-hash optimization**: Existing `document_hash` dedup applies — unchanged files are skipped automatically. This is already production-grade in the ingestion pipeline.

### L6. Planning Consumption Model — Recommended

**Current gap**: `invoke_planner()` receives only `objective_text`. Must receive knowledge context.

**Proposed flow**:

```
start_delivery(objective, owner, repo)
  ↓
plan_delivery activity (Temporal)
  ↓
[NEW] Resolve planning scopes:
  ├── Business doc scopes: from bot's knowledge binding (existing)
  ├── Code scope: auto-resolve "{client_id}/repo:{owner}/{repo}" (if indexed)
  └── Feature scope: auto-resolve "{client_id}/features:{owner}/{repo}" (if indexed)
  ↓
[NEW] Knowledge-grounded context retrieval:
  ├── Search business docs: "What does client say about {objective topic}?"
  ├── Search code: "What existing code is related to {objective}?"
  ├── Search features: "What issues/specs relate to {objective}?" (if available)
  └── Compose: grounding_context = formatted results with citations
  ↓
[MODIFIED] invoke_planner(objective_text, grounding_context, owner, repo)
  ↓
LLM decomposes objective using:
  ├── Objective text (what to build)
  ├── Business doc context (domain constraints, standards)
  ├── Code context (existing implementation, architecture)
  └── Feature context (related issues/specs, if available)
  ↓
WorkPacket with grounded:
  ├── in_scope: informed by actual codebase structure
  ├── acceptance_criteria: informed by project standards
  └── metadata.knowledge_sources: citation trail for audit
```

**Freshness in planning**:
- Each knowledge result carries `ingested_at` (business docs) or `commit_sha` + `indexed_at` (code)
- Planner prompt includes: `"Code context is from commit {sha} indexed at {date}. Business docs last updated {date}."`
- If code scope `last_indexed_at` is >24h stale, planner prompt includes warning: `"⚠ Code knowledge may be outdated — last indexed {N}h ago."`
- Planning does NOT block on fresh knowledge — uses best available, flags staleness

**Scope auto-resolution**:
- When `start_delivery` is called with `owner/repo`, the system auto-resolves `{client_id}/repo:{owner}/{repo}` scope
- If scope doesn't exist (repo not indexed), planning proceeds without code context — graceful degradation, same pattern as gateway plugin timeout
- No manual scope binding required for code — repo identity IS the scope identity

### L7. Metadata / Provenance Model for Evolving Knowledge

**Extended metadata per document**:

| Field | Source Type | Purpose |
|---|---|---|
| `source_type` | All | `business_doc`, `code_file`, `feature_artifact` — NEW, required |
| `ingested_at` | All | When this version was indexed — existing |
| `document_hash` | All | Content dedup — existing |
| `commit_sha` | Code | Git commit that produced this version — NEW |
| `branch` | Code | Tracked branch — NEW |
| `file_path` | Code | Full path within repo — NEW (currently `source` field, reuse) |
| `language` | Code | Programming language — NEW |
| `github_updated_at` | Feature | Last GitHub API update timestamp — NEW |
| `issue_state` | Feature | open/closed — NEW |
| `pr_state` | Feature | open/merged/closed — NEW |
| `approval_status` | Business doc | draft/approved — NEW, optional |
| `freshness_class` | All (computed) | `current` / `stale` / `unknown` based on source-specific rules — NEW |

**Freshness rules**:
- Business doc: always `current` (operator-managed, assumed intentional)
- Code: `current` if `indexed_at` < 24h AND `commit_sha` matches remote HEAD. `stale` otherwise.
- Feature: `current` if `github_updated_at` < 1h. `stale` otherwise.

**Schema impact**: Requires `ALTER TABLE document_records ADD COLUMN source_type, commit_sha, branch, ...` migration. Backward-compatible — new columns nullable, existing docs default to `source_type = 'business_doc'`.

### L8. Milestone Impact Assessment

#### What remains valid (unchanged)

| Phase | Status | Why unchanged |
|---|---|---|
| Phase 0: Foundation Hardening | Valid | Prerequisites for everything — config, retry, health. Must complete first. |
| Phase 1a: RAGAS Early Signal | Valid | Evaluation infrastructure needed before measuring code retrieval quality. |
| Phase 1b: Labeled Baseline | Valid | Authoritative quality gate. Extend golden dataset to include code queries in Phase 3a. |
| Phase 2a: Hybrid Retrieval | Valid | **More important now** — code search relies heavily on keyword matching (function names, imports). Sparse keyword retrieval is essential. |
| Phase 2b: Knowledge-API Extraction | Valid | Code ingestion pipeline should live in knowledge-api, not bot-config-api. Extract first, then build pipeline. |

#### What must be added (new committed work)

**Phase 3a: Code Repository Knowledge** (NEW — committed, not conditional)

| Task | Description |
|---|---|
| **C3a.1** Implement `source_type` metadata | Add `source_type`, `commit_sha`, `branch`, `file_path`, `language` columns. Migration script. |
| **C3a.2** Implement repo ingestion endpoint | `POST /api/v1/knowledge/{scope}/ingest-repo` — accepts `RepoIndexConfig` (owner, repo, branch, file patterns) |
| **C3a.3** Implement git clone + code-aware chunking | Sparse clone → file scan → code-aware chunking (already exists but unused) → embed → store |
| **C3a.4** Implement GitHub webhook handler | Receive push events → incremental re-index (add/modify/remove changed files) |
| **C3a.5** Implement scheduled reconcile | Full re-index every 6h (configurable). Catches missed webhooks, force pushes, branch changes. |
| **C3a.6** Implement freshness tracking | `last_indexed_sha` per scope. Freshness class computation. Stale warning in retrieval results. |
| **C3a.7** Extend labeled eval | Add code-specific queries to golden dataset. Measure code retrieval nDCG@5. |
| **C3a.8** **Code retrieval gate** | Code-specific nDCG@5 must meet threshold (defined after Phase 1b baseline extended with code queries). |

**Prerequisites**: Phase 2a (hybrid search — essential for code) + Phase 2b (knowledge-api — pipeline ownership).

**Phase 3b: Knowledge-Grounded Planning** (NEW — committed, not conditional)

| Task | Description |
|---|---|
| **P3b.1** Extend `ObjectivePlanInput` | Add `grounding_context: list[KnowledgeResult]` field |
| **P3b.2** Implement pre-planning knowledge retrieval | In `plan_delivery` activity: resolve scopes → search business + code knowledge → compose context |
| **P3b.3** Scope auto-resolution for repos | `start_delivery(owner, repo)` auto-resolves `{client_id}/repo:{owner}/{repo}` scope |
| **P3b.4** Freshness-aware prompt construction | Include `ingested_at`, `commit_sha` in planner context. Flag stale knowledge (>24h for code). |
| **P3b.5** Graceful degradation | If no indexed scopes exist, planning proceeds without knowledge context (existing behavior, no regression). |
| **P3b.6** WorkPacket citation trail | `metadata.knowledge_sources` lists which documents informed the plan. Audit trail. |
| **P3b.7** **Planning quality gate** | Qualitative: plans produced with knowledge context rated higher than without by operator review (manual gate, not automated). |

**Prerequisites**: Phase 3a (code knowledge available to query).

#### What remains conditional

**Feature artifact knowledge** — NOT committed. Pushback rationale in Section L4 (Source Type 3). Reconsidered when Phase 3a code knowledge is stable AND planner demonstrates it needs project management context.

### L9. Decision Points

**1. Should code knowledge ingestion be committed?**

**YES.** The client requirement is clear — SA planning must use code context. And hybrid search (Phase 2a) directly enables better code retrieval. But it must come AFTER Phase 2a/2b, not before. → **Phase 3a committed.**

**2. Should feature artifacts be committed?**

**NO — conditional.** Pushback: feature artifacts are the most volatile source, lowest signal-to-noise ratio, and highest integration complexity. Code knowledge provides more actionable planning context. Reconsider after Phase 3a proves the code knowledge pipeline. → **Remains conditional.**

**3. Is manual upload acceptable for business docs?**

**YES, for now.** No client has requested Confluence/GDrive auto-sync. Business docs change infrequently. Operator upload with two-phase atomic replace is production-grade. Source-based sync deferred until explicit demand. → **No change.**

**4. Should freshness metadata be first-class?**

**YES.** `source_type` and freshness fields should be designed in Phase 0 (schema migration) even though code ingestion comes later. This avoids a breaking migration later. → **Add `source_type` column to Phase 0 scope (small, backward-compatible).**

**5. Does milestone #34 need new committed issues?**

**YES.** Two new committed issues: Phase 3a (Code Repository Knowledge) and Phase 3b (Knowledge-Grounded Planning). One existing conditional issue updated: Feature Artifacts remains conditional but trigger condition refined.

### L10. Revised Execution Sequence

```
TRACK A: Retrieval Quality (sequential — UNCHANGED)
  Phase 0  → Foundation hardening + source_type metadata column (small addition)
  Phase 1a → RAGAS early signal
  Phase 1b → Labeled eval baseline
  Phase 2a → Hybrid retrieval (sparse TF + RRF) — ESSENTIAL for code search
  Phase 3+ → Conditional experiments

TRACK B: Service Architecture (UNCHANGED)
  Phase 2b → knowledge-api extraction (after 2a stable)

TRACK C: Always-Updated Planning Knowledge (NEW — after Track A Phase 2a + Track B)
  Phase 3a → Code repository knowledge (git sync, webhook, incremental re-index, freshness)
           → Requires: Phase 2a (hybrid search) + Phase 2b (knowledge-api owns pipeline)
           → Gate: code-specific nDCG@5 meets threshold
  Phase 3b → Knowledge-grounded planning (planner consumes knowledge, scope auto-resolution)
           → Requires: Phase 3a (code knowledge available)
           → Gate: operator-rated plan quality improvement
```

**Why Track C comes after A + B**:
1. Hybrid search is essential for code (function names, imports, identifiers need keyword retrieval)
2. Code ingestion pipeline should be built in knowledge-api, not in bot-config-api
3. Evaluation infrastructure must exist to measure code retrieval quality
4. The planner consuming knowledge only works if the knowledge is good

### L11. Provider Scope Boundary

Track C examples (git webhooks, GitHub API, `owner/repo` semantics) assume **Git-based provider flows** as the initial implementation target. This is a pragmatic starting point — Agentopia's current client repos and delivery workflows are GitHub-based.

This is explicitly **NOT a platform-wide provider abstraction decision**. Full provider abstraction (GitHub / Azure DevOps / GitLab / Bitbucket) is a separate platform workstream outside the scope of Super RAG. Super RAG Track C solves code-aware planning knowledge for the current Git/GitHub provider shape only.

**If provider abstraction is later required**: Track C's ingestion pipeline should be refactored behind a provider interface (e.g., `RepoProvider.clone()`, `RepoProvider.webhook_handler()`). That refactor is scoped to the provider abstraction workstream, not to Super RAG.

---

> All claims are linked inline to sources. Evidence is separated into repo-proven (Sections A-E, J, L1) and external directional (Section H).

*This debate document is READY_FOR_CTO_REVIEW.*
