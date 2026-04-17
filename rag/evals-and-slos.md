---
title: "Agentopia RAG — Evaluations and SLOs"
status: "ACTIVE"
---

# RAG Evaluations and SLOs

## Overview

This document defines the full set of service level objectives for the Agentopia RAG system. It covers five dimensions:

1. **Route Correctness** — queries reach the correct plane (Knowledge vs Operational vs Memory)
2. **Retrieval Quality** — knowledge chunks returned are relevant and ranked well
3. **Freshness** — time from source change to Qdrant update
4. **Contamination** — operational state must not appear in knowledge retrieval results
5. **Latency** — end-to-end retrieval time budget

For each dimension: definition, measurement method, baseline, target SLO, and breach response.

---

## 1. Route Correctness

### Definition

A query is correctly routed if it reaches the plane appropriate for its query family, and no disallowed plane is triggered. The six query families and their routing rules are defined in `architecture.md`, Section 1.

Route correctness must be measured across **all six families**, not only for one family or one example query class. A routing design that works for one family but fails for others is not correct.

| Family | Correct outcome |
|---|---|
| `operational_state` | governance-bridge fires, tool call made, knowledge-retrieval does NOT fire |
| `planning_readiness` | governance-bridge fires, tool call made, knowledge-retrieval does NOT fire |
| `knowledge_query` | knowledge-retrieval fires, Qdrant searched, governance-bridge does NOT fire |
| `memory_query` | mem0-api fires, memory recalled, knowledge-retrieval and governance-bridge do NOT fire (unless both are genuinely needed) |
| `runtime_meta` | no retrieval triggered; system prompt handles the response |
| `general_chat` | no retrieval triggered |
| `unknown_route` | no retrieval triggered from any plane; LLM responds from general context |

**`unknown_route` eval**: The labeled query set includes 5 queries specifically designed to test uncertain classification — query forms that could plausibly belong to multiple families. The correct outcome for these is `unknown_route` (no retrieval). If any of these queries trigger knowledge-retrieval or memory-retrieval, the router is incorrectly force-assigning uncertain queries to a retrieval-triggering family.

A misroute is counted when any plugin fires that should not fire for that query's family, or when no plugin fires when one should.

### Measurement

**Method**: Labeled query set — 60 queries, 10 per family. Each query is labeled with its correct family and expected plugin behavior.

For each query, log which plugins fire by inspecting `event.coordination` in the gateway event payload.

**Eval script**: `agentopia-protocol/evaluation/route-correctness.py`

```python
# Per-family misroute rates computed separately
for family in QUERY_FAMILIES:
    misroute_rate[family] = misrouted_in_family[family] / total_in_family[family]
overall_correctness = correct_routings / total_queries
```

### Baseline

Baseline established at P0.3. Current routing is heuristic-only (keyword regex). Coverage gaps are expected in `operational_state`, `planning_readiness`, `runtime_meta`, and `general_chat` families. Route correctness is expected to be below the target SLO until Phase 1 implementation is complete.

Baseline values: recorded in `agentopia-super-rag/evaluation/results/route-correctness-baseline.json` after P0.3 measurement.

### SLOs

| SLO | Target | Measurement window |
|---|---|---|
| Route correctness (overall) | ≥ 95% | Weekly on labeled query set |
| Misroute rate per family | ≤ 5% | Per family, weekly |
| Worst-case family misroute | ≤ 10% | No single family may exceed this |
| knowledge-retrieval fires on operational_state or planning_readiness | ≤ 2% | Weekly — this is the contamination-risk direction |

### Breach Response

- **Any family misroute > 10%**: Extend intent routing coverage for that family (Phase 1 pattern extension). Requires PR + eval re-run. Block further routing changes until gate passes.
- **knowledge-retrieval fires on operational_state or planning_readiness > 2%**: Extend `NON_KB_PATTERNS` with patterns covering those family forms. Verify `event.coordination.resolvedBy` is being set by governance-bridge.
- **governance-bridge fires on knowledge_query**: Narrow keyword patterns or add exclusion list to governance-bridge. Re-run eval.

---

## 2. Retrieval Quality

### Definition

For knowledge queries that correctly reach the knowledge-retrieval plugin, the returned chunks must be relevant to the query and well-ranked.

**Primary metric**: nDCG@5 (Normalized Discounted Cumulative Gain at rank 5) — measures both relevance and ranking quality.

**Secondary metrics**: MRR (Mean Reciprocal Rank), P@5, R@5.

### Measurement

**Method**: Golden question sets per scope in `agentopia-super-rag/evaluation/datasets/`. Each question has labeled relevant document IDs with relevance grades (1-3).

**Eval runner**: `PYTHONPATH=src python evaluation/phase1b_baseline.py --scope {scope}`

**Frequency**: Run after any retrieval pipeline change (chunking strategy, embedding model, ingest path). Run weekly as regression check.

### Baseline (Current — Production)

From `agentopia-super-rag/docs/evaluation.md`:

| Metric | Value | Date |
|---|---|---|
| nDCG@5 | 0.925 | 2026-04-09 (post W1) |
| MRR | 0.96 | 2026-04-09 |
| P@5 | 0.84 | 2026-04-09 |
| R@5 | 1.0 | 2026-04-09 |

This is the **floor**. No migration phase or retrieval change may land below this baseline.

### SLOs

| Metric | Floor (never breach) | Target | Scope |
|---|---|---|---|
| nDCG@5 | ≥ 0.90 | ≥ 0.95 | Per scope, all production scopes |
| MRR | ≥ 0.93 | ≥ 0.96 | Per scope |
| P@5 | ≥ 0.80 | ≥ 0.85 | Per scope |
| R@5 | ≥ 0.95 | 1.0 | Per scope |

**Floor vs target**: The floor (≥ 0.90) is a hard regression guard — breaching it requires immediate rollback. The target (≥ 0.95) is aspirational and is pursued incrementally via Phase 5 retrieval improvements only when evidence supports them.

**Pathway migration SLO**: During Phase 3 pilot, the floor is relaxed to nDCG@5 ≥ 0.90 (Pathway pipeline validation on one scope). After Phase 4 (all scopes migrated), the full floor (≥ 0.90) and production baseline (0.925) apply to all scopes. The ≥ 0.95 target is not a Phase 4 requirement — it is a post-migration improvement goal.

### Promotion Gates

- **Chunking strategy change**: nDCG@5 must not regress more than 0.01 from baseline
- **Retrieval mode change** (query expansion, HyDE, reranking): nDCG@5 must improve by ≥ 0.02 AND latency within approved budget
- **Embedding model change**: all scopes must maintain nDCG@5 ≥ 0.90 (0.025 margin)
- **Pathway ingest path change**: same as embedding model change gate

### Breach Response

- **nDCG@5 drops below 0.90 on any scope**: Immediate rollback of the most recent ingest or retrieval change. Root cause analysis required before re-applying.
- **nDCG@5 between 0.90 and 0.95**: Non-critical. Schedule fix within 2 sprints. Do not deploy further changes to that scope until resolved.

---

## 3. Freshness

### Definition

**Freshness lag** = time from source document change (S3 mtime, GitHub commit timestamp, Confluence last_modified) to the time the updated chunks are `status=active` in Qdrant.

This is the primary new SLO introduced by the Pathway initiative. It does not exist in the current system (batch ingest has no automated freshness mechanism).

### Measurement

**Pathway API**: `DocumentStore.statistics_query()` returns per-document `last_modification_time` and `last_indexing_time`.

```python
stats = document_store.statistics_query()
for doc_id, doc_stats in stats.items():
    lag = doc_stats["last_indexing_time"] - doc_stats["last_modification_time"]
    # emit as metric: pathway_indexing_lag_seconds{doc_id=..., scope=...}
```

**Prometheus metric**: `pathway_indexing_lag_seconds` (histogram, labels: `scope`, `source_type`)

**Reference**: Pathway `statistics_query()` API, https://pathway.com/developers/api-docs/pathway-xpacks-llm/document_store

**Measurement frequency**: Scraped by Prometheus every 30 seconds (matching Pathway poll interval).

### SLOs

| SLO | Target | Notes |
|---|---|---|
| Freshness lag p50 | ≤ 90 seconds | 30s poll + ~60s processing time |
| Freshness lag p95 | ≤ 180 seconds | For large documents or high-cardinality changes |
| Freshness lag maximum | ≤ 300 seconds | Breach if any single document exceeds 5 minutes |
| Stale scope count | ≤ 0 scopes with lag > 600s | Alert threshold: any scope consistently above 10 minutes |

**Manual upload SLO**: Operator-uploaded documents must be indexed within 2 minutes of S3 staging write. The same lag metric applies. This assumes a 30s poll cycle — the actual achieved lag will be established by the Phase 3 freshness measurement.

**Note on freshness targets**: The ≤ 90s p50 and ≤ 180s p95 targets are design targets derived from the 30s poll interval plus estimated processing time. They have not been measured in production. These numbers must be validated against P0.2 and P3.5 measurements and adjusted if the actual system does not meet them.

### Baseline

Before Pathway, freshness is not measured automatically. The effective freshness lag equals operator response time — the gap between a source change and the next manual ingest trigger. Expected range: hours to days.

Baseline value: recorded in `agentopia-knowledge-ingest/docs/freshness-baseline.md` after P0.2 measurement.

### Breach Response

| Condition | Response |
|---|---|
| p95 lag > 180s for >30 minutes | Check Pathway pod logs. Verify S3 connector is polling. Check embedding service latency (OpenRouter). |
| Any scope > 600s consistently | Check `pathway_last_successful_poll_timestamp` metric. If poll is stale, check S3 credentials and bucket access. Restart Pathway pod if stuck. |
| Pathway pod crash loop | Scale to 0, investigate state PVC. Follow rollback path 4.2 in `migration-plan.md`. |

---

## 4. Contamination

### Definition

**Knowledge contamination** occurs when retrieval returns content from the Operational State Plane (live workflow status, bot runtime config, current tool inventory) through the Knowledge Plane (Qdrant).

**Memory contamination** occurs when user-session memory facts (from the Memory Plane) appear in knowledge retrieval results.

Both are architectural violations. They indicate that plane boundaries have been breached in either the ingest pipeline (operational state was indexed into Qdrant knowledge collections) or the retrieval routing (wrong plugin fired).

### Measurement

**Contamination test set**: 10 queries designed to return operational state if contamination exists:
- "What is the current workflow status for bot X?"
- "What is the current SOUL configuration for the SA bot?"
- "List all tools available to the gateway bot"
- "What Temporal activities are pending?"
- "What is the relay token for bot Y?"

If knowledge-retrieval returns chunks for these queries, contamination has occurred.

**Method**: Run contamination test set against `GET /api/v1/knowledge/search` directly (bypassing the routing layer). If any relevant chunks are returned, the ingest pipeline has indexed operational state.

**Eval script**: `agentopia-super-rag/evaluation/contamination-check.py`

### SLO

| SLO | Target |
|---|---|
| Operational state contamination | 0 chunks returned for contamination test set |
| Memory contamination | 0 user-session facts in knowledge collections |

**This SLO is binary: 0 tolerance. Any contamination is a P0 incident.**

### Pathway Contamination Risk

The Pathway pipeline reads from configured sources (S3, GitHub, Confluence). It must NOT be connected to:
- Temporal history APIs
- bot-config-api config endpoints
- Session JSONL storage
- Any live operational state source

Source connection configuration is the contamination prevention control. Pathway connector source list is reviewed at every `agentopia-infra` deploy that touches Pathway config.

### Breach Response

1. Identify which ingest run introduced the contaminating chunks (check `ingested_at` payload in Qdrant)
2. Delete contaminating chunks from Qdrant: `DELETE /api/v1/knowledge/{scope}/documents/{source}`
3. Identify the source connection that fed operational state into Pathway (check Pathway connector config)
4. Remove the incorrect source connection
5. File post-mortem with root cause and prevention

---

## 5. Latency

### Definition

**End-to-end retrieval latency** = time from `knowledge-retrieval` plugin receiving the user query to injecting the `<domain-knowledge>` block into the effective prompt.

This includes:
1. Embed query (OpenRouter `text-embedding-3-small`)
2. Qdrant vector search (scope-filtered, top-5)
3. Build `<domain-knowledge>` block (token budget + confidence filter)

Current retrieval timeout budget: 5000ms (configurable via `retrieveTimeoutMs` in the knowledge-retrieval plugin).

### Measurement

**Prometheus metric**: `knowledge_retrieval_latency_seconds` (histogram, labels: `scope`, `result_count`)

**Components**:
- `knowledge_embed_latency_seconds` — embedding time
- `knowledge_qdrant_search_latency_seconds` — Qdrant search time
- `knowledge_context_build_latency_seconds` — context assembly time

### SLOs

| Metric | Target | Breach threshold |
|---|---|---|
| Retrieval latency p50 | ≤ 800ms | — |
| Retrieval latency p95 | ≤ 2000ms | > 3000ms |
| Retrieval latency p99 | ≤ 3000ms | > 4000ms |
| Retrieval timeout rate | ≤ 0.5% of queries | > 1% |

**With hybrid retrieval (P4)**: Latency budget expands to p95 ≤ 2500ms (BM25 + RRF fusion overhead).

**Reference**: Microsoft RAG best practices — "Retrieval latency is a hard constraint; build retrieval budget before calling LLM" — https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview

### Breach Response

- **p95 > 3000ms**: Check OpenRouter embedding latency. If spike, consider embedding result cache (Redis, TTL=5min, key=hash(query+model)).
- **Timeout rate > 1%**: Check Qdrant health. If Qdrant is degraded, knowledge-retrieval returns empty (non-blocking) and bot responds without knowledge context. This is the existing fallback behavior.
- **Pathway does not affect query latency** — Pathway writes to Qdrant asynchronously. Query latency is purely Qdrant search + embed time.

---

## 6. SLO Summary Table

| Dimension | SLO | Measurement | Frequency |
|---|---|---|---|
| Route correctness (overall) | ≥ 95% | Labeled query set (60 queries, 10 per family) | Weekly + after routing changes |
| Route correctness (per family) | ≤ 5% misroute per family; no family > 10% | Same labeled set | Same |
| knowledge-retrieval on operational families | ≤ 2% | Same labeled set | Same |
| nDCG@5 | ≥ 0.90 floor (hard); 0.925 preserve; ≥ 0.95 target (aspirational) | Golden question sets per scope | After each retrieval or ingest change; weekly regression |
| MRR | ≥ 0.93 floor, ≥ 0.96 target | Golden question sets | Same as nDCG@5 |
| Freshness lag p50 | ≤ 90s (design target — validate at P3.5) | `pathway_indexing_lag_seconds` Prometheus | 30s scrape |
| Freshness lag p95 | ≤ 180s (design target — validate at P3.5) | Same | Same |
| Freshness lag max | ≤ 300s | Same | Same |
| Contamination | 0 (binary, P0 incident if breached) | Contamination test set (10 queries) | Before each Pathway connector change; weekly |
| Retrieval latency p95 | ≤ 2000ms | `knowledge_retrieval_latency_seconds` | Continuous |
| Retrieval timeout rate | ≤ 0.5% | Timeout counter / total query count | Continuous |

---

## 7. Grafana Alerting

### Alert Rules

```yaml
# Retrieval quality regression
- alert: KnowledgeRetrievalQualityRegression
  expr: knowledge_ndcg5_score < 0.90
  for: 1h
  labels:
    severity: critical
  annotations:
    summary: "nDCG@5 below floor (0.90) for scope {{ $labels.scope }}"

# Freshness SLO breach
- alert: PathwayFreshnessLagHigh
  expr: histogram_quantile(0.95, pathway_indexing_lag_seconds_bucket) > 180
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Pathway freshness p95 lag > 180s"

- alert: PathwayFreshnessLagCritical
  expr: pathway_indexing_lag_seconds_max > 300
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Pathway freshness max lag > 300s — SLO breach"

# Contamination detection (triggered by eval script, not Prometheus)
# eval/contamination-check.py exits non-zero if contamination detected → CI alert

# Retrieval latency
- alert: KnowledgeRetrievalLatencyHigh
  expr: histogram_quantile(0.95, knowledge_retrieval_latency_seconds_bucket) > 3
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Knowledge retrieval p95 latency > 3s"
```

### Dashboard Panels

1. **Freshness status** — per-scope freshness lag (p50, p95, max), color-coded by SLO compliance
2. **Retrieval quality trend** — nDCG@5 over time per scope (weekly eval runs)
3. **Route correctness** — misroute rate trend (weekly labeled query set results)
4. **Latency profile** — retrieval p50/p95/p99 trend + timeout rate
5. **Contamination status** — pass/fail per scope (contamination check results)
6. **Pathway event throughput** — inserts/updates/deletes per minute by source type

---

## 8. Evaluation Artifacts Location

| Artifact | Location |
|---|---|
| Golden question sets | `agentopia-super-rag/evaluation/datasets/` |
| Retrieval eval runners | `agentopia-super-rag/evaluation/*.py` |
| Eval results | `agentopia-super-rag/evaluation/results/` |
| Contamination test set | `agentopia-super-rag/evaluation/contamination-check.py` |
| Route correctness eval | `agentopia-protocol/evaluation/route-correctness.py` |
| Freshness baseline doc | `agentopia-knowledge-ingest/docs/freshness-baseline.md` |
| Pathway freshness metrics | Prometheus `pathway_indexing_lag_seconds` histogram |
| Pre-Pathway retrieval baseline | `agentopia-super-rag/evaluation/results/baseline-pre-pathway-{date}.json` |
