---
title: "Roadmap"
description: "Agentopia platform capabilities — shipped and planned"
---


What Agentopia can do today, and where it's headed.

---

## Shipped

### Core Platform

| Feature | Description |
|---|---|
| **Multi-Bot Orchestration** | Deploy and manage multiple AI agents, each with distinct personality, role, and memory |
| **Role-Based Agent System** | Three canonical roles — Orchestrator, Worker, Reviewer — with authority contracts governing tool access |
| **A2A Protocol** | Agent-to-agent communication via JSON-RPC with thread-based multi-turn conversations, debate, and bridge skills |
| **Semantic Memory** | Per-agent memory powered by Qdrant (vector) + Neo4j (graph) via mem0-api |
| **Persistent File Memory** | Per-agent workspace with SOUL, USER, and session history on persistent storage |

### AI Delivery Orchestration

| Feature | Description |
|---|---|
| **Durable Delivery Workflows** | Temporal-powered workflow orchestration: plan → dispatch → develop → review → merge |
| **LLM-Powered Planning** | LangGraph planner decomposes objectives into structured work packets with scope and acceptance criteria |
| **Automated Code Delivery** | Workers autonomously create branches, write code, and submit pull requests |
| **Automated Code Review** | Reviewer agents inspect PRs, submit structured reviews, and approve or request changes |
| **Rework Loops** | Reviewer feedback triggers rework — worker receives specific comments, fixes code, resubmits |
| **Governed PR Review** | Bot-based review with policy enforcement, remediation suggestions, and finding closure tracking |
| **Trello Planning Projection** | One-way sync from workflow state to Trello boards for visual project tracking |

### Web Application

| Feature | Description |
|---|---|
| **Unified Web App** | React-based UI with bot sidebar, Communication + Workflow tabs, login/session management |
| **Communication Lane** | Chat with any bot via WebSocket streaming, conversation persistence, markdown rendering |
| **Workflow Lane** | Start deliveries, track phase timeline, view artifacts (branches, PRs), monitor progress |
| **Bot Management** | Create, deploy, stop, start, restart, and delete bots through the UI |
| **Role-Based UI Gating** | Orchestrator bots show workflow start; worker/reviewer bots show read-only workflow views |
| **Multi-Provider Model Selection** | Choose from multiple LLM providers (Anthropic, OpenAI, Google, Fireworks) per bot |

### Infrastructure & Operations

| Feature | Description |
|---|---|
| **GitOps Deployment** | ArgoCD-managed infrastructure with automated image promotion per environment |
| **Per-Environment CI/CD** | Separate image tags per branch (dev/uat) with automatic build and deploy |
| **Agent Discovery** | Agent card publishing and external agent card fetching with SSRF protection |
| **Execution Authorization** | Governance-enforced tool access — workers can only write code through authorized workflow dispatch |
| **Prometheus Observability** | Metrics, SLO alerts, and dashboards for A2A protocol, LLM proxy, and delivery workflows |

---

## In Progress

### Super RAG — Production-Grade Retrieval

Upgrade RAG from basic vector search to production-grade retrieval with hybrid search, evaluation, observability, and service extraction. Builds on the SA Knowledge Base foundation.

| Capability | Status | Description |
|---|---|---|
| **Phase 0: Foundation Hardening** | Planned | Plugin config fix, retry/circuit breaker, health checks, env var exposure |
| **Phase 1a: RAGAS Early Signal** | Planned | Reference-free evaluation (Faithfulness, Context Precision), CI regression gate |
| **Phase 1b: Labeled Baseline** | Blocked | Golden dataset from #307 pilot, nDCG@5/MRR/Precision@5, authoritative eval gate |
| **Phase 2a: Hybrid Retrieval** | Planned | Qdrant native BM25 + vector + RRF fusion, labeled eval gate (nDCG@5 ≥ 10%) |
| **Phase 2b: Knowledge-API Extraction** | Planned | Service extraction, proxy-first auth, binding sync/reconcile, topology gate |

**Status**: Planning complete. Milestone #34 created. 10 issues (#316-#325). Execution pending.

Tracking doc: [Super RAG Milestone](milestones/production-super-rag)

### SA Knowledge Base for Domain & Project Intelligence

SA bots use client-provided domain and project knowledge as a reliable working brain — not just conversation memory, but structured document knowledge with provenance, citation, and governance.

| Capability | Status | Description |
|---|---|---|
| **Client-scoped knowledge model** | Implemented | `{client_id}/{scope_name}` identity, server-side scope resolution, cross-client isolation |
| **File-upload ingestion** | Implemented | PDF, HTML, markdown, text, code — with SHA-256 hashing, two-phase replace, tombstone lifecycle |
| **Runtime retrieval injection** | Implemented | Gateway plugin (priority 10), auto-injects domain context before LLM inference |
| **Provenance & citation** | Implemented | Bot answers cite [N] source documents with filename, section, ingested_at timestamp |
| **Operator Knowledge UI** | Implemented | Client-first navigation, scope browser, document upload/delete, search |
| **Governance & auth** | Implemented | Operator session auth for write, bot bearer for read, dual-path access control |
| **Evaluation framework** | Implemented | 8 criteria (3 hard at 100%), 6 scenario types, answer contract (ADR-014) |
| **Live pilot evaluation** | Pending | #307 OPEN — requires live bot + client documents for final quality gate |

**Status**: Implemented. Automated verification complete (295+ tests). Architecture locked (ADRs 008-014). Live pilot evaluation pending — #307 open.

Tracking doc: [SA Knowledge Base Milestone](milestones/production-sa-knowledge-base)

---

## Recently Completed

### P1 — Web-App Primary Dual-Lane MVP (Closed 2026-03-30)

Web app is the primary operator path. Dual-lane product: Communication (chat) + Workflow (delivery). Three execution boundaries enforced. Workflow conversation with live bot reply.

Tracking doc: [P1 Milestone Trace](milestones/p1-web-app-primary-dual-lane-mvp)

---

## Planned

### Worker Pool Routing

Evolve from generic worker dispatch toward capability-aware pool routing — frontend tasks to frontend workers, backend tasks to backend workers, with sticky rework continuity.

Design doc: [Worker Pool Routing Improvement Plan](architecture/worker-pool-routing-improvement-plan)

### Multi-Step Planning with Human Approval

Enable operator approval checkpoints between planning steps. Pause for human review before committing to a delivery plan.

### Streaming & Real-Time Notifications

Stream workflow progress and LLM output to operators in real-time. Workflow phase notifications to user channels.

### Advanced Multi-Agent Collaboration

Complex graph patterns with parallel sub-agents, LLM-driven routing, multi-packet parallel delivery, and multi-agent collaborative conversation during review.

### Intelligence Layer (MCP + Tools)

Extend agent capabilities through MCP tool servers and tool-based integrations for codebase-aware responses.

### Observability & Cost Management

End-to-end token tracking across all LLM calls, cost attribution per bot and per workflow, usage dashboards and budget alerts.

### Governance & Compliance

OAuth2/OIDC authentication, audit logging, role-based access control for multi-tenant environments, and compliance reporting.

### Marketplace & SaaS

Agent template marketplace, multi-tenant workspace isolation, subscription management, and self-service onboarding.
