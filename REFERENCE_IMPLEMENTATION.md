# Reference Implementation Contract

**Spec version:** 0.4.1  
**Implementation:** OAAR reference implementation — **separate repository** at tag `v0.4.1`  
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
| This repo | implementation tag `v0.4.1` |
| Implementation repo | spec version `0.4.1`, recorded in `spec.json` and printed by `make validate` |

A change to either side that alters attribute names, vocabularies, or required fields is a
**spec change first**, then a harness change, then a pin bump on both sides. A change that
alters scenarios, dependencies, or dashboards only bumps the harness.

## Reproduce

Requirements: Docker (or colima) with ~4 GB free, and `ollama pull gemma4:12b-mlx` only if you
want the interactive assistant. The validation harness itself does not need a model.

```bash
git clone https://github.com/Carniatto-Labs/oaar-reference-impl && cd oaar-reference-impl
git checkout v0.4.1
make up          # pinned stack: app, product-api, payment-api, postgres, collector, tempo, prometheus, loki, grafana
make validate    # runs every demo + verify pair, prints the RFC section -> evidence matrix
```

Expected last lines:

```
individual checks: 80 pass, 0 fail
ALL SCENARIOS VALIDATED
```

Last verified end to end: **2026-10-22**, `80 pass / 0 fail` on a local stack (colima, Docker,
aarch64), with 14/14 vectors and 8/8 dashboard queries. A later run that fails is a real result,
not a documentation bug.

Individual scenarios can be run alone, each with its own target (`make abc-smoke`,
`make outcome-dedup-smoke`, `make hitl-smoke`, `make saga-smoke`, `make effect-smoke`,
`make fanout-smoke`, `make guardrail-smoke`, `make mcp-smoke`, `make vectors`,
`make dash-smoke`).

## What the pin covers

Per RFC §16, at tag `v0.4.1` the harness exercises §4–§8, §9 and §9.1 (effect evidence, including
lost acknowledgement, stale / missing / unauthorized readback), §9.2 (verified and unverified
compensation), §10 (typed links), §11 and §11.2, §12, and §13. The full section-by-section
matrix with the evidence behind each row is in the implementation's README.

The rows that matter for the denominators:

| Claim | Evidence |
|---|---|
| Execution completion 2/3 while artifacts are 2/2 and acceptance 1/2 | `make abc-smoke` — Run A fails after a checkpoint, Run B restores it, Run C is a separate operation |
| `pending` at the reporting cut is not a failure | `make abc-unknown-smoke` |
| One business outcome per operation, retries suppressed, conflicts flagged | `make outcome-dedup-smoke` — 3 operations / 4 runs / 6 observations → +3 |
| A lost acknowledgement leaves the effect `unresolved`; only a fresh, registry-corroborated readback from an authorized source resolves it | `make effect-smoke` — confirmed / contradicted / stale / missing / estimated, 1 confirmed, 1 unconfirmed, 3 unresolved |
| The §9.1 decision is reproducible without this project's code | `make vectors` — 13/13 cases of `oaar.effect-evidence` v0.1.0, digest-pinned to this spec |
| The relation to an independent execution contract is stated, not implied | [`mappings/aea-effect-authority.v0.1.md`](mappings/aea-effect-authority.v0.1.md) — nine requirements, one relation each, excerpt digests pinned to a source commit |
| `waiting` and `cancelled` are not failures | `make hitl-smoke`, `make guardrail-smoke` |
| Unique ids never become metric labels | enforced at emit time and asserted against the Prometheus series dump |

## What the pin does not cover

Stated here so the pin is not read as broader than it is:

1. **The authoritative source is simulated.** `effect-smoke` exercises lost acknowledgements,
   stale and missing readbacks, an unauthorized source and unverified compensation, but the
   readback comes from an in-process registry rather than a real device, ledger or acquirer.
   The semantics are exercised; the trustworthiness of a specific external system is not.
   The harness does not run the AEA verifier and does not claim conformance to it.
2. **One framework.** LangGraph only; RFC §14 asks for more than one, and Temporal is the
   planned second.
3. **The outcome registry is process-local.** The scenario proves the accounting rule, not the
   durability of the store behind it (production backs it with a unique index).
4. **Run vs upstream `execution.id`.** The harness implements this spec's choice (restore = new
   Run). The mapping to an upstream execution identifier is declared in RFC §4 but is not
   exercised against a runtime that preserves one.

## Citing the pair

> OAAR v0.4.1, validated by the OAAR reference implementation at tag `v0.4.1` (80/80 checks,
> 13 scenarios, including 13/13 portable effect-evidence vectors; LangGraph + Postgres
> checkpoints + OpenTelemetry → Tempo/Prometheus/Loki).

Citing one version without the other is not a conformance claim.
