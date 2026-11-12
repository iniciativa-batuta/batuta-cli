# AGENTS.md

## Language
- Use English for source code, identifiers, file names, technical documentation, comments, commit messages, branches, pull requests, and issues.
- Keep user-facing product copy in the language required by its context.
- Write concise, direct comments that explain intent or non-obvious decisions. Do not narrate obvious code.

## Engineering principles
- Prefer the simplest design that satisfies a demonstrated requirement.
- Keep modules cohesive and responsibilities explicit. A module should have one clear reason to change.
- Avoid duplicated logic, duplicated domain rules, and parallel implementations of the same contract.
- Before adding a helper, abstraction, interface, package, or dependency, check whether an existing one already solves the problem.
- Do not introduce speculative extension points, generic frameworks, or configuration options without a concrete use case.
- Keep files focused and reasonably small. Split large files by responsibility, not by arbitrary line counts.
- Do not create empty packages, placeholder layers, or directories without an established responsibility.
- Prefer explicit data structures and contracts over implicit conventions or loosely structured strings.

## Architecture and boundaries
- Keep domain rules independent from transport, persistence, CLI frameworks, operating-system details, and specific executors.
- Put application orchestration in application/use-case modules; keep infrastructure adapters at the edges.
- The central service owns authoritative Task and Execution lifecycle state.
- A Worker owns the runtime state of its local execution Steps. The central service reads that state through the defined Worker interface; Workers must not push routine Step progress notifications to the central service.
- Keep Worker and central-service responsibilities separate. Share versioned contracts and protocol code where useful, but do not couple one runtime to the other's internal implementation.
- Treat executors such as shell, OpenCode, Python, Docker, and Git as adapters. The core domain must not depend on any particular executor.
- Keep the Web interface optional and make it consume the same public application API as the CLI.

## Reliability and execution
- Model Task, Execution, Step, and Worker states separately.
- Define and validate legal state transitions at the boundary that owns each state.
- Make retries, cancellation, timeouts, and failure handling explicit. Do not silently reinterpret errors.
- Use stable identifiers and idempotent operations for requests that may be repeated.
- Do not use logs as a substitute for structured state or execution results.
- Bound process output, payload sizes, execution duration, and resource consumption.
- Never interpolate untrusted input into shell commands. Use structured process execution by default and an explicit shell executor when shell semantics are required.
- Do not store secrets in source control, plain-text task definitions, or logs.

## Dependencies and implementation
- Prefer the Go standard library when it adequately solves the problem.
- Every new third-party dependency must have a clear technical justification and a maintenance/security cost worth accepting.
- Follow the repository's actual Go version, tooling, formatting, linting, and test conventions. Verify them before changing project configuration.
- Do not add infrastructure such as brokers, databases, containers, or orchestration platforms unless a concrete requirement justifies it.

## Testing and documentation
- Test domain rules, state transitions, failure paths, cancellation, retries, and protocol behavior.
- Add or update tests with behavior changes. Prefer focused tests that verify observable behavior.
- Run the relevant checks when practical and report exactly which commands ran and whether they passed.
- Never claim that tests, builds, or checks passed if they were not executed.
- Update documentation and versioned contracts when behavior, configuration, commands, or public interfaces change.
- Keep examples consistent with the actual implementation.

## Git and repository safety
- Inspect the working tree and existing conventions before making changes.
- Never discard, overwrite, or reformat unrelated user changes.
- Do not commit, push, publish releases, create external resources, or perform destructive operations unless explicitly authorized.
- Do not add generated files or local machine state unless the repository requires them.
- Keep changes scoped to the requested task and summarize files changed, checks run, and known limitations.
