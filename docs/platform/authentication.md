# Authentication, Configuration & Base Module

> Last updated by: Task T08 — Authentication & Configuration

## Overview

The MishiPay platform uses a multi-layered authentication system that supports both registered Keycloak-authenticated users and anonymous guest shoppers. Authentication spans three mechanisms: **Keycloak JWT via Kong API gateway** (primary for mobile/web apps), **Django REST Framework Token Authentication** (legacy/internal), and **guest checkout** (anonymous shopping sessions). Platform configuration is managed through a global key-value store (`GlobalConfig`) with cache-through semantics.

This document covers three modules:
- **`mishipay_authentication`** (13 files) — Guest checkout authentication service
- **`mishipay_config`** (15 files) — Global platform configuration management
- **`mishipay`** (20 files) — Legacy base module with the original Item model and MishipayStoreType

---

## Key Concepts

- **MishiPay ID (`mishipay_id` / `mishid`)**: A unique device/session identifier sent via `x-mishipay-id` header. Used to track guest users across requests without requiring login.
- **Auth Service Type**: A string key (`mishipay_email_customer`, `decathlon_customer`, `mishipay_guest_checkout`) that selects a definition-driven authentication flow.
- **Auth Definitions**: Data-driven dictionaries in `auth_definitions.py` that declare the signup/login behavior for each auth service type — what entities to create, what prerequisites to check.
- **GlobalConfig**: A database-backed key-value store where keys are short strings (max 50 chars) and values are arbitrary JSON. Used for feature flags, store-level configuration, and marketing consent rules.
- **StoreType**: An abstract base class (`dos.base_store_type.StoreType`) that defines per-retailer behavior for item management, orders, payments, and refunds. `MishipayStoreType` is the default implementation.

---

## Architecture

### Authentication Flow Overview

```
Client Request
    │
    ├─── Kong API Gateway (validates Keycloak JWT)
    │         │
    │         ├── Sets x-mishipay-id header
    │         ├── Sets x-anonymous-consumer header (if auth failed)
    │         └── Passes Bearer token through
    │
    ▼
Django Middleware Chain
    │
    ├─── StartupMiddleware
    │         ├── Swaps x-authorization → Authorization header
    │         ├── Resolves store from store_id param
    │         ├── Sets i18n language + timezone
    │         └── Attaches request.store
    │
    ├─── SentryInterceptMiddleware
    │         └── Blocks Decathlon bundleid, logs 400s
    │
    ▼
DRF Authentication (per-request)
    │
    ├─── TokenAuthentication (DRF built-in)
    │         └── Checks Authorization: Token <key>
    │
    ├─── MishiKeycloakAuthentication (primary)
    │         ├── Reads x-mishipay-id (required)
    │         ├── If Bearer token: decode JWT, find/create user
    │         └── If no token: find/create guest user
    │
    └─── TransactionTokenAuthentication (special)
              └── One-time tokens for specific order operations
```

### Default DRF Configuration

From `mainServer/settings.py:306-318`:

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.TokenAuthentication',
        'mainServer.middleware.MishiKeycloakAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.AllowAny',
    ),
}
```

**Notable**: The default permission is `AllowAny`, meaning all endpoints are publicly accessible unless a view explicitly declares stricter permissions.

### Django Authentication Backends

From `mainServer/settings.py:407-411`:

```python
AUTHENTICATION_BACKENDS = (
    'django.contrib.auth.backends.ModelBackend',
    'social_core.backends.facebook.FacebookOAuth2',
    'social_core.backends.google.GoogleOAuth2',
)
```

Facebook and Google OAuth2 are configured as social auth backends via `python-social-auth`.

---

## MishiKeycloakAuthentication — Primary Auth Mechanism

**Source**: `mainServer/middleware.py:138-252`

This is the primary authentication class for modern client requests. It implements DRF's `BaseAuthentication` and handles both authenticated Keycloak users and anonymous guest shoppers.

### Flow

1. **Read `x-mishipay-id` header** — If absent, returns `(None, None)` (no auth attempted; falls through to next authenticator or AnonymousUser).

2. **Check `x-anonymous-consumer` header** — If `"true"` (set by Kong when auth failed at the gateway), and a Bearer token is present, raises `AuthenticationFailed`. This prevents clients from sending invalid tokens that bypass Kong validation.

3. **If Bearer token present** (registered user):
   - Parse token type; if not `"Bearer"`, return `AnonymousUser`
   - **Decode JWT without signature verification** (`verify_signature: False`) — relies on Kong having already validated the token
   - Extract `given_name`, `last_name`, `email`, `user_id` from decoded JWT claims
   - Look up Django `User` by `username=email`
   - If user not found, mark `first_scan = True` (new user to be created)

4. **If no Bearer token** (guest user):
   - Look up `Customer` by `mishipay_id` + `is_guest_user=True` + `is_active=True`
   - If found, use existing user
   - If `MultipleObjectsReturned`: keep first, deactivate duplicates (self-healing)
   - If not found, prepare guest user creation data

5. **Create user if needed**:
   - Create Django `User` with email as username
   - Create `Customer` with `c_id = 100000000 + user.id`, `mishipay_id`, `keycloak_user_id`

6. **Return** `(user, None)` — the token portion is always `None` since Kong handles token validation

### Security Considerations

- **JWT decoded without signature verification** (`middleware.py:179`). The comment states "No need for any other checks. Will be taken care by Kong." This means the Django server trusts Kong completely. If a request bypasses Kong and reaches Django directly, any valid-format JWT would be accepted.
- **Guest user auto-creation**: Any request with an `x-mishipay-id` header that doesn't match an existing customer will create a new guest user. There's no rate limiting at this layer.
- **Duplicate customer self-healing** (`middleware.py:212-226`): When multiple active customers share the same `mishipay_id`, only the first is kept active. This handles a known data integrity issue.

---

## mishipay_authentication Module — Guest Checkout

**Source**: `mainServer/mishipay_authentication/`

This module implements a definition-driven guest checkout authentication system. It provides a single API endpoint for guest user registration and login, supporting multiple auth service types through a declarative configuration pattern.

### Directory Structure

```
mishipay_authentication/
├── __init__.py
├── apps.py                              # MishipayAuthenticationConfig
├── models.py                            # Empty — no custom models
├── admin.py                             # Empty — no admin registrations
├── tests.py                             # Empty — no tests
├── auth_definitions.py                  # Auth service type definitions (3 types)
├── messages.py                          # 4 i18n message constants
├── serializers.py                       # GuestCheckoutInputSerializer, CustomerSerializer
├── views.py                             # GuestCheckout APIView
├── urls.py                              # Single URL route
├── services/
│   └── register_or_login_service.py     # RegisterOrLoginUserAsPerServiceType
├── helpers/
│   └── auth_helper.py                   # get_auth_definitions() lookup
└── migrations/
    └── __init__.py                      # No migrations (no models)
```

### Auth Service Definitions

**Source**: `mishipay_authentication/auth_definitions.py:1-63`

Three authentication service types are defined as data-driven configuration dictionaries:

| Service Type | Auth Backend | Guest? | Third-Party Login | mishid Required |
|---|---|---|---|---|
| `mishipay_email_customer` | `email_password` | No | No | No |
| `decathlon_customer` | `email_password` | No | Yes | No |
| `mishipay_guest_checkout` | `guest_login` | Yes | No | Yes |

Each definition contains:
- **`signup.transactional_elements`**: Flags for `create_auth_user`, `create_guest_customer`, `client_specific_customer_entry`
- **`signup.additional_details`**: Extra steps (e.g., `register_mishid` for guests)
- **`login.pre_existing_check`**: Flags for `check_customer` and `check_client_specific_customer_entry`
- **`login.customer_check_param`**: Field to look up existing customers (e.g., `email`)
- **`auth_backend`**: The backend to use (`email_password` or `guest_login`)

### API Endpoint

| Method | Path | Auth | Permission | Description |
|--------|------|------|------------|-------------|
| `POST` | `v1/guest-checkout/` | None | None | Register or login a guest user |

**Source**: `mishipay_authentication/views.py:39-102`

### Guest Checkout Flow

The `GuestCheckout` view (`views.py:39-102`) handles both instance ID generation and guest user registration:

1. **Merge request data**: Combines GET params and POST body
2. **Validate** with `GuestCheckoutInputSerializer` (requires `auth_service_type`, optionally `mishid`)
3. **If `mishid` not provided**: Generate a UUID4 instance ID and return it — this is the first step of a two-phase flow
4. **If `mishid` provided**:
   - Validate token data against `TokenData` model's JSONB schema
   - Create `RegisterOrLoginUserAsPerServiceType` with the auth service type
   - Call `register_or_login_user(request_data)`
   - On success: get/create DRF Token, update token data, log in user via Django auth, record analytics
   - Return customer data with access token

### RegisterOrLoginUserAsPerServiceType Service

**Source**: `mishipay_authentication/services/register_or_login_service.py:22-193`

This service class orchestrates user creation and login based on auth definitions:

**For guest services** (`mishipay_guest_checkout`):
1. Call `create_guest_user(api_data)` directly
2. Try to find existing customer by `guest_user_identifier=mishid` (returns most recent)
3. If found: update `user_language` and `last_visited_store_id`, return existing user
4. If not found: create new Django `User` with random email (`{random8}@guestuseremail.com`), create `Customer` with `c_id = 100000000 + user.id`, `is_guest_user=True`, `guest_user_identifier=mishid`

**For non-guest services** (`mishipay_email_customer`, `decathlon_customer`):
1. Call `check_login_prerequisites(api_data)` — looks up customer by the defined `customer_check_param`
2. If `all_pass`: call `perform_login()` — get user by email, return user + customer
3. If `partial_fail` (customer exists but no third-party login): call `complete_half_backed_login()` (method not implemented in source — likely handled elsewhere)
4. If `all_fail`: return unauthorized

### Serializers

**Source**: `mishipay_authentication/serializers.py:1-56`

| Serializer | Type | Fields | Notes |
|---|---|---|---|
| `GuestCheckoutInputSerializer` | `Serializer` | `mishid` (optional), `auth_service_type` (required, validated) | Only `mishipay_guest_checkout` accepted as service type |
| `CustomerSerializer` | `ModelSerializer` | `is_guest_user`, `cus_id`, `c_id`, `guest_user_email`, `email`, `first_name`, `last_name`, `opt_in`, `all_order_not_verified`, `access_token` (computed), `show_notifications` (always `False`) | Uses `dos.Customer` model |

### Messages

**Source**: `mishipay_authentication/messages.py:1-6`

| Constant | Value |
|---|---|
| `MISHID_NOT_IN_CORRECT_FORMAT` | "MISHID is not in correct format" |
| `INSTANCEID_GENERATED_SUCCESSFULLY` | "Instance Id generated successfully" |
| `LOGIN_FAILED` | "Login Failed!" |
| `LOGIN_SUCCESSFUL` | "Logged in successfully" |

All messages use Django's `gettext_lazy` for i18n support.

---

## TransactionTokenAuthentication

**Source**: `mishipay_core/authentication.py:1-39`

A specialized authentication class for one-time, order-scoped temporary tokens. Used for operations that require authentication but where the user may not have a standard session (e.g., post-payment operations triggered by external webhooks).

### Flow

1. Read `transaction-token` from request headers
2. Look up `Order` by `order_id` from POST data
3. Look up `TransactionToken` matching order + token key + not expired (`expiry_date >= now`)
4. Get user from `order.basket.customer.user`
5. Get or create a DRF `Token` for the user
6. Set `request.transaction_token_authenticated = True`
7. Mark token as used via `transaction_token.set_used()`
8. Return `(user, token)`

### Dependencies

- `mishipay_retail_customers.models.TransactionToken` — stores the temporary token with order reference, key, and expiry date
- `mishipay_retail_orders.models.Order` — the order being authenticated against

---

## Permission Classes

### Top-Level Permission (`mainServer/permissions.py:1-6`)

| Class | Logic |
|---|---|
| `ReadOnly` | Allows only safe HTTP methods (GET, HEAD, OPTIONS) |

### Core Module Permissions (`mishipay_core/permissions.py:1-142`)

| Class | Purpose | Logic |
|---|---|---|
| `ActiveStorePermission` | Ensures target store is active | Reads `store_id` from GET/POST/URL kwargs; checks `store.active` |
| `ActiveStaffShiftPermission` | Ensures active cashier kiosk shift | Only enforced on `android_cashier_kiosk` platform; checks `KioskStaffShift` by `shift_id` + `status=True` |
| `PriceOverridePermission` | Validates price override authority | Extends `ActiveStaffShiftPermission`; for `android_kiosk` delegates to `DashboardPermission` |
| `IsAdminUser` | Checks admin group membership | Verifies user is in `AdminUser` Django group |
| `InventoryUpdatePermission` | Checks inventory update access | Verifies user is in `InventoryUpdate` Django group |
| `UserStorePermission` | Checks user-store access | Verifies `retailer_store_id` is in the dashboard user's assigned stores |

### Permission Pattern

Permissions are group-based, using Django's built-in `Group` model:
- `AdminUser` — admin-level operations
- `InventoryUpdate` — inventory management
- `Customers` — standard customer group
- `GuestCustomer` — guest user group

Views declare permissions explicitly; the global default is `AllowAny`.

---

## mishipay_config Module — Global Configuration

**Source**: `mainServer/mishipay_config/`

### Directory Structure

```
mishipay_config/
├── __init__.py
├── apps.py                    # MishipayConfigConfig
├── models.py                  # GlobalConfig model
├── views.py                   # StoresConfigByRetailerView
├── urls.py                    # Single URL route
├── admin.py                   # GlobalConfigAdmin with PrettyJSONWidget
├── forms.py                   # GlobalConfigForm
├── factories.py               # GlobalConfigFactory (test factory)
├── utils.py                   # Config utility functions
├── constants.py               # Config key constants
├── tests.py                   # Empty
└── migrations/
    ├── __init__.py
    ├── 0001_initial.py        # 2022-02-15: Initial GlobalConfig
    ├── 0002_auto_20220223_1455.py  # 2022-02-23: value null→default=dict
    └── 0003_alter_globalconfig_value.py  # 2023-02-28: JSONField migration
```

### Data Model

**`GlobalConfig`** (`mishipay_config/models.py:6-35`)

| Field | Type | Constraints | Purpose |
|---|---|---|---|
| `id` | AutoField | PK | Auto-increment primary key |
| `key` | CharField(50) | `unique=True` | Configuration key identifier |
| `value` | JSONField | `blank=True, default=dict` | Arbitrary JSON configuration value |
| `cache_expiry_seconds` | PositiveIntegerField | `default=0` | Cache TTL in seconds (0 = no expiry) |
| `created` | DateTimeField | `auto_now_add` | Creation timestamp |
| `updated` | DateTimeField | `auto_now` | Last modification timestamp |

**Cache-Through Semantics**: The model overrides `save()` and `delete()` to synchronize with Django's cache framework:
- On save: `cache.set(key, value, cache_expiry_seconds)`
- On delete: `cache.delete(key)`
- On read (`get_config_value()`): Try cache first; if miss, query DB and populate cache

### API Endpoint

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `v1/stores-config-by-retailer/` | Default (Token/Keycloak) | Returns store configurations grouped by chain and region |

**Source**: `mishipay_config/views.py:9-88`

**Request Parameters**:
- `retailer_id` (optional) — filter by retailer ID
- `retailer` (optional) — filter by retailer name
- `demo` (optional, default `"false"`) — include demo stores

**Response Structure**:
```json
{
  "CHAIN_CODE": {
    "REGION_CODE": {
      "timezone": "Europe/London",
      "stores": [
        {
          "store_id": "uuid",
          "retailer_store_id": "SM04",
          "store_type": "StoreType",
          "retailer_id": 1,
          "config": { "key": "value" }
        }
      ]
    }
  }
}
```

The view builds the config key as `"{CHAIN}-{REGION}"` (uppercased) and looks up the corresponding `GlobalConfig` entry for each store.

### Utility Functions

**Source**: `mishipay_config/utils.py:1-14`

| Function | Purpose |
|---|---|
| `get_client_config_key(store)` | Generates config key as `"{chain}-{region}"` (uppercased); appends `"-DEMO"` for demo stores |
| `is_store_property_enabled(store, entity, property_value_key, property_value)` | Checks if a store has a specific property value enabled in its `store_properties` |

### Constants

**Source**: `mishipay_config/constants.py:1-2`

| Constant | Value | Purpose |
|---|---|---|
| `MARKETING_CONSENT_DISABLED_CONFIG_KEY` | `'MARKETING_CONSENTS_DISABLED_STORE_TYPES'` | GlobalConfig key for disabling marketing consent by store type |
| `MARKETING_CONSENT_PRE_TICK_FALSE_CONFIG_KEY` | `'MARKETING_CONSENT_PRE_TICK_FALSE_STORE_TYPES'` | GlobalConfig key for store types where consent checkbox is unticked by default |

### Admin Interface

**Source**: `mishipay_config/admin.py:1-30`

`GlobalConfigAdmin` provides:
- List display of all fields
- `PrettyJSONWidget` — custom textarea that formats JSON with 4-space indentation, auto-sizes rows (10-30) and columns (40-120)
- Search by `key` field
- Custom form (`GlobalConfigForm`) exposing `key`, `value`, `cache_expiry_seconds`

### Schema Evolution

| Migration | Date | Change |
|---|---|---|
| 0001_initial | 2022-02-15 | Created `GlobalConfig` with nullable JSONField |
| 0002 | 2022-02-23 | Changed `value` from `null=True` to `default=dict` |
| 0003 | 2023-02-28 | Migrated from `postgres.fields.jsonb.JSONField` to native `models.JSONField` (Django 4.x) |

### Test Factory

**Source**: `mishipay_config/factories.py:1-18`

`GlobalConfigFactory` (uses `factory_boy`) generates test fixtures with random `key` strings, `{"key": "value"}` payloads, and random cache expiry (0-3600 seconds).

---

## mishipay Base Module — Legacy Item Model & Store Type

**Source**: `mainServer/mishipay/`

This is the **original** MishiPay Django app — the earliest-written module in the platform. It contains the legacy `Item` model for MishiPay-branded stores and `MishipayStoreType`, which implements the full item/order/payment/refund lifecycle for the default store type. Most newer retailers use specialized store types (DOS, Dixons, etc.), but this remains the foundation.

### Directory Structure

```
mishipay/
├── __init__.py
├── models.py                  # Item model + forms
├── store_type.py              # MishipayStoreType (~1,088 lines)
├── admin.py                   # Import/export admin for Item
└── management/commands/
    ├── mishipay_migrate.py                              # Copy items from MishipayItem → Item
    ├── clerk_and_greens_item_copy.py                    # Copy ClerkAndGreen items → Item
    ├── migrate_items_to_mishipay_items_module.py        # CSV import to new items module
    ├── leroy_cisco_clerkandgreen_old_order_migration.py # Migrate old order formats
    └── mishipay_dixons_old_order_migration.py           # Migrate Dixons/MishiPay old orders
```

### Data Model

**`Item`** (`mishipay/models.py:8-34`) — extends `dos.models.base.SecureItem`

| Field | Type | Constraints | Purpose |
|---|---|---|---|
| `barcode` | CharField(63) | Required | Product barcode |
| `store` | FK → `dos.Store` | `null=True`, `on_delete=CASCADE` | Store this item belongs to |
| `category` | CharField(63) | `default=''`, nullable | Product category |
| `item_name` | CharField(511) | `default=''` | Display name |
| `vendor` | CharField(63) | `default=''`, blank | Manufacturer/vendor |
| `img` | URLField | Default: Cloudinary MishiPay logo | Product image URL |
| `price` | FloatField | `default=1.0` | Price (note: Float, not Decimal) |
| `story` | TextField | `default=''`, blank | Product description |
| `created_date` | DateTimeField | `auto_now_add` | Creation timestamp |
| `modified_date` | DateTimeField | `auto_now` | Last modification |
| `quantity` | IntegerField | `default=1` | Inventory stock count |
| `product_id` | CharField(63) | `default=''`, nullable | External product identifier |

**Indexes**: Composite index on `(barcode, store)`
**Constraints**: `unique_together = ['barcode', 'store']`
**DB Table**: `mishipay_items`

Inherits from `SecureItem` which likely provides `uid` (EPC/RFID identifier) and `sold` (boolean) fields.

### MishipayStoreType

**Source**: `mishipay/store_type.py:48-1088`

`MishipayStoreType` extends `dos.base_store_type.StoreType` and implements the complete retail operations interface for MishiPay-branded stores.

| Method | Purpose |
|---|---|
| `get_item(data)` | Lookup item by barcode (case-insensitive). Handles SATO/Cisco EPC deconstruction. |
| `get_items(data)` | List items for a store, optionally filtered by category, with pagination (10/page). |
| `get_item_category(data)` | Get distinct categories for a store. |
| `add_item(data)` | Create new item with barcode uniqueness check, price/quantity validation, RFID rules, Redis integration for EPC tracking. |
| `update_item(data)` | Update existing item. Manages Redis sold/unsold state for RFID items. |
| `delete_item(data)` | Delete item by barcode. |
| `apply_promotions(data)` | **Stub** — returns `NO_PROMO` with benefit_value=0. Promotions handled elsewhere. |
| `create_order(data)` | Full order creation: stock check → payment (Stripe/Verifone/Adyen/JudoPay) → order record → set_sold → receipt → dashboard publish. |
| `pay(data, platform)` | Dispatch payment to Stripe, Verifone, or Adyen. Verifone supports session-based and token-based (saved card) flows. |
| `refund_order(data)` | Full/partial refund: update non_refunded/refunded item lists, calculate refund amount, process via Stripe, restore inventory. |
| `set_sold(data)` | Decrement inventory quantity, set `sold=True` when qty=0, update Redis, sync with SATO/Nedap external systems. |
| `set_unsold(data)` | Restore item to unsold state, update Redis. |
| `update_unsold_status(item)` | Post-refund inventory restoration with SATO/Nedap integration. |
| `get_sold_status(data)` | Batch check sold status for list of barcodes. |
| `get_sales_report(data)` | Query `SalesSummary` for date range. |
| `add_external_oid(data, order)` | Register external order ID (restricted to Dixons, MishiPay Global, Mango stores). |
| `deconstruct_epc(epc)` | Convert Cisco QR code hex EPC to product code via binary deconstruction. |

### External Integrations in MishipayStoreType

| Integration | Methods | Purpose |
|---|---|---|
| **Redis** | `set_sold`, `set_unsold`, `add_item`, `update_item` | Track RFID item sold/unsold state in Redis (key=UID, value=0/1) |
| **Nedap !DCloud** | `set_sold_nedap`, `update_unsold_nedap` | Sync sold/unsold status to Nedap retail API for RFID items |
| **SATO (via ngrok)** | `set_sold_in_sato`, `set_item_to_unsold_sato`, `get_token`, `check_if_item_is_sellable` | Sync item status with SATO/Cisco CoreServices API |
| **Stripe** | `pay`, `refund_order` | Payment processing and refunds |
| **Verifone** | `pay` | Payment via session-based or token-based (saved card) flows |
| **Adyen** | `pay` | Payment via saved card or payload verification |

### Management Commands

| Command | Purpose |
|---|---|
| `mishipay_migrate` | Copy items from `dos.MishipayItem` → `mishipay.Item` (field-by-field) |
| `clerk_and_greens_item_copy` | Copy items from `ClerkAndGreenItem` → `mishipay.Item` |
| `migrate_items_to_mishipay_items_module` | CSV import to the newer `mishipay_items` module (`ItemCategory`, `ItemInventory`). Deletes existing items first. Maps 75+ retailers to countries. |
| `leroy_cisco_clerkandgreen_old_order_migration` | Migrate old order formats for Leroy Merlin, Cisco, and other legacy stores. Groups duplicate items by barcode. |
| `mishipay_dixons_old_order_migration` | Migrate Dixons/MishiPay order formats. Separates refunded/non-refunded items from `item_details` into dedicated fields. |

All management commands are one-time data migration scripts. They are legacy and unlikely to be run again.

### Admin Configuration

**Source**: `mishipay/admin.py:1-21`

`ItemAdmin` (via `ImportExportModelAdmin`):
- List display: store, barcode, item_name, category, vendor, uid, quantity, price, sold, dates, story, img
- Filters: store
- Search: store_id, barcode, item_name, uid
- Import/export via `django-import-export` with `ItemResource`

---

## Dependencies

### mishipay_authentication depends on:
- `dos.models.Customer`, `dos.models.customer.TokenData` — customer data
- `dos.analytics.customer_analytics` — login/signup analytics
- `mishipay_core.common_functions` — token update, response structure, i18n messages
- `mishipay_dashboard.util` — analytics recording
- `rest_framework.authtoken.models.Token` — DRF token management
- `django.contrib.auth.models.User`, `Group` — Django auth

### mishipay_config depends on:
- `dos.models.Store` — store model for config views
- `mishipay_core.models.Retailer` — retailer lookup
- `django.core.cache` — Django cache framework

### mishipay (base) depends on:
- `dos.models` — Store, Order, Customer
- `dos.models.base.SecureItem` — base item class
- `dos.base_store_type.StoreType` — abstract store type
- `dos.order.order_helper` — order creation
- `dos.payment` — BasePayment, StripePay, VerifonePay, AdyenPay
- `dos.util` — publishing, receipts, validation
- `mishipay_core.common_functions` — barcode extraction
- `analytics.models.SalesSummary` — sales reporting
- `redis` — RFID state tracking

### What depends on these modules:
- **`mishipay_authentication`**: Mounted at `auth/` in URL config. Called by mobile apps for guest checkout flow.
- **`mishipay_config`**: `GlobalConfig.get_config_value()` is used across the platform for feature flags and store-level config. Mounted at `config/` in URL config.
- **`mishipay` (base)**: `MishipayStoreType` is used by `dos.base_store_type.StoreType` dispatch for MishiPay-branded stores. The `Item` model is the legacy item table.

---

## Notable Patterns

### Definition-Driven Auth
The `auth_definitions.py` pattern separates authentication logic from configuration. Adding a new auth service type only requires adding a dictionary entry — no code changes needed to the service class. However, only `mishipay_guest_checkout` is actually supported by the `GuestCheckoutInputSerializer` validator (`serializers.py:10`), making the other definitions unreachable through this endpoint.

### Dual Guest User Systems
There are two independent guest user creation paths:
1. **`mishipay_authentication` module** (`register_or_login_service.py`): Uses `guest_user_identifier` (mishid) to identify returning guests. Creates random email addresses.
2. **`MishiKeycloakAuthentication` middleware** (`middleware.py`): Uses `mishipay_id` (x-mishipay-id header) to identify returning guests. Also creates random email addresses.

These two systems appear to serve different client versions — the authentication module for the older guest checkout flow, and the middleware for the newer Keycloak-based architecture.

### Cache-Through Configuration
`GlobalConfig.save()` writes to both the database and cache atomically (within the same method call, though not in a true distributed transaction). `get_config_value()` reads cache-first with DB fallback. The `cache_expiry_seconds` field allows per-key TTL control.

### Legacy Store Type Pattern
`MishipayStoreType` implements the Strategy pattern — each retailer has its own StoreType subclass. The base `mishipay` module's implementation contains direct integrations with Nedap, SATO, Stripe, Verifone, and Adyen, making it a monolithic class (~1,088 lines). Newer retailer integrations use more modular patterns.

### Hardcoded Credentials in store_type.py
- **Nedap API token** (`store_type.py:621`): Hardcoded Bearer token `d8baad55c2a9fb590ff7ceac99d3c5417b6926a528a7aa7ab33f35`
- **SATO/Cisco API** (`store_type.py:1022`): URL with hardcoded credentials `username=mishipay&password=clseur2019` via ngrok tunnel

---

## Configuration

### Environment Variables / Settings

| Setting | Location | Purpose |
|---|---|---|
| `REST_FRAMEWORK.DEFAULT_AUTHENTICATION_CLASSES` | `settings.py:309` | TokenAuthentication + MishiKeycloakAuthentication |
| `REST_FRAMEWORK.DEFAULT_PERMISSION_CLASSES` | `settings.py:313` | AllowAny (global default) |
| `AUTHENTICATION_BACKENDS` | `settings.py:407` | ModelBackend + Facebook + Google OAuth2 |
| `REDIS_HOST` | `settings.py` | Redis host for RFID state tracking |
| `ENV_TYPE` | `settings.py` | Environment identifier (used in Datadog tags) |

### GlobalConfig Keys (Known)

| Key Pattern | Purpose |
|---|---|
| `{CHAIN}-{REGION}` | Store configuration by chain + region (e.g., `STO-MV`) |
| `{CHAIN}-{REGION}-DEMO` | Demo store configuration |
| `MARKETING_CONSENTS_DISABLED_STORE_TYPES` | Store types with marketing consent disabled |
| `MARKETING_CONSENT_PRE_TICK_FALSE_STORE_TYPES` | Store types where consent checkbox defaults to unchecked |

---

## Open Questions

- **Are both guest user creation paths still active?** The `mishipay_authentication` module (using `mishid` / `guest_user_identifier`) and `MishiKeycloakAuthentication` middleware (using `mishipay_id`) appear to serve different client versions. Which is the current/preferred path? Are there plans to deprecate the older module?
- **Why is `AllowAny` the global default permission?** This means every endpoint is publicly accessible unless explicitly restricted. Is this intentional (relying on Kong for access control) or an oversight?
- **Is `complete_half_backed_login()` implemented?** The method is referenced in `register_or_login_service.py:182` but not defined in the class. Is it inherited from a parent class or is it dead code?
- **Are the `mishipay_email_customer` and `decathlon_customer` auth definitions used?** The `GuestCheckoutInputSerializer` only accepts `mishipay_guest_checkout`. Are these definitions used by another endpoint or are they dead code?
- **What GlobalConfig keys exist in production?** The config system is flexible but undocumented. A listing of actual keys and their purposes would be valuable.
- **Is the Nedap integration still active?** The hardcoded Bearer token and SATO ngrok tunnel URLs in `store_type.py` suggest these may be legacy integrations.
- **How is `ProcessStatsMiddleware` configured?** It sends Datadog metrics for specific routes but only for `HTTP_200_OK` and `HTTP_201_CREATED`. Is this intentional or should error latencies also be tracked?
