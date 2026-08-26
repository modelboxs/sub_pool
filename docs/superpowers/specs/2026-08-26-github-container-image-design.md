# GitHub Container Image Workflow

## Goal

Pushes to the existing `dev_pool` branch should build the repository's root
Dockerfile and publish a deployable application image to GitHub Container
Registry. PostgreSQL and Redis remain external services and are never started
by the production Compose file.

## Workflow

- Add `.github/workflows/container-image.yml`.
- Trigger image publishing on pushes to `dev_pool` and allow manual runs.
- Do not trigger on pull requests.
- Authenticate to `ghcr.io` with the repository `GITHUB_TOKEN` and package write
  permission.
- Build `linux/amd64` and `linux/arm64` using Docker Buildx and the root
  `Dockerfile`.
- Publish `latest` and an immutable `sha-<short SHA>` tag for `dev_pool` pushes.
- Use OCI labels and GitHub Actions cache to make images traceable and builds
  incremental.

The branch filter is intentionally kept in one visible `branches` entry so it
can be changed without altering the rest of the workflow.

## Compose Deployment

Update `deploy/docker-compose.yml` to contain only the `sub2api` service. The
PostgreSQL and Redis services, their volumes, network declarations, and
`depends_on` health conditions are removed. The application receives required
external endpoints through `DATABASE_HOST` and `REDIS_HOST` (with ports,
credentials, and TLS options remaining configurable through `.env`).

Keep `deploy/docker-compose.standalone.yml` as the explicit external-services
deployment example and use the GHCR image as its default image source.

## Operator Instructions

Document the following steps in the Docker deployment guide:

1. Ensure the repository allows GitHub Actions to write packages.
2. Push to `dev_pool` and monitor the `Container Image` workflow.
3. Make the GHCR package public or authenticate before pulling a private image.
4. Copy the example environment file, set external PostgreSQL/Redis hosts and
   credentials, and start the standalone Compose file.
5. Pin the immutable `sha-<short SHA>` tag when a reproducible deployment is
   required.

## Verification

- Validate workflow YAML syntax and required permissions/actions.
- Render the updated Compose file with representative external-service
  variables and confirm it contains only `sub2api`.
- Confirm no Compose service references `postgres` or `redis` through
  `depends_on`, service definitions, or internal host defaults.
- Run the repository's focused backend/frontend checks that are practical for
  the configuration-only change.
