# CI/CD Setup for BMI Health Tracker

## Overview
This repository uses GitHub Actions as the CI/CD platform. Two workflows were added:

- `.github/workflows/frontend-ci.yml` — builds, lints, tests, produces a production build, and builds & pushes the frontend Docker image.
- `.github/workflows/backend-ci.yml` — installs, lints, tests, and builds & pushes the backend Docker image.

Both workflows are designed to run on pushes and pull requests to `main` and use repository secrets for Docker registry authentication.

## Pipeline Architecture

- Runner: `ubuntu-latest` GitHub-hosted runners.
- Stages (each workflow):
  1. Checkout repository (`actions/checkout`).
  2. Setup Node.js (`actions/setup-node`).
  3. Cache dependencies (`actions/cache`).
  4. Install dependencies (`npm ci`).
  5. Lint (conditional; only runs if a `lint` script exists in package.json).
  6. Run tests (conditional; only runs if a `test` script exists).
  7. Build production bundle (frontend) or ensure backend build artifacts exist.
  8. Log in to Docker registry (`docker/login-action`).
  9. Build and push Docker image (`docker/build-push-action`).

## Files Added

- [.github/workflows/frontend-ci.yml](.github/workflows/frontend-ci.yml)
- [.github/workflows/backend-ci.yml](.github/workflows/backend-ci.yml)
- [CI_CD_Setup.md](CI_CD_Setup.md)

## Environment and Secrets

Add these secrets in your GitHub repository settings → Secrets and variables → Actions:

- `DOCKERHUB_USERNAME` — Docker Hub username (or registry username).
- `DOCKERHUB_TOKEN` — Docker Hub access token (or registry access token/password).

If you prefer GitHub Container Registry (GHCR) or another registry, update the login and `tags` settings in the workflow files.

## Triggering the Pipelines

- Automatic: pushes and pull requests to `main` (adjust branches in workflow `on:` if desired).
- Manual: use the Actions tab, select the workflow, and click "Run workflow" (if you add `workflow_dispatch` to the workflow `on:` block).

## Monitoring and Logs

1. Go to the repository on GitHub → **Actions**.
2. Select the workflow run to view logs for each step.
3. Expand individual steps to see output, cached keys, test failures, and Docker build logs.

Notifications: you can set branch protection rules and required status checks to block merges until workflows succeed.

## Deploying Images

The workflows push images tagged as `latest` and by commit SHA. Pull and deploy these images from your infrastructure (or a CD workflow) using the appropriate tag.

Example image names (Docker Hub):

```
<DOCKERHUB_USERNAME>/bmi-health-frontend:latest
<DOCKERHUB_USERNAME>/bmi-health-backend:latest
```

## Troubleshooting & Recommendations

- If lint/test scripts are missing, the workflows will skip those steps — add `lint` and `test` scripts to `frontend/package.json` and `backend/package.json` for stricter checks.
- For integration tests that require Postgres, consider adding a job service in the workflow that starts a `postgres` service and runs tests against it.
- For secure credential handling in production, prefer GitHub Actions secrets, or a secrets manager integrated into your CD.

