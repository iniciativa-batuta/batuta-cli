# Implement worker enrollment, credential storage, and heartbeat

## Context

A worker needs a durable identity and a liveness signal before it can receive work. The
heartbeat must not become a backdoor for step progress.

## Objective

Implement the worker join flow, secure credential storage, heartbeat loop, and central
offline detection.

## Scope

- Add `batuta worker join --service --token --name`.
- Exchange the enrollment token through `POST /v1/workers/enroll`.
- Write the worker ID and credential to the configured paths with mode `0600`.
- Add `batuta worker status` and `batuta worker list`.
- Add a heartbeat loop in the worker runtime with configurable interval and jitter.
- Add `POST /v1/worker/heartbeat`.
- Move a worker to `OFFLINE` when heartbeats are missed beyond `offline_after`.
- Add `batuta worker disable|revoke` for administrators.
- Add tests for token reuse, credential file permissions, heartbeat timing, and offline
  transitions.

## Out of scope

- Work channels and assignments.
- Step state and snapshots.
- TLS and credential rotation (rotation is a separate hardening issue).
- Automatic re-enrollment.

## Acceptance criteria

- [ ] `worker join` succeeds once per enrollment token.
- [ ] The credential file is created with `0600` and never printed in full.
- [ ] A second join attempt with the same token fails with a stable code.
- [ ] Heartbeats are sent at the configured interval with jitter.
- [ ] Missing heartbeats transition the worker to `OFFLINE` after the configured window.
- [ ] A heartbeat contains no step keys, step states, or progress percentages.
- [ ] `worker status` reports connection state, last heartbeat, and active execution
      count from the worker's local view.
- [ ] Revoked workers cannot heartbeat successfully.

## Dependencies

- Blocked by: #9, #10.
- Blocks: work channel, scheduler eligibility.

## Technical notes

- The worker credential is a bearer secret. Store it in a file with restrictive
  permissions or an OS keyring; do not put it in YAML.
- Heartbeat requests must be authenticated and scoped to the worker ID.
- Use jitter to avoid synchronized heartbeats after a central restart.
- Do not treat a missed heartbeat as a failed execution; it is a liveness condition.
- `worker status` should call the local runtime service, not duplicate its state logic.

## Validation

```console
$ go test ./internal/worker/... ./internal/httpapi/...
$ go test ./... -run Heartbeat
$ stat -c '%a' "$HOME/.config/batuta/worker.token"
```
