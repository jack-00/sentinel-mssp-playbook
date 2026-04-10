# Developer Notes — Things to Consider Before Writing Code

> **Purpose:** This file captures edge cases, oddities, architectural decisions, and gotchas that must be considered when writing KQL functions, workbook queries, and automation. Before writing any code read this file first. When something new comes up that could affect future code add it here immediately.
>
> **Who uses this:** Anyone writing KQL, workbook queries, Logic App expressions, or automation for this program.
>
> **Last Updated:** April 2026

---

## DetectionCatalog — Special Cases

### Detections That Query All Tables
Some AB-series detections use `search *` or `union *` to query across all tables simultaneously. For these detections the Table field in DetectionCatalog is set to `All` rather than a specific table name.

**Impact on KQL:**
The `mv-expand` join between DetectionCatalog and DataSourceInventory on the Table field will not match `All` against any specific table name. These detections will not appear in the per-source detection count in the workbook unless handled explicitly.

**How to handle it:**
When building the GetEnrichedInventory function add a separate branch that identifies detections where Table = `All` and surfaces them as workspace-level detections — not tied to any specific source. The workbook should show these separately, perhaps as a workspace-wide coverage indicator.

```kql
// Separate handling for All-table detections
let allTableDets = _GetWatchlist('DetectionCatalog')
| where Table == "All"
| summarize AllTableDetections = make_set(AnalyticRule);
```

**Do not** try to join `All` against every row in DataSourceInventory — this would incorrectly inflate detection counts for every source.

---

## DataSourceInventory — Datetime Handling

### LastSeen Is Stored as a String in the Watchlist
Watchlists store all values as strings regardless of original type. LastSeen is a raw datetime value in the spreadsheet but becomes a string after watchlist upload.

**Every workbook query that uses LastSeen must wrap it with todatetime():**
```kql
_GetWatchlist('DataSourceInventory')
| extend LastSeen = todatetime(LastSeen)
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
```

Forgetting `todatetime()` causes silent failures — date math returns null, DaysSinceLastLog returns null, status calculations break. No error message. Just wrong results.

### DaysSinceLastLog Is Not in the Watchlist
DaysSinceLastLog was deliberately excluded from the watchlist because it goes stale immediately after upload. Always calculate it live at query time from LastSeen.

### Status Is an Audit Snapshot — Not a Live Value
The Status field in the watchlist reflects what the status was at the time of the audit. Do not use watchlist Status for current health decisions in the workbook. Always recalculate Status live from the workspace using DaysSinceLastLog derived from a live query.

---

## DetectionCatalog — Table Field Handling

### Comma Separated Values Require mv-expand
The Table field contains comma separated table names. Always use `mv-expand` before joining against DataSourceInventory:

```kql
_GetWatchlist('DetectionCatalog')
| mv-expand Tables = split(Table, ",")
| extend TableName = trim(" ", tostring(Tables))
```

The `trim()` is important — if there are spaces after commas `SignInLogs, SecurityEvent` the split produces ` SecurityEvent` with a leading space which will not match `SecurityEvent` in DataSourceInventory.

Same applies to the Watchlists field — use `mv-expand` before any join on watchlist names.

### Table Names Must Match Exactly
The join between DetectionCatalog and DataSourceInventory depends on exact table name matching. Case sensitive. No aliases. `SecurityEvent` not `Security Event` not `securityevent`.

If a join produces unexpected nulls or zero matches check for:
- Trailing spaces
- Leading spaces after comma separation
- Different casing
- Abbreviated or aliased names

---

## ASIM Tables — Health Dependency

### ASIM Tables Do Not Have Independent Health
ASimNetworkSessionLogs, ASimAuditEventLogs, ASimWebSessionLogs are normalized views over raw tables. Their health depends entirely on the health of the source tables feeding them.

**Impact on workbook:**
Do not show ASIM table status as independent health indicators. If the workbook shows ASimNetworkSessionLogs as Active but CommonSecurityLog is Inactive the ASIM table is producing degraded output. The workbook should surface this dependency.

**How to handle it:**
When building the ASIM rows in the workbook add a note or indicator that health is derived from source tables. Consider adding a dependency check that warns when an ASIM source table is inactive.

### Transport = ASIM Parser
When filtering DetectionCatalog to understand what a detection depends on — if the detection queries an ASIM table remember that fixing the ASIM table means fixing its source tables, not the ASIM table itself.

---

## Shared Tables — Multiple Sources

### CommonSecurityLog, Syslog, AzureDiagnostics Have Multiple Rows
These tables appear multiple times in DataSourceInventory — one row per log source category writing to them. When joining DetectionCatalog against DataSourceInventory on these tables the join will produce multiple matches — one per log source row.

**Impact on detection counts:**
A detection that queries CommonSecurityLog will match against every CommonSecurityLog row in DataSourceInventory — Palo Alto, Fortinet, and any other vendor. This is correct behavior — losing any of those sources affects the detection. But the workbook needs to handle the resulting multiple matches cleanly without inflating counts.

**How to handle it:**
Summarize after the join to get distinct detection counts per table not per source row:
```kql
| summarize DetectionCount = dcount(AnalyticRule) by Table, LogSource
```

### AzureDiagnostics Category Breakout
AzureDiagnostics has multiple rows for different ResourceTypes AND multiple rows within each ResourceType for different log categories (e.g. Azure Firewall has ApplicationRule, NetworkRule, DnsProxy, ThreatIntelLog).

When a detection queries AzureDiagnostics it may only care about a specific ResourceType or Category. The current DetectionCatalog design does not capture this granularity — Table = `AzureDiagnostics` applies to all rows.

**Future consideration:** May need a ResourceType or Category sub-field in DetectionCatalog for AzureDiagnostics detections to enable accurate dependency mapping.

---

## ThreatIntelIndicators — Multiple Feeds

### Multiple Feeds Write to the Same Table
ThreatIntelIndicators receives data from multiple feeds identified by the SourceSystem field. In DataSourceInventory this table has multiple rows — one per feed.

**Impact on detections:**
A detection that joins against ThreatIntelIndicators depends on ALL feeds being healthy — if one feed goes stale the detection continues to fire but with reduced coverage. The workbook should surface this nuance rather than showing a simple active/inactive status.

### Always Filter on ExpirationDateTime
Any detection or workbook query that reads from ThreatIntelIndicators should filter on active indicators only:
```kql
| where ExpirationDateTime > now()
```

Without this filter stale expired indicators are included which causes false positives and inflates match counts.

### Legacy Table Still Exists
`ThreatIntelligenceIndicator` is the legacy table. Some older analytics rules may still reference it. New indicators only go to `ThreatIntelIndicators`. If a detection references the legacy table it will not match against new indicators. Check for this when reviewing detection effectiveness.

---

## Watchlist Size Limits

### 10MB Per Watchlist
Sentinel watchlists have a 10MB size limit. As DetectionCatalog grows — especially the Description field with long text — this limit could become a constraint.

**Monitor:** Check watchlist size periodically. If approaching the limit consider splitting into multiple watchlists by detection type (SLD vs standard detections) or by category.

---

## ingestion_time() Is a Function Not a Field

### Cannot Be Stored or Filtered Efficiently
`ingestion_time()` is calculated at query time. It cannot be stored as a field. It cannot be used to filter large datasets efficiently.

**For batch source health monitoring** use it in summary queries not in filters:
```kql
// Good — summarize then check
| summarize LastIngestion = max(ingestion_time())
| where datetime_diff('hour', now(), LastIngestion) > 25

// Bad — filtering millions of rows with ingestion_time()
| where ingestion_time() > ago(25h)
```

### _TimeReceived Is Resource Intensive
`_TimeReceived` is calculated each time it is used. Avoid using it to filter large numbers of records. Use it only for latency analysis on small summarized result sets.

---

## DCR Timestamp Standards

### TimeGenerated Should Always Be Event Time
TimeGenerated must always be explicitly set in DCR transforms to the source event timestamp. If not set it defaults to ingestion time which makes the data appear as if everything happened at once when a batch arrives.

### TimeIngested Field Convention
For batch sources that send daily dumps add `extend TimeIngested = _TimeReceived` in the DCR transform. This creates a stored field capturing when the batch actually arrived — separate from when the events happened. Use TimeIngested not TimeGenerated for health monitoring of batch sources.

### Salesforce and Other Batch Sources
Salesforce sends a daily dump. All records arrive at once but have event timestamps throughout the day. If TimeGenerated is correctly set to the source event timestamp the data looks like continuous activity even though it arrived in one batch. Always use `ingestion_time()` or TimeIngested for freshness checks on batch sources — not TimeGenerated.

---

## Multi-Workspace Considerations

### Workbook Queries Assume Single Workspace Context
All current queries are written for single workspace context. When the workbook needs to work across multiple client workspaces via Lighthouse the workspace scope must be explicitly handled.

**Future consideration:** Workbook parameters will need a workspace selector. Queries may need `workspace()` function calls. The GetEnrichedInventory function will need to accept a workspace parameter or be scoped per workspace deployment.

---

## GetEnrichedInventory Function Design

### Function Must Handle Missing Joins Gracefully
Not every table in DataSourceInventory will have matching rows in DetectionCatalog. Not every detection in DetectionCatalog will match a table in DataSourceInventory. Use `leftouter` joins not `inner` joins to preserve all DataSourceInventory rows.

```kql
| join kind=leftouter detections on Table
```

### Null Handling After Join
After a leftouter join rows with no matching detection will have null DetectionCount. Always use `coalesce()` to handle nulls:
```kql
| extend DetectionCount = coalesce(DetectionCount, 0)
| extend HasCoverage = DetectionCount > 0
```

### Performance Consideration
The `search *` in the live health query within GetEnrichedInventory is expensive across large workspaces. Consider limiting to `ago(24h)` and only running the live health portion when the workbook tab is actively viewed — not as part of a background refresh.

---

## General KQL Gotchas

### make_set Has a Default Limit of 128
`make_set()` truncates at 128 items by default. For DetectionCatalog joins where a table might have many detections use `make_set(AnalyticRule, 500)` to increase the limit.

### trim() After Split
Always `trim(" ", tostring(value))` after splitting comma separated fields. Leading and trailing spaces cause silent join failures.

### Case Sensitivity in Joins
KQL string joins are case sensitive by default. Table names in DetectionCatalog must match DataSourceInventory exactly including casing. `SignInLogs` != `signinlogs`.

---

*Add new notes here as they come up during development. Reference this file before writing any KQL function, workbook query, or automation logic.*