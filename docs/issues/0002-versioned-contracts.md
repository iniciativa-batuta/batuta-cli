# Define versioned contracts, identifiers, and the error model

## Context

The central service, CLI, and worker must agree on wire formats and error semantics.
Without a single versioned contract package, each side will grow its own DTOs and the
protocol will drift.

## Objective

Define the shared protocol types, identifier format, version rules, and stable error
envelope used by every component.

## Scope

- Create `internal/protocol` with version `v1` types for:
  - Project, Task, Execution, Worker, Step definition, Step observation, Artifact.
  - Snapshot envelopes, assignment envelopes, control requests, heartbeat, and
    enrollment.
  - The common JSON error envelope.
- Add a prefixed ULID generator with helpers for `prj_`, `tsk_`, `exe_`, `wkr_`, `stp_`,
  `req_`, `asn_`, `art_`, `aud_`, `wcr_`, and `wct_`.
- Add protocol version constants and validation helpers.
- Add JSON Schema or OpenAPI documents under `contracts/` for the external envelopes.
- Add golden JSON fixtures covering the examples in `docs/examples/`.
- Document versioning rules: additive changes within a major version, unknown fields
  ignored, unsupported major versions rejected.

## Out of scope

- HTTP handlers, storage mappings, scheduling, execution.
- Generated clients or code generation.
- Protobuf or gRPC.

## Acceptance criteria

- [ ] Every envelope includes an explicit protocol or schema version.
- [ ] Golden fixtures round-trip without data loss.
- [ ] Unknown JSON fields are ignored without failing.
- [ ] An unsupported major version returns `PROTOCOL_VERSION_UNSUPPORTED`.
- [ ] ID helpers reject malformed prefixes and duplicate invalid input.
- [ ] Error codes are typed constants, not ad hoc strings.
- [ ] Secret-bearing fields are not part of snapshot or task DTOs.
- [ ] Contract tests run with `go test ./internal/protocol/...`.

## Dependencies

- Blocked by: #1.
- Blocks: API, worker protocol, CLI client, persistence mapping.

## Technical notes

- Keep DTOs free of domain behavior. Domain and protocol types may be converted in
  `internal/app`.
- `apiVersion: batuta/v1` identifies the task document format; `protocol_version: "1"`
  identifies the worker wire protocol. They are related but not interchangeable.
- IDs are opaque to clients even though ULIDs sort by time.
- Use `encoding/json` with explicit struct tags. Reject duplicate JSON keys when
  decoding external input if the standard decoder allows configuration for it; otherwise
  document the limitation.
- Do not expose internal database columns in the public contract.

## Validation

```console
$ go test ./internal/protocol/...
$ go test ./... -run Contract
$ go vet ./...
```
