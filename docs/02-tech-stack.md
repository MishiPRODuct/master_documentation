# Tech Stack

> Last updated by: Task T00 — Project Overview & Architecture Map

## Overview

The MishiPay platform is built primarily on Python/Django, with additional services in Node.js and Java. Data is stored across PostgreSQL, MySQL, MongoDB, and Elasticsearch. The system uses Kafka for event streaming, RabbitMQ for task queuing, and Kong/Keycloak for API management and identity.

---

## Languages & Runtimes

| Language | Version | Usage |
|----------|---------|-------|
| **Python** | 3.9 | Monolith, promotions service, payment service, tax service, voucher service |
| **Node.js** | — | Payment gateway (`pay-gateway`) |
| **Java** | — | Keycloak (identity server) |

## Web Frameworks

| Framework | Version | Repository | Notes |
|-----------|---------|------------|-------|
| **Django** | 4.1.13 | MishiPay (monolith) | Main application framework |
| **Django** | 3.2.24 | service.promotions | Promotions microservice |
| **Django REST Framework** | 3.14.0+ | MishiPay | REST API layer for monolith |
| **Django REST Framework** | 3.11.2 | service.promotions | REST API layer for promotions |

## Databases

| Database | Version | Service | Usage |
|----------|---------|---------|-------|
| **PostgreSQL** | 11.8 | `backend_db_v2` | Primary data store for the monolith (items, orders, payments, customers, etc.) |
| **Percona MySQL** | 8.0.20 | `ms_promo_db` | Promotions service and loyalty service (shared instance in dev) |
| **MySQL** | 8.0.25 | `mishi-voucher-db`, `tax-db` | Voucher service, tax service |
| **MongoDB** | 4.4.0 | `mongo-db` | Payment service data store |
| **MongoDB** | 4.4 | `mongo_inventory` | Inventory service data store |
| **Elasticsearch** | 7.11.2 | `elastic_inventory` | Inventory search and indexing |

## Message Brokers & Task Queues

| Technology | Version | Purpose |
|------------|---------|---------|
| **Confluent Kafka** | Managed service | Event streaming — post-payment processing, analytics events |
| **RabbitMQ** | 3.8.5 | Message broker for Dramatiq background tasks |
| **Dramatiq** | 1.6.1 | Python task queue library (runs on RabbitMQ) |
| **Redis** | 6.0.6 | Dramatiq result backend, general caching |
| **Memcached** | 1.6.6 | Production caching layer |

## API Gateway & Authentication

| Technology | Version | Purpose |
|------------|---------|---------|
| **Kong** | — | API gateway — routing, SSL termination, rate limiting |
| **Keycloak** | — | Identity and access management (OAuth2/OpenID Connect) |
| **Django Token Auth** | — | REST Framework token-based authentication |
| **Social Auth** | 4.3.0+ | Social login (via `social-auth-app-django`) |

## Infrastructure & DevOps

| Technology | Version | Purpose |
|------------|---------|---------|
| **Docker** | Compose v3.8 | Containerization for all services |
| **NGINX** | 1.21.1 | Reverse proxy, static file serving |
| **Gunicorn** | 20.1.0+ | WSGI server for Django in production |
| **Datadog** | Agent v7 | APM, logging, monitoring (`ddtrace` for Python tracing) |
| **Sentry** | SDK 1.14.0+ | Error tracking and reporting |
| **Elastic APM** | 5.7.0 (monolith) / 6.9.1+ (promos) | Application performance monitoring |
| **Poetry** | — | Python dependency management (both repos) |
| **GitHub Actions** | — | CI/CD pipeline |
| **Azure Pipelines** | — | CI/CD pipeline (promotions service) |
| **Azure CLI** | 2.76.0 | Cloud resource management from containers |

## Key Python Libraries (Monolith)

### Payment Gateways
| Library | Version | Purpose |
|---------|---------|---------|
| `stripe` | 2.45.0 | Stripe payment integration |
| `braintree` | 3.41.0 | Braintree payment integration |
| `Adyen` | 3.0.0 | Adyen payment integration |
| `connect-sdk-python3` | 3.13.0 | Ingenico Connect payment SDK |
| `opp` | 1.3.1 | Open Payment Platform integration |

### Communication & Messaging
| Library | Version | Purpose |
|---------|---------|---------|
| `confluent-kafka` | 1.8.2+ | Kafka client for event streaming |
| `pika` | 1.2.0 | RabbitMQ client (used alongside Dramatiq) |
| `paho-mqtt` | 1.3.0 | MQTT messaging (IoT device communication) |
| `firebase-admin` | 2.13.0 | Firebase Cloud Messaging (push notifications) |
| `django-push-notifications` | 3.0.0+ | Push notification management |

### Email & Marketing
| Library | Version | Purpose |
|---------|---------|---------|
| `sib-api-v3-sdk` | 7.6.0+ | Sendinblue (Brevo) email API |
| `Sendinblue` | 2.0.5.1 | Legacy Sendinblue integration |

### Analytics & Tracking
| Library | Version | Purpose |
|---------|---------|---------|
| `mixpanel` | 4.9.0 | Mixpanel analytics |
| `mixpanel-py-async` | 0.3.0 | Async Mixpanel events |
| `datadog` | 0.44.0+ | Datadog metrics and events |
| `ddtrace` | 1.4.5+ | Datadog distributed tracing |

### Data Processing
| Library | Version | Purpose |
|---------|---------|---------|
| `pandas` | latest | Data manipulation and analysis |
| `XlsxWriter` | 1.1.2+ | Excel file generation |
| `xlwt` | 1.3.0+ | Legacy Excel file writing |
| `unicodecsv` | 0.14.1 | Unicode-safe CSV processing |
| `gspread` | 5.4.0+ | Google Sheets integration |

### XML & SOAP
| Library | Version | Purpose |
|---------|---------|---------|
| `zeep` | 2.5.0 | SOAP client (retailer integrations) |
| `xmltodict` | 0.11.0+ | XML parsing |
| `xmlformatter` | 0.1.1+ | XML formatting |
| `xmldiff` | 2.7.0 | XML comparison |

### gRPC
| Library | Version | Purpose |
|---------|---------|---------|
| `grpcio` | 1.53.2 | gRPC framework (tax service communication) |
| `grpcio-tools` | 1.43.* | Proto compilation tools |
| `protobuf` | 3.18.3+ | Protocol Buffers serialization |

### Django Extensions
| Library | Version | Purpose |
|---------|---------|---------|
| `django-cors-headers` | 3.13.0+ | CORS handling |
| `django-filter` | 2.4.0 | Queryset filtering for APIs |
| `django-import-export` | 2.9.0+ | Data import/export (admin) |
| `django-mptt` | 0.14.0+ | Modified Preorder Tree Traversal (hierarchical data) |
| `django-cron` | 0.6.0+ | Cron job scheduling |
| `django-guid` | 2.2.1 | Request GUID tracking |
| `django-health-check` | 3.16.4+ | Health check endpoints |
| `drf-yasg` | 1.21.5+ | Swagger/OpenAPI documentation generation |
| `django-admin-rangefilter` | 0.8.4+ | Admin date range filtering |
| `django-nested-inline` | 0.4.5+ | Nested inline admin editing |
| `django-migration-linter` | 1.3.0 | Migration safety checks |
| `whitenoise` | 6.4.0+ | Static file serving in production |

### Security & Auth
| Library | Version | Purpose |
|---------|---------|---------|
| `PyJWT` | 2.4.0+ | JSON Web Token encoding/decoding |
| `social-auth-app-django` | 5.0.0+ | Social authentication (OAuth) |
| `pycryptodome` | 3.19.1+ | Cryptographic operations |
| `pyodbc` | 4.0.39+ | ODBC database connections (SQL Server via MSODBCSQL17) |
| `requests-ntlm` | 1.2.0+ | NTLM authentication for Windows-based APIs |

### AI & LLM
| Library | Version | Purpose |
|---------|---------|---------|
| `anthropic` | 0.64.0+ | Anthropic Claude API client |
| `langchain` | 0.3.27+ | LLM orchestration framework |
| `langchain-anthropic` | 0.3.19+ | LangChain Anthropic integration |

### Other
| Library | Version | Purpose |
|---------|---------|---------|
| `geopy` | 2.1.0+ | Geocoding and distance calculations |
| `phonenumbers` | 8.10.7+ | Phone number parsing and validation |
| `cloudinary` | 1.38.0+ | Image/media cloud storage |
| `ksuid` | 1.3 | K-Sortable Unique Identifiers |
| `nanoid` | 2.0.0+ | Compact unique ID generation |
| `epc-encoding-utils` | 1.3+ | EPC/RFID tag encoding |
| `uttlv` | 0.7.0+ | TLV (Tag-Length-Value) data parsing |
| `azure-storage-blob` | 12.21.0+ | Azure Blob Storage integration |
| `azure-identity` | 1.17.1+ | Azure authentication |
| `better-profanity` | 0.7.0+ | Profanity filtering |

## Key Python Libraries (Promotions Service)

| Library | Version | Purpose |
|---------|---------|---------|
| `Django` | 3.2.24 | Web framework |
| `djangorestframework` | 3.11.2 | REST API |
| `mysqlclient` | 2.0.1 | MySQL database driver |
| `networkx` | 2.8.5+ | Graph algorithms (promotion cluster formation in best-discount algo) |
| `gunicorn` | 20.1.0+ | Production WSGI server |
| `sentry-sdk` | 1.45.1 | Error tracking |
| `ddtrace` | 0.47.0 | Datadog tracing |
| `elastic-apm` | 6.9.1+ | Elastic APM |
| `django-guid` | 2.2.0 | Request correlation |
| `django-health-check` | 3.16.5 | Health endpoints |
| `whitenoise` | 6.0.0+ | Static file serving |
| `uvicorn` | 0.17.0+ | ASGI server (async support) |
| `ksuid` | 1.3 | Unique ID generation |

## Development & Testing Tools

| Tool | Version | Purpose |
|------|---------|---------|
| `pytest` | 6.2.4+ | Test runner (monolith) |
| `pytest-django` | 4.4.0+ | Django test integration |
| `pytest-xdist` | 2.3.0+ | Parallel test execution |
| `pytest-cov` | 2.12.1+ | Code coverage |
| `factory-boy` | 3.2.0+ | Test data factories |
| `freezegun` | 1.1.0 | Time mocking in tests |
| `httpretty` | 1.0.5 | HTTP request mocking |
| `requests-mock` | 1.12.1+ | Requests library mocking |
| `flake8` | 3.9.2+ | Code linting |
| `black` | 21.6b0+ | Code formatting |
| `isort` | — | Import sorting |
| `bandit` | 1.7.0+ | Security linting |
| `mypy` | 0.910+ | Static type checking |
| `pre-commit` | 2.20.0+ | Git pre-commit hooks |
| `django-debug-toolbar` | 3.2.1+ | Debug toolbar (dev only) |
| `Werkzeug` | 3.0.6+ | Debug utilities |

## Base Docker Images

| Image | Version | Used By |
|-------|---------|---------|
| `python` | 3.9.9-slim-bullseye | Monolith, Promotions service |
| `postgres` | 11.8 | Monolith database |
| `percona` | 8.0.20-11-centos | Promotions / Loyalty MySQL database |
| `mysql` | 8.0.25 | Voucher / Tax databases |
| `mongo` | 4.4.0 / 4.4 | Payment service, Inventory service |
| `elasticsearch` | 7.11.2 | Inventory search |
| `rabbitmq` | 3.8.5-alpine | Background task messaging |
| `redis` | 6.0.6-alpine | Caching and result backend |
| `memcached` | 1.6.6-alpine | Production caching |
| `nginx` | 1.21.1 | Reverse proxy, static files |

## External Services

| Service | Purpose |
|---------|---------|
| **GitHub Container Registry** (`ghcr.io`) | Docker image hosting |
| **GitHub Packages** (`docker.pkg.github.com`) | Legacy Docker image hosting |
| **Confluent Kafka** | Managed Kafka event streaming |
| **Datadog** | Monitoring, APM, log aggregation |
| **Sentry** | Error tracking |
| **Elastic APM** | Application performance monitoring |
| **Sendgrid / Sendinblue (Brevo)** | Transactional email delivery |
| **Firebase** | Push notifications |
| **Mixpanel** | Product analytics |
| **Cloudinary** | Image/media management |
| **Azure Blob Storage** | File storage |
| **Google Sheets API** | Spreadsheet integration |
| **Zscaler** | Network security (developer tooling) |

## Notable Architecture Decisions

1. **No Foreign Keys in Promotions Service:** The promotions service explicitly avoids Django ForeignKey constraints for scalability. Relationships are maintained at the application level.

2. **Type Hints Mandatory in Promotions Service:** All code in `service.promotions` requires Python type hints, enforced via CI.

3. **Stateless Django in Promotions:** The promotions service is designed to be stateless — all important data resides in MySQL, not in the container.

4. **NetworkX for Promotion Clustering:** The best-discount algorithm uses the `networkx` graph library to form promotion clusters based on connected components of item-promotion relationships.

5. **Dual Database Strategy:** PostgreSQL for the monolith (relational data), MySQL for microservices (promotions, loyalty, voucher, tax), MongoDB for payments and inventory.

6. **Kong + Keycloak for Auth:** API gateway handles routing and SSL, while Keycloak manages identity (OAuth2/OIDC). The monolith also supports its own token authentication via DRF.

7. **MSODBCSQL17 Driver:** The monolith includes the Microsoft ODBC driver for SQL Server, suggesting integration with retailer SQL Server databases (likely DOS/retailer POS systems).

## Open Questions

- What specific Kafka topics exist and what events flow through them?
- What is the exact role split between `ms-tax` (gRPC, port 9000) and `mishi-tax` (HTTP, port 8008)? Likely a legacy → new migration.
- How is the `inventory-common` private package (from GitHub) structured and used?
- What is the `explorer` Django app in INSTALLED_APPS? (Possibly Django SQL Explorer for ad-hoc queries)
