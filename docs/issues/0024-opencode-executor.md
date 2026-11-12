# Add an optional OpenCode executor adapter

## Context

OpenCode is a useful executor for coding tasks, but it must remain an adapter. LLM output
is untrusted, provider credentials belong to the worker, and the core scheduler must not
depend on an agent runtime.

## Objective

Implement an optional OpenCode executor that runs a non-interactive request and returns
structured results and artifacts.

## Scope

- Add an `opencode` executor type with:
  - Binary path or command name.
  - Working directory inside `allowed_roots`.
  - Prompt or prompt file reference.
  - Optional model/agent arguments.
  - Timeout and output limits.
  - Environment resolution through `envFrom`.
- Run OpenCode non-interactively and capture bounded output.
- Extract a final result payload and any produced artifacts when the adapter supports it.
- Disable the adapter by default.
- Add tests with a fake executable that emits deterministic output.
- Document that model/provider credentials are worker-local and never stored centrally.

## Out of scope

- Managing model providers, API keys, or billing.
- Autonomous approval of destructive actions.
- Running OpenCode as the central service.
- A general plugin system.
- Interactive terminals.

## Acceptance criteria

- [ ] The adapter is refused unless enabled in worker configuration.
- [ ] The binary is invoked with structured arguments, not a shell string.
- [ ] The prompt is bounded and stored or referenced explicitly.
- [ ] Timeout and output limits are enforced.
- [ ] Missing binary returns `EXECUTOR_UNAVAILABLE`.
- [ ] Provider credentials are resolved from the worker environment and never logged.
- [ ] The fake-executable tests cover success, non-zero exit, timeout, and oversized
      output.
- [ ] The domain and scheduler have no OpenCode dependency.

## Dependencies

- Blocked by: #15, #19, #20.
- Blocks: agent-driven workflows beyond the MVP.

## Technical notes

- Define a narrow adapter contract; do not expose OpenCode-specific types to the domain.
- Treat all model output as untrusted text. Do not execute it automatically.
- If the adapter produces files, upload them through the artifact path rather than
  embedding them in snapshots.
- Record the adapter version/command in execution metadata for auditability, without
  secrets.
- A future approval policy for sensitive operations is separate and must be deterministic.

## Validation

```console
$ go test ./internal/executor/opencode/...
$ go test ./... -run OpenCode
```
