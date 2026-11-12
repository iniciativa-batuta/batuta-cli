# Implement snapshot reconciliation and status views

## Context

The central service must observe worker state without pretending to own it. Users need a
status view that shows step states, observation time, and staleness.

## Objective

Implement the central snapshot receiver, reconciliation loop, observation storage, and
CLI status/watch commands.

## Scope

- Add `POST /v1/worker/snapshots` with request correlation and idempotency.
- Validate snapshots against the assigned plan, protocol version, worker scope, sequence,
  and timestamp skew.
- Store `observed_at`, `received_at`, `sequence`, and the snapshot payload.
- Add a reconciler that requests snapshots for `ASSIGNED` and `RUNNING` executions with
  configurable intervals and backoff.
- Add `GET /v1/executions/{id}/snapshot`.
- Add stale detection based on `snapshot_stale_after`.
- Finalize execution and task state from terminal snapshots.
- Add CLI commands:
  - `batuta task show`
  - `batuta task watch`
  - `batuta execution show`
  - `batuta execution steps`
- Add tests for duplicate responses, stale sequences, request expiry, stale marking,
  and terminal finalization.

## Out of scope

- Cancellation timeouts, worker-offline finalization, and `LOST` policy.
- Artifact and log retrieval.
- Web UI.
- Direct pull optimization.

## Acceptance criteria

- [ ] A snapshot is accepted only for a pending request, the correct worker, and the
      correct execution.
- [ ] A repeated identical snapshot response is idempotent.
- [ ] A different body with the same `request_id` returns a conflict.
- [ ] An older sequence is rejected with `SNAPSHOT_SEQUENCE_STALE`.
- [ ] The stored observation is marked stale after the configured threshold.
- [ ] Repeating a query does not mutate state.
- [ ] A terminal snapshot finalizes the execution and task exactly once.
- [ ] CLI output shows `OBSERVED` and `STALE`, and never shows a percentage.
- [ ] `task watch` polls with backoff and exits cleanly on terminal state.

## Dependencies

- Blocked by: #8, #10, #13.
- Blocks: timeout/cancellation handling, MVP validation, web status views.

## Technical notes

- Do not overwrite the execution plan with snapshot data.
- The central service may accept a snapshot with an execution state that is behind the
  worker's latest state; the sequence rule decides which observation wins.
- Validate that every returned step key exists in the plan. Missing steps are allowed
  only if the snapshot is structurally valid and the plan expects them; represent them
  as `PENDING` in the view, not as unknown.
- Keep finalization idempotent and transactional.
- `watch` is a polling client, not a streaming protocol.

## Validation

```console
$ go test ./internal/httpapi/... ./internal/app/... ./internal/cli/...
$ go test ./... -run Reconcile
$ go test -race ./internal/app/...
```
