# Semantic Mapping — OAAR §9.1 ↔ Agent Effect Authority (AEA)

**Status:** non-normative comparison, published by OAAR
**Source:** [`hippoley/SpatialRuntime`](https://github.com/hippoley/SpatialRuntime) @ `a588b24d8998f1284622e12cfe1d8564f860be90`
**Files compared:** `interop/agent-effect-authority/manifest.json` (`agent-effect-authority.v0.1`) and `interop/agent-effect-authority/README.md`
**AEA has not reviewed, adopted or endorsed OAAR, and this document is not a conformance claim.**

## Why this exists

Two projects arrived at the same boundary from different directions. AEA is an execution contract: it
says what a runtime must not conflate before and after a consequential action. OAAR is an observability
architecture: it says what must remain distinguishable in telemetry. The overlap is real, and so are
the gaps — in both directions.

The point of writing it down is that "our semantics look like yours" is not checkable. A relation per
requirement, pinned to a commit and an excerpt digest, is.

## How to read the relations

| Relation | Meaning |
|---|---|
| `equivalent` | The requirement is expressible in OAAR, with the same observable consequence. |
| `overlaps` | Both cover the same ground; each is stricter or looser on a specific axis, noted per row. |
| `weaker` | OAAR covers part of it and cannot represent the rest. |
| `stronger` | OAAR requires more than AEA does on this point. |
| `orthogonal` | Different questions; no mapping is honest. |

## Summary

| AEA | Requirement | Relation | OAAR |
|---|---|---|---|
| `AEA-001` | Proposal is not authority | **orthogonal** | — |
| `AEA-002` | Logical effect identity is runtime-owned | **overlaps** | §9.1 |
| `AEA-003` | Effect class is known before execution | **equivalent** | §8 |
| `AEA-004` | Authorization is explicit and separate | **weaker** | §5, §12 |
| `AEA-005` | Transport evidence is not effect evidence | **equivalent** | §9.1 |
| `AEA-006` | Unresolved outcome is first-class | **equivalent** | §9.1, §11 |
| `AEA-007` | Retry identity prevents duplicate intent | **overlaps** | §8, §9.1 |
| `AEA-008` | Compensation, when supported, is a distinct linked effect | **overlaps** | §9.2 |
| `AEA-009` | Fresh authoritative state closes the loop | **overlaps** | §9.1 |

## Requirement by requirement

### AEA-001 — Proposal is not authority

> Model, planner, or agent output is treated as a proposal and cannot by itself authorize a consequential state or external effect.

`sha256:9f29a725b2da3b910f1f6a50f2851db98e39a461be1ce95fbb70d40e67da7a4f` · relation **orthogonal** · OAAR —

OAAR is an observability architecture: it records what a runtime did, not who may authorize it. The nearest observable is `agent.guardrail.evaluate` (§12), which records that a policy ran and what it decided, and `cancelled` (§5), which records that the run stopped short. Neither is an authority model.

### AEA-002 — Logical effect identity is runtime-owned

> Canonical logical effect identity is allocated or validated by trusted runtime code and cannot be forged by model output.

`sha256:3dbce7e000ce80b3f90e910358e449aed4da5c177ea973bb736be8a19847b057` · relation **overlaps** · OAAR §9.1

The runtime allocates `agent.effect.id`, and it must survive retries, replays and resumptions. OAAR does not state a rule about refusing a model-supplied identity; it states that the identity is runtime-owned by construction.

### AEA-003 — Effect class is known before execution

> The runtime distinguishes read-only from mutating or otherwise consequential effects before dispatch so policy, approval, and retry rules can be applied prospectively.

`sha256:dec8537acd610a045c75556096c3b73cec7e73870b1c5fad032c98cd7a693196` · relation **equivalent** · OAAR §8

`agent.tool.effect.class` is a closed vocabulary (`read_only`, `idempotent_write`, `mutable_write`) known before the call, which is what a prospective policy gate needs. The telemetry form is equivalent to the requirement.

### AEA-004 — Authorization is explicit and separate

> Intent, authorization, dispatch, acknowledgement, and observed outcome remain distinct lifecycle states; pending approval must not be represented as authorization or execution.

`sha256:7b00658928263640bee8656ff8f0c2a22b0724d69bdb7e98dd89296aef3f1369` · relation **weaker** · OAAR §5, §12

OAAR has no approval state. A guardrail span records the evaluation; a `cancelled` run records the outcome. 'Approval pending' is not representable, so OAAR cannot distinguish a held proposal from an authorized one.

### AEA-005 — Transport evidence is not effect evidence

> Successful dispatch or transport acknowledgement must not by itself prove that the intended external or physical effect occurred.

`sha256:3182ee990e95b9cdd92cbbe899daecc45284c02361cf1e2a71c07b25ec94f74a` · relation **equivalent** · OAAR §9.1

Stated explicitly, and enforced by shape: the effect span starts `unresolved` and only a readback can change it.

### AEA-006 — Unresolved outcome is first-class

> When authoritative effect evidence is absent, stale, contradictory, or unavailable, the runtime preserves an unresolved state instead of inferring success or failure from transport behavior.

`sha256:d1871f87566c6b69cb347a1ca2228dc96b558dc0b0d80c65fedb3dff52f2137a` · relation **equivalent** · OAAR §9.1, §11

`unresolved` is first-class, carries a reason, and is counted by status — never folded into success or failure.

### AEA-007 — Retry identity prevents duplicate intent

> Retries preserve one logical effect identity while attempts remain uniquely identifiable; retry handling must not silently create a second intentional mutation.

`sha256:52e5fd8d6a9573b4daeb4503e8918d944a64f85a308653a8f57ffc15c85cdeba` · relation **overlaps** · OAAR §8, §9.1

`agent.effect.id` is shared across attempts while `agent.effect.attempt.id` and `.index` are unique. Like AEA v0.1, OAAR does not normatively fix the persistence point of that identity before a replayable nondeterministic step.

### AEA-008 — Compensation, when supported, is a distinct linked effect *(conditional)*

> A runtime may omit compensation. If compensation is supported, it must be represented as a distinct effect related to the original, and compensation requested, dispatched, acknowledged, and verified must not be collapsed into one state. A runtime that does not support compensation must not imply that reconciliation or retry is equivalent to rollback.

`sha256:d70a32e992805a4da5273ce1a24dbaf0c7d36524a888390c362566b36a5320b9` · relation **overlaps** · OAAR §9.2

`agent.compensation.state` separates `requested`, `unverified` and `verified`, and an outcome is rejected at emit time unless the state is verified. Weaker than AEA on granularity: dispatch and acknowledgement are not separate states in OAAR.

### AEA-009 — Fresh authoritative state closes the loop

> A subsequent decision that depends on an external effect uses fresh authoritative state when required, rather than treating predicted, cached, or merely acknowledged state as physical truth.

`sha256:fbb7de30f8e65ee15e055a8489368b374e21e019e70a1998e983581d20c637d1` · relation **overlaps** · OAAR §9.1

Freshness is required for an observation to resolve an effect. That the *next* decision consumes fresh authoritative state is demonstrated by the reference scenario but not stated as a requirement.

## Where AEA is stronger

- **Authority before action.** AEA requires proposal, authorization, dispatch and acknowledgement to be
  distinct states. OAAR observes the dispatch and the effect but has no approval state at all
  (`AEA-001`, `AEA-004`). A runtime that implements AEA can be observed by OAAR; a runtime that only
  implements OAAR cannot express "authorized but not yet dispatched".
- **Trust provenance.** AEA v0.2 requires the evidence claim to resolve through a trusted registry.
  OAAR states the same rule (§9.1, checks 2 and 3) and implements it, but AEA got there first and states
  it normatively for execution, not only for telemetry.

## Where OAAR adds something AEA does not address

AEA is explicit that it is not an observability standard, so these are not gaps in AEA — they are the
other half of the problem:

- **Three levels of outcome** (execution / artifact / business) with disjoint vocabularies (§3.1, §6.1).
- **Exactly-once business outcome accounting** per logical operation (§11.2) — a rate built on retried
  attempts is not a rate.
- **Denominator declaration** and the cardinality rule for metrics (§11.1).
- **Typed causal links**, and the limit that a link is an execution relation, not evidence that a
  decision formed an action (§10).
- **Log correlation, privacy defaults and Collector enforcement** (§12, §13).

## What the vectors demonstrate, and what they do not

`vectors/effect-evidence.v0.1.json` exercises the `equivalent` and `overlaps` rows for evidence:
`no_observation`, `digest_mismatch`, `evidence_for_other_effect` and `evidence_for_other_attempt`
(`AEA-005`); the stale, unauthorized, invalidated, untrusted, pre-dispatch and unknown-verdict cases
(`AEA-006`); and `confirmed_after_retry` (`AEA-007`). `AEA-008` is exercised by `saga-smoke` and
`effect-smoke`.

Two rows are weaker on evidence than on semantics, and it is worth saying so: `AEA-003` is emitted
by `mcp-smoke` (`read_only`) and by the interactive purchase flow (`mutable_write`), but no check
asserts the vocabulary, so it is exercised as telemetry rather than as a gate. `AEA-001`, `AEA-004`
and `AEA-009` are **not** exercised at all, as the mapping says.

Passing those vectors says the runtime does not promote transport evidence, estimates, caches,
retracted records or self-declared sources into effect evidence. It does not say any external system is
authoritative, and it does not say OAAR conforms to AEA: the AEA verifier was not run, and three
requirements are out of OAAR's scope.

## Provenance

- Manifest: `interop/agent-effect-authority/manifest.json` at `a588b24d8998f1284622e12cfe1d8564f860be90` (`agent-effect-authority.v0.1`, status `experimental`).
- Each `must` excerpt above is the exact JSON string value from that file; the digest is
  `sha256` of its UTF-8 bytes, so a changed requirement cannot silently keep its mapping.
- Relations were assigned by the OAAR authors. An independent mapping from the AEA side would be more
  useful than agreement with this one.

