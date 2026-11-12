# Add SQLite storage, core schema, and migrations

## Context

The central service needs durable state for projects, tasks, executions, workers,
observed snapshots, artifacts, enrollment tokens, idempotency keys, and audit events.
The worker needs its own local database for execution plans and step observations.

## Objective

Provide a shared SQLite foundation with safe connection settings, an embedded migration
runner, and the initial schema.

## Scope

- Create `internal/store` with a connection opener that enables:
  - WAL journal mode.
  - Foreign keys.
  - A configurable busy timeout.
  - A single-writer-friendly pool configuration.
- Add embedded migrations with a schema version table.
- Add the initial tables:
  - `projects`, `tasks`, `task_dependencies`.
  - `executions`, `execution_steps`, `execution_snapshots`.
  - `workers`, `worker_credentials`, `enrollment_tokens`.
  - `artifacts`, `audit_events`, `idempotency_keys`.
- Add a transaction helper and migration status command.
- Add an explicit backup/checkpoint note for operators.
- Add tests against temporary databases.
- Document the driver choice and its CGO implications in
  `docs/architecture/decisions.md`.

## Out of scope

- Repository methods and domain mapping.
- PostgreSQL support.
- Automatic backups or replication.
- Query optimization beyond required indexes.

## Acceptance criteria

- [ ] A fresh database migrates from zero to the current version.
- [ ] Re-running migrations is a no-op.
- [ ] Foreign keys are enforced and tested.
- [ ] WAL mode is active in tests.
- [ ] Migration failure rolls back cleanly and leaves the previous version intact.
- [ ] No raw SQL leaks into the domain package.
- [ ] The driver builds with `CGO_ENABLED=0`.

## Dependencies

- Blocked by: #1.
- Blocks: repositories, worker local state, audit trail.

## Technical notes

- Prefer a pure-Go SQLite driver to keep cross-compilation simple. Record the trade-off
  if a CGO driver is chosen instead.
- Store timestamps as UTC RFC 3339 text or integer microseconds consistently; do not mix
  formats.
- Store structured payloads as validated JSON text, with size limits enforced before
  insertion.
- `execution_steps` stores the immutable plan. `execution_snapshots` stores observed
  state separately; do not merge them into one mutable step row.
- Add indexes for task state, execution state, worker heartbeat, and snapshot sequence.
- Migrations must be embedded so a single binary can initialize an empty directory.

## Validation

```console
$ go test ./internal/store/...
$ go test ./... -run Migration
$ CGO_ENABLED=0 go build ./...
```
