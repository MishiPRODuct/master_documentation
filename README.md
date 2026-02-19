# MishiPay Platform — Executive Summary

> A single-document overview of the entire system and how everything ties together.

---

## What MishiPay Is

MishiPay is a **scan-and-go retail technology platform**. Customers walk into a store, scan items with their phone (or at a kiosk), pay in-app, and leave — no traditional checkout required. The platform handles everything behind the scenes: product lookup, promotions, payments, receipts, loyalty, analytics, and retailer-specific compliance.

The system serves **100+ retailer integrations** across multiple countries, from Dubai Duty Free to Flying Tiger.

---

## The Three Repositories

The backend is split across three codebases:

```
┌─────────────────────────────────────────────────────────┐
│           MishiPay Monolith (Django 4.1)                │
│           ~2,955 Python files, 48+ Django apps          │
│                                                         │
│  Items · Orders · Payments · Customers · Analytics      │
│  Dashboard · Inventory · Emails · Coupons · Loyalty     │
│  100+ retailer integrations · Kafka pub/sub             │
└──────────┬──────────────────────┬───────────────────────┘
           │ HTTP REST            │ imports as library
           ▼                      ▼
┌──────────────────────┐  ┌──────────────────────────────┐
│ Promotions Service   │  │ Inventory Common Library      │
│ (Django 4.2)         │  │ (python-inventory-common)     │
│ ~106 Python files    │  │ ~321 Python files, ~47K lines │
│                      │  │                                │
│ 8 promo families     │  │ HTTP client for Inventory Svc  │
│ Best-discount algo   │  │ Pandas ETL framework           │
│ Basket evaluation    │  │ 44+ retailer import scripts    │
└──────────────────────┘  └──────────────────────────────┘
```

The **monolith** does almost everything. The **promotions service** was extracted because discount evaluation is computationally complex (graph-based clustering) and benefits from independent scaling. The **inventory common library** is a shared Python package providing the HTTP client and 44+ retailer-specific import scripts that feed product data from POS/ERP/e-commerce systems into MishiPay's Inventory Service.

---

## How a Customer Purchase Works (End to End)

This is the core flow that ties every module together:

```
 CUSTOMER SCANS ITEM
      │
      ▼
 ┌─ Items Module ─────────────────────────────────────────┐
 │  Barcode lookup → ItemInventory or Inventory Service   │
 │  Age-restricted? → ManualVerificationSession           │
 │  Add to Basket (customer + store scoped)               │
 └────────────────────────┬───────────────────────────────┘
                          │
      ▼                   ▼
 ┌─ Promotions ───────────────────────────────────────────┐
 │  Monolith sends basket → Promotions Microservice       │
 │  Engine evaluates all active promos for this store     │
 │  Best-discount algorithm picks optimal combination     │
 │  Coupons/vouchers applied via special_promos JSON      │
 │  BasketEntityAuditLog records what was applied         │
 └────────────────────────┬───────────────────────────────┘
                          │
                          ▼
 ┌─ Payments ─────────────────────────────────────────────┐
 │  PaymentVariant selects PSP for this store             │
 │  Legacy path: 22 direct PSP integrations               │
 │  New path: MS Pay microservice (35+ gateways)          │
 │  3DS authentication if required                        │
 │  Preauth → Capture on order completion                 │
 └────────────────────────┬───────────────────────────────┘
                          │
                          ▼
 ┌─ Order Completion ─────────────────────────────────────┐
 │  Assign o_id (sequential per store)                    │
 │  Assign transaction_id_poslog (for retailer POS)       │
 │  VAT / tax calculation (ms-tax via gRPC or HTTP)       │
 │  Retailer-specific fulfilment (30+ implementations)    │
 │  Receipt generation (email + POS printer)              │
 │  Coupon redemption via MS Loyalty Service              │
 │  RFID tag deactivation (if applicable)                 │
 │  Kafka event → post-payment-success topic              │
 │  Analytics event recorded (Subject-Verb-Object)        │
 └────────────────────────────────────────────────────────┘
```

Every module in the codebase exists to support some part of this flow, or to provide operational tooling around it (dashboard, analytics, inventory management).

---

## Service Architecture

The monolith doesn't run alone. It talks to several microservices:

```
                    ┌──────────────┐
                    │   Kong API   │ ◄── Validates Keycloak JWTs
                    │   Gateway    │     Sets x-mishipay-id header
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Monolith   │ ◄── Django + Gunicorn + Nginx
                    │  (backend)   │     Azure VMSS deployment
                    └──┬──┬──┬──┬──┘
                       │  │  │  │
          ┌────────────┘  │  │  └────────────┐
          ▼               ▼  ▼               ▼
    ┌───────────┐  ┌─────────────┐  ┌──────────────┐
    │ ms-promo  │  │   MS Pay    │  │  Inventory   │
    │ (promos)  │  │ (payments)  │  │  (Node.js)   │
    │ Django    │  │ 35+ PSPs    │  │  MongoDB     │
    │ MySQL     │  │             │  │              │
    └───────────┘  └─────────────┘  └──────────────┘
          ┌────────────┘  │
          ▼               ▼
    ┌───────────┐  ┌─────────────┐  ┌──────────────┐
    │  ms-tax   │  │ MS Loyalty  │  │   Kafka      │
    │  (tax)    │  │ (coupons/   │  │ (Confluent)  │
    │  gRPC +   │  │  referrals) │  │  6 topics    │
    │  HTTP     │  │             │  │  3 consumers │
    └───────────┘  └─────────────┘  └──────────────┘
```

**Communication patterns:**
- **Synchronous HTTP** — Monolith ↔ all microservices for request-response
- **gRPC** — Monolith → ms-tax (legacy path)
- **Kafka** — Async events after payment/refund/verification (6 producer types, 3 consumer daemons)
- **RabbitMQ/Dramatiq** — Internal background jobs (refunds, analytics, third-party calls)
- **MQTT** — Real-time RFID tag status notifications to store devices

---

## The Module Map

Here's how the 48+ Django apps in the monolith break down by responsibility:

### Core Infrastructure
| Module | What It Does |
|--------|-------------|
| `dos` (363 files) | Foundation layer — Store, Customer, Order, Item models. StoreType pattern dispatches retailer-specific behaviour. 100+ store types. |
| `mishipay_core` (211 files) | Shared utilities, permissions, auth backends, entity properties, config wrappers, 97 common functions |
| `mishipay_authentication` | Guest checkout sessions using MishiPay ID |
| `mishipay_config` | GlobalConfig key-value store for feature flags |

### Commerce
| Module | What It Does |
|--------|-------------|
| `mishipay_items` (649 files) | Largest module. Items, baskets, wishlists, barcode scanning, promotion evaluation, 74+ store-type-specific procedures |
| `mishipay_retail_orders` (223 files) | Order lifecycle, sequential IDs, fulfilment dispatch to 30+ retailers, receipt generation, RFID tracking, offline sync |
| `mishipay_retail_payments` (259 files) | Dual payment system (legacy 22 PSPs + MS Pay 35+ gateways), 3DS, preauth/capture, payouts, saved cards |
| `mishipay_retail_customers` | Customer profiles, 15+ loyalty integrations, privacy/marketing consent, addresses |

### Promotions & Discounts
| Module | What It Does |
|--------|-------------|
| `service.promotions` (separate repo) | Discount evaluation engine — 8 promotion families, graph-based best-discount algorithm |
| `mishipay_coupons` | Coupon validation/redemption via MS Loyalty Service |
| `mishipay_special_promos` | Vouchers, on-the-fly discounts, lottery codes, donations |
| `promotions` (82 files) | Promotion data import — parses retailer CSV/ASC/XML files, pushes to ms-promo |
| `mishipay_subscription` | Coffee subscription (Stripe recurring, daily usage limits) |
| `mishipay_referrals` | Referral campaigns via MS Loyalty Service |

### Operations & Staff Tools
| Module | What It Does |
|--------|-------------|
| `dashboard` + `mishipay_dashboard` | Staff portal — order management, refunds, inventory, settlement, LLM chat |
| `mishipay_cashier_kiosk` | Kiosk shift management, cash tracking, Kafka events |
| `mishipay_retail_analytics` | Event-sourced analytics (Subject-Verb-Object), V1+V2 dashboard APIs |
| `mishipay_retail_jobs` | Dramatiq background tasks (refunds, HTTP calls, POS logs) |

### Support & Integration
| Module | What It Does |
|--------|-------------|
| `inventory` (119 files) | Proxy to Node.js Inventory Service + batch import pipeline (35 management commands) |
| `mishipay_emails` (136 files) | Email config + POS receipt engine with 25 retailer strategies |
| `third_party_integrations` | OAuth2 provider (Decathlon SSO), Red Discount loyalty tiers |
| `pubsub` | Kafka messaging framework (6 producers, 3 consumers) |
| `fiscalisation` | Tax compliance — Italy (Venistar) and Greece (AADE myDATA) |
| `easy_store_creation` | Self-service store provisioning with template cloning |
| `admin_actions` | Django admin operational tooling |

---

## The Key Pattern: Store-Type Dispatch

The single most important architectural pattern in the codebase is **store-type dispatch**. Nearly every module uses it:

```python
# Simplified example of the pattern used everywhere
DISPATCH_MAP = {
    "decathlon": DecathlonHandler,
    "eroski": EroskiHandler,
    "ddf": DDFHandler,
    "flying_tiger": FlyingTigerHandler,
    # ... 100+ more
}

handler = DISPATCH_MAP.get(store.store_type.lower(), DefaultHandler)
handler.process(request)
```

This is how the platform supports 100+ retailers with different:
- Item lookup and barcode formats
- Payment gateways and flows
- Order fulfilment procedures
- Receipt layouts
- Loyalty integrations
- Fiscal compliance requirements

Each retailer gets a `StoreType` subclass (in `dos/`) and per-module dispatch entries. Adding a new retailer means implementing these dispatch targets — no core logic changes needed.

---

## Data Storage

| Database | What It Stores |
|----------|---------------|
| **PostgreSQL** | Monolith primary database — all Django models (stores, orders, customers, items, analytics, etc.) |
| **MySQL** | Promotions microservice — promo headers, groups, nodes |
| **MongoDB** | Inventory Service — canonical product catalog |
| **Redis** | Session cache, RFID sold/unsold status (EPC tags), rate limiting |
| **RabbitMQ** | Dramatiq task queue for background jobs |
| **Kafka (Confluent Cloud)** | Async event streaming — 6 topics for post-payment, refund, verification, shift events |

---

## Deployment

| Component | Where | How |
|-----------|-------|-----|
| **Monolith** | Azure VMSS | Azure Pipelines → Docker image → VMSS CustomScript pull. HAProxy for zero-downtime deploys. Ansible for config management. |
| **Promotions Service** | Kubernetes | GitHub Actions + Azure Pipelines → Docker image → Helm chart bump → K8s rollout |
| **Consumer workers** | Separate Azure VMSS | Same image as monolith, different entrypoint (Kafka/Dramatiq consumers) |

---

## How to Navigate the Full Documentation

Start here, then drill into what you need:

1. **Architecture & infrastructure** → [Architecture Overview](./01-architecture-overview.md), [Infrastructure](./03-infrastructure.md)
2. **How data flows end-to-end** → [Data Flow](./cross-cutting/data-flow.md)
3. **All API endpoints** → [Unified API Reference](./cross-cutting/api-reference.md) (~390 endpoints)
4. **Specific module** → Use the [Index](./00-index.md) to find the right doc
5. **Unfamiliar term** → [Glossary](./cross-cutting/glossary.md) (70+ terms)
6. **Promotions engine deep-dive** → Start at [Promotions Overview](./promotions-service/overview.md)
7. **Retailer integration pattern** → [Retailer Modules](./platform/retailer-modules.md)
8. **Inventory import pipeline** → [Inventory Common Library](./platform/inventory-common.md) (44+ retailer ETL scripts)
