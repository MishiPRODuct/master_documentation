# MishiPay Core Module (`mishipay_core`) & Shared Libraries (`lib/`)

> Last updated by: Task T28 — Lib Module Documentation

## Overview

`mishipay_core` is the foundational Django app of the MishiPay platform. It provides shared models, base classes, utility functions, permission classes, authentication backends, retailer configuration wrappers, notification services, and API endpoints used across all other platform modules. With 211 files, it acts as the connective tissue between the platform's domain-specific apps.

The module is registered as `mishipay_core` in `INSTALLED_APPS` and resides at `mainServer/mishipay_core/`.

## Key Concepts

- **Retailer System Configuration**: A wrapper system (`RetailerSystemConfiguration` / `RetailerSystemConfigurationV2`) that loads per-retailer configuration modules based on store type and demo/live mode. Provides access to product, order, firebase, customer, legal_requirements, and email settings.
- **Entity Properties**: A generic key-value system (`EntityProperty` model) for attaching arbitrary JSON configuration to entities (items, payments, orders, baskets, customers, etc.) without schema changes.
- **Entity Action Constraints**: A rule engine (`ActionConstraint` / `EntityActionConstraints` models) that evaluates dynamic expressions against variables to enforce business constraints on actions like item scanning, basket updates, or payments.
- **Interfaces**: Abstract service layers (`BaseInterface`, `CustomerInterface`, `ItemInterface`, `OrderInterface`, `PaymentInterface`) providing domain-specific operations scoped to a store and customer context.
- **Client Configuration**: A static asset serving system that caches and serves store themes, hardcoded items, and force-update information to frontend clients.

## Architecture

### Directory Structure

```
mishipay_core/
+-- models/                        # Data models (legacy + v2)
+-- interfaces/                    # Domain service interfaces
+-- utilities/                     # Focused utility modules
+-- logging/                       # JSON logging, middleware, filters
+-- management/commands/           # Django management commands (7)
+-- core_enhancements/             # Serializer enhancements
+-- retailer_entity_formatters/    # Retailer-specific item/promo formatters
+-- static_assets_config/          # Static asset URL/path configuration
+-- templates/email/               # 88 HTML email templates
+-- templates/system/              # System templates (timezone check)
+-- templatetags/                  # Custom Django template filters
+-- tests/                         # Test suite (16 test files)
+-- firebase_common/               # Firebase admin SDK credentials
+-- common_functions.py            # Mega utility file (4,200+ lines, 97 functions)
+-- wrappers.py                    # RetailerSystemConfiguration (V1, dict-based)
+-- wrappers_v2.py                 # RetailerSystemConfigurationV2 (V2, model-based)
+-- views.py                       # API views (12 view classes/functions)
+-- serializer.py                  # DRF serializers
+-- urls.py                        # URL routing (12 endpoints)
+-- permissions.py                 # DRF permission classes (7)
+-- authentication.py              # TransactionTokenAuthentication
+-- notification.py                # Firebase push notifications
+-- client_config.py               # Client configuration endpoints
+-- system_api.py                  # System/debug API endpoints
+-- admin.py                       # Django admin configuration
+-- choices.py                     # Choice enumerations
+-- constants.py                   # Module constants
+-- app_settings.py                # App-level settings and message strings
+-- messages.py                    # i18n response messages
+-- exception.py                   # Custom exception classes
+-- backends.py                    # Health check backend
+-- analytics.py                   # Privacy-aware analytics filtering
+-- custom_classes.py              # DecimalRetry for HTTP retries
+-- custom_fields.py               # PrettyJSONField for admin forms
+-- custom_widgets.py              # (not read — likely admin widgets)
+-- generic_interfaces.py          # Shopping bag item retrieval
+-- create_template_context.py     # Retailer-specific email template contexts
+-- generate_order_file_for_pos.py # POS order file generation
+-- generate_orders_excel.py       # Excel order export
+-- airport_codes.py               # Airport code reference data
+-- customer_duty.py               # Duty-free customer logic
+-- user_email.py                  # User email operations
+-- support_ticket.py              # Support ticket logic
```

## Data Models

### Legacy Models (`models/models.py`)

| Model | Fields | Purpose |
|-------|--------|---------|
| **ActionConstraint** | `constraint_name`, `constraint_expression` (max 1024), `constraint_expression_value` (max 256), `created`, `updated` | Defines a single evaluatable constraint expression. Uses `eval()` to check constraint against variables. |
| **EntityActionConstraints** | `entity_action` (choices from `EntityActionChoice`), `constraint_set_name`, `constraints` (M2M to ActionConstraint), `action`, `action_message`, `response_status`, `created`, `updated` | Groups multiple constraints for a specific entity action. Evaluates combined expression using `eval()` with `or` logic. |
| **Theme** | `store` (FK to `dos.Store`), `platform` (choices: web/ios/android/common) | Platform-specific theme configuration per store. Unique together on `(store, platform)`. |
| **ThemeProperty** | `theme` (FK to Theme), `label` (max 128), `value` (max 1024) | Key-value pairs for theme customization. Unique together on `(theme, label)`. |
| **StaticAssets** | `asset_name`, `asset_link` (URL), `version`, `dependency` (platform_dependent/independent), `platform`, `active`, `last_ran`, `created`, `updated` | Tracks versioned static assets served to frontend clients. |
| **SDKLicenseVerification** | `bundle_id`, `license_key` (text), `env` (sandbox/live), `web_url`, `scandit_license_key` (text), `active`, `expiry`, `activated_on`, `created`, `updated` | SDK license verification for mobile apps. Stores Scandit barcode scanner license keys. |
| **DutyChargeFactor** | `variable_factor_type` (COUNTRY), `variable_factor_value`, `variable_factor_code`, `created`, `updated` | Duty-free charge configuration by country. |
| **DutyChargeProperties** | `duty_charge_factor` (FK), `property_type` (Airport/City), `property_value`, `property_code` (IATA code), `property_region`, `created`, `updated` | Airport/city-level duty charge properties. Inline-edited via `DutyChargeFactorAdmin`. |
| **EntityProperty** | `entity` (choices from `EntityPropertyChoice` — 20 types), `name`, `variable_factor` (nullable), `priority` (nullable), `property_value` (JSONField), `created`, `updated` | Generic JSON configuration store for any entity type. Unique together on `(entity, name, variable_factor)`. Supports items, payments, orders, baskets, customers, promotions, loyalty, RFID, privacy policy, and more. |

### V2 Models (`models/models_v2.py`)

| Model | Fields | Purpose |
|-------|--------|---------|
| **BaseModel** (abstract) | `id` (UUID PK, auto-generated) | Base model providing UUID primary keys for all V2 models. |
| **BaseAuditModel** (abstract) | Inherits `BaseModel` + `created` (auto_now_add), `updated` (auto_now) | Adds audit timestamps to UUID-based models. |
| **Retailer** | `name` (text) | A retailer entity that owns stores. Referenced by `dos.Store` via reverse relation `stores`. |
| **StoreDenomination** | `store_type` (max 31), `region` (default "UK"), `currency_sign` (nullable text), `denominations` (ArrayField of CharField) | Cash denomination configuration per store type and region. Unique together on `(store_type, region)`. |

### Relationships

```
Retailer  --< Store (via dos.Store.retailer FK)
Theme     --< ThemeProperty
Theme     --> Store (FK)
DutyChargeFactor --< DutyChargeProperties
EntityActionConstraints --M2M-- ActionConstraint
```

### Security Concern: `eval()` in Constraints

Both `ActionConstraint.check_constraint()` (`models.py:38`) and `EntityActionConstraints.check_constraints()` (`models.py:78`) use Python's `eval()` to evaluate constraint expressions stored in the database. This is a **code injection risk** if constraint values can be set by untrusted input. The expressions are evaluated against a provided variables dictionary, but `eval()` can execute arbitrary Python code.

## API Endpoints

### Core URLs (`urls.py`)

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `v1/theme/` | `ThemeAPIView` | None | Retrieve theme properties for a store/platform |
| GET | `v1/get-additional-store-details/` | `AdditionalStoreDetails` | Authenticated | Get mixpanel token, payment methods, 3DS status for a store |
| GET | `v1/all-store-details/` | `AllStoreDetails` | None | Get all store names, addresses, and theme configs |
| GET | `v1/admin/all-store-details/` | `AdminStoreDetails` | Authenticated + AdminUser | Get stores matching a bundle ID (lat/lng included) |
| POST | `v1/customer/create-support-ticket/` | `SupportTicketView` | Authenticated | Create a support ticket via email |
| GET | `v1/sdk-verification/` | `SDKLisenceVerificationAPI` | None | Verify SDK bundle ID + license key, return Scandit key |
| GET | `v1/stores-by-country/<country>/` | `StoresByCountry` | None | List stores by country with chain grouping (V1) |
| GET | `v2/stores-by-country/<country>/` | `StoresByCountryV2` | None | List stores by country with theme properties and entity properties (V2) |
| POST | `v1/send-slack-msg/` | `SlackClient` | None | Send a Slack message (no auth — security concern) |
| GET | `keycloak-migrate/` | `KeycloakMigrationApiView` | None | Return user data for Keycloak migration |
| POST | `keycloak-migrate/` | `KeycloakMigrationApiView` | None | Validate username/password for Keycloak migration |
| GET | `v1/store-countries/` | `StoreCountries` | None | List distinct countries with active stores |
| GET | `v1/store-denominations/` | `GetStoreDenominations` | None | Get cash denominations for the request's store |

### Client Configuration URLs (registered in `mainServer/urls.py`, views in `client_config.py`)

| Method | Path | View Function | Description |
|--------|------|---------------|-------------|
| GET | `config/v<major>.<minor>/theme/<store_id>/` | `theme` | Per-store theme config |
| GET | `config/v<major>.<minor>/hardcoded/<store_id>/` | `harcoded` | Hardcoded items per store |
| GET | `config/master.json` | `master_conf` | Master config with asset links per platform |
| GET | `config/all_stores.<platform>.v<major>.<minor>.json` | `all_stores` | All store details (cached 24h) |
| GET | `config/app_force_update.v<major>.<minor>.json` | `force_update` | App force update config (cached 24h) |
| GET | `config/hardcoded_items.v<major>.<minor>.json` | `hardcoded_items` | Global hardcoded items (cached 24h) |

### System API (`system_api.py`)

| Method | Path | View Function | Description |
|--------|------|---------------|-------------|
| GET | `system/stores/` | `stores` | List all stores with basic info (id, name, type, lat/lng, demo, psp) |
| GET | `system/timezone-check/` | `timezone_check` | Debug endpoint to verify server timezone handling |

### Notable View Patterns

- **Geo-detection**: `StoresByCountry` and `StoresByCountryV2` use the Cloudflare `HTTP_CF_IPCOUNTRY` header to auto-detect the user's country when the `country` parameter is `"geo"`. Falls back to `"GB"`.
- **Partner filtering**: Decathlon stores are isolated — passing `partner=decathlon` shows only Decathlon stores; without it, Decathlon stores are excluded.
- **Production filtering**: In prod (`settings.ENV_TYPE == "prod"`), only active, non-demo stores are returned.
- **V1 vs V2 stores**: V2 (`StoresByCountryV2`) adds bundle-ID-based filtering, entity properties, and web theme properties (map banners, logos, pins) via `select_related`/`prefetch_related` for performance.

## Serializers

| Serializer | Type | Purpose |
|------------|------|---------|
| `ThemeGETParamQueryParamSerializer` | Serializer | Validates `store_id` (UUID) and `platform` for theme API |
| `StoreIDSerializer` | Serializer | Validates `store_id` (UUID) |
| `SendThankYouEmailSerializer` | Serializer | Validates email params (app_type, dates, consent flags) — currently unused (view commented out) |
| `MasterStaticAssetFileSerializer` | ModelSerializer | Serializes `StaticAssets` (dependency, platform, version, link, name) |
| `AdminStoreSerializer` | ModelSerializer | Serializes `Store` for admin (name aliased as `retailer` for backward compat) |
| `SDKVerificationOfInputSerializer` | Serializer | Validates bundle_id and license_key inputs |
| `SDKVerificationSerializer` | ModelSerializer | Returns `web_url` and `scandit_license_key` from `SDKLicenseVerification` |
| `StoreDenominationsSerializer` | ModelSerializer | Returns `currency_sign` and `denominations` array |

## Business Logic

### Retailer System Configuration (`wrappers.py`, `wrappers_v2.py`)

The `RetailerSystemConfiguration` (V1) and `RetailerSystemConfigurationV2` (V2) classes load per-retailer configuration at runtime. They are the primary mechanism for retailer-specific behavior across the platform.

**How it works:**
1. Constructor receives a store (dict for V1, model instance for V2)
2. Maps `store_type.lower()` to a configuration package under `mainServer/retailer_configurations/`
3. Loads either `demo_configuration` or `live_configuration` module based on `store.demo`
4. Exposes 6 setting categories as properties: `product`, `order`, `firebase`, `customer`, `legal_requirements`, `email`
5. Each property merges `common` settings with region-specific overrides

**Supported retailers (44):** saturn, decathlon, nedap, mishipay, picwic, dufry, bestseller, relay, avery_dennison, compass, mishipay_promotion, eroski, spar, londis, fast_eddys (connexus), muji, flying_tiger, univision, charlotte, lovbjerg, breeze_thru, desigual, standard, expresslane, decathlon_il, nude_foods, goloso, badiani, paradies, stellar, reitan, standard_cafe, rfidstandard, retail_reload, hudson, sto, virgin, loves, emmasleep, viva, dimona, bloomingwear, emmasgarden, emetro, hmshost, pantree, eventnetwork, plus `default_configuration` as fallback.

**V1 vs V2 differences:**
- V1 (`wrappers.py`): Takes a `store_dict` (dictionary), used throughout legacy code
- V2 (`wrappers_v2.py`): Takes a `Store` model instance directly, used in newer code
- Both have identical logic; V2 avoids the `model_to_dict()` conversion step

### Common Functions (`common_functions.py`)

This is the largest file in the module at **4,200+ lines** with **97 top-level functions** and **6 classes**. It serves as a catch-all utility module. Key function categories:

#### Response Helpers
- `get_generic_response_structure()` — Returns the standard `{data: {}, status: 200, message: "", error: ""}` response template used across all APIs
- `get_message_as_per_language(msg, lang)` — Returns localized message text
- `get_error_as_per_language(error_code, lang)` — Returns localized error text

#### Email & Receipt Generation
- `send_receipt(order, ...)` — Sends order receipt emails using retailer-specific templates
- `send_email_store(order, ...)` — Sends store notification emails for new orders
- `generate_context_for_receipt(order, ...)` — **~1,550 lines** — Builds the complete template context for order receipt emails. Handles promotions, taxes, loyalty, refunds, donations, click-and-collect, and retailer-specific formatting
- `generate_context_for_receipt_excluding_refund_items_qty(...)` — **~420 lines** — Variant that excludes refunded item quantities
- `send_referral_receipt(...)` — Sends referral program emails

#### Store & Customer Settings
- `get_store_settings(store_dict)` — Returns `RetailerSystemConfiguration` (V1) for a store
- `get_store_settings_v2(store)` — Returns `RetailerSystemConfigurationV2` for a store
- `get_store_properties(store_dict)` — Gets entity properties for a store
- `get_saved_cards(customer, store)` — Retrieves saved payment cards
- `get_customer_default_payment_method(customer, store)` — Gets default payment method
- `get_loyalty_number(customer, store_type)` — Gets customer loyalty card number

#### Data Manipulation
- `remove_items(main_list, sub_list)` — Removes items by barcode with quantity management
- `merge_items(list1, list2)` — Merges item lists by barcode
- `extract_barcode(barcode_str)` — Parses barcode strings
- `is_valid_uuid(val)` — UUID validation
- `get_rounded_value(value, decimal_places)` — Decimal rounding

#### Date/Time
- `julian_date_to_datetime(jd, fmt)` — Converts Julian dates to Python datetime (used for POS integrations)

#### External Services
- `send_slack_message(text)` — Posts to Slack via webhook
- `get_requests_session_client(max_retries, backoff_factor)` — Creates a `requests.Session` with retry configuration using `DecimalRetry`

#### Notable Functions
- `check_for_discrepancy(func)` — Decorator that checks for order discrepancies
- `get_reduce_item_price_and_barcode(barcode)` — Parses reduced-price item barcodes (type 1-5) with amount/percentage decoding
- `is_kafka_flow_enabled(store)` — Checks if a store uses Kafka-based order processing
- `get_kiosk_pos_tax_table(order)` / `new_get_kiosk_pos_tax_table(order)` — Generates tax tables for POS kiosk receipts

### Interfaces

The `interfaces/` package provides domain-scoped service layers, all inheriting from `BaseInterface`:

**`BaseInterface`** (`interfaces/base_interface.py`):
- Constructor takes `store_id` and optional `cus_id`, `store`, `customer`
- Lazily loads `Store` and `Customer` objects if not provided
- Scopes all operations to a specific store/customer context

**`CustomerInterface`** (`interfaces/customer_interface.py`):
- `openid_decathlon_update_profile_with_sso()` — Updates Decathlon SSO profile
- `sync_profile_with_sso()` — Syncs profile with SSO provider (dynamic dispatch by provider name)
- `get_oauth_account()` — Retrieves OAuth account for customer
- `get_past_purchase_records()` — Fetches purchase history
- `get_or_create_retailer_specific_customer_profile()` — Gets/creates retailer-specific profile
- `is_guest_customer()` — Checks guest user status
- `update_past_purchase_records(records)` — Updates purchase history with incremental values

**`ItemInterface`** (`interfaces/item_interface.py`):
- `get_items(filter_dict)` — Static: queries `ItemInventory`
- `get_store_items(filter_dict)` — Scoped to current store
- `get_red_user_discount_code(tier)` — Gets RED loyalty discount code
- `get_relay_coffee_incentive_constant()` / `get_londis_coffee_qty_purchased_constant()` — Retailer-specific constants

**`OrderInterface`** (`interfaces/order_interface.py`):
- `initial_o_id` — Property returning default order ID value
- `get_order(order_id)` — Static: fetches order by ID
- `get_order_for_store_and_customer(order_id)` — Scoped to store/customer

**`PaymentInterface`** (`interfaces/payment_interface.py`):
- `can_by_pass_payment(basket, store)` — Checks if payment can be bypassed (e.g., zero-value orders)
- `get_payment_variant()` — Gets payment gateway configuration for store
- `check_payment_method_applicable(method)` — Validates payment method availability
- `get_paypalpos_details()` — Gets PayPal POS details via BrainTree provider
- `get_payment_status()` — Returns payment status choices

### Notification System (`notification.py`)

Firebase Cloud Messaging (FCM) integration for push notifications:

- `publish_firebase_dashboard_notification(title, message, topic, data)` — Sends topic-based FCM notifications to dashboard apps (Android/iOS), plus webpush notifications to individual devices for click-and-collect orders
- `publish_firebase_mishipay_notification(store_id, user, trigger)` — Sends triggered notifications to individual customer devices (GCM + APNS)
- `validate_fcm_token(fcm_token)` — Validates an FCM token by sending a test notification; removes unregistered devices
- `get_firebase_bearer_token()` — Obtains Firebase bearer token from service account credentials
- Uses two Firebase projects: `mishipay-dashboard` (prod) and `mishipay-dashboard-test` (dev/test)
- **Note:** Contains `print()` statements in production code paths (`notification.py:63-66`, `notification.py:163-167`)

## Permissions

Seven DRF permission classes defined in `permissions.py`:

| Permission Class | Logic | Used By |
|-----------------|-------|---------|
| `ActiveStorePermission` | Checks `store.active == True` | Store-scoped endpoints |
| `ActiveStaffShiftPermission` | Requires active `KioskStaffShift` for `android_cashier_kiosk` platform; sets `request.shift` | Cashier kiosk endpoints |
| `PriceOverridePermission` | Extends `ActiveStaffShiftPermission`; for `android_kiosk` delegates to `DashboardPermission` | Price override operations |
| `IsAdminUser` | Checks user belongs to `AdminUser` Django group | Admin-only endpoints |
| `InventoryUpdatePermission` | Checks user belongs to `InventoryUpdate` Django group | Inventory update endpoints |
| `UserStorePermission` | Checks user has access to `retailer_store_id` via dashboard user stores | Store-restricted operations |

## Authentication

### TransactionTokenAuthentication (`authentication.py`)

Custom DRF authentication class for one-time transaction tokens:

1. Reads `transaction-token` from request header
2. Validates the associated order exists
3. Verifies token exists in `TransactionToken` model and is not expired
4. Returns the order's customer user + creates/gets a DRF auth token
5. Marks the transaction token as used (one-time use)
6. Sets `request.transaction_token_authenticated = True` flag

### Health Check Backend (`backends.py`)

`AppReachableBackend` extends `django-health-check`'s `BaseHealthCheckBackend`:
- Sends HTTP OPTIONS request to `http://localhost:8080/admin/` with the configured hostname
- Reports app as unreachable if request fails
- Timeout: 5 seconds

## Custom Exception Handling

Defined in `exception.py`:

- **`CustomAPIException`** — Generic API exception with customizable `detail` and `status_code` (default 400)
- **`NoActiveStaffShiftException`** — Raised when a cashier kiosk request has no active staff shift; returns 403 with structured response
- **`custom_exception_handler`** — Registered as DRF exception handler; wraps `NoActiveStaffShiftException` responses in the standard `{data, status, message, error}` format

## Logging Infrastructure

### JSON Logger (`logging/json_logger.py`)
- Configures `pythonjsonlogger.JsonFormatter` for structured JSON log output
- Defines `base_log_structure` with fields: `retailer_name`, `store_id`, `store_type`, `latitude`, `longitude`, `user_id`, `platform`, `bundle_id`

### Logging Middleware (`logging/middleware.py`)
- `LoggingUserIdMiddleware` — Tracks request timing and DB query counts
- Sets thread-local `user_id` for log correlation
- Prints DB call counts and response times (debug `print()` statements in production)

### Log Filter (`logging/filters.py`)
- `UserIdFilter` — Adds `user_id` from thread-local storage to log records

### Sentry Tagging (`logging/set_custom_tag.py`)
- `create_error_log_with_custom_tag()` / `create_debug_log_with_custom_tag()` — Logs with Sentry scope tags (identifier, name, platform, user_id)
- `create_custom_tags()` — Builds tag dictionary from customer/store data

## Management Commands

| Command | File | Description |
|---------|------|-------------|
| `consumers_health_check` | `consumers_health_check.py` | Runs `SELECT 1` health check for Kafka consumers; exits 0/1 |
| `rebuild_client_config` | `rebuild_client_config.py` | Clears client configuration caches (force_update, hardcoded_items, all_stores) |
| `push_notifications` | `push_notifications.py` | Sends push notifications to users from a CSV file of order IDs. Args: `--file_name`, `--store_type`, `--order_store_status`, `--title`, `--body` |
| `ftc_market_consent` | `ftc_market_consent.py` | Flying Tiger marketing consent management |
| `send_complete_purchase_email` | `send_complete_purchase_email.py` | Sends "complete your purchase" emails |
| `send_wishlist_email` | `send_wishlist_email.py` | Sends wishlist reminder emails |
| `thank_you_email_command` | `thank_you_email_command.py` | Sends first-purchase thank-you emails |

## Analytics & Privacy (`analytics.py`)

`filter_params_as_per_privacy_policy(params)`:
- Checks whether a customer has opted in to data collection for their store's privacy policy
- If opted out, removes restricted parameters from analytics events
- Uses `PrivacyPolicyInformation` model with version tracking
- Falls back to filtering (restrictive) if no consent record exists

## Choice Enumerations

### Entity Choices (`__init__.py`)

**`EntityActionChoice`** — Actions that can have constraints:
`ITEM_SCAN`, `ADD_UPDATE_BASKET`, `ADD_UPDATE_WISHLIST`, `CREATE_ORDER`, `PAY`, `SAVE_CARD`, `GEO_FENCING`

**`EntityPropertyChoice`** — Entity types for `EntityProperty`:
`ITEM`, `PAYMENT`, `SAVED_CARD`, `ORDER`, `LOGIN`, `REGISTER`, `PROMOTION`, `LOYALTY`, `BASKET`, `BASKET_ITEM`, `CUSTOMER`, `WISHLIST`, `WISHLIST_ITEM`, `BASKET_AUDIT_LOG`, `RFID`, `APPLICABLE_STORE_FEATURES`, `VAT`, `PRIVACY_POLICY`, `SIDE_NAV_OPTIONS`, `BASKET_OPTIONS`, `CUSTOM_STORE_CHAIN`

### Platform Choices (`choices.py`)

**Theme platforms:** `web`, `ios`, `android`, `common`

**App request platforms (9):**
`Web`, `Web_Android`, `Web_iOS`, `iOS`, `Android`, `Android_Kiosk`, `Android_PS20`, `Android_Cashier_Kiosk`, `Android_MPOS`

**Device type groups:**
- `ANDROID_PLATFORMS_LIST`: Android, Android_Kiosk, Android_PS20, Android_Cashier_Kiosk, Android_MPOS
- `KIOSK_DEVICE_TYPES`: android_cashier_kiosk, android_kiosk, android_mpos
- `SCAN_AND_GO_DEVICE_TYPES`: web, web_android, web_ios, android, ios

**`POSLOG_SEQUENCE_TYPE_CHOICES`:** `store`, `device`, `device_type` — Controls POS log sequence numbering granularity.

**`CommonOffersConfigurationChoices`:** `StoreID`, `StoreType`, `RegionType`, `SubRegionType` — Scope levels for promotions/loyalty configuration.

### Other Choices
- `SDKEnvironmentChoices`: `sandbox`, `live`
- `DutyChargeFactorChoices`: `country`
- `PropertyChoices`: `airport`, `city`
- `FRONTEND_URL_MAP`: Maps backend domains to frontend payment domains (dev/test/staging/prod)

## Configuration

### Constants (`constants.py`)
- `DEFAULT_COUNTRY_PARAM = "geo"` — Triggers Cloudflare geo-detection
- `CLOUDFLARE_COUNTRY_KEY = "HTTP_CF_IPCOUNTRY"` — Request header for country
- `TYPES_OF_MARKETING_CONSENTS = {"retailer", "mishipay"}` — Consent types

### App Settings (`app_settings.py`)
- `SUPPORTED_PLATFORMS = ['ios', 'android', 'web']`
- `EMAIL_STORE_LOGO_URL` — Logo URLs for email templates
- `ERROR_CODES` — Multi-language error messages (English, Hebrew, German, Dutch, Spanish, Basque)
- `CREATE_TEMPLATE_CONTEXT_FUNCTION_MAP` — Maps store types to custom email context functions
- `RED_LOGIN_SALUTATION` — Maps salutations to genders for RED loyalty
- `DUFRY_CATEGORIES_TO_HIDE_OFFER_LABEL` — Categories where offer labels are hidden
- `REDUCE_ITEM_*` — Constants for reduced-price barcode parsing (types 1-5, amount/percentage)

### Messages (`messages.py`)
- `MESSAGES` dictionary with i18n-ready response messages keyed as `SERVICE.ERR|OK.TAG`
- Covers: store errors, basket errors, manual verification errors, generic success, and retailer-specific notice messages (Eroski, MUJI, Flying Tiger)
- `render(error_code, language)` / `add_error(response, error_code)` / `generic_success(response, ...)` — Response formatting helpers

### Static Assets Configuration (`static_assets_config/config.py`)
- `STATIC_ASSETS_ENTITY_URL_MAP` — Defines asset URLs and file paths for: `master_file`, `app_force_update`, `all_store_details`, `hardcoded_items`

## Admin Configuration

Registered in `admin.py`:

| Model | Admin Class | Features |
|-------|-------------|----------|
| Theme | `ThemeAdmin` | List: id, store, platform. Filter: platform. Inline: `ThemePropertyInlineAdmin` |
| StaticAssets | `StaticAssetsAdmin` | List: name, link, version, dependency, active, dates. Full search/filter |
| DutyChargeFactor | `DutyChargeFactorAdmin` | Inline: `DutyChargePropertiesInlineAdmin` |
| EntityProperty | `EntityPropertyAdmin` | List: entity, name, variable_factor, property_value. `PrettyJSONWidget` for JSON |
| ActionConstraint | `ActionConstraintAdmin` | List: id, name, dates |
| StoreDenomination | `StoreDenominationsAdmin` | List: store_type, region, currency_sign, denominations, created |
| Retailer | `RetailerAdmin` | List: id, name, store_count (annotated), dates. Search by name |
| EntityActionConstraints | Default | Basic registration |
| SDKLicenseVerification | Default | Basic registration |

Additional: `TokenAdmin.raw_id_fields = ["user"]` — Improves admin performance for DRF token management.

Custom widget: `PrettyJSONWidget` — Formats JSON fields with indentation and auto-sizes the textarea.

## Retailer Entity Formatters

`retailer_entity_formatters/` provides retailer-specific data formatting:

- **Item formatters:** `best_seller.py`, `decathlon.py`, `picwic.py` — Format item data for specific retailers
- **Promotion update formatters:** `import_best_seller_promotion.py`, `update_best_seller_flat_discount.py` — Import/update promotions for BestSeller retailer

## Template Tags

`templatetags/custom_filters.py` provides 3 Django template filters:
- `remove_underscore_and_title` — Converts `snake_case` to `Title Case`
- `number_range` — Returns `range(value)` for template iteration
- `replace_fullstop_with_comma` — Replaces `.` with `,` for European number formatting (with zero-padding)

## Core Enhancements

`core_enhancements/serializer_enhancements.py`:
- `SerializerErrorMessageComplier` — Mixin that adds multi-language error message compilation to DRF serializers using `ERROR_CODES` from `app_settings.py`

## Custom Form Fields

`custom_fields.py`:
- `PrettyJSONField` — Django form field that validates, pretty-prints, and edits JSON data with configurable indentation
- `InvalidJSONInput` / `JSONString` — Marker classes for JSON validation states

## Email Templates

88 HTML email templates in `templates/email/` covering:
- **Receipt emails**: Per-retailer and per-locale (e.g., `muji_email_en.html`, `eroski_email_en.html`, `decathlon_de_email.html`)
- **Click-and-collect**: Order confirmation, ready-to-collect, collected, cancelled (standard + Badiani/Goloso variants)
- **Store notifications**: Store-side CC order templates
- **Marketing**: Complete purchase, wishlist, referral, promotion, bounce-back (Flying Tiger)
- **First-purchase thank-you**: Localized for 11 languages (en, es, fr, it, da, no, se, fi, ch, cz)
- **System**: Support ticket, MMI POSLOG sent notification

## Test Coverage

16 test files in `tests/` covering:

| Test File | Scope |
|-----------|-------|
| `test_authentication.py` | Transaction token auth |
| `test_client_config.py` | Client configuration endpoints |
| `test_closest_store_v1.py` | Store proximity (V1) |
| `test_closest_store_v3.py` | Store proximity (V3) |
| `test_common_functions.py` | Common utility functions |
| `test_default_payment_method.py` | Default payment method selection |
| `test_i18n_timezone.py` | i18n and timezone handling |
| `test_is_user_valid.py` | User validation logic |
| `test_ms_auth.py` | Microservice authentication |
| `test_ms_pay_default_payment_method.py` | MS Pay default payment method |
| `test_stores_by_country.py` | Stores by country API |
| `test_system_api_stores.py` | System API stores endpoint |

Supporting files: `base_tester.py` (base test class), `factories.py` (factory_boy factories), `default_input_data.json` (fixtures), `setup_test_db_interface.py` (DB setup), `utils.py` (test utilities).

## Dependencies

### What core depends on
- `dos.models` — `Store`, `Order`, `Customer`, `ThirdPartyLogin`, `TokenData`
- `mishipay_retail_payments` — `PaymentVariant`, `PaymentStatus`, `PSPType`, `CardBrandValues`
- `mishipay_retail_orders` — `Order`, `ReceiptStatus`
- `mishipay_retail_customers` — `TransactionToken`, `Customer`, `PastPurchase`, `RetailerSpecificCustomerProfile`
- `mishipay_items` — `ItemInventory`, `Basket`, `BasketItem`, `ItemCategory`
- `mishipay_config` — `GlobalConfig`, `get_client_config_key`
- `mishipay_dashboard` — `RefundOrderNew`, `DashboardPermission`, `User`
- `mishipay_cashier_kiosk` — `KioskStaffShift`
- `mishipay_retail_analytics` — `IndirectStoreStatusChoices`
- `third_party_integrations` — `OAuthApplication`, `DecathlonOAuthProfileUpdateView`
- `dashboard` — `DashboardPermission`, `User`
- `analytics` — `StoreNotification`
- `microservices_client` — `client`
- `config.loader` — `EMAIL_SETTINGS`

### What depends on core
- Virtually **every other app** in the platform depends on `mishipay_core` for:
  - `get_generic_response_structure()` and response helpers
  - `RetailerSystemConfiguration` / `RetailerSystemConfigurationV2`
  - Permission classes (`ActiveStorePermission`, `ActiveStaffShiftPermission`, etc.)
  - `TransactionTokenAuthentication`
  - `EntityProperty` and entity choice classes
  - Common utility functions
  - Email template rendering
  - Firebase notification publishing

## Notable Patterns

1. **Mega-file anti-pattern**: `common_functions.py` (4,200+ lines, 97 functions) mixes email generation, receipt rendering, store settings, payment logic, date conversion, Slack messaging, and more. The `generate_context_for_receipt` function alone is ~1,550 lines.

2. **Dual configuration system**: `RetailerSystemConfiguration` (V1, dict-based) and `RetailerSystemConfigurationV2` (model-based) coexist with identical logic. V2 was introduced to avoid the `model_to_dict()` conversion overhead.

3. **Dynamic dispatch**: `CustomerInterface.sync_profile_with_sso()` uses `getattr()` to dynamically call `{provider}_sync_profile_with_sso()` methods, following a convention-over-configuration pattern.

4. **`eval()` usage**: Constraint models use `eval()` for expression evaluation — a deliberate flexibility/security tradeoff. See Security Concern above.

5. **Debug prints in production**: Multiple files contain `print()` statements (`notification.py:63-66`, `logging/middleware.py:24,31`, `wrappers.py:101-107`) that should be replaced with proper logging.

6. **No-auth endpoints**: `SlackClient` (POST to send Slack messages) and `KeycloakMigrationApiView` (GET returns user data, POST validates credentials) have no authentication — potential security concerns.

7. **Hardcoded Firebase credentials**: `firebase_common/` contains a Firebase admin SDK JSON credentials file committed to the repository.

8. **Cache-aside pattern**: Client configuration uses Django's cache framework with 24-hour TTL, with `rebuild_client_config` management command to manually invalidate.

## Open Questions

- How are the `retailer_entity_formatters` loaded and invoked? No explicit import chain was found in the core module itself.
- Is the `ActionConstraint` / `EntityActionConstraints` constraint engine still actively used, or is it legacy? The `eval()` approach suggests early development patterns.
- Are there plans to split `common_functions.py` into focused modules? Its current size makes it difficult to maintain and test.
- The `SlackClient` endpoint has no authentication. Is it protected by network-level access controls (e.g., internal-only routing)?
- The `KeycloakMigrationApiView` returns user data and validates passwords without authentication. Is this secured by being temporary/removed in production, or protected at the infrastructure level?
- The Firebase admin SDK credentials JSON file is committed in `firebase_common/`. Is this a non-sensitive service account, or should it be in a secrets manager?
- `LoggingUserIdMiddleware` tracks DB query counts but uses `print()` instead of the logging framework. Is this intentionally enabled in production?

---

# Shared Libraries (`lib/`)

> Last updated by: Task T28 — Lib Module Documentation

## Overview

The `lib/` directory at `mainServer/lib/` contains 5 self-contained packages providing shared functionality across the MishiPay platform: performance profiling middleware, a SQL query IDE, a payment gateway client, an airline boarding pass decoder, and an HTTP request logger. These are a mix of vendored open-source libraries (explorer), in-house client libraries (pyoptile, python_bcbp), and development tooling (backend_profiling, logging).

Total: ~98 files (71 Python files, 13 HTML templates, 5 static assets, 8 migrations, test files).

## Architecture

```
lib/
├── backend_profiling/        # Django middleware for performance profiling
│   ├── middleware.py          # ProfileMiddleware — request lifecycle hooks
│   ├── models.py             # Dynamic module loader (MODULES global)
│   ├── settings.py           # Configuration from Django settings
│   ├── logger.py             # GenericLogger — styled console output
│   ├── sql.py                # Empty placeholder
│   └── modules/
│       ├── __init__.py        # ProfileModule base class
│       ├── sql.py             # SQLRealTimeModule, SQLSummaryModule
│       └── profile.py         # MemoryUseModule, LineProfilerModule, backend_code_profile
│   └── utils/
│       ├── time.py            # ms_from_timedelta()
│       ├── stats.py           # StatCollection — function call profiling
│       ├── http.py            # SlimWSGIRequestHandler
│       └── stack.py           # Stack trace utilities (Python 2 era)
├── explorer/                  # Django SQL Explorer v1.1.2 (vendored)
│   ├── models.py              # Query, QueryLog, QueryResult models
│   ├── views.py               # 10+ view classes (CRUD, play, export, schema)
│   ├── urls.py                # 12 URL patterns
│   ├── app_settings.py        # Configuration with blacklist/whitelist
│   ├── utils.py               # SQL param substitution, blacklist checking
│   ├── permissions.py         # View/change permission handlers
│   ├── exporters.py           # CSV, JSON, Excel export classes
│   ├── schema.py              # DB schema introspection
│   ├── forms.py, admin.py     # Django forms and admin
│   ├── connections.py         # DB connection management
│   ├── tasks.py               # Celery async tasks
│   ├── templates/explorer/    # 13 HTML templates
│   ├── static/explorer/       # CSS, JS assets
│   ├── migrations/            # 8 migration files (2015-2019)
│   └── tests/                 # Comprehensive test suite
├── pyoptile/                  # Optile (Payoneer) payment gateway client
│   ├── __init__.py            # Module config (api_key, api_base, etc.)
│   ├── api_requestor.py       # APIRequestor — HTTP request handling
│   ├── optile_object.py       # OptileObject — base resource class
│   ├── error.py               # 5 exception classes
│   ├── util.py                # Response conversion (incomplete)
│   └── api_resources/
│       ├── charge.py           # Charge — 8 payment operations
│       ├── list.py             # List — 8 payment session operations
│       ├── customer.py         # Customer — account detail retrieval
│       ├── registration.py     # Registration — account register/delete
│       └── abstract/
│           ├── api_resource.py # APIResource base class
│           └── custom_method.py# time_all_class_methods decorator
├── python_bcbp/               # IATA Boarding Pass Barcode Decoder
│   ├── __init__.py            # Exports Decoder class
│   ├── decoder.py             # Decoder — barcode validation & parsing
│   ├── fields.py              # Field definitions (OrderedDict)
│   ├── utils.py               # ParseFields — positional field extraction
│   ├── constants.py           # MINIMUM_LENGTH_OF_BOARDING_PASS_BARCODE = 60
│   └── exceptions.py          # InvalidBarcode exception
├── logging/                   # HTTP request/response JSON logger
│   └── requests.py            # json_log_factory() — requests.Session hook
└── setup.py                   # Package setup for django-sql-explorer
```

---

## 1. Backend Profiling (`backend_profiling/`)

### Purpose

A Django middleware-based performance profiling framework with a plugin architecture. Provides SQL query profiling, memory profiling, and line-by-line CPU profiling for development and debugging.

### Key Components

#### ProfileMiddleware (`middleware.py`)

The central Django middleware class that dispatches profiling events to registered modules.

| Lifecycle Hook | Behavior |
|---------------|----------|
| `__call__` | Sets `request.timestamp`, calls `process_request` then `get_response` then `process_response` |
| `process_request` | Sets `request._profiling_active = True`, calls `process_init`, dispatches to all modules |
| `process_init` | Resets `StatCollection`, dispatches `process_init` to all modules |
| `process_view` | Dispatches view function info to all modules |
| `process_response` | Dispatches to all modules, then calls `process_complete` |
| `process_exception` | Dispatches exception info to all modules |
| `should_process` | Filters out `STATIC_URL`, `MEDIA_URL`, `/favicon.ico`, and `PROFILING_IGNORED_PREFIXES` requests |

#### Module Loader (`models.py`)

Dynamically loads profiling modules at import time via `load_modules()`:

1. Reads `BACKEND_PROFILING_MODULES` from Django settings
2. Parses each entry as `module.path.ClassName`
3. Imports the module, instantiates the class with a `GenericLogger`
4. Appends to global `MODULES` list
5. Contains a `print("NAME", name)` debug statement at `models.py:51`

#### ProfileModule Base Class (`modules/__init__.py`)

Abstract base with no-op lifecycle methods: `process_request`, `process_response`, `process_exception`, `process_view`, `process_init`, `process_complete`.

#### SQL Profiling (`modules/sql.py`)

Based on `django-debug-toolbar`. Two module implementations:

- **`SQLRealTimeModule`**: Monkey-patches `django.db.backends.utils.CursorDebugWrapper` with `DatabaseStatTracker` to intercept and log every SQL query as it executes. Logs formatted SQL (via `sqlparse`), execution duration, and row counts.
- **`SQLSummaryModule`**: At request completion, summarizes all SQL queries with total count, duplicate count, and cumulative duration (using `termcolor` for colored console output).

`DatabaseStatTracker` overrides `execute()` and `executemany()` on the cursor wrapper. It filters queries via `BACKEND_PROFILING_FILTER_SQL`, truncates field lists via `BACKEND_PROFILING_TRUNCATE_SQL`, and respects `BACKEND_PROFILING_SQL_MIN_DURATION` threshold.

#### Memory & CPU Profiling (`modules/profile.py`)

Four profiling tools, all requiring optional dependencies:

- **`MemoryUseModule`**: Requires `memory_profiler`. Wraps view functions with `LineProfiler` for memory profiling. Auto-profiles `get`, `put`, `post` methods of class-based views.
- **`LineProfilerModule`**: Requires `line_profiler`. Line-by-line CPU profiling of view functions. Output in microseconds (`output_unit=1`).
- **`backend_code_profile`** (decorator): Annotate any view function/method to include it in profiling. Supports `follow=[]` parameter to add additional functions.
- **`line_profiler`** (standalone decorator): Profile any function independently (not tied to middleware lifecycle).

All modules degrade gracefully — if dependencies are missing, they emit a warning and become no-ops.

#### Utilities

| File | Function/Class | Purpose |
|------|---------------|---------|
| `utils/time.py` | `ms_from_timedelta(td)` | Convert `timedelta` to float milliseconds |
| `utils/stats.py` | `StatCollection` | Global statistics collector tracking function call counts, hit/miss rates, and timing grouped by key. Singleton instance at `stats`. |
| `utils/stats.py` | `track(func, key, logger)` | Decorator wrapping a function to track calls via `StatCollection.run()` |
| `utils/http.py` | `SlimWSGIRequestHandler` | Extended `WSGIRequestHandler` that hides static/media requests from logs and appends SQL query timing to log messages |
| `utils/stack.py` | `tidy_stacktrace(strace)` | Filters stack traces to remove Django/SocketServer internals. **Note**: imports `SocketServer` (Python 2 module name), so this file is incompatible with Python 3. |

### Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `BACKEND_PROFILING_MODULES` | `()` (empty tuple — all modules commented out) | List of module class paths to load |
| `BACKEND_PROFILING_FILTER_SQL` | `False` | Regex patterns to filter out matching SQL |
| `BACKEND_PROFILING_TRUNCATE_SQL` | `True` | Replace SELECT field lists with `...` |
| `BACKEND_PROFILING_TRUNCATE_AGGREGATES` | `False` | Also truncate aggregate queries |
| `BACKEND_PROFILING_SQL_MIN_DURATION` | `None` | Minimum duration (ms) to log a query |
| `BACKEND_PROFILING_AUTO_PROFILE` | `False` | Auto-profile all view functions |
| `BACKEND_PROFILING_AJAX_CONTENT_LENGTH` | `300` | Max AJAX content length to log |

**Current state**: All profiling modules are commented out in the default `BACKEND_PROFILING_MODULES` tuple (`settings.py:3-11`), meaning no profiling occurs unless explicitly enabled.

---

## 2. Django SQL Explorer (`explorer/`)

### Purpose

A vendored copy of [django-sql-explorer](https://github.com/groveco/django-sql-explorer) v1.1.2 (MIT license, original author Chris Clark). Provides a web-based interface for writing, executing, and exporting SQL queries against the platform's databases. Registered as `explorer` in `INSTALLED_APPS`.

This is the `explorer` app referenced in the platform's `settings.py` `INSTALLED_APPS` list (see [Open Questions from T06](./overview.md)).

### Data Models

| Model | Fields | Purpose |
|-------|--------|---------|
| **Query** | `title` (CharField 255), `sql` (TextField), `description` (TextField, nullable), `created_by_user` (FK to AUTH_USER_MODEL), `created_at` (auto), `last_run_date` (auto), `snapshot` (bool), `connection` (CharField 128, nullable) | Stores named SQL queries. Methods: `execute()`, `execute_with_logging()`, `passes_blacklist()`, `final_sql()`, `available_params()`. |
| **QueryLog** | `sql` (TextField), `query` (FK to Query, nullable), `run_by_user` (FK to AUTH_USER_MODEL), `run_at` (auto), `duration` (FloatField, ms), `connection` (CharField) | Audit log of query executions with duration tracking. |
| **QueryResult** | (not a Django model) | In-memory result object with `data`, `headers`, `duration`. Supports column statistics (Sum, Avg, Min, Max, NUL count). |

**Note**: `Query.Meta.permissions` defines unusual placeholder permissions: `can_drive`, `can_vote`, `can_drink` — these appear to be test/placeholder values from the original library.

### URL Routes (`urls.py`)

12 URL patterns mounted at the `explorer/` prefix (as configured in `mainServer/urls.py`):

| Path | View | Permission | Description |
|------|------|-----------|-------------|
| `<int:query_id>/` | `QueryView` | `view_permission` | View/edit a saved query |
| `<int:query_id>/download` | `DownloadQueryView` | `view_permission` | Download query results |
| `<int:query_id>/stream` | `StreamQueryView` | `view_permission` | Stream results inline |
| `download` | `DownloadFromSqlView` | `view_permission` | Download from raw SQL |
| `<int:query_id>/email_csv` | `EmailCsvQueryView` | `view_permission` | Email CSV results (async) |
| `<int:pk>/delete` | `DeleteQueryView` | `change_permission` | Delete a saved query |
| `new/` | `CreateQueryView` | `change_permission` | Create a new query |
| `play/` | `PlayQueryView` | `change_permission` | SQL playground (ad-hoc queries) |
| `schema/<path:connection>` | `SchemaView` | `change_permission` | Browse database schema |
| `logs/` | `ListQueryLogView` | `view_permission` | View query execution logs |
| `format/` | `format_sql` | (none — POST only) | Format SQL via sqlparse |
| `` (index) | `ListQueryView` | `view_permission_list` | List all saved queries |

### Permission System (`permissions.py`)

Three-tiered access control:

1. **`view_permission`**: User has `explorer.view_permission` Django perm, OR is listed in `EXPLORER_USER_QUERY_VIEWS` for this query, OR provides valid API token (if `EXPLORER_TOKEN_AUTH_ENABLED`)
2. **`view_permission_list`**: User has `explorer.view_permission` perm, OR has any query-specific access
3. **`change_permission`**: User has `explorer.super_permission` Django perm (required for create/edit/delete/playground)

Token auth uses `x-api-token` header or `token` query parameter, compared against `EXPLORER_TOKEN` setting (default: `"CHANGEME"`).

### SQL Safety (`utils.py`)

The `passes_blacklist(sql)` function enforces SQL safety:

- **Blacklist** (default): `ALTER`, `RENAME`, `DROP`, `TRUNCATE`, `INSERT INTO`, `UPDATE`, `REPLACE`, `DELETE`, `CREATE TABLE`, `GRANT`, `OWNER TO`, `CREATED`, `UPDATED`, `DELETED`, `REGEXP_REPLACE`
- **Whitelist**: Any terms in `EXPLORER_SQL_WHITELIST` are removed from SQL before blacklist checking
- **Parameter substitution**: Uses `$$param_name$$` token syntax (via `swap_params()` and `extract_params()`)

**Security concerns**:
- `CREATED`, `UPDATED`, `DELETED` in the blacklist will block legitimate SELECT queries that reference columns named `created`, `updated`, or `deleted` — these are common column names in the MishiPay schema
- Multiple `print()` statements in `views.py` (`views.py:71,76,79,89,384`) output permission and query debug info to stdout

### Export System (`exporters.py`)

Three pluggable exporter classes:

| Exporter | Content-Type | Extension | Notes |
|----------|-------------|-----------|-------|
| `CSVExporter` | `text/csv` | `.csv` | Configurable delimiter (default `,`), supports `tab` |
| `JSONExporter` | `application/json` | `.json` | Uses `DjangoJSONEncoder` for dates/UUIDs |
| `ExcelExporter` | `application/vnd.ms-excel` | `.xlsx` | Requires `xlsxwriter`, handles UUIDs and tz-aware datetimes |

### Schema Browser (`schema.py`)

Introspects database tables and columns using Django's `connection.introspection` API. Supports async schema building via Celery (if `EXPLORER_TASKS_ENABLED`). Schema results are cached in Django's cache framework.

Table filtering:
- **Include prefixes**: `EXPLORER_SCHEMA_INCLUDE_TABLE_PREFIXES` (whitelist)
- **Exclude prefixes**: `EXPLORER_SCHEMA_EXCLUDE_TABLE_PREFIXES` (default: `auth_`, `contenttypes_`, `sessions_`, `admin_`, `django_`)
- Non-superusers with `explorer.limited_permission` see the exclude filter applied

### Configuration Summary

| Setting | Default | Description |
|---------|---------|-------------|
| `EXPLORER_CONNECTIONS` | `{}` | Named DB connections available to explorer |
| `EXPLORER_DEFAULT_CONNECTION` | `None` | Default DB connection alias |
| `EXPLORER_SQL_BLACKLIST` | 14 terms (see above) | SQL keywords that block execution |
| `EXPLORER_SQL_WHITELIST` | `()` | Terms removed before blacklist check |
| `EXPLORER_DEFAULT_ROWS` | `1000` | Max rows returned by default |
| `EXPLORER_TOKEN` | `"CHANGEME"` | API token for token auth |
| `EXPLORER_TOKEN_AUTH_ENABLED` | `False` | Enable token-based API access |
| `EXPLORER_TASKS_ENABLED` | `False` | Enable Celery async tasks |
| `EXPLORER_S3_ACCESS_KEY/SECRET_KEY/BUCKET` | `None` | S3 config for query snapshots |
| `EXPLORER_UNSAFE_RENDERING` | `False` | Allow unsafe HTML in results |

---

## 3. Optile Payment Gateway Client (`pyoptile/`)

### Purpose

A Python client library for the [Optile](https://www.payoneer.com/) (now Payoneer Checkout) payment gateway API. Provides classes for payment sessions, charges, customer management, and account registration. Used by the DOS retailer module's `PayUnity` payment gateway class.

### Architecture

```
OptileObject (dict)             ← Base class with auth, request, sandbox/live handling
  └── APIResource               ← URL building, param validation, error responses
       ├── Charge                ← 8 payment charge operations
       ├── List                  ← 8 payment session/list operations
       ├── Customer              ← Customer account detail retrieval
       └── Registration          ← Account registration/deletion
```

### Configuration (`__init__.py`)

Module-level globals configured before use:

| Variable | Default | Description |
|----------|---------|-------------|
| `api_key` | `None` | API key (not used directly — auth uses username/password) |
| `client_id` | `None` | Client identifier |
| `api_base` | `"https://api.sandbox.oscato.com/api"` | API base URL (sandbox) |
| `api_version` | `None` | API version |
| `verify_ssl_certs` | `True` | SSL verification flag |
| `max_network_retries` | `0` | No retry by default |

### API Requestor (`api_requestor.py`)

`APIRequestor` handles HTTP communication:

- Uses Basic HTTP auth (`api_username`, `api_password`)
- JSON content type
- `interpret_response()` checks for success (2xx status) and also inspects response body for failure codes: `failed`, `rejected`, `declined`, `aborted`
- Interaction code validation: only `PROCEED`, `RELOAD`, `listed` are considered successful
- Returns standardized `{data, status, error}` response dict

### Resource Classes

#### Charge (`api_resources/charge.py`)

8 operations via `send_charge_request(request_type, **params)`:

| Request Type | Method | URL Pattern | Description |
|-------------|--------|-------------|-------------|
| `pay_with_preset` | POST | `/lists/{list_id}/charge/` | Charge using preset network/account |
| `standalone_charge` | POST | `/charges` | Standalone charge |
| `pay_with_network` | POST | `/lists/{list_id}/{network_code}/charge` | Charge via specific payment network |
| `pay_with_account` | POST | `/lists/{list_id}/accounts/{account_id}/charge/` | Charge via registered account |
| `charge_details` | GET | `/charges/{charge_id}/` | Get charge details |
| `cancel_charge` | DELETE | `/charges/{charge_id}/` | Cancel deferred charge |
| `pay_with_express` | POST | `/presets/{preset_id}/charge/` | Express preset charge |
| `charge_recurring` | POST | `/customers/{customer_id}/charge/` | Recurring charge for registered customer |

**Bug**: `recurring_charge_of_registered_user()` at `charge.py:113` uses `preset_id=params['params']['customer_id']` in the URL format string — the keyword argument `preset_id` should be `customer_id`. This appears to be a copy-paste error that doesn't affect functionality since Python's `str.format()` ignores extra keyword args.

#### List (`api_resources/list.py`)

8 operations via `send_list_request(request_type, **params)`:

| Request Type | Method | URL Pattern | Description |
|-------------|--------|-------------|-------------|
| `payment_session` | POST | `/lists` | Create new payment session |
| `list_details` | GET | `/lists/{list_id}/` | Get session details |
| `update_list` | PUT | `/lists/{list_id}/` | Update session |
| `delete_list` | DELETE | `/lists/{list_id}/` | Delete session |
| `payment_network` | GET | `/lists/{list_id}/{network_code}/` | Get payment network details |
| `registered_account` | GET | `/lists/{list_id}/account/{account_id}/` | Get registered account |
| `select_payment_network` | PUT | `/lists/{list_id}/{network_code}/` | Select payment network |
| `select_registered_account` | PUT | `/lists/{list_id}/account/{account_id}/` | Select registered account |

#### Customer (`api_resources/customer.py`)

Single operation: `customer_individual_account_info` — GET `/customers/{customer_id}/accounts/{account_id}/`

#### Registration (`api_resources/registration.py`)

2 operations:

| Request Type | Method | URL Pattern | Description |
|-------------|--------|-------------|-------------|
| `register_customer_account` | POST | `/lists/{list_id}/{network_code}/register/` | Register or update account |
| `delete_account` | DELETE | `/lists/{list_id}/accounts/{account_id}/` | Delete registered account |

### Error Handling (`error.py`)

5 exception classes inheriting from `Error`:

| Exception | Attributes | Use Case |
|-----------|-----------|----------|
| `Error` | (base) | Base exception |
| `InputError` | `expression`, `message` | Invalid input |
| `TransitionError` | `previous`, `next`, `message` | Invalid state transition |
| `ImproperlyConfigured` | `entity`, `entity_value`, `message` | Missing configuration (e.g., no `api_base` for live) |
| `MethodNotImplementedError` | `entity`, `entity_value`, `message` | Unknown request type |
| `APIError` | `method_name`, `status_code`, `message` | API response errors |

### Patterns

- **Request Type Map**: Each resource class defines a `REQUEST_TYPE_MAP` dict mapping logical operation names to internal method names, with `getattr()` dispatch
- **UPDATED_OBJECT_NAME**: A class-level mutable that changes the URL path segment based on the operation context (e.g., charge via list changes the base URL from `/charges` to `/lists`)
- **Param handling**: `handle_missing_params_in_request()` validates required params; `pop_params()` removes URL params from the request body before POST

---

## 4. Boarding Pass Barcode Decoder (`python_bcbp/`)

### Purpose

An in-house library for decoding IATA Bar Coded Boarding Pass (BCBP) format barcodes. Used by the duty-free customer processing pipeline (`mishipay_retail_customers`) to extract passenger and flight information from boarding pass scans at airport retail stores (e.g., Dubai Duty Free).

### How It Works

1. `Decoder(encoded_barcode)` takes the raw barcode string
2. `validate_barcode()` checks minimum length (60 chars) and valid leg count (0-3)
3. `decode_barcode()` iterates through legs, parsing mandatory and optional fields by fixed byte positions
4. Returns a dict keyed by leg number, each containing `mandatory_fields` and `optional_fields`

### Field Definitions (`fields.py`)

Four `OrderedDict` field maps with field names and byte lengths:

**First Leg Mandatory Fields (14 fields, 60 bytes)**:

| Field | Length | Description |
|-------|--------|-------------|
| `format_code` | 1 | Format code (usually `M`) |
| `no_of_legs` | 1 | Number of flight legs (1-3) |
| `passenger_name` | 20 | Passenger last/first name |
| `electronic_ticket_indicator` | 1 | `E` for e-ticket |
| `operating_carrier_pnr_code` | 7 | Airline PNR/booking reference |
| `from_city_airport_code` | 3 | Origin IATA airport code |
| `to_city_airport_code` | 3 | Destination IATA airport code |
| `operating_carrier_designator` | 3 | Airline code (e.g., `EK`) |
| `flight_number` | 5 | Flight number |
| `date_of_flight` | 3 | Julian date (day of year) |
| `compartment_code` | 1 | Cabin class |
| `seat_number` | 4 | Seat assignment |
| `check_in_sequence_no` | 5 | Check-in sequence |
| `passenger_status` | 1 | Status indicator |
| `airline_variable_length` | 2 | Hex-encoded length of optional section |

**First Leg Optional Fields (24 fields)**:
Includes version info, passenger description, boarding pass issuance source/date, document type, baggage tag numbers (conditional — included only if `field_size_of_structure_msg_unique > 24`), airline numeric code, frequent flyer info, fast track indicator, and individual airline use (variable-length).

**Additional Leg Mandatory Fields (11 fields, 37 bytes)**: Same as first leg minus `format_code`, `no_of_legs`, `passenger_name`, `electronic_ticket_indicator`.

**Additional Leg Optional Fields (15 fields)**: Same as first leg optional plus security data fields.

### Parsing Logic (`utils.py`)

`ParseFields` class maintains position state (`start_index`, `end_index`) across sequential leg parsing:

- `parse_first_leg(leg)`: Extracts mandatory fields by position, reads `airline_variable_length` as hex to determine optional section size, conditionally includes baggage tag fields based on `field_size_of_structure_msg_unique`
- `parse_nth_leg(leg)`: Parses additional legs with similar logic, maintains cumulative position tracking

### Limitations

- `get_first_leg_mandatory_fields()` is a stub (commented-out implementation at `decoder.py:51-69`)
- `get_indent_format()` and `extract_legs()` are empty stubs
- `find_no_of_legs()` uses bare `except Exception` and `print()` for error handling
- No unit tests included in the package

---

## 5. HTTP Request Logger (`logging/`)

### Purpose

A lightweight utility providing structured JSON logging of HTTP request/response pairs for use with the `requests` library's hook system. Used by microservice client code to log inter-service API calls.

### API

**`json_log_factory(logger=None, *, raise_exc=True)`** (`logging/requests.py`)

Returns a callable suitable for `requests.Session().hooks["response"]`:

```python
session = requests.Session()
session.hooks["response"].append(json_log_factory(my_logger, raise_exc=False))
```

Behavior:
- Logs request method, URL, and body (pretty-printed JSON if parseable)
- Logs response status and body (pretty-printed JSON if parseable)
- Labels outcome as `"Successful"` or `"Failed"` based on HTTP status code
- Successful responses logged at `DEBUG` level; failures at `ERROR` level
- If `raise_exc=True` (default), raises `HTTPError` on non-success status codes
- Handles binary request bodies by decoding to UTF-8
- Uses provided logger or creates one via `getLogger(__name__)`

This is the cleanest, most modern code in the `lib/` directory — uses type hints, f-strings, and Python 3 typing.

---

## Dependencies

### What `lib/` depends on

| Package | Dependencies |
|---------|-------------|
| `backend_profiling` | Django, `sqlparse`, `termcolor`, `memory_profiler` (optional), `line_profiler` (optional) |
| `explorer` | Django, `sqlparse`, `six`, `xlsxwriter`, `boto` (optional, for S3), `unicodecsv` (Python 2 only) |
| `pyoptile` | `requests`, `json` |
| `python_bcbp` | Standard library only (`collections.OrderedDict`) |
| `logging` | `requests` |

### What depends on `lib/`

- **`backend_profiling`**: Referenced in `settings.py` `INSTALLED_APPS` and middleware chain. Currently disabled (all modules commented out).
- **`explorer`**: Referenced in `settings.py` `INSTALLED_APPS`, mounted at `/explorer/` in URL config. Used by platform admins for ad-hoc database queries and reporting.
- **`pyoptile`**: Used by `dos/` module's payment gateway classes (specifically `PayUnity` PSP type in `dos/utils/payment_gateways/payunity.py`).
- **`python_bcbp`**: Used by `mishipay_retail_customers/` for duty-free boarding pass processing (airport retail stores like DDF).
- **`logging`**: Used by microservice client code for request/response logging.

## Notable Patterns

1. **Vendored library**: `explorer` is a vendored copy of `django-sql-explorer` v1.1.2 (setup.py references Django 1.8-1.10, Python 2.7/3.5-3.6). It has been modified in-place (MishiPay-specific permission logic, `print()` statements added to views). The upstream library has progressed significantly (v4.x+ as of 2024) — the vendored copy is over 7 years behind.

2. **Python 2/3 compatibility**: Both `explorer` and `pyoptile` contain `from __future__ import` statements, `six` usage, and Python 2 compatibility patterns (`__unicode__` methods, `PY3` conditionals). `backend_profiling/utils/stack.py` imports Python 2's `SocketServer` module and is incompatible with Python 3.

3. **Debug print statements**: Pervasive across the lib packages:
   - `backend_profiling/models.py:51` — `print("NAME", name)` during module loading
   - `explorer/views.py:71,76,79,89,384` — Permission and query debug info
   - `explorer/schema.py:21` — Exclude filter debug
   - `python_bcbp/decoder.py:29` — Exception printing

4. **Stripe-inspired architecture**: `pyoptile` follows Stripe's Python SDK design patterns — module-level config globals, `APIResource` base class, request type mapping, class-level `OBJECT_NAME`, and `instance_url()` URL building.

5. **Disabled-by-default profiling**: `backend_profiling` is installed but all modules are commented out in the default configuration. It's a development-only tool that can be activated by uncommenting module entries in Django settings.

6. **SQL safety limitations**: Explorer's blacklist includes `CREATED`, `UPDATED`, `DELETED` which are common column names in MishiPay models (e.g., `ActionConstraint.created`, `EntityProperty.updated`). Any `SELECT` query mentioning these columns would be blocked.

## Open Questions

- Is `backend_profiling` ever enabled in development environments? All modules are commented out, and `utils/stack.py` is incompatible with Python 3.
- Has the vendored `explorer` been considered for upgrade to a newer version of `django-sql-explorer`? The vendored v1.1.2 is significantly outdated and contains Python 2 compatibility code.
- What are the actual values of `EXPLORER_CONNECTIONS` and `EXPLORER_DEFAULT_CONNECTION` in production? The settings file shows `explorer` is connected to the default database with no read-only restriction.
- Is the `EXPLORER_TOKEN` setting changed from its default value `"CHANGEME"` in production? If `EXPLORER_TOKEN_AUTH_ENABLED` is `True`, this would allow unauthenticated API access with the default token.
- Is `pyoptile` still in active use? The Optile/Payoneer integration via `PayUnity` in the DOS module appears to be a legacy payment path. Which retailers, if any, still use Optile/PayUnity?
- The `python_bcbp` library has several stub methods (`get_first_leg_mandatory_fields`, `get_indent_format`, `extract_legs`). Is the `decode_barcode()` path the only one used in production, or are these methods planned for implementation?
- The Explorer's `Query.Meta.permissions` defines `can_drive`, `can_vote`, `can_drink` — are these used anywhere, or are they artifacts from the original library's test suite?
