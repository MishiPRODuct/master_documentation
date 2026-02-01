# Coupons, Special Promos & Promotions (Platform Side)

> Last updated by: Task T12 — Document mishipay_coupons, mishipay_special_promos, and promotions modules

## Overview

The MishiPay platform has three distinct modules handling discount and promotional logic on the monolith side. Together they form the platform-facing promotion layer that sits between clients and the promotions microservice (`service.promotions`):

1. **`mishipay_coupons`** (14 files) — Orchestration layer for coupon validation, redemption, and user coupon queries. Delegates to the external MS Loyalty Service and the referrals module.
2. **`mishipay_special_promos`** (13 files) — Handles non-coupon discount types: vouchers, discount group qualifiers, Italian lottery codes, donations, and on-the-fly discounts. Manages the `basket.special_promos` JSON structure.
3. **`promotions`** (82 files) — Promotion data import framework. Parses retailer-specific data files (CSV, ASC, XML-RPC) and pushes promotions to the promotions microservice via `PromotionBatchOperation`.

These modules do **not** define their own Django models. All persistent coupon/loyalty data lives in the external MS Loyalty Service, and all promotion definitions live in the promotions microservice. The platform modules store transient discount state in `Basket.special_promos` (a JSON field) and `Basket.extra_data`.

## Key Concepts

| Term | Definition |
|------|-----------|
| **MS Loyalty Service** | External microservice at `settings.MS_LOYALTY_SERVICE` that manages coupon lifecycle (create, validate, redeem) and user coupon inventories |
| **Promotions Microservice** | `service.promotions` — evaluates baskets against promotion rules. See [Promotions Service](../promotions-service/overview.md) |
| **`special_promos`** | JSON field on `Basket` model storing active discount programs for a shopping session |
| **Program** | A discount category: `REFERRAL`, `RETAILER_COUPON`, `RETAILER_COUPON_STUDENT_DISCOUNT_V2`, `RETAILER_COUPON_MISHIPAY`, `STUDENT_DISCOUNT`, `VOUCHER_DISCOUNT`, `IT_GOVT_LOTTERY_CODE`, `DONATION`, etc. |
| **`promo_group_qualifier`** | Identifier linking a coupon/discount to a promotion definition in the promotions microservice |
| **`PromotionBatchOperation`** | Client class from `mishipay_items.mpay_promo` for bulk create/delete of promotions in the microservice |
| **On-the-fly promo** | A discount created dynamically at checkout time (e.g., staff discount, ad-hoc override) rather than pre-configured |
| **Retailer string** | Format `"{CHAIN.upper()}-{REGION.upper()}"` (e.g., `"MUJI-GB"`, `"MISHIPAY-GB"`) used as namespace for promotions |

## Architecture

```
Client App
    │
    ├─── POST /coupons/v1/validate_coupons/  ──→  mishipay_coupons.views.ValidateCoupons
    │                                                    │
    │                                                    ├─→ MS Loyalty Service (HTTP)
    │                                                    └─→ mishipay_referrals (referral codes)
    │
    ├─── POST /special_promos/v1/validate/   ──→  mishipay_special_promos.views.ValidateSpecialPromo
    │                                                    │
    │                                                    ├─→ mishipay_coupons (COUPON type)
    │                                                    ├─→ VoucherService (VOUCHER_DISCOUNT)
    │                                                    └─→ Basket.special_promos (saved)
    │
    ├─── POST /special_promos/v1/special_promo_on_fly/  ──→  SpecialPromoOnFly
    │                                                            │
    │                                                            └─→ PromotionHelper → on-the-fly promo
    │
    └─── [Basket evaluation] ──→  Promotions Microservice
                                        │
                                        └─→ reads special_promos_info from basket
```

**Promotion Import Pipeline** (separate from runtime):
```
Retailer Data Files (CSV/ASC/XML-RPC/JSON)
    │
    ├─→ promotions.abc.PromotionImport  (extract → clean → transform)
    │
    ├─→ Retailer-specific mapping_service  (DB lookups, data transformation)
    │
    └─→ PromotionBatchOperation  ──→  Promotions Microservice API
```

---

## Module 1: `mishipay_coupons`

**Path:** `MishiPay/mainServer/mishipay_coupons/`

### Structure

```
mishipay_coupons/
├── __init__.py
├── admin.py                    # Empty
├── apps.py                     # Django app config
├── coupons_helper.py           # Core business logic (378 lines)
├── models.py                   # No models defined
├── tests.py                    # Empty
├── urls.py                     # 4 URL patterns
├── views.py                    # 4 view classes (122 lines)
├── migrations/
│   └── __init__.py
└── management/commands/
    ├── standard_coupon_import.py
    ├── mysalonstop_coupon_import.py
    ├── emmas_garden_loyalty_import.py
    ├── muji_coupons_import.py
    └── nude_foods_coupon_import_deprecated.py
```

### API Endpoints

All endpoints require `IsAuthenticated` permission.

| Method | Path | View | Purpose |
|--------|------|------|---------|
| GET | `/coupons/v1/user_coupons/` | `UserCoupons` | Fetch user's available coupons from MS Loyalty Service |
| POST | `/coupons/v1/user_coupons/` | `UserCoupons` | Create user coupons via MS Loyalty Service |
| POST | `/coupons/v1/validate_coupons/` | `ValidateCoupons` | Validate coupon codes against MS Loyalty Service |
| POST | `/coupons/v1/redeem_coupons/` | `RedeemCoupons` | Redeem coupons after order completion |
| GET | `/coupons/v1/retailer_coupons/` | `RetailerCoupons` | Get retailer-specific coupons with date/title filtering |

**Source:** `mishipay_coupons/urls.py:8-13`, `mishipay_coupons/views.py`

### Business Logic — `coupons_helper.py`

#### `validate_coupons(req_data, retailer, store_type, basket_obj=None)`

**Source:** `mishipay_coupons/coupons_helper.py:32-184`

Validates coupon codes with retailer-specific branching:

1. **Fetch basket** — Loads `Basket` by `basket_id` from request data
2. **MUJI gate** — If store is `MujiStoreType` and input type is `COUPON`, blocks validation when `"MUJIWEEKUK 20% off"` is in `basket.extra_data["applied_promotions"]` (prevents stacking)
3. **Carters gate** — If store is `CartersStoreType`, delegates to `CartersPromo` client (lazy-imported from `mishipay_items.carters_utility.client`). Stores validated coupon data in `basket.extra_data["coupon_data"]`
4. **Referral separation** — Splits input coupons into referral codes (prefix `"R-"`) and regular coupons
5. **Duplicate check** — If a `REFERRAL` program is already applied to the basket, blocks additional codes. Exceptions:
   - `RotanaStoreType` can override referral coupons for the same program (`STORE_TYPE_OVERRIDE_REFERRAL_COUPON` set at line 28)
   - `RETAILER_COUPON*` programs can coexist with referrals
6. **Regular coupon path** — POSTs to `{MS_LOYALTY_SERVICE}/coupon/v1/validate_coupons/` with retailer namespace. On success, builds `special_promo_data` structure and saves to basket via `save_basket_special_promos()`
7. **Referral path** — First checks `is_first_order_for_customer_for_retailer()` (referrals only for first-time buyers). Then validates via `mishipay_referrals.referral_helper.validate_referral_code()`. Stores referral campaign metadata in `extra_info`

#### `redeem_coupons_helper(basket, order_status, order_id)`

**Source:** `mishipay_coupons/coupons_helper.py:186-246`

Called during order completion to redeem applied coupons:

1. **Gate** — Only proceeds for `OrderStatus.COMPLETED` or `OrderStatus.VERIFIED`
2. **Discount extraction** — Reads `basket.ms_promo_response.basket.special_promos_info.applied_programs` to get actual discount amounts per program
3. **Iterates programs** — For each of `REFERRAL`, `RETAILER_COUPON_STUDENT_DISCOUNT_V2`, `RETAILER_COUPON`, `RETAILER_COUPON_MISHIPAY`:
   - Skips if discount is 0 or negative
   - Extracts coupon values from `basket.special_promos`
   - Calls `redeem_coupons()` with `redeem_mode: "FULL"`
4. **MISHIPAY retailer override** — For `RETAILER_COUPON_MISHIPAY` program, the retailer string is set to `"MISHIPAY-{region}"` instead of the store's chain (line 238). This allows MishiPay-branded discounts to work across any retailer.

#### `redeem_coupons(req_data, retailer, basket_obj=None)`

**Source:** `mishipay_coupons/coupons_helper.py:248-350`

Handles actual redemption by delegating to MS Loyalty Service or referral service. Mirrors the validation flow structure: separates referral codes from regular coupons, calls the appropriate external service, and updates `basket.special_promos` with `"redeemed": True`.

#### `user_coupons(req_data)`

**Source:** `mishipay_coupons/coupons_helper.py:353-378`

Fetches user coupons from MS Loyalty Service. Supports two query modes:
- **GLOBAL retailers** — Filters by `store_region` parameter
- **Standard retailers** — Filters by `retailer` string

### Retailer Coupons Filtering

**Source:** `mishipay_coupons/views.py:87-121`

The `RetailerCoupons.get()` endpoint applies post-processing to the MS Loyalty Service response:

1. Calls `filter_valid_coupons()` — Removes coupons outside their `start_date_time` / `end_date_time` window (ISO 8601 parsing with `"Z"` → `"+00:00"` conversion)
2. Reads `GlobalConfig.get_config_value("COUPON_TITLE_ORDERING")` — A dict mapping `store_type` (lowercase) to an ordered list of coupon titles
3. Sorts coupons: matched titles first (in configured order), then unmatched titles appended

### Management Commands

| Command | Retailer | Purpose |
|---------|----------|---------|
| `standard_coupon_import` | Generic | CSV-based import with headers: `COUPON_ID`, `PROMO_TYPE`, `PROMO_VALUE`, `END_DATE`, `MIN_THRESHOLD`, `MAX_APPLICABLE`, `MAX_USER_USAGE`, `START_DATE` |
| `mysalonstop_coupon_import` | MySalonStop | CSV import variant |
| `muji_coupons_import` | MUJI | Bulk coupon creation supporting multiple vendors (UniDays, etc.) and regions (GB, FR, IT, DE, ES, FI) |
| `emmas_garden_loyalty_import` | Emma's Garden | Complex import from 3 CSV files: discount customers, customer emails/tax, eligible items. Creates 4 coupon types based on discount/tax combinations |
| `nude_foods_coupon_import_deprecated` | Nude Foods | Deprecated command |

### Dependencies

- **External:** `settings.MS_LOYALTY_SERVICE` (HTTP)
- **Internal:** `mishipay_special_promos.special_promo_helper` (save/read basket special promos), `mishipay_referrals.referral_helper` (referral validation/redemption), `mishipay_items.models.Basket`, `dos.models.Store`, `mishipay_config.models.GlobalConfig`, `mishipay_items.carters_utility.client.CartersPromo` (lazy import)

---

## Module 2: `mishipay_special_promos`

**Path:** `MishiPay/mainServer/mishipay_special_promos/`

### Structure

```
mishipay_special_promos/
├── __init__.py
├── admin.py                    # Empty
├── app_settings.py             # Prerequisite function map (9 lines)
├── apps.py                     # Django app config
├── models.py                   # No models defined
├── pre_requisite_check.py      # Seventh Heaven validation (74 lines)
├── special_promo_helper.py     # Core business logic (605 lines)
├── urls.py                     # 4 URL patterns
├── views.py                    # 4 view classes (141 lines)
├── migrations/
│   └── __init__.py
└── tests/
    ├── __init__.py
    ├── test_seventh_heaven_advance_payment_integration.py  (759 lines)
    └── test_special_promo_apply_on_the_fly.py              (361 lines)
```

### API Endpoints

All endpoints require `IsAuthenticated` permission.

| Method | Path | View | Purpose |
|--------|------|------|---------|
| POST | `/special_promos/v1/validate/` | `ValidateSpecialPromo` | Validate special promos (coupons, vouchers, qualifiers, lottery codes, donations) |
| POST | `/special_promos/v1/remove/` | `RemoveSpecialPromo` | Remove a specific program from basket's special promos |
| POST | `/special_promos/v1/special_promo_on_fly/` | `SpecialPromoOnFly` | Apply ad-hoc discount at checkout |
| POST | `/special_promos/v1/special_promo_summary/` | `SpecialPromoSummary` | Get discount summary for a program on a business date |

**Source:** `mishipay_special_promos/urls.py:11-15`

### Data Structure — `basket.special_promos`

The central data structure is a JSON field on the `Basket` model:

```json
{
  "programs": [
    {
      "program": "REFERRAL",
      "program_title": "2.5 off on basket",
      "apply_promos": true,
      "redeemed": true,
      "discount": "10",
      "extra_info": {
        "referral_campaign_id": null,
        "source_mishipay_customer_id": "uuid",
        "target_mishipay_customer_id": "uuid",
        "source_mishipay_customer_discount_coupon_code": null,
        "target_mishipay_customer_discount_coupon_code": "AkCBf5"
      },
      "program_inputs": [
        {
          "input_type": "COUPON",
          "input_value": "R-OlLRIy",
          "promo_group_qualifiers": ["ref-2.5off"],
          "max_application_limit": "1",
          "promo_override": null
        }
      ]
    }
  ]
}
```

**Source:** `mishipay_special_promos/special_promo_helper.py:26-75` (inline comment with sample)

This structure is transformed to `special_promos_info` for the promotions microservice by `get_special_promos_info_for_promo_evaluate()` (line 77).

### Business Logic — `special_promo_helper.py`

#### `validate_special_promos(req_data, retailer_chain, basket_obj, store_obj)`

**Source:** `mishipay_special_promos/special_promo_helper.py:257-438`

Handles validation for non-coupon discount types:

| `input_type` | Validation Logic |
|-------------|-----------------|
| `VOUCHER_DISCOUNT` | Calls `VoucherService().validate_apply_promo()`. On success, stores voucher details and updates `basket.total_voucher_discount` and `basket.voucher_code`. Returns immediately. |
| `DISCOUNT_GROUP_QUALIFIER` | Always valid. Adds input values directly as `promo_group_qualifier` IDs. Used for staff discounts and pre-configured special promotions. |
| `IT_GOVT_LOTTERY_CODE` | Validates that the code is exactly 8 characters. Does not save to basket on failure. |
| `DONATION` | Validates donation amount against minimum configured via `get_ftc_donation_details(store)`. Amount must be a multiple of the minimum. On success, creates a donation basket item via `upsert_non_existent_item_to_basket_on_fly()` with hardcoded VAT=0, hide_item_qty=True. Does not save to `special_promos`. |

For all types except `VOUCHER_DISCOUNT` and `DONATION` (which return early or skip save), the function builds a `special_promo_data` structure and saves it to the basket.

#### `save_basket_special_promos(basket_obj, special_promo, save_to_basket=True)`

**Source:** `mishipay_special_promos/special_promo_helper.py:113-128`

Upserts a program into `basket.special_promos`:
- If `special_promos` is `None`, initializes the structure with the new program
- Otherwise, removes any existing entry with the same `program` name, then appends the new one
- Saves the basket to DB unless `save_to_basket=False`

#### `get_special_promos_info_for_promo_evaluate(basket_special_promos)`

**Source:** `mishipay_special_promos/special_promo_helper.py:77-101`

Transforms `basket.special_promos` into the format expected by the promotions microservice evaluation API:

```json
[{
  "program_name": "REFERRAL",
  "program_title": "2.5 off on Basket",
  "group_qualifiers": [{
    "input": "R-OlLRIy",
    "qualifier_ids": ["ref-2.5off"],
    "max_application_limit": "1",
    "promo_override": null
  }]
}]
```

Only includes programs where `apply_promos` is `true`.

#### Customer Order History Functions

| Function | Source | Purpose |
|----------|--------|---------|
| `get_orders_of_customer(customer_id)` | Line 131 | Returns completed/verified/refunded orders excluding `saved_card` payments, ordered by `-completion_date` |
| `is_first_order_for_customer_for_retailer(store_type, orders)` | Line 145 | Returns `True` if no previous orders exist for the store type. MUJI-specific: excludes zero-amount and click-and-collect orders |
| `is_mishipoly_promo_applicable(orders)` | Line 163 | **Always returns `False`** — disabled function |
| `is_mishipoly_promo_applicable2(orders)` | Line 166 | Complex logic checking if customer qualifies for cross-retailer loyalty promo (`MPLDN1000`). Looks for 2 orders from different store types in the most recent cycle |
| `get_customer_order_count_for_retailer(customer_id, store_type)` | Line 234 | Counts completed/verified/refunded orders for a customer at a given store type, excluding `saved_card` and zero-amount payments |

#### `get_basket_level_promotions_label_details(ms_promo_response)`

**Source:** `mishipay_special_promos/special_promo_helper.py:441-456`

Extracts human-readable promotion labels from the microservice response for display in receipts. Reads `group_qualifier_description` from each qualifier. Has a TODO hack for BWG orders with missing `group_qualifiers` data (line 447).

#### `PromotionHelper` Class

**Source:** `mishipay_special_promos/special_promo_helper.py:517-604`

Stateless helper that constructs on-the-fly promotion requests for the promotions microservice. Used by the `SpecialPromoOnFly` view when `apply_on_the_fly=True`.

| Method | Returns |
|--------|---------|
| `get_promo_family()` | Always `BASKET_THRESHOLD` |
| `get_promo_discount_type(coupon_detail)` | `PERCENT_OFF` if `discount_type == "p"`, else `VALUE_OFF` |
| `get_discount_value(coupon_detail)` | String discount value |
| `get_group_node_details(coupon_detail)` | Single group with `node_id="ALL"`, `node_type=ITEM` |
| `get_discount_value_on()` | Always `"f"` (final price) |
| `get_max_application_limit()` | Always `"1"` |
| `get_promo()` | Delegates to `mishipay_items.helpers.promotions.get_special_promo_request()` with all parameters |

### Views Detail

#### `ValidateSpecialPromo`

**Source:** `mishipay_special_promos/views.py:20-60`

Routes validation based on `input_type`:
- **COUPON type** — Delegates to `mishipay_coupons.coupons_helper.validate_coupons()`. Applies retailer-specific transformations:
  - `MP-` prefix codes → retailer set to `"MISHIPAY-{region}"`, program set to `RETAILER_COUPON_MISHIPAY`
  - `NEWMEMBER10` code → program set to `RETAILER_COUPON`
  - MujiStoreType + Android Kiosk → program forced to `REFERRAL`, coupons uppercased
  - RotanaStoreType → program forced to `REFERRAL`
- **Other types** — Delegates to `validate_special_promos()`

#### `SpecialPromoOnFly`

**Source:** `mishipay_special_promos/views.py:78-131`

Applies ad-hoc discounts:

1. Loads basket with `select_related('store')`
2. **Prerequisite check** — Looks up store type in `SPECIAL_PROMO_PREREQUISITES_FUNCTION_MAP`. If a check function exists, calls it. Returns 400 if check fails.
3. Builds promo title: `"{program}: {value} %off"` or `"{program}: {value} off"`
4. Generates `group_qualifier`: `"special_discount_{program_lowered}"`
5. Two paths:
   - `apply_on_the_fly=True` → Uses `PromotionHelper.get_promo()` to build a full promotion object
   - Otherwise → Uses simpler `special_promo_obj` with `promo_override` dict
6. Calls `validate_special_promos()` to save and return response

### Prerequisite Check System

**Source:** `mishipay_special_promos/app_settings.py:1-8`

```python
SPECIAL_PROMO_PREREQUISITES_FUNCTION_MAP = {
    "seventhheavenstoretype": seventh_heaven_special_promo_prerequisites_check,
}
```

Currently only Seventh Heaven has prerequisite checks. The function (`pre_requisite_check.py:12-73`) enforces:

- **Advance Payment Order bypass** — If the program being applied is an Advance Payment Order (checked by name matching), skip all checks. APO can coexist with all other discounts.
- **Discount mutual exclusivity** — If any discount coupons are already applied (excluding APO), reject new discounts
- **Points/discount exclusivity** — If loyalty points are redeemed or `LOYALTY_COUPON_CODE` is in applied specials (excluding APO), reject new discounts

### Test Coverage

| Test File | Lines | Scope |
|-----------|-------|-------|
| `test_seventh_heaven_advance_payment_integration.py` | 759 | E2E tests: APO + coupon, APO + points, coupon + APO (reverse), points + APO (reverse), coupon vs points mutual exclusivity. Uses mocked MS Promo and loyalty clients. |
| `test_special_promo_apply_on_the_fly.py` | 361 | Unit tests for `SpecialPromoOnFly` view: `apply_on_the_fly=True` (uses PromotionHelper), `False/None` (uses default), percentage and fixed discounts |

---

## Module 3: `promotions` (Import Framework)

**Path:** `MishiPay/mainServer/promotions/`

### Structure

```
promotions/
├── __init__.py
├── abc.py                          # Abstract base class (88 lines)
├── apps.py
├── utils.py                        # DataFrame cleaning utilities (47 lines)
├── validators.py                   # Argument validators (26 lines)
├── flying_tiger/                   # Flying Tiger retailer import
├── grandiose/                      # Grandiose retailer import
├── SAP/                            # SAP ERP integration
├── staff_discount/                 # Staff discount creation
├── odoo/v17/                       # Odoo ERP XML-RPC client
├── management/
│   ├── commands/                   # 12 Django management commands
│   └── utils/coupons.py           # Loyalty coupon creation helper
└── tests/                          # Test utilities
```

### Core Framework — `PromotionImport` ABC

**Source:** `promotions/abc.py:11-87`

Abstract base class implementing an ETL pipeline for parsing retailer data files:

```
extract()  →  clean()  →  set_defaults()  →  transform()
```

| Stage | Method | Behavior |
|-------|--------|----------|
| **Extract** | `extract()` | Reads CSV via `pandas.read_csv()` with configurable `skiprows`, `encoding`, `dtype=str` |
| **Clean** | `clean()` | Drops rows missing required columns, removes unused columns, deduplicates by unique columns |
| **Defaults** | `set_defaults()` | Fills NaN values with configured defaults |
| **Transform** | `transform()` | Renames columns per `new_column_names` mapping. Subclasses override for custom logic. |

Class attributes for subclasses to define:

| Attribute | Type | Purpose |
|-----------|------|---------|
| `column_names` | `Iterable[str]` | Column names for the CSV (or header names if `use_headers=True`) |
| `required_columns` | `List[str]` | Columns that must not be null |
| `unused_columns` | `Optional[List[str]]` | Columns to drop after extraction |
| `unique_columns` | `Optional[Iterable[str]]` | Columns that must be unique (duplicates dropped) |
| `default_values` | `Optional[Mapping]` | `{column: default}` for NaN replacement |
| `new_column_names` | `Optional[Mapping]` | `{old_name: new_name}` for column renaming |
| `skiprows` | `Optional` | Rows to skip during CSV read |
| `encoding` | `str` | File encoding (default `"utf-8"`) |
| `use_headers` | `bool` | If `True`, use file's first row as headers (default `False`) |

The `data` property lazily triggers the full pipeline on first access.

### Shared Utilities

**`promotions/utils.py`:**
- `drop_bad_rows(df, columns, warn)` — Drops rows where any specified column is NaN, with optional logging
- `ensure_column_unique(df, column, keep, warn)` — Removes duplicate rows for a column, keeping first occurrence

**`promotions/validators.py`:**
- `validate_between_0_and_100(value)` — `ArgumentTypeError` if value is not a float between 0 and 100. Used by management commands for discount percentages.

### Retailer Import Modules

#### Flying Tiger (`promotions/flying_tiger/`)

**Purpose:** Import promotions from CSV definition + mapping files.

**Pipeline:**
1. `FlyingTigerPromotionDefinitions(filepath).data` — Parses promotion metadata (dates, discount type, eligibility)
2. `FlyingTigerPromotionMappings(filepath).data` — Parses promotion-to-product mappings
3. `pd.merge()` on `promo_id_retailer` — Joins definitions with mappings
4. `remap_from_db(df)` — Replaces retailer store IDs with UUIDs via `Store` model, fetches product identifiers by promotion category
5. `to_promotions(df)` — Converts DataFrame rows to `Promotion` objects with fixed constants (retailer=`FLYING_TIGER`, `availability="a"`, `layer="1"`)
6. `create_promotions(regions, promotions)` — Deletes all existing promos for matching store IDs, then bulk creates new ones

**Source:** `promotions/flying_tiger/main.py:11-19`

**Key constants:** `evaluate_criteria="p"` (priority), `discount_value_on="m"` (MRP), `max_application_limit="1000"` (effectively unlimited).

#### SAP (`promotions/SAP/`)

**Purpose:** Import SAP ERP promotions from `.asc` files. Supports two formats:

| Format | Class | Fields | Purpose |
|--------|-------|--------|---------|
| PROMOTXN | `PromotxnImport` | 37 columns | Simple promotions (price overrides, percent-off) |
| MIXMATCH | `MixmatchImport` | 106 columns | Mix-and-match deals |

**Pipeline:**
1. Parse `.asc` file via `PromotxnImport` or `MixmatchImport`
2. `map_promotxn_df_to_promotions()` — Returns `(creates, deletes)` tuple of `Promotion` lists
3. `upload_promotxn_promotions()` — Fetches existing promos for the store, computes node-level diff (existing - deleted + created), then bulk deletes and recreates. Handles incremental updates rather than full replacement.
4. For mixmatch: Uses `batch_apply_promotions()` for individual processing

**Source:** `promotions/SAP/main.py:1-41`, `promotions/SAP/promotion_service.py:11-81`

The SAP promotion service (`promotion_service.py`) implements a sophisticated node-level diff algorithm that merges existing promotion nodes with incoming changes rather than replacing entire promotions.

#### Grandiose (`promotions/grandiose/`)

**Purpose:** Import promotions from Grandiose retailer data. The most complex parser with support for multiple data types.

**Constants define 5+ column sets:**
- `PERIODIC_DISCOUNT` — Main promotion table
- `PERIODIC_DISCOUNT_LINE` — Items in promotion
- `VALIDATION_PERIOD` — Date/time ranges
- `PERIODIC_DISCOUNT_BENEFITS` — Discount tiers
- `MIX_AND_MATCH_LINE_GROUP` — Complex grouping

**Pipeline:**
1. `GrandiosePromotionImport` parser — Complex multi-file parsing (24KB parser module)
2. `to_promotions()` — Maps parsed data to `Promotion` objects
3. `create_promotions()` — Pushes to microservice with optional clean (delete all first)
4. `create_coupons()` — Also creates loyalty coupons if `coupons` data is present

**Source:** `promotions/grandiose/main.py:10-32`

#### Staff Discount (`promotions/staff_discount/`)

**Purpose:** Create percentage-off staff discount promotions.

**Pipeline:**
1. `get_store_ids(store_type, retailer_store_ids)` — Fetches store UUIDs and chain name
2. `get_nodes_by_category()` or `get_nodes_by_subcategory()` — Finds eligible items by category filtering (include/exclude)
3. `build_promotion()` — Constructs a `Promotion` object:
   - Family: `BASKET_THRESHOLD` (default, configurable)
   - Discount: Percentage-off (configurable)
   - Availability: `SPECIAL` (requires group qualifier activation)
   - Duration: Now to 4 years (hardcoded at line 43-45 in `mapping_service.py`)
   - `special_promo_info` set with `group_qualifier_id`
4. `update_or_create_promotion()` — Deletes existing promos with same `promo_id_retailer` for all stores, then creates new. Handles non-existent promos gracefully.

**Source:** `promotions/staff_discount/main.py:12-72`

**Category filtering:** Cannot filter by both category and subcategory simultaneously (raises `ValueError`). Node types: `"c1"` for category, `"c2"` for subcategory.

#### Odoo (`promotions/odoo/v17/`)

**Purpose:** Fetch promotion price lists from Odoo v17 ERP via XML-RPC.

**Source:** `promotions/odoo/v17/odoo_promotions.py:10-99`

`OdooPromoClient` class:
- **Authentication:** XML-RPC to `{url}/xmlrpc/2/common`
- **Data fetch:** Paginates through `product.pricelist.item` records via `web_search_read`
- **Barcode enrichment:** For each pricelist item, fetches `product.template` to get barcode
- **Filtering:** Optionally filters by `pricelist_type == "discount_pricelist"`
- **Configuration:** `url`, `db`, `username`, `password`, `limit`, `offset`, `promo_uses_discount_pricelist` from `promo_settings` dict

### Management Commands

| Command | Retailer | Key Args |
|---------|----------|----------|
| `flying_tiger_promotions_import` | Flying Tiger | `--definitions-file`, `--mappings-file` |
| `create_staff_discount` | Generic | `store_type`, `retailer_store_ids`, `discount_percentage`, `group_qualifier_id`, `--categories`, `--subcategories` |
| `SAP_promotions_import` | SAP | `filepath`, `promo_type` (promotxn/mixmatch), `store_type`, `retailer_store_id` |
| `grandiose_promotion_import` | Grandiose | Store and path configuration |
| `shopify_promotion_import` | Shopify | Shopify API integration (55KB, complex) |
| `dubai_duty_free_promotions_import` | Dubai Duty Free | DDF-specific format |
| `dubai_duty_free_special_customer_promotions` | Dubai Duty Free | Special customer promos |
| `d365_promotions_import` | D365 | Dynamics 365 integration |
| `apos_plus_promotions_import` | AposPlus | AposPlus format |
| `kikimart_promo_import` | KikiMart | KikiMart format |
| `mmi_promotions_import` | MMI | MMI retailer format |
| `walmart_promo_import` | Walmart | Walmart format |

### Coupon Creation Utility

**Source:** `promotions/management/utils/coupons.py:1-130`

`import_coupons(coupons, store, create_coupons)` orchestrates:
1. Generates `unique_id`: `"{chain}-{region}-{title_with_hyphens}"`
2. If `create_coupons=True`: POSTs to `{MS_LOYALTY_SERVICE}/coupon/v1/create_retailer_coupons/` with hardcoded dates (`2023-11-20` to `2050-11-20`), unlimited usage counts (`111111111`)
3. Creates a promotion via `Promotion.from_json()` → `PromotionBatchOperation.delete()` → `create()` → `commit()`. Uses family `"b"` (basket threshold), `availability="s"` (special), `layer="100"`, `discount_type="p"` (percent-off).

---

## Relationship to Promotions Microservice

The three modules interact with `service.promotions` in distinct ways:

| Module | Interaction | Direction |
|--------|-------------|-----------|
| `mishipay_coupons` | Does **not** directly call the microservice. Stores promo qualifiers in `basket.special_promos` which are later sent to the microservice during basket evaluation. | Indirect |
| `mishipay_special_promos` | Same pattern — saves qualifiers to `basket.special_promos`. The `PromotionHelper` class builds promotion definitions compatible with the microservice format. `get_special_promos_info_for_promo_evaluate()` transforms the basket data for the evaluation API. | Indirect (data prep) |
| `promotions` | Directly creates/deletes promotion definitions in the microservice via `PromotionBatchOperation` HTTP calls. This is the **data loading** path — it populates the microservice's database. | Direct (write) |

**Flow of a coupon discount:**
1. User enters coupon code → `mishipay_coupons` validates via MS Loyalty → saves `promo_group_qualifier` to `basket.special_promos`
2. Basket evaluation triggered → `mishipay_special_promos.get_special_promos_info_for_promo_evaluate()` transforms data
3. Platform sends basket + `special_promos_info` to promotions microservice → microservice matches qualifiers to pre-loaded promotions → returns discount amounts
4. Order completion → `mishipay_coupons.redeem_coupons_helper()` redeems via MS Loyalty

See [Promotions Service — Business Logic](../promotions-service/business-logic.md) for the microservice evaluation engine, and [Items Module](./items.md) for the basket evaluation pipeline (`mpay_promo.py`).

---

## Configuration

| Setting | Source | Purpose |
|---------|--------|---------|
| `settings.MS_LOYALTY_SERVICE` | Django settings | Base URL for the external loyalty/coupon microservice |
| `GlobalConfig["COUPON_TITLE_ORDERING"]` | Database | Dict mapping `store_type` → ordered list of coupon titles for display sorting |
| `SPECIAL_PROMO_PREREQUISITES_FUNCTION_MAP` | `app_settings.py` | Maps store type (lowercase) → prerequisite check function |
| `STORE_TYPE_OVERRIDE_REFERRAL_COUPON` | `coupons_helper.py:28` | Set of store types that can override referral coupons (`{"RotanaStoreType"}`) |

---

## Notable Patterns

1. **No local models** — All three modules are stateless orchestration layers. `mishipay_coupons` and `mishipay_special_promos` have `models.py` files with no models defined. Promotion data lives in the microservice; coupon data lives in the loyalty service; transient state lives in `Basket.special_promos` JSON field.

2. **Retailer-specific branching** — Validation logic contains hardcoded store type checks: MUJI (coupon stacking prevention, kiosk uppercase, first-order referral exclusions), Carters (delegated to `CartersPromo` client), Rotana (referral override), Seventh Heaven (advance payment coexistence).

3. **Dual validation entry points** — `ValidateSpecialPromo` dispatches COUPON types to `mishipay_coupons` and all other types to `mishipay_special_promos`. This means coupon validation can be triggered from either module.

4. **Pipeline pattern for imports** — `PromotionImport` ABC provides a clean ETL abstraction. All retailer importers follow: parse file → clean/transform with pandas → map to `Promotion` objects → push via `PromotionBatchOperation`.

5. **Hardcoded dates in import utilities** — Several import paths use hardcoded dates (e.g., `2023-11-20T00:00:00+0000` to `2050-11-20T23:59:59+0000` in `coupons.py:74-75`). The staff discount module uses `now()` to `now() + 4 years`.

6. **Delete-then-create pattern** — Most import operations delete existing promotions before creating new ones, rather than performing incremental updates. Exception: SAP module implements node-level diffing.

7. **`is_mishipoly_promo_applicable()` always returns False** — The function at `special_promo_helper.py:163` is disabled. A more complex version (`is_mishipoly_promo_applicable2`) exists at line 166 with cross-retailer loyalty logic checking for `MPLDN1000` qualifier, but it's unclear if this is actively called.

---

## Open Questions

- Is `is_mishipoly_promo_applicable2()` actively called from any code path, or is it dead code alongside `is_mishipoly_promo_applicable()`? The cross-retailer loyalty concept (two orders from different store types in a cycle) seems unused.
- The `get_basket_level_promotions_label_details()` function has a TODO hack for BWG orders with missing `group_qualifiers` data (`special_promo_helper.py:447`). Is this still needed, or have all missing receipts been sent?
- `RetailerCoupons.get()` (`views.py:110-121`) does not return a `Response` when the loyalty service returns a non-200 status — only the 200 path has explicit ordering logic. The fallthrough `return Response(...)` at line 121 handles this, but the intermediate path at line 118 could mask errors if `available_coupons` filtering empties the list.
- The Shopify promotion import command (`shopify_promotion_import.py`) is 55KB — significantly larger than other commands. What Shopify API integration complexity does this handle?
- Several management commands use `print()` instead of `self.stdout.write()` or proper logging (e.g., `coupons.py:28`). Is this intentional for operator feedback during manual imports?
- The `emmas_garden_loyalty_import` command creates 4 coupon types based on discount/tax combinations. Is Emma's Garden still an active retailer, or is this command legacy?
- `STORE_TYPE_OVERRIDE_REFERRAL_COUPON` contains only `RotanaStoreType`. Is this set expected to grow, or is this a one-off override?
