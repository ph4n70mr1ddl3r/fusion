# Quick Start Guide: Oracle Fusion Cloud ERP Clone

**Branch**: `001-oracle-fusion-erp-clone` | **Date**: 2026-04-26

## Prerequisites

- **Rust**: Latest stable (MSRV pinned in `rust-toolchain.toml`)
- **Node.js**: 20+ (for frontend)
- **Docker**: 24+ and Docker Compose v2
- **PostgreSQL**: 16+ (via Docker or local)
- **NATS**: 2.x with JetStream (via Docker)
- **sqlx-cli**: `cargo install sqlx-cli --no-default-features --features postgres`
- **protoc**: Protocol Buffers compiler (v3.x)

## One-Command Development Start

```bash
# Start all infrastructure (PostgreSQL, NATS, MailHog)
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d

# Build all services
cargo build

# Run database migrations for a tenant (example)
sqlx migrate run --database-url "postgres://fusion:fusion@localhost:5432/fusion_gl_tenant_001" --source services/gl/migrations

# Start all services
cargo run --bin fusion-gateway &
cargo run --bin fusion-gl &
cargo run --bin fusion-ap &
cargo run --bin fusion-ar &
# ... or use Docker Compose for all services
```

## Project Structure

```text
fusion/
├── Cargo.toml                    # Workspace root
├── rust-toolchain.toml           # MSRV pin
├── docker-compose.yml            # Base infrastructure + service definitions
├── docker-compose.override.yml   # Dev overrides (MailHog, hot reload)
├── docker-compose.prod.yml       # Production overrides (resource limits)
├── .env.example                  # Environment variable template
│
├── contracts/                    # Proto definitions (source of truth)
│   ├── identity.proto
│   ├── gl.proto
│   ├── ap.proto
│   ├── ar.proto
│   ├── procurement.proto
│   ├── reporting.proto
│   ├── budget.proto
│   ├── consolidation.proto
│   ├── fixed-asset.proto
│   ├── tax.proto
│   ├── workflow.proto
│   └── notification.proto
│
├── crates/
│   ├── proto/                    # Generated protobuf Rust types
│   ├── types/                    # Shared domain types (Decimal, Money, etc.)
│   ├── auth/                     # JWT validation, tenant resolution middleware
│   ├── db/                       # Database connection pool management, tenant routing
│   ├── audit/                    # Audit log middleware and repository
│   ├── error/                    # Unified error types and gRPC→HTTP mapping
│   ├── observability/            # tracing, OpenTelemetry, Prometheus setup
│   ├── pagination/               # Cursor-based pagination utilities
│   ├── testing/                  # Test helpers, mock factories, fixture loaders
│   └── export/                   # PDF/XLSX generation utilities
│
├── services/
│   ├── gateway/                  # API gateway (axum, REST→gRPC translation)
│   ├── identity/                 # Authentication, users, tenants
│   ├── gl/                       # General Ledger
│   ├── ap/                       # Accounts Payable
│   ├── ar/                       # Accounts Receivable
│   ├── procurement/              # Purchase requisitions & orders
│   ├── reporting/                # Reports & dashboards
│   ├── budget/                   # Budget management
│   ├── consolidation/            # Multi-org & currency
│   ├── fixed-asset/              # Fixed asset management
│   ├── tax/                      # Tax calculation & reporting
│   ├── workflow/                 # Approval workflows
│   └── notification/             # In-app notifications
│
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   └── src/
│       ├── components/           # Reusable UI components
│       ├── pages/                # Route-level page components
│       ├── hooks/                # TanStack Query hooks per domain
│       ├── api/                  # API client (axios + TanStack Query)
│       ├── types/                # TypeScript domain types
│       └── utils/                # Formatters, validators
│
└── specs/
    └── 001-oracle-fusion-erp-clone/
        ├── spec.md
        ├── plan.md
        ├── research.md
        ├── data-model.md
        ├── quickstart.md
        └── contracts/
```

## Service Endpoints

| Service | gRPC Port | REST via Gateway | Description |
|---------|-----------|------------------|-------------|
| Gateway | — | :8080 | API gateway (REST → gRPC) |
| Identity | :50001 | /api/v1/auth/*, /api/v1/users/*, /api/v1/tenants/* | Auth & user management |
| GL | :50002 | /api/v1/gl/* | Chart of accounts, journal entries, periods |
| AP | :50003 | /api/v1/ap/* | Vendors, invoices, payments |
| AR | :50004 | /api/v1/ar/* | Customers, invoices, receipts |
| Procurement | :50005 | /api/v1/procurement/* | Requisitions, POs, goods receipts |
| Reporting | :50006 | /api/v1/reports/*, /api/v1/dashboard/* | Reports & dashboards |
| Budget | :50007 | /api/v1/budgets/* | Budget management |
| Consolidation | :50008 | /api/v1/consolidation/* | Legal entities, FX rates, consolidation |
| Fixed Asset | :50009 | /api/v1/assets/* | Asset register & depreciation |
| Tax | :50010 | /api/v1/tax/* | Tax codes & calculation |
| Workflow | :50011 | /api/v1/workflows/* | Approval workflows |
| Notification | :50012 | /api/v1/notifications/* | In-app notifications |

## Environment Variables

```bash
# .env.example

# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=fusion
DATABASE_PASSWORD=fusion

# NATS
NATS_URL=nats://localhost:4222

# JWT
JWT_PRIVATE_KEY_PATH=./keys/private.pem
JWT_PUBLIC_KEY_PATH=./keys/public.pem
JWT_ACCESS_TOKEN_TTL_MINUTES=15
JWT_REFRESH_TOKEN_TTL_DAYS=7

# SMTP (for password reset)
SMTP_HOST=localhost
SMTP_PORT=1025  # MailHog in dev
SMTP_USER=
SMTP_PASSWORD=
SMTP_FROM=noreply@fusion-erp.local

# Observability
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
RUST_LOG=fusion=debug,tower_http=debug

# Identity service (platform DB)
PLATFORM_DATABASE_URL=postgres://fusion:fusion@localhost:5432/fusion_platform
```

## Database Setup

```bash
# Create the platform database (identity service)
createdb fusion_platform
sqlx migrate run --source services/identity/migrations --database-url "$PLATFORM_DATABASE_URL"

# Create tenant databases (repeat per service per tenant)
for service in gl ap ar procurement reporting budget consolidation fixed-asset tax workflow notification; do
  createdb "fusion_${service}_tenant_001"
  sqlx migrate run --source "services/${service}/migrations" \
    --database-url "postgres://fusion:fusion@localhost:5432/fusion_${service}_tenant_001"
done
```

## Running Tests

```bash
# All unit + integration tests
cargo nextest run

# Per-service tests
cargo nextest run -p fusion-gl

# Contract tests (validate proto conformance)
cargo nextest run -p fusion-proto

# E2E tests (requires running services)
cd frontend && npx playwright test
```

## Tenant Provisioning

```bash
# Create a new tenant via CLI
cargo run --bin fusion-cli tenants create \
  --name "Acme Corp" \
  --slug "acme" \
  --plan "standard"

# This creates:
# 1. Tenant record in fusion_platform
# 2. Per-service databases: fusion_{service}_acme
# 3. Default admin user (email sent for password setup)
```

## Frontend Development

```bash
cd frontend
npm install
npm run dev  # Starts Vite dev server on :3000, proxies API to :8080
```
