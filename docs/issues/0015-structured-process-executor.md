# Implement the structured process executor and explicit shell executor

## Context

Running commands as shell strings invites injection and makes behavior
platform-dependent. Batuta's default must be a structured process invocation; shell must
be a separate, disabled-by-default adapter.

## Objective

Implement the executor contract, the process executor, and the explicit shell executor
with timeouts, output limits, working-directory restrictions, and cancellation.

## Scope

- Define an executor interface in `internal/executor` that accepts a context and a
  validated spec and returns a structured result.
- Implement `process`:
  - `executable`, `args`, `cwd`, literal `env`, `envFrom`, optional `stdin`, `timeout`.
  - No shell and no string interpolation.
  - Process-group creation and termination.
  - Bounded stdout and stderr.
  - Exit code and stable error codes.
- Implement `shell`:
  - Explicit `shell` and `script`.
  - Disabled by default.
  - Allowed shells enforced by worker configuration.
- Enforce `allowed_roots` for `cwd`.
- Resolve `envFrom` from the worker process environment without persisting values.
- Add tests with helper processes for success, non-zero exit, timeout, output overflow,
  missing executable, disallowed cwd, and cancellation.
- Add platform-specific process-group handling behind build tags.

## Out of scope

- The step runner and continuity profiles.
- Git, OpenCode, Docker, or container executors.
- Resource limits such as CPU/memory cgroups.
- PTY allocation.

## Acceptance criteria

- [ ] Arguments reach the process verbatim, including spaces, quotes, and glob
      characters.
- [ ] The process executor never invokes a shell implicitly.
- [ ] A non-zero exit returns `STEP_EXIT_NONZERO` with the exit code.
- [ ] A timeout returns `STEP_TIMEOUT` and terminates the entire process group.
- [ ] Cancellation terminates the process group and returns `STEP_CANCELLED`.
- [ ] Output beyond the limit is truncated with an explicit marker.
- [ ] A `cwd` outside `allowed_roots` returns `CWD_NOT_ALLOWED`.
- [ ] The shell executor is refused unless explicitly enabled.
- [ ] Environment values are not included in the result or logs.

## Dependencies

- Blocked by: #3, #5.
- Blocks: step runner, Git/OpenCode adapters, MVP validation.

## Technical notes

- Use `context.Context` for cancellation and a derived deadline for timeouts.
- Create a new process group and kill the group on timeout/cancel. A direct child kill
  can leave grandchildren running.
- On Windows, implement an equivalent job-object or process-tree strategy behind a build
  tag; do not silently degrade.
- Capture stdout and stderr concurrently to avoid pipe deadlocks.
- Bound `stdin` and reject unbounded streams.
- Do not use `sh -c` as a convenience wrapper.
- `envFrom` is a list of names, not values. The worker resolves them at execution time.

## Validation

```console
$ go test ./internal/executor/...
$ go test ./... -run Executor
$ go test -race ./internal/executor/...
$ CGO_ENABLED=0 go build ./...
```
