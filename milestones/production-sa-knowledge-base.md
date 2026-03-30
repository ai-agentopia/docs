---
title: "[Production] SA Knowledge Base for Domain & Project Intelligence"
---

# SA Knowledge Base for Domain & Project Intelligence

**Milestone**: [#33](https://github.com/ai-agentopia/agentopia-protocol/milestone/33)
**Status**: Debate phase — architecture not yet locked
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

- External vector database import (deferred pending debate)
- Graph RAG / entity extraction from documents
- Semantic chunking (requires evaluation against current strategies)
- Multi-tenant SaaS isolation (depends on tenancy debate)
- Knowledge-aware workflow planning (SA uses knowledge for answers, not for delivery planning)
- Cross-bot knowledge sharing (each SA bot accesses its own scopes unless explicitly shared)

---

## 5. Debate-Gated Decision Areas

Architecture is NOT locked. 7 debate sessions must conclude before final solution design.

| # | Decision Area | Issue | Status |
|---|---|---|---|
| D1 | Product Boundary + Tenancy + Isolation | [#294](https://github.com/ai-agentopia/agentopia-protocol/issues/294) | Open |
| D2 | Governance / Security / Compliance | [#295](https://github.com/ai-agentopia/agentopia-protocol/issues/295) | Open |
| D3 | Runtime Retrieval Architecture | [#296](https://github.com/ai-agentopia/agentopia-protocol/issues/296) | Open |
| D4 | Source of Truth / Provenance | [#297](https://github.com/ai-agentopia/agentopia-protocol/issues/297) | Open |
| D5 | Ingestion Model + Document Lifecycle | [#298](https://github.com/ai-agentopia/agentopia-protocol/issues/298) | Open |
| D6 | External Vector Import Contract | [#299](https://github.com/ai-agentopia/agentopia-protocol/issues/299) | Open |
| D7 | Quality / Evaluation / Answer Contract | [#300](https://github.com/ai-agentopia/agentopia-protocol/issues/300) | Open |

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

---

## 6. Planned Implementation Workstreams

All blocked on debate outputs. Implementation scope will be refined after debates conclude.

| Track | Issue | Blocked On |
|---|---|---|
| Scoped knowledge model + access control | [#301](https://github.com/ai-agentopia/agentopia-protocol/issues/301) | D1 + D2 |
| Runtime retrieval injection for SA bots | [#302](https://github.com/ai-agentopia/agentopia-protocol/issues/302) | D3 |
| Ingestion pipeline + document lifecycle | [#303](https://github.com/ai-agentopia/agentopia-protocol/issues/303) | D5 |
| Provenance + citation support | [#304](https://github.com/ai-agentopia/agentopia-protocol/issues/304) | D4 |
| Operator workflow + admin surfaces | [#305](https://github.com/ai-agentopia/agentopia-protocol/issues/305) | D2 + D5 |
| Evaluation + quality gates | [#306](https://github.com/ai-agentopia/agentopia-protocol/issues/306) | D7 |
| Rollout / UAT / production verification | [#307](https://github.com/ai-agentopia/agentopia-protocol/issues/307) | All |

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
- Client deployment model decision (dedicated vs shared — affects tenancy debate)
- Client data sensitivity classification (affects governance requirements)

### Internal Dependencies
- P1 milestone complete (web app primary — DONE)
- Gateway plugin architecture stable (mem0 plugin pattern proven)
- Qdrant + embedding infrastructure operational (proven in dev)

---

## 9. Production Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Knowledge disconnected from inference | SA bot cannot use documents | D3 (retrieval architecture) is the critical path |
| No auth on knowledge routes | Any caller can ingest/search/delete | D2 (governance) must add require_auth before production |
| Fixed-size chunking insufficient | Poor retrieval for domain-specific docs | Evaluate against client sample documents before committing |
| Stale documents degrade answers | Wrong advice from outdated sources | Lifecycle management + staleness alerting |
| No evaluation framework | Ship without knowing quality | D7 must define quality bar before production use |
| Token budget unmanaged | Knowledge context crowds out conversation | D3 must define explicit token budget |

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

1. What is the client deployment model? (Dedicated vs shared — affects D1)
2. What data classification do client documents fall under? (Affects D2)
3. Do clients actually have pre-embedded vectors? (Affects D6 priority)
4. What is the token budget for knowledge context alongside mem0 memory? (Affects D3)
5. What latency is acceptable for knowledge retrieval? (Affects D3)
6. Is the current embedding model (text-embedding-3-small) sufficient for domain documents?

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
