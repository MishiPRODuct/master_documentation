# Support Modules

> Last updated by: Task T32 — Fiscalisation, DDF Price Activation

## Overview

This document covers smaller support modules that provide operational tooling, cloud storage integration, and microservice client utilities. These modules are not part of the core business flow but support administrative operations and inter-service communication.

Modules documented here:
- **`admin_actions`** (21 files) — Django admin-based operational action framework for store creation, payment configuration, theme management, email sending, and dashboard token provisioning.
- **`clients`** (3 files) — Azure Blob Storage client and Promotions Service V2 client for CSV-based promotion import/export.
- **`shopping_list`** (21 files) — Customer shopping list feature with dual v1 (legacy monolith items) / v2 (inventory service) architecture, transfer-to-basket capability, and collaborative lists.
- **`rfid`** (4 files) — Keonn RFID hardware integration endpoint for checking EPC sold/unsold status via Redis lookup.
- **`jobs`** (3 files) — Scheduled cron jobs (`django-cron`) for daily order Excel reports and analytics summary emails for Saturn and Decathlon retailers.
- **`pubsub`** (24 files) — Kafka-based publish/subscribe messaging framework using Confluent Kafka with SASL/SSL authentication. Provides producer adapters for 6 business events and 3 consumer daemons.
- **`saturn`** (12 files) — Original Saturn retailer integration (Austria/Innsbruck) with barcode→EPC item mapping, async product detail fetching from DFL Media-Saturn APIs, SOAP-based fulfilment orders, PayUnity/PayPal payments, and Redis-backed sold-status tracking.
- **`saturn_smart_pay`** (8 files) — Second-generation Saturn integration (Germany/Hamburg) extending `StoreType` with CAR-NICE product API, NFC label scanning, XPay/Google Pay/Apple Pay payment support, Pfand (deposit) handling, and SOAP fulfilment.
- **`easy_store_creation`** (23 files) — Self-service store provisioning system with template-based store cloning, payment variant replication via MS-Pay API, theme copying, bulk CSV store upload/download, inventory/promotion/tax CSV upload, and Bitly campaign URL generation.
- **`fiscalisation`** (26 files) — Fiscal compliance module implementing two country-specific workflows: Venistar/Retail Pro polling-based fiscalisation for Italy (Lotteria degli Scontrini, itemised tax reporting) and Funky Buddha callback-based fiscalisation for Greece (AADE myDATA mark IDs). Includes pub/sub handler, email alerts, and per-store-type/region permission model.
- **`ddf_price_activation`** (22 files) — Dubai Duty Free RPM (Retail Price Management) price activation system. Three-phase workflow: automated JSON file ingestion from DDF's RPM system, manual CSV download by store staff, and price activation via the MishiPay Inventory Service. Custom Django admin UI with staff credential validation.

---

## Admin Actions Module (`admin_actions`)

### Purpose

A self-service operational tooling system built on top of Django admin. It allows staff users to perform privileged operations (store creation, payment variant setup, theme management, email sending, dashboard token provisioning) through a form-based UI, with full audit logging and re-run capability.

### Architecture

```
Django Admin UI
    |
    v
admin_action_selection (views.py)     ─── @staff_member_required
    |── Lists available actions based on user's Django permissions
    |── Optional: config form pre-step (e.g., select source store)
    |
    v
admin_action_populate_form (views.py)
    |── Config form → pre-populates action form from existing data
    |── Action form → validates input → saves AdminAction record
    |
    v
AdminAction.run() (models.py)
    |── Dynamic import: admin_privilege.admin_action_class
    |── __import__(module_path) → getattr(module, class_name)
    |── Instantiates action class with input_data
    |── Executes action_class.run()
    |── Records success/failure + stack trace
    |
    v
Action Classes (actions_definitions.py)
    |── SendEmailAction      → Django send_mail
    |── CreateStoreAction    → Store.objects.create()
    |── CreatePaymentVariantAction → PaymentVariant.objects.create()
    |── CreateThemeAction    → Theme + ThemeProperty management
    |── DashboardTokenAction → Bulk Token creation from CSV
```

### Data Models

#### `AdminPrivilege` (`models.py:27`)

Links a Django `Permission` to an executable action class.

| Field | Type | Purpose |
|-------|------|---------|
| `permission` | FK → `auth.Permission`, CASCADE | The Django permission that grants access to this action |
| `name` | CharField(256) | Action identifier (e.g., `"create_store"`, `"send_email"`) — used as key in `ACTION_FORMS_MAP` |
| `created` | DateTimeField(auto_now) | Creation timestamp |
| `admin_action_class` | CharField(1024) | Fully qualified Python class path (e.g., `"admin_actions.actions_definitions.CreateStoreAction"`) — dynamically imported at runtime |

#### `AdminAction` (`models.py:37`)

Audit log and execution record for each admin operation.

| Field | Type | Purpose |
|-------|------|---------|
| `action_name` | CharField(128) | Human-readable action name (auto-generated from privilege name) |
| `performed_by` | FK → `dos.User`, CASCADE | Staff user who created the action |
| `created` | DateTimeField(auto_now) | Creation timestamp |
| `updated` | DateTimeField(auto_now) | Last update timestamp |
| `last_run_status` | CharField(32), choices: `pending`/`success`/`failure` | Execution status |
| `last_run_at` | DateTimeField, nullable | Last execution timestamp |
| `output` | CharField(1024) | Execution output message |
| `stack_trace` | TextField | Error traceback on failure |
| `ticket_number` | CharField(1024) | Associated support ticket reference |
| `input_data` | JSONField | Serialized form input data |
| `admin_privilege` | FK → `AdminPrivilege`, CASCADE | The privilege (and therefore action class) to execute |
| `is_rerunnable` | BooleanField(default=False) | Whether the action can be re-executed after success |

**`run()` method** (`models.py:56`): Dynamically imports the action class from `admin_privilege.admin_action_class` using `__import__()`, instantiates it with `input_data`, calls `run()`, and records success/failure with stack trace. This is a **dynamic code execution pattern** — the `admin_action_class` field contains an arbitrary Python import path.

### Action Classes (`actions_definitions.py`)

All action classes inherit from `BaseAction` (ABC) and implement `run()`.

#### `SendEmailAction`

Sends an email via Django's `send_mail` using the `"transaction"` email settings from `config.loader.EMAIL_SETTINGS`.

- **Form:** `SendEmailForm` (message + email)
- **From:** `info@mishipay.com`
- **Subject:** `"Message from Mishipay"` (hardcoded)

#### `CreateStoreAction`

Creates a new `Store` record with comprehensive configuration.

- **Form:** `CreateStoreForm` (25+ fields covering retailer, location, payment, RFID, app type, etc.)
- **Notable logic:**
  - Auto-assigns next `object_id` by sorting all existing Store `object_id` values and incrementing the maximum (`models.py:93-99`)
  - Maps `app_type` to bundle IDs via `APP_TYPE_BUNDLE_ID_MAP` (MishiPay, Decathlon, Saturn)
  - Maps currency to sign via `CURRENCY_SIGN_MAP` (11 currencies: EUR, USD, GBP, AED, ILS, CHF, TRY, DKK, SEK, NOK)
  - Sets `white_list_status` based on app type (`"1"` for MishiPay, `"2"` for others)
  - Sets `item_source` based on `mishipay_inventory` boolean
- **Permission check:** `is_permitted = True # TODO: Check permission` — always permitted

#### `CreatePaymentVariantAction`

Creates a `PaymentVariant` record for a store's payment gateway configuration.

- **Form:** `PaymentVariantForm` (25+ fields covering PSP endpoints, merchant config, 3DS, Optile)
- **Validation:** Prevents duplicate configs per store/sandbox combination
- **Permission check:** `is_permitted = True # TODO: Check permission` — always permitted

#### `CreateThemeAction`

Creates or replaces theme properties for multiple stores across all platforms.

- **Form:** `CreateThemeForm` (multi-store select + themes JSON)
- **Logic:** For each selected store × each platform (`android`, `ios`, `web`):
  1. `get_or_create` Theme for store/platform
  2. Delete all existing `ThemeProperty` records
  3. Create new properties from JSON key-value pairs
  4. Calls `mishipay_core.client_config.clear_caches()` after completion
- **Returns:** Success message listing processed store IDs

#### `DashboardTokenAction`

Bulk creates dashboard invite tokens from a CSV file upload.

- **Form:** `CreateDashboardTokenForm` (store_id + permissions + CSV file)
- **CSV format:** Single column `dashboard_username`; max file size 100KB
- **Validation:** Checks usernames don't already exist in `DashboardUser` or `Token` tables
- **Logic:** Creates `Token` record for each username with specified store and permissions
- **Returns:** List of `{username, token}` pairs

### Forms

#### Action Forms (`action_forms.py`)

| Form | Fields Count | Purpose |
|------|-------------|---------|
| `BaseForm` | 1 (`is_rerunnable`) | Base class for all action forms |
| `SendEmailForm` | 3 | Email + message textarea |
| `CreateStoreForm` | 25+ | Complete store creation: retailer (raw ID widget), name, store_type, region/sub_region, address/address_json, lat/long, payment_platform, RFID, inventory, currency, demo, offers, delivery, app_type, activation, extra_data JSON, auth_providers, flow_type, exit_journey_flow_type, chain, timezone, tax settings |
| `PaymentVariantForm` | 25+ | Full PSP configuration: store (raw ID widget), merchant_id, 7 endpoint URLs, payment_procedure, token_or_key, shopper statement, country, currency, transaction_prefix, capture/preauth/refund booleans, PSP type, payment_class (dynamically discovered), extra_data JSON, sandbox, optile config, 3DS flow, payment methods, secondary PSP details |
| `CreateThemeForm` | 2 | Multi-store selector (FilteredSelectMultiple widget) + themes JSON |
| `CreateDashboardTokenForm` | 6 | Store (raw ID widget), 5 permission booleans (can_add, can_edit, can_delete, can_refund, allow_analytics), CSV file upload |

#### Config Forms (`config_forms.py`)

Two-step forms that pre-populate action forms from existing data:

| Form | Purpose |
|------|---------|
| `CreateStoreConfigForm` | Select an existing store → pre-populates `CreateStoreForm` via `StoreSerializer` |
| `PaymentVariantConfigForm` | Select store + sandbox/live → pre-populates `PaymentVariantForm` via `PaymentVariantSerializer`. Validates that the source config exists. |

### Serializers (`serializers.py`)

| Serializer | Model | Purpose |
|------------|-------|---------|
| `StoreSerializer` | `Store` | Used by `CreateStoreConfigForm` to serialize existing store data for form pre-population. Custom `ref_name='StoreAdmin'` to avoid OpenAPI name clashes. Computed fields: `mishipay_inventory` (from `item_source`), `orders_delivered` (from `is_address_required_for_order`), `app_type` (from `STORE_TYPE_APP_TYPE_MAP`), `activate` (always `False`), `store_name` (from `name`). |
| `PaymentVariantSerializer` | `PaymentVariant` | Used by `PaymentVariantConfigForm` to serialize existing payment config. Contains a nested `get_store()` that always returns `None` (unused). |

### Choices & Configuration (`choices.py`, `app_settings.py`)

#### Dynamic Payment Provider Discovery (`choices.py:33`)

`PaymentProviderClassChoices.get_choices()` uses `pkgutil.walk_packages()` to scan `mishipay_retail_payments.providers.*` and discover all subclasses of `BasicProvider`. This produces a dynamic choices tuple at import time.

#### Configuration Maps (`app_settings.py`)

| Map | Purpose |
|-----|---------|
| `STORE_TYPE_APP_TYPE_MAP` | Maps store types to app types: `SaturnSmartPayStoreType`/`SaturnStoreType` → Saturn, `DecathlonStoreType` → Decathlon, default → MishiPay |
| `CURRENCY_SIGN_MAP` | 11 currency code → symbol mappings (euro/eur → €, usd → $, gbp → £, aed → AED, ils → ₪, chf → CHF, try → ₺, dkk → Kr., sek/nok → kr) |
| `APP_TYPE_BUNDLE_ID_MAP` | 3 app types × 3 platforms (iOS/Android/Web) → bundle IDs |
| `ACTION_FORMS_MAP` | 5 action names → form classes |
| `AdminActionProperties.property_map` | 5 action names → config form associations and `allow_action_form_without_source` flags |

### Views (`views.py`)

Two Django template views (not DRF), both `@staff_member_required`:

| View | URL | Method | Purpose |
|------|-----|--------|---------|
| `admin_action_selection` | `select-action/` | GET/POST | Lists available actions based on user's Django permissions. POST triggers privilege lookup and optional config form display. |
| `admin_action_populate_form` | `populate-selection-form/` | POST | Two paths: (1) config form submission → pre-populates action form from existing data; (2) action form submission → saves `AdminAction` record and redirects to admin changelist. Uses `CustomFormParser` to serialize form data to JSON. |

### Admin Configuration (`admin.py`)

| Model | Admin Class | Key Features |
|-------|-------------|-------------|
| `AdminPrivilege` | `AdminPrivilegeAdmin` | List: id, name, created. `raw_id_fields`: permission. |
| `AdminAction` | `AdminActionAdmin` | List: id, action_name, performed_by, created, last_run_at, last_run_status. Search: action_name, performed_by, last_run_status. **No add permission** (`has_add_permission()` returns `False`). **Queryset filtering**: non-superusers see only their own actions. **All fields read-only on edit.** Custom `change_list_template`. **"Run" action**: executes `admin_action.run()` for selected records — skips already-successful non-rerunnable actions. |

### URL Routes

| Path | View | Name |
|------|------|------|
| `select-action/` | `admin_action_selection` | `admin_action_selection` |
| `populate-selection-form/` | `admin_action_populate_form` | `admin_action_form` |

### Validators (`form_validators.py`)

| Function | Purpose |
|----------|---------|
| `validate_json(json_str)` | Validates JSON string using `simplejson`. Raises `FormValidationError` on parse failure. |
| `validate_store_id(store_id)` | Validates store exists via `Store.objects.get(store_id=store_id)`. Raises `FormValidationError` if not found. |

### Tests

Empty stub file (`tests.py`): `from django.test import TestCase  # noqa: F401`

### Migrations

3 migrations:
- `0001_initial` — Initial schema
- `0002_auto_20191223_1428` — Auto-generated changes (December 2019)
- `0003_alter_adminaction_input_data` — Changed `input_data` to JSONField

### Dependencies

| Depends On | Purpose |
|------------|---------|
| `dos.models.Store` | Store creation, config form source, form field querysets |
| `dos.models.User` | `performed_by` FK on AdminAction |
| `dos.models.choices` | Currency, payment platform, RFID, auth provider, flow type choices |
| `dashboard.models.Token` | Dashboard token creation |
| `dashboard.models.User` | Username uniqueness validation |
| `mishipay_core.models.Theme` | Theme creation and property management |
| `mishipay_core.models.Retailer` | Retailer selection in store creation form |
| `mishipay_core.common_functions` | `strict_bool_cast`, `CustomFormParser`, `CustomMultipleModelChoiceField` |
| `mishipay_core.client_config` | `clear_caches()` after theme changes |
| `mishipay_core.custom_fields` | `PrettyJSONField` for JSON form fields |
| `mishipay_core.custom_widgets` | `ModelSelectRawIdWidget` for store/retailer selectors |
| `mishipay_retail_payments.models` | `PaymentVariant` creation and serialization |
| `mishipay_retail_payments.core` | `BasicProvider` for dynamic provider discovery |
| `mishipay_retail_payments` | Payment method choices, procedure choices, 3DS flow choices |
| `config.loader` | `EMAIL_SETTINGS` for email sending |
| `django_countries` | Country choices for store/payment forms |

### Notable Patterns

1. **Dynamic code execution** — `AdminAction.run()` uses `__import__()` with `admin_privilege.admin_action_class` to load arbitrary Python classes from a database field. While restricted to admin-created records, this is a code injection vector if the `admin_action_class` field is tampered with.

2. **Two-step form workflow** — Config forms (step 1) pre-populate action forms (step 2) from existing data, allowing "clone and modify" workflows for store and payment variant creation.

3. **Permission-gated action visibility** — Actions are only shown to users whose Django permissions match the `AdminPrivilege.permission` FK. However, `CreateStoreAction` and `CreatePaymentVariantAction` have `is_permitted = True # TODO: Check permission`, bypassing any secondary permission check.

4. **Audit trail with re-run** — Every action execution is recorded with input data, output, status, and stack trace. Actions can be re-run from the admin changelist if `is_rerunnable=True` or if they haven't succeeded yet.

5. **Auto-incrementing `object_id`** — `CreateStoreAction` determines the next `object_id` by sorting all existing integer `object_id` values and adding 1 to the maximum. This is not atomic and could produce duplicates under concurrent execution.

6. **Dynamic provider discovery** — `PaymentProviderClassChoices` scans the entire `mishipay_retail_payments.providers` package tree at import time using `pkgutil.walk_packages()` to build the payment class choices dynamically.

---

## Clients Module (`clients`)

### Purpose

Cloud storage and microservice client utilities. Provides an Azure Blob Storage client for file uploads and a Promotions Service client for fetching and exporting promotion data via CSV.

### Azure Storage Client (`azure_client.py`)

| Method | Purpose |
|--------|---------|
| `__init__()` | Creates `BlobServiceClient` from `AZURE_STORAGE_CONNECTION_STRING` environment variable |
| `generate_sas_token(blob_client)` | Generates a read-only SAS token valid for 30 minutes using `datetime.utcnow()` |
| `upload_blob_file(retailer, filepath, filename)` | Uploads file to `retailer-data` container at path `retailer/{RETAILER}/test/CA/promotions/{filename}`. Returns SAS-signed URL. |

**Container structure:** `retailer-data/retailer/{RETAILER_UPPERCASE}/test/CA/promotions/{filename}`

**Note:** The path includes `/test/CA/` — this appears to be hardcoded for a test/Canada environment. Production paths may differ, or this may be an oversight.

### Promotions Service V2 Helper (`promo_service_client.py`)

HTTP client for the promotions microservice and CSV file export utility.

| Method | Purpose |
|--------|---------|
| `get_promotions_list_by_retailer(retailer)` | `GET {MS_PROMO_SERVICE}/api/1.0/promotions/retailer/{retailer}/` — Fetches all promotions for a retailer. Returns JSON response or `None` on failure. |
| `get_promotions_list_by_store(store_id)` | `GET {MS_PROMO_SERVICE}/api/1.0/promotions/store/{store_id}/` — Fetches all promotions for a store. Returns JSON response or `None` on failure. |
| `upload_file(retailer, promotions)` | Formats promotions into CSV via `PromoHeaderRow`, writes to local file, uploads to Azure Blob Storage via `AzureStorageClient`. Returns SAS-signed URL. |

**`PromoHeaderRow`** — Data class representing a single promotion CSV row with 21 instance attributes (retailer, promo_id, promo_family, title, description, discount_type, discount_value, notes, requisite groups, dates, special/group config, item_id, priority, discounted items flag, strategy, active status, active_days). Provides:
- `header()` — Returns 18-column CSV header (note: `description`, `retailer`, `is_active`, `active_days` are instance attributes but excluded from CSV header/output)
- `to_list()` — Serializes to 18-element list matching header columns

### Dependencies

| Depends On | Purpose |
|------------|---------|
| `azure.storage.blob` | `BlobServiceClient`, `generate_blob_sas`, `BlobSasPermissions` for Azure Blob Storage |
| `settings.MS_PROMO_SERVICE` | Base URL for promotions microservice |
| `settings.AZURE_STORAGE_CONNECTION_STRING` (env var) | Azure Storage connection string |

### Notable Patterns

1. **No timeout on HTTP calls** — `PromoServiceV2Helper.get_promotions_list_by_retailer()` and `get_promotions_list_by_store()` call `requests.get()` without a `timeout` parameter. If the promotions service is unresponsive, these calls will hang indefinitely.

2. **Local file write for CSV export** — `upload_file()` writes CSV to the current working directory (using `open(filename, "w")`) before uploading. This temporary file is not cleaned up after upload.

3. **Hardcoded Azure path** — The blob path `retailer/{RETAILER}/test/CA/promotions/` includes `test/CA/`, suggesting this client may be configured for a test environment or Canada-specific use case.

4. **SAS token with naive datetime** — `generate_sas_token()` uses `datetime.utcnow()` (deprecated in Python 3.12+) for the 30-minute expiry calculation.

---

## Shopping List Module (`shopping_list`)

### Purpose

A customer-facing shopping list feature that allows users to create, manage, and share shopping lists scoped to specific stores. Lists can be transferred into baskets for checkout. The module maintains dual v1 (legacy monolith items) and v2 (inventory microservice items) code paths.

### Architecture

```
Customer (mobile/web)
    |
    v
v1 path (legacy):                       v2 path (inventory service):
ShoppingListViewSet                     ShoppingListV2ViewSet
ShoppingListItemViewSet                 ShoppingListItemV2ViewSet
  |── ShoppingList model                  |── ShoppingList model (shared)
  |── ShoppingListItem model              |── ShoppingListItemV2 model
  |── ItemInventory (FK)                  |── JSON blob (denormalized from inventory service)
  |                                       |
  v                                       v
transfer_to_basket()                    transfer_to_basket_inventory_service()
  |── ItemManagementClient                |── ItemManagementClient
  |── add_item_to_basket() × N items      |── add_item_to_basket() × N items
  v                                       v
Basket (mishipay_items)                 Basket (mishipay_items)
```

### Data Models

#### `ShoppingList` (extends `BaseModel`) — `models.py`

A shopping list scoped to a store, shared among one or more customers.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUIDField | PK, auto-generated | Inherited from `BaseModel` |
| `created_on` | DateTimeField | `auto_now_add=True` | Immutable |
| `updated_on` | DateTimeField | `auto_now=True` | |
| `store` | FK → `dos.Store` | `on_delete=CASCADE` | Related name: `shopping_lists` |
| `customers` | M2M → `dos.Customer` | Join table: `customer_shopping_lists` | Collaborative — multiple customers can share a list |
| `items` | M2M → `ItemInventory` | Through: `ShoppingListItem` | **Legacy** — not used by inventory service path |

**Meta:** Composite index on `(created_on, store)`.

#### `ShoppingListItem` (extends `BaseModel`) — `models.py`

Legacy through model linking a `ShoppingList` to an `ItemInventory`.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUIDField | PK | Inherited |
| `created_on` | DateTimeField | `auto_now_add=True` | |
| `updated_on` | DateTimeField | `auto_now=True` | |
| `shopping_list` | FK → `ShoppingList` | `on_delete=CASCADE` | |
| `item` | FK → `ItemInventory` | `on_delete=CASCADE`, `to_field="item_id"` | References `item_id`, not PK |
| `quantity` | IntegerField | `default=1` | |

**Meta:** `unique_together = ("shopping_list", "item")`.

#### `ShoppingListItemV2` (extends `BaseAuditModel`) — `models.py`

Inventory service item on a shopping list. Stores variant data as a denormalized JSON snapshot fetched at creation time.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `id` | UUIDField | PK | Inherited |
| `created` | DateTimeField | `auto_now_add=True` | Note: `created` not `created_on` (different base class) |
| `updated` | DateTimeField | `auto_now=True` | Note: `updated` not `updated_on` |
| `shopping_list` | FK → `ShoppingList` | `on_delete=CASCADE` | Related name: `inventory_service_items` |
| `item_variant_id` | `ObjectIDField` | max_length=24 | MongoDB ObjectID string |
| `item_variant` | JSONField | | Full variant data blob — snapshot from inventory service |
| `quantity` | IntegerField | `default=1` | |

**Meta:** `unique_together = ("shopping_list", "item_variant_id")`.

**Validation (`clean()`):** Raises `ValidationError` if `item_variant_id` does not match `item_variant.get("id")`.

**`ObjectIDField`** — Custom `CharField` hardcoded to `max_length=24` for MongoDB ObjectID strings.

### API Endpoints

**Base path:** `/shopping-list/` (namespace: `shopping-list`)
**Deprecated path:** `/api/v1/` (namespace: `shopping-list-v1`, v1 routes only)

All endpoints require authentication (`IsAuthenticated`) via `TokenAuthentication`, `SessionAuthentication`, or `MishiKeycloakAuthentication`.

| Method | Path | ViewSet | Purpose |
|--------|------|---------|---------|
| GET | `v1/shopping-list/` | `ShoppingListViewSet` | List user's shopping lists (legacy) |
| POST | `v1/shopping-list/` | `ShoppingListViewSet` | Create shopping list (legacy) |
| GET | `v1/shopping-list/{id}/` | `ShoppingListViewSet` | Retrieve shopping list (legacy) |
| PUT/PATCH | `v1/shopping-list/{id}/` | `ShoppingListViewSet` | Update shopping list (legacy) |
| DELETE | `v1/shopping-list/{id}/` | `ShoppingListViewSet` | Delete shopping list (legacy) |
| POST | `v1/shopping-list/{id}/transfer/` | `ShoppingListViewSet` | Transfer items to basket (legacy) |
| POST | `v1/shopping-list-item/` | `ShoppingListItemViewSet` | Add item to list (legacy) |
| DELETE | `v1/shopping-list-item/{id}/` | `ShoppingListItemViewSet` | Remove item from list (legacy) |
| GET | `v1/item/` | `ItemInventoryViewSet` | Search items in inventory |
| GET | `v2/shopping-list/` | `ShoppingListV2ViewSet` | List user's shopping lists (v2) |
| POST | `v2/shopping-list/` | `ShoppingListV2ViewSet` | Create shopping list (v2) |
| GET | `v2/shopping-list/{id}/` | `ShoppingListV2ViewSet` | Retrieve shopping list with items (v2) |
| PATCH | `v2/shopping-list/{id}/` | `ShoppingListV2ViewSet` | Update shopping list (v2, no PUT) |
| DELETE | `v2/shopping-list/{id}/` | `ShoppingListV2ViewSet` | Delete shopping list (v2) |
| POST | `v2/shopping-list/{id}/transfer/` | `ShoppingListV2ViewSet` | Transfer items to basket (v2) |
| POST | `v2/shopping-list-item/` | `ShoppingListItemV2ViewSet` | Add item to list (v2) |
| PATCH | `v2/shopping-list-item/{id}/` | `ShoppingListItemV2ViewSet` | Update item quantity (v2) |
| DELETE | `v2/shopping-list-item/{id}/` | `ShoppingListItemV2ViewSet` | Remove item from list (v2) |

**Pagination:** `SmallResultSetPagination` — `page_size=20`, `max_page_size=100`.

**Filters:** `store_id` (UUID), `search` (item name, icontains — v1 item endpoint only), ordering by `created_on`.

### Serializers (`serializers.py`)

| Serializer | Version | Purpose |
|------------|---------|---------|
| `ShoppingListItemBase` | Both | Abstract base with ownership validation (`validate_shopping_list()`) |
| `ShoppingListItemSerializer` | v1 | Legacy item serializer — nested `ItemInventorySerializer` on read |
| `ShoppingListItemV2Serializer` | v2 | Inventory service item — validates `item_variant_id` regex (`^[a-f0-9]{24}$`), fetches variant from `InventoryV1Client` on create |
| `StoreSerializer` | Both | Nested store details including `brand_logo` from store theme config |
| `ShoppingListSerializer` | v1 | List with items — N+1 query in `get_items()` |
| `ShoppingListV2BasicSerializer` | v2 | List/create — excludes items for performance |
| `ShoppingListV2DetailSerializer` | v2 | Retrieve — includes nested `ShoppingListItemV2Serializer` via `prefetch_related` |
| `BasketTransferSerializer` | Both | Response schema for transfer action (not used for serialization) |

**Key behavior:** `ShoppingListItemV2Serializer.create()` calls `InventoryV1Client.get_variant_by_id()` to fetch variant data from the external inventory service and stores it as a JSON snapshot. This data is **denormalized** — it does not auto-update if the variant changes.

### Business Logic: Transfer Service (`services/transfer_service.py`)

Two functions implement the "transfer shopping list to basket" feature:

| Function | Version | Dedup | Quantity | Click & Collect |
|----------|---------|-------|----------|-----------------|
| `transfer_to_basket(shopping_list, request)` | v1 | Skips items already in basket | Ignores (always 1) | No |
| `transfer_to_basket_inventory_service(shopping_list, customer)` | v2 | No dedup — duplicates possible | Respects `item.quantity` | Hardcoded `CLICK_AND_COLLECT` (with TODO to remove) |

Both functions add items **one-by-one** via `ItemManagementClient.add_item_to_basket()`. There is a FIXME in the code acknowledging this should be a bulk operation.

### Test Coverage

Comprehensive test suite across 6 files (~925 lines):

| File | Tests | Coverage |
|------|-------|----------|
| `test_models.py` | 1 | `ShoppingListItemV2.clean()` validation |
| `test_serializers.py` | ~8 | Serializer validation, save flows, variant error handling |
| `test_urls.py` | ~12 | Forward/reverse URL resolution for v1 and v2 |
| `test_views.py` | ~30 | Full integration: auth, CRUD, filtering, ordering, transfer, item operations, cross-user isolation |
| `conftest.py` | 3 fixtures | ShoppingList, ShoppingListItem, ShoppingListItemV2 |
| `factories.py` | 3 factories | Factory Boy factories with M2M hooks |

### Dependencies

| Depends On | Purpose |
|------------|---------|
| `dos.models.Store`, `dos.models.Customer` | Store scoping, customer ownership |
| `mishipay_core.models.BaseModel`, `BaseAuditModel` | UUID PK base classes |
| `mishipay_items.models.ItemInventory`, `Basket`, `PURCHASE_TYPES` | Legacy items, basket transfer |
| `mishipay_items.serializers` | `BasketVerboseSerializer`, `ItemInventorySerializer` |
| `inventory_service.client.InventoryV1Client` | Fetch variant data from inventory microservice |
| `microservices_client.ItemManagementClient` | Internal item management HTTP client for basket transfer |
| `mishipay_core.views.single_store_config` | Store theme/logo loading |

### Notable Patterns

1. **Dual v1/v2 architecture** — Parallel code paths for legacy monolith items and inventory microservice items. Both share the same `ShoppingList` model but use different item models.

2. **Denormalized JSON snapshot** — `ShoppingListItemV2.item_variant` stores a full copy of the variant data at creation time. Can become stale if product data changes in the inventory service.

3. **Two-layer authorization** — Queryset scoping (returns only current user's lists) plus serializer-level `validate_shopping_list()` ownership check on writes.

4. **N+1 query in v1** — `ShoppingListSerializer.get_items()` performs a separate query per shopping list. V2 avoids this via `prefetch_related` on retrieve.

5. **No admin, no tasks, no signals** — Purely API-driven with synchronous processing. Transfer operations (multiple HTTP calls) happen in the request-response cycle.

6. **Naming inconsistency** — `ShoppingList` uses `created_on`/`updated_on`; `ShoppingListItemV2` uses `created`/`updated` (different base classes). V2 serializers remap for API consistency.

---

## RFID Module (`rfid`)

### Purpose

A lightweight HTTP endpoint for **Keonn RFID hardware** to check whether items (identified by EPC tags) have been sold or are unsold. The module queries a Redis database where EPC sold statuses are stored by the order processing pipeline.

The module itself is minimal (4 files, ~90 lines of code) but sits at the center of a larger RFID ecosystem spanning multiple apps.

### Architecture

```
Keonn RFID Gate Hardware
    |
    | POST /rfid/keonn/check-epcs/  { "epcs": ["EPC1", "EPC2", ...] }
    v
rfid/views.py: Keonn (APIView)     ─── NO authentication, NO permissions
    |
    v
rfid/utils.py: check_epcs(epcs)
    |
    | redis.StrictRedis(settings.REDIS_HOST, 6379, db=0)
    | For each EPC: GET lowercase, fallback to uppercase
    v
Redis (mpay_redis:6379)
    |
    | "1" = sold, "0" = unsold/unknown
    v
Response: { "EPC1": "1", "EPC2": "0" }
    |── HTTP 200 if all sold
    |── HTTP 400 if any unsold (signals gate alarm)
```

### API Endpoint

| Method | Full Path | View | Auth | Purpose |
|--------|-----------|------|------|---------|
| POST | `/rfid/keonn/check-epcs/` | `Keonn` | **None** | Check EPC sold/unsold status |

**Request:**
```json
{ "epcs": ["EPC001ABC123DEF456", "EPC002ABC123DEF456"] }
```

**Response (200 — all sold):**
```json
{ "EPC001ABC123DEF456": "1", "EPC002ABC123DEF456": "1" }
```

**Response (400 — unsold items present, triggers gate alarm):**
```json
{ "EPC001ABC123DEF456": "1", "EPC002ABC123DEF456": "0" }
```

### Redis Lookup Logic (`utils.py`)

`check_epcs(epcs)` connects to `settings.REDIS_HOST:settings.REDIS_PORT` (defaults to `mpay_redis:6379`, db 0):

1. For each EPC string, queries Redis with both lowercase and uppercase keys (casing workaround)
2. If a lowercase result exists, uses that; otherwise falls back to uppercase
3. If Redis connection fails, EPC is treated as unsold (`"0"`) with a warning log
4. If EPC is not found in Redis, treated as unsold (`"0"`)
5. Returns `{"epc_statuses": {...}, "unsold_presence": bool}`

### The Broader RFID Ecosystem

The `rfid` module reads EPC status from Redis. The **write** side lives in `mishipay_retail_orders/rfid_status.py` (787 lines) which integrates with 6 RFID management platforms:

| Function | External Service | Purpose |
|----------|-----------------|---------|
| `set_sold_redis()` / `set_unsold_redis()` | Redis only | Basic EPC status tracking |
| `set_sold_mishipay_items()` / `set_unsold_mishipay_items()` | Redis + DB | Redis + `ItemInventory.stock_quantity` decrement |
| `set_sold_nedap()` / `set_unsold_nedap()` | Nedap Retail REST API | External RFID management |
| `set_sold_sato()` / `set_unsold_sato()` | SATO REST API | External RFID management (two-step unsold: returned → sellable) |
| `set_sold_decathlon()` / `set_unsold_decathlon()` | Decathlon Security REST API | EPC deactivation/reactivation |
| `set_sold_ilab()` / `set_unsold_ilab()` | Avery Dennison iLab SOAP | SOAP-based RFID status update |
| `set_sold_retail_reload()` / `set_unsold_retail_reload()` | Retail Reload (Mainetti) REST API | External RFID management |

**Dispatch:** `EPC_SOLD_FUNCTIONS_MAP` and `EPC_UNSOLD_FUNCTIONS_MAP` in `mishipay_retail_orders/app_settings.py` map store types to their RFID functions. Called during order completion and refund flows when the store's `rfid` display is `"RFID"` or `"RFID AND NON RFID"`.

**Basket procedure:** `RFIDStandardBasketProcedure` (`mishipay_items/procedures/rfid_standard.py`) enforces single-quantity, no-duplicate scanning for RFID items.

### Store-Level RFID Configuration

The `Store` model (DOS app) has an `rfid` field with display values:
- `"RFID"` — store uses RFID exclusively
- `"RFID AND NON RFID"` — store uses a mix

### Dependencies

| Depends On | Purpose |
|------------|---------|
| `redis.StrictRedis` | Redis connection for EPC lookups |
| `settings.REDIS_HOST`, `settings.REDIS_PORT` | Redis connection config (defaults: `mpay_redis`, `6379`) |
| `rest_framework.views.APIView` | DRF view base |

### Notable Patterns

1. **No authentication** — `permission_classes = []`, `authentication_classes = []`. Imports of `DashboardPermission`, `StorePermission`, and `TokenAuthentication` are commented out with `# noqa: F401`. This is a significant security concern.

2. **HTTP 400 for business logic** — The endpoint returns 400 to signal "unsold items present" rather than "invalid request." This is semantically incorrect but likely intentional — the Keonn hardware interprets non-200 as an alarm trigger.

3. **No connection pooling** — Creates a new `redis.StrictRedis` connection per call instead of using a connection pool or Django's cache framework.

4. **Case-sensitivity workaround** — Queries both `.lower()` and `.upper()` versions of each EPC, indicating inconsistent casing in the upstream data pipeline.

5. **Debug `print()` in production** — `views.py:27` has `print("epc infos ", epcs_info)`.

6. **URL name typo** — `rfid_keon` instead of `rfid_keonn`.

7. **No models** — The module is stateless from a database perspective, relying entirely on Redis. EPC fields exist on `BasketItem` and `ItemInventory` models in other apps.

8. **Fragmented ecosystem** — RFID logic is scattered across `rfid/`, `mishipay_retail_orders/rfid_status.py`, `mishipay_retail_orders/app_settings.py`, `mishipay_items/procedures/rfid_standard.py`, `dos/models`, and `mainServer/settings.py`. There is no central abstraction layer.

---

## Scheduled Jobs Module (`jobs`)

### Purpose

A collection of scheduled cron jobs powered by `django-cron` (v0.6.0). These jobs run daily at 21:00 UTC and produce two types of reports for Saturn and Decathlon retailers:

1. **Order summary Excel reports** emailed with `.xls` attachments
2. **Daily analytics summary emails** with inline HTML metrics

The module is not a registered Django app — it has no models, views, URLs, serializers, or admin. It is purely a consumer of the `dos` app's models.

### Architecture

```
django-cron framework
    |── python manage.py runcrons
    |
    v
jobs/order_crons.py
    |── SendSaturnOrderMailCronJob      → Excel report via xlwt
    |── SendDecathlonOrderMailCronJob   → Excel report via xlwt
    |
    v
jobs/daily_analytics_crons.py
    |── DecathlonDailyAnalyticsCronJob  → HTML analytics email
    |── SaturnDailyAnalyticsCronJob     → HTML analytics email
```

### Cron Job Registration (`settings.py:2177-2182`)

```python
CRON_CLASSES = [
    "jobs.order_crons.SendSaturnOrderMailCronJob",
    "jobs.order_crons.SendDecathlonOrderMailCronJob",
    "jobs.daily_analytics_crons.DecathlonDailyAnalyticsCronJob",
    "jobs.daily_analytics_crons.SaturnDailyAnalyticsCronJob",
]
DJ_CRONS_FROM_MAIL = "info@mishipay.com"
```

### Order Report Cron Jobs (`order_crons.py`)

**Abstract base:** `AbstractStoreOrderMailCronJob(CronJobBase)` — Template Method pattern.

**Workflow (`do()` method):**
1. Queries all `Store` objects matching `store_type`
2. For each store, calls `generate_sheet()` (from `mishipay_core/generate_orders_excel.py`) to create an `xlwt.Workbook` with 3 sheets: PROCESSING, FAILED, COMPLETED
3. Saves workbook to temp directory
4. Sends as email attachment via `EmailMessage` (HTML content subtype)
5. Cleans up temp file in `finally` block

**Concrete subclasses:**

| Class | Code | Store Type | Payment Filter | Schedule |
|-------|------|-----------|----------------|----------|
| `SendSaturnOrderMailCronJob` | `SendSaturnOrderSheetMailCronJob` | `SaturnSmartPayStoreType` | `xpay-paypal` | 21:00 |
| `SendDecathlonOrderMailCronJob` | `DecathlonStoreTypeCronJob` | `DecathlonStoreType` | None (all) | 21:00 |

**Recipients:** Hardcoded list of 4 MishiPay team emails (`saqlain@`, `tanvi@`, `theo@`, `ayesha@`).

**Excel generation (`mishipay_core/generate_orders_excel.py`):** Uses `xlwt` to generate `.xls` workbooks. Default date range: last 24 hours. Columns: Date, Time, OrderId, OrderSequenceId, ItemDetails (Name/Barcode/Price), OrderAmount.

### Daily Analytics Cron Jobs (`daily_analytics_crons.py`)

**Abstract base:** `AbstractDailyAnalyticsCronJob(CronJobBase)` — Template Method pattern.

**Workflow (`do()` method):**
1. Queries non-demo stores matching `store_type`
2. For each store, computes daily metrics from orders (today 00:00-23:59 UTC)
3. Sends HTML email with inline metrics

**Metrics computed per store:**

| Metric | Computation |
|--------|-------------|
| Number of Transactions | `successful_orders.count()` |
| Number of Items Purchased | Sum of `quantity`/`qty` from `order.item_details` JSON |
| Total Revenue | Sum of `discounted_price` (or `price` if no discount) |
| New Customers | Customers in today's orders whose `date_joined` >= today |
| New Registrations | `Customer` objects joined today at this store |
| Registration Platform Breakdown | Facebook / Gmail / Guest User / Email |
| Payment Method Map | Count by `payment_platform` |
| Same-Day Repeat Customers | Customers with 2+ orders today (count, transactions, revenue) |
| Different-Day Repeat Customers | Customers who registered before today (count, transactions, revenue) |

**Concrete subclasses:**

| Class | Code | Store Type | Success Statuses | Schedule |
|-------|------|-----------|-----------------|----------|
| `DecathlonDailyAnalyticsCronJob` | `DecathlonDailyAnalyticsCronJob` | `DecathlonStoreType` | `payment_completed`, `order_correct`, `payment_verified`, `FULL`, `PARTIAL` | 21:00 |
| `SaturnDailyAnalyticsCronJob` | `SaturnDailyAnalyticsCronJob` | `SaturnSmartPayStoreType` | `payment_completed`, `order_correct`, `payment_verified` | 21:00 |

**Note:** Decathlon considers full and partial refunds as "successful" for analytics; Saturn does not.

**Recipients:** Hardcoded list of 6 MishiPay team emails (4 from order crons + `mustafa@`, `sounak@`).

### Dependencies

| Depends On | Purpose |
|------------|---------|
| `django_cron` | `CronJobBase`, `Schedule` — cron job framework |
| `dos.models.Store`, `Order`, `Customer` | Data source for reports |
| `mishipay_core.generate_orders_excel` | `generate_sheet()` — Excel workbook generation |
| `xlwt` | Excel `.xls` generation (legacy format) |
| `django.core.mail` | `EmailMessage`, `send_mail` — email delivery |
| `mainServer.constants` | Order status constants (`PAYMENT_COMPLETED`, etc.) |

### Notable Patterns

1. **Template Method pattern** — Abstract base classes define the workflow; subclasses override configuration attributes only.

2. **Hardcoded email recipients** — Both files contain hardcoded personal email addresses. The two files have different recipient lists (4 vs 6 recipients).

3. **Timezone handling bug** — `daily_analytics_crons.py` uses `datetime.datetime.now().replace(tzinfo=pytz.utc)` which attaches UTC to the local server time rather than converting. Should use `datetime.datetime.now(tz=pytz.utc)` or `django.utils.timezone.now()`.

4. **`print()` instead of logging** — Extensive `print()` statements throughout `daily_analytics_crons.py` (logging is imported but unused, suppressed with `# noqa: F401`).

5. **No error handling within store loop** — If analytics computation fails for one store (e.g., `item_details` is `None`), the entire cron crashes and subsequent stores are skipped.

6. **ValidationError bug** — `order_crons.py:33` passes `'filename'` as the second positional arg (the `code` parameter) to `ValidationError` instead of appending it to the message string.

7. **`xlwt` is unmaintained** — Generates legacy `.xls` format (Excel 97-2003). `openpyxl` for `.xlsx` would be more appropriate.

8. **No retry or queueing** — If email sending fails, data is lost for that run.

9. **No tests** — No test files exist for the `jobs` module.

---

## Pub/Sub Messaging Module (`pubsub`)

### Purpose

A **Kafka-based publish/subscribe messaging framework** for the MishiPay platform. Implements the producer-consumer pattern using **Confluent Kafka** (`confluent_kafka` library) with SASL/SSL authentication. The module provides:

- A base framework (abstract producer, consumer, message, payload classes)
- 6 concrete producer adapters for publishing domain events
- 3 concrete consumers that subscribe to Kafka topics and dispatch to handler classes
- 3 Django management commands to run consumers as long-lived daemon processes

The module has no Django models, views, URLs, serializers, or admin. It is purely an infrastructure/messaging module.

### Architecture

```
Producers (called from business logic)          Consumers (long-lived daemon processes)
─────────────────────────────────                ────────────────────────────────────────
PostPaymentSuccessProducerAdapter  ──────>  mishi.streams.post-payment-success
                                                  └── PostPaymentSuccessConsumer
                                                        └── PostPaymentSuccessHandler

PostRefundSuccessProducerAdapter   ──────>  mishi.streams.common  [key=POST_REFUND_SUCCESS]
PostDashboardVerficationAdapter    ──────>  mishi.streams.common  [key=POST_DASHBOARD_VERIFICATION_SUCCESS]
(external producer)                ──────>  mishi.streams.post-transaction-success  [key=POST_TRANSACTION_SUCCESS]
                                                  └── CommonConsumer (routes by key)
                                                        ├── PostRefundSuccessHandler
                                                        ├── PostDashboardVerificationSuccessHandler
                                                        └── PostTransactionSuccessHandler

PostTransactionProducerAdapter     ──────>  mishi.streams.post-transaction
                                                  └── (consumed by external service)

TriggerPoslogAdapter               ──────>  flying_tiger.order.post_transaction
                                                  └── (consumed by external service)

CashDeskEndShiftEventAdapter       ──────>  flying_tiger.cash_desk.staff_shifts.end_shift_event
                                                  └── (consumed by external service)

(external producer)                ──────>  queuing.external.payments.analytics.json
                                                  └── PaymentOnlyClientsAnalyticsConsumer
                                                        └── ExternalPaymentsAnalyticsHandler
```

### Kafka Topics (`topics.py`)

| Constant | Topic String | Direction | Purpose |
|----------|-------------|-----------|---------|
| `POST_PAYMENT_SUCCESS_TOPIC` | `mishi.streams.post-payment-success` | Produce + Consume | Payment success events |
| `COMMON_TOPIC` | `mishi.streams.common` | Produce + Consume | Multiplexed: refund, dashboard verification |
| `POST_TRANSACTION_TOPIC` | `mishi.streams.post-transaction` | Produce only | Order fulfilment events |
| `POST_TRANSACTION_SUCCESS_TOPIC` | `mishi.streams.post-transaction-success` | Consume only | Transaction success (fiscalisation) |
| `FTC_POST_TRANSACTION` | `flying_tiger.order.post_transaction` | Produce only | Flying Tiger poslog events |
| `CASH_DESK_END_SHIFT_TOPIC` | `flying_tiger.cash_desk.staff_shifts.end_shift_event` | Produce only | Flying Tiger shift-end events |
| `EXTERNAL_PAYMENTS_ANALYTICS_TOPIC` | `queuing.external.payments.analytics.json` | Consume only | External payment analytics ingestion |
| `TEST1` | `test1` | — | Dev/test topic |

### Base Framework (`base/`)

#### `BasePayload` (`base/payload.py`)

Dynamic data container. Accepts arbitrary `**kwargs` and sets them as instance attributes. Provides `to_dict()` for serialization and `to_instance()` class method factory.

#### `BaseMessage` (`base/message.py`)

Message envelope with `encode()` and `decode()` methods. Envelope format:
```json
{
  "data": { /* payload.to_dict() */ },
  "server_timestamp": 1700000000000  /* ms since epoch */
}
```

#### `BaseProducer` (`base/producer.py`)

Wraps `confluent_kafka.Producer` with SASL/SSL configuration:

| Setting | Value |
|---------|-------|
| `bootstrap.servers` | `settings.CONF_KAFKA_BOOTSTRAP_SERVER` |
| `security.protocol` | `SASL_SSL` |
| `sasl.mechanisms` | `PLAIN` |
| `sasl.username` | `settings.CONF_KAFKA_API_KEY` |
| `sasl.password` | `settings.CONF_KAFKA_API_SECRET` |
| `request.timeout.ms` | `30000` |
| `delivery.timeout.ms` | `60000` |

Calls `flush()` after every single produce call (eliminates batching). Errors are logged but not re-raised (silently swallowed).

#### `BaseProducerAdapter` (`base/adapter.py`)

Template method orchestrator. Subclasses configure `topics`, `payload_class`, `payload_key`, `payload_data`. The `produce()` method builds payload → encodes message → produces to all configured topics.

#### `BaseConsumer` (`base/consumer.py`)

Wraps `confluent_kafka.Consumer`. The `process()` method subscribes to topics and enters an infinite polling loop (`poll(0.1)`). Subclasses implement `handle_message(key, message)`.

### Producer Adapters (`adapters/`)

| Adapter | Topic(s) | Payload Key | Payload Data | Called From |
|---------|----------|-------------|--------------|-------------|
| `PostPaymentSuccessProducerAdapter` | `mishi.streams.post-payment-success` | `order_id` | `payment_id`, optional `gateway_reference`, `final_amount`, `mspay_payment_id` | `mishipay_retail_payments/ms_pay.py`, `mspay_payment_response_handler.py` |
| `TriggerPoslogAdapter` | `flying_tiger.order.post_transaction` | None | `poslog_payload` dict | `mishipay_dashboard/util.py`, `mishipay_retail_orders/models.py`, FT management command |
| `PostRefundSuccessProducerAdapter` | `mishi.streams.common` | `order_id` | `order_id`, `refund_id`, `consumer_identifier_key: POST_REFUND_SUCCESS` | `mishipay_dashboard/util.py` |
| `PostDashboardVerficationProducerAdapter` | `mishi.streams.common` | `order_id` | `order_id`, `consumer_identifier_key: POST_DASHBOARD_VERIFICATION_SUCCESS` | `mishipay_dashboard/views.py` |
| `PostTransactionProducerAdapter` | `mishi.streams.post-transaction` | `order_id` | `external_id`, `order_id`, `store_type`, `payment_status`, `**kwargs` | `mishipay_retail_orders/order_fulfilment.py` |
| `CashDeskEndShiftEventProducerAdapter` | `flying_tiger.cash_desk.staff_shifts.end_shift_event` | None | `shift_data` dict | `mishipay_cashier_kiosk/views.py` |

### Consumers (`consumers/`)

#### `PostPaymentSuccessConsumer`

| Property | Value |
|----------|-------|
| **Topic** | `mishi.streams.post-payment-success` |
| **Consumer group** | `PAYMENT_GROUP` |
| **Max retries** | 3 |
| **Handler** | `PostPaymentSuccessHandler` → `PaymentService.payment_success_async_tasks()` |

#### `CommonConsumer`

| Property | Value |
|----------|-------|
| **Topics** | `mishi.streams.common`, `mishi.streams.post-transaction-success` |
| **Consumer group** | `COMMON_GROUP` |
| **Max retries** | 5 |

**Routing (via `consumer_identifier_key` in payload):**

| Key | Handler | Purpose |
|-----|---------|---------|
| `POST_REFUND_SUCCESS` | `PostRefundSuccessHandler` | Refund fulfilment + receipt email |
| `POST_DASHBOARD_VERIFICATION_SUCCESS` | `PostDashboardVerificationSuccessHandler` | Order fulfilment |
| `POST_TRANSACTION_SUCCESS` | `PostTransactionSuccessHandler` | Fiscalisation record update + receipt email |

#### `PaymentOnlyClientsAnalyticsConsumer`

| Property | Value |
|----------|-------|
| **Topic** | `queuing.external.payments.analytics.json` |
| **Consumer group** | `PAYMENT_GROUP` |
| **Max retries** | 5 |
| **Credentials** | **Separate** Kafka credentials (`CONF_KAFKA_PAYMENTS_API_KEY/SECRET`) |
| **Handler** | `ExternalPaymentsAnalyticsHandler` → Creates/updates `AggregatedAnalytics` records |

### Management Commands

| Command | Consumer | Purpose |
|---------|----------|---------|
| `python manage.py consumer_common` | `CommonConsumer` | Multi-handler consumer daemon |
| `python manage.py consumer_post_payment_success` | `PostPaymentSuccessConsumer` | Payment success daemon |
| `python manage.py consumer_payment_only_clients_analytics` | `PaymentOnlyClientsAnalyticsConsumer` | External analytics ingestion daemon |

All commands enter an infinite polling loop — designed to run as long-lived daemon processes (via supervisor, systemd, or container orchestration).

### Kafka Infrastructure Configuration

| Setting | Source |
|---------|--------|
| `CONF_KAFKA_BOOTSTRAP_SERVER` | `os.environ` |
| `CONF_KAFKA_API_KEY` | `os.environ` |
| `CONF_KAFKA_API_SECRET` | `os.environ` |
| `CONF_KAFKA_PAYMENTS_API_KEY` | `os.environ` (separate cluster/ACL) |
| `CONF_KAFKA_PAYMENTS_API_SECRET` | `os.environ` (separate cluster/ACL) |

Two sets of credentials: main (producers + 2 consumers) and payments (analytics consumer only).

### Error Handling Pattern

All consumers follow the same pattern:
1. Attempt to process message
2. On failure: close Django DB connection (handle stale connections in long-running process)
3. Retry up to max_retries (3 or 5)
4. On exhaustion: send Slack alert to `#alerts_kafka` channel, drop message

### Dependencies

| Depends On | Purpose |
|------------|---------|
| `confluent_kafka` | Kafka `Producer`, `Consumer`, `KafkaException` |
| `django.conf.settings` | Kafka bootstrap server, API keys |
| `django.db.connection` | DB connection reset on error |
| `mishipay_retail_payments.pubsub.handlers` | Payment/refund/verification handlers |
| `fiscalisation.pubsub.handlers` | Transaction success handler |
| `mishipay_core.common_functions.send_slack_message` | Alerting on retry exhaustion |

### Notable Patterns

1. **Template method / adapter pattern** — Clean separation between framework (`base/`) and business logic (`adapters/`, `consumers/`). Subclasses override configuration, not behavior.

2. **Topic multiplexing (envelope pattern)** — `mishi.streams.common` carries multiple event types, routed by `consumer_identifier_key` in the payload. Keeps topic count small but adds consumer complexity.

3. **DB connection reset on error** — All consumers call `connection.close()` on exception. This is a Django pattern for handling stale connections in long-running daemon processes (outside normal request-response lifecycle).

4. **Mutable default argument bug** — `BaseProducerAdapter.produce()` defaults `timestamp=datetime.utcnow()` which is evaluated once at class definition time. All messages produced without an explicit timestamp share the module import timestamp.

5. **`sync_produce` and `async_produce` are identical** — Both methods have the same implementation. `BaseProducer.produce()` always calls `flush()`, so there is no actual async path.

6. **No dead-letter queue (DLQ)** — Failed messages are dropped after max retries with only a Slack alert. No persistent failure record for later reprocessing.

7. **No graceful shutdown** — Infinite `while True` loop with no signal handling. Relies on container/process manager for termination.

8. **Consumer group collision risk** — `PostPaymentSuccessConsumer` and `PaymentOnlyClientsAnalyticsConsumer` both use `group.id: "PAYMENT_GROUP"` but subscribe to different topics. This can cause unexpected Kafka rebalancing.

9. **Missing handler fallthrough** — `CommonConsumer.get_handler()` returns `None` if `consumer_identifier_key` doesn't match any known key, causing `AttributeError` on `handler.process()`.

10. **Typo in class name** — `PostDashboardVerficationProducerAdapter` (missing "i" in "Verification").

11. **No schema validation** — Payloads are untyped dicts with no compile-time or runtime schema enforcement.

12. **No RabbitMQ** — Despite the task description mentioning RabbitMQ, this module is exclusively Kafka-based. RabbitMQ integration was not found.

---

## Saturn Module (`saturn`)

### Purpose

The original Saturn retailer integration, built for Saturn's Innsbruck (Austria) store. This module implements the `SaturnStoreType` — a concrete `StoreType` subclass that handles item lookup from Saturn's DFL Media-Saturn product APIs, order creation with PayUnity/PayPal payment processing, SOAP-based fulfilment order submission to Saturn's backend, and Redis-backed EPC sold/unsold status tracking.

The module also includes a `SaturnItemMap` model for mapping barcode/EPC/EAN identifiers to product metadata, and a management command for generating and emailing CSV sales reports.

### Architecture

```
Mobile App / Web
    |
    | get_item(barcode)
    v
SaturnStoreType (store_type.py)
    |── Parses barcode: NFC EPC ("httpssaturnmishipaycomtag...")
    |   or raw EAN barcode
    |
    |── EPC path: SaturnItemMap.objects.get(uid=epc) → resolve EAN
    |── EAN path: pad to 13 digits
    |
    |── Async fetch (asyncio):
    |   ├── get_product()       → DFL /v1/api/products/{ean}
    |   ├── get_product_asset() → DFL /v1/api/products/{ean}/images
    |   └── get_product_features() → DFL /v1/api/products/{ean}/features
    |
    |── Fallback: SaturnItemMap (demo kit items or API failure)
    |── Price override: items_map dict (hardcoded product_id → price)
    |
    v
create_order(data)
    |── PayUnity verify_payment() / PayPal verify_nonce()
    |── order_helper.create_order()
    |── create_fulfillment_order_request() → SOAP call via Zeep
    |── Mark items sold: SaturnItemMap.sold=True + Redis SET
    |── send_receipt() → Email
    |── publish_order() → Dashboard
    v
refund_order(data)
    |── FULL or PARTIAL return
    |── PayUnity refund / PayPal refund
    |── set_unsold() → SaturnItemMap.sold=False + Redis SET
    |── send_refund_confirmation_to_saturn() → Email to saturn.at
```

### Data Model

#### `SaturnItemMap` (`models.py`)

Maps physical item barcodes (including NFC EPC tags) to product identifiers and sold status.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `barcode` | CharField(63) | `unique=True` | Physical barcode string |
| `uid` | CharField(63) | nullable, blank | NFC EPC tag identifier |
| `ean` | CharField(63) | nullable, blank | European Article Number |
| `sold` | BooleanField | `default=False` | Whether item has been sold |
| `product_id` | CharField(63) | nullable, blank | Saturn product ID |
| `vendor` | CharField(127) | nullable, blank | Manufacturer name |
| `item_name` | CharField(127) | nullable, blank | Product display name |

**Meta:** Composite index on `(barcode, uid)` named `saturn_ind_1`. Table name: `saturn_item_maps`.

**Admin:** Registered with default `ModelAdmin`.

### Store Type: `SaturnStoreType` (`store_type.py`)

Extends `dos.base_store_type.StoreType`. This is the **first-generation** Saturn integration (Austria).

**Class attributes:**

| Attribute | Value | Purpose |
|-----------|-------|---------|
| `SERVER_URL` | `https://dfl.media-saturn.com/` | DFL product API base URL |
| `API_KEY` | `x-MishiPay-3d92` | DFL API key |
| `receipt_template` | `email/receipt.html` | Email template for purchase receipts |
| `outlet_id` | `745` | Saturn Innsbruck outlet ID |
| `items_map` | Dict (14 entries) | Hardcoded product_id → price overrides |

**Initialization:** Stores the `Store` object and creates a Redis connection to `gates.mishipay.com:6379`.

#### Item Lookup (`get_item()`)

1. Sanitizes barcode to alphanumeric characters
2. Detects NFC EPC tags via `httpssaturnmishipaycomtag` prefix → resolves via `SaturnItemMap(uid=epc)`
3. Fetches product details asynchronously using `asyncio` coroutines:
   - `get_product(data)` — `GET /v1/api/products/{ean}?locale=de_AT&apikey={key}&idType=gtin13`
   - `get_product_asset(data)` — `GET /v1/api/products/{ean}/images?locale=de_AT&apikey={key}&idType=gtin13`
   - `get_product_features(data)` — `GET /v1/api/products/{ean}/features?locale=de_AT&apikey={key}&idType=gtin13`
4. Falls back to `SaturnItemMap` for demo kit items (3 hardcoded EANs with fixed prices) or API failures
5. Applies price overrides from `items_map` dictionary
6. Also fetches sales price separately via `get_sales_price_for_product()` → `GET /v1/api/sales-prices/{product_id}/current?countryCode=AT&outletId=745`
7. Returns consolidated item dict: `price`, `ean`, `vendor`, `img` (from picscdn.redblue.de), `item_name`, `product_id`, `barcode`, `sold` status

**Additional methods:**
- `get_product_identity(data)` — Uses a separate Google Cloud API (`alice-expose-api-product-identifier`) to resolve product identity
- `set_sold(data)` / `set_unsold(data)` — Updates `SaturnItemMap.sold` and Redis key for RFID gate integration

#### Order Creation (`create_order()`)

1. Verifies payment via **PayUnity** (`payment.PAY_UNITY.verify_payment()`) or **PayPal** (`payment.PAYPAL.verify_nonce()`)
2. Creates order via `order_helper.create_order()`
3. Sends SOAP fulfilment order to Saturn's system via `create_fulfillment_order_request()`
4. Marks items as sold in `SaturnItemMap` and Redis
5. Sends purchase receipt email
6. Publishes order to dashboard

#### SOAP Fulfilment (`create_fulfillment_order_request()`)

Uses **Zeep** SOAP client with WS-Security (`UsernameToken`) to call Saturn's `doInsertFulfillmentOrder` service:

- **WSDL files:** `FulfillmentOrderManagement.wsdl` (demo) / `FulfillmentOrderManagement_prod.wsdl` (production)
- **Namespaces:** `http://www.media-saturn.com/esb/sdm/common/v1`, `http://www.media-saturn.com/esb/bdm/fulfillmentorder`
- **Request structure:** Order header (customer, address, payment info) + line items (product ID, EAN, quantity, price)
- **Hardcoded outlet:** `79` (Saturn Innsbruck)

#### Refund Processing (`refund_order()`)

Supports both `FULL` and `PARTIAL` refunds:
1. Calculates refund amount from matching item barcodes
2. Processes refund via PayUnity (`refund_payment()`) or PayPal
3. Updates item details with `refunded: True` and `refund_time`
4. Calls `set_unsold()` for refunded items (updates `SaturnItemMap` + Redis)
5. Sends refund notification email to `at.saturn.express@saturn.at`

#### Receipt & Reporting

- **Purchase receipt:** HTML email via `email/saturn_receipt.html` template. 20% tax rate. Sent from `at.saturn.express@kaufbestätigung.saturn.at`.
- **Refund notification:** Plain text email with order/customer/item details to Saturn team.
- **Sales report:** `SaturnSalesSummary` model (from `analytics` app), queried by date range.
- **Sold status:** `get_sold_status()` — batch lookup of barcode sold statuses from `SaturnItemMap`.

### Management Commands

#### `mail_sales_report` (`saturn/management/commands/mail_sales_report.py`)

Generates a CSV sales report for a hardcoded store ID and emails it.

| Argument | Required | Format | Purpose |
|----------|----------|--------|---------|
| `--from_date` | No | `ddmmYYYY` | Start date (defaults to today) |
| `--to_date` | No | `ddmmYYYY` | End date (defaults to from_date) |

**CSV format:** ORDER_ID, DATE, CUSTOMER_EMAIL, CUSTOMER_PHONE, PAYMENT_METHOD, REFUNDED, ITEM_EAN, PRODUCT_ID, ITEM_QUANTITY, ITEM_PRICE, TOTAL_VALUE, Transaction Count, Total Value, ATV

**Recipients:** Hardcoded (`tanvi@`, `mustafa@`, `sidharth@` at mishipay.com). Sent via SendGrid.

### Dependencies

| Depends On | Purpose |
|------------|---------|
| `dos.base_store_type.StoreType` | Abstract store type interface |
| `dos.models.Order`, `Store` | Order and store data |
| `dos.order.order_helper` | Order creation |
| `dos.payment.PAY_UNITY`, `PAYPAL` | Payment processing |
| `dos.util.publish_order` | Dashboard notification |
| `analytics.models.SaturnSalesSummary` | Sales report data |
| `config.loader.EMAIL_SETTINGS` | Email configuration |
| `redis.StrictRedis` | EPC sold/unsold status via Redis (`gates.mishipay.com:6379`) |
| `zeep` | SOAP client for Saturn fulfilment orders |
| `requests` | HTTP calls to DFL product APIs |

### Notable Patterns

1. **Async product detail fetching** — Uses `asyncio.coroutine` (legacy Python 3.4-era syntax with `yield from`) to concurrently fetch product details, assets, and features from DFL APIs. Creates a new event loop per request (`asyncio.new_event_loop()`).

2. **Dual item resolution** — NFC EPC tags are resolved via `SaturnItemMap` database lookup; regular barcodes go directly to Saturn's API. Demo kit items (3 hardcoded EANs) bypass the API entirely.

3. **Hardcoded price overrides** — `items_map` contains 14 product_id → price mappings that override API-returned prices. Purpose unclear (possibly demo/testing).

4. **WSDL-based SOAP integration** — Uses local WSDL files with Zeep for Saturn's fulfilment system. WS-Security credentials differ between demo (`MishiPay/MishiPay`) and production.

5. **Redis as RFID gate state** — EPC sold status is persisted to both `SaturnItemMap.sold` (database) and Redis key (for `rfid` module gate checks). See [RFID Module](#rfid-module-rfid) above.

6. **`@asyncio.coroutine` decorator** — Uses deprecated Python 3.4 coroutine syntax (`yield from`) rather than `async/await`. Should be updated for modern Python.

7. **No input sanitization on SOAP data** — Customer names and item descriptions are passed directly into the SOAP request without escaping or validation.

---

## Saturn Smart Pay Module (`saturn_smart_pay`)

### Purpose

The second-generation Saturn integration, built for Saturn's German stores (Hamburg and beyond). This module implements `SaturnSmartPayStoreType` — a more feature-rich store type that uses a different product API (CAR-NICE), supports NFC label scanning, integrates with XPay payment processing (credit card, PayPal, Apple Pay, Google Pay), and includes Pfand (bottle deposit) handling.

Unlike the original `saturn` module (which stores items locally in `SaturnItemMap`), this module fetches all item data live from Saturn's API on every scan — items are not stored in MishiPay's system.

### Architecture

```
Mobile App / Web
    |
    | get_item(barcode/product_label)
    v
SaturnSmartPayStoreType (store_type.py)
    |── Barcode path: pad to EAN-13 → CAR-NICE API /v1/car-nice-api/rest/awesome/eans/
    |── NFC label path: NFC API /v1/car-nfc/rest/NfcEslArticleMapping/{label}
    |   → resolves to product ID → CAR-NICE API /v1/car-nice-api/rest/awesome/
    |
    |── Filter product info (15+ fields)
    |── Age restriction check (department/group/category IDs)
    |── Pricing: salesPrice from realtimeData[outlet]
    |── Pfand (deposit): +€0.25 for applicable articles
    |
    v
create_order(data)
    |── Pfand line item injection
    |── order_helper.create_order()
    |── OrderMeta with verification_token
    |── Payment routing:
    |   ├── Google Pay → deferred to confirm_order()
    |   ├── Apple Pay → XPAY.make_payment() → confirm_order()
    |   └── XPay (card/PayPal) → XPAY.make_payment() → return payment_url
    v
confirm_order(data)
    |── Payment verification:
    |   ├── XPay live: verification_token match
    |   ├── XPay demo: follow redirect URL, check success URL
    |   ├── Google Pay: GooglePay.verify_nonce()
    |   └── Apple Pay: from create_order (pass-through)
    |── create_fulfillment_order_request() → SOAP
    |── send_receipt() → Email
    |── publish_order() → Dashboard
    v
refund_order(data)
    |── FULL refund only (partial refund blocked)
    |── Mark order.returned = True
    |── No payment reversal (manual process)
```

### Data Model

#### `SaturnSmartPayItemMap` (`models.py`)

Identical structure to `SaturnItemMap` in the `saturn` module.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `barcode` | CharField(63) | `unique=True` | Physical barcode string |
| `uid` | CharField(63) | nullable, blank | NFC identifier |
| `ean` | CharField(63) | nullable, blank | European Article Number |
| `sold` | BooleanField | `default=False` | Sold status |
| `product_id` | CharField(63) | nullable, blank | Saturn product ID |
| `vendor` | CharField(127) | nullable, blank | Manufacturer |
| `item_name` | CharField(127) | nullable, blank | Product display name |

**Meta:** Index on `(barcode, uid)` named `saturn_smart_pay_ind_1`. Table: `saturn_smart_pay_item_maps`.

**Note:** Despite this model existing, the `SaturnSmartPayStoreType` does not use it for item lookups. All item data is fetched live from Saturn's API. The model may be vestigial from the `saturn` module or used for edge cases not visible in the store type code.

### Constants (`constants.py`)

| Constant | Value | Purpose |
|----------|-------|---------|
| `PFUND_APPLICABLE_ARTICLE_NUMBERS` | `['1154505', '1326129', '2028716']` | Articles requiring Pfand (bottle deposit) |
| `PFUND_ARTICLE_NUMBER` | `'1661890'` | Product ID for the Pfand line item |
| `PFUND_PRICE` | `0.25` | Deposit amount per item (EUR) |

### Store Type: `SaturnSmartPayStoreType` (`store_type.py`)

Extends `dos.base_store_type.StoreType`. Second-generation Saturn integration (Germany).

**Class attributes:**

| Attribute | Value | Purpose |
|-----------|-------|---------|
| `SERVER_URL` | `https://dfl.media-saturn.com/` | DFL API base (unused — settings-driven) |
| `API_KEY` | `x-MishiPay-3d92` | DFL API key (unused — settings-driven) |
| `receipt_template` | `email/saturn-confirmation-de.html` | German-language receipt template |

**Initialization:** Loads per-store settings from `settings.STORE_DETAILS[saturn][demo|live][region][sub_region]`. Settings include: `client`, `language_code`, `api_host`, `api_key`, `get_item_outlet_id`, `create_order_outlet_id`, `outlet_id`, `fulfillment_username/password`, `fulfillment_wsdl_file`, `xpay_success_url`, `paypal_client_id`.

#### Item Lookup (`get_item()`)

1. Accepts either `barcode` or `product_label` (NFC)
2. **Barcode path:** Pads to EAN-13 → `GET {api_host}/v1/car-nice-api/rest/awesome/eans/MediaDE/de_DE?source=best&eans={ean}&outlet={outlet}&evaluateTemplate=true&evaluateFeatureFrame=true&apikey={key}`
3. **NFC label path:** `GET {api_host}/v1/car-nfc/rest/NfcEslArticleMapping/{label}?apikey={key}` → resolves to product ID → `GET {api_host}/v1/car-nice-api/rest/awesome/MediaDE/de_DE?products={id}&...`
4. Extracts and filters product information: `articleNumber`, `displayName`, `brandName`, `productEans`, `mainProductTax`, `mainFeatures`, `namedFeatures`, pricing from `realtimeData[outlet].salesPrice`
5. **Age restriction logic:** Item is restricted if:
   - Department ID in `[36, 39]` AND group ID not in allowed list `[710, 729, 701, 746, 747, 743]`
   - OR product group ID in a 39-entry disallowed list
   - OR category IDs intersect with `CAT_DE_SAT_10033` (FSK/USK 18) or `CAT_DE_MM_4895` (FSK 18)
   - OR price is negative (discount voucher)
6. **Pfand (deposit) handling:** If article number is in `PFUND_APPLICABLE_ARTICLE_NUMBERS`, adds `€0.25` to the price
7. **Image URLs:** Constructed from `assets.mmsrg.com/isr/166325/c1/-/{doi}/fee_786_587_png`

#### Order Creation (`create_order()`)

1. **Pfand line item injection:** If any item has a Pfand-applicable article number, subtracts `€0.25` from each item's price and adds a consolidated "PFAND" line item with `articleNumber: 1661890`
2. Creates order via `order_helper.create_order()`
3. Creates `OrderMeta` with a UUID `verification_token` for XPay payment verification
4. **Payment routing:**
   - **Google Pay:** Defers payment to `confirm_order()` (nonce verified there)
   - **Apple Pay:** Calls `XPAY.make_payment()` immediately, then chains to `confirm_order()`
   - **XPay (credit card/PayPal):** Calls `XPAY.make_payment()` → returns `payment_url` for frontend redirect

#### Order Confirmation (`confirm_order()`)

Separated from `create_order()` to handle redirect-based payment flows:

1. **XPay (live):** Matches `verification_token` from callback against stored `OrderMeta`
2. **XPay (demo):** Follows payment redirect URL, checks if final URL matches `xpay_success_url`
3. **Google Pay:** Calls `GooglePay.verify_nonce()` with order amount. Demo stores can be whitelisted in `settings.GOOGLE_PAY_PASS_STORES`
4. **Apple Pay:** Uses success status from `create_order()` pass-through. Demo stores whitelisted in `settings.APPLE_PAY_PASS_STORES`
5. On success: updates order status, creates SOAP fulfilment order, publishes to dashboard
6. On failure: sets status to `PAYMENT_FAILED` (or `XPAY_SAVED_CARD_FAILURE` for saved card flows)
7. Handles saved card orders: `XPAY_SAVED_CARD_ORDER` → `XPAY_SAVED_CARD_SUCCESS` / `XPAY_SAVED_CARD_FAILURE`

#### SOAP Fulfilment (`create_fulfillment_order_request()`)

Similar to the `saturn` module but settings-driven rather than hardcoded:

- **Credentials:** From `store_settings['fulfillment_username/password']`
- **WSDL:** From `store_settings['fulfillment_wsdl_file']`
- **Outlet ID:** From `store_settings['outlet_id']`
- **Order ID prefixed:** Uses `settings.ORDER_PREFIX` for retailer-specific prefixing
- **Address:** From `store.address_json` (parsed JSON) rather than hardcoded

#### Refund Processing (`refund_order()`)

- **Full refund only** — Partial refund explicitly blocked with error message
- Moves all `non_refunded_items` to `refunded_items`
- Sets `order.returned = True`
- **No payment reversal** — Unlike the `saturn` module, this does not call any payment processor for the refund. The refund appears to be order-status-only.

#### Receipt (`send_receipt()`)

- Template: `email/saturn-confirmation-de.html`
- Tax rate: 19% (German VAT vs 20% Austrian VAT in `saturn` module)
- Supports guest user emails
- Sent from: `saturn.hamburg.smartpay@saturn.de`
- Uses `EMAIL_SETTINGS.get_settings('transaction')` connection

### Management Commands

#### `saturn_order_migrations` (`saturn_smart_pay/management/commands/saturn_order_migrations.py`)

Migrates orders from the old architecture (`dos.models.Order`) to the new architecture (`mishipay_retail_orders.models.Order`).

**Migration logic:**
1. Queries old orders where `store.store_type = "SaturnSmartPayStoreType"` (excluding saved card orders)
2. For each old order:
   - Creates `Basket` + `BasketItem` objects (with extra_data for non-standard fields)
   - Resolves missing barcodes via `articleNumber` cross-reference
   - Maps refunded items by quantity comparison
   - Creates new `Order` with status mapping: `payment_completed` → COMPLETED, `order_correct` → VERIFIED, `order_incorrect` → VERIFICATION_FAILED
   - Maps `bundle_id` → platform: Saturn iOS/Android/Web and MishiPay iOS/Android/Web
   - Creates `Payments` record with payment method extraction (strips `xpay-`, `onepay-`, `adyen-`, etc. prefixes)

#### `saturn_saved_card_migrations` (`saturn_smart_pay/management/commands/saturn_saved_card_migrations.py`)

Migrates saved card data from `Customer.source`/`Customer.live_source` JSON fields to `mishipay_retail_payments.models.SavedCard`.

**Migration logic:**
1. Finds customers with `source__icontains='xpay'` across all Saturn Smart Pay stores
2. Parses JSON from `customer.source` (sandbox) and `customer.live_source` (production)
3. Creates `SavedCard` records with token, last 4 digits, brand mapping (MasterCard/VISA), expiry calculation

#### `mail_sales_report` (`saturn_smart_pay/management/commands/mail_sales_report.py`)

Identical to `saturn/management/commands/mail_sales_report.py` — generates and emails a CSV sales report. Same hardcoded store ID, same recipients, same format.

### Dependencies

| Depends On | Purpose |
|------------|---------|
| `dos.base_store_type.StoreType` | Abstract store type interface |
| `dos.models.Order`, `OrderMeta`, `Store` | Order and store data |
| `dos.order.order_helper` | Order creation |
| `dos.payment.XPAY`, `GooglePay`, `BasePayment` | Payment processing (XPay, Google Pay) |
| `dos.util.publish_order`, `get_query_param_dict`, `get_settings_store_map_name` | Dashboard, URL parsing, settings |
| `analytics.models.SaturnSalesSummary` | Sales report data |
| `config.loader.EMAIL_SETTINGS` | Email configuration |
| `mainServer.constants` | Payment status constants |
| `saturn_smart_pay.constants` | Pfand article numbers and price |
| `mishipay_retail_orders.models.Order` (new arch) | Migration target |
| `mishipay_items.models.Basket`, `BasketItem` | Migration target |
| `mishipay_retail_payments.models.Payments`, `PaymentVariant`, `SavedCard` | Migration target |
| `redis.StrictRedis` | Redis connection (via `__init__`) |
| `zeep` | SOAP client for fulfilment orders |
| `requests` | HTTP calls to Saturn APIs |
| `settings.STORE_DETAILS[saturn]` | Per-store configuration |

### Notable Patterns

1. **Two-phase order flow** — `create_order()` → frontend redirect → `confirm_order()`. This pattern supports redirect-based payment flows (XPay card, PayPal) where the user is sent to an external payment page and returned via callback.

2. **Pfand (deposit) accounting** — Pfand is split: the deposit amount is subtracted from each applicable item's price and consolidated into a single "PFAND" line item. This ensures the Saturn fulfilment system sees the correct per-item prices while MishiPay handles the deposit as a separate product.

3. **Age restriction via product hierarchy** — Item restriction is determined by a combination of department IDs, group IDs, product group IDs, and category IDs from Saturn's product API. The allowed/disallowed lists are hardcoded, not configurable.

4. **Settings-driven store configuration** — Unlike the first-generation `saturn` module (hardcoded values), this module reads all API endpoints, credentials, and outlet IDs from `settings.STORE_DETAILS`, allowing multiple German Saturn stores with different configurations.

5. **No payment reversal on refund** — `refund_order()` only updates order status; it does not call any payment processor. This contrasts with `saturn.SaturnStoreType.refund_order()` which processes PayUnity/PayPal refunds.

6. **Duplicated `mail_sales_report`** — Both `saturn` and `saturn_smart_pay` contain identical `mail_sales_report` management commands with the same hardcoded store ID. This is likely copy-paste duplication.

7. **Not-implemented methods** — `get_items()`, `add_item()`, `add_item_by_csv()`, `update_item()`, `delete_item()`, `apply_promotions()`, `pay()` all raise `NotImplementedError`. Items are fetched on-demand from Saturn's API, not managed in MishiPay's system.

8. **Migration commands as one-time scripts** — `saturn_order_migrations` and `saturn_saved_card_migrations` are data migration management commands for moving from the old order architecture to the new one. These are historical one-time migration scripts.

---

## Easy Store Creation Module (`easy_store_creation`)

### Purpose

A self-service store provisioning system that allows operations staff to rapidly create new demo stores by cloning from a master store template. The module handles store record creation, payment variant replication (via MS-Pay API), theme/branding copying, and provides bulk CSV upload/download for managing multiple stores, inventories, promotions, and tax configurations. It also generates Bitly campaign URLs for store QR codes.

### Architecture

```
Django Admin UI
    |
    v
StoreTemplate (admin form)
    |── Links a Retailer + Master Store → named template
    |── Name format: "{retailer}-{region}-{name}"
    |
    v
EasyStore (admin form or CSV bulk upload)
    |── Selects a StoreTemplate
    |── Provides: name, lat/long, region, currency, images
    |
    |── save() triggers:
    |   ├── save_store()          → Clone/update Store record from template
    |   ├── save_payment_variant() → Copy PaymentVariant via MS-Pay API × 2 (sandbox + live)
    |   └── save_theme()          → Copy Theme + ThemeProperty with image overrides
    |
    v
ListStore (proxy admin)                 Bulk Operations
    |── View/edit created stores             |
    |── QR code URL generation               |── CSV download: bulk_upload_stores.csv
    |── Inline theme editing                 |── CSV upload: EasyStoreAdminResource
    |── Inventory/Promotion/Tax upload       |── Bitly URL file generation
```

### Data Models

#### `StoreTemplate` (extends `BaseAuditModel`) — `models.py:29`

A reusable configuration template linking a retailer and master store.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `retailer` | FK → `mishipay_core.Retailer` | CASCADE | Auto-set from master store on save |
| `master_store` | FK → `dos.Store` | CASCADE | The source store to clone |
| `name` | TextField | | Auto-formatted: `{retailer}-{region}-{name}` |

**Meta:** `unique_together = ('master_store', 'name')`.

**`save()` behavior:** Strips any existing prefix from `name` (splits on last `-`), fetches the store's retailer, and formats name as `{retailer.name}-{store.region}-{name}`.

#### `EasyStore` (extends `BaseAuditModel`) — `models.py:52`

The main store provisioning record. Saving an `EasyStore` triggers the full store creation pipeline.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `template` | FK → `StoreTemplate` | CASCADE | Configuration template |
| `retailer_store_id` | TextField | blank | External store identifier |
| `use_master_inventory_store` | BooleanField | `default=False` | Share inventory with master store |
| `store_name` | TextField | | New store's display name |
| `region` | TextField | | ISO region code |
| `currency` | TextField | | Currency code |
| `currency_sign` | TextField | | Currency symbol |
| `longitude` | DecimalField(9,6) | -180.0 to 180.0 | Store location |
| `latitude` | DecimalField(9,6) | -90.0 to 90.0 | Store location |
| `address` | TextField | blank | Display address |
| `address_json` | TextField | nullable, blank, JSON-validated | Structured address |
| `timezone` | CharField(32) | blank | IANA timezone string |
| `extra_data` | JSONField | `default=dict`, blank | Additional configuration |
| `app_clip_id` | CharField(5) | blank | iOS App Clip identifier |
| `merchant_id` | TextField | blank | Payment merchant ID |
| `token_or_key` | TextField | blank | Payment token/key |
| `BrandImage` | URLField(1024) | nullable, blank | Brand image URL |
| `profileImage` | URLField(1024) | nullable, blank | Profile image URL |
| `BrandImageScanner` | URLField(1024) | nullable, blank | Scanner brand image URL |
| `storeSelectImage` | URLField(1024) | nullable, blank | Store selection image URL |
| `mapViewBanner` | URLField(1024) | nullable, blank | Map banner image URL |
| `mapViewPin` | URLField(1024) | nullable, blank | Map pin image URL |
| `mapViewlogo` | URLField(1024) | nullable, blank | Map logo image URL |

**Meta:** `unique_together = ('longitude', 'latitude')` — prevents duplicate stores at the same coordinates.

**`save()` pipeline:**

1. **`save_store()`** — Clones or updates a `Store` record:
   - If no store exists at the given lat/long → `deepcopy(template.master_store)` with `store_id=None` (creates new)
   - If store exists → updates in place
   - Copies all settings: region, currency, retailer, store properties (M2M)
   - Generates new `app_clip_id` (via `generate_app_clip_id()`)
   - If `use_master_inventory_store=True` → sets `parent_inventory_store`
   - Always sets `demo=True`, `via_easy_store=True`

2. **`save_payment_variant()`** — Called twice (sandbox + live):
   - Calls `settings.EASY_STORE_MS_PAY_VARAINT_SETUP_API` with `from_business_unit_id` (master) and `to_business_unit_id` (new store) to replicate variant on MS-Pay
   - Clones `PaymentVariant` record from master store via `deepcopy()`
   - Updates country and currency_code on the new variant

3. **`save_theme()`** — Copies theme configuration:
   - For each `Theme` on the master store (per platform: android/ios/web):
     - `get_or_create` Theme on new store
     - Deletes all existing `ThemeProperty` records
     - Copies all properties from master, overriding 7 image fields with EasyStore form values if provided

#### `ListStore` (proxy model) — `models.py:248`

A proxy on `dos.Store` for managing already-created Easy Store stores. Overrides `save()` to sync payment variant currency when the store's currency is changed.

### API Endpoints

**Base path:** `/easy_store/` (namespace: `easy_store_creation`)

All endpoints require authentication (`IsAuthenticated` + `SessionAuthentication`).

| Method | Path | View | Purpose |
|--------|------|------|---------|
| GET | `get_region/` | `GetRegion` | List distinct regions for a retailer (filter: `retailer_id`) |
| GET | `get_master_store/` | `GetMasterStore` | List master stores for a retailer+region (filter: `retailer_id`, `region`) |
| GET | `get_template_details/` | `GetTemplateDetails` | List store templates (filter: `id`, `name`) |
| GET | `download_template/` | `GetBulkEasyStoreTemplate` | Download CSV template for bulk store upload |
| GET | `inventory_promotion_tax_template/` | `InventoryPromotionTaxTemplate` | Download sample inventory or promotion CSV |
| GET/POST | `inventory_promotion_tax_upload/<store_id>` | `InventoryPromotionTaxUpload` | Upload inventory/promotion/tax CSV for a store |
| GET | `get_theme_url/` | `GetCloudinaryTheme` | Render Cloudinary theme upload form |
| GET/POST | `download_bitly_file/<store_ids>/` | `BitlyFileDownload` | Generate Bitly bulk URL CSV for QR codes |

### Serializers

| Serializer | Model | Fields | Purpose |
|------------|-------|--------|---------|
| `StoreRegionSerializer` | `Store` | `store_id`, `region`, `name`, `retailer`, `currency`, `currency_sign` | Region/store selection dropdowns |
| `StoreTemplateSerializer` | `StoreTemplate` | `retailer`, `region`, `master_store`, `name`, `currency`, `currency_sign` | Template details with derived `region`/`currency` from master store |

### Filters (`filters.py`)

| Filter | Fields | Purpose |
|--------|--------|---------|
| `RetailerFilter` | `retailer_id` (required UUID) | Filter stores by retailer |
| `MasterStoreFilter` | `retailer_id` + `region` (both required) | Filter stores by retailer and region |
| `EasyStoreTemplateFilter` | `id` (UUID, optional), `name` (optional) | Filter templates |

### Admin Configuration

#### `StoreTemplateAdmin` — `forms.py:81`

| Feature | Detail |
|---------|--------|
| Form | `StoreTemplateForm` — validates name uniqueness (as `{retailer}-{region}-{name}`) and prevents duplicate templates per master store |
| List display | id, name, retailer, master_store_name |
| Search | master_store name, retailer name, template name |
| Raw ID fields | `master_store` |

#### `EasyStoreAdmin` — `forms.py:98`

| Feature | Detail |
|---------|--------|
| Base | `ImportExportModelAdmin` (from `django-import-export`) + `ModelAdmin` |
| Form | `EasyStoreForm` — validates region matches master store when `use_master_inventory_store=True` |
| Resource | `EasyStoreAdminResource` — handles CSV import with `M:` prefix for required fields, lat/long validation |
| Change permission | **Disabled** — `has_change_permission()` returns `False` |
| Custom templates | `easy_store_form/change_form.html`, `easy_store_form/change_list.html` |

#### `StoreAdmin` (for `ListStore`) — `admin.py:57`

| Feature | Detail |
|---------|--------|
| Base | `NestedModelAdmin` (from `django-nested-inline`) |
| Inlines | `ThemeAdmin` → `ThemePropertyInlineAdmin` (nested, filtered to 8 image/branding fields) |
| List display | store_id, retailer, name, retailer_store_id, region, lat/long, demo, QR code URL, created |
| Search | store_id (exact), name, retailer name, retailer_store_id (exact), region (exact), lat/long |
| Queryset | `demo=True, active=True, via_easy_store=True`, ordered by newest |
| QR code | Generated via `qrcode.tec-it.com` API, encoding `https://{app}.mishipay.com/retailer/{store_id}` |
| List filter | Date range (created), retailer, region |
| Add/Delete | **Disabled** |

### Bulk CSV Upload: Store Data Upload Helper (`helpers/store_data_upload_helper.py`)

Central routing class for uploading inventory, promotion, and tax data to a store.

#### Inventory Upload (`upload_inventory()`)

Supports two paths based on store's `uses_inventory_service` flags:

**Path 1: Legacy BWG import** (when `uses_inventory_service_all` is False)
1. Maps Easy Store CSV columns to internal inventory format (14 → 22 columns)
2. Sets `country = store.region`, `requires_approval = "0"`
3. Tax handling: STATIC (new tax service) or DYNAMIC (old tax service)
4. If `clean=True`: deletes all existing `ItemInventory` and `ItemCategory` for the store
5. Calls `bwg_import()` from `mishipay_items`

**Path 2: Inventory Service import** (when `uses_inventory_service_any` is True)
1. Maps Easy Store CSV to inventory service format (14 → 18 columns)
2. Validates barcodes (max 50 chars), image URLs (falls back to store's BrandImage theme property)
3. Special handling: `DarAlAmiratStoreType` and `FloristStoreType` get barcode with leading digit stripped as alternate
4. GST support: `CGST` + `SGST` columns generate compound tax codes
5. Calls `standard_cafe_inventory_service_import()`

**Both paths run** if the store has mixed inventory service settings across platforms.

#### Promotion Upload (`upload_promotion()`)

Parses a CSV with 15-18 columns into `mpay_promo.Promotion` objects:

1. **Promotion families:** Easy (`e`), Combo (`c`), Requisite (`r`), Basket (`b`), EasyPlus (`p`)
2. **Discount types:** Amount (`v`), Percent (`p`), Fixed (`f`)
3. **Date formats:** `DD-MM-YYYY`, `DD/MM/YYYY`, `DD-MM-YYYY HH:MM:SS`, `DD/MM/YYYY HH:MM:SS`, `YYYY-MM-DD HH:MM:SS`
4. **Group/node structure:** Each promotion has one or more groups containing item nodes
5. **Special promotions:** `is_special=TRUE` creates availability `'s'` with `special_promo_info`
6. **Evaluate priority:** Auto-calculated from discount type/value if not specified
7. **Clean mode:** Deletes all existing promotions via `PromotionBatchOperation.clear_store()`
8. Skips expired promotions
9. Commits all promotions via `pbo.commit()` batch operation

#### Tax Upload (`upload_tax()`)

Creates tax table entries via `TaxClientSelector`:
- Tax rows are de-duplicated by `tax_code + tax_level + tax_percent + retailer_tax_level_id`
- Supports clean mode (deletes existing tax table)
- Used by both inventory upload paths when tax rows are generated

### CSV Template Downloads

#### Bulk Store Template (`GetBulkEasyStoreTemplate`)

Downloads a CSV with all `EasyStore` model fields as headers. Required fields prefixed with `M:`.

- Ignored fields: `id`, `created`, `updated`, `merchant_id`, `token_or_key`, `region`, `currency`, `currency_sign`
- Required fields: `template`, `store_name`, `latitude`, `longitude`

#### Inventory/Promotion Sample (`InventoryPromotionTaxTemplate`)

- **Inventory:** 14-column CSV with 11 sample product rows (MishiPay-branded sample data)
- **Promotion:** 18-column CSV with 12 sample promotion rows covering all family types (Easy, Combo, Requisite, Basket, EasyPlus)

### Bitly Campaign URL Generation (`BitlyFileDownload`)

Generates a CSV for Bitly bulk URL shortening:

1. Accepts comma-separated `store_ids` and a campaign count (1-9)
2. Validates all stores belong to the same retailer
3. Generates rows: Long URL, Backhalf, Tags, Title
4. URL format: `https://webapp.mishipay.com/retailer/{store_id}?utm_campaign={retailer_store_id}-{campaign_number}`
5. Backhalf format: `{ABBREVIATION}{retailer_store_id}-{campaign_number}`

### Resources (`resources.py`)

`EasyStoreAdminResource` — `django-import-export` resource for bulk CSV import:

- **`before_import()`:** Injects `id` column if missing; strips `M:` prefix from required field headers
- **`before_import_row()`:** Validates latitude (-90 to 90) and longitude (-180 to 180)
- **Excluded fields:** `created`, `updated`, `merchant_id`, `token_or_key`

### Dependencies

| Depends On | Purpose |
|------------|---------|
| `dos.models.Store`, `generate_app_clip_id` | Store creation and app clip ID generation |
| `dos.models.choices.CURRENCY_CHOICES` | Currency selection |
| `mishipay_core.models.BaseAuditModel` | UUID PK base class |
| `mishipay_core.models.Theme`, `ThemeProperty` | Theme management |
| `mishipay_core.models.Retailer` | Retailer FK |
| `mishipay_retail_payments.models.PaymentVariant` | Payment variant cloning |
| `mishipay_items.models.ItemInventory`, `ItemCategory` | Inventory deletion on clean |
| `mishipay_items.management.commands.bwg_inventory_process.bwg_import` | Legacy inventory import |
| `mishipay_items.mpay_promo` | Promotion object creation and batch operations |
| `mishipay_retail_orders.mpay_tax_new` | Tax table import |
| `mishipay_retail_orders.tax.TaxClientSelector` | Tax service client |
| `inventory_scripts.standard_cafe.main.standard_cafe_inventory_service_import` | Inventory service import |
| `settings.EASY_STORE_MS_PAY_VARAINT_SETUP_API` | MS-Pay variant replication endpoint |
| `django-import-export` | CSV import/export for `EasyStoreAdmin` |
| `django-nested-inline` | Nested admin inlines for theme editing |
| `rangefilter` | Date range filter in admin |

### Notable Patterns

1. **`deepcopy`-based cloning** — Store and PaymentVariant records are cloned via `copy.deepcopy()` with `id=None` / `store_id=None` to force database INSERT. This preserves all fields from the master while creating new records.

2. **Cascading save** — `EasyStore.save()` orchestrates store creation, payment variant setup, and theme copying in a single save operation. A failure in any step (e.g., MS-Pay API) is logged but does not prevent the `EasyStore` record from being saved.

3. **Always demo** — `EasyStore.save_store()` hardcodes `demo=True`. Easy Store–created stores are always demo stores by design.

4. **Two inventory paths** — The upload helper supports both legacy BWG import and the newer inventory service, running both if the store has mixed configuration. This mirrors the dual v1/v2 pattern seen in other modules.

5. **Promotion CSV parser** — A hand-rolled CSV parser (not using Python's `csv.DictReader`) processes promotion rows. Supports continuation rows (empty `promo_id`) for multi-group promotions, with state carried between rows via `last_promo`, `last_promo_group`, `pg_groups`.

6. **Typo in API URL setting** — `settings.EASY_STORE_MS_PAY_VARAINT_SETUP_API` — "VARAINT" instead of "VARIANT".

7. **No transactional safety** — The `EasyStore.save()` pipeline (store + 2 payment variants + theme) is not wrapped in `transaction.atomic()`. A failure partway through can leave the system in an inconsistent state.

8. **Disabled edit** — `EasyStoreAdmin.has_change_permission()` returns `False`, meaning Easy Store records cannot be edited after creation. This is intentional — the created `Store` record is edited via `ListStore` admin instead.

9. **Coordinate uniqueness** — `EasyStore` enforces `unique_together = ('longitude', 'latitude')`, but `save_store()` uses these coordinates to find existing stores. This means creating an `EasyStore` at existing coordinates updates the store rather than creating a new one.

10. **Print statements in production** — `store_data_upload_helper.py` contains multiple `print()` calls for debugging inventory and promotion uploads.

---

## Fiscalisation Module (`fiscalisation`)

### Purpose

A fiscal compliance module that ensures MishiPay transactions are registered with government tax authorities in countries requiring fiscalisation. Implements two distinct workflows for two retailer/country combinations: **Venistar/Retail Pro for Italy** and **Funky Buddha for Greece**.

The module bridges MishiPay's order system with external POS/fiscal systems that handle the legal registration of transactions, ensuring receipts carry government-required fiscal identifiers.

### Architecture

```
Workflow A: Venistar / Retail Pro (Italy)
─────────────────────────────────────────
Retail Pro POS (external)
    |
    |── GET  /v1/venistar/{retailer_id}/sale-sid/           ── Poll for oldest unfiscalised order
    |── GET  /v1/venistar/{retailer_id}/sale-sid-details/   ── Fetch order details for fiscal registration
    |── POST /v1/venistar/{retailer_id}/registration-process/ ── Bind fiscal registration_process_id to order
    |── POST /v1/venistar/{retailer_id}/dc-references/      ── Complete fiscalisation with document confirmation
    |
    v
VenistarFiscalisation model
    |
    v
venistar_fiscalisation_email_alert (cron) ── Email alert if fiscalisation stuck


Workflow B: Funky Buddha (Greece)
─────────────────────────────────
Order completion
    |
    v
PostTransactionSuccessHandler (pub/sub) ── Creates Fiscalisation record with external_id
    |
    v
Funky Buddha system (external)
    |── POST /v1/funky-buddha/register/ ── Register fiscalisation_id (AADE mark)
    |
    v
send_receipt_v2() ── Email receipt (requires mark ID to be legally valid)
```

### Data Models

Both models extend `BaseAuditModel` (`mishipay_core.models`), providing UUID primary key, `created` (`auto_now_add`), and `updated` (`auto_now`) fields.

#### `VenistarFiscalisation` (`models.py`)

Tracks the Venistar/Retail Pro fiscalisation lifecycle for Italian orders.

| Field | Type | Details |
|-------|------|---------|
| `order` | `OneToOneField` → `Order` | `on_delete=PROTECT`, `related_name="venistar_fiscalisation"` |
| `registration_process_id` | `TextField` | Region-unique fiscal ID from retailer, permanently bound to order once set |
| `dc_references` | `JSONField` | Document confirmation response from Retail Pro (defaults `{}`) |
| `is_complete` | `BooleanField` | Whether dc_references received and fiscalisation finished. Default `False` |

#### `Fiscalisation` (`models.py`)

Tracks the Funky Buddha / generic external fiscalisation lifecycle.

| Field | Type | Details |
|-------|------|---------|
| `order` | `OneToOneField` → `Order` | `on_delete=PROTECT`, `related_name="order_fiscalisation"` |
| `external_id` | `TextField` | Order ID from the retailer (GID). Indexed (`db_index=True`) |
| `fiscalisation_id` | `TextField` | Fiscal mark ID from the retailer/tax authority. Nullable |

### API Endpoints

| Method | Path | View | Purpose |
|--------|------|------|---------|
| GET | `v1/venistar/{retailer_id}/sale-sid/` | `GetSaleSIDView` | Poll for oldest unfiscalised order at a store |
| GET | `v1/venistar/{retailer_id}/sale-sid-details/` | `SaleSIDDetailsView` | Fetch full order details for fiscal registration |
| POST | `v1/venistar/{retailer_id}/registration-process/` | `RegistrationProcessView` | Bind a `registration_process_id` to an order |
| POST | `v1/venistar/{retailer_id}/dc-references/` | `DCReferencesView` | Complete fiscalisation with document confirmation references |
| POST | `v1/funky-buddha/register/` | `FunkyBuddhaRegisterFiscalisation` | Register a `fiscalisation_id` (AADE mark) for a transaction |

All Venistar endpoints are scoped by `retailer_id` (UUID) in the URL path. All endpoints require `FiscalisationPermission` (store-type + region group-based).

### Serializers

| Serializer | Model | Key Fields | Notes |
|------------|-------|------------|-------|
| `GetSaleSIDSerializer` | `Order` | `sale_sid` (from `order_id`) | Read-only, single field |
| `SaleSIDDetailsSerializer` | `Order` | `sale_sid`, `total_amount`, `completion_date`, `lottery_code`, customer info, `items`, `payment` | Most complex — 7 `SerializerMethodField` implementations |
| `RegistrationProcessSerializer` | `VenistarFiscalisation` | `sale_sid` (slug → order), `registration_process_id`, `state` | State is `ACCEPTED` (new) or `REJECTED` (duplicate) |
| `DCReferencesSerializer` | `VenistarFiscalisation` | `sale_sid`, `registration_process_id`, `dc_references` | Custom `validate()` looks up existing fiscalisation record |
| `FiscalisationSerializer` | `Fiscalisation` | `order_id`, `transaction_gid` (slug → `external_id`), `fiscalisation_id` | Used for Funky Buddha registration |

**`SaleSIDDetailsSerializer` business logic** (`serializers.py`):
- `get_items()` parses `basket.ms_promo_response["basket"]["items"]` to build itemised line items with `barcode`, `item_description`, `quantity`, `unit_price_with_tax`, `tax_rate`, `discount_amount`, `discount_reason`.
- Iterates both `discount_info` and `requisite_info` to calculate per-item discounts.
- **Integrity safeguard**: Compares computed `total_discount` against `ms_promo_response["basket"]["discount"]` and raises `RuntimeError` if they differ by more than 3 decimal places. Prevents fiscalisation with incorrect discount data.
- `get_lottery_code()` extracts the Italian government lottery code from `basket.special_promos` by scanning for program type `"IT_GOVT_LOTTERY_CODE"`.

### Country-Specific Fiscal Rules

#### Italy (Venistar Flow)

- **Lotteria degli Scontrini** (Receipt Lottery): Italian government program incentivising electronic receipts via lottery entry. The lottery code is stored in `basket.special_promos` under program type `"IT_GOVT_LOTTERY_CODE"` and passed through to the POS for fiscal registration.
- **Itemised Tax Reporting**: Each basket item includes `vat_rate` (tax rate), complying with Italian requirements for per-item tax breakdown.
- **Per-Item Discount Breakdown**: Italian fiscal law requires detailed discount reporting per line item. The serializer breaks down discounts from `discount_info` and `requisite_info` (from the promotions engine), reporting amount and reason (`"promotion"`) per item.
- **Document Confirmation (DC) References**: After fiscal registration, Retail Pro returns DC references proving the transaction was registered with Italian tax authorities. Stored as flexible `JSONField`.

#### Greece (Funky Buddha Flow)

- **AADE myDATA Mark ID**: Greek tax law requires electronic transactions to receive a unique "mark" (identification number) from AADE (Independent Authority for Public Revenue) via the myDATA platform. The `fiscalisation_id` field stores this mark.
- **Receipt Dependency**: `send_receipt_v2()` is intentionally deferred until the `fiscalisation_id` is recorded — the customer receipt must include the fiscal mark to be legally valid in Greece.

### Permissions Model

`FiscalisationPermission` (`permissions.py`) implements store-type and region-scoped access control:

- Constructs group name: `fiscalisation_manage_permission_{store_type}_{region}` (e.g., `fiscalisation_manage_permission_MujiStoreType_UK`)
- User must be authenticated AND belong to the matching Django auth group
- Object-level permission (checked per-order based on the order's store type and region)

### Pub/Sub Integration

`PostTransactionSuccessHandler` (`pubsub/handlers/post_transaction_success_handler.py`):
- Invoked from an external pub/sub subscription (wiring outside this module)
- Takes `order_id` and `fiscalisation_id` as constructor arguments
- Looks up the `Order` and `Fiscalisation` records
- Updates `fiscalisation.fiscalisation_id` with the provided value
- Triggers `send_receipt_v2()` for email receipt delivery
- Logs errors if `Fiscalisation` record not found or receipt sending fails

### Management Commands

#### `venistar_fiscalisation_email_alert`

Sends email alerts for Venistar fiscalisation records that started but did not complete within a timeout.

| Argument | Flag | Required | Description |
|----------|------|----------|-------------|
| `--store-type` | `-s` | Yes | Store type to filter by |
| `--region` | `-r` | Yes | Region code (2 letters, case-sensitive) |
| `--email-address` | `-e` | Yes | Recipient email address |
| `--timeout` | `-t` | Yes | Seconds after which incomplete fiscalisation is considered timed out |

Queries `VenistarFiscalisation` where `is_complete=False` and `created` older than timeout. Sends HTML email (`venistar-email-alert.html` template) listing Store ID, Order ID, and MishiPay Order ID for stuck orders.

### Admin Registrations

| Model | Admin Class | Key Config |
|-------|-------------|------------|
| `VenistarFiscalisation` | `VenistarFiscalisationAdmin` | `list_display`: id, order_id, registration_process_id, is_complete. `raw_id_fields`: order. `search_fields`: exact match on `order__order_id`, `order__o_id` |
| `Fiscalisation` | `FiscalisationAdmin` | `list_display`: id, order_o_id (custom), external_id, fiscalisation_id, created, updated. `list_filter`: created, updated. Ordered by `-created` |

### Migration History

| Migration | Date | Description |
|-----------|------|-------------|
| `0001_initial` | 2021-05-21 | Creates `VenistarFiscalisation` model (Django 2.2.22, postgres `JSONField`) |
| `0002_fiscalisation` | 2023-01-20 | Creates `Fiscalisation` model (Django 2.2.28) — added ~1.5 years later for Funky Buddha |
| `0003_alter_...` | 2023-02-28 | Migrates `dc_references` from postgres-specific to Django built-in `models.JSONField` (Django 4.1.7) |

### Test Coverage

- `tests/test_views.py` (629 lines) — Integration tests for all 5 views across 5 test classes. Covers invalid retailer UUIDs, missing orders, already-fiscalised orders, duplicate registration rejection, incorrect process IDs, already-complete fiscalisations, and success flows. Uses fixture data including Italian lottery codes and multi-layer promotion discount structures.
- `tests/test_permissions.py` (41 lines) — Tests `FiscalisationPermission` grant/deny.
- `tests/management/commands/test_venistar_fiscalisation_email_alert.py` (77 lines) — Tests email alert command filtering (timeout, region, completeness).
- `tests/factories.py` (24 lines) — Factory Boy factories for both models.

### Notable Patterns

1. **Dual-workflow architecture** — Two entirely separate fiscalisation flows (`VenistarFiscalisation` vs `Fiscalisation` models) for different countries, sharing the same Django app and URL namespace.
2. **Polling-based discovery** — The Venistar flow uses external polling (Retail Pro calls MishiPay) rather than MishiPay pushing to the POS, reflecting an integration constraint where the POS system initiates.
3. **Idempotent registration** — `RegistrationProcessView` returns `REJECTED` with existing data on duplicate registration rather than erroring, enabling safe retries.
4. **Discount integrity check** — `SaleSIDDetailsSerializer` raises `RuntimeError` if computed line-item discounts don't match the promotions engine total, preventing fiscalisation of orders with inconsistent discount data.
5. **Receipt gating** — Funky Buddha receipts are explicitly held until the fiscal mark is received, ensuring legal compliance of customer receipts in Greece.

---

## DDF Price Activation Module (`ddf_price_activation`)

### Purpose

Manages the price activation workflow for **Dubai Duty Free (DDF)** stores. When DDF's Retail Price Management (RPM) system generates price change files, this module ingests those files, presents them to DDF store staff for review and download, and then pushes the approved price updates to the MishiPay Inventory Service.

The module is entirely focused on the DDF retailer and implements a human-in-the-loop price update process where store staff must explicitly download and then activate price changes.

### Architecture

```
Phase 1: Ingestion (automated)
──────────────────────────────
DDF RPM System
    |
    v
RPM JSON files (dropped in directory via SFTP/etc.)
    |── RPM_REG_PRICE_CHANGE_DELTA_{date}_{time}.json
    |
    v
ddf_process_rpm_price_activation_file (management command)
    |── Detects file encoding (UTF-8/UTF-16/UTF-32 with BOM detection)
    |── Parses JSON: root key "NO_OF_RECORDS:{count}" → item array
    |── Creates RpmFiles + ItemsForPriceActivation records
    |── Moves processed files to success directory
    |
    v
PostgreSQL: RpmFiles, ItemsForPriceActivation tables


Phase 2: Download (manual, store staff)
───────────────────────────────────────
DDF store staff
    |── Login to Django admin → Price Activation Page
    |── Enter Staff Pass Number + Staff Number
    |── Click "Download"
    |
    v
DDFPriceActivatationActionDownload (POST)
    |── Validates staff credentials (DashboardUser lookup)
    |── Queries items: effective_date <= today (UAE), not activated
    |── Marks items as downloaded (first_downloaded_at, first_downloaded_by)
    |── Returns CSV file with 12 columns
    |
    v
Staff uses CSV to update physical shelf price labels


Phase 3: Activation (manual, store staff)
─────────────────────────────────────────
DDF store staff
    |── Click "Activate" (with JS confirmation dialog)
    |
    v
DDFPriceActivatationActionActivate (POST)
    |── Prerequisite: items must be downloaded first
    |── Deduplication: highest record_id per item_code wins
    |── Skips items with effective_date older than already-activated date
    |── Calls InventoryV1Client.update_inventory()
    |     └─ performInserts: false (update only, no new items)
    |── Marks all items as activated (atomic transaction)
    |
    v
MishiPay Inventory Service → item basePrice updated
```

### Data Models

#### `RpmFiles` (`models.py`)

Tracks every RPM price change file received from DDF.

| Field | Type | Details |
|-------|------|---------|
| `file_name` | `CharField(max_length=100, unique=True)` | RPM file name. Unique constraint prevents reprocessing |
| `processed_at` | `DateTimeField(auto_now=True)` | Timestamp auto-set on each save |
| `retailer_store_ids` | `ArrayField(CharField, size=100)` | PostgreSQL array of store/location IDs found in the file |
| `items_count` | `IntegerField` | Number of line items in the file |

#### `ItemsForPriceActivation` (`models.py`)

Core model — each record represents a single item whose price needs to change at a specific DDF store location.

| Field | Type | Details |
|-------|------|---------|
| `item_code` | `CharField(max_length=25)` | DDF item identifier (used as barcode in inventory system) |
| `price` | `DecimalField(max_digits=15, decimal_places=2)` | New price in AED |
| `retailer_store_id` | `CharField(max_length=25)` | DDF location ID |
| `rpm_file` | `ForeignKey` → `RpmFiles` | Source RPM file. `on_delete=PROTECT` prevents deletion of files with items |
| `record_id` | `IntegerField` | Unique record ID from RPM file (higher = newer, used for deduplication) |
| `effective_date` | `DateField` | Date when price change should take effect |
| `is_activated` | `BooleanField(default=False)` | Whether price has been pushed to inventory service |
| `activated_at` | `DateTimeField(null=True)` | When activation occurred |
| `activated_by` | `CharField(max_length=100, null=True)` | Name + staff number of activator |
| `is_downloaded` | `BooleanField(default=False)` | Whether item has been downloaded as CSV |
| `first_downloaded_at` | `DateTimeField(null=True)` | When first downloaded |
| `first_downloaded_by` | `CharField(max_length=100, null=True)` | Name + staff number of downloader |

**Database indexes (3):**
1. `loc_eff_date_actv_rpmi_idx` on `(retailer_store_id, effective_date, is_activated)` — activation query optimisation
2. `loc_eff_date_down_rpmi_idx` on `(retailer_store_id, effective_date, is_downloaded)` — download query optimisation
3. `eff_date_actv_rpmi_idx` on `(effective_date, activated_at)` — history lookups

#### `PriceActivationPage` (`models.py`)

A "virtual" model (`managed=False` in production) used solely to create an entry point in the Django admin sidebar. No database table — just a vehicle for custom permissions.

| Permission | Codename | Purpose |
|------------|----------|---------|
| DDF Price Activation Write | `ddf_price_activation_write` | Grants download and activate access |
| DDF Price Activation History | `ddf_price_activation_history` | Grants read-only history access |

### API Endpoints / Views

The module has no REST JSON API — it uses Django admin pages and CSV responses.

| Method | Path | View | Purpose |
|--------|------|------|---------|
| GET | _(Django admin changelist URL)_ | `DDFPriceActivatationPageView` | Renders the price activation HTML form |
| POST | `download/` | `DDFPriceActivatationActionDownload` | Downloads CSV of items pending activation |
| POST | `activate/` | `DDFPriceActivatationActionActivate` | Pushes prices to Inventory Service |

All views require `SessionAuthentication` + `IsAuthenticated` + `DdfPriceActivationFullAccessPermission`.

The download and activate URLs are served via both the app's `urls.py` and the Django admin's custom `get_urls()` override in `PriceActivationPageAdmin`, allowing access from the admin sidebar.

### Permissions

| Permission Class | Logic |
|------------------|-------|
| `DdfPriceActivationFullAccessPermission` | User must belong to `DdfPriceActivationFullAccess` auth group |
| `DdfPriceActivationHistoryPermission` | User must belong to either `DdfPriceActivationFullAccess` or `DdfPriceActivationHistoryAccess` |
| `ReadOnly` | Allows only safe HTTP methods (GET, HEAD, OPTIONS). Defined but not used in current views |

### Business Logic

#### File Ingestion (`management/commands/ddf_process_rpm_price_activation_file.py`)

**Invocation:**
```bash
python3 manage.py ddf_process_rpm_price_activation_file \
    --price-change-directory "/home/app/product_data/" \
    --price-change-success-directory-write "/home/app/product_data/success/"
```

**RPM JSON format:**
```json
{
    "NO_OF_RECORDS:5": [
        {
            "ITEM_CODE": "104600815",
            "SELLING_UNIT_RETAIL": 23.00,
            "LOCATION": "326",
            "RECORD_ID": 430919970,
            "EFFECTIVE_DATE": "27\\/09\\/2023 14:32:14.000"
        }
    ]
}
```

**Processing steps:**
1. Lists directory for files starting with `RPM_REG_PRICE_CHANGE`
2. Detects encoding via BOM (UTF-8-sig, UTF-16 LE/BE, UTF-32 LE/BE) with `chardet` fallback
3. Parses JSON, validates `NO_OF_RECORDS:{count}` root key format
4. Retrieves DDF store timezone from a `Store` with `store_type="DubaiDutyFreeStoreType"`
5. Creates `RpmFiles` record and bulk-creates `ItemsForPriceActivation` records (atomic transaction)
6. All-or-nothing: if any file fails, no files are moved to success directory

#### Download Logic (`views.py:107-235`)

1. Validates user has exactly one assigned store (price activation users must be single-location)
2. Validates DDF staff credentials via `DashboardUser` lookup (`DDF-{staff_number}-{staff_pass_number}`, `retailer_staff_pin="S"`)
3. Queries `ItemsForPriceActivation` for the user's store where `effective_date <= today (UAE)` and `is_activated=False`
4. Marks previously-undownloaded items with `is_downloaded=True`, `first_downloaded_at`, `first_downloaded_by`
5. Builds CSV with 12 columns: Item Code, Price (AED), Effective Date, Record ID, Location ID, Activation Status, Activated At, Activated By, Downloaded Status, First Downloaded At, First Downloaded By, RPM Filename
6. Returns as HTTP download attachment (`ddf-price-activation-downloaded-items-{datetime}.csv`)
7. Entire operation wrapped in `transaction.atomic()`

#### Activation Logic (`views.py:238-368`)

1. Same staff credential validation as download
2. Queries items for the user's store: effective date <= today (UAE), not activated, **already downloaded** (download is prerequisite)
3. **Deduplication logic:**
   - Fetches already-activated items from the past 10 weeks to build `item_code → latest effective_date` map
   - For pending items: groups by `item_code`, picks only the record with highest `record_id` (latest RPM record)
   - Filters out items whose `effective_date` is older than the already-activated effective date for that item code
4. Builds inventory update request for `InventoryV1Client`:
   ```json
   {
       "storeId": "...",
       "retailerId": "...",
       "categories": [],
       "performInserts": false,
       "items": [
           {"operation": "upsert", "barcodes": ["104600815"], "basePrice": "23.00"}
       ]
   }
   ```
5. Calls `InventoryV1Client(settings.INVENTORY_SERVICE_OPS_URL).update_inventory(body)`
6. Marks ALL items in queryset as activated (including duplicates that were filtered out of the API call)
7. Atomic transaction — if inventory service call fails, nothing is committed

### Django Admin UI

The price activation page is embedded in the Django admin using the `PriceActivationPage` dummy model. The `PriceActivationPageAdmin` class overrides `get_urls()` to serve:
- Root URL → activation page HTML form
- `activate/` → activation action
- `download/` → download action

All routes wrapped with `self.admin_site.admin_view()` for Django admin authentication.

**Template** (`templates/admin/activation_page.html`):
- Bootstrap 5.0.2 styling, jQuery 3.7.1 for AJAX
- Read-only fields: Location (`{retailer_store_id} - {store name}`), Date Today (UAE)
- Input fields: Staff Pass Number, Staff Number
- Actions: Download (blue button), Activate (red button with JS `confirm()` dialog)
- Status message display area, loading spinner

### External Integrations

| Integration | Direction | Details |
|-------------|-----------|---------|
| DDF RPM System | Inbound | JSON files dropped in directory. File naming: `RPM_REG_PRICE_CHANGE_DELTA_{date}_{time}.json`. Encoding varies. |
| MishiPay Inventory Service | Outbound | `InventoryV1Client` at `settings.INVENTORY_SERVICE_OPS_URL`. POST `/v1/inventory/update`. Updates-only (`performInserts: false`). |
| Dashboard User System | Internal | Staff credentials validated against `dashboard.models.User` with username pattern `DDF-{staff_number}-{staff_pass_number}` |

### Admin Registrations

| Model | Admin Class | Key Config |
|-------|-------------|------------|
| `RpmFiles` | `RpmFileAdmin` | `list_display`: id, file_name, items_count, processed_at. `search_fields`: file_name |
| `ItemsForPriceActivation` | `ItemsForPriceActivationAdmin` | `list_display`: 9 fields. `list_filter`: effective_date, is_activated, is_downloaded, retailer_store_id. `search_fields`: rpm_file__file_name, item_code |
| `PriceActivationPage` | `PriceActivationPageAdmin` | Custom `get_urls()` serving activation UI. No changelist — redirects to custom page |

### Migration History

| Migration | Date | Description |
|-----------|------|-------------|
| `0001_initial` | Oct 14, 2023 | Creates `RpmFiles` and `ItemsForPriceActivation` tables |
| `0002` | Oct 15, 2023 | Creates `PriceActivationPage` proxy model with custom permissions |
| `0003` | Oct 15, 2023 | `processed_at` → `auto_now=True`; `retailer_store_ids` array max size 100 |
| `0004` | Oct 15, 2023 | Adds `items_count`; `file_name` → unique; 2 database indexes |
| `0005` | Oct 16, 2023 | Renames activation index; adds download status index |
| `0006` | Oct 18, 2023 | Adds verbose names to all models |
| `0007` | Oct 23, 2023 | Sets `PriceActivationPage` to `managed=False` |

All 7 migrations span October 14-23, 2023, indicating rapid initial development.

### Test Coverage

- `tests/test_views.py` — Tests download and activate views. Setup creates a `DubaiDutyFreeStoreType` store, DashboardUser, 2 RPM files, and 5 `ItemsForPriceActivation` records (including duplicates and pre-activated items). Uses `requests_mock` to mock the Inventory Service endpoint.
- `tests/mock_data/download.csv` — Expected CSV output for test comparison (4 rows).

### Notable Patterns

1. **Human-in-the-loop price updates** — Download is a prerequisite for activation. Staff must review price changes before they are pushed to inventory. This prevents automated price errors from reaching customers.
2. **Deduplication with record_id** — When multiple RPM files contain price changes for the same item, only the record with the highest `record_id` is activated. Combined with 10-week lookback for already-activated items, this ensures monotonic price progression.
3. **Django admin as operational UI** — The `PriceActivationPage` dummy model pattern injects a custom operational page into Django admin, leveraging admin authentication and permissions without building a separate UI.
4. **All-or-nothing file processing** — The management command processes all files or rolls back entirely. Successfully parsed files are only moved to the success directory if ALL files process without error.
5. **Atomic activation** — The inventory service call and the database update marking items as activated are wrapped in `transaction.atomic()`. If the external call fails, no items are marked as activated.
6. **UAE timezone awareness** — All date comparisons use `Asia/Dubai` timezone via `get_date_in_uae_timezone()`, critical for correct effective date enforcement across time zones.
7. **No serializers** — Unlike most Django apps in this codebase, this module uses no DRF serializers. Data output is CSV; input is HTML form POST. The UI is rendered server-side within the Django admin.

---

## Open Questions

### Admin Actions
- Are the `TODO: Check permission` comments in `CreateStoreAction` and `CreatePaymentVariantAction` intentional, or should these actions verify specific permissions beyond the `AdminPrivilege.permission` gate?
- The `object_id` auto-increment in `CreateStoreAction` is not atomic. Has this caused duplicate `object_id` values in production?
- Is the dynamic `__import__()` in `AdminAction.run()` restricted to trusted `admin_action_class` values via database-level controls, or could a malicious admin record execute arbitrary code?
- The `SendEmailAction` logs `"[Admin Action] Running email admin action"` at `logger.error` level. Is this intentional or a logging level mistake?
- Are there additional action classes beyond the 5 documented in `actions_definitions.py`? The `admin_action_class` field supports any Python path.
- No tests exist for the `admin_actions` module. Is this intentional given the admin-only use case?

### Clients
- Is the hardcoded `/test/CA/` path in `AzureStorageClient.upload_blob_file()` correct for production use?
- Are there callers of `PromoServiceV2Helper` in the codebase, or is this a standalone utility? No imports of this class were found in the explored modules.
- The `PromoHeaderRow.header()` returns 18 columns but the class has 21 instance attributes — `description`, `retailer`, `is_active`, and `active_days` are excluded from CSV output. Is this intentional?

### Shopping List
- Is the v1 (legacy monolith item) code path still actively used, or has it been fully superseded by v2 (inventory service)? The deprecated `/api/v1/` mount is still active.
- The `transfer_to_basket_inventory_service()` (v2) does not check for existing basket items before adding, unlike v1. Could this create duplicate basket items on repeated transfers?
- `ShoppingListItemV2.item_variant` stores a denormalized JSON snapshot. Is there a mechanism to refresh stale variant data (e.g., if product prices change)?
- `transfer_to_basket()` (v1) ignores `ShoppingListItem.quantity` and always sends `item_quantity=1`. Is this intentional or a bug?
- The `brand_logo` field in `StoreSerializer` calls `single_store_config(store, "web")` to extract one image URL. Is there a more efficient DB-based lookup? (The code has a TODO noting this.)
- `transfer_to_basket_inventory_service()` hardcodes `purchase_type=CLICK_AND_COLLECT` with a TODO to remove. Is this feature flag still active?
- Shopping list models have no Django admin registration. Is this intentional, or should staff be able to view/manage lists?

### RFID
- Why is the `/rfid/keonn/check-epcs/` endpoint completely unauthenticated? The commented-out imports suggest authentication was intended but disabled. Is it protected at the infrastructure level (e.g., Kong gateway, internal-only network)?
- The `check_epcs()` utility queries Redis with both lowercase and uppercase EPC keys. What causes the casing inconsistency in the data pipeline that writes EPCs to Redis?
- Is Keonn the only RFID gate hardware vendor, or are other vendors integrated differently?
- Should the endpoint return HTTP 200 (with an `unsold_presence` boolean in the body) instead of HTTP 400 for unsold items? The current approach misuses 400 as a business-logic signal.
- The `rfid_status.py` functions for Nedap, SATO, Decathlon, and Avery Dennison have no retry logic for external API calls. Are EPC status updates reconciled if an external service is temporarily unavailable?
- Are the Nedap bearer tokens and SATO credentials in `settings.py` (`RFID_UPDATE_SETTINGS`) overridden by environment variables in production, or are the committed values the actual production credentials?

### Scheduled Jobs
- Are the hardcoded email recipients (`saqlain@`, `tanvi@`, `theo@`, `ayesha@`, `mustafa@`, `sounak@`) still the correct recipients for these reports? Should this be configurable via settings or database?
- Are these cron jobs still actively running in production? The retailers served (Saturn, Decathlon) may have their own reporting by now.
- The timezone handling in `daily_analytics_crons.py` uses `datetime.now().replace(tzinfo=pytz.utc)` which attaches UTC to local server time. Has this caused incorrect date boundaries for non-UTC servers?
- The `generate_orders_excel.py` utility uses `xlwt` (legacy `.xls` format, unmaintained). Are there plans to migrate to `openpyxl` (`.xlsx`)?
- Why does Decathlon count full and partial refunds as "successful" for analytics purposes while Saturn does not?

### Pub/Sub (Kafka)
- The `PostPaymentSuccessConsumer` and `PaymentOnlyClientsAnalyticsConsumer` share `group.id: "PAYMENT_GROUP"` but subscribe to different topics. Has this caused Kafka rebalancing issues?
- `BaseProducerAdapter.produce()` defaults `timestamp=datetime.utcnow()` — a mutable default argument bug. Is this causing all messages to share the module import timestamp, or do callers always pass an explicit timestamp?
- `CommonConsumer.get_handler()` returns `None` for unknown `consumer_identifier_key` values, causing `AttributeError`. Should there be a default/unknown handler or at least a logged warning?
- Are the `mishi.streams.post-transaction` and `flying_tiger.order.post_transaction` topics consumed by external services? No consumers for these topics exist in the codebase.
- Is there a dead-letter queue or persistent failure record for messages that exhaust all retries? Currently they are dropped after a Slack alert.
- Are the two sets of Kafka credentials (main vs payments) for separate clusters or ACL-scoped credentials on the same cluster?
- The `ExternalPaymentsAnalyticsHandler` creates `AggregatedAnalytics` records. This answers the open question from T21 about how `AggregatedAnalytics` is populated — it comes from external payment analytics data ingested via Kafka.

### Saturn
- The `items_map` dictionary in `SaturnStoreType` hardcodes 14 product_id → price overrides. What is the purpose of these overrides? Are they demo kit prices, promotional prices, or corrections for API inaccuracies?
- `SaturnStoreType.get_product_identity()` calls a Google Cloud `alice-expose-api` endpoint with a hardcoded API key. Is this still in use, or has it been superseded by the CAR-NICE API in `saturn_smart_pay`?
- The `@asyncio.coroutine` / `yield from` syntax is deprecated since Python 3.8 and removed in Python 3.11. Is the production environment running Python < 3.11?
- Redis is connected to `gates.mishipay.com:6379` in `SaturnStoreType.__init__()` but `settings.REDIS_HOST` (`mpay_redis`) in the `rfid` module. Are these the same Redis instance?
- The SOAP fulfilment in `saturn` uses outlet ID `79` (hardcoded), while `saturn_smart_pay` uses `store_settings['outlet_id']`. Is outlet 79 specific to Saturn Innsbruck?
- Is the Saturn Austria (Innsbruck) store still active, or has it been migrated to the `saturn_smart_pay` module?

### Saturn Smart Pay
- `SaturnSmartPayItemMap` exists but is not referenced anywhere in `SaturnSmartPayStoreType`. Is this model vestigial, or is it used by other parts of the system (e.g., RFID, admin tools)?
- `refund_order()` does not call any payment processor for refund reversal — it only marks the order as returned. Are Saturn Smart Pay refunds processed manually outside MishiPay?
- The age restriction logic uses hardcoded department/group/product group/category ID lists. How are these maintained when Saturn adds new restricted product categories?
- The `mail_sales_report` command is identical in both `saturn` and `saturn_smart_pay` with the same hardcoded store ID. Is this intentional duplication, or should they reference different stores?
- The `saturn_order_migrations` and `saturn_saved_card_migrations` management commands appear to be one-time migration scripts. Have these been run, and can they be removed?
- What is the current production status of XPay as a payment processor? The code references `xpay-`, `onepay-` payment platform prefixes in migrations.

### Easy Store Creation
- Is the `settings.EASY_STORE_MS_PAY_VARAINT_SETUP_API` endpoint always available? What happens to the store if this API call fails during `save_payment_variant()`?
- `EasyStore.save()` is not wrapped in `transaction.atomic()`. Has this caused inconsistent state (e.g., store created but payment variant missing)?
- The `save_store()` method reuses existing stores at matching coordinates via `Store.objects.filter(latitude=..., longitude=...)`. Is this intentional for updating existing demo stores, or could it accidentally overwrite an unrelated store?
- The Cloudinary image upload code in `EasyStore.create_cloudinary_image_url()` is fully commented out. Was this feature abandoned, or is it planned for future implementation?
- `EasyStore` always creates demo stores (`demo=True`). Is there a separate workflow for promoting an Easy Store demo to a production store?
- The `StoreDataUploadHelper.upload_inventory()` runs both legacy BWG import and inventory service import if the store has mixed platform settings. Could this cause duplicate items?
- Is the `DarAlAmiratStoreType` / `FloristStoreType` barcode-stripping hack (`barcode[1:]`) still needed, or is this specific to a decommissioned store type?

### Fiscalisation
- Is the Venistar flow used only for Italian retailers, or has it been extended to other countries with similar polling-based POS integration?
- How frequently does Retail Pro poll the `GetSaleSIDView` endpoint? Is there rate limiting or caching to handle high polling frequency?
- The `SaleSIDDetailsSerializer.get_items()` integrity check raises a bare `RuntimeError` on discount mismatch. Is this caught and handled upstream, or does it result in a 500 error to the Retail Pro client?
- `RetailerIDMixin` validates `retailer_id` as UUID but doesn't check that the retailer owns the store being queried. Is cross-retailer access prevented by the `FiscalisationPermission` group membership instead?
- The `PostTransactionSuccessHandler` is described as being invoked from an external pub/sub subscription, but the wiring is not in this module. Where is it configured — in the `pubsub` module, or in an external service?
- Are there plans to add more country-specific fiscalisation workflows (e.g., Germany's TSE/Fiskaly, Saudi Arabia's ZATCA Phase 2 e-invoicing)?
- The Funky Buddha `FunkyBuddhaRegisterFiscalisation` endpoint URL is not retailer-scoped (no `{retailer_id}` in path), unlike the Venistar endpoints. Is this intentional, or should it be scoped?

### DDF Price Activation
- The management command at line 91 appears to assign item objects instead of location IDs to `rpm_file.retailer_store_ids`: `rpm_file.retailer_store_ids = list(items_for_price_activation)` should likely be `rpm_file.retailer_store_ids = list(locations_in_file)`. Has this caused incorrect data in the `retailer_store_ids` field?
- `get_ddf_user_details()` splits the username on `'0'` to extract the staff name. This seems fragile — what happens for usernames containing `'0'` in the staff name or number?
- The download view at line 159 calls `items_for_download.order_by(...)` without reassigning. Since QuerySets are immutable, the ordering is discarded. Is the CSV output unordered?
- Step 6 of activation marks ALL items in `items_to_activate` queryset as activated, including duplicates that were filtered out of the inventory API call. Is this intentional (mark all as "processed") or should only the deduplicated items be marked?
- Is there retry logic if the `InventoryV1Client.update_inventory()` call fails? The atomic transaction prevents database marking, but does the management command or scheduler retry?
- The `PriceActivationPage` model's `managed` flag is `True` during tests but `False` in production (migration 0007). Does this mean the test database has a `ddf_price_activation_priceactivationpage` table that doesn't exist in production?
- How frequently is the `ddf_process_rpm_price_activation_file` command run? Is it cron-scheduled, triggered by file system events, or run manually?
- Are there other DDF-specific modules beyond `ddf_price_activation`? The `DubaiDutyFreeStoreType` is documented in other modules (emails, orders, dashboard). How do these coordinate?
