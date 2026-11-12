# Build the CLI client foundation and project/task commands

## Context

The CLI is the primary product interface. It must be a thin API client, not a second
implementation of scheduling, persistence, or validation.

## Objective

Implement the `batuta` command tree, HTTP client, authentication, output formats, exit
codes, and the first project/task commands.

## Scope

- Add a small command tree using the standard library `flag` package.
- Add global options:
  - `--server` or `BATUTA_SERVER`.
  - `--token-file` or `BATUTA_TOKEN_FILE`.
  - `--output table|json`.
  - `--timeout`.
- Add commands:
  - `batuta version`
  - `batuta project create|list|show`
  - `batuta task apply|list|show`
- Add a typed API client in `internal/cli/client` with request IDs, error decoding, and
  bounded responses.
- Add stable exit codes:
  - `0` success.
  - `1` operational error.
  - `2` usage error.
  - `3` task/execution failure when explicitly requested by a watch-style command.
- Add tests using `httptest` for success, validation errors, auth errors, and JSON
  output.

## Out of scope

- Worker commands, watch, cancel, logs, artifacts.
- Shell completion and interactive prompts.
- A CLI framework dependency.
- Local caching.

## Acceptance criteria

- [ ] `batuta project create demo` creates a project and prints its ID.
- [ ] `batuta project list` and `batuta project show` work with table output.
- [ ] `batuta task apply -f task.yaml` reads YAML, sends canonical JSON, and prints the
      task ID.
- [ ] Applying the same file twice reports `unchanged`.
- [ ] `--output json` returns machine-readable JSON with no table decoration.
- [ ] API errors are printed once with code and message, without stack traces.
- [ ] Missing configuration produces an actionable usage error and exit code `2`.
- [ ] The CLI does not import `internal/store`, `internal/domain`, or worker packages.

## Dependencies

- Blocked by: #3, #7, #8.
- Blocks: worker CLI commands, execution views, cancel and logs.

## Technical notes

- Keep command handlers thin: parse flags, call the client, format the result.
- Do not duplicate server validation. Surface field-level errors from the API.
- Use a table writer with bounded columns; disable colors when not a TTY.
- Never print the admin token or worker credential.
- The YAML-to-JSON conversion belongs in the CLI, not the server.
- If the command tree becomes unwieldy, open a separate decision issue before adding a
  framework.

## Validation

```console
$ go test ./internal/cli/...
$ go test ./... -run CLI
$ go run ./cmd/batuta project create demo --server http://127.0.0.1:8080
```
