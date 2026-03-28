---
title: "ADR-005: Relay Deprecation Timeline"
---

# ADR-005: Relay Deprecation Timeline

**Status:** Accepted
**Date:** 2026-03-11
**Context:** M1.9 A2A Adoption — relay→A2A migration plan

## Context

Agentopia currently supports two message delivery paths:

1. **Relay** (legacy): Gateway pods send messages directly via HTTP relay (`relay_direct`) or async queue (`relay_queue`) to bot-config-api. No LLM involvement in the delivery path.
2. **A2A** (new, M1.6+): Gateway pods send messages via the A2A JSON-RPC protocol through bot-config-api's mounted A2A endpoints. Includes LLM-powered debate, bridge, and epoch capabilities.

The relay path was the original delivery mechanism. A2A was introduced in M1.6 and hardened in M1.8 (Postgres task store, Redis cache, circuit breaker, rate limiter). M1.9 added adoption metrics, Grafana dashboards, and PrometheusRule alerts to track the migration.

Both paths currently coexist, with A2A taking priority when `RELAY_A2A_ENABLED=true` (default) and the bot has `a2aRole != "none"`. Relay serves as automatic fallback when A2A fails.

## Decision

Deprecate the relay delivery path in 3 phases over ~8 weeks, gated by metric thresholds. Each phase requires explicit operator approval before proceeding.

## Phases

### Phase 1: Shadow Mode (Weeks 1–2)

**Goal:** Validate A2A handles 100% of traffic with relay as silent fallback.

**Configuration:**
- `RELAY_A2A_ENABLED=true` (default)
- All bots: `a2aRole: "participant"`
- Relay code remains active as fallback

**Gate criteria to exit Phase 1:**
- `a2a_ratio >= 0.95` for all bots over 7 consecutive days
- No `A2AHighRelayFallbackRate` alerts fired in 7 days
- A2A delivery p95 latency < 10s sustained
- Zero data loss (all messages delivered, verified via thread transcripts)

**Monitoring:**
- Grafana dashboard "A2A Adoption (M1.9)" — daily review
- `GET /api/v1/metrics/a2a-adoption` — per-bot breakdown
- Fallback reason distribution should be empty or near-zero

### Phase 2: Relay Disabled (Weeks 3–6)

**Goal:** Run without relay fallback to prove A2A is self-sufficient.

**Configuration:**
- `RELAY_A2A_ENABLED=true` (A2A primary)
- Add `RELAY_FALLBACK_ENABLED=false` (new env var, disables relay fallback on A2A failure)
- Relay code still present in codebase (not removed)

**Gate criteria to exit Phase 2:**
- `a2a_ratio == 1.0` for all bots over 14 consecutive days
- No `A2ADeliveryCircuitBreakerOpen` alerts in 14 days
- No manual rollbacks triggered
- At least one LLM provider incident occurred and A2A recovered automatically (circuit breaker + fallback model)

**Rollback:** If any gate fails, revert to Phase 1 (`RELAY_FALLBACK_ENABLED=true`) and investigate.

### Phase 3: Relay Code Removal (Weeks 7–8)

**Goal:** Remove relay delivery code paths, simplify codebase.

**Actions:**
1. Remove `relay_direct` and `relay_queue` delivery paths from `relay.py`
2. Remove `RELAY_A2A_ENABLED` and `RELAY_FALLBACK_ENABLED` env vars
3. Remove `agentopia_delivery_total{path="relay_*"}` metric labels (keep `path="a2a"`)
4. Update Grafana dashboard: remove relay panels, simplify to A2A-only view
5. Update PrometheusRule: remove relay ratio alert (no longer applicable)
6. Update runbook: remove relay-specific procedures
7. Archive this ADR as completed

**Gate criteria to proceed:**
- Phase 2 gate fully passed
- Team sign-off on code removal PR
- No relay-specific code references remain in test suite

**No rollback** after Phase 3 — relay code is deleted. If A2A fails post-removal, the fix is forward (fix A2A), not backward (restore relay).

## Consequences

### Positive
- Simplified delivery path — single code path easier to maintain and debug
- Reduced metric cardinality (no `path` label variations)
- A2A enables LLM-powered features (debate, bridge, epoch) that relay cannot provide
- Circuit breaker and rate limiter provide better resilience than raw relay

### Negative
- Loss of simple relay fallback — A2A failures have no safety net after Phase 3
- A2A depends on LLM proxy availability — adds a dependency relay didn't have
- Single bot-config-api replica is SPOF for A2A (relay had same SPOF)

### Risks and Mitigations
| Risk | Mitigation |
|---|---|
| LLM proxy outage kills all A2A | Circuit breaker + fallback model; scale llm-proxy to 2+ replicas |
| bot-config-api SPOF | Scale to 2+ replicas with shared Postgres task store |
| Metric loss on pod restart | Prometheus scrapes every 15s; in-process counters are ephemeral by design |
| Phase gate criteria too strict | Operator can adjust thresholds based on traffic volume |

## References

- M1.9 milestone: issues #137, #138, #139, #140, #141, #142, #143
- Grafana dashboard: "A2A Adoption (M1.9)" in `agentopia-base` monitoring ConfigMap
- PrometheusRule: `a2a-adoption-alerts` in `agentopia-base` monitoring ConfigMap
- Runbook: `docs/runbook-a2a.md` — M1.9 sections
