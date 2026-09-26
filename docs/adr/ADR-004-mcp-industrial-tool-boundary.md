# ADR-004: Use MCP as the industrial tool boundary

- **Status:** Accepted
- **Date:** 2026-09-26

## Context

The Evidence Agent must access TEP now and later OPC-UA, historians, CMMS or customer APIs without embedding vendor-specific code in the agent. Tool access also requires independent authorization and observability.

## Decision

Expose industrial evidence operations through a read-only MCP Tool Gateway. V1 uses the 2026-07-28 stateless Streamable HTTP protocol profile. Only the Evidence Agent receives a tool identity.

## Alternatives considered

- direct Python function calls;
- REST endpoint per industrial source;
- vendor SDK calls from the Evidence Agent.

## Consequences

Benefits: replaceable adapters, standardized tool schemas, easier site-local gateway deployment. Cost: additional gateway and protocol validation layer.

## Revisit when

If a target company's infrastructure forbids MCP or mandates another tool-integration standard.
