# Implement the scheduler with capabilities, priority, and dependencies

## Context

The central service must choose a compatible worker for each task, respect capacity and
priority, and explain why a task is blocked. The scheduler is a central concern; workers
never select their own work.

## Objective

Implement deterministic scheduling from task queue to worker assignment.

## Scope

- Add a task queue query for `PENDING` tasks whose dependencies are satisfied.
- Implement eligibility checks:
  - OS, architecture, executor types, and labels.
  - Worker state `ONLINE` and not `DRAINING`.
  - Available concurrency.
  - Revoked or disabled workers excluded.
- Order by priority (`critical`, `high`, `normal`, `low`) and then FIFO by creation time.
- Create an execution and assignment transactionally.
- Persist and expose `blocked_reason` for tasks whose dependencies are not satisfied.
- Mark the task `SCHEDULED` and the execution `ASSIGNED`.
- Retry assignment when a worker goes offline before starting.
- Add deterministic tests with multiple workers and tasks.

## Out of scope

- Preemption.
- Fair share across projects.
- Parallel steps inside an execution.
- Affinity beyond exact label matching.
- Automatic retry after a failure.

## Acceptance criteria

- [ ] A task is assigned only to a worker that satisfies all requirements.
- [ ] Higher-priority tasks are selected first, with FIFO within a priority.
- [ ] A worker at `max_concurrency` receives no new assignment.
- [ ] `DRAINING`, `OFFLINE`, and `REVOKED` workers are excluded.
- [ ] A task with an unsatisfied dependency is not assigned and exposes
      `blocked_reason`.
- [ ] A task with a `PARTIAL`, `FAILED`, `CANCELLED`, or `LOST` dependency remains
      blocked by default.
- [ ] Assignment creation and task/execution state changes are one transaction.
- [ ] Scheduler tests are deterministic and do not depend on wall-clock sleeps.

## Dependencies

- Blocked by: #6, #9, #11, #14, #16.
- Blocks: end-to-end MVP validation.

## Technical notes

- Keep the eligibility rule in one function and test it directly.
- Use a database claim/lease pattern to avoid double assignment if the scheduler runs
  concurrently with itself. A single central process is not a reason to skip the
  transaction.
- `max_concurrency` is a worker-declared limit; the central service enforces it using its
  own assignment records.
- `blocked_reason` is computed and stored for display; it is not a separate task state.
- Do not add a broker. SQLite transactions are sufficient for the MVP.

## Validation

```console
$ go test ./internal/app/... ./internal/store/...
$ go test ./... -run Scheduler
$ go test -race ./internal/app/...
```
