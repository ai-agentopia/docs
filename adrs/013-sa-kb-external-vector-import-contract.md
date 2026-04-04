---
title: "ADR-013: SA Knowledge Base — External Vector Import Contract"
status: accepted
date: 2026-03-30
decision-makers: CTO
debate: "D6 (#299)"
milestone: "#33 — SA Knowledge Base"
depends-on: "ADR-008 (D1), ADR-009 (D2), ADR-010 (D3), ADR-011 (D4), ADR-012 (D5)"
---

# ADR-013: SA Knowledge Base — External Vector Import Contract

## Context

D4 (ADR-011) established: original document is canonical, embeddings are index-only, mandatory chunk metadata includes `document_hash` and `ingested_at`. D5 (ADR-012) established: file upload is the only first-release ingestion path. D6 must decide whether external vector import (loading pre-embedded vectors from a client or third-party system) is accepted, deferred, or rejected for first release.

### Current reality

- Embedding model: `text-embedding-3-small` (1536 dims) via OpenRouter
- Qdrant collections: one per scope, COSINE similarity
- No import endpoint exists today
- Open question #3 in milestone doc: "Do clients actually have pre-embedded vectors?" — unresolved

## Decision

**External vector import is deferred from first release.**

### Rationale

1. **Provenance conflict with D4**: D4 established that original documents are canonical and embeddings are derived indexes. Imported vectors invert this — they are the index with no canonical document in the system. Accepting them would weaken the provenance model.

2. **Metadata gap**: D4 mandatory metadata (`document_hash`, `ingested_at`, composite identity `scope + source + chunk_index`) would need to be synthesized for imports. Synthesized provenance fields are structurally weaker than natively-generated ones.

3. **Embedding model mismatch risk**: If client vectors were embedded with a different model (different dimensions or similarity semantics), they cannot coexist in the same Qdrant collection. Re-embedding would negate the import's value.

4. **No validated demand**: Open question #3 is unresolved. No client has confirmed they have pre-embedded vectors ready for import.

5. **Scope discipline**: D5 already constrained first release to file upload only. Adding vector import increases ingestion complexity without proven demand.

### What IS supported

Clients provide source documents (PDF, markdown, HTML, text, code). The system ingests, chunks, and embeds them through the proven file upload path. This preserves full provenance (D4) and lifecycle (D5).

### Future release path

If client demand is validated, a future release could add vector import with these minimum requirements:
- **Re-embedding mandatory**: Imported content must be re-embedded with the system's model to ensure collection consistency
- **Synthetic DocumentRecord**: `provenance_type: "imported"` marker to distinguish from native ingestion
- **D4 metadata required**: `source`, `chunk_index`, `scope`, `ingested_at`, `document_hash` must be provided or synthesized
- **Provenance degradation warning**: Citations from imported content should carry a weaker provenance signal
- **Same lifecycle**: DocumentRecord, tombstone, two-phase replace all apply

## Alternatives Considered

| Option | Verdict | Reason |
|---|---|---|
| Accept import with full provenance requirements | Rejected for first release | No validated demand; metadata synthesis weakens provenance |
| Accept import with re-embedding | Rejected for first release | Re-embedding negates import value; still needs synthetic metadata |
| Defer import | Accepted | No proven demand; provenance model preserved; can add later if needed |
| Reject import permanently | Rejected | Too strong — future demand may be real; architecture should not permanently block it |

## Consequences

### Decision-pending reclassification

Per the milestone doc §4a, external vector import was listed as decision-pending (gated on D6). D6 now defers it.

**Reclassification**: Move from "Decision-Pending" to "Deferred" — not a non-goal (could be added later), but not in first release scope.

### Downstream impact

- **#303**: No import endpoint needed for first release. Ingestion pipeline = file upload only.
- **#304**: Provenance model covers native ingestion only. No import-specific provenance handling.
- **#307**: Rollout does not need to verify import functionality.

### What D6 does NOT decide

- Quality evaluation (D7)
- Answer contract under conflicting/stale evidence (D7)
