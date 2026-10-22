# Changelog

All notable changes to this project are documented in this file.

## [0.2.0] - 2026-10-22

### Added

- **Logical Operation** as a first-class entity (§3), with `agent.operation.id` carried on events only.
- **Three levels of outcome** — execution, artifact, business — with disjoint vocabularies (§3.1).
- `agent.artifact.outcome` on the run span: `produced` / `pending` / `not_applicable` (§6.1).
- **Exactly-once business outcome accounting** per `(operation, definition, version)`: duplicates suppressed, conflicts flagged, non-terminal values never counted (§11.2).
- **Denominator declaration** rule: every rate declares unit and cohort; execution and business rates must not be joined (§11.1).
- Effect **intent / attempt / evidence** separation, logical effect id vs attempt id, and the `unresolved` confirmation state (§9.1).
- **Reference implementation** section and validation-status table (§16), plus the two-repository contract in `REFERENCE_IMPLEMENTATION.md`.

### Changed

- `enduser.id` → `enduser.pseudo.id` (§4), matching the pseudonymous identifier the section always described.
- `agent.tool.deduplication.status` vocabulary fixed to `unique` / `duplicate` / `not_attempted` (§8).
- §10 now states that a `triggered_by` link is an execution relation, not evidence that the earlier operation formed the later action.
- Extension namespaces list includes `agent.operation.*`, `agent.artifact.*`, and `agent.outcome.*` (§14).

### Fixed

- Broken `CONTRIBUTING.md` link in the README (the file is `Contributing.md`).

## [0.1.0] - 2026-08-19

### Added

- Initial architectural proposal for observability of agentic runtimes.
- Target architecture covering Runs, state transitions, checkpoints, causal links, external effects, compensations, privacy, and guardrails.
- Non-normative adoption guide for production implementation and operations.
- OpenTelemetry, W3C Trace Context, MCP, LangGraph persistence, and Saga-pattern references.

