# Retail Analytics & Background Jobs

> Last updated by: Task T29 — Added legacy analytics module (AppEvents, SalesSummary, Notifications), analytics_library (Mixpanel integration with 15 event types), and their relationship to the newer mishipay_retail_analytics system

## Overview

The MishiPay platform collects fine-grained analytics for every significant user interaction — item scans, order creation, payments, login/signup, feedback, refunds, and app opens. Two Django apps collaborate to support this:

- **`mishipay_retail_analytics`** (64 files) — An event-sourced analytics system that captures user actions via a Subject-Verb-Object data model, stores them in PostgreSQL, and exposes dashboard reporting APIs with two generations (V1 event-based, V2 aggregated).
- **`mishipay_retail_jobs`** (14 files) — A Dramatiq-based background task processing framework that tracks job lifecycle in the database via custom middleware. Handles refund processing, third-party HTTP calls, and POS log generation.

## Key Concepts

- **Event Context** — Every analytics event is wrapped in an `EventContext` that captures *who* (subject), *what* (verb), *where* (indirect object = store), plus device, location, IP, and application metadata. Domain-specific data (product, order, payment, etc.) is stored in separate "object" models linked 1:1 to the context.
- **Aggregated Analytics** — A pre-computed `AggregatedAnalytics` table stores daily per-store/platform summaries. V2 API views query this table instead of scanning raw event records, providing much better query performance.
- **Dramatiq** — The background task framework. Tasks are Python functions decorated with `@dramatiq.actor` and dispatched to RabbitMQ queues. A custom middleware tracks task state in the database.

---

## Architecture

### Event-Sourced Analytics Pattern

```
User Action (scan, pay, login, ...)
    |
    v
CreateContextObject  ──────────────────────────> PostgreSQL stored procedure
    |                                             fn_create_analytics_event1()
    |                                             (21 parameters, creates all
    |                                              context rows atomically)
    v
EventContext (hub)
    |── EventSubject (who)
    |── EventVerb (what)
    |── EventIndirectObject (store context)
    |── LocationFields (geolocation)
    |── IPAddressFields (client IP)
    |── ApplicationFields (bundle ID, platform)
    |── DeviceAndOperatingFields (user agent, device ID)
    |
    v
Domain Object (1:1 to EventContext)
    ProductObject | OrderObject | PaymentObject | ...
```

The `CreateContextObject` class (`create_context.py`) orchestrates event creation. Rather than using Django ORM saves, it calls a PostgreSQL stored procedure `fn_create_analytics_event1()` via raw SQL with 21 typed parameters. This creates all context rows atomically and returns the generated context ID, which is then used to query back the full object graph.

### V1 vs V2 API Architecture

**V1** (`views.py`) queries raw event tables (`PaymentObject`, `ProductObject`, `FeedbackObject`, `LoginSignupObject`) directly. Date filtering converts local timezone to UTC when store types are in the `ANALYTICS_LOCAL_TIMEZONE_STORE_TYPES` GlobalConfig list.

**V2** (`views_v2.py`) queries the pre-aggregated `AggregatedAnalytics` model. Date filtering uses `date` or `business_date` fields depending on whether stores support local timezone analytics.

The URL routing (`urls.py`) shows V2 has largely replaced V1 — the `v1/` URL paths for transaction and rating endpoints actually point to V2 view classes. Only `AnalyticsDashboard` and `ItemScannedAnalytics` still use V1 classes.

### Background Jobs Architecture

```
Application Code
    |
    v
@dramatiq.actor(queue_name=..., max_retries=..., min_backoff=...)
    |
    v
RabbitMQ Queue
    |
    v
Dramatiq Worker Process
    |── CustomAdminMiddleware.after_enqueue()    -> Task(status=ENQUEUED)
    |── CustomAdminMiddleware.before_process()   -> Task(status=RUNNING)
    |── CustomAdminMiddleware.after_process()    -> Task(status=DONE|FAILED)
    |
    v
Task model (database record)
```

---

## Data Models

### Analytics Core Context Models (8 models)

| Model | File | Key Fields | Purpose |
|-------|------|------------|---------|
| `EventSubject` | `models.py:19` | `subject_id` (UUID), `subject_unique_key`, `user_identifier` | Who performed the action (customer ID) |
| `EventVerb` | `models.py:28` | `verb_name` (CharField 100) | Action type (e.g., `item_scan`, `payment_captured`) |
| `EventIndirectObject` | `models.py:34` | `indirect_object_id` (UUID, indexed), `indirect_object_type`, `indirect_object_status` (int, choices), `indirect_object_region`, `indirect_object_retailer`, `indirect_object_sub_region`, `extra_data` (JSON) | Store context where event occurred |
| `ApplicationFields` | `models.py:50` | `app_id` (CharField 100), `platform` (CharField 25), `version` | Client application metadata |
| `DeviceAndOperatingFields` | `models.py:58` | `user_agent` (TextField), `device_id` (CharField 100) | Device information |
| `LocationFields` | `models.py:65` | `geo_latitude`, `geo_longitude` (FloatField), `geo_latitude_rounded`, `geo_longitude_rounded`, `outside_geofencing` (BooleanField), `geo_country`, `geo_region`, `geo_city`, `geo_zipcode`, `geo_region_name`, `geo_timezone` | Geolocation data; rounded coordinates used when outside geofencing |
| `IPAddressFields` | `models.py:85` | `ip_address` (GenericIPAddressField), `ip_isp`, `ip_country` | Client IP metadata |
| `EventContext` | `models.py:93` | `event_id` (UUID, unique), `timestamp` (DateTimeField, indexed), OneToOne FKs to all context models above | Central hub model linking all context fields |

**Relationship pattern:** `EventContext` has `OneToOneField` to `EventSubject`, `EventIndirectObject`, `LocationFields`, `IPAddressFields`, `ApplicationFields`, `DeviceAndOperatingFields`, and `ForeignKey` to `EventVerb`. All use `CASCADE` delete.

### Analytics Domain Event Models (10 models)

Each domain model has a `OneToOneField` to `EventContext`:

| Model | File | Key Fields | Purpose |
|-------|------|------------|---------|
| `ProductObject` | `models.py:116` | `item_name`, `product_identifier`, `product_identifier_type`, `epc`, `item_category`, `price` (Decimal 15,2), `is_restricted`, `is_item_sold`, `discount_percent`, `promo_code`, `discount_constraint` (JSON), `extra_data` (JSON) | Item scan events |
| `OrderObject` | `models.py:159` | `order_id` (UUID), `item_count` (int), `basket_id` (UUID, nullable), `extra_data` (JSON) | Order creation events |
| `PaymentObject` | `models.py:169` | `order_id` (UUID, indexed), `price` (Decimal 9,2), `price_excluding_vat` (Decimal 9,2), `payment_status` (indexed), `payment_psp`, `payment_method` (indexed), `item_count`, `extra_data` (JSON) | Payment capture events. Custom `save()` calculates `price_excluding_vat` by looking up the Order's basket VAT. `created` uses `auto_now_add=False` — timestamps are set explicitly. |
| `LoginSignupObject` | `models.py:201` | `screen` (CharField 50), `mode` (CharField 50), `extra_data` (JSON) | Login and signup events |
| `FeedbackObject` | `models.py:210` | `rating` (int, default 0), `feedback` (text), `order_id` (UUID), `tags` (text), `extra_data` (JSON) | Customer rating/feedback events |
| `AppOpenObject` | `models.py:221` | `extra_data` (JSON) | App launch tracking |
| `RefundObject` | `models.py:228` | `order_id` (UUID), `refund_amount` (Decimal 9,2), `refund_type`, `payment_id` (UUID), `item_count`, `refund_status`, `transaction_id`, `extra_data` (JSON) | Refund events |
| `VerifiedOrderObject` | `models.py:242` | `order_id` (UUID), `payment_id` (UUID), `order_status`, `price` (Decimal 9,2), `item_count` | Order verification events |
| `JourneyTypeObject` | `models.py:253` | `journey_type` (choices: entry/exit/not_available), `status`, `unique_identifier`, `extra_data` (JSON) | Store entry/exit journey tracking |
| `RecommendedItems` | `models.py:274` | `store` (FK to Store), `basket` (FK to Basket, indexed), `epc`, `product_identifier` (indexed), `retailer_product_id` | Recommended item tracking per basket |

### Aggregated Analytics Model

`AggregatedAnalytics` (`models.py:292`) — Pre-computed daily store/platform analytics:

| Field | Type | Purpose |
|-------|------|---------|
| `date` | DateField | Calendar date |
| `business_date` | DateField | Business/trading date (may differ for timezone-adjusted stores) |
| `store_name`, `store_id`, `store_type`, `region` | CharField/UUID | Store identification |
| `demo` | BooleanField | Whether store is in demo mode |
| `platform` | CharField | App platform (ios, android, web) |
| `device_id` | CharField (nullable) | Self-checkout device identifier |
| `total_txn_count` | IntegerField | Transaction count |
| `total_items_purchased` | IntegerField | Items purchased |
| `currency` | CharField(10) | Store currency |
| `total_basket_value` | Decimal(10,2) | Total sales value |
| `total_basket_value_tax_excluded` | Decimal(10,2) | Sales value excluding tax |
| `average_basket_value`, `average_basket_size` | Decimal(10,2) | Pre-computed averages |
| `total_items_scanned` | IntegerField | Scan count |
| `login_count`, `sign_up_count` | IntegerField | Auth event counts |
| `ratings_dict` | JSONField | Rating breakdown by score (e.g., `{"0": 0, "1": 5, ...}`) |
| `extra_data` | JSONField | Extended data including `payment_method_details` dict |

**Indexes:** Composite indexes on `(date, store_id, demo, platform, device_id)` and `(business_date, ...)`. Unique constraint on `(demo, platform, store_id, date, business_date, device_id)`.

### Jobs Models

| Model | File | Key Fields | Purpose |
|-------|------|------------|---------|
| `Task` | `mishipay_retail_jobs/models.py:37` | `job_id` (UUID, unique), `job_name`, `message_data` (BinaryField — serialized Dramatiq Message), `extra_data` (JSON), `params` (JSON), `manual_run`, `manual_cancel` (BooleanField), `queue_name`, `broker_params` (JSON), `retries` (PositiveSmallIntegerField), `traceback` (TextField), `status` (choices from TaskStatus), `priority` (choices from TaskPriority) | Database record of every background task |

**Manager:** `TaskManager` provides `create_or_update_from_message(message, **extra_fields)` and `delete_old_tasks(max_task_age)`.

---

## Constants & Enums

### Analytics Constants (`__init__.py`)

| Constant | Values | Purpose |
|----------|--------|---------|
| `IndirectStoreStatusChoices` | `DEMO=0`, `LIVE=1`, `NOT_AVAILABLE=2` | Store demo/live status for event context |
| `JourneyTypes` | `ENTRY='entry'`, `EXIT='exit'`, `NOT_AVAILABLE='not_available'` | Store entry/exit journey types. **Note:** `NOT_AVAILABLE` has a trailing comma making it a tuple `('not_available',)` — a latent bug that doesn't surface because the `JourneyTypeObject` model uses inline choices instead of referencing this constant (the import is commented out in `models.py:11`). |
| `ANALYTICS_PAYMENT_STATUS` | `{100: 'already_processed', 202: 'pending'}` | Payment status code mapping |

### Jobs Constants (`__init__.py`)

| Constant | Values | Purpose |
|----------|--------|---------|
| `TaskPriority` | `HIGH=0`, `NORMAL=50`, `LOW=100` | Priority levels (lower = higher priority) |
| `TaskStatus` | `SCHEDULED`, `ENQUEUED`, `RUNNING`, `DONE`, `FAILED`, `CANCELLED`, `DELAYED` | Task lifecycle states |

### Queue Names (`app_settings.py`)

| Constant | Value | Used By |
|----------|-------|---------|
| `REFUND_QUEUE_NAME` | `'decathlon_refund'` | `check_refund_status_and_update_entities` |
| `UPDATE_THIRD_PARTY_QUEUE_NAME` | `'update_third_party'` | `async_request` |
| `POSLOG_QUEUE` | `'process_transaction_for_poslog'` | `send_bwg_poslog_to_retailer` |

---

## Event Builder Classes

Each domain event type has a corresponding builder class that constructs and persists the domain object:

| Class | File | Creates | Notable Details |
|-------|------|---------|-----------------|
| `CreateContextObject` | `create_context.py` | `EventContext` + all sub-models | Uses raw SQL stored procedure `fn_create_analytics_event1()`. Builds subject, verb, indirect object, location, app, device, and IP sub-objects without ORM saves — all persisted via the stored procedure. Geofencing check uses `dos.util.check_distance_between_two_locations()`. |
| `ProductObjectClass` | `product_object.py` | `ProductObject` | Handles 3 item sources: (1) null item (just barcode), (2) inventory-service format (`source='inventory-service'`), (3) standard format with full product details |
| `OrderObjectClass` | `order_object.py` | `OrderObject` | Simple — maps `order_id`, `basket_id`, `item_count`, `extra_data` |
| `PaymentObjectClass` | `payment_object.py` | `PaymentObject` | Maps `order_id`, `price`, `status`, `psp`, `payment_method`. Uses `datetime.now()` instead of timezone-aware `django.utils.timezone.now()`. |
| `LoginSignupObjectClass` | `login_signup_object.py` | `LoginSignupObject` | Maps `screen` (login/signup) and `mode` (email/google/facebook/guest_email) |
| `FeedbackObjectClass` | `feedback_and_rating_object.py` | `FeedbackObject` | Maps `order_id`, `rating`, `feedback` text, `tags` |
| `AppOpenObjectClass` | `app_open_object.py` | `AppOpenObject` | Minimal — only `extra_data` from params |
| `RefundObjectClass` | `refund_object.py` | `RefundObject` | Maps refund details including `refund_type`, `refund_status`, `transaction_id`, `return_items` |
| `OrderVerifiedObjectClass` | `order_verified_object.py` | `VerifiedOrderObject` | Maps `order_id`, `payment_id`, `item_count`, `price`, `order_status` |
| `JourneyTypeObjectClass` | `journey_type_object.py` | `JourneyTypeObject` | Maps `journey_type` (lowercased), `status` (lowercased), `unique_identifier` |

---

## API Endpoints

### Analytics URL Routes (`urls.py`)

| Path | View Class | URL Name | Version |
|------|-----------|----------|---------|
| `v1/analytics-dashboard/` | `AnalyticsDashboard` (V1) | `analytics_dashboard` | V1 |
| `v1/transaction-retailer-breakdown/` | `TransactionByRetailerV2` | `transaction_retailer_breakdown_v2` | **V2 on V1 path** |
| `v1/transaction-breakdown/` | `TransactionBreakdownV2` | `transaction_breakdown_v2` | **V2 on V1 path** |
| `v1/item-scanned-analytics/` | `ItemScannedAnalytics` (V1) | `item_scanned_analytics` | V1 |
| `v1/ratings-analytics/` | `RatingAnalyticsV2` | `ratings_analytics_v2` | **V2 on V1 path** |
| `v1/login-signup-analytics/` | `LoginSignupAnalyticsV2` | `login_signup_analytics_v2` | **V2 on V1 path** |
| `v2/transaction-retailer-breakdown/` | `TransactionByRetailerV2` | `transaction_retailer_breakdown_v2` | V2 |
| `v2/transaction-breakdown/` | `TransactionBreakdownV2` | `transaction_breakdown_v2` | V2 |
| `v2/ratings-analytics/` | `RatingAnalyticsV2` | `ratings_analytics_v2` | V2 |
| `v2/login-signup-analytics/` | `LoginSignupAnalyticsV2` | `login_signup_analytics_v2` | V2 |

**Note:** The V1 `TransactionByRetailer`, `TransactionBreakdown`, `RatingsAnalytics`, and `LoginSignupAnalytics` view classes from `views.py` are no longer wired into URL routes. They still exist in code but are effectively dead code. The V1 paths now serve V2 views.

**Note:** `ItemScannedAnalyticsV2` exists in `views_v2.py` but is not wired into any URL pattern.

### View Details

#### `AnalyticsDashboard` (V1, `views.py:80`)
- **Method:** GET
- **Permissions:** None (no permission classes)
- **Query params:** `analytics_filter_type` (required: `allstores`, `store_type`, `multivalue_store_id`, `single_store_id`), `analytics_filter_values` (comma-separated), `timezone`, `start_date`, `end_date`, `store_id`
- **Returns:** Comprehensive dashboard payload with item scans, ratings, login/signup counts, transaction totals, payment method breakdowns, platform breakdowns — all computed from raw event tables
- **Uses:** `@transaction.atomic` decorator

#### `TransactionByRetailerV2` (`views_v2.py:57`)
- **Method:** GET
- **Permissions:** `DashboardPermission`, `AllowAnalyticsPermission`
- **Query params:** `start_date`, `end_date`, `store_ids`, `store_types`, `regions`, `app_clip_ids`, `show_demo`, `include_demo`, `include_inactive`, `base_currency` (default: `gbp`), `platform`, `sco_id`
- **Returns:** Per-store transaction breakdown with currency conversion, average order value, basket size, ratings. Sorted by transaction count descending.
- **Currency conversion:** Calls `exchangerate-api.com` (hardcoded API key). Sends Slack alert to `#dashboard_issue_alerts` when conversion rates are unavailable.

#### `TransactionBreakdownV2` (`views_v2.py:302`)
- **Method:** GET
- **Permissions:** `DashboardPermission`, `AllowAnalyticsPermission`
- **Query params:** Same filter set as TransactionByRetailer
- **Returns:** `platform_breakdown` (android/ios/web counts) and `payment_method_breakdown` (per-method counts from `extra_data.payment_method_details`)

#### `ItemScannedAnalytics` (V1, `views.py:539`)
- **Method:** GET
- **Permissions:** `DashboardPermission`, `AllowAnalyticsPermission`
- **Returns:** `total_item_scanned` count from raw `ProductObject` table

#### `RatingAnalyticsV2` (`views_v2.py:472`)
- **Method:** GET
- **Permissions:** `DashboardPermission`, `AllowAnalyticsPermission`
- **Returns:** `rating_breakdown` (0-5 star distribution), `average_rating`, `rating_breakdown_platform`

#### `LoginSignupAnalyticsV2` (`views_v2.py:431`)
- **Method:** GET
- **Permissions:** `DashboardPermission`, `AllowAnalyticsPermission`
- **Returns:** `login` and `signup` counts

---

## Business Logic

### Store Filtering (`get_user_allowed_stores`, `views.py:270`)

All dashboard API views use this function to determine which stores the current user can see. It:

1. Gets the user's assigned stores (`user.stores`)
2. Filters by `demo`, `active`, `include_demo`, `include_inactive` flags
3. Further filters by `store_ids`, `store_types`, `regions`, `app_clip_ids` query params
4. Returns a dict keyed by `store_id` with store details (region, currency, name, store_type, timezone)

### Date Filtering

**V1** (`filter_analytics_query_by_date`, `views.py:339`):
- Checks `ANALYTICS_LOCAL_TIMEZONE_STORE_TYPES` GlobalConfig to determine if timezone-aware filtering is needed
- If all stores are in the configured local timezone list and share the same timezone: converts dates to UTC using `get_utc_date_time()` with `logical_date_hour_difference` offset, then filters on `created__gt`/`created__lte`
- Otherwise: filters on `created__date__gte`/`created__date__lte` (date-only comparison)

**V2** (`filter_analytics_query_by_date`, `views_v2.py:25`):
- If stores use local timezone analytics and share a single timezone: filters on `business_date`
- Otherwise: filters on `date`

### Currency Conversion

Both `TransactionByRetailer` (V1) and `TransactionByRetailerV2` call the Exchange Rate API:
- **URL:** `https://v6.exchangerate-api.com/v6/24f8b7beb83be48d1eaa9ea4/latest/{base_currency}`
- **API key:** Hardcoded `24f8b7beb83be48d1eaa9ea4`
- **Default base currency:** GBP
- **Fallback:** When conversion fails, `exchange_rate = 1` (no conversion), `use_total_order_value = False`
- **Special case:** Currency `EURO` is normalized to `EUR`
- **Slack alerts:** Sends to `#dashboard_issue_alerts` when conversion rates are missing (production only)

### Platform Normalization

All platform-related views normalize case-insensitive platform strings into lowercase buckets:
- `web_ios` and `web_android` counts are rolled into `web`
- `ANDROID_PLATFORMS_LIST` variants are summed into `android`
- Unknown platforms default to `web` with a Slack alert

### Geofencing (`create_context.py:75-100`)

When creating location data for events:
1. Extracts `r_lat`/`r_long` from request query params (falls back to `store_latitude`/`store_longitude` from extra_params)
2. Validates coordinates with regex `[+-]?(\d+){1,3}.(\d+)`
3. Calls `dos.util.check_distance_between_two_locations()` to determine if user is outside the store's geofence
4. Sets `outside_geofencing = True` if distance exceeds `settings.GEOFENCING_RANGE`

---

## Background Tasks

### Defined Tasks (`mishipay_retail_jobs/tasks.py`)

| Task Function | Queue | Max Retries | Min Backoff | Purpose |
|---------------|-------|-------------|-------------|---------|
| `count_words(url)` | `local_queue` | 3 | default | Sample/test task — fetches URL and counts words |
| `SampleTask.perform(n)` | `tasks` | 2 | default | Sample GenericActor — prints count to N |
| `check_refund_status_and_update_entities(order_id, item_id_quantity_map, post_data)` | `decathlon_refund` | 18 | 150ms | Checks refund status with PSP, updates basket items and payment records, triggers order fulfilment |
| `async_request(method, url, headers, query_params, payload)` | `update_third_party` | 5 | 60s | Generic async HTTP request wrapper for third-party notifications |
| `send_bwg_poslog_to_retailer(order_id)` | `process_transaction_for_poslog` | 3 | 1s | Generates BWG POSLOG file for completed/verified orders |

### Refund Task Detail (`check_refund_status_and_update_entities`)

This is the most complex background task. Flow:

1. Look up Order and Payment objects
2. Get PSP provider via `provider_factory`
3. Call `provider.check_refund_status(payment_obj, post_data)`
4. **If HTTP 202 (pending):** Raises exception to trigger retry (up to 18 retries with 150ms initial backoff)
5. **If HTTP 200 (success):**
   - Updates `BasketItem.refunded_quantity` and `refund_processing_quantity`
   - Sets `is_refunded = True` when fully refunded
   - Updates `Payment.refunded_amount`
   - Changes payment status to `REFUNDED` when fully refunded
   - Checks if all basket items are refunded to set `Order.order_status = REFUNDED`
   - Triggers `DecathlonOrderFulfilment` for refund fulfilment
   - Stores fulfilment ID in `order.extra_data`
6. **If failed:** Resets `refund_processing_quantity` on basket items; re-marks EPCs as sold for RFID stores
7. **Always:** Appends refund response data to `payment.extra_data.payment_responses.refund_responses`; decrements `refund_processing_amount`

### BaseTask (`abstract.py`)

Abstract `GenericActor` base class for Dramatiq tasks:
- Default queue: `"tasks"`, max_retries: 5, priority: 0
- Subclasses must implement `get_task_name()`
- Only `SampleTask` uses this base class; the production tasks use `@dramatiq.actor` decorator directly

### CustomAdminMiddleware (`middleware.py`)

Dramatiq middleware that records task lifecycle in the database:

| Hook | Triggers | Updates |
|------|----------|---------|
| `after_enqueue(broker, message, delay)` | Task added to queue | Creates/updates `Task` with status `ENQUEUED` (or `DELAYED` if delay > 0). Records queue name, actor name, args/kwargs, broker connection info (host, port, virtual_host), priority, retry count. |
| `before_process_message(broker, message)` | Worker picks up task | Updates `Task` status to `RUNNING` |
| `after_process_message(broker, message, result, exception)` | Task completes | Updates `Task` status to `DONE` or `FAILED`. Records traceback if exception occurred. |

---

## Utility Functions

### Analytics Utilities (`util.py`)

| Function | Purpose |
|----------|---------|
| `get_lat_long_based_on_outside_geofencing(context_obj)` | Returns lat/long rounded to 2 decimal places when `outside_geofencing` is True; original precision otherwise |
| `get_analytics_common_params(context_obj)` | Builds standard analytics parameter dict from EventContext (timestamp, distinct_id, store_id, platform, geolocation, etc.) |
| `get_client_ip(request)` | Extracts client IP from `X-Forwarded-For` header (first IP), falling back to `REMOTE_ADDR` |
| `get_saturnsmartpay_product_details(item_data)` | Formats product details for Saturn SmartPay store type |
| `get_decathlon_product_details(item_data)` | Formats product details for Decathlon store type |
| `get_product_details_based_on_store_type(store_type, product_details)` | Dispatches to store-type-specific product formatter via `ITEM_DETAILS_MAPPING` |
| `ITEM_DETAILS_MAPPING` | Dict mapping store type keys to product detail functions: `saturnsmartpaystoretype` -> Saturn, `decathlonstoretype` -> Decathlon |

---

## Management Commands

Six management commands for migrating analytics data from CSV exports (originally from Mixpanel):

| Command | File | Purpose |
|---------|------|---------|
| `send_events_to_mixpanel_new_architecture` | `management/commands/send_events_to_mixpanel_new_architecture.py` | Reads existing analytics objects from DB and sends to external analytics service via `AnalyticsRecordClient`. Handles item scans, orders, payments, logins, and ratings. |
| `order_mixpanel_migration` | `management/commands/order_mixpanel_migration.py` | Imports order completion events from CSV. Creates full EventContext chain + OrderObject. Args: `-file` (CSV path), `-type` (event type) |
| `payment_mixpanel_migration` | `management/commands/payment_mixpanel_migration.py` | Imports payment captured events from CSV with PSP mapping logic. Args: `-file`, `-type`, `-store_type`. Includes `PAYMENT_PSP` mapping for Saturn and Decathlon payment methods. |
| `item_scan_mixpanel_migration` | `management/commands/item_scan_mixpanel_migration.py` | Imports item scan events from CSV. Uses `get_product_details_based_on_store_type()` for store-specific formatting. |
| `feedback_mixpanel_migration` | `management/commands/feedback_mixpanel_migration.py` | Imports feedback/rating events from CSV. |
| `login_signup_mixpanel_migration` | `management/commands/login_signup_mixpanel_migration.py` | Imports login/signup events from CSV. |

**Common patterns across CSV migration commands:**
- All include a `BUNDLE_ID_MAPPPING` (note: typo in variable name) dict mapping ~30 bundle IDs to platform (ios/android/web)
- All include `check_distance_between_two_locations()` and `calculate_geofencing_status()` for computing geofencing from CSV lat/long data
- All parse `store_specific_params` JSON field from CSV for customer ID, store ID, store type, and coordinates
- All explicitly set `created`/`updated` timestamps from CSV data (using `update_or_create` pattern to bypass `auto_now_add`)

---

## Admin Configuration

### Analytics Admin (`admin.py`)

14 models registered with Django admin, all using `CachedTablePaginator` for performance on large tables:

| Model | Search Fields | List Filters | Raw ID Fields |
|-------|--------------|-------------- |---------------|
| `EventSubject` | `subject_id`, `subject_unique_key` | `created`, `updated` | — |
| `EventVerb` | `verb_name` | `verb_name`, `created`, `updated` | — |
| `EventIndirectObject` | `indirect_object_id`, `indirect_object_type` | `indirect_object_type`, `created`, `updated` | — |
| `ApplicationFields` | `app_id`, `platform`, `version` | `app_id`, `platform`, `version`, `created`, `updated` | — |
| `DeviceAndOperatingFields` | — | `created`, `updated` | — |
| `LocationFields` | — | `outside_geofencing`, `created`, `updated` | — |
| `IPAddressFields` | `ip_address` | `created`, `updated` | — |
| `EventContext` | — | `timestamp`, `created`, `updated` | `subject`, `verb`, `indirect_obj`, `location`, `ip_address`, `application`, `device_and_operating` |
| `ProductObject` | — | `created`, `updated` | `event_context` |
| `OrderObject` | — | `created`, `updated` | `event_context` |
| `PaymentObject` | — | `payment_status`, `created`, `updated` | `event_context` |
| `LoginSignupObject` | — | `screen`, `created`, `updated` | `event_context` |
| `FeedbackObject` | — | `rating`, `created`, `updated` | `event_context` |
| `RefundObject` | — | `refund_type`, `created`, `updated` | `event_context` |
| `VerifiedOrderObject` | — | `order_status`, `created`, `updated` | `event_context` |
| `AppOpenObject` | — | — | `event_context` |
| `AggregatedAnalytics` | `store_id` | — | — |

### Jobs Admin (`mishipay_retail_jobs/admin.py`)

`TaskAdmin` with:
- **List display:** `job_id`, `job_name`, `params`, `status`, `queue_name`, `broker_params`, `retries`, `priority_value` (custom method), `manual_run`, `manual_cancel`
- **List filters:** `job_name`, `status`, `queue_name`, `manual_run`, `manual_cancel`
- **Search fields:** `job_id`, `job_name`, `params`, `status`, `queue_name`, `broker_params`, `created_at`, `updated_at`

---

## Test Coverage

### Analytics Tests

- **`tests/factories.py`** — 8 factory classes using `factory_boy`: `EventIndirectObjectFactory`, `ApplicationFieldsFactory`, `EventSubjectFactory`, `EventVerbFactory`, `EventContextFactory`, `PaymentObjectFactory`, `ProductObjectFactory`, `FeedbackObjectFactory`, `LoginSignupObjectFactory`
- **`tests/dashboard_analytics_test.py`** — 4 test methods in `TestDashboardAnalytics`:
  - `test_dashboard_analytics_no_store` — Verifies zero results when user has no store access
  - `test_dashboard_analytics_all_store_access` — Tests full access across 5 stores with various filters (store_ids, store_types, regions, date ranges)
  - `test_dashboard_analytics_limited_store_access` — Tests results scoped to single store
  - `test_dashboard_with_app_clip_id` — Tests filtering by app clip IDs

### Jobs Tests

- **`tests.py`** — Empty stub file (no tests implemented)

### Migrations

33 analytics migrations spanning 2019-2025 (`0001_initial` through `0033`).

---

## Dependencies

### Analytics Dependencies

| Depends On | Purpose |
|------------|---------|
| `dos.models.Store` | Store data for indirect objects, geofencing |
| `dos.util.check_distance_between_two_locations` | Geofencing calculation |
| `mishipay_items.models.Basket` | FK for `RecommendedItems` |
| `mishipay_retail_orders.models.Order` | Payment VAT calculation in `PaymentObject.save()` |
| `mishipay_config.models.GlobalConfig` | `ANALYTICS_LOCAL_TIMEZONE_STORE_TYPES` config |
| `mishipay_core.common_functions` | `get_rounded_value`, `send_slack_message`, `get_generic_response_structure` |
| `mishipay_core.choices` | Platform choice constants |
| `mishipay_dashboard.permissions` | `DashboardPermission`, `AllowAnalyticsPermission` |
| `mishipay_dashboard.util` | `get_average_ratings()` |
| `microservices_client.client` | `AnalyticsRecordClient` for external event forwarding |
| External: `exchangerate-api.com` | Currency conversion for dashboard |
| External: `pytz` | Timezone handling |

### Jobs Dependencies

| Depends On | Purpose |
|------------|---------|
| `dramatiq` | Task queue framework (GenericActor, Message, Middleware) |
| `mishipay_retail_orders.models.Order` | Refund task order lookup |
| `mishipay_items.models.BasketItem` | Refund task item updates |
| `mishipay_retail_payments.models.provider_factory` | PSP provider instantiation |
| `mishipay_retail_orders.third_party_integrations` | Decathlon order fulfilment |
| `mishipay_retail_orders.app_settings.EPC_SOLD_FUNCTIONS_MAP` | RFID EPC status management |
| `mishipay_core.common_functions.update_dict_using_dotted_path` | Nested dict updates |
| RabbitMQ | Message broker (broker params recorded in Task model) |

### Depended On By

| Module | Uses |
|--------|------|
| `mishipay_dashboard` | Imports analytics views for dashboard display |
| Various order/payment views | Call event builder classes to record analytics |

---

## Configuration

| Config Key | Source | Purpose |
|------------|--------|---------|
| `ANALYTICS_LOCAL_TIMEZONE_STORE_TYPES` | `GlobalConfig` (database) | Dict mapping store types to `logical_date_hour_difference` for timezone-aware date filtering |
| `GEOFENCING_RANGE` | `settings.py` | Distance threshold (meters) for outside-geofencing detection |
| `DRAMATIQ_DATABASE` | `settings.py` | Database label for task metadata storage |
| `ENV_TYPE` | `settings.py` | Environment identifier; currency conversion only runs in `prod`; demo filtering logic depends on this |
| `UNIT_TEST_MODE` | `settings.py` | Bypasses demo filtering and currency conversion in tests |

---

## Notable Patterns

1. **Stored Procedure for Event Creation** — `CreateContextObject.create_context_obj()` bypasses Django ORM for event creation, calling `fn_create_analytics_event1()` with 21 typed parameters. Individual sub-objects are constructed in Python but never `.save()`'d — all persistence happens in the stored procedure. The returned ID is used to query back the full object graph via ORM.

2. **V1/V2 Coexistence with Silent Migration** — V2 views have been silently deployed on V1 URL paths (e.g., `v1/transaction-retailer-breakdown/` serves `TransactionByRetailerV2`). V1 view classes remain in code but are unreachable. The V2 views share URL names with V2 paths, causing Django URL name collisions (duplicate names like `transaction_retailer_breakdown_v2` for both `v1/` and `v2/` paths).

3. **Hardcoded API Key** — The Exchange Rate API key `24f8b7beb83be48d1eaa9ea4` is hardcoded in both `views.py:384` and `views_v2.py:103`. This should be in environment configuration.

4. **PaymentObject Custom Save** — `PaymentObject.save()` (`models.py:185`) looks up the Order by `order_id` to calculate `price_excluding_vat` from `basket.total_vat_price`. This creates a hidden dependency and potential `DoesNotExist` exception (caught with fallback to `Decimal(0.0)`).

5. **Manual Timestamp Control** — `PaymentObject` uses `auto_now_add=False` on `created`, allowing explicit timestamp setting. The CSV migration commands also use `update_or_create` patterns to override `auto_now_add` timestamps.

6. **Dramatiq Middleware as Task Tracker** — Rather than using Dramatiq's built-in result backend, `CustomAdminMiddleware` records every task state transition in a Django model. This provides admin visibility and allows manual run/cancel flags via the admin interface.

7. **Retry-by-Exception in Refund Task** — `check_refund_status_and_update_entities` raises a bare `raise` (no exception type) when the refund is still pending (HTTP 202) to trigger Dramatiq's retry mechanism. Similarly, `async_request` raises bare `raise` on non-OK responses.

---

## Open Questions

- How and when is `AggregatedAnalytics` populated? No aggregation management command, Celery task, or signal was found in the `mishipay_retail_analytics` module. The data must be populated by an external process or a command in another module.
- The `AnalyticsDashboard` view (V1, `views.py:80`) has no `permission_classes`. Is this intentional, or should it have `DashboardPermission`? It requires `analytics_filter_type` in the query params but doesn't verify the caller has access to the requested stores.
- `ItemScannedAnalyticsV2` exists in `views_v2.py` but is not wired into any URL. Is there a plan to add it, or does V1 `ItemScannedAnalytics` remain the preferred endpoint?
- The `BUNDLE_ID_MAPPPING` variable name typo (double `P`) is repeated across all 5 CSV migration commands. Is this known?
- `PaymentObjectClass` (`payment_object.py:22`) uses `datetime.now()` instead of `django.utils.timezone.now()`. This produces naive datetime objects when `USE_TZ=True` is set. Is this intentional?
- The `JourneyTypes.NOT_AVAILABLE` constant has a trailing comma making it a tuple `('not_available',)`. The model definition works around this by using inline choices. Is the constant intended to be used elsewhere?
- How are the `manual_run` and `manual_cancel` fields on the `Task` model used? No code was found that reads these flags to trigger task re-execution or cancellation.
- The `count_words` and `SampleTask` in `tasks.py` appear to be development/testing artifacts. Are they used in any environment?
- Is the hardcoded Exchange Rate API key (`24f8b7beb83be48d1eaa9ea4`) a free-tier key with rate limits? If the API is unavailable, currency conversion silently falls back to 1:1 with a Slack alert.
- `RecommendedItems` model has FKs to `Store` and `Basket` but no builder class, no views, and no admin registration (unlike other analytics models). How is it populated?

---

## Legacy Analytics Module (`analytics`)

> Added by Task T29

The `analytics` Django app is the **original event tracking system**, predating `mishipay_retail_analytics`. It provides raw app event logging, daily sales summaries, and push notification management. The newer `mishipay_retail_analytics` module (documented above) is the current analytics engine; this module appears to serve legacy purposes and specific Saturn/store notification use cases.

### Data Models (5 models)

| Model | File | Key Fields | Purpose |
|-------|------|------------|---------|
| `AppEvents` | `analytics/models.py:22` | `event_name` (CharField 255), `customer_id` (CharField 127), `ip_address` (GenericIPAddress), `timestamp` (DateTime), `store_id` (CharField 127), `latitude`/`longitude` (Float), `unique_session_id` (CharField 31), `bundle_id` (CharField 127), `user_agent_string` (Text), `params` (JSON, default list), `network_type` (CharField 127), `device_details` (JSON, default list), `request_path` (CharField 2047), `app_version`/`bundle_version` (CharField 255) | Generic app event log — captures any named event with full request context |
| `SaturnSalesSummary` | `analytics/models.py:46` | `date` (DateField, unique), 28 integer/float fields covering: `footfall`, `total_transactions`, `transaction_total`, `average_transaction_value`, per-platform breakdowns for transactions/items/scans/downloads/signups/repeat users/session durations, `refunded_count`, `paypal_count`, `payunity_count` | Daily Saturn-specific sales rollup. Unique per date — single-store. |
| `SalesSummary` | `analytics/models.py:83` | `date` (DateField), `store_id` (CharField 63), same per-platform breakdown fields as Saturn plus `payment_type_count` (JSON dict) | Multi-store daily sales rollup. Unique on `(date, store_id)`. |
| `Notification` | `analytics/models.py:115` | `device_id` (TextField), `user` (FK auth.User, CASCADE), `notification_id` (CharField 31), `notification_sent` (Boolean), `sent_time` (DateTime, nullable), `device_type` (CharField 31, choices: APNS/FCM), `message` (Text), `link` (URL), `params` (JSON, default list) | Push notification records per device per user |
| `StoreNotification` | `analytics/models.py:127` | `message_title` (Text), `message_body` (Text, max 1024), `message_image` (URL, nullable), `store_id` (FK Store, CASCADE), `analytics_label` (CharField 50, regex-validated `^[a-zA-Z0-9-_.~%]{1,50}$`), `is_active` (Boolean), `click_action_android`/`click_action_iOS`/`click_action_web` (Text, nullable), `trigger` (CharField 50, choices: `Item Purchase`), `payload_data` (JSON, nullable) | Store-scoped notification configuration. Indexed on `(store_id, trigger, is_active)`. |

### API Endpoints

| Path | View | Method | Purpose |
|------|------|--------|---------|
| `v1/record_event/` | `EventRecordView` | POST | Records an event via `analytics_library.EventInterface`. Accepts `event_record_platform` (e.g., `"mixpanel"`), `event_type` (e.g., `"record_login"`), and `params` (JSON string). Dispatches to the configured third-party analytics platform. |

**`EventRecordView` flow** (`analytics/views/events.py:15`):
1. Parses `params` JSON from POST data
2. Extracts `store_type` from params (defaults to `mishipaystoretype`)
3. Looks up Mixpanel token via `settings.MIXPANEL_TOKEN[store_type]`
4. Creates `EventInterface(via=event_record_platform, store_type=store_type)`
5. Calls `connector_obj.event_handler(parameters, event_type)`
6. Returns success/failure response

### Admin Configuration

| Model | List Display | Notes |
|-------|-------------|-------|
| `AppEvents` | `id`, `event_name`, `latitude`, `longitude`, `bundle_id`, `request_path`, `timestamp` | — |
| `SaturnSalesSummary` | `id`, `date`, `total_transactions`, `transaction_total`, `average_transaction_value` | — |
| `SalesSummary` | `id`, `date`, `total_transactions`, `transaction_total`, `average_transaction_value`, `store_id` | — |
| `Notification` | `id`, `notification_id`, `user`, `device_type` | `raw_id_fields`: `user` |
| `StoreNotification` | `message_title`, `message_body`, `store_id`, `is_active` | Search on all display fields; `raw_id_fields`: `store_id` |

### Migrations

20 migrations spanning 2018–2022 (`0001_initial` through `0020`).

---

## Analytics Library (`analytics_library`)

> Added by Task T29

A Mixpanel integration library that provides the event dispatch mechanism for the legacy `analytics` module. Contains the client connector and 15 event type definitions.

### Architecture

```
EventRecordView (analytics/views/events.py)
    |
    v
EventInterface (analytics_library/connect_with_3rd_party.py)
    |
    |── Mixpanel (via mixpanel + mixpanel_async libraries)
    |   Uses: settings.MIXPANEL_TOKEN[store_type][ENV_TYPE]['python_token']
    |   Consumer: AsyncBufferedConsumer (batched async sends)
    |
    |── DummyEventInterface (no-op logger, used when network disabled)
    |
    v
EventType (analytics_library/mixpanel.py)
    |── 15 event recording methods
    |── Each tracks to Mixpanel with customer_id + event_name + params
```

### EventInterface (`connect_with_3rd_party.py`)

| Method | Purpose |
|--------|---------|
| `__init__(via, store_type)` | Initializes Mixpanel client. Only supports `via="mixpanel"`. Two store types: `mishipaystoretype` (default, used for all non-Decathlon) and `decathlonstoretype`. Uses `AsyncBufferedConsumer` for batched sending. |
| `allow_network_events()` | Returns `True` in `test` env; otherwise checks `DISABLE_NETWORK_ANALYTIC_EVENTS` env var. When disabled, uses `DummyEventInterface` (logs only). |
| `event_handler(parameters, event_type)` | Dispatches to `EventType` method via `getattr`. Returns boolean success status. |

### EventType Methods (`mixpanel.py`)

| Method | Mixpanel Event Name | Key Params | Status |
|--------|-------------------|------------|--------|
| `record_login` | `user_auth_login` | screen, type, mish_id + common params | Active |
| `record_signup` | `user_auth_signup` | screen, type, mish_id + common params | Active |
| `record_item_scan` | `item_scan` | item_name, product_identifier, scan_status, epc + common params | Active |
| `record_item_delete` | `item_delete` | Empty string values for all fields | **Legacy** — sends no actual data |
| `record_item_delete_view` | `item_delete_view` | Empty string values for all fields | **Legacy** — sends no actual data |
| `record_create_order` | `create_order` | item_count, order_id + common params | Active |
| `record_checkout` | `checkout` | Empty string values | **Legacy** — sends no actual data |
| `record_payment_auth` | `payment_auth` | Empty string values | **Legacy** — sends no actual data |
| `record_auth_change` | `auth_change` | Empty string values | **Legacy** — sends no actual data |
| `record_payment_captured` | `payment_captured` | price, status, order_id, item_count, psp, payment_method + common params | Active |
| `record_saved_card_profile` | `saved_card_profile` | timestamp, bundle_id, store_specific_params | Active (different param format) |
| `record_app_open` | `app_open` | Common params only | Active |
| `record_past_purchases` | `past_purchases` | Empty string values | **Legacy** — sends no actual data |
| `record_rating` | `rating` | order_id, feedback + common params | Active |
| `record_shopping_session` | `shopping_session` | Empty string values | **Legacy** — sends no actual data |
| `record_refund_order` | `refund_order` | order_id, refund_amount, item_count, refund_type, refund_status + common params | Active |
| `record_verified_order` | `verified_order` | order_id, item_count, price, order_status + common params | Active |
| `record_journey_scan` | `journey_scan` | status, journey_type, unique_identifier + common params | Active |

**Common params** (from `get_common_params()`): `timestamp`, `store_id`, `store_type`, `retailer`, `store_region`, `store_sub_region`, `device`, `mobile_platform`, `user_agent`, `bundle_id`, `latitude`, `longitude`, `outside_geofencing`, `store_status`.

### Relationship to `mishipay_retail_analytics`

The legacy `analytics` + `analytics_library` system and the newer `mishipay_retail_analytics` system coexist:

| Aspect | Legacy (`analytics` + `analytics_library`) | Current (`mishipay_retail_analytics`) |
|--------|---------------------------------------------|---------------------------------------|
| **Storage** | Mixpanel (external SaaS) + `AppEvents` table | PostgreSQL stored procedure + dedicated event models |
| **Event dispatch** | Via `EventRecordView` POST API → Mixpanel | Via `CreateContextObject` builder classes → stored procedure |
| **Sales summaries** | `SalesSummary` / `SaturnSalesSummary` (legacy rollup) | `AggregatedAnalytics` (current rollup) |
| **Dashboard** | Not used for dashboard | V2 dashboard API views query `AggregatedAnalytics` |
| **Notifications** | `Notification` / `StoreNotification` models | No notification models |

The legacy module's `EventRecordView` is still wired into URLs, suggesting it remains in use for Mixpanel event forwarding. The `SalesSummary` models may be populated by external processes or management commands not found in this module.

### Notable Patterns

1. **Dynamic method dispatch** — `event_handler()` uses `getattr(event_type_obj, event_type)` to call event methods by name. This means any method on `EventType` is callable via the API — no whitelist exists.

2. **Legacy placeholder methods** — 6 of 15 event methods (`record_item_delete`, `record_item_delete_view`, `record_checkout`, `record_payment_auth`, `record_auth_change`, `record_past_purchases`, `record_shopping_session`) send empty string values for all params. These appear to be early stubs that were never fully implemented.

3. **Store-type branching** — Only two Mixpanel projects exist: MishiPay (default) and Decathlon. All other store types use the MishiPay Mixpanel token.

4. **Async buffered sending** — Uses `mixpanel_async.AsyncBufferedConsumer` for batched event delivery to Mixpanel, reducing HTTP overhead.

### Open Questions (Legacy Analytics)

- Are the 6 legacy placeholder methods (`record_item_delete`, `record_checkout`, `record_payment_auth`, `record_auth_change`, `record_past_purchases`, `record_shopping_session`) still called from any client code, or are they dead code?
- Is Mixpanel still the primary external analytics platform, or has it been fully replaced by the internal `mishipay_retail_analytics` system?
- How are `SalesSummary` and `SaturnSalesSummary` populated? No management command, signal, or task was found in the `analytics` module.
- `EventRecordView` has no authentication/permissions — is it protected at the API gateway level?
- The `StoreNotification` model has a `trigger` field with only one choice (`Item Purchase`). Are additional trigger types planned?
- `DummyEventInterface` only has a `track()` method, but `EventInterface.event_handler()` calls methods on `EventType` which wraps the object. The dummy's `track()` signature differs from `EventType`'s methods — does this cause issues when network events are disabled?
