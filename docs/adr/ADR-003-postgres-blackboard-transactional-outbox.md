# ADR-003: Use PostgreSQL as blackboard and transactional outbox store

- **Status:** Accepted
- **Date:** 2026-09-26

## Context

ADI needs authoritative current investigation state, append-only decision history, relational integrity and safe publication of events after state mutations.

## Decision

Use PostgreSQL for the shared blackboard, audit records, processed-message markers and transactional outbox. Domain mutation + outbox + idempotency marker are one transaction.

## Alternatives considered

- event store as the only source of truth;
- separate document database;
- direct DB write followed by broker publish;
- broker-first state reconstruction.

## Consequences

Benefits: strong consistency and familiar tooling. Cost: outbox publisher and cleanup/retention must be implemented.

## Revisit when

If event-sourcing requirements become dominant or investigation scale requires specialized time-series/event storage.
