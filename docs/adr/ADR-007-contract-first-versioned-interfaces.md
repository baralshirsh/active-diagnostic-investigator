# ADR-007: Use contract-first versioned interfaces

- **Status:** Accepted
- **Date:** 2026-09-26

## Context

ADI is designed for independent agents, adapter replacement and customer-specific deployment. Implicit Python dictionaries and vendor SDK types would couple components and make evolution unsafe.

## Decision

Define contract semantics in `Contracts.md`, machine wire/domain shapes in versioned JSON Schemas under `contracts/`, and typed Python ports for internal/common-module interfaces.

## Alternatives considered

- code-first unversioned Pydantic models only;
- ad-hoc dictionaries;
- protobuf for all internal and external contracts from V1.

## Consequences

Benefits: contract tests, independent component evolution and future language interoperability. Cost: documentation/schema maintenance discipline.

## Revisit when

If performance or polyglot needs justify introducing protobuf/gRPC for selected high-throughput paths.
