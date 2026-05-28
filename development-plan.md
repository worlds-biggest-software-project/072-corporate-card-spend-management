# Corporate Card & Spend Management -- Development Plan

> Project: Corporate Card & Spend Management (Candidate #72)
> Created: 2026-05-25
> Status: Phased development plan

---

## Technology Decisions

### Language & Runtime

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Backend API** | TypeScript / Node.js (v22 LTS) | Stripe Issuing, Lithic, and Marqeta all publish first-class Node/TypeScript SDKs. The QuickBooks, Xero, and NetSuite integration ecosystem is strongest in JS/TS. TypeScript gives compile-time safety for financial data types without the cold-start penalty of JVM languages in serverless/container deployments. |
| **Web Frontend** | React 19 + Next.js 15 (App Router) | Dominant framework for B2B SaaS dashboards. Server Components reduce client bundle for data-heavy financial tables. The ecosystem has mature table, chart, and form component libraries (TanStack Table, Recharts, React Hook Form). |
| **Mobile** | React Native (Expo SDK 52) | Code-shared with web where possible. Receipt capture camera integration and push notifications for spend alerts are native requirements. Expo simplifies OTA updates for receipt-scanning UX iterations. |
| **AI/ML Services** | Python 3.12 microservices | OCR pipeline (Tesseract/PaddleOCR), GL coding model (fine-tuned transformer), and anomaly detection run as separate Python services behind gRPC. Python is the only viable choice for the ML toolchain (PyTorch, scikit-learn, Hugging Face). |

### Data Layer

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Primary Database** | PostgreSQL 16 | All four data model suggestions are PostgreSQL-native. RLS for multi-tenancy, JSONB for flexible fields, `ltree` for hierarchies, partitioning for audit logs. The industry standard for financial SaaS. |
| **Data Model** | Hybrid Relational + JSONB (Model 3) with selective elements from Models 1 and 2 | Model 3's 17-table design gives the fastest path to MVP while keeping monetary amounts and GL accounts in typed relational columns for aggregation safety. We adopt Model 1's separate `expense_codings` table (split-coding is a core requirement, and JSONB arrays are awkward to aggregate). We adopt Model 2's append-only `events` table for the audit trail (replacing a mutable audit_log with an immutable event store provides compliance-grade auditability without the full CQRS complexity). Model 4's graph layer is deferred to Phase 8 for multi-entity hierarchy queries. |
| **Cache** | Redis 7 (Valkey) | Budget utilization counters, rate limiting, session state, real-time spend alert pub/sub. |
| **Object Storage** | S3-compatible (AWS S3 / MinIO for self-hosted) | Receipt images, invoice PDFs, export files. |
| **Search** | PostgreSQL full-text search (tsvector) for MVP; OpenSearch deferred | Merchant name search, transaction memo search. OpenSearch added only if FTS performance degrades at scale. |
| **Message Queue** | BullMQ (Redis-backed) for MVP; NATS JetStream for scale | Webhook processing, integration sync jobs, OCR pipeline, notification delivery. BullMQ is zero-infra for MVP; NATS added when event throughput exceeds Redis capacity. |

### Infrastructure & DevOps

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Containerization** | Docker + Docker Compose (dev); Kubernetes (prod) | Self-hostable requirement demands container-native architecture. Compose for local dev; Helm charts for production K8s. |
| **CI/CD** | GitHub Actions | OSS-friendly, free for public repos, mature ecosystem for Node/Python matrix builds. |
| **IaC** | Terraform + Helm | Multi-cloud self-hosting target requires provider-agnostic IaC. |
| **Observability** | OpenTelemetry + Grafana stack (Loki, Tempo, Mimir) | OSS-native observability. OTel instrumentation from day one avoids retrofitting. |
| **Secrets** | HashiCorp Vault (prod); dotenv (dev) | PCI DSS 4.0.1 requires encrypted storage for integration tokens and API keys. |

### Authentication & Authorization

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Auth Provider** | OIDC/SAML 2.0 via configurable adapter (Keycloak self-hosted default; Auth0/Okta pluggable) | Enterprise SSO is table stakes per features research. SAML 2.0 required for companies >200 employees. Keycloak for self-hosted; cloud IdP adapters for SaaS deployment. |
| **API Auth** | OAuth 2.0 Authorization Code + PKCE (RFC 7636) for user sessions; API keys for server-to-server | FAPI 2.0 alignment per standards.md. PKCE mandatory for all client types in 2026. |
| **Multi-Tenancy** | PostgreSQL Row-Level Security (RLS) on `organization_id` | Database-level tenant isolation is defense-in-depth beyond application-layer checks. All four data models specify RLS. |

### API Design

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **API Style** | REST/JSON with OpenAPI 3.1 specification | Standards.md identifies OAS 3.1 as industry standard. Stripe, Brex, and Ramp all use REST. SDK generation from OpenAPI spec. |
| **Real-Time** | WebSocket (RFC 6455) for spend alerts and live transaction feed | Standards.md specifies WebSocket for real-time spend notifications. |
| **Webhook Delivery** | Outbox pattern with exponential backoff retries | Reliable event delivery to customer webhook endpoints for transaction, policy, and budget events. |

### Licence

| Choice | Rationale |
|--------|-----------|
| **Apache 2.0** | Features.md explicitly recommends against AGPL (Firefly III's licence is identified as unsuitable for SaaS). Apache 2.0 enables commercial adoption without copyleft obligations while remaining OSS. |

---

## Project Structure

```
corporate-card-spend-management/
|-- apps/
|   |-- api/                          # Main REST API (Node.js/Express or Fastify)
|   |   |-- src/
|   |   |   |-- modules/
|   |   |   |   |-- auth/             # Authentication, SSO, session management
|   |   |   |   |-- organizations/    # Org & entity CRUD, settings
|   |   |   |   |-- users/            # User management, roles, permissions
|   |   |   |   |-- cards/            # Card lifecycle, issuer integration
|   |   |   |   |-- transactions/     # Transaction ingestion, status management
|   |   |   |   |-- receipts/         # Receipt upload, OCR pipeline trigger
|   |   |   |   |-- budgets/          # Budget CRUD, utilization tracking
|   |   |   |   |-- policies/         # Policy engine, violation detection
|   |   |   |   |-- approvals/        # Approval workflow orchestration
|   |   |   |   |-- reimbursements/   # Expense reimbursement workflow
|   |   |   |   |-- vendors/          # Vendor management, contracts
|   |   |   |   |-- bills/            # AP automation, bill pay
|   |   |   |   |-- gl/               # Chart of accounts, coding rules
|   |   |   |   |-- integrations/     # ERP sync, card issuer connectors
|   |   |   |   |-- reports/          # Spend analytics, exports
|   |   |   |   |-- notifications/    # Email, Slack, push notifications
|   |   |   |-- middleware/           # Auth, RLS, rate limiting, error handling
|   |   |   |-- events/              # Event store, domain event publishers
|   |   |   |-- shared/              # Shared types, utilities, constants
|   |   |-- tests/
|   |   |-- openapi.yaml             # OpenAPI 3.1 specification
|   |-- web/                          # Next.js web dashboard
|   |   |-- src/
|   |   |   |-- app/                  # App Router pages
|   |   |   |-- components/           # Shared UI components
|   |   |   |-- hooks/                # Custom React hooks
|   |   |   |-- lib/                  # API client, utilities
|   |-- mobile/                       # React Native (Expo) mobile app
|-- packages/
|   |-- db/                           # Database migrations, seed data, types
|   |   |-- migrations/
|   |   |-- seeds/
|   |   |-- src/                      # Generated types, query builders
|   |-- ai/                           # Python ML services
|   |   |-- ocr/                      # Receipt OCR pipeline
|   |   |-- gl-coder/                 # GL auto-coding model
|   |   |-- anomaly/                  # Spend anomaly detection
|   |   |-- policy-agent/             # AI policy evaluation
|   |-- shared/                       # Shared TypeScript types & utilities
|   |-- card-issuer-adapters/         # Stripe/Lithic/Marqeta abstraction layer
|   |-- erp-adapters/                 # QuickBooks/Xero/NetSuite adapters
|-- infra/
|   |-- docker/                       # Dockerfiles, docker-compose.yml
|   |-- helm/                         # Kubernetes Helm charts
|   |-- terraform/                    # IaC modules
|-- docs/
|   |-- api/                          # Generated API docs from OpenAPI
|   |-- architecture/                 # ADRs, system diagrams
|-- .github/
|   |-- workflows/                    # CI/CD pipelines
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    |
    v
Phase 2: Card Programme Integration
    |
    +---------------------------+
    |                           |
    v                           v
Phase 3: Policy & Budget    Phase 4: Receipt & GL Coding
    |                           |
    +---------------------------+
    |
    v
Phase 5: Approval Workflows & Reimbursements
    |
    +---------------------------+
    |                           |
    v                           v
Phase 6: AP Automation      Phase 7: ERP Integrations
    |                           |
    +---------------------------+
    |
    v
Phase 8: Multi-Entity & Advanced Reporting
    |
    v
Phase 9: AI Policy Agent & Intelligence
    |
    v
Phase 10: Mobile App & Notifications
    |
    v
Phase 11: Security Hardening & Compliance
    |
    v
Phase 12: Self-Hosting, Docs & Launch
```

**Critical Path:** Phases 1 -> 2 -> 3 -> 5 -> 7 -> 11 -> 12

**Parallelizable:** Phases 3 and 4 can run concurrently. Phases 6 and 7 can run concurrently.

---

## Phase 1: Foundation

**Goal:** Establish project scaffolding, database schema, authentication, multi-tenant API skeleton, and CI/CD pipeline. All subsequent phases build on this foundation.

**Duration:** 3-4 weeks

### Task 1.1: Monorepo Scaffolding & Tooling

**What:** Initialize the monorepo with Turborepo, configure TypeScript, ESLint, Prettier, and create the workspace structure for `apps/api`, `apps/web`, `packages/db`, `packages/shared`, and `packages/card-issuer-adapters`.

**Design:**

```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "db:migrate": { "cache": false }
  }
}

// packages/shared/src/types/money.ts
export interface Money {
  amount: string;        // string to avoid floating-point — use decimal.js at runtime
  currency: CurrencyCode; // ISO 4217
}

export type CurrencyCode = string & { readonly __brand: 'ISO4217' };

export function money(amount: string, currency: string): Money {
  return { amount, currency: currency as CurrencyCode };
}
```

**Testing:**
- `turbo build` completes without errors across all workspaces
- `turbo lint` passes with zero warnings
- TypeScript strict mode enabled in all packages (`"strict": true`)
- `packages/shared` types are importable from `apps/api`
- Docker Compose starts all services (API, PostgreSQL, Redis) with `docker compose up`

### Task 1.2: Database Schema & Migrations

**What:** Create the initial PostgreSQL schema using the Hybrid Relational + JSONB model (Model 3), with the `expense_codings` table from Model 1 and the `events` table from Model 2. Set up migration tooling, seed reference data (ISO 4217 currencies, ISO 18245 MCC codes, ISO 3166-1 countries), and configure RLS.

**Design:**

```sql
-- packages/db/migrations/001_foundation.sql

-- Reference data tables
CREATE TABLE currencies (
    code            CHAR(3) PRIMARY KEY,
    numeric_code    CHAR(3),
    name            TEXT NOT NULL,
    minor_units     INT NOT NULL DEFAULT 2,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE merchant_category_codes (
    code            CHAR(4) PRIMARY KEY,
    description     TEXT NOT NULL,
    category_group  TEXT
);

-- Core tables (from Model 3)
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
    jurisdiction_config JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    employee_id     TEXT,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','inactive','terminated')),
    profile         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);

-- RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON users
    USING (organization_id = current_setting('app.current_org_id')::UUID);

-- Event store (from Model 2 — append-only audit trail)
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  TEXT NOT NULL,
    aggregate_id    UUID NOT NULL,
    organization_id UUID NOT NULL,
    event_type      TEXT NOT NULL,
    event_version   INT NOT NULL DEFAULT 1,
    sequence_number BIGINT NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    correlation_id  UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(aggregate_type, aggregate_id, sequence_number)
);
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_number);
CREATE INDEX idx_events_org ON events(organization_id, created_at);
```

```typescript
// packages/db/src/migrate.ts
import { Pool } from 'pg';
import { Umzug, SequelizeStorage } from 'umzug';
// Migration runner using Umzug with raw SQL files
```

**Testing:**
- Migration runs cleanly on a fresh PostgreSQL 16 database: `npm run db:migrate` exits 0
- Migration is idempotent: running twice produces no errors
- Rollback migration drops all tables cleanly
- Seed data: verify `SELECT COUNT(*) FROM currencies` returns 180+ rows (active ISO 4217 codes)
- Seed data: verify `SELECT COUNT(*) FROM merchant_category_codes` returns 400+ rows
- RLS test: set `app.current_org_id` to org A, insert user for org B, verify `SELECT` on users returns zero rows from org B
- UUID generation: insert 10,000 rows, verify zero collisions
- CHECK constraints: attempt to insert `organizations.status = 'invalid'`, verify constraint violation error

### Task 1.3: API Skeleton with Authentication

**What:** Stand up the Express/Fastify API server with health check, OpenTelemetry instrumentation, request logging, error handling middleware, and OIDC/SAML authentication using Passport.js or a Keycloak adapter. Implement the organization and user CRUD endpoints.

**Design:**

```typescript
// apps/api/src/server.ts
import Fastify from 'fastify';
import { initTelemetry } from './middleware/telemetry';
import { authPlugin } from './middleware/auth';
import { rlsPlugin } from './middleware/rls';
import { orgRoutes } from './modules/organizations/routes';
import { userRoutes } from './modules/users/routes';

const app = Fastify({ logger: true });

// Middleware
app.register(authPlugin);
app.register(rlsPlugin);

// Routes
app.register(orgRoutes, { prefix: '/api/v1/organizations' });
app.register(userRoutes, { prefix: '/api/v1/users' });

// Health check
app.get('/health', async () => ({ status: 'ok', version: process.env.APP_VERSION }));

// apps/api/src/middleware/rls.ts
// Sets PostgreSQL session variable for RLS on every request
export async function rlsPlugin(app: FastifyInstance) {
  app.addHook('preHandler', async (request) => {
    const orgId = request.user?.organizationId;
    if (orgId) {
      await request.dbPool.query(
        `SET LOCAL app.current_org_id = '${orgId}'`
      );
    }
  });
}
```

```yaml
# openapi.yaml (partial)
openapi: "3.1.0"
info:
  title: Corporate Card & Spend Management API
  version: "0.1.0"
paths:
  /api/v1/organizations:
    get:
      summary: List organizations
      security: [{ bearerAuth: [] }]
      responses:
        "200":
          description: List of organizations
  /api/v1/users:
    get:
      summary: List users in the current organization
      security: [{ bearerAuth: [] }]
```

**Testing:**
- `GET /health` returns `200 { status: "ok" }`
- Unauthenticated `GET /api/v1/organizations` returns `401`
- Authenticated `POST /api/v1/organizations` creates an organization and returns `201` with UUID
- Authenticated `GET /api/v1/users` returns only users in the caller's organization (RLS verified)
- OIDC login flow: obtain token from Keycloak, use it to call API, verify `200`
- Request with malformed JWT returns `401` with structured error body
- OpenTelemetry: verify trace IDs appear in response headers and in Grafana/Jaeger
- Rate limiting: send 1000 requests in 1 second, verify `429` responses after threshold
- API response conforms to OpenAPI spec (validated by `openapi-validator` middleware)

### Task 1.4: CI/CD Pipeline

**What:** Configure GitHub Actions for lint, typecheck, unit tests, integration tests (against a PostgreSQL container), and Docker image builds. Add branch protection rules.

**Design:**

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx turbo lint
      - run: npx turbo typecheck

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: spend_test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
      redis:
        image: redis:7
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx turbo test
```

**Testing:**
- Push to any branch triggers CI
- CI passes on a clean clone (no local state dependencies)
- CI fails if any lint error, type error, or test failure exists
- Docker image builds successfully: `docker build -t spend-api apps/api`
- Docker image starts and passes health check

### Task 1.5: Web Dashboard Skeleton

**What:** Initialize the Next.js 15 web app with App Router, Tailwind CSS, shadcn/ui component library, authentication redirect flow, and a shell layout (sidebar navigation, top bar with user menu).

**Design:**

```typescript
// apps/web/src/app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <AuthProvider>
          <div className="flex h-screen">
            <Sidebar />
            <main className="flex-1 overflow-auto">{children}</main>
          </div>
        </AuthProvider>
      </body>
    </html>
  );
}

// apps/web/src/app/(dashboard)/page.tsx
export default function DashboardPage() {
  return (
    <div className="p-6">
      <h1 className="text-2xl font-semibold">Dashboard</h1>
      {/* Placeholder for spend overview widgets */}
    </div>
  );
}
```

**Testing:**
- `npm run dev` starts the web app on port 3000
- Unauthenticated users are redirected to the login page
- After OIDC login, the dashboard shell renders with sidebar and top bar
- Sidebar navigation items are present: Dashboard, Cards, Transactions, Budgets, Policies, Settings
- Lighthouse accessibility score >= 90
- `npm run build` produces a production build without errors

### Definition of Done -- Phase 1

- [ ] Monorepo builds, lints, and type-checks with zero errors
- [ ] PostgreSQL schema deployed via migrations; RLS verified
- [ ] API serves authenticated CRUD for organizations and users
- [ ] Event store records all create/update operations
- [ ] CI pipeline passes on GitHub Actions
- [ ] Docker Compose starts the full stack locally
- [ ] Web dashboard shell loads after authentication
- [ ] OpenAPI 3.1 spec documents all Phase 1 endpoints
- [ ] All seed reference data loaded (currencies, MCC codes)

---

## Phase 2: Card Programme Integration

**Goal:** Integrate with Stripe Issuing (primary) and build the card issuer adapter abstraction so Lithic and Marqeta can be added later. Enable virtual and physical card creation, spend limit configuration, card lifecycle management, and real-time transaction ingestion via webhooks.

**Duration:** 3-4 weeks

**Depends on:** Phase 1

### Task 2.1: Card Issuer Adapter Layer

**What:** Build an abstraction layer (`packages/card-issuer-adapters`) that normalizes the Stripe Issuing, Lithic, and Marqeta APIs into a unified interface. Implement the Stripe adapter first; Lithic and Marqeta are stub implementations.

**Design:**

```typescript
// packages/card-issuer-adapters/src/types.ts
export interface CardIssuerAdapter {
  createCard(params: CreateCardParams): Promise<IssuerCard>;
  getCard(issuerCardId: string): Promise<IssuerCard>;
  updateCard(issuerCardId: string, params: UpdateCardParams): Promise<IssuerCard>;
  freezeCard(issuerCardId: string): Promise<void>;
  cancelCard(issuerCardId: string): Promise<void>;
  listCards(params: ListCardsParams): Promise<PaginatedResult<IssuerCard>>;
  parseWebhookEvent(payload: string, signature: string): Promise<CardWebhookEvent>;
}

export interface CreateCardParams {
  cardholderId: string;
  type: 'physical' | 'virtual';
  currency: CurrencyCode;
  spendLimit?: { amount: string; interval: SpendLimitInterval };
  metadata?: Record<string, string>;
}

export interface CardWebhookEvent {
  eventType: 'authorization' | 'clearing' | 'reversal' | 'decline';
  transaction: NormalizedTransaction;
  rawPayload: unknown;
}

// packages/card-issuer-adapters/src/stripe/adapter.ts
import Stripe from 'stripe';

export class StripeIssuingAdapter implements CardIssuerAdapter {
  private stripe: Stripe;

  constructor(apiKey: string) {
    this.stripe = new Stripe(apiKey);
  }

  async createCard(params: CreateCardParams): Promise<IssuerCard> {
    const card = await this.stripe.issuing.cards.create({
      cardholder: params.cardholderId,
      type: params.type,
      currency: params.currency,
      spending_controls: params.spendLimit ? {
        spending_limits: [{
          amount: Math.round(parseFloat(params.spendLimit.amount) * 100),
          interval: params.spendLimit.interval,
        }],
      } : undefined,
    });
    return this.normalizeCard(card);
  }
  // ...
}
```

**Testing:**
- Unit tests with mocked Stripe SDK: `createCard` returns normalized `IssuerCard` object
- `parseWebhookEvent` correctly verifies Stripe webhook signatures (use Stripe test fixtures)
- Adapter correctly converts Stripe cents to decimal amounts (e.g., `12550` -> `"125.50"`)
- Lithic and Marqeta stubs throw `NotImplementedError` with descriptive messages
- Factory function `createAdapter('stripe_issuing', config)` returns correct adapter instance
- Factory function `createAdapter('lithic', config)` returns stub with clear error

### Task 2.2: Card Management API Endpoints

**What:** Build CRUD endpoints for card programmes and cards. Cards are created via the issuer adapter and stored locally with tokenized references (never store PAN). Implement card lifecycle operations: create, freeze, unfreeze, cancel.

**Design:**

```typescript
// apps/api/src/modules/cards/routes.ts
app.post('/api/v1/cards', async (req, reply) => {
  // Validate: cardholder exists, org has card programme, user has card_admin role
  // Call issuer adapter to create card
  // Store card record locally (card_token, issuer_card_id, last_four — never PAN)
  // Emit CardCreated event to event store
  // Return card (without sensitive data)
});

app.patch('/api/v1/cards/:id/freeze', async (req, reply) => {
  // Call issuer adapter to freeze
  // Update local card status
  // Emit CardFrozen event
});

app.get('/api/v1/cards', async (req, reply) => {
  // List cards for current org (RLS enforced)
  // Filter by cardholder, status, type
  // Paginated response
});
```

**Testing:**
- `POST /api/v1/cards` with valid params creates a card and returns `201` with `id`, `last_four`, `status`, `card_type` (no PAN in response)
- Card record in database has `card_token` and `issuer_card_id` but no PAN column exists
- `PATCH /api/v1/cards/:id/freeze` changes status to `frozen` and calls issuer freeze API
- `PATCH /api/v1/cards/:id/unfreeze` on a frozen card restores `active` status
- `DELETE /api/v1/cards/:id` cancels the card with the issuer and sets local status to `cancelled`
- `GET /api/v1/cards` returns only cards belonging to the caller's organization
- Creating a card for a user in a different organization returns `403`
- Event store contains `CardCreated`, `CardFrozen`, `CardCancelled` events with correct aggregate IDs
- Spend limit amounts stored correctly as `NUMERIC(15,2)` without floating-point drift

### Task 2.3: Transaction Ingestion via Webhooks

**What:** Build a webhook endpoint that receives real-time card authorization, clearing, and reversal events from card issuers. Normalize the events through the adapter layer, create transaction records, and emit domain events. Implement webhook signature verification and idempotency.

**Design:**

```typescript
// apps/api/src/modules/transactions/webhook-handler.ts
app.post('/api/v1/webhooks/card-issuer/:issuer', async (req, reply) => {
  const adapter = getAdapter(req.params.issuer);

  // Verify webhook signature (PCI DSS requirement)
  const event = await adapter.parseWebhookEvent(
    req.rawBody,
    req.headers['stripe-signature'] ?? req.headers['x-lithic-signature']
  );

  // Idempotency: check if issuer_transaction_id already exists
  const existing = await findTransactionByIssuerTxnId(event.transaction.issuerTransactionId);
  if (existing) {
    return reply.status(200).send({ status: 'duplicate' });
  }

  // Create or update transaction
  // For authorization: create new transaction with status 'pending'
  // For clearing: update existing transaction to status 'cleared'
  // For reversal: update to 'reversed'
  // Merchant data enrichment: look up merchant by name, enrich with clean name and logo
  // Emit TransactionAuthorized / TransactionCleared / TransactionReversed event
  return reply.status(200).send({ status: 'processed' });
});
```

**Testing:**
- Webhook with valid Stripe signature creates a transaction record
- Webhook with invalid signature returns `400` and creates no record
- Duplicate webhook (same `issuer_transaction_id`) returns `200` without creating a duplicate
- Authorization webhook creates transaction with `status: 'pending'`
- Clearing webhook updates the same transaction to `status: 'cleared'` and sets `cleared_at`
- Reversal webhook updates to `status: 'reversed'`
- Multi-currency transaction: `amount: 100.00 EUR`, `billing_amount: 108.50 USD`, `fx_rate: 1.085`
- MCC code from webhook is stored in `mcc_code` column
- Merchant name enrichment: raw `"AMZN MKTP US"` is enriched to `clean_name: "Amazon"` in `merchant_data` JSONB
- Event store contains `TransactionAuthorized` event with full payload
- Webhook processing completes in < 200ms (p99) to avoid issuer timeouts

### Task 2.4: Transaction List & Detail UI

**What:** Build the transaction list page in the web dashboard with filtering (date range, card, cardholder, status, MCC category), sorting, and a transaction detail drawer showing full transaction data, merchant info, and event timeline.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/transactions/page.tsx
export default async function TransactionsPage({ searchParams }) {
  // Server Component: fetch transactions with filters
  // Render TanStack Table with columns:
  // Date | Merchant | Card (*1234) | Amount | Status | Receipt | GL Code
  return (
    <div className="p-6">
      <TransactionFilters />
      <TransactionTable data={transactions} />
      <TransactionDetailDrawer />
    </div>
  );
}
```

**Testing:**
- Transaction list loads with 100 transactions in < 500ms
- Date range filter correctly narrows results
- Status filter (pending/cleared/declined/reversed) works
- MCC category grouping filter works (e.g., "Travel", "Software")
- Clicking a transaction opens the detail drawer with merchant data, amounts, and event timeline
- Pagination: page 2 shows the next 25 transactions
- Empty state: "No transactions found" with clear message when filters match nothing
- Multi-currency transactions display both transaction currency and billing currency

### Definition of Done -- Phase 2

- [ ] Card issuer adapter abstraction layer with working Stripe Issuing implementation
- [ ] Card CRUD API: create, list, get, freeze, unfreeze, cancel
- [ ] Webhook endpoint ingests transactions in real-time with signature verification
- [ ] Transaction records stored with correct amounts, currencies, merchant data
- [ ] Idempotent webhook processing prevents duplicates
- [ ] Transaction list page with filtering, sorting, and detail drawer
- [ ] All card and transaction events written to event store
- [ ] No PAN or CVV data stored anywhere in the system

---

## Phase 3: Policy Engine & Budget Controls

**Goal:** Implement the configurable policy engine for spend limits, MCC category restrictions, and receipt requirements. Build budget management with department/project-level tracking and real-time utilization alerts.

**Duration:** 3-4 weeks

**Depends on:** Phase 2

### Task 3.1: Policy Engine Core

**What:** Build the policy evaluation engine that evaluates every transaction against the organization's active policies. Policies are stored as JSONB rule sets (Model 3 design). The engine supports conditions on amount, MCC code, merchant, time of day, and cardholder role, with actions: block, flag, require_approval, require_receipt, warn.

**Design:**

```typescript
// apps/api/src/modules/policies/engine.ts
export interface PolicyRule {
  conditions: PolicyCondition[];
  conditionLogic: 'all' | 'any';
  action: 'block' | 'flag' | 'require_approval' | 'require_receipt' | 'warn';
  actionConfig?: Record<string, unknown>;
}

export interface PolicyCondition {
  field: 'amount' | 'mcc_code' | 'merchant_name' | 'cardholder_role' |
         'time_of_day' | 'day_of_week' | 'country';
  operator: 'equals' | 'not_equals' | 'greater_than' | 'less_than' |
            'in' | 'not_in' | 'between' | 'contains';
  value: unknown;
}

export class PolicyEngine {
  async evaluate(
    transaction: Transaction,
    policies: Policy[],
    context: PolicyContext
  ): Promise<PolicyEvaluationResult> {
    const violations: PolicyViolation[] = [];

    // Sort policies by priority (lower number = higher priority)
    const sorted = policies.sort((a, b) => a.priority - b.priority);

    for (const policy of sorted) {
      if (!this.isInScope(policy.scope, context)) continue;
      const result = this.evaluateRules(policy.rules, transaction, context);
      if (result.violated) {
        violations.push({
          policyId: policy.id,
          policyName: policy.name,
          severity: result.severity,
          action: result.action,
          description: result.description,
        });
        if (result.action === 'block') break; // stop on blocking policy
      }
    }

    return { violations, compliant: violations.length === 0 };
  }
}
```

**Testing:**
- Policy with `amount > 500` flags a $600 transaction
- Policy with `mcc_code IN ['7995']` (gambling) blocks a gambling transaction
- Policy with `amount > 25` requires receipt: transaction without receipt gets `require_receipt` action
- Policy scope: policy scoped to Engineering department does not evaluate for Marketing users
- Policy priority: higher-priority block policy prevents evaluation of lower-priority warn policies
- `conditionLogic: 'all'` requires ALL conditions to match; `'any'` requires at least one
- Evaluation of 50 policies against one transaction completes in < 10ms
- Policy violations are stored in the transaction's `policy_results` JSONB column
- PolicyViolationDetected event emitted to event store

### Task 3.2: Policy Management API & UI

**What:** CRUD endpoints and web UI for creating, editing, and managing spend policies. Finance managers can define rules with a visual builder (condition + operator + value), set scope (all employees, specific departments, specific roles), and set enforcement actions.

**Design:**

```typescript
// apps/api/src/modules/policies/routes.ts
app.post('/api/v1/policies', requireRole('finance_manager'));
app.get('/api/v1/policies', requireRole('finance_manager'));
app.put('/api/v1/policies/:id', requireRole('finance_manager'));
app.delete('/api/v1/policies/:id', requireRole('finance_manager'));
app.post('/api/v1/policies/:id/test', requireRole('finance_manager'));
// Test endpoint: dry-run a policy against sample transactions
```

**Testing:**
- Only users with `finance_manager` role can create policies
- Created policy is immediately active and evaluates against new transactions
- Policy test endpoint: submit a sample transaction and get back what violations would trigger
- Editing a policy updates the JSONB rules and immediately affects new evaluations
- Deleting a policy (soft delete: `is_active = false`) stops it from evaluating
- UI: policy builder renders condition rows with field/operator/value dropdowns
- UI: scope selector shows departments, roles, and entities as checkboxes
- Audit: PolicyCreated, PolicyUpdated events recorded in event store

### Task 3.3: Budget Management Core

**What:** Build budget CRUD with real-time utilization tracking. When a transaction is processed (Task 2.3), update the associated budget's `spent_amount` and `committed_amount`. Implement budget period management and alert threshold checking.

**Design:**

```typescript
// apps/api/src/modules/budgets/service.ts
export class BudgetService {
  async recordSpend(transactionId: UUID, budgetId: UUID, amount: Money): Promise<void> {
    // Atomic update: UPDATE budgets SET spent_amount = spent_amount + $1
    // Check alert thresholds
    const budget = await this.getBudget(budgetId);
    const utilization = (budget.spentAmount + parseFloat(amount.amount))
                        / budget.limitAmount * 100;

    if (utilization >= budget.config.alertThresholds[0].pct) {
      await this.triggerAlert(budget, utilization);
    }

    // For pre-funded budgets: check available funds before spend
    if (budget.config.isPreFunded) {
      const available = budget.config.fundedAmount - budget.spentAmount;
      if (parseFloat(amount.amount) > available) {
        throw new InsufficientBudgetError(budgetId, amount, available);
      }
    }

    // Emit BudgetSpendRecorded event
  }
}
```

**Testing:**
- Creating a budget with `limit_amount: 10000, period: 'monthly'` stores correctly
- Transaction of $500 against a budget updates `spent_amount` from `0` to `500`
- Budget utilization at 80% triggers an alert (configurable threshold)
- Budget utilization at 100% triggers a second alert with higher severity
- Pre-funded budget: transaction exceeding available funds is rejected
- Budget period rollover: monthly budget resets `spent_amount` to 0 at period boundary
- Concurrent transactions: two simultaneous $500 spends against a $900 budget -- one succeeds, one is rejected (serializable isolation test)
- `GET /api/v1/budgets/:id` returns current utilization percentage and remaining amount
- Budget list shows all budgets for the organization with utilization bars
- BudgetCreated, BudgetSpendRecorded events in event store

### Task 3.4: Budget Management UI

**What:** Web dashboard pages for creating and managing budgets. Budget list view with utilization progress bars, budget detail view with spend-over-time chart, and budget creation form.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/budgets/page.tsx
// List view: cards showing each budget with name, owner, limit, spent, utilization bar
// Color coding: green (< 75%), yellow (75-90%), red (> 90%)

// apps/web/src/app/(dashboard)/budgets/[id]/page.tsx
// Detail view:
//   - Utilization donut chart
//   - Spend-over-time line chart (daily/weekly/monthly)
//   - Recent transactions against this budget
//   - Alert configuration
```

**Testing:**
- Budget list shows all budgets with correct utilization percentages
- Utilization bar color changes at threshold boundaries (green/yellow/red)
- Budget detail page shows spend-over-time chart with correct data points
- Creating a budget via the UI form creates it via the API and shows it in the list
- Editing a budget's limit amount updates the utilization percentage immediately

### Definition of Done -- Phase 3

- [ ] Policy engine evaluates every transaction against active policies
- [ ] Policy CRUD API and visual builder UI for finance managers
- [ ] Budget CRUD with real-time utilization tracking
- [ ] Budget alerts fire at configurable thresholds
- [ ] Pre-funded budget model prevents overspend
- [ ] Budget UI with utilization bars and spend charts
- [ ] All policy and budget events recorded in event store

---

## Phase 4: Receipt Processing & GL Coding

**Goal:** Enable receipt capture (upload, email, Slack), OCR extraction, and AI-powered GL auto-coding against the organization's chart of accounts.

**Duration:** 3-4 weeks

**Depends on:** Phase 2 (can run in parallel with Phase 3)

### Task 4.1: Receipt Upload & Storage

**What:** Build receipt upload endpoints supporting image (JPEG, PNG, HEIC) and PDF files. Store files in S3-compatible storage. Support upload via API, email forwarding (inbound email webhook), and Slack bot command.

**Design:**

```typescript
// apps/api/src/modules/receipts/routes.ts
app.post('/api/v1/receipts', async (req, reply) => {
  // Accept multipart file upload
  // Validate file type (image/jpeg, image/png, image/heic, application/pdf)
  // Validate file size (max 10MB)
  // Upload to S3 with organization-scoped key prefix
  // Create receipt record with source='upload'
  // Queue OCR processing job
  // Return receipt ID and upload status
});

app.post('/api/v1/receipts/email-webhook', async (req, reply) => {
  // Parse inbound email (SendGrid/Mailgun webhook)
  // Extract attachments
  // Match sender email to user
  // Create receipt with source='email'
  // Queue OCR processing
});

// Link receipt to transaction
app.post('/api/v1/transactions/:txnId/receipts', async (req, reply) => {
  // Attach existing receipt to a transaction
  // Update transaction: has_receipt = true in UI
});
```

**Testing:**
- Upload a JPEG receipt: returns `201` with receipt ID and `ocr_status: 'pending'`
- Upload a 15MB file: returns `413` with clear error message
- Upload an `.exe` file: returns `400` (invalid file type)
- Receipt stored in S3 at `/{org_id}/receipts/{receipt_id}.jpg`
- Email webhook: email from `alice@company.com` creates receipt linked to Alice's user record
- Email from unknown sender: receipt created but flagged for manual user assignment
- Linking receipt to transaction: `POST /api/v1/transactions/:id/receipts` updates the transaction record
- ReceiptUploaded event emitted with file metadata

### Task 4.2: OCR Pipeline

**What:** Build the OCR processing service (Python) that extracts vendor name, amount, currency, date, and line items from receipt images. Use Tesseract or PaddleOCR with a structured extraction layer. Results are stored in the receipt's `ocr_data` JSONB column.

**Design:**

```python
# packages/ai/ocr/service.py
from fastapi import FastAPI
from paddleocr import PaddleOCR

app = FastAPI()
ocr_engine = PaddleOCR(use_angle_cls=True, lang='en')

@app.post("/extract")
async def extract_receipt(file_url: str) -> ReceiptExtractionResult:
    # Download image from S3
    # Run OCR to get raw text
    # Apply structured extraction rules:
    #   - Amount: look for total/grand total patterns
    #   - Date: parse date patterns (MM/DD/YYYY, etc.)
    #   - Vendor: extract from header/top of receipt
    #   - Line items: extract individual items if present
    # Return structured result with confidence score
    return ReceiptExtractionResult(
        vendor_name="Delta Air Lines",
        amount=Decimal("450.00"),
        currency="USD",
        date=date(2026, 5, 10),
        confidence=0.94,
        raw_text=raw_text,
    )
```

**Testing:**
- Submit a clear receipt image: OCR extracts vendor name, amount, and date correctly
- Submit a blurry receipt: OCR returns lower confidence score (< 0.7) and flags for manual review
- Submit a PDF receipt (e.g., airline itinerary): multi-page extraction works
- Amount extraction handles formats: `$1,234.56`, `1.234,56 EUR`, `GBP 500.00`
- Date extraction handles formats: `05/10/2026`, `May 10, 2026`, `10-05-2026`
- OCR results stored in `receipts.ocr_data` JSONB column
- Processing completes in < 5 seconds for a typical receipt image
- Failed OCR (corrupt file) sets `ocr_status: 'failed'` and logs error

### Task 4.3: Chart of Accounts Management

**What:** Build CRUD for the organization's chart of accounts (GL accounts). Support hierarchical account structures (parent/child), account types (asset, liability, equity, revenue, expense), and import from QuickBooks/Xero format.

**Design:**

```typescript
// apps/api/src/modules/gl/routes.ts
app.get('/api/v1/gl-accounts', requireRole('finance_manager'));
app.post('/api/v1/gl-accounts', requireRole('finance_manager'));
app.put('/api/v1/gl-accounts/:id', requireRole('finance_manager'));
app.post('/api/v1/gl-accounts/import', requireRole('finance_manager'));
// Import accepts CSV with columns: code, name, type, parent_code
```

**Testing:**
- Create GL account `6200 - Software & SaaS` of type `expense`: returns `201`
- Create child account `6210 - Cloud Infrastructure` with `parent_id` referencing `6200`
- Hierarchical query: `GET /api/v1/gl-accounts?tree=true` returns nested structure
- Import CSV with 50 accounts: all created correctly with parent references resolved
- Duplicate account code within same org: returns `409 Conflict`
- Deactivating an account (`is_active: false`) prevents it from being used in new codings

### Task 4.4: AI GL Auto-Coding

**What:** Build the GL auto-coding service that assigns GL account, department, and cost center to each transaction based on the organization's historical coding patterns and configurable rules. Implement rule-based coding (MCC-to-GL mappings) as the baseline, with ML-based coding as an enhancement.

**Design:**

```typescript
// apps/api/src/modules/gl/coding-service.ts
export class GLCodingService {
  async codeTransaction(transaction: Transaction): Promise<CodingResult> {
    // 1. Try rule-based coding first (coding_rules table / policy rules JSONB)
    const ruleResult = await this.applyRules(transaction);
    if (ruleResult) {
      return { ...ruleResult, method: 'rule_based', confidence: 1.0 };
    }

    // 2. Try ML-based coding
    const mlResult = await this.mlCoder.predict({
      merchantName: transaction.merchantName,
      mccCode: transaction.mccCode,
      amount: transaction.amount,
      cardholderId: transaction.cardholderId,
      orgId: transaction.organizationId,
    });

    if (mlResult.confidence >= 0.85) {
      return { ...mlResult, method: 'ai_auto' };
    }

    // 3. Return suggestion for human review
    return { ...mlResult, method: 'ai_suggested' };
  }
}
```

**Testing:**
- Rule-based: MCC `5812` (restaurants) maps to GL `6300 - Meals & Entertainment` with confidence 1.0
- Rule-based: vendor "AWS" maps to GL `6200 - Software` with department from cardholder's profile
- ML-based: after 100 coded transactions, ML model suggests correct GL for a new AWS transaction with confidence > 0.85
- Low-confidence coding (< 0.85): transaction marked as `coding_method: 'ai_suggested'` requiring human review
- Split coding: $1000 transaction split into $700 GL 6200 (Compute) and $300 GL 6210 (Storage)
- Coding stored in `expense_codings` table (from Model 1) with `gl_account_id`, `department_id`, `coding_method`, `ai_confidence`
- TransactionCoded event emitted with coding details
- Coding rules CRUD: finance manager creates rule "Uber -> GL 6400 Travel" and it applies to next Uber transaction

### Definition of Done -- Phase 4

- [ ] Receipt upload via API, email, and Slack (stub) works
- [ ] OCR pipeline extracts vendor, amount, date from receipts with > 80% accuracy on clear images
- [ ] Chart of accounts CRUD with hierarchical structure
- [ ] GL auto-coding assigns account, department, cost center to transactions
- [ ] Rule-based coding handles known merchant/MCC mappings
- [ ] ML coding suggests GL accounts for unknown transactions
- [ ] Split coding supported for multi-category expenses
- [ ] Coding provenance tracked: manual vs. ai_auto vs. ai_suggested vs. rule_based

---

## Phase 5: Approval Workflows & Reimbursements

**Goal:** Implement configurable approval workflows for expenses, card requests, and budget requests. Build the employee reimbursement submission and payment workflow.

**Duration:** 3 weeks

**Depends on:** Phases 3 and 4

### Task 5.1: Approval Workflow Engine

**What:** Build a configurable approval workflow engine that routes requests through multi-step approval chains. Support approver types: direct manager, budget owner, specific role, specific user. Implement auto-approval below configurable thresholds and escalation after timeout.

**Design:**

```typescript
// apps/api/src/modules/approvals/engine.ts
export class ApprovalEngine {
  async createRequest(params: {
    type: 'expense' | 'reimbursement' | 'card_request' | 'budget_request';
    referenceId: UUID;
    requesterId: UUID;
    amount: Money;
  }): Promise<ApprovalRequest> {
    // Find matching workflow based on type and amount
    const workflow = await this.findWorkflow(params.type, params.amount);

    // Check auto-approval threshold
    if (workflow.steps[0].autoApproveBelow &&
        parseFloat(params.amount.amount) < workflow.steps[0].autoApproveBelow) {
      return this.autoApprove(params, workflow);
    }

    // Resolve first approver (direct_manager -> look up manager from user profile)
    const approver = await this.resolveApprover(workflow.steps[0], params.requesterId);

    // Create approval request at step 1
    // Send notification to approver
    // Start escalation timer
  }

  async decide(requestId: UUID, approverId: UUID, decision: 'approved' | 'rejected',
               comment?: string): Promise<void> {
    // Record decision
    // If approved and more steps: advance to next step, notify next approver
    // If approved and last step: mark request as approved, trigger downstream action
    // If rejected: mark request as rejected, notify requester
  }
}
```

**Testing:**
- Expense over $500 triggers 2-step approval (manager then finance)
- Expense under $50 auto-approves (configurable threshold)
- Manager approves step 1: request advances to step 2 (finance)
- Finance approves step 2: request marked approved, requester notified
- Manager rejects: request marked rejected, no step 2 needed
- Escalation: request unapproved after 48 hours escalates to next-level manager
- Approver cannot approve their own request
- Delegation: approver delegates to another user, delegatee can approve
- Approval decisions recorded in event store

### Task 5.2: Reimbursement Workflow

**What:** Build the end-to-end employee reimbursement flow: draft creation, receipt attachment, GL coding, submission, approval routing, and payment status tracking. Payment initiation (ACH) is a stub in this phase.

**Design:**

```typescript
// apps/api/src/modules/reimbursements/routes.ts
app.post('/api/v1/reimbursements', requireRole('employee'));
// Create draft reimbursement

app.put('/api/v1/reimbursements/:id', requireRole('employee'));
// Update draft (add receipts, edit amount, add memo)

app.post('/api/v1/reimbursements/:id/submit', requireRole('employee'));
// Submit for approval: validates receipts attached, GL coding present
// Triggers approval workflow

app.post('/api/v1/reimbursements/:id/approve', requireRole('finance_manager'));
// Approve reimbursement (also accessible via approval workflow engine)

app.post('/api/v1/reimbursements/:id/pay', requireRole('finance_manager'));
// Initiate payment (ACH stub in this phase)
```

**Testing:**
- Employee creates draft reimbursement with amount, date, merchant name: returns `201`
- Attaching a receipt to the reimbursement updates the record
- Submitting without a receipt (when policy requires receipt > $25): returns `400` with policy violation
- Submitting with receipt: triggers approval workflow, status changes to `submitted`
- Manager approves: status changes to `approved`
- Finance initiates payment: status changes to `processing_payment`
- Payment completes (stub): status changes to `paid`, `paid_at` timestamp set
- Full lifecycle events: ReimbursementCreated, ReimbursementSubmitted, ReimbursementApproved, ReimbursementPaid
- List view: employees see their own reimbursements; managers see their team's

### Task 5.3: Approval & Reimbursement UI

**What:** Build the approval queue UI (pending approvals for the logged-in user), reimbursement submission form (with receipt upload and GL coding), and reimbursement list view.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/approvals/page.tsx
// Approval queue: list of pending requests requiring action
// Each item shows: type, requester, amount, date, policy context
// Actions: Approve / Reject with optional comment

// apps/web/src/app/(dashboard)/reimbursements/new/page.tsx
// Form: amount, date, merchant, category, memo
// Receipt upload dropzone
// GL coding (auto-suggested, editable)
// Submit button
```

**Testing:**
- Approval queue shows only items assigned to the current user
- Approving an item removes it from the queue and shows success toast
- Reimbursement form validates required fields before submission
- Receipt drag-and-drop upload works and shows OCR-extracted data
- GL coding auto-fills from AI suggestion; user can override

### Definition of Done -- Phase 5

- [ ] Multi-step approval workflows configurable by finance managers
- [ ] Auto-approval below configurable amount thresholds
- [ ] Escalation after configurable timeout period
- [ ] Employee reimbursement workflow: draft -> submit -> approve -> pay
- [ ] Receipt requirement enforcement during reimbursement submission
- [ ] Approval queue UI for managers and finance team
- [ ] Reimbursement submission form with receipt upload and GL coding
- [ ] All approval and reimbursement events in event store

---

## Phase 6: Accounts Payable Automation

**Goal:** Build vendor management, bill capture and processing, approval routing for bills, and payment scheduling. This is the AP automation layer that competes with Airbase and Ramp's AP features.

**Duration:** 3 weeks

**Depends on:** Phase 5 (can run in parallel with Phase 7)

### Task 6.1: Vendor Management

**What:** Build vendor CRUD with profile management (legal name, tax ID, payment terms, default GL account, default payment method). Include contract tracking with renewal reminders.

**Design:**

```typescript
// apps/api/src/modules/vendors/routes.ts
app.post('/api/v1/vendors', requireRole('finance_manager'));
app.get('/api/v1/vendors');
app.put('/api/v1/vendors/:id', requireRole('finance_manager'));

// Vendor profile stored in JSONB (Model 3 pattern)
// Contracts as JSONB array within profile
// Renewal alert: cron job checks cancellation_deadline - renewal_notice_days
```

**Testing:**
- Create vendor "AWS" with payment terms `net_30`, default GL `6200`: returns `201`
- Add contract with `auto_renews: true`, `renewal_notice_days: 60`: stored in profile JSONB
- Renewal alert: contract with cancellation deadline 45 days from now triggers a notification
- Vendor search: partial name match "Amaz" returns "Amazon Web Services"
- Deactivating a vendor prevents new bills from being created for that vendor
- W-9 status tracking in vendor profile

### Task 6.2: Bill Capture & Processing

**What:** Build bill/invoice creation from OCR capture (PDF upload), manual entry, or email forwarding. Bills have line items with GL coding, approval routing, and payment scheduling.

**Design:**

```typescript
// apps/api/src/modules/bills/routes.ts
app.post('/api/v1/bills', requireRole('finance_manager'));
app.post('/api/v1/bills/upload', requireRole('finance_manager'));
// Upload PDF invoice -> OCR extracts vendor, amount, due date, line items

app.post('/api/v1/bills/:id/submit', requireRole('finance_manager'));
// Submit for approval

app.post('/api/v1/bills/:id/schedule-payment', requireRole('finance_manager'));
// Schedule payment for a specific date via ACH/wire/virtual card
```

**Testing:**
- Upload PDF invoice: OCR extracts vendor name, invoice number, amount, due date
- Manual bill entry: create bill with vendor, amount, line items, GL coding
- Bill line items sum to bill total amount (validation)
- Submit for approval: triggers approval workflow based on bill amount
- Payment scheduling: bill with `due_date: 2026-06-15` scheduled for payment on 2026-06-14
- Overdue detection: bill past `due_date` with `status != 'paid'` marked as overdue
- Bill list filtered by status, vendor, due date range

### Task 6.3: AP Dashboard UI

**What:** Build the AP dashboard showing bills by status (pending, approved, scheduled, paid, overdue), vendor spend summary, and upcoming payment calendar.

**Testing:**
- AP dashboard shows bill counts by status
- Overdue bills highlighted in red
- Payment calendar shows scheduled payments by date
- Vendor spend summary: total spend YTD per vendor with drill-down

### Definition of Done -- Phase 6

- [ ] Vendor CRUD with profile, contracts, and renewal alerts
- [ ] Bill capture from PDF upload (OCR) and manual entry
- [ ] Bill approval workflow with GL-coded line items
- [ ] Payment scheduling (ACH stub)
- [ ] AP dashboard with status overview and payment calendar
- [ ] Overdue bill detection and alerting

---

## Phase 7: ERP Integrations

**Goal:** Build two-way sync with QuickBooks Online, Xero, and NetSuite. Transactions, GL accounts, vendors, and departments sync between the spend management platform and the customer's accounting system.

**Duration:** 4 weeks

**Depends on:** Phase 5 (can run in parallel with Phase 6)

### Task 7.1: ERP Adapter Layer

**What:** Build an ERP adapter abstraction (`packages/erp-adapters`) that normalizes QuickBooks, Xero, and NetSuite APIs. Implement OAuth 2.0 connection flow, field mapping configuration, and bidirectional sync logic.

**Design:**

```typescript
// packages/erp-adapters/src/types.ts
export interface ERPAdapter {
  connect(authCode: string): Promise<ConnectionResult>;
  disconnect(): Promise<void>;
  syncGLAccounts(): Promise<SyncResult>;
  syncVendors(): Promise<SyncResult>;
  syncDepartments(): Promise<SyncResult>;
  pushTransactions(transactions: Transaction[]): Promise<SyncResult>;
  pushBills(bills: Bill[]): Promise<SyncResult>;
  getFieldMappings(): Promise<FieldMapping[]>;
}

// packages/erp-adapters/src/quickbooks/adapter.ts
export class QuickBooksAdapter implements ERPAdapter {
  async pushTransactions(transactions: Transaction[]): Promise<SyncResult> {
    // Map internal transactions to QuickBooks Purchase objects
    // Use integration_mappings to resolve GL account IDs
    // Handle rate limits (500 req/min per QBO docs)
    // Track sync status in sync_log
  }
}
```

**Testing:**
- OAuth 2.0 flow with QuickBooks sandbox: connect, obtain tokens, store encrypted
- `syncGLAccounts()`: pulls QBO chart of accounts, creates/updates local GL accounts, stores mappings
- `pushTransactions()`: sends 10 coded transactions to QBO as Purchase records
- Field mapping: internal `department` maps to QBO `Class` (configurable)
- Rate limiting: 500+ requests queued and throttled without errors
- Token refresh: expired access token auto-refreshes using refresh token
- Sync log: each sync operation recorded with records_processed/created/updated/failed counts
- Disconnect: revokes OAuth tokens and marks integration as disconnected

### Task 7.2: QuickBooks Online Integration

**What:** Full implementation of the QuickBooks adapter with sync for GL accounts, vendors, departments (as Classes), and transaction push.

**Testing:**
- End-to-end: create expense in platform -> auto-coded to GL 6200 -> synced to QBO as Purchase -> appears in QBO register
- GL account sync: 50 QBO accounts pulled and matched to internal accounts
- New GL account in QBO: next sync creates it locally
- Vendor sync: QBO vendors matched by name/tax ID
- Conflict resolution: if both platforms modify the same GL account name, most-recent-wins

### Task 7.3: Xero & NetSuite Integration Stubs

**What:** Implement the Xero and NetSuite adapters with OAuth connection flows and basic GL account sync. Full sync deferred to a future iteration.

**Testing:**
- Xero OAuth connection with sandbox succeeds
- NetSuite TBA (Token-Based Authentication) connection succeeds
- GL account pull from both platforms returns correctly normalized data
- Push operations return `NotImplementedError` with clear message

### Task 7.4: Integration Management UI

**What:** Settings page for managing ERP connections. Shows connected integrations, sync status, last sync time, field mapping configuration, and manual sync trigger.

**Testing:**
- "Connect QuickBooks" button initiates OAuth flow and returns to settings page
- Connected integration shows green status indicator and last sync time
- Field mapping UI: dropdowns for mapping internal fields to ERP fields
- "Sync Now" button triggers manual sync and shows progress
- Sync errors displayed with actionable error messages
- "Disconnect" button revokes tokens and updates status

### Definition of Done -- Phase 7

- [ ] ERP adapter abstraction layer with QuickBooks implementation
- [ ] OAuth 2.0 connection flow for QBO, Xero (stub), NetSuite (stub)
- [ ] Two-way GL account sync with QuickBooks
- [ ] Transaction push from platform to QuickBooks
- [ ] Vendor sync between platform and QuickBooks
- [ ] Field mapping configuration UI
- [ ] Sync logging with success/failure tracking
- [ ] Integration management settings page

---

## Phase 8: Multi-Entity & Advanced Reporting

**Goal:** Support multi-entity organizations (PE portfolio companies, multi-subsidiary enterprises) with entity-level budget controls and consolidated reporting. Add advanced spend analytics.

**Duration:** 3 weeks

**Depends on:** Phases 6 and 7

### Task 8.1: Multi-Entity Support

**What:** Enable organizations to create multiple entities (subsidiaries, business units) with entity-scoped budgets, cards, and transactions. Users can be assigned to one or more entities. Reporting can be entity-specific or consolidated.

**Design:**

```typescript
// apps/api/src/modules/organizations/entities.ts
app.post('/api/v1/organizations/:orgId/entities', requireRole('org_admin'));
app.get('/api/v1/organizations/:orgId/entities');

// Entity hierarchy: entities can have parent entities
// Entity-scoped data: transactions, budgets, cards, bills can be scoped to an entity
// Consolidated view: aggregate across all entities for org-level reporting
// Entity-level currency: each entity can have its own base currency
```

**Testing:**
- Create entity "Acme US" (USD) and "Acme UK" (GBP) under organization "Acme Corp"
- Transaction for a card assigned to "Acme US" is scoped to that entity
- Budget for "Acme UK Engineering" only counts UK entity transactions
- Consolidated report: total spend across both entities, converted to org base currency
- User assigned to both entities can switch entity context in the UI
- Entity-level GL account mappings: US and UK entities can have different chart of accounts
- Hierarchical entities: "Acme Europe" parent of "Acme UK" and "Acme Germany" -- roll-up works

### Task 8.2: Spend Analytics Dashboard

**What:** Build an analytics dashboard with interactive charts: spend by department, spend by category (MCC), spend by vendor, spend over time, budget vs. actual comparison, and top spenders.

**Design:**

```typescript
// apps/api/src/modules/reports/routes.ts
app.get('/api/v1/reports/spend-by-department');
app.get('/api/v1/reports/spend-by-category');
app.get('/api/v1/reports/spend-by-vendor');
app.get('/api/v1/reports/spend-over-time');
app.get('/api/v1/reports/budget-vs-actual');
// All reports accept: date_range, entity_id (optional), department_id (optional)

// Pre-aggregated materialized views for performance
// CREATE MATERIALIZED VIEW mv_spend_by_department AS ...
// Refreshed on a schedule (every 15 minutes) or on-demand
```

**Testing:**
- Spend by department: bar chart shows correct totals, drill-down to individual transactions
- Spend by category: pie chart with MCC group labels (Travel, Software, Office, Dining)
- Spend over time: line chart with daily/weekly/monthly granularity toggle
- Budget vs. actual: side-by-side bars showing budget limit and actual spend per department
- Date range filter: last 7 days, last 30 days, this quarter, custom range
- Entity filter: selecting "Acme UK" shows only UK entity spend
- Export: CSV and PDF export of any report
- Report loads in < 2 seconds with 100,000 transactions in the database

### Task 8.3: Data Export & OFX Support

**What:** Build export functionality for transaction data in CSV, OFX (Open Financial Exchange), and PDF formats. OFX export enables import into accounting systems that lack API integrations.

**Testing:**
- CSV export: all transaction columns, amounts formatted with correct decimal places
- OFX export: valid `<STMTTRN>` elements parseable by QuickBooks Desktop and other OFX-compatible software
- PDF export: formatted report with company logo, date range, and summary totals
- Export with 10,000 transactions completes in < 10 seconds
- Filtered export: only exports transactions matching current filter criteria

### Definition of Done -- Phase 8

- [ ] Multi-entity support with entity-scoped budgets, cards, and transactions
- [ ] Entity hierarchy with parent-child relationships
- [ ] Consolidated reporting across entities with currency conversion
- [ ] Spend analytics dashboard with department, category, vendor, and time-series charts
- [ ] Budget vs. actual comparison report
- [ ] Data export in CSV, OFX, and PDF formats
- [ ] Reports perform within 2 seconds on 100K+ transaction datasets

---

## Phase 9: AI Policy Agent & Intelligence

**Goal:** Build the AI-native differentiation features: real-time AI policy enforcement at point of spend, proactive budget forecasting, and anomaly detection.

**Duration:** 4 weeks

**Depends on:** Phase 8

### Task 9.1: AI Policy Agent

**What:** Build an AI policy agent that reviews 100% of expenses in real time against the full context of employee role, project budget, prior spend history, and policy intent. Replace blunt category blocks with nuanced, context-aware approvals.

**Design:**

```python
# packages/ai/policy-agent/service.py
class PolicyAgent:
    def evaluate(self, transaction: dict, context: dict) -> PolicyDecision:
        """
        Context includes:
        - Employee role, department, manager chain
        - Budget utilization for associated budget
        - Employee's spend history (last 30/90 days)
        - Policy rules (structured)
        - Organization's historical patterns for similar transactions

        Decision output:
        - allow / flag / require_approval / block
        - confidence score
        - reasoning (human-readable explanation)
        - similar_transactions: past transactions that inform the decision
        """
        # Feature engineering
        features = self.extract_features(transaction, context)

        # ML model inference
        prediction = self.model.predict(features)

        # Explainability: SHAP values for top contributing features
        explanation = self.explain(features, prediction)

        return PolicyDecision(
            action=prediction.action,
            confidence=prediction.confidence,
            reasoning=explanation.summary,
            factors=explanation.top_factors,
        )
```

**Testing:**
- Standard expense ($50 lunch, regular employee, within budget): AI allows with high confidence
- Unusual expense ($500 dinner, junior employee, budget at 95%): AI flags for review with reasoning
- Known-good pattern: employee regularly purchases AWS at $X/month; similar purchase auto-approved
- Anomaly: employee who never travels submits $2000 hotel charge; AI flags as anomalous
- Reasoning output: human-readable explanation like "Flagged: amount exceeds cardholder's 90-day average by 3.2x; budget utilization is 94%"
- Latency: policy evaluation completes in < 100ms (p95) to not delay transaction processing
- AI confidence calibration: when AI says 90% confident, it is correct ~90% of the time

### Task 9.2: Budget Forecasting & Burn Alerts

**What:** Build a forecasting model that projects month-end and quarter-end spend by department and budget based on historical cadence, committed expenses, and known recurring charges. Alert finance teams to projected overruns 2-3 weeks before period close.

**Design:**

```python
# packages/ai/anomaly/forecasting.py
class BudgetForecaster:
    def forecast(self, budget_id: str, as_of_date: date) -> BudgetForecast:
        # Pull historical spend pattern for this budget (last 6 months)
        # Identify recurring charges (subscriptions, vendor payments)
        # Apply seasonal adjustment
        # Project remaining spend for current period
        # Compare to budget limit
        return BudgetForecast(
            budget_id=budget_id,
            period_end=period_end,
            projected_spend=projected,
            budget_limit=limit,
            projected_overrun=max(0, projected - limit),
            confidence_interval=(lower, upper),
            contributing_factors=[
                "AWS costs trending 15% above prior month",
                "3 recurring subscriptions due before period end",
            ],
        )
```

**Testing:**
- Budget with consistent $8K/month spend: forecast at mid-month projects $8K +/- 10%
- Budget with seasonal spike: December budget (holiday events) projects higher than January
- Recurring charges: subscription renewal on the 15th is included in forecast after the 1st
- Alert: budget projected to exceed limit by $2K, alert sent 3 weeks before period end
- Dashboard widget: forecast bars show projected spend alongside budget limit
- Forecast accuracy: within 15% of actual for 80% of budgets after 3 months of data

### Task 9.3: Spend Anomaly Detection

**What:** Build an anomaly detection service that flags statistical outliers in transaction patterns: unusually large transactions, unusual merchants for a cardholder, unusual times, and velocity anomalies (many transactions in a short period).

**Testing:**
- Single large transaction: $5000 when cardholder average is $200 -- flagged
- Unusual merchant: cardholder who only spends at software vendors makes a purchase at a jewelry store -- flagged
- Velocity: 10 transactions in 5 minutes from the same card -- flagged
- Geographic anomaly: transaction in a country the cardholder has never transacted in -- flagged
- False positive rate: < 5% of flagged transactions are legitimate

### Definition of Done -- Phase 9

- [ ] AI policy agent evaluates all transactions with context-aware decisions
- [ ] Human-readable reasoning for AI decisions
- [ ] Budget forecasting projects period-end spend with confidence intervals
- [ ] Proactive overrun alerts sent 2-3 weeks before period close
- [ ] Anomaly detection flags outlier transactions
- [ ] AI latency < 100ms (p95) for policy evaluation
- [ ] AI dashboard showing evaluation statistics and accuracy metrics

---

## Phase 10: Mobile App & Notifications

**Goal:** Build the React Native mobile app for receipt capture, real-time spend alerts, and approval actions on the go. Implement push notifications and Slack/Teams integration for alerts.

**Duration:** 3 weeks

**Depends on:** Phase 9

### Task 10.1: Mobile App Core

**What:** Build the React Native (Expo) mobile app with authentication (OIDC), transaction list, receipt camera capture, and approval actions.

**Design:**

```typescript
// apps/mobile/src/screens/CaptureReceipt.tsx
export function CaptureReceiptScreen() {
  // Camera interface with auto-crop and perspective correction
  // OCR preview: show extracted vendor, amount, date before submission
  // Link to transaction: select from recent unreceipted transactions
  // Submit: upload to API -> trigger OCR pipeline
}

// apps/mobile/src/screens/Approvals.tsx
export function ApprovalsScreen() {
  // List of pending approvals with swipe-to-approve/reject
  // Detail view with transaction context and policy evaluation
  // Push notification deep-links to specific approval
}
```

**Testing:**
- Camera capture: photo taken, cropped, and uploaded as receipt
- OCR preview: extracted data shown to user before submission
- Transaction list: scrollable list with pull-to-refresh
- Approval action: approve/reject with haptic feedback
- Offline receipt capture: photo stored locally and uploaded when connection restored
- Push notification: "New approval request: $450 from Alice" taps through to approval detail
- Authentication: biometric (Face ID / fingerprint) unlock after initial OIDC login

### Task 10.2: Push Notifications & Alert Channels

**What:** Implement notification delivery for spend alerts, budget threshold warnings, approval requests, and policy violations via push notifications (mobile), email, Slack, and Microsoft Teams.

**Design:**

```typescript
// apps/api/src/modules/notifications/service.ts
export class NotificationService {
  async send(notification: Notification): Promise<void> {
    const preferences = await this.getUserPreferences(notification.recipientId);
    const channels: NotificationChannel[] = [];

    if (preferences.push) channels.push(new PushChannel());
    if (preferences.email) channels.push(new EmailChannel());
    if (preferences.slack) channels.push(new SlackChannel());
    if (preferences.teams) channels.push(new TeamsChannel());

    await Promise.allSettled(
      channels.map(ch => ch.deliver(notification))
    );
  }
}
```

**Testing:**
- Budget alert at 80%: notification sent to budget owner via configured channels
- Approval request: push notification delivered within 5 seconds of submission
- Slack integration: bot posts to configured channel with approve/reject buttons
- Email notification: rendered HTML email with transaction details and action links
- User preference: user disables email, only receives push and Slack
- Notification deduplication: same event does not send duplicate notifications

### Definition of Done -- Phase 10

- [ ] React Native app with authentication, transaction list, receipt capture
- [ ] Camera-based receipt capture with OCR preview
- [ ] Approval queue with approve/reject actions on mobile
- [ ] Push notifications for spend alerts, approvals, and policy violations
- [ ] Slack and email notification channels working
- [ ] Offline receipt capture with sync-on-reconnect

---

## Phase 11: Security Hardening & Compliance

**Goal:** Harden the platform for PCI DSS 4.0.1, SOC 2 Type II readiness, and GDPR compliance. Implement penetration testing, security scanning, and compliance documentation.

**Duration:** 3 weeks

**Depends on:** Phase 10

### Task 11.1: PCI DSS Scope Minimization & Assessment

**What:** Audit the full codebase and infrastructure for PCI DSS 4.0.1 compliance. Verify that no PAN, CVV, or cardholder authentication data is stored, processed, or transmitted. Implement remaining PCI controls: encryption at rest and in transit, quarterly network scans, access logging.

**Design:**

```typescript
// Security controls checklist (implemented throughout, verified here):
// 1. No PAN storage: grep entire codebase for card number patterns
// 2. TLS 1.2+ enforced on all API endpoints
// 3. Encryption at rest: PostgreSQL TDE or column-level encryption for sensitive fields
// 4. OAuth tokens encrypted with AES-256 before storage
// 5. Audit log captures all admin actions with IP and user agent
// 6. API rate limiting on all endpoints
// 7. Webhook signature verification on all inbound webhooks
// 8. CORS configuration restricts origins to known domains
// 9. CSP headers on web frontend
// 10. Dependency vulnerability scanning (Snyk/npm audit) in CI
```

**Testing:**
- PAN scan: `grep -r` for 16-digit card number patterns across codebase returns zero results
- TLS: HTTP (non-TLS) requests to API are rejected or redirected
- Encryption: `integrations.oauth_access_token_encrypted` is ciphertext, not plaintext
- Audit coverage: every API mutation endpoint produces an audit log entry
- Dependency scan: `npm audit` returns zero critical vulnerabilities
- OWASP ZAP scan: no high-severity findings on the API
- SQL injection: parameterized queries verified; SQLi attempts return 400 (not 500)
- XSS: CSP headers verified; inline script attempts blocked

### Task 11.2: SOC 2 & GDPR Controls

**What:** Implement SOC 2 Type II controls for Security, Availability, and Confidentiality trust criteria. Implement GDPR data subject rights (access, deletion, portability) and data retention policies.

**Testing:**
- Data subject access request: API endpoint returns all data associated with a user email
- Data deletion request: API endpoint anonymizes/deletes user data across all tables
- Data retention: audit log entries older than configurable retention period (e.g., 7 years) are archived
- Access control: verify RBAC enforced on all endpoints (matrix test)
- Incident response: documented procedure for data breach notification
- Change management: all production changes require PR review (enforced by branch protection)

### Task 11.3: Security Scanning in CI

**What:** Add automated security scanning to the CI pipeline: SAST (static analysis), SCA (dependency vulnerability scanning), container image scanning, and secret detection.

**Testing:**
- CI fails if a critical CVE is found in dependencies
- CI fails if a hardcoded secret (API key, password) is detected in code
- Container image scan: no critical vulnerabilities in base images
- SAST: no high-severity findings from CodeQL or Semgrep

### Definition of Done -- Phase 11

- [ ] PCI DSS scope assessment complete; no PAN/CVV stored anywhere
- [ ] All data encrypted at rest and in transit
- [ ] GDPR data subject rights endpoints (access, deletion, portability) implemented
- [ ] Data retention policy configured and enforced
- [ ] Automated security scanning in CI: SAST, SCA, secret detection, container scan
- [ ] OWASP ZAP scan passes with no high-severity findings
- [ ] SOC 2 control matrix documented with evidence
- [ ] Penetration test findings remediated

---

## Phase 12: Self-Hosting, Documentation & Launch

**Goal:** Package the platform for self-hosting with Docker Compose and Kubernetes Helm charts. Write comprehensive documentation (setup guide, API reference, admin guide). Prepare for public launch.

**Duration:** 3 weeks

**Depends on:** Phase 11

### Task 12.1: Self-Hosting Packaging

**What:** Create production-ready Docker images, Docker Compose configuration for single-server deployment, and Helm charts for Kubernetes deployment. Include all dependencies (PostgreSQL, Redis, MinIO, Keycloak) with sensible defaults.

**Design:**

```yaml
# infra/docker/docker-compose.prod.yml
services:
  api:
    image: ghcr.io/spend-management/api:latest
    environment:
      DATABASE_URL: postgres://...
      REDIS_URL: redis://...
      S3_ENDPOINT: http://minio:9000
      CARD_ISSUER: stripe_issuing
    depends_on: [postgres, redis, minio]

  web:
    image: ghcr.io/spend-management/web:latest

  worker:
    image: ghcr.io/spend-management/worker:latest
    # Processes background jobs: OCR, sync, notifications

  ocr:
    image: ghcr.io/spend-management/ocr:latest
    # Python OCR service

  postgres:
    image: postgres:16
    volumes: [pgdata:/var/lib/postgresql/data]

  redis:
    image: redis:7

  minio:
    image: minio/minio
```

**Testing:**
- `docker compose up` on a clean machine starts all services and passes health checks
- Fresh deployment: database migrations run automatically on first start
- Seed data loaded: currencies, MCC codes available
- Web dashboard accessible at `http://localhost:3000`
- API accessible at `http://localhost:8080`
- Configuration via environment variables: all settings overridable
- Upgrade path: pull new images, `docker compose up -d` -- migrations run automatically
- Backup: PostgreSQL backup script included and tested
- Helm chart: `helm install spend-mgmt ./helm` deploys to Kubernetes cluster

### Task 12.2: API Documentation

**What:** Generate API reference documentation from the OpenAPI 3.1 specification. Write getting-started guide, authentication guide, webhook integration guide, and ERP integration guide.

**Testing:**
- OpenAPI spec validates with `spectral lint` (zero errors)
- API reference site generated and serves at `/docs`
- Every endpoint has a working curl example
- Webhook integration guide: step-by-step for Stripe Issuing webhook setup
- Authentication guide covers OIDC, SAML, and API key flows

### Task 12.3: Admin & User Documentation

**What:** Write administrator setup guide (installation, configuration, SSO setup, card issuer connection) and end-user guides (submitting expenses, managing budgets, approving requests).

**Testing:**
- Admin guide: follow the guide on a clean Ubuntu 24.04 server; platform running within 30 minutes
- User guide: non-technical user can submit a reimbursement following the guide
- Configuration reference: every environment variable documented with default value and description

### Task 12.4: Launch Preparation

**What:** Final QA pass, performance testing, load testing, and public repository preparation (LICENSE, CONTRIBUTING.md, CHANGELOG, GitHub Issues templates).

**Testing:**
- Load test: 100 concurrent users, 500 transactions/minute sustained for 10 minutes
- p99 API latency < 500ms under load
- Zero data inconsistencies after load test (budget totals match transaction sums)
- Repository passes GitHub community standards checklist
- CI green on the release branch
- Docker images published to GitHub Container Registry

### Definition of Done -- Phase 12

- [ ] Docker Compose deployment works on a clean machine
- [ ] Helm chart deploys to Kubernetes
- [ ] API documentation generated from OpenAPI spec
- [ ] Admin setup guide tested on clean Ubuntu server
- [ ] Load test passes: 500 txn/min, p99 < 500ms
- [ ] LICENSE (Apache 2.0), CONTRIBUTING.md, CHANGELOG present
- [ ] GitHub repository public with issue templates and CI passing
- [ ] v1.0.0 release tagged

---

## Summary

| Phase | Duration | Depends On | Key Deliverables |
|-------|----------|------------|------------------|
| 1. Foundation | 3-4 weeks | -- | Monorepo, DB schema, auth, API skeleton, CI/CD, web shell |
| 2. Card Programme | 3-4 weeks | Phase 1 | Card issuer adapter, card CRUD, webhook transaction ingestion |
| 3. Policy & Budget | 3-4 weeks | Phase 2 | Policy engine, budget management, spend limits, alerts |
| 4. Receipt & GL Coding | 3-4 weeks | Phase 2 | Receipt capture, OCR, chart of accounts, AI GL coding |
| 5. Approvals & Reimbursements | 3 weeks | Phases 3, 4 | Approval workflows, reimbursement lifecycle |
| 6. AP Automation | 3 weeks | Phase 5 | Vendor management, bill capture, payment scheduling |
| 7. ERP Integrations | 4 weeks | Phase 5 | QuickBooks sync, Xero/NetSuite stubs, field mapping |
| 8. Multi-Entity & Reporting | 3 weeks | Phases 6, 7 | Multi-entity, analytics dashboard, data export |
| 9. AI Intelligence | 4 weeks | Phase 8 | AI policy agent, budget forecasting, anomaly detection |
| 10. Mobile & Notifications | 3 weeks | Phase 9 | React Native app, push notifications, Slack/email |
| 11. Security & Compliance | 3 weeks | Phase 10 | PCI DSS, SOC 2, GDPR, security scanning |
| 12. Self-Hosting & Launch | 3 weeks | Phase 11 | Docker/K8s packaging, docs, load testing, v1.0 release |

**Total estimated duration:** 38-42 weeks (with Phases 3/4 and 6/7 running in parallel, critical path is ~32-36 weeks)

**Data model:** Hybrid Relational + JSONB (Model 3, 17 tables) augmented with `expense_codings` table (Model 1) for split-coding and `events` table (Model 2) for immutable audit trail. Graph layer (Model 4) deferred to post-v1 for advanced hierarchy queries.
