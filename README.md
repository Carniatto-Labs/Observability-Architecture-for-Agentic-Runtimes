# Observability Architecture for Agentic Runtimes

> An OpenTelemetry-based architecture for observing durable agentic execution: Runs, state transitions, checkpoints, tool causality, external effects, compensation, and privacy-aware telemetry.

**Status:** Architectural proposal  
**Maintained by:** Carniatto Labs

## Why this exists

Most agent observability focuses on model calls: latency, token usage, cost, and tool invocations.

That is necessary, but not sufficient for production agentic systems.

Real agents execute durable workflows. They pause, resume from checkpoints, delegate work, call tools, change external systems, retry operations, and may need to compensate for partial failure.

This project proposes an observability architecture that treats the **Run** — the concrete execution of an agentic runtime — as the primary unit of observation.

## The Core Idea

```text
Thread or Session
  -> Run
     -> Workflow
        -> Decision
        -> State Transition
        -> Tool Call
        -> External Effect
        -> Checkpoint / Resume
        -> Compensation
```

The goal is to make runtime behavior understandable end to end:

- What initiated this execution?
- What state was it in?
- Which decision triggered an action?
- What changed outside the system?
- Was the action retried or compensated?
- Was the Run resumed from a checkpoint?
- What did it cost, and why did it fail?

## What This Specification Covers

- Run identity, lifecycle, and distributed correlation
- Durable state transitions and checkpoint recovery
- Agentic decision loops and operational rationale
- Tool execution and MCP integration
- External effects, idempotency, retries, and compensations
- W3C Trace Context propagation and non-linear causal links
- Metrics for latency, token usage, cost, retries, and recovery
- Privacy-aware logging, redaction, and guardrails
- OpenTelemetry Collector as the telemetry policy enforcement point

## Design Position

This is not a competing observability standard.

It builds on OpenTelemetry and reuses existing GenAI conventions wherever they apply. It proposes runtime-specific extensions for concepts that are not yet fully standardized, including:

- `agent.runtime.*`
- `agent.state.*`
- `agent.decision.*`
- `agent.effect.*`
- `agent.tool.*`

These extensions are intended to be validated in real implementations and discussed with the OpenTelemetry GenAI community.

## Read the Specification

The complete proposal is available in:

- [RFC: Observability Architecture for Agentic Runtimes](./RFC.md)

## Standards and References

This proposal builds on:

- [OpenTelemetry GenAI Semantic Conventions](https://github.com/open-telemetry/semantic-conventions-genai)
- [OpenTelemetry Trace API](https://opentelemetry.io/docs/specs/otel/trace/api/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [OpenTelemetry guidance for sensitive data](https://opentelemetry.io/docs/security/handling-sensitive-data/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [LangGraph persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Saga Patterns](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-patterns.html)

## Contributing

Feedback is welcome from teams building:

- durable agent workflows;
- multi-agent systems;
- MCP servers and clients;
- agent platforms;
- OpenTelemetry instrumentation;
- runtime orchestration systems.

Please open an issue to discuss use cases, terminology, implementation experience, or compatibility concerns before submitting major changes.

## Status and Scope

This repository contains an
