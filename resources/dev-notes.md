# Developer Notes — Things to Consider Before Writing Code

> **Purpose:** This file captures edge cases, oddities, architectural decisions, and gotchas that must be considered when writing KQL functions, workbook queries, and automation. Before writing any code read this file first. When something new comes up that could affect future code add it here immediately.
>
> **Who uses this:** Anyone writing KQL, workbook queries, Logic App expressions, or automation for this program.
>
> **Last Updated:** April 2026

---

## Final Watchlist Design — Locked

Five watchlists. No more, no less.

```
[mssname]-tables        — table pipe registry, backbone
[mssname]-sources       — all source detail, transport, SLS, functions
[mssname]-detections    — detection catalog, owned by detection team
[mssname]-dependencies  — credentials, API keys, expiry tracking
[mssname]-client        — client profile, contacts, links
```

### `[mssname]-tables` — FINAL
```
Table
Category
Details
Vetted
Notes
```
- Five fields only — lean pipe registry
- MonitoringFrequency removed — all monitoring handled in sources
- Every table gets at least one row in sources
- Vetted = No flags unknown new tables for investigation

### `[mssname]-sources` — FINAL
```
Table
LogSource
Category
Origin
Transport
Description
Purpose
SLA
DataConnector
DCRName
DCEName
FunctionName
SLS
MonitoringFrequency
SentinelCapabilities
Notes
```
- SLA = True/False — service level agreement commitment
- SLS = OC##### or Missing — the silent detection assigned to this source
- FunctionName null means no sub-function needed
- SentinelCapabilities — discovered during sources build process, not known upfront. Use to add CAP-series rows to detections watchlist after sources build is complete. Values: UEBA / Fusion ML / Threat Intelligence / SOC Optimization / None
- Vetted removed — if it is in sources it is already vetted by definition
- DateAdded removed — tracked in SharePoint instead
- Status, LastSeen, DaysSinceLastLog — calculated live by functions, never stored

### `[mssname]-detections` — FINAL
```
RuleId
AnalyticRule
Table
Watchlist
AlertClass
Description
```
- RuleId = OC##### — unique identifier, the join key
- Table field uses comma separated exact table names — must match sources exactly
- AlertClass = Security or Operational

### `[mssname]-dependencies` — FINAL
```
DependencyName
ServiceName
DependencyType
LinkedSource
ExpiryDate
RotationOwner
RotationContact
LastRotated
AppRegistrationName
Notes
```
- ExpiryDate stored as raw datetime
- DaysUntilExpiry calculated live by GetDependencyHealth() — never stored

### `[mssname]-client` — FINAL
```
ClientName, ShortName, Industry, Bio, OnboardingDate,
TenantID, WorkspaceID, WorkspaceName, SubscriptionID,
SentinelURL, SharePointURL, ServiceNowURL, PrimaryContact,
EscalationContact, TechnicalContact, SLSDocument,
ComplianceFrameworks, Notes
```

---

## SLA vs SLS — Critical Distinction

These are two different fields in [mssname]-sources and mean completely different things.

**SLA** = True / False — is this source part of our service level agreement commitment. Drives the SLS and Capabilities tab in the audit deck. Sources with SLA = True are prominently surfaced to the client.

**SLS** = AB##### or Missing — the specific silent detection rule assigned to monitor this source. Drives the silent detection coverage view.

Never confuse them. Never use one where the other is meant.

---

## MonitoringFrequency — Sources Only

MonitoringFrequency lives in `[mssname]-sources` only. It was removed from `[mssname]-tables` because every table has at least one row in sources — all monitoring is handled at the source level.

Values: None / 1h / 5h / 15h / 24h / 48h

The master source detection reads this field as a variable threshold at query time. Changing a threshold means updating the watchlist row — never editing the detection KQL.

```kql
// How the master detection uses MonitoringFrequency as a variable
let sources = _GetWatchlist('[mssname]-sources')
| where MonitoringFrequency != "None"
| project Table, LogSource, MonitoringFrequency;
GetSourceHealth()
| join kind=inner sources on Table, LogSource
| extend ThresholdHours = toint(MonitoringFrequency)
| extend HoursSince = datetime_diff('hour', now(), LastSeen)
| where HoursSince > ThresholdHours
```

Adding a new source to monitor = add a row to [mssname]-sources with MonitoringFrequency set. Zero detection changes needed.

---

## Two Master Silent Detections

**Master Source Monitor**
- Reads [mssname]-sources where MonitoringFrequency != None
- Calls KQL sub-functions to get LastSeen per specific source
- Fires per source exceeding its MonitoringFrequency threshold
- One detection covers all sources in every environment
- Adding a source = update watchlist only

**Custom SLS Detections**
- For sources needing specific logic beyond the master
- OC##### stored in the SLS field in [mssname]-sources
- Master is the floor — custom SLS detections are the ceiling

Note: A separate master table detection is not needed. Every table has at least one row in sources so the master source detection covers all tables automatically.

---

## Shared Table Health Check Architecture

### The Problem
Each shared table needs different filter conditions to identify its sub-sources. CommonSecurityLog uses DeviceVendor and DeviceProduct. AzureDiagnostics uses ResourceType and Category. Syslog uses HostName. ThreatIntelIndicators uses SourceSystem. These cannot be standardized at the filter level.

Dynamic KQL execution of stored filter strings is not possible. Storing filter conditions in a watchlist is fragile and not maintainable at scale.

### The Solution — Standardize the Output Not the Input
Each shared table gets its own KQL function. Every function returns exactly these four columns regardless of internal filter logic:

```
Table, LogSource, LastSeen, Status
```

A master function unions all sub-functions. The workbook calls the master function only — never changes regardless of how many shared tables exist.

### The Output Contract — Every Sub-Function Must Return These Four Columns
```kql
// Every shared table sub-function must return this exact schema
// Table      — exact table name
// LogSource  — matches LogSource value in [mssname]-sources exactly
// LastSeen   — raw datetime of most recent record for this source
// Status     — Active / Review / Inactive / No Data
```

### Example Sub-Function Structure
```kql
// GetAzureDiagnosticsHealth()
union
(
    AzureDiagnostics
    | where TimeGenerated > ago(24h)
    | where ResourceType == "AZUREFIREWALLS"
    | where Category == "AzureFirewallApplicationRule"
    | summarize LastSeen = max(TimeGenerated)
    | extend Table = "AzureDiagnostics"
    | extend LogSource = "Azure Firewall — Application Rule"
),
(
    AzureDiagnostics
    | where TimeGenerated > ago(24h)
    | where ResourceType == "VAULTS"
    | where Category == "AuditEvent"
    | summarize LastSeen = max(TimeGenerated)
    | extend Table = "AzureDiagnostics"
    | extend LogSource = "Azure Key Vault — Audit Events"
)
| extend DaysSince = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSince <= 1, "Active",
    DaysSince <= 7, "Review",
    DaysSince > 7,  "Inactive",
    "No Data"
)
| project Table, LogSource, LastSeen, Status
```

### The Master Function Structure
```kql
// GetSharedTableHealth()
union
    GetAzureDiagnosticsHealth(),
    GetCommonSecurityLogHealth(),
    GetSyslogHealth(),
    GetThreatIntelHealth(),
    GetASIMHealth()
| join kind=fullouter (
    _GetWatchlist('[mssname]-sources')
    | where isnotempty(FunctionName)
) on Table, LogSource
| extend FinalStatus = iff(isnull(LastSeen), "Missing", Status)
```

### The Fit Check
The fullouter join in the master function IS the fit check:
- In sources but LastSeen is null → Missing — source not reporting
- LastSeen present but not in sources → Unknown new source appeared
- Both present → compare Status

### How to Add a New Shared Table
1. Write sub-function following the four-column output contract
2. Add it to the union in GetSharedTableHealth()
3. Add rows to [mssname]-sources for each expected sub-source with FunctionName set
4. Done — workbook picks it up automatically

### Shared Tables That Need Sub-Functions
- AzureDiagnostics — ResourceType + Category
- CommonSecurityLog — DeviceVendor + DeviceProduct
- Syslog — HostName
- ThreatIntelIndicators — SourceSystem
- ThreatIntelObjects — SourceSystem
- ASimNetworkSessionLogs — EventVendor + EventProduct
- ASimAuditEventLogs — EventVendor + EventProduct
- ASimWebSessionLogs — EventVendor + EventProduct

---

## DetectionCatalog — Special Cases

### Detections That Query All Tables
Some detections use `search *` or `union *`. For these the Table field in [mssname]-detections is set to `All`.

The mv-expand join on the Table field will not match `All` against any specific table name. Handle separately:

```kql
let allTableDets = _GetWatchlist('[mssname]-detections')
| where Table == "All"
| summarize AllTableDetections = make_set(AnalyticRule);
```

Do not join `All` against every row in sources — this inflates detection counts for every source.

### Comma Separated Table Field Requires mv-expand
```kql
_GetWatchlist('[mssname]-detections')
| mv-expand Tables = split(Table, ",")
| extend TableName = trim(" ", tostring(Tables))
```

The `trim()` is critical — spaces after commas cause silent join failures.

---

## Sources Watchlist — Datetime Handling

Status, LastSeen, and DaysSinceLastLog are never stored in [mssname]-sources. All three are calculated live by the functions at query time.

If any query reads a datetime value from a watchlist it must wrap with todatetime():
```kql
| extend SomeDate = todatetime(SomeDate)
```

Watchlists store all values as strings. Forgetting todatetime() causes silent failures — date math returns null with no error message.

---

## ASIM Tables — Health Dependency

ASIM tables have no independent health. Their health depends entirely on the source tables feeding them.

Do not show ASIM table status as independent health indicators. When an ASIM source table is inactive the ASIM table may still appear active while producing degraded output.

Fixing an ASIM table means fixing its source tables — not the ASIM table itself.

---

## Shared Tables — Multiple Rows in Sources

CommonSecurityLog, Syslog, AzureDiagnostics, ThreatIntelIndicators have multiple rows in [mssname]-sources — one per sub-source. Joins against these tables produce multiple matches.

Summarize after joining to get distinct counts:
```kql
| summarize DetectionCount = dcount(RuleId) by Table, LogSource
```

---

## ThreatIntelIndicators — Special Cases

### Always Filter on ExpirationDateTime
```kql
| where ExpirationDateTime > now()
```
Without this filter stale expired indicators are included causing false positives.

### Legacy Table Still Exists
`ThreatIntelligenceIndicator` is the legacy table. New indicators only go to `ThreatIntelIndicators`. Rules referencing the legacy table will not match new indicators.

### Multiple Feeds Write to Same Table
Each feed is identified by SourceSystem. Handle per feed in the sub-function — one block per expected feed.

---

## GetEnrichedInventory — Design Rules

### Always Use leftouter Joins
```kql
| join kind=leftouter detections on Table
```
Inner joins drop sources with no matching detections. Always preserve all sources rows.

### Null Handling After Join
```kql
| extend DetectionCount = coalesce(DetectionCount, 0)
| extend HasCoverage = DetectionCount > 0
```

### Performance
The search * for live health is expensive on large workspaces. Limit to ago(24h). Only run the live health portion when the workbook tab is actively viewed.

---

## General KQL Gotchas

### make_set Default Limit
`make_set()` truncates at 128 items. For detection lists use:
```kql
| summarize DetectionList = make_set(AnalyticRule, 500)
```

### trim() After Split
Always `trim(" ", tostring(value))` after splitting comma separated fields. Leading and trailing spaces cause silent join failures with no error message.

### Case Sensitivity
KQL string joins are case sensitive. `SignInLogs` != `signinlogs`. Table names in [mssname]-detections must match [mssname]-sources exactly including casing.

---

## DCR Timestamp Standards

### TimeGenerated Should Always Be Event Time
TimeGenerated must be explicitly set in DCR transforms to the source event timestamp. If not set it defaults to ingestion time — batch arrivals appear as if everything happened at once.

### Batch Sources — TimeIngested Convention
For batch sources add `extend TimeIngested = _TimeReceived` in the DCR transform. Use TimeIngested for freshness checks on batch sources — not TimeGenerated.

### ingestion_time() Is a Function Not a Field
Cannot be stored. Cannot filter large datasets efficiently. Use in summary queries only:
```kql
// Good
| summarize LastIngestion = max(ingestion_time())
| where datetime_diff('hour', now(), LastIngestion) > 25

// Bad — expensive
| where ingestion_time() > ago(25h)
```

---

## Watchlist-Driven Variable Pattern — Core Methodology

Every detection uses watchlist values as variables wherever possible. The detection contains the logic. The watchlist contains the configuration. They never mix.

**Why this matters:**
- Same detection deploys to all clients
- Each client gets their own thresholds from their watchlist
- Changing a threshold = update watchlist, never touch detection code
- Onboarding a new client = upload watchlists, detections work immediately

**Apply this pattern to:**
- Silent detection thresholds — MonitoringFrequency from [mssname]-sources
- Critical asset checks — read from critical assets watchlist
- Client-specific logic — read from [mssname]-client

---

## Onboarding Engine Vision — Next Project

The watchlist-driven approach extends into a complete client onboarding system. New client gets a guided spreadsheet. They fill in what they know. We fill in technical fields. Upload watchlists. Workbook lights up automatically.

The intake spreadsheet IS the watchlist template. Same fields. Same structure. Fill it in once. Use it forever.

Onboarding is complete when the workbook shows green — not when someone ticks a box.

---

## Audit Deck Tab Order — Final
```
1. Splash
2. SLS and Capabilities
3. Table Health
4. Data Sources
5. Dependencies
6. Detections
7. MITRE Coverage
8. Ingestion and Cost
9. Opportunities and Action Items
```

Two modes — client and internal. Same functions, different field projections. Design every function to return all fields — the workbook query controls what is shown.

---

## Logic Apps — Future Automation

Build the KQL function and watchlist layer first. Introduce Logic Apps only when manual alternatives are unsustainable.

**Potential use cases:**
- Snapshot pipeline — write DataSourceAudit_CL weekly
- New source detector — compare workspace tables against [mssname]-tables
- Credential expiry notifications — read [mssname]-dependencies, send Teams alerts
- DetectionCatalog sync — trigger Python script when new rules deploy

Every Logic App needs its own silent detection. Only introduce when value justifies the overhead.

---

## Core Engineering Principles

**Repeatable** — any engineer can follow the same process and get the same result.

**Standardized** — follows the same pattern as everything else. New shared tables get sub-functions. New watchlists follow the naming convention. New detections follow the OC##### convention.

**Scalable** — adding more clients, tables, or detections requires adding to the system not rebuilding it.

**Automate where the value justifies the overhead.** Not everything should be automated.

---

## Sources Investigation Spreadsheet — Two Spreadsheet Design

The sources build process uses two separate spreadsheets that serve different purposes.

**`[mssname]-detections` spreadsheet — detection ledger:**
- One row per detection
- RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description
- Starting point for the sources build process — fed to AI to reorganize by table
- Becomes the [mssname]-detections watchlist
- Owned by detection team — do not modify structure

**Sources investigation spreadsheet — working reference:**
- One row per log source
- Built by AI from the detections spreadsheet
- Contains all fifteen [mssname]-sources fields PLUS four investigation fields:
  - Tier — 1 / 2 / 3 / 4 — assign using silent detection tier framework
  - Status — Active / Inactive / Review / No Data — from live environment
  - Verified — Yes / No — confirmed against live environment
  - ActionItems — anything that needs to be done before upload
- Named: `[ClientName]-sources-investigation-[YYYY-MM-DD].xlsx`
- When complete remove the four investigation columns and save as:
  `[ClientName]-sources-[YYYY-MM-DD].xlsx` for watchlist upload

**The flow:**
```
[mssname]-detections → feed to AI → reorganize by table
        ↓
AI produces CSV with 15 sources fields + 4 investigation fields
        ↓
Paste into sources investigation spreadsheet
        ↓
Verify against live environment — fill in Tier, Status, Verified, ActionItems
        ↓
Remove investigation columns
        ↓
[mssname]-sources watchlist upload
```

**Important — LogSource specificity:**
When reviewing AI output verify that LogSource values are specific enough to match what live breakout queries show. If the AI produces a generic value like Azure Diagnostics Logs rather than Azure Firewall — Application Rule that row needs manual correction before upload. The LogSource value in the watchlist and in the KQL function must match exactly — this is the join key.

---

*Add new notes here as they come up during development. Reference this file before writing any KQL function, workbook query, or automation logic.*

---

## Sources Investigation Spreadsheet — Two Spreadsheet Design

The sources build process uses two separate spreadsheets that serve different purposes.

**`[mssname]-detections` spreadsheet — detection ledger:**
- One row per detection
- RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description
- Starting point for the sources build process — fed to AI to reorganize by table
- Becomes the [mssname]-detections watchlist
- Owned by detection team
- Keep as-is — do not modify structure

**Sources investigation spreadsheet — working reference:**
- One row per log source
- Built by AI from the detections spreadsheet
- Contains all fifteen [mssname]-sources fields PLUS four investigation fields:
  - Tier — 1 / 2 / 3 / 4
  - Status — Active / Inactive / Review / No Data — from live environment
  - Verified — Yes / No — confirmed against live environment
  - ActionItems — anything that needs to be done
- Named: `[ClientName]-sources-investigation-[YYYY-MM-DD].xlsx`
- When complete remove investigation columns and save as:
  `[ClientName]-sources-[YYYY-MM-DD].xlsx` for watchlist upload

**The relationship:**
```
[mssname]-detections → feed to AI → sources investigation spreadsheet
                                           ↓ remove investigation columns
                                    [mssname]-sources watchlist
```

---

## AI Prompt — DetectionCatalog Has No RuleId in Reorganization Step

When feeding the detections CSV to AI in Step 1 of the sources build process to reorganize by table — the output is a table-first reference used for working purposes only. It does not need RuleId because it is not being uploaded anywhere at this stage.

However when the AI produces the full watchlist row output in Step 4 it works from detection KQL — there is no RuleId in the KQL itself. The AI identifies sources from filter conditions not from rule IDs.

**The gap to consider before finalizing the prompt:**
The current training prompt does not explicitly tell the AI what to do when a detection has no clear table filter — for example workspace-wide detections using search *. The prompt handles this with the All value but does not address how to handle cases where the AI cannot confidently identify the sub-source from the KQL alone.

**Recommendation:** When reviewing AI output for each table verify that LogSource values are specific enough to match what the live breakout queries show. If the AI produces a generic LogSource like Azure Diagnostics Logs rather than Azure Firewall — Application Rule that row needs manual correction before the watchlist is uploaded.

This will be reviewed and the prompt refined during the end-of-process review session.
