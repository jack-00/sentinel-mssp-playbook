# Data Source Audit Runbook

> **Purpose:** This runbook guides an engineer through a complete data source audit for a managed client environment in Microsoft Sentinel. The output is a structured, organized spreadsheet that becomes the foundation for the living workbook, the data source watchlist, and all ongoing health monitoring.
>
> **Who uses this:** Engineers conducting a first-time audit or a periodic re-audit of a client environment.
>
> **What you produce:** A completed, sorted, Excel-formatted audit spreadsheet ready for watchlist upload.
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

**The log source is the correct audit unit.** One row per originating source. One status per source. This is the only level of granularity that catches individual source failures before they cause missed detections.

---

## The Audit Workflow

Work through all ten steps in order. Do not skip ahead.

- [ ] **Step 1** — Run the main table inventory query in the Logs blade
- [ ] **Step 2** — Run the shared table breakout queries for CommonSecurityLog, Syslog, and SecurityEvent
- [ ] **Step 3** — Export results to CSV
- [ ] **Step 4** — Open in Excel and format as a structured table (Ctrl+T)
- [ ] **Step 5** — Save immediately as a dated Excel workbook and upload to SharePoint
- [ ] **Step 6** — Fill in all manual fields using the column definitions and pre-populated reference below
- [ ] **Step 7** — Run Phase 3 verification for every Active source
- [ ] **Step 8** — Document all Inactive, Review, and Flag sources in the Notes column with a next step and owner
- [ ] **Step 9** — Sort the completed spreadsheet alphabetically by Table then by Log Source within each table
- [ ] **Step 10** — Save and upload the sorted, completed spreadsheet to Sentinel as a watchlist

---

## Step 1 — Main Table Inventory Query

Run this query first in the Microsoft Sentinel Logs blade. It returns every table in the workspace that has ever received data — including tables that went silent months ago. This is your primary row list.

**Important — first run instruction:** On the very first audit for a client, remove the time filter entirely or set it to the maximum workspace retention period. You want to capture every table that has ever existed, not just recent ones. Tables that went silent six months ago are exactly what you are looking for. On subsequent audits the 30-day window is sufficient.

```kql
// ============================================================
// MAIN TABLE INVENTORY — Run first. One row per table.
// First run: remove the 'ago' filter to capture all history
// Subsequent runs: keep as-is for 30-day window
// ============================================================
search *
| where TimeGenerated > ago(30d) // Remove this line on first run
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count(),
    AvgDailyRecords = count() / 30
    by $table
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1,  "🟢 Active",
    DaysSinceLastLog <= 7,  "🟡 Review",
    DaysSinceLastLog > 7,   "🔴 Inactive",
    "⚫ No Data"
)
| extend LastSeen_UTC = format_datetime(LastSeen, 'yyyy-MM-dd HH:mm')
// ── Manual fields ── fill these in after export ──────────────
| extend LogSource =     "[Manual] — Specific originator e.g. Palo Alto PA-3200, DC01, Entra ID Sign-in"
| extend Source =        "[Manual] — Plain English name e.g. Microsoft Entra ID Sign-in Logs"
| extend Vendor =        "[Manual] — e.g. Microsoft, Palo Alto Networks, Fortinet, CrowdStrike"
| extend Category =      "[Manual] — Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Other"
| extend Purpose =       "[Manual] — One sentence: what attack or risk does this data help detect?"
| extend Collection =    "[Manual] — Microsoft Connector / AMA Agent / DCR / CEF Syslog / REST API / Logic App / Manual Custom"
| extend SilentDet =     "[Manual] — AB##### - Detection name that monitors this source for silence"
| extend DetStatus =     "[Manual] — Enabled and Healthy / Disabled / Unhealthy / Missing — verify in SentinelHealth"
| extend Detections =    "[Manual] — AB##### - Detection name, AB##### - Detection name"
| extend Watchlists =    "[Manual] — Watchlist names that reference this source, or None"
| extend Notes =         "[Manual] — Issues, action items, exceptions, client context"
// ─────────────────────────────────────────────────────────────
| project
    Table = $table,
    LogSource,
    Source,
    Vendor,
    Category,
    Status,
    LastSeen_UTC,
    DaysSinceLastLog,
    TotalRecords,
    AvgDailyRecords,
    Purpose,
    Collection,
    SilentDet,
    DetStatus,
    Detections,
    Watchlists,
    Notes
| order by LastSeen_UTC desc
```

---

## Step 2 — Shared Table Breakout Queries

Some tables receive data from multiple different log sources simultaneously. The main query above returns one row per table — which is not granular enough for shared tables. Run each query below that is relevant to the client environment. Each result adds additional rows to your spreadsheet, one per log source within that table.

**When to use these:** If the main query shows CommonSecurityLog, Syslog, or SecurityEvent — always run the corresponding breakout query. These three tables almost always have multiple sources.

---

### Breakout A — CommonSecurityLog (Firewalls, Network Devices, CEF Sources)

```kql
// ============================================================
// CommonSecurityLog BREAKOUT — One row per vendor/product
// Run if CommonSecurityLog appeared in the main query
// ============================================================
CommonSecurityLog
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count(),
    AvgDailyRecords = count() / 30
    by DeviceVendor, DeviceProduct
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1,  "🟢 Active",
    DaysSinceLastLog <= 7,  "🟡 Review",
    DaysSinceLastLog > 7,   "🔴 Inactive",
    "⚫ No Data"
)
| extend LastSeen_UTC = format_datetime(LastSeen, 'yyyy-MM-dd HH:mm')
| extend Table = "CommonSecurityLog"
| extend LogSource = strcat(DeviceVendor, " — ", DeviceProduct)
| extend Source =        "[Manual] — Plain English name e.g. Palo Alto Networks Firewall"
| extend Vendor =        "[Manual] — Use DeviceVendor value above as guide"
| extend Category =      "[Manual] — Firewall / Network / Other"
| extend Purpose =       "[Manual] — What does this specific device log help detect?"
| extend Collection =    "CEF / Syslog"
| extend SilentDet =     "[Manual] — AB##### - Silent detection for this specific source"
| extend DetStatus =     "[Manual] — Enabled and Healthy / Disabled / Unhealthy / Missing"
| extend Detections =    "[Manual] — AB-series detections relying on this source"
| extend Watchlists =    "[Manual] — Watchlists referencing this source or None"
| extend Notes =         "[Manual] — Notes"
| project
    Table,
    LogSource,
    Source,
    Vendor,
    Category,
    Status,
    LastSeen_UTC,
    DaysSinceLastLog,
    TotalRecords,
    AvgDailyRecords,
    Purpose,
    Collection,
    SilentDet,
    DetStatus,
    Detections,
    Watchlists,
    Notes
| order by DeviceVendor asc
```

---

### Breakout B — Syslog (Linux Agents, AMA Sources, Network Devices)

```kql
// ============================================================
// Syslog BREAKOUT — One row per host and process
// Run if Syslog appeared in the main query
// ============================================================
Syslog
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count(),
    AvgDailyRecords = count() / 30
    by HostName, ProcessName
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1,  "🟢 Active",
    DaysSinceLastLog <= 7,  "🟡 Review",
    DaysSinceLastLog > 7,   "🔴 Inactive",
    "⚫ No Data"
)
| extend LastSeen_UTC = format_datetime(LastSeen, 'yyyy-MM-dd HH:mm')
| extend Table = "Syslog"
| extend LogSource = strcat(HostName, " — ", ProcessName)
| extend Source =        "[Manual] — Plain English name e.g. Linux Domain Controller System Logs"
| extend Vendor =        "[Manual] — e.g. Microsoft, Linux, or specific appliance vendor"
| extend Category =      "[Manual] — Endpoint / Network / Identity / Other"
| extend Purpose =       "[Manual] — What does this host or process log help detect?"
| extend Collection =    "AMA Agent"
| extend SilentDet =     "[Manual] — AB##### - Silent detection for this specific host"
| extend DetStatus =     "[Manual] — Enabled and Healthy / Disabled / Unhealthy / Missing"
| extend Detections =    "[Manual] — AB-series detections relying on this source"
| extend Watchlists =    "[Manual] — Watchlists referencing this host or None"
| extend Notes =         "[Manual] — Notes"
| project
    Table,
    LogSource,
    Source,
    Vendor,
    Category,
    Status,
    LastSeen_UTC,
    DaysSinceLastLog,
    TotalRecords,
    AvgDailyRecords,
    Purpose,
    Collection,
    SilentDet,
    DetStatus,
    Detections,
    Watchlists,
    Notes
| order by HostName asc
```

---

### Breakout C — SecurityEvent (Windows Machines via AMA)

```kql
// ============================================================
// SecurityEvent BREAKOUT — One row per computer
// Run if SecurityEvent appeared in the main query
// ============================================================
SecurityEvent
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count(),
    AvgDailyRecords = count() / 30
    by Computer
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSinceLastLog <= 1,  "🟢 Active",
    DaysSinceLastLog <= 7,  "🟡 Review",
    DaysSinceLastLog > 7,   "🔴 Inactive",
    "⚫ No Data"
)
| extend LastSeen_UTC = format_datetime(LastSeen, 'yyyy-MM-dd HH:mm')
| extend Table = "SecurityEvent"
| extend LogSource = Computer
| extend Source =        "[Manual] — e.g. Windows Domain Controller Security Events"
| extend Vendor =        "Microsoft"
| extend Category =      "[Manual] — Identity / Endpoint — check if DC, server, or workstation"
| extend Purpose =       "[Manual] — What does this machine's security events help detect?"
| extend Collection =    "AMA Agent"
| extend SilentDet =     "[Manual] — AB##### - Silent detection or heartbeat detection for this machine"
| extend DetStatus =     "[Manual] — Enabled and Healthy / Disabled / Unhealthy / Missing"
| extend Detections =    "[Manual] — AB-series detections relying on this machine"
| extend Watchlists =    "[Manual] — Domain Controllers / Critical Assets / or None"
| extend Notes =         "[Manual] — Notes — flag Domain Controllers and servers explicitly"
| project
    Table,
    LogSource,
    Source,
    Vendor,
    Category,
    Status,
    LastSeen_UTC,
    DaysSinceLastLog,
    TotalRecords,
    AvgDailyRecords,
    Purpose,
    Collection,
    SilentDet,
    DetStatus,
    Detections,
    Watchlists,
    Notes
| order by Computer asc
```

---

## Steps 3-5 — Export, Format, and Save

**Step 3 — Export to CSV**
In the Logs blade, run each query and use the Export button to download results as CSV. You will have up to four CSV files — one from the main query and one each for any shared table breakouts you ran.

**Step 4 — Open in Excel and format as a structured table**
Open the main CSV in Excel. Copy and paste rows from any breakout CSVs below the main data — they share the same column structure. Then:
- Click any cell in the data
- Press **Ctrl+T**
- Confirm the "My table has headers" checkbox is checked
- Click OK

This formats the data as an Excel Table — not a pivot table. An Excel Table enables column filtering, sorting, and structured references. This is what you want. The column headers will get dropdown arrows. You can now filter by Status, Category, Vendor, or any other column instantly.

**Step 5 — Save as a dated Excel workbook and upload to SharePoint**
Immediately save the file in Excel Workbook format (.xlsx) — not CSV. Use this naming convention:

```
[ClientName]-DataSourceAudit-[YYYY-MM-DD].xlsx
```

Example: `AcmeCorp-DataSourceAudit-2026-04-08.xlsx`

Upload this file to the client's SharePoint folder before filling in any manual fields. This is your baseline snapshot — an unmodified record of exactly what the workspace looked like on this date. Even before the manual fields are filled in it has value as a timestamped record of what tables existed and their health status.

---

## Step 6 — Fill In Manual Fields

Every column with `[Manual]` in the query output needs to be filled in. Work through each row systematically. Do not leave `[Manual]` placeholder text in any cell — replace it with the actual value or N/A.

Use the pre-populated reference table below for Tier 1 and Tier 2 sources — most of the common Microsoft sources are already documented so you are validating rather than researching. For third-party and environment-specific sources you will need to research each one.

---

### Pre-Populated Reference — Common Sources

Use this table to quickly fill in manual fields for the most common sources. Copy the relevant values directly into your spreadsheet. Validate that they match what you see in the environment — do not assume.

| Table | Log Source | Source | Vendor | Category | Collection | Purpose |
|---|---|---|---|---|---|---|
| SignInLogs | Microsoft Entra ID | Microsoft Entra ID Interactive Sign-in Logs | Microsoft | Identity | Microsoft Connector | Detects compromised accounts, password spray, impossible travel, MFA fatigue, and unauthorized access attempts |
| AADNonInteractiveUserSignInLogs | Microsoft Entra ID | Microsoft Entra ID Non-Interactive Sign-in Logs | Microsoft | Identity | Microsoft Connector | Detects service-to-service auth abuse, token theft, and background authentication anomalies often missed in interactive logs |
| AADServicePrincipalSignInLogs | Microsoft Entra ID | Microsoft Entra ID Service Principal Sign-in Logs | Microsoft | Identity | Microsoft Connector | Detects abuse of service principals and app registrations — a common attacker pivot point that is frequently overlooked |
| AADManagedIdentitySignInLogs | Microsoft Entra ID | Microsoft Entra ID Managed Identity Sign-in Logs | Microsoft | Identity | Microsoft Connector | Detects abuse of managed identities used by Azure resources to authenticate to other services |
| AuditLogs | Microsoft Entra ID | Microsoft Entra ID Audit Logs | Microsoft | Identity | Microsoft Connector | Detects directory changes — role assignments, group membership changes, app registrations, and admin operations |
| AADRiskyUsers | Microsoft Entra ID | Microsoft Entra ID Identity Protection — Risky Users | Microsoft | Identity | Microsoft Connector | Surfaces accounts flagged by Microsoft ML as compromised or at risk — leaked credentials, impossible travel, malware-linked IPs |
| AADUserRiskEvents | Microsoft Entra ID | Microsoft Entra ID Identity Protection — Risk Events | Microsoft | Identity | Microsoft Connector | Detailed risk event data per sign-in — feeds UEBA and risk-based conditional access detections |
| OfficeActivity | Microsoft 365 | Microsoft 365 Unified Audit Log | Microsoft | Compliance and Audit | Microsoft Connector | Detects data exfiltration via SharePoint/OneDrive, suspicious mailbox access, email forwarding rules, and OAuth consent abuse |
| AzureActivity | Azure | Azure Activity — Management Plane | Microsoft | Cloud Infrastructure | Microsoft Connector | Detects unauthorized Azure resource changes — role assignments, VM creation, firewall rule modifications, Key Vault access policy changes |
| SecurityAlert | Multiple | Microsoft Security Alerts — All Products | Microsoft | Cloud Infrastructure | Microsoft Connector | Consolidated alert feed from all connected Microsoft security products — source of record for XDR-generated alerts in Sentinel |
| SecurityIncident | Sentinel / XDR | Microsoft Sentinel and XDR Incidents | Microsoft | Cloud Infrastructure | Microsoft Connector | All incidents in the workspace including XDR-correlated multi-stage incidents — primary pipeline source for ServiceNow tickets |
| BehaviorAnalytics | UEBA | UEBA — Behavioral Analytics Output | Microsoft | Capabilities | Microsoft Connector | ML-generated behavioral anomaly scores per user and entity — requires 14-21 day baseline period before reliable output |
| IdentityInfo | UEBA | UEBA — Identity Information Table | Microsoft | Capabilities | Microsoft Connector | Enriched user identity data synchronized from Entra ID and on-prem AD — used by detections for user context |
| ThreatIntelIndicators | Threat Intelligence | Microsoft Defender Threat Intelligence | Microsoft | Threat Intelligence | Microsoft Connector | Current IOC feed — malicious IPs, domains, URLs, file hashes matched against all ingested log data |
| ThreatIntelObjects | Threat Intelligence | Microsoft Threat Intelligence STIX Objects | Microsoft | Threat Intelligence | Microsoft Connector | STIX 2.1 structured threat intelligence — threat actors, attack patterns, campaign relationships |
| Heartbeat | AMA Agent | Azure Monitor Agent Heartbeat | Microsoft | Endpoint | AMA Agent | One heartbeat per minute per AMA-connected machine — primary mechanism for detecting agent and machine availability |
| SentinelHealth | Sentinel | Microsoft Sentinel Health and Audit | Microsoft | Compliance and Audit | Microsoft Connector | Operational health of analytics rules, data connectors, and automation — critical for detecting silent failures |
| SentinelAudit | Sentinel | Microsoft Sentinel Configuration Audit | Microsoft | Compliance and Audit | Microsoft Connector | Records all configuration changes to the Sentinel workspace — who changed what and when |
| SecurityEvent | Windows Machine | Windows Security Events — [Computer Name] | Microsoft | Endpoint / Identity | AMA Agent | Windows security event log — authentication events, process creation, privilege use. Critical for on-prem identity and lateral movement detection |
| CommonSecurityLog | CEF Device | [DeviceVendor] — [DeviceProduct] | [Vendor] | Firewall / Network | CEF / Syslog | Perimeter and network device logs — traffic patterns, connection events, threat signatures. Required for network-based detection |
| Syslog | Linux Host | [HostName] — [ProcessName] | [Vendor] | Endpoint / Network | AMA Agent | Linux system and application logs — varies by host purpose. Domain controllers, servers, and network appliances each have different detection value |

---

## Column Definitions

Every column is defined below. Short header is what appears in the spreadsheet. Fill in every field. Write N/A if not applicable — do not leave cells blank.

---

**Table** — The exact Log Analytics table name as it appears in query results. Do not modify. This is the key that links every row back to the workspace.

---

**Log Source** — The specific originator writing data to this table. For tables with a single dedicated source this is the same as the Source column. For shared tables (CommonSecurityLog, Syslog, SecurityEvent) this identifies the specific device, host, or process. Format: plain text describing the specific source. Examples: `Palo Alto PA-3200 Series`, `DC01 — Windows Security`, `Entra ID Sign-in`.

---

**Source** — Plain English name of what is sending this data. Write it so a client can understand it without technical knowledge. Capitalize properly. Do not use abbreviations. Example: `Microsoft Entra ID Sign-in Logs`, `Palo Alto Networks Next-Generation Firewall`.

---

**Vendor** — The company or product that produces this data source. Use the official vendor name. Examples: `Microsoft`, `Palo Alto Networks`, `Fortinet`, `CrowdStrike`, `Okta`, `Amazon Web Services`. For Microsoft-native sources the vendor is always Microsoft.

---

**Category** — Select one from the approved list:

| Value | Use For |
|---|---|
| Identity | User authentication, directory services, account management |
| Endpoint | Device telemetry, EDR, antivirus, host-based events |
| Email | Mail flow, phishing detection, attachment scanning |
| Network | Traffic logs, DNS, DHCP, proxy |
| Firewall | Perimeter firewall and NGFW logs |
| Cloud Infrastructure | Azure resource logs, management plane, cloud workload protection |
| SaaS Application | Third-party cloud apps — Salesforce, ServiceNow, Okta |
| Threat Intelligence | IOC feeds, indicator matching |
| Compliance and Audit | M365 audit logs, Sentinel health, regulatory compliance events |
| Vulnerability Management | Scanner results, CVE data, asset risk scoring |
| Authentication | MFA, SSO, PAM, privileged access |
| Capabilities | UEBA output tables, Fusion ML output — sources that are capability outputs not raw inputs |
| Other | Anything not fitting the above — add explanation in Notes |

---

**Status** — The current health status of this log source. Assign one of seven values:

| Status | Emoji | Assigned By | Meaning |
|---|---|---|---|
| Active | 🟢 | Query | Data flowing within expected timeframe. Volume looks normal. No action needed. |
| Inactive | 🔴 | Query | Had data, now silent. Something broke. Investigate immediately. Document cause in Notes. |
| Review | 🟡 | Query | Data flowing but volume anomaly, inconsistent timestamps, or partial configuration suspected. Needs closer look before marking Active. |
| No Data | ⚫ | Query | In reference list, never received data. Confirm whether not applicable or never configured. Document in Notes. |
| Missing | 🔵 | Manual | Expected based on connector configuration but not found anywhere in workspace. Connector may be enabled but never successfully sent data. |
| Flag | 🟠 | Query | New or unexpected source appeared since last audit. Investigate and classify. Do not leave as Flag — resolve to another status. |
| Decom | 🗑️ | Manual only | Marked for retirement. Source is no longer needed or the device has been decommissioned. Do not delete from watchlist — mark Decom with date and reason in Notes. |

**Important:** Assign status based on query results plus judgment — not connector UI status. A connector showing green is not evidence of a healthy log source.

**Status guidance by source type:**

- Identity sources (SignInLogs, AuditLogs) — should have data every few minutes in any active environment. More than 4 hours without data → Review. More than 24 hours → Inactive.
- Endpoint sources — depend on device activity. Low volume on weekends may be normal. No data for 48 hours in a business environment → Inactive.
- Firewall and network sources — near-continuous in any production environment. More than 6 hours → Review. More than 24 hours → Inactive.
- Threat intelligence tables — may update once daily or less. Check vendor cadence before marking Inactive.
- UEBA output tables (BehaviorAnalytics, IdentityInfo) — should populate continuously once enabled and past the 14-21 day baseline period.

---

**Last Seen** — Timestamp of the most recent record in this table from this log source. Comes directly from query results. Format: `YYYY-MM-DD HH:MM UTC`. If no data ever: `Never`.

---

**Daily Vol** — Average daily ingestion volume over the last 30 days from query results. Format: use MB for smaller sources, GB for larger ones. Round to one decimal. Examples: `2.4 GB`, `450 MB`, `< 1 MB`. This is your baseline — significant drops in future audits are an early warning sign even before a source goes Inactive.

---

**Purpose** — One to two sentences explaining why this log source matters from a security perspective. Write it so a client can understand it. Answer the question: what would an attacker be able to do undetected if this source disappeared?

Good example: `Firewall logs are the primary source for detecting outbound connections to known malicious infrastructure, port scanning, and unauthorized lateral movement across network segments. Without this data, network-based attack patterns are completely invisible.`

Poor example: `This table contains firewall logs.`

---

**Collection** — How data gets from the source into the workspace. Select one:

| Value | Use For |
|---|---|
| Microsoft Connector | Native Microsoft service-to-service integration — no agent required |
| AMA Agent | Azure Monitor Agent installed on a host sending logs via DCR |
| CEF / Syslog | Third-party device sending logs over Syslog or Common Event Format via a forwarder |
| REST API | Source pushing data via API — often through a Logic App or Function App |
| Logic App | Custom Logic App polling or receiving data from a third-party source |
| Manual / Custom | Custom ingestion pipeline not covered by the above |

---

**Silent Det** — The AB-series detection ID and name of the silent log source detection that monitors this specific source for silence. Every log source must have one. Format: `AB##### - Detection Name`. If missing: `[Missing] — needs to be created`. This is a mandatory field — do not leave blank.

---

**Det Status** — The current health of the silent detection. Verify against SentinelHealth before filling in. Values: `Enabled and Healthy`, `Enabled but Unhealthy`, `Disabled`, `Missing`. Use the verification query in Phase 3 below.

---

**Detections** — The AB-series detection IDs and names from the master detection catalog that query data from this log source. If a detection goes blind when this source stops sending data, list it here. Format: one detection per line within the cell. If none: `None — Microsoft Content Hub only` or `None`.

---

**Watchlists** — Watchlists that reference data from this log source or are used in detections that depend on it. Example: `Domain Controllers, Critical Assets`. If none: `None`.

---

**Notes** — Free text. Date-stamp action items. Use for: root cause of Inactive status, what specifically needs attention for Review, whether No Data is not applicable or not configured, outstanding action items with owner and target date.

---

## Phase 3 — Verification

For every source marked Active — verify the configuration is complete and correct. Data flowing is not the same as everything being configured correctly. A source can be partially configured — streaming some log categories but missing others — and appear healthy while having significant blind spots.

---

### Verify Silent Detection Is Enabled and Healthy

Run this for every log source. Cross-reference against your Det Status column.

```kql
// Silent Detection Health — All Custom Rules
// Filters to AB-series rules — your custom detections
SentinelHealth
| where SentinelResourceType == "Analytics Rule"
| where SentinelResourceName matches regex @"^AB\d+"
| summarize LastStatus = arg_max(TimeGenerated, Status, Description)
    by SentinelResourceName
| project
    DetectionName = SentinelResourceName,
    HealthStatus = LastStatus,
    LastChecked = TimeGenerated,
    Description
| order by HealthStatus asc
```

Any rule not returning `Success` needs to be investigated. Any AB-series rule not appearing in this list at all may not exist — cross-reference against your SilentDet column to find gaps.

---

### Verify Microsoft Connector Sources — Log Category Completeness

For Microsoft native sources, enabling the connector is not enough. Each log category must be explicitly enabled in diagnostic settings. Run this to confirm each expected table is receiving data:

```kql
// Template — replace TABLE_NAME with each expected table
// Run once per expected log category for each Microsoft source
TABLE_NAME
| where TimeGenerated > ago(24h)
| summarize Count = count(), LastSeen = max(TimeGenerated)
```

**Entra ID minimum required tables — verify each:**
- `SignInLogs`
- `AADNonInteractiveUserSignInLogs`
- `AADServicePrincipalSignInLogs`
- `AADManagedIdentitySignInLogs`
- `AuditLogs`
- `AADRiskyUsers`
- `AADUserRiskEvents`

If any of these return zero results, the diagnostic setting for that log category is not configured in Entra ID. Navigate to Azure portal → Entra ID → Monitoring → Diagnostic Settings and confirm each category is routed to the correct workspace.

---

### Verify AMA Agent Sources — Critical Event IDs

For Windows Security Event sources, the DCR must be configured to collect the right event IDs. A DCR that collects only System logs while missing Security logs appears healthy but is blind to authentication events.

```kql
// Windows Security Events — Critical Event ID Verification
// All of these should be present in any active Windows environment
SecurityEvent
| where TimeGenerated > ago(24h)
| where Computer == "COMPUTER_NAME" // Replace with specific machine or remove for all
| where EventID in (4624, 4625, 4648, 4672, 4688, 4720, 4728, 4732, 4756)
| summarize Count = count() by EventID
| order by EventID asc
```

| Event ID | What It Captures |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4648 | Logon using explicit credentials |
| 4672 | Special privileges assigned to new logon |
| 4688 | New process created |
| 4720 | User account created |
| 4728 | Member added to global security group |
| 4732 | Member added to local security group |
| 4756 | Member added to universal security group |

Any missing event IDs indicate the DCR is not collecting them. Review the DCR configuration in Azure Monitor → Data Collection Rules.

---

### Verify CEF / Syslog Sources — Vendor Specific

For firewall and network device sources writing to CommonSecurityLog:

```kql
// CEF Source Verification — Replace vendor values
// Run once per vendor/product combination found in breakout query
CommonSecurityLog
| where TimeGenerated > ago(24h)
| where DeviceVendor == "VENDOR_NAME"    // e.g. "Palo Alto Networks"
| where DeviceProduct == "PRODUCT_NAME"  // e.g. "PAN-OS"
| summarize Count = count(), LastSeen = max(TimeGenerated)
```

If this returns zero results while the breakout query showed historical data — the Syslog forwarder stopped receiving from this device. Investigate:
1. Confirm the Syslog forwarder VM is running
2. Confirm the AMA on the forwarder is healthy
3. Confirm the firewall is still configured to send Syslog to the forwarder IP
4. Confirm the forwarder is listening on the correct port

---

### Verify Heartbeat Coverage

Confirm every AMA-connected machine is sending heartbeats. Any machine in your watchlists (Domain Controllers, Critical Assets) that is not sending heartbeats should be flagged immediately.

```kql
// Heartbeat Coverage — All AMA Machines
// Machines not seen in last 1 hour are flagged
Heartbeat
| where TimeGenerated > ago(24h)
| summarize LastHeartbeat = max(TimeGenerated) by Computer, OSType, Category
| extend HoursSinceLastHeartbeat = datetime_diff('hour', now(), LastHeartbeat)
| extend HeartbeatStatus = iff(HoursSinceLastHeartbeat <= 1, "🟢 Healthy", "🔴 Missing")
| project Computer, OSType, Category, LastHeartbeat, HoursSinceLastHeartbeat, HeartbeatStatus
| order by HoursSinceLastHeartbeat desc
```

---

## Steps 8 and 9 — Document and Sort

**Step 8 — Document all non-Active sources**
Before moving to sort, every row marked Inactive, Review, Flag, or Missing must have a Notes entry that includes:
- What the issue appears to be
- What the next step is
- Who owns the follow-up
- Target resolution date

A row cannot be signed off with a non-Active status and an empty Notes field.

**Step 9 — Sort alphabetically**
Sort the completed spreadsheet in Excel:
1. Click any cell in the Table
2. Go to Data → Sort
3. Sort by **Table** (A to Z) first
4. Then add a second sort level by **Log Source** (A to Z)

This ensures all rows for CommonSecurityLog are grouped together, all rows for Syslog are grouped together, and within each table all log sources are in alphabetical order. This is the sort order the watchlist will maintain when uploaded to Sentinel — making it scannable and predictable for anyone querying it.

---

## Step 10 — Upload as Watchlist

Once the spreadsheet is complete and sorted it becomes the data source watchlist in Sentinel. This watchlist is the foundation for the living workbook — the join source that provides all the curated context that cannot be queried from the workspace alone.

**Upload steps:**
1. In the Sentinel or Defender portal navigate to Watchlists
2. Create a new watchlist — name it `DataSourceInventory`
3. Upload the completed and sorted .xlsx file
4. Set the SearchKey to the `Table` column
5. Confirm the watchlist loads correctly — row count should match your spreadsheet

**After upload — validate the watchlist:**
```kql
// Confirm watchlist loaded correctly
_GetWatchlist('DataSourceInventory')
| summarize TotalRows = count()
| extend ExpectedRows = 0 // Replace with your spreadsheet row count
```

---

## Known Limitations

The following limitations affect what is visible during the audit. These are documented here so engineers understand what they cannot see and why. These limitations are tracked as future work items in `11-access-and-permissions/`.

**Lighthouse access scope**
Our access is delegated via Azure Lighthouse scoped to the Sentinel workspace and Log Analytics. Depending on what roles were delegated we may not have access to:
- Azure VM extensions — cannot verify AMA agent installation status directly
- Azure Resource tags — cannot query resource classification tags
- Azure Policy — cannot verify tagging policies
- Azure Resource Graph — cannot pull full resource inventory

**Workaround:** Heartbeat queries in Sentinel confirm AMA machine health without needing VM extension access. For resource inventory gaps document them in Notes and flag for the client IT team.

**Tables that never received data**
Log Analytics only registers a table in the workspace schema after at least one record has been written to it. A connector that was configured but never successfully sent a single record will not appear in any query. These sources will show status Missing when compared against the reference watchlist. They are invisible to the main query alone.

**Shared table granularity**
The breakout queries for CommonSecurityLog, Syslog, and SecurityEvent provide log source granularity but depend on the fields DeviceVendor, HostName, and Computer being populated correctly. Poorly configured sources may have blank or inconsistent values in these fields — document these in Notes and investigate the source configuration.

---

## Onboarding Note

This audit process is designed to be repeatable and eventually automated. The completed watchlist from this audit becomes the baseline that drives the living workbook, the change detection analytics rules, and the automated Teams notifications.

In the future this process will be integrated into the client onboarding workflow — new clients will have this audit completed during onboarding so the watchlist exists from day one. A living onboarding workbook to track data source configuration progress and integrate the tagging strategy is planned as a future capability. See `PROGRAM-MAP.md` for the full future state design.

---

## Audit Sign-Off

Add this block to the bottom of the completed spreadsheet:

```
Audit Completed By:      [Engineer Name]
Date Completed:          [YYYY-MM-DD]
Client:                  [Client Name]
Total Log Sources:       [Number]
Active:                  [Number]
Review:                  [Number]
Inactive:                [Number]
No Data:                 [Number]
Missing:                 [Number]
Flag:                    [Number]
Decom:                   [Number]
Silent Det Coverage:     [X of Y log sources have a silent detection]
Next Audit Due:          [YYYY-MM-DD]
Notes:                   [Overall observations]
Watchlist Uploaded:      Yes / No
SharePoint Backup:       Yes / No — [link]
```

No log source may be left in Review, Inactive, Flag, or Missing status without a documented action item and owner in the Notes column. Sign-off is not complete until every non-Active row is accounted for.

---

*This runbook is maintained in the sentinel-mssp-playbook repository under 02-data-sources. Update it when the audit process changes, new source types require specific guidance, or new shared tables are identified. Version and date any significant changes.*