# Platform Infrastructure — CI/CD, Deployment, Configuration & Operations

> Last updated by: Task T35 — Ansible Deployment & Configuration

## Overview

The MishiPay platform uses **two distinct deployment strategies** across its repositories:

1. **MishiPay Monolith** — Deployed to **Azure Virtual Machine Scale Sets (VMSS)** via Azure Pipelines. No build pipeline exists in the repo; Docker images are built externally and published to GitHub Container Registry. VMSS instances pull the latest image via CustomScript extensions.
2. **Promotions Service** — Deployed to **Kubernetes** via Helm charts. Dual CI/CD pipelines (Azure Pipelines + GitHub Actions) handle build, test, version, release, and Helm chart bumping. Documented in [Promotions Service DevOps](../promotions-service/devops.md).

This document covers the **monolith's** full infrastructure: CI/CD pipelines, Ansible deployment, HAProxy load balancing, configuration management (`conf/`), database initialization, monitoring, and the complete operational scripts inventory.

---

## Branch Strategy

```
master  ←── Production deployments (manual trigger via Azure Pipelines)
  │
test    ←── Test/staging deployments (auto-trigger on push)
  │
feature/* ←── Feature branches (PR-based development)
```

| Branch | Environment | Trigger | VMSS Target |
|--------|-------------|---------|-------------|
| `master` | Production | **Manual** (`trigger: none`) | Backend VMSS + Consumer VMSS |
| `test` | Test/Staging | **Automatic** (push to `test`) | Test VMSS |

There are **no GitHub Actions workflows** in the monolith repository. All CI/CD is handled by Azure Pipelines.

---

## Azure Pipelines — Deployment

The monolith has **5 Azure Pipeline files**, all focused on VMSS deployment (not build/test). The Docker image is built externally and referenced by version tag.

### Pipeline Inventory

| File | Target | Trigger | Azure Subscription | Job Name |
|------|--------|---------|-------------------|----------|
| `azure-pipelines-vmss-test.yml` | Test backend VMSS | Push to `test` | `AzureResourceManager (Pay-as-you-go) - Staging_UAT RG` | `UpdateTestVMSS` |
| `azure-pipelines-vmss-prod.yml` | Prod backend VMSS | Manual | `AzureResourceManager (Pay-as-you-go) - Production RG` | `UpdateProdVMSS` |
| `azure-pipelines-vmss-prod-new.yml` | Prod backend VMSS | Manual | `ARM-AzureSubscription1` | `UpdateProdVMSS` |
| `azure-pipelines-vmss-consumers-prod.yml` | Prod consumer VMSS | Manual | `AzureResourceManager (Pay-as-you-go) - Production RG` | `UpdateProdConsumerVMSS` |
| `azure-pipelines-vmss-consumers-prod-new.yml` | Prod consumer VMSS | Manual | `ARM-AzureSubscription1` | `UpdateProdConsumerVMSS` |

All pipelines run on `pool: vmImage: ubuntu-latest` and use `AzureCLI@2` tasks.

### Two Deployment Strategies

#### Strategy 1: Simple Rolling Update (3 pipeline files)

Used by: `azure-pipelines-vmss-test.yml`, `azure-pipelines-vmss-prod.yml`, `azure-pipelines-vmss-consumers-prod.yml`

**Process:**
1. **Update VMSS model** — `az vmss extension set --force-update --no-wait` triggers CustomScript extension re-execution on all instances
2. **Rolling instance update** — Queries instances where `latestModelApplied==false`, iterates each, runs `az vmss update-instances --instance-ids $instanceId`

```
Step 1: az vmss extension set --force-update (trigger CustomScript)
Step 2: For each outdated instance:
          └── az vmss update-instances --instance-ids $id
```

No traffic drain or health checks — instances are updated in-place sequentially.

#### Strategy 2: Graceful Rolling Deploy with Traffic Drain (1 pipeline file)

Used by: `azure-pipelines-vmss-prod-new.yml`

**Process:**
1. **Update VMSS model** — Same `az vmss extension set --force-update --no-wait`
2. **For each outdated instance:**
   a. **Get private IP** — Looks up the instance's NIC private IP
   b. **Stop traffic** — Removes the IP from the Application Gateway backend address pool
   c. **Drain** — `sleep 60` (60-second connection drain period)
   d. **Deploy** — `az vmss update-instances --instance-ids $instanceId`
   e. **Restore traffic** — Re-adds the IP to the Application Gateway backend pool

```
For each outdated VMSS instance:
  ├── 1. Get private IP from NIC
  ├── 2. Remove IP from Application Gateway address pool
  ├── 3. Sleep 60s (connection drain)
  ├── 4. az vmss update-instances (apply new model)
  └── 5. Add IP back to Application Gateway address pool
```

This ensures **zero-downtime deployment** by removing instances from the load balancer before updating them.

### Pipeline Variables (External)

All pipelines reference these Azure DevOps pipeline variables (not defined in YAML):

| Variable | Used By | Purpose |
|----------|---------|---------|
| `$(resource_group_name)` | All pipelines | Azure resource group name |
| `$(vmss_name)` | All pipelines | VMSS instance name |
| `$(app_gateway_name)` | Prod-new only | Application Gateway name |
| `$(address_pool_name)` | Prod-new only | Backend address pool name |

### Azure Subscription Naming

Two Azure subscription configurations exist:

| Name | Used By | Notes |
|------|---------|-------|
| `AzureResourceManager (Pay-as-you-go) - Production RG` | `vmss-prod.yml`, `vmss-consumers-prod.yml` | Older naming convention |
| `ARM-AzureSubscription1` | `vmss-prod-new.yml`, `vmss-consumers-prod-new.yml` | Newer naming convention |

Both target the same production environment. The "new" pipelines also use the graceful traffic drain strategy, suggesting they are the preferred deployment path.

### Deployment Architecture

```
                    Azure Pipelines
                         │
         ┌───────────────┼───────────────────┐
         │               │                   │
    Test VMSS       Prod Backend        Prod Consumer
    (auto on          VMSS                  VMSS
     test push)   (manual trigger)     (manual trigger)
         │               │                   │
         │       ┌───────┴────────┐          │
         │       │                │          │
         │    Simple          Graceful       │
         │    Rolling        w/ Traffic      │
         │    Update          Drain          │
         │                                   │
    ┌────┴───────────────────────────────────┴────┐
    │         Azure VMSS Instances                 │
    │    (Docker containers on each VM)            │
    │                                              │
    │    CustomScript Extension pulls new          │
    │    Docker image and restarts containers      │
    └──────────────────────────────────────────────┘
```

---

## Makefile — Development Toolchain

**Source:** `Makefile` (36 lines)

The Makefile provides shortcuts for common development tasks, all executed via `docker exec` against the running `backend_v2` container.

| Target | Command | Description |
|--------|---------|-------------|
| `test` | `docker exec backend_v2 pytest` | Run full test suite (reuse existing DB) |
| `test-clean` | `docker exec backend_v2 pytest --create-db` | Run tests with fresh database |
| `test-django` | `docker exec backend_v2 python3 manage.py test --keepdb --parallel` | Django's native test runner (parallel) |
| `t` | `docker exec backend_v2 pytest --create-db -n 1 "$(p).tests.$(t)"` | Run specific test: `make t p=myapp t=test_module` |
| `migrate` | `docker exec backend_v2 python3 manage.py migrate` | Apply database migrations |
| `migrations` | `docker exec backend_v2 python3 manage.py makemigrations` | Generate new migration files |
| `flake8` | Two `docker run` passes with `alpine/flake8:3.5.0` | Lint: (1) critical errors `E901,E999,F821,F822,F823`, (2) unused imports `F401,F405` excluding migrations and dos |
| `messages` | `docker exec backend_v2 python3 manage.py makemessages --locale=...` | Generate i18n messages for 12 locales: fr, es, eu, sv, da, it, fi, pt, de, he, no, nb |
| `compilemessages` | `docker exec backend_v2 python3 manage.py compilemessages` | Compile i18n `.po` → `.mo` files |
| `proto` | `docker exec backend_v2 python3 -m grpc_tools.protoc ...` | Generate gRPC Python stubs from `protos/tax.proto` |

**Notable details:**
- Flake8 uses an old image (`alpine/flake8:3.5.0`) — runs outside the main container via `docker run`
- Flake8 excludes `*/migrations/*` and `dos/*` from unused import checks
- The `messages` target generates translations for 12 locales (French, Spanish, Basque, Swedish, Danish, Italian, Finnish, Portuguese, German, Hebrew, Norwegian, Norwegian Bokmål)
- The `proto` target generates `tax_pb2.py` and `tax_pb2_grpc.py` from `protos/tax.proto`

---

## Operational Scripts

### Database Scripts

**Source:** `scripts/`

| Script | Purpose |
|--------|---------|
| `reset_db.sh` | Resets both main (`mpmain`) and test (`test_mishipay`) databases from the latest production dump, then applies any pending migrations |
| `restore_database.sh` | Fetches the latest database dump from `ubuntu@app.mishipay.com:/home/ubuntu/backups/`, drops and recreates the target database, restores from dump |

The database restore flow:
```
1. SSH to app.mishipay.com
2. Find latest dump file (ls -t | head -1)
3. SCP dump to local
4. DROP DATABASE (with confirmation prompt)
5. CREATE DATABASE
6. psql $db < $dump
7. Remove dump file
```

Default target database is `test_mishipay` (test); pass `mpmain` as argument for the main database.

### Legacy Deployment Scripts

**Source:** `scripts/shell_scripts/misc/`

| Script | Purpose | Technology |
|--------|---------|------------|
| `deploy.sh` | Sync codebase and restart — legacy pre-Docker deployment | `rsync` + `systemctl restart $ENV-uwsgi` to `10.0.3.6` |
| `deploy_poslog.sh` | Deploy Decathlon POS log Node.js service | `git pull` + `supervisorctl restart`, ports 7000 (prod) / 7002 (preprod) |
| `filebeat-install-ubuntu.sh` | Install Filebeat log shipper | System setup script |

**Note:** `deploy.sh` references `uwsgi` (pre-Docker era) and a static IP `10.0.3.6`, indicating this is a legacy script from before the Docker/VMSS migration. The current deployment uses Azure Pipelines with VMSS as documented above.

### Retailer Shell Scripts

**Source:** `scripts/shell_scripts/retailers/`

Per-retailer scripts invoked by production cron jobs. Organized by retailer:

| Retailer | Scripts | Operations |
|----------|---------|------------|
| **Flying Tiger** | 4 scripts | Inventory upload, POSLOG generation, BI file generation, staff discount |
| **Muji** | 6 scripts | Inventory upload, POSLOG generation (UK + JP), promotions upload, user-based coupons |
| **BWG** | 5 scripts + 3 SQL | Inventory upload, POSLOG generation, SPAR promotions import |
| **Dufry** | 2 scripts | Inventory upload, POSLOG upload |
| **Picwic** | 3 scripts + 1 SQL | Inventory upload, import |
| **Fast Eddy's** | 2 scripts | Inventory import, POSLOG generation |
| **Paradies** | 2 scripts | POSLOG generation, item import |
| **Bestseller** | 1 script | POSLOG upload |
| **Lagardere** | (referenced in crontabs) | Inventory upload, POSLOG upload |

These scripts typically:
1. Run `docker exec backend_v2 python3 manage.py <command>` to invoke Django management commands
2. Use SFTP/`lftp` to download/upload files from retailer servers
3. Use `rclone` to sync data to Azure Blob Storage

---

## Production Cron Jobs

### Backend Server (`scripts/crontabs/production_backend`)

**Host:** `mpay@prod-backend-1`

Active cron jobs (non-commented entries):

| Schedule | Retailer | Operation | Method |
|----------|----------|-----------|--------|
| `*/3 20 * * *` | MishiPay (internal) | Dashboard store access grant | `manage.py dashboard_all_stores_access` |
| `0 20 * * *` | MishiPay App | Thank You Email (yesterday) | `manage.py send_thank_you_email --app_type mishipay` |
| `5 20 * * *` | MishiPay App | Thank You Email (today) | `manage.py send_thank_you_email --app_type mishipay` |
| `10 20 * * *` | Decathlon App | Thank You Email (yesterday) | `manage.py send_thank_you_email --app_type decathlon` |
| `15 20 * * *` | Decathlon App | Thank You Email (today) | `manage.py send_thank_you_email --app_type decathlon` |
| `*/2 * * * *` | Decathlon | Verify payments | `manage.py verify_payments --store_type decathlonstoretype` |
| `0 4 * * *` | Lagardere/Relay | Daily item upload | Shell script |
| `20 21 * * *` | Lagardere/Relay | POSLOG generation | `manage.py relay_daily_poslog` |
| `30 21 * * *` | Lagardere/Relay | POSLOG upload | Shell script |
| `0 2 * * *` | Picwic | Daily item upload | Shell script |
| `0 5 * * *` | Dufry | Daily item upload | Shell script |
| `*/15 * * * *` | Dufry | POSLOG upload | Shell script (every 15 min) |
| `20 * * * *` | BWG | Inventory upload | Shell script (hourly) |
| `* * * * *` | BWG | POSLOG generation | Shell script (**every minute**) |
| `0 0 * * SUN` | BWG | Azpiral promotions update | `manage.py bwg_azpiral_promotions` (weekly) |
| `0 10 * * *` | Fast Eddy's | POSLOG generation | Shell script |
| `*/5 * * * *` | Fast Eddy's | Inventory import | Shell script (every 5 min) |
| `10 * * * *` | Eroski (×4 stores) | Inventory upload | Python script per store (hourly) |
| `15 7 * * *` | Muji | Inventory upload | Shell script |
| `30 21 * * *` | Muji | POSLOG generation (JP) | Shell script |
| `30 22 * * *` | Muji | POSLOG generation (UK) | Shell script |
| `30 5 * * *` | Muji | Promotions upload | Shell script |
| `0 4 * * *` | Flying Tiger | Inventory + promotions (DK→SE→NO) | Shell script (chained) |
| `5 20 * * *` | Flying Tiger | Finance POSLOG (DK→SE→NO) | Shell script (chained) |
| `30 21 * * *` | Flying Tiger | BI file generation | Shell script |

**Commented-out jobs** (legacy/disabled): Saturn, Decathlon (old), DirectWines, Compass, Bestseller/Veromoda, old thank-you emails, Lovbjerg, Paradies — totaling ~20 legacy entries.

**Key observations:**
- BWG POSLOG runs **every minute** — the most frequent job
- Decathlon payment verification runs **every 2 minutes**
- Most retailer data operations run daily between 2:00–7:00 UTC (inventory) and 20:00–22:00 UTC (POSLOG/reports)
- Eroski uses a standalone Python script (`/mpay/com.mishipay.server/scripts/eroski.py`) running outside the Docker container on the host
- All Docker-based jobs use `docker exec backend_v2 python3 manage.py <command>`

### SFTP Server (`scripts/crontabs/production_sftp`)

**Host:** `root@SFTPServer` + `sftp_downloads@SFTPServer`

Two crontab users manage the SFTP data flow:

#### Root User — Downloads from Retailer Servers

| Schedule | Retailer | Operation |
|----------|----------|-----------|
| `1 23 * * *` | Lagardere | Download via `lftp` from Lagardere FTP |
| `30 4,14 * * *` | Dufry | Download from Dufry SFTP (twice daily) |
| `30 1 * * *` | Picwic | Cleanup local CSV files older than 30 days |
| `* * * * *` | BWG | POSLOG upload (every minute) |
| `0 0 * * *` | BWG | Inventory file cleanup |
| `30 10 * * *` | Fast Eddy's | POSLOG upload |
| `*/5 * * * *` | Fast Eddy's | Inventory ACK upload (every 5 min) |
| `0 22 * * *` | Muji | POSLOG upload |
| `0 5 * * *` | Muji | Promotions file rename |
| `30 3 * * *` | Flying Tiger | Inventory download via `lftp` |
| `30 22 * * *` | Flying Tiger | POSLOG upload (Finance + BI) |
| `*/15 * * * *` | All retailers | `rsync` sync to `/home/sftp_downloads/retailers/` |

#### sftp_downloads User — Azure Blob Sync

Uses `rclone` to sync retailer data to Azure Blob Storage containers:

| Schedule | Retailer | Azure Container |
|----------|----------|-----------------|
| `5 * * * *` | BWG (Londis) | `bwg-londis` |
| `10 * * * *` | BWG (Spar) | `bwg-spar` |
| `30 5,15 * * *` | Dufry | `dufry` |
| `30 20 * * *` | Dufry | Cleanup: MasterData files older than 30 days |
| `20 * * * *` | Eroski | `rsync` from `mishipay_user@13.94.233.233:~/eroski/datos/` |
| `30 * * * *` | Eroski | `eroski` (Azure Blob) |
| `30 4 * * *` | Lagardere | `lagardere` |
| `30 2 * * *` | Picwic | `picwic` |
| `30 3 * * *` | Muji | `muji` |
| `45 2 * * *` | Fast Eddy's | `fast-eddys` |

**Data flow:**
```
Retailer Servers ──SFTP/FTP──► SFTP Server ──rsync──► /home/sftp_downloads/retailers/
                                                            │
                                                     rclone sync
                                                            │
                                                            ▼
                                                   Azure Blob Storage
                                                   (per-retailer containers)
```

**Acronis backup** runs on the SFTP server (daily + hourly schedules).

---

## Utility Scripts

### Helper Scripts (`scripts/helps/`)

| Script | Purpose |
|--------|---------|
| `find-nedap-gates.py` | Utility to locate Nedap RFID gates |
| `mqtt_keep_alive.py` | MQTT connection keep-alive utility |
| `retailer-helper.py` | General retailer operations helper |
| `test-socket.py` | WebSocket connectivity test |

### `scripts/mishipay_util.py`

Identical to the promotions service's `mishipay_util.py` — contains `elapsed_interval()` and `send_slack_message()` with the same hardcoded Slack bot token. Used by the Eroski cron scripts running on the host (outside Docker).

---

## Build Process

### Monolith Docker Image

The monolith does **not** have a build pipeline in its repository. Based on cross-references:

- The Docker image is `ghcr.io/mishipay-ltd/python-django-monolith-service:{version}` (production) or `ghcr.io/mishipay-ltd/python-django-monolith-service-ci:{version}` (CI/test)
- The build process likely exists in a separate CI system or repository
- The current production version (from `docker-compose-prod.yml`) is `v6.21.17`
- The Dockerfile in the repo defines the image build, but no pipeline in this repo triggers the build

### Promotions Service Build

Fully documented in [Promotions Service DevOps](../promotions-service/devops.md). Summary:
- Dual pipelines: Azure Pipelines (→ ACR) + GitHub Actions (→ GHCR)
- Pipeline: `flake8 → metadata → build-ci → test → version → release [→ helm_bump]`
- SemVer versioning via conventional commits
- Kubernetes deployment via Helm chart bumping

---

## Comparison: Monolith vs Promotions Service CI/CD

| Aspect | MishiPay Monolith | service.promotions |
|--------|------------------|--------------------|
| **CI Platform** | Azure Pipelines only | Azure Pipelines + GitHub Actions |
| **Build Pipeline** | Not in repo | In repo (full lint → build → test → release) |
| **Test Pipeline** | Makefile targets (local only) | Automated in CI (DB + manage.py check + migrations + test) |
| **Linting** | Makefile `flake8` (local) | CI pipeline flake8 (same rules) |
| **Deployment Target** | Azure VMSS | Kubernetes (Helm) |
| **Deployment Strategy** | Rolling VMSS update (with optional traffic drain) | Helm chart version bump → K8s rolling update |
| **Image Registry** | GHCR | ACR (production) + GHCR (release/dev) |
| **Version Strategy** | Manual version tags (in compose files) | SemVer via GitVersion/github-tag-action |
| **Trigger (prod)** | Manual (`trigger: none`) | Automatic (push to `master`) |
| **Trigger (test)** | Automatic (push to `test`) | Automatic (pull requests) |
| **Zero-Downtime** | Application Gateway drain (new pipelines) | K8s rolling update (built-in) |

---

## Ansible Production Deployment (HAProxy)

**Source:** `automation/prod-deployment/` (13 files)

In addition to the Azure VMSS pipelines documented above, the monolith repository contains an **Ansible-based deployment system** using HAProxy for zero-downtime rolling deploys. This is a separate deployment mechanism from the Azure Pipelines approach.

### Architecture

```
Ansible Control Machine
        │
   [serial: 1]
        │
        ▼
  For each backend_web server:
  ┌─────────────────────────────────┐
  │ 1. Disable in HAProxy (all LBs)│──delegate──► backend_lb hosts
  │ 2. Run deploy-prod.sh (sudo)   │             (HAProxy admin socket)
  │ 3. Enable in HAProxy (all LBs) │──delegate──► backend_lb hosts
  └─────────────────────────────────┘
```

### Playbook: `deploy-prod.yml`

The main playbook targets the `backend_web` group with `serial: 1` (one server at a time). It uses the `community.general` Ansible collection for the HAProxy module.

**Task sequence per server:**

| Step | Task | Mechanism |
|------|------|-----------|
| 1 | **Stop traffic** | Disable server in HAProxy via admin socket; delegates to all `backend_lb` hosts |
| 2 | **Deploy code** | Execute `/mpay/com.mishipay.server/automation/deploy-prod.sh` with sudo |
| 3 | **Start traffic** | Re-enable server in HAProxy; delegates to all `backend_lb` hosts |

**HAProxy module options** (shared anchor `&haproxy-opts`):
- `fail_on_not_found: false` — don't error if host missing from HAProxy
- `health: true` — check health status after state change
- `wait: true` — wait for state transition to complete
- `socket` — HAProxy admin socket path (from hostvars)

### Inventory: `hosts.yml`

**Global variables:**
- `ansible_user: mishipay_user` — SSH user for all hosts
- `ansible_python_interpreter: /usr/bin/python3`

**Active hosts:**

| Group | IP | Role | Host Var |
|-------|-----|------|----------|
| `backend_web` | `10.0.5.10` | Application server | `haproxy_host_name: prod-backend-7` |
| `backend_web` | `10.0.5.11` | Application server | `haproxy_host_name: prod-backend-8` |
| `backend_lb` | `10.0.3.20` | HAProxy load balancer | `haproxy_admin_sock_path: /run/haproxy/admin1.sock` |
| `backend_lb` | `10.0.3.21` | HAProxy load balancer | `haproxy_admin_sock_path: /run/haproxy/admin1.sock` |

**Commented-out hosts:** 4 additional backend servers (`10.0.5.12`, `10.0.5.14`, `10.0.5.15`, `10.0.5.16`) and 2 additional load balancers (`10.0.3.23`, `10.0.3.25`) — suggesting previous or planned capacity.

### Molecule Testing Framework

The deployment includes a full **Molecule** test suite (`molecule/default/`) for validating the Ansible playbook in isolated Docker containers.

**Test environment:**

| Container | Image | Role | Port |
|-----------|-------|------|------|
| `lb_1` | `mock_haproxy` (custom, from `haproxy:lts`) | HAProxy LB | 80 (web), 8404 (stats) |
| `backend_1` | `mock_backend` (custom, from `httpd`) | Mock web server | 80 |

Both containers share a `backend_net` Docker bridge network.

**Molecule workflow:**
1. `molecule create` — Build and start mock HAProxy + backend containers
2. `molecule converge` — Run `deploy-prod.yml` against mock infrastructure (via `converge.yml` which imports the main playbook)
3. `molecule verify` — Run TestInfra tests (`test_default_web.py`) to verify deployment script created `/deployed` marker file with correct ownership (`root:root`)
4. `molecule destroy` — Tear down containers

**Mock `deploy-prod.sh`** creates a `/deployed` marker file. The real production script (not in this directory) would handle the actual application deployment (Docker image pull, container restart, etc.).

**Linting:** yamllint (100-char max, truthy enforcement) + ansible-lint + flake8.

**Note from README:** The playbook is explicitly **not idempotent** — each step must be tested independently.

### Ansible vs Azure Pipelines Comparison

| Aspect | Ansible (HAProxy) | Azure Pipelines (VMSS) |
|--------|--------------------|------------------------|
| **LB Integration** | HAProxy admin socket (direct) | Azure Application Gateway API |
| **Target** | Individual VMs by IP | VMSS instances (auto-discovered) |
| **Traffic Drain** | HAProxy disable + health check | Remove from App Gateway pool + 60s sleep |
| **Deployment Script** | `deploy-prod.sh` (on-host) | CustomScript Extension (VM-level) |
| **Testing** | Molecule + Docker + TestInfra | None in repo |
| **Active Hosts** | 2 backend + 2 LB (with 6 commented out) | VMSS auto-scaling |

Both approaches implement zero-downtime deployment by removing servers from the load balancer before deploying. The Ansible approach is more granular (direct HAProxy socket control), while the Azure approach is cloud-native (VMSS + Application Gateway).

---

## Configuration Management (`conf/`)

**Source:** `conf/` (72 files)

The `conf/` directory contains all environment configuration files for every service in the platform. Files follow the naming pattern `{service}.{environment}.env` with environments: `local`, `dev`, `test`, `prod` (or `prd`).

### Configuration File Inventory

| Service | Files | Primary Database | Key Secrets |
|---------|-------|------------------|-------------|
| **Backend Monolith** | `local-variables.env.default`, `backend-variables.env` | PostgreSQL | Django secret key, HMAC key, Kafka creds, Azure Storage, encryption keys |
| **Payment Service (ms.pay)** | `ms.pay.{local,dev,test,prod}.env` (4 files) | MongoDB | Stripe keys (EU+US), TokenEx creds, Checkout.com HMAC, Razorpay key, Redis password |
| **Payment Gateway** | `ms.pay_gateway.{local,dev,test,prod}.env` (4 files) | N/A | Sentry credentials |
| **Tax Service (ms.tax)** | `ms.tax.{local,dev,test,prd}.env` (4 files) | MySQL | Database password, Sentry DSN |
| **Loyalty Service (ms.loyalty)** | `ms.loyalty.{local,dev,test,prd}.env` (4 files) | MySQL | Database password, Sentry DSN |
| **Promotions Service (ms.promo)** | `ms.promo.{local,dev,test,prd}.env` (4 files) | MySQL | Database password, Sentry DSN |
| **Voucher Service (ms.voucher)** | `ms.voucher.{local,dev,test,prod}.env` (4 files) | MySQL | Database password, Sentry DSN |
| **Keycloak (auth)** | `auth/keycloak.env` | PostgreSQL | Admin password, DB password |
| **Kong API Gateway** | `auth/kong/config.yml` | PostgreSQL | Admin key, API consumer keys, JWT issuer URLs |
| **Konga Admin UI** | `auth/konga/seed_nodes.data`, `auth/konga/seed_users.data` | — | Admin password (`adminpassword`) |
| **Datadog** | `datadog.{dev,test,prod}.env` (3 files) | — | DD_API_KEY |
| **MongoDB (Inventory)** | `mongodb.local.env` | — | Root password, app user password |
| **Voucher DB** | `voucherdb.{local,dev,test,prod}.env` (4 files) | MySQL | Root password, user password |
| **Tax DB** | `taxdb.{local,prod}.env`, `tax.{local,prod}.env` (4 files) | MySQL | Root password, user password |
| **PostgreSQL** | `postgres/postgresql.conf` | — | Performance tuning (no secrets) |

### Backend Monolith Configuration (`local-variables.env.default`)

The monolith's configuration defines connections to all microservices and external services:

**Microservice routing:**

| Variable | Service | Protocol |
|----------|---------|----------|
| `INVENTORY_SERVICE_*` | Inventory Service | HTTP |
| `TAX_SERVICE_*` | Tax Service (FastAPI) | HTTP |
| `TAX_SERVICE_GRPC_*` | Tax Service (legacy gRPC) | gRPC |
| `PAYMENT_SERVICE_*` | Payment Service | HTTP |
| `PROMO_SERVICE_*` | Promotions Service | HTTP |
| `LOYALTY_SERVICE_*` | Loyalty Service | HTTP |

**External service credentials:**
- `KAFKA_BOOTSTRAP_SERVER`, `KAFKA_API_KEY`, `KAFKA_API_SECRET` — Message queue
- `AZURE_STORAGE_CONNECTION_STRING` — Blob storage
- `HUDSON_SDK_KEY`, `HUDSON_SDK_IV` — Hudson loyalty decryption
- `SEVENTH_HEAVEN_ENCRYPTION_KEY` — Seventh Heaven encryption

### Kong API Gateway Configuration (`auth/kong/config.yml`)

The Kong declarative configuration defines the full API routing topology:

**Services:**

| Kong Service | Upstream Target | Purpose |
|-------------|-----------------|---------|
| `kong-admin` | `http://127.0.0.1:8001` | Kong Admin API |
| `keycloak-auth-service` | `http://ms_keycloak:8005` | Authentication IdP |
| `pay-gateway-service` | `http://pay-gateway:3000` | Payment gateway (Node.js) |
| `backend-v2-service` | `http://backend_v2:8080` | Monolith API |
| `static-ms-promo-service` | — | Promotions static files |
| `static-ms-loyalty-service` | — | Loyalty static files |

**Authentication:** JWT-Keycloak plugin protects the backend API; anonymous access for public routes.

**Consumers:** Multiple API consumers with key-auth credentials — `ios-consumer`, `web-consumer`, `android-consumer`, `dashboard-consumer`, `whitelabel-consumer`, and per-retailer variants.

**Certificates:** Wildcard certificate for `*.mishipay.com` domains.

### Nginx Configuration (`nginx_conf_local/`)

**Main config (`nginx.conf`):**
- Worker processes: auto-detect
- Max open files: 10,240
- Rate limiting: 5 req/s per IP, burst 20
- Connection limit: 20 per IP
- Gzip compression enabled
- Static asset caching: 1-year expiry for CSS/JS/fonts/images
- Access logging disabled for performance

**Virtual host (`conf.d/localhost.conf`):**
- Domains: `test.mishipay.com`, `dev.mishipay.com`, `localhost`
- SSL: Cloudflare-managed certificates for `*.mishipay.com`
- Static file aliases: `/static/`, `/static-ms-loyalty/`, `/static-ms-promo/`
- Route exclusions: `/silk` (debug toolbar) → 404, `/item-management/v1/promotions` → 404

**Supporting configs:**
- `mpay_proxy` — Reverse proxy to `backend_v2:8080` with header forwarding (`X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, `Host`)
- `cors_support` — CORS headers injected for `*.mishipay.com` origins
- `security_headers` — `X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`, XSS protection

### Gunicorn Configuration (`gunicorn/`)

**`gunicorn.conf.py`:**

| Setting | Value |
|---------|-------|
| Application | `mainServer.wsgi:application` |
| Workers | 3 |
| Threads | 4 per worker (`gthread` class) |
| Max requests | 3,000 (+ 250 jitter) |
| Keepalive | 5 seconds |
| Bind | `0.0.0.0:8080` |
| User/Group | `mpay` |
| StatsD | `ddagent:8125` (Datadog) |
| Logs | stdout (error + access) |

**`start.sh`:** Entry point wraps Gunicorn with `ddtrace-run` for Datadog APM tracing.

### Datadog Monitoring Configuration

**Environment files (`datadog.{dev,test,prod}.env`):**

| Variable | Purpose |
|----------|---------|
| `DD_API_KEY` | Datadog API authentication |
| `DD_SITE` | `datadoghq.eu` (EU region) |
| `DD_LOGS_ENABLED` | Log collection enabled |
| `DD_LOGS_INJECTION` | Log correlation with traces |
| `DD_TRACE_ENABLED` | APM tracing enabled |
| `DD_APM_ENABLED` | APM agent enabled |
| `DD_PROCESS_AGENT_ENABLED` | Process-level monitoring |
| `DD_DOGSTATSD_NON_LOCAL_TRAFFIC` | Accept StatsD from other containers |
| `DD_CONTAINER_EXCLUDE_LOGS` | Exclude specific container logs |

**Service log collection (`automation/datadog/`):**

| Config | Service | Log Path |
|--------|---------|----------|
| `python.conf.yaml` | `pay` (Payment Service) | `/logs/ms_pay/file.log` |
| `nodejs.conf.yaml` | `pay-gateway` (Payment Gateway) | `/logs/ms_pay_gateway/file.log` |

**Application logging (`instrumentation/logging.ini`):**
- Root logger: INFO level
- Handler: StreamHandler → stdout
- Format: JSON (timestamps, line numbers, message context)

### Secrets Management (`conf/secrets/`)

Per `secrets/README.md`, the `secrets/` directory (git-ignored except README) holds sensitive credentials:
- `com.gcc.firebase.dashboard.json` — Firebase credentials for Google Cloud dashboard
- Mounted read-only at `/home/app/conf/secrets/` in the `backend_v2` container

### PostgreSQL Tuning (`postgres/postgresql.conf`)

Key production tuning parameters:
- `shared_buffers = 2GB`
- Optimized for connection pooling
- Memory and performance settings tuned for production workload

### Configuration Templates (`templates/`)

Infrastructure service templates for new environment setup:

| Template | Service | Key Variables |
|----------|---------|---------------|
| `mongo-inventory.env.template` | MongoDB | Root creds, keyfile, replica set, app user |
| `elastic-inventory.env.template` | Elasticsearch | Username/password |
| `inventory.env.template` | Inventory Service (Node.js) | MongoDB connection, Elasticsearch, StatsD |

---

## Database Initialization

**Source:** `automation/postgres_init/`, `automation/ms.promo/`, `automation/ms.tax/`, `automation/docker/`

### PostgreSQL (Monolith Database)

**Source:** `automation/postgres_init/`

| File | Purpose |
|------|---------|
| `init.sh` | Creates `test_mpmain` user, `mpay` and `test_mpay` databases, restores dumps |
| `mpay.dump` | PostgreSQL custom-format dump for main database |
| `test_mpay.dump` | PostgreSQL custom-format dump for test database |
| `README.md` | Documentation |

**`init.sh` flow:**
1. Create role `test_mpmain` with password `mishipay@123!`
2. Create database `mpay` owned by `test_mpmain`
3. Create database `test_mpay` owned by `test_mpmain`
4. `pg_restore` the main dump into `mpay`
5. `pg_restore` the test dump into `test_mpay`

**Per README:** The `.dump` files should be updated no more than once a quarter. Django automatically applies any migrations newer than the dump. This avoids running the full migration chain (~500+ migrations) on fresh environments.

### MySQL — Promotions & Loyalty Services

**Source:** `automation/ms.promo/mysqld_init/`

| File | Purpose |
|------|---------|
| `000.sql` | Creates `test_ms_promo` database; grants all to `mpay_ms_promo` |
| `001.sql` | Creates `mpay_ms_loyalty` user (password: `mishipay@123!`), `ms_loyalty` and `test_ms_loyalty` databases |

**Note:** The loyalty service database init lives in the `ms.promo` directory (not `ms.loyalty`, which does not exist). Both the promo and loyalty MySQL databases are initialized from the same Docker MySQL container.

### MySQL — Tax Service

**Source:** `automation/ms.tax/`

| File | Purpose |
|------|---------|
| `mysqld_init/000.sql` | Creates `test_ms_tax` database; grants all to `mpay_ms_tax` |
| `migrations/1_taxlevel.up.sql` | Creates `tax_level` table with UUID PK, store/tax fields, unique constraint |
| `migrations/1_taxlevel.down.sql` | Drops `tax_level` table |

**`tax_level` table schema:**

| Column | Type | Constraints |
|--------|------|-------------|
| `id` | `BINARY(16)` | PK, default UUID |
| `store_id` | `VARCHAR(64)` | Part of unique key |
| `tax_code` | `VARCHAR(25)` | Indexed |
| `tax_level` | `VARCHAR(25)` | — |
| `retailer_tax_level_id` | `VARCHAR(25)` | Indexed, part of unique key |
| `tax_percent` | `NUMERIC(13,2)` | — |
| `created_at` | `TIMESTAMP` | Auto-populated |

**Unique constraint:** `(store_id, tax_code, retailer_tax_level_id)`

The tax service uses numbered SQL migration files (up/down pattern) rather than Django ORM migrations, consistent with it being a Go/FastAPI service rather than Django.

### MongoDB — Inventory Service

**Source:** `automation/docker/mongo-inventory/`

| File | Purpose |
|------|---------|
| `mongo-entrypoint.sh` | Custom Docker entrypoint: creates replica set keyfile from `MONGO_KEYFILE_TEXT` env var, sets `0400` permissions |
| `001_mongo_init.js` | JavaScript init: configures replica set (`mongo-inventory:27017`), creates root admin user (SCRAM-SHA-1), creates app user with `readWrite` on inventory + test databases |

**Initialization flow:**
1. Entrypoint creates keyfile for replica set authentication
2. Init script reads env vars: `MONGODB_REPLICA_SET`, `MONGODB_ROOT_USERNAME/PASSWORD`, `MONGODB_DB_NAME`, `MONGODB_USERNAME/PASSWORD`
3. Initializes replica set configuration
4. Polls until MongoDB becomes master (`db.isMaster().ismaster`)
5. Creates root admin user with full `root` role
6. Creates application user with `readWrite` on both production and test databases

### Database Initialization Summary

| Database | Type | Init Location | Users | Databases Created |
|----------|------|---------------|-------|-------------------|
| Monolith | PostgreSQL | `automation/postgres_init/` | `test_mpmain` | `mpay`, `test_mpay` |
| Promotions | MySQL | `automation/ms.promo/mysqld_init/` | `mpay_ms_promo` | `test_ms_promo` |
| Loyalty | MySQL | `automation/ms.promo/mysqld_init/` | `mpay_ms_loyalty` | `ms_loyalty`, `test_ms_loyalty` |
| Tax | MySQL | `automation/ms.tax/mysqld_init/` | `mpay_ms_tax` | `test_ms_tax` |
| Inventory | MongoDB | `automation/docker/mongo-inventory/` | root + app user | per env var |

All development/test database passwords use `mishipay@123!`.

---

## Eroski ETL Pipeline

**Source:** `scripts/eroski_cron.py`, `scripts/eroski_delta.py`, `scripts/eroski_item_recommendations_promotions_upload.py`, `scripts/eroski_item_reco_groups_data.py`

The most complex retailer integration pipeline runs **outside Docker** on the host machine.

### `eroski_cron.py` — Main ETL Pipeline

Called hourly at `:10` for 4 stores (0088, 0096, 0244, 0410).

**Flow:**
1. Connect to SFTP (`sftp_downloads@sftp.mishipay.com`)
2. Download `.DAT.TAR` delta files from Eroski
3. Extract and decompress archives
4. Run `strings` to extract ASCII content from binary `.DAT` files
5. Invoke `docker exec backend_v2 python3 manage.py eroski_inventory_and_promo_import`
6. Upload alcohol data separately
7. Compress results, archive processed files to `.APPLIED/` directory
8. Send Slack start/completion notifications with elapsed time

### `eroski_delta.py` — Error Recovery

Searches for `_delta.err` files in applied archives, converts them to recoverable `VAR*` files, bundles into a single TAR archive for reprocessing. Used for manual recovery after failed imports.

### `eroski_item_recommendations_promotions_upload.py` — Promotion Upload

Creates item recommendation promotions with nested product groupings. Batch-creates via HTTP POST to `http://localhost:8001/api/1.0/promotions/transaction/`. Targets 5 production store UUIDs with promotions valid until 2051.

### `eroski_item_reco_groups_data.py` — Recommendation Data

Contains 400+ product recommendation group definitions mapping product IDs to related items. Imported by the promotions upload script.

---

## Scripts Inventory — Extended

### One-Off Scripts (`scripts/oneoff/`)

**Source:** 48 files for critical one-time operations and bug fixes.

**Categories:**

| Category | Examples | Count |
|----------|----------|-------|
| **Configuration/Setup** | `mishipay.create_dashboard_token.py`, `decathlon.create_shopping_bags.py`, `store.chain.update.py` | ~8 |
| **Data Fixes/Migrations** | `bugs.GPP1046.py`, `relay.promo-fix.py` (40k items), `muji.ms-promo.prod_issues.py`, `eroski.ms-promo.fix.dates.py` | ~12 |
| **Data Integrity** | `mishipay.check_duplicate_barcodes.py`, `mishipay.delete_duplicate_product_identifiers.py`, `mishipay.check_outside_geo_fencing.py` | ~6 |
| **Data Export/Analysis** | `research.random_item.py`, `research.generate_barcode_db.py`, `mishipay.send_receipt.py` | ~5 |
| **Retailer-Specific** | `bwg.azpiral.manual_promotions.py`, `bwg.set_exit_qr_code.py`, `flying_tiger_promo_inv_script.py` | ~10 |
| **SQL Queries** | `sql/get_device_id_user_agent.sql`, `sql/get_user_promotion_details.sql`, `sql/get_item_info_for_completed_orders.sql` | 3 |

### Scratch Scripts (`scripts/SCRATCH_SCRIPTS/`)

Ad-hoc analytics scripts for Django shell execution:

| Script | Purpose |
|--------|---------|
| `checkout_time.py` | Analyze checkout duration from events CSV |
| `saturn_orders_sort_by_hour.py` | Group orders by hour for specific customer/store |
| `item_scanned.py` | Extract barcodes from item scan events; geopy distance calculation |
| `sales_report.py` | CSV export of orders, items, returns for date range |
| `nrf_agent_code.py` | NRF agent class: RFID item lifecycle management + SATO API integration |
| `saturn_epc_update.py` | Update Saturn EPC→ProductID mapping via CSV; sync sold status to Redis |
| `items_purchased_count.py` | Aggregate items purchased by customer between dates |
| `user_details.py` | Comprehensive customer profile: signup, transactions, geolocation, payments |
| `check_epc_in_gate.py` | Socket test to RFID gate (`gates.mishipay.com:8888`); latency measurement |
| `sending_notification.py` | Push notification broadcast: FCM/APNS, reads templates from files |
| `saturn_stock_check.py` | Inventory status by product ID; checks type mismatches in order items |

### Helper Scripts (`scripts/helps/`) — Expanded

| Script | Purpose | External Services |
|--------|---------|-------------------|
| `find-nedap-gates.py` | Network scan `192.168.1.*` for Nedap RFID gates (HTTP GET, timeout=0.3s) | Nedap gate HTTP |
| `mqtt_keep_alive.py` | MQTT publisher for item status changes; 5-retry error recovery with email alerts | MQTT broker (`35.176.179.166:1883`), Gmail SMTP |
| `retailer-helper.py` | REST API client for store/retailer onboarding: add stores, agents, generate dashboard tokens | Backend API |
| `test-socket.py` | Socket connection test to RFID gate (`app.mishipay.com:8888`); EPC reader latency | Socket server |

### Legacy Scripts (`scripts/old/`)

9 deprecated scripts from the pre-Docker era:
- Socket server implementations (`socketServer.py`, `socketServer_new.py`, `socketServer.py.old`)
- Legacy endpoint handlers (`payment_method.py`, `registration.py`, `feedback.py`)
- Payment integration (`transaction_platform.py`)
- Performance testing (`load_test.py`)
- Deployment hooks (`reconfigure_apache.sh`, `reconfigure_db.sh`, `post-receive.sh`, `post-receive.py`)
- Analytics query (`registrations-per-day.py`)

---

## Notable Patterns

1. **No monolith build pipeline in repo.** The Docker image build process is not defined in the monolith repository. The `Dockerfile` exists but no CI pipeline triggers it. The build likely happens in a separate system or is triggered by another repository. The Docker image build process is not defined in the monolith repository. The `Dockerfile` exists but no CI pipeline triggers it. The build likely happens in a separate system or is triggered by another repository.

2. **Dual production pipelines.** Two sets of prod pipelines exist (`*-prod.yml` and `*-prod-new.yml`) with different Azure subscription names and deployment strategies. The "new" versions use `ARM-AzureSubscription1` and implement graceful traffic drain via Application Gateway manipulation.

3. **VMSS-based deployment.** The monolith runs on Azure VMSS (not Kubernetes). The CustomScript extension mechanism pulls and runs new Docker images when triggered by `az vmss extension set --force-update`.

4. **Heavy cron job reliance.** Production runs 20+ active cron jobs on the backend server and 15+ on the SFTP server. This is a significant operational burden that could be migrated to a scheduled task framework (e.g., Airflow, Azure Functions, or Dramatiq periodic tasks).

5. **Two-server SFTP data flow.** Retailer data flows through a dedicated SFTP server that acts as an intermediary: downloading from retailer FTP/SFTP servers, syncing locally, then uploading to Azure Blob Storage via `rclone`.

6. **Legacy deployment scripts.** `deploy.sh` uses `rsync` + `uwsgi` restart — predating the Docker/VMSS era. Still committed but likely unused.

7. **Manual prod deployments.** All production pipelines have `trigger: none`, requiring manual execution from Azure DevOps. This is a safety measure but means deployments depend on human action.

8. **No automated testing in CI.** Unlike the promotions service which runs automated tests in CI, the monolith's test infrastructure is local-only (Makefile targets). The Docker image build (which presumably includes tests) is not in this repo.

9. **Dual deployment systems coexist.** The Ansible/HAProxy deployment (`automation/prod-deployment/`) and the Azure Pipelines/VMSS deployment both target production backends. The Ansible inventory has 2 active backend servers (10.0.5.10, 10.0.5.11) while the VMSS approach manages instances dynamically. It is unclear which is the primary active system, or whether they target different server pools.

10. **72 configuration files in `conf/`.** Every microservice has 4 environment variants (local/dev/test/prod). Secrets appear in plain text across all env files — database passwords, Stripe keys, TokenEx credentials, API keys. The `secrets/` subdirectory (git-ignored) is intended for sensitive files, but most secrets are in the env files themselves.

11. **Database initialization via dumps.** PostgreSQL init uses pre-built `pg_dump` files updated quarterly, bypassing the full migration chain. This is pragmatic but creates a dependency on keeping dumps current.

12. **MongoDB replica set for inventory.** The inventory MongoDB uses a replica set with keyfile authentication, even in development. The custom entrypoint + init JS creates both root and application users with scoped permissions.

13. **Eroski runs outside Docker.** The Eroski ETL pipeline (`eroski_cron.py`) runs directly on the host machine, connecting to SFTP, extracting binary `.DAT` files, and invoking Docker commands. This is the only retailer integration that runs outside the container.

14. **48 one-off scripts committed.** The `scripts/oneoff/` directory contains data fixes, bug remediation scripts, and setup utilities — some referencing specific JIRA tickets (e.g., `bugs.GPP1046.py`). These represent institutional knowledge of past production issues.

15. **Ansible playbook is not idempotent.** Explicitly stated in the README. The `deploy-prod.sh` script is designed for single execution, not repeated runs. This limits automated retry on failure.

16. **Kong declarative config defines the full API topology.** The `auth/kong/config.yml` maps all services, routes, consumers, and authentication — a single YAML file that is the authoritative source for API gateway routing.

---

## Open Questions

### From T34 — CI/CD Pipelines

- Where is the monolith Docker image build pipeline? It's not in the MishiPay repository. Is it in a separate CI/CD configuration, or triggered by Azure DevOps pipeline definitions stored elsewhere?
- Are the "old" prod pipelines (`azure-pipelines-vmss-prod.yml`, `azure-pipelines-vmss-consumers-prod.yml`) still used, or have they been fully superseded by the "new" versions with traffic drain?
- What CustomScript extension is installed on the VMSS instances? The `az vmss extension set --force-update` command doesn't specify the script content — it relies on an already-configured extension that pulls and runs the latest Docker image.
- Is the `AzureResourceManager (Pay-as-you-go) - Production RG` subscription still active, or has it been fully migrated to `ARM-AzureSubscription1`?
- The 60-second drain period in `vmss-prod-new.yml` — is this sufficient for long-running requests (e.g., promotion evaluation, payment processing)?
- Are the cron schedules in `production_backend` and `production_sftp` the current production schedules, or are they snapshots from a specific point in time?
- Is there a plan to migrate from VMSS to Kubernetes for the monolith, similar to the promotions service?

### From T35 — Ansible Deployment & Configuration

- Is the Ansible/HAProxy deployment the **active** production deployment mechanism, or has it been superseded by the Azure Pipelines/VMSS approach? The `hosts.yml` has 4 of 6 backend servers and 2 of 4 load balancers commented out, suggesting reduced capacity or migration.
- What does the real production `deploy-prod.sh` script do? The `automation/prod-deployment/molecule/default/files/deploy-prod.sh` is a mock (creates `/deployed` marker). The actual production script at `/mpay/com.mishipay.server/automation/deploy-prod.sh` is not in the repository.
- Are the `conf/*.prod.env` and `conf/*.prd.env` files the actual production configuration, or are they templates? Some files contain what appear to be real API keys and tokens.
- The Konga admin UI has hardcoded credentials (`admin`/`adminpassword` in `auth/konga/seed_users.data`). Is this changed in production?
- Kong config (`auth/kong/config.yml`) defines API consumers with key-auth credentials in plain YAML. Are these the actual production API keys, or are they overridden at runtime?
- How is the `conf/` directory deployed to production? Are env files baked into Docker images, mounted as volumes, or injected via a secrets manager?
- The `automation/ms.loyalty/` directory does not exist. Loyalty service database init lives in `automation/ms.promo/mysqld_init/001.sql`. Is this intentional, or should the loyalty init be in its own directory?
- How often are the PostgreSQL dump files (`automation/postgres_init/mpay.dump`, `test_mpay.dump`) actually updated? The README says quarterly, but is this followed?
- ~~How is the Eroski cron job running outside Docker on the host?~~ **Answered in T35:** The `eroski_cron.py` script runs directly on the host via cron (`/mpay/com.mishipay.server/scripts/eroski_cron.py`), connects to SFTP for data download, uses the `strings` shell utility to extract ASCII from binary `.DAT` files, then invokes `docker exec backend_v2 python3 manage.py ...` for the actual Django import. It requires host-level Python 3 with `paramiko` (SFTP), `subprocess`, and access to the `scripts/mishipay_util.py` Slack utility.
- The `nginx.conf` rate limiting (5 req/s per IP) — is this active in production, or does Cloudflare/Kong handle rate limiting upstream?
- Is the `conf/postgres/postgresql.conf` (2GB shared_buffers) actually used in production, or is PostgreSQL configured separately on the production VMSS/VM?

---

## Dependencies

- **Azure DevOps:** Pipeline execution, variable management
- **Azure VMSS:** Production VM hosting
- **Azure Application Gateway:** Load balancing (used by "new" pipeline for traffic drain)
- **Azure Blob Storage:** Retailer data sync (via `rclone`)
- **GitHub Container Registry:** Docker image hosting
- **SFTP Server:** Retailer file exchange hub
- **Cron daemon:** Production job scheduling on both backend and SFTP servers
- **Ansible:** HAProxy-based deployment automation (`automation/prod-deployment/`)
- **HAProxy:** Load balancer for Ansible-deployed backend servers
- **Kong API Gateway:** Request routing, JWT auth, API consumer management
- **Keycloak:** Identity provider (OIDC/JWT)
- **Nginx:** Reverse proxy, static files, rate limiting, CORS
- **Gunicorn:** WSGI application server (gthread, 3×4 workers)
- **Datadog:** APM, logs, tracing, StatsD metrics (EU region)
- **Sentry:** Error tracking for all microservices
- **Molecule:** Ansible playbook testing framework (Docker-based)

See [Infrastructure — Docker & Service Architecture](../03-infrastructure.md) for Docker compose, networking, and database details.
See [Promotions Service DevOps](../promotions-service/devops.md) for the promotions service CI/CD pipeline.
