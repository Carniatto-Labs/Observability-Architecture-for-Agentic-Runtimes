# Contributing

Thank you for your interest in improving this architectural proposal.

## Before Opening a Pull Request

Please open an issue first for substantial changes. Describe the use case, the runtime or framework involved, the telemetry data available at runtime, and the compatibility impact.

## Contribution Principles

- Prefer established OpenTelemetry semantic conventions when they fit.
- Keep proposed attributes observable, actionable, and framework-agnostic.
- Do not add an attribute without describing the trace, metric, log, or event that records it.
- Avoid collecting prompts, responses, secrets, personal data, or private model reasoning by default.
- Distinguish architectural requirements in `RFC.md` from practical adoption guidance in `ADOPTION_GUIDE.md`.
- Keep pull requests focused and document user-visible changes in `CHANGELOG.md`.

## Feedback We Value

We especially welcome feedback from implementations involving durable workflows, multi-agent orchestration, MCP, stateful runtimes, retries, idempotency, compensation, security controls, and OpenTelemetry instrumentation.

## License

By contributing, you agree that your contributions will be licensed under the Apache License 2.0.
