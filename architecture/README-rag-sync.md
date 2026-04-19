---
title: "RAG sync contract — architecture/ → Agentopia knowledge base"
description: "How files in architecture/ flow into the Agentopia RAG platform and what the GitHub → S3 workflow guarantees."
---

# RAG sync contract

Files under `architecture/` in this repository are the canonical corpus for
the Agentopia knowledge base (`scope: utop/oddspark`). This document
describes what gets synced, how, and what lives outside this path.

## What this sync covers (RAG-platform #26)

| Property | Value |
|---|---|
| Source | `architecture/**/*.md` on branch `main` |
| S3 bucket | `utop-oddspark-document` |
| S3 region | `ap-northeast-1` |
| S3 prefix | `architecture/` |
| Key mapping | `architecture/{path relative to repo root}` — 1:1 with repo |
| Ingest scope | `utop/oddspark` |
| Qdrant collection | `kb-45294aa854667d3b` |
| Trigger | push to `main` touching `architecture/**/*.md` or the workflow file; also `workflow_dispatch` |
| Delete semantics | `aws s3 sync --delete` — file removed in repo → object removed in S3 → Pathway retracts chunks from Qdrant |
| Rename semantics | Treated as delete + add (new `document_id` in Qdrant payload) |
| Freshness | Pathway polls S3 every 30 s; expect chunks in Qdrant within 1–2 poll cycles of the Action completing |

## What this sync does NOT cover

- **GitHub issues, pull requests, commits, review comments** — ingested via
  Airbyte under a separate initiative (RAG-platform #27). That structured
  stream never flows through this workflow.
- **Binary formats (`.docx`, `.pdf`)** — the Pathway pipeline currently uses
  `format="plaintext_by_object"`. Binary parsing (`format="binary"` +
  UnstructuredParser) is a pending pipeline upgrade; until then this
  workflow deliberately filters to `*.md` only.
- **Any content outside `architecture/`** — the include pattern is explicit
  and will not leak sibling trees into S3.

## Secret contract

The workflow requires the following GitHub Actions repository secrets:

| Secret | Purpose |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key for an IAM principal that can `PutObject`, `DeleteObject`, and `ListBucket` on `arn:aws:s3:::utop-oddspark-document/architecture/*` |
| `AWS_SECRET_ACCESS_KEY` | Matching secret |

The minimal IAM policy is documented in `agentopia-rag-platform/docs/github-ingestion-sync.md`.

## Operator runbook

1. **Editing**: edit any `architecture/*.md` file on a feature branch, open
   a PR, merge to `main`.
2. **Verify rollout**: the "Sync architecture docs to RAG S3" Action runs
   automatically. Its job summary lists the exact objects put/deleted.
3. **Confirm indexing**: within ~1 minute the updated content is searchable
   in the knowledge UI for client `utop`, scope `oddspark`. Retrieval via
   `GET /api/v1/knowledge/utop--oddspark/documents` shows the refreshed
   `ingested_at` timestamp.
4. **Manual resync**: run the workflow via `Actions → Sync architecture docs
   to RAG S3 → Run workflow` when you need to replay the current tree without
   touching the repo (e.g. after a Pathway restart).

## Change control

Include/exclude policy is intentionally narrow. Extending the sync to new
file types or directories requires:

1. A pipeline change in `agentopia-rag-platform` confirming the new format
   is parsed correctly (e.g. binary-mode parser for DOCX).
2. An update to both this contract document and the workflow's
   `--include` patterns.
3. Live re-verification that sync produces retrievable content, not
   garbage chunks.
