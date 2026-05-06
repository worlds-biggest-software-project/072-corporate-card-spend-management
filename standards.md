# Standards & API Reference

> Project: Corporate Card & Spend Management · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO 20022 — Financial Services Universal Financial Industry Message Scheme**
- URL: https://www.iso20022.org/iso-20022-message-definitions
- The global standard for financial message exchange between institutions. Covers payments, FX, cards, trade finance, and securities. Corporate card spend platforms need ISO 20022 alignment for cross-border reimbursements, ACH/wire payment initiation, and interbank settlement. As of November 2026, SWIFT CBPR+ payment messages must use structured address formats and conform to ISO 20022 CBPR+ API payload specifications — non-conformant API requests will be rejected. ISO 20022 is progressively replacing the older card payment messaging standard ISO 8583 for corporate card programme management.

**ISO 4217 — Currency Codes**
- URL: https://www.iso.org/iso-4217-currency-codes.html
- The international standard defining three-letter currency codes (USD, EUR, GBP, etc.) and numeric codes for all currencies. Any multi-currency expense management or reimbursement system must implement ISO 4217 across all transaction records, FX conversion events, and reporting outputs. Mandatory for platforms supporting global card programmes.

**ISO 8583 — Financial Transaction Card Originated Messages**
- URL: https://www.iso.org/standard/31628.html
- Legacy card payment message standard used at point-of-sale and ATM networks. Still active in card authorisation and clearing flows; corporate card programmes built on Visa/Mastercard networks communicate card authorisations using ISO 8583 message formats. Spend management platforms receiving real-time card authorisation webhooks from card-issuing partners (Stripe Issuing, Marqeta, Lithic) will encounter ISO 8583-derived data structures.

**ISO/IEC 27001 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The international standard for information security management systems (ISMS). Corporate finance data is a high-value target; spend management platforms storing card data, transaction history, and employee expense records are expected to hold ISO 27001 certification by enterprise procurement teams. Often evaluated alongside SOC 2 Type II.

---

### W3C & IETF Standards

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational authorization protocol for delegated access. All major spend management platforms (Ramp, Brex, BILL, SAP Concur) use OAuth 2.0 for their developer API authentication. The Authorization Code with PKCE flow (RFC 7636) is the recommended pattern in 2026 for all client types — web apps, SPAs, mobile, and CLIs. Any open-source spend management platform must implement OAuth 2.0 for both its own API and for connecting to accounting systems (QuickBooks, Xero, NetSuite).

**RFC 7636 — Proof Key for Code Exchange (PKCE)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Extension to OAuth 2.0 Authorization Code flow that prevents authorization code interception attacks. Required by FAPI 2.0 and mandated as the universal OAuth flow for all client types in 2026. Spend management mobile apps and SPAs must implement PKCE.

**SAML 2.0 — Security Assertion Markup Language 2.0**
- URL: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- The dominant enterprise SSO federation protocol. Enterprise procurement teams at mid-market and large companies expect SAML 2.0 support for single sign-on. Ramp, Brex, and SAP Concur all support SAML 2.0 SSO. Any spend management platform targeting companies above 200 employees must implement both SAML 2.0 and OIDC; SAML 2.0 is embedded in FedRAMP, HIPAA, PCI-DSS, and banking sector compliance frameworks.

**OpenID Connect (OIDC) — Identity Layer on OAuth 2.0**
- URL: https://openid.net/connect/
- Thin authentication layer on top of OAuth 2.0 that adds ID Tokens (JWT), a userinfo endpoint, and standardized scopes. The preferred SSO protocol for new deployments; winning against SAML for mid-market SaaS buyers. Spend management platforms should support OIDC for modern IdP integrations (Okta, Auth0, Azure AD, Google Workspace).

**RFC 6455 — WebSocket Protocol**
- URL: https://datatracker.ietf.org/doc/html/rfc6455
- Standard for persistent bidirectional browser-server connections. Relevant for real-time spend notification delivery (e.g., alerting finance teams of policy violations or budget threshold breaches the moment a card transaction clears).

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://swagger.io/specification/
- The de-facto standard for describing RESTful API contracts. Used by Brex (whose API is described in OpenAPI and published to Postman), Stripe Issuing, and most modern spend management developer portals. Any open-source spend management platform should publish an OAS 3.1 specification to enable SDK generation, API documentation tooling, and iPaaS platform auto-discovery of endpoints, authentication methods, and data structures. Over 90% of Fortune 500 companies use OpenAPI to manage API ecosystems.

**OFX (Open Financial Exchange) Format**
- URL: https://www.ofx.net/
- A unified data format for exchanging financial data between financial institutions and consumer/business finance software. OFX/QBO (QuickBooks Online bank feed format) is the most widely supported format for importing card and bank transaction history into accounting software. Spend management platforms should support OFX export so finance teams can import transaction history into accounting systems that lack native API integrations.

**IIF (Intuit Interchange Format)**
- URL: https://developer.intuit.com/
- Intuit's proprietary import/export format for QuickBooks Desktop (not QuickBooks Online). Supports bills, invoices, journal entries, and accounting data import directly into company files. Legacy format; still relevant for SMB customers on QuickBooks Desktop. Secondary to OFX/QBO for most integrations.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Standard for describing and validating JSON document structure. Spend management API request/response bodies, webhook payloads, and data export formats should all have JSON Schema definitions to enable automated validation, SDK generation, and integration testing.

---

### Security & Authentication Standards

**PCI DSS 4.0.1 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/
- The mandatory global security standard for all entities that store, process, or transmit cardholder data. All 64 PCI DSS v4.0.1 requirements became mandatory on March 31, 2025 and remain fully in force in 2026. Corporate card spend management platforms must achieve PCI DSS Level 1 compliance (highest level) for card programme management. Virtual card issuance requires tokenization controls. Even platforms that outsource card processing to partners (Stripe, Marqeta, Lithic) must complete annual PCI Self-Assessment Questionnaires (SAQ) and cannot rely solely on their partners' PCI compliance. Key 2026 requirements include: quarterly network scans, real-time monitoring, and annual scope documentation.

**SOC 2 Type II — Service Organization Control 2**
- URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2
- The primary security audit report demanded by enterprise buyers as a vendor qualification criterion. Corporate finance platforms handling payment card data and employee expense records are expected to hold SOC 2 Type II certification covering the Security, Availability, and Confidentiality trust service criteria. In 2026, auditors are applying tighter scrutiny to third-party risk management, AI system security, and continuous monitoring postures. Spend management platforms using AI for GL coding or policy enforcement must document AI decision traceability in SOC 2 controls.

**GDPR — General Data Protection Regulation**
- URL: https://gdpr.eu/
- Binding EU law governing personal data of EU residents. Spend management platforms with European users or SEPA payment flows must implement consent management, data minimisation, user rights support (subject access and deletion requests), and breach notification obligations. GDPR fines can reach €20M or 4% of global revenue. Platforms targeting European mid-market (competing with Spendesk or Pleo) need GDPR-compliant data models and processing agreements.

**FAPI 2.0 — Financial-grade API Security Profile**
- URL: https://openid.net/specs/fapi-2_0-security-profile.html
- The OpenID Foundation's security profile for high-value financial APIs, reaching final specification status in February 2025. Mandates Pushed Authorization Requests (PAR), PKCE for all clients, and sender-constrained tokens via mTLS or DPoP. The Berlin Group's NextGenPSD2 framework (adopted by ~75% of European banks) is built on FAPI. Spend management platforms integrating with EU open banking data feeds should implement FAPI 2.0-compliant OAuth flows.

---

### Payment & Settlement Standards

**NACHA Operating Rules — ACH Network**
- URL: https://www.nacha.org/
- The rules governing the US ACH network used for employee expense reimbursements, vendor payments, and bill pay within spend management platforms. Key 2026 changes: Phase 1 (March 20, 2026) requires large originators (≥ 6M ACH entries in 2023) to implement risk-based fraud monitoring; Phase 2 (June 22, 2026) extends this requirement to all non-consumer originators regardless of volume. Two new standardized Company Entry Descriptions are now mandatory: "PAYROLL" for wage/salary credits and "PURCHASE" for purchase-related entries. Any spend management platform originating ACH reimbursements must comply with these rules.

**PSD2 / PSD3 — EU Payment Services Directives**
- URL: https://finance.ec.europa.eu/regulation-and-supervision/financial-services-legislation/implementing-and-delegated-acts/payment-services-directive_en
- The European regulatory framework enabling open banking — third-party access to bank account data via standardized APIs. Allows spend management platforms to pull transaction data from bank accounts without screen-scraping. PSD3 (publication expected summer 2026) expands mandatory permission dashboards, eliminates screen-scraping fallbacks entirely, and mandates payee verification for credit transfers. Relevant for European spend management platforms integrating with GoCardless/Nordigen (now Nordigen is GoCardless) bank data feeds.

**SWIFT CBPR+ — Cross-Border Payments and Reporting Plus**
- URL: https://www.swift.com/standards/iso-20022/iso-20022-standards
- The SWIFT implementation of ISO 20022 for cross-border payments. From November 2026, payment messages must use structured address formats and API requests not meeting specifications will be rejected. Relevant for spend management platforms processing international reimbursements or cross-border bill payments.

---

## Similar Products — Developer Documentation & APIs

### Stripe Issuing

- **Description:** Stripe's card issuance API for building commercial card programs. Allows platforms to create virtual and physical Visa or Mastercard corporate cards, configure spend controls per card, and receive real-time authorisation webhooks. The most developer-friendly card issuance infrastructure available; no setup fees.
- **API Documentation:** https://docs.stripe.com/issuing
- **API Reference:** https://docs.stripe.com/api/issuing/cards
- **Virtual Card Guide:** https://docs.stripe.com/issuing/cards/virtual
- **SDKs/Libraries:** Official SDKs for JavaScript/Node.js, Python, Ruby, PHP, Go, Java, .NET — https://stripe.com/docs/libraries
- **Developer Guide:** https://docs.stripe.com/issuing/for-your-business
- **Standards:** REST/JSON; OpenAPI-compatible; PCI DSS compliant via Issuing Elements (card details never touch platform servers)
- **Authentication:** API key (secret key in server-side requests); Stripe.js for client-side PCI-compliant card display

---

### Lithic (Card Issuing)

- **Description:** Developer-first card issuance infrastructure for building corporate card programs. Provides virtual and physical card issuance, real-time spend controls, and authorisation decision webhooks. Positioned as an API-first alternative to Marqeta with simpler integration.
- **API Documentation:** https://docs.lithic.com/docs/welcome
- **API Basics:** https://docs.lithic.com/docs/api-basics
- **Cards Guide:** https://docs.lithic.com/docs/cards
- **SDKs/Libraries:** TypeScript and Python client SDKs; cURL examples throughout documentation — https://github.com/lithic-com/
- **Developer Guide:** https://www.lithic.com/blog/quick-start-guide (nine-section Quick Start covering API key generation, card creation, spend control customisation, and secure card number display)
- **Standards:** REST/JSON; API-first design; PCI-compliant card display
- **Authentication:** API key

---

### Marqeta (Card Issuing)

- **Description:** Enterprise card issuance platform powering major fintech card programs (Cash App Card, DoorDash, Instacart). Provides virtual and physical card issuance, real-time authorisation controls (Just-in-Time funding), and comprehensive program management. More complex integration than Stripe or Lithic; suited for high-volume enterprise card programs.
- **API Documentation:** https://www.marqeta.com/docs/core-api/introduction
- **Developer Portal:** https://www.marqeta.com/developer-overview (includes sandbox and interactive API explorer)
- **Open APIs & Webhooks:** https://www.marqeta.com/platform/open-api-webhooks
- **Quick Start:** https://www.marqeta.com/docs/developer-guides/core-api-quick-start
- **SDKs/Libraries:** SDKs available; specific languages listed at developer portal
- **Standards:** REST/JSON; webhook event delivery; interactive API browser in documentation
- **Authentication:** HTTP Basic Auth with application credentials (username/password pair)

---

### Ramp (Spend Management)

- **Description:** Corporate card and spend management platform offering a REST API for programmatic access to transactions, cards, users, budgets, receipts, and reimbursements. Includes sandbox environment mirroring production endpoints.
- **API Documentation:** https://docs.ramp.com/developer-api/
- **Webhooks Reference:** https://docs.ramp.com/developer-api/v1/webhooks
- **Developer Portal:** https://ramp.com/developer-api
- **Sandbox Access:** https://demo-api.ramp.com (mirrors production endpoints)
- **SDKs/Libraries:** REST API with JSON; no official SDKs listed; community integrations via Zapier and Pipedream
- **Standards:** REST/JSON; OAuth 2.0 for authentication; webhook events for real-time updates
- **Authentication:** OAuth 2.0 (client credentials flow for server-to-server; authorization code for partner applications)

---

### Brex (Corporate Cards & Spend Management)

- **Description:** Corporate cards and spend management platform (acquired by Capital One, April 2026). Provides a REST API described in OpenAPI specification for programmatic access to cards, transactions, users, expenses, and accounts. Strongest multi-currency and international capabilities.
- **API Documentation:** https://developer.brex.com/
- **FAQ:** https://developer.brex.com/docs/faq
- **Postman Collection:** Brex publishes an official Postman collection of their API
- **SDKs/Libraries:** REST API described in OpenAPI; SDK generation supported via OpenAPI spec
- **Standards:** REST/JSON; OpenAPI specification; OAuth 2.0 with PKCE
- **Authentication:** OAuth 2.0 (user tokens for own account; OAuth tokens for partner applications authenticating other Brex accounts)

---

### BILL Spend & Expense API

- **Description:** BILL's corporate card and spend management API providing programmatic access to virtual card creation, budget management, transaction data, and reimbursements. Separate from BILL's AP automation API; specifically focused on the Divvy-heritage card and spend product.
- **API Documentation:** https://developer.bill.com/docs/spend-expense-api
- **API Platform Home:** https://developer.bill.com/docs/home
- **Reimbursements API:** https://developer.bill.com/docs/reimbursements
- **Changelog:** https://developer.bill.com/changelog (active — April 2026 added transaction status field; January 2026 added updatedTime and authorizedTime filters)
- **SDKs/Libraries:** REST API; token-based authentication; no official SDKs listed
- **Standards:** REST/JSON (v3 API); HTTP standard methods (GET, POST, PUT, PATCH, DELETE) with HTTP response codes
- **Authentication:** API token (BILL S&E token required; authentication guide at developer.bill.com)

---

### SAP Concur API

- **Description:** Enterprise travel and expense management platform with APIs for expense report submission, approval, card reconciliation, invoice management, and travel booking. Deep integration with SAP ERP ecosystem. Largest installed base in large enterprise.
- **API Documentation:** https://developer.concur.com/ (live); https://preview.developer.concur.com/ (preview)
- **SAP API Hub:** https://api.sap.com/products/SAPConcur/apis/REST
- **Expense API Package:** https://api.sap.com/package/ConcurExpense
- **GitHub (docs repo):** https://github.com/SAP-docs/preview.developer.concur.com
- **SDKs/Libraries:** REST API; SAP Business Technology Platform for native SAP integrations; Postman collections available via SAP API Hub
- **Standards:** REST/JSON; OAuth 2.0 Application Management for access grants and scopes; OpenAPI documentation (Swagger)
- **Authentication:** OAuth 2.0 (application management via Concur developer portal)

---

### Expensify API

- **Description:** Expense reporting platform API for programmatically downloading expense report data, provisioning accounts, and submitting expenses. Older integration architecture (Integration Server pattern); webhook support is limited compared to modern platforms.
- **API Documentation:** https://integrations.expensify.com/Integration-Server/doc/
- **Web Services:** https://www.expensify.com/api-services.html
- **GitHub (integrations):** https://github.com/Expensify/Integrations
- **SDKs/Libraries:** No official SDK; REST-like Integration Server pattern with partnerUserID/partnerUserSecret credentials
- **Standards:** Custom Integration Server pattern (not standard REST); JSON request/response bodies
- **Authentication:** API key pair (partnerUserID + partnerUserSecret, generated at https://www.expensify.com/tools/integrations/)

---

### QuickBooks Online API (Intuit)

- **Description:** The primary accounting platform API for SMB expense management integrations. Provides access to customers, vendors, invoices, payments, expenses, GL accounts, and reports. Used by Ramp, Brex, BILL, and virtually every spend management platform for two-way accounting sync.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **Integration Guide:** https://www.getknit.dev/blog/quickbooks-online-api-integration-guide-in-depth
- **SDKs/Libraries:** Official SDKs for multiple languages via Intuit Developer Portal; community SDKs on GitHub
- **Standards:** REST/JSON; proprietary SQL-like query language (QueryService / SuiteQL-style) via /query endpoint; OpenAPI-compatible
- **Authentication:** OAuth 2.0 (Authorization Code with PKCE)
- **Rate Limits:** 500 requests/minute per company; 10 concurrent request limit; batch operations at 40/minute

---

### NetSuite REST API (Oracle)

- **Description:** Oracle NetSuite's REST web services and SuiteQL query API for mid-market and enterprise ERP integration. Spend management platforms targeting companies using NetSuite (common in venture-backed and PE-backed companies) must integrate with this API for GL sync, vendor management, and expense coding.
- **API Documentation:** https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/chapter_1540391670.html
- **SuiteQL Guide:** https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/section_158394344595.html
- **Developer Guide:** https://www.brokenrubik.com/blog/netsuite-rest-api-guide
- **SDKs/Libraries:** SuiteScript 2.1 for server-side automation; REST web services for external integrations; no official external SDK
- **Standards:** REST/JSON; SuiteQL (SQL-like query language via POST to /suiteql endpoint); maximum 100,000 results per query
- **Authentication:** OAuth 2.0 (Token-Based Authentication — TBA; note: TBA deprecation deadline approaching in NetSuite 2026.1 — check current Oracle guidance)

---

### Plaid (Bank Data / Open Banking)

- **Description:** Financial data infrastructure connecting bank accounts to applications. Used by spend management platforms to pull bank transaction history, verify account balances, and initiate ACH payments — particularly for out-of-pocket expense reimbursement verification and bank feed imports.
- **API Documentation:** https://plaid.com/docs/api/
- **Transactions API:** https://plaid.com/docs/api/products/transactions/
- **Core Exchange Reference:** https://plaid.com/core-exchange/docs/reference/5.1/
- **SDKs/Libraries:** Official SDKs for Node.js, Python, Ruby, Java, Go — https://plaid.com/docs/api/#libraries
- **Standards:** REST/JSON; OpenAPI spec published; webhook-driven transaction update model (webhook receiver endpoint required during integration)
- **Authentication:** OAuth 2.0; Plaid Link for user-facing bank authentication; API keys for server-to-server

---

## Notes

**Card Issuance Layer — Regulatory Constraint**
All corporate card issuance requires a card network membership agreement (Visa or Mastercard) which in practice means sponsorship by a licensed issuing bank acting as a programme manager. This layer is not accessible without regulated bank partnerships. Stripe Issuing, Marqeta, and Lithic abstract this complexity through their own bank sponsor relationships — platforms building on top of these APIs do not need their own bank licence but must comply with each provider's programme management requirements, which include PCI DSS certification and KYB (Know Your Business) verification for cardholders.

**Emerging Standards to Watch**
- **ISO 20022 JSON payloads**: The SWIFT CBPR+ JSON API guide from Standard Chartered shows the industry moving toward JSON-native ISO 20022 message encoding rather than XML-only. Spend management platforms building cross-border payment capabilities should track this evolution.
- **PSD3 Open Finance**: Expected publication summer 2026; will mandate payee verification and eliminate screen-scraping fallback. Platforms using open banking data feeds in Europe should monitor PSD3 implementation timelines.
- **FAPI 2.0 adoption**: As more European banks adopt NextGenPSD2, FAPI 2.0 compliance becomes a prerequisite for reliable open banking data access. US platforms expanding to Europe should build FAPI 2.0-compliant OAuth flows.
- **NetSuite TBA Deprecation**: Oracle's Token-Based Authentication deprecation in NetSuite 2026.1 may require updates to existing NetSuite integrations. Monitor Oracle release notes.
