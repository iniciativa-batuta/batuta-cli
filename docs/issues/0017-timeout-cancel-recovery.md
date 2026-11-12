# Implement timeout, cancellation, restart, and offline recovery semantics

## Context

The happy path is not enough. Batuta must behave predictably when a step hangs, a user
cancels, a worker restarts, or a worker disappears. These paths need explicit codes and
idempotent handling, not ad hoc cleanup.

## Objective

Implement timeouts, cancellation propagation, worker restart reconciliation, and
offline/lost finalization.

## Scope

- Add `POST /v1/executions/{id}/cancel` and `batuta task cancel`.
- Record `cancel_requested_at` and deliver `execution.cancel` through the channel.
- Enforce:
  - Step timeout: `STEP_TIMEOUT`.
  - Execution timeout: cancel running step, skip unstarted with `EXECUTION_TIMEOUT`,
    final result `FAILED`.
  - Assignment deadline: `LOST` if the worker never starts.
- Implement worker restart reconciliation for `RUNNING` steps with `WORKER_RESTARTED`.
- Implement offline handling:
  - Mark observations stale when the worker is offline.
  - Finalize `LOST` after the configured offline execution timeout.
  - Reject late snapshots for terminal executions.
- Implement `cancel_timeout`: finalize as `CANCELLED` with `cancel_confirmed=false` when
  the worker cannot be reached.
- Add tests for duplicate cancellation, cancellation during a step, cancellation before
  start, timeout during a step, restart mid-step, and worker offline.

## Out of scope

- Automatic retry policy.
- High availability of the central service.
- Preemption by higher-priority tasks.
- Process checkpointing or reattachment.

## Acceptance criteria

- [ ] Cancelling an assigned execution before start yields `CANCELLED`.
- [ ] Cancelling a running execution terminates the process group and records
      `STEP_CANCELLED` for the active step.
- [ ] Duplicate cancellation requests are idempotent.
- [ ] A step timeout produces `STEP_TIMEOUT` and applies the continuity policy.
- [ ] An execution timeout produces `EXECUTION_TIMEOUT` and skips unstarted steps.
- [ ] A worker restart marks interrupted steps `FAILED` with `WORKER_RESTARTED`.
- [ ] Offline executions become stale before becoming `LOST`.
- [ ] A snapshot for a `LOST` or `CANCELLED` execution is rejected with
      `EXECUTION_TERMINAL`.
- [ ] No late snapshot can revive a terminal execution.

## Dependencies

- Blocked by: #14, #16.
- Blocks: MVP validation and operator runbook.

## Technical notes

- Cancellation is an asynchronous request; the central service records intent before
  the worker confirms.
- Use a process group to terminate the whole tree.
- `cancel_confirmed` must be visible in the API and CLI so operators know whether the
  worker actually acknowledged.
- Define all grace periods as configuration with documented defaults.
- Keep restart reconciliation in the worker, not in the central service.
- Do not use `LOST` as a synonym for `FAILED`; it describes an observation/assignment
  loss, not a step error.

## Validation

```console
$ go test ./internal/worker/... ./internal/app/... ./internal/httpapi/...
$ go test ./... -run 'Cancel|Timeout|Restart|Offline'
$ go test -race ./internal/worker/...
```
