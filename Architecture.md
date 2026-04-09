# Architecture — sentinel-mssp-playbook

> This document is the master blueprint for the sentinel-mssp-playbook program. It explains what we are building, why each piece exists, how everything connects, and where the program is going. Read this before working in any individual folder or document. Everything else in this repo is a chapter — this document is the book.
>
> For full program scope, file status, session notes, and ideas parking lot see `PROGRAM-MAP.md`.
>
> **Last Updated:** April 2026

---

## What We Are Building and Why

We manage Microsoft Sentinel for ten co-managed MSSP clients via Azure Lighthouse. Every client environment is different — different licenses, different data sources, different integrations, different detection coverage. Managing ten environments without a standardized system means ten different ways of doing everything, ten different levels of quality, and no way to scale.

This program exists to solve that problem. It is a standardized operating system for managing Sentinel environments at scale — one set of processes, one set of standards, one set of tools that works across every client.

The mailbomb incident that started this program is the clearest illustration of why it matters. A client received a mailbomb attack. Defender XDR caught it and created an incident. But Sentinel never saw it because the XDR connector was not configured. The client asked why we did not catch it. We had no good answer. That is what happens without a system.

This program ensures that never happens again — for any client, for any data source, for any detection.

---

## The System in One Picture

```
CLIENTS AND ENGINEERS
        ↓ view and interact via
INTERACTION LAYER
Workbook — Visualization and client-facing dashboards
Portal — Future engineer management interface
        ↓ reads from
FUNCTION LAYER
GetEnrichedInventory()     — enriches data sources with live health and detection info
GetDetectionCoverage()     — joins detections with health and table availability
GetIntegrationHealth()     — joins integrations with credential expiry checks
GetClientProfile()         — returns full client context for any query
        ↓ joins
DATA LAYER — Watchlists
DataSourceInventory        ← Platform team owns
DetectionCatalog           ← Detection team owns
IntegrationRegistry        ← Platform team owns
ClientProfile              ← Account team owns
WatchlistRegistry          ← Platform team owns
        ↑ populated and updated by
AUTOMATION LAYER
Snapshot Logic App         — writes DataSourceAudit_CL weekly
Python Script              — detection team populates DetectionCatalog
SentinelHealth feed        — rule health monitoring
API expiry checker         — credential monitoring
New source detector        — flags unknown tables in workspace
```

Each layer has a clear owner, a clear purpose, and a clear interface. No layer reaches past the one adjacent to it. This is what keeps the system clean, maintainable, and extensible as it grows.

---

## The Repository Structure

```
sentinel-mssp-playbook/
├── README.md                              ← Vision — what this is and why it exists
├── ARCHITECTURE.md                        ← This document — the master blueprint
├── PROGRAM-MAP.md                         ← Full scope, file status, session notes
│
├── 01-baseline/
│   ├── architect-checklist.md             ← Universal baseline checklist
│   └── license-entitlement-review.md      ← License tier and capability impact
│
├── 02-data-sources/
│   ├── data-source-audit-runbook.md       ← Complete audit process
│   └── workbook-vision-and-design.md      ← Workbook tab design and client experience
│
├── 03-detection-library/
│   ├── catalog-index.md                   ← Master detection table
│   ├── detection-template.md              ← Template for new detection files
│   └── detections/
│       └── AB00000.md                     ← One file per detection
│
├── 04-capabilities/
│   ├── ueba.md                            ← UEBA dependency chain
│   ├── threat-feed.md                     ← Threat feed dependency chain
│   ├── fusion-ml.md                       ← Fusion ML requirements
│   └── soc-optimization.md               ← SOC optimization configuration
│
├── 05-xdr-migration/
│   ├── pre-migration-checklist.md
│   ├── post-migration-checklist.md
│   └── schema-changes.md
│
├── 06-watchlist-management/
│   ├── watchlist-schemas.md               ← All watchlist field definitions
│   ├── kql-functions.md                   ← GetEnrichedInventory and other functions
│   └── watchlist-health-queries.md        ← Staleness and coverage gap queries
│
├── 07-integrations/
│   ├── api-key-inventory-schema.md        ← IntegrationRegistry watchlist schema
│   └── credential-health-queries.md       ← Expiry and dependency queries
│
├── 08-silent-detections/
│   ├── silent-detection-standards.md      ← Tier framework and methodology
│   ├── library/                           ← Pre-canned silent detection KQL files
│   └── heartbeat-detection-guide.md       ← AMA heartbeat detection configuration
│
├── 09-detection-catalog/
│   ├── detection-catalog-spec.md          ← DetectionCatalog watchlist specification
│   └── python-script-guide.md            ← How detection team populates the catalog
│
├── 10-automation-pipeline/
│   ├── snapshot-logic-app.md              ← DataSourceAudit_CL snapshot design
│   ├── teams-notification-playbook.md     ← SOC Teams alert design
│   └── change-detection-rules.md         ← Analytics rules for status changes
│
├── 11-access-and-permissions/
│   ├── lighthouse-delegation-guide.md
│   └── graph-api-exploration.md
│
└── resources/
    ├── scratch.md                         ← Working scratchpad — not a finished document
    ├── taxonomy.md                        ← Nine-category workspace taxonomy
    ├── kql-snippets.md                    ← Reusable KQL patterns
    ├── glossary.md                        ← Terms and concepts
    └── table-reference/
        ├── README.md                      ← Entry template and index
        └── SecurityEvent.md               ← One file per documented table
```

---

## The Workspace Taxonomy

Before working in any layer understand that a Sentinel workspace contains nine distinct types of components. Using consistent terminology across all documents is what makes this program navigable at scale.

| Category | What It Is | Examples |
|---|---|---|
| **Log Sources** | Specific originators writing data to a table. One table can have many log sources. | Palo Alto PA-3200 → CommonSecurityLog, DC01 → Syslog |
| **Tables** | Log Analytics destinations where data lands. The ground truth layer. | SignInLogs, CommonSecurityLog, BehaviorAnalytics |
| **Watchlists** | Persistent reference data queryable in KQL at any time. The metadata backbone. | DataSourceInventory, DetectionCatalog, IntegrationRegistry |
| **Capabilities** | Features that depend on multiple sources working together. Break one dependency and the capability degrades silently. | UEBA, Threat Feed, Fusion ML, SOC Optimization |
| **Detections** | Analytics rules that consume table data and produce alerts | AB-series rules, Content Hub rules |
| **Silent Detections** | Detections that monitor whether a log source is actively sending data. Every tracked log source must have one at Tier 1 or 2. | AB##### - SLD - SignInLogs Silence |
| **Automation** | Logic Apps and playbooks that respond to incidents and run scheduled tasks | Snapshot Logic App, ServiceNow ticket creation |
| **Integrations** | API keys, service principals, OAuth tokens that connect services. Expire silently. | Threat feed API key, Zoom API key |
| **Workbooks** | Interactive live dashboards that visualize workspace data | Client Security Operations Workbook |

---

## The Data Layer — Watchlists

Watchlists are the backbone of the entire system. They store the human-curated metadata that cannot be derived from live workspace queries alone. Every function, every workbook tab, and every automation references one or more watchlists.

The core principle: **watchlists own persistent metadata. Everything else reads from them.**

### DataSourceInventory
**Owner:** Platform team
**Purpose:** The single source of truth for every tracked log source across a client environment. One row per log source. Powers the workbook, the silent detection management system, and the snapshot pipeline.

**Key fields:**
```
Table, Status, LastSeen, LogSource, Origin, Transport,
Category, Purpose, SilentDet, Detections,
DateAdded, Vetted, Notes
```

**Status values:** Active / Inactive / Review / No Data / Missing / Flag / Decom
**Vetted values:** Yes / No — has this source been fully investigated and signed off
**DateAdded:** Raw datetime — when this source was first documented

**How it gets populated:** Data source audit runbook — `02-data-sources/data-source-audit-runbook.md`

**How it stays current:** Snapshot Logic App detects new tables and sets Status = Flag. Engineer investigates, classifies, sets Vetted = Yes.

---

### DetectionCatalog
**Owner:** Detection team
**Purpose:** Master catalog of all AB-series detections. Documents which tables each detection queries, which watchlists it references, and where it routes results. Powers the GetEnrichedInventory function join that auto-populates detection dependencies in the DataSourceInventory.

**Key fields:**
```
DetectionID, DetectionName, DetectionType, Tables,
Watchlists, Severity, Enabled, Pipeline,
LastModified, MITRETactic, MITRETechnique, Notes
```

**DetectionType values:** SLD — Silent Log Source Detection / Detection — standard analytic rule
**Pipeline values:** Logic App name — e.g. LA-Sentinel-Stage / LA-Sentinel-Prod
**Tables field:** Comma separated exact table names. Must match DataSourceInventory Table values exactly for the join to work.

**How it gets populated:** Detection team Python script hits the Sentinel Analytics Rules API and Automation Rules API, parses table references from KQL query text, and outputs a CSV for watchlist upload.

**How it stays current:** Python script runs on a schedule. Watchlist is updated when detections are added, modified, or removed.

**The join with DataSourceInventory:**
```kql
// Simplified example of how DetectionCatalog enriches DataSourceInventory
let inventory = _GetWatchlist('DataSourceInventory');
let detections = _GetWatchlist('DetectionCatalog')
| mv-expand Tables = split(Tables, ",")
| extend Table = trim(" ", tostring(Tables))
| summarize
    Detections = make_set(DetectionID),
    HasSLD = countif(DetectionType == "SLD") > 0
    by Table;
inventory
| join kind=leftouter detections on Table
```

---

### IntegrationRegistry
**Owner:** Platform team
**Purpose:** Tracks every API key, service principal, OAuth token, and credential that keeps client integrations running. Drives Tab 6 of the workbook and triggers Teams notifications 30 days before expiry.

**Key fields:**
```
IntegrationName, ServiceName, CredentialType,
ExpiryDate, DaysUntilExpiry, Owner,
DependentSources, LastRotated, Notes
```

**How it gets populated:** Manually during client onboarding and updated when credentials are rotated.

**How it stays current:** GetIntegrationHealth() function calculates DaysUntilExpiry live from ExpiryDate. A scheduled analytics rule fires when any credential is within 30 days of expiry.

---

### ClientProfile
**Owner:** Account team
**Purpose:** Stores client-level metadata — license tier, subscriptions, key contacts, onboarding date, compliance requirements. Referenced by the workbook to adapt what is shown based on what the client has access to.

**Key fields:**
```
ClientName, TenantID, M365License, EntraIDLicense,
DefenderProducts, SentinelTier, Subscriptions,
ComplianceRequirements, OnboardingDate,
PrimaryContact, Notes
```

**How it gets populated:** License entitlement review — `01-baseline/license-entitlement-review.md`

---

### WatchlistRegistry
**Owner:** Platform team
**Purpose:** A watchlist of watchlists. Documents every watchlist that exists in the client workspace — name, owner, purpose, row count, last updated. Enables workbook health monitoring of the watchlist layer itself.

**Key fields:**
```
WatchlistName, Owner, Purpose, SearchKey,
RowCount, LastUpdated, Notes
```

---

## The Function Layer — KQL Functions

KQL functions sit between the watchlists and everything that consumes them. They handle joins, enrichment, and live data calculations so that workbook queries and detection queries stay simple and consistent.

**The core principle: one function per domain. Update the function once, every consumer gets the update automatically.**

### GetEnrichedInventory()
Joins DataSourceInventory against DetectionCatalog and live workspace health data. Returns one enriched row per log source with current status, days since last log, detection coverage, and SLD coverage.

Every workbook tab that displays data source information calls this function. No workbook query joins watchlists directly.

```kql
// GetEnrichedInventory() — conceptual structure
let inventory = _GetWatchlist('DataSourceInventory')
| extend LastSeen = todatetime(LastSeen);
let detections = _GetWatchlist('DetectionCatalog')
| mv-expand Tables = split(Tables, ",")
| extend Table = trim(" ", tostring(Tables))
| summarize
    Detections = make_set(DetectionID),
    SLDCoverage = iff(countif(DetectionType == "SLD") > 0, "Covered", "Missing")
    by Table;
let liveHealth = search *
| where TimeGenerated > ago(24h)
| summarize LastSeen_Live = max(TimeGenerated) by $table
| project Table = $table, LastSeen_Live;
inventory
| join kind=leftouter detections on Table
| join kind=leftouter liveHealth on Table
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen_Live)
| extend CurrentStatus = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
```

### GetDetectionCoverage()
Joins DetectionCatalog against SentinelHealth. Returns detection health status, last run time, and whether each detection is enabled and healthy. Powers Tab 3 Silent Detections and the detection health section of Tab 1 Summary.

### GetIntegrationHealth()
Reads IntegrationRegistry and calculates days until expiry for every credential. Returns status — Healthy / Expiring Soon / Expired. Powers Tab 6 Integrations.

### GetClientProfile()
Returns the full ClientProfile watchlist row for the current workspace. Used as a context layer in workbook queries that need to adapt based on client license tier.

---

## The Workbook System

The Client Security Operations Workbook is the operational engine of the program. It runs inside Sentinel against each client workspace in real time. Engineers use it daily. Clients see it in reviews.

**The workbook never owns data. It only visualizes it.**

Everything the workbook displays comes from either a live KQL query or a function call that joins watchlists with live data. Nothing is hardcoded in the workbook. If the underlying data changes the workbook reflects it automatically.

### Seven-Tab Structure

**Tab 1 — Summary**
The opening view. Rolled-up health in under ten seconds. X of Y metric cards for data sources, capabilities, silent detections, detections, integrations. Source group health by category.

**Tab 2 — Data Sources**
Full detail view. One row per log source from GetEnrichedInventory(). Status, health, origin, transport, category, purpose, detection coverage. Filterable by status, category, vetted status. The working engineer view.

**Tab 3 — Silent Detections**
X of Y healthy. Missing and unhealthy sources flagged. Tier breakdown — how many Tier 1 sources have coverage vs how many are missing. Direct link to create a new silent detection for any missing source.

**Tab 4 — Capabilities**
One card per capability. UEBA, Threat Feed, Fusion ML, SOC Optimization. Each card shows overall status derived from dependency chain health. Individual dependency status visible per card. License availability shown from ClientProfile.

**Tab 5 — Insights** *(Version 2 — requires snapshot pipeline)*
Historical view from DataSourceAudit_CL. Volume trends, status change history, new and missing sources over time, cost analysis. Not available until the snapshot Logic App is running.

**Tab 6 — Integrations**
One row per credential from GetIntegrationHealth(). Expiry date, days until expiry, dependent sources, status. Teams notification fires when any credential is within 30 days of expiry.

**Tab 7 — Detection Value**
30-day incident contribution by data source. Which sources are actively contributing to incidents. Which sources are ingesting data but producing zero detection hits — cost optimization signal. Cross-reference with ingestion volume from Usage table.

### Full workbook design
See `02-data-sources/workbook-vision-and-design.md` for the complete tab-by-tab design, client meeting narrative, and version roadmap.

---

## The Two-Team Model

The program is built around two teams with clearly separated responsibilities. Neither team can do the other's job and neither team needs to.

**Platform Team — owns the infrastructure**
- DataSourceInventory watchlist
- IntegrationRegistry watchlist
- WatchlistRegistry watchlist
- Silent detection management — all SLD-tagged detections
- Snapshot pipeline and automation
- Workbook build and maintenance
- Client onboarding and baseline

**Detection Team — owns the content**
- DetectionCatalog watchlist
- All AB-series detections that are not SLD-tagged
- MITRE mapping and coverage
- Python script for catalog population
- Detection health and tuning

**The join between teams happens at the function layer.** GetEnrichedInventory() joins DataSourceInventory against DetectionCatalog. Neither team needs to maintain what the other team owns. The function handles the relationship automatically.

**The critical contract between teams:**
The Tables field in DetectionCatalog must use exact table names matching DataSourceInventory. This is the join key. If it drifts the automation breaks. Both teams must agree on and maintain this standard.

---

## The Silent Detection System

Every tracked log source at Tier 1 or Tier 2 must have a silent detection. A silent detection is an analytics rule that fires when a log source stops sending data. It is not a security detection — it is an operational health check.

### The Tier Framework

| Tier | Level | Criteria | Alert Behavior |
|---|---|---|---|
| Tier 1 | Required | A detection depends on it / A capability depends on it / Part of incident pipeline / Compliance obligation | High severity — immediate Teams notification |
| Tier 2 | Recommended | High investigation value / Custom integration that breaks silently / Compliance reporting value | Medium severity — standard review window |
| Tier 3 | Workbook only | ASIM tables / Capability output tables / Operational meta tables | Workbook health dashboard only — no alert |
| Tier 4 | Document only | No detections / No compliance / No investigation value | No monitoring |

### Naming Convention
```
AB##### - SLD - [Source Description]
```
Examples: `AB00012 - SLD - SignInLogs Silence` / `AB00031 - SLD - Palo Alto Silence`

### Full documentation
See `08-silent-detections/silent-detection-standards.md`

---

## The Automation Pipeline

The automation layer keeps the system current without manual intervention. It is built in phases.

### Phase 1 — Snapshot Pipeline *(planned)*
A scheduled Logic App runs weekly and writes the current state of all workspace tables to a custom log table called `DataSourceAudit_CL`. This enables Tab 5 Insights — historical trends, status changes over time, volume analysis.

### Phase 2 — New Source Detection *(planned)*
The snapshot Logic App compares the current workspace table list against DataSourceInventory. Any table not in the watchlist gets a new row created with Status = Flag, Vetted = No, DateAdded = today. A Teams notification fires — "New log source detected — investigate and classify."

### Phase 3 — Silent Detection Coverage Verification *(planned)*
A scheduled analytics rule queries DataSourceInventory for all Tier 1 and Tier 2 sources and cross-references against SentinelHealth. Any Tier 1 source with no healthy silent detection creates a medium severity incident — "Missing silent detection — [Source]."

### Phase 4 — Credential Expiry Monitoring *(planned)*
A scheduled analytics rule reads IntegrationRegistry and fires 30 days before any credential expires. Teams notification to the responsible engineer. Ticket created in ServiceNow.

### Phase 5 — Multi-Tenant Content Distribution *(planned)*
Once the silent detection library is complete distribute updates to all client workspaces from one place via the Defender portal multi-tenant content distribution capability. New pre-canned detection deployed once — pushed to all ten clients automatically.

---

## The Program Layers

Do not skip ahead. A client is not ready for Layer 4 if Layer 1 is incomplete.

```
Layer 1 — Baseline and License Review
        ↓
Layer 2 — Data Source Audit and Watchlist
        ↓
Layer 3 — Detection Library and Coverage
        ↓
Layer 4 — Workbook and Client Dashboard
        ↓
Layer 5 — Automation and Scale
```

### Layer 1 — Universal Baseline
Every client starts here. Complete the license entitlement review first — it determines what should exist in the environment. Then work through the baseline checklist — connectors, diagnostic settings, UEBA, threat intelligence, XDR connector. Document everything in the license-entitlement-review.md for this client.

### Layer 2 — Data Source Audit
Run the data source audit runbook. Produce the DataSourceInventory watchlist. Upload to Sentinel. This is the backbone — everything else builds on it.

### Layer 3 — Detection Coverage
Platform team verifies silent detection coverage against the DataSourceInventory tier assignments. Detection team populates the DetectionCatalog. GetEnrichedInventory() function is written and tested. Detection gaps are identified and prioritized.

### Layer 4 — Workbook
Build the workbook tabs in order — Tab 1 Summary, Tab 2 Data Sources, Tab 3 Silent Detections, Tab 4 Capabilities, Tab 6 Integrations, Tab 7 Detection Value. Tab 5 Insights comes after the snapshot pipeline is running.

### Layer 5 — Automation
Build the snapshot pipeline. Build the new source detector. Build the credential expiry monitor. Build the silent detection coverage verifier. At this point the system largely manages itself.

---

## The Tool Selection Framework

Every engineering decision should use the right tool for the right job. This framework prevents over-engineering and drift.

| Tool | Best For | Not Good For | Drift Risk |
|---|---|---|---|
| Azure Resource Tags | Cost attribution, resource organization, Azure Policy | Anything in query results, anything that changes | Low |
| DCR Transformations | Enriching every record with static context — client name, environment | Anything that changes, anything derived | Medium |
| Watchlists | Persistent curated metadata, joining context against live data | High frequency updates, large datasets | Low if maintained |
| KQL Functions | Encapsulating join logic, enrichment, calculations | Storing data, anything persistent | None |
| Workbooks | Visualization, filtering, client dashboards | Storing data, owning logic | None |
| Analytics Rules | Detection, alerting, triggering automation | Storing reference data | Low |
| Logic Apps | Scheduled tasks, API calls, writing to custom tables | Storing data, visualization | Medium |
| Custom Log Tables | Time-series audit history, anything queryable over time | Reference data read frequently | Low |

**The core principles:**
1. Do not add a new system when an existing one solves the problem
2. Watchlists own persistent metadata — everything else reads from them
3. DCR transformations only for data that needs to be in the raw record
4. Azure tags only for Azure resource management
5. Workbooks never own data — they only visualize it
6. Custom log tables only for time-series audit history
7. KQL functions are the glue — they join and enrich, never store

---

## Key Design Principles

**Silent gaps are the enemy.** A broken detection fires incorrectly and you notice. A missing log source produces nothing and you do not notice — until a client does. Every system in this program is designed to surface silent gaps before they become client-visible failures.

**Audit by log source not by table.** A table can have many sources. One source going silent while others remain active leaves the table appearing healthy. The audit system tracks at the log source level.

**Tables do not lie. Connector widgets do.** A connector can show connected while data has silently stopped. Always query tables and log sources directly.

**Not every source needs a row.** A log source earns a watchlist row only if losing it would matter — a detection depends on it, a capability depends on it, a compliance obligation requires it, or it has meaningful investigation value.

**Not every source needs a silent detection.** Silent detections are tiered. Tier 1 is non-negotiable. Tier 4 is document only. Engineering effort goes where it creates the most protection.

**The function is the interface.** Workbook queries and detection queries never join watchlists directly. They call functions. Update the function once, every consumer gets the update.

**Two teams, one system.** Platform team owns the infrastructure. Detection team owns the content. The function layer is where they meet. Neither team needs to maintain what the other owns.

**Engineer for the destination.** Every decision made today should be compatible with where the system is going — interactive portal, multi-tenant distribution, automated coverage verification. Do not build things that will need to be thrown away.

**Use the right tool for the job.** Watchlists for metadata. Functions for logic. Workbooks for visualization. Custom tables for history. Do not use a hammer when you need a screwdriver.

---

## What Is Built vs What Is Planned

### Built and in the repo:
- Baseline checklist — `01-baseline/architect-checklist.md`
- License entitlement review — `01-baseline/license-entitlement-review.md`
- Data source audit runbook — `02-data-sources/data-source-audit-runbook.md`
- Workbook vision and design — `02-data-sources/workbook-vision-and-design.md`
- Silent detection standards — `08-silent-detections/silent-detection-standards.md`
- Table reference library — `resources/table-reference/`
- Investigation queries — `resources/scratch.md`

### Designed but not yet built:
- DetectionCatalog watchlist and spec — `09-detection-catalog/`
- KQL functions — `06-watchlist-management/kql-functions.md`
- IntegrationRegistry watchlist schema — `07-integrations/`
- ClientProfile watchlist schema
- WatchlistRegistry schema
- Workbook JSON and deployment

### Planned for future phases:
- Snapshot Logic App — `10-automation-pipeline/`
- New source detector automation
- Silent detection coverage verifier
- Credential expiry monitor
- Multi-tenant content distribution
- Interactive management portal

---

*This document is the master blueprint. Update it when the program architecture changes — not when individual documents change. Those changes are tracked in their own files and summarized in PROGRAM-MAP.md.*