# LogPocket — Detailed Development Plan

> **Status:** Planning, no application code implemented by these documents
> **Date:** 2026-10-08
> **Architecture:** [`architecture.md`](architecture.md)
> **Implementation companion:** [`IMPLEMENTATION_GUIDE.md`](IMPLEMENTATION_GUIDE.md)
> **First release goal:** publicly usable v0.1.0, locally scoped single-host Docker logs

## 1. Approach and schedule

**Recommended approach:** build one vertical slice at a time (real source → normalize → commit → query → UI), not "all backend first, then all frontend". Publish public issues and accept incremental OSS contributions once basic project conventions are in place.

**Effort estimate for a solo learner, part-time:** approximately **8–12 weeks**, assuming roughly 8–12 focused hours weekly and no major compatibility blockers. More realistic than a 7-day sprint for a secure, tested, usable release; not a promise or deadline. Track actual effort and revise.

| Phase | Working milestone | Rough effort | Public outcome |
|---|---|---|---|
| 0 | Discovery, stack spikes, repo bootstrap | 1 week | Project exists, risk spikes passed |
| 1 | Core domain + SQLite persistence | 1–2 weeks | Save and query sample logs after restart |
| 2 | Docker ingestion MVP | 1–2 weeks | Real containers stream into DB |
| 3 | Search + API + SSE | 1–2 weeks | API provides history, filters and reliable live delivery |
| 4 | Polished UI vertical slice | 1–2 weeks | Logs explorable without terminal |
| 5 | Retention, reliability, security/packaging | 1–2 weeks | Single-run deployment with explicit safety limits |
| 6 | Test, docs, benchmark, beta/release | 1–2 weeks | Tagged v0.1.0 with docs and binaries |

This is a **priority order**, not permission to rush a milestone with failing quality gates. Some UI/API work should begin earlier on the same vertical slice.

---

## 2. Non-negotiable release acceptance criteria

- [ ] README quickstart works on a clean supported host (without modifying global Docker socket permissions).
- [ ] Explicit Docker collector enablement is clear, warns about daemon privilege, and works on a documented supported configuration.
- [ ] Sources are correctly identified by IDs, including container recreate behavior.
- [ ] Plain text and common structured JSON messages are ingested and normalized.
- [ ] History persists after restarting LogPocket.
- [ ] Search by text, source, level and time works with pagination.
- [ ] Streaming only publishes committed messages, and reconnect can restore from DB without a fetch/SSE race.
- [ ] UI is usable on desktop and mobile, with meaningful empty/error/loading states.
- [ ] Retention cleans log data and FTS index; size controls are honestly documented as soft limits.
- [ ] Queue sizes, line sizes, query sizes and follower count are bounded.
- [ ] Failures are visible: collector offline, unsupported driver, storage full, lag and possible gaps.
- [ ] CI green (Go unit/integration/race, frontend tests/build, Docker smoke, lint/security checks).
- [ ] LICENSE, CONTRIBUTING, SECURITY, architecture, configuration and troubleshooting docs published.
- [ ] Tagged binaries/images/checksums, demo walkthrough and known limitations documented.

---

## 3. Phase 0 — Validate risky assumptions and establish the repo

**Goal:** build confidence in the highest-risk dependencies before constructing the product. Do not write fancy abstractions yet.

### Tasks

- [ ] **LP-001** Create repo: license Apache-2.0, README, architecture, roadmap, `.editorconfig`, `.gitignore`, GitHub issue/PR templates, CODE_OF_CONDUCT and SECURITY.
- [ ] **LP-002** Install/verify Go 1.27.x, Node LTS and npm/pnpm (pick one and commit lockfile), Docker on macOS arm64/Linux. Record `go version`, `node --version`, Docker API version.
- [ ] **LP-003** Build Go hello-world `logpocket serve` using `net/http` and graceful SIGTERM/SIGINT handling; startup health route.
- [ ] **LP-004** Build Svelte 5/Vite skeleton; configure Vite output to `internal/web/dist` (or deterministic copy), then test `go:embed` integration and correct SPA fallback without accidentally serving API routes as HTML.
- [ ] **LP-005** **Critical SQLite spike:** verify the selected Go SQLite driver actually supports FTS5, external-content triggers, WAL, JSON handling, database/sql connection pooling, binary builds Linux/macOS; compare with ncruces/go-sqlite3 if needed.
- [ ] **LP-006** **Critical Docker spike:** programmatically list Docker containers, stream a test container's stdout/stderr, decode TTY versus non-TTY, record Docker Engine log-driver compatibility and socket permissions on macOS.
- [ ] **LP-007** Basic GitHub Actions: Go test, `go vet`, frontend typecheck/test/build; branch protection recommendation.
- [ ] **LP-008** Create `docs/adr` entries for backend, embedded DB, SSE, Docker trust boundary; publish project board and labels `good first issue`, `bug`, `docs`, `enhancement`.

### Deliverable

A reproducible tiny binary that serves an embedded Svelte page, reads a Docker demo container and writes/searches a few rows in a separate test DB. **Spike code may be discarded/refactored.**

### Gate

FTS5 works and a Docker log stream can be decoded on the intended reference machine. If not, choose/benchmark an alternative **now**, not midway through the UI.

### Learn

Go modules, Go stdlib HTTP, embed, SQLite, CGO vs pure-Go drivers, Docker Engine API, Svelte/Vite.

---

## 4. Phase 1 — Domain, storage and ingestion skeleton

**Goal:** robust storage and correctness independent of Docker; use a deterministic in-memory sample producer first.

### Tasks

- [ ] **LP-009** Define `Source`, `RawRecord`, `LogEvent`, `QueryFilter` and typed error codes (clean domain package; no SQL/Docker imports).
- [ ] **LP-010** Create initial migrations for `sources`, `logs`, `source_checkpoints`, `schema_migrations`, FTS5 and insert/delete triggers.
- [ ] **LP-011** Implement DB init: WAL, pragma assertions, busy timeout/foreign keys per connection, schema version, data dir permissions.
- [ ] **LP-012** Implement single DB writer that batches ≤250 events / 100 ms and atomically writes events + checkpoint updates.
- [ ] **LP-013** Add test generator that emits known INFO/WARN/ERROR plain-text, JSON, null/invalid timestamp, UTF-8 and large payload fixtures.
- [ ] **LP-014** Normalize timestamps/levels/raw JSON; preserve original sanitized line and flag invalid/truncated cases; set `UNKNOWN` when appropriate.
- [ ] **LP-015** Implement query service with source/severity/time filters, time+ID keyset cursor, cancellable bounded queries.
- [ ] **LP-016** DB integration tests for FTS insert/search/delete, rollback correctness, WAL read concurrency, restart persistence, huge input bounds.

### Deliverable

A Go CLI/test harness ingests synthetic events, stops, restarts, and searches them via a query service.

### Gate

No live event is exposed pre-commit; FTS remains consistent after deletion and transaction rollback. DB initialization works from an empty data directory and an existing one.

### Learn

Migrations, WAL, indexes, FTS5, Go concurrency, bounded memory, SQL query plans, test fixtures.

---

## 5. Phase 2 — Docker collector and restart semantics

**Goal:** collect logs from actual local containers and make incomplete coverage visible.

### Tasks

- [ ] **LP-017** Implement Docker daemon detect/ping/version and enumerate containers with kind/name/labels/driver health.
- [ ] **LP-018** Per-container follower goroutines; decode stdout/stderr multiplex vs TTY combined streams, line framing and caps.
- [ ] **LP-019** Subscribe to Docker lifecycle events and handle containers created/stopped/restarted/recreated.
- [ ] **LP-020** Persist source checkpoints in the event insert transaction; implement short timestamp overlap on reconnect and clear best-effort duplicate policy.
- [ ] **LP-021** Backpressure: bounded channels **by both record count and aggregate payload bytes**, blocked-reader monitoring, source failure/retry state, capped exponential backoff/jitter.
- [ ] **LP-022** Support initial tail limit (500), max follower budget and graceful shutdown/drain; no default all-history backfill.
- [ ] **LP-023** Tests with fake Docker HTTP API and real Docker demo, including daemon outage, TTY, non-TTY, empty lines, partial lines and unsupported drivers.

### Deliverable

`logpocket serve --docker-enabled` shows actual Docker container logs collected into SQLite, then retrievable after application restart.

### Gate

Docker daemon outage cannot crash the UI/database. Unsupported drivers and possible gaps are visible, and checkpoints never advance beyond committed logs.

### Learn

Docker APIs, goroutines, streaming protocol demultiplex, state machines, context cancellation, replay semantics.

---

## 6. Phase 3 — HTTP search and committed-log streaming

**Goal:** a stable foundation for the browser and future clients.

### Tasks

- [ ] **LP-024** Versioned routes: `/api/v1/status`, `/sources`, `/logs`, `/logs/{id}`, `/stats`, `/capabilities`, `/healthz`, `/readyz`.
- [ ] **LP-025** Strict query validator, escaped FTS search, bound SQL params, source/level whitelisting, request timeouts, max page size 500.
- [ ] **LP-026** Implement stable keyset pagination with opaque cursor tied to filters (invalid/stale cursor yields a documented error, not SQL exception).
- [ ] **LP-027** SSE after-commit event hub with bounded subscriber channels, 15s heartbeats and cleanup on client disconnect.
- [ ] **LP-028** Implement DB catch-up handoff from last ID to live stream without missed commit race; cover `Last-Event-ID`, retention gap and resync behavior.
- [ ] **LP-029** Unit/integration test HTTP status, malformed filter, XSS-like text, FTS special input, stream reconnect/slow consumer, concurrent inserts and queries.
- [ ] **LP-030** Write OpenAPI v3 contract manually or generate it from checked-in contract after routes stabilize; keep generated types optional.

### Deliverable

`curl` can list containers, query persisted logs and subscribe to a durable-resumable live feed.

### Gate

During a repeatable test with concurrent inserts and reconnects, client eventually sees all retained committed IDs without silent gap. API has stable error codes and bounded queries.

### Learn

API design, streaming, SSE, keyset cursors, race conditions, consistency semantics, API versioning.

---

## 7. Phase 4 — Build the actual developer experience

**Goal:** someone who doesn't know Go or SQL can diagnose a container failure in a few clicks.

### Tasks

- [ ] **LP-031** Design tokens (spacing/color/typography), responsive shell, source sidebar and header with health summary.
- [ ] **LP-032** Log Explorer table with timestamp, severity, source, message, and a bounded/virtualized scroll region.
- [ ] **LP-033** Search bar, source multi-select, severity chips, predefined time ranges, loading/empty/error UX, URL-persisted filters.
- [ ] **LP-034** SSE live/pause/reconnect; auto-scroll policy with "new logs" badge and source disconnected notices.
- [ ] **LP-035** Log detail drawer with raw line (escaped text), formatted JSON, metadata, copy action, precise timestamp/timezone display.
- [ ] **LP-036** Sources and Settings/Status read-only views; warnings for unsupported driver, storage pressure, queue blocking and Docker privilege.
- [ ] **LP-037** Keyboard/focus accessibility, dark/light theme (system default), mobile responsive tests; avoid animations/large UI framework bundles.
- [ ] **LP-038** Vitest + Playwright tests with deterministic mock API and real local app smoke test; screenshots for README.

### Deliverable

An integrated UI lets users view live and old logs, filter by type/source/time and inspect raw JSON on laptop and phone.

### Gate

Two distinct users unfamiliar with the project can diagnose a deliberately broken demo container without terminal instructions after installation.

### Learn

Svelte 5, frontend streaming, accessibility, virtualized lists, client cache and cross-layer integration.

---

## 8. Phase 5 — Operations and privacy-conscious packaging

**Goal:** turn a development prototype into a reasonably safe local-use OSS tool.

### Tasks

- [ ] **LP-039** Implement time retention by `ingested_at_us`, bounded delete transactions and FTS cleanup; monitor DB/WAL file sizes.
- [ ] **LP-040** Implement 2 GiB soft storage budget, pressure warnings, pause behavior when full; exercise WAL checkpoint and page reuse.
- [ ] **LP-041** Config loader: validation, CLI > env > file > defaults, safe file permissions; `version`, `config check` commands.
- [ ] **LP-042** Graceful signal handling, queue drain deadline, shutdown tests, predictable exit codes and useful logs.
- [ ] **LP-043** Build Docker image and Compose example: loopback-published port, persistent volume, non-root where feasible, explicit socket risk and optional trusted proxy.
- [ ] **LP-044** Produce Linux amd64/arm64 and macOS arm64 binaries; check UI embedded assets and SQLite FTS work in release artifacts.
- [ ] **LP-045** Write installation, configuration, limits, security, Docker compatibility and troubleshooting docs; security reporting policy.
- [ ] **LP-046** Add failure tests: disk full, read-only directory, corrupted DB, unsupported log drivers, Docker restart and full storage.

### Deliverable

A fresh-machine install that survives restarts, retains the configured period, and explains its limitations clearly.

### Gate

All security warnings are prominent; nothing binds to the public network by default. Disk pressure does not silently advance checkpoints or claim collection is healthy.

### Learn

Service packaging, platform differences, release engineering, defensive design, fault injection, operator docs.

---

## 9. Phase 6 — Release quality and open-source launch

**Goal:** v0.1 users can trust basic functionality and easily report/contribute improvements.

### Tasks

- [ ] **LP-047** Benchmark scripts: 5–20 containers, 500 logs/s for 10 min, burst tests, 100k stored rows, realistic search queries; publish machine specs/results.
- [ ] **LP-048** Run `go test ./...`, `go test -race ./...`, `go vet ./...`, static checks; frontend typecheck/Vitest/Playwright and Docker smoke.
- [ ] **LP-049** External peer review of Docker socket security and sensitive-log behavior; ensure no telemetry/network calls other than required local sources.
- [ ] **LP-050** Check dependency licenses, SBOM/checksums (as feasible), reproducible build inputs, and release artifact integrity.
- [ ] **LP-051** Release candidate `v0.1.0-rc.1`, run Linux amd64/arm64 and macOS arm64 fresh-install smoke tests; fix blockers.
- [ ] **LP-052** Tag `v0.1.0`, publish changelog, supported platforms, limitations, screenshots/GIF, demo compose and getting-started instructions.
- [ ] **LP-053** Post project in relevant communities with accurate limitations and ask for usability feedback, not stars; label actionable issues for contributors.

### Deliverable

Versioned GitHub release with source, portable binaries, multiarch container, license, CI, documentation, demo, known issues and contributor guide.

### Gate

A stranger can install it, use it and remove it safely; tests pass on supported platforms and the readme does not promise capabilities we don't have.

### Learn

Performance evaluation, benchmarking honesty, security review, maintainer practice, OSS release management.

---

## 10. First 10 GitHub issues to open (sorted)

These are deliberately small enough to let someone complete and review each PR.

| Priority | Issue title | Size | Acceptance |
|---|---|---|---|
| P0 | `chore: bootstrap Go CLI and HTTP health endpoint` | S | One binary serves `/healthz`, exits on SIGINT |
| P0 | `chore: bootstrap Svelte frontend and embed into Go binary` | M | Browser shows app shell from built binary |
| P0 | `spike: verify FTS5 and WAL in selected pure-Go SQLite driver` | M | Automated test compiles on Linux/macOS |
| P0 | `spike: stream Docker logs and demux stdout/stderr` | M | Test container parsed correctly, TTY noted |
| P0 | `feat(storage): add migrations for sources, logs, FTS5` | M | Migration tests green; FTS delete consistency |
| P0 | `feat(ingest): normalize JSON and plain text logs` | M | Known fixtures pass |
| P0 | `feat(storage): atomic log batch and source checkpoint` | M | Rollback doesn't advance checkpoint |
| P1 | `feat(api): source and historical log endpoints` | M | Filtered, bounded, paginated responses |
| P1 | `feat(collector): discover and follow Docker containers` | L | Stop/restart/recreate test flows pass |
| P1 | `feat(ui): source sidebar and searchable log explorer` | L | Can locate known error from UI |

Use GitHub milestones for M0–M6. Put `good first issue` labels on bounded improvements *after* package APIs are stable (docs, parsers with fixtures, accessibility, tests)—not on hard Docker privilege or recovery semantics.

---

## 11. Development workflow and contribution standards

### Branch and PR workflow

- `main`: protected and always releasable; no direct unreviewed pushes after community grows.
- Feature branches: `feat/<short-topic>`, fixes: `fix/<short-topic>`, docs: `docs/<short-topic>`.
- Small commits that explain engineering changes, not commit-count padding.
- Conventional commit style is recommended (`feat:`, `fix:`, `docs:`, `test:`, `refactor:`); automated release notes are optional early on.
- One issue/PR should have tests and docs changes appropriate to scope; link the issue in the PR.

### Pull request checklist

- [ ] Clearly states user problem, approach, alternatives.
- [ ] No unnecessary runtime dependencies; SQL/FTS calls use bound params.
- [ ] Adds tests for new behavior and failure cases.
- [ ] Maintains bounds on memory/queue/request sizes.
- [ ] Does not expose Docker socket or logs insecurely.
- [ ] Adds docs/API contract/ADR for externally visible changes.
- [ ] CI passes; screenshots for UI changes; performance numbers for hot paths.

### Suggested local developer commands (targets to implement)

```bash
# backend
go run ./cmd/logpocket serve --config ./configs/logpocket.example.yaml
go test ./...
go test -race ./...
go vet ./...

# frontend
cd web
npm ci
npm run dev
npm run check
npm test
npm run build

# later: make dev, make test, make smoke, make release-dry-run
```

**Do not copy these commands blindly before scripts exist**; implement `package.json` scripts and configuration during Phase 0.

---

## 12. MVP demo scenario — our regression story

This is the project’s north-star acceptance demonstration:

1. Start a demo Docker Compose stack with `order`, `payment`, `redis` simulator logs.
2. Start LogPocket locally with Docker collector explicitly enabled.
3. Select only `payment-service` in the browser.
4. Observe live INFO messages and a simulated timeout ERROR.
5. Pause live feed, search `timeout`, filter `ERROR`, open row detail to see JSON attributes.
6. Stop LogPocket; generate additional logs while stopped.
7. Restart LogPocket; confirm old history remains and source resumes with gap/duplicate behavior honestly reported.
8. Open a second tab; disconnect and reconnect its SSE stream, confirm catch-up with no silent missing committed IDs.
9. Set short retention in demo; confirm expired events disappear from search and FTS.
10. Turn off Docker daemon: UI remains available and explicitly reports collection degraded.

If any core step fails, it is a release blocker or clearly documented unsupported operating mode.

---

## 13. Major risks and mitigation priority

| Risk | Likelihood/impact | Early action |
|---|---|---|
| Pure-Go SQLite + FTS5 compatibility/performance | Medium / high | Week-1 driver spike with real FTS deletes, WAL, binaries |
| Docker socket privileges on different OSes | High / high | Threat model, local-only default, test restricted proxy and permissions |
| Docker log replay no stable per-line offset | High / medium | Acknowledge possible duplicates/gaps; integration tests and clear counters |
| FTS storage bloat and retention cost | Medium / medium | Reproducible 100k/1M log benchmark before claiming targets |
| SSE races at history-live transition | Medium / high | Define/verify atomic handoff + disconnect test before UI polish |
| Huge logs exhaust memory/browser | Medium / high | Fixed channel/line/page limits, virtualized UI, slow subscriber policy |
| Scope creep into mini-Grafana | High / high | MVP non-goals, contributions follow milestones, label post-MVP proposals |
| Design/UX looks like internal demo | Medium / medium | Design system, real-world demo, screenshot and usability release gate |

---

## 14. After v0.1 — learning-oriented OSS roadmap

### v0.2: Broader sources (Go + system design)

- File tailer with persisted inode/offset; rotation and symlink handling.
- HTTP JSONL ingestion with API key, quotas and import diagnostics.
- Export and basic redaction rules; SQLite doctor/backup.

### v0.3: Operational value (database/search learning)

- Error grouping using stacktrace fingerprint.
- Alert rules and event-rate windows.
- Syslog/journald adapters, regex/structured queries if requested.

### v0.4: Multi-host learning (networking/security)

- Optional agent with mTLS/API tokens, local spool and retry protocol.
- Remote source identity and source clock-skew handling.
- Chaos tests for partitions, collector outages and disk-full states.

### v0.5+: Research experiments (optional Rust / storage)

- Benchmark Rust tokenizer/parser as optional module: choose by measured gain.
- Parquet archival and compact indexes, if SQLite scale becomes painful.
- OpenTelemetry log ingest, optional local-LLM-assisted summarization (privacy default off).

**Admission rule:** A new feature is accepted because it solves a documented user issue, not because we want a trendy technology in the stack.

---

## 15. How to learn while building

| Topic | Build task | Learning depth |
|---|---|---|
| Go fundamentals | CLI, contexts, interfaces and goroutines | Syntax through real concurrency |
| Go performance | bounded queues, batches, benchmarks and profiling | CPU/memory tradeoffs |
| SQLite internals | WAL, checkpoints, FTS, indexes, query plan, retention | Database mechanics |
| Protocols | Docker demultiplexing and SSE, reconnection | Real streaming and failure semantics |
| Svelte/TypeScript | reactive log browser, filters, virtual list | Modern frontend beyond React/Angular |
| Security | Docker socket, file permissions, log confidentiality | Threat modeling |
| Release engineering | multiarch builds, Go embedding, CI, checksums | Distribution and DevOps |
| OSS leadership | issues, docs, peer review, versioning | Maintenance and community |

Suggested personal rule: write a tiny `docs/learning-notes.md` entry only when something surprising/important changes an engineering decision. This is optional; it must not become burdensome bookkeeping.

---

## 16. Release success metrics (not GitHub stars)

Evaluate v0.1 after real users try it:

- A first-time user can install, connect Docker, search for known error and inspect detail using docs alone.
- A crash/restart preserves prior committed logs; reconnect recovery limitations are documented/visible.
- The app remains responsive under documented test load.
- One week after release: issues distinguish bugs from product requests; each bug includes repro steps.
- Contributors can run tests and make a documentation/parser UI improvement without asking you for private infrastructure.
- You personally use LogPocket to debug another project at least once—its usefulness is validated by actual use.

---

## 17. Definition of ready for each task

Before starting an issue, it must have:

1. User problem or engineering invariant being addressed.
2. Clear input/output or API/database impact.
3. Acceptance criteria including at least one failure case.
4. Dependencies/milestone and approximate size.
5. Security/performance/documentation concerns if applicable.

This keeps GitHub progress credible and makes the project friendlier to other contributors.

**Recommended immediate implementation order:** LP-001 → LP-003 → LP-004 → LP-005 → LP-006 → LP-010 → LP-012 → LP-017. The first valuable vertical slice is **Docker container → persistent SQLite row → search result → browser log line**; build and demo it as soon as these pieces are sufficient.
