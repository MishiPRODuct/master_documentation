# MishiPay Platform — Third-Party Integrations & Microservices Client

> Last updated by: Task T25 — Third-Party Integrations & Microservices Client

## Overview

The MishiPay platform communicates with external services and internal microservices through two dedicated Django apps and a gRPC service definition. The `third_party_integrations` app (33 files, ~4,000 LOC) implements a custom OAuth2 provider framework primarily for **Decathlon** single sign-on. The `microservices_client` module (8 files, ~180 LOC) provides an internal HTTP client abstraction layer that routes requests between platform components. Two protobuf-generated files (`tax_pb2.py`, `tax_pb2_grpc.py`) define a gRPC **Tax** service for tax evaluation and tax table management.

## Key Concepts

- **OAuth2 Provider Framework**: A custom, allauth-inspired OAuth2 implementation tailored to MishiPay's multi-tenant retail model. Each retailer's store can be associated with an `OAuthApplication` that defines provider-specific API endpoints, keys, and configuration.
- **Decathlon SSO**: The dominant use case — customers authenticate via Decathlon's identity API. The platform supports two parallel integration approaches: an older direct-API approach (`views/decathlon.py`) and a newer OAuth2 callback-based flow (`providers/openid/`).
- **Red Discount / Loyalty Tiers**: A loyalty discount system with tiers (Bronze=5%, Silver=5%, Gold=7%, Platinum=10%) applied based on `customer_tier` from the OAuth provider.
- **Microservices Client**: An internal routing layer that uses Django's test `Client()` to make HTTP requests to other internal endpoints, acting as an in-process service bus rather than true network-based microservice communication.
- **Tax gRPC Service**: A protobuf-defined service for evaluating basket taxes, managing per-store tax tables, and supporting tax-inclusive/exclusive pricing.

## Architecture

### Module Structure

```
third_party_integrations/          # Django app — OAuth2 & external auth
├── __init__.py                    # AuthProviders enum, AccessTokenMethods, RedDiscountMap
├── models.py                      # OAuthApplication, OAuthAccount, OAuthToken
├── urls.py                        # 8 URL patterns
├── serializers.py                 # 4 DRF serializers
├── admin.py                       # 3 admin registrations
├── app_settings.py                # AppSettings (SOCIALACCOUNT_ prefix)
├── constants.py                   # PROVIDER_DECATHLON, DECATHLON_ERROR_MAP
├── front_end_messages.py          # 11 translatable message constants
├── providers/
│   ├── base.py                    # Abstract Provider base class
│   └── openid/
│       ├── provider.py            # OAuth2Provider, DecathlonOAuth2Provider
│       ├── client.py              # OAuth2Client base (token exchange, user CRUD)
│       ├── decathlon.py           # OAuth2Decathlon — Decathlon-specific client
│       ├── views.py               # OAuthCallBackView, DecathlonOAuthCallBackView
│       └── serializer.py          # UserModel, CustomerModel, DecathlonPostData serializers
├── views/
│   ├── customer.py                # OAuthCallBackView, OAuthLogOutView, OAuthProfileUpdate
│   └── decathlon.py               # DecathlonSignup/Login/CheckEmail/Update/ForgotPassword
├── tests/
│   └── test_api.py                # TestDecathlonAuthApi (signup/login tests)
└── migrations/                    # 10 migration files (2019–2022)

microservices_client/              # Internal service client abstraction
├── core.py                        # BaseClient (Django test Client, language header)
├── client.py                      # AnalyticsRecordClient
├── item_microservice_client.py    # ItemManagementClient (10 methods)
├── payment_microservice_client.py # PaymentManagementClient (8 methods + 1 disabled)
├── order_microservice_client.py   # OrderManagementClient (3 methods)
├── customer_microservice_client.py# CustomerManagementClient (3 methods)
└── thirdparty_client.py           # LoginSignupClient, InformationUpdateClient

mainServer/
├── tax_pb2.py                     # Generated protobuf messages for Tax service
└── tax_pb2_grpc.py                # Generated gRPC stubs for Tax service
```

### Design Patterns

1. **Provider Pattern**: Abstract `Provider` base class (`providers/base.py:10`) defines lifecycle enums (`AuthProcess`, `AuthAction`, `AuthError`) and interface methods. Concrete providers (`OAuth2Provider`, `DecathlonOAuth2Provider`) implement callback and profile sync dispatch.

2. **Client Factory**: `get_client()` (`providers/openid/client.py:399`) maps provider IDs to client classes — `openid` maps to `OAuth2Client`, `openid_decathlon` maps to `OAuth2Decathlon`. This dispatches per-application provider behavior.

3. **Dual Integration Path**: Decathlon has two integration implementations:
   - **Legacy** (`views/decathlon.py`): Direct API calls using store-level `THIRD_PARTY_INTEGRATION` settings from Django config. Uses `get_current_third_party_settings()` for per-region config lookup. Classes: `DecathlonSignup`, `DecathlonLogin`, `CheckEmailExistsOnDecathlon`, `UpdateDecathlonUserInfo`, `DecathlonForgotPassword`.
   - **OAuth2 Flow** (`providers/openid/`): Authorization code grant via `OAuthApplication` model config. Uses `DecathlonOAuthCallBackView` for callback handling with JWT token introspection. Supports profile sync and update via `DecathlonOAuthProfileUpdateView`.

4. **In-Process Service Bus**: `microservices_client` uses Django's test `Client()` to POST to internal URL endpoints resolved via `django.urls.reverse()`. This is an in-process routing mechanism, not network-based microservice communication.

## Data Models

### `OAuthAbstractApplication` (Abstract Base)

Defined in `third_party_integrations/models.py:16`. Stores OAuth2 client registration data.

| Field | Type | Description |
|-------|------|-------------|
| `client_id` | CharField(191) | OAuth2 client identifier |
| `client_secret` | TextField | OAuth2 client secret |
| `provider` | CharField(30) | Provider identifier (e.g., `openid`, `openid_decathlon`) |
| `name` | CharField(40) | Human-readable application name |
| `access_token_method` | CharField(10) | HTTP method for token endpoint (GET/POST/PUT) — choices from `AccessTokenMethods` |
| `redirect_uri` | TextField | OAuth redirect URI |
| `scope` | TextField | OAuth scopes |
| `access_token_url` | TextField | Token endpoint URL |
| `authorize_url` | TextField | Authorization endpoint URL |
| `callback_url` | TextField | Callback URL |
| `user_info_url` | TextField | User info endpoint URL |
| `retailer_identifier` | CharField(100) | Links to store's `common_store_identifier` |
| `sandbox` | BooleanField | Whether this config applies to demo/sandbox stores |
| `extra_data` | JSONField | Provider-specific config (API keys, endpoint URLs, notification preferences) |
| `created_at` | DateTimeField | Auto-set on creation |
| `updated_at` | DateTimeField | Auto-updated |

### `OAuthApplication`

Concrete model extending `OAuthAbstractApplication` (`models.py:107`). Adds:
- `natural_key()` returning `(client_id, client_secret)`
- `get_provider_obj()` method that maps provider string to provider class via `provider_classes` registry from `providers/openid/provider.py`

### `OAuthAccount`

Defined in `models.py:143`. Links a MishiPay customer to an external OAuth provider account.

| Field | Type | Description |
|-------|------|-------------|
| `user` | ForeignKey → `Customer` | The MishiPay customer (via `OAUTH_USER_MODEL`, defaults to `dos.Customer`) |
| `uid` | CharField(191) | External user ID from the provider (e.g., Decathlon `id_person`) |
| `provider` | CharField(30) | Provider name |
| `sandbox` | BooleanField | Sandbox/demo flag |
| `last_login` | DateTimeField | Last authentication timestamp |
| `date_joined` | DateTimeField | Account creation timestamp |
| `extra_data` | JSONField | Provider-specific user data (profile info, loyalty, contact info, notification preferences) |

### `OAuthToken`

Defined in `models.py:207`. Stores access/refresh tokens for an OAuth account.

| Field | Type | Description |
|-------|------|-------------|
| `app` | ForeignKey → `OAuthApplication` | The OAuth application |
| `account` | ForeignKey → `OAuthAccount` | The linked OAuth account |
| `token` | TextField | Access token |
| `token_secret` | TextField | Refresh token |
| `expires_at` | DateTimeField | Token expiration time |
| `extra_data` | JSONField | Full token response (includes `token_claim_set` for Decathlon) |
| `created_at` | DateTimeField | Creation timestamp |
| `updated_at` | DateTimeField | Last update timestamp |

### Model Relationships

```
OAuthApplication (1) ──── (N) OAuthToken
                                    │
OAuthAccount (1) ──────── (N) OAuthToken
       │
       └── user → Customer (dos.Customer)
```

## API Endpoints

### Third-Party Integration URLs

Defined in `third_party_integrations/urls.py`. All prefixed with `/v1/`.

| URL Pattern | Name | View | Method | Description |
|-------------|------|------|--------|-------------|
| `decathlon/signup/` | `decathlon_signup` | `DecathlonSignup` | POST | Legacy Decathlon signup (direct API) |
| `decathlon/login/` | `decathlon_login` | `DecathlonLogin` | POST | Legacy Decathlon login/signup combo |
| `decathlon/update-user-info/` | `update_user_info_decathlon` | `UpdateDecathlonUserInfo` | POST | Update user info on Decathlon + local DB |
| `decathlon/check-user-exists/` | `check_user_exists_on_decathlon` | `CheckEmailExistsOnDecathlon` | GET | Check if email exists on Decathlon |
| `decathlon/forgot-password/` | `decathlon_forgot_password` | `DecathlonForgotPassword` | POST | Trigger password reset email via Decathlon |
| `customer-oauth-callback/` | `customer_oauth_callback` | `OAuthCallBackView` | POST | OAuth2 authorization code callback |
| `customer-oauth-logout/` | `customer_oauth_logout` | `OAuthLogOutView` | POST | OAuth2 logout (revokes SSO session) |
| `customer-profile/` | `customer_profile` | `OAuthProfileUpdate` | GET/PATCH | Sync or update user profile via SSO |

### Request/Response Serializers

| Serializer | File | Purpose |
|-----------|------|---------|
| `ValidateOAuthDataSerializer` | `serializers.py:9` | Validates `code` (CharField), `error` (optional), `store_id` (UUID) for OAuth callbacks |
| `CustomerSerializer` | `serializers.py:15` | Serializes Customer with computed `access_token`, `show_notifications`, `profile_incomplete` fields |
| `OAuthLogoutSerializer` | `serializers.py:41` | Validates `token` and `store_id` for logout, checks store existence |
| `PrerequisiteSerializer` | `serializers.py:54` | Validates `store_id` (UUID) for profile operations |
| `UserModelSerializer` | `providers/openid/serializer.py:9` | Serializes Django User `first_name`, `last_name` |
| `CustomerModelSerializer` | `providers/openid/serializer.py:18` | Extends UserModel with `phone_number` validation (digits only) |
| `DecathlonPostDataSerializer` | `providers/openid/serializer.py:31` | Validates `gender` (male/female) and `notification_preference` (0/1) |

## Business Logic

### OAuth2 Callback Flow (Generic)

Handled by `OAuthCallBackView` (`views/customer.py:50`) and `OAuthCallBackView` (`providers/openid/views.py:131`):

1. Client POSTs `code` + `store_id` to `/v1/customer-oauth-callback/`
2. View resolves `OAuthApplication` via `store_obj.common_store_identifier` + `sandbox` flag
3. Provider object dispatches to appropriate callback view class
4. `OAuth2Client.get_access_token(code=...)` exchanges auth code for access token at provider's token endpoint
5. `OAuth2Client.get_user_info()` retrieves user profile from provider's user info endpoint
6. `check_user_exists()` checks if `OAuthAccount` exists for this `uid` + `sandbox`
7. If existing user → `OAuthLoginView.complete_login()`: updates token, creates DRF auth `Token`
8. If new user → `OAuthSignupView.new_user()`: creates `User`, `Customer`, `OAuthAccount`, `OAuthToken`
9. Records analytics (`record_login_signup_analytics`, `add_login_signup_to_analytics`)
10. Returns serialized customer data with access token

### Decathlon OAuth2 Callback Flow

Handled by `DecathlonOAuthCallBackView` (`providers/openid/views.py:200`). Extends the generic flow:

1. Same auth code exchange via `get_access_token()`
2. **JWT Introspection**: Decodes access token (without signature verification) to extract `personid` from claims
3. Calls `get_user_info()` — fetches from Decathlon identity API with `id_person`, `ppays` (region), `x-api-key`
4. Calls `get_user_contact_info()` — fetches contact details (email, phone) from separate Decathlon endpoint
5. Merges user info + contact info
6. Checks local existence, creates or updates user
7. For new users: creates `ThirdPartyLogin` entry with loyalty number, creates `RetailerSpecificCustomerProfile`
8. Marks `profile_incomplete` if name is default "NAME"/"SURNAME"

### Decathlon Profile Sync (`sync_user_info_with_sso`)

Defined in `providers/openid/decathlon.py:245`. Called from `OAuthProfileUpdate.get()`:

1. Refreshes access token via refresh token exchange
2. Fetches latest user info, contact info, and notification preferences from Decathlon APIs
3. Updates local models in an atomic transaction:
   - Django `User` (first_name, last_name)
   - `Customer` (first_name, last_name, phone_number)
   - `ThirdPartyLogin` (access_token, refresh_token)
   - `OAuthAccount` (extra_data with full profile, profile_incomplete flag)
   - `OAuthToken` (token, token_secret, expires_at)
   - `RetailerSpecificCustomerProfile` (phone_number, gender, extra_data, marketing opt-in)
4. On failure: creates support ticket via `create_support_ticket()`

### Decathlon Profile Update (`update_user_info`)

Defined in `providers/openid/decathlon.py:445`. Called from `OAuthProfileUpdate.patch()`:

1. Syncs with SSO first (calls `sync_user_info_with_sso`)
2. Pushes changes to Decathlon API:
   - Profile info update (name, surname, gender) via `update_profile_info_api`
   - Contact update (phone number) via `update_profile_contact_api` with regional phone validation
3. Updates local models atomically (User, Customer, OAuthAccount, RetailerSpecificCustomerProfile)
4. Optionally updates notification preferences asynchronously via `async_request.send()`

### Legacy Decathlon Login/Signup (`views/decathlon.py`)

`DecathlonLogin.post()` (`views/decathlon.py:648`):

1. Gets client access token via `get_access_token()` (Basic Auth with bearer token)
2. Checks if `ThirdPartyLogin` exists for this email + `DecathlonStoreType`
3. If no ThirdPartyLogin → calls `signup_user()`:
   - If user doesn't exist on Decathlon → registers via `register_user_on_decathlon()` (POST to registration API)
   - If user exists → logs in via `login_user_on_decathlon()` (POST to login API)
   - Creates local `User` + `Customer` + `ThirdPartyLogin`
4. If ThirdPartyLogin exists → calls `login_user()` → `login_user_on_decathlon()`
5. Returns customer data with DRF auth token, login_or_signup indicator

### Decathlon Error Handling

Error codes from Decathlon API are mapped to user-friendly messages in `constants.py`:

| Code | Message |
|------|---------|
| 5002 | Invalid password |
| 5004 | Invalid phone number |
| 5005 | Account blocked |
| 5006 | Account temporarily blocked |
| 5007 | Password reset required |
| 5011 | Max login attempts reached |
| 5014 | Identification error |
| 5021, 5022 | Invalid password |
| 5023, 5024 | Token expired |
| 5207 | Account not found |

### OAuth Logout

`OAuthLogOutView.post()` (`views/customer.py:166`):

1. Extracts token from `Authorization` header, validates with `OAuthLogoutSerializer`
2. Looks up `OAuthApplication` and `OAuthToken` for the user
3. Refreshes access token (gets new one from refresh token)
4. Calls provider's logout API with `id_token_hint`
5. On success: sets token `expires_at` to now, deletes DRF `Token`

## Microservices Client

### Architecture

The `microservices_client` module (`mainServer/microservices_client/`) provides a client abstraction layer for internal inter-module communication. It uses Django's test `Client()` (not HTTP over the network) to POST to internal URL endpoints resolved via `django.urls.reverse()`.

### BaseClient (`core.py:5`)

```python
class BaseClient(object):
    def __init__(self, language="en"):
        self.c = Client()           # Django test Client
        self.language = language

    def prepare(self, data):
        data["mpay_lng"] = self.language  # Adds language header
        return data
```

All client subclasses follow the same pattern: accept a `data_from_exposed_api` dict, call `self.prepare()` to add the language header, then POST to a named URL via `self.c.post(reverse("url_name"), data=...)`.

### Client Classes

| Client | File | Methods | Service Domain |
|--------|------|---------|----------------|
| `AnalyticsRecordClient` | `client.py` | `record_event(event_type, params)` | Analytics/Mixpanel event recording |
| `ItemManagementClient` | `item_microservice_client.py` | 10 methods | Item scanning, basket CRUD, wishlist, rating, promotions |
| `PaymentManagementClient` | `payment_microservice_client.py` | 8 methods + 1 disabled | Payment methods, Adyen integration, sessions, verification |
| `OrderManagementClient` | `order_microservice_client.py` | 3 methods | Order creation, order detail retrieval |
| `CustomerManagementClient` | `customer_microservice_client.py` | 3 methods | Customer address, feedback |
| `LoginSignupClient` | `thirdparty_client.py` | 3 methods | Decathlon login/signup, email check, password reset |
| `InformationUpdateClient` | `thirdparty_client.py` | 1 method | Decathlon user info update |

### ItemManagementClient Methods

| Method | Target URL | Description |
|--------|-----------|-------------|
| `scan_item()` | `scan_an_item_as_service` | Scan a product barcode |
| `add_item_to_basket()` | `add_item_to_basket_as_service` | Add scanned item to basket |
| `bulk_update_hardcoded_basket_items()` | `bulk_update_hardcoded_basket_items_as_service` | Bulk update with JSON content type |
| `add_item_to_wishlist()` | `add_item_to_wishlist_as_service` | Add item to wishlist |
| `get_basket_details()` | `get_basket_details_as_service` | Get current basket contents |
| `get_wishlist_details()` | `get_wishlist_details_as_service` | Get wishlist contents |
| `remove_item_from_basket()` | `remove_item_from_basket_as_service` | Remove item from basket |
| `remove_item_from_wishlist()` | `remove_item_from_wishlist_as_service` | Remove item from wishlist |
| `rate_item()` | `rate_item_as_service` | Rate a purchased item |
| `apply_picwic_promotions()` | `apply_picwic_promotions` | Apply Picwic-specific promotions to basket |

### PaymentManagementClient Methods

| Method | Target URL | Description |
|--------|-----------|-------------|
| `get_payment_methods()` | `get_payment_methods_as_service` | List available payment methods |
| `adyen_payment_method()` | `get_adyent_payment_as_service` | Get Adyen payment method details |
| `adyen_hpp_generate_form_data()` | `adyen_hpp_generate_form_data_service` | Generate Adyen HPP form data |
| `create_payment_session()` | `create_payment_session_as_service` | Create payment session |
| `create_psp_session()` | `create_psp_session_as_service` | Create PSP session |
| `verify_payment()` | `verify_payment_as_service` | Verify payment result |
| `confirm_payment()` | `confirm_payment_as_service` | Confirm payment completion |
| `by_pass_payment()` | `by_pass_payment_as_service` | Bypass payment (for zero-amount or test) |
| `save_card()` | N/A | **Disabled** — raises `Exception("PaymentManagementClient.save_card disabled")` |

## gRPC Tax Service

### Overview

Defined via protobuf in `tax.proto` (proto3 syntax). Generated stubs in `mainServer/tax_pb2.py` and `mainServer/tax_pb2_grpc.py`. Provides tax evaluation and tax table management for the platform.

### Service Definition (`proto.Tax`)

| RPC Method | Request | Response | Description |
|-----------|---------|----------|-------------|
| `EvaluateTax` | `TaxEvaluateRequest` | `TaxEvaluateResponse` | Calculate taxes for a basket of items |
| `GetStoreTaxTable` | `GetStoreTaxTableRequest` | `StoreTaxTable` | Retrieve tax table for a store |
| `ImportStoreTaxCodeData` | `ImportStoreTaxDataRequest` | `Empty` | Import/update tax codes for a store |
| `DeleteStoreTaxTable` | `DeleteStoreTaxTableRequest` | `Empty` | Delete all tax data for a store |

### Protobuf Messages

**`TaxEvaluateRequest`**:
- `store_data` (`StoreData`) — store_id, tax_included_in_price (bool), tax_calculator (enum: DYNAMIC)
- `commit` (bool) — whether to commit the tax calculation
- `items` (repeated `ItemData`) — each with item_id, total_amount, tax_code

**`TaxEvaluateResponse`**:
- `total_basket_amount` (string) — total basket amount
- `total_taxable_amount` (string) — taxable portion
- `total_tax` (string) — total tax amount
- `total_basket_amount_after_tax` (string) — grand total after tax
- `tax_breakup` (repeated `TaxLevel`) — breakdown by tax level (retailer_tax_level_id, tax_level, taxable_amount, tax_amount)
- `item_tax` (repeated `ItemTax`) — per-item tax (item_id, total_amount, taxable_amount, tax_percent, tax_amount, retailer_tax_level_id, tax_code)

**`TaxTable`**: tax_code, tax_level, tax_percent, retailer_tax_level_id

**`StoreLocation`** (defined but not referenced in service RPCs): city, country, line1, postal_code, region

**`ImportStoreTaxDataRequest`**: store_id, tax_table (repeated TaxTable), delete_tax_codes (repeated string — codes to remove)

### gRPC Client (`TaxStub`)

Located in `tax_pb2_grpc.py:9`. Standard gRPC stub with unary-unary methods for all 4 RPCs. Requires a `grpc.Channel` for initialization.

## Dependencies

### External Dependencies

| Dependency | Used By | Purpose |
|-----------|---------|---------|
| Decathlon Identity API | `third_party_integrations` | User registration, login, profile management |
| Decathlon Loyalty API | `third_party_integrations` | Loyalty number retrieval |
| Decathlon Notification API | `providers/openid/decathlon.py` | Notification preference management |
| gRPC (protobuf) | `tax_pb2.py`, `tax_pb2_grpc.py` | Tax evaluation service |

### Internal Dependencies

| Module | Depends On | Purpose |
|--------|-----------|---------|
| `third_party_integrations` | `dos.models.Customer` | User model |
| `third_party_integrations` | `dos.models.Store` | Store/region lookups |
| `third_party_integrations` | `dos.models.customer.ThirdPartyLogin` | Legacy third-party auth records |
| `third_party_integrations` | `mishipay_core.common_functions` | Response structures, language-based messages |
| `third_party_integrations` | `mishipay_core.utilities.oauth_utility` | `get_application()` helper |
| `third_party_integrations` | `mishipay_retail_customers.models.RetailerSpecificCustomerProfile` | Retailer-specific customer data |
| `third_party_integrations` | `mishipay_retail_jobs.tasks.async_request` | Async HTTP requests (notification preference updates) |
| `third_party_integrations` | `mishipay_core.support_ticket` | Support ticket creation on failures |
| `third_party_integrations` | `dos.analytics.customer_analytics` | Login/signup analytics recording |
| `third_party_integrations` | `mishipay_dashboard.util` | Dashboard analytics |
| `third_party_integrations` | `mishipay_retail_customers.app_settings.SSO_PHONE_VALIDATION` | Per-region phone number regex patterns |
| `microservices_client` | `django.urls.reverse` | Internal URL resolution |
| `microservices_client` | `dos.util.default` | JSON serialization helper |

See [Customers](./customers.md) for details on `Customer`, `ThirdPartyLogin`, and `RetailerSpecificCustomerProfile` models.
See [Payments](./payments.md) for details on payment processing endpoints called by `PaymentManagementClient`.
See [Analytics](./analytics.md) for details on the analytics system used by `AnalyticsRecordClient`.
See [Items](./items.md) for details on item scanning and basket operations called by `ItemManagementClient`.

## Configuration

### OAuth Settings (`app_settings.py`)

All settings are read from Django settings with the `SOCIALACCOUNT_` prefix:

| Setting | Default | Description |
|---------|---------|-------------|
| `SOCIALACCOUNT_AUTO_SIGNUP` | `True` | Bypass signup form using provider data |
| `SOCIALACCOUNT_PROVIDERS` | `{}` | Provider-specific config dict |
| `SOCIALACCOUNT_STORE_TOKENS` | `True` | Whether to persist tokens |
| `OAUTH_USER_MODEL` | `dos.Customer` | Configurable user model reference |
| `OAUTH_ACCOUNT_MODEL` | `third_party_integrations.OAuthAccount` | Configurable account model |
| `OAUTH_APPLICATION_MODEL` | `third_party_integrations.OAuthApplication` | Configurable app model |
| `OAUTH_TOKEN_MODEL` | `third_party_integrations.OAuthToken` | Configurable token model |

### Decathlon Configuration

Per-store Decathlon config is loaded from `settings.THIRD_PARTY_INTEGRATION['decathlon']` (used by the legacy path). Config is separated by `live`/`demo` status and `region` (e.g., `NL`, `IL`). Each regional config contains:

- `client_id`, `client_secret` — OAuth2 credentials
- `x-api-key` — Decathlon API key
- `bearer_token_access_token` — Base64-encoded Basic auth token for client credentials flow
- `get_access_token_url` — Token endpoint
- `login_api` — Login endpoint (formatted with `client_id`)
- `get_registration_user_url` — Registration endpoint
- `get_user_email` — Email lookup endpoint
- `get_extra_user_information` — User info endpoint
- `get_loyalty_no_url` — Loyalty number endpoint
- `get_forgot_password_url` — Password reset endpoint
- `update_details_url` — Profile update endpoint
- `country_code`, `language_code` — Regional settings
- `is_phone_no_required` — Whether phone is required for registration
- `default_num_third_usual`, `default_type_third_usual` — Store association defaults

The OAuth2-based path uses `OAuthApplication.extra_data` JSON field to store similar config:
- `x-api-key`, `update_profile_info_method/api`, `update_profile_contact_api/method`
- `user_notification_preference_keys`, notification preference APIs
- `user_contact_info_url`, `logout_api`

### Gender & Notification Maps

```python
DECATHLON_GENDER_MAP = {1: "male", 2: "female"}
DECATHLON_GENDER_MAP_REVERSED = {"male": 1, "female": 2}
DECATHLON_NOTIFICATION_PREFERENCE_MAP = {1: True, 0: False}
```

### Auth Provider Registry

```python
AuthProviders.GOOGLE = 'google'
AuthProviders.FACEBOOK = 'facebook'
AuthProviders.RED_LOGIN = 'red_login'
```

### Loyalty Discount Map

```python
RedDiscountMap = {'Bronze': 5, 'Silver': 5, 'Gold': 7, 'Platinum': 10}
```

## Notable Patterns

### Dual Decathlon Integration Paths

The codebase contains two parallel implementations for Decathlon auth:

1. **Legacy** (`views/decathlon.py`, 850 lines): Uses `settings.THIRD_PARTY_INTEGRATION` config, direct API calls, stores auth in `ThirdPartyLogin`. The `DecathlonSignup` view references undefined functions `get_access_token_from_code` and `get_email` (marked `# noqa: F821`), suggesting this code path may be partially deprecated.

2. **OAuth2** (`providers/openid/`): Uses `OAuthApplication` model, OAuth2 authorization code flow with PKCE-capable client, JWT token introspection, `ThirdPartyLogin` + `OAuthAccount` + `OAuthToken` storage. More complete and actively maintained.

### In-Process Microservices Client

The `microservices_client` pattern is unusual — it uses Django's test `Client()` for in-process HTTP routing rather than network calls. This means:
- No network overhead
- No service discovery needed
- Requests go through Django's full middleware stack
- URL resolution via `reverse()` ensures endpoint coupling
- The `prepare()` method adds `mpay_lng` (language) to all requests

### Explicitly Disabled Features

`PaymentManagementClient.save_card()` raises an exception with "PaymentManagementClient.save_card disabled", indicating card saving was intentionally removed from the service bus.

### Missing Phone Validation Pattern

The `decathlon.py` provider uses `mishipay_retail_customers.app_settings.SSO_PHONE_VALIDATION` with a store-type + region key (`SSO_PHONE_VALIDATION['decathlonstoretype'][store_object.region]`) for regex-based phone validation per region.

### Atomic Transactions

Both the legacy and OAuth2 Decathlon paths wrap multi-model updates in `transaction.atomic()` blocks. On failure in the OAuth2 path, support tickets are created via `create_support_ticket()`.

### JWT Without Signature Verification

`DecathlonOAuthCallBackView.callback()` (`providers/openid/views.py:229`) decodes JWT access tokens with `options={"verify_signature": False}`. This is by design — the token is used for claim extraction (person ID), not for security validation, since the token was just received directly from Decathlon's token endpoint.

## Test Coverage

`tests/test_api.py` contains `TestDecathlonAuthApi` with 2 test methods:
- `test_signup`: Creates a Decathlon store (NL region, demo=True), posts signup data, verifies 200 response and lowercase email
- `test_login`: Runs signup first, then login, verifies same assertions

Both tests are gated by `allow_network_tests()` — skipped unless network tests are explicitly enabled. Test data uses timestamp-based unique emails.

### Schema Evolution

10 migration files spanning 2019–2022:
- `0001_initial` (2019): Initial OAuthApplication, OAuthAccount, OAuthToken tables
- `0002`–`0005` (Sep 2019): Rapid field adjustments during initial development
- `0006`–`0007` (Aug/Sep 2020): Significant schema changes (89-line migration)
- `0008` (Jun 2022): Field updates
- `0009` (latest): Alters `extra_data` field type (likely `TextField` → `JSONField`)

## Open Questions

1. **Legacy vs OAuth2 path**: The `DecathlonSignup` class in `views/decathlon.py:577` references undefined functions (`get_access_token_from_code`, `get_email`). Is this code path still used, or has it been fully superseded by the OAuth2 flow?

2. **Google/Facebook providers**: `AuthProviders` defines Google and Facebook, but no provider implementations exist beyond OpenID/Decathlon. Are these planned or deprecated?

3. **Red Login provider**: `AuthProviders.RED_LOGIN` is defined but no corresponding provider class exists. What is the Red Login provider?

4. **Tax gRPC service location**: `tax_pb2.py` and `tax_pb2_grpc.py` are generated stubs, but the actual gRPC server implementation is not in the explored source paths. Where is the Tax service implemented?

5. **StoreLocation message**: The `StoreLocation` protobuf message is defined but not used by any RPC method. Is it used elsewhere or planned for future use?

6. **microservices_client URL targets**: The URL names used by the microservices client (e.g., `scan_an_item_as_service`, `create_order_as_service`) suggest separate "as_service" endpoints. Are these distinct from the customer-facing API endpoints, or do they share implementations?

7. **Israel discount logic**: `ThirdPartyLogin.extra_data['israel_discount_applicable']` has special handling in `login_user()` — it's toggled based on unclear boolean logic. What is the business rule for Israel-specific discounts?
