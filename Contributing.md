# Contributing

Thank you for helping improve this architectural proposal. We welcome feedback from practitioners, researchers, platform teams, runtime maintainers, and OpenTelemetry contributors.

This repository is an open proposal. Changes should make the architecture more observable, interoperable, implementable, and safe across agentic runtimes.

## Ways to Contribute

You can contribute by:

- asking a question in GitHub Discussions;
- reporting implementation feedback through an issue;
- proposing a change to the RFC, attribute model, or terminology;
- improving documentation, references, or examples; and
- validating the architecture in a real runtime or observability backend.

## Start with the Right Place

Use **GitHub Discussions** for open-ended questions, implementation experience, and early architectural debate.

Use **GitHub Issues** when the topic is actionable and can be tracked to a decision or pull request. For a substantial change, open an issue before opening a pull request.

Use a **Pull Request** only when there is a concrete, focused change ready for review.

## Before Proposing a Substantial Change

Open an Architecture Proposal issue and describe:

1. The problem or runtime behavior that is not sufficiently observable.
2. The runtime, framework, protocol, or backend where the problem occurs.
3. The telemetry data available at runtime.
4. The proposed span, event, metric, attribute, or requirement.
5. The impact on compatibility, privacy, cardinality, and existing conventions.
6. Whether the proposal applies across more than one framework or implementation.

Please do not submit a large pull request before the proposal has received initial maintainer feedback.

## Contribution Workflow

1. Fork the repository to your GitHub account.
2. Create a branch from `main` in your fork.
3. Make one focused change.
4. Update `CHANGELOG.md` when the change affects readers or implementers.
5. Open a pull request targeting `main`.
6. Respond to review feedback and keep the pull request focused on its original purpose.

The `main` branch is protected. Changes are merged through pull requests after review.

## Branch Naming

Use a short, descriptive branch name with one of these prefixes:

```text
docs/improve-adoption-guide
proposal/add-checkpoint-event
feedback/langgraph-persistence
fix/correct-trace-context-reference
```

## Commit Messages

Use concise Conventional Commit-style messages:

```text
docs: clarify checkpoint recovery semantics
docs: add adoption guidance for external effects
fix: correct W3C Trace Context reference
```

## Pull Request Expectations

Every pull request should:

- explain the problem and the proposed change;
- identify whether it changes `RFC.md`, `ADOPTION_GUIDE.md`, or both;
- avoid unrelated formatting or refactoring;
- cite a standard, implementation, or runtime evidence when introducing a new semantic requirement;
- explain how the change can be captured at runtime; and
- avoid including sensitive data in examples, traces, prompts, or screenshots.

## Contribution Principles

- Prefer established OpenTelemetry semantic conventions when they fit.
- Keep proposed attributes observable, actionable, and framework-agnostic.
- Do not add an attribute without describing the trace, metric, log, or event that records it.
- Treat privacy, security, cost, and telemetry cardinality as architectural concerns.
- Do not collect prompts, responses, secrets, personal data, or private model reasoning by default.
- Distinguish normative requirements in `RFC.md` from non-normative operational guidance in `ADOPTION_GUIDE.md`.
- Keep pull requests small enough to review and discuss clearly.

## Review and Decision Process

Maintainers review proposals for:

- cross-runtime applicability;
- alignment with OpenTelemetry and W3C standards;
- feasibility of instrumentation;
- operational usefulness;
- privacy and security implications; and
- backward-compatibility impact.

Architecture decisions should be documented in the related issue or discussion. Changes that alter established attribute names, semantics, or compatibility expectations should be labeled as `breaking-change` and discussed before merge.

## Feedback We Value

We especially welcome feedback from implementations involving durable workflows, multi-agent orchestration, MCP, stateful runtimes, retries, idempotency, compensation, security controls, and OpenTelemetry instrumentation.

## Code of Conduct

By participating, you agree to follow the [Code of Conduct](CODE_OF_CONDUCT.md).

## License

By contributing, you agree that your contributions will be licensed under the Apache License 2.0.
