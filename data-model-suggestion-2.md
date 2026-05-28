# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Corporate Card & Spend Management · Created: 2026-05-12

## Philosophy

This model treats every state change as an immutable domain event appended to an event store. The event store is the single source of truth; all "current state" views (card balances, budget utilization, expense status) are materialized read models rebuilt from the event stream. The architecture follows Command Query Responsibility Segregation (CQRS): writes go through command handlers that validate business rules and emit events, reads are served from denormalized projections optimized for each query pattern.

This pattern is proven in financial systems where regulatory compliance demands complete audit trails. Banks, trading platforms, and payment processors (including Stripe's internal ledger) use event sourcing to provide bit-perfect reconstruction of any state at any point in time. The approach answers questions like "what was the budget utilization at 3pm on March 15?" or "who changed the policy that allowed this transaction?" without separate audit tables — the event log IS the audit trail.

The event-sourced model is ideal for organizations operating under SOX, SOC 2, or financial regulatory requirements where auditors demand provable, tamper-evident records of every action. It also enables powerful AI analytics: the full event stream provides training data for anomaly detection, spend pattern analysis, and predictive budget forecasting that would require complex joins across multiple tables in a normalized model.

**Best for:** Organizations with strict compliance/audit requirements, teams building AI-driven analytics on spend patterns, and deployments where temporal queries ("what was true at time T?") are a core requirement.

**Trade-offs:**
- (+) Complete, immutable audit trail by construction — every change is recorded, not just final state
- (+) Temporal queries are trivial: replay events up to any timestamp to reconstruct past state
- (+) AI/ML training on event streams reveals spend patterns impossible to detect from snapshot data
- (+) Independent scaling of read and write paths; read models can be rebuilt without downtime
- (+) New read models (new report, new dashboard) can be added without schema migration
- (-) Higher storage requirements — events are never deleted, only compacted via snapshots
- (-) Eventual consistency between event store and read models adds complexity
- (-) Debugging requires understanding event replay; harder for SQL-native teams
- (-) Schema evolution of events requires careful versioning (upcasting)
- (-) Simple CRUD queries require maintaining projections; overkill for small organizations

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 4217 | All monetary event payloads include `currency_code` per ISO 4217 |
| ISO 18245 (MCC) | `TransactionAuthorized` events include `mcc_code` from card network |
| ISO 3166-1 | Jurisdiction-related events carry `country_code` for multi-region compliance |
| PCI DSS 4.0.1 | Card-related events store only tokens, never PANs; event store encryption at rest |
| NACHA ACH | `ReimbursementPaymentInitiated` events include NACHA-compliant fields |
| OCSF | Event schema structure influenced by Open Cybersecurity Schema Framework for structured security event logging |
| ISO 20022 | Cross-border payment events structured to map to `pain.001` fields |
| CloudEvents 1.0 | Event envelope follows CloudEvents spec for interoperability with event-driven integrations |

---

## Event Store (Source of Truth)

```sql
-- ============================================================
-- CORE EVENT STORE
-- ============================================================

-- The immutable append-only event log. This is the single source of truth.
-- All other tables are materialized projections rebuilt from these events.

CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  TEXT NOT NULL,                  -- 'transaction','card','budget','policy',
                                                   -- 'reimbursement','bill','user','organization'
    aggregate_id    UUID NOT NULL,                  -- the entity this event belongs to
    organization_id UUID NOT NULL,                  -- tenant isolation
    event_type      TEXT NOT NULL,                  -- e.g. 'TransactionAuthorized',
                                                   -- 'BudgetLimitChanged', 'PolicyViolationDetected'
    event_version   INT NOT NULL,                   -- schema version for upcasting
    sequence_number BIGINT NOT NULL,                -- per-aggregate ordering
    payload         JSONB NOT NULL,                 -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',    -- actor, ip, user_agent, correlation_id
    caused_by       UUID,                           -- event that triggered this event (causation)
    correlation_id  UUID,                           -- groups related events across aggregates
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Ensure ordering within an aggregate
    UNIQUE(aggregate_type, aggregate_id, sequence_number)
);

-- Primary query: replay events for an aggregate
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_number);

-- Tenant isolation
CREATE INDEX idx_events_org ON events(organization_id, created_at);

-- Event type filtering (for projections and subscriptions)
CREATE INDEX idx_events_type ON events(event_type, created_at);

-- Correlation tracking
CREATE INDEX idx_events_correlation ON events(correlation_id);

-- Time-range queries for analytics
CREATE INDEX idx_events_org_time ON events(organization_id, created_at)
    INCLUDE (aggregate_type, event_type);

-- Partition by month for performance and retention management
-- In production: CREATE TABLE events PARTITION BY RANGE (created_at);

-- ============================================================
-- AGGREGATE SNAPSHOTS (performance optimization)
-- ============================================================

-- Periodic snapshots to avoid replaying entire event history
CREATE TABLE aggregate_snapshots (
    aggregate_type  TEXT NOT NULL,
    aggregate_id    UUID NOT NULL,
    organization_id UUID NOT NULL,
    snapshot_version BIGINT NOT NULL,               -- sequence_number at time of snapshot
    state           JSONB NOT NULL,                 -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, snapshot_version)
);

-- ============================================================
-- OUTBOX (for reliable event publishing to message broker)
-- ============================================================

CREATE TABLE event_outbox (
    id              BIGSERIAL PRIMARY KEY,
    event_id        UUID NOT NULL REFERENCES events(event_id),
    destination     TEXT NOT NULL,                  -- topic/queue name
    published       BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    retry_count     INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_outbox_unpublished ON event_outbox(published, created_at)
    WHERE published = false;
```

---

## Event Type Catalogue

```sql
-- ============================================================
-- EVENT TYPE REFERENCE
-- ============================================================

-- Documents all known event types and their payload schemas.
-- Used for validation, documentation, and schema evolution tracking.

CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    aggregate_type  TEXT NOT NULL,
    description     TEXT NOT NULL,
    current_version INT NOT NULL DEFAULT 1,
    payload_schema  JSONB NOT NULL,                -- JSON Schema for validation
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types and their payloads (inserted as reference data):
-- 
-- TransactionAuthorized:
-- {
--   "card_id": "uuid",
--   "cardholder_id": "uuid",
--   "amount": 125.50,
--   "currency_code": "USD",
--   "merchant_name": "AWS",
--   "merchant_id": "uuid",
--   "mcc_code": "5734",
--   "authorization_method": "online",
--   "issuer_authorization_id": "iauth_abc123",
--   "billing_amount": 125.50,
--   "billing_currency": "USD"
-- }
--
-- TransactionCleared:
-- {
--   "authorization_id": "uuid",
--   "cleared_amount": 125.50,
--   "cleared_currency": "USD",
--   "settlement_date": "2026-05-12"
-- }
--
-- TransactionCoded:
-- {
--   "gl_account_code": "6200",
--   "department_code": "ENG",
--   "cost_center_code": "CC-100",
--   "project_code": "P-INFRA",
--   "coding_method": "ai_auto",
--   "ai_confidence": 0.94,
--   "split_lines": [
--     {"gl_account_code": "6200", "amount": 100.00, "memo": "Compute"},
--     {"gl_account_code": "6210", "amount": 25.50, "memo": "Storage"}
--   ]
-- }
--
-- PolicyViolationDetected:
-- {
--   "policy_id": "uuid",
--   "rule_id": "uuid",
--   "violation_type": "over_limit",
--   "severity": "warning",
--   "transaction_amount": 500.00,
--   "policy_limit": 250.00,
--   "ai_review_score": 0.87,
--   "recommended_action": "flag"
-- }
--
-- BudgetAllocated:
-- {
--   "budget_name": "Engineering Q2",
--   "limit_amount": 50000.00,
--   "currency_code": "USD",
--   "period": "quarterly",
--   "period_start": "2026-04-01",
--   "period_end": "2026-06-30",
--   "owner_id": "uuid",
--   "department_id": "uuid"
-- }
--
-- BudgetSpendRecorded:
-- {
--   "transaction_id": "uuid",
--   "amount": 125.50,
--   "currency_code": "USD",
--   "running_total": 23456.78,
--   "utilization_pct": 46.91,
--   "alert_triggered": false
-- }
--
-- CardCreated:
-- {
--   "card_type": "virtual",
--   "card_purpose": "vendor_locked",
--   "issuer": "stripe_issuing",
--   "issuer_card_id": "ic_abc123",
--   "last_four": "4242",
--   "network": "visa",
--   "currency_code": "USD",
--   "spend_limit_amount": 1000.00,
--   "spend_limit_interval": "monthly",
--   "cardholder_id": "uuid"
-- }
--
-- ReimbursementSubmitted:
-- {
--   "employee_id": "uuid",
--   "amount": 89.99,
--   "currency_code": "USD",
--   "expense_date": "2026-05-10",
--   "merchant_name": "Office Depot",
--   "mcc_code": "5943",
--   "receipt_ids": ["uuid1", "uuid2"],
--   "memo": "Office supplies for Q2 offsite"
-- }
--
-- ReimbursementApproved:
-- {
--   "approved_by": "uuid",
--   "approved_amount": 89.99,
--   "gl_account_code": "6100",
--   "department_code": "ENG"
-- }
--
-- ReimbursementPaid:
-- {
--   "payment_method": "ach",
--   "payment_reference": "ACH-2026-05-12-001",
--   "amount": 89.99,
--   "currency_code": "USD",
--   "paid_at": "2026-05-12T15:30:00Z"
-- }
```

---

## Read Model Projections

```sql
-- ============================================================
-- PROJECTION: Current Transaction State
-- ============================================================

-- Materialized from: TransactionAuthorized, TransactionCleared,
-- TransactionDeclined, TransactionReversed, TransactionCoded,
-- ReceiptAttached, PolicyViolationDetected

CREATE TABLE rm_transactions (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    entity_id       UUID,
    card_id         UUID NOT NULL,
    cardholder_id   UUID NOT NULL,
    
    -- Amounts
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    billing_amount  NUMERIC(15,2),
    billing_currency CHAR(3),
    fx_rate         NUMERIC(12,6),
    
    -- Merchant
    merchant_id     UUID,
    merchant_name   TEXT,
    mcc_code        CHAR(4),
    
    -- Issuer
    issuer_authorization_id TEXT,
    issuer_transaction_id TEXT,
    authorization_method TEXT,
    
    -- Status (derived from latest event)
    status          TEXT NOT NULL,
    transaction_type TEXT NOT NULL,
    
    -- GL Coding (denormalized from TransactionCoded events)
    gl_account_code TEXT,
    gl_account_name TEXT,
    department_code TEXT,
    department_name TEXT,
    cost_center_code TEXT,
    project_code    TEXT,
    coding_method   TEXT,
    ai_confidence   NUMERIC(3,2),
    is_split_coded  BOOLEAN NOT NULL DEFAULT false,
    
    -- Receipt
    has_receipt     BOOLEAN NOT NULL DEFAULT false,
    receipt_count   INT NOT NULL DEFAULT 0,
    
    -- Policy
    has_violation   BOOLEAN NOT NULL DEFAULT false,
    violation_count INT NOT NULL DEFAULT 0,
    violation_resolved BOOLEAN,
    
    -- Budget
    budget_id       UUID,
    budget_name     TEXT,
    
    -- Timestamps (from events)
    authorized_at   TIMESTAMPTZ,
    cleared_at      TIMESTAMPTZ,
    coded_at        TIMESTAMPTZ,
    
    -- Projection metadata
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    projection_version BIGINT NOT NULL
);
CREATE INDEX idx_rm_txn_org ON rm_transactions(organization_id);
CREATE INDEX idx_rm_txn_card ON rm_transactions(card_id);
CREATE INDEX idx_rm_txn_cardholder ON rm_transactions(cardholder_id);
CREATE INDEX idx_rm_txn_status ON rm_transactions(organization_id, status);
CREATE INDEX idx_rm_txn_date ON rm_transactions(organization_id, authorized_at);
CREATE INDEX idx_rm_txn_mcc ON rm_transactions(mcc_code);
CREATE INDEX idx_rm_txn_no_receipt ON rm_transactions(organization_id)
    WHERE has_receipt = false AND status = 'cleared';

-- ============================================================
-- PROJECTION: Current Card State
-- ============================================================

CREATE TABLE rm_cards (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    card_programme_id UUID NOT NULL,
    cardholder_id   UUID NOT NULL,
    cardholder_name TEXT NOT NULL,
    card_token      TEXT NOT NULL,
    issuer_card_id  TEXT NOT NULL,
    card_type       TEXT NOT NULL,
    card_purpose    TEXT NOT NULL,
    last_four       CHAR(4),
    network         TEXT,
    currency_code   CHAR(3) NOT NULL,
    spend_limit_amount NUMERIC(15,2),
    spend_limit_interval TEXT,
    current_period_spend NUMERIC(15,2) NOT NULL DEFAULT 0,
    total_spend     NUMERIC(15,2) NOT NULL DEFAULT 0,
    transaction_count INT NOT NULL DEFAULT 0,
    status          TEXT NOT NULL,
    budget_id       UUID,
    budget_name     TEXT,
    locked_vendor_name TEXT,
    locked_project_name TEXT,
    activated_at    TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    projection_version BIGINT NOT NULL
);
CREATE INDEX idx_rm_cards_org ON rm_cards(organization_id);
CREATE INDEX idx_rm_cards_cardholder ON rm_cards(cardholder_id);
CREATE INDEX idx_rm_cards_status ON rm_cards(organization_id, status);

-- ============================================================
-- PROJECTION: Current Budget State
-- ============================================================

CREATE TABLE rm_budgets (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    entity_id       UUID,
    name            TEXT NOT NULL,
    budget_type     TEXT NOT NULL,
    owner_id        UUID NOT NULL,
    owner_name      TEXT NOT NULL,
    department_id   UUID,
    department_name TEXT,
    project_id      UUID,
    project_name    TEXT,
    limit_amount    NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    period          TEXT NOT NULL,
    period_start    DATE,
    period_end      DATE,
    
    -- Live spend tracking (updated on every BudgetSpendRecorded event)
    spent_amount    NUMERIC(15,2) NOT NULL DEFAULT 0,
    committed_amount NUMERIC(15,2) NOT NULL DEFAULT 0,
    available_amount NUMERIC(15,2) NOT NULL,
    utilization_pct NUMERIC(5,2) NOT NULL DEFAULT 0,
    
    -- Pre-spend model
    is_pre_funded   BOOLEAN NOT NULL DEFAULT false,
    funded_amount   NUMERIC(15,2) DEFAULT 0,
    
    -- Alerts
    alert_threshold_pct NUMERIC(5,2),
    alert_triggered BOOLEAN NOT NULL DEFAULT false,
    
    status          TEXT NOT NULL,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    projection_version BIGINT NOT NULL
);
CREATE INDEX idx_rm_budgets_org ON rm_budgets(organization_id);
CREATE INDEX idx_rm_budgets_owner ON rm_budgets(owner_id);
CREATE INDEX idx_rm_budgets_dept ON rm_budgets(department_id);

-- ============================================================
-- PROJECTION: Policy Violation Dashboard
-- ============================================================

CREATE TABLE rm_policy_violations (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    transaction_id  UUID,
    reimbursement_id UUID,
    policy_id       UUID NOT NULL,
    policy_name     TEXT NOT NULL,
    rule_description TEXT,
    user_id         UUID NOT NULL,
    user_name       TEXT NOT NULL,
    department_name TEXT,
    violation_type  TEXT NOT NULL,
    severity        TEXT NOT NULL,
    description     TEXT,
    amount          NUMERIC(15,2),
    currency_code   CHAR(3),
    resolution      TEXT NOT NULL DEFAULT 'pending',
    resolved_by_name TEXT,
    resolved_at     TIMESTAMPTZ,
    ai_review_score NUMERIC(3,2),
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL,
    projection_version BIGINT NOT NULL
);
CREATE INDEX idx_rm_violations_org ON rm_policy_violations(organization_id);
CREATE INDEX idx_rm_violations_status ON rm_policy_violations(organization_id, resolution);
CREATE INDEX idx_rm_violations_user ON rm_policy_violations(user_id);

-- ============================================================
-- PROJECTION: Reimbursement State
-- ============================================================

CREATE TABLE rm_reimbursements (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    entity_id       UUID,
    employee_id     UUID NOT NULL,
    employee_name   TEXT NOT NULL,
    department_name TEXT,
    title           TEXT NOT NULL,
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    expense_date    DATE NOT NULL,
    merchant_name   TEXT,
    gl_account_code TEXT,
    gl_account_name TEXT,
    has_receipt     BOOLEAN NOT NULL DEFAULT false,
    receipt_count   INT NOT NULL DEFAULT 0,
    status          TEXT NOT NULL,
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    approved_by_name TEXT,
    paid_at         TIMESTAMPTZ,
    payment_method  TEXT,
    payment_reference TEXT,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    projection_version BIGINT NOT NULL
);
CREATE INDEX idx_rm_reimb_org ON rm_reimbursements(organization_id);
CREATE INDEX idx_rm_reimb_employee ON rm_reimbursements(employee_id);
CREATE INDEX idx_rm_reimb_status ON rm_reimbursements(organization_id, status);

-- ============================================================
-- PROJECTION: Spend Analytics (pre-aggregated)
-- ============================================================

CREATE TABLE rm_spend_by_period (
    organization_id UUID NOT NULL,
    entity_id       UUID,
    period_type     TEXT NOT NULL,                  -- 'daily','weekly','monthly'
    period_start    DATE NOT NULL,
    department_id   UUID,
    department_name TEXT,
    gl_account_code TEXT,
    gl_account_name TEXT,
    mcc_code        CHAR(4),
    mcc_description TEXT,
    vendor_name     TEXT,
    
    transaction_count INT NOT NULL DEFAULT 0,
    total_amount    NUMERIC(15,2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL,
    avg_amount      NUMERIC(15,2),
    max_amount      NUMERIC(15,2),
    
    last_event_id   UUID NOT NULL,
    projection_version BIGINT NOT NULL,
    
    PRIMARY KEY (organization_id, period_type, period_start,
                 COALESCE(entity_id, '00000000-0000-0000-0000-000000000000'),
                 COALESCE(department_id, '00000000-0000-0000-0000-000000000000'),
                 COALESCE(gl_account_code, ''),
                 COALESCE(mcc_code, ''))
);
CREATE INDEX idx_rm_spend_period ON rm_spend_by_period(organization_id, period_type, period_start);
```

---

## Supporting Tables (Non-Event)

```sql
-- ============================================================
-- CONFIGURATION TABLES (mutable, not event-sourced)
-- These store configuration that doesn't need event-sourced history.
-- Changes to these tables ARE logged as events for audit purposes.
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    legal_name      TEXT,
    country_code    CHAR(2) NOT NULL,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    status          TEXT NOT NULL DEFAULT 'active',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    employee_id     TEXT,
    department_id   UUID,
    manager_id      UUID REFERENCES users(id),
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);
CREATE INDEX idx_users_org ON users(organization_id);

-- Reference data (same as Model 1)
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

-- Integration connections (mutable state, changes logged as events)
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'connected',
    oauth_access_token_encrypted TEXT,
    oauth_refresh_token_encrypted TEXT,
    oauth_token_expires_at TIMESTAMPTZ,
    external_entity_id TEXT,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- File storage references for receipts
CREATE TABLE receipt_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    file_url        TEXT NOT NULL,
    file_type       TEXT,
    file_size_bytes BIGINT,
    uploaded_by     UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION CHECKPOINTS
-- ============================================================

-- Tracks where each projection has processed up to in the event stream
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

```sql
-- ============================================================
-- TEMPORAL QUERY: What was the budget utilization at a specific time?
-- ============================================================

-- Replay BudgetSpendRecorded events up to a specific timestamp
SELECT
    e.aggregate_id AS budget_id,
    (e.payload->>'running_total')::NUMERIC AS spend_at_time,
    (e.payload->>'utilization_pct')::NUMERIC AS utilization_at_time,
    e.created_at AS as_of
FROM events e
WHERE e.aggregate_type = 'budget'
  AND e.aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
  AND e.event_type = 'BudgetSpendRecorded'
  AND e.created_at <= '2026-03-15 15:00:00+00'
ORDER BY e.sequence_number DESC
LIMIT 1;

-- ============================================================
-- AUDIT QUERY: Full history of a transaction
-- ============================================================

SELECT
    e.event_type,
    e.payload,
    e.metadata->>'actor_id' AS actor,
    e.metadata->>'actor_type' AS actor_type,
    e.created_at
FROM events e
WHERE e.aggregate_type = 'transaction'
  AND e.aggregate_id = '550e8400-e29b-41d4-a716-446655440001'
ORDER BY e.sequence_number;

-- Returns timeline:
-- TransactionAuthorized  → card swipe at merchant
-- ReceiptAttached        → employee uploaded receipt
-- TransactionCoded       → AI auto-coded to GL 6200
-- PolicyViolationDetected → exceeded $250 policy limit
-- PolicyViolationResolved → manager approved exception
-- TransactionCleared     → settlement complete

-- ============================================================
-- ANALYTICS: Spend anomaly detection (event stream analysis)
-- ============================================================

-- Find transactions where amount exceeds 3x the cardholder's
-- rolling 30-day average (using event data)
WITH cardholder_history AS (
    SELECT
        e.payload->>'cardholder_id' AS cardholder_id,
        (e.payload->>'amount')::NUMERIC AS amount,
        e.created_at,
        AVG((e.payload->>'amount')::NUMERIC) OVER (
            PARTITION BY e.payload->>'cardholder_id'
            ORDER BY e.created_at
            RANGE BETWEEN INTERVAL '30 days' PRECEDING AND CURRENT ROW
        ) AS rolling_avg
    FROM events e
    WHERE e.event_type = 'TransactionAuthorized'
      AND e.organization_id = '550e8400-e29b-41d4-a716-446655440002'
      AND e.created_at >= now() - INTERVAL '90 days'
)
SELECT *
FROM cardholder_history
WHERE amount > rolling_avg * 3
ORDER BY created_at DESC;

-- ============================================================
-- REBUILD: Reconstruct current card state from events
-- ============================================================

-- This is how the rm_cards projection is rebuilt
WITH card_events AS (
    SELECT
        e.aggregate_id AS card_id,
        e.event_type,
        e.payload,
        e.created_at,
        e.event_id,
        ROW_NUMBER() OVER (
            PARTITION BY e.aggregate_id
            ORDER BY e.sequence_number DESC
        ) AS rn
    FROM events e
    WHERE e.aggregate_type = 'card'
      AND e.organization_id = '550e8400-e29b-41d4-a716-446655440002'
)
SELECT
    card_id,
    payload->>'card_type' AS card_type,
    payload->>'status' AS current_status,
    payload->>'last_four' AS last_four,
    created_at AS last_event_at
FROM card_events
WHERE rn = 1;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | `events`, `aggregate_snapshots`, `event_outbox` |
| Event Registry | 1 | `event_type_registry` |
| Read Model: Transactions | 1 | `rm_transactions` |
| Read Model: Cards | 1 | `rm_cards` |
| Read Model: Budgets | 1 | `rm_budgets` |
| Read Model: Violations | 1 | `rm_policy_violations` |
| Read Model: Reimbursements | 1 | `rm_reimbursements` |
| Read Model: Analytics | 1 | `rm_spend_by_period` |
| Configuration | 3 | `organizations`, `users`, `integrations` |
| Reference Data | 2 | `currencies`, `merchant_category_codes` |
| Infrastructure | 2 | `receipt_files`, `projection_checkpoints` |
| **Total** | **18** | Plus additional projections as needed |

---

## Key Design Decisions

1. **Single `events` table as source of truth** — all domain state changes are immutable events. The event store replaces 20+ mutable tables from the normalized model with a single append-only table, simplifying backup, replication, and compliance.

2. **Aggregate-based event grouping** — events are grouped by `aggregate_type` + `aggregate_id` with per-aggregate `sequence_number`, enabling efficient replay of individual entities (e.g., one transaction's full lifecycle) without scanning the entire event store.

3. **JSONB payloads with schema registry** — event data is stored as JSONB for flexibility, but the `event_type_registry` table documents expected schemas. This balances schema-on-read flexibility with discoverability and validation.

4. **Outbox pattern for reliable event publishing** — the `event_outbox` table implements the transactional outbox pattern: events are written to both `events` and `event_outbox` in the same transaction, then a background worker publishes to message brokers. This guarantees at-least-once delivery without distributed transactions.

5. **Denormalized read models prefixed with `rm_`** — all projection tables are clearly namespaced and treated as disposable. They can be dropped and rebuilt from the event store at any time, enabling zero-downtime schema evolution for read paths.

6. **Pre-aggregated analytics projection** — `rm_spend_by_period` pre-computes spend rollups by department, GL account, MCC code, and vendor at daily/weekly/monthly granularity. This avoids expensive real-time aggregation queries for dashboard rendering.

7. **Snapshot optimization** — `aggregate_snapshots` stores periodic state snapshots for aggregates with long event histories (e.g., budgets with thousands of spend events). Replay starts from the latest snapshot instead of event #1, keeping reconstruction fast.

8. **Temporal queries by construction** — answering "what was the budget utilization on March 15?" requires no special tables or SCD patterns. Simply replay events up to that timestamp. This is architecturally impossible to achieve with the same fidelity in a mutable-state model.

9. **Causation and correlation tracking** — `caused_by` and `correlation_id` fields enable tracing event chains: a single card swipe can trigger TransactionAuthorized → PolicyViolationDetected → ApprovalRequested → ApprovalGranted, all linked by correlation ID.

10. **Configuration tables remain mutable** — not everything benefits from event sourcing. Organization settings, user profiles, and integration tokens are mutable CRUD tables. Changes to these ARE logged as events for audit, but the current state is read from the mutable table, not replayed from events.
