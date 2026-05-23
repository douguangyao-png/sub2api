# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Sub2API is an AI API gateway that distributes upstream AI subscription quotas (Claude, OpenAI/Codex, Gemini, Antigravity, etc.) through platform-issued API keys, with authentication, billing, rate limiting, load balancing, and a built-in payment system. Go 1.26.3 backend + Vue 3 frontend, shipped as a single binary with the frontend embedded.

See also `DEV_GUIDE.md` for local-env quirks (Chinese), `backend/migrations/README.md` for migration rules, and `docs/PAYMENT.md` for payment configuration.

## Commands

Top-level `Makefile` orchestrates both sides:

```bash
make build              # backend + frontend
make test               # backend + frontend (lint + typecheck + critical vitest)
make test-frontend-critical  # only the auth/payment/settings vitest specs
```

### Backend (`backend/`)

```bash
go run ./cmd/server                    # run server (uses ./config.yaml)
go run ./cmd/server -setup             # CLI setup wizard
make build                             # → bin/server (no embedded frontend)
go build -tags embed -o sub2api ./cmd/server   # binary with embedded frontend

# Tests are split by build tag:
go test -tags=unit ./...               # unit tests (or: make test-unit)
go test -tags=integration ./...        # integration tests (uses testcontainers Postgres/Redis)
go test ./internal/handler/...         # run a package
go test -tags=unit -run TestFoo ./internal/service   # run a single test

golangci-lint run ./...                # lint (CI pins v2.9)
make generate                          # regenerate ent and wire code
```

CI (`.github/workflows/backend-ci.yml`) requires **Go 1.26.3** exactly and **golangci-lint v2.9**, and runs `make test-unit`, `make test-integration`, `make test-frontend`. Don't change Go version without updating `backend/go.mod`.

### Frontend (`frontend/`) — pnpm only

```bash
pnpm install                # use frozen lockfile in CI; locally just `pnpm install`
pnpm dev                    # Vite dev server on :3000, proxies /api /v1 /setup → http://localhost:8080
pnpm build                  # outputs to ../backend/internal/web/dist (consumed by `-tags embed`)
pnpm lint:check             # eslint
pnpm typecheck              # vue-tsc --noEmit
pnpm test:run               # vitest one-shot
```

`pnpm-lock.yaml` must be committed — CI uses `--frozen-lockfile`. Do not mix npm.

### Code generation that must be re-run and committed

- **Ent schema** (`backend/ent/schema/*.go`) → `cd backend && go generate ./ent`, commit the regenerated `backend/ent/`.
- **Wire DI** (`cmd/server/wire.go`, `internal/*/wire.go`) → `go generate ./cmd/server`, commit `wire_gen.go`.

## Architecture

### Backend layering

`cmd/server/main.go` → `setup` (first-run wizard) **or** `runMainServer` → `initializeApplication` (Wire-generated in `wire_gen.go`). The Wire graph composes provider sets from each layer:

```
config.ProviderSet
  → repository.ProviderSet  (ent client + Redis + S3 backup + caches)
    → service.ProviderSet   (~200 services; business logic, background workers)
      → payment.ProviderSet (provider registry: Stripe, Alipay, WeChat, EasyPay, Airwallex)
        → middleware.ProviderSet + handler.ProviderSet
          → server.ProviderSet (router, http.Server)
```

The Wire-built `Cleanup` func (in `cmd/server/wire.go`) is the canonical list of long-running background services — `TokenRefreshService`, `OpsAggregationService`, `BillingCacheService`, `ChannelMonitorRunner`, `UsageRecordWorkerPool`, `BackupService`, etc. When adding a service that owns a goroutine/cron, register a `Stop()` step there or the process won't shut down cleanly.

### Request flow

`internal/server/router.go` mounts:
- `routes.RegisterCommonRoutes(r)` — health/status (no `/api/v1` prefix).
- `/api/v1/auth|user|admin|payment` — JWT-authed admin/user endpoints (`routes/auth.go`, `user.go`, `admin.go`, `payment.go`).
- `routes.RegisterGatewayRoutes` — the upstream-proxy gateway (Anthropic/OpenAI/Gemini/Antigravity/Bedrock paths), authed by **API key** middleware, not JWT.
- Embedded frontend served by `internal/web` (only when built with `-tags embed`); injects public settings (and CSP nonces) into `index.html` at request time, with cache invalidation hooked into `SettingService` updates.

Middleware order matters: `Recovery → RequestLogger → Logger → CORS → SecurityHeaders` (CSP with dynamic `frame-src` for iframe-embedded admin extensions), then auth middleware per route group.

### Persistence

- **Postgres** via **Ent ORM** (`backend/ent/`) — schema files in `ent/schema/`, generated code in `ent/<entity>/`. Don't edit generated files; edit schema and regenerate.
- **Custom migration runner** (`internal/repository/migrations_runner.go`) auto-runs SQL files in `backend/migrations/` (embedded via `migrations.FS`) on startup. Tracked in `schema_migrations` with **SHA256 checksums** — once applied, files are immutable. Files ending in `_notx.sql` run outside a transaction and may only contain `CREATE/DROP INDEX CONCURRENTLY ... IF [NOT] EXISTS`. See `backend/migrations/README.md` before adding any migration.
- **Redis** for caching (api-key auth cache, billing cache, rate limits, concurrency control, scheduler outbox).

### Run modes & first-run setup

- `setup.NeedsSetup()` triggers a wizard server when `config.yaml` is missing; `AutoSetupEnabled()` (Docker) bootstraps from env vars instead.
- `config.RunModeSimple` ("simple") **disables billing/balance checks** — see the startup warning in `main.go`. New revenue/quota logic must check the run mode.

### Frontend

- Vue 3 + Vite + Pinia + vue-router + vue-i18n (runtime build aliased in `vite.config.ts` to avoid CSP `unsafe-eval`).
- API clients in `src/api/`, views in `src/views/{admin,user,auth,setup,public}/`, components in `src/components/`.
- Build output goes to `backend/internal/web/dist/` and is picked up by `//go:embed all:dist` in `internal/web/embed_on.go`. A non-embed build (no `-tags embed`) returns 404 for frontend paths.

## Gotchas

- **`-tags embed`** must be passed at build time to include the frontend; otherwise the binary is API-only.
- **Build tag isolation**: tests use `//go:build unit` or `//go:build integration`. Untagged `go test ./...` will skip both. Integration tests spin up real Postgres/Redis via testcontainers — they need Docker.
- **Interface changes** ripple into many test stubs/mocks under `internal/service` and `internal/handler` — grep for `Stub` / `Mock` after modifying an interface or the build will break in non-obvious places.
- **Don't modify an applied migration** — checksum will mismatch and startup will fail. Add a new migration instead.
- **Trusted proxies**: empty `server.trusted_proxies` in release mode disables X-Forwarded-For trust (logged as a warning). Set it when behind a reverse proxy.
- **Nginx + Codex CLI**: needs `underscores_in_headers on;` in the `http` block, otherwise sticky-session headers like `session_id` are dropped.
