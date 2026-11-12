# Build the central HTTP API foundation

## Context

Every central feature will be exposed through the same HTTP server. A consistent
foundation for routing, authentication, errors, limits, logging, and shutdown prevents
each handler from reinventing incompatible behavior.

## Objective

Implement the HTTP server skeleton and cross-cutting middleware without adding business
endpoints.

## Scope

- Create `internal/httpapi` using `net/http`.
- Add route registration under `/v1`.
- Add middleware for:
  - Request ID generation and propagation.
  - Panic recovery.
  - Structured request logging with redaction.
  - Authentication (admin token for administrative routes).
  - JSON content-type validation.
  - Request body limits.
  - Consistent error responses.
- Add `GET /v1/health` and `GET /v1/version`.
- Add read/write/idle timeouts and graceful shutdown.
- Add tests for status codes, error envelopes, auth failures, oversized bodies, and
  graceful shutdown.

## Out of scope

- Project, task, worker, execution, or artifact endpoints.
- Worker credential authentication (the middleware hook is added here; full worker auth
  is a later issue).
- Rate limiting beyond body limits.
- TLS termination; configuration is loaded but not implemented here.

## Acceptance criteria

- [ ] `GET /v1/health` returns a bounded JSON document without secrets.
- [ ] `GET /v1/version` returns product version, protocol version, commit, and build date.
- [ ] Unknown routes return a consistent `404` error envelope.
- [ ] Wrong methods return `405` with an `Allow` header.
- [ ] All errors use the common envelope and do not leak stack traces or SQL.
- [ ] Requests over `max_body_bytes` return `PAYLOAD_TOO_LARGE`.
- [ ] Authentication failure returns `401` or `403` with a stable code.
- [ ] Request IDs appear in logs and responses.
- [ ] Shutdown stops accepting new requests and waits for in-flight requests.

## Dependencies

- Blocked by: #2, #3.
- Blocks: every API endpoint and CLI client feature.

## Technical notes

- Use the standard library router. Do not add a web framework.
- Define one error-writing helper; handlers must not write ad hoc JSON errors.
- Logs are structured (`log/slog`) and must not include Authorization headers, tokens,
  or request bodies.
- Authentication middleware should accept a verifier interface so worker and admin
  credentials can share the pipeline without sharing semantics.
- The health endpoint must not disclose database contents or dependency details.

## Validation

```console
$ go test ./internal/httpapi/...
$ go test ./... -run HTTP
$ go vet ./...
```
