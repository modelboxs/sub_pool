# GitHub Container Image Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and publish the application image to GHCR when `dev_pool` receives a push, while production Compose deployments use external PostgreSQL and Redis services.

**Architecture:** A dedicated GitHub Actions workflow owns container publishing and is independent from the tag-based release workflow. The production Compose manifests run only `sub2api`; database and cache endpoints are required environment inputs. The development Compose manifest remains a local integration stack and is documented separately.

**Tech Stack:** GitHub Actions, Docker Buildx, GHCR, Docker Compose, YAML, Markdown.

---

### Task 1: Add the push-to-GHCR workflow

**Files:**
- Create: `.github/workflows/container-image.yml`

- [ ] **Step 1: Define triggers and permissions**

Create a workflow named `Container Image` with `push.branches: [dev_pool]`,
`workflow_dispatch`, and no `pull_request` trigger. Set job permissions to
`contents: read` and `packages: write`.

- [ ] **Step 2: Configure metadata and registry login**

Check out the repository, set up QEMU and Buildx, derive the lowercase image
name from `${{ github.repository }}`, and log in to `ghcr.io` with
`${{ github.actor }}` and `${{ secrets.GITHUB_TOKEN }}`.

- [ ] **Step 3: Build and publish tags**

Use `docker/metadata-action` so pushes to `dev_pool` publish `latest` and
`sha-${{ github.sha }}` (short SHA), while manual runs publish the SHA tag.
Use `docker/build-push-action` with `context: .`, `file: ./Dockerfile`,
`platforms: linux/amd64,linux/arm64`, `push: true`, OCI labels, and GitHub
Actions cache scopes for the image.

- [ ] **Step 4: Commit the workflow**

Run `git diff --check`, then commit only the workflow with:

```bash
git add .github/workflows/container-image.yml
git commit -m "ci: publish container image on dev pool pushes"
```

### Task 2: Make production Compose use external services

**Files:**
- Modify: `deploy/docker-compose.yml`
- Modify: `deploy/docker-compose.local.yml`

- [ ] **Step 1: Replace internal service defaults**

Change the application image to `ghcr.io/modelboxs/sub_pool:latest` and change
`DATABASE_HOST=postgres` and `REDIS_HOST=redis` to required `${DATABASE_HOST:?DATABASE_HOST is required}` and `${REDIS_HOST:?REDIS_HOST is required}` values.

- [ ] **Step 2: Remove dependency wiring**

Delete each production manifest's `depends_on`, `networks`, `postgres`, and
`redis` sections, including their volumes and health checks. Keep the
application health check and its persistent `/app/data` volume.

- [ ] **Step 3: Update production quick-start comments**

Change comments and file headers so they say PostgreSQL and Redis must already
be reachable outside Compose. Remove instructions that create local database or
Redis data directories.

- [ ] **Step 4: Render both manifests**

Run `docker compose --env-file deploy/.env.example -f deploy/docker-compose.yml config` and the equivalent command for `docker-compose.local.yml` with temporary required host values. Confirm each output has only the `sub2api` service and no `depends_on`/`postgres`/`redis` service definitions.

- [ ] **Step 5: Commit Compose changes**

```bash
git add deploy/docker-compose.yml deploy/docker-compose.local.yml
git commit -m "deploy: use external postgres and redis services"
```

### Task 3: Document GHCR and external-service operations

**Files:**
- Modify: `deploy/DOCKER.md`
- Modify: `deploy/README.md`
- Modify: `deploy/.env.example`
- Modify: `deploy/docker-deploy.sh`

- [ ] **Step 1: Document GitHub Actions and GHCR setup**

Add the `dev_pool` push trigger, the `ghcr.io/modelboxs/sub_pool` image names,
the `latest`/SHA tag behavior, and the repository Settings → Actions → General
workflow permission requirement (`Read and write permissions`).

- [ ] **Step 2: Document private-package authentication**

Show `docker login ghcr.io` with a GitHub token that has package read access,
and explain that public packages do not require login.

- [ ] **Step 3: Document external database and Redis variables**

Add explicit `DATABASE_HOST` and `REDIS_HOST` entries to `.env.example`, mark
their credentials as external-service credentials, and show the standalone
Compose command:

```bash
docker compose -f docker-compose.standalone.yml up -d
```

- [ ] **Step 4: Align the preparation script**

Stop generating `POSTGRES_PASSWORD`, stop creating `postgres_data` and
`redis_data`, and print the required external-service configuration steps.

- [ ] **Step 5: Commit documentation changes**

```bash
git add deploy/DOCKER.md deploy/README.md deploy/.env.example deploy/docker-deploy.sh
git commit -m "docs: explain GHCR and external service deployment"
```

### Task 4: Verify the complete change

**Files:**
- Verify: `.github/workflows/container-image.yml`
- Verify: `deploy/docker-compose.yml`
- Verify: `deploy/docker-compose.local.yml`

- [ ] **Step 1: Validate YAML and shell syntax**

Run `ruby -e 'require "yaml"; YAML.load_file(".github/workflows/container-image.yml")'` and `bash -n deploy/docker-deploy.sh`.

- [ ] **Step 2: Validate Compose structure**

Render both production manifests with required external hosts and assert that
the service list is exactly `sub2api`.

- [ ] **Step 3: Check changed-file hygiene**

Run `git diff --check` and `git status --short`; ensure no unrelated files are
modified.

- [ ] **Step 4: Commit verification follow-up if needed**

If verification requires a correction, make the smallest fix, rerun all checks,
and commit it with a focused message.
