---
title: "ADR-012: SA Knowledge Base — Ingestion Model and Document Lifecycle"
status: accepted
date: 2026-03-30
decision-makers: CTO
debate: "D5 (#298)"
milestone: "#33 — SA Knowledge Base"
depends-on: "ADR-008 (D1), ADR-009 (D2), ADR-010 (D3), ADR-011 (D4)"
---

# ADR-012: SA Knowledge Base — Ingestion Model and Document Lifecycle

## Context

D4 (ADR-011) established: original document is canonical, latest-wins versioning, tombstone ledger for superseded versions, `ingested_at` + `document_hash` mandatory per chunk, composite chunk identity. D5 must now define the ingestion paths and document lifecycle operations for first release.

### Current ingestion reality (code-verified)

- **Paths**: File upload (multipart POST), webhook (query params). Both unauthenticated.
- **Formats**: PDF, HTML, Markdown, Text, Code (12 extensions mapped)
- **Chunking**: Fixed-size (512 chars/64 overlap), paragraph, code-aware. Semantic: not implemented.
- **Dedup**: MD5 of chunk text. Identical chunks silently skipped.
- **Re-upload**: Changed chunks added alongside old. Old NOT cleaned. No document_hash tracking.
- **Delete**: In-memory only. Qdrant points orphaned. No tombstone.
- **Document metadata**: Only scope-level `KnowledgeScope{name, document_count, chunk_count, last_indexed}`. No per-document records.

## Decision

### 1. Supported Ingestion Paths

**File upload only for first release.** Authenticated (D2), scoped to `{client_id}/{scope_name}` (D1).

- File upload: operator uploads via multipart POST. Format auto-detected from extension.
- Webhook: **deferred from first release**. Current implementation sends content via query params — not production-safe (URL length limits, content in access logs, no body signing). A future webhook must use request body with signed payload contract.
- Bulk import: deferred beyond first release.

### 2. Document Lifecycle Model

```
Upload → Active → (Re-upload → Active [new version]) → Delete
                                                      ↓
                                              Tombstone (metadata only)
```

| State | Description |
|---|---|
| Active | Chunks in Qdrant, searchable, one version per `(scope, source)` |
| Superseded | Previous version replaced by re-upload. Tombstone record retained. Chunk data deleted. |
| Deleted | Operator explicitly deleted. Tombstone record retained. Chunk data deleted. |

Not in first release: draft/approval states, automated lifecycle transitions, "deprecated but searchable" state.

### 3. Latest-Wins Re-Upload Flow

When an operator uploads a file with the same `source` (filename) to the same scope:

1. Compute `document_hash` (SHA-256) of uploaded content
2. Look up existing DocumentRecord for `(scope, source)`
3. **If hash matches**: skip entirely. Return `status=unchanged, chunks_created=0`.
4. **If hash differs** — two-phase replace:
   - **Phase 1 (prepare new)**: Chunk new content, embed, prepare Qdrant points. Old document remains active and searchable throughout this phase. No state is mutated yet.
   - **Phase 2 (commit swap)**: Delete old Qdrant points for `(scope, source)` → upsert new points → create tombstone for old DocumentRecord → update DocumentRecord to new version.

Phase 2 is NOT atomic in a distributed-transaction sense — it involves Qdrant writes + in-memory state. Failure semantics are defined below.

### 3a. Failure Semantics

The old document is authoritative until the new one is fully committed. If any step fails:

| Failure Point | State After Failure | Recovery |
|---|---|---|
| Phase 1 fails (chunking/embedding) | Old document remains active, unchanged. No side effects. | Operator retries upload. |
| Phase 2: Qdrant delete succeeds, upsert fails | Old chunks gone, new chunks not stored. Document has zero chunks. DocumentRecord still points to old version. | #303 must detect and retry upsert. If unrecoverable, operator re-uploads. Scope search returns no results for this source until resolved. |
| Phase 2: Qdrant succeeds, tombstone/record update fails | New chunks in Qdrant, but DocumentRecord is stale. | #303 must treat Qdrant as source of truth for chunk state. Record reconciliation on next access or startup. |

**Key principle**: The old document is not tombstoned until the new content is confirmed in Qdrant. Phase 1 is side-effect-free. Phase 2 failures leave the system in a recoverable state — not silently corrupt.

**What #303 must implement**: Retry logic for Phase 2 partial failures. #307 must prove: no mixed old+new chunks under normal operation, and that partial-failure states are recoverable.

### 4. DocumentRecord Contract

Per-document metadata record (D4 immutable source reference):

| Field | Type | Purpose |
|---|---|---|
| `source` | str | Filename/URL |
| `scope` | str | Knowledge scope (`{client_id}/{scope_name}`) |
| `document_hash` | str | SHA-256 of original content |
| `ingested_at` | float | Unix timestamp of ingestion |
| `chunk_count` | int | Number of chunks |
| `format` | str | DocumentFormat value |
| `status` | str | `active` / `superseded` / `deleted` |
| `superseded_at` | float or null | When replaced |
| `deleted_at` | float or null | When deleted |

Storage durability is an implementation detail for #303. D5 defines the contract.

### 5. Delete Operations

**Delete document**: Removes chunks from Qdrant + in-memory. Creates tombstone (`status=deleted, deleted_at=now`). Invalidates dedup hashes.

**Delete scope**: Deletes all documents in scope (with tombstones). Deletes Qdrant collection. Removes scope metadata.

Both must actually clean Qdrant — the current in-memory-only delete is a bug that #304 must fix.

### 6. Operator Workflow

| Operation | What happens |
|---|---|
| Upload document | Chunk → embed → store → DocumentRecord created (active) |
| Re-upload (same hash) | Skip. Return `unchanged`. |
| Re-upload (different hash) | Phase 1: prepare new (no mutation). Phase 2: swap (delete old → upsert new → tombstone → update record). Old active until committed. |
| Delete document | Tombstone → delete chunks from Qdrant + memory |
| Delete scope | Tombstone all docs → delete Qdrant collection |
| List documents | Show active documents with `source, chunk_count, ingested_at, document_hash` |
| Check staleness | Per-document `ingested_at` enables per-document staleness detection |
| Reindex | Clear dedup cache. Next ingest re-processes. |

### 7. Staleness Handling

- Per-document staleness: `list_documents` includes `ingested_at` per document (new)
- Operator identifies stale documents and decides action
- No automated staleness actions in first release
- Staleness-based search ranking is D7 territory

### 8. Chunking Strategy

First release uses existing strategies unchanged:
- Fixed-size (default: 512 chars, 64 overlap)
- Paragraph-based
- Code-aware

Semantic chunking: defined in enum but not implemented. Deferred.

Chunking strategy evaluation against client sample documents is a #306 / D7 quality concern.

## Alternatives Considered

### Ingestion paths

| Option | Verdict | Reason |
|---|---|---|
| File upload only | Accepted | Production-safe. Operator-controlled. Auth via session. |
| File upload + current webhook | Rejected | Current webhook sends content via query params — URL length limits, content in access logs, no signing. Not production-safe. |
| File upload + redesigned webhook | Deferred | Future release: request body + signed payload. |
| File upload + webhook + bulk | Rejected for first release | Complexity; can add later |

### Lifecycle model

| Option | Verdict | Reason |
|---|---|---|
| No lifecycle tracking | Rejected | Cannot audit, cannot manage, D4 requires metadata |
| Upload/active/delete only | Rejected | No supersession tracking — violates D4 tombstone requirement |
| Upload/active/superseded/deleted | Accepted | Minimal viable lifecycle with D4 tombstone support |
| Full lifecycle with draft/approval | Rejected for first release | No operator workflow for approval gates |

### Re-upload behavior

| Option | Verdict | Reason |
|---|---|---|
| Append (old + new coexist) | Rejected | D4 mandates latest-wins. Mixed chunks = provenance confusion. |
| Replace without hash check | Rejected | Unnecessary re-embedding if content unchanged |
| Latest-wins with hash check + two-phase replace | Accepted | Skip unchanged. Two-phase replace with defined failure semantics. D4 compliant. |

## Consequences

### Downstream constraints

- **D6 (#299)**: External import must create DocumentRecord with `status=active`. Same lifecycle applies.
- **#303**: Must implement DocumentRecord, two-phase replace with retry logic for Phase 2 partial failures, tombstone creation, Qdrant delete methods, per-document staleness in list API.
- **#305**: Operator UI must show upload/re-upload feedback, document list with `ingested_at`, delete confirmation.
- **#307**: Must prove: no mixed old+new chunks under normal operation, partial-failure states are recoverable, tombstone creation on supersede/delete works.

### What D5 does NOT decide

- External vector import contract (D6)
- Answer behavior under stale/conflicting evidence (D7)
- Quality evaluation and chunking strategy assessment (D7)
- Storage durability for DocumentRecord (#303 implementation)
