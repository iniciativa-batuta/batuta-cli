# Execution and steps

## Principles

- A task defines **what** should run. An execution is **one attempt** to run it.
- The central service defines the step list, order, dependencies, and executor
  specifications. The worker executes that plan without redefining it.
- The worker owns the runtime state of the steps it runs.
- The central service observes that state through snapshots.
- Progress is represented by step states, never by invented percentages.
- Failures, skips, partial results, timeouts, and cancellation are explicit data.

## Task definition

```yaml
apiVersion: batuta/v1
kind: Task
metadata:
  project: demo
  name: test-and-package
  labels:
    suite: backend
spec:
  priority: normal
  continuity: CONTINUE_ON_ERROR
  timeout: 30m
  requirements:
    os: linux
    arch: amd64
    executors: [process]
    labels:
      toolchain: go1.24
  dependsOn: []
  steps:
    - id: deps
      name: Download dependencies
      executor:
        type: process
        executable: go
        args: ["mod", "download"]
        cwd: /workspace/app
        timeout: 5m
    - id: test
      name: Run tests
      dependsOn: [deps]
      executor:
        type: process
        executable: go
        args: ["test", "./..."]
        cwd: /workspace/app
        env:
          CGO_ENABLED: "0"
        timeout: 15m
        output:
          stdout_limit_bytes: 2097152
          stderr_limit_bytes: 2097152
    - id: package
      name: Build release binary
      dependsOn: [test]
      executor:
        type: process
        executable: go
        args: ["build", "-trimpath", "-o", "dist/batuta", "./cmd/batuta"]
        cwd: /workspace/app
        timeout: 10m
```

### Requirements

Requirements are scheduling hints and worker-side policy inputs, not trust claims.

| Field | Meaning |
| --- | --- |
| `os` | Required operating system (`linux`, `darwin`, `windows`). |
| `arch` | Required architecture (`amd64`, `arm64`, ...). |
| `executors` | Executor types that must be enabled on the worker. |
| `labels` | Exact-match worker labels. |

The central service never assigns a task to a worker that does not declare the required
capabilities. The worker re-validates the plan and refuses work it cannot safely run.

### Task dependencies

`spec.dependsOn` lists other task IDs in the same project. A task is eligible to run
only when every dependency is `SUCCEEDED`. A dependency that ends `PARTIAL`, `FAILED`,
`CANCELLED`, or `LOST` blocks the dependent task with an explicit `blocked_reason`.

There is no implicit success for partial results. A future `onDependency` policy may
change this; it is not part of the MVP.

## Step definitions

Each step has:

- `id`: stable key, unique in the task.
- `name`: display name.
- `dependsOn`: step keys that must be `COMPLETED` before this step starts.
- `executor`: structured executor specification.
- `timeout`: optional step timeout; defaults to the remaining execution timeout.
- `output`: optional capture limits.

The central service validates the graph before placing the task:

- No duplicate step IDs.
- No dependency on an unknown step.
- No cycle.
- At least one step.
- Every executor type is known to the schema.

The MVP executes one step at a time. Dependencies still matter because they determine
which steps are ready when a failure changes the plan.

## Executor specifications

### Process executor

The default executor. It never invokes a shell.

```yaml
executor:
  type: process
  executable: /usr/bin/python3
  args: ["-m", "pytest", "-q"]
  cwd: /workspace/app
  env:
    CI: "1"
  envFrom:
    - PYPI_TOKEN
  stdin: null
  timeout: 10m
  output:
    stdout_limit_bytes: 1048576
    stderr_limit_bytes: 1048576
```

Rules:

- `executable` and `args` are passed to the operating system verbatim. There is no
  string interpolation and no implicit shell.
- `cwd` must be an absolute path inside an allowed root configured on the worker.
- `env` contains non-secret literal values only.
- `envFrom` lists variable names resolved from the worker process environment. Values
  are never sent to the central service, stored in task specs, or included in snapshots.
- `stdin` is optional and bounded.
- Output beyond the limit is truncated with an explicit marker; the full output may be
  stored locally or uploaded as an artifact when the implementation supports it.

### Shell executor

Shell is explicit and disabled by default.

```yaml
executor:
  type: shell
  shell: /bin/bash
  script: |
    set -euo pipefail
    go test ./...
```

Rules:

- The worker configuration must enable the shell executor and may whitelist shells.
- The script is executed as a single unit with the configured shell.
- The worker never interpolates untrusted task fields into a shell command. A shell step
  is trusted code submitted by an authorized user; policy still controls whether it may
  run.
- Shell output uses the same limits and redaction rules as the process executor.

### Future executors

`git`, `opencode`, `docker`, and other adapters must implement the same executor
contract: validate the spec, run with a context/timeout, capture bounded output, return
a structured result, and support cancellation. They are disabled unless explicitly
configured.

## Step states

| State | Meaning | Terminal |
| --- | --- | --- |
| `PENDING` | Defined, not started. | No |
| `RUNNING` | A process or adapter is active. | No |
| `COMPLETED` | Finished successfully. | Yes |
| `FAILED` | Finished with an error, timeout, or crash. | Yes |
| `SKIPPED` | Not started because of policy, a failed dependency, or cancellation. | Yes |
| `CANCELLED` | Started and then cancelled. | Yes |

Legal transitions:

```text
PENDING → RUNNING | SKIPPED | CANCELLED
RUNNING → COMPLETED | FAILED | CANCELLED
```

Any other transition is rejected. A terminal step is immutable.

### Skip reasons

Every `SKIPPED` step carries exactly one reason:

| Reason | Meaning |
| --- | --- |
| `PREVIOUS_STEP_FAILED` | `FAIL_FAST` stopped the sequence. |
| `DEPENDENCY_FAILED` | A dependency did not complete successfully. |
| `CANCELLED_BY_USER` | The execution was cancelled before the step started. |
| `EXECUTION_TIMEOUT` | The execution deadline expired before the step started. |
| `POLICY_SKIPPED` | A worker policy or continuity rule skipped it. |
| `PLAN_REJECTED` | The worker refused the plan before starting. |

### Error codes

| Code | Meaning |
| --- | --- |
| `STEP_EXIT_NONZERO` | The process exited non-zero. |
| `STEP_TIMEOUT` | The step timeout expired. |
| `EXECUTION_TIMEOUT` | The execution deadline expired. |
| `STEP_CANCELLED` | The step was cancelled by an authorized actor. |
| `WORKER_RESTARTED` | The worker restarted while the step was running. |
| `EXECUTOR_UNAVAILABLE` | The executor is not enabled or not installed. |
| `OUTPUT_LIMIT_EXCEEDED` | Output exceeded the configured hard limit. |
| `INVALID_SPEC` | The executor specification is invalid. |
| `CWD_NOT_ALLOWED` | The working directory is outside the allowed roots. |
| `ENV_NOT_ALLOWED` | A requested environment variable is not permitted. |
| `PROCESS_START_FAILED` | The executable could not be started. |

## Continuity profiles

The continuity policy is configured once per task in `spec.continuity`. The MVP does not
support per-step overrides. A retry creates a new execution; it does not mutate terminal
steps.

### `FAIL_FAST`

Stop at the first failed step.

1. Record the failed step, its exit code, error code, and bounded output reference.
2. Do not start any further step.
3. Mark every unstarted step `SKIPPED` with `PREVIOUS_STEP_FAILED`.
4. Final execution result: `FAILED`, even if earlier steps completed.

Use when later steps are unsafe or meaningless after a failure.

Example:

```yaml
spec:
  continuity: FAIL_FAST
  steps:
    - id: migrate
      name: Apply migrations
      executor: {type: process, executable: ./migrate, args: ["up"]}
    - id: deploy
      name: Deploy
      dependsOn: [migrate]
      executor: {type: process, executable: ./deploy}
```

If `migrate` fails, `deploy` is `SKIPPED` with `PREVIOUS_STEP_FAILED` and the execution is
`FAILED`.

### `CONTINUE_ON_ERROR`

Record the failure and continue in declared order while dependencies and safety allow.

1. Record the failed step.
2. Move to the next declared step.
3. If its dependencies are satisfied, run it.
4. If a dependency failed, skipped, or was cancelled, mark it `SKIPPED` with
   `DEPENDENCY_FAILED`.
5. Final result:
   - `SUCCEEDED` if no step failed.
   - `PARTIAL` if at least one step completed and at least one failed or was skipped
     because of a failure.
   - `FAILED` if no step completed and at least one failed.

Example:

```yaml
spec:
  continuity: CONTINUE_ON_ERROR
  steps:
    - id: lint
      executor: {type: process, executable: golangci-lint, args: ["run"]}
    - id: test
      executor: {type: process, executable: go, args: ["test", "./..."]}
    - id: package
      dependsOn: [test]
      executor: {type: process, executable: go, args: ["build", "./..."]}
```

If `lint` fails but `test` succeeds, `package` runs and the execution is `PARTIAL`.

### `BEST_EFFORT`

Maximize the number of attempted steps.

1. Record each failure.
2. After a failure, choose any remaining step whose dependencies are satisfied, not
   necessarily the next declared step.
3. Continue until no remaining step can run.
4. Mark blocked steps `SKIPPED` with `DEPENDENCY_FAILED`.
5. Produce a summary of completed, failed, skipped, and cancelled steps.
6. Final result uses the same mapping as `CONTINUE_ON_ERROR`. Never convert failures to
   success.

With the MVP's sequential execution, `BEST_EFFORT` differs from `CONTINUE_ON_ERROR` when
a failed step blocks an early chain but later independent steps are still ready. The
runner may skip ahead to those steps. Parallel execution will make the distinction more
visible; it is not part of the MVP.

## Timeouts

- **Step timeout:** the step is `FAILED` with `STEP_TIMEOUT`; the continuity policy
  applies.
- **Execution timeout:** the running step is cancelled, unstarted steps are `SKIPPED`
  with `EXECUTION_TIMEOUT`, and the execution is `FAILED` with `EXECUTION_TIMEOUT`.
- **Assignment deadline:** if the worker never starts before the deadline, the execution
  is `LOST` and the task may be retried manually.
- Timeouts are enforced by the worker using process groups. Killing only the direct
  child is not sufficient.

## Cancellation

Cancellation is a request, not an immediate state mutation.

1. An authorized actor requests cancellation.
2. The central service records `cancel_requested_at`.
3. The worker receives `execution.cancel`.
4. The worker terminates the running process group, marks running steps `CANCELLED` with
   `STEP_CANCELLED`, and marks unstarted steps `SKIPPED` with `CANCELLED_BY_USER`.
5. The execution and task become `CANCELLED` when the snapshot confirms it.
6. If the worker is unreachable beyond `cancel_timeout`, the central service finalizes
   the execution as `CANCELLED` with `cancel_confirmed=false`. Later snapshots for that
   execution are rejected as `EXECUTION_TERMINAL`.

## Retries

The MVP does not retry steps automatically inside an execution.

- A failed or partial execution can be retried explicitly, creating a new execution.
- The new execution gets a new plan version and a new set of step observations.
- Terminal steps from the previous execution are never reused or mutated.
- The task's official state follows the latest accepted execution.
- Automatic retry policy, backoff, and per-step retry are deferred.

This keeps failure semantics honest: a retry is a new attempt, not a rewritten history.

## Worker restart

The worker persists the plan and step observations before and during execution.

On restart:

1. Load unfinished executions.
2. Any step that was `RUNNING` becomes `FAILED` with `WORKER_RESTARTED`.
3. `PENDING` steps remain pending.
4. The continuity policy decides whether to continue.
5. Report the reconciled snapshot.

A process cannot be safely reattached across a worker restart. Pretending it can would
produce false state.

## Output and logs

- Stdout and stderr are bounded per step.
- The structured result includes exit code and error code, not a transcript.
- Logs are referenced by `output_ref`; they are not embedded in snapshots.
- Secrets are redacted before storage. A worker must never include environment values in
  output metadata.
- Logs are not a substitute for step state.

## Examples

See:

- [Full task definition](../examples/task.yaml)
- [Fail-fast task](../examples/task-fail-fast.yaml)
- [Snapshot payload](../examples/snapshot.json)
- [CLI session](../examples/cli-session.md)
