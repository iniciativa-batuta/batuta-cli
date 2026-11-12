# Domain model

The domain is intentionally small. Each entity exists because it has its own identity,
lifecycle, or persistence requirement. Payloads, step definitions, and observations are
value objects or owned records, not independent aggregates.

## Aggregates and entities

### Project

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `prj_...` | Prefixed ULID. |
| `name` | string | Unique, URL-safe slug, immutable after creation. |
| `description` | string | Optional. |
| `created_at` | timestamp | UTC. |
| `updated_at` | timestamp | UTC. |

A Project is a namespace. It does not own execution state and must not accumulate
duplicated task data.

### Task

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `tsk_...` | Prefixed ULID. |
| `project_id` | `prj_...` | Immutable. |
| `name` | string | Unique inside the project. |
| `spec_version` | string | Format version, for example `batuta/v1`. |
| `spec` | normalized JSON | Immutable after submission; the YAML input is not stored as text. |
| `spec_hash` | SHA-256 hex | Used for idempotent application. |
| `priority` | enum | `low`, `normal`, `high`, `critical`. |
| `continuity` | enum | `FAIL_FAST`, `CONTINUE_ON_ERROR`, `BEST_EFFORT`. |
| `state` | enum | See [Task states](#task-states). |
| `blocked_reason` | string or null | Computed when a dependency is not satisfied. |
| `created_at` | timestamp | UTC. |
| `updated_at` | timestamp | UTC. |

Tasks are definitions plus current state. They are not execution logs.

### Execution

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `exe_...` | Prefixed ULID. |
| `task_id` | `tsk_...` | Immutable. |
| `worker_id` | `wkr_...` or null | Assigned worker. |
| `plan_version` | integer | Increments only when a new execution plan is created. |
| `plan_hash` | SHA-256 hex | Hash of the immutable execution plan. |
| `state` | enum | See [Execution states](#execution-states). |
| `assigned_at` | timestamp or null | When the assignment was created. |
| `started_at` | timestamp or null | First observation of `RUNNING`, or worker-reported start. |
| `finished_at` | timestamp or null | Terminal state time. |
| `cancel_requested_at` | timestamp or null | Set before the worker confirms. |
| `result` | enum or null | `SUCCEEDED`, `PARTIAL`, `FAILED`, `CANCELLED`, `LOST`. |
| `error_code` | string or null | Stable code, for example `EXECUTION_TIMEOUT`. |
| `error_message` | string or null | Human-readable summary; no secrets. |

An Execution is an attempt. A retry is a new Execution with the same task and a new
plan/version if needed. Executions are never resurrected from a terminal state.

### Execution plan

The plan is immutable for the lifetime of an execution. It contains:

- The ordered list of step definitions.
- Each step's key, name, dependencies, executor specification, timeout, and output
  limits.
- The continuity policy copied from the task at assignment time.
- The execution deadline.

The central service stores the plan. The worker receives it, validates it, and executes
it. The worker cannot add, remove, or reorder steps.

### Step definition

| Field | Type | Notes |
| --- | --- | --- |
| `key` | string | Unique inside the execution, stable, immutable. |
| `position` | integer | Declared order. |
| `name` | string | Human-readable. |
| `depends_on` | string[] | Step keys that must complete successfully first. |
| `executor` | object | Versioned executor specification. |
| `timeout` | duration | Optional; defaults to the execution timeout. |
| `output` | object | Stdout/stderr capture limits. |

### Step observation

The worker's runtime state for a step. It is stored by the worker and reported in
snapshots. The central service stores the last observed copy, not as authoritative
runtime state.

| Field | Type | Notes |
| --- | --- | --- |
| `step_key` | string | References the step definition. |
| `state` | enum | See [Step states](#step-states). |
| `attempt` | integer | `1` in the MVP; retries create a new execution. |
| `started_at` | timestamp or null | UTC. |
| `finished_at` | timestamp or null | UTC. |
| `exit_code` | integer or null | Process exit code when applicable. |
| `error_code` | string or null | Stable code. |
| `error_message` | string or null | Redacted. |
| `skip_reason` | string or null | Required when `SKIPPED`. |
| `output_ref` | string or null | Reference to bounded output storage. |

The domain must not invent a progress percentage. The step state list is the progress
model.

### Worker

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `wkr_...` | Prefixed ULID. |
| `name` | string | Human-readable, not unique. |
| `state` | enum | See [Worker states](#worker-states). |
| `capabilities` | object | OS, architecture, executors, labels, resource hints. |
| `max_concurrency` | integer | Defaults to `1` in the MVP. |
| `last_heartbeat_at` | timestamp or null | Liveness only. |
| `registered_at` | timestamp | UTC. |
| `revoked_at` | timestamp or null | Terminal. |

Capabilities are declared by the worker and validated by the central service. They are
not a security boundary by themselves; policy is enforced by the worker.

### Worker credential

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `wcr_...` | Prefixed ULID. |
| `worker_id` | `wkr_...` | Scope. |
| `token_hash` | string | Argon2id or equivalent; never the raw token. |
| `created_at` | timestamp | UTC. |
| `expires_at` | timestamp or null | UTC. |
| `revoked_at` | timestamp or null | UTC. |
| `last_used_at` | timestamp or null | UTC. |

### Artifact

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `art_...` | Prefixed ULID. |
| `execution_id` | `exe_...` | Owner. |
| `step_key` | string or null | Producing step, when known. |
| `name` | string | Sanitized display name. |
| `media_type` | string | Validated, not trusted for rendering. |
| `size` | integer | Bytes. |
| `sha256` | string | Verified on upload and download. |
| `uri` | string | `artifact://art_...`; never a raw filesystem path. |
| `created_at` | timestamp | UTC. |

### Audit event

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `aud_...` | Prefixed ULID. |
| `actor_type` | enum | `user`, `worker`, `system`. |
| `actor_id` | string | Token ID, worker ID, or `system`. |
| `action` | string | Stable verb, for example `task.cancel`. |
| `target_type` | string | Entity type. |
| `target_id` | string | Entity ID. |
| `request_id` | string or null | Correlation. |
| `metadata` | JSON | Non-sensitive structured context. |
| `created_at` | timestamp | UTC. |

Audit events are append-only. Do not use a general event log as a substitute for
structured execution state.

## State machines

### Task states

```text
PENDING ──► SCHEDULED ──► RUNNING ──► SUCCEEDED
   │             │            │
   │             │            ├─────► PARTIAL
   │             │            ├─────► FAILED
   │             │            └─────► CANCELLED
   │             └──────────────────► PENDING   (assignment abandoned before start)
   └────────────────────────────────► CANCELLED
```

- `PENDING`: created, not yet assigned. May carry a computed `blocked_reason`.
- `SCHEDULED`: an execution is assigned or about to be assigned.
- `RUNNING`: at least one execution has started.
- `SUCCEEDED`: all planned steps completed successfully.
- `PARTIAL`: at least one step did not succeed and the execution policy allowed others
  to complete.
- `FAILED`: the execution failed under its policy.
- `CANCELLED`: cancelled by an authorized actor.

Terminal states: `SUCCEEDED`, `PARTIAL`, `FAILED`, `CANCELLED`.

Task state is derived from the latest execution when a new execution starts. A task may
have more than one execution over its lifetime; the official task state always reflects
the latest accepted attempt.

### Execution states

```text
ASSIGNED ──► RUNNING ──► SUCCEEDED
    │           │
    │           ├──────► PARTIAL
    │           ├──────► FAILED
    │           ├──────► CANCELLED
    │           └──────► LOST
    ├──────────────────► CANCELLED
    └──────────────────► LOST
```

- `ASSIGNED`: worker selected and assignment created.
- `RUNNING`: first coherent observation shows execution started.
- `SUCCEEDED`: every step completed.
- `PARTIAL`: at least one step succeeded and at least one did not, under a policy that
  permits continuation.
- `FAILED`: the execution failed under its policy.
- `CANCELLED`: cancellation confirmed or finalized by policy.
- `LOST`: worker unavailable, assignment never started, or offline grace expired.

Terminal states: all except `ASSIGNED` and `RUNNING`.

### Worker states

```text
REGISTERED ──► ONLINE ──► DRAINING ──► OFFLINE
    │             │            │
    │             └────────────┴──────► REVOKED
    └─────────────────────────────────► REVOKED
```

- `REGISTERED`: credential issued, no heartbeat yet.
- `ONLINE`: heartbeat within the configured window.
- `DRAINING`: no new assignments; running executions continue.
- `OFFLINE`: heartbeat missed beyond the window.
- `REVOKED`: terminal; credentials invalid.

`ONLINE`/`OFFLINE` are liveness states, not step progress.

### Step states

```text
PENDING ──► RUNNING ──► COMPLETED
   │           │
   │           ├──────► FAILED
   │           └──────► CANCELLED
   ├──────────────────► SKIPPED
   └──────────────────► CANCELLED
```

Terminal states: `COMPLETED`, `FAILED`, `SKIPPED`, `CANCELLED`.

Transitions are owned by the worker. The central service validates that a reported
snapshot is structurally legal and within the assigned plan, but does not drive step
transitions.

## Invariants

1. A Task belongs to exactly one Project and cannot move between projects.
2. A Task's normalized spec and spec hash are immutable after submission.
3. An Execution belongs to exactly one Task and at most one Worker.
4. An execution plan's step keys are unique and every `depends_on` references a step in
   the same plan.
5. The plan is immutable after assignment.
6. A terminal Execution never transitions to a non-terminal state.
7. A terminal Step never transitions to a non-terminal state.
8. `SKIPPED` steps always have a `skip_reason`.
9. `FAILED` steps always have an `error_code` and a finished timestamp.
10. Step observations cannot reference a step key absent from the plan.
11. The central service never reports a step observation without `observed_at`.
12. Secrets never appear in task specs, snapshots, audit metadata, or logs.

## Result aggregation

Given a set of step observations, the execution result is:

| Condition | Result |
| --- | --- |
| Every step `COMPLETED` | `SUCCEEDED` |
| At least one `COMPLETED` and at least one `FAILED`/`SKIPPED` after a failure | `PARTIAL` |
| No `COMPLETED` step and at least one `FAILED` | `FAILED` |
| All steps `SKIPPED` because the first step failed | `FAILED` |
| Cancellation confirmed | `CANCELLED` |
| Worker lost before any start | `LOST` |

`FAIL_FAST` overrides the partial rule: any failed step makes the execution `FAILED`,
even when earlier steps completed.

## Identifiers

- Format: lowercase prefix, underscore, Crockford base32 ULID.
- Examples: `prj_01JQ8Z6K3M8Y4W2A7C9E1F5G0R`, `tsk_...`, `exe_...`, `wkr_...`.
- IDs are opaque to clients. Do not parse timestamps from them in the domain.
- Request IDs use `req_...`; they are not entities and are not persisted as aggregates.

## What does not get its own aggregate

- Payloads: value objects inside task specs, results, and snapshots.
- Step definitions: owned by the execution plan.
- Step observations: owned by the worker's local execution state; the central service
  stores the last observed snapshot.
- Logs: bounded output referenced by steps, not domain state.
- Generic events: use audit events for sensitive actions and snapshots for execution
  history. Do not create a second, competing history store.
