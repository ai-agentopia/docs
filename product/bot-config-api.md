---
title: "Agentopia API Server"
description: "Central API server powering bot lifecycle management, multi-agent collaboration, and delivery orchestration for the Agentopia platform."
---


The Agentopia API Server is the control plane for the platform. It handles bot provisioning, inter-agent communication, collaborative reasoning threads, workflow orchestration, and credential governance -- all through a single, unified API surface.

## Bot Management

**AI-Powered Provisioning** -- Describe what you need in plain text and the API server generates a fully configured bot using LLM-driven config generation. Each bot receives a personality profile (SOUL), user context, and runtime configuration, then deploys automatically through the GitOps pipeline.

- **Natural language input**: Provide free-text requirements; the system produces structured bot configuration files
- **Role templates**: Pre-built templates for common roles -- DevOps, Backend, Frontend, Code Assistant, Research, and General purpose
- **Preview before deploy**: Generate and review bot configurations without committing to a full deployment
- **Automated deployment**: End-to-end provisioning from config generation through to a running pod, with step-by-step progress tracking
- **Fleet visibility**: List all managed bots with live status, provider, team grouping, and health information

## Inter-Agent Communication (Relay)

Bots can communicate directly with each other through the relay system, enabling collaborative problem-solving across specialized agents.

- **Direct delivery**: Synchronous request-response messaging between any two bots in the fleet
- **Async queuing**: Fire-and-forget mode for non-blocking communication with automatic queue fallback on timeout
- **Flexible addressing**: Route messages by bot name or Telegram username -- the relay handles name resolution transparently
- **Peer discovery**: Bots can query the registry to discover available peers and their capabilities
- **Authenticated routing**: Every relay message is authenticated against the sender's unique token

## Collaborative Reasoning (Threads)

Multi-turn brainstorm sessions where an orchestrator bot coordinates structured discussions among specialist agents.

- **Topic-scoped threads**: Create focused discussion threads with defined participants and turn limits
- **Turn-based coordination**: The orchestrator directs individual turns to specific bots, building a coherent multi-perspective discussion
- **Epoch checkpoints**: Automatic or manual summarization at configurable intervals, compressing long discussions into actionable summaries
- **Human-in-the-loop governance**: Checkpoint approval gates allow human review before threads continue
- **Idempotent turns**: Duplicate turn submissions return cached responses, ensuring reliability under retries
- **Conclusion with memory persistence**: Thread conclusions generate final summaries and persist key insights to each participant's long-term memory
- **Full transcript access**: Retrieve complete thread history including all turns and epoch summaries

## Delivery Orchestration (Workflows)

A structured workflow system that coordinates multi-step delivery processes across bot teams.

- **Role-based access**: Built-in roles (orchestrator, worker, reviewer) with support for custom role definitions
- **Actor binding**: Bind bots to workflow roles dynamically -- bindings are durable and survive restarts
- **Telegram integration**: Bots respond to `/wf` commands in Telegram for workflow status, gate approvals, and reporting
- **Automated setup**: Deploying a bot with a workflow role automatically binds the actor and registers Telegram commands

## Credential Governance (MCP)

Per-bot credential management for external tool integrations through the Model Context Protocol.

- **Scoped credentials**: Each bot has its own isolated set of credentials for external services (e.g., GitHub, Jira)
- **Tool-level access control**: Configure which MCP tools each bot can use, with explicit allow and deny lists
- **Secure storage**: Credentials are stored as platform secrets and never exposed in API responses
- **Runtime re-application**: Credential changes take effect automatically without manual restarts
- **Audit-friendly**: Credential presence and creation timestamps are queryable without revealing secret values

## Domain Knowledge (SA Knowledge Base)

Client-scoped knowledge bases that enable SA bots to answer from domain documents with provenance and citation.

- **Client-scoped isolation**: Knowledge is organized as `{client_id}/{scope_name}`. Each bot subscribes to specific scopes at creation time. Cross-client access is prevented by server-side scope resolution.
- **File-upload ingestion**: Operators upload PDF, HTML, markdown, text, or code files. Documents are chunked, embedded (Qdrant), and tracked with SHA-256 hashes and ingested_at timestamps.
- **Document lifecycle**: Same-content re-uploads are no-ops. Modified content triggers atomic two-phase replace. Deleted documents are tombstoned.
- **Runtime retrieval**: A gateway plugin auto-retrieves relevant chunks before LLM inference and injects them as cited context. The bot's answer contract forbids fabricated citations and requires unavailability disclosure.
- **Operator UI**: Client-first Knowledge Base page — browse clients, view scopes, upload documents, search, delete. Configured bot scopes appear immediately, even before first document upload.
- **Dual-path auth**: Write operations (upload, delete) require operator session. Read/search supports operator session or bot bearer token.

**Current status**: Implemented. Automated verification complete. Live pilot evaluation pending.

## Deployment Model

The API server operates as a GitOps-native service:

- **Declarative infrastructure**: All bot resources are managed through ArgoCD with automatic reconciliation
- **Zero-trust token handling**: Sensitive tokens are stored exclusively in-cluster and never committed to version control
- **Graceful degradation**: When external services are unavailable, affected provisioning steps are skipped rather than failed, allowing partial deployments in development environments
- **Continuous delivery**: Application code changes flow automatically through CI/CD to the running service

## Authentication

All mutating API operations require bearer token authentication. Each bot receives unique gateway and relay tokens at provisioning time, ensuring that inter-agent communication is authenticated end-to-end.
