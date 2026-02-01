# Retail Payments Module

> Last updated by: Task T19 — Views, payment processing logic, gateway integrations, webhook handlers

## Overview

The `mishipay_retail_payments` module (259 files) handles all payment processing for the MishiPay platform. It supports **two payment architectures**: a legacy direct-PSP integration system (22 PSP types, 21 provider implementations) and a newer **MS Pay** microservice-based system (35+ payment gateways). The module manages payment variant configuration per store, transaction lifecycle, saved cards, payout scheduling, and payment analytics.

This is the largest module by file count after `mishipay_items` and `mishipay_retail_orders`, reflecting the complexity of multi-PSP, multi-country payment processing.

## Key Concepts

| Concept | Description |
|---------|-------------|
| **PaymentVariant** | Per-store PSP configuration defining which payment provider a store uses, including API keys, endpoints, and supported payment methods |
| **PSP (Payment Service Provider)** | Third-party payment processor (Adyen, Stripe, Braintree, etc.) |
| **MS Pay** | MishiPay's internal payment microservice — the newer architecture replacing direct PSP integrations |
| **Payment Session** | A transient payment attempt — created before the customer pays, updated as the payment progresses |
| **3DS (3D Secure)** | Card authentication protocol requiring additional verification (redirects, OTPs) |
| **Preauth → Capture** | Two-step payment flow: authorize the amount first, then capture it upon order completion |
| **Payout** | Transfer of collected funds from MishiPay's platform account to the retailer's bank account |
| **Bypass Payment** | Zero-amount orders that skip payment processing entirely (e.g., 100% discounted items) |

## Architecture

### Dual Payment System

The module operates two parallel payment architectures:

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT REQUEST                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  LEGACY PATH (v1/)                MS PAY PATH (v1/ms-pay/)  │
│  ┌──────────────────┐             ┌──────────────────────┐  │
│  │ CreatePayment     │             │ MsPayCreatePayment   │  │
│  │ Session            │             │ Session               │  │
│  │ VerifyPayment     │             │ MsPayAddPayment      │  │
│  │ ConfirmPayment    │             │ MsPayPaymentDetails  │  │
│  └────────┬─────────┘             └──────────┬───────────┘  │
│           │                                    │              │
│  ┌────────▼─────────┐             ┌──────────▼───────────┐  │
│  │ provider_factory()│             │ ms_pay.py (101K)     │  │
│  │ → 21 PSP providers│             │ → HTTP to MS Pay svc │  │
│  └────────┬─────────┘             └──────────┬───────────┘  │
│           │                                    │              │
│  ┌────────▼─────────┐             ┌──────────▼───────────┐  │
│  │ Adyen, Stripe,   │             │ MS Pay Microservice  │  │
│  │ Braintree, etc.  │             │ (35+ gateways)       │  │
│  └──────────────────┘             └──────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Legacy path**: Direct PSP integration via `provider_factory()` which dynamically loads provider classes from `providers/` directory. Each provider implements the `BasicProvider` interface (`core.py`).

**MS Pay path**: Delegates to the MS Pay microservice via HTTP. Handles 35+ payment gateways including terminal-based payments, wallets, and BNPL. Defined in `ms_pay.py` (101K) and `ms_pay_constants.py`.

### Module File Structure

```
mishipay_retail_payments/
├── __init__.py              ← Legacy enumerations (PaymentStatus, PSPType, CardBrandValues, etc.)
├── models.py                ← 8 Django models + 2 factory functions
├── serializers.py           ← 50+ serializer classes + dispatcher functions
├── views.py                 ← 92K — all view classes (documented in T19)
├── urls.py                  ← 59 URL patterns across 3 tiers
├── admin.py                 ← 7 admin registrations + 4 admin actions
├── api.py                   ← 20K — internal service API views
├── app_settings.py          ← 19K — serializer dispatch maps, constants, error messages
├── core.py                  ← BasicProvider abstract base class
├── ms_pay.py                ← 101K — MS Pay microservice client (T19)
├── ms_pay_constants.py      ← Modern payment method/gateway enums
├── ms_pay_serializers.py    ← MS Pay request/response serializers
├── ms_pay_helpers.py        ← MS Pay payload generation
├── ms_voucher.py            ← Voucher validation service client
├── payment_helper.py        ← Payment method summary, session lookup
├── permissions.py           ← 2 constraint-based permission classes
├── analytics.py             ← Payment analytics event recording
├── utils.py                 ← Bypass payment logic, Walmart pay helpers
├── providers/               ← 21 PSP provider implementations
│   ├── Adyen/               (4 files)
│   ├── AdyenDropin/         (2 files)
│   ├── Braintree/           (2 files)
│   ├── BrainTreePOS/        (4 files)
│   ├── ByPass/              (2 files)
│   ├── Mercanet/            (2 files)
│   ├── Nets/                (2 files)
│   ├── Ogone/               (5 files)
│   ├── OnePay/              (4 files)
│   ├── OnepayWidget/        (2 files)
│   ├── Optile/              (8 files — most complex)
│   ├── Payline/             (2 files)
│   ├── PayPal/              (2 files)
│   ├── PayPlus/             (3 files)
│   ├── QuickPay/            (2 files)
│   ├── SOGEcommerce/        (2 files)
│   ├── Spreedly/            (3 files)
│   ├── Stripe/              (2 files)
│   ├── StripeConnect/       (3 files)
│   ├── WorldPay/            (2 files)
│   └── Xpay/                (2 files)
├── libs/                    ← PSP client libraries
│   ├── onepay/              (6 files — HTTP client, services)
│   └── quickpay/            (9 files — client, DTOs, mappers, tests)
├── pubsub/                  ← Pub/Sub message handlers
├── reconciliation/          ← Payment reconciliation (Nets)
├── templates/               ← HTML forms for hosted payment pages
│   └── (adyen, mercanet, ogone, onepay, payline, payplus, quickpay, sogecommerce, worldpay)
├── management/              ← Management commands
├── migrations/              ← 119 migration files
└── tests/                   ← Test fixtures and mock data
```

## Data Models

### PaymentVariant

Per-store PSP configuration. Each store has one or more PaymentVariant records defining how payments are processed.

**Source**: `models.py:65-126`

| Field | Type | Description |
|-------|------|-------------|
| `store` | FK → `Store` | The store this variant belongs to (CASCADE) |
| `merchant_id` | CharField(255) | PSP merchant identifier |
| `checkout_endpoint` | URLField | PSP checkout URL |
| `get_payment_methods_endpoint` | URLField | PSP payment methods URL |
| `process_payment_endpoint` | URLField | PSP process payment URL |
| `success_redirect_url` | URLField | Redirect after successful payment |
| `failure_redirect_url` | URLField | Redirect after failed payment |
| `notify_redirect_url` | URLField | PSP notification callback URL |
| `status_url` | URLField (nullable) | Payment status check URL |
| `confirm_url` | URLField | Payment confirmation URL |
| `refund_url` | URLField | Refund processing URL |
| `payment_procedure` | CharField(20) | One of: `sdk`, `hosted_payment_page`, `cse`, `sdk+hosted_payment` |
| `token_or_key` | TextField | API key or token for PSP authentication |
| `shopperstatement` | CharField(50) | Text appearing on shopper's bank statement |
| `country` | CharField(2) | ISO country code |
| `currency_code` | CharField(3) | ISO currency code |
| `transaction_prefix` | CharField(10) | Prefix for transaction IDs (nullable) |
| `capture` | BooleanField | Whether to auto-capture (default: True) |
| `preauth` | BooleanField | Whether to use pre-authorization (default: False) |
| `refund` | BooleanField | Whether refunds are enabled (default: True) |
| `psp` | CharField(20) | PSP type — one of 22 `PSPType` values (default: `adyen`) |
| `payment_class` | CharField(255) | Fully-qualified Python class path for provider implementation |
| `extra_data` | JSONField | Additional PSP-specific configuration |
| `sandbox` | BooleanField | Whether using sandbox/test mode (default: True) |
| `via_optile` | BooleanField | Whether routed through Optile aggregator (default: False) |
| `default_payment_method` | CharField(32) | Default payment method (`credit_card` etc.) |
| `flow_for_3ds` | CharField(50) | 3DS flow: `verify_then_confirm` or `confirm` |
| `payment_method` | MultiSelectField | Supported payment methods (allows multiple) |
| `allow_3d_support` | BooleanField | Whether 3DS is enabled (default: False) |
| `optile_specific_params` | JSONField | Optile-specific config (username, password, URLs) |
| `secondary_psp_details` | JSONField | Secondary PSP config (merchant_id, endpoints, etc.) |
| `created` | DateTimeField | Auto-set on creation |
| `updated` | DateTimeField | Auto-set on modification |

**Key methods**:
- `secondary_psp_details_formatted()` — Pretty-prints secondary PSP JSON
- `extra_data_formatted()` — Pretty-prints extra_data JSON

### Payments

Represents a single payment transaction. **OneToOne relationship with Order** — each order has exactly one Payments record.

**Source**: `models.py:129-552`

| Field | Type | Description |
|-------|------|-------------|
| `variant` | FK → `PaymentVariant` | PSP config used (nullable — null for MS Pay) |
| `status` | CharField(30) | Transaction status (see PaymentStatus below) |
| `order` | OneToOneField → `Order` | The order being paid for (CASCADE) |
| `transaction_id` | CharField(255) | PSP transaction ID (nullable) |
| `psp_reference` | CharField(255) | PSP reference number |
| `payout_reference` | CharField(255) | Payout reference (nullable) |
| `payout_reference_for_refund` | CharField(255) | Refund payout reference (nullable) |
| `currency` | CharField(10) | Payment currency code |
| `payment_method` | CharField(32) | Payment method used (indexed) |
| `total` | Decimal(9,2) | Gross amount (default: 0.0) |
| `delivery` | Decimal(9,2) | Delivery charge (default: 0.0) |
| `tax` | Decimal(9,2) | Tax amount (default: 0.0) |
| `extra_data` | JSONField | PSP response history (nested: `payment_responses.{verify/confirm/capture/refund}_responses`) |
| `message` | TextField | Last status message |
| `token` | CharField(100) | Session/merchant token |
| `session_id` | CharField(100) | Payment session ID |
| `captured_amount` | Decimal(9,2) | Amount actually captured |
| `application_fee_amount` | Decimal(9,2) | Platform fee amount |
| `transaction_count` | IntegerField | Number of payment attempts |
| `save_card_for_transaction` | BooleanField | Whether to save card for future use |
| `paid_by_3ds` | BooleanField | Whether 3DS was used |
| `additional_payment_details` | JSONField | Extra details (saved_card_id, PSP API response history) |
| `refunded_amount` | Decimal(9,2) | Total refunded so far |
| `refund_transaction_count` | IntegerField | Number of refund attempts |
| `refund_processing_amount` | Decimal(9,2) | Refund amount currently processing |
| `demo_mode_transaction` | BooleanField (nullable) | Whether this is a demo transaction |
| `platform` | CharField(25) | Client platform (android, ios, web, kiosk, etc.) |
| `payment_started_at` | DateTimeField (nullable) | When payment was initiated |
| `payment_completed_at` | DateTimeField (nullable) | When payment completed |
| `payment_time_data` | JSONField (nullable) | Timing telemetry data |
| `ms_pay_payment_session_id` | CharField(24) | MS Pay session ID (indexed) |
| `payment_type` | CharField(8) | `original` or `return` |
| `created` | DateTimeField | Auto-set on creation |
| `updated` | DateTimeField | Auto-set on modification |

**Key methods** (business logic embedded in model):

| Method | Description |
|--------|-------------|
| `change_status(status, message)` | Updates status and saves |
| `preauth_for_amount(data, customer)` | Pre-authorizes via provider — sets `PREAUTH` or `REJECTED` |
| `capture_for_amount(response, amount)` | Captures pre-authorized amount, calls `order.complete_status()` on success |
| `refund(post_data)` | Processes refund via provider — tracks `refunded_amount` and `refund_processing_amount` separately |
| `payment_session(api_data)` | Creates payment session via provider, increments `transaction_count` |
| `confirm_payment(api_data)` | Confirms payment, handles dropin actions and saved card IDs |
| `optile_confirm_payment(api_data)` | Optile-specific confirmation flow |
| `verify_payment(api_data)` | Verifies payment result, auto-captures if not 3DS/Adyen preauth |
| `by_pass_payment(api_data)` | Bypass payment for zero-amount orders |
| `optile_verify_payment(api_data)` | Optile-specific verification |
| `update_status_as_per_input(eventCode, params)` | Webhook handler: maps PSP events to internal status |

**Refund logic** (`models.py:282-328`):
- Increments `refund_transaction_count` before processing
- MUJI special case: allows refund on already-refunded payments (`PaymentStatus.REFUNDED` added to allowed list)
- Validates `requested_refund_amount <= captured_amount - refunded_amount - refund_processing_amount`
- On HTTP 202: adds to `refund_processing_amount` (async refund)
- On HTTP 200: adds to `refunded_amount`, marks as `REFUNDED` when fully refunded

**Capture flow** (`models.py:241-280`):
- Skips provider capture call for PSPs that capture inline: XPay, OnePay, WorldPay, Braintree, PayPlus, SOGEcommerce, Mercanet, StripeConnect, Ogone, Payline
- For other PSPs: requires `PREAUTH` status, calls `provider.capture_amount()`
- On success: sets `CAPTURED`, calculates `captured_amount` from basket, calls `order.complete_status()`
- Stores all capture responses in `extra_data.payment_responses.capture_response[]`

### CustomerOptileProfile

Optile-specific customer account profile for saved payment methods.

**Source**: `models.py:555-567`

| Field | Type | Description |
|-------|------|-------------|
| `customer` | FK → `Customer` | Customer reference (CASCADE) |
| `account_id` | CharField(255) | Optile account ID |
| `registration` | BooleanField | Whether registered (default: True) |
| `recurrence` | BooleanField | Whether recurring (default: False) |
| `card_brand` | CharField(30) | Card brand from `CardBrandValues.CHOICES` |
| `masked_account` | JSONField | Masked account details |
| `extra_data` | JSONField | Additional Optile data |
| `created` / `updated` | DateTimeField | Timestamps |

**Note**: TODO comment at `models.py:556` suggests `customer` + `account_id` should have `unique_together` constraint but doesn't.

### SavedCard

Tokenized saved card for returning customers. Cards are stored per-customer per-payment-variant.

**Source**: `models.py:570-595`

| Field | Type | Description |
|-------|------|-------------|
| `customer` | FK → `Customer` | Card owner (CASCADE) |
| `saved_card_id` | UUIDField | Public identifier (unique, auto-generated, non-editable) |
| `token` | CharField(64) | PSP tokenization token |
| `last_4_digits` | CharField(4) | Last 4 digits of card number |
| `first_4_digits` | CharField(4) | First 4 digits (nullable) |
| `expiry` | DateField | Card expiry date |
| `card_brand` | CharField(30) | Brand from `CardBrandValues.CHOICES` |
| `is_3ds_card` | BooleanField | Whether the card requires 3DS |
| `extra_data` | JSONField | Additional card metadata |
| `payment_variant` | FK → `PaymentVariant` | Which PSP config this card is saved with (CASCADE) |
| `created` / `updated` / `last_used` | DateTimeField | Timestamps |

**Constraints**: `unique_together = ('token', 'payment_variant', 'customer')`

### PaymentProviderDetails

Stores supported card networks per PSP per region.

**Source**: `models.py:630-633`

| Field | Type | Description |
|-------|------|-------------|
| `payment_provider` | CharField(20) | PSP type from `PSPType.CHOICES` |
| `retailer_region` | CharField(20) | Region identifier |
| `supported_card_networks` | JSONField | Card networks supported in this region |

### PayoutScheduler

Configures scheduled payouts from MishiPay's platform account to retailer bank accounts.

**Source**: `models.py:640-717`

| Field | Type | Description |
|-------|------|-------------|
| `payout_mishipay_id` | UUIDField (PK) | Primary key |
| `name` | TextField | Payout schedule name (indexed) |
| `stores` | M2M → `Store` | Linked stores (through `PayoutSchedulerStoreMapping`) |
| `automatic` | BooleanField | Whether payouts run automatically (indexed) |
| `active` | BooleanField | Whether schedule is active (indexed) |
| `next_run` | DateField | Next scheduled payout date (nullable, indexed) |
| `last_run` | DateField | Last payout execution date (nullable) |
| `platform_account` | CharField(20) | Stripe platform: `"Mishipay Inc"` (US) or `"Mishipay"` (EU) |
| `payment_gateway` | CharField(32) | Gateway: `adyen`, `STRIPE_CONNECT`, or `VIVA_WALLET` |
| `store_type` | CharField(31) | Store type (choices set dynamically in `__init__`, indexed) |
| `account_id` | CharField(50) | Stripe connected account ID (nullable) |
| `stripe_api_key` | CharField(150) | Stripe API key for this account (nullable) |
| `statement_descriptor` | TextField | Payout statement descriptor (nullable) |
| `payout_type` | CharField(10) | `SINGLE` or `MULTIPLE` payouts per store |
| `payout_frequency` | CharField(10) | `DAILY`, `WEEKLY`, or `MONTHLY` |
| `currency` | CharField(4) | Currency code from 170+ supported currencies |
| `region` | CharField(2) | Region code (default: `"US"`) |
| `balance_transfer_config` | JSONField | Balance transfer configuration (Viva Wallet specific) |
| `created` / `updated` | DateTimeField | Timestamps |

**Validation** (`clean()` at `models.py:708-714`): Viva Wallet with balance transfer config cannot use `MULTIPLE` payout type.

**Key method**: `get_retailer_store_ids()` — Returns list of retailer store IDs for all linked stores.

### PayoutSchedulerStoreMapping

Through table for M2M between `PayoutScheduler` and `Store`.

**Source**: `models.py:720-724`

| Field | Type | Description |
|-------|------|-------------|
| `payout_scheduler` | FK → `PayoutScheduler` | Schedule reference (CASCADE) |
| `store` | OneToOneField → `Store` | Store reference (CASCADE, **unique** — each store belongs to one scheduler) |
| `created` / `updated` | DateTimeField | Timestamps |

### PayoutRun

Individual payout execution record.

**Source**: `models.py:727-735`

| Field | Type | Description |
|-------|------|-------------|
| `run_id` | UUIDField (PK) | Primary key |
| `amount` | Decimal(9,2) | Payout amount (nullable) |
| `payout_scheduler` | FK → `PayoutScheduler` | Parent schedule (CASCADE) |
| `retailer_store` | TextField | Retailer store identifier (nullable) |
| `status` | CharField(10) | `PENDING`, `SUCCEED`, `FAILED`, or `CANCELLED` |
| `response` | TextField | PSP response text |
| `created` / `updated` | DateTimeField | Timestamps |

## Enumerations & Constants

### PaymentStatus (Legacy + MS Pay)

Defined in `__init__.py:1-28`. 12 statuses representing the payment lifecycle:

| Status | Value | Description |
|--------|-------|-------------|
| `WAITING` | `"waiting"` | Initial state — payment created but not started |
| `PREAUTH` | `"preauth"` | Amount pre-authorized by PSP |
| `PSP_CAPTURE_WAITING` | `"psp_capture_waiting"` | Waiting for PSP to confirm capture |
| `CAPTURED` | `"captured"` | Payment successfully captured |
| `PARTIAL_CAPTURED` | `"partial_captured"` | Partial amount captured |
| `AUTHORIZED_AT_HOOK` | `"authorised_at_hook"` | Authorized via webhook callback |
| `REJECTED` | `"rejected"` | Payment rejected by PSP |
| `ERROR` | `"error"` | Processing error |
| `ABANDONED` | `"abandoned"` | Payment abandoned by customer |
| `PENDING` | `"pending"` | Awaiting async processing |
| `ACTION_REQUIRED` | `"action_required"` | Customer action needed (e.g., 3DS) |
| `REFUNDED` | `"refunded"` | Fully refunded |

### PSPType (Legacy — 22 types)

Defined in `__init__.py:45-98`. Used by `PaymentVariant.psp` field:

`adyen`, `adyen_dropin`, `judopay`, `xpay`, `onepay`, `onepay_widget`, `spreedly`, `stripe`, `braintree`, `bypass`, `worldpay`, `payplus`, `sogecommerce`, `nets`, `mercanet`, `braintreepos`, `stripeconnect`, `ogone`, `payline`, `paypal`, `quickpay`, `ms_pay`, `checkout`, `viva_wallet`, `verifone`

### ENUM_PAYMENT_GATEWAY (MS Pay — 35+ gateways)

Defined in `ms_pay_constants.py:144-183`. The modern gateway enumeration used by the MS Pay microservice:

`adyen`, `STRIPE_CONNECT`, `QUICK_PAY`, `MERCANET`, `PAYPLUS`, `BARCLAYCARD`, `ADYEN_TERMINAL`, `NORDEA_CONNECT`, `AURUS`, `BANK_OF_MALDIVES`, `CHECKOUT`, `RAZORPAY`, `CYBERSOURCE`, `VIVA_WALLET`, `MOYASAR`, `MASHREQ_BANK_TERMINAL`, `SUREPAY`, `NETWORK_INTERNATIONAL`, `VIVA_WALLET_TERMINAL`, `NETWORK_INTERNATIONAL_TERMINAL`, `CAPSTONE`, `VERIFONE`, `NEARPAY`, `VIVA_WALLET_TERMINAL_P2P`, `MAGNATI_TERMINAL`, `GEIDEA`, `EDFAPAY`, `EDFAPAY_TERMINAL`, `POWERTRANZ`, `TAMARA`, `WALMART_PAY`, `CASH`, `LOYALTY`, `BUDGET_CODE`, `VAT`, `NETWORK_INTERNATIONAL_PUSH_TO_PAY`, `PINELABS_TERMINAL`

### Payment Methods

**Legacy** (`__init__.py:221-248`) — 25 methods:
`credit_card`, `card_terminal`, `3d_secure_card`, `apple_pay`, `paypal`, `ideal`, `google_pay`, `saved_card`, `ali_pay`, `mobilepay`, `wechat_pay`, `swish`, `vipps`, `twint`, `amazon_pay`, `cash`, `netbanking`, `wallet`, `emi`, `upi`, `hosted`, `loyalty`, `budget_code`, `bucks`, `vat`, `walmartpay`

**MS Pay** (`ms_pay_constants.py:26-48`) — 19 methods:
`CARD`, `CARD_TERMINAL`, `APPLE_PAY`, `GOOGLE_PAY`, `VIPPS`, `ALI_PAY`, `WECHAT_PAY`, `SWISH`, `AMAZON_PAY`, `MOBILE_PAY`, `PAYPAL`, `TWINT`, `IDEAL`, `CASH`, `HOSTED`, `LOYALTY`, `BUDGET_CODE`, `BUCKS`, `VAT`, `WALMART_PAY`

Bidirectional mapping between the two systems is defined in `PAYMENT_METHOD_MAP` and `PAYMENT_METHOD_MAP_ENUM` (`ms_pay_constants.py:74-118`).

### Card Brand Mappings

Each PSP has its own card brand string format. Card brand mapping dictionaries translate between PSP-specific names and the internal `CardBrandValues`:

| PSP | Source | Example mapping |
|-----|--------|-----------------|
| Adyen | (status codes in `ADYEN_STATUS_CODE_MAP`) | 35 status codes |
| XPay | `XPAY_CARD_MAPPING` | `"visa"` → `"VISA"` |
| OnePay | `ONEPAY_CARD_MAPPING` | `"VISA"` → `"visa"` |
| Mercanet | `MERCANET_CARD_MAPPING` | `"MASTERCARD"` → `"master_card"` |
| WorldPay | `WORLDPAY_CARD_MAPPING` | `"ECMC"` → `"master_card"` |
| PayPlus | Domestic + International maps | `"1"` → `"isracard"` (domestic) |
| Optile | `OPTILE_CARD_MAPPING` | Standard mapping |
| Stripe Connect | `STRIPE_CONNECT_CARD_MAP` | `"discover"` → `"discover"` |
| Ogone | `OGONE_CARD_MAPPING` | `"American Express"` → `"amex"` |

### CardBrandValues (11 brands)

`master_card`, `visa`, `amex`, `maestro`, `isracard`, `diners`, `jcb`, `leumicard`, `discover`, `unionpay`, `private_label`

Includes `CARD_LENGTHS` dictionary mapping brand to card number length (all 16 except Amex at 15).

### Other Enumerations

| Enumeration | Source | Values |
|-------------|--------|--------|
| `PaymentProcedure` | `__init__.py:31-42` | `sdk`, `hosted_payment_page`, `cse`, `sdk+hosted_payment` |
| `Payment3dsFlow` | `__init__.py:214-218` | `verify_then_confirm`, `confirm` |
| `STRIPE_PLATFORM_ACCOUNT` | `__init__.py:479-490` | US = `"Mishipay Inc"` (2-day delay), EU = `"Mishipay"` (7-day delay) |
| `PAYOUT_TYPE` | `__init__.py:493-496` | `SINGLE`, `MULTIPLE` |
| `PAYOUT_RUN_STATUS` | `__init__.py:499-509` | `PENDING`, `SUCCEED`, `FAILED`, `CANCELLED` |
| `PAYOUT_FREQUENCY` | `__init__.py:512-517` | `DAILY` (1 day), `WEEKLY` (7 days), `MONTHLY` (30 days) |
| `PaymentType` | `__init__.py:520-527` | `original`, `return` |
| `PAYMENT_GATEWAY_CHOICES` | `__init__.py:530-539` | `adyen`, `STRIPE_CONNECT`, `VIVA_WALLET` (for PayoutScheduler) |
| `MS_PAY_API_VERSION_CHOICES` | `__init__.py:542-549` | `v1`, `v2` |
| `ENUM_PAYMENT_SESSION_STATUS` | `ms_pay_constants.py:4-11` | `CREATED`, `SUCCESS`, `FAILED`, `PENDING`, `CANCELLED`, `ACTION_REQUIRED` |
| `ENUM_SETUP_SESSION_STATUS` | `ms_pay_constants.py:14-22` | Same as payment session |
| `ENUM_REFUND_STATUS` | `ms_pay_constants.py:51-55` | `CREATED`, `PROCESSING`, `SUCCESS`, `FAILED` |
| `ENUM_NOTIFICATION_TYPE` | `ms_pay_constants.py:58-60` | `PAYMENT_SESSION`, `REFUND` |

### MS_PAY_CURRENCY_CODES

170+ ISO 4217 currency codes defined as Django choices in `__init__.py:304-476`. Used by `PayoutScheduler.currency`.

## Provider Factory Pattern

Two factory functions in `models.py` dynamically load PSP provider classes:

### provider_factory (**variant_details)

**Source**: `models.py:599-613`

1. Looks up `PaymentVariant` by provided criteria
2. Reads `payment_class` field (e.g., `"mishipay_retail_payments.providers.Adyen.provider_class.AdyenProvider"`)
3. Uses `__import__` + `getattr` to dynamically load the class
4. Instantiates with the `PaymentVariant` object

### provider_optile_factory (**variant_details)

**Source**: `models.py:617-627`

Hardcodes the Optile provider class path regardless of variant's `payment_class` field.

### BasicProvider (Abstract Base)

**Source**: `core.py:5-38`

Defines the provider interface:
- `save_psp_responses(payment_obj, psp_api_name, response)` — Static method that timestamps and appends PSP responses to `additional_payment_details.psp_api_responses.{api_name}[]`
- `refund(payment, amount)` — Abstract (raises `NotImplementedError`)
- `delete_token_from_psp(post_data)` — Abstract (raises `NotImplementedError`)

## Serializers

### Dispatcher Functions

The serializers file defines two primary dispatch functions that select the correct serializer based on channel (web/android/ios), PSP type, and payment method:

**`get_payment_verify_serializer_class(channel, psp_type, payment_method)`** — `serializers.py:17-189`
Returns the appropriate verify serializer. Supports ~13 PSP × method combinations per channel.

**`get_payment_confirm_serializer_class(channel, psp_type, payment_method)`** — `serializers.py:192-396`
Returns the appropriate confirm serializer. Supports ~18 PSP × method combinations per channel.

Both use a 3-level nested dictionary: `channel → psp_type → payment_method → SerializerClass`.

### Validation Functions

13 standalone validators in `serializers.py:399-488`:

| Function | Validates |
|----------|-----------|
| `validate_card_brand(card_brand)` | Card brand in `CardBrandValues.CHOICES` |
| `validate_store_id(store_id)` | Store exists in DB |
| `validate_customer_id(customer)` | Customer exists in DB |
| `validate_payment_method(payment_method)` | Method in `PAYMENT_METHOD_CHOICES` |
| `validate_custom_payment_method(payment_method)` | Must be `apple_pay` or `google_pay` |
| `validate_saved_card_payment_method(payment_method)` | Must be `saved_card` |
| `validate_channel(channel)` | Channel in `android`, `web`, `ios` |
| `validate_order_id(order_id)` | Order exists in DB |
| `validate_merchantsig(merchantsig)` | **No-op** — validation commented out with FIXME |
| `validate_country_code(country_code)` | Exactly 2 characters |
| `validate_sdk_url(returnUrl)` | URL starts with `app://` or `mishipay://` |
| `validate_psp_type(psp_type)` | PSP type in `PSPType.CHOICES` |
| `validate_custom_psp_type(psp_type)` | Must be `stripeconnect` |
| `validate_email_address(email)` | Uses `email_validator` library (supports non-ASCII) |

### Serializer Categories

**Model Serializers** (output):

| Serializer | Model | Purpose |
|------------|-------|---------|
| `PaymentSerializer` | `Payments` | Basic payment fields (variant, order, transaction_id, currency, total, message, payment_method) |
| `SavedCardsSerializer` | `SavedCard` | Card creation with brand/customer validation |
| `SavedCardReprSerializer` | `SavedCard` | Read-only representation with masked card number (`**** 1234`) |
| `GetPaymentMethodSerializer` | `PaymentVariant` | Exposes `payment_method` and `default_payment_method` |
| `StoreBasicSerializer` | `Store` | Store info with backward-compatible `retailer` field |

**Prerequisite/Session Serializers** (input — step 1 of payment flow):

| Serializer | Purpose |
|------------|---------|
| `PaymentSessionPrerequisiteSerializer` | Validates channel, psp_type, store_id, payment_method, order_id |
| `PaymentVerifyPrerequisiteSerializer` | Same as above |
| `PaymentConfirmPrerequisiteSerializer` | Channel, psp_type, store_id, payment_method |
| `SaveCardPrerequisiteSerializer` | Channel, psp_type, store_id |
| `PaymentSessionWebParamsSerializer` | Web session with token, SDK version, origin, returnUrl |
| `PaymentSessionAndroidParamsSerializer` | Android session with SDK return URL |
| `PaymentSessionIosParamsSerializer` | iOS session with SDK return URL |
| `PaymentSessionWithoutOrder` | Session creation without an order (store_id + customer only) |
| `PaymentSessionCheckStatusSerializer` | Check status with order_id, store_id, payment_session_id |

**PSP-Specific Session Serializers**:

| Serializer | PSP | Notes |
|------------|-----|-------|
| `AdyenPaymentSerializer` | Adyen | Basic: customer, store_id, order_id |
| `AdyenDropinSessionKeySerializer` | Adyen Dropin | Channel + psp_type + store_id + order_id |
| `AdyenApplePaySessionSerializer` | Adyen | merchantIdentifier, displayName, initiative, initiativeContext |
| `AdyenApplePaySessionWebSerializer` | Adyen | Adds validationUrl for web |
| `WorldPaySessionSerializer` | WorldPay | Adds optional device_id |
| `OnepayCreditCardSessionSerializer` | OnePay | With save_card_id validation |
| `OnepayIdealSessionSerializer` | OnePay iDEAL | redirect_url + bank_name (12 Dutch banks) |
| `OnepayWidgetSessionKeySerializer` | OnePay Widget | Channel + psp_type + store_id + order_id |
| `PayplusCreditCardSessionSerializer` | PayPlus | With redirect_url |
| `SogEcommerceCreditCardSessionSerializer` | SOGEcommerce | Basic session |
| `PaylineCreditCardSessionSerializer` | Payline | Basic session |
| `StripeConnectCreditCardSessionSerializer` | Stripe Connect | With save_card_id + return_url validation |
| `StripeConnectApplePaySessionSerializer` | Stripe Connect | Apple Pay-specific |
| `OgoneHPPSessionSerializer` | Ogone | HPP with redirect_url + save_card_id |
| `QuickPaySessionSerializer` | QuickPay | With continue_url + cancel_url |
| `SpreedlySDKSessionKeySerializer` | Spreedly | Basic session |
| `GooglePaySDKSessionKeySerializer` | Google Pay | Basic session |
| `XpayCardPaymentSessionSerializer` | XPay | order_id + vat_rate |
| `StripeSavedCardSerializer` | Stripe | store_id + customer |

**Verify Serializers** (step 2 — verify payment result):

| Serializer | PSP | Notes |
|------------|-----|-------|
| `WorldPayVerifySerializer` | WorldPay | encryptedAccount, network_code, save_card validation |
| `WorldPayVerifyInputSerializer` | WorldPay | encrypted_data |
| `MercanetVerifySerializer` | Mercanet | data_element + seal |
| `NetsVerifySerializer` | Nets | response_code only |
| `StripeConnectVerifySerializer` | Stripe Connect | stripe_payment_method_id (required for card/apple/google pay), return_url, DB query validation |
| `OnePayVerifySerializer` | OnePay | order_id + store_id + is_poll_call |
| `OnePayPaypalVerifySerializer` | OnePay PayPal | Adds nonce |
| `PayPlusVerifySerializer` | PayPlus | result_url |
| `OgoneHPPVerifySerializer` | Ogone | result_url |
| `SogEcommerceVerifySerializer` | SOGEcommerce | result_url |
| `XpayVerifySerializer` | XPay | With save_card_id validation |
| `BraintreeGooglePayVerifySerializer` | Braintree | nonce |
| `CashPaymentVerifySerializer` | Cash | order_id + store_id only |
| `BrainTreePayPalPOSVerifySerializer` | BrainTree POS | nonce or billing_agreement_id (regex validation) |
| `PaylineVerifySerializer` | Payline | order_id + store_id |

**Confirm Serializers** (step 3 — confirm payment):

| Serializer | PSP | Notes |
|------------|-----|-------|
| `AdyenSDKConfirmSerializer` | Adyen SDK | payload or save_card_id required |
| `AdyenDropinConfirmSerializer` | Adyen Dropin | token, additional_data, dropin_data, originUrl, shopperIP |
| `AdyenGooglePayConfirmSerializer` | Adyen | token |
| `AdyenApplePayConfirmSerializer` | Adyen/WorldPay/Mercanet | token |
| `AdyenSavedCardConfirmSerializer` | Adyen | save_card_id + payment_method (must be `saved_card`) |
| `StripeConfirmSerializer` | Stripe | token or save_card_id |
| `StripeConnectConfirmSerializer` | Stripe Connect | save_card_in_db flag |
| `StripeConnectWalletConfirmSerializer` | Stripe Connect | stripe_payment_method_id |
| `SpreedlySDKPaymentConfirmSerializer` | Spreedly | token or save_card_id |
| `SpreedlySavedCardConfirmSerializer` | Spreedly | save_card_id (required) |
| `XpayCardConfirmSerializer` | XPay | save_card_id |
| `XpayApplePayConfirmSerializer` | XPay | token |
| `OnepayWidgetConfirmSerializer` | OnePay Widget | optional token + save_card_id |
| `OnePayConfirmSerializer` | OnePay | Basic |
| `WorldPayOptileCreditCardConfirmSerializer` | WorldPay+Optile | optile_response_params JSON validation |
| `WorldPayConfirmInputSerializer` | WorldPay | save_card_id |
| `PayPlusConfirmSerializer` | PayPlus | save_card_id + card_verification_code |
| `QuickPayConfirmSerializer` | QuickPay | Basic: order_id, store_id, customer |
| `PayPalConfirmSerializer` | PayPal | paypal_authorized_payment_id |
| `ByPassConfirmSerializer` | Bypass | Validates store has `payment_platform == "bypass"` |

**Webhook/Notification Serializers**:

| Serializer | Purpose |
|------------|---------|
| `AdyenNotifySerializer` | Adyen webhook notification fields |
| `AdyenConfirmSerializer` | Adyen redirect confirmation with merchantSig |
| `PreAuthEncryptedDataSerializer` | Encrypted card data for preauth |

**Card Management Serializers**:

| Serializer | Purpose |
|------------|---------|
| `GetSavedCardsParamsSerializer` | Get cards: store_id + customer |
| `SaveCardParamsSerializer` | Save card: store_id + customer |
| `DeleteSavedCardSerializer` | Delete card with existence validation |
| `DeleteTokenFromPSPSerializer` | Delete token from PSP |

**Guest User Serializers**:

| Serializer | Purpose |
|------------|---------|
| `GuestUserSerializer` | First name, last name, email |
| `MishiPayGuestUserSerializer` | Email only |

**Custom Action Serializers**:

| Serializer | Purpose |
|------------|---------|
| `PaymentCustomActionSerializer` | Bypass payment validation — checks basket total is zero |
| `PaymentWalletSerializer` | Wallet payment — restricted to Apple Pay/Google Pay + StripeConnect |

### MS Pay Serializers

Defined in `ms_pay_serializers.py`, these handle the newer MS Pay microservice integration:

**Response Serializers**:

| Serializer | Purpose |
|------------|---------|
| `MishiPaymentSerializer` | Base response: status, action, _id, error_message, amount, payment_gateway, payment_method, context, channel |
| `MishiPaymentWithCardDataSerializer` | Extends above with card_network, card_number_last4 |
| `PartialPaymentMishiPaymentSerializer` | Extends above with nested `new_payment_session` for partial payments |
| `MishiSetupSerializer` | Setup session response |

**Request Serializers**:

| Serializer | Purpose |
|------------|---------|
| `MishiCreatePaymentSessionSerializer` | Create payment session — 20+ fields including payment_method, channel, variant IDs, geo-location, terminal, gift card, payer info |
| `MishiCreateSetupSessionSerializer` | Create setup session for card tokenization |
| `MishiGetAllowedPaymentMethodsSerializer` | Get allowed methods for a basket |
| `MishiGetRefundMethodsSerializer` | Get refund methods for a store |
| `FetchPaymentsSerializer` | Fetch payments by order IDs (list of UUIDs) |
| `MishiCreateReturnPaymentSessionSerializer` | Create return/refund payment session — kiosk-only (channel/platform validation) |
| `MishiGuestUserSerializer` | Guest user email for MS Pay |
| `InitiateWalmartPaySessionSerializer` | Walmart Pay session: order_id + is_associate flag |
| `GiftCardBalanceSerializer` | Check gift card balance: terminal_id, device_id, card number, store, variant |

**Key validation** in `MishiCreatePaymentSessionSerializer`:
- `CARD_TERMINAL` payment method restricted to kiosk device types only (`models.py:97-105`)

### Voucher Serializer

In `ms_voucher.py`:

| Serializer | Purpose |
|------------|---------|
| `VoucherValidateApplySerializer` | Validates promo code, basket_id, user_id, business_unit_id, business_unit_group_id |

`VoucherService.validate_apply_promo()` proxies to the external voucher service at `VOUCHER_VALIDATE_URL`.

## Serializer Dispatch Maps

Defined in `app_settings.py`, two large 3-level dictionaries map `(channel, psp_type, payment_method)` to serializer classes:

**`PaymentSessionMap`** (`app_settings.py:122-200+`) — Maps to session creation serializers for each PSP combination. Supports:
- 13 PSP types on web (adyen, adyen_dropin, onepay_widget, spreedly, worldpay, xpay, braintree, onepay, payplus, sogecommerce, mercanet, nets, stripeconnect, ogone, payline, paypal)
- Similar coverage on android and iOS with platform-specific differences

These maps determine which serializer validates the incoming request based on the client's payment context.

## API Endpoints

### URL Structure (59 routes)

**Source**: `urls.py`

#### Legacy Payment APIs (20 endpoints)

| Path | View | Method |
|------|------|--------|
| `v1/get-payment-methods/` | `GetPaymentMethods` | GET |
| `v1/adyen-payment/` | `AdyenPayment` | POST |
| `v1/get-origin-keys/` | `GetOriginKeys` | GET |
| `v1/adyen/generate-hpp-form-data/` | `AdyenHPPGenerateFormData` | POST |
| `v1/adyen/notify/` | `AdyenNotifyUrl` | POST |
| `v1/adyen/confirm-api/` | `AdyenConfirm` | POST |
| `v1/adyen/webhook` | `AdyenWebhookView` | POST |
| `v1/adyen/web-redirect` | `AdyenWebRedirect` | GET |
| `v1/create-payment-session/` | `CreatePaymentSession` | POST |
| `v1/create-psp-session/` | `CreatePaymentSessionWithoutOrder` | POST |
| `v1/verify-payment/` | `VerifyPayment` | POST |
| `v1/confirm-payment/` | `ConfirmPayment` | POST |
| `v1/custom-action/` | `PaymentCustomAction` | POST |
| `v1/get-or-save-card/` | `GetOrSaveCard` | GET/POST |
| `v1/delete-saved-card/` | `DeleteSavedCard` | POST |
| `v1/delete-token-from-psp/` | `DeleteTokenFromPSP` | POST |
| `v1/notify-payment/` | `XpayNotify` | POST |
| `v1/optile-notify/` | `OptileNotify` | POST |
| `v1/sog/notify/` | `SogEcommerceNotify` | POST |
| `v1/mercanet/notify/` | `MercanetNotify` | POST |
| `v1/optile-summary/` | `OptileSummary` | GET |
| `v1/payment-wallet-details/` | `PaymentWalletDetails` | POST |
| `v1/quickpay/web-redirect` | `QuickPayWebRedirect` | GET |

#### Internal Service APIs (9 endpoints)

| Path | View | Purpose |
|------|------|---------|
| `api/v1/get-payment-methods/` | `GetPaymentMethodsService` | Internal: fetch methods |
| `api/v1/adyen-payment/` | `AdyenPaymentService` | Internal: Adyen payment |
| `api/v1/adyen/generate-hpp-form-data/` | `AdyenHPPGenerateFormDataService` | Internal: HPP form |
| `api/v1/payment-session/` | `CreatePaymentSessionService` | Internal: create session |
| `api/v1/api-psp-session/` | `CreatePaymentSessionWithoutOrderService` | Internal: PSP session |
| `api/v1/api-confirm-payment/` | `ConfirmPaymentService` | Internal: confirm |
| `api/v1/by-pass/` | `ByPassPaymentService` | Internal: bypass |
| `api/v1/api-verify-payment/` | `VerifyPaymentService` | Internal: verify |
| `api/v1/payment_method_summary/` | `PaymentMethodSummary` | Internal: summary |

#### MS Pay APIs (24 endpoints)

| Path | View | Purpose |
|------|------|---------|
| `v1/ms-pay/payment-methods/` | `MsPayPaymentMethods` | Get allowed payment methods |
| `v1/ms-pay/payment-session/` | `MsPayCreatePaymentSession` | Create payment session |
| `v1/ms-pay/payment-session/check-status/` | `MsPayPaymentSessionCheckStatus` | Check session status |
| `v1/ms-pay/payment-session/{id}/add-payment/` | `MsPayAddPayment` | Add payment to session |
| `v1/ms-pay/payment-session/{id}/payment-details/` | `MsPayPaymentDetails` | Get payment details |
| `v1/ms-pay/payment-session/{id}/abort-payment/` | `MsPayAbortPayment` | Abort payment |
| `v1/ms-pay/setup-session/` | `MsPayCreateSetupSession` | Create card tokenization session |
| `v1/ms-pay/setup-session/{id}/add-card/` | `MsPayAddCardToSetup` | Add card to setup |
| `v1/ms-pay/setup-session/{id}/setup-details/` | `MsPaySetupDetails` | Get setup details |
| `v1/ms-pay/saved-cards/` | `MsPaySavedCards` | List/manage saved cards |
| `v1/ms-pay/webhook/` | `MsPayWebhook` | MS Pay webhook receiver |
| `v1/ms-pay/validate-client/` | `MsPayValidateClient` | Validate client |
| `v1/ms-pay/fetch-payment/` | `MsPayFetchPayment` | Fetch payment by IDs |
| `v1/ms-voucher/validate-and-apply/` | `MsVoucherValidateApply` | Voucher validation |
| `v1/ms-pay/kiosk/stripe/terminal/` | `MsPayKioskStripeToken` | Kiosk Stripe terminal token |
| `v1/ms-pay/refund-methods/` | `MsPayRefundMethods` | Get refund methods |
| `v1/ms-pay/return-payment-session/` | `MsPayCreateReturnPaymentSession` | Create refund session |
| `v1/ms-pay/return-payment-session/{id}/return-details/` | `MsPayReturnDetails` | Get refund details |
| `v1/ms-pay/return-payment-session/{id}/abort-return/` | `MsPayAbortReturn` | Abort refund |
| `v1/ms-pay/viva-wallet-terminal/device-status/` | `MsPayVivaWalletTerminalDeviceStatus` | Terminal status |
| `v1/ms-pay/void-payment/` | `MsPayVoidPayment` | Void payment |
| `v1/ms-pay/edfapay/payment-session/` | `MsPayEdfaPayCreatePaymentSessionView` | EdfaPay session |
| `v1/ms-pay/edfapay/payment-session/{id}/payment-details/` | `MsPayEdfaPayPaymentDetailsView` | EdfaPay details |
| `v1/ms-pay/walmartpay/session/initiate/` | `MsPayInitiateWalmartPaySession` | Walmart Pay session |
| `v1/ms-pay/gift-card-balance/` | `MsPayGiftCardBalance` | Gift card balance check |

## Permissions

Two custom permission classes using the `EntityActionConstraint` system (eval-based):

### PaymentPermission

**Source**: `permissions.py:12-46`

Checks `EntityActionChoice.PAY` constraints against the store. Requires both `store_id` and `order_id` in request params. Passes `locals()` to `check_constraints()` — which uses `eval()` on database-stored expressions.

### SaveCardPermission

**Source**: `permissions.py:49-76`

Checks `EntityActionChoice.SAVE_CARD` constraints. If constraint has `response_status`, raises `CustomAPIException` (returns specific HTTP status). Otherwise returns 403.

## Admin Configuration

7 model registrations with specialized admin features:

### PaymentsAdmin

**Source**: `admin.py:42-71`

- **List display**: id, customer email, store, order_id, o_id, status, created, updated
- **Search**: exact match on customer email, store_id, order_id; prefix on o_id
- **Filters**: created, updated dates
- **Optimizations**: `list_select_related` for order/basket; `prefetch_related` for customer/store; `CachedTablePaginator`
- **Custom form**: `PaymentsAdminForm` — sorts payment method choices alphabetically

### PaymentVariantAdmin

**Source**: `admin.py:74-97`

- **List display**: id, store_id, retailer, sandbox, store_type, payment_procedure, payment_class, timestamps
- **Search**: exact store_id, store name, exact store_type, exact payment_procedure
- **Read-only fields**: `secondary_psp_details_formatted`, `extra_data_formatted` (pretty JSON)
- **Optimizations**: `prefetch_related` for store

### SavedCardAdmin

**Source**: `admin.py:100-126`

- **List display**: id, customer email, last 4 digits, first 4 digits, saved_card_id, store_id, store_type, retailer, timestamps
- **Search**: customer email, store type, retailer, store_id, last 4, first 4
- **Optimizations**: `prefetch_related` for customer + payment_variant + store (deep traversal)

### PaymentProviderDetailsAdmin

**Source**: `admin.py:129-132`

Simple list with `payment_provider` and `retailer_region`.

### PayoutSchedulerAdmin

**Source**: `admin.py:167-343`

- **Inline**: `PayoutSchedulerStoreMappingInline` (TabularInline) — filtered to `active=True` stores only
- **Inline validation**: `PayoutSchedulerStoreMappingBaseInlineFormSet` enforces Viva Wallet can only have one store with balance transfer config
- **Delete disabled**: `has_delete_permission() → False`
- **4 admin actions**:

| Action | Description |
|--------|-------------|
| `get_stripe_payout_balance` | Calls MS Pay to get Stripe account payable balance |
| `calculate_payout_amount` | Calls MS Pay/Viva to calculate payout amount (frequency-aware date ranges) |
| `get_stripe_payout_settings` | Fetches Stripe payout settings |
| `set_stripe_payout_settings_manual` | Sets Stripe payout to manual mode |

All actions are single-selection only and Stripe Connect-specific (except `calculate_payout_amount` which also supports Viva Wallet).

### PayoutRunAdmin

**Source**: `admin.py:346-363`

- **List display**: run_id, payout_name, status, amount (with currency prefix), retailer_store, timestamps
- **Ordered by**: `-created` (newest first)
- **Optimizations**: `prefetch_related` for payout_scheduler

## Helper Functions

### payment_helper.py

| Function | Purpose |
|----------|---------|
| `get_payment_method_summary(req_data)` | Calculates total captured amount for a payment method on a specific device for a business date. Iterates all successful orders and filters by `retailer_device_id` + `device_uuid`. |
| `get_payment_session(payment_session_id, response)` | Finds a specific payment session by `_id` in a list of session responses |
| `process_serializer_errors(errors)` | Formats DRF serializer errors (supports both single-object and `many=True` error dicts) into a flat error list |

### ms_pay_helpers.py

**`generate_ms_pay_create_payment_session_payload()`** — Builds the full request payload for creating an MS Pay payment session. 30+ fields including:
- Business identifiers (store_id, retailer_id, basket_id, user_id)
- Amount conversion to cents (`amount * 100`)
- Auto-capture disabled (`auto_capture: False`)
- Context from basket (`purchase_type` or default `"SCAN_AND_GO"`)
- Card scan tokenization flag (web only when `store.supports_card_scan`)
- In-store payment detection (cash/budget_code/VAT for stores in `ms_pay_in_store_payment_storetypes` GlobalConfig)

### utils.py

| Function | Purpose |
|----------|---------|
| `can_by_pass_payment(basket, store)` | Returns `True` if basket total is `0.0` AND store type is in whitelist (14 store types including Londis, Spar, MUJI, Flying Tiger, etc.) |
| `prepare_walmart_pay_basket_response(order, store, is_associate)` | Prepares Walmart Pay basket by invoking `WalmartBasketProcedure.get_basket_details()` |

### analytics.py

**`record_payment_captured_analytics(request, response, order_id)`** — Records payment capture event to analytics service. Skips for 3DS and explicit skip_analytics flag. Creates context object, payment object, and sends to analytics microservice via `AnalyticsRecordClient`. Errors create support tickets.

## Dependencies

### This Module Depends On

| Module | Usage |
|--------|-------|
| `dos.models` | `Store`, `Customer`, `get_store_type_choices()` |
| `mishipay_retail_orders.models` | `Order`, `Basket` |
| `mishipay_retail_customers.models` | `Customer` |
| `mishipay_core` | `EntityActionChoice`, `common_functions`, `choices`, `exception`, `cached_admin` |
| `mishipay_config.models` | `GlobalConfig` (MS Pay in-store payment store types) |
| `mishipay_items.models` | `Basket` (for voucher service) |
| `mishipay_retail_analytics` | Analytics context, payment objects |
| `mishipay_alerts` | `date_util` (for payment summary) |
| `microservices_client` | Analytics record client |

### Depends On This Module

| Module | Usage |
|--------|-------|
| `mishipay_retail_orders` | Order completion triggers payment capture |
| `mishipay_dashboard` | Refund processing via payment models |
| `mishipay_alerts` | Settlement reconciliation reads payment data |

## Configuration

### Environment/Settings

| Setting | Purpose |
|---------|---------|
| `MS_PAY_PAYOUT_SETTINGS_BALANCE_URL` | MS Pay payout balance endpoint |
| `MS_PAY_CALCULATE_PAYOUT_URL` | MS Pay payout calculation endpoint |
| `MS_PAY_PAYOUT_SETTINGS_ACCOUNT_URL` | MS Pay payout account settings endpoint |
| `MS_PAY_PAYOUT_SETTINGS_SET_MANUAL_URL` | MS Pay set manual payout endpoint |
| `VIVA_TRANSFER_CALCULATE_URL` | Viva Wallet transfer calculation endpoint |
| `VOUCHER_VALIDATE_URL` | Voucher validation service endpoint |

### App Settings Constants (`app_settings.py`)

90+ error/success message constants defined as translatable strings (`gettext_lazy`). Key constants:

| Constant | Value |
|----------|-------|
| `CREDIT_CARD` | `"credit_card"` |
| `APPLE_PAY` | `"apple_pay"` |
| `GOOGLE_PAY` | `"google_pay"` |
| `SAVED_CARD` | `"saved_card"` |
| `DEFAULT_TRANSACTION_ID` | `"100000"` |
| `CustomPaymentActions.BY_PASS` | `"by_pass_payment"` |

## Notable Patterns

1. **Business logic in models**: The `Payments` model contains ~300 lines of business logic (preauth, capture, refund, verify, confirm). This is unusual for Django models and mixes data concerns with payment processing. The TODO comment at `models.py:598` acknowledges `provider_factory` should be moved.

2. **Dynamic class loading**: `provider_factory()` uses `__import__` and `getattr` to load provider classes from the `payment_class` string field. This provides flexibility but means provider class paths are stored in the database, making refactoring risky.

3. **Dual payment architecture**: The legacy direct-PSP system (22 PSPs, provider classes) and the MS Pay microservice (35+ gateways) coexist. The legacy system uses the `provider_factory` pattern while MS Pay uses HTTP calls to an external service via `ms_pay.py`.

4. **Triple-dispatched serializer selection**: Serializers are selected via 3-level nested dictionaries keyed by `(channel, psp_type, payment_method)`. This creates large, repetitive dispatch maps in both `serializers.py` and `app_settings.py`.

5. **eval()-based permission system**: Both `PaymentPermission` and `SaveCardPermission` use `EntityActionConstraint.check_constraints(locals())` which internally calls `eval()` on database-stored expressions. This is a security concern flagged in previous documentation.

6. **No-op validation**: `validate_merchantsig()` (`serializers.py:449-454`) does nothing — the actual validation is commented out with a FIXME note.

7. **MUJI special-casing**: Refund logic allows re-refunding already-refunded payments specifically for MUJI stores (`models.py:288-289`).

8. **Payout scheduler Stripe API key storage**: `PayoutScheduler.stripe_api_key` stores Stripe API keys directly in the database as plaintext CharField.

9. **119 migrations**: Extensive schema evolution over the module's lifetime, suggesting frequent model changes.

## Views — Legacy Payment Flow

**Source**: `views.py:1-1480` (92K total file)

The legacy views handle direct-PSP payment flows where the mainServer communicates with the PSP via internal `PaymentManagementClient` HTTP calls (server-to-server to the internal payment microservice, not the same as MS Pay).

### Common Pattern: Eager Customer Lock

Nearly every legacy view uses the same `GPP-1046` pattern to prevent concurrent payment mutations:

```python
@transaction.atomic
def post(self, request):
    try:
        Customer.objects.select_for_update().get(
            cus_id=request.user.customer.cus_id, is_active=True
        )
    except Customer.DoesNotExist:
        pass
```

This acquires a row-level lock on the customer record within a transaction, preventing race conditions when the same customer makes concurrent requests.

### GetPaymentMethods

**Source**: `views.py:133-168` | **Method**: GET | **Auth**: `IsAuthenticated`

Fetches available payment methods for a store. Validates `store_id` and `customer` via `GetPaymentMethodsSerializer`, then delegates to `PaymentManagementClient.get_payment_methods()`. Returns PSP-supported methods with localized messages.

### AdyenPayment

**Source**: `views.py:171-206` | **Method**: POST | **Auth**: `IsAuthenticated`

Adyen-specific payment initiation. Validates via `AdyenPaymentSerializer`, then delegates to `PaymentManagementClient.adyen_payment_method()`. Used for Adyen's pre-authorization flow.

### GetOriginKeys

**Source**: `views.py:209-260` | **Method**: GET | **Auth**: `IsAuthenticated`

Fetches Adyen origin keys (CSE encryption keys) for a store. Looks up the `PaymentVariant` with `payment_procedure=HOSTED_PAYMENT_PAGE` and returns its `extra_data` containing the origin keys.

### CreatePaymentSession

**Source**: `views.py:260-400` (approx) | **Method**: POST | **Auth**: `IsAuthenticated + PaymentPermission`

Core legacy payment session creation. Two-step serializer validation:
1. `PaymentSessionPrerequisiteSerializer` validates channel, psp_type, store_id, payment_method, order_id
2. `PaymentSessionMap[channel][psp_type][payment_method]` selects and runs the PSP-specific serializer

Delegates to `PaymentManagementClient.create_payment_session()`. For PSPs in `{ADYEN, ADYEN_DROPIN, SPREEDLY, XPAY, BRAINTREE, WORLDPAY, ONEPAY, ONEPAY_WIDGET, PAYPLUS, STRIPECONNECT, OGONE}`, uses the internal payment microservice. Records analytics via `record_create_order_analytics()`.

### VerifyPayment

**Source**: `views.py` (follows CreatePaymentSession) | **Method**: POST | **Auth**: `IsAuthenticated + PaymentPermission`

Verifies a payment after the client-side payment step. Uses two-step validation:
1. `PaymentVerifyPrerequisiteSerializer`
2. `get_payment_verify_serializer_class(channel, psp_type, payment_method)` — triple-dispatched serializer selection

Delegates to `PaymentManagementClient.verify_payment()`. Records analytics via `record_payment_captured_analytics()`.

### ConfirmPayment

**Source**: `views.py` | **Method**: POST | **Auth**: `IsAuthenticated + PaymentPermission`

Confirms a payment (step 3 of the 3-step legacy flow). Uses two-step validation:
1. `PaymentConfirmPrerequisiteSerializer`
2. `get_payment_confirm_serializer_class(channel, psp_type, payment_method)`

Delegates to `PaymentManagementClient.confirm_payment()`. Records analytics on successful confirmation.

### PaymentCustomAction (Bypass Payment)

**Source**: `views.py` | **Method**: POST | **Auth**: `IsAuthenticated`

Handles zero-amount orders. Validates via `PaymentCustomActionSerializer` (checks basket total is zero), then delegates to `PaymentManagementClient.by_pass_payment()`. Used when 100% discounts result in $0.00 orders.

### GetOrSaveCard

**Source**: `views.py` | **Methods**: GET, POST | **Auth**: `IsAuthenticated + SaveCardPermission`

- **GET**: Returns saved cards for a customer at a store. Validates via `GetSavedCardsParamsSerializer`, calls `get_saved_cards()`, serializes with `SavedCardReprSerializer`.
- **POST**: Saves a new card. Two-step validation with `SaveCardPrerequisiteSerializer` → `SaveCardSessionMap[channel][psp_type]`. Delegates to `PaymentManagementClient.get_or_save_card()`.

### CreatePaymentOrderWithSavedCard

**Source**: `views.py:1395-1480` | **Method**: POST | **Auth**: `IsAuthenticated`

Creates an order and immediately pays using a saved card. Delegates order creation to `create_saved_card_order()`, then creates a payment session via `PaymentManagementClient.create_payment_session()` for PSPs in `{ADYEN, ADYEN_DROPIN, SPREEDLY, XPAY, BRAINTREE, WORLDPAY, ONEPAY, ONEPAY_WIDGET, PAYPLUS, STRIPECONNECT, OGONE}`.

### DeleteSavedCard

**Source**: `views.py:1483-1535` | **Method**: POST | **Auth**: `IsAuthenticated`

Deletes a saved card. Uses `select_for_update()` for customer locking. If the card's variant uses Optile (`via_optile`), calls `provider_class.delete_saved_card()` first, then deletes the local `SavedCard` record.

### DeleteTokenFromPSP

**Source**: `views.py:1538-1589` | **Method**: POST | **Auth**: `IsAuthenticated`

Deletes a payment token directly from the PSP. Resolves the store's payment variant via `PaymentInterface`, then calls `provider_class.delete_token_from_psp()`.

### CreatePaymentSessionWithoutOrder

**Source**: `views.py:1592-1643` | **Method**: POST | **Auth**: `IsAuthenticated`

Creates a payment session without a prior order (used for card tokenization setup). Two-step validation, then delegates to `PaymentManagementClient.create_psp_session()`.

### PaymentWalletDetails

**Source**: `views.py:1646-1718` | **Method**: POST | **Auth**: `IsAuthenticated + PaymentPermission`

Returns the `publishable_api_key` from the store's `PaymentVariant.extra_data`. Used by clients to initialize wallet payment SDKs (Apple Pay, Google Pay).

## Views — Webhook Handlers

PSP-initiated callbacks (server-to-server). These views receive asynchronous payment notifications and update local payment state.

### XpayNotify

**Source**: `views.py:1721-1819` | **Method**: POST | **Auth**: None (unauthenticated)

Receives XPay card tokenization callbacks. Workflow:
1. Extracts `token` query param → splits into `verification_token` + `order_id`
2. Validates the order exists and verification token matches
3. Skips for guest users (no saved cards)
4. Parses payment results from JSON body
5. Maps XPay card brands (`"Visa Card"` → `"visa"`, `"Master Card"` → `"master_card"`)
6. Creates a `SavedCard` record if the card doesn't already exist

### OptileNotify

**Source**: `views.py:1822-1919` | **Method**: GET | **Auth**: None (unauthenticated)

Receives Optile (now Payoneer) callbacks. Handles three entity types:
- **`session` / `payment`**: Ignored (no action needed)
- **`account`**: Stores `optile_customer_id`, `optile_account_id`, and `card_brand` in `Payments.additional_payment_details`
- **`customer`**: Calls `update_optile_customer_details()` to register the customer profile

If the payment used a saved card or `save_card_for_transaction` is true, invokes `optile_provider.save_card()`.

### SogEcommerceNotify

**Source**: `views.py:1930-1970` | **Method**: POST | **Auth**: None (unauthenticated)

Receives SOGEcommerce (Société Générale) payment notifications. Extracts `vads_trans_date`, `vads_trans_id`, `vads_cust_id`, `vads_order_id` from POST data. Stores the raw payment details in `Payments.additional_payment_details`.

### MercanetNotify

**Source**: `views.py:1973-2046` | **Method**: POST | **Auth**: None (unauthenticated)

Receives Mercanet (BNP Paribas) payment callbacks. Unlike other webhooks, this handler also **verifies the payment** and **redirects the user**:

1. Parses the pipe-separated `Data` field and `Seal` for signature validation
2. Looks up the payment object by customer + order
3. Delegates to `PaymentManagementClient.verify_payment()`
4. Redirects the user to `client_redirect_url` with status query params:
   - `?status=success&code=1` (payment captured)
   - `?status=rejected&code=2` (payment rejected)
   - `?status=failure&code=0` (error)
5. Records payment analytics

### OptileSummary

**Source**: `views.py:1922-1927` | **Method**: GET | **Auth**: None

No-op callback. Logs the received params and returns a simple response.

## Views — Adyen-Specific Views

### AdyenNotifyUrl

**Source**: `views.py` (follows core views) | **Method**: POST | **Auth**: None

Receives Adyen webhook notifications. Validates via `AdyenNotifySerializer`. Maps Adyen event codes to internal payment statuses using `ADYEN_STATUS_CODE_MAP` (35 status mappings). Calls `payment_obj.update_status_as_per_input()` to update state.

### AdyenHPPGenerateFormData

**Source**: `views.py` | **Method**: POST | **Auth**: `IsAuthenticated`

Generates form data for Adyen's Hosted Payment Page. Delegates to `PaymentManagementClient.generate_hpp_form_data()`.

### AdyenConfirm

**Source**: `views.py` | **Method**: POST | **Auth**: `IsAuthenticated`

Handles Adyen redirect confirmations after HPP. Validates `AdyenConfirmSerializer`, delegates to `PaymentManagementClient.adyen_confirm()`.

### AdyenWebhookView

**Source**: `views.py` | **Method**: POST | **Auth**: None

Modern Adyen webhook handler (distinct from `AdyenNotifyUrl`). Processes structured webhook payloads.

### AdyenWebRedirect

**Source**: `views.py` | **Method**: GET | **Auth**: None

Handles Adyen web redirect flows (3DS returns). Renders an HTML template with redirect parameters.

### QuickPayWebRedirect

**Source**: `views.py` | **Method**: GET | **Auth**: None

Handles QuickPay web redirect flows. Similar to `AdyenWebRedirect`.

## Views — MS Pay Flow

**Source**: `views.py:2049-2399`

MS Pay views are thin wrappers around `PaymentService` (defined in `ms_pay.py`). They handle authentication, basic parameter extraction, and delegate all logic to the service class.

### MsPayPaymentMethods

**Source**: `views.py:2049-2064` | **Method**: GET | **Auth**: `IsAuthenticated`

Returns allowed payment methods for a basket. Requires `basket_id` query param. Delegates to `PaymentService.get_allowed_payment_methods()`.

### MsPayCreatePaymentSession

**Source**: `views.py:2067-2074` | **Method**: POST | **Auth**: `IsAuthenticated`

Creates an MS Pay payment session. Records `start_datetime` for payment timing telemetry. Delegates to `PaymentService.create_payment_session()`.

### MsPayAddPayment

**Source**: `views.py:2077-2086` | **Method**: POST | **Auth**: `IsAuthenticated`

Adds a payment to an existing session (sends card/wallet payment data). Takes `payment_session_id` path param. Delegates to `PaymentService.add_payment()`.

### MsPayPaymentSessionCheckStatus

**Source**: `views.py:2089-2128` | **Method**: GET | **Auth**: `IsAuthenticated`

Checks payment session status. Validates via `PaymentSessionCheckStatusSerializer` (requires `order_id`, `payment_session_id`). Calls `PaymentService.get_all_payment_session_details()` then filters to the specific session.

### MsPayPaymentDetails

**Source**: `views.py:2131-2143` | **Method**: POST | **Auth**: `IsAuthenticated | JwtTokenPermission`

Gets payment details (3DS completion, polling). Supports both standard auth and JWT token auth (for web redirect flows). Delegates to `PaymentService.payment_details()`.

### MsPayAbortPayment

**Source**: `views.py:2146-2158` | **Method**: POST | **Auth**: `IsAuthenticated`

Aborts a payment session. Delegates to `PaymentService.abort_payment()`. The service supports optional `cancel_order` flag that also cancels the order (blocked if already completed/verified).

### MsPayCreateSetupSession / MsPayAddCardToSetup / MsPaySetupDetails

**Source**: `views.py:2161-2186` | **Method**: POST | **Auth**: `IsAuthenticated`

Card tokenization flow (3 endpoints):
1. `create_setup_session()` — Creates a tokenization session
2. `add_card_to_setup()` — Sends card data for tokenization
3. `setup_details()` — Completes the tokenization flow

All delegate to `PaymentService` and forward to the MS Pay microservice's setup session endpoints.

### MsPayWebhook

**Source**: `views.py:2188-2196` | **Method**: POST | **Auth**: `AllowAny`

**Critical security note**: This endpoint accepts unauthenticated requests. HMAC validation happens inside `PaymentService.handle_webhook()`, not at the view level. The webhook handler routes to either `_handle_payment_notification()` or `_handle_refund_notification()` based on `notification_type`.

### MsPaySavedCards

**Source**: `views.py:2198-2211` | **Methods**: GET, POST | **Auth**: `IsAuthenticated`

- **GET**: Returns saved cards from MS Pay for the authenticated customer
- **POST**: Deletes a saved card token (sends `user_id` + card data to MS Pay)

### MsPayValidateClient

**Source**: `views.py:2214-2220` | **Method**: POST | **Auth**: `IsAuthenticated`

Validates Apple Pay merchant domain. Proxies the request to MS Pay's verify client endpoint.

### MsPayFetchPayment

**Source**: `views.py:2223-2241` | **Methods**: GET, POST | **Auth**: `AllowAny`

Fetches payment data by order ID. **Security**: Uses HMAC signature validation instead of user auth (for server-to-server calls).
- **GET**: Single payment by `order_id` + `hmac` query params
- **POST**: Batch fetch by list of `order_ids` + `hmac` in body

### MsVoucherValidateApply

**Source**: `views.py:2244-2259` | **Method**: GET | **Auth**: `AllowAny`

Validates and applies a voucher/promo code. Requires `basket_id` and `voucher_code` query params. Delegates to `VoucherService.validate_apply_promo()`.

### MsPayKioskStripeToken

**Source**: `views.py:2261-2273` | **Method**: POST | **Auth**: `AllowAny`

Creates a Stripe terminal connection token for kiosk devices. Requires `region` in body.

### MsPayRefundMethods

**Source**: `views.py:2276-2290` | **Method**: GET | **Auth**: `IsAuthenticated`

Returns available refund methods for a store/platform combination.

### MsPayCreateReturnPaymentSession / MsPayReturnDetails / MsPayAbortReturn

**Source**: `views.py:2293-2326` | **Method**: POST | **Auth**: `IsAuthenticated`

Return/refund payment flow (3 endpoints):
1. `create_return_payment_session()` — Initiates a return payment session
2. `return_details()` — Completes the return payment
3. `abort_return()` — Cancels the return

### MsPayVivaWalletTerminalDeviceStatus

**Source**: `views.py:2329-2345` | **Method**: GET | **Auth**: `AllowAny`

Checks Viva Wallet card terminal device connectivity status. Requires `store_id` and `terminal_id` query params.

### MsPayVoidPayment

**Source**: `views.py:2348-2354` | **Method**: POST | **Auth**: `IsAuthenticated`

Voids a partial payment on a kiosk. Delegates to `PaymentService.void_payment()`.

### MsPayEdfaPayCreatePaymentSessionView / MsPayEdfaPayPaymentDetailsView

**Source**: `views.py:2357-2372` | **Method**: POST | **Auth**: `IsAuthenticated`

EdfaPay-specific payment flow (Middle East PSP). Separate from the standard MS Pay session flow — uses dedicated endpoints on the MS Pay microservice.

### MsPayGiftCardBalance

**Source**: `views.py:2375-2381` | **Method**: POST | **Auth**: `IsAuthenticated`

Checks gift card balance. Delegates to `PaymentService.get_gift_card_balance()`.

### MsPayInitiateWalmartPaySession

**Source**: `views.py:2384-2390` | **Method**: POST | **Auth**: `AllowAny`

Initiates a Walmart Pay session. Uses HMAC auth (not user auth). Delegates to `PaymentService.initiate_walmart_pay_session()`.

### PaymentMethodSummary

**Source**: `views.py:2393-2399` | **Method**: POST | **Auth**: `IsAuthenticated`

Returns aggregated payment method totals for a device on a business date. Used by kiosk/cashier devices for end-of-day reconciliation.

## PaymentService — MS Pay Microservice Client

**Source**: `ms_pay.py:134-2463` (101K)

`PaymentService` is the primary service class mediating between the mainServer views and the MS Pay microservice. All methods communicate with MS Pay via HTTP (`requests` library).

### MS Pay URL Configuration

| Setting | Variable | Purpose |
|---------|----------|---------|
| `PAYMENT_METHODS_URL` | `settings.PAYMENT_METHODS_URL` | Get allowed payment methods |
| `PAYMENT_SESSION_URL` | `settings.PAYMENT_SESSION_URL` | Create/manage payment sessions |
| `SAVED_CARDS_URL` | `settings.SAVED_CARDS_URL` | Saved card management |
| `SETUP_SESSION_URL` | `settings.SETUP_SESSION_URL` | Card tokenization sessions |
| `REFUND_URL` | `settings.REFUND_URL` | Process refunds |
| `VERIFY_CLIENT_URL` | `settings.VERIFY_CLIENT_URL` | Apple Pay domain verification |
| `MS_PAY_KIOSK_STRIPE_TOKEN_URL` | `settings.MS_PAY_KIOSK_STRIPE_TOKEN_URL` | Kiosk Stripe tokens |
| `REFUND_METHODS_URL` | `settings.REFUND_METHODS_URL` | Available refund methods |
| `RETURN_SESSION_URL` | `settings.RETURN_SESSION_URL` | Return payment sessions |
| `VIVA_DEVICE_STATUS_URL` | `settings.VIVA_DEVICE_STATUS_URL` | Viva Wallet terminal status |
| `VOID_PAYMENT_URL` | `settings.VOID_PAYMENT_URL` | Void payments |
| `EDFAPAY_PAYMENT_SESSION_URL` | `settings.EDFAPAY_PAYMENT_SESSION_URL` | EdfaPay sessions |
| `GIFT_CARD_BALANCE_URL` | `settings.GIFT_CARD_BALANCE_URL` | Gift card balance |
| `MPAY_HMAC` | `settings.MPAY_HMAC` | Shared HMAC secret for webhook verification |

### Payment Session Lifecycle

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ create_payment   │     │ add_payment()    │     │ payment_details()│
│ _session()       │────>│ POST /add-payment│────>│ POST /payment-   │
│ POST /session/   │     │                  │     │ details/          │
└──────────────────┘     └──────────────────┘     └──────────────────┘
         │                        │                        │
         │                        │                        │
         ▼                        ▼                        ▼
  Creates Payments         Updates payment          3DS completion,
  record, sends            data to MS Pay,          terminal polling,
  basket/order data        card/wallet info         status finalization
```

#### create_payment_session()

**Source**: `ms_pay.py:196-525`

The most complex method (~330 lines). Handles:

1. **Input validation**: `MishiCreatePaymentSessionSerializer`
2. **Shipping address**: Saves `BuyerAddress` with source attribution (`APPLE_PAY`, `GPAY`, `MISHIPAY`)
3. **Order validation**: Verifies order ownership (`cus_id` match)
4. **Idempotency**: If payment already `CAPTURED` or `REFUNDED`, returns existing payment data
5. **Partial payment continuation**: Respects `PARTIAL_CAPTURED` status — doesn't override `card_terminal` method with `loyalty`
6. **Payment record creation**: Creates or updates `Payments` object with:
   - Platform, currency, variant, amount, timing data
   - Terminal ID, merchant ID (for kiosk/POS)
   - Loyalty/BUCKS/VAT payment variant IDs for split payments
7. **MS Pay platform check**: Validates store supports MS Pay for the request channel (web/android/ios/kiosk)
8. **Amount calculation**:
   - Gets basket total with sponsored amounts (POS-log excluded discounts)
   - Subtracts loyalty/BUCKS/VAT split amounts
   - Subtracts already-captured partial amounts
   - Allows amount override for MLSE store type
9. **JWT return URL**: For web channel, generates a signed JWT (`HS256`) appended to `return_url`
10. **Payload generation**: Delegates to `generate_ms_pay_create_payment_session_payload()`
11. **V2 API support**: Replaces URL version based on `store.ms_pay_api_version`
12. **Response handling**: `_handle_mishi_payment_response()` processes the response

#### add_payment()

**Source**: `ms_pay.py:527-595`

Adds payment data (card token, wallet data) to an existing session:
1. Looks up `Payments` by `ms_pay_payment_session_id`
2. Handles guest user email updates
3. Enables `tokenize_from_encrypted_values` for card-scan web stores
4. POSTs to `PAYMENT_SESSION_URL/{id}/add-payment/`

#### payment_details()

**Source**: `ms_pay.py:597-622`

Completes a payment after client-side actions (3DS, terminal tap). Supports V2 API for kiosk/cashdesk via `store_id` query param. POSTs to `PAYMENT_SESSION_URL/{id}/payment-details/`.

#### abort_payment()

**Source**: `ms_pay.py:624-672`

Aborts a payment session. If `cancel_order=True` in request body, also cancels the order (blocked for `COMPLETED`/`VERIFIED` orders).

### Response Handler: _handle_mishi_payment_response()

**Source**: `ms_pay.py:1312-1554`

Central response processing (~240 lines). Handles all MS Pay responses uniformly:

1. **Error forwarding**: Non-OK responses forwarded as-is
2. **Payment lookup**: Finds or validates `Payments` by `order_id`
3. **Session tracking**: Updates `ms_pay_payment_session_id` and `payment_gateway`
4. **Extra data processing**: Converts amounts from cents to decimal (handles `amount_paid`, `amount_to_return`, `loyalty_redeemed_amount`)
5. **Split payment tracking**: Stores split payment details in `extra_data.split_payments[]`
6. **Success handling**:
   - For Razorpay/Viva Wallet: Updates payment method, email, mobile
   - Stores MS Pay payment details per `payment_id` (for partial payment refunds)
   - **Partial payment detection**: For Verifone, Network International Terminal, Adyen Terminal, Loyalty, VAT, Viva Wallet Terminal P2P — checks if `final_amount + captured_amount < basket_amount`
   - If partial: calls `_handle_partial_payment()`, returns partial response
   - If complete: calls `_mark_payment_success()`
7. **Non-success handling**: Sets status to `ACTION_REQUIRED`, `PENDING`, or `ERROR`
8. **Timing telemetry**: Records elapsed time in `payment.payment_time_data`

### Payment Success: _mark_payment_success()

**Source**: `ms_pay.py:1744-1823`

Marks a payment as successfully captured within an atomic transaction:

1. **Idempotency check**: If already `CAPTURED`, updates `payment_id` and returns
2. Sets `psp_reference`, `status=CAPTURED`, `captured_amount`, `application_fee_amount`
3. Calculates `payment_eta` (time from `payment_started_at` to `payment_completed_at`)
4. Calls `order.complete_status(extra_steps=False)` — marks order as completed
5. **Post-payment processing** (two paths):
   - **Kafka path** (`store.supports_kafka_flow`): Publishes event via `PostPaymentSuccessProducerAdapter`. On failure, falls back to sync tasks + Slack alert
   - **Sync path**: Calls `payment_success_async_tasks()`:
     - `order.complete_status_extra()` — triggers extra completion steps (receipt, POS log, etc.)
     - `add_payment_to_analytics()` — records to analytics
     - `add_order_to_analytics()` — records order analytics
     - `add_mix_panel_event()` — sends Mixpanel event

### Partial Payment Logic

**Source**: `ms_pay.py:1462-1733`

Partial payments occur when a single order is paid across multiple transactions (e.g., first tap covers $20 of $50, second tap covers remaining $30).

#### _handle_partial_payment()

**Source**: `ms_pay.py:1702-1733`

Updates `Payments` record for a partial payment:
- Sets `status = PARTIAL_CAPTURED`
- Converts `payment_id` to a list (for tracking multiple MS Pay payment IDs)
- Increments `captured_amount` by the partial amount
- Deduplicates by checking if `payment_id` is already recorded

#### _handle_partial_payment_response()

**Source**: `ms_pay.py:1556-1700`

Generates the response for a partial payment and auto-creates the next payment session for remaining amount:

1. Calculates remaining amount: `order_amount - captured_amount`
2. **Auto-loyalty/BUCKS/VAT**: If remaining equals the loyalty/BUCKS/VAT split amount, auto-creates a new session for that payment method
3. **Auto-terminal**: For Adyen Terminal, auto-creates a new session for remaining card amount
4. Returns `PartialPaymentMishiPaymentSerializer` response containing both current and `new_payment_session` data

### Webhook Handler: handle_webhook()

**Source**: `ms_pay.py:724-736`

Entry point for MS Pay webhooks. Validates HMAC signature using `MPAY_HMAC` shared secret:

```python
sig = self._hmac_generator(MPAY_HMAC, str(json.dumps(data, sort_keys=True)))
if sig != hmac:
    return Response({"message": "hmac does not match sig"}, status=403)
```

Routes based on `notification_type`:
- `PAYMENT_SESSION` → `_handle_payment_notification()`
- `REFUND` → `_handle_refund_notification()`

#### _handle_payment_notification()

**Source**: `ms_pay.py:868-1077`

Processes asynchronous payment status updates from MS Pay:

1. Looks up payment by `order_id`
2. Updates `ms_pay_payment_session_id` and `payment_gateway`
3. Processes `extra_data` (amount conversions, PSP reference)
4. **On success**: Stores `ms_pay_payment_details` per payment_id, handles partial payments for Verifone/Network International/Adyen terminals, calls `_mark_payment_success()`
5. **On non-success**: Sets `PENDING` or `ERROR` status
6. **Riyadh Tameer special handling**: For `riyadhtameerstoretype` stores, calls `confirm_riyadh_tameer_payment()` to confirm payment with the mall's API

#### _handle_refund_notification()

**Source**: `ms_pay.py:738-807`

Processes asynchronous refund status updates. Uses `@transaction.atomic`:

1. Looks up `Payments` by `order_id` and optionally `RefundOrderNew` by `refund_order_id`
2. For partial payments (payment_id is a list): delegates to `_handle_partial_payment_refund_notification()`
3. **On success**:
   - Increments `refunded_amount`, decrements `refund_processing_amount`
   - Sets `REFUNDED` status when fully refunded
   - Updates `RefundOrderNew` record — sets POSLOG sequence, calls `post_successful_refund_processing()` (sends email for WHSmith stores)
4. **On failure**: Decrements `refund_processing_amount`, sets `PSP_FAILED` status on refund order

### Refund Processing

**Source**: `ms_pay.py:1079-1310`

Three refund methods handle different scenarios:

#### refund()

**Source**: `ms_pay.py:1079-1151`

Standard single-payment refund:
1. Validates `status == CAPTURED`
2. Validates `refund_amount <= captured_amount - refunded_amount - refund_processing_amount`
3. For partial payments (payment_id is list): routes to `_handle_partial_payment_refund()` or `_handle_partial_payment_client_app_refund()`
4. POSTs to `REFUND_URL` with `payment_id`, `amount` (in cents), `sponsored_amount`, `terminal_id`
5. Returns status: `REFUND_SUCCESSFUL` (200), `REFUND_PROCESSING` (202), or `REFUND_FAILED` (500)

#### _handle_partial_payment_refund()

**Source**: `ms_pay.py:1153-1252`

Refunds across multiple partial payments. Distributes the refund amount across payment_ids:
1. Iterates `ms_pay_payment_details` to find refundable amounts per payment_id
2. Sends individual refund requests per payment_id
3. Tracks per-payment refunded/processing amounts
4. Returns composite status: `REFUND_PARTIAL_SUCCESSFUL`, `REFUND_PARTIAL_PROCESSING`

#### _handle_partial_payment_client_app_refund()

**Source**: `ms_pay.py:1254-1310`

Client-app initiated refund for partial payments. Accepts pre-split refund amounts from the client (via `refunds` array with per-payment amounts). Updates payment records first, then sends refund requests. Uses 2.5s timeout on MS Pay calls.

### Card Tokenization (Setup Session)

**Source**: `ms_pay.py:680-722`

Three methods manage card tokenization without a payment:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `create_setup_session()` | `POST SETUP_SESSION_URL` | Creates setup session with user_id, channel, geo, TokenEx gateway |
| `add_card_to_setup()` | `POST SETUP_SESSION_URL/{id}/add-token/` | Sends card data for tokenization |
| `setup_details()` | `POST SETUP_SESSION_URL/{id}/setup-details/` | Completes tokenization |

### Return Payment Sessions

**Source**: `ms_pay.py:1990-2170`

Return/refund payment flow for kiosk (3 methods):

| Method | Purpose |
|--------|---------|
| `create_return_payment_session()` | Creates a return session — validates order ownership, builds payload with `refund_variant_id`, sets `payment_type=RETURN`. Cash returns allowed for Flying Tiger and Walmart stores. |
| `return_details()` | Completes the return payment |
| `abort_return()` | Cancels the return |

### Void Payment

**Source**: `ms_pay.py:2183-2273`

Voids partial payments (reverses individual payment_ids). Calls `VOID_PAYMENT_URL`, then:
1. Removes voided payment_ids from `additional_payment_details.payment_id` list
2. Recalculates `captured_amount` by subtracting voided amounts
3. Removes matching `psp_reference` entries from `extra_data.split_payments`
4. If all payments voided: resets to `ACTION_REQUIRED` status with empty extra_data

### EdfaPay Integration

**Source**: `ms_pay.py:2275-2341`

Two methods for the EdfaPay PSP (Middle East):
- `edfapay_create_payment_session()` — Maps `store_id`/`retailer_store_id`/`retailer_id` to `business_unit_id`/`business_unit_code`/`business_unit_group_id`. Uses retry adapter (2 retries, 10s timeout).
- `edfapay_payment_details()` — Completes EdfaPay payment. 1s timeout.

### Walmart Pay Integration

**Source**: `ms_pay.py:2344-2417`

`initiate_walmart_pay_session()` — Prepares Walmart Pay checkout data:
1. HMAC-validates the request
2. Fetches order, store address, basket items
3. For associates: applies employee discount via `prepare_walmart_pay_basket_response()`
4. Returns store address, basket items with unit prices, and discount amount

### Gift Card Balance

**Source**: `ms_pay.py:2438-2462`

`get_gift_card_balance()` — Checks gift card balance via MS Pay. Maps `store_id` → `business_unit_id`. Uses retry adapter (2 retries, 5s timeout).

### Miscellaneous Service Methods

| Method | Source | Purpose |
|--------|--------|---------|
| `get_saved_cards()` | `ms_pay.py:674-678` | GET request to `SAVED_CARDS_URL` with `user_id` |
| `verify_client()` | `ms_pay.py:1835-1839` | POST to `VERIFY_CLIENT_URL` (Apple Pay domain verification) |
| `delete_card_token()` | `ms_pay.py:1841-1844` | POST to `SAVED_CARDS_URL/delete/` |
| `create_kiosk_stripe_terminal_token()` | `ms_pay.py:1956-1961` | POST to `MS_PAY_KIOSK_STRIPE_TOKEN_URL` |
| `get_refund_methods()` | `ms_pay.py:1963-1988` | GET to `REFUND_METHODS_URL` with retry adapter (2 retries, 1s timeout) |
| `get_viva_wallet_terminal_device_status()` | `ms_pay.py:2172-2181` | GET to `VIVA_DEVICE_STATUS_URL` (5s timeout) |
| `get_all_payment_session_details()` | `ms_pay.py:2419-2436` | GET to `GET_ALL_PAYMENT_SESSIONS_URL` (retry adapter, 5s timeout) |
| `_hmac_generator()` | `ms_pay.py:1825-1833` | Static method: HMAC-SHA256 of message with secret, base64-encoded |
| `_get_jwt_key_return_url()` | `ms_pay.py:1846-1889` | Generates JWT-signed return URL for web channel |
| `fetch_payment()` | `ms_pay.py:1891-1914` | HMAC-verified single payment fetch by order_id |
| `fetch_payments()` | `ms_pay.py:1916-1954` | HMAC-verified batch payment fetch |

## Views — Internal Service APIs

**Source**: `api.py` (20K, ~494 lines)

Internal API views for server-to-server calls within the platform. These bypass user authentication and operate directly on payment models and provider classes.

### Helper Functions

| Function | Source | Purpose |
|----------|--------|---------|
| `_get_provider_function(store_id, psp_type)` | `api.py:35-45` | Resolves PSP provider. Special-cases Saturn SmartPay → XPay |
| `payment_session_without_order(api_data)` | `api.py:48-55` | Creates a PSP session without an order via provider |
| `_get_payment_serializer_data(api_data)` | `api.py:58-108` | Builds `PaymentSerializer` input data: resolves currency, transaction_id prefix, variant, payment method. Falls back to MS Pay currency if no `PaymentVariant` exists. Auto-detects `saved_card` payment method from basket items |
| `check_payment_status(payment_obj)` | `api.py:111-122` | Returns 100 CONTINUE if payment already `CAPTURED` or `REFUNDED` |

### GetPaymentMethodsService

**Source**: `api.py:125-146` | **Method**: POST

Internal payment methods fetch. Calls `provider.get_payment_methods()` via `_get_provider_function()`.

### AdyenPaymentService

**Source**: `api.py:149-217` | **Method**: POST

Internal Adyen payment flow:
1. Creates `Payments` record if not exists
2. Calculates discounted amount + delivery
3. If `WAITING` or `REJECTED`: calls `payment_obj.preauth_for_amount()`
4. If preauth succeeds and `variant.capture` is true: calls `payment_obj.capture_for_amount()`

### AdyenHPPGenerateFormDataService

**Source**: `api.py:220-270` | **Method**: POST

Internal HPP form data generation. Creates payment record, saves merchant signature in `payment.token`.

### CreatePaymentSessionService

**Source**: `api.py:273-348` | **Method**: POST

Internal session creation with significant additional logic:
- Checks payment status (skips if already processed)
- For XPay, WorldPay, BrainTreePOS: increments `o_id` to the next available for the store
- Calls `payment_obj.payment_session()` (provider method)
- Stores all session responses in `extra_data.payment_responses.session_responses[]`

### VerifyPaymentService

**Source**: `api.py:358-400` | **Method**: POST

Internal verify with Optile routing. Routes to `optile_verify_payment()` if `variant.via_optile`, otherwise `verify_payment()`.

### ConfirmPaymentService

**Source**: `api.py:403-451` | **Method**: POST

Internal confirm with Optile routing. Routes to `optile_confirm_payment()` if `variant.via_optile`, otherwise `confirm_payment()`. Returns `dropin_action` data for Adyen Drop-in.

### ByPassPaymentService

**Source**: `api.py:454-493` | **Method**: POST

Internal bypass payment. Creates payment record if needed, delegates to `payment_obj.by_pass_payment()`.

## Payment Processing Flows

### Legacy Three-Step Flow (Direct PSP)

```
Client                    MainServer                    Internal Svc           PSP
  │                            │                              │                  │
  │── CreatePaymentSession ──>│                              │                  │
  │                            │── PaymentManagementClient ─>│                  │
  │                            │                              │── provider.     │
  │                            │                              │   payment_      │
  │                            │                              │   session() ──>│
  │                            │<────────────────────────────│<─── session ───│
  │<── session_data ──────────│                              │                  │
  │                            │                              │                  │
  │── VerifyPayment ─────────>│                              │                  │
  │                            │── PaymentManagementClient ─>│                  │
  │                            │                              │── provider.     │
  │                            │                              │   verify() ──>│
  │                            │<────────────────────────────│<─── result ────│
  │<── verify_result ─────────│                              │                  │
  │                            │                              │                  │
  │── ConfirmPayment ────────>│                              │                  │
  │                            │── PaymentManagementClient ─>│                  │
  │                            │                              │── provider.     │
  │                            │                              │   confirm() ──>│
  │                            │<────────────────────────────│<─── result ────│
  │<── confirm_result ────────│                              │                  │
```

### MS Pay Flow (Microservice)

```
Client                    MainServer (views.py)      PaymentService         MS Pay
  │                            │                        (ms_pay.py)          Microservice
  │                            │                              │                  │
  │── payment-methods/ ──────>│                              │                  │
  │                            │── get_allowed_payment_ ────>│                  │
  │                            │   methods()                  │── GET /methods ─>│
  │<── methods ───────────────│<─────────────────────────────│<── methods ─────│
  │                            │                              │                  │
  │── payment-session/ ──────>│                              │                  │
  │                            │── create_payment_ ─────────>│                  │
  │                            │   session()                  │── POST /session/ >│
  │                            │   [creates Payments obj]     │                  │
  │<── session_id ────────────│<─────────────────────────────│<── session ─────│
  │                            │                              │                  │
  │── add-payment/ ──────────>│                              │                  │
  │   {card_token, wallet}    │── add_payment() ────────────>│                  │
  │                            │                              │── POST /add- ──>│
  │<── status ────────────────│<─────────────────────────────│<── result ──────│
  │                            │                              │                  │
  │── payment-details/ ──────>│    (if 3DS / terminal)       │                  │
  │                            │── payment_details() ───────>│                  │
  │                            │                              │── POST /details >│
  │<── final_result ──────────│<─────────────────────────────│<── final ───────│
  │                            │                              │                  │
  │                            │                              │       (async)    │
  │                            │                              │<── WEBHOOK ─────│
  │                            │── handle_webhook() ────────>│                  │
  │                            │   [_mark_payment_success()]  │                  │
  │                            │   [order.complete_status()]  │                  │
```

### Partial Payment Flow (Kiosk/Terminal)

```
Terminal                  MainServer                MS Pay              PSP
  │                          │                         │                  │
  │── create-session ───────>│                         │                  │
  │<── session_id ──────────│                         │                  │
  │                          │                         │                  │
  │── add-payment ($20) ───>│                         │                  │
  │                          │── POST /add-payment ──>│── charge $20 ──>│
  │                          │<── status: partial ────│<── $20 ok ──────│
  │                          │                         │                  │
  │<── partial_captured,     │                         │                  │
  │    new_payment_session   │                         │                  │
  │                          │                         │                  │
  │── add-payment ($30) ───>│                         │                  │
  │   (to new session)      │── POST /add-payment ──>│── charge $30 ──>│
  │                          │<── status: success ────│<── $30 ok ──────│
  │                          │                         │                  │
  │<── captured ($50 total)  │                         │                  │
  │                          │                         │                  │
  │                          │── _mark_payment_success │                  │
  │                          │   order.complete_status │                  │
```

## Open Questions

- Why does `CustomerOptileProfile` lack a `unique_together` constraint on `customer` + `account_id`? The TODO at `models.py:556` acknowledges this.
- Is the legacy direct-PSP payment path being actively phased out in favor of MS Pay? Which PSPs are still using the legacy path?
- The `validate_merchantsig()` function is a complete no-op. Is this intentionally disabled, or is it a forgotten fix?
- Are the 170+ currency codes in `MS_PAY_CURRENCY_CODES` all actually supported, or is this a copy of the full ISO 4217 list?
- `PayoutScheduler.stripe_api_key` stores API keys in plaintext in the database. Is this secured at the database access level, or should it use encryption at rest?
- The `payment_helper.get_payment_method_summary()` function iterates all orders in a date range and filters in Python instead of SQL. Is this a performance concern for high-volume stores?
- `PaymentVariant.via_optile` and `optile_specific_params` suggest Optile was used as a payment aggregator. Is Optile still in use, or has it been replaced by MS Pay?
- The `extra_data` JSON field on `Payments` accumulates response history without any cleanup. Does this field grow indefinitely?
- `MsPayWebhook` uses `AllowAny` permission — HMAC validation happens only inside `PaymentService.handle_webhook()`. A middleware-level check would be more secure.
- `create_payment_session()` uses mutable default argument `start_datetime=datetime.datetime.utcnow()` — this evaluates at function definition time, not call time, though callers always pass `start_datetime` explicitly.
- The `XpayNotify` handler uses `print()` statements instead of `logger`. Several other webhook handlers also use `print()` for debugging.
- `_handle_partial_payment_client_app_refund()` uses a 2.5s timeout — is this sufficient for terminal-based refunds?
