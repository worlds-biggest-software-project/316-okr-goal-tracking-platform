# OKR & Goal Tracking Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for company-wide OKR management, alignment visualisation, and progress tracking.

The OKR & Goal Tracking Platform helps growth-stage companies and enterprises set, cascade, and track Objectives and Key Results across company, team, and individual levels. It is built for CEOs, People Ops teams, and engineering and product leaders who want outcome-based management without the cost and complexity of incumbent enterprise suites.

---

## Why OKR & Goal Tracking Platform?

- **Enterprise pricing locks out mid-market teams.** Lattice starts at $11/person/month, Asana Goals requires the $24.99/user/month Business plan, and Betterworks frequently exceeds $25/user/month with implementation fees.
- **Standalone OKR depth is being eroded by bundles.** Asana Goals and monday.com Goals offer shallow OKR functionality bolted onto work-OS platforms, while HR-first tools like Lattice optimise for review cycles rather than pure OKR workflows.
- **AI features are uneven across the market.** Only Profit.co and Betterworks offer strong AI-assisted authoring; Mooncamp's AI was still under development as of 2026; Weekdone has none.
- **Cross-team dependency detection is an open gap.** Most platforms visualise alignment but do not automatically detect when one team's off-track goal will impact another team's targets.
- **Narrative reporting is manual everywhere.** No incumbent generates board-ready executive summaries from OKR data, despite this being a recurring need for quarterly reviews and all-hands.

---

## Key Features

### Core OKR Management

- Cascading OKR hierarchies linking company, team, and individual goals
- Real-time progress tracking with visual dashboards and status indicators (on track, at risk, off track)
- Goal ownership and accountability assignment
- Weekly and monthly check-in workflow with confidence ratings
- Multi-user collaboration with commenting and feedback on goals

### Integrations & Automation

- Automated progress updates from Jira, GitHub, and Salesforce via native connectors
- REST API for third-party integrations
- Email and Slack notifications for check-ins and status changes
- Task linkage connecting daily work to strategic goals
- Multi-quarter planning with goal history and versioning

### Enterprise Readiness

- Role-based access control for team hierarchies and management levels
- SSO via SAML 2.0 and OAuth 2.0 for enterprise identity management
- SCIM provisioning for automated user management at 500+ user scale
- Export to PDF and Excel for reporting and all-hands presentations
- Advanced filtering and search across goal hierarchies

### AI-Assisted Workflows

- AI-assisted OKR authoring suggesting well-formed objectives based on team context
- Confidence scoring and risk flagging for goals trending toward failure
- AI-powered quarterly retrospectives summarising check-in trends into narrative insights
- Cross-team dependency detection identifying risks where one team's off-track goal will impact another
- Board deck generation with executive summary of company OKR progress

---

## AI-Native Advantage

Unlike incumbents that bolt AI onto rule-based systems, this platform treats AI as core infrastructure: connectors auto-update key results from CRMs, analytics, and engineering tools without manual check-ins; an OKR quality scorer coaches leaders on measurability and ambition during goal-writing; cascade visualisation includes risk scoring that surfaces likely downstream impact when lower-level goals slip; and quarterly narrative generation produces plain-language executive summaries from structured progress data.

---

## Tech Stack & Deployment

The platform is designed for both self-hosted and cloud deployment with a REST API as the integration backbone. It aligns with established enterprise standards: SAML 2.0 and OAuth 2.0 for SSO, SCIM for user provisioning from HR systems, and REST/GraphQL APIs for live data pulls from Jira, GitHub, Salesforce, and BI tools. The OKR (Doerr/Grove) framework is the foundational methodology, with optional Balanced Scorecard and SMART Goals support for adjacent goal-setting needs.

---

## Market Context

The global OKR software market was valued at approximately $1.5 billion in 2024 and is growing at roughly 16% CAGR. SMB tools cluster at $5–$9/user/month, mid-market platforms at $10–$15/user/month, and enterprise contracts (Betterworks, Lattice, Quantive) frequently exceed $25/user/month with implementation fees. Primary buyers are CEOs and strategy leaders at 50–500 employee growth-stage companies, HR and People Ops teams, and VPs of Product and Engineering tracking team-level outcomes.

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
