# RFC: Observability Architecture for Agentic Runtimes

**Status:** Architectural Proposal  
**Version:** 0.4.0  
**Last updated:** 2026-10-22  
**Scope:** Agent execution runtimes, their durable workflows, tools, external effects, and distributed integrations.

**Changes in 0.4.0**: portable conformance vectors for §9.1 (`vectors/`, suite
`oaar.effect-evidence` v0.1.0) so the rule can be tested by a runtime that imports nothing from
this project.

**Changes in 0.3.0** (evidence: reference implementation at `v0.3.0`, see §16): the effect
evidence rule behind §9.1 becomes testable — what a readback must be bound to, and why it
stays `unresolved` — and the conformance table marks §9 exercised.

**Changes in 0.2.0**: logical operation identity, the three levels of outcome, exactly-once
business outcome accounting, effect intent/attempt/evidence separation, and a corrected
end-user attribute.

## 1. Purpose

This specification defines an observability architecture for agentic systems centered on the **execution runtime**.

The agent is not treated as the primary observable unit. The root entity is the temporal execution instance, called a **Run**. A Run represents the concrete fulfillment of an intent: it may last milliseconds or days, pause, resume, delegate work, alter external systems, and execute compensations.

The architecture uses OpenTelemetry as its interoperability foundation and adds a semantic layer for aspects that are not yet fully standardized: durable state, checkpoints, resumption, decision causality, external effects, and compensation.

A Run is also not the only unit a question can be asked about. Operations need executions, and
the business needs outcomes. Section 3 defines the three levels and the identity that connects
them; conflating them is the most common source of wrong rates in agent observability.

## 2. Principles

1. **The runtime is the center of observation.** The model, agent, tool, and workflow are execution participants.
2. **Causality is explicit.** Every material action must be attributable to the decision, state, and context that produced it.
3. **Durability is visible.** Pauses, checkpoints, and resumptions are first-class events.
4. **External effects require dedicated semantics.** Technical success of a call is insufficient to represent its real-world impact.
5. **Telemetry is secure by default.** Sensitive content is not collected by default.
6. **OTel conformance comes first.** Standardized attributes are preferred; runtime extensions use their own namespace.
7. **Levels of outcome are distinct.** Execution success is not artifact production, and artifact production is not business acceptance. Each level has its own identity, vocabulary, and denominator.

## 3. Conceptual Model

```text
Thread or Session
  -> Logical Operation
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
| `Logical Operation` | The unit of work a Run contributes to; the level at which an outcome is counted |
| `Workflow` | A graph, state machine, or process coordinated by the runtime |
| `Decision` | An operational choice that determines the next step |
| `Tool Call` | An invocation of a tool, MCP server, API, process, or function |
| `External Effect` | An observable change outside the runtime |
| `Checkpoint` | Persisted state that enables resumption |
| `Compensation` | An action intended to restore consistency after failure |

### 3.1 The three levels of outcome

| Level | Question | Unit | Vocabulary |
|---|---|---|---|
| Execution | Did the run execute to completion? | Run | `completed`, `failed`, `cancelled` |
| Artifact | Did the operation produce its artifact? | Run → operation | `produced`, `pending`, `not_applicable` |
| Business | Did the customer accept the result? | Logical Operation | `accepted`, `declined`, `pending`, `unknown` |

The levels are not derived from one another. A run can complete while its operation produces
nothing; an operation can produce its artifact while the business declines it; a business
outcome can arrive after every run of the operation has ended. Telemetry MUST preserve the
level an observation belongs to, and consumers MUST NOT promote an observation from one level
to another.

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
| `agent.operation.id` | Logical operation this Run contributes to, when the application defines one |
| `enduser.pseudo.id` | Pseudonymous end-user identifier, when authorized |

`run.id` identifies a specific execution. `thread.id` connects multiple executions that belong
to the same operational history. `trace_id` connects distributed causality within a propagated
context. `operation.id` identifies the work, and is the only one of the four that MAY be shared
by several Runs across several threads.

`agent.operation.id` is application-assigned. It MUST NOT be derived from `thread.id` or
`trace.id`: a thread can carry unrelated operations, and an operation can span threads. It is
carried on events only and MUST NOT be used as a metric dimension (§11).

`agent.runtime.run.id` is the Run. Where an upstream convention preserves a single execution
identifier across suspend and resume, the mapping SHOULD be declared explicitly: this
specification treats a restore as a **new Run** (§6), and an upstream `execution.id` maps to
`agent.operation.id` when the resumed segment continues the same work.

## 5. Run Lifecycle

```text
created -> running -> waiting -> checkpointed -> resumed -> completed
                     |             |              |
                     v             v              v
                   failed ------> compensating -> compensated
```

The runtime MUST emit lifecycle events for creation, start, wait, completion, failure, cancellation, resume, compensation start, and compensation completion.

Every transition MUST record its cause, timestamp, responsible actor, and state version.

The terminal execution outcome is `completed`, `failed`, or `cancelled`. A Run that is waiting,
checkpointed, or blocked by a policy is not failed. Cancellation is a decision, not an error.

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

### 6.1 Artifact outcome

A Run that contributes to a logical operation MUST record the artifact level on `agent.run`:

| Attribute | Vocabulary |
|---|---|
| `agent.artifact.outcome` | `produced`, `pending`, `not_applicable` |

`pending` means the artifact has not been produced and the operation has not resolved to a
state in which it never will be. A Run that ends in failure without producing its artifact is
`pending`, not `not_applicable`. The attribute MUST NOT carry a business value: the
vocabularies are disjoint by design (§3.1).

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
| `agent.tool.deduplication.status` | `unique`, `duplicate`, or `not_attempted` |
| `agent.tool.mcp.server` | MCP server identity, when applicable |

The idempotency key MUST be owned by the runtime and stable across retries of the same logical
effect. The target system is responsible for deduplicating on it; a retry that returns the
original decision reports `deduplication.status=duplicate`, and a call that never reached the
target reports `not_attempted`.

Context propagation between clients, MCP servers, subagents, and external services MUST use W3C Trace Context.

## 9. External Effects and Compensation

A call that modifies data, files, infrastructure, financial systems, or any external resource MUST emit an `agent.effect` span.

The effect MUST record a safe resource identifier, mutation type, idempotency key, confirmation status, originating decision, originating tool call, and compensation availability.

### 9.1 Intent, attempt, and evidence are separate

- `agent.effect.id` identifies the **logical effect** and MUST survive retries, replays, and resumptions.
- `agent.effect.attempt.id` identifies one dispatch of that effect; `agent.effect.attempt.index` orders them.
- `agent.effect.confirmation.status` is the **evidence** state: `confirmed`, `unconfirmed`, or `unresolved`.

A transport acknowledgement, a successful span, or a returned tool result is evidence of
dispatch, not of effect. An effect span therefore starts `unresolved`, whatever the transport
said.

A readback is its own observation, on an `agent.effect.observe` span, carrying
`agent.effect.evidence.id`, `.digest`, `.source`, `.kind`, `.verdict`, `.age.ms` and `.fresh`,
bound to the effect and the attempt it observes. Such an observation resolves the effect only
when all of the following hold:

1. it is bound to this logical effect and this attempt;
2. the trusted evidence registry corroborates the exact evidence id and digest — a self-declared
   source name is not evidence;
3. the policy authorizes that source for this effect domain and observation kind, so a model
   estimate or a cached value cannot resolve a durable or physical effect;
4. it was observed after the attempt; and
5. it is fresh.

Otherwise the status MUST remain `unresolved`, with the reason on
`agent.effect.resolution.reason`. An observation that contradicts the intended effect resolves
it as `unconfirmed`; that is knowledge, not failure. An unresolved effect MUST be counted by
status and MUST NOT be reported as a success or a failure.

Reconciliation against an authoritative source is outside the instrumented span; the span
records that it has not happened yet.

Portable conformance vectors for this section — inputs, evaluation order and expected
`(status, reason)` pairs — are published in [`vectors/`](vectors/README.md). They are the
interface: a runtime conforms by reaching the same pairs without importing anything from this
project.

### 9.2 Compensation

When a defined inverse action exists, the runtime MUST create an `agent.compensate` span. Compensation MUST be linked to the original effect through a span link and `agent.effect.id`.

Compensation outcomes are `succeeded`, `failed`, `partial`, or `not_applicable`. Requesting a
compensation is not evidence that it happened: `succeeded` is reserved for a verified outcome.

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

Every link MUST declare one relationship type: `resume`, `delegates_to`, `triggered_by`, `compensates`, `retries`, or `aggregates`. OpenTelemetry defines no standard link-type attribute; the runtime carries it as `agent.runtime.link.type` on the link itself.

A link is an **execution relation**, and MUST NOT be read as evidence that the content of the
earlier operation formed or selected the later action. When that stronger claim is not
observable, the relationship is unresolved, and consumers MUST NOT infer it from correlation,
temporal order, or textual similarity alone.

## 11. Metrics

The runtime MUST produce metrics for total Run duration; time to first token, when available; token usage; estimated Run cost; decision, iteration, tool, retry, and compensation counts; completion, failure, cancellation, and compensation rates; checkpoint age at recovery; wait and resume duration; and guardrail decisions.

Unique identifiers such as `run.id` and `thread.id` MUST NOT be metric dimensions because they create excessive cardinality.

### 11.1 Denominators

Every rate MUST declare its unit and its cohort. Execution rates count Runs. Business rates
count Logical Operations. A rate whose numerator and denominator come from different levels is
invalid, and the two MUST NOT be joined into a single panel.

### 11.2 Exactly-once accounting for business outcomes

A business outcome is counted once per `(agent.operation.id, agent.outcome.definition,
agent.outcome.definition.version)`:

- a repeated terminal value is kept as history and MUST NOT increment the counter;
- a non-terminal value (`pending`, `unknown`) never counts and never blocks a later terminal value;
- a conflicting terminal value MUST NOT overwrite the recorded one; the conflict is recorded and flagged.

Estimated cost MUST include a pricing-table version, currency, and source.

## 12. Logs, Privacy, and Guardrails

Logs complement traces and MUST include `trace_id`, `span_id`, `run.id`, `thread.id`, severity, category, and data classification.

Prompts, responses, tool arguments, tool results, and state snapshots are opt-in telemetry. When enabled, they MUST be subject to classification, redaction, allowlists, and retention policies.

Every security, authorization, privacy, cost, or risk control MUST emit an `agent.guardrail.evaluate` span with the policy identifier and version, outcome, risk category, and no sensitive source content.

Business outcome subjects and end-user identifiers are linkable data. They MAY appear on
events and logs; they MUST NOT appear as metric dimensions.

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

Runtime-specific attributes use the extension namespaces `agent.runtime.*`, `agent.operation.*`, `agent.state.*`, `agent.decision.*`, `agent.effect.*`, `agent.artifact.*`, `agent.outcome.*`, and `agent.tool.*` until a stable upstream equivalent exists.

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
- [OAAR reference implementation](REFERENCE_IMPLEMENTATION.md)
- [Effect-evidence conformance vectors](vectors/README.md)

## 16. Reference implementation and validation status

The specification and its reproduction are separate repositories, pinned to the same version.
The coupling between them is [REFERENCE_IMPLEMENTATION.md](REFERENCE_IMPLEMENTATION.md); no
other file in this repository depends on the implementation.

Conformance status of this version:

| Section | Requirement | Status |
|---|---|---|
| §4 | Run identity, thread, operation, parent run, pseudonymous end user | Exercised |
| §5 | Lifecycle events; waiting and cancelled are not failures | Exercised |
| §6 | State transitions, checkpoint save/restore, restore = new Run | Exercised |
| §6.1 | Artifact outcome, vocabulary disjoint from business | Exercised |
| §7 | Decision spans, closed kind vocabulary, sanitized basis | Exercised |
| §8 | Tool semantics, W3C propagation, idempotent retry with dedup | Exercised |
| §9 | Effect span, logical effect id, intent/attempt/evidence separation | Exercised |
| §9.1 | `unresolved` effect evidence: lost ACK, stale / missing / unauthorized readback | Exercised in traces and by the portable vector suite (13 cases) |
| §9.2 | Compensation requested vs verified | Exercised (verified and unverified paths) |
| §10 | Typed links; link is not content-derivation evidence | Exercised (typed links), guidance only for the semantic limit |
| §11 | Run-level and operation-level denominators, no unique-id dimensions | Exercised |
| §11.2 | Exactly-once business outcome accounting | Exercised |
| §12 | Correlated logs, opt-in content, guardrail span | Exercised |
| §13 | Collector as enforcement point | Exercised |

Every requirement in this version is exercised by the pinned implementation. What the
implementation cannot prove is that a *particular* external system is trustworthy: its
authoritative source is in-process, so the claim is about the semantics surviving lost
acknowledgements, stale and missing readbacks, unauthorized sources and unverified compensation
— not about any real device or ledger.
