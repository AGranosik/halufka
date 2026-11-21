# halufka - structure agreement (health-check walking skeleton)

Status: AGREED (amendment 1, round 4; minors folded, user decisions recorded 2026-11-21); previous AGREED text was round 3
Author: architect | Date: 2026-11-21 | Reviewers: golang-developer, react-developer, database-designer

**Recommendation.** React+TS SPA (Vite) -> Go API (stdlib `net/http` + `pgx`) -> PostgreSQL 17. Locally: `docker compose up --build` (db, one-shot migrate, api, nginx-served SPA with `/api` proxied, so same-origin, no Node on host). Azure later: Static Web Apps Free, scale-to-zero Container App, Postgres Flexible Server. The one working connection is a single endpoint, `GET /api/v1/health`, that also queries the database; the SPA page shows backend and database status.

**Amendment 1 (round 4).** User decision (binding): "prepare code for migrations but no need for schema now. Just for backend healthcheck run database health check as well. This will be enough."
- Removed: `pings`/`ping_counter`, `0001_create_pings.sql`, `POST /api/v1/pings`, the button feature, `internal/ping`, the CTE, `.UTC()` rules, id/total contract, ping error classification, `/healthz`, smoke id/total assertions. Added: one health endpoint with a DB check (`Pinger`), a no-op baseline migration, a migration integration test replacing the old gate, a status page.
- Round-3 build notes stand: `-count=1` and test-level skip detection in gatecheck, `setx` then restart terminals, README per shell, golangci v2 module path, 499/no-envelope on client cancel, DB reuse between gates 4 and 7. Round-4 reviewer minors are folded in (objection log IDs 4-*). **[VERIFY]** marks pricing/quota/product claims not checked online.

## Goal and non-goals

Goal: a tested skeleton proving browser -> API -> DB -> API -> browser locally through one health check, with clean extension points (layers, small interfaces, migration runner, versioned API, error envelope) and a high-level Azure mapping.

Acceptance (testable):
1. With only Docker, `docker compose up --build`, open `http://localhost:3000`: the page shows Backend ok and Database ok from a live call. "Check again" repeats it. Manual: after `docker compose stop db`, "Check again" shows Database unavailable, backend reachable.
2. The same API image runs on Azure with env-var changes only. The SPA bundle does not: `VITE_API_BASE_URL` is baked at build time, so the SPA is rebuilt per environment (runtime config deferred).
3. All quality gates below pass locally.

Non-goals: auth, users, any hall-ball domain model, any application schema or table (deferred), realtime, i18n, PWA, custom domain, multi-region, caching, queues, metrics/tracing, Azure infra/deploy in this run.

## Repository layout

```
halufka/
|-- README.md, .gitignore, .gitattributes, .env.example, compose.yaml   README gains a quickstart incl. env setup; ignore node_modules/ dist/ coverage/; `* text=auto eol=lf` (autocrlf breaks Dockerfiles/nginx conf/SQL); compose services db, migrate (one-shot), api, web
|-- backend/                      Go module github.com/AGranosik/halufka/backend
|   |-- go.mod, Dockerfile, .dockerignore, .golangci.yml   `go 1.26` + `toolchain go1.26.3`; `tool` directive pins golangci-lint; golang:1.26 -> distroless nonroot, one image; lint config `version: "2"`, `run.build-tags: [integration, smoke]`, `formatters: enable: [gofmt]`
|   |-- cmd/halufka/main.go       entrypoint only: signal context, subcommand (serve|migrate|healthcheck), builds the listener from HTTP_ADDR, exit code
|   |-- internal/config/, app/    env parsing + fail-fast validation; `Run(ctx, cfg, ln net.Listener)`: lifecycle (testable with a `:0` listener), http.Server timeouts, graceful drain; healthcheck client reads only HTTP_ADDR
|   |-- internal/httpapi/         router (takes a `Deps` struct), health handler, `Pinger` interface, middleware, error envelope
|   |-- internal/db/              pgxpool, goose Provider runner, `//go:embed migrations/*.sql` (only `0001_baseline.sql`, no DDL)
|   |-- tools/gatecheck/main.go   gate 4 helper (`go run`): runs `go test -json`, fails on any skipped test or total < 80%
|   `-- smoke/smoke_test.go       build tag `smoke`: black-box test against the compose stack
`-- frontend/                     Vite + React 19 + TypeScript strict
    |-- package.json + package-lock.json (committed), Dockerfile, .dockerignore (node_modules dist coverage), default.conf (nginx `server{}` block only, `listen 8080`)
    `-- vite.config.ts, vitest.config.ts, tsconfig.json, eslint.config.js, src/ (main.tsx, App.tsx, api/health.ts, *.test.{ts,tsx})
```
`infra/` and `.github/workflows/` are later phases. Extension point: a new feature adds `internal/<feature>` taking a `*pgxpool.Pool` (or a small interface) and owning its SQL; `httpapi` handlers depend on small interfaces declared in `httpapi` (like `Pinger`, fakeable), wired through `Deps` in `app.Run`.

## Components and boundaries

**Frontend.** One page that calls the health endpoint on load and shows Backend and Database status; "Check again" resets the view to loading, then repeats the call. States: loading / slow (>3 s, so it does not flash when the server's own 2 s DB deadline hits: "Waking the server up...") / ok / error. Envelope `db_unavailable` -> Backend ok, Database unavailable (with requestId); any other envelope code (`internal`, `not_found`, `method_not_allowed`) -> Backend error, Database unknown; `network`/`timeout`/`bad_response` -> Backend unreachable, Database unknown. No business rules, no hard-coded URLs (only `VITE_API_BASE_URL`, empty = same-origin, trailing slash stripped), no auto-retry, no router/state lib/UI kit.
- Client returns a discriminated union `{kind:'ok',data} | {kind:'error',code,message,status?,requestId?}`. Success is HTTP 200 with body `status === 'ok' && db === 'ok'` (hand-written guard); any other response that is not a valid envelope (nginx 502, ingress 503, SWA 404, HTML, wrong body) is `bad_response`, a rejected `fetch` is `network`. A 30 s timer sets a `timedOut` flag and aborts: flag set -> `timeout`; an abort without it (the on-load effect's unmount cleanup, StrictMode) is silent, no state update. One client unit test asserts `fetch` is called with method `GET`, no headers/body and no `cache: 'no-store'` or other header-adding option (the server's `Cache-Control: no-store` suffices; extra headers would trigger CORS preflight on Azure).
- A11y: `aria-disabled` plus an in-flight `useRef` guard on "Check again" (not `disabled`, which loses focus); the ref is reset in a `finally`/effect cleanup so StrictMode's double mount cannot leave it stuck; the `aria-live="polite"` status region is mounted from first render, empty, only its text changes.
- Tooling: one `tsconfig.json` (no solution-style `references`, which makes `tsc --noEmit` check nothing): `strict`, `noEmit`, `include: [src, vite.config.ts, vitest.config.ts]`; the builder confirms gate 5 fails once on a deliberate type error. Scripts `typecheck`, `lint` (`eslint . --max-warnings 0`), `test:ci` (`vitest run --coverage`), `build`; coverage thresholds (lines/statements 80, branches 70) in `vitest.config.ts`, with `include: ['src/**/*.{ts,tsx}']` and `exclude` tests and `src/main.tsx`; tests are `*.test.{ts,tsx}`, using fake timers for the 3 s slow state and the 30 s timeout.

**Backend.** Validates config, serves the API, owns the pool and all SQL (none yet), logs JSON (`slog`), drains on SIGTERM. No SPA serving, no implicit migrations in `serve`, no per-request state, no DSN in logs, no ORM/framework; the health handler sees only `Pinger`.
- `internal/app.Run` builds the pool (lazy: no connect at start, so the API starts during a DB outage; `ConnConfig.ConnectTimeout` 3 s, since pgx's 2 min default would let hung dials in a packet-dropping outage hold pool slots past the 2 s health deadline) and `http.Server` (ReadHeaderTimeout 5 s, ReadTimeout 10 s, WriteTimeout 15 s, IdleTimeout 60 s); on cancel `Shutdown` (8 s, under compose `stop_grace_period: 15s`), then closes the pool. `halufka healthcheck` reads only `HTTP_ADDR` (not `config.Load`, which needs `DATABASE_URL`), GETs `http://127.0.0.1:<port>/api/v1/health` (3 s timeout, above the 2 s DB check), exit 0 on 200 else 1 (distroless has no curl).

**Database.** No application schema yet; reachable only by API and migrate. `migrations/` is owned by golang-developer (baseline only); database-designer reviews migrations once real schema starts.

## API contract

Public API under `/api/v1`. JSON `application/json; charset=utf-8`. Every response carries `X-Request-Id` (generated; a valid inbound one of <=64 chars `[A-Za-z0-9-]` is reused) and `Cache-Control: no-store`.

**`GET /api/v1/health`** - the single health check: compose healthcheck, later Azure probes and the SPA all use it; no `/healthz`, no `/readyz`. No body or parameters. It calls `Pinger.Ping(ctx)` under a 2 s `context.WithTimeout`.
`200`: `{ "status": "ok", "db": "ok" }` (clients ignore unknown fields). Any `Ping` error -> `503` envelope `db_unavailable` with `Retry-After: 2`. A client disconnect is neither: the handler checks `r.Context().Err()` first (not only `errors.Is(err, context.Canceled)` on the `Ping` error), writes no body, and the access log records a fixed status 499 at info; not counted as a server error.
`Pinger` is `interface{ Ping(ctx context.Context) error }`, declared in `httpapi`; `*pgxpool.Pool` satisfies it (`Ping(ctx) error`, one server round trip, equivalent to `SELECT 1`). Unit tests use a fake; an integration test uses a real pool (200) and a closed pool (503; `Ping` returns `ErrClosedPool`).

Error envelope (every non-2xx incl. 404/405/panic): `{ "error": { "code": "db_unavailable", "message": "...", "requestId": "..." } }`

| Status | `code` | When |
|---|---|---|
| 404 | `not_found` | unmatched path |
| 405 | `method_not_allowed` | wrong method on a known path (`/api/v1/health`; sets `Allow: GET`) |
| 503 | `db_unavailable` | `Pinger` returned an error or timed out; `Retry-After: 2` |
| 500 | `internal` | anything else, e.g. panic (details in logs only) |

- Routing (stdlib mux emits plain-text 404/405): register `GET /api/v1/health`, method-less `/api/v1/health` (enveloped 405), catch-all `/` (enveloped 404); the routing test covers `POST /api/v1/health`. Middleware outermost first: request-id -> access-log -> CORS -> recover -> mux; recover re-panics `http.ErrAbortHandler`. CORS sets headers before `next`, so ACAO is present on 404/405/500/503 too; the log sees the final status.
- CORS: local is same-origin via nginx, none needed. On Azure (different origins) the API sends `Access-Control-Allow-Origin: <origin>` + `Vary: Origin` only when `Origin` exactly matches `CORS_ALLOWED_ORIGINS` (comma list, no wildcard, no credentials); empty = no headers; compose default `http://localhost:3000`. No `OPTIONS` handling in v1.
- Accepted trade-off (user): probes now depend on the DB, so a DB outage makes the API unhealthy; on Azure that can mean liveness restart loops and an opaque ingress 503 instead of the envelope, without CORS headers, so the cross-origin SPA shows "Backend unreachable" instead of "Database unavailable" (compose has no restart policy: it only shows unhealthy). If it hurts, split liveness later: a DB-free liveness path, this endpoint stays as readiness.

## Database

`postgres:17-bookworm` (Debian, floating tag: minor updates arrive on pull, accepted) locally, since Alpine musl collation differs from Azure glibc. Azure must offer 17 **[VERIFY]**.

Migration conventions: `timestamptz`, identity columns, snake_case, explicit constraint names, file names `NNNN_verb_object.sql` (baseline excepted), PK/FK column types match, explicit `ON DELETE` on every FK, an index on every FK column, one concern per migration, forward-only (no Down section), expand/contract only (two API revisions overlap during rollout). `CREATE INDEX CONCURRENTLY` needs `-- +goose NO TRANSACTION`. The integration-test helper's role needs CREATEDB.

Migration runner (ready, ships no schema): goose via `halufka migrate` (same image, never run by `serve`). Provider API (goose >= v3.21 **[VERIFY]**) with the Postgres session locker from `goose/lock` (mandatory); the Provider needs a `*sql.DB` (`pgx/v5/stdlib`, e.g. `OpenDBFromPool`) and the embedded FS via `fs.Sub(..., "migrations")`. The locker holds one pooled connection for the whole run, so config validation requires `DB_MAX_CONNS >= 2` (chosen over a separate `migrate` pool). Commands `up` and `status` only (both take the session lock), no `down`. Direct port 5432, never a transaction-pooling proxy. Compose `migrate` runs after `db` healthy and `api` waits on `service_completed_successfully`; on Azure the deploy pipeline runs `halufka migrate` (same image) as a step before the new revision gets traffic; how it reaches the DB (one-off Container Apps Job or the runner) is decided in the infra phase.

Empty migration set (resolved): goose's `NewProvider` returns `ErrNoMigrations` when the FS has no migrations, and `//go:embed migrations/*.sql` with zero matches does not compile. So ship one no-op baseline, `internal/db/migrations/0001_baseline.sql`: `-- +goose Up` plus a plain comment (intentionally DDL-free, real schema starts at 0002), no Down; the comment must not contain the text `+goose` (goose parses `-- +goose` prefixes). Immutable once applied anywhere: later changes go in new migrations. Verified 2026-11-20 against goose v3.30.1 source (`provider.go`) and a scratch program outside the repo: empty `fs.FS` gives `ErrNoMigrations`; with the baseline a Provider is built and the SQL parser returns 0 statements, no error. Not run on a live DB: the integration test confirms `up` records version 1. `status` is not read-only: on a fresh DB it creates `goose_db_version` plus a version-0 row (read in goose v3.30.1; tool bookkeeping, not schema). Rejected: treating `ErrNoMigrations` as success, which leaves the locker and `up` path unexercised until the first real migration.

Migration integration test (replaces the old up/down/up gate; run through gatecheck on a per-test database): `up` on an empty Postgres succeeds; `status` shows the baseline pending before and applied after; a second `up` is a no-op; the set of tables in schema `public` is exactly `{goose_db_version}` (goose bookkeeping, not application schema); a separate connection queries `pg_locks` (cluster-wide, so scoped with `pg_locks.database = (SELECT oid FROM pg_database WHERE datname = current_database())`) and sees no session advisory lock after `up` returns (not asserted while `up` runs: the no-op baseline finishes in milliseconds, so there is no window to observe).

Roles: DEFERRED. One role `halufka` for migrate and serve. Locally it is the image's superuser (hides permission errors); accepted while no tables exist. On Azure use a non-admin owner role, never the server admin; PG15+ stops non-owner roles creating objects in `public`, so the migrate role must own the database or hold CREATE on schema `public` (goose creates `goose_db_version` there).

## Local development

Docker with Compose v2 only; no Node, az or make. Host Go is for the fast loop, tests, smoke, lint. No bind mounts (Windows traps); use `--build`; `down -v` resets the DB.

| Service | Image / build | Host port | Notes |
|---|---|---|---|
| `db` | `postgres:17-bookworm`, volume `halufka-pgdata` | `127.0.0.1:${DB_HOST_PORT:-5432}` | healthcheck `pg_isready -h 127.0.0.1 -U halufka -d halufka` (TCP, since the init server is socket-only); `interval: 2s, timeout: 3s, retries: 30, start_period: 10s` (cold initdb fits the 120 s budget) |
| `migrate` / `api` | `./backend`: `migrate` (one-shot, exits 0) / `serve` | `${API_PORT:-8080}` (api) | `api` waits for `migrate`; healthcheck `halufka healthcheck` (the health endpoint, so it includes the DB), `interval: 2s, timeout: 5s, retries: 15, start_period: 5s` (the 30 s default interval would eat the 120 s budget); `stop_grace_period: 15s` |
| `web` | `./frontend` runtime | `${WEB_PORT:-3000}` -> 8080 | waits for `api` healthy; healthcheck `wget -q --spider http://127.0.0.1:8080/`, `interval: 2s, timeout: 3s, retries: 15, start_period: 5s` |

Env (compose defaults, documented in `.env.example`): `DATABASE_URL` (required, no code default; local `postgres://halufka:halufka@db:5432/halufka?sslmode=disable`), `HTTP_ADDR` `:8080`, `CORS_ALLOWED_ORIGINS` (compose `http://localhost:3000`, code default empty), `DB_MAX_CONNS` 5 (validated >= 2), `DB_MIN_CONNS` 0, `LOG_LEVEL` info, `VITE_API_BASE_URL` (build arg, empty), `POSTGRES_USER/PASSWORD/DB` (`halufka`), port overrides.

Frontend container:
- Runtime `nginxinc/nginx-unprivileged:<x.y>-alpine` (exact tag) **[VERIFY tag]**; `default.conf` (a `server{}` block only, `listen 8080`) copied to `/etc/nginx/conf.d/default.conf`.
- R1 fix: a literal `proxy_pass http://api:8080` is resolved once at startup and cached. Use Docker's resolver and a variable: `resolver 127.0.0.11 valid=10s ipv6=off;`, `set $api_upstream api:8080;`, `location /api/ { proxy_pass http://$api_upstream; }` (no URI part, path passes unchanged), so `/api/v1/health` is proxied like any API path. `location /` serves `dist/`.
- Stages: `deps` (`npm ci`) -> sibling `check` (typecheck, lint, tests) and `build` (`ARG VITE_API_BASE_URL`) -> `runtime` (copies `dist` from `build`). BuildKit skips unreferenced stages, so gates build `--target check` explicitly.
- Lockfile bootstrap, once, no mounts: a Dockerfile stage `lockfile` (`node:24-alpine`, `npm install --package-lock-only`, then `FROM scratch` exporting `package-lock.json`; Node 24 LTS **[VERIFY]**, same tag as `deps`), run as `docker build --target lockfile --output type=local,dest=frontend ./frontend`; commit the lockfile.

One-time setup (README quickstart): config for tests and smoke lives in Windows user-level environment variables, set once. No defaults in Go code, no `.env` loader.
```
setx TEST_DATABASE_URL "postgres://halufka:halufka@localhost:5432/halufka?sslmode=disable"
setx SMOKE_BASE_URL "http://localhost:3000"
setx SMOKE_API_URL "http://localhost:8080"
```
`setx` affects only processes started afterwards: restart terminals and the IDE. A missing variable makes the test FAIL with a message naming it (never skip). If `DB_HOST_PORT`/`API_PORT`/`WEB_PORT` are overridden, update the matching variable.

Tests: Go unit `go -C backend test ./...` (no DB). Integration tests use a helper that builds the name with `pgx.Identifier{"test_<random>"}.Sanitize()` for `CREATE DATABASE` (one documented `//nolint:gosec`) and in `t.Cleanup` closes its pool then runs `DROP DATABASE ... WITH (FORCE)`. Frontend: `docker build --target check ./frontend`. Fast backend loop: `docker compose up -d db migrate`, then `go -C backend run ./cmd/halufka serve` with `DATABASE_URL` (at `localhost`) set in the shell.

## Azure hosting

- SPA: Static Web Apps Free, built with `VITE_API_BASE_URL=https://<api-fqdn>`. API: Container Apps Consumption, 0.25 vCPU / 0.5 GiB, min 0, max 2; image on ghcr.io; external ingress; startup + liveness probes on `/api/v1/health` (DB-dependent, accepted trade-off in API contract). Cold start from 0 replicas (a few to ~10 s) is absorbed by the lazy pool, the "waking up" UI and the 30 s timeout; `minReplicas=1` probably exceeds the free grant **[VERIFY]**.
- Migrations: run by the deploy pipeline, never on API startup (same image, `migrate`; see Database). DB: Flexible Server Burstable B1ms, PG 17, public access + TLS + firewall. Budget alert. Free tiers **[VERIFY]**: SWA Free, a Container Apps monthly grant, Postgres B1ms free 12 months (then ~12-20 USD/month).
- Networking/secrets: non-admin role, firewall "allow Azure services" **[VERIFY]**. Azure DSN `sslmode=verify-full` **[VERIFY CA chain]**; if the distroless CA bundle lacks the Azure chain, bake the root PEM and set `sslrootcert`. `DATABASE_URL` is a Container Apps secret. Pool: `DB_MAX_CONNS=5`, `MaxConnIdleTime=2m`, `MaxConnLifetime=30m`; before raising max replicas check `2 x maxReplicas x DB_MAX_CONNS + 5 <= 80% of max_connections` (2 = revision overlap; B1ms limit **[VERIFY]**).

## Scalability

- Horizontal: stateless API replicas (HTTP concurrency scale rule, max 2 as cost cap) and a static SPA at the edge; one pool per replica (budget above), scale the DB vertically first. Stateless rules: no cross-request state, no sticky sessions, config from env only, SIGTERM drains, expand/contract migrations, every handler has a context deadline.
- Abuse: with the limiter deferred, the public health endpoint (one DB round trip per call) is bounded by max 2 replicas, `DB_MAX_CONNS`, the 2 s deadline and the budget alert.
- Observability: JSON access log per request (`ts, level, msg, request_id, method, path, status, duration_ms`), error log with `request_id`, the health endpoint, Container Apps built-in metrics.

## CI/CD

Later, a small `.github/workflows/ci.yml` runs the gates below on push/PR (Postgres service `postgres:17-bookworm`; workflow env sets `TEST_DATABASE_URL` and `SMOKE_*`; pin actions). Deploy workflow with infra: push image, OIDC login, migrate step, update revision, deploy SPA, smoke. Rollback = previous revision (safe: expand/contract).

## Quality gates for the build phase

From repo root. Every command is a single plain command, identical in PowerShell, cmd and Git Bash (`go -C backend` instead of `cd`; no `&&`, env prefixes, `${PWD}`, `-w`, pipes or redirects). Each must exit 0. Variables come from the one-time setup (Local development). Order: gates 1-3, 5, 6 any time; gate 4 strictly before gate 7, never in parallel (both use the host DB port and gate 7 ends with `down -v`).

| # | Gate | Command(s), in order | Threshold |
|---|---|---|---|
| 1 | Go format | none separate: `formatters: enable: [gofmt]` in `.golangci.yml`, enforced by gate 3 | unformatted file = gate 3 fails |
| 2 | Go vet | `go -C backend vet ./...` | clean |
| 3 | Go lint | `go -C backend tool golangci-lint run` (tool pinned in go.mod, no Docker) | zero findings (govet, staticcheck, errcheck, errorlint, gosec, bodyclose, rowserrcheck, sqlclosecheck, gofmt). Binary install is the upstream-preferred route; go.mod tool deps accepted for minimal scope. |
| 4 | Go tests + coverage | `docker compose up -d --wait db`; then `go -C backend run ./tools/gatecheck` (needs `TEST_DATABASE_URL`) | `gatecheck` runs `go test -tags=integration -covermode=atomic -coverpkg=./internal/... -coverprofile=coverage.out -json ./internal/...` then `go tool cover -func=coverage.out`; exit 1 (message printed) if any test has action `skip`, any test fails, or `total:` < 80.0%; exit 0 otherwise. Includes the migration and health integration tests. Only `./internal/...` measured; no per-package gate. |
| 5 | Frontend check | `docker build --target check ./frontend` | strict tsc clean, eslint 0 warnings, vitest thresholds |
| 6 | Frontend build | `docker build --target runtime ./frontend` | succeeds |
| 7 | Compose smoke | `docker compose build`; `docker compose up -d --wait`; `go -C backend test -tags=smoke -count=1 ./smoke/...` (needs `SMOKE_BASE_URL`, `SMOKE_API_URL`); `docker compose down -v` (cleanup, run even on failure) | `--wait` plus assertions finish within 120 s after images are built (pulls and builds excluded) |

Smoke assertions: `GET {web}/` 200 HTML; `GET {web}/api/v1/health` (the frontend's route, through the nginx proxy) 200 with `status` and `db` ok; `GET {web}/api/v1/nope` -> 404 envelope; `GET {api}/api/v1/health` with `Origin: http://localhost:3000` returns that ACAO, with `Origin: http://evil.test` none; `GET {api}/api/v1/nope` with the allowed origin returns the 404 envelope plus ACAO. `up --wait` with a one-shot service needs checking on the installed Compose **[VERIFY]**; fallback `up -d` and poll the health endpoint.

Build order: (1) golang-developer: config, db (incl. `0001_baseline.sql`), httpapi, app, main, tests, Dockerfile, lint, `tools/gatecheck`; (2) react-developer: scaffold, App, client, tests, Dockerfile, default.conf, lockfile; (3) integrator: compose, `.env.example`, git files, smoke, README, gate 7. Contract frozen once reviewers agree; later changes go through the objection log.

## Decision log

| # | Decision | Rejected | Why |
|---|---|---|---|
| 1 | PostgreSQL 17 | Azure SQL free offer, Cosmos | User prefers Postgres; local parity and `pg_dump` exit. Free only 12 months **[VERIFY]**. |
| 2 | `/api/v1` prefix (incl. the health endpoint), JSON error envelope with stable `code` | unversioned; RFC 9457 | Public paths and error shape are hard to change once clients exist. |
| 3 | One-shot forward-only migrations (goose, session lock), expand/contract; no-op baseline `0001` | migrate on start; prod down migrations; runner that tolerates zero migrations | No replica races or revision-overlap breakage; exercises the real runner now. |
| 4 | SPA on SWA + API on Container Apps | single container with `go:embed`; App Service F1 | Edge-served page, independent deploys. Fallback: run the `web` image as a second Container App. |
| 5 | One health endpoint `GET /api/v1/health` that includes a DB check (user decision) | separate liveness `/healthz` + readiness | One check, one path for compose, probes and SPA. Price: DB outage = unhealthy API (API contract). |

### Deferred / later

- First real feature: its table(s), write endpoint and `internal/<feature>` package; database-designer returns then, and a blocking lock test (the test holds the advisory lock, `up` must wait) is added with that first real migration. The removed ping material (counter/CTE design etc.) is deliberately not kept in this spec. Liveness/readiness split (DB-free liveness path) if the DB-dependent probe hurts.
- Rate limiting (never on probes), DB role split (`halufka_owner`/`halufka_app`, `MIGRATE_DATABASE_URL`; with the first real table), CORS preflight and a `web-cors` compose profile, runtime-injected SPA config.
- govulncheck, `-race`, per-package coverage, full CI/deploy YAML, `infra/main.bicep`, cold-start measurement, custom domain.
- PgBouncer, read replicas, zone-redundant HA, VNet + private endpoint, Key Vault, OpenTelemetry/metrics, sqlc/chi, caching, queues, realtime, auth.

## Open questions for the user

1. New Azure free account (12-month Postgres benefit **[VERIFY]**), which region, and what after month 12? Default: assume eligible; stay on Postgres, decide at month 10.
2. Is the repo/ghcr.io image public? Default: public (no secrets in it).
3. Budget alert threshold/e-mail? Default: 5 EUR/month.

## Objection log

| round | reviewer | objection | architect response (accept/reject+reason) | status |
|---|---|---|---|---|
| 1 | react-developer | R1 BLOCKER: nginx caches `api` DNS; conf.d placement; pin alpine tag | Accept. Docker resolver + variable `proxy_pass`; `/etc/nginx/conf.d/default.conf`; pinned tag. | resolved |
| 1 | react-developer | R2: Dockerfile stage graph, coverage export, lockfile bootstrap, .dockerignore | Accept. Sibling `check`/`build` off `deps`, `coverage` stage, docker-run lockfile, `.dockerignore`. | resolved |
| 1 | react-developer | R3: "same SPA bundle on Azure" false; base-URL normalisation; `web-cors` profile | Accept claim and normalisation (acceptance 2 reworded, slash stripped). Profile accepted as deferred: smoke covers ACAO at API level; browser cross-origin is for the infra phase. | resolved; profile deferred |
| 1 | react-developer | R4: non-envelope errors; discriminated union; AbortController; ACAO on errors | Accept. Union with `bad_response`/`network`/`timeout`, 30 s abort, CORS outside recover, smoke asserts ACAO on 404. | resolved |
| 1 | react-developer | R5: aria-disabled + in-flight ref; live region from first render | Accept. Specified in Frontend. | resolved |
| 1 | react-developer | R6: tsconfig/scripts/coverage thresholds in vitest.config | Accept. Specified in Frontend. | resolved |
| 1 | golang-developer | G1 BLOCKER: per-package gate not computable; `cmd/` at 0% breaks total | Accept. Per-package gate dropped (Deferred); one total >= 80% over `./internal/...` with `-coverpkg`. | resolved |
| 1 | golang-developer | G2: plain-text 404/405; middleware order | Accept. Method-less route + catch-all; order request-id, access-log, CORS, recover, mux. | resolved |
| 1 | golang-developer | G3: no home for lifecycle/timeouts/healthcheck | Accept. `internal/app.Run`, server timeouts, `halufka healthcheck`. | resolved |
| 1 | golang-developer | G4: goose locker; Provider API, >= v3.21, direct 5432 | Accept. As specified in Database. | resolved |
| 1 | golang-developer | G5: limiter scope (POST only, X-Forwarded-For) | Accept, moot: limiter deferred; POST-only scope recorded in Deferred. Abuse bound noted in Scalability. | deferred; superseded by round 4 |
| 1 | golang-developer | G6: `/readyz` opaque 503; use `/healthz` | Accept. Probes use `/healthz`; `/readyz` deferred. | superseded by round 4 |
| 1 | golang-developer | G7: 503 vs 500 classification | Accept. `ping.ErrUnavailable` + `errors.Is` mapping. | superseded by round 4 |
| 1 | golang-developer | G8: integration DB isolation; build tags in golangci | Accept. Per-test `CREATE DATABASE` helper; `run.build-tags` set. | resolved |
| 1 | golang-developer | G9: go directive and linter Go pinning | Accept. `go 1.26` + `toolchain go1.26.3`; images Go >= 1.26. | resolved |
| 1 | database-designer | D1: local superuser parity; role SPLIT decision (owner/app, bootstrap.sql, MIGRATE_DATABASE_URL) | Accept as DEFERRED, explicitly: split is out of minimal scope (one throwaway table, no sensitive data). Gap documented in Database; Azure still uses a non-admin owner. Revisit with first real table. | deferred |
| 1 | database-designer | D2: `count(*)` O(n) on public write | Accept. `ping_counter` row + single-statement CTE. | superseded by round 4 |
| 1 | database-designer | D3: `pg_isready` passes during init | Accept. `pg_isready -h 127.0.0.1 -U halufka -d halufka`. | resolved |
| 1 | database-designer | D4: locker mandatory; migrate up/status only | Accept. Locker mandatory; `up`/`status`; no Down section. | resolved |
| 1 | database-designer | D5: connection budget with revision overlap; idle 2m; SafeToRetry | Accept, short form: overlap-aware budget rule, `MaxConnIdleTime=2m`, one retry on `SafeToRetry`. PgBouncer deferred. | resolved; `SafeToRetry` part superseded by round 4 |
| 1 | database-designer | D6: verify-full not require | Accept. Azure DSN `verify-full` **[VERIFY CA chain]**; detail finalised in infra phase. | resolved; detail deferred |
| 1 | database-designer | D7: alpine musl vs Azure glibc collation | Accept. `postgres:17-bookworm` local and CI. | resolved |
| 3 | user | U1: lint via Go tool in go.mod, `go tool golangci-lint run`, no Docker/`-w`; config `version: "2"` | Accept (user decision). `tool` directive; officially-preferred binary install noted, go.mod tool deps accepted. | resolved |
| 3 | user | U2: TEST_DATABASE_URL / SMOKE_BASE_URL / SMOKE_API_URL as user-level env vars (`setx`), no code defaults, no `.env` loader, missing = FAIL | Accept (user decision). One-time setup block, setx restart note, CI later, port overrides need matching update. | resolved |
| 3 | user | U3: Postgres image stays floating `postgres:17-bookworm` | Accept (user decision). Wording removed; floating tag stated. | resolved |
| 3 | user | U4: no shell dependence in gates; shell-independent coverage/skip check | Accept (user decision). `go -C backend ...`; `tools/gatecheck` (`go run`) does `go test -json` skip check and 80% total; exit 0/1 defined. | resolved |
| 3 | golang-developer | 3-G1 IMPORTANT (N1): mixed shell dialects, `${PWD}`, `-w /app` | Accept. Resolved by U1/U2/U4; no gate command uses env prefixes, `${PWD}`, `-w`, `&&` or pipes. | resolved |
| 3 | golang-developer | 3-G2 IMPORTANT (N2): gate 4 not self-contained; unnamed threshold/skip commands; ordering vs gate 7 | Accept. Gate 4 lists `up -d --wait db`, env var, `gatecheck`; order stated (4 before 7, not parallel). | resolved |
| 3 | golang-developer | 3-G3 (N3): `context.Canceled`; 57P02/57P03; define connect failures | Accept. Canceled is logged at info, not a server error; classes added; `*pgconn.ConnectError` or `net.Error`. | resolved; classes part superseded by round 4 (Canceled stands) |
| 3 | golang-developer | 3-G4 (N4): `app.Run` testability; healthcheck must not call `config.Load` | Accept. `Run(ctx, cfg, ln net.Listener)`; healthcheck reads only `HTTP_ADDR`. | resolved |
| 3 | golang-developer | 3-G5 (N5): `POST /healthz` gives 404; recover and `ErrAbortHandler` | Accept. Method-less `/healthz` (enveloped 405, `Allow: GET`), routing test, recover re-panics `http.ErrAbortHandler`. | superseded by round 4 (route is now `/api/v1/health`) |
| 3 | golang-developer | 3-G6 (N6): Shutdown 10 s vs compose 10 s grace | Accept. Shutdown 8 s, `stop_grace_period: 15s`, pool closed after Shutdown. | resolved |
| 3 | golang-developer | 3-G7 (N7): gosec G201/G202 on `CREATE DATABASE` | Accept. `pgx.Identifier{}.Sanitize()` plus one documented `//nolint:gosec`. | resolved |
| 3 | golang-developer | 3-G8 (N8): goose needs `*sql.DB`, `goose/lock`, `fs.Sub`; lock test | Accept. Stated in Database; migration test asserts no advisory lock remains. | resolved |
| 3 | golang-developer | 3-G9 (N9): `DROP DATABASE ... WITH (FORCE)`; port overrides | Accept. Pool closed then `DROP ... WITH (FORCE)` in `t.Cleanup`; overrides mirrored into env vars (U2). | resolved |
| 3 | react-developer | 3-R1 (N1): solution-style tsconfig makes `tsc --noEmit` check nothing | Accept. Single `tsconfig.json`, `include` set, strict, noEmit; builder proves gate 5 fails on a deliberate type error. | resolved |
| 3 | react-developer | 3-R2 (N2): "no preflight" needs no custom headers/body | Accept. One client unit test: `fetch` called with `POST`, no headers/body. | superseded by round 4 (now `GET`) |
| 3 | react-developer | 3-R3 (N3): "no localhost URL in bundle" has no command | Accept. Clause dropped from gate 6 (`VITE_API_BASE_URL` empty by default; scoped grep deferred). | resolved |
| 3 | react-developer | 3-R4 (N4): `nginx.conf` vs `conf.d/default.conf` | Accept. File renamed `default.conf`, `server{}` block only, `listen 8080`. | resolved |
| 3 | react-developer | 3-R5 (N5): no `${PWD}`/`-w` in remaining gates | Accept. None remain; lockfile bootstrap now a Dockerfile `lockfile` stage exported with `--output`, no mount. | resolved |
| 3 | react-developer | 3-R6 (N6): "120 s from cold" ambiguous | Accept. 120 s covers `--wait` plus assertions after images are built; pulls/builds excluded. | resolved |
| 3 | database-designer | 3-D1 (N1): text claims a pin, tag is floating | Accept. See U3. | resolved |
| 3 | database-designer | 3-D2 (N2): `pg_isready -d` rationale; healthcheck timings | Accept. Reworded (socket-only init server); `interval 2s, timeout 3s, retries 30, start_period 10s`. | resolved |
| 3 | database-designer | 3-D3 (N3): "tables dropped later" contradicts expand/contract | Accept. Drop is a contract migration at least one release after code stops using the tables. | superseded by round 4 |
| 3 | database-designer | 3-D4 (N4): `timestamptz` local zone vs RFC 3339 UTC; `id` vs `total` | Accept. `.UTC()` in `internal/ping`, unit test for `Z`; example uses differing values; smoke keeps +1 and larger id. | superseded by round 4 |
| 3 | database-designer | 3-D5 (N5): distroless CA bundle vs Azure chain | Accept. Risk line in Azure hosting: bake root PEM plus `sslrootcert`, keep `verify-full`. | resolved; detail deferred |
| 3 | database-designer | 3-D6 (N6): "explicit constraint names" vs unnamed PKs | Accept. PKs named `pings_pkey`, `ping_counter_pkey` in the DDL. | superseded by round 4 |
| 4 | user | A1-1: "prepare code for migrations but no need for schema now. Just for backend healthcheck run database health check as well." Remove pings table, counter, `POST /api/v1/pings`, button, `internal/ping`, `0001_create_pings.sql` and all that only supported them | Accept (user decision). Removed outright, incl. CTE, `.UTC()` rule, id/total contract, ping error classification, smoke id/total assertions. Earlier rows on these are history, marked superseded in their status column: round 1 G5, G6, G7, D2 (D5 in part); round 3 3-R2, 3-G5, 3-D3, 3-D4, 3-D6 (3-G3 in part). | resolved |
| 4 | user | A1-2: keep the migration mechanism, ship zero schema; resolve goose empty-FS behaviour | Accept. goose v3.30.1 source plus a scratch program outside the repo: `NewProvider` returns `ErrNoMigrations` on an empty FS. Chosen: no-op `0001_baseline.sql` (parser: 0 statements, no error). Runner, locker, `migrate up/status`, compose `migrate`, conventions kept. Live-DB check is in the integration test **[VERIFY]**. | resolved; live-DB check at build |
| 4 | user | A1-3: one health endpoint that also checks the DB (200 / enveloped 503), replacing `/healthz`; probes now depend on the DB | Accept. `GET /api/v1/health`, 200 `{"status":"ok","db":"ok"}` or 503 `db_unavailable`, `Pinger`, proxied by nginx `/api/`; compose, SPA and smoke share it; no `/readyz` (G6 superseded). Go-reviewer concern (DB outage -> unhealthy API, restart loops, opaque ingress 503) recorded as accepted trade-off; mitigation: DB-free liveness path later. | resolved; trade-off accepted by user |
| 4 | user | A1-4: frontend shows backend + database status; gates and tests updated | Accept. Auto-call on load plus "Check again"; union client, 30 s timeout, a11y kept; POST/ids/totals dropped. Old up/down/up gate replaced by migration and health integration tests via gatecheck; smoke updated. | resolved |
| 4 | user | A1-5: extension point replacing `ping.Store`; write endpoint and schema deferred | Accept. New features add `internal/<feature>` taking a pgx pool or small interface; `httpapi` depends on small interfaces via `Deps`. Write endpoint/schema wait for the first feature (database-designer returns); removed material not kept, no appendix. | resolved; deferred |
| 4 | react-developer | 4-R1 (H1): `aria-disabled` + in-flight ref rationale; ref stuck under StrictMode | Accept. Rationale (`disabled` loses focus) and `finally`/cleanup reset stated in A11y. | resolved |
| 4 | react-developer | 4-R2 (H2): tell unmount/StrictMode abort from own 30 s timeout | Accept. `timedOut` flag: set -> `timeout`; abort without it is silent. | resolved |
| 4 | react-developer | 4-R3 (H3): UI states for other envelope codes; "Check again" reset | Accept. Other codes -> Backend error, Database unknown; "Check again" resets to loading first. | resolved |
| 4 | react-developer | 4-R4 (H4): define `bad_response` and the success guard | Accept. 200 and `status === 'ok' && db === 'ok'`; any other non-envelope is `bad_response`. | resolved |
| 4 | react-developer | 4-R5 (H5): no `cache: 'no-store'` or header-adding fetch option | Accept. Client test asserts GET, no headers/body/`cache`; server `no-store` suffices. | resolved |
| 4 | react-developer | 4-R6 (H6): vitest coverage include/exclude, test globs, fake timers | Accept. In Frontend tooling; layout glob now `*.test.{ts,tsx}`. | resolved |
| 4 | react-developer | 4-R7 (H7, shared with 4-G6, 4-D6): A1-1 superseded list incomplete | Accept. See 4-D6. | resolved |
| 4 | react-developer | 4-R8 (H8): ingress 503 has no CORS headers, SPA shows "Backend unreachable" on DB outage | Accept. Clause added to the accepted trade-off. | resolved |
| 4 | database-designer | 4-D1 (H1): migration test wording "no application table" | Accept. Test asserts the `public` tables are exactly `{goose_db_version}` (bookkeeping). | resolved |
| 4 | database-designer | 4-D2 (H2): advisory-lock assertion via `pg_locks`, cluster-wide | Accept in part (user decision, 2026-11-21). Separate connection, scoped to the current database, asserts the lock is absent after `up`. "Held during `up`" dropped: the no-op baseline leaves no window to observe it. A blocking test (the test holds the lock, `up` must wait) is deferred to the first real migration. | resolved |
| 4 | database-designer | 4-D3 (H3): goose locker holds one pooled connection during `migrate` | Accept. Chosen: config validation requires `DB_MAX_CONNS >= 2` (no second pool, no new code path). | resolved |
| 4 | database-designer | 4-D4 (H4): migration conventions (names, FK types/`ON DELETE`/index, `CONCURRENTLY`, CREATEDB) | Accept. Added tersely to conventions as listed; baseline excepted from the name pattern. | resolved |
| 4 | database-designer | 4-D5 (H5): PG15+ non-owner roles cannot create in `public` | Accept. One sentence in Roles: migrate role owns the database or holds CREATE on `public`. | resolved |
| 4 | database-designer | 4-D6 (H6): stale log rows | Accept. Marked superseded: G5, G6, G7, D2, 3-R2, 3-G5, 3-D3, 3-D4, 3-D6; partly: D5, 3-G3. R5 left (still applies: "Check again" keeps the a11y rules). A1-1 list matches. | resolved |
| 4 | database-designer | 4-D7 (H7): baseline comment must not begin with `-- +goose` | Accept. Merged with 4-G4: comment never contains `+goose`. | resolved |
| 4 | database-designer | 4-D8 (H8): baseline immutable once applied | Accept. One clause in the baseline paragraph. | resolved |
| 4 | golang-developer | 4-G1 (H1): pgx `ConnectTimeout` default is 2 min | Accept. `ConnConfig.ConnectTimeout` 3 s set in code (not left to the DSN). | resolved |
| 4 | golang-developer | 4-G2 (H2): client-cancel detection; contract for cancelled requests | Accept. `r.Context().Err()` checked first; no body; fixed 499 in the log (API contract). | resolved |
| 4 | golang-developer | 4-G3 (H3): compose healthcheck timings for `api` and `web` | Accept. api `interval 2s, timeout 5s, retries 15, start_period 5s`; web `interval 2s, timeout 3s, retries 15, start_period 5s`. | resolved |
| 4 | golang-developer | 4-G4 (H4): `+goose` text only on the real `-- +goose Up` line | Accept. See 4-D7. | resolved |
| 4 | golang-developer | 4-G5 (H5): table set exactly `{goose_db_version}`; `status` also locks | Accept. In migration test and runner sentence. | resolved |
| 4 | golang-developer | 4-G6 (H6): superseded list | Accept. See 4-D6. | resolved |
| 4 | golang-developer | 4-G7 (H7): slow threshold 3 s, not 2 s | Accept. Slow state is >3 s so it does not flash on the server's 2 s DB deadline. | resolved |
| 4 | user | M1: migrations run in the deploy pipeline, never on API startup | Accept (user decision, 2026-11-21). Matches decision 3: `serve` never migrates, `halufka migrate` is a separate one-shot (compose `migrate` locally, a pipeline step on Azure). Azure wording changed from "Container Apps Job" to "pipeline step"; how the step reaches the DB (one-off Job or runner) is left to the infra phase. | resolved |
