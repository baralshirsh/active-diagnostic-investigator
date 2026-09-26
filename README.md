# Active Diagnostic Investigator (ADI)

A local-first, cloud-ready **multi-agent system for active industrial fault diagnosis under partial observability**.

ADI does not assume that all relevant telemetry is already available. It maintains competing hypotheses, chooses the **next best diagnostic evidence**, acquires only approved read-only evidence, revises its hypotheses, independently challenges the leading explanation, and either produces an auditable human-reviewed diagnosis or explicitly returns `INCONCLUSIVE`.

## V1 research question

> Can a multi-agent diagnostic system reach the correct industrial fault diagnosis while selectively acquiring less evidence than a fixed broad-data strategy, while keeping every decision, tool call, policy decision, and conclusion traceable?

V1 uses the **Tennessee Eastman Process (TEP)** as a controlled benchmark. The first end-to-end acceptance scenario is a hidden reactor cooling-loop fault; the benchmark fault label is isolated from all diagnostic agents and is available only to the evaluation service after the investigation terminates.

## Product position

Typical condition-monitoring, predictive-maintenance, and RCA solutions focus on detecting anomalies, estimating failures, ranking root causes, and recommending standard actions.

ADI focuses on the diagnostic gap between detection and repair:

> **What evidence should be collected next to reduce diagnostic uncertainty most efficiently?**

```text
Anomaly
  ↓
Competing hypotheses
  ↓
Next-best evidence proposal
  ↓
Policy + budget check
  ↓
Evidence acquisition
  ↓
Hypothesis revision
  ↓
Independent verification
  ↓
More evidence? ── yes ──┐
  │                      │
  no                     │
  ↓                      │
Diagnosis / INCONCLUSIVE ←┘
  ↓
Human confirmation
```

## V1 agents

ADI is multi-agent from inception. Each reasoning agent is independently deployable and has a narrow permission boundary.

| Agent | Responsibility | Direct industrial tool access |
|---|---|---|
| **Hypothesis Agent** | Create and revise competing fault hypotheses | No |
| **Diagnostic Planner Agent** | Choose the next diagnostic action that best discriminates hypotheses within budget | No |
| **Evidence Agent** | Execute approved read-only tools and create grounded evidence | Yes, through the MCP Tool Gateway |
| **Verification Agent** | Challenge grounding, contradictions, alternatives, and stopping readiness | No |

A deterministic **Investigation Coordinator** manages lifecycle, policy gates, budgets, retries, and state transitions. It is intentionally **not** an LLM agent and never performs fault diagnosis.

## Architecture principles

1. Agents communicate through **versioned structured contracts**, not chat history.
2. PostgreSQL is the authoritative **shared investigation blackboard**.
3. Agent state changes and outgoing events are committed atomically through a **transactional outbox**.
4. NATS JetStream is the asynchronous event backbone; all consumers are **idempotent**.
5. MCP is the industrial **tool boundary**.
6. Policy enforcement is external to prompts and defaults to **fail closed**.
7. Only the Evidence Agent has an industrial-tool identity in V1.
8. All V1 industrial tools are **read-only**.
9. OpenTelemetry provides operational observability; domain audit state remains separately persisted.
10. Hidden benchmark truth is isolated from all diagnostic services.
11. Domain and agent logic depend on **contracts/ports**, not vendor SDKs.
12. The same containerized services should run locally now and on Kubernetes/cloud infrastructure later.

## High-level topology

```text
                    ┌─────────────────────┐
                    │     UI / REST API   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Investigation       │
                    │ Coordinator         │
                    │ deterministic       │
                    └──────────┬──────────┘
                               │
                           NATS JetStream
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
  Hypothesis Agent      Planner Agent       Evidence Agent
          │                    │                    │
          └────────────────────┼────────────────────┘
                               ▼
                     Verification Agent
                               │
                               ▼
                         PostgreSQL
                     Shared Blackboard
                               │
                  ┌────────────┴────────────┐
                  ▼                         ▼
          MCP Tool Gateway            Evaluation Service
                  │                  (hidden truth only)
                  ▼
              TEP Adapter

Cross-cutting:
Model Gateway | OPA | OpenTelemetry | Audit | OIDC-ready identity
```

## Documentation map

The project documentation is intentionally separated by responsibility:

| File / folder | Responsibility |
|---|---|
| [`README.md`](./README.md) | Project/product overview, scope, quick orientation |
| [`Architecture.md`](./Architecture.md) | System topology, runtime flows, reliability, security and deployment architecture |
| [`Contracts.md`](./Contracts.md) | Human-readable communication and module-interface specification |
| [`Plan.md`](./Plan.md) | Dependency-ordered 14-day implementation plan |
| [`contracts/`](./contracts/) | Machine-readable versioned JSON Schemas for wire/domain contracts |
| [`docs/adr/`](./docs/adr/) | Architecture Decision Records explaining why major choices were made |

**Source-of-truth rule:** machine-transmitted objects must validate against a schema under `contracts/`. `Contracts.md` explains those schemas and also defines typed internal module interfaces that are not wire messages.

## V1 acceptance scenarios

| Scenario | Hidden truth | Primary capability tested |
|---|---|---|
| S1 | Reactor cooling-water valve sticking | Core active diagnosis |
| S2 | Reactor cooling-water inlet-temperature step | Reject valve-fault confirmation bias |
| S3 | Random reactor cooling-water inlet-temperature variation | Temporal discrimination |
| S4 | Condenser cooling-water valve sticking | Generalization to another subsystem |
| S5 | Reaction-kinetics drift omitted from the V1 knowledge catalogue | Open-set abstention / `INCONCLUSIVE` |

## V1 success criteria

Mandatory:

- no hidden-label leakage;
- zero fabricated evidence in acceptance runs;
- every final evidence claim maps to persisted evidence and a deterministic source/tool result;
- every tool call and policy decision is auditable;
- query budget and maximum tool-call count are always enforced;
- all diagnoses require human confirmation;
- no write/control industrial tool is exposed;
- duplicate event delivery never creates duplicate domain state.

Experimental targets:

- at least 80% correct outcomes for supported S1–S4 runs;
- at least 80% correct `INCONCLUSIVE` outcomes for S5;
- median diagnostic query cost no more than 60% of the fixed broad-data baseline.

## Proposed V1 stack

- Python 3.12+
- FastAPI
- Pydantic v2
- LangGraph **inside** individual reasoning agents only
- PostgreSQL + Alembic
- NATS + JetStream
- MCP 2026-07-28 Streamable HTTP tool gateway
- Open Policy Agent (OPA)
- Ollama through an internal Model Gateway
- OpenTelemetry SDK + Collector
- Grafana/Tempo/Prometheus/Loki or another compatible local backend
- Docker Compose
- pytest

The domain contracts must not depend on LangGraph, Ollama, NATS, MCP SDK classes, OPA SDK classes, or a particular observability backend.

## Repository target

```text
active-diagnostic-investigator/
├── apps/
│   ├── api/
│   └── coordinator/
├── agents/
│   ├── hypothesis/
│   ├── planner/
│   ├── evidence/
│   └── verifier/
├── domain/
│   ├── investigations/
│   ├── hypotheses/
│   ├── evidence/
│   ├── actions/
│   └── events/
├── platform/
│   ├── messaging/
│   ├── model_gateway/
│   ├── policy/
│   ├── observability/
│   └── persistence/
├── tools/
│   └── mcp_tep/
├── adapters/
│   └── tep/
├── evaluation/
│   ├── scenarios/
│   ├── baselines/
│   └── scoring/
├── contracts/
│   ├── domain/
│   ├── agents/
│   ├── events/
│   ├── tools/
│   └── platform/
├── docs/
│   ├── adr/
│   └── devlog/
├── deploy/
│   ├── docker/
│   └── kubernetes/
├── tests/
│   ├── unit/
│   ├── contract/
│   ├── integration/
│   └── acceptance/
├── README.md
├── Architecture.md
├── Contracts.md
└── Plan.md
```

## Local V1 target

```bash
docker compose up --build
```

A user should then be able to create an investigation, watch all four agents participate, inspect the evidence/hypothesis history, inspect policy/tool activity, see the terminal diagnosis or abstention, and reveal the benchmark ground truth only after the investigation reaches a terminal state.

## Safety boundary

V1 is strictly read-only.

Allowed: read benchmark telemetry, compute deterministic statistics, read process relationships, build evidence, form hypotheses, recommend diagnostic checks.

Not allowed: change controller values, reset alarms, actuate valves, stop/start machinery, create physical work orders automatically, or read hidden evaluation labels.

The final industrial decision remains with a human.
