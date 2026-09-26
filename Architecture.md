# Architecture

## 1. Purpose

This document defines the V1 system architecture of the **Active Diagnostic Investigator (ADI)**. It explains component responsibilities, deployment boundaries, event flow, reliability behavior, policy enforcement, observability, and the exact multi-agent IDV14 investigation sequence.

Detailed object fields and JSON examples live in [`Contracts.md`](./Contracts.md) and the executable schemas under [`contracts/`](./contracts/).

## 2. Architectural style

ADI uses an **event-driven multi-agent blackboard architecture** with deterministic lifecycle control.

The architecture intentionally separates:

- **probabilistic reasoning** — performed by specialized agents;
- **deterministic control** — performed by the Coordinator, policy engine, budget evaluator, schema validators, and tool implementations;
- **authoritative state** — stored in PostgreSQL;
- **transport** — NATS JetStream;
- **tool access** — MCP Tool Gateway;
- **operational telemetry** — OpenTelemetry;
- **business/audit record** — persisted domain history.

There is no central LLM supervisor.

## 3. Core invariants

1. The Coordinator never diagnoses.
2. No agent depends on another agent's conversation history.
3. All inter-service messages are schema-versioned.
4. PostgreSQL is authoritative investigation state.
5. A state mutation and its outgoing event are committed atomically through an outbox row.
6. JetStream delivery is treated as at-least-once; duplicate delivery is expected and safe.
7. Each consumer commits domain state and its `processed_messages` marker before acknowledging the message.
8. Only the Evidence Agent can request industrial data/tools.
9. The Tool Gateway independently enforces read-only access.
10. Ground truth is invisible until a terminal investigation event reaches the Evaluation Service.
11. Policy denials are normal deterministic outcomes, not prompt instructions.
12. `INCONCLUSIVE` is a valid successful terminal state.
13. OpenTelemetry is not the authoritative audit database.
14. Full prompts, secrets, and raw telemetry arrays are not standard span attributes.

## 4. Logical component topology

```text
Experience Layer
┌──────────────────────┐
│ UI / FastAPI         │
└──────────┬───────────┘
           │
           ▼
Control Layer
┌──────────────────────┐
│ Investigation        │
│ Coordinator          │
│ deterministic FSM    │
└──────────┬───────────┘
           │
           ▼
Event Backbone
┌──────────────────────┐
│ NATS + JetStream     │
└─┬────────┬────────┬──┘
  │        │        │
  ▼        ▼        ▼
Hypothesis Planner  Evidence
Agent      Agent    Agent
  │        │        │
  └────────┼────────┘
           ▼
     Verification
        Agent
           │
           ▼
State Layer
┌──────────────────────┐
│ PostgreSQL           │
│ shared blackboard    │
│ outbox + audit       │
└──────────┬───────────┘
           │
     ┌─────┴──────────┐
     ▼                ▼
Tool/Data Layer    Evaluation
MCP Gateway        Service
     │             hidden truth
     ▼
TEP Adapter

Cross-cutting: Model Gateway | OPA | OpenTelemetry | Identity | Secrets
```

## 5. Agent responsibilities and isolation

### Hypothesis Agent

Owns the diagnostic explanation space. It creates competing hypotheses, links evidence to support/contradiction, revises weights/status, and records unresolved questions.

It has no MCP/tool identity.

### Diagnostic Planner Agent

Chooses exactly one bounded next diagnostic action per V1 planning cycle. It considers active hypotheses, unresolved questions, current evidence, available tool descriptors, remaining query budget, and prior actions.

It proposes an action but cannot authorize or execute it.

### Evidence Agent

Executes only Coordinator-approved diagnostic actions. It performs a defense-in-depth policy check, calls an MCP read-only tool, validates structured output, persists the tool execution, and converts deterministic output into a bounded `Evidence` object.

It may not invent evidence when a tool fails.

### Verification Agent

Challenges the leading hypothesis. It checks grounding, ignored contradictions, plausible alternatives, evidence independence, and whether another diagnostic observation is required.

It recommends `VERIFIED`, `NEEDS_MORE_EVIDENCE`, `REJECT_CONCLUSION`, or `INCONCLUSIVE`; deterministic hard-stop rules remain authoritative.

## 6. Deterministic Coordinator

The Coordinator owns investigation lifecycle, not diagnosis.

```text
CREATED
  ↓
HYPOTHESIS_PENDING
  ↓
PLANNING
  ↓
ACTION_POLICY_CHECK
  ├── denied → PLANNING
  ↓ allowed
EVIDENCE_PENDING
  ├── recoverable failure → PLANNING
  └── unrecoverable failure → SAFE_ABORT
  ↓
HYPOTHESIS_UPDATE_PENDING
  ↓
VERIFYING
  ├── NEEDS_MORE_EVIDENCE → PLANNING
  ├── REJECT_CONCLUSION   → PLANNING
  ├── INCONCLUSIVE        → INCONCLUSIVE
  └── VERIFIED
         ↓
    HARD_STOP_CHECK
       ├── false → PLANNING
       └── true  → DIAGNOSED
```

The Coordinator also owns:

- query-cost accounting;
- maximum tool-call enforcement;
- optimistic investigation versioning;
- policy-gate invocation;
- terminal-state creation;
- transition validity;
- recovery from denied/unavailable evidence paths.

## 7. Shared blackboard

PostgreSQL stores current snapshots and append-only history. Core tables are:

```text
investigations
initial_observations
hypotheses
hypothesis_revisions
diagnostic_actions
tool_executions
evidence
verification_results
conclusions
agent_executions
policy_decisions
outbox_events
processed_messages
```

The current state is query-efficient, while revision/execution tables preserve replayability.

## 8. Transactional outbox and idempotent consumers

Every event-producing state transition follows:

```text
BEGIN
  check processed_messages(event_id, consumer)
  validate current aggregate version
  perform local business decision
  persist state/history
  persist outgoing outbox event(s)
  persist processed_messages marker
COMMIT
ACK inbound NATS message
```

An Outbox Publisher repeatedly publishes unpublished rows to JetStream and marks them published only after broker acknowledgement.

If publication succeeds but the publisher crashes before updating `published_at`, the same event may be published again. This is expected. Every consumer uses `(consumer_name, event_id)` idempotency to avoid duplicated domain effects.

## 9. Event backbone

Subjects use:

```text
adi.v1.<domain>.<event>
```

Examples:

```text
adi.v1.investigation.started
adi.v1.hypothesis.create_requested
adi.v1.hypothesis.created
adi.v1.hypothesis.update_requested
adi.v1.hypothesis.updated
adi.v1.plan.requested
adi.v1.plan.proposed
adi.v1.plan.denied
adi.v1.evidence.requested
adi.v1.evidence.acquired
adi.v1.evidence.failed
adi.v1.verification.requested
adi.v1.verification.completed
adi.v1.investigation.concluded
adi.v1.investigation.inconclusive
adi.v1.investigation.safe_abort
```

Large telemetry arrays are not sent over NATS. Events carry references to blackboard/tool data.

### JetStream retry policy

V1 uses durable consumers and explicit acknowledgement.

Suggested delivery policy:

```text
MaxDeliver = 5
Backoff = 1s, 5s, 30s, 2m, 10m
```

After the retry budget is exhausted, a compact DLQ event is emitted and the Coordinator either chooses a recovery path or moves to `SAFE_ABORT` if progress is impossible.

## 10. Tool boundary: MCP

The Evidence Agent reaches industrial data only through the MCP Tool Gateway.

V1 follows the MCP `2026-07-28` stateless Streamable HTTP model. Each modern request carries the protocol version plus method/tool routing headers and trace context in `_meta`.

V1 tools:

```text
tep.get_signal_snapshot
tep.get_signal_window
tep.analyze_signal
tep.compare_with_baseline
tep.get_process_relationships
```

All are read-only and deterministic for a fixed dataset run.

The Tool Gateway removes/denies fields such as:

```text
fault_id
idv_number
ground_truth
private_scenario_name
source_filename_that_reveals_fault
```

## 11. Policy architecture

The Coordinator and Evidence Agent are Policy Enforcement Points. OPA is the Policy Decision Point.

V1 policy principles:

```text
Hypothesis Agent: no industrial tool access
Planner Agent:    no industrial tool access
Verifier Agent:   no industrial tool access
Evidence Agent:   only approved read-only diagnostic action
all services:     no hidden-evaluation namespace
all actions:      tenant/site scope respected
all tool calls:   remaining budget respected
unknown policy:   deny
OPA unavailable:  fail closed after bounded transport retry
```

A policy denial is persisted and returned to the Planner so it can propose a legal alternative. It is not retried as if it were a transient failure.

## 12. Model Gateway

No reasoning agent calls Ollama directly.

All agents depend on a common `ModelGateway` port that provides:

```text
structured output validation
model-profile selection
timeouts and bounded transport retry
one schema-repair attempt
prompt/model version metadata
token accounting
input/output hashing
OpenTelemetry GenAI instrumentation
```

This allows future replacement of Ollama by vLLM, a customer-managed endpoint, or an approved cloud provider without changing agent business logic.

## 13. Exact IDV14 interaction sequence

The first acceptance flow uses an opaque scenario handle. The Coordinator and agents never see the `IDV14` label.

```text
1. API → Coordinator
   POST /investigations

2. Coordinator transaction
   create Investigation + InitialObservation
   outbox: investigation.started

3. Coordinator consumes investigation.started
   state → HYPOTHESIS_PENDING
   outbox: hypothesis.create_requested

4. Hypothesis Agent
   reads current context
   creates competing hypotheses
   persists Hypotheses + AgentExecution
   outbox: hypothesis.created

5. Coordinator
   state → PLANNING
   outbox: plan.requested

6. Planner Agent
   reads hypotheses/evidence/budget/tool descriptors
   proposes a low-cost discriminating action, normally XMV10 analysis first
   persists DiagnosticAction(PROPOSED)
   outbox: plan.proposed

7. Coordinator → OPA
   authorization/budget check

8a. denied
   persist PolicyDecision + Action(DENIED)
   outbox: plan.denied then plan.requested

8b. allowed
   persist PolicyDecision + Action(APPROVED)
   state → EVIDENCE_PENDING
   outbox: evidence.requested

9. Evidence Agent
   idempotency check
   loads approved action
   repeats policy check
   calls MCP read-only tool
   validates deterministic result
   transactionally persists:
      ToolExecution
      Evidence
      actual query cost
      tool-call count
      Action(COMPLETED)
      outbox: evidence.acquired

10. Coordinator
    state → HYPOTHESIS_UPDATE_PENDING
    outbox: hypothesis.update_requested

11. Hypothesis Agent
    revises competing hypotheses using the new evidence
    persists HypothesisRevisions
    outbox: hypothesis.updated

12. Coordinator
    state → VERIFYING
    outbox: verification.requested

13. Verification Agent
    checks grounding, contradictions, alternatives and evidence independence
    persists VerificationResult
    outbox: verification.completed

14. Coordinator deterministic stop check

    if insufficient:
       state → PLANNING
       outbox: plan.requested
       repeat, commonly with XMEAS21 then a process-side discriminating check

    if verified + hard rules satisfied:
       persist DiagnosticConclusion(DIAGNOSED)
       outbox: investigation.concluded

    if no legal/affordable decisive evidence remains:
       persist DiagnosticConclusion(INCONCLUSIVE)
       outbox: investigation.inconclusive

15. Evaluation Service consumes terminal event
    only now resolves opaque_run_handle → hidden_ground_truth
    calculates experiment metrics
```

### Typical S1 evidence progression

The exact query is an agent decision, but the expected reasoning shape is:

```text
Initial reactor anomaly
  ↓
XMV10 cooling-water actuation behavior
  ↓
XMEAS21 cooling-loop thermal response
  ↓
process-side discriminating observation
  ↓
optional longer baseline comparison
  ↓
verified diagnosis or INCONCLUSIVE
```

The acceptance test must not hard-code the final answer into the planning prompt.

## 14. Hard stop rules

A known-fault diagnosis may be committed only when all V1 conditions hold:

```text
leading reasoning weight >= 0.75
margin over second hypothesis >= 0.25
>= 3 independent persisted evidence items
>= 1 mechanism/subsystem-specific evidence item
no unresolved HIGH-severity contradiction
Verification Agent grounding check passes
budget/tool-call constraints remain valid
```

Weights are reasoning weights, not calibrated probabilities.

If conditions cannot be achieved before the investigation budget is exhausted, the expected terminal result is `INCONCLUSIVE`.

## 15. Component retry matrix

| Operation | Retry behavior |
|---|---|
| NATS message | up to configured delivery limit; idempotent consumer |
| PostgreSQL transient transaction error | bounded transaction retry with jitter |
| LLM transport timeout/unavailable | two bounded transport attempts |
| Invalid LLM structured output | one repair attempt, then agent failure |
| OPA policy denial | no retry; deterministic denial |
| OPA transport failure | bounded retry; then fail closed |
| Read-only MCP network failure | bounded retry using same action correlation ID |
| MCP schema-invalid result | no semantic retry; tool failure |
| Hidden-label access attempt | deny immediately + security audit |
| Unsupported/write tool request | deny immediately + security audit |

## 16. OpenTelemetry design

Every investigation is searchable by a domain attribute such as:

```text
adi.investigation.id
```

W3C trace context is propagated in event envelopes and MCP `_meta`.

Representative spans:

```text
POST /investigations                      SERVER
adi.coordinator.transition                INTERNAL
publish adi.v1.plan.requested             PRODUCER
process adi.v1.plan.requested             CONSUMER
adi.agent.invoke                          INTERNAL
gen_ai.* model invocation                 CLIENT / GenAI
adi.policy.evaluate                       CLIENT
adi.mcp.call tep.analyze_signal           CLIENT
POST /mcp                                 CLIENT
mcp.tool tep.analyze_signal               SERVER
db transaction                            CLIENT
adi.evaluation.score                      INTERNAL
```

For asynchronous message handling, producer context is propagated with each message. Consumer spans may use the message creation context or an explicit link depending on the chosen instrumentation pattern, while `adi.investigation.id` preserves domain-level correlation.

Custom `adi.*` fields must remain bounded and avoid raw process arrays or sensitive data.

## 17. Audit vs observability

OpenTelemetry data may be sampled or expire. It is therefore not the authoritative decision record.

The persistent audit path is:

```text
agent_executions
policy_decisions
tool_executions
hypothesis_revisions
evidence
verification_results
conclusions
outbox/domain events
```

An investigation should remain replayable even when old telemetry traces are unavailable.

## 18. Security and data boundary

V1 uses public benchmark data and no personal data. Nevertheless, the architecture follows the same boundaries needed for later enterprise deployments:

- least-privilege service identities;
- read-only industrial connector credentials;
- hidden evaluation schema inaccessible to agents;
- no secrets in messages or traces;
- policy-driven tenant/site boundaries;
- local tool gateway capable of remaining inside a customer network;
- model selection abstracted behind a gateway for data-residency choices.

## 19. Local-to-cloud deployment path

Local V1:

```text
Docker Compose
single PostgreSQL
single NATS/JetStream
local Ollama
local MCP/TEP service
local OTel collector/backend
```

Future enterprise/cloud:

```text
Kubernetes/OpenShift
HA/managed PostgreSQL
clustered NATS / site leaf nodes
horizontally scaled independent agent workers
site-local MCP gateways
enterprise OIDC
customer/approved inference endpoint
central or federated observability
```

The contracts and service responsibilities remain unchanged.

## 20. External agent interoperability

NATS remains the internal event backbone. If external enterprise agents need to delegate work in later versions, an A2A-compatible façade can be added at the external boundary without changing the internal event/domain model.

## 21. Architecture decision records

Major decisions are captured under [`docs/adr/`](./docs/adr/). ADRs explain *why* a design was chosen; this document explains *how the chosen system works*.
