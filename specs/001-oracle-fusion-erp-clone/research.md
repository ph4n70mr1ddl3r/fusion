# Research: Oracle Fusion Cloud ERP Clone

**Date**: 2026-04-25 | **Branch**: `001-oracle-fusion-erp-clone`

## Research Tasks

### R1: Frontend Framework Selection

**Decision**: React 19 + TypeScript 5.x with Vite

**Rationale**:
- Largest ecosystem of enterprise UI component libraries (AG Grid, MUI, Ant Design)
- TypeScript provides type safety for complex financial data structures
- Vite offers fast HMR and build times critical for developer productivity
- TanStack Query handles server state management (caching, background refetch, optimistic updates)
- Recharts provides charting for dashboards and financial visualizations
- Constitution mandates Rust for backend only; React/TypeScript is the industry standard for enterprise SPA frontends

**Alternatives Considered**:
- *Vue 3 + TypeScript*: Smaller enterprise ecosystem; less hiring pool
- *Svelte/SvelteKit*: Immature enterprise component libraries
- *Leptos (Rust WASM)*: Impressive technology but insufficient enterprise UI library ecosystem for complex data grids, date pickers, and form controls needed in an ERP

### R2: Message Broker Selection (NATS vs RabbitMQ)

**Decision**: NATS with JetStream

**Rationale**:
- Lightweight, written in Go, single binary deployment
- JetStream provides persistent streams with exactly-once semantics
- Lower operational overhead than RabbitMQ for our microservices topology
- Native Rust client (`async-nats`) with excellent tokio integration
- Subject-based addressing maps naturally to our domain events (e.g., `erp.gl.journal.posted`)
- At-least-once delivery with deduplication support for financial transaction events

**Alternatives Considered**:
- *RabbitMQ*: More mature but heavier; complex clustering; better for complex routing we don't need
- *Apache Kafka*: Overkill for our scale; operational complexity unjustified for <100 concurrent users
- *Redis Streams*: Acceptable but lacks the durability guarantees needed for financial events

### R3: Multi-Tenancy Strategy

**Decision**: Schema-per-tenant within shared database instances (hybrid approach)

**Rationale**:
- Full database-per-tenant isolation for P1 launch (up to ~50 tenants)
- Each service manages tenant schemas independently via sqlx migrations
- Tenant ID propagated via JWT claims and enforced at repository layer
- Migration to dedicated instances for large tenants is a future optimization
- Aligns with constitution: "each service owns its data store" — tenant schemas are isolated per service

**Alternatives Considered**:
- *Row-level tenant ID*: Simpler but requires disciplined query filtering everywhere; single bug exposes cross-tenant data
- *Database-per-tenant*: Maximum isolation but PostgreSQL connection pool explosion at scale; defer to later
- *Separate deployments per tenant*: Operational nightmare for updates; rejected

**Implementation Notes**:
- `crates/db` provides `TenantPool` that resolves the correct schema per request
- All SQL queries use `SET search_path TO tenant_{id}` on connection checkout
- Middleware in each service extracts tenant ID from JWT and injects into request context

### R4: Authentication Mechanism

**Decision**: JWT (RS256) with short-lived access tokens (15 min) and long-lived refresh tokens (7 days)

**Rationale**:
- Stateless authentication fits microservices architecture — no shared session store
- RS256 (asymmetric) allows services to verify tokens using public key only (no secret sharing)
- Identity service holds the private key; all other services verify with public key
- Refresh tokens stored in HTTP-only cookies; access tokens in memory (React)
- Supports future SSO integration (OIDC bridge in identity service)

**Alternatives Considered**:
- *Session-based*: Requires shared session store (Redis); tighter coupling between services
- *Opaque tokens*: Requires identity service call on every request; adds latency
- *mTLS*: Appropriate for inter-service but not user-facing auth

### R5: File/Document Storage

**Decision**: Local filesystem with S3-compatible API abstraction (MinIO for dev, S3 for prod)

**Rationale**:
- ERP requires attachment support (invoice PDFs, receipt scans, budget spreadsheets)
- S3-compatible API is the industry standard; MinIO provides local dev parity
- `crates/storage` shared library abstracts the S3 API
- No special database needed; metadata (filename, size, content_type) stored in PostgreSQL with the owning entity

**Alternatives Considered**:
- *PostgreSQL BYTEA*: DB bloat; poor backup characteristics for large files
- *NFS/shared filesystem*: Doesn't scale in containerized environments
- *Database-only*: No attachments support in v1 → rejected; attachments are a core ERP need

### R6: Caching Strategy

**Decision**: Application-level caching with `moka` crate, no external cache initially

**Rationale**:
- Most ERP reads are relatively static within a period (chart of accounts, vendor masters, customer masters)
- `moka` provides a high-performance concurrent cache compatible with tokio
- Cache invalidation via NATS events (e.g., `erp.gl.account.updated` → cache invalidation)
- No Redis dependency needed at our scale (100 concurrent users)
- Add Redis later if performance testing reveals a need (YAGNI — Principle VII)

**Alternatives Considered**:
- *Redis*: Standard choice but premature at 100 concurrent users; adds operational burden
- *No caching*: Viable but repeated GL lookups during posting would be wasteful
- *HTTP cache headers*: Useful for frontend but insufficient for inter-service caching

### R7: Inter-Service Transaction Pattern

**Decision**: Saga pattern (choreography-based) with compensating transactions

**Rationale**:
- Financial transactions often span services (e.g., AP invoice → GL posting)
- Distributed transactions (2PC) are an anti-pattern in microservices
- Saga choreography via NATS: each service publishes events; downstream services react
- Compensating transactions for failure recovery (e.g., reverse GL entry if payment fails)
- Saga state tracked in originating service's database (saga log table)

**Alternatives Considered**:
- *2PC / XA transactions*: Couples services tightly; failure propagation is catastrophic
- *Orchestration saga*: Central coordinator becomes bottleneck; choreography is simpler
- *Eventual consistency without sagas*: Works for reads but financial transactions need defined recovery paths

### R8: Frontend State Management

**Decision**: Zustand for client state + TanStack Query for server state

**Rationale**:
- Clear separation: Zustand manages UI state (sidebar open, selected period, filters); TanStack Query manages server data (accounts, invoices, reports)
- TanStack Query's cache invalidation, background refetch, and optimistic updates handle ERP data patterns naturally
- Zustand is minimal (1KB), TypeScript-first, and avoids Redux boilerplate
- No need for complex middleware or devtools overhead at this scale

**Alternatives Considered**:
- *Redux Toolkit*: Overkill for our needs; TanStack Query eliminates most server state concerns
- *Jotai/Recoil*: Atomic state management is unnecessary for our mostly-module-isolated UI
- *React Context only*: Insufficient for cross-module state (e.g., current period, tenant context)

### R9: Report Generation Engine

**Decision**: Server-side PDF generation via `genpdf` (Rust) + HTML-to-PDF fallback

**Rationale**:
- `genpdf` crate provides basic PDF generation suitable for structured financial reports
- For complex layouts (balance sheets, income statements with styling), use HTML templates rendered to PDF via a headless browser service or `wkhtmltopdf`
- Spreadsheet export via `rust_xlsxwriter` crate
- Reports generated on-demand with async job queue for large datasets

**Alternatives Considered**:
- *Client-side PDF (jsPDF)*: Inconsistent rendering; heavy browser resource usage for large reports
- *External reporting service (Jasper, BIRT)*: Adds non-Rust dependency; operational complexity
- *Pure Rust PDF only*: Insufficient formatting capabilities for professional financial reports

### R10: Database Migration Strategy

**Decision**: sqlx-cli with versioned SQL migrations per service

**Rationale**:
- sqlx-cli is the standard Rust/PostgreSQL migration tool
- Each service has its own `migrations/` directory
- Migrations are run on service startup in dev, explicitly in CI/prod
- Tenant schema migrations applied programmatically via `crates/db` helper

**Alternatives Considered**:
- *Diesel migrations*: Tied to Diesel ORM; we use sqlx
- *Custom migration runner*: Unnecessary complexity; sqlx-cli is battle-tested
- *Single shared migration repo*: Violates service independence (Principle II)
