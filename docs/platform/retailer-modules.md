# Retailer Modules — DOS Integration

> Last updated by: Task T26 — Retailer-specific modules (Dixons, Decathlon, Leroy Merlin, Cisco, Lego, Mango)

## Overview

The `dos` module (located at `mainServer/dos/`) is the **foundational retail integration layer** of the MishiPay platform. "DOS" historically stands for "Deloitte Operating System" — reflecting the module's origins as MishiPay's first deployment partner. Despite the name, it now serves as the **core application** powering all retailer integrations, containing the primary data models (Store, Customer, Order, Item), the agent/item inheritance hierarchy for retailer-specific behaviour, payment gateway integrations, MQTT-based real-time notifications, and shared utilities used across the entire platform.

The DOS module contains ~363 files and is the largest module in the codebase. This Part 1 documents the data models, core logic, and architectural patterns. Part 2 (T24) will cover views, APIs, and URL routing.

## Key Concepts

- **Store Type**: Each retailer gets a `StoreType` subclass that defines how items are fetched, sold, and paid for. The store's `store_type` string field maps to a class via `STORE_TYPE_MAP`.
- **Agent Pattern**: Legacy mechanism for retailer-specific item operations. `BaseAgent` defines abstract methods; subclasses (MishipayAgent, CamaieuAgent, etc.) implement retailer-specific item lookup, sold-status tracking, and inventory management.
- **SecureItem**: Items with a `uid` field (EPC/RFID tag identifier) enabling anti-theft tracking via Redis and MQTT.
- **Geofencing**: Distance-based access control ensuring customers are physically near a store before scanning/purchasing.

## Architecture

### Module Structure

```
dos/
├── models/
│   ├── __init__.py          # Re-exports: BaseAgent, SecureItem, Store, Customer, Order, Item, etc.
│   ├── store.py             # Store model (primary config entity)
│   ├── customer.py          # Customer model
│   ├── order.py             # Order model + OrderCreateForm
│   ├── item.py              # Base Item model
│   ├── choices.py           # Payment platform choices
│   ├── serializer.py        # ModelSerializer model (stores serialized data)
│   ├── transaction.py       # TransactionHistory model
│   ├── coupon.py            # Coupon model
│   ├── device.py            # Device model (for push notifications)
│   ├── feedback.py          # Feedback model
│   ├── gate.py              # Gate model (exit gates)
│   ├── app_specific_models.py  # AppVersionDetails model
│   ├── base/
│   │   ├── base_agent.py    # BaseAgent abstract class
│   │   └── base_item.py     # SecureItem base model
│   ├── mishipay/            # MishiPay-specific agent & item
│   ├── camaieu/             # Camaieu-specific agent & item
│   ├── clerk_and_green/     # Clerk & Green agent & item
│   ├── ilab/                # iLab agent (external API)
│   └── leroy_merlin/        # Leroy Merlin agent (external API)
├── base_store_type.py       # StoreType base class (abstract interface)
├── store_map.py             # STORE_TYPE_MAP — maps 100+ retailer names to classes
├── serializers.py           # DRF serializers for Customer, Store, Order, etc.
├── payment.py               # Payment gateway classes (PayUnity, PayPal, Stripe)
├── order.py                 # OrderHelper — order creation logic
├── util.py                  # Shared utilities (MQTT, email, receipts, geofencing)
├── db.py                    # DatabaseDao — raw SQL helper
├── permissions.py           # DRF permissions (IsBlockedUser)
├── decorators.py            # @authenticate decorator
├── mqtt.py                  # MQTT client for SATO/Cisco Kinetic integration
├── scripts.py               # Data migration scripts
├── app_settings.py          # Translatable UI strings
├── apps.py                  # Django AppConfig (name='dos')
├── services/
│   └── geo_service.py       # Geolocation calculations
└── views/                   # (Documented in T24)
    ├── store.py, customer.py, order.py, item.py, payment.py,
    ├── coupon.py, gate.py, socket.py, ingenico.py,
    └── app_specific_views.py
```

### Design Patterns

1. **Strategy Pattern (StoreType)**: Each retailer implements a `StoreType` subclass that provides retailer-specific item, order, payment, and receipt behaviour. The `STORE_TYPE_MAP` dictionary dispatches to the correct implementation at runtime.
2. **Template Method (BaseAgent)**: The agent classes define a consistent interface (`get_item`, `set_sold`, `get_items`) with retailer-specific implementations.
3. **Inheritance Hierarchy (Items)**: `Item` → `SecureItem` → retailer-specific item models, each adding fields relevant to that retailer.
4. **Singleton Helper (OrderHelper)**: A module-level `order_helper` instance provides order creation logic used across the platform.

---

## Data Models

### Store

**File**: `mainServer/dos/models/store.py`

The `Store` model is the central configuration entity. Every transaction, item lookup, and customer interaction is scoped to a store.

| Field | Type | Description |
|-------|------|-------------|
| `store_id` | `UUIDField` (PK) | Primary key, auto-generated UUID |
| `retailer_store_id` | `CharField(255)` | Retailer's own store identifier |
| `retailer` | `ForeignKey → Retailer` | Link to retailer entity (via `mishipay_core`) |
| `name` | `CharField(255)` | Display name (aliased as `retailer` in serializers for backward compat) |
| `chain` | `CharField(255)` | Retailer chain identifier |
| `region` | `CharField(255)` | Geographic region (e.g., "UK", "FR") |
| `sub_region` | `CharField(255)` | Sub-region |
| `address` | `TextField` | Store address |
| `address_json` | `JSONField` | Structured address data |
| `longitude` / `latitude` | `FloatField` | Geolocation coordinates |
| `currency` | `CharField(10)` | Currency code (e.g., "GBP") |
| `currency_sign` | `CharField(5)` | Currency symbol (e.g., "£") |
| `store_type` | `CharField(255)` | Maps to `STORE_TYPE_MAP` key (e.g., "MishipayStoreType") |
| `item_source` | `CharField(255)` | Item data source type |
| `payment_platform` | `CharField(255)` | Primary payment platform |
| `payment_merchant` | `CharField(255)` | Merchant identifier for payment processing |
| `demo` | `BooleanField` | Whether this is a demo/test store |
| `active` | `BooleanField` | Whether the store is active |
| `mishipay_enabled` | `BooleanField` | Whether MishiPay scanning is enabled |
| `globally_available` | `BooleanField` | Whether store appears in global store list |
| `rfid` | `BooleanField` | Whether RFID scanning is enabled |
| `flow_type` | `CharField` | Checkout flow variant |
| `exit_journey_flow_type` | `CharField` | Exit verification flow type |
| `has_manual_check` | `BooleanField` | Whether manual staff verification is required |
| `is_address_required_for_order` | `BooleanField` | Whether delivery address is needed |
| `offers_available` | `BooleanField` | Whether promotions are active |
| `payments_attempted` | `BooleanField` | Whether any payment has been attempted |
| `scanner_type` | `CharField` | Barcode scanner technology |
| `timezone` | `CharField` | Store timezone |
| `stripe_location_id` | `CharField` | Stripe Terminal location ID |
| `telephone_country_code` | `CharField` | Phone number country prefix |
| `extra_data` | `JSONField` | Extensible JSON blob for cluster config, promotion procedures, etc. |
| `auth_provider_details` | `JSONField` | Authentication provider configuration |
| `supports_ms_pay_web/android/ios` | `BooleanField` | MS Pay support per platform |
| `uses_inventory_service_web/ios/android` | `BooleanField` | Inventory service usage per platform |

**Dynamic properties** (computed via `store_properties` relation or `extra_data`):
- `payment_methods` — Available payment methods
- `country_code` — Derived from region
- `apple_pay_merchant_id` / `google_pay_merchant_id` — Platform-specific merchant IDs
- `auth_providers` — List of auth provider names
- `keycloak_openid_hint` — Keycloak login hint
- `applicable_features` — Feature flags (loyalty, boarding pass, kiosk, etc.)

The `get_store_type_choices()` function dynamically generates choice tuples from `STORE_TYPE_MAP` keys.

### Customer

**File**: `mainServer/dos/models/customer.py`

| Field | Type | Description |
|-------|------|-------------|
| `cus_id` | `UUIDField` (PK) | Primary key |
| `user` | `OneToOneField → auth.User` | Django auth user link |
| `email` | `EmailField` | Customer email |
| `first_name` / `last_name` | `CharField` | Name fields |
| `password` | `CharField` | Legacy password field (pre-auth migration) |
| `is_facebook_user` | `BooleanField` | Whether registered via Facebook |
| `source` | `TextField` | Saved payment cards JSON (demo/sandbox) |
| `live_source` | `TextField` | Saved payment cards JSON (production) |
| `sign_up_store` | `ForeignKey → Store` | Store where customer first registered |
| `all_order_not_verified` | `BooleanField` | Blocks scanning if unverified orders exist |
| `device_id` | `CharField` | Device identifier |
| `extra_data` | `JSONField` | Extensible customer data |

### Order

**File**: `mainServer/dos/models/order.py`

| Field | Type | Description |
|-------|------|-------------|
| `order_id` | `UUIDField` (PK) | Primary key |
| `o_id` | `IntegerField` (unique-per-store) | Sequential order number (starts at 100001) |
| `customer` | `ForeignKey → Customer` | Buyer |
| `store` | `ForeignKey → Store` | Store where purchase occurred |
| `date` | `DateTimeField` (auto_now_add) | Order creation timestamp |
| `price` | `FloatField` | Total price |
| `discounted_price` | `FloatField` | Price after discounts |
| `discount_code` | `CharField` | Applied discount code |
| `item_details` | `JSONField` | Full item list snapshot |
| `non_refunded_items` | `JSONField` | Items not yet refunded |
| `refunded_items` | `JSONField` | Items that have been refunded |
| `refund_processing_items` | `JSONField` | Items with refund in progress |
| `returned` | `BooleanField` | Fully returned flag |
| `partially_returned` | `BooleanField` | Partially returned flag |
| `n_scans` | `IntegerField` | Number of verification scans |
| `last_scanned` | `DateTimeField` | Last verification scan time |
| `status` | `CharField` | Order status string |
| `payment_details` | `JSONField` | Payment response data |
| `payment_platform` | `CharField` | Payment platform used |
| `external_order_id` | `CharField` | External system order reference |
| `external_payment_id` | `CharField` | External payment reference |
| `external_payment_reference` | `CharField` | Additional external reference |
| `merchant_name` | `CharField` | Merchant display name |
| `customer_token` | `CharField` | Customer payment token |
| `card_token` | `CharField` | Card payment token |
| `transaction_reference` | `CharField` | Transaction reference from payment gateway |
| `demo` | `BooleanField` | Whether this is a demo order |
| `address_json` | `JSONField` | Delivery address |
| `order_prefix` | `CharField` | Retailer-specific order ID prefix |
| `bundle_id` | `CharField` | App bundle identifier |

**Unique constraint**: `(o_id, store, demo)` — ensures sequential order numbers are unique per store per environment.

`OrderCreateForm` is a `ModelForm` that handles order creation with all fields.

### Item (Base)

**File**: `mainServer/dos/models/item.py`

The base `Item` model stores product data with minimal fields:

| Field | Type | Description |
|-------|------|-------------|
| `store` | `ForeignKey → Store` | Associated store |
| `barcode` | `CharField(255)` | Product barcode (EAN/UPC) |
| `sold` | `BooleanField` | Whether the item has been sold |
| `tax` | `FloatField` | Tax percentage |

### SecureItem

**File**: `mainServer/dos/models/base/base_item.py`

Extends `Item` with RFID/EPC tracking:

| Field | Type | Description |
|-------|------|-------------|
| All `Item` fields | — | Inherited |
| `uid` | `CharField(255)` | EPC/RFID tag unique identifier |

The `uid` field is central to the anti-theft system. When an item is scanned/sold, its `uid` is set in Redis (`r.set(item.uid, 1)`) and published via MQTT so exit gates can verify paid status in real time.

### Retailer-Specific Item Models

Each retailer can extend `SecureItem` with additional fields:

**MishipayItem** (`models/mishipay/mishipay_item.py`):
| Field | Type |
|-------|------|
| `bar_ext` | `CharField(63)` — Barcode extension (combined with barcode for unique ID) |
| `category` | `CharField(63)` |
| `item_name` | `CharField(511)` |
| `vendor` | `CharField(63)` |
| `img` | `URLField` — Product image URL |
| `price` | `FloatField` |
| `story` | `TextField` — Product description |
| `product_id` | `TextField` |

**CamaieuItem** (`models/camaieu/camaieu_item.py`):
Similar fields to MishipayItem. Uses custom `db_table = 'camieu_items'` (note: typo in table name is intentional/legacy).

**ClerkAndGreenItem** (`models/clerk_and_green/clerk_and_green_item.py`):
Extended item model with additional e-commerce fields:
| Field | Type |
|-------|------|
| `bar_ext`, `category`, `item_name`, `vendor`, `img`, `price`, `story` | Standard fields |
| `item_id` | `CharField(31)` — External item identifier |
| `variant_title` | `CharField(255)` — Product variant |
| `variant_id` | `CharField(255)` — Variant identifier |
| `quantity` | `IntegerField` — Stock quantity |
| `product_type` | `TextField` |
| `gender` | `TextField` |

Has a composite database index on `(barcode, bar_ext, store)` for fast lookups.

### TransactionHistory

**File**: `mainServer/dos/models/transaction.py`

| Field | Type | Description |
|-------|------|-------------|
| `customer` | `ForeignKey → Customer` | Paying customer |
| `transaction_reference` | `CharField` | Payment gateway reference |
| `payment_method` | `CharField` | Method used (card, wallet, etc.) |
| `store` | `ForeignKey → Store` | Store context |
| `meta_data` | `JSONField` | Full transaction metadata (amount, etc.) |
| `created_at` | `DateTimeField` | Auto-set timestamp |

### Coupon

**File**: `mainServer/dos/models/coupon.py`

| Field | Type | Description |
|-------|------|-------------|
| `coupon_code` | `CharField(64)` | Unique coupon identifier |
| `store` | `ForeignKey → Store` | Associated store |
| `discount` | `FloatField` | Discount percentage |
| `valid_till` | `DateTimeField` | Expiry date |
| `quantity` | `IntegerField` | Number of uses remaining |

### Device

**File**: `mainServer/dos/models/device.py`

Tracks mobile devices for push notifications:

| Field | Type | Description |
|-------|------|-------------|
| `customer` | `ForeignKey → Customer` | Owner |
| `device_type` | `CharField` | Platform (iOS/Android) |
| `device_token` | `CharField` | Push notification token |
| `store` | `ForeignKey → Store` | Associated store |

### Feedback

**File**: `mainServer/dos/models/feedback.py`

| Field | Type | Description |
|-------|------|-------------|
| `store` | `ForeignKey → Store` | Store context |
| `customer` | `ForeignKey → Customer` | Submitter |
| `rating` | `IntegerField` | Numeric rating |
| `comments` | `TextField` | Free-text feedback |
| `order` | `ForeignKey → Order` | Associated order |

### Gate

**File**: `mainServer/dos/models/gate.py`

Models physical exit gates in stores:

| Field | Type | Description |
|-------|------|-------------|
| `store` | `ForeignKey → Store` | Store location |
| `gate_id` | `CharField` | Gate hardware identifier |
| `gate_type` | `CharField` | Gate hardware type |
| `active` | `BooleanField` | Whether gate is operational |

### AppVersionDetails

**File**: `mainServer/dos/models/app_specific_models.py`

Tracks app versions for force-update requirements:

| Field | Type | Description |
|-------|------|-------------|
| `platform` | `CharField` | ios / android / web |
| `app_type` | `CharField` | Store type (choices from `STORE_TYPE_MAP`) |
| `url` | `URLField` | App store download URL |
| `version` | `CharField(10)` | Version string |
| `force_update` | `BooleanField` | Whether update is mandatory |

### ModelSerializer (stored serializer data)

**File**: `mainServer/dos/models/serializer.py`

A generic model for storing serialized configuration data:

| Field | Type | Description |
|-------|------|-------------|
| `store` | `ForeignKey → Store` | Associated store |
| `data_type` | `CharField` | Type identifier for the data |
| `data` | `JSONField` | The serialized configuration payload |

### Payment Platform Choices

**File**: `mainServer/dos/models/choices.py`

Defines `PAYMENT_PLATFORM_CHOICES` with options including: `stripe`, `paypal`, `payunity`, `adyen`, `ingenico`, `nexi`, `worldpay`, `payu`, and others.

---

## Agent Pattern (Retailer-Specific Item Logic)

**File**: `mainServer/dos/models/base/base_agent.py`

The `BaseAgent` class defines the interface for retailer-specific item operations. Each agent is initialized with a `store` instance and implements methods for item CRUD:

```python
class BaseAgent:
    store = None

    def __init__(self, store):
        self.store = store

    def get_item(self, data):    # Fetch single item by barcode
    def set_sold(self, data):    # Mark item as sold
    def get_items(self, data):   # List items for store
    def add_item(self, data):    # Add new item
    def update_item(self, data): # Update existing item
    def delete_item(self, data): # Delete item
```

### Agent Implementations

| Agent | Source | Item Model | Item Source | Notable Behaviour |
|-------|--------|------------|-------------|-------------------|
| **MishipayAgent** | `models/mishipay/mishipay_agent.py` | `MishipayItem` | Local DB | Barcode + `bar_ext` lookup with Q filters, Redis sold-status caching, Nedap !DCloud integration, pagination support |
| **CamaieuAgent** | `models/camaieu/camaieu_agent.py` | `CamaieuItem` | Local DB | Simple barcode filter, alphanumeric-only barcode sanitization |
| **ClerkAndGreenAgent** | `models/clerk_and_green/clerk_and_green_agent.py` | `ClerkAndGreenItem` | Local DB | Complex Q queries with `bar_ext` fallback, Redis caching, Nedap !DCloud sell/return API calls, MQTT publishing, category filtering, full CRUD operations |
| **ILabAgent** | `models/ilab/ilab_agent.py` | N/A (remote) | External API | Fetches items from `avery.r2retail.com` API, transforms response to standard item dict |
| **LeroyMerlinAgent** | `models/leroy_merlin/leroy_merlin_agent.py` | N/A (remote) | External API | Fetches items from `api.leroymerlin.fr` product repository, `set_sold` is a no-op |

### ClerkAndGreenAgent: The Most Feature-Complete Agent

`ClerkAndGreenAgent` (`models/clerk_and_green/clerk_and_green_agent.py`) demonstrates the full agent pattern with:

1. **Item lookup** with dual Q-query (handles both empty string and null `bar_ext`)
2. **EPC URL decoding** — barcodes containing `httpswebappmishipaycomtag` are decoded to extract the EPC
3. **Redis sold-status caching** — `r.set(item.uid, 1)` on sale, `r.set(item.uid, 0)` on return
4. **MQTT publishing** — `publish_item()` broadcasts sold/unsold status changes
5. **Nedap !DCloud integration** — `set_sold_nedap()` and `update_unsold_nedap()` call Nedap's transaction API for RFID-tracked items
6. **Paginated item listing** — `get_items()` supports page-based pagination
7. **Category filtering** — `get_item_category()` returns distinct categories
8. **Full CRUD** — `add_item()`, `update_item()`, `delete_item()` with form validation

---

## Store Type System

### Base StoreType

**File**: `mainServer/dos/base_store_type.py`

`StoreType` is the abstract base class that every retailer integration must implement. It defines the complete interface for retailer-specific behaviour:

```python
class StoreType:
    item_model = Item                  # Django model for items
    item_serializer = ItemSerializer   # DRF serializer for items
    store = None                       # Store instance
    payment_classes = [BasePayment]    # List of payment gateway classes
    receipt_template = 'email/receipt.html'  # Email receipt template

    def get_item(self, data)           # Fetch item by barcode
    def set_sold(self, data)           # Mark item as sold
    def set_unsold(self, data)         # Mark item as unsold (for returns)
    def get_items(self, data)          # List store items
    def add_item(self, data)           # Add item to inventory
    def add_item_by_csv(self, data)    # Bulk import via CSV
    def update_item(self, data)        # Update item
    def delete_item(self, data)        # Remove item
    def apply_promotions(self, data)   # Apply promotion rules
    def create_order(self, data)       # Create purchase order
    def pay(self, data)                # Process payment
    def refund_order(self, data)       # Process refund
    def update_pending_payment_statuses(self, data)  # Sync payment statuses
    def send_receipt(self, data)       # Email receipt to customer
```

All methods raise `NotImplementedError` — subclasses must override them.

### Store Type Map

**File**: `mainServer/dos/store_map.py`

The `STORE_TYPE_MAP` dictionary maps ~100 retailer string identifiers to their `StoreType` implementations. Of these:

- **9 are imported as classes** (direct Python imports): `CiscoStoreType`, `DecathlonStoreType`, `DixonsStoreType`, `LegoStoreType`, `LeroyMerlinStoreType`, `MangoStoreType`, `MishipayStoreType`, `SaturnStoreType`, `SaturnSmartPayStoreType`
- **~91 are stored as strings** (lazy-loaded or resolved at runtime): `'StandardStoreType'`, `'FlyingTigerStoreType'`, `'WalmartStoreType'`, etc.

This pattern means most newer retailers use a generic/configurable store type that's resolved dynamically, while the original 9 retailers have dedicated Python modules with custom behaviour.

---

## Payment Gateway Classes

**File**: `mainServer/dos/payment.py` (~500+ lines)

### BasePayment

Abstract base class with a single `make_payment(data)` method.

### PayUnityPayment

Integration with PayUnity/OPP (Open Payment Platform):

- **Environments**: Test (`test.oppwa.com`) vs Live (`oppwa.com`), controlled by `store.demo`
- **Capabilities**: `create_checkout()`, `register_card()`, `verify_payment()`, `refund_payment()`, `save_card()`
- **Success codes**: `000.000.000`, `000.000.100`, `000.100.110`, etc.
- **Card storage**: Cards are saved as JSON in `Customer.source` (demo) or `Customer.live_source` (production)
- **Deduplication**: `add_unique_card()` prevents duplicate card entries by matching last4/holder/expiry

### PayPalPayment (Braintree)

Integration with PayPal via Braintree SDK:

- **Environments**: Sandbox vs Production, controlled by `store.demo`
- **Capabilities**: `create_client_token()`, `verify_nonce()`, `refund_payment()`
- **Token flow**: Client token generated server-side → nonce received from client → transaction settlement

### StripePayments

Integration with Stripe:

- **API keys**: `STRIPE_SECRET_DEMO_KEY` / `STRIPE_SECRET_LIVE_KEY` from settings
- **Capabilities**: Direct charge, saved card charge, customer creation, card saving
- **Card storage**: Same pattern as PayUnity — JSON in `Customer.source`/`live_source`
- **Save card flow**: Creates Stripe Customer → creates Charge → saves card details locally

### cal_key() (Order ID Generation)

Shared function (also duplicated in `OrderHelper`) that generates sequential order IDs:
```python
def cal_key(store):
    present_keys = Order.objects.filter(store=store, demo=store.demo)
        .order_by('-o_id').values_list('o_id', flat=True)
    return present_keys[0] + 1 if present_keys else 100001
```
Order IDs start at **100001** per store and auto-increment. A retry loop (up to 10 attempts) handles `IntegrityError` on concurrent inserts.

---

## Order Creation Logic

**File**: `mainServer/dos/order.py`

The `OrderHelper` class (module-level singleton: `order_helper`) handles order creation:

1. **Customer lookup** by `cus_id`
2. **Address parsing** — extracts `state`, `postal`, `streetAddress`, `addressLine`, `country`, `city`, `name`, `deliveryNote` from `address_json`
3. **Price normalization** — replaces commas with dots via regex (`re.sub`)
4. **Retailer prefix** — looks up `settings.ORDER_PREFIX` by retailer name
5. **Order ID assignment** — `cal_key()` gets next sequential ID, retries up to 10 times on collision
6. **Form-based creation** — uses `OrderCreateForm` for validation and persistence

---

## Serializers

**File**: `mainServer/dos/serializers.py`

### Validation Functions

| Function | Rules |
|----------|-------|
| `validate_first_name` | Required, 2-50 chars, alphanumeric + spaces only |
| `validate_last_name` | Optional (empty allowed), 2-50 chars, alphanumeric + spaces only |
| `validate_store_id` | Must match existing `Store.store_id` |
| `validate_password` | Minimum 10 characters |

### Serializer Classes

| Serializer | Model | Key Fields |
|------------|-------|------------|
| `CustomerSerializer` | Customer | `cus_id`, `email` |
| `StoreSerializer` | Store | Core fields (store_id, location, region, currency, etc.) + `retailer` alias for backward compat |
| `LoginSerializer` | — | `email`, `password` |
| `SignupSerializer` | — | `email`, `password`, `first_name`, `last_name`, `store_id` (all validated) |
| `OrderSerializer` | Order | Nested Customer + Store, all order fields including refund tracking |
| `AppVersionDetailsSerializer` | AppVersionDetails | `platform`, `app_type`, `url`, `force_update`, `version` |
| `ExitQRCodeSerializer` | — | `order_id` (UUID), `store_id` (UUID) |

### ClosestStoreDetailSerializer

The most complex serializer (~200 lines). Extends `StoreSerializer` with **computed fields**:

- `applicable_features` — Full feature flag dictionary including loyalty, kiosk, click-and-collect, boarding pass, security tag popup, basket options, side nav options, special discount configs (referral, student), item recommendations, coffee subscription
- `privacy_policy_types` — Privacy policies from store properties
- `payment_methods`, `country_code`, `auth_providers` — Computed from store config
- `psp_details` — Payment service provider details (currently returns empty dict)
- `random_verification_check` — Enabled for FlyingTiger and Stellar store types
- `logo`, `map_view_*` — Branding images from store properties
- `cluster_id/type/name` — Store clustering config from `extra_data`
- `supports_ms_pay_*` — MS Pay support per platform (disabled for demo stores in prod)
- `retailer_id` — Retailer UUID from retailer relation

**StoreDetailSerializer** is similar but adds `theme` (from LoyaltyDetails), click-and-collect images, and `telephone_country_code`.

**ClosestStoreDetailSerializerV1** extends `ClosestStoreDetailSerializer` but removes `auth_provider_details` for backward compatibility.

---

## Utility Functions

**File**: `mainServer/dos/util.py` (~1010 lines)

### Store Discovery

- `get_closest_store_util(longitude, latitude)` — Iterates all stores, calculates geodesic distance, returns closest store within 1000m or falls back to global store at (0,0)

### MQTT Publishing

Real-time notifications via MQTT (paho) to two brokers (`m14.cloudmqtt.com:11315` and `35.176.179.166:1883`):

| Function | Topic | Purpose |
|----------|-------|---------|
| `publish_item()` | `mishipay/{store_id}/item` | Item sold/unsold status change |
| `publish_first_scan()` | `mishipay/{store_id}/items` | Customer's first item scan |
| `publish_order()` | `mishipay/{store_id}/order` | New order created |
| `publish_refund_order()` | `mishipay/{store_id}/refund` | Order refunded |
| `publish_customer_order_updates()` | `mishipay/{cus_id}` | Order status update to customer |
| `publish_customer_updates()` | `mishipay/{cus_id}` | General customer notification |
| `publish_dashboard_auth_updates()` | `mishipay/{store_id}/order/auth` | Order auth status to dashboard |

### Firebase Push Notifications

- `publish_firebase_notification()` — Customer-facing push notifications via FCM (Firebase Cloud Messaging)
- `publish_firebase_dashboard_notification()` — Dashboard push notifications for store staff (new scan, new purchase, refund)

### Email / Receipt

- `send_email()` — Basic SMTP email sending
- `send_receipt(order)` — Generates HTML receipt email (runs in separate `Process`). Handles VAT receipt template for specific stores, validates image URLs, calculates tax

### Refund Calculation

- `refund_full_amount()` — Calculates full refund for requested items
- `refund_partial_amount()` — Handles partial refund with quantity tracking
- `find_amount_and_db_entries_for_refund()` — Dispatcher for full/partial refund
- `update_refunded_items()` — Merges new refund items with previously refunded items
- `calculate_refund_amount_and_item_count()` — Totals refund amount and item count from order
- `calculate_refund_amount_and_item_count_for_failed_refund()` — Same for failed refund retry

### Session & Analytics

- `set_session_parameter()` — Sets analytics session vars (cus_id, store_id, lat/long, platform, network, device, bundle_id, outside_geofencing)
- `set_session_parameter_after_redirection()` — Simplified version for post-redirect flows
- `set_parameter_without_request()` — Extracts analytics params from a generic order object
- `set_params_for_events()` — Converts session to analytics event params with privacy filtering

### Mixins

- `GeoFencingMixin` — View mixin that checks distance between user and store, sets `outside_geofencing_check` in session
- `CustomerNotVerifiedMixin` — View mixin that blocks customers with `all_order_not_verified = True`

### Helpers

- `error_message()` / `success_message()` — Standard JSON response helpers
- `validate_price()` — Validates positive decimal with max 2 decimal places
- `validate_quantity()` — Validates positive integer
- `is_valid_uuid()` — UUID format validation
- `UUIDEncoder` — JSON encoder that handles UUID objects
- `create_identifier_pattern()` / `decode_identifier_pattern()` — Encode/decode order ID + verification token for XPay redirect URLs
- `get_settings_store_map_name()` — Converts store name to settings key format
- `update_customer_verification_status()` — Clears `all_order_not_verified` when no unverified orders remain

---

## Geolocation Service

**File**: `mainServer/dos/services/geo_service.py`

Provides distance calculation and geofencing:

- `calculate_geo_distance(from, to, unit)` — Geodesic distance between two coordinates
- `calculate_geo_fencing(distance, radius)` — Checks if distance exceeds `settings.GEOFENCING_RANGE`
- `calculate_geo_location_features(from, to)` — Combined distance + geofencing result
- Unit conversion between meters, kilometers, miles, centimeters

---

## Database Access Object

**File**: `mainServer/dos/db.py`

`DatabaseDao` provides raw SQL query execution using Django's `connection.cursor()`:

- `query()` — Execute SELECT, return raw tuples
- `query_with_keys()` — Execute SELECT, return list of dicts (column name → value)
- `query_with_group_by()` — Execute SELECT with grouping, return dict of grouped results
- `insert()` / `update()` — Execute INSERT/UPDATE statements

This is a legacy pattern used for performance-critical queries that bypass the ORM.

---

## Permissions

**File**: `mainServer/dos/permissions.py`

`IsBlockedUser` — DRF permission class that denies access to users in the `blocked_user` group. Returns a 403 with a support contact message.

---

## Decorators

**File**: `mainServer/dos/decorators.py`

`@authenticate` — Function-based view decorator that validates customer session. **Currently bypassed** — the function returns immediately (`return func(request)`) before reaching the session check logic. The dead code suggests this was disabled during development and never re-enabled.

---

## MQTT Client

**File**: `mainServer/dos/mqtt.py`

Dedicated MQTT client for **SATO shelf-monitoring integration**:

- Connects to SATO's MQTT broker with TLS
- Subscribes to replenishment zone topics (`/com/sgs/vision/replenishment/zone/...`)
- On message: checks `addList` and `updateList` for opened issues with positive counts, republishes to `mishipay/nrf/notification` topic
- Contains commented-out Cisco Kinetic RFID integration code

---

## Migration Scripts

**File**: `mainServer/dos/scripts.py`

Data migration utilities:

- `migrate_data_from_customer_to_user()` — Creates Django `auth.User` for each `Customer`, adds to "Customer" group
- `migrate_data_from_dashboard_user_to_user()` — Creates Django `auth.User` for each `DashboardUser`, adds to "DashboardUser" group
- `create_reset_link()` — Generates signed password reset URL
- `send_reset_link_to_all_customers()` — Bulk password reset emails (excluding Facebook users)

---

## App Settings

**File**: `mainServer/dos/app_settings.py`

Translatable string constants:
- `AUTHENTICATION_FAILED` — Login failure message
- `REGISTERATION_PROMOTION` — MUJI-specific registration incentive
- `RANDOM_VERIFICATION_MESSAGE` — Exit verification warning
- `STORE_NOT_FOUND` — Store lookup failure

---

## Dependencies

### Depends On
- `mishipay_core` — `Retailer` model, `EntityPropertyChoice`, analytics privacy filtering
- `mishipay_core.interfaces.customer_interface` — `CustomerInterface` for user group choices
- `dashboard.models` — `DashboardUser` model (for migration scripts)
- `config.loader` — `EMAIL_SETTINGS` for email configuration
- `mainServer.settings` — `VAT_RECEIPT_STORE_IDS`, environment keys
- `mainServer.constants` — `ORDER_NOT_VERIFIED`, `ANALYTICS_DEFAULT_STORETYPE`
- External services: Stripe, Braintree (PayPal), PayUnity/OPP, Nedap !DCloud, Avery R2 Retail, Leroy Merlin API, Firebase FCM, SATO MQTT
- Infrastructure: Redis (sold-status cache), MQTT brokers (paho), geopy

### Depended On By
- All retailer-specific modules (`cisco/`, `decathlon/`, `dixons/`, `lego/`, `mango/`, `saturn/`, etc.)
- `dashboard` — order management, store configuration
- All views and API endpoints in the platform
- `mishipay_retail_payments` — payment provider details

---

## Configuration

| Setting | Source | Description |
|---------|--------|-------------|
| `STRIPE_SECRET_DEMO_KEY` / `STRIPE_SECRET_LIVE_KEY` | Django settings | Stripe API keys |
| `PAYUNITY_USER_ID`, `PAYUNITY_PASSWORD`, `PAYUNITY_ENTITY_ID` | Env vars | PayUnity credentials |
| `PAYPAL_MERCHANT_ID`, `PAYPAL_PUBLIC_KEY`, `PAYPAL_PRIVATE_KEY` | Env vars | Braintree credentials |
| `REDIS_HOST` | Django settings | Redis host for sold-status cache |
| `GEOFENCING_RANGE` | Django settings | Maximum distance (meters) for store geofencing |
| `ORDER_PREFIX` | Django settings dict | Per-retailer order ID prefixes |
| `VAT_RECEIPT_STORE_IDS` | Django settings | Store IDs requiring VAT receipts |
| `mqtt_username`, `mqtt_password` | Env vars | MQTT broker credentials |
| `email_address`, `email_password` | Env vars | SMTP credentials |
| `ENV_TYPE` | Django settings | Environment type ("prod", etc.) |

---

## Notable Patterns

1. **Dual card storage**: Customer payment cards are stored as JSON strings in two separate fields — `source` for demo/sandbox, `live_source` for production. This approach avoids test cards leaking to production.

2. **Order ID sequencing**: Order IDs (`o_id`) are per-store sequential integers starting at 100001, with a retry loop (10 attempts) to handle concurrent inserts. The function `cal_key()` is duplicated in both `order.py` and `payment.py`.

3. **Bypassed authentication decorator**: The `@authenticate` decorator in `decorators.py` is effectively disabled — it returns immediately before checking the session. This appears to be a development/testing convenience that was never reverted.

4. **Barcode sanitization**: Multiple agents use `''.join(filter(lambda c: c in string.ascii_letters + string.digits, barcode))` to strip non-alphanumeric characters from barcodes before lookup.

5. **EPC URL decoding**: The pattern `httpswebappmishipaycomtag` (a URL-stripped NFC tag) is detected and decoded to extract the EPC identifier.

6. **Feature flags via store_properties**: The `applicable_features` serializer method constructs a large default feature dictionary and then overlays store-specific overrides from `store_properties`. This allows per-store feature toggling without code changes.

7. **Dual MQTT broker publishing**: Many publish functions send to both `m14.cloudmqtt.com` (CloudMQTT) and `35.176.179.166` (self-hosted), suggesting a migration was in progress.

---

## Open Questions

1. **DOS naming origin**: While "DOS" likely stands for "Deloitte Operating System", the exact naming history and whether there's a formal connection to Deloitte (beyond being an early deployment partner) is unclear.

2. **Agent vs StoreType overlap**: Both the `BaseAgent` pattern (in `models/base/base_agent.py`) and the `StoreType` pattern (in `base_store_type.py`) define similar interfaces (`get_item`, `set_sold`, etc.). The relationship between them — whether agents are used inside store types, or represent an older pattern — needs clarification from T24 when views are documented.

3. **Payment credential handling**: Some payment credentials appear hardcoded as defaults alongside environment variable lookups. The security implications of these fallback values should be reviewed.

4. **Dead code in serializers**: `get_psp_details()` in both `ClosestStoreDetailSerializer` and `StoreDetailSerializer` has `return psp_details` (empty dict) **before** the actual logic, making all subsequent code unreachable.

5. **MQTT migration status**: The dual-broker publishing pattern suggests an incomplete migration from CloudMQTT to a self-hosted broker. Current operational status is unclear.

---
---

# Part 2: Views, APIs & URL Routing

> Added by: Task T24

This section documents the DOS module's view layer — all API endpoints, URL routing, request/response patterns, authentication strategies, analytics integration, and test coverage.

---

## Views Architecture

### Module Structure

**File**: `mainServer/dos/views/__init__.py`

The views package uses wildcard re-exports from 10 submodules:

```
dos/views/
├── __init__.py              # Re-exports from all submodules
├── store.py                 # Store discovery & configuration (1121 lines)
├── customer.py              # Authentication, profile, social auth (2464 lines)
├── order.py                 # Order creation, retrieval, confirmation (917 lines)
├── item.py                  # Item scanning, promotions, coupons (401 lines)
├── payment.py               # Payment platform integrations (1215 lines)
├── gate.py                  # Exit gate hardware integration (142 lines)
├── coupon.py                # Coupon lookup (30 lines)
├── socket.py                # Socket server status check (36 lines)
├── ingenico.py              # Ingenico hosted checkout (72 lines)
└── app_specific_views.py    # App version force-update (73 lines)
```

### View Patterns

The codebase uses **two parallel patterns** for API endpoints:

1. **Function-Based Views (legacy)**: Decorated with `@authenticate` (bypassed), `@csrf_exempt`, `@transaction.atomic`. These use session-based authentication and are prefixed without `v2/`.

2. **Class-Based Views (DRF)**: Extend `APIView` with `permission_classes = [IsAuthenticated]` and token-based authentication. These are prefixed with `v2/` or `v3/`.

Both patterns coexist — many endpoints have both a legacy function-based and a newer class-based implementation with identical logic.

### Authentication Strategies

| Strategy | Mechanism | Used By |
|----------|-----------|---------|
| **Token Auth** | DRF `TokenAuthentication` via `Authorization: Token <key>` header | All `v2/` class-based views |
| **Session Auth** | Django session cookies | Legacy function-based views |
| **No Auth** | `permission_classes = []`, `authentication_classes = []` | Store discovery, app open, device registration, social auth endpoints |
| **@authenticate decorator** | Session-based (currently bypassed — returns immediately) | Legacy FBVs like `get_item`, `process_order` |

### Response Format

All endpoints use two standard response helpers from `dos/util.py`:

- `success_message(data)` → `{"success": 1, "result": <data>}`
- `error_message(error_name, message, status)` → `{"success": 0, "error": <error_name>, "message": <message>}`

Some newer views return raw DRF `Response` objects instead.

---

## URL Routing

**File**: `mainServer/dos/urls.py` (174 lines)

All DOS URLs are mounted under the `/d/` prefix (configured in the project-level `urls.py`). The routing uses both `re_path` (regex) and `path` (literal).

### Customer Endpoints

| URL | View | Method | Auth | Description |
|-----|------|--------|------|-------------|
| `customer/signup` | `sign_up` | POST | None | Email registration |
| `customer/signin` | `sign_in` | POST | None | Email login |
| `customer/signout` | `SignOut` | POST | Token | Delete auth token (logout) |
| `customer/guest-signin` | `sign_in_guest` | POST | None | Guest user creation |
| `customer/update-guest-email` | `UpdateGuestUserInfoAPIView` | POST | Token | Update guest user email/name |
| `customer/autoregister` | `AutoRegister` | POST | Token | Convert guest to registered user post-payment |
| `customer/generate-otp` | `GenerateOTP` | POST | None | Send OTP email (throttled) |
| `customer/verify-otp` | `VerifyOTP` | GET | None | Verify OTP and get token |
| `customer/update-tokensdata` | `UserTokensDataUpdate` | PUT | Token | Update token metadata (store_id, etc.) |
| `customer/facebook` | `fb_authenticate` | POST | None | Facebook OAuth login/signup |
| `customer/apple` | `apple_authenticate` | POST | None | Apple Sign-In |
| `social/<backend>/` | `exchange_token` | POST | None | Google OAuth via Python Social Auth |
| `v2/customer/profile/` | `CustomerProfile` | GET/PUT | Token | Get/update customer profile |
| `v2/customer/update-user-meta/` | `UpdateUserMeta` | PUT | Token | Update user metadata JSON |
| `v2/customer/recover-password/` | `PasswordRecovery` | POST | None | Send password reset email |
| `v2/customer/reset-password/` | `ResetPassword` | POST | None | Reset password with signed token |
| `v2/customer/verify_mail/` | `CustomerMailVerification` | POST | Token | Send email verification |
| `v2/customer/verify_code/` | `CustomerCodeVerification` | POST | Token | Verify email with code |
| `v2/customer/deactivate_self/` | `DeactivateCustomerBySelf` | POST | Token | Self-deactivation |
| `v2/customer/feedback/` | `CustomerFeedbackView` | POST | Token | Submit order/store feedback |
| `v2/webapp/recover-password/` | `PasswordRecoveryForWebApp` | POST | None | Web app password recovery |
| `v2/webapp/reset-password/` | `ResetPasswordForWebApp` | POST | None | Web app password reset |

### Decathlon-Specific Customer Endpoints

| URL | View | Method | Auth | Description |
|-----|------|--------|------|-------------|
| `v2/decathlon-login-signup/` | `decathlon_login_callback` | POST | None | Decathlon SSO login/signup via third-party client |
| `v2/update-user-information/` | `UpdateCustomerInfoDecathlon` | POST | None | Update Decathlon user info |
| `v2/decathlon-check-email-exists/` | `CheckEMailExistsDecathlon` | GET | Token | Check email in Decathlon system |
| `v2/decathlon-forgot-password/` | `ForgotPasswordDecathlon` | POST | Token | Decathlon password reset |

### Store Endpoints

| URL | View | Method | Auth | Description |
|-----|------|--------|------|-------------|
| `v2/store/` | `StoreView` | GET/POST | None | Get closest store by GPS + bundle ID; or create store |
| `store/closest` | `get_closest_store` | GET | None | Legacy closest store by GPS |
| `v2/store/mango` | `get_closest_mango_store` | GET | None | Closest Mango-type store |
| `v2/stores/closest-stores/` | `ClosestStoreView` | GET | None | Nearest stores with geofencing, clustering, payments |
| `v3/stores/closest-stores/` | `ClosestStoreViewV3` | GET | None | V3 with MS Pay optimization and theme pre-population |
| `v2/stores/list/` | `StoreList` | GET | Token | List stores by bundle ID with search |
| `v2/store/find-store/` | `ClusterStoreConfirmationView` | POST | Token | Cluster store entry/exit journey QR validation |
| `v2/stores/get-store-id/` | `AppClipStoreID` | GET | None | Resolve app clip ID to store |
| `v3/store/<uuid>/` | `GetStoreViewV3` | GET | None | Get single store by UUID (uses `RetrieveAPIView`) |
| `v4/store/<uuid>/` | `GetStoreViewV4` | GET | None | V4 with full payment/theme pre-population |
| `get-global-config/<uuid>/` | `GetGlobalConfigView` | GET | Token | Get promotion config for store |

### Item Endpoints

| URL | View | Method | Auth | Description |
|-----|------|--------|------|-------------|
| `v2/item/get/` | `ItemView` | GET | Token | Scan item by barcode, with store type routing |
| `v2/item/status/` | `ItemStatusView` | POST | Token | Get sold status for items |
| `v2/item/get/all/` | `AllItemView` | GET | Token | Get all items for a store |
| `v2/item/promotion/` | `ItemPromotion` | GET | Token | Apply promotions to items |
| `v2/item/shopping-bags/` | `GetShoppingBags` | GET | Token | Get shopping bag items |

### Order Endpoints

| URL | View | Method | Auth | Description |
|-----|------|--------|------|-------------|
| `v2/order/` | `OrderView` | GET | Token | Get order history for customer |
| `v2/order/create/` | `OrderView` | POST | Token | Create order via store type |
| `v2/order/confirm/` | `OrderConfirm` | GET/POST | Token/None | Confirm order (GET = XPay redirect, POST = app-driven) |
| `v2/order/single` | `OrderFetch` | GET | Token | Fetch single order by ID |
| `v2/judo/keys/` | `JudoKey` | GET | Token | Get JudoPay API credentials |
| `v2/spreedly/keys/` | `SpreedlyKey` | GET | Token | Get Spreedly environment key |

### Payment Endpoints

| URL | View | Method | Auth | Description |
|-----|------|--------|------|-------------|
| `v2/payment/update/` | `PaymentUpdateView` | POST | Token | Save card details (JudoPay format) |
| `v2/payment/remove_card/` | `PaymentRemoveCardView` | POST | Token | Remove card (multi-platform) |
| `v2/payment/success/` | `payment_success` | POST | None | JudoPay payment success redirect |
| `v2/payment/failure/` | `payment_failure` | POST | None | JudoPay payment failure redirect |
| `v2/payment/initialize/` | `PaymentInitialize` | POST | Token | Initialize JudoPay web payment |
| `stripe/pay` | `StripePayment` | POST | None | Process Stripe payment |
| `stripe/remove_card` | `remove_stripe_card` | POST | None | Remove Stripe saved card |
| `v2/payunity/checkout` | `PayunityCheckout` | POST | None | Create PayUnity checkout session |
| `v2/payunity/result` | `PayunityResult` | GET | None | PayUnity result callback (placeholder) |
| `v2/payunity/remove_card` | `PayunityRemoveCard` | POST | Token | Remove PayUnity card |
| `v2/payunity/add_card` | `PayunityAddCard` | POST | Token | Register PayUnity card |
| `v2/paypal/client` | `PaypalClient` | GET | Token | Get PayPal/Braintree client token |
| `v2/verifone/session` | `VerifoneSession` | GET | Token | Create Verifone session |
| `v2/verifone/save-card` | `VerifoneRegisterCard` | POST | Token | Save card via Verifone |
| `v2/adyen/session` | `AdyenSession` | POST | Token | Create Adyen session |
| `v2/adyen/save-card` | `AdyenRegisterCard` | POST | Token | Save card via Adyen |
| `v2/adyen/authorize` | `AdyenAuthorizeOrder` | POST | Token | Authorize and capture Adyen payment |
| `v2/xpay/notify/` | `xpay_notify_save_card` | POST | None | XPay card-save webhook |
| `v2/xpay/save-card/` | `XPaySaveCardAPI` | POST | Token | Initiate XPay card save |
| `v2/apple-pay/session/` | `ApplePaySession` | POST | Token | Create Apple Pay session |
| `v2/google-pay/<action>/` | `GooglePayAPIView` | POST | Token | Google Pay actions (session) |
| `v2/onepay/save-card/<action>/` | `OnePaySaveCardAPIView` | POST | Token | OnePay card operations |

### Other Endpoints

| URL | View | Method | Auth | Description |
|-----|------|--------|------|-------------|
| `v2/gate/info/` | `GateInfo` | GET | Token | Get gate hardware info by serial number |
| `v2/gate/items/` | `GateItems` | GET | Token | Get items for gate's store |
| `v2/coupon/add` | `get_coupon_details` | GET | None | Look up coupon by code + store |
| `v2/socket/status/` | `SocketStatus` | GET | Token | Check socket server status |
| `v2/app/open/` | `AppOpenView` | GET | None | Log app open event |
| `v2/app/register/` | `AppRegister` | POST | Token | Register device for push notifications (APNS/FCM) |
| `v2/app/event/` | `AppEvent` | POST | None | Log generic app event |
| `v2/device/register` | `RegisterDevice` | POST | None | Register device UUID + FCM token |
| `v2/app-force-update-status/` | `AppForceUpdateStatus` | GET/POST | None | Check/list force-update versions |

---

## View Details by Domain

### Store Views

**File**: `mainServer/dos/views/store.py` (1121 lines)

The store views handle **store discovery** — finding the nearest store based on user GPS coordinates and the app's bundle ID.

#### `StoreView` (lines 195-304)

The primary store discovery endpoint. No authentication required.

**GET flow**:
1. Extract `bundleid` from request header, `latitude`/`longitude` from query params
2. Filter stores by bundle ID via `Store.objects.find_by_bundle_id(bundle_id)`
3. Calculate geodesic distance from user to each store
4. Find the closest store
5. Apply **geofencing**: if distance > `settings.GEOFENCING_RANGE`, flag as `outside_geofencing`
6. For MishiPay-branded apps (`white_list_status == '1'`), assign the **global store** (at coordinates 0,0) when outside geofencing
7. Return serialized store data with `auth_provider_details`, `static_assets_url`, and geofencing flag

#### `ClosestStoreView` (lines 331-545)

Returns **multiple stores** within a map radius. Supports **store clustering** (multiple physical locations grouped under one cluster ID).

**Key logic**:
- `pre_populate_stores()` — Batch-fetches `store_properties`, `ThemeProperty` (web theme), `PaymentVariant`, and `OAuthApplication` data for all matched stores in a minimal number of DB queries. Attaches results as `cur_*` attributes on store instances to avoid N+1 queries during serialization.
- Store clustering: stores with `extra_data.cluster_id` are grouped, and only the closest store per cluster is returned.
- Falls back to the global store if no stores are found within range.

#### `ClosestStoreViewV3` (lines 548-764)

Optimized V3 with **MS Pay support filtering**. Key differences from V2:
- `store_pre_populate_and_get_clusters()` combines pre-population and clustering into one pass
- Only fetches `PaymentVariant` for stores where MS Pay is not fully enabled on all platforms
- Uses `StoreDetailSerializer` instead of `ClosestStoreDetailSerializer` for richer theme data

#### `ClusterStoreConfirmationView` (lines 875-1015)

Handles the **entry/exit QR code journey** for cluster stores:

**Entry flow**: Customer scans a QR code with a `cluster_store_identifier` → lookup store by `extra_data__cluster_store_identifier` → return store details.

**Exit flow**: If no store matches the identifier directly, it's treated as an HMAC-signed exit QR. The view:
1. Validates `order_id` and `store_id` from request data
2. Computes HMAC-SHA256 of the store's cluster identifier using `settings.CLUSTER_SECRET_KEY`
3. Compares the computed hash to the scanned identifier
4. On match: verifies the order, sets `order_status = VERIFIED`, returns order details with exit instructions
5. Records journey analytics throughout

#### `GetStoreViewV4` (lines 1025-1091)

Single-store retrieval by UUID with full pre-population of payment and theme data. Uses `ActiveStorePermission` (no auth but store must be active).

### Customer Views

**File**: `mainServer/dos/views/customer.py` (2464 lines)

The largest view file. Handles all authentication flows, profile management, social auth, and device registration.

#### Authentication Flows

**Email Sign-Up** (`sign_up` function, not shown in class-based form):
1. Validate with `SignupSerializer` (email, password ≥10 chars, name 2-50 chars)
2. Check for existing customer (including deactivated — hashed email lookup)
3. Create Django `User` + `Customer` objects
4. Add user to "Customers" group
5. Generate DRF `Token`
6. Store `TokenData` (store_id, user_language, etc.) via `update_token_data()`
7. Return customer dict with `access_token`

**Email Sign-In** (`sign_in` function):
1. Validate with `LoginSerializer` (email, password)
2. Look up Customer by email (also checks hashed email for deactivated accounts)
3. Verify password via `check_password()`
4. Django login + DRF token creation
5. Record login analytics
6. Return customer dict with `access_token` and `show_notifications` flag

**Guest Sign-In** (`sign_in_guest` / `guest_signin`):
1. Create a temporary Django User with auto-generated credentials
2. Create Customer with `is_guest_user=True` and `guest_user_identifier` UUID
3. No email/password required
4. Return token for immediate API access

**Sign-Out** (`SignOut`, line 2172):
Deletes the DRF `Token` object, forcing re-authentication.

#### Social Authentication

**Apple Sign-In** (`apple_authenticate`, lines ~1440-1552):
1. Validate `AppleSocialSerializer` (requires `code`, optional `first_name`, `last_name`, `store_id`)
2. Decode Apple's `id_token` JWT using `jwt.decode()` with Apple's public keys (fetched from `https://appleid.apple.com/auth/keys`)
3. Exchange authorization code for tokens via Apple's token endpoint
4. Extract email from JWT claims (handles private relay emails)
5. Find or create Customer with `is_apple_user=True`, `is_email_verified=True`

**Facebook Auth** (`fb_authenticate`, lines 1560-1695):
1. Validate `FacebookSocialSerializer` (requires `access_token`)
2. Fetch user info from `https://graph.facebook.com/me?fields=email,first_name,last_name`
3. Find or create Customer with `is_facebook_user=True`
4. Store Facebook access token on customer
5. Record analytics

**Google Auth** (`exchange_token`, lines 1703-1835):
1. Uses `@psa()` decorator (Python Social Auth)
2. Validate `GoogleSocialSerializer` (requires `id_token`)
3. Verify ID token via `https://www.googleapis.com/oauth2/v3/tokeninfo`
4. Find or create Customer with `is_google_user=True`
5. Record analytics

**Decathlon SSO** (`decathlon_login_callback`, lines 2015-2105):
1. Delegates to `thirdparty_client.LoginSignupClient` microservice
2. Handles basket replication from guest user to authenticated user
3. Records analytics

#### Profile Management

**`CustomerProfile`** (GET/PUT):
- GET: Returns customer dict with cards, email verification status, feedback count
- PUT: Updates first/last name (validated 2-50 chars)

**`UpdateUserMeta`** (PUT): Updates the `user_metadata` JSON field on Customer.

**`UpdateGuestUserInfoAPIView`** (POST): Updates guest user's email, first/last name.

#### Password Recovery

Two parallel flows exist:

**Mobile app flow** (`PasswordRecovery` + `ResetPassword`):
1. `PasswordRecovery.post()`: Generates a signed token using `TimestampSigner`, sends reset email
2. `ResetPassword.post()`: Validates signed token, sets new password

**Web app flow** (`PasswordRecoveryForWebApp` + `ResetPasswordForWebApp`):
1. Same as mobile but uses `WEBAPP_RESET_PASSWORD_URL` instead of deep links

#### OTP Authentication

**`GenerateOTP`** (POST, throttled via `throttle_scope = 'otp'`):
1. Validate `OTPSerializer` (email must match existing customer)
2. Generate 6-digit random code via `random.SystemRandom()`
3. Send OTP via email
4. Store code + timestamp on customer

**`VerifyOTP`** (GET):
1. Validate `OTPVerifySerializer` (email + otp)
2. Check OTP matches and is within 10-minute window
3. Return DRF token on success

#### Auto-Registration

**`AutoRegister`** (POST, line 2219):
Converts a guest user to a registered user after payment:
1. Validate `AutoregisterSerializer` (requires `order_id`, `email_id`, optional `full_name`)
2. Check order exists, is completed/verified, and belongs to a guest user
3. If email matches existing customer → link order to that customer, transfer `BuyerAddress` records
4. If new email → convert guest customer to registered
5. Set `is_auto_registered=True`, clear `guest_user_identifier`

#### Device & App Lifecycle

**`AppOpenView`** (GET, no auth): Logs app-open event, resets `first_scan` flag.

**`AppRegister`** (POST): Registers device for push notifications — creates/updates `APNSDevice` (iOS) or `GCMDevice` (Android/FCM) records. Validates FCM tokens via `validate_fcm_token()`.

**`RegisterDevice`** (POST, no auth): Registers device UUID + FCM token in the `Device` model.

**`AppEvent`** (POST, no auth): Generic event logger (currently just returns success without persisting).

#### Customer Feedback

**`CustomerFeedbackView`** (POST): Records customer feedback for an order. Creates `CustomerFeedback` object, increments `customer.feedback_count`, marks order as `is_rated`. Also handles standalone feedback without an order. Records rating analytics.

### Order Views

**File**: `mainServer/dos/views/order.py` (917 lines)

#### `OrderView` (lines 247-403)

The primary order endpoint.

**GET** — Retrieve order history:
1. Get customer from `request.user.customer`
2. Filter orders by customer, excluding whitelisted stores
3. Apply store-type-specific status filters:
   - `SaturnSmartPayStoreType`: Only show `PAYMENT_COMPLETED`, `ORDER_CORRECT`, `ORDER_INCORRECT`, `ORDER_NOT_VERIFIED`
   - `DecathlonStoreType` / `LegoStoreType`: Only show `PAYMENT_COMPLETED`
   - Others: Show all orders
4. Include `non_refunded_items` and `refunded_items` in response
5. Null values in item details are replaced with empty strings

**POST** — Create order:
1. Get store by `store_id`
2. Instantiate store type via `STORE_TYPE_MAP.get(store.store_type)(store)`
3. Handle guest user context update for Apple Pay payments
4. Call `store_type.create_order(data)` → returns `(result, success, secondary_message)`
5. Record order creation and payment analytics
6. Return order data or error

#### `OrderConfirm` (lines 457-618)

Handles order confirmation — the second step in a two-phase payment flow (used by XPay, OnePay).

**GET** (redirect-based confirmation, e.g., XPay callback):
1. Decode `identification_token` query param → extract `order_id` + `verification_token`
2. Look up order and its store settings
3. Call `store_type.confirm_order(data)`
4. On success: redirect to `order_success_redirect_url`
5. On failure: redirect to `order_failure_redirect_url`
6. Record payment analytics with platform, price, item count

**POST** (app-driven confirmation):
1. Look up order by `order_id` + customer
2. Call `store_type.confirm_order(data)`
3. Upgrade push notification validity on success
4. Return result or error

#### `OrderFetch` (lines 405-454)

Single order retrieval by `order_id`. Returns full order details including refund status.

#### `OrderCreateView` (lines 621-670, deprecated)

Legacy order creation endpoint. Same logic as `OrderView.post()` but passes `request` as an additional argument to `create_order()`.

#### `create_order_from_stripe` (lines 697-825)

Utility function called by `StripePayment.post()` after successful Stripe charge:
1. Create order via `OrderCreateForm` with retry loop for ID collisions
2. Mark items as sold via `store.agent.set_sold(item)`
3. Send receipt email
4. Publish order to dashboard via MQTT

#### `JudoKey` / `SpreedlyKey` (lines 673-851)

Return payment gateway API credentials from environment variables. JudoKey returns test/live tokens and secrets. SpreedlyKey returns the Spreedly environment key (demo vs live based on store).

### Item Views

**File**: `mainServer/dos/views/item.py` (401 lines)

#### `ItemView` (lines 147-276)

The primary item scanning endpoint. Uses `CustomerNotVerifiedMixin` to block customers with unverified orders, and `IsBlockedUser` permission.

**GET flow**:
1. Validate `TokenData` serializer against query params
2. Get store by `store_id`
3. Update `TokenData` on the auth token
4. Route to store type: `STORE_TYPE_MAP.get(store.store_type)(store).get_item(request.GET)`
5. Record item scan analytics (success/failure)
6. If item not found and store is `MISHIPAY_STORE_TYPE`: **fall back to globally available stores** — iterates all `globally_available=True` stores, attempting `store.agent.get_item(data)` on each
7. If item not found and store is `MANGO_STORE_TYPE`: check for `not_in_store` error
8. Ensure essential keys exist: `name`, `image`, `variant_title`, `price`, `vendor`
9. Append `store_id` and `outside_geofencing_check` to response

#### `ItemStatusView` (lines 279-297)

**POST**: Returns sold status for items via `store_type.get_sold_status(request.data)`.

#### `GetShoppingBags` (lines 114-144)

**GET**: Returns shopping bag items via `get_shopping_bag_items(store)` from `mishipay_core.generic_interfaces`.

#### `sold_status` (lines 367-400, deprecated)

Legacy RFID sold-status endpoint using Django cache. Accepts a JSON list of UIDs, checks cache (populated from `Item.objects.filter(uid__isnull=False)`), returns `0` (not sold), `1` (sold), or `2` (unknown).

### Payment Views

**File**: `mainServer/dos/views/payment.py` (1215 lines)

This file contains integrations with **9 payment platforms**. Each has card-saving and/or payment processing capabilities.

#### Card Management Pattern

All platforms store cards as JSON arrays in `Customer.source` (demo) or `Customer.live_source` (production). Cards are identified by a combination of `last4`, `exp_month`, `exp_year`, and platform-specific fields.

**`PaymentUpdateView`** (lines 495-571): Generic card save using JudoPay format. Checks for duplicate cards by matching last4 + expiry.

**`PaymentRemoveCardView`** (lines 889-957): **Universal card removal** supporting all platforms. Identifies matching cards using platform-specific fields:
- PayUnity: `registration_id` + `last4`
- Verifone: `registration_id` + `card_pan` + `card_number_hash`
- Adyen: `token` + `last4` + `card_bin`
- Stripe: `token` + `last4` + `card_bin`
- OnePay: `id` + `last4`
- XPay: `id` + `last4`
- JudoPay (default): `id` + `last4`

#### Platform-Specific Views

**JudoPay** (`PaymentInitialize`, `TestPaymentInitialize`, lines 610-976):
- Initializes web payments via `https://gw1.judopay-sandbox.com/webpayments/payments`
- Uses Basic auth with Judo API token + secret
- `payment_success` / `payment_failure`: Redirect handlers that construct client-side callback URLs

**Stripe** (`StripePayment`, lines 699-745):
- No authentication required (handles its own auth via Stripe API keys)
- Three flows: saved card charge, new card with save, new card without save
- Currency hardcoded to EUR
- Calls `create_order_from_stripe()` on success

**PayUnity** (`PayunityCheckout`, `PayunityResult`, `PayunityRemoveCard`, `PayunityAddCard`, lines 748-852):
- Saturn store hardcoded as default (`554e5e0e-7720-4ba7-aa5b-7644b92118c6`)
- `PayunityCheckout`: Creates checkout session via `PAY_UNITY(store).create_checkout(data)`
- `PayunityAddCard`: Registers card via `PAY_UNITY(store).register_card(data)`

**PayPal** (`PaypalClient`, lines 855-886):
- Returns Braintree client token for client-side SDK
- Saturn store hardcoded as default

**XPay** (`XPaySaveCardAPI`, `xpay_notify_save_card`, lines 144-373):
- `XPaySaveCardAPI.post()`: Creates a minimal-amount (€0.01) order, then initiates XPay payment to trigger card save flow
- `xpay_notify_save_card`: Webhook endpoint that receives card details from XPay after save. Extracts card brand via regex, stores card with `platform: XPAY` identifier

**OnePay** (`OnePaySaveCardAPIView`, lines 376-492):
- Two-action endpoint (URL param): `generate-checkout-id` and `save`
- Only supports `DecathlonStoreType`
- Creates minimal order → generates checkout ID → processes payment with `registration: True` → saves card

**Verifone** (`VerifoneSession`, `VerifoneRegisterCard`, lines 979-1025):
- `VerifoneSession.get()`: Returns session GUID + passcode
- `VerifoneRegisterCard.post()`: Registers card from web form session data

**Adyen** (`AdyenSession`, `AdyenRegisterCard`, `AdyenAuthorizeOrder`, lines 1028-1163):
- Configured for Mango, MishiPay, and Lego store types (`store_type_name_map`)
- `AdyenSession.post()`: Creates payment session
- `AdyenRegisterCard.post()`: Verifies payment and saves card
- `AdyenAuthorizeOrder.post()`: Full authorize → adjust → capture flow. On success: updates order status to `CAPTURED`, sends receipt, publishes dashboard update

**Apple Pay** (`ApplePaySession`, lines 1166-1189):
- Creates Apple Pay session via `ApplePay().generate_session()`

**Google Pay** (`GooglePayAPIView`, lines 1192-1215):
- Action-based endpoint (currently only `session` action)
- Creates Google Pay session via `GooglePay(customer, store).generate_session()`

### Gate Views

**File**: `mainServer/dos/views/gate.py` (142 lines)

Gates represent physical exit barriers in stores. Each gate is identified by a `serial_number`.

**`GateInfo`** (GET): Returns gate config. If the gate's serial number isn't registered, creates a new `Gate` record. If the gate is linked to a store, returns all items with their `uid` and `sold` status.

**`GateItems`** (GET): Similar to `GateInfo` but returns items via `store.agent.get_items()` instead of raw DB queries — providing formatted item data rather than raw `uid`/`sold` tuples.

Both function-based (`get_gate_info`, `get_gate_items`) and class-based (`GateInfo`, `GateItems`) versions exist with identical logic.

### Coupon Views

**File**: `mainServer/dos/views/coupon.py` (30 lines)

Single function `get_coupon_details(request)`:
- Looks up `Coupon` by `coupon_code` + `store_id`
- Returns `model_to_dict(coupon)` on success

### Socket Views

**File**: `mainServer/dos/views/socket.py` (36 lines)

Both function-based and class-based versions. Attempts to connect to a TCP socket at `address:port` (from query params, defaulting to `localhost:8888`). Returns status message. Used for health-checking socket-based services.

### Ingenico Views

**File**: `mainServer/dos/views/ingenico.py` (72 lines)

`ingenico_redirect_url(request)`: Creates Ingenico hosted checkout session:
1. Get customer/store from session
2. Initialize Ingenico SDK client from config file
3. Create `HostedCheckoutRequest` with amount (from `cost` param, multiplied by 100 for cents)
4. Currency hardcoded to USD, country to US
5. Return redirect URL

### App Version Views

**File**: `mainServer/dos/views/app_specific_views.py` (73 lines)

`AppForceUpdateStatus`:
- **GET**: Check if a specific app version requires force update. Params: `app_type`, `platform`, `current_version`. Returns `{force_update: bool, url: string}`.
- **POST**: List all force-update versions grouped by platform (android/ios/web) and app type.

---

## Analytics Integration

**Directory**: `mainServer/dos/analytics/` (7 files)

All analytics functions follow the same pattern:
1. Extract common parameters from request via `set_params_for_events(request)`
2. Add event-specific parameters
3. Call `client.AnalyticsRecordClient().record_event(event_name, params=params)` — sending to the analytics microservice

### Analytics Events

| Event | File | Trigger |
|-------|------|---------|
| `record_login` | `customer_analytics.py` | Email/social sign-in |
| `record_signup` | `customer_analytics.py` | Email/social registration |
| `record_guest_email` | `customer_analytics.py` | Guest email capture |
| `record_app_open` | `customer_analytics.py` | App opened |
| `record_item_scan` | `items_analytics.py` | Barcode scanned (success/failure) |
| `record_create_order` | `order_analytics.py` | Order created |
| `record_verified_order` | `order_update_analytics.py` | Order verified at exit |
| `record_payment_captured` | `payment_analytics.py` | Payment processed |
| `record_rating` | `feedback_analytics.py` | Customer feedback submitted |
| `record_refund_order` | `refund_order_analytics.py` | Refund processed |

### Common Analytics Parameters

Every event includes:
- `mish_id` — Customer identifier
- `store_id`, `store_type`
- `latitude`, `longitude`
- `platform`, `network`, `device`
- `bundle_id`
- `outside_geofencing_check`

---

## Test Coverage

**Directory**: `mainServer/dos/tests/` (18 files)

### Test Files

| File | Coverage Area |
|------|---------------|
| `factories.py` | Test data factories for models |
| `test_models.py` | Model validation and constraints |
| `test_views.py` | General view integration tests |
| `test_apple_authenticate.py` | Apple Sign-In flow |
| `test_auto_registeration.py` | Auto-registration (guest → registered) — comprehensive (14.9KB) |
| `test_oauth.py` | OAuth authentication flows |
| `test_otp.py` | OTP generation and verification (7.2KB) |
| `utility_test.py` | Utility function tests |
| `_test_global_config.py` | Global config endpoint |
| `_test_items.py` | Item scan/lookup tests (15.4KB — most comprehensive) |
| `_test_stores.py` | Store discovery tests |
| `_test_view_coupon.py` | Coupon lookup tests |
| `_test_view_customer.py` | Customer view tests (6.9KB) |
| `_test_view_gate.py` | Gate integration tests |
| `_test_view_item.py` | Item view tests (4KB) |
| `_test_view_order.py` | Order creation/retrieval tests |
| `_test_view_store.py` | Store view tests |

Files prefixed with `_test_` follow a different naming convention — likely older tests or tests that may be excluded from default test runs.

---

## Agent vs StoreType Clarification

From the views layer, the relationship between agents and store types is now clear:

1. **StoreType** is the **primary dispatch mechanism** used by class-based views. Views call `STORE_TYPE_MAP.get(store.store_type)(store)` to get a store type instance, then call methods like `get_item()`, `create_order()`, `confirm_order()`.

2. **Agents** are used by **legacy function-based views** and as a **fallback** within views. Views call `store.agent.get_item()` when `store.store_type` is falsy (empty/null). The agent is accessed via a property on the Store model.

3. In the `ItemView` (the most complex item endpoint), the logic is:
   ```python
   if store.store_type:
       store_type = STORE_TYPE_MAP.get(store.store_type)(store)
       item = store_type.get_item(request.GET)
   else:
       item = store.agent.get_item(data)
   ```

This confirms that **agents are the older pattern**, and store types are the newer, more feature-complete abstraction. Stores without a `store_type` fall back to agent-based item lookup.

---

## Notable Patterns (Views-Specific)

1. **Dual endpoint pattern**: Many endpoints exist in both function-based (legacy) and class-based (v2) forms with identical logic. Examples: `get_gate_info` / `GateInfo`, `get_item` / `ItemView`, `process_order` / `OrderView`.

2. **Hardcoded store IDs**: Several payment views hardcode Saturn's store ID (`554e5e0e-7720-4ba7-aa5b-7644b92118c6`) as a default. PayUnity and PayPal views fall back to this ID if no `store_id` is provided.

3. **Two-phase payment flow**: XPay and OnePay use a create-order → redirect/callback → confirm-order pattern. The `OrderConfirm` view's GET handler processes redirect callbacks with an `identification_token` encoding `order_id + verification_token`.

4. **Global store fallback**: Item scanning (`ItemView`) searches globally-available stores when an item isn't found in the customer's current store, but only for `MISHIPAY_STORE_TYPE`.

5. **Per-store-type order filtering**: `OrderView.get()` applies different status filters based on store type — Saturn shows verification statuses, Decathlon/Lego only show completed, others show all.

6. **Card storage as JSON strings**: Payment cards are stored as JSON-serialized arrays in `Customer.source`/`live_source` text fields, with per-platform deduplication logic in `PaymentRemoveCardView`.

7. **`@transaction.atomic` everywhere**: Nearly every view method is wrapped in `@transaction.atomic`, even read-only GET endpoints. This provides rollback safety but may impact performance under high concurrency.

8. **Deactivated user detection**: Sign-in and social auth flows check for deactivated accounts by hashing the email with SHA-1 and comparing against stored email values — deactivated users have their email replaced with the hash.

9. **`per_store` flag overridden**: In `OrderView.get()` at line 257, `per_store` is immediately set to `0` regardless of the query parameter, effectively disabling per-store filtering.

---

## Open Questions (Updated)

1. ~~**Agent vs StoreType overlap**~~ — **Resolved**: Agents are the older pattern; store types are the primary dispatch. Stores without `store_type` fall back to agents.

2. **PayUnity/PayPal hardcoded store ID**: The Saturn store UUID `554e5e0e-7720-4ba7-aa5b-7644b92118c6` is hardcoded as a default in multiple payment views. This suggests these payment flows were originally Saturn-only and may not work correctly for other stores without explicit `store_id`.

3. **`per_store` flag disabled**: `OrderView.get()` sets `per_store = 0` after reading it from the request, making the query parameter ineffective. It's unclear if this is intentional (perhaps to fix a bug) or an oversight.

4. **`PayunityResult` is a placeholder**: The `PayunityResult.get()` endpoint just returns `success_message("something")` — it doesn't process any actual payment result data.

5. **`AppEvent` doesn't persist**: The `AppEvent.post()` view constructs an event params dict but never sends it to analytics or saves it. It just returns `success_message('success')`.

6. **`AppForceUpdateStatus.get_response_format` has incorrect signature**: The static method at line 44 accepts `self` as a parameter despite being decorated with `@staticmethod`, which would cause a runtime error when called.

7. **Test file naming inconsistency**: Test files use two conventions — `test_*.py` and `_test_*.py`. The underscore-prefixed files may be excluded from automatic test discovery by default.

8. **Missing `tasks.py`**: The task definition in `tasks.json` lists `dos/tasks.py` as a source file, but no such file exists in the module. There are no Celery or background task definitions in the DOS module.

---
---

# Part 3: Retailer-Specific Modules (Dixons, Decathlon, Leroy Merlin, Cisco, Lego, Mango)

> Added by: Task T26 — Smaller retailer-specific modules documented with purpose, scope, customizations, and common patterns

This section documents the six dedicated retailer modules that extend the `dos.StoreType` base class. Unlike the ~91 newer retailers that use generic/configurable store types resolved dynamically, these six (along with Saturn, documented elsewhere) have their own Python modules with custom behaviour.

---

## Overview: Retailer Module Pattern

All six modules follow the same architectural pattern:

1. **Each module contains a `StoreType` subclass** that extends `dos.base_store_type.StoreType`
2. **The subclass is registered** in `dos/store_map.py` as a direct class import
3. **Methods override** the base class interface: `get_item()`, `create_order()`, `refund_order()`, `send_receipt()`, etc.
4. **External APIs** are called for product data (Leroy Merlin, Decathlon, Lego, Mango, Cisco/Clover) — most items are not stored locally
5. **DOS module utilities** (`order_helper`, `publish_order`, `send_receipt`, `error_message`) are used heavily

### Module Size Comparison

| Module | Files | Lines (core) | Models | Payment Gateway | Item Source | Status |
|--------|-------|-------------|--------|----------------|-------------|--------|
| **dixons** | 6 | 765 | `Item` (local DB) | Stripe, Verifone, Judopay | Local DB | Active |
| **decathlon** | 7 | 1,306 | None (uses dos.Order) | OnePay | Decathlon API | Active |
| **leroy_merlin** | 2 | 362 | None | Stripe | Leroy Merlin API | Active |
| **cisco** | 12 | ~450 | `ItemMap` | Spreedly → Clover | Clover POS API | Legacy (ngrok URLs) |
| **lego** | 2 | 395 | None | Adyen | Lego API | Active |
| **mango** | 4 | 1,179 | None (commented out) | Adyen | Mango API (disabled) | Partially disabled |

---

## Dixons Module

**Location**: `mainServer/dixons/` (6 files, 824 lines total)
**Class**: `DixonsStoreType` — `mainServer/dixons/store_type.py:34` (765 lines)

### Purpose

Dixons (Currys/PC World parent company) integration with local item storage, RFID tracking, and multi-gateway payment support. One of the earliest retailer integrations — the initial migration dates to 2018-08-29 (Django 1.11.5).

### Data Model

**`Item`** (`models.py:8-34`) — Extends `SecureItem` from `dos.models.base`

| Field | Type | Description |
|-------|------|-------------|
| `barcode` | `CharField(63)` | Product identifier |
| `store` | `ForeignKey(Store)` | Store reference, related_name `'dixons_store'` |
| `category` | `CharField(63)` | Product category |
| `item_name` | `CharField(511)` | Display name |
| `vendor` | `CharField(63)` | Supplier name |
| `img` | `URLField` | Product image (default: Cloudinary placeholder) |
| `price` | `FloatField` | Price (default: 1.0) |
| `story` | `TextField` | Product description |
| `created_date` | `DateTimeField` | Auto-set on creation |
| `modified_date` | `DateTimeField` | Auto-set on save |
| `quantity` | `IntegerField` | Stock quantity (default: 1) |
| `product_id` | `CharField(63)` | External product identifier |

**Inherited from SecureItem**: `uid` (RFID/EPC), `sold` (boolean)

**Constraints**: Unique `(barcode, store)`. Index `identifier_dixons_item` on `(barcode, store)`.

**Forms**: `ItemUpdateForm` (excludes `store_id`, `barcode`, `tax`), `ItemCreationForm` (explicit 10-field whitelist).

**Admin**: `ItemAdmin` registered with 13-column `list_display`.

### Key Methods

| Method | Lines | Purpose |
|--------|-------|---------|
| `get_item(data)` | 47-72 | Barcode lookup (case-insensitive, sanitized alphanumeric) |
| `get_items(data)` | 75-95 | Paginated listing (page size 10), optional category filter |
| `add_item(data)` | 103-160 | Create item with RFID validation (qty must be 0 or 1 for UID items) |
| `update_item(data)` | 175-232 | Update with Redis sold-status sync |
| `delete_item(data)` | 235-259 | Remove item from store |
| `set_sold(data)` | 453-496 | Decrement quantity, mark sold at qty=0, Redis `uid→1` |
| `set_unsold(data)` | 610-637 | Mark unsold, Redis `uid→0` |
| `update_unsold_status(item)` | 640-662 | Restore inventory after refund, handles EPC URL-encoded barcodes |
| `create_order(data)` | 282-380 | Multi-platform order: Stripe/Judopay/Verifone, mark items sold, send receipt |
| `pay(data, platform)` | 383-450 | Stripe direct charge or Verifone session/token-based payment |
| `refund_order(data)` | 499-607 | Full/partial refund with per-item tracking, Stripe refund or backward-compat |
| `apply_promotions(data)` | 262-279 | **Stub** — always returns `NO_PROMO` with benefit=0 |
| `get_sales_report(data)` | 709-727 | Query `SalesSummary` by date range |

### Payment Support

- **Stripe**: Direct charge, saved card charge, refunds via `StripePay`
- **Verifone**: Session-based and token-based transactions, card registration
- **Judopay**: Legacy backward compatibility via `order_helper`

### RFID Tracking

Items with `uid` have strict constraints: quantity must be 0 or 1, Redis tracks sold status (`uid→0/1`), and `publish_item()` broadcasts state changes via MQTT.

### Notable Issues

1. **Orphaned Exception objects** (`store_type.py:305,312,317`): `Exception(...)` created but not raised — validation errors silently discarded during order creation
2. **Stock validation disabled** (`store_type.py:307-308`): Stock availability check commented out in `create_order()`
3. **Debug print statements**: Multiple `print()` calls throughout (lines 293, 413-415, 422-429, 760, 764)
4. **Missing store filter in `get_sold_status()`** (`store_type.py:704`): `Item.objects.filter(barcode__in=barcodes)` queries without store filter — could return items from other stores
5. **`send_receipt()` raises `NotImplementedError`**: Dixons has no receipt implementation

---

## Decathlon Module

**Location**: `mainServer/decathlon/` (4 Python files + 3 HTML email templates)
**Class**: `DecathlonStoreType` — `mainServer/decathlon/store_type.py:42` (1,306 lines)

### Purpose

Full Decathlon retail integration including product catalog API (dual fallback endpoints), order fulfillment to Decathlon POS, RFID tag deactivation via GS1 Databar → EPC binary conversion, and Dutch-language receipt emails. The most feature-complete of the smaller retailer modules.

### Architecture

```
DecathlonStoreType
├── Product APIs (Type 1 + Type 2 with fallback)
├── Order Management (OnePay payment)
├── RFID Tag Management (GS1 → EPC decode → deactivate)
├── POS Fulfillment (REST to Decathlon back-office)
└── Receipt Emails (Dutch templates)
```

### Key Methods

| Method | Lines | Purpose |
|--------|-------|---------|
| `get_token()` | 53-74 | OAuth2 access token from Decathlon API |
| `_get_ean(barcode)` | 76-87 | Extract EAN from 13-digit, 12-digit (UPC→EAN), or 16+ digit (GS1 Databar) |
| `_get_ami_from_ean(code)` | 89-102 | Extract Article Master ID from EAN (prefixes "200", "203", "21") |
| `_get_id_values(token, field, value)` | 104-141 | Fetch product article/model metadata from Decathlon API |
| `_get_item_type_1(token, model, barcode, ean, serial)` | 143-232 | Product details from Type 1 API (primary) |
| `_get_item_type_2(token, model, barcode, ean, serial)` | 234-328 | Product details from Type 2 API (fallback) |
| `get_hard_coded_item_details()` | 330-389 | 3 hardcoded shopping bags (bypass API) |
| `get_item(data)` | 391-485 | Main entry: sanitize → check hardcoded → get token → branch by format → Type 1 → Type 2 |
| `create_order(data)` | 487-566 | Create order with OnePay payment session |
| `confirm_order(data)` | 568-677 | Verify payment, implement RFID, fulfill to POS, send receipt |
| `make_order_fulfilment_request(order)` | 679-809 | Send transaction to Decathlon POS (tax mapping, loyalty number, articles) |
| `implement_rfid(order)` | 811-857 | Orchestrate: decode EPC → deactivate tags |
| `decode_epc_for_rfid(order)` | 898-959 | GS1 Databar → 94-bit EPC binary: company prefix (20 bits) + middle (24 bits) + serial (38 bits) + header |
| `deactivate_epc_for_rfid(token, epc_list)` | 961-988 | POST sgtins to Decathlon RFID system |
| `activate_epc_for_rfid(token, epc_list)` | 990-1020 | Activate tags (**defined but never called**) |
| `send_receipt(order)` | 1029-1099 | Dutch email receipt (`'Bevestiging van betaling'`) |
| `refund_order(data)` | 1101-1214 | Full/partial refund via OnePay with async status tracking |
| `update_refund_status(store, order, txn_id)` | 1238-1305 | Module-level function — check refund status, merge items, trigger POS refund |

### Product Lookup Flow

1. Check 3 hardcoded shopping bag barcodes (bypass API)
2. Get OAuth token from Decathlon
3. If barcode chars 2-4 == "02": EAN → AMI → article_id lookup
4. Otherwise: EAN → model lookup via `product_model_url`
5. Try Type 1 API, then Type 2 API (fallback)

### RFID Tag Workflow

GS1 Databar → EPC conversion algorithm (`decode_epc_for_rfid`, lines 912-941):
1. Validate barcode starts with "01" and has "21" at positions 16-18
2. Extract: filler_bit (pos 2), EAN (pos 3-16), serial (pos 19+)
3. Split EAN: company_prefix (6 chars), middle (7 chars)
4. Convert to binary: step1 (company prefix), step2 (filler + middle, 24 bits), step3 (serial, 38 bits)
5. Concatenate: `EPC_HEADER_BITS` + padding + step1 + step2 + step3 = 94-bit EPC
6. Convert to hex, POST to Decathlon RFID deactivation endpoint

### POS Fulfillment

Tax code mapping (hardcoded, `store_type.py:683-687`): `"1"→21%`, `"3"→9%`, `"8"→0%`

Customer identification: First tries Decathlon loyalty number from `ThirdPartyLogin(provider=PROVIDER_DECATHLON)`, falls back to internal `cus_id`.

### Migration Script

`management/commands/decathlon_migration_script_order_payment.py` (211 lines) — migrates orders from `dos.Order` to `mishipay_retail_orders.Order` with field mapping, payment normalization, and iDEAL bank extraction.

### Configuration

All settings loaded from `settings.STORE_DETAILS['decathlon'][demo|live][region][sub_region]` — includes product API URLs/keys, fulfillment endpoints, RFID service configuration, and timezone offset.

### Notable Issues

1. **`activate_epc_for_rfid()` defined but never called** — RFID reactivation for returns not implemented
2. **Hardcoded tax rates** (21%, 9%, 0%) — new rates require code change
3. **Silent email failures** — `send_receipt()` catches all exceptions and logs but doesn't propagate
4. **Synchronous RFID calls** block order confirmation
5. **Redis connection to `gates.mishipay.com:6379`** — hardcoded external host

---

## Leroy Merlin Module

**Location**: `mainServer/leroy_merlin/` (2 files)
**Class**: `LeroyMerlinStoreType` — `mainServer/leroy_merlin/store_type.py:24` (362 lines)

### Purpose

Read-only product integration with Leroy Merlin's French product API. Items exist only in Leroy Merlin's system — all inventory management methods raise `NotImplementedError`. The simplest of the retailer modules.

### Key Methods

| Method | Lines | Purpose |
|--------|-------|---------|
| `get_item(data)` | 35-74 | GET from `api.leroymerlin.fr`, returns item dict |
| `set_sold(data)` | 76-85 | **No-op** — returns `True` |
| `set_unsold(data)` | 87-96 | **No-op** — returns `True` |
| `get_items(data)` | 98-107 | **NotImplementedError** |
| `add_item(data)` | 109-118 | **NotImplementedError** |
| `add_item_by_csv(data)` | 120-129 | **NotImplementedError** |
| `update_item(data)` | 131-140 | **NotImplementedError** |
| `delete_item(data)` | 142-151 | **NotImplementedError** |
| `apply_promotions(data)` | 153-169 | **Stub** — always returns `NO_PROMO` with benefit=0 |
| `create_order(data)` | 171-202 | Create via `order_helper`, mark sold, send receipt, publish |
| `pay(data)` | 204-214 | **NotImplementedError** (TODO) |
| `refund_order(data)` | 216-337 | Full/partial refund with Stripe, barcode normalization |
| `send_receipt(data)` | 352-361 | **NotImplementedError** |

### Item Retrieval

- **API URL**: `https://api.leroymerlin.fr/backend-amgp-lmfr/v2/products/repository?ean={barcode}&storeId=11`
- **API Key**: `jOz8NzN0THYj9JAKqibDH0DTcx73moJZ` (hardcoded in source, `store_type.py:50`)
- **Store ID**: Hardcoded to `11` (single French location)
- **Vendor**: Always `'Leroy Merlin'`

### Refund Logic

Supports full and partial refunds with per-item quantity tracking. Normalizes barcodes (lowercase, alphanumeric only) for matching. Uses `StripePay.make_refund()` for Stripe orders, with backward compatibility for older Judopay orders.

### Notable Issues

1. **Hardcoded API key in source code** (`store_type.py:50`) — security risk
2. **Hardcoded store ID `11`** — not configurable for multi-store deployments
3. **Redis `localhost:6379`** — different from other modules that use `settings.REDIS_HOST` or `gates.mishipay.com`
4. **7 methods raise `NotImplementedError`** — read-only integration, items not managed locally

---

## Cisco Module

**Location**: `mainServer/cisco/` (12 files, ~450 lines of core code)
**Class**: `CiscoStoreType` — `mainServer/cisco/store_type.py:22` (346 lines)

### Purpose

Cisco retail store integration using **Clover POS** for inventory and orders, **Spreedly** for payment tokenization/delivery, and **SATO** for RFID shelf monitoring. Designed for Cisco's physical retail stores where products are tracked with QR/RFID tags.

### Architecture

```
CiscoStoreType
├── CloverPOS (clover.py) — Clover REST API client
├── SpreedlyPayment (spreedly.py) — Payment tokenization
├── ItemMap (models.py) — Maps Clover product IDs to codes
└── SATO Integration — RFID item tracking (ngrok URLs)
```

### Data Model

**`ItemMap`** (`models.py:5-13`)

| Field | Type | Description |
|-------|------|-------------|
| `clover_product_id` | `CharField(63)`, unique | Clover POS product ID |
| `product_code` | `CharField(63)` | Internal product code |
| `img` | `URLField(2047)` | Product image (default: Cloudinary placeholder) |
| `quantity` | `IntegerField` | Stock count (default: 0) |
| `sold` | `BooleanField` | Sold status (default: False) |

**Admin**: `ItemMap` registered with default admin.

### CloverPOS Client (`clover.py`, 192 lines)

| Method | Purpose |
|--------|---------|
| `get_item(item_id)` | GET `/v3/merchants/{id}/items/{item_id}` |
| `create_order(data)` | POST `/v3/merchants/{id}/orders` |
| `add_bulk_line_items(order_id, data)` | POST `/v3/merchants/{id}/orders/{order_id}/bulk_line_items` |
| `get_payment_key()` | GET `/v2/merchant/{id}/pay/key` — encryption key for payments |
| `add_to_item_map(data)` | Bulk insert/update `ItemMap` from Clover items |
| `inventory_items_update()` | Fetch incremental inventory changes from last 1 hour |

### SpreedlyPayment Client (`spreedly.py`, 96 lines)

| Method | Purpose |
|--------|---------|
| `create_receiver()` | POST `/v1/receivers.json` — create payment receiver |
| `create_delivery(data)` | POST `/v1/receivers/{token}/deliver.json` — deliver payment to Clover |
| `get_payment_method(token)` | GET `/v1/payment_methods/{token}.json` — fetch card details |

### Key Methods

| Method | Lines | Purpose |
|--------|-------|---------|
| `deconstruct_epc(epc)` | 44-60 | Convert hex EPC → binary → extract bits 32-56 as product code |
| `get_item(data)` | 62-103 | Detect `httpsciscomishipaycom` QR format → decode EPC → ItemMap → Clover API |
| `set_sold(data)` | 105-125 | Mark sold in ItemMap + Redis + SATO |
| `set_unsold(data)` | 127-146 | Mark unsold in ItemMap + Redis |
| `create_order_in_clover(data)` | 148-175 | Create Clover order with line items (prices ÷100) |
| `create_order(data)` | 177-254 | Full flow: Clover order → Spreedly payment → DOS order → mark sold → receipt |
| `get_token()` | 272-289 | SATO auth: POST `https://cs-b11.ngrok.io/CoreServices/public/users/sessions` |
| `set_sold_in_sato(uid)` | 291-308 | PATCH SATO item status → 'sold', readpoint → 'check-out' |
| `set_item_to_unsold(data)` | 310-330 | Two PATCH calls: status → 'returned' then → 'sellable' |
| `check_if_item_is_sellable(epc)` | 332-345 | GET SATO item existence check |

### Item Data Flow

```
Scanner → QR Code (https://cisco.mishipay.com/{EPC_HEX})
  → CiscoStoreType.get_item()
    → deconstruct_epc() → product_code
    → ItemMap.objects.get(product_code=...)
    → CloverPOS.get_item(clover_product_id)
  → Return item dict with hardcoded image URL: https://www.newgen.tv/cisco/sanjose/{code}.jpg
```

### Configuration

Clover: `CLOVER_DEMO_URL`/`CLOVER_LIVE_URL`, `CLOVER_DEMO_ACCESS_TOKEN`/`CLOVER_ACCESS_TOKEN`, `CLOVER_DEMO_MERCHANT_NAME`/`CLOVER_MERCHANT_NAME`.

Spreedly: `SPREEDLY_ENV_DEMO_KEY`/`SPREEDLY_ENV_LIVE_KEY`, `SPREEDLY_DEMO_ACCESS_SECRET`/`SPREEDLY_LIVE_ACCESS_SECRET`, `SPREEDLY_DEMO_RECEIVER_TOKEN`/`SPREEDLY_LIVE_RECEIVER_TOKEN`.

### Notable Issues

1. **SATO URLs are ngrok tunnels** (`store_type.py:278,300,312,333`): `https://cs-b11.ngrok.io/CoreServices/...` — development-only URLs in production code
2. **Hardcoded SATO credentials** (`store_type.py:278`): `username=mishipay&password=3%235dc31875` in source
3. **Hardcoded image server** (`store_type.py:94`): `https://www.newgen.tv/cisco/sanjose/{code}.jpg`
4. **Debug log messages** (`clover.py:84-88`): `"REACHED HERE: PRE CLOVER"`, `"REACHED HERE: POST CLOVER"`
5. **Unreachable code** (`store_type.py:345`): `raise ValueError(...)` after `return True` in `check_if_item_is_sellable()`
6. **No retry logic** on any external API calls (Clover, Spreedly, SATO)
7. **Payment delivery body uses string formatting** (`store_type.py:201-214`) — fragile, potential injection risk
8. **Migration conflict artifacts**: Two `0003_auto_*` migrations merged via `0004_merge_*`

---

## Lego Module

**Location**: `mainServer/lego/` (2 files)
**Class**: `LegoStoreType` — `mainServer/lego/store_type.py:33` (395 lines)

### Purpose

Lego retail integration with product data from Lego's barcode API, Adyen payment processing (including Apple Pay and saved cards), and English-language receipt emails. No local item storage.

### Key Methods

| Method | Lines | Purpose |
|--------|-------|---------|
| `get_item(data)` | 42-94 | Query Lego API by barcode with market/language params, extract images with width params |
| `create_order(data)` | 96-206 | Three-branch payment flow: Apple Pay / saved card / new card (Adyen session) |
| `confirm_order(data)` | 208-319 | Authorize (saved card/Apple Pay/payload) → capture → receipt → publish |
| `verify_order(order, status)` | 321-326 | Set `ORDER_CORRECT` / `ORDER_INCORRECT` |
| `send_receipt(order)` | 328-394 | Render `email/lego_receipt.html`, send from `info@mishipay.com` |

### Item Retrieval

- **API**: `self.store_settings['item_fetch_api_url']` with params `AJAXSearchBarcode`, `barcode`, `market`, `language`
- **Response fields**: `productBarcode`, `title`, `price`, `itemID`, `productBullets`
- **Images**: Multiple image fields (`ImageN` pattern) with configurable width (`?width=750` primary, `?width=320` thumbnail)

### Payment Flow

```
create_order()
├── Apple Pay → confirm_order() immediately
├── Saved Card → get_card_details() → confirm_order()
└── New Card → AdyenPay.generate_session() → return session to client
                ↓ (client authorizes)
confirm_order()
├── Saved Card / Apple Pay → AdyenPay.pay_saved_card()
├── New Card → AdyenPay.verify_payment()
└── → AdyenPay.capture_payment() → check '[capture-received]'
    → PAYMENT_COMPLETED → send_receipt() + publish_order()
```

### Configuration

All settings from `settings.STORE_DETAILS['lego'][demo|live][region]` — includes `item_fetch_api_url`, `item_market`, `item_language`, `item_image_host`.

### Notable Issues

1. **Inconsistent payment processor string** (`store_type.py:226`): Uses `'mango'` instead of `'lego'` when creating `AdyenPay` in `confirm_order()` — may cause routing issues
2. **Hardcoded 19% VAT** (`store_type.py:351`): `price - (price * 100) / (100 + 19)` — not configurable per region
3. **Redis to `gates.mishipay.com:6379`** — hardcoded host, same as Decathlon
4. **No retry or timeout** on Lego API or Adyen calls
5. **Error messages returned to client** — exception strings may leak sensitive info

---

## Mango Module

**Location**: `mainServer/mango/` (4 files, 1,179 lines in `store_type.py`)
**Class**: `MangoStoreType` — `mainServer/mango/store_type.py:42` (1,179 lines)

### Purpose

Mango fashion retailer integration with XML-based product API, Adyen payment processing with authorize/capture/cancel workflow, bilingual receipt emails (English/Spanish), and detailed transaction log emails to Mango staff.

### Architecture

**Critical Finding**: The item fetch is **disabled**. `fetch_mango_item()` at line 85 has an early return `return {"error": "MP0007"}` before any API logic, making 95 lines of Mango API code unreachable.

```
MangoStoreType
├── get_item() → fetch_mango_item() → DISABLED (returns MP0007 error)
├── Inventory CRUD (uses mishipay.models.Item locally)
├── Order Management (Adyen authorize/capture/cancel)
├── Refund Processing (Adyen refund)
└── Receipts (EN/ES) + Transaction Logs to Mango staff
```

### Item Source

The module uses `mishipay.models.Item` (imported from the `mishipay` base module) for local item storage. The `mango/models.py` file contains a **commented-out** Mango-specific `Item` model that would have extended `SecureItem` with fashion-specific fields (`color_id`, `color_name`, `short_name`, `final_price`, `discounted_price`, `family`, `care_desc`, `care_icons`, `website_url`). The admin registration is similarly commented out.

### Disabled Mango API (`fetch_mango_item`, lines 68-180)

The **unreachable** logic (after line 85's early return) would:
1. Determine country code: UK → `'006'`, EU → `'001'`
2. Determine language: English → `'IN'`, else → `'ES'`
3. GET `https://www.mango.com/web/app/iphone.php?accion=ean&ean13={barcode}&idioma={lang}&pais={country}&v=3`
4. Parse XML response via `xmltodict`
5. Extract: name, price, colors/sizes, description, care instructions, images
6. Track customer first-scan notification

**Side effect that IS executed** (lines 77-84): Updates `customer.first_scan` flag and publishes thank-you notification.

### Key Methods

| Method | Lines | Purpose |
|--------|-------|---------|
| `get_item(data)` | 53-66 | Calls `fetch_mango_item()` — **always returns error** |
| `add_item(data)` | 262-331 | Create item with RFID validation (qty 0/1 for UID items) |
| `update_item(data)` | 344-403 | Update with Redis sold-status sync |
| `delete_item(data)` | 405-431 | Remove item from store |
| `apply_promotions(data)` | 433-450 | **Stub** — always returns `NO_PROMO` |
| `create_order(data)` | 452-511 | Pay via Adyen → AUTHORIZED status → create order → publish |
| `pay(data, store)` | 513-526 | Dispatch to `AdyenPay.pay_saved_card()` or `verify_payment()` |
| `authorize_order(data, order, store_id)` | 528-592 | Dashboard: adjust authorization amount or capture payment |
| `cancel_order(data, order, store_id)` | 594-630 | Dashboard: cancel authorized order via Adyen |
| `refund_order(data)` | 632-697 | Full/partial refund via Adyen with item tracking |
| `send_receipt(order, lang, store)` | 913-986 | Bilingual receipt (EN: `confirmation-en.html`, ES: `confirmation-es.html`) |
| `send_transaction_log(order, store)` | 988-1086 | Staff transaction log with today's summary stats |
| `send_cancelled_receipt(order, store_id)` | 1088-1155 | Cancellation email to customer |
| `get_transactions_summary(store)` | 1157-1178 | Today's order count, item count, total amount (CAPTURED only) |

### Order Lifecycle

Mango uses a **three-stage authorize/capture** flow:

```
create_order() → status: AUTHORIZED
  ↓
authorize_order(adjust_auth=False) → AdyenPay.capture_payment()
  → status: CAPTURED
  → send_receipt() + send_transaction_log() + publish_order()

OR

authorize_order(adjust_auth=True) → status: PENDING_AUTH
  → publish customer update requesting re-authorization
  → customer re-authorizes → capture

OR

cancel_order() → AdyenPay.cancel_payment()
  → status: CANCELLED
  → send_cancelled_receipt()
```

### Transaction Log Emails

After each receipt, `send_transaction_log()` sends a detailed email to Mango staff:
- **Demo**: `MANGO_DEV_EMAIL_CC` env var
- **Production**: `MANGO_PROD_EMAIL_CC` + region-specific: `tda00869@mango.com` (UK) or `tda01832@mango.com` (EU/ES)

Includes today's aggregate stats: `number_transactions`, `number_items`, `total_amount`.

### Notable Issues

1. **Item fetch disabled** (`store_type.py:85`): `return {"error": "MP0007"}` — Mango API integration is non-functional
2. **Commented-out model and admin** (`models.py`, `admin.py`) — entire data model disabled
3. **Dead refund code** (lines 699-804): Old Stripe-based refund implementation commented out, replaced by Adyen
4. **Undefined variable** (`store_type.py:188`): `color_image_url` not initialized in `get_color_image()` if `image_url` is empty
5. **Wildcard import** (`store_type.py:26`): `from dos.util import *`
6. **`type(x) is str`** used instead of `isinstance()` (lines 194, 224, 1017)
7. **Hardcoded Mango staff emails** (lines 1052, 1054)

---

## Common Patterns Across Retailer Modules

### 1. Barcode Sanitization

All modules use the same alphanumeric-only filter:
```python
''.join(filter(lambda c: c in string.ascii_letters + string.digits, barcode))
```
Applied in: Dixons, Leroy Merlin, Cisco, Mango. Decathlon and Lego use similar sanitization.

### 2. Promotions Stub

All six modules return a stub response for `apply_promotions()`:
```python
[{'items': parsed_items, 'benefit_value': 0, 'description': 'NO_PROMO'}]
```
Promotions for these retailers are either handled elsewhere (promotions microservice) or not supported.

### 3. DOS Utility Dependency

Every module depends on the same `dos` utilities:
- `order_helper.create_order(data, store)` — order creation
- `publish_order(order)` — MQTT dashboard notification
- `send_receipt(order)` — email receipt (some override, some use default)
- `error_message(name, msg, status)` — standard error response format

### 4. Redis RFID State Tracking

Modules with RFID support (Dixons, Cisco, Mango) all use the same pattern:
```python
self.r = redis.StrictRedis(host=..., port=6379, db=0)
self.r.set(uid, 1)  # sold
self.r.set(uid, 0)  # unsold
```
But Redis hosts differ: `settings.REDIS_HOST` (Dixons, Mango), `gates.mishipay.com` (Decathlon, Lego), `localhost` (Leroy Merlin).

### 5. Payment Gateway Diversity

| Module | Payment Gateways |
|--------|-----------------|
| Dixons | Stripe, Verifone, Judopay |
| Decathlon | OnePay |
| Leroy Merlin | Stripe |
| Cisco | Spreedly → Clover |
| Lego | Adyen |
| Mango | Adyen |

No two retailers (except Lego/Mango) share the same primary payment gateway.

### 6. Receipt Email Approaches

| Module | Receipt Implementation |
|--------|----------------------|
| Dixons | `NotImplementedError` (no receipts) |
| Decathlon | Custom Dutch template (`decathlon_update_receipt.html`) |
| Leroy Merlin | `NotImplementedError` (uses default `send_receipt` from dos) |
| Cisco | Delegates to `dos.util.send_receipt()` |
| Lego | Custom English template (`lego_receipt.html`) |
| Mango | Custom bilingual EN/ES templates + staff transaction log |

### 7. Tax/VAT Hardcoding

Decathlon hardcodes `"1"→21%, "3"→9%, "8"→0%`. Lego hardcodes 19%. Mango has no VAT calculation visible. None use configurable tax rates.

---

## Integration Points with Core Platform

All six modules integrate with the core platform through the same interfaces:

```
dos/store_map.py
├── "DixonsStoreType"    → dixons.store_type.DixonsStoreType
├── "DecathlonStoreType" → decathlon.store_type.DecathlonStoreType
├── "LeroyMerlinStoreType" → leroy_merlin.store_type.LeroyMerlinStoreType
├── "CiscoStoreType"     → cisco.store_type.CiscoStoreType
├── "LegoStoreType"      → lego.store_type.LegoStoreType
└── "MangoStoreType"     → mango.store_type.MangoStoreType
```

The `dos/views/item.py:ItemView` dispatches to these via:
```python
store_type = STORE_TYPE_MAP.get(store.store_type)(store)
item = store_type.get_item(request.GET)
```

---

## Open Questions (T26)

1. **Mango API disabled**: Why is `fetch_mango_item()` returning `"MP0007"` on every call? Has the Mango API integration been replaced by inventory service or another mechanism? The commented-out model suggests a larger architectural change.

2. **SATO ngrok URLs in Cisco**: The `https://cs-b11.ngrok.io` URLs are development-only tunnels. Is the Cisco SATO integration still active in production, or is it legacy/disabled?

3. **Lego `'mango'` string**: `LegoStoreType.confirm_order()` at line 226 creates `AdyenPay(self.store, 'mango')` instead of `'lego'`. Does `AdyenPay`'s second parameter affect merchant configuration, and if so, are Lego payments using Mango's Adyen merchant settings?

4. **Leroy Merlin API key exposure**: The API key `jOz8NzN0THYj9JAKqibDH0DTcx73moJZ` is hardcoded at `leroy_merlin/store_type.py:50`. Is this key still valid, and should it be in environment configuration?

5. **Dixons stock validation disabled**: The stock availability check in `create_order()` is commented out (`store_type.py:307-308`). Was this intentional (e.g., to avoid blocking sales for stale inventory data), or is this a debugging remnant?

6. **Inconsistent Redis hosts**: Four different Redis configurations across modules: `settings.REDIS_HOST`, `gates.mishipay.com`, `localhost`, and `settings.REDIS_HOST` again. Are these all the same Redis instance in production, or do they point to different servers?

7. **Decathlon RFID activate unused**: `activate_epc_for_rfid()` is defined but never called. Is RFID tag reactivation for returns handled elsewhere, or is this feature incomplete?

8. **Cisco Spreedly payment delivery body**: The hardcoded certificate/encryption template in `create_order()` (lines 201-214) uses string substitution. Is this template maintained separately, and could it be vulnerable to injection if item data contains template markers?
