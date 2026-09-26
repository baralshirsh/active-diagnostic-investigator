# ADR-005: Keep lifecycle control deterministic; do not use an LLM supervisor

- **Status:** Accepted
- **Date:** 2026-09-26

## Context

Industrial diagnosis requires strict budgets, policy enforcement, retry rules, terminal-state guarantees and replayable transitions. These are control-flow responsibilities rather than diagnostic reasoning.

## Decision

Use a deterministic state-machine Coordinator. Specialized agents reason about hypotheses, evidence planning and verification. The Coordinator never decides the fault cause.

## Alternatives considered

- supervisor LLM routing all agents;
- LangGraph mega-graph as global orchestrator;
- free agent handoffs.

## Consequences

Benefits: bounded lifecycle behavior, easier testing and safety governance. Cost: explicit state machine and transition code.

## Revisit when

If future workflows require dynamic agent discovery; even then, safety/budget/policy gates should remain deterministic.
