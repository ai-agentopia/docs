---
title: "Agentopia RAG — Implementation Plan"
status: "ACTIVE"
---

# RAG Implementation Plan

## Guiding Principles

1. **Measure first** — every phase starts with a baseline and ends with a gate measurement. No phase ships without eval evidence.
2. **Two parallel workstreams** — control-plane routing and knowledge-plane streaming are independent. Phase ordering is driven by risk and dependency, not by which workstream is more interesting.
3. **Current state is the starting point** — where current behavior differs from target behavior, that difference is explicitly called out and becomes implementation work.
4. **Single publisher per scope** — once a Qdrant scope is assigned to the chosen ingest component, only that component writes to it. No dual-write.
5. **ArgoCD for all deployments** — no manual helm installs. Every new component goes through a Helm chart in `agentopia-infra`.

> **Knowledge data plane component status**: Phases 3–7 of this plan are written with Pathway as the current preferred ingest component. This reflects the candidate evaluation outcome at the time of writing. **The final component choice is confirmed at the P0 architecture review gate** — if the mixed-architecture analysis or P3 pilot results change the recommendation, Phases 3–7 will be updated accordingly. The control-plane phases (1–2) are independent of this choice.

---

## Phase Map

```
Phase 0: Baselines + Architecture Lock
    │  gate: baselines documented, architecture approved
    ▼
Phase 1: Control-Plane Routing  ────────────────────────────────┐
    │  gate: misroute rate ≤ 5% across all families              │
    ▼                                                            │ parallel
Phase 2: Operational State Plane  ──────────────────────────────┘
    │  gate: operational_state and planning_readiness
    │        fully covered by governance-bridge
    ▼
Phase 3: Pathway Pilot (Knowledge Plane)
    │  gate: nDCG@5 ≥ 0.90 on pilot scope, freshness lag confirmed
    ▼
Phase 4: Pathway Primary + Multi-Source
    │  gate: all scopes migrated, nDCG@5 ≥ 0.925, freshness SLO met
    ▼
Phase 5: Retrieval Quality Upgrades  (conditional — evidence-gated)
    │  gate: ≥ 3% nDCG@5 improvement required to proceed
    ▼
Phase 6: Observability + SLO Enforcement  (parallel from Phase 3)
    │  gate: all SLOs defined and measurable in Grafana
    ▼
Phase 7: Production Cutover + Rollback Verification
    │  gate: legacy ingest frozen, rollback drill passed
    ▼
  RAG_ARCHITECTURE_DOC_READY
```

Phases 1 and 2 can run in parallel. Phase 6 can start alongside Phase 3.

---

## Phase 0: Baselines + Architecture Lock

**Purpose**: Establish the measurement floor that all subsequent phases must not breach. Lock the architecture design before implementation begins.

**Why first**: Implementation decisions in Phases 1–4 cannot be evaluated without baseline measurements. Proceeding without baselines means no ability to detect regressions.

### P0.1 — Retrieval quality baseline

Snapshot the current production retrieval quality before any pipeline changes. This becomes the floor: no phase may land below it.

| Metric | Current value |
|---|---|
| nDCG@5 | 0.925 |
| MRR | 0.96 |
| P@5 | 0.84 |
| R@5 | 1.0 |

**Deliverable**: Baseline snapshot committed to `agentopia-super-rag/evaluation/results/baseline-pre-initiative-{date}.json`.

**Owner**: agentopia-super-rag team

### P0.2 — Freshness baseline

Measure the current knowledge freshness gap: time from source document change to Qdrant chunk update.

The current mechanism is fully manual. The effective lag equals operator response time. Measurement method:
1. Record S3 object `LastModified` for 10 test documents
2. Record operator upload timestamp for each
3. Record Qdrant `ingested_at` chunk payload for each
4. Compute `ingest_delay = ingested_at - LastModified`

**Deliverable**: Freshness baseline documented in `agentopia-knowledge-ingest/docs/freshness-baseline.md`.

**Owner**: agentopia-knowledge-ingest team

### P0.3 — Route correctness baseline

Measure what fraction of queries from each family are correctly routed to the appropriate plane. This must cover all six families, not only governance queries.

**Method**: Run a labeled query set of 60 queries (10 per family) through the gateway plugin chain. Log which plugins fire per query. Count misroutes per family.

**Expected findings**: `operational_state` and `planning_readiness` families will have the highest misroute rate given known gaps in the current keyword matching. `runtime_meta` and `general_chat` may also misroute since no explicit handling exists today.

**Deliverable**: Route correctness baseline per family documented in `docs/rag/evals-and-slos.md` baseline section.

**Owner**: Gateway/protocol team

### P0.4 — Architecture approval

The architecture documents in `docs/rag/` must be reviewed and approved by the CTO before Phase 1 implementation begins. This ensures the query family model, plane separation, and phase structure are correct before code is written.

**Deliverable**: Architecture approved (GitHub PR to `docs/rag/` merged, or explicit sign-off recorded in `docs/rag/decision-log.md`).

### P0 Exit Gate

- [ ] Retrieval baseline snapshotted and committed
- [ ] Freshness baseline documented
- [ ] Route correctness baseline per family documented
- [ ] Architecture review complete

---

## Phase 1: Control-Plane Routing Hardening

**Purpose**: Implement formal query family classification so every query reaches its correct source of truth. This is the highest-priority implementation work because routing errors corrupt the system's answer quality across all families.

**Current state**: Heuristic keyword matching only. Known coverage gaps across all families.

**Target state**: Intent router classifies each query into one of six families before the plugin chain runs. Plugin skip patterns fully cover their respective non-target families.

### P1.1 — Define and document query family patterns

Before writing code, document the classification rules for each family:
- What query forms belong to each family
- What signals distinguish overlapping families (e.g., `planning_readiness` vs `knowledge_query` about a process)
- What the confidence threshold is for each family classification — below threshold, the query must be assigned `unknown_route`
- How `unknown_route` is handled: no retrieval triggered from any plane; LLM responds from general context only

**Uncertain classification policy**: when the intent router cannot classify a query with sufficient confidence, it must assign `unknown_route`. Defaulting uncertain queries to `knowledge_query` is explicitly prohibited — it recreates the current failure mode where operational and planning queries enter KB retrieval and surface irrelevant or misleading results.

This is a design document exercise, not code. Output is a specification used to implement P1.2 and P1.3.

**Deliverable**: Query family pattern specification in `agentopia-protocol/docs/intent-routing-spec.md`.

**Owner**: Gateway/protocol team

### P1.2 — Extend governance-bridge coverage

Extend the governance-bridge keyword patterns to cover the full `operational_state` and `planning_readiness` families as defined in P1.1.

**Current behavior**: Fixed keyword regex. Misses plural forms, paraphrase variations, and non-English intent expressions.

**Target behavior**: Pattern set covers the query forms documented in P1.1. Coverage is verified against the P0.3 labeled query set.

**Coordination signal**: Ensure `event.coordination.resolvedBy = ["governance-bridge"]` is set whenever governance-bridge resolves a query. This is the suppression signal that prevents knowledge-retrieval from firing on resolved queries.

**Owner**: Gateway/protocol team (agentopia-protocol)

### P1.3 — Extend NON_KB_PATTERNS and NON_MEMORY_META_PATTERNS

Extend the exclusion pattern lists in knowledge-retrieval and mem0-api to match all families that should not trigger those plugins:

- `NON_KB_PATTERNS` (knowledge-retrieval): must exclude `operational_state`, `planning_readiness`, `runtime_meta` families
- `NON_MEMORY_META_PATTERNS` (mem0-api): must exclude `operational_state`, `planning_readiness`, `runtime_meta` families

These patterns are the secondary suppression mechanism (primary is `event.coordination.resolvedBy`). Both mechanisms must be present for defense-in-depth.

**Owner**: Gateway/protocol team (agentopia-protocol)

### P1.4 — Route correctness eval gate

After P1.2 and P1.3 are deployed, run the P0.3 labeled query set again. Compute misroute rate per family.

**Gate**: ≤ 5% misroute rate across all families. Per-family floor: no family may have > 10% misroute rate.

**Iteration limit**: If gate fails, extend patterns and re-run. Maximum two iteration rounds. If the gate still fails after two rounds, the heuristic approach is insufficient — see P1.5.

**Owner**: Gateway/protocol team

### P1.5 — Classifier upgrade (conditional — triggered only if P1.4 fails twice)

Heuristic routing is the Phase 1 initial implementation. It is a transitional mechanism, not the intended long-term architecture. If heuristic routing cannot achieve the route-correctness gate after two iteration rounds, the architecture must advance to a confidence-scored classifier.

**Classifier design requirements**:
- Assigns a confidence score per family classification
- Queries below the confidence threshold are assigned `unknown_route` (not force-assigned to any default family)
- Must not add more than 200ms p95 latency to the gateway request path
- Must be implemented as a gateway plugin — not an external service call in the hot path

**Classifier options** (to be evaluated if P1.5 is triggered):
- Small local model (e.g., distilBERT fine-tuned on the P0.3 labeled set) — lower latency, requires training infrastructure
- LLM prompt-based classification (single fast LLM call per query) — higher latency, simpler to deploy initially
- Embedding similarity to family prototype queries — no training required, may not generalize

The choice between options is not prescribed here. It is decided at the point P1.5 is triggered, based on the specific failure modes observed in P1.4 evaluation output.

**Owner**: Gateway/protocol team
**Trigger**: P1.4 gate fails after two iteration rounds

### P1 Exit Gate

- [ ] Query family pattern specification approved (including `unknown_route` policy)
- [ ] governance-bridge extended and deployed
- [ ] `NON_KB_PATTERNS` and `NON_MEMORY_META_PATTERNS` extended and deployed
- [ ] `event.coordination.resolvedBy` signal set consistently by governance-bridge
- [ ] Route correctness eval: ≤ 5% misroute overall, no family > 10%
- [ ] If P1.5 triggered: classifier deployed and gate re-verified

---

## Phase 2: Operational State Plane Formalization

**Purpose**: Ensure the Operational State Plane fully covers the `operational_state` and `planning_readiness` families — including tool availability, fallback behavior when tools fail, and grounding rules.

**Note**: Phases 1 and 2 can run in parallel. Phase 2 is more design-heavy; Phase 1 is more pattern-matching implementation.

### P2.1 — Tool inventory audit

Document every tool currently available to governance-bridge:
- What live system each tool queries (Temporal, GitHub, K8s)
- What families of queries each tool handles
- What happens when each tool is unavailable (timeout, API error)

This audit may reveal gaps where a query family is classified correctly but the tool needed to answer it does not exist.

**Deliverable**: Tool inventory documented in `agentopia-protocol/docs/governance-bridge-tools.md`.

**Owner**: Gateway/protocol team

### P2.2 — Fallback behavior for each family

Define what the bot says when the authoritative source for a query family is unavailable:

| Family | Unavailability scenario | Required fallback |
|---|---|---|
| `operational_state` | Temporal or K8s API unreachable | Acknowledge; do not guess. E.g., "I can't reach the deployment system right now." |
| `planning_readiness` | GitHub API rate-limited or unreachable | Acknowledge; do not estimate from documents. |
| `knowledge_query` | Qdrant returns no results above threshold | State explicitly no matching document was found. Do not fabricate. |
| `memory_query` | mem0-api returns empty | State no relevant memory found. Proceed from general context. |

**Target**: Grounding behavior is part of the SOUL.md system prompt or enforced via the answer-composition step in the gateway. The bot must not fabricate live state.

**Owner**: Gateway/protocol team

### P2.3 — Answer composition verification

Verify that the gateway assembles the final context correctly per query family:
- `operational_state`: context = tool call result only. No knowledge-plane injection.
- `planning_readiness`: context = tool call result only. No knowledge-plane injection.
- `knowledge_query`: context = knowledge chunks only. No operational state.
- `memory_query`: context = memory recall only (or combined with knowledge if explicitly mixed family).
- `runtime_meta`: context = system prompt only.

**Method**: Run labeled queries from each family and inspect the `effectivePrompt` assembled by the gateway before it reaches the LLM.

**Owner**: Gateway/protocol team

### P2 Exit Gate

- [ ] Tool inventory documented
- [ ] Fallback behavior defined and verified for each family
- [ ] Answer composition verified per family via prompt inspection
- [ ] No operational state in knowledge-plane context injection confirmed

---

## Phase 3: Pathway Pilot (Knowledge Plane)

**Purpose**: Prove that Pathway can ingest into Qdrant correctly, maintain chunk quality, and propagate deletes — on a single scope in the dev environment.

**Scope**: One knowledge scope (`{client_id}/api-docs`) on dev. S3 connector only.

### P3.1 — Pathway deployment

Deploy Pathway as a new Kubernetes Deployment via `agentopia-infra`:

```
agentopia-infra/charts/agentopia-base/templates/
├── pathway-pipeline-deployment.yaml
├── pathway-pipeline-configmap.yaml
├── pathway-pipeline-pvc.yaml          ← state persistence
└── pathway-pipeline-secrets.yaml      ← S3 credentials, OpenRouter key, Qdrant URL
```

ArgoCD manages the deployment. No manual helm installs.

**Owner**: agentopia-infra team

### P3.2 — S3 connector and pilot scope setup

Configure Pathway S3 connector for the pilot scope. Set `scope_ingest_mode: pathway` for the pilot scope in bot-config-api. This makes Pathway the exclusive writer to that scope's Qdrant collection. The legacy ingest path is blocked for this scope.

**Single-publisher enforcement**: `scope_ingest_mode: pathway` triggers a guard in the upload API and orchestrator that returns 409 Conflict if called for a Pathway-managed scope. No dual-write is possible.

**Owner**: knowledge-ingest team

### P3.3 — Chunk quality validation

After Pathway indexes the pilot scope, spot-check 20 chunks against what the legacy path would have produced. Verify:
- Chunk boundaries are correct (especially for markdown-aware strategy)
- Metadata fields (`section_path`, `source`, `document_id`, `status`) are populated
- `status=active` on all current chunks

**Owner**: agentopia-super-rag team

### P3.4 — Delete propagation test

1. Upload a test document to S3
2. Confirm chunks appear in Qdrant with `status=active`
3. Delete the document from S3
4. Within 2× poll interval (≤ 60 seconds target): confirm chunks removed from Qdrant

**Owner**: knowledge-ingest team

### P3.5 — Freshness measurement

Measure indexing lag using `statistics_query()` for 10 documents updated in S3. Compare against freshness baseline from P0.2. The Pathway result should be significantly lower.

**Owner**: knowledge-ingest team

### P3.6 — Retrieval eval on pilot scope

Run the golden question set for the pilot scope:

```bash
PYTHONPATH=src python evaluation/phase1b_baseline.py --scope {pilot_scope}
```

**Gate**: nDCG@5 ≥ 0.90 on pilot scope.

**Owner**: agentopia-super-rag team

### P3 Exit Gate

- [ ] Pathway pod running stable for 7 days on pilot scope
- [ ] nDCG@5 ≥ 0.90 on pilot scope
- [ ] Chunk quality spot-check passed
- [ ] Delete propagation confirmed within 60 seconds
- [ ] Freshness lag documented (compare against P0.2 baseline)
- [ ] No regression on non-pilot scopes

---

## Phase 4: Pathway Primary + Multi-Source Ingestion

**Purpose**: Migrate all scopes from the legacy batch ingest to Pathway. Expand source connectors to GitHub and Confluence.

### P4.1 — Scope-by-scope migration

For each remaining scope:
1. Set `scope_ingest_mode: pathway` in bot-config-api
2. Wait for Pathway to fully index all existing documents in scope
3. Run retrieval eval: nDCG@5 must match or exceed pre-migration baseline for that scope
4. Confirm delete propagation
5. Mark scope as `PATHWAY_ONLY` in migration tracking

**Batch limit**: Migrate at most 3 scopes per day. Monitor for 24 hours before continuing.

See `migration-plan.md` for the full cutover sequence and rollback procedures.

**Owner**: knowledge-ingest team

### P4.2 — Connector migration

Replace the three legacy connectors in `agentopia-knowledge-ingest/src/connectors/`:

| Legacy connector | Replaced by |
|---|---|
| `openrag_s3.py` / `aws_s3_wrapper.py` | Pathway S3 connector |
| `openrag_gdrive.py` / `google_drive_wrapper.py` | Pathway Google Drive connector — verify API shape from Pathway connector registry |
| `openrag_onedrive.py` / `onedrive_wrapper.py` | Pathway SharePoint/OneDrive connector — verify availability; fallback: Microsoft Graph polling wrapper |

Existing connector code is kept (frozen, not running) until all scopes are confirmed on Pathway and the P4 exit gate passes.

### P4.3 — GitHub connector

Add GitHub as a Pathway source for scopes that include documentation from GitHub repos (wikis, ADR directories, markdown docs):

```python
# Verify API shape against current Pathway connector registry before implementation
source_github = pw.io.github.read(
    repo=GITHUB_REPO,
    path_pattern="docs/**/*.md",
    poll_interval=PATHWAY_POLL_INTERVAL_SECS,
    token=GITHUB_TOKEN,
)
```

**Reference**: Pathway GitHub connector, https://pathway.com/developers/api-docs/pathway-io/github — verify current availability.

### P4.4 — Confluence connector

Pathway does not have a first-party Confluence connector. Two options:

**Option A** (recommended for speed): Confluence → scheduled S3 export → Pathway S3 connector picks up. Adds export latency beyond the 2-minute freshness target.

**Option B**: Custom Pathway connector wrapping the Confluence REST API (`/wiki/rest/api/content`) via `pw.io.python.read()`. Achieves sub-2-minute freshness but requires custom connector code and ongoing maintenance.

**Decision required**: CTO selects Option A or B. If Option B, a proof-of-concept is required before committing to it as the production path.

### P4.5 — Normalizer and orchestrator deprecation

After all scopes are on Pathway and the P4 exit gate passes, mark the following as deprecated in the source:
- `agentopia-knowledge-ingest/src/normalizer/` — Pathway handles format parsing
- `agentopia-knowledge-ingest/src/orchestrator.py` — Pathway pipeline is the orchestrator

These are not deleted at Phase 4. Deletion is a Phase 7 cleanup task.

### P4 Exit Gate

- [ ] All scopes migrated to Pathway
- [ ] nDCG@5 ≥ 0.925 across all scopes (full production baseline preserved)
- [ ] Freshness SLO met across all scopes
- [ ] Manual upload path end-to-end via Pathway (upload API → S3 → Pathway → Qdrant)
- [ ] GitHub source live for at least one scope
- [ ] Confluence source live (Option A or B confirmed)
- [ ] Legacy connector code marked deprecated

---

## Phase 5: Retrieval Quality Upgrades

**Condition**: Phase 5 is gated on evidence. It must not be started on schedule — it starts only when new evaluation evidence shows ≥ 3% nDCG@5 improvement is achievable for a specific technique.

Hybrid retrieval (W2) was previously evaluated and frozen due to insufficient improvement on the pilot corpus. Reranking (W4) was evaluated and resulted in nDCG@5 regression (−0.1238). These outcomes stand until new evidence contradicts them.

### P5.1 — Hybrid retrieval (W2 reopen)

**Condition to reopen**: New evaluation on a representative corpus shows ≥ 3% nDCG@5 improvement over the dense-only baseline. Requires Qdrant ≥ 1.7 for sparse vector support.

If the condition is met:
- Pathway generates sparse BM25 vectors alongside dense embeddings in one pipeline pass
- Qdrant serves both; retrieval path implements RRF fusion
- Gate: nDCG@5 improvement ≥ 3%, latency p95 ≤ 2.5s

### P5.2 — Reranking (re-evaluation)

**Condition to reopen**: New evaluation with a different reranker (e.g., local ONNX cross-encoder rather than LLM listwise) shows nDCG@5 improvement ≥ 2% with latency increase within approved budget.

The W4 outcome (gpt-4o-mini listwise reranking: −0.1238 nDCG@5) is specific to that configuration. A cross-encoder reranker may behave differently. Requires new evaluation run before any implementation.

### P5 Exit Gate

- [ ] New evaluation evidence collected and documented
- [ ] Gate criteria met for the specific technique
- [ ] Latency budget approved before implementation begins

---

## Phase 6: Observability + SLO Enforcement

**Start**: Can begin alongside Phase 3. Does not block other phases but is required before declaring the system production-ready.

### P6.1 — Pathway metrics

Expose Pathway pipeline metrics to Prometheus:

| Metric | Description |
|---|---|
| `pathway_indexing_lag_seconds` | `last_indexing_time - last_modification_time` per document |
| `pathway_events_total{type=insert\|update\|delete}` | Event throughput |
| `pathway_pipeline_restart_total` | Fault tolerance signal |
| `pathway_last_successful_poll_timestamp` | Connector health |

**Reference**: Pathway OpenTelemetry integration, https://pathway.com/developers/user-guide/development/monitoring-and-debugging

### P6.2 — Route correctness automated eval

Implement the route correctness eval as a scheduled CI job. Run the labeled query set (60 queries, 10 per family) weekly and on every gateway plugin change.

**Deliverable**: `agentopia-protocol/evaluation/route-correctness.py` running in CI.

### P6.3 — Retrieval quality regression gate

Retrieval eval must run automatically after any ingest pipeline change. Gate: nDCG@5 must not drop below the P0.1 baseline floor.

**Deliverable**: Eval runner integrated into CI on `agentopia-super-rag`.

### P6.4 — Contamination check

Run the contamination test set before each Pathway connector configuration change and weekly:

**Deliverable**: `agentopia-super-rag/evaluation/contamination-check.py` running in CI.

### P6.5 — Grafana dashboard

RAG-specific dashboard:
- Pathway: indexing lag p50/p95 trend, event throughput, stale scope count
- Retrieval: nDCG@5 trend per scope, latency p50/p95/p99
- Routing: misroute rate per family trend (weekly)
- Contamination: pass/fail per scope

### P6 Exit Gate

- [ ] Pathway metrics in Prometheus
- [ ] Route correctness eval in CI
- [ ] Retrieval quality regression gate in CI
- [ ] Contamination check in CI
- [ ] Grafana dashboard deployed with all panels

---

## Phase 7: Production Cutover + Rollback Verification

**Purpose**: Freeze legacy ingest code, verify rollback procedures work, and declare the system production-ready.

### P7.1 — Legacy ingest freeze

With all scopes on Pathway (P4 complete) and observability in place (P6 complete):
1. Set `LEGACY_INGEST_ENABLED=false` in knowledge-ingest
2. Upload API writes to S3 staging only (no orchestrator call)
3. Orchestrator module: frozen in place, not running
4. `super-rag /ingest` endpoint: rate-limited (1 req/min), logs warning on each call

### P7.2 — Rollback drill

Before legacy code is deleted, run a full rollback drill:
1. Set `scope_ingest_mode: legacy` for one scope
2. Re-enable legacy ingest for that scope
3. Verify legacy path works end-to-end: upload → orchestrator → Qdrant
4. Verify retrieval quality unchanged after rollback
5. Re-set to Pathway after drill completes

**This is required.** The rollback path must be verified while it is still possible to verify it.

### P7.3 — Legacy code removal (30-day hold)

After 30 days of stable operation post-P7.1, with no rollbacks executed:
- PR: remove `agentopia-knowledge-ingest/src/connectors/openrag_*.py`
- PR: remove `agentopia-knowledge-ingest/src/normalizer/` and `orchestrator.py`
- PR: remove `scope_ingest_mode` flag (all scopes are Pathway; flag is now vestigial)

Each removal is a separate PR. No bundled removals.

### P7 Exit Gate

- [ ] Legacy ingest frozen (`LEGACY_INGEST_ENABLED=false`)
- [ ] Rollback drill completed and documented
- [ ] 30-day stability window passed
- [ ] Legacy code removal PRs merged
- [ ] Status declared: **RAG_ARCHITECTURE_DOC_READY** if all gates pass

---

## Ownership Summary

| Phase | Owner | Key Dependency |
|---|---|---|
| P0 | All teams (parallel) | None |
| P1 | Gateway/protocol team | P0.3 complete |
| P2 | Gateway/protocol team | P0.4 complete; can parallel with P1 |
| P3 | knowledge-ingest + super-rag + infra | P0 complete; Pathway Helm in agentopia-infra |
| P4 | knowledge-ingest team | P3 exit gate |
| P5 | super-rag team | P4 complete + new eval evidence |
| P6 | infra + super-rag + gateway teams | Starts at P3; required before P7 |
| P7 | knowledge-ingest + infra | P4 + P6 complete |

---

## Risk Register

| Risk | Severity | Mitigation |
|---|---|---|
| Pathway connector unavailable for required source (OneDrive, Confluence) | High | Verify connector registry before P3. Use S3 bridge as fallback for Confluence. Wrap Microsoft Graph API for OneDrive if Pathway connector is absent. |
| Chunk quality regression after Pathway normalizer replaces knowledge-ingest normalizer | High | P3/P4 eval gates require nDCG@5 ≥ 0.925 across all scopes. Rollback available per scope. |
| Control-plane routing not achieving ≤ 5% misroute after P1 | Medium | P1.4 gate blocks Phase 2+ until met. Iterate on pattern coverage. LLM-based classifier considered if heuristics prove insufficient. |
| Pathway state PVC failure | Medium | PVC with backup StorageClass. Pathway cold-start re-indexes from source on new PVC. |
| Pathway connector misses a delete event | Medium | Weekly full-sync scan: compare source file list vs Qdrant `document_records`. Tombstone orphans. |
| P5 hybrid retrieval reopened without sufficient evidence | Low | P5 gate is strict: ≥ 3% nDCG@5 delta required before implementation begins. No exceptions. |
| Confluence S3 bridge adds freshness lag beyond SLO | Low | If Option A (S3 bridge) exceeds SLO, escalate to Option B or accept relaxed SLO for Confluence sources specifically. |
