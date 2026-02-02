# Inventory Common Library (`python-inventory-common`)

> Last updated by: Post-T39 addition — New repository documented

## Overview

`inventory-common` (v2.116.3) is a shared Python library providing the HTTP client, ETL framework, and 44+ retailer-specific import scripts for MishiPay's Inventory Service. It is the bridge between dozens of retailer POS/ERP/e-commerce systems and MishiPay's canonical product catalog stored in the Node.js Inventory Service (MongoDB).

The library is installed as a private dependency from GitHub (`mishipay-ltd/python-inventory-common`) and used by the monolith's [`inventory` module](./inventory.md) via its management commands. It is **not** a Django app — it's a standalone Python package with no Django dependency.

**Repository:** `mishipay-ltd/python-inventory-common`
**Stats:** 321 Python files, ~47,000 lines of code

---

## Architecture

The library contains three packages, each serving a distinct layer:

```
Retailer POS/ERP/E-Commerce Systems
    │
    │  SOAP, REST, ODBC, CSV, JSON, XML, Shopify API, Odoo API, ...
    ▼
┌────────────────────────────────────────────────────────────────┐
│  inventory_scripts/          44+ retailer implementations      │
│  (fetch + parse retailer data into pandas DataFrames)          │
└──────────────────────────────┬─────────────────────────────────┘
                               │ pd.DataFrame
                               ▼
┌────────────────────────────────────────────────────────────────┐
│  inventory_import/           ETL pipeline framework            │
│  (normalize → clean → transform → map to API payload → chunk) │
└──────────────────────────────┬─────────────────────────────────┘
                               │ JSON payload
                               ▼
┌────────────────────────────────────────────────────────────────┐
│  inventory_service/          HTTP client (InventoryV1Client)   │
│  (POST to Inventory Service REST API)                          │
└──────────────────────────────┬─────────────────────────────────┘
                               │ HTTP
                               ▼
┌────────────────────────────────────────────────────────────────┐
│  Inventory Service           Node.js / Express / MongoDB       │
│  (node-express-inventory-service)                              │
└────────────────────────────────────────────────────────────────┘
```

---

## Package 1: `inventory_service` — HTTP Client

**Location:** `src/inventory_service/`

A thin HTTP client wrapping all Inventory Service REST endpoints. Uses `requests.Session` with retry logic (max 2 retries) and structured JSON request/response logging.

### Client API

**Source:** `client.py` — `InventoryV1Client`

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `create_items(items_request)` | `POST /v1/inventory/import` | Bulk create items (full import) |
| `update_inventory(inventory_request)` | `POST /v1/inventory/update` | Delta import — upsert/remove items |
| `clear_items(store_id)` | `POST /v1/item/delete-all` | Delete all items for a store |
| `clear_categories(store_id, retailer_id)` | `POST /v1/category/delete-all` | Delete all categories for a store |
| `list_items(store_id, filters, sort, limit, previous, next)` | `POST /v1/item/list` | List items with filtering and cursor pagination |
| `stock_check(store_id, barcodes)` | `POST /v1/item/stock` | Check stock levels for barcodes |
| `create_retailer(retailer_id)` | `POST /v1/retailer/create` | Register a new retailer |
| `create_store(retailer_id, store_id)` | `POST /v1/store/create` | Register a new store |
| `addon_import(addon_request)` | `POST /v1/addon-group/create` | Create addon groups |
| `get_addon_groups(store_id, slugs, fetch_items)` | `POST /v1/addon-group/get` | Fetch addon groups by slug |
| `get_addon_groups_list(request_data)` | `POST /v1/addon-group/list` | List all addon groups |
| `update_addon_group_on_existing_group(addon_request)` | `POST /v1/addon-group/update` | Update an existing addon group |

**Convenience methods** for single-variant lookup (merge item + variant data):

| Method | Looks up by | Raises |
|--------|------------|--------|
| `get_variant_by_barcode(store_id, barcode)` | Barcode | `VariantNotFoundError`, `MultipleVariantsFoundError` |
| `get_variant_by_retailer_product_id(store_id, id)` | Retailer product ID | `VariantNotFoundError`, `MultipleVariantsFoundError` |
| `get_variant_by_id(store_id, item_variant_id)` | Item variant ID | `VariantNotFoundError`, `MultipleVariantsFoundError` |
| `get_variant_by_filters(store_id, filters)` | Arbitrary filter dict | `VariantNotFoundError`, `MultipleVariantsFoundError` |
| `list_variants_by_filters(store_id, filters, ...)` | Arbitrary filters (multi-result) | `VariantNotFoundError` |
| `list_variants_by_category_name(store_id, name)` | Category name | `VariantNotFoundError` |
| `list_variants_by_barcodes(store_id, barcodes)` | Multiple barcodes | `VariantNotFoundError` |

### Enums

**Source:** `enums.py`

| Enum | Values | Purpose |
|------|--------|---------|
| `BuyingGuidanceType` | `RESTRICTED_ITEM`, `AGE_RESTRICTION`, `MAX_BASKET_QUANTITY`, `ADDON_ITEM`, `REQUIRES_ADDITIONAL_DATA` | Item purchase restrictions |
| `SellingGuidanceType` | `SCAN_PAY_GO`, `CLICK_COLLECT`, `DELIVERY` | Available selling modes |
| `Theme` | `INVARIANT`, `COLOUR_SIZE`, `COLOUR_HEX_SIZE`, `ADDON` | Item variant display themes |
| `OperationType` | `REMOVE`, `UPSERT` | Delta import operation types |
| `PoslogGuidanceType` | `LAG_ITEM` | POS log item guidance |

### Exceptions

| Exception | When |
|-----------|------|
| `InventoryError` | Base exception for all inventory errors |
| `VariantNotFoundError` | Filter search returns 0 results |
| `MultipleVariantsFoundError` | Filter search returns >1 result |

### Request Logging

**Source:** `requests_utils.py`

The `json_log_factory()` function creates a `requests.Session` hook that logs formatted JSON request/response bodies on every HTTP call. Successful calls log at `DEBUG`; failed calls log at `ERROR` and optionally raise `HTTPError`.

---

## Package 2: `inventory_import` — ETL Pipeline

**Location:** `src/inventory_import/`

A declarative, pandas-based ETL framework for normalizing retailer data files into the standardized inventory API format.

### DataImport Base Class

**Source:** `data_import_base.py`

The abstract base class that all retailer parsers extend. Defines a 4-phase pipeline:

```
extract() → clean() → set_defaults() → transform()
```

Subclasses configure the pipeline by setting class attributes:

| Attribute | Type | Purpose |
|-----------|------|---------|
| `column_names` | `Iterable[str]` | Columns to read from the source file |
| `required_columns` | `List[str]` | Columns that must have values (rows with nulls are dropped) |
| `unused_columns` | `List[str]` | Columns to drop after extraction |
| `unique_columns` | `Iterable[str]` | Columns to deduplicate on |
| `default_values` | `Mapping[str, str]` | Default values for null cells |
| `new_column_names` | `Mapping[str, str]` | Column rename mapping (`{old: new}`) |
| `skiprows` | `int` or `Iterable` | Rows to skip during extraction |
| `encoding` | `str` | File encoding (default: `utf-8`) |
| `use_headers` | `bool` | Use first row as headers (default: `True`) |
| `extract_args` | `Dict` | Additional kwargs passed to `pd.read_csv()` |

The pipeline is **lazy** — transformation only runs when `.data` is accessed:

```python
parser = RetailerParser("items.csv")
df = parser.data  # triggers extract → clean → set_defaults → transform
```

### Standard Import Schema

**Source:** `inventory_service_standard_import.py`

`InventoryServiceStandardImport` and its siblings (`AddonThemedInventoryServiceImport`, `InvariantAddonThemedInventoryServiceImport`) provide the standard column mapping:

```
item_name, item_image_url, retailer_product_code, retailer_barcode,
category_a, category_b, category_c, price_with_vat, price_without_vat,
vat_percent, description, stock_level, groupingKey, variation_theme,
variation_kind_1, variation_kind_2, clickcollect, tax_code
```

The transform phase handles:
- Barcode splitting (comma-separated → list)
- Category hierarchy construction (A → B → C parent chain)
- Variant grouping by `groupingKey` (items with same key become variants)
- Theme assignment (`INVARIANT` for single variants, `COLOUR_SIZE`/`ADDON` for grouped)
- Buying guidance (age restrictions)
- Selling guidance (scan-pay-go vs click-collect modes)
- Pricing guidance (VAT, tax codes)

### Import Orchestration

**Source:** `inventory_request_service.py`

Two main entry points:

| Function | Purpose |
|----------|---------|
| `import_inventory(url, store_id, retailer_id, items_df, delta, clean)` | Standard import — optionally clears existing data, maps DataFrame to API payload, calls client |
| `import_inventory_in_chunks(url, store_id, retailer_id, items_df, delta, clean, chunk_size=50000)` | Same but splits large datasets into 50K-row chunks |

Additional functions for addon management: `create_addon_group()`, `update_addon_group()`, `get_addon_groups()`.

### DataFrame → API Payload Mapper

**Source:** `mappers.py`

`map_items_df_to_inventory_request()` converts a pandas DataFrame into the JSON payload expected by the Inventory Service:

```json
{
  "storeId": "uuid",
  "retailerId": "uuid",
  "performInserts": true,
  "categories": [{"name": "...", "image": "...", "parent": "..."}],
  "items": [{"barcodes": [...], "operation": "upsert", ...}]
}
```

Key behaviours:
- Deduplicates categories by name (prioritizes non-hidden ones)
- For delta imports, separates `REMOVE` and `UPSERT` operations
- Cleans redundant barcodes in reverse order (first occurrence takes precedence)

`to_category_hierarchy()` builds nested category structures with parent references, supporting optional `common_parent` mode where all children share the same parent.

### Data Cleaning Utilities

**Source:** `utils/data_cleaning.py`, `utils/barcode_utils.py`

| Function | Purpose |
|----------|---------|
| `remove_incomplete_rows(df, columns)` | Drop rows with nulls in critical columns |
| `remove_duplicate_rows(df, column, keep)` | Deduplicate by column |
| `to_decimal(value)` | Safe string → Decimal conversion |
| `upc_a_add_checksum(upc)` | Calculate UPC-A check digit (11-digit codes) |
| `ean_add_checksum(ean)` | Calculate EAN check digit (7 or 12-digit codes) |

---

## Package 3: `inventory_scripts` — Retailer Integrations

**Location:** `src/inventory_scripts/`

44+ retailer-specific import implementations, each following a consistent pattern:

```
inventory_scripts/<retailer>/
├── main.py        # Entry point function
├── parsers.py     # DataImport subclass(es)
├── client.py      # (optional) Custom API/SOAP client
└── constants.py   # (optional) Retailer-specific config
```

### Retailer Catalogue

| Retailer | Directory | Data Source | Protocol | Notable Features |
|----------|-----------|------------|----------|------------------|
| **LS Retail** | `ls_retail/` | LS Central POS | SOAP + NTLM auth | Paginated table reads, multi-table joins, CSV cache, retry with timeouts |
| **Shopify** | `standard_shopify/` | Shopify REST API | HTTP REST | Product + variant fetching, metafield support, image handling |
| **Odoo v17** | `odoo/v17/` | Odoo ERP | HTTP REST (JSON-RPC) | Cloudinary image upload, attribute-based variants |
| **Dynamics 365** | `standard_d365/` | Microsoft D365 | HTTP REST | Standard ERP integration |
| **Dubai Duty Free** | `dubai_duty_free/` | DDF systems | Custom | Airport retail specifics |
| **Eroski** | `eroski/` | Eroski POS | Custom | Spanish supermarket chain |
| **Flying Tiger** | `flying_tiger/` | Flying Tiger systems | Custom | Danish variety retailer |
| **Grandiose** | `grandiose/` | Restaurant chain | Custom | Food service items |
| **Grandiose FnB** | `grandiose_fnb/` | Food & Beverage | Custom | Addon-themed menu items |
| **Hudson** | `hudson_unify/` | Hudson Retail + FnB | Custom | Dual retail/food variants |
| **HMS Host** | `hmshost/` | Airport food service | Custom | Hospitality items |
| **SSP** | `ssp/` | Travel retail | Custom | Travel food service |
| **IKEA** | `ikea/` | IKEA systems | Custom | Furniture retail |
| **Kiko Milano** | `kiko_milano_azadea/` | Azadea Group | Custom | Cosmetics via Azadea |
| **King College Hospital** | `king_college_hospital/` | Hospital POS | SOAP + NTLM | Legacy on-premise system |
| **Loves Travel** | `loves_travel/` | Travel stops | Custom | US travel retail |
| **Muji** | `muji/` | Muji POS | Custom | Japanese retailer |
| **Paradies** | `paradies/` | Duty-free | Custom | Airport duty-free |
| **Virgin Megastore** | `virgin_megastore/` | Virgin systems | Custom | Entertainment retail |
| **WH Smith** | `wh_smith/` | WH Smith POS | Custom | UK newsagent/books |
| **Walmart (Odoo)** | `walmart_odoo/` | Odoo ERP | HTTP REST | Walmart via Odoo backend |
| **Urban Agri** | `urban_agri/` | Odoo ERP | HTTP REST | Agriculture retail |
| **Pantree (Odoo)** | `pantree_odoo/` | Odoo ERP | HTTP REST | Pantry/grocery |
| **Dimona** | `dimona/` | Beverage service | Custom | Beverage items |
| **Badiani** | `badiani/` | Ice cream chain | Custom | Food service |
| **MMI** | `mmi/` | MMI systems | Custom | Beverages retail |
| **Rotana** | `rotana/` | Rotana Hotels | Custom | Hotel retail |
| **Seventh Heaven** | `seventh_heaven/` | Seventh Heaven | Custom | Retail chain |
| **Flash Facial** | `flash_facial/` | Flash Facial | Custom | Beauty services |
| **GMG** | `gmg/` | GMG Group | Custom | Sports/lifestyle retail |
| **MLSE** | `mlse/` | Sports entertainment | Custom | Venue retail |
| **Event Network** | `eventnetwork/` | Museum/attraction shops | Custom | Event venue retail |
| **CGS** | `cgs/` | CGS | Custom | Retail |
| **Goloso** | `goloso/` | Food retail | Custom | Food service |
| **STO** | `sto/` | STO | Custom | Retail |
| **Emmasgarden** | `emmasgarden/` | Garden centre | Custom | Garden retail |
| **Funky Buddha** | `funky_buddha/` | Fashion retail | Custom | Greek fashion |
| **Flying Tiger Azadea** | `flying_tiger_azadea/` | Azadea Group | Custom | Flying Tiger Middle East |

**Generic/Multi-use scripts:**

| Directory | Purpose |
|-----------|---------|
| `mishipay_all_in_one/` | Universal import handler for retailers using the standard CSV format |
| `standard_cafe/` | Standard cafe/food service import |
| `standard_add_on_cc/` | Standard addon + click-collect import |
| `addons_upload/` | Standalone addon group import |
| `associate_addon_groups_with_items/` | Link existing addon groups to items |

### Common Entry Point Pattern

Every retailer follows the same function signature:

```python
def <retailer>_inventory_service_import(
    inventory_service_url: str,      # Inventory Service base URL
    store_id: str,                   # MishiPay store UUID
    retailer_id: str,                # MishiPay retailer UUID
    default_image_url: str,          # Fallback product image
    inventory_settings: dict,        # Retailer API credentials/config
    inventory_data_filepath: str = None,       # Optional: bypass API, use local file
    inventory_data_write_filepath: str = None,  # Optional: cache fetched data
    clean: bool = False,             # Full refresh vs delta
) -> None:
```

### Shared Utility

**Source:** `utils.py`

`get_existing_inventory_data(url, store_id, clean)` — Paginates through the entire current inventory for a store (500 items per page) to enable delta comparison. Returns empty list if `clean=True` (full refresh).

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `requests` | ^2.26.0 | HTTP client |
| `requests-ntlm` | ^1.1.0 | NTLM auth for legacy SOAP systems (LS Retail) |
| `pandas` | ^1.3.2 | Data manipulation (optional, `[import]` extra) |
| `numpy` | 1.26.3 | Numerical operations |
| `beautifulsoup4` | ^4.10.0 | HTML parsing |
| `xmltodict` | ^0.11.0 | XML → dict (SOAP responses) |
| `pyodbc` | ^4.0.39 | ODBC database connections (on-premise ERPs) |
| `openpyxl` | ^3.1.2 | Excel file parsing |
| `cloudinary` | ^1.34.0 | Image CDN upload/transform |
| `cchardet` | ^2.1.7 | Character encoding detection |

---

## How It Connects to the Platform

```
Monolith (Django)
    │
    ├── inventory/management/commands/     ← 35+ management commands
    │       │
    │       └── calls inventory_scripts.<retailer>.main.<retailer>_inventory_service_import()
    │                │
    │                ├── inventory_scripts/<retailer>/parsers.py   ← fetch + parse
    │                ├── inventory_import/                         ← normalize + map
    │                └── inventory_service/client.py               ← HTTP POST
    │                         │
    │                         ▼
    │                 Inventory Service (Node.js, MongoDB)
    │
    ├── inventory/views.py                 ← 7 proxy endpoints also use InventoryV1Client
    └── inventory/basket_svc.py            ← Basket operations use InventoryV1Client
```

The monolith's `inventory` module (documented in [Inventory](./inventory.md)) imports this library and calls the retailer-specific entry points from Django management commands. The same `InventoryV1Client` is also used directly by the monolith's proxy views and basket service for real-time item lookups.

---

## Quality & Testing

- **Code coverage requirement:** 90% (enforced via `pytest-cov`)
- **Linting:** black, flake8 (with bugbear, builtins, comprehensions, docstrings, printf-formatting), pep8-naming, bandit (security)
- **Type checking:** mypy with `types-requests`
- **Testing:** pytest with `requests-mock` for HTTP mocking, `pytest-mock` for general mocking, `Faker` for test data
- **Python version:** ^3.9

---

## Notable Patterns

1. **Lazy evaluation** — The `DataImport.data` property triggers the full ETL pipeline only on first access. Subsequent accesses return the cached DataFrame.

2. **Declarative schema** — Retailer parsers define their data contract via class attributes (`column_names`, `required_columns`, etc.) rather than imperative code. The base class handles extraction and cleaning.

3. **Delta imports** — The `OperationType` enum (`UPSERT`/`REMOVE`) enables incremental updates. Existing inventory is fetched via `get_existing_inventory_data()` and compared to detect changes, avoiding full re-imports.

4. **Protocol diversity** — The library handles SOAP/NTLM (LS Retail), REST/JSON (Shopify, Odoo), ODBC (direct database), CSV/TSV files, and Excel spreadsheets — all normalized into the same pandas DataFrame format before reaching the import pipeline.

5. **Local file fallback** — Every retailer script accepts an optional `inventory_data_filepath` parameter to bypass live API calls and use a cached/local file instead. This supports offline testing and debugging.
