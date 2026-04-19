---
title: "GitHub → S3 sync proof-of-life"
description: "Live round-trip marker for agentopia-rag-platform issue #26. Safe to remove after proof."
---

# GitHub → S3 sync proof-of-life

This file is the initial end-to-end proof that the GitHub Actions workflow
`sync-architecture-to-s3.yml` puts new content into
`s3://utop-oddspark-document/architecture/` and that the Pathway pipeline
indexes it into Qdrant under scope `utop/oddspark`.

## Unique marker

`zebra-ionosphere-20260419`

Searching the live knowledge API for this marker must return this document
as a retrieval hit once the sync + indexing pipeline has run.

## What was tested

- A push to `main` that touches `architecture/**/*.md` triggers the
  workflow.
- The workflow syncs `architecture/` → `s3://utop-oddspark-document/architecture/`
  with `--delete` enabled.
- Pathway's 30 s S3 poll discovers this new object, parses it as markdown,
  chunks it, embeds the chunks, and upserts them into collection
  `kb-45294aa854667d3b`.
- The knowledge-api retrieves the chunks via search; the marker string is
  the easy-to-verify signal that the round trip succeeded.

## Safe to remove

This file is a disposable test marker. Removing it in a subsequent commit
exercises the delete path of the sync (`aws s3 sync --delete` →
Pathway retracts the chunk rows → Qdrant points disappear).
