# Corporate Card & Spend Management — Feature & Functionality Survey

> Candidate #72 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Ramp | Commercial SaaS | Proprietary / interchange-funded + premium tier | https://ramp.com |
| Brex | Commercial SaaS (acquired by Capital One, Apr 2026) | Proprietary / freemium + premium | https://www.brex.com |
| BILL Spend & Expense (Divvy) | Commercial SaaS | Proprietary / interchange-funded | https://www.bill.com/product/spend-and-expense |
| Navan (fka TripActions) | Commercial SaaS | Proprietary / freemium + travel revenue | https://navan.com |
| Airbase (Paylocity) | Commercial SaaS | Proprietary / tiered by company size | https://www.airbase.com |
| Spendesk | Commercial SaaS | Proprietary / custom | https://www.spendesk.com |
| Pleo | Commercial SaaS | Proprietary / per-user | https://www.pleo.io |
| Expensify | Commercial SaaS | Proprietary / per-user | https://www.expensify.com |
| SAP Concur | Commercial SaaS | Proprietary / per-user + services | https://www.concur.com |
| Firefly III | Open Source | AGPL v3 | https://www.firefly-iii.org |

## Feature Analysis by Solution

### Ramp

**Core features**
- Corporate charge cards (physical and virtual) with configurable spend limits per card and merchant category
- Real-time expense management with auto-captured receipts at point of swipe
- AI-powered GL coding: auto-codes every transaction and bill across GL, department, class, location, and custom fields
- Accounts payable automation with bill capture, approval routing, and payment scheduling
- Budget management with department and project-level spend tracking and alerts
- Employee reimbursement for out-of-pocket expenses
- Vendor management with contract tracking, renewal reminders, and price intelligence
- Travel booking integration via Navan partnership

**Differentiating features**
- Ramp Intelligence AI platform: four AI agents for accounts payable alone (auto-coding, fraud prevention, approval routing, automatic payment); Policy Agent reviews 100% of expenses automatically with up to 99% policy enforcement accuracy
- Accounting Agent: automates bookkeeping by reviewing and auto-coding transactions at the moment they happen — 100% of transactions coded, not just high-confidence ones
- Ramp Token Spend Intelligence (2026): tracks AI tool costs at the model, team, and prompt level — unique capability for companies managing significant LLM spend
- Procurement AI agents (2026): triage employee purchase requests, source vendors, review contract terms, and handle compliance checks autonomously
- Vendor price intelligence benchmarking: flags when Ramp customer's unit costs for SaaS tools or services exceed anonymised peer benchmarks
- Remained independent in 2026 (Capital One acquired Brex, not Ramp); customer-driven product roadmap continues

**UX patterns**
- Card-first navigation with spend feed as the primary interface
- Receipt submission via SMS, Slack, Microsoft Teams, or mobile app; OCR extracts invoice data with claimed 99% accuracy
- Real-time budget dashboards showing current-period spend vs. budget by department, project, and cost centre
- Manager approval queue with AI-pre-scored policy compliance context

**Integration points**
- Native accounting integrations: QuickBooks, Xero, NetSuite, Sage Intacct, Microsoft Dynamics 365
- HRIS integrations: Workday, BambooHR, Rippling for automated card provisioning/deprovisioning on hire/termination
- SSO via SAML 2.0 and OIDC
- REST API and webhooks
- 300+ third-party integrations via native connectors and Zapier

**Known gaps**
- US-focused; international card issuance and multi-currency reimbursement limited compared to Brex or Spendesk
- Credit underwriting model (charge card, not credit card) means no revolving credit; some companies require credit flexibility
- No native travel booking product (dependent on Navan partnership)

**Licence / IP notes**
- Fully proprietary SaaS (Ramp Financial Corporation; privately held)
- Card issuance infrastructure runs on Visa network via programme management agreements with partner banks

---

### Brex

**Core features**
- Corporate cards (physical and virtual) for both startups and global enterprise
- Expense management with AI-generated receipts for 1,000+ merchants and pre-populated memos
- Business accounts with FDIC-insured deposits (up to $6M via sweep networks)
- Global reimbursements in 60+ currencies
- Travel booking via Brex Travel with integrated corporate cards
- Bill pay with AP automation
- Budget management with team and project-level spend controls

**Differentiating features**
- Acquired by Capital One in April 2026 for $5.15 billion — the largest corporate card acquisition in US history; future product direction uncertain pending integration
- AI-powered expense automation: auto-generated receipts for 1,000+ merchants eliminates manual receipt collection for a large portion of common business spend
- Global focus: strongest multi-currency and international reimbursement capabilities among US-headquartered spend platforms
- Brex AI: expense categorisation, policy enforcement, and variance detection embedded throughout the product

**UX patterns**
- Modern, visually refined interface; historically praised for consumer-grade design in a B2B product
- Employee expense submission through mobile app, email forwarding, or Slack
- Spend analytics with configurable dimensions and benchmark comparisons

**Integration points**
- Native integrations with QuickBooks, Xero, NetSuite, Sage, Oracle, and SAP
- HRIS integrations with Workday, Rippling, and others
- REST API and webhooks

**Known gaps**
- Post-Capital One acquisition: product roadmap and pricing changes expected; independent Brex identity may diminish
- SMB segment de-emphasised since 2022 pivot to enterprise; less compelling for early-stage startups
- Some enterprise customers report slower product innovation post-Series D compared to pre-unicorn era

**Licence / IP notes**
- Fully proprietary (acquired by Capital One Financial Corporation, NYSE: COF, April 2026)
- Card issuance historically via Column Bank and other programme management partners; post-acquisition may shift to Capital One's own bank licence

---

### BILL Spend & Expense (Divvy)

**Core features**
- Corporate cards with pre-loaded budget model: funds allocated to budgets before spend, preventing overspend by design
- Real-time virtual card creation for specific vendors, projects, or time periods
- Receipt capture and expense categorisation
- Budget owner workflow: business owners allocate spend to budget buckets; employees request funds from their allocation
- Reimbursement management for out-of-pocket expenses
- AP automation integrated with BILL's broader accounts payable platform

**Differentiating features**
- Pre-spend budget model is architecturally unique: rather than approving expenses after the fact, Divvy pre-allocates funds to budgets; employees spend only what has been approved — policy violations are structurally prevented rather than detected reactively
- Interchange-funded model: core platform is free for employers because the business model monetises interchange revenue on card transactions
- Deep integration with BILL AP for a combined spend management + accounts payable workflow on one platform

**UX patterns**
- Budget-centric navigation with budget allocation as the primary financial management surface
- Fund request workflow: employees request spending authority before committing to purchases
- Receipt submission via mobile app with OCR auto-categorisation

**Integration points**
- Native integrations with QuickBooks, Xero, NetSuite, and Sage
- BILL AP integration for combined payables workflow
- REST API for custom integrations

**Known gaps**
- Pre-spend model requires behaviour change from employees accustomed to traditional expense reimbursement; higher change management burden
- Less sophisticated AI coding and policy enforcement than Ramp
- International support limited compared to Brex or Spendesk

**Licence / IP notes**
- Fully proprietary (BILL Holdings Inc., NYSE: BILL); Divvy acquired 2021 for $2.5B

---

### Navan (formerly TripActions)

**Core features**
- Integrated travel booking (flights, hotels, rail, car) with corporate card in a single platform
- Virtual cards automatically created and tied to each travel booking
- Expense management for travel and non-travel spend
- Policy engine with approved-vendor catalogues and negotiated rates
- Travel programme analytics with savings tracking vs. out-of-policy alternatives
- Employee safety and duty-of-care tracking during travel

**Differentiating features**
- Only major spend management platform that started as a travel platform and built cards/expense management on top; uniquely deep travel + spend integration
- Virtual card per booking: each trip generates a single-use virtual card pre-loaded with the approved itinerary cost, eliminating manual reconciliation entirely
- Corporate travel savings benchmarking based on negotiated rates and aggregated booking data

**UX patterns**
- Travel-first interface with booking as the primary entry point
- Integrated itinerary management with expense auto-capture during travel
- Expense review workflows triggered automatically by trip completion

**Integration points**
- Native integrations with major accounting and ERP systems
- HR integrations for traveller profile sync
- Concur data migration tools for enterprises switching from SAP Concur

**Known gaps**
- Travel specialisation means non-travel spend categories (SaaS, office supplies, recurring vendor payments) are less sophisticated than Ramp or Brex
- AP automation capabilities less developed than Airbase or Ramp
- IPO anticipated but not yet completed as of 2026; growth-stage capital structure

**Licence / IP notes**
- Fully proprietary (Navan Inc.; privately held at ~$9.2B valuation)

---

### SAP Concur

**Core features**
- Travel booking with negotiated corporate rates and policy enforcement
- Expense report submission, approval, and reimbursement
- Invoice management and AP automation
- Corporate card reconciliation and statement management
- Global expense compliance and VAT reclaim for international travellers
- Analytics and reporting on travel and expense programmes

**Differentiating features**
- Dominant in large enterprise and SAP-shop environments; deepest SAP ERP integration of any travel and expense tool
- Global compliance depth: VAT reclaim, per diem rules, and local tax regulations for 100+ countries
- TripLink: captures off-channel travel bookings (direct hotel, airline) in the Concur data model automatically

**UX patterns**
- Complex, multi-screen expense report interface consistent with enterprise-grade software circa 2015
- Mobile app for receipt capture and approval on the go (functional but less refined than Ramp/Brex apps)

**Integration points**
- Deep native SAP ERP integration via SAP Business Technology Platform
- Connections to all major GDS (Amadeus, Sabre, Travelport) for travel content
- REST API for custom integrations

**Known gaps**
- Widely regarded as having the worst user experience of any major expense platform; consistently lowest satisfaction scores
- Expensive total cost of ownership including services and customisation
- Legacy architecture limits agility for new feature development compared to cloud-native competitors

**Licence / IP notes**
- Fully proprietary (SAP SE, NYSE: SAP)

---

### Firefly III

**Core features**
- Personal and small-business finance tracking with transaction categorisation
- Multi-account management with bank import
- Budget tracking with configurable category budgets
- Recurring transaction management
- Basic reporting and chart generation

**Differentiating features**
- Only open-source option in the broader spend tracking space; self-hostable with full data ownership
- No usage limits; suitable for individuals and very small teams tracking personal finances

**UX patterns**
- Traditional web application with list-based transaction management
- Import tools for bank CSV/OFX/QIF statement files

**Integration points**
- REST API for programmatic access
- Bank import via Nordigen/GoCardless open banking connection (EU)

**Known gaps**
- Not designed for corporate card programmes, multi-user expense workflows, or team-level budget management
- No virtual card issuance, AP automation, or policy enforcement capabilities
- Not a viable base for a corporate spend management platform

**Licence / IP notes**
- AGPL v3: strong copyleft; network use triggers copyleft obligation; not suitable as a base for a SaaS-hosted OSS project without full open-source commitment

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Physical and virtual corporate card issuance with configurable spend limits per card
- Real-time transaction feed with merchant data enrichment
- Receipt capture via mobile, email, SMS, and Slack/Teams integration
- Automated GL coding and expense categorisation with machine learning
- Policy engine with configurable rules (spend limits, merchant categories, required receipt thresholds)
- Employee reimbursement workflow for out-of-pocket expenses
- ERP/accounting sync (QuickBooks, Xero, NetSuite minimum) with configurable GL mapping
- Budget management with department and project-level spend tracking and alerts
- AP/bill pay automation with invoice capture, approval routing, and scheduled payments
- Role-based access for card admin, budget owner, finance manager, and employee personas
- Audit trail on all transactions, approvals, policy exceptions, and configuration changes

### Differentiating Features
- AI Policy Agent reviewing 100% of expenses automatically vs. sampling-based human review
- Automated receipt generation for common merchants (eliminating employee receipt-hunting entirely)
- Pre-spend budget model (Divvy/BILL model) structurally preventing overspend rather than detecting it reactively
- Virtual card per booking/project/vendor with automated reconciliation on close
- AI-powered GL coding of 100% of transactions across all dimension fields
- Vendor price intelligence benchmarking using anonymised peer spend data
- AI procurement agents: sourcing vendors, reviewing contract terms, handling compliance checks autonomously
- AI token spend intelligence: cost tracking for LLM API consumption by team and model
- Integrated travel booking with card and expense reconciliation (Navan model)

### Underserved Areas / Opportunities
- Open-source spend management software layer: card issuance requires bank/network partnerships, but the policy engine, receipt processing, GL coding, and budget management layers are pure software problems fully buildable as OSS
- Integration with existing card programmes (Stripe Issuing, Lithic, Marqeta) for enterprises that already have banking relationships and want the software layer without card-issuer lock-in
- Multi-entity spend management for PE-backed portfolio companies and multi-subsidiary enterprises — most platforms handle single entities well but multi-entity consolidation is a gap
- Mid-market European spend management: Spendesk and Pleo serve SMB; SAP Concur serves enterprise; the mid-market is underserved for SEPA-native spend management
- AI-driven contract renewal management: proactive alerts and negotiation intelligence 60–90 days before vendor renewals

### AI-Augmentation Candidates
- Real-time policy enforcement at point of spend: AI evaluating each transaction against role, project, prior spend history, and policy intent — nuanced decisions vs. blunt category blocks
- Receipt-to-GL coding: model trained on organisation's historical coding patterns auto-coding 80–90% of transactions without human review
- Budget forecasting and burn alerts: projecting month-end and quarter-end spend 2–3 weeks ahead based on committed expenses and historical cadence
- Vendor negotiation intelligence: surfacing price benchmarks and committed-use discount eligibility from anonymised peer data
- Anomaly and fraud detection: flagging statistical outliers in transaction patterns before payment settlement
- Procurement agent: autonomous vendor sourcing, contract term extraction, and compliance verification for new vendor requests

---

## Legal & IP Summary

Corporate card issuance is legally constrained: card network membership (Visa, Mastercard) requires an issuing bank partnership or programme manager arrangement, regulated by each network's rules. This layer is not buildable as open source. However, the spend management, policy enforcement, receipt processing, GL coding, and budget management software layers have no IP constraints and are fully buildable as OSS. Ramp's AI agents and pricing intelligence capabilities are proprietary implementations but use standard ML/NLP techniques with no known patent protection. BILL's pre-spend budget model architecture is a product design approach, not a patented system. Firefly III's AGPL v3 licence makes it unsuitable as a codebase for a SaaS-hosted commercial open-source project; any new OSS spend management project should be built from scratch on Apache 2.0 or MIT. The Capital One acquisition of Brex (April 2026) changes the competitive landscape: Brex's independent product roadmap is now subject to Capital One's strategic priorities, which may create switching demand among startups that previously relied on Brex's startup-friendly positioning.

---

## Recommended Feature Scope

**Must-have (MVP)**:
- Integration with virtual card issuance APIs (Stripe Issuing, Lithic, or Marqeta) for card programme management without a proprietary card product
- Real-time transaction ingestion with merchant enrichment and receipt capture (mobile, email, Slack)
- AI-powered GL coding and expense categorisation mapped to configurable chart of accounts
- Policy engine with configurable rules for spend limits, merchant categories, and receipt requirements
- Employee reimbursement workflow with approval routing and ACH settlement
- Budget management with department/project-level tracking and real-time alerts
- ERP sync: QuickBooks, Xero, and NetSuite at minimum
- Role-based access with card admin, budget owner, finance manager, and employee personas

**Should-have (v1.1)**:
- AI Policy Agent reviewing 100% of expenses automatically with configurable enforcement actions
- AP/bill pay automation with invoice OCR capture, approval routing, and payment scheduling
- Multi-entity support with entity-level budget controls and consolidated reporting
- Virtual card management: single-use and recurring cards tied to vendors, projects, or bookings
- Vendor management with contract tracking, renewal reminders, and price intelligence

**Nice-to-have (backlog)**:
- Pre-spend budget model (fund allocation before spend, Divvy-style) as an optional configuration mode
- AI procurement agents for autonomous vendor sourcing and contract review
- Multi-currency support with real-time FX rates and foreign transaction handling
- Travel booking integration via partner API
- Budget forecasting and burn alerts with 2–3 week forward projection
