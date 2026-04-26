<!--
Sync Impact Report
==================
Version change: N/A → 1.0.0
Modified principles: N/A (initial constitution)
Added sections:
  - Core Principles: I–VII (7 principles)
  - Technology Stack
  - Development Workflow
  - Governance
Removed sections: N/A
Templates requiring updates:
  - .specify/templates/plan-template.md ✅ compatible (no changes needed)
  - .specify/templates/spec-template.md ✅ compatible (no changes needed)
  - .specify/templates/tasks-template.md ✅ compatible (no changes needed)
  - .specify/templates/checklist-template.md ✅ compatible (no changes needed)
Follow-up TODOs: None
-->

# Fusion Constitution

## Core Principles

### I. Test-Driven Development (NON-NEGOTIABLE)

TDD is mandatory for every service, library, and module in the project. No
implementation code MUST be written before its corresponding test exists and
fails.

- Red-Green-Refactor cycle strictly enforced: write failing test → get user
  approval → confirm failure → implement → confirm pass → refactor.
- Every PR MUST include tests that were written before the implementation.
- Test coverage MUST NOT decrease on any merge; target ≥ 90% line coverage.
- Test categories: unit tests (`cargo test`), integration tests (per-service
  `tests/` directory), contract tests (inter-service API contracts), and
  end-to-end tests (cross-service user journeys).

**Rationale**: TDD ensures correctness-by-construction, reduces defect rates,
and provides living documentation of expected behavior. In a microservices
architecture, well-tested contracts are the only safety net against regressions.

### II. Microservices Architecture

The system MUST be decomposed into independently deployable, loosely coupled
services that communicate via well-defined APIs.

- Each service MUST own its data store; no direct database sharing between
  services.
- Inter-service communication MUST use either asynchronous messaging (via a
  message broker) or synchronous HTTP/gRPC with explicit versioned contracts.
- Every service MUST be independently buildable, testable, and deployable.
- Service boundaries MUST align with bounded contexts from the business domain
  (e.g., Accounting, Inventory, HR, CRM, Procurement).
- Each service MUST have its own Cargo.toml and be part of a Cargo workspace.

**Rationale**: Microservices enable independent scaling, technology choices per
domain, fault isolation, and parallel team development—all critical for an ERP
system with diverse functional areas.

### III. Rust-First Development

Rust is the primary language for all backend services and shared libraries.

- All service implementations MUST be written in Rust (latest stable toolchain).
- Unsafe code MUST be justified in code review and isolated behind a safe API.
- Every crate MUST compile with zero warnings (`RUSTFLAGS="-D warnings"`).
- Dependencies MUST be audited; `cargo audit` MUST pass in CI before merge.
- Use `tokio` as the async runtime; `axum` or `tonic` for HTTP/gRPC servers.
- Shared types and protocol definitions MUST live in versioned `proto` or
  `types` crates within the workspace.

**Rationale**: Rust provides memory safety without garbage collection,
predictable performance, and a type system that catches entire classes of bugs
at compile time—essential for an ERP system processing financial and business-
critical data.

### IV. API-First Design with Versioned Contracts

Every service MUST define its external API contract before implementation.

- Contracts MUST be defined using Protocol Buffers (`.proto`) for gRPC services
  or OpenAPI 3.x specifications for REST endpoints.
- All contracts MUST be versioned using semantic versioning (MAJOR.MINOR.PATCH).
- Breaking changes MUST increment the MAJOR version and provide a migration path.
- Contract tests MUST validate that service implementations conform to their
  published contracts.
- Generated code from contracts MUST be committed to version control.

**Rationale**: Explicit, versioned contracts prevent integration drift between
microservices and enable parallel development with confidence. Contract testing
catches breaking changes before deployment.

### V. Observability and Reliability

Every service MUST be instrumented for production observability from day one.

- Structured logging (via `tracing` crate) is mandatory; no `println!` in
  production code.
- Every service MUST expose health check (`/health`) and readiness (`/ready`)
  endpoints.
- Distributed tracing (OpenTelemetry) MUST be integrated across all services.
- Metrics (request latency, error rate, throughput) MUST be exported in
  Prometheus format.
- Circuit breakers and retry policies MUST be configured for all inter-service
  calls.

**Rationale**: Microservices are opaque individually; without deep observability,
debugging distributed failures is impractical. Instrumentation is not optional
infrastructure—it is part of the service contract.

### VI. Workspace and Crate Organization

The repository MUST use a Cargo workspace to manage all services and shared
libraries.

- Each microservice MUST be a separate binary crate under `services/<name>/`.
- Shared libraries MUST be published as crates under `crates/<name>/`.
- Protocol/generated types MUST live under `crates/proto/` or `crates/types/`.
- No circular dependencies between crates; the dependency graph MUST remain a
  DAG.
- Workspace-level `Cargo.toml` MUST pin a consistent dependency version set.

**Rationale**: A workspace ensures unified dependency management, faster builds
through shared artifact caching, and a single source of truth for project
structure.

### VII. Simplicity and YAGNI

Implementation MUST favor the simplest design that satisfies current requirements.

- Do not build for hypothetical future scale; design for current requirements
  with clear extension points.
- Prefer standard library and battle-tested crates over custom implementations.
- Eliminate dead code, unused dependencies, and premature abstractions via
  `cargo clippy` and regular review.
- Every abstraction layer MUST justify its existence with a concrete current
  need, not a future possibility.

**Rationale**: Over-engineering is the primary risk in microservices projects.
YAGNI keeps the codebase maintainable, reviewable, and agile to changing
requirements.

## Technology Stack

| Layer              | Technology                          |
|--------------------|-------------------------------------|
| Language           | Rust (latest stable)                |
| Async Runtime      | tokio                               |
| HTTP Framework     | axum                                |
| gRPC Framework     | tonic + prost                       |
| Database           | PostgreSQL (per-service schema/DB)  |
| ORM / Query        | sqlx (compile-time checked)         |
| Message Broker     | NATS (JetStream)                    |
| Observability      | tracing + OpenTelemetry + Prometheus|
| Containerization   | Docker (multi-stage builds)         |
| Orchestration      | Docker Compose (dev + initial prod); Kubernetes deferred to post-v1 |
| CI/CD              | GitHub Actions / GitLab CI          |
| Testing            | cargo test, cargo nextest           |
| Linting            | cargo clippy, rustfmt               |
| Security Audit     | cargo audit                         |

## Development Workflow

### Red-Green-Refactor Cycle (Enforced)

1. **Red**: Write a failing test that captures the requirement.
2. **Review**: Get test approved before proceeding.
3. **Green**: Write the minimum implementation to make the test pass.
4. **Refactor**: Clean up while keeping all tests green.
5. **Commit**: Only when all tests pass.

### Code Review Gates

- All PRs MUST pass: `cargo test`, `cargo clippy -- -D warnings`, `cargo fmt
  --check`, `cargo audit`.
- No PR MUST be merged with failing tests or reduced coverage.
- Every PR MUST include a description linking to the spec and user story.

### Service Independence

- Each service MUST build and test independently via `cargo build -p <service>`
  and `cargo test -p <service>`.
- Integration tests between services MUST use contract test fixtures, not live
  services.

### Branching Model

- `main` branch is always deployable.
- Feature branches follow `###-feature-name` convention.
- No direct pushes to `main`; all changes via merge request.

## Governance

This constitution is the supreme governing document for the Fusion project. It
supersedes all ad-hoc practices, verbal agreements, and prior conventions.

### Amendment Procedure

1. Propose amendment as a markdown document referencing affected principles.
2. Document the rationale, impact on existing code, and migration plan.
3. Require team review and explicit approval before merging.
4. Update constitution version and sync dependent templates.

### Versioning Policy

- **MAJOR**: Principle removal, redefinition, or backward-incompatible changes.
- **MINOR**: New principle added or materially expanded guidance.
- **PATCH**: Clarifications, wording fixes, non-semantic refinements.

### Compliance

- All code reviews MUST verify compliance with this constitution.
- Complexity that violates Simplicity (Principle VII) MUST be justified in a
  complexity tracking table in the implementation plan.
- Use `.specify/templates/plan-template.md` for development workflow guidance.

**Version**: 1.0.0 | **Ratified**: 2026-04-25 | **Last Amended**: 2026-04-25
