# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Corporate Card & Spend Management · Created: 2026-05-12

## Philosophy

This model uses a graph layer on top of a relational foundation to model the rich relationship networks inherent in corporate spend management: organizational hierarchies (departments, entities, cost centers), approval chains (employee reports to manager reports to VP), budget ownership graphs (budget allocated to department, subdivided to projects, assigned to cards), vendor relationship networks, and policy scope assignments. The operational CRUD data (transactions, cards, receipts) lives in standard relational tables, while the relationship-heavy aspects are modeled as a property graph using PostgreSQL's `ltree` extension for hierarchies and dedicated node/edge tables for complex relationship queries.

Graph-relational patterns are used in compliance and fraud detection systems at financial institutions, where the core question is "what is the relationship between entity A and entity B, and through what path?" In spend management, this manifests as: "Can this employee approve this expense?" (requires traversing the org hierarchy), "Which budgets does this transaction draw from?" (requires traversing budget allocation chains), and "Is there a conflict of interest between this vendor's signatory and any employee authorized to approve their bills?" (requires traversing people-vendor-organization relationships).

This model is ideal for multi-entity organizations with complex organizational structures, PE portfolio companies needing consolidated spend visibility across subsidiaries, and platforms that want to build sophisticated conflict-of-interest detection or spend authorization analysis.

**Best for:** Multi-entity organizations with deep hierarchies, PE portfolio companies, and platforms requiring conflict-of-interest detection, complex approval chain traversal, or organizational spend consolidation.

**Trade-offs:**
- (+) Organizational hierarchy queries (roll-up reporting, approval chain traversal) are elegant and fast
- (+) Multi-entity consolidation with parent-child entity graphs is natural
- (+) Conflict-of-interest and relationship analysis queries that would require multiple self-joins are simple graph traversals
- (+) Budget hierarchy (parent budgets allocated to child budgets) is first-class
- (+) New relationship types added without schema changes — just new edge types
- (-) `ltree` extension required — not available on all managed PostgreSQL providers
- (-) Graph query patterns are less familiar to typical application developers
- (-) Maintaining graph consistency alongside relational FKs adds complexity
- (-) Recursive CTEs and ltree queries can be harder to optimize than flat table scans
- (-) Graph layer adds operational overhead for what may be simple hierarchy needs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 4217 | `currency_code CHAR(3)` on all monetary tables |
| ISO 18245 (MCC) | `mcc_code CHAR(4)` on transactions; MCC-based spend analysis via graph edges |
| ISO 3166-1 | `country_code CHAR(2)` on organization and entity nodes; jurisdiction graph for compliance |
| PCI DSS 4.0.1 | Card tokens only; graph edges never contain cardholder data |
| ISO 20022 | Payment entity nodes carry structured party data aligned with ISO 20022 party identification |
| NACHA ACH | Payment method edges carry ACH routing and settlement fields |
| HR-XML / HR Open Standards | Organization hierarchy node structure influenced by HR-XML org chart modeling |

---

## Graph Layer

```sql
-- ============================================================
-- GRAPH INFRASTRUCTURE
-- ============================================================

-- Requires: CREATE EXTENSION IF NOT EXISTS ltree;
CREATE EXTENSION IF NOT EXISTS ltree;

-- ============================================================
-- GRAPH NODES
-- ============================================================

-- Universal node table for all entities that participate in relationships.
-- Each node has a type, a reference to the "real" relational record, and
-- properties stored as JSONB.

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    node_type       TEXT NOT NULL
                    CHECK (node_type IN (
                        'organization','entity','department','cost_center',
                        'project','user','card','budget','vendor','policy',
                        'gl_account'
                    )),
    -- Reference to the relational table record
    ref_table       TEXT NOT NULL,                  -- 'organizations','users','cards', etc.
    ref_id          UUID NOT NULL,                  -- PK in the referenced table

    -- Human-readable label for graph visualization
    label           TEXT NOT NULL,

    -- Hierarchy path (for ltree-based hierarchy queries)
    -- Example: 'org.entity_us.dept_eng.team_platform'
    hierarchy_path  LTREE,

    -- Node properties (denormalized from relational table for graph queries)
    properties      JSONB NOT NULL DEFAULT '{}',

    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE(ref_table, ref_id)
);
CREATE INDEX idx_nodes_org ON graph_nodes(organization_id);
CREATE INDEX idx_nodes_type ON graph_nodes(organization_id, node_type);
CREATE INDEX idx_nodes_ref ON graph_nodes(ref_table, ref_id);
CREATE INDEX idx_nodes_path ON graph_nodes USING gist(hierarchy_path);
CREATE INDEX idx_nodes_properties ON graph_nodes USING gin(properties);

-- ============================================================
-- GRAPH EDGES
-- ============================================================

-- Directed edges between nodes. Each edge has a type, optional weight,
-- and temporal validity (valid_from/valid_to for historical relationships).

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    edge_type       TEXT NOT NULL
                    CHECK (edge_type IN (
                        -- Organizational hierarchy
                        'parent_of',           -- entity → entity, dept → dept
                        'belongs_to',          -- user → department, card → user
                        'reports_to',          -- user → user (manager chain)
                        'member_of',           -- user → entity
                        
                        -- Budget relationships
                        'owns_budget',         -- user → budget
                        'funded_by',           -- budget → parent_budget
                        'allocated_to',        -- budget → department/project
                        'charged_to',          -- card → budget
                        
                        -- Spend relationships
                        'transacts_with',      -- user → vendor (spend relationship)
                        'supplies',            -- vendor → organization
                        'contracted_with',     -- vendor → entity
                        
                        -- Policy relationships
                        'governed_by',         -- user/dept/entity → policy
                        'approves_for',        -- user → user/dept (approval authority)
                        'delegates_to',        -- user → user (delegation of authority)
                        
                        -- GL relationships
                        'maps_to',             -- department → gl_account (default coding)
                        'child_account_of',    -- gl_account → gl_account
                        'cost_allocated_to'    -- cost_center → gl_account
                    )),

    -- Edge properties
    weight          NUMERIC(10,4) DEFAULT 1.0,     -- for weighted graph algorithms
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties by edge type:
    -- reports_to:       {"title": "Engineering Manager", "direct": true}
    -- owns_budget:      {"role": "primary", "delegation_allowed": true}
    -- approves_for:     {"max_amount": 5000.00, "currency": "USD", "categories": ["all"]}
    -- transacts_with:   {"total_spend_ytd": 45000.00, "transaction_count": 23}
    -- contracted_with:  {"contract_value": 120000.00, "end_date": "2026-12-31"}

    -- Temporal validity (for historical relationship tracking)
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                   -- NULL = currently active

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_edges_org ON graph_edges(organization_id);
CREATE INDEX idx_edges_source ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_edges_target ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_edges_type ON graph_edges(organization_id, edge_type);
CREATE INDEX idx_edges_active ON graph_edges(organization_id, edge_type)
    WHERE valid_to IS NULL;
CREATE INDEX idx_edges_properties ON graph_edges USING gin(properties);
CREATE INDEX idx_edges_temporal ON graph_edges(valid_from, valid_to);
```

---

## Relational Layer: Core Identity

```sql
-- ============================================================
-- ORGANIZATIONS & ENTITIES
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    country_code    CHAR(2) NOT NULL,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','suspended','closed')),
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    country_code    CHAR(2) NOT NULL,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    -- Hierarchy managed in graph layer (parent_of edges + ltree paths)
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_entities_org ON entities(organization_id);

-- ============================================================
-- USERS
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    employee_id     TEXT,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','inactive','terminated')),
    -- Department, manager, and role assignments are in the graph layer
    -- (belongs_to, reports_to, member_of edges)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);
CREATE INDEX idx_users_org ON users(organization_id);

-- ============================================================
-- DEPARTMENTS
-- ============================================================

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    code            TEXT,
    -- Parent department relationship managed in graph layer (parent_of edges)
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_departments_org ON departments(organization_id);

-- ============================================================
-- ROLES & PERMISSIONS
-- ============================================================

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_roles_org_name ON roles(organization_id, name);
```

---

## Relational Layer: Financial Operations

```sql
-- ============================================================
-- CARDS
-- ============================================================

CREATE TABLE cards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    cardholder_id   UUID NOT NULL REFERENCES users(id),
    card_token      TEXT NOT NULL,
    issuer_card_id  TEXT NOT NULL,
    issuer          TEXT NOT NULL
                    CHECK (issuer IN ('stripe_issuing','lithic','marqeta')),
    card_type       TEXT NOT NULL
                    CHECK (card_type IN ('physical','virtual')),
    last_four       CHAR(4),
    network         TEXT CHECK (network IN ('visa','mastercard')),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    spend_limit_amount NUMERIC(15,2),
    spend_limit_interval TEXT,
    card_purpose    TEXT NOT NULL DEFAULT 'general',
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','frozen','cancelled','expired')),
    -- Budget assignment managed via graph edge (charged_to)
    activated_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cards_org ON cards(organization_id);
CREATE INDEX idx_cards_cardholder ON cards(cardholder_id);
CREATE INDEX idx_cards_status ON cards(organization_id, status);
CREATE INDEX idx_cards_issuer ON cards(issuer_card_id);

-- ============================================================
-- GL ACCOUNTS
-- ============================================================

CREATE TABLE gl_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    account_code    TEXT NOT NULL,
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL
                    CHECK (account_type IN ('asset','liability','equity','revenue','expense')),
    -- Parent account hierarchy managed in graph layer (child_account_of edges + ltree)
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, account_code)
);
CREATE INDEX idx_gl_org ON gl_accounts(organization_id);

-- ============================================================
-- TRANSACTIONS
-- ============================================================

CREATE TABLE transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    card_id         UUID NOT NULL REFERENCES cards(id),
    cardholder_id   UUID NOT NULL REFERENCES users(id),
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    billing_amount  NUMERIC(15,2),
    billing_currency CHAR(3),
    fx_rate         NUMERIC(12,6),
    merchant_name   TEXT,
    mcc_code        CHAR(4),
    issuer_transaction_id TEXT NOT NULL,
    issuer_authorization_id TEXT,
    authorization_method TEXT,
    transaction_type TEXT NOT NULL
                    CHECK (transaction_type IN ('purchase','refund','reversal','fee')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','cleared','declined','reversed','disputed')),
    -- GL coding
    gl_account_id   UUID REFERENCES gl_accounts(id),
    department_id   UUID REFERENCES departments(id),
    coding_method   TEXT DEFAULT 'manual',
    ai_confidence   NUMERIC(3,2),
    coding_details  JSONB NOT NULL DEFAULT '{}',   -- split coding, custom dimensions, memo
    -- Policy results
    policy_results  JSONB NOT NULL DEFAULT '{}',
    -- Merchant enrichment
    merchant_data   JSONB NOT NULL DEFAULT '{}',
    authorized_at   TIMESTAMPTZ,
    cleared_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_txn_org ON transactions(organization_id);
CREATE INDEX idx_txn_card ON transactions(card_id);
CREATE INDEX idx_txn_cardholder ON transactions(cardholder_id);
CREATE INDEX idx_txn_status ON transactions(organization_id, status);
CREATE INDEX idx_txn_date ON transactions(organization_id, authorized_at);
CREATE INDEX idx_txn_mcc ON transactions(mcc_code);
CREATE INDEX idx_txn_gl ON transactions(gl_account_id);
CREATE INDEX idx_txn_dept ON transactions(department_id);

-- ============================================================
-- BUDGETS
-- ============================================================

CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    budget_type     TEXT NOT NULL
                    CHECK (budget_type IN ('department','project','team','vendor','card','custom')),
    owner_id        UUID NOT NULL REFERENCES users(id),
    limit_amount    NUMERIC(15,2) NOT NULL,
    spent_amount    NUMERIC(15,2) NOT NULL DEFAULT 0,
    committed_amount NUMERIC(15,2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    period          TEXT NOT NULL
                    CHECK (period IN ('monthly','quarterly','yearly','one_time','custom')),
    period_start    DATE,
    period_end      DATE,
    is_pre_funded   BOOLEAN NOT NULL DEFAULT false,
    funded_amount   NUMERIC(15,2) DEFAULT 0,
    alert_threshold_pct NUMERIC(5,2) DEFAULT 80.00,
    -- Parent budget and allocation targets managed in graph layer
    -- (funded_by, allocated_to edges)
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','paused','closed','exhausted')),
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_budgets_org ON budgets(organization_id);
CREATE INDEX idx_budgets_owner ON budgets(owner_id);
CREATE INDEX idx_budgets_status ON budgets(organization_id, status);

-- ============================================================
-- RECEIPTS
-- ============================================================

CREATE TABLE receipts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    transaction_id  UUID REFERENCES transactions(id),
    reimbursement_id UUID,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    file_url        TEXT NOT NULL,
    file_type       TEXT,
    file_size_bytes BIGINT,
    source          TEXT NOT NULL DEFAULT 'upload',
    ocr_data        JSONB NOT NULL DEFAULT '{}',
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
    employee_id     UUID NOT NULL REFERENCES users(id),
    title           TEXT NOT NULL,
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    expense_date    DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','submitted','under_review','approved',
                                      'rejected','processing_payment','paid','cancelled')),
    gl_account_id   UUID REFERENCES gl_accounts(id),
    department_id   UUID REFERENCES departments(id),
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reimb_org ON reimbursements(organization_id);
CREATE INDEX idx_reimb_employee ON reimbursements(employee_id);
CREATE INDEX idx_reimb_status ON reimbursements(organization_id, status);

ALTER TABLE receipts ADD CONSTRAINT fk_receipts_reimb
    FOREIGN KEY (reimbursement_id) REFERENCES reimbursements(id);

-- ============================================================
-- VENDORS
-- ============================================================

CREATE TABLE vendors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    email           TEXT,
    default_gl_account_id UUID REFERENCES gl_accounts(id),
    default_payment_method TEXT,
    country_code    CHAR(2),
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','inactive','blocked')),
    profile         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_vendors_org ON vendors(organization_id);

-- ============================================================
-- BILLS
-- ============================================================

CREATE TABLE bills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    vendor_id       UUID NOT NULL REFERENCES vendors(id),
    bill_number     TEXT,
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','pending_approval','approved','scheduled',
                                      'paid','partially_paid','overdue','cancelled')),
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bills_org ON bills(organization_id);
CREATE INDEX idx_bills_vendor ON bills(vendor_id);
CREATE INDEX idx_bills_status ON bills(organization_id, status);

-- ============================================================
-- POLICIES
-- ============================================================

CREATE TABLE policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    description     TEXT,
    priority        INT NOT NULL DEFAULT 100,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    rules           JSONB NOT NULL,
    -- Scope managed via graph edges (governed_by edges from users/depts to policy)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_policies_org ON policies(organization_id);

-- ============================================================
-- INTEGRATIONS
-- ============================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'connected',
    config          JSONB NOT NULL DEFAULT '{}',
    mappings        JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_integrations_org ON integrations(organization_id);

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
    changes         JSONB,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);

-- Reference data
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

## Graph Query Examples

```sql
-- ============================================================
-- QUERY: Full approval chain for a user
-- "Who can approve expenses for user X, in order of authority?"
-- ============================================================

WITH RECURSIVE approval_chain AS (
    -- Start with the user's direct manager
    SELECT
        e.target_node_id AS approver_node_id,
        n.label AS approver_name,
        n.ref_id AS approver_user_id,
        1 AS level,
        (e.properties->>'max_amount')::NUMERIC AS max_approval_amount
    FROM graph_edges e
    JOIN graph_nodes n ON n.id = e.target_node_id
    WHERE e.source_node_id = (
        SELECT id FROM graph_nodes WHERE ref_table = 'users' AND ref_id = $1
    )
    AND e.edge_type = 'reports_to'
    AND e.valid_to IS NULL

    UNION ALL

    -- Walk up the management chain
    SELECT
        e.target_node_id,
        n.label,
        n.ref_id,
        ac.level + 1,
        (e.properties->>'max_amount')::NUMERIC
    FROM approval_chain ac
    JOIN graph_edges e ON e.source_node_id = ac.approver_node_id
        AND e.edge_type = 'reports_to'
        AND e.valid_to IS NULL
    JOIN graph_nodes n ON n.id = e.target_node_id
    WHERE ac.level < 10  -- prevent infinite loops
)
SELECT * FROM approval_chain ORDER BY level;

-- ============================================================
-- QUERY: Department spend roll-up using ltree
-- "Total spend for Engineering including all sub-departments"
-- ============================================================

SELECT
    d.name AS department_name,
    d.code AS department_code,
    SUM(t.billing_amount) AS total_spend,
    COUNT(*) AS transaction_count
FROM transactions t
JOIN departments d ON d.id = t.department_id
JOIN graph_nodes gn ON gn.ref_table = 'departments' AND gn.ref_id = d.id
WHERE gn.hierarchy_path <@ (
    -- Find the ltree path for "Engineering" department
    SELECT hierarchy_path FROM graph_nodes
    WHERE ref_table = 'departments'
      AND ref_id = $1  -- Engineering department UUID
)
AND t.authorized_at >= '2026-04-01'
AND t.authorized_at < '2026-07-01'
GROUP BY d.name, d.code
ORDER BY total_spend DESC;

-- ============================================================
-- QUERY: Budget hierarchy - total allocation and spend across
-- parent and child budgets
-- ============================================================

WITH RECURSIVE budget_tree AS (
    -- Start with the parent budget
    SELECT
        b.id,
        b.name,
        b.limit_amount,
        b.spent_amount,
        b.currency_code,
        0 AS depth,
        ARRAY[b.id] AS path
    FROM budgets b
    WHERE b.id = $1  -- parent budget UUID

    UNION ALL

    -- Find child budgets via funded_by edges
    SELECT
        b.id,
        b.name,
        b.limit_amount,
        b.spent_amount,
        b.currency_code,
        bt.depth + 1,
        bt.path || b.id
    FROM budget_tree bt
    JOIN graph_edges e ON e.target_node_id = (
        SELECT id FROM graph_nodes WHERE ref_table = 'budgets' AND ref_id = bt.id
    )
    AND e.edge_type = 'funded_by'
    AND e.valid_to IS NULL
    JOIN graph_nodes sn ON sn.id = e.source_node_id AND sn.ref_table = 'budgets'
    JOIN budgets b ON b.id = sn.ref_id
    WHERE NOT (b.id = ANY(bt.path))  -- prevent cycles
)
SELECT
    id,
    name,
    depth,
    limit_amount,
    spent_amount,
    limit_amount - spent_amount AS remaining,
    ROUND(spent_amount / NULLIF(limit_amount, 0) * 100, 1) AS utilization_pct
FROM budget_tree
ORDER BY depth, name;

-- ============================================================
-- QUERY: Conflict of interest detection
-- "Find vendors where any of their contacts also appear as
--  employees with approval authority in this organization"
-- ============================================================

SELECT DISTINCT
    v.name AS vendor_name,
    u.full_name AS employee_name,
    u.email AS employee_email,
    e_approves.properties->>'max_amount' AS approval_limit,
    e_vendor.properties->>'relationship' AS vendor_relationship
FROM graph_edges e_vendor
JOIN graph_nodes vn ON vn.id = e_vendor.source_node_id
    AND vn.node_type = 'vendor'
JOIN vendors v ON v.id = vn.ref_id
JOIN graph_nodes un ON un.id = e_vendor.target_node_id
    AND un.node_type = 'user'
JOIN users u ON u.id = un.ref_id
-- Check if this user also has approval authority
JOIN graph_edges e_approves ON e_approves.source_node_id = un.id
    AND e_approves.edge_type = 'approves_for'
    AND e_approves.valid_to IS NULL
WHERE e_vendor.edge_type = 'transacts_with'
  AND e_vendor.organization_id = $1
  AND e_vendor.valid_to IS NULL;

-- ============================================================
-- QUERY: Policy scope resolution
-- "Find all policies that govern a specific user, including
--  policies inherited from their department, entity, and org"
-- ============================================================

SELECT
    p.id AS policy_id,
    p.name AS policy_name,
    p.priority,
    gn_target.node_type AS applied_via,
    gn_target.label AS applied_via_name
FROM graph_edges e
JOIN graph_nodes gn_source ON gn_source.id = e.source_node_id
JOIN graph_nodes gn_target ON gn_target.id = e.target_node_id
JOIN policies p ON p.id = gn_target.ref_id
WHERE e.edge_type = 'governed_by'
  AND e.valid_to IS NULL
  AND gn_source.ref_id IN (
      -- The user themselves
      $1,
      -- Their department (via belongs_to edge)
      (SELECT gn2.ref_id FROM graph_edges e2
       JOIN graph_nodes gn2 ON gn2.id = e2.target_node_id
       WHERE e2.source_node_id = (SELECT id FROM graph_nodes WHERE ref_table = 'users' AND ref_id = $1)
         AND e2.edge_type = 'belongs_to'
         AND gn2.node_type = 'department'
         AND e2.valid_to IS NULL
       LIMIT 1),
      -- Their entity (via member_of edge)
      (SELECT gn3.ref_id FROM graph_edges e3
       JOIN graph_nodes gn3 ON gn3.id = e3.target_node_id
       WHERE e3.source_node_id = (SELECT id FROM graph_nodes WHERE ref_table = 'users' AND ref_id = $1)
         AND e3.edge_type = 'member_of'
         AND gn3.node_type = 'entity'
         AND e3.valid_to IS NULL
       LIMIT 1)
  )
ORDER BY p.priority;

-- ============================================================
-- QUERY: Organizational chart traversal using ltree
-- "All users under the VP of Engineering, at any depth"
-- ============================================================

SELECT
    u.full_name,
    u.email,
    d.name AS department,
    subpath(gn_user.hierarchy_path,
            nlevel((SELECT hierarchy_path FROM graph_nodes WHERE ref_table = 'users' AND ref_id = $1)),
            1) AS relative_position
FROM graph_nodes gn_user
JOIN users u ON u.id = gn_user.ref_id
LEFT JOIN graph_edges e_dept ON e_dept.source_node_id = gn_user.id
    AND e_dept.edge_type = 'belongs_to'
    AND e_dept.valid_to IS NULL
LEFT JOIN graph_nodes gn_dept ON gn_dept.id = e_dept.target_node_id
LEFT JOIN departments d ON d.id = gn_dept.ref_id
WHERE gn_user.node_type = 'user'
  AND gn_user.hierarchy_path <@ (
      SELECT hierarchy_path FROM graph_nodes
      WHERE ref_table = 'users' AND ref_id = $1  -- VP of Engineering
  )
  AND gn_user.ref_id != $1  -- exclude the VP themselves
  AND gn_user.status = 'active'
ORDER BY gn_user.hierarchy_path;
```

---

## Graph Maintenance

```sql
-- ============================================================
-- FUNCTIONS: Graph consistency helpers
-- ============================================================

-- Function to create a graph node when a relational record is inserted
CREATE OR REPLACE FUNCTION create_graph_node(
    p_org_id UUID,
    p_node_type TEXT,
    p_ref_table TEXT,
    p_ref_id UUID,
    p_label TEXT,
    p_hierarchy_path LTREE DEFAULT NULL,
    p_properties JSONB DEFAULT '{}'
) RETURNS UUID AS $$
DECLARE
    v_node_id UUID;
BEGIN
    INSERT INTO graph_nodes (organization_id, node_type, ref_table, ref_id,
                             label, hierarchy_path, properties)
    VALUES (p_org_id, p_node_type, p_ref_table, p_ref_id,
            p_label, p_hierarchy_path, p_properties)
    RETURNING id INTO v_node_id;
    RETURN v_node_id;
END;
$$ LANGUAGE plpgsql;

-- Function to create a graph edge
CREATE OR REPLACE FUNCTION create_graph_edge(
    p_org_id UUID,
    p_source_ref_table TEXT,
    p_source_ref_id UUID,
    p_target_ref_table TEXT,
    p_target_ref_id UUID,
    p_edge_type TEXT,
    p_properties JSONB DEFAULT '{}'
) RETURNS UUID AS $$
DECLARE
    v_source_node UUID;
    v_target_node UUID;
    v_edge_id UUID;
BEGIN
    SELECT id INTO v_source_node FROM graph_nodes
    WHERE ref_table = p_source_ref_table AND ref_id = p_source_ref_id;

    SELECT id INTO v_target_node FROM graph_nodes
    WHERE ref_table = p_target_ref_table AND ref_id = p_target_ref_id;

    IF v_source_node IS NULL OR v_target_node IS NULL THEN
        RAISE EXCEPTION 'Source or target node not found';
    END IF;

    INSERT INTO graph_edges (organization_id, source_node_id, target_node_id,
                             edge_type, properties)
    VALUES (p_org_id, v_source_node, v_target_node, p_edge_type, p_properties)
    RETURNING id INTO v_edge_id;
    RETURN v_edge_id;
END;
$$ LANGUAGE plpgsql;

-- Function to "soft-close" an edge (set valid_to) instead of deleting
CREATE OR REPLACE FUNCTION close_graph_edge(
    p_edge_id UUID
) RETURNS VOID AS $$
BEGIN
    UPDATE graph_edges SET valid_to = now() WHERE id = p_edge_id;
END;
$$ LANGUAGE plpgsql;

-- Materialized view for common "user with all relationships" query
CREATE MATERIALIZED VIEW mv_user_context AS
SELECT
    u.id AS user_id,
    u.organization_id,
    u.full_name,
    u.email,
    u.status,
    d.id AS department_id,
    d.name AS department_name,
    d.code AS department_code,
    mgr.id AS manager_id,
    mgr.full_name AS manager_name,
    e_entity.ref_id AS entity_id,
    e_entity.label AS entity_name,
    gn_user.hierarchy_path,
    ARRAY_AGG(DISTINCT r.name) FILTER (WHERE r.name IS NOT NULL) AS role_names
FROM users u
JOIN graph_nodes gn_user ON gn_user.ref_table = 'users' AND gn_user.ref_id = u.id
-- Department
LEFT JOIN graph_edges ge_dept ON ge_dept.source_node_id = gn_user.id
    AND ge_dept.edge_type = 'belongs_to' AND ge_dept.valid_to IS NULL
LEFT JOIN graph_nodes gn_dept ON gn_dept.id = ge_dept.target_node_id AND gn_dept.node_type = 'department'
LEFT JOIN departments d ON d.id = gn_dept.ref_id
-- Manager
LEFT JOIN graph_edges ge_mgr ON ge_mgr.source_node_id = gn_user.id
    AND ge_mgr.edge_type = 'reports_to' AND ge_mgr.valid_to IS NULL
LEFT JOIN graph_nodes gn_mgr ON gn_mgr.id = ge_mgr.target_node_id
LEFT JOIN users mgr ON mgr.id = gn_mgr.ref_id
-- Entity
LEFT JOIN graph_edges ge_ent ON ge_ent.source_node_id = gn_user.id
    AND ge_ent.edge_type = 'member_of' AND ge_ent.valid_to IS NULL
LEFT JOIN graph_nodes e_entity ON e_entity.id = ge_ent.target_node_id AND e_entity.node_type = 'entity'
-- Roles (via governed_by edges to role nodes, or a simpler role assignment)
LEFT JOIN graph_edges ge_role ON ge_role.source_node_id = gn_user.id
    AND ge_role.edge_type = 'governed_by' AND ge_role.valid_to IS NULL
LEFT JOIN graph_nodes gn_role ON gn_role.id = ge_role.target_node_id AND gn_role.node_type = 'policy'
LEFT JOIN roles r ON r.id = gn_role.ref_id
GROUP BY u.id, u.organization_id, u.full_name, u.email, u.status,
         d.id, d.name, d.code, mgr.id, mgr.full_name,
         e_entity.ref_id, e_entity.label, gn_user.hierarchy_path;

CREATE UNIQUE INDEX idx_mv_user_context ON mv_user_context(user_id);

-- Refresh materialized view on user/edge changes
-- (in production, triggered by application or pg_cron)
-- REFRESH MATERIALIZED VIEW CONCURRENTLY mv_user_context;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Infrastructure | 2 | `graph_nodes`, `graph_edges` |
| Organizations & Entities | 2 | `organizations`, `entities` |
| Users & Roles | 2 | `users`, `roles` |
| Departments | 1 | `departments` |
| Cards | 1 | `cards` |
| GL Accounts | 1 | `gl_accounts` |
| Transactions | 1 | `transactions` |
| Budgets | 1 | `budgets` |
| Receipts | 1 | `receipts` |
| Reimbursements | 1 | `reimbursements` |
| Vendors & Bills | 2 | `vendors`, `bills` |
| Policies | 1 | `policies` |
| Integrations | 1 | `integrations` |
| Audit | 1 | `audit_log` |
| Reference Data | 2 | `currencies`, `merchant_category_codes` |
| Materialized Views | 1 | `mv_user_context` |
| **Total** | **21** | Plus 1 materialized view, 3 functions |

---

## Key Design Decisions

1. **Dual-layer architecture: graph for relationships, relational for operations** — the graph layer (2 tables: `graph_nodes` and `graph_edges`) handles all relationship queries (org hierarchy, approval chains, budget ownership, policy scope). The relational layer (19 tables) handles operational CRUD (transactions, cards, budgets). This separation means relationship queries don't pollute operational tables with self-referential foreign keys and recursive patterns.

2. **`ltree` for hierarchy performance** — PostgreSQL's `ltree` extension enables `<@` (is descendant of) and `@>` (is ancestor of) operators with GiST index support. Department roll-up spend reporting that would require recursive CTEs in the normalized model becomes a single indexed `WHERE hierarchy_path <@ 'org.entity_us.dept_eng'` clause.

3. **Temporal edges for historical relationship tracking** — `valid_from` / `valid_to` on edges enables "who was this person's manager on March 1?" queries without a separate history table. Edges are never deleted; they are "closed" by setting `valid_to`. This provides built-in audit trail for all relationship changes.

4. **Edge properties for authorization metadata** — the `approves_for` edge carries `max_amount` and `categories` in its JSONB `properties`, so the approval chain query returns not just who can approve but what they can approve up to. Similarly, `transacts_with` edges accumulate `total_spend_ytd` for vendor relationship analysis.

5. **Conflict-of-interest detection as a first-class query** — the graph model makes it trivial to find people who appear in both the "employee" and "vendor contact" contexts by looking for shared nodes with both `transacts_with` and `approves_for` edges. In the normalized model, this query would require joining across users, vendors, vendor contacts, and approval workflow tables.

6. **Policy scope via graph edges instead of junction tables** — rather than a `policy_assignments` table with `target_type` and `target_id` columns, policy scope is expressed as `governed_by` edges from users, departments, or entities to policy nodes. This unifies policy scope resolution with the same graph traversal engine used for approval chains and org hierarchy.

7. **Materialized view for common user context** — the `mv_user_context` materialized view pre-joins the most common user-department-manager-entity-role graph traversal, avoiding the multi-join cost on every API request. Refreshed periodically or on org change events.

8. **Graph nodes reference relational records** — `graph_nodes.ref_table` and `graph_nodes.ref_id` point back to the canonical relational record. The graph layer is an overlay, not a replacement. Deleting a user from the `users` table should cascade to deactivating their graph node.

9. **Edge type enumeration for safety** — the `edge_type` CHECK constraint lists all valid edge types, preventing typos and ensuring queries can enumerate all relationship types. New edge types require an ALTER TABLE (or a reference table lookup) to add.

10. **Budget hierarchy as a graph subproblem** — parent-child budget relationships (`funded_by` edges), budget-to-department allocations (`allocated_to` edges), and card-to-budget assignments (`charged_to` edges) are all modeled as graph edges. This enables the budget tree query to traverse arbitrarily deep budget hierarchies using the same recursive pattern as the org chart.
