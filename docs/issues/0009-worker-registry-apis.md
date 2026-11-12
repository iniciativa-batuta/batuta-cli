# Implement worker enrollment, registry, and execution read APIs

## Context

Workers need an identity before they can receive work. Administrators need to list,
inspect, disable, and revoke them. Execution records must be readable as soon as they
are created, even before the worker channel exists.

## Objective

Implement worker enrollment, credential issuance, worker administration, and execution
read endpoints.

## Scope

- Add `POST /v1/enrollment-tokens` for administrators to create a one-time token.
- Add `POST /v1/workers/enroll` to exchange a token for a worker identity and credential.
- Add `GET /v1/workers`, `GET /v1/workers/{id}`.
- Add `POST /v1/workers/{id}/disable` and `POST /v1/workers/{id}/revoke`.
- Add `GET /v1/executions/{id}` and `GET /v1/tasks/{id}/executions`.
- Hash credentials before storage; return the raw credential only once.
- Store enrollment token hashes with TTL and single-use semantics.
- Validate capabilities and normalize them.
- Add tests for reuse, expiry, revocation, and cross-worker access.

## Out of scope

- The long-poll channel and heartbeat.
- Scheduler assignment.
- TLS and certificate management.
- OAuth, OIDC, or per-user accounts.

## Acceptance criteria

- [ ] An enrollment token can be created with a name and TTL and is returned once.
- [ ] Reusing an enrollment token returns `ENROLLMENT_TOKEN_USED`.
- [ ] An expired token returns `ENROLLMENT_TOKEN_EXPIRED`.
- [ ] The raw worker credential is returned only in the enrollment response.
- [ ] The database stores only a strong hash of the credential and token.
- [ ] Disabling a worker blocks new assignments without invalidating running work unless
      revocation is requested.
- [ ] Revoking a worker invalidates its credentials immediately.
- [ ] A worker credential cannot read another worker's details or executions.
- [ ] Execution read responses include the plan version, state, and observation
      metadata, but not fabricated step progress.

## Dependencies

- Blocked by: #5, #6, #7.
- Blocks: worker runtime, scheduler, CLI worker commands.

## Technical notes

- Use 32 bytes of cryptographically secure randomness for credentials and tokens.
- Hash with Argon2id or an equivalent memory-hard function; record the parameters.
- Enrollment tokens are scoped to one intended worker name but must still be validated
  on exchange.
- `disable` means "stop new assignments"; `revoke` means "invalidate credentials and
  stop all use". Keep the two operations distinct in the API and audit trail.
- Do not return credential hashes in any response.

## Validation

```console
$ go test ./internal/httpapi/...
$ go test ./internal/app/...
$ go test ./... -run Enrollment
```
