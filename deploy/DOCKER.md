# Sub2API Docker Image

Sub2API is published as a multi-architecture image by GitHub Actions.

## GitHub Actions Build

The workflow in `.github/workflows/container-image.yml` runs only when code is
pushed to the `dev_pool` branch. Pull requests do not start this workflow.
Manual runs are available from the Actions tab.

Each push publishes:

```text
ghcr.io/modelboxs/sub_pool:latest
ghcr.io/modelboxs/sub_pool:sha-<short-commit-sha>
```

The image contains only the Sub2API application. PostgreSQL and Redis must be
provided separately and be reachable from the application container.

### Enable Package Publishing

In the GitHub repository, open **Settings > Actions > General**, find
**Workflow permissions**, select **Read and write permissions**, and save.
The workflow uses the automatic `GITHUB_TOKEN`; no personal access token is
needed to publish from Actions.

### Pull the Image

Public GHCR packages can be pulled directly. For a private package, create a
GitHub token with `read:packages` and log in before pulling:

```bash
echo "$GITHUB_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
docker pull ghcr.io/modelboxs/sub_pool:latest
```

Use the immutable SHA tag for a reproducible deployment:

```bash
docker pull ghcr.io/modelboxs/sub_pool:sha-<short-commit-sha>
```

## Docker Compose

The production Compose files start only `sub2api`. Configure external services
in `.env` before starting:

```dotenv
DATABASE_HOST=postgres.example.internal
DATABASE_PORT=5432
DATABASE_USER=sub2api
DATABASE_PASSWORD=replace-with-the-external-postgres-password
DATABASE_DBNAME=sub2api
DATABASE_SSLMODE=require

REDIS_HOST=redis.example.internal
REDIS_PORT=6379
REDIS_PASSWORD=replace-with-the-external-redis-password
REDIS_ENABLE_TLS=true
```

The `POSTGRES_*` variables in `.env.example` are retained for Apple container
deployments. Production Docker Compose uses the explicit `DATABASE_*` variables
shown above.

Start the external-service deployment with:

```bash
cp .env.example .env
# Edit .env and set the external PostgreSQL/Redis connection values.
docker compose -f docker-compose.standalone.yml up -d
```

For local-directory application data, use:

```bash
docker compose -f docker-compose.local.yml up -d
```

`docker-compose.dev.yml` is the exception: it intentionally starts local
PostgreSQL and Redis for development and integration testing.

## Startup and Database Recovery

Sub2API runs database migrations while starting. PostgreSQL may still be
recovering briefly after a host, container runtime, or database restart. The application
retries transient PostgreSQL startup and connection errors with bounded
exponential backoff, then continues startup when the database is ready.
Permanent errors such as invalid credentials, migration checksum mismatches,
SQL errors, and incompatible data fail immediately.

Because PostgreSQL is external to the production Compose deployment, ensure it
is healthy and reachable before starting Sub2API. Application-level retries
also cover brief interruptions while the external database recovers.

## Environment Variables

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `DATABASE_HOST` | Hostname or IP of external PostgreSQL | Yes | - |
| `DATABASE_PORT` | External PostgreSQL port | No | `5432` |
| `DATABASE_USER` | PostgreSQL user | Yes | `sub2api` |
| `DATABASE_PASSWORD` | PostgreSQL password | Yes | - |
| `DATABASE_DBNAME` | PostgreSQL database name | No | `sub2api` |
| `DATABASE_SSLMODE` | PostgreSQL TLS mode | No | `disable` |
| `REDIS_HOST` | Hostname or IP of external Redis | Yes | - |
| `REDIS_PORT` | External Redis port | No | `6379` |
| `REDIS_USERNAME` | Redis ACL username | No | - |
| `REDIS_PASSWORD` | Redis password | No | - |
| `REDIS_ENABLE_TLS` | Enable Redis TLS | No | `false` |
| `SERVER_PORT` | Server port | No | `8080` |
| `JWT_SECRET` | Persistent JWT signing secret | Recommended | Auto-generated |
| `TOTP_ENCRYPTION_KEY` | Persistent TOTP encryption key | Recommended | Auto-generated |

See `.env.example` for the complete configuration list.
