# Effect-Evidence Conformance Vectors

**Suite:** `oaar.effect-evidence` v0.1.0 · **Section:** RFC §9.1 · **File:** [`effect-evidence.v0.1.json`](effect-evidence.v0.1.json)

Thirteen cases that decide one question: does an observation resolve an external effect, or
does it stay unresolved? The JSON file is the interface. A runtime conforms if, for every case,
it reaches the same `(status, reason)` pair **without importing OAAR code**.

The vectors are deliberately semantic, not telemetry. They say nothing about span names,
attribute names, or where the decision is emitted — only about the decision itself. A runtime
that maps OAAR telemetry must be able to produce these pairs from its own records.

## What a case contains

| Field | Meaning |
|---|---|
| `effect` | `logical_id`, `attempt_id`, `domain`, `mutation`. The effect under test. |
| `observation` | One readback bound to an effect, attempt, evidence id + digest, source, observation kind, verdict, age. `null` means nobody looked. |
| `registry` | Trusted evidence records, each bound to the same `(effect, attempt, evidence id, digest)` tuple, plus `verifier_id`, `trust`, `invalidation`, `observed_after_attempt`, `fresh`. |
| `policy` (suite level) | For each effect domain and observation kind, the sources authorized to speak about it. |
| `freshness_ms` (suite level) | The window beyond which an observation is stale. |
| `expect` | The `status` and `reason` the runtime must reach. |

## Evaluation order

Reason precedence is part of the contract. A case may violate several rules; the runtime must
report the first one in this order:

1. `no_observation` — nothing was read back;
2. `evidence_for_other_effect` — the observation is bound to a different logical effect;
3. `evidence_for_other_attempt` — bound to a different dispatch of the same effect;
4. `unknown_verdict` — the verdict is not `CONFIRMED` or `CONTRADICTED`;
5. `evidence_not_registered` — the trusted registry has no record for the exact id and digest;
6. `evidence_not_trusted` — the record exists but is not trusted;
7. `evidence_invalidated` — the record was retracted;
8. `source_not_authorized` — the policy does not let this source speak for this domain and kind;
9. `observed_before_attempt` — the observation predates the dispatch;
10. `stale_evidence` — the observation is older than the freshness window, or the record says it
    is not fresh;
11. otherwise `confirmed` / `unconfirmed` with reason `confirmed` / `contradicted`.

`unconfirmed` is knowledge, not failure. `unresolved` is neither success nor failure and must be
counted as its own status.

## Running them

```bash
# reference runner (this repo's implementation)
python -m oaar_impl.vectors

# any runtime: implement resolve(case) -> {status, reason} and compare with expect
```

The file is self-contained: no fixture imports, no shared library, no network. A conforming
runner is roughly 80 lines, as the reference one demonstrates.

## Reporting conformance

State the suite version, the spec version, and the number of cases that matched:

> `oaar.effect-evidence` v0.1.0 (RFC §9.1): 13/13.

This is a claim about semantics, not about trustworthiness of any particular external system.
Passing the vectors does not mean a device, ledger or acquirer is authoritative; it means the
runtime does not promote transport evidence, stale evidence, estimates or self-declared sources
into effect evidence.

## Changing the suite

Adding a case is additive; changing `expect`, the evaluation order, or an existing field is a
breaking change and requires a new file version. Consumers pin the file (or its digest) so a
historical conformance result cannot change underneath them.
