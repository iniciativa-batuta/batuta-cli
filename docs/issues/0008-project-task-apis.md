# Implement project and task APIs with idempotent apply

## Context

Users need to create projects and submit task definitions before any scheduling or
execution exists. The API must validate definitions deterministically and avoid
duplicate tasks when the same file is applied twice.

## Objective

Expose project and task creation, listing, inspection, and idempotent task application.

## Scope

- Add project endpoints:
  - `POST /v1/projects`
  - `GET /v1/projects`
  - `GET /v1/projects/{id}`
- Add task endpoints:
  - `POST /v1/projects/{project}/tasks`
  - `GET /v1/projects/{project}/tasks`
  - `GET /v1/tasks/{id}`
  - `GET /v1/tasks/{id}/executions`
- Define the task apply request and response contracts, including `created` vs
  `unchanged`.
- Normalize the YAML task definition to the canonical JSON spec and compute its hash.
- Validate the spec with the domain package, including step graph validation.
- Apply the idempotency rule from
  [decisions.md](https://github.com/iniciativa-batuta/batuta-cli/blob/main/docs/architecture/decisions.md).
- Add pagination with `limit` and `cursor` for list endpoints.
- Add tests for validation errors, idempotency, conflicts, pagination, and not-found
  cases.

## Out of scope

- Scheduling and execution creation.
- Task update or revision history.
- Task cancellation (covered with execution cancellation).
- YAML parsing on the server; the CLI converts YAML to the canonical request.

## Acceptance criteria

- [ ] A project can be created with a unique URL-safe name.
- [ ] Duplicate project names return `409` with a stable code.
- [ ] A valid task is stored with a normalized spec and spec hash.
- [ ] Applying the same project + name + spec hash returns the existing task with
      `created=false`.
- [ ] Applying a different spec to an existing name returns `409` unless update mode is
      explicitly supported later.
- [ ] Invalid step graphs return field-level validation errors.
- [ ] Unknown project or task returns `404` with a stable code.
- [ ] List endpoints are bounded and support cursor pagination.
- [ ] No task can reference a project other than the path project.

## Dependencies

- Blocked by: #5, #6, #7.
- Blocks: CLI task commands, scheduler, execution views.

## Technical notes

- The CLI is responsible for reading YAML and sending a canonical JSON request. This
  keeps the server independent of YAML formatting and comments.
- Normalize defaults before hashing so semantically identical specs produce the same
  hash.
- Reject duplicate keys and unknown fields in the task document.
- Do not start an execution on task creation. Scheduling is a separate concern.
- Pagination cursors must be opaque and validated; do not expose raw offsets as a
  contract.

## Validation

```console
$ go test ./internal/httpapi/...
$ go test ./internal/app/...
$ go test ./... -run TaskAPI
```
