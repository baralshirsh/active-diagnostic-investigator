# ADR-002: Use NATS JetStream as the internal event backbone

- **Status:** Accepted
- **Date:** 2026-09-26

## Context

Agents need asynchronous decoupled communication locally and later in clustered deployments. V1 also needs durable consumption, redelivery and explicit acknowledgement without introducing heavyweight infrastructure.

## Decision

Use NATS + JetStream with durable consumers, explicit acknowledgement, bounded redelivery/backoff and idempotent application consumers. Treat delivery as at-least-once.

## Alternatives considered

- Kafka;
- RabbitMQ;
- direct HTTP/gRPC;
- database polling only.

## Consequences

Benefits: small local footprint, strong pub/sub model, persistence, cloud/on-prem path. Cost: team must correctly implement idempotency, subject/version conventions and outbox publishing.

## Revisit when

If enterprise target environments standardize on a mandated broker and the adapter cost is lower than operating NATS.
