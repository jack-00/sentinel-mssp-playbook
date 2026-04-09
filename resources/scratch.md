# Scratch — Working Document

> ⚠️ **This is a working scratchpad — not a finished document.**
> Content here is temporary, experimental, or in progress. Anything worth
> keeping permanently gets moved to the appropriate document in the repo.

---

## How to Use This File

Use this file during active audit and build sessions to:
- Store queries you are testing before they are validated
- Copy prompts to run in Copilot or the Logs blade
- Capture observations and notes as you work through tables
- Park ideas that come up mid-session before they get lost

When something is proven and worth keeping permanently:
- Validated queries → `resources/kql-snippets.md`
- Table documentation → `resources/table-reference/`
- Process updates → `02-data-sources/data-source-audit-runbook.md`
- New ideas → `PROGRAM-MAP.md` ideas parking lot

---

## Key Decisions — Read Before Running Any Query

**Raw datetime only — no format_datetime()**
All queries output LastSeen as a raw datetime value.
Do not add format_datetime() to any query.
Excel handles display formatting — select the LastSeen column
and format as YYYY-MM-DD HH:MM after export.
Raw datetime keeps the value usable for all downstream
workbook queries, snapshot pipeline, and change detection rules.

**DaysSinceLastLog — spreadsheet only**
Calculated in the query for audit reference.
Do not include in the watchlist upload.
The workbook calculates this live from LastSeen.

**Status — audit snapshot only**
Calculated in the query for audit reference.
Stored in the watchlist as a record of status at audit time.
The workbook recalculates Status live from workspace queries.
Do not rely on watchlist Status in workbook logic.

**todatetime() in workbook queries**
Watchlists store all values as strings.
Any workbook query that uses LastSeen from the watchlist
must wrap it with todatetime():
_GetWatchlist('DataSourceInventory')
| extend LastSeen = todatetime(LastSeen)

---

## Copilot Prompt — Table Schema Analysis

```
I am conducting a data source audit of this Microsoft Sentinel workspace.

I need your help analyzing the [TABLE_NAME] table. Please do the following:

1. Pull the schema for the table and identify the most important fields
   for security analysis — specifically fields that identify the source,
   the type of event, the user account involved, and any fields that
   distinguish different log sources writing to this table.

2. Tell me which field I should use to identify individual sources
   within this table so I can break out one row per log source.

3. Identify which event types or categories are present in this table
   over the last 30 days and how many records each one has.

4. Tell me if there are any unusual or unexpected fields that might
   indicate custom DCR transformations or non-standard configurations.

The goal is to understand exactly what is in this table, who is writing
to it, and what security value it provides so I can document it in our
data source inventory.
```

---

## Standard Queries

### Main Table Inventory
```kql
// MAIN TABLE INVENTORY
// FIRST RUN: Remove the time filter line entirely
// SUBSEQUENT RUNS: Keep as-is
search *
| where TimeGenerated > ago(30d) // REMOVE ON FIRST RUN
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by $table
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| extend LogSource =  "[Manual]"
| extend Origin =     "[Manual]"
| extend Transport =  "[Manual]"
| extend Category =   "[Manual]"
| extend Purpose =    "[Manual]"
| extend SilentDet =  "[Manual]"
| extend Detections = "[Manual]"
| extend Notes =      "[Manual]"
| project
    Table = $table,
    Status,
    LastSeen,
    DaysSinceLastLog,
    LogSource,
    Origin,
    Transport,
    Category,
    Purpose,
    SilentDet,
    Detections,
    Notes
| order by Status asc, LastSeen desc
```

### Usage Volume
```kql
// USAGE VOLUME — Run separately
// Keep export open alongside main spreadsheet
// Add DailyVolume to Notes column for each table
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize
    TotalGB = round(sum(Quantity) / 1000, 3),
    AvgDailyGB = round(sum(Quantity) / 1000 / 30, 4)
    by DataType
| extend DailyVolume = case(
    AvgDailyGB == 0,    "< 1 MB",
    AvgDailyGB < 0.001, "< 1 MB",
    AvgDailyGB < 1,     strcat(tostring(round(AvgDailyGB * 1000, 1)), " MB"),
    strcat(tostring(round(AvgDailyGB, 2)), " GB")
)
| project
    Table = DataType,
    DailyVolume,
    TotalGB_30Days = TotalGB
| order by TotalGB_30Days desc
```

---

## Multi-Source Table Investigation

### The Deep Dive Principle

Identifying that a source writes to a shared table is only step one.
Step two is understanding what that source is actually sending.

The Azure Firewall example:
- Surface level: ResourceType = AZUREFIREWALLS — one row
- Deep level: four categories — ApplicationRule, NetworkRule,
  DnsProxy, ThreatIntelLog — each completely different coverage
- DnsProxy missing = DNS blind spot invisible at surface level

Apply this to every shared table. Run Level 1 to find sources.
Run Level 2 to find what each source sends. Decide rows based
on whether categories serve different security purposes.

---

### Generic Category Breakout Query

Use this template for any shared table after identifying
the source field in Level 1. Replace the three variables:
- TABLE_NAME — the actual table
- SOURCE_FIELD — the field that identifies the source
- SOURCE_VALUE — the specific source value to break out
- CATEGORY_FIELD — the field that identifies event categories

```kql
// GENERIC CATEGORY BREAKOUT
// Replace variables before running
TABLE_NAME
| where TimeGenerated > ago(30d)
| where SOURCE_FIELD == "SOURCE_VALUE"
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by SOURCE_FIELD, CATEGORY_FIELD
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| extend Table =     "TABLE_NAME"
| extend Origin =    "[Manual]"
| extend Transport = "[Manual]"
| extend Category =  "[Manual]"
| extend Purpose =   "[Manual]"
| extend SilentDet = "[Manual]"
| extend Detections ="[Manual]"
| extend Notes =     strcat(SOURCE_FIELD, ": ", SOURCE_VALUE,
                     " | ", CATEGORY_FIELD, ": ",
                     tostring(column_ifexists(CATEGORY_FIELD, "")))
| project
    Table,
    Status,
    LastSeen,
    DaysSinceLastLog,
    Origin,
    Transport,
    Category,
    Purpose,
    SilentDet,
    Detections,
    Notes
| order by CATEGORY_FIELD asc
```

The Notes field auto-populates with the source and category
values so you always know what each row represents.

---

### AzureDiagnostics

**Level 1 — What resource types are writing here?**
```kql
AzureDiagnostics
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by ResourceType, ResourceProvider
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by TotalRecords desc
```

**Level 2 — What categories does each resource type send?**
```kql
// AzureDiagnostics — Category Breakout
// Replace AZUREFIREWALLS with your ResourceType
AzureDiagnostics
| where TimeGenerated > ago(30d)
| where ResourceType == "AZUREFIREWALLS"
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by ResourceType, Category
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| project
    ResourceType,
    Category,
    Status,
    LastSeen,
    DaysSinceLastLog,
    TotalRecords
| order by Category asc
```

**Level 3 — Check specific resources within a category**
```kql
// Useful when multiple Key Vaults or Firewalls exist
AzureDiagnostics
| where TimeGenerated > ago(30d)
| where ResourceType == "VAULTS"
| where Category == "AuditEvent"
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by Resource
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| order by TotalRecords desc
```

**Level 4 — Verify NSG Flow Logs are actually present**
```kql
// NSG flow logs are frequently missing even when logging appears configured
AzureDiagnostics
| where TimeGenerated > ago(30d)
| where ResourceType == "NETWORKSECURITYGROUPS"
| where Category == "NetworkSecurityGroupFlowEvent"
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
// Zero results = flow logs not configured. Only rule events present.
```

**Known categories by ResourceType:**

AZUREFIREWALLS:
- AzureFirewallApplicationRule — layer 7 allow/deny (HIGH VALUE)
- AzureFirewallNetworkRule — layer 3/4 allow/deny (HIGH VALUE)
- AzureFirewallDnsProxy — DNS queries via firewall (only if DNS proxy enabled)
- AzureFirewallThreatIntelLog — TI indicator matches (HIGH VALUE)

VAULTS:
- AuditEvent — secret access, key operations (HIGH VALUE)
- AzurePolicyEvaluationDetails — policy compliance

NETWORKSECURITYGROUPS:
- NetworkSecurityGroupEvent — rule changes
- NetworkSecurityGroupRuleCounter — rule hit counters
- NetworkSecurityGroupFlowEvent — actual flow logs (HIGH VALUE — often missing)

WORKFLOWS:
- WorkflowRuntime — covers runs, actions, triggers

---

### AzureMetrics

**Level 1 — What resource types are sending metrics?**
```kql
AzureMetrics
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by ResourceType, Namespace
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by TotalRecords desc
```

**Level 2 — What metrics does each resource type send?**
```kql
// Replace VAULTS with your ResourceType
AzureMetrics
| where TimeGenerated > ago(30d)
| where ResourceType == "VAULTS"
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by ResourceType, MetricName
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| extend Table =     "AzureMetrics"
| extend Origin =    "Microsoft Azure"
| extend Transport = "Diagnostic Setting"
| extend Category =  "[Manual]"
| extend Purpose =   "[Manual]"
| extend SilentDet = "[Manual]"
| extend Detections ="[Manual]"
| extend Notes =     strcat("ResourceType: ", ResourceType,
                     " | Metric: ", MetricName)
| project
    Table,
    Status,
    LastSeen,
    DaysSinceLastLog,
    Origin,
    Transport,
    Category,
    Purpose,
    SilentDet,
    Detections,
    Notes
| order by MetricName asc
```

---

### CommonSecurityLog

**Level 1 — What vendors and products are writing here?**
```kql
CommonSecurityLog
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by DeviceVendor, DeviceProduct
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by TotalRecords desc
```

**Level 2 — What event categories does each vendor send?**
```kql
// Replace Palo Alto Networks with your DeviceVendor
CommonSecurityLog
| where TimeGenerated > ago(30d)
| where DeviceVendor == "Palo Alto Networks"
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by DeviceVendor, DeviceProduct, DeviceEventCategory
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| extend Table =     "CommonSecurityLog"
| extend Origin =    "[Manual] — Network Device or vendor name"
| extend Transport = "CEF — AMA — DCR"
| extend Category =  "[Manual] — Firewall / Network / Other"
| extend Purpose =   "[Manual] — What does this category help detect?"
| extend SilentDet = "[Manual] — AB##### or Missing"
| extend Detections ="[Manual] — AB#####, AB##### or None"
| extend Notes =     strcat("Vendor: ", DeviceVendor,
                     " | Product: ", DeviceProduct,
                     " | Category: ", DeviceEventCategory)
| project
    Table,
    Status,
    LastSeen,
    DaysSinceLastLog,
    Origin,
    Transport,
    Category,
    Purpose,
    SilentDet,
    Detections,
    Notes
| order by DeviceEventCategory asc
```

**Level 3 — Check allow vs deny distribution**
```kql
CommonSecurityLog
| where TimeGenerated > ago(7d)
| where DeviceVendor == "Palo Alto Networks"
| summarize Count = count() by DeviceAction
| order by Count desc
```

---

### Syslog

**Level 1 — What hosts are writing here?**
```kql
Syslog
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by HostName
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by TotalRecords desc
```

**Level 2 — What facilities does each host send?**
```kql
// Replace HOSTNAME with your host
Syslog
| where TimeGenerated > ago(30d)
| where HostName == "HOSTNAME"
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by HostName, Facility, SeverityLevel
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| extend Table =     "Syslog"
| extend Origin =    "[Manual]"
| extend Transport = "AMA — DCR"
| extend Category =  "[Manual] — Endpoint / Network / Identity / Other"
| extend Purpose =   "[Manual] — What does this host log help detect?"
| extend SilentDet = "[Manual] — AB##### or Missing"
| extend Detections ="[Manual] — AB#####, AB##### or None"
| extend Notes =     strcat("Host: ", HostName,
                     " | Facility: ", Facility,
                     " | Severity: ", SeverityLevel)
| project
    Table,
    Status,
    LastSeen,
    DaysSinceLastLog,
    Origin,
    Transport,
    Category,
    Purpose,
    SilentDet,
    Detections,
    Notes
| order by Facility asc
```

**Level 3 — Verify auth facility is present**
```kql
// Auth/authpriv is the highest value Syslog facility
// Verify it is present for hosts that should have it
Syslog
| where TimeGenerated > ago(24h)
| where HostName == "HOSTNAME"
| where Facility in ("auth", "authpriv")
| summarize
    LastSeen = max(TimeGenerated),
    Count = count()
// Zero results = authentication events not being collected
```

---

### ThreatIntelIndicators

**Level 1 — What feeds are writing here?**
```kql
ThreatIntelIndicators
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by SourceSystem
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by TotalRecords desc
```

**Level 2 — What indicator types does each feed provide?**
```kql
// Replace Microsoft Sentinel with your SourceSystem
ThreatIntelIndicators
| where TimeGenerated > ago(30d)
| where SourceSystem == "Microsoft Sentinel"
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by SourceSystem, IndicatorType
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| extend Table =     "ThreatIntelIndicators"
| extend Origin =    "Microsoft Sentinel"
| extend Transport = "TAXII — Polling"
| extend Category =  "Threat Intelligence"
| extend Purpose =   "[Manual] — What does this feed provide?"
| extend SilentDet = "[Manual] — AB##### or Missing"
| extend Detections ="[Manual] — AB#####, AB##### or None"
| extend Notes =     strcat("Feed: ", SourceSystem,
                     " | Type: ", IndicatorType)
| project
    Table,
    Status,
    LastSeen,
    DaysSinceLastLog,
    Origin,
    Transport,
    Category,
    Purpose,
    SilentDet,
    Detections,
    Notes
| order by IndicatorType asc
```

**Level 3 — Check indicator freshness**
```kql
ThreatIntelIndicators
| where TimeGenerated > ago(30d)
| summarize
    TotalIndicators = count(),
    FreshIndicators = countif(ExpirationDateTime > now()),
    StaleIndicators = countif(ExpirationDateTime <= now()),
    OldestExpiry = min(ExpirationDateTime),
    NewestExpiry = max(ExpirationDateTime)
    by SourceSystem
```

**Level 4 — Verify legacy table status**
```kql
// ThreatIntelligenceIndicator is LEGACY
// ThreatIntelIndicators is CURRENT
// Check if legacy table still has data
// Rules referencing legacy table should be updated
ThreatIntelligenceIndicator
| where TimeGenerated > ago(7d)
| summarize
    LastSeen = max(TimeGenerated),
    Count = count()
```

---

### ThreatIntelObjects

**Level 1 — What feeds and object types are present?**
```kql
ThreatIntelObjects
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by SourceSystem, ObjectType
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by TotalRecords desc
```

---

### ASimNetworkSessionLogs

**Level 1 — What sources are normalized here?**
```kql
ASimNetworkSessionLogs
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by EventVendor, EventProduct
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by TotalRecords desc
```

**Level 2 — What event types are present?**
```kql
ASimNetworkSessionLogs
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by EventVendor, EventType, DvcAction
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by EventVendor asc
```

**Level 3 — Verify normalization quality**
```kql
// Low coverage = parser misconfiguration
ASimNetworkSessionLogs
| where TimeGenerated > ago(24h)
| summarize
    TotalRecords = count(),
    HasSrcIP = countif(isnotempty(SrcIpAddr)),
    HasDstIP = countif(isnotempty(DstIpAddr)),
    HasDstPort = countif(isnotempty(DstPortNumber)),
    HasAction = countif(isnotempty(DvcAction))
| extend SrcIPCoverage = round(todouble(HasSrcIP) / TotalRecords * 100, 1)
| extend DstIPCoverage = round(todouble(HasDstIP) / TotalRecords * 100, 1)
```

---

### ASimAuditEventLogs

**Level 1 — What sources are normalized here?**
```kql
ASimAuditEventLogs
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by EventVendor, EventProduct
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by TotalRecords desc
```

**Level 2 — What audit event types are present?**
```kql
ASimAuditEventLogs
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by EventVendor, EventType, EventResult
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| order by EventVendor asc
```

---

### ASimWebSessionLogs

**Level 1 — What sources are normalized here?**
```kql
ASimWebSessionLogs
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by EventVendor, EventProduct
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by TotalRecords desc
```

**Level 2 — What web session types and actions are present?**
```kql
ASimWebSessionLogs
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by EventVendor, EventType, DvcAction
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| order by EventVendor asc
```

---

### SecurityEvent

**Level 1 — What machines are writing here?**
```kql
SecurityEvent
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count(),
    MachineCount = dcount(Computer)
    by Type
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
```

**Level 2 — What Event IDs are present?**
```kql
SecurityEvent
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    Count = count()
    by EventID
| order by Count desc
```

**Level 3 — Check critical Event ID coverage**
```kql
let critical_ids = datatable(EventID:int, Description:string)
[
    4624, "Successful logon",
    4625, "Failed logon",
    4648, "Logon with explicit credentials",
    4672, "Special privileges assigned",
    4688, "New process created",
    4698, "Scheduled task created",
    4720, "User account created",
    4728, "Member added to global group",
    4732, "Member added to local group",
    4740, "Account locked out",
    4771, "Kerberos pre-auth failed",
    4776, "NTLM credential validation",
    4768, "Kerberos TGT requested",
    4769, "Kerberos service ticket requested"
];
let present_ids = SecurityEvent
    | where TimeGenerated > ago(24h)
    | summarize by EventID;
critical_ids
| join kind=leftanti present_ids on EventID
| project EventID, Description,
    Status = "MISSING — not being collected"
```

**Level 4 — Check command line coverage for 4688**
```kql
// Zero CommandLineCoverage = process auditing on but
// CommandLine logging not configured in Windows audit policy
SecurityEvent
| where TimeGenerated > ago(24h)
| where EventID == 4688
| summarize
    TotalProcessCreation = count(),
    HasCommandLine = countif(isnotempty(CommandLine))
| extend CommandLineCoverage =
    round(todouble(HasCommandLine) / TotalProcessCreation * 100, 1)
```

---

### Operation

**Level 1 — What operation categories are present?**
```kql
Operation
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by OperationCategory, Solution
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| order by TotalRecords desc
```

**Level 2 — Check for errors**
```kql
Operation
| where TimeGenerated > ago(7d)
| where OperationStatus != "Succeeded"
| summarize
    LastSeen = max(TimeGenerated),
    Count = count()
    by OperationCategory, Detail
| order by Count desc
```

---

### Salesforce Tables

**Level 1 — Check if multiple orgs write here**
```kql
// Replace table name as needed
Salesforce_ListViewEvent
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by OrganizationId
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| order by TotalRecords desc
```

**Level 2 — Check login types and status**
```kql
Salesforce_LoginEvent
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by LoginType, LoginStatus
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| order by TotalRecords desc
```

---

### Heartbeat

**Level 1 — What machines are sending heartbeats?**
```kql
Heartbeat
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count()
    by Computer, OSType, Category, Version
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| order by LastSeen desc
```

**Level 2 — Machines that recently stopped**
```kql
Heartbeat
| where TimeGenerated > ago(30d)
| summarize LastSeen = max(TimeGenerated) by Computer
| where LastSeen < ago(24h)
| extend HoursSince = datetime_diff('hour', now(), LastSeen)
| order by HoursSince desc
```

---

## Notes and Observations

*Add notes here as you work through tables. Date stamp anything important.*

---

## Queries in Progress

*Paste queries here while testing.*
