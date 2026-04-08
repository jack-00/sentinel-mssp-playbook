# Data Source Audit Runbook

> **Purpose:** This runbook guides an engineer through a complete data source audit for a managed client Microsoft Sentinel environment. The output is a two-tab Excel workbook that becomes the foundation for the data source watchlist, the living workbook, and all ongoing health monitoring.
>
> **Who uses this:** Engineers conducting a first-time audit or a periodic re-audit of a client environment.
>
> **What you produce:** A completed, sorted, two-tab Excel workbook. Tab 1 becomes the Sentinel watchlist. Tab 2 is the technical configuration reference.
>
> **Prerequisites:** Complete `01-baseline/license-entitlement-review.md` before starting this audit. Knowing what the client is licensed for tells you what data sources should exist and what capabilities should be enabled.
>
> **Last Updated:** April 2026

---

## Understanding the Workspace Before You Audit

A Microsoft Sentinel workspace is not a flat list of data sources. It contains nine distinct types of components that each play a different role. Understanding these categories is what separates a surface-level audit from a complete one. Every field in the audit spreadsheet maps back to this taxonomy.

| Category | What It Is | Examples |
|---|---|---|
| **Log Sources** | Specific originators writing data to a table. One table can have many log sources. | Palo Alto PA-3200 → CommonSecurityLog, DC01 → Syslog |
| **Tables** | Log Analytics destinations where data lands. The ground truth layer. | SignInLogs, CommonSecurityLog, BehaviorAnalytics |
| **Watchlists** | Static reference data used by detections and capabilities | VIP Users, Critical Assets, Domain Controllers |
| **Capabilities** | Features depending on multiple sources — UEBA, Threat Feed, Fusion ML | If any dependency breaks the capability degrades silently |
| **Detections** | Analytics rules that consume table data and produce alerts | AB-series rules, Content Hub rules |
| **Silent Detections** | Detections that verify each log source is actively sending data | Every log source must have one |
| **Automation** | Logic Apps and playbooks that respond to incidents | Enrichment playbook, ServiceNow ticket creation |
| **Reports** | Scheduled outputs delivered on a cadence | Logic App reports using watchlists |
| **Integrations** | API keys and credentials that keep services connected | Threat feed API key, VirusTotal key |
| **Workbooks** | Interactive live dashboards | Client Security Operations Workbook |

---

## Why We Audit by Log Source — Not by Connector or Table

Microsoft Sentinel presents data sources through a connector UI that makes it easy to assume that if a connector shows as connected, everything is working. This assumption is dangerous and wrong.

A connector is a configuration wizard. It helps you enable a data source but it does not guarantee data is actually flowing, that all log categories are enabled, that the correct tables are being written to, or that volume is healthy. A connector can show green while silently sending nothing.

Tables are closer to the truth but still not granular enough. A table like `CommonSecurityLog` or `Syslog` can have many different sources writing to it simultaneously. If a Palo Alto firewall stops sending logs but a Fortinet device keeps writing — `CommonSecurityLog` still appears Active. The Palo Alto source is blind and you will never know from the table level alone.

**The log source is the correct audit unit.** One row per originating source category. One status per source. This is the only level of granularity that catches individual source failures before they cause missed detections.

**Important distinction — log source category vs individual device:**
A log source is not every individual machine or device. It is the category of thing writing to the table. Fifty Windows machines sending data to SecurityEvent via AMA is one log source — "Windows Security Events via AMA." We only create separate rows when fundamentally different things are writing to the same table — for example a Palo Alto firewall and a Fortinet VPN both writing to CommonSecurityLog are two different log sources because they are different vendors with different purposes and different detection value.

---

## The Spreadsheet Structure

The audit produces a two-tab Excel workbook. Understanding both tabs before you start is important.

### Tab 1 — Data Sources
This is the operational view. It becomes the Sentinel watchlist that powers the living workbook. Every row is a log source. Keep it clean, accurate, and client-readable. This is what drives the workbook, the watchlist, and the client conversation.

**Columns — in this exact order:**

| Column | Short Header | What Goes Here |
|---|---|---|
| Table | Table | Exact Log Analytics table name |
| Status | Status | Current health — plain text value from approved list |
| Last Seen | LastSeen_UTC | Timestamp of most recent record |
| Days Since Last Log | DaysSinceLastLog | Number of days since last record — calculated by query |
| Log Source | LogSource | Plain English description of what is writing to this table |
| Vendor | Vendor | Company that produces this data |
| Category | Category | Type of data — from approved category list |
| Purpose | Purpose | One sentence — what attack or risk does this detect |
| Collection | Collection | How data gets into this table — from approved list |
| Silent Detection | SilentDet | AB-series detection that monitors this source for silence |
| Detections | Detections | AB-series detections that depend on this source |
| Notes | Notes | Issues, action items, context, exceptions |

### Tab 2 — Configuration
This is the technical reference view. Engineer-facing only. Never shown to clients. Documents the exact technical configuration behind each log source — connectors, DCRs, DCEs, subscriptions, and license context. Linked to Tab 1 by Table and LogSource.

**Columns:**

| Column | What Goes Here |
|---|---|
| Table | Exact table name — matches Tab 1 |
| LogSource | Log source — matches Tab 1 |
| Data Connector | Name of the Sentinel data connector if applicable |
| Connector Status | Connected / Disconnected / Not Applicable |
| DCR Name | Name of the Data Collection Rule if AMA-based |
| DCE Name | Name of the Data Collection Endpoint if applicable |
| Diagnostic Setting | Name of the diagnostic setting if applicable |
| Subscription | Azure subscription name this source lives in |
| Workspace | Log Analytics workspace name |
| License Required | What license is needed for this source |
| Notes | Any technical notes, issues, or configuration details |

Tab 2 is filled in manually during Phase 3 verification. It does not come from a query.

---

## Status Values

Status is always plain text — no emojis. Emojis in watchlist field values cause problems when KQL queries match against them. Plain text is clean, reliable, and queryable.

| Status | Meaning | Who Assigns |
|---|---|---|
| Active | Data flowing within expected timeframe. Volume looks normal. No action needed. | Query — based on DaysSinceLastLog |
| Inactive | Had data, now silent. Something broke. Investigate immediately. Document cause in Notes before sign-off. | Query — based on DaysSinceLastLog |
| Review | Data flowing but something warrants attention. Volume anomaly, inconsistent timestamps, or partial configuration suspected. Needs closer look before marking Active. | Query — based on DaysSinceLastLog, or manual override |
| No Data | Never received data. Confirm in Notes whether this is not applicable for this client or whether it was never configured and should be. | Manual |
| Missing | Expected to exist based on connector configuration or license but not found anywhere in the workspace. The connector may be enabled but never successfully sent a single record. | Manual |
| Decom | Marked for retirement. Source is no longer needed or the device has been decommissioned. Do not delete from the watchlist — mark Decom with a date and reason in Notes so there is a historical record. | Manual only |

**Status guidance by source type:**
- Identity sources (SignInLogs, AuditLogs) — data expected every few minutes in any active environment. More than 4 hours without data → Review. More than 24 hours → Inactive.
- Endpoint sources — depend on device activity. Low volume on weekends may be normal. No data for 48 hours in a business environment → Inactive.
- Firewall and network sources — near-continuous in any production environment. More than 6 hours → Review. More than 24 hours → Inactive.
- Threat intelligence tables — may update once daily or less. Check vendor cadence before marking Inactive.
- UEBA output tables — should populate continuously once enabled and past the 14-21 day baseline period.

**Note on Flag status:** Flag is reserved for the future automation pipeline. When the snapshot Logic App is running it will automatically detect new tables that appear between snapshots and assign them Flag status for investigation. During a manual audit you will not assign Flag — you classify everything as you go. Flag is documented here so engineers understand it when they see it in the future.

---

## Approved Category Values

Select exactly one. Do not create new categories — use Other and explain in Notes.

| Category | Use For |
|---|---|
| Identity | User authentication, directory services, account management, sign-in activity |
| Endpoint | Device telemetry, EDR, antivirus, host-based security events |
| Email | Mail flow, phishing detection, attachment and URL scanning |
| Network | Traffic logs, DNS, DHCP, proxy, network flow data |
| Firewall | Perimeter firewall and next-generation firewall logs |
| Cloud Infrastructure | Azure resource logs, management plane activity, cloud workload protection |
| SaaS Application | Third-party cloud applications — Salesforce, ServiceNow, Okta |
| Threat Intelligence | IOC feeds, indicator matching, threat actor data |
| Compliance and Audit | M365 audit logs, Sentinel health and audit, regulatory compliance events |
| Vulnerability Management | Scanner results, CVE data, asset risk scoring |
| Authentication | MFA, SSO, PAM, privileged access management |
| Capabilities | Tables produced by capabilities — UEBA output, Fusion ML output. These are not raw inputs, they are capability outputs. |
| Other | Anything not fitting the above — explain in Notes |

---

## Approved Collection Values

Select exactly one. This describes the mechanism that collects and ships data to the workspace.

| Collection | Use For |
|---|---|
| Microsoft Connector | Native Microsoft service-to-service integration. Configured in the Sentinel data connectors page. No agent required. Examples: Entra ID, Office 365, Azure Activity, Defender XDR. |
| AMA Agent | Azure Monitor Agent installed on a host. Ships data to the workspace via a Data Collection Rule (DCR). Examples: SecurityEvent, Syslog, Heartbeat. |
| CEF Syslog | Third-party device sending logs in Common Event Format or Syslog format to a Linux forwarder VM, which then ships to the workspace via AMA. Examples: CommonSecurityLog from firewalls and network devices. |
| REST API | Data pulled from or pushed via an API. Usually implemented through a Logic App or Azure Function App. Examples: third-party SaaS integrations. |
| Logic App | A scheduled or triggered Logic App that polls a data source and ingests results. Examples: custom threat feed, scheduled report data. |
| Manual Custom | Any custom ingestion pipeline that does not fit the above categories. Document the specific mechanism in Notes. |

---

## The Audit Workflow

Work through all steps in order. Do not skip ahead.

- [ ] **Step 0** — Complete `license-entitlement-review.md` for this client
- [ ] **Step 1** — Run the main table inventory query
- [ ] **Step 2** — Run shared table breakout queries where applicable
- [ ] **Step 3** — Run the Usage volume query
- [ ] **Step 4** — Export all results and open in Excel
- [ ] **Step 5** — Merge breakout rows and format as Excel Table
- [ ] **Step 6** — Save as dated workbook and upload to SharePoint
- [ ] **Step 7** — Fill in all manual fields
- [ ] **Step 8** — Complete Tab 2 configuration reference
- [ ] **Step 9** — Run Phase 3 verification for every Active source
- [ ] **Step 10** — Document all non-Active sources in Notes
- [ ] **Step 11** — Sort alphabetically by Table then LogSource
- [ ] **Step 12** — Upload Tab 1 as the DataSourceInventory watchlist

---

## Step 1 — Main Table Inventory Query

Run this query in the Microsoft Sentinel Logs blade. It returns every table in the workspace that has ever received data — including tables that went silent months ago.

**First run instruction:** On the very first audit for a client remove the time filter entirely. You want every table that has ever existed in this workspace — including ones that went silent months ago. On subsequent audits keep the 30-day filter.

**Status values are plain text** — no emojis. This ensures the values are watchlist-compatible and cleanly queryable in KQL.

```kql
// ============================================================
// MAIN TABLE INVENTORY — Run first. One row per table.
// FIRST RUN: Remove the 'ago' time filter line entirely
// SUBSEQUENT RUNS: Keep as-is for 30-day window
// ============================================================
search *
| where TimeGenerated > ago(30d) // REMOVE THIS LINE ON FIRST RUN
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
| extend LastSeen_UTC = format_datetime(LastSeen, 'yyyy-MM-dd HH:mm')
// ── Manual fields — fill these in after export ───────────────
| extend LogSource =  "[Manual] — Plain English description of what writes to this table e.g. Windows Security Events via AMA, Microsoft Entra ID Sign-in Logs"
| extend Vendor =     "[Manual] — Company that produces this data e.g. Microsoft, Palo Alto Networks, Fortinet"
| extend Category =   "[Manual] — Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other"
| extend Purpose =    "[Manual] — One sentence: what attack or risk does this data help detect? Write for a business audience."
| extend Collection = "[Manual] — Microsoft Connector / AMA Agent / CEF Syslog / REST API / Logic App / Manual Custom"
| extend SilentDet =  "[Manual] — AB##### Detection name that monitors this source for silence. Write Missing if none exists."
| extend Detections = "[Manual] — AB##### Detection name, AB##### Detection name. Write None if no custom detections depend on this source."
| extend Notes =      "[Manual] — Issues, action items, context, exceptions. Date stamp any action items."
// ─────────────────────────────────────────────────────────────
| project
    Table = $table,
    Status,
    LastSeen_UTC,
    DaysSinceLastLog,
    LogSource,
    Vendor,
    Category,
    Purpose,
    Collection,
    SilentDet,
    Detections,
    Notes
| order by Status asc, LastSeen_UTC desc
```

**Reading the output:**
- Every row is a table — this becomes your row list
- Status is auto-assigned based on DaysSinceLastLog — review it for each row and override manually if the auto-assignment does not match reality for that source type
- LastSeen_UTC and DaysSinceLastLog are your health indicators
- All columns starting with [Manual] need to be filled in after export
- The order is sorted by Status first so Inactive and Review sources appear at the top — deal with those first

---

## Step 2 — Shared Table Breakout Queries

Some tables receive data from multiple fundamentally different sources. The main query returns one row per table which is not granular enough for these. Run the breakout query for any shared table that appeared in Step 1.

**The three shared tables to always check:**
- `CommonSecurityLog` — firewalls, network devices, any CEF source
- `Syslog` — Linux machines, network appliances, any Syslog source
- `SecurityEvent` — Windows machines via AMA

**How to use breakout results:**
1. Find the single shared table row in your main query output
2. Delete that row
3. Paste the breakout rows in its place — one row per source type
4. Each breakout row already has the correct Table value
5. The LogSource column differentiates each one

---

### Breakout A — CommonSecurityLog

```kql
// ============================================================
// CommonSecurityLog BREAKOUT — One row per vendor and product
// Run if CommonSecurityLog appeared in the main query
// ============================================================
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
| extend LastSeen_UTC = format_datetime(LastSeen, 'yyyy-MM-dd HH:mm')
| extend Table = "CommonSecurityLog"
| extend LogSource = strcat(DeviceVendor, " — ", DeviceProduct)
| extend Vendor =     "[Manual] — Use DeviceVendor value as your guide"
| extend Category =   "[Manual] — Firewall / Network / Other"
| extend Purpose =    "[Manual] — What does this specific device log help detect?"
| extend Collection = "CEF Syslog"
| extend SilentDet =  "[Manual] — AB##### silent detection for this source"
| extend Detections = "[Manual] — AB-series detections relying on this source"
| extend Notes =      "[Manual] — Notes"
| project
    Table,
    Status,
    LastSeen_UTC,
    DaysSinceLastLog,
    LogSource,
    Vendor,
    Category,
    Purpose,
    Collection,
    SilentDet,
    Detections,
    Notes
| order by DeviceVendor asc
```

---

### Breakout B — Syslog

```kql
// ============================================================
// Syslog BREAKOUT — One row per host
// Run if Syslog appeared in the main query
// ============================================================
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
| extend LastSeen_UTC = format_datetime(LastSeen, 'yyyy-MM-dd HH:mm')
| extend Table = "Syslog"
| extend LogSource = HostName
| extend Vendor =     "[Manual] — e.g. Microsoft for AMA Linux, or specific appliance vendor"
| extend Category =   "[Manual] — Endpoint / Network / Identity / Other"
| extend Purpose =    "[Manual] — What does this host log help detect?"
| extend Collection = "AMA Agent"
| extend SilentDet =  "[Manual] — AB##### silent detection for this host"
| extend Detections = "[Manual] — AB-series detections relying on this host"
| extend Notes =      "[Manual] — Note the host role e.g. Domain Controller, CEF Forwarder, Application Server"
| project
    Table,
    Status,
    LastSeen_UTC,
    DaysSinceLastLog,
    LogSource,
    Vendor,
    Category,
    Purpose,
    Collection,
    SilentDet,
    Detections,
    Notes
| order by HostName asc
```

---

### Breakout C — SecurityEvent

```kql
// ============================================================
// SecurityEvent BREAKOUT — Summary view
// SecurityEvent is a single log source category — Windows
// machines via AMA. Do NOT create one row per machine.
// This query shows you which machines are enrolled so you
// can document the scope in Notes.
// ============================================================
SecurityEvent
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count(),
    MachineCount = dcount(Computer),
    Machines = make_set(Computer, 50)
    by Type
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
| extend LastSeen_UTC = format_datetime(LastSeen, 'yyyy-MM-dd HH:mm')
| extend Table = "SecurityEvent"
| extend LogSource = "Windows Security Events via AMA"
| extend Vendor =     "Microsoft"
| extend Category =   "Endpoint"
| extend Purpose =    "Detects authentication-based attacks, lateral movement, privilege escalation, and persistence mechanisms across Windows machines"
| extend Collection = "AMA Agent"
| extend SilentDet =  "[Manual] — AB##### silent detection for Windows security events"
| extend Detections = "[Manual] — AB-series detections relying on Windows security events"
| extend Notes =      strcat("[Manual — add any issues] Enrolled machines: ", tostring(MachineCount), " — ", tostring(Machines))
| project
    Table,
    Status,
    LastSeen_UTC,
    DaysSinceLastLog,
    LogSource,
    Vendor,
    Category,
    Purpose,
    Collection,
    SilentDet,
    Detections,
    Notes
```

**Note on SecurityEvent:** This query pre-fills most fields because SecurityEvent is always the same log source category regardless of environment. The Notes column includes the machine list so you have that context without creating separate rows. One row in the spreadsheet covers the entire SecurityEvent table.

---

## Step 3 — Usage Volume Query

Run this separately to get ingestion volume in GB for every table. Use it to fill in a reference column during manual field entry. This is separate from the main query because it sources from the Usage table not from the raw data.

```kql
// ============================================================
// USAGE VOLUME — Run separately
// Provides daily average ingestion in GB per table
// Use this alongside the main query output
// ============================================================
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize
    TotalGB = round(sum(Quantity) / 1000, 3),
    AvgDailyGB = round(sum(Quantity) / 1000 / 30, 4)
    by DataType
| extend DailyVolume = case(
    AvgDailyGB == 0,       "< 1 MB",
    AvgDailyGB < 0.001,    "< 1 MB",
    AvgDailyGB < 1,        strcat(tostring(round(AvgDailyGB * 1000, 1)), " MB"),
    strcat(tostring(round(AvgDailyGB, 2)), " GB")
)
| project
    Table = DataType,
    DailyVolume,
    TotalGB_30Days = TotalGB
| order by TotalGB_30Days desc
```

**How to use this:** Keep the Usage export open alongside your main spreadsheet. When filling in the Notes column for each table add the daily volume as context — for example "Daily Vol: 2.4 GB." This gives anyone reading the spreadsheet a sense of the data scale without it being a formal column. It also helps identify high-volume tables with no detections — a cost optimization signal.

---

## Steps 4-6 — Export, Format, and Save

**Step 4 — Export results**

Export each query result as CSV from the Logs blade export button. You will have:
- One CSV from the main table inventory query
- One CSV per shared table breakout you ran
- One CSV from the Usage volume query — keep this as a separate reference

**Step 5 — Open in Excel and merge correctly**

Open the main inventory CSV in Excel. Before pasting breakout rows:

1. Find the shared table row in the main output — for example the single `CommonSecurityLog` row
2. Delete that row entirely
3. Paste the breakout rows in its place — one row per source type
4. The Table column in the breakout rows already says `CommonSecurityLog` — correct
5. The LogSource column differentiates each source
6. Repeat for Syslog and SecurityEvent

**What the merged spreadsheet looks like:**

| Table | Status | LastSeen_UTC | ... | LogSource |
|---|---|---|---|---|
| CommonSecurityLog | Active | 2026-04-09 07:14 | ... | Palo Alto Networks — PAN-OS |
| CommonSecurityLog | Inactive | 2026-03-28 14:22 | ... | Fortinet — FortiGate |
| SecurityEvent | Active | 2026-04-09 07:18 | ... | Windows Security Events via AMA |
| Syslog | Active | 2026-04-09 06:55 | ... | LINUXSRV01 |
| SignInLogs | Active | 2026-04-09 07:21 | ... | [Manual] |

Do not add breakout rows alongside the original row — replace it. Duplicate rows will cause watchlist upload problems.

Once merged:
- Click any cell in the data
- Press **Ctrl+T**
- Confirm "My table has headers" is checked
- Click OK
- Go to Home → Format → AutoFit Column Width

This formats the data as an Excel Table with filtering and sorting on every column.

**Step 6 — Save and upload to SharePoint**

Save immediately as an Excel Workbook (.xlsx) using this naming convention:
```
[ClientName]-DataSourceAudit-[YYYY-MM-DD].xlsx
```

Upload to the client's SharePoint folder before filling in any manual fields. This is your unmodified baseline snapshot — a timestamped record of exactly what the workspace looked like on this date.

---

## Step 7 — Fill In Manual Fields

Every cell containing `[Manual]` must be replaced with the actual value or N/A. Do not leave placeholder text in any cell.

Use the pre-populated reference table below for common Microsoft sources — most values are already documented so you are validating not researching. For third-party and environment-specific sources you will need to research each one.

Reference the `resources/table-reference/` folder for any table that has a documented entry. The quick reference block at the top of each file gives you copy-paste values for every manual field.

---

### Pre-Populated Reference — Common Sources

| Table | LogSource | Vendor | Category | Collection | Purpose |
|---|---|---|---|---|---|
| SignInLogs | Microsoft Entra ID Interactive Sign-in Logs | Microsoft | Identity | Microsoft Connector | Detects compromised accounts, password spray, impossible travel, MFA fatigue, and unauthorized access |
| AADNonInteractiveUserSignInLogs | Microsoft Entra ID Non-Interactive Sign-in Logs | Microsoft | Identity | Microsoft Connector | Detects service-to-service auth abuse, token theft, and background authentication anomalies |
| AADServicePrincipalSignInLogs | Microsoft Entra ID Service Principal Sign-in Logs | Microsoft | Identity | Microsoft Connector | Detects abuse of service principals and app registrations — a frequent attacker pivot point |
| AADManagedIdentitySignInLogs | Microsoft Entra ID Managed Identity Sign-in Logs | Microsoft | Identity | Microsoft Connector | Detects abuse of managed identities used by Azure resources to authenticate to other services |
| AuditLogs | Microsoft Entra ID Audit Logs | Microsoft | Identity | Microsoft Connector | Detects directory changes — role assignments, group membership, app registrations, admin operations |
| AADRiskyUsers | Microsoft Entra ID Identity Protection — Risky Users | Microsoft | Identity | Microsoft Connector | Surfaces accounts flagged by Microsoft ML as compromised — leaked credentials, impossible travel |
| AADUserRiskEvents | Microsoft Entra ID Identity Protection — Risk Events | Microsoft | Identity | Microsoft Connector | Specific risk detections per sign-in — feeds UEBA and risk-based detections. Requires Entra ID P2. |
| OfficeActivity | Microsoft 365 Unified Audit Log | Microsoft | Compliance and Audit | Microsoft Connector | Detects data exfiltration via SharePoint and OneDrive, mailbox access, email forwarding rules, OAuth consent abuse |
| AzureActivity | Azure Management Plane Activity | Microsoft | Cloud Infrastructure | Microsoft Connector | Detects unauthorized Azure resource changes — role assignments, VM creation, firewall rule changes, Key Vault access |
| SecurityAlert | Microsoft Security Alerts — All Products | Microsoft | Cloud Infrastructure | Microsoft Connector | Consolidated alert feed from all connected Microsoft security products |
| SecurityIncident | Microsoft Sentinel and XDR Incidents | Microsoft | Cloud Infrastructure | Microsoft Connector | All incidents including XDR-correlated multi-stage incidents — primary pipeline source for ServiceNow tickets |
| BehaviorAnalytics | UEBA Behavioral Analytics Output | Microsoft | Capabilities | Microsoft Connector | ML-generated behavioral anomaly scores — requires 14-21 day baseline period before reliable output |
| IdentityInfo | UEBA Identity Information | Microsoft | Capabilities | Microsoft Connector | Enriched user identity data from Entra ID and on-prem AD — used by detections for user context |
| ThreatIntelIndicators | Microsoft Defender Threat Intelligence — Indicators | Microsoft | Threat Intelligence | Microsoft Connector | Current IOC feed — malicious IPs, domains, URLs, file hashes matched against ingested log data |
| ThreatIntelObjects | Microsoft Defender Threat Intelligence — STIX Objects | Microsoft | Threat Intelligence | Microsoft Connector | STIX 2.1 structured threat intelligence — threat actors, attack patterns, campaign relationships |
| Heartbeat | Azure Monitor Agent Heartbeat | Microsoft | Endpoint | AMA Agent | One heartbeat per minute per AMA-connected machine — primary mechanism for detecting agent and machine availability |
| SentinelHealth | Microsoft Sentinel Operational Health | Microsoft | Compliance and Audit | Microsoft Connector | Health status of analytics rules, connectors, and automation — critical for detecting silent failures |
| SentinelAudit | Microsoft Sentinel Configuration Audit | Microsoft | Compliance and Audit | Microsoft Connector | Records all configuration changes to the Sentinel workspace — who changed what and when |
| SecurityEvent | Windows Security Events via AMA | Microsoft | Endpoint | AMA Agent | Detects authentication attacks, lateral movement, privilege escalation, and persistence across Windows machines |
| CommonSecurityLog | [DeviceVendor] — [DeviceProduct] | [Vendor] | Firewall | CEF Syslog | Perimeter and network device logs — traffic patterns, connection events, threat signatures |
| Syslog | [HostName] | [Vendor] | Endpoint | AMA Agent | Linux system logs — varies by host role. Domain controllers, servers, and appliances each have different detection value |

---

## Step 8 — Complete Tab 2 Configuration Reference

Create a second tab in the Excel workbook named **Configuration**. Fill in the technical details for each log source. This tab is engineer-facing only — it documents the exact wiring behind each source so any engineer can troubleshoot without having to rediscover the configuration.

**Tab 2 columns:**

| Column | What Goes Here |
|---|---|
| Table | Exact table name — must match Tab 1 exactly |
| LogSource | Log source — must match Tab 1 exactly |
| Data Connector | Name of the Sentinel data connector if applicable. Example: Microsoft Defender XDR, Microsoft Entra ID, Windows Security Events via AMA |
| Connector Status | Connected / Disconnected / Not Applicable |
| DCR Name | Name of the Data Collection Rule if this source uses AMA. Find in Azure portal → Monitor → Data Collection Rules |
| DCE Name | Name of the Data Collection Endpoint if applicable. Find alongside DCRs in Azure Monitor |
| Diagnostic Setting | Name of the diagnostic setting if this is an Azure resource log. Find in the source resource → Diagnostic Settings |
| Subscription | Azure subscription name where this source lives |
| Workspace | Log Analytics workspace name receiving this data |
| License Required | What license is needed. Example: Entra ID P2, Microsoft 365 E5, Defender for Identity |
| Notes | Technical notes, known issues, configuration details, last verified date |

**Filling in Tab 2 efficiently:**
- Microsoft Connector sources — check the Data Connectors page in Sentinel for connector name and status. Diagnostic setting name comes from the source service.
- AMA Agent sources — DCR name comes from Azure Monitor → Data Collection Rules. Filter by the workspace to find relevant rules.
- CEF Syslog sources — the forwarder VM is the key detail. Document the forwarder VM name and IP in Notes.
- REST API and Logic App sources — document the Logic App name and the API endpoint in Notes.

---

## Step 9 — Phase 3 Verification

For every source marked Active — verify configuration is complete and correct. Data flowing does not mean everything is configured correctly. A source can be partially configured and appear healthy while having significant blind spots.

---

### Verify Silent Detection Health

Every log source must have a silent detection monitoring it. Run this to check which AB-series detections exist and are healthy:

```kql
// Silent Detection Health — All Custom AB-series Rules
SentinelHealth
| where SentinelResourceType == "Analytics Rule"
| where SentinelResourceName matches regex @"^AB\d+"
| summarize LastStatus = arg_max(TimeGenerated, Status, Description)
    by SentinelResourceName
| project
    DetectionName = SentinelResourceName,
    HealthStatus = Status,
    LastChecked = TimeGenerated,
    Description
| order by HealthStatus asc
```

Cross-reference the results against your SilentDet column. Any log source with no matching detection needs one created — mark SilentDet as Missing in the spreadsheet and add a Notes action item.

---

### Verify Microsoft Connector Log Category Completeness

Enabling a Microsoft connector is not enough. Each log category must be explicitly configured in diagnostic settings. Verify each expected table is receiving data:

```kql
// Template — replace TABLE_NAME with each expected table
TABLE_NAME
| where TimeGenerated > ago(24h)
| summarize Count = count(), LastSeen = max(TimeGenerated)
```

**Minimum required Entra ID tables — verify each:**
- `SignInLogs`
- `AADNonInteractiveUserSignInLogs`
- `AADServicePrincipalSignInLogs`
- `AADManagedIdentitySignInLogs`
- `AuditLogs`
- `AADRiskyUsers` — requires Entra ID P2
- `AADUserRiskEvents` — requires Entra ID P2

If any return zero results the diagnostic setting for that category is not configured. Navigate to Azure portal → Entra ID → Monitoring → Diagnostic Settings to verify and correct.

---

### Verify AMA Agent Sources — Heartbeat Health

Confirm every AMA-connected machine is sending heartbeats. Any machine missing heartbeats should be flagged immediately:

```kql
// Heartbeat Coverage — All AMA Machines
Heartbeat
| where TimeGenerated > ago(24h)
| summarize LastHeartbeat = max(TimeGenerated) by Computer, OSType, Category
| extend HoursSince = datetime_diff('hour', now(), LastHeartbeat)
| extend HeartbeatStatus = iff(HoursSince <= 1, "Healthy", "Missing")
| project Computer, OSType, Category, LastHeartbeat, HoursSince, HeartbeatStatus
| order by HoursSince desc
```

---

### Verify CEF Sources — Forwarder Health

For CommonSecurityLog sources, verify each vendor is actively sending:

```kql
// CEF Source Verification
// Replace VENDOR_NAME and PRODUCT_NAME with actual values
CommonSecurityLog
| where TimeGenerated > ago(24h)
| where DeviceVendor == "VENDOR_NAME"
| where DeviceProduct == "PRODUCT_NAME"
| summarize Count = count(), LastSeen = max(TimeGenerated)
```

If this returns zero while the main query showed historical data — the Syslog forwarder stopped receiving from this device. Investigate:
1. Confirm the forwarder VM is running
2. Confirm AMA on the forwarder is healthy — check Heartbeat table
3. Confirm the device is still configured to send Syslog to the forwarder IP
4. Confirm the forwarder is listening on the correct port

---

## Steps 10-11 — Document and Sort

**Step 10 — Document all non-Active sources**

Every row with Status of Inactive, Review, No Data, or Missing must have a Notes entry containing:
- What the issue appears to be
- What the next step is
- Who owns the follow-up
- Target resolution date

A row cannot be signed off without a documented action item if its status is not Active.

**Step 11 — Sort alphabetically**

Before uploading as a watchlist sort the spreadsheet:
1. Click any cell in the data
2. Go to Data → Sort
3. Sort by Table (A to Z) first
4. Add a second level — sort by LogSource (A to Z)

This groups all CommonSecurityLog rows together, all Syslog rows together, and within each table keeps log sources in alphabetical order. The watchlist will maintain this order making it scannable and predictable for anyone querying it.

---

## Step 12 — Upload Tab 1 as Watchlist

Tab 1 of the completed spreadsheet becomes the DataSourceInventory watchlist in Sentinel.

**Upload steps:**
1. In the Sentinel or Defender portal navigate to Watchlists
2. Create a new watchlist — name it `DataSourceInventory`
3. Upload the Tab 1 data as a CSV — export Tab 1 only as CSV for upload
4. Set the SearchKey to the `Table` column
5. Confirm row count matches your spreadsheet

**Validate the watchlist loaded correctly:**
```kql
_GetWatchlist('DataSourceInventory')
| summarize TotalRows = count()
```

---

## Known Limitations

**Lighthouse access scope**
Access via Azure Lighthouse is scoped to the Sentinel workspace and Log Analytics. Depending on delegated roles the following may not be accessible:
- Azure VM extensions — cannot verify AMA agent installation directly
- Azure Resource tags — cannot query resource classification tags
- Azure Policy — cannot verify tagging policies
- Azure Resource Graph — cannot pull full resource inventory

Workaround: Heartbeat queries confirm AMA machine health without VM extension access. Document gaps in Notes and flag for the client IT team.

**Tables that never received data**
Log Analytics only registers a table after at least one record is written. A connector enabled but never successfully sending data will not appear in any query. These sources appear as Missing when compared against expected tables. Document in Tab 2 Notes with the connector status.

**Shared table field reliability**
The breakout queries for CommonSecurityLog, Syslog, and SecurityEvent depend on fields like DeviceVendor, HostName, and Computer being correctly populated. Poorly configured sources may have blank or inconsistent values. Document these in Notes and investigate the source configuration.

---

## Onboarding Note

This audit process is designed to be repeatable and eventually automated. The completed watchlist becomes the baseline that drives the living workbook, the change detection analytics rules, and the automated Teams notifications. In the future this process will be part of client onboarding so the watchlist exists from day one. See `PROGRAM-MAP.md` for the full automation pipeline design.

---

## Audit Sign-Off

Add this block to the bottom of Tab 1:

```
Audit Completed By:      [Engineer Name]
Date Completed:          [YYYY-MM-DD]
Client:                  [Client Name]
Total Log Sources:       [Number]
Active:                  [Number]
Inactive:                [Number]
Review:                  [Number]
No Data:                 [Number]
Missing:                 [Number]
Decom:                   [Number]
Silent Det Coverage:     [X of Y log sources have a documented silent detection]
Next Audit Due:          [YYYY-MM-DD]
Notes:                   [Overall observations]
Watchlist Uploaded:      Yes / No
SharePoint Backup:       Yes / No — [link]
Tab 2 Complete:          Yes / No
```

No log source may be signed off with a non-Active status without a documented action item and owner in the Notes column.

---

*This runbook is maintained in sentinel-mssp-playbook under 02-data-sources. Update it when the audit process changes, new source types require guidance, or new shared tables are identified. Version and date any significant changes.*