# Emails, Receipts & Payment Alerts

> Last updated by: Task T14 — MishiPay Platform Alerts & Subscriptions

## Overview

The `mishipay_emails` module (136 files) is the platform's email and receipt generation system. It handles two primary subsystems:

1. **Email Configuration & Templating** -- Database-driven email configuration for transactional emails (receipts, thank-you messages) using Django's template engine.
2. **Print Receipt Generation** -- A sophisticated receipt rendering engine for kiosk/POS printers (Epson format) with retailer-specific customization via the Strategy pattern.

The module does **not** expose any REST API endpoints. All management is done through Django admin. There are no background tasks (`tasks.py`) or signal handlers (`signals.py`) -- receipt generation is entirely synchronous.

## Key Concepts

| Term | Meaning |
|------|---------|
| **Receipt Type** | Category of receipt: ORDER, REFUND, GIFT, STEB, PAYMENT_FAILED, COUNTER, PAYMENT, CAMPAIGN |
| **STEB** | Saudi Tax and Excise Board -- duty-free bag tracking receipts for Dubai Duty Free |
| **Strategy** | A retailer-specific subclass of `PrintReceipt` that customizes receipt formatting |
| **Config Fields** | JSON blob stored in `PrintReceiptConfiguration.config_fields` that controls receipt layout (column widths, labels, flags) |
| **Epson Markup** | POS printer format using tags: `[L]` (left), `[R]` (right), `[C]` (center), `<b>`, `<img>`, `<qrcode>` |
| **Hierarchical Config Lookup** | Configuration is resolved from most-specific to least: store+region > store+ALL > ALL+region > ALL+ALL |

## Architecture

```
mishipay_emails/
├── models.py                    # 4 Django models + ReceiptType constants
├── admin.py                     # 4 admin registrations with custom widgets
├── forms.py                     # 2 ModelForms with dynamic store_type choices
├── utils.py                     # EmailHelper + PrintReceiptHelper (config lookup + template rendering)
├── views.py                     # Empty (no API endpoints)
├── print_receipt.py             # PrintReceipt base class (843 lines) -- core receipt engine
├── printer_receipt_template.py  # GeneratePosReceipt class (481 lines) -- Epson POS receipt engine
├── mapping.py                   # STRATEGY_MAPPING dict: (ReceiptType, store_type) -> Strategy class
├── scripts/
│   ├── common/
│   │   ├── StrategyFactory.py   # Factory: creates retailer-specific strategy instance
│   │   └── ContextUpdater.py    # Context pattern: delegates update() to strategy
│   ├── retailers/               # 25 retailer-specific strategy classes
│   └── data/
│       └── thank_you_configs.py # Localized thank-you email configs (9 languages)
├── migrations/                  # 12 migration files (2022-2023+)
└── tests/                       # 16 test files + 80+ test data files
```

### Design Patterns

1. **Strategy Pattern** -- Each retailer can override receipt formatting by subclassing `PrintReceipt` and implementing `update(context)`. The `StrategyFactory` dispatches to the correct strategy based on `(receipt_type, store_type)`.

2. **Template Method Pattern** -- `PrintReceipt.get_receipt_and_context()` defines the receipt generation algorithm: `build_context()` -> `update_context_with_receipt_text()` -> `get_receipt_data()`. Retailer strategies override specific steps.

3. **Hierarchical Configuration** -- Both `EmailHelper` and `PrintReceiptHelper` resolve configuration by checking store+region-specific configs first, falling back to broader defaults. For email: most specific wins. For print receipts: configs are **merged** from broadest to most specific (so store-specific fields override ALL defaults).

## Data Models

### ReceiptType (Constants Class)

**File:** `mishipay_emails/models.py:5-24`

Not a Django model -- a plain class defining receipt type constants:

| Constant | Value | Used For |
|----------|-------|----------|
| `ORDER` | `"order"` | Standard purchase receipt |
| `REFUND` | `"refund"` | Refund receipt |
| `GIFT` | `"gift"` | Gift receipt (no prices, Carters/Seventh Heaven) |
| `STEB` | `"steb"` | Saudi Tax & Excise Board bag tracking (Dubai Duty Free) |
| `PAYMENT_FAILED` | `"payment_failed"` | Failed payment receipt (Dubai Duty Free) |
| `COUNTER` | `"counter"` | Counter receipt |
| `PAYMENT` | `"payment"` | Payment receipt |
| `CAMPAIGN` | `"campaign"` | Campaign/promotional receipt (DDF Visa campaign) |

### EmailTemplate

**File:** `mishipay_emails/models.py:27-34`

Stores email template HTML/text content. Templates use Django template syntax (validated on save via admin).

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | AutoField | PK |
| `name` | CharField(50) | `unique=True` |
| `content` | TextField | Django template syntax |
| `created` | DateTimeField | `auto_now_add` |
| `updated` | DateTimeField | `auto_now` |

### EmailConfiguration

**File:** `mishipay_emails/models.py:37-59`

Maps a trigger event + store + region to an email template with rendering variables.

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | AutoField | PK |
| `region` | CharField(3) | Default `'ALL'`, auto-uppercased on save |
| `store_type` | CharField(31) | `db_index=True` |
| `trigger_event` | CharField(50) | Default `'order_completed'` |
| `subject` | CharField(255) | Default `'Mishipay'`, supports Django template vars |
| `template` | FK -> EmailTemplate | `CASCADE` |
| `from_email` | CharField(255) | Supports Django template vars |
| `variables` | JSONField | Dynamic template variables |
| `enabled` | BooleanField | Default `True` |
| `extra_data` | JSONField | Flexible extra configuration |
| `created` | DateTimeField | `auto_now_add` |
| `updated` | DateTimeField | `auto_now` |

**Constraints:** `unique_together = ("region", "store_type", "trigger_event", "enabled")`

### PrintReceiptTemplate

**File:** `mishipay_emails/models.py:62-69`

Stores print receipt template content (Django template syntax with Epson printer markup).

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | AutoField | PK |
| `name` | CharField(50) | `unique=True` |
| `content` | TextField | Django template + Epson markup |
| `created` | DateTimeField | `auto_now_add` |
| `updated` | DateTimeField | `auto_now` |

### PrintReceiptConfiguration

**File:** `mishipay_emails/models.py:72-93`

Maps a store + region + receipt type to a print template and config fields.

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | AutoField | PK |
| `store_type` | CharField(31) | `db_index=True` |
| `region` | CharField(3) | Default `'ALL'`, auto-uppercased on save |
| `config_fields` | JSONField | Receipt formatting parameters (see below) |
| `receipt_type` | CharField(20) | Default `ReceiptType.ORDER` |
| `template` | FK -> PrintReceiptTemplate | `CASCADE`, `null=True, blank=True` |
| `enabled` | BooleanField | Default `True` |
| `created` | DateTimeField | `auto_now_add, null=True` |
| `updated` | DateTimeField | `auto_now` |

**Constraints:** `unique_together = ("region", "store_type", "receipt_type", "enabled")`

#### Key Config Fields (JSON)

The `config_fields` JSONField controls receipt layout. Key fields include:

| Config Key | Type | Purpose | Example |
|------------|------|---------|---------|
| `basket_item_header_name` | string | Item column header | `"Item Name"` |
| `basket_item_header_name_length` | int | Column width in chars | `21` |
| `basket_item_header_qty` | string | Qty column header | `"Qty"` |
| `basket_item_header_qty_length` | int | Qty column width | `4` |
| `basket_item_header_price` | string | Price column header | `"Price"` |
| `basket_item_header_price_length` | int | Price column width | `8` |
| `basket_item_header_total` | string | Total column header | `"Total"` |
| `basket_item_header_total_length` | int | Total column width | `10` |
| `datetime_format` | string | strftime format | `"%d/%m/%Y %H:%M"` |
| `vat_statement` | string | Tax label | `"20% VAT (included)"` |
| `order_total_header` | string | Total label | `"Total"` |
| `currency_sign` | string | Currency symbol | `"$"` |
| `show_tax_label` | bool | Show tax section | `True` |
| `show_tax_table` | bool | Show detailed tax table | `False` |
| `kiosk_pos_requires_receipt_id_qr_type` | string | QR code content type | `"order_id"`, `"order_receipt_url"`, `"zatka_qr"` |
| `kiosk_pos_show_original_price_in_item_list` | bool | Show pre-discount price | `False` |
| `kiosk_pos_print_item_code_with_qty_price` | bool | Print product code | `True` |
| `kiosk_pos_print_promo_after_every_item` | bool | Inline promo text per item | `False` |
| `receipt_uses_pos_txn_number` | bool | Use POS transaction number | `False` |
| `exclude_vat_from_total` | bool | Subtract VAT from subtotal | `True` |
| `include_basket_threshold_discount_in_total` | bool | Add basket promo to subtotal | `True` |
| `show_subtotal_without_discount` | bool | Show original subtotal | `False` |
| `show_item_level_discount` | bool | Show per-item discount | `False` |
| `tax_registration_header` | string | Tax number label | `"Tax Reg. No"` |
| `retailer_t_and_c` | string | Footer terms/conditions | (long text) |
| `store_footer_note` | string | Receipt footer text | `"www.mishipay.com"` |

## Email System

### EmailHelper

**File:** `mishipay_emails/utils.py:6-65`

Static utility class for the email pipeline.

**`get_store_region_config(store_type, region, trigger_event)`** -- Resolves the best-matching email configuration. Lookup priority (first match wins):
1. `{store_type}_{region}` (most specific)
2. `{store_type}_ALL`
3. `ALL_{region}`
4. `ALL_ALL` (least specific)

**`get_template(template_id)`** -- Fetches template content from `EmailTemplate` by ID.

**`get_email_data(template_content, variables, subject, from_email)`** -- Renders the template, subject, and from_email using Django's `Template` + `Context`. Returns `{"html", "subject", "from_email", "variables"}`.

### Thank-You Email Configs

**File:** `mishipay_emails/scripts/data/thank_you_configs.py`

Provides localized thank-you email configurations in **9 languages**: English (ALL), Spanish (ES), French (FR), German (CH), Danish (DK), Finnish (FI), Italian (IT), Swedish (SE), Norwegian (NO). Each config includes:
- `subject`: Localized email subject (uses `{{ retailer }}` template var)
- `variables`: `hi`, `congratulate_msg`, `team_msg`, `feedback` (with survey link), `thank_you_msg`, `regards`, `position`, `shop_smart`, `visit`

## Print Receipt System

### PrintReceipt Base Class

**File:** `mishipay_emails/print_receipt.py:30-842`

The core receipt generation engine (843 lines). Handles order data extraction, context building, and receipt text formatting.

#### Initialization

```python
class PrintReceipt:
    def __init__(self, store, **kwargs):
        # kwargs: config_fields, order_id, template
        # Loads Order, Basket, Customer, Store, Payments, BasketEntityAuditLogs
```

Fetches the full order graph via `Order.objects.select_related("basket", "basket__customer", "basket__store")`.

#### Receipt Generation Pipeline

```
get_receipt_and_context()
  ├── build_context(config_fields)
  │     ├── default_context()      # Sets 60+ context variables from order/store/payment data
  │     └── ContextUpdater.update_object()  # Delegates to retailer strategy's update() method
  ├── update_context_with_receipt_text()
  │     ├── update_items_text_for_receipt()              # Format item lines
  │     ├── update_total_quantity_label_val()             # Format quantity summary
  │     ├── update_basket_promos_title_and_discount_val() # Format basket-level promos
  │     ├── update_total_vat_and_order_total_val()        # Format VAT + total
  │     ├── update_total_discount_val()                   # Format total discount
  │     ├── update_tax_table_val()                        # Format tax table (if shown)
  │     └── update_order_payment_info_val()               # Format payment details
  └── get_receipt_data(context)    # Render template with context
```

#### Key Methods

| Method | Lines | Purpose |
|--------|-------|---------|
| `default_context()` | 97-214 | Populates 60+ context variables: store info, order ID, dates, items, totals, promotions, payment details, QR values, staff/device details |
| `get_items_list()` | 298-470 | Extracts basket items with pricing, discounts, addons, STEB details, tax amounts. Two code paths: with audit logs (promo-aware) and without (simple fallback) |
| `get_steb_list_data()` | 472-507 | Extracts STEB bag barcode data for Dubai Duty Free |
| `get_refund_items_data()` | 509-529 | Aggregates refund quantities and amounts per item |
| `get_order_summary_data()` | 531-560 | Calculates subtotal, VAT, total, basket discount. Subtracts refunded amounts. Applies config flags for VAT/discount inclusion |
| `update_items_text_for_receipt()` | 623-684 | Formats item lines with 3 display modes: translation mode (with `<img>` tags), code-first mode (product ID + name), and default mode (retailer product ID + name). Handles addons, per-item promos |
| `get_zakat_qr_value()` | 823-841 | Generates Saudi Zatka QR code using TLV encoding (seller name, VAT number, timestamp, totals) encoded as Base64. Uses `uttlv` library |
| `get_purchase_date()` | 241-270 | Resolves purchase date with timezone conversion. Special handling for Click & Collect orders (uses CC status update date) |
| `format_width_of_str()` | 586-594 | Pads/trims strings to fixed width. Supports newline wrapping for long strings |
| `get_boarding_pass_details()` | 790-802 | Retrieves boarding pass data for DDF (airline, flight, passenger name) via `RetailerSpecificCustomerProfile` |

### GeneratePosReceipt Class

**File:** `mishipay_emails/printer_receipt_template.py:23-481`

An independent Epson POS receipt generator (481 lines) that produces receipts section-by-section using Epson markup. Uses the same `PrintReceiptConfiguration` model but assembles receipt strings directly rather than via Django template rendering.

#### Key Differences from PrintReceipt

| Aspect | PrintReceipt | GeneratePosReceipt |
|--------|-------------|-------------------|
| Output | Django template rendered text | Directly concatenated Epson markup |
| Strategy | Uses Strategy pattern (retailer subclasses) | Monolithic class, no strategy dispatch |
| Refund handling | Via strategy subclasses | Built-in refund subtraction from items |
| Promo display | Config-driven (show_item_level_discount) | Always shows item-level and basket-level |
| Cash handling | Not built-in | Built-in cash/card/change display |

#### Key Methods

| Method | Lines | Purpose |
|--------|-------|---------|
| `print_epson_order_receipt()` | 459-470 | Assembles full order receipt: store header + order header + membership + items table + summary + footer |
| `print_epson_refund_order_receipt()` | 472-480 | Assembles refund receipt with refund items and refund summary |
| `epson_printer_store_header()` | 237-244 | Store logo, name, address, tax reg number |
| `epson_printer_order_header()` | 246-272 | QR code (configurable: order ID / URL / none), receipt number, date |
| `epson_printer_membership_header()` | 274-280 | Loyalty number display |
| `epson_printer_basket_item_contents()` | 307-364 | Item rows with discount display (show_subtotal_without_discount mode shows original + discount line) |
| `epson_printer_order_summary()` | 366-413 | Subtotal, savings, basket promos, VAT, total. Separate cash/card payment sections with change calculation |
| `epson_printer_receipt_footer()` | 447-457 | T&C, transaction ID barcode, footer note |

## Strategy System

### StrategyFactory

**File:** `mishipay_emails/scripts/common/StrategyFactory.py`

```python
class StrategyFactory:
    @staticmethod
    def create_strategy(store, region, receipt_type, **kwargs):
        store_type = store.store_type.lower()
        strategy_class = STRATEGY_MAPPING[receipt_type].get(store_type, PrintReceipt)
        return strategy_class(store, **kwargs)
```

Falls back to the base `PrintReceipt` class if no retailer-specific strategy exists.

### ContextUpdater

**File:** `mishipay_emails/scripts/common/ContextUpdater.py`

Minimal context pattern wrapper -- calls `strategy.update(object_to_update)`.

### Strategy Mapping

**File:** `mishipay_emails/mapping.py`

Maps `(ReceiptType, store_type_lowercase)` to strategy classes:

| Receipt Type | Retailers with Custom Strategies |
|-------------|----------------------------------|
| **ORDER** (18) | Flying Tiger Azadea, MLSE, Flying Tiger, Dubai Duty Free, Grandiose, MMI, KikiMart, Dar Al Amirat, KCH, Carters, Florist, Gillan, Riyadh Tameer, Seventh Heaven, FlashFacial, Hudson, WHSmith, Vengo |
| **REFUND** (6) | Flying Tiger, MLSE, Walmart, Gillan, Seventh Heaven, Vengo |
| **STEB** (1) | Dubai Duty Free |
| **PAYMENT_FAILED** (1) | Dubai Duty Free |
| **GIFT** (2) | Carters, Seventh Heaven |

**`ADDITIONAL_RECEIPTS`** -- `[ReceiptType.GIFT]` -- receipt types that generate extra receipts alongside the main order receipt.

### Retailer Strategy Details

All 25 strategy files are in `mishipay_emails/scripts/retailers/`. Each subclasses `PrintReceipt` and overrides `update(context)` to add retailer-specific context variables. Some also override receipt text methods.

#### Dubai Duty Free (most complex)

**Files:** `dubai_duty_free_order_strategy.py` (270 lines), `dubai_duty_free_steb_strategy.py` (120 lines), `dubai_duty_free_payment_failed_strategy.py` (42 lines)

- **Order strategy**: Adds boarding pass data (passenger name, airline, flight, departure/destination), extensive payment terminal details (NI EPOS framework, card read method, auth code, merchant ID, terminal ID, TVR, TSI, AC, CID, application version), Dynamic Currency Conversion (DCC) handling with exchange rate and margin display (Visa/Mastercard specific), STEB bag barcodes per item, per-item VAT breakdown with net-before-VAT calculation, custom footer QR with date-device-txn format.
- **STEB strategy**: Generates one receipt per distinct STEB bag barcode. Groups items by bag, displays item code/description/quantity. Reads `DDF_COUPON_RECEIPT` from `GlobalConfig` for gold promotion coupons.
- **Payment failed strategy**: Extends `DubaiDutyFreeOrderStrategy`, overrides context with basket amounts (not captured amounts since payment failed), adds `ms_pay_payment_session_id` for debugging. Also produces a campaign receipt for Visa cardholders.

#### Flying Tiger Commerce

**Files:** `flying_tiger_order_strategy.py` (154 lines), `flying_tiger_refund_strategy.py`, `flying_tiger_azadea_order_strategy.py`

- Handles offline transactions with `offline_receipt_reference_id` fallback
- Overrides `get_items_list()` to exclude refunded items and subtract refund quantities
- Custom item formatting using `remove_special_characters()` (strips non-ASCII)
- Separate cash/card payment display with change calculation
- Uses `format_amount_with_sign()` for consistent amount formatting

#### MLSE (Toronto Sports)

**File:** `mlse_order_strategy.py` (209 lines), `mlse_refund_strategy.py`

- Multiple payment methods: Bucks (loyalty currency), Budget Code (corporate accounts with staff name), Card Terminal (with split payments)
- Split payment handling: iterates `split_payments` array, creating separate Bucks and Card payment lines
- First Nations tax code support (`tax_override_details` with `type: "first_nations"`)
- Tax exemption display
- ADDON item ordering: uses `get_ordered_item_list()` to group parent items with their addon sub-items (using `customized_item` references in extra_data)
- Hardcoded retailer terms & conditions with store-specific `receipt_store_name`

#### Carters

**File:** `carters_order_strategy.py` (209 lines)

- Integrates with `CartersPostTransaction` loyalty client for customer data (first/last name, account number, estimated points, reward redemption)
- POS till details via `get_till_details()` utility
- Address JSON parsing for structured address display
- Custom barcode string generation: `{barcode_id}00{store_id}{register}{txn_id}{date}`
- Per-item promo details with `item_print_each_promo_separately` flag
- Currency symbol injection via `add_currency_symbol_to_amount()`

#### Other Notable Strategies

| Strategy | Key Feature |
|----------|-------------|
| **Grandiose** | UAE grocery -- custom item formatting |
| **MMI** | UAE alcohol retailer -- custom order formatting |
| **KikiMart** | Custom receipt layout |
| **Gillan** | Order + refund strategies |
| **Seventh Heaven** | Order + refund + gift receipt support |
| **Walmart** | Refund-only strategy |
| **WHSmith** | UK retailer -- custom formatting |
| **Hudson** | Travel retail -- custom formatting |
| **Vengo** | Reused for both order and refund receipt types |

## Admin Interface

**File:** `mishipay_emails/admin.py`

4 admin registrations with custom widgets:

| Admin Class | Model | Custom Features |
|------------|-------|-----------------|
| `EmailTemplateAdmin` | EmailTemplate | `PrettyTextWidget` (50-row textarea), Django template syntax validation on save |
| `EmailConfigurationAdmin` | EmailConfiguration | `PrettyJSONWidget` for `variables`/`extra_data`, dynamic `store_type` choices from `STORE_TYPE_MAP` |
| `PrintReceiptTemplateAdmin` | PrintReceiptTemplate | `PrettyTextWidget`, template syntax validation on save |
| `PrintReceiptConfigurationAdmin` | PrintReceiptConfiguration | `PrettyJSONWidget` for `config_fields`, dynamic `store_type` + `ReceiptType` choices |

### Custom Widgets

- **`PrettyJSONWidget`** (`admin.py:15-22`): Formats JSON with `indent=4`, auto-sizes textarea rows/cols to fit content (min 10/40, max 30/120).
- **`PrettyTextWidget`** (`admin.py:25-30`): Fixed 50-row, 120-column textarea for template content editing.

### Forms

**File:** `mishipay_emails/forms.py`

- **`EmailConfigurationForm`**: Dynamic `store_type` choices populated from `dos.store_map.STORE_TYPE_MAP` plus `'ALL'`.
- **`PrintReceiptConfigurationForm`**: Same dynamic `store_type` choices plus `ReceiptType.CHOICES` for receipt type selection.

## Dependencies

### Internal Dependencies

| Dependency | Used For |
|-----------|----------|
| `mishipay_retail_orders.models.Order` | Order data, payments, completion date |
| `mishipay_retail_orders.models.BasketItem` | Basket item data (implicit via basket) |
| `mishipay_retail_customers.models.RetailerSpecificCustomerProfile` | Loyalty numbers, boarding pass data |
| `mishipay_retail_customers.common_functions.BoardingPass` | Boarding pass parsing (DDF) |
| `mishipay_dashboard.models.RefundOrderNew` | Refund data for receipt adjustments |
| `mishipay_core.common_functions` | `get_rounded_value`, `get_show_tax_label`, `get_receipt_url`, `get_promo_title_and_value_for_basket_promos`, `get_all_item_applied_promos`, `get_kiosk_pos_tax_table`, `get_kiosk_receipt_store_logo`, `get_value_from_payment_response` |
| `mishipay_core.app_settings` | `EMAIL_STORE_LOGO_URL` mapping |
| `mishipay_special_promos.special_promo_helper` | `get_basket_level_promotions_label_details()` |
| `mishipay_config.models.GlobalConfig` | `DDF_COUPON_RECEIPT` config for STEB |
| `mishipay_items.constants` | `WITH_ADDONS`, `ADDON` item type constants |
| `mishipay_items.ddf_utility.utils` | `get_card_brand_from_card_number()` (DDF) |
| `mishipay_items.carters_utility.client` | `CartersPostTransaction`, `get_till_details()` |
| `dos.store_map.STORE_TYPE_MAP` | Store type registry for admin forms |
| `dos.models.choices.PURCHASE_TYPES` | `CLICK_AND_COLLECT` purchase type |

### External Libraries

| Library | Used For |
|---------|----------|
| `uttlv` | Zatka (Saudi Arabia) QR code TLV encoding |
| `pytz` | Timezone conversion for receipt dates |
| `dateutil.parser` | Date string parsing for Click & Collect orders |

## Configuration

### Email Configuration Hierarchy

Resolution order (first match wins):
1. `store_type={specific} AND region={specific}` -- store+region exact match
2. `store_type={specific} AND region=ALL` -- store-wide default
3. `store_type=ALL AND region={specific}` -- region-wide default
4. `store_type=ALL AND region=ALL` -- global default

### Print Receipt Configuration Hierarchy

Resolution order (**merging**, not first-match -- broader configs are applied first, narrower ones override):
1. Start with `ALL_ALL` config
2. Merge `ALL_{region}` fields on top
3. Merge `{store_type}_ALL` fields on top
4. Merge `{store_type}_{region}` fields on top (most specific wins)

This means store-specific config only needs to specify **override** fields; all other fields inherit from the ALL_ALL defaults.

### Thank-You Email Languages

9 localized configurations in `scripts/data/thank_you_configs.py`:

| Region Code | Language |
|-------------|----------|
| ALL | English |
| ES | Spanish |
| FR | French |
| CH | German |
| DK | Danish |
| FI | Finnish |
| IT | Italian |
| SE | Swedish |
| NO | Norwegian |

## Schema Evolution

12 migrations from initial creation through 2023+:

| Migration | Key Change |
|-----------|-----------|
| `0001_initial` | Creates `EmailTemplate`, `EmailConfiguration` |
| `0002-0004` | Schema adjustments to email models |
| `0005_printreceiptconfiguration` | Adds `PrintReceiptConfiguration` model |
| `0005_alter_emailconfiguration_extra_data_and_more` | Parallel branch: email config field changes |
| `0006_merge_20230512_1002` | Merges two divergent migration branches |
| `0007_alter_printreceiptconfiguration_config_fields` | Config fields changes |
| `0008_printreceipttemplate_and_more` | Adds `PrintReceiptTemplate` model (previously template was inline in config) |
| `0009-0010_alter_receipt_type` | Receipt type field changes (adding new types) |
| `0011_alter_printreceiptconfiguration_template` | Makes template FK nullable (`null=True, blank=True`) |

## Test Coverage

**16 test files**, organized by retailer, with 80+ test data files:

| Test File | Retailer | Focus |
|-----------|----------|-------|
| `test_flying_tiger_receipt.py` | Flying Tiger Commerce | GB, DK, UAE variants |
| `test_flying_tiger_azadea_receipt.py` | Flying Tiger Azadea | UAE variant |
| `test_mlse_receipt.py` | MLSE | Bucks, card, budget code, split payments, First Nations |
| `test_dubai_duty_free_receipt.py` | Dubai Duty Free | DCC, promotions, STEB, payment failed |
| `test_grandiose_receipt.py` | Grandiose | Order receipts |
| `test_carters_receipt.py` | Carters | Loyalty, barcode generation |
| `test_seventh_heaven_receipt.py` | Seventh Heaven | Order, refund, gift |
| `test_walmart_receipt.py` | Walmart | Refund receipts |
| `test_gillan_receipt.py` | Gillan | Order + refund |
| `test_mmi_receipt.py` | MMI | Order receipts |
| `test_kikimart_receipt.py` | KikiMart | Order receipts |
| `test_hudson_receipt.py` | Hudson | Order receipts |
| `test_vengo_receipt.py` | Vengo | Order receipts |
| `test_florist_receipt.py` | Florist | Order receipts |
| `test_dar_al_amirat_receipt.py` | Dar Al Amirat | Order receipts |
| `test_riyadhtameer_receipt.py` | Riyadh Tameer | Order receipts |

### Test Infrastructure

**File:** `mishipay_emails/tests/common.py`

- `get_expected_receipt(receipt_path)` -- Loads expected receipt output from file for comparison
- `create_default_template()` -- Creates a standard receipt template in DB
- `create_default_config(default_template)` -- Creates ALL_ALL order + refund configs with full default config_fields
- `create_default_steb_template()` / `create_default_steb_config()` -- STEB receipt test fixtures
- `create_default_payment_failed_config()` -- Payment failed receipt test fixtures
- `create_default_gift_template()` / `create_default_gift_config()` -- Gift receipt test fixtures

Tests use snapshot-style testing: generate a receipt, compare with expected output from test data files.

## Notable Patterns

1. **No REST API** -- The module has an empty `views.py`. All email/receipt configuration is managed through Django admin. Receipt generation is called from other modules (likely order completion flows).

2. **No Background Tasks** -- No `tasks.py` or `signals.py`. Receipt generation is synchronous. For high-volume stores, this could impact response times.

3. **Dual Receipt Engines** -- Both `PrintReceipt` (strategy-based, template-rendered) and `GeneratePosReceipt` (monolithic, direct string assembly) exist. `GeneratePosReceipt` appears to be a newer, simpler implementation that doesn't use the strategy pattern.

4. **Typo in Class Name** -- `CartersOrderStratergy` (`carters_order_strategy.py:14`) has a typo ("Stratergy" instead of "Strategy"). Similarly `seventh_heaven_order_stratergy.py`. These are consistent with each other but differ from the standard naming.

5. **`is not ""` Comparisons** -- Multiple occurrences of `if receipt_str is not ""` (`print_receipt.py:683,706,724,770`; strategy files). This uses identity comparison instead of `!=` or truthiness check. In Python 3, this works due to string interning but is not guaranteed and triggers linter warnings.

6. **Hardcoded Default Logo** -- `print_receipt.py:102` hardcodes a default logo URL to `pngtree.com` -- likely a placeholder that should be configurable.

7. **Epson Markup in Business Logic** -- Receipt formatting strings (e.g., `[L]`, `[R]`, `[C]`, `<b>`, `<img>`) are embedded throughout Python code rather than in templates, coupling the formatting to the business logic.

8. **DDF Complexity** -- Dubai Duty Free strategies are significantly more complex than others due to: boarding pass integration, Dynamic Currency Conversion (DCC), STEB bag tracking, NI EPOS payment terminal details, Visa campaign receipts, and multi-currency display.

## Open Questions (Emails)

- Are emails sent through this module's `EmailHelper` pipeline, or is there a separate email sending service? The module defines configuration and templates but no actual sending logic (no SMTP/API calls visible).
- How is `GeneratePosReceipt` (`printer_receipt_template.py`) invoked? It doesn't integrate with the `StrategyFactory` or `mapping.py`. Is it used by a different code path (e.g., kiosk hardware integration)?
- The `COUNTER` and `PAYMENT` receipt types are defined in `ReceiptType` but have no entries in `STRATEGY_MAPPING` and no test coverage. Are they used?
- The `CAMPAIGN` receipt type is only used by `DubaiDutyFreeOrderStrategy.get_additional_receipts()` for Visa cardholders. Is this still active?
- Why does `PrintReceiptHelper.get_store_region_config()` merge configs from broadest to most specific (opposite of `EmailHelper`'s first-match approach)? Is this intentional to allow partial overrides?
- Test data includes `tests/data/flying_tiger/old_code/` -- is this legacy test data that should be cleaned up?
- Several retailers (e.g., Dar Al Amirat, FlashFacial, Florist, KCH, Riyadh Tameer) have strategy classes but no visible REFUND strategy. Are refund receipts for these retailers handled by the base `PrintReceipt` class?

---

# Payment Alerts & Settlement Reconciliation

> Added by: Task T14 — MishiPay Platform Alerts & Subscriptions

## Overview

The `mishipay_alerts` module (58 files, ~5,600 lines) implements a **payment alerting and settlement reconciliation pipeline**. It detects discrepancies between MishiPay order records and Payment Service Provider (PSP) records, tracks retailer import/export jobs, and produces settlement reports for financial reconciliation.

The vast majority of business logic (4,262+ lines) lives in **management commands** executed as scheduled cron jobs, not REST views. The module's 4 REST endpoints are thin wrappers for job tracking and payout reference updates.

## Key Concepts

| Term | Meaning |
|------|---------|
| **PSP** | Payment Service Provider — Stripe, QuickPay, Checkout.com, Adyen, etc. |
| **Settlement** | The process of reconciling MishiPay order amounts with PSP-charged amounts |
| **Payout** | A PSP transfer of collected funds to the merchant's bank account |
| **Reconciliation** | Three-way comparison: MishiPay order amounts vs POS log amounts vs PSP charged amounts |
| **Business Date** | The trading day for settlement purposes (may differ from calendar date for some retailers) |
| **MS Pay** | MishiPay's payments microservice — the modern abstraction layer over PSP APIs |
| **POSLOG** | Point-of-Sale log — the retailer's record of transactions |

## Architecture

```
mishipay_alerts/
├── models.py                             # 10 models (1 abstract): alerts, jobs, settlement records
├── views.py                              # 4 thin REST endpoints (auth commented out)
├── urls.py                               # URL routing
├── admin.py                              # 9 admin registrations
├── alert_util.py                         # Slack alert formatting
├── date_util.py                          # Date range utilities
├── inventory_promo_poslog_alert_helper.py # Import/export job tracking helpers
├── helpers/
│   └── psp_recon_payouts_helper.py       # Dashboard reconciliation helpers (1,379 lines)
├── management/commands/
│   ├── stripe_connect_payment_alert.py          # Stripe payment discrepancy detection (330 lines)
│   ├── quickpay_payment_alert.py                # QuickPay/MobilePay/Vipps/Swish alerts (353 lines)
│   ├── stripe_connect_psp_settlement_payout.py  # Stripe settlement & payouts (862 lines)
│   ├── ms_pay_based_settlement_payout.py        # Universal MS Pay settlement (599 lines)
│   ├── quickpay_psp_settlement_by_files.py      # Nordic file-based settlement (895 lines)
│   ├── mishipay_settlement_report_final.py      # Final aggregated reports (627 lines)
│   ├── checkout_psp_settlement_transactions.py  # Checkout.com settlement (319 lines)
│   ├── create_psp_reconciliation_summary.py     # CSV reconciliation reports (300 lines)
│   └── quickpay_psp_settlement_deprecated.py    # Deprecated QuickPay settlement (297 lines)
├── tests.py                              # 937 lines of tests
└── migrations/                           # Schema migrations
```

### Design Patterns

1. **Cron-driven pipeline** — Settlement processing runs as scheduled Django management commands, not real-time API handlers. Commands are idempotent: they delete existing records for a date range, then `bulk_create` new ones inside `transaction.atomic()`.

2. **Three-way reconciliation** — The system compares amounts from three sources: MishiPay order totals (`mishipay_*_amount`), POS log totals (`poslog_*_amount`), and PSP charged amounts (`psp_*_amount`).

3. **No-FK design for partitioning** — `SettlementOrder.order_id` is a `UUIDField`, not a `ForeignKey`. The code comments explain: "Foreign keys are not used to support partitioning in future."

4. **PSP API evolution** — The codebase shows a clear migration from direct PSP API calls (Stripe/QuickPay) to the MS Pay microservice abstraction layer.

## Data Models

### PaymentAlert

**File:** `mishipay_alerts/models.py:36`

Tracks payment discrepancies between MishiPay orders and PSP records.

| Field | Type | Constraints |
|-------|------|-------------|
| `alert_id` | UUIDField | PK, default=uuid4 |
| `alert_type` | CharField(255) | See alert types below |
| `store_id` | UUIDField | No FK (design choice) |
| `store_type` | CharField(63) | |
| `order_id` | UUIDField | |
| `psp_type` | CharField(63) | PSPType choices |
| `psp_reference_id` | CharField(63) | |
| `payment_method` | CharField(63) | |
| `order_amount` | DecimalField(15,2) | MishiPay amount |
| `psp_amount` | DecimalField(15,2) | PSP amount |
| `status` | CharField(31) | STARTED/CREATED/RESOLVED/CANCELLED/RESOLVED_BY_ALERT/FAILED/SUCCESS/WARNING |
| `extra_data` | JSONField | Additional metadata |
| `created` | DateTimeField | auto_now_add |
| `updated` | DateTimeField | auto_now |

#### Alert Types

| Constant | Description | Auto-Remediation |
|----------|-------------|-----------------|
| `ORDER_SUCCESS_PSP_SUCCESS_MULTIPLE_CHARGES` | Order OK but PSP shows multiple charges | Yes — auto-refunds duplicate via `stripe.Refund.create()` |
| `ORDER_SUCCESS_PSP_NOT_SUCCESS` | Order OK in MishiPay, PSP shows failure | No — manual investigation |
| `ORDER_NOT_SUCCESS_PSP_SUCCESS` | Order failed in MishiPay, PSP charged | Yes — auto-updates order status via `update_order_payment_info_manual()` |
| `ORDER_NOT_SUCCESS_PSP_SUCCESS_MULTIPLE_CHARGES` | Order failed, multiple PSP charges | Manual — refund required |
| `PSP_AUTHORIZED_BUT_NOT_CAPTURED` | Payment authorized but never captured | Manual — capture or void (QuickPay only) |

#### Alert Type to Action Mapping (`ALERT_TYPE_TO_ACTIONS`)

```
ORDER_SUCCESS_PSP_SUCCESS_MULTIPLE_CHARGES → "Refund the extra charge"
ORDER_SUCCESS_PSP_NOT_SUCCESS              → "Could be timing issue. Investigate"
ORDER_NOT_SUCCESS_PSP_SUCCESS              → "Charge customer and Update Order"
ORDER_NOT_SUCCESS_PSP_SUCCESS_MULTIPLE_CHARGES → "Refund the extra charge and update order"
PSP_AUTHORIZED_BUT_NOT_CAPTURED            → "Bug in App, you may need to capture manually"
```

### RetailerCronJobBase (Abstract)

**File:** `mishipay_alerts/models.py:74`

Abstract base class for retailer import/export job tracking.

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | UUIDField | PK, default=uuid4 |
| `status` | CharField(31) | STARTED/SUCCESS/WARNING/FAILED |
| `business_date` | DateField | |
| `retailer` | CharField(31) | |
| `region` | CharField(31) | Default 'ALL' |
| `store_chain` | CharField(63) | Default '' |
| `extra_data` | JSONField | |
| `created` | DateTimeField | auto_now_add |
| `updated` | DateTimeField | auto_now |

### RetailerImportJob / RetailerExportJob

**File:** `mishipay_alerts/models.py:107`, `models.py:126`

Concrete subclasses of `RetailerCronJobBase`. Track inventory/stock/promo imports and poslog/reconciliation exports.

| Additional Field | Model | Type |
|-----------------|-------|------|
| `import_type` | RetailerImportJob | CharField(31) |
| `export_type` | RetailerExportJob | CharField(31) |
| `from_datetime` | RetailerExportJob | DateTimeField (null) |
| `to_datetime` | RetailerExportJob | DateTimeField (null) |

### SettlementOrder

**File:** `mishipay_alerts/models.py:136`

Core settlement record linking MishiPay orders to PSP charges.

| Field | Type | Constraints |
|-------|------|-------------|
| `store_id` | UUIDField | No FK |
| `order_id` | UUIDField | Nullable — "some orders don't exist in our system" |
| `order_date` | DateTimeField | |
| `business_date` | DateField | |
| `store_type` | CharField(63) | |
| `store_chain` | CharField(63) | |
| `region` | CharField(31) | |
| `psp_type` | CharField(63) | PSPType choices |
| `psp_reference_id` | CharField(63) | |
| `psp_charged_amount` | DecimalField(15,2) | |
| `psp_fee_amount` | DecimalField(15,**4**) | 4 decimal places for fee precision |
| `psp_net_amount` | DecimalField(15,2) | Charged minus fee |
| `poslog_excluded_discount` | DecimalField(15,2) | MishiPay-sponsored discount |
| `poslog_sent` | BooleanField | Default False |
| `status` | CharField(31) | |
| `manual_action_reason` | TextField | |
| `payout_reference_id` | CharField(63) | |
| `payout_available_on` | DateField | Null |
| `payout_actual_date` | DateField | Null |
| `extra_data` | JSONField | Card brand, auth code, cashier code, platform, etc. |
| `created` | DateTimeField | auto_now_add |
| `updated` | DateTimeField | auto_now |

**Constraint:** `unique_together = ('order_id', 'psp_reference_id')`

### SettlementOrderRefund

**File:** `mishipay_alerts/models.py:171`

Settlement record for refund transactions. Similar structure to `SettlementOrder` with additional refund-specific fields.

| Additional Field | Type | Constraints |
|-----------------|------|-------------|
| `refund_order_id` | UUIDField | |
| `psp_refund_amount` | DecimalField(15,2) | |
| `psp_refund_fee_amount` | DecimalField(15,4) | |
| `psp_refund_net_amount` | DecimalField(15,2) | |

**Constraint:** `unique_together = ('order_id', 'refund_order_id', 'psp_reference_id')`

### SettlementPayoutByDate

**File:** `mishipay_alerts/models.py:213`

PSP payout records grouped by date.

| Field | Type | Constraints |
|-------|------|-------------|
| `store_type` | CharField(63) | |
| `store_chain` | CharField(63) | |
| `region` | CharField(31) | |
| `psp_type` | CharField(63) | |
| `payout_reference_id` | CharField(63) | |
| `payout_amount` | DecimalField(15,2) | |
| `payout_available_on` | DateField | |
| `payout_actual_date` | DateField | |
| `extra_data` | JSONField | |

**Constraint:** `unique_together = ('store_type', 'psp_type', 'payout_reference_id')`

### SettlementReconByDate

**File:** `mishipay_alerts/models.py:236`

Per-store, per-date, per-PSP aggregated reconciliation data.

| Field | Type | Purpose |
|-------|------|---------|
| `business_date` | DateField | |
| `store_id` | UUIDField | |
| `psp_type` | CharField(63) | |
| `psp_order_amount` | DecimalField(15,2) | Sum of PSP charges |
| `psp_order_fee_amount` | DecimalField(15,4) | Sum of PSP fees |
| `psp_order_count` | IntegerField | Number of orders |
| `psp_refund_amount` | DecimalField(15,2) | Sum of PSP refunds |
| `psp_refund_fee_amount` | DecimalField(15,4) | |
| `psp_refund_count` | IntegerField | |
| `poslog_not_sent_order_count` | IntegerField | Orders not sent to POS log |
| `poslog_not_sent_refund_count` | IntegerField | Refunds not sent to POS log |

### SettlementFinalSummary

**File:** `mishipay_alerts/models.py:272`

Final summary with net amounts, fees, and payout comparison by business date.

| Field Group | Fields | Purpose |
|------------|--------|---------|
| **MishiPay** | `mishipay_order_amount`, `mishipay_refund_amount`, `mishipay_net_amount` | What MishiPay recorded |
| **POSLOG** | `poslog_order_amount`, `poslog_refund_amount`, `poslog_net_amount` | What POS log recorded |
| **PSP** | `psp_order_amount`, `psp_order_fee_amount`, `psp_refund_amount`, `psp_refund_fee_amount`, `psp_net_amount_without_fee`, `psp_net_fee_amount`, `psp_net_amount` | What PSP recorded |
| **Payout** | `payout_amount`, `payout_amount_cummulative` | What was paid out (note: "cummulative" is misspelled) |
| **Counts** | `order_count`, `refund_count`, `poslog_not_sent_order_count`, `poslog_not_sent_refund_count` | Transaction counts |

### SettlementFinalSummaryByPayoutDates

**File:** `mishipay_alerts/models.py:316`

Same structure as `SettlementFinalSummary` but grouped by `payout_actual_date` instead of `business_date`. Used for payout-centric reconciliation views.

## API Endpoints

**File:** `mishipay_alerts/urls.py:5-12`, `mishipay_alerts/views.py`

| Method | Path | View | Purpose |
|--------|------|------|---------|
| POST | `v1/retailer_import_job/` | `RetailerImportJob` | Start a retailer import job (creates record with STARTED status) |
| PATCH | `v1/retailer_import_job/` | `RetailerImportJob` | End an import job (updates status to SUCCESS/WARNING/FAILED) |
| POST | `v1/retailer_export_job/` | `RetailerExportJob` | Start a retailer export job |
| PATCH | `v1/retailer_export_job/` | `RetailerExportJob` | End an export job |
| PATCH | `v1/stripe_connect_payout_reference_update/` | `StripeConnectPayout` | Update payout reference IDs in settlement records |
| PATCH | `v1/stripe_connect_payout_reference_update_by_ms_pay/` | `MsPayConnectPayout` | Update payout references via MS Pay service |

**Security concern:** Authentication is **commented out** on all views (`views.py:2,15,55`). The `IsAuthenticated` import and `permission_classes` decorators are present but commented.

### Import/Export Job API Flow

```
External Service → POST v1/retailer_import_job/ {store_id, import_type, business_date}
                 → Creates RetailerImportJob with status=STARTED
                 → ... job runs ...
                 → PATCH v1/retailer_import_job/ {job_id, status, extra_data}
                 → Updates job status to SUCCESS/WARNING/FAILED
```

The helper functions (`inventory_promo_poslog_alert_helper.py`) resolve Store objects by either `store_id` (UUID) or `retailer_store_id` + `store_chain`.

## Business Logic — Management Commands

### Payment Alert Pipeline

Two management commands detect payment discrepancies:

#### Stripe Payment Alert

**File:** `mishipay_alerts/management/commands/stripe_connect_payment_alert.py` (330 lines)

**Algorithm:**
1. For each store with Stripe payment variants, calls `stripe.PaymentIntent.list()` with date range filter.
2. For each PaymentIntent, finds matching MishiPay order by `psp_reference_id`.
3. Compares order status with PaymentIntent status.
4. Creates `PaymentAlert` records for discrepancies.
5. **Auto-remediation:**
   - Duplicate charges → automatically creates `stripe.Refund.create()` for the extra charge, sets alert to `RESOLVED_BY_ALERT`.
   - PSP success but order failed → auto-updates order via `update_order_payment_info_manual()`, sets to `RESOLVED_BY_ALERT`.
6. Sends Slack notifications to `#alerts_payments` (prod) or `#alerts_test` (non-prod).

#### QuickPay Payment Alert

**File:** `mishipay_alerts/management/commands/quickpay_payment_alert.py` (353 lines)

Handles MobilePay (DK), Vipps (NO), and Swish (SE) payment methods. Adds `PSP_AUTHORIZED_BUT_NOT_CAPTURED` alert type (unique to QuickPay). Uses priority separation:
- High-priority alerts → `#alerts_payments`
- Low-priority (authorized-not-captured) → `#alerts_payments_low`

### Settlement Processing Pipeline

Settlement commands run in stages:

```
Stage 1: Collect settlement data from PSP
  ├── ms_pay_based_settlement_payout.py     (modern — via MS Pay API, supports 18+ PSPs)
  ├── stripe_connect_psp_settlement_payout.py (transitional — direct Stripe + MS Pay)
  ├── quickpay_psp_settlement_by_files.py   (file-based — MobilePay CSV, Swish XML, Vipps XML)
  └── checkout_psp_settlement_transactions.py (Checkout.com via MS Pay, Virgin stores only)

Stage 2: Aggregate and report
  ├── mishipay_settlement_report_final.py   (3-way recon → SettlementReconByDate + FinalSummary)
  └── create_psp_reconciliation_summary.py  (CSV reports for retailers)
```

#### MS Pay Settlement (Modern — Universal)

**File:** `mishipay_alerts/management/commands/ms_pay_based_settlement_payout.py` (599 lines)

The most modern settlement command. Uses the MS Pay microservice to abstract PSP-specific details. Supports 18 payment gateway variants mapped to internal PSP types:

| MS Pay Variant | Internal PSP Type |
|---------------|-------------------|
| `stripe`, `stripe_intent`, `stripe_terminal`, `stripe_link` | STRIPECONNECT |
| `quickpay`, `mobilepay_online` | QUICKPAY |
| `checkout` | CHECKOUT |
| `adyen`, `adyen_pos` | ADYEN |
| `viva_wallet` | VIVA_WALLET |
| `network_international` | NETWORK_INTERNATIONAL |
| `mashreq` | MASHREQ |
| `verifone`, `verifone_pos` | VERIFONE |
| `walmart_pay` | WALMART_PAY |
| `pinelabs` | PINELABS |
| `cybersource` | CYBERSOURCE |

**Key behaviors:**
- Batches stores in chunks of 30 for API calls to `MS_PAY_BASE_URL/api/v1/payout/get-payments-refunds/`.
- Stores per-order metadata in `SettlementOrder.extra_data`: card brand, card last 4, auth code, cashier code, platform, offline transaction flag.
- DubaiDutyFree business date offset: +1 hour via `STORE_TYPE_BUSINESS_DATE_HOUR_DIFFERENCE_MAP`.
- Extracts cashier code from `order.additional_order_details.staff_details` checking both `"transaction"` and `"scan_and_go"` platform keys.

#### Stripe Settlement & Payouts

**File:** `mishipay_alerts/management/commands/stripe_connect_psp_settlement_payout.py` (862 lines)

Supports both direct Stripe API and MS Pay approaches. The largest management command.

**Stripe object graph traversal:**
```
Charges → Transfers → Destination Charges → Balance Transactions
```

**Weekend payout adjustment:** Stripe payouts on Saturday shift to Monday; Sunday payouts shift to Tuesday.

**Key constant:** `LOWEST_DENOMINATION_MULTIPLIER = Decimal("100")` — Stripe amounts are in cents.

#### Nordic File-Based Settlement

**File:** `mishipay_alerts/management/commands/quickpay_psp_settlement_by_files.py` (895 lines)

Specific to Flying Tiger stores. Parses settlement files from three Nordic payment providers:

| Method | File Format | Encoding | Region | Payout Reference Format |
|--------|------------|----------|--------|------------------------|
| MobilePay | TellerDK CSV | Latin-1 (ISO-8859-1) | DK | `{txn_date}-{payout_date}-FTC-MOBILEPAY-DK` |
| Swish | ISO 20022 camt.053 XML | UTF-8 | SE | From XML `NtryRef` element |
| Vipps | Vipps SettlementReport-3.0 XML | UTF-8 | NO | From XML `PayoutDate` element |

**Note:** Swish refund processing is **disabled** with comment: "Refund disabled in swish as we don't receive proper data." Hardcoded file paths: `/app/reconciliation/flying_tiger/{region}/live/`.

#### Final Settlement Report

**File:** `mishipay_alerts/management/commands/mishipay_settlement_report_final.py` (627 lines)

Aggregation pipeline that generates final reconciliation reports:

1. **`fill_missing_settlement_orders()`** — Finds MishiPay Orders with no matching `SettlementOrder` record and creates FAILED status records. Uses PSP reference ID prefix heuristic: `ch_`/`pi_` → STRIPECONNECT, `pay` → CHECKOUT, else → QUICKPAY.
2. **`update_poslog_status_of_records()`** — Re-checks `order.extra_data['poslog_sent']` for previously-missing records.
3. **`generate_settlement_poslog_recon_report()`** — Django ORM `values().annotate(Sum, Count)` aggregation → `SettlementReconByDate` + `SettlementFinalSummary`.
4. **`generate_settlement_payout_recon_report()`** — Aggregation by payout date → `SettlementFinalSummaryByPayoutDates`.

### Dashboard Reconciliation Helpers

**File:** `mishipay_alerts/helpers/psp_recon_payouts_helper.py` (1,379 lines)

Dashboard-facing helper functions for settlement reconciliation views, CSV downloads, and payout matching. Used by the `mishipay_dashboard` module.

**Key capabilities:**
- **Summary views** — `get_payment_settlement_summary_all_regions_new()` aggregates settlement data with PSP display name mapping (e.g., `quickpay` → "Mobilepay" in DK, "Swish" in SE, "Vipps" in NO).
- **POSLOG reconciliation** — `get_payment_settlement_poslog_recon_by_region()` with filters for auth code, tender, transaction ID range, register number, and comma-separated store IDs.
- **CSV export** — `download_csv_payment_settlement_poslog_recon_by_region()` with retailer-specific column sets (DubaiDutyFree adds SCO ID; WHSmith adds Cashier Code; Walmart removes register/card_last4).
- **Payout matching** — `get_payment_settlement_recon_payout_by_region_and_psp()` uses a "closest amount" heuristic when exact match fails.
- **Combination-sum algorithm** — `find_combination_of_numbers()` recursively finds which set of daily recon amounts match a payout total when a single payout covers multiple business dates.

**Retailer-specific defaults:**
- WHSmith: Cashier code defaults to `"5555"` for Scan&Go platforms, `"1"` for Kiosk platforms.
- WHSmith: Register number defaults to `"200"`.

## Utility Functions

### alert_util.py

**File:** `mishipay_alerts/alert_util.py` (14 lines)

| Function | Purpose |
|----------|---------|
| `send_payment_slack_alert(payment_alerts, alert_channel, alert_type_to_actions)` | Formats PaymentAlert records as CSV-like rows and sends to Slack |
| `get_minimal_store_type(store_type)` | Strips "StoreType" suffix — e.g., `"WHSmithStoreType"` → `"WHSmith"` |

### date_util.py

**File:** `mishipay_alerts/date_util.py` (33 lines)

| Function | Purpose |
|----------|---------|
| `get_date_range(business_date_str, timezone)` | Returns `(start_date, end_date)` tuple — start at 00:00:00.000000, end at 23:59:59.999999 with timezone |
| `get_date_range_without_timezone(business_date_str)` | Same but returns naive datetimes |

## Admin Interface

**File:** `mishipay_alerts/admin.py` (120 lines)

9 admin registrations for all non-abstract models:

| Admin Class | Model | Key list_display |
|------------|-------|------------------|
| `PaymentAlertAdmin` | PaymentAlert | alert_type, psp_type, payment_method, status, store_id |
| `RetailerImportJobAdmin` | RetailerImportJob | import_type, business_date, retailer, status |
| `RetailerExportJobAdmin` | RetailerExportJob | export_type, business_date, retailer, status |
| `SettlementOrderAdmin` | SettlementOrder | order_id, psp_reference_id, payout_reference_id, psp_charged_amount |
| `SettlementOrderRefundAdmin` | SettlementOrderRefund | refund_order_id, psp_refund_amount, payout_available_on |
| `SettlementReconByDateAdmin` | SettlementReconByDate | business_date, psp_order_amount, psp_order_fee_amount |
| `SettlementPayoutByDateAdmin` | SettlementPayoutByDate | payout_reference_id, payout_amount, payout_actual_date |
| `SettlementFinalSummaryAdmin` | SettlementFinalSummary | business_date, psp_net_amount, payout_amount, payout_amount_cummulative |
| `SettlementFinalSummaryByPayoutDatesAdmin` | SettlementFinalSummaryByPayoutDates | payout_actual_date, psp_net_amount, payout_amount |

All use exact-match search (`=field`) on UUID fields. All ordered by descending date fields.

## Dependencies (Alerts Module)

### Internal Dependencies

| Dependency | Used For |
|-----------|----------|
| `dos.models.Store` | Store lookup by store_id, retailer_store_id, chain, region, timezone |
| `mishipay_retail_orders.models.Order` | Order lookup, status verification, extra_data, additional_order_details |
| `mishipay_retail_orders.models.OrderStatus` | Order status constants |
| `mishipay_retail_orders.models.SUCCESS_ORDER_STATUSES` | Set of successful statuses for filtering |
| `mishipay_retail_payments.models.PaymentVariant` | PSP account credentials per store |
| `mishipay_retail_payments.models.Payments` | Payment records |
| `mishipay_retail_payments.PSPType` | PSP type constants (STRIPECONNECT, QUICKPAY, CHECKOUT, etc.) |
| `mishipay_dashboard.models.RefundOrderNew` | Refund order records |
| `mishipay_dashboard.util.update_order_payment_info_manual` | Auto-remediation of order status |
| `mishipay_core.common_functions.send_slack_message` | Slack notifications |
| `mishipay_core.choices` | SCAN_AND_GO_DEVICE_TYPES, ANDROID_KIOSK, ANDROID_CASHIER_KIOSK |

### External Services

| Service | Integration Method | Used In |
|---------|-------------------|---------|
| Stripe API | Direct via `stripe` library | stripe_connect_payment_alert, stripe_connect_psp_settlement_payout |
| QuickPay API | Direct HTTP | quickpay_payment_alert, quickpay_psp_settlement_deprecated |
| MS Pay Microservice | HTTP via `settings.MS_PAY_BASE_URL` | ms_pay_based_settlement_payout, stripe_connect_psp_settlement_payout, checkout_psp_settlement_transactions |
| Slack | Via `send_slack_message` | Alert commands |

### What Depends on This Module

- `mishipay_dashboard` — uses `psp_recon_payouts_helper.py` for settlement reconciliation views and CSV downloads.

## Test Coverage (Alerts)

**File:** `mishipay_alerts/tests.py` (937 lines)

| Test Class | Tests | Focus |
|-----------|-------|-------|
| `SettlementOrderFilterTest` | UUID filtering with/without hyphens | Ensures records can be filtered by both UUID formats |
| `GetPaymentSettlementSummaryAllRegionsTest` | Summary function with mocked data | Tests `get_payment_settlement_summary_all_regions_new()` |
| `GetPaymentSettlementPosLogReconByRegionTest` | Poslog recon with mocked dependencies | Tests filtering and pagination |
| `CashierCodeCSVTest` | WHSmith cashier code defaults in CSV | Verifies Scan&Go→5555, Kiosk→1 defaults |
| `SettlementCSVRefundTransactionIDTest` | Refund transaction ID fallback | Tests WHSmith-specific fallback to `row.status` |

Tests use `unittest.mock.Mock` and `unittest.mock.patch` extensively.

## Notable Patterns (Alerts)

1. **Management commands as business logic** — 4,262 of ~5,600 lines live in management commands. The REST API surface is minimal (4 thin endpoints).

2. **Idempotent delete-then-create** — Every settlement command deletes existing records for the date range within `transaction.atomic()`, then `bulk_create`s new ones. Safe for re-runs.

3. **PSP API evolution** — Clear migration path: Legacy (direct Stripe/QuickPay API) → Transitional (both approaches) → Modern (MS Pay only). The deprecated `quickpay_psp_settlement_deprecated.py` is effectively dead code.

4. **Demo/prod toggle** — Commands check `settings.ENV_TYPE == 'prod'` for Slack channel selection.

5. **Retailer-specific logic** — WHSmith (cashier codes, register numbers), DubaiDutyFree (business date hour offset, SCO ID), Flying Tiger (file-based Nordic settlement), Virgin (Checkout.com), Walmart (specific CSV column removal).

6. **Combination-sum for payout matching** — When a single PSP payout covers multiple business dates, uses a recursive combination-sum algorithm to find which daily amounts compose the total.

## Open Questions (Alerts)

- Authentication is commented out on all REST endpoints in `views.py`. Are these endpoints protected at the infrastructure level (e.g., internal-only network routing), or is this a security gap?
- `apps.py` defines `MishipayCouponsConfig` instead of `MishipayAlertsConfig` — a copy-paste artifact from `mishipay_coupons`. Does this cause any runtime issues?
- `inventory_promo_poslog_alert_helper.py:19-21` has a bug: references `store.retailer_store_id` before `store` is assigned in the `else` branch. This would cause a `NameError` when `store_id` is not provided. Is this code path ever executed?
- The `RESOLVED` status constant is defined twice in `models.py` (lines 9 and 19) with the same value. Is the intent to have distinct "alert resolved" vs "recon resolved" statuses?
- Swish refund processing is disabled in `quickpay_psp_settlement_by_files.py` — how are Swish refunds reconciled?
- `quickpay_psp_settlement_deprecated.py` has its `process_quickpay_payouts` function entirely commented out. Should this file be removed?
- Are the hardcoded file paths in `quickpay_psp_settlement_by_files.py` (`/app/reconciliation/flying_tiger/{region}/live/`) specific to the production container filesystem?
- The PSP reference ID prefix heuristic (`ch_`/`pi_` → Stripe, `pay` → Checkout) in `mishipay_settlement_report_final.py` — is this reliable for all current PSPs, or could it misclassify newer providers?
