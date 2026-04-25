# Data Model: Oracle Fusion Cloud ERP Clone

**Date**: 2026-04-25 | **Branch**: `001-oracle-fusion-erp-clone`

## Entity Ownership by Service

Each service owns its entities and their storage. Cross-service references use UUIDs and domain events (no foreign keys across service boundaries).

---

## General Ledger Service (`service-gl`)

### ChartOfAccount

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | Auto-generated |
| tenant_id | UUID | NOT NULL, INDEX | Tenant isolation |
| code | VARCHAR(50) | NOT NULL, UNIQUE per tenant | Account code (e.g., "1000.001") |
| name | VARCHAR(255) | NOT NULL | Account name |
| account_type | ENUM | NOT NULL | ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE |
| parent_id | UUID | FK → ChartOfAccount.id, NULLABLE | Multi-level hierarchy |
| segment_values | JSONB | NULLABLE | { "company": "01", "department": "IT", "location": "NY" } |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Soft deactivation |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### FinancialPeriod

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| name | VARCHAR(100) | NOT NULL | e.g., "January 2026" |
| start_date | DATE | NOT NULL | Period start |
| end_date | DATE | NOT NULL | Period end |
| status | ENUM | NOT NULL | OPEN, CLOSED, PERMANENTLY_CLOSED |
| period_year | INT | NOT NULL | Fiscal year |
| period_number | INT | NOT NULL | 1-12 (or 1-13) |
| closed_by | UUID | NULLABLE | User who closed |
| closed_at | TIMESTAMPTZ | NULLABLE | When closed |

**Unique**: `(tenant_id, period_year, period_number)`

### JournalEntry

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| entry_number | VARCHAR(50) | NOT NULL, UNIQUE per tenant | Auto-generated (e.g., "JE-2026-00001") |
| description | TEXT | NULLABLE | |
| entry_date | DATE | NOT NULL | Accounting date |
| period_id | UUID | FK → FinancialPeriod.id, NOT NULL | |
| source | ENUM | NOT NULL | MANUAL, AP_INVOICE, AR_INVOICE, PAYMENT, RECEIPT, DEPRECIATION, REVALUATION, CONSOLIDATION |
| source_reference | VARCHAR(255) | NULLABLE | Cross-service reference (e.g., "AP-INV-001") |
| status | ENUM | NOT NULL | DRAFT, PENDING_APPROVAL, POSTED, REVERSED |
| total_debit | DECIMAL(18,2) | NOT NULL | Computed from lines |
| total_credit | DECIMAL(18,2) | NOT NULL | Computed from lines |
| currency_code | CHAR(3) | NOT NULL, DEFAULT 'USD' | ISO 4217 |
| reversal_of_id | UUID | FK → JournalEntry.id, NULLABLE | For reversed entries |
| created_by | UUID | NOT NULL | |
| posted_by | UUID | NULLABLE | |
| posted_at | TIMESTAMPTZ | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### JournalEntryLine

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| journal_entry_id | UUID | FK → JournalEntry.id, NOT NULL | |
| line_number | INT | NOT NULL | Sequential within entry |
| account_id | UUID | FK → ChartOfAccount.id, NOT NULL | |
| description | TEXT | NULLABLE | Line description |
| debit_amount | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | |
| credit_amount | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | |

**Constraint**: `CHECK (debit_amount >= 0 AND credit_amount >= 0)` — one must be > 0 per line

---

## Accounts Payable Service (`service-ap`)

### Vendor

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| vendor_code | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| name | VARCHAR(255) | NOT NULL | |
| address | JSONB | NULLABLE | { line1, line2, city, state, postal_code, country } |
| payment_terms | VARCHAR(50) | NOT NULL, DEFAULT 'NET_30' | e.g., NET_30, NET_60, IMMEDIATE |
| tax_id | VARCHAR(100) | NULLABLE | |
| bank_details | JSONB | NULLABLE | Encrypted; { bank_name, account_number, routing_number } |
| contact_email | VARCHAR(255) | NULLABLE | |
| contact_phone | VARCHAR(50) | NULLABLE | |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### ApInvoice

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| invoice_number | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| vendor_id | UUID | FK → Vendor.id, NOT NULL | |
| invoice_date | DATE | NOT NULL | |
| due_date | DATE | NOT NULL | |
| description | TEXT | NULLABLE | |
| subtotal | DECIMAL(18,2) | NOT NULL | Before tax |
| tax_amount | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | |
| total_amount | DECIMAL(18,2) | NOT NULL | subtotal + tax_amount |
| currency_code | CHAR(3) | NOT NULL, DEFAULT 'USD' | |
| status | ENUM | NOT NULL | DRAFT, PENDING_APPROVAL, APPROVED, POSTED, PAID, CANCELLED |
| payment_status | ENUM | NOT NULL | UNPAID, PARTIALLY_PAID, PAID |
| paid_amount | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | |
| po_reference | VARCHAR(50) | NULLABLE | PO number for 3-way match |
| gl_journal_entry_id | UUID | NULLABLE | Cross-service ref to GL journal entry |
| approved_by | UUID | NULLABLE | |
| created_by | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### ApInvoiceLine

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL | |
| invoice_id | UUID | FK → ApInvoice.id, NOT NULL | |
| line_number | INT | NOT NULL | |
| description | TEXT | NOT NULL | |
| quantity | DECIMAL(18,4) | NOT NULL | |
| unit_price | DECIMAL(18,2) | NOT NULL | |
| line_total | DECIMAL(18,2) | NOT NULL | quantity × unit_price |
| tax_code | VARCHAR(50) | NULLABLE | Reference to tax service |
| gl_account_id | UUID | NOT NULL | Expense account |

### Payment

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| payment_number | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| vendor_id | UUID | FK → Vendor.id, NOT NULL | |
| payment_date | DATE | NOT NULL | |
| amount | DECIMAL(18,2) | NOT NULL | |
| currency_code | CHAR(3) | NOT NULL | |
| payment_method | ENUM | NOT NULL | CHECK, WIRE_TRANSFER, ACH, CASH |
| bank_account | VARCHAR(100) | NULLABLE | Originating bank account |
| status | ENUM | NOT NULL | DRAFT, ISSUED, RECONCILED, CANCELLED |
| gl_journal_entry_id | UUID | NULLABLE | Cross-service ref |
| created_by | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### PaymentInvoiceAllocation

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| payment_id | UUID | FK → Payment.id, NOT NULL | |
| invoice_id | UUID | FK → ApInvoice.id, NOT NULL | |
| amount | DECIMAL(18,2) | NOT NULL | Amount applied to this invoice |

---

## Accounts Receivable Service (`service-ar`)

### Customer

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| customer_code | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| name | VARCHAR(255) | NOT NULL | |
| address | JSONB | NULLABLE | Same structure as Vendor |
| payment_terms | VARCHAR(50) | NOT NULL, DEFAULT 'NET_30' | |
| credit_limit | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | 0 = unlimited |
| tax_id | VARCHAR(100) | NULLABLE | |
| contact_email | VARCHAR(255) | NULLABLE | |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | |
| on_credit_hold | BOOLEAN | NOT NULL, DEFAULT false | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### ArInvoice

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| invoice_number | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| customer_id | UUID | FK → Customer.id, NOT NULL | |
| invoice_date | DATE | NOT NULL | |
| due_date | DATE | NOT NULL | |
| subtotal | DECIMAL(18,2) | NOT NULL | |
| tax_amount | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | |
| total_amount | DECIMAL(18,2) | NOT NULL | |
| currency_code | CHAR(3) | NOT NULL | |
| status | ENUM | NOT NULL | DRAFT, SENT, POSTED, PARTIALLY_PAID, PAID, CANCELLED |
| payment_status | ENUM | NOT NULL | UNPAID, PARTIALLY_PAID, PAID |
| paid_amount | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | |
| gl_journal_entry_id | UUID | NULLABLE | Cross-service ref |
| created_by | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### ArInvoiceLine

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL | |
| invoice_id | UUID | FK → ArInvoice.id, NOT NULL | |
| line_number | INT | NOT NULL | |
| description | TEXT | NOT NULL | |
| quantity | DECIMAL(18,4) | NOT NULL | |
| unit_price | DECIMAL(18,2) | NOT NULL | |
| line_total | DECIMAL(18,2) | NOT NULL | |
| tax_code | VARCHAR(50) | NULLABLE | |
| gl_account_id | UUID | NOT NULL | Revenue account |

### Receipt

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| receipt_number | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| customer_id | UUID | FK → Customer.id, NOT NULL | |
| receipt_date | DATE | NOT NULL | |
| amount | DECIMAL(18,2) | NOT NULL | |
| currency_code | CHAR(3) | NOT NULL | |
| payment_method | ENUM | NOT NULL | CHECK, WIRE_TRANSFER, CASH, CREDIT_CARD |
| reference | VARCHAR(255) | NULLABLE | Bank reference, check number |
| status | ENUM | NOT NULL | UNAPPLIED, PARTIALLY_APPLIED, APPLIED |
| unapplied_amount | DECIMAL(18,2) | NOT NULL | Remaining unallocated amount |
| gl_journal_entry_id | UUID | NULLABLE | Cross-service ref |
| created_by | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### ReceiptInvoiceAllocation

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| receipt_id | UUID | FK → Receipt.id, NOT NULL | |
| invoice_id | UUID | FK → ArInvoice.id, NOT NULL | |
| amount | DECIMAL(18,2) | NOT NULL | |

---

## Procurement Service (`service-procurement`)

### PurchaseRequisition

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| requisition_number | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| requester_id | UUID | NOT NULL | User who requested |
| department | VARCHAR(100) | NULLABLE | |
| description | TEXT | NULLABLE | |
| status | ENUM | NOT NULL | DRAFT, PENDING_APPROVAL, APPROVED, REJECTED, CONVERTED, CANCELLED |
| total_estimated | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | |
| approved_by | UUID | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### RequisitionLine

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| requisition_id | UUID | FK → PurchaseRequisition.id, NOT NULL | |
| line_number | INT | NOT NULL | |
| description | TEXT | NOT NULL | |
| quantity | DECIMAL(18,4) | NOT NULL | |
| estimated_unit_price | DECIMAL(18,2) | NULLABLE | |
| category | VARCHAR(100) | NULLABLE | Procurement category |
| preferred_vendor_id | UUID | NULLABLE | Cross-service ref to AP Vendor |

### PurchaseOrder

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| po_number | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| vendor_id | UUID | NOT NULL | Cross-service ref to AP Vendor |
| requisition_id | UUID | FK → PurchaseRequisition.id, NULLABLE | |
| order_date | DATE | NOT NULL | |
| expected_delivery_date | DATE | NULLABLE | |
| status | ENUM | NOT NULL | DRAFT, ISSUED, PARTIALLY_RECEIVED, RECEIVED, CLOSED, CANCELLED |
| subtotal | DECIMAL(18,2) | NOT NULL | |
| tax_amount | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | |
| total_amount | DECIMAL(18,2) | NOT NULL | |
| currency_code | CHAR(3) | NOT NULL | |
| created_by | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### PurchaseOrderLine

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| po_id | UUID | FK → PurchaseOrder.id, NOT NULL | |
| line_number | INT | NOT NULL | |
| description | TEXT | NOT NULL | |
| quantity_ordered | DECIMAL(18,4) | NOT NULL | |
| quantity_received | DECIMAL(18,4) | NOT NULL, DEFAULT 0 | |
| unit_price | DECIMAL(18,2) | NOT NULL | |
| line_total | DECIMAL(18,2) | NOT NULL | quantity_ordered × unit_price |

### GoodsReceipt

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| receipt_number | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| po_id | UUID | FK → PurchaseOrder.id, NOT NULL | |
| received_date | DATE | NOT NULL | |
| received_by | UUID | NOT NULL | |
| status | ENUM | NOT NULL | DRAFT, CONFIRMED | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### GoodsReceiptLine

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| goods_receipt_id | UUID | FK → GoodsReceipt.id, NOT NULL | |
| po_line_id | UUID | FK → PurchaseOrderLine.id, NOT NULL | |
| quantity_received | DECIMAL(18,4) | NOT NULL | |
| condition | ENUM | NOT NULL | ACCEPTED, DAMAGED, REJECTED |

---

## Identity & Access Service (`service-identity`)

### User

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| email | VARCHAR(255) | NOT NULL, UNIQUE per tenant | |
| password_hash | VARCHAR(255) | NOT NULL | Argon2 hash |
| full_name | VARCHAR(255) | NOT NULL | |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | |
| last_login_at | TIMESTAMPTZ | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### Role

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| name | VARCHAR(100) | NOT NULL, UNIQUE per tenant | e.g., "AP Clerk", "GL Manager", "CFO" |
| description | TEXT | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### Permission

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| role_id | UUID | FK → Role.id, NOT NULL | |
| module | VARCHAR(50) | NOT NULL | e.g., "gl", "ap", "ar" |
| action | VARCHAR(50) | NOT NULL | e.g., "create", "read", "update", "delete", "post", "approve" |
| scope | ENUM | NOT NULL | OWN, DEPARTMENT, ALL |

**Unique**: `(role_id, module, action, scope)`

### UserRole

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| user_id | UUID | FK → User.id, NOT NULL | |
| role_id | UUID | FK → Role.id, NOT NULL | |

**PK**: `(user_id, role_id)`

### RefreshToken

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| user_id | UUID | FK → User.id, NOT NULL | |
| token_hash | VARCHAR(255) | NOT NULL | SHA-256 hash of token |
| expires_at | TIMESTAMPTZ | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| revoked_at | TIMESTAMPTZ | NULLABLE | |

### AuditLog

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| user_id | UUID | NOT NULL | |
| action | ENUM | NOT NULL | CREATE, READ, UPDATE, DELETE, LOGIN, LOGOUT |
| module | VARCHAR(50) | NOT NULL | |
| entity_type | VARCHAR(100) | NOT NULL | |
| entity_id | UUID | NOT NULL | |
| changes | JSONB | NULLABLE | { "field": "status", "old": "DRAFT", "new": "POSTED" } |
| ip_address | INET | NULLABLE | |
| timestamp | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

**Append-only**: No UPDATE or DELETE allowed on this table

---

## Workflow Service (`service-workflow`)

### ApprovalWorkflow

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| name | VARCHAR(255) | NOT NULL | |
| module | VARCHAR(50) | NOT NULL | e.g., "gl", "ap", "procurement" |
| trigger_type | VARCHAR(100) | NOT NULL | e.g., "journal_entry", "invoice", "purchase_order" |
| condition | JSONB | NOT NULL | { "amount_gt": 10000, "department": "IT" } |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### ApprovalStep

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| workflow_id | UUID | FK → ApprovalWorkflow.id, NOT NULL | |
| step_order | INT | NOT NULL | Sequential order |
| approver_type | ENUM | NOT NULL | USER, ROLE, MANAGER_OF_SUBMITTER |
| approver_id | UUID | NULLABLE | User or Role ID (if type is USER or ROLE) |
| timeout_minutes | INT | NULLABLE | Escalation timeout |
| escalation_approver_id | UUID | NULLABLE | Fallback approver after timeout |

**Unique**: `(workflow_id, step_order)`

### ApprovalInstance

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| workflow_id | UUID | FK → ApprovalWorkflow.id, NOT NULL | |
| entity_type | VARCHAR(100) | NOT NULL | e.g., "journal_entry" |
| entity_id | UUID | NOT NULL | The transaction being approved |
| current_step | INT | NOT NULL | Current step in the chain |
| status | ENUM | NOT NULL | PENDING, APPROVED, REJECTED, ESCALATED, CANCELLED |
| submitted_by | UUID | NOT NULL | |
| submitted_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| completed_at | TIMESTAMPTZ | NULLABLE | |

### ApprovalAction

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| instance_id | UUID | FK → ApprovalInstance.id, NOT NULL | |
| step_number | INT | NOT NULL | |
| approver_id | UUID | NOT NULL | |
| action | ENUM | NOT NULL | APPROVE, REJECT, DELEGATE, ESCALATE |
| delegated_to | UUID | NULLABLE | If action is DELEGATE |
| comment | TEXT | NULLABLE | |
| acted_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

---

## Notification Service (`service-notification`)

### Notification

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| user_id | UUID | NOT NULL, INDEX | Recipient |
| type | ENUM | NOT NULL | APPROVAL_REQUIRED, APPROVAL_COMPLETED, APPROVAL_ESCALATED, SYSTEM_ALERT, PERIOD_CLOSE |
| title | VARCHAR(255) | NOT NULL | |
| body | TEXT | NOT NULL | |
| entity_type | VARCHAR(100) | NULLABLE | Related entity |
| entity_id | UUID | NULLABLE | Related entity ID |
| channel | ENUM | NOT NULL | IN_APP, EMAIL, BOTH |
| is_read | BOOLEAN | NOT NULL, DEFAULT false | |
| read_at | TIMESTAMPTZ | NULLABLE | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

---

## Reporting Service (`service-reporting`)

### SavedReport

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| name | VARCHAR(255) | NOT NULL | |
| report_type | ENUM | NOT NULL | INCOME_STATEMENT, BALANCE_SHEET, CASH_FLOW, TRIAL_BALANCE, AGING, CUSTOM |
| definition | JSONB | NOT NULL | { fields, filters, groupings, sort } |
| created_by | UUID | NOT NULL | |
| is_shared | BOOLEAN | NOT NULL, DEFAULT false | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

*Note*: Standard reports (income statement, balance sheet, cash flow, trial balance, aging) are generated dynamically from GL data via gRPC calls — no persistent storage needed.

---

## Consolidation Service (`service-consolidation`)

### LegalEntity

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| entity_code | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| name | VARCHAR(255) | NOT NULL | |
| base_currency | CHAR(3) | NOT NULL | ISO 4217 |
| parent_entity_id | UUID | FK → LegalEntity.id, NULLABLE | For hierarchy |
| fiscal_year_start_month | INT | NOT NULL, DEFAULT 1 | |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

### ExchangeRate

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL | |
| from_currency | CHAR(3) | NOT NULL | |
| to_currency | CHAR(3) | NOT NULL | |
| rate | DECIMAL(18,6) | NOT NULL | |
| effective_date | DATE | NOT NULL | |
| source | VARCHAR(100) | NULLABLE | e.g., "manual", "ECB" |

**Unique**: `(tenant_id, from_currency, to_currency, effective_date)`

### ConsolidationRun

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL | |
| period_year | INT | NOT NULL | |
| period_number | INT | NOT NULL | |
| reporting_currency | CHAR(3) | NOT NULL | Target currency |
| status | ENUM | NOT NULL | RUNNING, COMPLETED, FAILED |
| started_at | TIMESTAMPTZ | NOT NULL | |
| completed_at | TIMESTAMPTZ | NULLABLE | |
| initiated_by | UUID | NOT NULL | |

---

## Budget Service (`service-budget`)

### Budget

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| name | VARCHAR(255) | NOT NULL | |
| fiscal_year | INT | NOT NULL | |
| department | VARCHAR(100) | NULLABLE | |
| status | ENUM | NOT NULL | DRAFT, APPROVED, ACTIVE, CLOSED |
| total_amount | DECIMAL(18,2) | NOT NULL | Sum of all lines |
| approved_by | UUID | NULLABLE | |
| created_by | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### BudgetLine

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| budget_id | UUID | FK → Budget.id, NOT NULL | |
| gl_account_id | UUID | NOT NULL | Cross-service ref to GL account |
| period_number | INT | NOT NULL | 1-12 |
| budgeted_amount | DECIMAL(18,2) | NOT NULL | |
| actual_amount | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | Updated via GL events |

**Unique**: `(budget_id, gl_account_id, period_number)`

---

## Tax Service (`service-tax`)

### TaxRate

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| tax_code | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| name | VARCHAR(255) | NOT NULL | |
| jurisdiction | VARCHAR(100) | NOT NULL | |
| rate_percentage | DECIMAL(8,4) | NOT NULL | |
| tax_category | VARCHAR(50) | NULLABLE | e.g., "sales", "vat", "gst" |
| effective_from | DATE | NOT NULL | |
| effective_to | DATE | NULLABLE | NULL = currently active |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | |

### TaxTransaction

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL | |
| tax_rate_id | UUID | FK → TaxRate.id, NOT NULL | |
| transaction_type | ENUM | NOT NULL | PAYABLE, RECEIVABLE |
| entity_id | UUID | NOT NULL | Invoice ID |
| tax_base_amount | DECIMAL(18,2) | NOT NULL | |
| tax_amount | DECIMAL(18,2) | NOT NULL | |
| transaction_date | DATE | NOT NULL | |
| period_year | INT | NOT NULL | |
| period_number | INT | NOT NULL | |

---

## Fixed Asset Service (`service-asset`)

### FixedAsset

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL, INDEX | |
| asset_code | VARCHAR(50) | NOT NULL, UNIQUE per tenant | |
| description | VARCHAR(255) | NOT NULL | |
| acquisition_date | DATE | NOT NULL | |
| acquisition_cost | DECIMAL(18,2) | NOT NULL | |
| useful_life_months | INT | NOT NULL | |
| depreciation_method | ENUM | NOT NULL | STRAIGHT_LINE, DECLINING_BALANCE |
| declining_balance_rate | DECIMAL(5,4) | NULLABLE | For declining balance method |
| salvage_value | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | |
| accumulated_depreciation | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | |
| net_book_value | DECIMAL(18,2) | NOT NULL | cost - accumulated_depreciation |
| location | VARCHAR(255) | NULLABLE | Physical location |
| status | ENUM | NOT NULL | ACTIVE, DISPOSED, FULLY_DEPRECIATED |
| gl_asset_account_id | UUID | NOT NULL | Cross-service ref |
| gl_depreciation_account_id | UUID | NOT NULL | Cross-service ref |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |
| updated_at | TIMESTAMPTZ | NOT NULL | |

### DepreciationEntry

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL | |
| asset_id | UUID | FK → FixedAsset.id, NOT NULL | |
| period_year | INT | NOT NULL | |
| period_number | INT | NOT NULL | |
| depreciation_amount | DECIMAL(18,2) | NOT NULL | |
| gl_journal_entry_id | UUID | NULLABLE | Cross-service ref |
| posted_at | TIMESTAMPTZ | NULLABLE | |

**Unique**: `(asset_id, period_year, period_number)`

### AssetDisposal

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | UUID | PK | |
| tenant_id | UUID | NOT NULL | |
| asset_id | UUID | FK → FixedAsset.id, NOT NULL | |
| disposal_date | DATE | NOT NULL | |
| disposal_proceeds | DECIMAL(18,2) | NOT NULL | Amount received |
| net_book_value_at_disposal | DECIMAL(18,2) | NOT NULL | NBV at time of disposal |
| gain_loss_amount | DECIMAL(18,2) | NOT NULL | proceeds - NBV |
| gl_journal_entry_id | UUID | NULLABLE | Cross-service ref |
| created_by | UUID | NOT NULL | |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() | |

---

## Cross-Service Conventions

1. **All tables** include `tenant_id` (UUID) for multi-tenancy
2. **All monetary fields** use `DECIMAL(18,2)` unless higher precision needed (exchange rates use `DECIMAL(18,6)`)
3. **All cross-service references** use UUID fields with naming convention `{entity}_id` — no foreign keys across service boundaries
4. **All tables** include `created_at` (TIMESTAMPTZ); mutable tables also include `updated_at`
5. **Status enums** follow a lifecycle pattern: `DRAFT → … → POSTED/COMPLETED`
6. **Soft delete**: Prefer status changes (e.g., `CANCELLED`, `is_active = false`) over hard delete
7. **Audit trail**: The identity service's `AuditLog` captures changes across all services via NATS event subscriptions
