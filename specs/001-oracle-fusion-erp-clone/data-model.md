# Data Model: Oracle Fusion Cloud ERP Clone

**Branch**: `001-oracle-fusion-erp-clone` | **Date**: 2026-04-26 | **Spec**: [spec.md](./spec.md)

## Overview

This document defines all entities, their fields, relationships, validation rules, and state transitions. Entities are grouped by owning service. All entities include standard audit columns: `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`, `created_by_user_id UUID`, `updated_by_user_id UUID`. All transactional entities include `version INT NOT NULL DEFAULT 1` for optimistic locking and `department_id UUID` for scope-based access control (FR-031).

**Multi-tenancy**: All tables below exist within per-tenant databases. The `tenant_id` is NOT stored in these tables — it is implicit from the database connection. The `fusion_platform` database (identity service) stores the tenant registry.

---

## Shared / Cross-Cutting Types

### Enum: PaymentMethod
```
CHECK | WIRE_TRANSFER | ACH | CASH
```

### Enum: CurrencyCode
```
ISO 4217 three-letter codes stored as VARCHAR(3) with CHECK constraint
```

### Enum: DepreciationMethod
```
STRAIGHT_LINE | DECLINING_BALANCE
```

---

## Platform Database (`fusion_platform`) — Identity Service

### tenants

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK, DEFAULT gen_random_uuid() | Tenant identifier |
| name | VARCHAR(255) | NOT NULL | Organization display name |
| slug | VARCHAR(100) | NOT NULL, UNIQUE | URL-safe identifier |
| db_host | VARCHAR(255) | NOT NULL | PostgreSQL host for tenant DBs |
| db_port | INTEGER | NOT NULL, DEFAULT 5432 | PostgreSQL port |
| db_name_prefix | VARCHAR(100) | NOT NULL | Prefix for tenant DB names (e.g., `fusion_{slug}_`) |
| db_credentials_ref | VARCHAR(512) | NOT NULL | Encrypted reference to DB credentials |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'active' | active, suspended, terminated |
| plan | VARCHAR(50) | NOT NULL, DEFAULT 'standard' | Billing/feature tier |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Validation**: `slug` must match `^[a-z0-9][a-z0-9-]{1,98}[a-z0-9]$`

### users

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | User identifier (also used as `created_by_user_id` in tenant DBs) |
| tenant_id | UUID | FK → tenants.id, NOT NULL | Owning tenant |
| email | VARCHAR(255) | NOT NULL | Login email |
| password_hash | VARCHAR(255) | NOT NULL | Argon2 hash |
| display_name | VARCHAR(255) | NOT NULL | |
| department_id | UUID | NULLABLE | User's department (in tenant DB); stored as UUID reference |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | |
| last_login_at | TIMESTAMPTZ | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Validation**: `email` must be valid email format, unique per tenant.

### refresh_tokens

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| user_id | UUID | FK → users.id, NOT NULL | |
| token_hash | VARCHAR(255) | NOT NULL, UNIQUE | Hashed refresh token |
| expires_at | TIMESTAMPTZ | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### password_reset_tokens

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| user_id | UUID | FK → users.id, NOT NULL | |
| token_hash | VARCHAR(255) | NOT NULL, UNIQUE | |
| expires_at | TIMESTAMPTZ | NOT NULL | Default 1 hour |
| used_at | TIMESTAMPTZ | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

---

## General Ledger Service

### segment_configs

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| name | VARCHAR(100) | NOT NULL | Segment name (e.g., "Company", "Department") |
| position | INTEGER | NOT NULL | Order in account code |
| delimiter | VARCHAR(5) | NOT NULL, DEFAULT '-' | Separator between segments |
| required | BOOLEAN | NOT NULL, DEFAULT true | |
| valid_values | JSONB | NULLABLE | Allowed values: `{"values": [{"code": "100", "name": "US Operations"}]}` |

### chart_of_accounts

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| account_code | VARCHAR(50) | NOT NULL, UNIQUE | Derived from segments concatenation |
| name | VARCHAR(255) | NOT NULL | |
| account_type | VARCHAR(20) | NOT NULL, CHECK IN ('asset','liability','equity','revenue','expense') | |
| segments | JSONB | NOT NULL | Segment values: `{"Company":"100","Department":"200","Account":"4000"}` |
| parent_account_id | UUID | FK → chart_of_accounts.id, NULLABLE | For multi-level hierarchy |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | |
| description | TEXT | NULLABLE | |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Validation**: `account_code` must be unique across active accounts. `segments` must satisfy `segment_configs` constraints.

### financial_periods

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| name | VARCHAR(100) | NOT NULL | e.g., "Q1 2026", "Jan 2026" |
| start_date | DATE | NOT NULL | |
| end_date | DATE | NOT NULL | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'open' | open, closed, permanently_closed |
| period_type | VARCHAR(20) | NOT NULL, DEFAULT 'monthly' | monthly, quarterly, yearly |
| closed_by_user_id | UUID | NULLABLE | |
| closed_at | TIMESTAMPTZ | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Validation**: Date ranges must not overlap for the same period type. `status` transitions: open → closed → permanently_closed (one-way).

**State Transitions**:
```
open ──[close_period]──→ closed ──[permanent_close]──→ permanently_closed
```

### journal_entries

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| entry_number | VARCHAR(50) | NOT NULL, UNIQUE | Auto-generated: "JE-YYYYMMDD-NNNN" |
| description | TEXT | NULLABLE | |
| entry_date | DATE | NOT NULL | |
| period_id | UUID | FK → financial_periods.id, NOT NULL | |
| source_document_type | VARCHAR(50) | NULLABLE | 'manual', 'ap_invoice', 'ar_invoice', 'payment', 'depreciation', 'revaluation', 'consolidation' |
| source_document_id | UUID | NULLABLE | ID of the originating document |
| correlation_id | UUID | NOT NULL, DEFAULT gen_random_uuid() | Cross-service tracing |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'draft' | draft, pending_approval, posted, reversed |
| is_reversing | BOOLEAN | NOT NULL, DEFAULT false | True if this entry reverses another |
| reversing_entry_id | UUID | FK → journal_entries.id, NULLABLE | The entry being reversed |
| posted_at | TIMESTAMPTZ | NULLABLE | |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Validation**: Cannot post to a closed period (FK + application check). Total debits must equal total credits (enforced at application level before status → posted). Reversed entries must have status = posted.

**State Transitions**:
```
draft ──[submit_for_approval]──→ pending_approval ──[approve]──→ posted
draft ──[post_direct]──→ posted (if no approval workflow or below threshold)
posted ──[reverse]──→ reversed (creates new reversing entry with is_reversing=true)
```

### journal_entry_lines

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| journal_entry_id | UUID | FK → journal_entries.id ON DELETE CASCADE, NOT NULL | |
| account_id | UUID | FK → chart_of_accounts.id, NOT NULL | |
| description | TEXT | NULLABLE | |
| debit_amount | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| credit_amount | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| line_order | INTEGER | NOT NULL | |

**Validation**: `CHECK (debit_amount >= 0 AND credit_amount >= 0)`. `CHECK (NOT (debit_amount > 0 AND credit_amount > 0))` — a line is either debit or credit, not both.

### journal_entry_source (Financial Audit Trail)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| journal_entry_id | UUID | FK → journal_entries.id, NOT NULL | |
| source_type | VARCHAR(50) | NOT NULL | 'manual', 'ap_invoice', 'ar_invoice', 'ap_payment', 'ar_receipt', 'depreciation', 'revaluation', 'consolidation', 'budget' |
| source_service | VARCHAR(50) | NOT NULL | Service that originated the transaction |
| source_id | UUID | NOT NULL | ID in the source service |
| correlation_id | UUID | NOT NULL | Matches journal_entries.correlation_id |

**Indexes**: `UNIQUE(source_type, source_id, journal_entry_id)`, `INDEX(correlation_id)`

---

## Accounts Payable Service

### vendors

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| vendor_code | VARCHAR(50) | NOT NULL, UNIQUE | Auto-generated or manual |
| name | VARCHAR(255) | NOT NULL | |
| address_line1 | VARCHAR(255) | NULLABLE | |
| address_line2 | VARCHAR(255) | NULLABLE | |
| city | VARCHAR(100) | NULLABLE | |
| state | VARCHAR(100) | NULLABLE | |
| postal_code | VARCHAR(20) | NULLABLE | |
| country | VARCHAR(3) | NULLABLE | ISO 3166-1 alpha-3 |
| payment_terms_days | INTEGER | NOT NULL, DEFAULT 30 | Net payment terms in days |
| tax_id | VARCHAR(50) | NULLABLE | |
| bank_name | VARCHAR(255) | NULLABLE | |
| bank_account_number | VARCHAR(100) | NULLABLE | Encrypted at rest |
| bank_routing_number | VARCHAR(50) | NULLABLE | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'active' | active, inactive |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### ap_invoices

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| invoice_number | VARCHAR(50) | NOT NULL | Vendor's invoice number |
| vendor_id | UUID | FK → vendors.id, NOT NULL | |
| invoice_date | DATE | NOT NULL | |
| due_date | DATE | NOT NULL | |
| description | TEXT | NULLABLE | |
| subtotal | NUMERIC(19,4) | NOT NULL | Before tax |
| tax_amount | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| total_amount | NUMERIC(19,4) | NOT NULL | subtotal + tax_amount |
| currency | VARCHAR(3) | NOT NULL, DEFAULT 'USD' | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'draft' | draft, pending_approval, approved, posted, paid, hold, cancelled |
| purchase_order_id | UUID | NULLABLE | FK reference to procurement service PO |
| goods_receipt_id | UUID | NULLABLE | FK reference to procurement service GR |
| match_status | VARCHAR(20) | NULLABLE | unmatched, matched, variance_hold |
| hold_reason | TEXT | NULLABLE | |
| correlation_id | UUID | NOT NULL, DEFAULT gen_random_uuid() | |
| posted_journal_entry_id | UUID | NULLABLE | GL journal entry ID |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**State Transitions**:
```
draft ──[submit]──→ pending_approval ──[approve]──→ approved ──[post]──→ posted ──[pay]──→ paid
draft ──[submit]──→ pending_approval ──[reject]──→ cancelled
posted ──[match_fail]──→ hold ──[release]──→ posted / ──[reject]──→ cancelled
```

### ap_invoice_lines

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| invoice_id | UUID | FK → ap_invoices.id ON DELETE CASCADE, NOT NULL | |
| line_number | INTEGER | NOT NULL | |
| description | TEXT | NOT NULL | |
| quantity | NUMERIC(19,4) | NOT NULL | |
| unit_price | NUMERIC(19,4) | NOT NULL | |
| line_amount | NUMERIC(19,4) | NOT NULL | quantity × unit_price |
| tax_code_id | UUID | NULLABLE | FK reference to tax service |
| tax_amount | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| po_line_id | UUID | NULLABLE | Matching PO line reference |

### ap_payments

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| payment_number | VARCHAR(50) | NOT NULL, UNIQUE | Auto-generated: "PAY-YYYYMMDD-NNNN" |
| payment_date | DATE | NOT NULL | |
| payment_method | VARCHAR(20) | NOT NULL | CHECK, WIRE_TRANSFER, ACH, CASH |
| vendor_id | UUID | FK → vendors.id, NOT NULL | |
| total_amount | NUMERIC(19,4) | NOT NULL | |
| currency | VARCHAR(3) | NOT NULL, DEFAULT 'USD' | |
| bank_account_ref | VARCHAR(255) | NULLABLE | Reference to bank account used |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'draft' | draft, issued, reconciled, voided |
| batch_id | UUID | NULLABLE | Payment batch reference |
| correlation_id | UUID | NOT NULL, DEFAULT gen_random_uuid() | |
| reconciliation_status | VARCHAR(20) | NOT NULL, DEFAULT 'unreconciled' | unreconciled, reconciled |
| bank_statement_line_id | VARCHAR(255) | NULLABLE | Reference to imported bank statement line |
| posted_journal_entry_id | UUID | NULLABLE | GL journal entry ID |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### ap_payment_invoices (junction)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| payment_id | UUID | FK → ap_payments.id, NOT NULL | |
| invoice_id | UUID | FK → ap_invoices.id, NOT NULL | |
| applied_amount | NUMERIC(19,4) | NOT NULL | |

**PK**: `(payment_id, invoice_id)`

---

## Accounts Receivable Service

### customers

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| customer_code | VARCHAR(50) | NOT NULL, UNIQUE | |
| name | VARCHAR(255) | NOT NULL | |
| address_line1 | VARCHAR(255) | NULLABLE | |
| address_line2 | VARCHAR(255) | NULLABLE | |
| city | VARCHAR(100) | NULLABLE | |
| state | VARCHAR(100) | NULLABLE | |
| postal_code | VARCHAR(20) | NULLABLE | |
| country | VARCHAR(3) | NULLABLE | |
| payment_terms_days | INTEGER | NOT NULL, DEFAULT 30 | |
| credit_limit | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | 0 = unlimited |
| tax_id | VARCHAR(50) | NULLABLE | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'active' | active, inactive, on_hold |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Validation**: When `outstanding_balance > credit_limit AND credit_limit > 0`, status must be `on_hold`.

### ar_invoices

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| invoice_number | VARCHAR(50) | NOT NULL, UNIQUE | Auto-generated: "INV-YYYYMMDD-NNNN" |
| customer_id | UUID | FK → customers.id, NOT NULL | |
| invoice_date | DATE | NOT NULL | |
| due_date | DATE | NOT NULL | |
| description | TEXT | NULLABLE | |
| subtotal | NUMERIC(19,4) | NOT NULL | |
| tax_amount | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| total_amount | NUMERIC(19,4) | NOT NULL | |
| currency | VARCHAR(3) | NOT NULL, DEFAULT 'USD' | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'draft' | draft, pending_approval, posted, paid, partially_paid, voided, uncollectible |
| correlation_id | UUID | NOT NULL, DEFAULT gen_random_uuid() | |
| posted_journal_entry_id | UUID | NULLABLE | GL journal entry ID |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**State Transitions**:
```
draft ──[submit]──→ pending_approval ──[approve]──→ posted ──[full_payment]──→ paid
posted ──[partial_payment]──→ partially_paid ──[full_payment]──→ paid
posted/partially_paid ──[void]──→ voided
posted ──[write_off]──→ uncollectible
```

### ar_invoice_lines

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| invoice_id | UUID | FK → ar_invoices.id ON DELETE CASCADE, NOT NULL | |
| line_number | INTEGER | NOT NULL | |
| description | TEXT | NOT NULL | |
| quantity | NUMERIC(19,4) | NOT NULL | |
| unit_price | NUMERIC(19,4) | NOT NULL | |
| line_amount | NUMERIC(19,4) | NOT NULL | |
| tax_code_id | UUID | NULLABLE | |
| tax_amount | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |

### ar_receipts

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| receipt_number | VARCHAR(50) | NOT NULL, UNIQUE | Auto-generated: "RCT-YYYYMMDD-NNNN" |
| customer_id | UUID | FK → customers.id, NOT NULL | |
| receipt_date | DATE | NOT NULL | |
| amount | NUMERIC(19,4) | NOT NULL | Total received |
| unapplied_amount | NUMERIC(19,4) | NOT NULL | Remaining unmatched |
| currency | VARCHAR(3) | NOT NULL, DEFAULT 'USD' | |
| payment_method | VARCHAR(20) | NOT NULL | |
| reference | VARCHAR(255) | NULLABLE | Customer's payment reference |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'unapplied' | unapplied, partially_applied, fully_applied |
| correlation_id | UUID | NOT NULL, DEFAULT gen_random_uuid() | |
| posted_journal_entry_id | UUID | NULLABLE | |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### ar_receipt_invoices (junction)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| receipt_id | UUID | FK → ar_receipts.id, NOT NULL | |
| invoice_id | UUID | FK → ar_invoices.id, NOT NULL | |
| applied_amount | NUMERIC(19,4) | NOT NULL | |

**PK**: `(receipt_id, invoice_id)`

---

## Procurement Service

### purchase_requisitions

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| requisition_number | VARCHAR(50) | NOT NULL, UNIQUE | "REQ-YYYYMMDD-NNNN" |
| requester_id | UUID | NOT NULL | User who requested |
| description | TEXT | NULLABLE | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'draft' | draft, pending_approval, approved, rejected, converted, cancelled |
| total_estimated_cost | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**State Transitions**:
```
draft ──[submit]──→ pending_approval ──[approve]──→ approved ──[convert_to_po]──→ converted
pending_approval ──[reject]──→ rejected
```

### purchase_requisition_lines

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| requisition_id | UUID | FK → purchase_requisitions.id ON DELETE CASCADE, NOT NULL | |
| line_number | INTEGER | NOT NULL | |
| item_description | TEXT | NOT NULL | |
| quantity | NUMERIC(19,4) | NOT NULL | |
| unit_of_measure | VARCHAR(20) | NOT NULL, DEFAULT 'EA' | |
| estimated_unit_price | NUMERIC(19,4) | NOT NULL | |
| category | VARCHAR(100) | NULLABLE | For approval routing |

### purchase_orders

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| po_number | VARCHAR(50) | NOT NULL, UNIQUE | "PO-YYYYMMDD-NNNN" |
| requisition_id | UUID | FK → purchase_requisitions.id, NULLABLE | |
| vendor_id | UUID | NOT NULL | FK reference to AP service vendor |
| order_date | DATE | NOT NULL | |
| delivery_date | DATE | NULLABLE | Expected delivery |
| description | TEXT | NULLABLE | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'draft' | draft, pending_approval, approved, partially_received, received, closed, cancelled |
| total_amount | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| currency | VARCHAR(3) | NOT NULL, DEFAULT 'USD' | |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**State Transitions**:
```
draft ──[submit]──→ pending_approval ──[approve]──→ approved ──[partial_receipt]──→ partially_received ──[full_receipt]──→ received ──[close]──→ closed
```

### purchase_order_lines

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| po_id | UUID | FK → purchase_orders.id ON DELETE CASCADE, NOT NULL | |
| line_number | INTEGER | NOT NULL | |
| item_description | TEXT | NOT NULL | |
| quantity_ordered | NUMERIC(19,4) | NOT NULL | |
| quantity_received | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| unit_price | NUMERIC(19,4) | NOT NULL | |
| line_amount | NUMERIC(19,4) | NOT NULL | |
| delivery_date | DATE | NULLABLE | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'open' | open, partially_received, received, closed |

### goods_receipts

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| receipt_number | VARCHAR(50) | NOT NULL, UNIQUE | "GR-YYYYMMDD-NNNN" |
| po_id | UUID | FK → purchase_orders.id, NOT NULL | |
| received_date | DATE | NOT NULL | |
| received_by | UUID | NOT NULL | User who received |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'pending' | pending, confirmed |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### goods_receipt_lines

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| goods_receipt_id | UUID | FK → goods_receipts.id ON DELETE CASCADE, NOT NULL | |
| po_line_id | UUID | FK → purchase_order_lines.id, NOT NULL | |
| quantity_received | NUMERIC(19,4) | NOT NULL | |
| notes | TEXT | NULLABLE | |

---

## Reporting Service

### saved_reports

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| name | VARCHAR(255) | NOT NULL | |
| description | TEXT | NULLABLE | |
| report_type | VARCHAR(20) | NOT NULL | 'standard', 'custom' |
| source_services | TEXT[] | NOT NULL | Array of service names to query |
| field_selections | JSONB | NOT NULL | Fields to include |
| filters | JSONB | NOT NULL | Filter criteria |
| groupings | JSONB | NULLABLE | Group-by fields |
| aggregation_functions | JSONB | NULLABLE | SUM, COUNT, AVERAGE config |
| created_by_user_id | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

---

## Budget Service

### budgets

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| name | VARCHAR(255) | NOT NULL | |
| fiscal_year | INTEGER | NOT NULL | e.g., 2026 |
| description | TEXT | NULLABLE | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'draft' | draft, pending_approval, approved, active, closed |
| total_budget | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | Computed from lines |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**State Transitions**:
```
draft ──[submit]──→ pending_approval ──[approve]──→ approved ──[period_start]──→ active ──[period_end]──→ closed
```

### budget_lines

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| budget_id | UUID | FK → budgets.id ON DELETE CASCADE, NOT NULL | |
| account_id | UUID | NOT NULL | FK reference to GL chart_of_accounts |
| department_id | UUID | NULLABLE | Optional department override |
| period_amounts | JSONB | NOT NULL | `{"01": 10000.00, "02": 10000.00, ...}` keyed by period |
| total_amount | NUMERIC(19,4) | NOT NULL | Sum of period_amounts |
| variance_threshold_pct | NUMERIC(5,2) | NOT NULL, DEFAULT 10.00 | Alert threshold |

### budget_variance_alerts

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| budget_line_id | UUID | FK → budget_lines.id, NOT NULL | |
| period | VARCHAR(7) | NOT NULL | "YYYY-MM" |
| budgeted_amount | NUMERIC(19,4) | NOT NULL | |
| actual_amount | NUMERIC(19,4) | NOT NULL | |
| variance_amount | NUMERIC(19,4) | NOT NULL | |
| variance_pct | NUMERIC(5,2) | NOT NULL | |
| alert_type | VARCHAR(20) | NOT NULL | 'warning', 'critical' |
| acknowledged_by | UUID | NULLABLE | |
| acknowledged_at | TIMESTAMPTZ | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

---

## Multi-Org / Consolidation Service

### legal_entities

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| entity_code | VARCHAR(50) | NOT NULL, UNIQUE | |
| name | VARCHAR(255) | NOT NULL | |
| base_currency | VARCHAR(3) | NOT NULL | |
| parent_entity_id | UUID | FK → legal_entities.id, NULLABLE | For consolidation hierarchy |
| fiscal_year_start_month | INTEGER | NOT NULL, DEFAULT 1 | |
| is_consolidating | BOOLEAN | NOT NULL, DEFAULT false | True for parent entities |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'active' | active, inactive |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### exchange_rates

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| from_currency | VARCHAR(3) | NOT NULL | |
| to_currency | VARCHAR(3) | NOT NULL | |
| rate | NUMERIC(19,6) | NOT NULL, CHECK (rate > 0) | |
| effective_date | DATE | NOT NULL | |
| created_by_user_id | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Indexes**: `UNIQUE(from_currency, to_currency, effective_date)`

### consolidation_runs

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| parent_entity_id | UUID | FK → legal_entities.id, NOT NULL | |
| reporting_period_id | UUID | NOT NULL | FK reference to GL financial_periods |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'pending' | pending, in_progress, completed, failed |
| exchange_rates_used | JSONB | NULLABLE | Snapshot of rates used |
| eliminations_applied | JSONB | NULLABLE | IC elimination details |
| total_eliminations | NUMERIC(19,4) | NULLABLE | |
| started_at | TIMESTAMPTZ | NULLABLE | |
| completed_at | TIMESTAMPTZ | NULLABLE | |
| error_message | TEXT | NULLABLE | |
| created_by_user_id | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### consolidation_run_entities (subsidiaries included)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| consolidation_run_id | UUID | FK → consolidation_runs.id, NOT NULL | |
| subsidiary_entity_id | UUID | FK → legal_entities.id, NOT NULL | |
| exchange_rate_used | NUMERIC(19,6) | NOT NULL | |

**PK**: `(consolidation_run_id, subsidiary_entity_id)`

---

## Fixed Asset Service

### fixed_assets

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| asset_code | VARCHAR(50) | NOT NULL, UNIQUE | |
| description | TEXT | NOT NULL | |
| acquisition_date | DATE | NOT NULL | |
| acquisition_cost | NUMERIC(19,4) | NOT NULL, CHECK (acquisition_cost > 0) | |
| useful_life_months | INTEGER | NOT NULL, CHECK (useful_life_months > 0) | |
| depreciation_method | VARCHAR(20) | NOT NULL | STRAIGHT_LINE, DECLINING_BALANCE |
| declining_balance_rate | NUMERIC(5,4) | NULLABLE | Required if DECLINING_BALANCE (e.g., 0.2000 for 200%) |
| salvage_value | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| accumulated_depreciation | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | Computed |
| net_book_value | NUMERIC(19,4) | NOT NULL | acquisition_cost - accumulated_depreciation |
| depreciation_account_id | UUID | NOT NULL | FK reference to GL expense account |
| asset_account_id | UUID | NOT NULL | FK reference to GL asset account |
| accumulated_dep_account_id | UUID | NOT NULL | FK reference to GL contra-asset account |
| location | VARCHAR(255) | NULLABLE | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'active' | active, disposed |
| legal_entity_id | UUID | NOT NULL | FK → legal_entities.id |
| created_by_user_id | UUID | NOT NULL | |
| department_id | UUID | NOT NULL | |
| version | INT | NOT NULL, DEFAULT 1 | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### depreciation_runs

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| period_id | UUID | NOT NULL | FK reference to GL financial_periods |
| run_date | DATE | NOT NULL | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'pending' | pending, completed, failed |
| total_depreciation | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| correlation_id | UUID | NOT NULL | |
| posted_journal_entry_id | UUID | NULLABLE | |
| created_by_user_id | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### depreciation_run_lines

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| run_id | UUID | FK → depreciation_runs.id, NOT NULL | |
| asset_id | UUID | FK → fixed_assets.id, NOT NULL | |
| depreciation_amount | NUMERIC(19,4) | NOT NULL | |
| prior_accumulated_dep | NUMERIC(19,4) | NOT NULL | |

### asset_disposals

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| asset_id | UUID | FK → fixed_assets.id, NOT NULL | |
| disposal_date | DATE | NOT NULL | |
| disposal_proceeds | NUMERIC(19,4) | NOT NULL, DEFAULT 0 | |
| net_book_value_at_disposal | NUMERIC(19,4) | NOT NULL | Snapshot |
| gain_loss_amount | NUMERIC(19,4) | NOT NULL | disposal_proceeds - net_book_value |
| correlation_id | UUID | NOT NULL | |
| posted_journal_entry_id | UUID | NULLABLE | |
| created_by_user_id | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

---

## Tax Service

### tax_codes

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| code | VARCHAR(50) | NOT NULL, UNIQUE | e.g., "US-CA-SALES" |
| name | VARCHAR(255) | NOT NULL | |
| jurisdiction | VARCHAR(100) | NOT NULL | |
| rate_percentage | NUMERIC(7,4) | NOT NULL, CHECK (rate_percentage >= 0) | e.g., 8.2500 for 8.25% |
| effective_from | DATE | NOT NULL | |
| effective_to | DATE | NULLABLE | NULL = currently active |
| tax_category | VARCHAR(50) | NOT NULL | sales_tax, use_tax, vat, withholding |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Validation**: Date ranges for the same code must not overlap. Only one active rate per jurisdiction+category at any given date.

---

## Workflow Service

### approval_workflows

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| name | VARCHAR(255) | NOT NULL | |
| document_type | VARCHAR(50) | NOT NULL | 'journal_entry', 'ap_invoice', 'ar_invoice', 'purchase_order', 'purchase_requisition', 'budget' |
| trigger_conditions | JSONB | NOT NULL | `{"amount_min": 1000, "amount_max": null, "departments": ["SALES"]}` |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | |
| escalation_timeout_minutes | INTEGER | NOT NULL, DEFAULT 1440 | 24 hours default |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### approval_workflow_steps

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| workflow_id | UUID | FK → approval_workflows.id ON DELETE CASCADE, NOT NULL | |
| step_number | INTEGER | NOT NULL | Execution order |
| approver_type | VARCHAR(20) | NOT NULL | 'specific_user', 'role', 'department_manager' |
| approver_ref | UUID | NOT NULL | User ID, Role ID, or Department ID based on approver_type |
| delegation_to_user_id | UUID | NULLABLE | Fallback approver |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### approval_instances

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| workflow_id | UUID | FK → approval_workflows.id, NOT NULL | |
| document_type | VARCHAR(50) | NOT NULL | |
| document_id | UUID | NOT NULL | |
| submitted_by | UUID | NOT NULL | |
| current_step | INTEGER | NOT NULL, DEFAULT 1 | |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'pending' | pending, approved, rejected, escalated |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### approval_instance_steps

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| instance_id | UUID | FK → approval_instances.id, NOT NULL | |
| step_number | INTEGER | NOT NULL | |
| assigned_to_user_id | UUID | NOT NULL | |
| action | VARCHAR(20) | NULLABLE | approved, rejected |
| acted_at | TIMESTAMPTZ | NULLABLE | |
| comments | TEXT | NULLABLE | |
| is_escalated | BOOLEAN | NOT NULL, DEFAULT false | |
| escalated_to_user_id | UUID | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

---

## Notification Service

### notifications

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| recipient_user_id | UUID | NOT NULL | |
| type | VARCHAR(20) | NOT NULL | approval, system, alert, budget_variance |
| title | VARCHAR(255) | NOT NULL | |
| body | TEXT | NOT NULL | |
| reference_entity_type | VARCHAR(50) | NULLABLE | e.g., 'journal_entry', 'ap_invoice' |
| reference_entity_id | UUID | NULLABLE | |
| is_read | BOOLEAN | NOT NULL, DEFAULT false | |
| read_at | TIMESTAMPTZ | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Indexes**: `INDEX(recipient_user_id, is_read, created_at DESC)` for efficient polling query.

---

## Cross-Cutting: Security Audit Log

*Each service maintains its own `audit_log` table in the tenant's database.*

### audit_log

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PK | |
| user_id | UUID | NOT NULL | Acting user |
| action | VARCHAR(20) | NOT NULL | CREATE, READ, UPDATE, DELETE |
| entity_type | VARCHAR(100) | NOT NULL | Table/entity name |
| entity_id | UUID | NOT NULL | |
| changes | JSONB | NULLABLE | `{"field": {"old": "x", "new": "y"}}` for updates |
| ip_address | VARCHAR(45) | NULLABLE | |
| user_agent | VARCHAR(512) | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Retention**: 7 years per clarified policy. Partitioned by `created_at` (monthly partitions) for efficient purge.

---

## Entity Relationship Summary

```
fusion_platform (identity service):
  tenants ← users ← refresh_tokens
                  ← password_reset_tokens

GL Service:
  segment_configs ← chart_of_accounts (parent self-ref)
  financial_periods ← journal_entries ← journal_entry_lines → chart_of_accounts
  journal_entries → journal_entry_source (audit trail)

AP Service:
  vendors ← ap_invoices ← ap_invoice_lines
  ap_invoices ← ap_payment_invoices → ap_payments
  ap_payments (self: batch_id groups)

AR Service:
  customers ← ar_invoices ← ar_invoice_lines
  ar_invoices ← ar_receipt_invoices → ar_receipts → customers

Procurement Service:
  purchase_requisitions ← purchase_requisition_lines
  purchase_requisitions → purchase_orders ← purchase_order_lines
  purchase_orders → goods_receipts ← goods_receipt_lines → purchase_order_lines

Budget Service:
  budgets ← budget_lines → budget_variance_alerts

Consolidation Service:
  legal_entities (parent self-ref) ← consolidation_runs ← consolidation_run_entities
  exchange_rates (from/to currency)

Fixed Asset Service:
  fixed_assets ← depreciation_run_lines → depreciation_runs
  fixed_assets → asset_disposals

Tax Service:
  tax_codes

Workflow Service:
  approval_workflows ← approval_workflow_steps
  approval_workflows ← approval_instances ← approval_instance_steps

Notification Service:
  notifications (per-user, per-entity)

Cross-cutting:
  audit_log (per-service)
```
