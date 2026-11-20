# halufka - structure agreement (one-button walking skeleton)

Status: PROPOSAL (round 3)
Author: architect | Date: 2026-11-20 | Reviewers: golang-developer, react-developer, database-designer

**Recommendation.** React+TS SPA (Vite) -> Go API (stdlib `net/http` + `pgx`) -> PostgreSQL 17. Locally: `docker compose up --build` (db, one-shot migrate, api, nginx-served SPA with `/api` proxied, so same-origin, no Node on host). Azure (later phase): SPA on Static Web Apps Free, API as a scale-to-zero Container App, Postgres Flexible Server. The button `POST /api/v1/pings` inserts a row and returns id, timestamp and a running total read back from the DB.

**Round 2.** Scope cut hard: rate limiter, DB role split, `/readyz`, CORS preflight, cross-origin compose profile, CI YAML, govulncheck/race and long Azure detail moved to "Deferred / later" (under Decision log). Blockers R1 (nginx DNS) and G1 (coverage gate) are fixed in the design.

**Round 3.** Every gate is a plain command valid in PowerShell, cmd and Git Bash (`go -C backend ...`, no env prefixes, no `${PWD}`, no pipes). Lint and the coverage/skip check are Go tools run via `go tool` / `go run`; test and smoke settings are user-level Windows env vars, no code defaults.

**[VERIFY]** marks every pricing, quota or product-behaviour claim. All come from memory; none was checked online.

## Goal and non-goals

Goal: a tested skeleton proving browser -> API -> DB -> API -> browser locally, with clean extension points (layers, store interface, migrations, versioned API, error envelope) and a high-level Azure mapping.

Acceptance (testable):
1. With only Docker, `docker compose up --build`, open `http://localhost:3000`, click: shows id, timestamp and a total that rises by 1 per click, all read from Postgres.
2. The same API image runs on Azure with env-var changes only. The SPA bundle does not: `VITE_API_BASE_URL` is baked at build time, so the SPA is rebuilt per environment (runtime config deferred).
3. All quality gates below pass locally.

Non-goals: auth, users, any hall-ball domain model, realtime, i18n, PWA, custom domain, multi-region, caching, queues, metrics/tracing, building or running Azure infra/deploy in this run.

## Repository layout

```
halufka/
|-- README.md, .gitignore         existing; README gains a short quickstart incl. the one-time env setup; ignore node_modules/ dist/ coverage/
|-- .gitattributes                `* text=auto eol=lf` (Windows autocrlf would break Dockerfiles/nginx conf/SQL)
|-- .env.example, compose.yaml    documented env; services db, migrate (one-shot), api, web
|-- backend/                      Go module github.com/AGranosik/halufka/backend
|   |-- go.mod                    `go 1.26` + `toolchain go1.26.3`; `tool` directive pins golangci-lint (`go get -tool`); Dockerfile image Go >= 1.26
|   |-- Dockerfile, .dockerignore, .golangci.yml   golang:1.26 -> distroless nonroot, one image; config starts `version: "2"`, `run.build-tags: [integration, smoke]`, `formatters: enable: [gofmt]`
|   |-- cmd/halufka/main.go       entrypoint only: signal context, subcommand (serve|migrate|healthcheck), builds the listener from HTTP_ADDR, exit code
|   |-- internal/app/             Run(ctx, cfg, ln net.Listener): lifecycle (testable with a `:0` listener), http.Server timeouts, graceful drain; healthcheck client reads only HTTP_ADDR
|   |-- internal/config/          env parsing + validation, fail fast
|   |-- internal/httpapi/         router, handlers, middleware, error envelope
|   |-- internal/ping/            `Store` interface, `ErrUnavailable`, pgx implementation (only code writing ping SQL)
|   |-- internal/db/              pgxpool, goose Provider runner, `//go:embed migrations/*.sql` (0001_create_pings.sql)
|   |-- tools/gatecheck/main.go   gate 4 helper (`go run`): runs `go test -json`, fails on any skipped test or total < 80%
|   `-- smoke/smoke_test.go       build tag `smoke`: black-box test against the compose stack
`-- frontend/                     Vite + React 19 + TypeScript strict
    |-- package.json + package-lock.json (committed), Dockerfile, .dockerignore (node_modules dist coverage), default.conf (nginx `server{}` block only, `listen 8080`)
    `-- vite.config.ts, vitest.config.ts, tsconfig.json, eslint.config.js, src/ (main.tsx, App.tsx, api/pings.ts, *.test.tsx)
```
`infra/` and `.github/workflows/` are later phases.

## Components and boundaries

```mermaid
flowchart LR
  B[Browser] -->|static| W["nginx (local) / Static Web Apps (Azure)"]
  B -->|"POST /api/v1/pings"| A["API: Go, stateless"]
  W -.->|"local only: proxy /api/"| A
  A -->|"pgx pool"| D[("PostgreSQL 17")]
  M["migrate (one-shot)"] --> D
```

**Frontend.** One page, one button. States: idle / loading / slow (>2 s: "Waking the server up...") / success (id, time, total) / error (message, requestId if present). No business rules, no hard-coded URLs (only `VITE_API_BASE_URL`, empty = same-origin, trailing slash stripped), no auto-retry of the POST, no router/state lib/UI kit.
- Client returns a discriminated union `{kind:'ok',data} | {kind:'error',code,message,status?,requestId?}`. Non-envelope responses (nginx 502, ingress 503, SWA 404, HTML, network failure) map to `bad_response`/`network`; a 30 s `AbortController` maps to `timeout`. Success body checked by a hand-written guard. One client unit test asserts `fetch` is called with method `POST` and no headers/body (what keeps the request CORS-simple).
- A11y: `aria-disabled` plus an in-flight `useRef` guard (not only `disabled`); the `aria-live="polite"` result region is mounted from first render, empty, and only its text changes.
- Tooling: one `tsconfig.json` (no solution-style `references`, which makes `tsc --noEmit` check nothing): `strict`, `noEmit`, `include: [src, vite.config.ts, vitest.config.ts]`; the builder confirms gate 5 fails once on a deliberate type error. Scripts `typecheck` (`tsc --noEmit`), `lint` (`eslint . --max-warnings 0`), `test:ci` (`vitest run --coverage`), `build`. Coverage thresholds (lines/statements 80, branches 70) live in `vitest.config.ts`.

**Backend.** Validates config, serves the API, owns all SQL and the pool, logs JSON (`slog`), drains on SIGTERM. No SPA serving, no implicit migrations in `serve`, no per-request state, no DSN in logs, no ORM/framework; handlers see only `ping.Store` (fakeable). `/healthz` is not proxied by web.
- `internal/app.Run` builds the pool and `http.Server` (ReadHeaderTimeout 5 s, ReadTimeout 10 s, WriteTimeout 15 s, IdleTimeout 60 s), and on cancel calls `Shutdown` (8 s, under compose `stop_grace_period: 15s` on `api`), then closes the pool. `halufka healthcheck` reads only `HTTP_ADDR` (not `config.Load`, which needs `DATABASE_URL`), GETs `http://127.0.0.1:<port>/healthz` (2 s timeout), exit 0/1 (distroless has no curl).

**Database.** Persists rows and constraints; no triggers, no extensions; reachable only by API and migrate. Ping tables belong to `internal/ping`; `migrations/` to database-designer.

## API contract

Public API under `/api/v1`. JSON `application/json; charset=utf-8`. Every response carries `X-Request-Id` (generated; a valid inbound one of <=64 chars `[A-Za-z0-9-]` is reused) and `Cache-Control: no-store`.

**`POST /api/v1/pings`.** No body (ignored, capped 1 KiB; browser sends no custom headers, so it is a CORS simple request, no preflight). One SQL statement inserts the row, bumps the counter and returns both.
`201`: `{ "id": 42, "createdAt": "2026-11-20T08:15:30.123456Z", "total": 40 }` (illustrative: `id` and `total` differ in general; camelCase, RFC 3339 UTC, clients ignore unknown fields). `internal/ping` calls `.UTC()` on the pgx `time.Time` (local zone otherwise); a unit test asserts the `Z` suffix.

Error envelope (every non-2xx incl. 404/405/panic): `{ "error": { "code": "db_unavailable", "message": "...", "requestId": "..." } }`

| Status | `code` | When |
|---|---|---|
| 404 | `not_found` | unmatched path |
| 405 | `method_not_allowed` | wrong method on a known path (`/api/v1/pings`, `/healthz`; sets `Allow`) |
| 503 | `db_unavailable` | store returned `ping.ErrUnavailable`; `Retry-After: 2` |
| 500 | `internal` | anything else (details in logs only) |

- Classification: `internal/ping` wraps (`%w`) as `ErrUnavailable` pool-acquire/connect failures (`errors.As` to `*pgconn.ConnectError` or `net.Error`), `context.DeadlineExceeded` (5 s per-request deadline) and `pgconn` errors of class 08, 57P01, 57P02, 57P03, 53300. `context.Canceled` (client disconnect) is neither: logged at info, no envelope, not counted as a server error. Handlers: `errors.Is(err, ping.ErrUnavailable)` -> 503, else 500. The store retries once only if `pgconn.SafeToRetry(err)` (request never sent, so no duplicate row).
- Routing (stdlib mux emits plain-text 404/405): register `POST /api/v1/pings`, method-less `/api/v1/pings` (enveloped 405, `Allow: POST`), `GET /healthz`, method-less `/healthz` (enveloped 405, `Allow: GET`), catch-all `/` (enveloped 404); the routing test covers `POST /healthz`. Middleware outermost first: request-id -> access-log -> CORS -> recover -> mux; recover re-panics `http.ErrAbortHandler`. CORS sets headers before `next`, so ACAO is present on 404/405/500/503 too; the log sees the final status.
- Idempotency: pings are additive; retries add rows, hence no client auto-retry. Real writes later need an `Idempotency-Key` policy.
- `GET /healthz` -> `200 {"status":"ok"}`: liveness only, never touches the DB (no restart loops on DB outage). Used by compose and Container Apps probes. `/readyz` deferred: POST already proves the DB, and an opaque probe 503 is worse than an enveloped 503.
- CORS: local is same-origin via nginx, none needed. On Azure (different origins) the API sends `Access-Control-Allow-Origin: <origin>` + `Vary: Origin` only when `Origin` exactly matches `CORS_ALLOWED_ORIGINS` (comma list, no wildcard, no credentials); empty = no headers. Compose default `http://localhost:3000` lets smoke test the path. No `OPTIONS` handling in v1.

## Database

`postgres:17-bookworm` (Debian, floating tag: minor updates arrive on pull, accepted) locally, since Alpine musl collation differs from Azure glibc. Azure must offer 17 **[VERIFY]**.

```sql
-- 0001_create_pings.sql (no Down section: forward-only in prod)
-- +goose Up
CREATE TABLE pings (
    id         bigint      GENERATED ALWAYS AS IDENTITY CONSTRAINT pings_pkey PRIMARY KEY,
    created_at timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE ping_counter (
    id    smallint CONSTRAINT ping_counter_pkey PRIMARY KEY CONSTRAINT ping_counter_singleton CHECK (id = 1),
    total bigint   NOT NULL
);
INSERT INTO ping_counter (id, total) VALUES (1, 0);
```
The whole data layer, one statement (implicit transaction, no `count(*)` on a public write path):
```sql
WITH ins AS (INSERT INTO pings DEFAULT VALUES RETURNING id, created_at),
     upd AS (UPDATE ping_counter SET total = total + 1 WHERE id = 1 RETURNING total)
SELECT ins.id, ins.created_at, upd.total FROM ins, upd;
```
The counter row briefly serialises concurrent pings: acceptable. Both tables go away via a contract migration shipped at least one release after the code stops using them (expand/contract).

Conventions: `timestamptz`, identity columns, snake_case, explicit constraint names, one concern per migration, expand/contract only (two API revisions overlap during rollout).

Migration: goose via `halufka migrate` (same image, never run by `serve`). Provider API (goose >= v3.21 **[VERIFY]**) with the Postgres session locker from `goose/lock` (mandatory); the Provider needs a `*sql.DB` (`pgx/v5/stdlib`, e.g. `OpenDBFromPool`) and the embedded FS via `fs.Sub(..., "migrations")`. Commands `up` and `status` only, no `down`; local reset is `docker compose down -v`. Direct port 5432, never a transaction-pooling proxy. Local: compose `migrate` runs after `db` healthy, `api` waits on `service_completed_successfully`. Azure: a Container Apps Job before the new revision. Integration test: `up` on a fresh DB, tables exist and counter seeded, second `up` is a no-op, and no session advisory lock remains after `up` returns (lock acquired and released).

Roles: DEFERRED. One role `halufka` for migrate and serve. Locally it is the image's superuser (hides permission errors); accepted for one throwaway table. On Azure use a non-admin owner role, never the server admin.

## Local development

Prerequisite: Docker with Compose v2. No Node, az or make. Host Go is for the fast loop, tests, smoke, lint (golangci-lint comes from go.mod). No bind mounts (Windows traps); use `--build`. One command: `docker compose up --build` (`down -v` resets the DB).

| Service | Image / build | Host port | Notes |
|---|---|---|---|
| `db` | `postgres:17-bookworm`, volume `halufka-pgdata` | `127.0.0.1:${DB_HOST_PORT:-5432}` | healthcheck `pg_isready -h 127.0.0.1 -U halufka -d halufka` (TCP probe fails during image init because the init server is socket-only, `listen_addresses=''`); `interval: 2s, timeout: 3s, retries: 30, start_period: 10s` (cold initdb fits the 120 s budget) |
| `migrate` | `./backend`, `migrate` | - | one-shot, exits 0 |
| `api` | `./backend`, `serve` | `${API_PORT:-8080}` | healthcheck `halufka healthcheck`; `stop_grace_period: 15s`; waits for `migrate` |
| `web` | `./frontend` runtime | `${WEB_PORT:-3000}` -> 8080 | waits for `api` healthy; healthcheck `wget -q --spider http://127.0.0.1:8080/` |

Env (compose defaults, documented in `.env.example`): `DATABASE_URL` (required, no code default; local `postgres://halufka:halufka@db:5432/halufka?sslmode=disable`), `HTTP_ADDR` `:8080`, `CORS_ALLOWED_ORIGINS` (compose `http://localhost:3000`, code default empty), `DB_MAX_CONNS` 5, `DB_MIN_CONNS` 0, `LOG_LEVEL` info, `VITE_API_BASE_URL` (build arg, empty), `POSTGRES_USER/PASSWORD/DB` (`halufka`), port overrides.

Frontend container:
- Runtime `nginxinc/nginx-unprivileged:<x.y>-alpine` (exact tag) **[VERIFY tag]**; `default.conf` (a `server{}` block only, `listen 8080`) copied to `/etc/nginx/conf.d/default.conf`.
- R1 fix: a literal `proxy_pass http://api:8080` is resolved once at startup and cached. Use Docker's resolver and a variable: `resolver 127.0.0.11 valid=10s ipv6=off;`, `set $api_upstream api:8080;`, `location /api/ { proxy_pass http://$api_upstream; }` (no URI part, path passes unchanged). `location /` serves `dist/`.
- Stages: `deps` (`npm ci`) -> sibling `check` (typecheck, lint, tests) and `build` (`ARG VITE_API_BASE_URL`) -> `runtime` (copies `dist` from `build`). BuildKit skips unreferenced stages, so gates build `--target check` explicitly. Optional `FROM scratch AS coverage` exports via `--output type=local,dest=frontend/coverage`.
- Lockfile bootstrap, once, no mounts: a Dockerfile stage `lockfile` (`node:24-alpine`, `npm install --package-lock-only`, then `FROM scratch` exporting `package-lock.json`; Node 24 LTS **[VERIFY]**, same tag as `deps`), run as `docker build --target lockfile --output type=local,dest=frontend ./frontend`; commit the lockfile.

One-time setup (README quickstart): config for tests and smoke lives in Windows user-level environment variables, set once. No defaults in Go code, no `.env` loader.
```
setx TEST_DATABASE_URL "postgres://halufka:halufka@localhost:5432/halufka?sslmode=disable"
setx SMOKE_BASE_URL "http://localhost:3000"
setx SMOKE_API_URL "http://localhost:8080"
```
`setx` affects only processes started afterwards: restart terminals and the IDE. A missing variable makes the test FAIL with a message naming it (never skip). If `DB_HOST_PORT`/`API_PORT`/`WEB_PORT` are overridden, update the matching variable. CI will set the same variables in its workflow.

Tests: Go unit `go -C backend test ./...` (no DB). Go integration and the gate-4 sequence are in gate 4; a helper builds the name with `pgx.Identifier{"test_<random>"}.Sanitize()` for `CREATE DATABASE` (one documented `//nolint:gosec` on that line), migrates it, and in `t.Cleanup` closes its pool then runs `DROP DATABASE ... WITH (FORCE)`. Frontend: `docker build --target check ./frontend`. Fast backend loop: `docker compose up -d db migrate`, then `go -C backend run ./cmd/halufka serve` with `DATABASE_URL` (at `localhost`) set in the current shell.

## Azure hosting

High level; nothing built in this run.

| Component | Azure service | Notes |
|---|---|---|
| SPA | Static Web Apps, Free | built with `VITE_API_BASE_URL=https://<api-fqdn>` |
| API | Container Apps Consumption, 0.25 vCPU / 0.5 GiB, min 0, max 2 | image on ghcr.io; external ingress; startup + liveness probes on `/healthz` |
| Migrations | Container Apps Job (manual) | same image, `migrate` |
| DB | Flexible Server, Burstable B1ms, PG 17 | public access + TLS + firewall |
| Cost guard | Cost Management budget + alert | **[VERIFY]** free |

Free-tier facts (all **[VERIFY]**): SWA Free has bandwidth/size caps, no announced expiry; Container Apps has a recurring monthly free grant (about 180k vCPU-s, 360k GiB-s, 2M requests); the Postgres B1ms free benefit lasts 12 months from sign-up, then bills (roughly 12-20 USD/month); Azure SQL has a lifetime free offer but another engine (rejected, decision 1).

Networking/secrets: public Postgres endpoint, TLS, random password for a non-admin role, firewall "allow Azure services" **[VERIFY]** (accepted weakness; VNet later). Azure DSN uses `sslmode=verify-full` with system roots **[VERIFY CA chain]**; local uses `disable`. Risk: the distroless CA bundle may lack the Azure server chain; fallback is baking the root PEM into the image and setting `sslrootcert` (keep `verify-full`). `DATABASE_URL` is a Container Apps secret (`secretRef`); no Key Vault in v1. Pool: `DB_MAX_CONNS=5`, `MaxConnIdleTime=2m`, `MaxConnLifetime=30m`. Before raising max replicas check `2 x maxReplicas x DB_MAX_CONNS + 5 <= 80% of max_connections` (2 = old and new revision overlap; B1ms limit **[VERIFY]**).

Cold start: at 0 replicas the first request waits for pull + start + first DB connection (a few to ~10 s; measure later). Mitigations: tiny image, lazy pool, "waking up" UI after 2 s, 30 s timeout. Fallback `minReplicas=1` probably exceeds the free grant **[VERIFY]**. Exit cost: static SPA, plain OCI image, plain Postgres (`pg_dump | pg_restore`); calendar reminder at month 10 for the Postgres bill.

## Scalability

- Horizontal: stateless API replicas (HTTP concurrency scale rule, max 2 as cost cap) and a static SPA at the edge.
- Statelessness rules: no cross-request state in memory/disk, no sticky sessions, config from env only, SIGTERM drains, expand/contract migrations, every handler has a context deadline.
- DB: one pool per replica (budget above); scale vertically first; the rest is deferred.
- Abuse: with the limiter deferred, the public POST is bounded by max 2 replicas, `DB_MAX_CONNS`, the 5 s deadline, the O(1) statement and the budget alert.
- Observability: JSON access log per request (`ts, level, msg, request_id, method, path, status, duration_ms`), error log with `request_id`, `/healthz`, Container Apps built-in metrics.

## CI/CD

Later, a small `.github/workflows/ci.yml` runs the gates below on push/PR (Postgres service `postgres:17-bookworm`; workflow env sets `TEST_DATABASE_URL` for gate 4 and the `SMOKE_*` variables; pin actions). Deploy (`deploy.yml`, with infra, manual trigger): push image, OIDC login, run migrate Job, update revision, build/deploy SPA, post-deploy smoke. Rollback = reactivate previous revision (safe: expand/contract).

## Quality gates for the build phase

From repo root. Every command is a single plain command, identical in PowerShell, cmd and Git Bash (`go -C backend` instead of `cd`; no `&&`, env prefixes, `${PWD}`, `-w`, pipes or redirects). Each must exit 0. Variables come from the one-time setup (Local development). Order: gates 1-3, 5, 6 any time; gate 4 strictly before gate 7, never in parallel (both use the host DB port and gate 7 ends with `down -v`).

| # | Gate | Command(s), in order | Threshold |
|---|---|---|---|
| 1 | Go format | none separate: `formatters: enable: [gofmt]` in `.golangci.yml`, enforced by gate 3 | unformatted file = gate 3 fails |
| 2 | Go vet | `go -C backend vet ./...` | clean |
| 3 | Go lint | `go -C backend tool golangci-lint run` (tool pinned in go.mod, no Docker) | zero findings (govet, staticcheck, errcheck, errorlint, gosec, bodyclose, rowserrcheck, sqlclosecheck, gofmt). The golangci-lint project officially prefers a binary install; go.mod tool deps are accepted for minimal scope. Same mechanism for any other Go tool. |
| 4 | Go tests + coverage | `docker compose up -d --wait db`; then `go -C backend run ./tools/gatecheck` (needs `TEST_DATABASE_URL`) | `gatecheck` runs `go test -tags=integration -covermode=atomic -coverpkg=./internal/... -coverprofile=coverage.out -json ./internal/...` then `go tool cover -func=coverage.out`; exit 1 (message printed) if any test event has action `skip`, any test fails, or the `total:` line is < 80.0%; exit 0 otherwise. Only `./internal/...` measured; no per-package gate. |
| 5 | Frontend check | `docker build --target check ./frontend` | strict tsc clean, eslint 0 warnings, vitest thresholds |
| 6 | Frontend build | `docker build --target runtime ./frontend` | succeeds |
| 7 | Compose smoke | `docker compose build`; `docker compose up -d --wait`; `go -C backend test -tags=smoke -count=1 ./smoke/...` (needs `SMOKE_BASE_URL`, `SMOKE_API_URL`); `docker compose down -v` (cleanup, run even on failure) | `--wait` plus assertions finish within 120 s after images are built (pulls and builds excluded) |

Smoke assertions: `GET /` 200 HTML; `GET {api}/healthz` 200; POST via web -> 201, `id>0`, RFC 3339 `createdAt`; second POST `total == first.total + 1`, larger `id`; `GET /api/v1/nope` -> 404 envelope; `POST {api}/api/v1/pings` with `Origin: http://localhost:3000` returns that ACAO, with `Origin: http://evil.test` returns none; `GET {api}/api/v1/nope` with the allowed origin returns the 404 envelope plus ACAO. `up --wait` with a one-shot service needs checking on the installed Compose **[VERIFY]**; fallback `up -d` and poll `/healthz`.

Build order: (1) database-designer: migration; (2) golang-developer: config, db, ping, httpapi, app, main, tests, Dockerfile, lint, `tools/gatecheck`; (3) react-developer: scaffold, App, client, tests, Dockerfile, default.conf, lockfile; (4) integrator: compose, `.env.example`, git files, smoke, README, gate 7. Contract frozen once reviewers agree; later changes go through the objection log.

## Decision log

Only hard-to-reverse decisions.

| # | Decision | Rejected | Why |
|---|---|---|---|
| 1 | PostgreSQL 17 | Azure SQL free offer, Cosmos | User prefers Postgres; local parity and `pg_dump` exit. Price: free only 12 months **[VERIFY]**. Cheapest to switch now. |
| 2 | `/api/v1` prefix, unprefixed `/healthz`, JSON error envelope with stable `code` | unversioned; RFC 9457 | Public paths and error shape are hard to change once clients exist. |
| 3 | One-shot forward-only migrations (goose, session lock), expand/contract | migrate on start; prod down migrations | No replica races; no revision-overlap breakage. |
| 4 | SPA on SWA + API on Container Apps | single container with `go:embed`; App Service F1 | Edge-served page, independent deploys. Fallback: run the `web` image as a second Container App. |

### Deferred / later

- Rate limiting (token bucket, then per-IP/shared; POST only, never probes): bounded by max replicas + budget alert for now.
- DB role split (`halufka_owner`/`halufka_app`, `db/bootstrap.sql`, `MIGRATE_DATABASE_URL`): pointless for one throwaway table; do with the first real table.
- `/readyz` and readiness probes.
- CORS preflight/`Allow-Headers`/`Max-Age`/expose headers; `web-cors` compose profile for a browser cross-origin test.
- Runtime-injected SPA config so one bundle serves all environments.
- govulncheck, `-race`, per-package coverage via a tool, full CI YAML and deploy workflow.
- PgBouncer, read replicas, zone-redundant HA.
- VNet + private endpoint, Key Vault, outbound-IP firewall rule.
- OpenTelemetry/metrics, sqlc/chi (at ~10 queries/routes), caching, queues, realtime, auth.
- `infra/main.bicep`, Azure bootstrap, cold-start measurement, budget alert, custom domain.

## Open questions for the user

Defaults let the build proceed.
1. New Azure free account (12-month Postgres benefit **[VERIFY]**) and which region? Default: assume eligible; decide before infra.
2. After month 12: pay, move the DB, or switch engine? Default: stay on Postgres, decide at month 10.
3. Is the repo/ghcr.io image public? Default: public (no secrets in it).
4. Docker-only host and a neutral "Ping the database" button OK? Default: yes.
5. Budget alert threshold/e-mail? Default: 5 EUR/month.

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
| 1 | golang-developer | G5: limiter scope (POST only, X-Forwarded-For) | Accept, moot: limiter deferred; POST-only scope recorded in Deferred. Abuse bound noted in Scalability. | deferred |
| 1 | golang-developer | G6: `/readyz` opaque 503; use `/healthz` | Accept. Probes use `/healthz`; `/readyz` deferred. | resolved |
| 1 | golang-developer | G7: 503 vs 500 classification | Accept. `ping.ErrUnavailable` + `errors.Is` mapping. | resolved |
| 1 | golang-developer | G8: integration DB isolation; build tags in golangci | Accept. Per-test `CREATE DATABASE` helper; `run.build-tags` set. | resolved |
| 1 | golang-developer | G9: go directive and linter Go pinning | Accept. `go 1.26` + `toolchain go1.26.3`; images Go >= 1.26. | resolved |
| 1 | database-designer | D1: local superuser parity; role SPLIT decision (owner/app, bootstrap.sql, MIGRATE_DATABASE_URL) | Accept as DEFERRED, explicitly: split is out of minimal scope (one throwaway table, no sensitive data). Gap documented in Database; Azure still uses a non-admin owner. Revisit with first real table. | deferred |
| 1 | database-designer | D2: `count(*)` O(n) on public write | Accept. `ping_counter` row + single-statement CTE. | resolved |
| 1 | database-designer | D3: `pg_isready` passes during init | Accept. `pg_isready -h 127.0.0.1 -U halufka -d halufka`. | resolved |
| 1 | database-designer | D4: locker mandatory; migrate up/status only | Accept. Locker mandatory; `up`/`status`; no Down section. | resolved |
| 1 | database-designer | D5: connection budget with revision overlap; idle 2m; SafeToRetry | Accept, short form: overlap-aware budget rule, `MaxConnIdleTime=2m`, one retry on `SafeToRetry`. PgBouncer deferred. | resolved |
| 1 | database-designer | D6: verify-full not require | Accept. Azure DSN `verify-full` **[VERIFY CA chain]**; detail finalised in infra phase. | resolved; detail deferred |
| 1 | database-designer | D7: alpine musl vs Azure glibc collation | Accept. `postgres:17-bookworm` local and CI. | resolved |
| 3 | user | U1: lint via Go tool in go.mod, `go tool golangci-lint run`, no Docker/`-w`; config `version: "2"` | Accept (user decision). `tool` directive; officially-preferred binary install noted, go.mod tool deps accepted. | resolved |
| 3 | user | U2: TEST_DATABASE_URL / SMOKE_BASE_URL / SMOKE_API_URL as user-level env vars (`setx`), no code defaults, no `.env` loader, missing = FAIL | Accept (user decision). One-time setup block, setx restart note, CI later, port overrides need matching update. | resolved |
| 3 | user | U3: Postgres image stays floating `postgres:17-bookworm` | Accept (user decision). Wording removed; floating tag stated. | resolved |
| 3 | user | U4: no shell dependence in gates; shell-independent coverage/skip check | Accept (user decision). `go -C backend ...`; `tools/gatecheck` (`go run`) does `go test -json` skip check and 80% total; exit 0/1 defined. | resolved |
| 3 | golang-developer | 3-G1 IMPORTANT (N1): mixed shell dialects, `${PWD}`, `-w /app` | Accept. Resolved by U1/U2/U4; no gate command uses env prefixes, `${PWD}`, `-w`, `&&` or pipes. | resolved |
| 3 | golang-developer | 3-G2 IMPORTANT (N2): gate 4 not self-contained; unnamed threshold/skip commands; ordering vs gate 7 | Accept. Gate 4 lists `up -d --wait db`, env var, `gatecheck`; order stated (4 before 7, not parallel). | resolved |
| 3 | golang-developer | 3-G3 (N3): `context.Canceled`; 57P02/57P03; define connect failures | Accept. Canceled is logged at info, not a server error; classes added; `*pgconn.ConnectError` or `net.Error`. | resolved |
| 3 | golang-developer | 3-G4 (N4): `app.Run` testability; healthcheck must not call `config.Load` | Accept. `Run(ctx, cfg, ln net.Listener)`; healthcheck reads only `HTTP_ADDR`. | resolved |
| 3 | golang-developer | 3-G5 (N5): `POST /healthz` gives 404; recover and `ErrAbortHandler` | Accept. Method-less `/healthz` (enveloped 405, `Allow: GET`), routing test, recover re-panics `http.ErrAbortHandler`. | resolved |
| 3 | golang-developer | 3-G6 (N6): Shutdown 10 s vs compose 10 s grace | Accept. Shutdown 8 s, `stop_grace_period: 15s`, pool closed after Shutdown. | resolved |
| 3 | golang-developer | 3-G7 (N7): gosec G201/G202 on `CREATE DATABASE` | Accept. `pgx.Identifier{}.Sanitize()` plus one documented `//nolint:gosec`. | resolved |
| 3 | golang-developer | 3-G8 (N8): goose needs `*sql.DB`, `goose/lock`, `fs.Sub`; lock test | Accept. Stated in Database; migration test asserts no advisory lock remains. | resolved |
| 3 | golang-developer | 3-G9 (N9): `DROP DATABASE ... WITH (FORCE)`; port overrides | Accept. Pool closed then `DROP ... WITH (FORCE)` in `t.Cleanup`; overrides mirrored into env vars (U2). | resolved |
| 3 | react-developer | 3-R1 (N1): solution-style tsconfig makes `tsc --noEmit` check nothing | Accept. Single `tsconfig.json`, `include` set, strict, noEmit; builder proves gate 5 fails on a deliberate type error. | resolved |
| 3 | react-developer | 3-R2 (N2): "no preflight" needs no custom headers/body | Accept. One client unit test: `fetch` called with `POST`, no headers/body. | resolved |
| 3 | react-developer | 3-R3 (N3): "no localhost URL in bundle" has no command | Accept. Clause dropped from gate 6 (`VITE_API_BASE_URL` empty by default; scoped grep deferred). | resolved |
| 3 | react-developer | 3-R4 (N4): `nginx.conf` vs `conf.d/default.conf` | Accept. File renamed `default.conf`, `server{}` block only, `listen 8080`. | resolved |
| 3 | react-developer | 3-R5 (N5): no `${PWD}`/`-w` in remaining gates | Accept. None remain; lockfile bootstrap now a Dockerfile `lockfile` stage exported with `--output`, no mount. | resolved |
| 3 | react-developer | 3-R6 (N6): "120 s from cold" ambiguous | Accept. 120 s covers `--wait` plus assertions after images are built; pulls/builds excluded. | resolved |
| 3 | database-designer | 3-D1 (N1): text claims a pin, tag is floating | Accept. See U3. | resolved |
| 3 | database-designer | 3-D2 (N2): `pg_isready -d` rationale; healthcheck timings | Accept. Reworded (socket-only init server); `interval 2s, timeout 3s, retries 30, start_period 10s`. | resolved |
| 3 | database-designer | 3-D3 (N3): "tables dropped later" contradicts expand/contract | Accept. Drop is a contract migration at least one release after code stops using the tables. | resolved |
| 3 | database-designer | 3-D4 (N4): `timestamptz` local zone vs RFC 3339 UTC; `id` vs `total` | Accept. `.UTC()` in `internal/ping`, unit test for `Z`; example uses differing values; smoke keeps +1 and larger id. | resolved |
| 3 | database-designer | 3-D5 (N5): distroless CA bundle vs Azure chain | Accept. Risk line in Azure hosting: bake root PEM plus `sslrootcert`, keep `verify-full`. | resolved; detail deferred |
| 3 | database-designer | 3-D6 (N6): "explicit constraint names" vs unnamed PKs | Accept. PKs named `pings_pkey`, `ping_counter_pkey` in the DDL. | resolved |
