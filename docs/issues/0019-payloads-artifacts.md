# Implement payload and artifact envelopes with limits

## Context

Execution results can be small text, JSON, files, or large archives. Embedding
everything in snapshots makes them slow and unsafe. The MVP needs a bounded inline
payload and a reference-based artifact path.

## Objective

Implement versioned payload envelopes, inline size limits, artifact upload/download,
checksum verification, and quotas.

## Scope

- Define `batuta.payload/v1` and `batuta.artifact/v1` contracts.
- Enforce `max_inline_payload_bytes` before persistence.
- Add artifact upload:
  - `POST /v1/executions/{id}/artifacts` with name, media type, size, and SHA-256.
  - Stream the body to a content-addressed path; do not buffer the whole file.
- Add artifact metadata endpoints:
  - `GET /v1/artifacts/{id}`
  - `GET /v1/artifacts/{id}/content`
  - `GET /v1/executions/{id}/artifacts`
- Verify SHA-256 on upload and download.
- Add per-execution and global quotas.
- Add CLI `batuta execution artifacts` and `batuta artifact download`.
- Add tests for oversized inline payloads, checksum mismatch, path traversal, quota
  exhaustion, and unauthorized access.

## Out of scope

- S3/GCS/object storage.
- Compression and deduplication beyond content-addressed storage.
- Malware scanning.
- Artifact retention automation.
- Inline binary data in task specs.

## Acceptance criteria

- [ ] An inline payload over the limit is rejected with `PAYLOAD_TOO_LARGE`.
- [ ] Artifact uploads require a declared size and checksum.
- [ ] A checksum mismatch aborts the upload and removes partial data.
- [ ] Artifact names cannot contain path separators or traversal sequences.
- [ ] Artifact download verifies the checksum before streaming.
- [ ] A worker can only upload artifacts for its own executions.
- [ ] A user can list and download artifacts through the CLI.
- [ ] Quotas are enforced before accepting the full body when the declared size is
      already over the limit.

## Dependencies

- Blocked by: #8, #15.
- Blocks: Git/OpenCode adapters, MVP validation, web artifact views.

## Technical notes

- Content-addressed storage keyed by SHA-256 avoids duplicate files and makes integrity
  checks explicit.
- Store metadata in SQLite and bytes on the filesystem under a configured root. Never
  trust a client-provided path.
- Use a temporary file and atomic rename for uploads.
- Bound artifact size and request duration.
- Return `artifact://art_...` URIs in payloads; never return raw filesystem paths.
- Do not put artifact bytes in snapshots.

## Validation

```console
$ go test ./internal/httpapi/... ./internal/store/...
$ go test ./... -run Artifact
$ go test -race ./internal/httpapi/...
```
