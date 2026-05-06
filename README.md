# Corporate Card & Spend Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native spend management platform that delivers virtual cards, budget controls, and real-time spend visibility without locking enterprises into a proprietary card issuer.

Corporate Card & Spend Management is a software layer for finance teams who want the policy enforcement, receipt processing, GL coding, and budget management capabilities of Ramp or Brex while integrating with their existing card programmes via Stripe Issuing, Lithic, or Marqeta. It targets startup CFOs, mid-market finance leaders, and multi-entity controllers who need modern spend tooling without surrendering their banking relationships or financial data.

---

## Why Corporate Card & Spend Management?

- **No open-source alternative exists for corporate spend.** Every major platform (Ramp, Brex, BILL Spend & Expense, Navan, Airbase, Spendesk, Pleo, Expensify, SAP Concur) is proprietary. Firefly III, the only OSS option in the adjacent space, is AGPL-licensed and not designed for corporate card programmes.
- **Card-issuer lock-in is unnecessary.** Card issuance requires network membership, but the spend management software layer is a pure software problem. Enterprises with existing banking relationships are forced to adopt proprietary card products to access modern spend tooling.
- **Incumbent pricing leaves gaps.** Paid tiers run $5–$17/user/month (Expensify, Pleo, Ramp Plus, Brex Premium); SAP Concur adds $8–$12/user/month plus services. Free interchange-funded tiers (Ramp Core, BILL Spend & Expense) trade independence for revenue capture on every swipe.
- **Capital One's April 2026 acquisition of Brex creates switching demand.** Startups that relied on Brex's independent product roadmap now face uncertainty over pricing and product direction.
- **Mid-market and multi-entity buyers are underserved.** Spendesk and Pleo cover European SMB; SAP Concur dominates large enterprise; PE-backed portfolios and multi-subsidiary companies struggle to consolidate spend across entities.

---

## Key Features

### Card Programme Integration

- Integration with virtual card issuance APIs: Stripe Issuing, Lithic, and Marqeta
- Physical and virtual card support with configurable spend limits per card
- Single-use and recurring virtual cards tied to vendors, projects, or bookings
- Card admin role with provisioning and deprovisioning workflows

### Real-Time Spend & Receipts

- Real-time transaction ingestion with merchant data enrichment
- Receipt capture via mobile, email, SMS, and Slack/Teams
- Automated GL coding and expense categorisation against a configurable chart of accounts
- Audit trail across transactions, approvals, policy exceptions, and configuration changes

### Policy & Budget Controls

- Configurable policy engine for spend limits, merchant categories, and receipt requirements
- Department- and project-level budget tracking with real-time alerts
- Optional pre-spend budget model (BILL/Divvy-style fund allocation before spend)
- Multi-entity support with entity-level budget controls and consolidated reporting

### Workflows & Reimbursement

- Employee reimbursement workflow with approval routing and ACH settlement
- AP/bill pay automation with invoice OCR capture, approval routing, and scheduled payments
- Role-based access for card admins, budget owners, finance managers, and employees
- Vendor management with contract tracking and renewal reminders

### Integrations

- Native ERP/accounting sync: QuickBooks, Xero, NetSuite (minimum)
- HRIS integration patterns for automated provisioning on hire/termination
- SSO via SAML 2.0 and OIDC
- REST API and webhooks for custom workflows

---

## AI-Native Advantage

An AI Policy Agent reviews 100% of expenses in real time against the full context of employee role, project budget, prior spend history, and policy intent — replacing blunt category blocks with nuanced approvals. AI-powered receipt-to-GL coding learns from an organisation's historical coding patterns to auto-code 80–90% of transactions, escalating only ambiguous items. Forward-looking budget forecasting projects month-end and quarter-end spend 2–3 weeks ahead from committed expenses and historical cadence, surfacing overruns before period close. Vendor negotiation intelligence draws on anonymised peer benchmarks to flag overpriced contracts and committed-use discount eligibility, building a network-effect data moat without proprietary card lock-in.

---

## Tech Stack & Deployment

The platform is designed as a self-hostable software layer that integrates with third-party card issuers (Stripe Issuing, Lithic, Marqeta) rather than operating its own card network programme. Relevant standards include PCI DSS for cardholder data handling, ISO 4217 for multi-currency transactions, ISO 20022 / SWIFT for cross-border flows, NACHA ACH for reimbursement settlement, and Open Banking / PSD2 for bank-feed data ingestion. SOC 2 Type II is expected by enterprise buyers. Integrations target QuickBooks, Xero, NetSuite, Sage Intacct, and Microsoft Dynamics 365, with SSO via SAML 2.0 and OIDC.

---

## Market Context

The spend management platform market is $25.78B in 2025, growing to $29.19B in 2026 (13.2% CAGR) and projected at $45.93B by 2030 (Research and Markets). The commercial / corporate card market reaches $28.4B in 2025 and is projected at $58.7B by 2034 (DataIntelo), with virtual card transaction volume forecast at $6.8 trillion in 2026. Primary buyers are startup CFOs (Series A–C, 10–200 employees), VPs of Finance at mid-market companies (200–2,000 employees), enterprise procurement and AP directors, and controllers at multi-entity or international companies.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
