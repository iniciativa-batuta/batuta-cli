# Architecture decisions

These are the decisions that shape the MVP. They are recorded here instead of a formal
ADR bureaucracy. Each decision states the recommendation, the alternatives considered,
the trade-off, and whether it can be deferred.

Status values: **accepted**, **provisional** (revisit after the MVP), **deferred**.

## D1 — One binary with subcommands

**Status:** accepted

**Decision:** Ship one `batuta` binary with `server`, `worker`, and client subcommands.

**Alternatives:** Separate `batuta-server`, `batuta-cli`, and `batuta-worker` binaries.

**Trade-off:** A single binary simplifies installation and version alignment. It also
means the worker runtime and the central service share a module and dependency set,
which can grow if adapters are added. The processes remain independently deployable and
are configured separately. Split later only if binary size, privilege separation, or
release cadence justifies it.

**MVP impact:** `batuta server start`, `batuta worker run`, and client commands are
different processes from the same artifact.

## D2 — Long polling for work acquisition

**Status:** accepted

**Decision:** Workers use HTTP long polling with a configurable wait and jitter.

**Alternatives:** Short polling; WebSocket; server-sent events; a message broker.

**Trade-off:** Long polling works through proxies and NAT, is simple to operate, and
needs no extra infrastructure. It holds a connection per worker and has slightly higher
latency than a push channel. Jitter and backoff prevent thundering herds.

**MVP impact:** One open channel per worker runtime. The implementation must not
tight-loop when no work is available.

## D3 — Channel-based snapshot requests, optional direct pull

**Status:** accepted

**Decision:** The central service requests snapshots by delivering a control request on
the worker's outbound channel. When a worker advertises a reachable address, the central
service may optionally pull the same snapshot directly.

**Alternatives:** Require inbound connectivity to every worker; have the worker push
snapshots periodically; use a reverse tunnel or a broker.

**Trade-off:** The channel path works behind NAT and restricted egress and keeps
observation request-driven. It adds a control-request queue and request correlation. The
direct path reduces latency in LANs but cannot be the only path.

**MVP impact:** Channel mode is mandatory. Direct mode is opt-in and may be deferred.

## D4 — Central service stores observed snapshots, not step authority

**Status:** accepted

**Decision:** The central service stores the last accepted snapshot per execution with
`observed_at`, `sequence`, and a staleness flag. It does not treat that copy as a live
step state.

**Alternatives:** The central service owns all step state and the worker reports every
transition; the central service keeps no snapshot and always queries on demand.

**Trade-off:** The stored snapshot makes the CLI fast and supports auditing, but it can
be stale. The API must expose the observation time and stale flag. Always querying on
demand would be simpler but slower and would make history invisible.

**MVP impact:** Execution finalization requires a terminal snapshot or an explicit
timeout/loss rule.

## D5 — Heartbeat is liveness only

**Status:** accepted

**Decision:** Heartbeats carry worker liveness and load hints only. They never carry step
state or step transitions.

**Alternatives:** Include step deltas in heartbeats; use heartbeats as a progress
notification channel.

**Trade-off:** A separate snapshot request costs one extra round trip. It keeps the
ownership boundary honest and avoids a second, competing state channel.

**MVP impact:** Offline detection uses the heartbeat window. Step progress uses
snapshots.

## D6 — Worker-local state in SQLite

**Status:** accepted

**Decision:** The worker persists execution plans, step observations, and sequence
numbers in a local SQLite database under its state directory.

**Alternatives:** In-memory state with a JSON journal; bbolt; a remote state store.

**Trade-off:** SQLite provides transactions, durability, and queryability with the same
driver family used centrally. It adds a file and a migration path on every worker. JSON
journals are simpler but error-prone under concurrent updates and restarts.

**MVP impact:** A worker restart can reconcile state without losing completed steps.

## D7 — Explicit ownership split

**Status:** accepted

**Decision:** The central service owns Task and Execution state and the execution plan.
The worker owns the runtime state of its local steps. The central service observes that
state through snapshots.

**Alternatives:** Central ownership of every step transition; worker ownership of the
task lifecycle.

**Trade-off:** The split requires a protocol and staleness handling, but it matches the
physical reality of remote execution. Neither side claims data it cannot guarantee.

**MVP impact:** The worker cannot reorder or redefine steps. The central service cannot
fabricate step transitions.

## D8 — Continuity policy at task level

**Status:** accepted

**Decision:** Configure `FAIL_FAST`, `CONTINUE_ON_ERROR`, or `BEST_EFFORT` once per task.

**Alternatives:** Per-execution override; per-step override; a full policy expression
language.

**Trade-off:** One point of configuration is predictable and testable. Some workflows
may want a stricter step inside a lenient task; that can be added later with a concrete
use case.

**MVP impact:** The execution plan copies the task policy. Retries reuse the task
policy.

## D9 — Sequential execution with dependency-aware selection

**Status:** accepted

**Decision:** The MVP runs one step at a time. The runner still honors `dependsOn` and
may choose the next ready step under `BEST_EFFORT`.

**Alternatives:** Parallel steps from day one; a general DAG scheduler.

**Trade-off:** Sequential execution is easier to reason about, test, and debug. It
underuses machines with spare capacity and makes `BEST_EFFORT` less distinct. Parallelism
is a later, explicit feature.

**MVP impact:** No fan-out, no dynamic steps, no per-step concurrency.

## D10 — Structured process executor; explicit shell

**Status:** accepted

**Decision:** The default executor takes `executable`, `args`, `cwd`, `env`, and timeout.
Shell execution is a separate executor, disabled by default.

**Alternatives:** Shell strings everywhere; a template language; container-only execution.

**Trade-off:** Structured execution prevents injection and makes audit easier. It is
less convenient for ad-hoc pipelines. Shell remains available when explicitly enabled.

**MVP impact:** No string interpolation into commands. Working directories are restricted
to configured roots. Environment secrets are resolved from the worker environment.

## D11 — Inline payloads are small; artifacts are referenced

**Status:** accepted

**Decision:** Inline payloads have a small size limit. Larger results are uploaded as
artifacts with size, media type, and SHA-256, and referenced by `artifact://` URI.

**Alternatives:** Put everything in snapshots; use only external object storage.

**Trade-off:** Bounded snapshots stay fast and safe. Artifact storage adds lifecycle,
quota, and cleanup work. Object storage is deferred.

**MVP impact:** Oversized inline payloads are rejected with a clear error. Artifact
checksums are verified.

## D12 — SQLite with WAL and embedded migrations

**Status:** accepted

**Decision:** Use SQLite in WAL mode, foreign keys enabled, a busy timeout, and
migrations embedded in the binary.

**Alternatives:** PostgreSQL from day one; an external migration tool; an ORM.

**Trade-off:** SQLite is a single file, easy to back up, and sufficient for the MVP. It
limits write concurrency and multi-node central services. PostgreSQL remains a future
adapter if a real requirement appears.

**MVP impact:** One central process owns the database file. Tests use temporary
databases.

## D13 — Bearer credentials over TLS; mTLS deferred

**Status:** provisional

**Decision:** Admin and worker requests use bearer credentials. TLS is required outside
loopback. Worker credentials are scoped to the worker and stored hashed centrally.

**Alternatives:** mTLS; OAuth/OIDC; signed requests; no auth on a private network.

**Trade-off:** Bearer tokens are simple and work with the CLI and long polling. They
require careful storage and revocation. mTLS adds stronger binding but complicates
enrollment and local development.

**MVP impact:** One admin token and per-worker credentials. Rotation and revocation are
required. RBAC beyond admin/worker is deferred.

## D14 — Versioned JSON protocol, additive changes

**Status:** accepted

**Decision:** Use JSON envelopes with `protocol_version: "1"` and schema names such as
`batuta.worker.snapshot/v1`. Additive fields are allowed; breaking changes require a new
major version.

**Alternatives:** Protobuf/gRPC; unversioned JSON; form-encoded requests.

**Trade-off:** JSON is inspectable, easy to debug, and works with the standard library.
It is more verbose and has weaker schema enforcement unless validated. Protobuf can be
considered if transport efficiency becomes a demonstrated problem.

**MVP impact:** Unknown fields are ignored. Unsupported majors are rejected before any
state change.

## D15 — Go package layout by responsibility

**Status:** accepted

**Decision:** Use `cmd/batuta`, `internal/domain`, `internal/app`, `internal/store`,
`internal/httpapi`, `internal/worker`, `internal/executor`, `internal/protocol`,
`internal/config`, and `internal/cli`.

**Alternatives:** Flat package; layered packages named after patterns; a monorepo with
multiple modules.

**Trade-off:** Responsibilities are explicit and the domain stays independent. Too many
packages too early can be noise; packages are created only when they have a real
responsibility.

**MVP impact:** No empty placeholder layers. Dependencies point inward.

## D16 — Installation and updates are manual

**Status:** accepted

**Decision:** Distribute a single binary with checksums. The MVP does not self-update.

**Alternatives:** Auto-update; package manager only; containers only.

**Trade-off:** Manual updates are predictable and avoid a remote code execution channel.
Operators must track releases. Packaging can be added per platform later.

**MVP impact:** `batuta version` reports version, commit, and build date. Release
artifacts include checksums.

## D17 — Web is optional and API-only

**Status:** deferred

**Decision:** A web interface may be added after the API stabilizes. It consumes the same
public API as the CLI, with no direct database, shell, or worker access.

**Alternatives:** Web-first; a web UI embedded in the worker; direct database access.

**Trade-off:** The CLI remains the source of truth for operations. A web UI can improve
visibility later without becoming a second backend.

**MVP impact:** None. The API must not depend on web-specific behavior.

## D18 — No broker or external queue

**Status:** accepted

**Decision:** Use SQLite and long polling for the MVP. Do not add Redis, RabbitMQ, Kafka,
or a custom broker.

**Alternatives:** A message broker for assignments and events.

**Trade-off:** Fewer moving parts and simpler operations. SQLite is a single writer and
the central service is a single point of failure. A broker may be justified if
multi-node central services or very high worker counts become real requirements.

**MVP impact:** Assignment and control queues live in SQLite with claim/lease semantics.

## D19 — Prefixed ULIDs for identifiers

**Status:** accepted

**Decision:** Use prefixed ULIDs such as `tsk_01JQ...`, `exe_...`, and `wkr_...`.

**Alternatives:** UUIDv4; UUIDv7; auto-increment integers.

**Trade-off:** Prefixed IDs are self-describing, sortable by creation time, and safe in
logs. They are longer than integers and not entirely opaque if a client parses the
timestamp; clients must treat them as opaque.

**MVP impact:** IDs are generated centrally and never reused.

## D20 — No automatic retries in the MVP

**Status:** provisional

**Decision:** Retrying an execution is an explicit action that creates a new execution.
Steps are not retried automatically.

**Alternatives:** Retry failed steps in place; exponential backoff per step; a retry
policy expression.

**Trade-off:** Explicit retries preserve history and avoid hidden duplicate side effects.
They require operator action. Automatic retries need idempotency guarantees that the MVP
does not yet have.

**MVP impact:** Terminal steps are immutable. A new attempt is a new execution.

## D21 — OpenCode and Git are adapters

**Status:** accepted

**Decision:** OpenCode and Git are optional executor/workspace adapters, not domain
concepts. The core must remain usable without them.

**Alternatives:** Make OpenCode the primary execution model; bake Git operations into
the task model.

**Trade-off:** Adapters keep the domain general and testable. They add integration code
and configuration. The MVP defers both.

**MVP impact:** No domain package imports an integration. Adapters are disabled by
default.

## D22 — Task application is idempotent by spec hash

**Status:** provisional

**Decision:** `batuta task apply` uses project + task name + normalized spec hash as the
idempotency key. Applying the same definition returns the existing task. Applying a
different definition to an existing name is rejected as a conflict unless an explicit
update mode is requested.

**Alternatives:** Always create a new task; always overwrite; version every change
automatically.

**Trade-off:** Prevents accidental duplicates and silent drift. It makes edits explicit.
Task revision history is deferred; a future revision model can keep the same idempotency
principle.

**MVP impact:** No silent overwrite of a running task definition.
