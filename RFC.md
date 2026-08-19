# RFC: Observability Architecture for Agentic Runtimes

**Status:** Architectural Proposal  
**Version:** 0.1.0  
**Last updated:** 2026-08-19  
**Scope:** Agent execution runtimes, their durable workflows, tools, external effects, and distributed integrations.

## 1. Purpose

This specification defines an observability architecture for agentic systems centered on the **execution runtime**.

The agent is not treated as the primary observable unit. The root entity is the temporal execution instance, called a **Run**. A Run represents the concrete fulfillment of an intent: it may last milliseconds or days, pause, resume, delegate work, alter external systems, and execute compensations.

The architecture uses OpenTelemetry as its interoperability foundation and adds a semantic layer for aspects that are not yet fully standardized: durable state, checkpoints, resumption, decision causality, external effects, and compensation.

## 2. Principles

1. **The runtime is the center of observation.** The model, agent, tool, and workflow are execution participants.
2. **Causality is explicit.** Every material action must be attributable to the decision, state, and context that produced it.
3. **Durability is visible.** Pauses, checkpoints, and resumptions are first-class events.
4. **External effects require dedicated semantics.** Technical success of a call is insufficient to represent its real-world impact.
5. **Telemetry is secure by default.** Sensitive content is not collected by default.
6. **OTel conformance comes first.** Standardized attributes are preferred; runtime extensions use their own namespace.

## 3. Conceptual Model

```text
Thread or Session
  -> Run
     -> Workflow
        -> Decision
        -> State Transition
        -> Tool Call
        -> External Effect
        -> Checkpoint or Resume
        -> Compensation, when required
```

| Entity | Definition |
|---|---|
| `Run` | A complete instance of a runtime execution |
| `Thread` | A logical line of continuity across multiple Runs |
| `Workflow` | A graph, state machine, or process coordinated by the runtime |
| `Decision` | An operational choice that determines the next step |
| `Tool Call` | An invocation of a tool, MCP server, API, process, or function |
| `External Effect` | An observable change outside the runtime |
| `Checkpoint` | Persisted state that enables resumption |
| `Compensation` | An action intended to restore consistency after failure |

## 4. Run Identity and Correlation

Every Run MUST have a root span named `agent.run`.

| Attribute | Meaning |
|---|---|
| `agent.runtime.run.id` | Unique identifier of the Run |
| `agent.runtime.thread.id` | Continuous identity of the conversation or workflow |
| `agent.runtime.session.id` | Interaction session, when applicable |
| `agent.runtime.workflow.name` | Logical workflow name |
| `agent.runtime.workflow.version` | Executed workflow version |
| `agent.runtime.parent_run.id` | Run that originated a delegation |
| `agent.runtime.resume.reason` | Reason for resumption, when applicable |
| `enduser.id` | Pseudonymized user identifier, when authorized |

`run.id` identifies a specific execution. `thread.id` connects multiple executions that belong to the same operational history. `trace_id` connects distributed causality within a propagated context.

## 5. Run Lifecycle

```text
created -> running -> waiting -> checkpointed -> resumed -> completed
                     |             |              |
                     v             v              v
                   failed ------> compensating -> compensated
```

The runtime MUST emit lifecycle events for creation, start, wait, completion, failure, cancellation, resume, compensation start, and compensation completion.

Every transition MUST record its cause, timestamp, responsible actor, and state version.

## 6. State and Checkpoints

Workflow state is a critical operational resource. Every state transition MUST produce an `agent.state.transition` event.

| Attribute | Meaning |
|---|---|
| `agent.state.version` | Monotonically increasing state version |
| `agent.state.from` | Previous logical state |
| `agent.state.to` | Next logical state |
| `agent.state.changed_keys` | Changed keys |
| `agent.state.diff_hash` | Hash of the change, without exposing content |
| `agent.state.transition.reason` | Reason for the change |
| `agent.state.transition.actor` | User, runtime, model, tool, or operator |

Checkpoints MUST be represented by dedicated spans:

- `agent.checkpoint.save`
- `agent.checkpoint.restore`

Each checkpoint MUST include `checkpoint.id`, `thread.id`, state version, persistence backend, duration, and outcome.

Restoring a checkpoint MUST NOT continue the original span, because completed spans are immutable. It starts a new Run with the same `thread.id`, a reference to the restored `checkpoint.id`, and a link to the prior causal context when available.

## 7. Decisions, Inference, and the Agentic Loop

Model inference MUST use OpenTelemetry GenAI conventions, including model, provider, token usage, and duration.

The runtime MUST represent every decision cycle as `agent.decision`, including:

- `agent.decision.id`
- `agent.decision.kind`: `plan`, `select_tool`, `delegate`, `respond`, `compensate`, or `terminate`
- `agent.decision.basis`: sanitized operational summary
- `agent.decision.outcome`
- `agent.decision.iteration`

`agent.decision.basis` records a short, auditable justification. It MUST NOT contain private chain-of-thought, confidential content, or detailed model reasoning.

The observe-act-iterate loop MUST maintain an iteration identifier to group inference, tool execution, observation, and the next decision.

## 8. Tools and MCP Integrations

Tool calls MUST use `gen_ai.execute_tool` and compatible `gen_ai.tool.*` attributes whenever applicable.

| Attribute | Meaning |
|---|---|
| `agent.tool.call.id` | Unique call identifier |
| `agent.tool.effect.class` | `read_only`, `idempotent_write`, or `mutable_write` |
| `agent.tool.retry.count` | Attempt number |
| `agent.tool.idempotency_key` | Idempotency key, when available |
| `agent.tool.deduplication.status` | Deduplication result |
| `agent.tool.mcp.server` | MCP server identity, when applicable |

Context propagation between clients, MCP servers, subagents, and external services MUST use W3C Trace Context.

## 9. External Effects and Compensation

A call that modifies data, files, infrastructure, financial systems, or any external resource MUST emit an `agent.effect` span.

The effect MUST record a safe resource identifier, mutation type, idempotency key, confirmation status, originating decision, originating tool call, and compensation availability.

When a defined inverse action exists, the runtime MUST create an `agent.compensate` span. Compensation MUST be linked to the original effect through a span link and `agent.effect.id`.

Compensation outcomes are `succeeded`, `failed`, `partial`, or `not_applicable`.

Compensation is not equivalent to transactional rollback. It represents an explicit attempt to restore consistency in a distributed environment.

## 10. Non-Linear Causality

Parent-child spans represent direct synchronous execution. The following relationships MUST use span links:

- recovery from checkpoint;
- fan-out to subagents;
- asynchronous work scheduled through queues;
- aggregation of parallel results;
- retries;
- compensation; and
- tool actions triggered by completed model decisions.

Every link MUST declare one relationship type: `resume`, `delegates_to`, `triggered_by`, `compensates`, `retries`, or `aggregates`.

## 11. Metrics

The runtime MUST produce metrics for total Run duration; time to first token, when available; token usage; estimated Run cost; decision, iteration, tool, retry, and compensation counts; completion, failure, cancellation, and compensation rates; checkpoint age at recovery; wait and resume duration; and guardrail decisions.

Unique identifiers such as `run.id` and `thread.id` MUST NOT be metric dimensions because they create excessive cardinality.

Estimated cost MUST include a pricing-table version, currency, and source.

## 12. Logs, Privacy, and Guardrails

Logs complement traces and MUST include `trace_id`, `span_id`, `run.id`, `thread.id`, severity, category, and data classification.

Prompts, responses, tool arguments, tool results, and state snapshots are opt-in telemetry. When enabled, they MUST be subject to classification, redaction, allowlists, and retention policies.

Every security, authorization, privacy, cost, or risk control MUST emit an `agent.guardrail.evaluate` span with the policy identifier and version, outcome, risk category, and no sensitive source content.

## 13. Collection Architecture

```text
Agentic Runtime
  -> OpenTelemetry SDK / OTLP
  -> OpenTelemetry Collector
     -> validation and enrichment
     -> redaction and filtering
     -> sampling and routing
  -> observability backends
```

The OpenTelemetry Collector is the central policy enforcement point for telemetry processing. It MUST apply filtering, redaction, transformation, and routing before export.

Applications remain responsible for data minimization at the source.

## 14. Extensibility and Stability

Where an OpenTelemetry semantic convention exists, it MUST be preferred.

Runtime-specific attributes use the extension namespaces `agent.runtime.*`, `agent.state.*`, `agent.decision.*`, `agent.effect.*`, and `agent.tool.*` until a stable upstream equivalent exists.

These extensions SHOULD be evaluated for upstream standardization after they have been validated in real implementations across more than one framework.

## 15. References

- [OpenTelemetry GenAI Semantic Conventions](https://github.com/open-telemetry/semantic-conventions-genai)
- [OpenTelemetry Trace API](https://opentelemetry.io/docs/specs/otel/trace/api/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Handling Sensitive Data with OpenTelemetry](https://opentelemetry.io/docs/security/handling-sensitive-data/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Saga Patterns](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-patterns.html)
