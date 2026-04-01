---
title: "[Production] Super RAG — Production-Grade Retrieval"
---

# Super RAG — Production-Grade Retrieval

**Milestone**: [#34](https://github.com/ai-agentopia/agentopia-protocol/milestone/34)
**Status**: In progress. Phase 0 (#316) implementation complete and in PR review.
**Date**: 2026-04-01
**Type**: Production program document
**Primary repos**: `agentopia-protocol` (bot-config-api, knowledge-api), `agentopia-infra` (Helm charts)

---

## 1. Traceability

### GitHub Artifacts

- Milestone #34: `[Production] Super RAG — Production-Grade Retrieval`
- **Committed issues**:
  - #316: Phase 0 — Foundation Hardening
  - #317: Phase 1a — RAGAS Evaluation Early Signal
  - #318: Phase 1b — Labeled Evaluation Baseline
  - #319: Phase 2a — Hybrid Retrieval (Dense + Sparse TF + RRF)
  - #320: Phase 2b — Knowledge-API Service Extraction
- **Conditional issues**:
  - #321: Phase 3 Candidate — Contextual Retrieval
  - #322: Phase 3 Candidate — Reranker
  - #323: Phase 3 Candidate — Semantic Chunking
- **Deferred issues**:
  - #324: Deferred — Graph RAG
  - #325: Deferred — Agentic RAG / Auto-Ingestion / Feedback Loop

### Related Docs

- Architecture debate: [Super RAG Debate](../architecture/super-rag-debate.md)
- Blueprint gap analysis: [Super RAG Blueprint](../architecture/super-rag-blueprint.md)
- Foundation: [SA Knowledge Base Milestone](production-sa-knowledge-base.md) (milestone #33)
- ADRs: 008-014 (SA-KB architecture, locked)

---

## 2. Context / Business Driver

The SA Knowledge Base (milestone #33) provides a working foundation: scoped ingestion, vector search, provenance/citations, operator UI, and evaluation design. However, retrieval quality is unmeasured — pure vector search with no hybrid, no reranking, no quality metrics. Production clients need reliable, measurable retrieval quality.

Super RAG upgrades the retrieval stack from "operationally present" to "production-grade" — measurable quality, hybrid search, observability, and independent service lifecycle.

---

## 3. Foundation (SA-KB Baseline)

This milestone builds on SA-KB (milestone #33). The following are **already implemented and tested**:

| Component | Status | Evidence |
|---|---|---|
| Scoped knowledge model | Implemented | `{client_id}/{scope_name}`, 18 auth tests |
| File-upload ingestion | Implemented | PDF/HTML/Markdown/code, SHA-256 dedup, two-phase atomic replace |
| Postgres document lifecycle | Implemented | active/superseded/deleted states, tombstone ledger |
| Provenance & citations | Implemented | source, section, page, chunk_index, score — 35 tests |
| Runtime retrieval plugin | Production-pending | Gateway extension (priority 10), Helm config gap (fixed in Phase 0) |
| Vector search | Operationally present | Qdrant v1.16.1, cosine similarity, topK=5 |
| Operator Knowledge UI | Implemented | Client-first navigation, scope browser, upload/delete/search |
| Evaluation framework (design) | Design-complete | ADR-014: 8 criteria, 6 scenarios, 22 contract tests |
| Test suite | Implemented | 150+ knowledge tests, 2,494 total bot-config-api tests |

---

## 4. Program Objective

1. **Measure** retrieval quality with reference-free (RAGAS) and labeled (nDCG@5, MRR) metrics
2. **Improve** retrieval via hybrid search (dense + sparse TF + RRF fusion via Qdrant)
3. **Observe** retrieval behavior with production Prometheus metrics
4. **Extract** knowledge service into independent knowledge-api within the `agentopia-protocol` monorepo (service extraction, not repo split)
5. **Gate** all improvements with labeled evaluation metrics — no unmeasured claims

---

## 5. Two-Track Execution Plan

### Track A: Retrieval Quality (sequential)

```
Phase 0  → Foundation hardening (config, retry, circuit breaker, health)
Phase 1a → RAGAS early signal (reference-free, immediate after Phase 0)
Phase 1b → Labeled eval baseline (after #307 pilot, golden dataset, nDCG@5/MRR/Precision@5)
Phase 2a → Hybrid retrieval (dense + sparse TF + RRF, labeled eval gate: nDCG@5 ≥ 10%)
Phase 3+ → Conditional experiments (evidence-driven only)
```

### Track B: Service Architecture (after Track A Phase 2a stabilizes)

```
Phase 2b → Knowledge-API extraction (new service in agentopia-protocol monorepo, NOT new repo)
           Proxy-first auth model
           Bot bearer via K8s Secret read
           Binding sync webhook + cache-miss fallback + periodic reconcile
           Topology gate: p95 latency must not increase >200ms
```

**Locked constraint**: knowledge-api is a new service directory inside `agentopia-protocol`. Repo extraction deferred — only reconsidered with evidence of independent release cadence, distinct team ownership, or monorepo CI/CD blocking delivery.

**Why separated**: If Phase 2a causes quality regression → retrieval logic. If Phase 2b causes issues → topology. Cannot diagnose if both change simultaneously.

---

## 6. Required Gates

| Gate | Phase | Metric | Threshold |
|---|---|---|---|
| RAGAS regression | 1a | Faithfulness score | Warn if drops >15% (directional only) |
| Labeled baseline | 1b | nDCG@5, MRR, Precision@5 | Document current values as baseline |
| Hybrid eval | 2a | nDCG@5 improvement | ≥ 10% over Phase 1b labeled baseline |
| Extraction topology | 2b | Retrieval latency p95 | Must not increase >200ms vs Phase 2a |

---

## 7. Explicit Deferrals

| Item | Reason | Revisit When |
|---|---|---|
| Graph RAG | No client demand, high complexity | Multi-hop query failures in production |
| Reranker | Hybrid may suffice, adds latency | Phase 2a nDCG@5 plateaus below target |
| Contextual Retrieval | Unvalidated on Agentopia data | Phase 2a shows chunk quality limits recall |
| Semantic chunking | Current chunking untested | Phase 1 shows chunk boundaries are the problem |
| Agentic RAG | Injection-based is correct for current use case | Injection consistently misses context |
| Auto-ingestion | Manual upload sufficient | Scale demand from clients |

---

## 8. Definition of Done

1. Retrieval quality measured with labeled metrics (nDCG@5, MRR, Precision@5)
2. Hybrid search (dense + sparse TF + RRF) deployed and passing labeled eval gate (≥ 10% nDCG@5 improvement)
3. Knowledge service extracted to independent knowledge-api with proxy-first auth
4. Observability: 6+ retrieval-specific Prometheus metrics in production
5. All conditional/deferred items have explicit trigger conditions documented
6. Migration/rollback from knowledge-api to bot-config-api tested and documented

---

## 9. Issue Status

| # | Phase | Title | Status |
|---|---|---|---|
| #316 | 0 | Foundation Hardening | PR review |
| #317 | 1a | RAGAS Evaluation Early Signal | PR review |
| #318 | 1b | Labeled Evaluation Baseline | Wave 1 complete, Wave 2 pending #307 |
| #319 | 2a | Hybrid Retrieval (Sparse TF + RRF) | Wave 1 complete — real comparison run on live Qdrant. Wave 2 pending #307. |
| #320 | 2b | Knowledge-API Extraction | Open |
| #321 | 3 | Conditional: Contextual Retrieval | Conditional |
| #322 | 3 | Conditional: Reranker | Conditional |
| #323 | 3 | Conditional: Semantic Chunking | Conditional |
| #324 | — | Deferred: Graph RAG | Deferred |
| #325 | — | Deferred: Agentic RAG / Auto-Ingestion / Feedback Loop | Deferred |

---

## Change Log

### 2026-04-01

- Milestone #34 created
- 10 issues created (#316-#325): 5 committed, 3 conditional, 2 deferred
- Tracking doc created
- Roadmap updated
- Phase 0 (#316) implementation complete: retry/backoff, circuit breaker, externalized config, Qdrant health, score_threshold, source_type metadata. 20 new tests, 153 existing pass.
- Phase 1a (#317) implementation complete: RAGAS evaluation harness, 5-sample dataset, 3 reference-free metrics, cost-controlled runner, JSON+MD artifacts. 20 tests.
- Phase 1b (#318) Wave 1 complete: labeled baseline harness, seed dataset (5 queries, graded relevance), nDCG@5/MRR/Precision@5/Recall@5 metrics, e2e baseline run (nDCG@5=0.8774). Wave 2 pending #307 pilot data. 26 tests.
- Phase 2a (#319) Wave 1 COMPLETE: hybrid retrieval (dense + sparse TF + RRF) implemented and real comparison run on live Qdrant. Dense nDCG@5=1.0, Hybrid nDCG@5=0.9262 on 5-doc seed (dense-optimal for tiny corpus — hybrid advantage expected on larger data). 16 tests. Wave 2 pending (authoritative eval + #307 pilot data).
