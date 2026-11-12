# Add an optional web interface

## Context

The CLI is the primary interface. A web UI can improve visibility for teams, but it must
not become a second backend or bypass the API's authorization rules.

## Objective

Provide an optional, minimal web interface that consumes the same public API as the CLI.

## Scope

- Add a web command or configuration flag that is disabled by default.
- Provide pages for:
  - Projects and tasks.
  - Task detail with execution and step states.
  - Observation timestamp and stale indicator.
  - Cancel action, subject to the same authorization as the API.
  - Workers and liveness.
- Use the public API for every read and write.
- Reuse the API error envelope and show actionable errors.
- Add basic accessibility: keyboard navigation, labels, sufficient contrast, no
  color-only state.
- Add end-to-end tests against a running API.
- Document the deployment and authentication model.

## Out of scope

- A real-time streaming protocol.
- A SPA framework or a separate Node.js build pipeline unless justified by a concrete
  requirement.
- Direct database, shell, or worker access.
- Multi-user accounts, RBAC, or SSO.
- Editing task definitions in the browser.

## Acceptance criteria

- [ ] The web interface is disabled by default.
- [ ] Every request goes through the public API; no direct database or worker access.
- [ ] The task view shows step states, observation time, and staleness, never a
      percentage.
- [ ] Cancellation uses the same authorization and audit path as the CLI.
- [ ] An unauthenticated user cannot view or mutate data.
- [ ] The interface works without JavaScript for the read-only views, or the limitation
      is documented and justified.
- [ ] Accessibility checks pass for the implemented pages.
- [ ] The CLI workflow remains complete with the web interface disabled.

## Dependencies

- Blocked by: #14, #20.
- Blocks: none; this is optional.

## Technical notes

- Prefer server-rendered Go templates over a separate frontend toolchain for the MVP.
- Do not add a second authorization model. Reuse the API token flow.
- Do not expose internal IDs as secrets; they are opaque but not credentials.
- If a browser token is stored, define its scope, lifetime, and logout behavior.
- Keep the web package at the edge; it must not be imported by `internal/app` or
  `internal/domain`.

## Validation

```console
$ go test ./internal/web/...
$ go test ./test/e2e/... -run Web
```
