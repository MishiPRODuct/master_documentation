# MishiPay Platform — Retail Customers

> Last updated by: Task T20 — Retail Customers (models, views, loyalty, privacy, marketing consent)

## Overview

The `mishipay_retail_customers` Django app manages all customer-facing data beyond basic authentication: customer addresses (billing/delivery), feedback & ratings, retailer-specific customer profiles (loyalty cards, boarding passes, tax IDs, user groups), privacy policy consent, push notification permissions, marketing consent, and past purchase records. It also orchestrates integration with **15+ external loyalty services** across different retailers (Picwic, Eroski, Azpiral/Spar/Londis, Desigual, Virgin, Carters, Grandiose/TalonOne, Flying Tiger, Seventh Heaven/Reelo, MMI, Event Network, and more).

The app exposes both **public-facing authenticated endpoints** (used by mobile/web clients) and **internal service-to-service APIs** (used by the Customer Management microservice).

## Key Concepts

| Concept | Description |
|---------|-------------|
| **RetailerSpecificCustomerProfile** | A per-retailer, per-customer profile that stores loyalty data, tax info, boarding pass scans, nationality, and user group. A customer can have multiple profiles — one per retailer. |
| **Loyalty Card Scan** | The primary mechanism for linking a customer to a retailer's loyalty programme. Each retailer has its own loyalty service implementation dispatched via `CUSTOMER_CALLABLE_SERVICE`. |
| **Privacy Policy** | Versioned opt-in/opt-out consent tracked per customer + store type, supporting "don't ask again" and MishiPay app vs white-label app distinction. |
| **Marketing Consent** | GDPR-compliant consent tracking with versioning, per-retailer and per-MishiPay consent, IP hashing, and configurable default behaviour per region. |
| **BuyerAddress** | A newer address model (v2) supporting billing/shipping types, multiple sources (MishiPay, Apple Pay, GPay), and full CRUD via REST API. |
| **CustomerAddress** | The legacy address model supporting billing/delivery types, linked to basket during checkout. |
| **Transaction Token** | A short-lived (10-minute) UUID token used for securing certain transaction-related operations. |

## Architecture

```
mishipay_retail_customers/
├── __init__.py                  # Constants: address/gender choices, PrivacyPolicyChoices, UserGroupChoices, CommunicationType
├── apps.py                      # MishipayRetailCustomersConfig
├── models.py                    # 9 models
├── serializers.py               # 20+ serializers
├── views.py                     # 10 view classes (public-facing)
├── urls.py                      # URL routing (public + internal)
├── admin.py                     # Admin registrations for all models
├── app_settings.py              # Constants, error messages, function maps, loyalty config, consent mappings
├── api_utils.py                 # Internal service API views (3 classes)
├── customer_address.py          # Address add/update business logic
├── customer_feedback.py         # Feedback submission logic
├── common_functions.py          # BoardingPass decoder utility
├── utils.py                     # Push notification helpers, guest user context, Eroski/7H encryption
├── permissions.py               # JwtTokenPermission (JWT-based auth for unauthenticated flows)
├── analytics.py                 # Rating analytics recording
├── retailer_service.py          # 15+ retailer-specific loyalty service implementations
├── exceptions.py                # ValidationException
├── api/                         # REST API v1 (BuyerAddress CRUD)
│   ├── urls.py
│   └── v1/
│       ├── urls.py
│       ├── views.py             # BuyerAddressView, BuyerAddressListView
│       └── serializers.py       # BuyerAddressSerializer
├── virgin_loyalty_utility/      # Virgin Megastore loyalty SOAP client
│   └── client.py
├── carters_loyalty_utility/     # Carter's loyalty REST client
│   └── client.py
└── reelo_loyalty_utility/       # Reelo loyalty REST client (Seventh Heaven)
    ├── main.py                  # ReeloLoyaltyClient
    └── loyalty.py               # SeventhHeavensLoyalty (coupon listing/redemption)
```

### Design Patterns

- **Strategy Pattern for Loyalty Services**: `CUSTOMER_CALLABLE_SERVICE` in `app_settings.py` maps `(customer_resource_type, store_type)` → callable service function. Each retailer has its own loyalty implementation in `retailer_service.py`.
- **Strategy Pattern for Addresses**: `ADD_CUSTOMER_ADDRESS_FUNCTION_MAP` and `UPDATE_CUSTOMER_ADDRESS_FUNCTION_MAP` dispatch address operations per store type.
- **Serializer-per-Entity**: `PROFILE_ENTITY_SERIALIZER_MAP` maps profile entity types (`tax_info`, `boarding_pass_scan`, `dufry_info`, `loyalty_card_scan`, `sso_update_profile`, `user_group`) to their respective serializers.
- **Eager Row Locking**: All mutating views use `Customer.objects.select_for_update()` to prevent race conditions (referenced as GPP-1046).
- **Internal Service APIs**: The `api_utils.py` module provides service-to-service endpoints (`AddOrUpdateCustomerAddressService`, `UpdateCustomerAddressService`, `AddCustomerFeedbackService`) used by the Customer Management microservice.

## Data Models

Source: `mainServer/mishipay_retail_customers/models.py`

### TransactionToken

Short-lived token for securing transaction-related operations.

| Field | Type | Description |
|-------|------|-------------|
| `key` | UUID | Unique token (auto-generated) |
| `order_id` | UUID | Associated order |
| `expiry_date` | DateTime | Auto-set to now + 10 minutes on creation |
| `last_used` | DateTime | Nullable; set when token is consumed |
| `extra_data` | JSON | Arbitrary additional data |
| `created` / `updated` | DateTime | Timestamps |

**Key behaviour**: Once expired, the token cannot be modified (`save()`, `set_used()`, `expire()` all raise `AssertionError`). The `active` property checks `expiry_date >= now`.

### CustomerAddress (Legacy)

Per-customer address for billing/delivery during checkout.

| Field | Type | Description |
|-------|------|-------------|
| `address_id` | UUID | Auto-generated identifier |
| `customer` | FK → Customer | Owner |
| `postcode` | CharField(256) | Postal code |
| `country_code` | CharField(2) | ISO country code |
| `country_area` | CharField(256) | Region/state |
| `city` | CharField(256) | City |
| `name_of_user_to_refer` | CharField(256) | Recipient name |
| `email_of_user_to_refer` | EmailField(256) | Recipient email |
| `delivery_instruction` | TextField | Delivery notes |
| `address_1` / `address_2` | CharField(256) | Address lines |
| `phone_number` | PhoneNumberField | Contact phone |
| `fax_number` | PhoneNumberField | Fax (nullable) |
| `type_of_address` | CharField(10) | `Billing` or `Delivery` |

### CustomerFeedback

Post-transaction star rating and feedback.

| Field | Type | Description |
|-------|------|-------------|
| `customer` | FK → Customer | Author |
| `store` | FK → Store | Store where feedback was given |
| `order` | FK → Order | Nullable; linked order |
| `feedback` | TextField | Free-text comment |
| `rating` | IntegerField | Star rating |
| `tags` | TextField | Issue tags (comma-separated) |

**Business rules** (from `customer_feedback.py`):
- 1-star ratings require minimum 20 characters of feedback (unless from Android kiosk/mPOS).
- iOS placeholder text in EN/HE/FR/NL/DE is rejected as not genuine feedback.
- An order can only be rated once (`order_obj.is_rated` check).
- Maximum 10 feedbacks without an order per customer.

### PrivacyPolicyInformation

Versioned privacy policy consent tracking.

| Field | Type | Description |
|-------|------|-------------|
| `customer` | FK → Customer | User |
| `store_type` | CharField(64) | Lowercased store type (validated) |
| `region` | CharField(64) | Store region (validated) |
| `sub_region` | CharField(64) | Store sub-region (validated) |
| `opt_in` | BooleanField | Current consent status |
| `opt_in_date` / `opt_out_date` | DateTime | Timestamps of consent changes |
| `opted_version` | CharField(8) | Version of the policy consented to |
| `dont_ask_again` | BooleanField | Permanent opt-out flag |
| `for_mishipay_app` | BooleanField | Whether this applies to MishiPay app vs white-label |
| `is_expired` | BooleanField | Soft-delete for outdated consent records |
| `privacy_policy_type` | CharField(32) | Type: `saturn_privacy_policy` or `adyen_relay_privacy_policy` |

### PushNotificationPermission

Controls push notification delivery per customer per retailer.

| Field | Type | Description |
|-------|------|-------------|
| `customer` | FK → Customer | User |
| `store_type` | CharField(64) | Retailer store type |
| `show_notifications` | BooleanField | Whether notifications are enabled |
| `valid_till` | DateTime | Expiration (renewed to +1 year on update) |

**Unique constraint**: `(customer, store_type)`

### RetailerSpecificCustomerProfile

Central per-retailer customer profile storing loyalty, tax, boarding pass, and demographic data.

| Field | Type | Description |
|-------|------|-------------|
| `retailer_customer_profile_id` | UUID (PK) | Primary key |
| `customer` | FK → Customer | Owner |
| `store_type` | CharField(31) | Retailer type (dynamic choices from `get_store_type_choices()`) |
| `tax_id` | CharField(10) | Tax identifier |
| `loyalty_number` | CharField(127) | Loyalty card number (validated: digits only) |
| `loyalty_info` | JSON | Full loyalty service response data |
| `phone_number` | CharField(17) | Validated phone (+999999999 format) |
| `nationality` | CharField(50) | Country choice |
| `gender` | CharField(10) | `male` or `female` |
| `age_above_18` | BooleanField | Age verification flag |
| `duty_applicable` | BooleanField | Whether duty-free applies |
| `encoded_boarding_pass_barcode` | TextField | Raw boarding pass barcode |
| `boarding_pass_scan_data` | JSON | Decoded boarding pass data |
| `no_of_legs` | CharField(100) | Number of flight legs |
| `user_group` | CharField(150) | `airport_staff_user`, `special_customer`, or `not_available` |
| `extra_data` | JSON | Arbitrary additional data (transit info, current leg, etc.) |

### PastPurchase / PastPurchaseRecord

Tracks customer purchase history per retailer.

| Model | Field | Type | Description |
|-------|-------|------|-------------|
| PastPurchase | `retailer_specific_customer_profile` | FK → RetailerSpecificCustomerProfile | Profile link |
| PastPurchaseRecord | `past_purchase` | FK → PastPurchase | Parent purchase |
| PastPurchaseRecord | `key` / `value` | CharField(64) | Key-value purchase data |

### MarketingConsentInfo

GDPR-compliant marketing consent with auto-incrementing version.

| Field | Type | Description |
|-------|------|-------------|
| `marketing_consent_id` | UUID (PK) | Primary key |
| `customer` | FK → Customer | User |
| `retailer` | CharField(64) | Store type of retailer |
| `store_id` | UUID | Specific store |
| `region` | CharField(30) | Store region |
| `sub_region` | CharField(30) | Store sub-region |
| `communication_type` | CharField(50) | `email`, `push_notification`, or `email_and_notification` |
| `consent_version` | IntegerField | Auto-incremented on creation per (customer, retailer, region) |
| `consent_value` | BooleanField | Current consent status |
| `hashed_ip_address` | CharField(100) | SHA-256 hash of client IP for audit |
| `timestamp` | DateTime | When consent was given/changed |

**Key behaviour**: On creation, `save()` automatically finds the latest `consent_version` for the same `(customer, retailer, region)` and increments it.

### BuyerAddress (v2)

Modern address model with richer fields and full CRUD API.

| Field | Type | Description |
|-------|------|-------------|
| `address_id` | UUID (PK) | Primary key |
| `address_type` | CharField(10) | `billing` or `shipping` |
| `address_source` | CharField(10) | `mishipay`, `apple_pay`, or `gpay` |
| `customer` | FK → Customer | Owner |
| `contact_first_name` / `contact_last_name` | CharField(256) | Contact name |
| `address1` / `address2` / `address3` | CharField(256) | Address lines |
| `postal_code` | CharField(12) | Postal code |
| `city` | CharField(256) | City |
| `area` | CharField(256) | Area/district |
| `country_code` | CharField(2) | ISO code |
| `country` | CharField(256) | Country name |
| `contact_number` | CharField(17) | Phone (validated regex) |
| `additional_details` | JSON | Extra data |

## API Endpoints

### Public-Facing Endpoints (require `IsAuthenticated` unless noted)

Source: `mainServer/mishipay_retail_customers/urls.py`

| Endpoint | Method | View | Description |
|----------|--------|------|-------------|
| `v1/add-or-update-customer-address/` | POST | `AddOrUpdateCustomerAddress` | Add or update billing/delivery addresses |
| `v1/update-customer-address/` | PUT | `UpdateCustomerAddress` | Update existing addresses |
| `v1/get-additional-customer-details/` | GET | `GetAdditionalCustomerDetails` | Retrieve full customer context (baskets, addresses, profile, payment methods, privacy policies, subscriptions) |
| `v1/add-customer-feedback/` | POST | `AddCustomerFeedback` | Submit feedback/rating. Auth: `IsAuthenticated \| JwtTokenPermission` |
| `v1/privacy-policy/` | GET, POST | `PrivacyPolicy` | Get/update privacy policy consent |
| `v1/update-notification-permission/` | POST | `UpdateNotificationPermission` | Toggle push notifications |
| `v1/retailer-customer-profile/` | POST, PUT, DELETE | `UpdateRetailerSpecificCustomerProfile` | Create/update/delete retailer profiles (loyalty scan, tax info, boarding pass, etc.). Auth: `IsAuthenticated + ActiveStaffShiftPermission` |
| `v1/loyalty-otp/` | POST | `OTPValidationView` | OTP validation for loyalty redemption |
| `v1/marketing-consent/` | GET, POST | `MarketingConsentView` | Get/update marketing consent. Auth: `IsAuthenticated \| JwtTokenPermission` |
| `v1/nationalities/` | GET | `NationalitiesView` | List available nationalities |
| `v1/guest/marketing-consent/` | GET, POST | `MarketingConsentGuestView` | Guest marketing consent (no auth) |

### BuyerAddress REST API (v2)

Source: `mainServer/mishipay_retail_customers/api/v1/urls.py`

| Endpoint | Method | View | Description |
|----------|--------|------|-------------|
| `api/v1/address/` | POST | `BuyerAddressView` | Create new address |
| `api/v1/address/<uuid:address_id>/` | PUT | `BuyerAddressView` | Update address |
| `api/v1/address/<uuid:address_id>/` | DELETE | `BuyerAddressView` | Delete address |
| `api/v1/address-list/` | GET | `BuyerAddressListView` | List all customer addresses |

### Internal Service APIs (no auth required — service-to-service)

Source: `mainServer/mishipay_retail_customers/api_utils.py`

| Endpoint | Method | View | Description |
|----------|--------|------|-------------|
| `api/v1/add-or-update-customer-address/` | POST | `AddOrUpdateCustomerAddressService` | Internal address add/update |
| `api/v1/update-customer-address/` | POST | `UpdateCustomerAddressService` | Internal address update |
| `api/v1/add-customer-feedback/` | POST | `AddCustomerFeedbackService` | Internal feedback submission |

## Business Logic

### GetAdditionalCustomerDetails — The Customer Context Endpoint

`GetAdditionalCustomerDetails.get()` (`views.py:276`) is the primary endpoint called when a customer opens a store in the app. It returns a comprehensive customer context:

1. **SSO Profile Sync** — Syncs the customer profile with SSO (e.g., Decathlon OpenID).
2. **Basket Management** — Expires other-store baskets and completed baskets, returns active basket IDs.
3. **Wishlist Data** — Returns wishlist IDs with opt-in status per wishlist.
4. **Customer Addresses** — Serializes all saved addresses.
5. **Retailer Profile** — Returns `RetailerSpecificCustomerProfile` data with boarding pass info, loyalty data, user group.
6. **Payment Methods** — Fetches default payment methods (legacy + MS Pay).
7. **Auth Token** — Returns Django REST token (even if login was via Keycloak).
8. **Privacy Policy Status** — For stores with privacy policy configured, returns per-policy-type opt-in status and whether consent needs to be asked.
9. **Subscription Status** — Checks if coffee subscription is active.
10. **Retailer-Specific Enrichment** — For Carter's, fetches live loyalty details post-transaction.

### Retailer Customer Profile Lifecycle

The `UpdateRetailerSpecificCustomerProfile` view (`views.py:801`) handles the full CRUD lifecycle for retailer profiles:

**POST (Create)**:
1. Validates prerequisites (`store_id`, `customer_resource_type`).
2. Handles encrypted loyalty numbers (Eroski AES-CBC decryption + checksum).
3. Dispatches to the appropriate serializer via `PROFILE_ENTITY_SERIALIZER_MAP`.
4. If profile already exists: returns existing profile data (special handling for Hudson, Seventh Heaven, Grandiose).
5. If a `CUSTOMER_CALLABLE_SERVICE` exists for the `(resource_type, store_type)`: calls the external loyalty service, attaches response as `service_data`.
6. For Carter's phone lookup returning multiple emails: returns email list without creating profile.
7. For Seventh Heaven: encrypts phone number with Fernet before saving.
8. Creates the `RetailerSpecificCustomerProfile` record.

**PUT (Update)**: Similar flow but updates existing profile. Validates service response if `validation_required` flag is set.

**DELETE**: Deletes the profile record entirely.

### Loyalty Service Architecture

Source: `mainServer/mishipay_retail_customers/retailer_service.py` + utility clients

The loyalty system uses a dispatch map (`CUSTOMER_CALLABLE_SERVICE` in `app_settings.py`) to route loyalty operations to retailer-specific implementations:

```python
CUSTOMER_CALLABLE_SERVICE = {
    "loyalty_card_scan": {
        "picwicstoretype": picwic_loyalty_service,
        "eroskistoretype": eroski_loyalty_service,
        "sparstoretype": azpiral_loyalty_service,
        # ... 15+ retailers
    },
    "sso_update_profile": {
        "decathlonstoretype": decathlon_update_profile,
    }
}
```

#### Loyalty Service Implementations

| Retailer | Function | Protocol | Key Features |
|----------|----------|----------|--------------|
| **Picwic** | `picwic_loyalty_service` | SOAP/XML | XML template rendering, namespace stripping, loyalty points extraction |
| **Eroski** | `eroski_loyalty_service` | SOAP/XML | Card number validation (13 digits, scheme prefix "230"), basic auth, Spanish response key mapping |
| **Azpiral** (Spar, Londis) | `azpiral_loyalty_service` | REST/XML | Query param-based lookup, XML response parsing, error code mapping (PBERR01-17) |
| **Desigual** | `desigual_loyalty_service` | Local | Simple email-based loyalty (no external call) |
| **Common** (Muji, eMetro, Funky Buddha, 18 Degrees) | `common_loyalty_service` | Local | Direct number storage; eMetro requires exactly 16 digits |
| **Event Network** | `event_network_loyalty_service` | Local | Configurable length validation (fixed/range), alphanumeric support, confirmation popup flow |
| **Virgin** | `virgin_loyalty_service` | SOAP/XML | `VirginLoyaltyClient` — GET customer by mobile, auto-register if not found, SHA1 token auth |
| **Emma's Garden** | `emmas_garden_loyalty_service` | REST | Coupon validation via MS Loyalty Service |
| **Carter's** | `carters_loyalty_service` | REST | `CartersLoyaltyClient` — Auth token management (auto-refresh), email/phone lookup, reward details |
| **Grandiose** | `grandiose_loyalty_service` | REST | `GrandioseLoyaltyClientTalonOne` — Login/signup flow with phone, 303 redirect for signup, customer details fetch |
| **MMI** | `mmi_loyalty_service` | Local | Phone number SHA1 hash lookup against Django User table, license DXB support |
| **Flying Tiger** | `flying_tiger_loyalty_service` | REST | `FlyingTigerLoyalty` client — customer lookup + coupon listing |
| **Florist** | `florist_loyalty_service` | Local | Phone SHA1 hash lookup, business metadata (company name, VAT, commercial reg) |
| **Flash Facial** | `flash_facial_loyalty_service` | Local | Emirates ID scan — captures biometric/document data |
| **Seventh Heaven** | `seventh_heaven_loyalty_service` | REST | `ReeloLoyaltyClient` — customer lookup, coupon code generation with unique suffix |
| **Hudson** | `hudson_loyalty_service` | REST | Imported from `mishipay_items.hudson_utility.loyalty_client` |
| **Decathlon** (SSO) | `decathlon_update_profile` | REST | Profile update via `CustomerInterface.openid_decathlon_update_profile_with_sso()` |

#### External Loyalty Clients

**VirginLoyaltyClient** (`virgin_loyalty_utility/client.py`):
- SOAP-based client with GET (lookup by mobile) and POST (register new customer).
- SHA1 token generation from mobile + date.
- Auto-registers customer if GET returns 404.
- Handles phone number conflicts (409) by falling back to GET.

**CartersLoyaltyClient** (`carters_loyalty_utility/client.py`):
- REST client with token-based auth (auto-refresh on 401, cached with expiry).
- Supports lookup by email (`check_loyalty_customer`) and phone (`get_loyalty_details_using_phone`).
- Post-transaction customer details fetch for enriching profile.
- Reward details, tier info, and member status.

**ReeloLoyaltyClient** (`reelo_loyalty_utility/main.py`):
- REST client for Reelo POS integrations.
- Customer details, coupon authentication, OTP-based redemption (cashback/coupon).
- Bill submission for points accrual, point reversal, bill updates.
- Menu sync to Reelo for item-level tracking.
- Per-store configuration via `GlobalConfig` + `store.extra_data` (merchant_id, customer_key).

### Privacy Policy Logic

Source: `mainServer/mishipay_retail_customers/views.py:476`

The `PrivacyPolicy` view manages versioned consent:

**GET Flow**:
1. Check if store has privacy policy configured.
2. Check "don't ask again" flag — if set, return `ask_consent: false`.
3. Query for existing consent record matching current version.
4. If no record found: return `ask_consent: true`.
5. If version matches latest: return `ask_consent: false`.

**POST Flow**:
1. Find or create `PrivacyPolicyInformation` record.
2. If creating new record: expire all previous records for the same store type.
3. Update opt-in status, timestamps, and `dont_ask_again` flag.
4. Cascade updates to all related policies (same customer + app scope).

**Key configuration**: Privacy policy types and versions are configured per store via `store_properties` (entity: `privacy_policy`). The `legal_requirements` section of store settings defines policy versions and additional filter fields.

### Marketing Consent Logic

Source: `mainServer/mishipay_retail_customers/views.py:1147`

The `MarketingConsentView` supports both retailer-specific and MishiPay-platform consent:

**GET** returns an array of consent items:
- **Retailer consent**: Checks `MarketingConsentInfo` for the retailer. Returns `true` if explicit consent exists OR if the store region is in `CONSENT_DEFAULT_TRUE_REGIONS` (and the store type is not in the pre-tick-false config).
- **MishiPay consent**: Same logic but for MishiPay-platform consent (with a default store UUID for MishiPay: `11111111-1111-1111-1111-111111111111`).
- Text codes are mapped per store type via `TEXT_CODE_MAPPING` (e.g., `flyingtigerstoretype` → `flyingtiger_consent`).
- Privacy links are provided per store type + region via `PRIVACY_LINK_MAPPING`.

**POST** accepts a `consent_details` JSON array and creates `MarketingConsentInfo` records with:
- SHA-256 hashed IP address for audit trail.
- Communication type (email, push notification, or both).
- Auto-versioned consent records.

### Customer Address Operations

Source: `mainServer/mishipay_retail_customers/customer_address.py`

Address operations are dispatched per store type via function maps:

**Add Address** (`mishipay_add_customer_address`):
1. Check if address already exists for customer (reject duplicates).
2. Validate and save delivery address (within atomic transaction).
3. Validate and save billing address.
4. Sync addresses to basket via `mishipay_add_or_update_basket_address`.

**Update Address** (`mishipay_update_customer_address`):
1. Look up both delivery and billing address objects by ID.
2. Validate exactly 2 addresses exist.
3. Update each address and sync to basket.

### Boarding Pass Processing

Source: `mainServer/mishipay_retail_customers/common_functions.py`

The `BoardingPass` class decodes IATA boarding pass barcodes for duty-free stores:

1. **Decoding**: Uses `python_bcbp` library to decode barcode data.
2. **Leg Detection**: Determines current flight leg based on store airport code and arrival/departure store configuration.
3. **Validation**: Checks flight date is within ±2 days of current date, validates source/destination airport codes against known IATA codes.
4. **DDF Integration**: Formats flight codes and looks up airline names from `GlobalConfig` (`DDF_CONFIG`).

### Encryption & Security

Source: `mainServer/mishipay_retail_customers/utils.py`

- **Eroski loyalty decryption**: AES-CBC decryption with unpadding for encrypted loyalty card numbers from SDK.
- **Eroski checksum**: EAN-13 check digit calculation for loyalty card validation.
- **Seventh Heaven encryption**: Fernet symmetric encryption for phone numbers before storage.
- **IP hashing**: SHA-256 hashing of client IP for marketing consent audit trail.

### JwtTokenPermission

Source: `mainServer/mishipay_retail_customers/permissions.py`

A custom DRF permission class that validates JWT tokens for unauthenticated flows (e.g., feedback from receipt links). The JWT contains `order_id`, `timestamp`, and `amount`, and expires after 15 minutes.

## Dependencies

### This Module Depends On

| Module | Usage |
|--------|-------|
| `dos.models.Customer` | Core customer model |
| `dos.models.Store` | Store model (store_type, region, store_properties, extra_data) |
| `mishipay_retail_orders.models.Order` | Order lookup for feedback, JWT-based flows |
| `mishipay_items.models.Basket` | Basket management, address sync |
| `mishipay_core.common_functions` | Generic response structure, store settings, email validation |
| `mishipay_core.interfaces.CustomerInterface` | SSO sync, Decathlon profile update |
| `mishipay_core.interfaces.PaymentInterface` | Payment method retrieval |
| `mishipay_config.models.GlobalConfig` | Loyalty configuration, consent config |
| `microservices_client.CustomerManagementClient` | Service-to-service calls for address/feedback |
| `mishipay_items.grandiose_utility.client` | Grandiose TalonOne loyalty client |
| `mishipay_items.flying_tiger_utility.loyalty` | Flying Tiger loyalty client |
| `mishipay_items.hudson_utility.loyalty_client` | Hudson loyalty client |
| `mishipay_subscription` | Coffee subscription status check |
| `mishipay_dashboard.util` | Rating analytics |

### Modules That Depend On This

| Module | Usage |
|--------|-------|
| Customer Management Microservice | Calls internal APIs for address/feedback |
| `mishipay_retail_orders` | Customer profile data during order creation |
| `mishipay_retail_payments` | Payment method defaults |
| `mishipay_items` | Loyalty-driven pricing/promotions |

## Configuration

### Constants & Choices (`__init__.py`)

| Constant | Value | Description |
|----------|-------|-------------|
| `BILLING_ADDRESS` / `DELIVERY_ADDRESS` | `'Billing'` / `'Delivery'` | Legacy address types |
| `BILLING_ADDRESS_TYPE` / `SHIPPING_ADDRESS_TYPE` | `'billing'` / `'shipping'` | v2 address types |
| `ADDRESS_SOURCE_MISHIPAY` / `APPLEPAY` / `GPAY` | `'mishipay'` / `'apple_pay'` / `'gpay'` | Address sources |
| `TRANSACTION_TOKEN_VALIDITY` | `10` (minutes) | Token expiry time |
| `PrivacyPolicyChoices` | `saturn_privacy_policy`, `adyen_relay_privacy_policy` | Policy types |
| `UserGroupChoices` | `airport_staff_user`, `special_customer`, `not_available` | User group types |
| `CommunicationType` | `email`, `push_notification`, `email_and_notification` | Consent communication types |

### App Settings (`app_settings.py`)

| Setting | Description |
|---------|-------------|
| `CUSTOMER_FEEDBACK_LIMIT` | Max 10 feedbacks without an order |
| `CONSENT_TRANSACTION_RESET_COUNT` | Re-ask consent after 5 transactions |
| `DEFAULT_STORE_ID_MISHIPAY` | UUID for MishiPay's own consent store |
| `EROSKI_LOYALTY_CARD_NUMBER_LENGTH` | 13 digits |
| `EROSKI_LOYALTY_CARD_SCHEME_NUMBER` | Prefix "230" |
| `TEXT_CODE_MAPPING` | Store type → Lokalise text code for consent UI |
| `PRIVACY_LINK_MAPPING` | Store type + region → privacy policy URL |
| `SSO_PHONE_VALIDATION` | Decathlon phone regex per country (60+ countries) |
| `MARKETING_CONSENT_RETAILER_REGION_TEXT_CODE_MAPPING` | Override text codes per region |

### Environment Variables

| Variable | Usage |
|----------|-------|
| `EROSKI_SDK_LOYALTY_NUMBER_DECRYPTION_KEY` | AES key for Eroski loyalty decryption |
| `EROSKI_SDK_LOYALTY_NUMBER_DECRYPTION_INITIALIZATION_VECTOR` | AES IV for Eroski loyalty decryption |
| `SEVENTH_HEAVEN_ENCRYPTION_KEY` | Fernet key for phone number encryption |
| `SECRET_KEY` | Django secret key (used for JWT encoding/decoding) |
| `MS_LOYALTY_SERVICE` | URL for microservice loyalty service |
| `CONSENT_DEFAULT_TRUE_REGIONS` | Regions where marketing consent defaults to true |

## Notable Patterns

1. **Dual Address System**: The legacy `CustomerAddress` model is used by the address add/update flow (synced to baskets via microservice), while the newer `BuyerAddress` model provides a clean REST API with source tracking (Apple Pay, GPay). Both coexist.

2. **Phone-to-User Lookup via SHA1 Hash**: MMI and Florist loyalty services look up customers by hashing phone numbers (with and without leading zero, with country code) and matching against Django `User.username`. This allows phone-based identification without storing plain phone numbers in the username field.

3. **Loyalty Confirmation Popup Flow**: Event Network supports a configurable popup-based loyalty confirmation (bypassing validation) controlled by `store.store_properties` feature flags.

4. **Per-Store Loyalty Configuration**: Loyalty service URLs, API keys, templates, and response structures are all stored in store settings (`get_store_settings(store_dict).customer`) rather than hardcoded, enabling per-store customisation without code changes.

5. **Consent Version Auto-Increment**: `MarketingConsentInfo.save()` automatically computes the next version number per `(customer, retailer, region)` — creating an immutable audit trail.

6. **Lazy Imports in app_settings.py**: Several imports are placed at the bottom of `app_settings.py` to avoid circular dependency issues between models, serializers, and retailer services.

## Open Questions

1. **CustomerAddress vs BuyerAddress migration**: It's unclear whether there's a planned migration path from the legacy `CustomerAddress` to the newer `BuyerAddress` model, or if both will continue to coexist.

2. **OTPValidationView**: This view is referenced in URL routing but its implementation wasn't fully explored. It likely handles OTP validation for loyalty redemption flows (e.g., Reelo/Seventh Heaven).

3. **MarketingConsentGuestView**: Referenced in URLs but not fully explored. Appears to handle consent for guest (unauthenticated) users.

4. **NationalitiesView**: Referenced in URLs but not fully explored. Likely returns the list of available `CountryChoice.CHOICES`.

5. **Test coverage**: The test directory for this module was not explored. Coverage scope and patterns are unknown.
