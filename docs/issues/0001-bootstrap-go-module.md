# Bootstrap the Go module and binary skeleton

## Context

The repository currently contains documentation, a license, and ignore rules, but no Go
module or executable. Every later issue depends on a reproducible build, a stable package
layout, and a version command.

## Objective

Create the minimal Go foundation for the `batuta` binary without implementing product
behavior.

## Scope

- Initialize the module as `github.com/iniciativa-batuta/batuta-cli`.
- Add `cmd/batuta/main.go` with a root command and a `version` subcommand.
- Add `internal/version` with `Version`, `Commit`, and `BuildDate` variables set through
  linker flags.
- Add the package layout described in
  [the README layout](https://github.com/iniciativa-batuta/batuta-cli/blob/main/README.md#planned-repository-layout),
  creating only packages that have code in this issue.
- Add a `Makefile` with `build`, `test`, `vet`, `fmt`, and `clean` targets.
- Document the exact Go version in `go.mod` and in the README development section.
- Ensure `CGO_ENABLED=0` builds succeed for the default target.

## Out of scope

- Domain types, HTTP server, SQLite, CLI client commands, worker runtime.
- CI provider configuration.
- Release automation.
- Choosing a CLI framework; use the standard library command tree for now.

## Acceptance criteria

- [ ] `go build ./...` succeeds from a clean checkout.
- [ ] `go test ./...` succeeds.
- [ ] `go vet ./...` succeeds.
- [ ] `go run ./cmd/batuta version` prints version, commit, and build date.
- [ ] Builds work with `CGO_ENABLED=0`.
- [ ] No empty packages, placeholder interfaces, or speculative abstractions exist.
- [ ] The README explains the local build and test commands.

## Dependencies

- Blocked by: none.
- Blocks: every implementation issue.

## Technical notes

- Use the latest stable Go release available at implementation time and record it in
  `go.mod`; do not guess a version in documentation.
- Inject build metadata with `-ldflags "-X ..."`.
- Keep `main.go` limited to wiring. Business logic belongs in `internal/`.
- Do not add a CLI framework yet. Revisit only if the command tree becomes hard to
  maintain.

## Validation

```console
$ go build ./...
$ go test ./...
$ go vet ./...
$ go run ./cmd/batuta version
$ CGO_ENABLED=0 go build ./...
```
