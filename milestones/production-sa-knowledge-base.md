---
title: "[Production] SA Knowledge Base for Domain & Project Intelligence"
---

# SA Knowledge Base for Domain & Project Intelligence

**Milestone**: [#33](https://github.com/ai-agentopia/agentopia-protocol/milestone/33)
**Status**: Implementation complete, automated verification complete. #301-#306 closed. #307 OPEN — live pilot evaluation pending first-client deployment. Architecture locked (ADRs 008-014).
**Date**: 2026-03-30
**Type**: Production program document

---

## 1. Context / Business Driver

Production client requirement: SA bots must use client-provided domain and project knowledge as a reliable working brain — not just conversation memory, but structured document knowledge with provenance, citation, and governance.

SA bots today can remember conversation facts (via mem0) and answer from LLM training data. They cannot reliably answer from client-specific documentation: architecture specs, API references, project standards, domain knowledge. This gap makes SA bots unreliable for production client work where accuracy and traceability matter.

---

## 2. Current Baseline in Code

### Knowledge System (KnowledgeService)

**Exists today**:
- Document ingestion: PDF, HTML, markdown, text, code
- Chunking: fixed-size (default 512 tokens), paragraph, code-aware
- Vector search: Qdrant backend with OpenRouter text-embedding-3-small (1536 dims)
- Scope isolation: collection-per-scope in Qdrant
- Deduplication: content hash per chunk (MD5)
- Citation model: `Citation(source, section, page, chunk_index, score)`
- Incremental indexing: unchanged chunks skipped
- Stale scope detection: `list_stale_scopes(max_age_secs)`
- API: file upload, webhook ingest, search, scope CRUD, reindex

**Does NOT exist**:
- No authentication on knowledge routes (no `require_auth`)
- No access control per scope
- No runtime injection into bot inference (knowledge disconnected from gateway)
- No versioning/supersession model
- No external vector import
- No evaluation framework
- No operator UI for knowledge management
- Semantic chunking listed in enum but not implemented

### Memory System (mem0-api)

**Exists today**:
- Conversational fact extraction via LLM
- Hybrid vector (Qdrant) + graph (Neo4j) storage
- Semantic recall with multi-scope fan-out
- Gateway plugin: auto-recall before inference, auto-capture after response

**Is NOT knowledge**:
- mem0 extracts facts from conversations, not from documents
- No citation tracking, no source provenance
- No document lifecycle management
- Different storage model, different retrieval semantics

### Critical Architecture Gap

Knowledge system and gateway inference path are **completely disconnected**. The mem0 plugin injects conversation memories into the LLM prompt via `before_agent_start`. No equivalent path exists for document knowledge. A bot with ingested domain documents today cannot use them during inference.

---

## 3. Program Objective

Build a production-ready knowledge foundation that enables SA bots to:
1. Ingest client-provided documents (markdown, text, HTML, PDF, code)
2. Retrieve relevant knowledge during inference — automatically or on-demand
3. Cite source documents in answers (filename, section)
4. Operate within governed scope isolation (no cross-client leakage)
5. Maintain knowledge quality over time (lifecycle, staleness, evaluation)

---

## 4. Non-Goals (First Release)

- Graph RAG / entity extraction from documents
- Semantic chunking (requires evaluation against current strategies)
- Knowledge-aware workflow planning (SA uses knowledge for answers, not for delivery planning)
- Cross-bot knowledge sharing (each SA bot accesses its own scopes unless explicitly shared)

## 4a. Debate-Resolved Scope Decisions

Items below were debate-gated before D1-D6 concluded. Their status is now resolved.

| Item | Resolved By | Outcome | First Release? |
|---|---|---|---|
| Tenancy and isolation model | D1 — [ADR-008](../adrs/008-sa-kb-product-boundary-tenancy-isolation.md) | Hybrid tenancy, client→scope two-level isolation | **Yes** — in scope |
| External vector import | D6 — [ADR-013](../adrs/013-sa-kb-external-vector-import-contract.md) | Deferred. No validated demand. Provenance conflict. | **No** — deferred to future release |

---

## 5. Debate-Gated Decision Areas

All 7 debate sessions are closed. Architecture is locked. ADRs 008-014 accepted. Implementation phase ready.

| # | Decision Area | Issue | Status |
|---|---|---|---|
| D1 | Product Boundary + Tenancy + Isolation | [#294](https://github.com/ai-agentopia/agentopia-protocol/issues/294) | **Closed** — [ADR-008](../adrs/008-sa-kb-product-boundary-tenancy-isolation.md) (accepted) |
| D2 | Governance / Security / Compliance | [#295](https://github.com/ai-agentopia/agentopia-protocol/issues/295) | **Closed** — [ADR-009](../adrs/009-sa-kb-governance-security-compliance.md) (accepted) |
| D3 | Runtime Retrieval Architecture | [#296](https://github.com/ai-agentopia/agentopia-protocol/issues/296) | **Closed** — [ADR-010](../adrs/010-sa-kb-runtime-retrieval-architecture.md) (accepted) |
| D4 | Source of Truth / Provenance | [#297](https://github.com/ai-agentopia/agentopia-protocol/issues/297) | **Closed** — [ADR-011](../adrs/011-sa-kb-source-of-truth-and-provenance.md) (accepted) |
| D5 | Ingestion Model + Document Lifecycle | [#298](https://github.com/ai-agentopia/agentopia-protocol/issues/298) | **Closed** — [ADR-012](../adrs/012-sa-kb-ingestion-model-document-lifecycle.md) (accepted) |
| D6 | External Vector Import Contract | [#299](https://github.com/ai-agentopia/agentopia-protocol/issues/299) | **Closed** — [ADR-013](../adrs/013-sa-kb-external-vector-import-contract.md) (accepted) |
| D7 | Quality / Evaluation / Answer Contract | [#300](https://github.com/ai-agentopia/agentopia-protocol/issues/300) | **Closed** — [ADR-014](../adrs/014-sa-kb-quality-evaluation-answer-contract.md) (accepted) |

### Debate Dependency Chain

```
D1 (Boundary + Tenancy)
  → D2 (Governance)
    → D3 (Runtime Retrieval)
      → D4 (Source of Truth)
        → D5 (Ingestion + Lifecycle)
          → D6 (External Import)
            → D7 (Quality + Answer Contract)
```

### D1 Outcome — Product Boundary + Tenancy + Isolation

**ADR**: [008-sa-kb-product-boundary-tenancy-isolation](../adrs/008-sa-kb-product-boundary-tenancy-isolation.md)

| Decision Area | Outcome |
|---|---|
| Product boundary | Client/project knowledge service consumed by SA bots (not per-bot, not platform-wide) |
| Tenancy model | Hybrid — shared deployment with logical isolation for first release; physical isolation as upgrade path |
| Isolation hierarchy | Two-level: `client_id` → `scope_name`. Client is the hard wall. |
| Sharing model | Within-client: subscription-based (bots access only explicitly assigned scopes). Cross-client: blocked. Authorization model owned by D2. |
| Scope identity | `{client_id}/{scope_name}` — Qdrant collection: `kb-{client_id}-{scope_name}` |

**Downstream constraints set by D1:**
- D2 must define how `client_id` is assigned to bots and enforced at auth layer
- D3 must resolve scopes using the `{client_id}/{scope_name}` format
- #301 scope model must carry `client_id` with client-bounded subscriptions
- #305 operator UX is organized by client then scope, not by bot

### D2 Outcome — Governance / Security / Compliance

**ADR**: [009-sa-kb-governance-security-compliance](../adrs/009-sa-kb-governance-security-compliance.md)

| Decision Area | Outcome |
|---|---|
| Authentication | All knowledge routes authenticated. Write ops = operator session. Read ops = operator session OR bot bearer. Scope listing: operator sees all client scopes, bot sees only subscribed scopes. |
| Authorization | Config-driven subscription model: `client_id` + `knowledge_scopes` assigned at deploy time. Bot accesses only subscribed scopes. |
| Client binding | `client_id` as deploy-time config. Bot identity resolved at runtime to enforce client + scope boundaries. |
| Operator model | Single operator (admin_user) for first release. Multi-operator RBAC deferred. |
| Audit | 10 event types via structured JSON logs. Denials at WARN. DB-persisted audit deferred. |
| Retention/deletion | Immediate permanent deletion. No mandatory retention for documents. Staleness surfaced, not auto-purged. |
| Audit retention | **Provisional** — environment-dependent. Production go-live must define + verify minimum retention. Added to #307. |
| Provisional (D3) | Exact enforcement point, query audit shape, and bot identity propagation mechanism depend on D3. |

**Downstream constraints set by D2:**
- D3 must authenticate bot runtime retrieval and carry bot identity — enforcement and identity mechanism are provisional but the constraint is final
- #301 must add auth guards to all knowledge routes and implement `client_id` + `knowledge_scopes` on bot registration
- #305 operator UX must include scope subscription management in bot deploy/update flow
- #307 must prove: auth on all routes, cross-client blocked, unsubscribed-scope blocked, audit captured

### D3 Outcome — Runtime Retrieval Architecture

**ADR**: [010-sa-kb-runtime-retrieval-architecture](../adrs/010-sa-kb-runtime-retrieval-architecture.md)

| Decision Area | Outcome |
|---|---|
| Mechanism | Gateway plugin using `before_agent_start` hook — auto-inject on every turn (same pattern as mem0) |
| Scope resolution | Server-side — bot-config-api resolves scopes from bot registration. Bot sends identity, not scopes. |
| Precedence | System prompt → knowledge (priority 10) → mem0 (priority 20) → history → user message |
| Token budget | 3000 tokens max, topK=5, drop lowest-scoring if over budget |
| Timeout | 5000ms, non-blocking. Timeout/error → skip + warn + continue. |
| Injection format | XML `<domain-knowledge>` with numbered inline citations `[N] (source, section)` |
| Bot identity | Per-bot `AGENTOPIA_RELAY_TOKEN` bearer + `X-Bot-Name` header. Token is per-bot unique (K8s Secret). Validated via forward lookup (proven relay auth pattern). |

**D2 provisional items resolved by D3:**
- Enforcement point: bot-config-api search endpoint with server-side scope resolution
- Audit shape: `knowledge_search: bot={bot_name} client={client_id} scopes=[...] results={count} latency={ms}`
- Identity propagation: per-bot `AGENTOPIA_RELAY_TOKEN` bearer + `X-Bot-Name` header

**Downstream constraints set by D3:**
- D4 must define provenance rules compatible with `[N] (source, section)` citation format
- #302 must implement the knowledge-retrieval gateway plugin per this architecture
- #304 citation implementation must use SearchResult.citation fields in the `[N]` format
- #306 quality evaluation must measure auto-inject relevance and citation accuracy

### D4 Outcome — Source of Truth / Provenance

**ADR**: [011-sa-kb-source-of-truth-and-provenance](../adrs/011-sa-kb-source-of-truth-and-provenance.md)

| Decision Area | Outcome |
|---|---|
| Canonical authority | Original uploaded document. Chunks/embeddings are derived indexes. Immutable source reference record is mandatory (source, scope, document_hash, ingested_at, chunk_count, format). Operator retains original file; system retains reference. |
| Citation contract | Mandatory: `source`, `chunk_index`, `scope`, `ingested_at` (new), `document_hash` (new). Best-effort: `section`, `page`. |
| Active chunk identity | `scope + source + chunk_index` — sufficient for active version (one version per scope+source). |
| Historical chunk identity | `scope + source + chunk_index + document_hash` — discriminates versions. First release = document-version-level historical provenance only (chunk text not recoverable). |
| Versioning | Latest-wins: re-upload replaces previous. One active version per `(scope, source)`. Tombstone metadata ledger for superseded versions. |
| Conflict handling | Structural: `ingested_at` freshness metadata on every chunk. Behavioral: deferred to D7 (answer contract). |
| Deterministic tracing | Active: `[N]` → `source + chunk_index + scope`. Historical: add `document_hash`. Audit must include per-result provenance identifiers. |
| Internal vs displayed | Displayed: `source`, `section`, `page`. Internal: `document_hash`, `ingested_at`, `chunk_index`, `scope`, `format`. |

**Downstream constraints set by D4:**
- D5 must implement latest-wins re-upload, document-level metadata storage, and tombstone ledger
- ~~D6 imported vectors must carry minimum metadata or be rejected~~ → **Superseded by D6**: import deferred from first release
- #302 must extend runtime audit to include per-result provenance (`{source, chunk_index, scope, document_hash, score}` per chunk)
- #304 must add `ingested_at` + `document_hash` to metadata; composite storage identity; tombstone ledger; fix Qdrant delete bug
- #307 must verify per-result audit provenance end-to-end and tombstone ledger functionality
- D7 must define answer behavior under conflict using `ingested_at` freshness data

### D5 Outcome — Ingestion Model + Document Lifecycle

**ADR**: [012-sa-kb-ingestion-model-document-lifecycle](../adrs/012-sa-kb-ingestion-model-document-lifecycle.md)

| Decision Area | Outcome |
|---|---|
| Ingestion paths | File upload (multipart) only. Webhook deferred (current query-param shape not production-safe). |
| Lifecycle model | Upload → Active → Re-upload (supersede) or Delete. Tombstone for superseded/deleted. |
| Re-upload behavior | Latest-wins with hash check. Same hash = skip. Different hash = two-phase replace (prepare new → commit swap). Old active until committed. Failure semantics defined. |
| DocumentRecord | Per-document metadata: `source, scope, document_hash, ingested_at, chunk_count, format, status, superseded_at, deleted_at`. |
| Delete behavior | Must clean Qdrant + memory. Tombstone record created. |
| Operator workflow | Upload, re-upload (feedback: unchanged/updated), delete, list with `ingested_at`, staleness per document. |
| Staleness | Per-document via `ingested_at`. Operator-initiated action only. No automation. |
| Chunking | Existing strategies (fixed-size, paragraph, code-aware). Semantic: deferred. |

**Downstream constraints set by D5:**
- ~~D6 imported documents must create DocumentRecord with same lifecycle~~ → **Superseded by D6**: import deferred from first release
- #303 must implement DocumentRecord, two-phase replace with retry logic for partial failures, tombstone, Qdrant delete
- #305 operator UI must show upload feedback, document list with ingested_at, delete confirmation
- #307 must prove: no mixed old+new chunks under normal operation, partial-failure states are recoverable, tombstone creation works

### D6 Outcome — External Vector Import Contract

**ADR**: [013-sa-kb-external-vector-import-contract](../adrs/013-sa-kb-external-vector-import-contract.md)

| Decision Area | Outcome |
|---|---|
| Import decision | **Deferred** from first release |
| Rationale | No validated demand. Provenance conflict with D4 (imports lack canonical document). Embedding model mismatch risk. Metadata synthesis weakens provenance. |
| Supported alternative | Clients provide source documents → system ingests/chunks/embeds via file upload |
| Future path | Re-embedding mandatory, synthetic DocumentRecord with `provenance_type: "imported"`, D4 metadata required |

**Impact on implementation:** No import endpoint needed for first release. #303/#304/#307 scope reduced.

### D7 Outcome — Quality / Evaluation / Answer Contract

**ADR**: [014-sa-kb-quality-evaluation-answer-contract](../adrs/014-sa-kb-quality-evaluation-answer-contract.md)

| Decision Area | Outcome |
|---|---|
| Quality bar | First-client pilot bar. Manual evaluation, 8 criteria, all must pass. Hard requirements at 100%. |
| Evaluation method | Operator-run: ≥20 sample questions + required scenario matrix (6 edge-case types). |
| Answer — insufficient evidence | Disclose unavailability. Label any general answer as non-authoritative. No silent fallback. No fabricated citations. |
| Answer — conflict | Cite ALL conflicting sources with `[N]`. Surface contradiction explicitly. Recommend newer, but do not suppress older. |
| Answer — stale source | Use source, note age if relevant. Operator manages freshness. |
| Citation rules | Must cite `[N]` from knowledge. Must NOT cite from general knowledge. No silent fallback. Best-effort, prompt-enforced. |

**Go-live criteria (8 checks, 3 are hard 100% requirements):**
1. Retrieval relevance ≥80% (20+ questions per scope)
2. Citation accuracy ≥90%
3. Answer grounding — no systematic hallucination (20+ answers)
4. **Scope isolation — 100%** (hard: zero leakage)
5. **Zero fabricated citations — 100%** (hard: no `[N]` without knowledge)
6. **Unavailability disclosure — 100%** (hard: always discloses when knowledge unavailable)
7. Conflict surfacing — cites all conflicting sources
8. Staleness visibility — timestamps accurate and visible

**Required scenario matrix:** Positive grounded (≥20), empty retrieval (≥3), timeout (≥1), conflicting sources (≥2), stale source (≥1), scope isolation negative (≥1).

**Downstream constraints set by D7:**
- #302 must include answer contract instructions in plugin prompt template (unavailability disclosure, conflict surfacing, citation rules)
- #306 must create evaluation checklist with 8 criteria, scenario matrix (6 types), and pass/fail thresholds including 3 hard 100% requirements
- #307 must verify: evaluation was run, all 8 criteria passed (3 hard requirements non-negotiable), scenario matrix fully covered, answer contract behaviors proven

---

## 6. Planned Implementation Workstreams

Debates complete. Implementation executing in waves: A (#304 + #301) → B (#303 + #302) → C (#305 + #306) → D (#307).

| Track | Issue | Status | Depends On |
|---|---|---|---|
| Provenance + citation infrastructure | [#304](https://github.com/ai-agentopia/agentopia-protocol/issues/304) | **Code complete** — provenance fields, composite point ID, Qdrant deletion, audit logging. 29 tests passing. | D4 (resolved) |
| Scoped knowledge model + access control | [#301](https://github.com/ai-agentopia/agentopia-protocol/issues/301) | **Code complete** — client_id/knowledge_scopes on deploy, auth guards, bot scope resolution, CRD persistence, startup rebuild. 23 tests passing. | D1 + D2 (resolved) |
| Ingestion pipeline + document lifecycle | [#303](https://github.com/ai-agentopia/agentopia-protocol/issues/303) | **Code complete** — Postgres DocumentRecord, two-phase replace, same-hash short-circuit, tombstone lifecycle, webhook 410 Gone. 24 tests passing. | D5 + #304 + #301 |
| Runtime retrieval injection for SA bots | [#302](https://github.com/ai-agentopia/agentopia-protocol/issues/302) | **Code complete** — knowledge-retrieval gateway plugin, priority 10, RELAY_TOKEN auth, server-side scope resolution, D3 XML injection + D7 answer contract, 5s timeout, token budget. Helm conditional enablement. Deploy path wires `knowledgeRetrieval.enabled: true` into Helm values when bot has knowledge config. PATCH path reconciles enable/disable. 19 vitest passing. Hotfix: PR #311 wired deploy + PATCH → Helm values. | D3 + #301 + #304 |
| Operator workflow + admin surfaces | [#305](https://github.com/ai-agentopia/agentopia-protocol/issues/305) | **Code complete** — KnowledgePage with client-first navigation, merged data model (bot config + ingested scopes). Configured clients/scopes visible before document ingest. "Not indexed" state for empty scopes. Upload into configured scope without recreation. 27 knowledge tests + 174 total. | #301 + #303 + #304 |
| Evaluation + quality gates | [#306](https://github.com/ai-agentopia/agentopia-protocol/issues/306) | **Code complete** — Full D7 evaluation pack: checklist (8 criteria, 3 hard), scenario matrix (6 types), question template (20+), sample fixtures, setup guide, go/no-go template. 22 validation tests passing. | D7 + #302 + #303 |
| Rollout / UAT / production verification | [#307](https://github.com/ai-agentopia/agentopia-protocol/issues/307) | **OPEN** — Automated verification complete (295 tests). Live pilot evaluation pending. Criteria 1-3, 5-7 require live bot + client docs. Hard #5/#6 not yet proven at 100%. | All above |

---

## 7. Milestone Issue Map

### Group A — Debate / Decision (7 issues)
- #294: Product Boundary + Tenancy + Isolation
- #295: Governance / Security / Compliance
- #296: Runtime Retrieval Architecture
- #297: Source of Truth / Provenance
- #298: Ingestion Model + Document Lifecycle
- #299: External Vector Import Contract
- #300: Quality / Evaluation / Answer Contract

### Group B — Implementation Workstreams (7 issues)
- #301: Scoped knowledge model + access control
- #302: Runtime retrieval injection
- #303: Ingestion pipeline + lifecycle
- #304: Provenance + citation
- #305: Operator workflow + admin
- #306: Evaluation + quality gates
- #307: Rollout / UAT / production

**Total**: 14 issues

---

## 8. Dependencies and Sequencing

### External Dependencies
- Client document samples (needed for chunking evaluation and quality testing)
- ~~Client deployment model decision~~ → **Resolved by D1**: shared deployment with logical isolation for first release; physical isolation as upgrade path
- Client data sensitivity classification (affects governance requirements)

### Internal Dependencies
- P1 milestone complete (web app primary — DONE)
- Gateway plugin architecture stable (mem0 plugin pattern proven)
- Qdrant + embedding infrastructure operational (proven in dev)

---

## 9. Production Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Knowledge disconnected from inference | SA bot cannot use documents | Architecture path resolved by D3 (gateway plugin via `before_agent_start`); implementation remains in #302 |
| No auth on knowledge routes | Any caller can ingest/search/delete | D2 (governance) must add require_auth before production |
| Fixed-size chunking insufficient | Poor retrieval for domain-specific docs | Evaluate against client sample documents before committing |
| Stale documents degrade answers | Wrong advice from outdated sources | Lifecycle management + staleness alerting |
| No evaluation framework | Ship without knowing quality | Quality bar defined by D7 (8 criteria, 3 hard 100%). Implementation in #306, verification in #307. |
| Token budget unmanaged | Knowledge context crowds out conversation | Architecture resolved by D3 (3000 token budget, topK=5); enforcement remains in #302 |

---

## 10. Production Acceptance Criteria

- [ ] SA bot answers domain questions using ingested client documents
- [ ] Answers cite source documents (filename, section at minimum)
- [ ] Knowledge is scope-isolated (no cross-client/cross-scope leakage)
- [ ] Knowledge routes are authenticated and audited
- [ ] Retrieval quality meets defined evaluation threshold
- [ ] Operator can upload, update, and deprecate knowledge documents
- [ ] Knowledge lifecycle does not degrade answer quality over time
- [ ] Answer contract defined: bot behavior on insufficient/conflicting/stale evidence

---

## 11. Open Questions

1. ~~What is the client deployment model?~~ → **Partially resolved by D1**: shared with logical isolation for first release, physical isolation as upgrade path. Remaining: client-specific infra requirements TBD.
2. What data classification do client documents fall under? (Affects D2)
3. ~~Do clients actually have pre-embedded vectors?~~ → **Resolved by D6**: Unvalidated. Import deferred. If demand emerges, future release adds with re-embedding + synthetic metadata.
4. ~~What is the token budget for knowledge context alongside mem0 memory?~~ → **Resolved by D3**: 3000 tokens max for knowledge (topK=5 chunks), ~4000 total with mem0. <2% of 200K context.
5. ~~What latency is acceptable for knowledge retrieval?~~ → **Resolved by D3**: 5000ms timeout, non-blocking. Skip + warn on timeout. Typical: 750-2100ms for 3-scope search.
6. Is the current embedding model (text-embedding-3-small) sufficient for domain documents?
7. How is `client_id` represented in the database? (New table? Field on existing tables?) — raised by D1
8. How does the operator authenticate and prove client membership? — raised by D1

---

## 12. Exit Criteria: Debate Phase → Implementation Phase

Debates are complete when:
1. All 7 debate issues have ADR decisions documented
2. Implementation workstream issues are unblocked with concrete scope
3. Acceptance criteria for each workstream are refined based on debate outputs
4. No major open question remains that could change the architecture direction
5. CTO approves transition to implementation phase

---

## 13. Recommended Debate Order

1. **D1** — Product Boundary + Tenancy (everything depends on this)
2. **D2** — Governance (frames constraints for all downstream work)
3. **D3** — Runtime Retrieval (core value delivery — critical path)
4. **D4** — Source of Truth / Provenance (depends on injection format from D3)
5. **D5** — Ingestion + Lifecycle (depends on versioning from D4)
6. **D6** — External Import (special case of ingestion)
7. **D7** — Quality + Answer Contract (final gate, needs all above)
