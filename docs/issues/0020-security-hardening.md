# Harden worker authentication, transport security, and secret handling

## Context

Remote process execution is a security boundary. A stolen or over-scoped credential can
run arbitrary code on a worker. TLS, scoped tokens, rotation, and redaction are required
before the MVP can be used outside a trusted loopback setup.

## Objective

Make authentication and secret handling explicit across the central service, CLI, and
worker.

## Scope

- Require authentication on every non-public endpoint.
- Scope worker credentials to their worker ID and assigned executions.
- Add credential rotation:
  - `POST /v1/worker/credentials/rotate`
  - `batuta worker rotate`.
- Add revocation with immediate effect.
- Require TLS for non-loopback listeners and worker connections unless an explicit
  development flag is set.
- Add a startup warning when TLS is disabled on a non-loopback address.
- Redact `Authorization`, tokens, cookies, and secret-like fields from logs and errors.
- Enforce `envFrom` resolution on the worker only; never send values to the central
  service.
- Add tests for cross-worker access, expired credentials, rotation, revocation, TLS
  requirements, and log redaction.

## Out of scope

- mTLS and client certificates.
- OAuth/OIDC, SSO, and multi-user RBAC.
- Hardware-backed key storage.
- Automatic certificate issuance.

## Acceptance criteria

- [ ] Every administrative and worker endpoint rejects missing or invalid credentials.
- [ ] A worker cannot read or mutate another worker's executions, snapshots, or
      artifacts.
- [ ] Rotating a credential invalidates the old credential immediately.
- [ ] Revoking a worker invalidates all of its credentials.
- [ ] Non-loopback HTTP listeners are rejected unless the development flag is explicit.
- [ ] Secrets do not appear in logs, error responses, audit metadata, or snapshots.
- [ ] Expired credentials return a stable error and do not leak the expected value.
- [ ] Tests cover the policy matrix for admin, worker, and unauthenticated access.

## Dependencies

- Blocked by: #7, #9, #11, #12, #14.
- Blocks: MVP use outside loopback, audit trail, integrations.

## Technical notes

- Use constant-time comparison for token hashes and avoid early-exit leaks where
  practical.
- Store credentials hashed with a memory-hard function; record parameters and support
  rehashing on rotation.
- Do not implement custom cryptography.
- TLS configuration belongs to the server and worker adapters; domain code must not know
  about certificates.
- A development flag that disables TLS must be loud, explicit, and refused in release
  builds if possible.

## Validation

```console
$ go test ./internal/httpapi/... ./internal/worker/...
$ go test ./... -run 'Auth|TLS|Redact'
$ go test -race ./internal/httpapi/...
```
