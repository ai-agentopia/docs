---
title: "Agentopia RAG — Migration Plan"
status: "ACTIVE"
---

# Migration Plan: Two-Dimensional Knowledge Platform Migration

## Scope

This document covers the **knowledge-plane migration** across two dimensions:

1. **Ingest path migration** — transition from operator-driven batch ingest to Pathway streaming. This covers pipeline mechanics: source connectors, change detection, delete propagation, freshness SLO.

2. **Service consolidation** — migration from the current split architecture (`agentopia-knowledge-ingest` + `agentopia-super-rag`) to a single consolidated service (`agentopia-rag-platform`). This covers which code moves where, deprecation sequencing, and service ownership transfer.

These dimensions are coupled: `agentopia-rag-platform` is the service that will run the Pathway pipeline. The ingest path migration and the service consolidation are sequenced together, not independently.

Control-plane routing changes (intent router, governance-bridge extension, pattern updates) are covered in `implementation-plan.md` Phases 1–2 and do not interact with this migration.

## Purpose

This document defines the current knowledge-plane state precisely, the consolidation target (`agentopia-rag-platform`), the transitional architecture during co-existence, the cutover sequence, and rollback paths.

---

## 1. Current State (Before Migration)

### 1.1 Ingest Path

```
Operator triggers upload
    │
    ▼
knowledge-ingest Upload API
    │  POST /api/v1/upload (multipart)
    │  validates format, stores to S3 (original + normalized artifacts)
    ▼
knowledge-ingest Orchestrator
    │  normalizer → normalizer → metadata extractor → (sync boundary)
    ▼
super-rag POST /api/v1/knowledge/{scope}/ingest
    │  chunk → embed → Qdrant upsert
    │  PostgreSQL document_records update: status=active
    ▼
Qdrant kb-{scope_hash} collection updated
```

**Freshness gap**: Time from source change to Qdrant update = operator response time. No automated polling. Typical lag: hours to days.

### 1.2 Services and Responsibilities

| Service | Current Role | Migration Target |
|---|---|---|
| `agentopia-knowledge-ingest` | Upload API, S3 raw/normalized artifacts, PostgreSQL document registry, orchestrator, connectors | **Deprecated** — ingest and connector layer moves to `agentopia-rag-platform`; normalizer and orchestrator removed after Phase 7 |
| `agentopia-super-rag` | Chunking, embedding, Qdrant collections, PostgreSQL document_records, retrieval serving | **Migrated** → serving and eval code moves to `agentopia-rag-platform` after Phase 4 |
| `agentopia-rag-platform` | _(does not exist yet)_ | **Target** — knowledge-plane consolidation owner. Owns: Pathway ingest pipeline, upload API, retrieval serving, Qdrant lifecycle, eval framework |
| `agentopia-protocol` (bot-config-api) | Scope bindings, bot CRUD, internal token auth | **Minor** — `scope_ingest_mode` flag added per scope for migration |

### 1.3 Active Connectors

From `agentopia-knowledge-ingest/src/connectors/`:

| Connector | Source | Mechanism | Status |
|---|---|---|---|
| `openrag_s3.py` | AWS S3 | Operator-triggered poll, wraps boto3 | Active |
| `openrag_gdrive.py` | Google Drive | Operator-triggered poll, wraps Drive API | Active |
| `openrag_onedrive.py` | OneDrive/SharePoint | Operator-triggered poll, wraps Graph API | Active |

All three are synchronous poll wrappers invoked by the orchestrator when an operator triggers an ingest job. They do not run continuously.

### 1.4 Current Weaknesses

- **No continuous monitoring** — source changes are not detected without operator action
- **No delete propagation** — if a document is removed from S3, its chunks remain in Qdrant with `status=active` until an operator explicitly deletes them
- **Full re-ingest on update** — any document change triggers re-embedding of all chunks, not just changed sections
- **Connector depth is shallow** — only S3, Google Drive, OneDrive. GitHub, Confluence, Jira are not connected

---

## 1.5 Consolidation Target — agentopia-rag-platform

`agentopia-rag-platform` does not yet exist. It is bootstrapped in Phase RB (see `implementation-plan.md`). The consolidation follows the recommended sequencing from Phase RB.3:

**Recommended sequencing:**

| Phase | Ingest path | Serving path | Ownership |
|---|---|---|---|
| PRB | — | — | Repo bootstrap only; no traffic |
| Phase 3 (pilot) | Pathway runs in `agentopia-rag-platform` | `agentopia-super-rag` unchanged | `agentopia-rag-platform` owns ingest for pilot scope |
| Phase 4 (full migration) | Pathway in `agentopia-rag-platform` for all scopes | `agentopia-super-rag` unchanged | `agentopia-rag-platform` owns ingest for all scopes |
| Post Phase 4 | `agentopia-rag-platform` (stable) | Serving code migrates from `agentopia-super-rag` → `agentopia-rag-platform` | `agentopia-rag-platform` owns both ingest + serving |
| Phase 7 | Legacy code removed | `agentopia-super-rag` deprecated | `agentopia-rag-platform` is sole knowledge-plane owner |

**Deprecation definitions:**
- **Deprecated** = frozen; no new feature work. Maintained for bug fixes and rollback support only.
- **Removed** = decommissioned and archived 30 days after Phase 7 exit gate passes.

**Source of truth during transition:**
- Qdrant collections remain the serving source of truth throughout. Collection names and payload schema do not change.
- The `document_records` PostgreSQL table migrates to `agentopia-rag-platform` ownership during the serving migration (post Phase 4).
- `scope_ingest_mode` flag in `bot-config-api` controls which service writes to each scope. The flag remains until all scopes are fully in `agentopia-rag-platform` and then is removed in Phase 7.3.

---

## 2. Transitional Architecture

During migration, both the legacy ingest path and the Pathway pipeline coexist. This prevents service disruption during the migration window.

### 2.1 Single-Publisher Invariant

**No dual-write into the same Qdrant collection is permitted at any time.**

Each scope is assigned to exactly one ingest path. Once a scope is assigned to Pathway, only Pathway writes to that scope's `kb-{scope_hash}` collection. The legacy path is blocked for that scope.

Enforcement:
- `scope_ingest_mode` flag in bot-config-api determines the ingest path per scope
- When `scope_ingest_mode: pathway`, the upload API returns 409 Conflict if called for a direct-to-Qdrant ingest
- When `scope_ingest_mode: legacy`, Pathway does not poll that scope's S3 prefix
- There is no "observation period" during which both paths write — the transition is instantaneous per scope at the point of flag change

Non-migrated scopes (`scope_ingest_mode: legacy`) remain on the existing path until their individual cutover. They are unaffected by Pathway deployment.

```
scope_ingest_mode:
  {client_id}/api-docs:     "pathway"    # migrated — Pathway is sole writer
  {client_id}/product-docs: "legacy"     # not yet migrated — legacy is sole writer
  default:                  "legacy"     # all unmigrated scopes
```

This flag is managed in `bot-config-api` scope configuration. The upload API and orchestrator check this flag:
- `pathway`: upload to S3 staging bucket → Pathway picks up; orchestrator call blocked
- `legacy`: existing orchestrator flow unchanged; Pathway does not poll

### 2.2 Migration State Machine

```
LEGACY ──────────────────────────────► PATHWAY
         Pilot                          
   scope assigned to Pathway            
   Pathway ingest from S3              
         │                              
         ▼                              
   DUAL_OBSERVE (P1 only)              
   Pathway writes to pilot scope        
   Legacy writes to non-pilot scopes   
   Eval gates run on pilot scope       
         │ P1 gates pass               
         ▼                              
   PATHWAY_PRIMARY (P2)               
   All scopes migrated one-by-one      
   Legacy path disabled per scope      
         │ All scopes migrated         
         ▼                              
   PATHWAY_ONLY                       
   Legacy code frozen, not deleted     
```

### 2.3 Qdrant Collection Ownership

During transition:
- Pathway-owned scopes: Pathway is the only writer. Legacy ingest API returns 409 Conflict if called for a Pathway scope.
- Legacy-owned scopes: existing orchestrator is the only writer. Pathway does not poll these S3 prefixes.

This is enforced by the `scope_ingest_mode` flag checked at the ingest API boundary.

---

## 3. Cutover Sequence

### Phase A — Pilot Scope Cutover (P1)

**Prerequisites**:
- Pathway pod running and healthy
- State PVC bound
- S3 connector configured for pilot scope prefix

**Steps**:

1. Select pilot scope: `{client_id}/api-docs` on dev environment
2. Set `scope_ingest_mode: pathway` for pilot scope in bot-config-api
3. Verify Pathway begins polling S3 (check pod logs: `pathway: new source registered`)
4. Wait one full poll cycle (30 seconds), verify chunks appear in Qdrant
5. Run retrieval eval on pilot scope: `PYTHONPATH=src python evaluation/phase1b_baseline.py --scope {pilot_scope}`
6. Verify nDCG@5 ≥ 0.90

**Rollback trigger**: If eval fails or chunks are incorrect, set `scope_ingest_mode: legacy`. Pathway stops writing. Legacy path resumes.

### Phase B — Scope-by-Scope Migration (P2)

For each remaining scope:

1. Set `scope_ingest_mode: pathway` for the scope
2. Pathway begins polling within one poll cycle
3. Wait for Pathway to fully index all existing documents in scope (check `statistics_query()` — all `last_indexing_time` populated)
4. Run eval: nDCG@5 must not regress
5. Verify delete propagation: remove one test document from S3, confirm Qdrant chunks removed within 60 seconds
6. Mark scope as `PATHWAY_ONLY` in migration tracking

**Batch size**: Migrate at most 3 scopes per day. Observe for 24 hours before continuing.

**Rollback trigger per scope**: Any nDCG@5 regression or eval failure → revert scope to `legacy`. Pathway auto-pauses for that scope.

### Phase C — Legacy Path Freeze (post-P2)

Once all scopes are on Pathway:

1. Set `LEGACY_INGEST_ENABLED=false` environment variable in knowledge-ingest
2. Upload API still accepts files — writes to S3 staging (Pathway picks up) — but does NOT call orchestrator
3. Orchestrator module remains in codebase (frozen, not running)
4. `POST /api/v1/knowledge/{scope}/ingest` in super-rag: rate-limited, logs `WARN: direct ingest bypasses Pathway`

**This is NOT a code deletion.** Legacy code stays frozen for 30 days before removal PR.

### Phase D — Legacy Code Removal (30 days post-Phase C)

After 30-day observation window with no regressions:

1. PR: remove `agentopia-knowledge-ingest/src/connectors/openrag_*.py`
2. PR: remove `agentopia-knowledge-ingest/src/normalizer/` and `orchestrator.py`
3. PR: remove legacy ingest path from super-rag (`/ingest` endpoint — or permanently rate-limit)
4. PR: remove `scope_ingest_mode` flag from bot-config-api (all scopes now Pathway)

Each removal is a separate PR. No bundled removals.

---

## 4. Rollback Paths

### 4.1 Single Scope Rollback

**Trigger**: Eval failure or retrieval regression on a specific scope after Pathway cutover.

```
1. Set scope_ingest_mode: legacy (instant — API flag change)
2. Pathway stops writing to that scope's Qdrant collection
3. Legacy orchestrator resumes for that scope (requires operator trigger)
4. Run full re-ingest via legacy path to ensure Qdrant is current
5. File post-mortem: what did Pathway get wrong?
```

**Recovery time objective**: ≤ 10 minutes (flag change + legacy re-ingest for a single scope)

### 4.2 Full Pathway Rollback

**Trigger**: Pathway pod crashing repeatedly, state corruption, or multiple scope regressions simultaneously.

```
1. Set PATHWAY_ENABLED=false in agentopia-infra (ArgoCD applies)
2. All scopes revert to scope_ingest_mode: legacy
3. Pathway pod scaled to 0 (ArgoCD)
4. Operators trigger re-ingest for any scopes that had pending updates
5. Pathway PVC retained (do NOT delete — state preserved for re-investigation)
6. File incident report
```

**Recovery time objective**: ≤ 30 minutes (ArgoCD sync + operator re-ingest triggers)

### 4.3 State Corruption Rollback

**Trigger**: Pathway state PVC corrupted (pods fail to restart, state replay errors).

```
1. Scale Pathway deployment to 0
2. DO NOT DELETE the PVC — snapshot it for debugging
3. Create a new PVC (pathway-state-pvc-recovery)
4. Deploy Pathway with new PVC — it will cold-start and re-index from connectors
5. Cold-start re-indexes all documents from source systems (no data loss — source is authoritative)
6. Validate all scopes after cold-start completes
```

**Cold-start duration**: Depends on document count. Estimate: 1000 documents × 30s avg = ~8 hours. During cold-start, knowledge retrieval falls back to the last-known Qdrant state (retrieval still works; freshness is paused).

### 4.4 Qdrant Data Recovery

Qdrant data is not the source of truth — the source systems (S3, GitHub, Confluence) are. If Qdrant collection data is corrupted or lost:

```
1. Delete affected Qdrant collection(s)
2. Pathway cold-start (or trigger manual re-index via /ingest emergency path)
3. Qdrant is repopulated from source documents
```

No backup of Qdrant vectors is needed — they are fully reproducible from source documents via the embedding pipeline.

---

## 5. Migration Tracking

Track migration state for each scope in `bot-config-api` scope metadata:

```json
{
  "scope": "joblogic-kb/api-docs",
  "ingest_mode": "pathway",
  "migration_status": "PATHWAY_ONLY",
  "pathway_first_indexed_at": "2026-05-01T10:00:00Z",
  "pathway_last_indexing_time": "2026-05-15T08:30:00Z",
  "legacy_disabled_at": "2026-05-08T00:00:00Z"
}
```

Migration dashboard in Grafana: scope count by `migration_status` (LEGACY, DUAL_OBSERVE, PATHWAY_PRIMARY, PATHWAY_ONLY).

---

## 6. Compatibility Preserved

The following are NOT changed during migration:

| Component | Current value | After migration |
|---|---|---|
| Qdrant collection name format | `kb-{sha256_hex[:16]}` | Unchanged |
| Scope identity format | `{client_id}/{scope_name}` | Unchanged |
| Retrieval API | `GET /api/v1/knowledge/search` | Unchanged |
| Auth model | Internal token + bot bearer | Unchanged |
| Embedding model | `text-embedding-3-small`, 1536d | Unchanged |
| Chunk format in Qdrant | `document_id, scope, version, section_path, status` | Unchanged (Pathway uses same payload schema) |
| Bot-config-api scope bindings | K8s CRD annotations | Unchanged |
| Eval framework | `evaluation/` in super-rag | Unchanged (additional Pathway freshness metrics added) |

Migration is invisible to retrieval consumers (gateway plugins, operator search UI). Only the ingest path changes.
