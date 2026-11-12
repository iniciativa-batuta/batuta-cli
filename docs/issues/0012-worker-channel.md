# Implement the worker work acquisition and control channel

## Context

Workers behind NAT cannot accept inbound connections. The protocol must deliver work and
control requests over a worker-initiated long poll, while keeping snapshot observation
request-driven.

## Objective

Implement the long-poll channel that delivers assignments and control requests, with
idempotent delivery, cursors, deadlines, and retry behavior.

## Scope

- Add `POST /v1/worker/channel` with `wait_seconds`, `cursor`, and worker identity.
- Add a pending control-request queue in the central store.
- Deliver:
  - Assignments (using a test assignment source until the scheduler exists).
  - `snapshot.request` control requests.
  - `execution.cancel` control requests.
- Add request TTL, delivery attempts, and a cursor that prevents immediate redelivery.
- Add worker-side channel loop with exponential backoff and jitter.
- Deduplicate assignments by `assignment_id`.
- Add tests for long-poll timeout, duplicate delivery, cursor progression, request
  expiry, and NAT-only connectivity (worker makes outbound calls only).

## Out of scope

- Creating real assignments from the scheduler.
- Executing steps.
- Snapshot payload generation.
- Direct worker pull mode.

## Acceptance criteria

- [ ] An idle channel returns `type: "none"` after `wait_seconds`, not a tight loop.
- [ ] An assignment is delivered once; a duplicate is ignored by the worker.
- [ ] A control request is delivered at least once and expires after its deadline.
- [ ] Cursors advance and prevent immediate redelivery of the same request.
- [ ] The worker reconnects with backoff after network errors.
- [ ] A worker cannot poll on behalf of another worker.
- [ ] A revoked worker is rejected.
- [ ] The channel path works with the worker making only outbound connections.

## Dependencies

- Blocked by: #2, #9, #11.
- Blocks: worker step execution, snapshot reconciliation, cancellation.

## Technical notes

- Store control requests with `request_id`, `worker_id`, `execution_id`, action,
  deadline, delivery attempts, and completion state.
- The cursor is opaque to the worker. It may be a ULID or a sequence, but do not expose
  database offsets as a contract.
- Assignment delivery is at-least-once. The worker must persist and deduplicate before
  starting anything.
- Long polling must respect server shutdown and client disconnects.
- Do not implement a WebSocket or SSE path in the MVP.
- An empty channel response is normal; log it at debug level only.

## Validation

```console
$ go test ./internal/httpapi/... ./internal/worker/...
$ go test ./... -run Channel
$ go test -race ./internal/worker/...
```
