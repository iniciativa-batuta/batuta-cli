# Examples

These files illustrate the intended behavior of Batuta. They are documentation, not
versioned contracts; the contracts live under `contracts/` once implemented.

| File | Purpose |
| --- | --- |
| [task.yaml](task.yaml) | A `CONTINUE_ON_ERROR` task with dependencies. |
| [task-fail-fast.yaml](task-fail-fast.yaml) | A task that stops at the first failure. |
| [task-best-effort.yaml](task-best-effort.yaml) | A task that attempts independent steps after a failure. |
| [server.yaml](server.yaml) | Central service configuration with limits and timeouts. |
| [worker.yaml](worker.yaml) | Worker configuration with executor policy. |
| [snapshot.json](snapshot.json) | A worker snapshot response in progress. |
| [payloads.json](payloads.json) | Inline payload and artifact reference examples. |
| [cli-session.md](cli-session.md) | A complete local CLI walkthrough. |

When behavior changes, update the examples and the architecture documents in the same
change. Do not let examples drift from the actual CLI or payload schemas.
