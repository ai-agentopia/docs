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

### Phase 2a: Hybrid Retrieval (within current boundary)

**Why within current boundary**: If retrieval quality or latency changes, we know it's the retrieval logic, not topology.

| Task | Description |
|---|---|
| **R2a.1** Enable Qdrant native BM25 | Sparse vector config + text tokenization per collection |
| **R2a.2** Implement hybrid search | Vector + BM25 parallel query via Qdrant prefetch API |
| **R2a.3** Implement RRF fusion | `score = 1/(k + rank_vector) + 1/(k + rank_text)` |
| **R2a.4** Collection migration | Re-create collections with BM25 support (migration script) |
| **R2a.5** Observability metrics | Prometheus counters/histograms for retrieval stack |
| **R2a.6** **Labeled eval gate** | Require nDCG@5 ≥ 10% improvement over Phase 1b baseline. **Must use labeled metrics, not RAGAS.** |

**Cost**: $0 — Qdrant native features. Code changes only.

### Phase 2b: Knowledge-API Extraction (separate track, after 2a stabilizes)

**Prerequisite**: Phase 2a hybrid search is stable and baselined.

| Task | Description |
|---|---|
| **S2b.1** Create knowledge-api service | New FastAPI service, Dockerfile, Helm chart, ArgoCD app |
| **S2b.2** Implement binding sync endpoint | `POST /internal/bindings/sync` — bot-config-api notifies on deploy/PATCH |
| **S2b.3** Implement auth layer | Operator session + bot bearer token verification (see Section J3) |
| **S2b.4** Deploy alongside bot-config-api | bot-config-api proxies `/api/v1/knowledge/*` to knowledge-api |
| **S2b.5** Switch gateway plugin target | Update Helm `apiUrl` to knowledge-api directly |
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
| Knowledge service extraction | **RECOMMENDED — separate track from retrieval** | Repo: coupled to bot deploy flow. See Section J |

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
- Search: vector search, hybrid/BM25, RRF fusion, token budget
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
                                        knowledge-api self-heals on restart via rebuild_from_k8s()
```

**Auth after extraction**:

| Auth path | Termination | How |
|---|---|---|
| **Operator session** | knowledge-api owns its own session middleware (same ADMIN_PASSWORD env var) | Independent — no proxy needed |
| **Bot bearer** | knowledge-api verifies relay token directly | Option A: read K8s Secret (needs RBAC). Option B: bot-config-api passes token hash during binding sync |
| **Gateway plugin** | Calls knowledge-api directly (Helm `apiUrl` updated) | Clean — URL-agnostic contract |
| **UI** | Calls knowledge-api directly for scope/doc/search | Clean — same REST contract |

**Rollback path**:
1. Revert gateway plugin `apiUrl` in Helm to point back to bot-config-api
2. ArgoCD selfHeal redeploys gateway pods with old target
3. Knowledge-api pods can remain running (no harm) or be scaled to 0
4. Bot-config-api still has all knowledge code (removed only after extraction proven stable)

**Compatibility during migration**:
- Both services run simultaneously
- bot-config-api proxies `/api/v1/knowledge/*` to knowledge-api (reverse proxy)
- Clients (gateway, UI) continue calling bot-config-api URL
- Once stable, update Helm `apiUrl` to knowledge-api directly
- Remove proxy + knowledge code from bot-config-api

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
  Phase 2a → Hybrid search within current boundary (Qdrant native BM25 + RRF)
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
2. Gateway plugin calls knowledge-api directly (Helm apiUrl updated)
3. All existing knowledge tests pass against knowledge-api
4. Retrieval latency p95 does not increase >200ms vs Phase 2a baseline
5. Rollback to bot-config-api proxy is tested and documented

---

> All claims are linked inline to sources. Evidence is separated into repo-proven (Sections A-E, J) and external directional (Section H).

*This debate document is READY_FOR_CTO_REVIEW.*
