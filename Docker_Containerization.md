# Docker Containerization

This document explains the Dockerfile structure used for the `frontend` and `backend`, optimization techniques applied, and how to build and run images locally.

## Files

- `backend/Dockerfile` — builds a production Node.js image that runs `src/server.js` on port 3000.
- `frontend/Dockerfile` — multi-stage: builds the Vite React app in a Node image, then copies the static `dist` into an `nginx` image to serve on port 80.

## Key Dockerfile structure

- Use an official, pinned base image (e.g., `node:18-alpine`) to keep images small and predictable.
- Copy only `package.json` (and `package-lock.json` if present) before installing dependencies so Docker can cache `npm install` layers.
- Install production dependencies when possible (`npm ci --production` or `npm install --production`) for runtime images.
- For frontend, use a multi-stage build:
  - Stage 1: `node:18-alpine` build stage to `npm install` and `npm run build`.
  - Stage 2: `nginx:stable-alpine` to serve compiled static files, copying artifacts from the build stage.
- Add a `.dockerignore` to exclude `node_modules`, `.env`, `.git`, and other local-only files to reduce build context size.

## Image optimization techniques used

- Multi-stage builds: separate build tooling from runtime image to keep final images minimal.
- Use alpine-based images (`*-alpine`) to reduce size.
- Layer caching: copy `package.json` first, run `npm install`, then copy source — this prevents reinstalling deps unless package files change.
- Read-only mounts for init scripts (for Postgres) so initialization runs only on first-time container startup.
- Avoid including devDependencies in runtime images.
- Limit number of RUN layers by combining commands where appropriate.
- Use `EXPOSE` to document ports and (optionally) add a `HEALTHCHECK` instruction in production images to allow orchestration tools to detect unhealthy containers.

## Security & best-practices

- Do not bake secrets into images. Use environment variables or secrets management.
- Add `.env` to `.dockerignore` and pass runtime secrets via compose or orchestration secrets.
- Run containers as non-root where possible (not yet applied in these Dockerfiles; can be added with a created user).

## Build images locally

From project root, build backend image:

```bash
docker build -t bmi-health-backend:local -f backend/Dockerfile ./backend
```

Build frontend image (multi-stage already builds and produces a runtime nginx image):

```bash
docker build -t bmi-health-frontend:local -f frontend/Dockerfile ./frontend
```

Or use `docker-compose` (recommended for stack with DB):

```bash
docker-compose up --build
```

To run a built image directly:

```bash
# Backend (exposes 3000)
docker run --rm -p 3000:3000 \
  -e DB_HOST=host.docker.internal -e DB_PORT=5432 -e DB_USER=bmi_user -e DB_PASSWORD=change_me -e DB_NAME=bmidb \
  bmi-health-backend:local

# Frontend (exposes 80)
docker run --rm -p 8080:80 bmi-health-frontend:local
```

When running with `docker run`, set environment variables to point the backend to your database. When using `docker-compose`, the `docker-compose.yml` already wires `db`, `backend`, and `frontend` services together.

## Using `.env` with docker-compose

Create a `.env` file in the project root to override compose variables (not currently required by the provided compose file). Example contents:

```
POSTGRES_USER=bmi_user
POSTGRES_PASSWORD=change_me
POSTGRES_DB=bmidb
```

Start the stack:

```bash
docker-compose up --build
```

Remove volumes (warning: deletes DB data) and restart to reinitialize Postgres migrations:

```bash
docker-compose down -v
docker-compose up --build
```

## Optional improvements

- Add `HEALTHCHECK` lines to Dockerfiles so orchestrators can mark failing containers.
- Run backend migrations on startup (idempotent migration runner) instead of relying solely on Postgres init scripts.
- Switch to `npm ci` for reproducible installs in CI and local builds.
- Use a non-root user in runtime images.
- Use multi-arch images and automated CI that builds and publishes manifests.

If you want, I can update the Dockerfiles to add `HEALTHCHECK`, non-root users, and a simple migration runner in the backend.
