# Retail Orders Module (`mishipay_retail_orders`)

> Last updated by: Task T17 — Views, order fulfilment, receipt generation, RFID tracking, offline sync, management commands, tests

## Overview

The `mishipay_retail_orders` module manages the complete order lifecycle in the MishiPay platform — from order creation at checkout through payment capture, completion, fulfilment, receipt generation, and post-transaction processing. It is one of the largest modules in the codebase (223 files) and acts as the central orchestration point where baskets become paid orders and trigger downstream processes across dozens of retailer integrations.

The module links a `Basket` (from `mishipay_items`) to an `Order`, assigns sequential identifiers (`o_id`, `transaction_id_poslog`), manages order/payment status transitions, and dispatches retailer-specific logic via a store-type dispatch pattern consistent with other MishiPay modules.

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Order** | A record linking a completed basket to a store, with payment status, sequence IDs, and lifecycle tracking |
| **o_id** | Sequential order number per store, generated at checkout via `StoreTransactionSequenceNumber` with `SELECT FOR UPDATE` locking |
| **transaction_id_poslog** | Sequential POS log transaction ID assigned after payment, used for retailer ERP/POS integration |
| **Order Completion** | Multi-step process triggered after payment: VAT calculation, poslog assignment, retailer-specific fulfilment, receipt sending, coupon redemption, notifications |
| **Click & Collect (CC)** | Alternative purchase flow with its own status lifecycle (`created` → `accepted` → `ready_to_collect` → `collected`) |
| **Store-Type Dispatch** | Pattern where behaviour is selected via `store.store_type.lower()` lookups in dispatch maps (70+ store types registered) |
| **Quota** | Generic quota tracking model used for retailer-specific purchase limits (e.g., DDF duty-free allowances) |

## Architecture

```
mishipay_retail_orders/
├── __init__.py                     # Status constants (OrderStatus, OrderPaymentStatus, CCOrderStatus, ReceiptStatus)
├── models.py                       # Order, Quota models (26K)
├── serializers.py                  # V1 serializers (21K)
├── serializers_v2.py               # V2 serializers + audit log + offline sync (28K)
├── admin.py                        # Django admin registrations (2.4K)
├── app_settings.py                 # Store-type dispatch maps & constants (26K)
├── views.py                        # Main views (163K) — covered in T17
├── urls.py                         # URL routing (4.5K)
├── api.py                          # Internal service APIs (5.5K)
├── permissions.py                  # CreateOrderPermission, CreateOrderPermissionV2
├── create_order.py                 # Order creation logic (5.3K)
├── complete_order.py               # Eroski-specific completion (1.3K)
├── order_util.py                   # Sequence generation, discrepancy checks (9.3K)
├── order_fulfilment.py             # Retailer-specific fulfilment dispatch (83K) — covered in T17
├── order_verification.py           # Verification stub (206B)
├── get_order_details.py            # Order detail retrieval functions (18K)
├── receipt.py                      # Receipt generation (60K)
├── rfid_status.py                  # RFID sold/unsold tracking (27K)
├── offline_sync_with_monolith.py   # Offline transaction sync (44K)
├── saved_card_order.py             # Saved card order flow (9.3K)
├── tax.py                          # Tax client selector (1.1K)
├── mpay_tax.py                     # Legacy tax service client (8.9K)
├── mpay_tax_new.py                 # New ms-tax service client (9.6K)
├── analytics.py                    # Order analytics recording (2.7K)
├── utils.py                        # Date range helper, Brevo SMS sending (4.1K)
├── update_stock_quantity.py        # Stock quantity updates (1.0K)
├── goloso_poslog.py                # Goloso POS log (9.5K)
├── customer_retail_specific_profile_dao.py  # Retailer customer profile updates (9.5K)
├── management/commands/            # 78 management commands (poslog scripts, etc.)
├── migrations/                     # 38 migrations spanning Mar 2019 – Dec 2025
├── templates/                      # XML poslog templates (6 files)
├── tests/                          # 60+ test files with retailer-specific mock data
├── third_party_integrations/       # Clover, Decathlon, SAP integrations
├── views_v1/                       # Legacy WhatsApp views
├── whatsapp/                       # WhatsApp messaging client
└── emaar_utility/                  # Emaar integration client
```

## Data Models

### Order

**File:** `mishipay_retail_orders/models.py:49`

The central model representing a customer transaction. Links a `Basket` (one-to-one) to a `Store` (foreign key) with payment status tracking, sequential identifiers, and extensible JSON fields.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | AutoField | PK | Internal DB primary key |
| `order_id` | UUIDField | unique, default=uuid4 | Public unique order identifier |
| `o_id` | BigIntegerField | default=100000 | Sequential order number per store |
| `transaction_id_poslog` | CharField(10) | nullable | Sequential POS log ID assigned after payment |
| `offline_receipt_reference_id` | CharField(128) | nullable | Offline receipt reference for offline transactions |
| `offline_o_id` | CharField(128) | nullable | Offline order ID for offline transactions |
| `external_order_id` | CharField(128) | nullable | External system order ID (e.g., Clover) |
| `basket` | OneToOneField → Basket | CASCADE | The purchased basket |
| `store` | ForeignKey → Store | PROTECT | Store where order was created |
| `payment_status` | CharField(30) | indexed, default=`not_captured` | See OrderPaymentStatus |
| `bundle_id` | CharField(128) | required | Client app bundle identifier |
| `platform` | CharField(25) | nullable | Client platform (android, ios, web, etc.) |
| `is_rated` | BooleanField | default=False | Whether customer has rated this order |
| `order_status` | CharField(30) | indexed, default=`created` | See OrderStatus |
| `order_poslog_sequence_type` | CharField(32) | nullable | Poslog sequence type at time of completion |
| `created` | DateTimeField | auto_now_add | Order creation timestamp |
| `updated` | DateTimeField | auto_now | Last modification timestamp |
| `completion_date` | DateTimeField | nullable, indexed | When order was completed/paid |
| `receipt_status` | CharField(10) | default=`not_sent` | See ReceiptStatus |
| `pos_status` | BooleanField | default=False | Whether POS fulfilment succeeded |
| `discount_constraints_description` | CharField(255) | nullable | Discount constraint description |
| `order_prefix` | CharField(10) | nullable | Retailer-specific order prefix |
| `extra_data` | JSONField | default=dict | Extensible JSON (shift_id, poslog data, carters_transaction_id, qr_identifier, zatca_value, etc.) |
| `additional_order_details` | JSONField | default=dict | Additional details (guest_user_email, loyalty_points_info) |
| `cc_order_status` | CharField(30) | blank | Click & Collect status |
| `auto_registration` | BooleanField | default=False | Whether order is used in AutoReg flow |
| `cancelled_at` | DateTimeField | nullable | Cancellation timestamp |

**Database Indexes (6 composite indexes):**

| Index | Fields | Purpose |
|-------|--------|---------|
| 1 | `store`, `order_status`, `payment_status`, `completion_date` | Dashboard order listing queries |
| 2 | `store`, `o_id` | Order lookup by store and sequential ID |
| 3 | `store`, `transaction_id_poslog` | Poslog lookup |
| 4 | `store`, `order_status`, `payment_status`, `platform`, `completion_date` | Platform-filtered order queries |
| 5 | `store`, `offline_receipt_reference_id` | Offline receipt lookup |
| 6 | `store`, `offline_o_id` | Offline order lookup |

**Key Methods:**

- **`complete_status(completion_date=None, extra_steps=True)`** (`models.py:153`) — The main order completion method. Performs:
  1. Calculates `total_vat_price` (skipped for `saved_card` payments)
  2. Sets `payment_status` to `CAPTURED_DEMO_MODE` or `CAPTURED_LIVE_MODE`
  3. Sets `order_status` to `VERIFIED` (if auto-verification enabled) or `COMPLETED`
  4. Records `completion_date`
  5. Increments cashier kiosk shift transaction count (if platform is `android_cashier_kiosk`)
  6. Marks basket as expired
  7. Calculates VAT via `calculate_vat()`
  8. Assigns `transaction_id_poslog` via `get_store_order_poslog_sequence_after_payment()` — uses `SELECT FOR UPDATE` locking; WHSmith gets special double-lock handling for concurrent requests
  9. For Click & Collect orders: sets `cc_order_status` to `CREATED` and generates `token_number`; Badiani/StandardCafe auto-advance to `READY_TO_COLLECT`
  10. Flying Tiger: builds poslog payload and publishes to Kafka via `TriggerPoslogAdapter`
  11. Grandiose: calls loyalty fulfilment for loyal customers
  12. Carters: generates `carters_transaction_id` (format: `88` + store + register + poslog + date)
  13. Saudi Arabia stores: generates ZATCA QR code value
  14. Calls `complete_status_extra()` for post-order processing

- **`complete_status_extra()`** (`models.py:330`) — Post-payment processing:
  1. Firebase push notifications (dashboard topic + customer app)
  2. Calls `COMPLETE_ORDER_FUNCTION_MAP` (currently only Eroski)
  3. Sets EPCs to sold via `EPC_SOLD_FUNCTIONS_MAP` for RFID stores
  4. Executes order fulfilment via `ORDER_FULFILMENT_FUNCTION_MAP` (30+ retailer-specific functions)
  5. Updates retailer-specific customer profile via `RETAILER_PROFILE_FUNCTION_MAPPING`
  6. Redeems coupons via `redeem_coupons_helper()`
  7. Sends receipt via `send_receipt_v2()` (excluding Eroski, Hudson, Virgin, serial-number-required stores)
  8. Validates basket audit log totals against payment captured amount
  9. For Click & Collect: sends store email notification and logs subscription usage

- **`cancel_status()`** (`models.py:590`) — Sets status to `CANCELLED`, records `cancelled_at`, assigns poslog sequence
- **`check_sum_of_item_prices_with_payment_captured_amount()`** (`models.py:519`) — Discrepancy detection: compares sum of `BasketEntityAuditLog.modified_monetary_value` against `payment.captured_amount`, alerts Slack on mismatch
- **`check_for_basket_item_to_audit_logs_discrepancy()`** (`models.py:560`) — Checks for missing items or quantity mismatches between basket items and audit logs

### Quota

**File:** `mishipay_retail_orders/models.py:599`

Generic quota tracking model for retailer-specific purchase limits (e.g., Dubai Duty Free duty-free allowances).

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | AutoField | PK | Internal DB primary key |
| `store_type` | CharField(31) | unique_together with `unique_identifier` | Store type string |
| `unique_identifier` | CharField(128) | unique_together with `store_type` | Quota identifier (e.g., customer ID, product category) |
| `details` | JSONField | default=dict | Quota details (remaining amounts, limits, etc.) |
| `created` | DateTimeField | auto_now_add | Creation timestamp |
| `updated` | DateTimeField | auto_now | Last update timestamp |

**Indexes:** Composite index on `(store_type, unique_identifier)`.

## Order Lifecycle & Status Transitions

### OrderStatus (`__init__.py:1`)

| Status | Value | Description |
|--------|-------|-------------|
| `CREATED` | `created` | Order created at checkout, payment not yet captured |
| `COMPLETED` | `completed` | Payment captured, order processing complete |
| `REFUNDED` | `refunded` | Full or partial refund processed |
| `VERIFIED` | `verified` | Order verified at exit gate (auto or manual) |
| `VERIFICATION_FAILED` | `verification_failed` | Verification failed |
| `CANCELLED` | `cancelled` | Order cancelled before completion |

**Status Groups:**
- `DASHBOARD_ORDER_STATUSES` = `[COMPLETED, REFUNDED, VERIFIED, VERIFICATION_FAILED]` — visible on dashboard
- `SUCCESS_ORDER_STATUSES` = `[COMPLETED, REFUNDED, VERIFIED]` — considered successful

### OrderPaymentStatus (`__init__.py:19`)

| Status | Value | Description |
|--------|-------|-------------|
| `NOT_CAPTURED` | `not_captured` | Payment not yet captured (default) |
| `CAPTURED_DEMO_MODE` | `captured_demo_mode` | Payment captured in demo/sandbox mode |
| `CAPTURED_LIVE_MODE` | `captured_live_mode` | Payment captured in live/production mode |

### CCOrderStatus (Click & Collect) (`__init__.py:31`)

| Status | Value | Description |
|--------|-------|-------------|
| `CREATED` | `created` | CC order placed, awaiting store acceptance |
| `ACCEPTED` | `accepted` | Store accepted the order |
| `REJECTED_STORE` | `rejected_store` | Store rejected the order |
| `READY_TO_COLLECT` | `ready_to_collect` | Order ready for customer pickup |
| `COLLECTED` | `collected` | Customer collected the order |
| `NOT_COLLECTED` | `not_collected` | Customer did not collect |
| `REJECTED_CUSTOMER` | `rejected_customer` | Customer rejected/cancelled |

**Cancelled CC Statuses:** `CC_CANCELLED_ORDER_STATUSES` = `[REJECTED_STORE, NOT_COLLECTED, REJECTED_CUSTOMER]`

### ReceiptStatus (`__init__.py:59`)

| Status | Value | Description |
|--------|-------|-------------|
| `SENT` | `sent` | Receipt sent to customer |
| `NOT_SENT` | `not_sent` | Receipt not yet sent (default) |

### Lifecycle Flow

```
[Basket Created] → CreateOrder → Order(status=CREATED, payment=NOT_CAPTURED)
                                    │
                          [Payment Gateway]
                                    │
                          complete_status()
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
         auto_verification    no_auto_verify    demo_mode
                    │               │               │
              VERIFIED         COMPLETED     CAPTURED_DEMO_MODE
                    │               │               │
                    └───────────────┼───────────────┘
                                    │
                          complete_status_extra()
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Notifications   Order Fulfilment   Receipt Sent
              (Firebase)      (POS/ERP sync)     (Email/SMS)
                    │               │               │
                    └───────────────┼───────────────┘
                                    │
                              [Optional]
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
               REFUNDED      VERIFICATION_FAILED  CANCELLED
```

## Order Sequencing System

**File:** `order_util.py`

Orders receive two sequential identifiers, both managed via the `StoreTransactionSequenceNumber` model (from `dos.models`) with database-level `SELECT FOR UPDATE` locking to prevent duplicates under concurrent requests.

### o_id — Order Sequence Number

Generated at checkout via `get_store_order_sequence_during_checkout()` (`order_util.py:46`):
- Default range: 100,000 – 999,999 (wraps around)
- Flying Tiger: 100,000,000 – 999,999,999 (larger range)
- Uses `sequence_identifier = "any"` (always store-wide)
- Backward compatible: if existing sequence < 100,000, reads last `o_id` from Order table

### transaction_id_poslog — POS Log Sequence

Assigned after payment via `get_store_order_poslog_sequence_after_payment()` (`order_util.py:80`):
- Default range: 1 – 999,999
- Paradies/Univision/Charlotte: 1 – 9,999 (smaller range)
- `sequence_identifier` varies by `store.poslog_sequence_type`:
  - `STORE`: `"any"` (store-wide sequence)
  - `DEVICE`: `retailer_device_id` from `extra_data` (per-device sequence, e.g., DDF)
  - `DEVICE_TYPE`: platform name or `"sng_device_type"` (per-device-type)
- WHSmith: special concurrent-safe handling with double `SELECT FOR UPDATE` in `complete_status()` (`models.py:210-227`)
- Saved card baskets return `"saved-card"` instead of a numeric sequence

### Before-Payment Sequence

`get_store_order_poslog_sequence_before_payment()` (`order_util.py:153`): Returns the **next** poslog sequence number without incrementing — used for stores that need the poslog ID before payment completion (e.g., stores with `poslog_sequence_before_payment` store property).

## Serializers

### Input/Request Serializers (`serializers.py`)

| Serializer | Purpose | Key Validations |
|------------|---------|-----------------|
| `CreateOrderSerializer` | Validates order creation request | `customer` (UUID), `store_id` (validated exists), `basket_id` (validated not expired, has required addresses, not requiring approval), `bundle_id`, `platform` (validated against `PLATFORM_CHOICES_FROM_APP_REQUEST`) |
| `OrderDetailsSerializer` | Validates order details request | `store_id` (UUID, validated), `order_id` (UUID) |
| `OrderLotteryCodeSerializer` | Italian lottery code submission | `order_id` (UUID), `lottery_code` (max 8 chars) |
| `OrderTransferSerializer` | Order transfer between stores | `order_id` (UUID), `store_id` (UUID) |
| `GetOrderDetailsGuestQuerySerializer` | Guest order details lookup | `order_id` (UUID) |
| `BasketIdSerializer` | Basket validation for loyalty | `basket_id` (UUID) — validates basket exists, not expired, has `member_id` in `extra_data` |

### Output/Response Serializers

#### OrderSerializer (`serializers.py:114`)

Simple model serializer returning: `order_id`, `basket` (nested `BasketSerializer`), `o_id`, `order_status`. Uses `ref_name = 'Orders.Order'` to avoid OpenAPI schema clashes.

#### OrderBasketDetailsSerializer (V1) (`serializers.py:129`)

Full order response serializer with 40+ fields, all computed via `SerializerMethodField`. Key computed fields:

| Field | Source | Logic |
|-------|--------|-------|
| `basket` | `BasketItem` queryset | Returns `BasketItemSerializer` data for all items |
| `total_price` | `basket.get_total_amount()` | Original basket total |
| `discounted_price` | `basket.get_total_modified_monetary_value()[0]` | Price after discounts |
| `currency` / `currency_sign` | `store.ms_pay_currency` / `store.currency_sign` | Store currency |
| `date` | `order.completion_date` | Formatted completion date (truncates microseconds) |
| `payment_method` | `order.payments` | Defaults to `credit_card`; `saved_card` → `credit_card`; `ideal` → `ideal-{bank_name}`; alerts Slack if payment object missing |
| `demo` | `order.payments` | Checks `demo_mode_transaction` for ms_pay stores, otherwise `variant.sandbox` |
| `order_verification_required` | Store property | Checks `entity="order", variable_factor="order_verification_required"` |
| `is_order_verification_required` | Store property + risk score | Also checks `risk_engine_validation` and customer `risk_score` against `RISK_SCORE_THRESHOLD` |
| `order_identifier` | Store type | `"qr_identifier"` for Eroski/SA region, `"barcode_identifier"` for Carters, otherwise `"order_id"` |
| `tax_breakup` | `basket.tax_level_breakup` | Filters to non-zero tax amounts |
| `show_tax_breakup` | GlobalConfig | Checks `IN_APP_RECEIPT_TAX_BREAKUP_STORES` |
| `consent_required` | Customer consent check | Marketing consent check via `is_consent_required()` |
| `store_logo` / `store_banner_color` | ThemeProperty | Web platform theme values for `brandImage` / `retailerBannerColor` |
| `sub_total` | Calculated | `discounted_price - total_vat (if tax_calculator_type) - other_charges` |
| `show_sub_total` | GlobalConfig | Checks `SHOW_SUBTOTAL_REGION_STORE_TYPES` by region |
| `invoice_notice` | Store type | MUJI IT: formatted with `o_id`; Flying Tiger: random verification message |
| `other_charges` | `basket.extra_data["other_charges"]` | Additional charges (e.g., bag fees) |

#### OrderBasketDetailsSerializerV2 (`serializers_v2.py:21`)

Enhanced V2 version with additional fields over V1:

| Additional V2 Field | Purpose |
|---------------------|---------|
| `barcode_identifier` | Carters-specific transaction barcode from `extra_data["carters_transaction_id"]` |
| `qr_identifier` | QR code value from `extra_data["qr_identifier"]` |
| `loyalty_number` | Flying Tiger loyalty number from `RetailerSpecificCustomerProfile` |
| `o_id` (overridden) | Returns `transaction_id_poslog` instead of `o_id` for stores with `poslog_sequence_before_payment` |

Key V2 differences:
- Accesses store via `obj.store` (direct FK) instead of `obj.basket.store` — more efficient
- Adds DDF Visa footer QR in `get_additional_payment_details()` and Fiskaly QR footer
- Removes `vat_number`, `basket_amount_without_vat`, `show_vat` fields from V1

**Note:** `get_order_verification_required()` is defined **twice** in the V2 serializer (lines 153 and 201) — the second definition shadows the first.

#### BasketEntityAuditLogSerializerV3 (`serializers_v2.py:436`)

Serializes basket audit log entries for order detail responses. Returns per-item pricing with promotion details:

| Field | Source |
|-------|--------|
| `applicable_entity_type` | Model field |
| `retailer_code` | Model field |
| `applicable_offers` | `extra_data["item_info"]["applicable_offers"]` (strips `meta_data`) |
| `applied_offer` | Computed: `price_before_offer`, `price_after_offer`, `savings`, `quantity`, `sticker_text`, `label_text` |
| `vat_free_price_of_item` | `item_vat_free_price` property |
| `vat_type` | From store property `vat_type_visible` |
| `serial_number` | `extra_data["item_info"]["serial_number"]` |

The `to_representation()` override merges `item_info` data, adds `discount_price`, `price_after_discount`, `reference_id`, `cod_category`, `allowed_restricted_item`, `item_type`, `item_addons`, and handles Carters shopping bag product identifier masking.

#### Offline Sync Serializers (`serializers_v2.py:577-665`)

| Serializer | Purpose |
|------------|---------|
| `TaxDetailSerializer` | Tax detail: `tax_id`, `tax_code`, `tax_level`, `tax_percent`, `taxable_amount` |
| `TaxesAppliedSerializer` | Tax amount with nested detail |
| `CategoryDetailSerializer` | Category: `category_id`, `name` |
| `PricingGuidanceDetailSerializer` | 8 unit/total pricing fields (original, payable, discount, tax) |
| `BasketOrderItemSerializer` | Item in offline order: `item_id`, `name`, `item_type`, `scanned_barcode`, `retailer_product_id`, `quantity`, categories, pricing, taxes |
| `OfflineMonolithSyncObjectSerializer` | Full offline order sync: store/retailer/order/basket/user/payment IDs, pricing totals, items list, email, platform, status, timestamps, poslog details |
| `FailedOrdersSerializer` | Failed order summary: `order_id`, `o_id`, `transaction_id_poslog`, statuses, `created`, `device_id` (from extra_data), `total_order_amount` (from payments) |

### Validation Helper Functions (`serializers.py:53-65`)

- **`validate_store_id(store_id)`** — Checks store exists in database
- **`validate_platform(platform)`** — Validates against `PLATFORM_CHOICES_FROM_APP_REQUEST`

### StoreBasicSerializer (`serializers.py:69`)

Lightweight store serializer used in order responses: `store_id`, `store_type`, `item_source`, `retailer` (aliased from `name` for backward compatibility), `name`, `demo`, `region`, `sub_region`, `address`, `address_json`.

## Permissions

**File:** `permissions.py`

| Permission | Applied To | Logic |
|------------|-----------|-------|
| `CreateOrderPermission` | V1 order creation | Validates `store_id` and `basket_id` from request, checks `EntityActionConstraint` rules on the store (uses `eval()`-based constraint engine from `mishipay_core`) |
| `CreateOrderPermissionV2` | V2 order creation | Simplified: only validates `store_id` and `basket_id` presence, skips constraint evaluation |

## Store-Type Dispatch Maps

**File:** `app_settings.py` (26K)

The `app_settings.py` file is the central configuration hub, defining store-type → function/class dispatch maps for order creation, completion, fulfilment, receipt, and RFID operations. Contains 70+ registered store types.

### CREATE_ORDER_FUNCTION_MAP (66 entries)

Maps store types to either:
- **Functions** (e.g., `cisco_create_order`) — called as `func(post_data)`
- **BasketProcedure classes** (e.g., `EroskiBasketProcedure`) — instantiated as `Class(post_data, store_dict).create_order(post_data)`

Default fallback: `mishipay_create_order` (used for ~10 legacy store types not in the map).

### ORDER_DETAILS_FUNCTION_MAP (68 entries)

Maps store types to order detail retrieval functions:
- `mishipay_get_order_details` — ~8 legacy store types (MishiPay, DirectWines, Cisco, Decathlon, etc.)
- `compass_get_order_details` — ~58 store types (majority of retailers)
- `mlse_get_order_details` — MLSE-specific

### ALL_ORDER_DETAILS_FUNCTION_MAP (68 entries)

Same structure as `ORDER_DETAILS_FUNCTION_MAP` but for fetching all orders:
- `mishipay_get_all_order_details` vs `compass_get_all_order_details`

### COMPLETE_ORDER_FUNCTION_MAP (1 entry)

Currently only Eroski: `eroski_complete_order` — generates QR identifier string with outlet_id, date, customer, items.

### ORDER_FULFILMENT_FUNCTION_MAP (32 entries)

Maps store types to tuples of fulfilment functions (multiple functions per store type supported):

| Store Type | Fulfilment Functions |
|------------|---------------------|
| `decathlonstoretype` | `decathlon_order_fulfilment` |
| `virginstoretype` | `virgin_order_fulfilment`, `virgin_invoice` (2 functions) |
| `kikomilanostoretype` | `kiko_milano_zatca_qr_update`, `kiko_milano_order_fulfilment` (2 functions) |
| `flyingtigerstoretype` | `flying_tiger_loyalty_antavo_redeem_coupons` |
| `walmartstoretype` | `walmart_odoo_order_fulfilment` |
| `vengostoretype` | `vengo_order_fulfilment`, `generic_odoo_order_fulfilment` (2 functions) |
| ... | (26 more entries) |

### EPC_SOLD_FUNCTIONS_MAP / EPC_UNSOLD_FUNCTIONS_MAP

RFID EPC status tracking:

| Store Type | Sold Functions | Unsold Functions |
|------------|---------------|------------------|
| `mishipaystoretype` | `set_sold_mishipay_items` | `set_unsold_mishipay_items` |
| `nedapstoretype` | `set_sold_nedap` | `set_unsold_nedap` |
| `ciscostoretype` | `set_sold_redis`, `set_sold_sato` (2) | `set_unsold_redis`, `set_unsold_sato` (2) |
| `decathlonstoretype` | `set_sold_decathlon` | — |
| `averydennisonstoretype` | `set_sold_ilab` | `set_unsold_ilab` |
| `retailreloadstoretype` | `set_sold_retail_reload` | `set_unsold_retail_reload` |
| `virginstoretype` | `virgin_rfid` | — |

### ORDER_VERIFICATION_FUNCTION_MAP (2 entries)

- `mishipaystoretype`: `mishipay_order_verification`
- `dufrystoretype`: `mishipay_order_verification`

### RETAILER_PROFILE_FUNCTION_MAPPING (9 entries)

Updates retailer-specific customer profiles after order:
- `compassstoretype`: `create_retailer_customer_profile_object`
- `relaystoretype`: `relay_retailer_customer_profile_update`
- `eroskistoretype`: `eroski_retailer_customer_profile_update`
- `londisstoretype`: `bwg_retailer_customer_profile_update` (deprecated)
- `londisnewstoretype`, `applebynewstoretype`, `golosostoretype`, `badianistoretype`, `standardcafestoretype`: `retailer_customer_profile_update_new`

### Other Maps

- **`SERVERLESS_FUNCTION_FOR_COMPLETE_STATUS`** = `["flyingtigerstoretype"]` — Flying Tiger uses Kafka-based serverless completion
- **`CANCEL_SALE_FUNCTION_MAP`** = `{'grandiosestoretype': GrandioseBasketProcedure}` — Cancel sale support

## App Constants

**File:** `app_settings.py:1-50`

| Constant | Value | Description |
|----------|-------|-------------|
| `SUCCESS_RESPONSE` | `'success'` | Standard success status |
| `FAILED_RESPONSE` | `'failed'` | Standard failure status |
| `DEFAULT_O_ID_VALUE` | `100000` | Starting order sequence number (in `__init__.py`) |
| `POSLOG_MONTHLY` | `'monthly'` | Monthly poslog period |
| `VAT_APPLICABLE_STORE_TYPES` | `['dufrystoretype', 'relaystoretype']` | Store types requiring VAT display |

**Internationalized exit instruction messages** (20+): Store-specific messages displayed to customers after order (e.g., "Keep your receipt open", "Visit the exit verification counter", Finnish translations, security tag instructions).

**Receipt promo templates**: Discount and referral promotion text templates with `{discount}` and `{retailer}` placeholders.

## Order Creation Flow

### Standard Flow (`create_order.py`)

`mishipay_create_order(post_data)` (`create_order.py:44`):
1. Validates basket exists (unexpired, matching customer/store)
2. Checks if order already exists for this basket — if yes, returns existing order data and sends Slack warning
3. Creates `Order` with: basket, store, `bundle_id`, `platform`, `order_prefix`, `o_id` (from sequence), `extra_data` (device info)
4. If store has `risk_score_applicable`, updates customer risk score

### Cisco/Clover Flow (`create_order.py:106`)

`cisco_create_order(post_data)`:
1. Calls `mishipay_create_order()` first
2. Calls `clover_order_fulfilment()` to create order in Clover POS
3. Stores `clover_order_id` as `external_order_id`
4. If Clover creation fails: **deletes the order** and returns failure

### Store-Type Dispatch Flow (`api.py`)

`CreateOrderService.post()` (`api.py:60`):
1. Looks up store type in `CREATE_ORDER_FUNCTION_MAP`
2. If value is a **function**: calls `func(api_data)` directly
3. If value is a **class** (BasketProcedure): instantiates `Class(api_data, store_dict).create_order(api_data)`
4. Falls back to `mishipay_create_order` for unknown store types

## Tax System

**Files:** `tax.py`, `mpay_tax.py` (8.9K), `mpay_tax_new.py` (9.6K)

### TaxClientSelector (`tax.py:6`)

Strategy selector between legacy and new tax service clients based on `store.supports_new_tax_service`:

| Method | Legacy (TaxServiceClient) | New (TaxServiceClientNew) |
|--------|--------------------------|---------------------------|
| `delete_tax_table()` | `clear_all()` | `delete_tax_table()` |
| `import_tax_table(data)` | Queues rows then `commit_updates()` | Direct `import_tax_table()` call |

### Tax Service Architecture (from `mpay_tax_new.py` comments)

- Single table schema: `(id, store_id, tax_code, tax_level, retailer_tax_level_id, tax_percent)`
- Unique constraint: `(store_id, tax_code, retailer_tax_level_id)`
- Each item has a `tax_code`; each code can have multiple tax levels (state, federal)
- Tax calculation: exclusive = `taxable_amount * tax_percent / 100`; inclusive = `taxable_amount * tax_percent / (100 + tax_percent)`
- `STORE_TO_TAX_ROUNDING_RULE_MAP` — per-store-type tax rounding configuration

## Django Admin

**File:** `admin.py`

### OrderAdmin (`admin.py:10`)

| Feature | Configuration |
|---------|--------------|
| `list_display` | `order_id`, `basket_customer_email`, `guest_user_email`, `store`, `o_id`, `order_status`, `created`, `updated`, `transaction_id_poslog`, `completion_date` |
| `list_filter` | `created`, `updated` |
| `search_fields` | `=order_id` (exact match) |
| `raw_id_fields` | `basket` (avoids loading all baskets in dropdown) |
| `list_select_related` | `basket__customer`, `basket__store` (optimizes N+1) |
| Custom columns | `basket_customer_email` (from `basket.customer.email`), `guest_user_email` (from `additional_order_details`) |
| Actions | `resend_email_receipts` — bulk resend receipts for completed/verified orders via `send_receipt()`, updates `receipt_status` to SENT |

### QuotaAdmin (`admin.py:72`)

| Feature | Configuration |
|---------|--------------|
| `list_display` | `store_type`, `unique_identifier`, `created`, `updated` |
| `list_filter` | `store_type` |
| `search_fields` | `=unique_identifier` (exact match) |

## API Endpoints (URL Summary)

**File:** `urls.py` — 35 URL patterns across 5 categories.

### Public Customer APIs

| Method | Path | View | Description |
|--------|------|------|-------------|
| POST | `v1/create-order/` | `CreateOrder` | Create order (V1) |
| POST | `v2/create-order/` | `CreateOrderV2` | Create order (V2) |
| POST | `v1/get-order-details/` | `GetOrderDetails` | Get single order details |
| POST | `v1/get-all-order-details/` | `GetAllOrderDetails` | Get all customer orders |
| POST | `v2/get-order-details/` | `GetOrderDetailsV2` | Order details V2 |
| POST | `v3/get-order-details/` | `GetOrderDetailsV3` | Order details V3 |
| POST | `v1/return-order-info/` | `ReturnOrderInfo` | Return/refund order info |
| POST | `v1/order-receipt/` | `OrderReceipt` | Generate receipt |
| POST | `v2/order-receipt/` | `OrderReceiptV2` | Receipt V2 |
| GET | `v1/get-receipt-banners/` | `GetReceiptMessage` | Receipt promotional banners |
| POST | `v1/order/lottery-code` | `OrderLotteryCodeView` | Submit Italian lottery code |
| POST | `v1/order/order-transfer` | `OrderTransfer` | Transfer order to another store |
| POST | `v1/order/verify-for-exit-gate/` | `OrderVerifyForExitGate` | Exit gate verification V1 |
| POST | `v2/order/verify-for-exit-gate/` | `OrderVerifyForExitGateV2` | Exit gate verification V2 |
| POST | `v1/cancel-sale/` | `CancelSaleSessionView` | Cancel sale session |
| POST | `v1/failed-orders/` | `GetFailedOrders` | List failed orders |

### Guest (No Auth) APIs

| Method | Path | View | Description |
|--------|------|------|-------------|
| GET | `v1/guest/get-receipt-banners/` | `GetReceiptMessageGuestView` | Receipt banners (guest) |
| POST | `v2/guest/get-order-details/` | `GetOrderDetailsGuestView` | Order details (guest) |
| POST | `v2/guest/update_customer_email_and_send_order_receipt` | `SendEmailReceiptGuestView` | Update email & send receipt (guest) |

### Staff/Dashboard APIs

| Method | Path | View | Description |
|--------|------|------|-------------|
| POST | `v2/staff/order_receipt` | `StaffOrderReceipt` | Staff receipt generation |
| POST | `v3/staff/order_receipt` | `StaffOrderReceiptV2` | Staff receipt V2 |
| POST | `v2/staff/update_customer_email_and_send_order_receipt` | `UpdateCustomerEmailAndSendEmailReceipt` | Update customer email + resend |
| POST | `v2/order_receipt_excluding_refunds` | `OrderReceiptExcludingRefunds` | Receipt without refund items |
| POST | `v2/ddf/staff/failed_order_receipt` | `DDFStaffOrderReceipt` | DDF-specific failed order receipt |

### Internal Service APIs

| Method | Path | View | Description |
|--------|------|------|-------------|
| POST | `api/v1/create-order/` | `CreateOrderService` | Internal order creation service |
| POST | `api/v1/get-order-details/` | `GetOrderDetailsService` | Internal order details service |
| POST | `api/v1/get-all-order-details/` | `GetAllOrderDetailsService` | Internal all orders service |

### Specialty APIs

| Method | Path | View | Description |
|--------|------|------|-------------|
| POST | `v1/post-transaction/` | `OrderPostTransactionView` | Hudson post-transaction number |
| POST | `v1/lookup-transaction/` | `HudsonTransactionLookupView` | Hudson transaction lookup |
| POST | `v1/whatsapp/order-receipt` | `SendWhatsAppMessageView` | WhatsApp receipt |
| POST | `v1/whatsapp/webhook` | `WhatsAppMessagesWebhookView` | WhatsApp webhook |
| POST | `v1/offline/full_data_sync/` | `OfflineSyncMonolithData` | Offline → monolith data sync |
| POST | `v1/order/verify-for-exit-gate-vengo/` | `OrderVerifyForExitGateVengoV1` | Vengo exit gate verification |
| GET | `v1/vengo-entry-gate/get-qr-data/` | `GetEntryQRData` | Vengo entry QR data |
| POST | `v1/vengo-entry-gate/validate-qr/` | `ValidateEntryQRCode` | Vengo QR validation |

## Supporting Utilities

### Order Utilities (`order_util.py`)

- **`get_store_order_sequence_during_checkout(basket)`** — Atomic o_id generation with `SELECT FOR UPDATE`
- **`get_store_order_poslog_sequence_after_payment(order)`** — Atomic poslog sequence generation
- **`get_store_order_poslog_sequence_before_payment(order)`** — Non-incrementing poslog preview
- **`get_sequence_identifier(basket=None, order=None)`** — Determines sequence scope: `"any"` (store), `retailer_device_id` (device), platform (device_type)
- **`check_basket_discrepancies(order, store, ...)`** — Finds items missing in audit logs or vice versa
- **`log_and_alert_discrepancy(order, message, store, details)`** — Logs and Slack-alerts basket discrepancies
- **`generate_order_token(store)`** — Atomic Click & Collect token number generation via `TokenConfig`/`TokenCounter`

### General Utilities (`utils.py`)

- **`get_date_range(poslog_date, timezone)`** — Returns start/end datetime for a date in given timezone
- **`send_sms_with_brevo(order_obj)`** — Sends transactional SMS via Brevo (Sendinblue) API. Config-driven via `GlobalConfig["BREVO_SMS_CONFIG"]`. Currently supports KikoMilano-specific order number formatting.

### Analytics (`analytics.py`)

- **`record_create_order_analytics(request, response, status=True)`** — Records order creation event to analytics microservice via `AnalyticsRecordClient`. Captures `item_count`, `order_id`, common analytics params.

### Eroski Complete Order (`complete_order.py`)

- **`eroski_complete_order(order)`** — Generates Eroski QR identifier string (format: `S&G|01|{outlet}|{date}|{time}|{customer}|{order}|{items...}`) and saves to `extra_data["qr_identifier"]`, sets status to COMPLETED.

## Migration History

38 migrations spanning **March 2019 to December 2025** (Django 1.11.5 → 4.1.13):

| Period | Key Changes |
|--------|-------------|
| 2019 Q1 | Initial Order model (basket FK, order_status, receipt_status) |
| 2019 Q2 | Added `platform`, `discount_constraints_description`, `pos_status`, `extra_data`, `external_order_id` |
| 2019-2020 | `completion_date`, `additional_order_details`, composite indexes |
| 2021 | `transaction_id_poslog`, `cc_order_status`, `auto_registration` |
| 2022-2023 | More composite indexes, JSONField migrations, `order_poslog_sequence_type`, Quota model |
| 2023-2025 | Platform choices migration, `cancelled_at`, offline order fields (`offline_o_id`, `offline_receipt_reference_id`) with indexes |

5 merge migrations indicate parallel development branches reconciled during active development periods (2023).

## Dependencies

### Depends On
- **`mishipay_items`** — `Basket`, `BasketItem`, `BasketEntityAuditLog`, `BasketAddress`, BasketProcedure classes (70+)
- **`dos`** — `Store`, `StoreTransactionSequenceNumber`, `TokenConfig`, `TokenCounter`, `PURCHASE_TYPES`
- **`mishipay_core`** — `common_functions` (send_slack_message, send_receipt, etc.), `EntityActionConstraint`, `ThemeProperty`, notifications, permissions
- **`mishipay_retail_customers`** — `RetailerSpecificCustomerProfile`, address constants
- **`mishipay_coupons`** — `redeem_coupons_helper`
- **`mishipay_config`** — `GlobalConfig`
- **`mishipay_retail_analytics`** — Analytics recording
- **`mishipay_cashier_kiosk`** — Shift activity tracking
- **`mishipay_subscription`** — Subscription order logging
- **`pubsub`** — `TriggerPoslogAdapter` (Kafka)
- **`analytics`** — `NOTIFICATION_TRIGGER_CHOICES`
- **`microservices_client`** — Analytics client

### Depended On By
- **`mishipay_dashboard` / `dashboard`** — Order listing, refund processing, verification
- **`mishipay_alerts`** — Payment alerts and settlement reconciliation
- **`mishipay_retail_payments`** — Payment ↔ Order linkage (reverse relation `order.payments`)

## Notable Patterns

1. **Heavy `complete_status()` method** — The `Order.complete_status()` method at ~150 lines is the most critical code path. It runs synchronously in the request cycle and performs: VAT calculation, poslog sequencing (with DB locks), retailer-specific logic (Flying Tiger Kafka, Carters transaction IDs, ZATCA QR for Saudi Arabia, Grandiose loyalty), basket expiry, and saves. The separate `complete_status_extra()` handles async-capable work (notifications, fulfilment, receipts).

2. **Dual dispatch patterns** — `app_settings.py` supports both **function-based** and **class-based** dispatch. `CreateOrderService` checks `isinstance(func, types.FunctionType)` to differentiate, calling functions directly vs instantiating classes and calling `.create_order()`.

3. **Print statement in production** — `models.py:475` uses `print(err)` instead of `logger.error()` in the receipt-sending exception handler. Also `update_stock_quantity.py` uses `print()` for exceptions.

4. **`app_settings.py` circular import workaround** — The file has a large comment at the top explaining that basket procedure imports cannot be at the top of the file due to circular imports with Dramatiq management commands.

5. **Duplicate method definition** — `OrderBasketDetailsSerializerV2` defines `get_order_verification_required()` twice (`serializers_v2.py:153` and `serializers_v2.py:201`). The second definition shadows the first.

6. **Side effect in serializer** — `get_payment_method()` in both serializers writes to the database (`obj.save(update_fields=["extra_data"])`) to flag missing payment objects and sends Slack alerts — unexpected behavior in a read-only serialization path.

7. **No `models/`, `serializers/`, `services/`, `views/` directories** — All code is in flat files. No `tasks.py` or `signals.py` either. Business logic is organized across specialized flat files (83K `order_fulfilment.py`, 60K `receipt.py`, 44K `offline_sync_with_monolith.py`, etc.).

8. **Bare `except` clauses** — Multiple bare `except:` clauses in serializers (e.g., `get_date`, `get_show_tax_breakup`) and views (`HudsonTransactionLookupView` at line 1060 catches everything including `SystemExit`).

9. **`extra_data` as audit trail** — All external API calls store request, response, and success status in `order.extra_data` (JSON field), enabling debugging, retry, and operational visibility. The `poslog_sent` flag provides idempotency guards.

10. **Massive fulfilment file** — `order_fulfilment.py` (83K) contains 30+ retailer-specific fulfilment functions integrating with REST APIs, SOAP, Kafka, Fiskaly, ZATCA, Odoo, Shopify, XStore, and loyalty platforms. All follow the same pattern: check idempotency → build payload → call service → save response → Slack on failure.

11. **Monolithic views file** — `views.py` (3,359 lines, 30 view classes) with no directory-based organization. `StaffOrderReceipt` alone is ~530 lines.

12. **Copy-paste duplication** — `GetReceiptMessage` and `GetReceiptMessageGuestView` are near-identical copies (~400 lines each). `OrderVerifyForExitGateVengoV1._validate_as_entry_qr()` and `ValidateEntryQRCode` duplicate entry QR validation. `ReturnOrderInfo` uses `jjj`, `kkk` variable names.

13. **V1 → V2 migration pattern** — V1 views delegate to `OrderManagementClient` (internal API hop). V2 views dispatch directly via `CREATE_ORDER_FUNCTION_MAP`/`ORDER_DETAILS_FUNCTION_MAP`. Both coexist.

14. **Management commands as batch infrastructure** — 78 management commands (40+ poslog generators) serve as the primary batch processing mechanism. No Celery/Dramatiq tasks in this module. The fulfilment retry command uses `inspect.signature()` for function introspection.

15. **Slack as alerting backbone** — Failures across all modules are reported to retailer-specific Slack channels: `#alerts_poslog`, `#alerts_virgin_poslog`, `#alerts_grandiose`, `#event_network_nav_poslog`, `#alerts_ftc_loyalty_*`, `#alerts_config`, `#alerts_settlement`, `#whatsapp_alerts`.

## Open Questions

### From T16 (Models & Serializers)

- The `Order` model has a no-op `__init__` override (`models.py:141-142`) — `super(Order, self).__init__(*args, **kwargs)` does nothing extra. Is this leftover from a removed customization?
- `CreateOrderPermission` uses the `EntityActionConstraint` `eval()`-based engine (passing `locals()`) — is this constraint system actively used for order creation in production, or is it legacy? `CreateOrderPermissionV2` skips it entirely.
- `get_payment_method()` serializer writes to DB and sends Slack alerts — is this intentional side effect acceptable in a serialization path? It could cause unexpected writes during read operations.
- `complete_order.py` only has Eroski logic. Are other retailers' completion steps handled entirely within `complete_status()` or via `ORDER_FULFILMENT_FUNCTION_MAP`?
- The `sto_order_fulfilment` function is imported twice in `app_settings.py:79,81` — is this a copy-paste artifact?
- `check_sum_of_item_prices_with_payment_captured_amount` skips stores in `BASKET_AUDIT_LOG_NOT_CREATED_STORETYPES` — which store types are in this list and why don't they create audit logs?
- The `OfflineMonolithSyncObjectSerializer` suggests full offline → online sync capability. How widely is offline mode deployed?
- WhatsApp receipt sending is supported but appears to be a single view for specific stores. Which stores use WhatsApp receipts?

### From T17 (Views & Logic)

- `CreateOrderV2` has a bug at line 1829: `r_long = request.GET.get("r_lat")` copies latitude into longitude. Is this known? It means longitude data is never captured for V2 orders.
- `GetOrderDetailsV2` has no authentication (`authentication_classes = []`). Is this protected at the infrastructure level (Kong API gateway), or is this a security gap?
- Multiple views update customer records as a GET side effect (`GetOrderDetailsV2`, `GetOrderDetailsV3`, `GetOrderDetailsGuestView`). Is this intentional for email capture?
- `HudsonTransactionLookupView` has a bare `except:` clause at line 1060 and no authentication. Is this endpoint still in use?
- `StaffOrderReceipt` at ~530 lines is the most complex single view. Are there plans to refactor into the Strategy pattern (partially migrated via `NEW_PRINT_RECEIPT_CONFIGURATION_STORES`)?
- VenGo exit gate views use `verify=False` (SSL bypass) and have default credentials `'Test1234'` in code. Are these overridden in production?
- `order_verification.py` is a no-op stub (always returns `True`). Is actual verification logic implemented elsewhere, or is this feature disabled?
- The `set_unsold_decathlon()` function appears to use the same `deactivate_epc_url` as `set_sold_decathlon()`. Is this correct, or should it use a separate reactivation endpoint?
- `generate_tax_service_response()` in offline sync currently generates Flying Tiger Copenhagen-specific tax responses. Is offline sync only deployed for Flying Tiger, or does this need generalization?
- The offline sync flow (`generate_monolith_sync_objects`) creates User records with `is_guest_user=True`. Does this create orphan users over time, or are they cleaned up?
- SAP transaction log generation (`sap/transaction_logs.py`, 866 lines) implements SAP POS 2.3 fixed-width records. Which retailers use SAP TLogs and how are they delivered (SFTP, API)?
- The Emaar API client sends sales data to mall management. How many Emaar stores are active, and is the monthly report also generated via management command?

---

## Views Layer (T17)

**File:** `views.py` (3,359 lines) — the monolithic views file containing 30 view classes.
**Additional file:** `views_v1/whatsapp_view.py` (158 lines) — 2 WhatsApp-specific views.
**No `views/` directory exists** — all views are in the single `views.py` file.

### Module-Level Setup (`views.py:1-165`)

**Key data structures:**
- `stores_with_exit_instructions` (dict): Maps 18 store types to region lists that should receive exit instructions after order completion.
- `security_tag_list` (dict): Maps 3 store types (MUJI, Desigual, Virgin) to regions requiring security tag removal instructions.

**Helper functions:**
- **`get_exit_instructions(store_type, region, order)`** (`views.py:167-237`) — Builds exit instruction payloads via an extensive `if/elif` chain for store-type-specific messaging (Flying Tiger, Nude Foods, Virgin, Emma Sleep, Loves, MUJI with regional variants DK/IT/FI). Falls back to `store.extra_data.exit_instructions` or a default message. A `GlobalConfig` key `EXIT_INSTRUCTIONS_DISABLED` can disable per store-region.
- **`_get_store_obj_and_cancel_sale_session_list_function(store_type)`** (`views.py:240`) — Serializes a store and resolves the cancel sale session handler from `CANCEL_SALE_FUNCTION_MAP`.

### Order Creation Views

#### CreateOrder (V1) (`views.py:248`)

| Property | Value |
|----------|-------|
| **URL** | `POST v1/create-order/` |
| **Permissions** | `IsAuthenticated`, `CreateOrderPermission`, `ActiveStaffShiftPermission` |
| **Business Logic** | V1 order creation. Acquires `select_for_update` lock on the customer (GPP-1046 eager lock). Validates via `CreateOrderSerializer`. Delegates to `OrderManagementClient.create_order()` (internal HTTP call to order microservice). Records analytics on success. |
| **Notable** | Contains `print("serializer", post_data)` debug artifact. Language hardcoded to `'en'`. |

#### CreateOrderV2 (`views.py:1813`)

| Property | Value |
|----------|-------|
| **URL** | `POST v2/create-order/` |
| **Permissions** | `IsAuthenticated`, `CreateOrderPermissionV2`, `ActiveStaffShiftPermission` |
| **Business Logic** | V2 order creation (GPP-4830). No internal API call, no serializer validation. Directly validates basket existence, expiry, and address requirements. Uses **`CREATE_ORDER_FUNCTION_MAP`** store-type dispatch — if value is a function, calls directly; if class, instantiates with `(post_data, store_dict)` and calls `.create_order()`. Default fallback: `mishipay_create_order`. |
| **Bug** | Line 1829: `r_long = request.GET.get("r_lat")` copies latitude into longitude variable. |

### Order Details Views

#### GetOrderDetails (V1) (`views.py:322`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/get-order-details/` |
| **Permissions** | `IsAuthenticated` |
| **Business Logic** | V1 order details. Eager customer lock. Validates via `OrderDetailsSerializer`. Delegates to `OrderManagementClient.order_details()`. Enriches response with exit instructions and app header message. |

#### GetOrderDetailsV2 (`views.py:835`)

| Property | Value |
|----------|-------|
| **URL** | `GET v2/get-order-details/` |
| **Authentication** | None (`authentication_classes = []`) |
| **Business Logic** | Unauthenticated order details via `payment_session_id` + `order_id` query params. If order not completed, calls `PaymentService.payment_details()` to refresh payment status. Uses store-type dispatch via `ORDER_DETAILS_FUNCTION_MAP`. Enriches with exit instructions and app header. |
| **Side Effect** | Updates customer email from `additional_order_details` and calls `customer.save()` on a GET request. |
| **Security Concern** | No authentication at all. |

#### GetOrderDetailsV3 (`views.py:909`)

| Property | Value |
|----------|-------|
| **URL** | `GET v3/get-order-details/` |
| **Authentication** | Custom JWT (not DRF auth classes) |
| **Business Logic** | Decodes JWT from Authorization header using `settings.SECRET_KEY`. Extracts `order_id`, `timestamp`, `amount`. Enforces 15-minute JWT expiry. Same store-type dispatch as V2. |
| **Side Effect** | Same customer email save as V2. |

#### GetAllOrderDetails (`views.py:360`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/get-all-order-details/` |
| **Permissions** | `IsAuthenticated` |
| **Business Logic** | All orders for a customer. Eager customer lock. Delegates to `OrderManagementClient.get_all_order_details()`. |

#### GetOrderDetailsGuestView (`views.py:2375`)

| Property | Value |
|----------|-------|
| **URL** | `GET v2/guest/get-order-details/` |
| **Authentication** | None |
| **Business Logic** | Guest variant of V2. Validates UUID via `GetOrderDetailsGuestQuerySerializer`. Same store-type dispatch. Adds `hide_price_override_label: True`. For `riyadhtameerstoretype`, includes ZATCA fiscalisation value. Returns `customer_id` in response. |
| **Notable** | Dangling `else: pass` at line 2430. |

### Receipt & Banner Views

#### GetReceiptMessage (`views.py:384`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/get-receipt-banners/` |
| **Permissions** | `IsAuthenticated | JwtTokenPermission` |
| **Business Logic** | Returns promotional banners for receipt screen. ~237 lines of store-type-specific branching supporting: MUJI (GB, SE, FR, IT, FI), Londis (deactivated), SPAR, Breeze Thru, Nude Foods, Eroski. Banners include referral offers, discount offers, app download prompts, and overlay dialogs. Guest vs registered user differentiation (register CTA vs share-code CTA). |
| **Code Concern** | Extremely long method with heavily duplicated code across store types. Hardcoded URLs, discount amounts, and localized strings. Nearly fully duplicated in `GetReceiptMessageGuestView`. |

#### GetReceiptMessageGuestView (`views.py:1949`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/guest/get-receipt-banners/` |
| **Permissions** | None |
| **Business Logic** | Identical to `GetReceiptMessage` but takes `customer_id` as query param instead of deriving from `request.user`. ~389 lines, nearly a full copy-paste. |

#### OrderReceipt (V1) (`views.py:804`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/order-receipt/` |
| **Permissions** | `IsAdminUser` |
| **Business Logic** | V1 receipt rendering. Generates receipt context via `generate_context_for_receipt()`. Looks up email template settings from `settings.EMAIL_RECEIPT_SETTINGS`. Renders HTML template and returns raw `HttpResponse`. |

#### OrderReceiptV2 (`views.py:1743`)

| Property | Value |
|----------|-------|
| **URL** | `GET v2/order-receipt/` |
| **Permissions** | None |
| **Business Logic** | Uses `EmailReceipt` class. Supports `entity_type` (user/retailer) and `event` params. Returns raw HTML or structured JSON data if `get_data_only` query param set. |
| **Security Concern** | No authentication. |

#### OrderReceiptExcludingRefunds (`views.py:1790`)

| Property | Value |
|----------|-------|
| **URL** | `GET v2/order_receipt_excluding_refunds` |
| **Permissions** | `IsAuthenticated` |
| **Business Logic** | Receipt excluding refunded items/quantities via `generate_context_for_receipt_excluding_refund_items_qty()`. |

### Staff/Dashboard Views

#### StaffOrderReceipt (`views.py:1105`)

| Property | Value |
|----------|-------|
| **URL** | `GET v2/staff/order_receipt` |
| **Permissions** | `IsAuthenticated`, `ActiveStaffShiftPermission` |
| **Business Logic** | The most complex single view class (~530 lines). Generates POS printer receipts for staff/kiosk devices. Two code paths: (1) **New path**: checks `GlobalConfig("NEW_PRINT_RECEIPT_CONFIGURATION_STORES")` and delegates to `StrategyFactory` for receipt generation (EPSON format, STEB receipts, counter receipts, refund receipts). (2) **Old path**: For Flying Tiger delegates to `StaffOrderReceiptV2`; others use `store_type_class_map()` dispatch to `DDFPrintReceipt`, `GrandiosePrintReceipt`, `MLSEReceipt`, `GMGPrintReceipt`; fallthrough builds inline EPSON receipt markup (lines 1346-1632) with store headers, item tables, tax tables, payment info, barcodes, QR codes. |
| **Code Concern** | Inline EPSON receipt builder is extremely long. Uses `is not ""` (identity check) instead of `!= ""`. |

#### StaffOrderReceiptV2 (`views.py:1635`)

| Property | Value |
|----------|-------|
| **URL** | `GET v3/staff/order_receipt` |
| **Permissions** | `IsAuthenticated`, `ActiveStaffShiftPermission` |
| **Business Logic** | Simplified receipt using `GeneratePosReceipt` class. Supports `order` and `refund` receipt types. |

#### UpdateCustomerEmailAndSendEmailReceipt (`views.py:1671`)

| Property | Value |
|----------|-------|
| **URL** | `POST v2/staff/update_customer_email_and_send_order_receipt` |
| **Permissions** | `IsAuthenticated` |
| **Business Logic** | Updates customer email/phone on order and customer record, then sends receipt via email (`send_receipt_v2()`) and/or SMS (`send_sms_with_brevo()`). Checks fiscalisation status before sending. |

#### DDFStaffOrderReceipt (`views.py:2461`)

| Property | Value |
|----------|-------|
| **URL** | `GET v2/ddf/staff/failed_order_receipt` |
| **Permissions** | `IsAuthenticated`, `ActiveStaffShiftPermission` |
| **Business Logic** | DDF-specific receipt for failed transactions. Delegates to `StaffOrderReceipt.use_new_print_receipt_template()` or legacy `DDFPrintReceipt` with `failed_transaction=True`. Returns receipt duplicated as `[receipt, receipt]` (two copies). |

#### SendEmailReceiptGuestView (`views.py:2341`)

| Property | Value |
|----------|-------|
| **URL** | `POST v2/guest/update_customer_email_and_send_order_receipt` |
| **Permissions** | None |
| **Business Logic** | Guest variant of email receipt. Updates customer email then sends via `send_receipt_v2()`. |
| **Bug** | Error response passes raw `err` object (not stringified). |

### Order Operations Views

#### ReturnOrderInfo (`views.py:625`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/return-order-info/` |
| **Permissions** | `IsAuthenticated` |
| **Business Logic** | Retrieves order information structured for returns processing. Reads basket's `ms_promo_response` and structures items into `items_available` with sub-groups: `no_promos` and `promos`. Handles promo grouping with discount_info and requisite_info. |
| **Code Concern** | Uses variable names like `jjj`, `kkk` (poor readability). |

#### OrderLotteryCodeView (`views.py:994`)

| Property | Value |
|----------|-------|
| **URL** | `POST v1/order/lottery-code` |
| **Permissions** | `IsAuthenticated` |
| **Business Logic** | Saves Italian lottery code to `order.extra_data['lottery_code']`. Validates ownership. |
| **Typo** | Success message: "Succefully". |

#### OrderTransfer (`views.py:1064`)

| Property | Value |
|----------|-------|
| **URL** | `POST v1/order/order-transfer` |
| **Permissions** | `IsAuthenticated` |
| **Business Logic** | Transfers an order's basket to the current authenticated user. Within `transaction.atomic()`: expires all existing baskets for the user at the same store, then reassigns basket's customer. Returns basket financial summary. |

#### CancelSaleSessionView (`views.py:2645`)

| Property | Value |
|----------|-------|
| **URL** | `POST v1/cancel-sale/` |
| **Permissions** | `IsAuthenticated`, `ActiveStorePermission` |
| **Business Logic** | Cancels a customer sale session. Validates basket via `BasketIdSerializer`. Uses `CANCEL_SALE_FUNCTION_MAP` dispatch: resolves cancel handler class by store type and calls `.cancel_sale_session(member_id, basket_obj)`. Custom status 460 for expired baskets. |

#### GetFailedOrders (`views.py:2585`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/failed-orders/` (ListAPIView) |
| **Permissions** | `DashboardPermission`, `StorePermission` |
| **Business Logic** | Paginated list of failed orders (status `created`/`cancelled` with `ms_pay_payment_session_id`). Default: last 48 hours unless `o_id`/`order_id` specified. Calls payment microservice (`FAILED_PAYMENT_SESSIONS_URL`) for payment session IDs and merges into response. |
| **Serializer** | `FailedOrdersSerializer` |

### Exit Gate Verification Views

#### OrderVerifyForExitGate (`views.py:2512`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/order/verify-for-exit-gate/` |
| **Permissions** | None (by design — called by gate hardware) |
| **Business Logic** | Grandiose exit gate verification. Takes `qrcode` query param (UUID). Validates UUID v4. Checks for completed/verified order within last 2 minutes. Returns `0` (valid) or `1` (invalid). |

#### OrderVerifyForExitGateV2 (`views.py:2548`)

| Property | Value |
|----------|-------|
| **URL** | `GET v2/order/verify-for-exit-gate/` |
| **Permissions** | None |
| **Business Logic** | Grandiose V2. More flexible — accepts either order UUID or transaction ID string (format: `T{retailer_device_id:10}{transaction_id:9}`). Queries all non-demo Grandiose stores for matches within 2 minutes. |

#### OrderVerifyForExitGateVengoV1 (`views.py:2744`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/order/verify-for-exit-gate-vengo/` |
| **Permissions** | None |
| **Business Logic** | VenGo dual-purpose gate verification: **Exit QR** (integer in `order.extra_data['exit_qr']`, queries orders completed within 10 minutes, triggers physical exit gate via HTTP API) and **Entry QR** (format `{retailer_store_id}{MMDDHHMM}`, decoded via `_decode_entry_qr()`, validates expiry with year-wrapping logic, triggers entry gate). Gate HTTP calls use Basic Auth from `GlobalConfig`. |
| **Security Concerns** | Uses `verify=False` (SSL bypass). Default credentials `'Test1234'` in code. |

#### GetEntryQRData (`views.py:3034`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/vengo-entry-gate/get-qr-data/` |
| **Permissions** | `IsAuthenticated` |
| **Business Logic** | Generates entry QR code data for VenGo stores. Validates numeric `retailer_store_id` (max 6 digits). Generates 10-minute-validity QR: `{retailer_store_id}{MMDDHHMM}`. |

#### ValidateEntryQRCode (`views.py:3138`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/vengo-entry-gate/validate-qr/` |
| **Permissions** | None |
| **Business Logic** | Standalone entry QR validation for VenGo. Decodes QR payload, checks expiry, triggers gate. Response uses `authorized: true/false` and `action: "open_door"/"deny"`. |
| **Code Concern** | Largely duplicates logic from `OrderVerifyForExitGateVengoV1._validate_as_entry_qr()`. |

### Hudson-Specific Views

#### OrderPostTransactionView (`views.py:1013`)

| Property | Value |
|----------|-------|
| **URL** | `POST v1/post-transaction/` |
| **Permissions** | `IsAuthenticated` |
| **Business Logic** | Hudson-specific. Triggers post-transaction processing for completed Hudson orders via `hudson_order_fulfilment()`. Only works with `HudsonStoreType` stores. |

#### HudsonTransactionLookupView (`views.py:1031`)

| Property | Value |
|----------|-------|
| **URL** | `GET v1/lookup-transaction/` |
| **Permissions** | None |
| **Business Logic** | Hudson-specific transaction number lookup for completed orders via `HudsonTransactionLookup` client. Guards against orders already tagged with a transaction number. |
| **Security Concern** | No authentication, no permissions. |
| **Bug** | Bare `except:` clause at line 1060 catches everything including `SystemExit`, `KeyboardInterrupt`. |

### Offline & WhatsApp Views

#### OfflineSyncMonolithData (`views.py:2711`)

| Property | Value |
|----------|-------|
| **URL** | `POST v1/offline/full_data_sync/` |
| **Permissions** | None (default) |
| **Business Logic** | Accepts offline order/basket JSON and syncs to monolith database. Validates via `OfflineMonolithSyncObjectSerializer`. Delegates to `generate_monolith_sync_objects()`. In DEBUG mode includes detailed errors. |

#### SendWhatsAppMessageView (`views_v1/whatsapp_view.py:22`)

| Property | Value |
|----------|-------|
| **URL** | `POST v1/whatsapp/order-receipt` |
| **Permissions** | `IsAuthenticated` |
| **Business Logic** | Sends order receipt via WhatsApp. Creates `WhatsAppClient` and calls `send_receipt(phone_number)`. On error, alerts Slack `#whatsapp_alerts`. |

#### WhatsAppMessagesWebhookView (`views_v1/whatsapp_view.py:55`)

| Property | Value |
|----------|-------|
| **URL** | `GET/POST v1/whatsapp/webhook` |
| **Permissions** | None |
| **Business Logic** | **GET**: WhatsApp webhook verification — validates `hub.verify_token` and returns `hub.challenge`. **POST**: Webhook handler — verifies HMAC-SHA256 signature via `X-Hub-Signature-256`, processes message status updates, alerts Slack on delivery failures. |

### View-Layer Cross-Cutting Patterns

1. **GPP-1046 Eager Lock**: Used in CreateOrder, GetOrderDetails, GetAllOrderDetails, GetReceiptMessage. Acquires `select_for_update()` on the customer within `@transaction.atomic`. Silently catches `DoesNotExist`.

2. **Response Structure**: Most views use `get_generic_response_structure()` returning `{error, status, message, data}`. Inconsistency: some use this, others use DRF's standard `Response`, some pass raw exception objects instead of strings.

3. **Authentication Gaps**: 10+ views have no authentication: `GetOrderDetailsV2`, `OrderReceiptV2`, `HudsonTransactionLookupView`, all guest views, all exit gate views, and `OfflineSyncMonolithData`. Exit gate views intentionally lack auth (hardware-called).

4. **Internationalization**: Uses `gettext_lazy` for some MUJI FR/IT/FI strings, but most strings are hardcoded English. Language often hardcoded to `'en'`.

---

## Order Fulfilment System (T17)

**File:** `order_fulfilment.py` (83K) — the largest single file in the module. Contains retailer-specific post-payment fulfilment functions that sync order data to external POS/ERP systems.

### Fulfilment Architecture

Each fulfilment function follows a consistent pattern:
1. Check idempotency guard (`order.extra_data.get('poslog_sent', False)`)
2. Build request payload (retailer-specific format: JSON, XML, SOAP, CSV)
3. Call external service (REST API, SOAP, file write, Kafka)
4. Save response to `order.extra_data`
5. Set `poslog_sent = True` on success
6. Alert Slack on failure

### Decorator: `@check_for_discrepancy`

Applied to many fulfilment functions. Validates basket totals vs payment captured amounts before proceeding with fulfilment. Prevents syncing incorrect data to retailer systems.

### Fulfilment Functions by Retailer

| Function | Retailer | Integration Type | Key Details |
|----------|----------|-----------------|-------------|
| `hudson_order_fulfilment(order)` | Hudson | REST API (`HudsonPostTransaction`) | Builds detailed order payload with item details, tax breakups, boarding pass data. Stores response in `extra_data["poslog_response"]`. Supports retry via `poslog_history`. |
| `decathlon_order_fulfilment(order)` | Decathlon | REST API (`DecathlonOrderFulfilment`) | Calls Decathlon's fulfilment API. Stores `external_order_id` from response. |
| `saturn_order_fulfilment(order)` | Saturn (MediaSaturn) | SOAP via `zeep` | Constructs SOAP request for Saturn's ESB fulfillment service with delivery details. |
| `clover_order_fulfilment(post_data)` | Cisco/Clover | REST API | Creates order in Clover POS (`/v3/merchants/{id}/orders`), adds line items, returns `clover_order_id`. |
| `bwg_order_fulfilment(order)` | BWG (Londis, Spar, etc.) | Async Dramatiq task | Enqueues `send_bwg_poslog_to_retailer` with 2-second delay. Handles "lucky draw" loyalty amounts. |
| `bestseller_order_poslog(order)` | Bestseller (VeroModa) | XML file write | Generates XML POSlog from Django template. Calculates Denmark-specific 25% VAT. |
| `picwic_order_poslog(order)` | Picwic | XML via HTTP POST | Generates XML POSlog via templates. Handles loyalty card/points integration. |
| `dufry_order_fulfilment(order)` | Dufry (Duty Free) | XML file write (~250 lines) | Complex: boarding pass parsing, Julian date conversions, duty-free logic, card brand identification, XML POSlog generation. |
| `virgin_order_fulfilment(order, refund_order)` | Virgin | REST API (`VirginPostTransaction`) | Handles sale and refund transactions. Saves `sales_order_id`. `@check_for_discrepancy`. |
| `virgin_invoice(order, refund_order)` | Virgin | REST API (`VirginInvoice`) | Posts invoice after successful `transaction_posted`. |
| `dimona_order_fulfilment(order, refund_order)` | Dimona | REST API (`DimonaPostTransaction`) | Standard req/res pattern. |
| `d365_order_fulfilment(order)` | D365 retailers (Viva, BloomingWear) | REST API (`D365PostTransaction`) | Config key: `{chain}-{region}` from `GlobalConfig`. `@check_for_discrepancy`. |
| `pantree_order_fulfilment(order)` | Pantree | REST API (`PantreePostTransaction`) | Sets order status to `VERIFIED`. |
| `gillan_order_fulfilment(order, refund_order)` | Gillan | REST API | Sale or refund via `GillanPostTransaction`/`GillanRefundPostTransaction`. |
| `mysalon_stop_order_fulfilment(order, refund_order)` | MySalonStop | REST API | Sale or refund transactions. |
| `carters_order_fulfilment(order)` | Carter's | REST API (`CartersPostTransaction`) | Delegates to `standard_client_order_fulfilment_common`. |
| `funky_buddha_order_fulfilment(order)` | Funky Buddha | REST API + Kafka | Greek fiscal compliance (creates `Fiscalisation` records). Optionally publishes to Kafka via `PostTransactionProducerAdapter` for async mark ID retrieval. |
| `generic_shopify_order_fulfilment(order, refund_order, daily_job)` | Shopify retailers | REST API | `ShopifyPostTransaction`/`ShopifyRefundTransaction`. Supports daily retry jobs. |
| `grandiose_order_fulfilment(order)` | Grandiose | REST API (`GrandiosePostTransaction`) | Standard pattern. |
| `grandiose_order_loyalty_fulfilment(member_id, order)` | Grandiose | TalonOne loyalty API | Calculates loyalty points from effects via `GrandioseLoyaltyClientTalonOne`. |
| `dubai_duty_free_order_fulfilment(order)` | Dubai Duty Free | Internal DB | Updates DDF quota via `DDFQuota` after order completion. |
| `ssp_order_fulfilment(order)` | SSP (food service) | REST API (`SSPCompletePayment`) | Standard pattern. |
| `mlse_order_fulfilment(order, refund_order)` | MLSE | REST API (`MLSEPostTransaction`) | Handles refund orders. |
| `flying_tiger_azadea_moengage_add_event(order)` | Flying Tiger (Azadea) | MoEngage API | Marketing event tracking via `MoengageProcessBulkCreateEvent`. |
| `event_network_order_fulfilment(order)` | Event Network | XStore POS (multi-step) | 5-step flow: (1) create retail transaction, (2) add line items, (3) validate card, (4) add card, (5) commit transaction. |
| `generic_odoo_order_fulfilment(order, refund_order, client)` | Odoo retailers | Odoo API | Generic Odoo POS integration via `OdooPostTransaction`/`OdooRefundPostTransaction`. |
| `walmart_odoo_order_fulfilment(order, refund_order)` | Walmart | Odoo API | Wraps `generic_odoo_order_fulfilment` with Walmart-specific clients. |
| `vengo_order_fulfilment(order, refund_order)` | VenGo | Fiskaly API | German fiscal compliance (e-invoices). Generates exit QR codes. |
| `flying_tiger_loyalty_antavo_redeem_coupons(order)` | Flying Tiger | Antavo loyalty API | Redeems loyalty coupons by iterating applied coupons. |
| `kch_deduct_inventory(order)` | KCH | REST API | Inventory deduction via `KCHInventoryDeductClient`. |
| `apos_plus_order_fulfilment(order, poslog_sent_key)` | APOS+ retailers | REST API | Two-phase: build JSON then POST. Checks shift open after success. |
| `kiko_milano_order_fulfilment(order)` | Kiko Milano | APOS+ | Delegates to `apos_plus_order_fulfilment`. |
| `kiko_milano_zatca_qr_update(order)` | Kiko Milano (SA) | Cygnet ZATCA API | Saudi ZATCA QR code generation for tax compliance. |
| `seventh_heaven_order_fulfilment(order, refund_order)` | Seventh Heaven | REST API | Full refund: `revert_bill`; partial refund: `update_bill_for_refund`. |

### External Services Called

| Service | Integration Method |
|---------|-------------------|
| Hudson POS | REST API |
| Decathlon ERP | REST API |
| Saturn ESB | SOAP via zeep |
| Clover POS | REST API |
| BWG/Londis | Async Dramatiq + REST |
| Dufry/Bestseller | XML file write |
| Virgin, Dimona, D365, Pantree, Gillan, MySalonStop, Carter's, SSP, MLSE, Grandiose | REST API |
| XStore (Event Network) | Multi-step REST |
| Shopify, Odoo, Walmart | REST API |
| TalonOne (Grandiose), Antavo (Flying Tiger) | Loyalty REST API |
| Fiskaly (VenGo, Funky Buddha) | German fiscal API |
| ZATCA/Cygnet (Kiko Milano) | Saudi fiscal API |
| MoEngage (Flying Tiger Azadea) | Marketing API |
| Kafka | Pub/sub (Flying Tiger, Funky Buddha) |
| Slack | Alerting on all failures |

---

## Order Details Retrieval (T17)

**File:** `get_order_details.py` (18K) — functions to retrieve order details for customer-facing app (order history, order summary).

### Functions

| Function | Used By | Description |
|----------|---------|-------------|
| `mishipay_get_order_details(post_data)` | ~8 legacy store types | Basic single-order detail retrieval. Uses V1 `OrderBasketDetailsSerializer`. Computes `total_save`, `percent_off`, `delivery_amount`. |
| `compass_get_order_details(post_data)` | ~58 store types (majority) | Enhanced V2 details. Uses `OrderBasketDetailsSerializerV2` + `BasketEntityAuditLogSerializerV3`. Contains store-type-specific branching for: VenGo (VAT label), Dufry (red discount), DDF (QR identifier), Riyadh Tameer (order identifier), Londis (custom VAT calc), Connexus/Fast Eddys (price+VAT adjustment). Handles `hide_price_override_labels`. |
| `mlse_get_order_details(post_data)` | MLSE only | Wraps `compass_get_order_details` then re-orders basket items to properly sequence add-on items. |
| `mishipay_get_all_order_details(post_data)` | Legacy store types | All orders for a customer. Filters by store type or white-list. Returns most recent `NO_OF_ORDERS`. |
| `compass_get_all_order_details(post_data)` | Majority of store types | Enhanced "all orders" with audit-log-based basket data. Same store-type branching as single order. Filters demo orders (excludes MishiPay Global Store at lat=0/lon=0). |
| `all_orders_add_themes(response)` | Both all-orders functions | Adds store theme/branding configs for each unique `store_id`. |
| `__trim_basket_response(order_response)` | Internal | Removes unnecessary fields from basket items (`discount_constraint`, `is_restricted`, `refunded_quantity`) to reduce payload size. |

### Key Design Notes

- All queries exclude saved-card orders: `~Q(payments__payment_method="saved_card")`.
- Demo/sandbox filtering handled: production separated from demo transactions.
- iOS compatibility hack: basket item prices cast to `float` with rounding.
- Uses `prefetch_related` for performance optimization on all-orders queries.

---

## Receipt Generation System (T17)

**File:** `receipt.py` (60K) — handles email receipt generation and sending for completed orders and refunds.

### EmailReceipt Class (`receipt.py`)

The core class for email receipt generation, handling the entire pipeline from order data extraction to email delivery.

**Constructor**: Takes `order_id`, `mail_to_entity` ("user" or "store"), `event` ("status_update" or "refund"), optional pre-loaded `order` and `refund_order`.

#### Core Methods

| Method | Purpose |
|--------|---------|
| `send_receipt(send_mail=True)` | Main orchestrator. Gets region config, builds context, renders template, sends email. Handles Click & Collect status eligibility. Skips non-prod store emails unless configured. |
| `trigger_event()` | Determines template trigger event: `"order_completed"`, `"order_completed_refund"`, or CC-status-based events (`"user_cc_confirmed"`, `"store_cc_refund"`, etc.). |
| `build_context(config_extra_data, variables)` | Dispatches to refund or receipt context builder based on event. |
| `default_context()` | Builds base context with ~40+ fields: store info, order details, payment card info, QR codes, staff details, retailer-specific overrides. |
| `get_order_items_info()` | Iterates basket items and audit logs. Builds detailed item list with prices, promos, addons, weighted items, serial numbers, and image validation (HTTP HEAD check on image URLs). |
| `get_payment_details(total, payment_framework)` | Builds legacy and new payment detail dicts. Includes card number masking, EMV terminal data (TVR, TSI, AC, CID), split payments. |
| `get_recipient_list()` | Returns email recipients: customer email for "user", store email for "store". Handles guest users. |
| `send_mail(subject, from_email, recipients, reply_to, html)` | Sends via Django `EmailMultiAlternatives` using transaction-specific email settings. Updates `receipt_status` to `SENT`. Uses `select_for_update()` for some store types. Alerts Slack on failure. |

#### Retailer-Specific Receipt Customization

| Retailer | Customization |
|----------|---------------|
| Hudson/HudsonRetail | Tax table, DCC details, Club Avolta loyalty, till number by platform |
| WHSmith | Cashier number logic (5555 for kiosk/scan-and-go, else staff/fallback 102) |
| Flying Tiger | `poslog_transaction_number` as `o_id` |
| MUJI | PSP reference as `transaction_id` |
| VenGo | Fiskaly QR code and exit QR |
| Virgin | Sales order ID, formatted `o_id`, footer banner |
| SA region | ZATCA QR value |
| Kiko Milano (SA) | Composite `transaction_id` format |
| FT Azadea (BH, KW) | APOS Plus retailer device ID mapping |
| Carter's | Loyalty points (estimated points, reward redeemed) |
| Funky Buddha | Fiscalisation `mark_id` |
| Dufry | Category-based offer label hiding |
| Decathlon (CZ) | Price-exclusive-of-tax adjustments |
| Londis/Spar/MUJI/FT | Custom VAT calculation from `vat_rate` |
| Connexus (Fast Eddys) | Price + VAT as discounted price |
| MMI | Loyalty customer name, terminal number |
| DDF | Staff details, boarding pass data, basket promos title/discount |
| Reitan | Payment method formatting (underscore → space, title case) |
| MLSE | Customized item reordering (addons) |
| SSP | Addon items hidden |
| Event Network | Donation items excluded from tax table |

#### Top-Level Functions

| Function | Purpose |
|----------|---------|
| `send_receipt_v2(store_type, region, order_id, ...)` | Entry point. Checks `GlobalConfig("NEW_EMAIL_RECEIPT_CONFIGURATION_STORES")` to decide between new `EmailReceipt` class or legacy `send_receipt`/`send_email_store` functions. |
| `calculate_item_payable_price(promo_entry, tax_amount, store)` | Per-item payable price considering tax-inclusive vs tax-exclusive pricing. |

### Receipt Types Supported

- **Order completion receipts** (user and store)
- **Refund receipts** (user and store)
- **Click & Collect status update receipts** (per-status: confirmed, ready, collected, cancelled, refund)

---

## RFID Status Tracking System (T17)

**File:** `rfid_status.py` (27K) — manages RFID EPC (Electronic Product Code) status tracking across 7 RFID vendor integrations.

### How EPC Status Tracking Works

1. Each `BasketItem` has an optional `epc` field (RFID tag identifier).
2. On sale: `set_sold_*` function called based on retailer's RFID vendor.
3. On refund: `set_unsold_*` function called with refunded EPCs list.
4. Results persisted to `basket.extra_data` or `order.extra_data` for audit trail.
5. Settings loaded from `settings.RFID_UPDATE_SETTINGS[store_type][region][demo/live]`.

### Vendor Integrations

| Vendor | Sold Function | Unsold Function | Integration |
|--------|--------------|-----------------|-------------|
| **Redis** (Generic) | `set_sold_redis(order)` — sets EPC to `1` | `set_unsold_redis(order, epc_list)` — sets to `0` | Direct Redis |
| **MishiPay Items** | `set_sold_mishipay_items(order)` — decrements `stock_quantity`, marks `is_item_sold=True`, sets Redis | `set_unsold_mishipay_items(order, epc_list)` — sets Redis unsold | Redis + Django ORM |
| **Nedap** | `set_sold_nedap(order)` — sends `disposition: "retail_sold"` | `set_unsold_nedap(order, epc_list)` — sends `disposition: "sellable_accessible"` | Nedap REST API (Bearer token, verify-after-set) |
| **SATO** | `set_sold_sato(order)` — PATCH to `set_sold_url` | `set_unsold_sato(order, epc_list)` — 2-step: PATCH `return_item_url` then `set_sellable_url` | SATO REST API (username/password auth) |
| **Decathlon** | `set_sold_decathlon(order, basket_item_ids)` — POST SGTIN list to `deactivate_epc_url`, partial item filtering | `set_unsold_decathlon(order)` — same API for unsold | Decathlon API (OAuth2 client_credentials) |
| **iLab** (Avery Dennison) | `set_sold_ilab(order, epc_list)` — SOAP `RFIDSoldTag`/`RFIDSoldTagGlobal`, updates `ItemInventory` | `set_unsold_ilab(order, epc_list)` — SOAP `RFIDReturnedTag`/`RFIDReturnedTagGlobal`, restores inventory | SOAP via zeep |
| **Retail Reload** | `set_sold_retail_reload(order)` — POST URL-encoded with `tx_id`, `epc_list` | `set_unsold_retail_reload(order, epc_list)` — same with "returned" key | REST API (form-encoded) |

### Error Handling

- Per-EPC error handling: individual EPC failures don't block others.
- Results tracked in `basket.extra_data` (iLab, Decathlon track success/failed EPC lists).
- Older code uses `print()`; newer code uses `logger.error()`.

---

## Offline Transaction Sync System (T17)

**File:** `offline_sync_with_monolith.py` (44K) — synchronizes transactions processed offline (kiosk devices that completed sales while disconnected) back to the monolith database.

### Sync Flow

```
Offline Order JSON → generate_monolith_sync_objects()
                         │ (transaction.atomic)
                         ├─ generate_user_and_customer()  → User + Customer (is_guest_user=True)
                         ├─ generate_basket()              → Basket (purchase_type by platform)
                         ├─ generate_basket_items()         → BasketItems (bulk_create)
                         ├─ generate_audit_logs()           → BasketEntityAuditLog (via store procedure)
                         ├─ generate_retailer_specific_customer_profile()
                         ├─ generate_order()                → Order (o_id, poslog, completion_date)
                         └─ generate_payment()              → Payments (method, card, PSP ref)
                         │
                    [Outside transaction]
                         │
                    If COMPLETED & poslog not sent → order.complete_status()
                         │
                    Normal fulfilment pipeline triggered
```

### Data Classes

| Class | Purpose |
|-------|---------|
| `TaxBreakup` | Tax level breakup: `retailer_tax_level_id`, `tax_level`, `taxable_amount`, `tax_amount` |
| `ItemTax` | Per-item tax: `item_id`, `total_amount`, `taxable_amount`, `tax_percent`, `tax_amount`, `tax_code` |
| `TaxServiceResponse` | Aggregates total basket tax plus lists of `TaxBreakup` and `ItemTax` |

### Key Functions

| Function | Purpose |
|----------|---------|
| `generate_monolith_sync_objects(data)` | **Main orchestrator**. Wrapped in `transaction.atomic()`. Calls all generate functions. On success, triggers `complete_status()` for completed orders. Slack alerts on failure. |
| `generate_user_and_customer(store, user_id, email, order_id)` | Creates User + Customer if not exist. Sets `is_guest_user=True`. |
| `generate_basket(store, customer, basket_details, order_id)` | Creates Basket. Sets purchase type by platform. Idempotent (skips if exists). |
| `generate_basket_items(store, basket, basket_items)` | Creates BasketItems. Deletes existing first (idempotent). Uses `bulk_create`. |
| `generate_audit_logs(store, customer, basket, basket_details)` | Creates audit logs via store-specific basket procedure from `GET_BASKET_DETAILS_FUNCTION_MAP`. Calls `apply_promotions()` + `calculate_taxes()`. |
| `generate_order(store, basket, order_details, order_status)` | Creates Order. Flying Tiger: parses `offline_receipt_reference_id` via regex for `poslog_transaction_number` and `offline_o_id`. |
| `generate_payment(store, order, order_details)` | Creates Payments. Maps payment method, card details, POS terminal info, PSP reference. |

### Error Handling

- **Atomic transactions**: all record creation rolls back on failure.
- **Idempotency**: each `generate_*` checks object exists before creating.
- **Separate try-catch for POSlog**: fulfilment failures don't roll back data sync.
- **Detailed logging**: full stack traces + Slack alerts to `#alerts_poslog`.

---

## Saved Card Order Flow (T17)

**File:** `saved_card_order.py` (9.3K) — creates orders for "saved card" transactions to tokenize a customer's payment card.

### Flow

1. Create/get `ItemCategory` with slug `saved-card-{store_id}`
2. Dispatch to PSP-specific item creator via `SAVED_CARD_ITEM_FUNCTION`
3. Create `Basket` with `is_saved_card_basket=True`
4. Create basket item with PSP-specific product
5. Dispatch to `cisco_create_order` (Cisco stores) or `mishipay_create_order` (others)

### PSP Dispatch Map (`SAVED_CARD_ITEM_FUNCTION`)

| PSP | Product Identifier | Price |
|-----|-------------------|-------|
| `adyen` | `adyen-saved-card` | 0.01 |
| `adyen_dropin` | `adyen-dropin-saved-card` | 0.00 |
| `spreedly` | `spreedly-saved-card` | 0.01 |
| `stripe` | `stripe-saved-card` | 0.01 |
| `xpay` | `xpay-saved-card` | 0.01 |
| `worldpay` | `worldpay-saved-card` | 0.00 |
| `onepay` | `onepay-saved-card` | 0.01 |
| `onepay_widget` | `onepay-widget-saved-card` | 0.00 |
| `payplus` | `payplus-saved-card` | 0.01 |
| `stripeconnect` | `stripeconnect-saved-card` | 0.00 |
| `ogone` | `ogone-saved-card` | 0.01 |

Some PSPs require a non-zero charge to tokenize (price `0.01`), others allow zero-amount. Saved card orders are filtered from normal queries via `~Q(payments__payment_method="saved_card")`.

---

## Customer Profile Updates (T17)

**File:** `customer_retail_specific_profile_dao.py` (217 lines) — updates retailer-specific customer profiles after order completion.

| Function | Retailer | Purpose |
|----------|----------|---------|
| `create_retailer_customer_profile_object(order)` | Compass (generic) | Creates/updates `RetailerSpecificCustomerProfile`. Tracks `stores_purchased_in` via `extra_data`. |
| `relay_retailer_customer_profile_update(order)` | Relay | Coffee loyalty tracking. Counts qualifying items against `coffee_incentive_promo_id` and updates past purchase records. |
| `bwg_retailer_customer_profile_update(order)` | BWG (Londis/Appleby) | **Deprecated**. Hot drink catalog tracking. |
| `retailer_customer_profile_update_new(order)` | Londis, Appleby, Goloso, Badiani, StandardCafe | Current generic implementation. Processes `basket.special_promos["programs"]` for redeemed nth-item-free incentives. |
| `get_eroski_customer_club_profile(basket, order_id)` | Eroski | **Deactivated** (returns `None`). When active: validates 5-euro coupon eligibility (spend > 20 euros), prevents duplicate usage across accounts sharing loyalty number. |
| `eroski_retailer_customer_profile_update(order)` | Eroski | Records club 5-euro coupon usage in profile `extra_data`. |

---

## Third-Party Integrations (T17)

**Directory:** `third_party_integrations/`

### Clover POS Integration (`order_fulfilment/clover.py`, 126 lines)

**`CloverOrderFulfilment`** class:
- **`__init__(order)`**: Loads settings from `settings.CLOVER_SETTINGS[store_type][region][sub_region][demo/live]`. Extracts `api_url`, `access_token`, `merchant_id`.
- **`create_order()`**: POST to `/v3/merchants/{merchant_id}/orders` with `state: "open"` and total in cents.
- **`add_line_items()`**: POST to `/v3/merchants/{merchant_id}/orders/{order_id}/bulk_line_items`. Maps items to Clover format: `clover_product_id`, price in cents, name, quantity.
- **`make_order_fulfilment_request()`**: Orchestrates create + add items. Skips line items for saved-card orders.

### Decathlon ERP Integration (`order_fulfilment/decathlon.py`, 222 lines)

**`DecathlonOrderFulfilment`** class:
- Two-level loyalty resolution: `OAuthAccount.extra_data["loyalty"]` then `ThirdPartyLogin.extra_data["loyalty_no"]`.
- Builds article list with `articleId`, `passion`, `serialNumber`, `taxes`, `discount`.
- Maps payment method to `transaction_type_id`.
- POST to fulfilment URL with `x-api-key` auth. Returns `transactionId` as `external_order_id`.

### SAP Transaction Logs (`sap/transaction_logs.py`, 866 lines)

Generates **SAP POS 2.3 format** fixed-width record files. 10 record types:

| Record | Type | Purpose |
|--------|------|---------|
| `HeaderRecord` | `0` | Transaction header: store, register, cashier, datetime, transaction number |
| `ItemRecord` | `I ` | Line item: SKU, department, taxes (1-4), quantity, prices, discounts |
| `I2Record` | `I2` | Extended item: taxes (5-16), loyalty programs, promotions |
| `I6Record` | `I6` | Foreign currency, cost, promo pricing, return identifier |
| `I8Record` | `I8` | Item flags, descriptions, group/category/vendor |
| `DiscountRecord` | `D ` | Discount: type, percentage, amount, coupon/employee flags |
| `TotalRecord` | `T ` | Transaction total: sale amount, taxes (1-4), total saved |
| `T2Record` | `T2` | Extended totals: taxes (5-16) |
| `TransactionTaxRecord` | `TX` | Per-tax-ID breakdown |
| `PaymentRecord` | `PN` | Tender: amount, currency, card token, foreign currency |

Features: fixed-width positional encoding, letter+digit discount ID scheme (A=100-109, B=110-119, ... P=250-255), `validate_discount_id()` maps string IDs to numeric codes.

---

## WhatsApp Integration (T17)

**Directory:** `whatsapp/`

### WhatsAppClient (`client.py`, 143 lines)

| Method | Purpose |
|--------|---------|
| `send_receipt(recipient)` | Hashes phone (SHA-256 with salt from `settings.WHATSAPP_CUSTOMER_NUMBER_HASH_SALT`), saves hash in `extra_data`, looks up per-store-type config (`country_code`, `template_name`, `language_code`), sends template message via Meta WhatsApp Cloud API with body params (store name, order number) and URL button (receipt link). Saves API response. |
| `get_order_number(store, order)` | Returns `o_id`; KikoMilano returns `{retailer_store_id}.SAL.{retailer_device_id}.{transaction_id_poslog}`. |
| `get_webhook_token()` / `get_app_secret()` | Returns WhatsApp webhook verification credentials. |

### OrderReceiptSerializer (`serializer.py`, 5 lines)

Simple serializer: `phone_number` (CharField), `order_id` (UUIDField).

---

## Emaar Integration (T17)

**Directory:** `emaar_utility/`

### EmaarAPIClient (`client.py`, 222 lines)

Sends daily and monthly sales reports to Emaar mall management API.

- **Config**: `GlobalConfig` key per store (via `get_client_config_key(store)`)
- **Payload**: `SalesDataCollection.SalesInfo` with `UnitNo`, `LeaseCode`, `SalesDate`, `TransactionCount`, `NetSales`. Monthly adds `SalesDateFrom`, `SalesDateTo`, `TotalSales`, `Remarks`.
- **Auth**: `x-apikey` header
- **Validation**: Slack alert to `#alerts_poslog` if required store fields missing

---

## Supporting Utilities (T17)

### Order Verification (`order_verification.py`, 11 lines)

**`mishipay_order_verification(order)`** — Minimal stub that logs an info message and returns `True` unconditionally. Used by `ORDER_VERIFICATION_FUNCTION_MAP` for `mishipaystoretype` and `dufrystoretype`. Effectively a no-op.

### Stock Quantity Update (`update_stock_quantity.py`, 26 lines)

**`update_stock_quantity(order, product_identifier_quantity_map, store)`** — Restores stock after refund/cancellation:
- RFID stores: sets `stock_quantity = 1`
- Non-RFID stores: increments by refunded quantity
- Sets `is_item_sold = False`
- Uses `print()` instead of `logger.error()` for exceptions

### Goloso POS Log (`goloso_poslog.py`, 265 lines)

Management command generating daily CSV POS log files for Goloso stores. Uses `GolosoTransactionRow` data class. Queries orders + refunds, sorts via `heapq`, processes basket audit logs for per-item pricing, marks orders with `poslog_sent` flag.

---

## Management Commands (T17)

**Directory:** `management/commands/` — **78 management command files**, the primary mechanism for batch operations.

### POS Log Generation (40+ commands)

The largest category. Each retailer has its own poslog format/command:

| Command | Retailer | Format |
|---------|----------|--------|
| `standard_poslog.py` (356 lines) | Multi-retailer (generic) | CSV with heap-sorted transactions |
| `standard_poslog_v2.py` | Generic V2 | Enhanced CSV |
| `WHSmith_Poslog_script.py` | WHSmith | Custom (2,417 lines of tests) |
| `dubai_duty_free_poslog.py` | Dubai Duty Free | Custom |
| `eventnetwork_poslog.py` | Event Network | Custom |
| `paradies_poslog.py` / `_v2.py` | Paradies | Custom |
| `loves_poslog.py` | Love's | Custom |
| `flying_tiger_poslog.py` | Flying Tiger | Custom |
| `flying_tiger_azadea_poslog.py` | Flying Tiger Azadea | Custom |
| `muji_poslog.py` | MUJI | Custom |
| `bwg_poslog.py` | BWG | Custom |
| `gmg_poslog.py` | GMG | Custom |
| `desigual_poslog.py` | Desigual | Custom |
| `emetro_poslog.py` | eMETRO | Custom |
| `virgin_poslog.py` | Virgin | Custom |
| + 25 more retailer-specific commands | | |

**Standard poslog pattern** (`standard_poslog.py`): Queries orders filtered by store/date/demo/`SUCCESS_ORDER_STATUSES` + `RefundOrderNew`. Uses `heapq` for chronological sorting. Generates CSV, bulk-updates `poslog_sent` flag. Integrates with `start_retailer_export_job`/`end_retailer_export_job` for monitoring.

### Order Fulfilment Retry (1 command)

**`order_fulfilment_retry.py`** (97 lines) — Retries failed order fulfilment for a store type within a date range. Finds captured orders without `poslog_sent`. Uses `inspect.signature()` to detect if function accepts `daily_job` parameter.

### Financial Reports (2 commands)

| Command | Purpose |
|---------|---------|
| `walmart_finance_report.py` (384 lines) | Generates Walmart EOD finance/journal entry reports. Validates discrepancies. Builds SAP-style journal entries. Outputs Excel (openpyxl) or sends via Walmart API. Manages per-store sequence numbers via `GlobalConfig`. Custom exceptions: `DiscrepancyError`, `PayloadError`, `NotZeroSumError`. |
| `settlement_report_check_discrepancy.py` (73 lines) | Compares poslog vs recon amounts, expected vs actual payout. Slack alerts to `#alerts_settlement`. |

### Customer Communications (2 commands)

| Command | Purpose |
|---------|---------|
| `send_thank_you_email.py` (285 lines) | **Deprecated**. Sends first-order thank-you emails. Deduplicates by email. Tracks in `user_metadata` and `extra_data`. |
| `customer_retention.py` (35 lines) | Exports customer email data with order counts to CSV (hardcoded 2020-04-01 to 2021-03-31). |

### Other Notable Commands

| Command | Purpose |
|---------|---------|
| `order_fulfilment_retry.py` | Retry failed fulfilment per store type/date range |
| `connexus_catalog_movement.py` | Connexus catalog movement |
| `connexus_poslog_journal.py` | Connexus poslog journal |
| `create_missing_payment_analytics.py` | Backfill missing payment analytics |
| `desigual_incentive_report.py` | Desigual incentive reporting |
| `desigual_loyalty_customer.py` | Desigual loyalty customer sync |
| `eroski_order_match.py` | Eroski order matching |
| `flash_facial_emaar_sales_report.py` | Emaar sales report generation |
| `flying_tiger_completed_to_verified.py` | Status transition for FT orders |
| `flying_tiger_eod_cron.py` | Flying Tiger end-of-day cron |
| `generic_odoo_inventory_tax_import.py` | Odoo inventory/tax import |
| `nude_foods_coupon_usage.py` | Nude Foods coupon usage tracking |
| `send_order_file_by_sftp.py` | SFTP file delivery |
| `virgin_invoice.py` | Virgin invoice generation |
| `virgin_post_transaction.py` | Virgin post-transaction processing |

---

## Background Tasks & Signals (T17)

**`tasks.py` — does not exist.** The module does not define any Celery/Dramatiq tasks directly. Background work is handled via management commands (78 total) and some async behavior through the `pubsub` module (e.g., Flying Tiger Kafka poslog via `TriggerPoslogAdapter`, BWG Dramatiq task `send_bwg_poslog_to_retailer`).

**`signals.py` — does not exist.** All post-order processing is handled imperatively within `Order.complete_status()` and `Order.complete_status_extra()`, which call dispatch maps from `app_settings.py`.

**`services/` directory — does not exist.** Business logic is organized across flat files: `create_order.py`, `complete_order.py`, `order_fulfilment.py`, `receipt.py`, `get_order_details.py`, `offline_sync_with_monolith.py`, `saved_card_order.py`, `customer_retail_specific_profile_dao.py`.

---

## Test Coverage Summary (T17)

**Directory:** `tests/` — **49 test files, 17,808 total lines of test code**.

### Test Metrics

| Metric | Value |
|--------|-------|
| Total test files | 49 |
| Total lines | 17,808 |
| Largest file | `test_WHsmith_poslog.py` (2,417 lines) |
| Framework | pytest with `@pytest.mark.django_db` |
| Factory library | `factory_boy` (`DjangoModelFactory`) |
| Mocking | `mocker.patch` (pytest-mock) |
| API testing | DRF's `APIClient` with `force_authenticate` |

### Tests by Category

| Category | Files | Lines (approx) | Coverage |
|----------|-------|-----------------|----------|
| **POS Log Generation** | 25+ | ~10,000 | WHSmith, DDF, Event Network, Paradies, GMG, Loves, MUJI, FT Azadea, Appleby, Desigual, Relay, MMI, Emma's Garden, Express Lane, Emma Sleep, Azpiral, BWG, Londis, eMETRO, SAP, Spar, Breeze Thru, Standard (V1+V2), BWG via Dramatiq |
| **Order Details & Receipts** | 8 | ~1,000 | V2, V3, Guest order details; receipt generation API; receipt banners; till numbers; staff receipts |
| **Offline Sync** | 1 | 1,308 | Full sync flow: success, bad requests, missing keys, multiple items, loyalty, idempotency, audit log tax, order status transitions |
| **Order Fulfilment** | 2 | ~870 | VenGo fulfilment, VenGo exit gate |
| **Tax Calculation** | 2 | ~600 | Legacy tax service, new ms-tax service |
| **Financial Reports** | 2 | ~1,150 | Walmart finance/journal entry, settlement discrepancy |
| **Order Utilities** | 4 | ~630 | Token generation, QR identifiers, poslog sequences, cashier numbers |
| **Integration Features** | 5 | ~1,400 | Kiosk order transfer, Vengo entry gate QR, Brevo SMS, return order info, Grandiose cancel sale |
| **Serializers** | 1 | 79 | V2 serializers |

### Factory Definition (`factories.py`)

`OrderFactory` using `factory_boy`:
- `SubFactory(BasketFactory)`, `SubFactory(StoreFactory)`
- `LazyFunction(uuid4)` for `order_id`
- `FuzzyChoice` for `order_status`

### Retailers With Dedicated Poslog Tests

At least **20 unique retailers** have dedicated test files. Heaviest coverage: WHSmith (2,417 lines), Dubai Duty Free (1,445 lines), Event Network (884 lines).
