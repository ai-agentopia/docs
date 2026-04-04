---
title: "ADR-014: SA Knowledge Base — Quality, Evaluation, and Answer Contract"
status: accepted
date: 2026-03-30
decision-makers: CTO
debate: "D7 (#300)"
milestone: "#33 — SA Knowledge Base"
depends-on: "ADR-008 (D1), ADR-009 (D2), ADR-010 (D3), ADR-011 (D4), ADR-012 (D5), ADR-013 (D6)"
---

# ADR-014: SA Knowledge Base — Quality, Evaluation, and Answer Contract

## Context

D1-D6 have defined the full architecture: product boundary, governance, runtime retrieval, provenance, ingestion lifecycle, and import scope. D7 is the final gate — it must define the production quality bar, evaluation method, and answer contract before the system goes live with a client.

### Locked inputs

- Runtime: gateway plugin auto-injects topK=5 chunks in `<domain-knowledge>` XML with `[N] (source, section)` citations (D3)
- Provenance: `ingested_at` freshness on every chunk, original document is canonical (D4)
- Ingestion: file upload only, latest-wins with two-phase replace (D5)
- No evaluation framework exists today

## Decision

### 1. Production Quality Bar (First-Client Pilot)

**Manual evaluation with defined criteria.** Scoped as a first-client pilot bar — thresholds may be raised for subsequent clients as evaluation corpus grows. Automated golden test sets are a future enhancement.

Go-live criteria (all must pass):

| # | Criterion | Method | Threshold |
|---|---|---|---|
| 1 | Retrieval relevance | Manual review: do top-5 chunks contain relevant info? | ≥ 20 questions per scope, ≥ 80% with at least one relevant chunk |
| 2 | Citation accuracy | Manual verification: does `[N]` reference contain the cited information? | ≥ 90% of citations verified correct |
| 3 | Answer grounding | Manual review: does the bot answer from injected context? | ≥ 20 answers reviewed, no systematic hallucination |
| 4 | Scope isolation | Automated test: query with bot A, verify no chunks from unsubscribed scopes | **100%** — zero cross-scope leakage (hard requirement) |
| 5 | Zero fabricated citations | Test: ask questions with no relevant knowledge, verify bot does NOT produce `[N]` citations | **100%** — zero fabricated citations (hard requirement) |
| 6 | Unavailability disclosure | Test: empty retrieval + timeout scenarios, verify bot discloses knowledge unavailability | **100%** — bot always discloses when domain knowledge is unavailable |
| 7 | Conflict surfacing | Test: ingest conflicting documents, verify bot surfaces contradiction with both citations | Bot cites all conflicting sources, does not silently pick one |
| 8 | Staleness visibility | Manual: verify `ingested_at` timestamps accurate and visible in operator tooling | Timestamps present and correct |

**Hard requirements (must be 100%):** Scope isolation (#4), zero fabricated citations (#5), unavailability disclosure (#6).

**Required evaluation scenario matrix (explicit edge-case coverage):**

| Scenario | Must Be Tested | Expected Behavior |
|---|---|---|
| Positive grounded answer | ≥ 20 questions | Answer with `[N]` citations from knowledge |
| Empty retrieval | ≥ 3 questions with no matching knowledge | Disclose unavailability, label general answer |
| Retrieval timeout | ≥ 1 simulated timeout | Disclose unavailability, no fabricated citations |
| Conflicting sources | ≥ 2 deliberately conflicting document pairs | Surface both with citations |
| Stale source | ≥ 1 old document (stale `ingested_at`) | Use source, note age if relevant |
| Scope isolation negative | ≥ 1 cross-scope query attempt | Zero leakage, 403 or empty results |

Not required for first release: automated regression suite, precision/recall metrics, A/B testing, continuous quality monitoring.

### 2. Answer Contract

#### Under insufficient evidence

| Scenario | Bot Behavior |
|---|---|
| Relevant knowledge found | Answer using knowledge. Cite via `[N]`. |
| No relevant knowledge (empty retrieval) | **Disclose**: "I don't have domain documentation on this topic." May offer a non-authoritative general answer clearly labeled: "Based on general knowledge (not from project documentation): ..." No fabricated citations. |
| Retrieval timed out | Same as empty retrieval — disclose unavailability. Do not mention the timeout itself, but do not silently fall back to an ungrounded answer. Label any general answer as non-authoritative. |
| Partial relevance | Use relevant chunks only. Cite only chunks used. If the question is only partially answerable from knowledge, state what is covered and what is not. |

**Key principle — no silent fallback**: An ungrounded answer must NEVER look equivalent to a grounded one. When domain knowledge is unavailable, the bot must disclose this. The disclosure distinguishes production-trustworthy answers from general-knowledge best-efforts.

#### Under conflicting or stale sources

| Scenario | Bot Behavior |
|---|---|
| Two sources conflict | Cite ALL conflicting sources with `[N]` references. Surface the conflict: "Sources [1] and [3] provide different information on this topic." Present both positions. Do not suppress either. |
| Newer contradicts older | Cite both sources. Recommend the newer as likely current: "[2] (newer) states X, while [1] (older) stated Y. The information may have been updated." Do not suppress the older source. |
| Source appears stale | Use the source, note age if relevant. Do not suppress stale sources — operator manages freshness, not the bot. |

**Key principle — transparent conflict**: The bot cites all relevant sources including conflicting ones. It may recommend the newer source but always surfaces the contradiction with full citations so the user can judge.

### 3. Citation Requirements

1. When using information from `<domain-knowledge>`, the bot MUST cite using `[N]` references.
2. When answering from general knowledge, the bot MUST NOT include `[N]` citations.
3. The bot SHOULD NOT use vague attribution ("according to docs") without a specific `[N]`.
4. Citation is best-effort — LLM may occasionally miss or miscite. The evaluation checklist catches systematic problems.

Implementation: Via SOUL.md / knowledge retrieval prompt template, not code enforcement. D3's injection already includes "Cite sources when using this information." D7 adds edge-case instructions.

### 4. Evaluation Process

1. **Pre-go-live**: Operator ingests client sample documents into test scope. Additionally ingest deliberately conflicting and stale documents for edge-case scenarios.
2. **Question set**: Operator prepares ≥ 20 domain questions with known answers, plus edge-case scenarios from the required scenario matrix above.
3. **Evaluation run**: Ask each question, record answer + citations + relevance. Run all 6 scenario types explicitly.
4. **Checklist review**: Verify against 8 criteria above. All hard requirements (100% thresholds) must fully pass.
5. **Go/no-go**: All criteria must pass. Hard requirements are non-negotiable. Failures → fix → re-evaluate.

## Alternatives Considered

### Quality bar

| Option | Verdict | Reason |
|---|---|---|
| Automated golden test set | Rejected for first release | No client sample data yet; automation premature |
| Manual evaluation with criteria | Accepted | Practical, achievable, sufficient for first client |
| No evaluation | Rejected | Production client requirement — cannot ship blind |

### Conflict behavior

| Option | Verdict | Reason |
|---|---|---|
| Silently prefer newest | Rejected | Hides conflict from user; violates transparency |
| Present both with citations | Accepted | Transparent; user can judge; citations provide traceability |
| Refuse to answer | Rejected | Over-cautious; unhelpful for production use |

### Citation enforcement

| Option | Verdict | Reason |
|---|---|---|
| Code-enforced citation validation | Rejected for first release | Would require post-processing LLM output; fragile |
| Prompt-based best-effort | Accepted | Practical; evaluation catches systematic failures |
| No citation requirements | Rejected | Provenance untraceable; violates D4 |

## Consequences

### Downstream constraints

- **#302**: Knowledge retrieval plugin prompt template must include D7 answer contract instructions (insufficient evidence, conflict, citation rules).
- **#306**: Must create evaluation checklist template with 8 criteria (including 3 hard 100% requirements), scenario matrix (6 edge-case types), pass/fail thresholds, test scope setup guide.
- **#307**: Must verify evaluation was actually run, all 8 criteria passed (3 hard requirements non-negotiable), scenario matrix fully covered, results documented. Must prove answer contract behaviors (unavailability disclosure, conflict surfacing, citation accuracy, zero fabrication).

### What D7 does NOT decide

- Chunking strategy optimization (requires real client data)
- Automated quality monitoring (future enhancement)
- Precision/recall statistical thresholds (requires larger evaluation corpus)
