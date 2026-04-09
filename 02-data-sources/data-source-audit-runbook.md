# Data Source Audit Runbook

> **Purpose:** This runbook guides an engineer through a complete data source audit for a managed client Microsoft Sentinel environment. The output is a two-tab Excel workbook that becomes the foundation for the data source watchlist, the living workbook, and all ongoing health monitoring.
>
> **Who uses this:** Engineers conducting a first-time audit or a periodic re-audit of a client environment.
>
> **What you produce:** A completed, sorted, two-tab Excel workbook. Tab 1 becomes the Sentinel watchlist. Tab 2 is the technical configuration reference.
>
> **Prerequisites:** Complete `01-baseline/license-entitlement-review.md` for this client before starting. Knowing what the client is licensed for tells you what data sources should exist and what capabilities should be enabled.
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

Microsoft Sentinel presents data sources through a connector UI that makes it easy to assume that if a connector shows as connected everything is working. This assumption is dangerous and wrong.

A connector is a configuration wizard. It helps you enable a data source but it does not guarantee data is actually flowing, that all log categories are enabled, that the correct tables are being written to, or that volume is healthy. A connector can show green while silently sending nothing.

Tables are closer to the truth but still not granular enough. A table like `CommonSecurityLog` or `Syslog` can have many different sources writing to it simultaneously. If a Palo Alto firewall stops sending logs but a Fortinet device keeps writing — `CommonSecurityLog` still appears Active. The Palo Alto source is blind and you will never know from the table level alone.

**The log source is the correct audit unit.** One row per originating source category. One status per source. This is the only level of granularity that catches individual source failures before they cause missed detections.

**Important distinction — log source category vs individual device:**
A log source is not every individual machine or device. It is the category of thing writing to the table. Fifty Windows machines sending data to SecurityEvent via AMA is one log source — Windows Security Events via AMA. We only create separate rows when fundamentally different things are writing to the same table — for example a Palo Alto firewall and a Fortinet VPN both writing to CommonSecurityLog are two different log sources because they are different vendors with different purposes and different detection value.

---

## Log Source Inclusion Criteria

Not every source within a shared table deserves its own row in the watchlist. Documenting everything you find in a breakout query creates noise, adds maintenance burden, and makes the workbook harder to read — with no security return.

**The core rule:**

> A log source gets its own row only if losing it would matter.

Before creating a row for any source ask yourself — if this specific source went silent tomorrow would anyone care? Would a detection go blind? Would a capability degrade? Would a compliance obligation go unmet?

If the answer is no to all of those — it does not need a row.

**A source earns a row if it meets at least one of these criteria:**

- A detection actively queries this source — losing it creates a blind spot
- A capability depends on it — UEBA, Threat Intelligence, Fusion ML
- A compliance obligation requires it to be continuously monitored
- It has meaningful investigation or threat hunting value
- It is a custom integration that could break silently without obvious indication

**A source does not need a row if:**

- No detection queries it
- No compliance obligation requires it
- It has no investigation value
- Losing it would not change your detection coverage or create a gap
- It is a metric or counter rather than a security event

**Practical examples of what to skip:**

- AzureDiagnostics from Logic Apps — workflows/runs/actions, workflows/runs/triggers. Unless you have detections querying Logic App diagnostic logs or a compliance requirement, a brief note in the Notes column of the relevant AzureActivity row is enough. No dedicated rows needed.
- AzureDiagnostics NSG rule counters — NetworkSecurityGroupRuleCounter. This is a metric counter not a security event. Document NSG Flow Events if present but skip the counter.
- CommonSecurityLog from a vendor sending only system health events — if those events feed no detection and have no hunting value, skip the row.
- Syslog from a host sending only cron or kernel messages with no auth facility — low value, no detection dependency, skip it.

**What to do with sources you are skipping:**

Do not just ignore them. If a breakout query returns a source you are not documenting as its own row add a brief note to the parent table Notes column — for example "AzureDiagnostics also receives Logic App runtime logs — no detection dependency, not tracked." This tells future engineers that you saw it and made a deliberate decision not to track it rather than missing it.

**The goal:**

A lean, intentional watchlist where every row earns its place. When someone opens the workbook and sees a row they should be able to immediately understand why that source is being tracked and what breaks if it goes silent. If you cannot answer that question for a row it probably should not be there.

---

## The Spreadsheet Structure

The audit produces a two-tab Excel workbook. Understanding both tabs before you start is important.

### Tab 1 — Data Sources
The operational view. Becomes the Sentinel watchlist that powers the living workbook. Every row is a log source. Keep it clean, accurate, and purposeful. This is what drives the workbook, the watchlist, and the client conversation.

**Final column set — in this exact order:**

| # | Column | What Goes Here |
|---|---|---|
| 1 | Table | Exact Log Analytics table name — never modify |
| 2 | Status | Current health — plain text from approved list. Stored in watchlist as audit snapshot. Workbook recalculates live from workspace queries — do not rely on watchlist Status in workbook logic. |
| 3 | LastSeen | Raw datetime of most recent record. No formatting — raw datetime value only. Excel formats the column for display. Workbook queries wrap with todatetime() when reading from watchlist. |
| 4 | DaysSinceLastLog | Days since last record — calculated by query for audit reference only. Do NOT include in watchlist upload. Workbook calculates this live from LastSeen at query time. |
| 5 | LogSource | Plain English description of what writes to this table |
| 6 | Origin | What generated this data — from approved list |
| 7 | Transport | How data gets into Sentinel — from approved list |
| 8 | Category | Type of security data — from approved list |
| 9 | Purpose | One sentence — what attack or risk does this detect |
| 10 | SilentDet | AB##### of the silent detection monitoring this source |
| 11 | Detections | AB##### numbers of detections that depend on this source |
| 12 | Notes | Issues, action items, context, exceptions |

### Tab 2 — Configuration
Technical reference. Engineer-facing only. Never shown to clients. Documents the exact technical wiring behind each log source. Linked to Tab 1 by Table and LogSource.

| Column | What Goes Here |
|---|---|
| Table | Exact table name — matches Tab 1 |
| LogSource | Log source — matches Tab 1 |
| Data Connector | Sentinel data connector name if applicable |
| Connector Status | Connected / Disconnected / Not Applicable |
| DCR Name | Data Collection Rule name if AMA-based |
| DCE Name | Data Collection Endpoint name if applicable |
| Diagnostic Setting | Diagnostic setting name if Azure resource log |
| Subscription | Azure subscription name |
| Workspace | Log Analytics workspace name |
| License Required | License needed for this source |
| Notes | Technical notes, issues, last verified date |

---

## Approved Field Values

### Status — Plain Text Only
No emojis. Plain text is watchlist-compatible and cleanly queryable in KQL. Emojis in watchlist field values cause matching problems in KQL queries.

| Status | Meaning | Who Assigns |
|---|---|---|
| Active | Data flowing within expected timeframe. No action needed. | Query |
| Inactive | Had data, now silent. Investigate immediately. Document cause in Notes. | Query |
| Review | Data flowing but something warrants attention. Volume anomaly, partial config, or inconsistent timestamps. | Query or Manual override |
| No Data | Never received data. Document in Notes whether not applicable or never configured. | Manual |
| Missing | Expected based on connector or license but not found in workspace. Connector may be enabled but never sent data. | Manual |
| Decom | Marked for retirement. Do not delete — mark Decom with date and reason in Notes. | Manual only |

**Note on Flag status:** Flag is reserved for the future automation pipeline. When the snapshot Logic App detects a new table between snapshots it will assign Flag automatically. During a manual audit you classify everything as you go — you will never assign Flag manually. It is documented here so engineers understand it when they encounter it in the future.

**Status guidance by source type:**
- Identity sources — data expected every few minutes. More than 4 hours → Review. More than 24 hours → Inactive.
- Endpoint sources — depend on device activity. No data 48 hours in business environment → Inactive.
- Firewall and network — near-continuous. More than 6 hours → Review. More than 24 hours → Inactive.
- Threat intelligence — may update daily or less. Check vendor cadence before marking Inactive.
- UEBA output tables — continuous once past the 14-21 day baseline period.

---

### Origin — What Generated This Data

| Origin | Use For |
|---|---|
| Microsoft Entra ID | All identity and access logs — sign-ins, audit, identity protection, provisioning |
| Microsoft 365 | All M365 workload logs — SharePoint, OneDrive, Exchange, Teams, admin activity |
| Microsoft Defender | All Defender product logs — MDE, MDO, MDI, MDA, MDC |
| Microsoft Azure | Azure Activity, resource diagnostics, Azure Monitor, storage, Key Vault |
| Microsoft Sentinel | Incidents, alerts, health, audit, UEBA output, threat intelligence |
| Windows OS | Windows Security Event Log generated by Windows machines |
| Network Device | Firewalls, routers, switches, VPN concentrators |
| AWS | Amazon Web Services logs |
| Salesforce | Salesforce platform logs |
| Zoom | Zoom meeting and user activity logs |
| Keeper Security | Keeper PAM and password manager logs |
| SecureW2 | SecureW2 certificate authentication and NAC logs |
| Drupal | Drupal CMS application logs |
| Multiple Sources | ASIM tables fed by more than one origin |
| Custom | Built by your team — no external origin |
| Unknown | Not yet identified — investigate and update |

---

### Transport — How Data Gets Into Sentinel

| Transport | What It Means | When to Use |
|---|---|---|
| Microsoft Connector | Native Microsoft service-to-service. No agent or DCR needed. Configured in Sentinel data connectors page. | Entra ID, Office 365, Azure Activity, Defender for Cloud |
| XDR Connector | Flows through the Microsoft Defender XDR unified connector. Requires XDR connector enabled in Sentinel. | DeviceEvents, EmailEvents, CloudAppEvents, IdentityLogonEvents |
| AMA — DCR | Azure Monitor Agent on a host shipping via a Data Collection Rule. Both AMA and DCR must be healthy. | SecurityEvent, Syslog, Heartbeat, Perf |
| AMA — DCR — DCE | AMA shipping via DCR with a custom Data Collection Endpoint. Used for specific routing requirements. | Custom AMA configurations |
| CEF — AMA — DCR | Network device sends CEF/Syslog to a Linux forwarder VM. Forwarder runs AMA and ships via DCR. Three components must all be healthy. | CommonSecurityLog from firewalls |
| Cribl — AMA — DCR | Logs routed through Cribl for filtering and transformation then shipped via AMA and DCR. | Varies by environment |
| Cribl — DCR | Logs routed through Cribl sent directly via DCR without AMA. | Varies by environment |
| Diagnostic Setting | Azure resource diagnostic setting routing logs directly to the workspace. Configured per resource in Azure portal. | StorageBlobLogs, AzureDiagnostics, KeyVaultData |
| Logic App — Polling | Logic App running on a schedule polling an API and ingesting results. | KeeperLogs_CL, Zoom_CL, Salesforce tables |
| Logic App — Webhook | Logic App triggered by an incoming webhook from an external source. | Varies |
| REST API — Push | External source pushing data directly via the Log Ingestion API. | Custom_CL tables |
| TAXII — Polling | Threat intelligence connector polling a TAXII server for IOC feeds. | ThreatIntelIndicators, ThreatIntelObjects |
| Sentinel Native | Produced internally by Sentinel itself. No external transport. | SentinelHealth, SentinelAudit, SecurityIncident, SecurityAlert |
| ASIM Parser | Data normalized from raw tables at query or ingestion time. Health depends entirely on source table health. | ASimNetworkSessionLogs, ASimAuditEventLogs |

**Why Transport detail matters:**
When something breaks you go to a completely different place depending on the transport:
- AMA — DCR broken → Check Azure Monitor → Data Collection Rules. Check AMA extension on host. Check Heartbeat.
- CEF — AMA — DCR broken → Check forwarder VM, then AMA on forwarder, then firewall Syslog destination. Three places.
- Logic App — Polling broken → Check Logic App run history. Check API credentials expiry.
- Microsoft Connector broken → Check connector page in Sentinel. Check diagnostic settings in source service.
- ASIM Parser degraded → Check the source tables it feeds from. Fix the source, ASIM recovers.

---

### Category — Security Domain

| Category | Use For |
|---|---|
| Identity | User authentication, directory services, account management, sign-in activity |
| Endpoint | Device telemetry, EDR, antivirus, host-based security events |
| Email | Mail flow, phishing detection, attachment and URL scanning |
| Network | Traffic logs, DNS, DHCP, proxy, network flow data |
| Firewall | Perimeter firewall and next-generation firewall logs |
| Cloud Infrastructure | Azure resource logs, management plane, cloud workload protection |
| SaaS Application | Third-party cloud applications — Salesforce, Zoom, Drupal |
| Threat Intelligence | IOC feeds, indicator matching, threat actor data |
| Compliance and Audit | M365 audit logs, Sentinel health and audit, regulatory compliance |
| Vulnerability Management | Scanner results, CVE data, asset risk scoring |
| Authentication | MFA, SSO, PAM, privileged access management |
| Capabilities | Tables produced by capabilities — UEBA output, Fusion ML. Not raw inputs. |
| Other | Does not fit above — explain in Notes |

---

### Detections — Format

AB numbers only. No detection names in this field. The detection library has the full names. The watchlist needs the IDs so KQL can match against them cleanly.

**Correct:** `AB00034, AB00041, AB00055`
**Incorrect:** `AB00034 - Impossible Travel Detection, AB00041 - Legacy Auth Detection`

If no custom detections depend on this source write `None`.

---

### Purpose — Format

One sentence maximum. Written for a business audience not a technical one. Answers the question: what attack or risk would be invisible if this data source disappeared?

**Good:** `Detects compromised accounts, password spray attacks, impossible travel, and MFA fatigue across all user sign-in activity.`
**Poor:** `This table contains sign-in logs.`

---

## The Audit Workflow

Work through all steps in order. Do not skip ahead.

- [ ] **Step 0** — Complete license-entitlement-review.md for this client
- [ ] **Step 1** — Run the main table inventory query
- [ ] **Step 2** — Run shared table breakout queries where applicable
- [ ] **Step 3** — Run the Usage volume query
- [ ] **Step 4** — Export all results and open in Excel
- [ ] **Step 5** — Merge breakout rows correctly and format as Excel Table
- [ ] **Step 6** — Save as dated workbook and upload to SharePoint
- [ ] **Step 7** — Fill in all manual fields
- [ ] **Step 8** — Complete Tab 2 configuration reference
- [ ] **Step 9** — Run Phase 3 verification for every Active source
- [ ] **Step 10** — Document all non-Active sources in Notes
- [ ] **Step 11** — Sort alphabetically by Table then LogSource
- [ ] **Step 12** — Upload Tab 1 as the DataSourceInventory watchlist

---

## Step 1 — Main Table Inventory Query

Run this in the Microsoft Sentinel Logs blade. Returns every table that has ever received data including tables that went silent months ago.

**First run instruction:** Remove the time filter entirely on the first audit for a client. You want every table that has ever existed. On subsequent audits keep the 30-day filter.

**Status values are plain text** — no emojis. Watchlist compatible and cleanly queryable.

```kql
// ============================================================
// MAIN TABLE INVENTORY — Run first. One row per table.
// FIRST RUN: Remove the time filter line entirely
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
// ── Manual fields — fill these in after export ───────────────
| extend LogSource =  "[Manual] — Plain English description e.g. Windows Security Events via AMA, Microsoft Entra ID Sign-in Logs"
| extend Origin =     "[Manual] — Microsoft Entra ID / Microsoft 365 / Microsoft Defender / Microsoft Azure / Microsoft Sentinel / Windows OS / Network Device / AWS / Salesforce / Zoom / Keeper Security / SecureW2 / Drupal / Multiple Sources / Custom / Unknown"
| extend Transport =  "[Manual] — Microsoft Connector / XDR Connector / AMA — DCR / AMA — DCR — DCE / CEF — AMA — DCR / Cribl — AMA — DCR / Cribl — DCR / Diagnostic Setting / Logic App — Polling / Logic App — Webhook / REST API — Push / TAXII — Polling / Sentinel Native / ASIM Parser"
| extend Category =   "[Manual] — Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other"
| extend Purpose =    "[Manual] — One sentence: what attack or risk does this data help detect? Write for a business audience."
| extend SilentDet =  "[Manual] — AB##### or Missing if none exists"
| extend Detections = "[Manual] — AB#####, AB##### or None"
| extend Notes =      "[Manual] — Issues, action items, context. Date stamp action items."
// ─────────────────────────────────────────────────────────────
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

**Reading the output:**
- Every row is a table — this is your row list
- Status is auto-assigned based on DaysSinceLastLog — override manually if needed for the source type
- Sorted by Status first so Inactive and Review appear at the top — deal with those first
- All [Manual] fields need to be filled in after export

---

## Step 2 — Shared Table Breakout Queries

Some tables receive data from multiple fundamentally different sources. The main query returns one row per table which is not granular enough. Run the breakout for any shared table that appeared in Step 1.

**The three shared tables to always check:**
- `CommonSecurityLog` — firewalls, network devices, any CEF source
- `Syslog` — Linux machines, network appliances, any Syslog source
- `SecurityEvent` — Windows machines via AMA

**How to use breakout results:**
1. Find the single shared table row in your main query output
2. Delete that row entirely
3. Paste the breakout rows in its place
4. Each breakout row already has the correct Table value
5. LogSource differentiates each source

---

### Breakout A — CommonSecurityLog

```kql
// CommonSecurityLog BREAKOUT — One row per vendor and product
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
| extend Table = "CommonSecurityLog"
| extend LogSource = strcat(DeviceVendor, " — ", DeviceProduct)
| extend Origin =     "[Manual] — Network Device or specific vendor name"
| extend Transport =  "CEF — AMA — DCR"
| extend Category =   "[Manual] — Firewall / Network / Other"
| extend Purpose =    "[Manual] — What does this specific device log help detect?"
| extend SilentDet =  "[Manual] — AB##### or Missing"
| extend Detections = "[Manual] — AB#####, AB##### or None"
| extend Notes =      "[Manual] — Document forwarder VM name and IP"
| project
    Table,
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
| order by DeviceVendor asc
```

---

### Breakout B — Syslog

```kql
// Syslog BREAKOUT — One row per host
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
| extend Table = "Syslog"
| extend LogSource = HostName
| extend Origin =     "[Manual] — Windows OS for AMA Linux / specific appliance vendor"
| extend Transport =  "AMA — DCR"
| extend Category =   "[Manual] — Endpoint / Network / Identity / Other"
| extend Purpose =    "[Manual] — What does this host log help detect?"
| extend SilentDet =  "[Manual] — AB##### or Missing"
| extend Detections = "[Manual] — AB#####, AB##### or None"
| extend Notes =      "[Manual] — Note host role e.g. Domain Controller, CEF Forwarder, App Server"
| project
    Table,
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
| order by HostName asc
```

---

### Breakout C — SecurityEvent

```kql
// SecurityEvent BREAKOUT
// SecurityEvent is a single log source category — Windows machines via AMA
// Do NOT create one row per machine
// This query shows enrollment scope for Notes documentation
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
| extend Table =      "SecurityEvent"
| extend LogSource =  "Windows Security Events via AMA"
| extend Origin =     "Windows OS"
| extend Transport =  "AMA — DCR"
| extend Category =   "Endpoint"
| extend Purpose =    "Detects authentication attacks, lateral movement, privilege escalation, and persistence across Windows machines"
| extend SilentDet =  "[Manual] — AB##### or Missing"
| extend Detections = "[Manual] — AB#####, AB##### or None"
| extend Notes = strcat("[Manual — add issues] Enrolled machines: ", tostring(MachineCount), " — verify critical Event IDs in Phase 3")
| project
    Table,
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
```

**Note:** SecurityEvent pre-fills most fields because it is always the same log source regardless of environment. One row covers the entire table.

---

## Step 3 — Usage Volume Query

Run separately to get ingestion volume in GB per table. Keep the export open alongside your spreadsheet and add volume to the Notes column for context.

```kql
// USAGE VOLUME — Run separately
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

Add daily volume to the Notes column for each table — for example `Daily Vol: 2.4 GB`. This surfaces high-volume tables with no detections as cost optimization candidates.

---

## Steps 4-6 — Export, Format, and Save

**Step 4 — Export results**
Export each query from the Logs blade using the Export button. You will have up to four CSVs — main inventory, shared table breakouts, and Usage volume.

**Step 5 — Open in Excel and merge correctly**

Open the main inventory CSV. For each shared table:
1. Find and delete the single shared table row from the main output
2. Paste the breakout rows in its place
3. Table column in breakout rows is already correct
4. LogSource column differentiates each source

Once merged:
- Click any cell in the data
- Press **Ctrl+T**
- Confirm My table has headers is checked
- Click OK
- Go to Home → Format → AutoFit Column Width
- Select the LastSeen column → right click → Format Cells → Custom → type `yyyy-mm-dd hh:mm` → OK

This formats LastSeen for human readability while keeping the underlying raw datetime value intact. Excel date functions and downstream queries can still use it correctly. Never convert LastSeen to a text string.

**Step 6 — Save and upload to SharePoint**

```
[ClientName]-DataSourceAudit-[YYYY-MM-DD].xlsx
```

Upload to the client SharePoint folder before filling in any manual fields. This is your unmodified baseline snapshot.

---

## Step 7 — Fill In Manual Fields

Every [Manual] placeholder must be replaced with the actual value or N/A. Use the pre-populated reference below for common Microsoft sources. Use `resources/table-reference/` for any table with a documented entry.

---

### Pre-Populated Reference — Common Sources

| Table | LogSource | Origin | Transport | Category | Purpose |
|---|---|---|---|---|---|
| SignInLogs | Microsoft Entra ID Interactive Sign-in Logs | Microsoft Entra ID | Microsoft Connector | Identity | Detects compromised accounts, password spray, impossible travel, and MFA fatigue |
| AADNonInteractiveUserSignInLogs | Microsoft Entra ID Non-Interactive Sign-in Logs | Microsoft Entra ID | Microsoft Connector | Identity | Detects service-to-service auth abuse, token theft, and background authentication anomalies |
| AADServicePrincipalSignInLogs | Microsoft Entra ID Service Principal Sign-in Logs | Microsoft Entra ID | Microsoft Connector | Identity | Detects abuse of service principals and app registrations — a frequent attacker pivot point |
| AADManagedIdentitySignInLogs | Microsoft Entra ID Managed Identity Sign-in Logs | Microsoft Entra ID | Microsoft Connector | Identity | Detects abuse of managed identities used by Azure resources to authenticate to other services |
| AuditLogs | Microsoft Entra ID Audit Logs | Microsoft Entra ID | Microsoft Connector | Identity | Detects directory changes — role assignments, group membership, app registrations, admin operations |
| AADRiskyUsers | Microsoft Entra ID Identity Protection — Risky Users | Microsoft Entra ID | Microsoft Connector | Identity | Surfaces accounts flagged by Microsoft ML as compromised — leaked credentials, impossible travel |
| AADUserRiskEvents | Microsoft Entra ID Identity Protection — Risk Events | Microsoft Entra ID | Microsoft Connector | Identity | Specific risk detections per sign-in — requires Entra ID P2 license |
| OfficeActivity | Microsoft 365 Unified Audit Log | Microsoft 365 | Microsoft Connector | Compliance and Audit | Detects data exfiltration via SharePoint and OneDrive, mailbox access, email forwarding rules, OAuth consent abuse |
| AzureActivity | Azure Management Plane Activity | Microsoft Azure | Microsoft Connector | Cloud Infrastructure | Detects unauthorized Azure resource changes — role assignments, VM creation, firewall modifications, Key Vault access |
| SecurityAlert | Microsoft Security Alerts — All Products | Microsoft Sentinel | Microsoft Connector | Cloud Infrastructure | Consolidated alert feed from all connected Microsoft security products |
| SecurityIncident | Microsoft Sentinel and XDR Incidents | Microsoft Sentinel | Sentinel Native | Cloud Infrastructure | All incidents including XDR-correlated multi-stage incidents — primary pipeline source for ServiceNow tickets |
| BehaviorAnalytics | UEBA Behavioral Analytics Output | Microsoft Sentinel | Sentinel Native | Capabilities | ML-generated behavioral anomaly scores — requires 14-21 day baseline period |
| IdentityInfo | UEBA Identity Information | Microsoft Sentinel | Sentinel Native | Capabilities | Enriched user identity data from Entra ID and on-prem AD — used by detections for user context |
| ThreatIntelIndicators | Microsoft Defender Threat Intelligence Indicators | Microsoft Sentinel | TAXII — Polling | Threat Intelligence | Current IOC feed — malicious IPs, domains, URLs, file hashes matched against ingested log data |
| ThreatIntelObjects | Microsoft Defender Threat Intelligence STIX Objects | Microsoft Sentinel | TAXII — Polling | Threat Intelligence | STIX 2.1 structured threat intelligence — threat actors, attack patterns, campaign relationships |
| Heartbeat | Azure Monitor Agent Heartbeat | Microsoft Azure | AMA — DCR | Endpoint | One heartbeat per minute per AMA machine — primary mechanism for detecting agent and machine availability |
| SentinelHealth | Microsoft Sentinel Operational Health | Microsoft Sentinel | Sentinel Native | Compliance and Audit | Detects silent failures in analytics rules, connectors, and automation before clients notice |
| SentinelAudit | Microsoft Sentinel Configuration Audit | Microsoft Sentinel | Sentinel Native | Compliance and Audit | Records all configuration changes to the workspace — who changed what and when |
| SecurityEvent | Windows Security Events via AMA | Windows OS | AMA — DCR | Endpoint | Detects authentication attacks, lateral movement, privilege escalation, and persistence across Windows machines |
| CommonSecurityLog | [DeviceVendor] — [DeviceProduct] | Network Device | CEF — AMA — DCR | Firewall | Perimeter and network device logs — traffic patterns, connection events, threat signatures |
| Syslog | [HostName] | Windows OS | AMA — DCR | Endpoint | Linux system logs — varies by host role. Domain controllers, servers, and appliances each have different detection value |
| ASimNetworkSessionLogs | ASIM Normalized Network Sessions | Multiple Sources | ASIM Parser | Network | Normalized network sessions enabling vendor-agnostic detection rules across multiple source tables |
| ASimAuditEventLogs | ASIM Normalized Audit Events | Multiple Sources | ASIM Parser | Compliance and Audit | Normalized audit events enabling vendor-agnostic audit detection rules across multiple source tables |
| ASimWebSessionLogs | ASIM Normalized Web Sessions | Multiple Sources | ASIM Parser | Network | Normalized web session data enabling vendor-agnostic web detection rules across multiple source tables |
| CloudAppEvents | Microsoft Defender for Cloud Apps Events | Microsoft Defender | XDR Connector | SaaS Application | Detects SaaS application anomalies — mass downloads, impossible travel in apps, OAuth abuse, shadow IT |
| DeviceEvents | Microsoft Defender for Endpoint — Device Events | Microsoft Defender | XDR Connector | Endpoint | Endpoint security events from enrolled devices — process, network, file, and registry activity |
| DeviceNetworkEvents | Microsoft Defender for Endpoint — Network Events | Microsoft Defender | XDR Connector | Endpoint | Detects suspicious outbound connections, C2 communication, and lateral movement at the network layer |
| DeviceLogonEvents | Microsoft Defender for Endpoint — Logon Events | Microsoft Defender | XDR Connector | Endpoint | Detects suspicious logon patterns, credential abuse, and lateral movement on enrolled devices |
| DeviceProcessEvents | Microsoft Defender for Endpoint — Process Events | Microsoft Defender | XDR Connector | Endpoint | Detects malware execution, living-off-the-land attacks, and suspicious process chains |
| DeviceFileEvents | Microsoft Defender for Endpoint — File Events | Microsoft Defender | XDR Connector | Endpoint | Detects suspicious file creation, modification, and deletion — ransomware staging, malware drops |
| DeviceRegistryEvents | Microsoft Defender for Endpoint — Registry Events | Microsoft Defender | XDR Connector | Endpoint | Detects persistence mechanisms and configuration tampering via Windows registry modifications |
| DeviceImageLoadEvents | Microsoft Defender for Endpoint — Image Load Events | Microsoft Defender | XDR Connector | Endpoint | Detects DLL injection, side-loading, and suspicious module loading patterns |
| DeviceFileCertificateInfo | Microsoft Defender for Endpoint — Certificate Info | Microsoft Defender | XDR Connector | Endpoint | File signing certificate data — detects unsigned or suspiciously signed executables |
| DeviceNetworkInfo | Microsoft Defender for Endpoint — Network Info | Microsoft Defender | XDR Connector | Endpoint | Network adapter and configuration data from enrolled devices |
| DeviceInfo | Microsoft Defender for Endpoint — Device Info | Microsoft Defender | XDR Connector | Endpoint | Device inventory data — OS version, enrollment status, risk level |
| EmailEvents | Microsoft Defender for Office 365 — Email Events | Microsoft Defender | XDR Connector | Email | Detects phishing attempts, malware delivery, and suspicious email patterns |
| EmailAttachmentInfo | Microsoft Defender for Office 365 — Attachments | Microsoft Defender | XDR Connector | Email | Detects malicious attachments — file type abuse, macro-enabled documents, suspicious file names |
| EmailUrlInfo | Microsoft Defender for Office 365 — URLs | Microsoft Defender | XDR Connector | Email | Detects malicious URLs in email — phishing links, credential harvesting sites |
| EmailPostDeliveryEvents | Microsoft Defender for Office 365 — Post Delivery | Microsoft Defender | XDR Connector | Email | Detects threats identified after email delivery — detonation results, user clicks on malicious links |
| UrlClickEvents | Microsoft Defender for Office 365 — URL Clicks | Microsoft Defender | XDR Connector | Email | Detects user clicks on URLs in email and Teams — tracks when users interact with potentially malicious links |
| IdentityLogonEvents | Microsoft Defender for Identity — Logon Events | Microsoft Defender | XDR Connector | Identity | Detects suspicious authentication patterns against on-premises Active Directory |
| StorageBlobLogs | Azure Storage Account Blob Access Logs | Microsoft Azure | Diagnostic Setting | Cloud Infrastructure | Detects unauthorized data access, mass downloads, and exfiltration from Azure blob storage |
| StorageFileLogs | Azure Storage Account File Access Logs | Microsoft Azure | Diagnostic Setting | Cloud Infrastructure | Detects unauthorized file share access and suspicious file operations in Azure file storage |
| StorageQueueLogs | Azure Storage Account Queue Logs | Microsoft Azure | Diagnostic Setting | Cloud Infrastructure | Detects suspicious queue operations that may indicate application-layer attacks |
| StorageTableLogs | Azure Storage Account Table Logs | Microsoft Azure | Diagnostic Setting | Cloud Infrastructure | Detects unauthorized access and suspicious operations against Azure table storage |
| AzureDiagnostics | Azure Resource Diagnostic Logs | Microsoft Azure | Diagnostic Setting | Cloud Infrastructure | Aggregated diagnostic logs from multiple Azure resources — Key Vault, NSG, SQL, App Service |
| AzureMetrics | Azure Resource Metrics | Microsoft Azure | Diagnostic Setting | Cloud Infrastructure | Performance and health metrics from Azure resources — useful for detecting resource abuse |
| Perf | Performance Counters via AMA | Microsoft Azure | AMA — DCR | Endpoint | System performance data — CPU, memory, disk. Detects resource abuse like cryptomining |
| Anomalies | Microsoft Sentinel ML Anomalies | Microsoft Sentinel | Sentinel Native | Capabilities | ML-detected behavioral anomalies produced by Sentinel fusion and anomaly detection rules |
| UserPeerAnalytics | UEBA User Peer Analytics | Microsoft Sentinel | Sentinel Native | Capabilities | Peer group analysis data from UEBA — used to detect deviations from peer group behavior |
| Usage | Log Analytics Ingestion and Billing | Microsoft Sentinel | Sentinel Native | Compliance and Audit | Tracks data ingestion volume and billing per table — operational tool, not a security data source |
| AWSCloudTrail | AWS CloudTrail API Activity Logs | AWS | Microsoft Connector | Cloud Infrastructure | Detects unauthorized API calls, suspicious resource changes, and privilege escalation in AWS |

---

## Step 8 — Complete Tab 2 Configuration Reference

Create a second tab named Configuration. Fill in technical details for each log source. This tab is engineer-facing only — it documents the exact wiring behind each source.

**Finding the information for Tab 2:**
- Data Connector name → Sentinel or Defender portal → Data Connectors page
- Connector Status → Same page — Connected or Disconnected
- DCR Name → Azure portal → Monitor → Data Collection Rules — filter by workspace
- DCE Name → Azure portal → Monitor → Data Collection Endpoints
- Diagnostic Setting → Source resource in Azure portal → Diagnostic Settings
- Subscription → Azure portal → Subscriptions
- Workspace → Log Analytics workspace name
- License Required → Reference license-entitlement-review.md for this client

---

## Step 9 — Phase 3 Verification

For every Active source verify configuration is complete and correct. Data flowing does not mean everything is configured correctly.

### Verify Silent Detection Health

```kql
// All AB-series custom rules — health status
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

Cross-reference against SilentDet column. Any log source with no matching detection needs one created — mark SilentDet as Missing and add a Notes action item.

---

### Verify Microsoft Connector Log Category Completeness

```kql
// Template — replace TABLE_NAME with each expected table
TABLE_NAME
| where TimeGenerated > ago(24h)
| summarize Count = count(), LastSeen = max(TimeGenerated)
```

Minimum required Entra ID tables — verify each:
- SignInLogs
- AADNonInteractiveUserSignInLogs
- AADServicePrincipalSignInLogs
- AADManagedIdentitySignInLogs
- AuditLogs
- AADRiskyUsers — requires Entra ID P2
- AADUserRiskEvents — requires Entra ID P2

If any return zero — the diagnostic setting for that category is not configured in Entra ID → Monitoring → Diagnostic Settings.

---

### Verify AMA Sources — Heartbeat Health

```kql
// Heartbeat — All AMA Machines
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

```kql
// CEF Source Verification
// Replace VENDOR_NAME and PRODUCT_NAME with actual values
CommonSecurityLog
| where TimeGenerated > ago(24h)
| where DeviceVendor == "VENDOR_NAME"
| where DeviceProduct == "PRODUCT_NAME"
| summarize Count = count(), LastSeen = max(TimeGenerated)
```

If zero results — forwarder stopped receiving from this device. Check forwarder VM, AMA health, device Syslog destination, and forwarder port.

---

## Steps 10-11 — Document and Sort

**Step 10** — Every non-Active row must have Notes containing: what the issue is, next step, owner, target date. No exceptions.

**Step 11** — Sort by Table A to Z, then LogSource A to Z. Groups all rows for the same table together and keeps log sources alphabetical within each table.

---

## Step 12 — Upload Tab 1 as Watchlist

**Before uploading — three important notes:**

**Remove DaysSinceLastLog before upload.**
DaysSinceLastLog is calculated at audit time and goes stale immediately after upload. Delete this column from the CSV before uploading or exclude it during watchlist column mapping. The workbook calculates DaysSinceLastLog live at query time from LastSeen.

**Status is stored as an audit snapshot.**
The Status column reflects what the status was at the time of the audit. It goes stale as soon as the watchlist is uploaded. The workbook always recalculates Status live from workspace queries — it never reads Status from the watchlist for current health decisions.

**LastSeen becomes a string in the watchlist.**
Watchlists store all values as strings regardless of original type. Any workbook query that reads LastSeen from the watchlist must wrap it with todatetime() before doing any date math or comparisons:
```kql
_GetWatchlist('DataSourceInventory')
| extend LastSeen = todatetime(LastSeen)
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
```
This is required in every workbook query that uses LastSeen. Without it comparisons silently fail or return incorrect results.

**Upload steps:**
1. Navigate to Watchlists in Sentinel or Defender portal
2. Create new watchlist named `DataSourceInventory`
3. Export Tab 1 as CSV — remove DaysSinceLastLog column first
4. Upload that CSV
5. Set SearchKey to Table column
6. Validate:

```kql
_GetWatchlist('DataSourceInventory')
| summarize TotalRows = count()
```

---

## Known Limitations

**Lighthouse access scope** — access scoped to Sentinel workspace and Log Analytics. May not have access to VM extensions, resource tags, Azure Policy, or Resource Graph. Heartbeat queries cover AMA health without VM extension access.

**Tables that never received data** — Log Analytics only registers a table after at least one record is written. A connector enabled but never sending data will not appear in any query. Document as Missing.

**Shared table field reliability** — breakout queries depend on DeviceVendor, HostName, and Computer being correctly populated. Blank or inconsistent values need Notes documentation and source investigation.

**ASIM tables** — health depends entirely on source table health. If a source table goes inactive the ASIM table may still appear active while producing degraded output. Always check ASIM source tables during verification.

---

## Onboarding Note

This audit process is designed to be repeatable and eventually automated. The completed watchlist becomes the baseline for the living workbook, change detection analytics rules, and automated Teams notifications. In the future this process will be part of client onboarding. See PROGRAM-MAP.md for the full automation pipeline design.

---

## Audit Sign-Off

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

No log source may be signed off with a non-Active status without a documented action item and owner in Notes.

---

*This runbook is maintained in sentinel-mssp-playbook under 02-data-sources. Update when the audit process changes, new source types require guidance, or new shared tables are identified.*