# Program Vision — Sentinel MSSP Operating System

> **Purpose:** This document captures the full vision for what this program is building and why. It is the north star. Every technical decision, every watchlist field, every workbook tab, every automation exists to serve this vision. When you are unsure why something is being built read this document first.
>
> **Last Updated:** April 2026

---

## What We Are Actually Building

This is not a workbook project. This is not a detection project. This is not a reporting project.

This is a **managed security program operating system** — a complete, standardized, automated system for delivering world-class Sentinel MSSP services across all clients simultaneously. When it is complete an engineer can run a single command and know exactly where every client environment stands, what is working, what is broken, what is missing, and what the next move is. No tribal knowledge required. No manual preparation. No guesswork.

The audit deck is the output — the thing you show the client. But the operating system is the foundation that makes the audit deck possible, accurate, and effortless to produce.

---

## The Problem We Are Solving

We are new to this. We do not have a documented baseline of what good looks like. We do not have standardized systems. We are reactive. We are manually creating audit content before every client review. We do not have confidence that we know what is actually deployed, what is working, and what the client is entitled to.

This creates real problems:

- We cannot scale — each client requires custom manual effort
- We cannot be confident — we are not sure if we are meeting our commitments
- We cannot innovate — we spend all our time maintaining instead of improving
- We cannot grow — onboarding a new client means reinventing everything
- We are reactive — we find out about problems when clients do

**The goal of this program is to eliminate all of those problems permanently.**

---

## The Moment Everything Clicks

Imagine this scenario.

A new client signs with your MSSP. Within 48 hours they receive a clean guided spreadsheet. It has every field they need to fill in — their data sources, their critical assets, their key contacts, their compliance requirements. Dropdowns for approved values. Instructions built in. No technical knowledge required.

They fill it in. You review it, add the technical fields — connector names, DCR references, detection assignments. You upload five watchlists to their Sentinel workspace.

The workbook opens.

Every data source they documented is now visible with a live health status. Every detection is mapped to the sources it depends on. Every SLA commitment is tracked and measurable. Every API key and credential has an expiry date and an owner. The MITRE coverage map shows exactly where they are protected and where the gaps are.

The onboarding progress tab shows 73% complete. You can see exactly what is missing — three sources need silent detections, two connectors need to be configured, one capability needs a dependency resolved. No manual checklist. No CSM spreadsheet. The data tells the truth.

Two weeks later everything is green. Onboarding is complete. Not because someone ticked a box — because the system says so.

A quarter later you sit down with the client for their first audit review. You open the workbook. Everything is live and current. You did not prepare anything. You did not pull any reports. You did not ask anyone for status updates. You just open it and start talking.

The client sees their data sources, their detection coverage, their MITRE map, their costs, their dependencies. They see that you caught three issues before they did and resolved them. They see that their Salesforce API key expires in 23 days and you already have a rotation scheduled. They see that the detection covering their Azure Firewall fired six times last month and all six were legitimate threats that were triaged.

They are not asking if you are doing your job. They are asking what is next.

That is what this program builds. Not a workbook. Not a runbook. Not a set of watchlists. A system that makes excellence the default — for every client, every engineer, every review, every time.

---

## The Three Phases

### Phase 1 — Baseline and Standard

Define exactly what a good Sentinel environment looks like. Document every decision and why it was made. Create a repeatable, standardized baseline that every client environment is measured against.

**What this means:**
- A complete documented baseline checklist — every connector, every diagnostic setting, every capability, every optimization that should be enabled out of the box
- The why behind every decision — not just what to do but why it matters
- A standard we can defend to a client, to an auditor, to a new team member
- A client environment that is tight, optimized, and ready as a platform for data sources and detections

**What good looks like:**
When a new client is onboarded we run the baseline process and within a defined time window their environment meets every standard. No exceptions without documented justification. Any gap has an owner, a remediation plan, and a target date.

---

### Phase 2 — Visibility and Control

Build the systems that give us real-time visibility into every client environment simultaneously. Know exactly what is there, what is working, what is missing, and what has changed.

**What this means:**
- DataSourceInventory, DetectionCatalog, LogSourceRegistry — the watchlist backbone
- GetEnrichedInventory and supporting KQL functions
- Silent detection coverage for every tracked source
- The living workbook — open it and the truth is right there
- Automated change detection — something new appears, something goes inactive, something breaks — we know before the client does

**What good looks like:**
At any point in time an engineer can open the workbook for any client and immediately answer:
- What data sources are active and healthy
- What detections are running and what tables they depend on
- What capabilities are fully operational
- What has changed since last week
- What is our SLS commitment and are we meeting it
- What integrations are expiring and when

No preparation required. No manual data gathering. The system knows.

---

### Phase 3 — Value Delivery and Innovation

With the foundation solid and visibility complete we shift from reactive maintenance to proactive value delivery. This is where the real MSSP differentiation happens.

**What this means:**
- Audit deck that builds itself from live data — no manual preparation
- Detection coverage optimization — are we using all the data we are collecting
- Cost optimization — are we collecting data that nothing depends on
- MITRE coverage visibility — where are the gaps and what would close them
- Client-specific recommendations — based on their vertical, their risk profile, their business
- Repository-based management — push changes to all clients from one place
- AI-assisted triage and audit checks
- Automated onboarding for new clients

**What good looks like:**
The audit deck is generated in minutes not hours. The client meeting is a strategic conversation about improvement and innovation not a status update about what broke. The engineer is thinking about what is next not what is on fire.

---

## The Audit Deck Vision

The audit deck is the moment where all of this becomes visible to the client. It is not a report — it is a conversation. It is proof that we know what we are doing, that we are on top of everything, and that we are delivering real value.

### The Client Experience

A client sitting in an audit review should walk away knowing seven things:

**1. We are using everything they are paying for**
Every feature, every license entitlement, every capability that is available to them is either enabled and working or there is a documented decision about why it is not. Nothing is being left on the table. We have taken inventory of what they have access to and we have made deliberate choices.

**2. We know exactly what was agreed upon and we are delivering it**
Our SLS commitment is documented, visible, and measurable. Here is what we promised. Here is how we are doing against it. No ambiguity. No surprises. If we are meeting it we show the evidence. If we are not we have a plan.

**3. Their data is protected and monitoring is in place**
Every data source is tracked. Every tracked source has a health monitor. When something breaks we know before they do. Here is the silent detection coverage. Here is the last time something went inactive and how quickly we caught it. This is how the monitoring works and this is what we are responsible for versus what their team is responsible for.

**4. We are making full use of their data**
Every data source they are paying to collect has detections running against it. Here is the detection coverage per source. Here is what we are catching. Here is the MITRE coverage map — these are the tactics and techniques we can detect, these are the gaps, and here is what it would take to close them. We are not collecting data and ignoring it.

**5. Their money is being spent wisely**
Here is what is being ingested and what it costs. Here is what has detection coverage and what does not. Here are the optimization opportunities — sources that cost money but produce no detection value. Here are the noisy detections that might benefit from tuning. Here is what we recommend.

**6. We are evolving with the platform**
We are staying current with Microsoft's direction. Here is how we are leveraging the Defender portal migration. Here is what new capabilities we have evaluated or adopted. Here is what is on our roadmap. We are not standing still.

**7. We understand their business**
We know their vertical. We know what is most important to them. Our detection priorities and recommendations are aligned with their specific risk profile. We are not giving them a generic service — we are giving them a service that understands their business.

---

### The MSSP Experience

From our side the audit deck must be effortless to produce. Not because we are lazy but because manual preparation time is time stolen from actual security work.

**The audit deck is generated not created.**

Open the workbook. The data is live and current. Take the key metrics. Add the narrative. Done. Total preparation time — minutes not hours.

This is only possible because the operating system underneath is solid. The watchlists are current. The functions are running. The workbook is live. The data tells the truth automatically.

---

## The Audit Deck Structure

The audit deck is client facing. Every tab is designed to tell a story that a non-technical client can follow. Data that is for internal MSSP use only belongs in the separate MSSP internal workbook — not the audit deck.

The audit deck has two modes — the same workbook serves both. Client mode shows clean visualizations and plain English summaries. Internal mode shows the full technical detail underneath. Same data, different presentation.

---

### Tab 1 — Splash Screen
**Who is this client and are they fundamentally healthy right now?**

- Client name and logo
- One paragraph bio — who they are, their industry, why security matters to them
- Onboarding date — how long we have been their partner
- Current overall status — one indicator: Healthy / Needs Attention / Critical
- Data sources tracked — total count
- Active vs inactive breakdown
- Primary escalation contact
- Key links — Sentinel workspace, SharePoint, ServiceNow
- Last audit date and next audit date

Data source: `[mssname]-client` watchlist

---

### Tab 2 — SLS and Capabilities
**What did we commit to and are we delivering it?**

- Every item in our SLS commitment with current status
- Meeting / At Risk / Not Meeting — with evidence not just a status
- Capability health — UEBA, Threat Intelligence, Fusion ML, SOC Optimization
- Each capability shown as healthy or degraded with plain English explanation
- Any items not meeting SLS — root cause and remediation plan

Data source: `[mssname]-client`, `[mssname]-sources`, live capability queries

---

### Tab 3 — Table Health
**What data pipes do we have and are they working?**

- Every tracked table with current status
- Active, Inactive, Review, Missing
- Clean simple view — no technical detail
- New tables that appeared since last audit highlighted
- Tables that went inactive since last audit highlighted

Data source: `[mssname]-tables`, live workspace query via GetTableHealth()

---

### Tab 4 — Data Sources
**What is flowing through each pipe and is it all working?**

- Every tracked data source with current health
- Last seen, days since last log, status
- Associated detections per source — count only in main view, detail on click
- SLS sources clearly identified
- Sources with zero detection coverage flagged as gaps

Data source: `[mssname]-sources`, `[mssname]-detections`, live queries via GetSourceHealth()

---

### Tab 5 — Dependencies
**What external things could silently break our data collection?**

- Every API key, client secret, OAuth token, and credential tracked
- Days until expiry — traffic light green / amber / red
- What breaks if it expires — plain English list of affected sources
- Who owns the rotation and when it was last rotated
- Anything expiring within 30 days called out prominently

Data source: `[mssname]-dependencies` watchlist via GetDependencyHealth()

---

### Tab 6 — Detections
**What are we watching for and is it all working?**

- Every custom detection — what it does, enabled status, last fired
- 30 day alert count per detection
- Split by Security and Operational
- Detections that have not fired in 30 days flagged for review
- Detections that are enabled but their source is inactive flagged as blind

Data source: `[mssname]-detections`, SentinelHealth, SecurityAlert

---

### Tab 7 — MITRE Coverage
**How well are we protecting against known attack techniques?**

- MITRE ATT&CK heatmap — covered tactics and techniques
- Detections that are available but not yet enabled — what data would activate them
- Gap analysis — where coverage is thin and what would improve it
- Story of where we have invested and where we can grow

Data source: `[mssname]-detections`, Microsoft built-in MITRE workbook reference

---

### Tab 8 — Ingestion and Cost
**Is the investment being used wisely?**

- Ingestion cost breakdown by data source — last 30 days
- Sources with high ingestion and zero detection coverage flagged
- Month over month trend
- Optimization opportunities — what could be reduced without losing coverage
- Value delivered — incidents detected, response metrics

Data source: Usage table, `[mssname]-sources`, `[mssname]-detections`

---

### Tab 9 — Opportunities and Action Items
**What is on our radar and what do we want to discuss?**

- Open items from last audit — status update on each
- New recommendations this quarter — specific and actionable
- Detections available to unlock — what data they need
- Things to clean up or remove
- Client requests and items they have flagged
- Open conversation — this tab is the agenda for the discussion

Data source: `[mssname]-client` notes, manual input per review

---

## The Watchlist System

Five watchlists power the entire program. Every piece of data we need lives in one of these five. No redundancy. No overlap. Each one has a clear owner and a clear purpose.

**The rule for watchlist existence:** A watchlist is only justified if the data changes independently, multiple functions consume it, and it cannot always be derived from a live query.

---

### `[mssname]-tables` — Table Registry
**Owner:** Platform team
**Purpose:** One row per table. The backbone. Tells us what tables exist and whether any new ones have appeared. The table is treated as a pipe — just a container. No transport or ingestion details here.

| Field | What Goes Here |
|---|---|
| Table | Exact table name |
| Category | Security category |
| Description | Plain English — what kind of data lands here |
| DateAdded | When first documented |
| Vetted | Yes / No |
| MonitoringFrequency | None / 1h / 5h / 15h / 24h / 48h |
| Notes | Anything worth knowing at the table level |

---

### `[mssname]-sources` — Data Source Registry
**Owner:** Platform team
**Purpose:** One row per tracked data source within each table. This is where all the detail lives — transport, SLS, functions, connectors, DCR, DCE. Everything needed to monitor, troubleshoot, and report on each specific source.

| Field | What Goes Here |
|---|---|
| Table | Links to [mssname]-tables |
| LogSource | Plain English source name |
| Category | Security category |
| Origin | What generated this data |
| Transport | How it gets into Sentinel |
| Purpose | One sentence — what does this detect or provide |
| SLA | True / False — is this source part of the service level agreement |
| DataConnector | Sentinel connector name if applicable — reference only, no status tracking |
| DCRName | Data Collection Rule name if applicable |
| DCEName | Data Collection Endpoint name if applicable |
| FunctionName | KQL sub-function name — null means no function needed |
| SLS | AB##### or Missing — the silent log source detection assigned to this source |
| MonitoringFrequency | None / 1h / 5h / 15h / 24h / 48h — read by silent detection as variable |
| DateAdded | When first documented |
| Notes | Technical notes, gotchas, environment specific context |

---

### `[mssname]-detections` — Detection Catalog
**Owner:** Detection team
**Purpose:** Every custom OC-series detection. What it does, what tables it needs, what watchlists it references, and whether it is security or operational.

| Field | What Goes Here |
|---|---|
| RuleId | OC##### unique identifier |
| AnalyticRule | Full rule name |
| Table | Comma separated exact table names |
| Watchlist | Comma separated watchlist names or None |
| AlertClass | Security / Operational |
| Description | Plain English what does this catch or monitor |

---

### `[mssname]-dependencies` — Dependency Registry
**Owner:** Platform team
**Purpose:** Every external credential, API key, client secret, and OAuth token that keeps data sources running. Tracks expiry dates, rotation owners, and which sources break when something expires.

| Field | What Goes Here |
|---|---|
| DependencyName | Plain English credential name |
| ServiceName | What service this authenticates to |
| DependencyType | API Key / Client Secret / OAuth Token / Certificate |
| LinkedSource | Comma separated source names that break if this expires |
| ExpiryDate | Raw datetime |
| RotationOwner | Who is responsible for rotating |
| RotationContact | Their contact information |
| LastRotated | Raw datetime |
| AppRegistrationName | Entra ID app registration name if applicable |
| Notes | Logic App name, Lambda ARN, technical details |

---

### `[mssname]-client` — Client Profile
**Owner:** Account team
**Purpose:** Everything you need to know about the client that does not come from Sentinel. Feeds the splash screen, support tickets, and client communications.

| Field | What Goes Here |
|---|---|
| ClientName | Full legal name |
| ShortName | What we call them day to day |
| Industry | Their vertical |
| Bio | Two to three sentences about who they are |
| OnboardingDate | When they became a client |
| TenantID | Azure tenant ID |
| WorkspaceID | Log Analytics workspace ID |
| WorkspaceName | Workspace name |
| SubscriptionID | Primary Azure subscription ID |
| SentinelURL | Direct link to Sentinel workspace |
| SharePointURL | Client folder in SharePoint |
| ServiceNowURL | ServiceNow instance or queue |
| PrimaryContact | Main security contact |
| EscalationContact | Who to call for critical incidents |
| TechnicalContact | Client IT contact |
| SLSDocument | Link to signed SLS agreement |
| ComplianceFrameworks | Comma separated applicable frameworks |
| Notes | Anything else worth knowing |

This watchlist feeds the audit deck splash screen automatically.

---

## The Logic App and ServiceNow Pipeline

You mentioned the Logic App feeding ServiceNow has issues. This is a critical piece of the operating system — it is how incidents become tickets, how alerts become action, how the SIEM connects to the service management layer.

Before we rebuild it we need to understand what is broken and what the ideal state looks like. Some things to consider:

- Should every Sentinel incident create a ServiceNow ticket or only certain severities
- What enrichment should happen before the ticket is created — client name, affected assets, recommended actions
- Should the ticket include a link back to the Sentinel incident
- Should ticket updates sync back to Sentinel — comment on the incident when the ticket is resolved
- Is the current Logic App approach the right architecture or should we consider the native Sentinel ServiceNow connector

This deserves its own document and its own session. Flag it as a priority item in PROGRAM-MAP.

---

## The Onboarding Process

When this program is complete onboarding a new client should be a defined repeatable process with a clear timeline and a clear definition of done. The watchlist-driven variable approach makes this possible at scale.

**The onboarding engine concept:**

The same spreadsheet format used for ongoing operations becomes the onboarding intake form. A new client receives a pre-formatted guided spreadsheet with field definitions, dropdown values for approved fields, and instructions built in. They fill in what they know. We fill in the technical fields. Upload the watchlists. The workbook and detections light up automatically.

No manual status tracking. No CSM spreadsheets. The data tells the truth.

**The onboarding workbook tab shows:**
- Which sources are documented and vetted
- Which sources have functions configured
- Which sources have silent detections assigned
- Which SLA sources are active and healthy
- Overall onboarding completion percentage — derived live from watchlist state

Onboarding is complete when the workbook shows green across all required fields. That is the definition of done — not a checklist someone fills in manually.

**The onboarding process:**
1. Deploy the baseline configuration — all standard connectors, diagnostic settings, capabilities
2. Provide the client with the guided intake spreadsheet
3. Client fills in what they know — data sources, compliance requirements, key contacts, critical assets
4. We review, validate, and complete the technical fields — FunctionName, DCRName, SLS detection IDs
5. Upload all watchlists — client is now in the system
6. Detections deploy and immediately use watchlist values as variables
7. Workbook lights up — onboarding progress visible automatically
8. Verify silent detection coverage — every SLA source has monitoring
9. Deliver the first audit deck — baseline established, client knows where they stand
10. Schedule recurring audit cadence — quarterly reviews, monthly health checks

The onboarding timeline, checklist, and definition of done will be documented in a dedicated onboarding guide. This is the next major project after the current process documentation and function building is complete.

---

## The Repository Vision

The endgame is managing everything from a repository. Detection changes, watchlist updates, workbook deployments, baseline configurations — all version controlled, all reviewable, all deployable with a command.

This means:
- Detection KQL stored as files in the repo — deployed via Bicep or Terraform to all client workspaces
- Watchlist CSV files stored in the repo — deployed via the Sentinel API
- Workbook JSON stored in the repo — deployed via ARM template
- Baseline configuration stored as Bicep — deployed to new clients during onboarding
- All changes reviewed via pull request — no undocumented changes to client environments
- AI-assisted review — flag potential issues before deployment

This is the long game. It requires the operating system to be solid first. But every decision we make now should be compatible with this future state.

---

## Guiding Principles

**Baseline first.** You cannot improve what you have not defined. Define what good looks like before trying to be great.

**Systems over heroics.** A repeatable system beats a brilliant engineer every time. Build systems that work even when the best engineer is not in the room.

**Earn the trust, then innovate.** The client needs to trust that the basics are covered before they care about innovation. Get the baseline right. Get the SLS right. Then bring the innovation.

**The workbook does not lie.** Manual reports can be massaged. Live data cannot. Everything in the audit deck comes from live data or it does not belong in the audit deck.

**Engineer for the destination.** Every decision made today should be compatible with the repository vision. Do not build things that will need to be thrown away.

**Automate what scales, document what does not.** Not everything can be automated yet. But everything can be documented. A documented manual process is one step away from an automated one.

**One client's problem is every client's opportunity.** When you solve a problem for one client build the solution so it works for all of them. That is how you scale.

---

*This document is the north star for the sentinel-mssp-playbook program. It does not contain implementation details — those live in their respective documents. This document captures the vision, the why, and the destination. Update it when the vision evolves. Reference it when decisions get hard.*