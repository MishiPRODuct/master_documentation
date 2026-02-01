# Dashboard

> Last updated by: Task T15 — Referrals & Dashboard

## Overview

The dashboard system is split across **two Django modules** that together provide a comprehensive retail staff portal for managing orders, refunds, inventory, users, and analytics:

1. **`dashboard`** (50 files, ~3,300 lines) — User management, authentication, and access control for dashboard staff. Provides login flows, bulk user creation, and role-based permissions.
2. **`mishipay_dashboard`** (52 files, ~7,000+ lines) — The core operational API: order management, refund processing, inventory operations, settlement reconciliation, receipt generation, manual verification, and LLM-powered chat.

Both modules serve the same web-based dashboard UI used by retail staff at MishiPay-equipped stores.

## Key Concepts

| Term | Meaning |
|------|---------|
| **Dashboard User** | A staff member with login credentials and store-level permissions, belonging to the `DashboardUser` Django group |
| **Token** | An invite/registration token with pre-configured permissions, used to onboard new dashboard users |
| **Store Permission** | Access control linking a user to specific stores they can manage |
| **RefundOrderNew** | A refund record tracking partial/full refunds processed through the new refund architecture |
| **Manual Verification** | Staff-initiated order verification process (bag checks, item counts) |
| **Click & Collect (CC)** | Order type where customers order online and pick up in-store; has its own status workflow |
| **Settlement Reconciliation** | Reports comparing payment gateway records against internal order/payment data |
| **DDF** | Dubai Duty Free — a major retailer with custom authentication (QR + airport pass), tab permissions, and kiosk workflows |

## Architecture

### Module Relationship

```
┌─────────────────────────────────────────────────────────┐
│                   Dashboard UI (Web)                     │
└───────────────────────┬─────────────────────────────────┘
                        │
          ┌─────────────┼──────────────┐
          ▼                            ▼
┌─────────────────┐          ┌──────────────────────┐
│    dashboard     │          │  mishipay_dashboard   │
│ (Auth & Users)   │          │ (Operations API)      │
│                  │          │                        │
│ - Login/signup   │          │ - Order management     │
│ - Bulk user mgmt │          │ - Refund processing    │
│ - Permissions    │          │ - Inventory CRUD       │
│ - Store access   │          │ - Settlement reports   │
│ - Password/PIN   │          │ - Manual verification  │
│   recovery       │          │ - Receipt generation   │
│                  │          │ - Analytics queries    │
│ Models:          │          │ - LLM chat             │
│ - User           │          │                        │
│ - Token          │          │ Model:                 │
└─────────────────┘          │ - RefundOrderNew       │
                              └──────────────────────┘
```

### Store-Type Dispatch Pattern

Both modules use a store-type dispatch pattern to handle retailer-specific logic. Operations like refunds, order verification, and item management are delegated to store-type-specific handlers via configuration maps:

```python
# mishipay_dashboard/app_settings.py
REFUND_PSP_MAPPING = {
    'decathlonstoretype': decathlon_refund,
    'mujistoretype': muji_order_refund,
    # ... per-retailer refund handlers
}

UPDATE_BASKET_MAPPING_FOR_REFUND = {
    'saturnsmartpaystoretype': mishipay_update_basket_for_refund,
    'mishipaystoretype': mishipay_update_basket_for_refund,
    'decathlonstoretype': decathlon_update_basket_for_refund,
    # ... per-retailer basket update handlers
}

# dashboard/views/orders.py uses dos.store_map.STORE_TYPE_MAP
```

---

# Module 1: dashboard (User Management & Authentication)

## Data Models

### User

**File:** `dashboard/models/user.py` (~50 lines)

The core dashboard user entity. Linked to Django's `auth.User` via a OneToOneField.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `user_id` | UUIDField | PK, default=uuid4 | |
| `auth_user` | OneToOneField(auth.User) | null, on_delete=CASCADE | Links to Django auth system |
| `username` | TextField | unique | Login username |
| `password` | TextField | — | Hashed password (duplicated in auth_user) |
| `email` | EmailField | blank, default='' | |
| `qr` | CharField(127) | unique, null | QR code for kiosk login |
| `retailer_staff_id` | CharField(50) | db_index | Worker/staff ID from retailer |
| `retailer_staff_pin` | TextField | null | PIN for passcode login |
| `force_reset_retailer_staff_pin` | BooleanField | default=False | Force PIN reset on next login |
| `store_id` | UUIDField | — | Primary store association |
| `stores` | ManyToManyField(Store) | — | Multi-store access |
| `mishipay_staff` | BooleanField | default=False | Internal MishiPay staff flag |
| `can_add` | BooleanField | default=True | Permission: add items |
| `can_edit` | BooleanField | default=True | Permission: edit items |
| `can_delete` | BooleanField | default=False | Permission: delete items |
| `can_refund` | BooleanField | default=False | Permission: process refunds |
| `allow_analytics` | BooleanField | default=False | Permission: view analytics |
| `hide_customer_info` | BooleanField | default=False | Hide customer PII in order views |
| `allow_settlement_report` | BooleanField | default=False | Permission: settlement reports |
| `end_date_time` | DateTimeField | null | Account expiration (manual) |
| `created` | DateTimeField | auto_now=True | |
| `allow_users_create` | BooleanField | default=False | Permission: bulk create users |
| `allow_users_llm` | BooleanField | default=False | Permission: LLM chat features |

### Token

**File:** `dashboard/models/token.py` (~36 lines)

Invite/registration tokens for onboarding new dashboard users. Tokens carry pre-configured permissions that are inherited by the created user.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `token_id` | UUIDField | PK, default=uuid4 | |
| `store_id` | UUIDField | — | Store this token is for |
| `expired` | BooleanField | default=False | |
| `can_add` | BooleanField | default=True | Inherited by user |
| `can_edit` | BooleanField | default=True | Inherited by user |
| `can_delete` | BooleanField | default=False | Inherited by user |
| `can_refund` | BooleanField | default=False | Inherited by user |
| `allow_analytics` | BooleanField | default=False | Inherited by user |
| `username` | TextField | null | Username when used |
| `created` | DateField | default=datetime.now | |
| `used` | DateField | null | Date the token was consumed |

## API Endpoints

**File:** `dashboard/urls.py` (57 lines)

### Authentication Endpoints

| Method | Path | View | Purpose |
|--------|------|------|---------|
| POST | `v2/user/login` | `LoginView` | Username/password login |
| POST | `v2/user/worker-login` | `PasscodeLoginView` | Staff ID + PIN login |
| POST | `v2/user/sign-up` | `SignUpView` | New user registration with invite token |
| POST | `v2/user/recover-password` | `PasswordRecoveryView` | Send password recovery email |
| POST | `v2/user/reset-password` | `PasswordResetView` | Reset password with signed link |
| POST | `v2/user/reset-passcode` | `PasscodeResetView` | Reset 6-digit PIN |
| POST | `v2/staff/login_with_qr` | `StaffLoginQRView` | QR code-based kiosk login |
| POST | `v2/staff/login` | `StaffLoginView` | Regular staff login |

### Order Management Endpoints

| Method | Path | View | Purpose |
|--------|------|------|---------|
| GET | `v2/order/list/all` | `AllOrderListView` | List orders across all user's stores |
| GET/POST | `v2/order/get` | `OrderViewSet` | Retrieve specific order |
| GET | `v2/order/list` | `OrderListViewSet` | List store orders (paginated) |
| GET | `v2/order/scan` | `OrderView` | Lookup by barcode or order_id |
| POST | `v2/order/refund` | `RefundOrder` | Process refund (legacy flow) |
| POST | `v2/order/verify` | `VerifyOrder` | Verify order correctness |
| POST | `v2/order/update-all-payment-statuses` | `UpdateAllOrderPaymentStatuses` | Sync payment status |
| POST | `v2/order/register-order` | `RegisterExternalOrder` | Register external order ID |
| POST | `v2/order/authorize` | `AuthorizeOrder` | Authorize order (Mango) |
| POST | `v2/order/cancel` | `CancelOrder` | Cancel order (Mango) |

### Item Management Endpoints

| Method | Path | View | Purpose |
|--------|------|------|---------|
| GET | `v2/item/all` | `ItemView` | List items (paginated) |
| GET | `v2/item/category/` | `ItemCategoryView` | List item categories |
| POST | `v2/item/update` | `ItemUpdateView` | Update item details |
| POST | `v2/item/add` | `ItemAddView` | Add new item |
| POST | `v2/item/delete` | `ItemDeleteView` | Delete item |
| POST | `v2/item/upload` | `ItemCsvUploadView` | Bulk CSV import |
| POST | `v2/item/epc/upload` | `SaturnUploadEpcList` | Upload RFID EPC list (Saturn) |
| POST | `v2/item/fetch` | `ItemFetchView` | Fetch single item details |

### User Management Endpoints

| Method | Path | View | Purpose |
|--------|------|------|---------|
| GET/POST | `v2/bulk-user-create` | `BulkDashboardUserCreateViewV2` | Bulk user creation, listing, download |
| GET | `v2/bulk-user-create-template` | `BulkDashboardUserCreateTemplateViewV2` | CSV template download |
| GET/POST | `bulk-user-create` | `BulkDashboardUserCreateView` | Legacy bulk user creation |
| GET | `bulk-user-create-template/` | `BulkDashboardUserCreateTemplateView` | Legacy CSV template |

### Reporting Endpoints

| Method | Path | View | Purpose |
|--------|------|------|---------|
| GET | `v2/report/sales_summary` | `SalesReportSummary` | Sales analytics report |

## Business Logic

### Authentication Flows

**File:** `dashboard/views/authentication.py` (644 lines)

#### Username/Password Login (`LoginView`)

1. Validates username and password
2. Looks up `User` and verifies credentials
3. If `third_party_login` flag is set, delegates to `_client_specific_login()` which maps store types to specialized login functions (e.g., Carters)
4. Returns: access token, user_id, store details, permissions, list of authorized stores
5. Supports optional `store_id` parameter for store-specific login context

#### Token-Based Signup (`SignUpView`)

1. Validates invite `Token` exists and `expired=False`
2. Creates new `User` with permissions inherited from Token
3. Creates linked `auth.User` and adds to `DashboardUser` group
4. Marks token as expired with usage date
5. Sends password reset email
6. Returns login response

#### QR Login (`StaffLoginQRView`)

1. Looks up user by QR code
2. Validates `end_date_time` hasn't expired
3. **DDF-specific:** Validates airport staff pass via `get_ddf_staff_details()`
4. Returns `visible_tabs` and `user_groups` for kiosk UI rendering
5. DDF IT staff restricted from transaction actions

#### Password/PIN Recovery

- **Password recovery:** Generates Django `TimestampSigner`-based link valid for 24 hours, sent via email
- **Password reset:** Validates signed code and updates both `User.password` and `auth_user.password`
- **Passcode reset:** Validates 6-digit format, updates `retailer_staff_pin`, clears `force_reset` flag

### Bulk User Creation

**File:** `dashboard/utils/bulk_user_creation.py` (318 lines)

`BulkDashboardUserCreateHelper` handles CSV-based mass user onboarding:

1. **Parse:** Validates 13-column CSV (username, email, store IDs, permissions, user_type, etc.)
2. **Create:** For each row:
   - Resolves `retailer_store_id` → `store_id` via Store model
   - Checks for duplicate emails
   - Creates Token → User → auth.User → AuthToken chain
   - Generates random 8-char password if not provided
   - Assigns Django groups based on `user_type`: `DashboardUser`, `DFF_STAFF`, or `DDF_ADMIN`
3. **Notify:** Sends password setup emails with timestamped signed links
4. **List:** Supports paginated listing of dashboard users with filter by retailer/store

### Permissions System

**File:** `dashboard/permissions.py` (75 lines)

| Permission Class | Purpose |
|-----------------|---------|
| `DashboardPermission` | Checks `DashboardUser` group membership (allows unauthenticated for login endpoints) |
| `DashboardOrderRefundPermission` | Checks `user.can_refund == True` |
| `StorePermission` | Validates user has access to the requested `store_id` |

#### Tab Permissions (DDF-Specific)

**11 permission tabs** for kiosk UI: `kiosk`, `scan_and_go`, `mpos`, `scan_receipt`, `basket_verification`, `active_item_report`, `price_activation`, `revenue_report`, `item_report`, `incomplete_orders_report`, `settlements`

- `DDF_STAFF` group: restricted to `{kiosk, scan_and_go, mpos, active_item_report}`
- `DDF_ADMIN` group: all tabs
- Regular users: empty permissions dict

### Serializers

**File:** `dashboard/serializer.py` (89 lines)

| Serializer | Key Features |
|-----------|-------------|
| `DashboardStoreSerializer` | Store metadata with `features_applicable` (merged from defaults + store_properties), `promo_version` (1=legacy, 2=new), `mixpanel_token` |
| `DashboardUserSerializer` | User details with computed `retailer_store_ids` (semicolon-separated), QR barcode URL via tec-it service |

### Admin Interface

**File:** `dashboard/admin.py` (38 lines)

| Admin Class | Display Fields | Filters |
|------------|---------------|---------|
| `UserAdmin` | user_id, username, store_id, can_add/edit/delete/refund | user_id, email |
| `TokenAdmin` | token_id, store_id, username, expired | — |

`UserAdmin` uses `filter_horizontal` for multi-store selection and excludes the password field.

### Test Coverage

| Test File | Lines | Focus |
|-----------|-------|-------|
| `test_user_level_permissions.py` | 245 | DDF_STAFF/DDF_ADMIN tab permissions, group assignment, login response |
| `test_bulk_create_user_from_dashboard.py` | 623 | CSV upload, validation, user creation, listing, download, error handling |

---

# Module 2: mishipay_dashboard (Operations API)

## Data Models

### RefundOrderNew

**File:** `mishipay_dashboard/models.py` (79 lines)

Tracks refund orders processed through the new refund architecture (as opposed to legacy refunds in the `dashboard` module's `RefundOrder` view).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| `refund_order_id` | UUIDField | unique, default=uuid4 | |
| `mode` | CharField(1) | choices: F (Full), P (Partial) | |
| `refund_amount` | DecimalField(15,2) | — | Total amount to refund |
| `refund_processing_amount` | DecimalField(15,2) | — | Amount currently in flight |
| `refunded_amount` | DecimalField(15,2) | — | Amount already refunded |
| `store_id` | UUIDField | db_index | Store where order was placed |
| `processed_at_store_id` | UUIDField | blank, null | Store where refund is processed |
| `processed_by_user_id` | UUIDField | blank, null | Staff processing the refund |
| `order_id` | UUIDField | db_index | Original order reference |
| `refund_transaction_id_poslog` | CharField(10) | blank, null | POS transaction reference |
| `items` | JSONField | default=dict | Items being refunded |
| `payment_gateway_response` | TextField | blank, null | PSP response data |
| `status` | CharField | choices: see below | |
| `extra_data` | JSONField | default=dict | Additional metadata |
| `created` | DateTimeField | auto_now_add | |
| `updated` | DateTimeField | auto_now | |

**Status lifecycle:**
```
CREATED → PROCESSING → REFUNDED
                     → PSP_FAILED
                     → FAILED
```

## API Endpoints

**File:** `mishipay_dashboard/urls.py` (98 lines)

### Order & Refund Endpoints

| Method | Path | View | Auth | Purpose |
|--------|------|------|------|---------|
| GET | `v1/order/scan/` | `GetOrderView` | Authenticated | Retrieve order by ID/barcode/item |
| GET | `v1/dashboard-order/details/` | `GetDashboardOrderDetails` | Authenticated | Full order details (tracks dashboard access count) |
| GET | `v1/order/list/` | `GetStoreOrdersView` | Dashboard+Store | Paginated order listing (v1) |
| GET | `v2/order/list/` | `GetStoreOrdersViewV3` | Dashboard+Store | Order listing with filters (v2) |
| GET | `v3/order/list/` | `GetStoreOrdersViewV4` | Dashboard+Store | Enhanced filtering (v3) |
| GET | `v4/order/list/` | `GetStoreOrdersViewV4` | Dashboard+Store | Current version (v4) |
| GET | `v1/order/list/all-stores/` | `GetAllStoreOrdersView` | Dashboard | Orders across all user's stores |
| POST | `v1/refund-order/` | `RefundOrder` | Dashboard | Legacy refund endpoint |
| POST | `v1/refund-order-new/` | `RefundOrderNewView` | Dashboard | New refund flow |
| GET | `v1/refund-order-new/status` | `GetRefundOrderNewStatus` | Dashboard | Check refund status |
| POST | `v1/change-order-status/` | `ChangeCCOrderStatus` | Dashboard | Click & Collect status transitions |
| POST | `v1/update-order-status/` | `OrderVerificationDashboard` | Dashboard | Manual order verification |
| GET | `v1/manual-check/{id}/` | `ManualVerificationDashboard` | Dashboard | Manual verification details |
| GET | `v1/basket-manual-check/` | `DashboardVerificationAllSessions` | Dashboard | List pending verifications |

### Inventory Endpoints

| Method | Path | View | Auth | Purpose |
|--------|------|------|------|---------|
| GET | `v1/item-scan/` | `GetItemScanView` | Dashboard | Item lookup |
| GET | `v1/item-inventory/list/` | `GetStoreItemInventoryView` | Dashboard+Store | Store inventory listing |
| POST | `v1/add-item-inventory/` | `AddSingleItemInventory` | Dashboard | Add inventory item |
| POST | `v1/update-item-inventory/` | `UpdateItemInventory` | Dashboard | Update inventory item |
| POST | `v1/delete-item-inventory/` | `DeleteInventoryItems` | Dashboard | Soft delete inventory |
| POST | `v1/item-bulk-upload/` | `ItemsBulkUpload` | Dashboard | Bulk CSV import |

### Settlement & Analytics Endpoints

| Method | Path | View | Auth | Purpose |
|--------|------|------|------|---------|
| GET | `v1/dashboard-analytics/` | `DashboardAnalytics` | Analytics perm | Analytics query endpoint |
| GET | `v1/settlement_summary_all_regions/` | `PaymentSettlementSummaryAllRegions` | Dashboard | Settlement summary |
| GET | `v1/settlement_summary_all_regions_old/` | `PaymentSettlementSummaryAllRegionsOld` | Dashboard | Legacy settlement |
| GET | `v1/settlement_poslog_recon_report_by_region/` | `PaymentSettlementPosLogReconByRegion` | Dashboard | POS log reconciliation |
| GET | `v1/settlement_recon_payout_report_by_region_and_psp/` | `PaymentSettlementReconPayoutByRegionAndPSP` | Dashboard | Payout reconciliation |
| GET | `v1/download_poslog_recon_report_by_region/` | `DownloadPaymentSettlementPosLogReconByRegion` | Dashboard | Download reconciliation CSV |
| GET | `v1/download_settlement_recon_payout_report_by_region_and_psp/` | `DownloadPaymentSettlementReconPayoutByRegionAndPSP` | Dashboard | Download payout CSV |
| GET | `v1/cash-report/` | `GetCashReport` | Dashboard | Cash reconciliation report |

### Miscellaneous Endpoints

| Method | Path | View | Auth | Purpose |
|--------|------|------|------|---------|
| GET | `v1/user/stores/` | `UserStoresListView` | Authenticated | List user's accessible stores |
| POST | `v1/signout/` | `SignOut` | Dashboard | User logout |
| POST | `v1/adyen/session/` | `AdyenSessionView` | Dashboard | Create Adyen payment session |
| GET | `v1/get-printed-receipt/` | `GetOrderPrintedReceipt` | Dashboard | Formatted receipt output |
| POST | `v1/basket-item-serial-number/` | `DashboardBasketItemSerialNumberView` | Dashboard | Record serial numbers |
| POST | `v1/order-notes/` | `DashboardOrderNotesView` | Dashboard | Add notes to orders |
| POST | `v1/dashboard-email/` | `DashboardEmailView` | Dashboard | Send order email |
| POST | `v1/chat/` | `DashboardChatView` | LLM perm | LLM chat endpoint |
| GET | `v2/chat/` | `DashboardChatStreamView` | LLM perm | Streaming LLM chat (SSE) |
| POST | `v1/stock-mark/` | `DashboardStockMarkView` | Dashboard | Mark item stock status |
| GET | `v1/payment-refund-test/` | `PaymentRefundTestView` | Dashboard | Test refund flow |
| POST | `v1/user/deactivate/` | `DeactivateUserByStaffView` | Dashboard | Disable user account |
| GET | `v1/vengo/blocked-users/` | `GetVengoBlockedUsers` | Dashboard | Vengo blocked customers |
| POST | `v1/vengo/update-blocked-status/` | `UpdateCustomerBlockedStatus` | Dashboard | Block/unblock Vengo customer |

## Business Logic

### Refund Processing (New Flow)

**File:** `mishipay_dashboard/views.py:1641` (`RefundOrderNewView` — ~1,665 lines)

The largest view class, handling the complete refund lifecycle for the new architecture.

**Key static methods:**
- `get_item_info()` — Constructs item refund data with tax calculations
- `get_promo_item_info()` — Handles promotional item refund logic
- `get_payment_data()` — Processes multi-payment scenarios

**POST flow:**
1. Validates refund items and amounts
2. Checks refund eligibility and policy (return window, partial refund support)
3. Creates `RefundOrderNew` record with status `CREATED`
4. Calls store-type-specific PSP refund handler (via `REFUND_PSP_MAPPING`)
5. Updates basket item quantities (`refunded_quantity`, `is_refunded`)
6. Triggers analytics recording and notifications
7. Manages RFID deactivation if applicable
8. Updates `RefundOrderNew` status to `REFUNDED` or `PSP_FAILED`

**Store-specific refund handlers** (from `psp_refund_functions.py`):

| Function | Store Type | Notes |
|----------|-----------|-------|
| `decathlon_refund()` | Decathlon | Creates async refund status check job |
| `muji_order_refund()` | MUJI | Updates payment refund data in extra_data |
| Default (via `refund_to_psp_new_flow()`) | All others | Routes to store-type `.refund_order()` |

**Basket update handlers** (from `update_basket.py`):

| Function | Purpose |
|----------|---------|
| `mishipay_update_basket_for_refund()` | Generic: increments `refunded_quantity`, sets `is_refunded` |
| `decathlon_update_basket_for_refund()` | Tracks `refund_processing_quantity` separately |
| `mishipay_update_basket_for_refund_new_flow()` | New architecture: handles addons, tax adjustments, PosLog updates |

### Order Listing (V4 — Current)

**File:** `mishipay_dashboard/views.py` (`GetStoreOrdersViewV4`)

- Default 1-day filter (unless date params provided)
- Supports 90-day search window when filtering by ID/transaction
- Timezone-aware date filtering using store's configured timezone
- Filters: email, price range, purchase type, CC order status, transaction ID

### Click & Collect Status

**File:** `mishipay_dashboard/views.py` (`ChangeCCOrderStatus`)

Allowed status transitions:
```
created → rejected_store
created → accepted
created → rejected_customer
```

Sends email notifications on status changes. Supports `GlobalConfig` overrides.

### Manual Order Verification

**File:** `mishipay_dashboard/views.py` (`OrderVerificationDashboard`, `DashboardVerificationAllSessions`)

- Creates `ManualVerificationSession` records
- Tracks reviewer and verification actions
- Eroski-specific logic: shows all baskets until explicitly marked as verified
- Filters by non-expired baskets

### LLM Chat Integration

**File:** `mishipay_dashboard/chat.py` (~100 lines)

`ChatService` class integrates **Anthropic Claude** via LangChain:

- **Configuration** pulled from `GlobalConfig`: model ID, temperature, max_tokens, allowed_tables
- `get_schema_string_for_managed_replica()` — Introspects Django model schemas for LLM context
- `sanitize_sql(sql)` — Regex-based SQL injection prevention (blocks DROP, ALTER, INSERT, UPDATE, DELETE, TRUNCATE)
- Provides LLM with tool definitions for safe read-only database access
- **Permissions:** Requires `AllowLLMPermission` (checks `user.allow_users_llm`)

### Settlement Reconciliation

Multiple views provide settlement reporting:
- `PaymentSettlementSummaryAllRegions` — Aggregate settlement by region
- `PaymentSettlementPosLogReconByRegion` — POS log reconciliation
- `PaymentSettlementReconPayoutByRegionAndPSP` — Payout reconciliation by PSP
- CSV download variants for all reports

### Receipt Generation

**File:** `mishipay_dashboard/printed_receipts.py`, `mishipay_dashboard/receipts/dufry_receipt.py`

Dufry-specific printed receipt generator with configurable formatting (line width, margins, separators). Components: header (store name, address, helpline), item listing, footer (thank you message).

## Permissions (mishipay_dashboard)

**File:** `mishipay_dashboard/permissions.py` (95 lines)

| Permission Class | Purpose |
|-----------------|---------|
| `DashboardPermission` | Checks `DashboardUser` group membership |
| `StorePermission` | Validates user access to requested store |
| `AddItemsStorePermission` | Checks `user.can_add` flag |
| `AllowAnalyticsPermission` | Checks `user.allow_analytics` flag |
| `AllowLLMPermission` | Checks `user.allow_users_llm` flag |
| `AnalyticsStorePermission` | Validates store access for analytics (supports multi-store and store-type filters) |

## Serializers (mishipay_dashboard)

**File:** `mishipay_dashboard/serializers.py` (1,439 lines)

Key serializers:

| Serializer | Purpose |
|-----------|---------|
| `OrderSerializer` | Full order details: basket, prices, customer, payments, risk score, transaction ID |
| `BasketItemSerializer` | Item details with tax, discounts, ratings, addons |
| `RefundOrderNewSerializer` | Refund creation validation: item quantities, eligibility |
| `RefundParamsSerializer` | Legacy refund validation |
| `ItemInventorySerializer` | Product details with country, category, discount info |
| `DashboardStoreSerializer` | Store metadata for dashboard UI with features and promo version |
| `PaymentsSerializer` | Payment normalization with PSP mapping |
| `ManualVerificationSessionSerializer` | Verification session with reviewer tracking |
| `DashboardChatSerializer` | LLM query validation |
| `DashboardEmailSerializer` | Email sending parameters |
| `UpdateCCOrderStatusSerializer` | Click & Collect state transitions |

## Pagination

**File:** `mishipay_dashboard/pagination.py` (20 lines)

| Class | Page Size | Max |
|-------|-----------|-----|
| `OrderPagination` | 10 | 100 |
| `ItemPagination` | 10 | 100 |
| `CashReportPagination` | 10 | 100 |

## Configuration & Constants

**File:** `mishipay_dashboard/app_settings.py` (~100+ lines)

Key store-type configuration maps:

| Map | Purpose |
|-----|---------|
| `REFUND_PSP_MAPPING` | Routes refunds to store-specific PSP handlers |
| `UPDATE_BASKET_MAPPING_FOR_REFUND` | Routes basket updates to store-specific handlers |
| `PRINT_ORDER_RECEIPT_DETAILS_FUNCTION_MAP` | Store-specific receipt formatters |
| `REFUND_ORDER_FULFILMENT_FUNCTION_MAP` | Post-refund fulfillment handlers (Virgin, Dimona, MySalonStop) |
| `REFUND_OLD_DASHBOARD_STORE_TYPES` | Stores still using legacy refund flow |
| `PARTIAL_REFUND_BLOCKED_STORES` | Stores where partial refunds are disabled (e.g., SaturnSmartPay) |
| `REFUND_DISABLE_PAYMENT_METHODS_BY_STORE_TYPE` | Payment method restrictions per retailer |

**File:** `mishipay_dashboard/config.py` (15 lines)

```python
SPLIT_KEY = '-#$@'  # Delimiter for compound field values
```

## Management Commands

| Command | File | Purpose |
|---------|------|---------|
| `auto_verify_orders` | `auto_verify_orders.py` (~80 lines) | Auto-verifies COMPLETED orders after 1 hour; excludes orders with active refunds; Slack error notifications |
| `analytics_reconciliation` | `analytics_reconciliation.py` | Reconciles aggregated analytics across regions; compares order counts, totals, payment statuses |

**Usage:**
```bash
python manage.py auto_verify_orders --retailer-store-id=X --store-type=Y [--start-date=] [--end-date=]
```

## Admin Interface (mishipay_dashboard)

**File:** `mishipay_dashboard/admin.py` (16 lines)

| Admin Class | Display | Filters | Search |
|------------|---------|---------|--------|
| `RefundOrderAdmin` | refund_order_id, order_id, store_id, mode, refund_amount, status, created | created, store_id, status | refund_order_id, order_id, store_id (exact) |

## Test Coverage (mishipay_dashboard)

**14 test files, ~300KB total**

| Test File | Focus |
|-----------|-------|
| `test_refund_order_new.py` | Payment data extraction, refundable amount calculations |
| `test_dashboard_refund.py` (66KB) | Comprehensive refund flows, tax validation, store-specific logic |
| `test_get_store_orders_v4.py` (40KB) | Order listing with filters, date ranges, price ranges |
| `test_new_order_verification_screen.py` (51KB) | Manual verification session management |
| `test_seventh_heaven_refund_with_addons.py` (29KB) | Addon item refund handling |
| `test_aggregated_analytics.py` (37KB) | Analytics aggregation and reporting |
| `test_psp_refund_functions.py` | Payment processor refund testing |
| `test_llm_chat.py` (15KB) | LLM chat functionality |
| `test_get_vengo_blocked_users.py` (21KB) | Vengo integration testing |
| Additional tests | Email, deactivate user, order status updates |

---

## Dependencies

### Internal Dependencies

| Dependency | Used For |
|-----------|----------|
| `dos.models` (Store, Customer, Order, Item, StoreTransactionSequenceNumber) | Core data models |
| `dos.store_map.STORE_TYPE_MAP` | Store-type polymorphic dispatch |
| `mishipay_retail_orders.models.Order` | Order data |
| `mishipay_retail_payments.models.Payments` | Payment data and PSP types |
| `mishipay_items.models` (ItemInventory, BasketItem, ManualVerificationSession) | Item/basket data |
| `mishipay_core.common_functions` | Generic utilities, response structure |
| `mishipay_emails` | Receipt generation, email sending |
| `mishipay_cashier_kiosk.pagination` | SmallResultSetPagination |

### External Services

| Service | Integration | Purpose |
|---------|------------|---------|
| Adyen | REST API via `AdyenSessionView` | Payment sessions |
| Stripe | Via `mishipay_retail_payments` | Payment refunds (Leroy Merlin) |
| Anthropic Claude | Via LangChain in `chat.py` | LLM-powered analytics chat |
| Redis | Direct connection | RFID EPC tag tracking |
| Mixpanel | Via analytics utilities | Event tracking |
| Firebase | Via `mishipay_core` | Push notifications |

## Notable Patterns

1. **Dual-module architecture** — Authentication/user management (`dashboard`) is separated from operational APIs (`mishipay_dashboard`). Both serve the same UI but have distinct responsibilities.

2. **Four-version order listing** — `v1/order/list/` through `v4/order/list/` all remain active. V4 is the current version with comprehensive filtering. Backward compatibility is maintained but adds maintenance burden.

3. **Store-type dispatch** — Refund processing, basket updates, receipt generation, and fulfillment are all dispatched via store-type configuration maps. This is extensible but relies on lowercase string matching.

4. **Multi-payment refund support** — `RefundOrderNewView` handles orders with multiple payment methods, splitting refund amounts across PSPs.

5. **Dual password storage** — `User.password` and `auth_user.password` are stored separately. Changes must update both, creating a synchronization risk.

6. **DDF-specific code paths** — Dubai Duty Free has custom authentication (QR + airport pass validation), tab permissions, and kiosk workflows scattered through both modules.

7. **LLM database access** — The chat feature introspects Django model schemas and allows Claude to query a managed database replica. SQL injection prevention is regex-based, which is not comprehensive.

8. **Legacy code coexistence** — Both modules maintain legacy code paths alongside new implementations (legacy refund vs `RefundOrderNewView`, V1 vs V4 order listings, `dashboard/utils/orderManager.py` remote API calls).

9. **Atomic transactions** — Complex operations like refunds use `@transaction.atomic` decorators for rollback on failure.

10. **Token-based onboarding** — New users are created via invite tokens that carry pre-configured permissions, allowing managers to control what new staff can access.

## Open Questions

- Why is password stored in both `User.password` and `auth_user.password`? Should this be consolidated to `auth_user` only?
- Are legacy utilities in `dashboard/utils/` (`orderManager.py`, `chart.py`, `customerManager.py`) still used? They appear to make remote API calls to hardcoded URLs.
- Is the regex-based SQL sanitization in `chat.py` sufficient for production use? Parameterized queries would be more robust.
- How are concurrent refund requests on the same order handled? No explicit locking mechanism was found.
- What happens if a PSP refund succeeds but the basket update fails (in `RefundOrderNewView`)?
- Is `hide_customer_info` on the User model enforced in serializers, or only in the UI?
- Are there brute-force protections on login endpoints? No rate limiting was observed.
- How is `end_date_time` on User managed? It requires manual date setting with no automated expiration job.
- Why does `DashboardVerificationAllSessions` have Eroski-specific logic (showing all baskets until verified) rather than using a configurable approach?
- The `REFUND_OLD_DASHBOARD_STORE_TYPES` list — are these stores being migrated to the new refund flow, or will legacy support persist indefinitely?
