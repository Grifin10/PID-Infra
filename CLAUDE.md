# PID — Infra (PID-Infra)

Shared context for Claude Code across the team. This file lives at the root of **PID-Infra**. Sibling files exist in **PID-Front** and **PID-Back** — see "Related repos" at the bottom.

## Project context
- University group web app project (React frontend + Fastify backend + Postgres), fully Dockerized.
- Professor's constraint: no managed/PaaS platforms — the app must run via Docker and be continuously deployed so it can be reviewed at any time.
- Three repos, cloned side by side inside a parent `Proyecto/` folder: `PID-Front`, `PID-Back`, `PID-Infra` (this one).
- Team works across Apple Silicon Macs (M1/M2) and at least one Windows desktop — keep cross-platform tooling in mind (line endings, shell scripts, etc.).

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
- Watch for `package-lock.json` accidentally getting committed into this repo — it belongs in `PID-Front`/`PID-Back` only, and has landed here by mistake from a wrong-directory `npm install`.

## Related repos
- **PID-Front** — Vite/React, dev port `5173`. Also holds the blue light/dark color palette reference. See its `CLAUDE.md`.
- **PID-Back** — Fastify, dev port `4000`, `"type": "module"`. See its `CLAUDE.md`.
