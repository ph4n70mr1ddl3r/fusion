# Implementation Plan: Oracle Fusion Cloud ERP Clone

**Branch**: `001-oracle-fusion-erp-clone` | **Date**: 2026-04-28 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-oracle-fusion-erp-clone/spec.md`

## Summary

Build a comprehensive cloud-based ERP system cloning Oracle Fusion Cloud ERP's core financial modules: General Ledger, Accounts Payable, Accounts Receivable, Procurement, Financial Reporting, Budgeting, Multi-Org/Multi-Currency, Fixed Assets, Tax Management, Workflow/Approvals, Role-Based Access Control, and Notifications.

**Technical approach**: Rust microservices architecture using `axum` (HTTP) and `tonic` (gRPC), PostgreSQL with database-per-tenant isolation, NATS JetStream for async event-driven communication, transactional outbox pattern for cross-service consistency, and a React 19 frontend with TanStack Query. Deployed via Docker Compose. All 13 domain services + API gateway communicate through versioned protobuf contracts. Tenant databases are routed via `moka` cache of `sqlx::PgPool` instances keyed by tenant ID resolved from JWT claims.

## Technical Context

**Language/Version**: Rust (latest stable, MSRV pinned in `rust-toolchain.toml`)
**Primary Dependencies**: `tokio` (async runtime), `axum` (HTTP), `tonic` + `prost` (gRPC), `sqlx` (compile-time checked PostgreSQL queries), `async-nats` (NATS JetStream), `tracing` + OpenTelemetry (observability), React 19 + TanStack Query (frontend)
**Storage**: PostgreSQL 16+ — database-per-tenant model; each service owns its per-tenant databases. `fusion_platform` database for identity/tenant registry. NATS JetStream for durable async messaging.
**Testing**: `cargo test` / `cargo nextest` (unit + integration), contract tests (proto conformance), Playwright (E2E)
**Target Platform**: Linux server (Docker Compose), modern web browsers (Chrome, Firefox, Safari, Edge)
**Project Type**: Web application (Rust microservices backend + React SPA frontend)
**Performance Goals**: 100 concurrent users × 5 req/min; 95th percentile < 3s for transactions/reports; reports generate < 10s for 100K transactions; dashboard KPIs ≤ 60s staleness
**Constraints**: Period close for 1K+ assets and 10K+ journal entries within 5 minutes; approval notification persisted within 5 seconds; 100% audit trail accuracy; currency calculations accurate to 2 decimal places
**Scale/Scope**: 100 concurrent users, ~50 screens/modules, 13 microservices + gateway, 100K+ posted transactions per tenant

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. Test-Driven Development | ✅ PASS | All artifacts mandate TDD; quickstart.md defines test categories (unit, integration, contract, E2E); plan phases require tests before implementation |
| II. Microservices Architecture | ✅ PASS | 13 independently deployable services + gateway, each owning its data; bounded contexts aligned with business domains (GL, AP, AR, Procurement, etc.); async NATS + gRPC communication |
| III. Rust-First Development | ✅ PASS | All backend services in Rust; `tokio` runtime; `axum`/`tonic` servers; `sqlx` for DB; zero-warning policy (`RUSTFLAGS="-D warnings"`); `cargo audit` in CI |
| IV. API-First Design with Versioned Contracts | ✅ PASS | All 15 proto contracts defined in `contracts/` directory; semantic versioning; generated code committed; contract tests validate conformance |
| V. Observability and Reliability | ✅ PASS | `tracing` crate mandatory; health/readiness endpoints per service; OpenTelemetry distributed tracing; Prometheus metrics; circuit breakers for inter-service calls |
| VI. Workspace and Crate Organization | ✅ PASS | Cargo workspace with services under `services/<name>/`, shared crates under `crates/<name>/`, proto types under `crates/proto/`; DAG dependency graph |
| VII. Simplicity and YAGNI | ✅ PASS | Docker Compose (not K8s); polling notifications (not WebSocket); manual FX rates (not API feeds); sequential report aggregation (not data warehouse); event-driven only where needed (outbox pattern) |

**Pre-Phase 0 Gate**: ✅ ALL PRINCIPLES PASS — no violations to justify.

## Project Structure

### Documentation (this feature)

```text
specs/001-oracle-fusion-erp-clone/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output — all technical unknowns resolved
├── data-model.md        # Phase 1 output — complete entity definitions per service
├── quickstart.md        # Phase 1 output — developer onboarding guide
├── contracts/           # Phase 1 output — 15 protobuf service contracts
│   ├── common.proto
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
│   ├── notification.proto
│   └── asset.proto
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

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
│   ├── gateway/                  # API gateway (axum, REST→gRPC translation, rate limiting)
│   ├── identity/                 # Authentication, users, tenants, departments
│   ├── gl/                       # General Ledger (CoA, journal entries, periods, trial balance)
│   ├── ap/                       # Accounts Payable (vendors, invoices, payments)
│   ├── ar/                       # Accounts Receivable (customers, invoices, receipts)
│   ├── procurement/              # Purchase requisitions, POs, goods receipts
│   ├── reporting/                # Reports, dashboards, report builder
│   ├── budget/                   # Budget management, variance tracking
│   ├── consolidation/            # Multi-org, legal entities, FX rates, consolidation
│   ├── fixed-asset/              # Asset register, depreciation, disposal
│   ├── tax/                      # Tax codes, calculation, reporting
│   ├── workflow/                 # Approval workflows, escalation, delegation
│   └── notification/             # In-app notifications, SMTP for password reset
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
└── specs/                        # Feature specifications and plans
```

**Structure Decision**: Web application architecture (Option 2) — Rust microservices backend under `services/` and `crates/`, React SPA frontend under `frontend/`. Each service is an independent binary crate within the Cargo workspace. Shared concerns (auth, DB routing, observability) are extracted into library crates under `crates/` to avoid code duplication while maintaining service independence.

## Complexity Tracking

> No violations detected. All design decisions align with Constitution Principle VII (Simplicity and YAGNI).

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| — | — | — |
