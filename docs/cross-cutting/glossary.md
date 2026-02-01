# Glossary of MishiPay-Specific Terms

> Last updated by: Task T39 — Finalization: Index, Glossary & Review

This glossary defines terms, acronyms, and concepts specific to the MishiPay platform. Terms are grouped by domain and alphabetically sorted within each group.

---

## Business & Domain Terms

| Term | Definition | See Also |
|------|-----------|----------|
| **Basket** | A shopping cart scoped to a specific customer + store pair. Contains `BasketItem` entries, tracks promotion application state, and can expire. The basket's `special_promos` JSON field stores active discount programs. | [Items](../platform/items.md) |
| **BasketEntityAuditLog** | The primary record of how promotions/discounts were applied to basket items. Drives the basket display (prices, savings, quantities) in the checkout flow. | [Items](../platform/items.md) |
| **Bank Drop** | Cash removed from the till during a kiosk shift, tracked as `removed_shift_amount`. | [Cashier Kiosk](../platform/cashier-kiosk.md) |
| **Business Date** | The local-timezone date of a shift's start time, used for daily aggregation of kiosk shift data. | [Cashier Kiosk](../platform/cashier-kiosk.md) |
| **Bypass Payment** | Zero-amount orders that skip payment processing entirely (e.g., 100% discounted items via subscription or staff discount). | [Payments](../platform/payments.md) |
| **Click & Collect (CC)** | Alternative purchase flow where customers order online and pick up in-store. Has its own status lifecycle: `created` → `accepted` → `ready_to_collect` → `collected`. | [Orders](../platform/orders.md), [Dashboard](../platform/dashboard.md) |
| **Coffee Subscription** | The only subscription type currently implemented — a recurring Stripe payment granting 100% discount on eligible coffee items at participating stores, with daily quantity limits. | [Subscriptions & Referrals](../platform/subscriptions-referrals.md) |
| **Dashboard User** | A staff member with login credentials and store-level permissions, belonging to the `DashboardUser` Django group. Accesses the web-based retail staff portal. | [Dashboard](../platform/dashboard.md) |
| **Denomination Breakup** | A JSON dictionary mapping currency denominations to their total value in the till (e.g., `{"0.01": 0.05, "10.0": 100}`). Validated to sum to the declared amount at shift end. | [Cashier Kiosk](../platform/cashier-kiosk.md) |
| **Evaluate Criteria** | Determines whether a promotion is evaluated per-product (`p`) or per-basket (`b`). | [Business Logic](../promotions-service/business-logic.md) |
| **Geofencing** | Distance-based access control ensuring customers are physically near a store before scanning/purchasing items. | [Retailer Modules](../platform/retailer-modules.md) |
| **Item** | A product in a store's inventory (`ItemInventory`), identified by a `product_identifier` (barcode/databar) and scoped to an `ItemCategory` which belongs to a `Store`. | [Items](../platform/items.md) |
| **Kiosk Shift** | A time-bounded work session for a staff member on a specific Android-based POS cashier kiosk device. Only one active shift is allowed per device at a time. | [Cashier Kiosk](../platform/cashier-kiosk.md) |
| **Loyalty Card Scan** | The primary mechanism for linking a customer to a retailer's loyalty programme. Each retailer has its own loyalty service implementation dispatched via `CUSTOMER_CALLABLE_SERVICE`. | [Customers](../platform/customers.md) |
| **ManualVerificationSession** | Age/alcohol verification workflow for restricted items, initiated by store staff via the dashboard. | [Items](../platform/items.md), [Dashboard](../platform/dashboard.md) |
| **Marketing Consent** | GDPR-compliant consent tracking with versioning, per-retailer and per-MishiPay consent, IP hashing, and configurable default behaviour per region. | [Customers](../platform/customers.md) |
| **MishiPay ID (`mishid`)** | A unique device/session identifier sent via `x-mishipay-id` header. Used to track guest users across requests without requiring login. | [Authentication](../platform/authentication.md) |
| **o_id** | Sequential order number per store, generated at checkout via `StoreTransactionSequenceNumber` with `SELECT FOR UPDATE` locking. | [Orders](../platform/orders.md) |
| **On-the-fly promo** | A discount created dynamically at checkout time (e.g., staff discount, ad-hoc override) rather than pre-configured in the promotions microservice. | [Coupons & Promotions](../platform/coupons-promotions.md) |
| **Order Completion** | Multi-step process triggered after payment: VAT calculation, poslog assignment, retailer-specific fulfilment, receipt sending, coupon redemption, notifications. | [Orders](../platform/orders.md) |
| **Payout** | Transfer of collected funds from MishiPay's platform account to the retailer's bank account. Managed by the payment module's scheduling system. | [Payments](../platform/payments.md) |
| **Preauth → Capture** | Two-step payment flow: authorize the amount first, then capture it upon order completion. Used by most PSPs. | [Payments](../platform/payments.md) |
| **Privacy Policy** | Versioned opt-in/opt-out consent tracked per customer + store type, supporting "don't ask again" and MishiPay app vs white-label app distinction. | [Customers](../platform/customers.md) |
| **Program** | A discount category in `special_promos`: `REFERRAL`, `RETAILER_COUPON`, `VOUCHER_DISCOUNT`, `IT_GOVT_LOTTERY_CODE`, `DONATION`, etc. | [Coupons & Promotions](../platform/coupons-promotions.md) |
| **Promotion Family** | One of 8 promotion types in the promotions microservice: Easy (`e`), Easy Plus (`p`), Combo (`c`), Line Special (`l`), Basket Threshold (`b`), Basket Threshold with Discounted Target (`t`), Meal Deal (`m`), Receipt (`r`). | [Business Logic](../promotions-service/business-logic.md) |
| **Quota** | Generic quota tracking model used for retailer-specific purchase limits (e.g., DDF duty-free allowances). | [Orders](../platform/orders.md) |
| **Receipt Type** | Category of receipt: `ORDER`, `REFUND`, `GIFT`, `STEB`, `PAYMENT_FAILED`, `COUNTER`, `PAYMENT`, `CAMPAIGN`. | [Emails & Alerts](../platform/emails-alerts.md) |
| **Reduce to Clear (RTC)** | Avery Dennison markdown barcode format (`MP` prefix, 25 chars) encoding a barcode + discount (fixed price or percent off) with Luhn checksum validation. | [Inventory](../platform/inventory.md) |
| **Referral Campaign** | A time-bound program where existing users (source) share codes with new users (target), both receiving coupon rewards. Managed by MS Loyalty Service. | [Subscriptions & Referrals](../platform/subscriptions-referrals.md) |
| **Retailer String** | Namespace format `"{CHAIN.upper()}-{REGION.upper()}"` (e.g., `"MUJI-GB"`, `"DECATHLON-FR"`) used to scope promotions to a specific retailer + region. | [Coupons & Promotions](../platform/coupons-promotions.md) |
| **Scan-and-go** | MishiPay's core product offering: customers scan items with their mobile phone, pay in-app, and exit the store without visiting a traditional checkout. | [Architecture Overview](../01-architecture-overview.md) |
| **SecureItem** | Items with a `uid` field (EPC/RFID tag identifier) enabling anti-theft tracking via Redis and MQTT. | [Retailer Modules](../platform/retailer-modules.md) |
| **Settlement Reconciliation** | Reports comparing payment gateway records against internal order/payment data. Used by finance teams. | [Dashboard](../platform/dashboard.md), [Emails & Alerts](../platform/emails-alerts.md) |
| **SimplePromotion** | Lightweight promotion type (simple discount, meal deal, X for Y) used by specific retailers directly on the platform, without the promotions microservice. | [Items](../platform/items.md) |
| **Special Promos** | JSON field on `Basket` model (`basket.special_promos`) storing active discount programs for a shopping session: coupons, vouchers, lottery codes, donations. | [Coupons & Promotions](../platform/coupons-promotions.md) |
| **Store-Type Dispatch** | Pattern where behaviour is selected via `store.store_type.lower()` lookups in dispatch maps. Over 100 store types registered across all modules. | [Retailer Modules](../platform/retailer-modules.md) |
| **transaction_id_poslog** | Sequential POS log transaction ID assigned after payment, used for retailer ERP/POS integration. | [Orders](../platform/orders.md) |
| **Transaction Token** | A short-lived (10-minute) UUID token used for securing certain transaction-related operations. | [Customers](../platform/customers.md) |

---

## Architecture & Technical Terms

| Term | Definition | See Also |
|------|-----------|----------|
| **ACR** | Azure Container Registry — used for production Docker images (promotions service). | [Promotions DevOps](../promotions-service/devops.md) |
| **Agent Pattern** | Legacy mechanism for retailer-specific item operations. `BaseAgent` defines abstract methods; subclasses implement retailer-specific item lookup, sold-status tracking, and inventory management. | [Retailer Modules](../platform/retailer-modules.md) |
| **Auth Definitions** | Data-driven dictionaries in `auth_definitions.py` that declare the signup/login behaviour for each auth service type — what entities to create, what prerequisites to check. | [Authentication](../platform/authentication.md) |
| **Best-Discount Algorithm** | Graph-based clustering algorithm in the promotions microservice that finds the optimal combination of promotions for a basket. Two variants: exhaustive (brute-force, O(n!)) and marginal-value (greedy, O(n²)). | [Business Logic](../promotions-service/business-logic.md) |
| **Confluent Cloud** | Managed Apache Kafka service (West Europe Azure region) used for asynchronous event streaming between platform components. | [Infrastructure](../03-infrastructure.md), [Data Flow](./data-flow.md) |
| **DOS** | "Deloitte Operating System" — the foundational retail integration layer of the MishiPay platform. Despite the historical name, it serves as the core application powering all retailer integrations (363 files). | [Retailer Modules](../platform/retailer-modules.md) |
| **Dramatiq** | Background task framework used for async processing. Tasks are Python functions decorated with `@dramatiq.actor` and dispatched to RabbitMQ queues. Custom middleware tracks task state in the database. | [Analytics](../platform/analytics.md) |
| **Entity Action Constraints** | A rule engine (`ActionConstraint` / `EntityActionConstraints` models) that evaluates dynamic expressions against variables to enforce business constraints on actions like item scanning, basket updates, or payments. | [Core Module](../platform/core.md) |
| **Entity Properties** | A generic key-value system (`EntityProperty` model) for attaching arbitrary JSON configuration to entities (items, payments, orders, baskets, customers, etc.) without schema changes. | [Core Module](../platform/core.md) |
| **Epson Markup** | POS printer format using tags: `[L]` (left), `[R]` (right), `[C]` (center), `<b>`, `<img>`, `<qrcode>`. Used for kiosk receipt rendering. | [Emails & Alerts](../platform/emails-alerts.md) |
| **Event Context** | Every analytics event is wrapped in an `EventContext` that captures *who* (subject), *what* (verb), *where* (indirect object = store), plus device, location, IP, and application metadata. | [Analytics](../platform/analytics.md) |
| **GHCR** | GitHub Container Registry — used for development/CI Docker images. | [Promotions DevOps](../promotions-service/devops.md) |
| **GlobalConfig** | A database-backed key-value store where keys are short strings (max 50 chars) and values are arbitrary JSON. Used for feature flags, store-level configuration, and marketing consent rules. | [Authentication](../platform/authentication.md) |
| **Hierarchical Config Lookup** | Configuration resolution from most-specific to least: store+region > store+ALL > ALL+region > ALL+ALL. Used for receipt and email configuration. | [Emails & Alerts](../platform/emails-alerts.md) |
| **Inventory Service** | External Node.js microservice (`node-express-inventory-service`) that stores the canonical product catalog in MongoDB. The monolith's `inventory` module acts as a proxy. | [Inventory](../platform/inventory.md), [Infrastructure](../03-infrastructure.md) |
| **Kong** | Open-source API Gateway that sits in front of the Django monolith. Validates Keycloak JWTs, sets `x-mishipay-id` and `x-anonymous-consumer` headers. | [Infrastructure](../03-infrastructure.md), [Authentication](../platform/authentication.md) |
| **KSUID** | K-Sortable Unique Identifier — used as primary keys in the promotions service instead of auto-increment IDs, for scalability. | [Promotions Data Models](../promotions-service/data-models.md) |
| **Keycloak** | Open-source identity and access management system. MishiPay uses it for JWT-based authentication via the Kong API gateway. | [Authentication](../platform/authentication.md), [Infrastructure](../03-infrastructure.md) |
| **Microservices Client** | Internal routing layer (`microservices_client` module) that uses Django's test `Client()` to make HTTP requests to other internal endpoints, acting as an in-process service bus. | [Integrations](../platform/integrations.md) |
| **mpay_net** | Docker network used for internal service communication. Services on this network communicate without authentication. | [Infrastructure](../03-infrastructure.md) |
| **MS Loyalty Service** | External microservice that manages coupon lifecycle (create, validate, redeem), user coupon inventories, and referral campaigns. | [Coupons & Promotions](../platform/coupons-promotions.md), [Subscriptions & Referrals](../platform/subscriptions-referrals.md) |
| **MS Pay** | MishiPay's internal payment microservice — the newer architecture replacing direct PSP integrations. Supports 35+ payment gateways and handles Stripe webhooks for subscriptions. | [Payments](../platform/payments.md) |
| **ms-promo / service.promotions** | Standalone Django microservice for promotional data ingestion and discount evaluation. Evaluates baskets against promotions using 8 family types and a best-discount algorithm. | [Promotions Service Overview](../promotions-service/overview.md) |
| **ms-tax** | Tax evaluation microservice. Two interfaces: legacy gRPC (`tax_pb2_grpc`) and newer HTTP REST. Handles per-store tax tables and basket tax evaluation. | [Integrations](../platform/integrations.md), [Infrastructure](../03-infrastructure.md) |
| **PaymentVariant** | Per-store PSP configuration model defining which payment provider a store uses, including API keys, endpoints, and supported payment methods. | [Payments](../platform/payments.md) |
| **PromoHeader / PromoGroup / PromoGroupNode** | Hierarchical data model in the promotions microservice. A `PromoHeader` defines a promotion containing `PromoGroups` (sets of qualifying items/categories), which contain `PromoGroupNodes` (individual SKUs or categories). | [Promotions Data Models](../promotions-service/data-models.md) |
| **PSP** | Payment Service Provider — third-party payment processor (Adyen, Stripe, Braintree, etc.). The platform supports 22 legacy PSP types and 35+ via MS Pay. | [Payments](../platform/payments.md) |
| **pubsub** | Kafka-based publish/subscribe messaging framework in the monolith. Uses Confluent Kafka with SASL/SSL authentication. Provides 6 producer adapters and 3 consumer daemons. | [Support Modules](../platform/support-modules.md), [Data Flow](./data-flow.md) |
| **RetailerSystemConfiguration** | Wrapper system (V1: dict-based, V2: model-based) that loads per-retailer configuration modules based on store type and demo/live mode. Provides access to product, order, firebase, customer, legal_requirements, and email settings. | [Core Module](../platform/core.md) |
| **StoreType** | Abstract base class (`dos.base_store_type.StoreType`) that defines per-retailer behaviour for item management, orders, payments, and refunds. Over 100 concrete implementations mapped via `STORE_TYPE_MAP`. | [Retailer Modules](../platform/retailer-modules.md) |
| **VMSS** | Azure Virtual Machine Scale Sets — the deployment target for the monolith. Production and consumer instances run in separate VMSS. | [Platform Infrastructure](../platform/infrastructure.md) |

---

## Retailer-Specific Terms

| Term | Definition | See Also |
|------|-----------|----------|
| **AADE myDATA** | Greek tax authority system — fiscal compliance for Greece requires obtaining a mark ID from AADE for each transaction. | [Support Modules](../platform/support-modules.md) |
| **DDF** | Dubai Duty Free — a major airport retailer with custom authentication (QR + airport boarding pass), tab-based permissions, kiosk workflows, and duty-free quota tracking. | [Dashboard](../platform/dashboard.md), [Orders](../platform/orders.md) |
| **Decathlon SSO** | Custom OAuth2 integration for Decathlon customers. Supports two parallel flows: older direct-API approach and newer OAuth2 callback-based flow. Includes Red Discount loyalty tiers (Bronze=5%, Silver=5%, Gold=7%, Platinum=10%). | [Integrations](../platform/integrations.md) |
| **Eroski** | Spanish retailer — the first customer for the promotions microservice. Used as the reference implementation for many promotion features. | [Promotions Service Overview](../promotions-service/overview.md) |
| **Fiscalisation** | Tax compliance process for generating legally-required fiscal receipts. Two country-specific implementations: Italy (Venistar/Retail Pro polling) and Greece (Funky Buddha AADE callback). | [Support Modules](../platform/support-modules.md) |
| **inventory-common** | Private Python package (`python-inventory-common`) providing `InventoryV1Client`, enums, and import utilities for communicating with the Inventory Service. | [Inventory](../platform/inventory.md) |
| **Lotteria degli Scontrini** | Italian receipt lottery system — fiscal compliance requires generating lottery-compatible receipt codes. | [Support Modules](../platform/support-modules.md) |
| **Pfand** | German deposit/return system for bottles and cans. Handled by the Saturn Smart Pay integration with special barcode prefixes. | [Support Modules](../platform/support-modules.md) |
| **promo_group_qualifier** | Identifier linking a coupon/discount to a promotion definition in the promotions microservice. Used by special promos and loyalty programs. | [Coupons & Promotions](../platform/coupons-promotions.md) |
| **Red Discount** | Decathlon loyalty discount system with tiers: Bronze (5%), Silver (5%), Gold (7%), Platinum (10%). Applied based on `customer_tier` from the OAuth provider. | [Integrations](../platform/integrations.md) |
| **RPM (Retail Price Management)** | Dubai Duty Free's price management system. DDF sends JSON files with price updates that are ingested by the `ddf_price_activation` module. | [Support Modules](../platform/support-modules.md) |
| **STEB** | Saudi Tax and Excise Board — duty-free bag tracking receipts for Dubai Duty Free. A special receipt type. | [Emails & Alerts](../platform/emails-alerts.md) |
| **Venistar** | Italian fiscal compliance provider. Uses a polling-based workflow where the platform sends fiscal data and polls for completion status. | [Support Modules](../platform/support-modules.md) |

---

## Acronyms

| Acronym | Expansion |
|---------|-----------|
| **ACR** | Azure Container Registry |
| **CC** | Click & Collect |
| **DDF** | Dubai Duty Free |
| **DRF** | Django REST Framework |
| **EPC** | Electronic Product Code (RFID tag identifier) |
| **GHCR** | GitHub Container Registry |
| **KSUID** | K-Sortable Unique Identifier |
| **MQTT** | Message Queuing Telemetry Transport |
| **MPTT** | Modified Preorder Tree Traversal (Django library for hierarchical models) |
| **PSP** | Payment Service Provider |
| **RFID** | Radio-Frequency Identification |
| **RPM** | Retail Price Management (DDF) |
| **RTC** | Reduce to Clear |
| **SemVer** | Semantic Versioning |
| **SSO** | Single Sign-On |
| **STEB** | Saudi Tax and Excise Board |
| **VMSS** | Virtual Machine Scale Sets (Azure) |
| **3DS** | 3D Secure (card authentication protocol) |
