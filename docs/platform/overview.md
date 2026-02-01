# MishiPay Platform — Project Structure & Settings

> Last updated by: Task T06 — Project Structure & Settings

## Overview

The MishiPay platform is a Django 4.1.13 monolith (project name `mainServer`) that serves as the core backend for MishiPay's scan-and-go retail solution. It contains 48+ Django apps handling items, orders, payments, customers, analytics, retailer integrations, and admin dashboards. The project is managed with Poetry and deployed as a Docker container (`backend_v2`) behind a Kong API Gateway.

## Project Structure

```
MishiPay/mainServer/
├── manage.py                      # Standard Django management entry point
├── permissions.py                 # Top-level shared permission classes
├── conftest.py                    # Global pytest fixtures
├── _test.py                       # Ad-hoc Mango analytics script (not a test suite)
├── pyproject.toml                 # Poetry config, dependencies, tool settings
├── setup.cfg                      # Coverage, flake8, mypy config
├── config/                        # Email configuration subsystem
│   ├── __init__.py                # Exports CONFIG_PATH (defaults to config/data/)
│   ├── loader.py                  # Instantiates EmailConfig from emails.json
│   ├── commons/emails/            # Email config classes and helper
│   └── data/emails.json           # Default email provider settings (JSON)
├── mainServer/                    # Django project package
│   ├── __init__.py
│   ├── settings.py                # Main settings (~3,007 lines)
│   ├── urls.py                    # Root URL configuration
│   ├── middleware.py              # Custom middleware classes (4 classes)
│   ├── constants.py               # Global constants (payment methods, store types, etc.)
│   ├── wsgi.py                    # WSGI application entry point
│   ├── ingenico.conf              # Ingenico/GlobalCollect SDK config
│   ├── locale/                    # 14 translation directories (ar, da, de, el, es, eu, fi, fr, he, it, nb, no, pt, sv)
│   └── retailer_configurations/   # Per-retailer settings (~49 retailer directories)
├── mishipay_core/                 # Core shared module
├── mishipay_items/                # Product catalog (largest: 649 files)
├── mishipay_retail_orders/        # Order management (223 files)
├── mishipay_retail_payments/      # Payment processing (259 files)
├── dos/                           # DOS retailer integration (363 files)
├── mishipay_emails/               # Email system (136 files)
├── inventory/                     # Inventory management (119 files)
├── ... (48+ Django apps total)
└── lib/                           # Shared library (78 files, added to sys.path)
```

## Django Settings

**File:** `mainServer/mainServer/settings.py` (3,007 lines)

The settings file is exceptionally large because it contains not only Django configuration but also extensive retailer-specific configuration data (payment gateways, RFID endpoints, store details, email templates).

### Core Settings

| Setting | Value | Source |
|---------|-------|--------|
| `BASE_DIR` | Parent of `mainServer/` package | Computed |
| `ENV_TYPE` | `DEPLOYMENT_ENV` env var | Default: `"test"` |
| `SECRET_KEY` | `MPAY_SECRET_KEY` env var | Default: `"localhost"` |
| `DEBUG` | `DEBUG` env var | Default: `False` |
| `ALLOWED_HOSTS` | Comma-separated from env | Default: `"localhost"` + `THIS_HOST_IP` + `POD_IP` |
| `ROOT_URLCONF` | `mainServer.urls` | Hardcoded |
| `WSGI_APPLICATION` | `mainServer.wsgi.application` | Hardcoded |
| `SITE_ID` | `1` | Hardcoded |
| `LANGUAGE_CODE` | `en-us` | Hardcoded |
| `TIME_ZONE` | `UTC` | Hardcoded |
| `DATA_UPLOAD_MAX_NUMBER_FIELDS` | `30000` | Raised to handle large forms |

### Databases

Two PostgreSQL databases are configured:

| Alias | Purpose | Default Host | Default DB Name |
|-------|---------|-------------|-----------------|
| `default` | Primary read-write | `backend_db_v2` | `mpay` |
| `read_replica` | Read-only replica | `backend_db_v2` | `mpay` |

Both use `django.db.backends.postgresql` with `CONN_MAX_AGE=120` (persistent connections). Database credentials come from environment variables (`MPAY_DB_HOST`, `MPAY_DB_NAME`, `MPAY_DB_USER`, `MPAY_DB_PWD` for primary; `READ_REPLICA_*` for replica).

### Caching

The cache configuration has a quirk — it conditionally sets Memcached for `dev`/`test`/`prod` environments, but then **unconditionally overrides** it with `LocMemCache` on `settings.py:599-604`. This means Memcached is effectively never used (likely a debugging override left in place).

```python
# Line 584-597: Conditional Memcached setup (for dev/test/prod)
CACHES = { 'default': { 'BACKEND': '...PyMemcacheCache', 'LOCATION': f"{MEMCACHED_HOST}:{MEMCACHED_PORT}" } }

# Line 599-604: Unconditional override (always wins)
CACHES = { 'default': { 'BACKEND': '...LocMemCache', 'LOCATION': 'django-cache-local' } }
```

| Service | Config | Environment Variable |
|---------|--------|---------------------|
| Memcached | `MEMCACHED_HOST:MEMCACHED_PORT` | Default: `memcached_v2:11211` |
| Redis | `MPAY_REDIS_HOST:6379` | Default: `mpay_redis` (used by Dramatiq results backend) |

### Task Queue (Dramatiq)

| Setting | Value |
|---------|-------|
| Broker | `dramatiq.brokers.rabbitmq.RabbitmqBroker` |
| Broker URL | `MPAY_DRAMATIQ_BROKER` env var (default: `amqp://mpay_broker:...@mpay_rabbitmq:5672/mpay`) |
| Result Backend | `dramatiq.results.backends.redis.RedisBackend` (Redis) |
| Result TTL | 60,000 ms |
| Database | `default` |

Dramatiq middleware chain:
1. `AgeLimit` — expires old jobs
2. `TimeLimit` — cancels long-running jobs
3. `Callbacks` — `on_success` / `on_failure` hooks
4. `Retries` — retry with backoff
5. `CustomAdminMiddleware` (`mishipay_retail_jobs`) — custom DB entry management
6. `DbConnectionsMiddleware` (`django_dramatiq`) — manages database connections

In test mode, Dramatiq uses `StubBroker` instead of RabbitMQ.

### Static Files

| Setting | Value |
|---------|-------|
| `STATIC_URL` | `/static/` |
| `STATIC_ROOT` | `/app/static/` (from env `STATIC_ROOT_DIR_PATH`) |
| `STATICFILES_STORAGE` | `whitenoise.storage.CompressedStaticFilesStorage` |

WhiteNoise serves static files directly from the Django process in production, avoiding the need for a separate NGINX static file server.

## Installed Apps (Categorized)

### Django Built-in (7)

- `django.contrib.sessions`
- `django.contrib.admin`
- `django.contrib.auth`
- `django.contrib.contenttypes`
- `django.contrib.messages`
- `django.contrib.staticfiles`
- `django.contrib.sites`

### Third-Party Libraries (21)

| App | Purpose |
|-----|---------|
| `django_extensions` | Management command extensions |
| `crispy_forms` | Form rendering |
| `rest_framework` | Django REST Framework (API layer) |
| `rest_framework.authtoken` | Token-based authentication |
| `corsheaders` | CORS handling |
| `django_dramatiq` | Dramatiq task queue integration |
| `django_user_agents` | User agent parsing middleware |
| `push_notifications` | Firebase push notifications |
| `social_django` | Social auth (Facebook, Google) |
| `explorer` | Django SQL Explorer (ad-hoc SQL queries) |
| `django_cron` | Cron job framework |
| `import_export` | Data import/export (CSV, Excel) |
| `drf_yasg` | Swagger/OpenAPI docs |
| `mptt` | Modified Preorder Tree Traversal (tree structures) |
| `cloudinary` | Image hosting |
| `nested_inline` | Nested inline admin forms |
| `health_check` + `.db` + `.contrib.migrations` | Health check endpoints |
| `django_filters` | Queryset filtering for DRF |
| `phonenumber_field` | Phone number field type |
| `multiselectfield` | Multi-select model field |
| `django_migration_linter` | Migration safety checks |
| `rangefilter` | Admin date range filters |
| `django_guid` | Request correlation IDs |

### MishiPay Core Platform Apps (18)

| App | File Count | Description |
|-----|-----------|-------------|
| `mishipay_core` | — | Core module: shared models, views, utilities |
| `mishipay_items` | 649 | Product catalog, pricing, categories |
| `mishipay_retail_orders` | 223 | Order lifecycle management |
| `mishipay_retail_payments` | 259 | Payment processing, gateway integrations |
| `mishipay_retail_customers` | 77 | Customer profiles, loyalty |
| `mishipay_retail_analytics` | 64 | Analytics data collection |
| `mishipay_retail_jobs` | 14 | Background job management |
| `mishipay_dashboard` | 52 | Dashboard management API |
| `mishipay_cashier_kiosk` | 20 | POS kiosk functionality |
| `mishipay_referrals` | 11 | Referral program |
| `mishipay_coupons` | 14 | Coupon management |
| `mishipay_special_promos` | 13 | Special promotions |
| `mishipay_alerts` | 58 | Alert/notification system |
| `mishipay_subscription` | 37 | Subscription management |
| `mishipay_emails` | 136 | Email template system |
| `mishipay_config` | 15 | Platform configuration management |
| `mishipay_authentication` | 13 | Auth management |
| `mishipay` | — | Base MishiPay module |

### Feature & Integration Apps (14)

| App | Description |
|-----|-------------|
| `dos` (363 files) | DOS retailer integration (largest retailer module) |
| `dashboard` (50 files) | Admin dashboard UI |
| `analytics` (26 files) | Analytics module |
| `inventory` (119 files) | Inventory management (MongoDB integration) |
| `promotions` (82 files) | Platform-side promotions |
| `shopping_list` (21 files) | Shopping list feature |
| `pubsub` (24 files) | Kafka/RabbitMQ pub/sub messaging |
| `third_party_integrations` (33 files) | External service integrations |
| `easy_store_creation` (23 files) | Store onboarding wizard |
| `fiscalisation` (25 files) | Fiscal compliance / receipt generation |
| `ddf_price_activation` (20 files) | DDF pricing module |
| `admin_actions` (18 files) | Custom admin actions |
| `mainServer` | The Django project itself (registered for locale, etc.) |
| `rfid` (4 files) | RFID tag integration |

### Retailer-Specific Apps (8)

| App | Description |
|-----|-------------|
| `dixons` | Dixons Carphone integration |
| `saturn` | Saturn/Media Markt integration |
| `saturn_smart_pay` | Saturn SmartPay payment flow |
| `mango` | Mango fashion retailer |
| `leroy_merlin` | Leroy Merlin DIY retailer |
| `cisco` | Cisco store (RFID, Clover POS) |
| `lego` | LEGO store integration |
| `decathlon` | Decathlon sporting goods |

### Conditional Apps

- `ddtrace.contrib.django` — Added when `ENV_TYPE` is `dev`, `test`, or `prod` (Datadog APM tracing)
- `elasticapm.contrib.django` — Added when `ELASTIC_APM_ENABLED=True`

## Middleware Chain

**File:** `mainServer/mainServer/middleware.py` (327 lines) + `settings.py:233-247`

The middleware chain executes in this order (top = outermost):

| # | Middleware | Source | Purpose |
|---|-----------|--------|---------|
| 1 | `GuidMiddleware` | `django_guid` | Assigns correlation ID to each request for log tracing |
| 2 | `UserAgentMiddleware` | `django_user_agents` | Parses `User-Agent` header, attaches to request |
| 3 | `SentryInterceptMiddleware` | `mainServer.middleware` | Logs 400-level responses; **blocks Decathlon `bundleid` requests** (returns 403) |
| 4 | `CorsMiddleware` | `corsheaders` | Handles CORS headers |
| 5 | `WhiteNoiseMiddleware` | `whitenoise` | Serves static files |
| 6 | `SessionMiddleware` | Django | Session handling |
| 7 | `SecurityMiddleware` | Django | HTTPS redirects, security headers |
| 8 | `ProcessStatsMiddleware` | `mainServer.middleware` | Records response latency to Datadog for specific routes |
| 9 | `CommonMiddleware` | Django | URL normalization, `Content-Length` |
| 10 | `AuthenticationMiddleware` | Django | Attaches `request.user` from session |
| 11 | `MessageMiddleware` | Django | Flash messages |
| 12 | `StartupMiddleware` | `mainServer.middleware` | i18n/timezone activation, store lookup, `x-authorization` header swap |
| 13 | `LoggingUserIdMiddleware` | `mishipay_core.logging` | Adds user ID to log context |

**Conditional middleware:**
- `TracingMiddleware` (`elasticapm`) — Appended when `ELASTIC_APM_ENABLED=True`

### Custom Middleware Details

#### StartupMiddleware (`middleware.py:28-93`)

Runs as a `process_view` hook. Key behavior:
1. **Header swap**: Copies `x-authorization` header to standard `authorization` header (allows API gateway to pass auth tokens via a custom header)
2. **Store resolution**: Looks up `Store` by `store_id` query param or URL `pk` kwarg; returns `403 Forbidden` if store is inactive; attaches `request.store`
3. **Language activation**: Resolves i18n language from `mpay_lng`/`user_language` params, POST data, GET params, or the authenticated customer's `user_language`
4. **Timezone activation**: Sets timezone from the resolved store's `timezone` field

#### SentryInterceptMiddleware (`middleware.py:95-135`)

Despite the name, this middleware does **not** use the Sentry SDK. It:
1. **Blocks Decathlon**: Returns `403 Forbidden` for any request where the `bundleid` header contains `"decathlon"` (line 108)
2. **Logs 400-level responses**: Captures responses with `400 <= status < 500` or responses that return `200` but contain `{ "status": 400-500 }` in the response data
3. Skips logging for `403` and `500+` responses

#### MishiKeycloakAuthentication (`middleware.py:138-252`)

Not actually a middleware — it's a DRF `BaseAuthentication` class (configured in `REST_FRAMEWORK.DEFAULT_AUTHENTICATION_CLASSES`). Authentication flow:

1. Requires `x-mishipay-id` header; returns `(None, None)` if absent
2. If `x-anonymous-consumer: "true"` header is set and a Bearer token is present, raises `AuthenticationFailed` (Kong auth failure)
3. **Authenticated user path**: Decodes JWT (RS256, **signature not verified** — delegated to Kong), extracts `given_name`, `last_name`, `email`, `user_id`; looks up Django `User` by email
4. **Guest user path**: Looks up `Customer` by `mishipay_id` + `is_guest_user=True`; if not found, creates a new Django `User` and `Customer` with a random email (`{random}@guestuseremail.com`)
5. Handles `MultipleObjectsReturned` by deactivating all but the first matching customer

**Security note:** JWT signature verification is disabled (`"verify_signature": False` at line 179). The comment states this is delegated to Kong API Gateway.

#### ProcessStatsMiddleware (`middleware.py:255-327`)

Records response latency for a specific set of monitored routes (16 routes including item scan, basket operations, order creation, payment session management) as Datadog histograms (`mishipay.monolith_service.http.request.latency.ms`). Only records for `200`/`201` status codes (except DubaiDutyFree which records all).

## URL Routing Structure

**File:** `mainServer/mainServer/urls.py` (168 lines)

The root URL configuration maps to 30+ app URL patterns. Root path (`/`) redirects to `/dashboard/`.

### Application Routes

| Path Prefix | App / Module | Description |
|-------------|-------------|-------------|
| `d/` | `dos.urls` | DOS retailer endpoints |
| `admin/` | Django admin | Admin interface |
| `dashboard/` | `dashboard.urls` | Admin dashboard (namespace: `dashboard`) |
| `analytics/` | `analytics.urls` | Analytics endpoints |
| `third-party-integrations/` | `third_party_integrations.urls` | Third-party service endpoints |
| `rfid/` | `rfid.urls` | RFID tag management |
| `admin/explorer/` | `explorer.urls` | SQL Explorer (ad-hoc queries) |
| `auth-management/` | `mishipay_authentication.urls` | Authentication management |
| `item-management/` | `mishipay_items.urls` | Product/item APIs |
| `customer-management/` | `mishipay_retail_customers.urls` | Customer APIs |
| `order-management/` | `mishipay_retail_orders.urls` | Order APIs |
| `payment-management/` | `mishipay_retail_payments.urls` | Payment APIs |
| `dashboard-management/` | `mishipay_dashboard.urls` | Dashboard management APIs |
| `referral-management/` | `mishipay_referrals.urls` | Referral program APIs |
| `coupon-management/` | `mishipay_coupons.urls` | Coupon APIs |
| `special-promo-management/` | `mishipay_special_promos.urls` | Special promotions APIs |
| `mishipay-subscription/` | `mishipay_subscription.urls` | Subscription APIs |
| `staff-shifts/` | `mishipay_cashier_kiosk.urls` | Cashier/kiosk staff APIs |
| `core-services/` | `mishipay_core.urls` | Core platform services |
| `mishipay-dashboard-analytics/` | `mishipay_retail_analytics.urls` | Dashboard analytics APIs |
| `admin-actions/` | `admin_actions.urls` | Custom admin actions |
| `inventory/` | `inventory.urls` | Inventory management (namespace: `inventory`) |
| `fiscalisation/` | `fiscalisation.urls` | Fiscal compliance (namespace: `fiscalisation`) |
| `easy-store-creation/` | `easy_store_creation.urls` | Store onboarding (namespace: `easy-store-creation`) |
| `alerts/` | `mishipay_alerts.urls` | Alert/notification endpoints |
| `shopping-list/` | `shopping_list.urls` | Shopping list (namespace: `shopping-list`) |
| `api/v1/` | `shopping_list` v1 router | **DEPRECATED** — legacy shopping list compatibility |
| `mishipay-config/` | `mishipay_config.urls` | Platform configuration |
| `ddf-price-activation/` | `ddf_price_activation.urls` | DDF price activation (namespace: `ddf-price-activation`) |

### Special Routes

| Path | Handler | Purpose |
|------|---------|---------|
| `/` | `redirect_to_dashboard` | Redirects to `/dashboard/` |
| `/theme/` | `ThemeAPIView` | Store theming API |
| `/apple-app-site-association/` | `DeepLinkIOSView` | iOS Universal Links config |
| `/get-additional-store-details/` | `AdditionalStoreDetails` | Extra store metadata |
| `/health-check` | `HealthCheck.as_view()` | Health check endpoint |

### Client Configuration Routes

Static-file-style routes that serve dynamic JSON config to mobile/web clients:

| Pattern | Handler | Purpose |
|---------|---------|---------|
| `/static/mishipay_static_assets/master_static_conf.json` | `client_config.master_conf` | Master client configuration |
| `/static/mishipay_static_assets/master_static_conf.v1.0.json` | `client_config.master_conf` | Versioned master config |
| `/static/mishipay_static_assets/all_store_details.{platform}.v{M}.{m}.json` | `client_config.all_stores` | Per-platform store list |
| `/static/mishipay_static_assets/app_force_update.v{M}.{m}.json` | `client_config.force_update` | App force update rules |
| `/static/mishipay_static_assets/hardcoded_items.v{M}.{m}.json` | `client_config.hardcoded_items` | Hardcoded item data |
| `/client-config/v1/theme/{store_id}/` | `client_config.theme` | Per-store theme config |
| `/client-config/v1/harcoded/{store_id}/` | `client_config.harcoded` | Per-store hardcoded config (note: typo `harcoded` in URL) |

### Internal System APIs

| Path | Handler | Purpose |
|------|---------|---------|
| `/system-api/v1/stores/` | `system_api.stores` | Internal store listing |
| `/system-api/v1/i18n-check/` | `system_api.timezone_check` | Timezone validation |

### API Documentation

| Path | Format |
|------|--------|
| `/openapi.json` or `/openapi.yaml` | Raw OpenAPI schema |
| `/swagger-ui/` | Swagger UI interactive docs |

### Debug-Only Routes (when `DEBUG=True`)

- `/static/{path}` — Django staticfiles serving
- `/__debug__/` — Django Debug Toolbar

### Admin Customization

```python
admin.site.site_header = "MishiPay Admin"
admin.site.site_title = "MishiPay Admin Portal"
admin.site.index_title = "Welcome to MishiPay Admin"
```

## REST Framework Configuration

**File:** `settings.py:306-326`

| Setting | Value |
|---------|-------|
| Default Authentication | `TokenAuthentication`, `MishiKeycloakAuthentication` |
| Default Permission | `AllowAny` (permissive default; endpoints must set stricter permissions) |
| Default Filter Backend | `DjangoFilterBackend` |
| Throttling | `ScopedRateThrottle` with `otp` scope at `10/min` |
| Exception Handler | `mishipay_core.exception.custom_exception_handler` |

## Authentication Configuration

### Authentication Backends

1. `django.contrib.auth.backends.ModelBackend` — Standard Django auth
2. `social_core.backends.facebook.FacebookOAuth2` — Facebook login
3. `social_core.backends.google.GoogleOAuth2` — Google login

### Social Auth Pipeline

Standard `python-social-auth` pipeline: `social_details` -> `social_uid` -> `auth_allowed` -> `social_user` -> `get_username` -> `create_user` -> `associate_user` -> `load_extra_data` -> `user_details`. Note: `associate_by_email` is commented out.

### CORS Configuration

- `CORS_ORIGIN_ALLOW_ALL = False`
- Whitelist from `CORS_ORIGIN_WHITELIST` env var (default: `http://localhost:8080`)
- Custom headers allowed: `bundleid`, `uniquesessionid`, `x-authorization`, `appversion`, `bundleversion`, `x-mishipay-id`, `x-api-key`, `x-anonymous-consumer`, `x-logrocket-session`, Datadog trace headers

## Microservice Connections

The monolith connects to several internal microservices:

| Service | Protocol | Default Host:Port | Env Vars | Purpose |
|---------|----------|-------------------|----------|---------|
| Promotions (`ms-promo`) | HTTP | `ms-promo:8000` | `MS_PROMO_HOST`, `MS_PROMO_PORT` | Promotion evaluation |
| Loyalty (`ms-loyalty`) | HTTP | `ms-loyalty:8000` | `MS_LOYALTY_HOST`, `MS_LOYALTY_PORT` | Loyalty program |
| Tax (gRPC) | gRPC | `ms-tax:9000` | `MS_TAX_HOST`, `MS_TAX_PORT` | Tax calculation (legacy gRPC) |
| Tax (HTTP) | HTTP | `ms_tax:80` | `MS_TAX_NEW_HOST`, `MS_TAX_NEW_PORT` | Tax calculation (new HTTP) |
| Payments (`ms-pay`) | HTTP | `pay:8003` | `MS_PAY_HOST`, `MS_PAY_PORT` | Payment orchestration |
| Inventory | HTTP | `inventory:8000` | `INVENTORY_SERVICE_HOST` | Inventory management |
| Inventory (offline) | HTTP | `inventory:8000` | `INVENTORY_OFFLINE_SERVICE_OPS_HOST` | Batch inventory jobs |
| Voucher | HTTP | `voucher` (hardcoded) | — | Coupon validation |
| Hudson | HTTP | From env | `HUDSON_SERVER_HOST` | Hudson retail integration |

The `ms-pay` service exposes 20+ URL endpoints configured in settings (payment sessions, refunds, saved cards, payouts, subscriptions, gift cards, device status, etc.).

## Logging Configuration

**File:** `settings.py:417-578`

Uses a custom logging library (`mishipay-python-logger`, private GitHub package) with multiple handlers:

### Log Filters

| Filter | Purpose |
|--------|---------|
| `correlation_id` | Injects `django_guid` correlation ID |
| `require_debug_false` / `require_debug_true` | Standard Django debug filters |
| `requires_local_env` | Only passes in `local`/`unittest` environments |
| `requires_prod_env` | Only passes in `prod` environment |
| `redact_sensitive_information` | Redacts sensitive patterns (patterns list is empty — placeholder) |

### Log Handlers

| Handler | Environment | Purpose |
|---------|-------------|---------|
| `datadog_json_logs_handler` | All | JSON-formatted logs for Datadog ingestion |
| `db_logs_handler` | Local/unittest only | Writes DB query logs to file |
| `error_logs_stream_handler` | All | Error-level stream output |
| `requests_logs_handler` | All | HTTP request logs to file |
| `security_logs_handler` | Prod only | Security event logs to file |
| `local_standard_logs_stream_handler` | Local/unittest | Standard stream output (local dev) |
| `prod_standard_logs_stream_handler` | Prod | Standard stream output (production) |

### Logger Configuration

| Logger | Handlers | Level |
|--------|----------|-------|
| Root (`''`) | prod_standard, local_standard, error, datadog | `ROOT_LOG_LEVEL` env (default: `INFO`) |
| `django` | prod_standard, local_standard, datadog | `DJANGO_LOG_LEVEL` env (default: `INFO`) |
| `django.server` | requests_logs | Propagates to root |
| `django.security.DisallowedHost` | prod_standard, local_standard, datadog | `WARNING` |
| `django.security.*` | security_logs, datadog | `CRITICAL` |
| `django.db.backends` | db_logs, datadog | `DEBUG` |
| `django_guid` | prod_standard, local_standard, datadog | `WARNING` |

## Monitoring & APM

### Datadog

- `ddtrace` installed for `dev`/`test`/`prod` environments
- `ProcessStatsMiddleware` sends latency histograms for 16 critical routes
- Tags: `retailer`, `env`, `resource` (method + route), `status_code`
- Metric name: `mishipay.monolith_service.http.request.latency.ms`

### Elastic APM (optional)

Enabled via `ELASTIC_APM_ENABLED=True`. Service name: `"monolith-service"`.

### Sentry (optional)

Enabled via `SENTRY_ENABLED=True`. Uses `DjangoIntegration` and `LoggingIntegration` (event level: `CRITICAL`). Max string length raised to `10240`.

## Retailer Configuration System

**Directory:** `mainServer/mainServer/retailer_configurations/` (49 retailer directories)

Each retailer has a directory with `demo_configuration.py` and `live_configuration.py` files containing retailer-specific settings dictionaries. Example structure (from `eroski/`):

```python
ORDER_SETTINGS = { 'common': { 'qr_identifier_type': 'S&G', 'version_of_qr': '01' } }
PRODUCT_SETTINGS = { 'common': { 'host': '...', 'authSource': '...', 'username': '...', ... } }
CUSTOMER_SETTINGS = { 'common': { 'loyalty_service_url': '...', ... } }
```

**Retailers with configurations (49):** avery_dennison, badiani, bestseller, bloomingwear, breeze_thru, charlotte, compass, decathlon, decathlon_il, default_configuration, desigual, dimona, dufry, emetro, emmasgarden, emmasleep, eroski, eventnetwork, expresslane, fast_eddys, flying_tiger, goloso, hmshost, hudson, londis, lovbjerg, loves, mishipay, mishipay_promotion, muji, nedap, nude_foods, pantree, paradies, picwic, reitan, relay, retail_reload, rfidstandard, saturn, spar, standard, standard_cafe, stellar, sto, univision, virgin, viva.

Additionally, `settings.py` contains large inline configuration dictionaries for specific retailers:
- `STORE_DETAILS` — Adyen/payment settings by retailer and region (mishipay, nedap, mango, saturn, etc.)
- `RFID_UPDATE_SETTINGS` — RFID transaction URLs for RetailReload, Nedap, Cisco
- `CLOVER_SETTINGS` — Clover POS configuration for Cisco stores
- `DECATHLON_STORE_DETAILS` — Decathlon API endpoints by country (NL, IL, DE)
- `EMAIL_RECEIPT_SETTINGS` — Per-store-type email template, from address, and subject (20+ store types with locale variants)

## Global Constants

**File:** `mainServer/mainServer/constants.py` (121 lines)

| Category | Constants |
|----------|-----------|
| Refund types | `FULL_REFUND`, `PARTIAL_REFUND` |
| Payment methods (13) | `CREDIT_CARD`, `PAYPAL`, `IDEAL`, `STRIPE`, `JUDO_PAY`, `VERIFONE`, `ADYEN`, `PAYUNITY`, `GOOGLE_PAY`, `APPLE_PAY`, `XPAY`, `WORLD_PAY`, `ONEPAY` |
| Platforms | `ANDROID`, `IOS`, `WEB` |
| Order statuses (Mango) | `AUTHORIZED`, `CAPTURED`, `CANCELLED`, `PENDING_AUTH`, `AUTH_DECLINED` |
| Payment statuses | `PAYMENT_PROCESSING`, `PAYMENT_COMPLETED`, `PAYMENT_FAILED`, `PAYMENT_VERIFIED` |
| Order verification | `ORDER_CORRECT`, `ORDER_INCORRECT`, `ORDER_NOT_VERIFIED` |
| Store types | `MISHIPAY_STORE_TYPE`, `MANGO_STORE_TYPE`, `DIRECTWINES_STORE_TYPE`, `NEDAP_STORE_TYPE` |
| Locations | `GLOBAL`, `LONDON` (lat/long/radius), `BARCELONA` (lat/long/radius) |
| Store IDs | `NEDAP_STORES` (2 UUIDs), `SATO_STORES` (1 UUID), `VAT_RECEIPT_STORE_IDS` |
| Retailer names | `DECATHLON`, `MISHIPAY`, `NEDAP`, `SATURN` |
| Bundle ID mapping | Maps app bundle IDs to platforms (`ios`/`android`/`web`) |
| Return policy | `RETURN_POLICY_DAYS` (60 days for `mlsestoretype`, `flyingtigerstoretype`) |
| `BUNDLE_ID_MAPPPING` | 7 entries mapping bundle IDs to platform names |

## Top-Level Permissions

**File:** `mainServer/permissions.py` (7 lines)

A single permission class:

```python
class ReadOnly(permissions.BasePermission):
    def has_permission(self, request, view):
        return request.method in permissions.SAFE_METHODS
```

Allows only `GET`, `HEAD`, `OPTIONS` requests. Used by views that should be publicly readable but not writable.

## Email Configuration System

**Files:** `config/loader.py`, `config/commons/emails/email_config.py`, `config/commons/emails/helper.py`, `config/data/emails.json`

A multi-layered email configuration system with three possible sources (in priority order, controlled by settings flags):

1. **`FORCE_EMAIL_SETTINGS=True`** (current default) — Uses Django `settings.EMAIL_*` / `settings.COMMERCIAL_EMAIL_*` values directly
2. **`FORCE_EMAIL_JSON_SETTINGS=True`** — Reads from `config/data/emails.json`
3. **`FORCE_EMAIL_OS_SETTINGS=True`** — Reads from `email_*` / `commercial_email_*` environment variables

Five email configuration types: `environ`, `environ_commercial`, `default`, `error`, `transaction`. Each provides SMTP host, port, username, password, TLS settings, and a `get_connection()` method.

Current email provider: **Sendinblue** (`smtp-relay.sendinblue.com:587`). Commented-out SendGrid config is also present.

## Test Configuration

### pytest (from `pyproject.toml`)

```
--ds=mainServer.settings --reuse-db -n 2
```

- Uses `mainServer.settings` as the Django settings module
- Reuses the test database across runs
- Parallelism: 2 workers (`pytest-xdist`)
- Excludes: `lib/`, migrations, static, templates

### Global Fixtures (`conftest.py`)

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `enable_db_access` | Session, autouse | Enables DB access for all tests |
| `user` | Function | Creates a `User` via `UserFactory` |
| `customer` | Function | Creates a `Customer` linked to `user` |
| `store` | Function | Creates a `Store` via `StoreFactory` |
| `item_inventory` | Function | Creates an `ItemInventory` via `ItemInventoryFactory` |
| `login_client` | Function | Returns an authenticated `APIClient` (password: `"ChangeM3!"`) |

### Test-Mode Overrides (`settings.py:2989-3006`)

When running tests:
- Dramatiq broker → `StubBroker`
- Password hasher → `MD5PasswordHasher` (faster)
- Cache → `DummyCache`
- Email → `locmem.EmailBackend`
- Templates debug → `True`
- `DEBUG = True`

### Code Quality (from `setup.cfg` and `pyproject.toml`)

| Tool | Configuration |
|------|--------------|
| Coverage | Min 28%, excludes migrations/tests/static/admin/templates |
| Flake8 | Max line 100, max complexity 10, Google docstring convention |
| Black | Line length 88, target Python 3.6 |
| isort | Compatible with Black (profile 3, trailing comma) |
| mypy | Python 3.6, strict optional, ignore missing imports |

## Miscellaneous Files

### `_test.py`

An ad-hoc interactive script (not part of the test suite) for querying Mango Fast Pay order analytics by date range and store (London/Barcelona). Uses `dos.models.Order` to count payment method usage. Contains hardcoded store UUIDs.

### `ingenico.conf`

ConnectSDK configuration for Ingenico/GlobalCollect payment gateway sandbox:
- Integrator: `Mishipay`
- Endpoint: `api-sandbox.globalcollect.com`
- Auth: `v1HMAC`
- Timeouts: connect 5s, socket 300s

## Cron Jobs

**File:** `settings.py:2177-2182`

```python
CRON_CLASSES = [
    "jobs.order_crons.SendSaturnOrderMailCronJob",
    "jobs.order_crons.SendDecathlonOrderMailCronJob",
    "jobs.daily_analytics_crons.DecathlonDailyAnalyticsCronJob",
    "jobs.daily_analytics_crons.SaturnDailyAnalyticsCronJob",
]
```

## Dependencies

See [Tech Stack](../02-tech-stack.md) for full dependency inventory.

Key additions noted from `pyproject.toml` not previously documented:
- `anthropic` (^0.64.0), `langchain` (^0.3.27), `langchain-anthropic` (^0.3.19) — AI/LLM integration
- `python-magic` (^0.4.27) — File type detection
- `python-slugify` (^8.0.4) — URL slug generation
- `azure-storage-blob` (^12.21.0), `azure-identity` (^1.17.1) — Azure blob storage
- `better-profanity` (^0.7.0) — Profanity filtering
- `psutil` (^6.1.0) — System resource monitoring
- `gspread` (^5.4.0) — Google Sheets API
- `uttlv` (^0.7.0) — TLV (Tag-Length-Value) data encoding
- `pyodbc` (^4.0.39), `requests-ntlm` (^1.2.0) — MSSQL/NTLM connectivity (likely for retailer ERP integrations)

## Notable Patterns

1. **Settings as configuration database**: The 3,007-line settings file doubles as a retailer configuration store. Payment gateway keys, API endpoints, store-specific URLs, and email templates are all embedded in settings rather than in the database or a configuration service.

2. **Hardcoded secrets in settings**: Multiple API keys, tokens, and passwords are hardcoded in `settings.py` (Cloudinary, Stripe, Adyen, Spreedly, Clover, JudoPay, PayUnity, PayPal, MQTT, email, social auth). While some are for sandbox/demo environments, several appear to be production credentials.

3. **Cache override bug**: The Memcached configuration (lines 584-597) is unconditionally overridden by `LocMemCache` (lines 599-604), meaning Memcached is never actually used despite being configured.

4. **Decathlon block**: `SentryInterceptMiddleware` returns `403 Forbidden` for any request with `"decathlon"` in the `bundleid` header — this appears to be a deliberate block of the Decathlon app.

5. **JWT without signature verification**: `MishiKeycloakAuthentication` decodes JWTs without verifying the signature, relying on Kong API Gateway for that validation.

6. **Dual tax services**: Both a legacy gRPC tax service (`ms-tax:9000`) and a newer HTTP tax service (`ms_tax:80`) are configured, confirming the legacy-to-new migration pattern.

7. **`lib/` on sys.path**: `settings.py:18` adds `lib/` to `sys.path`, allowing direct imports from the `lib/` directory without package prefixes.

## Open Questions

- Why is the Memcached configuration overridden with `LocMemCache`? Is this intentional for the current environment or a debugging leftover?
- Is the Decathlon `bundleid` block in `SentryInterceptMiddleware` still needed, or is it a temporary measure?
- How are the `retailer_configurations/` Python files loaded and used? Many directories contain empty files. No import mechanism was found in `settings.py`.
- The `_test.py` file at the project root contains hardcoded store UUIDs — is this still used?
- Several production-looking API keys and tokens are hardcoded in `settings.py`. Are these overridden by environment variables in production, or are they the actual production values?
- The `explorer` app (Django SQL Explorer) is enabled in all environments with connection to the default database. Is access restricted by additional auth rules?
