# Promotions Service — Overview

> Last updated by: Task T05 — API, Utilities & Tests Summary

## Overview

The Promotions Service (`service.promotions`) is a standalone Django microservice focused on ingesting promotional data and evaluating discounts on customer baskets. It implements a sophisticated discount engine with 8 promotion family types and a best-discount algorithm that uses graph-based clustering to find the optimal combination of promotions for a given basket.

The service is stateless by design — all important data resides in MySQL, and the Django container holds no valuable state. It was originally built to support Eroski (a Spanish retailer) but has been extended to serve multiple retailers.

## Key Design Principles

1. **TDD is mandatory** — Tests are written before code.
2. **No ForeignKeys or AutoIncrements** — Avoided for scalability. Relationships are managed at the application level.
3. **Python Type Hints are mandatory** — All method signatures must include type hints, enforced via CI.
4. **Django must not communicate with the outside world** — Any external service communication goes through a dedicated proxy in docker-compose.
5. **Stateless containers** — All data resides in the database; no in-container state.

## Architecture

### Project Structure

```
service.promotions/
├── service/
│   ├── promotions/           # Django project config
│   │   ├── settings.py       # Django settings
│   │   ├── urls.py           # URL routing
│   │   ├── wsgi.py           # WSGI application
│   │   └── asgi.py           # ASGI application
│   ├── mpromo/               # Main (and only) Django app
│   │   ├── models/           # 6 database models
│   │   │   ├── promo_header.py
│   │   │   ├── promo_group.py
│   │   │   ├── promo_group_node.py
│   │   │   ├── promo_header_store.py
│   │   │   ├── promo_header_special_promo.py
│   │   │   └── promo_application_filter.py
│   │   ├── promo_core/       # Business logic engine (16 files)
│   │   │   ├── promo_logic.py          # V1 evaluation engine (~77KB)
│   │   │   ├── promo_logic_v2.py       # V2 evaluation engine (~73KB)
│   │   │   ├── promo_data_manager.py   # Data access layer
│   │   │   ├── promo_db_helper.py      # ORM-based DB queries
│   │   │   ├── promo_db_helper_raw_sql.py  # Raw SQL queries
│   │   │   ├── promo_family_factory.py # Family type selection
│   │   │   ├── promo_family_executer.py # Family execution
│   │   │   ├── easy_promo_family.py
│   │   │   ├── easy_plus_promo_family.py
│   │   │   ├── combo_promo_family.py
│   │   │   ├── line_special_promo_family.py
│   │   │   ├── basket_threshold_promo_family.py
│   │   │   ├── basket_threshold_with_discounted_target_promo_family.py
│   │   │   ├── requisite_groups_with_discounted_target_promo_family.py
│   │   │   └── evenly_distributed_multiline_discount_promo_family.py
│   │   ├── views.py          # API endpoints (~30KB)
│   │   ├── serializers.py    # DRF serializers
│   │   ├── admin.py          # Django admin config
│   │   ├── config.py         # Business constants
│   │   ├── tasks.py          # Background tasks
│   │   ├── math_util.py      # Math utilities
│   │   ├── mishipay_util.py  # MishiPay-specific utilities
│   │   ├── management/       # Django management commands
│   │   ├── migrations/       # DB migrations
│   │   └── tests/            # Test suite (47 items)
│   └── gunicorn.conf.py      # Gunicorn production config
├── conf/                      # Configuration templates
├── scripts/                   # Dev setup/teardown scripts
├── env/                       # Environment files (generated)
├── .azure/                    # Azure Pipelines CI/CD
├── .github/                   # GitHub Actions CI/CD
├── Dockerfile
├── docker-compose.yml         # Dev environment
├── docker-compose-azure-ci.yml # CI environment
└── pyproject.toml             # Python dependencies
```

## Django Settings

**Source:** `service/promotions/settings.py`

### Key Configuration

| Setting | Value | Notes |
|---------|-------|-------|
| `DEPLOYMENT_ENV` | `os.environ["DEPLOYMENT_ENV"]` | **Required env var.** Controls debug mode and Sentry initialization |
| `DEBUG` | `DEPLOYMENT_ENV == "local"` | Only enabled in local development |
| `ALLOWED_HOSTS` | `localhost, 127.0.0.1, ms-promo, promo-test.mishipay.com, promo.mishipay.com` | Also accepts `POD_IP` env var for Kubernetes |
| `ROOT_URLCONF` | `promotions.urls` | |
| `WSGI_APPLICATION` | `promotions.wsgi.application` | |

### Database Configuration

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': os.environ["MYSQL_DATABASE"],       # ms_promo
        'USER': os.environ["MYSQL_USER"],            # mpay_ms_promo
        'PASSWORD': os.environ["MYSQL_PASSWORD"],    # mishipay@123!
        'HOST': os.environ["MYSQL_DB_HOST"],         # percona-mysql-db
    }
}
```

All database credentials are provided via environment variables. The service uses **Percona MySQL 8.0** as its backing store.

### Installed Apps

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'ddtrace.contrib.django',    # Datadog tracing
    'django_guid',               # Request correlation IDs
    'mpromo',                    # The promotions app
    'health_check',              # Health check endpoints
    'health_check.db',           # DB health check
]
```

When `ELASTIC_APM_ENABLED=True` (non-local environments), `elasticapm.contrib.django` is also added.

### Middleware Stack

```python
MIDDLEWARE = [
    'django_guid.middleware.GuidMiddleware',                    # Request correlation ID
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',               # Static file serving
]
```

When Elastic APM is enabled, `elasticapm.contrib.django.middleware.TracingMiddleware` is appended.

### Monitoring & Observability

| Tool | Integration | Details |
|------|-------------|---------|
| **Sentry** | Enabled in non-local environments | `CRITICAL` level logging integration, sends PII |
| **Datadog** | `ddtrace.contrib.django` | Tracing always installed, JSON log handler |
| **Elastic APM** | Conditional on `ELASTIC_APM_ENABLED` | Service name: `ms-promo` |

### Logging Architecture

The service uses a custom logging library (`mishipay-python-logger`) with multiple specialized handlers:

| Handler | Purpose | Environment |
|---------|---------|-------------|
| `datadog_json_logs_handler` | Structured JSON logs for Datadog | All |
| `db_logs_handler` | Database query logs to file | Local/unittest only |
| `error_logs_handler` | Error logs to file | All |
| `error_logs_stream_handler` | Error logs to stdout/stderr | All |
| `requests_logs_handler` | HTTP request logs to file | All |
| `security_logs_handler` | Security event logs to file | Production only |
| `local_standard_logs_stream_handler` | Console logs | Local/unittest only |
| `prod_standard_logs_stream_handler` | Console logs | Production only |

**Log filters:**
- `correlation_id` — Injects `django_guid` correlation IDs
- `redact_sensitive_information` — Redacts sensitive data (patterns not configured)
- `requires_local_env` / `requires_prod_env` — Environment-based filtering

**Log directory:** Configured via `LOG_DIR` env var, defaults to `./logs/ms-promo/`.

## Configuration Constants

**Source:** `service/mpromo/config.py`

### Supported Retailers (Legacy)

| Constant | Value | Retailer |
|----------|-------|----------|
| `RETAILER_EROSKI` | `"1"` | Eroski |
| `RETAILER_CONNEXUS` | `"2"` | Connexus |
| `RETAILER_MUJI` | `"3"` | Muji |
| `RETAIL_FLYING_TIGER` | `"4"` | Flying Tiger |

> Note: These retailer constants are for backwards compatibility only. New clients should not be added here.

### Promotion Families

| Constant | Code | Display Name |
|----------|------|--------------|
| `FAMILY_EASY` | `'e'` | Easy |
| `FAMILY_EASY_PLUS` | `'p'` | Easy+ |
| `FAMILY_REQUISITE_GROUPS_WITH_DISCOUNTED_TARGET` | `'r'` | Requisite Groups With Discounted Target |
| `FAMILY_COMBO` | `'c'` | Combo |
| `FAMILY_LINE_SPECIAL` | `'l'` | Line Item Specific |
| `FAMILY_BASKET_THRESHOLD` | `'b'` | Basket Threshold Value |
| `FAMILY_BASKET_THRESHOLD_WITH_DISCOUNTED_TARGET` | `'t'` | Basket Threshold Value With Discounted Target |
| `FAMILY_EVENLY_DISTRIBUTED_MULTILINE_DISCOUNT` | `'m'` | Evenly Distributed Multiline Discount |

### Evaluation Criteria

| Constant | Code | Description |
|----------|------|-------------|
| `PROMO_EVALUATE_PRIORITY` | `'p'` | Select promo with smaller priority number |
| `PROMO_EVALUATE_BEST_DISCOUNT` | `'b'` | Select promo with highest discount |

### Discount Types

| Constant | Code | Display |
|----------|------|---------|
| `DISCOUNT_TYPE_PERCENT_OFF` | `'p'` | %Off |
| `DISCOUNT_TYPE_VALUE_OFF` | `'v'` | $Off |
| `DISCOUNT_TYPE_FIXED` | `'f'` | $ (fixed price) |

### Discount Type Strategies

| Constant | Code | Description |
|----------|------|-------------|
| `DISCOUNT_TYPE_STRATEGY_ALL_ITEMS` | `'a'` | Discount applied on all items collectively |
| `DISCOUNT_TYPE_STRATEGY_EACH_ITEM` | `'e'` | Discount applied on each item individually |

### Discount Value On (Price Type)

| Constant | Code | Description |
|----------|------|-------------|
| `DISCOUNT_ON_MRP` | `'m'` | Applied on Maximum Retail Price |
| `DISCOUNT_ON_SP` | `'s'` | Applied on Sale Price |
| `DISCOUNT_ON_FP` | `'f'` | Applied on Final Price |

### Application Types

| Constant | Code | Description |
|----------|------|-------------|
| `APPLY_ON_BASKET` | `'b'` | Discount applied on basket |
| `APPLY_AS_CASHBACK` | `'c'` | Discount given as cashback |
| `APPLY_AS_TENDER` | `'t'` | Discount applied as tender |
| `APPLY_AS_POSLOG_EXCLUSION` | `'p'` | Exclude promo discounts from POSLog |

### Availability

| Constant | Code | Description |
|----------|------|-------------|
| `AVAILABILITY_ALL` | `'a'` | Available to all customers (general) |
| `AVAILABILITY_SPECIAL` | `'s'` | Available to special/loyalty customers only |

### Target Group Item Selection Criteria

| Constant | Code | Description |
|----------|------|-------------|
| `GROUP_ITEM_SELECTION_CRITERIA_FAVOR_RETAILER` | `'l'` | Pick least expensive items (favors retailer — less discount) |
| `GROUP_ITEM_SELECTION_CRITERIA_FAVOR_CUSTOMER` | `'lc'` | Pick least expensive items (favors customer) |
| `GROUP_ITEM_SELECTION_CRITERIA_MOST_EXPENSIVE` | `'m'` | Pick most expensive item first |

### Node Types

| Constant | Code | Description |
|----------|------|-------------|
| `NODE_TYPE_ITEM` | `'i'` | Individual item (SKU) |
| `NODE_TYPE_CATEGORY1` through `NODE_TYPE_CATEGORY7` | `'c1'` through `'c7'` | Category hierarchy levels 1-7 |

### Evenly Distributed Multiline Discount Split Types

| Constant | Code | Description |
|----------|------|-------------|
| `EVENLY_DISTRIBUTED_MULTILINE_DISCOUNT_EQUAL_SPLIT` | `'e'` | Equal split across items |
| `EVENLY_DISTRIBUTED_MULTILINE_DISCOUNT_PROPORTIONAL_SPLIT` | `'p'` | Proportional split based on item price |

### Retailer-Specific Behavior Flags

| Constant | Retailers | Description |
|----------|-----------|-------------|
| `RETAILER_APPLY_ZERO_DISCOUNT_PROMOS` | DDF, MLSE-CA, KIKOMILANO, Grandiose | Apply promotions even when discount is zero |
| `RETAILER_APPLY_ONE_BASKET_PROMO_PER_LAYER` | DDF | Limit to one basket-level promotion per layer |

### Other Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `PROMO_MAX_APPLICATION_LIMIT` | `100000` | Maximum number of times a promotion can be applied |
| `SPLIT_KEY` | `'-#$@'` | Delimiter used for internal string splitting |

## API Endpoints

**Source:** `service/promotions/urls.py`

| Method | Path | View | Description |
|--------|------|------|-------------|
| GET/POST | `/api/1.0/promotions/` | `PromotionCreateAndList` | Create or list promotions |
| POST | `/api/1.0/promotions/bulk_transaction/` | `PromotionBulkTransaction` | Bulk create/update promotions |
| POST | `/api/1.0/promotions/transaction/` | `PromotionBulkTransaction` | **Deprecated** — alias for `bulk_transaction` |
| POST | `/api/1.0/promotions/evaluate/` | `PromotionEvaluate` | Evaluate promotions for a basket |
| GET | `/api/1.0/promotions/retailer/<retailer>/` | `PromotionsByRetailer` | List promotions by retailer |
| GET | `/api/1.0/promotions/store/<store_id>/` | `PromotionsByStore` | List promotions by store |
| GET | `/api/1.0/promotions/bulk/<store_id>/` | `PromotionsBulkByStore` | Bulk list promotions by store |
| GET | `/api/1.0/promotions/suggestions/` | `PromotionSuggestionListByNodes` | Get promotion suggestions by product nodes |
| GET | `/api/1.0/promotions/item_recommendations/` | `PromoBasedItemRecommendations` | Get item recommendations based on promotions |
| GET | `/api/1.0/promotions/<store_id>/<retailer_id>` | `PromotionDetailByStore` | Get promotion details for a store |
| GET | `/api/1.0/p/<store_id>/<retailer_id>` | `PromotionDetailByStore` | **Deprecated** — short alias |
| — | `/health_check/` | Health check | Django health check endpoint |
| — | `/admin/` | Django Admin | Admin interface ("MishiPay Promotions Admin") |

## Dependencies

### Depends On
- **Percona MySQL 8.0** — Primary data store
- **Datadog Agent** — APM and log collection
- **mishipay-python-logger** — Private logging library (installed from GitHub via SSH)

### Depended On By
- **MishiPay Monolith** (`backend_v2`) — Calls the promotions API for basket evaluation
- **MishiPay Monolith Compose** — Includes `ms-promo` as a service in the main docker-compose

## Notable Patterns

1. **Single Django App:** The entire service is a single Django app (`mpromo`) within the `promotions` project.
2. **Large Logic Files:** The core evaluation engines (`promo_logic.py` at 77KB, `promo_logic_v2.py` at 73KB) are very large monolithic files.
3. **No REST Framework Auth:** The service does not configure DRF authentication — it relies on network-level security (internal Docker network).
4. **PostgreSQL Config in MySQL Service:** The `conf/postgresql.conf` file exists in the repo despite the service using MySQL. This appears to be shared configuration used by the monolith's PostgreSQL database when both are managed together.

## Utility Functions

**Source:** `service/mpromo/mishipay_util.py` (19 lines)

### `elapsed_interval(start, end)`

Formats a `datetime.timedelta` as `HH:MM:SS` string. Used by the `delete_promos` management command for timing operations.

```python
def elapsed_interval(start, end):
    elapsed = end - start
    min, secs = divmod(elapsed.days * 86400 + elapsed.seconds, 60)
    hour, minutes = divmod(min, 60)
    return '%.2d:%.2d:%.2d' % (hour, minutes, secs)
```

### `send_slack_message(data, env, channel="#alerts_client_data")`

Sends a Slack notification via the Slack API. Formats the message with an environment prefix (`[ENV=<env>]`).

**Security issue:** Contains a **hardcoded Slack bot token** (`xoxb-...`) directly in source code (`mishipay_util.py:12`). This token should be moved to an environment variable.

**Note:** This function is not imported or called by any code within the promotions service itself. It may be called by external scripts or by the monolith via a shared utility pattern.

## Background Tasks

**Source:** `service/mpromo/tasks.py` (31 lines)

### `promo_delete_job()`

Processes soft-deleted promotions. Iterates all `PromoHeaderStore` records where `soft_delete=True` and performs cleanup within a single atomic transaction.

**Logic:**
1. For each soft-deleted `PromoHeaderStore` entry:
   - **If other active stores exist** for the same promo: just unlink (delete the soft-deleted store entry)
   - **If no other stores exist**: cascade-delete everything — `PromoHeaderStore`, `PromoGroupNode`, `PromoGroup`, `PromoHeaderSpecialPromo`, and `PromoHeader`

**Deletion order:** Store entries -> Nodes -> Groups -> Special promos -> Header

**Performance note:** This is a per-record implementation (N+1 queries for each soft-deleted entry). For production use at scale, the `delete_promos` management command provides a bulk-optimized version. See [Management Commands](#management-commands).

## Management Commands

**Source:** `service/mpromo/management/commands/delete_promos.py` (218 lines)

### `delete_promos`

Bulk-optimized version of `promo_delete_job()` with additional capabilities. Accessed via:

```bash
docker exec backend_v2 python3 manage.py delete_promos --softdelete --batch-size 5000
```

**Arguments:**

| Flag | Description |
|------|-------------|
| `--softdelete` | Delete promotions with soft-deleted store entries |
| `--deactivated` | Delete promotions with `is_active=False` |
| `--dry-run` | Preview what would be deleted without executing |
| `--batch-size N` | Limit the number of `promo_header_id`s to process (ordered by KSUID, oldest first) |

**Implementation class:** `PromoDelete`

#### `delete_promos_with_soft_delete_true(dry_run, batch_size)` (`delete_promos.py:24-120`)

Bulk-optimized soft-delete cleanup:

1. Queries all `promo_header_id`s with `soft_delete=True` entries (optionally limited by `batch_size`)
2. Separates into two sets:
   - **Unlink set** — promos that have other active (non-soft-deleted) stores → only remove the soft-deleted `PromoHeaderStore` entries
   - **Full delete set** — promos with no active stores → cascade delete all related records
3. Bulk-deletes each entity type in dependency order (nodes -> groups -> stores -> special promos -> headers)

This is significantly more efficient than `promo_delete_job()` because it uses set-based operations instead of per-record queries.

#### `delete_deactivated_promos(dry_run, batch_size)` (`delete_promos.py:123-179`)

Deletes all promotions where `is_active=False`:

1. Queries deactivated promo IDs (optionally limited by `batch_size`, ordered by KSUID)
2. Collects related `PromoGroup` IDs
3. Bulk-deletes in order: nodes -> groups -> stores -> special promos -> headers

Both methods:
- Run within `transaction.atomic()` for all-or-nothing behavior
- Support `--dry-run` mode that logs counts without deleting
- Support `--batch-size` to limit scope (KSUID ordering ensures oldest promos are processed first)
- Log elapsed time using `elapsed_interval()` from `mishipay_util.py`

## Test Coverage Summary

**Source:** `service/mpromo/tests/` (44 test files)

The test suite is comprehensive, with dedicated test files for each promotion family, cross-family combinations, and edge cases. All tests extend `PromoTestCase` (defined in `tests/base.py`), which provides helper methods for creating promotions, evaluating baskets, and validating responses.

**Run command:** `docker exec ms-promo python3 manage.py test --keepdb mpromo.tests.<module>`

### Test Base Class (`tests/base.py`)

`PromoTestCase` extends Django's `TestCase` and provides:

| Helper | Purpose |
|--------|---------|
| `validate_ksuid()` | Validates KSUID format |
| `assertGroupResponse()` | Asserts promo group structure |
| `assertNodeResponse()` | Asserts promo group node structure |
| `evaluate_promo()` | Sends evaluate request and returns response |
| `validate_basket_response_info()` | Validates basket response fields |
| `create_basket_threshold_promo_*()` | Factory methods for basket threshold test data |

### Test Categories

#### CRUD / Batch Operations (3 files)

| File | Scenarios |
|------|-----------|
| `test_batch.py` | Bulk header update, node update, create duplicate detection, create+delete transaction, bulk load by store, multi-promo lifecycle (7 tests) |
| `test_crud_promo_family.py` | Basic CRUD operations per family type |
| `test_store.py` | Store-level delete cascade |

#### Per-Family Evaluation (8+ files)

Each promotion family has at least one dedicated test file:

| File | Family |
|------|--------|
| `test_easy_promo_family.py` | Easy |
| `test_easy_plus_promo_family.py` | Easy+ |
| `test_combo_promo_family.py` | Combo |
| `test_line_special_promo_family.py` | Line Special |
| `test_basket_threshold_promo_family.py` | Basket Threshold |
| `test_requisite_groups_*_1_group.py` | Requisite Groups (1 group) |
| `test_requisite_groups_*_2_groups.py` | Requisite Groups (2 groups) |
| `test_evenly_distributed_*_1_group.py` | Evenly Distributed (1 group) |
| `test_evenly_distributed_*_2_group.py` | Evenly Distributed (2 groups) |

#### Easy Family Variants (4 files)

| File | Variant |
|------|---------|
| `test_easy_promo_family_by_category.py` | Category-based matching |
| `test_easy_promo_family_configured_as_requisite_family.py` | Easy configured as requisite |
| `test_easy_promo_family_weighted.py` | Weighted items |
| `test_easy_promo_family_with_addons.py` | Add-on items |

#### Evenly Distributed Proportional Variants (6+ files)

| File | Variant |
|------|---------|
| `test_evenly_distributed_*_proportional_buy2get1.py` | Buy-2-get-1 proportional |
| `test_evenly_distributed_*_proportional_extra_requisite.py` | Extra requisite items |
| `test_evenly_distributed_*_proportional_group.py` | Group-level proportional |
| `test_evenly_distributed_*_proportional_group2.py` | Group variant 2 |
| `test_evenly_distributed_*_proportional_group3_value_off.py` | Group 3 with value-off |
| `test_evenly_distributed_*_proportional_group4.py` | Group variant 4 |
| `test_evenly_distributed_*_valueoff.py` | Value-off variant |

#### Cross-Family Combinations (7 files)

Tests evaluate baskets where both a basket threshold promo and another family type are active:

| File | Combination |
|------|-------------|
| `test_basket_threshold_and_easy.py` | Basket Threshold + Easy |
| `test_basket_threshold_and_easy_plus.py` | Basket Threshold + Easy+ |
| `test_basket_threshold_and_combo.py` | Basket Threshold + Combo |
| `test_basket_threshold_and_line_special.py` | Basket Threshold + Line Special |
| `test_basket_threshold_and_requisite_1_group.py` | Basket Threshold + Requisite (1 group) |
| `test_basket_threshold_and_requisite_2_groups.py` | Basket Threshold + Requisite (2 groups) |
| `test_basket_threshold_and_evenly_distributed.py` | Basket Threshold + Evenly Distributed |

#### Best Discount Algorithm (2 files)

| File | Scenario |
|------|----------|
| `test_best_discount_common_items_promos.py` | Promotions sharing common items |
| `test_best_discount_no_common_items_promos.py` | Promotions with disjoint items |

#### Special Features (6 files)

| File | Feature |
|------|---------|
| `test_on_the_fly_special_promo.py` | On-the-fly special promo evaluation |
| `test_promo_with_availabiltiy_special.py` | Special availability (loyalty) promos |
| `test_stackable_promos.py` | Multi-layer stackable promotions |
| `test_edge_case_promos.py` | Edge case handling |
| `test_promo_suggestion.py` | Promotion suggestion engine |
| `test_applicalble_promotions_list.py` | Applicable promotions listing |

**Note:** Several test filenames contain typos: `availabiltiy` (should be `availability`), `applicalble` (should be `applicable`), `test_basket_threshold_special_promo_evaluate_max_apllication_limit.py` (`apllication` should be `application`).

#### Retailer-Specific (2 files)

| File | Retailer |
|------|----------|
| `test_retailer_muji_promos.py` | MUJI-specific promo behavior |
| `test_eroski_multi_pack_promo.py` | Eroski multi-pack promotions |

#### Management Commands (1 file)

| File | Coverage |
|------|----------|
| `test_delete_promos_command.py` | 20+ tests: soft-delete cleanup, deactivated cleanup, dry-run mode, batch-size limiting, atomicity, edge cases |

## Open Questions

- ~~What is the functional difference between V1 and V2 evaluation engines?~~ **Answered in T03**: See [Business Logic](./business-logic.md). V1/V2 differ in basket promo processing order, cluster types, and cross-layer item propagation.
- ~~What is the `mishipay_util.py` utility used for?~~ **Answered in T05**: Contains `elapsed_interval()` (time formatting) and `send_slack_message()` (Slack notifications with hardcoded bot token). Neither function is imported within the promotions service itself.
- How does the monolith communicate with this service — direct HTTP or via the API gateway? (Based on docker-compose, likely direct HTTP over `mpay_net`)
