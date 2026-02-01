# Promotions Service -- Core Business Logic

> Last updated by: Task T04 -- Promotions Service Promotion Families (deep-dive into all 8 family implementations + math utilities)

## Overview

The promotions engine evaluates a basket of items against active promotions for a given store and returns the optimal set of discounts. It supports 8 promotion family types, layered evaluation, two best-discount algorithms (exhaustive and marginal-value), and two engine versions (V1 and V2). The core logic lives in `service/mpromo/promo_core/`.

---

## Architecture

The business logic layer follows a **Factory/Executer** design pattern with four distinct sub-layers:

```
Request
  |
  v
[PromoLogic / PromoLogicV2]  -- Evaluation orchestrator
  |
  v
[promo_db_helper / promo_db_helper_raw_sql]  -- Data access
  |
  v
[PromoDataManager]  -- Assembles flat DB rows into hierarchical Promo objects
  |
  v
[PromoFamilyExecuter]  -- Dispatches to the correct family class
  |
  v
[PromoFamilyFactory subclasses]  -- 8 concrete family implementations
```

### Key Files

| File | Class/Function | Purpose |
|------|----------------|---------|
| `promo_logic.py` | `PromoLogic` | V1 evaluation engine (~1,315 lines) |
| `promo_logic_v2.py` | `PromoLogicV2` | V2 evaluation engine (~1,235 lines) |
| `promo_family_executer.py` | `PromoFamilyExecuter` | Dispatcher: maps family codes to singleton class instances |
| `promo_family_factory.py` | `PromoFamilyFactory` | Abstract base class with shared discount calculation logic (~416 lines) |
| `promo_data_manager.py` | `Promo`, `PromoDataManager` | Data assembly: flat rows to hierarchical objects |
| `promo_db_helper.py` | `get_active_promotions()` | ORM-based active promotion query |
| `promo_db_helper_raw_sql.py` | `get_active_promotions()` | Raw SQL version + on-the-fly promotions |
| `easy_promo_family.py` | `EasyPromoFamily` | Easy family implementation |
| `easy_plus_promo_family.py` | `EasyPlusPromoFamily` | Easy+ family implementation |
| `combo_promo_family.py` | `ComboPromoFamily` | Combo family implementation |
| `line_special_promo_family.py` | `LineSpecialPromoFamily` | Line Special family implementation |
| `basket_threshold_promo_family.py` | `BasketThresholdPromoFamily` | Basket Threshold family implementation |
| `basket_threshold_with_discounted_target_promo_family.py` | `BasketThresholdWithDiscountedTargetPromoFamily` | Basket Threshold with Discounted Target |
| `requisite_groups_with_discounted_target_promo_family.py` | `RequisiteGroupsWithDiscountedTargetPromoFamily` | Requisite Groups with Discounted Target |
| `evenly_distributed_multiline_discount_promo_family.py` | `EvenlyDistributedMultilineDiscountPromoFamily` | Evenly Distributed Multiline Discount |
| `config.py` | (constants) | Family codes, discount types, node types, evaluation criteria |

---

## Evaluation Pipeline

### High-Level Flow

The `evaluate()` method is the main entry point. Both V1 and V2 follow the same general pipeline:

```
1. Parse basket items -> build SKU/category dictionaries
2. Fetch active promotions from DB (via raw SQL helper)
3. Sort promotions into layers
4. For each layer:
   a. Prune invalid promotions
   b. Cluster promotions by evaluation criteria
   c. Apply best-discount promotions (exhaustive or marginal-value)
   d. Apply priority-based promotions
   e. [V2 only] Apply basket-level best-discount and priority promotions
5. Build response (aggregate discounts, format output)
6. [V1 only] Process pack items
```

### Step 1: Basket Parsing

`generate_sku_to_basket_items_and_category_to_sku_dict_from_evaluate_req()` processes the incoming request and builds three data structures:

| Data Structure | Type | Purpose |
|----------------|------|---------|
| `sku_to_basket_items_dict` | `Dict[str, List[Dict]]` | Maps SKU to list of basket item dicts (with mrp, sp, qty, sno, etc.) |
| `category_type_value_to_sku_list_dict` | `Dict[str, List[str]]` | Maps `"type-#$@-value"` keys to list of SKU strings |
| `category_values_set` | `Set[str]` | All unique category values for DB lookup |

Each basket item is augmented with default fields: `discount="0.000"`, `applied_promos=[]`, `applied_promos_prev_layers=[]`, `item_source="remaining_info"`, sequential `sno`.

### Step 2: Fetch Active Promotions

Both engines use `promo_db_helper_raw_sql.get_active_promotions()` (imported as `promo_db_helper`), which:

1. Executes a single raw SQL JOIN across `promo_group_node`, `promo_group`, `promo_header_store`, and `promo_header` tables
2. Filters by: `store_id`, `is_active=True`, `start_date <= now <= end_date`, node IDs matching basket SKUs/categories (or `"ALL"`)
3. Separates general-availability promos from special-availability promos
4. Handles special promo overrides from the request (e.g., loyalty program qualifiers)
5. Optionally creates on-the-fly promotions from `promo_override` data without DB persistence
6. Returns a `PromoDataManager` instance

See [Data Models](./data-models.md) for the underlying table schema.

#### ORM vs Raw SQL

There are two implementations of `get_active_promotions()`:

| Aspect | `promo_db_helper.py` (ORM) | `promo_db_helper_raw_sql.py` (Raw SQL) |
|--------|---------------------------|---------------------------------------|
| Query style | Django ORM chained filters | Single raw SQL with `django.db.connection.cursor()` |
| On-the-fly promos | Not supported | Supported via `get_on_the_fly_promotion()` |
| Active import | Not used by evaluation engines | Imported by both `PromoLogic` and `PromoLogicV2` |

The raw SQL helper is the active implementation. The ORM helper appears to be a legacy or fallback version.

### Step 3: Layer Sorting

`sort_promo_by_layers()` groups promotions by their `layer` field (integer). Layers are processed sequentially from lowest to highest. This enables multi-pass evaluation where later layers operate on the results of earlier layers.

Basket threshold promotions are forced to layer >= 100 via `create_promo()` in their family classes (`basket_threshold_promo_family.py:30`).

### Step 4: Promotion Clustering

Within each layer, promotions are clustered into evaluation groups:

#### V1 Cluster Types

| Cluster | Key | Criteria |
|---------|-----|----------|
| Priority | `priority` | `evaluate_criteria == 'p'` and not basket family |
| Best Discount | `best_discount` | `evaluate_criteria == 'b'` and not basket family |
| Basket | `basket` | `family == 'b'` or `family == 't'` (basket threshold families) |

#### V2 Cluster Types

V2 adds a fourth cluster type:

| Cluster | Key | Criteria |
|---------|-----|----------|
| Priority | `priority` | `evaluate_criteria == 'p'` and not basket family |
| Best Discount | `best_discount` | `evaluate_criteria == 'b'` and not basket family |
| Basket Priority | `basket` | Basket families with `evaluate_criteria == 'p'` |
| Basket Best Discount | `basket_best_discount` | Basket families with `evaluate_criteria == 'b'` |

### Step 5: Best Discount Algorithm

The best-discount algorithm ensures the customer receives the maximum possible discount when multiple competing promotions target overlapping items. It operates in two phases:

#### Phase A: Sub-Clustering via NetworkX Graph

`create_best_discount_sub_clusters()` uses the `networkx` library to partition promotions into independent sub-problems:

1. For each promotion, collect all target SKUs (resolving categories to SKUs via `category_type_value_to_sku_list_dict`)
2. If any promotion targets `node_id="ALL"`, all promos form a single cluster
3. Otherwise, build an undirected graph where SKUs within the same promotion are connected via `nx.add_path()`
4. A synthetic node (`"count@-123"`) is appended to each promo's SKU list to ensure single-item promotions form edges
5. `nx.connected_components()` identifies independent sub-clusters
6. Map SKUs back to promotions via `sku_to_promo_header_ids`

This reduces the combinatorial search space significantly when promotions target non-overlapping item sets.

#### Phase B: Finding Optimal Ordering

Two strategies exist, selected by a threshold:

**Standard Flow** (sub-cluster size <= 6): Exhaustive permutation search

```python
permutations = list(itertools.permutations(cluster))
for permutation in permutations:
    # Assign sequential priorities based on permutation order
    # Apply all promos in priority order
    # Track total discount
    # Keep permutation with maximum total discount
```

This has O(n!) complexity, hence the size-6 threshold. Performance logging tracks `BEST_DISCOUNT_PERF` with cluster size, permutation count, and time taken.

**New Flow** (sub-cluster size > 6): Marginal Value Ranking

Uses the **Microsoft Dynamics Marginal Value Ranking Method** (referenced in code: `learn.microsoft.com/en-us/dynamics365/commerce/optimal-combination-overlapping-discounts`):

1. For each promotion, compute standalone discount and items consumed
2. For each pair of promotions, compute overlap (shared items)
3. Calculate marginal value = `(total_discount - non_overlap_discount) / overlap_item_count`
4. Use highest marginal value as priority (negated for ascending sort)
5. Apply all promotions in marginal-value priority order

This reduces complexity from O(n!) to O(n^2) at the cost of not always finding the global optimum.

### Step 6: Priority-Based Application

`apply_promos_for_priority()` processes promotions in ascending `evaluate_priority` order:

1. Sort promotions by `evaluate_priority`
2. For each promotion:
   a. Get the family class instance via `PromoFamilyExecuter().get_instance(family_code)`
   b. Call `calculate_discount()` on the family class
   c. Update basket item quantities (reduce available qty)
   d. Accumulate discount info and promo suggestions

### Step 7: Response Building

`build_promo_evaluate_response()` aggregates all applied discounts into the final response:

- Iterates over all basket items, matching each to its promo application info via composite SKU key (`sku-#$@-sno-#$@-mrp-#$@-sp-#$@-item_id`)
- Computes per-item: discount_info, requisite_info, remaining_info
- Computes basket-level totals: `total_mrp`, `total_sp`, `discount`, `total_after_promos`
- Aggregates basket threshold promo info and special promo program info
- Filters promo suggestions (hides suggestions when all item quantities are consumed by item-level promos, but still shows basket threshold suggestions)
- Detects negative remaining quantities as errors (`NegativeRemainingQty`)

---

## V1 vs V2 Differences

| Aspect | V1 (`PromoLogic`) | V2 (`PromoLogicV2`) |
|--------|-------------------|---------------------|
| **Class** | `PromoLogic` | `PromoLogicV2` |
| **Pack item support** | Yes -- `update_quantity_for_pack()` and `update_promo_for_pack()` expand/collapse pack quantities | No pack item handling |
| **Empty basket guard** | Returns early with `empty_basket_promo_response()` | No early return for empty baskets |
| **Basket promo handling** | Collected during layer loop, applied separately after all layers via `apply_promos_for_basket_threshold()` | Applied inline within each layer: `apply_basket_promos_for_best_discount()` then `apply_basket_promos_for_priority()` |
| **Cluster types** | 3: priority, best_discount, basket | 4: priority, best_discount, basket, basket_best_discount |
| **Cross-layer handling** | Basket items rebuilt from response for basket threshold processing | `add_undiscounted_items_of_previous_layer_in_current_layer()` carries forward undiscounted items from previous layers |
| **Response building** | Once after item-level promos, then again after basket promos | Within each layer iteration, then used to rebuild basket items for next layer |
| **Best-discount items excluded in next layer** | Items with best-discount promos skip next layer via `generate_sku_to_basket_items_helper()` check | Same behavior |
| **Family `calculate_discount()` call** | `promo_version` defaults to 1 | `promo_version` passed as 2 |
| **Basket best discount (V2 only)** | N/A | Uses marginal value ranking to select the single best basket promo per cluster |
| **Duplicate handling in sub-clusters** | Uses `final_promo_header_ids_set` to deduplicate | Directly appends without deduplication set |

### V2 Cross-Layer Item Handling

V2's `add_undiscounted_items_of_previous_layer_in_current_layer()` merges previous-layer info:

- If a SKU was not in the current layer, carry forward its entire previous-layer promo application info
- If a SKU was in both layers, only carry forward best-discount entries from the previous layer (appended to discount_info)

### V1 Pack Item Processing

V1 handles "pack items" (items sold in packs, e.g., a 6-pack of drinks):

1. **Pre-processing** (`update_quantity_for_pack()`): Expands pack quantities to individual units (e.g., qty=2 of a 6-pack becomes qty=12 at unit price)
2. **Post-processing** (`update_promo_for_pack()`): Collapses results back to pack quantities, recalculating per-pack discount/final_price from the per-unit results. Preserves original response in `original_promo_response`.

---

## Promotion Family Types

All 8 families extend `PromoFamilyFactory` and implement three abstract methods: `create_promo()`, `generate_promo_short_code()`, and `calculate_discount()`.

### Family Summary Table

| Family | Code | Groups | Short Code Pattern | Description |
|--------|------|--------|-------------------|-------------|
| Easy | `e` | 1 | `BuyNofGforV{type}` | Buy N items from group, get discount. Only exact multiples qualify. |
| Easy+ | `p` | 1 | `BuyNofGforV{type}` | Buy >= N items from group, get discount on all. Extra items beyond min_qty get prorated discount. |
| Combo | `c` | 2+ | `BuyNiofGiforV{type}` | Buy required qty from each group, get discount across all groups. |
| Line Special | `l` | 1+ | `BuyNiofGiforVi{type}` | Each group has its own discount config (type, value). |
| Basket Threshold | `b` | 1 | `Buy$NofGforV{type}` | Basket total exceeds threshold, discount applied across basket items. |
| Basket Threshold + Target | `t` | 2 | `Buy$MofG1GetNofG2forV{type}` | Basket total for group 1 exceeds threshold, discount applied to group 2 items. |
| Requisite Groups + Target | `r` | 1-2 | `BuyNofG1GetNofG2forV{type}` | Buy N requisite items, get discount on M target items. 1-group variant: same items are both requisite and discounted. 2-group variant: separate requisite and target groups. |
| Evenly Distributed Multiline | `m` | 1 | (custom) | Discount is distributed evenly across ALL items in the promotion (both requisite and discounted). Supports equal or proportional split. |

### Discount Types

| Code | Name | Behavior |
|------|------|----------|
| `p` | %Off | Percentage discount off the price |
| `v` | $Off | Fixed value discount off the price |
| `f` | $ (Fixed) | Set item to a fixed price (discount = original - fixed) |

### Discount Type Strategies

| Code | Name | Behavior |
|------|------|----------|
| `a` | On All Items | Discount value applies to the entire qualifying group (prorated by price across items) |
| `e` | On Each Item | Discount value applies individually to each qualifying item |

### Discount Value On (Price Base)

| Code | Name | Applied To |
|------|------|-----------|
| `m` | On MRP | Maximum Retail Price |
| `s` | On Sale Price | Current selling price |
| `f` | On Final Price | Price after previous promotions |

### Item Selection Criteria

| Code | Name | Sort Order |
|------|------|-----------|
| `l` | Favor Retailer | Cheapest items first (minimizes retailer cost) |
| `lc` | Favor Customer | Cheapest items first for customer benefit |
| `m` | Most Expensive First | Most expensive items first |

### Family Detail: Easy (`e`)

**Source**: `easy_promo_family.py` (~338 lines)
**Class**: `EasyPromoFamily`

#### Purpose

The simplest promotion: buy exactly N items from a group, get a discount. Only exact multiples of the minimum quantity qualify — a 4th item beyond a "Buy 3" promotion gets no discount.

#### Algorithm

1. Validate exactly 1 promo group exists (`easy_promo_family.py:56`)
2. Extract promo configs (retailer, price base, selection criteria) via `get_promo_configs()`
3. Call `populate_promo_group_sku_with_basket_qty_ungrouped_item_info()` to match basket items to promo group nodes, ungroup items (1 entry per unit), sort by selection criteria, and build promo suggestions
4. If total item count < `qty_or_value_min`, return early with only suggestions
5. Collect all item prices into `to_be_consumed_prices` list for proration
6. Loop through items in batches of `discounted_group_sku_min_qty` (step size = min_qty):
   - Check `promo_application_times < max_application_limit` and sufficient items remain
   - Call `calculate_discounted_group_each_item_discount_strategy_value_by_item_prices()` to prorate discount across items in the current batch by their relative prices
   - For each item in batch: compute discount via `calculate_discount_amount_for_each_item()`, compute final_price = price - discount
   - Skip batch if discount <= 0 or final_price < 0 (with warning)
   - Build promo application info (discount_info) and track consumed qty
   - Process addon items if `item_type == 'WITH_ADDONS'`
7. **V2 behavior** (`promo_version == 2`): after processing complete batches, leftover items (below min_qty threshold) are emitted as undiscounted entries via `populate_sku_promo_application_info_undiscounted_items_current_layer()` (`easy_promo_family.py:156-179`). V1 breaks out of the loop entirely.

#### Addon Item Support

`apply_promo_on_addons()` (`easy_promo_family.py:236-337`) handles `WITH_ADDONS` item types:

1. Builds a synthetic addon group from the base item's `item_addons` list
2. Calls `addons_populate_promo_group_sku_with_basket_qty_ungrouped_item_info()` to match addon SKUs
3. Computes discount for each addon using the same promo discount config
4. Tracks cumulative discount/final_price per addon `item_id`
5. Populates `item_addons` with `discount_cummulative` and `final_price_cummulative`

This feature is noted as "Not 100% stable, added as urgent requirement" (`easy_promo_family.py:239`).

#### Legacy Method

`calculate_discount_by_picking_homegeneous_items()` (`easy_promo_family.py:184-234`) is an older implementation that operates on grouped (not ungrouped) items. It processes items per-SKU rather than per-unit and does not support price-based proration. This method is not called by the main evaluation pipeline.

#### Concrete Examples (from source comments)

| Promo | Items | Discount |
|-------|-------|----------|
| Buy 3 pens ($15 each) for $10 off | 4 pens | $10 → [$3.33, $3.33, $3.34] on 3 pens. 4th pen: no discount |
| Buy 3 pens for $10 off Each | 4 pens | $30 → [$10, $10, $10] on 3 pens |
| Buy 3 pens for 10% off | 4 pens | $4.50 → [$1.50, $1.50, $1.50] on 3 pens |
| Buy 3 pens for $10 fixed price | 4 pens | $35 → [$11.67, $11.67, $11.66] on 3 pens |

---

### Family Detail: Easy+ (`p`)

**Source**: `easy_plus_promo_family.py` (~123 lines)
**Class**: `EasyPlusPromoFamily`

#### Purpose

Like Easy, but applies discount to ALL items once the minimum quantity threshold is met — not just exact multiples. The 4th item beyond "Buy >= 3" still receives the prorated discount.

#### Algorithm

1. Validate exactly 1 promo group exists
2. Extract promo configs and populate ungrouped items (same as Easy)
3. Read `qty_or_value_max` from the group — this is an upper cap on how many items can receive the discount
4. If item count < min_qty, return early
5. Call `calculate_discounted_group_each_item_discount_strategy_value()` (equal distribution, not price-based) to compute per-item discount values
6. Loop through ALL items from 0 to `discounted_group_item_count` (not in batches):
   - If `qty_or_value_max` is set, stop at that limit (`easy_plus_promo_family.py:92`)
   - Apply discount cyclically: `discount_values[i % discounted_group_sku_min_qty]`
   - If discount <= 0 for any item, abort the entire promo (returns empty)
7. `promo_application_times` is always 1 — the promo applies once across all qualifying items

#### Key Differences from Easy

| Aspect | Easy | Easy+ |
|--------|------|-------|
| Discount application | Only exact multiples of min_qty | All items >= min_qty |
| Batching | Steps by min_qty | No batching, processes all items |
| `max_application_limit` | Controls how many batches | Not used (always 1 application) |
| `qty_or_value_max` | Not used | Caps maximum discounted items |
| Discount proration | Price-based (`_by_item_prices`) | Equal distribution |
| Error behavior | Skips batch, continues to next | Aborts entire promo on any error |

#### Concrete Examples (from source comments)

| Promo | Items | Discount |
|-------|-------|----------|
| Buy >= 3 pens ($15 each) for $10 off | 4 pens | $13.33 → [$3.33, $3.33, $3.34, $3.33] across all 4 |
| Buy >= 3 pens for $10 off Each | 4 pens | $40 → [$10, $10, $10, $10] |
| Buy >= 3 pens for 10% off | 4 pens | $6 → [$1.50, $1.50, $1.50, $1.50] |
| Buy >= 3 pens for $10 fixed price | 4 pens | $46.67 → [$11.67, $11.67, $11.66, $11.67] |

---

### Family Detail: Combo (`c`)

**Source**: `combo_promo_family.py` (~167 lines)
**Class**: `ComboPromoFamily`

#### Purpose

Multi-group promotion: the customer must buy the required quantity from each of 2+ groups simultaneously. The discount is shared across all items from all groups, prorated by price.

#### Algorithm

1. Validate at least 2 promo groups exist (`combo_promo_family.py:66`)
2. Extract promo configs and populate ungrouped items for each group
3. Track `discounted_groups_sku_min_qty_all_groups` = sum of all group min_qty values
4. Enter `while True` loop incrementing `promo_application_times`:
   - Break if `promo_application_times > max_application_limit`
   - **Price collection pass**: for each group, check that enough items exist for the current application (using `lower`/`upper` index range), collect item prices into `counsumed_item_prices` (note: typo in source)
   - If prices collected != total min_qty across all groups, break (insufficient items)
   - **Discount distribution**: call `calculate_discounted_group_each_item_discount_strategy_value_by_item_prices()` to prorate discount across all items proportionally by price
   - **Application pass**: for each item in each group, compute discount and final_price, skip on negative values, build promo application info

#### Key Design

- Items from ALL groups are treated as a single discount pool for proration
- The `lower`/`upper` index range advances by `min_qty * promo_application_times` for each group independently
- Breaking out of the inner loop on a negative discount/final_price stops only that group, but the loop continues to attempt the next application

#### Concrete Examples (from source comments)

| Promo | Items | Discount |
|-------|-------|----------|
| Buy 3 burgers ($10) + 2 cokes ($15) for $50 off | 3B + 2C | $50 → [$12.50, $12.50, $8.33, $8.33, $8.34] proportional to price |
| Buy 3B + 2C for 50% off | 3B + 2C | $30 → [$7.50, $7.50, $5, $5, $5] |
| Buy 3B + 2C for $50 fixed | 3B + 2C | $10 → [$2.50, $2.50, $1.67, $1.67, $1.66] |

---

### Family Detail: Line Special (`l`)

**Source**: `line_special_promo_family.py` (~171 lines)
**Class**: `LineSpecialPromoFamily`

#### Purpose

Each group within the promotion has its own independent discount configuration (type and value set at the **node level**, not the header level). This allows per-line-item discount customization within a single promotion.

#### Algorithm

1. Validate at least 1 promo group exists
2. Extract promo configs and populate ungrouped items for each group
3. Each group also tracks an `offset_for_negative_discount_or_final_price` counter (initialized to 0)
4. Enter `while True` loop incrementing `promo_application_times`:
   - **Pre-check**: verify sufficient items exist in all groups for this application
   - **Application pass**: for each group, loop through items using `lower`/`upper` range (adjusted by offset):
     - Read `discount_type` and `discount_value` from the **item-level** `discount_item_info` (set from node data during ungrouping) — NOT from the promo header
     - Compute discount via `calculate_discount_amount_for_each_item()`
     - **Offset mechanism**: if discount < 0 or final_price < 0, increment `offset_for_negative_discount_or_final_price` and `upper`, then `continue` to the next item (`line_special_promo_family.py:126-139`). This skips problematic items and tries the next one.
   - Uses a temporary dict `temp_sku_to_promo_application_info` per application round — only committed to the main dict if all groups produce sufficient items
5. Supports `RETAILER_APPLY_ZERO_DISCOUNT_PROMOS` — zero-discount entries are kept for specific retailers

#### Unique Feature: Per-Node Discount Override

Unlike all other families where discount type/value comes from `promo_header`, Line Special reads `discount_type` and `discount_value` from each node in the promo group. This enables promotions like: "Buy 2 Books at 10% off, 2 Pens at $10 fixed, 1 Pencil at $10 off" — all within a single promotion.

#### Concrete Examples (from source comments)

| Promo | Items | Discount |
|-------|-------|----------|
| Buy 2 books (10%) + 2 pens ($10 fixed) + 1 pencil ($10 off) | 2B($10) + 2P($15) + 1Pc($20) | $22 → [$1, $1, $5, $5, $10] |

Note: Pencil variant with $5 price would be skipped (final_price < 0 for $10 off), so variant with $20 price is chosen instead via the offset mechanism.

---

### Family Detail: Basket Threshold (`b`)

**Source**: `basket_threshold_promo_family.py` (~350 lines)
**Class**: `BasketThresholdPromoFamily`

#### Purpose

Discount applied when the basket total (sum of qualifying item prices) exceeds a monetary threshold. The discount is spread across all qualifying basket items.

#### Forced Overrides in `create_promo()`

- `discount_type_strategy` → forced to `ALL_ITEMS` (ignores "each" — `basket_threshold_promo_family.py:47`)
- `discounted_group_item_selection_criteria` → forced to `FAVOR_RETAILER` (cheapest items first — `basket_threshold_promo_family.py:49-50`)
- `layer` → forced to >= 100 if set below (`basket_threshold_promo_family.py:30-33`)

#### Algorithm

1. Validate exactly 1 promo group exists
2. Populate ungrouped items; interpret `qty_or_value_min` as the **monetary basket threshold** (not item count)
3. For value-off (`v`) discount type: threshold is `max(qty_or_value_min, discount_value)` to ensure basket total >= discount amount (`basket_threshold_promo_family.py:64-66`)
4. Collect `unused_ungrouped_items_not_part_of_basket_promo_groups` — items in basket but not in the promo group
5. Sum all qualifying item prices into `to_be_consumed_prices_total`
6. If `to_be_consumed_prices_total < threshold`, emit all items as undiscounted entries and return early
7. Enter application loop:
   - Prorate discount across all items by price via `calculate_discounted_group_each_item_discount_strategy_value_by_item_prices()`
   - For each item:
     - Check `exclude_item_from_basket_discount()` — skip already-discounted items unless `apply_on_discounted_items` is True
     - Compute discount with `max_discount` cap support — remaining cap decreases as items consume it (`basket_threshold_promo_family.py:226-228`)
     - Track prices for next application round (`to_be_consumed_prices_for_next_promo_application`)
   - After processing all items, update prices for next loop iteration
8. Emit unused basket items as undiscounted entries

#### Retailer-Specific Logic

- **MUJI**: Forces `apply_on_discounted_items = True` regardless of promo config (`basket_threshold_promo_family.py:91-92`). Also has special 2-decimal rounding, expected total adjustment on the last item, and a "Free tea" hack that short-circuits basket promo application for certain promo titles/IDs (`basket_threshold_promo_family.py:84-88`)

#### Helper Methods

- `exclude_item_from_basket_discount()` (`basket_threshold_promo_family.py:314-320`): Returns True if item should be skipped — either already has discount > 0, or is a requisite item for a %-off basket promo
- `populate_existing_already_applied_discounts()` (`basket_threshold_promo_family.py:322-349`): Re-emits items that don't qualify for the basket discount with their existing discount/price info (preserves previous-layer discounts in the output)

---

### Family Detail: Basket Threshold with Discounted Target (`t`)

**Source**: `basket_threshold_with_discounted_target_promo_family.py` (~270 lines)
**Class**: `BasketThresholdWithDiscountedTargetPromoFamily`

#### Purpose

Two-group basket threshold: the basket total for one group (requisite) must exceed a threshold, and the discount is applied to items in a different group (target).

#### Forced Overrides in `create_promo()`

Same as Basket Threshold: `layer >= 100` if misconfigured (`basket_threshold_with_discounted_target_promo_family.py:19-22`).

#### Algorithm

1. Validate exactly 2 promo groups exist
2. Identify which group is the target (matches `target_discounted_group_name` from promo header) and which is the requisite group
3. Sum requisite group item prices → `to_be_consumed_requisite_prices_total`
4. **If threshold NOT met** (`requisite_total < min_basket_amount`):
   - Remove shared items between target and requisite lists (using `.remove()`)
   - Emit all target items, requisite items, and unused items as undiscounted
   - Return early
5. **If threshold met**:
   - Calculate discount values via `calculate_discounted_group_each_item_discount_strategy_value()` (equal distribution)
   - Determine `max_application_limit_by_basket_threshold = requisite_total // threshold`
   - Loop through target items in batches of `discounted_group_sku_min_qty`:
     - Max batches = `min(max_application_limit, max_application_limit_by_basket_threshold)`
     - Apply discount to each target item
     - Remove consumed target items from requisite list (handles item overlap)
6. Process remaining requisite items as undiscounted entries
7. Populate `discounted_item_missing_info` — sets `free_item_missing: True` if promo could apply more times (100% off, remaining requisite items >= threshold) to hint to the customer

#### Key Design

- The `discounted_item_missing_info()` method (`basket_threshold_with_discounted_target_promo_family.py:225-234`) generates a customer-facing hint: when `discount_type == 'p'` and `discount_value == 100` (free items) and there are enough unused requisite items, it adds `free_item_missing: True` to promo suggestions
- Shared items between requisite and target groups are handled by `.remove()` calls with `ValueError` catch to avoid double-counting

---

### Family Detail: Requisite Groups with Discounted Target (`r`)

**Source**: `requisite_groups_with_discounted_target_promo_family.py` (~473 lines)
**Class**: `RequisiteGroupsWithDiscounteTargetPromoFamily` (note: typo in class name — missing 'd' in "Discounted")

#### Purpose

Buy N requisite items to qualify, get M target items discounted. Supports both 1-group (same items are requisite and target) and 2-group (separate requisite and target) variants.

#### 1-Group Variant Algorithm

1. Populate items from the single group
2. **Deep copy** the item list for both requisite and discounted roles
3. Split `qty_or_value_min` into:
   - `discounted_group_sku_min_qty` = `target_discounted_group_qty_min` (from promo header)
   - `requisite_group_sku_min_qty` = original min_qty - `target_discounted_group_qty_min`
4. **Edge case**: if `requisite_group_sku_min_qty == 0`, the promo is misconfigured — delegate to `EasyPromoFamily` via lazy import of `PromoFamilyExecuter` (`requisite_groups_with_discounted_target_promo_family.py:131-136`)
5. Calculate `promo_applied_times = min(item_count / (discounted_qty + requisite_qty), max_application_limit)`
6. **Selection criteria adjustments** for single-group:
   - **Favor Retailer** (`l`): discounted items (cheapest) come first; requisite items start at `requisite_items_start_index` = `promo_applied_times * discounted_group_sku_min_qty` (reorders requisite list)
   - **Favor Customer** (`lc`): items NOT participating in promo are the cheapest; the discounted/requisite items are rotated so unmatched items are at the beginning

#### 2-Group Variant Algorithm

1. Identify target group (matches `target_discounted_group_name`) and requisite group
2. **Reverse** the requisite group item list (`requisite_group_basket_qty_ungrouped_items.reverse()`) — this causes most-expensive requisite items to be consumed first (opposite of the sorting applied by the base class)

#### Shared Processing (Both Variants)

1. Compute per-item discount values via `calculate_discounted_group_each_item_discount_strategy_value()` (equal distribution)
2. Loop through discounted items in batches of `discounted_group_sku_min_qty`:
   - For each discounted item: compute discount, validate (skip on negative/zero with `RETAILER_APPLY_ZERO_DISCOUNT_PROMOS` check), remove consumed item from the requisite list
   - For each requisite item: emit as `requisite_info` with discount = 0, remove from discounted list
   - Track `promo_application_times_final` and `requisite_group_item_count_final`
3. Generate `discounted_item_missing_info` — same "free_item_missing" hint as Basket Threshold Target
4. **V2 behavior**: after processing complete batches, emit unprocessed discounted and requisite items as undiscounted entries (`requisite_groups_with_discounted_target_promo_family.py:317-364`). Includes a debug `print()` statement left in the code (`line 346`).

#### Item List Manipulation

Both variants use `.remove()` with `ValueError` catch to prevent double-counting when the same item appears in both requisite and discounted roles. For 1-group with "Favor Retailer" selection:
- Requisite list is reversed before `.remove()` and reversed back after — this ensures the most-expensive instance is removed from the requisite role (leaving the cheapest for discounting)

#### Concrete Examples (from source comments)

**1-Group** (Buy 5 mice, get 2 discounted):

| Promo | Items | Discount |
|-------|-------|----------|
| Buy 3, get 2 for $100 off | 5 mice ($150 each) | $100 → [$50, $50] on 2 cheapest mice. 3 mice at full price in requisite_info |
| Buy 3, get 2 for 100% off | 5 mice ($150 each) | $300 → [$150, $150] on 2 cheapest mice |
| Buy 3, get 2 for $100 fixed | 5 mice ($150 each) | $200 → [$100, $100] on 2 mice |

**2-Group** (Buy 3 keyboards, get 2 mice discounted):

| Promo | Items | Discount |
|-------|-------|----------|
| Buy 3 keyboards, get 2 mice for $100 off | 3K + 2M ($150/mouse) | $100 → [$50, $50] on mice. Keyboards in requisite_info |
| Buy 3 keyboards, get 2 mice free | 3K + 2M ($150/mouse) | $300 → [$150, $150] on mice |

---

### Family Detail: Evenly Distributed Multiline Discount (`m`)

**Source**: `evenly_distributed_multiline_discount_promo_family.py` (~599 lines)
**Class**: `EvenlyDistributedMultilineDiscountPromoFamily`

#### Purpose

The discount is split evenly across ALL items participating in the promotion (both requisite and discounted), rather than concentrating it on the discounted items alone. Supports two distribution strategies: equal split and proportional split.

#### Split Types

The split type is read from `promo_header.extra_data` JSON field under key `evenly_distributed_multiline_discount_split_type`:

| Split Type | Constant | Behavior |
|------------|----------|----------|
| Proportional (`p`) | `EVENLY_DISTRIBUTED_MULTILINE_DISCOUNT_PROPORTIONAL_SPLIT` | Discount prorated proportionally by each item's price relative to the set total. Default if not specified. |
| Equal (`e`) | `EVENLY_DISTRIBUTED_MULTILINE_DISCOUNT_EQUAL_SPLIT` | Each item in the set receives the same absolute discount amount. |

#### Algorithm

1. Validate 1 or 2 groups (same group handling as Requisite family — 1-group uses `target_discounted_group_qty_min` split, 2-group identifies target by name)
2. If `requisite_group_sku_min_qty == 0`, delegate to `EasyPromoFamily` (same edge case as Requisite)
3. Calculate `promo_applied_times` and apply selection-criteria adjustments (same as Requisite family)
4. Compute core discount values:
   - `discount_val_per_item = discount_value * discounted_group_mrp / 100` (for %-off)
   - `total_discount_per_set = discount_val_per_item * discounted_group_sku_min_qty`
   - `discount_per_item_in_set = total_discount_per_set / (discounted_min_qty + requisite_min_qty)` (for equal split)
5. Call `_calculate_per_set_prices()` to collect item prices organized by application set
6. **Per-set discount calculation** (for each set of items forming one promo application):
   - **Proportional split**:
     - `prorating_factor = sum_discounted_items_prices / sum_all_items_prices`
     - For %-off: each item gets `prorating_factor * 100` as its percentage discount
     - For $-off/fixed: each item gets `price * (discount_value / sum_all_items_prices)` with rounding adjustment on the last item
     - Result is split into discounted-group values and requisite-group values
   - **Equal split**:
     - For %-off: convert to effective percentage `(discount_per_item_in_set * 100) / discounted_group_mrp` and apply uniformly
     - For $-off: `discount_value / min_qty_total` per item
7. Apply discounted group items (same loop pattern as Requisite family, with temp dict)
8. Apply requisite group items:
   - **Proportional split**: uses the requisite-group discount values, with an adjustment on the final requisite item to ensure the total discount for the set matches exactly (`evenly_distributed_multiline_discount_promo_family.py:480-486`)
   - **Equal split**: uses `same_discount` (the computed per-item discount) for all requisite items
9. V2 leftover handling (same as Requisite)

#### Limitation

Currently works only for **non-variant cases** — all items in a group must have the same price. The source comments explicitly state: "Not Accepted Case: 1 group present with 2 variants."

#### Concrete Examples (from source comments)

| Promo | Items | Per-Item Discount |
|-------|-------|-------------------|
| Buy 3 oranges ($20) get 2 free | 5 oranges | Each gets $8 off (total $40 / 5 items). Final: $12 each |
| Buy 3 oranges ($20) get 2 at 50% off | 3 oranges | Discount = $20 → $6.67 off each of 3 items |
| Buy 2 apples ($30) get 1 orange ($20) for $6 off | 3 items | $2 off each ($6 / 3 items). Apple: $28, Orange: $18 |

---

## PromoFamilyFactory (Abstract Base Class)

**Source**: `promo_family_factory.py` (~416 lines)

The abstract base provides shared logic used by all 8 families:

### Abstract Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `create_promo` | `(promo: PromoSerializer) -> PromoSerializer` | Post-creation hook (set short code, override layer) |
| `generate_promo_short_code` | `(promo: PromoSerializer) -> str` | Generate human-readable short code |
| `calculate_discount` | `(promo, sku_dict, category_dict, promo_version=1) -> Dict` | Core discount calculation |

### Key Shared Methods

| Method | Lines | Purpose |
|--------|-------|---------|
| `get_promo_configs()` | 31-58 | Extracts retailer, discount base price key (`mrp`/`sp`/`final_price`), evaluation criteria, max_discount, and all promo header metadata into a config dict |
| `populate_promo_group_sku_with_basket_qty_ungrouped_item_info()` | 176-235 | Matches promo group nodes to basket items, "ungroups" items (1 entry per quantity unit), sorts by selection criteria, handles excluded items and `ALL` nodes |
| `get_promo_group_excluded_item_ids()` | 246-263 | Extracts excluded item IDs from nodes with `is_excluded=True` — supports both item-level and category-level exclusions |
| `calculate_discount_amount_for_each_item()` | 111-130 | Computes discount for a single item given price, discount type, and value. Handles %off (`price * value / 100`), $off (`value * weight`), fixed price (`price - value * weight`). Applies `max_discount` cap. All results rounded to 2 decimal places. |
| `calculate_discounted_group_each_item_discount_strategy_value()` | 66-81 | Computes per-item discount values using **equal distribution**. For "each" strategy: all items get full discount_value. For "all" + %off: all items get full percentage. For "all" + $off/fixed: `discount_value / min_qty` per item, with rounding remainder added to the last item. |
| `calculate_discounted_group_each_item_discount_strategy_value_by_item_prices()` | 83-109 | Computes per-item discount values using **price-based proration**. For "all" + $off/fixed: `price * (discount_value / sum_prices)` per item. Rounding remainder added to the last item. Used by Easy, Combo, and Basket Threshold families. |
| `populate_sku_promo_application_info()` | 313-356 | Builds the promo application result dict (discount_info or requisite_info) with all metadata including layer, discount_apply_type, extra_data, zero_discount_applied flag, and evaluate_criteria |
| `populate_sku_promo_application_info_undiscounted_items_current_layer()` | 359-377 | V2-only variant that emits items without discount application (empty `applied_promos` list). Used for leftover items in Easy, Requisite, and Evenly Distributed families when `promo_version == 2`. |
| `get_sku_key_for_current_promo_application_grouping()` | 60-64 | Generates composite key: `"sku-#$@-sno-#$@-mrp-#$@-sp-#$@-final_price-#$@-discount-#$@-item_id"` |
| `populate_promo_group_sku_with_basket_qty_grouped_item_info()` | 132-155 | Legacy variant of ungrouping that keeps items grouped by SKU (used only by legacy `calculate_discount_by_picking_homegeneous_items` methods) |
| `addons_populate_promo_group_sku_with_basket_qty_ungrouped_item_info()` | 157-174 | Addon-specific variant that filters ungrouped items to only include items whose `item_id` is in the addon group's `addon_item_ids` list |
| `get_unused_ungrouped_items_not_part_of_basket_promo_groups()` | 381-415 | Compares the promo group's ungrouped items against the full basket to find items NOT consumed by the promotion. Used by Basket Threshold families to emit all basket items in the response. |
| `get_sort_key()` | 237-244 | Returns the sort key for item ordering — uses the price field determined by `discount_value_on_item_key`. Contains commented-out logic for item_type-based sorting (REGULAR vs WITH_ADDONS). |

### Item Ungrouping

`populate_promo_group_sku_with_basket_qty_ungrouped_item_info()` converts grouped basket items into a flat list where each entry represents one unit. For example, a basket item with `qty=3` becomes 3 separate `deepcopy` entries each with `qty=1`. This enables per-unit discount calculation and respects sorting order (cheapest/most expensive first).

The method handles three node targeting strategies:
1. **`node_id == "ALL"`**: matches every SKU in the basket (excluding explicitly excluded items)
2. **Item nodes** (`node_type == "i"`): matches by exact SKU
3. **Category nodes**: resolves category to SKU list via `category_type_value_to_sku_list_dict`

Each ungrouped item copy receives `discount_type` and `discount_value` from its matching node (critical for Line Special family, ignored by others).

### Discount Calculation: Two Distribution Strategies

Two methods compute how a header-level discount value is distributed across items:

**Equal distribution** (`calculate_discounted_group_each_item_discount_strategy_value`):
- %off: each item gets the same percentage → `[10%, 10%, 10%]`
- $off (all): divide equally → `[$3.33, $3.33, $3.34]` (rounding remainder on last item)
- Used by: Easy+, Line Special, Basket Threshold Target, Requisite, Evenly Distributed (equal split)

**Price-based proration** (`calculate_discounted_group_each_item_discount_strategy_value_by_item_prices`):
- %off: each item gets the same percentage (same as equal)
- $off (all): distribute proportionally to price → `[price_i / sum_prices * discount_value]` for each item
- Used by: Easy, Combo, Basket Threshold

The choice between these two methods is hardcoded per-family, not configurable.

---

## Data Access Layer

### PromoDataManager

**Source**: `promo_data_manager.py`

Two classes handle data assembly:

**`Promo`**: Wraps a promotion with its header, groups, nodes, and special promo info. Supports dict-style access (`promo['promo_header']`) via `__getitem__`. Exposes:
- `promo_header` -- dict of PromoHeader fields
- `promo_groups` -- list of group dicts, each containing `promo_group_nodes` list
- `special_promo_info` -- special availability program info or `None`

**`PromoDataManager`**: Assembles flat database rows into hierarchical `Promo` objects:
- Input: flat list of records (each row = one node with joined group/header fields)
- Output: `promos` (list of `Promo` objects) and `node_id_to_promos` (reverse index for node-based lookup)
- Groups rows by `promo_header_ksuid`, then by `promo_group_ksuid`

### On-the-Fly Promotions

The raw SQL helper supports `get_on_the_fly_promotion()` which creates ephemeral `Promo` objects from `promo_override` data in the request. These are never persisted to the database and exist only for the duration of the evaluation. This enables dynamic promotions from external systems (e.g., loyalty platforms) without requiring pre-configuration in the promotions DB.

---

## Configuration Constants

**Source**: `config.py`

### Key Constants

| Constant | Value | Usage |
|----------|-------|-------|
| `SPLIT_KEY` | `'-#$@'` | Delimiter for composite dictionary keys throughout the engine |
| `PROMO_MAX_APPLICATION_LIMIT` | `100000` | Default max application limit |

### Retailer-Specific Overrides

| Constant | Retailers | Effect |
|----------|-----------|--------|
| `RETAILER_APPLY_ZERO_DISCOUNT_PROMOS` | DDF, MLSE-CA, KIKOMILANO, Grandiose | Keep zero-discount promo entries in response |
| `RETAILER_APPLY_ONE_BASKET_PROMO_PER_LAYER` | DDF | Limit basket promos to one per layer |

### Legacy Retailer Codes

Hardcoded retailer constants exist for backwards compatibility: Eroski (`"1"`), Connexus (`"2"`), Muji (`"3"`), Flying Tiger (`"4"`). New clients should not be added to these constants.

---

## Response Format

The evaluate endpoint returns a response with this structure:

```json
{
  "status": true,
  "status_msg": null,
  "customer_id": "...",
  "is_guest_user": false,
  "basket": {
    "id": "basket-123",
    "total_mrp": "100.000",
    "total_sp": "90.000",
    "discount": "15.000",
    "total_after_promos": "75.000",
    "special_promos_info": {
      "discount": "5.000",
      "applied_programs": [...]
    },
    "basket_threshold_promos": {
      "discount_already_deducted_from_basket_total": true,
      "discount": "10.000",
      "applied_promos": [...]
    },
    "items": [
      {
        "sku": "SKU001",
        "barcode": "123456",
        "mrp": "50.000",
        "sp": "45.000",
        "qty": "2",
        "uom": "EA",
        "weight": 1,
        "categories": [...],
        "item_type": "REGULAR",
        "item_id": "i",
        "item_addons": null,
        "discount_info": [
          {
            "consumed_qty": 1,
            "discount": "5.000",
            "final_price": "40.000",
            "applied_promos": [
              {
                "promo_id": "...",
                "promo_id_retailer": "...",
                "promo_family": "e",
                "promo_title": "...",
                "discount": "5.000",
                "final_price": "40.000",
                "priority": 1,
                "promo_application_times": 1,
                "promo_max_application_limit": 1,
                "special_promo_info": null,
                "evaluate_criteria": "b",
                "item_source": "discount_info"
              }
            ],
            "applied_promos_prev_layers": []
          }
        ],
        "requisite_info": [],
        "remaining_info": {
          "remaining_qty": 1,
          "promo_suggestions": [...]
        }
      }
    ]
  }
}
```

### Per-Item Breakdown

Each item in the response contains:

| Field | Description |
|-------|-------------|
| `discount_info` | List of discount application entries (items that received a discount) |
| `requisite_info` | List of requisite entries (items that qualified a promotion but received no discount themselves) |
| `remaining_info` | Remaining quantity not consumed by any promotion, plus promo suggestions for the item |
| `item_addons` | Addon item discount info (for `WITH_ADDONS` items) |

---

## Notable Patterns

1. **Composite String Keys**: The engine uses `SPLIT_KEY` (`-#$@`) as a delimiter throughout to create composite dictionary keys (e.g., `"sku-#$@-sno-#$@-mrp-#$@-sp-#$@-item_id"`). This avoids nested dicts but makes key parsing fragile.

2. **Deep Copy Heavy**: Both engines make extensive use of `copy.deepcopy()` for the best-discount algorithm, as each permutation/ranking attempt needs a clean state. This is a significant performance cost.

3. **Singleton Pattern for Families**: `PromoFamilyExecuter` creates singleton instances of each family class in `PROMO_FAMILY_FACTORY_CLASS_MAP`, instantiated at module load time.

4. **Decimal Arithmetic**: All monetary calculations use Python's `Decimal` type with 3-decimal-place rounding via a custom `round_custom()` function (retailer-specific rounding rules).

5. **No Foreign Keys**: Consistent with the service-wide pattern documented in [Data Models](./data-models.md), all inter-model references use string KSUID fields rather than Django ForeignKey constraints.

6. **Performance Logging**: The best-discount algorithm logs `BEST_DISCOUNT_PERF` at WARNING level with retailer, basket ID, cluster size, permutation count, and time taken.

7. **Error Tolerance**: Negative remaining quantities are logged as errors but don't raise exceptions -- the response is returned with `status: false` and an error message. Similarly, individual discount calculations that produce negative values (price < discount) are skipped with warnings.

8. **Error Handling Varies by Family**: Easy aborts only the current batch on negative discount (continues to next batch). Easy+ aborts the entire promo. Combo and Line Special skip the individual item but may abort the application round. Basket Threshold stops the application loop entirely. This inconsistency means different families have different fault tolerance for edge-case items.

9. **Offset Mechanism (Line Special only)**: Line Special uses an `offset_for_negative_discount_or_final_price` counter per group to skip problematic items and try the next one. No other family has this tolerance — they either skip or abort.

10. **Cross-Family Delegation**: Both Requisite and Evenly Distributed families delegate to Easy when `requisite_group_sku_min_qty == 0` (misconfigured promo). This uses a lazy import of `PromoFamilyExecuter` to avoid circular imports and mutates the promo's `family` field in-place.

11. **Free Item Missing Hint**: Both Basket Threshold Target and Requisite families generate `free_item_missing: True` suggestions when the promo is 100% off and there are enough remaining requisite items for another application. This is a customer-facing feature to encourage adding more items.

12. **Debug Print Statements**: Both Requisite (`requisite_groups_with_discounted_target_promo_family.py:346`) and Evenly Distributed (`evenly_distributed_multiline_discount_promo_family.py:526, 564`) contain `print()` statements (prefixed "dora ->") that appear to be debugging artifacts left in production code.

13. **Weight-Based Discounts**: `calculate_discount_amount_for_each_item()` multiplies $off and fixed-price discount values by the item's `weight` field. This enables weight-based pricing (e.g., produce sold by weight). The weight parameter defaults to `"1"` if None.

---

## Math Utilities

**Source**: `math_util.py` (~21 lines)

Two rounding functions are provided:

### `round_custom(amount, places=2, retailer=None)`

The **active** rounding function used throughout the engine. Uses Python's `Decimal.quantize()` with `ROUND_HALF_UP` strategy:

| `places` | Quantize Pattern | Example |
|-----------|-----------------|---------|
| 3 | `Decimal('.001')` | `3.5655` → `3.566` |
| 2 (default) | `Decimal('.01')` | `3.565` → `3.57` |

The `retailer` parameter is accepted but **currently unused** — all retailers get the same ROUND_HALF_UP behavior. Previous code comments suggest retailer-specific rounding was once planned or partially implemented.

**Important**: When `amount` is a Python `float` converted to `Decimal`, floating-point representation issues can affect results. The source comments note: `round_custom(Decimal('3.565'), 2)` → `3.57` but `round_custom(Decimal(3.565), 2)` → `3.56` due to float imprecision.

### `round_custom2(amount, places=2, retailer=None)`

An **alternative** rounding function (not imported by any family class). Uses `math.floor`/`math.ceil` for retailer-specific behavior:

| Retailer | Rounding | Method |
|----------|----------|--------|
| Eroski | Round half-down | `math.floor(amount * 10^places) / 10^places` |
| All others | Round half-up (ceiling) | `math.ceil(amount * 10^places) / 10^places` |

Both results are quantized to 2 decimal places. This function has known floating-point issues (noted in source: "Have some floating point issues") and appears to be a legacy/experimental implementation.

### Usage Across Families

All 8 family classes import `round_custom` from `math_util` and use it for:
- Computing `final_price = price - discount` (3 decimal places)
- MUJI-specific 2-decimal rounding in Basket Threshold
- Basket Threshold max_discount cap calculations

The MUJI retailer has special rounding at 2 decimal places in `basket_threshold_promo_family.py:112-113`, while most other calculations use 3 decimal places.

---

## Dependencies

| Module | Depends On |
|--------|-----------|
| `promo_logic.py` / `promo_logic_v2.py` | `promo_db_helper_raw_sql`, `PromoFamilyExecuter`, `Promo`, `config`, `math_util`, `networkx`, `itertools`, `copy` |
| `promo_family_factory.py` | `config`, `math_util` |
| `promo_family_executer.py` | All 8 family classes, `PromoFamilyFactory` |
| `promo_db_helper_raw_sql.py` | `PromoDataManager`, `django.db.connection`, `config` |
| `promo_db_helper.py` | Django ORM models, `PromoDataManager`, `config` |

External dependency: **NetworkX** is used exclusively for the graph-based sub-clustering in the best-discount algorithm.
