---
title: "ADR-016: Off-Cluster Backup Target for Observability (Cloudflare R2)"
---

# ADR-016: Off-Cluster Backup Target for Observability (Cloudflare R2)

**Status**: **On Hold — execution deferred 2026-04-24.** Off-cluster backup was reviewed as optional at this stage. The decision (Cloudflare R2) is locked and the infra groundwork ([agentopia-infra#175](https://github.com/ai-agentopia/agentopia-infra/pull/175)) is merged and ready to activate, but R2 provisioning, the seed script run, CronJob apply/unsuspend, and the first restore rehearsal are explicitly not going to happen until this ticket is re-prioritized. See [agentopia-infra#168](https://github.com/ai-agentopia/agentopia-infra/issues/168) for the deferral context; accepted risk is node/disk loss → observability data loss up to the §8 retention ceiling. When this ADR moves back to active, no architectural changes are expected — the activation is a sequence of operator actions, not a re-decision.
**Date**: 2026-04-24
**Context**: [agentopia-infra#168](https://github.com/ai-agentopia/agentopia-infra/issues/168), [H3 Observability Production Design §9 + §14.5 criterion 4](../architecture/harness-control/h3-observability-production-design). Blocks promotion of the production design from Draft to Accepted.

---

## Decision

Use **Cloudflare R2** as the single off-cluster backup target for the observability subsystem.

All three streams land in the same bucket (separated by prefix):

- `agentopia-obs-backup/clickhouse/<yyyy>/<ww>/full/...` — weekly ClickHouse `BACKUP`
- `agentopia-obs-backup/clickhouse/<yyyy>/<mm>/<dd>/incr/...` — nightly `BACKUP INCREMENTAL`
- `agentopia-obs-backup/minio/<yyyy>/<mm>/<dd>/...` — weekly `mc mirror` of the Langfuse MinIO contents

Postgres backup coverage for the `langfuse` database is inherited from the shared system Postgres backup job and is **not** in scope of this ADR (scope clarification already posted on #168 on 2026-04-24).

---

## Why this decision is needed now

Production promotion criterion 4 of [§14.5 of the production design](../architecture/harness-control/h3-observability-production-design) requires an **off-cluster** backup target to be live before Langfuse exits Draft. Our current deployment in `agentopia-dev` (delivered by [infra#163](https://github.com/ai-agentopia/agentopia-infra/issues/163)) writes ClickHouse parts and MinIO blobs to node-local `local-path` volumes only — that is not a backup, it's a convenience copy that will not survive the single node the cluster runs on.

---

## Hard requirements from #168

| # | Requirement | How R2 satisfies it |
|---|---|---|
| 1 | Off-cluster (different failure domain from the observability cluster) | R2 is an entirely separate provider; different physical sites, different operator, different blast radius |
| 2 | Encryption at rest; keys in Vault | R2 has SSE enabled by default on every object. For belt-and-braces, `rclone crypt` with keys at `secret/langfuse/backup-encryption-key` can wrap payloads before upload (enable via config flag, defaults off to keep restore simple) |
| 3 | Retention aligned with §8 lifecycle (90d warm + 2y cold archive) | R2 lifecycle rules: `delete after 2y` for the archive prefix; no cold/deep-archive tier in R2, but at this volume the cost difference is negligible |
| 4 | Restore RTO: Postgres ≤ 30 min, ClickHouse ≤ 1 h, MinIO ≤ 30 min | Postgres is off-scope (shared system). At steady-state volume (§11.1: ~30k spans/day → ~15 GB compressed ClickHouse over 2y), an R2 → cluster pull at 100 Mbps finishes in 20 min — within the 1 h ClickHouse target with headroom |
| 5 | Cost bound | At steady state: **~$0.45/month** (storage); rehearsal egress free. See cost table below. |

---

## Candidate comparison

Options evaluated against the §11.1 volume profile (~30k spans/day, ~45k observations/day, ~10% of spans attach a MinIO blob averaging ~5 KB). Steady-state total footprint across both ClickHouse backup chain and MinIO mirror at 2-year retention: **~30 GB**.

| Option | Storage cost (30 GB) | Egress cost (quarterly restore rehearsal, ~30 GB full read) | S3-compatible? | Operational fit | Restore path |
|---|---|---|---|---|---|
| **Cloudflare R2** (recommended) | $0.015/GB/mo × 30 GB = **$0.45/mo** | **$0 — R2 has no egress fee** | Yes | Cloudflare already in Agentopia's operational context (Cloudflare Tunnel per [CLAUDE.md](https://github.com/ai-agentopia/agentopia-infra/blob/dev/CLAUDE.md) / [cloudflare-tunnel.md](../operations/cloudflare-tunnel)); no net-new vendor relationship | `rclone sync` / ClickHouse `BACKUP FROM S3(...)` |
| Backblaze B2 | $0.006/GB/mo × 30 GB = $0.18/mo | $0.01/GB but first 3× storage = ~90 GB free/mo → quarterly restore free, ad-hoc pulls may bill | Yes | Mature, simple, strong track record, zero Agentopia adoption today | `rclone` / S3 API |
| Wasabi | $6.99/TB/mo with **1 TB billing minimum** → **$7/mo for 30 GB of actual data** | Zero within fair use | Yes | 14× overkill on cost at this scale | `rclone` / S3 API |
| AWS S3 Standard | $0.023/GB/mo × 30 GB = $0.69/mo | $0.09/GB → ~$2.70 per 30 GB restore | Yes | Most features; most expensive egress | native Postgres WAL archiver, `mc`, ClickHouse `BACKUP TO S3` |
| Separate MinIO on different host | Hardware capital + ops overhead | Free within site | Yes | Requires a second site and second box. server36 is a single hardware site → same failure domain → **fails requirement 1** unless a genuinely off-site box exists | `mc mirror` |
| Hybrid (local secondary + cold provider) | Complexity not justified at 30 GB total | — | — | Over-engineered for this volume | — |

### Why R2 beats B2 (the two serious contenders)

At steady state B2 is slightly cheaper on storage ($0.18 vs $0.45 per month — a $3/year difference). The deciding factor is **operational** rather than $:

1. **Zero egress fees** — quarterly restore rehearsals and any ad-hoc historical pulls don't need to think about egress billing. B2's free-egress tier covers this today but caps at 3× storage per month, which gets thin as data grows.
2. **Cloudflare is already in the Agentopia operational surface** (Cloudflare Tunnel per the operations docs). Adding R2 reuses the existing account and billing relationship instead of opening a second vendor contract.
3. **S3-compat parity is equal** — `rclone`, `mc`, and ClickHouse's native `BACKUP TO S3(...)` all work against R2's S3 API with no special handling. Zero tooling cost to pick R2 over B2.

At 30 GB the $3/year difference is not decisive; reusing the existing vendor is.

---

## Credentials and Vault wiring

Mirror of the existing Langfuse DSN pattern (see [§5.4 of the production design](../architecture/harness-control/h3-observability-production-design) — Vault source of truth, K8s Secret operator-refreshed mirror):

| Target | Path | Keys |
|---|---|---|
| **Vault KV (source of truth)** | `secret/langfuse/backup-r2` | `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`, `R2_ENDPOINT` |
| **K8s Secret (runtime mirror in `agentopia-dev`)** | `langfuse-backup-r2-auth` | same keys |

Backup encryption key (optional `rclone crypt` wrapping layer):

| Target | Path | Keys |
|---|---|---|
| Vault KV | `secret/langfuse/backup-encryption-key` | `RCLONE_CONFIG_CRYPT_PASSWORD`, `RCLONE_CONFIG_CRYPT_PASSWORD2` |

A seed script `agentopia-infra/scripts/seed-langfuse-backup-vault.sh` will be added in the infra groundwork PR; the operator runs it once after provisioning the R2 account.

---

## Implementation sequence

1. **Operator** (approval required — see §Open approval):
   1. Provision Cloudflare R2 bucket `agentopia-obs-backup` (or equivalent name).
   2. Create an R2 API token with read/write on that bucket only.
   3. Run the seed script to populate Vault `secret/langfuse/backup-r2` and mirror the K8s Secret `langfuse-backup-r2-auth` in `agentopia-dev`.
2. **Infra** (pre-wired, mergeable groundwork PR — parallel to this ADR):
   1. `manifests/off-cluster-backup/clickhouse-backup-cronjob.yaml` — weekly `BACKUP TABLE ... TO S3(...)` + nightly `BACKUP INCREMENTAL`.
   2. `manifests/off-cluster-backup/minio-mirror-cronjob.yaml` — weekly `mc mirror` from in-cluster MinIO to R2.
   3. `scripts/seed-langfuse-backup-vault.sh` — Vault + K8s Secret seeding with R2 credentials.
   4. `operations/observability-backup-restore.md` — restore runbook outline.
3. **Activation** (after credentials exist):
   1. Apply the two CronJob manifests to `agentopia-dev`.
   2. Confirm the first nightly/weekly runs land objects in R2.
   3. Run the first restore rehearsal into `observability-restore-test` namespace.
   4. Measure actual RTO; record in the runbook.
4. **Quarterly ongoing**: restore rehearsal per §9.3 of the production design. Non-completion for two consecutive quarters is a production incident.

---

## Consequences

- Net-new vendor billing relationship: **none** — reuses the existing Cloudflare account.
- Net-new ops surface: an R2 bucket + an API token + two CronJobs + one restore runbook.
- **Retention cost ceiling**: 30 GB at $0.015/GB/month = $0.45/month = $5.40/year. Scales linearly with span volume; a 10× volume increase still costs under $5/month. This fits comfortably inside any steady-state platform budget.
- **Reversibility**: all three streams use S3-compatible APIs. Switching to B2, AWS S3, or a self-hosted MinIO is a credential swap, not a rewrite. This ADR is therefore a **deployment-level commitment**, not an architectural lock-in.
- **Vault dependency**: restore requires Vault to be up to retrieve R2 credentials. This inherits the existing Vault-is-load-bearing property Agentopia already accepts (gateway reads LLM API keys from Vault). No net-new risk.
- **Egress model risk**: if Cloudflare changes R2's zero-egress pricing, the monthly cost at this scale is still trivial even at AWS-S3-equivalent egress rates. The decision remains sound.

---

## Explicit non-goals

- Not a decision on **off-cluster backup for the shared system Postgres** — that is the shared system's responsibility and is verified (not implemented) by #168. Confirmation step documented in the runbook.
- Not a decision on **off-cluster backup for the cluster observability stack (LGTM)** — that is [agentopia-infra#64](https://github.com/ai-agentopia/agentopia-infra/issues/64), separate.
- Not a decision on **WAL-archive timings, retention policies beyond the §8 tiers, or cold-tier selection within R2** — R2 has no deep-archive tier and at this volume the cold/hot distinction doesn't pay back.

---

## Open approval

**The provider selection is recommended; the signup + credential provisioning is an operator action that is not within the scope of this ADR's author.** What requires explicit approval:

1. Use of the existing Cloudflare account for R2 (or creation of a dedicated R2 sub-account if desired for billing separation).
2. Provisioning the `agentopia-obs-backup` bucket and an R2 API token scoped to it.
3. Running the seed script (once the two items above exist) to populate Vault and the K8s Secret mirror.

Once (1)–(3) are in place, the pre-wired infra groundwork PR becomes activatable without further architectural change.

---

## References

- [H3 Observability Production Design §9 (backup contract)](../architecture/harness-control/h3-observability-production-design)
- [H3 Observability Production Design §14.5 (promotion criteria)](../architecture/harness-control/h3-observability-production-design)
- [agentopia-infra#168](https://github.com/ai-agentopia/agentopia-infra/issues/168) — this issue
- [ADR-015: Langfuse as self-hosted trace backend](./015-h3-02-trace-backend-langfuse) — related, for the subsystem being backed up
- [Cloudflare R2 pricing](https://developers.cloudflare.com/r2/pricing/) — verified 2026-04-24
- [Backblaze B2 pricing](https://www.backblaze.com/b2/cloud-storage-pricing.html) — verified 2026-04-24
