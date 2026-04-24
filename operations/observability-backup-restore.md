---
title: "Observability Backup & Restore Runbook"
---

# Observability Backup & Restore Runbook

**Scope**: ClickHouse and MinIO backup + restore procedures for the Langfuse observability subsystem. Off-cluster target: **Cloudflare R2** ([ADR-016](../adrs/016-h3-off-cluster-backup-cloudflare-r2)).

**Out of scope**: Postgres (the `langfuse` database) — that is the shared system Postgres's backup responsibility, not this subsystem's.

**Status**: **Draft — first version**. RTOs below are architectural targets from [H3 Production Design §9.1](../architecture/harness-control/h3-observability-production-design); the first restore rehearsal records actual measured values into this document.

---

## 1. What gets backed up

| Stream | What | Cadence | Target prefix in R2 |
|---|---|---|---|
| ClickHouse full | All event tables via `BACKUP TABLE ... TO S3(...)` | Weekly (Sunday 03:00 UTC) | `clickhouse/<yyyy>/<ww>/full/` |
| ClickHouse incremental | Changes since last full via `BACKUP TABLE ... TO S3(...) SETTINGS incremental = true` | Nightly (03:00 UTC) | `clickhouse/<yyyy>/<mm>/<dd>/incr/` |
| MinIO | Full mirror via `mc mirror --overwrite` from in-cluster MinIO bucket | Weekly (Sunday 04:00 UTC) | `minio/<yyyy>/<mm>/<dd>/` |

Retention in R2: lifecycle rule deletes objects older than **2 years** (§8 cold-archive ceiling). No Glacier/deep-archive tier — R2 has no such tier; at this volume the cost difference is negligible.

## 2. Prerequisites (one-time)

- [ ] Cloudflare R2 bucket provisioned: `agentopia-obs-backup` (or chosen equivalent).
- [ ] R2 API token created, scoped to that bucket only, with read+write.
- [ ] Vault path `secret/langfuse/backup-r2` populated with `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`, `R2_ENDPOINT`.
- [ ] K8s Secret `langfuse-backup-r2-auth` mirrored into `agentopia-dev` (via `agentopia-infra/scripts/seed-langfuse-backup-vault.sh`).
- [ ] CronJob manifests applied (`manifests/off-cluster-backup/clickhouse-backup-cronjob.yaml` and `minio-mirror-cronjob.yaml`).
- [ ] First nightly + weekly runs observed to land objects in R2 (check in the Cloudflare dashboard or `rclone ls`).

## 3. Restore procedure (hot path — actually restoring a lost subsystem)

> **Read before starting.** Restore restores the observability subsystem. The Langfuse `langfuse` database on the shared system Postgres is **not** restored here — coordinate with the shared system team if that database is also lost.

### 3.1 Restore into `observability-restore-test` first

Never restore into the live `agentopia-dev` namespace without first validating the restore payload in an isolated namespace.

1. Create the isolated namespace: `kubectl create ns observability-restore-test`.
2. Copy the four Langfuse subchart secrets into the isolated namespace (encryption key, ClickHouse password, Redis password, MinIO creds). They must match what's in `agentopia-dev` for the restored data to decrypt.
3. Apply a copy of the Langfuse ArgoCD Application pointing at the restore namespace (or run `helm template` and apply directly).
4. Wait for the empty ClickHouse + MinIO pods to become Ready.

### 3.2 ClickHouse restore

Inside the target ClickHouse pod (`kubectl exec`):

```sql
-- Full restore from the most recent weekly full:
RESTORE TABLE default.* FROM S3('<R2-endpoint>/<bucket>/clickhouse/<yyyy>/<ww>/full/', '<access-key>', '<secret-key>');

-- Then replay incremental backups since the full, in chronological order:
RESTORE TABLE default.* FROM S3('<R2-endpoint>/<bucket>/clickhouse/<yyyy>/<mm>/<dd>/incr/', ...);
-- ... for each day since the full.
```

**Target RTO**: ≤ 1 h for the whole ClickHouse restore. **Actual RTO**: (recorded after first rehearsal).

### 3.3 MinIO restore

From a pod with `mc` installed, or an ad-hoc `rclone` pod:

```bash
mc alias set r2 "${R2_ENDPOINT}" "${R2_ACCESS_KEY_ID}" "${R2_SECRET_ACCESS_KEY}"
mc alias set tgt "http://langfuse-s3.<restore-ns>.svc.cluster.local:9000" "<new-minio-root-user>" "<new-minio-root-pw>"
mc mirror --overwrite r2/agentopia-obs-backup/minio/<yyyy>/<mm>/<dd>/ tgt/langfuse/
```

**Target RTO**: ≤ 30 min for MinIO. **Actual RTO**: (recorded after first rehearsal).

### 3.4 Post-restore validation

- [ ] ClickHouse row counts within expected range for the restore date.
- [ ] MinIO object count reconciles against ClickHouse's references (spot check via Langfuse UI on the restore namespace — open a trace that attaches a MinIO blob).
- [ ] New trace ingest to the restored Langfuse correlates with historical traces — the Langfuse UI should show both pre-restore and post-restore runs in the same project.

If all checks pass and the live namespace is confirmed unrecoverable, copy the restored data into the live namespace (mirror the reverse of §3.3), then delete the restore-test namespace.

## 4. Quarterly rehearsal procedure

Per [H3 Production Design §9.3](../architecture/harness-control/h3-observability-production-design): quarterly restore rehearsal against `observability-restore-test`. Non-completion for two consecutive quarters is a **production incident**.

Rehearsal checklist:

- [ ] Create `observability-restore-test` namespace.
- [ ] Restore latest full + incrementals per §3.2 and §3.3.
- [ ] Record actual RTO measurements for each stream.
- [ ] Verify §3.4 post-restore checks pass.
- [ ] Delete the restore-test namespace.
- [ ] Log the rehearsal date + RTO numbers in this runbook (append to §6 history).

## 5. Failure modes + escalation

| Failure | Detection | Response |
|---|---|---|
| CronJob didn't run | ArgoCD / Prometheus CronJob alert | Inspect pod logs; re-run manually if transient; if R2 is unreachable escalate to Cloudflare status page |
| Restore can't decrypt payload | `rclone`/ClickHouse reports decryption error | Verify encryption key in Vault matches what wrote the backup (rotation note in ADR-016 — if the encryption key was rotated, the old key is still at `secret/langfuse/backup-encryption-key-previous` for one rotation cycle) |
| Restore exceeds target RTO | Measured at rehearsal | Investigate: network bandwidth from R2 region to cluster, ClickHouse RESTORE parallelism, R2 LIST latency on large prefix. Record in §6. |
| Vault outage during restore | Vault pod not Ready | Restore from Vault snapshot first (shared cluster Vault has its own backup surface); this is the known single-point-of-failure explicitly accepted in H3 production design §9.4 |

## 6. Rehearsal history

| Date | Who | Streams restored | Actual RTO (CH / MinIO) | Notes |
|---|---|---|---|---|
| _(first rehearsal pending — run after CronJobs have produced their first full cycle)_ | | | | |

## 7. Confirmation: shared-system Postgres covers the `langfuse` database

Not implemented here, but part of the promotion gate per [§14.5 criterion 4](../architecture/harness-control/h3-observability-production-design):

- [ ] Confirm the shared system Postgres nightly dump / WAL archive includes the `langfuse` database.
- [ ] Confirm restore has been exercised for at least one shared-system database within the last quarter.
- [ ] Record the confirmation in this document once done (below).

_Confirmation: pending._

---

## References

- [ADR-016: Off-cluster backup target (Cloudflare R2)](../adrs/016-h3-off-cluster-backup-cloudflare-r2)
- [H3 Production Design §9 (backup contract), §14.5 (promotion criteria)](../architecture/harness-control/h3-observability-production-design)
- [agentopia-infra#168](https://github.com/ai-agentopia/agentopia-infra/issues/168) — tracking issue
