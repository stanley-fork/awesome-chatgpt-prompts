# Fix Docker setup: proper multi-container architecture with Docker Compose

## Summary

The previous Docker setup bundled PostgreSQL and the Next.js app into a single image managed by `supervisord`. This is an anti-pattern — it violates the "one container, one process" principle, clones the repo and runs the full build at runtime (3–5 min cold start), and makes scaling, updating, and debugging significantly harder.

This PR replaces it with a proper Docker Compose setup following the "Docker way":

- **`app`** — Next.js application container (multi-stage build, standalone output)
- **`db`** — Official `postgres:17-bookworm` container

Database data is persisted via a named Docker volume and survives container restarts, rebuilds, and image updates.

## Changes

### New files
- **`compose.yml`** — Docker Compose configuration with `app` and `db` services, healthchecks, and a named volume for PostgreSQL data
- **`docker/entrypoint.sh`** — Lightweight entrypoint: waits for the database, runs `prisma migrate deploy`, then starts the app

### Modified files
- **`docker/Dockerfile`** — Replaced the single-stage bootstrap image with a proper 4-stage multi-stage build:
  - `base` — Shared `node:24-bookworm-slim` with system dependencies
  - `deps` — Runs `npm ci` (and `prisma generate` via postinstall)
  - `builder` — Runs `npm run build`, produces `.next/standalone/`
  - `runner` — Minimal production image; copies standalone output, installs only Prisma CLI for migrations
- **`.dockerignore`** — Added `compose.yml`, test config files
- **`DOCKER.md`** — Complete rewrite covering the new compose-based workflow, standalone single-container usage, migration guide from the old setup, backup/restore, and production deployment tips

### Deleted files
- `docker/bootstrap.sh` — Replaced by `docker/entrypoint.sh`
- `docker/supervisord.conf` — No longer needed (one process per container)

## Key design decisions

- **PCHAT_* env vars work at runtime** — `src/lib/config/index.ts` already applies overrides via `applyEnvOverrides()`. No rebuild is needed to change branding or feature flags; just update environment variables and restart.
- **Migrations run at startup** — `prisma migrate deploy` runs in the entrypoint before the app starts, keeping deployments simple without init containers.
- **Seeding is manual** — `docker compose exec app npx prisma db seed`
- **All images are Debian-based** — `node:24-bookworm-slim` (bookworm) and `postgres:17-bookworm`
- **`.github/workflows/docker-publish.yml` needs no changes** — it already builds `./docker/Dockerfile` with context `.`

## Quick start after this PR

```bash
git clone https://github.com/f/prompts.chat.git
cd prompts.chat
docker compose up -d
```

Open http://localhost:4444

## Testing

- `docker compose build` — builds successfully
- `docker compose up -d` — both services start, DB healthcheck passes before app starts
- `curl http://localhost:4444/api/health` — returns `{"status":"healthy","database":"connected"}`
- `curl http://localhost:4444/` — homepage returns 200
- Data persists across `docker compose down && docker compose up -d`
