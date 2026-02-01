# Cross-Cutting — Data Flow & Event-Driven Architecture

> Last updated by: Task T37 — Comprehensive Data Flow Documentation

## Overview

This document maps how data flows through the MishiPay platform — from user actions through API processing, inter-service communication, event-driven messaging, and storage. The system uses a combination of synchronous HTTP/gRPC calls for request-response flows and asynchronous Kafka messaging for post-transaction processing.

This is the primary reference for understanding how data moves end-to-end across the platform. Individual module docs cover implementation details; this document connects them into coherent flows.

---

## Communication Protocols

The platform uses 5 inter-service communication methods:

| Protocol | Used For | Services |
|----------|----------|----------|
| **HTTP REST** | Synchronous request-response | Monolith ↔ ms-promo, ms-loyalty, ms_tax (new), pay, voucher, inventory |
| **gRPC** | Synchronous request-response (legacy) | Monolith → ms-tax (legacy) |
| **Kafka** | Asynchronous event streaming | Monolith producers/consumers ↔ Confluent Cloud |
| **RabbitMQ/Dramatiq** | Background task queue | Monolith internal (Dramatiq workers) |
| **In-process HTTP** | Internal module routing | `microservices_client` (Django test `Client()` via `reverse()`) |

---

## Event-Driven Architecture

### Kafka Topology

```
                    Confluent Cloud (West Europe Azure)
                    ══════════════════════════════════

  PRODUCERS (Monolith)                      CONSUMERS (Monolith Containers)
  ════════════════════                      ═══════════════════════════════

  PostPaymentSuccess ──►  mishi.streams.post-payment-success  ──► PostPaymentSuccessConsumer
  ProducerAdapter              │                                      │ group: PAYMENT_GROUP
                               │                                      ▼
                               │                               PostPaymentSuccessHandler
                               │                                 → PaymentService async tasks
                               │                                 → (fulfilment, receipts, etc.)

  PostRefundSuccess  ──►  mishi.streams.common  ──────────────► CommonConsumer
  ProducerAdapter              ▲                                  │ group: COMMON_GROUP
                               │                                  │ Routes on consumer_identifier_key:
  PostDashboard     ──────────┘                                  │
  VerficationAdapter           ▲                                  ├─ POST_REFUND_SUCCESS
                               │                                  │    → PostRefundSuccessHandler
  (External service) ──► mishi.streams.post-transaction-success  │    → Store-type refund fulfilment
                                                                  │    → Receipt email
                                                                  │
                                                                  ├─ POST_DASHBOARD_VERIFICATION_SUCCESS
                                                                  │    → PostDashboardVerificationSuccessHandler
                                                                  │    → Store-type order fulfilment
                                                                  │
                                                                  └─ POST_TRANSACTION_SUCCESS
                                                                       → PostTransactionSuccessHandler
                                                                       → Update Fiscalisation record
                                                                       → Receipt email with mark ID

  PostTransaction   ──► mishi.streams.post-transaction  ──────► (External consumer — unknown)
  ProducerAdapter

  TriggerPoslog     ──► flying_tiger.order.post_transaction ──► (External consumer — unknown)
  Adapter

  CashDeskEndShift  ──► flying_tiger.cash_desk.staff_shifts ──► (External consumer — unknown)
  EventAdapter           .end_shift_event

  (External: pay    ──► queuing.external.payments  ──────────► PaymentOnlyClientsAnalytics
   service)              .analytics.json                         Consumer
                                                                  │ group: PAYMENT_GROUP
                                                                  │ (separate credentials)
                                                                  ▼
                                                           ExternalPaymentsAnalyticsHandler
                                                             → Upsert AggregatedAnalytics
                                                             → Amounts: cents → decimal
```

### Topic Categories

**Bidirectional (Produce + Consume within monolith):**
- `mishi.streams.post-payment-success` — Payment completion triggers async processing
- `mishi.streams.common` — Multiplexed topic for refund, verification, and transaction events

**Produce-only (monolith → external):**
- `mishi.streams.post-transaction` — Transaction events (Flying Tiger integration)
- `flying_tiger.order.post_transaction` — Flying Tiger POSLOG generation
- `flying_tiger.cash_desk.staff_shifts.end_shift_event` — Cash desk shift closure

**Consume-only (external → monolith):**
- `mishi.streams.post-transaction-success` — Fiscalisation completion (from external fiscal authority)
- `queuing.external.payments.analytics.json` — Payment analytics (from `pay` microservice)

---

## End-to-End Flows

### 1. End-to-End Order Flow (Complete Lifecycle)

The order flow spans the entire customer journey from scanning the first item through post-payment fulfilment. This is the primary data flow in the platform.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 1: BASKET BUILDING                                                       │
│                                                                                 │
│  Customer scans item (barcode/RFID)                                             │
│      │                                                                          │
│      ▼                                                                          │
│  Mobile App → Kong Gateway → backend_v2                                         │
│      POST /items/v2/scan/ {barcode, store_id, basket_id}                        │
│      │                                                                          │
│      ├── StoreType dispatch → item_scan_functions.py                            │
│      │   └── Resolve barcode → ItemInventory (PostgreSQL)                       │
│      │       OR → Inventory Service (MongoDB/Elasticsearch)                     │
│      │                                                                          │
│      ├── Stock check (stock_quantity > 0, is_item_sold == False)                │
│      │                                                                          │
│      ├── Age restriction check → ManualVerificationSession (if restricted)      │
│      │                                                                          │
│      ├── Create/update BasketItem (quantity, price snapshot, denormalized data) │
│      │                                                                          │
│      ├── Promotion evaluation → ms-promo :8001 POST /evaluate/                  │
│      │   {basket_items[], store_id, customer_loyalty_info}                       │
│      │   └── Returns: per-item discounts, basket-level promotions               │
│      │                                                                          │
│      ├── Create BasketEntityAuditLog entries (modified prices, promo details)   │
│      │                                                                          │
│      └── Return: {item, price, promotions, basket_total}                        │
│                                                                                 │
│  [Repeat for each item scanned]                                                 │
│                                                                                 │
│  Basket state: BasketItem[] + BasketEntityAuditLog[] + Basket.ms_promo_response │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 2: ORDER CREATION                                                        │
│                                                                                 │
│  Customer taps "Checkout"                                                       │
│      │                                                                          │
│      ▼                                                                          │
│  Mobile App → POST /orders/v1/create-order/                                     │
│      {customer_id, store_id, basket_id, bundle_id, platform}                    │
│      │                                                                          │
│      ├── Validate basket (not expired, has items, address if required)           │
│      ├── Validate basket approval (if requires_approval, check verification)    │
│      │                                                                          │
│      ├── Create Order record:                                                   │
│      │   status=CREATED, payment_status=NOT_CAPTURED                            │
│      │   o_id = StoreTransactionSequenceNumber (SELECT FOR UPDATE locking)      │
│      │   basket = OneToOne link                                                 │
│      │                                                                          │
│      ├── Create Payments record:                                                │
│      │   status=WAITING, order=OneToOne, variant=store's PaymentVariant         │
│      │                                                                          │
│      └── Return: {order_id, o_id}                                               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 3: PAYMENT PROCESSING (see detailed payment flow below)                  │
│                                                                                 │
│  [Abbreviated — full detail in Payment Processing Flow section]                  │
│  Mobile App → create session → verify/confirm → PSP captures funds              │
│  Result: Payments.status = CAPTURED, captured_amount set                        │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 4: ORDER COMPLETION (Order.complete_status())                            │
│                                                                                 │
│  Triggered immediately after payment capture (synchronous):                     │
│      │                                                                          │
│      ├── 1. Calculate total_vat_price via tax client                            │
│      │      └── ms-tax (gRPC or HTTP) → tax_breakup[], total_tax               │
│      │                                                                          │
│      ├── 2. Set payment_status → CAPTURED_LIVE_MODE (or DEMO)                   │
│      │                                                                          │
│      ├── 3. Set order_status → COMPLETED (or VERIFIED if auto-verify)           │
│      │                                                                          │
│      ├── 4. Record completion_date                                              │
│      │                                                                          │
│      ├── 5. Increment cashier kiosk shift count (if kiosk platform)             │
│      │      └── KioskStaffShiftSummary.transaction_count += 1                   │
│      │                                                                          │
│      ├── 6. Mark basket as expired (is_expired=True)                            │
│      │                                                                          │
│      ├── 7. Assign transaction_id_poslog                                        │
│      │      └── StoreTransactionSequenceNumber (SELECT FOR UPDATE)              │
│      │      └── WHSmith: double-lock handling for concurrency                   │
│      │                                                                          │
│      ├── 8. [Flying Tiger] Build POSLOG → Kafka (TriggerPoslogAdapter)          │
│      │                                                                          │
│      ├── 9. [Saudi stores] Generate ZATCA QR code (TLV → Base64)               │
│      │                                                                          │
│      └── 10. [Carters] Generate carters_transaction_id                          │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PHASE 5: POST-PAYMENT PROCESSING (Order.complete_status_extra())               │
│                                                                                 │
│  Still synchronous, runs after complete_status():                               │
│      │                                                                          │
│      ├── 1. Firebase push notifications                                         │
│      │      ├── Dashboard topic (store staff)                                   │
│      │      └── Customer app (order confirmation)                               │
│      │                                                                          │
│      ├── 2. Order fulfilment dispatch (ORDER_FULFILMENT_FUNCTION_MAP)           │
│      │      └── 30+ retailer-specific functions:                                │
│      │          ├── POSLOG XML generation (WHSmith, Paradies, etc.)             │
│      │          ├── POS integration (Clover, SAP, Decathlon)                    │
│      │          ├── External API calls (Saturn SOAP, Leroy Merlin, etc.)        │
│      │          └── ERP notification (Dixons, Lego, Mango)                      │
│      │                                                                          │
│      ├── 3. RFID EPC status → SOLD (EPC_SOLD_FUNCTIONS_MAP)                    │
│      │      └── Vendor-specific: Nedap, SML, Keonn, Chainway, Sensormatic     │
│      │                                                                          │
│      ├── 4. Retailer customer profile update (RETAILER_PROFILE_FUNCTION_MAP)   │
│      │      └── Loyalty points, purchase history, tier updates                  │
│      │                                                                          │
│      ├── 5. Coupon redemption (redeem_coupons_helper())                         │
│      │                                                                          │
│      ├── 6. Receipt generation & delivery (send_receipt_v2())                   │
│      │      └── [See Receipt Generation Flow below]                             │
│      │                                                                          │
│      ├── 7. Basket audit log validation                                         │
│      │      └── Compare sum(BasketEntityAuditLog.modified_monetary_value)       │
│      │          vs Payments.captured_amount → Slack alert on mismatch           │
│      │                                                                          │
│      └── 8. [Click & Collect] Store email + subscription usage log              │
│                                                                                 │
│  THEN (asynchronous):                                                           │
│      │                                                                          │
│      └── PRODUCE to Kafka: mishi.streams.post-payment-success                   │
│          └── PostPaymentSuccessConsumer picks up → additional async tasks        │
│              ├── Analytics event recording                                       │
│              ├── Fiscalisation submission (Italy/Greece)                         │
│              └── Additional retailer-specific async fulfilment                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Data created across the order lifecycle:**

| Phase | Records Created/Updated | Storage |
|-------|------------------------|---------|
| Basket building | `BasketItem`, `BasketEntityAuditLog`, `Basket.ms_promo_response` | PostgreSQL |
| Order creation | `Order`, `Payments` | PostgreSQL |
| Payment | `Payments` (status, captured_amount, PSP refs) | PostgreSQL + MongoDB (pay svc) |
| Completion | `Order` (status, poslog ID, completion_date, extra_data) | PostgreSQL |
| Post-payment | POSLOG XML, RFID status, receipts, analytics events, Kafka messages | PostgreSQL + Redis + Kafka + external |

---

### 2. Payment Processing Flow (Detailed)

The platform supports two payment architectures. All stores are migrating toward MS Pay, but both paths coexist.

#### 2a. MS Pay Flow (Modern — 35+ Gateways)

```
Mobile App              Kong        backend_v2           pay-gateway    pay (MongoDB)    PSP
    │                    │              │                    │              │              │
    │  POST /payments/   │              │                    │              │              │
    │  v1/ms-pay/        │              │                    │              │              │
    │  create-session/   │              │                    │              │              │
    │───────────────────►│─────────────►│                    │              │              │
    │                    │              │                    │              │              │
    │                    │              │  Resolve PaymentVariant for store │              │
    │                    │              │  (psp, merchant_id, keys, etc.)  │              │
    │                    │              │                    │              │              │
    │                    │              │  ms_pay.py:        │              │              │
    │                    │              │  create_payment_   │              │              │
    │                    │              │  session()         │              │              │
    │                    │              │──────────────────►│──────────────►│              │
    │                    │              │                    │              │  HTTP to PSP │
    │                    │              │                    │              │─────────────►│
    │                    │              │                    │              │              │
    │                    │              │                    │              │  Session     │
    │                    │              │◄──────────────────│◄──────────────│◄─────────────│
    │                    │              │                    │              │              │
    │  {session_id,      │              │                    │              │              │
    │   client_secret,   │              │                    │              │              │
    │   payment_methods} │              │                    │              │              │
    │◄───────────────────│◄─────────────│                    │              │              │
    │                    │              │                    │              │              │
    │  [Customer enters  │              │                    │              │              │
    │   card / selects   │              │                    │              │              │
    │   payment method]  │              │                    │              │              │
    │                    │              │                    │              │              │
    │  POST /payments/   │              │                    │              │              │
    │  v1/ms-pay/        │              │                    │              │              │
    │  add-payment/      │              │                    │              │              │
    │───────────────────►│─────────────►│                    │              │              │
    │                    │              │  ms_pay.py:        │              │              │
    │                    │              │  add_payment()     │              │              │
    │                    │              │──────────────────►│──────────────►│──────────────►│
    │                    │              │                    │              │              │
    │                    │              │  ┌── 3DS Required? │              │              │
    │                    │              │  │   YES: return   │              │              │
    │                    │              │  │   redirect URL  │              │              │
    │                    │              │  │   to client     │              │              │
    │                    │              │  │   NO: continue  │              │              │
    │                    │              │  └────────────────│              │              │
    │                    │              │                    │              │              │
    │                    │              │  Payment confirmed │              │              │
    │                    │              │◄──────────────────│◄──────────────│◄─────────────│
    │                    │              │                    │              │              │
    │                    │              │  ┌─────────────────────────────┐  │              │
    │                    │              │  │ Order.complete_status()     │  │              │
    │                    │              │  │ Payments.captured_amount =  │  │              │
    │                    │              │  │   basket total              │  │              │
    │                    │              │  │ Payments.status = CAPTURED  │  │              │
    │                    │              │  │ → [Full order completion    │  │              │
    │                    │              │  │    pipeline from Phase 4-5] │  │              │
    │                    │              │  └─────────────────────────────┘  │              │
    │                    │              │                    │              │              │
    │  {order_complete,  │              │                    │              │              │
    │   receipt_url}     │              │                    │              │              │
    │◄───────────────────│◄─────────────│                    │              │              │
```

#### 2b. Legacy PSP Flow (Direct Integration — 22 PSPs)

```
Mobile App              Kong        backend_v2                              PSP
    │                    │              │                                     │
    │  POST /payments/   │              │                                     │
    │  v1/session/       │              │                                     │
    │───────────────────►│─────────────►│                                     │
    │                    │              │                                     │
    │                    │              │  PaymentVariant.payment_class       │
    │                    │              │  → provider_factory()               │
    │                    │              │  → e.g., StripeConnect provider     │
    │                    │              │                                     │
    │                    │              │  Payments.payment_session()         │
    │                    │              │  → provider.create_session()        │
    │                    │              │  → direct PSP API call             │
    │                    │              │────────────────────────────────────►│
    │                    │              │                                     │
    │  {session_data}    │◄─────────────│◄────────────────────────────────────│
    │◄───────────────────│              │                                     │
    │                    │              │                                     │
    │  POST /payments/   │              │                                     │
    │  v1/verify/        │              │                                     │
    │───────────────────►│─────────────►│                                     │
    │                    │              │                                     │
    │                    │              │  Payments.verify_payment()          │
    │                    │              │  → provider.verify()               │
    │                    │              │────────────────────────────────────►│
    │                    │              │                                     │
    │                    │              │  ┌── 3DS flow? ──────────────┐      │
    │                    │              │  │  YES: return redirect,    │      │
    │                    │              │  │  wait for confirm call    │      │
    │                    │              │  │  NO: auto-capture below   │      │
    │                    │              │  └───────────────────────────┘      │
    │                    │              │                                     │
    │                    │              │  ┌── Preauth PSP? ──────────┐      │
    │                    │              │  │  YES: provider.capture()  │      │
    │                    │              │  │  NO: already captured     │      │
    │                    │              │  └───────────────────────────┘      │
    │                    │              │                                     │
    │                    │              │  → Order.complete_status()          │
    │  {order_complete}  │◄─────────────│                                     │
    │◄───────────────────│              │                                     │
```

#### 2c. Special Payment Flows

**Bypass Payment (Zero-Amount Orders):**
```
100% discount basket → Payments.by_pass_payment()
  → No PSP call
  → Payments.status = CONFIRMED
  → Order.complete_status() runs normally
```

**Saved Card Flow:**
```
Customer selects saved card → POST /payments/v1/ms-pay/add-payment/
  → Include saved_card_id in request
  → MS Pay uses stored token → PSP charges card
  → May trigger 3DS if card requires it
```

**Terminal Payment (Kiosk/POS):**
```
Kiosk device → POST /payments/v1/ms-pay/add-payment/
  → payment_method: "card_terminal" / "adyen_pos"
  → MS Pay sends to physical terminal
  → Terminal prompts customer → card tap/insert
  → Terminal confirms → MS Pay → backend_v2
```

**Preauth → Capture Flow:**
```
Phase 1: Authorization
  POST /payments/v1/session/ → provider.preauth_for_amount()
  → PSP authorizes (holds) the amount
  → Payments.status = PREAUTH

Phase 2: Capture (after order actions)
  provider.capture_amount() → PSP captures authorized amount
  → Payments.status = CAPTURED
  → Order.complete_status()
```

**Payment status transitions:**
```
WAITING → (session created)
  │
  ├── → PREAUTH → CAPTURED → [REFUNDED]
  │                             (partial or full)
  │
  ├── → CONFIRMED → (bypass/zero-amount)
  │
  ├── → REJECTED (PSP declined)
  │
  └── → ERROR (PSP/network failure)
```

---

### 3. Promotion Evaluation Flow (Detailed)

Promotions are evaluated on every basket modification (item add/remove/quantity change). The flow involves the monolith, the ms-promo microservice, and optionally the subscription system.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TRIGGER: Item scanned / quantity changed / coupon applied                   │
│                                                                             │
│  backend_v2 (basket_and_wishlist.py / mpay_promo.py)                        │
│      │                                                                      │
│      ├── 1. Build evaluation request:                                       │
│      │      {                                                               │
│      │        store_id,                                                     │
│      │        items: [{sku, mrp (retail price), sp (selling price),         │
│      │                 qty, category_type, category_value}],                │
│      │        special_promos: {loyalty_tier, subscription_qty, ...},        │
│      │        promo_override: [{on-the-fly promo definitions}],             │
│      │        coupon_codes: [applied coupon codes]                           │
│      │      }                                                               │
│      │                                                                      │
│      ├── 2. Check subscription eligibility (if applicable):                 │
│      │      subscription_helper.get_customer_subscription_applicable_info() │
│      │      └── CustomerSubscriptionMapping → check daily usage log         │
│      │      └── Returns: remaining discount_item_qty for eligible items     │
│      │      └── Feeds into special_promos in evaluation request             │
│      │                                                                      │
│      └── 3. HTTP POST to ms-promo :8001 /evaluate/                          │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ms-promo (service.promotions) — Evaluation Engine                          │
│                                                                             │
│  PromoLogic.evaluate() / PromoLogicV2.evaluate():                           │
│      │                                                                      │
│      ├── Step 1: Parse basket items                                         │
│      │   └── Build sku_to_basket_items_dict                                 │
│      │   └── Build category_type_value_to_sku_list_dict                     │
│      │                                                                      │
│      ├── Step 2: Fetch active promotions (raw SQL)                          │
│      │   └── JOIN: promo_header + promo_header_store + promo_group          │
│      │             + promo_group_node                                        │
│      │   └── Filter: store_id, is_active, date range, matching SKUs/cats    │
│      │   └── Build PromoDataManager → hierarchical Promo objects            │
│      │                                                                      │
│      ├── Step 3: Sort promotions into layers (priority groups)              │
│      │                                                                      │
│      ├── Step 4: For each layer:                                            │
│      │   ├── Prune invalid promos (quantity thresholds, excluded items)      │
│      │   ├── Cluster by evaluation criteria                                 │
│      │   │                                                                  │
│      │   ├── Apply best-discount promos:                                    │
│      │   │   ├── Exhaustive algorithm (small sets):                         │
│      │   │   │   enumerate all combinations → pick lowest total             │
│      │   │   └── Marginal-value algorithm (large sets):                     │
│      │   │       rank by per-unit discount → greedy assignment              │
│      │   │                                                                  │
│      │   ├── Apply priority-based promos (deterministic order)              │
│      │   │                                                                  │
│      │   └── [V2] Apply basket-level best-discount + priority promos        │
│      │       └── BasketThreshold: spend-X-get-Y across entire basket        │
│      │                                                                      │
│      └── Step 5: Build response                                             │
│          └── Per-item: {sku, discount, applied_promos[]}                    │
│          └── Basket-level: {threshold_discounts[]}                          │
│                                                                             │
│  8 Promotion Family Types:                                                  │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ Family                          │ Discount Logic                 │       │
│  ├─────────────────────────────────┼────────────────────────────────┤       │
│  │ Easy                            │ Simple % or $ off per item     │       │
│  │ Easy+                           │ Buy-X-get-Y-% off             │       │
│  │ Combo                           │ Bundle price for item set      │       │
│  │ Line Special                    │ Multi-buy (3 for £10)          │       │
│  │ Basket Threshold                │ Spend-X-get-basket-$-off       │       │
│  │ Basket Threshold + Disc. Target │ Spend-X-get-item-Y-discounted  │       │
│  │ Requisite Groups + Disc. Target │ Buy-from-A-get-B-discounted    │       │
│  │ Evenly Distributed Multiline    │ Spread discount across items   │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  backend_v2 — Apply promotion results to basket                             │
│      │                                                                      │
│      ├── Store raw response in Basket.ms_promo_response                     │
│      │                                                                      │
│      ├── For each item with a discount:                                     │
│      │   └── Create/update BasketEntityAuditLog:                            │
│      │       ├── modified_monetary_value = price after discount             │
│      │       ├── promo_titles = promotion names applied                     │
│      │       ├── discount_amount = savings amount                           │
│      │       └── promo_details = full promo metadata                        │
│      │                                                                      │
│      ├── Update Basket:                                                     │
│      │   ├── promotion_applied = True                                       │
│      │   ├── applied_offer = selected promotion                             │
│      │   ├── price_after_discount = new total                               │
│      │   ├── total_basket_amount_after_discount = cached total              │
│      │   └── last_promotion_applied_date = now                              │
│      │                                                                      │
│      └── Return updated basket to client                                    │
│          {items[], promotions[], total, savings}                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Data flow summary:**

```
BasketItem     →  ms-promo evaluate  →  BasketEntityAuditLog
(raw prices)      (discount engine)     (discounted prices, promo details)
                                              │
                                              ▼
                                        Basket model
                                        (cached totals, applied_offer)
                                              │
                                              ▼
                                        Order creation
                                        (uses audit log totals for payment amount)
```

---

### 4. Refund Processing Flow (Detailed)

```
Dashboard UI            backend_v2                  PSP              Kafka
    │                       │                        │                 │
    │  POST /dashboard/     │                        │                 │
    │  refund-order/        │                        │                 │
    │  {order_id,           │                        │                 │
    │   items_to_refund[],  │                        │                 │
    │   refund_reason}      │                        │                 │
    │──────────────────────►│                        │                 │
    │                       │                        │                 │
    │                       │  1. Create RefundOrderNew record        │
    │                       │     (links to Order, stores refund      │
    │                       │      items, amounts, reason)            │
    │                       │                        │                 │
    │                       │  2. Calculate refund amount             │
    │                       │     sum(item.price * refund_qty)        │
    │                       │     + applicable promo adjustments      │
    │                       │                        │                 │
    │                       │  3. Validate:                           │
    │                       │     refund_amount <=                    │
    │                       │     captured_amount -                   │
    │                       │     refunded_amount -                   │
    │                       │     refund_processing_amount            │
    │                       │                        │                 │
    │                       │  4. PSP refund call:   │                 │
    │                       │     ├── MS Pay: HTTP   │                 │
    │                       │     │   to pay service │                 │
    │                       │     │   /refund/       │                 │
    │                       │     └── Legacy: direct │                 │
    │                       │         provider call  │                 │
    │                       │───────────────────────►│                 │
    │                       │                        │                 │
    │                       │  ┌── HTTP 200: ────────│─────────┐      │
    │                       │  │ Instant refund      │         │      │
    │                       │  │ refunded_amount +=  │         │      │
    │                       │  │ Payments.status →   │         │      │
    │                       │  │   REFUNDED (if full)│         │      │
    │                       │  │                     │         │      │
    │                       │  ├── HTTP 202: ────────│─────────┤      │
    │                       │  │ Async refund        │         │      │
    │                       │  │ refund_processing_  │         │      │
    │                       │  │ amount += amount    │         │      │
    │                       │  └─────────────────────│─────────┘      │
    │                       │                        │                 │
    │                       │  5. Update kiosk shift │(if applicable) │
    │                       │     └── increment refund count          │
    │                       │                        │                 │
    │                       │  6. PRODUCE to Kafka:  │                 │
    │                       │     mishi.streams.common                 │
    │                       │     key: POST_REFUND_SUCCESS            │
    │                       │─────────────────────────────────────────►│
    │  {refund_confirmed}   │                        │                 │
    │◄──────────────────────│                        │                 │
    │                       │                        │                 │
    │                       │           CONSUME: common               │
    │                       │           PostRefundSuccessHandler       │
    │                       │  ┌────────────────────────────────◄─────│
    │                       │  │                     │                 │
    │                       │  │  → Store-type refund fulfilment      │
    │                       │  │    (reverse POSLOG, POS update)      │
    │                       │  │                     │                 │
    │                       │  │  → Receipt email (refund variant)    │
    │                       │  │    StrategyFactory → refund strategy  │
    │                       │  │    → email/print receipt              │
    │                       │  │                     │                 │
    │                       │  │  → RFID EPC → UNSOLD (if RFID store) │
    │                       │  └─────────────────────│                 │
```

---

### 5. Receipt Generation Flow

Receipts are generated synchronously during `complete_status_extra()` for orders, and asynchronously via Kafka for refunds. Both email and print (Epson POS) receipts use the same Strategy pattern.

```
Order/Refund completion
    │
    ▼
send_receipt_v2() / PostRefundSuccessHandler
    │
    ├── 1. Determine receipt type: ORDER, REFUND, GIFT, STEB, etc.
    │
    ├── 2. Resolve PrintReceiptConfiguration:
    │      Hierarchical merge (broadest → most specific):
    │      ALL_ALL → ALL_{region} → {store_type}_ALL → {store_type}_{region}
    │
    ├── 3. StrategyFactory.create_strategy(store, region, receipt_type)
    │      └── STRATEGY_MAPPING[(receipt_type, store_type)]
    │      └── Falls back to base PrintReceipt class
    │
    ├── 4. PrintReceipt.get_receipt_and_context():
    │      ├── build_context(config_fields)
    │      │   ├── default_context() → 60+ variables from order/store/payment
    │      │   └── ContextUpdater → retailer strategy.update(context)
    │      │       └── e.g., DDF: boarding pass, DCC, STEB barcodes
    │      │       └── e.g., MLSE: Bucks, budget codes, split payments
    │      │       └── e.g., Carters: loyalty points, custom barcodes
    │      │
    │      ├── update_context_with_receipt_text()
    │      │   ├── format item lines (name, qty, price)
    │      │   ├── format basket-level promotions
    │      │   ├── format VAT + order total
    │      │   └── format payment details
    │      │
    │      └── get_receipt_data() → render Django template with context
    │
    ├── 5. Email receipt:
    │      ├── EmailHelper.get_store_region_config()
    │      │   (first-match: store+region → store+ALL → ALL+region → ALL+ALL)
    │      ├── EmailHelper.get_email_data() → render template
    │      └── Send email (external email service)
    │
    └── 6. [Kiosk] Print receipt:
           └── GeneratePosReceipt → Epson POS markup
           └── Send to printer hardware
```

**Additional receipts:** Some receipt types trigger additional receipts. For example:
- Dubai Duty Free ORDER generates STEB receipts (one per distinct bag barcode)
- Dubai Duty Free PAYMENT_FAILED generates a CAMPAIGN receipt for Visa cardholders
- Carters/Seventh Heaven ORDER generates a GIFT receipt alongside the main receipt

---

### 6. Settlement & Reconciliation Flow

Settlement runs as scheduled cron jobs (Django management commands), not real-time. The pipeline has two stages.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  STAGE 1: Collect settlement data from PSPs (daily cron)                    │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ Modern path: ms_pay_based_settlement_payout.py                   │       │
│  │   └── For each store (batched in chunks of 30):                  │       │
│  │       └── HTTP GET pay-service /api/v1/payout/get-payments-      │       │
│  │           refunds/ {store_ids[], date_range}                     │       │
│  │       └── For each transaction:                                  │       │
│  │           ├── Match to MishiPay Order by psp_reference_id        │       │
│  │           ├── Create SettlementOrder record:                     │       │
│  │           │   {order_id, psp_charged_amount, psp_fee_amount,     │       │
│  │           │    psp_net_amount, card_brand, auth_code, etc.}      │       │
│  │           └── Create SettlementOrderRefund (for refund txns)     │       │
│  │   └── Supports 18+ PSP variants (mapped to internal types)      │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ Legacy path: stripe_connect_psp_settlement_payout.py             │       │
│  │   └── Direct Stripe API: Charges → Transfers → Balance Txns     │       │
│  │   └── Amounts in cents (÷ 100 for decimal)                      │       │
│  │   └── Weekend payout adjustment (Sat→Mon, Sun→Tue)              │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ File-based path: quickpay_psp_settlement_by_files.py             │       │
│  │   └── Flying Tiger only:                                         │       │
│  │       ├── MobilePay (DK): TellerDK CSV (Latin-1 encoding)       │       │
│  │       ├── Swish (SE): ISO 20022 camt.053 XML                    │       │
│  │       └── Vipps (NO): Vipps SettlementReport-3.0 XML            │       │
│  │   └── Reads from /app/reconciliation/flying_tiger/{region}/live/ │       │
│  └──────────────────────────────────────────────────────────────────┘       │
│                                                                             │
│  All paths: delete-then-create within transaction.atomic() (idempotent)     │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STAGE 2: Aggregate and report (daily cron, runs after Stage 1)             │
│                                                                             │
│  mishipay_settlement_report_final.py:                                       │
│      │                                                                      │
│      ├── fill_missing_settlement_orders()                                   │
│      │   └── Find Orders with no SettlementOrder record                     │
│      │   └── Create FAILED status records (PSP prefix heuristic:            │
│      │       ch_/pi_ → Stripe, pay → Checkout, else → QuickPay)            │
│      │                                                                      │
│      ├── update_poslog_status_of_records()                                  │
│      │   └── Re-check order.extra_data['poslog_sent'] flag                  │
│      │                                                                      │
│      ├── generate_settlement_poslog_recon_report()                          │
│      │   └── Three-way reconciliation:                                      │
│      │       MishiPay amounts vs POSLOG amounts vs PSP amounts              │
│      │   └── Django ORM aggregation → SettlementReconByDate                 │
│      │   └── → SettlementFinalSummary (per business_date)                   │
│      │                                                                      │
│      └── generate_settlement_payout_recon_report()                          │
│          └── Aggregation by payout date → SettlementFinalSummaryByPayoutDates│
│                                                                             │
│  create_psp_reconciliation_summary.py:                                      │
│      └── Generate CSV reports for retailers                                  │
│      └── Retailer-specific columns (DDF: SCO ID; WHSmith: Cashier Code)     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Settlement data model relationships:

  SettlementOrder ──────────────────► SettlementReconByDate
  (per-transaction)                    (per-store, per-date aggregate)
         │                                      │
         │                                      ▼
  SettlementOrderRefund              SettlementFinalSummary
  (per-refund transaction)           (3-way recon: MishiPay vs POSLOG vs PSP)
         │                                      │
         ▼                                      ▼
  SettlementPayoutByDate             SettlementFinalSummaryByPayoutDates
  (PSP payout records)               (same, grouped by payout date)
```

---

### 7. Payment Alert Flow

Two management commands detect payment discrepancies between MishiPay orders and PSP records.

```
Scheduled cron (daily)
    │
    ├── stripe_connect_payment_alert.py:
    │   └── For each Stripe-enabled store:
    │       └── stripe.PaymentIntent.list(date_range)
    │       └── For each PaymentIntent:
    │           ├── Find matching MishiPay Order by psp_reference_id
    │           ├── Compare order status vs PaymentIntent status
    │           └── Discrepancy? → Create PaymentAlert record
    │               │
    │               ├── Duplicate charges → AUTO-REFUND via stripe.Refund.create()
    │               │   └── Alert status → RESOLVED_BY_ALERT
    │               │
    │               ├── PSP success, order failed → AUTO-UPDATE order status
    │               │   └── update_order_payment_info_manual()
    │               │   └── Alert status → RESOLVED_BY_ALERT
    │               │
    │               └── Other → Alert status → CREATED (manual investigation)
    │
    ├── quickpay_payment_alert.py:
    │   └── Same pattern for MobilePay/Vipps/Swish
    │   └── Additional type: PSP_AUTHORIZED_BUT_NOT_CAPTURED
    │
    └── Send Slack notification:
        ├── #alerts_payments (prod, high priority)
        ├── #alerts_payments_low (low priority)
        └── #alerts_test (non-prod)
```

---

### 8. Inventory & Item Scan Flow

Items can come from two sources: the monolith's PostgreSQL database or the Inventory Service (MongoDB + Elasticsearch).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DATA IMPORT (Batch — management commands, scheduled cron)                  │
│                                                                             │
│  Source files (CSV/XML) → 35 management commands, 25+ retailers             │
│      │                                                                      │
│      ├── Path A: Direct DB import                                           │
│      │   └── Parse CSV → create/update ItemInventory (PostgreSQL)           │
│      │   └── Update stock_quantity, price, promotional_offer_codes          │
│      │   └── source_hash for change detection (skip unchanged rows)         │
│      │                                                                      │
│      ├── Path B: Inventory Service import                                   │
│      │   └── Parse CSV → HTTP POST inventory-service /api/v1/items/batch/  │
│      │   └── Inventory Service stores in MongoDB + indexes in Elasticsearch │
│      │                                                                      │
│      └── Path C: DDF Price Activation (RPM file ingestion)                  │
│          └── Parse RPM files → PriceActivation records (PostgreSQL)         │
│          └── Staff-mediated activation flow before prices go live            │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  ITEM SCAN (Real-time — customer scans barcode)                             │
│                                                                             │
│  Mobile App → POST /items/v2/scan/ {barcode, store_id}                      │
│      │                                                                      │
│      ├── StoreType dispatch → item_scan_functions.py                        │
│      │                                                                      │
│      ├── Path A: Monolith DB lookup                                         │
│      │   └── ItemInventory.objects.filter(                                  │
│      │         product_identifier=barcode,                                  │
│      │         item_category__store=store,                                  │
│      │         soft_delete=False)                                           │
│      │   └── OR: AdditionalBarcode.objects.filter(barcode=barcode)          │
│      │                                                                      │
│      ├── Path B: Inventory Service lookup                                   │
│      │   └── HTTP GET inventory-service /api/v1/items/{barcode}/            │
│      │   └── Elasticsearch search for barcode → MongoDB document            │
│      │                                                                      │
│      ├── Barcode normalization:                                              │
│      │   ├── Reduce to Clear (RtC): strip RtC prefix → original barcode    │
│      │   ├── GS1 DataBar expansion: parse AI codes → extract GTIN          │
│      │   ├── EAN-13/UPC-A/UPC-E conversion                                 │
│      │   └── Leading zero handling (some retailers strip leading zeros)      │
│      │                                                                      │
│      └── Result: ItemInventory data + price + stock + restriction flags     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 9. Customer Lifecycle Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  AUTHENTICATION                                                              │
│                                                                             │
│  ┌── Returning Customer ──────────────────────────────────────────┐         │
│  │   Mobile App → Kong → Keycloak :8005                           │         │
│  │   POST /auth/realms/{realm}/protocol/openid-connect/token      │         │
│  │   {username, password, grant_type=password}                    │         │
│  │   → JWT access_token + refresh_token                           │         │
│  │   → All subsequent API calls include Authorization: Bearer JWT │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                                                             │
│  ┌── New Customer ────────────────────────────────────────────────┐         │
│  │   Mobile App → POST /customers/v1/register/                    │         │
│  │   {email, phone, first_name, last_name, store_type}            │         │
│  │   → Create Customer record (PostgreSQL)                        │         │
│  │   → Create Keycloak user (HTTP to Keycloak admin API)          │         │
│  │   → Return JWT tokens                                          │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                                                             │
│  ┌── Guest Checkout ──────────────────────────────────────────────┐         │
│  │   Mobile App → POST /customers/v1/guest-checkout/              │         │
│  │   {email, store_id}                                            │         │
│  │   → Find or create Customer with guest flag                    │         │
│  │   → Temporary Keycloak user → JWT                              │         │
│  └────────────────────────────────────────────────────────────────┘         │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  CUSTOMER PROFILE & LOYALTY                                                  │
│                                                                             │
│  Per order completion (RETAILER_PROFILE_FUNCTION_MAPPING):                  │
│      │                                                                      │
│      ├── Create/update RetailerSpecificCustomerProfile                      │
│      │   └── Loyalty ID, points, tier, boarding pass (DDF)                  │
│      │                                                                      │
│      ├── 15+ retailer loyalty integrations:                                 │
│      │   ├── Carters → CartersPostTransaction API                           │
│      │   ├── MLSE → Bucks/budget code tracking                             │
│      │   ├── DDF → Boarding pass + quota tracking                           │
│      │   ├── Grandiose → Loyalty fulfilment                                │
│      │   └── Others: WHSmith, Lego, Decathlon, etc.                         │
│      │                                                                      │
│      └── Privacy/consent management:                                         │
│          ├── CustomerPrivacyConsent (marketing, terms, data sharing)         │
│          └── Per-store_type consent records with version tracking            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 10. Click & Collect Flow

```
Customer                backend_v2                   Store Staff
    │                       │                            │
    │  Create basket +      │                            │
    │  select C&C purchase  │                            │
    │  type                 │                            │
    │──────────────────────►│                            │
    │                       │                            │
    │  Create Order (normal │                            │
    │  payment flow)        │                            │
    │──────────────────────►│                            │
    │                       │                            │
    │                       │  Order.complete_status():  │
    │                       │  cc_order_status = CREATED │
    │                       │  token_number = generated  │
    │                       │                            │
    │                       │  Email notification ──────►│
    │                       │  (store staff)             │
    │                       │                            │
    │                       │  ◄── PATCH cc_status ──────│
    │                       │      = ACCEPTED            │
    │  Push notification    │                            │
    │  "Order accepted"     │                            │
    │◄──────────────────────│                            │
    │                       │                            │
    │                       │  ◄── PATCH cc_status ──────│
    │                       │      = READY_TO_COLLECT    │
    │  Push notification    │                            │
    │  "Ready for pickup"   │                            │
    │◄──────────────────────│                            │
    │                       │                            │
    │  Customer arrives,    │                            │
    │  shows token_number   │                            │
    │                       │  ◄── PATCH cc_status ──────│
    │                       │      = COLLECTED           │
    │                       │                            │

CC Status Transitions:
  CREATED → ACCEPTED → READY_TO_COLLECT → COLLECTED
       │         │              │
       ▼         ▼              ▼
  REJECTED   REJECTED      NOT_COLLECTED
  _STORE     _CUSTOMER
```

---

### 11. Offline Transaction Sync Flow

For stores with unreliable connectivity, the kiosk/app caches transactions locally and syncs when online.

```
Offline Kiosk                              backend_v2 (when online)
    │                                          │
    │  Customer scans + pays                   │
    │  (stored locally as JSON)                │
    │                                          │
    │  POST /orders/v1/offline-sync/           │
    │  {                                       │
    │    customer: {email, phone, ...},        │
    │    basket: {items[], store_id},           │
    │    order: {o_id, completion_date, ...},   │
    │    payment: {amount, method, ref, ...}    │
    │  }                                       │
    │─────────────────────────────────────────►│
    │                                          │
    │                                          │  1. Find or create Customer
    │                                          │  2. Create Basket + BasketItems
    │                                          │  3. Create Order (status=COMPLETED)
    │                                          │     └── offline_receipt_reference_id set
    │                                          │     └── offline_o_id set
    │                                          │  4. Create Payments (status=CAPTURED)
    │                                          │  5. Run promotion evaluation
    │                                          │  6. Create BasketEntityAuditLogs
    │                                          │  7. Run complete_status_extra()
    │                                          │     └── fulfilment, receipts, etc.
    │                                          │
    │  {order_id, sync_status: "success"}      │
    │◄─────────────────────────────────────────│
```

---

### 12. RFID Tracking Flow

For RFID-enabled stores, item scanning uses EPC (Electronic Product Code) instead of barcodes, and the platform tracks EPC sold/unsold status across 7 vendor integrations.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  SCAN: EPC-based item identification                                        │
│                                                                             │
│  RFID Reader → Mobile App → POST /items/v2/scan/                            │
│  {epc, store_id}                                                            │
│      │                                                                      │
│      ├── Lookup ItemInventory by epc field                                  │
│      ├── Check epc_status (must not be SOLD)                                │
│      └── Proceed with normal basket flow                                    │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  SOLD: After payment (in complete_status_extra())                           │
│                                                                             │
│  EPC_SOLD_FUNCTIONS_MAP[store_type](order)                                  │
│      │                                                                      │
│      ├── Nedap → HTTP PUT /article-eas-status (batch)                       │
│      ├── SML → HTTP POST /SmartEAS/EpcMgmt/SetEpcTagStatus                 │
│      ├── Keonn → Redis SADD sold-epc-set + HTTP POST /api/gate/            │
│      ├── Chainway → HTTP POST vendor API                                    │
│      ├── Sensormatic → HTTP POST vendor API                                 │
│      ├── Sievert → HTTP POST vendor API                                     │
│      └── Tego → HTTP POST vendor API                                        │
│                                                                             │
│  Updates: ItemInventory.epc_status → SOLD                                   │
│  Redis: Stores sold EPC set for gate controller queries                     │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  UNSOLD: After refund (PostRefundSuccessHandler)                            │
│                                                                             │
│  Reverse the SOLD operation → vendor API call to mark UNSOLD                │
│  Updates: ItemInventory.epc_status → NOT_SOLD                               │
│  Redis: Remove from sold EPC set                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

Gate Security:
  Keonn RFID gate → queries Redis sold-epc-set
  → EPC in set? → allow exit (paid item)
  → EPC not in set? → trigger alarm (unpaid item)
```

---

### 13. Subscription Flow

```
Management Command              ms-promo              Stripe (via MS Pay)
    │                              │                       │
    │  create_subscription_promo   │                       │
    │  → Create 100% discount      │                       │
    │    promotions for eligible    │                       │
    │    items at mapped stores    │                       │
    │─────────────────────────────►│                       │
    │                              │                       │
    │                              │                       │

Customer                    backend_v2              MS Pay            Stripe
    │                          │                      │                 │
    │  POST /subscriptions/    │                      │                 │
    │  v1/stripe/create-       │                      │                 │
    │  session/                │                      │                 │
    │─────────────────────────►│                      │                 │
    │                          │  HTTP POST           │                 │
    │                          │──────────────────────►│                 │
    │                          │                      │  Checkout       │
    │                          │                      │  Session        │
    │                          │                      │─────────────────►│
    │  {checkout_url}          │◄─────────────────────│◄─────────────────│
    │◄─────────────────────────│                      │                 │
    │                          │                      │                 │
    │  [Customer completes     │                      │                 │
    │   Stripe payment]        │                      │                 │
    │                          │                      │                 │
    │                          │  Webhook (HMAC)      │  Stripe event  │
    │                          │◄─────────────────────│◄─────────────────│
    │                          │                      │                 │
    │                          │  Create/update CustomerSubscriptionMapping │
    │                          │  status: TRIAL or ACTIVE                   │
    │                          │  expiry_on: calculated from Stripe period  │
    │                          │                      │                 │

Per-Order Usage:
    │  [Customer shops]        │                      │                 │
    │  Basket evaluation       │                      │                 │
    │─────────────────────────►│                      │                 │
    │                          │  subscription_helper.                  │
    │                          │  get_customer_subscription_            │
    │                          │  applicable_info_for_store()           │
    │                          │  ├── Check CustomerSubscriptionMapping │
    │                          │  ├── Check daily usage vs quantity_max │
    │                          │  └── Return remaining qty to promo    │
    │                          │                                        │
    │                          │  ms-promo evaluate:                    │
    │                          │  100% discount on eligible items       │
    │                          │  (limited to remaining daily qty)      │
    │                          │                                        │
    │  After order:            │                                        │
    │                          │  log_subscription_order_items()        │
    │                          │  → CustomerSubscriptionUsageLog        │
```

---

### 14. Fiscalisation Flow (Italy/Greece)

For stores in Italy (Venistar) and Greece (Funky Buddha), fiscal compliance requires registering transactions with national fiscal authorities.

```
consumer_post_payment    backend_v2              External Fiscal    consumer_common
_success                    │                   Authority              │
    │                       │                       │                  │
    │  PostPaymentSuccess   │                       │                  │
    │  Handler triggers     │                       │                  │
    │  fiscalisation        │                       │                  │
    │──────────────────────►│                       │                  │
    │                       │                       │                  │
    │                       │  Create Fiscalisation │                  │
    │                       │  record (status=PENDING)                 │
    │                       │                       │                  │
    │                       │  PRODUCE:             │                  │
    │                       │  mishi.streams.       │                  │
    │                       │  post-transaction     │                  │
    │                       │──────────────────────►│                  │
    │                       │                       │  Register with  │
    │                       │                       │  fiscal authority│
    │                       │                       │  (SOAP for      │
    │                       │                       │  Venistar/Italy) │
    │                       │                       │                  │
    │                       │  PRODUCE: post-transaction-success      │
    │                       │                       │─────────────────►│
    │                       │                       │                  │
    │                       │                 PostTransactionSuccessHandler
    │                       │                       │  → Update Fiscal-│
    │                       │                       │    isation record│
    │                       │                       │    status=SUCCESS│
    │                       │                       │    mark_id = fiscal ID
    │                       │                       │  → Send receipt  │
    │                       │                       │    email with    │
    │                       │                       │    fiscal mark ID│
```

---

### 15. Tax Evaluation Flow

```
backend_v2                  ms-tax (Legacy)       ms_tax (New)
    │                       gRPC :9000            HTTP :80
    │                           │                     │
    │  Tax client selector      │                     │
    │  (per-store config:       │                     │
    │   tax_calculator_type)    │                     │
    │                           │                     │
    │  ┌── Legacy store ───────►│                     │
    │  │   gRPC EvaluateTax()   │                     │
    │  │   {store_data, items}  │                     │
    │  │                        │                     │
    │  │   {total_tax,          │                     │
    │  │    tax_breakup[],      │                     │
    │  │    item_tax[]}         │                     │
    │  │◄───────────────────────│                     │
    │  │                        │                     │
    │  └── New store ──────────────────────────────►  │
    │      HTTP POST /evaluate  │                     │
    │      {store_data, items}  │                     │
    │                           │                     │
    │      {total_tax, ...}     │                     │
    │  ◄───────────────────────────────────────────── │
```

Tax evaluation is called during `Order.complete_status()` to calculate `total_vat_price`. Both services implement the same contract: evaluate tax for basket items given store location and tax codes. The result is stored in `Basket.total_vat_price`, `Basket.tax_level_breakup`, and `Basket.total_taxable_amount`.

---

### 16. External Payments Analytics Flow

```
pay service              Kafka                 consumer_payment_only
(Python/MongoDB)            │                  _clients_analytics
    │                       │                       │
    │  Payment completed    │                       │
    │  PRODUCE: queuing.    │                       │
    │  external.payments.   │                       │
    │  analytics.json       │                       │
    │──────────────────────►│                       │
    │                       │                       │
    │                 CONSUME                        │
    │                       │──────────────────────►│
    │                       │                       │
    │                       │  ExternalPayments     │
    │                       │  AnalyticsHandler     │
    │                       │  → Upsert Aggregated  │
    │                       │    Analytics record   │
    │                       │  → Amounts: cents ÷ 100│
    │                       │  → Recalculate avg    │
    │                       │    basket value       │
```

---

## Synchronous Inter-Service Communication Map

| Caller | Callee | Protocol | Purpose | Config Vars |
|--------|--------|----------|---------|-------------|
| backend_v2 | ms-promo | HTTP :8001 | Promotion evaluation, CRUD, bulk ops | `MS_PROMO_{PROTOCOL,HOST,PORT}` |
| backend_v2 | ms-loyalty | HTTP :8002 | Referral campaigns, code validation | `MS_LOYALTY_{HOST,PORT}` |
| backend_v2 | ms-tax (legacy) | gRPC :9000 | Tax calculation | `MS_TAX_{HOST,PORT}` |
| backend_v2 | ms_tax (new) | HTTP :80 | Tax calculation | `MS_TAX_NEW_{PROTOCOL,HOST,PORT}` |
| backend_v2 | pay | HTTP :80 | Payment sessions, refunds, card ops | `MS_PAY_{HOST,PORT,PROTOCOL}` |
| backend_v2 | voucher | HTTP :80 | Voucher/gift card redemption | (service URL in settings) |
| backend_v2 | inventory | HTTP :8000 | Item lookup, search, batch import | `INVENTORY_SERVICE_{PROTOCOL,HOST,PORT}` |
| pay-gateway | pay | HTTP :80 | Client-facing payment proxy | `PAYMENT_SERVICE_HOST` |
| client apps | pay-gateway | HTTP :3000 | Payment operations | Kong route: `pg-{env}.mishipay.com` |

---

## Asynchronous Event Map (Kafka)

| Event | Topic | Producer | Consumer | Message Key | Payload |
|-------|-------|----------|----------|-------------|---------|
| Payment success | `mishi.streams.post-payment-success` | `PostPaymentSuccessProducerAdapter` | `PostPaymentSuccessConsumer` | `order_id` | `{ payment_id }` |
| Refund success | `mishi.streams.common` | `PostRefundSuccessProducerAdapter` | `CommonConsumer` | `order_id` | `{ order_id, refund_id, consumer_identifier_key }` |
| Dashboard verification | `mishi.streams.common` | `PostDashboardVerficationProducerAdapter` | `CommonConsumer` | `order_id` | `{ order_id, consumer_identifier_key }` |
| Transaction submitted | `mishi.streams.post-transaction` | `PostTransactionProducerAdapter` | External | `order_id` | `{ external_id, order_id, store_type, payment_status }` |
| Transaction confirmed | `mishi.streams.post-transaction-success` | External | `CommonConsumer` | — | `{ order_id, fiscalisation_id }` |
| FTC POSLOG | `flying_tiger.order.post_transaction` | `TriggerPoslogAdapter` | External | — | Full poslog payload |
| Cash desk shift end | `flying_tiger.cash_desk.staff_shifts.end_shift_event` | `CashDeskEndShiftEventProducerAdapter` | External | — | Full shift data |
| Payment analytics | `queuing.external.payments.analytics.json` | External (`pay`) | `PaymentOnlyClientsAnalyticsConsumer` | — | `{ store_id, total_basket_value, total_txn_count, ... }` |

---

## Background Task Queue (Dramatiq/RabbitMQ)

In addition to Kafka for inter-service events, the monolith uses **Dramatiq** with **RabbitMQ** for internal background task processing:

| Task | Module | Purpose |
|------|--------|---------|
| Refund processing | `mishipay_retail_analytics` | Process refund operations asynchronously |
| Async HTTP calls | `mishipay_retail_analytics` | Non-blocking external API calls |
| POSLOG generation | `mishipay_retail_analytics` | Generate POS transaction logs |
| Task lifecycle tracking | `mishipay_retail_analytics` | `CustomAdminMiddleware` tracks task execution |

RabbitMQ runs as `mpay_rabbitmq` container on `mpay_net` (AMQP port 5672). Dramatiq workers run within the `backend_v2` container.

---

## Data Storage Distribution

| Data | Storage | Service | Access Pattern |
|------|---------|---------|----------------|
| Orders, customers, items, baskets | PostgreSQL (`backend_db_v2`) | Monolith | CRUD via Django ORM |
| Keycloak users/realms | PostgreSQL (`backend_db_v2`) | Keycloak | Identity management |
| Promotions | Percona MySQL (`ms_promo_db`) | ms-promo | Evaluation engine |
| Loyalty/referrals | Percona MySQL (`ms_promo_db`) | ms-loyalty | Campaign management |
| Tax tables (legacy) | Percona MySQL (`ms_tax_db`) | ms-tax (gRPC) | Tax calculation |
| Tax tables (new) | MySQL (`tax-db`) | ms_tax (HTTP) | Tax calculation |
| Payment transactions | MongoDB (`mongo-db`) | pay | Payment processing |
| Vouchers/gift cards | MySQL (`voucher-db`) | voucher | Voucher management |
| Item inventory | MongoDB + Elasticsearch (`mongo-inventory`, `elastic-inventory`) | inventory | Item lookup + search |
| Session/cache data | Redis (`mpay_redis`) | Monolith | Caching, RFID EPC tracking |
| Event streaming | Confluent Cloud Kafka | All | Async event processing |
| Task queue | RabbitMQ (`mpay_rabbitmq`) | Monolith (Dramatiq) | Background jobs |
| Settlement records | PostgreSQL (`backend_db_v2`) | Monolith | Cron-based reconciliation |
| Subscription state | PostgreSQL (`backend_db_v2`) | Monolith | Lifecycle tracking |
| Fiscal records | PostgreSQL (`backend_db_v2`) | Monolith | Compliance tracking |

---

## Cross-Flow Data Dependencies

This matrix shows which data entities flow between the major subsystems:

```
                    ┌──────────┬──────────┬──────────┬──────────┬──────────┐
                    │  Items   │  Orders  │ Payments │Promotions│ Customers│
  ──────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
  Items             │    ●     │BasketItem│          │ SKU/cat  │          │
                    │          │ snapshot │          │ lookup   │          │
  ──────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
  Orders            │ stock    │    ●     │ 1:1 link │ audit log│ basket   │
                    │ decrement│          │          │ totals   │ owner    │
  ──────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
  Payments          │          │ captured │    ●     │          │ saved    │
                    │          │ amount   │          │          │ cards    │
  ──────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
  Promotions        │ offer    │ discount │ payment  │    ●     │ loyalty  │
                    │ codes    │ applied  │ amount   │          │ tier     │
  ──────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
  Receipts          │ item     │ order    │ payment  │ promo    │ customer │
                    │ details  │ details  │ details  │ details  │ email    │
  ──────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
  Settlement        │          │ order    │ PSP refs │          │          │
                    │          │ amounts  │ + fees   │          │          │
  ──────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
  Analytics         │ item     │ order    │ payment  │ promo    │ customer │
                    │ scans    │ events   │ events   │ savings  │ events   │
  ──────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

---

## Dependencies

- **Upstream:** [Architecture Overview](../01-architecture-overview.md) for the full system diagram
- **Kafka details:** [Support Modules — Pub/Sub](../platform/support-modules.md#pubsub-messaging-module-pubsub) for producer/consumer base class implementation
- **Microservices client:** [Platform Integrations](../platform/integrations.md#microservices-client-microservices_client) for in-process routing details
- **gRPC Tax service:** [Platform Integrations](../platform/integrations.md#grpc-tax-service) for protobuf message details
- **Payment service:** [Platform Payments](../platform/payments.md) for payment flow details
- **Promotions service:** [Promotions Service Overview](../promotions-service/overview.md) for evaluation API
- **Order lifecycle:** [Platform Orders](../platform/orders.md) for order model and completion pipeline
- **Receipt generation:** [Platform Emails & Alerts](../platform/emails-alerts.md) for receipt strategy pattern
- **Fiscalisation:** [Support Modules — Fiscalisation](../platform/support-modules.md#fiscalisation-module-fiscalisation) for fiscal compliance
- **RFID tracking:** [Platform Orders — RFID](../platform/orders.md) for vendor integrations
- **Settlement:** [Platform Emails & Alerts — Settlement](../platform/emails-alerts.md#payment-alerts--settlement-reconciliation) for reconciliation pipeline
- **Subscriptions:** [Platform Subscriptions](../platform/subscriptions-referrals.md) for subscription lifecycle
- **Inventory:** [Platform Inventory](../platform/inventory.md) for import pipeline

## Open Questions

- What external service(s) consume the `mishi.streams.post-transaction`, `flying_tiger.order.post_transaction`, and `flying_tiger.cash_desk.staff_shifts.end_shift_event` topics? No consumers for these exist in the monolith codebase.
- What service produces to `mishi.streams.post-transaction-success`? The monolith only consumes from it. Is this the fiscal authority callback service?
- Is there a plan to add dead-letter queues for Kafka messages that fail after retry exhaustion?
- How are Kafka consumer container replicas managed in production? Is there auto-scaling, or are they fixed at 1 instance per consumer type?
- The `pay` service publishes to `queuing.external.payments.analytics.json` — is this the only topic it produces to, or are there additional topics from the payment service?
- Is Dramatiq/RabbitMQ being phased out in favour of Kafka for all async processing, or do they serve distinct use cases?
- How does the platform handle partial payment failures mid-transaction (e.g., terminal timeout during capture)? Is there a retry/recovery mechanism?
- For offline sync: how are duplicate offline transactions detected if the same transaction syncs twice?
- Settlement reconciliation uses a PSP reference prefix heuristic (`ch_`/`pi_` → Stripe, `pay` → Checkout). Is this reliable across all current and future PSPs?
