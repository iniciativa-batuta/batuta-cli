# Worker protocol

## Design rules

1. **The worker initiates every connection.** The central service must work with workers
   behind NAT, firewalls, and private networks.
2. **Work is pulled.** The worker long-polls for assignments and control requests.
3. **Step progress is pulled, not pushed.** The central service asks for a snapshot. The
   worker does not send spontaneous step updates.
4. **Heartbeats are liveness only.** A heartbeat says "this worker process is alive and
   can accept connections". It never carries step state.
5. **Every mutation is scoped and idempotent.** A worker may only affect its own
   assignments. Repeated delivery must be safe.
6. **A snapshot is a coherent observation.** It has an `observed_at` timestamp and a
   monotonic sequence per execution. The central service stores it as observed data,
   never as authoritative worker state.

## Endpoints

All central endpoints are under `/v1`. JSON is UTF-8. The worker sends
`Authorization: Bearer <worker-credential>` on every authenticated call.

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/v1/workers/enroll` | Exchange a one-time enrollment token for a worker identity and credential. |
| `POST` | `/v1/worker/channel` | Long-poll for an assignment or a control request. |
| `POST` | `/v1/worker/heartbeat` | Report liveness and load. No step state. |
| `POST` | `/v1/worker/snapshots` | Respond to a snapshot request with an execution snapshot. |
| `POST` | `/v1/worker/credentials/rotate` | Rotate the worker credential. |
| `GET` | `/v1/local/executions/{id}/snapshot` | Optional direct snapshot endpoint on the worker, used when the central service can reach it. |

The optional direct endpoint is not the default. The channel path is the portable
mechanism and must work for every worker.

## Enrollment

An administrator creates a one-time enrollment token:

```console
$ batuta worker token create --name laptop --ttl 15m
wct_01JQ8Z6K3M8Y4W2A7C9E1F5G0R
```

The worker exchanges it:

```http
POST /v1/workers/enroll
Content-Type: application/json

{
  "protocol_version": "1",
  "enrollment_token": "wct_01JQ8Z6K3M8Y4W2A7C9E1F5G0R",
  "name": "laptop",
  "capabilities": {
    "os": "linux",
    "arch": "amd64",
    "executors": ["process"],
    "labels": {"toolchain": "go1.24"}
  },
  "max_concurrency": 1
}
```

```json
{
  "protocol_version": "1",
  "worker_id": "wkr_01JQ8Z7A1B2C3D4E5F6G7H8J9K",
  "credential_id": "wcr_01JQ8Z7A1B2C3D4E5F6G7H8J9K",
  "credential": "batuta_wkr_...",
  "heartbeat_interval_seconds": 15,
  "channel_wait_seconds": 30
}
```

Rules:

- The enrollment token is single-use and expires.
- The raw worker credential is returned once. The central service stores only its hash.
- The worker stores the credential in a file with mode `0600` or in an OS keyring. It is
  never logged.
- The worker ID is immutable for the lifetime of the installation. Re-enrolling creates
  a new worker unless an administrator explicitly revokes the old one.

## Channel long-poll

The worker keeps one long-poll open per runtime. The request includes a cursor so the
central service can avoid delivering the same control request twice.

```http
POST /v1/worker/channel
Authorization: Bearer <worker-credential>
Content-Type: application/json

{
  "protocol_version": "1",
  "worker_id": "wkr_01JQ8Z7A1B2C3D4E5F6G7H8J9K",
  "wait_seconds": 30,
  "cursor": "cur_01JQ8Z7A1B2C3D4E5F6G7H8J9K"
}
```

### Assignment response

```json
{
  "protocol_version": "1",
  "type": "assignment",
  "assignment_id": "asn_01JQ8Z8B1C2D3E4F5G6H7J8K9M",
  "execution_id": "exe_01JQ8Z8B1C2D3E4F5G6H7J8K9M",
  "task_id": "tsk_01JQ8Z6K3M8Y4W2A7C9E1F5G0R",
  "plan_version": 1,
  "plan_hash": "sha256:...",
  "deadline": "2026-11-12T12:30:00Z",
  "continuity": "CONTINUE_ON_ERROR",
  "steps": [
    {
      "key": "deps",
      "position": 0,
      "name": "Download dependencies",
      "depends_on": [],
      "executor": {
        "type": "process",
        "executable": "go",
        "args": ["mod", "download"],
        "cwd": "/workspace/app",
        "timeout": "5m"
      }
    }
  ]
}
```

The worker must validate:

- `plan_version` and `plan_hash` are present and internally consistent.
- Step keys are unique.
- Every dependency references a step in the same plan.
- Every executor type is enabled by local policy.
- The requested working directory and environment are allowed.

If validation fails, the worker does **not** start the execution. It reports a
`PLAN_REJECTED` execution result through the next snapshot request or directly through a
`POST /v1/worker/snapshots` with a rejected execution state, as defined by the
implementation. A rejection must include a stable reason code.

### Control response

```json
{
  "protocol_version": "1",
  "type": "control",
  "request_id": "req_01JQ8Z9C1D2E3F4G5H6J7K8M9N",
  "action": "snapshot.request",
  "execution_id": "exe_01JQ8Z8B1C2D3E4F5G6H7J8K9M",
  "deadline": "2026-11-12T12:00:05Z"
}
```

Supported actions in the MVP:

| Action | Meaning | Response |
| --- | --- | --- |
| `snapshot.request` | Return the current state of one execution. | `POST /v1/worker/snapshots` |
| `execution.cancel` | Stop the execution and terminate its process group. | Reflected in the next snapshot; an explicit ack is optional. |

### No work response

```json
{
  "protocol_version": "1",
  "type": "none",
  "cursor": "cur_01JQ8Z9C1D2E3F4G5H6J7K8M9N",
  "retry_after_ms": 1500
}
```

The worker waits at least `retry_after_ms`, adds jitter, and reconnects. It must not
tight-loop.

## Heartbeat

```http
POST /v1/worker/heartbeat
Authorization: Bearer <worker-credential>
Content-Type: application/json

{
  "protocol_version": "1",
  "worker_id": "wkr_01JQ8Z7A1B2C3D4E5F6G7H8J9K",
  "load": {
    "active_executions": 1,
    "max_concurrency": 1
  }
}
```

```json
{
  "protocol_version": "1",
  "accepted": true,
  "server_time": "2026-11-12T12:00:00Z"
}
```

Heartbeat rules:

- Sent every `heartbeat_interval_seconds` with jitter.
- Missing heartbeats move the worker to `OFFLINE` after the configured window.
- Heartbeats never contain step states, step transitions, or progress percentages.
- `active_executions` is a scheduling hint, not authoritative execution state.

## Snapshot response

The worker responds to a `snapshot.request`:

```http
POST /v1/worker/snapshots
Authorization: Bearer <worker-credential>
Content-Type: application/json

{
  "protocol_version": "1",
  "request_id": "req_01JQ8Z9C1D2E3F4G5H6J7K8M9N",
  "snapshot": {
    "schema": "batuta.worker.snapshot/v1",
    "worker_id": "wkr_01JQ8Z7A1B2C3D4E5F6G7H8J9K",
    "execution_id": "exe_01JQ8Z8B1C2D3E4F5G6H7J8K9M",
    "plan_version": 1,
    "sequence": 7,
    "observed_at": "2026-11-12T12:00:04Z",
    "execution": {
      "state": "RUNNING",
      "started_at": "2026-11-12T12:00:01Z",
      "finished_at": null,
      "result": null,
      "error_code": null
    },
    "steps": [
      {
        "key": "deps",
        "state": "COMPLETED",
        "attempt": 1,
        "started_at": "2026-11-12T12:00:01Z",
        "finished_at": "2026-11-12T12:00:08Z",
        "exit_code": 0,
        "error_code": null,
        "skip_reason": null
      },
      {
        "key": "test",
        "state": "RUNNING",
        "attempt": 1,
        "started_at": "2026-11-12T12:00:09Z",
        "finished_at": null,
        "exit_code": null,
        "error_code": null,
        "skip_reason": null
      }
    ]
  }
}
```

Response:

```json
{
  "protocol_version": "1",
  "accepted": true,
  "next_snapshot_after_ms": 5000
}
```

Rules:

- `observed_at` is generated by the worker at the moment the snapshot is taken.
- `sequence` is monotonic per execution. The central service rejects a snapshot whose
  sequence is older than the last accepted observation for that execution.
- `request_id` must match a pending request for this worker and execution.
- Repeating the same `request_id` with an identical body is idempotent and returns the
  same result. Reusing it with a different body is a conflict.
- A snapshot contains no secrets. Environment variable values are never included.
- Snapshots are bounded by `max_snapshot_bytes`. Oversized output is referenced by
  `output_ref`, not embedded.

## State reconciliation

The central service runs a reconciler that requests snapshots for executions in
`ASSIGNED` or `RUNNING`. The interval is configurable and applies backoff when snapshots
are slow or the worker is under load.

- The reconciler stores the last accepted snapshot with `observed_at` and `sequence`.
- A snapshot older than `snapshot_stale_after` is marked stale in the API and CLI.
- Repeating a query does not mutate the execution or the snapshot.
- The central service finalizes an execution only from a snapshot that reports a
  terminal execution state, or from an explicit timeout/loss policy.
- If a worker is offline, the execution keeps its last observed state and is marked
  stale. It is marked `LOST` only after the documented offline grace period or when the
  assignment expires before the first start.

## Optional direct pull

When the worker advertises a reachable address and both sides have compatible
credentials, the central service may call:

```http
GET /v1/local/executions/{execution_id}/snapshot
Authorization: Bearer <worker-credential>
```

The response is the same snapshot object, without the outer `request_id` envelope. The
central service records the same observation metadata.

Direct pull is an optimization. The channel path remains mandatory, because it is the
only path that works for NAT and restricted egress.

## Cancellation

1. An authorized actor calls `POST /v1/executions/{id}/cancel` or
   `batuta task cancel`.
2. The central service records `cancel_requested_at`.
3. If the worker's channel is open, the next poll receives `execution.cancel`.
4. The worker terminates the running process group, marks running steps `CANCELLED`, and
   marks unstarted steps `SKIPPED` with `CANCELLED_BY_USER`.
5. The next snapshot confirms the result. The central service finalizes the execution as
   `CANCELLED`.
6. If the worker is unreachable for longer than `cancel_timeout`, the central service
   finalizes the execution as `CANCELLED` with `cancel_confirmed=false`. The worker must
   discard any later snapshot for that execution and must not revive it.

## Recovery and failure handling

### Worker restart

- The worker persists the execution plan and step state locally before starting work.
- On startup, it loads unfinished executions.
- A step that was `RUNNING` when the worker died is marked `FAILED` with
  `WORKER_RESTARTED`, because the process cannot be safely reattached.
- The continuity policy then decides whether to continue remaining steps.
- The worker reports the reconciled state in the next snapshot.

### Central service restart

- Pending snapshot requests expire after their deadline.
- The reconciler re-creates requests for active executions.
- Assignment delivery is at-least-once; the worker deduplicates by `assignment_id`.
- Idempotency keys and accepted snapshot sequences make replays safe.

### Duplicate delivery

- `assignment_id`: the worker accepts the first delivery and ignores later deliveries,
  returning the current state on request.
- `request_id` for snapshots: repeated identical responses are idempotent.
- `execution.cancel`: cancelling an already terminal execution is a no-op.
- Enrollment tokens: single use; a second attempt is rejected.

### Clock skew

- Both sides use UTC.
- The worker's `observed_at` is authoritative for when the observation was taken.
- The central service's `received_at` is recorded separately.
- A snapshot with `observed_at` too far in the future is rejected as
  `INVALID_TIMESTAMP`.

## Versioning

- The protocol major version is `1` (`"protocol_version": "1"`).
- Envelope schemas use explicit names such as `batuta.worker.snapshot/v1`.
- Additive fields are allowed within a major version. Unknown fields are ignored.
- Removing or changing the meaning of a field requires a new major version.
- A central service rejects an unsupported major version with
  `PROTOCOL_VERSION_UNSUPPORTED` and does not partially process the request.
- The product version and the protocol version are independent.

## Error envelope

```json
{
  "error": {
    "code": "SNAPSHOT_REQUEST_UNKNOWN",
    "message": "No pending snapshot request matches this request ID.",
    "request_id": "req_01JQ8Z9C1D2E3F4G5H6J7K8M9N",
    "retryable": false
  }
}
```

Stable codes used by the protocol include:

| Code | Meaning |
| --- | --- |
| `PROTOCOL_VERSION_UNSUPPORTED` | The major version is not supported. |
| `AUTHENTICATION_REQUIRED` | Missing or invalid credential. |
| `WORKER_REVOKED` | The worker credential or worker was revoked. |
| `WORKER_SCOPE_VIOLATION` | The worker tried to access another worker's data. |
| `ASSIGNMENT_UNKNOWN` | The assignment does not exist or is not for this worker. |
| `SNAPSHOT_REQUEST_UNKNOWN` | No matching pending request. |
| `SNAPSHOT_REQUEST_EXPIRED` | The request deadline passed. |
| `SNAPSHOT_SEQUENCE_STALE` | The snapshot is older than the last accepted one. |
| `INVALID_TIMESTAMP` | Timestamp is missing, malformed, or outside the allowed skew. |
| `INVALID_SNAPSHOT` | The snapshot violates the schema or the assigned plan. |
| `PAYLOAD_TOO_LARGE` | The request exceeds the configured limit. |
| `EXECUTION_TERMINAL` | The execution is already terminal. |
| `EXECUTION_LOST` | The execution was finalized as lost. |

## Open questions

- Whether the worker should keep the channel open per execution or one channel for the
  whole runtime. The MVP uses one channel per runtime.
- Whether cancellation requires an explicit acknowledgment in addition to the next
  snapshot. The MVP relies on the snapshot and the `cancel_timeout` fallback.
- Whether direct pull should be enabled by default in LAN deployments. It is opt-in
  until mutual authentication is implemented.
