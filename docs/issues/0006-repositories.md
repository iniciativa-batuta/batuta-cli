# Implement repositories and transition persistence tests

## Context

The domain model is only useful if transitions are persisted atomically and concurrent
updates cannot silently overwrite each other. The repository layer also needs to keep
the execution plan separate from observed snapshots.

## Objective

Implement repositories for the aggregate roots, snapshot storage, and audit events, with
transactional state changes and optimistic concurrency.

## Scope

- Add repositories for Projects, Tasks, Executions, Workers, credentials, artifacts,
  snapshots, and audit events.
- Add a `version` column to mutable aggregate tables and enforce compare-and-swap
  updates.
- Persist execution plans immutably and snapshot observations separately.
- Add dependency queries for task eligibility and execution lookup.
- Add transactional use-case helpers for create-task, assign-execution, observe-snapshot,
  and cancel-execution.
- Add tests that run against a real temporary SQLite database.

## Out of scope

- HTTP handlers and CLI commands.
- Scheduler selection.
- Snapshot reconciliation loops.
- Artifact binary storage.

## Acceptance criteria

- [ ] CRUD works for every aggregate root.
- [ ] An illegal transition is rejected without writing partial state.
- [ ] Two concurrent updates based on the same version cannot both succeed.
- [ ] Storing a snapshot does not mutate the execution plan.
- [ ] Snapshot sequences are enforced: an older sequence is rejected.
- [ ] Task dependency queries return the correct blocked reason.
- [ ] Repository errors are wrapped with stable codes and never leak SQL strings to the
      API.
- [ ] Tests cover rollback, unique constraints, and foreign-key failures.

## Dependencies

- Blocked by: #4, #5.
- Blocks: API use cases, scheduler, worker snapshot handling.

## Technical notes

- Do not return database rows as domain entities without mapping and validation.
- Use explicit transactions for multi-row changes.
- Keep the plan and the observed snapshot in separate tables. The central service must
  never overwrite the plan with observed data.
- Store `observed_at`, `received_at`, and `sequence` for snapshots.
- Prefer small repository interfaces defined by the consuming use case, not one giant
  store interface.
- Do not create a generic ORM abstraction.

## Validation

```console
$ go test ./internal/store/...
$ go test ./... -run Repository
$ go test -race ./internal/store/...
```
