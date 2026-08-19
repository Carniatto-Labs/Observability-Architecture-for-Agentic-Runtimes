# Adoption Guide: Observability for Production Agents

**Status:** Non-normative implementation guide  
**Version:** 0.1.0

This guide explains a practical path for adopting the target architecture defined in the [RFC](RFC.md). It does not change, reduce, or replace the target architecture.

## The Central Idea

Instead of asking only, "Which model responded?", operations teams need to answer:

- What did this execution attempt to do?
- What state was it in?
- Which decision led to which action?
- What changed outside the system?
- Was there a retry, resumption, or compensation?
- What did it cost, and why did it fail?

The answer is to observe the runtime as a durable workflow.

## The Flow That Must Be Visible

```text
Run
  -> model decision
  -> state transition
  -> tool call
  -> external effect
  -> checkpoint
  -> next decision or termination
```

If a flow fails after producing an external effect, the observability view must clearly show the compensation attempt and its outcome.

## Initial Operational Baseline

An implementation should first establish these five capabilities:

1. One `agent.run` root span per execution.
2. Correlation between `run_id`, `thread_id`, and `checkpoint_id`.
3. Spans for LLM calls, tools, and subagents.
4. Checkpoint and state-transition events.
5. Sensitive-content redaction in the OpenTelemetry Collector.

This baseline makes it possible to investigate an end-to-end failed execution without relying on disconnected logs.

## External-Effect Visibility

The next operational concern is visibility into mutable external effects:

- idempotency;
- deduplication;
- retries;
- classification of mutable operations;
- compensations; and
- estimated cost and Run-level budgets.

At this point, the agent is no longer treated as a chat interface with tools. It is operated as a durable workflow with explicit external consequences.

## Operational Dashboards and Alerts

| Signal | Operational Question |
|---|---|
| Run error rate | Is the system failing? |
| Resume duration | Are checkpoints and recovery working? |
| Retries by tool | Is an external dependency unstable? |
| Executed compensations | Are external effects failing mid-workflow? |
| Cost by Run or thread | Where is consumption occurring? |
| Guardrail blocks | Is a safety policy being triggered? |

## Privacy Rule

Do not send full prompts or complete results by default. Record metadata, hashes, size, classification, and safe references instead.

When content is essential for debugging, enable it explicitly with redaction and limited retention.

## Explainability Rule

Record a concise operational explanation, such as: "The agent selected tool X because data Y was unavailable."

Do not record detailed private model reasoning.

## Expected Outcome

An operator should be able to open any Run and understand, within minutes:

- its origin;
- its state history;
- the involved decisions and tools;
- the external effects produced;
- what was reverted;
- the likely cause of failure; and
- its cost and latency.
