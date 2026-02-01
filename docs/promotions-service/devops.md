# Promotions Service — DevOps

> Last updated by: Task T01 — Overview, Settings & Configuration

## Overview

The Promotions Service uses Docker for containerization, with dual CI/CD pipelines on both **Azure Pipelines** and **GitHub Actions**. The release process uses **SemVer** versioning based on commit messages, pushes Docker images to Azure Container Registry (ACR) for production and GitHub Container Registry (GHCR) for development, and automatically bumps Helm chart versions for Kubernetes deployment.

## Docker

### Dockerfile

**Source:** `Dockerfile`

| Property | Value |
|----------|-------|
| **Base image** | `python:3.9.9-slim-bullseye` |
| **App user** | `mpay` |
| **App home** | `/com.mishipay` |
| **Exposed port** | `8001` |
| **Startup command** | `/bin/sh start.sh` |
| **Datadog service name** | `ms-promo` |

**Build stages:**
1. OS updates: `gcc`, `build-essential`, `ssh`, `git`
2. System dependencies: `default-libmysqlclient-dev`, `libpcre3-dev`, `libxml2-dev`, `libxslt-dev`, `zlib1g-dev`, plus debug tools (`htop`, `nano`, `lynx`, `lsof`, `curl`)
3. Python dependencies via Poetry (with SSH mount for private repos)
4. Copy source code to `/com.mishipay/service`
5. Run as non-root `mpay` user

**Key build features:**
- Uses `DOCKER_BUILDKIT=1` with SSH mount (`--ssh default`) to access private GitHub repos during `poetry install`
- Poetry virtualenv created in-project at `/requirements/.venv`
- `PYTHONUNBUFFERED=1` for real-time log output

### Docker Compose — Development

**Source:** `docker-compose.yml`

Three services for local development:

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `percona-mysql-db` | `percona:8.0.20-11-centos` | 3306 | MySQL database (dev only) |
| `ms-promo` | `ghcr.io/mishipay-ltd/ms-promo-ci:${CI_IMAGE_VERSION:-local}` | 8001 | Promotions service |
| `ddagent` | `gcr.io/datadoghq/agent:7` | 8126, 8125/udp | Datadog agent |

**Development specifics:**
- `ms-promo` runs `python3 manage.py runserver 0.0.0.0:8001` (Django dev server, not Gunicorn)
- Source code is bind-mounted (`./service:/com.mishipay/service:rw`) for live reload
- MySQL uses native password authentication with `--skip-mysqlx`
- Database initialized via `conf/mysql_init/000.sql`

### Docker Compose — Azure CI

**Source:** `docker-compose-azure-ci.yml`

Identical structure to development compose, but uses Azure Container Registry:
- Image: `acrhubprodwe01.azurecr.io/ms-promo:${CI_IMAGE_VERSION:-local}` (instead of GHCR)

### .dockerignore

Excludes from Docker context: `*.pyc`, `*.pyo`, `*.db`, `.git/`, `env`, `logs`, `conf`, `Dockerfile`, IDE files.

## Database Initialization

**Source:** `conf/mysql_init/000.sql`

```sql
CREATE DATABASE test_ms_promo CHARACTER SET utf8;
GRANT ALL PRIVILEGES ON test_ms_promo.* TO 'mpay_ms_promo'@'%';
FLUSH PRIVILEGES;
```

Creates a separate `test_ms_promo` database for unit tests. The main `ms_promo` database is created automatically by the Percona MySQL container via the `MYSQL_DATABASE` environment variable.

## Environment Configuration

### Template: ms-promo.env

**Source:** `conf/templates/ms-promo.env`

| Variable | Default Value | Description |
|----------|---------------|-------------|
| `DJANGO_SETTINGS_MODULE` | `promotions.settings` | Django settings module |
| `DEPLOYMENT_ENV` | `local` | Deployment environment (`local`, `prod`) |
| `HOSTNAME` | `localhost` | Service hostname |
| `SERVICE_NAME` | `ms-promo` | Service identifier |
| `MYSQL_DATABASE` | `ms_promo` | MySQL database name |
| `MYSQL_USER` | `mpay_ms_promo` | MySQL user |
| `MYSQL_PASSWORD` | `mishipay@123!` | MySQL password |
| `MYSQL_ROOT_PASSWORD` | `mishipay@123!admin` | MySQL root password |
| `MYSQL_DB_HOST` | `percona-mysql-db` | MySQL host (Docker service name) |
| `LOG_DIR` | `/com.mishipay/logs/` | Log file directory |

### Template: datadog.env

**Source:** `conf/templates/datadog.env`

| Variable | Value | Description |
|----------|-------|-------------|
| `DD_API_KEY` | `<datadog_key_here>` | Datadog API key (must be replaced) |
| `DD_SITE` | `datadoghq.eu` | Datadog EU site |
| `DD_LOGS_ENABLED` | `true` | Enable log collection |
| `DD_LOGS_INJECTION` | `true` | Inject trace IDs into logs |
| `DD_LOG_LEVEL` | `INFO` | Agent log level |
| `DD_APM_ENABLED` | `true` | Enable APM tracing |
| `DD_SERVICE` | `ms-promo` | Service name |
| `DD_ENV` | `local` | Environment tag |
| `DD_HOSTNAME` | `dd-agent` | Agent hostname |
| `DD_VERSION` | `0.1.0` | Service version |
| `DD_TRACE_ENABLED` | `true` | Enable distributed tracing |
| `MAX_TRACES_PER_SECOND` | `0` | Trace sampling rate (0 = unlimited) |
| `DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL` | `true` | Collect logs from all containers |
| `DD_CONTAINER_EXCLUDE_LOGS` | `name:dd-agent-promo` | Exclude agent's own logs |

## PostgreSQL Configuration

**Source:** `conf/postgresql.conf`

This file is a PostgreSQL configuration tuned for the **monolith's database**, not for the promotions service itself (which uses MySQL). It is present in this repo likely because the promotions service is deployed alongside the monolith. Key tuning parameters:

| Parameter | Value | Notes |
|-----------|-------|-------|
| `max_connections` | 256 | |
| `shared_buffers` | 2GB | |
| `temp_buffers` | 32MB | |
| `work_mem` | 32MB | |
| `maintenance_work_mem` | 512MB | |
| `effective_io_concurrency` | 450 | High — optimized for SSD |
| `max_worker_processes` | 2 | |
| `wal_buffers` | 16MB | |
| `random_page_cost` | 1.1 | Low — optimized for SSD |
| `effective_cache_size` | 6GB | |
| `autovacuum_vacuum_scale_factor` | 0.01 | Aggressive vacuuming |
| `autovacuum_analyze_scale_factor` | 0.02 | Aggressive analyze |

## Gunicorn Configuration

**Source:** `service/gunicorn.conf.py`

Production WSGI server configuration:

| Setting | Value | Notes |
|---------|-------|-------|
| `wsgi_app` | `promotions.wsgi:application` | |
| `bind` | `0.0.0.0:8001` | Configurable via `GUNICORN_BIND_ADDRESS` |
| `worker_class` | `gthread` | Thread-based workers |
| `workers` | `3` | Configurable via `GUNICORN_WORKERS` |
| `threads` | `4` | Configurable via `GUNICORN_THREADS` |
| `max_requests` | `3000` | Worker recycled after 3000 requests |
| `max_requests_jitter` | `250` | Jitter to prevent all workers recycling simultaneously |
| `keepalive` | `5` seconds | |
| `preload_app` | `True` | Preload app before forking workers |
| `loglevel` | `debug` (default) | Configurable via `GUNICORN_LOGLEVEL` |
| `capture_output` | `True` | Capture stdout/stderr |
| `statsd_host` | `ddagent:8125` | Sends metrics to Datadog via StatsD |
| `statsd_prefix` | `backend_v2` | Metric prefix |

Log format: `%(levelname)s | %(asctime)s | %(name)s | %(module)s | %(process)d | %(message)s`

## Development Scripts

### setup_env.sh

**Source:** `scripts/setup_env.sh`

Copies environment template files and creates log directory structure:
1. Copies `conf/templates/datadog.env` → `env/`
2. Copies `conf/templates/ms-promo.env` → `env/`
3. Creates `logs/` directory with `db`, `errors`, `requests`, `security` files
4. Sets `chmod 777` on logs directory

### build-docker.sh

**Source:** `scripts/build-docker.sh`

Builds the development Docker image using BuildKit with SSH forwarding:
```bash
DOCKER_BUILDKIT=1 docker build \
    --ssh default \
    --file "${PROJECT_DIR}/Dockerfile" \
    --tag ghcr.io/mishipay-ltd/ms-promo-ci:local \
    "$@" \
    "${PROJECT_DIR}"
```

### teardown-env.sh

**Source:** `scripts/teardown-env.sh`

Interactive cleanup script that:
1. Prompts for confirmation (destructive operation)
2. Stops all Docker containers and removes volumes: `docker-compose down --volumes --remove-orphans --rmi local`
3. Deletes `env/` directory and recreates it with `.gitkeep`

### wait_for_db.sh

**Source:** `scripts/wait_for_db.sh`

Simple database readiness check that polls MySQL on port 3306 up to 25 times with 1-second intervals.

## CI/CD Pipelines

### GitHub Actions Pipeline

**Source:** `.github/workflows/v1-pull.yml`

Triggered on:
- Pull requests (opened, edited, reopened, synchronized)
- Push to `master`

**Pipeline jobs (sequential):**

```
flake8 → metadata → build-ci → test → version → release
```

| Job | Runs On | Description |
|-----|---------|-------------|
| **flake8** | `ubuntu-24.04` | Code linting: critical errors (`E901,E999,F821,F822,F823`) + general (`F401,F405`) |
| **metadata** | `ubuntu-24.04` | Generates Docker image name/version from branch name |
| **build-ci** | `ubuntu-24.04` | Builds Docker image, uploads as artifact (uses Docker Buildx, GHA cache) |
| **test** | `ubuntu-24.04` | Spins up MySQL + service via docker-compose, runs `manage.py check`, `makemigrations --check`, `test --keepdb` |
| **version** | `ubuntu-24.04` | (master only) Bumps SemVer tag using `mathieudutour/github-tag-action`, creates GitHub release |
| **release** | `ubuntu-24.04` | (master only) Builds production image, pushes to GHCR, creates Sentry release |

**Key details:**
- Uses SSH deployment key for private repo access
- Docker image caching via GitHub Actions cache (`type=gha`)
- SemVer versioning based on conventional commits
- Sentry release creation on deploy

### Azure Pipelines

**Source:** `.azure/azure-pipelines.yml`

Triggered on:
- Push to `master`
- All pull requests

**Pipeline jobs (sequential):**

```
flake8 → metadata → build_ci → test → version → release → helm_bump
```

| Job | Pool | Description |
|-----|------|-------------|
| **flake8** | `ubuntu-20.04` | Same linting as GitHub Actions |
| **metadata** | `ubuntu-20.04` | Generates image name/version |
| **build_ci** | `ubuntu-20.04` | Builds Docker image, saves as tar artifact |
| **test** | `ubuntu-latest` | Uses `docker-compose-azure-ci.yml`, same test suite |
| **version** | `ubuntu-20.04` | (master only) Uses **GitVersion** for SemVer (unlike GitHub Actions which uses tag-action) |
| **release** | `adolinux` (self-hosted) | Builds and pushes image to **Azure Container Registry** (`acrhubprodwe01.azurecr.io/ms-promo`) |
| **helm_bump** | default | Bumps Helm chart version in `mishipay-ltd/helm-charts` repo, creates PR |

**Key differences from GitHub Actions:**
- Uses Azure Container Registry (ACR) instead of GHCR for production images
- Includes **Helm chart version bumping** — automatically creates a PR in the `helm-charts` repo to update the `promo-service` chart
- Uses **GitVersion** (configurable via `.azure/GitVersion.yml`) for SemVer calculation
- Release job runs on a **self-hosted agent** (`adolinux`)

### GitVersion Configuration

**Source:** `.azure/GitVersion.yml`

Commit message patterns for version bumping:

| Bump Level | Pattern | Example |
|------------|---------|---------|
| **Major** | `BREAKING CHANGE(scope):` | `BREAKING CHANGE: remove legacy API` |
| **Minor** | `feat(scope):` | `feat(easy-promo): add multi-quantity` |
| **Patch** | `build\|chore\|ci\|docs\|fix\|perf\|refactor\|revert\|style\|test(scope):` | `fix(combo): correct discount calc` |

## Deployment Architecture

```
Developer → Push to master
                │
    ┌───────────┼───────────────┐
    │           │               │
    ▼           ▼               ▼
 GitHub      Azure           Azure
 Actions    Pipelines       Pipelines
    │           │               │
    ▼           ▼               ▼
  GHCR       ACR             Helm
 (image)   (image)          Charts
                               │
                               ▼
                           Kubernetes
                           (via Helm)
```

**Production deployment flow (Azure):**
1. Merge to `master` triggers Azure Pipeline
2. Flake8 → Build → Test → Version (GitVersion SemVer)
3. Release: Build production image, push to ACR (`acrhubprodwe01.azurecr.io/ms-promo:vX.Y.Z`)
4. Helm bump: Clone `helm-charts` repo, bump `promo-service` chart version with new image tag, create PR
5. PR merged → Kubernetes picks up new Helm chart → Deploys new pods

**Production deployment flow (GitHub):**
1. Merge to `master` triggers GitHub Actions
2. Flake8 → Build → Test → Version (tag action)
3. Release: Push to GHCR, create Sentry release
4. (No Helm bump — handled by Azure pipeline)

## Docker Image Registries

| Registry | URL | Purpose |
|----------|-----|---------|
| **Azure Container Registry** | `acrhubprodwe01.azurecr.io/ms-promo` | Production images (Azure pipeline) |
| **GitHub Container Registry** | `ghcr.io/mishipay-ltd/ms-promo` | Release images (GitHub Actions) |
| **GitHub Container Registry** | `ghcr.io/mishipay-ltd/ms-promo-ci` | CI/dev images |

## Open Questions

- Is the Azure Pipeline the primary production deployment path, with GitHub Actions as secondary? Both run on push to master but only Azure handles Helm.
- What does the `adolinux` self-hosted agent pool consist of?
- What Kubernetes cluster/namespace is the service deployed to?
- Is the `postgresql.conf` in this repo actually used, or is it a leftover from a shared configuration approach?
