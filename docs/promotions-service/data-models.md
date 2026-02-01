# Promotions Service — Data Models

> Last updated by: Task T02 — Data Models

## Overview

The promotions service uses 6 Django models organized in a hierarchical structure. A **PromoHeader** defines a promotion, which contains **PromoGroups** (sets of qualifying items/categories), which in turn contain **PromoGroupNodes** (individual SKUs or categories). Supporting models link promotions to **stores** and **special promo programs** (loyalty), and **PromoApplicationFilter** provides advanced per-node filtering rules.

All models use **KSUID** (K-Sortable Unique Identifiers) as primary keys, and **no Django ForeignKey relationships** are used — all inter-model references are plain `CharField` columns managed at the application level. This is a deliberate design decision for scalability (see [Overview](./overview.md)).

## Entity Relationship Diagram

```
┌──────────────────────────┐
│      PromoHeader         │
│  (PK: ksuid)             │
│  retailer, family,       │
│  discount_type, layer,   │
│  start/end dates ...     │
└─────────┬────────────────┘
          │ 1
          │
          │ promo_header_id (CharField, not FK)
          │
    ┌─────┼────────────────────────┬──────────────────────┐
    │     │ *                      │ *                    │ *
    ▼     ▼                        ▼                      ▼
┌─────────────────┐  ┌────────────────────────┐  ┌──────────────────────┐
│   PromoGroup    │  │  PromoHeaderStore      │  │PromoHeaderSpecialPromo│
│  (PK: ksuid)    │  │  (PK: ksuid)           │  │  (PK: ksuid)         │
│  name, qty_min, │  │  store (UUID),         │  │  group_qualifier_id, │
│  qty_max        │  │  soft_delete           │  │  description         │
└────────┬────────┘  └────────────────────────┘  └──────────────────────┘
         │ 1
         │
         │ promo_group_id (CharField, not FK)
         │
    ┌────┴──────────────┐
    │ *                  │ *
    ▼                    ▼
┌──────────────────┐  ┌─────────────────────────┐
│ PromoGroupNode   │  │ PromoApplicationFilter  │
│ (PK: ksuid)      │  │ (PK: ksuid)             │
│ node_id, type,   │  │ promo_header_id,        │
│ is_excluded,     │  │ promo_group_id,         │
│ discount_type,   │  │ node_id,                │
│ discount_value   │  │ filters_json            │
└──────────────────┘  └─────────────────────────┘
```

## Models

### 1. PromoHeader

**Source:** `service/mpromo/models/promo_header.py`
**Table:** `mpromo_promoheader`

The central model defining a promotion. Each PromoHeader represents a single promotion rule with its discount, scheduling, and behavioral configuration.

#### Fields

| Field | Type | Constraints | Default | Description |
|-------|------|-------------|---------|-------------|
| `ksuid` | `CharField(40)` | **PK**, auto-generated, not editable | `get_ksuid()` | K-Sortable Unique ID |
| `retailer` | `CharField(63)` | indexed | — | Retailer identifier |
| `promo_id_retailer` | `CharField(255)` | nullable, indexed | `""` | Retailer's own promo ID |
| `family` | `CharField(1)` | choices: `FAMILIES`, indexed | `'e'` (Easy) | Promotion family type (see [Overview](./overview.md#promotion-families)) |
| `evaluate_criteria` | `CharField(1)` | choices: `PROMO_EVALUATE_CRITERIA`, indexed | `'p'` (Priority) | How to select when multiple promos apply |
| `evaluate_priority` | `PositiveSmallIntegerField` | nullable | — | Lower number = higher priority |
| `promo_short_code` | `CharField(32)` | nullable, indexed | `""` | Short reference code |
| `title` | `CharField(255)` | — | `""` | Human-readable title |
| `description` | `CharField(255)` | nullable | — | Optional description |
| `discount_type` | `CharField(1)` | nullable, choices: `DISCOUNT_TYPES`, indexed | `'p'` (%Off) | Type of discount applied |
| `discount_type_strategy` | `CharField(1)` | choices: `DISCOUNT_TYPE_STRATEGIES`, indexed | `'a'` (All Items) | Whether discount applies to all items collectively or each individually |
| `discount_value` | `DecimalField(15,2)` | nullable | `0.0` | Discount amount or percentage |
| `discount_value_on` | `CharField(1)` | choices: `DISCOUNT_VALUE_ON_PRICE_TYPES`, indexed | `'m'` (MRP) | Which price the discount applies on (MRP, Sale Price, Final Price) |
| `max_application_limit` | `PositiveSmallIntegerField` | — | `1` | Max times this promo can apply to a basket |
| `is_active` | `BooleanField` | — | `True` | Whether the promo is currently active |
| `active_days` | `CharField(7)` | — | `"1111111"` | Binary string for active days (Mon=pos0 through Sun=pos6). `"1111111"` = every day. |
| `start_date_time` | `DateTimeField` | — | — | Promotion start time |
| `end_date_time` | `DateTimeField` | — | — | Promotion end time |
| `is_happy_hour` | `BooleanField` | indexed | `False` | If `True`, promo only applies during the time-of-day window defined by start/end times |
| `discounted_group_item_selection_criteria` | `CharField(2)` | choices: `GROUP_ITEM_SELECTION_CRITERIAS` | `'l'` (Favor Retailer) | How to pick items for discounting from the target group |
| `requisite_groups_item_selection_criteria` | `CharField(2)` | nullable, choices: `GROUP_ITEM_SELECTION_CRITERIAS` | `'l'` (Favor Retailer) | How to pick items from requisite (qualifying) groups |
| `target_discounted_group_name` | `CharField(64)` | nullable, indexed | — | Name of the group that receives the discount (e.g., `"g1"`) |
| `target_discounted_group_qty_min` | `PositiveSmallIntegerField` | nullable | — | Min quantity in discounted group (for requisite family with single group) |
| `layer` | `PositiveSmallIntegerField` | — | `1` | Evaluation layer — promotions are evaluated layer by layer in ascending order |
| `discount_apply_type` | `CharField(1)` | choices: `DISCOUNT_APPLY_TYPES` | `'b'` (Basket) | How the discount is applied (basket, cashback, tender, or POSLog exclusion) |
| `availability` | `CharField(1)` | choices: `AVAILABILITIES` | `'a'` (All) | Whether promo is for all customers or special/loyalty only |
| `apply_on_discounted_items` | `BooleanField` | indexed | `True` | Whether this promo can stack on items already discounted by other promos |
| `extra_data` | `JSONField` | nullable | — | Arbitrary extra data (flexible extension point) |
| `updated_on` | `DateTimeField` | auto_now | — | Last modification timestamp |
| `max_discount` | `DecimalField(15,2)` | nullable | `None` | Maximum discount cap (e.g., "up to $50 off") |

#### Indexes

| Fields | Type | Name |
|--------|------|------|
| `start_date_time`, `end_date_time`, `is_active` | Composite index | auto |

#### Key Concepts

- **Layer:** Promotions are evaluated in layers (1, 2, 3...). Layer 1 promos are evaluated first. Layer 2 promos can apply on top of items already discounted by layer 1, depending on `apply_on_discounted_items`.
- **Family:** Determines which evaluation algorithm is used (see [Overview](./overview.md#promotion-families) for all 8 family types).
- **Evaluate Criteria:** When multiple promos in the same layer match, this field controls selection: by priority number (`'p'`) or by best discount (`'b'`).
- **Active Days:** A 7-character binary string where each position represents a day of the week (Monday=0 through Sunday=6). `"1010100"` means active on Mon, Wed, Fri only.
- **Happy Hour:** When enabled, the promotion only applies during the time-of-day portion of `start_date_time` and `end_date_time`, regardless of the date.
- **Target Discounted Group:** For multi-group promotions (e.g., "Buy X from group g1, get Y from group g2 at discount"), this field specifies which group receives the discount.

---

### 2. PromoGroup

**Source:** `service/mpromo/models/promo_group.py`
**Table:** `mpromo_promogroup`

A group of items or categories within a promotion. Each promotion has one or more groups. Groups define the qualifying conditions (what the customer needs in their basket) and/or the discount targets (what gets discounted).

#### Fields

| Field | Type | Constraints | Default | Description |
|-------|------|-------------|---------|-------------|
| `ksuid` | `CharField(40)` | **PK**, auto-generated, not editable | `get_ksuid()` | K-Sortable Unique ID |
| `name` | `CharField(64)` | — | `""` | Group name — convention is `g1`, `g2`, `g3`, etc. |
| `promo_header_id` | `CharField(40)` | indexed | — | Reference to `PromoHeader.ksuid` (not a FK) |
| `qty_or_value_min` | `PositiveSmallIntegerField` | required | — | Minimum quantity or value to qualify |
| `qty_or_value_max` | `PositiveSmallIntegerField` | nullable | — | Maximum quantity or value (optional cap) |

#### Constraints

| Type | Fields | Name |
|------|--------|------|
| `UniqueConstraint` | `promo_header_id`, `name` | `unique_promo_header_id_groupname` |

A promotion cannot have two groups with the same name. This enforces the `g1`, `g2` naming convention at the database level.

#### Key Concepts

- **Group naming:** Groups follow the convention `g1`, `g2`, `g3`, etc. The `target_discounted_group_name` on PromoHeader references these names.
- **Quantity semantics:** `qty_or_value_min` represents the minimum number of items (or minimum basket value, depending on the promo family) needed to qualify. For basket threshold families, this is a monetary value; for other families, it is a quantity.

---

### 3. PromoGroupNode

**Source:** `service/mpromo/models/promo_group_node.py`
**Table:** `mpromo_promogroupnode`

An individual node (item SKU or category) within a PromoGroup. Nodes define which specific items or categories are part of a group.

#### Fields

| Field | Type | Constraints | Default | Description |
|-------|------|-------------|---------|-------------|
| `ksuid` | `CharField(40)` | **PK**, auto-generated, not editable | `get_ksuid()` | K-Sortable Unique ID |
| `promo_group_id` | `CharField(40)` | — | — | Reference to `PromoGroup.ksuid` (not a FK) |
| `node_id` | `CharField(128)` | indexed | — | The actual identifier — a SKU, barcode, or category ID |
| `node_type` | `CharField(2)` | choices: `NODE_TYPES`, indexed | `'i'` (Item) | Whether this is an item (`'i'`) or category level (`'c1'` through `'c7'`) |
| `is_excluded` | `BooleanField` | indexed | `False` | If `True`, this node is **excluded** from the group (exception) |
| `discount_type` | `CharField(1)` | nullable, choices: `DISCOUNT_TYPES`, indexed | — | Node-level override for discount type |
| `discount_value` | `DecimalField(15,2)` | nullable | — | Node-level override for discount value |

#### Indexes

| Fields | Type |
|--------|------|
| `promo_group_id`, `node_id` | Composite index |

#### Key Concepts

- **Node types:** A node can represent an individual item (`'i'`), or a category at any of 7 hierarchy levels (`'c1'` through `'c7'`). Category nodes match **all items** within that category.
- **Exclusions:** Setting `is_excluded=True` removes this specific item/category from the group. This allows patterns like "all items in category X except SKU Y".
- **Per-node discount overrides:** The `discount_type` and `discount_value` fields on PromoGroupNode allow node-level discount overrides, meaning different items within the same group can have different discount amounts. When `NULL`, the header-level discount applies.
- **No FK on promo_group_id:** Note that `promo_group_id` is **not indexed by default** (`db_index=False`), but is part of the composite index with `node_id`. This is optimized for lookups that filter by both group and node.

---

### 4. PromoHeaderStore

**Source:** `service/mpromo/models/promo_header_store.py`
**Table:** `mpromo_promoheaderstore`

Links a promotion to a specific store. A promotion is only active in stores where a PromoHeaderStore record exists with `soft_delete=False`.

#### Fields

| Field | Type | Constraints | Default | Description |
|-------|------|-------------|---------|-------------|
| `ksuid` | `CharField(40)` | **PK**, auto-generated, not editable | `get_ksuid()` | K-Sortable Unique ID |
| `promo_header_id` | `CharField(40)` | indexed | — | Reference to `PromoHeader.ksuid` (not a FK) |
| `soft_delete` | `BooleanField` | — | `False` | Soft delete flag — `True` deactivates the promo for this store |
| `store` | `UUIDField` | — | — | Store UUID from the monolith (`backend_v2`) |

#### Constraints

| Type | Fields | Name |
|------|--------|------|
| `UniqueConstraint` | `store`, `soft_delete`, `promo_header_id` | `unique_store_soft_delete_promo_header_id` |

#### Key Concepts

- **Store scoping:** A promotion without any PromoHeaderStore records is not active in any store. Stores must be explicitly assigned.
- **Soft delete:** Instead of deleting the store-promo link, `soft_delete` is set to `True`. The unique constraint includes `soft_delete`, meaning the same store-promo pair can have both an active (`False`) and deleted (`True`) record. This preserves history.
- **Store UUID:** The `store` field is a UUID that comes from the monolith's store model, bridging the two systems.

---

### 5. PromoHeaderSpecialPromo

**Source:** `service/mpromo/models/promo_header_special_promo.py`
**Table:** `mpromo_promoheaderspecialpromo`

Links a promotion to a special/loyalty program qualifier. Used when `PromoHeader.availability = 's'` (special). The customer must present a matching qualifier (e.g., loyalty card number, loyalty tier) to unlock the promotion.

#### Fields

| Field | Type | Constraints | Default | Description |
|-------|------|-------------|---------|-------------|
| `ksuid` | `CharField(40)` | **PK**, auto-generated, not editable | `get_ksuid()` | K-Sortable Unique ID |
| `promo_header_id` | `CharField(40)` | indexed | — | Reference to `PromoHeader.ksuid` (not a FK) |
| `group_qualifier_id` | `CharField(64)` | indexed | — | Loyalty program group qualifier identifier |
| `description` | `CharField(255)` | nullable | — | Human-readable description of the qualifier |

#### Key Concepts

- **Special promotions:** When a PromoHeader has `availability='s'`, only customers who match at least one `PromoHeaderSpecialPromo.group_qualifier_id` can access the promotion.
- **Group qualifiers:** These identifiers represent loyalty tiers, membership groups, or other customer segments. The evaluate endpoint receives `special_promos` in the request body, which are matched against these qualifiers.
- **Multiple qualifiers:** A single promotion can have multiple special promo records, meaning customers from any of those groups qualify.

---

### 6. PromoApplicationFilter

**Source:** `service/mpromo/models/promo_application_filter.py`
**Table:** `mpromo_promoapplicationfilter`

Provides fine-grained filtering rules for promotion application at the node level. Contains JSON-based filter criteria that can impose additional conditions on when a specific node qualifies for a promotion.

#### Fields

| Field | Type | Constraints | Default | Description |
|-------|------|-------------|---------|-------------|
| `ksuid` | `CharField(40)` | **PK**, auto-generated, not editable | `get_ksuid()` | K-Sortable Unique ID |
| `promo_header_id` | `CharField(40)` | indexed | — | Reference to `PromoHeader.ksuid` (not a FK) |
| `promo_group_id` | `CharField(40)` | indexed | — | Reference to `PromoGroup.ksuid` (not a FK) |
| `node_id` | `CharField(40)` | indexed | — | Reference to `PromoGroupNode.ksuid` (not a FK) |
| `filters_json` | `JSONField` | required | — | JSON object containing filter criteria |

#### Key Concepts

- **Three-level reference:** This model references all three levels of the hierarchy — header, group, and node — allowing filters to be applied at any granularity.
- **JSON filters:** The `filters_json` field contains arbitrary filter rules. The structure is not enforced at the model level, giving maximum flexibility for different filter types.
- **Not registered in admin:** Unlike the other 5 models, PromoApplicationFilter is not registered in the Django admin interface.

---

## Serializers

**Source:** `service/mpromo/serializers.py`

### Model Serializers (CRUD)

| Serializer | Model | Fields | Notes |
|------------|-------|--------|-------|
| `PromoHeaderSerializer` | `PromoHeader` | `__all__` | `ksuid` is read-only; `promo_short_code` is optional |
| `PromoHeaderStoreSerializer` | `PromoHeaderStore` | `__all__` | `ksuid` is read-only |
| `PromoGroupNodeSerializer` | `PromoGroupNode` | `__all__` | `ksuid` is read-only |
| `PromoGroupSerializer` | `PromoGroup` | `ksuid`, `name`, `promo_header_id`, `qty_or_value_min`, `qty_or_value_max`, `promo_group_nodes` | Includes nested `promo_group_nodes` via `SerializerMethodField` — loads all nodes belonging to this group |
| `PromoHeaderSpecialPromoSerializer` | `PromoHeaderSpecialPromo` | `group_qualifier_id`, `description` | Minimal — only exposes qualifier info, not ksuid or promo_header_id |

### Composite Serializer

| Serializer | Description |
|------------|-------------|
| `PromoSerializer` | Full promotion representation. Based on `PromoHeader` with `__all__` fields plus three computed fields: `promo_groups` (nested groups with nodes), `stores` (list of store UUIDs), `special_promo_info` (qualifier data). **Note:** `promo_groups` returns `None` if the promotion has no stores assigned. |

`PromoSerializer` assembles the full promotion hierarchy in a single response:

```json
{
  "ksuid": "...",
  "retailer": "...",
  "family": "e",
  "discount_type": "p",
  "discount_value": "10.00",
  "...other header fields...": "...",
  "stores": ["uuid-1", "uuid-2"],
  "special_promo_info": [
    {"group_qualifier_id": "GOLD", "description": "Gold members"}
  ],
  "promo_groups": [
    {
      "ksuid": "...",
      "name": "g1",
      "promo_header_id": "...",
      "qty_or_value_min": 2,
      "qty_or_value_max": null,
      "promo_group_nodes": [
        {
          "ksuid": "...",
          "promo_group_id": "...",
          "node_id": "SKU-12345",
          "node_type": "i",
          "is_excluded": false,
          "discount_type": null,
          "discount_value": null
        }
      ]
    }
  ]
}
```

### Evaluation Request Serializers

These serializers define the shape of the **basket evaluation** request (POST to `/api/1.0/promotions/evaluate/`):

| Serializer | Fields | Description |
|------------|--------|-------------|
| `PromoEvaluateRequestSerializer` | `customer_id`, `store_id`, `special_promos`, `basket` | Top-level evaluate request |
| `Basket` | `id`, `pre_tax`, `tax`, `total`, `items` | Basket totals and item list |
| `BasketItem` | `id`, `barcode`, `sku`, `mrp`, `sp`, `uom`, `qty_or_weight`, `units`, `catgories` | Individual basket item with pricing |
| `BasketItemCategory` | `name`, `value` | Category classification for an item |
| `SpecialPromoProgram` | `program_name`, `group_qualifiers` | Loyalty program data |
| `SpecialPromoProgramGroupQualifier` | `input`, `qualifier_ids` | Qualifier group with IDs |

**BasketItem fields explained:**
- `mrp` — Maximum Retail Price (original price)
- `sp` — Sale Price (current selling price, may differ from MRP)
- `uom` — Unit of Measure (e.g., `"kg"`, `"unit"`)
- `qty_or_weight` — Quantity or weight depending on UOM
- `units` — Number of units
- `catgories` — List of category name/value pairs (note: typo in field name, `catgories` not `categories`)

## Django Admin Configuration

**Source:** `service/mpromo/admin.py`

Five of the 6 models are registered in the Django admin (`PromoApplicationFilter` is excluded):

| Model | List Display | List Filter | Search Fields | Ordering |
|-------|-------------|-------------|---------------|----------|
| `PromoHeader` | ksuid, promo_id_retailer, is_active, title, evaluate_priority, discount_type, discount_value, availability, family, max_discount | retailer, family | =ksuid, promo_id_retailer, title, =discount_type, =discount_value | -ksuid (newest first) |
| `PromoHeaderStore` | store, promo_header_id, soft_delete | — | =store, =promo_header_id | -ksuid |
| `PromoGroup` | ksuid, name, promo_header_id, qty_or_value_min, qty_or_value_max | — | =ksuid, =promo_header_id | -ksuid |
| `PromoGroupNode` | ksuid, promo_group_id, node_id, node_type, is_excluded | — | =promo_group_id, =node_id | -ksuid |
| `PromoHeaderSpecialPromo` | ksuid, promo_header_id, group_qualifier_id, description | — | =promo_header_id, =group_qualifier_id | -ksuid |

**Note:** Search fields prefixed with `=` indicate exact match searches (not partial/contains).

## Design Patterns

### No Foreign Keys

All inter-model relationships use `CharField` with manually managed IDs instead of Django `ForeignKey`:
- `PromoGroup.promo_header_id` → `PromoHeader.ksuid`
- `PromoGroupNode.promo_group_id` → `PromoGroup.ksuid`
- `PromoHeaderStore.promo_header_id` → `PromoHeader.ksuid`
- `PromoHeaderSpecialPromo.promo_header_id` → `PromoHeader.ksuid`
- `PromoApplicationFilter.promo_header_id` → `PromoHeader.ksuid`
- `PromoApplicationFilter.promo_group_id` → `PromoGroup.ksuid`
- `PromoApplicationFilter.node_id` → `PromoGroupNode.ksuid`

**Consequences:**
- No cascading deletes — deletions must be managed in application code
- No database-level referential integrity checks
- No Django ORM reverse relations (e.g., `promo_header.groups.all()`)
- All related-object queries must be explicit: `PromoGroup.objects.filter(promo_header_id=header.ksuid)`
- Enables horizontal scaling and avoids FK constraint overhead at high volumes

### KSUID Primary Keys

All models use KSUID (K-Sortable Unique Identifiers) as primary keys instead of auto-incrementing integers:
- **Globally unique** across all tables and services
- **Time-sortable** — KSUIDs encode a timestamp, so ordering by ksuid is chronological ordering
- **Generated at application level** — no database sequence dependency
- **String type** — 27-character base62-encoded string stored in `CharField(40)`

### Soft Delete Pattern

Only `PromoHeaderStore` uses soft delete (`soft_delete` boolean field). Other models are presumably hard-deleted when no longer needed. The unique constraint on PromoHeaderStore includes `soft_delete`, allowing the same store-promo pair to coexist in both active and deleted states.

## Promotion Data Flow Example

A "Buy 2 shirts, get 10% off the cheapest" promotion would be structured as:

```
PromoHeader:
  family: 'e' (Easy)
  discount_type: 'p' (%Off)
  discount_value: 10.00
  max_application_limit: 1
  target_discounted_group_name: 'g1'
  discounted_group_item_selection_criteria: 'l' (least expensive / favor retailer)

PromoGroup (g1):
  name: 'g1'
  qty_or_value_min: 2

PromoGroupNode (for each qualifying shirt SKU):
  node_type: 'i' (item)
  node_id: 'SHIRT-001', 'SHIRT-002', etc.
  is_excluded: False

PromoHeaderStore (for each store):
  store: <store-uuid>
  soft_delete: False
```

When a basket is evaluated, the engine:
1. Finds active PromoHeaders for the store
2. Matches basket items to PromoGroupNodes
3. Checks quantity requirements against PromoGroup thresholds
4. Calculates the discount using the family-specific algorithm
5. Returns the best combination of discounts for the basket
