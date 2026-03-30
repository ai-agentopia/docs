---
title: "Automated Code Review Workflow"
description: "How Agentopia's bot-based governed review workflow automates pull request reviews with policy awareness, incremental re-review, and full auditability."
---


Agentopia provides automated pull request reviews through a governed bot workflow. A dedicated reviewer bot inspects code changes, applies repository-specific review policies, and publishes structured feedback directly on the pull request.

---

## How It Works

When a pull request is opened or updated, the platform receives the event and orchestrates a review through these steps:

1. **Event intake** — The platform receives the PR event, validates the repository is onboarded, and deduplicates repeated triggers.
2. **Review run creation** — A persistent review run is created, linked to any prior reviews on the same pull request.
3. **Context assembly** — The platform fetches changed files, applies the repository's review policy, and packages everything the reviewer needs.
4. **Bot dispatch** — The review package is sent to a QA reviewer bot bound to that repository.
5. **Review execution** — The reviewer bot inspects the code, reasons over changes against the review policy, and publishes its review on GitHub.
6. **Completion verification** — The platform confirms the review was posted and records the outcome.

```
PR Event → Intake → Run Tracking → Context Assembly → Reviewer Bot → GitHub Review → Verification
```

---

## Reviewer Bot

The reviewer bot is the execution actor. It receives a structured review package from the platform and uses governed tools to:

- Read changed files and their full context
- Evaluate changes against the repository's review policy
- Organize findings by review concern (security, schema changes, API behavior, operations)
- Publish a single, concise review with a clear verdict

The bot operates within scoped permissions. It can read repository contents and submit reviews, but cannot approve, merge, or modify code.

---

## Repository Review Policy

Each onboarded repository includes a review policy that shapes how the bot conducts its review. The policy can specify:

- **Technology stack context** so the reviewer understands the codebase
- **Paths to ignore** (generated files, vendored dependencies)
- **High-risk patterns** that warrant extra scrutiny
- **Severity guidance** for different types of findings
- **Review focus areas** to prioritize what matters most
- **Comment limits** to keep feedback actionable, not noisy

The platform loads this policy and injects it into the review package before dispatching to the bot.

---

## Incremental Re-Review

When a pull request is updated after a review, the platform detects the new push and triggers a re-review. The re-review includes:

- A summary of prior findings from the previous review
- Only the new or changed files since the last review
- Context about which prior issues were addressed

This prevents the reviewer from repeating itself and focuses feedback on what actually changed.

---

## What Makes This Different

| Capability | Description |
|---|---|
| **Governed permissions** | The reviewer bot operates within a defined role with scoped access to repository tools |
| **Policy-aware review** | Reviews are shaped by each repository's specific policy, not generic rules |
| **Multi-concern review** | Findings are organized by concern area (security, schema, API, ops) for clarity |
| **Incremental re-review** | PR updates trigger focused follow-up reviews linked to prior findings |
| **Full auditability** | Every review run is persisted with status, verdict, and traceability |
| **Low-noise output** | Structured synthesis with comment limits produces actionable feedback, not walls of nitpicks |

---

## What This Is Not

- **Not an auto-approve system** — the bot reviews, humans decide
- **Not a style linter** — it focuses on meaningful concerns, not formatting
- **Not a replacement for human review** — it augments the process and catches what humans miss
- **Not a generic comment bot** — reviews follow governed policy with structured output

---

## Architecture Summary

The workflow separates concerns into two planes:

**Platform (control plane):**
- Event intake and deduplication
- Repository onboarding and reviewer binding
- Review policy loading and context packaging
- Dispatch, completion verification, and audit

**Reviewer bot (execution plane):**
- Code inspection using governed tools
- Reasoning over changes and policy
- Structured finding organization
- Review publication on GitHub

This separation ensures the platform handles orchestration and tracking while the reviewer bot handles the actual code analysis. The platform never bypasses the bot to review code directly.

---

## Repository Integration

To onboard a repository:

1. Add a review policy file to the repository
2. Configure a GitHub Action to send PR events to the platform
3. The platform binds the repository to a reviewer bot

Once onboarded, every qualifying pull request is automatically reviewed with no further configuration needed.
