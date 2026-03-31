---
title: "Agentopia Super RAG — Production Blueprint"
description: "Gap analysis against production RAG blueprint, current state mapping, and phased upgrade plan"
---

# Agentopia Super RAG — Production Blueprint

> **Purpose**: Map the [RAG Production Blueprint](../rag-production) against what Agentopia actually has today, identify gaps honestly, and propose a phased upgrade plan for CTO verdict.
>
> **Date**: 2026-03-31
> **Author**: DevOps/Platform team
> **Status**: DRAFT — awaiting CTO verdict

---

## 1. Blueprint vs Current State — Honest Mapping

### 1.1 API Layer

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Entry point for all requests | `bot-config-api` FastAPI — REST endpoints for ingest/search/delete | **HAVE** |
| Authentication | Dual-path: operator session cookie + bot bearer token (AGENTOPIA_RELAY_TOKEN) | **HAVE** |
| Rate limiting | **NOT IMPLEMENTED** — no per-client or per-bot rate limiting | **GAP** |
| Request logging | Basic FastAPI access logs — no structured audit logging per knowledge request | **PARTIAL** |

### 1.2 Agent Orchestrator

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Manage agent workflow | Gateway plugin system — `knowledge-retrieval` extension (priority 10) auto-injects before LLM turn | **HAVE** |
| Decide which tools to call | Knowledge retrieval is **non-tool-based** — always injects if results found. Not LLM-decided. | **DIFFERENT APPROACH** |
| Control reasoning steps | No multi-step reasoning for retrieval. Single query → single search → inject. | **GAP** |

**Architecture note**: Current design is intentionally non-tool-based (ADR-010). Knowledge is guaranteed to be injected — the LLM doesn't choose whether to search. This is a deliberate trade-off: reliability over flexibility.

### 1.3 Reasoning Engine

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Intent understanding | Not implemented — raw user query goes directly to embedding | **GAP** |
| Context gathering | Single-pass: embed query → cosine search → top-K | **BASIC** |
| Constraint identification | Not implemented | **GAP** |
| Option generation | Not implemented | **GAP** |
| Trade-off analysis | Not implemented | **GAP** |
| Structured output (Problem → Assumptions → Options → Recommendation) | D7 answer contract exists (cite [N], handle conflicts) but no structured reasoning template | **PARTIAL** |

### 1.4 RAG System — Data Ingestion

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Document parsing | PDF (PyPDF2), HTML (BeautifulSoup), Markdown, plain text, code files | **HAVE** |
| Chunking | 3 strategies: fixed-size (512/64), paragraph, code-aware. Semantic chunking defined but **NOT implemented**. | **PARTIAL** |
| Metadata tagging | Rich: source, format, scope, section (markdown heading), chunk_index, total_chunks, ingested_at, document_hash | **HAVE** |
| Embedding | `qwen/qwen3-embedding-8b` via OpenRouter (1024 dims, multilingual SOTA) | **HAVE** |
| Vector DB | Qdrant v1.16.1, collection-per-scope, cosine similarity, 5Gi PVC | **HAVE** |
| Versioning support | Two-phase atomic replace, SHA-256 same-hash short-circuit, tombstone lifecycle (active/superseded/deleted) in Postgres | **HAVE** |
| Incremental updates | **NOT IMPLEMENTED** — full re-embed on document update. Same-hash skip only. | **GAP** |

### 1.5 RAG System — Retrieval Strategy

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Vector search (semantic) | Qdrant cosine similarity, topK=5 | **HAVE** |
| Keyword search (BM25) | **NOT IMPLEMENTED** — pure vector only | **GAP** |
| Hybrid retrieval | **NOT IMPLEMENTED** | **GAP** |
| Reranking (cross-encoder / LLM) | **NOT IMPLEMENTED** — results ordered by raw Qdrant score | **GAP** |
| Context builder (business constraints + system state) | Token budget enforcement (3000 tokens), confidence threshold (0.3), but no business constraint injection | **PARTIAL** |
| Query expansion / decomposition | **NOT IMPLEMENTED** | **GAP** |

### 1.6 Knowledge Types

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Static knowledge (docs, policies) | Document upload → Qdrant via knowledge scopes | **HAVE** |
| Dynamic knowledge (recent decisions) | **NOT IMPLEMENTED** — no auto-ingestion of new decisions/conversations | **GAP** |
| Structured knowledge (schemas, configs) | **NOT IMPLEMENTED** — all documents treated as unstructured text | **GAP** |

### 1.7 Memory System

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Episodic memory (conversation history) | Gateway session history + PVC file persistence | **HAVE** |
| Semantic memory (extracted facts) | mem0-api → Qdrant `agentopia_memory` collection, LLM-extracted facts | **HAVE** |
| Procedural memory (how problems were solved) | **NOT IMPLEMENTED** — no decision pattern storage | **GAP** |
| Graph memory (entity relationships) | mem0-api → Neo4j (entity graph with relationships) | **HAVE** |
| Short-term storage | In-memory session context | **HAVE** |
| Long-term storage | Qdrant (vector) + Neo4j (graph) | **HAVE** |
| Memory retrieval + knowledge retrieval → final prompt | Both inject separately: knowledge (priority 10) then memory (priority 20) before LLM. Not merged/fused. | **PARTIAL** |

### 1.8 Tooling Layer

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Analytical tools (cost estimator, perf calculator) | **NOT IMPLEMENTED** | **GAP** |
| Knowledge tools (document search as tool) | Knowledge search is **auto-injected**, not exposed as callable tool | **DIFFERENT APPROACH** |
| Generation tools (diagram generator, proposal generator) | **NOT IMPLEMENTED** | **GAP** |
| External integrations (APIs) | MCP bridge for GitHub tools — but not knowledge-related | **PARTIAL** |
| Tool invocation flow | Not applicable — current RAG is injection-based, not tool-based | **N/A** |

### 1.9 Evaluation System

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Correctness evaluation | ADR-014 defines 8 criteria (3 hard at 100%) — but **no automated evaluation pipeline** | **DESIGN ONLY** |
| Faithfulness (grounded in retrieved data) | D7 answer contract requires [N] citations — but **no automated faithfulness check** | **DESIGN ONLY** |
| Completeness evaluation | **NOT IMPLEMENTED** | **GAP** |
| Reasoning quality evaluation | **NOT IMPLEMENTED** | **GAP** |
| Automated evaluation pipeline | **NOT IMPLEMENTED** — evaluation is manual via live pilot (#307) | **GAP** |
| Score tracking / regression detection | **NOT IMPLEMENTED** | **GAP** |

### 1.10 Guardrails & Safety

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Input validation | Basic: min query length (5 chars), format validation on upload | **BASIC** |
| Prompt constraints | D7 answer contract: cite sources, don't fabricate citations, handle conflicts | **HAVE** |
| Output validation | **NOT IMPLEMENTED** — no post-generation check for hallucination | **GAP** |
| Post-processing filters | **NOT IMPLEMENTED** | **GAP** |
| Hallucination detection | **NOT IMPLEMENTED** | **GAP** |

### 1.11 Production Infrastructure

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| System architecture | Frontend → API Gateway → bot-config-api → Qdrant + Postgres | **HAVE** |
| Observability (logging) | FastAPI access logs, gateway plugin logs (non-blocking) | **BASIC** |
| Observability (tracing) | **NOT IMPLEMENTED** — no per-retrieval tracing | **GAP** |
| Observability (metrics) | No Qdrant-specific metrics, no retrieval latency SLO | **GAP** |
| Caching | **NOT IMPLEMENTED** — every query re-embeds and re-searches | **GAP** |
| Scalability (async processing) | Single-threaded sequential ingestion, single-replica Qdrant | **GAP** |

### 1.12 Continuous Learning

| Blueprint Requirement | Current State | Verdict |
|---|---|---|
| Auto data ingestion | **NOT IMPLEMENTED** — manual upload only | **GAP** |
| Re-indexing | Full re-embed on update, no incremental | **BASIC** |
| User feedback loop | **NOT IMPLEMENTED** | **GAP** |
| Model improvement cycle | **NOT IMPLEMENTED** | **GAP** |

---

## 2. Scorecard Summary

| Blueprint Section | Items | HAVE | PARTIAL | GAP | Score |
|---|---|---|---|---|---|
| API Layer | 4 | 2 | 1 | 1 | 63% |
| Agent Orchestrator | 3 | 1 | 0 | 2 | 33% |
| Reasoning Engine | 6 | 0 | 2 | 4 | 17% |
| Data Ingestion | 6 | 5 | 1 | 0 | 92% |
| Retrieval Strategy | 6 | 1 | 1 | 4 | 25% |
| Knowledge Types | 3 | 1 | 0 | 2 | 33% |
| Memory System | 7 | 5 | 1 | 1 | 79% |
| Tooling Layer | 5 | 0 | 1 | 3 | 10% |
| Evaluation System | 6 | 0 | 0 | 6 | 0% |
| Guardrails | 5 | 2 | 0 | 3 | 40% |
| Infrastructure | 5 | 1 | 1 | 3 | 30% |
| Continuous Learning | 4 | 0 | 1 | 3 | 13% |
| **TOTAL** | **60** | **18** | **9** | **32** | **37%** |

**Current coverage: 37% of production RAG blueprint.**

**Strengths** (what we genuinely have):
- Data ingestion pipeline is solid (92%) — parsing, chunking, metadata, lifecycle
- Memory system is strong (79%) — semantic + graph + episodic
- Scope-based multi-tenancy with server-side resolution

**Critical gaps**:
- Retrieval quality (25%) — no hybrid search, no reranking
- Evaluation (0%) — cannot measure if RAG actually works well
- Continuous learning (13%) — no feedback loop

---

## 3. Super RAG Upgrade Plan — Phased Approach

### Design Principles

1. **Measure before optimize** — build evaluation first so we know if changes improve quality
2. **Highest ROI first** — reranking gives the biggest retrieval quality lift for least effort
3. **Don't break what works** — ingestion pipeline and scope model are solid, build on them
4. **Production-grade means observable** — every component must have metrics

---

### Phase 1: Evaluation Foundation (prerequisite for everything)

**Why first**: Without evaluation, we cannot prove any improvement. Every subsequent phase needs a quality baseline.

| Task | Description | Effort |
|---|---|---|
| **E1.1 Golden dataset** | Curate 50+ query-answer pairs per knowledge scope with expected source citations | Dataset curation |
| **E1.2 Retrieval metrics** | Implement nDCG@K, MRR, Recall@K, Precision@K on golden dataset | Backend code |
| **E1.3 Answer quality metrics** | Implement faithfulness score (% claims with valid [N] citation), completeness score | LLM-as-judge |
| **E1.4 Automated eval pipeline** | CI-runnable: ingest golden docs → run queries → compute metrics → report | CI integration |
| **E1.5 Baseline measurement** | Run eval on current system. This is the number we improve against. | Run once |

**Deliverable**: Baseline retrieval quality score. All future phases must show improvement on this baseline.

**Dependencies**: Golden dataset requires client collaboration (real documents + expected answers).

---

### Phase 2: Retrieval Quality (highest ROI)

**Why**: Reranking is the single highest-impact improvement. Research shows reranking improves nDCG@10 by 15-30% over raw vector search (Nogueira et al., 2020; ColBERT benchmarks).

| Task | Description | Effort |
|---|---|---|
| **R2.1 Reranker service** | Deploy cross-encoder reranker (options: `BAAI/bge-reranker-v2-m3` local via ONNX, or Cohere Rerank API). Rerank top-20 → return top-5. | New service + Helm chart |
| **R2.2 Hybrid search** | Enable Qdrant full-text index alongside vector. Query with both, fuse via Reciprocal Rank Fusion (RRF). | Qdrant config + backend code |
| **R2.3 Query preprocessing** | Strip noise, normalize query. Optional: LLM query expansion for ambiguous queries. | Backend code |
| **R2.4 Eval gate** | Run Phase 1 eval. Require nDCG@5 improvement ≥ 10% vs baseline to merge. | CI gate |

**Retrieval flow after Phase 2**:
```
Query → embed → Qdrant vector search (top-20)
                 ↓
Query → BM25 full-text search (top-20)
                 ↓
         RRF fusion (top-20 merged)
                 ↓
         Cross-encoder reranker (top-20 → top-5)
                 ↓
         Token budget + confidence filter
                 ↓
         Inject into LLM context
```

**Trade-off**: Reranker adds 100-300ms latency. Current 5s timeout budget has room. Monitor via metrics.

---

### Phase 3: Semantic Chunking + Advanced Ingestion

**Why**: Current fixed-size chunking breaks semantic boundaries. Semantic chunking preserves meaning units, improving retrieval precision.

| Task | Description | Effort |
|---|---|---|
| **I3.1 Semantic chunking** | Implement the `SEMANTIC` strategy: embed sliding windows, split where cosine similarity drops below threshold. Uses same embedding model. | Backend code |
| **I3.2 Hierarchical metadata** | Extract document structure (headings, sections, subsections) as tree metadata. Enable section-level retrieval. | Backend code |
| **I3.3 Incremental re-indexing** | On document update, diff chunks (by hash) — only re-embed changed chunks, not full document. | Backend code |
| **I3.4 Structured knowledge** | Support JSON/YAML schema ingestion: extract field descriptions, relationships, constraints as typed chunks. | Backend code |
| **I3.5 Eval gate** | Require precision@5 improvement on golden dataset. | CI gate |

---

### Phase 4: Graph RAG + Knowledge Fusion

**Why**: Combine document knowledge (Qdrant) with entity knowledge (Neo4j) for richer context. Currently these are separate systems that don't talk to each other.

| Task | Description | Effort |
|---|---|---|
| **G4.1 Entity extraction on ingest** | During document ingestion, extract named entities + relationships via LLM. Store in Neo4j alongside mem0 graph. | Backend code |
| **G4.2 Graph-augmented retrieval** | On search: vector search → identify entities in results → traverse Neo4j for related entities → enrich context. | Backend code |
| **G4.3 Memory-knowledge fusion** | Merge knowledge retrieval (priority 10) and memory recall (priority 20) into single ranked context, not sequential injection. | Gateway extension refactor |
| **G4.4 Eval gate** | Require completeness score improvement on multi-hop questions. | CI gate |

**Architecture change**: This bridges the currently separate knowledge and memory systems.

---

### Phase 5: Observability + Production Hardening

| Task | Description | Effort |
|---|---|---|
| **O5.1 Retrieval metrics** | Prometheus counters: retrieval_latency_seconds, retrieval_result_count, retrieval_confidence_histogram, reranker_latency_seconds | Backend code |
| **O5.2 Per-query tracing** | OpenTelemetry trace: embed → search → rerank → inject. Trace ID in gateway logs. | Backend + gateway |
| **O5.3 Query cache** | Redis cache for frequent queries: key=hash(query+scopes), TTL=5min. Invalidate on document update. | Backend + Redis |
| **O5.4 Rate limiting** | Per-client rate limit on ingest and search endpoints. | Backend code |
| **O5.5 Grafana dashboard** | RAG-specific dashboard: latency p50/p95/p99, cache hit rate, retrieval quality trend | Grafana |

---

### Phase 6: Guardrails + Continuous Learning

| Task | Description | Effort |
|---|---|---|
| **S6.1 Hallucination detection** | Post-generation check: verify every [N] citation maps to actual retrieved chunk. Flag uncitable claims. | Backend code |
| **S6.2 Answer confidence scoring** | Compute answer confidence: (avg retrieval score × citation coverage). Surface to operator UI. | Backend + UI |
| **S6.3 Feedback collection** | Operator can thumbs-up/down bot answers. Store with query + retrieved chunks + answer. | UI + Backend |
| **S6.4 Feedback-driven re-ranking** | Use collected feedback to fine-tune reranker or adjust confidence thresholds per scope. | ML pipeline |
| **S6.5 Auto-ingestion hooks** | Webhook/watch on document sources (Git repo, shared drive). Auto-ingest on change. | Backend code |

---

## 4. Phase Priority & Dependencies

```
Phase 1 (Evaluation)     ← MUST DO FIRST — everything depends on this
    ↓
Phase 2 (Retrieval)      ← Highest ROI, needs eval to prove improvement
    ↓
Phase 3 (Ingestion)      ← Improves input quality, needs eval to measure
    ↓
Phase 4 (Graph RAG)      ← Most complex, needs solid retrieval foundation
    ↓
Phase 5 (Observability)  ← Can start in parallel with Phase 2-3
    ↓
Phase 6 (Guardrails)     ← Needs all above running in production
```

**Parallel tracks possible**:
- Phase 5 (Observability) can start alongside Phase 2
- Phase 1 (Evaluation) can start immediately

---

## 5. Risk Assessment

| Risk | Impact | Mitigation |
|---|---|---|
| Reranker latency exceeds 5s timeout | Bot answers without knowledge context | Monitor latency, set reranker timeout at 2s, fallback to raw vector results |
| Semantic chunking degrades quality for some document types | Worse retrieval than fixed-size | Eval gate per phase — must improve metrics to merge. Keep strategy configurable per scope. |
| Graph RAG entity extraction hallucinations | Bad entities in Neo4j corrupt retrieval | LLM extraction → human review queue for high-value scopes |
| Golden dataset not representative | Eval metrics don't reflect real usage | Start with real client queries from pilot (#307). Update dataset quarterly. |
| Cost increase from reranker + additional LLM calls | Higher per-query cost | Local ONNX reranker (BAAI/bge-reranker-v2-m3) = zero API cost. Query cache reduces repeat calls. |

---

## 6. Decision Points for CTO

1. **Phase 1 scope**: How many golden query-answer pairs per scope? 50 minimum recommended.
2. **Reranker choice**: Local ONNX (zero cost, +100ms latency, needs GPU/CPU) vs Cohere API ($1/1K queries, +200ms)?
3. **Graph RAG priority**: Phase 4 is the most complex. Should it be deferred to post-MVP Super RAG?
4. **Hybrid search**: Qdrant native full-text or external Elasticsearch/Meilisearch?
5. **Feedback loop**: Is operator feedback collection (Phase 6) a requirement for V1 Super RAG?

---

## 7. Success Criteria

| Metric | Current Baseline | Phase 2 Target | Phase 4 Target |
|---|---|---|---|
| nDCG@5 | Unknown (no eval) | ≥ 15% improvement over baseline | ≥ 25% improvement |
| Retrieval latency p95 | ~2000ms (estimated) | < 3000ms (with reranker) | < 3500ms |
| Citation faithfulness | Unknown | ≥ 90% valid citations | ≥ 95% |
| Answer completeness | Unknown | Measured | ≥ 80% on multi-hop queries |
| Blueprint coverage | 37% | 55% | 75% |

---

*Awaiting CTO verdict on phase priorities and decision points before implementation begins.*
