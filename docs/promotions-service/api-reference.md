# Promotions Service — API Reference

> Last updated by: Task T05 — API, Utilities & Tests Summary

## Overview

The Promotions Service exposes a REST API built with Django REST Framework's `APIView` classes. All endpoints are unauthenticated — the service relies on network-level security (internal Docker network `mpay_net`). All views are defined in `service/mpromo/views.py` (697 lines) with URL routing in `service/promotions/urls.py`.

**Base URL:** `/api/1.0/promotions/`

## Endpoint Summary

| Method | Path | View Class | Description |
|--------|------|------------|-------------|
| POST | `/api/1.0/promotions/` | `PromotionCreateAndList` | Create a single promotion |
| POST | `/api/1.0/promotions/bulk_transaction/` | `PromotionBulkTransaction` | Bulk create/update/delete promotions |
| POST | `/api/1.0/promotions/transaction/` | `PromotionBulkTransaction` | **Deprecated** — alias for `bulk_transaction` |
| POST | `/api/1.0/promotions/evaluate/` | `PromotionEvaluate` | Evaluate promotions for a basket |
| GET | `/api/1.0/promotions/retailer/<retailer>/` | `PromotionsByRetailer` | List all promotions for a retailer |
| GET | `/api/1.0/promotions/store/<store_id>/` | `PromotionsByStore` | List promotions for a store |
| DELETE | `/api/1.0/promotions/store/<store_id>/` | `PromotionsByStore` | Hard-delete all promotions for a store |
| POST | `/api/1.0/promotions/bulk/<store_id>/` | `PromotionsBulkByStore` | Filter promotions by store + retailer IDs |
| POST | `/api/1.0/promotions/suggestions/` | `PromotionSuggestionListByNodes` | Get promotion suggestions by product nodes |
| POST | `/api/1.0/promotions/item_recommendations/` | `PromoBasedItemRecommendations` | Get item recommendations based on promotions |
| GET | `/api/1.0/promotions/<store_id>/<retailer_id>` | `PromotionDetailByStore` | Get a specific promotion by store + retailer ID |
| DELETE | `/api/1.0/promotions/<store_id>/<retailer_id>` | `PromotionDetailByStore` | Soft-delete a specific promotion |
| GET | `/api/1.0/p/<store_id>/<retailer_id>` | `PromotionDetailByStore` | **Deprecated** — short alias |
| — | `/health_check/` | `health_check` | Django health check |
| — | `/admin/` | Django Admin | Admin interface |

---

## Data Shapes

### Promotion Response (PromoSerializer)

**Source:** `service/mpromo/serializers.py:52-76`

The full promotion response includes the header, its groups (with nested nodes), active store IDs, and special promo info. Serialized via `PromoSerializer` which extends `PromoHeader` with computed fields.

```json
{
  "ksuid": "string (read-only)",
  "promo_id_retailer": "string",
  "retailer": "string",
  "family": "char (e|p|r|c|l|b|t|m)",
  "evaluate_criteria": "char (p|b)",
  "evaluate_priority": "integer",
  "promo_short_code": "string (optional)",
  "title": "string",
  "description": "string",
  "discount_type": "char (p|v|f)",
  "discount_type_strategy": "char (a|e)",
  "discount_value": "decimal",
  "discount_value_on": "char (m|s|f)",
  "max_application_limit": "integer",
  "is_active": "boolean",
  "active_days": "string",
  "start_date_time": "datetime",
  "end_date_time": "datetime",
  "is_happy_hour": "boolean",
  "discounted_group_item_selection_criteria": "string",
  "requisite_groups_item_selection_criteria": "string",
  "target_discounted_group_name": "string",
  "target_discounted_group_qty_min": "integer",
  "layer": "integer",
  "discount_apply_type": "char (b|c|t|p)",
  "availability": "char (a|s)",
  "apply_on_discounted_items": "boolean",
  "promo_groups": [
    {
      "ksuid": "string",
      "name": "string",
      "promo_header_id": "string",
      "qty_or_value_min": "decimal",
      "qty_or_value_max": "decimal",
      "promo_group_nodes": [
        {
          "ksuid": "string",
          "node_id": "string",
          "node_type": "string (default: 'i')",
          "promo_group_id": "string",
          "is_excluded": "boolean",
          "discount_type": "string (nullable)",
          "discount_value": "decimal (nullable)"
        }
      ]
    }
  ],
  "stores": ["string (store_id)", "..."],
  "special_promo_info": [
    {
      "group_qualifier_id": "string",
      "description": "string"
    }
  ]
}
```

**Note:** The `promo_groups` field returns `null` (not an empty list) if the promotion has no active stores (`serializers.py:72-76`). The `stores` field only returns non-soft-deleted store IDs.

### Evaluate Request (PromoEvaluateRequestSerializer)

**Source:** `service/mpromo/serializers.py:111-116`

```json
{
  "customer_id": "string",
  "store_id": "string",
  "version": "string (optional, default: 'v1')",
  "special_promos": [
    {
      "program_name": "string",
      "group_qualifiers": [
        {
          "input": "string",
          "qualifier_ids": ["string"]
        }
      ]
    }
  ],
  "basket": {
    "id": "string",
    "pre_tax": "decimal",
    "tax": "decimal",
    "total": "decimal",
    "items": [
      {
        "id": "string",
        "barcode": "string",
        "sku": "string",
        "mrp": "decimal (max_digits=8, decimal_places=2)",
        "sp": "decimal (max_digits=8, decimal_places=2)",
        "uom": "string",
        "qty_or_weight": "decimal",
        "units": "integer",
        "catgories": [
          {
            "name": "string",
            "value": "string"
          }
        ]
      }
    ]
  }
}
```

**Note:** The field `catgories` (missing 'e') is a known typo in `serializers.py:101`. See [Open Questions](../questions.md#promotions-service--data-models-from-t02).

---

## Endpoint Details

### POST `/api/1.0/promotions/` — Create Single Promotion

**View:** `PromotionCreateAndList` (`views.py:605-616`)

Creates a single promotion with all related records (groups, nodes, stores, special promo info) in a single atomic transaction.

**Request Body:** A single promotion object matching the `PromoHeader` fields, plus nested `promo_groups` (with `promo_group_nodes`), `stores` array, and optional `special_promo_info`.

```json
{
  "retailer": "string (required)",
  "promo_id_retailer": "string (required)",
  "family": "char (required)",
  "promo_groups": [
    {
      "name": "string",
      "qty_or_value_min": "decimal",
      "qty_or_value_max": "decimal",
      "promo_group_nodes": [
        {
          "node_id": "string",
          "node_type": "string (default: 'i')"
        }
      ]
    }
  ],
  "stores": ["store_id_1", "store_id_2"],
  "special_promo_info": [
    {
      "group_qualifier_id": "string",
      "description": "string"
    }
  ]
}
```

**Response:**
- `200 OK` — Full promotion object via `PromoSerializer`
- `400 Bad Request` — Validation error string

**Internal flow:** Uses `PromotionBatchOperation.process()` with a single-item list. See [Batch Operation Helper](#batch-operation-helper-promotionbatchoperation).

---

### POST `/api/1.0/promotions/bulk_transaction/` — Bulk Transaction

**View:** `PromotionBulkTransaction` (`views.py:242-462`)

The primary endpoint for managing promotions in bulk. Supports 6 action types within a single atomic transaction. This is the main endpoint used by the monolith for syncing promotion data.

**Request Body:**

```json
{
  "retailer": "string (required)",
  "ops": [
    {
      "action": "c|d|u|n|g|s",
      "promo_id_retailer": "string",
      "store_id": "string (for d/u/n/g/s actions)",
      "stores": ["string"] ,
      ...
    }
  ]
}
```

**Action Types:**

#### `"c"` — Create (`views.py:445-455`)

Creates a new promotion. Create operations are **batched** — they are queued and bulk-created when the next non-create action is encountered or at the end of the transaction. This is a performance optimization for bulk ingestion.

```json
{
  "action": "c",
  "promo_id_retailer": "string",
  "stores": ["store_1", "store_2"],
  "promo_groups": [...],
  "special_promo_info": [...],
  ...PromoHeader fields...
}
```

**Validation:** Checks for duplicate `promo_id_retailer` within the same store in the current batch and against existing records.

#### `"d"` — Delete (Soft-Delete) (`views.py:299-319`)

Soft-deletes a promotion for a specific store by setting `PromoHeaderStore.soft_delete = True`. Does not remove the promotion data — it remains available for other stores.

```json
{
  "action": "d",
  "promo_id_retailer": "string",
  "store_id": "string"
}
```

**Behavior:** Flushes the create queue first (`commit_transaction()`), then performs soft-delete. If the update count is not exactly 1, logs an error but does **not** raise an exception — the transaction continues. Failed deletes are silently logged (`views.py:318-319`).

#### `"u"` — Update Header (`views.py:421-442`)

Updates promotion header fields. Only fields in the `UPDATE_FIELDS` whitelist are updated (28 fields).

```json
{
  "action": "u",
  "promo_id_retailer": "string",
  "store_id": "string",
  "header": {
    ...updatable PromoHeader fields...,
    "promo_groups": [
      {
        "name": "group_name",
        "qty_or_value_min": "decimal",
        "qty_or_value_max": "decimal"
      }
    ]
  }
}
```

**Updatable fields (28):** `family`, `evaluate_criteria`, `evaluate_priority`, `promo_short_code`, `title`, `description`, `discount_type`, `discount_type_strategy`, `discount_value`, `discount_value_on`, `max_application_limit`, `is_active`, `active_days`, `start_date_time`, `end_date_time`, `is_happy_hour`, `discounted_group_item_selection_criteria`, `requisite_groups_item_selection_criteria`, `target_discounted_group_name`, `target_discounted_group_qty_min`, `layer`, `discount_apply_type`, `availability`, `apply_on_discounted_items`.

**Group update (DEP-4025):** If `header.promo_groups` is provided, existing groups are matched by `name` and their `qty_or_value_min`/`qty_or_value_max` fields are bulk-updated via `update_group()` (`views.py:263-288`).

#### `"n"` — Update Nodes (Single Group) (`views.py:323-342`)

Replaces all nodes in a promotion's **single** group. Deletes existing nodes and bulk-creates new ones.

```json
{
  "action": "n",
  "promo_id_retailer": "string",
  "store_id": "string",
  "nodes": [
    {
      "node_id": "string",
      "node_type": "string (default: 'i')"
    }
  ]
}
```

**Limitation:** Assumes the promotion has exactly one group (`views.py:331` uses `.get()` on `PromoGroup`, which will raise `MultipleObjectsReturned` if the promotion has multiple groups). For multi-group node updates, use the `"g"` action.

#### `"g"` — Update Groups with Nodes (`views.py:345-385`)

Replaces all groups and nodes for a promotion. Deletes existing groups/nodes, then creates new groups with their nodes.

```json
{
  "action": "g",
  "promo_id_retailer": "string",
  "store_id": "string",
  "group_to_nodes": {
    "group_name_1": {
      "qty_or_value_min": "decimal",
      "qty_or_value_max": "decimal",
      "nodes": [
        {
          "node_id": "string",
          "node_type": "string (default: 'i')",
          "is_excluded": "boolean",
          "discount_type": "string",
          "discount_value": "decimal"
        }
      ]
    },
    "group_name_2": { ... }
  }
}
```

#### `"s"` — Update Special Promo Info (`views.py:388-418`)

Creates or updates `PromoHeaderSpecialPromo` records. Uses upsert logic: tries to `.get()` an existing record; if found, updates `group_qualifier_id` and `description`; if not found, creates a new one.

```json
{
  "action": "s",
  "promo_id_retailer": "string",
  "store_id": "string",
  "special_promo_info": [
    {
      "group_qualifier_id": "string",
      "description": "string"
    }
  ]
}
```

**Response:**
- `204 No Content` — All operations succeeded
- `400 Bad Request` — `SaveModelException` error message

**Transaction behavior:** The entire `ops` list is processed within a single `transaction.atomic()` block. If any operation raises a `SaveModelException`, the entire transaction is rolled back.

---

### POST `/api/1.0/promotions/evaluate/` — Evaluate Basket

**View:** `PromotionEvaluate` (`views.py:653-670`)

The core endpoint. Evaluates all applicable promotions for a basket and returns the optimal discount combination.

**V1/V2 Selection (`views.py:659`):** The evaluation engine version is selected **per-request** via the `version` field in the request body:
```python
if post_data.get("version", "v1").lower() == "v2":
    resp = PromoLogicV2().evaluate(post_data)
else:
    resp = PromoLogic().evaluate(post_data)
```

Default is V1. Any value other than `"v2"` (case-insensitive) falls through to V1.

**Request Body:** See [Evaluate Request](#evaluate-request-promoevaluaterequestserializer) above.

**Response:**
- `200 OK` — Evaluation result with `status: true`
- `400 Bad Request` — Either `status: false` from the engine, or exception message

**Response structure** (from evaluation engine):
```json
{
  "status": true,
  "status_msg": "string",
  "basket": { ... },
  "applied_promotions": [ ... ],
  "discount_total": "decimal"
}
```

See [Business Logic](./business-logic.md) for details on the V1/V2 evaluation algorithms.

---

### GET `/api/1.0/promotions/retailer/<retailer>/` — List by Retailer

**View:** `PromotionsByRetailer` (`views.py:465-472`)

Returns all promotions (active and inactive) for a given retailer ID.

**Parameters:**
- `retailer` (path) — Retailer ID string

**Response:** `200 OK` — Array of full promotion objects

**Note:** No filtering by `is_active`, `soft_delete`, or date range. Returns all promotions for the retailer regardless of status.

---

### GET `/api/1.0/promotions/store/<store_id>/` — List by Store

**View:** `PromotionsByStore.get()` (`views.py:487-493`)

Returns all non-soft-deleted promotions for a store.

**Parameters:**
- `store_id` (path) — Store ID string

**Response:** `200 OK` — Array of full promotion objects

**Filter:** Only returns promotions with at least one `PromoHeaderStore` record where `store=store_id` and `soft_delete=False`.

---

### DELETE `/api/1.0/promotions/store/<store_id>/` — Hard-Delete All by Store

**View:** `PromotionsByStore.delete()` (`views.py:495-526`)

**Hard-deletes** all promotions for a store, including all related data (groups, nodes, special promos). Uses raw SQL for cascade deletion.

**Parameters:**
- `store_id` (path) — Store ID string (hyphens are stripped: `views.py:501`)

**Response:** `200 OK`
```json
{
  "message": "Deleted",
  "data": {
    "cnt_store": 0,
    "cnt_header": 0,
    "cnt_special_promo": 0,
    "cnt_group": 0,
    "cnt_node": 0
  }
}
```

**Implementation details:**
- Runs within `@transaction.atomic` decorator
- Uses **raw SQL** queries for deletion (`views.py:548-558`), bypassing the Django ORM
- Deletion order: header stores -> headers -> special promos -> groups -> nodes (groups and nodes only if groups exist)
- The `sql_string_from_ids()` helper converts Python tuples to SQL `IN` clause format, handling the single-element tuple trailing comma issue (`views.py:598-602`)
- **Warning:** Does not check if a promotion is shared with other stores before deleting. The `@todo` comment at `views.py:497-498` notes this behavior is "not in use by main app"

---

### POST `/api/1.0/promotions/bulk/<store_id>/` — Bulk Filter by Store

**View:** `PromotionsBulkByStore` (`views.py:475-483`)

Returns promotions for a store filtered by a list of retailer promo IDs.

**Parameters:**
- `store_id` (path) — Store ID string

**Request Body:**
```json
{
  "retailer_ids": ["promo_id_retailer_1", "promo_id_retailer_2"]
}
```

**Response:** `200 OK` — Array of matching promotion objects

---

### POST `/api/1.0/promotions/suggestions/` — Promotion Suggestions

**View:** `PromotionSuggestionListByNodes` (`views.py:672-683`)

Returns promotion suggestions based on product node IDs. Delegates to `PromoLogic().list_promo_by_nodes()`.

**Request Body:** Passed directly to `PromoLogic().list_promo_by_nodes(post_data)`.

**Response:** `200 OK` — Suggestion results from the engine
**Error:** `400 Bad Request` — Exception message string

**Note:** Always uses V1 engine (`PromoLogic`), regardless of any version field.

---

### POST `/api/1.0/promotions/item_recommendations/` — Item Recommendations

**View:** `PromoBasedItemRecommendations` (`views.py:685-696`)

Returns item recommendations based on active promotions. Delegates to `PromoLogic().item_recommendation()`.

**Request Body:** Passed directly to `PromoLogic().item_recommendation(post_data)`.

**Response:** `200 OK` — Recommendation results from the engine
**Error:** `400 Bad Request` — Exception message string

**Note:** Always uses V1 engine (`PromoLogic`), regardless of any version field.

---

### GET `/api/1.0/promotions/<store_id>/<retailer_id>` — Get Promotion Detail

**View:** `PromotionDetailByStore.get()` (`views.py:632-637`)

Returns a single promotion by store ID and retailer promo ID.

**Parameters:**
- `store_id` (path) — Store ID string
- `retailer_id` (path) — `promo_id_retailer` value

**Response:**
- `200 OK` — Full promotion object
- `404 Not Found` — `"Promotion Not Found"` string

---

### DELETE `/api/1.0/promotions/<store_id>/<retailer_id>` — Soft-Delete Promotion

**View:** `PromotionDetailByStore.delete()` (`views.py:639-650`)

Soft-deletes a specific promotion for a store by setting `PromoHeaderStore.soft_delete = True`. The promotion data is preserved.

**Parameters:**
- `store_id` (path) — Store ID string
- `retailer_id` (path) — `promo_id_retailer` value

**Response:**
- `204 No Content` — Successfully soft-deleted
- `404 Not Found` — `"Promotion Not Found"` string

---

## Internal Components

### Batch Operation Helper (PromotionBatchOperation)

**Source:** `views.py:65-240`

`PromotionBatchOperation` is a helper class that handles bulk creation of promotions with all related records. It is used by both `PromotionCreateAndList` (single create) and `PromotionBulkTransaction` (batch create action `"c"`).

**Constructor:**
- `retailer_id` — The retailer ID for all promotions in this batch

**Processing pipeline (`process()`, `views.py:110-189`):**

1. **Validation** — Checks max 10,000 operations per transaction (`views.py:111-112`)
2. **Registration** — Iterates operations, extracting and registering groups, nodes, stores, and special promo info into separate maps
3. **Bulk create headers** — `PromoHeader.objects.bulk_create()` (`views.py:146`)
4. **Map headers** — Queries back to build `promo_id_retailer -> ksuid` map (`views.py:150-153`)
5. **Bulk create groups** — Creates groups linked to header KSUIDs (`views.py:160`)
6. **Map groups** — Queries back to build `group_name -> ksuid` map (`views.py:164-165`)
7. **Bulk create nodes** — Creates nodes linked to group KSUIDs (`views.py:172`)
8. **Bulk create stores** — Creates store associations (`views.py:180`)
9. **Bulk create special promos** — Creates special promo records (`views.py:188`)

**Key design:** Uses `register_*()` methods to accumulate records into queues, then bulk-creates in correct dependency order (headers -> groups -> nodes -> stores -> special promos). Each entity gets a KSUID assigned at registration time.

### SaveModelException

**Source:** `views.py:61-62`

Custom exception used throughout the views layer for validation and business logic errors. Caught at the view level and returned as `400 Bad Request`.

### UPDATE_FIELDS Whitelist

**Source:** `views.py:29-58`

Controls which `PromoHeader` fields can be modified via the `"u"` (update) action. Contains 28 fields. Notably excludes: `ksuid`, `promo_id_retailer`, `retailer`, `created_at`, `updated_at`.

---

## Serializers

**Source:** `service/mpromo/serializers.py`

### Model Serializers

| Serializer | Model | Fields | Notes |
|------------|-------|--------|-------|
| `PromoHeaderSerializer` | `PromoHeader` | `__all__` | `ksuid` read-only, `promo_short_code` optional |
| `PromoGroupSerializer` | `PromoGroup` | Explicit list + `promo_group_nodes` | Computed nested nodes via `get_promo_group_nodes()` |
| `PromoGroupNodeSerializer` | `PromoGroupNode` | `__all__` | `ksuid` read-only |
| `PromoHeaderStoreSerializer` | `PromoHeaderStore` | `__all__` | `ksuid` read-only |
| `PromoHeaderSpecialPromoSerializer` | `PromoHeaderSpecialPromo` | `group_qualifier_id`, `description` | No `ksuid` or `promo_header_id` in output |
| `PromoSerializer` | `PromoHeader` | `__all__` + computed | Full promo with nested groups, stores, special info |

### Request Serializers (Evaluate Endpoint)

| Serializer | Purpose |
|------------|---------|
| `PromoEvaluateRequestSerializer` | Top-level evaluate request |
| `Basket` | Basket container with items |
| `BasketItem` | Individual basket item |
| `BasketItemCategory` | Item category (name/value pair) |
| `SpecialPromoProgram` | Special promo program definition |
| `SpecialPromoProgramGroupQualifier` | Group qualifier within a special promo |

**Note:** The request serializers (`PromoEvaluateRequestSerializer`, `Basket`, `BasketItem`, etc.) are defined in `serializers.py` but are **not actually used** by the `PromotionEvaluate` view — it passes `request.data` directly to the evaluation engine without serializer validation (`views.py:656`). These serializers appear to exist for documentation or future validation purposes.

---

## Deletion Patterns

The service uses three distinct deletion approaches:

### 1. Soft-Delete (Preferred)

Used by `PromotionDetailByStore.delete()` and bulk transaction `"d"` action. Sets `PromoHeaderStore.soft_delete = True`. The promotion data remains intact for other stores. Soft-deleted records are later cleaned up by the `promo_delete_job()` background task or the `delete_promos` management command.

### 2. Raw SQL Hard-Delete

Used by `PromotionsByStore.delete()`. Executes raw SQL `DELETE` statements via `connection.cursor()` for performance. Deletes all related records (headers, groups, nodes, stores, special promos) in a single atomic transaction. Does **not** check for shared promotions across stores.

### 3. ORM Cascade Delete

Used by `promo_delete_job()` (background task) and `delete_promos` management command. Uses Django ORM `.delete()` calls in correct dependency order (children first: nodes -> groups -> stores -> special promos -> headers).

---

## Error Handling

All view classes follow this pattern:
- `SaveModelException` is caught and returned as `400 Bad Request` with the error message as a string
- For the evaluate endpoint, any `Exception` is caught, logged with full traceback (`exc_info=True`), and returned as `400 Bad Request` with `{"status": False, "status_msg": str(error)}`
- No authentication or permission errors (no auth configured)
- The bulk transaction `"d"` (delete) action catches and silently logs errors rather than failing the transaction (`views.py:318-319`)

---

## Deprecated Endpoints

| Deprecated Path | Use Instead | Notes |
|-----------------|-------------|-------|
| `/api/1.0/promotions/transaction/` | `/api/1.0/promotions/bulk_transaction/` | Same view, endpoint renamed (`urls.py:20`) |
| `/api/1.0/p/<store_id>/<retailer_id>` | `/api/1.0/promotions/<store_id>/<retailer_id>` | Short alias (`urls.py:30`) |

---

## Open Questions

- The request serializers (`PromoEvaluateRequestSerializer`, `Basket`, `BasketItem`) are defined but unused in the evaluate view. Were they originally used for validation and later bypassed for performance, or are they intended for future use?
- The `PromotionsByStore.delete()` uses raw SQL and does not check for shared promotions. Is this endpoint called from the monolith, or is it only for development/admin cleanup?
- The `PromotionSuggestionListByNodes` and `PromoBasedItemRecommendations` endpoints always use V1 (`PromoLogic`). Should they support V2?
