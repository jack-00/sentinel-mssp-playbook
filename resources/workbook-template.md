# Workbook Tab Templates

> **Purpose:** Header templates and KQL queries for each tab in the audit deck workbook. Copy and paste directly into Sentinel workbook elements. Headers go in markdown blocks. Queries go in query blocks.
>
> **Last Updated:** April 2026

---

## How to Use This File

Each tab has two parts:
1. **Header** — paste into a markdown element above the query
2. **Query** — paste into a query element below the header

In the workbook editor:
- Add markdown element → paste header
- Add query element → paste KQL → configure columns and formatting

---

## Tab 1 — Splash Screen

### Header
```markdown
# 🛡️ [Client Name]

[Company URL]

*[Industry]*

---

[Bio — two to three sentences about who the client is and why security matters to their business.]

---

| | |
|---|---|
| **Onboarding Date** | [YYYY-MM-DD] |
| **Primary Contact** | [Name — Email] |
| **Escalation Contact** | [Name — Email] |
| **Sentinel Workspace** | [Link] |

*Last reviewed: [YYYY-MM-DD]*
```

---

## Tab 2 — Table Health

### Header
```markdown
## 📊 Table Health

This view shows every data pipeline flowing into the Sentinel environment. Each row represents a Log Analytics table — the container where a specific type of security data lands. Tables highlighted in **red** have stopped receiving data and require immediate investigation. Tables marked **Unknown** are new and have not yet been classified — these should be reviewed and added to the watchlist.

| Status | Meaning |
|---|---|
| 🟢 Active | Data flowing within expected timeframe |
| 🟡 Review | Data flowing but getting stale |
| 🔴 Inactive | Data has stopped — investigate immediately |
| ⚫ No Data | Table exists but has never received data |
| 🔵 Unknown | New table detected — not yet classified |
```

### Query
```kql
let known = _GetWatchlist('[mssname]-tables')
| project Table, Category, Details, Vetted, Notes;
let live = search *
| where TimeGenerated > ago(30d)
| summarize LastSeen = max(TimeGenerated) by $table
| extend DaysSince = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSince <= 1, "Active",
    DaysSince <= 7, "Review",
    DaysSince > 7,  "Inactive",
    "No Data"
)
| project Table = $table, Status, LastSeen, DaysSince;
live
| join kind=fullouter known on Table
| extend Status = iff(isempty(Status), "Unknown", Status)
| extend Category = iff(isempty(Category), "Unknown — not in watchlist", Category)
| extend Details = iff(isempty(Details), "New table detected — investigate and classify", Details)
| project Table, Status, Category, LastSeen, DaysSince, Details, Notes
| order by Status asc, Table asc
```

**Color coding rules for Status column:**
- Active → green background
- Review → amber background
- Inactive → red background
- No Data → dark gray background
- Unknown → blue background

---

## Tab 3 — SLS and Capabilities

### Header
```markdown
## 🎯 SLS and Capabilities

This view shows every data source under our service level commitment and the health of key Sentinel capabilities. SLA sources are the data pipelines we are contractually responsible for monitoring. A red status here means we are not meeting our commitment and action is required immediately.

| Status | Meaning |
|---|---|
| 🟢 Active | Source is healthy and flowing |
| 🟡 Review | Source is getting stale — investigate |
| 🔴 Inactive | Source has stopped — SLA breach risk |
| 🔵 Unknown | Source not yet classified |
```

---

## Tab 4 — Data Sources

### Header
```markdown
## 🔌 Data Sources

This view shows every tracked data source flowing into the environment. Each row represents a specific source within a table — for example a Palo Alto firewall writing to CommonSecurityLog is one source, a Fortinet VPN writing to the same table is another. Sources with zero detections are flagged as coverage gaps.

| Status | Meaning |
|---|---|
| 🟢 Active | Data flowing within expected timeframe |
| 🟡 Review | Data flowing but getting stale |
| 🔴 Inactive | Data has stopped — investigate immediately |
| ⚫ No Data | Source exists but has never received data |
```

---

## Tab 5 — Dependencies

### Header
```markdown
## 🔑 Dependencies

This view shows every external credential, API key, and integration that keeps data sources running. When a credential expires the dependent data source goes silent — often without any obvious indication. Items expiring within 30 days are highlighted and require immediate action.

| Status | Meaning |
|---|---|
| 🟢 Valid | More than 60 days until expiry |
| 🟡 Expiring Soon | Less than 60 days until expiry |
| 🔴 Critical | Less than 30 days — action required |
| ⛔ Expired | Past expiry date |
```

### Query
```kql
_GetWatchlist('[mssname]-dependencies')
| extend ExpiryDate = todatetime(ExpiryDate)
| extend DaysUntilExpiry = datetime_diff('day', ExpiryDate, now())
| extend Status = case(
    DaysUntilExpiry < 0,   "Expired",
    DaysUntilExpiry <= 30, "Critical",
    DaysUntilExpiry <= 60, "Expiring Soon",
    "Valid"
)
| project DependencyName, ServiceName, DependencyType,
    LinkedSource, ExpiryDate, DaysUntilExpiry, Status,
    RotationOwner, Notes
| order by DaysUntilExpiry asc
```

**Color coding rules for Status column:**
- Valid → green background
- Expiring Soon → amber background
- Critical → orange background
- Expired → red background

---

## Tab 6 — Detections

### Header
```markdown
## 🔍 Detections

This view shows every custom detection deployed in this environment. Security detections watch for threats and anomalies. Operational detections monitor platform health. Detections that have not fired in 30 days may indicate a misconfiguration or a source that has gone silent.

| Alert Class | Meaning |
|---|---|
| 🔴 Security | Threat detection — monitors for attacks and anomalies |
| 🔵 Operational | Platform health — monitors data sources and infrastructure |
| 🟡 Compliance | Regulatory requirement monitoring |
| 🟢 Reporting | Feeds workbook and scheduled reports |
| ⚫ Capability | Sentinel capability dependency monitoring |
```

### Query
```kql
let catalog = _GetWatchlist('[mssname]-detections');
let health = SentinelHealth
| where SentinelResourceType == "Analytics Rule"
| summarize LastRun = max(TimeGenerated), Status = arg_max(TimeGenerated, Status)
    by SentinelResourceName
| project RuleName = SentinelResourceName, LastRun, HealthStatus = Status;
let fired = SecurityAlert
| where TimeGenerated > ago(30d)
| summarize AlertCount = count() by AlertName
| project RuleName = AlertName, AlertCount;
catalog
| join kind=leftouter health on $left.AnalyticRule == $right.RuleName
| join kind=leftouter fired on $left.AnalyticRule == $right.RuleName
| extend AlertCount = coalesce(AlertCount, 0)
| extend HealthStatus = coalesce(HealthStatus, "Unknown")
| project RuleId, AnalyticRule, AlertClass, Table,
    HealthStatus, LastRun, AlertCount, Description
| order by AlertClass asc, AnalyticRule asc
```

---

## Tab 7 — MITRE Coverage

### Header
```markdown
## 🗺️ MITRE Coverage

This view shows detection coverage mapped against the MITRE ATT&CK framework. Each enabled detection is mapped to the tactics and techniques it covers. Gaps in coverage represent attack techniques that could go undetected in this environment.

The built-in Microsoft Sentinel MITRE workbook provides the full interactive heatmap. The table below shows detection-to-technique mappings from our custom detection catalog.
```

---

## Tab 8 — Ingestion and Cost

### Header
```markdown
## 💰 Ingestion and Cost

This view shows data ingestion volume and cost per table over the last 30 days. Tables with high ingestion and zero detection coverage represent cost without security value — these are candidates for optimization.
```

### Query
```kql
let coverage = _GetWatchlist('[mssname]-detections')
| mv-expand Tables = split(Table, ",")
| extend TableName = trim(" ", tostring(Tables))
| summarize DetectionCount = dcount(RuleId) by TableName;
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize
    TotalGB = round(sum(Quantity) / 1000, 3),
    AvgDailyGB = round(sum(Quantity) / 1000 / 30, 4)
    by DataType
| extend DailyVolume = case(
    AvgDailyGB < 0.001, "< 1 MB",
    AvgDailyGB < 1, strcat(tostring(round(AvgDailyGB * 1000, 1)), " MB"),
    strcat(tostring(round(AvgDailyGB, 2)), " GB")
)
| join kind=leftouter coverage on $left.DataType == $right.TableName
| extend DetectionCount = coalesce(DetectionCount, 0)
| extend CoverageStatus = iff(DetectionCount == 0, "⚠️ No Detection Coverage", "✅ Covered")
| project Table = DataType, DailyVolume, TotalGB_30Days = TotalGB,
    DetectionCount, CoverageStatus
| order by TotalGB_30Days desc
```

---

## Tab 9 — Opportunities

### Header
```markdown
## 💡 Opportunities and Action Items

This section captures open items, recommendations, and items for discussion from this review. Use this tab as the agenda for the client conversation. Items carry forward from previous reviews until resolved.

| Priority | Meaning |
|---|---|
| 🔴 High | Immediate action required |
| 🟡 Medium | Address in next 30 days |
| 🟢 Low | Address when time permits |
```

---

*This file is maintained in the sentinel-mssp-playbook repository. Add new tab templates as tabs are built and validated.*