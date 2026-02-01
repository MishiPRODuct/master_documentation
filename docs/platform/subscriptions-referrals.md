# Subscriptions & Referrals

> Last updated by: Task T15 — Referrals & Dashboard

## Overview

The `mishipay_subscription` module (37 files, ~1,740 non-migration lines) implements a **subscription-based item discount system**. Currently used for a "Coffee Subscription" product where customers pay a recurring fee (via Stripe) and receive a daily quantity of free (100% discounted) items at participating stores.

The module manages the full subscription lifecycle: checkout session creation, webhook processing for Stripe events, subscription status tracking, daily usage enforcement, and promotion generation for the discount engine.

The `mishipay_referrals` module (11 files, ~606 lines) implements a **referral campaign system** as a stateless proxy to an external Microsoft Loyalty Service. It does not store referral data locally — all campaign, code, and redemption state is managed by the loyalty microservice.

## Key Concepts

| Term | Meaning |
|------|---------|
| **Subscription** | A recurring product definition — type, price, trial period, eligible items, PSP config |
| **Coffee Subscription** | The only subscription type currently implemented — 100% discount on eligible coffee items |
| **Store Mapping** | Links a subscription to specific stores with per-store allowed items and daily quantity limits |
| **Customer Mapping** | Tracks a customer's active subscription state (TRIAL, ACTIVE, CANCELLED, EXPIRED) |
| **Usage Log** | Per-item record of subscription benefit redemption, used to enforce daily quantity limits |
| **State Log** | Audit trail of subscription state transitions |
| **MS Pay** | MishiPay's payments microservice — handles Stripe API calls on behalf of this module |
| **HMAC Webhook** | Webhook events from Stripe are relayed through MS Pay with HMAC-SHA256 authentication |
| **Referral Campaign** | A time-bound program where existing users (source) share codes with new users (target), both receiving coupon rewards |
| **Referral Code** | A unique code generated per user per campaign, shareable to invite new users |
| **Source User** | The existing customer who shares a referral code (the referrer) |
| **Target User** | The new customer who receives and redeems a referral code (the referee) |
| **MS Loyalty Service** | External microservice that stores and manages all referral campaign data, codes, and redemptions |

## Architecture

```
mishipay_subscription/
├── models.py                 # 6 Django models: subscription config + customer lifecycle
├── views.py                  # 4 REST endpoints: status, checkout, webhook, billing portal
├── urls.py                   # URL routing (all under v1/)
├── serializers.py            # 3 serializers (1 unused ModelSerializer)
├── constants.py              # Stripe event type enum + status mapping
├── admin.py                  # 6 admin registrations
├── subscription_helper.py    # Core business logic: status checks, usage tracking (199 lines)
├── ms_pay_subscription.py    # HTTP client for MS Pay subscription APIs (50 lines)
├── management/commands/
│   └── create_subscription_promo.py  # Generates promotions for subscription stores (108 lines)
└── tests/
    ├── fixtures/             # Stripe event JSON payloads
    └── test_*.py             # 5 test files + 1 empty placeholder
```

### Subscription Lifecycle Flow

```
1. SETUP (Management Command)
   create_subscription_promo → Reads Subscription.promo config
                             → Creates 100% discount promotions in promotions microservice
                             → Links promotions to eligible stores via SubscriptionStoreMapping

2. SUBSCRIBE (Customer Action)
   Customer → POST /v1/stripe/create-session/ {return_url}
           → SubscriptionServiceClient → MS Pay → Stripe Checkout Session
           → Customer completes payment in Stripe
           → Stripe fires webhook → MS Pay → POST /v1/webhook/ (HMAC-signed)
           → Creates/updates CustomerSubscriptionMapping (status: TRIAL or ACTIVE)
           → Creates CustomerSubscriptionStateLog + CustomerSubscriptionTransaction

3. SHOP (Per-Order)
   Basket evaluation → subscription_helper.get_customer_subscription_applicable_info_for_store()
                     → Checks daily usage log vs quantity_max
                     → Returns remaining discount_item_qty to promotions engine
                     → Promotion applies 100% discount on eligible items
                     → After order: log_subscription_order_items_of_customer()
                     → Creates CustomerSubscriptionUsageLog entries

4. MANAGE (Customer Action)
   Customer → POST /v1/stripe/create-billing-portal-session/ {return_url}
           → SubscriptionServiceClient → MS Pay → Stripe Billing Portal
           → Customer can cancel/update payment in Stripe portal
           → Status changes arrive via webhook

5. EXPIRE (Automatic)
   Any status check → if expiry_on < now → marks mapping as EXPIRED
                    → Trial period not offered on re-subscription
```

## Data Models

### Subscription

**File:** `mishipay_subscription/models.py:42-91`

Top-level subscription product definition.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUIDField | PK, default=uuid4 | |
| `type` | CharField(31) | choices: `COFFEE_SUBSCRIPTION` | Only one type exists |
| `name` | CharField(255) | unique | e.g., "Coffee Subscription" |
| `description` | TextField | blank, null | |
| `active` | BooleanField | default=True | |
| `trial_period_days` | PositiveSmallIntegerField | default=14 | |
| `recurring_interval` | CharField(7) | choices: DAY/WEEK/MONTH/YEAR, default=MONTH | |
| `psp_type` | CharField(63) | PSPType choices, default=STRIPECONNECT | |
| `psp_merchant_id` | CharField(63) | required | Stripe Connect merchant ID |
| `psp_allowed_items` | JSONField | see below | Stripe product/price IDs + limits |
| `currency` | CharField(4) | MS_PAY_CURRENCY_CODES, default="GBP" | |
| `promo` | JSONField | see below | Promotion engine configuration |
| `created` | DateTimeField | auto_now_add | |
| `updated` | DateTimeField | auto_now | |

#### psp_allowed_items JSON Structure

```json
{
  "products": [
    {
      "product_id": "<Stripe product ID>",
      "price_id": "<Stripe price ID>",
      "unit_price": "<price string>",
      "quantity_max": "<daily max>"
    }
  ]
}
```

#### promo JSON Structure

```json
{
  "family": "e",
  "discount_type": "p",
  "discount_value": "100",
  "group_qualifier": "coffee-subscription"
}
```

This configures a 100% percentage discount using the "Easy" promotion family, linked to the `coffee-subscription` group qualifier.

### SubscriptionStoreMapping

**File:** `mishipay_subscription/models.py:94-108`

Maps subscriptions to specific stores with per-store product lists.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `subscription` | FK → Subscription | on_delete=PROTECT | |
| `store` | FK → Store | on_delete=PROTECT | From `dos.models` |
| `allowed_items` | JSONField | default structure | Per-store product list with `quantity_max` |
| `created` | DateTimeField | auto_now_add | |
| `updated` | DateTimeField | auto_now | |

The `allowed_items` JSON includes per-store product identifiers:
```json
{
  "products": [
    {
      "product_identifier": "<product ID>",
      "quantity_max": "<daily limit>"
    }
  ]
}
```

### CustomerSubscriptionMapping

**File:** `mishipay_subscription/models.py:111-129`

Tracks a customer's active subscription status. One record per customer+subscription pair.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `customer` | FK → Customer | on_delete=PROTECT | |
| `subscription` | FK → Subscription | on_delete=PROTECT | |
| `psp_customer_id` | CharField(63) | required | Stripe customer ID |
| `psp_response` | JSONField | blank, null | Full Stripe subscription response |
| `status` | CharField(31) | choices: TRIAL/ACTIVE/CANCELLED/EXPIRED, default=TRIAL | |
| `expiry_on` | DateTimeField | db_index | |
| `created` | DateTimeField | auto_now_add | |
| `updated` | DateTimeField | auto_now | |

**Constraint:** `unique_together = ('customer', 'subscription')` — one mapping per customer per subscription.

#### Status Lifecycle

```
OPEN_FOR_TRIAL → TRIAL → ACTIVE → CANCELLED
                   │        │         ↑
                   ↓        ↓         │
                EXPIRED  EXPIRED   EXPIRED
                              (can re-subscribe with trial_period_days=0)
```

- `OPEN_FOR_TRIAL` — Not stored in DB; returned when no mapping exists.
- `TRIAL` — Set via webhook when Stripe status is "trialing".
- `ACTIVE` — Set via webhook when Stripe status is "active".
- `CANCELLED` — Set via webhook when Stripe status is "canceled".
- `EXPIRED` — Set by status check when `expiry_on < now`.
- `NOT_SUPPORTED_FOR_STORE` — Returned when store has no `SubscriptionStoreMapping`.

### CustomerSubscriptionTransaction

**File:** `mishipay_subscription/models.py:131-143`

Records payment transactions (invoices) for subscriptions.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUIDField | PK, default=uuid4 | |
| `customer` | FK → Customer | on_delete=PROTECT | |
| `subscription` | FK → Subscription | on_delete=PROTECT | |
| `psp_type` | CharField(63) | default=STRIPECONNECT | |
| `psp_amount` | DecimalField(15,2) | default=0.0 | Amount in currency units (not cents) |
| `psp_subscription_id` | CharField(63) | blank, null, db_index | Stripe subscription ID |
| `psp_invoice_id` | CharField(63) | blank, null, db_index | Stripe invoice ID |
| `psp_charge_id` | CharField(63) | blank, null, db_index | Stripe charge ID |
| `psp_response` | JSONField | blank, null | Full Stripe invoice response |
| `completed_on` | DateTimeField | null, blank, db_index | |
| `created` | DateTimeField | auto_now_add | |
| `updated` | DateTimeField | auto_now | |

### CustomerSubscriptionStateLog

**File:** `mishipay_subscription/models.py:145-161`

Audit log for subscription state changes. Created on every webhook event.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUIDField | PK, default=uuid4 | |
| `customer` | FK → Customer | on_delete=PROTECT | |
| `subscription` | FK → Subscription | on_delete=PROTECT | |
| `psp_type` | CharField(63) | default=STRIPECONNECT | |
| `psp_subscription_id` | CharField(63) | blank, null, db_index | |
| `psp_response` | JSONField | blank, null | Full webhook payload |
| `status` | CharField(31) | choices: TRIAL/SUCCESS/FAILED/CANCELLED | |
| `completed_on` | DateTimeField | null, blank, db_index | |
| `created` | DateTimeField | auto_now_add | |
| `updated` | DateTimeField | auto_now | |

### CustomerSubscriptionUsageLog

**File:** `mishipay_subscription/models.py:164-184`

Tracks individual item-level usage of subscription benefits (e.g., each free coffee redeemed).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `subscription` | FK → Subscription | on_delete=PROTECT | |
| `customer` | FK → Customer | on_delete=PROTECT | |
| `store` | FK → Store | on_delete=PROTECT | |
| `order_type` | CharField(7) | choices: ORDER/REFUND, default=ORDER | |
| `item_type` | CharField(15) | BASKET_ITEM_TYPE_CHOICES, blank, null | REGULAR, WITH_ADDONS, etc. |
| `order` | FK → Order | on_delete=PROTECT | |
| `order_max_subscription_discount_qty` | SmallIntegerField | default=0 | Max discountable qty for this order |
| `refund_order` | FK → RefundOrderNew | on_delete=PROTECT, blank, null | |
| `product_identifier` | CharField(63) | | |
| `retailer_product_id` | CharField(63) | | |
| `mrp` | DecimalField(15,2) | default=0.0 | Original price |
| `sale_price` | DecimalField(15,2) | default=0.0 | |
| `final_price` | DecimalField(15,2) | default=0.0 | After subscription discount |
| `quantity` | PositiveSmallIntegerField | default=1 | |
| `completed_on` | DateTimeField | auto_now_add | |
| `updated` | DateTimeField | auto_now | |

## API Endpoints

**File:** `mishipay_subscription/urls.py`, `mishipay_subscription/views.py` (238 lines)

| Method | Path | View | Auth | Purpose |
|--------|------|------|------|---------|
| GET | `v1/customer-details/` | `CustomerSubscriptionDetails` | IsAuthenticated | Returns customer subscription status and message |
| POST | `v1/stripe/create-session/` | `CreateStripeSubscriptionSession` | IsAuthenticated | Creates Stripe Checkout session for subscribing |
| POST | `v1/webhook/` | `SubscriptionWebhook` | AllowAny (HMAC) | Receives subscription/invoice events from MS Pay |
| POST | `v1/stripe/create-billing-portal-session/` | `CreateStripeBillingPortalSession` | IsAuthenticated | Creates Stripe Billing Portal for subscription management |

### GET v1/customer-details/

**View:** `CustomerSubscriptionDetails` (`views.py:42-57`)

Returns the subscription status for the authenticated customer.

**Query params:** `type` (optional, defaults to `COFFEE_SUBSCRIPTION`)

**Response:** `{"status": "<STATUS>", "detail": "<human-readable message>"}`

Possible statuses: `OPEN_FOR_TRIAL`, `TRIAL`, `ACTIVE`, `CANCELLED`, `EXPIRED`, `NOT_SUPPORTED_FOR_STORE`.

### POST v1/stripe/create-session/

**View:** `CreateStripeSubscriptionSession` (`views.py:60-108`)

Creates a Stripe Checkout session for subscribing.

**Request body:** `{"return_url": "<URL>"}`

**Logic:**
1. Looks up `Subscription` with `name="Coffee Subscription"` and `active=True`. Returns 404 if not found.
2. Checks existing `CustomerSubscriptionMapping`:
   - If EXPIRED (not CANCELLED): marks as EXPIRED, sets `trial_period_days=0` (no re-trial).
   - If TRIAL or ACTIVE: returns 400 "already in active or trial state".
   - If CANCELLED or not found: proceeds with checkout.
3. Delegates to `SubscriptionServiceClient.create_stripe_subscription_session()`.

### POST v1/webhook/

**View:** `SubscriptionWebhook` (`views.py:111-207`)

Receives webhook events from MS Pay (not directly from Stripe). Publicly accessible but secured via HMAC-SHA256.

**Security:** HMAC-SHA256 signature verification using `settings.MPAY_HMAC` as shared secret. The `hmac` field is popped from the payload before signature computation. Payload is JSON-sorted for deterministic signing.

**Handles two notification types:**

| Type | Action |
|------|--------|
| `SUBSCRIPTION` | Creates/updates `CustomerSubscriptionMapping` with mapped status. Creates `CustomerSubscriptionStateLog`. Maps Stripe statuses: `"trialing"` → TRIAL, `"active"` → ACTIVE, `"canceled"` → CANCELLED. |
| `INVOICE` | Creates `CustomerSubscriptionTransaction` with `psp_amount = amount_paid / 100` (cents to currency). |

### POST v1/stripe/create-billing-portal-session/

**View:** `CreateStripeBillingPortalSession` (`views.py:210-238`)

Creates a Stripe Billing Portal session so customers can cancel or update payment.

**Request body:** `{"return_url": "<URL>"}`

Looks up the customer's `CustomerSubscriptionMapping` for "Coffee Subscription", then delegates to `SubscriptionServiceClient.create_stripe_billing_portal_session()`.

## Business Logic

### Subscription Status Checks

**File:** `mishipay_subscription/subscription_helper.py:24-118`

#### get_customer_subscription_status(customer, type)

Returns `(status_string, human_readable_message)`. Automatically marks expired subscriptions.

#### is_customer_subscription_active(customer, type)

Returns `True` if status is TRIAL or ACTIVE.

#### get_customer_subscription_status_for_store(customer, store_id, type)

First checks `SubscriptionStoreMapping` for the store. Returns `NOT_SUPPORTED_FOR_STORE` if no mapping exists. Otherwise returns detailed `subscription_info` dict:
```python
{
    "id": subscription.id,
    "name": subscription.name,
    "promo_group_qualifier_id": subscription.promo["group_qualifier"],
    "psp_type": subscription.psp_type,
    "psp_merchant_id": subscription.psp_merchant_id,
    "allowed_items": store_mapping.allowed_items,
    ...
}
```

### Daily Usage Tracking

**File:** `mishipay_subscription/subscription_helper.py:121-156`

#### get_customer_subscription_applicable_info_for_store(customer, store_id, type, basket)

The most complex function. Determines how many discounted items a subscriber can still receive today.

**Algorithm:**
1. Gets customer subscription status for the store.
2. If active/trial, retrieves `quantity_max` from `allowed_items`.
3. Queries `CustomerSubscriptionUsageLog` for today's date using `.iterator()` for memory efficiency.
4. Iterates usage logs:
   - Skips refunded orders with status `rejected_store` or `not_collected`.
   - For REGULAR/WITH_ADDONS items in ORDER type: adds `min(order_max_subscription_discount_qty, quantity)` to used count.
   - For REFUND type: decrements used count by 1.
5. Returns `(True, {"discount_item_qty": max(0, quantity_max - used_count), ...})` or `(False, 0)`.

### Order Completion Logging

**File:** `mishipay_subscription/subscription_helper.py:159-199`

#### log_subscription_order_items_of_customer(order)

Called after order completion to record subscription usage.

**Logic:**
1. Checks if store type is excluded from basket audit logs (`settings.BASKET_AUDIT_LOG_NOT_CREATED_STORETYPES`).
2. Reads `basket.special_promos` JSON field for `COFFEE_SUBSCRIPTION` program entries.
3. Iterates `BasketEntityAuditLog` entries matching the basket.
4. Matches items by `retailer_code` prefix against the `promo_group_qualifier_id`.
5. Creates `CustomerSubscriptionUsageLog` entries via `bulk_create()`.

### Service Client

**File:** `mishipay_subscription/ms_pay_subscription.py` (50 lines)

#### SubscriptionServiceClient

HTTP client that communicates with the MishiPay payments microservice. Uses `requests.post()` synchronously.

| Method | External URL Setting | Purpose |
|--------|---------------------|---------|
| `create_stripe_subscription_session()` | `settings.CREATE_STRIPE_SUBSCRIPTION_SESSION_URL` | Create Stripe Checkout session. Converts price to cents (`int(Decimal(unit_price) * 100)`). |
| `create_stripe_billing_portal_session()` | `settings.CREATE_BILLING_PORTAL_SESSION_URL` | Create Stripe Billing Portal session. |

Both return `(response_json, status_code)` tuples. No error handling, timeouts, or retry logic.

## Management Command

### create_subscription_promo

**File:** `mishipay_subscription/management/commands/create_subscription_promo.py` (108 lines)

**Usage:**
```bash
python3 manage.py create_subscription_promo --subscription_id "<UUID>" --store_types GolosoStoreType BadianiStoreType
```

Generates 100% discount promotions in the promotions microservice for each store mapped to the subscription.

**Algorithm:**
1. Queries `SubscriptionStoreMapping` for the given subscription.
2. For each store matching the provided `--store_types`:
   - Reads promo config from `subscription.promo` JSON.
   - Constructs a full promotion definition:
     - `family: "e"` (Easy), `evaluate_criteria: "p"`, `discount_apply_type: "p"` (POSLOG exclusion)
     - `availability: "s"` (store-specific)
     - `special_promo_info` with `group_qualifier_id`
     - `promo_groups` with nodes built from `allowed_items.products`
   - Uses `is_store_type_promo_uses_retailer_id()` to determine node ID type.
   - Calls promotions microservice: `Promotion.from_json()` → `PromotionBatchOperation.delete()` → `.create()` → `.commit()`.

This command is the bridge between the subscription module and the [Promotions Service](../promotions-service/overview.md).

## Serializers

**File:** `mishipay_subscription/serializers.py` (15 lines)

| Serializer | Type | Fields | Used By |
|-----------|------|--------|---------|
| `SubscriptionSerializer` | ModelSerializer | Meta.model=Subscription, **no `fields` specified** | Unused — missing `fields` attribute |
| `StripeSubscriptionSerializer` | Serializer | `return_url: CharField` | `CreateStripeSubscriptionSession` view |
| `StripeBillingPortalSerializer` | Serializer | `return_url: CharField` | `CreateStripeBillingPortalSession` view |

## Constants

**File:** `mishipay_subscription/constants.py` (14 lines)

### STRIPE_EVENT_NOTIFICATION_TYPE_ENUM

```python
class STRIPE_EVENT_NOTIFICATION_TYPE_ENUM(str, Enum):
    SUBSCRIPTION = "SUBSCRIPTION"
    INVOICE = "INVOICE"
```

### STRIPE_SUBSCRIPTION_STATUS_MAPPING

Maps Stripe subscription statuses to internal statuses:

| Stripe Status | Internal Status |
|--------------|-----------------|
| `"trialing"` | TRIAL |
| `"active"` | ACTIVE |
| `"canceled"` | CANCELLED |

Any unmapped status defaults to CANCELLED (in views) or FAILED (in state logs).

## Admin Interface

**File:** `mishipay_subscription/admin.py` (91 lines)

All 6 models are registered:

| Admin Class | Model | Key list_display | list_filter |
|------------|-------|------------------|-------------|
| `SubscriptionAdmin` | Subscription | name, active, psp_type, psp_merchant_id | type, psp_type |
| `SubscriptionStoreMappingAdmin` | SubscriptionStoreMapping | subscription, store | — |
| `CustomerSubscriptionMappingAdmin` | CustomerSubscriptionMapping | customer, psp_customer_id, status, expiry_on | status |
| `CustomerSubscriptionTransactionAdmin` | CustomerSubscriptionTransaction | psp_subscription_id, psp_invoice_id, psp_charge_id | psp_type |
| `CustomerSubscriptionStateLogAdmin` | CustomerSubscriptionStateLog | customer, psp_subscription_id, completed_on | psp_type |
| `CustomerSubscriptionUsageLogAdmin` | CustomerSubscriptionUsageLog | customer, order_id, product_identifier, mrp, final_price, quantity | item_type, completed_on |

Several admin classes use `autocomplete_fields = ("customer",)` and `autocomplete_fields = ("store",)`.

## Dependencies

### Internal Dependencies

| Dependency | Used For |
|-----------|----------|
| `dos.models.Store` | Store model for store mappings |
| `dos.models.Customer` | Customer model for subscription mappings |
| `mishipay_retail_orders.models.Order` | Order references in usage logs |
| `mishipay_dashboard.models.RefundOrderNew` | Refund references in usage logs |
| `mishipay_retail_payments.PSPType` | PSP type constants |
| `mishipay_retail_payments.MS_PAY_CURRENCY_CODES` | Currency code choices |
| `mishipay_items.constants` | `BASKET_ITEM_TYPE_CHOICES`, `REGULAR`, `WITH_ADDONS` |
| `mishipay_items.models.BasketEntityAuditLog` | Audit logs for order completion tracking |
| `mishipay_items.mpay_promo` | `Promotion`, `PromotionBatchOperation` — promotions microservice client |
| `mishipay_core.common_functions` | `get_rounded_value`, `is_store_type_promo_uses_retailer_id` |

### External Services

| Service | Integration Method | Purpose |
|---------|-------------------|---------|
| MS Pay Microservice | HTTP via `settings.CREATE_STRIPE_SUBSCRIPTION_SESSION_URL`, `settings.CREATE_BILLING_PORTAL_SESSION_URL` | Stripe Checkout & Billing Portal session creation |
| Promotions Microservice | Via `mishipay_items.mpay_promo` | Creating 100% discount promotions for subscribed stores |

### Configuration Settings

| Setting | Purpose |
|---------|---------|
| `settings.MPAY_HMAC` | Shared secret for HMAC-SHA256 webhook verification |
| `settings.CREATE_STRIPE_SUBSCRIPTION_SESSION_URL` | MS Pay endpoint for checkout sessions |
| `settings.CREATE_BILLING_PORTAL_SESSION_URL` | MS Pay endpoint for billing portal sessions |
| `settings.BASKET_AUDIT_LOG_NOT_CREATED_STORETYPES` | Store types excluded from usage logging |

## Schema Evolution

18 migrations from November 2021 through October 2023:

| Migration | Key Change |
|-----------|-----------|
| `0001_initial` | Creates all 6 models |
| `0002-0004` | Early field adjustments |
| `0005` | Adds `currency` field to Subscription |
| `0006` | Adds `promo_group_qualifier_id` field |
| `0007-0009` | Additional adjustments |
| `0010-0011` | November 2021 changes |
| `0012-0014` | 2022 changes |
| `0015` | Alters `CustomerSubscriptionMapping.psp_response` and related |
| `0016` | Alters `CustomerSubscriptionStateLog.psp_type` and related |
| `0017` | Merge migration (October 2023) |
| `0018` | Post-merge state log alterations |

## Test Coverage

**5 test files** + 1 empty placeholder + 2 JSON fixture files:

| Test File | Tests | Focus |
|-----------|-------|-------|
| `test_customer_subscription_details_api_view.py` | 1 | Expired subscription detection |
| `test_create_stripe_subscription_session_api_view.py` | 7 | Session creation with various subscription states |
| `test_create_stripe_billing_portal_session_api_view.py` | 4 | Billing portal session creation |
| `test_create_subscription_webhook_api_view.py` | 4 | HMAC validation + subscription/invoice event processing |
| `test_ms_pay_subscription.py` | 2 | Service client HTTP calls |
| `test_subscription_flow_with_item_discount.py` | 0 | **Empty** — placeholder for integration tests |

**Test fixtures:**
- `subscription_event.json` (128 lines): Full Stripe subscription object — `status: "trialing"`, product `prod_Kcec5uRJn3pgo0`, amount 1000 (10.00 GBP).
- `invoice_event.json` (176 lines): Full Stripe invoice object — `amount_paid: 1000`, `billing_reason: "subscription_create"`, `account_name: "MishiPay Coffee"`.

Tests use DRF `APITestCase` with `Token` auth and `unittest.mock.patch` for external service mocking. Webhook tests manually compute HMAC signatures to validate the security mechanism.

## Notable Patterns

1. **Indirect Stripe integration** — The module never calls Stripe APIs directly. All PSP communication goes through the MS Pay microservice. Webhooks from Stripe are relayed through MS Pay with HMAC authentication.

2. **Hardcoded subscription name** — `"Coffee Subscription"` is hardcoded in `views.py:68` and `views.py:223`. Only one subscription product is supported without code changes.

3. **Promotion engine bridge** — The `create_subscription_promo` management command creates 100% discount "Easy" family promotions in the promotions microservice, connecting subscriptions to the discount engine documented in [Promotions Service — Business Logic](../promotions-service/business-logic.md).

4. **Daily usage enforcement** — The system tracks per-day item usage to enforce a `quantity_max` limit. Uses `.iterator()` for memory efficiency on potentially large result sets.

5. **All models use `on_delete=PROTECT`** — No cascade deletes; all related records must be explicitly removed.

6. **No background tasks or signals** — All processing is synchronous. `log_subscription_order_items_of_customer()` is called from the order completion flow.

7. **Trial once** — When an expired subscription is detected during re-subscription, `trial_period_days` is set to 0 (no re-trial).

## Open Questions

- `"Coffee Subscription"` is hardcoded in views.py. Is there a plan to support multiple subscription types, or is this intentionally a single-product feature?
- `SubscriptionSerializer` (ModelSerializer) lacks a `fields` attribute and is not imported anywhere. Is this dead code?
- `datetime.fromtimestamp()` is called without timezone in the webhook handler (`views.py:166`). Should this use `datetime.fromtimestamp(ts, tz=timezone.utc)` for consistency?
- `Decimal(data["data"]["amount_paid"] / 100)` in `views.py:199` — integer division occurs before Decimal conversion, losing precision. Should be `Decimal(data["data"]["amount_paid"]) / Decimal(100)`.
- `subscription_helper.py` uses bare `except Exception` (lines 46, 100) which silently swallows all errors and defaults to `OPEN_FOR_TRIAL`. Should this be narrowed to `CustomerSubscriptionMapping.DoesNotExist`?
- `SubscriptionServiceClient` has no timeout, retry, or error handling on `requests.post()` calls. Is there retry logic at the MS Pay service level?
- The empty `test_subscription_flow_with_item_discount.py` suggests planned integration tests for the discount flow. Is this still planned?
- Two different `Customer` model imports appear across test files: `dos.models.customer.Customer` vs `mishipay_retail_customers.models.Customer`. Are these the same model re-exported, or is there an inconsistency?

---

# Referral Program

> Added by: Task T15 — Referrals & Dashboard

## Overview

The `mishipay_referrals` module (11 files, ~606 lines) implements a **referral campaign management system** using the **proxy/gateway pattern**. It does not store any referral data locally in the database — instead, it acts as a thin API layer that forwards all requests to an external Microsoft Loyalty Service microservice.

The referral flow allows existing customers (source users) to share unique codes with new customers (target users). When a target user redeems a code, both parties receive coupon rewards (e.g., "Give £2.50, get £2.50").

## Architecture

```
mishipay_referrals/
├── models.py               # Empty — no local models (stateless proxy)
├── urls.py                 # 6 URL routes (15 lines)
├── views.py                # 6 API view classes (126 lines)
├── referral_helper.py      # 2 helper functions (18 lines)
├── admin.py                # Empty — no models to register
├── apps.py                 # App config (5 lines)
├── migrations/__init__.py  # Empty (no migrations needed)
└── tests/
    ├── test_referral_code.py  # Code generation tests (140 lines)
    └── test_referral_flow.py  # End-to-end flow tests (296 lines)
```

### Proxy Pattern

All referral data lives in the external **Microsoft Loyalty Service** (accessed via `settings.MS_LOYALTY_SERVICE`). This module provides:

1. **Authentication** — All endpoints require `IsAuthenticated`
2. **Request routing** — Forwards to corresponding loyalty service endpoints
3. **Retailer derivation** — Auto-derives retailer from store data when not provided
4. **Response wrapping** — Uses `get_generic_response_structure()` for consistent format

```
Client → mishipay_referrals API → MS Loyalty Service
                                     ↕
                              (stores all data)
```

## Data Models

**None.** The `models.py` file is empty (commented-out boilerplate). All referral campaign, code, and redemption data is stored in the external loyalty service.

## API Endpoints

**File:** `mishipay_referrals/urls.py` (15 lines), `mishipay_referrals/views.py` (126 lines)

All endpoints require `IsAuthenticated` permission.

| Method | Path | View | Purpose |
|--------|------|------|---------|
| POST | `v1/referral_campaign/` | `ReferralCampaign` | Create a new referral campaign |
| GET | `v1/referral_campaign_latest/` | `ReferralCampaignLatest` | Get latest campaign for a retailer |
| GET | `v1/referral_campaign_latest_all_retailers/` | `ReferralCampaignLatestOfAllRetailers` | Get campaigns across all retailers |
| GET | `v1/referral_code/` | `ReferralCode` | Generate/retrieve a referral code for a user |
| POST | `v1/referral_code_validate/` | `ReferralCodeValidate` | Validate a referral code |
| POST | `v1/referral_code_redeem/` | `ReferralCodeRedeem` | Redeem a referral code |

### POST v1/referral_campaign/

**View:** `ReferralCampaign` (`views.py:18-30`)

Creates a new referral campaign in the loyalty service.

**Request body fields** (from test data):
- `retailer`: Chain-region format, e.g., `"MUJI-GB"`
- `program_title`: e.g., `"MUJI Referral: Give £2.50, get £2.50"`
- `program_description`: Campaign description text
- `is_active`: Boolean
- `start_date_time`, `end_date_time`: ISO 8601 timestamps
- `source_max_user_use_count`, `target_max_user_use_count`: Usage limits per user
- `source_user_coupon_title`, `target_user_coupon_title`: Coupon display names
- `app_redirect_links`: `{ "android": "<url>", "ios": "<url>", "web": "<url>" }`
- `terms_and_condtions_link`: URL (note: typo — missing 'i' in "conditions")

Forwards to: `{MS_LOYALTY_SERVICE}/referral/v1/referral_campaign/`

### GET v1/referral_campaign_latest/

**View:** `ReferralCampaignLatest` (`views.py:33-45`)

Retrieves the latest active campaign for a specific retailer.

**Query params:** `retailer` (required, e.g., `"MUJI-GB"`)

Forwards to: `{MS_LOYALTY_SERVICE}/referral/v1/referral_campaign_latest/?retailer={retailer}`

### GET v1/referral_campaign_latest_all_retailers/

**View:** `ReferralCampaignLatestOfAllRetailers` (`views.py:48-73`)

Retrieves campaigns across all retailers, optionally filtered by region.

**Query params:** `region` (optional)

**Timeout:** 30 seconds (the only endpoint with an explicit timeout)

**Response transformation:**
```python
{
    "campaigns": [...campaign data...],
    "message": "status message"
}
```

Returns `"NO_ACTIVE_REFERRAL_CAMPAIGN"` error if loyalty service returns non-200.

### GET v1/referral_code/

**View:** `ReferralCode` (`views.py:76-102`)

Generates or retrieves a unique referral code for a user. Codes are idempotent — the same user always receives the same code for a given campaign.

**Query params:**
- `user_id_third_party`: Customer ID (required)
- `retailer` OR `store_id`: Retailer identifier (one required)
- `app_source`: `"android"` or `"ios"`

**Smart retailer derivation:** If `retailer` is not provided, the view:
1. Looks up `Store` by `store_id` (via `dos.models.Store`)
2. Derives retailer as `{store.chain.upper()}-{store.region.upper()}`
3. Example: store with chain="MUJI", region="GB" → `"MUJI-GB"`

Returns `"NO_ACTIVE_REFERRAL_CAMPAIGN"` (400) if no active campaigns exist.

### POST v1/referral_code_validate/

**View:** `ReferralCodeValidate` (`views.py:105-114`)

Validates a referral code before redemption. Delegates to `validate_referral_code()` helper.

### POST v1/referral_code_redeem/

**View:** `ReferralCodeRedeem` (`views.py:117-126`)

Redeems a referral code, granting discounts/coupons to both source and target users. Delegates to `redeem_referral_code()` helper.

## Business Logic

### Helper Functions

**File:** `mishipay_referrals/referral_helper.py` (18 lines)

Two thin wrapper functions that forward requests to the loyalty service:

| Function | Loyalty Service Endpoint | Purpose |
|----------|------------------------|---------|
| `validate_referral_code(request_data)` | POST `/referral/v1/referral_code_validate/` | Validates a referral code |
| `redeem_referral_code(request_data)` | POST `/referral/v1/referral_code_redeem/` | Redeems a referral code |

Both return the raw `requests.Response` object. No local validation logic is performed.

### Referral Flow

```
1. CAMPAIGN CREATION (Admin)
   POST /v1/referral_campaign/ → Loyalty Service creates campaign
     - Defines: retailer, rewards, limits, dates, app links

2. CODE GENERATION (Source User)
   GET /v1/referral_code/?user_id_third_party=<id>&retailer=<r>&app_source=<s>
     → Loyalty Service generates/returns unique code
     → Same user always gets same code (idempotent)

3. CODE SHARING (Source → Target)
   Source user shares code via app redirect links (android/ios/web)

4. CODE VALIDATION (Target User at Checkout)
   POST /v1/referral_code_validate/ {code, user_id, ...}
     → Loyalty Service validates code eligibility
     → Returns: status=true, "Validated succesfully" [sic]

5. CODE REDEMPTION (Target User at Checkout)
   POST /v1/referral_code_redeem/ {code, user_id, ...}
     → Target user gets discount coupon
     → Source user gets reward coupon generated automatically
     → Returns: "Referral Coupon redeemed. Coupon for source user is generated."

6. COUPON REDEMPTION (Both Users)
   Coupons managed by loyalty service, redeemed via mishipay_coupons module
   See [Coupons & Promotions](./coupons-promotions.md)
```

## Dependencies

### Internal Dependencies

| Dependency | Used For |
|-----------|----------|
| `dos.models.Store` | Retailer derivation from store_id in `ReferralCode` view |
| `mishipay_core.common_functions.get_generic_response_structure()` | Consistent response wrapping |

### External Services

| Service | Integration | Purpose |
|---------|------------|---------|
| MS Loyalty Service | HTTP via `settings.MS_LOYALTY_SERVICE` | All referral data storage and business logic |

**Loyalty service endpoints called:**
- POST `/referral/v1/referral_campaign/`
- GET `/referral/v1/referral_campaign_latest/`
- GET `/referral/v1/referral_campaign_latest_all_retailers/`
- GET `/referral/v1/referral_code/`
- POST `/referral/v1/referral_code_validate/`
- POST `/referral/v1/referral_code_redeem/`

## Test Coverage

**2 test files, 436 lines total**

| Test File | Lines | Tests | Focus |
|-----------|-------|-------|-------|
| `test_referral_code.py` | 140 | 1 method (5 scenarios) | Campaign creation, code generation, code idempotency, code uniqueness per user |
| `test_referral_flow.py` | 296 | 1 method (8 scenarios) | Full E2E: campaign → code → validate → redeem → coupon generation → coupon retrieval → coupon redemption |

Both use `MPTestCase` base class with MUJI-GB test retailer. `test_referral_flow.py` also tests integration with the promotion batch operation and coupon redemption systems.

## Notable Patterns

1. **Stateless proxy** — No local database models. All referral state is externalized to the loyalty microservice. This keeps the module simple but creates a hard dependency on service availability.

2. **Smart retailer derivation** — `ReferralCode` view can derive the retailer string from a `store_id` by looking up the store's chain and region. This avoids requiring the client to know the retailer format.

3. **Inconsistent timeout handling** — Only `ReferralCampaignLatestOfAllRetailers` has a 30-second timeout. All other endpoints make `requests.get()`/`requests.post()` calls with no timeout, risking indefinite hangs.

4. **No error handling for network failures** — Views don't catch `requests.RequestException` or connection errors. A loyalty service outage would cause unhandled 500 errors.

5. **No input validation** — Request data is forwarded directly to the loyalty service without local schema validation.

6. **No logging** — No logging of referral activities. Debugging issues requires checking the loyalty service logs.

7. **Field name typo** — `terms_and_condtions_link` (missing 'i' in "conditions") is present in both the API contract and tests. Likely a legacy typo inherited from the loyalty service API.

## Open Questions

- What is the structure and deployment of the MS Loyalty Service? Is it a shared service or MishiPay-specific?
- How are referral coupons integrated with the promotions microservice? The test shows integration with `PromotionBatchOperation` — does the loyalty service create promotions directly?
- Is there a maximum number of referral codes per campaign? The `source_max_user_use_count` and `target_max_user_use_count` limit per-user usage, but overall campaign limits are unclear.
- Are referral campaigns actively used in production? The test data uses MUJI-GB only.
- Why do `validate_referral_code()` and `redeem_referral_code()` exist as separate helper functions in `referral_helper.py` when they are simple one-line pass-throughs?
