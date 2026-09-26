# Contracts

## 1. Purpose

This document is the human-readable contract specification for the **Active Diagnostic Investigator (ADI)**.

It covers:

- domain objects shared across the system;
- agent input/output contracts;
- NATS event contracts;
- MCP tool contracts;
- platform/common-module interfaces;
- intra-agent module interfaces;
- error contracts;
- identifiers, tracing, idempotency and security metadata;
- schema compatibility and versioning rules.

Machine-transmitted/domain objects must validate against the corresponding JSON Schema under [`contracts/`](./contracts/).

## 2. Contract hierarchy and source of truth

```text
Business meaning / interface intent
        ↓
Contracts.md
        ↓
Machine wire/domain shape
        ↓
contracts/**/*.json
        ↓
Pydantic models generated/maintained from same definitions
        ↓
Runtime validation + contract tests
```

Rules:

1. Every network/event/tool payload has `schema_version` or is wrapped in a versioned envelope.
2. A change to a machine schema and the matching section in this document must land in the same change set.
3. Python implementation classes may add internal behavior but must not silently change the wire contract.
4. Infrastructure adapters implement typed ports; core domain/agent logic does not import vendor SDK types.

## 3. Cross-cutting conventions

### 3.1 Versions

V1 schemas use:

```json
{"schema_version": "1.0"}
```

The file naming convention is:

```text
<contract-name>.v1.json
```

### 3.2 Identifiers

Runtime identifiers use UUID strings unless the type is explicitly a stable business code.

Important identifiers:

```text
investigation_id
agent_execution_id
diagnostic_action_id
evidence_id
hypothesis_id
tool_execution_id
event_id
correlation_id
causation_id
policy_decision_id
opaque_run_handle
```

`opaque_run_handle` must never encode the benchmark fault label.

### 3.3 Time

All timestamps are RFC 3339 / ISO-8601 strings with timezone information and are persisted as `TIMESTAMPTZ`.

### 3.4 Unknown enum behavior

Do not add new enum values in a minor-compatible schema revision unless existing consumers explicitly implement an `UNKNOWN`/fallback path. Otherwise the change is breaking.

### 3.5 Trace context

Asynchronous/event contracts carry:

```json
{
  "trace": {
    "traceparent": "00-...",
    "tracestate": null
  }
}
```

MCP calls propagate W3C trace context through `_meta` using the standard `traceparent`, `tracestate`, and optionally `baggage` keys.

### 3.6 Idempotency

Every message has a globally unique `event_id`.

Each consumer persists:

```text
(consumer_name, event_id)
```

inside the same transaction as its business state mutation.

Tool execution is correlated to a unique `diagnostic_action_id`; read-only retries reuse that action ID.

### 3.7 Security/data minimization

No contract exposed to diagnostic agents may contain:

```text
fault_id
idv_number
ground_truth
private scenario label
source filename/path that reveals ground truth
secret/token/password
```

Large/raw telemetry vectors should be referenced or stored in a bounded tool-result store rather than embedded into normal events.

---

# 4. Domain contracts

Machine schemas: [`contracts/domain/`](./contracts/domain/).

## 4.1 Investigation

Represents the aggregate root of a diagnostic investigation.

Key fields:

```text
id
scenario_public_id
asset_id
subsystem
state
query_budget
query_cost_used
tool_call_count
max_tool_calls
version
created_at
updated_at
```

Legal terminal states:

```text
DIAGNOSED
INCONCLUSIVE
SAFE_ABORT
```

The investigation version is incremented for optimistic concurrency control.

## 4.2 Hypothesis

Represents one competing diagnostic explanation.

```text
id
investigation_id
code
title
description
status
prior_weight
current_weight
supporting_evidence_ids[]
contradicting_evidence_ids[]
unresolved_questions[]
version
```

Statuses:

```text
ACTIVE
WEAKENED
REJECTED
LEADING
CONFIRMED
UNKNOWN
```

`current_weight` is a reasoning weight. It must never be displayed as a calibrated probability unless a later calibration subsystem explicitly provides that guarantee.

## 4.3 Evidence

Evidence is the only object that may support a final diagnostic claim.

```text
id
investigation_id
diagnostic_action_id
tool_execution_id
source_type
observation
signal_refs[]
time_window
value_summary
supports_hypothesis_ids[]
contradicts_hypothesis_ids[]
reliability
diagnostic_relevance
query_cost
created_at
```

Source types include:

```text
telemetry
baseline_comparison
derived_metric
process_knowledge
historical_pattern
```

No Evidence object may be created from an MCP failure or an unvalidated tool response.

## 4.4 DiagnosticAction

A proposed or approved bounded information-acquisition action.

```text
id
investigation_id
status
tool_name
arguments
reason
target_hypothesis_ids[]
expected_discrimination
estimated_cost
actual_cost
policy_decision_id
```

Statuses:

```text
PROPOSED
APPROVED
DENIED
EXECUTING
COMPLETED
FAILED
```

The Planner may produce only `PROPOSED`. The Coordinator/policy layer owns approval. The Evidence Agent owns execution/completion.

## 4.5 VerificationResult

The independent challenge output produced by the Verification Agent.

```text
decision
leading_hypothesis_id
grounding_ok
contradictions_resolved
evidence_independence_ok
budget_ok
reasons[]
remaining_questions[]
recommendation
```

Decisions:

```text
VERIFIED
NEEDS_MORE_EVIDENCE
REJECT_CONCLUSION
INCONCLUSIVE
```

A `VERIFIED` output is necessary but not sufficient for a diagnosis; deterministic Coordinator stop rules still apply.

## 4.6 DiagnosticConclusion

Terminal decision record.

```text
status
leading_hypothesis_id
hypothesis_strength
supporting_evidence_ids[]
contradicting_evidence_ids[]
alternative_hypothesis_ids[]
remaining_uncertainties[]
recommended_next_check
query_cost_used
tool_call_count
human_confirmation_required
```

`human_confirmation_required` is always `true` in V1.

---

# 5. Agent contracts

Machine schemas: [`contracts/agents/`](./contracts/agents/).

Every reasoning agent receives a purpose-built projection of the blackboard rather than direct unrestricted database access.

## 5.1 Hypothesis Agent

### Input

```text
schema_version
investigation summary
initial observations
existing hypotheses
new evidence
bounded process context / fault catalogue version
```

### Output

```text
agent_execution_id
hypotheses[] / revisions[]
leading_hypothesis_code (optional)
sufficient_for_diagnosis (advisory only)
```

Contract rules:

- output may reference only evidence IDs present in the input context;
- agent may not request/call tools;
- initial call should ordinarily create multiple plausible hypotheses rather than a forced single diagnosis.

## 5.2 Diagnostic Planner Agent

### Input

```text
investigation budget summary
active hypotheses
persisted evidence summaries
available tool descriptors
policy constraints
recent diagnostic actions
```

### Output

Exactly one `DiagnosticActionProposal` in V1:

```text
tool_name
arguments
reason
target_hypothesis_ids[]
expected_discrimination
estimated_cost
```

Contract rules:

- estimated cost must fit remaining budget;
- tool must exist in the supplied catalogue;
- repeated identical low-value action requires explicit justification;
- Planner output is never implicitly approved.

## 5.3 Evidence Agent

### Input

```text
investigation summary
approved DiagnosticAction
policy_decision_id
maximum authorized cost
```

### Output

```text
agent_execution_id
tool_execution summary
evidence
```

Contract rules:

- action status must be `APPROVED`;
- output Evidence requires a successful validated tool result;
- actual cost must not exceed policy obligation;
- raw MCP response must be hash-linked to the tool execution.

## 5.4 Verification Agent

### Input

```text
investigation budget/use summary
current hypotheses
evidence summaries
hard-stop rule description
```

### Output

`VerificationResult`.

Contract rules:

- verifier cannot create new evidence;
- verifier cannot override deterministic stop rules;
- all grounding statements must reference supplied evidence/hypothesis state.

---

# 6. Event contracts

Machine schemas: [`contracts/events/`](./contracts/events/).

## 6.1 EventEnvelope

Every NATS domain event uses the same envelope:

```json
{
  "event_id": "uuid",
  "event_type": "evidence.acquired",
  "schema_version": "1.0",
  "occurred_at": "2026-09-26T16:00:00Z",
  "tenant_id": "local",
  "site_id": "tep-lab",
  "investigation_id": "uuid",
  "correlation_id": "uuid",
  "causation_id": "uuid-or-null",
  "producer": "evidence-agent",
  "trace": {
    "traceparent": "00-...-...-01",
    "tracestate": null
  },
  "payload": {}
}
```

For V1, `correlation_id` is normally the investigation ID.

## 6.2 Event types

```text
investigation.started
investigation.concluded
investigation.inconclusive
investigation.safe_abort

hypothesis.create_requested
hypothesis.created
hypothesis.update_requested
hypothesis.updated
hypothesis.failed

plan.requested
plan.proposed
plan.denied
plan.failed

evidence.requested
evidence.acquired
evidence.failed

verification.requested
verification.completed
verification.failed
```

Payloads should be small and reference blackboard IDs rather than reproduce large state.

## 6.3 DLQ event

A DLQ payload contains only safe diagnostic metadata:

```text
original_event_id
original_subject
consumer_name
attempts
error_type
error_summary
investigation_id
last_traceparent
```

Never include full prompts, raw telemetry, credentials, or hidden benchmark labels.

---

# 7. Tool contracts

Machine schemas: [`contracts/tools/`](./contracts/tools/).

The Evidence Agent uses an MCP Tool Gateway. V1 tools are read-only.

## 7.1 `tep.get_signal_snapshot`

Input:

```text
investigation_id
signal_ids[]
sample_index / timestamp
```

Output:

```text
values[]
units
missing flags
opaque source reference
```

## 7.2 `tep.get_signal_window`

Input:

```text
investigation_id
signal_id
relative or absolute time/sample window
```

Output:

```text
signal_id
timestamps/sample indices
values
units
sample_count
missing_count
opaque source reference
```

## 7.3 `tep.analyze_signal`

Input:

```text
investigation_id
signal_id
window
metrics[]
```

Allowed deterministic metrics:

```text
mean
std
variance
min
max
slope
rate_of_change
deviation_from_baseline
rolling_variance
persistence
correlation
```

Output:

```text
numeric metrics
bounded qualitative classifications
sample/missing counts
opaque source reference
```

The LLM does not calculate these metrics.

## 7.4 `tep.compare_with_baseline`

Input:

```text
investigation_id
signal_id
window
baseline_profile/version
```

Output:

```text
baseline summary
observed summary
normalized deviation
trend difference
```

## 7.5 `tep.get_process_relationships`

Input:

```text
component_or_signal
relationship_depth
```

Output:

```text
nodes[]
edges[]
relationship version
```

No hidden scenario/fault label may appear in any MCP output.

---

# 8. Platform/common-module contracts

Machine request/response schemas that cross process boundaries live in [`contracts/platform/`](./contracts/platform/). In-process interfaces are typed Python ports/protocols defined by the signatures below.

## 8.1 ModelGateway

All reasoning agents depend on this interface, never directly on Ollama/vLLM/cloud SDKs.

```python
class ModelGateway(Protocol):
    async def invoke_structured(
        self,
        *,
        agent_name: str,
        prompt_version: str,
        model_profile: str,
        input_payload: BaseModel,
        output_type: type[BaseModel],
        correlation: CorrelationContext,
    ) -> BaseModel: ...
```

Guarantees:

- validated structured output or a typed failure;
- bounded timeout/retry;
- one schema-repair attempt at most;
- token/model/prompt execution metadata persisted;
- trace context preserved.

## 8.2 PolicyClient

```python
class PolicyClient(Protocol):
    async def evaluate(
        self,
        request: PolicyEvaluationRequest,
        correlation: CorrelationContext,
    ) -> PolicyEvaluationResult: ...
```

Fail-closed rule: inability to establish an allow decision results in denial after bounded transport retry.

## 8.3 EventPublisher

Business logic writes outbox rows, not direct NATS messages. The outbox worker depends on:

```python
class EventPublisher(Protocol):
    async def publish(self, event: EventEnvelope) -> PublishReceipt: ...
```

`PublishReceipt` only confirms transport publication; it does not change domain state by itself.

## 8.4 BlackboardRepository ports

Use aggregate-specific repositories rather than leaking SQLAlchemy sessions into business logic.

Representative interfaces:

```python
class InvestigationRepository(Protocol):
    async def get(self, investigation_id: UUID) -> Investigation: ...
    async def create(self, investigation: Investigation) -> None: ...
    async def update_if_version(self, investigation: Investigation, expected_version: int) -> None: ...

class EvidenceRepository(Protocol):
    async def list_for_investigation(self, investigation_id: UUID) -> list[Evidence]: ...
    async def add(self, evidence: Evidence) -> None: ...
```

A `UnitOfWork` owns the database transaction.

## 8.5 UnitOfWork / transaction contract

```python
class UnitOfWork(Protocol):
    investigations: InvestigationRepository
    evidence: EvidenceRepository
    hypotheses: HypothesisRepository
    actions: DiagnosticActionRepository
    outbox: OutboxRepository
    processed_messages: ProcessedMessageRepository

    async def __aenter__(self): ...
    async def commit(self) -> None: ...
    async def rollback(self) -> None: ...
```

Business handlers must not ACK an inbound event until `commit()` succeeds.

## 8.6 CorrelationContext

```text
investigation_id
correlation_id
causation_id
traceparent
tracestate
actor/service
```

This object is passed between platform ports without exposing NATS or OTel implementation classes to core logic.

## 8.7 ToolClient

The Evidence Agent depends on a generic tool client abstraction:

```python
class ToolClient(Protocol):
    async def call(
        self,
        *,
        tool_name: str,
        arguments: dict,
        action_id: UUID,
        correlation: CorrelationContext,
    ) -> ToolCallResult: ...
```

The V1 implementation is MCP Streamable HTTP.

---

# 9. Intra-agent module contracts

Each agent is independently deployable, but its internal modules also use explicit typed contracts to avoid tightly coupling business reasoning, prompts, transport, and persistence.

## 9.1 Common worker pipeline

```text
EventConsumer
    ↓ EventEnvelope
InputProjectionLoader
    ↓ AgentContext
ReasoningStrategy
    ↓ AgentDecision
DecisionValidator
    ↓ ValidatedDecision
ApplicationHandler
    ↓ domain writes + outbox
UnitOfWork
```

### EventConsumer → handler

```python
async def handle(event: EventEnvelope, correlation: CorrelationContext) -> None
```

### InputProjectionLoader

```python
class InputProjectionLoader(Protocol, Generic[TContext]):
    async def load(self, investigation_id: UUID, trigger: EventEnvelope) -> TContext: ...
```

### ReasoningStrategy

```python
class ReasoningStrategy(Protocol, Generic[TContext, TDecision]):
    async def decide(self, context: TContext) -> TDecision: ...
```

This allows later replacement such as:

```text
LLMPlanningStrategy → PlanningDecision
InformationGainPlanningStrategy → PlanningDecision
HybridPlanningStrategy → PlanningDecision
```

without changing the Planner worker contract.

### DecisionValidator

```python
class DecisionValidator(Protocol, Generic[TDecision]):
    def validate(self, decision: TDecision, context: object) -> None: ...
```

It performs deterministic semantic checks beyond Pydantic shape validation, e.g. verifying referenced evidence IDs exist in the input projection.

## 9.2 Hypothesis Agent internals

```text
HypothesisContextLoader
HypothesisReasoningStrategy
HypothesisSemanticValidator
HypothesisPersistenceService
```

`HypothesisReasoningStrategy` never receives an MCP client.

## 9.3 Planner Agent internals

Core internal context:

```text
PlanningContext
  investigation budget
  hypotheses
  evidence
  available ToolDescriptor[]
  policy constraints
  recent actions
```

Core output:

```text
PlanningDecision
  DiagnosticActionProposal
  rationale metadata
```

A future non-LLM information-gain strategy must produce the same `PlanningDecision`.

## 9.4 Evidence Agent internals

```text
ApprovedActionLoader
PolicyGuard
ToolExecutor
ToolResultValidator
EvidenceBuilder
EvidencePersistenceService
```

Only `ToolExecutor` knows the `ToolClient` implementation.

`EvidenceBuilder` must receive a successful validated `ToolCallResult`; it cannot accept an exception/empty response as evidence.

## 9.5 Verification Agent internals

```text
VerificationContextLoader
VerificationReasoningStrategy
GroundingValidator
StopRecommendationMapper
```

The final Coordinator hard-stop evaluator is **not** part of the Verification Agent.

---

# 10. Error contracts

Machine schema: [`contracts/domain/error.v1.json`](./contracts/domain/error.v1.json).

All expected cross-boundary failures map to a bounded error envelope:

```text
code
category
message
retryable
correlation_id
investigation_id
component
safe_details
```

Categories:

```text
CONTRACT_VALIDATION
POLICY_DENIED
BUDGET_EXCEEDED
TOOL_UNAVAILABLE
TOOL_INVALID_RESULT
MODEL_UNAVAILABLE
MODEL_INVALID_OUTPUT
PERSISTENCE_CONFLICT
MESSAGING_FAILURE
SECURITY_VIOLATION
INTERNAL
```

Do not serialize raw stack traces, prompts, tokens, secrets, or raw telemetry into event/DLQ error payloads.

---

# 11. Observability contract

Use standard OpenTelemetry semantic conventions where available and bounded custom attributes for ADI domain state.

Required domain attributes:

```text
adi.investigation.id
adi.agent.name
adi.agent.execution_id
adi.event.type
adi.action.id
adi.tool.name
adi.query.cost
adi.query.cost_used
adi.verification.decision
adi.policy.allowed
adi.policy.decision_id
adi.scenario.public_id
```

Avoid high-cardinality/full-content attributes such as raw signal arrays, full prompts/responses, or secrets.

Persistent audit records, not traces, are the source of truth for why a decision occurred.

---

# 12. Compatibility/versioning policy

## Compatible change

Generally allowed within the same major schema only when all consumers tolerate it:

- add an optional field with a safe default/absence semantics;
- add metadata that consumers ignore safely;
- widen documentation without changing semantics.

## Breaking change

Requires a new major schema/subject/tool contract version:

- remove or rename a field;
- change field type;
- change the meaning of an existing field;
- make an optional field required;
- alter identifier semantics;
- add a strict enum value when consumers do not support unknown values;
- change tool behavior from read-only to state-changing.

Example:

```text
1.x → 2.0
```

## Event evolution

Prefer new event payload versions while keeping the stable domain event meaning. If event semantics themselves change, use a new subject version (`adi.v2...`) rather than overloading `adi.v1`.

## Tool evolution

A breaking tool change gets a new tool name/version or incompatible MCP schema version. A write-capable tool must never silently replace a V1 read-only tool.

---

# 13. Contract-test requirements

Every JSON Schema must have tests for:

```text
valid fixture accepted
required field missing → rejected
invalid UUID/type → rejected
unknown strict enum → rejected
serialization round-trip
schema_version validation
forbidden hidden-label field absent from agent-facing fixture
```

Cross-component contract tests must additionally prove:

1. an event payload produced by service A validates before service B consumes it;
2. duplicate event delivery does not repeat domain effects;
3. policy denial prevents tool execution;
4. tool failure cannot create Evidence;
5. evidence references valid source/tool execution IDs;
6. conclusion evidence IDs exist in the same investigation;
7. agent outputs cannot reference evidence they were not given;
8. ground-truth schemas/credentials are inaccessible to agent services;
9. trace context survives event and MCP boundaries.

---

# 14. Contract ownership matrix

| Contract family | Owner | Consumers |
|---|---|---|
| Domain objects | Core/domain | all services |
| Event envelope/subjects | Platform messaging | Coordinator + all workers |
| Hypothesis agent I/O | Hypothesis Agent | Coordinator/tests |
| Planner agent I/O | Planner Agent | Coordinator/tests |
| Evidence agent I/O | Evidence Agent | Coordinator/tests |
| Verification agent I/O | Verification Agent | Coordinator/tests |
| MCP tools | Tool Gateway | Evidence Agent |
| Policy request/result | Platform policy | Coordinator, Evidence Agent |
| Model Gateway | Platform model | reasoning agents |
| Audit/trace attributes | Platform observability | all services |

A contract change requires review by its owner and at least one downstream consumer owner.
