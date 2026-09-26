# Machine-readable contracts

These JSON Schemas are the executable wire/domain contract definitions for ADI V1.

```text
contracts/
├── domain/     core shared objects
├── agents/     four agent input/output payloads
├── events/     NATS event/DLQ envelopes
├── tools/      MCP tool request/result contracts
└── platform/   policy/model/correlation contracts
```

All schemas use JSON Schema Draft 2020-12.

## Rule

`Contracts.md` explains semantics and internal typed interfaces. These JSON files define the machine-validated shapes that cross process/module boundaries or are persisted as canonical structured objects.

During Day 1, Pydantic v2 models should be created from the same definitions and contract tests must verify representative fixtures against both Pydantic and JSON Schema.

Breaking changes require a new major contract version; do not silently modify V1 semantics.
