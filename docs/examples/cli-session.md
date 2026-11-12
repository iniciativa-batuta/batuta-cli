# CLI session

A complete local walkthrough. IDs and timestamps are illustrative.

## 1. Initialize and start the central service

```console
$ batuta server init --dir ./batuta-data
Initialized Batuta server configuration at ./batuta-data/server.yaml
Created admin token at ./batuta-data/admin.token

$ batuta server start --config ./batuta-data/server.yaml
level=INFO msg="batuta server listening" address=127.0.0.1:8080
```

## 2. Create a project and apply a task

```console
$ batuta project create demo --description "Local demonstration"
prj_01JQ8Z5A1B2C3D4E5F6G7H8J9K

$ batuta task apply -f docs/examples/task.yaml
tsk_01JQ8Z6K3M8Y4W2A7C9E1F5G0R

$ batuta task apply -f docs/examples/task.yaml
tsk_01JQ8Z6K3M8Y4W2A7C9E1F5G0R  unchanged
```

The second apply is idempotent because the normalized spec hash did not change.

## 3. Register a worker

```console
$ batuta worker token create --name workstation --ttl 15m
wct_01JQ8Z6M1N2P3Q4R5S6T7V8W9X

$ batuta worker join --service http://127.0.0.1:8080 \
    --token wct_01JQ8Z6M1N2P3Q4R5S6T7V8W9X \
    --name workstation
Joined worker wkr_01JQ8Z7A1B2C3D4E5F6G7H8J9K
Credential written to /home/user/.config/batuta/worker.token (mode 0600)

$ batuta worker run --config ./worker.yaml
level=INFO msg="worker online" worker_id=wkr_01JQ8Z7A1B2C3D4E5F6G7H8J9K
```

## 4. Watch the execution

```console
$ batuta task list --project demo
ID                              NAME               STATE      UPDATED
tsk_01JQ8Z6K3M8Y4W2A7C9E1F5G0R  test-and-package   RUNNING    12:00:09

$ batuta task watch tsk_01JQ8Z6K3M8Y4W2A7C9E1F5G0R
EXECUTION                       STATE    OBSERVED   STALE
exe_01JQ8Z8B1C2D3E4F5G6H7J8K9M  RUNNING  12:00:09   no

STEP     NAME                    STATE      STARTED   FINISHED  EXIT
deps     Download dependencies   COMPLETED  12:00:01  12:00:08  0
test     Run tests               RUNNING    12:00:09  —         —
package  Build release binary    PENDING    —         —         —
```

The view shows step states and the observation timestamp. It does not show a fabricated
percentage.

## 5. Inspect results

```console
$ batuta execution show exe_01JQ8Z8B1C2D3E4F5G6H7J8K9M
EXECUTION  exe_01JQ8Z8B1C2D3E4F5G6H7J8K9M
TASK       tsk_01JQ8Z6K3M8Y4W2A7C9E1F5G0R
WORKER     wkr_01JQ8Z7A1B2C3D4E5F6G7H8J9K
STATE      PARTIAL
RESULT     PARTIAL
OBSERVED   12:04:12 (stale: no)

$ batuta execution logs exe_01JQ8Z8B1C2D3E4F5G6H7J8K9M --step test
...

$ batuta execution artifacts exe_01JQ8Z8B1C2D3E4F5G6H7J8K9M
ID                              NAME              SIZE    SHA256
art_01JQ8Z8D1E2F3G4H5J6K7M8N9P  test-report.json  4096    2cf24dba...
```

## 6. Cancel an execution

```console
$ batuta task cancel tsk_01JQ8Z6K3M8Y4W2A7C9E1F5G0R
Cancellation requested.

$ batuta task show tsk_01JQ8Z6K3M8Y4W2A7C9E1F5G0R
STATE      CANCELLED
CANCELED   12:05:01
```

The worker confirms through the next snapshot. If the worker is unreachable, the
central service finalizes the execution after `cancel_timeout` and records that the
cancellation was not confirmed.

## 7. Inspect worker state

```console
$ batuta worker list
ID                              NAME         STATE    LAST HEARTBEAT
wkr_01JQ8Z7A1B2C3D4E5F6G7H8J9K  workstation  ONLINE   12:05:02
```

## 8. Failure example

With `FAIL_FAST`, a failed first step skips the rest:

```console
$ batuta task show tsk_01JQ8Z9D1E2F3G4H5J6K7M8N9P
STATE  FAILED

STEP     STATE    ERROR
migrate  FAILED   STEP_EXIT_NONZERO (exit 1)
deploy   SKIPPED  PREVIOUS_STEP_FAILED
```

With `CONTINUE_ON_ERROR`, independent steps still run and the execution can end
`PARTIAL`. See [execution and steps](../architecture/execution-and-steps.md).
