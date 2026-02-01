# Architecture Overview

> Last updated by: Task T00 — Project Overview & Architecture Map

## Overview

MishiPay is a retail technology platform that enables scan-and-go shopping. The system is composed of a **Django monolith** (the main platform) and several **microservices**, the most significant being the **promotions service**. The monolith handles the majority of business logic — items, orders, payments, customers, analytics — while microservices handle specialized concerns like promotions evaluation, tax calculation, payments orchestration, loyalty, vouchers, inventory, and authentication.

## Repositories

### 1. MishiPay (Main Monolith)

- **Path:** `MishiPay/`
- **Type:** Django 4.1.13 monolith
- **Scale:** ~2,955 Python files across 48+ Django apps
- **Database:** PostgreSQL 11.8
- **Container name:** `backend_v2`
- **Port:** 8080 (internal), exposed via Kong API Gateway on port 80/443
- **Purpose:** Core retail platform — handles items, orders, payments, customers, analytics, retailer integrations, and the admin dashboard

### 2. service.promotions (Promotions Microservice)

- **Path:** `service.promotions/`
- **Type:** Django 3.2.24 microservice
- **Scale:** ~106 Python files, 1 Django app (`mpromo`)
- **Database:** Percona MySQL 8.0
- **Container name:** `ms-promo`
- **Port:** 8001
- **Purpose:** Ingests promotional data and evaluates discounts on customer baskets. Implements a sophisticated discount engine with 8 promotion family types and a best-discount algorithm using graph-based clustering.

## System Architecture

```
                         ┌──────────────────────────────┐
                         │     Kong API Gateway          │
                         │  (ms_gateway — port 80/443)   │
                         └──────────┬───────────────────┘
                                    │
               ┌────────────────────┼────────────────────┐
               │                    │                     │
               ▼                    ▼                     ▼
  ┌────────────────────┐ ┌──────────────────┐ ┌──────────────────┐
  │    backend_v2      │ │   Keycloak       │ │   Static Files   │
  │  (Django Monolith) │ │ (ms_keycloak)    │ │   (NGINX)        │
  │  Port: 8080        │ │ Port: 8005       │ │                  │
  └──────┬─────────────┘ └──────────────────┘ └──────────────────┘
         │
         │  Internal Network (mpay_net)
         │
    ┌────┼──────┬──────────┬──────────┬──────────┬──────────┐
    │    │      │          │          │          │          │
    ▼    ▼      ▼          ▼          ▼          ▼          ▼
  ┌────┐┌────┐┌─────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
  │PG  ││Red.││Rab. │ │ms-promo│ │ms-loy. │ │ms-tax  │ │ms-pay  │
  │DB  ││    ││MQ   │ │  :8001 │ │  :8002 │ │  :9000 │ │  :8003 │
  └────┘└────┘└─────┘ └───┬────┘ └───┬────┘ └────────┘ └───┬────┘
                           │          │                      │
                       ┌───┴────┐ ┌───┴────┐            ┌───┴────┐
                       │MySQL   │ │MySQL   │            │MongoDB │
                       │(Perc.) │ │(shared)│            │        │
                       └────────┘ └────────┘            └────────┘

  Additional Services:
  ┌──────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────────┐
  │pay-gate  │ │voucher   │ │inventory svc │ │  Datadog     │
  │way :3000 │ │  :8006   │ │(+Mongo+ES)   │ │  Agent       │
  └──────────┘ └──────────┘ └──────────────┘ └──────────────┘
```

## Services Summary

| Service | Container | Image | Port | Database | Purpose |
|---------|-----------|-------|------|----------|---------|
| **Django Monolith** | `backend_v2` | `python-django-monolith-service` | 8080 | PostgreSQL 11.8 | Core platform: items, orders, payments, customers, analytics |
| **Promotions** | `ms-promo` | `ms-promo` | 8001 | Percona MySQL 8.0 | Promotion evaluation engine, discount calculation |
| **Loyalty** | `ms-loyalty` | `python-django-loyalty-service` | 8002 | MySQL (shared with promo) | Customer loyalty program |
| **Tax (legacy)** | `ms-tax` | `service.tax/ms_tax` | 9000 | MySQL | Tax calculation via gRPC |
| **Tax (new)** | `mishi-tax` | `mishipay-tax-service` | 8008 (→80) | MySQL | Updated tax service |
| **Payment Service** | `mishi-pay` | `mishipay-payment-service` | 8003 | MongoDB | Payment orchestration |
| **Payment Gateway** | `mishi-payment-gateway` | `ms_pay_gateway` | 3000 | — | Node.js payment gateway proxy |
| **Voucher** | `mishi-voucher` | `ms_voucher` | 8006 (→80) | MySQL | Voucher management |
| **Inventory** | `inventory_service` | `inventory-service` | — | MongoDB + Elasticsearch | Inventory management and search |
| **Keycloak** | `ms_keycloak` | `service.auth/ms_keycloak` | 8005 | PostgreSQL (shared) | Identity and access management (IAM) |
| **Kong Gateway** | `ms_gateway` | `service.auth/ms_gateway` | 80/443 | — | API gateway, routing, SSL termination |
| **Kafka Consumers** | `consumer_*` | same as monolith | — | PostgreSQL (shared) | Event-driven post-payment processing, analytics |
| **RabbitMQ** | `mpay_rabbitmq` | `rabbitmq:3.8.5-alpine` | — | — | Message broker for Dramatiq background tasks |
| **Redis** | `mpay_redis` | `redis:6.0.6-alpine` | — | — | Task result backend, caching |
| **Memcached** | `memcached_v2` | `memcached:1.6.6-alpine` | — | — | Caching (production) |
| **NGINX** | `reverse_proxy_v2` / `static_files` | `nginx:1.21.1` | — | — | Static file serving, reverse proxy |
| **Datadog Agent** | `dd-agent` | `gcr.io/datadoghq/agent:7` | 8126 | — | APM, monitoring, log collection |

## Networking

The platform uses two Docker networks to enforce security isolation:

| Network | Type | Purpose |
|---------|------|---------|
| `mpay_pub` | `internal: false` | Public network — services that need internet access (3rd-party APIs, Sentry, Kafka). In dev, the monolith is **disconnected** from this network by default for security. |
| `mpay_net` | `internal: true` | Private inter-service network. All services communicate here. No external internet access. |

**Key networking rules:**
- The Django monolith is restricted to `mpay_net` in development (no internet). Developers must uncomment `mpay_pub` to enable external API access.
- In production, the monolith connects to both networks.
- The Kong API Gateway (`ms_gateway`) is the only public-facing entry point (ports 80/443).

## Kafka Consumers (Event-Driven)

The monolith runs dedicated Kafka consumer processes as separate containers sharing the same image:

| Consumer | Container | Purpose |
|----------|-----------|---------|
| `consumer_post_payment_success` | `consumer_post_payment_success` | Post-payment processing (receipts, inventory updates, notifications) |
| `consumer_common` | `consumer_common` | General-purpose event consumption |
| `consumer_payment_only_clients_analytics` | `consumer_payment_only_clients_analytics` | Analytics processing for payment-only clients |

All consumers use Confluent Kafka (managed service) and connect to both `mpay_pub` (for Kafka) and `mpay_net` (for the database). They share the same environment variables and database as the monolith.

## Background Task Processing

The monolith uses **Dramatiq** with **RabbitMQ** as the message broker and **Redis** as the result backend. This handles asynchronous tasks like email sending, report generation, and data processing.

```
Django (backend_v2)  →  RabbitMQ (mpay_rabbitmq)  →  Dramatiq Workers
                                                        ↓
                                                   Redis (result backend)
```

## Service Communication Patterns

1. **REST API (HTTP):** The monolith communicates with microservices (`ms-promo`, `ms-loyalty`, `ms-pay`, `ms-tax`, `voucher`, `inventory`) via internal HTTP calls over `mpay_net`.
2. **gRPC:** The monolith communicates with the legacy tax service (`ms-tax:9000`) via gRPC. Proto definitions are in `MishiPay/mainServer/protos/`.
3. **Kafka:** Event-driven processing via Confluent Kafka. Consumers run as separate containers from the same monolith image.
4. **RabbitMQ + Dramatiq:** Async task queuing for background jobs within the monolith.
5. **Kong API Gateway:** All external client requests route through Kong, which handles authentication (Keycloak), rate limiting, and request routing.

## Development vs Production Differences

| Aspect | Development | Production |
|--------|------------|------------|
| Database | Local PostgreSQL container | External managed database |
| Monolith networking | `mpay_net` only (no internet) | `mpay_pub` + `mpay_net` |
| Django server | uWSGI / `manage.py runserver` | Gunicorn |
| Static files | NGINX container | NGINX + WhiteNoise |
| Monitoring | Optional Datadog agent | Datadog agent + APM tracing |
| Caching | Redis | Redis + Memcached |
| Kafka | Confluent Cloud | Confluent Cloud |
| Image versioning | `local` tag | SemVer tags (e.g., `v6.21.17`) |

## Release Process

### Normal Release
1. Open PR `test` → `master` (named "Production Release YYMMDD")
2. Ensure CI passes
3. Merge and inform DevOps for complete (migrations + Docker) or fast (git-pull) deployment

### HotFix Process
1. Cherry-pick specific commits from `test` into a `hotfix-*` branch off `master`
2. Open PR `hotfix-*` → `master`, wait for CI
3. After merge, open reverse PR `master` → `test` to sync changes back

**Branch strategy:**
- `test` — integration/staging branch (squash merge from feature branches)
- `master` — production branch (regular merge from `test`)
- Feature branches — named with Jira ticket numbers

## Open Questions

- What is the exact Kafka topic structure and message format? (To be documented in infrastructure tasks)
- How does the payment gateway (`pay-gateway`, Node.js) interact with the Python payment service (`mishi-pay`)? (To be documented in payments tasks)
- What are the specific Keycloak realm/client configurations? (To be documented in authentication tasks)
- How does the inventory service (separate repo) integrate with the monolith? (To be documented in integrations tasks)
