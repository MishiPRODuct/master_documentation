# Inventory Module

> Last updated by: Task T27 — Inventory Module (119 files)

## Overview

The `inventory` Django app serves as the monolith's interface to an external **Node.js Inventory Service** (`node-express-inventory-service`). Rather than managing product catalogs directly in the monolith database, this module acts as a **proxy layer** for real-time inventory queries and provides a **batch import pipeline** for loading retailer inventory files into both the monolith's legacy `ItemInventory` model and the external Inventory Service.

The module has two distinct responsibilities:
1. **API Proxy** — 7 REST endpoints that forward requests to the Inventory Service and apply post-processing (sorting, category ordering)
2. **Batch Import Pipeline** — 35+ management commands that parse retailer-specific inventory files (CSV, TSV, WooCommerce API) and upsert items into the monolith database

The batch import pipeline relies on the [`inventory-common` shared library](./inventory-common.md) (`python-inventory-common`), which provides the `InventoryV1Client` HTTP client, a pandas-based ETL framework, and 44+ retailer-specific import scripts.

## Key Concepts

| Term | Definition |
|------|-----------|
| **Inventory Service** | External Node.js microservice (`node-express-inventory-service`) that stores the canonical product catalog in MongoDB. Accessed via `INVENTORY_SERVICE_URL` (default: `http://inventory:8000`) |
| **`inventory-common`** | Private Python package (`python-inventory-common`, v2.116.2) providing `InventoryV1Client`, enums (`SellingGuidanceType`, `BuyingGuidanceType`, `Theme`), and import utilities (`to_decimal`). Installed from GitHub |
| **`ItemInventory`** | Legacy monolith model (`mishipay_items.models.ItemInventory`) for storing product data in PostgreSQL. Used by retailers that haven't migrated to the Inventory Service |
| **`ItemCategory`** | Legacy monolith model for product categories, linked to stores. Items belong to exactly one category |
| **Delta vs Full Import** | Delta imports (`delta=True`) only update/create items in the file, leaving other items untouched. Full imports (`delta=False`) delete all items for the store not present in the file |
| **Reduce to Clear (RTC)** | Avery Dennison markdown barcode format (`MP` prefix, 25 chars) encoding a barcode + discount (fixed price or percent off) with Luhn checksum validation |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Monolith (Django)                        │
│                                                             │
│  ┌─────────────┐   ┌──────────────────┐   ┌─────────────┐  │
│  │ API Proxy   │   │ Basket Service   │   │ Batch Import│  │
│  │ (views.py)  │   │ (basket_svc.py)  │   │ (mgmt cmds) │  │
│  └──────┬──────┘   └────────┬─────────┘   └──────┬──────┘  │
│         │                   │                     │         │
│         │  HTTP POST        │  InventoryV1Client  │         │
│         ▼                   ▼                     ▼         │
│  ┌──────────────────────────────────┐   ┌───────────────┐  │
│  │   Inventory Service (Node.js)   │   │  PostgreSQL   │  │
│  │   http://inventory:8000         │   │  ItemInventory│  │
│  │   MongoDB backend               │   │  ItemCategory │  │
│  └──────────────────────────────────┘   └───────────────┘  │
│                                                             │
│  ┌──────────────────┐   ┌──────────────────┐               │
│  │ ms-tax (gRPC)    │   │ RetailerImportJob│               │
│  │ Tax table upload │   │ (mishipay_alerts)│               │
│  └──────────────────┘   └──────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

The module operates in three modes:

1. **Proxy Mode** — Client apps call the monolith's `/inventory/v1/*` endpoints, which forward to the Inventory Service and return the response (with optional post-processing)
2. **Service Client Mode** — Other monolith modules (e.g., `mishipay_items` basket procedures) call `basket_service.py` functions to look up items and create `BasketItem` records from Inventory Service data
3. **Import Mode** — Management commands parse retailer files and bulk-create/update `ItemInventory` records in PostgreSQL (for retailers not yet on the Inventory Service)

## Data Models

This module has **no Django models of its own**. It interacts with models from other modules:

| Model | Module | Usage |
|-------|--------|-------|
| `ItemInventory` | `mishipay_items` | Target for batch imports; stores product data (name, price, barcode, category, tax, image) |
| `ItemCategory` | `mishipay_items` | Product categories per store; created during imports |
| `ItemCatalog` | `mishipay_items` | Catalog items (QR code mappings); used by Express Lane |
| `AdditionalBarcode` | `mishipay_items` | Secondary barcodes per item; created by Stellar imports |
| `BasketItem` | `mishipay_items` | Shopping basket items; created by `basket_service.py` from Inventory Service data |
| `Basket` | `mishipay_items` | Shopping basket; FK target for basket items |
| `Store` | `dos` | Store lookup for retailer-store mapping |
| `GlobalConfig` | `mishipay_config` | Category ordering configuration per store |
| `RetailerImportJob` | `mishipay_alerts` | Import job tracking (start/end with status) |

## API Endpoints

All endpoints are **POST-based proxies** to the Inventory Service. No authentication is enforced at the view level.

Source: `inventory/views.py` (131 lines), `inventory/urls.py` (52 lines)

| Method | Path | View Class | Purpose |
|--------|------|-----------|---------|
| POST | `/inventory/v1/category/list-hierarchy/` | `CategoryListHierarchyV1View` | Get category hierarchy for a store. Post-processes: sorts root categories by `GlobalConfig` `kiosk_categories_order`, sorts sub-categories for Rotana/Grandiose via `{chain}_CNC` config |
| POST | `/inventory/v1/item/list/` | `ItemListV1View` | List items for a store. Post-processes: sorts by description for Love's, sorts by stock level + alignment for Grandiose |
| POST | `/inventory/v1/item/search/` | `ItemSearchV1View` | Search items. Same post-processing as item list |
| POST | `/inventory/v1/item/list-filters/` | `ItemListFiltersV1View` | Get available item filters for a store. Pure proxy, no post-processing |
| POST | `/inventory/v1/addon-group/get` | `GetAddonGroupV1View` | Get addon group details by slug. Pure proxy |
| POST | `/inventory/v1/store/create` | `StoreCreateV1View` | Create a store in the Inventory Service. Pure proxy |
| POST | `/inventory/v1/retailer/create` | `RetailerCreateV1View` | Create a retailer in the Inventory Service. Pure proxy |

### Proxy Mechanism

The `post_proxy()` function (`views.py:28-40`) handles all proxying:
1. Looks up the `Store` by `storeId` from request data
2. Resolves the **inventory store** via `get_inventory_store_for_platform(store, platform)` — this allows platform-specific store mapping (e.g., a kiosk may use a different inventory store than the mobile app)
3. If the inventory store differs from the request store, substitutes the `storeId`
4. Forwards the request via `requests.post()` to the Inventory Service
5. Returns the response with original status code and headers

### Post-Processing (Data Preprocessors)

Source: `inventory/inventory_service_data_preprocessors.py` (130 lines)

| Function | Retailer | Purpose |
|----------|----------|---------|
| `sort_items_for_loves_kiosk()` | Love's | Sorts `variants` dict by `description` field (numerically). Documented as "temporary hack for test env" |
| `sort_categories()` | All (config-driven) | Reorders root hierarchy children based on `GlobalConfig` `kiosk_categories_order`. Unmatched categories appended alphabetically |
| `sort_sub_categories()` | Rotana, Grandiose | Reorders sub-categories based on `{chain}_CNC` config's `click_and_collect_categories` |
| `sort_items()` | Grandiose | Sorts items: in-stock first (by `poslogGuidance.alignment`), then out-of-stock |

## Business Logic

### Basket Service (Core Service Layer)

Source: `inventory/inventory_service/services/basket_service.py` (310 lines)

This is the primary interface used by basket procedures throughout the monolith to interact with the Inventory Service.

**Item Lookup Functions:**

| Function | Lines | Purpose |
|----------|-------|---------|
| `get_variant_by_barcode()` | 15–41 | Look up item by barcode (supports comma-separated multi-barcode). Uses `InventoryV1Client` |
| `get_variant_by_retailer_product_id()` | 44–60 | Look up item by retailer product ID |
| `list_variants_by_category_name()` | 63–81 | Paginated list by category name |
| `list_variants_by_barcodes()` | 84–101 | Paginated list by multiple barcodes |
| `list_variants_by_filters()` | 255–276 | Generic filter-based variant listing (barcodes, retailerProductId, categoryName) |

**BasketItem Creation Functions:**

| Function | Lines | Purpose |
|----------|-------|---------|
| `create_basketitem()` | 104–136 | Create a `BasketItem` from a barcode. Fetches variant from Inventory Service, maps via `to_basketitem()`, sets basket/quantity/platform, optionally saves |
| `create_addon_basketitem()` | 139–169 | Create addon `BasketItem` (e.g., bottle deposit) using `buyingGuidance` filters. Marks with `is_deposit_item` and `is_addon_item` flags |
| `create_bulk_basketitems()` | 221–252 | Bulk-create basket items from a barcode-to-quantity map. Uses `list_variants_by_barcodes()` then `bulk_create()` |
| `create_bulk_basketitems_with_epcs()` | 279–309 | Bulk-create basket items from EPC-to-barcode mapping (RFID). Creates one `BasketItem` per EPC with deduplication |

**Stock Checking:**

| Function | Lines | Purpose |
|----------|-------|---------|
| `check_stock_levels()` | 172–218 | Verify all basket items have sufficient stock. Supports `check_only_cnc_items` flag (Click & Collect only). Returns `None` if OK, or list of `{product_identifier, available_quantity, basket_quantity}` for insufficient stock |

### Item Variant Mapper

Source: `inventory/inventory_service/mappers.py` (65 lines)

The `to_basketitem()` function maps an Inventory Service variant dict to a `BasketItem` model instance:

| Inventory Service Field | BasketItem Field |
|------------------------|------------------|
| `name` | `item_name` |
| `retailerProductId` | `retailer_product_id` |
| `description` | `description` |
| `basePrice` | `price`, `modified_base_price` |
| `stockLevel` | `stock_quantity` |
| `images[0]` | `img` |
| `barcodes[0]` | `product_identifier` |
| `categories[0].name` | `item_category` |
| `pricingGuidance.vatPercentage` | `extra_data.vat_rate` |
| `pricingGuidance.excludingVat` | `extra_data.price_without_vat` |
| `pricingGuidance.taxCode` | `tax_code` |
| `buyingGuidance.AGE_RESTRICTION` | `requires_approval=True`, `extra_data.minimum_age_required` |

The remaining variant data is stored in `extra_data.inventory_service` for downstream access. `is_refundable` and `show_item_in_price_overide` (note: typo in source) are also added to `extra_data`.

### Reduce to Clear (RTC) System

Source: `inventory/reduce_to_clear/` (4 files, ~200 lines)

Handles Avery Dennison "Reduce to Clear" markdown barcodes — special 25-character barcodes that encode a product barcode plus a discount.

**Barcode Format:**
```
MP[BarcodeType][14-digit padded barcode][ValueId][5-digit value][checksum]
│  │            │                        │        │              └── Luhn
│  │            │                        │        └── Price/percent × 100
│  │            │                        └── P=percent off, A=fixed price
│  │            └── Original barcode, left-padded with zeros
│  └── 1=EAN-13, 2=EAN-8, 3=UPC-A, 4=UPC-E, 5=EAN-14
└── Fixed prefix "MP"
```

**Key Functions:**

| Function | File | Purpose |
|----------|------|---------|
| `is_reduce_to_clear_barcode()` | `utils.py:38` | Validate: 25 chars, `MP` prefix, valid barcode type, valid value identifier, Luhn checksum |
| `parse_reduce_to_clear_barcode()` | `utils.py:52` | Parse into `(barcode, value_identifier, value)` tuple |
| `has_valid_luhn_checksum()` | `utils.py:14` | Luhn algorithm validation |
| `get_rtc_variant_by_barcode()` | `basket_service.py:31` | Look up real item, apply RTC pricing, add `originalPrice` key |
| `create_rtc_basketitem()` | `basket_service.py:59` | Create basket item with RTC-modified `modified_base_price` |
| `_calculate_rtc_price()` | `basket_service.py:16` | `FIXED` → use value directly; `PERCENT_OFF` → `(100-value)/100 × originalPrice` |

### Bulk Update/Create Pattern

Three variants of the same bulk upsert algorithm exist across the module:

| File | Unique Key | Extra Behavior |
|------|-----------|----------------|
| `utils/db.py` (77 lines) | `retailer_product_id` | Base pattern |
| `nude_foods/db.py` (77 lines) | `product_identifier` | Identical logic, different key field |
| `stellar/db.py` (121 lines) | `product_identifier` | Adds `AdditionalBarcode` creation + tax table upload via `TaxClientSelector` |

**Algorithm** (shared across all three):
1. Validate uniqueness of key field using NumPy
2. Query existing items by key → build `{key: pk}` map
3. If `delta=False`: delete all store items NOT in the incoming set
4. Split incoming items into update set (existing PKs) and create set (new)
5. `bulk_update()` existing items (batch size 20,000)
6. `bulk_create()` new items (batch size 20,000)

Note: The `breeze_thru/database_service.py` has a simpler version that only supports `bulk_create()` (no update), with a TODO to add update support.

## Client Modules (Retailer-Specific Import Logic)

Each client module follows a consistent ETL pattern:

```
[Retailer File] → Parser (PromotionImport ABC) → Mapper (DataFrame → Model) → DB Service (bulk upsert)
```

### Module Structure Pattern

Each module typically contains:
- `constants.py` — Column names, default values, mappings
- `parsers.py` — File parsing classes extending `PromotionImport` ABC (extract, clean, transform)
- `mappers.py` — Functions mapping DataFrames to Django model instances
- `database_service.py` — Bulk create/update functions
- `main.py` — Orchestrator function tying it all together
- `tests/` — Unit tests for each layer

### Breeze Thru

Source: `inventory/breeze_thru/` (5 source files + 4 test files)

| Aspect | Details |
|--------|---------|
| Store Type | `BreezeThruStoreType` |
| Input Files | Items CSV (11 columns: Product ID, UPC, Department ID, Description, Price, etc.) + Categories CSV (4 columns: Department, Pricebook Export Code, Tax Level, Tax %) |
| Parser | `BreezeThruItemsImport`, `BreezeThruCategoriesImport` (both extend `PromotionImport`) |
| Special Logic | UPC-A barcode normalization (zero-padding + checksum), age-restricted categories (Beer/Wine, Cigarettes, OTP), zero-price item removal |
| Import Mode | Full only (clean flag deletes all categories first) |
| DB Target | `ItemCategory` + `ItemInventory` in monolith PostgreSQL |
| Tax | Per-category tax codes from categories file, merged into items via Pandas join |

### Express Lane

Source: `inventory/express_lane/` (6 source files)

| Aspect | Details |
|--------|---------|
| Store Type | `ExpressLaneStoreType` |
| Input Files | Items CSV + optional Item Catalog CSV |
| Special Logic | Tax table upload via `TaxClientSelector` (gRPC to ms-tax), item catalog with QR codes for fountain/hot drink categorization |
| Two Import Functions | `express_lane_inventory_import()` — standard items/categories import; `express_lane_item_catalogue_import()` — QR-to-barcode catalog mapping with `CATALOG-FOUNTAIN`/`CATALOG-HOTDRINK` sub-categories |
| DB Target | `ItemCategory` + `ItemInventory` + `ItemCatalog` |

### Muji

Source: `inventory/muji/` (5 source files + 4 test files + 1 CSV test data)

| Aspect | Details |
|--------|---------|
| Store Type | `MujiStoreType` |
| Input Files | UTF-16 encoded CSV (18 columns including colour codes, sizes, web links) |
| Parser | `MujiMonolithImport` — extensive data cleaning (strip whitespace, validate barcodes ≥8 digits, validate decimal prices, validate restricted item flags) |
| Special Logic | 95 colour name → hex code mappings in `constants.py`, category code stripping (leading digit removal), restricted item boolean mapping, colour/size/vat stored in `extra_data` |
| Import Mode | Full (`delta=False`) with `@transaction.atomic` |
| Job Tracking | Uses `start_retailer_import_job()` / `end_retailer_import_job()` from `mishipay_alerts` |

### Stellar

Source: `inventory/stellar/` (6 source files, **no tests**)

| Aspect | Details |
|--------|---------|
| Store Type | `StellarStoreType` |
| Input Files | 3 TSV files (ISO-8859-1 encoding): items, scan codes, tax rates |
| Parser | `StellarItemImport`, `StellarScanCodeImport`, `StellarTaxImport` — all tab-separated |
| Special Logic | Multi-store support (single file contains multiple `POSStoreID` values), merchandise hierarchy extraction (5 promo category codes → structured hierarchy), `AdditionalBarcode` creation post-import, tax table upload via `TaxClientSelector` |
| Composite Store ID | `{PSDeptID}-{POSStoreID}` format |
| Import Mode | Delta (`delta=True`) with per-store `transaction.atomic` |
| Job Tracking | Uses `start_retailer_import_job()` / `end_retailer_import_job()` |

### Nude Foods

Source: `inventory/nude_foods/` (3 source files + 1 test file + 2 JSON test data files)

| Aspect | Details |
|--------|---------|
| Store Type | `NudeFoodStoreType` |
| Data Source | **WooCommerce V3 API** (not file-based) — `WooCommerceV3Client` fetches products/variations with auto-pagination |
| Special Logic | WooCommerce credentials stored in `store.extra_data`, product variant support (parent+child variants), bottle deposit items created from `product-fee-name`/`product-fee-amount` meta data, donation item auto-creation ($1.00 Regenerative Agriculture donation) |
| Import Mode | Always delta (`delta=True`) — deposit items created separately |
| DB Target | `ItemCategory` + `ItemInventory` in monolith PostgreSQL |

### WooCommerce Client

Source: `inventory/woocommerce/client.py` (73 lines)

Generic `WooCommerceV3Client` class used by the Nude Foods module:
- Endpoints: `list_products()`, `list_product_variations()`, `list_tax_rates()`
- `depaginate()` method: auto-paginates through all results (100 per page max)
- Custom User-Agent (`WooCommerce-Python-REST-API/3.0.0`) to avoid blocks
- JSON response logging via `json_log_factory`

## Management Commands

Source: `inventory/management/commands/` (35 Python files + 5 test files + 13 mock data files)

### Commands Summary

| Command | Retailer | Data Source | Key Module |
|---------|----------|-------------|------------|
| `breeze_thru_inventory_import` | Breeze Thru | CSV | `inventory.breeze_thru.main` |
| `express_lane_inventory_import` | Express Lane | CSV | `inventory.express_lane.main` |
| `muji_inventory_import_new` | Muji | CSV (UTF-16) | `inventory.muji.main` |
| `muji_inventory_service_import` | Muji | CSV | `inventory_scripts.muji.main` (external) |
| `stellar_inventory_import` | Stellar | TSV × 3 | `inventory.stellar.main` |
| `nude_foods_inventory_import` | Nude Foods | WooCommerce API | `inventory.nude_foods.main` |
| `dubai_duty_free_inventory_import` | Dubai Duty Free | CSV | `inventory_scripts.dubai_duty_free.main` (external) |
| `dubai_duty_free_delta_inventory_import` | Dubai Duty Free | CSV (delta) | External |
| `dubai_duty_free_inventory_files_process` | Dubai Duty Free | File processing | External |
| `whsmith_inventory_service_import` | WHSmith | DCN/MNT files | `inventory_scripts.wh_smith.main` (external) |
| `flying_tiger_inventory_service_import` | Flying Tiger | CSV | `inventory_scripts.flying_tiger.main` (external) |
| `flying_tiger_azadea_inventory_service_import` | Flying Tiger Azadea | CSV | External |
| `eroski_inventory_service_import` | Eroski | CSV (multiple) | `inventory_scripts.eroski.main` (external) |
| `eroski_inventory_service_alcohol_import` | Eroski | CSV (alcohol) | External |
| `virgin_megastore_inventory_service_import` | Virgin Megastore | CSV | External |
| `virgin_megastore_update_images_import` | Virgin Megastore | Image update | External |
| `paradies_inventory_service_import` | Paradies | CSV | External |
| `paradies_update_deposit_items_import` | Paradies | Deposit items | External |
| `seventh_heaven_inventory_import` | Seventh Heaven | CSV | External |
| `ssp_inventory_service_import` | SSP | CSV | External |
| `mlse_inventory_service_import` | MLSE | CSV | External |
| `rotana_inventory_service_import` | Rotana | CSV | External |
| `grandiose_inventory_import` | Grandiose | CSV | External |
| `d365_inventory_service_import` | D365 | CSV | External |
| `goloso_inventory_service_import` | Goloso | CSV | External |
| `badiani_inventory_service_import` | Badiani | CSV | External |
| `funky_buddha_inventory_import` | Funky Buddha | CSV | External |
| `funky_buddha_images_fetch` | Funky Buddha | Image fetch | External |
| `kiko_milano_azadea_inventory_import` | Kiko Milano Azadea | CSV | External |
| `king_college_hospital_inventory_import` | King's College Hospital | CSV | External |
| `flash_facial_inventory_import_command` | Flash Facial | CSV | External |
| `flash_facial_inventory_files_download_command` | Flash Facial | File download | External |
| `standard_cafe_inventory_service_import` | Standard Cafe | CSV | External |
| `addon_excel_import` | Generic | CSV/Excel | `inventory_scripts.addons_upload.main` (external) |
| `associate_addon_groups_with_items` | Generic | CSV | `inventory_scripts.associate_addon_groups_with_items.main` (external) |
| `create_inventory_service_stores` | Generic | Database | Internal (creates stores in Inventory Service) |

### Two Import Patterns

**Pattern A — Monolith Import** (Breeze Thru, Express Lane, Muji monolith, Stellar, Nude Foods):
- Parser/mapper/DB service code lives **inside** `inventory/` module
- Data written to monolith `ItemInventory`/`ItemCategory` tables
- Commands use `--retailer-store-id` as primary identifier

**Pattern B — Inventory Service Import** (most other retailers):
- Logic lives in `inventory_scripts.*` packages (from `inventory-common` pip package)
- Data written to external Inventory Service via REST API
- Commands typically use `--store-id` and `--retailer-id` (UUIDs)

### Test Coverage

| Test File | Lines | Coverage |
|-----------|-------|---------|
| `tests/test_views.py` | ~35K | All 7 API proxy views, store substitution, Love's/Grandiose sorting, category ordering, sub-category ordering |
| `tests/test_urls.py` | 20 | URL resolution for 4 core endpoints |
| `management/tests/test_dubai_duty_free_inventory_import.py` | ~26K | DDF import command |
| `management/tests/test_seventh_heaven_inventory_import.py` | ~20K | Seventh Heaven import |
| `management/tests/test_addon_excel_import.py` | ~14.5K | Addon import |
| `management/tests/test_mlse_inventory_service_import.py` | ~3K | MLSE import |
| `management/tests/test_breeze_thru_inventory_import.py` | ~900 | Breeze Thru import (basic) |
| `breeze_thru/tests/` | 4 files | Parsers, mappers, database service, main function |
| `muji/tests/` | 4 files + 1 CSV | Parsers, mappers, database service, main function |
| `nude_foods/tests/` | 1 file + 2 JSON | Main function with mock WooCommerce data |
| `inventory_service/tests/` | 2 files | Mapper (variant → BasketItem) + basket service (~14K lines) |
| `reduce_to_clear/tests/` | 2 files | RTC barcode parsing + basket service |
| `utils/tests/` | 2 files | Barcode checksum + bulk DB operations |

## Shared Utilities

Source: `inventory/utils/` (3 files)

| File | Function | Purpose |
|------|----------|---------|
| `barcode.py` | `upc_a_add_checksum()` | Calculate and append UPC-A checksum to 11-digit barcode |
| `db.py` | `bulk_update_or_create_items()` | Bulk upsert `ItemInventory` records using NumPy for set operations (by `retailer_product_id`) |
| `logging.py` | `log_time()` | Decorator that logs function execution time |

### Testing Utilities

Source: `inventory/testing.py` (52 lines), `inventory/conftest.py` (9 lines)

`create_flat_item_variant()` generates fake Inventory Service variant dicts using Faker for testing:
- Random MongoDB-style ObjectId
- `SellingGuidanceType` modes: SCAN_PAY_GO, CLICK_COLLECT, DELIVERY
- Random pricing, barcodes (EAN-13), images, stock levels
- Used by the `item_variant` pytest fixture in `conftest.py`

## Dependencies

### This Module Depends On

| Module | Usage |
|--------|-------|
| `dos.models.Store` | Store lookup by `store_id`, `store_type`, `retailer_store_id` |
| `mishipay_items.models` | `BasketItem`, `Basket`, `ItemInventory`, `ItemCategory`, `ItemCatalog`, `AdditionalBarcode` |
| `mishipay_core.common_functions` | `get_inventory_store_for_platform()` (platform-specific store mapping), `get_rounded_value()` |
| `mishipay_config.models.GlobalConfig` | Category ordering config |
| `mishipay_config.utils` | `get_client_config_key()` |
| `mishipay_alerts` | Import job tracking (`start_retailer_import_job()`, `end_retailer_import_job()`, `get_file_stats()`) |
| `mishipay_retail_orders` | `TaxClientSelector`, `TaxRow` for tax table uploads |
| `promotions.abc.PromotionImport` | Base class for file parsers (ETL pipeline) |
| `promotions.utils` | `ensure_column_unique()` |
| `inventory-common` (pip) | `InventoryV1Client`, enums, `to_decimal()` utility |
| `lib.logging.requests` | `json_log_factory()` for HTTP request/response logging |

### What Depends On This Module

| Module | Usage |
|--------|-------|
| `mishipay_items` basket procedures | Call `basket_service.py` functions to create basket items from Inventory Service data |
| `mishipay_items` views | Call `reduce_to_clear.basket_service` for RTC barcode handling |

## Configuration

| Setting | Source | Default | Purpose |
|---------|--------|---------|---------|
| `INVENTORY_SERVICE_PROTOCOL` | Env var | `http` | Protocol for Inventory Service |
| `INVENTORY_SERVICE_HOST` | Env var | `inventory` | Host for Inventory Service |
| `INVENTORY_SERVICE_PORT` | Env var | `8000` | Port for Inventory Service |
| `INVENTORY_SERVICE_URL` | Computed | `http://inventory:8000` | Full URL used by views and client |
| `INVENTORY_SERVICE_OPS_URL` | Computed | Same as above | Operations URL (same value) |
| `GlobalConfig[{client_config_key}].kiosk_categories_order` | DB | `[]` | Category display ordering per store |
| `GlobalConfig[{chain}_CNC].click_and_collect_categories` | DB | `{}` | Sub-category ordering for Click & Collect |

## Notable Patterns

1. **PromotionImport Reuse** — File parsers reuse the `PromotionImport` ABC from the `promotions` module, even though these imports are for inventory, not promotions. This provides a ready-made ETL pipeline (extract → clean → transform) with column validation, deduplication, and default value handling.

2. **NumPy for Set Operations** — The bulk upsert functions use `numpy.where()`, `numpy.isin()`, and `numpy.unique()` for splitting items into update/create sets, rather than Python sets. This is efficient for large item lists.

3. **Duplicated `bulk_update_or_create_items()`** — Three nearly identical implementations exist in `utils/db.py`, `nude_foods/db.py`, and `stellar/db.py`, differing only in the unique key field (`retailer_product_id` vs `product_identifier`) and post-commit hooks. These should ideally be parameterized into a single function.

4. **Dual Import Targets** — Some retailers import to the monolith DB (Pattern A), others to the Inventory Service (Pattern B via `inventory_scripts`). The module is in transition — newer retailers use the Inventory Service, while legacy retailers still use `ItemInventory`.

5. **`print()` in Production** — `muji/main.py:44` and `stellar/main.py:53-55` use `print()` instead of the logging framework for import progress messages.

6. **No Authentication on API Endpoints** — All 7 proxy views inherit from `APIView` with no `authentication_classes` or `permission_classes`. Security is presumably handled at the API gateway (Kong) level.

7. **WooCommerce Client User-Agent Workaround** — The client sets a custom `WooCommerce-Python-REST-API/3.0.0` User-Agent because the default `requests` User-Agent is blocked by the WooCommerce server.

8. **`is` Comparison on Price Override Typo** — The mapper sets `show_item_in_price_overide: True` (note "overide" typo) in `extra_data`. This key name is used downstream and cannot be corrected without updating all consumers.

## Open Questions

1. **Duplicated bulk upsert functions** — Why do `utils/db.py`, `nude_foods/db.py`, and `stellar/db.py` have near-identical `bulk_update_or_create_items()` implementations? Could these be consolidated into a parameterized function?

2. **`sort_items_for_loves_kiosk()` is documented as a "temporary hack for test env"** (`inventory_service_data_preprocessors.py:8`). Is this still in use or should it be removed?

3. **Stellar module has no tests** — Unlike other client modules, `stellar/` has no `tests/` directory and no `__init__.py`. Is this intentional or an oversight?

4. **`breeze_thru/database_service.py` only supports create, not update** (TODO at line 25). Has this been addressed or are Breeze Thru imports always run with `--clean`?

5. **Pattern A vs Pattern B migration** — Is there a plan to migrate all Pattern A retailers (monolith DB imports) to Pattern B (Inventory Service imports)? What's the timeline?

6. **`StellarItemImport` and `StellarTaxImport` docstrings say "Import Muji"** — copy-paste artifacts from the Muji module. Should be "Import Stellar".

7. **`express_lane_item_catalogue_import()`** reads CSV with manual line parsing (`open(filepath, 'r')` with `split(",")`) instead of using pandas or the `PromotionImport` pattern. Is this intentional?

8. **No retry or timeout on `post_proxy()`** — The proxy `requests.post()` call (`views.py:37`) has no timeout parameter. If the Inventory Service is slow or unresponsive, the request will hang indefinitely.

9. **Nude Foods `donation_item` uses `get_or_create()` with many fields** — This may fail to match existing donation items if any field changes slightly (e.g., `description` update). Should the lookup fields be narrowed?
