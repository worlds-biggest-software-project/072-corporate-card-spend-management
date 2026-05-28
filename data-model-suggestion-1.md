# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Corporate Card & Spend Management · Created: 2026-05-12

## Philosophy

This model follows classical relational database design with full normalization (3NF+). Every concept in the domain — organizations, users, cards, transactions, budgets, policies, receipts, GL accounts, vendors — gets its own dedicated table with explicit foreign key relationships and junction tables for many-to-many associations. The schema is designed around referential integrity as the primary guarantee: no orphaned transactions, no budget allocations without valid budget parents, no GL codings without valid chart-of-accounts entries.

This approach mirrors how established financial software (SAP, Oracle ERP, NetSuite) structures data internally. It treats the database as the single source of truth and enforces business rules at the schema level through constraints, check clauses, and foreign keys rather than relying on application logic alone.

The normalized model is best suited for organizations that prioritize data correctness, need complex cross-entity reporting (e.g., "total spend by vendor across all entities for Q3, broken down by GL category and cost center"), and operate in regulated environments where auditors expect clean, well-defined data lineage.

**Best for:** Regulated environments with complex multi-entity reporting, SOX compliance requirements, and teams comfortable with traditional relational data modeling.

**Trade-offs:**
- (+) Maximum data integrity — constraints prevent inconsistent states at the database level
- (+) Complex analytical queries are natural with standard SQL JOINs
- (+) Well-understood by DBAs, auditors, and ERP integration teams
- (+) Schema serves as executable documentation of business rules
- (-) High table count increases migration complexity and JOIN depth for common queries
- (-) Adding jurisdiction-specific or policy-specific fields requires schema migrations
- (-) Write-heavy workloads (high transaction volumes) may hit contention on heavily-indexed tables
- (-) Audit trail requires separate history/audit tables, adding further complexity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 4217 | `currency_code CHAR(3)` on all monetary tables references ISO 4217 codes; `currencies` reference table pre-loaded with active codes |
| ISO 18245 (MCC) | `merchant_category_code CHAR(4)` on transaction records; `merchant_category_codes` reference table with descriptions |
| ISO 3166-1 | `country_code CHAR(2)` on addresses, entities, and jurisdiction-specific policy tables |
| ISO 20022 | Payment initiation records structured to map to `pain.001` message fields for cross-border reimbursements |
| PCI DSS 4.0.1 | Card PANs never stored; only tokenized references (`card_token`) from issuing partner APIs (Stripe, Lithic, Marqeta) |
| NACHA ACH | Reimbursement payment records include ACH-specific fields (`routing_number`, `account_number_token`, `company_entry_description`) |
| OAuth 2.0 / OIDC | `oauth_connections` table stores token metadata for accounting system integrations |
| OFX | Export format support informed by transaction field structure matching OFX `<STMTTRN>` elements |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- ORGANIZATIONS & MULTI-TENANCY
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,                          -- EIN, VAT number, etc.
    country_code    CHAR(2) NOT NULL,              -- ISO 3166-1 alpha-2
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',-- ISO 4217
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','suspended','closed')),
    settings        JSONB NOT NULL DEFAULT '{}',   -- org-level config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organizations_status ON organizations(status);

-- For multi-entity / PE portfolio companies
CREATE TABLE organization_entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    country_code    CHAR(2) NOT NULL,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    parent_entity_id UUID REFERENCES organization_entities(id),
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','suspended','closed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_entities_org ON organization_entities(organization_id);
CREATE INDEX idx_org_entities_parent ON organization_entities(parent_entity_id);

-- ============================================================
-- USERS & ROLES
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    employee_id     TEXT,                          -- HR system identifier
    department_id   UUID,                          -- FK added after departments table
    manager_id      UUID REFERENCES users(id),
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','inactive','terminated')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);
CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_manager ON users(manager_id);
CREATE INDEX idx_users_department ON users(department_id);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,                  -- 'card_admin','budget_owner','finance_manager','employee'
    description     TEXT,
    permissions     TEXT[] NOT NULL DEFAULT '{}',   -- array of permission slugs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_roles_org_name ON roles(organization_id, name);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id),
    role_id         UUID NOT NULL REFERENCES roles(id),
    entity_id       UUID REFERENCES organization_entities(id), -- scoped to entity if set
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id, entity_id)
);

-- Row-Level Security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_users ON users
    USING (organization_id = current_setting('app.current_org_id')::UUID);
```

---

## Organizational Structure

```sql
-- ============================================================
-- DEPARTMENTS & COST CENTERS
-- ============================================================

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_id       UUID REFERENCES organization_entities(id),
    name            TEXT NOT NULL,
    code            TEXT,                          -- short code e.g. 'ENG', 'MKT'
    parent_id       UUID REFERENCES departments(id),
    manager_id      UUID REFERENCES users(id),
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_departments_org ON departments(organization_id);
CREATE INDEX idx_departments_parent ON departments(parent_id);

ALTER TABLE users ADD CONSTRAINT fk_users_department
    FOREIGN KEY (department_id) REFERENCES departments(id);

CREATE TABLE cost_centers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    department_id   UUID REFERENCES departments(id),
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, code)
);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    code            TEXT,
    department_id   UUID REFERENCES departments(id),
    cost_center_id  UUID REFERENCES cost_centers(id),
    owner_id        UUID REFERENCES users(id),
    start_date      DATE,
    end_date        DATE,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','completed','cancelled')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_projects_org ON projects(organization_id);
```

---

## Chart of Accounts & GL

```sql
-- ============================================================
-- CHART OF ACCOUNTS
-- ============================================================

CREATE TABLE gl_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    account_code    TEXT NOT NULL,                  -- e.g. '6100'
    name            TEXT NOT NULL,                  -- e.g. 'Travel & Entertainment'
    account_type    TEXT NOT NULL
                    CHECK (account_type IN ('asset','liability','equity','revenue','expense')),
    parent_id       UUID REFERENCES gl_accounts(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, account_code)
);
CREATE INDEX idx_gl_accounts_org ON gl_accounts(organization_id);
CREATE INDEX idx_gl_accounts_parent ON gl_accounts(parent_id);
CREATE INDEX idx_gl_accounts_type ON gl_accounts(organization_id, account_type);

-- Custom accounting dimensions (class, location, custom fields)
CREATE TABLE accounting_dimensions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,                  -- 'class', 'location', 'project', custom
    is_required     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE accounting_dimension_values (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dimension_id    UUID NOT NULL REFERENCES accounting_dimensions(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES accounting_dimension_values(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_dim_values_dimension ON accounting_dimension_values(dimension_id);
```

---

## Card Programme

```sql
-- ============================================================
-- CARD ISSUER INTEGRATION
-- ============================================================

CREATE TABLE card_programmes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    issuer          TEXT NOT NULL
                    CHECK (issuer IN ('stripe_issuing','lithic','marqeta','manual')),
    issuer_programme_id TEXT,                      -- external programme ID
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CARDS
-- ============================================================

CREATE TABLE cards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    card_programme_id UUID NOT NULL REFERENCES card_programmes(id),
    cardholder_id   UUID NOT NULL REFERENCES users(id),
    card_token      TEXT NOT NULL,                  -- tokenized ref from issuer (never store PAN)
    issuer_card_id  TEXT NOT NULL,                  -- issuer's external card ID
    card_type       TEXT NOT NULL
                    CHECK (card_type IN ('physical','virtual')),
    card_purpose    TEXT NOT NULL DEFAULT 'general'
                    CHECK (card_purpose IN ('general','single_use','vendor_locked','project','subscription')),
    last_four       CHAR(4),
    network         TEXT CHECK (network IN ('visa','mastercard')),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    spend_limit_amount    NUMERIC(15,2),
    spend_limit_interval  TEXT CHECK (spend_limit_interval IN ('per_transaction','daily','weekly','monthly','yearly','all_time')),
    locked_vendor_id      UUID REFERENCES vendors(id),  -- for vendor-locked cards
    locked_project_id     UUID REFERENCES projects(id), -- for project cards
    budget_id       UUID,                          -- FK added after budgets table
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','frozen','cancelled','expired')),
    activated_at    TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cards_org ON cards(organization_id);
CREATE INDEX idx_cards_cardholder ON cards(cardholder_id);
CREATE INDEX idx_cards_programme ON cards(card_programme_id);
CREATE INDEX idx_cards_issuer_id ON cards(issuer_card_id);
CREATE INDEX idx_cards_status ON cards(organization_id, status);
```

---

## Transactions & Receipts

```sql
-- ============================================================
-- MERCHANTS
-- ============================================================

CREATE TABLE merchants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    clean_name      TEXT,                          -- enriched / normalized name
    mcc_code        CHAR(4),                       -- ISO 18245
    mcc_description TEXT,
    logo_url        TEXT,
    website         TEXT,
    country_code    CHAR(2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_merchants_name ON merchants USING gin(to_tsvector('english', name));
CREATE INDEX idx_merchants_mcc ON merchants(mcc_code);

-- ============================================================
-- TRANSACTIONS
-- ============================================================

CREATE TABLE transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_id       UUID REFERENCES organization_entities(id),
    card_id         UUID NOT NULL REFERENCES cards(id),
    cardholder_id   UUID NOT NULL REFERENCES users(id),

    -- Amounts
    amount          NUMERIC(15,2) NOT NULL,        -- in transaction currency
    currency_code   CHAR(3) NOT NULL,              -- ISO 4217
    billing_amount  NUMERIC(15,2),                 -- in org base currency
    billing_currency CHAR(3),
    fx_rate         NUMERIC(12,6),

    -- Merchant
    merchant_id     UUID REFERENCES merchants(id),
    merchant_name_raw TEXT,                        -- as received from card network
    mcc_code        CHAR(4),                       -- ISO 18245

    -- Issuer references
    issuer_transaction_id TEXT NOT NULL,
    issuer_authorization_id TEXT,
    authorization_method TEXT,                     -- 'chip','contactless','online','keyed'

    -- Status
    transaction_type TEXT NOT NULL
                    CHECK (transaction_type IN ('purchase','refund','reversal','fee','adjustment')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','cleared','declined','reversed','disputed')),

    -- Timestamps
    authorized_at   TIMESTAMPTZ,
    cleared_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_transactions_org ON transactions(organization_id);
CREATE INDEX idx_transactions_card ON transactions(card_id);
CREATE INDEX idx_transactions_cardholder ON transactions(cardholder_id);
CREATE INDEX idx_transactions_status ON transactions(organization_id, status);
CREATE INDEX idx_transactions_date ON transactions(organization_id, authorized_at);
CREATE INDEX idx_transactions_merchant ON transactions(merchant_id);
CREATE INDEX idx_transactions_mcc ON transactions(mcc_code);

-- ============================================================
-- RECEIPTS
-- ============================================================

CREATE TABLE receipts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    transaction_id  UUID REFERENCES transactions(id),
    reimbursement_id UUID,                         -- FK added after reimbursements table
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    file_url        TEXT NOT NULL,
    file_type       TEXT,                          -- 'image/jpeg','application/pdf', etc.
    file_size_bytes BIGINT,
    ocr_status      TEXT DEFAULT 'pending'
                    CHECK (ocr_status IN ('pending','processing','completed','failed')),
    ocr_vendor_name TEXT,
    ocr_amount      NUMERIC(15,2),
    ocr_currency    CHAR(3),
    ocr_date        DATE,
    ocr_raw_text    TEXT,
    source          TEXT NOT NULL DEFAULT 'upload'
                    CHECK (source IN ('upload','email','sms','slack','teams','auto_generated')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_receipts_transaction ON receipts(transaction_id);
CREATE INDEX idx_receipts_org ON receipts(organization_id);
```

---

## GL Coding & Expense Lines

```sql
-- ============================================================
-- EXPENSE CODING (GL line items per transaction)
-- ============================================================

CREATE TABLE expense_codings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id  UUID REFERENCES transactions(id),
    reimbursement_id UUID,                         -- FK added later
    organization_id UUID NOT NULL REFERENCES organizations(id),

    -- GL coding
    gl_account_id   UUID NOT NULL REFERENCES gl_accounts(id),
    department_id   UUID REFERENCES departments(id),
    cost_center_id  UUID REFERENCES cost_centers(id),
    project_id      UUID REFERENCES projects(id),

    -- Split amount (for split-coded transactions)
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    memo            TEXT,

    -- AI coding metadata
    coding_method   TEXT NOT NULL DEFAULT 'manual'
                    CHECK (coding_method IN ('manual','ai_auto','ai_suggested','rule_based')),
    ai_confidence   NUMERIC(3,2),                  -- 0.00 to 1.00
    coding_rule_id  UUID,                          -- FK to coding_rules if rule-based

    -- Review
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_expense_codings_txn ON expense_codings(transaction_id);
CREATE INDEX idx_expense_codings_gl ON expense_codings(gl_account_id);
CREATE INDEX idx_expense_codings_dept ON expense_codings(department_id);
CREATE INDEX idx_expense_codings_org ON expense_codings(organization_id);

-- Dimension assignments for custom accounting dimensions
CREATE TABLE expense_coding_dimensions (
    expense_coding_id  UUID NOT NULL REFERENCES expense_codings(id),
    dimension_id       UUID NOT NULL REFERENCES accounting_dimensions(id),
    dimension_value_id UUID NOT NULL REFERENCES accounting_dimension_values(id),
    PRIMARY KEY (expense_coding_id, dimension_id)
);

-- AI/rule-based coding rules
CREATE TABLE coding_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    priority        INT NOT NULL DEFAULT 100,
    match_criteria  JSONB NOT NULL,
    -- Example match_criteria:
    -- {"mcc_codes": ["5812","5813"], "merchant_name_contains": "uber", "amount_max": 100}
    gl_account_id   UUID NOT NULL REFERENCES gl_accounts(id),
    department_id   UUID REFERENCES departments(id),
    cost_center_id  UUID REFERENCES cost_centers(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_coding_rules_org ON coding_rules(organization_id);
```

---

## Budgets

```sql
-- ============================================================
-- BUDGETS
-- ============================================================

CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_id       UUID REFERENCES organization_entities(id),
    name            TEXT NOT NULL,
    budget_type     TEXT NOT NULL
                    CHECK (budget_type IN ('department','project','team','vendor','card','custom')),

    -- Ownership
    owner_id        UUID NOT NULL REFERENCES users(id),
    department_id   UUID REFERENCES departments(id),
    project_id      UUID REFERENCES projects(id),

    -- Limits
    limit_amount    NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    period          TEXT NOT NULL
                    CHECK (period IN ('monthly','quarterly','yearly','one_time','custom')),
    period_start    DATE,
    period_end      DATE,

    -- Pre-spend model (Divvy-style optional)
    is_pre_funded   BOOLEAN NOT NULL DEFAULT false,
    funded_amount   NUMERIC(15,2) DEFAULT 0,

    -- Alerts
    alert_threshold_pct NUMERIC(5,2) DEFAULT 80.00, -- alert at 80% of budget

    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','paused','closed','exhausted')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_budgets_org ON budgets(organization_id);
CREATE INDEX idx_budgets_owner ON budgets(owner_id);
CREATE INDEX idx_budgets_dept ON budgets(department_id);
CREATE INDEX idx_budgets_project ON budgets(project_id);

ALTER TABLE cards ADD CONSTRAINT fk_cards_budget
    FOREIGN KEY (budget_id) REFERENCES budgets(id);

-- Budget period snapshots for historical tracking
CREATE TABLE budget_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_id       UUID NOT NULL REFERENCES budgets(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    limit_amount    NUMERIC(15,2) NOT NULL,
    spent_amount    NUMERIC(15,2) NOT NULL DEFAULT 0,
    committed_amount NUMERIC(15,2) NOT NULL DEFAULT 0, -- authorized but not cleared
    currency_code   CHAR(3) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','closed','over_budget')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_budget_periods_budget ON budget_periods(budget_id);
CREATE INDEX idx_budget_periods_dates ON budget_periods(period_start, period_end);
```

---

## Policy Engine

```sql
-- ============================================================
-- SPEND POLICIES
-- ============================================================

CREATE TABLE policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    description     TEXT,
    policy_type     TEXT NOT NULL
                    CHECK (policy_type IN ('spend_limit','mcc_block','mcc_allow','receipt_required',
                                           'approval_required','time_restriction','vendor_restriction')),
    -- Scope: who does this policy apply to?
    applies_to_all  BOOLEAN NOT NULL DEFAULT false,
    priority        INT NOT NULL DEFAULT 100,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_policies_org ON policies(organization_id);

-- Policy scope assignments
CREATE TABLE policy_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policies(id),
    target_type     TEXT NOT NULL
                    CHECK (target_type IN ('user','role','department','entity','card')),
    target_id       UUID NOT NULL,                 -- ID of the target (user, role, dept, etc.)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_policy_assign_policy ON policy_assignments(policy_id);
CREATE INDEX idx_policy_assign_target ON policy_assignments(target_type, target_id);

-- Policy rules (the specific conditions)
CREATE TABLE policy_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policies(id),
    rule_type       TEXT NOT NULL,                  -- 'max_amount','mcc_list','receipt_threshold', etc.
    operator        TEXT NOT NULL DEFAULT 'equals'
                    CHECK (operator IN ('equals','not_equals','greater_than','less_than',
                                        'in','not_in','between','contains')),
    value           TEXT NOT NULL,                  -- serialized value
    value_currency  CHAR(3),                       -- for monetary rules
    action          TEXT NOT NULL DEFAULT 'flag'
                    CHECK (action IN ('block','flag','require_approval','require_receipt','warn')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_policy_rules_policy ON policy_rules(policy_id);

-- Policy violation log
CREATE TABLE policy_violations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    transaction_id  UUID REFERENCES transactions(id),
    reimbursement_id UUID,
    policy_id       UUID NOT NULL REFERENCES policies(id),
    policy_rule_id  UUID NOT NULL REFERENCES policy_rules(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    violation_type  TEXT NOT NULL,
    severity        TEXT NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('info','warning','violation','block')),
    description     TEXT,
    resolution      TEXT CHECK (resolution IN ('pending','approved_exception','rejected','auto_resolved')),
    resolved_by     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    ai_review_score NUMERIC(3,2),                  -- AI policy agent confidence
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_violations_org ON policy_violations(organization_id);
CREATE INDEX idx_violations_txn ON policy_violations(transaction_id);
CREATE INDEX idx_violations_user ON policy_violations(user_id);
CREATE INDEX idx_violations_resolution ON policy_violations(resolution);
```

---

## Approval Workflows

```sql
-- ============================================================
-- APPROVAL WORKFLOWS
-- ============================================================

CREATE TABLE approval_workflows (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    trigger_type    TEXT NOT NULL
                    CHECK (trigger_type IN ('expense_over_amount','policy_violation',
                                            'reimbursement','bill_payment','card_request',
                                            'budget_request')),
    trigger_threshold NUMERIC(15,2),
    trigger_currency  CHAR(3),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_workflow_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID NOT NULL REFERENCES approval_workflows(id),
    step_order      INT NOT NULL,
    approver_type   TEXT NOT NULL
                    CHECK (approver_type IN ('direct_manager','budget_owner','role','specific_user')),
    approver_role_id UUID REFERENCES roles(id),
    approver_user_id UUID REFERENCES users(id),
    auto_approve_below NUMERIC(15,2),              -- auto-approve if under this amount
    escalation_hours INT DEFAULT 48,               -- hours before escalation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_approval_steps_workflow ON approval_workflow_steps(workflow_id);

CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    workflow_id     UUID NOT NULL REFERENCES approval_workflows(id),
    current_step_id UUID REFERENCES approval_workflow_steps(id),
    request_type    TEXT NOT NULL,
    reference_type  TEXT NOT NULL,                  -- 'transaction','reimbursement','bill','card_request'
    reference_id    UUID NOT NULL,
    requester_id    UUID NOT NULL REFERENCES users(id),
    amount          NUMERIC(15,2),
    currency_code   CHAR(3),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','approved','rejected','escalated','cancelled')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_approval_req_org ON approval_requests(organization_id);
CREATE INDEX idx_approval_req_status ON approval_requests(status);
CREATE INDEX idx_approval_req_ref ON approval_requests(reference_type, reference_id);

CREATE TABLE approval_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    approval_request_id UUID NOT NULL REFERENCES approval_requests(id),
    step_id         UUID NOT NULL REFERENCES approval_workflow_steps(id),
    approver_id     UUID NOT NULL REFERENCES users(id),
    decision        TEXT NOT NULL
                    CHECK (decision IN ('approved','rejected','delegated')),
    comment         TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_approval_dec_request ON approval_decisions(approval_request_id);
```

---

## Reimbursements & Bill Pay

```sql
-- ============================================================
-- REIMBURSEMENTS
-- ============================================================

CREATE TABLE reimbursements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_id       UUID REFERENCES organization_entities(id),
    employee_id     UUID NOT NULL REFERENCES users(id),
    title           TEXT NOT NULL,
    description     TEXT,
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    expense_date    DATE NOT NULL,
    merchant_name   TEXT,
    mcc_code        CHAR(4),
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','submitted','under_review','approved',
                                      'rejected','processing_payment','paid','cancelled')),
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    approved_by     UUID REFERENCES users(id),
    paid_at         TIMESTAMPTZ,
    payment_method  TEXT CHECK (payment_method IN ('ach','wire','check','other')),
    payment_reference TEXT,                        -- ACH trace number, check number, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reimbursements_org ON reimbursements(organization_id);
CREATE INDEX idx_reimbursements_employee ON reimbursements(employee_id);
CREATE INDEX idx_reimbursements_status ON reimbursements(organization_id, status);

ALTER TABLE receipts ADD CONSTRAINT fk_receipts_reimbursement
    FOREIGN KEY (reimbursement_id) REFERENCES reimbursements(id);

ALTER TABLE expense_codings ADD CONSTRAINT fk_expense_codings_reimbursement
    FOREIGN KEY (reimbursement_id) REFERENCES reimbursements(id);

-- ============================================================
-- VENDORS & BILLS (AP Automation)
-- ============================================================

CREATE TABLE vendors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,                          -- W-9 / tax ID
    email           TEXT,
    phone           TEXT,
    website         TEXT,
    payment_terms   TEXT,                          -- 'net_30','net_60','due_on_receipt'
    default_gl_account_id UUID REFERENCES gl_accounts(id),
    default_payment_method TEXT
                    CHECK (default_payment_method IN ('ach','wire','check','virtual_card')),
    country_code    CHAR(2),
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','inactive','blocked')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_vendors_org ON vendors(organization_id);

CREATE TABLE bills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_id       UUID REFERENCES organization_entities(id),
    vendor_id       UUID NOT NULL REFERENCES vendors(id),
    bill_number     TEXT,
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    tax_amount      NUMERIC(15,2) DEFAULT 0,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','pending_approval','approved','scheduled',
                                      'paid','partially_paid','overdue','cancelled','voided')),
    payment_method  TEXT,
    payment_date    DATE,
    payment_reference TEXT,
    ocr_file_url    TEXT,                          -- scanned invoice
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bills_org ON bills(organization_id);
CREATE INDEX idx_bills_vendor ON bills(vendor_id);
CREATE INDEX idx_bills_status ON bills(organization_id, status);
CREATE INDEX idx_bills_due ON bills(organization_id, due_date);

-- Bill line items with GL coding
CREATE TABLE bill_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bill_id         UUID NOT NULL REFERENCES bills(id),
    description     TEXT,
    amount          NUMERIC(15,2) NOT NULL,
    gl_account_id   UUID NOT NULL REFERENCES gl_accounts(id),
    department_id   UUID REFERENCES departments(id),
    cost_center_id  UUID REFERENCES cost_centers(id),
    project_id      UUID REFERENCES projects(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bill_items_bill ON bill_line_items(bill_id);

-- Vendor contracts for renewal tracking
CREATE TABLE vendor_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    vendor_id       UUID NOT NULL REFERENCES vendors(id),
    contract_name   TEXT NOT NULL,
    annual_value    NUMERIC(15,2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    start_date      DATE NOT NULL,
    end_date        DATE,
    auto_renews     BOOLEAN NOT NULL DEFAULT false,
    renewal_notice_days INT DEFAULT 30,
    cancellation_deadline DATE,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','expired','cancelled','pending_renewal')),
    document_url    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_vendor_contracts_org ON vendor_contracts(organization_id);
CREATE INDEX idx_vendor_contracts_vendor ON vendor_contracts(vendor_id);
CREATE INDEX idx_vendor_contracts_renewal ON vendor_contracts(cancellation_deadline)
    WHERE status = 'active' AND auto_renews = true;
```

---

## Integrations & Sync

```sql
-- ============================================================
-- ACCOUNTING SYSTEM INTEGRATIONS
-- ============================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider        TEXT NOT NULL
                    CHECK (provider IN ('quickbooks','xero','netsuite','sage_intacct',
                                        'dynamics365','stripe','lithic','marqeta',
                                        'plaid','slack','teams','hris_workday',
                                        'hris_bamboohr','hris_rippling')),
    status          TEXT NOT NULL DEFAULT 'connected'
                    CHECK (status IN ('connected','disconnected','error','pending')),
    oauth_access_token_encrypted  TEXT,
    oauth_refresh_token_encrypted TEXT,
    oauth_token_expires_at TIMESTAMPTZ,
    external_entity_id TEXT,                       -- company ID in external system
    last_sync_at    TIMESTAMPTZ,
    sync_cursor     TEXT,                          -- pagination cursor for incremental sync
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_integrations_org ON integrations(organization_id);

-- Mapping between internal and external IDs
CREATE TABLE integration_mappings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES integrations(id),
    internal_type   TEXT NOT NULL,                  -- 'gl_account','vendor','department','employee'
    internal_id     UUID NOT NULL,
    external_id     TEXT NOT NULL,
    external_name   TEXT,
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_int_mappings_integration ON integration_mappings(integration_id);
CREATE UNIQUE INDEX idx_int_mappings_unique ON integration_mappings(integration_id, internal_type, internal_id);

-- Sync log
CREATE TABLE sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES integrations(id),
    sync_type       TEXT NOT NULL,                  -- 'full','incremental','push','pull'
    entity_type     TEXT NOT NULL,                  -- 'transactions','gl_accounts','vendors'
    records_processed INT NOT NULL DEFAULT 0,
    records_created INT NOT NULL DEFAULT 0,
    records_updated INT NOT NULL DEFAULT 0,
    records_failed  INT NOT NULL DEFAULT 0,
    error_details   TEXT,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running','completed','failed','partial'))
);
CREATE INDEX idx_sync_log_integration ON sync_log(integration_id);
```

---

## Audit Trail

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    actor_id        UUID REFERENCES users(id),     -- NULL for system actions
    actor_type      TEXT NOT NULL DEFAULT 'user'
                    CHECK (actor_type IN ('user','system','api','ai_agent')),
    action          TEXT NOT NULL,                  -- 'create','update','delete','approve','reject', etc.
    resource_type   TEXT NOT NULL,                  -- 'transaction','card','budget','policy', etc.
    resource_id     UUID NOT NULL,
    changes         JSONB,                         -- {"field": {"old": "v1", "new": "v2"}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log(organization_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_id);
CREATE INDEX idx_audit_date ON audit_log(organization_id, created_at);

-- Partition by month for performance
-- CREATE TABLE audit_log PARTITION BY RANGE (created_at);
```

---

## Reference Data

```sql
-- ============================================================
-- REFERENCE DATA
-- ============================================================

CREATE TABLE currencies (
    code            CHAR(3) PRIMARY KEY,           -- ISO 4217
    numeric_code    CHAR(3),
    name            TEXT NOT NULL,
    minor_units     INT NOT NULL DEFAULT 2,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE merchant_category_codes (
    code            CHAR(4) PRIMARY KEY,           -- ISO 18245
    description     TEXT NOT NULL,
    category_group  TEXT                            -- 'travel','dining','office','software','utilities', etc.
);

CREATE TABLE countries (
    code            CHAR(2) PRIMARY KEY,           -- ISO 3166-1 alpha-2
    name            TEXT NOT NULL,
    alpha3          CHAR(3),
    numeric_code    CHAR(3)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organizations & Tenancy | 2 | `organizations`, `organization_entities` |
| Users & Roles | 3 | `users`, `roles`, `user_roles` |
| Org Structure | 3 | `departments`, `cost_centers`, `projects` |
| Chart of Accounts | 3 | `gl_accounts`, `accounting_dimensions`, `accounting_dimension_values` |
| Card Programme | 2 | `card_programmes`, `cards` |
| Transactions & Receipts | 3 | `transactions`, `merchants`, `receipts` |
| GL Coding | 3 | `expense_codings`, `expense_coding_dimensions`, `coding_rules` |
| Budgets | 2 | `budgets`, `budget_periods` |
| Policies | 3 | `policies`, `policy_assignments`, `policy_rules` |
| Policy Violations | 1 | `policy_violations` |
| Approval Workflows | 4 | `approval_workflows`, `approval_workflow_steps`, `approval_requests`, `approval_decisions` |
| Reimbursements | 1 | `reimbursements` |
| Vendors & Bills | 4 | `vendors`, `bills`, `bill_line_items`, `vendor_contracts` |
| Integrations | 3 | `integrations`, `integration_mappings`, `sync_log` |
| Audit | 1 | `audit_log` |
| Reference Data | 3 | `currencies`, `merchant_category_codes`, `countries` |
| **Total** | **41** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation without coordination, safe for multi-region deployment and API exposure without leaking sequence information.

2. **Explicit junction tables for many-to-many** — `user_roles`, `policy_assignments`, `expense_coding_dimensions` rather than arrays or JSONB, ensuring referential integrity and queryability.

3. **Separate `expense_codings` table for GL line items** — supports split-coding (one transaction coded across multiple GL accounts/departments) and tracks coding provenance (manual vs. AI vs. rule-based) with confidence scores.

4. **Card tokens, never PANs** — `card_token` and `issuer_card_id` reference the card issuer's tokenized representations. Full PAN, CVV, and expiry are never stored in this database, maintaining PCI DSS scope minimization.

5. **Hierarchical departments via self-referential FK** — `departments.parent_id` references `departments.id`, enabling recursive CTE queries for roll-up reporting (e.g., total spend for "Engineering" including all sub-departments).

6. **Budget periods as separate table** — `budget_periods` snapshots each period's allocation and spend, enabling historical budget-vs-actual analysis without recalculating from transactions.

7. **Policy engine as data, not code** — policies, assignments, and rules are all stored as relational data, making the policy engine configurable without code deployments and auditable through the standard audit log.

8. **Row-Level Security for multi-tenancy** — PostgreSQL RLS policies on `organization_id` enforce tenant isolation at the database level, not just the application layer.

9. **ISO reference tables pre-loaded** — `currencies` (ISO 4217), `merchant_category_codes` (ISO 18245), and `countries` (ISO 3166-1) are reference tables populated at deployment, ensuring consistent standards-aligned data.

10. **Audit log designed for partitioning** — `audit_log` is structured for range partitioning by `created_at`, enabling efficient retention management and time-range queries without impacting write performance.
