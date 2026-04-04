---
title: "Roadmap"
description: "Agentopia platform — what's shipped, what's next, and what's planned"
---

# Roadmap

What Agentopia can do today, and where it's headed.

> **Legend**: Shipped = production-ready, verified. In Progress = active development. Approved = planned and scoped, not yet started. Planned = future intent, no active work.

---

## Shipped

### Multi-Bot Orchestration

Deploy and manage specialized AI agents — each with distinct personality, role (Orchestrator / Worker / Reviewer), and persistent memory. Bots communicate via the A2A (Agent-to-Agent) JSON-RPC protocol with debate, bridge, and thread-based multi-turn patterns.

- Role-based authority contracts governing tool access per agent
- Agent card publishing at `/.well-known/agent-card.json` with SSRF-protected external discovery
- Per-agent semantic memory (Qdrant + Neo4j via mem0-api) and file-based workspace (SOUL, USER, session history)

### AI Delivery Orchestration

Temporal-powered durable workflows: plan → dispatch → develop → review → merge.

- LangGraph planner decomposes objectives into structured work packets
- Workers autonomously create branches, write code, and submit pull requests
- Reviewer agents inspect PRs, submit structured reviews, approve or request changes
- Rework loops: reviewer feedback triggers fix cycle — worker receives comments, patches code, resubmits
- Trello planning projection: one-way sync from workflow state to Trello boards

### Governed PR Review

Bot-based code review with policy enforcement, remediation suggestions, and finding closure tracking.

- Control-plane intake with deduplication and reviewer binding (workflow / repo-bound modes)
- `.agentopia/review-policy.yml` loaded per-repo — security, schema, and custom lanes
- Reviewer bot executes full review via A2A, reports findings through completion callback
- Fix commands and safe-fix registry for automated remediation suggestions
- Delta re-review: finding reconciliation, closure audit trail, stale finding detection
- Execution authorization matrix: 29 governance tools classified, role + context enforcement

### Web Application

React-based unified web app (P1 milestone, closed 2026-03-30).

- Dual-lane product: Communication (chat with any bot via WebSocket) + Workflow (start deliveries, track phases, view artifacts)
- Bot management: create, deploy, stop, start, restart, delete through the UI
- Role-based UI gating: orchestrator bots show workflow controls, worker/reviewer bots show read-only views
- Multi-provider model selection: Anthropic, OpenAI, OpenRouter, Fireworks — per-bot provider/model editable after deploy
- Workflow cancel with structured cleanup: operator-initiated cancel, honest partial-cleanup reporting, durable cleanup result persisted in workflow detail
- Delivery target validation: fail-closed allowlist, GitHub slug format enforcement, source issue reuse
- Canceled workflow timeline: reflects actual last phase reached (not false full-completion)

### Infrastructure & Operations

- GitOps via ArgoCD with automated image promotion per environment
- Per-environment CI/CD: separate image tags per branch (dev/uat), automatic build and deploy
- Prometheus observability: metrics, SLO alerts, dashboards for A2A protocol, LLM proxy, and delivery workflows
- Custom Rust LLM proxy with 8+ provider backends, dynamic provider loading from Vault secrets

---

## In Progress

### SA Knowledge Base — Domain & Project Intelligence

SA bots use client-provided documents as a reliable working brain — structured knowledge with provenance, citation, and governance. Implementation complete, pending live pilot.

| Capability | Status |
|---|---|
| Client-scoped knowledge model (`{client_id}/{scope_name}`) | Implemented |
| File-upload ingestion (PDF, HTML, markdown, text, code) | Implemented |
| Runtime retrieval injection (gateway plugin, priority 10) | Implemented |
| Provenance & citation (source, section, page, score) | Implemented |
| Operator Knowledge UI (client-first, scope browser) | Implemented |
| Governance & dual-path auth (operator session + bot bearer) | Implemented |
| Evaluation framework (ADR-014: 8 criteria, 6 scenarios) | Implemented |
| Live pilot evaluation (#307) | Pending — first client |

**295+ tests. Architecture locked (ADRs 008-014).** Tracking: [SA-KB Milestone](milestones/production-sa-knowledge-base)

---

## Approved — Not Yet Started

### Super RAG — Production-Grade Retrieval

Upgrade retrieval from basic vector search to production-grade: hybrid search, evaluation, observability, and knowledge service extraction. Builds on SA-KB foundation.

| Phase | What |
|---|---|
| **0** | Foundation hardening — plugin config, retry/circuit breaker, health checks |
| **1a** | RAGAS evaluation early signal (reference-free, directional) |
| **1b** | Labeled evaluation baseline (nDCG@5, MRR, golden dataset from pilot) |
| **2a** | Hybrid retrieval — dense + sparse TF + RRF fusion via Qdrant |
| **2b** | Knowledge-API service extraction (monorepo, proxy-first auth) |

**Planning complete. [Milestone #34](https://github.com/ai-agentopia/agentopia-protocol/milestone/34), 10 issues.** Tracking: [Super RAG Milestone](milestones/production-super-rag) | [Architecture Debate](architecture/super-rag-debate)

---

## Planned

Items below have GitHub milestones and/or design docs but no active execution.

| Area | Description | Reference |
|---|---|---|
| **Worker Pool Routing** | Capability-aware dispatch — frontend tasks to frontend workers, backend to backend, sticky rework | [Design doc](architecture/worker-pool-routing-improvement-plan) |
| **Multi-Step Planning with Human Approval** | Operator approval checkpoints between planning steps | Wave F milestone |
| **Streaming & Real-Time Notifications** | Stream workflow progress and LLM output to operators in real-time | Wave F milestone |
| **Advanced Multi-Agent Collaboration** | Parallel sub-agents, LLM-driven routing, multi-packet delivery, collaborative review | Wave G milestone |
| **Intelligence Layer (MCP + Tools)** | MCP tool servers and codebase-aware integrations | M3 milestone |
| **Observability & Cost Management** | End-to-end token tracking, cost attribution per bot/workflow, budget alerts | M5 milestone |
| **Governance & Compliance** | OAuth2/OIDC, audit logging, RBAC for multi-tenant, compliance reporting | M7 milestone |
| **Marketplace & SaaS** | Agent template marketplace, multi-tenant isolation, subscription management | M8 milestone |

---

## Recently Completed

| Milestone | Date | Summary |
|---|---|---|
| **Workflow Cancel + Cleanup** | 2026-04-04 | Operator-initiated workflow cancel with structured cleanup, durable partial-cleanup reporting, honest Temporal divergence handling |
| **Delivery Target Validation** | 2026-04-04 | Fail-closed repo allowlist, slug format enforcement, source issue reuse for workflow retry |
| **Canceled Timeline Fix** | 2026-04-04 | Canceled workflows reflect actual last phase reached, not false full-completion |
| **Dispatch Timeout + Retry** | 2026-04-04 | 10-minute dispatch timeout (up from 5), timeout treated as retryable instead of terminal failure |
| **P1 — Web-App Primary Dual-Lane MVP** | 2026-03-30 | Web app as primary operator path. Dual-lane: Communication + Workflow. [Tracking](milestones/p1-web-app-primary-dual-lane-mvp) |
