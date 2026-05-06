# Standards & API Reference

> Project: OKR & Goal Tracking Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 30414:2025 — Human Resource Management: Requirements and Recommendations for Human Capital Reporting and Disclosure**
- URL: https://www.iso.org/standard/69338.html
- The world's first international standard for measuring, managing, and reporting on human capital. Provides 69 standardised metrics across 11 key areas (including workforce productivity and performance outcomes). OKR platforms that serve HR and enterprise customers should align goal-tracking metrics with this standard to enable regulatory and ESG-level reporting.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational authorisation protocol used by all major OKR platforms (Lattice, Workboard, Weekdone, Quantive). Required for delegated access when integrating with external data sources (Salesforce, Jira, GitHub) to auto-update key result progress values.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Specifies how OAuth 2.0 bearer tokens are transmitted in API requests via the `Authorization: Bearer <token>` header. Universally adopted across OKR platform APIs.

**OpenID Connect Core 1.0 (OIDC)**
- URL: https://openid.net/developers/how-connect-works/
- Authentication layer built on top of OAuth 2.0. Required for enterprise SSO (Single Sign-On) integration with identity providers (Okta, Azure AD, Google Workspace). Supports federated login flows used by all enterprise OKR platforms. Standardises the ID token format (JWT).

**RFC 7643 & RFC 7644 — SCIM 2.0 (System for Cross-domain Identity Management)**
- URL: https://scim.cloud/
- REST-based protocol for automating user provisioning and de-provisioning. OKR platforms use SCIM to synchronise users, teams, and organisational hierarchies from HRIS/IdP systems (Workday, BambooHR, Okta) into the OKR tool. Profit.co explicitly exposes a SCIM Key alongside the API Key in its settings.

**SAML 2.0 — Security Assertion Markup Language**
- URL: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- XML-based SSO federation standard required for enterprise identity management. Complements OAuth 2.0/OIDC, particularly in legacy enterprise environments. All major enterprise OKR tools (Lattice, Workboard, Quantive) support SAML 2.0 as an enterprise SSO option.

**RFC 5545 — iCalendar (Internet Calendaring and Scheduling Core Object Specification)**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Standard format for representing calendar events, to-dos, and milestones. Relevant for OKR platforms that support deadline scheduling, review cadences (weekly check-ins, quarterly planning ceremonies), and calendar-based goal milestones that users may wish to export to Google Calendar, Outlook, or Apple Calendar.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- The foundational HTTP specification underpinning all REST API interactions used by OKR platforms. Defines standard methods (GET, POST, PUT, PATCH, DELETE), status codes, and content negotiation.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.0**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The de facto standard for describing REST APIs. OpenAPI 3.1 is a full superset of JSON Schema, enabling machine-readable API documentation. OKR platforms should expose their APIs using OpenAPI to enable SDK generation, Postman collections, and integration discovery. Profit.co provides a Postman public workspace.

**JSON Schema (draft-07 / 2020-12)**
- URL: https://json-schema.org/
- Standard for describing and validating the shape of JSON data. Directly relevant for defining the OKR data model: Objective, Key Result, Initiative, Check-in, and Progress structures. OpenAPI 3.1 uses JSON Schema natively for request/response body definitions.

**GraphQL (June 2018 specification)**
- URL: https://spec.graphql.org/
- Perdoo uses GraphQL as the standard for its API, enabling clients to query exactly the data they need (e.g., fetch all key results for a team filtered by cycle). GraphQL is increasingly relevant for OKR platforms that expose complex hierarchical data (Company → Team → Individual OKRs).

**Webhook conventions (informal industry standard)**
- Slack and Microsoft Teams both publish incoming webhook standards that OKR platforms use to deliver goal-update notifications. Webhook payloads should follow the JSON-based incoming webhook format documented by each platform:
  - Slack: https://api.slack.com/incoming-webhooks
  - Microsoft Teams: https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook

---

### Security & Authentication Standards

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- De facto checklist for securing REST and GraphQL APIs. Directly applicable to OKR platform APIs that handle sensitive performance data (individual goal attainment, ratings). Key risks include Broken Object Level Authorisation (ensuring a user cannot read another user's private OKRs) and Excessive Data Exposure.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- Employee performance data (OKR attainment, check-ins, ratings) is personal data under GDPR. Platforms operating in the EU must comply: lawful basis for processing, data minimisation, the right to object to automated decision-making, and Data Protection Impact Assessments (DPIAs) for automated performance scoring. Non-compliance carries fines up to €20M or 4% of global revenue.

**CCPA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- Analogous to GDPR for California residents. Relevant for US-headquartered OKR platforms storing employee goal and performance data.

**JWT — JSON Web Tokens (RFC 7519)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard compact token format used in OAuth 2.0 and OpenID Connect flows. All major OKR platform APIs use JWT-based Bearer tokens for API authentication.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is directly relevant to an AI-native OKR platform. An MCP server exposing OKR data would allow AI agents and LLM-powered assistants to:
- Retrieve current OKRs for a user or team
- Update key result progress values
- Query historical check-in data for trend analysis
- Draft new OKRs based on strategic context

- **MCP Specification**: https://modelcontextprotocol.io/specification
- **MCP Python SDK**: https://github.com/modelcontextprotocol/python-sdk
- **MCP TypeScript SDK**: https://github.com/modelcontextprotocol/typescript-sdk

---

## Similar Products — Developer Documentation & APIs

### Lattice

- **Description:** HR-integrated OKR and performance management SaaS platform used by mid-market and enterprise teams. Combines OKRs with performance reviews, engagement surveys, and compensation.
- **API Documentation:** https://developers.lattice.com/reference/introduction
- **API Base URL:** `https://{LATTICE_ENDPOINT}/api/v1`
- **SDKs/Libraries:** No official SDK; Python source via dltHub (https://dlthub.com/context/source/lattice); unified connector via Unified.to (https://unified.to/integrations/lattice)
- **Standards:** REST/JSON, OAuth 2.0 client_credentials flow
- **Authentication:** `POST /api/v1/oauth/token` with `client_id`, `client_secret`, `grant_type=client_credentials`; subsequent requests use `Authorization: Bearer <access_token>`

---

### Quantive Results (formerly Gtmhub)

- **Description:** Enterprise OKR platform with a fully exposed RESTful API covering all OKR entities. API-first design enables automated key result updates from external data sources.
- **API Documentation (EU):** https://app.quantive.com/swagger/index.html
- **API Documentation (US):** https://us.quantive.com/swagger/index.html
- **Quickstart Guide:** https://help.quantive.com/en/articles/1273148-gtmhub-api-quickstart
- **Best Practices:** https://help.quantive.com/en/articles/5521198-best-practices-using-the-api
- **Extensibility SDK:** https://github.com/gtmhub/extensibility-sdk
- **Standards:** REST/JSON, Swagger/OpenAPI
- **Authentication:** API token passed as `Bearer` token; token scoped to account by account ID

---

### Profit.co

- **Description:** AI-powered OKR platform with built-in task management and strategy execution. Exposes a comprehensive API and provides a public Postman workspace.
- **API Documentation:** https://www.profit.dev/
- **Postman Workspace:** https://www.postman.com/profitokrs/workspace/profit-co-s-public-workspace/documentation/
- **Standards:** REST/JSON; SCIM 2.0 for user provisioning
- **Authentication:** API Key + Access Key (found in Settings → Security → API Access); SCIM Key also exposed for identity provisioning

---

### Workboard

- **Description:** Enterprise OKR and strategy-execution platform with a RESTful API for writing connectors to external data sources. Supports major enterprise integrations (Salesforce, Jira, Azure DevOps, PowerBI, Slack, Teams).
- **API Documentation:** https://www.workboard.com/developer
- **OAuth Flow:** https://www.workboard.com/developer/authentication-oauth.php
- **Atlassian Marketplace (OKRs for Jira):** https://marketplace.atlassian.com/apps/1221224/workboard-okrs-for-jira
- **Standards:** REST/JSON; OAuth 2.0 Authorization Code Grant for multi-user apps
- **Authentication:** OAuth 2.0 (Client ID + Client Hash); API token passed in `Authorization` header over HTTPS

---

### Weekdone

- **Description:** OKR tracking platform with weekly reporting and team alignment dashboards. Exposes a RESTful API using OAuth 2.0 Authorization Code Grant.
- **API Documentation:** https://weekdone.com/developer
- **Standards:** REST/JSON; OAuth 2.0 (Authorization Code Grant only)
- **Authentication:** OAuth 2.0; output format is JSON

---

### Perdoo

- **Description:** OKR platform targeting mid-sized teams. API is GraphQL-based (not REST), enabling flexible queries for OKRs, Initiatives, and KPIs. Supports creating goals and generating custom dashboards.
- **API Documentation:** Available via Personal settings → Integrations within the Perdoo app; also surfaced at Peoplelogic.dev integration docs (https://docs.peoplelogic.dev/guides/getting-started-with-the-composable-talent-platform/universal-talent-api/performance-management/perdoo-okrs)
- **Data Export:** CSV and PDF export of users, teams, goals (OKRs, KPIs, Initiatives) — https://support.perdoo.com/en/articles/2630593-export-data
- **Standards:** GraphQL
- **Authentication:** API token (accessible in Perdoo settings)

---

### Tability

- **Description:** Lean OKR tracking platform focused on weekly check-ins and progress signalling. Exposes a REST API with a Plans endpoint for creating and reading goal structures.
- **API Documentation:** https://guides.tability.io/api/tability-api/api-reference/plans
- **Standards:** REST/JSON
- **Authentication:** API key (details in platform settings)

---

### Asana (Goals API)

- **Description:** Asana's Goals API is part of the broader Asana REST API and supports creating, reading, updating, and deleting goal objects. Webhooks can be registered to receive real-time events when goal properties change.
- **API Documentation:** https://developers.asana.com/reference/rest-api-reference
- **Goals Endpoint Reference:** https://developers.asana.com/reference/goals
- **Webhooks Guide:** https://developers.asana.com/docs/webhooks-guide
- **Create Webhook Endpoint:** https://developers.asana.com/reference/createwebhook
- **Standards:** REST/JSON; OpenAPI-described
- **Authentication:** Personal Access Token (PAT) or OAuth 2.0; webhooks require `webhooks:write` scope. Goal objects use a `gid` (globally unique identifier) and `resource_type: "goal"`.

---

### Microsoft Viva Goals (via Microsoft Graph)

- **Description:** Microsoft's OKR module within the Viva employee experience platform. Goals data is accessible via Microsoft Graph (beta). Supports 45+ integrations via public APIs, all using OAuth 2.0.
- **API Documentation:** https://developer.microsoft.com/en-us/viva
- **Microsoft Graph Goals Resource:** https://github.com/microsoftgraph/microsoft-graph-docs-contrib/blob/main/api-reference/beta/resources/goals.md
- **Integration Admin Docs:** https://learn.microsoft.com/en-us/viva/goals/vg-integrations-administration-overview
- **Standards:** REST/JSON; Microsoft Graph API (OpenAPI-described); OAuth 2.0
- **Authentication:** OAuth 2.0 (oAuth2-based); falls back to API token or username/password for integrations that do not support OAuth

---

## Notes

**No universal OKR data interchange standard exists.** The industry lacks a shared open specification for OKR data models (analogous to iCalendar for scheduling or FHIR for health records). Each platform defines its own schema for Objectives, Key Results, Cycles, and Check-ins. This is a meaningful opportunity for an open-source AI-native platform to publish a reference JSON Schema / OpenAPI spec that could become the de facto standard.

**GraphQL vs REST split.** Most OKR platforms use REST (Quantive, Workboard, Lattice, Tability, Weekdone, Asana), while Perdoo uses GraphQL. Given the hierarchical nature of OKR data (Company → Department → Team → Individual), GraphQL's ability to traverse nested relationships in a single query is architecturally attractive.

**SCIM is the enterprise gate.** Enterprises will not adopt an OKR platform without SCIM support for automated user/team provisioning from their HRIS (Workday, BambooHR) or IdP (Okta, Azure AD). This must be an early architectural decision.

**Webhook-based progress automation.** The key differentiator among advanced OKR platforms is the ability to auto-update key result values by pulling data from external systems (Jira tickets closed, Salesforce revenue closed, GitHub PRs merged). Standardising a webhook/event model early — following the patterns of Slack and Teams incoming webhooks — is essential.

**AI and MCP.** An MCP server exposing OKR read/write operations would be a meaningful differentiator, allowing LLM-powered assistants (Claude, Copilot) to read team OKRs, surface at-risk key results, and draft weekly check-in updates on behalf of users.
