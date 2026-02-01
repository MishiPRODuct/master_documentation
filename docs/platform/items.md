# Items Module (`mishipay_items`)

> Last updated by: Task T11 — Items Module Part 3 (Business Logic)

## Overview

The `mishipay_items` module is the **largest module** in the MishiPay platform (~649 files). It manages the entire product lifecycle: item inventory, shopping baskets, wishlists, promotions, product regulations, manual verification, and item ratings. Items are scoped to a **store** (via `ItemCategory`), and baskets are scoped to a **customer + store** pair.

This document covers the data models, serializers, admin configuration, API endpoints, and business logic (promotion evaluation, barcode scanning, store-type procedures, helpers, and test coverage).

## Key Concepts

- **Item** — A product in a store's inventory (`ItemInventory`), identified by a `product_identifier` (barcode/databar) and scoped to an `ItemCategory` which belongs to a `Store`.
- **Basket** — A shopping cart for a customer at a specific store. Contains `BasketItem` entries. Can expire, and tracks promotion application state.
- **Wishlist** — Similar to a basket but for saved-for-later items.
- **Promotion** — Platform-side promotion definitions that link retailer codes to `PromotionVariant` handlers. Distinct from the `service.promotions` microservice (see [Promotions Service](../promotions-service/overview.md)).
- **BasketEntityAuditLog** — The primary record of how promotions/discounts were applied to basket items. This model drives the basket display (prices, savings, quantities) in the checkout flow.
- **ManualVerificationSession** — Age/alcohol verification workflow for restricted items.
- **SimplePromotion** — Lightweight promotion type (simple discount, meal deal, X for Y) used by specific retailers.

## Architecture

### Module Structure

```
mishipay_items/
├── __init__.py              (21K — country choices, flags, discount constraints, constants)
├── models/
│   ├── __init__.py           (re-exports from models.py + models_v2.py)
│   ├── models.py             (68K — 20 legacy models + 3 custom managers)
│   └── models_v2.py          (1.4K — 1 new MPTT-based Category model)
├── serializers.py            (91K — 53 serializer classes)
├── serializers_v2.py         (26K — 13 V2 serializer classes)
├── admin.py                  (13K — 17 admin registrations with import/export)
├── entity_base_classes.py    (11K — BasketProcedure base class)
├── constants.py              (494B — basket item type choices)
├── views.py                  (78K)
├── api.py / api_v2.py        (26K / 48K)
├── urls.py                   (7.3K)
├── basket_and_wishlist.py    (126K — largest file, basket/wishlist operations)
├── app_settings.py           (50K — retailer-specific settings)
├── mpay_promo.py             (31K — promotion procedures)
├── item_scan_functions.py    (23K — barcode scanning logic)
├── promotions/               (promotion subsystem)
├── procedures/               (74+ store-type-specific procedures)
├── helpers/                  (9 helper modules)
├── tests/                    (76+ test files)
├── 30+ retailer utilities/   (retailer-specific item processing)
└── migrations/               (142 migration files)
```

### Model Relationship Diagram

```
Store (dos.models)
 ├── ItemCategory  ──► ItemInventory ──► AdditionalBarcode
 │                                   ──► ItemProperty (DEPRECATED)
 │                                   ──► ItemCatalog
 ├── Basket ──► BasketItem
 │          ──► BasketAddress
 │          ──► BasketEntityAuditLog
 │          ──► ManualVerificationSession
 ├── Wishlist ──► WishlistItem
 │            ──► WishlistExtraDetails
 ├── SimplePromotion
 ├── InventoryUpdateLog
 └── RateItem

Customer (dos.models)
 ├── Basket
 ├── Wishlist
 └── RateItem

PromotionVariant ──► Promotion

Category (V2, MPTT) ──► Retailer (mishipay_core)
```

## Data Models

### Legacy Models (`models/models.py`)

#### ItemCategory

Categorises items within a store. Each category is unique per store.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `category_name` | `CharField(50)` | db_index | Name of the category |
| `category_slug` | `SlugField(255)` | unique | Auto-generated from `{category_name}-{store.store_id}` |
| `image_url` | `URLField` | default: Cloudinary placeholder | Category image |
| `retailer_category_id` | `CharField(200)` | nullable | Retailer's own category ID |
| `store` | `FK → Store` | CASCADE | Store this category belongs to |
| `category_deal` | `TextField` | nullable | Deal text for the category |
| `requires_approval` | `BooleanField` | default: False | Triggers manual verification |
| `created` / `updated` | `DateTimeField` | auto | Timestamps |

**Constraints:** `unique_together = (category_name, store)`
**Custom save:** Auto-generates `category_slug` from `{category_name}-{store.store_id}`.

Source: `models/models.py:60–101`

---

#### AdditionalBarcode

Stores extra barcodes for items beyond the primary `product_identifier`. Some retailers assign multiple barcodes to a single product.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `item` | `FK → ItemInventory` | CASCADE, related: `additional_barcodes` | Parent item |
| `barcode` | `CharField(55)` | — | The additional barcode value |

**Constraints:** `unique_together = (barcode, item)`
**Inherits from:** `BaseModel` (mishipay_core)

Source: `models/models.py:104–113`

---

#### ItemInventory

The **core product model**. Represents a single item in a store's inventory. This is the central entity that basket items, audit logs, and promotions reference.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `item_id` | `UUIDField` | unique, non-editable | Public identifier |
| `item_name` | `CharField(255)` | — | Display name |
| `img` | `URLField` | default: Cloudinary placeholder | Product image URL |
| `epc` | `CharField(200)` | nullable | RFID Electronic Product Code |
| `product_identifier` | `CharField(200)` | db_index | Primary barcode/databar (scanned by app) |
| `original_product_identifier` | `CharField(200)` | — | Barcode as received from retailer (before modifications) |
| `retailer_product_id` | `CharField(200)` | db_index, nullable | Retailer's internal product ID |
| `style_number` | `CharField(200)` | nullable | Style/variant number |
| `item_category` | `FK → ItemCategory` | CASCADE | Category (also determines store) |
| `country` | `CharField(25)` | choices: `CountryChoice.CHOICES` | Country of origin |
| `description` | `TextField` | nullable | Product description |
| `stock_quantity` | `IntegerField` | default: 9999 | Available stock (9999 = unlimited) |
| `price` | `DecimalField(15,2)` | — | Duty-free price |
| `modified_base_price` | `DecimalField(15,2)` | nullable | Duty-paid price |
| `is_restricted` | `BooleanField` | default: False | Whether item is restricted |
| `allowed_restricted_item` | `BooleanField` | default: False | Allow restricted item (Dufry-specific) |
| `is_item_sold` | `BooleanField` | default: False | If True, stock-out error on basket ops |
| `epc_status` | `CharField(30)` | choices: `RFIDStatusChoice` | RFID tag status |
| `discount_percent` | `DecimalField(9,2)` | default: 0.0 | Base discount percentage |
| `promo_code` | `CharField(20)` | nullable | Legacy promotion code |
| `discount_constraint` | `JSONField` | default: dict | Discount rules as JSON |
| `extra_data` | `JSONField` | default: dict | Flexible key-value storage |
| `soft_delete` | `BooleanField` | default: False | Soft delete flag |
| `requires_approval` | `BooleanField` | default: False | Triggers manual verification |
| `promotional_offer_codes` | `ArrayField(CharField(32))` | default: list | All promo codes for this item |
| `source_hash` | `CharField(255)` | default: "" | Hash of last import line (change detection) |
| `max_basket_qty` | `PositiveSmallIntegerField` | default: 0 | Max quantity allowed in basket (0 = unlimited) |
| `ref_promo_id` | `CharField(128)` | nullable | Internal promo ID (used by Eroski) |
| `deposit_item_retailer_id` | `CharField(200)` | nullable | Deposit item link |
| `tax_code` | `CharField(64)` | nullable | Tax classification code |
| `created` / `updated` | `DateTimeField` | auto, `updated` db_index | Timestamps |

**Indexes:**
1. `(soft_delete, item_category, product_identifier)` — item scan by barcode
2. `(item_name, item_category)` — shopping list search
3. `(retailer_product_id, item_category)` — bulk update/create

**Key properties:**
- `item_specific_promotions` — queries `Promotion` objects matching this item's `promotional_offer_codes`
- `item_promotions` — active promotions for this item (date-filtered)
- `get_applied_offer(quantity)` — evaluates all applicable promotions and returns the lowest-price offer

Source: `models/models.py:116–251`

---

#### ItemProperty (DEPRECATED)

Dynamic key-value properties for items per store. Only used by Paradies retailer; slated for removal after migration to inventory service.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `item` | `FK → ItemInventory` | CASCADE | Parent item |
| `store` | `FK → Store` | CASCADE | Store context |
| `name` | `TextField` | indexed | Property name |
| `value` | `TextField` | indexed | Property value |

**Constraints:** `unique_together = (item, store, name)`
**Inherits from:** `BaseModel`

Source: `models/models.py:253–278`

---

#### ProductRegulation

Rules and restrictions for products — controls quantity limits, age verification requirements, and other regulatory constraints.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `identifier` | `CharField(200)` | — | Barcode/product code/category to match |
| `identifier_type` | `CharField(50)` | choices: `IDENTIFIER_TYPE` | Type: category, barcode, group, etc. |
| `regulation_type` | `CharField(100)` | choices: `REGULATION_TYPE` | Quantity-based or verification-based |
| `allowed_quantity` | `IntegerField` | default: 1 | Max allowed quantity |
| `priority` | `IntegerField` | default: 10 | Rule priority (lower = higher priority) |
| `applicable_level` | `CharField(100)` | choices: `REGULATION_APPLICABLE_TYPE` | item, basket, or basket_and_item |
| `store_type` | `CharField(200)` | — | Store type this rule applies to |
| `condition_to_evaluate` | `CharField(1024)` | nullable | Additional evaluation condition |
| `is_valid` | `BooleanField` | default: True | Whether rule is active |
| `valid_from_date` / `valid_till_date` | `DateTimeField` | auto_add / nullable | Validity period |
| `created` / `updated` | `DateTimeField` | auto | Timestamps |

Source: `models/models.py:281–310`

---

#### ItemCatalog

Maps a single catalog identifier to multiple items. Used for grouping items under a producer/catalog code.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `catalog_identifier` | `CharField(40)` | — | Catalog/producer identifier |
| `item` | `FK → ItemInventory` | CASCADE | Linked item |
| `soft_delete` | `BooleanField` | default: False | Soft delete flag |
| `created` / `updated` | `DateTimeField` | auto | Timestamps |

Source: `models/models.py:312–331`

---

#### Basket

The shopping cart. Created per customer + store pair. Contains rich promotion/discount state.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `basket_id` | `UUIDField` | unique, non-editable | Public identifier |
| `customer` | `FK → Customer` | CASCADE | Owner |
| `store` | `FK → Store` | CASCADE | Store context |
| `expiry` | `DateTimeField` | nullable | When the basket expired |
| `is_expired` | `BooleanField` | default: False | Expiration flag |
| `extra_data` | `JSONField` | default: dict | Flexible data (loyalty info, other charges, etc.) |
| `applicable_offers` | `JSONField` | default: dict | Available promotions |
| `applied_offer` | `JSONField` | default: dict | Currently applied promotion |
| `price_after_discount` | `DecimalField(15,2)` | default: 0.0 | Price after promotions |
| `total_vat_price` | `DecimalField(15,2)` | default: 0.0 | Total VAT |
| `promotion_applied` | `BooleanField` | default: False | Whether promotions have been evaluated |
| `is_basket_stale` | `BooleanField` | default: False | Needs re-evaluation flag |
| `last_promotion_applied_date` | `DateTimeField` | nullable | Last promo evaluation time |
| `ms_promo_response` | `JSONField` | default: dict | Raw response from promotions microservice |
| `special_promos` | `JSONField` | nullable | Special promotions data |
| `total_basket_amount_after_discount` | `DecimalField(15,2)` | nullable | Cached total after discount |
| `total_taxable_amount` | `DecimalField(15,2)` | nullable | Cached taxable amount |
| `total_tax` | `DecimalField(15,2)` | nullable | Cached total tax |
| `total_basket_amount_after_tax` | `DecimalField(15,2)` | nullable | Cached total after tax |
| `tax_level_breakup` | `JSONField` | nullable | Per-tax-level breakdown |
| `tax_calculator_type` | `CharField(15)` | choices: `TAX_CALCULATOR_TYPE_CHOICES` | Tax calculation method |
| `total_voucher_discount` | `DecimalField(15,2)` | nullable | Voucher discount amount |
| `voucher_code` | `JSONField` | nullable | Applied voucher code(s) |
| `purchase_type` | `TextField` | choices: `PURCHASE_TYPES`, default: SCAN_AND_GO | Purchase channel |
| `is_saved_card_basket` | `BooleanField` | nullable | Dummy basket for card saving |
| `platform` | `CharField(25)` | choices: `PLATFORM_CHOICES` | Platform origin (iOS/Android/web) |
| `created` / `updated` | `DateTimeField` | auto | Timestamps |

**Custom Manager:** `BasketManager` with methods:
- `get_active_baskets()` — non-expired baskets
- `get_customer_active_baskets(customer_id)` — active baskets for a customer
- `get_other_store_baskets(store_id)` — baskets from other stores

**Key methods:**
- `requires_approval` — complex property checking for age-restricted items, previous verification history, and retailer-specific overrides (Eroski Leioa/Deusto hack, FlyingTiger cache-based sampling)
- `get_total_amount(store)` — calculates basket total, with special handling for tax calculator types and retailer-specific logic (Connexus/Fast Eddys)
- `get_discounted_amount(store)` — applies category-based discount constraints (wine quantity tiers for DirectWines/MishiPay)
- `calculate_total_vat(store, items)` — VAT calculation with retailer-specific calculators
- `get_vat_table()` — VAT breakup grouped by VAT identifier
- `get_total_modified_monetary_value()` — total after all promotions (from audit logs or discount calculation)

**Index:** `(customer, purchase_type, is_expired, updated)` — for customer detail queries

Source: `models/models.py:334–833`

---

#### BasketItem

Individual items within a basket. Largely mirrors `ItemInventory` fields (denormalized at basket-add time).

| Field | Type | Constraints | Description |
|---|---|---|---|
| `item_id` | `UUIDField` | unique, non-editable | Public identifier |
| `basket` | `FK → Basket` | CASCADE | Parent basket |
| `item_name` | `CharField(255)` | — | Snapshot of item name |
| `img` | `URLField(1024)` | — | Item image |
| `epc` | `CharField(200)` | nullable | RFID EPC |
| `product_identifier` | `CharField(200)` | db_index | Barcode |
| `retailer_product_id` | `CharField(200)` | nullable | Retailer product ID |
| `item_category` | `CharField(50)` | — | Category name (denormalized string, not FK) |
| `country` | `TextField` | choices | Country of origin |
| `description` | `TextField` | nullable | Product description |
| `item_quantity` | `IntegerField` | default: 1, db_index | Quantity in basket |
| `price` | `DecimalField(15,2)` | — | Unit price at time of addition |
| `modified_base_price` | `DecimalField(15,2)` | nullable | Reduced/duty-paid price |
| `is_restricted` | `BooleanField` | default: False | Restricted item flag |
| `allowed_restricted_item` | `BooleanField` | default: False | Allow restricted (Dufry) |
| `discount_percent` | `DecimalField(9,2)` | default: 0.0 | Item-level discount |
| `promo_code` | `CharField(20)` | nullable | Promotion code |
| `is_refunded` | `BooleanField` | default: False | Refund flag |
| `refund_processing_quantity` | `IntegerField` | default: 0 | Qty being refunded |
| `refunded_quantity` | `IntegerField` | default: 0 | Qty already refunded |
| `discount_constraint` | `JSONField` | default: dict | Discount rules |
| `stock_quantity` | `IntegerField` | nullable | Stock at add time (TODO: pointless per code comment) |
| `extra_data` | `JSONField` | default: dict | Flexible data |
| `vat_amount` | `DecimalField(15,2)` | nullable | VAT amount |
| `vat_identifier` | `CharField(100)` | nullable | VAT type identifier |
| `requires_approval` | `BooleanField` | default: False | Manual verification |
| `applicable_offers` | `JSONField` | default: dict | Available offers |
| `applied_offer` | `JSONField` | default: dict | Applied offer (TODO: marked for deletion) |
| `deposit_item_retailer_id` | `CharField(200)` | nullable | DEPRECATED (FTC migration) |
| `tax_code` | `CharField(64)` | nullable | Tax code |
| `is_return_item` | `BooleanField` | default: False | Return item flag |
| `is_recommended` | `BooleanField` | default: False | Recommendation flag |
| `item_type` | `CharField(15)` | choices: REGULAR/WITH_ADDONS/ADDON/COMBO | Item classification |
| `platform` | `CharField(25)` | nullable | Platform origin |
| `created` / `updated` | `DateTimeField` | auto, both db_indexed | Timestamps |

**Key methods:**
- `get_price_before_discount()` — original price from audit logs or `price * quantity`
- `get_price_after_discount()` — applies discount_percent, with retailer-specific overrides (Compass/Dufry/Relay use audit log values)
- `vat_rate`, `vat_type`, `net_sales_amount`, `total_vat` — VAT computation properties

Source: `models/models.py:835–998`

---

#### BasketAddress

Delivery or billing address linked to a basket.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `address_id` | `UUIDField` | non-editable | Identifier |
| `basket` | `FK → Basket` | CASCADE | Parent basket |
| `postcode` | `CharField(256)` | — | Postal code |
| `country_code` | `CharField(2)` | — | ISO country code |
| `country_area` | `CharField(256)` | — | Region/state |
| `city` | `CharField(256)` | — | City |
| `name_of_user_to_refer` | `CharField(256)` | — | Recipient name |
| `email_of_user_to_refer` | `EmailField(256)` | — | Recipient email |
| `delivery_instruction` | `TextField` | nullable | Special instructions |
| `address_1` / `address_2` | `CharField(256)` | — | Address lines |
| `phone_number` | `PhoneNumberField` | nullable | Phone |
| `fax_number` | `PhoneNumberField` | nullable | Fax |
| `type_of_address` | `CharField(10)` | choices: `ADDRESS_CHOICES` | delivery or billing |
| `created` / `updated` | `DateTimeField` | auto | Timestamps |

**Constraints:** `unique_together = (basket, type_of_address)`

Source: `models/models.py:1000–1032`

---

#### Wishlist / WishlistItem / WishlistExtraDetails

Mirror the Basket/BasketItem pattern but for wishlists. Wishlist is per customer + store.

- **Wishlist**: `wishlist_id` (UUID), `customer` (FK), `store` (FK), `is_expired`, `expiry`
- **WishlistItem**: Nearly identical fields to `BasketItem` (item_id, item_name, product_identifier, price, quantity, etc.)
- **WishlistExtraDetails**: Links wishlist to customer/store with `send_email` / `send_email_status` flags

Source: `models/models.py:1035–1117`

---

#### RateItem

Customer ratings and feedback for products.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `customer` | `FK → Customer` | CASCADE | Reviewer |
| `product_identifier` | `CharField(200)` | db_index | Product barcode |
| `store` | `FK → Store` | CASCADE | Store context |
| `feedback` | `TextField` | nullable | Text feedback |
| `rating` | `IntegerField` | — | Rating value (1-5) |
| `tags` | `TextField` | nullable | Feedback tags |
| `created` / `updated` | `DateTimeField` | auto | Timestamps |

Source: `models/models.py:1120–1136`

---

#### RemoteItemMap

Maps merchant-specific remote item IDs to scanned item IDs. Used for external system integrations.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `merchant_id` | `CharField(64)` | — | Merchant identifier |
| `remote_item_id` | `CharField(63)` | — | ID in remote system |
| `scanned_item_id` | `CharField(63)` | — | ID from scan |
| `extra_data` | `JSONField` | default: dict | Additional data |

**Constraints:** Two `unique_together`: `(merchant_id, remote_item_id)` and `(merchant_id, scanned_item_id)`

Source: `models/models.py:1139–1151`

---

#### PromotionVariant

Defines a promotion execution handler. Each variant maps to a Python class that implements the discount calculation.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `variant_id` | `UUIDField` | unique | Identifier |
| `variant_code` | `CharField(32)` | unique | Short code |
| `variant_name` | `CharField(128)` | — | Display name |
| `handler` | `CharField(1024)` | — | Fully-qualified Python class path (e.g., `mishipay_items.promotions.handlers.XForYHandler`) |
| `discount_type` | `CharField(32)` | nullable | Discount classification |
| `extra_data` | `JSONField` | default: dict | Handler configuration |

**Key pattern:** The `handler` field is dynamically imported at runtime via `__import__()` in `Promotion.apply_offer()`.

Source: `models/models.py:1154–1173`

---

#### Promotion

Platform-side promotion definition. Links retailer promotion codes to `PromotionVariant` handlers. Distinct from the `service.promotions` microservice — this model handles promotions evaluated locally within the monolith.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `promotion_id` | `UUIDField` | unique | Public identifier |
| `name` | `CharField(1024)` | — | Promotion name |
| `description` | `CharField(2048)` | — | Full description |
| `display_name` | `CharField(128)` | — | UI display label |
| `retailer_code` | `CharField(32)` | — | Retailer's promotion code |
| `mishipay_code` | `FK → PromotionVariant` | CASCADE | Handler variant |
| `store_type` | `CharField(64)` | — | Applicable store type |
| `configuration_type` | `CharField(128)` | choices: `COMMON_OFFERS_CONFIGURATION_CHOICES` | How to match (store_type, loyalty, etc.) |
| `configuration_value` | `CharField(128)` | — | Value for the configuration type |
| `meta_data` | `JSONField` | default: dict | Handler parameters (value, unit, max_instances, etc.) |
| `priority` | `IntegerField` | default: 1 | Evaluation priority |
| `applicability_checks` | `MultiSelectField` | choices: `PromotionApplicabilityConfig` | Which checks to run |
| `dynamic_config` | `CharField(32)` | choices: `DynamicConfigurationSettings` | Dynamic configuration key |
| `is_blocked` | `BooleanField` | default: False | Whether promotion is disabled |
| `image` | `URLField` | default: "" | Promotion image |
| `applicable_item_class` | `CharField(32)` | default from `APPLICABLE_PROMOTION_CLASS_CHOICES` | item, basket, etc. |
| `display_data` | `JSONField` | default: `{sticker_text: "Offer", label_text: ""}` | UI display configuration |
| `start_date` / `end_date` | `DateTimeField` | nullable | Validity period |

**Custom Manager:** `PromotionManager` with methods:
- `get_store_type_promotions(store_type)` — promotions for a store type
- `get_item_specific_promotions(item_id)` — promotions matching item's `promotional_offer_codes` and class=item
- `get_item_promotions(item_id)` — active promotions (date-filtered)
- `get_basket_level_promotions()` — promotions with class=basket
- `get_one_time_redeem_per_user_promotions()` — one-time-use promotions

**Key method:** `apply_offer(items_detail, **kwargs)` — dynamically imports and instantiates the handler class from `mishipay_code.handler`, passes item details and meta_data, and returns the discount result.

Source: `models/models.py:1240–1303`

---

#### BasketEntityAuditLog

**Critical model** — records how each promotion/discount was applied to basket items. This is the primary data source for basket display in checkout, showing per-item pricing, savings, and promotion details.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `uuid` | `UUIDField` | primary_key | Identifier |
| `basket` | `FK → Basket` | CASCADE | Parent basket |
| `applicable_entity_type` | `CharField(64)` | db_index | Type: item, group, nopromo, basket |
| `applicable_entity_value` | `CharField(64)` | — | Entity identifier value |
| `retailer_code` | `CharField(64)` | — | Retailer's promotion code |
| `mishipay_code` | `CharField(64)` | — | MishiPay's promotion code |
| `original_monetary_value` | `DecimalField(15,2)` | default: 0.0 | Price before discount |
| `modified_monetary_value` | `DecimalField(15,4)` | default: 0.0 | Price after discount (4 decimal places) |
| `monetary_value_savings` | `DecimalField(15,2)` | default: 0.0 | Savings amount |
| `application_entity_count` | `IntegerField` | default: 1 | Quantity of items |
| `entity_identifier` | `UUIDField` | nullable, db_index | References basket item's item_id |
| `is_return_item` | `BooleanField` | default: False | Return item flag |
| `extra_data` | `JSONField` | default: dict | Rich nested data including `item_info` |
| `applied_promo_retailer_ids` | `ArrayField(CharField(255))` | default: list | All applied promo IDs |
| `tax_code` | `CharField(64)` | nullable | Tax classification |
| `taxable_amount` | `DecimalField(15,4)` | default: 0.0 | Taxable amount |
| `total_tax_amount` | `DecimalField(15,4)` | default: 0.0 | Total tax |
| `tax_level_breakup` | `JSONField` | nullable | Per-level tax breakdown |
| `item_type` | `CharField(15)` | choices: BASKET_ITEM_TYPE_CHOICES | Item classification |
| `created` / `updated` | `DateTimeField` | auto, `created` db_indexed | Timestamps |

**Custom Manager:** `BasketEntityAuditLogManager` with:
- `get_basket_logs(basket_ref_id)` — all logs for a basket
- `get_basket_item_logs/group_logs/nopromo_logs` — filtered by entity type
- `get_total_monetary_savings_on_basket(basket)` — dispatches to retailer-specific savings function via `RETAILER_SPECIFIC_MONETARY_FUNCTION_MAP`

**Notable `__init__` override:** The constructor recalculates `monetary_value_savings` on load if there's a mismatch with `original - modified - price_override_difference`. This handles cases where the same SKU has multiple barcodes creating separate audit log rows. If `is_return_item`, savings are forced to 0.

Source: `models/models.py:1306–1465`

---

#### ManualVerificationSession

Tracks age/alcohol verification sessions for baskets with restricted items.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `uuid` | `UUIDField` | primary_key | Identifier |
| `basket` | `FK → Basket` | CASCADE | Basket requiring verification |
| `code` | `PositiveSmallIntegerField` | — | Verification code |
| `status` | `CharField(1)` | choices: `ManualVerificationChoices`, db_index | P/A/R (Pending/Approved/Rejected) |
| `reviewer` | `FK → auth.User` | nullable, CASCADE | Staff who reviewed |
| `reason` | `CharField(1)` | choices: `VerificationReasons`, db_index | Alcohol or Age |
| `created` / `updated` | `DateTimeField` | auto, `created` db_indexed | Timestamps |

Source: `models/models.py:1468–1482`

---

#### SimplePromotion

Lightweight promotion type for specific retailers. Simpler than the full `Promotion` model — no handler dispatch, just a type/price.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `uuid` | `UUIDField` | primary_key | Identifier |
| `store` | `FK → Store` | CASCADE | Store |
| `code` | `CharField(255)` | db_index | Promo code |
| `barcode` | `CharField(255)` | db_index | Associated barcode |
| `display` | `CharField(255)` | — | Display text |
| `min_items` | `PositiveSmallIntegerField` | default: 1 | Minimum item count |
| `price` | `DecimalField(15,2)` | — | Promo price |
| `is_active` | `BooleanField` | default: True, db_index | Active flag |
| `promo_type` | `PositiveSmallIntegerField` | choices, db_index | 0=Simple Discount, 1=Meal Deal, 2=X for Y |
| `is_loyalty_promotion` | `BooleanField` | default: False | Loyalty-only flag |
| `start_date` / `end_date` | `DateTimeField` | — | Validity window |
| `source_hash` | `CharField(255)` | db_index | Import change detection hash |

Source: `models/models.py:1485–1519`

---

#### InventoryUpdateLog

Tracks inventory update requests and their processing status.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `store` | `FK → Store` | CASCADE | Target store |
| `update_request` | `JSONField` | default: dict | Request payload |
| `status` | `CharField(10)` | choices: `INVENTORY_UPDATE_STATUS_CHOICES`, db_index | Pending/Success/Error |
| `error_message` | `TextField` | nullable | Error details |

Source: `models/models.py:1522–1533`

---

### V2 Models (`models/models_v2.py`)

#### Category (V2)

New hierarchical category model using `django-mptt` (Modified Preorder Tree Traversal). Scoped to a `Retailer` rather than a `Store`, enabling categories to be shared across stores.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `parent` | `TreeForeignKey → self` | nullable, CASCADE | Parent category (null for root) |
| `retailer` | `FK → Retailer` | CASCADE, related: `categories` | Owning retailer |
| `name` | `TextField` | — | Category name |
| `retailer_category_id` | `TextField` | nullable | Retailer's category ID |

**Inherits from:** `BaseAuditModel` (mishipay_core) + `MPTTModel` (django-mptt)
**MPTTMeta:** `order_insertion_by = "name"` (alphabetical ordering in tree)

This is the **replacement** for `ItemCategory`. The legacy model ties categories to individual stores; the V2 model ties them to retailers, supporting multi-store category hierarchies.

Source: `models/models_v2.py:1–45`

---

## Serializers

The module has **53+ serializer classes** across `serializers.py` (91K) and `serializers_v2.py` (26K). They fall into three categories: input validation, model output, and composite response serializers.

### Validation Functions

Six standalone validation functions used across serializers:

| Function | Purpose |
|---|---|
| `validate_product_identifier` | Non-empty check |
| `validate_store_id` | Store exists check |
| `validate_retailer_name` | Retailer exists check |
| `validate_wishlist_id` | Wishlist exists check |
| `validate_customer_id` | Customer exists check |
| `validate_quantity` | Non-zero check |
| `validate_reference_id` | Valid UUID + BasketEntityAuditLog exists |
| `_validate_basket_id` | Basket exists + expiry check + payment status check |

Source: `serializers.py:53–128`

### Input/Request Serializers

| Serializer | Purpose | Key Fields |
|---|---|---|
| `ItemScanSerializer` | Item barcode scan | `product_identifier`, `store_id` |
| `ItemScanSerializerv2` | V2 scan (no max_length on barcode) | `product_identifier`, `store_id` |
| `ItemRecommendationSerializer` | Item recommendations | `product_identifier`, `store_id`, `basket_id` |
| `AddItemToBasketSerializer` | Add item to basket | `product_identifier`, `store_id`, `customer`, `basket_id`, `item_quantity` |
| `RemoveItemFromBasketSerializer` | Remove item from basket | `basket_id`, `item_id`, `customer`, `store_id`, `reference_id` |
| `BasketAndItemIdCheckSerializer` | Update item quantity | `basket_id`, `item_id`, `quantity`, `store_id` |
| `BulkAddItemsToBasketSerializer` | Bulk add items | `store_id`, `basket_id`, `items` (JSON array) |
| `BulkUpdateHardcodedBasketItemsSerializer` | Bulk update HARDCODED items | `store_id`, `basket_id`, `items` — validates only HARDCODED category items |
| `GetBasketDetailsSerializer` | Get basket info | `basket_id`, `store_id` |
| `AddItemToWishlistSerializer` | Add to wishlist | `product_identifier`, `store_id`, `customer`, `wishlist_id`, `item_quantity` |
| `RemoveItemFromWishlistSerializer` | Remove from wishlist | `wishlist_id`, `item_id`, `customer`, `store_id` |
| `RateItemSerializer` | Submit rating (1-5) | `customer`, `store`, `rating`, `product_identifier`, `tags`, `feedback` |

**Notable validation logic:**
- `AddItemToBasketSerializer.validate_product_identifier()` — max length is 200 normally but **750 for Carters** (`serializers.py:515-518`)
- `BulkUpdateHardcodedBasketItemsSerializer.validate()` — enforces that only items in categories containing "HARDCODED" can be bulk-updated, and checks stock levels

### V2 Request Serializers (serializers_v2.py)

| Serializer | Purpose | Key Fields |
|---|---|---|
| `RedeemPointsOnBasket` | Loyalty points redemption | `store_id`, `basket_id`, `membership_id`, `redeem_points` |
| `CancelRedeemedPointsOnBasket` | Cancel redemption | `store_id`, `basket_id`, `membership_id` |
| `SaveRewardsOnBasket` | Save reward points | `basket_id`, `reward_points` — Carters: multiple of 5, max $30 |
| `GetAddOnItemsSerializer` | Fetch add-on items | `addon_group_slug`, `store_id`, `filter` |
| `ListLoyaltyCouponsSerializer` | List loyalty coupons | `store_id`, `basket_id`, `membership_id` |
| `RedeemLoyaltyCouponSerializer` | Redeem coupon | `store_id`, `basket_id`, `membership_id`, `coupon_code`, `otp`, `mobile` |
| `BasketRedeemUnredeemLoyaltySerializer` | Redeem/unredeem coupon | `store_id`, `basket_id`, `coupon_code`, `action` (redeem/unredeem) |
| `BasketRedeemUnredeemPointsSerializer` | Redeem/unredeem points | `store_id`, `basket_id`, `points`, `action` |

### Output/Model Serializers

| Serializer | Model | Purpose |
|---|---|---|
| `ItemCategorySerializer` | `ItemCategory` | Category name only |
| `CreateItemInventorySerializer` | `ItemInventory` | Item creation (write) — validates `extra_data` against JSON schema |
| `ItemInventorySerializer` | `ItemInventory` | Item detail (read) — adds computed fields: `country_flag`, `discount_price`, `store_id`, `allowed_quantity` |
| `BasketSerializer` | `Basket` | Basket summary: `basket_id`, `store`, `requires_approval` |
| `BasketItemSerializer` | `BasketItem` | Basket item detail — 20+ fields with computed totals, ratings, wishlists |
| `BasketItemVerboseSerializer` | `BasketItem` | Extended version with `is_return_item` and `item_category` |
| `BasketItemForPromotionSerializer` | `BasketItem` | Minimal item data for promotion engine input |
| `WishlistSerializer` | `Wishlist` | Wishlist summary |
| `WishlistItemSerializer` | `WishlistItem` | Wishlist item detail with computed fields |
| `StoreBasicSerializer` | `Store` | Store summary with backwards-compat `retailer` → `name` mapping |
| `CustomerBasicSerializer` | `Customer` | Customer UUID only |

### Promotion Serializers

| Serializer | Purpose |
|---|---|
| `PromotionListSerializer` | List view — `name`, `promotion_id`, `display_name`, `retailer_code` |
| `PromotionSerializer` | Detail view — adds `description`, `image` |
| `ItemPromotionSerializer` | Item data for promotion context |
| `PromotionGroupListSerializer` | Groups promotions into `offers` and `deals` (Meal Deals) |
| `BasketItemPromotionSerializer` | Full promotion detail with `display_data`, `sticker_text`, `label_text` |
| `BasketPromotionSerializer` | Identical to `BasketItemPromotionSerializer` (duplicate) |

### Composite Response Serializers

#### BasketEntityAuditLogSerializer (`serializers.py:957`)

The most complex serializer — transforms audit log records into rich basket item display data. Key computed fields:
- `applied_offer` — price before/after, savings, quantity, sticker/label text, combo discount info
- `applicable_offers` — available promotions from `extra_data.item_info`
- `vat_free_price_of_item`, `item_vat` — VAT calculations
- `quantity_limit_reached` — checks buying guidance and category quantity limits
- `serial_number` — from extra_data

The `to_representation()` override (line 1154) performs extensive enrichment:
- Merges `item_info` from extra_data into the response
- Fetches live stock quantities (from context cache or DB query)
- Calculates addon pricing fields
- Handles weighted items (price per Kg)
- Applies `SHOW_ITEM_CODE_IN_BASKET_SCREEN_REGION_STORE_TYPES` config

#### BasketResponseSerializer (V2) (`serializers_v2.py:14`)

Comprehensive basket response including 24+ fields:
- `basket_item` — uses `BasketItemVerboseSerializer`
- `total_amount`, `basket_total_amount` — basket totals
- `percent_off` — discount description
- `total_vat_on_basket`, `basket_amount_without_vat` — VAT breakdowns
- `restriction_regulation`, `requires_approval`, `approval_status` — verification state
- `prompt_for_tax_info`, `by_pass_payment` — payment flow flags
- `tax_breakup`, `show_vat`, `sub_total`, `show_sub_total` — tax display
- `purchase_type`, `other_charges`, `other_charges_detail`

#### BasketEntityAuditLogSerializerV2 (`serializers_v2.py:209`)

Streamlined version of `BasketEntityAuditLogSerializer` — removes `wishlist_item_id`, `serial_number`, `quantity_limit_reached`, `mishipay_code` from output. Same `to_representation()` enrichment pattern.

---

## Admin Configuration

17 models are registered in Django admin (`admin.py`), with `django-import-export` support for bulk data management.

| Admin Class | Model | Features |
|---|---|---|
| `ItemCategoryAdmin` | `ItemCategory` | Import/export, search by store/name, raw_id for store |
| `ItemCatalogAdmin` | `ItemCatalog` | Import/export, search by catalog/item/store, soft_delete filter |
| `ItemInventoryAdmin` | `ItemInventory` | `CachedTablePaginator`, search by `=product_identifier` (exact), raw_id for category |
| `ProductRegulationAdmin` | `ProductRegulation` | Basic registration |
| `BasketItemAdmin` | `BasketItem` | **60-day queryset filter** — only shows recent items |
| `BasketAdmin` | `Basket` | Search by basket_id/customer/store, read-only offers fields |
| `RateItemAdmin` | `RateItem` | Search by product/customer/store |
| `WishlistExtraDetailsAdmin` | `WishlistExtraDetails` | Date hierarchy on `updated` |
| `WishlistItemAdmin` | `WishlistItem` | Search by item/wishlist |
| `WishlistAdmin` | `Wishlist` | Search by wishlist_id/customer/store |
| `RemoteItemMapAdmin` | `RemoteItemMap` | Filter by merchant_id |
| `PromotionVariantAdmin` | `PromotionVariant` | Search by variant fields |
| `PromotionAdmin` | `Promotion` | Search by name/code/store, filter by priority/store_type |
| `BasketEntityAuditLogAdmin` | `BasketEntityAuditLog` | **60-day queryset filter**, search by basket_id, raw_id for basket |
| `ManualVerificationSessionAdmin` | `ManualVerificationSession` | Search by `=basket__basket_id` (exact) |
| `SimplePromotionAdmin` | `SimplePromotion` | Filter by promo_type, search by barcode/code |
| `CategoryAdmin` | `Category` (V2) | Search by `=retailer__id`, filter by MPTT `level` |
| `InventoryUpdateLogAdmin` | `InventoryUpdateLog` | Basic registration |

**Import/Export resources:** `ItemCategoryResource`, `ItemCatalogResource`, `ItemInventoryResource` with custom `CustomDecimalWidget` for price handling.

**Notable patterns:**
- `BasketItemAdmin` and `BasketEntityAuditLogAdmin` both filter querysets to the last 60 days, preventing admin page timeouts on large tables
- `ItemInventoryAdmin` uses `CachedTablePaginator` (from mishipay_core) for performance and exact-match search on `product_identifier`
- Multiple admin classes use `raw_id_fields` for FK lookups to avoid loading large select dropdowns

---

## Entity Base Classes

The `entity_base_classes.py` file defines `BasketProcedure` — a base class (using `attrs`) for basket operations. It provides:

- `get_product_data(product_identifier)` — fetches item, applies regulation rules, returns serialized data
- `create_basket()` — creates empty basket for customer+store
- `create_basket_item_and_add_to_basket()` — creates BasketItem from product data, with regulation checks
- `add_new_item_to_basket()` / `update_item_in_basket()` — orchestrates add/update workflows
- `add_or_update_basket()` — main entry point: routes to add or update based on `item_exists` flag
- `get_basket_details()` — returns serialized basket via `BasketVerboseSerializer`

This class is extended by the 74+ store-type-specific procedures in `procedures/`.

Source: `entity_base_classes.py:1–269`

---

## Constants & Configuration

### Country System (`__init__.py`)

The module defines **~240 country constants** and a `CountryChoice.CHOICES` list used by `ItemInventory.country` and `BasketItem.country`. A `FLAG_LIST` dictionary maps ~25 countries to static flag image URLs on `static-assets.mishipay.com`.

### Discount Constraint Map (`__init__.py:510-583`)

A hardcoded `DISCOUNT_CONSTRAINT_MAP` defines quantity-tier discounts by store type and category:
- `mishipaystoretype` and `directwinesstoretype`: Wine discounts at 12+ (10%), 24+ (15%), 34+ (20%)
- Most other store types: empty wine constraints (no quantity discounts)

### Basket Item Types (`constants.py`)

```python
BASKET_ITEM_TYPE_CHOICES = (
    (REGULAR, REGULAR),
    (WITH_ADDONS, WITH_ADDONS),
    (ADDON, ADDON),
    (COMBO, COMBO)  # For Future, not used now
)
```

---

## API Architecture

The items module exposes APIs through **three tiers**, each with different access patterns and dispatch mechanisms:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Client (Mobile App / Web)                                           │
│         │                                                            │
│         ▼                                                            │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  V1 Public APIs (views.py)                                   │    │
│  │  Path: /items/v1/...                                         │    │
│  │  Auth: IsAuthenticated                                       │    │
│  │  Dispatch: → ItemManagementClient (microservice HTTP)        │    │
│  └──────────────────────────────────────────────────────────────┘    │
│         │                                                            │
│         ▼                                                            │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Internal Service APIs (api.py)                              │    │
│  │  Path: /items/api/v1/...                                     │    │
│  │  Auth: None (internal, called by ItemManagementClient)       │    │
│  │  Dispatch: → Store-type function map → BasketProcedure       │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  V2 APIs (api_v2.py)                                         │    │
│  │  Path: /items/v2/...                                         │    │
│  │  Auth: IsAuthenticated + ActiveStore + ActiveStaffShift      │    │
│  │  Dispatch: → Store-type function map → BasketProcedure       │    │
│  │  (Kiosk/staff-facing, bypasses ItemManagementClient)         │    │
│  └──────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

### Store-Type Dispatch Pattern

All three tiers ultimately invoke the same business logic through **store-type function maps** defined in `app_settings.py`. Each operation (scan, add to basket, remove, etc.) has a corresponding map:

```python
# app_settings.py (50K) — key maps:
ITEM_SCAN_FUNCTION_MAP = { "mishipaystoretype": ..., "dosstoretype": ..., ... }
ADD_ITEM_TO_BASKET_FUNCTION_MAP = { ... }
GET_BASKET_DETAILS_FUNCTION_MAP = { ... }
REMOVE_ITEM_FROM_BASKET_FUNCTION_MAP = { ... }
ADD_ITEM_TO_WISHLIST_FUNCTION_MAP = { ... }
GET_WISHLIST_DETAILS_FUNCTION_MAP = { ... }
REMOVE_ITEM_FROM_WISHLIST_FUNCTION_MAP = { ... }
BULK_UPDATE_HARDCODED_BASKET_ITEMS_FUNCTION_MAP = { ... }
CREATE_ITEM_FOR_BASKET_FUNCTION_MAP = { ... }
VOID_ITEM_FOR_BASKET_FUNCTION_MAP = { ... }
RETURN_ITEM_VIA_BASKET_FUNCTION_MAP = { ... }
UPDATE_ITEM_FOR_BASKET_FUNCTION_MAP = { ... }
MODIFY_ITEM_NOTES_FUNCTION_MAP = { ... }
# Loyalty maps:
LIST_LOYALTY_COUPONS_FUNCTION_MAP = { ... }
REDEEM_LOYALTY_COUPON_FUNCTION_MAP = { ... }
UNREDEEM_LOYALTY_COUPON_FUNCTION_MAP = { ... }
BASKET_REDEEM_UNREDEEM_LOYALTY_COUPON_FUNCTION_MAP = { ... }
REDEEM_LOYALTY_POINTS_FUNCTION_MAP = { ... }
```

The dispatch resolves in two styles:
1. **Function-based** (`types.FunctionType`) — called directly: `fn(api_data, store_dict)`
2. **Class-based** — instantiated, then method called: `cls(api_data, store_dict).add_or_update_basket()`

Source: `app_settings.py` (50K, ~700+ store-type-to-function mappings)

### Helper Functions for Dispatch

Each tier provides helper functions that serialize the store and look up the appropriate dispatch function:

| Helper | Source | Returns |
|---|---|---|
| `_get_store_obj_and_scan_function(store_id)` | `api.py:66` | `store_dict` with `scan_function` |
| `_get_store_obj_and_item_to_basket_function(store_id)` | `api.py:75` | `store_dict` with `add_item_to_basket` |
| `_get_store_obj_and_get_basket_details_function(store_id)` | `api.py:91` | `store_dict` with `get_basket_details` |
| `_get_store_obj_and_remove_item_from_basket_function(store_id)` | `api.py:115` | `store_dict` with `remove_item_from_basket` |
| `_get_store_obj_and_create_item_for_basket_function(store_id)` | `api.py:131` | `store_dict` with `create_item_for_basket` |
| `_get_store_obj_and_void_item_for_basket_function(store)` | `views.py:94` | `store_dict` with `void_item_for_basket` |
| `_get_store_obj_and_loyalty_list_function(store)` | `views.py:102` | `store_dict` with `loyalty_list` |
| `_get_store_obj_and_redeem_loyalty_function(store)` | `views.py:110` | `store_dict` with `loyalty_redeem` |

The V2 helpers (`api_v2.py`) also add `store_dict['store'] = store` (the Django ORM object), unlike V1 helpers which only store the serialized dict.

---

## URL Routing

All item routes are registered under the `items/` prefix in the main URL configuration. The module defines **47 URL patterns** across three tiers.

Source: `urls.py:1–144`

### V1 Public Endpoints

| # | Method | Path | View Class | Name | Auth |
|---|---|---|---|---|---|
| 1 | GET | `v1/item-scan/` | `ItemScan` | `scan_an_item` | IsAuthenticated |
| 2 | GET | `v1/item-recommendations/` | `ItemRecommendations` | `item_recommendations` | IsAuthenticated |
| 3 | POST | `v1/add-item-to-basket/` | `AddItemToBasket` | `add_item_to_basket` | IsAuthenticated, ActiveStore |
| 4 | POST | `v1/bulk-add-item-to-basket-old/` | `BulkAddItemToBasket` | `bulk_add_item_to_basket` | IsAuthenticated, ActiveStore |
| 5 | POST | `v1/bulk-update-hardcoded-basket-items/` | `BulkUpdateHardcodedBasketItems` | `bulk_update_hardcoded_basket_items` | IsAuthenticated, ActiveStore |
| 6 | GET | `v1/get-basket-details/` | `GetBasketDetails` | `get_basket_details` | IsAuthenticated |
| 7 | POST | `v1/add-item-to-wishlist/` | `AddItemToWishlist` | `add_item_to_wishlist` | IsAuthenticated |
| 8 | GET | `v1/get-wishlist-details/` | `GetWishlistDetails` | `get_wishlist_details` | IsAuthenticated |
| 9 | POST | `v1/remove-item-from-basket/` | `RemoveItemFromBasket` | `remove_item_from_basket` | IsAuthenticated |
| 10 | POST | `v1/remove-item-from-wishlist/` | `RemoveItemFromWishlist` | `remove_item_from_wishlist` | IsAuthenticated |
| 11 | POST | `v1/send-email-for-wishlist/` | `SendEmailForWishlist` | `send_email_for_wishlist` | IsAuthenticated |
| 12 | POST | `v1/rate-item/` | `RateItem` | `rate_item` | IsAuthenticated |
| 13 | GET | `v1/get-hard-coded-items/` | `GetHardcodedItems` | `get_hardcoded_items` | None |
| 14 | GET | `v1/promotions/` | `PromotionList` | `promotions` | IsAuthenticated |
| 15 | GET | `v1/apply-picwic-promotions/` | `ApplyPicwicPromotion` | `apply_picwic_promotions` | IsAuthenticated |
| 16 | GET/POST | `v1/manual-check/<basket_id>/` | `ManualVerification` | `manual_verification` | None |
| 17 | GET | `v1/search/` | `ItemSearch` | `item_search` | IsAuthenticated |
| 18 | POST | `v1/create-item-for-basket/` | `CreateItemForBasket` | `create_item_for_basket` | IsAuthenticated |
| 19 | POST | `v1/update-item-from-basket/` | `UpdateItemFromBasket` | `update_item_from_basket` | IsAuthenticated, ActiveStore, PriceOverride |
| 20 | POST | `v1/return-item-via-basket/` | `ReturnItemViaBasket` | `return_item_via_basket` | IsAuthenticated, ActiveStore, ActiveStaffShift |
| 21 | POST | `v1/modify-item-notes/` | `ModifyItemNotes` | `modify_item_notes` | IsAuthenticated, ActiveStore, ActiveStaffShift |
| 22 | POST | `v1/basket_redeem_points` | `BasketRedeemPoints` | `redeem_points_on_basket` | IsAuthenticated |
| 23 | POST | `v1/basket_cancel_redeemed_points` | `BasketCancelRedeemedPoints` | `reject_redeemed_points` | IsAuthenticated |
| 24 | PATCH | `v1/baskets/<basket_id>/` | `UpdateBasketProperties` | `update_basket_properties` | IsAuthenticated, ActiveStore, Dashboard |
| 25 | POST | `v1/basket_save_rewards` | `BasketSaveRewards` | `save_rewards_on_basket` | IsAuthenticated |
| 26 | POST | `v1/list-addon-items/` | `AddOnItems` | `list_addon_items` | None |
| 27 | POST | `v1/list-loyalty-coupons` | `ListLoyaltyCoupons` | `list_loyalty_coupons` | IsAuthenticated, ActiveStaffShift |
| 28 | POST | `v1/redeem-loyalty` | `RedeemLoyalty` | `redeem_loyalty` | IsAuthenticated, ActiveStaffShift |
| 29 | POST | `v1/unredeem-loyalty` | `UnRedeemLoyalty` | `unredeem_loyalty` | IsAuthenticated, ActiveStaffShift |
| 30 | POST | `v1/basket-redeem-unredeem-loyalty` | `BasketRedeemUnredeemLoyalty` | `basket_redeem_unredeem_loyalty` | IsAuthenticated, ActiveStaffShift |
| 31 | POST | `v1/redeem-points` | `RedeemLoyaltyPointsView` | `redeem_points` | IsAuthenticated, ActiveStaffShift |
| 32 | GET | `v1/carters-image-proxy/` | `image_proxy` | `carters_image_proxy` | — |
| 33 | PATCH | `v1/inventory/update/` | `InventoryUpdate` | `inventory_update` | ActiveStore, InventoryUpdate, UserStore |

### V2 Enhanced Endpoints

| # | Method | Path | View Class | Name | Auth |
|---|---|---|---|---|---|
| 34 | GET | `v2/item-scan/` | `ItemScanV2` | `scan_an_item_v2` | IsAuthenticated, ActiveStore, ActiveStaffShift |
| 35 | POST | `v2/add-item-to-basket/` | `AddItemToBasketV2` | `add_item_to_basket_v2` | IsAuthenticated, ActiveStore, ActiveStaffShift |
| 36 | POST | `v2/bulk-add-item-to-basket/` | `BulkAddItemToBasketV2` | `bulk_add_item_to_basket_v2` | IsAuthenticated, ActiveStore, ActiveStaffShift |
| 37 | GET | `v2/get-basket-details/` | `GetBasketDetailsV2` | `get_basket_details_v2` | IsAuthenticated, ActiveStore, ActiveStaffShift |
| 38 | POST | `v2/remove-item-from-basket/` | `RemoveItemFromBasketV2` | `remove_item_from_basket_v2` | IsAuthenticated, ActiveStore, ActiveStaffShift |
| 39 | POST | `v2/create-item-for-basket/` | `CreateItemForBasketV2` | `create_item_for_basket_v2` | IsAuthenticated, ActiveStore, ActiveStaffShift |
| 40 | GET | `v2/add-on-items/` | `GetAddOnItems` | `add-on-items` | None |
| 41 | GET/POST | `v2/promotions/` | `PromotionListV2` | `list_or_download_promotions_v2` | MLSEDashboardPromosView |
| 42 | POST | `v2/staff/manual-verify-kiosk/` | `ManualVerificationKiosk` | `manual_verification_kiosk` | Dashboard |
| 43 | POST | `v1/user/manual-verify/` | `UserManualVerification` | `user_manual_verification` | IsAuthenticated |

### Internal Service APIs

| # | Method | Path | View Class | Name |
|---|---|---|---|---|
| 44 | POST | `api/v1/item-scan/` | `ItemScanService` | `scan_an_item_as_service` |
| 45 | POST | `api/v1/add-item-to-basket/` | `AddItemToBasketService` | `add_item_to_basket_as_service` |
| 46 | POST | `api/v1/bulk-update-hardcoded-basket-items/` | `BulkUpdateHardcodedBasketItemsService` | `bulk_update_hardcoded_basket_items_as_service` |
| 47 | POST | `api/v1/add-item-to-wishlist/` | `AddItemToWishlistService` | `add_item_to_wishlist_as_service` |
| 48 | POST | `api/v1/get-basket-details/` | `GetBasketDetailsService` | `get_basket_details_as_service` |
| 49 | POST | `api/v1/get-wishlist-details/` | `GetWishlistDetailsService` | `get_wishlist_details_as_service` |
| 50 | POST | `api/v1/remove-item-from-basket/` | `RemoveItemFromBasketService` | `remove_item_from_basket_as_service` |
| 51 | POST | `api/v1/remove-item-from-wishlist/` | `RemoveItemFromWishlistService` | `remove_item_from_wishlist_as_service` |
| 52 | POST | `api/v1/rate-item/` | `RateItemService` | `rate_item_as_service` |
| 53 | POST | `api/v1/apply-picwic-promotions/` | `ApplyPicwicPromotionService` | `apply_picwic_promotions` |

Internal APIs have **no authentication** — they are called by `ItemManagementClient` from the V1 public tier and are expected to be network-isolated.

---

## API Endpoints — Detail

### Item Scanning

#### `ItemScan` — V1 (`views.py:140`)

**GET** `v1/item-scan/`

Scans an item by barcode. Validates input via `ItemScanSerializer`, then **delegates to `ItemManagementClient.scan_item()`** which makes an HTTP call to the internal service API.

**Query params:** `product_identifier`, `store_id`, `platform`, `version`

**Behavior:**
1. Validates input via `ItemScanSerializer`
2. Checks store `blocked_for_platform` and `blocked_version` in `store.extra_data` — returns 503 if platform/version is blocked
3. Delegates to `ItemManagementClient.scan_item(data)` (HTTP call to internal `api/v1/item-scan/`)
4. Records scan analytics via `record_item_scan_analytics()`

**Internal dispatch** (`ItemScanService`, `api.py:140`): Extracts barcode via `extract_barcode()`, looks up store-type scan function from `ITEM_SCAN_FUNCTION_MAP`, dispatches via function or class call.

#### `ItemScanV2` — V2 (`api_v2.py:156`)

**GET** `v2/item-scan/`

Enhanced scan endpoint for kiosk/staff use. **Dispatches directly** to store-type function (no ItemManagementClient hop). Supports multiple barcodes in a single request (comma-separated, used by Virgin).

**Additional behavior vs V1:**
- Uses `request.store` from middleware (not DB lookup)
- Requires `ActiveStorePermission` + `ActiveStaffShiftPermission`
- Supports `product_identifier_type` parameter (e.g., `code128`) for barcode type metadata
- Virgin-specific multi-barcode handling: splits on comma, extracts each barcode
- NudeFood and RetailReload: URLs as barcodes (no `extract_barcode()`)
- Records scan analytics

Source: `api_v2.py:156–272`

---

### Basket Operations

#### `AddItemToBasket` — V1 (`views.py:284`)

**POST** `v1/add-item-to-basket/`

Adds an item to a customer's basket. Supports two modes:
- **New item** (no `item_id`): uses `AddItemToBasketSerializer`
- **Existing item** (`item_id` in POST data): uses `BasketAndItemIdCheckSerializer` (updates quantity)

Delegates to `ItemManagementClient.add_item_to_basket()`.

**Custom status 460** returned when basket is expired (detected by serializer).

#### `AddItemToBasketV2` — V2 (`api_v2.py:335`)

**POST** `v2/add-item-to-basket/`

Enhanced add-to-basket for kiosk/staff. Direct dispatch, no ItemManagementClient.

**Additional params:**
- `retailer_device_id` (from query params)
- `is_item_scanned` (from query params, boolean)
- `is_student_discount` (from query params)
- `exclude_basket_promo_labels_at_item_level`

**Key difference:** Calls `expire_other_store_baskets()` **after** validation (V1 BulkAdd calls it before dispatch, V2 Add calls it after serializer validation).

Source: `api_v2.py:335–407`

#### `BulkAddItemToBasket` — V1 (`views.py:327`)

**POST** `v1/bulk-add-item-to-basket-old/`

Adds multiple items in a loop. Each item is processed individually through the store-type dispatch. Failed items are collected in an `error_items` list.

**Note:** URL suffix `-old` — replaced by `BulkAddItemToBasketV2` at the same base path `v1/bulk-add-item-to-basket/`.

#### `BulkAddItemToBasketV2` — V2 (`api_v2.py:729`)

**POST** `v1/bulk-add-item-to-basket/` and `v2/bulk-add-item-to-basket/`

Improved bulk add. Pre-processes all items to extract/modify barcodes, then calls `bulk_add_or_update_basket()` on the store-type class in a single operation (vs V1's per-item loop).

Source: `api_v2.py:729–803`

#### `RemoveItemFromBasket` — V1 (`views.py:568`)

**POST** `v1/remove-item-from-basket/`

Removes an item from a basket. Validates via `RemoveItemFromBasketSerializer`, delegates to `ItemManagementClient.remove_item_from_basket()`.

#### `RemoveItemFromBasketV2` — V2 (`api_v2.py:410`)

**POST** `v2/remove-item-from-basket/`

Direct dispatch version with kiosk/staff permissions. Passes `retailer_device_id` from query params.

Source: `api_v2.py:410–470`

#### `GetBasketDetails` — V1 (`views.py:455`)

**GET** `v1/get-basket-details/`

Retrieves basket contents and state. Validates via `GetBasketDetailsSerializer`, delegates to `ItemManagementClient.get_basket_details()`.

**Query params:** `basket_id`, `store_id`, `platform`, `exclude_basket_promo_labels_at_item_level`

#### `GetBasketDetailsV2` — V2 (`api_v2.py:275`)

**GET** `v2/get-basket-details/`

Direct dispatch. Adds `force_calculate_basket` query param support — when `true`, forces basket recalculation even if cached values exist.

Source: `api_v2.py:275–332`

#### `BulkUpdateHardcodedBasketItems` — V1 (`views.py:417`)

**POST** `v1/bulk-update-hardcoded-basket-items/`

Updates items in the "HARDCODED" category (e.g., shopping bags, standard charges). Uses `BulkUpdateHardcodedBasketItemsSerializer` which enforces that only items in categories containing "HARDCODED" can be updated.

#### `UpdateBasketProperties` — V1 (`views.py:1226`)

**PATCH** `v1/baskets/<basket_id>/`

Updates basket-level properties. Currently supports three keys in `extra_data`:
- `apply_tax` — toggle tax calculation
- `split_totals` — split payment totals
- `tax_override_details` — override tax details

Mutually exclusive: setting `apply_tax` removes `tax_override_details` and vice versa.

**Auth:** IsAuthenticated + ActiveStore + DashboardPermission

Source: `views.py:1226–1277`

---

### Item Creation & Modification

#### `CreateItemForBasket` — V1 (`api.py:543`)

**POST** `v1/create-item-for-basket/`

Creates a new item and adds it to a basket (e.g., jar deposits, tips, custom items). Not a scan — creates an `ItemInventory` entry with a specified price.

**Key params:** `store_id`, `price`, `basket_id`, `is_jar_deposit`, `is_tips_item`

Uses `CreateItemForBasketSerializer` and dispatches via `CREATE_ITEM_FOR_BASKET_FUNCTION_MAP`.

#### `CreateItemForBasketV2` — V2 (`api_v2.py:532`)

**POST** `v2/create-item-for-basket/`

Enhanced version. Adds `is_shopping_bag` flag.

Source: `api_v2.py:532–591`

#### `UpdateItemFromBasket` — V2 (`api_v2.py:594`)

**POST** `v1/update-item-from-basket/`

Updates an existing basket item (e.g., price override). Requires `PriceOverridePermission`.

**Key params:** `discount_type` (default: "fixed")

Dispatches via `UPDATE_ITEM_FOR_BASKET_FUNCTION_MAP`.

Source: `api_v2.py:594–650`

#### `ReturnItemViaBasket` — V2 (`api_v2.py:473`)

**POST** `v1/return-item-via-basket/`

Handles item returns through the basket flow. Sets `is_return_item = True` in the request data and dispatches via `RETURN_ITEM_VIA_BASKET_FUNCTION_MAP`. The dispatch target calls `add_or_update_basket()` — the same method as add-to-basket, but the return flag triggers different behavior in the store-type procedure.

Source: `api_v2.py:473–529`

#### `ModifyItemNotes` — V2 (`api_v2.py:653`)

**POST** `v1/modify-item-notes/`

Modifies notes/metadata on a basket item. Uses `ModifyItemNotesSerializer` and dispatches via `MODIFY_ITEM_NOTES_FUNCTION_MAP`.

Source: `api_v2.py:653–726`

---

### Manual Verification

#### `ManualVerification` — V1 (`views.py:790`)

**GET/POST** `v1/manual-check/<basket_id>/`

Handles age/alcohol verification for baskets with restricted items.

**GET** — Returns the current verification status (`pending`/`approved`/`rejected`).

**POST** — Creates a new verification session:
1. Checks if a session already exists (returns existing status if so)
2. Creates `ManualVerificationSession` with random 4-digit code
3. Sends Firebase push notification to the store's dashboard topic

**No authentication required** — the basket_id in the URL serves as the access token.

Source: `views.py:790–861`

#### `ManualVerificationKiosk` — V2 (`views.py:864`)

**GET/POST** `v2/staff/manual-verify-kiosk/`

Staff-side verification endpoint. Requires `DashboardPermission`.

**GET** — Returns verification status for a basket.

**POST** — Accepts or rejects a verification:
- `action: "accept"` → status = APPROVED
- `action: "reject"` → status = REJECTED, then **deletes age-restricted items** from basket via `delete_age_restricted_basket_items()`

**DubaiDutyFree special handling:** On rejection, only removes items with the highest `minimum_age_required` value (not all restricted items).

Also removes addon items associated with restricted items (via `item_addons` in `extra_data`).

Source: `views.py:864–984`

#### `UserManualVerification` — V1 (`views.py:987`)

**GET/POST** `v1/user/manual-verify/`

Inherits from `ManualVerificationKiosk` but uses `IsAuthenticated` instead of `DashboardPermission`. Allows customers to check their own verification status.

---

### Wishlist Operations

#### `AddItemToWishlist` — V1 (`views.py:495`)

**POST** `v1/add-item-to-wishlist/`

Adds an item to a customer's wishlist. Supports same two modes as basket (new item or existing item).

#### `GetWishlistDetails` — V1 (`views.py:535`)

**GET** `v1/get-wishlist-details/`

Retrieves wishlist contents.

#### `RemoveItemFromWishlist` — V1 (`views.py:605`)

**POST** `v1/remove-item-from-wishlist/`

Removes an item from a wishlist.

#### `SendEmailForWishlist` — V1 (`views.py:666`)

**POST** `v1/send-email-for-wishlist/`

Toggles email notifications for a wishlist. Creates or updates a `WishlistExtraDetails` record.

All wishlist endpoints delegate to `ItemManagementClient` (same pattern as basket V1).

---

### Item Search & Discovery

#### `ItemSearch` — V1 (`views.py:1022`)

**GET** `v1/search/`

Searches items by name (case-insensitive `icontains`). **Restricted to specific store types** — currently only `NudeFoodStoreType` is allowed (enforced via `stores_with_item_search` class attribute).

**Query params:** `item_name`, `store_id`, `platform`, `version`

Returns items matching the name filter, serialized via `ItemInventorySerializer`.

**Performance note:** The code comment states "Computationally expensive" — this is a full `icontains` query without pagination, limited to NudeFood per GPP-4576.

Source: `views.py:1022–1061`

#### `ItemRecommendations` — V1 (`views.py:189`)

**GET** `v1/item-recommendations/`

Fetches promotion-based item recommendations for a scanned item.

**Query params:** `product_identifier`, `store_id`, `basket_id`

**Flow:**
1. Resolves barcode to `ItemInventory` object
2. Calls `PromoBasedItemRecommendations(store_id, customer).process([item_data])` to get recommended barcodes from the promotions engine
3. Looks up recommended items in `ItemInventory` — by `retailer_product_id` for retailer-ID-based promo stores, else by `product_identifier`
4. Deduplicates by `item_name` (`.distinct('item_name')`)
5. Records recommended items in `RecommendedItems` model (from `mishipay_retail_analytics`)

**Reitan-specific:** Filters recommendations to same category as source item.

Source: `views.py:189–281`

#### `GetHardcodedItems` — V1 (`views.py:709`)

**GET** `v1/get-hard-coded-items/`

Returns all items in categories containing "HARDCODED" (shopping bags, standard charges, etc.), grouped by store_id and then by category slug. **No authentication required.**

Source: `views.py:709–738`

#### `GetAddOnItems` — V1/V2 (`views.py:991`)

**GET** `v2/add-on-items/`

Returns items from "Shopping Bags" category for a store. **No authentication required.**

Source: `views.py:991–1017`

#### `AddOnItems` — V1 (`views.py:1474`)

**POST** `v1/list-addon-items/`

Fetches add-on items by group slug from the **inventory service** (external). Uses `InventoryV1Client.get_addon_groups()`.

**Request body:** `addon_group_slug` (list), `store_id`, `filter` (optional, with `categories` list)

**Flow:**
1. Calls inventory service to get addon groups and their items
2. Merges item data with variant data
3. Optionally filters by categories if `filter.categories` is provided
4. Returns `{addon_group_slug: [items...]}` map

Source: `views.py:1474–1571`

---

### Promotions

#### `PromotionList` — V1 (`views.py:741`)

**GET** `v1/promotions/`

Lists all platform-side promotions for a store. Uses `helpers/promotions.py:get_promotions()`.

**Query params:** `store_id`

**Flow:**
1. Fetches `Promotion` objects for the store's `store_type`
2. For each promotion, queries `ItemInventory` to find applicable items (matching `promotional_offer_codes`)
3. Groups promotions by `display_name`

Returns: `{promotion_title, offers: [{name, data}]}`

Source: `views.py:741–757`, `helpers/promotions.py:23–62`

#### `PromotionListV2` — V2 (`api_v2.py:806`)

**GET** `v2/promotions/`

Dashboard-facing promotions endpoint. Requires `MLSEDashboardPromosViewPermission` (user must be in `MLSEDashboardPromoAccess` group).

Fetches promotions from the **promotions microservice** via `PromoServiceV2Helper`.

**Query params:** `retailer_name` OR `store_id`, `download` (optional)

**GET behavior:**
1. Fetches promotions by retailer or store from microservice
2. Serializes via `PromotionsListV2ResponseSerializer`
3. Flattens and sorts by `promo_id_retailer`
4. Deduplicates display (sets `promo_id_retailer = ""` for continuation lines)
5. Paginates via `StandardResultsSetPagination`
6. If `download=true`, uploads CSV to Azure and returns `file_url`

**POST** `v2/promotions/`

Uploads promotions from a CSV file to a store.

**Request body:** `store_id`, `file` (CSV), `clean` (boolean)

If `clean=true`, deletes all non-special promotions from the store before uploading. On upload failure with `clean=true`, re-imports the deleted promotions as a rollback.

Source: `api_v2.py:806–949`

#### `ApplyPicwicPromotion` — V1 (`views.py:760`)

**GET** `v1/apply-picwic-promotions/`

Applies Picwic-specific promotions to a basket. Delegates to `ItemManagementClient.apply_picwic_promotions()`.

**Internal dispatch** (`ApplyPicwicPromotionService`, `api.py:463`): Calls `apply_picwic_promotions()` helper, processes the response to create `BasketEntityAuditLog` entries, then returns updated basket details via `compass_get_basket_details()`.

Source: `views.py:760–787`, `api.py:463–540`

---

### Loyalty & Rewards

#### `BasketRedeemPoints` — V1 (`views.py:1063`)

**POST** `v1/basket_redeem_points`

Redeems loyalty points on a basket via **Grandiose TalonOne** loyalty client.

**Request body:** `basket_id`, `membership_id`, `redeem_points`

**Flow:**
1. Validates via `RedeemPointsOnBasket` serializer
2. If previous redemption exists, adds to reject list
3. Calls `GrandioseLoyaltyClientTalonOne.redeem_member_points()`
4. On success, stores redemption data in `basket.extra_data` (transaction IDs, conversion rate, approve/reject lists)
5. On failure, sends Slack alert to `#alerts_grandiose`

Source: `views.py:1063–1117`

#### `BasketCancelRedeemedPoints` — V1 (`views.py:1120`)

**POST** `v1/basket_cancel_redeemed_points`

Cancels a previous point redemption. Calls `reject_member_points()` on the loyalty client and cleans up `basket.extra_data`.

Source: `views.py:1120–1161`

#### `BasketSaveRewards` — V1 (`views.py:1164`)

**POST** `v1/basket_save_rewards`

Saves Carters reward points to a basket. Validates via `SaveRewardsOnBasket` (enforces multiples of 5, max $30). Updates `basket.extra_data` with reward amount and loyalty customer info from `RetailerSpecificCustomerProfile`.

Source: `views.py:1164–1223`

#### `ListLoyaltyCoupons` — V1 (`views.py:1278`)

**POST** `v1/list-loyalty-coupons`

Lists available loyalty coupons from third-party APIs. Uses store-type dispatch via `LIST_LOYALTY_COUPONS_FUNCTION_MAP`.

**Auth:** IsAuthenticated + ActiveStaffShift

Source: `views.py:1278–1326`

#### `RedeemLoyalty` / `UnRedeemLoyalty` — V1 (`views.py:1329`, `views.py:1372`)

**POST** `v1/redeem-loyalty` / `v1/unredeem-loyalty`

Redeems or unredeems a loyalty coupon. Uses store-type dispatch via `REDEEM_LOYALTY_COUPON_FUNCTION_MAP` / `UNREDEEM_LOYALTY_COUPON_FUNCTION_MAP`.

**Auth:** IsAuthenticated + ActiveStaffShift

#### `BasketRedeemUnredeemLoyalty` — V1 (`views.py:1411`)

**POST** `v1/basket-redeem-unredeem-loyalty`

Combined redeem/unredeem endpoint. Action determined by `action` field (`"redeem"` or `"unredeem"`). Uses `BASKET_REDEEM_UNREDEEM_LOYALTY_COUPON_FUNCTION_MAP`.

Source: `views.py:1411–1471`

#### `RedeemLoyaltyPointsView` — V1 (`views.py:1574`)

**POST** `v1/redeem-points`

Generic loyalty points redeem/unredeem endpoint. Similar to `BasketRedeemUnredeemLoyalty` but uses `REDEEM_LOYALTY_POINTS_FUNCTION_MAP` and `BasketRedeemUnredeemPointsSerializer`.

Source: `views.py:1574–1632`

---

### Inventory Update

#### `InventoryUpdate` — V2 (`api_v2.py:952`)

**PATCH** `v1/inventory/update/`

Updates item prices in inventory. Currently used only for DubaiDutyFree (DDF).

**Auth:** ActiveStore + InventoryUpdate + UserStore (no IsAuthenticated — uses API key or store-level auth)

**Query params:** `retailer_store_id`

**Request body:** List of items (validated by `InventoryUpdateItemSerializer`, max 20,000 items per request)

**Flow:**
1. Resolves store by `retailer_store_id` + retailer ID from `request.stores`
2. Validates all items via `InventoryUpdateItemSerializer(many=True)`
3. Calls `store_inventory_update_request()` utility

Source: `api_v2.py:952–994`

---

### Ratings

#### `RateItem` — V1 (`views.py:636`)

**POST** `v1/rate-item/`

Submits a customer rating (1–5) for a product. Validates via `RateItemSerializer`, delegates to `ItemManagementClient.rate_item()`.

**Internal dispatch** (`RateItemService`, `api.py:407`): Calls `rate_item()` function from `rating_of_items.py`.

---

### Utility Endpoints

#### `image_proxy` — V1 (`urls.py:140`)

**GET** `v1/carters-image-proxy/`

Proxies image requests for Carters retailer. Function-based view from `mishipay_items.carters_utility.client`.

---

## Custom Permissions

The module defines 4 custom permission classes in `permissions.py`:

| Permission | Used By | Logic |
|---|---|---|
| `ItemScanPermission` | Not directly used in views (may be applied via middleware or unused) | Checks `EntityActionConstraint` with `ITEM_SCAN` action on the store |
| `AddUpdateBasketPermission` | Not directly used in views | Checks `EntityActionConstraint` with `ADD_UPDATE_BASKET` action |
| `AddUpdateWishlistPermission` | Not directly used in views | Checks `EntityActionConstraint` with `ADD_UPDATE_WISHLIST` action |
| `MLSEDashboardPromosViewPermission` | `PromotionListV2` | Checks user is in `MLSEDashboardPromoAccess` Django group |

The constraint-based permissions (`ItemScanPermission`, `AddUpdateBasketPermission`, `AddUpdateWishlistPermission`) use the `EntityActionConstraint` model from `mishipay_core` — which evaluates store-level constraints using `eval()`. These permissions allow dynamically disabling operations per store.

**Note:** The first three permissions are defined but not applied to any view class in the current codebase. They may be applied via `app_settings.py` configuration or used in other modules.

Source: `permissions.py:1–104`

---

## Promotion Helpers

The `helpers/promotions.py` module provides utility functions for promotion-related operations:

| Function | Purpose | Source |
|---|---|---|
| `get_promotions(data)` | Fetches platform-side promotions for a store, groups by display_name | `helpers/promotions.py:23` |
| `get_basket_store_promotions_details(basket_id)` | Returns basket-level promotions (Red Discount for Dufry, £2 off for Relay) | `helpers/promotions.py:65` |
| `get_red_discount_offer_details(customer, basket_item_obj)` | Applies Red Discount tier-based offers for Dufry customers | `helpers/promotions.py:112` |
| `get_total_value_of_red_discount_items(basket_obj)` | Sums savings from Red Discount promotions on a basket | `helpers/promotions.py:136` |
| `get_special_promo_request(store, basket, program, coupon_code, **kwargs)` | Builds a promotion microservice request for special/on-the-fly promotions | `helpers/promotions.py:146` |

**Red Discount** is a Dufry-specific loyalty tier discount. It checks `OAuthAccount` for OpenID-connected customers and maps their tier to a retailer code (`GET_RED_DISCOUNT_RETAILER_CODE_MAPPING`).

**Relay £2 off** — for Relay stores, if basket amount after promotions is ≥ £5, applies retailer code `905000619`.

---

## Business Logic

### Promotion Evaluation Engine (`mpay_promo.py`)

The `mpay_promo.py` file (780 lines, 31K) is the **adapter/facade layer** between the Django monolith and the external [Promotions Microservice](../promotions-service/overview.md). It translates basket data into the microservice's API contract and post-processes responses.

#### Architecture

```
Django Monolith (basket, items, store)
        │
        ▼
mpay_promo.py (Adapter Layer)
        │
        ▼
Promotions Microservice (service.promotions)
  POST /api/1.0/promotions/evaluate/
  POST /api/1.0/promotions/transaction/
  POST /api/1.0/promotions/suggestions/
  POST /api/1.0/promotions/item_recommendations/
```

#### Data Model Classes

| Class | Lines | Purpose |
|---|---|---|
| `Node` | `85–113` | Single promotion node (item/category in a group). Fields: `node_id`, `discount_type`, `discount_value`, `sku`, `barcode`, `name` |
| `Group` | `116–148` | Promotion group (tier). Contains list of `Node` objects + `qty_or_value_min/max` thresholds |
| `Promotion` | `151–261` | Full promotion configuration (23 fields): retailer, family, layer, discount config, scheduling, groups, special_promo_info |

All three classes provide `from_json()`, `to_json()`, and `clone()` methods.

#### Promotion Management Classes

| Class | Lines | Purpose |
|---|---|---|
| `PromotionEntityOperation` | `263–289` | Single promotion CRUD — `get()`, `delete()` via MS REST API |
| `PromotionBatchOperation` | `325–450` | Queue-based batch operations. Queues create/update/delete/set_nodes/set_groups/set_special_promo_info, then `commit()` sends all as a single transaction POST to `/api/1.0/promotions/transaction/` in chunks of 10,000 |
| `QueueEntry` | `319–323` | Internal queue entry data class |

#### Evaluation Classes

| Class | Lines | Purpose |
|---|---|---|
| `EvaluateBasket` | `469–682` | **Core evaluation engine**. Transforms basket items into MS request format, calls `/api/1.0/promotions/evaluate/`, post-processes response |
| `PromoSuggestion` | `684–732` | Gets suggested promotions for items. Calls `/api/1.0/promotions/suggestions/` |
| `PromoBasedItemRecommendations` | `734–780` | Gets recommended items based on active promotions. Calls `/api/1.0/promotions/item_recommendations/` |

#### EvaluateBasket Flow

1. **Iterate basket items** — extract SKU (`retailer_product_id` or `product_identifier` based on store type), price, categories, weight, addons
2. **Category extraction** — supports two formats:
   - **New format** (preferred): `categories` array + `promo_categories_new` from `extra_data`
   - **Deprecated format**: `item_category`, `sub_category`, `mixmatch_id`, `promo_categories`
3. **Special category tagging** — adds tags for: Bottle Deposit, Donation Item, RTC Item, Tips Item, Return Item, Zero Priced (MLSE/SeventhHeaven), Price Override (KikoMilano)
4. **Item exclusion** — per-retailer excluded categories (items not eligible for any discount):
   - FlyingTiger: "Recycle / Deposit (Adm.)", "Donation", "Return Item"
   - Charlotte/Univision/Paradies: "Bottle Deposit"
   - EventNetwork: "Donation Item", "Tips Item"
   - Evokes: "RTC Item"
   - SSP: dynamic from GlobalConfig `SSP-US.additional_charge_types`
   - MLSE: "Return Item", "Zero Priced", "OnSale", "Donation Item"
   - KikoMilano: "Price Overide"
   - SeventhHeaven: "Zero Priced"
5. **Build MS request** — JSON with `version`, `customer_id`, `store_id`, `basket` (items + totals), `special_promos_info`
6. **POST to microservice** — 10-second timeout, expects 200 OK
7. **Post-process response** — append excluded items back to response, recalculate totals (`total_mrp`, `total_sp`, `total_after_promos`)

Source: `mpay_promo.py:469–682`

#### Custom Exceptions

| Exception | Line | Purpose |
|---|---|---|
| `GetError` | `57` | Fetching promotions from MS failed |
| `CreateError` | `61` | Creating/updating promotions via MS failed |
| `EvaluateError` | `65` | Basket evaluation failed |
| `PromotionEntityError` | `68` | Promotion entity operation failed |

#### Known Issues

- **Hardcoded timezone**: Store timezone is `"UTC+1:00"` for all stores (`mpay_promo.py:645`) — incorrect for global retailers
- **Bug**: `PromotionBatchOperation.load()` at line 302 **returns** an error instead of **raising** it — callers receive an Exception object silently
- **No retry logic**: 10-second timeout with no retries on MS evaluation calls
- **Response not logged**: Only the request is logged (`mpay_promo.py:661`); discount results are not logged, making debugging difficult
- **13+ hardcoded store-type checks**: Each new retailer may require updates to category handling, exclusion lists, and special category tags

---

### Barcode Scanning System (`item_scan_functions.py`)

The `item_scan_functions.py` file (549 lines, 23K) handles product identification across multiple retailer ecosystems. It provides retailer-specific scan functions that are dispatched via `ITEM_SCAN_FUNCTION_MAP` in `app_settings.py`.

#### Functions

| Function | Lines | Purpose |
|---|---|---|
| `mishipay_item_scan()` | `45–155` | Core domestic inventory lookup with barcode normalization |
| `nedap_item_scan()` | `158–226` | Netherlands RFID/barcode integration via Nedap API |
| `cisco_item_scan()` | `235–291` | Cisco San Jose venue — EPC binary decoding + Clover POS lookup |
| `cisco_deconstruct_epc()` | `229–232` | Utility: hex EPC → binary → extract 24-bit product code (bits 32–55) |
| `saturn_item_scan()` | `294–495` | German retailer — NFC label support, Saturn Product API, pfund (deposit) handling |
| `decathlon_item_scan()` | `498–530` | Decathlon — delegates to `DecathlonItemFetch` third-party integration |
| `avery_dennison_item_scan()` | `533–549` | RFID tag integration via `AveryDennisonItemFetch` |

#### Barcode Processing Logic (`mishipay_item_scan`)

1. **Catalog lookup** — exact match on `ItemCatalog.catalog_identifier` (returns multiple items if matched)
2. **Inventory lookup** — exact match on `ItemInventory.product_identifier` for the store
3. **Barcode type normalization** — based on `product_identifier_type` header:
   - `"upca"`, `"ean13upca"`, `"ean-13"`, `"upc-a"`: EAN-13 (13 digits with leading `0`) → strip first digit to get UPC-A; shorter codes → left-pad with zeros to 13 digits
   - `"interleavedtwooffive"`: strip leading zero, then **suffix match** (`product_identifier__endswith`) for flexible partial matching
4. **Response enrichment** — all scan results are enriched with wishlist status and item rating

#### Barcode Format Support

| Format | Handling | Retailer |
|---|---|---|
| EAN-13/UPC-A | Normalization & conversion | All domestic |
| Interleaved 2-of-5 | Suffix matching | All domestic |
| Generic barcode | Exact match | All domestic |
| NFC Label | Tag ID → Product ID via NFC API | Saturn |
| EPC (RFID hex) | Hex → binary → 24-bit product code extraction | Cisco |
| EPC (RFID long) | Substring GTIN extraction (positions 2–16) | Nedap |

#### External Service Integrations

| Service | Function | Method | Auth |
|---|---|---|---|
| Nedap API | `nedap_item_scan()` | GET | Bearer token from store settings |
| Saturn NFC API | `saturn_item_scan()` | GET | API key |
| Saturn Product API | `saturn_item_scan()` | GET | API key |
| Clover POS | `cisco_item_scan()` | Via `CloverItemFetch` | Configured per store |
| Decathlon API | `decathlon_item_scan()` | Via `DecathlonItemFetch` | Configured per store |
| Avery Dennison | `avery_dennison_item_scan()` | Via `AveryDennisonItemFetch` | Configured per store |

#### Known Issues

- **Hardcoded values**: country `'netherlands'` (Nedap, line 188), `'usa'` (Cisco, line 270), VAT rate `19.0` (Saturn, line 464), stock quantities `1000`/`10000`
- **No input validation**: `product_identifier` is never checked for empty/null/charset
- **Unvalidated API responses**: External API JSON is accessed without structure validation — will crash on unexpected formats
- **Uses `print()` instead of logger**: Saturn function has 3+ `print()` calls in production code
- **3–4 DB queries per scan minimum**: ItemCatalog + ItemInventory + WishlistItem + Rating (no caching)
- **Wishlist query bug**: Uses `wishlist_obj[0]` list indexing without empty-list check — should use `.first()`

Source: `item_scan_functions.py:1–549`

---

### Promotion Handlers Subsystem (`promotions/`)

The `promotions/` directory (10 files, ~123K) implements a **plugin-based promotion handler system** for platform-side promotions. These handlers are referenced by `PromotionVariant.handler` and dynamically loaded via `Promotion.apply_offer()` using `__import__()`.

#### Directory Structure

```
promotions/
├── __init__.py                          (0B — empty)
├── handlers.py                          (39K — 9 generic handlers)
├── constants.py                         (1.5K — entity class constants)
├── applicability_checks.py              (1.9K — promotion eligibility checks)
├── dynamic_configuration.py             (1.6K — runtime config adjustments)
├── compass_promotions/handlers.py       (20K — 3 Compass-specific handlers)
├── eroski_promotions/handlers.py        (21K — Eroski-specific handlers)
├── picwic_promotions/handlers.py        (7.9K — 3 Picwic-specific handlers)
├── relay_promotions/handlers.py         (30K — 4 Relay-specific handlers)
└── relay_promotions/helpers.py          (1.1K — loyalty calculator)
```

#### Handler Contract

All handlers follow the same interface:

```python
class PromotionHandler:
    def __init__(self, items_detail, **kwargs):
        self.items_detail = items_detail      # List of item dicts
        self.meta_data = kwargs["meta_data"]  # Config from Promotion.meta_data
        self.applicable_entity_type = "item" | "group" | "basket"

    def apply(self, **kwargs) -> dict:
        # Returns: {"db_entries": [...], "items_summary": [...], "applicable_entity_type": str}
```

Each offer detail record contains: `item_id`, `quantity_used`, `quantity_remaining`, `actual_price`, `discounted_price`, `discount`, `price_before_discount`, `price_after_discount`, `applicable_entity_type`, `applicable_entity_value`, and optional `deal_name`, `combo_discount`, `extra_data`.

#### Generic Handlers (`handlers.py`)

| Handler | Lines | Discount Type | Description |
|---|---|---|---|
| `SetValueHandler` | `11–45` | Fixed amount | Reduces item price by a fixed amount |
| `DiscountPercentHandler` | `48–82` | Percentage | Applies percentage discount to each item |
| `BuyXforYPriceHandler` | `85–118` | Fixed group price | Buy X units for fixed Y price |
| `BuyAnyXSaveYPercentHandler` | `121–275` | Group percentage | Buy any X items, get Y% off the group |
| `BuyXSecondYPercentOffHandler` | `308–474` | Combo percentage | Buy X, get Y% off the second set of X |
| `DiscountPriceHandler` | `477–510` | Fixed per-unit | Fixed amount discount per unit quantity |
| `BuyAnyXForYpriceHandler` | `513–689` | Group fixed price | Buy any X items for fixed Y total price |
| `BaseMealDealComboOffer` | `692–727` | Abstract combo | Base class for meal deal combos |
| `BuyXSecondYPercentOffIncentivesHandler` | `728–903` | Combo + category | Buy X, get Y% off with category constraints |
| `handle_non_offers_items()` | `278–305` | No promotion | Creates "nopromo" audit records for items without discounts |

#### Retailer-Specific Handlers

| Retailer | Module | Handlers | Key Feature |
|---|---|---|---|
| **Compass** | `compass_promotions/handlers.py` | `MealDealComboOffer`, `HealthyMealDealComboOffer`, `SeasonalDiscountPercentHandler` | 3-item meal deals (Meal+Snack+Drink categories), one-time seasonal discount per user |
| **Eroski** | `eroski_promotions/handlers.py` | Same structure as Compass | Parallel implementation for Eroski retailer |
| **Picwic** | `picwic_promotions/handlers.py` | `BuyNUnitsGetYPercentOnMQuantityHandler`, `BuyNUnitsGetYPercentOnNUnitsHandler`, `BuyNUnitsGetFixedPriceOfNUnitsHandler` | Threshold-based tiered discounts (buy N, get % or fixed price) |
| **Relay** | `relay_promotions/handlers.py` | `MealDealComboOneOffer`, `MealDealComboTwoOffer`, `RelayBuyXSecondYPercentOffHandler`, `RelayBuyForXAmountGetYDiscountOnFirstOrder` | Capped combos with `max_instances_per_basket`, basket-level spend thresholds |

#### Applicability Check System (`applicability_checks.py`)

Determines whether a promotion applies to a given basket/item context:

```python
PromotionApplicabilityCheck(
    retailer_code, item_data, store, customer
).is_applicable()  # Returns bool (AND logic — all checks must pass)
```

**Registry** (`PromotionApplicabilityConfig.config`): maps promotion type names to check functions. Currently only `relay_coffee_incentive` is registered — checks if store is an airport store and customer has available free espressos.

Source: `promotions/applicability_checks.py:7–64`

#### Dynamic Configuration System (`dynamic_configuration.py`)

Allows promotions to adjust their `meta_data` at runtime based on customer state:

```python
DynamicConfiguration(
    retailer_code, item_data, store, customer
).get_offer_updates()  # Returns dict with updated meta_data
```

Currently only `relay_coffee_incentive_dynamic_config` is registered — sets `max_instances_per_basket` based on the customer's accumulated coffee purchase history.

The free espresso calculation (`relay_promotions/helpers.py:4–24`) queries `CustomerInterface.get_past_purchase_records()` for the customer's coffee purchase count, then computes available free coffees as `(total_coffees / unit) - already_used`.

Source: `promotions/dynamic_configuration.py:5–55`

#### Entity Class Constants (`constants.py`)

| Constant | Value | Meaning |
|---|---|---|
| `ITEM_CLASS` | `'item'` | Discount on single item |
| `GROUP_CLASS` | `'group'` | Discount on combo group |
| `BASKET_CLASS` | `'basket'` | Discount on entire basket |
| `CUSTOMER_CLASS` | `'customer'` | Discount on customer account (not implemented) |
| `NO_PROMOTION_CLASS` | `'nopromo'` | Item not eligible for discount |

Also defines `PROMOTION_OFFER_SUFFIX` with localized UI labels (French/English) and `PromotionApplicableClassChoices` for model field choices.

---

### Store-Type Procedures (`procedures/`)

The `procedures/` directory contains **73 files** implementing a **hierarchical strategy pattern** for basket operations. Each retailer customizes behavior by extending a base class and overriding specific methods.

#### Inheritance Hierarchy

```
SimpleBasketProcedureV2              (simple_v2.py, 72K, ~1621 lines)
  └── PromotionBasketProcedureV2     (ms_promo_v2.py, 33K, ~729 lines)
      └── InventoryBasketProcedureV2 (inventory_service_v2.py, 41K, ~930 lines)
          └── [Retailer Procedures]  (73 files, 400B–96K each)
```

#### Base Class: `SimpleBasketProcedureV2` (`simple_v2.py`)

The foundational class with ~50+ methods for all basket operations:

| Method | Purpose |
|---|---|
| `get_product_data(product_identifier)` | Scans and validates items |
| `basket_create()` | Creates a new basket (DB transaction) |
| `basket_create_item()` | Adds item to basket with addon support |
| `add_or_update_basket()` | Main entry point for item operations |
| `prepare_basket_response()` | Calculates totals, taxes, promotions |
| `remove_item_from_basket()` | Item removal with cascade for addons |
| `create_order(post_data)` | Finalizes order from basket |
| `calculate_taxes(audit_logs, save, evaluate_response)` | Tax calculation via `TaxClientSelector` |
| `inventory_stock_check()` | Validates stock for click-and-collect |
| `load_item()` / `load_item_custom()` | Item lookup with customization hook |
| `process_upca()` / `upce_to_upca()` | Barcode conversion utilities |

**Custom exceptions**: `SimpleBasketError`, `InvalidBarcodeError`, `ItemNotFoundError`, `RestrictedItemError`, `StaffPassCodeScanError`, `ClientApiError`, `AddonItemError`

Source: `procedures/simple_v2.py:1–1621`

#### Layer 2: `PromotionBasketProcedureV2` (`ms_promo_v2.py`)

Extends `SimpleBasketProcedureV2` with promotion/discount evaluation:

| Method | Purpose |
|---|---|
| `promo_use_retailer_id()` | Determines if promotions reference retailer IDs |
| `get_promo_version()` | Returns promotion version (default `"v1"`) |
| `promo_label_interceptor(title, barcode)` | Customizes promo display text |
| `allow_zero_basket_for_payment()` | Override hook for zero-cost baskets |
| `allow_negative_basket_for_payment()` | Override hook for negative amounts |
| `is_loyalty_promotion(barcode)` | Checks if barcode is loyalty-based |
| `auto_apply_coupon_code()` | Automatic referral/coupon application |

Adds `ZeroBasketError` exception.

Source: `procedures/ms_promo_v2.py:1–729`

#### Layer 3: `InventoryBasketProcedureV2` (`inventory_service_v2.py`)

Extends `PromotionBasketProcedureV2` with inventory service integration:

| Method | Purpose |
|---|---|
| `uses_inventory_service()` | Checks platform-specific configuration |
| `get_product_data()` | Enhanced to call inventory service when enabled |
| `get_product_data_by_barcode_or_retailer_product_id()` | Fallback logic to legacy systems |
| `is_transaction_allowed()` | Validates transaction eligibility |

Adds exceptions: `AddonItemError`, `QuantityRestrictionError`, `EPCAlreadyExistError`, `EPCDecodeError`, `QuotaExceededError`, `ProfanityError`

Source: `procedures/inventory_service_v2.py:1–930`

#### Retailer Customization Depth

| Depth | Files | Override Count | Examples |
|---|---|---|---|
| **Minimal** (pass-through) | ~5 | 0 | `nedap.py`, `hmshost.py`, `pantree.py` (<400B) |
| **Light** | ~15 | 1–2 methods | `desigual.py`, `fast_eddys.py`, `bloomingwear.py` |
| **Moderate** | ~20 | 3–5 methods | `hudson.py`, `paradies.py`, `carters.py` |
| **Heavy** | ~5 | 10+ methods + custom classes | `dubai_duty_free.py` (38K), `flying_tiger.py` (38K), `mlse.py` (34K), `ssp.py` (27K), `bwg.py` (96K) |

#### Key Methods Commonly Overridden

| Method | Override Purpose | Examples |
|---|---|---|
| `get_product_data()` | Custom barcode handling | DDF (weighted items `28XXWEIGHTXX`), MLSE (EPC RFID decoding), SSP (POS menu lookup) |
| `add_extra_special_promos_based_on_basket_and_customer()` | Custom promotion application | FlyingTiger (first-time/repeat user discounts, Mishipoly loyalty) |
| `load_item_custom()` | Custom item resolution | CommonBasketProcedure (barcode cleaning) |
| `allow_zero_basket_for_payment()` | Payment validation | StandardBasketProcedure (allows zero) |
| `calculate_taxes()` | Tax computation | DDF (multi-currency), SSP (POS-synced) |
| `create_order()` | Order finalization | SSP (Omnivore POS integration), DDF (quota validation) |

#### Notable Heavy Retailer Implementations

**DubaiDutyFreeBasketProcedure** (`dubai_duty_free.py`, 38K) — weighted item barcode parsing (`28XXWEIGHTXX` format), boarding pass destination validation, AMEX card QR discounts, duty-free quota management via `DDFQuota`, multi-currency VAT handling.

**FlyingTigerBasketProcedure** (`flying_tiger.py`, 38K) — time-bounded promotions (first-time user, repeat within 6 weeks), regional variant logic (GB-specific), marketing consent validation, bottle deposit/return workflow, Mishipoly loyalty integration.

**MLSEBasketProcedure** (`mlse.py`, 34K) — EPC RFID decoding from 24-char tags, pseudo GTIN parsing, jersey name/number customization with profanity checking, MLSE Bucks split payment tracking, budget code management.

**SSPBasketProcedure** (`ssp.py`, 27K) — Omnivore POS system integration, external menu service lookup, basket-to-POS total reconciliation, UPC-E to UPC-A barcode conversion, food service ticketing.

---

### Additional Helper Modules

#### Compass Promotions Helper (`helpers/compass_promotions.py`, 9K)

Handles promotion application logic for Compass store type:

| Function | Line | Purpose |
|---|---|---|
| `sort_offers_by_priority(offers_dict)` | `15` | Sorts offers by priority for sequential application |
| `populate_offer_data(basket, offer_data, items_dict, applied_offer_data)` | `20` | Creates `BasketEntityAuditLog` entries with offer details |
| `apply_compass_promotions(basket_id)` | `51` | Main orchestration: applies all promotions in priority order with two-phase processing (item offers first, then non-offer items via `handle_non_offers_items`) |
| `populate_basket_level_offer_data(basket, offer_data, applied_offer_data)` | `143` | Creates audit logs for basket-level promotions |
| `apply_basket_promotions(basket_id)` | `172` | Applies basket-level promotions |
| `update_offer_data(offer_data, items_dict)` | `204` | Updates item quantities after previous offers applied |
| `apply_offer(offer_id, offer_data)` | `221` | Retrieves promotion and applies handler |

Uses `PromotionApplicabilityCheck` and `DynamicConfiguration` before each offer application. Tracks `quantity_requested`, `quantity_used`, `quantity_remaining` per item.

Source: `helpers/compass_promotions.py:15–229`

#### Picwic Promotions Helper (`helpers/picwic_promotions.py`, 11K)

Integrates with an external Picwic SOAP/XML promotion service:

| Function | Line | Purpose |
|---|---|---|
| `populate_basket_audit_log_picwic(basket, offer_data, item_data_dict)` | `20` | Creates audit log entries for Picwic promotions |
| `apply_picwic_promotions(basket)` | `41` | Calls external Picwic promotion service via SOAP/XML POST, parses response, applies discounts, extracts loyalty points |

**Flow**: Builds XML request using Django template (`picwic_promotion_service.xml`), POSTs to `promotions_service_url` (10s timeout), parses SOAP response with `xmltodict`, aggregates discounts by `product_identifier`, creates audit log entries. Stores raw service response in `basket.extra_data` for audit trail.

**Concern**: No retry logic; hardcoded SOAP namespace URNs; no XML validation on response.

Source: `helpers/picwic_promotions.py:20–276`

#### Dufry Promotions Helper (`helpers/dufry_promotions.py`, 921B)

| Function | Line | Purpose |
|---|---|---|
| `apply_basket_red_discount(basket_id, basket_price, basket_offer)` | `4` | Applies percentage-based Red Discount to entire basket by wrapping basket as a single item and delegating to `DiscountPercentHandler` |

Source: `helpers/dufry_promotions.py:4–30`

#### Loyalty Abstract Base (`helpers/loyalty.py`, 609B)

Defines the `Loyalty` abstract base class (ABC) for loyalty program integrations:

| Abstract Method | Purpose |
|---|---|
| `list_coupons()` | Retrieve available coupons |
| `redeem_coupon()` | Redeem a coupon |
| `unredeem_coupon()` | Reverse a coupon redemption |
| `get_customer()` | Fetch customer loyalty profile |

**Constructor**: `__init__(store, membership_id, basket=None)`

**Note**: Class name typo — `LoyaltyCLientError` (should be `LoyaltyClientError`).

Source: `helpers/loyalty.py:7–31`

#### VAT Calculation Helpers (`helpers/basket_items/helpers.py`, ~6.5K)

Store-type-specific VAT calculation functions:

| Function | Line | Purpose |
|---|---|---|
| `calculate_vat(basket)` | `128` | Main orchestration: determines store type, fetches items, calculates VAT, bulk updates basket items |
| `net_sales_amount(basket_item)` | `11` | Net sales (Compass: divide by 1.2; others: direct) |
| `compass_net_sales_amount(basket_item)` | `26` | Compass: removes 20% VAT multiplier |
| `net_vat_amount(basket_item, store)` | `38` | Routes to store-specific VAT function |
| `compass_net_vat_amount(basket_item)` | `70` | Compass: sales_amount × vat_rate |
| `dufry_net_vat_amount(audit_log_item)` | `104` | Dufry: tax-inclusive extraction from modified_monetary_value |
| `dufry_sales_vat_amount(basket_item)` | `115` | Dufry: extracts VAT from price using vat_rate from extra_data |
| `relay_net_vat_amount(basket_item)` | `194` | Relay: tax-inclusive formula with Decimal math |
| `mishipay_net_vat_amount(basket_item)` | `203` | MishiPay native: always returns 0 |
| `net_vat_amount_after_promotions(audit_log, store)` | `84` | Post-promotion VAT for Dufry-type stores (others return 0) |
| `populate_vat_details_in_db(item_ids, mapping, fn)` | `164` | Bulk updates basket items with calculated VAT |

**Store-type VAT routing**:

| Store Type | Formula |
|---|---|
| Compass | Divide by 1.2 (assumes 20% VAT included) |
| Dufry, Spar, Londis, FlyingTiger, Lovbjerg, Muji | Tax-inclusive: `price - price/(1 + vat_rate/100)` |
| Relay | Tax-inclusive with Decimal math |
| MishiPay native | 0 (no VAT) |
| Others | Multiply by vat_rate directly |

Uses `GET_VAT_FUNCTION_MAPPING` from `app_settings.py` and `settings.BASKET_AUDIT_LOG_NOT_CREATED_STORETYPES` to determine calculation source (basket items vs audit logs).

Source: `helpers/basket_items/helpers.py:11–203`

---

### Retailer-Specific Utilities

The module contains several retailer-specific utility subdirectories (not a standalone `utils/` directory):

#### DDF Utility (`ddf_utility/utils.py`, 18K)

| Class/Function | Lines | Purpose |
|---|---|---|
| `DDFQuota` class | `22–362` | Manages duty-free quota tracking: boarding pass parsing, flight-based customer identification, quota interval management (daily/monthly/yearly), purchase history queries |
| `DDFQuota.update_ddf_quota_order_complete()` | `251–290` | Updates customer quota after order completion |
| `DDFQuota.get_basket_item_quota_config()` | `84–130` | Determines if basket item has quota via UDA (User Defined Attributes) |
| `get_ddf_staff_details(staff_pass_no)` | `365–387` | Authenticates staff from QR code/pass number via `dashboard.models.User` |
| `get_card_brand_from_card_number(card_number)` | `390–421` | Identifies payment card brand (Visa/Mastercard/Amex/Diners) from BIN |

Source: `ddf_utility/utils.py:22–421`

#### Carters Utility (`carters_utility/utils.py`, 4.6K)

| Function/Decorator | Lines | Purpose |
|---|---|---|
| `@pricing_payload` decorator | `9–92` | Wraps basket methods to construct pricing API payload — excludes shopping bags and addon items, handles coupon codes and loyalty customer data |
| `carters_employee_login(**kwargs)` | `95–121` | Authenticates Carters employee via external auth endpoint, returns `"Carters-{username}"` on success |

Source: `carters_utility/utils.py:9–121`

#### MLSE Utility (`mlse_utility/utils.py`, 1.3K)

| Function | Lines | Purpose |
|---|---|---|
| `sort_customized_item(item_list, ignore_custom_item)` | `5–43` | Sorts item list to maintain parent→customization→addon hierarchy for UI display |

Source: `mlse_utility/utils.py:5–43`

#### Management Utils (`management/utils/utils.py`, 987B)

| Function | Lines | Purpose |
|---|---|---|
| `get_items_from_inventory_service(store_id)` | `4–39` | Fetches complete inventory from external inventory service with automatic pagination via `InventoryV1Client` |

Source: `management/utils/utils.py:4–39`

---

### Missing Source Paths

Several paths specified in the task do not exist in the codebase:

| Path | Status | Equivalent |
|---|---|---|
| `mishipay_items/services/` | **Does not exist** | Business logic lives in `procedures/`, `helpers/`, `promotions/` |
| `mishipay_items/tasks.py` | **Does not exist** | No background tasks defined in this module (tasks exist in `mishipay_retail_jobs/tasks.py`) |
| `mishipay_items/signals.py` | **Does not exist** | No signal handlers. `apps.py` is minimal with no `ready()` method |
| `mishipay_items/utils/` | **Does not exist** as standalone directory | Utils are in retailer-specific subdirectories (`ddf_utility/`, `carters_utility/`, etc.) |

---

### Test Coverage Summary

The `tests/` directory contains **110 test files** with **~744 test methods** across ~52,000 lines of test code.

#### Test Infrastructure

| Component | Description |
|---|---|
| Base classes | `MPTestCase` (mishipay_core), `APITestCase` (DRF), `TestCase` (Django), pytest fixtures |
| Factories (`factories.py`, 4.7K) | `ItemInventoryFactory`, `BasketFactory`, `ItemCategoryFactory`, `BasketItemFactory`, `BasketEntityAuditLogFactory`, `AdditionalBarcodeFactory`, `ItemPropertyFactory`, `ItemCatalogFactory`, `RateItemFactory` |
| Utilities (`util.py`, 575B) | QR code HMAC validation |
| Fixture data | 18 directories with retailer-specific CSV, JSON, and binary fixtures |

#### Coverage by Category

| Category | Files | ~Methods | Scope |
|---|---|---|---|
| **Basket & Transaction Flows** | 52 (44 procedures + 8 root) | 500+ | Retailer-specific basket procedures, complete shopping flows |
| **Promotions & Loyalty** | 10 | 100+ | Promotion evaluation, loyalty programs, coupon management |
| **Import & Data Integration** | 13 | 80+ | Inventory/promotion imports from retailers (Londis, Muji, Eroski, etc.) |
| **API Endpoints** | 15 | 60+ | REST API endpoint testing (scan, basket, search, verification) |
| **Inventory & Stock** | 8 | 50+ | Stock updates, bulk updates, inventory management |
| **Data Quality & Reconciliation** | 3 | 30+ | Basket discrepancy fixes, approval workflows, order matching |
| **Payment Processing** | 3 | 20+ | Adyen credit card flows, order modes, manual verification |
| **Retailer Demo Kits** | 12 | 80+ | Retailer-specific demo environment tests |

#### Most-Tested Retailers

Flying Tiger (6 dedicated test files), Eroski (5 files), Muji (3 files), Londis (3 files) — indicating these are the most complex implementations.

#### Largest Test Files (comprehensive regression suites)

| File | Size | Methods | Retailer |
|---|---|---|---|
| `procedures/test_seventh_heaven_basket_procedure.py` | 187K | 46 | Seventh Heaven |
| `procedures/test_mlse_basket_procedure.py` | 186K | 47 | MLSE |
| `procedures/test_grandiose_basket_procedure.py` | 138K | 41 | Grandiose |
| `procedures/test_dubai_duty_free_basket_procedure.py` | 105K | 29 | Dubai Duty Free |
| `test_flying_tiger_loyalty.py` | 120K | 23 | Flying Tiger |

#### Test Organization Pattern

- **Root test files**: Feature-focused (1–5 test methods each) — unit/integration level
- **Procedure test files**: Comprehensive scenario-based (20–80 test methods each) — full regression suites covering scanning → basket → promotion → payment → order flows
- Tests organized by **retailer/feature**, not by MVC layers

---

## Dependencies

### This module depends on:
- `dos.models` — `Store`, `Customer` (core entities)
- `mishipay_core` — `BaseModel`, `BaseAuditModel`, `Retailer`, `common_functions`, choices, permissions
- `mishipay_config` — `GlobalConfig` (runtime configuration)
- `mishipay_retail_customers` — `ADDRESS_CHOICES`
- `mishipay_retail_payments` — `PaymentStatus` (basket validation)
- `inventory_service` — `BuyingGuidanceType` enum
- `django-mptt` — tree structure for V2 `Category`
- `django-import-export` — admin bulk import/export
- `multiselectfield` — `Promotion.applicability_checks`
- `phonenumber_field` — `BasketAddress.phone_number`

### What depends on this module:
- `mishipay_retail_orders` — creates orders from baskets
- `mishipay_retail_payments` — processes payments for baskets
- `mishipay_retail_analytics` — analytics on items/baskets
- `mishipay_emails` — receipt generation from basket/order data
- `promotions` — platform-side promotion evaluation
- `mishipay_coupons` — coupon application to baskets
- `dos` — store-type-specific item operations
- 30+ retailer utility modules — retailer-specific item processing

---

## Notable Patterns

1. **Denormalized BasketItem** — When items are added to a basket, all item data is copied into `BasketItem`. This snapshot approach means basket contents are preserved even if the source `ItemInventory` changes, but creates data duplication.

2. **Dual category systems** — Legacy `ItemCategory` (per store) and V2 `Category` (per retailer, MPTT hierarchy) coexist. The V2 model is registered in admin but integration depth is unclear.

3. **Retailer-specific branching** — Throughout models, serializers, and views, store_type checks drive different behavior (e.g., Carters barcode length, BestSeller flat discounts, Connexus/Fast Eddys VAT, Eroski age verification bypasses, FlyingTiger sampling, Virgin multi-barcode scans, NudeFood URL barcodes).

4. **Dynamic handler dispatch** — `Promotion.apply_offer()` uses `__import__()` to dynamically load and instantiate promotion handler classes from `PromotionVariant.handler`. This is a plugin pattern but carries import security implications.

5. **BasketEntityAuditLog as source of truth** — For stores using the promotion microservice, the audit log (not the basket item) is the canonical source for pricing, quantities, and discount breakdowns. The complex `to_representation()` in `BasketEntityAuditLogSerializer` merges audit data with live stock info.

6. **60-day admin filters** — `BasketItemAdmin` and `BasketEntityAuditLogAdmin` restrict admin queries to the last 60 days to prevent timeout on large datasets.

7. **Cached fields on Basket** — Fields like `total_basket_amount_after_discount`, `total_tax`, `tax_level_breakup` are cached on the Basket model to avoid recalculation. These are populated by the promotion evaluation pipeline.

8. **`source_hash` for change detection** — Both `ItemInventory` and `SimplePromotion` have a `source_hash` field that stores a hash of the import data, enabling efficient "has this changed?" checks during bulk imports.

9. **Three-tier API architecture** — V1 public APIs delegate to `ItemManagementClient` (HTTP hop to internal APIs), while V2 APIs dispatch directly to store-type functions. The internal service APIs (`api/v1/`) have no authentication and are expected to be network-isolated.

10. **Store-type function map dispatch** — All item operations are routed through function maps in `app_settings.py` keyed by `store_type.lower()`. This creates ~700+ function map entries but allows per-retailer customization without inheritance.

11. **Dual dispatch styles** — Function maps can return either a plain function (called directly with `fn(api_data, store_dict)`) or a class (instantiated with `cls(api_data, store_dict)` then method called, e.g., `.add_or_update_basket()`). Both styles coexist.

12. **Custom 460 status code** — Expired baskets return HTTP 460 (non-standard) via serializer validation. The serializer sets `is_basket_expired = True` which the view checks.

13. **Promotion upload rollback** — `PromotionListV2.post()` implements manual rollback for CSV promotion uploads: if `clean=True` was used and the new upload fails, previously deleted promotions are re-created.

14. **Hierarchical procedure inheritance** — Basket operations use a 4-layer inheritance chain: `SimpleBasketProcedureV2` → `PromotionBasketProcedureV2` → `InventoryBasketProcedureV2` → `[Retailer]`. Each layer adds a concern (base ops → promotions → inventory), and retailers override only the methods they need. ~70% of retailers require only 1–3 method overrides.

15. **Dual promotion systems** — Two parallel promotion evaluation paths coexist: (a) the **promotions microservice** (`mpay_promo.py` → `service.promotions`) for complex multi-item discount evaluation, and (b) **platform-side handlers** (`promotions/handlers.py` + retailer-specific handlers) for simpler per-item or combo promotions. The microservice path is used for stores with `ms_promo_response`; the handler path is used for legacy/simple stores via `Promotion.apply_offer()`.

16. **Excluded items workaround in MS evaluation** — `EvaluateBasket.process()` removes items with excluded categories from the MS request, but then appends them back to the response and adds their prices to all totals (`total_mrp`, `total_sp`, `total_after_promos`). This ensures the microservice doesn't apply discounts to ineligible items while the caller still gets a complete basket picture.

17. **SOAP/XML integration for Picwic** — The Picwic promotions helper uses Django templates to build SOAP/XML requests, `xmltodict` to parse SOAP responses with namespace stripping, and stores raw service responses in `basket.extra_data` for audit trail. This is the only SOAP integration in the items module.

18. **Two-phase compass promotion application** — Compass promotions apply in two phases: first pass applies all eligible offers in priority order (tracking `quantity_used`/`quantity_remaining` per item), second pass calls `handle_non_offers_items()` to create "nopromo" audit log entries for unmatched items.

19. **Strategy pattern for loyalty** — The `Loyalty` abstract base class (ABC) defines a pluggable interface (`list_coupons`, `redeem_coupon`, `unredeem_coupon`, `get_customer`) that allows different loyalty provider implementations per retailer.

20. **No background tasks or signals** — Unlike most Django modules, `mishipay_items` has no `tasks.py` (background tasks) or `signals.py` (signal handlers). All processing is synchronous within request/response cycles. Background job processing lives in `mishipay_retail_jobs`.

---

## Open Questions

- Is the V2 `Category` model (MPTT) actively used in production, or is it a work-in-progress? No migration of `ItemCategory` references to `Category` was observed.
- The `stock_quantity` field on `BasketItem` has a TODO comment saying it's "pointless" since stock checks use `ItemInventory` / inventory service. Should it be removed?
- `applied_offer` on `BasketItem` has a TODO marked for deletion. Is it still read anywhere?
- The `DISCOUNT_CONSTRAINT_MAP` is hardcoded in `__init__.py`. Is this used in production, or has it been superseded by the promotions microservice?
- Why does `BasketEntityAuditLog.modified_monetary_value` use 4 decimal places while `original_monetary_value` uses 2?
- The `BasketPromotionSerializer` and `BasketItemPromotionSerializer` have identical code. Is this intentional or should they be consolidated?
- `BasketVerboseSerializerV2` appears to be defined twice in `serializers.py` (lines ~1643 and ~1856). Which definition takes precedence?
- The constraint-based permissions (`ItemScanPermission`, `AddUpdateBasketPermission`, `AddUpdateWishlistPermission`) are defined in `permissions.py` but not applied to any view in the current codebase. Are they used elsewhere (e.g., via `app_settings.py` or middleware)?
- V1 APIs use `ItemManagementClient` (HTTP hop to internal service APIs), while V2 APIs dispatch directly. Is V1 being deprecated in favor of V2, or do both tiers serve different clients?
- `ItemSearch` is restricted to `NudeFoodStoreType` only. Is there a plan to expand search to other store types, or does search happen via the inventory service for most retailers?
- `ManualVerification` has no authentication — the `basket_id` in the URL is the only access control. Is this a security concern?
- `BulkAddItemToBasket` (V1) at path `v1/bulk-add-item-to-basket-old/` — is this still in use, or has it been fully replaced by `BulkAddItemToBasketV2`?
- The `InventoryUpdate` endpoint (`v1/inventory/update/`) has no `IsAuthenticated` requirement, only `ActiveStorePermission` + `InventoryUpdatePermission` + `UserStorePermission`. How is it authenticated — API key? Store token?
- **Hardcoded timezone in mpay_promo.py** — `store_timezone` is set to `"UTC+1:00"` for all stores (`mpay_promo.py:645`). Is this correct for global retailers, or should it use the store's actual timezone?
- **Bug in PromotionBatchOperation.load()** — Line 302 of `mpay_promo.py` **returns** an error instead of **raising** it. Callers would receive an Exception object silently. Is this known?
- **Deprecated category format** — `mpay_promo.py` supports two category formats (new and deprecated). Comments say "Never use for new retailers" — are there plans to remove the deprecated path?
- **LoyaltyCLientError typo** — The exception class in `helpers/loyalty.py:7` is misspelled as `LoyaltyCLientError` instead of `LoyaltyClientError`. Is this a known issue? Renaming would affect all callers.
- **Hardcoded VAT rate for Saturn** — `item_scan_functions.py:464` hardcodes `19.0` as the VAT rate. Is this dynamic in production?
- **print() statements in production code** — `item_scan_functions.py` (Saturn scan) and several other files use `print()` instead of the logging framework. Are these intentional debug statements?
- **No background tasks in items module** — All item processing (scanning, basket operations, promotions) is synchronous. Are there plans to offload heavy operations (e.g., promotion evaluation for large baskets) to background tasks?
- **Eroski promotion handlers** mirror Compass handlers almost identically (`compass_promotions/handlers.py` vs `eroski_promotions/handlers.py`). Can these be consolidated into a shared base with retailer-specific configuration?
