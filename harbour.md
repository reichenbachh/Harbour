# Harbor

A self-hosted application platform (a mini Vercel / Fly.io), built from scratch in TypeScript on Node.js.

You run `harbor deploy` in an app folder. Harbor builds it, stores the build, runs it on your own machines, load-balances traffic to it, and shows its logs and metrics. Multiple accounts share one Harbor without seeing each other's apps.

## Why this project

It forces five areas to work together in one real product:

| Area                          | Where it lives in Harbor                                             |
| ----------------------------- | -------------------------------------------------------------------- |
| Identity & permissions        | Accounts, orgs, API tokens, "can this user deploy this app?"         |
| Queuing & parallel processing | Build jobs queued and run in parallel by workers                     |
| Asset management              | Source uploads and every build version stored, with rollback         |
| Load balancing                | Traffic spread across app instances, failover, zero-downtime deploys |
| Observability                 | Logs, metrics and charts for every service and app                   |

---

## Ground rules

**Build yourself:** the proxy, queue, storage, identity, observability, scheduler, CLI, dashboard.

**Allowed off-the-shelf:**

- PostgreSQL (metadata only)
- Docker Engine (only as a sandbox to run builds and apps, via its HTTP API on the Unix socket)
- Node.js built-in modules
- `pg` (Postgres driver), the only runtime dependency
- Dev only: `typescript`, `esbuild`

**Not allowed:** nginx, HAProxy, Express/Fastify (in the gate), http-proxy, Redis, RabbitMQ, Kafka, BullMQ, MinIO, Prometheus, Grafana, Keycloak, Auth0, Clerk, any cloud service.

---

## Engineering principles (Node-specific)

1. **Stream everything.** Never read a whole file or body into memory. Use `stream.pipeline`.
2. **Bound everything.** Every queue, buffer, cache, pool and retry has a maximum.
3. **Never block the event loop.** CPU-heavy work (hashing, compression) goes to `worker_threads`; builds go to child processes / containers.
4. **Cap the heap.** Every service runs with `--max-old-space-size`. A leak should crash loudly.
5. **Observable from line one.** Every service exposes `/metrics` and `/health` and uses the shared logger.
6. **Fail safely.** Every network call has a timeout. Every job can be retried safely (idempotent).
7. **Prove it with tests.** Each feature is done when its exam passes under siege load and chaos, not when it "works once."

### Memory budgets (heap cap per service)

| Service | Heap cap                         |
| ------- | -------------------------------- |
| gate    | 128 MB                           |
| control | 128 MB                           |
| id      | 96 MB                            |
| queue   | 128 MB                           |
| vault   | 128 MB                           |
| builder | 96 MB (builds run in containers) |
| runner  | 64 MB                            |
| scope   | 192 MB                           |

Every service reports `process_resident_memory_bytes`, `heap_used_bytes` and `event_loop_lag_p99_ms`.

---

## Repository layout

```
harbor/
  packages/
    core/            # logger, metrics, http helpers, service client, config, errors
  services/
    id/
    control/
    queue/
    builder/
    vault/
    runner/
    gate/
    scope/
  apps/
    dash/            # dashboard (React + Vite, or server-rendered HTML)
    cli/             # `harbor` command
  tools/
    siege/           # load generator
    chaos/           # failure injection
  infra/
    vm/              # VM setup scripts
    sql/             # Postgres migrations (plain .sql files)
  docs/
    HARBOR.md
    decisions/       # one short file per design decision (ADR)
```

pnpm workspaces. Each service is bundled with esbuild into one JS file.

---

## Phase one services

### 0. @harbor/core (shared package, build first)

- JSON-lines logger: level, time, service, app, request_id, message. Writes to stdout and sends batches to scope through a bounded buffer (drops with a counter when full, never blocks).
- Metrics registry: counters, gauges, histograms. Renders text at `/metrics`.
- Default metrics: RSS, heap, event-loop lag, request count/duration by route and status.
- Request ID: read `x-request-id` or create one; propagate on every outgoing call and queue message.
- Service client: `fetch` wrapper with timeout, retries (safe methods only), internal auth header.
- Config loader from environment variables, with validation at startup.

### 1. harbor-id — Identity & permissions

- Signup, login, logout. Passwords hashed with `crypto.scrypt`. Session cookie (HTTP-only, signed).
- Orgs, memberships with roles `owner` | `member`.
- API tokens for the CLI: create, list, revoke. Stored as hashes only.
- `POST /authorize` → `{ allowed: boolean }` for `(token|session, action, resource)`. Every service calls this.
- Phase one actions: `app:create`, `app:read`, `app:deploy`, `app:delete`, `logs:read`.
- _Later:_ OIDC provider, 2FA, teams, policy language, key rotation.

### 2. harbor-control — Control plane

- Public API for the CLI and dashboard.
- Owns apps, releases, deployments, machines, instances.
- Orchestrates a deploy (see flow below).
- Scheduling: place each instance on the healthy machine with the fewest instances; spread copies of one app across machines.
- Marks a machine dead after 3 missed heartbeats and reschedules its instances.
- Pushes the route table to the gate whenever instances change.

### 3. harbor-queue — Queuing

- Single broker, data on local disk.
- Topics with append-only segment files. Record format: `length (4 bytes) | crc32 (4 bytes) | payload (JSON)`.
- Operations: publish, receive (long-poll), ack, nack.
- Visibility timeout: received-but-unacked messages reappear after N seconds.
- Retry with exponential backoff; dead-letter topic after 5 attempts.
- fsync before acknowledging a publish (batch fsyncs for throughput).
- On startup: scan segments, skip/truncate a corrupt tail record, rebuild in-memory state.
- _Later:_ replication and failover, consumer groups, tenant fairness, compaction.

### 4. harbor-builder — Parallel processing

- Worker that consumes `build.requested`.
- Runs up to `BUILD_CONCURRENCY` builds at once (default: CPU cores − 1).
- Each build: download source from vault → run container (`npm ci && npm run build`) → package output → upload to vault → publish `build.succeeded` or `build.failed`.
- Streams build logs to scope tagged with `release_id`.
- Hard timeout per build; always removes the container and temp files, including on crash.
- Idempotent: if a release already has a build artifact, ack and skip.
- _Later:_ build cache, cancellation, per-tenant concurrency limits.

### 5. harbor-vault — Asset management

- Single storage node.
- Files split into fixed 4 MiB chunks; each chunk stored once at `chunks/ab/cd/<sha256>`.
- Hashing in a worker-thread pool.
- Metadata in Postgres: buckets, objects, versions, version→chunk list.
- Streaming upload and download; checksum verified on every read.
- Every version kept (rollback needs this).
- _Later:_ replication across nodes, garbage collection, presigned URLs, resumable uploads, quotas, public bucket API for apps.

### 6. harbor-runner — Machine agent

- One per app-hosting machine.
- Registers with control on start; heartbeats every 5 s with CPU, memory, disk, instance states.
- Start instance: download artifact → start container via Docker API → wait for app health → report healthy.
- Stop instance: report draining → wait for gate drain → stop container → clean up.
- Ships container logs to scope.
- Instance states: `pending → starting → healthy → draining → stopped` (or `failed`).

### 7. harbor-gate — Load balancing

- Built on `node:http` / `node:net`. One process per CPU core.
- Host-based routing: `<app>.harbor.local` → app's healthy instances.
- Algorithms per app: `round_robin` | `least_connections`.
- Active health checks every 2 s. Instance state machine: `healthy → (3 fails) → unhealthy → (2 passes) → healthy`.
- Passive ejection: mark unhealthy after consecutive proxy errors.
- Retry once on a different instance for idempotent requests only.
- Streaming proxy with backpressure; no full buffering.
- Draining: no new requests to a draining instance; existing ones finish (max 30 s).
- Route table pushed from control; updates without restart.
- Adds `x-request-id`; logs and measures every request.
- _Later:_ TLS, rate limiting, canary rollouts, a second gate, consistent hashing.

### 8. harbor-scope — Observability

- **Logs:** ingest endpoint for batches; stored in append-only daily files per app; simple index by app, level, time; search and live tail (Server-Sent Events).
- **Metrics:** scrapes `/metrics` from every service and instance every 10 s; stores samples in its own time-series files (one file per series per day); query by series name, labels and time range; basic `rate()` and percentile.
- Retention: logs 7 days, metrics 14 days.
- Scope going down never affects apps (senders buffer and drop).
- _Later:_ tracing, alerting, compression, downsampling.

### 9. harbor-dash — Dashboard

- Login / signup
- Apps list and create
- App page: releases, current deployment, instances and their states, roll back button
- Live logs with filters
- Charts: requests/sec, error rate, p50/p95/p99 latency, memory, event-loop lag
- Platform page: machines and their health

### 10. harbor CLI

```
harbor login
harbor apps create <name>
harbor apps list
harbor deploy            # in an app folder
harbor releases <app>
harbor rollback <app> <release>
harbor logs <app> [--follow]
harbor scale <app> <count>
```

### Tools

- **harbor-siege:** target URL, requests/sec, concurrency, duration. Reports success rate, status codes, p50/p95/p99 latency.
- **harbor-chaos:** kill a named process or container, stop a VM's service, (later) add latency / packet loss with `tc netem`.

---

## Deploy flow

1. CLI tars the app folder and calls `POST /apps/:app/releases` on control with an API token.
2. Control asks id: can this token `app:deploy` on this app? If not → 403.
3. Control creates a release (`status: uploading`) and streams the tarball to vault.
4. Control publishes `build.requested { release_id }` to queue; release → `building`.
5. A builder takes the job, builds, uploads the artifact to vault, publishes `build.succeeded`.
6. Control consumes it, release → `built`, and schedules N new instances across runners.
7. Runners start the instances and report `healthy`.
8. Control pushes a new route table to gate: new instances in, old instances `draining`.
9. After drain, control tells runners to stop old instances. Release → `live`.
10. Any failure → release `failed`, old version keeps serving.

Rollback = steps 6–9 using an existing built release.

---

## Data model (Postgres, phase one)

```
users(id, email, password_hash, created_at)
orgs(id, name, created_at)
memberships(user_id, org_id, role)
sessions(id, user_id, expires_at)
api_tokens(id, user_id, org_id, token_hash, name, last_used_at, revoked_at)

apps(id, org_id, name, instance_count, lb_algorithm, created_at)
releases(id, app_id, number, status, source_object_id, artifact_object_id, created_by, created_at)
machines(id, hostname, address, last_heartbeat_at, status)
instances(id, app_id, release_id, machine_id, port, state, updated_at)

buckets(id, org_id, name)
objects(id, bucket_id, key)
object_versions(id, object_id, size, sha256, created_at)
version_chunks(version_id, position, chunk_sha256)
```

## Queue topics

| Topic             | Producer | Consumer                   |
| ----------------- | -------- | -------------------------- |
| `build.requested` | control  | builder                    |
| `build.succeeded` | builder  | control                    |
| `build.failed`    | builder  | control                    |
| `*.dead`          | queue    | you (inspect in dashboard) |

## Ports

| Service                | Port |
| ---------------------- | ---- |
| gate                   | 80   |
| control                | 4000 |
| id                     | 4001 |
| queue                  | 4002 |
| vault                  | 4003 |
| scope                  | 4004 |
| builder (metrics only) | 4005 |
| runner                 | 4006 |
| dash                   | 3000 |

Internal calls use a shared internal service token from environment variables in phase one (replaced in a later phase).

## Machine layout (3 VMs, 2 GB RAM each)

| VM  | Runs                                            |
| --- | ----------------------------------------------- |
| vm1 | gate, control, id, queue, scope, dash, Postgres |
| vm2 | vault, builder, runner                          |
| vm3 | runner                                          |

Add `*.harbor.local` to your hosts file (or a tiny DNS entry) pointing at vm1.

---

## Build order for phase one

Each step ends with a checkpoint. Don't move on until it passes.

| Step | Build                                                       | Checkpoint                                                              |
| ---- | ----------------------------------------------------------- | ----------------------------------------------------------------------- |
| 0    | VMs, monorepo, `@harbor/core`, siege, chaos (kill only)     | A hello-world service shows logs and `/metrics`                         |
| 1    | gate with a static route table + a dummy app on vm2 and vm3 | Siege at 1,000 req/s; kill one dummy app; < 0.1% errors                 |
| 2    | queue                                                       | 100,000 messages; `kill -9` mid-publish; zero acked messages lost       |
| 3    | vault                                                       | Upload 2 GB file; RSS stays < 200 MB; download checksum matches         |
| 4    | builder                                                     | 5 builds in parallel; kill builder mid-build; job retried and completes |
| 5    | runner + control                                            | Containers start/stop on command; dead machine detected in 15 s         |
| 6    | CLI + full deploy flow                                      | `harbor deploy` takes a Node app to a live URL                          |
| 7    | scope (logs, metrics, charts)                               | Every service and app visible in charts and logs                        |
| 8    | id, wired into control, vault, scope                        | Second account fully isolated                                           |
| 9    | dashboard                                                   | All CLI actions available in the UI                                     |
| 10   | Phase one exam                                              | All tests below pass                                                    |

---

## Phase one exam (definition of done)

1. Sign up, create an app, deploy a real Node app with `harbor deploy`.
2. It runs as 2 instances on vm2 and vm3 at `<app>.harbor.local`.
3. Redeploy while siege sends 500 req/s: **zero** failed requests.
4. `kill -9` one instance under load: error rate < 0.1%, gate removes it within 6 s, control replaces it.
5. Restart the queue mid-build: the build still completes, exactly one release goes live.
6. Roll back to the previous release from the dashboard, with zero failed requests.
7. A second account gets 403/404 on every endpoint for the first account's apps, releases, objects and logs (automated test).
8. Under one hour of siege load, every service stays under its heap cap with no memory growth trend, and event-loop lag p99 stays < 50 ms.
9. Stop scope entirely: apps keep serving; services keep running.
10. Every failure above is diagnosable from Harbor's own logs and charts, without SSH.

---

## Later phases (outline)

- **Phase 2 — Resilience:** queue replication and failover, vault replication + repair + garbage collection, second gate, TLS with your own CA.
- **Phase 3 — Identity deep dive:** OIDC provider ("Login with Harbor" for deployed apps), 2FA, teams, refresh token rotation, key rotation, policy language compiled to JS.
- **Phase 4 — Observability deep dive:** distributed tracing across gate → app → queue → builder, alerting rules, metric compression and downsampling.
- **Phase 5 — Fairness & scale:** tenant-fair build scheduling, rate limiting, canary rollouts, consistent hashing, quotas, build cache.
- **Final exam:** Harbor hosts its own dashboard and your portfolio, and survives a 30-minute random chaos run with no user-visible downtime.

---

## Working log

Keep a short entry per session: what you built, what broke, what you learned. Record every design decision in `docs/decisions/` (context, options, choice, why).
