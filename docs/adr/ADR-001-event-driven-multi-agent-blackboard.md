# ADR-001: Use an event-driven multi-agent blackboard architecture

- **Status:** Accepted
- **Date:** 2026-09-26

## Context

ADI must be multi-agent from V1, auditable, locally deployable, cloud-ready, and adaptable to customer environments. A central LLM supervisor would concentrate routing, reasoning and failure modes in one probabilistic component. Free-form agent handoffs would make replay and governance harder.

## Decision

Use four independently deployable reasoning agents coordinated through versioned events and a PostgreSQL shared blackboard. A deterministic Coordinator owns lifecycle/budgets/policy transitions, not diagnosis.

## Alternatives considered

- central supervisor LLM with sub-agents;
- direct agent-to-agent RPC/chat handoffs;
- one monolithic agent graph;
- free-form swarm.

## Consequences

Benefits: clear responsibility boundaries, replayability, independent scaling, easier customer integration and future agent replacement. Cost: more contracts, event/state discipline and integration tests.

## Revisit when

If V1 evidence shows the event/blackboard overhead is disproportionate or a safety-certified workflow engine is required.
