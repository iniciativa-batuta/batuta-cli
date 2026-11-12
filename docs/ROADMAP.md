# Roadmap

This roadmap is ordered by real dependencies, not by feature appeal. Every phase has an
exit criterion that can be demonstrated. Dates are intentionally absent.

## Phase 0 — Documentation and decisions (current)

**Deliverables**

- Product vision, architecture, domain model, protocol, execution semantics, and
  decision records.
- A reviewable work breakdown in [docs/issues](issues/README.md), grouped into the
  `MVP` (phases 1–8) and `Post-MVP` (phase 9) milestones.
- Organization-wide standards, templates, and repository baseline.

**Exit criteria**

- The MVP scope is explicit.
- Ownership of Task, Execution, and Step state is unambiguous.
- The worker observation protocol has a concrete recommendation and documented limits.
- No issue depends on undefined behavior.

## Phase 1 — Foundation

**Deliverables**

- Go module, binary skeleton, package layout, build and test targets.
- Versioned shared contracts, identifiers, and error model.
- Configuration loading and validation.
- SQLite connection, WAL settings, and an embedded migration runner.

**Exit criteria**

- `go build ./...`, `go test ./...`, and `go vet ./...` pass from a clean checkout.
- `batuta version` prints version, commit, and build date.
- Server and worker example configurations parse and validate.
- A fresh database migrates from zero and re-running migrations is a no-op.

**Risks**

- Choosing a SQLite driver that complicates cross-compilation.
- Over-designing the package layout before a second use case exists.

## Phase 2 — Domain and persistence

**Deliverables**

- Project, Task, Execution, Worker, Step definition/observation, Artifact, and audit
  entities.
- Legal state machines with explicit transition errors.
- Repositories with transactional updates and optimistic concurrency.
- Tests for illegal transitions and concurrent updates.

**Exit criteria**

- The domain package imports no transport, storage, or OS packages.
- Every legal transition has a test; every illegal transition is rejected atomically.
- Terminal states are immutable.
- The central service can persist a task and its immutable execution plan.

## Phase 3 — Central API and CLI

**Deliverables**

- HTTP API foundation: routing, request IDs, consistent errors, auth middleware, body
  limits, graceful shutdown.
- Project and task endpoints, including idempotent task application.
- Worker registry and enrollment endpoints.
- CLI client foundation with project/task commands and `--output json`.

**Exit criteria**

- A project and task can be created and inspected entirely through the CLI.
- Re-applying an identical task definition is idempotent.
- Invalid task definitions return field-level validation errors.
- Every endpoint rejects unauthenticated requests when auth is enabled.

## Phase 4 — Worker runtime and snapshots

**Deliverables**

- Worker enrollment, credential storage, rotation/revocation, and liveness heartbeat.
- Long-poll channel for work assignment and control requests.
- Worker-local step state, persisted across restarts.
- Snapshot provider and central reconciliation loop.
- CLI task/execution status views with observed timestamps and stale markers.

**Exit criteria**

- A worker running behind NAT can receive work using outbound connections only.
- Duplicate assignment or control delivery is handled idempotently.
- A repeated snapshot query does not change state.
- The central service stores the last observed snapshot without claiming it is current.
- `batuta task show` and `batuta task watch` show step states, not percentages.

## Phase 5 — Execution semantics

**Deliverables**

- Structured process executor and an explicit, disabled-by-default shell executor.
- Step runner with dependency gating.
- `FAIL_FAST`, `CONTINUE_ON_ERROR`, and `BEST_EFFORT` continuity profiles.
- Step/execution timeouts, cancellation, worker-restart reconciliation, and offline
  handling.

**Exit criteria**

- Arguments are passed verbatim; no shell interpolation occurs in the process executor.
- A timed-out process group is terminated and the step records `STEP_TIMEOUT`.
- Unstarted steps are `SKIPPED` with an explicit reason.
- Each continuity profile has table-driven tests covering dependency failures.
- Partial results are never reported as success.

## Phase 6 — Scheduling

**Deliverables**

- Eligibility by OS, architecture, labels, executors, and worker capacity.
- Priority ordering and FIFO within the same priority.
- Task dependencies and explicit `blocked_reason`.
- Assignment retry when a worker becomes unavailable before starting.

**Exit criteria**

- An incompatible worker is never selected.
- A blocked task explains why it is blocked.
- Concurrency limits are respected.
- Scheduler tests are deterministic.

## Phase 7 — Payloads, artifacts, and security hardening

**Deliverables**

- Versioned inline payload envelope with size limits.
- Artifact upload/download by reference, with checksums and quotas.
- TLS requirements, credential rotation, scoped worker credentials, and secret
  redaction.
- Append-only audit trail for sensitive operations.

**Exit criteria**

- Oversized inline payloads are rejected with a clear error.
- Artifact checksums are verified on upload and download.
- A worker cannot read or mutate another worker's executions.
- Secrets never appear in logs, errors, task definitions, or snapshots.
- Sensitive operations are queryable through the audit API/CLI.

## Phase 8 — MVP validation

**Deliverables**

- End-to-end test harness covering local and remote workers.
- Demonstration of `FAIL_FAST` and `CONTINUE_ON_ERROR`.
- Recovery tests: worker restart, worker offline, duplicate snapshot query, cancellation.
- Runbook and recorded CLI session.

**Exit criteria**

- The MVP checklist in [the architecture overview](architecture/overview.md) is fully
  demonstrated.
- The end-to-end test runs in CI.
- Known limitations are documented, including NAT behavior and cancellation while a
  worker is offline.

## Phase 9 — Optional integrations and interface

**Deliverables**

- Optional Git workspace/integration adapter.
- Optional OpenCode executor adapter.
- Optional web interface consuming the same public API.

**Exit criteria**

- Integrations are disabled by default and policy-gated.
- The web interface has no direct database, shell, or worker access.
- Disabling every optional component still leaves a complete CLI workflow.
