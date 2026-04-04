---
title: "ADR-011: SA Knowledge Base — Source of Truth and Provenance"
status: accepted
date: 2026-03-30
decision-makers: CTO
debate: "D4 (#297)"
milestone: "#33 — SA Knowledge Base"
depends-on: "ADR-008 (D1), ADR-009 (D2), ADR-010 (D3)"
---

# ADR-011: SA Knowledge Base — Source of Truth and Provenance

## Context

D3 (ADR-010) locked the runtime injection format: `[N] (source, section)` with XML `<domain-knowledge>` tags. The citation display shape is fixed. D4 must now define what those citations actually mean — what is authoritative, how provenance is tracked, and what guarantees exist for production traceability.

### Current provenance reality (code-verified)

- **Stored per chunk in Qdrant**: `text`, `metadata{source, scope, format, page, section, language, chunk_index, total_chunks}`, `vector` (1536-dim), `point_id` (MD5[:8])
- **NOT stored**: Ingest timestamp, document version, full document hash, embedding model version
- **Original documents**: NOT stored — only chunks exist. Cannot reconstruct original.
- **Re-upload**: Changed chunks added alongside old (old chunks NOT cleaned). No version tracking.
- **Delete**: In-memory only — Qdrant points orphaned.
- **Section extraction**: Regex `^#{1,3}\s+` at ingest time — markdown only, best-effort.

## Decision

### 1. Source of Truth

**Original uploaded document is the canonical source of truth.** Chunks and embeddings are derived retrieval indexes.

- When a citation references `(source: api-docs.md, section: Authentication)`, the authority is the original file as uploaded
- Chunk text is a derivative — it changes if chunking strategy changes
- Embeddings are a retrieval index — they change when embedding models change
- Neither chunks nor embeddings are authoritative by themselves

**First release retention contract**: The system must durably retain an immutable source reference record for every active document: `{source, scope, document_hash (SHA-256), ingested_at, chunk_count, format}`. This record is the provenance anchor — it allows verification that a cited source existed, when it was ingested, and what content hash it had. The operator retains the original file externally; the system retains the reference record internally. Together, these are sufficient to verify provenance: the operator can re-hash their file and compare against the stored `document_hash`.

Full document text storage (retaining the uploaded file content within the system) is a D5 decision. D4 does not require it — but D4 does require the metadata record above to be durable and immutable for active documents.

### 2. Citation Contract

D3 display format: `[N] (source, section)` — locked.

**Minimum internal metadata per chunk (mandatory):**

| Field | Status | Purpose |
|---|---|---|
| `source` | Mandatory | Reference back to original document |
| `chunk_index` | Mandatory | Deterministic position in document |
| `scope` | Mandatory | Access control context |
| `ingested_at` | Mandatory (new) | Freshness — when chunk entered the knowledge base |
| `document_hash` | Mandatory (new) | Version identity — SHA-256 of original document content |

**Best-effort metadata (stored when available):**

| Field | Status | Purpose |
|---|---|---|
| `section` | Best-effort | Markdown heading — empty for non-markdown or headingless chunks |
| `page` | Best-effort | PDF page number — null for non-PDF |

**Active-version chunk identity**: `scope + source + chunk_index` uniquely identifies a chunk in the current active version. This triple is always resolvable because only one version of a document exists per `(scope, source)` at any time. Citation `[N] (source: X, section: Y)` maps back to the active chunk via this triple.

**Historical chunk identity**: For historical provenance (tracing a past citation after the document has been superseded), the active-version triple is insufficient — `chunk_index=3` in version A is different content than `chunk_index=3` in version B. The full historical identity is: `scope + source + chunk_index + document_hash`. The `document_hash` discriminates which version the chunk belonged to.

**Stable storage identity contract**: The current Qdrant point ID (`int(MD5(text)[:8], 16)`) is derived from chunk text only. This is insufficient for provenance — identical text across different documents or scopes produces the same ID, causing silent overwrites. First release must change the storage identity to a composite key: `scope + source + chunk_index`. Implementation may use a hash of this composite or a structured ID. #304 must implement this change.

### 3. Versioning / Supersession

**Latest-wins with document-level tracking for first release.**

- Re-upload of the same source file within a scope replaces the previous version
- Old chunks for that source are deleted before new chunks are ingested
- Only one active version per `(scope, source)` pair at any time
- Document-level metadata tracks: `source`, `scope`, `document_hash` (SHA-256), `ingested_at`, `chunk_count`
- If re-upload has same `document_hash`: skip re-ingest entirely (content unchanged)
- **Historical provenance (document-version level only)**: When a document is superseded, chunk data is deleted but a tombstone metadata record is retained: `{source, scope, document_hash, ingested_at, superseded_at, chunk_count}`. This allows answering "what document version was active on date X?" but does NOT allow recovering the exact chunk text that a past citation referenced. First release guarantees document-version-level historical provenance — not exact historical chunk provenance. A past citation can be traced to "document X, version hash Y, was active from T1 to T2" but the cited chunk content is not recoverable from the system alone (operator may retain the original file externally).

### 4. Conflict Handling

**D4 provides structural metadata. D7 defines behavioral response.**

What D4 settles:
- Every chunk carries `ingested_at` — freshness is always known
- Document-level metadata tracks which sources exist per scope
- Search results can be ordered or filtered by freshness if needed

What D7 owns:
- Whether the bot should prefer newer sources
- Whether the bot should flag conflicting information
- Whether the bot should refuse to answer when sources conflict
- The answer contract under insufficient or contradictory evidence

### 5. End-to-End Provenance Chain

| Stage | Provenance preserved |
|---|---|
| Document upload | `source`, `document_hash`, `ingested_at`, `scope`, `format` |
| Chunking | Above + `section`, `chunk_index`, `page` |
| Qdrant storage | All above in `payload.metadata` + `text` + `vector` |
| Search/retrieval | `text`, `score`, `source`, `section`, `page`, `chunk_index`, `scope`, `ingested_at` |
| Plugin injection | `[N] (source, section)` displayed. Per-result provenance identifiers must be included in audit log (see below). |
| Bot answer | Bot cites `[N]`. Active-version chunks traceable via `source + chunk_index + scope`. |

**Audit provenance requirement**: D3 (ADR-010) defined the runtime audit shape as `knowledge_search: bot={bot_name} client={client_id} scopes=[...] results={count} latency={ms}`. This is insufficient for provenance tracing — it does not capture which specific chunks were injected. D4 constrains #302 and #307 to extend the runtime audit to include per-result provenance identifiers: for each injected chunk, log `{source, chunk_index, scope, document_hash, score}`. This enables tracing from a bot answer back to the specific chunks that informed it.

### 6. Internal vs Displayed Provenance

**Displayed in bot answer** (via D3 format): `source`, `section` (when available), `page` (for PDFs).

**Internal only** (for traceability, not shown to user): `document_hash`, `ingested_at`, `chunk_index`, `scope`, `format`.

## Alternatives Considered

### Source of truth

| Option | Verdict | Reason |
|---|---|---|
| Original document | Accepted | Clear authority chain; verification possible |
| Chunk text | Rejected | Derivative — changes with chunking strategy |
| Embeddings | Rejected | Retrieval index only — cannot reconstruct knowledge; changes with model |

### Versioning

| Option | Verdict | Reason |
|---|---|---|
| Replace-in-place (current) | Rejected | Broken — old chunks not cleaned on re-upload |
| Append-only versions | Rejected for first release | Complexity; multiple versions confuse search results |
| Latest-wins with tracking | Accepted | Simple; operator uploads new, old is replaced; document metadata tracked |
| Explicit supersession with history | Rejected for first release | Over-engineering; no version management UX |

### Conflict handling

| Option | Verdict | Reason |
|---|---|---|
| No structural tracking | Rejected | Cannot even identify conflicts without freshness data |
| Explicit conflict metadata | Rejected | Contradiction detection is an AI problem — not reliable |
| Freshness metadata (D4) + behavioral rules (D7) | Accepted | Structural support now; behavioral decisions deferred to D7 |
| Source ranking model | Rejected for first release | No ranking UX; freshness is sufficient structural signal |

## Consequences

### Downstream constraints

- **D5 (#298)**: Must implement latest-wins re-upload (delete old chunks → ingest new). Must store document-level metadata (`source`, `scope`, `document_hash`, `ingested_at`, `chunk_count`). Must add `ingested_at` and `document_hash` to chunk metadata.
- **D6 (#299)**: Imported vectors must carry the same minimum metadata (`source`, `chunk_index`, `scope`, `ingested_at`, `document_hash`). If external vectors lack this metadata, import must be rejected or metadata must be synthesized.
- **#302**: Must extend runtime audit to include per-result provenance identifiers (`{source, chunk_index, scope, document_hash, score}` per injected chunk). D3's `results={count}` alone is insufficient for provenance tracing.
- **#304**: Must add `ingested_at` and `document_hash` to `DocumentMetadata` model. Must implement composite storage identity (`scope + source + chunk_index`). Must implement tombstone ledger for superseded documents. Must fix delete to clean Qdrant (current bug).
- **#306 / D7**: Quality evaluation must verify citation accuracy — does `[N]` map to the correct active chunk? Must define answer behavior under conflict (freshness metadata available via `ingested_at`).
- **#307**: Must verify per-result audit provenance is captured end-to-end. Must verify tombstone ledger works for superseded documents.

### What D4 does NOT decide

- Ingestion paths, lifecycle operations, operator workflow (D5)
- External vector import contract (D6)
- Answer behavior under conflict, insufficient evidence, or stale sources (D7)
- Full document text storage policy (D5 implementation detail)
