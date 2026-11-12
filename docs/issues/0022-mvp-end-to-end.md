# Validate the MVP end to end with local and remote workers

## Context

Individual features can pass their tests while the product still fails the core promise:
create a task, assign it to a worker, observe step state, and recover from real failures.

## Objective

Build an end-to-end validation harness and runbook that proves the MVP checklist in
[the architecture overview](https://github.com/iniciativa-batuta/batuta-cli/blob/main/docs/architecture/overview.md#mvp-checklist).

## Scope

- Add an end-to-end test that:
  - Starts the central service with a temporary database.
  - Creates a project and applies a task.
  - Starts a worker in a separate process.
  - Verifies assignment, execution, snapshots, and final task state.
  - Runs a local worker and a worker that only makes outbound connections.
- Cover:
  - `FAIL_FAST` and `CONTINUE_ON_ERROR`.
  - Worker restart mid-execution.
  - Worker offline, stale observation, and `LOST` after the grace period.
  - Duplicate snapshot query.
  - Cancellation during a running step.
  - At least one artifact upload/download.
- Add a runbook with exact commands, expected output, cleanup, and known limitations.
- Add the end-to-end test to the CI workflow when CI exists.
- Record a CLI session and link it from the README.

## Out of scope

- Web UI.
- Git and OpenCode integrations.
- High availability, multi-node central service, or performance benchmarking.
- Production deployment automation.

## Acceptance criteria

- [ ] The harness starts and stops every process reliably, including on failure.
- [ ] The local-worker scenario passes.
- [ ] The outbound-only worker scenario passes.
- [ ] `FAIL_FAST` produces `FAILED` with skipped steps.
- [ ] `CONTINUE_ON_ERROR` produces the documented `PARTIAL` or `FAILED` result.
- [ ] A worker restart marks the interrupted step `WORKER_RESTARTED`.
- [ ] Offline handling shows stale before `LOST`.
- [ ] A repeated snapshot query does not mutate state.
- [ ] Cancellation stops the process group and finalizes `CANCELLED`.
- [ ] Artifact checksum verification passes.
- [ ] The runbook can be followed by someone who did not write the code.

## Dependencies

- Blocked by: #14, #16, #17, #18, #19, #20, #21.
- Blocks: declaring the MVP complete.

## Technical notes

- Prefer the built binary over in-process handlers for the end-to-end test. The point is
  to test the real process boundary.
- Use random free ports and temporary directories; do not depend on fixed paths.
- Simulate "remote" by running the worker in a separate process and asserting it opens no
  inbound listener required by the central service. Do not require root or network
  namespaces.
- Bound every wait with a timeout and collect logs on failure.
- Do not hide flaky behavior with long sleeps.

## Validation

```console
$ go test ./test/e2e/... -v
$ go test -race ./...
$ make e2e
```
