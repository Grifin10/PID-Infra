# PID — Infra (PID-Infra)

Shared context for Claude Code across the team. This file lives at the root of **PID-Infra**. Sibling files exist in **PID-Front** and **PID-Back** — see "Related repos" at the bottom.

## Project context
- University group web app project (React frontend + Fastify backend + Postgres), fully Dockerized.
- Professor's constraint: no managed/PaaS platforms — the app must run via Docker and be continuously deployed so it can be reviewed at any time.
- Three repos, cloned side by side inside a parent `Proyecto/` folder: `PID-Front`, `PID-Back`, `PID-Infra` (this one).
- Team works across Apple Silicon Macs (M1/M2) and at least one Windows desktop — keep cross-platform tooling in mind (line endings, shell scripts, etc.).

## App concept
A web app (responsive — must work well on phone too) that connects students and teachers.
- **Single account type**: role (`teacher` or `student`) is chosen at signup, not separate signup flows.
- **Teachers**: pick which subjects they teach from a fixed list of available subjects, and set their availability — specific dates and start times. Classes are always **1 hour long**, and can only start on the hour or half-hour (`:00` or `:30`).
- **Students**: search/browse teachers, view their profile and subjects, see which teachers are available and their open class slots.
- **Booking**: a student picking a slot **reserves it** — it disappears from availability for other students once booked.
- **Payments/pricing**: out of scope for this version — no external payment provider to provision for.

## Tech stack (this repo)
- **Docker + Docker Compose** for both local dev and production.
- **GitHub Container Registry (GHCR)** hosts built images.
- **GitHub Actions** CI/CD — cross-builds `linux/arm64` images using QEMU + Docker Buildx, since GitHub's runners are x86 but the target VPS is ARM.
- **Caddy** as the reverse proxy in production.
- Target VPS: **Oracle Cloud's Always Free ARM tier** (2 OCPU / 12GB RAM) — chosen for the permanent free tier and generous specs.
- Fallback providers under evaluation if Oracle capacity issues persist: Azure, AWS, GCP, DigitalOcean, Hetzner.

## Local dev ports
- Frontend: `127.0.0.1:5173`
- Backend: `127.0.0.1:4000`
- Postgres: `127.0.0.1:5432`

## Lessons learned / gotchas
- Oracle's Always Free ARM tier has repeatedly thrown "no available instances" (capacity contention) when trying to provision. If this happens, retry later or pivot to a fallback provider rather than losing time.
- `docker-compose.dev.yml` had a stale port mapping (`3000`) left over from before the frontend switched from Create React App to Vite (Vite's default is `5173`) — double-check port mappings after any frontend tooling change.
- Production frontend container is nginx-based with a `try_files` fallback so React Router routes work on refresh/direct URL.
- Postgres must **not** be exposed on a public port in production compose — only reachable from the backend container.
- `.gitattributes` is in place in all three repos to normalize line endings across Mac/Windows contributors.
- **After changing dependencies, rebuild with `-V` — `--build` alone is not enough.** Both app services mount an anonymous volume at `/app/node_modules` (so the host bind mount of the source doesn't shadow the container's install). Docker seeds that volume from the image **only the first time it is created**; after that it persists and hides whatever a rebuilt image contains. So `up --build` updates the image while the container keeps running the *old* dependency tree. Symptom: Vite fails with `Failed to resolve import "<some-package>"` even though the package is in `package.json` and the build just succeeded. Fix:
  ```
  docker compose -f docker-compose.dev.yml up --build -V
  ```
  `-V` (`--renew-anon-volumes`) refreshes only anonymous volumes — the named `pgdata_dev` volume is untouched. **Do not use `down -v` for this**: that deletes `pgdata_dev` and wipes the database. To confirm a volume is stale, compare the image against the running container:
  ```
  docker run --rm pid-infra-frontend sh -c 'ls /app/node_modules | wc -l'
  docker exec pid-infra-frontend-1 sh -c 'ls /app/node_modules | wc -l'
  ```
  Different counts = stale volume.
- **`npm ci` failing with `Cannot read properties of null (reading 'edgesOut')` during a compose build is not a Docker problem** — it means `package.json` and `package-lock.json` are out of sync in that app repo. `node:20-alpine` ships npm 10.8.2, which crashes with that unhelpful message instead of printing the usual "lock file out of sync" error. Fix in the app repo (not here): run `npm install`, then commit the regenerated lockfile.
- If `127.0.0.1:5173` and `localhost:5173` show **different apps**, another process is bound to `[::1]:5173`. On macOS `localhost` resolves to `::1` first, and a specific `[::1]` bind beats Docker's wildcard `*:5173` bind — so `localhost` serves the stray process (typically another project's forgotten `vite` dev server) while `127.0.0.1` serves our container. Diagnose with `lsof -nP -iTCP:5173 -sTCP:LISTEN`. This is why the ports above are written as `127.0.0.1`.
- Watch for `package-lock.json` accidentally getting committed into this repo — it belongs in `PID-Front`/`PID-Back` only, and has landed here by mistake from a wrong-directory `npm install`.

## Related repos
- **PID-Front** — Vite/React, dev port `5173`. Also holds the blue light/dark color palette reference. See its `CLAUDE.md`.
- **PID-Back** — Fastify, dev port `4000`, `"type": "module"`. See its `CLAUDE.md`.
