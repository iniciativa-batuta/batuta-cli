# Implement the domain model and state machines

## Context

State ownership is the core correctness property of Batuta: the central service owns
Task and Execution state; the worker owns runtime Step state. These rules must exist as
pure domain code before storage or HTTP can depend on them.

## Objective

Implement the domain entities, identifiers, invariants, and legal state transitions for
Projects, Tasks, Executions, Workers, Steps, and audit events.

## Scope

- Create `internal/domain` with:
  - Project, Task, Execution, Worker, WorkerCredential, Artifact, AuditEvent.
  - ExecutionPlan, StepDefinition, StepObservation value objects.
  - State enums for Task, Execution, Worker, and Step.
  - Transition functions that return typed domain errors.
  - Result aggregation for the three continuity profiles.
  - Skip reasons and error codes as typed constants.
- Enforce the invariants listed in `docs/architecture/domain-model.md`.
- Add table-driven tests for every legal and illegal transition.
- Add tests for plan validation: duplicate keys, missing dependencies, cycles, empty
  plans.

## Out of scope

- SQL, HTTP, CLI, worker process execution.
- Scheduler selection logic.
- Retry policy beyond "a retry is a new execution".
- Any import of `internal/store`, `internal/httpapi`, or `internal/executor`.

## Acceptance criteria

- [ ] The domain package imports only the standard library and `internal/protocol` if
      needed for shared value types.
- [ ] Every legal transition has a test.
- [ ] Every illegal transition returns a typed error and does not mutate state.
- [ ] Terminal Task, Execution, and Step states are immutable.
- [ ] `SKIPPED` requires a reason; `FAILED` requires an error code and finish time.
- [ ] Plan validation rejects duplicates, unknown dependencies, and cycles.
- [ ] Result aggregation is tested for `FAIL_FAST`, `CONTINUE_ON_ERROR`, and
      `BEST_EFFORT`.
- [ ] No progress percentage exists in the domain.

## Dependencies

- Blocked by: #1, #2.
- Blocks: repositories, API use cases, worker step runner.

## Technical notes

- Pass timestamps into transition functions instead of calling `time.Now()` inside them,
  so tests are deterministic.
- Keep state transitions as methods or functions on the entity, not as free-form string
  comparisons in handlers.
- Do not add a generic event bus or a second history model.
- `PARTIAL` means partial success and is never treated as success by dependency rules.
- The worker's step transitions are part of this package? Keep the transition rules in a
  shared domain package and let the worker call them; the worker still owns persistence
  and execution. Do not let the central service call step transitions.

## Validation

```console
$ go test ./internal/domain/...
$ go test ./... -run Transition
$ go vet ./...
```
