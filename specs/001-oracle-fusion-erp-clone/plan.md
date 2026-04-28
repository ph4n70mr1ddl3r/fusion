# Implementation Plan: Oracle Fusion Cloud ERP Clone

**Branch**: `001-oracle-fusion-erp-clone` | **Date**: 2026-04-26 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-oracle-fusion-erp-clone/spec.md`

## Summary

Build a comprehensive cloud-based ERP system cloning Oracle Fusion Cloud ERP's financial modules, implemented as a Rust microservices architecture with a React/TypeScript web frontend. The system covers General Ledger, Accounts Payable, Accounts Receivable, Procurement, Financial Reporting, Budgeting, Multi-Org/Multi-Currency, Access Control, Workflow Approvals, Tax Management, and Fixed Asset Management across 12 domain microservices plus an API gateway.

**Technical approach**: Each domain service owns its PostgreSQL database and communicates via gRPC (synchronous) and NATS (asynchronous events). The API gateway (axum) serves as a Backend-for-Frontend, translating REST calls into gRPC. All services follow strict TDD, API-first contract design with Protocol Buffers, and full observability instrumentation (tracing, OpenTelemetry, Prometheus).

## Technical Context

**Language/Version**: Rust (latest stable, MSRV pinned in `rust-toolchain.toml`)
**Primary Dependencies**:
- Backend: axum 0.8, tonic 0.12, prost 0.13, tokio 1.x, sqlx 0.8 (PostgreSQL, compile-time checked), serde 1.x, tracing 0.1, uuid 1.x, chrono 0.4
- Frontend: React 19, TypeScript 5.x, TanStack Query, Recharts, TanStack Table
- Additional: lettre 0.1 (SMTP email), genpdf 0.2 (PDF generation), rust_xlsxwriter 0.8 (XLSX export), moka 0.12 (caching), criterion 0.5 (benchmarking)
**Storage**: PostgreSQL 16 (one database per service per tenant for strict isolation)
**Message Broker**: NATS 2 with JetStream (selected for lightweight footprint, native JetStream durable messaging, and first-class Rust client via `async-nats`)
**Testing**: cargo test / cargo nextest (unit + integration), tonic mock-based contract tests, Playwright (E2E)
**Target Platform**: Linux server (Docker containers), modern web browsers (Chrome, Firefox, Safari, Edge)
**Project Type**: Web service (microservices) + single-page web application
**Performance Goals**: 100 concurrent users, <3s response time, <10s report generation for 100K transactions, approval notifications within 5s
**Constraints**: Multi-tenant data isolation (database-per-tenant), 100% audit trail integrity, no manual GL corrections for currency, 7-year data retention
**Scale/Scope**: 12 domain services + 1 gateway + 14 shared crates + 1 frontend SPA; ~30 endpoints per service; 18 key entities (see spec.md Key Entities section)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principle | Status | Notes |
|---|-----------|--------|-------|
| I | Test-Driven Development | ✅ PASS | Red-Green-Refactor enforced in task breakdown; every service starts with failing tests |
| II | Microservices Architecture | ✅ PASS | 12 domain services + gateway; each owns its DB; gRPC + NATS inter-service with versioned contracts |
| III | Rust-First Development | ✅ PASS | All backend services in Rust (axum + tonic); frontend in React/TypeScript (constitution does not mandate Rust for UI) |
| IV | API-First with Versioned Contracts | ✅ PASS | Proto files defined in `contracts/` and `proto/` before implementation; semantically versioned |
| V | Observability and Reliability | ✅ PASS | All services: `/health`, `/ready`, `tracing` (OpenTelemetry), Prometheus metrics, circuit breakers via `tower` |
| VI | Workspace and Crate Organization | ✅ PASS | Cargo workspace: `services/<name>/`, `crates/<name>/`; no circular deps; DAG enforced |
| VII | Simplicity and YAGNI | ✅ PASS | Build P1 services first; P2/P3 incrementally; no speculative abstractions |

**Pre-Phase 0 Gate Result**: ✅ ALL PASS

## Project Structure

### Documentation (this feature)

```text
specs/001-oracle-fusion-erp-clone/
├── plan.md              # This file (/speckit.plan command output)
├── spec.md              # Feature specification
├── research.md          # Phase 0 output - technical research & decisions
├── data-model.md        # Phase 1 output - entity definitions
├── quickstart.md        # Phase 1 output - developer setup guide
├── contracts/           # Phase 1 output - design-time gRPC service contracts (source of truth for review)
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
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
fusion/
├── Cargo.toml                    # Workspace root
├── rust-toolchain.toml           # MSRV pin
├── docker-compose.yml            # Base infrastructure + service definitions
├── docker-compose.override.yml   # Dev overrides (MailHog, debug ports)
├── docker-compose.prod.yml       # Production overrides (resource limits)
├── .env.example                  # Environment variable template
├── Makefile                      # Common commands (build, test, migrate, lint)
│
├── proto/                        # Proto definitions (build-time source of truth, versioned)
│   └── */v1/*.proto               # Copied from specs/.../contracts/ via T005; compiled by crates/proto/
│
├── crates/
│   ├── proto/                    # Auto-generated protobuf Rust types (build.rs from contracts/)
│   ├── types/                    # Shared domain types (Decimal wrapper, Money, Currency, enums)
│   ├── auth/                     # JWT validation middleware, tenant ID extraction, RBAC guard
│   ├── db/                       # PgPool factory, tenant-aware routing (moka cache), migration runner
│   ├── audit/                    # Audit log middleware (write to per-service audit_log table)
│   ├── error/                    # Unified error types, gRPC→HTTP status mapping, error responses
│   ├── observability/            # tracing subscribers, OTEL pipeline, Prometheus metrics registry
│   ├── pagination/               # Cursor-based pagination (encode/decode, page tokens)
│   ├── testing/                  # Test DB fixtures, mock gRPC clients, test data factories
│   ├── export/                   # PDF (genpdf) and XLSX (rust_xlsxwriter) generation
│   ├── messaging/                # NATS JetStream publisher/subscriber, outbox pattern infrastructure
│   ├── resilience/               # Circuit breaker, retry policies, gRPC client interceptor
│   ├── currency/                 # ExchangeRateClient trait, PassthroughExchangeRateClient (single-currency default)
│   └── document_numbering/       # Sequential document numbering per-tenant/type/year (FR-044)
│
├── services/
│   ├── gateway/                  # API gateway (axum): REST→gRPC translation, auth, rate limiting
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       ├── routes/           # REST route handlers per domain
│   │       ├── middleware/       # Auth, tenant resolution, rate limiting
│   │       └── error.rs
│   │
│   ├── identity/                 # Auth, users, tenants, RBAC
│   │   ├── Cargo.toml
│   │   ├── migrations/           # sqlx migrations (fusion_platform DB)
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs        # Tonic service implementation
│   │       ├── repository.rs     # sqlx queries
│   │       └── models.rs         # Domain models
│   │
│   ├── gl/                       # General Ledger
│   │   ├── Cargo.toml
│   │   ├── migrations/
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs
│   │       ├── repository.rs
│   │       ├── models.rs
│   │       └── period_close.rs   # Period close orchestration
│   │
│   ├── ap/                       # Accounts Payable
│   │   ├── Cargo.toml
│   │   ├── migrations/
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs
│   │       ├── repository.rs
│   │       └── matching.rs       # Three-way matching logic
│   │
│   ├── ar/                       # Accounts Receivable
│   │   ├── Cargo.toml
│   │   ├── migrations/
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs
│   │       ├── repository.rs
│   │       └── credit.rs         # Credit limit enforcement
│   │
│   ├── procurement/              # Purchase Requisitions & Orders
│   │   ├── Cargo.toml
│   │   ├── migrations/
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs
│   │       └── repository.rs
│   │
│   ├── reporting/                # Reports, Dashboards, Export
│   │   ├── Cargo.toml
│   │   ├── migrations/           # saved_reports table
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs
│   │       ├── aggregator.rs     # Cross-service gRPC aggregation
│   │       └── export.rs         # PDF/XLSX generation
│   │
│   ├── budget/                   # Budget Management
│   │   ├── Cargo.toml
│   │   ├── migrations/
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs
│   │       ├── repository.rs
│   │       └── import.rs         # CSV/XLSX budget line import
│   │
│   ├── consolidation/            # Multi-Org & Currency
│   │   ├── Cargo.toml
│   │   ├── migrations/
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs
│   │       ├── revaluation.rs    # Currency revaluation logic
│   │       └── eliminations.rs   # Intercompany elimination
│   │
│   ├── asset/                    # Fixed Asset Management
│   │   ├── Cargo.toml
│   │   ├── migrations/
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs
│   │       ├── depreciation.rs   # Straight-line & declining balance
│   │       └── disposal.rs
│   │
│   ├── tax/                      # Tax Calculation & Reporting
│   │   ├── Cargo.toml
│   │   ├── migrations/
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs
│   │       └── calculator.rs
│   │
│   ├── workflow/                 # Approval Workflows
│   │   ├── Cargo.toml
│   │   ├── migrations/
│   │   └── src/
│   │       ├── main.rs
│   │       ├── service.rs
│   │       ├── engine.rs         # Workflow step execution & escalation
│   │       └── escalation.rs     # Timeout monitoring
│   │
│   └── notification/             # In-App Notifications
│       ├── Cargo.toml
│       ├── migrations/
│       └── src/
│           ├── main.rs
│           └── service.rs
│
└── web/
    ├── package.json
    ├── tsconfig.json
    ├── vite.config.ts
    ├── index.html
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── routes.tsx             # React Router config
        ├── api/
        │   ├── client.ts         # Axios instance with JWT interceptor
        │   └── hooks/            # TanStack Query hooks per domain
        ├── components/
        │   ├── layout/           # Shell, sidebar, header, notifications bell
        │   ├── ui/               # Design system primitives (Button, Input, Table, etc.)
        │   └── shared/           # Domain-aware shared components
        ├── pages/
        │   ├── auth/             # Login, password reset
        │   ├── gl/               # Chart of accounts, journal entries, periods
        │   ├── ap/               # Vendors, invoices, payments
        │   ├── ar/               # Customers, invoices, receipts
        │   ├── procurement/      # Requisitions, purchase orders
        │   ├── reports/          # Standard reports, custom report builder, dashboard
        │   ├── budgets/          # Budget management, variance analysis
        │   ├── assets/           # Fixed asset register
        │   ├── admin/            # Users, roles, workflow config, tax codes
        │   └── notifications/    # Notification center
        ├── types/                 # TypeScript interfaces matching proto models
        └── utils/                 # Formatters, validators, constants
```

**Structure Decision**: Web application (Option 2) with Rust microservices backend and React/TypeScript frontend. The workspace structure follows Constitution Principle VI with `services/`, `crates/`, and `web/` directories.

**Proto workflow**: Design-time proto files live in `specs/001-oracle-fusion-erp-clone/contracts/` (for spec review and version tracking). Task T005 copies them to the repo-root `proto/` directory with versioned paths (`proto/common/v1/types.proto`, etc.). The `crates/proto/` build.rs compiles from the repo-root `proto/` directory. The specs-level contracts are the authoritative design; the repo-root proto files are the authoritative build source.

## Implementation Phases

### Phase 0: Research ✅ COMPLETE

**Output**: [research.md](./research.md)

All 13 technical unknowns resolved:
1. Tenant DB routing strategy (moka cache of PgPool per tenant)
2. JWT RS256 with JWKS for multi-service auth
3. gRPC for sync, NATS for async communication
4. Correlation ID chain for financial audit trail
5. Reporting service as gRPC aggregator
6. Tenant provisioning via CLI/API
7. TanStack Query + polling for frontend
8. Optimistic locking via version column
9. Sequential period close with resumability
10. lettre SMTP + MailHog (dev)
11. JSONB segments on chart of accounts
12. Docker Compose with dev/prod overrides
13. genpdf + rust_xlsxwriter for report export

### Phase 1: Design & Contracts ✅ COMPLETE

**Output**: [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

- **data-model.md**: 18 key entities with full field definitions, types, constraints, validation rules, state transitions, and indexes. Grouped by owning service. Cross-cutting audit_log and tenant routing documented.
- **contracts/**: 12 Protocol Buffer v3 service definitions covering all gRPC interfaces: identity, gl, ap, ar, procurement, reporting, budget, consolidation, fixed-asset, tax, workflow, notification.
- **quickstart.md**: Developer onboarding guide with project structure, service ports, environment variables, database setup, test commands, and tenant provisioning.

### Phase 2: Implementation (via /speckit.tasks)

Task decomposition follows a layered build-up approach:

**Build Order** (dependency-aware):
1. **Shared crates** (proto, types, error, db, auth, observability, pagination, audit, testing, export, messaging, resilience, currency) — no service dependencies
2. **Identity service** — foundational; all services depend on auth
3. **API Gateway** — depends on identity for auth middleware; routes to all services
4. **GL service** — core financial module; other financial services post to GL
5. **Workflow service** — approval engine used by AP, AR, Procurement, GL, Budget
6. **Notification service** — used by workflow for approval notifications
7. **AP service** — depends on GL (posting) + Workflow (approvals) + Tax (calculation)
8. **AR service** — depends on GL (posting) + Workflow (approvals) + Tax (calculation)
9. **Procurement service** — depends on AP (vendor ref, three-way match) + Workflow
10. **Tax service** — depends on AP/AR (tax events from invoices); consumed during invoice processing
11. **Fixed Asset service** — depends on GL (posting) + Consolidation (entity ref)
12. **Budget service** — depends on GL (actuals) + Workflow (approvals)
13. **Consolidation service** — depends on GL (period close, exchange rates)
14. **Reporting service** — aggregator; depends on GL, AP, AR, Budget
15. **Web frontend** — depends on all backend services via gateway

### Phase 3: Integration & E2E Testing (via /speckit.tasks)

Cross-service integration tests covering:
- Procure-to-pay (requisition → PO → goods receipt → invoice → payment → GL)
- Order-to-cash (customer → invoice → receipt → GL)
- Period close (depreciation → revaluation → consolidation → close)
- Approval workflows across document types
- Multi-tenant isolation verification
- RBAC scope enforcement (OWN, DEPARTMENT, ALL)

## Post-Phase 1 Constitution Re-Check

| # | Principle | Status | Notes |
|---|-----------|--------|-------|
| I | TDD | ✅ PASS | Task breakdown will enforce Red-Green-Refactor per task |
| II | Microservices | ✅ PASS | 12 services + gateway; each owns per-tenant databases; gRPC + NATS |
| III | Rust-First | ✅ PASS | All backend Rust; frontend React/TS (per constitution) |
| IV | API-First | ✅ PASS | 12 proto contracts defined before implementation |
| V | Observability | ✅ PASS | All services have health/ready endpoints, tracing, metrics |
| VI | Workspace | ✅ PASS | Cargo workspace with services/ and crates/ directories; DAG deps |
| VII | Simplicity | ✅ PASS | No speculative abstractions; YAGNI applied to feature selection |

**Post-Phase 1 Gate Result**: ✅ ALL PASS

## Complexity Tracking

> No constitution violations detected. No entries required.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| *(none)* | | |

## Key Architectural Decisions

### Database-Per-Tenant Connection Routing

Each service maintains a `moka::Cache<UUID, PgPool>` mapping tenant IDs to database connection pools. Pools are created lazily on first request and evicted after idle timeout. The tenant ID is extracted from the JWT by the API gateway and propagated via gRPC metadata.

### Inter-Service Communication

- **gRPC (synchronous)**: Used when the caller needs an immediate response — journal posting, report data aggregation, tax calculation. Defined in `contracts/*.proto`.
- **NATS JetStream (asynchronous)**: Used for fire-and-forget events — "invoice posted", "payment received", "approval needed". Events are durable (at-least-once delivery). Subject naming: `fusion.{service}.{event_type}` (e.g., `fusion.ap.invoice_posted`).

### API Gateway Pattern

The gateway is a Backend-for-Frontend (BFF) that:
1. Terminates TLS
2. Validates JWT tokens (public key from identity service JWKS)
3. Extracts tenant_id from token claims
4. Resolves tenant's database connection info from identity service
5. Passes tenant_id via gRPC metadata to downstream services
6. Enforces per-tenant rate limiting (1000 req/min)
7. Translates REST JSON → gRPC protobuf

### Financial Audit Trail

Two distinct audit systems:
1. **Financial Audit Trail (FR-006)**: GL service's `journal_entry_source` table maps source documents to journal entries via `correlation_id`. Enables tracing any posted transaction from source to GL to report.
2. **Security Audit Log (FR-032)**: Per-service `audit_log` table recording all CRUD operations with user, timestamp, changes. Retained for 7 years. Partitioned by month for efficient purge.
