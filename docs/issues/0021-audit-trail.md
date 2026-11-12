# Add an append-only audit trail for sensitive operations

## Context

Creating tasks, cancelling executions, enrolling workers, rotating credentials, and
disabling workers are security-relevant actions. Operators need a durable record of who
did what, without turning logs into the source of truth.

## Objective

Record and expose audit events for sensitive operations.

## Scope

- Emit audit events for:
  - Project and task creation.
  - Task/execution cancellation.
  - Worker enrollment, disable, revoke, and credential rotation.
  - Artifact access, when sensitive.
  - Authentication failures at a bounded rate.
- Store actor type, actor ID, action, target, request ID, timestamp, and non-sensitive
  metadata.
- Add `GET /v1/audit-events` with filters by actor, action, target, and time range.
- Add `batuta audit list`.
- Ensure events are append-only at the application level. No update or delete endpoint.
- Add tests for event emission, filtering, pagination, and secret exclusion.

## Out of scope

- SIEM integration.
- Tamper-proof ledgers or cryptographic chaining.
- Long-term retention automation.
- A general-purpose event store for step progress.

## Acceptance criteria

- [ ] Every sensitive operation emits exactly one audit event per successful action.
- [ ] Failed authorization attempts are recorded without logging the credential.
- [ ] Events include a request ID that can be correlated with logs.
- [ ] Audit metadata never contains tokens, passwords, environment values, or raw
      request bodies.
- [ ] The API supports bounded pagination and time-range filters.
- [ ] There is no endpoint to update or delete an audit event.
- [ ] Audit writes do not make the primary operation fail after it has committed; the
      consistency rule is documented.

## Dependencies

- Blocked by: #8, #9, #20.
- Blocks: MVP validation, operational runbook.

## Technical notes

- Prefer writing the audit event in the same transaction when the operation is local and
  the event is essential. If that is not possible, document the gap and use an
  outbox-style retry.
- Use stable action names such as `task.cancel` and `worker.credential.rotate`.
- Do not store full request bodies. Store identifiers and bounded metadata.
- Audit events are not execution state. Step history comes from snapshots.

## Validation

```console
$ go test ./internal/app/... ./internal/httpapi/...
$ go test ./... -run Audit
```
