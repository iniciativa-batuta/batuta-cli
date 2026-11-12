# Add an optional Git integration adapter

## Context

Many tasks start from a repository and produce a commit, branch, or diff. Git is useful
but must not become a domain concept. The core must work without it.

## Objective

Provide an optional, policy-gated Git adapter for workspace preparation and common
operations.

## Scope

- Add an executor or workspace adapter with explicit operations:
  - `clone`, `fetch`, `checkout`, `commit`, `push`, `diff`.
- Use structured arguments, never shell interpolation.
- Resolve credentials through `envFrom` or a worker-local credential helper; never store
  them in task specs.
- Capture a diff or patch as an artifact.
- Restrict working directories to `allowed_roots`.
- Disable the adapter by default.
- Add tests against a local bare repository.
- Document configuration and failure modes.

## Out of scope

- GitHub/GitLab APIs, pull requests, reviews, checks, or webhooks.
- Managing SSH keys centrally.
- Force-push or destructive history rewrites by default.
- Submodule and LFS support.

## Acceptance criteria

- [ ] The adapter is disabled unless explicitly enabled in worker configuration.
- [ ] Clone, fetch, checkout, commit, push, and diff work against a local bare repo.
- [ ] Arguments are passed without shell interpolation.
- [ ] Credentials do not appear in logs, errors, or artifacts.
- [ ] A diff can be stored as an artifact with a checksum.
- [ ] Disallowed working directories are rejected.
- [ ] The core domain has no Git dependency.

## Dependencies

- Blocked by: #15, #19, #20.
- Blocks: Git-based workflows beyond the MVP.

## Technical notes

- Prefer invoking the `git` binary with explicit args over embedding a Git library. It
  keeps the adapter small and matches operator expectations.
- Set a minimal environment and avoid inheriting unrelated credentials.
- Never run `git` in a directory outside the allowed roots.
- Treat repository content as untrusted; do not execute hooks from cloned repositories
  unless explicitly configured.
- Keep branch and remote names validated.

## Validation

```console
$ go test ./internal/executor/git/...
$ go test ./... -run Git
```
