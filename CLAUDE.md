# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview
Local Docker Compose dev environment: PostgreSQL 16 + `test-app` (Node 20, TypeScript, Express 5, `pg`, `pino`) whose only job is to validate DB connectivity. Meant as a template to start real apps from. README.md and docs/setup.md (architecture, in Italian) are the user-facing docs.

## Three git repos, not one
- Root (`environment`): `docker-compose.yaml`, root `.env.example`, README, `docs/`.
- `database/`: its own repo (origin `w3bd3v3lop3rNico/postgres`).
- `test-app/`: its own repo (origin `w3bd3v3lop3rNico/test`).

The module folders are nested independent repos (not submodules) and appear untracked from the root. Commit module changes inside the module directory; each module has its own `.gitignore` excluding `.env`.

## Commands
Run Compose from the **repo root**. From a module directory the project name changes and Compose uses a different, empty volume (e.g. `database_pgdata` instead of `devenv_pgdata`).
- `docker compose up -d --build`: build and start everything; `docker compose ps` should show both services `(healthy)`
- `docker compose logs -f test-app`: JSON logs
- `docker compose exec postgres psql -U dev -d devdb`: SQL shell
- `docker compose down -v && docker compose up -d`: wipe the DB and re-run `database/init/*.sql`
- `curl localhost:3000/health` / `curl localhost:3000/db-check`: verify
- In `test-app/`: `npm run build` (tsc; the only type check), `npm run dev` (tsc --watch). No tests or linter are configured. Node is not needed on the host; builds happen in Docker.

Setup: copy `.env.example` → `.env` in the root, `database/`, and `test-app/`. Requires Compose v2.20+ (`include:`).

## Architecture
- **Root compose defines no services.** It `include:`s `database/compose.yaml` and `test-app/compose.yaml` with `env_file: .env`, so the root `.env` only drives interpolation (`COMPOSE_PROJECT_NAME=devenv`, `POSTGRES_PORT`, `APP_PORT`). Each module's own `.env` is passed into its container via `env_file`.
- **Networking:** both modules declare the same fixed-name network `dev-network`. The app reaches Postgres by service name (`DB_HOST=postgres`); `localhost` inside a container is the container itself. Published host ports are only for external clients.
- **Credentials must match:** `POSTGRES_*` in `database/.env` = `DB_*` in `test-app/.env`. Changing the password after the first start has no effect, because it is stored in the volume (use `ALTER USER` or `down -v`).
- **Init scripts** (`database/init/`, mounted at `/docker-entrypoint-initdb.d`) run once, in alphabetical order, only on an empty volume. `01-schema.sql` creates `healthcheck_log` and hardcodes the user `dev`.
- **test-app** is a single file, `src/index.ts`:
  - Config is validated at startup; a missing or invalid variable means a `fatal` "invalid configuration" log and exit 1. `DATABASE_URL` overrides `DB_*`. The password is never logged.
  - `/health` is liveness and is used by the container healthcheck (via busybox `wget`; no curl in alpine). It must not touch the DB.
  - `/db-check` is readiness. It queries `healthcheck_log`, so it depends on the init schema, and returns 503 on DB errors.
  - A `pool.on('error')` handler keeps DB restarts from crashing the process. SIGTERM/SIGINT trigger a graceful shutdown (10s forced-exit timer).
- **Start order:** `depends_on: postgres: condition: service_healthy, required: false`. The app waits for `pg_isready`, but `required: false` lets `test-app` run standalone without the database module.
- **Dockerfile** is multi-stage (tsc build → prod deps + `dist/`) and runs as user `node`.

## Adding a new app module
Copy `test-app/` to a new folder, rename the service and host port, add it to `include:` in the root `docker-compose.yaml`, keep `depends_on: postgres` and the `dev-network` network, and optionally add `database/init/02-<app>.sql` (runs only on an empty volume).
