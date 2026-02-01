# Cashier Kiosk

> Last updated by: Task T22 — Cashier Kiosk module (shift management, cash tracking, Kafka integration)

## Overview

The `mishipay_cashier_kiosk` module (12 files, ~550 lines of application code) implements **staff shift management for Android-based POS cashier kiosk devices**. It tracks cash-in-drawer amounts across shifts, records per-transaction sales and refund activity, produces daily shift summaries, and publishes shift-end events via Kafka for Flying Tiger stores. The module operates as an auxiliary layer that hooks into the existing order and refund processing pipelines.

This module does **not** handle the actual checkout or payment flow — those are managed by `mishipay_retail_orders`, `mishipay_retail_payments`, and `mishipay_items` through the standard `android_cashier_kiosk` platform path. Instead, this module wraps those flows with shift-scoped cash accounting.

## Key Concepts

- **Shift** — A time-bounded work session for a staff member on a specific kiosk device. Only one active shift is allowed per device at a time.
- **Denomination Breakup** — A JSON dictionary mapping currency denominations to their total value in the till (e.g., `{"0.01": 0.05, "10.0": 100}`). Validated to sum to the declared amount.
- **Business Date** — The local-timezone date of a shift's start time, used for daily aggregation.
- **Store Shift ID** — An auto-incrementing integer per store+device pair, providing a human-readable shift sequence number.
- **Mismatch** — The difference between expected cash in the till (start + sales - returns) and the actual counted amount at shift end.
- **Bank Drop** — Cash removed from the till during a shift (tracked as `removed_shift_amount`).

## Architecture

```
mishipay_cashier_kiosk/
├── __init__.py           # Empty
├── apps.py               # MishipayCashierKioskConfig
├── models.py             # 3 models: KioskStaffShift, KioskStaffShiftSummary, KioskStaffShiftActivity
├── serializers.py        # 4 serializers + 1 helper
├── views.py              # 4 views: StartShift, EndShift, ActiveShift, ListStaffShifts
├── urls.py               # 4 URL patterns under v1/
├── utils.py              # 4 utility functions (called by other modules)
├── admin.py              # 3 admin registrations
├── pagination.py         # SmallResultSetPagination (page_size=20, max=100)
├── tests.py              # 1 test class, 8 test methods (~455 lines)
└── migrations/           # 9 migrations (March 2023 – October 2023)
```

### Module Dependencies

```
mishipay_cashier_kiosk
├── depends on:
│   ├── dos.models.Store                          (store FK)
│   ├── dashboard.models.user.User                (staff FK)
│   ├── mishipay_retail_orders.models.Order        (activity FK)
│   ├── mishipay_retail_orders.SUCCESS_ORDER_STATUSES (transaction counting)
│   ├── mishipay_dashboard.models.RefundOrderNew   (refund counting)
│   ├── mishipay_core.common_functions             (response structure, rounding)
│   ├── mishipay_core.permissions                  (ActiveStaffShiftPermission)
│   ├── mishipay_dashboard.permissions             (DashboardPermission)
│   └── pubsub.adapters.CashDeskEndShiftEventProducerAdapter (Kafka)
│
├── depended on by:
│   ├── mishipay_retail_orders.models.Order.complete_status()
│   │   └── imports: create_kiosk_staff_shift_activity, increment_shift_transaction_count
│   ├── mishipay_dashboard.views.RefundOrderNewView
│   │   └── imports: create_kiosk_staff_shift_activity, increment_shift_transaction_count
│   │   └── imports: KioskStaffShiftSummary, KioskStaffShiftSummarySerializer
│   └── mishipay_core.permissions.ActiveStaffShiftPermission
│       └── imports: KioskStaffShift
```

## Data Models

### KioskStaffShift

Represents a single work shift for a staff member on a specific kiosk device. Source: `models.py:15-45`.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | AutoField | PK | Auto-generated integer PK |
| `shift_id` | UUIDField | unique, non-editable | Public-facing shift identifier |
| `store_shift_id` | IntegerField | part of unique_together | Auto-incrementing per store+device sequence number |
| `staff` | FK → `dashboard.User` | PROTECT | Dashboard user who opened the shift |
| `retailer_device_id` | CharField(30) | not null, part of unique_together | Kiosk terminal identifier (e.g., `"K1"`, `"C1"`) |
| `worker_id` | CharField(50) | not null | Staff worker ID (from `DashboardUser.retailer_staff_id`) |
| `store` | FK → `dos.Store` | PROTECT, part of unique_together | Store this shift belongs to |
| `status` | BooleanField | default `True` | `True` = active (open), `False` = ended |
| `start_time` | DateTimeField | auto_now_add | When shift was started |
| `end_time` | DateTimeField | nullable | When shift was ended |
| `start_shift_amount` | Decimal(15,2) | required | Cash in drawer at shift start |
| `end_shift_amount` | Decimal(15,2) | nullable | Cash declared at shift end (after bank drop) |
| `end_shift_till_cash_count` | Decimal(15,2) | nullable | Actual total cash counted in till at end |
| `has_mismatch` | BooleanField | nullable | Whether a cash mismatch was detected |
| `mismatch_amount` | Decimal(15,2) | default 0.0 | Mismatch amount (can be negative) |
| `removed_shift_amount` | Decimal(15,2) | nullable | Cash removed (bank drop) during shift |
| `start_denomination_breakup` | JSONField | default dict | Denomination-level breakdown at start |
| `end_denomination_breakup` | JSONField | default dict, nullable | Denomination-level breakdown at end |
| `removed_denomination_breakup` | JSONField | default dict, nullable | Denomination-level breakdown of bank drop |
| `sales_count` | IntegerField | default 0 | Count of cash payment activities (sales + refunds) |
| `total_sales_count` | IntegerField | default 0 | Count of all transactions (all payment methods) |
| `sales_amount` | Decimal(15,2) | default 0 | Running total of cash sales |
| `returns_amount` | Decimal(15,2) | default 0 | Running total of cash refunds |
| `taxes` | Decimal(15,2) | default 0 | Running total of tax on cash sales |
| `extra_data` | JSONField | default dict | Stores `order_list` and `refund_list` arrays |

**Constraints:**
- `unique_together`: (`store`, `retailer_device_id`, `store_shift_id`)
- **Index:** (`store`, `retailer_device_id`, `status`, `-start_time`) — optimized for active shift lookup

### KioskStaffShiftSummary

Daily aggregate of all shifts for a given device at a store. Created or updated when a shift ends. Source: `models.py:48-64`.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | AutoField | PK | Auto-generated PK |
| `store` | FK → `dos.Store` | PROTECT, part of unique_together | Store |
| `business_date` | DateField | part of unique_together | Local-timezone date of the shift start |
| `retailer_device_id` | CharField(30) | part of unique_together | Device identifier |
| `shifts_list` | ArrayField(UUIDField) | required | List of shift_ids in this daily summary |
| `shifts_count` | IntegerField | default 0 | Number of shifts in the day |
| `sales_count` | IntegerField | default 0 | Accumulated cash activity count |
| `total_sales_count` | IntegerField | default 0 | Accumulated all-payment transaction count |
| `sales_amount` | Decimal(15,2) | default 0 | Accumulated cash sales |
| `taxes` | Decimal(15,2) | default 0 | Accumulated tax |
| `returns_amount` | Decimal(15,2) | default 0 | Accumulated cash refunds |

**Constraints:**
- `unique_together`: (`retailer_device_id`, `store`, `business_date`)
- **Index:** (`store`, `retailer_device_id`, `-business_date`)

### KioskStaffShiftActivity

Individual transaction record linking an order (and optionally a refund) to a shift. Created only for **cash** payment transactions. Source: `models.py:67-75`.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | AutoField | PK | Auto-generated PK |
| `shift` | FK → `KioskStaffShift` | PROTECT | Parent shift |
| `order` | FK → `Order` | PROTECT | Associated order |
| `refund_order_id` | UUIDField | nullable | Refund order ID (null for sales) |
| `amount` | Decimal(15,2) | required | Sale amount (0 for refund records) |
| `tax` | Decimal(15,2) | required | Tax amount (0 for refund records) |
| `refund_amount` | Decimal(15,2) | required | Refund amount (0 for sale records) |
| `created` | DateTimeField | auto_now_add | Timestamp |

### Entity Relationship

```
dos.Store ─────────────── KioskStaffShift ──── KioskStaffShiftActivity
    │                          │                        │
    │                          │ FK (staff)              │ FK (order)
    │                          ▼                        ▼
    │                   dashboard.User        mishipay_retail_orders.Order
    │
    └──────────────── KioskStaffShiftSummary
                     (daily aggregate per device)
```

## API Endpoints

All endpoints are mounted at `/staff-shifts/` via `mainServer/urls.py`. Source: `urls.py:1-10`.

| Method | Path | View | Permissions | Description |
|--------|------|------|------------|-------------|
| POST | `/staff-shifts/v1/start-shift/` | `StartShift` | `DashboardPermission` | Open a new shift on a device |
| POST | `/staff-shifts/v1/end-shift/` | `EndShift` | `DashboardPermission`, `ActiveStaffShiftPermission` | Close the active shift |
| GET | `/staff-shifts/v1/list-shifts/` | `ListStaffShifts` | `DashboardPermission` | Retrieve shift details by IDs |
| GET | `/staff-shifts/v1/get-active-shift/` | `ActiveShift` | `DashboardPermission` | Check for active shift on a device |

### POST `/staff-shifts/v1/start-shift/`

Opens a new shift. Source: `views.py:30-82`.

**Request:**
```json
{
    "store_id": "uuid",
    "retailer_device_id": "K1",
    "start_shift_amount": 100.25,
    "start_denomination_breakup": {
        "0.01": 0.05,
        "0.02": 0.20,
        "5.0": 0,
        "10.0": 100
    }
}
```

**Validation (StartShiftSerializer):**
1. No active shift (status=True) may exist for the same store + device
2. Denomination breakup values must sum to `start_shift_amount`
3. Staff user must have a non-empty `retailer_staff_id` (used as `worker_id`)

**Business Logic:**
1. Auto-increments `store_shift_id` by querying the last shift for the store+device
2. Wrapped in `@transaction.atomic`
3. Uses `force_insert=True` on save

**Response:** `{ "shift_id": "uuid" }`

### POST `/staff-shifts/v1/end-shift/`

Closes the active shift, calculates totals, and publishes a Kafka event (for Flying Tiger). Source: `views.py:85-182`.

**Request:**
```json
{
    "shift_id": "uuid",
    "store_id": "uuid",
    "end_shift_amount": 50.00,
    "total_cash_in_till": 90.25,
    "removed_shift_amount": 40.25,
    "end_denomination_breakup": { "10.0": 50 },
    "mismatch_amount": -10.00
}
```

**Validation (EndShiftSerializer):**
Two cash balance equations must hold:
1. `total_cash_in_till == start_shift_amount + sales_amount + mismatch_amount - returns_amount`
2. `start_shift_amount + sales_amount + mismatch_amount == returns_amount + end_shift_amount + removed_shift_amount`

Denomination breakup must sum to `end_shift_amount`.

**Business Logic:**
1. Validates the shift exists (via `ActiveStaffShiftPermission` which sets `request.shift`)
2. Calculates `total_sales_count` by re-querying orders and refunds (`count_total_transactions()`)
3. Creates or updates a `KioskStaffShiftSummary` for the business date (store timezone)
4. Sets `shift.status = False`, records end amounts
5. Sets `has_mismatch = True` if `mismatch_amount != 0`
6. **For Flying Tiger only:** Publishes a Kafka event via `CashDeskEndShiftEventProducerAdapter`

**Kafka Event (Flying Tiger only):**
- **Topic:** `flying_tiger.cash_desk.staff_shifts.end_shift_event`
- **Condition:** `store.store_type == "FlyingTigerStoreType"` AND `not store.demo` AND `store.active`
- **Payload:**
```json
{
    "shiftId": 1,
    "register": "K1",
    "storeId": "GB01054",
    "workerId": "W001",
    "currency": "GBP",
    "transactionCount": 5,
    "startTimestamp": "2023-06-15T08:00:00Z",
    "closeTimestamp": "2023-06-15T16:00:00Z",
    "startingAmount": "100.25",
    "bankDrop": "40.25",
    "tenderDeclarationAmount": "50.00"
}
```

### GET `/staff-shifts/v1/get-active-shift/`

Returns the currently active shift for a device. Source: `views.py:210-237`.

**Query Params:** `retailer_device_id` (required), `store_id` (injected by middleware)

**Response:** `ShiftExistenceSerializer` output with `shift_id`, `store_shift_id`, `staff_username`, `retailer_device_id`, `status`, `start_time`. Returns empty `data: {}` if no active shift.

### GET `/staff-shifts/v1/list-shifts/`

Retrieves shift details by comma-separated shift IDs. Paginated. Source: `views.py:185-207`.

**Query Params:** `shift_ids` (comma-separated UUIDs)

**Response:** Paginated `ShiftSerializer` output with full shift details including denomination breakups and financial totals.

### GET `/v1/cash-report/` (in `mishipay_dashboard`)

The daily cash report is served from `mishipay_dashboard`, not this module. Source: `mishipay_dashboard/urls.py`.

**View:** `GetCashReport` (ListAPIView) — queries `KioskStaffShiftSummary` for a single store, ordered by `-business_date`.

**Serializer:** `KioskStaffShiftSummarySerializer` (defined in this module, imported by dashboard) — adds `register` field formatted as `"{retailer_store_id}-{retailer_device_id}"`.

## Business Logic

### Shift Lifecycle

```
START SHIFT                    DURING SHIFT                     END SHIFT
─────────────────────────────────────────────────────────────────────────────
Staff opens shift              Orders/refunds processed         Staff closes shift
  │                              │                               │
  ├── Validate no active shift   ├── Order.complete_status()     ├── Recount transactions
  ├── Validate denomination sum  │   triggers:                   ├── Create/update daily summary
  ├── Validate worker_id         │   increment_shift_count()     ├── Record end amounts
  ├── Auto-assign store_shift_id │   (all payment methods)       ├── Calculate mismatch
  └── Create KioskStaffShift     │                               ├── Mark shift inactive
      (status=True)              │   create_shift_activity()     └── Publish Kafka (FT only)
                                 │   (cash payments only)
                                 │
                                 ├── Dashboard refund triggers:
                                 │   increment_shift_count()
                                 │   create_shift_activity()
                                 │   (cash refunds only)
                                 │
                                 └── Shift.extra_data tracks:
                                     order_list: [order_id, ...]
                                     refund_list: [refund_id, ...]
```

### Transaction Tracking (Two-Track System)

The module tracks transactions through two complementary mechanisms:

**Track 1: `KioskStaffShiftActivity` records** — Created only for **cash** payments. Captures detailed per-transaction amounts (sales_amount, tax, refund_amount). Updates shift running totals (`sales_count`, `sales_amount`, `returns_amount`, `taxes`). Source: `utils.py:11-25`.

**Track 2: `extra_data` order/refund lists** — Updated for **all** payment methods. Appends order_id or refund_order_id to JSON arrays. Increments `total_sales_count`. Source: `utils.py:28-46`.

This dual tracking explains the two count fields:
- `sales_count` — Cash-only transactions (incremented by `create_kiosk_staff_shift_activity`)
- `total_sales_count` — All transactions regardless of payment method (incremented by `increment_shift_transaction_count`, recalculated at shift end by `count_total_transactions`)

### Integration with Order Completion

When an order completes with `platform == "android_cashier_kiosk"`, the `Order.complete_status()` method (`mishipay_retail_orders/models.py:190-202`) calls:

1. `increment_shift_transaction_count(shift_id, order_id)` — always
2. `create_kiosk_staff_shift_activity(shift_id, order, None, captured_amount, vat_price, 0)` — only for cash payments with positive captured_amount
3. For negative captured_amount: records as refund (`amount=0, refund_amount=abs(captured_amount)`)

### Integration with Refund Processing

When a refund is processed for an `android_cashier_kiosk` order, `mishipay_dashboard.views.RefundOrderNewView` (`mishipay_dashboard/views.py:3264-3270`) calls:

1. `increment_shift_transaction_count(shift_id, order_id, refund_order_id)` — always
2. `create_kiosk_staff_shift_activity(shift_id, order, refund_order_id, 0, 0, refund_amount)` — only for cash refunds

### End-of-Shift Transaction Recount

`count_total_transactions()` (`utils.py:49-73`) provides an independent recount at shift end by querying:
- `Order` objects with `platform='android_cashier_kiosk'`, successful status, completion after shift start, where `order.extra_data['shift_id']` matches
- `RefundOrderNew` objects with matching store, created after shift start, `REFUNDED` or `PROCESSING` status, where `refund.extra_data` matches platform, store_shift_id, and retailer_device_id

This recount guards against missed `increment_shift_transaction_count` calls.

### Cash Balance Validation

The `EndShiftSerializer` enforces two cash accounting equations:

**Equation 1 — Till balance:**
```
total_cash_in_till = start_shift_amount + sales_amount + mismatch_amount - returns_amount
```

**Equation 2 — Cash disposition:**
```
start_shift_amount + sales_amount + mismatch_amount = returns_amount + end_shift_amount + removed_shift_amount
```

Where:
- `start_shift_amount` = cash counted at shift start
- `sales_amount` / `returns_amount` = running totals from cash transactions during the shift
- `mismatch_amount` = unexplained difference (can be negative for shortages)
- `removed_shift_amount` = cash taken out (bank drop)
- `end_shift_amount` = cash remaining in the drawer
- `total_cash_in_till` = total cash physically counted

### Denomination Breakup Validation

`_validate_denomination_breakup()` (`serializers.py:9-15`) sums the values in the breakup dictionary and compares against the declared amount using `get_rounded_value()` (2 decimal places). Note: uses `float` arithmetic for the summation, which could cause floating-point precision issues with many small denominations.

## Permissions

| Permission Class | Module | Description |
|-----------------|--------|-------------|
| `DashboardPermission` | `mishipay_dashboard.permissions` | Validates the user belongs to `DashboardUser` group |
| `ActiveStaffShiftPermission` | `mishipay_core.permissions` | Only enforces for `android_cashier_kiosk` platform; validates an active shift exists for the given `shift_id` and sets `request.shift` |

`ActiveStaffShiftPermission` (`mishipay_core/permissions.py:43-60`) is a conditional permission that:
- Returns `True` immediately for non-kiosk platforms
- For `android_cashier_kiosk`: requires a valid `shift_id` parameter and verifies an active shift exists
- Raises `NoActiveStaffShiftException` (HTTP 403) if no active shift is found

## Configuration

No environment variables, feature flags, or settings are specific to this module. It uses the standard Django database and Kafka infrastructure.

## Admin Interface

Three models are registered in the Django admin. Source: `admin.py:1-81`.

| Model | List Display | Search Fields |
|-------|-------------|---------------|
| `KioskStaffShift` | shift_id, username, retailer_store_id, status, start/end time, amounts, mismatch | staff username/email, store_id, retailer_store_id, retailer_device_id |
| `KioskStaffShiftSummary` | retailer_store_id, device_id, business_date, counts, amounts | store_id, retailer_store_id, retailer_device_id, business_date |
| `KioskStaffShiftActivity` | shift_id, order_id, refund_order_id, amount, tax, created | shift_id, order_id |

Custom display methods (`username`, `retailer_store_id`) traverse FK relationships for readability.

## Schema Evolution

9 migrations spanning March 2023 to October 2023:

| Migration | Date | Change |
|-----------|------|--------|
| `0001_initial` | March 2023 | All 3 models created, indexes and unique constraints |
| `0002` | March 2023 | Added `sales_count`, `sales_amount`, `returns_amount`, `taxes` to `KioskStaffShift` |
| `0003` | March 2023 | Migrated JSONField from `django.contrib.postgres.fields.jsonb` to `django.db.models.JSONField` |
| `0004` | 2023 | Added `worker_id` to `KioskStaffShift` |
| `0005` | 2023 | Changed `unique_together` from `(store, store_shift_id)` to `(store, retailer_device_id, store_shift_id)` |
| `0006` | June 2023 | Added `total_sales_count` to both `KioskStaffShift` and `KioskStaffShiftSummary` |
| `0007` | September 2023 | Added `extra_data` JSONField (for order_list/refund_list tracking) |
| `0008` | October 2023 | Merge migration |
| `0009` | October 2023 | Altered `extra_data` field options |

Initial creation was on Django 2.2.28, with migrations 0003 and 0006 generated under Django 4.1.7.

## Test Coverage

Source: `tests.py` — 1 test class (`TestCashierKioskShifts`), 8 test methods, ~455 lines.

**Setup:** Uses Flying Tiger store type (`FlyingTigerStoreType`), ByPass payment provider, dashboard user authentication.

| Test | Description |
|------|-------------|
| `test_get_active_shift_when_no_shift_exists` | Verifies empty response for no active shift |
| `test_create_shift_success` | Creates shift, verifies DB record, active shift query, and list query |
| `test_create_shift_when_shift_exists` | Verifies 400 error when opening a second shift on same device |
| `test_create_shift_with_invalid_data` | Verifies 400 when denomination sum doesn't match amount |
| `test_end_shift_success` | Creates and ends shift, verifies status set to False |
| `test_end_shift_with_incorrect_shift_id` | Verifies 403 for non-existent shift ID |
| `test_end_shift_with_incorrect_data` | Verifies 400 when cash equations don't balance |
| `test_shift_full_update_flow` | End-to-end: start shift → create order → process refund → end shift → verify cash report |
| `test_multiple_shifts_update_flow` | Two consecutive shifts with orders and refunds, verifies aggregation in daily summary |

The tests exercise the full integration path including `create_cashier_kiosk_order()` which calls order creation (`create_order_v2`), payment (`by_pass_payment_as_service`), and payment verification — confirming that `KioskStaffShiftActivity` records are created as a side effect of the order completion flow.

## Notable Patterns

1. **Bare `except` for store resolution** — Both `StartShift.post()` and `EndShift.post()` use a bare `except:` clause (`views.py:40-41`, `views.py:96-98`) to handle missing `request.store`, falling back to a DB lookup. This swallows all exceptions and masks potential errors.

2. **Naive datetime at shift end** — `datetime.utcnow()` is used (`views.py:142`) instead of `django.utils.timezone.now()`, producing a naive datetime when `USE_TZ=True`. The `start_time` field uses `auto_now_add=True` which respects Django's timezone setting, creating an inconsistency.

3. **Non-atomic shift end** — While `EndShift.post()` uses `@transaction.atomic`, the Kafka event is produced inside the transaction. If the Kafka publish fails, the entire shift end is rolled back. Conversely, if the transaction fails after publishing, the Kafka event cannot be retracted.

4. **Float arithmetic in validation** — `_validate_denomination_breakup()` (`serializers.py:10-12`) sums denomination values using `float` arithmetic, which can accumulate rounding errors. The `Decimal` type would be more appropriate for currency calculations.

5. **N+1 query in `count_total_transactions`** — The function iterates all orders and refunds in Python to check `extra_data` JSON fields (`utils.py:65-72`), rather than filtering in the database. For shifts with many transactions, this could be slow.

6. **Race condition on shift counters** — `create_kiosk_staff_shift_activity()` reads shift counters, increments in Python, and saves back (`utils.py:20-24`). Concurrent order completions on the same shift could cause lost updates since there's no `F()` expression or `select_for_update()`.

7. **Dual-track counting design** — The separation between `sales_count` (cash only) and `total_sales_count` (all payments) is deliberate but the naming is confusing. `sales_count` doesn't count just sales — it also counts refund activities.

## Dependencies

### Inbound (other modules depend on this)

| Module | What it imports | Trigger |
|--------|----------------|---------|
| `mishipay_retail_orders.models` | `create_kiosk_staff_shift_activity`, `increment_shift_transaction_count` | Order completion for `android_cashier_kiosk` platform |
| `mishipay_dashboard.views` | `create_kiosk_staff_shift_activity`, `increment_shift_transaction_count`, `KioskStaffShiftSummary`, `KioskStaffShiftSummarySerializer` | Refund processing + cash report endpoint |
| `mishipay_core.permissions` | `KioskStaffShift` | `ActiveStaffShiftPermission` validation |

### Outbound (this module depends on)

| Module | What is imported | Purpose |
|--------|-----------------|---------|
| `dos.models.Store` | FK target | Store reference |
| `dashboard.models.user.User` | FK target | Staff reference |
| `mishipay_retail_orders.models.Order` | FK target + query | Activity records + recount |
| `mishipay_retail_orders.SUCCESS_ORDER_STATUSES` | Constants | Transaction recount filter |
| `mishipay_dashboard.models.RefundOrderNew` | Query | Transaction recount filter |
| `mishipay_core.common_functions` | `get_generic_response_structure`, `get_rounded_value` | Response formatting, validation |
| `mishipay_core.permissions` | `ActiveStaffShiftPermission` | Shift validation |
| `mishipay_dashboard.permissions` | `DashboardPermission` | Auth check |
| `pubsub.adapters` | `CashDeskEndShiftEventProducerAdapter` | Kafka event publishing |

See [Core Module](./core.md) for `ActiveStaffShiftPermission` details, [Dashboard](./dashboard.md) for `DashboardPermission` and `GetCashReport` view, [Orders](./orders.md) for `Order.complete_status()` integration.

## Open Questions

1. **Flying Tiger-only Kafka events** — The shift-end Kafka event is only published for `FlyingTigerStoreType`. Are other retailers expected to receive these events in the future, or does Flying Tiger have unique POS integration requirements?

2. **Race condition on shift counters** — `create_kiosk_staff_shift_activity()` does not use `F()` expressions or `select_for_update()` when incrementing `sales_count`, `sales_amount`, etc. Are concurrent transactions on the same shift a real-world scenario (e.g., multiple staff members on one device)?

3. **`sales_count` naming** — `sales_count` is incremented for both sales AND refund activities (`utils.py:20`). Should it be renamed to `cash_activity_count` to avoid confusion with `total_sales_count`?

4. **Missing `verify_active_session_exists` callers** — `utils.py:76-82` defines `verify_active_session_exists()` but it is not imported or called by any code in the kiosk module or elsewhere visible in the codebase. Is this dead code?

5. **Bare `except:` for store resolution** — `views.py:40-41` and `views.py:96-98` use bare `except:` to catch missing `request.store`. This suppresses all exceptions. Should this be `except AttributeError:` at minimum?

6. **`end_time` uses `datetime.utcnow()`** — This produces a naive datetime (`views.py:142`) while `start_time` uses Django's timezone-aware `auto_now_add`. Could this cause issues in timezone-aware queries or reporting?

7. **Denomination validation uses float** — `_validate_denomination_breakup()` sums with `float` arithmetic (`serializers.py:10-12`). Has this caused precision issues in production with many small denomination entries?
