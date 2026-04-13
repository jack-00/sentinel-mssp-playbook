# Architecture — sentinel-mssp-playbook

> This document is the master blueprint for the sentinel-mssp-playbook program. It explains what we are building, why each piece exists, how everything connects, and where the program is going. Read this before working in any individual folder or document. Everything else in this repo is a chapter — this document is the book.
>
> For full program scope, file status, session notes, and ideas parking lot see `program-map.md`.
>
> **Last Updated:** April 2026

---

## What We Are Building and Why

We manage Microsoft Sentinel for co-managed MSSP clients via Azure Lighthouse. Every client environment is different — different licenses, different data sources, different integrations, different detection coverage. Managing environments without a standardized system means different ways of doing everything, different levels of quality, and no way to scale.

This program exists to solve that problem. It is a standardized operating system for managing Sentinel environments at scale — one set of processes, one set of standards, one set of tools that works across every client.

The mailbomb incident that started this program is the clearest illustration of why it matters. A client received a mailbomb attack. Defender XDR caught it and created an incident. But Sentinel never saw it because the XDR connector was not configured. The client asked why we did not catch it. We had no good answer. That is what happens without a system.

This program ensures that never happens again — for any client, for any data source, for any detection.

---

## The System in One Picture

```
CLIENTS AND ENGINEERS
        ↓ view and interact via
INTERACTION LAYER
Audit Deck Workbook     — client facing, nine tabs, live data
MSSP Internal Workbook  — engineer facing, full technical detail
        ↓ reads from
FUNCTION LAYER
GetEnrichedInventory()      — complete unified view, one call
GetTableHealth()            — table level health from live workspace
GetSourceHealth()           — source level health via sub-functions
GetSharedTableHealth()      — unions all shared table sub-functions
GetAzureDiagnosticsHealth() — AzureDiagnostics sub-sources
GetCommonSecurityLogHealth() — CommonSecurityLog sub-sources
GetSyslogHealth()           — Syslog sub-sources
GetThreatIntelHealth()      — ThreatIntelIndicators sub-sources
GetASIMHealth()             — ASIM table sub-sources
GetDetectionCoverage()      — detection health and 30 day fire count
GetDependencyHealth()       — credential expiry status
GetClientContext()          — client profile for workbook headers
        ↓ joins watchlists against live workspace data
DATA LAYER — Five Watchlists
[mssname]-tables        ← Platform team — table pipe registry
[mssname]-sources       ← Platform team — all source detail and monitoring
[mssname]-detections    ← Detection team — detection catalog
[mssname]-dependencies  ← Platform team — credentials and expiry
[mssname]-client        ← Account team — client profile
        ↑ populated and maintained by
PROCESS LAYER
Data source audit runbook   — populates tables and sources watchlists
AI-assisted build process   — populates sources watchlist with functions
DetectionCatalog build      — populates detections watchlist
Silent detection standards  — governs SLS field in sources watchlist
```

Each layer has a clear owner, a clear purpose, and a clear interface. No layer reaches past the one adjacent to it.

---

## The Repository Structure

```
sentinel-mssp-playbook/
├── readme.md                                          ← What this repo is
├── vision.md                                          ← Why we exist and where we are going
├── story.md                                           ← The narrative — read this first
├── architecture.md                                    ← This document
├── program-map.md                                     ← Current state, progress, working notes
│
├── 01-baseline/
│   ├── architect-checklist.md                         ← Universal baseline checklist
│   └── license-entitlement-review.md                  ← License tier and capability impact
│
├── 02-data-sources/
│   ├── data-source-audit-runbook.md                   ← Complete audit process for tables watchlist
│   └── workbook-vision-and-design.md                  ← Workbook tab design and client experience
│
├── 04-capabilities/
│   └── threat-feed.md                                 ← Threat feed lifecycle and configuration
│
├── 06-watchlist-management/
│   ├── log-resource-requirements-guide.md             ← Requirements gathering process
│   └── sources-watchlist-build-guide.md               ← AI-assisted sources watchlist build process
│
├── 08-silent-detections/
│   └── silent-detection-standards.md                  ← Tier framework and SLS methodology
│
└── resources/
    ├── dev-notes.md                                   ← Engineering gotchas — read before coding
    ├── glossary.md                                    ← Terms and concepts
    ├── kql-snippets.md                                ← Reusable KQL patterns
    ├── scratch.md                                     ← Working scratchpad and breakout query library
    ├── taxonomy.md                                    ← Nine-category workspace taxonomy
    └── table-reference/
        ├── README.md                                  ← Table entry template and index
        └── SecurityEvent.md                           ← One file per documented table
```

---

## The Workspace Taxonomy

A Sentinel workspace contains nine distinct types of components. Using consistent terminology across all documents is what makes this program navigable at scale.

| Category | What It Is | Examples |
|---|---|---|
| **Log Sources** | Specific originators writing data to a table. One table can have many log sources. | Palo Alto PA-3200 → CommonSecurityLog, DC01 → Syslog |
| **Tables** | Log Analytics destinations where data lands. The ground truth layer. | SignInLogs, CommonSecurityLog, BehaviorAnalytics |
| **Watchlists** | Persistent reference data queryable in KQL at any time. The metadata backbone. | [mssname]-tables, [mssname]-sources, [mssname]-detections |
| **Capabilities** | Features that depend on multiple sources working together. Break one dependency and the capability degrades silently. | UEBA, Threat Feed, Fusion ML, SOC Optimization |
| **Detections** | Analytics rules that consume table data and produce alerts | OC-series rules, Content Hub rules |
| **Silent Detections** | Detections that monitor whether a log source is actively sending data. | OC##### - SLD - SignInLogs Silence |
| **Automation** | Logic Apps and playbooks that respond to incidents and run scheduled tasks | Snapshot Logic App, ServiceNow ticket creation |
| **Integrations** | API keys, service principals, OAuth tokens that connect services. Expire silently. | Threat feed API key, Zoom API key |
| **Workbooks** | Interactive live dashboards that visualize workspace data | Audit Deck Workbook, MSSP Internal Workbook |

---

## The Data Layer — Five Watchlists

Five watchlists hold every piece of curated knowledge about a client environment. Every piece of data we need lives in one of these five. No redundancy. No overlap.

**The rule for watchlist existence:** A watchlist is only justified if the data changes independently, multiple functions consume it, and it cannot always be derived from a live query.

**The naming convention:** `[mssname]-watchlistname` — all lowercase. The mssname prefix is the company abbreviation used consistently across all client watchlists.

---

### `[mssname]-tables` — Table Registry
**Owner:** Platform team
**Purpose:** One row per table. The backbone. Tells us what tables exist and catches new unknown tables appearing. The table is treated as a pipe — just a container. All monitoring and operational detail lives in sources.

**Fields:**
```
Table               — exact table name
Category            — security category
Details             — broad context about this table
Vetted              — Yes / No — has this table been investigated
Notes               — anything worth knowing at the table level
```

**How it works:** The fullouter join between live workspace data and this watchlist surfaces any table that exists in the workspace but has no matching watchlist row. Those are unknown new sources — flag for investigation.

---

### `[mssname]-sources` — Data Source Registry
**Owner:** Platform team
**Purpose:** One row per tracked data source. Every table gets at least one row here. Single source tables get one row. Shared tables get one row per sub-source. This is where all monitoring lives.

**Fields:**
```
Table               — links to [mssname]-tables
LogSource           — plain English source name
Category            — security category
Origin              — what generated this data
Transport           — how it gets into Sentinel
Description         — what this source is and what data it contains
Purpose             — one sentence — what does this detect or provide
SLA                 — True / False — service level agreement commitment
DataConnector       — connector name — reference only, no status tracking
DCRName             — Data Collection Rule name if applicable
DCEName             — Data Collection Endpoint name if applicable
FunctionName        — KQL sub-function name — null means none needed
SLS                 — AB##### or Missing — silent detection for this source
MonitoringFrequency — None / 1h / 5h / 15h / 24h / 48h
SentinelCapabilities — capabilities that depend on this source — UEBA / Fusion ML / Threat Intelligence / None
Notes               — technical notes, gotchas, environment context
```

**How MonitoringFrequency works:** The master source detection reads this field as a variable threshold. A source with MonitoringFrequency = 24h gets monitored for silence at the 24 hour mark. Change the threshold by updating the watchlist — never touch the detection KQL.

**SLA vs SLS distinction:**
- SLA = True/False — is this source part of our service level commitment
- SLS = AB##### — the specific silent detection assigned to this source

---

### `[mssname]-detections` — Detection Catalog
**Owner:** Detection team
**Purpose:** Every custom OC-series detection. What it does, what tables it queries, what watchlists it references, and who is responsible for it.

**Fields:**
```
RuleId              — OC##### unique identifier
AnalyticRule        — full rule name exactly as it appears in Sentinel
Table               — comma separated exact table names this rule queries
Watchlist           — comma separated watchlist names or None
AlertClass          — Security / Operational
Description         — plain English what does this catch or monitor
```

**AlertClass values:**
- Security — threat detections, TI matching, anomaly rules, behavioral detections
- Operational — silent log source detections, heartbeat monitoring, Logic App failures, data limit alerts

**The join with sources:** The GetEnrichedInventory function joins sources against detections on the Table field. Detection counts per source are derived automatically. No manual entry needed in the sources watchlist.

---

### `[mssname]-dependencies` — Dependency Registry
**Owner:** Platform team
**Purpose:** Every external credential, API key, client secret, and OAuth token that keeps data sources running. Tracks expiry dates, rotation owners, and which sources break when something expires.

**Fields:**
```
DependencyName      — plain English credential name
ServiceName         — what service this authenticates to
DependencyType      — API Key / Client Secret / OAuth Token / Certificate
LinkedSource        — comma separated source names that break if this expires
ExpiryDate          — raw datetime
RotationOwner       — who is responsible for rotating
RotationContact     — their contact information
LastRotated         — raw datetime
AppRegistrationName — Entra ID app registration name if applicable
Notes               — Logic App name, Lambda ARN, technical details
```

**How it works:** GetDependencyHealth() calculates DaysUntilExpiry live from ExpiryDate. A scheduled analytics rule fires 30 days before any credential expires. No stored calculated fields — everything volatile is calculated live.

---

### `[mssname]-client` — Client Profile
**Owner:** Account team
**Purpose:** Everything you need to know about the client that does not come from Sentinel. Feeds the audit deck splash screen, support tickets, and client communications.

**Fields:**
```
ClientName          — full legal name
ShortName           — what we call them day to day
Industry            — their vertical
Bio                 — two to three sentences about who they are
OnboardingDate      — when they became a client
TenantID            — Azure tenant ID
WorkspaceID         — Log Analytics workspace ID
WorkspaceName       — workspace name
SubscriptionID      — primary Azure subscription ID
SentinelURL         — direct link to Sentinel workspace
SharePointURL       — client folder in SharePoint
ServiceNowURL       — ServiceNow instance or queue
PrimaryContact      — main security contact
EscalationContact   — who to call for critical incidents
TechnicalContact    — client IT contact
SLSDocument         — link to signed SLS agreement
ComplianceFrameworks — comma separated applicable frameworks
Notes               — anything else worth knowing
```

---

## The Function Layer — KQL Functions

KQL functions sit between the watchlists and everything that consumes them. They handle joins, enrichment, and live data calculations so that workbook queries stay simple and consistent.

**The core principle:** One function per domain. Update the function once, every consumer gets the update automatically. The workbook never joins watchlists directly — it always calls a function.

**The output contract for shared table sub-functions:**
Every sub-function must return exactly these four columns and no others:
```
Table, LogSource, LastSeen, Status
```
This standardized output is what makes the master function scalable — add a new sub-function, add it to the union, done.

### Function Inventory

**Sub-functions — one per shared table:**
- `GetAzureDiagnosticsHealth()` — filters by ResourceType and Category
- `GetCommonSecurityLogHealth()` — filters by DeviceVendor and DeviceProduct
- `GetSyslogHealth()` — filters by HostName
- `GetThreatIntelHealth()` — filters by SourceSystem
- `GetASIMHealth()` — filters by EventVendor and EventProduct

**Master functions:**
- `GetSharedTableHealth()` — unions all sub-functions, joins against sources for metadata
- `GetTableHealth()` — table level health using search * against tables watchlist
- `GetSourceHealth()` — calls GetSharedTableHealth() plus handles single source tables
- `GetEnrichedInventory()` — the top level function, joins everything into one unified view

**Supporting functions:**
- `GetDetectionCoverage()` — joins detections against SentinelHealth and SecurityAlert
- `GetDependencyHealth()` — reads dependencies, calculates DaysUntilExpiry live
- `GetClientContext()` — returns client profile for workbook headers and splash screen

### How GetEnrichedInventory Works

```kql
// GetEnrichedInventory() — conceptual structure
// Joins all layers into one complete view
let sources = _GetWatchlist('[mssname]-sources');
let detections = _GetWatchlist('[mssname]-detections')
| mv-expand Tables = split(Table, ",")
| extend TableName = trim(" ", tostring(Tables))
| summarize
    DetectionCount = dcount(RuleId),
    DetectionList = make_set(AnalyticRule, 500)
    by TableName;
let liveHealth = GetSourceHealth();
sources
| join kind=leftouter detections on $left.Table == $right.TableName
| join kind=leftouter liveHealth on Table, LogSource
| extend DetectionCount = coalesce(DetectionCount, 0)
| extend HasCoverage = DetectionCount > 0
```

### How to Add a New Shared Table

1. Write a new sub-function following the four-column output contract
2. Add it to the union in GetSharedTableHealth()
3. Add rows to [mssname]-sources for each expected sub-source
4. Done — workbook picks it up automatically

---

## The Two Master Silent Detections

Two detections cover all monitoring. No hardcoded values. Both read entirely from watchlists.

**Master Source Monitor**
Reads [mssname]-sources where MonitoringFrequency != None. Calls sub-functions to get LastSeen per specific source. Fires per source that exceeds its MonitoringFrequency threshold. One detection covers every source in every environment.

**Custom SLS Detections**
For sources needing specific logic beyond the master. AB##### stored in the SLS field in sources. The master is the floor — custom SLS detections are the ceiling.

```kql
// Master Source Monitor — conceptual pattern
let sources = _GetWatchlist('[mssname]-sources')
| where MonitoringFrequency != "None"
| project Table, LogSource, MonitoringFrequency;
GetSourceHealth()
| join kind=inner sources on Table, LogSource
| extend ThresholdHours = toint(MonitoringFrequency)
| extend HoursSince = datetime_diff('hour', now(), LastSeen)
| where HoursSince > ThresholdHours
// fires per source exceeding its own threshold
```

**The key principle:** Adding a new source to monitor means updating [mssname]-sources. Zero detection changes needed. The detection reads configuration from the watchlist — never hardcodes values.

---

## The Audit Deck — Nine Tabs

The audit deck is client facing. Built from live data. Zero manual preparation.

| Tab | Title | What It Shows | Primary Data Source |
|---|---|---|---|
| 1 | Splash | Client overview, status, contacts | [mssname]-client |
| 2 | SLS and Capabilities | Service commitments, capability health | [mssname]-client, [mssname]-sources |
| 3 | Table Health | All tables, active vs inactive, new tables | [mssname]-tables, GetTableHealth() |
| 4 | Data Sources | All sources, health, detection coverage | [mssname]-sources, GetEnrichedInventory() |
| 5 | Dependencies | Credentials, expiry dates, what breaks | [mssname]-dependencies, GetDependencyHealth() |
| 6 | Detections | All detections, enabled status, 30 day count | [mssname]-detections, SentinelHealth |
| 7 | MITRE Coverage | Heatmap, gaps, available detections | [mssname]-detections, MITRE workbook |
| 8 | Ingestion and Cost | What it costs, coverage gaps, optimization | Usage table, [mssname]-sources |
| 9 | Opportunities | Open items, recommendations, client discussion | Manual input per review |

**Two modes — same workbook:**
Client mode shows clean visualizations and plain English summaries. Internal mode shows full technical detail. Same functions, different field projections in the workbook queries. Every function returns all fields — the workbook query controls what is shown.

---

## The SharePoint Structure

SharePoint is the operational home for all company and client work. Five root folders each with a clear purpose.

```
00-Company/     — agreements, policies, compliance
01-Program/     — standards, templates, processes, functions, detections, AI prompts
02-Clients/     — one folder per client, identical structure
03-Knowledge-Base/ — internal wiki, table references, compliance, verticals
04-Internal/    — team operations, training, research
05-References/  — vendor contacts, Microsoft support, licensing
```

Full structure and governance rules documented in SharePoint at `01-Program/Standards/sharepoint-governance.md`.

**Global vs client separation:**
- Applies to all clients → lives in 01-Program
- Specific to one client → lives in 02-Clients/[ClientName]
- Never mix them

---

## Key Design Principles

**Watchlists are configuration. Functions are logic. The workbook is visualization.**
Nothing crosses those boundaries. A watchlist never contains logic. A function never contains hardcoded values that belong in a watchlist. The workbook never calculates anything — it only displays what functions return.

**Code never changes for configuration updates.**
Adding a new source to monitor means updating the watchlist. Changing a threshold means updating the watchlist. The detection and function code never changes for configuration updates.

**The logs tell the truth. Connectors lie.**
A connector can show connected while data has silently stopped. Always query tables and log sources directly. The DataConnector field in sources is reference only — no status tracking.

**Silent gaps are the enemy.**
A broken detection fires incorrectly and you notice. A missing log source produces nothing and you do not notice — until a client does. Every system in this program is designed to surface silent gaps before they become client-visible failures.

**SLA is a commitment. SLS is the detection.**
SLA = True/False — is this source part of our service level commitment. SLS = AB##### — the specific silent detection that monitors it. Two different things with two different fields.

**Repeatable, standardized, scalable.**
Every solution must pass all three tests. Repeatable — any engineer can follow it and get the same result. Standardized — follows the same pattern as everything else. Scalable — adding more clients or sources does not require rebuilding.

**Engineer for the destination.**
Every decision made today should be compatible with where the system is going — repository-based management, automated deployment, AI-assisted review.

---

## What Is Built vs What Is Planned

### Built and current:
- Baseline checklist — `01-baseline/architect-checklist.md`
- License entitlement review — `01-baseline/license-entitlement-review.md`
- Data source audit runbook — `02-data-sources/data-source-audit-runbook.md`
- Threat feed guide — `04-capabilities/threat-feed.md`
- Sources watchlist build guide — `06-watchlist-management/sources-watchlist-build-guide.md`
- Silent detection standards — `08-silent-detections/silent-detection-standards.md`
- Table reference library — `resources/table-reference/`
- Dev notes — `resources/dev-notes.md`

### In progress — first client environment:
- [mssname]-detections watchlist — uploaded
- [mssname]-tables watchlist — in progress
- [mssname]-sources watchlist — building via AI process

### Planned — next steps:
- KQL sub-functions — one per shared table
- GetSharedTableHealth() and GetEnrichedInventory()
- [mssname]-dependencies watchlist
- [mssname]-client watchlist
- Audit deck workbook build
- MSSP internal workbook build

### Future phases:
- Snapshot Logic App — DataSourceAudit_CL for historical trending
- New source detector automation
- Credential expiry monitor
- Multi-tenant content distribution
- Repository-based deployment pipeline
- Client onboarding automation

---

*This document is the master blueprint. Update it when the program architecture changes — not when individual documents change. Those changes are tracked in their own files and summarized in program-map.md.*