# Plan

## 14-day dependency-ordered implementation plan

Assumption: approximately **3 focused hours/day**, about **42 hours total**.

The order is deliberate: freeze contracts and anti-leakage boundaries first; build persistence/reliability/tool/observability foundations second; only then implement reasoning agents. UI is deliberately late.

Every day ends with a **Gate**. Do not intentionally move to the next day while the gate is failing.

## Day 1 — Repository, contract system, TEP anti-leakage boundary

### Goal

Create the skeleton and freeze the language/interfaces the rest of the system will depend on.

### Tasks

1. Initialize Python monorepo and package tooling.
2. Create the repository folders documented in `README.md`.
3. Configure Python 3.12+, package manager, Ruff, pytest, typing, and pre-commit.
4. Treat `Contracts.md` as the human-readable specification and `contracts/**/*.json` as the machine-readable wire/domain source of truth.
5. Implement Pydantic models corresponding to the V1 JSON Schemas:
   - Investigation;
   - Hypothesis;
   - Evidence;
   - DiagnosticAction;
   - VerificationResult;
   - DiagnosticConclusion;
   - ErrorEnvelope;
   - EventEnvelope;
   - four agent I/O pairs;
   - core tool request/results;
   - policy request/result;
   - model gateway request/result metadata.
6. Implement typed internal ports from `Contracts.md`:
   - `ModelGateway`;
   - `PolicyClient`;
   - `ToolClient`;
   - `EventPublisher`;
   - repository interfaces;
   - `UnitOfWork`;
   - common `InputProjectionLoader`, `ReasoningStrategy`, `DecisionValidator` protocols.
7. Add contract tests using valid/invalid fixtures.
8. Acquire/prepare the public TEP benchmark.
9. Implement offline preprocessing that:
   - maps source columns to canonical signal IDs;
   - creates opaque run handles;
   - removes fault labels, IDV labels, revealing filenames/paths from agent-facing artifacts;
   - stores hidden truth only in evaluation-only data.
10. Create opaque S1 fixture.
11. Add anti-leakage tests.
12. Mark initial ADRs `Accepted` or `Proposed` after review.

### Gate

- all V1 machine schemas validate through tests;
- Pydantic serialization round-trips against them;
- S1 can be loaded through the agent-facing adapter without exposing the hidden fault label;
- core domain code imports no NATS/MCP/Ollama/OPA SDK types.

---

## Day 2 — PostgreSQL blackboard + transactional outbox

### Goal

Build durable authoritative state before event transport or agents.

### Tasks

1. Add PostgreSQL to Docker Compose.
2. Add SQLAlchemy/async driver and Alembic.
3. Implement blackboard tables:
   - investigations;
   - initial observations;
   - hypotheses;
   - hypothesis revisions;
   - diagnostic actions;
   - evidence;
   - agent executions;
   - tool executions;
   - verification results;
   - conclusions;
   - policy decisions;
   - outbox events;
   - processed messages.
4. Implement repository adapters for the Day-1 interfaces.
5. Implement optimistic `investigations.version` updates.
6. Implement `UnitOfWork` and atomic state + outbox + processed-message commit.

### Gate

An integration test persists a state mutation plus outbox row atomically, rolls back both on injected failure, and rejects optimistic-version conflicts.

---

## Day 3 — NATS JetStream + idempotent worker foundation

### Goal

Create reliable asynchronous communication before reasoning agents exist.

### Tasks

1. Add NATS/JetStream.
2. Configure V1 streams, subjects and durable consumers.
3. Implement Outbox Publisher using the `EventPublisher` port.
4. Implement common consumer worker:
   - schema validation;
   - trace extraction;
   - idempotency check;
   - transaction;
   - ACK only after commit.
5. Implement bounded redelivery/backoff and DLQ.
6. Implement a dummy producer/consumer transition.

### Gate

Forced duplicate delivery and worker restart produce one domain mutation, while outbox transport can safely republish the same event.

---

## Day 4 — MCP TEP Tool Gateway + OPA policy layer

### Goal

Build the deterministic evidence boundary before the Evidence Agent exists.

### Tasks

1. Implement MCP `2026-07-28` Streamable HTTP server.
2. Implement read-only tools:
   - `tep.get_signal_snapshot`;
   - `tep.get_signal_window`;
   - `tep.analyze_signal`;
   - `tep.compare_with_baseline`;
   - `tep.get_process_relationships`.
3. Validate request/result against `contracts/tools` schemas.
4. Add TEP adapter behind tools.
5. Add OPA.
6. Implement policy bundle:
   - evidence identity only;
   - approved action only;
   - read-only tools only;
   - no hidden evaluation namespace;
   - site/tenant/budget restrictions;
   - unknown → deny.
7. Implement `PolicyClient` and MCP `ToolClient` adapters.

### Gate

Evidence identity can query XMV10 through MCP; Planner/Hypothesis/Verifier identities cannot. Hidden-label and write-like access is impossible through the registered V1 tool surface.

---

## Day 5 — OpenTelemetry and audit foundation

### Goal

Make traceability a runtime property before adding LLM decisions.

### Tasks

1. Add OpenTelemetry SDK + Collector.
2. Configure local compatible trace/metric/log backend.
3. Instrument FastAPI, PostgreSQL, NATS, OPA HTTP and MCP HTTP/tool execution.
4. Propagate W3C trace context through EventEnvelope and MCP `_meta`.
5. Implement bounded `adi.*` domain attributes.
6. Confirm audit state remains in PostgreSQL, not only telemetry.
7. Add redaction tests.

### Gate

One dummy tool flow is traceable API → NATS → worker → OPA → MCP → DB using the investigation ID, and raw signal vectors/secrets are absent from exported attributes.

---

## Day 6 — Model Gateway + common agent worker harness

### Goal

Build the structured reasoning boundary once before implementing four agents.

### Tasks

1. Add/connect Ollama.
2. Implement `ModelGateway` adapter behind Day-1 interface.
3. Add agent model profiles.
4. Implement:
   - structured Pydantic output;
   - timeout/retry;
   - one schema-repair attempt;
   - prompt/model versions;
   - token accounting;
   - input/output hashes;
   - OTel model span.
5. Implement common agent-worker shell using:
   - `InputProjectionLoader`;
   - `ReasoningStrategy`;
   - `DecisionValidator`;
   - `UnitOfWork`.
6. Add deterministic fake-model strategy for integration tests.

### Gate

A fake strategy and real local-model smoke test both produce a contract-valid structured decision through the same Model Gateway/worker interfaces.

---

## Day 7 — Hypothesis Agent

### Goal

Implement the first independent reasoning service.

### Tasks

1. Implement Hypothesis projection loader.
2. Implement prompt/reasoning strategy v1.
3. Implement deterministic semantic validation:
   - only supplied evidence IDs referenced;
   - valid fault catalogue codes;
   - no forced diagnosis on initial context.
4. Persist hypotheses/revisions + execution + outbox atomically.
5. Confirm container/service has no MCP credential.

### Gate

S1 initial context produces multiple plausible cooling/process hypotheses and cannot access industrial tools or hidden labels.

---

## Day 8 — Diagnostic Planner Agent

### Goal

Implement the differentiating next-best-evidence behavior.

### Tasks

1. Implement PlanningContext projection.
2. Implement PlanningStrategy v1.
3. Provide only contract-defined ToolDescriptors, costs, policy constraints, budget and recent actions.
4. Produce one `DiagnosticActionProposal` per cycle.
5. Add semantic checks for legal tool, budget, repeated query and referenced hypotheses.
6. Coordinator handles policy evaluation of proposal and produces approved/denied path.

### Gate

For S1, Planner proposes a legal low-cost cooling-loop discriminating action (normally XMV10-related) without direct MCP access.

---

## Day 9 — Evidence Agent

### Goal

Execute approved actions and create grounded evidence only from deterministic results.

### Tasks

1. Implement ApprovedActionLoader.
2. Implement PolicyGuard.
3. Implement MCP ToolExecutor.
4. Validate MCP result schema.
5. Implement EvidenceBuilder.
6. Atomically persist ToolExecution + Evidence + cost/tool-count + action completion + outbox.
7. Implement `evidence.failed` recovery path.

### Gate

Planner → OPA → Evidence Agent → MCP → persisted Evidence works for the first S1 action, and an MCP failure creates no Evidence object.

---

## Day 10 — Verification Agent + deterministic stop evaluator

### Goal

Prevent confirmation-biased diagnosis.

### Tasks

1. Implement VerificationContext projection.
2. Implement verifier reasoning strategy.
3. Add deterministic GroundingValidator.
4. Persist VerificationResult.
5. Implement Coordinator stop evaluator independently:
   - leading-weight threshold;
   - margin;
   - minimum independent evidence;
   - mechanism-specific evidence;
   - contradiction rule;
   - budget/tool-count rule.
6. Implement terminal conclusion contracts.

### Gate

Premature conclusions are rejected, unresolved high-severity contradictions block diagnosis, and budget exhaustion can deterministically produce `INCONCLUSIVE`.

---

## Day 11 — Full S1 / IDV14 event-driven investigation

### Goal

Join the four independent agents into the complete investigation loop.

### Tasks

1. Implement full Coordinator transition table.
2. Wire all event subjects.
3. Run with fake strategies first.
4. Inject duplicate events and worker restart during the scenario.
5. Run with local LLM strategies.
6. Tune prompts only after inspecting persisted evidence/decisions.
7. Add Evaluation Service terminal hook.
8. Reveal hidden truth only on terminal event.

### Gate

`POST /investigations` completes S1 as `DIAGNOSED` or justified `INCONCLUSIVE`; all four agents participate and the entire path is reconstructable from blackboard/audit state.

---

## Day 12 — S2–S5 acceptance suite + baselines

### Goal

Turn the project from one demonstration into an experiment.

### Tasks

1. Add S2, S3, S4, S5 opaque fixtures.
2. Remove S5 fault from diagnostic knowledge catalogue.
3. Implement evaluation metrics:
   - correct outcome;
   - abstention;
   - query cost;
   - tool count;
   - evidence traceability;
   - hallucinated-evidence count;
   - budget compliance;
   - hypothesis revision behavior.
4. Implement fixed broad-bundle baseline.
5. Implement random-query baseline.
6. Add machine-readable experiment report.

### Gate

All five scenarios execute automatically and generate comparable evaluation results without exposing hidden truth during diagnosis.

---

## Day 13 — REST/UI + Docker usability hardening

### Goal

Make the system usable without direct database access.

### Tasks

1. Finish FastAPI investigation endpoints.
2. Add investigation timeline/hypothesis/evidence/action/verification/result projections.
3. Add minimal web UI or server-rendered dashboard if time permits.
4. Add health/readiness checks.
5. Add `.env.example`, Docker health dependencies and migration startup.
6. Verify clean-clone Docker Compose run.

### Gate

A new developer can `docker compose up --build`, start an investigation through an API/UI, and inspect its complete reasoning/evidence history.

---

## Day 14 — Release hardening, compliance/security notes, cloud portability

### Goal

Freeze a deployable `v0.1.0` rather than a development-only prototype.

### Tasks

1. Run all unit/contract/integration/acceptance tests.
2. Run S1–S5 experiment matrix.
3. Verify least privilege, OPA fail-closed, no hidden-label leakage, no secret/raw-array telemetry logging, and evidence-grounded conclusions.
4. Document data inventory and retention defaults.
5. Document intended use, limitations, read-only boundary and human confirmation requirement.
6. Add Kubernetes/Helm deployment skeleton only after Compose is stable.
7. Review/update ADR statuses and actual implementation deviations.
8. Tag `v0.1.0`.

### Gate / Definition of Done

```text
create investigation
→ four independent agents participate
→ evidence is acquired selectively
→ policy gates every tool call
→ duplicate delivery is safe
→ diagnosis or abstention is auditable
→ hidden truth appears only after terminal state
→ evaluation score is generated
→ distributed trace is inspectable
```

## Daily working discipline

For each day:

1. Work from that day's Gate backwards.
2. Create a small branch such as `day-03/nats-foundation`.
3. Implement only dependencies required by the current gate.
4. Add tests with every contract/behavior change.
5. Run lint, type checking, unit tests and the relevant integration test before closing the day.
6. Record deviations/decisions in `docs/devlog/day-XX.md` and create/update an ADR when the architecture changes materially.

## If the schedule slips

Protect first:

```text
contracts + anti-leakage
Postgres/outbox/idempotency
policy + MCP tools
observability/audit
four-agent loop
evaluation
```

Defer first:

```text
polished frontend
A2A implementation
production Kubernetes tuning
RAG/vector database
CMMS integration
historical incident search
formal entropy/information-gain optimizer
multiple model providers
automated control actions
```

## Post-V1 sequence

V1.1: formal information-gain scoring, missing/noisy/delayed sensors, better cost model.

V1.2: maintenance documentation/knowledge MCP, historical incidents, process ontology/knowledge graph.

V2: customer adapter SDK, OPC-UA/MQTT/historian connectors, A2A external agent façade, enterprise OIDC, site-local gateways, Kubernetes production profile and stronger multi-tenant policy bundles.
