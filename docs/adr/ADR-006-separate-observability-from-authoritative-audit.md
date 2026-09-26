# ADR-006: Separate operational observability from authoritative audit state

- **Status:** Accepted
- **Date:** 2026-09-26

## Context

Traces/logs may be sampled, redacted or retained for shorter periods, but diagnostic decisions must remain replayable and auditable.

## Decision

Use OpenTelemetry for operational traces/metrics/log correlation, while PostgreSQL domain records remain the authoritative record of agent executions, evidence, hypotheses, policy decisions, tool calls and conclusions.

## Alternatives considered

- use trace backend as decision audit database;
- log-only audit.

## Consequences

Benefits: stable business audit independent of telemetry retention. Cost: some metadata is represented in both telemetry and domain audit records.

## Revisit when

If a regulated deployment mandates a dedicated immutable/WORM audit service; the same audit contract can then be dual-written/exported.
