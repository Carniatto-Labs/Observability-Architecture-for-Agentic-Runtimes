# Adoption Guide: Observability for Production Agents

**Status:** Non-normative implementation guide  
**Version:** 0.2.0

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

## Three Levels of Outcome

Before adding a single panel, decide which question each one answers. The same execution can
be healthy while the operation produced nothing, and the operation can produce its artifact
while the business says no.

| Level | Question | Counted per | Vocabulary |
|---|---|---|---|
| Execution | did the run finish? | Run | completed / failed / cancelled |
| Artifact | did the operation produce its artifact? | Run, rolled up to operation | produced / pending / not_applicable |
| Business | did the customer accept it? | Logical Operation | accepted / declined / pending / unknown |

Rules that prevent the usual mistakes:

- `waiting`, `checkpointed`, and `cancelled` are not failures. A run parked for human approval
  is doing exactly what it was designed to do.
- A declined business outcome is a **completed** execution. It belongs on an event
  (`agent.outcome.observed`), never in the `failed` series.
- The execution panel and the operation panel are never joined into one number. Show both, side
  by side, labelled.

## Worked Example: One Operation, Two Executions, Two Operations, Three Outcomes

The scenario every implementation should be able to render, and the one the reference
implementation runs as `make abc-smoke`:

| Run | Thread | Logical operation | Result | Execution | Artifact |
|---|---|---|---|---|---|
| A | T1 | P | checkpoint saved (state v3), then fails before producing P | failed | pending |
| B | T1 | P | restores A's checkpoint v3, produces P | completed | produced |
| C | T2 | Q | produces Q | completed | produced |

Business results arrive after every run has ended: P is declined, Q is accepted.

| Denominator | Value | What it answers |
|---|---|---|
| Execution completion | 2/3 | is the runtime able to execute what it was asked to execute? |
| Artifact production | 2/2 | did the work get done, regardless of how many attempts it took? |
| Customer acceptance | 1/2 | did the business get what it wanted? |

All three numbers are correct. They are not reconciled, averaged, or joined: each one has its
own unit, its own cohort, and its own panel. Counting distinct `thread.id` would not recover
the acceptance rate — T1 continues with later, unrelated work.

Two additional cases belong in the same example, and both must be visible:

- if Q's decision has not arrived at the reporting cut, its outcome is `unknown`, not an
  invented rejection, and the acceptance denominator is reported with the cut-off stated;
- if the same operation reports its terminal outcome twice (a retry after a restore), the
  observation is kept as history and **not** counted twice.

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
| Business outcomes by operation | Did the work we did turn into what the business wanted? |
| Suppressed duplicate outcomes / flagged conflicts | Is our business denominator trustworthy? |

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

And, at the operation level, answer the two questions the Run cannot answer:

- did this operation ever produce its artifact; and
- what did the business decide, and when did we learn it.

## Reproducing This Guide

This guide is non-normative and stays valid on its own. The numbers in the worked example come
from the OAAR reference implementation, which lives in a **separate repository** pinned to the
same version as this specification; see
[REFERENCE_IMPLEMENTATION.md](REFERENCE_IMPLEMENTATION.md) for the pin, the commands, and the
list of what it does and does not prove.

