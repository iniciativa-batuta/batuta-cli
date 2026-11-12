# Implement configuration loading and validation

## Context

The server and worker need readable configuration, predictable precedence, and validation
before they open a database or a network listener. Secrets must never be logged or
embedded in example files.

## Objective

Implement typed configuration loading for the central service and worker, with strict
validation and documented precedence.

## Scope

- Create `internal/config` with `ServerConfig` and `WorkerConfig`.
- Load YAML from a file, apply environment overrides, then command-line overrides.
- Validate required fields, durations, ports, URLs, paths, and mutually exclusive options.
- Resolve secret references from files or environment variables at use time.
- Add `batuta config validate --kind server|worker --file <path>`.
- Support the examples in `docs/examples/server.yaml` and
  `docs/examples/worker.yaml`.
- Reject unknown YAML keys with a clear error.

## Out of scope

- Hot reload or remote configuration.
- A configuration UI.
- Storing secrets in the database.
- Defining domain defaults that belong to a use case.

## Acceptance criteria

- [ ] Both example files parse and validate.
- [ ] Invalid durations, missing required fields, and unknown keys produce actionable
      errors that name the file and field.
- [ ] Environment overrides take precedence over file values and are documented.
- [ ] Secret values are redacted from error messages, logs, and `config validate` output.
- [ ] `ServerConfig` rejects a non-loopback listener without TLS unless an explicit
      development flag is set.
- [ ] Tests cover precedence, invalid input, and secret redaction.

## Dependencies

- Blocked by: #1.
- Blocks: service startup, worker runtime, CLI configuration.

## Technical notes

- YAML requires a third-party decoder. Use a maintained YAML v3 library and record the
  justification in `docs/architecture/decisions.md` if it is not already covered.
- Keep configuration structs separate from domain structs.
- Environment variable names should be derived from a documented prefix and field path;
  avoid a grab-bag of unrelated names.
- `admin_token_file` and `credential_file` contain paths, not token values.
- Do not log the final merged configuration if it contains any secret-bearing field.

## Validation

```console
$ go test ./internal/config/...
$ go run ./cmd/batuta config validate --kind server --file docs/examples/server.yaml
$ go run ./cmd/batuta config validate --kind worker --file docs/examples/worker.yaml
```
