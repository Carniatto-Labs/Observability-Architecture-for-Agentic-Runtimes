# Reference Implementation Contract

**Spec version:** 0.2.0  
**Implementation:** OAAR reference implementation — **separate repository** at tag `v0.2.0`  
**Location:** https://github.com/Carniatto-Labs/oaar-reference-impl

This file is the only coupling between the specification and its reproduction. They are
deliberately separate repositories: the spec versions on its own, the harness versions on its
own, and this page states exactly which pair of versions was measured together.

## Why two repositories

- **Different lifespans.** The spec changes when the semantics change; the harness changes when
  a scenario, dependency, or backend changes. One repo would force version bumps on both sides
  for either kind of change.
- **Different consumers.** Instrumentation authors read the spec; operators and reviewers run
  the harness. Neither should have to clone the other.
- **Different failure modes.** A red `make validate` is evidence about a runtime, not about the
  specification. Keeping them apart prevents a broken demo from looking like a broken standard.

What keeps them together is this file plus a version pin in both directions.

## Pin

| Side | Pins |
|---|---|
| This repo | implementation tag `v0.2.0` |
| Implementation repo | spec version `0.2.0`, recorded in `spec.json` and printed by `make validate` |

A change to either side that alters attribute names, vocabularies, or required fields is a
**spec change first**, then a harness change, then a pin bump on both sides. A change that
alters scenarios, dependencies, or dashboards only bumps the harness.

## Reproduce

Requirements: Docker (or colima) with ~4 GB free, and `ollama pull gemma4:12b-mlx` only if you
want the interactive assistant. The validation harness itself does not need a model.

```bash
git clone https://github.com/Carniatto-Labs/oaar-reference-impl && cd oaar-reference-impl
git checkout v0.2.0
make up          # pinned stack: app, product-api, payment-api, postgres, collector, tempo, prometheus, loki, grafana
make validate    # runs every demo + verify pair, prints the RFC section -> evidence matrix
```

Expected last lines:

```
individual checks: 60 pass, 0 fail
ALL SCENARIOS VALIDATED
```

Individual scenarios can be run alone, each with its own target (`make abc-smoke`,
`make outcome-dedup-smoke`, `make hitl-smoke`, `make saga-smoke`, `make fanout-smoke`,
`make guardrail-smoke`, `make mcp-smoke`, `make dash-smoke`).

## What the pin covers

Per RFC §16, at tag `v0.2.0` the harness exercises §4–§8, §9 (effect span and compensation on
the verified path), §10 (typed links), §11 and §11.2, §12, and §13. The full section-by-section
matrix with the evidence behind each row is in the implementation's README.

The rows that matter for the denominators:

| Claim | Evidence |
|---|---|
| Execution completion 2/3 while artifacts are 2/2 and acceptance 1/2 | `make abc-smoke` — Run A fails after a checkpoint, Run B restores it, Run C is a separate operation |
| `pending` at the reporting cut is not a failure | `make abc-unknown-smoke` |
| One business outcome per operation, retries suppressed, conflicts flagged | `make outcome-dedup-smoke` — 3 operations / 4 runs / 6 observations → +3 |
| `waiting` and `cancelled` are not failures | `make hitl-smoke`, `make guardrail-smoke` |
| Unique ids never become metric labels | enforced at emit time and asserted against the Prometheus series dump |

## What the pin does not cover

Stated here so the pin is not read as broader than it is:

1. **`unresolved` external effects (§9.1)** are not exercised. Every effect in the harness is
   `confirmed`. Dispatch-with-lost-acknowledgement, reconciliation against an authoritative
   source, and compensation-requested-but-not-verified are the next scenarios to build.
2. **One framework.** LangGraph only; RFC §14 asks for more than one, and Temporal is the
   planned second.
3. **The outcome registry is process-local.** The scenario proves the accounting rule, not the
   durability of the store behind it (production backs it with a unique index).
4. **Run vs upstream `execution.id`.** The harness implements this spec's choice (restore = new
   Run). The mapping to an upstream execution identifier is declared in RFC §4 but is not
   exercised against a runtime that preserves one.

## Citing the pair

> OAAR v0.2.0, validated by the OAAR reference implementation at tag `v0.2.0` (60/60 checks,
> 11 scenarios, LangGraph + Postgres checkpoints + OpenTelemetry → Tempo/Prometheus/Loki).

Citing one version without the other is not a conformance claim.
