---
title: "Super RAG Debate — CTO Architecture Decision Session"
description: "Formal debate on production-ready Super RAG for Agentopia. Grounded in current implementation, not theory."
---

# Super RAG — CTO Debate Session

> **Date**: 2026-03-31
> **Participants**: CTO, Platform/DevOps
> **Status**: READY_FOR_CTO_REVIEW
> **Assumption**: English-first production. Vietnamese/multilingual is optional enhancement.

---

## A. Current State Assessment

### What Exists Today (Proven)

| Component | Evidence | Production-Ready? |
|---|---|---|
| **Scoped knowledge model** | `{client_id}/{scope_name}`, collection-per-scope in Qdrant, server-side bot→scope resolution | Yes |
| **Authenticated knowledge routes** | Dual-path: operator session + bot bearer. 18 auth tests passing. | Yes |
| **File-upload ingestion** | PDF/HTML/Markdown/code parsing, 3 chunking strategies, SHA-256 dedup, two-phase atomic replace | Yes |
| **Postgres document lifecycle** | `document_records` table: active/superseded/deleted states, tombstone ledger | Yes |
| **Provenance & citations** | Source, section, page, chunk_index, score, ingested_at, document_hash — 35 provenance tests | Yes |
| **Runtime retrieval plugin** | Gateway extension (priority 10), 5s timeout, non-blocking, XML injection with D7 answer contract | Yes (with config fix needed) |
| **Vector search** | Qdrant v1.16.1, `qwen/qwen3-embedding-8b` (1024d), cosine similarity, topK=5 | Yes |
| **Memory system** | mem0-api: Qdrant (semantic) + Neo4j (graph), separate from knowledge | Yes |
| **Operator Knowledge UI** | Client-first navigation, scope browser, upload/delete/search | Yes |
| **Evaluation framework** | ADR-014: 8 criteria (3 hard at 100%), 6 scenario types, 7 evaluation artifacts created | Design + artifacts, not automated |
| **Test suite** | 150+ knowledge-specific tests, 2,494 total bot-config-api tests | Yes |

### What Works Well

1. **Ingestion pipeline is solid** — Two-phase atomic replace, hash-based dedup, tombstone lifecycle. This is not MVP-quality, it's production-grade data management. No need to touch it.

2. **Scope isolation is proven** — Server-side resolution, zero cross-scope leakage tested. ADR-008/009 are locked and validated.

3. **Non-blocking retrieval design** — Gateway plugin times out gracefully. Bot always answers. This is the right architecture — knowledge enhances answers, never blocks them.

4. **Evaluation framework is well-designed** — ADR-014 defines concrete criteria with thresholds. 3 hard requirements (100% scope isolation, zero fabricated citations, always disclose unavailability) are already automated in tests.

### What Is Weak (Honest)

1. **Retrieval quality is unoptimized** — Pure vector search with no reranking, no hybrid. Top-5 results rely entirely on embedding quality. For English technical docs, this may be sufficient. But we have no data to prove it.

2. **No production quality measurement** — Evaluation framework exists as design + test artifacts, but no retrieval quality metrics (nDCG, MRR) and no automated regression. We literally cannot answer "how good is our retrieval?"

3. **Configuration fragility** — Knowledge retrieval plugin config is NOT properly exposed in Helm configmap-config.yaml. Plugin silently disables if apiUrl not set. Embedding timeout hardcoded at 30s. No retry/circuit breaker on OpenRouter API.

4. **No observability** — Zero retrieval-specific metrics. No latency tracking, no cache hit rate, no result quality histogram. Cannot diagnose retrieval issues in production.

5. **Live pilot gate still open** — #307 blocks production deployment. Requires real client documents + manual evaluation.

---

## B. Critical Review of Existing Blueprint

The current `super-rag-blueprint.md` has several problems:

### Problems with the Blueprint

**1. Overstates gaps by comparing against generic RAG wishlist**

The blueprint scored current state at 37% by counting 60 items from a generic RAG blueprint. Many of those items are research-grade features irrelevant to production:

- "Reasoning Engine" (intent understanding, constraint identification, option generation) — This is not a RAG concern. This is agent design. Scoring RAG against it inflates the gap.
- "Tooling Layer" (cost estimator, diagram generator) — Not RAG. Not relevant.
- "Continuous Learning" (feedback loop, model improvement) — Aspirational, not production-blocking.

**Corrected scope**: If we count only items that actually affect retrieval quality and production readiness, current coverage is closer to **55-60%**, not 37%.

**2. Underweights current implementation strengths**

The blueprint gives ingestion 92% but doesn't emphasize that this is **already production-grade**. Two-phase atomic replace, SHA-256 dedup, tombstone lifecycle — these are enterprise patterns. Most RAG systems ship without this.

Similarly, evaluation framework gets 0% because "no automated pipeline exists." But ADR-014 + 22 evaluation artifact tests + 3 hard requirements automated is not 0%. It's **design-complete, automation-pending**.

**3. Puts Graph RAG too early (Phase 4)**

Blueprint places Graph RAG as Phase 4 — technically after retrieval quality. But the blueprint still includes it in the "production path." Graph RAG is a research enhancement that requires:
- Entity extraction pipeline (new LLM calls per ingest)
- Neo4j schema design for knowledge entities (separate from mem0 graph)
- Graph traversal at query time (latency impact)

**Recommendation**: Graph RAG should be **explicitly deferred** to post-production. It's not a production requirement.

**4. Missing a foundation hardening phase**

Blueprint jumps to "Phase 1: Evaluation" without addressing known fragility:
- Knowledge plugin Helm config gap (silently disabled)
- No retry/circuit breaker on embedding API
- No Qdrant health check
- Hardcoded timeouts and dimensions

These must be fixed BEFORE any quality measurement is meaningful. A **Phase 0** is missing.

**5. Misstates evaluation maturity**

Blueprint says evaluation is 0%. Reality:
- ADR-014 defines 8 criteria with thresholds ✅
- 3 hard requirements tested in automated suite ✅
- 7 evaluation artifacts created ✅
- 22 evaluation contract tests passing ✅
- Manual evaluation pending (live pilot #307) ⏳
- Automated quality regression pipeline ❌

Evaluation is **~60% complete**, not 0%.

---

## C. Candidate Solution Options — Debate

### C1. Retrieval Strategy

#### Option A: Vector-only baseline (current)
- **Pros**: Already working. Zero additional latency. Zero cost. Proven for English technical docs with good embeddings.
- **Cons**: Fails on exact keyword matches (e.g., error codes, API names). No way to improve without changing embedding model.
- **Evidence**: `qwen/qwen3-embedding-8b` scores #1 on MTEB for multilingual — but English-only production doesn't need multilingual SOTA. `text-embedding-3-large` or `voyage-3-large` may actually score better on English technical retrieval.
- **Verdict**: Sufficient for MVP. Not optimal.

#### Option B: Vector + keyword hybrid (Qdrant native)
- **Pros**: Catches exact matches that embeddings miss. Qdrant v1.16.1 supports native full-text search via payload text indexing — **zero new infrastructure**. Also supports sparse vectors for BM25-like scoring.
- **Cons**: Requires code changes (~500 lines) to enable text indexing + fusion logic. Requires re-creating collections (migration).
- **Cost**: $0 — Qdrant already deployed and running.
- **Verdict**: Best ROI. Uses capabilities we're already paying for.

#### Option C: Vector + keyword + reranker
- **Pros**: Reranking adds 15-30% nDCG improvement per research benchmarks (Nogueira et al., 2020). Catches edge cases where both vector and BM25 return noise.
- **Cons**: Adds 100-300ms latency. Requires either new service (ONNX reranker) or API cost (Cohere Rerank). Added operational complexity.
- **Evidence**: Current 5s timeout budget has room for +300ms. But adding a service increases failure surface.
- **Verdict**: High value but adds complexity. Should come AFTER hybrid proves insufficient.

**Recommendation: Option B first. Option C only if eval shows hybrid is insufficient.**

### C2. Reranker Choice (if needed)

#### Option A: Local ONNX (BAAI/bge-reranker-v2-m3)
- **Pros**: Zero API cost. Predictable latency (~100ms). No network dependency.
- **Cons**: Requires GPU or beefy CPU. New container deployment. Model updates require image rebuild.
- **Resource**: Needs ~2GB RAM, 0.5 CPU minimum. k3s on server36 has limited resources.

#### Option B: API-based (Cohere Rerank)
- **Pros**: No infrastructure. Pay per query (~$1/1K queries). Always latest model.
- **Cons**: Network dependency. Another API key to manage. Latency variability.

**Recommendation: Defer reranker decision until hybrid search eval data proves it's needed. If needed, start with Cohere API (lower operational burden) and migrate to ONNX if cost becomes an issue.**

### C3. Search Backend Strategy

#### Option A: Qdrant native capabilities first
- **Pros**: Already deployed (v1.16.1). Native full-text search since v1.7. Native sparse vectors. Zero new services. Zero additional cost.
- **Cons**: Full-text search less mature than Elasticsearch. No advanced NLP features (stemming, synonyms).
- **Evidence**: Qdrant full-text search uses payload text indexing with tokenization. For English technical docs, this is adequate.

#### Option B: External search engine (Elasticsearch/Meilisearch)
- **Pros**: Battle-tested full-text search. Advanced NLP (stemming, fuzzy matching, synonyms).
- **Cons**: New service to deploy, configure, monitor. Memory-hungry (Elasticsearch: 2-4GB min). Operational overhead. Data sync complexity.

**Recommendation: Option A — Qdrant native. We have the capability deployed and unused. Adding Elasticsearch for a knowledge base that will have < 10K documents in year 1 is overengineering.**

### C4. Ingestion Evolution

#### Option A: Keep current chunking
- **Pros**: Working. Tested. Three strategies cover most document types.
- **Cons**: Fixed-size (default) breaks semantic boundaries. No empirical data on which strategy works best.

#### Option B: Add semantic chunking
- **Pros**: Preserves meaning units. Better retrieval precision in theory.
- **Cons**: Requires additional embedding calls per chunk boundary decision. More complex. Already defined in enum but not implemented — suggests it was deferred intentionally.

#### Option C: Add structured ingestion (JSON/YAML schemas)
- **Pros**: API docs, config files get proper typed chunks. Higher-quality metadata.
- **Cons**: Niche use case. Limited to specific document types.

#### Option D: Add incremental re-indexing
- **Pros**: Faster updates for large documents. Only re-embed changed chunks.
- **Cons**: Complexity in diffing chunks. Current hash-based dedup already skips identical re-uploads.

**Recommendation: Keep current chunking for Phase 0-2. Add semantic chunking in Phase 3 ONLY IF eval data shows chunking quality is a bottleneck. Structured ingestion and incremental re-indexing are future enhancements.**

### C5. Graph RAG

#### Include now
- **Pros**: Enables multi-hop reasoning. Richer context.
- **Cons**: Highest complexity. Requires entity extraction pipeline (new LLM calls per ingest = cost + latency). Neo4j schema for knowledge entities is separate from mem0 graph. Graph traversal at query time adds 200-500ms. No proven demand from current clients.

#### Defer
- **Pros**: Focus on core retrieval quality first. Graph RAG value depends on having good base retrieval to build on. Current system doesn't even have quality metrics to measure Graph RAG impact.
- **Cons**: Delays a differentiation feature.

**Recommendation: Explicitly defer. Graph RAG requires a proven retrieval foundation + quality metrics to measure its impact. Building it now would be research, not production hardening.**

### C6. Evaluation Strategy

#### Option A: Manual-only (current path via #307)
- **Pros**: Already designed. Artifacts exist. #307 is the gate.
- **Cons**: Not repeatable. Cannot detect regressions. One-time.

#### Option B: Artifact + automated regression pipeline
- **Pros**: Repeatable. Detects regressions on code changes. CI-integrated.
- **Cons**: Requires golden dataset (client collaboration needed). Significant upfront effort.

#### Option C: Minimal viable eval gate for production promotion
- **Pros**: Pragmatic. Define 3-5 core queries per scope, check retrieval relevance + citation accuracy on every deploy. Not full pipeline, but catches regressions.
- **Cons**: Limited coverage. May miss edge cases.

**Recommendation: Option C first (minimal eval gate), evolve to Option B after first client deployment provides real data for golden dataset. #307 manual pilot runs in parallel.**

---

## D. Recommended Target Architecture

### Target Retrieval Stack

```
Query
  ↓
Qdrant vector search (cosine, top-20)     ← existing, increase limit
  +
Qdrant full-text search (BM25, top-20)    ← NEW: enable native text index
  ↓
Reciprocal Rank Fusion (RRF)              ← NEW: merge + deduplicate
  ↓
Top-5 by fused score
  ↓
Confidence threshold (configurable)        ← existing, make configurable via Helm
  ↓
Token budget enforcement                   ← existing
  ↓
XML injection into LLM context             ← existing
```

**No reranker in V1.** Added only if eval data proves hybrid is insufficient.

### Target Ingestion Stack

**No changes to ingestion pipeline in V1.** Current pipeline is production-grade.

Future enhancement (post-production): semantic chunking if eval shows chunk quality is a bottleneck.

### Target Evaluation Approach

1. **Minimal eval gate** (automated): 5 core queries per scope, check that retrieval returns relevant chunks (score > threshold) and citations map to real sources. Run on every deploy.
2. **Live pilot** (manual): Complete #307 with first client. 20+ queries, 6 scenario types.
3. **Regression pipeline** (future): Golden dataset from #307 results → automated nDCG/MRR tracking in CI.

### Target Observability Approach

| Metric | Type | Purpose |
|---|---|---|
| `knowledge_search_latency_seconds` | Histogram | Track p50/p95/p99 retrieval latency |
| `knowledge_search_results_count` | Histogram | Distribution of result counts per query |
| `knowledge_search_top_score` | Histogram | Track confidence of best result |
| `knowledge_ingest_chunks_total` | Counter | Volume of indexed data |
| `knowledge_embedding_latency_seconds` | Histogram | OpenRouter API latency tracking |
| `knowledge_search_errors_total` | Counter (by type) | Qdrant failures, timeout, embedding errors |

### Explicit Deferrals

| Item | Reason | Revisit When |
|---|---|---|
| **Graph RAG** | No proven demand. Requires retrieval foundation first. High complexity. | After Phase 2 eval shows multi-hop queries fail |
| **Reranker** | Adds latency + operational complexity. Hybrid search may be sufficient. | After Phase 2 eval shows hybrid insufficient |
| **Semantic chunking** | Current chunking untested against quality metrics. May be fine. | After Phase 1 eval shows chunk quality is bottleneck |
| **Structured ingestion** | Niche use case. No current demand. | Client request |
| **Auto-ingestion hooks** | Premature automation. Manual upload sufficient for first clients. | Scale demand |
| **Feedback loop / RLHF** | Research feature. No foundation to build on yet. | Post-production |
| **Multilingual optimization** | English-first production. qwen/qwen3-embedding-8b already handles English well. | Non-English client demand |

---

## E. Execution Phases

### Phase 0: Foundation Hardening (prerequisite)

**Why**: Known fragility that will cause production incidents if not fixed. Must be done before any quality measurement is meaningful.

| Task | Description | Risk if Skipped |
|---|---|---|
| **F0.1** Fix knowledge plugin Helm config | Add `knowledgeRetrieval` section to `configmap-config.yaml` — currently silently disabled | Knowledge injection doesn't work for new bots |
| **F0.2** Add retry + exponential backoff on embedding API | `tenacity.retry` with 3 attempts, backoff 1s→2s→4s | Transient OpenRouter failures cause ingest failures |
| **F0.3** Add circuit breaker on OpenRouter | After 5 failures, fast-fail for 5 minutes | Cascading failures during API outage |
| **F0.4** Expose embedding timeout as env var | Replace hardcoded 30s with `EMBEDDING_TIMEOUT_SECS` | Cannot tune without code redeploy |
| **F0.5** Expose vector dimension as env var | Replace hardcoded 1024 with `EMBEDDING_VECTOR_SIZE` | Model change requires code change |
| **F0.6** Add Qdrant health check endpoint | `/health` returns Qdrant reachability | Silent search failures undetectable |
| **F0.7** Add `score_threshold` to Qdrant search | Use Qdrant native param instead of post-filtering | Slightly more efficient, correct behavior |

**Scope**: Code changes only. No infrastructure changes. No new services. Estimated: 1-2 PRs.

### Phase 1: Evaluation Baseline

**Why**: Cannot improve what we cannot measure. Must run BEFORE hybrid search so we have a baseline to compare against.

| Task | Description |
|---|---|
| **E1.1** Define 5 core eval queries per scope | Work with first client to create representative queries with expected results |
| **E1.2** Implement retrieval metrics | nDCG@5, MRR, Precision@5 — computed against eval queries |
| **E1.3** Implement citation accuracy check | Verify every [N] maps to a real retrieved chunk |
| **E1.4** Minimal eval gate in CI | On PR to knowledge code: run eval queries, fail if nDCG@5 drops > 10% |
| **E1.5** Complete #307 live pilot | First client, 20+ queries, 6 scenarios, CTO sign-off |
| **E1.6** Baseline metrics report | Document: current nDCG@5, MRR, citation accuracy, latency p95 |

**Dependency**: E1.1 requires first client documents. E1.5 = #307 closure.

### Phase 2: Retrieval Quality (Hybrid Search)

**Why**: Highest ROI improvement that uses infrastructure we already have.

| Task | Description |
|---|---|
| **R2.1** Enable Qdrant payload text indexing | Create text index on `payload.text` field in each collection |
| **R2.2** Implement Qdrant native full-text search | Add text search query alongside vector search |
| **R2.3** Implement Reciprocal Rank Fusion (RRF) | Fuse vector + text results: `score = 1/(k + rank_vector) + 1/(k + rank_text)` |
| **R2.4** Collection migration | Re-create collections with text index (data migration script) |
| **R2.5** Add observability metrics | `knowledge_search_latency_seconds`, `knowledge_search_top_score`, etc. |
| **R2.6** Eval gate | Require nDCG@5 improvement ≥ 10% over Phase 1 baseline |

**Cost**: $0 — Qdrant native features. Code changes + collection migration only.

### Phase 3: Ingestion Improvements (if data supports)

**Gate**: Only proceed if Phase 1 eval shows chunk quality is a retrieval bottleneck.

| Task | Description |
|---|---|
| **I3.1** Implement semantic chunking | Embed sliding windows, split where cosine drops below threshold |
| **I3.2** Eval comparison | Compare nDCG@5: fixed-size vs paragraph vs semantic chunking on same golden dataset |
| **I3.3** Per-scope strategy config | Allow operators to choose chunking strategy per scope via UI |

### Phase 4+: Optional Advanced Tracks (post-production)

| Track | Trigger |
|---|---|
| **Reranker** | Phase 2 eval shows hybrid is insufficient (nDCG@5 < target) |
| **Graph RAG** | Multi-hop query failures identified in production |
| **Query expansion** | Eval shows short/ambiguous queries have low recall |
| **Caching** | Observability shows >30% duplicate queries within 5min |
| **Auto-ingestion** | Client requests automated document sync |

---

## F. Decision Points — Recommendations

### 1. Should we add Phase 0 foundation hardening before evaluation work?

**YES.** Phase 0 is mandatory. Known issues (Helm config gap, no retry, hardcoded values) will cause production incidents. Evaluation on a fragile foundation produces unreliable metrics. Phase 0 scope is small (1-2 PRs) and should take < 1 week.

### 2. Is evaluation truly Phase 1, or does anything have to happen before it?

**Phase 0 → Phase 1.** Evaluation is correctly Phase 1, but Phase 0 must come first. Additionally, Phase 1 depends on first client documents (for eval queries) — this is a business dependency, not a technical one.

### 3. Is hybrid + reranker the highest-ROI retrieval path?

**Hybrid YES, reranker DEFER.** Hybrid search using Qdrant native full-text indexing is zero-cost infrastructure and catches exact keyword matches that embeddings miss. Reranker adds latency and operational complexity — defer until eval data proves it's needed.

### 4. Should Graph RAG be deferred?

**YES — explicitly deferred.** Reasons:
- No proven client demand for multi-hop queries
- Requires entity extraction pipeline (new LLM calls per ingest = cost)
- Requires quality metrics to measure impact (not available until Phase 1)
- High implementation complexity for uncertain value
- Focus should be on core retrieval quality first

### 5. What is the minimum production-ready eval gate?

**Minimal viable**: 5 core queries per scope + automated nDCG@5 check in CI + #307 manual pilot completion with CTO sign-off. Full regression pipeline is Phase 1+ enhancement.

### 6. Which parts are required for production, which are optional?

| Requirement Level | Items |
|---|---|
| **REQUIRED for production** | Phase 0 (foundation hardening), #307 completion (live pilot), basic observability metrics |
| **REQUIRED for production quality** | Phase 1 (evaluation baseline), Phase 2 (hybrid search) |
| **OPTIONAL future work** | Reranker, Graph RAG, semantic chunking, auto-ingestion, feedback loop, query expansion |

---

## G. Decision Summary

| Decision | Verdict | Rationale |
|---|---|---|
| Add Phase 0 before evaluation | **RECOMMENDED** | Known fragility must be fixed first |
| Hybrid search via Qdrant native | **RECOMMENDED** | Zero cost, uses deployed capability, highest ROI |
| Reranker (any type) | **DEFERRED** | Wait for hybrid eval data |
| Graph RAG | **DEFERRED** | No demand, high complexity, needs retrieval foundation |
| Semantic chunking | **DEFERRED** | Current chunking untested; may be sufficient |
| External search engine (ES/Meilisearch) | **REJECTED** | Qdrant native is sufficient for current scale |
| Multilingual optimization | **DEFERRED** | English-first production assumption |
| Auto-ingestion hooks | **DEFERRED** | Manual upload sufficient for first clients |
| Feedback loop / RLHF | **REJECTED for now** | Research feature, no foundation to build on |
| Minimal eval gate for CI | **RECOMMENDED** | Pragmatic, catches regressions, evolves to full pipeline |
| Full golden dataset pipeline | **DEFERRED** | Requires real client data from #307 |

### Blueprint Revision Required

The existing `super-rag-blueprint.md` should be updated to:
1. ~~37% coverage score~~ → Reframe as ~55-60% with production-relevant scope
2. Add Phase 0 (foundation hardening)
3. Move Graph RAG from Phase 4 to "Deferred — post-production"
4. Remove non-RAG items from gap scoring (Reasoning Engine, Tooling Layer)
5. Correct evaluation maturity from 0% to ~60%
6. Specify Qdrant native for hybrid search (not generic "BM25")
7. Remove reranker from core production path (move to conditional Phase 4)

---

*This debate document is READY_FOR_CTO_REVIEW.*
