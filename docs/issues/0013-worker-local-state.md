# Implement worker-local step state and snapshot generation

## Context

The worker is the owner of runtime step state. It must persist that state across
restarts and expose a coherent snapshot without asking the central service for
permission or validation.

## Objective

Implement the worker-local execution store and snapshot provider, including sequence
numbers, restart reconciliation, and the optional direct snapshot endpoint.

## Scope

- Add a worker-local SQLite schema for:
  - Received execution plans.
  - Step observations.
  - Snapshot sequence counters.
  - Process metadata needed for cancellation and restart reconciliation.
- Persist the plan before starting any step.
- Apply the legal step transitions from the domain package.
- Implement snapshot generation with:
  - `schema`, `worker_id`, `execution_id`, `plan_version`.
  - Monotonic `sequence` per execution.
  - `observed_at` in UTC.
  - Execution state and all step observations.
  - Bounded output references, never raw environment values.
- Add `GET /v1/local/executions/{id}/snapshot` on the worker with credential auth.
- Add `POST /v1/worker/snapshots` delivery to the central service when responding to a
  `snapshot.request`.
- Add restart reconciliation tests: a running step becomes `FAILED` with
  `WORKER_RESTARTED`.
- Use a stub executor in tests; real process execution is a separate issue.

## Out of scope

- The central reconciler loop.
- Real process execution.
- Continuity profiles and cancellation behavior.
- Snapshot retention policy beyond the current execution.

## Acceptance criteria

- [ ] The plan is persisted before the first step starts.
- [ ] Step observations survive a worker restart.
- [ ] Snapshot sequence is monotonic per execution.
- [ ] A stale sequence is rejected by the central receiver.
- [ ] A snapshot never includes environment values or credentials.
- [ ] The direct endpoint requires the worker credential and rejects cross-worker
      access.
- [ ] A restarted worker reports `WORKER_RESTARTED` for steps that were running.
- [ ] Snapshots validate against the shared schema fixtures.

## Dependencies

- Blocked by: #4, #5, #12.
- Blocks: central reconciliation, execution status views, cancellation.

## Technical notes

- The worker may share the SQLite driver with the central service but must use a
  separate database file and schema.
- Do not send a snapshot on every step transition. Generate one when requested or when
  responding to a control request.
- Keep `observed_at` and `sequence` generation in one place to avoid inconsistent
  snapshots.
- The direct endpoint is opt-in and must not become the only supported path.
- On restart, do not attempt to reattach to a process. Mark the step failed and let the
  policy decide.

## Validation

```console
$ go test ./internal/worker/...
$ go test ./... -run Snapshot
$ go test -race ./internal/worker/...
```
