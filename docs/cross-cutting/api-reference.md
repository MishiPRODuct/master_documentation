# Unified API Reference

> Last updated by: Task T38 — Cross-Cutting Unified API Reference

## Overview

This document consolidates every REST API endpoint across both the MishiPay Platform (Django monolith) and the Promotions Microservice into a single reference. Endpoints are grouped by service domain and listed with their HTTP method, full URL path (relative to service root), authentication requirements, and purpose.

**Platform base URL**: All monolith endpoints are served from the Django application behind Kong API Gateway.
**Promotions Service base URL**: `/api/1.0/promotions/` (internal network only, no auth).

### Authentication Legend

| Code | Meaning |
|------|---------|
| **None** | No authentication required |
| **Token** | DRF `TokenAuthentication` (Authorization: Token header) |
| **Keycloak** | JWT via Kong gateway (`MishiKeycloakAuthentication`) — primary auth for mobile/web |
| **Default** | Uses DRF default auth classes (Token + Keycloak) |
| **Dashboard** | Requires `DashboardPermission` (user in `DashboardUser` group) |
| **Admin** | Requires `IsAdminUser` (user in `AdminUser` group) |
| **ActiveStore** | Requires `ActiveStorePermission` (store must be active) |
| **ActiveShift** | Requires `ActiveStaffShiftPermission` (active kiosk shift) |
| **TransactionToken** | One-time `TransactionTokenAuthentication` via `transaction-token` header |
| **JWT** | `JwtTokenPermission` — JWT-based auth for unauthenticated flows |
| **HMAC** | Webhook HMAC signature verification |
| **AllowAny** | Explicitly set to `AllowAny` (public) |

### Response Format

Most platform endpoints return the standard response envelope:

```json
{
  "data": {},
  "status": 200,
  "message": "",
  "error": ""
}
```

---

## 1. Promotions Microservice

All endpoints are unauthenticated (relies on network-level security via internal Docker network `mpay_net`).

See [Promotions Service API Reference](../promotions-service/api-reference.md) for full request/response schemas.

| Method | Path | View | Description |
|--------|------|------|-------------|
| POST | `/api/1.0/promotions/` | `PromotionCreateAndList` | Create a single promotion |
| POST | `/api/1.0/promotions/bulk_transaction/` | `PromotionBulkTransaction` | Bulk create/update/delete promotions |
| POST | `/api/1.0/promotions/transaction/` | `PromotionBulkTransaction` | **Deprecated** alias for `bulk_transaction` |
| POST | `/api/1.0/promotions/evaluate/` | `PromotionEvaluate` | Evaluate promotions for a basket |
| GET | `/api/1.0/promotions/retailer/<retailer>/` | `PromotionsByRetailer` | List all promotions for a retailer |
| GET | `/api/1.0/promotions/store/<store_id>/` | `PromotionsByStore` | List promotions for a store |
| DELETE | `/api/1.0/promotions/store/<store_id>/` | `PromotionsByStore` | Hard-delete all promotions for a store |
| POST | `/api/1.0/promotions/bulk/<store_id>/` | `PromotionsBulkByStore` | Filter promotions by store + retailer IDs |
| POST | `/api/1.0/promotions/suggestions/` | `PromotionSuggestionListByNodes` | Get promotion suggestions by product nodes |
| POST | `/api/1.0/promotions/item_recommendations/` | `PromoBasedItemRecommendations` | Get item recommendations based on promotions |
| GET | `/api/1.0/promotions/<store_id>/<retailer_id>` | `PromotionDetailByStore` | Get specific promotion by store + retailer ID |
| DELETE | `/api/1.0/promotions/<store_id>/<retailer_id>` | `PromotionDetailByStore` | Soft-delete a specific promotion |
| GET | `/api/1.0/p/<store_id>/<retailer_id>` | `PromotionDetailByStore` | **Deprecated** short alias |
| GET | `/health_check/` | `health_check` | Health check endpoint |

---

## 2. DOS Module (`/d/`)

The DOS (Direct Operating System) module is the central hub for customer-facing retail operations. All routes are prefixed with `/d/`.

See [Retailer Modules](../platform/retailer-modules.md) for full details.

### Customer Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/d/customer/signup` | `sign_up` | None | Email registration |
| POST | `/d/customer/signin` | `sign_in` | None | Email login |
| POST | `/d/customer/signout` | `SignOut` | Token | Delete auth token (logout) |
| POST | `/d/customer/guest-signin` | `sign_in_guest` | None | Guest user creation |
| POST | `/d/customer/update-guest-email` | `UpdateGuestUserInfoAPIView` | Token | Update guest user email/name |
| POST | `/d/customer/autoregister` | `AutoRegister` | Token | Convert guest to registered user post-payment |
| POST | `/d/customer/generate-otp` | `GenerateOTP` | None | Send OTP email (throttled: 10/min) |
| GET | `/d/customer/verify-otp` | `VerifyOTP` | None | Verify OTP and get auth token |
| PUT | `/d/customer/update-tokensdata` | `UserTokensDataUpdate` | Token | Update token metadata (store_id, etc.) |
| POST | `/d/customer/facebook` | `fb_authenticate` | None | Facebook OAuth login/signup |
| POST | `/d/customer/apple` | `apple_authenticate` | None | Apple Sign-In |
| POST | `/d/social/<backend>/` | `exchange_token` | None | Google OAuth via Python Social Auth |
| GET/PUT | `/d/v2/customer/profile/` | `CustomerProfile` | Token | Get/update customer profile |
| PUT | `/d/v2/customer/update-user-meta/` | `UpdateUserMeta` | Token | Update user metadata JSON |
| POST | `/d/v2/customer/recover-password/` | `PasswordRecovery` | None | Send password reset email |
| POST | `/d/v2/customer/reset-password/` | `ResetPassword` | None | Reset password with signed token |
| POST | `/d/v2/customer/verify_mail/` | `CustomerMailVerification` | Token | Send email verification |
| POST | `/d/v2/customer/verify_code/` | `CustomerCodeVerification` | Token | Verify email with code |
| POST | `/d/v2/customer/deactivate_self/` | `DeactivateCustomerBySelf` | Token | Self-deactivation |
| POST | `/d/v2/customer/feedback/` | `CustomerFeedbackView` | Token | Submit order/store feedback |
| POST | `/d/v2/webapp/recover-password/` | `PasswordRecoveryForWebApp` | None | Web app password recovery |
| POST | `/d/v2/webapp/reset-password/` | `ResetPasswordForWebApp` | None | Web app password reset |

### Decathlon-Specific Customer Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/d/v2/decathlon-login-signup/` | `decathlon_login_callback` | None | Decathlon SSO login/signup |
| POST | `/d/v2/update-user-information/` | `UpdateCustomerInfoDecathlon` | None | Update Decathlon user info |
| GET | `/d/v2/decathlon-check-email-exists/` | `CheckEMailExistsDecathlon` | Token | Check email in Decathlon system |
| POST | `/d/v2/decathlon-forgot-password/` | `ForgotPasswordDecathlon` | Token | Decathlon password reset |

### Store Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET/POST | `/d/v2/store/` | `StoreView` | None | Get closest store by GPS + bundle ID; or create store |
| GET | `/d/store/closest` | `get_closest_store` | None | Legacy closest store by GPS |
| GET | `/d/v2/store/mango` | `get_closest_mango_store` | None | Closest Mango-type store |
| GET | `/d/v2/stores/closest-stores/` | `ClosestStoreView` | None | Nearest stores with geofencing, clustering, payments |
| GET | `/d/v3/stores/closest-stores/` | `ClosestStoreViewV3` | None | V3 with MS Pay optimization and theme pre-population |
| GET | `/d/v2/stores/list/` | `StoreList` | Token | List stores by bundle ID with search |
| POST | `/d/v2/store/find-store/` | `ClusterStoreConfirmationView` | Token | Cluster store entry/exit QR validation |
| GET | `/d/v2/stores/get-store-id/` | `AppClipStoreID` | None | Resolve app clip ID to store |
| GET | `/d/v3/store/<uuid>/` | `GetStoreViewV3` | None | Get single store by UUID |
| GET | `/d/v4/store/<uuid>/` | `GetStoreViewV4` | None | V4 with full payment/theme pre-population |
| GET | `/d/get-global-config/<uuid>/` | `GetGlobalConfigView` | Token | Get promotion config for store |

### Item Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/d/v2/item/get/` | `ItemView` | Token | Scan item by barcode, with store type routing |
| POST | `/d/v2/item/status/` | `ItemStatusView` | Token | Get sold status for items |
| GET | `/d/v2/item/get/all/` | `AllItemView` | Token | Get all items for a store |
| GET | `/d/v2/item/promotion/` | `ItemPromotion` | Token | Apply promotions to items |
| GET | `/d/v2/item/shopping-bags/` | `GetShoppingBags` | Token | Get shopping bag items |

### Order Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/d/v2/order/` | `OrderView` | Token | Get order history for customer |
| POST | `/d/v2/order/create/` | `OrderView` | Token | Create order via store type |
| GET/POST | `/d/v2/order/confirm/` | `OrderConfirm` | Token/None | Confirm order (GET = XPay redirect, POST = app) |
| GET | `/d/v2/order/single` | `OrderFetch` | Token | Fetch single order by ID |
| GET | `/d/v2/judo/keys/` | `JudoKey` | Token | Get JudoPay API credentials |
| GET | `/d/v2/spreedly/keys/` | `SpreedlyKey` | Token | Get Spreedly environment key |

### Payment Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/d/v2/payment/update/` | `PaymentUpdateView` | Token | Save card details (JudoPay) |
| POST | `/d/v2/payment/remove_card/` | `PaymentRemoveCardView` | Token | Remove saved card |
| POST | `/d/v2/payment/success/` | `payment_success` | None | JudoPay success redirect |
| POST | `/d/v2/payment/failure/` | `payment_failure` | None | JudoPay failure redirect |
| POST | `/d/v2/payment/initialize/` | `PaymentInitialize` | Token | Initialize JudoPay web payment |
| POST | `/d/stripe/pay` | `StripePayment` | None | Process Stripe payment |
| POST | `/d/stripe/remove_card` | `remove_stripe_card` | None | Remove Stripe saved card |
| POST | `/d/v2/payunity/checkout` | `PayunityCheckout` | None | Create PayUnity checkout session |
| GET | `/d/v2/payunity/result` | `PayunityResult` | None | PayUnity result callback |
| POST | `/d/v2/payunity/remove_card` | `PayunityRemoveCard` | Token | Remove PayUnity card |
| POST | `/d/v2/payunity/add_card` | `PayunityAddCard` | Token | Register PayUnity card |
| GET | `/d/v2/paypal/client` | `PaypalClient` | Token | Get PayPal/Braintree client token |
| GET | `/d/v2/verifone/session` | `VerifoneSession` | Token | Create Verifone session |
| POST | `/d/v2/verifone/save-card` | `VerifoneRegisterCard` | Token | Save card via Verifone |
| POST | `/d/v2/adyen/session` | `AdyenSession` | Token | Create Adyen session |
| POST | `/d/v2/adyen/save-card` | `AdyenRegisterCard` | Token | Save card via Adyen |
| POST | `/d/v2/adyen/authorize` | `AdyenAuthorizeOrder` | Token | Authorize and capture Adyen payment |
| POST | `/d/v2/xpay/notify/` | `xpay_notify_save_card` | None | XPay card-save webhook |
| POST | `/d/v2/xpay/save-card/` | `XPaySaveCardAPI` | Token | Initiate XPay card save |
| POST | `/d/v2/apple-pay/session/` | `ApplePaySession` | Token | Create Apple Pay session |
| POST | `/d/v2/google-pay/<action>/` | `GooglePayAPIView` | Token | Google Pay actions |
| POST | `/d/v2/onepay/save-card/<action>/` | `OnePaySaveCardAPIView` | Token | OnePay card operations |

### Other DOS Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/d/v2/gate/info/` | `GateInfo` | Token | Get gate hardware info by serial number |
| GET | `/d/v2/gate/items/` | `GateItems` | Token | Get items for gate's store |
| GET | `/d/v2/coupon/add` | `get_coupon_details` | None | Look up coupon by code + store |
| GET | `/d/v2/socket/status/` | `SocketStatus` | Token | Check socket server status |
| GET | `/d/v2/app/open/` | `AppOpenView` | None | Log app open event |
| POST | `/d/v2/app/register/` | `AppRegister` | Token | Register device for push notifications |
| POST | `/d/v2/app/event/` | `AppEvent` | None | Log generic app event |
| POST | `/d/v2/device/register` | `RegisterDevice` | None | Register device UUID + FCM token |
| GET/POST | `/d/v2/app-force-update-status/` | `AppForceUpdateStatus` | None | Check/list force-update versions |

---

## 3. Core Services (`/core-services/`)

See [Core Module](../platform/core.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/core-services/v1/theme/` | `ThemeAPIView` | None | Retrieve theme properties for a store/platform |
| GET | `/core-services/v1/get-additional-store-details/` | `AdditionalStoreDetails` | Default | Get mixpanel token, payment methods, 3DS for a store |
| GET | `/core-services/v1/all-store-details/` | `AllStoreDetails` | None | Get all store names, addresses, and themes |
| GET | `/core-services/v1/admin/all-store-details/` | `AdminStoreDetails` | Admin | Get stores matching a bundle ID |
| POST | `/core-services/v1/customer/create-support-ticket/` | `SupportTicketView` | Default | Create a support ticket via email |
| GET | `/core-services/v1/sdk-verification/` | `SDKLisenceVerificationAPI` | None | Verify SDK license key, return Scandit key |
| GET | `/core-services/v1/stores-by-country/<country>/` | `StoresByCountry` | None | List stores by country (V1) |
| GET | `/core-services/v2/stores-by-country/<country>/` | `StoresByCountryV2` | None | List stores by country (V2 with themes, entity properties) |
| POST | `/core-services/v1/send-slack-msg/` | `SlackClient` | **None** | Send a Slack message (**no auth -- security concern**) |
| GET/POST | `/core-services/keycloak-migrate/` | `KeycloakMigrationApiView` | **None** | Keycloak migration: return/validate user data |
| GET | `/core-services/v1/store-countries/` | `StoreCountries` | None | List countries with active stores |
| GET | `/core-services/v1/store-denominations/` | `GetStoreDenominations` | None | Get cash denominations for request's store |

---

## 4. Authentication (`/auth-management/`)

See [Authentication](../platform/authentication.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/auth-management/v1/guest-checkout/` | `GuestCheckout` | None | Register or login a guest user |

---

## 5. Item Management (`/item-management/`)

47 URL patterns across three tiers. See [Items](../platform/items.md) for full details.

### V1 Public Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/item-management/v1/item-scan/` | `ItemScan` | Default | Scan item by barcode |
| GET | `/item-management/v1/item-recommendations/` | `ItemRecommendations` | Default | Get item recommendations |
| POST | `/item-management/v1/add-item-to-basket/` | `AddItemToBasket` | Default, ActiveStore | Add item to basket |
| POST | `/item-management/v1/bulk-add-item-to-basket-old/` | `BulkAddItemToBasket` | Default, ActiveStore | Bulk add items to basket |
| POST | `/item-management/v1/bulk-update-hardcoded-basket-items/` | `BulkUpdateHardcodedBasketItems` | Default, ActiveStore | Bulk update hardcoded basket items |
| GET | `/item-management/v1/get-basket-details/` | `GetBasketDetails` | Default | Get basket details |
| POST | `/item-management/v1/add-item-to-wishlist/` | `AddItemToWishlist` | Default | Add item to wishlist |
| GET | `/item-management/v1/get-wishlist-details/` | `GetWishlistDetails` | Default | Get wishlist details |
| POST | `/item-management/v1/remove-item-from-basket/` | `RemoveItemFromBasket` | Default | Remove item from basket |
| POST | `/item-management/v1/remove-item-from-wishlist/` | `RemoveItemFromWishlist` | Default | Remove item from wishlist |
| POST | `/item-management/v1/send-email-for-wishlist/` | `SendEmailForWishlist` | Default | Send wishlist email |
| POST | `/item-management/v1/rate-item/` | `RateItem` | Default | Rate an item |
| GET | `/item-management/v1/get-hard-coded-items/` | `GetHardcodedItems` | None | Get hardcoded items for a store |
| GET | `/item-management/v1/promotions/` | `PromotionList` | Default | List promotions for a store |
| GET | `/item-management/v1/apply-picwic-promotions/` | `ApplyPicwicPromotion` | Default | Apply Picwic promotions |
| GET/POST | `/item-management/v1/manual-check/<basket_id>/` | `ManualVerification` | **None** | Manual basket verification |
| GET | `/item-management/v1/search/` | `ItemSearch` | Default | Search items (NudeFood only) |
| POST | `/item-management/v1/create-item-for-basket/` | `CreateItemForBasket` | Default | Create item for basket |
| POST | `/item-management/v1/update-item-from-basket/` | `UpdateItemFromBasket` | Default, ActiveStore, PriceOverride | Update item in basket (price override) |
| POST | `/item-management/v1/return-item-via-basket/` | `ReturnItemViaBasket` | Default, ActiveStore, ActiveShift | Return item via basket |
| POST | `/item-management/v1/modify-item-notes/` | `ModifyItemNotes` | Default, ActiveStore, ActiveShift | Modify item notes |
| POST | `/item-management/v1/basket_redeem_points` | `BasketRedeemPoints` | Default | Redeem loyalty points on basket |
| POST | `/item-management/v1/basket_cancel_redeemed_points` | `BasketCancelRedeemedPoints` | Default | Cancel redeemed points |
| PATCH | `/item-management/v1/baskets/<basket_id>/` | `UpdateBasketProperties` | Default, ActiveStore, Dashboard | Update basket properties |
| POST | `/item-management/v1/basket_save_rewards` | `BasketSaveRewards` | Default | Save rewards on basket |
| POST | `/item-management/v1/list-addon-items/` | `AddOnItems` | None | List add-on items |
| POST | `/item-management/v1/list-loyalty-coupons` | `ListLoyaltyCoupons` | Default, ActiveShift | List loyalty coupons |
| POST | `/item-management/v1/redeem-loyalty` | `RedeemLoyalty` | Default, ActiveShift | Redeem loyalty |
| POST | `/item-management/v1/unredeem-loyalty` | `UnRedeemLoyalty` | Default, ActiveShift | Unredeem loyalty |
| POST | `/item-management/v1/basket-redeem-unredeem-loyalty` | `BasketRedeemUnredeemLoyalty` | Default, ActiveShift | Basket redeem/unredeem loyalty |
| POST | `/item-management/v1/redeem-points` | `RedeemLoyaltyPointsView` | Default, ActiveShift | Redeem loyalty points |
| GET | `/item-management/v1/carters-image-proxy/` | `image_proxy` | None | Carter's image proxy |
| PATCH | `/item-management/v1/inventory/update/` | `InventoryUpdate` | ActiveStore, InventoryUpdate, UserStore | Update inventory |

### V2 Enhanced Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/item-management/v2/item-scan/` | `ItemScanV2` | Default, ActiveStore, ActiveShift | Item scan (V2, kiosk) |
| POST | `/item-management/v2/add-item-to-basket/` | `AddItemToBasketV2` | Default, ActiveStore, ActiveShift | Add to basket (V2) |
| POST | `/item-management/v2/bulk-add-item-to-basket/` | `BulkAddItemToBasketV2` | Default, ActiveStore, ActiveShift | Bulk add to basket (V2) |
| GET | `/item-management/v2/get-basket-details/` | `GetBasketDetailsV2` | Default, ActiveStore, ActiveShift | Basket details (V2) |
| POST | `/item-management/v2/remove-item-from-basket/` | `RemoveItemFromBasketV2` | Default, ActiveStore, ActiveShift | Remove from basket (V2) |
| POST | `/item-management/v2/create-item-for-basket/` | `CreateItemForBasketV2` | Default, ActiveStore, ActiveShift | Create item for basket (V2) |
| GET | `/item-management/v2/add-on-items/` | `GetAddOnItems` | None | Get add-on items (V2) |
| GET/POST | `/item-management/v2/promotions/` | `PromotionListV2` | Dashboard | List/download promotions (V2) |
| POST | `/item-management/v2/staff/manual-verify-kiosk/` | `ManualVerificationKiosk` | Dashboard | Manual kiosk verification |
| POST | `/item-management/v1/user/manual-verify/` | `UserManualVerification` | Default | User-initiated manual verification |

### Internal Service APIs (no auth -- service-to-service)

| Method | Path | View | Description |
|--------|------|------|-------------|
| POST | `/item-management/api/v1/item-scan/` | `ItemScanService` | Internal item scan |
| POST | `/item-management/api/v1/add-item-to-basket/` | `AddItemToBasketService` | Internal add to basket |
| POST | `/item-management/api/v1/bulk-update-hardcoded-basket-items/` | `BulkUpdateHardcodedBasketItemsService` | Internal bulk update |
| POST | `/item-management/api/v1/add-item-to-wishlist/` | `AddItemToWishlistService` | Internal add to wishlist |
| POST | `/item-management/api/v1/get-basket-details/` | `GetBasketDetailsService` | Internal basket details |
| POST | `/item-management/api/v1/get-wishlist-details/` | `GetWishlistDetailsService` | Internal wishlist details |
| POST | `/item-management/api/v1/remove-item-from-basket/` | `RemoveItemFromBasketService` | Internal remove from basket |
| POST | `/item-management/api/v1/remove-item-from-wishlist/` | `RemoveItemFromWishlistService` | Internal remove from wishlist |
| POST | `/item-management/api/v1/rate-item/` | `RateItemService` | Internal rate item |
| POST | `/item-management/api/v1/apply-picwic-promotions/` | `ApplyPicwicPromotionService` | Internal Picwic promotions |

---

## 6. Order Management (`/order-management/`)

See [Orders](../platform/orders.md) for full details.

### Public Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/order-management/v1/create-order/` | `CreateOrder` | Default | Create order (V1) |
| POST | `/order-management/v2/create-order/` | `CreateOrderV2` | Default | Create order (V2) |
| POST | `/order-management/v1/get-order-details/` | `GetOrderDetails` | Default | Get single order details |
| POST | `/order-management/v1/get-all-order-details/` | `GetAllOrderDetails` | Default | Get all customer orders |
| POST | `/order-management/v2/get-order-details/` | `GetOrderDetailsV2` | **None** | Order details V2 (no auth) |
| POST | `/order-management/v3/get-order-details/` | `GetOrderDetailsV3` | Default | Order details V3 |
| POST | `/order-management/v1/return-order-info/` | `ReturnOrderInfo` | Default | Return/refund order info |
| POST | `/order-management/v1/order-receipt/` | `OrderReceipt` | Default | Generate receipt |
| POST | `/order-management/v2/order-receipt/` | `OrderReceiptV2` | Default | Receipt V2 |
| GET | `/order-management/v1/get-receipt-banners/` | `GetReceiptMessage` | Default | Receipt promotional banners |
| POST | `/order-management/v1/order/lottery-code` | `OrderLotteryCodeView` | Default | Submit Italian lottery code |
| POST | `/order-management/v1/order/order-transfer` | `OrderTransfer` | Default | Transfer order to another store |
| POST | `/order-management/v1/order/verify-for-exit-gate/` | `OrderVerifyForExitGate` | Default | Exit gate verification V1 |
| POST | `/order-management/v2/order/verify-for-exit-gate/` | `OrderVerifyForExitGateV2` | Default | Exit gate verification V2 |
| POST | `/order-management/v1/cancel-sale/` | `CancelSaleSessionView` | Default | Cancel sale session |
| POST | `/order-management/v1/failed-orders/` | `GetFailedOrders` | Default | List failed orders |

### Guest Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/order-management/v1/guest/get-receipt-banners/` | `GetReceiptMessageGuestView` | None | Receipt banners (guest) |
| POST | `/order-management/v2/guest/get-order-details/` | `GetOrderDetailsGuestView` | None | Order details (guest) |
| POST | `/order-management/v2/guest/update_customer_email_and_send_order_receipt` | `SendEmailReceiptGuestView` | None | Update email & send receipt (guest) |

### Staff Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/order-management/v2/staff/order_receipt` | `StaffOrderReceipt` | Default | Staff receipt generation |
| POST | `/order-management/v3/staff/order_receipt` | `StaffOrderReceiptV2` | Default | Staff receipt V2 |
| POST | `/order-management/v2/staff/update_customer_email_and_send_order_receipt` | `UpdateCustomerEmailAndSendEmailReceipt` | Default | Update customer email + resend |
| POST | `/order-management/v2/order_receipt_excluding_refunds` | `OrderReceiptExcludingRefunds` | Default | Receipt without refund items |
| POST | `/order-management/v2/ddf/staff/failed_order_receipt` | `DDFStaffOrderReceipt` | Default | DDF-specific failed order receipt |

### Special Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/order-management/v1/post-transaction/` | `OrderPostTransactionView` | Default | Hudson post-transaction number |
| POST | `/order-management/v1/lookup-transaction/` | `HudsonTransactionLookupView` | **None** | Hudson transaction lookup |
| POST | `/order-management/v1/whatsapp/order-receipt` | `SendWhatsAppMessageView` | Default | WhatsApp receipt |
| POST | `/order-management/v1/whatsapp/webhook` | `WhatsAppMessagesWebhookView` | None | WhatsApp webhook |
| POST | `/order-management/v1/offline/full_data_sync/` | `OfflineSyncMonolithData` | Default | Offline-to-monolith data sync |
| POST | `/order-management/v1/order/verify-for-exit-gate-vengo/` | `OrderVerifyForExitGateVengoV1` | Default | Vengo exit gate verification |
| GET | `/order-management/v1/vengo-entry-gate/get-qr-data/` | `GetEntryQRData` | Default | Vengo entry QR data |
| POST | `/order-management/v1/vengo-entry-gate/validate-qr/` | `ValidateEntryQRCode` | Default | Vengo QR validation |

### Internal Service APIs (no auth)

| Method | Path | View | Description |
|--------|------|------|-------------|
| POST | `/order-management/api/v1/create-order/` | `CreateOrderService` | Internal order creation |
| POST | `/order-management/api/v1/get-order-details/` | `GetOrderDetailsService` | Internal order details |
| POST | `/order-management/api/v1/get-all-order-details/` | `GetAllOrderDetailsService` | Internal all orders |

---

## 7. Payment Management (`/payment-management/`)

59 routes across three tiers. See [Payments](../platform/payments.md) for full details.

### Legacy Payment APIs

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/payment-management/v1/get-payment-methods/` | `GetPaymentMethods` | Default | Get payment methods for store |
| POST | `/payment-management/v1/adyen-payment/` | `AdyenPayment` | Default | Process Adyen payment |
| GET | `/payment-management/v1/get-origin-keys/` | `GetOriginKeys` | Default | Get Adyen origin keys |
| POST | `/payment-management/v1/adyen/generate-hpp-form-data/` | `AdyenHPPGenerateFormData` | Default | Generate Adyen HPP form |
| POST | `/payment-management/v1/adyen/notify/` | `AdyenNotifyUrl` | None | Adyen notification webhook |
| POST | `/payment-management/v1/adyen/confirm-api/` | `AdyenConfirm` | Default | Confirm Adyen payment |
| POST | `/payment-management/v1/adyen/webhook` | `AdyenWebhookView` | None | Adyen webhook |
| GET | `/payment-management/v1/adyen/web-redirect` | `AdyenWebRedirect` | None | Adyen web redirect |
| POST | `/payment-management/v1/create-payment-session/` | `CreatePaymentSession` | Default | Create payment session |
| POST | `/payment-management/v1/create-psp-session/` | `CreatePaymentSessionWithoutOrder` | Default | Create PSP session without order |
| POST | `/payment-management/v1/verify-payment/` | `VerifyPayment` | Default | Verify payment |
| POST | `/payment-management/v1/confirm-payment/` | `ConfirmPayment` | Default | Confirm payment |
| POST | `/payment-management/v1/custom-action/` | `PaymentCustomAction` | Default | Payment custom action |
| GET/POST | `/payment-management/v1/get-or-save-card/` | `GetOrSaveCard` | Default | Get or save payment card |
| POST | `/payment-management/v1/delete-saved-card/` | `DeleteSavedCard` | Default | Delete saved card |
| POST | `/payment-management/v1/delete-token-from-psp/` | `DeleteTokenFromPSP` | Default | Delete token from PSP |
| POST | `/payment-management/v1/notify-payment/` | `XpayNotify` | None | XPay notification |
| POST | `/payment-management/v1/optile-notify/` | `OptileNotify` | None | Optile notification |
| POST | `/payment-management/v1/sog/notify/` | `SogEcommerceNotify` | None | SOG ecommerce notification |
| POST | `/payment-management/v1/mercanet/notify/` | `MercanetNotify` | None | Mercanet notification |
| GET | `/payment-management/v1/optile-summary/` | `OptileSummary` | Default | Optile payment summary |
| POST | `/payment-management/v1/payment-wallet-details/` | `PaymentWalletDetails` | Default | Payment wallet details |
| GET | `/payment-management/v1/quickpay/web-redirect` | `QuickPayWebRedirect` | None | QuickPay web redirect |

### MS Pay APIs (24 endpoints)

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/payment-management/v1/ms-pay/payment-methods/` | `MsPayPaymentMethods` | Default | Get allowed payment methods |
| POST | `/payment-management/v1/ms-pay/payment-session/` | `MsPayCreatePaymentSession` | Default | Create payment session |
| POST | `/payment-management/v1/ms-pay/payment-session/check-status/` | `MsPayPaymentSessionCheckStatus` | Default | Check session status |
| POST | `/payment-management/v1/ms-pay/payment-session/{id}/add-payment/` | `MsPayAddPayment` | Default | Add payment to session |
| GET | `/payment-management/v1/ms-pay/payment-session/{id}/payment-details/` | `MsPayPaymentDetails` | Default | Get payment details |
| POST | `/payment-management/v1/ms-pay/payment-session/{id}/abort-payment/` | `MsPayAbortPayment` | Default | Abort payment |
| POST | `/payment-management/v1/ms-pay/setup-session/` | `MsPayCreateSetupSession` | Default | Create card tokenization session |
| POST | `/payment-management/v1/ms-pay/setup-session/{id}/add-card/` | `MsPayAddCardToSetup` | Default | Add card to setup |
| GET | `/payment-management/v1/ms-pay/setup-session/{id}/setup-details/` | `MsPaySetupDetails` | Default | Get setup details |
| GET | `/payment-management/v1/ms-pay/saved-cards/` | `MsPaySavedCards` | Default | List saved cards |
| POST | `/payment-management/v1/ms-pay/webhook/` | `MsPayWebhook` | None | MS Pay webhook |
| POST | `/payment-management/v1/ms-pay/validate-client/` | `MsPayValidateClient` | Default | Validate client |
| POST | `/payment-management/v1/ms-pay/fetch-payment/` | `MsPayFetchPayment` | Default | Fetch payment by IDs |
| POST | `/payment-management/v1/ms-voucher/validate-and-apply/` | `MsVoucherValidateApply` | Default | Voucher validation/application |
| POST | `/payment-management/v1/ms-pay/kiosk/stripe/terminal/` | `MsPayKioskStripeToken` | Default | Kiosk Stripe terminal token |
| GET | `/payment-management/v1/ms-pay/refund-methods/` | `MsPayRefundMethods` | Default | Get refund methods |
| POST | `/payment-management/v1/ms-pay/return-payment-session/` | `MsPayCreateReturnPaymentSession` | Default | Create refund session |
| GET | `/payment-management/v1/ms-pay/return-payment-session/{id}/return-details/` | `MsPayReturnDetails` | Default | Get refund details |
| POST | `/payment-management/v1/ms-pay/return-payment-session/{id}/abort-return/` | `MsPayAbortReturn` | Default | Abort refund |
| GET | `/payment-management/v1/ms-pay/viva-wallet-terminal/device-status/` | `MsPayVivaWalletTerminalDeviceStatus` | Default | Terminal status |
| POST | `/payment-management/v1/ms-pay/void-payment/` | `MsPayVoidPayment` | Default | Void payment |
| POST | `/payment-management/v1/ms-pay/edfapay/payment-session/` | `MsPayEdfaPayCreatePaymentSessionView` | Default | EdfaPay session |
| GET | `/payment-management/v1/ms-pay/edfapay/payment-session/{id}/payment-details/` | `MsPayEdfaPayPaymentDetailsView` | Default | EdfaPay details |
| POST | `/payment-management/v1/ms-pay/walmartpay/session/initiate/` | `MsPayInitiateWalmartPaySession` | Default | Walmart Pay session |
| GET | `/payment-management/v1/ms-pay/gift-card-balance/` | `MsPayGiftCardBalance` | Default | Gift card balance check |

### Internal Service APIs (no auth)

| Method | Path | View | Description |
|--------|------|------|-------------|
| POST | `/payment-management/api/v1/get-payment-methods/` | `GetPaymentMethodsService` | Internal: fetch methods |
| POST | `/payment-management/api/v1/adyen-payment/` | `AdyenPaymentService` | Internal: Adyen payment |
| POST | `/payment-management/api/v1/adyen/generate-hpp-form-data/` | `AdyenHPPGenerateFormDataService` | Internal: HPP form |
| POST | `/payment-management/api/v1/payment-session/` | `CreatePaymentSessionService` | Internal: create session |
| POST | `/payment-management/api/v1/api-psp-session/` | `CreatePaymentSessionWithoutOrderService` | Internal: PSP session |
| POST | `/payment-management/api/v1/api-confirm-payment/` | `ConfirmPaymentService` | Internal: confirm |
| POST | `/payment-management/api/v1/by-pass/` | `ByPassPaymentService` | Internal: bypass payment |
| POST | `/payment-management/api/v1/api-verify-payment/` | `VerifyPaymentService` | Internal: verify |
| POST | `/payment-management/api/v1/payment_method_summary/` | `PaymentMethodSummary` | Internal: summary |

---

## 8. Customer Management (`/customer-management/`)

See [Customers](../platform/customers.md) for full details.

### Public Endpoints

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/customer-management/v1/add-or-update-customer-address/` | `AddOrUpdateCustomerAddress` | Default | Add or update addresses |
| PUT | `/customer-management/v1/update-customer-address/` | `UpdateCustomerAddress` | Default | Update existing addresses |
| GET | `/customer-management/v1/get-additional-customer-details/` | `GetAdditionalCustomerDetails` | Default | Full customer context (baskets, addresses, profile, payment, privacy) |
| POST | `/customer-management/v1/add-customer-feedback/` | `AddCustomerFeedback` | Default or JWT | Submit feedback/rating |
| GET/POST | `/customer-management/v1/privacy-policy/` | `PrivacyPolicy` | Default | Get/update privacy policy consent |
| POST | `/customer-management/v1/update-notification-permission/` | `UpdateNotificationPermission` | Default | Toggle push notifications |
| POST/PUT/DELETE | `/customer-management/v1/retailer-customer-profile/` | `UpdateRetailerSpecificCustomerProfile` | Default, ActiveShift | CRUD retailer profiles (loyalty, tax, boarding pass) |
| POST | `/customer-management/v1/loyalty-otp/` | `OTPValidationView` | Default | OTP validation for loyalty |
| GET/POST | `/customer-management/v1/marketing-consent/` | `MarketingConsentView` | Default or JWT | Get/update marketing consent |
| GET | `/customer-management/v1/nationalities/` | `NationalitiesView` | Default | List available nationalities |
| GET/POST | `/customer-management/v1/guest/marketing-consent/` | `MarketingConsentGuestView` | **None** | Guest marketing consent |

### BuyerAddress REST API (V2)

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/customer-management/api/v1/address/` | `BuyerAddressView` | Default | Create new address |
| PUT | `/customer-management/api/v1/address/<uuid>/` | `BuyerAddressView` | Default | Update address |
| DELETE | `/customer-management/api/v1/address/<uuid>/` | `BuyerAddressView` | Default | Delete address |
| GET | `/customer-management/api/v1/address-list/` | `BuyerAddressListView` | Default | List all customer addresses |

### Internal Service APIs (no auth)

| Method | Path | View | Description |
|--------|------|------|-------------|
| POST | `/customer-management/api/v1/add-or-update-customer-address/` | `AddOrUpdateCustomerAddressService` | Internal address add/update |
| POST | `/customer-management/api/v1/update-customer-address/` | `UpdateCustomerAddressService` | Internal address update |
| POST | `/customer-management/api/v1/add-customer-feedback/` | `AddCustomerFeedbackService` | Internal feedback |

---

## 9. Dashboard (`/dashboard/`)

See [Dashboard](../platform/dashboard.md) for full details.

### Authentication

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/dashboard/v2/user/login` | `LoginView` | None | Username/password login |
| POST | `/dashboard/v2/user/worker-login` | `PasscodeLoginView` | None | Staff ID + PIN login |
| POST | `/dashboard/v2/user/sign-up` | `SignUpView` | None | Registration with invite token |
| POST | `/dashboard/v2/user/recover-password` | `PasswordRecoveryView` | None | Password recovery email |
| POST | `/dashboard/v2/user/reset-password` | `PasswordResetView` | None | Password reset |
| POST | `/dashboard/v2/user/reset-passcode` | `PasscodeResetView` | None | PIN reset |
| POST | `/dashboard/v2/staff/login_with_qr` | `StaffLoginQRView` | None | QR code kiosk login |
| POST | `/dashboard/v2/staff/login` | `StaffLoginView` | None | Regular staff login |

### Order Operations

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/dashboard/v2/order/list/all` | `AllOrderListView` | Dashboard | Orders across all stores |
| GET/POST | `/dashboard/v2/order/get` | `OrderViewSet` | Dashboard | Retrieve specific order |
| GET | `/dashboard/v2/order/list` | `OrderListViewSet` | Dashboard | Paginated order list |
| GET | `/dashboard/v2/order/scan` | `OrderView` | Dashboard | Lookup by barcode/order_id |
| POST | `/dashboard/v2/order/refund` | `RefundOrder` | Dashboard | Process refund (legacy) |
| POST | `/dashboard/v2/order/verify` | `VerifyOrder` | Dashboard | Verify order correctness |
| POST | `/dashboard/v2/order/update-all-payment-statuses` | `UpdateAllOrderPaymentStatuses` | Dashboard | Sync payment status |
| POST | `/dashboard/v2/order/register-order` | `RegisterExternalOrder` | Dashboard | Register external order ID |
| POST | `/dashboard/v2/order/authorize` | `AuthorizeOrder` | Dashboard | Authorize order (Mango) |
| POST | `/dashboard/v2/order/cancel` | `CancelOrder` | Dashboard | Cancel order (Mango) |

### Item Operations

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/dashboard/v2/item/all` | `ItemView` | Dashboard | List items (paginated) |
| GET | `/dashboard/v2/item/category/` | `ItemCategoryView` | Dashboard | List categories |
| POST | `/dashboard/v2/item/update` | `ItemUpdateView` | Dashboard | Update item |
| POST | `/dashboard/v2/item/add` | `ItemAddView` | Dashboard | Add item |
| POST | `/dashboard/v2/item/delete` | `ItemDeleteView` | Dashboard | Delete item |
| POST | `/dashboard/v2/item/upload` | `ItemCsvUploadView` | Dashboard | Bulk CSV import |
| POST | `/dashboard/v2/item/epc/upload` | `SaturnUploadEpcList` | Dashboard | Upload RFID EPC list (Saturn) |
| POST | `/dashboard/v2/item/fetch` | `ItemFetchView` | Dashboard | Fetch single item |

### Bulk User Management

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET/POST | `/dashboard/v2/bulk-user-create` | `BulkDashboardUserCreateViewV2` | Dashboard | Bulk user creation/listing |
| GET | `/dashboard/v2/bulk-user-create-template` | `BulkDashboardUserCreateTemplateViewV2` | Dashboard | CSV template download |
| GET/POST | `/dashboard/bulk-user-create` | `BulkDashboardUserCreateView` | Dashboard | Legacy bulk user creation |
| GET | `/dashboard/bulk-user-create-template/` | `BulkDashboardUserCreateTemplateView` | Dashboard | Legacy CSV template |

### Reports

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/dashboard/v2/report/sales_summary` | `SalesReportSummary` | Dashboard | Sales analytics report |

---

## 10. Dashboard Management (`/dashboard-management/`)

See [Dashboard](../platform/dashboard.md) for full details.

### Order Management

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/dashboard-management/v1/order/scan/` | `GetOrderView` | Default | Order lookup by ID/barcode |
| GET | `/dashboard-management/v1/dashboard-order/details/` | `GetDashboardOrderDetails` | Default | Full order details (tracks access count) |
| GET | `/dashboard-management/v1/order/list/` | `GetStoreOrdersView` | Dashboard, UserStore | Paginated order listing (v1) |
| GET | `/dashboard-management/v2/order/list/` | `GetStoreOrdersViewV3` | Dashboard, UserStore | Order listing with filters (v2) |
| GET | `/dashboard-management/v3/order/list/` | `GetStoreOrdersViewV4` | Dashboard, UserStore | Enhanced filtering (v3) |
| GET | `/dashboard-management/v4/order/list/` | `GetStoreOrdersViewV4` | Dashboard, UserStore | Current version (v4) |
| GET | `/dashboard-management/v1/order/list/all-stores/` | `GetAllStoreOrdersView` | Dashboard | Orders across all stores |
| POST | `/dashboard-management/v1/refund-order/` | `RefundOrder` | Dashboard | Legacy refund |
| POST | `/dashboard-management/v1/refund-order-new/` | `RefundOrderNewView` | Dashboard | New refund flow |
| GET | `/dashboard-management/v1/refund-order-new/status` | `GetRefundOrderNewStatus` | Dashboard | Refund status |
| POST | `/dashboard-management/v1/change-order-status/` | `ChangeCCOrderStatus` | Dashboard | Click & Collect status transitions |
| POST | `/dashboard-management/v1/update-order-status/` | `OrderVerificationDashboard` | Dashboard | Manual order verification |
| GET | `/dashboard-management/v1/manual-check/{id}/` | `ManualVerificationDashboard` | Dashboard | Manual verification details |
| GET | `/dashboard-management/v1/basket-manual-check/` | `DashboardVerificationAllSessions` | Dashboard | List pending verifications |

### Item & Inventory Management

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/dashboard-management/v1/item-scan/` | `GetItemScanView` | Dashboard | Item lookup |
| GET | `/dashboard-management/v1/item-inventory/list/` | `GetStoreItemInventoryView` | Dashboard, UserStore | Store inventory listing |
| POST | `/dashboard-management/v1/add-item-inventory/` | `AddSingleItemInventory` | Dashboard | Add inventory item |
| POST | `/dashboard-management/v1/update-item-inventory/` | `UpdateItemInventory` | Dashboard | Update inventory item |
| POST | `/dashboard-management/v1/delete-item-inventory/` | `DeleteInventoryItems` | Dashboard | Soft delete inventory |
| POST | `/dashboard-management/v1/item-bulk-upload/` | `ItemsBulkUpload` | Dashboard | Bulk CSV import |

### Analytics & Reports

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/dashboard-management/v1/dashboard-analytics/` | `DashboardAnalytics` | Analytics perm | Analytics query |
| GET | `/dashboard-management/v1/settlement_summary_all_regions/` | `PaymentSettlementSummaryAllRegions` | Dashboard | Settlement summary |
| GET | `/dashboard-management/v1/settlement_summary_all_regions_old/` | `PaymentSettlementSummaryAllRegionsOld` | Dashboard | Legacy settlement |
| GET | `/dashboard-management/v1/settlement_poslog_recon_report_by_region/` | `PaymentSettlementPosLogReconByRegion` | Dashboard | POS log reconciliation |
| GET | `/dashboard-management/v1/settlement_recon_payout_report_by_region_and_psp/` | `PaymentSettlementReconPayoutByRegionAndPSP` | Dashboard | Payout reconciliation |
| GET | `/dashboard-management/v1/download_poslog_recon_report_by_region/` | `DownloadPaymentSettlementPosLogReconByRegion` | Dashboard | Download reconciliation CSV |
| GET | `/dashboard-management/v1/download_settlement_recon_payout_report_by_region_and_psp/` | `DownloadPaymentSettlementReconPayoutByRegionAndPSP` | Dashboard | Download payout CSV |
| GET | `/dashboard-management/v1/cash-report/` | `GetCashReport` | Dashboard | Cash reconciliation report |

### Miscellaneous

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/dashboard-management/v1/user/stores/` | `UserStoresListView` | Default | List user's accessible stores |
| POST | `/dashboard-management/v1/signout/` | `SignOut` | Dashboard | User logout |
| POST | `/dashboard-management/v1/adyen/session/` | `AdyenSessionView` | Dashboard | Create Adyen session |
| GET | `/dashboard-management/v1/get-printed-receipt/` | `GetOrderPrintedReceipt` | Dashboard | Formatted receipt output |
| POST | `/dashboard-management/v1/basket-item-serial-number/` | `DashboardBasketItemSerialNumberView` | Dashboard | Record serial numbers |
| POST | `/dashboard-management/v1/order-notes/` | `DashboardOrderNotesView` | Dashboard | Add notes to orders |
| POST | `/dashboard-management/v1/dashboard-email/` | `DashboardEmailView` | Dashboard | Send order email |
| POST | `/dashboard-management/v1/chat/` | `DashboardChatView` | LLM perm | LLM chat endpoint |
| GET | `/dashboard-management/v2/chat/` | `DashboardChatStreamView` | LLM perm | Streaming LLM chat (SSE) |
| POST | `/dashboard-management/v1/stock-mark/` | `DashboardStockMarkView` | Dashboard | Mark item stock status |
| GET | `/dashboard-management/v1/payment-refund-test/` | `PaymentRefundTestView` | Dashboard | Test refund flow |
| POST | `/dashboard-management/v1/user/deactivate/` | `DeactivateUserByStaffView` | Dashboard | Disable user account |
| GET | `/dashboard-management/v1/vengo/blocked-users/` | `GetVengoBlockedUsers` | Dashboard | Vengo blocked customers |
| POST | `/dashboard-management/v1/vengo/update-blocked-status/` | `UpdateCustomerBlockedStatus` | Dashboard | Block/unblock Vengo customer |

---

## 11. Analytics (`/analytics/`, `/mishipay-dashboard-analytics/`)

See [Analytics](../platform/analytics.md) for full details.

### Analytics Endpoints (`/analytics/`)

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/analytics/v1/analytics-dashboard/` | `AnalyticsDashboard` (V1) | **None** | Comprehensive analytics dashboard |
| GET | `/analytics/v1/transaction-retailer-breakdown/` | `TransactionByRetailerV2` | Dashboard, Analytics | Per-retailer transaction breakdown |
| GET | `/analytics/v1/transaction-breakdown/` | `TransactionBreakdownV2` | Dashboard, Analytics | Transaction breakdown |
| GET | `/analytics/v1/item-scanned-analytics/` | `ItemScannedAnalytics` (V1) | Dashboard, Analytics | Item scan analytics |
| GET | `/analytics/v1/ratings-analytics/` | `RatingAnalyticsV2` | Dashboard, Analytics | Rating analytics |
| GET | `/analytics/v1/login-signup-analytics/` | `LoginSignupAnalyticsV2` | Dashboard, Analytics | Login/signup analytics |
| GET | `/analytics/v2/transaction-retailer-breakdown/` | `TransactionByRetailerV2` | Dashboard, Analytics | V2 transaction by retailer |
| GET | `/analytics/v2/transaction-breakdown/` | `TransactionBreakdownV2` | Dashboard, Analytics | V2 transaction breakdown |
| GET | `/analytics/v2/ratings-analytics/` | `RatingAnalyticsV2` | Dashboard, Analytics | V2 rating analytics |
| GET | `/analytics/v2/login-signup-analytics/` | `LoginSignupAnalyticsV2` | Dashboard, Analytics | V2 login/signup analytics |

**Note:** V1 paths for transaction, rating, and login/signup now serve V2 view classes.

---

## 12. Coupons (`/coupon-management/`)

See [Coupons & Promotions](../platform/coupons-promotions.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET/POST | `/coupon-management/v1/user_coupons/` | `UserCoupons` | Default | Get/create user coupons (MS Loyalty) |
| POST | `/coupon-management/v1/validate_coupons/` | `ValidateCoupons` | Default | Validate coupon codes |
| POST | `/coupon-management/v1/redeem_coupons/` | `RedeemCoupons` | Default | Redeem coupons post-order |
| GET | `/coupon-management/v1/retailer_coupons/` | `RetailerCoupons` | Default | Get retailer-specific coupons |

---

## 13. Special Promos (`/special-promo-management/`)

See [Coupons & Promotions](../platform/coupons-promotions.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/special-promo-management/v1/validate/` | `ValidateSpecialPromo` | Default | Validate special promos (coupons, vouchers, qualifiers, lottery, donations) |
| POST | `/special-promo-management/v1/remove/` | `RemoveSpecialPromo` | Default | Remove a program from basket |
| POST | `/special-promo-management/v1/special_promo_on_fly/` | `SpecialPromoOnFly` | Default | Apply ad-hoc discount at checkout |
| POST | `/special-promo-management/v1/special_promo_summary/` | `SpecialPromoSummary` | Default | Discount summary for a program on a date |

---

## 14. Subscriptions (`/mishipay-subscription/`)

See [Subscriptions & Referrals](../platform/subscriptions-referrals.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/mishipay-subscription/v1/customer-details/` | `CustomerSubscriptionDetails` | Default | Customer subscription status |
| POST | `/mishipay-subscription/v1/stripe/create-session/` | `CreateStripeSubscriptionSession` | Default | Create Stripe Checkout session |
| POST | `/mishipay-subscription/v1/webhook/` | `SubscriptionWebhook` | AllowAny (HMAC) | Subscription/invoice webhook from MS Pay |
| POST | `/mishipay-subscription/v1/stripe/create-billing-portal-session/` | `CreateStripeBillingPortalSession` | Default | Create Stripe Billing Portal |

---

## 15. Referrals (`/referral-management/`)

See [Subscriptions & Referrals](../platform/subscriptions-referrals.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/referral-management/v1/referral_campaign/` | `ReferralCampaign` | Default | Create referral campaign |
| GET | `/referral-management/v1/referral_campaign_latest/` | `ReferralCampaignLatest` | Default | Get latest campaign for a retailer |
| GET | `/referral-management/v1/referral_campaign_latest_all_retailers/` | `ReferralCampaignLatestOfAllRetailers` | Default | Campaigns across all retailers |
| GET | `/referral-management/v1/referral_code/` | `ReferralCode` | Default | Generate/retrieve referral code |
| POST | `/referral-management/v1/referral_code_validate/` | `ReferralCodeValidate` | Default | Validate referral code |
| POST | `/referral-management/v1/referral_code_redeem/` | `ReferralCodeRedeem` | Default | Redeem referral code |

---

## 16. Staff Shifts / Cashier Kiosk (`/staff-shifts/`)

See [Cashier Kiosk](../platform/cashier-kiosk.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/staff-shifts/v1/start-shift/` | `StartShift` | Dashboard | Open a new shift on a device |
| POST | `/staff-shifts/v1/end-shift/` | `EndShift` | Dashboard, ActiveShift | Close the active shift |
| GET | `/staff-shifts/v1/list-shifts/` | `ListStaffShifts` | Dashboard | Retrieve shift details by IDs |
| GET | `/staff-shifts/v1/get-active-shift/` | `ActiveShift` | Dashboard | Check for active shift on device |

---

## 17. Inventory (`/inventory/`)

Proxy endpoints to the external Inventory Service. See [Inventory](../platform/inventory.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/inventory/v1/category/list-hierarchy/` | `CategoryListHierarchyV1View` | **None** | Get category hierarchy for a store |
| POST | `/inventory/v1/item/list/` | `ItemListV1View` | **None** | List items for a store |
| POST | `/inventory/v1/item/search/` | `ItemSearchV1View` | **None** | Search items |
| POST | `/inventory/v1/item/list-filters/` | `ItemListFiltersV1View` | **None** | Get available item filters |
| POST | `/inventory/v1/addon-group/get` | `GetAddonGroupV1View` | **None** | Get addon group by slug |
| POST | `/inventory/v1/store/create` | `StoreCreateV1View` | **None** | Create store in Inventory Service |
| POST | `/inventory/v1/retailer/create` | `RetailerCreateV1View` | **None** | Create retailer in Inventory Service |

**Note:** All inventory proxy endpoints have no authentication -- security relies on Kong API gateway.

---

## 18. Third-Party Integrations (`/third-party-integrations/`)

See [Integrations](../platform/integrations.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/third-party-integrations/v1/decathlon/signup/` | `DecathlonSignup` | None | Legacy Decathlon signup |
| POST | `/third-party-integrations/v1/decathlon/login/` | `DecathlonLogin` | None | Legacy Decathlon login/signup |
| POST | `/third-party-integrations/v1/decathlon/update-user-info/` | `UpdateDecathlonUserInfo` | Default | Update Decathlon user info |
| GET | `/third-party-integrations/v1/decathlon/check-user-exists/` | `CheckEmailExistsOnDecathlon` | Default | Check email on Decathlon |
| POST | `/third-party-integrations/v1/decathlon/forgot-password/` | `DecathlonForgotPassword` | Default | Decathlon password reset |
| POST | `/third-party-integrations/v1/customer-oauth-callback/` | `OAuthCallBackView` | None | OAuth2 authorization code callback |
| POST | `/third-party-integrations/v1/customer-oauth-logout/` | `OAuthLogOutView` | Default | OAuth2 logout (revokes SSO) |
| GET/PATCH | `/third-party-integrations/v1/customer-profile/` | `OAuthProfileUpdate` | Default | Sync/update user profile via SSO |

---

## 19. Shopping List (`/shopping-list/`)

See [Support Modules](../platform/support-modules.md) for full details.

### V1 (Legacy)

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/shopping-list/v1/shopping-list/` | `ShoppingListViewSet` | Default | List user's shopping lists |
| POST | `/shopping-list/v1/shopping-list/` | `ShoppingListViewSet` | Default | Create shopping list |
| GET | `/shopping-list/v1/shopping-list/{id}/` | `ShoppingListViewSet` | Default | Retrieve shopping list |
| PUT/PATCH | `/shopping-list/v1/shopping-list/{id}/` | `ShoppingListViewSet` | Default | Update shopping list |
| DELETE | `/shopping-list/v1/shopping-list/{id}/` | `ShoppingListViewSet` | Default | Delete shopping list |
| POST | `/shopping-list/v1/shopping-list/{id}/transfer/` | `ShoppingListViewSet` | Default | Transfer items to basket |
| POST | `/shopping-list/v1/shopping-list-item/` | `ShoppingListItemViewSet` | Default | Add item to list |
| DELETE | `/shopping-list/v1/shopping-list-item/{id}/` | `ShoppingListItemViewSet` | Default | Remove item from list |
| GET | `/shopping-list/v1/item/` | `ItemInventoryViewSet` | Default | Search items in inventory |

### V2

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/shopping-list/v2/shopping-list/` | `ShoppingListV2ViewSet` | Default | List shopping lists (v2) |
| POST | `/shopping-list/v2/shopping-list/` | `ShoppingListV2ViewSet` | Default | Create shopping list (v2) |
| GET | `/shopping-list/v2/shopping-list/{id}/` | `ShoppingListV2ViewSet` | Default | Retrieve with items (v2) |
| PATCH | `/shopping-list/v2/shopping-list/{id}/` | `ShoppingListV2ViewSet` | Default | Update shopping list (v2) |
| DELETE | `/shopping-list/v2/shopping-list/{id}/` | `ShoppingListV2ViewSet` | Default | Delete shopping list (v2) |
| POST | `/shopping-list/v2/shopping-list/{id}/transfer/` | `ShoppingListV2ViewSet` | Default | Transfer to basket (v2) |
| POST | `/shopping-list/v2/shopping-list-item/` | `ShoppingListItemV2ViewSet` | Default | Add item (v2) |
| PATCH | `/shopping-list/v2/shopping-list-item/{id}/` | `ShoppingListItemV2ViewSet` | Default | Update item quantity (v2) |
| DELETE | `/shopping-list/v2/shopping-list-item/{id}/` | `ShoppingListItemV2ViewSet` | Default | Remove item (v2) |

---

## 20. RFID (`/rfid/`)

See [Support Modules](../platform/support-modules.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| POST | `/rfid/keonn/check-epcs/` | `Keonn` | **None** | Check EPC sold/unsold status |

---

## 21. Fiscalisation (`/fiscalisation/`)

See [Support Modules](../platform/support-modules.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/fiscalisation/v1/venistar/{retailer_id}/sale-sid/` | `GetSaleSIDView` | Default | Poll for oldest unfiscalised order |
| GET | `/fiscalisation/v1/venistar/{retailer_id}/sale-sid-details/` | `SaleSIDDetailsView` | Default | Fetch full order details for fiscal registration |
| POST | `/fiscalisation/v1/venistar/{retailer_id}/registration-process/` | `RegistrationProcessView` | Default | Bind registration_process_id to order |
| POST | `/fiscalisation/v1/venistar/{retailer_id}/dc-references/` | `DCReferencesView` | Default | Complete fiscalisation with DC references |
| POST | `/fiscalisation/v1/funky-buddha/register/` | `FunkyBuddhaRegisterFiscalisation` | Default | Register fiscalisation_id (AADE mark) |

---

## 22. Easy Store Creation (`/easy-store-creation/`)

See [Support Modules](../platform/support-modules.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/easy-store-creation/get_region/` | `GetRegion` | Default | List regions for a retailer |
| GET | `/easy-store-creation/get_master_store/` | `GetMasterStore` | Default | List master stores for retailer+region |
| GET | `/easy-store-creation/get_template_details/` | `GetTemplateDetails` | Default | List store templates |
| GET | `/easy-store-creation/download_template/` | `GetBulkEasyStoreTemplate` | Default | Download CSV template for bulk upload |
| GET | `/easy-store-creation/inventory_promotion_tax_template/` | `InventoryPromotionTaxTemplate` | Default | Download sample inventory/promo CSV |
| GET/POST | `/easy-store-creation/inventory_promotion_tax_upload/<store_id>` | `InventoryPromotionTaxUpload` | Default | Upload inventory/promo/tax CSV |
| GET | `/easy-store-creation/get_theme_url/` | `GetCloudinaryTheme` | Default | Cloudinary theme upload form |
| GET/POST | `/easy-store-creation/download_bitly_file/<store_ids>/` | `BitlyFileDownload` | Default | Generate Bitly bulk URL CSV for QR codes |

---

## 23. DDF Price Activation (`/ddf-price-activation/`)

See [Support Modules](../platform/support-modules.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/ddf-price-activation/` | `DDFPriceActivatationPageView` | Admin | Price activation HTML form |
| POST | `/ddf-price-activation/download/` | `DDFPriceActivatationActionDownload` | Admin | Download CSV of items pending activation |
| POST | `/ddf-price-activation/activate/` | `DDFPriceActivatationActionActivate` | Admin | Push prices to Inventory Service |

---

## 24. Configuration (`/mishipay-config/`)

See [Authentication & Configuration](../platform/authentication.md) for full details.

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/mishipay-config/v1/stores-config-by-retailer/` | `StoresConfigByRetailerView` | Default | Store configs grouped by chain and region |

---

## 25. Client Configuration & System APIs

Registered at the root URL level in `mainServer/urls.py`.

### Client Configuration

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/static/mishipay_static_assets/master_static_conf.json` | `client_config.master_conf` | None | Master client configuration |
| GET | `/static/mishipay_static_assets/master_static_conf.v1.0.json` | `client_config.master_conf` | None | Versioned master config |
| GET | `/static/mishipay_static_assets/all_store_details.{platform}.v{M}.{m}.json` | `client_config.all_stores` | None | Per-platform store list (24h cache) |
| GET | `/static/mishipay_static_assets/app_force_update.v{M}.{m}.json` | `client_config.force_update` | None | App force update rules (24h cache) |
| GET | `/static/mishipay_static_assets/hardcoded_items.v{M}.{m}.json` | `client_config.hardcoded_items` | None | Hardcoded item data (24h cache) |
| GET | `/client-config/v1/theme/{store_id}/` | `client_config.theme` | None | Per-store theme config |
| GET | `/client-config/v1/harcoded/{store_id}/` | `client_config.harcoded` | None | Per-store hardcoded config (note: typo in URL) |

### System APIs

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/system-api/v1/stores/` | `system_api.stores` | None | Internal store listing |
| GET | `/system-api/v1/i18n-check/` | `system_api.timezone_check` | None | Timezone validation (debug) |

### Special Routes

| Method | Path | View | Auth | Description |
|--------|------|------|------|-------------|
| GET | `/` | `redirect_to_dashboard` | None | Redirects to `/dashboard/` |
| GET | `/theme/` | `ThemeAPIView` | None | Store theming API |
| GET | `/apple-app-site-association/` | `DeepLinkIOSView` | None | iOS Universal Links config |
| GET | `/get-additional-store-details/` | `AdditionalStoreDetails` | Default | Extra store metadata |
| GET | `/health-check` | `HealthCheck.as_view()` | None | Health check |
| GET/POST | `/openapi.json` or `/openapi.yaml` | drf-yasg | None | OpenAPI schema |
| GET | `/swagger-ui/` | drf-yasg | None | Swagger UI interactive docs |

---

## Endpoint Count Summary

| Domain | Public | Internal/Service | Total |
|--------|--------|-----------------|-------|
| Promotions Microservice | 14 | -- | **14** |
| DOS Module (`/d/`) | ~55 | -- | **~55** |
| Core Services | 12 | -- | **12** |
| Authentication | 1 | -- | **1** |
| Item Management | 33 | 10 | **43** |
| Order Management | 16 + 3 guest + 5 staff + 8 special | 3 | **35** |
| Payment Management | 23 legacy + 25 MS Pay | 9 | **57** |
| Customer Management | 11 + 4 address | 3 | **18** |
| Dashboard | ~25 | -- | **~25** |
| Dashboard Management | ~35 | -- | **~35** |
| Analytics | 10 | -- | **10** |
| Coupons | 4 | -- | **4** |
| Special Promos | 4 | -- | **4** |
| Subscriptions | 4 | -- | **4** |
| Referrals | 6 | -- | **6** |
| Staff Shifts | 4 | -- | **4** |
| Inventory | 7 | -- | **7** |
| Third-Party Integrations | 8 | -- | **8** |
| Shopping List | 18 | -- | **18** |
| RFID | 1 | -- | **1** |
| Fiscalisation | 5 | -- | **5** |
| Easy Store Creation | 8 | -- | **8** |
| DDF Price Activation | 3 | -- | **3** |
| Configuration | 1 | -- | **1** |
| Client Config & System | 11 | -- | **11** |
| **Total** | | | **~390** |

---

## Security Notes

The following endpoints have **no authentication** and are flagged as potential security concerns (documented in their respective module docs):

| Path | Concern |
|------|---------|
| `/core-services/v1/send-slack-msg/` | POST to send Slack messages with no auth |
| `/core-services/keycloak-migrate/` | Returns user data / validates passwords with no auth |
| `/order-management/v2/get-order-details/` | Order details with no auth (authentication_classes = []) |
| `/order-management/v1/lookup-transaction/` | Hudson transaction lookup with no auth |
| `/item-management/v1/manual-check/<basket_id>/` | Manual verification with no auth (basket_id is the only access control) |
| `/rfid/keonn/check-epcs/` | EPC status check with no auth |
| `/inventory/v1/*` (all 7 endpoints) | All inventory proxy endpoints have no auth |
| `/analytics/v1/analytics-dashboard/` | Analytics dashboard with no permission classes |

All **internal service APIs** (`api/v1/*` paths in item, order, payment, and customer management) also have no authentication -- they are designed for service-to-service calls and rely on network isolation.

---

## Cross-References

- [Promotions Service API Reference](../promotions-service/api-reference.md) -- Full request/response schemas for the promotions microservice
- [Platform Overview](../platform/overview.md) -- URL routing structure and middleware chain
- [Authentication](../platform/authentication.md) -- Auth mechanisms and permission classes
- [Items](../platform/items.md) -- Item management views and business logic
- [Orders](../platform/orders.md) -- Order lifecycle and processing
- [Payments](../platform/payments.md) -- Payment gateway integrations
- [Customers](../platform/customers.md) -- Customer profiles, loyalty, and consent
- [Dashboard](../platform/dashboard.md) -- Dashboard management operations
- [Retailer Modules](../platform/retailer-modules.md) -- DOS module and retailer-specific endpoints
