# Research: Oracle Fusion Cloud ERP Clone

**Branch**: `001-oracle-fusion-erp-clone` | **Date**: 2026-04-26 | **Spec**: [spec.md](./spec.md)

## Overview

This document captures research decisions, rationale, and alternatives evaluated for each technical unknown identified during plan creation. All findings feed into the implementation plan and data model design.

---

## 1. Database-Per-Tenant Connection Routing Strategy

**Decision**: Use a connection pool manager at the API gateway/service layer that resolves the tenant's database connection string from a central tenant registry, then routes each request to the appropriate per-tenant PostgreSQL database.

**Rationale**: 
- The spec (as clarified) mandates database-per-tenant isolation.
- `sqlx` supports dynamic connection strings at runtime via `PgPoolOptions::connect()`.
- A tenant registry table (in the identity service's shared `fusion_platform` database) maps `tenant_id` → `{host, port, database_name, credentials_ref}`.
- Each domain service maintains a pool of `PgPool` instances keyed by `tenant_id`, with lazy creation and idle eviction (e.g., via `moka` cache).
- This avoids creating `N_tenants × N_services` pools at startup; pools are created on first request per tenant per service.

**Alternatives Considered**:
- *PgBouncer per-tenant*: Adds infrastructure complexity; connection routing would still need tenant-aware middleware.
- *Schema-per-tenant with SET search_path*: Rejected as spec clarifies database-per-tenant.
- *Hardcoded pool per tenant in config*: Does not scale for dynamic tenant provisioning.

---

## 2. Multi-Service Authentication & JWT Validation

**Decision**: The identity service issues JWT RS256 tokens containing `tenant_id`, `user_id`, `roles[]`, and `department_id` claims. All other services validate tokens using the public key (fetched from identity service's JWKS endpoint at startup and refreshed every hour). No shared database needed for auth.

**Rationale**:
- RS256 (asymmetric) allows services to validate tokens without sharing the private key.
- `tenant_id` in the JWT is extracted by the API gateway and used for database routing.
- `roles[]` and `department_id` claims enable scope-based access control (FR-031) without additional service calls per request.
- Token lifetime: access token 15 minutes, refresh token 7 days.

**Alternatives Considered**:
- *Shared auth database*: Couples services; violates microservice independence.
- *API gateway-only validation*: Services behind the gateway would have no auth context; compromised internal network = no protection.
- *HMAC (HS256) symmetric*: All services would need the secret key; greater blast radius on key compromise.

---

## 3. Inter-Service Communication Patterns (gRPC vs NATS)

**Decision**: 
- **Synchronous (request-response)**: gRPC via `tonic` for queries where the caller needs an immediate response (e.g., GL service calling identity service to validate user, report builder aggregating data across services).
- **Asynchronous (event-driven)**: NATS JetStream for fire-and-forget events where eventual consistency is acceptable (e.g., "invoice posted" → GL creates journal entry, "payment received" → AR updates customer balance).

**Rationale**:
- Financial transactions requiring immediate consistency (e.g., balanced journal entry + GL posting) use gRPC.
- Event-driven notifications, audit trail propagation, and cross-module side effects use NATS for decoupling and reliability.
- NATS JetStream provides durable messaging with at-least-once delivery, replay, and consumer groups — eliminating the need for a heavier broker like Kafka.

**Alternatives Considered**:
- *All gRPC*: Tight coupling; cascading failures if any service is down.
- *Kafka*: Overkill for the scale (100 concurrent users); heavier operational footprint.
- *RabbitMQ*: Viable but NATS has a first-class Rust async client (`async-nats`) and simpler ops.

---

## 4. Financial Audit Trail Architecture

**Decision**: Each service maintains its own `audit_events` table in the tenant's database. The financial audit trail (FR-006) is implemented as a correlation chain: each transaction carries a `correlation_id` that propagates across services via gRPC metadata or NATS message headers. The audit trail is assembled by querying the GL service's `journal_entry_source` table, which maps `source_document_type + source_document_id → journal_entry_id`.

**Rationale**:
- The GL service is the authoritative source for financial audit trails since all financial transactions ultimately post to the GL.
- The security audit log (FR-032) lives in each service independently and captures all CRUD operations.
- `correlation_id` enables end-to-end tracing across service boundaries without a centralized audit database.

**Alternatives Considered**:
- *Centralized audit service*: Single point of failure; adds latency to every transaction.
- *Event sourcing*: Requires rebuilding state from events; over-engineering for v1 given the spec's CRUD-focused requirements.

---

## 5. Report Builder Data Aggregation Across Services

**Decision**: The reporting service acts as an aggregator. For custom reports (FR-022), it makes sequential gRPC calls to source services (GL, AP, AR, Budget) with the user's query parameters. Each source service returns a `protobuf` structured dataset. The reporting service performs in-memory aggregation (SUM, COUNT, AVERAGE) and returns the result. If any source service is unavailable, partial results are returned with a warning (per spec FR-022).

**Rationale**:
- The spec explicitly states this approach in FR-022.
- Sequential calls (vs. fan-out) simplify error handling and avoid distributed transaction complexity.
- In-memory aggregation is sufficient for the scale (100K transactions → fits in memory after pre-filtering at source).

**Alternatives Considered**:
- *Materialized views across services*: Violates service data ownership.
- *Data warehouse / ETL pipeline*: Over-engineering for v1 scale.
- *GraphQL federation*: Adds significant infrastructure complexity; not aligned with the Rust/gRPC stack.

---

## 6. Tenant Provisioning Workflow

**Decision**: A `fusion_platform` database (owned by the identity service) stores the tenant registry. Tenant provisioning is a CLI command or admin API endpoint that:
1. Creates the tenant record in `fusion_platform.tenants`.
2. For each domain service, creates a new PostgreSQL database named `{service}_{tenant_id}`.
3. Runs schema migrations against each new database.
4. Creates a default admin user for the tenant.

**Rationale**:
- Database-per-tenant requires provisioning databases per service per tenant.
- Migrations are managed per-service using `sqlx-cli` with per-tenant targeting.
- The `fusion_platform` database is the only shared (non-per-tenant) database.

**Alternatives Considered**:
- *Manual provisioning scripts*: Error-prone; no audit trail.
- *Kubernetes Operator*: Deferred with K8s (post-v1).

---

## 7. Frontend State Management & Real-Time Updates

**Decision**: React 19 with TanStack Query for server state (API data caching, background refetching, optimistic updates). The dashboard (FR-021) auto-refreshes every 60 seconds using TanStack Query's `refetchInterval`. Notifications use polling every 30 seconds (matching the in-app polling approach clarified for FR-035). No WebSocket infrastructure in v1.

**Rationale**:
- TanStack Query eliminates the need for a global client state store (Redux, Zustand) for server data.
- 60-second polling for dashboard KPIs matches the spec's "no older than 60 seconds" requirement.
- Keeps the frontend architecture simple; WebSocket can be added later without changing the data layer.

**Alternatives Considered**:
- *Redux + RTK Query*: Heavier; TanStack Query is more focused on server state.
- *Server-Sent Events (SSE)*: Viable for notifications but adds complexity for minimal v1 benefit.
- *WebSocket everywhere*: Over-engineering for 100 concurrent users; spec says polling is fine.

---

## 8. Optimistic Locking for Concurrent Edits

**Decision**: All transactional entities include a `version INT NOT NULL DEFAULT 1` column. On update, the query includes `WHERE id = $1 AND version = $2`. If `affected_rows == 0`, the service returns a conflict error (409). The client must refresh and retry.

**Rationale**:
- Spec edge case #4 explicitly requires this behavior.
- Simple to implement with `sqlx`; no need for distributed locks.
- `version` column doubles as a idempotency check for retry scenarios.

**Alternatives Considered**:
- *Last-write-wins*: Loses data; violates spec.
- *Distributed lock (Redis)*: Unnecessary at this scale; adds infrastructure dependency.

---

## 9. Period-End Processing Orchestration

**Decision**: Period-end processing (depreciation, currency revaluation, consolidation) is orchestrated by a dedicated `PeriodCloseService` within the GL service. It executes steps sequentially in a defined order: (1) verify no pending approvals, (2) run depreciation (calls fixed asset service), (3) run currency revaluation (calls multi-currency service), (4) run consolidation if applicable, (5) close the period. Each step records its status. If any step fails, the process halts and can be resumed from the last successful step.

**Rationale**:
- Sequential execution avoids race conditions during period close.
- Resumability is critical — period close is a long-running operation that shouldn't restart from scratch on failure.
- Keeping the orchestrator in the GL service avoids adding another microservice.

**Alternatives Considered**:
- *Saga pattern with NATS*: Over-engineering for a sequential, deterministic process.
- *Temporal/Workflow engine*: Adds significant infrastructure dependency for a single use case.

---

## 10. Password Reset & Email Infrastructure

**Decision**: FR-033a requires email-based password reset. Use `lettre` crate with SMTP transport. For development, use a local MailHog container. For production, configure SMTP credentials via environment variables pointing to a transactional email service (e.g., Amazon SES, SendGrid). The identity service sends reset emails with a time-limited token (default 1 hour).

**Rationale**:
- `lettre` is the standard Rust SMTP library; well-maintained and async-compatible.
- MailHog provides a zero-config local email testing environment.
- SMTP-based approach is provider-agnostic; no vendor lock-in.

**Alternatives Considered**:
- *Skip email in v1*: FR-033a explicitly requires it.
- *Direct API integration (SES, SendGrid SDK)*: Vendor lock-in; SMTP is universal.
- *In-app only reset*: Not specified; email is the standard for password recovery.

---

## 11. Chart of Accounts Segment Architecture

**Decision**: Store account segments as a JSONB column `segments` on the `chart_of_accounts` table. The account code is derived from concatenating segment values with a configurable delimiter (default `-`). Validation rules (which segments are required, valid values per segment) are stored in a `segment_config` table.

**Rationale**:
- JSONB allows flexible multi-segment account structures without requiring a separate table per segment.
- PostgreSQL JSONB supports indexing via GIN indexes for segment-based queries.
- The derived `account_code` column (with unique constraint) provides a human-readable identifier.

**Alternatives Considered**:
- *Separate `account_segments` table*: More normalized but requires JOINs for every account lookup.
- *Fixed segment columns (`seg1`, `seg2`, ...)*: Inflexible; different tenants have different segment structures.

---

## 12. Docker Compose Production Deployment

**Decision**: Use Docker Compose for both development and production. The compose file defines all 12 services + gateway + PostgreSQL + NATS + MailHog (dev) / SMTP relay (prod). For production, resource limits are set per container. Tenant databases are created as separate PostgreSQL databases on the same PostgreSQL container (or managed RDS instance).

**Rationale**:
- Spec explicitly states Docker Compose for initial deployment.
- A single `docker-compose.yml` with override files (`docker-compose.override.yml` for dev, `docker-compose.prod.yml` for production) keeps configuration DRY.
- For database-per-tenant, a single PostgreSQL instance can host multiple databases; connection pooling per-tenant per-service is handled in the application layer.

**Alternatives Considered**:
- *Kubernetes*: Explicitly deferred per spec.
- *Single-process monolith*: Violates constitution Principle II (Microservices Architecture).

---

## 13. Report Export (PDF/XLSX)

**Decision**: 
- **PDF**: Use `genpdf` crate for server-side PDF generation from report data. Simple table-based layout for financial reports.
- **XLSX**: Use `rust_xlsxwriter` crate for Excel export. Supports multi-sheet workbooks, formatting, and formulas (for totals).
- Exports are generated synchronously for reports under 10,000 rows. For larger exports, generate asynchronously and provide a download link.

**Rationale**:
- Both crates are pure Rust (no external dependencies like `wkhtmltopdf`).
- Synchronous generation is sufficient for v1 scale (spec says <10 seconds for 100K transactions).
- The spec differentiates report export (PDF/XLSX) from data table CSV export (cross-cutting concern).

**Alternatives Considered**:
- *Client-side PDF generation (jsPDF)*: Limited formatting; poor for complex financial reports.
- *Headless Chrome + HTML→PDF*: Heavy dependency; startup cost per report.
- *Apache POi (via sidecar)*: Requires JVM; defeats Rust-first principle.

---

## Summary of Resolved Unknowns

| # | Unknown | Decision |
|---|---------|----------|
| 1 | Tenant DB routing | Moka cache of PgPool instances keyed by tenant_id |
| 2 | Auth token strategy | JWT RS256 with JWKS; claims carry tenant_id, roles, dept |
| 3 | gRPC vs NATS usage | gRPC for sync queries; NATS for async events |
| 4 | Audit trail design | Correlation ID chain; GL owns financial audit trail |
| 5 | Report aggregation | Reporting service calls source services via gRPC sequentially |
| 6 | Tenant provisioning | CLI/API creates per-tenant DBs + runs migrations |
| 7 | Frontend state | TanStack Query + polling (60s dashboard, 30s notifications) |
| 8 | Concurrency control | Optimistic locking via version column |
| 9 | Period close orchestration | Sequential steps in GL service with resumability |
| 10 | Email infrastructure | lettre SMTP + MailHog (dev) + SES/SendGrid (prod) |
| 11 | Account segment storage | JSONB segments column with derived account_code |
| 12 | Deployment | Docker Compose with dev/prod override files |
| 13 | Report export | genpdf (PDF) + rust_xlsxwriter (XLSX); synchronous for <10K rows |
