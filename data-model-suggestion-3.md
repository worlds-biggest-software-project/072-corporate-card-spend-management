# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Corporate Card & Spend Management · Created: 2026-05-12

## Philosophy

This model uses a pragmatic hybrid approach: core structural data (organizations, users, cards, transactions) lives in strongly-typed relational columns with foreign keys, while variable, jurisdiction-specific, and rapidly-evolving data lives in JSONB columns with GIN indexes. The philosophy is "relational where it matters, flexible where it varies."

The hybrid pattern is widely used in modern SaaS platforms that must serve diverse customer configurations without schema migrations for every new use case. Stripe's internal data model, Shopify's metafields, and Salesforce's custom fields all follow variants of this pattern. The key insight is that in spend management, the structural relationships (card belongs to cardholder, transaction references card, expense coding maps to GL account) are stable, but the details vary enormously: different organizations have different policy rules, different GL structures, different approval chains, different compliance requirements per jurisdiction.

This is the ideal model for a startup building an MVP that needs to ship fast, support diverse customer configurations without per-customer schema changes, and iterate on features without migration-heavy releases. It reduces the table count significantly compared to a fully normalized model while preserving referential integrity on the critical paths.

**Best for:** Rapid MVP development, multi-region platforms with jurisdiction-specific requirements, teams that want relational integrity without migration overhead, and products where customer configuration varies widely.

**Trade-offs:**
- (+) Dramatically fewer tables than full normalization — faster to develop, deploy, and understand
- (+) JSONB columns absorb customer-specific and jurisdiction-specific variation without migrations
- (+) GIN indexes on JSONB enable efficient queries on dynamic fields
- (+) Schema evolution is painless for JSONB fields — add new fields without ALTER TABLE
- (+) PostgreSQL JSONB operators are mature and well-optimized
- (-) No foreign key constraints inside JSONB — application must enforce referential integrity for dynamic fields
- (-) Complex JSONB queries can be slower than indexed relational columns for high-cardinality data
- (-) Reporting tools and BI platforms handle JSONB less gracefully than flat columns
- (-) JSONB field documentation lives in code/comments, not in the schema itself
- (-) Risk of JSONB columns becoming unstructured "junk drawers" without discipline

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 4217 | `currency_code CHAR(3)` as relational column on all monetary tables |
| ISO 18245 (MCC) | `mcc_code CHAR(4)` as relational column on transactions; full MCC data in `merchant_data` JSONB |
| ISO 3166-1 | `country_code CHAR(2)` relational; jurisdiction-specific rules in `jurisdiction_config` JSONB |
| PCI DSS 4.0.1 | Card tokens only — PAN never stored; JSONB fields encrypted at column level for sensitive metadata |
| NACHA ACH | Reimbursement payment details in `payment_details` JSONB with NACHA-specific fields |
| ISO 20022 | Cross-border payment metadata in `payment_details` JSONB structured for `pain.001` mapping |
| OpenAPI 3.1 | JSONB column structures documented as JSON Schema definitions in API spec |
| OAuth 2.0 | Integration credentials in `config` JSONB on integrations table |

---

## Core Tables

```sql
-- ============================================================
-- ORGANIZATIONS (with JSONB settings)
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    country_code    CHAR(2) NOT NULL,              -- ISO 3166-1
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',-- ISO 4217
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','suspended','closed')),

    -- JSONB: org-level settings that vary by customer
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "receipt_required_above": 25.00,
    --   "auto_lock_on_termination": true,
    --   "default_approval_chain": "manager_then_finance",
    --   "fiscal_year_start_month": 1,
    --   "enabled_integrations": ["quickbooks", "slack"],
    --   "branding": {"logo_url": "...", "primary_color": "#1a73e8"}
    -- }

    -- JSONB: jurisdiction-specific compliance configuration
    jurisdiction_config JSONB NOT NULL DEFAULT '{}',
    -- Example jurisdiction_config:
    -- {
    --   "US": {
    --     "1099_threshold": 600.00,
    --     "w9_required_for_vendors": true,
    --     "state_tax_rules": {"CA": {"use_tax_rate": 0.0725}}
    --   },
    --   "EU": {
    --     "vat_reclaim_enabled": true,
    --     "gdpr_data_retention_months": 84,
    --     "sepa_enabled": true
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_settings ON organizations USING gin(settings);

-- Multi-entity support as JSONB array on org (simple) or separate table (complex)
-- For MVP: entities as JSONB; for scale: promote to separate table
CREATE TABLE entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    country_code    CHAR(2) NOT NULL,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    parent_id       UUID REFERENCES entities(id),
    config          JSONB NOT NULL DEFAULT '{}',    -- entity-specific overrides
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_entities_org ON entities(organization_id);
```

---

## Users & Access

```sql
-- ============================================================
-- USERS (roles and permissions in JSONB)
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    employee_id     TEXT,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','inactive','terminated')),

    -- JSONB: role and org-structure assignments
    -- Replaces 3 separate tables (roles, user_roles, departments)
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "roles": ["employee", "budget_owner"],
    --   "department": {"id": "uuid", "name": "Engineering", "code": "ENG"},
    --   "cost_center": {"id": "uuid", "code": "CC-100"},
    --   "manager_id": "uuid",
    --   "manager_name": "Jane Smith",
    --   "entity_id": "uuid",
    --   "hire_date": "2024-03-15",
    --   "permissions": ["submit_expense", "approve_under_500", "view_team_budgets"],
    --   "bank_account": {
    --     "routing_number_token": "tok_xxx",
    --     "account_number_token": "tok_yyy",
    --     "account_type": "checking"
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);
CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_roles ON users USING gin((profile->'roles'));
CREATE INDEX idx_users_department ON users((profile->>'department'));
CREATE INDEX idx_users_manager ON users(((profile->>'manager_id')::UUID));

-- Row-Level Security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON users
    USING (organization_id = current_setting('app.current_org_id')::UUID);
```

---

## Cards

```sql
-- ============================================================
-- CARDS (core fields relational, config in JSONB)
-- ============================================================

CREATE TABLE cards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    cardholder_id   UUID NOT NULL REFERENCES users(id),
    card_token      TEXT NOT NULL,                  -- tokenized ref (PCI-compliant)
    issuer_card_id  TEXT NOT NULL,
    issuer          TEXT NOT NULL
                    CHECK (issuer IN ('stripe_issuing','lithic','marqeta')),
    card_type       TEXT NOT NULL
                    CHECK (card_type IN ('physical','virtual')),
    last_four       CHAR(4),
    network         TEXT CHECK (network IN ('visa','mastercard')),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','frozen','cancelled','expired')),

    -- JSONB: spend controls and card configuration
    -- Replaces separate spend_limits and card_restrictions tables
    controls        JSONB NOT NULL DEFAULT '{}',
    -- Example controls:
    -- {
    --   "spend_limit": {"amount": 5000.00, "interval": "monthly"},
    --   "purpose": "vendor_locked",
    --   "locked_vendor": {"id": "uuid", "name": "AWS"},
    --   "locked_project": {"id": "uuid", "name": "Infrastructure"},
    --   "budget_id": "uuid",
    --   "allowed_mcc_codes": ["5734", "5045", "7372"],
    --   "blocked_mcc_codes": ["5813", "7995"],
    --   "allowed_countries": ["US", "CA", "GB"],
    --   "single_use": false,
    --   "auto_close_after_days": null
    -- }

    activated_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cards_org ON cards(organization_id);
CREATE INDEX idx_cards_cardholder ON cards(cardholder_id);
CREATE INDEX idx_cards_status ON cards(organization_id, status);
CREATE INDEX idx_cards_issuer ON cards(issuer_card_id);
CREATE INDEX idx_cards_controls ON cards USING gin(controls);
```

---

## Transactions

```sql
-- ============================================================
-- TRANSACTIONS (core fields relational, enrichment in JSONB)
-- ============================================================

CREATE TABLE transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_id       UUID REFERENCES entities(id),
    card_id         UUID NOT NULL REFERENCES cards(id),
    cardholder_id   UUID NOT NULL REFERENCES users(id),

    -- Core monetary fields (always relational for aggregation)
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    billing_amount  NUMERIC(15,2),
    billing_currency CHAR(3),
    fx_rate         NUMERIC(12,6),

    -- Merchant (core fields relational for filtering/grouping)
    merchant_name   TEXT,
    mcc_code        CHAR(4),

    -- Status
    transaction_type TEXT NOT NULL
                    CHECK (transaction_type IN ('purchase','refund','reversal','fee')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','cleared','declined','reversed','disputed')),

    -- Timestamps
    authorized_at   TIMESTAMPTZ,
    cleared_at      TIMESTAMPTZ,

    -- JSONB: enriched merchant data, issuer details, network data
    merchant_data   JSONB NOT NULL DEFAULT '{}',
    -- Example merchant_data:
    -- {
    --   "merchant_id": "merch_abc123",
    --   "clean_name": "Amazon Web Services",
    --   "logo_url": "https://...",
    --   "website": "aws.amazon.com",
    --   "mcc_description": "Computer Software Stores",
    --   "address": {
    --     "street": "410 Terry Ave",
    --     "city": "Seattle",
    --     "state": "WA",
    --     "country": "US",
    --     "postal_code": "98109"
    --   },
    --   "category_group": "software"
    -- }

    -- JSONB: issuer-specific authorization details
    issuer_data     JSONB NOT NULL DEFAULT '{}',
    -- Example issuer_data:
    -- {
    --   "authorization_id": "iauth_abc123",
    --   "transaction_id": "itxn_abc123",
    --   "authorization_method": "online",
    --   "network_data": {"request_id": "...", "response_code": "0000"},
    --   "token": "tok_visa_xxx",
    --   "wallet": "apple_pay",
    --   "verification": {"cvv_check": "match", "address_check": "match"}
    -- }

    -- JSONB: GL coding (replaces separate expense_codings table for simple cases)
    coding          JSONB NOT NULL DEFAULT '{}',
    -- Example coding (single):
    -- {
    --   "gl_account": {"code": "6200", "name": "Software & SaaS"},
    --   "department": {"code": "ENG", "name": "Engineering"},
    --   "cost_center": {"code": "CC-100"},
    --   "project": {"code": "P-INFRA", "name": "Infrastructure"},
    --   "coding_method": "ai_auto",
    --   "ai_confidence": 0.94,
    --   "reviewed_by": null,
    --   "reviewed_at": null
    -- }
    -- Example coding (split):
    -- {
    --   "is_split": true,
    --   "lines": [
    --     {"gl_account": {"code": "6200"}, "amount": 100.00, "memo": "Compute"},
    --     {"gl_account": {"code": "6210"}, "amount": 25.50, "memo": "Storage"}
    --   ],
    --   "coding_method": "manual"
    -- }

    -- JSONB: custom dimensions (replaces expense_coding_dimensions table)
    custom_dimensions JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "class": {"code": "OPEX", "name": "Operating Expense"},
    --   "location": {"code": "SF", "name": "San Francisco"},
    --   "custom_field_1": "value"
    -- }

    -- JSONB: policy evaluation results
    policy_results  JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "evaluated_at": "2026-05-12T10:30:00Z",
    --   "violations": [
    --     {
    --       "policy_id": "uuid",
    --       "policy_name": "Travel > $250 requires approval",
    --       "severity": "warning",
    --       "action": "require_approval",
    --       "resolution": "approved_exception",
    --       "resolved_by": "uuid",
    --       "resolved_at": "2026-05-12T11:00:00Z"
    --     }
    --   ],
    --   "ai_review_score": 0.87,
    --   "compliant": false
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core relational indexes
CREATE INDEX idx_txn_org ON transactions(organization_id);
CREATE INDEX idx_txn_card ON transactions(card_id);
CREATE INDEX idx_txn_cardholder ON transactions(cardholder_id);
CREATE INDEX idx_txn_status ON transactions(organization_id, status);
CREATE INDEX idx_txn_date ON transactions(organization_id, authorized_at);
CREATE INDEX idx_txn_mcc ON transactions(mcc_code);

-- JSONB indexes for common query patterns
CREATE INDEX idx_txn_coding_gl ON transactions
    USING gin((coding->'gl_account'));
CREATE INDEX idx_txn_coding_dept ON transactions
    ((coding->>'department'));
CREATE INDEX idx_txn_merchant_data ON transactions
    USING gin(merchant_data);
CREATE INDEX idx_txn_violations ON transactions
    USING gin((policy_results->'violations'))
    WHERE policy_results->'violations' IS NOT NULL
      AND jsonb_array_length(policy_results->'violations') > 0;

-- Example query: find all transactions coded to GL 6200 in Engineering
-- SELECT * FROM transactions
-- WHERE organization_id = $1
--   AND coding->'gl_account'->>'code' = '6200'
--   AND coding->'department'->>'code' = 'ENG'
--   AND authorized_at >= '2026-04-01'
-- ORDER BY authorized_at DESC;

-- Example query: find transactions with unresolved policy violations
-- SELECT * FROM transactions
-- WHERE organization_id = $1
--   AND policy_results->'violations' @> '[{"resolution": "pending"}]'
-- ORDER BY authorized_at DESC;
```

---

## Budgets

```sql
-- ============================================================
-- BUDGETS (hybrid: core limits relational, rules in JSONB)
-- ============================================================

CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_id       UUID REFERENCES entities(id),
    name            TEXT NOT NULL,
    owner_id        UUID NOT NULL REFERENCES users(id),

    -- Core budget fields (relational for aggregation)
    limit_amount    NUMERIC(15,2) NOT NULL,
    spent_amount    NUMERIC(15,2) NOT NULL DEFAULT 0,
    committed_amount NUMERIC(15,2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    period          TEXT NOT NULL
                    CHECK (period IN ('monthly','quarterly','yearly','one_time','custom')),
    period_start    DATE,
    period_end      DATE,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','paused','closed','exhausted')),

    -- JSONB: budget configuration and rules
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "budget_type": "department",
    --   "department": {"id": "uuid", "code": "ENG", "name": "Engineering"},
    --   "project": null,
    --   "is_pre_funded": false,
    --   "funded_amount": 0,
    --   "alert_thresholds": [
    --     {"pct": 75, "notify": ["owner"]},
    --     {"pct": 90, "notify": ["owner", "finance"]},
    --     {"pct": 100, "action": "freeze_cards"}
    --   ],
    --   "allowed_categories": ["software", "travel", "office"],
    --   "blocked_mcc_codes": ["7995"],
    --   "rollover_unused": false,
    --   "child_budget_ids": ["uuid1", "uuid2"]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_budgets_org ON budgets(organization_id);
CREATE INDEX idx_budgets_owner ON budgets(owner_id);
CREATE INDEX idx_budgets_status ON budgets(organization_id, status);
CREATE INDEX idx_budgets_config ON budgets USING gin(config);
CREATE INDEX idx_budgets_period ON budgets(organization_id, period_start, period_end);
```

---

## Policies

```sql
-- ============================================================
-- POLICIES (entire policy engine in JSONB)
-- ============================================================

CREATE TABLE policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    description     TEXT,
    priority        INT NOT NULL DEFAULT 100,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: policy rules and conditions
    -- The entire policy rule set lives in JSONB, making the policy engine
    -- data-driven without requiring separate rules/conditions tables
    rules           JSONB NOT NULL,
    -- Example rules:
    -- {
    --   "conditions": [
    --     {"field": "amount", "operator": "greater_than", "value": 250.00},
    --     {"field": "mcc_code", "operator": "in", "value": ["5812", "5813"]}
    --   ],
    --   "condition_logic": "all",
    --   "action": "require_approval",
    --   "action_config": {
    --     "approval_chain": ["direct_manager", "finance_manager"],
    --     "auto_approve_below": 50.00,
    --     "escalation_hours": 48
    --   }
    -- }

    -- JSONB: scope (who this policy applies to)
    scope           JSONB NOT NULL DEFAULT '{"applies_to_all": true}',
    -- Example scope:
    -- {
    --   "applies_to_all": false,
    --   "departments": ["ENG", "MKT"],
    --   "roles": ["employee"],
    --   "entities": ["uuid1"],
    --   "exclude_users": ["uuid_cfo"]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_policies_org ON policies(organization_id);
CREATE INDEX idx_policies_active ON policies(organization_id)
    WHERE is_active = true;
CREATE INDEX idx_policies_rules ON policies USING gin(rules);
CREATE INDEX idx_policies_scope ON policies USING gin(scope);
```

---

## Receipts & Reimbursements

```sql
-- ============================================================
-- RECEIPTS
-- ============================================================

CREATE TABLE receipts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    transaction_id  UUID REFERENCES transactions(id),
    reimbursement_id UUID,                          -- FK added below
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    file_url        TEXT NOT NULL,
    file_type       TEXT,
    file_size_bytes BIGINT,
    source          TEXT NOT NULL DEFAULT 'upload'
                    CHECK (source IN ('upload','email','sms','slack','teams','auto_generated')),

    -- JSONB: OCR extraction results
    ocr_data        JSONB NOT NULL DEFAULT '{}',
    -- Example ocr_data:
    -- {
    --   "status": "completed",
    --   "vendor_name": "Delta Air Lines",
    --   "amount": 450.00,
    --   "currency": "USD",
    --   "date": "2026-05-10",
    --   "line_items": [
    --     {"description": "SFO-JFK Economy", "amount": 350.00},
    --     {"description": "Seat upgrade", "amount": 100.00}
    --   ],
    --   "tax_amount": 38.25,
    --   "raw_text": "...",
    --   "confidence": 0.96
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_receipts_txn ON receipts(transaction_id);
CREATE INDEX idx_receipts_org ON receipts(organization_id);

-- ============================================================
-- REIMBURSEMENTS
-- ============================================================

CREATE TABLE reimbursements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_id       UUID REFERENCES entities(id),
    employee_id     UUID NOT NULL REFERENCES users(id),
    title           TEXT NOT NULL,
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    expense_date    DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','submitted','under_review','approved',
                                      'rejected','processing_payment','paid','cancelled')),

    -- JSONB: reimbursement details
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "merchant_name": "Office Depot",
    --   "mcc_code": "5943",
    --   "description": "Office supplies for Q2 offsite",
    --   "coding": {
    --     "gl_account": {"code": "6100", "name": "Office Supplies"},
    --     "department": {"code": "ENG"},
    --     "cost_center": {"code": "CC-100"}
    --   },
    --   "approval_history": [
    --     {"approver_id": "uuid", "decision": "approved", "at": "2026-05-11T14:00:00Z"}
    --   ],
    --   "payment": {
    --     "method": "ach",
    --     "reference": "ACH-2026-05-12-001",
    --     "paid_at": "2026-05-12T15:30:00Z"
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reimb_org ON reimbursements(organization_id);
CREATE INDEX idx_reimb_employee ON reimbursements(employee_id);
CREATE INDEX idx_reimb_status ON reimbursements(organization_id, status);

ALTER TABLE receipts ADD CONSTRAINT fk_receipts_reimb
    FOREIGN KEY (reimbursement_id) REFERENCES reimbursements(id);
```

---

## Vendors & Bills

```sql
-- ============================================================
-- VENDORS
-- ============================================================

CREATE TABLE vendors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','inactive','blocked')),

    -- JSONB: vendor profile and payment config
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "legal_name": "Amazon Web Services, Inc.",
    --   "tax_id": "26-3990536",
    --   "email": "billing@aws.amazon.com",
    --   "website": "aws.amazon.com",
    --   "payment_terms": "net_30",
    --   "default_gl_account": {"code": "6200", "name": "Software & SaaS"},
    --   "default_payment_method": "virtual_card",
    --   "address": {
    --     "street": "410 Terry Ave N",
    --     "city": "Seattle", "state": "WA",
    --     "postal_code": "98109", "country": "US"
    --   },
    --   "contracts": [
    --     {
    --       "name": "AWS Enterprise Support",
    --       "annual_value": 120000.00,
    --       "start_date": "2026-01-01",
    --       "end_date": "2026-12-31",
    --       "auto_renews": true,
    --       "renewal_notice_days": 60,
    --       "cancellation_deadline": "2026-10-31"
    --     }
    --   ],
    --   "w9_on_file": true,
    --   "w9_received_date": "2025-12-15"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_vendors_org ON vendors(organization_id);
CREATE INDEX idx_vendors_profile ON vendors USING gin(profile);

-- ============================================================
-- BILLS (AP Automation)
-- ============================================================

CREATE TABLE bills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_id       UUID REFERENCES entities(id),
    vendor_id       UUID NOT NULL REFERENCES vendors(id),
    bill_number     TEXT,
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','pending_approval','approved','scheduled',
                                      'paid','partially_paid','overdue','cancelled')),

    -- JSONB: line items, coding, and approval history
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "description": "AWS monthly invoice - May 2026",
    --   "tax_amount": 0,
    --   "line_items": [
    --     {
    --       "description": "EC2 Compute",
    --       "amount": 8500.00,
    --       "gl_account": {"code": "6200"},
    --       "department": {"code": "ENG"},
    --       "cost_center": {"code": "CC-100"}
    --     },
    --     {
    --       "description": "S3 Storage",
    --       "amount": 1200.00,
    --       "gl_account": {"code": "6210"},
    --       "department": {"code": "ENG"}
    --     }
    --   ],
    --   "approval_history": [
    --     {"approver_id": "uuid", "decision": "approved", "at": "2026-05-10T09:00:00Z"}
    --   ],
    --   "payment": {
    --     "method": "ach",
    --     "scheduled_date": "2026-06-01",
    --     "reference": "ACH-2026-06-01-003"
    --   },
    --   "ocr_file_url": "https://..."
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bills_org ON bills(organization_id);
CREATE INDEX idx_bills_vendor ON bills(vendor_id);
CREATE INDEX idx_bills_status ON bills(organization_id, status);
CREATE INDEX idx_bills_due ON bills(organization_id, due_date);
```

---

## Chart of Accounts

```sql
-- ============================================================
-- CHART OF ACCOUNTS (relational — core financial reference)
-- ============================================================

-- GL accounts remain fully relational because they're a core reference
-- table that every transaction, bill, and reimbursement references.
-- Keeping this relational ensures consistent coding across the platform.

CREATE TABLE gl_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    account_code    TEXT NOT NULL,
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL
                    CHECK (account_type IN ('asset','liability','equity','revenue','expense')),
    parent_id       UUID REFERENCES gl_accounts(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',   -- custom fields per account
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, account_code)
);
CREATE INDEX idx_gl_org ON gl_accounts(organization_id);
CREATE INDEX idx_gl_type ON gl_accounts(organization_id, account_type);
```

---

## Integrations & Sync

```sql
-- ============================================================
-- INTEGRATIONS (config-heavy, JSONB-native)
-- ============================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'connected'
                    CHECK (status IN ('connected','disconnected','error','pending')),

    -- JSONB: provider-specific configuration and credentials
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config (QuickBooks):
    -- {
    --   "realm_id": "1234567890",
    --   "oauth_access_token_encrypted": "enc_xxx",
    --   "oauth_refresh_token_encrypted": "enc_yyy",
    --   "token_expires_at": "2026-05-12T20:00:00Z",
    --   "sync_settings": {
    --     "auto_sync_transactions": true,
    --     "sync_interval_minutes": 15,
    --     "sync_gl_accounts": true,
    --     "sync_vendors": true,
    --     "default_expense_account": "6000"
    --   },
    --   "field_mappings": {
    --     "department": "Class",
    --     "cost_center": "Location",
    --     "project": "Customer"
    --   }
    -- }

    -- JSONB: ID mappings between internal and external systems
    mappings        JSONB NOT NULL DEFAULT '{}',
    -- Example mappings:
    -- {
    --   "gl_accounts": {
    --     "internal-uuid-1": {"external_id": "67", "name": "Travel"},
    --     "internal-uuid-2": {"external_id": "68", "name": "Software"}
    --   },
    --   "vendors": {
    --     "internal-uuid-3": {"external_id": "45", "name": "AWS"}
    --   }
    -- }

    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_integrations_org ON integrations(organization_id);
```

---

## Audit Trail

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL DEFAULT 'user',
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    changes         JSONB,                         -- {"field": {"old": "v1", "new": "v2"}}
    metadata        JSONB NOT NULL DEFAULT '{}',   -- ip, user_agent, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_id);

-- Reference data (same as other models)
CREATE TABLE currencies (
    code CHAR(3) PRIMARY KEY,
    numeric_code CHAR(3),
    name TEXT NOT NULL,
    minor_units INT NOT NULL DEFAULT 2,
    is_active BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE merchant_category_codes (
    code CHAR(4) PRIMARY KEY,
    description TEXT NOT NULL,
    category_group TEXT
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations | 2 | `organizations`, `entities` |
| Users | 1 | `users` (roles/departments in JSONB `profile`) |
| Cards | 1 | `cards` (controls in JSONB) |
| Transactions | 1 | `transactions` (coding, merchant, policy results in JSONB) |
| Budgets | 1 | `budgets` (config in JSONB) |
| Policies | 1 | `policies` (rules and scope in JSONB) |
| Receipts | 1 | `receipts` (OCR data in JSONB) |
| Reimbursements | 1 | `reimbursements` (details in JSONB) |
| Vendors & Bills | 2 | `vendors`, `bills` (profiles and details in JSONB) |
| GL Accounts | 1 | `gl_accounts` (relational — core reference) |
| Integrations | 1 | `integrations` (config and mappings in JSONB) |
| Audit | 1 | `audit_log` |
| Reference Data | 2 | `currencies`, `merchant_category_codes` |
| **Total** | **17** | vs. 41 in normalized model |

---

## Key Design Decisions

1. **17 tables instead of 41** — the JSONB hybrid eliminates 24 tables compared to the normalized model (junction tables, dimension tables, policy rule tables, approval step tables, bill line item tables). This dramatically reduces migration complexity and cognitive overhead for developers.

2. **Monetary amounts stay relational** — `amount`, `currency_code`, `billing_amount` on transactions, budgets, and bills remain typed relational columns. JSONB is never used for amounts that need aggregation (`SUM`, `AVG`, `GROUP BY`), because relational numeric columns are 5-10x faster for aggregation queries.

3. **GL accounts stay relational** — the chart of accounts is the backbone of financial reporting. It remains a proper relational table with foreign keys because every transaction coding, bill line item, and reimbursement references it.

4. **Policy engine entirely in JSONB** — policy rules, conditions, operators, and scope definitions live in JSONB on the `policies` table. This makes the policy engine infinitely extensible without migrations: adding a new condition type (e.g., "time_of_day") requires only application code changes, not schema changes.

5. **Approval history embedded in parent records** — instead of separate `approval_requests` and `approval_decisions` tables, approval history is stored as a JSONB array within the `details` field of reimbursements and bills. This simplifies reads (one query gets the record with its full approval chain) at the cost of harder cross-record approval analytics.

6. **User profile JSONB replaces 3 tables** — roles, department assignments, and permissions are in the `profile` JSONB column. This is appropriate for a spend management platform where user profiles are read frequently and written infrequently, and the profile structure varies by organization.

7. **Vendor contracts embedded in vendor profile** — rather than a separate `vendor_contracts` table, contracts live as a JSONB array within `vendors.profile`. This works well when contracts are viewed in the context of a vendor; a separate table would be warranted if contract-centric queries (e.g., "all contracts expiring this month across all vendors") become a primary use case.

8. **Integration mappings in JSONB** — ID mappings between internal and external systems (QuickBooks account ID 67 = internal GL account UUID X) live in the `integrations.mappings` JSONB column. This avoids a mapping table with potentially thousands of rows per integration.

9. **GIN indexes on JSONB columns** — every JSONB column used for filtering gets a GIN index. The `coding`, `policy_results`, `controls`, and `config` columns all have targeted GIN indexes for the specific JSONB paths queried most frequently.

10. **JSONB column documentation via comments** — since JSONB columns lack self-documenting schema constraints, every JSONB column includes inline SQL comments showing the expected structure. In production, these structures would be validated by JSON Schema in the application layer and documented in the OpenAPI 3.1 specification.
