# Quickstart Guide: Oracle Fusion Cloud ERP Clone

**Date**: 2026-04-25 | **Branch**: `001-oracle-fusion-erp-clone`

## Prerequisites

- **Rust**: Latest stable toolchain (`rustup update stable`)
- **Node.js**: v20+ and npm v10+ (for frontend)
- **Docker** + **Docker Compose**: For local PostgreSQL and NATS
- **sqlx-cli**: `cargo install sqlx-cli --no-default-features --features postgres`
- **protoc**: Protocol Buffer compiler (`apt install protobuf-compiler` or `brew install protobuf`)
- **cargo-nextest** (optional): `cargo install cargo-nextest` (faster test runner)

## Initial Setup

### 1. Clone and enter the repository

```bash
git clone <repo-url> fusion
cd fusion
git checkout 001-oracle-fusion-erp-clone
```

### 2. Start infrastructure

```bash
docker compose up -d postgres nats
# This starts:
#   - PostgreSQL on port 5432 (per-service databases created automatically)
#   - NATS with JetStream on port 4222
#   - NATS monitoring on port 8222
```

### 3. Build the workspace

```bash
# Build all crates (workspace)
cargo build

# Build a specific service
cargo build -p service-gl
```

### 4. Run database migrations

```bash
# Run migrations for a specific service
cd services/gl
sqlx migrate run
cd ../..

# Or use the workspace-level helper (if configured)
cargo run -p service-gl -- --migrate-only
```

### 5. Run tests

```bash
# All tests (workspace)
cargo test

# With nextest (faster, better output)
cargo nextest run

# Specific service tests
cargo test -p service-gl

# Contract tests only
cargo test -p service-gl --test contract

# Run with logging
RUST_LOG=debug cargo test -p service-gl -- --nocapture
```

### 6. Start services

```bash
# Start all services via Docker Compose
docker compose up -d

# Or run a service locally for development
cargo run -p service-gl
# Defaults to http://localhost:8081 (gRPC) and http://localhost:8081/health (HTTP)

# Run the API gateway
cargo run -p service-gateway
# Defaults to http://localhost:8080
```

### 7. Start the frontend

```bash
cd web
npm install
npm run dev
# Opens at http://localhost:5173 (proxies API calls to gateway at :8080)
```

## Service Port Map

| Service | HTTP Port | gRPC Port | Database |
|---------|-----------|-----------|----------|
| gateway | 8080 | — | — |
| gl | 8081 | 9091 | `fusion_gl` |
| ap | 8082 | 9092 | `fusion_ap` |
| ar | 8083 | 9093 | `fusion_ar` |
| identity | 8084 | 9094 | `fusion_identity` |
| workflow | 8085 | 9095 | `fusion_workflow` |
| notification | 8086 | 9096 | — |
| procurement | 8087 | 9097 | `fusion_procurement` |
| reporting | 8088 | 9098 | `fusion_reporting` |
| consolidation | 8089 | 9099 | `fusion_consolidation` |
| budget | 8090 | 9100 | `fusion_budget` |
| tax | 8091 | 9101 | `fusion_tax` |
| asset | 8092 | 9102 | `fusion_asset` |

## Development Workflow

### TDD Cycle (Constitution Principle I)

```bash
# 1. RED: Write a failing test
# Create test file: services/gl/tests/journal_entry_test.rs
cargo test -p service-gl --test journal_entry  # Should FAIL

# 2. GREEN: Write minimal implementation
# Edit services/gl/src/service.rs
cargo test -p service-gl --test journal_entry  # Should PASS

# 3. REFACTOR: Clean up while keeping tests green
cargo test -p service-gl  # All tests must pass
cargo clippy -- -D warnings
cargo fmt --check
```

### Adding a New Endpoint

1. **Define the proto contract** in `proto/<service>/v1/<service>.proto`
2. **Regenerate proto types**: `cargo build -p crates-proto` (build.rs handles this)
3. **Write contract test** first in `services/<service>/tests/`
4. **Implement handler** in `services/<service>/src/handlers.rs`
5. **Implement service logic** in `services/<service>/src/service.rs`
6. **Implement repository** in `services/<service>/src/repository.rs`
7. **Add migration** if new table/column needed: `sqlx migrate add <name>`
8. **Run all checks**: `cargo test && cargo clippy -- -D warnings && cargo fmt --check`

### Database Migrations

```bash
# Create a new migration
cd services/gl
sqlx migrate add create_journal_entry_tags

# Edit the generated file in migrations/
# Run migration
sqlx migrate run

# Rollback (if needed)
sqlx migrate revert
```

## Environment Variables

Each service reads from environment variables (or `.env` file):

```bash
# Database
DATABASE_URL=postgres://fusion:fusion@localhost:5432/fusion_gl

# NATS
NATS_URL=nats://localhost:4222

# Auth
JWT_PUBLIC_KEY_PATH=./keys/public.pem

# Observability
RUST_LOG=info
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# Service
SERVICE_PORT=8081
GRPC_PORT=9091
```

## Code Quality Gates

Before every commit, ensure:

```bash
# All tests pass
cargo test

# No clippy warnings
cargo clippy -- -D warnings

# Formatted code
cargo fmt --check

# No known vulnerabilities
cargo audit

# Per-service build
cargo build -p service-gl
```

## Docker Compose Services

```yaml
# Key services in docker-compose.yml
services:
  postgres:
    image: postgres:16
    ports: ["5432:5432"]
    environment:
      POSTGRES_USER: fusion
      POSTGRES_PASSWORD: fusion

  nats:
    image: nats:2-alpine
    ports: ["4222:4222", "8222:8222"]
    command: ["--jetstream"]
```

## Useful Commands

```bash
# Check workspace dependency tree
cargo tree

# Verify no circular dependencies
cargo tree --duplicates

# Run a specific integration test
cargo nextest run -p service-gl -E 'test(journal_entry_posting)'

# View NATS streams
nats stream list

# Generate keys for JWT (initial setup)
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
```
