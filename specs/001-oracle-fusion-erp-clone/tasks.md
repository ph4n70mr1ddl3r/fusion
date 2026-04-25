# Tasks: Oracle Fusion Cloud ERP Clone

**Input**: Design documents from `/specs/001-oracle-fusion-erp-clone/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/

**Tests**: Included per Constitution Principle I (Test-Driven Development). Every service follows Red-Green-Refactor.

**Organization**: Tasks grouped by user story for independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Backend services**: `services/<name>/` — each owns its DB, gRPC server, migrations
- **Shared crates**: `crates/<name>/` — shared libraries (proto, types, db, auth, error, messaging, testing)
- **Proto definitions**: `proto/<service>/v1/<service>.proto` — canonical gRPC contracts
- **Frontend SPA**: `web/src/` — React + TypeScript + Vite application
- **Infrastructure**: `docker-compose.yml`, `Dockerfile`, `.github/workflows/`

---

## TDD Process Enforcement (Constitution Principle I)

Per the project constitution, **every** user story phase MUST follow this gate:

1. **RED**: Write contract + model tests — they MUST fail.
2. **⏸ GATE — User Approval**: Pause. Present failing tests to the user.
   Implementation MUST NOT begin until the user explicitly approves the test
   suite as correctly capturing the requirement.
3. **GREEN**: Write minimum implementation to pass all tests.
4. **REFACTOR**: Clean up while keeping all tests green.

This gate is enforced at each "Tests" subsection within Phases 3–13. Do not
skip from test tasks directly to implementation tasks without user sign-off.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Cargo workspace initialization, shared crates, Docker infrastructure, proto compilation, frontend scaffolding, CI pipeline

- [ ] T001 Create Cargo workspace root with `Cargo.toml`, `rust-toolchain.toml` (MSRV pin), and workspace members for all 13 services + 9 shared crates
- [ ] T002 [P] Initialize `web/` frontend with Vite + React 19 + TypeScript 5 + TanStack Query + Zustand + Recharts + TanStack Table in `web/package.json` and `web/vite.config.ts`
- [ ] T003 [P] Create `docker-compose.yml` with PostgreSQL 16 and NATS 2 (JetStream enabled) services with health checks
- [ ] T004 [P] Create multi-stage `Dockerfile` template for Rust service builds in `Dockerfile`
- [ ] T005 Copy proto definitions from `specs/001-oracle-fusion-erp-clone/contracts/` to versioned `proto/` directory structure (`proto/common/v1/types.proto`, `proto/gl/v1/gl.proto`, etc.)
- [ ] T006 [P] Implement `crates/proto/` with `build.rs` for proto compilation via `tonic-build`, generating Rust types from all proto files
- [ ] T007 [P] Implement `crates/types/` with shared domain types (`Money`, `Currency`, `DecimalValue`, `AuditInfo`, `TenantContext`, `Uuid`) mapping to proto types
- [ ] T008 [P] Implement `crates/db/` with sqlx PostgreSQL pool management, tenant schema resolution (`SET search_path`), and migration runner helper
- [ ] T009 [P] Implement `crates/auth/` with JWT RS256 validation, axum middleware for tenant extraction from JWT claims, and `AuthorizationService` trait
- [ ] T010 [P] Implement `crates/error/` with unified error types, API error mapping, and `tonic::Status` conversion for gRPC handlers
- [ ] T011 [P] Implement `crates/messaging/` with NATS JetStream publisher/subscriber abstraction using `async-nats` crate
- [ ] T011a [P] Implement `crates/resilience/` shared crate with circuit breaker pattern (via `tower`), configurable retry policies, and gRPC client interceptor — all inter-service calls MUST use this interceptor for day-one reliability (Constitution Principle V)
- [ ] T012 [P] Implement `crates/testing/` with test fixtures, mock gRPC services, DB seeders, and `TestDb` helper for isolated test databases
- [ ] T013 [P] Configure CI pipeline in `.github/workflows/ci.yml` with cargo build, cargo nextest, clippy, fmt check, and cargo audit
- [ ] T014 [P] Create `.env.example` with all service environment variables (DATABASE_URL, NATS_URL, JWT_PUBLIC_KEY_PATH, RUST_LOG, etc.)
- [ ] T015 Generate RSA-2048 key pair in `keys/private.pem` and `keys/public.pem` for JWT signing/verification
- [ ] T016 [P] Configure `cargo-nextest` in `.nextest.toml` with test threading and retry settings
- [ ] T016a [P] Implement `crates/observability/` shared crate with OpenTelemetry tracing initialization, span propagation helpers for gRPC and NATS, and standard Prometheus metrics registry (request latency, error rate, throughput counters) — all services MUST depend on this crate for day-one observability (Constitution Principle V)
- [ ] T017 [P] Create `web/src/api/` API client scaffolding with Axios/TanStack Query configuration and gateway base URL
- [ ] T018 [P] Create `web/src/types/` TypeScript type definitions generated from proto contract structures
- [ ] T019 [P] Create `web/src/stores/` Zustand stores for global app state (current user, current tenant, sidebar, selected period)
- [ ] T020 [P] Create `web/src/components/` shared UI scaffolding: `Layout.tsx`, `Sidebar.tsx`, `Header.tsx`, `DataTable.tsx`, `FormBuilder.tsx`, `ConfirmDialog.tsx`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure services (Identity, Gateway, Workflow, Notification) that ALL user stories depend on. Each service owns its PostgreSQL database, exposes gRPC endpoints, and follows the layered `handlers → service → repository` architecture.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Identity & Access Service (Basic Authentication)

- [ ] T021 Scaffold identity service structure in `services/identity/` with `Cargo.toml`, `src/main.rs`, `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T022 Create identity database migrations for `users`, `roles`, `permissions`, `user_roles`, `refresh_tokens`, `audit_logs` tables in `services/identity/migrations/`
- [ ] T023 Write identity model tests for `User`, `Role`, `Permission` entities with validation rules in `services/identity/tests/model_test.rs`
- [ ] T024 Implement `User` model and `UserRepository` (CRUD, find by email, active filtering) in `services/identity/src/repository/user_repo.rs`
- [ ] T025 [P] Implement `Role` model and `RoleRepository` (CRUD, permissions by role) in `services/identity/src/repository/role_repo.rs`
- [ ] T026 [P] Implement `RefreshTokenRepository` (create, find by hash, revoke) in `services/identity/src/repository/token_repo.rs`
- [ ] T027 Implement `AuthenticationService` (login with Argon2 verification, JWT RS256 access token + refresh token generation, refresh flow, logout/revocation) in `services/identity/src/service/auth_service.rs`
- [ ] T028 [P] Implement `UserService` (create with Argon2 hashing, get, list, update, deactivate) in `services/identity/src/service/user_service.rs`
- [ ] T029 [P] Implement `RoleService` (create, get, list, assign role to user, revoke role) in `services/identity/src/service/role_service.rs`
- [ ] T030 Implement identity gRPC handlers for `UserService`, `RoleService`, `AuthenticationService` in `services/identity/src/handlers/`
- [ ] T031 Implement identity server bootstrap with health/readiness endpoints, gRPC server startup, and `crates/observability` integration (tracing + Prometheus metrics) in `services/identity/src/main.rs`
- [ ] T032 Write identity service integration tests (registration, login, token refresh, role assignment) in `services/identity/tests/integration_test.rs`

### API Gateway

- [ ] T033 Scaffold gateway service structure in `services/gateway/` with `Cargo.toml`, `src/main.rs`, `src/routes/`, `src/middleware/`
- [ ] T034 Implement gateway REST-to-gRPC routing with tonic clients for each backend service in `services/gateway/src/routes/mod.rs`
- [ ] T035 Implement gateway auth middleware (JWT validation via `crates/auth`, tenant extraction, user context injection) in `services/gateway/src/middleware/auth.rs`
- [ ] T036 [P] Implement gateway rate limiting middleware via `tower` in `services/gateway/src/middleware/rate_limit.rs`
- [ ] T037 Implement gateway server bootstrap with health/readiness, all REST route registrations, and `crates/observability` integration (tracing + Prometheus metrics) in `services/gateway/src/main.rs`
- [ ] T038 Write gateway integration tests (auth middleware, proxy routing, rate limiting) in `services/gateway/tests/integration_test.rs`

### Workflow Service (Basic Approvals)

- [ ] T039 Scaffold workflow service structure in `services/workflow/` with `Cargo.toml`, `src/main.rs`, `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T040 Create workflow database migrations for `approval_workflows`, `approval_steps`, `approval_instances`, `approval_actions` tables in `services/workflow/migrations/`
- [ ] T041 Write workflow model tests for `ApprovalWorkflow`, `ApprovalInstance`, `ApprovalAction` in `services/workflow/tests/model_test.rs`
- [ ] T042 Implement `ApprovalWorkflowRepository` (CRUD workflows with steps) in `services/workflow/src/repository/workflow_repo.rs`
- [ ] T043 [P] Implement `ApprovalInstanceRepository` (create instance, list pending, update status, record actions) in `services/workflow/src/repository/instance_repo.rs`
- [ ] T044 Implement `ApprovalWorkflowService` (create, get, list, basic condition matching) in `services/workflow/src/service/workflow_service.rs`
- [ ] T045 Implement `ApprovalInstanceService` (submit for approval, approve, reject, find matching workflow) in `services/workflow/src/service/instance_service.rs`
- [ ] T046 Implement workflow gRPC handlers for `ApprovalWorkflowService` and `ApprovalInstanceService` in `services/workflow/src/handlers/`
- [ ] T047 Implement workflow server bootstrap with health/readiness, NATS event subscription, and `crates/observability` integration (tracing + Prometheus metrics) in `services/workflow/src/main.rs`
- [ ] T048 Write workflow service integration tests (workflow CRUD, submit/approve/reject flow) in `services/workflow/tests/integration_test.rs`

### Notification Service (Basic In-App)

- [ ] T049 Scaffold notification service structure in `services/notification/` with `Cargo.toml`, `src/main.rs`, `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T050 Create notification database migration for `notifications` table in `services/notification/migrations/`
- [ ] T051 Implement `NotificationRepository` (create, list by user, mark read, unread count) in `services/notification/src/repository/notification_repo.rs`
- [ ] T052 Implement `NotificationService` (create, list, mark read, mark all read, unread count) in `services/notification/src/service/notification_service.rs`
- [ ] T053 Implement notification NATS event subscriber for incoming approval and system events in `services/notification/src/subscriber.rs`
- [ ] T054 Implement notification gRPC handlers for `NotificationService` and `NotificationEventService` in `services/notification/src/handlers/`
- [ ] T055 Implement notification server bootstrap with health/readiness and `crates/observability` integration (tracing + Prometheus metrics) in `services/notification/src/main.rs`
- [ ] T056 Write notification service tests in `services/notification/tests/integration_test.rs`

### P1 Service Scaffolding

- [ ] T057 Scaffold GL service structure in `services/gl/` with `Cargo.toml`, `src/main.rs` (health/readiness stub), `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T058 [P] Scaffold AP service structure in `services/ap/` with `Cargo.toml`, `src/main.rs` (health/readiness stub), `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T059 [P] Scaffold AR service structure in `services/ar/` with `Cargo.toml`, `src/main.rs` (health/readiness stub), `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`

**Checkpoint**: Foundation ready — Identity, Gateway, Workflow, and Notification services are operational. GL, AP, AR services are scaffolded. User story implementation can now begin in parallel.

---

## Phase 3: User Story 1 — General Ledger Management (Priority: P1) 🎯 MVP

**Goal**: Enable financial controllers to manage chart of accounts, post journal entries, manage financial periods, and generate trial balance reports.

**Independent Test**: Create a chart of accounts → post journal entries → run trial balance → close a period — delivering a standalone functional accounting system.

### Tests for User Story 1

- [ ] T060 [US1] Write GL contract tests validating all proto RPCs (`ChartOfAccountsService`, `JournalEntryService`, `FinancialPeriodService`, `TrialBalanceService`) in `services/gl/tests/contract_test.rs`
- [ ] T061 [P] [US1] Write GL model tests for `ChartOfAccount`, `FinancialPeriod`, `JournalEntry`, `JournalEntryLine` with validation rules in `services/gl/tests/model_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing GL contract and model tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 1

- [ ] T062 [US1] Create GL database migrations for `chart_of_accounts`, `financial_periods`, `journal_entries`, `journal_entry_lines` tables in `services/gl/migrations/`
- [ ] T063 [US1] Implement `ChartOfAccount` model and `AccountRepository` (CRUD, hierarchy, segment filtering) in `services/gl/src/repository/account_repo.rs`
- [ ] T064 [P] [US1] Implement `FinancialPeriod` model and `PeriodRepository` (CRUD, find by date range, open/close) in `services/gl/src/repository/period_repo.rs`
- [ ] T065 [P] [US1] Implement `JournalEntry` model and `JournalRepository` (CRUD, status transitions, source filtering) in `services/gl/src/repository/journal_repo.rs`
- [ ] T066 [US1] Implement `ChartOfAccountsService` (create, get, list with pagination, update, deactivate, hierarchy traversal) in `services/gl/src/service/account_service.rs`
- [ ] T067 [US1] Implement `FinancialPeriodService` (create, get, list, close period with posting prevention, permanently close) in `services/gl/src/service/period_service.rs`
- [ ] T068 [US1] Implement `JournalEntryService` (create draft, validate balanced debits/credits, post to open period only, reverse with audit trail, auto-generate entry numbers) in `services/gl/src/service/journal_service.rs`
- [ ] T069 [US1] Implement `TrialBalanceService` (calculate opening balances, period activity, closing balances, debit/credit totals) in `services/gl/src/service/trial_balance_service.rs`
- [ ] T070 [US1] Implement GL NATS event publisher for `JournalEntryPosted`, `PeriodClosed` events in `services/gl/src/events.rs`
- [ ] T071 [US1] Implement GL NATS subscriber for receiving AP/AR invoice posting requests in `services/gl/src/subscriber.rs`
- [ ] T072 [US1] Implement `ChartOfAccountsService` gRPC handlers in `services/gl/src/handlers/account_handler.rs`
- [ ] T073 [P] [US1] Implement `JournalEntryService` gRPC handlers in `services/gl/src/handlers/journal_handler.rs`
- [ ] T074 [P] [US1] Implement `FinancialPeriodService` gRPC handlers in `services/gl/src/handlers/period_handler.rs`
- [ ] T075 [P] [US1] Implement `TrialBalanceService` gRPC handler in `services/gl/src/handlers/trial_balance_handler.rs`
- [ ] T076 [US1] Implement GL server bootstrap with gRPC + HTTP health endpoints, NATS subscriptions, and `crates/observability` integration (tracing + Prometheus metrics) in `services/gl/src/main.rs`
- [ ] T077 [US1] Write GL integration tests (account CRUD, journal entry lifecycle with balance validation, period close with posting prevention, trial balance accuracy) in `services/gl/tests/integration_test.rs`
- [ ] T078 [US1] Add GL REST routes to gateway proxy in `services/gateway/src/routes/gl.rs`

### Frontend Implementation for User Story 1

- [ ] T079 [US1] Create `web/src/hooks/useGL.ts` with TanStack Query hooks for chart of accounts, journal entries, periods, and trial balance API calls
- [ ] T080 [P] [US1] Create `web/src/pages/gl/ChartOfAccounts.tsx` — account list with hierarchy tree, create/edit/deactivate account modal, segment display
- [ ] T081 [P] [US1] Create `web/src/pages/gl/JournalEntry.tsx` — journal entry form with multi-line debit/credit grid, real-time balance validation, post button, reversal
- [ ] T082 [P] [US1] Create `web/src/pages/gl/FinancialPeriods.tsx` — period list with status badges, close period action with confirmation
- [ ] T083 [P] [US1] Create `web/src/pages/gl/TrialBalance.tsx` — trial balance table with opening/period/closing columns, debit/credit totals, export to CSV
- [ ] T084 [US1] Add GL module routes to `web/src/App.tsx` router with sidebar navigation entries

**Checkpoint**: General Ledger is fully functional — chart of accounts management, journal entry posting with validation, period close, and trial balance reporting all working independently.

---

## Phase 4: User Story 2 — Accounts Payable (Priority: P1)

**Goal**: Enable AP clerks to manage vendor records, enter and approve supplier invoices, process payments (including batch), and generate AP aging reports.

**Independent Test**: Create vendor → enter invoice → approve → post → verify GL journal entry created → process payment → verify GL entry → run aging report.

### Tests for User Story 2

- [ ] T085 [US2] Write AP contract tests validating all proto RPCs (`VendorService`, `ApInvoiceService`, `PaymentService`, `ApAgingService`) in `services/ap/tests/contract_test.rs`
- [ ] T086 [P] [US2] Write AP model tests for `Vendor`, `ApInvoice`, `ApInvoiceLine`, `Payment`, `PaymentInvoiceAllocation` in `services/ap/tests/model_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing AP contract and model tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 2

- [ ] T087 [US2] Create AP database migrations for `vendors`, `ap_invoices`, `ap_invoice_lines`, `payments`, `payment_invoice_allocations` tables in `services/ap/migrations/`
- [ ] T088 [US2] Implement `Vendor` model and `VendorRepository` (CRUD, search by name, active filtering) in `services/ap/src/repository/vendor_repo.rs`
- [ ] T089 [P] [US2] Implement `ApInvoice` model and `InvoiceRepository` (CRUD, status transitions, vendor filtering, date range) in `services/ap/src/repository/invoice_repo.rs`
- [ ] T090 [P] [US2] Implement `Payment` model and `PaymentRepository` (CRUD, batch operations, vendor filtering) in `services/ap/src/repository/payment_repo.rs`
- [ ] T091 [US2] Implement `VendorService` (create, get, list, update vendor records) in `services/ap/src/service/vendor_service.rs`
- [ ] T092 [US2] Implement `ApInvoiceService` (create with line items, auto-calculate totals, approve, post with GL journal entry via NATS saga) in `services/ap/src/service/invoice_service.rs`
- [ ] T093 [US2] Implement `PaymentService` (create single payment, process batch with payment method grouping, allocate to invoices, GL entry via NATS) in `services/ap/src/service/payment_service.rs`
- [ ] T094 [US2] Implement `ApAgingService` (aging report with time buckets: current, 30, 60, 90+ days) in `services/ap/src/service/aging_service.rs`
- [ ] T095 [US2] Implement AP NATS event publisher (`InvoicePosted`, `PaymentProcessed`) and subscriber (GL confirmation, workflow approval requests) in `services/ap/src/events.rs`
- [ ] T096 [US2] Implement `VendorService` gRPC handlers in `services/ap/src/handlers/vendor_handler.rs`
- [ ] T097 [P] [US2] Implement `ApInvoiceService` gRPC handlers in `services/ap/src/handlers/invoice_handler.rs`
- [ ] T098 [P] [US2] Implement `PaymentService` gRPC handlers in `services/ap/src/handlers/payment_handler.rs`
- [ ] T099 [P] [US2] Implement `ApAgingService` gRPC handler in `services/ap/src/handlers/aging_handler.rs`
- [ ] T100 [US2] Implement AP server bootstrap with gRPC + HTTP health and NATS in `services/ap/src/main.rs`
- [ ] T101 [US2] Write AP integration tests (vendor CRUD, invoice create/approve/post lifecycle, payment batch processing, aging report accuracy, GL integration) in `services/ap/tests/integration_test.rs`
- [ ] T102 [US2] Add AP REST routes to gateway proxy in `services/gateway/src/routes/ap.rs`

### Frontend Implementation for User Story 2

- [ ] T103 [US2] Create `web/src/hooks/useAP.ts` with TanStack Query hooks for vendors, invoices, payments, and aging API calls
- [ ] T104 [P] [US2] Create `web/src/pages/ap/Vendors.tsx` — vendor list with search, create/edit vendor modal, address and payment terms
- [ ] T105 [P] [US2] Create `web/src/pages/ap/Invoices.tsx` — invoice list with status filters, create invoice form with line item grid, auto-calculation, approve/post actions
- [ ] T106 [P] [US2] Create `web/src/pages/ap/Payments.tsx` — payment list, create payment with invoice allocation, batch payment processing
- [ ] T107 [P] [US2] Create `web/src/pages/ap/ApAging.tsx` — aging report table with vendor rows and time bucket columns, totals row
- [ ] T108 [US2] Add AP module routes to `web/src/App.tsx` router with sidebar navigation entries

**Checkpoint**: Accounts Payable is fully functional — vendor management, invoice processing with GL integration, payment processing with batch support, and aging reporting all working.

---

## Phase 5: User Story 3 — Accounts Receivable (Priority: P1)

**Goal**: Enable AR clerks to manage customer records, create sales invoices, record receipts with invoice matching, enforce credit limits, and generate AR aging reports.

**Independent Test**: Create customer → create invoice → post → verify GL entry → record receipt → match to invoice → verify GL entry → check credit limit enforcement → run aging report.

### Tests for User Story 3

- [ ] T109 [US3] Write AR contract tests validating all proto RPCs (`CustomerService`, `ArInvoiceService`, `ReceiptService`, `ArAgingService`) in `services/ar/tests/contract_test.rs`
- [ ] T110 [P] [US3] Write AR model tests for `Customer`, `ArInvoice`, `ArInvoiceLine`, `Receipt`, `ReceiptInvoiceAllocation` in `services/ar/tests/model_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing AR contract and model tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 3

- [ ] T111 [US3] Create AR database migrations for `customers`, `ar_invoices`, `ar_invoice_lines`, `receipts`, `receipt_invoice_allocations` tables in `services/ar/migrations/`
- [ ] T112 [US3] Implement `Customer` model and `CustomerRepository` (CRUD, credit balance tracking, search by name, active filtering) in `services/ar/src/repository/customer_repo.rs`
- [ ] T113 [P] [US3] Implement `ArInvoice` model and `InvoiceRepository` (CRUD, status transitions, customer filtering, date range) in `services/ar/src/repository/invoice_repo.rs`
- [ ] T114 [P] [US3] Implement `Receipt` model and `ReceiptRepository` (CRUD, matching, unapplied amount tracking) in `services/ar/src/repository/receipt_repo.rs`
- [ ] T115 [US3] Implement `CustomerService` (create, get, list, update, credit limit check with outstanding balance calculation) in `services/ar/src/service/customer_service.rs`
- [ ] T116 [US3] Implement `ArInvoiceService` (create with line items, auto-calculate totals, post with revenue recognition GL entry via NATS saga, credit limit check before posting) in `services/ar/src/service/invoice_service.rs`
- [ ] T117 [US3] Implement `ReceiptService` (create with optional pre-match, match/unmatch to invoices, update unapplied amount, GL entry via NATS) in `services/ar/src/service/receipt_service.rs`
- [ ] T118 [US3] Implement `ArAgingService` (aging report with time buckets: current, 30, 60, 90+ days) in `services/ar/src/service/aging_service.rs`
- [ ] T119 [US3] Implement credit limit enforcement service (check outstanding balance, place customer on hold, emit credit hold notification event) in `services/ar/src/service/credit_service.rs`
- [ ] T120 [US3] Implement AR NATS event publisher (`InvoicePosted`, `ReceiptProcessed`, `CreditHoldTriggered`) and subscriber in `services/ar/src/events.rs`
- [ ] T121 [US3] Implement `CustomerService` gRPC handlers in `services/ar/src/handlers/customer_handler.rs`
- [ ] T122 [P] [US3] Implement `ArInvoiceService` gRPC handlers in `services/ar/src/handlers/invoice_handler.rs`
- [ ] T123 [P] [US3] Implement `ReceiptService` gRPC handlers in `services/ar/src/handlers/receipt_handler.rs`
- [ ] T124 [P] [US3] Implement `ArAgingService` gRPC handler in `services/ar/src/handlers/aging_handler.rs`
- [ ] T125 [US3] Implement AR server bootstrap with gRPC + HTTP health and NATS in `services/ar/src/main.rs`
- [ ] T126 [US3] Write AR integration tests (customer CRUD, invoice create/post lifecycle, receipt matching, credit limit enforcement, aging accuracy, GL integration) in `services/ar/tests/integration_test.rs`
- [ ] T127 [US3] Add AR REST routes to gateway proxy in `services/gateway/src/routes/ar.rs`

### Frontend Implementation for User Story 3

- [ ] T128 [US3] Create `web/src/hooks/useAR.ts` with TanStack Query hooks for customers, invoices, receipts, credit status, and aging API calls
- [ ] T129 [P] [US3] Create `web/src/pages/ar/Customers.tsx` — customer list with search, create/edit modal, credit limit display, credit hold indicator
- [ ] T130 [P] [US3] Create `web/src/pages/ar/Invoices.tsx` — invoice list with status filters, create invoice form with line items, auto-calculation, post action
- [ ] T131 [P] [US3] Create `web/src/pages/ar/Receipts.tsx` — receipt entry form with invoice matching grid, unapplied amount display, match/unmatch actions
- [ ] T132 [P] [US3] Create `web/src/pages/ar/ArAging.tsx` — aging report table with customer rows and time bucket columns, totals row
- [ ] T133 [US3] Add AR module routes to `web/src/App.tsx` router with sidebar navigation entries

**Checkpoint**: Accounts Receivable is fully functional — customer management, invoicing with GL integration, receipt matching, credit limit enforcement, and aging reporting all working.

---

## Phase 6: User Story 8 — Role-Based Access & Security (Priority: P2)

**Goal**: Enable system administrators to define roles with granular permissions, assign users to roles, enforce access control across all modules, and maintain an immutable audit log.

**Independent Test**: Create roles (AP Clerk, GL Manager, CFO) → assign module/function permissions → assign users → verify each user can only access authorized functions → check audit log records all actions.

### Tests for User Story 8

- [ ] T134 [US8] Write RBAC contract tests for `AuditLogService` and permission-enforced operations in `services/identity/tests/contract_test.rs`
- [ ] T134a [P] [US8] Write identity contract tests for password policy validation and session management RPCs in `services/identity/tests/session_contract_test.rs`
- [ ] T135 [P] [US8] Write RBAC integration test scenarios (role CRUD, permission enforcement, scope filtering, audit trail immutability) in `services/identity/tests/rbac_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing RBAC, session, and audit contract tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 8

- [ ] T136 [US8] Implement permission enforcement middleware for gRPC handlers (check role permissions from JWT claims before executing RPCs) in `crates/auth/src/permission.rs`
- [ ] T137 [US8] Implement scope-based data filtering (`OWN`, `DEPARTMENT`, `ALL`) at repository layer in `crates/db/src/scoping.rs`
- [ ] T138 [US8] Implement `AuditLogService` with append-only repository (no UPDATE/DELETE allowed) and query filtering in `services/identity/src/service/audit_service.rs`
- [ ] T139 [US8] Implement NATS audit event subscriber that captures domain events from all services into the audit log in `services/identity/src/subscriber.rs`
- [ ] T140 [US8] Implement `AuditLogService` gRPC handlers (get, list with filtering by user/module/entity/date) in `services/identity/src/handlers/audit_handler.rs`
- [ ] T141 [US8] Add permission checks to GL service gRPC handlers (create/post/reverse journal entries, manage accounts, close periods) in `services/gl/src/handlers/`
- [ ] T142 [P] [US8] Add permission checks to AP service gRPC handlers (manage vendors, create/approve/post invoices, process payments) in `services/ap/src/handlers/`
- [ ] T143 [P] [US8] Add permission checks to AR service gRPC handlers (manage customers, create/post invoices, process receipts) in `services/ar/src/handlers/`
- [ ] T144 [US8] Implement strong password policy validation (min length, complexity rules) in `services/identity/src/service/auth_service.rs`
- [ ] T145 [US8] Implement configurable session timeout and token expiration management in `services/identity/src/service/auth_service.rs`
- [ ] T146 [US8] Write RBAC integration tests (permission enforcement across GL/AP/AR, scope filtering, audit log completeness) in `services/identity/tests/integration_rbac_test.rs`

### Frontend Implementation for User Story 8

- [ ] T147 [US8] Create `web/src/hooks/useAdmin.ts` with TanStack Query hooks for roles, permissions, users, and audit log API calls
- [ ] T148 [P] [US8] Create `web/src/pages/admin/Roles.tsx` — role list, create/edit role with permission matrix (module × action × scope checkboxes)
- [ ] T149 [P] [US8] Create `web/src/pages/admin/Users.tsx` — user list, create/edit user, role assignment with multi-select, deactivate action
- [ ] T150 [P] [US8] Create `web/src/pages/admin/AuditLog.tsx` — audit log viewer with filters (user, module, entity type, date range), change details expandable row
- [ ] T151 [US8] Add admin module routes to `web/src/App.tsx` router with admin-only sidebar section
- [ ] T152 [US8] Add permission-based UI element visibility (hide/show actions based on user permissions) in `web/src/components/PermissionGuard.tsx`

**Checkpoint**: Full RBAC operational — roles with granular permissions, scope-based access, immutable audit logging across all modules, and admin UI for managing access control.

---

## Phase 7: User Story 9 — Workflow & Approval Management (Priority: P2)

**Goal**: Enable business process owners to configure multi-step approval workflows with conditions, escalation, and delegation for journal entries, invoices, and purchase orders.

**Independent Test**: Configure approval rule → submit transaction → verify routing to correct approver → approve → verify posting proceeds → test escalation timeout → test delegation.

### Tests for User Story 9

- [ ] T153 [US9] Write workflow configuration contract tests (CRUD workflows with multi-step definitions, condition matching) in `services/workflow/tests/contract_test.rs`
- [ ] T154 [P] [US9] Write workflow integration tests (multi-step approval chain, escalation timeout, delegation, notification delivery) in `services/workflow/tests/escalation_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing workflow contract and escalation tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 9

- [ ] T155 [US9] Implement full `ApprovalWorkflowService` CRUD with JSON condition parsing (amount thresholds, department matching, transaction type) in `services/workflow/src/service/workflow_service.rs`
- [ ] T156 [US9] Implement multi-step approval routing (evaluate conditions, create instance, route through sequential steps) in `services/workflow/src/service/instance_service.rs`
- [ ] T157 [US9] Implement escalation timer service using NATS delayed messages (trigger on timeout, escalate to fallback approver) in `services/workflow/src/service/escalation_service.rs`
- [ ] T158 [US9] Implement approval delegation (transfer pending approval to another user, record in audit trail) in `services/workflow/src/service/delegation_service.rs`
- [ ] T159 [US9] Implement `ApprovalWorkflowService` gRPC handlers (full CRUD, activate/deactivate) in `services/workflow/src/handlers/workflow_handler.rs`
- [ ] T160 [US9] Implement `ApprovalInstanceService` gRPC handlers (submit, approve, reject, delegate, history) in `services/workflow/src/handlers/instance_handler.rs`
- [ ] T161 [US9] Implement workflow-GL integration (journal entries exceeding threshold route to approval before posting) in `services/gl/src/service/journal_service.rs`
- [ ] T162 [P] [US9] Implement workflow-AP integration (invoices exceeding threshold route to approval before posting) in `services/ap/src/service/invoice_service.rs`
- [ ] T163 [P] [US9] Implement workflow-AR integration (credit limit override approval) in `services/ar/src/service/credit_service.rs`
- [ ] T164 [US9] Implement workflow-notification integration (notify approvers on submission, escalation, completion) in `services/workflow/src/events.rs`
- [ ] T165 [US9] Write end-to-end workflow integration tests (configure → submit → approve → verify posting → test escalation → test delegation) in `services/workflow/tests/integration_test.rs`
- [ ] T165a [US9] Write notification delivery latency benchmark test: submit approval transaction → measure time to notification delivery → assert < 5 seconds (SC-005) in `services/notification/tests/latency_test.rs`

### Frontend Implementation for User Story 9

- [ ] T166 [US9] Create `web/src/hooks/useWorkflow.ts` with TanStack Query hooks for workflow CRUD, pending approvals, and approval actions
- [ ] T167 [P] [US9] Create `web/src/pages/workflow/Workflows.tsx` — workflow list, create/edit workflow with step builder (drag reorder, add/remove steps, condition editor)
- [ ] T168 [P] [US9] Create `web/src/pages/workflow/ApprovalInbox.tsx` — pending approvals list with entity details, approve/reject/delegate actions with comment
- [ ] T169 [P] [US9] Create `web/src/pages/workflow/ApprovalHistory.tsx` — approval history with timeline view showing all actions taken
- [ ] T170 [US9] Add workflow module routes to `web/src/App.tsx` router with approval inbox badge count in sidebar

**Checkpoint**: Full workflow system operational — configurable multi-step approvals, escalation with timeouts, delegation, notification integration, and workflow management UI.

---

## Phase 8: User Story 4 — Procurement & Purchasing (Priority: P2)

**Goal**: Enable purchasing agents to create purchase requisitions, convert approved requisitions to purchase orders, record goods receipts, and support 3-way matching with AP invoices.

**Independent Test**: Create requisition → submit for approval → approve → convert to PO → issue PO → record goods receipt → verify PO status updates → create AP invoice with PO reference → 3-way match.

### Tests for User Story 4

- [ ] T171 [US4] Write procurement contract tests validating all proto RPCs (`RequisitionService`, `PurchaseOrderService`, `GoodsReceiptService`) in `services/procurement/tests/contract_test.rs`
- [ ] T172 [P] [US4] Write procurement model tests for `PurchaseRequisition`, `PurchaseOrder`, `GoodsReceipt` entities in `services/procurement/tests/model_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing procurement contract and model tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 4

- [ ] T173 [US4] Scaffold procurement service structure in `services/procurement/` with `Cargo.toml`, `src/main.rs`, `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T174 [US4] Create procurement database migrations for `purchase_requisitions`, `requisition_lines`, `purchase_orders`, `purchase_order_lines`, `goods_receipts`, `goods_receipt_lines` tables in `services/procurement/migrations/`
- [ ] T175 [US4] Implement `PurchaseRequisition` model and `RequisitionRepository` (CRUD, status transitions, requester filtering) in `services/procurement/src/repository/requisition_repo.rs`
- [ ] T176 [P] [US4] Implement `PurchaseOrder` model and `PurchaseOrderRepository` (CRUD, status transitions, vendor filtering, quantity tracking) in `services/procurement/src/repository/po_repo.rs`
- [ ] T177 [P] [US4] Implement `GoodsReceipt` model and `GoodsReceiptRepository` (CRUD, PO line quantity updates) in `services/procurement/src/repository/goods_receipt_repo.rs`
- [ ] T178 [US4] Implement `RequisitionService` (create with lines, submit for approval, approve/reject via workflow, mark converted) in `services/procurement/src/service/requisition_service.rs`
- [ ] T179 [US4] Implement `PurchaseOrderService` (create from approved requisition with vendor assignment, issue, track receipts, close when fully received) in `services/procurement/src/service/po_service.rs`
- [ ] T180 [US4] Implement `GoodsReceiptService` (create against PO, confirm receipt, update PO quantities received, validate against ordered quantities) in `services/procurement/src/service/goods_receipt_service.rs`
- [ ] T181 [US4] Implement procurement-workflow integration (requisition approval routing based on amount/category) in `services/procurement/src/service/requisition_service.rs`
- [ ] T182 [US4] Implement 3-way match service (compare PO quantities, goods receipt quantities, and AP invoice quantities/amounts) in `services/procurement/src/service/match_service.rs`
- [ ] T183 [US4] Implement procurement NATS event publisher (`RequisitionApproved`, `POIssued`, `GoodsReceived`) in `services/procurement/src/events.rs`
- [ ] T184 [US4] Implement `RequisitionService` gRPC handlers in `services/procurement/src/handlers/requisition_handler.rs`
- [ ] T185 [P] [US4] Implement `PurchaseOrderService` gRPC handlers in `services/procurement/src/handlers/po_handler.rs`
- [ ] T186 [P] [US4] Implement `GoodsReceiptService` gRPC handlers in `services/procurement/src/handlers/goods_receipt_handler.rs`
- [ ] T187 [US4] Implement procurement server bootstrap with gRPC + HTTP health and NATS in `services/procurement/src/main.rs`
- [ ] T188 [US4] Write procurement integration tests (requisition lifecycle, PO creation from requisition, goods receipt with quantity tracking, 3-way match validation) in `services/procurement/tests/integration_test.rs`
- [ ] T189 [US4] Add procurement REST routes to gateway proxy in `services/gateway/src/routes/procurement.rs`

### Frontend Implementation for User Story 4

- [ ] T190 [US4] Create `web/src/hooks/useProcurement.ts` with TanStack Query hooks for requisitions, POs, and goods receipts API calls
- [ ] T191 [P] [US4] Create `web/src/pages/procurement/Requisitions.tsx` — requisition list with status filters, create requisition form with line items, approve/reject actions
- [ ] T192 [P] [US4] Create `web/src/pages/procurement/PurchaseOrders.tsx` — PO list, create PO from requisition, issue PO, view receipt status, close PO
- [ ] T193 [P] [US4] Create `web/src/pages/procurement/GoodsReceipts.tsx` — create goods receipt against PO, quantity entry per line, condition selection
- [ ] T194 [US4] Add procurement module routes to `web/src/App.tsx` router with sidebar navigation entries

**Checkpoint**: Procurement module operational — requisition to PO workflow, goods receipt tracking, 3-way match with AP, and full procurement UI.

---

## Phase 9: User Story 5 — Financial Reporting & Dashboards (Priority: P2)

**Goal**: Enable CFOs and analysts to view real-time financial dashboards, generate standard financial reports (income statement, balance sheet, cash flow), build custom reports, and export to PDF/XLSX.

**Independent Test**: Post transactions across GL periods → generate income statement → generate balance sheet → verify balance sheet balances → view dashboard KPIs → create custom report → export to PDF.

### Tests for User Story 5

- [ ] T195 [US5] Write reporting contract tests validating all proto RPCs (`StandardReportService`, `DashboardService`, `CustomReportService`, `ReportExportService`) in `services/reporting/tests/contract_test.rs`
- [ ] T196 [P] [US5] Write reporting service tests for income statement, balance sheet, and cash flow calculations in `services/reporting/tests/report_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing reporting contract and calculation tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 5

- [ ] T197 [US5] Scaffold reporting service structure in `services/reporting/` with `Cargo.toml`, `src/main.rs`, `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T198 [US5] Create reporting database migration for `saved_reports` table in `services/reporting/migrations/`
- [ ] T199 [US5] Implement `SavedReportRepository` (CRUD, filtering by type and creator) in `services/reporting/src/repository/report_repo.rs`
- [ ] T200 [US5] Implement gRPC client wrappers for GL, AP, AR services in `services/reporting/src/clients/` (GL client for account balances, AP client for payables, AR client for receivables)
- [ ] T201 [US5] Implement `IncomeStatementService` (aggregate GL revenue/expense accounts by period, support comparison periods) in `services/reporting/src/service/income_statement_service.rs`
- [ ] T202 [P] [US5] Implement `BalanceSheetService` (aggregate GL asset/liability/equity accounts as of date, verify balancing) in `services/reporting/src/service/balance_sheet_service.rs`
- [ ] T203 [P] [US5] Implement `CashFlowStatementService` (derive from income statement + balance sheet changes) in `services/reporting/src/service/cash_flow_service.rs`
- [ ] T204 [US5] Implement `DashboardService` (aggregate KPIs: revenue, expenses, cash position, AR aging, AP aging via gRPC calls to GL/AP/AR) in `services/reporting/src/service/dashboard_service.rs`
- [ ] T205 [US5] Implement `CustomReportService` (create/save/run custom reports with configurable fields, filters, groupings stored as JSON definition) in `services/reporting/src/service/custom_report_service.rs`
- [ ] T206 [US5] Implement `ReportExportService` (PDF via `genpdf`, XLSX via `rust_xlsxwriter`, CSV export with async job queue for large datasets) in `services/reporting/src/service/export_service.rs`
- [ ] T207 [US5] Implement `StandardReportService` gRPC handlers in `services/reporting/src/handlers/standard_handler.rs`
- [ ] T208 [P] [US5] Implement `DashboardService` gRPC handler in `services/reporting/src/handlers/dashboard_handler.rs`
- [ ] T209 [P] [US5] Implement `CustomReportService` gRPC handlers in `services/reporting/src/handlers/custom_handler.rs`
- [ ] T210 [P] [US5] Implement `ReportExportService` gRPC handler in `services/reporting/src/handlers/export_handler.rs`
- [ ] T211 [US5] Implement reporting server bootstrap with gRPC + HTTP health in `services/reporting/src/main.rs`
- [ ] T212 [US5] Write reporting integration tests (standard reports from seeded GL data, dashboard KPI accuracy, custom report lifecycle, export file generation) in `services/reporting/tests/integration_test.rs`
- [ ] T213 [US5] Add reporting REST routes to gateway proxy in `services/gateway/src/routes/reporting.rs`

### Frontend Implementation for User Story 5

- [ ] T214 [US5] Create `web/src/hooks/useReporting.ts` with TanStack Query hooks for standard reports, dashboard, custom reports, and export API calls
- [ ] T215 [US5] Create `web/src/pages/reporting/Dashboard.tsx` — financial dashboard with KPI cards (revenue, expenses, cash, receivables, payables), trend indicators via Recharts, alert banners
- [ ] T216 [P] [US5] Create `web/src/pages/reporting/StandardReports.tsx` — income statement, balance sheet, cash flow views with period selector and comparison toggle
- [ ] T217 [P] [US5] Create `web/src/pages/reporting/CustomReports.tsx` — report builder with field selector, filter builder, grouping config, save/load reports
- [ ] T218 [P] [US5] Create `web/src/pages/reporting/SavedReports.tsx` — saved report list, run report, export to PDF/XLSX/CSV
- [ ] T219 [US5] Add reporting module routes to `web/src/App.tsx` router with sidebar navigation and dashboard as landing page

**Checkpoint**: Financial reporting operational — real-time dashboard, all three standard financial statements, custom report builder, and multi-format export.

---

## Phase 10: User Story 6 — Budgeting & Planning (Priority: P3)

**Goal**: Enable budget managers to create budgets by account/department/period, track actuals against budget in real time, and generate budget-vs-actual variance reports with forecasts.

**Independent Test**: Create budget template → distribute amounts across accounts and periods → approve budget → post transactions → verify actuals tracked → run budget-vs-actual report → verify variance calculation.

### Tests for User Story 6

- [ ] T220 [US6] Write budget contract tests validating all proto RPCs (`BudgetService`, `BudgetAnalysisService`) in `services/budget/tests/contract_test.rs`
- [ ] T221 [P] [US6] Write budget model tests for `Budget`, `BudgetLine` entities in `services/budget/tests/model_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing budget contract and model tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 6

- [ ] T222 [US6] Scaffold budget service structure in `services/budget/` with `Cargo.toml`, `src/main.rs`, `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T223 [US6] Create budget database migrations for `budgets`, `budget_lines` tables in `services/budget/migrations/`
- [ ] T224 [US6] Implement `Budget` model and `BudgetRepository` (CRUD, status transitions, department filtering) in `services/budget/src/repository/budget_repo.rs`
- [ ] T225 [US6] Implement `BudgetService` (create with lines, approve, upload from CSV/Excel file, calculate totals) in `services/budget/src/service/budget_service.rs`
- [ ] T226 [US6] Implement budget GL event subscriber (update `actual_amount` on budget lines when GL journal entries are posted) in `services/budget/src/subscriber.rs`
- [ ] T227 [US6] Implement `BudgetAnalysisService` (budget-vs-actual report with variance calculation, variance threshold flagging, forecast projection) in `services/budget/src/service/analysis_service.rs`
- [ ] T228 [US6] Implement `BudgetService` gRPC handlers in `services/budget/src/handlers/budget_handler.rs`
- [ ] T229 [P] [US6] Implement `BudgetAnalysisService` gRPC handlers in `services/budget/src/handlers/analysis_handler.rs`
- [ ] T230 [US6] Implement budget server bootstrap with gRPC + HTTP health and NATS in `services/budget/src/main.rs`
- [ ] T231 [US6] Write budget integration tests (budget CRUD, CSV upload, actuals tracking via GL events, variance report accuracy) in `services/budget/tests/integration_test.rs`
- [ ] T232 [US6] Add budget REST routes to gateway proxy in `services/gateway/src/routes/budget.rs`

### Frontend Implementation for User Story 6

- [ ] T233 [US6] Create `web/src/hooks/useBudget.ts` with TanStack Query hooks for budget CRUD, upload, and analysis API calls
- [ ] T234 [P] [US6] Create `web/src/pages/budget/Budgets.tsx` — budget list with status filters, create budget form with account/period grid, CSV upload
- [ ] T235 [P] [US6] Create `web/src/pages/budget/BudgetVsActual.tsx` — variance report table with budgeted/actual/variance columns, threshold highlighting, chart visualization
- [ ] T236 [US6] Add budget module routes to `web/src/App.tsx` router with sidebar navigation

**Checkpoint**: Budgeting module operational — budget creation, CSV upload, real-time actuals tracking, and variance analysis reporting.

---

## Phase 11: User Story 7 — Multi-Organization & Multi-Currency (Priority: P3)

**Goal**: Enable group controllers to manage multiple legal entities with independent ledgers, handle multi-currency transactions with configurable exchange rates, perform period-end currency revaluation, and consolidate financial results with intercompany elimination.

**Independent Test**: Create two legal entities with different base currencies → post transactions in each → run currency revaluation → verify unrealized gains/losses → run consolidation → verify intercompany elimination and currency translation.

### Tests for User Story 7

- [ ] T237 [US7] Write consolidation contract tests validating all proto RPCs (`LegalEntityService`, `ExchangeRateService`, `RevaluationService`, `ConsolidationService`) in `services/consolidation/tests/contract_test.rs`
- [ ] T238 [P] [US7] Write consolidation model tests for `LegalEntity`, `ExchangeRate`, `ConsolidationRun` in `services/consolidation/tests/model_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing consolidation contract and model tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 7

- [ ] T239 [US7] Scaffold consolidation service structure in `services/consolidation/` with `Cargo.toml`, `src/main.rs`, `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T240 [US7] Create consolidation database migrations for `legal_entities`, `exchange_rates`, `consolidation_runs` tables in `services/consolidation/migrations/`
- [ ] T241 [US7] Implement `LegalEntity` model and `LegalEntityRepository` (CRUD, hierarchy, active filtering) in `services/consolidation/src/repository/entity_repo.rs`
- [ ] T242 [P] [US7] Implement `ExchangeRate` model and `ExchangeRateRepository` (CRUD, find current rate, effective date filtering) in `services/consolidation/src/repository/rate_repo.rs`
- [ ] T243 [US7] Implement `LegalEntityService` (create, get, list, update entities with base currency and fiscal year settings) in `services/consolidation/src/service/entity_service.rs`
- [ ] T244 [US7] Implement `ExchangeRateService` (create, get, list, get current rate as of date) in `services/consolidation/src/service/rate_service.rs`
- [ ] T245 [US7] Implement `RevaluationService` (identify foreign currency accounts, calculate unrealized gains/losses using current rates, post adjusting GL entries via NATS) in `services/consolidation/src/service/revaluation_service.rs`
- [ ] T246 [US7] Implement `ConsolidationService` (fetch entity trial balances via GL gRPC, currency translation using exchange rates, intercompany elimination, produce consolidated financial statements) in `services/consolidation/src/service/consolidation_service.rs`
- [ ] T247 [US7] Implement `LegalEntityService` gRPC handlers in `services/consolidation/src/handlers/entity_handler.rs`
- [ ] T248 [P] [US7] Implement `ExchangeRateService` gRPC handlers in `services/consolidation/src/handlers/rate_handler.rs`
- [ ] T249 [P] [US7] Implement `RevaluationService` gRPC handler in `services/consolidation/src/handlers/revaluation_handler.rs`
- [ ] T250 [P] [US7] Implement `ConsolidationService` gRPC handlers in `services/consolidation/src/handlers/consolidation_handler.rs`
- [ ] T251 [US7] Implement consolidation server bootstrap with gRPC + HTTP health and NATS in `services/consolidation/src/main.rs`
- [ ] T252 [US7] Write consolidation integration tests (entity CRUD, exchange rate management, revaluation with GL posting, consolidation with elimination) in `services/consolidation/tests/integration_test.rs`
- [ ] T253 [US7] Add consolidation REST routes to gateway proxy in `services/gateway/src/routes/consolidation.rs`

### Frontend Implementation for User Story 7

- [ ] T254 [US7] Create `web/src/hooks/useConsolidation.ts` with TanStack Query hooks for entities, exchange rates, revaluation, and consolidation API calls
- [ ] T255 [P] [US7] Create `web/src/pages/consolidation/LegalEntities.tsx` — entity list, create/edit entity with currency and fiscal year settings, hierarchy display
- [ ] T256 [P] [US7] Create `web/src/pages/consolidation/ExchangeRates.tsx` — rate table, add/edit rates, effective date management, current rate lookup
- [ ] T257 [P] [US7] Create `web/src/pages/consolidation/Revaluation.tsx` — run revaluation form with entity/period selection, results display with gain/loss breakdown
- [ ] T258 [P] [US7] Create `web/src/pages/consolidation/Consolidation.tsx` — run consolidation form, status tracking, results display with elimination details
- [ ] T259 [US7] Add consolidation module routes to `web/src/App.tsx` router with sidebar navigation

**Checkpoint**: Multi-org and multi-currency operational — legal entity management, exchange rates, period-end revaluation, and financial consolidation with intercompany elimination.

---

## Phase 12: User Story 10 — Tax Management (Priority: P3)

**Goal**: Enable tax accountants to configure tax rates by jurisdiction, automatically calculate taxes on AP/AR transactions, and generate tax reports by jurisdiction and period.

**Independent Test**: Configure tax rates → process taxable AP and AR invoices → verify tax calculation → run tax report → verify collected vs. owed tax breakdown.

### Tests for User Story 10

- [ ] T260 [US10] Write tax contract tests validating all proto RPCs (`TaxRateService`, `TaxCalculationService`, `TaxReportService`) in `services/tax/tests/contract_test.rs`
- [ ] T261 [P] [US10] Write tax model tests for `TaxRate`, `TaxTransaction` in `services/tax/tests/model_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing tax contract and model tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 10

- [ ] T262 [US10] Scaffold tax service structure in `services/tax/` with `Cargo.toml`, `src/main.rs`, `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T263 [US10] Create tax database migrations for `tax_rates`, `tax_transactions` tables in `services/tax/migrations/`
- [ ] T264 [US10] Implement `TaxRate` model and `TaxRateRepository` (CRUD, effective date range queries, jurisdiction filtering) in `services/tax/src/repository/rate_repo.rs`
- [ ] T265 [P] [US10] Implement `TaxTransaction` model and `TaxTransactionRepository` (create, query by period/jurisdiction) in `services/tax/src/repository/transaction_repo.rs`
- [ ] T266 [US10] Implement `TaxRateService` (create, get, list, update with effective date management) in `services/tax/src/service/rate_service.rs`
- [ ] T267 [US10] Implement `TaxCalculationService` (lookup rate by code and effective date, calculate tax amount, create tax transaction record) in `services/tax/src/service/calculation_service.rs`
- [ ] T268 [US10] Implement tax NATS subscriber (listen for AP/AR invoice posted events, auto-calculate and record tax transactions) in `services/tax/src/subscriber.rs`
- [ ] T269 [US10] Implement `TaxReportService` (aggregate tax collected/owed by jurisdiction, category, and period) in `services/tax/src/service/report_service.rs`
- [ ] T270 [US10] Implement `TaxRateService` gRPC handlers in `services/tax/src/handlers/rate_handler.rs`
- [ ] T271 [P] [US10] Implement `TaxCalculationService` gRPC handler in `services/tax/src/handlers/calculation_handler.rs`
- [ ] T272 [P] [US10] Implement `TaxReportService` gRPC handler in `services/tax/src/handlers/report_handler.rs`
- [ ] T273 [US10] Implement tax server bootstrap with gRPC + HTTP health and NATS in `services/tax/src/main.rs`
- [ ] T274 [US10] Write tax integration tests (rate CRUD, tax calculation accuracy, AP/AR event processing, tax report aggregation) in `services/tax/tests/integration_test.rs`
- [ ] T275 [US10] Add tax REST routes to gateway proxy in `services/gateway/src/routes/tax.rs`

### Frontend Implementation for User Story 10

- [ ] T276 [US10] Create `web/src/hooks/useTax.ts` with TanStack Query hooks for tax rates, calculation, and reports
- [ ] T277 [P] [US10] Create `web/src/pages/tax/TaxRates.tsx` — tax rate list with jurisdiction/category filters, create/edit rate with effective dates
- [ ] T278 [P] [US10] Create `web/src/pages/tax/TaxReports.tsx` — tax report with period selector, jurisdiction breakdown, collected vs. owed display
- [ ] T279 [US10] Add tax module routes to `web/src/App.tsx` router with sidebar navigation

**Checkpoint**: Tax management operational — configurable tax rates, automatic tax calculation on transactions, and jurisdiction-based tax reporting.

---

## Phase 13: User Story 11 — Fixed Asset Management (Priority: P3)

**Goal**: Enable fixed asset accountants to track asset acquisition, run depreciation (straight-line and declining balance), and process asset disposals with automatic GL posting.

**Independent Test**: Create asset → run depreciation for multiple periods → verify GL depreciation entries → dispose asset → verify gain/loss GL entry → verify asset register accuracy.

### Tests for User Story 11

- [ ] T280 [US11] Write asset contract tests validating all proto RPCs (`FixedAssetService`, `DepreciationService`) in `services/asset/tests/contract_test.rs`
- [ ] T281 [P] [US11] Write asset model tests for `FixedAsset`, `DepreciationEntry`, `AssetDisposal` with depreciation calculations in `services/asset/tests/model_test.rs`

**⏸ TDD GATE (Constitution Principle I)**: Pause. Present failing asset contract and model tests to the user. Implementation MUST NOT begin until the user explicitly approves the test suite.

### Backend Implementation for User Story 11

- [ ] T282 [US11] Scaffold asset service structure in `services/asset/` with `Cargo.toml`, `src/main.rs`, `src/handlers/`, `src/service/`, `src/repository/`, `migrations/`
- [ ] T283 [US11] Create asset database migrations for `fixed_assets`, `depreciation_entries`, `asset_disposals` tables in `services/asset/migrations/`
- [ ] T284 [US11] Implement `FixedAsset` model and `AssetRepository` (CRUD, status filtering, location filtering) in `services/asset/src/repository/asset_repo.rs`
- [ ] T285 [P] [US11] Implement `DepreciationEntry` model and `DepreciationRepository` (create entries, query by asset/period, history) in `services/asset/src/repository/depreciation_repo.rs`
- [ ] T286 [US11] Implement `FixedAssetService` (create asset with GL account references, get, list, update details, get asset register) in `services/asset/src/service/asset_service.rs`
- [ ] T287 [US11] Implement `DepreciationService` (straight-line and declining balance calculation, run for period, post depreciation GL entries via NATS, update accumulated depreciation) in `services/asset/src/service/depreciation_service.rs`
- [ ] T288 [US11] Implement asset disposal service (calculate NBV, recognize gain/loss, post disposal GL entries via NATS, update asset status) in `services/asset/src/service/disposal_service.rs`
- [ ] T289 [US11] Implement `FixedAssetService` gRPC handlers (CRUD, asset register, disposal) in `services/asset/src/handlers/asset_handler.rs`
- [ ] T290 [P] [US11] Implement `DepreciationService` gRPC handlers (run depreciation, get history) in `services/asset/src/handlers/depreciation_handler.rs`
- [ ] T291 [US11] Implement asset NATS event publisher (`AssetAcquired`, `DepreciationPosted`, `AssetDisposed`) in `services/asset/src/events.rs`
- [ ] T292 [US11] Implement asset server bootstrap with gRPC + HTTP health and NATS in `services/asset/src/main.rs`
- [ ] T293 [US11] Write asset integration tests (asset creation with GL posting, depreciation across periods, disposal with gain/loss, asset register accuracy) in `services/asset/tests/integration_test.rs`
- [ ] T294 [US11] Add asset REST routes to gateway proxy in `services/gateway/src/routes/asset.rs`

### Frontend Implementation for User Story 11

- [ ] T295 [US11] Create `web/src/hooks/useAssets.ts` with TanStack Query hooks for asset CRUD, depreciation, and disposal API calls
- [ ] T296 [P] [US11] Create `web/src/pages/assets/AssetRegister.tsx` — asset list with register view (cost, accumulated depreciation, NBV), create/edit asset form
- [ ] T297 [P] [US11] Create `web/src/pages/assets/Depreciation.tsx` — run depreciation form with period selection, history view per asset, depreciation schedule table
- [ ] T298 [P] [US11] Create `web/src/pages/assets/Disposal.tsx` — dispose asset form with proceeds entry, gain/loss preview, GL posting confirmation
- [ ] T299 [US11] Add asset module routes to `web/src/App.tsx` router with sidebar navigation

**Checkpoint**: Fixed asset management operational — asset register, multi-method depreciation, disposal with GL integration, and complete asset lifecycle UI.

---

## Phase 14: Polish & Cross-Cutting Concerns

**Purpose**: Improvements affecting multiple user stories, final validation, and production readiness

- [ ] T300 [P] Add application-level caching with `moka` crate for frequently accessed data (chart of accounts, vendor/customer masters, exchange rates) with NATS-based cache invalidation across services
- [ ] T301 [P] Extend `crates/resilience/` circuit breaker with gateway-specific rate-limit-aware retry configuration and per-service tuning in `services/gateway/src/middleware/resilience.rs`
- [ ] T304 [P] Implement comprehensive error handling with user-friendly error messages in `web/src/components/ErrorBoundary.tsx` and API error interceptors

> **Note**: T302 (OpenTelemetry) and T303 (Prometheus metrics) have been promoted to Phase 1 as T016a (`crates/observability/`) to satisfy Constitution Principle V ("day-one observability"). All service bootstrap tasks now include observability integration.

### Edge Case Validation (Cross-Cutting)

**Edge Case Coverage Map (spec.md Edge Cases 1–10)**:

| # | Edge Case | Test Task |
|---|-----------|-----------|
| EC-1 | Post to closed financial period | T300a |
| EC-2 | Missing or stale currency exchange rates | T300e |
| EC-3 | Circular approval reference / no eligible approvers | T300f |
| EC-4 | Concurrent editing of same invoice or journal entry | T300a, T300b |
| EC-5 | Vendor invoice amount exceeds PO amount | T300b |
| EC-6 | Customer payment not matching any open invoice (unapplied cash) | T300c |
| EC-7 | Budget fully consumed, additional expense attempted | T300a |
| EC-8 | Differing intercompany exchange rates during consolidation | T300e |
| EC-9 | Tax rate changes mid-period | T300e |
| EC-10 | Partial receipt / over-delivery against purchase order | T300d |

- [ ] T300a [P] [US1] Write GL edge-case tests: posting to closed period (EC-1), concurrent journal entry editing (EC-4), budget fully consumed posting (EC-7) in `services/gl/tests/edge_case_test.rs`
- [ ] T300b [P] [US2] Write AP edge-case tests: vendor invoice exceeding PO amount (EC-5), concurrent invoice editing (EC-4) in `services/ap/tests/edge_case_test.rs`
- [ ] T300c [P] [US3] Write AR edge-case tests: unapplied cash / payment not matching any invoice (spec edge 6) in `services/ar/tests/edge_case_test.rs`
- [ ] T300d [P] [US4] Write procurement edge-case tests: partial/over-delivery against PO (spec edge 10) in `services/procurement/tests/edge_case_test.rs`
- [ ] T300e [P] [US7] Write consolidation edge-case tests: missing/stale exchange rates (spec edge 2), differing intercompany exchange rates (spec edge 8), mid-period tax rate change (spec edge 9) in `services/consolidation/tests/edge_case_test.rs`
- [ ] T300f [P] [US9] Write workflow edge-case tests: circular approval reference (spec edge 3), no eligible approvers (spec edge 3) in `services/workflow/tests/edge_case_test.rs`
- [ ] T305 [P] Add responsive layout and mobile-friendly navigation in `web/src/components/Layout.tsx`
- [ ] T306 [P] Implement optimistic updates with TanStack Query mutations for frequently modified entities (invoice posting, payment processing, approval actions)
- [ ] T307 [P] Add CSV/Excel export functionality to all data tables using `web/src/utils/export.ts`
- [ ] T308 Run quickstart.md validation — verify all setup steps, build commands, test commands, and service startup work as documented
- [ ] T308a Write performance benchmark suite in `tests/perf/`:
  - SC-002: 100 concurrent users submitting transactions, 95th percentile response time < 3s
  - SC-003: Generate standard reports against 100K posted transactions, assert < 10s
  - SC-009: Period-end close with 1K+ assets and 10K+ journal entries, assert < 5 minutes
  - Use `criterion` for Rust benchmarks and `k6` or `wrk` for HTTP load testing
- [ ] T309 [P] Add `web/src/pages/Login.tsx` and `web/src/pages/ForgotPassword.tsx` authentication pages with token refresh handling
- [ ] T310 Final workspace validation: `cargo test`, `cargo clippy -- -D warnings`, `cargo fmt --check`, `npm run build` (frontend) all passing

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 completion — **BLOCKS all user stories**
- **User Stories (Phase 3–13)**: All depend on Phase 2 completion
  - **Phase 3 (US1/GL)**: Can start after Phase 2 — no dependency on other stories
  - **Phase 4 (US2/AP)**: Depends on US1 (GL integration for journal posting)
  - **Phase 5 (US3/AR)**: Depends on US1 (GL integration for journal posting)
  - **Phase 6 (US8/RBAC)**: Can start after Phase 2 — enhances existing services
  - **Phase 7 (US9/Workflow)**: Can start after Phase 2 — enhances existing services
  - **Phase 8 (US4/Procurement)**: Depends on US2 (AP integration for 3-way match)
  - **Phase 9 (US5/Reporting)**: Depends on US1 (GL data), US2 (AP data), US3 (AR data)
  - **Phase 10 (US6/Budget)**: Depends on US1 (GL accounts and events)
  - **Phase 11 (US7/Multi-Org)**: Depends on US1 (GL trial balances for consolidation)
  - **Phase 12 (US10/Tax)**: Depends on US2 (AP) and US3 (AR) for tax transactions
  - **Phase 13 (US11/Assets)**: Depends on US1 (GL for depreciation posting)
- **Polish (Phase 14)**: Can run incrementally alongside any user story phase

### User Story Dependency Graph

```
Phase 1 (Setup)
    └── Phase 2 (Foundational)
            ├── Phase 3 (US1 - GL) ← MVP
            │       ├── Phase 4 (US2 - AP)
            │       │       └── Phase 8 (US4 - Procurement)
            │       └── Phase 5 (US3 - AR)
            ├── Phase 6 (US8 - RBAC) [parallel, after Phase 2]
            ├── Phase 7 (US9 - Workflow) [parallel, after Phase 2]
            ├── Phase 12 (US10 - Tax) [needs US2 + US3]
            ├── Phase 9 (US5 - Reporting) [needs US1 + US2 + US3]
            ├── Phase 10 (US6 - Budget) [needs US1]
            ├── Phase 11 (US7 - Multi-Org) [needs US1]
            └── Phase 13 (US11 - Assets) [needs US1]
    └── Phase 14 (Polish) [incremental, alongside all phases]
```

### Within Each User Story

1. Tests (contract + model) written FIRST — must FAIL before implementation
2. Database migrations created
3. Repository layer implemented (data access)
4. Service layer implemented (business logic)
5. gRPC handlers implemented (API layer)
6. NATS event publishers/subscribers (inter-service)
7. Server bootstrap and integration tests
8. Gateway REST routes added
9. Frontend pages and hooks implemented
10. End-to-end validation

### Parallel Opportunities

**Phase 1**: All `[P]` tasks (T002–T020) can run in parallel after T001

**Phase 2**: After scaffolding, repository/service/handler tasks within each infrastructure service can run in parallel:
- Identity repos (T024, T025, T026) in parallel
- Identity services (T027, T028, T029) in parallel
- Workflow repos (T042, T043) in parallel
- P1 scaffolds (T057, T058, T059) in parallel

**Phase 3–5 (P1 Stories)**: After US1 is complete, US2 and US3 can proceed in parallel (both depend only on US1)

**Phase 6–7 (P2 Infrastructure)**: US8 and US9 can proceed in parallel with each other and partially in parallel with P1 stories

**Phase 8–9 (P2 Business)**: US4 and US5 can partially overlap (US4 depends on US2, US5 depends on all P1)

**Phase 10–13 (P3)**: All P3 stories can proceed in parallel after their P1 dependencies are met

---

## Parallel Example: User Story 1 (GL)

```bash
# Step 1: Write all tests first (must FAIL)
Task T060: "Contract tests in services/gl/tests/contract_test.rs"
Task T061: "Model tests in services/gl/tests/model_test.rs"

# Step 2: Database and models (can parallelize repos)
Task T062: "Create migrations in services/gl/migrations/"
# Then parallel:
Task T063: "AccountRepository in services/gl/src/repository/account_repo.rs"
Task T064: "PeriodRepository in services/gl/src/repository/period_repo.rs"
Task T065: "JournalRepository in services/gl/src/repository/journal_repo.rs"

# Step 3: Services (sequential dependencies)
Task T066: "ChartOfAccountsService"
Task T067: "FinancialPeriodService"
Task T068: "JournalEntryService"
Task T069: "TrialBalanceService"

# Step 4: Handlers and events (can parallelize)
Task T070: "NATS events in services/gl/src/events.rs"
Task T071: "NATS subscriber in services/gl/src/subscriber.rs"
Task T072: "Account gRPC handler"
Task T073: "Journal gRPC handler"
Task T074: "Period gRPC handler"
Task T075: "Trial balance gRPC handler"

# Step 5: Frontend (can parallelize pages)
Task T079: "useGL hooks"
Task T080: "ChartOfAccounts page"
Task T081: "JournalEntry page"
Task T082: "FinancialPeriods page"
Task T083: "TrialBalance page"

# Step 6: Integration
Task T076: "Server bootstrap"
Task T077: "Integration tests"
Task T078: "Gateway routes"
Task T084: "Router configuration"
```

---

## Parallel Example: P1 Stories (US2 + US3)

```bash
# After US1 (GL) is complete, US2 and US3 can proceed in parallel:

# Developer A: User Story 2 (AP)
Tasks T085-T108: "AP service + frontend"

# Developer B: User Story 3 (AR)
Tasks T109-T133: "AR service + frontend"

# No cross-dependencies between AP and AR — both only need GL
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete **Phase 1**: Setup (shared crates, Docker, proto)
2. Complete **Phase 2**: Foundational (identity, gateway, workflow, notification)
3. Complete **Phase 3**: User Story 1 (General Ledger)
4. **STOP and VALIDATE**: Test GL independently
5. Deploy/demo: Functional accounting system with chart of accounts, journal entries, period management, trial balance

### Incremental Delivery

1. Setup + Foundational → Foundation ready
2. **+ US1 (GL)** → MVP! Functional accounting system
3. **+ US2 (AP)** → Complete payables management
4. **+ US3 (AR)** → Complete receivables management
5. **+ US8 (RBAC)** → Enterprise-grade access control
6. **+ US9 (Workflow)** → Full approval workflows
7. **+ US4 (Procurement)** → End-to-end procurement
8. **+ US5 (Reporting)** → Financial dashboards and reports
9. **+ US6 (Budget)** → Budget management
10. **+ US7 (Multi-Org)** → Multi-entity consolidation
11. **+ US10 (Tax)** → Tax compliance
12. **+ US11 (Assets)** → Fixed asset management
13. **+ Polish** → Production-ready

### Parallel Team Strategy

With multiple developers:

1. **Week 1–2**: Team completes Setup + Foundational together
2. **Week 3–4**: Developer A builds US1 (GL) — CRITICAL PATH
3. **Week 5–6**: Developer A builds US2 (AP) ∥ Developer B builds US3 (AR)
4. **Week 5–6** (overlap): Developer C builds US8 (RBAC) ∥ Developer D builds US9 (Workflow)
5. **Week 7–8**: Developer A builds US4 (Procurement) ∥ Developer B builds US5 (Reporting)
6. **Week 9+**: P3 stories in parallel (US6, US7, US10, US11)

---

## Notes

- **[P]** tasks = different files, no dependencies on incomplete work
- **[Story]** label maps each task to its user story for traceability
- Each user story is independently completable and testable
- Tests follow TDD: write first (RED), implement (GREEN), refactor
- Commit after each task or logical group
- Stop at any checkpoint to validate a story independently
- All services follow the same layered architecture: `handlers → service → repository`
- Inter-service communication: gRPC (synchronous) + NATS (asynchronous events)
- All monetary values use `DECIMAL(18,2)` in PostgreSQL and `Money` proto type
- All tables include `tenant_id` for multi-tenant isolation
- All cross-service references use UUIDs (no foreign keys across service boundaries)
