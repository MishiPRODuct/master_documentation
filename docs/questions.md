# Open Questions

> Consolidated open questions discovered during documentation. Questions are removed once answered in subsequent tasks.

## Architecture (from T00)

- What is the exact Kafka topic structure and message format?
- How does the payment gateway (`pay-gateway`, Node.js) interact with the Python payment service (`mishi-pay`)?
- What are the specific Keycloak realm/client configurations?
- How does the inventory service (separate repo) integrate with the monolith?

## Tech Stack (from T00)

- What specific Kafka topics exist and what events flow through them?
- What is the exact role split between `ms-tax` (gRPC, port 9000) and `mishi-tax` (HTTP, port 8008)? Likely a legacy-to-new migration.
- How is the `inventory-common` private package (from GitHub) structured and used?
- What is the `explorer` Django app in INSTALLED_APPS? (Possibly Django SQL Explorer for ad-hoc queries)

## Promotions Service — Overview (from T01)

- ~~What is the functional difference between V1 (`promo_logic.py`) and V2 (`promo_logic_v2.py`) evaluation engines?~~ **Answered in T03**: See [Business Logic](./promotions-service/business-logic.md#v1-vs-v2-differences). Key differences: V1 has pack item support and processes basket promos separately after all layers; V2 processes basket promos inline within each layer, has 4 cluster types (vs 3), and handles cross-layer item propagation differently.
- How does the monolith communicate with this service — direct HTTP or via the API gateway? (Based on docker-compose, likely direct HTTP over `mpay_net`)
- ~~What is the `mishipay_util.py` utility used for?~~ **Answered in T05**: Contains `elapsed_interval()` (time formatting) and `send_slack_message()` (Slack notifications). The `send_slack_message()` function has a hardcoded Slack bot token — a security concern. Neither function is imported within the promotions service itself; `elapsed_interval()` is only used by the `delete_promos` management command.

## Promotions Service — DevOps (from T01)

- Is the Azure Pipeline the primary production deployment path, with GitHub Actions as secondary? Both run on push to master but only Azure handles Helm.
- What does the `adolinux` self-hosted agent pool consist of?
- What Kubernetes cluster/namespace is the service deployed to?
- Is the `postgresql.conf` in this repo actually used, or is it a leftover from a shared configuration approach?

## Promotions Service — Data Models (from T02)

- What is the structure/schema of `PromoApplicationFilter.filters_json`? No serializer or admin registration exists for this model.
- The `BasketItem` serializer has a typo: field is named `catgories` instead of `categories`. Is this intentional for backwards compatibility or a bug?
- ~~How are cascading deletes handled when a PromoHeader is deleted, given no FK constraints exist?~~ **Answered in T05**: See [API Reference — Deletion Patterns](./promotions-service/api-reference.md#deletion-patterns). Three approaches exist: (1) Soft-delete via `PromoHeaderStore.soft_delete=True` (preferred, used by detail endpoint and bulk transaction `"d"` action); (2) Raw SQL hard-delete in `PromotionsByStore.delete()` (bypasses ORM); (3) ORM cascade delete in `promo_delete_job()` and `delete_promos` management command (children-first order: nodes -> groups -> stores -> special promos -> headers).

## Promotions Service — Business Logic (from T03)

- ~~What triggers the selection between V1 (`PromoLogic`) and V2 (`PromoLogicV2`) at the API/view layer?~~ **Answered in T05**: See [API Reference](./promotions-service/api-reference.md#post-api10promotionsevaluate--evaluate-basket). Selection is **per-request** via `request.data.get("version", "v1")` in `views.py:659`. Default is V1; only `"v2"` (case-insensitive) triggers V2. The suggestions and recommendations endpoints always use V1.
- Is V1 deprecated or still actively used for specific retailers? When was V2 introduced?
- Why does the ORM-based `promo_db_helper.py` still exist alongside the raw SQL version? Is it a legacy fallback or used in specific contexts (e.g., testing)?
- How do the `apply_on` types (`APPLY_ON_BASKET`, `APPLY_AS_CASHBACK`, `APPLY_AS_TENDER`, `APPLY_AS_POSLOG_EXCLUSION`) affect downstream processing? They don't appear to change the core discount calculation.
- ~~What is the exact behavior of `math_util.round_custom()` for different retailers?~~ **Answered in T04**: See [Business Logic — Math Utilities](./promotions-service/business-logic.md#math-utilities). `round_custom()` uses `ROUND_HALF_UP` for all retailers (the `retailer` parameter is accepted but unused). A second function `round_custom2()` exists with Eroski-specific floor rounding but is not imported by any family class. MUJI has special 2-decimal rounding in basket threshold only.
- The `is_valid_for_promo_application()` pruning method always returns `True` with commented-out logic. Is additional validation planned or handled elsewhere?

## Promotions Service — Promotion Families (from T04)

- The class `RequisiteGroupsWithDiscounteTargetPromoFamily` has a typo in its name (missing 'd' in "Discounted"). Is this known? Renaming would require updating `PromoFamilyExecuter`'s class map.
- Debug `print()` statements with "dora ->" prefix exist in `requisite_groups_with_discounted_target_promo_family.py:346` and `evenly_distributed_multiline_discount_promo_family.py:526,564`. Are these intentional or should they be removed?
- The `EasyPromoFamily.calculate_discount_by_picking_homegeneous_items()` and `RequisiteGroupsWithDiscounteTargetPromoFamily.calculate_discount_by_picking_homegeneous_items()` legacy methods operate on grouped items rather than ungrouped. Are they still called from any code path, or are they dead code?
- The `EvenlyDistributedMultilineDiscountPromoFamily` states it only works for "non-variant cases" (single item type per group). Is variant support planned?
- Why does `calculate_discount_amount_for_each_item()` have identical branches for MUJI vs other retailers (`promo_family_factory.py:127-130`)? Was retailer-specific rounding partially removed?
- The MUJI "Free tea" hack in `BasketThresholdPromoFamily` (`basket_threshold_promo_family.py:84-88`) references specific promo title strings and promo_id_retailer prefixes. Is this a permanent feature or a temporary override?
- Why is `round_custom2()` in `math_util.py` never imported? Is it a legacy function awaiting removal, or used by another codebase?

## Promotions Service — API, Utilities & Tests (from T05)

- The evaluate request serializers (`PromoEvaluateRequestSerializer`, `Basket`, `BasketItem`) are defined in `serializers.py` but not used by `PromotionEvaluate.post()` — it passes `request.data` directly to the engine. Were these originally used for validation and bypassed for performance, or are they intended for future use?
- `PromotionsByStore.delete()` uses raw SQL cascade deletion without checking if promotions are shared with other stores (unlike the soft-delete path). Is this endpoint called from the monolith, or only used for dev/admin cleanup? The code has a `@todo` comment acknowledging this.
- The `PromotionSuggestionListByNodes` and `PromoBasedItemRecommendations` endpoints always use V1 (`PromoLogic`). Should they support V2?
- `mishipay_util.py:12` contains a hardcoded Slack bot token (`xoxb-...`). This is a security concern — should this be moved to an environment variable or secrets manager?
- `send_slack_message()` in `mishipay_util.py` is not imported anywhere within the promotions service. Is it called from external scripts or is it dead code?
- Several test filenames contain typos: `availabiltiy`, `applicalble`, `apllication`. Are these known, or should they be corrected?

## MishiPay Platform — Core Module (from T07)

- How are the `retailer_entity_formatters` loaded and invoked? No explicit import chain was found in the core module itself — they may be called from item import or promotion update pipelines in other modules.
- Is the `ActionConstraint` / `EntityActionConstraints` constraint engine still actively used, or is it legacy? The `eval()` approach suggests early development patterns and raises security concerns (arbitrary code execution from database-stored expressions).
- Are there plans to split `common_functions.py` (4,200+ lines, 97 functions) into focused modules? Functions like `generate_context_for_receipt` (~1,550 lines) are strong candidates for extraction.
- The `SlackClient` endpoint (`POST v1/send-slack-msg/`) has no authentication. Is it protected by network-level access controls (e.g., internal-only routing)?
- The `KeycloakMigrationApiView` returns user data (GET) and validates passwords (POST) without authentication. Is this secured by being temporary/removed in production, or protected at the infrastructure level?
- The Firebase admin SDK credentials JSON file is committed in `firebase_common/mishipay-a5e8e-firebase-adminsdk-dwx39-e00bf8bda5.json`. Is this a non-sensitive service account, or should it be in a secrets manager?
- `LoggingUserIdMiddleware` (`logging/middleware.py`) tracks DB query counts and response times but uses `print()` instead of the logging framework. Is this intentionally enabled in production?

## MishiPay Platform — Project Structure & Settings (from T06)

- Why is the Memcached configuration (`settings.py:584-597`) unconditionally overridden by `LocMemCache` (`settings.py:599-604`)? Is this intentional or a debugging leftover?
- Is the Decathlon `bundleid` block in `SentryInterceptMiddleware` (`middleware.py:107-108`) still needed, or is it a temporary measure from a past incident?
- How are the `retailer_configurations/` Python files loaded and used at runtime? Many directories contain empty files. No import mechanism was found in `settings.py` itself.
- Multiple production-looking API keys and tokens are hardcoded in `settings.py` (Cloudinary API secret, Stripe live key, Adyen live keys, Sendinblue SMTP password, social auth secrets, etc.). Are these overridden by environment variables in production, or are they the actual production credentials?
- The `explorer` app (Django SQL Explorer) is enabled in all environments with connection to the default database. Is access restricted by additional auth rules beyond the Django admin?
- The `_test.py` file at the project root is an ad-hoc interactive Mango analytics script (not a test) with hardcoded store UUIDs. Is this still used or should it be removed?

## Authentication & Configuration (from T08)

- Are both guest user creation paths still active? The `mishipay_authentication` module (using `mishid` / `guest_user_identifier`) and `MishiKeycloakAuthentication` middleware (using `mishipay_id`) appear to serve different client versions. Which is the current/preferred path? Are there plans to deprecate the older module?
- Why is `AllowAny` the global default permission for the REST framework? This means every endpoint is publicly accessible unless explicitly restricted. Is this intentional (relying on Kong for access control) or an oversight?
- Is `complete_half_backed_login()` implemented? The method is referenced in `register_or_login_service.py:182` but not defined in the `RegisterOrLoginUserAsPerServiceType` class. Is it inherited from a parent class or is it dead code?
- Are the `mishipay_email_customer` and `decathlon_customer` auth definitions actually used? The `GuestCheckoutInputSerializer` only accepts `mishipay_guest_checkout` as a valid service type. Are these definitions used by another endpoint or are they dead code?
- What GlobalConfig keys exist in production? The config system is flexible but the key namespace is undocumented. A listing of actual keys and their purposes would be valuable.
- Is the Nedap integration still active? The hardcoded Bearer token (`d8baad55c2a9fb590ff7ceac99d3c5417b6926a528a7aa7ab33f35`) and SATO ngrok tunnel URLs in `store_type.py` suggest these may be legacy integrations.
- How is `ProcessStatsMiddleware` configured in production? It sends Datadog metrics for specific routes but only for `HTTP_200_OK` and `HTTP_201_CREATED` responses. Is this intentional, or should error latencies also be tracked?

## Items Module — Models & Serializers (from T09)

- Is the V2 `Category` model (MPTT, per retailer) actively used in production, or is it a work-in-progress? No migration of `ItemCategory` references to `Category` was observed.
- The `stock_quantity` field on `BasketItem` has a TODO comment saying it's "pointless" since stock checks use `ItemInventory` / inventory service. Should it be removed?
- `applied_offer` on `BasketItem` has a TODO marked for deletion. Is it still read anywhere?
- The `DISCOUNT_CONSTRAINT_MAP` is hardcoded in `__init__.py`. Is this used in production, or has it been superseded by the promotions microservice?
- Why does `BasketEntityAuditLog.modified_monetary_value` use 4 decimal places while `original_monetary_value` uses 2?
- The `BasketPromotionSerializer` and `BasketItemPromotionSerializer` have identical code. Is this intentional or should they be consolidated?
- `BasketVerboseSerializerV2` appears to be defined twice in `serializers.py` (lines ~1643 and ~1856). Which definition takes precedence?

## Items Module — Views & APIs (from T10)

- The constraint-based permissions (`ItemScanPermission`, `AddUpdateBasketPermission`, `AddUpdateWishlistPermission`) are defined in `permissions.py` but not applied to any view in the current codebase. Are they used elsewhere (e.g., via `app_settings.py` or middleware)?
- V1 APIs use `ItemManagementClient` (HTTP hop to internal service APIs), while V2 APIs dispatch directly. Is V1 being deprecated in favor of V2, or do both tiers serve different clients?
- `ItemSearch` is restricted to `NudeFoodStoreType` only. Is there a plan to expand search to other store types, or does search happen via the inventory service for most retailers?
- `ManualVerification` has no authentication — the `basket_id` in the URL is the only access control. Is this a security concern?
- `BulkAddItemToBasket` (V1) at path `v1/bulk-add-item-to-basket-old/` — is this still in use, or has it been fully replaced by `BulkAddItemToBasketV2`?
- The `InventoryUpdate` endpoint (`v1/inventory/update/`) has no `IsAuthenticated` requirement, only `ActiveStorePermission` + `InventoryUpdatePermission` + `UserStorePermission`. How is it authenticated — API key? Store token?

## Items Module — Business Logic (from T11)

- `mpay_promo.py:645` hardcodes `store_timezone` to `"UTC+1:00"` for all stores sent to the promotions microservice. Is this overridden at a different layer, or is this a bug for non-UTC+1 retailers?
- `PromotionBatchOperation.load()` (`mpay_promo.py:302`) returns a `PromotionEntityError` instead of raising it, meaning callers silently receive an Exception object. Is this a known bug?
- The deprecated category format in `mpay_promo.py` (lines 527–537) — `item_category`, `sub_category`, `mixmatch_id`, `promo_categories` — is marked "Never use for new retailers." Which existing retailers still use this format, and are there plans to migrate?
- `LoyaltyCLientError` in `helpers/loyalty.py:7` is misspelled (`CLient` instead of `Client`). Is this known? A rename would require updating all callers.
- `item_scan_functions.py` uses `print()` instead of `logger` in the Saturn and Cisco scan functions. Are these intentional debug statements or should they use the logging framework?
- `item_scan_functions.py:464` hardcodes Saturn's VAT rate to `19.0`. Is this configurable per store in production?
- Compass and Eroski promotion handlers (`compass_promotions/handlers.py` and `eroski_promotions/handlers.py`) are structurally identical — is there a plan to consolidate them into a shared base?
- The Picwic SOAP integration (`helpers/picwic_promotions.py`) has hardcoded SOAP namespace URNs and no XML validation or retry logic. Is this integration still active, or has Picwic migrated to the promotions microservice?
- The `mishipay_items` module has no `tasks.py` or `signals.py` — all processing is synchronous. For large baskets with many items, is promotion evaluation latency a concern?

## Coupons, Special Promos & Promotions (from T12)

- Is `is_mishipoly_promo_applicable2()` (`special_promo_helper.py:166`) actively called from any code path, or is it dead code alongside `is_mishipoly_promo_applicable()` (which always returns `False`)? The cross-retailer loyalty concept (two orders from different store types in a cycle, checking `MPLDN1000` qualifier) appears unused.
- The `get_basket_level_promotions_label_details()` function has a TODO hack for BWG orders with missing `group_qualifiers` data (`special_promo_helper.py:447`). Is this still needed, or have all missing receipts been sent?
- `RetailerCoupons.get()` (`views.py:110-121`) does not return a `Response` when `available_coupons` filtering produces an empty list — the fallthrough at line 121 handles non-200 status from the loyalty service but the 200 path with empty results may produce unexpected behavior. Is this a known issue?
- The Shopify promotion import command (`shopify_promotion_import.py`) is 55KB — significantly larger than other commands. What Shopify API integration complexity does it handle?
- Several management commands in the `promotions` module use `print()` instead of `self.stdout.write()` or proper logging (e.g., `management/utils/coupons.py:28`). Is this intentional for operator feedback during manual imports?
- The `emmas_garden_loyalty_import` command creates 4 coupon types based on discount/tax combinations. Is Emma's Garden still an active retailer, or is this command legacy?
- `STORE_TYPE_OVERRIDE_REFERRAL_COUPON` at `coupons_helper.py:28` contains only `{"RotanaStoreType"}`. Is this set expected to grow, or is this a one-off override?

## Emails & Receipt Generation (from T13)

- Are emails sent through `EmailHelper`'s rendering pipeline or is there a separate email sending service? The `mishipay_emails` module defines configuration and templates but contains no actual email sending logic (no SMTP calls, no SendGrid/Sendinblue API integration visible). The actual sending mechanism must be in another module.
- How is `GeneratePosReceipt` (`printer_receipt_template.py`) invoked? It doesn't integrate with `StrategyFactory` or `mapping.py`. Is it used by a different code path than `PrintReceipt` (e.g., a separate kiosk hardware integration)?
- The `COUNTER` and `PAYMENT` receipt types are defined in `ReceiptType` but have no entries in `STRATEGY_MAPPING`, no test coverage, and no configuration fixtures. Are they used in production?
- The `CAMPAIGN` receipt type is only used by `DubaiDutyFreeOrderStrategy.get_additional_receipts()` for Visa cardholders. Is this feature still active?
- `print_receipt.py:102` hardcodes a default logo URL to `https://png.pngtree.com/...` (a stock image site). Is this actually used as a fallback in production, or is it always overridden by `get_kiosk_receipt_store_logo()`?
- Multiple occurrences of `if receipt_str is not ""` (identity comparison instead of `!=`) throughout `print_receipt.py` and strategy files. This relies on Python string interning and is not guaranteed. Is this a known code quality issue?
- `CartersOrderStratergy` and `seventh_heaven_order_stratergy.py` both have "Stratergy" typos in class/file names. Are these known? Renaming would require updating `mapping.py` imports.

## Payment Alerts & Settlement Reconciliation (from T14)

- Authentication is commented out on all REST endpoints in `mishipay_alerts/views.py`. Are these endpoints protected at the infrastructure level (e.g., internal-only network routing), or is this a security gap?
- `mishipay_alerts/apps.py` defines `MishipayCouponsConfig` instead of `MishipayAlertsConfig` — a copy-paste artifact from `mishipay_coupons`. Does this cause any runtime issues?
- `inventory_promo_poslog_alert_helper.py:19-21` has a bug: references `store.retailer_store_id` before `store` is assigned in the `else` branch. This would cause a `NameError` when `store_id` is not provided. Is this code path ever executed?
- The `RESOLVED` status constant is defined twice in `mishipay_alerts/models.py` (lines 9 and 19) with the same value. Is the intent to have distinct "alert resolved" vs "recon resolved" statuses?
- Swish refund processing is disabled in `quickpay_psp_settlement_by_files.py` — how are Swish refunds reconciled?
- `quickpay_psp_settlement_deprecated.py` has its `process_quickpay_payouts` function entirely commented out. Should this file be removed?
- Are the hardcoded file paths in `quickpay_psp_settlement_by_files.py` (`/app/reconciliation/flying_tiger/{region}/live/`) specific to the production container filesystem?
- The PSP reference ID prefix heuristic (`ch_`/`pi_` → Stripe, `pay` → Checkout) in `mishipay_settlement_report_final.py` — is this reliable for all current PSPs, or could it misclassify newer providers?

## Subscriptions (from T14)

- `"Coffee Subscription"` is hardcoded in `mishipay_subscription/views.py:68,223`. Is there a plan to support multiple subscription types, or is this intentionally a single-product feature?
- `SubscriptionSerializer` (ModelSerializer) in `mishipay_subscription/serializers.py` lacks a `fields` attribute and is not imported anywhere. Is this dead code?
- `datetime.fromtimestamp()` is called without timezone in the webhook handler (`mishipay_subscription/views.py:166`). Should this use `datetime.fromtimestamp(ts, tz=timezone.utc)` for consistency?
- `Decimal(data["data"]["amount_paid"] / 100)` in `mishipay_subscription/views.py:199` — integer division occurs before Decimal conversion, potentially losing precision. Should be `Decimal(data["data"]["amount_paid"]) / Decimal(100)`.
- `subscription_helper.py` uses bare `except Exception` (lines 46, 100) which silently swallows all errors and defaults to `OPEN_FOR_TRIAL`. Should this be narrowed to `CustomerSubscriptionMapping.DoesNotExist`?
- `SubscriptionServiceClient` (`ms_pay_subscription.py`) has no timeout, retry, or error handling on `requests.post()` calls. Is there retry logic at the MS Pay service level?
- The empty `test_subscription_flow_with_item_discount.py` suggests planned integration tests for the discount flow. Is this still planned?
- Two different `Customer` model imports appear across test files: `dos.models.customer.Customer` vs `mishipay_retail_customers.models.Customer`. Are these the same model re-exported, or is there an inconsistency?

## Referrals (from T15)

- What is the structure and deployment of the MS Loyalty Service? Is it a shared service or MishiPay-specific?
- How are referral coupons integrated with the promotions microservice? The test shows integration with `PromotionBatchOperation` — does the loyalty service create promotions directly?
- Is there a maximum number of referral codes per campaign? `source_max_user_use_count` and `target_max_user_use_count` limit per-user usage, but overall campaign limits are unclear.
- Are referral campaigns actively used in production? Test data uses MUJI-GB only.
- `terms_and_condtions_link` (missing 'i' in "conditions") is present throughout the API contract and tests. Is this a known typo inherited from the loyalty service?
- Only `ReferralCampaignLatestOfAllRetailers` has a 30-second timeout. Other endpoints make HTTP calls with no timeout — is this a risk for loyalty service outages?

## Dashboard — User Management (from T15)

- Why is password stored in both `dashboard.User.password` and `auth.User.password`? Should this be consolidated to `auth_user` only?
- Are legacy utilities in `dashboard/utils/` (`orderManager.py`, `chart.py`, `customerManager.py`, `storeManager.py`) still used? They appear to make remote API calls to hardcoded URLs.
- Are there brute-force protections on the `LoginView` and `PasscodeLoginView` endpoints? No rate limiting was observed.
- How is `User.end_date_time` managed? It requires manual date setting with no automated expiration job.
- Is `retailer_staff_pin` stored as plaintext in the database? The `TextField` type and 6-digit format validation suggest no hashing.
- The `PasscodeResetView` only validates 6-digit PIN format — are there complexity requirements (e.g., no sequential digits, no repeated digits)?
- How does third-party login (`_client_specific_login()` in `LoginView`) prevent unauthorized account creation? Auto-creation of users without admin approval seems risky.
- `apps.py` defines `DashboardConfig` with `name='dashboard'` — could this conflict with third-party Django dashboard packages?

## Dashboard — Operations API (from T15)

- Is the regex-based SQL sanitization in `chat.py` sufficient for production use? It blocks keywords like DROP/ALTER/INSERT but may miss injection via subqueries, comments, or encoding tricks. Parameterized queries would be more robust.
- How are concurrent refund requests on the same order handled in `RefundOrderNewView`? No explicit locking mechanism was found — could two staff members process overlapping refunds?
- What happens if a PSP refund succeeds but the basket update fails (in the new refund flow)? Is there a reconciliation process?
- `RefundOrderNewView` POST method is ~1,665 lines. Are there plans to refactor into service classes?
- Why does `DashboardVerificationAllSessions` have Eroski-specific logic (showing all baskets until verified) rather than using a configurable approach?
- The `REFUND_OLD_DASHBOARD_STORE_TYPES` list — are these stores being migrated to the new refund flow, or will legacy support persist indefinitely?
- `mishipay_dashboard/apps.py` — does it have a copy-paste naming issue similar to `mishipay_alerts` (which defines `MishipayCouponsConfig` instead of `MishipayAlertsConfig`)?
- Authentication is absent on some mishipay_dashboard views that rely solely on `DashboardPermission` group check. Is the `DashboardUser` group membership sufficient, or should specific action permissions (e.g., `can_refund`) be enforced at the view level?
- `GetDashboardOrderDetails` tracks dashboard access count and sets `dashboard_accessed` flag after 2nd access — what is the business purpose? Is this specific to Riyadh Tameer only?
- Tax calculation logic in refund processing has store-type conditional branches (WHSmith price-based vs equal division). Where is the business requirement for these different approaches documented?

## Retail Orders — Models & Serializers (from T16)

- The `Order` model has a no-op `__init__` override (`models.py:141-142`) — `super(Order, self).__init__(*args, **kwargs)` does nothing extra. Is this leftover from a removed customization?
- `CreateOrderPermission` uses the `EntityActionConstraint` `eval()`-based engine (passing `locals()`) — is this constraint system actively used for order creation in production, or is it legacy? `CreateOrderPermissionV2` skips it entirely.
- `get_payment_method()` serializer writes to DB and sends Slack alerts — is this intentional side effect acceptable in a serialization path? It could cause unexpected writes during read operations.
- `complete_order.py` only has Eroski logic. Are other retailers' completion steps handled entirely within `complete_status()` or via `ORDER_FULFILMENT_FUNCTION_MAP`?
- The `sto_order_fulfilment` function is imported twice in `app_settings.py:79,81` — is this a copy-paste artifact?
- `check_sum_of_item_prices_with_payment_captured_amount` skips stores in `BASKET_AUDIT_LOG_NOT_CREATED_STORETYPES` — which store types are in this list and why don't they create audit logs?
- The `OfflineMonolithSyncObjectSerializer` suggests full offline → online sync capability. How widely is offline mode deployed?
- WhatsApp receipt sending is supported but appears to be a single view for specific stores. Which stores use WhatsApp receipts?

## Retail Orders — Views & Logic (from T17)

- `CreateOrderV2` has a bug at `views.py:1829`: `r_long = request.GET.get("r_lat")` copies latitude into longitude. Is this known? Longitude data is never captured for V2 orders.
- `GetOrderDetailsV2` has no authentication (`authentication_classes = []`). Is this protected at the infrastructure level (Kong API gateway), or is this a security gap?
- Multiple views update customer records as a GET side effect (`GetOrderDetailsV2`, `GetOrderDetailsV3`, `GetOrderDetailsGuestView`). Is this intentional for email capture?
- `HudsonTransactionLookupView` has a bare `except:` clause at line 1060 and no authentication/permissions. Is this endpoint still in use?
- `StaffOrderReceipt` at ~530 lines is the most complex single view. Are there plans to complete the migration to the Strategy pattern (`NEW_PRINT_RECEIPT_CONFIGURATION_STORES`)?
- VenGo exit gate views use `verify=False` (SSL bypass) and have default credentials `'Test1234'` in code. Are these overridden in production?
- `order_verification.py` is a no-op stub (always returns `True`). Is actual verification logic implemented elsewhere, or is this feature disabled?
- `set_unsold_decathlon()` appears to use the same `deactivate_epc_url` as `set_sold_decathlon()`. Should it use a separate reactivation endpoint?
- `generate_tax_service_response()` in offline sync generates Flying Tiger Copenhagen-specific tax responses only. Is offline sync deployed only for Flying Tiger, or does this need generalization?
- The offline sync flow creates User records with `is_guest_user=True`. Are orphan users cleaned up?
- SAP transaction log generation (`sap/transaction_logs.py`, 866 lines) implements SAP POS 2.3 fixed-width records. Which retailers use SAP TLogs and how are they delivered (SFTP, API)?
- The Emaar API client sends daily/monthly sales data. How many Emaar stores are active, and is the monthly report generated via management command?

## Retail Analytics & Jobs (from T21)

- ~~How and when is `AggregatedAnalytics` populated?~~ **Answered in T30**: The `ExternalPaymentsAnalyticsHandler` (in `mishipay_retail_payments/pubsub/handlers/`) consumes from the `queuing.external.payments.analytics.json` Kafka topic via `PaymentOnlyClientsAnalyticsConsumer`. It creates/updates `AggregatedAnalytics` records using an upsert pattern (try insert, on `IntegrityError` update existing). Monetary values arrive in cents and are divided by 100.
- The `AnalyticsDashboard` view (V1, `views.py:80`) has no `permission_classes`. Is this intentional, or should it have `DashboardPermission`? It requires `analytics_filter_type` in the query params but doesn't verify the caller has access to the requested stores.
- `ItemScannedAnalyticsV2` exists in `views_v2.py` but is not wired into any URL. Is there a plan to add it, or does V1 `ItemScannedAnalytics` remain the preferred endpoint?
- The Exchange Rate API key `24f8b7beb83be48d1eaa9ea4` is hardcoded in both `views.py:384` and `views_v2.py:103`. Should this be in environment configuration or a secrets manager?
- `PaymentObjectClass` (`payment_object.py:22`) uses `datetime.now()` instead of `django.utils.timezone.now()`. This produces naive datetime objects when `USE_TZ=True`. Is this intentional?
- The `JourneyTypes.NOT_AVAILABLE` constant in `__init__.py` has a trailing comma making it a tuple `('not_available',)`. The model works around this by using inline choices. Is the constant used elsewhere?
- How are the `manual_run` and `manual_cancel` fields on the `Task` model used? No code was found that reads these flags to trigger task re-execution or cancellation.
- `RecommendedItems` model has FKs to `Store` and `Basket` but no builder class, no views, and no admin registration. How is it populated?
- The V1 URL paths (`v1/transaction-retailer-breakdown/`, etc.) silently serve V2 view classes. Both `v1/` and `v2/` paths share the same Django URL names, causing potential URL name collisions. Is this intentional?

## Cashier Kiosk (from T22)

- The shift-end Kafka event is only published for `FlyingTigerStoreType`. Are other retailers expected to receive these events, or does Flying Tiger have unique POS integration requirements?
- `create_kiosk_staff_shift_activity()` does not use `F()` expressions or `select_for_update()` when incrementing shift counters (`utils.py:20-24`). Are concurrent transactions on the same shift a real-world scenario?
- `sales_count` is incremented for both sales AND refund activities (`utils.py:20`). Should this be renamed to `cash_activity_count` for clarity?
- `verify_active_session_exists()` (`utils.py:76-82`) is not imported or called anywhere in the codebase. Is this dead code?
- `StartShift.post()` and `EndShift.post()` use bare `except:` clauses (`views.py:40-41`, `views.py:96-98`) to handle missing `request.store`. Should these be narrowed to `except AttributeError:`?
- `end_time` is set via `datetime.utcnow()` (`views.py:142`) producing a naive datetime, while `start_time` uses timezone-aware `auto_now_add`. Could this cause issues?
- `_validate_denomination_breakup()` uses `float` arithmetic for currency summation (`serializers.py:10-12`). Has this caused precision issues in production?

## Third-Party Integrations & Microservices Client (from T25)

- The `DecathlonSignup` class in `views/decathlon.py:577` references undefined functions `get_access_token_from_code` and `get_email` (marked `# noqa: F821`). Is this legacy code path still used, or has it been fully superseded by the OAuth2 callback flow?
- `AuthProviders` defines Google, Facebook, and Red_Login, but no provider implementations exist beyond OpenID/Decathlon. Are these providers planned, deprecated, or implemented elsewhere?
- `ThirdPartyLogin.extra_data['israel_discount_applicable']` has special handling in `login_user()` (`views/decathlon.py:126-134`) — a boolean toggle with unclear logic (`if not discount_applicable and not isinstance(discount_applicable, bool)`). What is the business rule for Israel-specific discounts?
- `tax_pb2.py` and `tax_pb2_grpc.py` are generated gRPC stubs, but the actual Tax service implementation was not found in the explored source paths. Where is the Tax gRPC server implemented?
- The `StoreLocation` protobuf message is defined but not referenced by any RPC method in the Tax service. Is it used elsewhere or reserved for future use?
- The `microservices_client` URL targets use `_as_service` suffix naming (e.g., `scan_an_item_as_service`, `create_order_as_service`). Are these distinct endpoints from the customer-facing APIs, or aliases to the same views?
- `DecathlonOAuthCallBackView.callback()` (`providers/openid/views.py:229`) decodes JWT access tokens with `verify_signature=False`. While this is expected since the token comes from Decathlon's token endpoint, is there any validation of the token issuer or audience?
- `PaymentManagementClient.save_card()` raises an explicit exception. When was card saving disabled and why?

## Retailer Modules — Dixons, Decathlon, Leroy Merlin, Cisco, Lego, Mango (from T26)

- Why is `fetch_mango_item()` disabled with an early return `{"error": "MP0007"}` at `mango/store_type.py:85`? Has the Mango API integration been replaced by the inventory service or another mechanism?
- Are the SATO ngrok URLs (`https://cs-b11.ngrok.io`) in `cisco/store_type.py` still valid in production, or is the Cisco SATO integration legacy/disabled?
- `LegoStoreType.confirm_order()` at `lego/store_type.py:226` creates `AdyenPay(self.store, 'mango')` instead of `'lego'`. Does AdyenPay's second parameter affect merchant configuration, and are Lego payments using Mango's Adyen merchant settings?
- The Leroy Merlin API key `jOz8NzN0THYj9JAKqibDH0DTcx73moJZ` is hardcoded at `leroy_merlin/store_type.py:50`. Is this key still valid, and should it be moved to environment configuration?
- Dixons stock availability check is commented out in `create_order()` (`dixons/store_type.py:307-308`). Was this intentional or a debugging remnant?
- Four different Redis configurations exist across the 6 modules: `settings.REDIS_HOST`, `gates.mishipay.com`, `localhost`, and `settings.REDIS_HOST`. Are these the same instance in production?
- Decathlon's `activate_epc_for_rfid()` is defined but never called. Is RFID reactivation for returns handled elsewhere, or is this incomplete?
- Cisco's Spreedly payment delivery body uses string substitution with a hardcoded certificate/encryption template (`cisco/store_type.py:201-214`). Is this template maintained separately?

## Inventory Module (from T27)

- Why do `inventory/utils/db.py`, `inventory/nude_foods/db.py`, and `inventory/stellar/db.py` have near-identical `bulk_update_or_create_items()` implementations? Could these be consolidated into a parameterized function with a configurable unique key field?
- `sort_items_for_loves_kiosk()` is documented as a "temporary hack for test env" (`inventory_service_data_preprocessors.py:8`). Is this still in use or should it be removed?
- The Stellar client module (`inventory/stellar/`) has no `tests/` directory and no `__init__.py`, unlike all other client modules. Is this intentional or an oversight?
- `breeze_thru/database_service.py` only supports `bulk_create()`, not update (TODO at line 25). Has this been addressed, or are Breeze Thru imports always run with `--clean`?
- Is there a plan to migrate all Pattern A retailers (monolith DB imports via `inventory/`) to Pattern B (Inventory Service imports via `inventory_scripts`)? Which retailers are planned for migration?
- `StellarItemImport` and `StellarTaxImport` docstrings say "Import Muji" — copy-paste artifacts from the Muji module. Are these known?
- `express_lane_item_catalogue_import()` reads CSV with manual line parsing (`open(filepath, 'r')` with `split(",")`) instead of using pandas or the `PromotionImport` pattern. Is this intentional?
- The proxy `requests.post()` call in `views.py:37` (`post_proxy()`) has no timeout parameter. If the Inventory Service is slow or unresponsive, the request will hang indefinitely. Should a timeout be added?
- All 7 inventory proxy views have no `authentication_classes` or `permission_classes`. Is security handled entirely at the API gateway (Kong) level, or should view-level auth be added?

## Shared Libraries — Lib Module (from T28)

- Is `backend_profiling` ever enabled in development environments? All modules are commented out in the default `BACKEND_PROFILING_MODULES` tuple, and `utils/stack.py` imports Python 2's `SocketServer`, making it incompatible with Python 3.
- Has the vendored `explorer` (v1.1.2) been considered for upgrade to a newer version of `django-sql-explorer`? The vendored copy is over 7 years behind upstream (v4.x+) and contains Python 2 compatibility code.
- What are the actual values of `EXPLORER_CONNECTIONS` and `EXPLORER_DEFAULT_CONNECTION` in production? The app_settings defaults show no connection configured.
- Is the `EXPLORER_TOKEN` setting changed from its default value `"CHANGEME"` in production? If `EXPLORER_TOKEN_AUTH_ENABLED` is `True`, this would allow unauthenticated API access with the default token.
- Is `pyoptile` still in active use? The Optile/Payoneer integration via `PayUnity` in the DOS module appears to be a legacy payment path. Which retailers, if any, still use Optile/PayUnity?
- The `python_bcbp` library has stub methods (`get_first_leg_mandatory_fields`, `get_indent_format`, `extract_legs`). Is the `decode_barcode()` path the only one used in production?
- Explorer's SQL blacklist includes `CREATED`, `UPDATED`, `DELETED` — common column names in MishiPay models. Has this caused issues with legitimate SELECT queries?
- Explorer's `Query.Meta.permissions` defines `can_drive`, `can_vote`, `can_drink` — are these used anywhere, or are they artifacts from the original library's test suite?
- `pyoptile/api_resources/charge.py:113` has a likely copy-paste bug: `preset_id=params['params']['customer_id']` — the keyword argument should be `customer_id`. Has this caused any issues?

## Admin Actions (from T29)

- Are the `TODO: Check permission` comments in `CreateStoreAction` and `CreatePaymentVariantAction` intentional, or should these actions verify specific permissions beyond the `AdminPrivilege.permission` gate?
- The `object_id` auto-increment in `CreateStoreAction` (`actions_definitions.py:93-99`) is not atomic — it sorts all existing `object_id` values in Python and adds 1. Has this caused duplicate `object_id` values under concurrent execution?
- Is the dynamic `__import__()` in `AdminAction.run()` restricted to trusted `admin_action_class` values via database-level controls, or could a malicious admin record execute arbitrary code?
- The `SendEmailAction` logs `"[Admin Action] Running email admin action"` at `logger.error` level (`actions_definitions.py:47`). Is this intentional or a logging level mistake?
- No tests exist for the `admin_actions` module (`tests.py` is an empty stub). Is this intentional given the admin-only use case?
- `PaymentVariantSerializer` has a nested `get_store()` method inside `Meta` class (`serializers.py:68-70`) that always returns `None`. Is this dead code?

## Legacy Analytics & Analytics Library (from T29)

- Are the 6 legacy placeholder methods in `analytics_library/mixpanel.py` (`record_item_delete`, `record_item_delete_view`, `record_checkout`, `record_payment_auth`, `record_auth_change`, `record_past_purchases`, `record_shopping_session`) still called from any client code, or are they dead code? These send empty string values for all params.
- Is Mixpanel still the primary external analytics platform, or has it been fully replaced by the internal `mishipay_retail_analytics` system?
- How are `SalesSummary` and `SaturnSalesSummary` (in `analytics/models.py`) populated? No management command, signal, or task was found in the `analytics` module.
- `EventRecordView` (`analytics/views/events.py`) has no authentication or permission classes. Is it protected at the API gateway level?
- The `StoreNotification` model has a `trigger` field with only one choice (`Item Purchase`). Are additional trigger types planned?
- `EventInterface.event_handler()` dispatches to `EventType` methods via `getattr()` with no whitelist — any method name can be called via the API. Is this a security concern?
- `DummyEventInterface` only has a `track()` method, but `EventInterface.event_handler()` wraps it in `EventType`, whose methods call `self.mixpanel_obj.track()`. When `record_item_delete` (which uses `cus_id` key) is called with a `DummyEventInterface`, do the `EventType` methods still work correctly?

## Clients Module (from T29)

- Is the hardcoded `/test/CA/` path in `AzureStorageClient.upload_blob_file()` (`azure_client.py:27`) correct for production use?
- Are there callers of `PromoServiceV2Helper` in the codebase, or is it a standalone utility? No imports were found in the explored modules.
- `PromoHeaderRow` has 21 instance attributes but `header()` and `to_list()` only include 18 — `description`, `retailer`, `is_active`, and `active_days` are excluded from CSV output. Is this intentional?
- `PromoServiceV2Helper.get_promotions_list_by_retailer()` and `get_promotions_list_by_store()` call `requests.get()` without a `timeout` parameter. If the promotions service is unresponsive, these calls will hang indefinitely.

## Shopping List Module (from T30)

- Is the v1 (legacy monolith item) code path still actively used, or has it been fully superseded by v2 (inventory service)? The deprecated `/api/v1/` mount is still active.
- `transfer_to_basket_inventory_service()` (v2) does not check for existing basket items before adding. Could this create duplicate basket items on repeated transfers?
- `ShoppingListItemV2.item_variant` stores a denormalized JSON snapshot from the inventory service at creation time. Is there a mechanism to refresh stale variant data?
- `transfer_to_basket()` (v1) ignores `ShoppingListItem.quantity` and always sends `item_quantity=1`. Is this intentional?
- `transfer_to_basket_inventory_service()` hardcodes `purchase_type=CLICK_AND_COLLECT` with a TODO to remove. Is this feature flag still active?
- `ShoppingListSerializer.get_items()` has an N+1 query problem — one DB query per shopping list in list views. Is this a performance concern with real user data?
- Shopping list models have no Django admin registration. Is this intentional?

## RFID Module (from T30)

- Why is the `/rfid/keonn/check-epcs/` endpoint completely unauthenticated (`authentication_classes = []`, `permission_classes = []`)? The commented-out imports suggest authentication was intended. Is it protected at the infrastructure level?
- The `check_epcs()` utility queries Redis with both `.lower()` and `.upper()` versions of each EPC key. What causes the casing inconsistency in the upstream data pipeline?
- Is Keonn the only RFID gate hardware vendor, or are other vendors integrated via different endpoints?
- The RFID ecosystem is fragmented across 6+ Django apps with no central abstraction. Is there a plan to consolidate?
- RFID status functions in `rfid_status.py` (Nedap, SATO, Decathlon, Avery Dennison, Retail Reload) have no retry logic for external API calls. How are failed EPC status updates reconciled?
- Are the Nedap bearer tokens and SATO credentials hardcoded in `settings.py:RFID_UPDATE_SETTINGS` overridden in production environments?

## Scheduled Jobs (from T30)

- Are the hardcoded email recipients in `order_crons.py` and `daily_analytics_crons.py` still the correct team members? Should recipient lists be configurable?
- Are these Saturn and Decathlon cron jobs still actively running in production?
- The timezone handling in `daily_analytics_crons.py` uses `datetime.now().replace(tzinfo=pytz.utc)` — this attaches UTC to local server time rather than converting. Has this caused incorrect date boundaries?
- Why does Decathlon count full and partial refunds as "successful" for daily analytics while Saturn does not?
- The `generate_orders_excel.py` utility uses the unmaintained `xlwt` library (legacy `.xls` format). Are there plans to migrate to `openpyxl` (`.xlsx`)?

## Pub/Sub Kafka Messaging (from T30)

- `PostPaymentSuccessConsumer` and `PaymentOnlyClientsAnalyticsConsumer` share `group.id: "PAYMENT_GROUP"` but subscribe to different topics. Has this caused Kafka consumer rebalancing issues?
- `BaseProducerAdapter.produce()` has `timestamp=datetime.utcnow()` as a mutable default argument — evaluated once at import time. Do callers always pass an explicit timestamp, or are messages silently sharing the module import timestamp?
- `CommonConsumer.get_handler()` returns `None` for unknown `consumer_identifier_key` values, causing `AttributeError` downstream. Should there be a default handler or at least a logged warning?
- Are the `mishi.streams.post-transaction` and `flying_tiger.order.post_transaction` topics consumed by external services? No consumers for these topics exist in this codebase.
- Is there a dead-letter queue or persistent failure record for messages that exhaust all retries (currently dropped after Slack alert)?
- Are the two sets of Kafka credentials (main vs payments) for separate Kafka clusters or ACL-scoped credentials on the same cluster?
- ~~How is `AggregatedAnalytics` populated?~~ **Answered**: The `ExternalPaymentsAnalyticsHandler` (consumed by `PaymentOnlyClientsAnalyticsConsumer` from `queuing.external.payments.analytics.json`) creates/updates `AggregatedAnalytics` records via upsert. This resolves the open question from T21.

## Saturn Module (from T31)

- The `items_map` dictionary in `SaturnStoreType` hardcodes 14 product_id → price overrides. What is the purpose? Demo kit prices, promotional prices, or API corrections?
- `SaturnStoreType.get_product_identity()` calls a Google Cloud `alice-expose-api` with a hardcoded API key. Is this endpoint still active?
- The `@asyncio.coroutine` / `yield from` syntax is deprecated since Python 3.8 and removed in Python 3.11. Is production running Python < 3.11?
- Redis is connected to `gates.mishipay.com:6379` in Saturn but `mpay_redis` (via `settings.REDIS_HOST`) in the `rfid` module. Are these the same Redis instance?
- Is the Saturn Austria (Innsbruck) store still active, or has it been fully migrated to `saturn_smart_pay`?
- The SOAP fulfilment in `saturn` hardcodes outlet ID `79` while `saturn_smart_pay` uses settings-driven outlet IDs. Is `79` Saturn Innsbruck–specific?

## Saturn Smart Pay Module (from T31)

- `SaturnSmartPayItemMap` model exists but is not used by `SaturnSmartPayStoreType`. Is it vestigial or used elsewhere?
- `refund_order()` only marks orders as returned without calling any payment processor. Are Saturn Smart Pay refunds processed manually outside MishiPay?
- Age restriction logic uses hardcoded department/group/category ID lists. How are these maintained when Saturn adds new restricted categories?
- The `mail_sales_report` command is duplicated identically in both `saturn` and `saturn_smart_pay` with the same hardcoded store ID. Intentional or accidental duplication?
- The `saturn_order_migrations` and `saturn_saved_card_migrations` commands are one-time migration scripts. Have these been run and can they be removed?
- What is the current production status of XPay as a payment processor?

## Easy Store Creation Module (from T31)

- Is the `settings.EASY_STORE_MS_PAY_VARAINT_SETUP_API` endpoint always available? What happens if it fails during `save_payment_variant()`?
- `EasyStore.save()` is not wrapped in `transaction.atomic()`. Has this caused inconsistent state (store created but payment variant missing)?
- `save_store()` reuses existing stores at matching lat/long coordinates. Could this accidentally overwrite an unrelated store?
- The Cloudinary image upload code is fully commented out in `EasyStore.create_cloudinary_image_url()`. Feature abandoned or planned?
- Easy Store always creates `demo=True` stores. Is there a promotion workflow for moving an Easy Store to production?
- `StoreDataUploadHelper.upload_inventory()` runs both legacy BWG and inventory service imports for stores with mixed platform settings. Could this cause duplicate items?
- Is the `DarAlAmiratStoreType`/`FloristStoreType` barcode-stripping hack (appending `barcode[1:]` as alternate) still needed?

## Fiscalisation Module (from T32)

- Is the Venistar/Retail Pro flow used only for Italian retailers, or has it been extended to other countries?
- How frequently does Retail Pro poll the `GetSaleSIDView` endpoint? Is there rate limiting?
- `SaleSIDDetailsSerializer.get_items()` raises a bare `RuntimeError` on discount mismatch — is this caught upstream or does it produce a 500 error?
- Where is the `PostTransactionSuccessHandler` pub/sub wiring configured — in the `pubsub` module or an external service?
- Are there plans for additional country-specific fiscalisation workflows (e.g., Germany TSE/Fiskaly, Saudi Arabia ZATCA Phase 2)?
- The Funky Buddha endpoint URL is not retailer-scoped unlike Venistar endpoints. Is this intentional?
- `RetailerIDMixin` validates UUID format but doesn't check retailer ownership of the queried store. Is this handled by `FiscalisationPermission` group membership?

## DDF Price Activation Module (from T32)

- Management command line 91: `rpm_file.retailer_store_ids = list(items_for_price_activation)` — likely should be `list(locations_in_file)`. Has this caused incorrect data?
- `get_ddf_user_details()` splits username on `'0'` — fragile for usernames containing `'0'` in the name or number portion.
- Download view line 159: `items_for_download.order_by(...)` result is discarded (QuerySets are immutable). Is the CSV output unordered?
- Activation marks ALL items as activated including duplicates filtered from the API call. Is this intentional?
- Is there retry logic if `InventoryV1Client.update_inventory()` fails?
- How frequently is `ddf_process_rpm_price_activation_file` run — cron, file watcher, or manual?
- `PriceActivationPage` model has `managed=True` in tests but `managed=False` in production. Does this cause test/production schema divergence?
- Are there other DDF-specific modules beyond `ddf_price_activation`?

## Infrastructure — Docker & Service Architecture (from T33)

- Are all env files committed with actual credentials (API keys, database passwords, Stripe tokens, Kafka secrets, Azure Storage connection strings)? Several files in `conf/` contain what appear to be real keys — are these development-only values overridden in production?
- How are production env files managed and deployed? The `conf/` directory contains `*.prod.env` and `*.prd.env` files — are these actual production values or templates?
- The production `docker-compose-prod.yml` only defines 7 services. How are the remaining services (pay, pay-gateway, voucher, inventory, ms-promo, ms-loyalty, databases, Kafka consumers) deployed in production — Kubernetes, VMs, or separate compose files?
- `ms-tax` (gRPC, Go, port 9000) and `ms_tax` (HTTP, FastAPI, port 80) coexist with separate databases. Is the gRPC service fully deprecated, or do specific code paths still use it?
- The Gunicorn `loglevel` is set to `debug` with a TODO comment to change to `info`. Is this the actual production configuration?
- Nginx rate limiting (5 req/s per IP, burst 20) — is this applied in production, or does Cloudflare/Kong handle rate limiting upstream?
- The RabbitMQ credentials (`mpay_broker` / `Mpt43Gwsfdsa`) are hardcoded in `docker-compose.yml`. Are these production values or dev-only?
- ~~What is the `explorer` Django app in INSTALLED_APPS?~~ **Answered in T28**: It is a vendored copy of django-sql-explorer v1.1.2 (in `lib/explorer/`) providing a query IDE and schema browser.

## Infrastructure — CI/CD Pipelines (from T34)

- Where is the monolith Docker image build pipeline? No build/test CI pipeline exists in the MishiPay repository. The `Dockerfile` is present but no pipeline triggers it. Is the build defined in a separate Azure DevOps pipeline configuration or another repository?
- Are the "old" prod pipelines (`azure-pipelines-vmss-prod.yml`, `azure-pipelines-vmss-consumers-prod.yml`) still actively used, or have they been fully superseded by the "new" versions (`*-prod-new.yml`) that use `ARM-AzureSubscription1` and graceful traffic drain?
- What CustomScript extension is installed on the VMSS instances? The `az vmss extension set --force-update` command re-triggers an already-configured extension — what does it do (pull latest Docker image, restart containers)?
- Is the 60-second `sleep` drain period in `azure-pipelines-vmss-prod-new.yml` sufficient for long-running API requests (e.g., payment processing, promotion evaluation with large baskets)?
- The Eroski cron job runs Python directly on the host (`python3 /mpay/com.mishipay.server/scripts/eroski.py`) instead of via `docker exec`. Does the host have a separate Python environment with the required dependencies?
- BWG POSLOG generation runs **every minute** (`* * * * *`). Is this still required, or is it a legacy schedule that could be reduced in frequency?
- Are the production crontab files in `scripts/crontabs/` kept in sync with the actual production servers, or are they historical snapshots?
- Is there a plan to migrate the monolith from Azure VMSS to Kubernetes, aligning with the promotions service's Helm-based deployment?
- ~~How is the Eroski cron job running outside Docker on the host?~~ **Answered in T35**: `eroski_cron.py` runs directly on the host, connects to SFTP via `paramiko`, extracts binary `.DAT` files with `strings`, then invokes `docker exec backend_v2 python3 manage.py eroski_inventory_and_promo_import`.

## Ansible Deployment & Configuration (from T35)

- Is the Ansible/HAProxy deployment (`automation/prod-deployment/`) the **active** production deployment mechanism, or has it been superseded by the Azure Pipelines/VMSS approach? The `hosts.yml` has 4 of 6 backend servers and 2 of 4 load balancers commented out.
- What does the real production `deploy-prod.sh` script do? The repo only contains a mock version (in `molecule/default/files/`). The actual script at `/mpay/com.mishipay.server/automation/deploy-prod.sh` is not committed.
- Are the `conf/*.prod.env` and `conf/*.prd.env` files the actual production configuration, or templates? Some contain what appear to be real Stripe keys, TokenEx credentials, and database passwords.
- The Konga admin UI has hardcoded credentials (`admin`/`adminpassword` in `auth/konga/seed_users.data`). Is this changed in production?
- Kong config (`auth/kong/config.yml`) defines API consumers with key-auth credentials in plain YAML. Are these the actual production API keys?
- How is the `conf/` directory deployed to production? Are env files baked into Docker images, mounted as volumes, or injected via a secrets manager?
- The `automation/ms.loyalty/` directory does not exist. Loyalty service database init lives in `automation/ms.promo/mysqld_init/001.sql`. Is this intentional?
- How often are the PostgreSQL dump files (`automation/postgres_init/mpay.dump`, `test_mpay.dump`) actually updated? The README says quarterly.
- The `nginx.conf` rate limiting (5 req/s per IP) — is this active in production, or does Cloudflare/Kong handle rate limiting upstream?
- Is the `conf/postgres/postgresql.conf` (2GB shared_buffers) actually used in production, or is PostgreSQL configured separately?
- The `eroski_item_recommendations_promotions_upload.py` creates promotions valid until 2051. Is this intentional (permanent promotions) or a placeholder?

## Kafka Consumers & Microservices (from T36)

- Are the two sets of Kafka credentials (`CONF_KAFKA_API_KEY/SECRET` vs `CONF_KAFKA_PAYMENTS_API_KEY/SECRET`) for the same Confluent Cloud cluster with different ACLs, or for entirely separate Kafka clusters?
- Which stores have been migrated from the legacy gRPC tax service (`ms-tax` on port 9000) to the new HTTP tax service (`ms_tax` on port 80)? Is there a timeline for completing the migration?
- What external service(s) consume the `mishi.streams.post-transaction`, `flying_tiger.order.post_transaction`, and `flying_tiger.cash_desk.staff_shifts.end_shift_event` topics? No consumers for these exist in the monolith codebase.
- What service produces to `mishi.streams.post-transaction-success`? The monolith only consumes from it. Is this a fiscal authority callback service or another MishiPay microservice?
- The `StoreLocation` protobuf message is defined in `tax.proto` but not referenced by any RPC method. Is it reserved for future use or dead code?
- Is there a plan to add dead-letter queues for Kafka messages that fail after retry exhaustion? Currently failed messages are dropped after Slack notification to `#alerts_kafka`.
- The `consumers_health_check` management command only tests database connectivity (`SELECT 1`). Should it also verify Kafka broker reachability to prevent containers marked healthy but unable to consume messages?
- `BaseProducerAdapter.produce()` has `timestamp=datetime.utcnow()` as a mutable default argument — evaluated once at import time. Do all callers pass an explicit timestamp, or could messages silently share the module import timestamp?
- Is Dramatiq/RabbitMQ being phased out in favour of Kafka for all async processing, or do they serve distinct use cases (Dramatiq for internal tasks, Kafka for inter-service events)?
- The `pay` service publishes to `queuing.external.payments.analytics.json`. Does it publish to any other Kafka topics, or is this its only event output?
- How are Kafka consumer container replicas managed in production? Is there auto-scaling based on topic lag, or are they fixed at 1 instance per consumer type?
