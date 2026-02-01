# Infrastructure — Docker & Service Architecture

> Last updated by: Task T36 — Kafka Consumers & Microservices

## Overview

The MishiPay platform runs as a set of Docker containers orchestrated via `docker-compose`. There are **4 compose files** for the monolith repo and **2 for the promotions service**, covering development, production, test, and CI environments. The system uses a **dual-network** topology to separate internet-accessible services from internal microservices.

This document covers Docker images, container configurations, networking, environment variable management, database initialization, the Nginx reverse proxy, Gunicorn application server, the Kong API gateway, and the differences between development, production, test, and CI setups.

---

## Docker Compose Files

### MishiPay Monolith

| File | Purpose | Services |
|------|---------|----------|
| `docker-compose.yml` | **Development** — Full local stack | 22 services: backend, 3 Kafka consumers, databases, microservices, gateway, static files |
| `docker-compose-prod.yml` | **Production** — Deployed stack | 7 services: backend, nginx, ms-tax (gRPC), memcached, Datadog, Keycloak, Kong |
| `docker-compose-test.yml` | **Test overlay** — Env overrides | Overrides env files for 8 services to use `*.test.env` variants |
| `docker-compose-ci.yml` | **CI overlay** — Image override | Overrides `backend_v2` to use CI-specific image tag |

### Promotions Service

| File | Purpose | Services |
|------|---------|----------|
| `docker-compose.yml` | **Development** — Local stack | 3 services: ms-promo, percona-mysql-db, ddagent |
| `docker-compose-azure-ci.yml` | **Azure CI** — CI variant | 3 services: identical structure, uses Azure Container Registry image |

---

## Service Catalog

### Development Environment (docker-compose.yml)

All 22 services in the monolith development stack:

#### Application Services

| Service | Container Name | Image | Port(s) | Database | Network(s) | Purpose |
|---------|---------------|-------|---------|----------|-------------|---------|
| `backend_v2` | `backend_v2` | `ghcr.io/mishipay-ltd/python-django-monolith-service:{version}` | 8080 (internal) | PostgreSQL (`backend_db_v2`) | `mpay_net` | Django monolith — main platform |
| `consumer_post_payment_success` | `consumer_post_payment_success` | Same as backend | — | Same as backend | `mpay_pub` + `mpay_net` | Kafka consumer: post-payment events |
| `consumer_common` | `consumer_common` | Same as backend | — | Same as backend | `mpay_pub` + `mpay_net` | Kafka consumer: multiplexed common events |
| `consumer_payment_only_clients_analytics` | `consumer_payment_only_clients_analytics` | Same as backend | — | Same as backend | `mpay_pub` + `mpay_net` | Kafka consumer: external payments analytics |

#### Microservices

| Service | Container Name | Image | Port(s) | Database | Network(s) | Purpose |
|---------|---------------|-------|---------|----------|-------------|---------|
| `ms-promo` | `ms-promo` | `ghcr.io/mishipay-ltd/ms-promo:v0.2.35` | 8001 | Percona MySQL (`ms_promo_db`) | `mpay_pub` + `mpay_net` | Promotions evaluation engine |
| `ms-loyalty` | `ms-loyalty` | `ghcr.io/mishipay-ltd/python-django-loyalty-service:v0.0.11` | 8002 | Shared MySQL (`ms_promo_db`) | `mpay_pub` + `mpay_net` | Loyalty & referral campaigns |
| `ms-tax` (gRPC) | `ms-tax` | `ghcr.io/mishipay-ltd/service.tax/ms_tax:1.0` | 9000 | Percona MySQL (`ms_tax_db`) | `mpay_net` | Legacy tax service (Go/gRPC) |
| `ms_tax` (HTTP) | `mishi-tax` | `ghcr.io/mishipay-ltd/mishipay-tax-service:v0.4.0` | 8008→80 | MySQL (`tax-db`) | `mpay_pub` + `mpay_net` | New tax service (FastAPI/HTTP) |
| `pay-gateway` | `mishi-payment-gateway` | `ghcr.io/mishipay-ltd/mishi-platform/ms_pay_gateway:1.59` | 3000 | — (proxies to `pay`) | `mpay_pub` + `mpay_net` | Payment gateway (Node.js) |
| `pay` | `mishi-pay` | `ghcr.io/mishipay-ltd/mishipay-payment-service:v1.12.9` | 8003 | MongoDB (`mongo-db`) | `mpay_pub` + `mpay_net` | Payment service (Python) |
| `voucher` | `mishi-voucher` | `ghcr.io/mishipay-ltd/mishi-platform/ms_voucher:1.2` | 8006→80 | MySQL (`voucher-db`) | `mpay_pub` + `mpay_net` | Voucher/gift card service |
| `inventory` | `inventory_service` | `ghcr.io/mishipay-ltd/inventory-service:v1.2.1` | — (internal) | MongoDB (`mongo-inventory`) + Elasticsearch (`elastic-inventory`) | `mpay_net` | Item inventory & search |

#### Infrastructure Services

| Service | Container Name | Image | Port(s) | Network(s) | Purpose |
|---------|---------------|-------|---------|-------------|---------|
| `ms_keycloak` | `ms_keycloak` | `ghcr.io/mishipay-ltd/service.auth/ms_keycloak:1.1` | 8005→8080 | `mpay_pub` + `mpay_net` | Keycloak identity provider |
| `ms_gateway` | `ms_gateway` | `ghcr.io/mishipay-ltd/service.auth/ms_gateway:1.1` | 80→8000, 443→8443, 8007→8001, 8444 | `mpay_pub` + `mpay_net` | Kong API Gateway |
| `static_files` | — | `nginx:1.21.1` | — | `mpay_pub` + `mpay_net` | Nginx reverse proxy / static file server |
| `mpay_rabbitmq` | `mpay_rabbitmq` | `rabbitmq:3.8.5-alpine` | — | `mpay_net` | RabbitMQ message broker |
| `mpay_redis` | `mpay_redis` | `redis:6.0.6-alpine` | — | `mpay_net` | Redis cache |

#### Database Services

| Service | Container Name | Image | Port(s) | Volume | Purpose |
|---------|---------------|-------|---------|--------|---------|
| `backend_db_v2` | `backend_db_v2` | `postgres:11.8` | 5432 | `pgdata_v2` | Main monolith + Keycloak database |
| `ms_promo_db` | `ms_promo_db` | `percona:8.0.20-11-centos` | 3316→3306 | `ms_promo` | Promo + Loyalty MySQL database |
| `ms_tax_db` | `ms_tax_db` | `percona:8.0.20-11-centos` | 3306 | `ms_tax` | Legacy Tax MySQL database |
| `tax-db` | `tax-db` | `mysql:8.0.25` | 9002→3306 | `tax_db_mysqld` | New Tax MySQL database |
| `voucher-db` | `mishi-voucher-db` | `mysql:8.0.25` | 9001→3306 | `voucher_db_mysqld` | Voucher MySQL database |
| `mongo-db` | `mongo-db` | `mongo:4.4.0` | — | `mongodb_data` | Payment service MongoDB |
| `mongo-inventory` | `mongo_inventory` | `mongo:4.4` | — | `mongo_inventory` | Inventory service MongoDB (replica set) |
| `elastic-inventory` | `elastic_inventory` | `elasticsearch:7.11.2` | — | `elastic_inventory` | Inventory search engine |

---

## Networking

### Dual-Network Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  mpay_pub (internal: false)                                     │
│  Open network — internet access for 3rd-party APIs              │
│  Production subnet: 172.34.0.0/16                               │
│                                                                 │
│  Connected: backend_v2, ms-promo, ms-loyalty, ms_tax (HTTP),    │
│             pay-gateway, pay, voucher, ms_keycloak, ms_gateway, │
│             static_files, Kafka consumers, ddagent               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  mpay_net (internal: true)                                      │
│  Private network — inter-service communication only             │
│  Production subnet: 172.28.0.0/16                               │
│                                                                 │
│  Connected: ALL services (every container is on mpay_net)       │
│  Internal-only: backend_db_v2, ms_promo_db, ms_tax_db, tax-db, │
│                 voucher-db, mongo-db, mongo-inventory,          │
│                 elastic-inventory, mpay_rabbitmq, mpay_redis,   │
│                 ms-tax (gRPC)                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Key design principle:** Database and message broker containers are **only** on `mpay_net` (private). Services that need to communicate with external APIs (Kafka, Sentry, payment gateways, Decathlon, etc.) are also on `mpay_pub`.

### Service Aliases

- `backend_v2` has the network alias `backend-v2` on `mpay_net` (and on `mpay_pub` in production)
- Other services are referenced by their container names (Docker DNS resolution)

### DNS Configuration

- `pay` (mishi-pay) has explicit DNS servers configured: `1.1.1.1`, `8.8.8.8`, `8.8.4.4` — needed for external payment gateway API resolution
- Production compose adds `extra_hosts` for Eroski API endpoints: `gersoa2prepro.eroski.es` and `gersoa2.eroski.es` → `172.30.227.254`

---

## Docker Images

### Monolith Dockerfile (`MishiPay/Dockerfile`)

Multi-step build on `python:3.9.9-slim-bullseye`:

| Stage | Description |
|-------|-------------|
| **OS packages** | `cmake gcc g++ build-essential ssh git`, then `nano lynx lsof curl htop postgresql-client lftp zip` |
| **Service deps** | `zlib1g libxslt libxml2 python3-dev libpq-dev gettext` |
| **Azure CLI** | Version `2.76.0-1~bullseye` — used by Django management commands in Airflow |
| **ODBC Driver** | Microsoft ODBC 17 for SQL Server connectivity |
| **Python deps** | Poetry-based install (`poetry install --no-root`) with SSH mount for private GitHub packages |
| **App code** | `./mainServer/` → `/app/service/`, `./conf/` → `/app/conf/` |
| **Runtime** | Gunicorn config + start script, runs as `mpay` user |

**Key details:**
- `APP_HOME=/app`, `PYTHONPATH=/app/service`
- Datadog environment: `DD_SERVICE=monolith-service`
- Virtual env at `/requirements/.venv`
- Exposes port **8080**
- Healthcheck: `python manage.py health_check` (20s interval, 15s timeout, 3 retries, 10s start period)

### Promotions Service Dockerfile (`service.promotions/Dockerfile`)

Multi-step build on `python:3.9.9-slim-bullseye`:

| Stage | Description |
|-------|-------------|
| **OS packages** | `gcc build-essential ssh git` |
| **Service deps** | `default-libmysqlclient-dev libpcre3-dev libxml2-dev libxslt-dev zlib1g-dev htop nano lynx lsof curl` |
| **Python deps** | Poetry-based install with SSH mount |
| **App code** | `./service/` → `/com.mishipay/service` |

**Key details:**
- `APP_HOME=/com.mishipay`
- Datadog environment: `DD_SERVICE=ms-promo`
- Runs as `mpay` user
- Exposes port **8001**
- Startup: `/bin/sh start.sh`

---

## Application Server — Gunicorn

Configuration at `conf/gunicorn/gunicorn.conf.py`:

| Setting | Value | Notes |
|---------|-------|-------|
| `wsgi_app` | `mainServer.wsgi:application` | Django WSGI entry point |
| `bind` | `0.0.0.0:8080` | Listens on all interfaces |
| `worker_class` | `gthread` | Thread-based workers |
| `workers` | `3` | 3 worker processes |
| `threads` | `4` | 4 threads per worker (12 total) |
| `max_requests` | `3000` | Worker recycling after 3000 requests |
| `max_requests_jitter` | `250` | Random jitter to avoid thundering herd |
| `keepalive` | `5` seconds | |
| `preload_app` | `True` | App loaded before forking |
| `user` / `group` | `mpay` | Non-root execution |
| `loglevel` | `debug` | **TODO in code**: should be `info` in production |
| `statsd_host` | `ddagent:8125` | Sends metrics to Datadog agent |
| `statsd_prefix` | `backend_v2` | Metric namespace |

The startup script (`conf/gunicorn/start.sh`) wraps Gunicorn with Datadog APM tracing:
```bash
exec ddtrace-run gunicorn --config /app/gunicorn.conf.py
```

---

## Reverse Proxy — Nginx

Nginx (`nginx:1.21.1`) serves as the static file server and reverse proxy.

### nginx.conf Key Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| `worker_processes` | `auto` | Matches CPU cores |
| `worker_rlimit_nofile` | `10240` | File descriptor limit |
| `worker_connections` | `1024` | Max connections per worker |
| `keepalive_timeout` | `300` seconds | Long-lived connections |
| `keepalive_requests` | `100000` | Requests per connection |
| `client_body_timeout` | `10` seconds | Request body read timeout |
| `send_timeout` | `10` seconds | Response send timeout |
| `server_tokens` | `off` | Hides Nginx version |
| `gzip` | `on` | Compression enabled (min 10240 bytes) |
| `limit_conn` | `20` per IP | Connection rate limiting |
| `limit_req` | `5r/s` burst `20` | Request rate limiting (custom 460 status) |

### Virtual Host (`conf.d/localhost.conf`)

- **Server names:** `test.mishipay.com`, `dev.mishipay.com`, `localhost`
- **SSL:** Cloudflare wildcard cert (`star.mishipay.com.crt/.key`)
- **Blocked routes:** `/item-management/v1/promotions` (returns 404), `/silk` (returns 404)
- **Static files:** Served directly from Nginx (bypasses Django):
  - `/static/` → `/static/` (monolith static files)
  - `/static-ms-loyalty/` → `/static-loyalty/`
  - `/static-ms-promo/` → `/static-promo/`
- **Dynamic proxy:** `mpay_proxy` config (commented out in local; active in production) proxies to `http://backend_v2:8080`

### Proxy Configuration (`mpay_proxy`)

```
proxy_pass http://backend_v2:8080;
```

Headers forwarded: `Host`, `X-Real-IP`, `X-Scheme`, `REMOTE_ADDR`, `Referer`, `X-Forwarded-For`, `X-Forwarded-Proto`, `HTTP_CF_IPCOUNTRY` (Cloudflare country header).

### Security Headers

```
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
```

### CORS Support

Origin-based CORS allowing `*.mishipay.org` and `*.mishipay.com` subdomains with all standard methods (GET, POST, PUT, DELETE, OPTIONS, HEAD). Custom headers allowed: `authorization`, `content-type`, `x-csrftoken`, `bundleid`, `uniquesessionid`, `x-authorization`, `appversion`, `bundleversion`, `transaction-token`. OPTIONS preflight returns 204.

---

## API Gateway — Kong

Kong (`ms_gateway`, `service.auth/ms_gateway:1.1`) acts as the external-facing API gateway.

### Port Mapping

| Host Port | Container Port | Purpose |
|-----------|---------------|---------|
| 80 | 8000 | HTTP Listener |
| 443 | 8443 | HTTPS Listener (SSL) |
| 8007 | 8001 | Admin API |
| 8444 | 8444 | Admin API (SSL) |

### Route Configuration (`conf/auth/kong/config.yml`)

| Kong Service | Upstream URL | Route | Plugins |
|-------------|-------------|-------|---------|
| `backend_v2` | `http://backend_v2:8080` | Hosts: `dev.mishipay.com`, `test.mishipay.com`, `localhost` | `jwt-keycloak` (token verification) |
| `keycloak` | `http://ms_keycloak:8005` | Hosts: `auth-dev.mishipay.com`, `auth-test.mishipay.com` | — |
| `pay-gateway` | `http://pay-gateway:3000` | Hosts: `pg-dev.mishipay.com`, `pg-test.mishipay.com` | — |
| `mishi-gateway-admin` | `http://127.0.0.1:8001` | Path: `/kong-admin` | `key-auth` |
| `static` | `http://backend_v2:8080/static` | Path: `/static` | — |
| `static_ms_promo` | `http://ms-promo:8001/static-promo` | Path: `/static-ms-promo` | — |
| `static_ms_loyalty` | `http://ms-loyalty:8002/static-loyalty` | Path: `/static-ms-loyalty` | — |

### JWT Authentication

The `jwt-keycloak` plugin verifies Keycloak-issued JWT tokens on the `backend_v2` service. Configuration:
- **Anonymous consumer ID:** `996f6f74-4233-4f45-b5ea-9209892facd1` — allows unauthenticated requests to pass through with anonymous identity
- **Allowed issuers:** `http://auth-{test,dev,local}.mishipay.com/auth/realms/mishipay`

### API Consumers

| Consumer | API Key | Purpose |
|----------|---------|---------|
| `mishi-gateway-admin` | `KongAdminKey` | Kong admin API access |
| `ios` | `iOSApiKey` | iOS mobile app |
| `android` | `androidApiKey` | Android mobile app |
| `web` | `webApiKey` | Web application |
| `dashboardWeb` | `dashboardWebApiKey` | Dashboard web app |
| `dashboardNative` | `dashboardNativeApiKey` | Dashboard native app |
| `whitelabelWeb` | `whitelabelWebApiKey` | White-label web |
| `whitelabelAndroid` | `whitelabelAndroidApiKey` | White-label Android |
| `whitelabeliOS` | `whitelabeliOSApiKey` | White-label iOS |
| `anonymous_users` | — (UUID-based) | Unauthenticated requests |

### SSL Configuration

Wildcard certificate for `*.mishipay.com` — placeholder values in the config (`INSERT_CERT_HERE` / `INSERT_KEY_HERE`) replaced at deployment.

---

## Keycloak Identity Provider

- **Image:** `ghcr.io/mishipay-ltd/service.auth/ms_keycloak:1.1`
- **Database:** PostgreSQL (shared `backend_db_v2` instance, database `ms_keycloak`, user `mpay_ms_keycloak`)
- **Port:** 8005 → 8080 (internal)
- **SSL:** Uses `star.mishipay.com` wildcard cert
- **Production ports:** Additional cluster ports exposed (7600, 57600, 45700, 55200, 54200, 23364, 45688) for JGroups cluster communication

---

## Database Architecture

### 5 Database Systems

```
┌──────────────────────────────────────────────────────────────┐
│                    PostgreSQL 11.8                            │
│                  (backend_db_v2)                              │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  mpay    │  │  test_mpay   │  │ ms_keycloak  │           │
│  │ (main)   │  │   (tests)    │  │   (auth)     │           │
│  └──────────┘  └──────────────┘  └──────────────┘           │
└──────────────────────────────────────────────────────────────┘

┌───────────────────────────┐  ┌───────────────────────────┐
│   Percona MySQL 8.0       │  │   Percona MySQL 8.0       │
│   (ms_promo_db)           │  │   (ms_tax_db)             │
│  ┌──────────┐ ┌─────────┐│  │  ┌──────────────┐         │
│  │ ms_promo │ │ms_loyalty││  │  │ test_ms_tax  │         │
│  └──────────┘ └─────────┘│  │  └──────────────┘         │
└───────────────────────────┘  └───────────────────────────┘

┌───────────────────────────┐  ┌───────────────────────────┐
│   MySQL 8.0.25            │  │   MySQL 8.0.25            │
│   (tax-db)                │  │   (voucher-db)            │
│  ┌──────────┐             │  │  ┌──────────────┐         │
│  │  ms_tax  │             │  │  │ mishi_voucher│         │
│  └──────────┘             │  │  └──────────────┘         │
└───────────────────────────┘  └───────────────────────────┘

┌───────────────────────────┐  ┌───────────────────────────┐
│   MongoDB 4.4.0           │  │   MongoDB 4.4             │
│   (mongo-db)              │  │   (mongo-inventory)       │
│   DB: mishi               │  │   Replica Set: rs0        │
│   Used by: pay service    │  │   Auth: keyfile + SCRAM   │
└───────────────────────────┘  │   Used by: inventory svc  │
                                └───────────────────────────┘

┌───────────────────────────┐
│  Elasticsearch 7.11.2     │
│  (elastic-inventory)      │
│  Single node, xpack auth  │
│  JVM: 2048m               │
│  Used by: inventory svc   │
└───────────────────────────┘
```

### Database Initialization

**PostgreSQL** (`automation/postgres_init/init.sh`):
1. Creates role `test_mpmain` with `CREATEDB` + `LOGIN`
2. Creates databases `mpay` and `test_mpay`
3. Restores from `mpay.dump` (main DB) and `test_mpay.dump` (test DB)
4. Dumps are updated quarterly — Django applies missing migrations on startup

**PostgreSQL configuration** (`conf/postgres/postgresql.conf`): Stock PostgreSQL configuration file (21K) — customizations not visible in the first 50 lines.

**MongoDB (Inventory)** (`automation/docker/mongo-inventory/`):
1. `mongo-entrypoint.sh` creates a keyfile for replica set authentication from env var `MONGO_KEYFILE_TEXT`
2. `001_mongo_init.js` initializes replica set `rs0`, creates root admin user, creates inventory service user with `readWrite` on both production and test databases

**MongoDB (Inventory) config** (`conf/mongo-inventory/mongod.yml`):
- Security: keyfile-based authentication
- Replication: replica set `rs0` with majority read concern

**MySQL databases** use `mysql_native_password` authentication plugin with `--skip-mysqlx` (X Protocol disabled). Init scripts are mounted from `automation/ms.{service}/mysqld_init/`.

---

## Environment Configuration

### Environment File Structure

Each service has up to 4 environment file variants:

| Suffix | Purpose |
|--------|---------|
| `.local.env` | Local development |
| `.dev.env` | Shared development server |
| `.test.env` | Test environment |
| `.prod.env` / `.prd.env` | Production |

### Monolith Environment Variables (`conf/local-variables.env.default`)

| Category | Variables | Description |
|----------|-----------|-------------|
| **Django** | `DEBUG`, `DJANGO_SETTINGS_MODULE`, `MPAY_ENV_TYPE`, `DEPLOYMENT_ENV` | App framework config |
| **Security** | `MPAY_SECRET_KEY`, `MPAY_HMAC` | Django secret + HMAC signing key |
| **Database** | `MPAY_DB_HOST`, `MPAY_DB_NAME`, `MPAY_DB_USER`, `MPAY_DB_PWD` | PostgreSQL connection |
| **Inventory Service** | `INVENTORY_SERVICE_{PROTOCOL,HOST,PORT}` | HTTP connection to inventory service |
| **Inventory Ops** | `INVENTORY_SERVICE_OPS_{PROTOCOL,HOST,PORT}` | Batch/bulk operations endpoint |
| **Legacy Tax (gRPC)** | `MS_TAX_HOST`, `MS_TAX_PORT` | gRPC at `ms-tax:9000` |
| **New Tax (HTTP)** | `MS_TAX_NEW_{PROTOCOL,HOST,PORT}` | HTTP at `ms_tax:80` |
| **Payment Service** | `MS_PAY_{HOST,PORT,PROTOCOL}` | HTTP at `pay:80` |
| **Kafka** | `CONF_KAFKA_BOOTSTRAP_SERVER`, `CONF_KAFKA_API_KEY`, `CONF_KAFKA_API_SECRET` | Confluent Cloud (West Europe) |
| **Promo Service** | `MS_PROMO_{PROTOCOL,HOST,PORT}` | HTTP at `ms-promo:8001` |
| **Loyalty Service** | `MS_LOYALTY_{HOST,PORT}` | HTTP at `ms-loyalty:8002` |
| **Azure Storage** | `AZURE_STORAGE_CONNECTION_STRING` | Blob storage connection |
| **Retailer-specific** | `HUDSON_SERVER_HOST`, `EROSKI_SDK_LOYALTY_NUMBER_DECRYPTION_KEY/IV`, `SEVENTH_HEAVEN_ENCRYPTION_KEY` | Retailer API credentials |

### Backend Base Variables (`conf/backend-variables.env`)

```
DEFAULT_DATABASE_HOST=backend_db_v2
INVENTORY_DATABASE_HOST=backend_db_v2
GRPC_DNS_RESOLVER=native
```

The `GRPC_DNS_RESOLVER=native` setting forces gRPC to use the system DNS resolver instead of the c-ares resolver, ensuring Docker DNS resolution works correctly for service names.

### Microservice Environment Summary

| Service | Key Env Vars | Database Host | Notes |
|---------|-------------|---------------|-------|
| **ms-promo** | `MYSQL_DB_HOST=ms_promo_db`, `MYSQL_DATABASE=ms_promo` | `ms_promo_db` | Shared MySQL with ms-loyalty |
| **ms-loyalty** | `MYSQL_DB_HOST=ms_promo_db`, `MYSQL_DATABASE=ms_loyalty` | `ms_promo_db` | Uses promo service's MySQL instance |
| **ms-tax (gRPC)** | `MYSQL_HOST=ms_tax_db` | `ms_tax_db` | Legacy Go gRPC service |
| **ms-tax (HTTP)** | `MYSQL_HOST=tax-db` | `tax-db` | New FastAPI service |
| **pay** | `MONGODB_HOST=mongo-db`, `MONGODB_DATABASE=mishi` | `mongo-db` | Also has Redis, Stripe, TokenEx config |
| **pay-gateway** | `PAYMENT_SERVICE_HOST=http://pay:80` | — | Node.js proxy to pay service |
| **voucher** | `MYSQL_HOST=voucher-db` | `voucher-db` | SQLAlchemy connect URL also provided |

---

## Monitoring — Datadog

Datadog Agent (`gcr.io/datadoghq/agent:7`) is deployed in all environments with these capabilities:

| Feature | Setting | Value |
|---------|---------|-------|
| **Logs** | `DD_LOGS_ENABLED` | `true` |
| **APM** | `DD_APM_ENABLED` | `true` |
| **Tracing** | `DD_TRACE_ENABLED` | `true` |
| **Process monitoring** | `DD_PROCESS_AGENT_ENABLED` | `true` |
| **Container logs** | `DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL` | `true` |
| **StatsD** | `DD_DOGSTATSD_NON_LOCAL_TRAFFIC` | `true` |
| **Site** | `DD_SITE` | `datadoghq.eu` (EU region) |

**Per-service Datadog labels** on containers:
- `com.datadoghq.ad.logs` — Log source type (`python`, `nodejs`, `jboss_wildfly`, `kong`)
- `com.datadoghq.tags.service` — Service name
- `com.datadoghq.tags.version` — Version tag
- `com.datadoghq.tags.env` — Environment (`local`, `test`, `prod`)

**Excluded from log collection (test/dev):** `dd-agent`, `backend_db_v2`, `mpay_rabbitmq`, `mpay_redis`, `ms_promo_db`, `ms_tax_db`, `mongo-db`, `mongo_inventory`, `elastic_inventory`

### Metrics Pipeline

```
Gunicorn (backend_v2)  ──statsd──►  ddagent:8125  ──►  Datadog EU
Django (ddtrace-run)   ──APM────►  ddagent:8126  ──►  Datadog EU
```

---

## Volume Mounts

### Backend Application Volumes

The monolith container mounts several host directories for data persistence and file exchange:

| Container Path | Host Path | Mode | Purpose |
|---------------|-----------|------|---------|
| `/app/conf/` | `./conf/` | `ro` | Configuration files |
| `/app/service/` | `./mainServer/` | `rw` | Source code (dev hot-reload) |
| `/app/static/` | `./automation/static/` | `rw` | Django static files |
| `/app/poslog/` | `../poslog/` | `rw` | POS transaction logs |
| `/app/marketing_consent/` | `../marketing_consent/` | `rw` | Marketing consent exports |
| `/app/transaction_logs/` | `../transaction_logs/` | `rw` | Transaction log files |
| `/app/product_data/` | `../product_data/` | `rw` | Product CSV exports (e.g., Relay) |
| `/app/promotions/` | `../promotions/` | `ro` | Promotion import files |
| `/app/reconciliation/` | `../reconciliation/` | `rw` | Settlement CSV files (e.g., Flying Tiger) |
| `/app/additional_reports/` | `../additional_reports/` | `rw` | Additional CSV reports (e.g., Desigual) |

**Note:** The `pay` service shares the `../reconciliation` volume with `backend_v2` for settlement report exchange.

---

## Development vs Production vs Test vs CI

### Development (`docker-compose.yml`)

- **22 services** — full local stack including all databases
- Backend image uses `CI_IMAGE_VERSION` env var (defaults to `local`)
- Source code mounted as `rw` for hot-reload
- Django `runserver` used in promotions service (not Gunicorn)
- All databases run locally with init scripts
- Kafka consumers connect to Confluent Cloud (managed Kafka)
- PostgreSQL shared memory: `shm_size: 1g`
- macOS optimization: `:delegated` on database volume mounts

### Production (`docker-compose-prod.yml`)

- **7 services only** — no databases (managed externally), no Kafka consumers (separate deployment)
- Fixed image version: `ghcr.io/mishipay-ltd/python-django-monolith-service:v6.21.17`
- Backend on both `mpay_pub` and `mpay_net` with `backend-v2` alias
- **Memcached** added (`memcached:1.6.6-alpine`, 512MB) — not in dev
- **Datadog Agent** added with production env config
- Keycloak exposes additional JGroups clustering ports (7600, 57600, 45700, 55200, 54200, 23364, 45688)
- Networks have explicit IPAM subnets: `mpay_pub=172.34.0.0/16`, `mpay_net=172.28.0.0/16`
- Extra hosts for Eroski API: `gersoa2prepro.eroski.es`, `gersoa2.eroski.es` → `172.30.227.254`
- Datadog labels on all services for log routing and tagging

### Test (`docker-compose-test.yml`)

- **Overlay only** — extends base `docker-compose.yml`
- Swaps env files to `*.test.env` variants for 8 services
- Sets `DD_ENV=test` on backend
- Same structure as dev but with test-specific credentials and endpoints

### CI (`docker-compose-ci.yml`)

- **Minimal overlay** — only overrides `backend_v2` image
- Uses `ghcr.io/mishipay-ltd/python-django-monolith-service-ci:{version}` (separate CI image with test dependencies)

### Promotions Service Environments

| Environment | Image Source | App Server | Datadog |
|-------------|------------|------------|---------|
| Dev (`docker-compose.yml`) | `ghcr.io/mishipay-ltd/ms-promo-ci` (GitHub CR) | `runserver` | Agent included |
| Azure CI (`docker-compose-azure-ci.yml`) | `acrhubprodwe01.azurecr.io/ms-promo` (Azure CR) | `runserver` | Agent included |
| Production | `ghcr.io/mishipay-ltd/ms-promo:v0.2.35` (from monolith compose) | Gunicorn (`start.sh`) | Via monolith Datadog agent |

---

## Service Communication Patterns

### Internal Service Routing

```
                     ┌──────────────────────────────────────┐
                     │        External Clients               │
                     │  (iOS, Android, Web, Dashboard)       │
                     └─────────────┬────────────────────────┘
                                   │
                          ┌────────▼────────┐
                          │   Kong Gateway   │
                          │  :80 / :443      │
                          │  JWT-Keycloak    │
                          └────────┬────────┘
                                   │
                ┌──────────────────┼──────────────────────┐
                │                  │                       │
                ▼                  ▼                       ▼
         ┌─────────────┐  ┌──────────────┐      ┌──────────────┐
         │ backend_v2  │  │  Keycloak    │      │ pay-gateway  │
         │   :8080     │  │   :8005      │      │   :3000      │
         └──────┬──────┘  └──────────────┘      └──────┬───────┘
                │                                       │
    ┌───────────┼───────────┬───────────┐              │
    │           │           │           │              ▼
    ▼           ▼           ▼           ▼        ┌──────────┐
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐    │   pay    │
│ms-promo│ │ms-tax  │ │ms-tax  │ │voucher │    │  :8003   │
│ :8001  │ │(gRPC)  │ │(HTTP)  │ │  :80   │    └──────────┘
│  HTTP  │ │ :9000  │ │  :80   │ │  HTTP  │
└────────┘ └────────┘ └────────┘ └────────┘

    ┌───────────────────────────────────────────┐
    │           Kafka Consumers                  │
    │  consumer_post_payment_success             │
    │  consumer_common                           │
    │  consumer_payment_only_clients_analytics   │
    │  (same image as backend_v2, different CMD) │
    └───────────────────────────────────────────┘
```

### Protocol Summary

| From → To | Protocol | Port | Notes |
|-----------|----------|------|-------|
| Kong → backend_v2 | HTTP | 8080 | `preserve_host: true` |
| Kong → Keycloak | HTTP | 8005 | Auth realm access |
| Kong → pay-gateway | HTTP | 3000 | Payment gateway |
| backend_v2 → ms-promo | HTTP | 8001 | Promotion evaluation |
| backend_v2 → ms-loyalty | HTTP | 8002 | Loyalty/referral campaigns |
| backend_v2 → ms-tax (legacy) | gRPC | 9000 | Tax calculation |
| backend_v2 → ms_tax (new) | HTTP | 80 | New tax service |
| backend_v2 → pay | HTTP | 80 | Payment processing (via MS Pay microservice client) |
| backend_v2 → inventory | HTTP | 8000 | Item inventory |
| backend_v2 → voucher | HTTP | 80 | Voucher redemption |
| pay-gateway → pay | HTTP | 80 | Gateway-to-service proxy |
| pay → mongo-db | MongoDB | 27017 | Payment data storage |
| inventory → mongo-inventory | MongoDB | 27017 | Inventory data |
| inventory → elastic-inventory | HTTP | 9200 | Search index |
| Kafka consumers → Confluent Cloud | SASL/SSL | 9092 | Event streaming |
| backend_v2 → mpay_redis | Redis | 6379 | Caching |
| backend_v2 → mpay_rabbitmq | AMQP | 5672 | Dramatiq task queue |

---

## Kafka Consumer Services

Three Kafka consumer services run as separate containers using the same Docker image as `backend_v2` but with different management commands:

| Container | Command | Health Check | Consumer Group | Topics |
|-----------|---------|-------------|----------------|--------|
| `consumer_post_payment_success` | `python manage.py consumer_post_payment_success` | `consumers_health_check` | `PAYMENT_GROUP` | `mishi.streams.post-payment-success` |
| `consumer_common` | `python manage.py consumer_common` | `consumers_health_check` | `COMMON_GROUP` | `mishi.streams.common`, `mishi.streams.post-transaction-success` |
| `consumer_payment_only_clients_analytics` | `python manage.py consumer_payment_only_clients_analytics` | `consumers_health_check` | `PAYMENT_GROUP` | `queuing.external.payments.analytics.json` |

All consumers:
- Connect to **Confluent Cloud** (managed Kafka, West Europe Azure region) via `mpay_pub` network
- Use SASL/SSL authentication with API key/secret
- Have 20s health check intervals with 15s timeout and 3 retries
- Health check runs `python manage.py consumers_health_check` — a simple `SELECT 1` database connectivity check (`mishipay_core/management/commands/consumers_health_check.py`)
- Auto-commit enabled (1000ms interval), offset reset: `earliest`

**Consumer group concern:** `PostPaymentSuccessConsumer` and `PaymentOnlyClientsAnalyticsConsumer` share `group.id: "PAYMENT_GROUP"` but subscribe to different topics. This is valid Kafka behaviour but can cause unexpected rebalancing. They also use **different credentials** — the analytics consumer uses separate `CONF_KAFKA_PAYMENTS_API_KEY/SECRET`, suggesting isolated topic access or a different Kafka cluster.

### Kafka Topics

8 topics defined in `mainServer/pubsub/topics.py`:

| Topic | Variable | Direction | Consumer | Producer Adapter |
|-------|----------|-----------|----------|------------------|
| `mishi.streams.post-payment-success` | `POST_PAYMENT_SUCCESS_TOPIC` | Produce + Consume | `PostPaymentSuccessConsumer` | `PostPaymentSuccessProducerAdapter` |
| `mishi.streams.common` | `COMMON_TOPIC` | Produce + Consume | `CommonConsumer` | `PostRefundSuccessProducerAdapter`, `PostDashboardVerficationProducerAdapter` |
| `mishi.streams.post-transaction-success` | `POST_TRANSACTION_SUCCESS_TOPIC` | Consume only | `CommonConsumer` | — (produced by external service) |
| `mishi.streams.post-transaction` | `POST_TRANSACTION_TOPIC` | Produce only | — | `PostTransactionProducerAdapter` |
| `flying_tiger.order.post_transaction` | `FTC_POST_TRANSACTION` | Produce only | — | `TriggerPoslogAdapter` |
| `flying_tiger.cash_desk.staff_shifts.end_shift_event` | `CASH_DESK_END_SHIFT_TOPIC` | Produce only | — | `CashDeskEndShiftEventProducerAdapter` |
| `queuing.external.payments.analytics.json` | `EXTERNAL_PAYMENTS_ANALYTICS_TOPIC` | Consume only | `PaymentOnlyClientsAnalyticsConsumer` | — (produced by external pay service) |
| `test1` | `TEST1` | — | — | — (test/dev only) |

### Consumer → Handler Dispatch

Each consumer delegates message processing to handler classes located in other Django apps:

```
┌─────────────────────────────────────────────────────────────────────┐
│  PostPaymentSuccessConsumer                                         │
│  Topic: mishi.streams.post-payment-success                         │
│  Message key: order_id  │  Payload: { payment_id }                 │
│  Handler: PostPaymentSuccessHandler                                 │
│    → Retrieves Payment object                                       │
│    → Calls PaymentService.payment_success_async_tasks()             │
│    → (fiscalisation, order fulfilment, receipt generation)          │
│  Retries: 3  │  On failure: Slack #alerts_kafka                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  CommonConsumer (multiplexed router)                                │
│  Topics: mishi.streams.common + mishi.streams.post-transaction-    │
│          success                                                    │
│  Routes on consumer_identifier_key in payload:                     │
│                                                                     │
│  POST_REFUND_SUCCESS → PostRefundSuccessHandler                    │
│    → Store-type refund fulfilment + receipt email                   │
│                                                                     │
│  POST_DASHBOARD_VERIFICATION_SUCCESS →                             │
│    PostDashboardVerificationSuccessHandler                          │
│    → Store-type order fulfilment after staff verification           │
│                                                                     │
│  POST_TRANSACTION_SUCCESS → PostTransactionSuccessHandler          │
│    → Updates Fiscalisation record with mark ID + receipt email      │
│  Retries: 5  │  On failure: Slack #alerts_kafka                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  PaymentOnlyClientsAnalyticsConsumer                                │
│  Topic: queuing.external.payments.analytics.json                   │
│  Payload: { store_id, total_basket_value, total_txn_count,         │
│             total_basket_value_tax_excluded, date, business_date,   │
│             platform }                                              │
│  Handler: ExternalPaymentsAnalyticsHandler                         │
│    → Upsert AggregatedAnalytics (amounts in cents → decimal)       │
│  Retries: 5  │  On failure: Slack #alerts_kafka                    │
└─────────────────────────────────────────────────────────────────────┘
```

Handler file locations:
- `mishipay_retail_payments/pubsub/handlers/mark_payment_success_handler.py` — `PostPaymentSuccessHandler`
- `mishipay_retail_payments/pubsub/handlers/mark_refund_success_handler.py` — `PostRefundSuccessHandler`
- `mishipay_retail_payments/pubsub/handlers/mark_dashboard_verification_success_handler.py` — `PostDashboardVerificationSuccessHandler`
- `mishipay_retail_payments/pubsub/handlers/external_payments_analytics_handler.py` — `ExternalPaymentsAnalyticsHandler`
- `fiscalisation/pubsub/handlers/post_transaction_success_handler.py` — `PostTransactionSuccessHandler`

### Producer Adapters

6 producer adapters in `mainServer/pubsub/adapters/`:

| Adapter | Topic | Message Key | Payload Fields | Called From |
|---------|-------|-------------|----------------|-------------|
| `PostPaymentSuccessProducerAdapter` | `mishi.streams.post-payment-success` | `order_id` | `payment_id` | Payment completion flow |
| `PostRefundSuccessProducerAdapter` | `mishi.streams.common` | `order_id` | `order_id`, `refund_id`, `consumer_identifier_key` | Dashboard refund flow |
| `PostDashboardVerficationProducerAdapter` | `mishi.streams.common` | `order_id` | `order_id`, `consumer_identifier_key` | Dashboard verification |
| `PostTransactionProducerAdapter` | `mishi.streams.post-transaction` | `order_id` | `external_id`, `order_id`, `store_type`, `payment_status` | Flying Tiger transactions |
| `TriggerPoslogAdapter` | `flying_tiger.order.post_transaction` | — (none) | Full poslog payload | Flying Tiger POSLOG |
| `CashDeskEndShiftEventProducerAdapter` | `flying_tiger.cash_desk.staff_shifts.end_shift_event` | — (none) | Full shift data | Cashier kiosk shift end |

### Kafka Message Format

All messages follow a standard envelope (`mainServer/pubsub/base/message.py`):

```json
{
  "data": {
    "payment_id": "...",
    "order_id": "..."
  },
  "server_timestamp": 1706745600000
}
```

Encoding: JSON → UTF-8 bytes. Decoding reconstructs payload via `BasePayload.to_instance(**data["data"])`.

### Kafka Credentials

Two credential sets (from `mainServer/mainServer/settings.py:132-137`):

| Setting | Environment Variable | Used By |
|---------|---------------------|---------|
| `CONF_KAFKA_BOOTSTRAP_SERVER` | `CONF_KAFKA_BOOTSTRAP_SERVER` | All consumers/producers |
| `CONF_KAFKA_API_KEY` | `CONF_KAFKA_API_KEY` | Main consumers + all producers |
| `CONF_KAFKA_API_SECRET` | `CONF_KAFKA_API_SECRET` | Main consumers + all producers |
| `CONF_KAFKA_PAYMENTS_API_KEY` | `CONF_KAFKA_PAYMENTS_API_KEY` | `PaymentOnlyClientsAnalyticsConsumer` only |
| `CONF_KAFKA_PAYMENTS_API_SECRET` | `CONF_KAFKA_PAYMENTS_API_SECRET` | `PaymentOnlyClientsAnalyticsConsumer` only |

Default bootstrap server: `pkc-1wvvj.westeurope.azure.confluent.cloud:9092` (Confluent Cloud, West Europe Azure).

### Error Handling & Retry

| Consumer | Max Retries | On Each Retry | On Exhaustion |
|----------|-------------|---------------|---------------|
| `PostPaymentSuccessConsumer` | 3 | Close DB connection, re-init handler | Log error, Slack `#alerts_kafka`, drop message |
| `CommonConsumer` | 5 | Close DB connection, re-init handler | Log error, Slack `#alerts_kafka`, drop message |
| `PaymentOnlyClientsAnalyticsConsumer` | 5 | Close DB connection, re-init handler | Log error, Slack `#alerts_kafka`, drop message |

No dead-letter queue exists. Failed messages are dropped after Slack notification.

---

## Microservices Architecture

The monolith communicates with **7 external microservices** via HTTP and gRPC. Each microservice is an independently deployed container with its own database.

### Microservice Catalog

```
┌─────────────────────────────────────────────────────────────────────┐
│                        backend_v2 (Monolith)                        │
│                         Django / PostgreSQL                          │
└──────┬──────┬──────┬──────┬──────┬──────┬──────┬───────────────────┘
       │      │      │      │      │      │      │
       ▼      ▼      ▼      ▼      ▼      ▼      ▼
   ┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐┌─────────┐
   │ms-   ││ms-   ││ms-tax││ms_tax││pay + ││vouch-││inventory│
   │promo ││loyal-││(gRPC)││(HTTP)││pay-  ││er    ││         │
   │      ││ty    ││      ││      ││gate- ││      ││         │
   │:8001 ││:8002 ││:9000 ││:80   ││way   ││:80   ││:8000    │
   │HTTP  ││HTTP  ││gRPC  ││HTTP  ││:3000 ││HTTP  ││HTTP     │
   │      ││      ││      ││      ││+:80  ││      ││         │
   │MySQL ││MySQL ││MySQL ││MySQL ││Mongo ││MySQL ││Mongo+ES │
   └──────┘└──────┘└──────┘└──────┘└──────┘└──────┘└─────────┘
```

### ms-promo — Promotions Service

| Property | Value |
|----------|-------|
| **Image** | `ghcr.io/mishipay-ltd/ms-promo:v0.2.35` |
| **Port** | 8001 |
| **Protocol** | HTTP (REST) |
| **Database** | Percona MySQL 8.0 (`ms_promo_db`, database: `ms_promo`) |
| **Framework** | Django 3.2.24 |
| **Source repo** | `service.promotions/` (documented separately in Phase 1) |

**Database initialization** (`automation/ms.promo/mysqld_init/000.sql`):
- Creates `test_ms_promo` database (UTF-8)
- Grants privileges to `mpay_ms_promo` user

**Monolith connection config:**
```
MS_PROMO_PROTOCOL=http
MS_PROMO_HOST=ms-promo
MS_PROMO_PORT=8001
```

**Primary usage:** Promotion evaluation (basket discount calculation), promotion CRUD, bulk batch operations. The monolith calls `POST /api/1.0/promotions/evaluate/` during item scanning and basket operations.

See [Promotions Service Documentation](./promotions-service/overview.md) for full details.

### ms-loyalty — Loyalty & Referral Service

| Property | Value |
|----------|-------|
| **Image** | `ghcr.io/mishipay-ltd/python-django-loyalty-service:v0.0.11` |
| **Port** | 8002 |
| **Protocol** | HTTP (REST) |
| **Database** | Shared Percona MySQL (`ms_promo_db`, database: `ms_loyalty`) |
| **Framework** | Django (version unspecified) |

**Database initialization** (`automation/ms.promo/mysqld_init/001.sql`):
- Creates user `mpay_ms_loyalty` with password
- Creates databases `ms_loyalty` and `test_ms_loyalty` (UTF-8)
- Grants full privileges

**Note:** The loyalty service's database init script lives in the `ms.promo/` automation directory — `automation/ms.loyalty/` does not exist. Both services share the same MySQL instance (`ms_promo_db`) but use separate databases.

**Monolith connection config:**
```
MS_LOYALTY_HOST=ms-loyalty
MS_LOYALTY_PORT=8002
```

**Primary usage:** Referral campaign management (CRUD, code generation, validation, redemption). The monolith's `mishipay_referrals` module acts as a stateless proxy to this service.

### ms-tax (Legacy) — Tax Service (gRPC)

| Property | Value |
|----------|-------|
| **Image** | `ghcr.io/mishipay-ltd/service.tax/ms_tax:1.0` |
| **Port** | 9000 |
| **Protocol** | gRPC |
| **Database** | Percona MySQL 8.0 (`ms_tax_db`, database: `test_ms_tax`) |
| **Language** | Go |
| **Network** | `mpay_net` only (no internet access) |

**Database initialization** (`automation/ms.tax/mysqld_init/000.sql`):
- Creates `test_ms_tax` database (UTF-8)
- Grants privileges to `mpay_ms_tax` user

**Schema migration** (`automation/ms.tax/migrations/1_taxlevel.up.sql`):
```sql
CREATE TABLE IF NOT EXISTS tax_level(
    id BINARY(16) PRIMARY KEY DEFAULT (UUID_TO_BIN(UUID())),
    store_id varchar(64),
    tax_code varchar(25),
    tax_level varchar(25),
    retailer_tax_level_id varchar(25),
    tax_percent NUMERIC(13, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY store_id__tax_code__retailer_tax_level_id
        (store_id, tax_code, retailer_tax_level_id)
);
```

**gRPC service definition** (`mainServer/protos/tax.proto`):

4 RPC methods:

| RPC | Request → Response | Purpose |
|-----|-------------------|---------|
| `EvaluateTax` | `TaxEvaluateRequest` → `TaxEvaluateResponse` | Calculate tax for basket items |
| `GetStoreTaxTable` | `GetStoreTaxTableRequest` → `StoreTaxTable` | Retrieve store's tax configuration |
| `ImportStoreTaxCodeData` | `ImportStoreTaxDataRequest` → `google.protobuf.Empty` | Bulk import tax codes |
| `DeleteStoreTaxTable` | `DeleteStoreTaxTableRequest` → `google.protobuf.Empty` | Remove store's tax data |

Key protobuf messages:
- `TaxEvaluateRequest`: `store_data` (store_id, tax_included_in_price, tax_calculator), `commit` flag, `items[]` (item_id, total_amount, tax_code)
- `TaxEvaluateResponse`: `total_basket_amount`, `total_taxable_amount`, `total_tax`, `total_basket_amount_after_tax`, `tax_breakup[]`, `item_tax[]`

**Monolith connection config:**
```
MS_TAX_HOST=ms-tax
MS_TAX_PORT=9000
GRPC_DNS_RESOLVER=native  # Forces system DNS for Docker resolution
```

**Generated stubs:** `mainServer/protos/tax_pb2.py` and `mainServer/protos/tax_pb2_grpc.py` (also duplicated at `mainServer/tax_pb2.py` and `mainServer/tax_pb2_grpc.py`).

### ms_tax (New) — Tax Service (HTTP)

| Property | Value |
|----------|-------|
| **Image** | `ghcr.io/mishipay-ltd/mishipay-tax-service:v0.4.0` |
| **Port** | 80 (mapped from host 8008) |
| **Protocol** | HTTP (REST) |
| **Database** | MySQL 8.0.25 (`tax-db`, database: `ms_tax`) |
| **Framework** | FastAPI (Python) |
| **Migrations** | Alembic |

**Monolith connection config:**
```
MS_TAX_NEW_PROTOCOL=http
MS_TAX_NEW_HOST=ms_tax
MS_TAX_NEW_PORT=80
```

**The monolith has a tax client selector** that routes to either the legacy gRPC or new HTTP service based on store configuration. This enables a gradual per-store migration.

### pay + pay-gateway — Payment Services

| Property | pay | pay-gateway |
|----------|-----|-------------|
| **Image** | `ghcr.io/mishipay-ltd/mishipay-payment-service:v1.12.9` | `ghcr.io/mishipay-ltd/mishi-platform/ms_pay_gateway:1.59` |
| **Port** | 8003 (→80 internal) | 3000 |
| **Protocol** | HTTP | HTTP |
| **Database** | MongoDB 4.4 (`mongo-db`, database: `mishi`) | — (proxies to `pay`) |
| **Language** | Python | Node.js |

**Monolith connection config:**
```
MS_PAY_HOST=pay
MS_PAY_PORT=80
MS_PAY_PROTOCOL=http
```

**`pay-gateway`** is a Node.js reverse proxy that sits between external clients (mobile apps, web) and the `pay` service. Client-facing payment operations go through `pay-gateway:3000`, while the monolith calls `pay:80` directly.

**`pay`** handles payment session lifecycle, PSP integration (35+ gateways), card saving, refunds, and publishes analytics events to the `queuing.external.payments.analytics.json` Kafka topic.

### voucher — Voucher/Gift Card Service

| Property | Value |
|----------|-------|
| **Image** | `ghcr.io/mishipay-ltd/mishi-platform/ms_voucher:1.2` |
| **Port** | 80 (mapped from host 8006) |
| **Database** | MySQL 8.0.25 (`voucher-db`, database: `mishi_voucher`) |

### inventory — Inventory Service

| Property | Value |
|----------|-------|
| **Image** | `ghcr.io/mishipay-ltd/inventory-service:v1.2.1` |
| **Port** | 8000 (internal only) |
| **Database** | MongoDB 4.4 (`mongo-inventory`, replica set `rs0`) + Elasticsearch 7.11.2 |
| **Network** | `mpay_net` only |

**Monolith connection config:**
```
INVENTORY_SERVICE_PROTOCOL=http
INVENTORY_SERVICE_HOST=inventory
INVENTORY_SERVICE_PORT=8000
INVENTORY_SERVICE_OPS_PROTOCOL=http
INVENTORY_SERVICE_OPS_HOST=inventory
INVENTORY_SERVICE_OPS_PORT=8000
```

Two endpoint groups: regular (item lookup, category, search) and "ops" (bulk import/export, batch operations). Both currently point to the same host.

### Internal Service Client (`microservices_client/`)

In addition to the external microservice calls above, the monolith has an **in-process service client** (`mainServer/microservices_client/`) that uses Django's test `Client()` to make HTTP-style requests to internal URL endpoints via `django.urls.reverse()`. This is **not** network-based communication — it routes within the same Django process.

7 client classes provide 30+ methods covering items, orders, payments, customers, analytics, and third-party auth (Decathlon). All requests include `mpay_lng` language parameter. These "_as_service" endpoints are used by the legacy DOS module to delegate to newer V1/V2 view implementations.

See [Platform Integrations — Microservices Client](./platform/integrations.md#microservices-client-microservices_client) for full method documentation.

### Service Initialization Flow

The `automation/setup_env_services.sh` script initializes all services in order:

1. **backend_v2** — Django `migrate` (PostgreSQL)
2. **ms-promo** — Django `migrate` (MySQL via `ms_promo_db`)
3. **ms-loyalty** — Django `migrate` (MySQL via `ms_promo_db`, custom manage.py path)
4. **ms_tax** — Alembic `upgrade head` (MySQL via `tax-db`)
5. **MongoDB** — User creation for authentication

Retry logic waits up to 25 seconds for MySQL availability before running migrations.

---

## Notable Patterns

1. **Shared database instances:** `ms_promo_db` serves both the promotions and loyalty services. `backend_db_v2` serves both the monolith and Keycloak. This reduces infrastructure cost but creates coupled failure domains.

2. **Dual tax services:** Two tax services coexist — `ms-tax` (Go/gRPC on port 9000) and `ms_tax` (FastAPI/HTTP on port 80). The monolith references both via separate env vars (`MS_TAX_HOST`/`MS_TAX_PORT` vs `MS_TAX_NEW_*`), suggesting an ongoing migration from gRPC to HTTP. A tax client selector in the monolith routes per-store to enable gradual migration.

3. **Consumer-as-container:** Kafka consumers reuse the full monolith Docker image with different `command` directives. This means each consumer loads the entire Django application despite only needing Kafka handling code — a tradeoff of simplicity vs resource efficiency.

4. **Production minimalism:** The production compose file includes only 7 services. Databases, Kafka consumers, and most microservices are presumably deployed separately (likely in Kubernetes, given Helm references in CI/CD pipelines documented in T01).

5. **File-based data exchange:** The monolith uses mounted host volumes for file-based data exchange (POSLOGs, reconciliation CSVs, marketing consent files, transaction logs). The `pay` service shares the `/reconciliation` volume with `backend_v2`.

6. **Two Docker registries:** Images are published to both GitHub Container Registry (`ghcr.io/mishipay-ltd/`) and Azure Container Registry (`acrhubprodwe01.azurecr.io/`). The monolith and most services use GHCR; the promotions service Azure CI uses ACR.

7. **Topic-level routing vs payload routing:** Some events have dedicated topics (`mishi.streams.post-payment-success`), while others share the `mishi.streams.common` topic and use a `consumer_identifier_key` field in the payload for dispatch. This creates a hybrid routing model.

8. **No dead-letter queue:** Failed Kafka messages are dropped after retry exhaustion with only a Slack notification. No persistent failure record or DLQ topic exists for later reprocessing.

9. **Synchronous producers:** All producers call `flush()` after each `produce()`, making them effectively synchronous despite the async naming of `async_produce()`. This ensures delivery confirmation but adds latency.

10. **Mutable default timestamp bug:** `BaseProducerAdapter.produce()` has `timestamp=datetime.utcnow()` as a default argument — evaluated once at import time. Callers that don't pass an explicit timestamp would silently share the module import timestamp.

11. **Missing loyalty automation directory:** `automation/ms.loyalty/` does not exist. The loyalty service's database initialization lives in `automation/ms.promo/mysqld_init/001.sql`, tightly coupling the two services' deployment.

12. **gRPC generated stubs duplicated:** The tax gRPC stubs (`tax_pb2.py`, `tax_pb2_grpc.py`) exist in both `mainServer/protos/` and `mainServer/` (root). The Makefile `grpc-compile` target generates them into `protos/`.

---

## CI/CD Pipelines Summary

The two repositories use **different CI/CD strategies**:

### MishiPay Monolith — Azure VMSS Deployment

The monolith has **5 Azure Pipeline files** — all focused on VMSS deployment (not build/test). No build pipeline exists in the repo; the Docker image is built externally.

| Pipeline | Target | Trigger | Strategy |
|----------|--------|---------|----------|
| `azure-pipelines-vmss-test.yml` | Test backend VMSS | Push to `test` | Simple rolling update |
| `azure-pipelines-vmss-prod.yml` | Prod backend VMSS | Manual | Simple rolling update |
| `azure-pipelines-vmss-prod-new.yml` | Prod backend VMSS | Manual | **Graceful: App Gateway drain → deploy → restore** |
| `azure-pipelines-vmss-consumers-prod.yml` | Prod consumer VMSS | Manual | Simple rolling update |
| `azure-pipelines-vmss-consumers-prod-new.yml` | Prod consumer VMSS | Manual | Simple rolling update |

**Branch strategy:** `master` → production (manual), `test` → staging (auto).

**Development toolchain:** The `Makefile` provides 10 targets for testing (`pytest`, Django test runner), linting (`flake8`), migrations, i18n (12 locales), and gRPC stub generation.

**Production cron jobs:** 20+ active jobs on the backend server (thank-you emails, payment verification, inventory uploads, POSLOG generation) and 15+ on a dedicated SFTP server (retailer data sync via `rclone` to Azure Blob Storage).

### Promotions Service — Kubernetes Deployment

Dual CI/CD pipelines (Azure Pipelines + GitHub Actions) with automated build → test → version → release → Helm chart bump. Deploys to Kubernetes.

**Full details:** See [Promotions Service DevOps](./promotions-service/devops.md).

### Detailed CI/CD Documentation

See [Platform Infrastructure](./platform/infrastructure.md) for full monolith CI/CD documentation including:
- Azure Pipeline deployment strategies (simple rolling vs graceful traffic drain)
- Makefile development toolchain (10 targets)
- Production cron job schedules (backend + SFTP servers)
- Operational scripts (database restore, legacy deploy, retailer shell scripts)

---

## Open Questions

- Are all env files committed with actual credentials (API keys, database passwords, Stripe tokens, etc.), or are these development-only values? Several files contain what appear to be real keys (Sentry DSNs, Kafka credentials, Azure Storage connection strings, Stripe test keys, TokenEx keys).
- How are production env files managed? The `conf/` directory contains `*.prod.env` and `*.prd.env` files — are these actual production values or templates?
- The production compose only has 7 services. How are the remaining services (pay, pay-gateway, voucher, inventory, ms-promo, ms-loyalty, databases, Kafka consumers) deployed in production? Kubernetes (Helm charts referenced in T01 CI/CD), VMs, or separate compose files?
- `ms-tax` and `ms_tax` (two different compose service names) reference two different tax service implementations. Is the gRPC service (`ms-tax`) still active or fully deprecated?
- The Gunicorn `loglevel` is set to `debug` with a TODO comment to change to `info`. Is this the actual production value?
- Nginx rate limiting (5 req/s per IP, burst 20) — is this applied in production, or does Cloudflare/Kong handle rate limiting before traffic reaches Nginx?
- The RabbitMQ credentials (`mpay_broker` / `Mpt43Gwsfdsa`) are hardcoded in `docker-compose.yml`. Are these production values?
- Are the two sets of Kafka credentials (`CONF_KAFKA_API_KEY/SECRET` vs `CONF_KAFKA_PAYMENTS_API_KEY/SECRET`) for the same Confluent Cloud cluster with different ACLs, or for entirely separate clusters?
- Which stores have been migrated from the legacy gRPC tax service to the new HTTP tax service? Is there a timeline for completing the migration?
- The `mishi.streams.post-transaction` and `flying_tiger.order.post_transaction` topics are produce-only from the monolith. What external service(s) consume these topics?
- The `mishi.streams.post-transaction-success` topic is consume-only in the monolith. What external service produces to this topic? Is it the fiscalisation provider?
- Is the `StoreLocation` protobuf message (defined but not referenced by any RPC) reserved for future use or dead code?
- The `consumers_health_check` only tests database connectivity (`SELECT 1`). Should it also verify Kafka broker reachability?

---

## Dependencies

- **Upstream:** Relies on Cloudflare for DNS, SSL termination (in production), and DDoS protection. Confluent Cloud for managed Kafka.
- **Container Registries:** GitHub Container Registry (`ghcr.io/mishipay-ltd/`), Azure Container Registry (`acrhubprodwe01.azurecr.io/`)
- **Monitoring:** Datadog EU region (`datadoghq.eu`)
- **External APIs:** Sentry for error tracking, Azure Blob Storage for file uploads

See [Architecture Overview](./01-architecture-overview.md) for the full system diagram.
See [Tech Stack](./02-tech-stack.md) for version details of all technologies.
See [Platform Overview](./platform/overview.md) for Django settings and middleware chain.
See [Platform Infrastructure](./platform/infrastructure.md) for CI/CD pipelines, cron jobs, and deployment scripts.
See [Promotions Service DevOps](./promotions-service/devops.md) for promotions service CI/CD.
