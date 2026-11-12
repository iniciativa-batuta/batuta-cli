# Implement the step runner and continuity profiles

## Context

Once the worker can execute one process, it needs to run an execution plan in the right
order, apply the task's continuity policy, and record every skip and failure with an
explicit reason.

## Objective

Implement the sequential step runner with dependency gating and the three continuity
profiles.

## Scope

- Add a runner that:
  - Loads the persisted execution plan.
  - Executes one ready step at a time.
  - Honors `dependsOn`.
  - Updates the worker-local step state after each transition.
  - Produces the final execution result.
- Implement:
  - `FAIL_FAST`: stop on first failure, skip remaining with `PREVIOUS_STEP_FAILED`,
    final result `FAILED`.
  - `CONTINUE_ON_ERROR`: continue in declared order while dependencies allow, skip
    blocked steps with `DEPENDENCY_FAILED`, aggregate `SUCCEEDED`/`PARTIAL`/`FAILED`.
  - `BEST_EFFORT`: choose any remaining ready step, maximize attempts, aggregate with
    the same mapping as `CONTINUE_ON_ERROR`.
- Add table-driven tests covering success, failure, dependency failure, all-failed, and
  partial success for each profile.
- Ensure snapshots reflect the state at the time they are requested.

## Out of scope

- Parallel step execution.
- Dynamic step creation.
- Automatic retries.
- Timeout/cancellation policy details (separate issue).
- Scheduler behavior.

## Acceptance criteria

- [ ] Steps run only when all dependencies are `COMPLETED`.
- [ ] A failed dependency causes `SKIPPED` with `DEPENDENCY_FAILED`.
- [ ] `FAIL_FAST` records `PREVIOUS_STEP_FAILED` for unstarted steps and ends `FAILED`.
- [ ] `CONTINUE_ON_ERROR` can end `PARTIAL` and never reports partial success as
      success.
- [ ] `BEST_EFFORT` attempts independent ready steps after a failure.
- [ ] Every `SKIPPED` step has exactly one reason.
- [ ] The final execution result is computed from step observations, not from an
      optimistic assumption.
- [ ] No progress percentage is produced.

## Dependencies

- Blocked by: #5, #13, #15.
- Blocks: timeout/cancellation, scheduler integration, MVP validation.

## Technical notes

- The runner is a loop over ready steps; do not build a general workflow engine.
- For `BEST_EFFORT`, selection must be deterministic when multiple steps are ready:
  declared position is a stable tie-breaker.
- Persist the transition before starting the next step so a crash cannot lose the fact
  that a step completed.
- On failure, capture exit code, error code, and output reference before applying policy.
- `PARTIAL` is terminal and blocks dependent tasks by default.

## Validation

```console
$ go test ./internal/worker/...
$ go test ./... -run Continuity
$ go test -race ./internal/worker/...
```
