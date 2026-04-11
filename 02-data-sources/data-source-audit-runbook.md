# Data Source Audit Runbook

> **Purpose:** This runbook guides an engineer through a complete data source audit for a managed client Microsoft Sentinel environment. The primary output is the `[mssname]-tables` watchlist — the table pipe registry that forms the backbone of the entire system. After completing this runbook proceed to `06-watchlist-management/sources-watchlist-build-guide.md` to build the `[mssname]-sources` watchlist.
>
> **Who uses this:** Engineers conducting a first-time audit or a periodic re-audit of a client environment.
>
> **What you produce:** A completed `[mssname]-tables` watchlist uploaded to Sentinel.
>
> **Prerequisites:** Complete `01-baseline/license-entitlement-review.md` for this client before starting. Knowing what the client is licensed for tells you what data sources should exist and what capabilities should be enabled.
>
> **Last Updated:** April 2026

---

## Understanding the Workspace Before You Audit

A Microsoft Sentinel workspace is not a flat list of data sources. It contains nine distinct types of components that each play a different role. Understanding these categories is what separates a surface-level audit from a complete one.

| Category | What It Is | Examples |
|---|---|---|
| **Log Sources** | Specific originators writing data to a table. One table can have many log sources. | Palo Alto PA-3200 → CommonSecurityLog, DC01 → Syslog |
| **Tables** | Log Analytics destinations where data lands. The ground truth layer. | SignInLogs, CommonSecurityLog, BehaviorAnalytics |
| **Watchlists** | Persistent reference data queryable in KQL at any time. | [mssname]-tables, [mssname]-sources, [mssname]-detections |
| **Capabilities** | Features depending on multiple sources — UEBA, Threat Feed, Fusion ML | If any dependency breaks the capability degrades silently |
| **Detections** | Analytics rules that consume table data and produce alerts | OC-series rules, Content Hub rules |
| **Silent Detections** | Detections that verify each log source is actively sending data | Every tracked source must have one |
| **Automation** | Logic Apps and playbooks that respond to incidents | Enrichment playbook, ServiceNow ticket creation |
| **Integrations** | API keys and credentials that keep services connected | Threat feed API key, Zoom API key |
| **Workbooks** | Interactive live dashboards | Audit Deck Workbook, MSSP Internal Workbook |

---

## Why We Audit by Log Source — Not by Connector or Table

A connector can show green while silently sending nothing. Tables are closer to the truth but still not granular enough. A table like `CommonSecurityLog` can have many different sources writing to it simultaneously. If a Palo Alto firewall stops sending logs but a Fortinet device keeps writing — `CommonSecurityLog` still appears Active. The Palo Alto source is blind.

**The log source is the correct audit unit.** One row per originating source category. This is the only level of granularity that catches individual source failures before they cause missed detections.

**The two-watchlist approach:**
This runbook produces `[mssname]-tables` — one row per table, the pipe registry. The sources watchlist build guide produces `[mssname]-sources` — one row per log source within each table, where all monitoring detail lives. Both are needed. Do this one first.

---

## The Two Watchlists and Their Relationship

**`[mssname]-tables`** — what this runbook produces:
- One row per table
- Five fields — lean, purposeful, stable
- Catches new unknown tables appearing in the workspace
- Vetted = No flags tables that need investigation

**`[mssname]-sources`** — what the sources build guide produces:
- One row per log source within each table
- Fifteen fields — all monitoring, transport, SLS, functions
- Drives the workbook data sources tab and silent detections
- Every table in [mssname]-tables gets at least one row here

---

## The Tables Watchlist — Field Set

**`[mssname]-tables` fields — in this exact order:**

| Field | What Goes Here |
|---|---|
| Table | Exact Log Analytics table name — never modify |
| Category | Type of security data — from approved list |
| Details | Broad context about this table — shared table, ASIM, single source, what kind of data lands here |
| Vetted | Yes / No — has this table been investigated and classified |
| Notes | Issues, action items, context, observations |

**What is deliberately not in this watchlist:**
- Status, LastSeen, DaysSinceLastLog — calculated live by functions, never stored
- Transport, Origin, Purpose — live at the source level in [mssname]-sources
- SilentDet, Detections — live at the source level in [mssname]-sources

---

## Approved Field Values

### Category

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
| Capabilities | Tables produced by capabilities — UEBA output, Fusion ML |
| Other | Does not fit above — explain in Notes |

### Vetted

| Value | Meaning |
|---|---|
| Yes | Table has been investigated — Category and Details are accurate |
| No | Table appeared and has not yet been fully investigated |

Set Vetted = No for any table you cannot fully classify during the audit. Come back to it — do not leave it blank.

### Details — Format

Two to three sentences maximum. Describes what kind of data lands in this table at a broad level. For shared tables note that multiple sources write here. For ASIM tables note that health depends on source tables.

**Good:** `Shared table. Receives CEF format logs from network devices via a Linux forwarder running AMA. Palo Alto Networks firewalls and Fortinet VPN appliances both write here.`

**Good:** `ASIM normalized network session logs. Health depends entirely on the source tables feeding the parsers — primarily CommonSecurityLog and AzureFirewallDnsProxy.`

**Poor:** `Firewall logs.`

---

## The Audit Workflow

Work through all steps in order.

- [ ] **Step 0** — Complete license-entitlement-review.md for this client
- [ ] **Step 1** — Run the main table inventory query
- [ ] **Step 2** — Export results and open in Excel
- [ ] **Step 3** — Format as Excel Table
- [ ] **Step 4** — Save as dated workbook and upload to SharePoint
- [ ] **Step 5** — Fill in all manual fields
- [ ] **Step 6** — Run Usage volume query — add volume to Notes
- [ ] **Step 7** — Sort alphabetically by Table
- [ ] **Step 8** — Upload as [mssname]-tables watchlist
- [ ] **Step 9** — Proceed to sources watchlist build guide

---

## Step 1 — Main Table Inventory Query

Run this in the Microsoft Sentinel Logs blade. Returns every table that has ever received data.

**First run instruction:** Remove the time filter entirely on the first audit. You want every table that has ever existed. On subsequent audits keep the 30-day filter.

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
| extend CurrentStatus = case(
    DaysSinceLastLog <= 1, "Active",
    DaysSinceLastLog <= 7, "Review",
    DaysSinceLastLog > 7,  "Inactive",
    "No Data"
)
// ── Manual fields — fill these in after export ────────────────
| extend Category = "[Manual] — Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other"
| extend Details =  "[Manual] — Broad context about this table. Is it shared? ASIM? Single source? What kind of data lands here?"
| extend Vetted =   "No"
| extend Notes =    "[Manual] — Observations, action items, context"
// ─────────────────────────────────────────────────────────────
| project
    Table = $table,
    CurrentStatus,
    LastSeen,
    DaysSinceLastLog,
    Category,
    Details,
    Vetted,
    Notes
| order by CurrentStatus asc, LastSeen desc
```

**Note on CurrentStatus and DaysSinceLastLog:**
These are included in the query output for your reference during the audit — they help you prioritize which tables need attention. Do NOT include them in the watchlist upload. Delete these columns before uploading. The workbook calculates both live from the workspace at query time.

**Reading the output:**
- Sorted by CurrentStatus first — Inactive and Review at the top
- Deal with inactive and review tables first — these are gaps
- Every [Manual] field must be replaced before upload

---

## Step 2 — Export Results

Export the query output from the Logs blade using the Export button. Save the CSV.

---

## Step 3 — Format as Excel Table

1. Open the CSV in Excel
2. Click any cell in the data
3. Press **Ctrl+T**
4. Confirm My table has headers is checked
5. Click OK
6. Go to Home → Format → AutoFit Column Width

---

## Step 4 — Save and Upload to SharePoint

Save as:
```
[ClientName]-tables-audit-[YYYY-MM-DD].xlsx
```

Upload to the client SharePoint folder at:
```
02-Clients/[ClientName]/04-Watchlists/Current/
```

Do this before filling in any manual fields. This is your unmodified baseline snapshot.

---

## Step 5 — Fill In Manual Fields

Every [Manual] placeholder must be replaced. Use the pre-populated reference below for common Microsoft tables.

**Before filling in Details — check for shared tables:**
For any shared table run the appropriate breakout query from `resources/scratch.md` to understand what sources are writing to it. Use that knowledge to write the Details field accurately. The breakout queries are in the scratch file — AzureDiagnostics, CommonSecurityLog, Syslog, ThreatIntelIndicators, and all ASIM tables.

---

### Pre-Populated Reference — Common Tables

| Table | Category | Details | Vetted |
|---|---|---|---|
| SignInLogs | Identity | Single source. Microsoft Entra ID interactive sign-in logs. Every user sign-in attempt flows through here including successful, failed, and MFA-challenged authentications. | Yes |
| AADNonInteractiveUserSignInLogs | Identity | Single source. Non-interactive sign-ins — service-to-service authentication, token refresh, and background processes that authenticate without direct user involvement. | Yes |
| AADServicePrincipalSignInLogs | Identity | Single source. Service principal and application sign-ins. Applications, automation, and services authenticating to Azure AD resources. | Yes |
| AADManagedIdentitySignInLogs | Identity | Single source. Managed identity authentications from Azure resources authenticating to other services without stored credentials. | Yes |
| AuditLogs | Identity | Single source. Microsoft Entra ID directory audit events — role assignments, group changes, app registrations, conditional access policy changes. | Yes |
| AADRiskyUsers | Identity | Single source. Entra ID Identity Protection risky user signals — accounts flagged by Microsoft ML as potentially compromised. Requires Entra ID P2. | Yes |
| AADUserRiskEvents | Identity | Single source. Individual risk event detections per sign-in from Identity Protection. Requires Entra ID P2. | Yes |
| OfficeActivity | Compliance and Audit | Single source. Microsoft 365 Unified Audit Log covering SharePoint, OneDrive, Exchange, Teams, and admin operations across all M365 workloads. | Yes |
| AzureActivity | Cloud Infrastructure | Single source. Azure management plane activity — resource creation, deletion, role assignments, policy changes, and all ARM operations. | Yes |
| SecurityAlert | Cloud Infrastructure | Single source. Consolidated alert feed from all connected Microsoft security products flowing into Sentinel. | Yes |
| SecurityIncident | Cloud Infrastructure | Single source. All Sentinel and XDR incidents including multi-stage correlated incidents from Fusion ML. Primary source for ServiceNow ticket pipeline. | Yes |
| BehaviorAnalytics | Capabilities | Single source. UEBA behavioral anomaly output produced by Sentinel ML. Requires 14-21 day baseline period before meaningful output begins. | Yes |
| IdentityInfo | Capabilities | Single source. Enriched user identity data from Entra ID and on-premises AD used by detections for user context enrichment. | Yes |
| ThreatIntelIndicators | Threat Intelligence | Shared table. Multiple feeds write here identified by SourceSystem field. Each feed gets its own row in [mssname]-sources. Always filter on ExpirationDateTime > now() in any query. | Yes |
| ThreatIntelObjects | Threat Intelligence | Shared table. STIX 2.1 structured threat intelligence objects. Multiple feeds write here identified by SourceSystem. | Yes |
| Heartbeat | Endpoint | Single source. Azure Monitor Agent heartbeat — one per minute per enrolled machine. Primary mechanism for detecting AMA agent and machine availability. | Yes |
| SentinelHealth | Compliance and Audit | Single source. Microsoft Sentinel operational health for analytics rules, connectors, and automation. Critical for detecting silent failures before clients notice. | Yes |
| SentinelAudit | Compliance and Audit | Single source. All configuration changes to the Sentinel workspace — who changed what and when. | Yes |
| SecurityEvent | Endpoint | Single source. Windows Security Event Log from all enrolled machines via AMA. One row in sources covers the whole table regardless of how many machines are enrolled. | Yes |
| CommonSecurityLog | Firewall | Shared table. Receives CEF format logs from network devices via a Linux forwarder running AMA. Each vendor and product combination gets its own row in [mssname]-sources. Run the CommonSecurityLog breakout query to identify all sources. | Yes |
| Syslog | Endpoint | Shared table. Linux system logs and appliance logs. Each distinct host or appliance category gets its own row in [mssname]-sources. Run the Syslog breakout query to identify all sources. | Yes |
| AzureDiagnostics | Cloud Infrastructure | Shared table. Aggregated diagnostic logs from multiple Azure resources — Key Vault, Azure Firewall, NSG, App Service, and others. Each ResourceType and Category combination may need its own row in [mssname]-sources. Run the AzureDiagnostics breakout queries. | Yes |
| AzureMetrics | Cloud Infrastructure | Shared table. Performance and health metrics from Azure resources. Multiple ResourceTypes write here. Primarily operational value — assess whether any detections depend on it before adding to sources. | Yes |
| ASimNetworkSessionLogs | Network | Shared table. ASIM normalized network sessions. Health depends entirely on source tables feeding the parsers. Run the ASIM breakout query to identify contributing sources. | Yes |
| ASimAuditEventLogs | Compliance and Audit | Shared table. ASIM normalized audit events. Health depends entirely on source tables. | Yes |
| ASimWebSessionLogs | Network | Shared table. ASIM normalized web sessions. Health depends entirely on source tables. | Yes |
| CloudAppEvents | SaaS Application | Single source. Microsoft Defender for Cloud Apps events from enrolled SaaS applications. Requires XDR connector. | Yes |
| DeviceEvents | Endpoint | Single source. Microsoft Defender for Endpoint device events — process, network, file, and registry activity from enrolled devices. Requires XDR connector. | Yes |
| DeviceNetworkEvents | Endpoint | Single source. Network activity from MDE enrolled devices. Requires XDR connector. | Yes |
| DeviceLogonEvents | Endpoint | Single source. Logon events from MDE enrolled devices. Requires XDR connector. | Yes |
| DeviceProcessEvents | Endpoint | Single source. Process creation events from MDE enrolled devices. Requires XDR connector. | Yes |
| DeviceFileEvents | Endpoint | Single source. File system events from MDE enrolled devices. Requires XDR connector. | Yes |
| DeviceRegistryEvents | Endpoint | Single source. Registry modification events from MDE enrolled devices. Requires XDR connector. | Yes |
| DeviceImageLoadEvents | Endpoint | Single source. DLL and module load events from MDE enrolled devices. Requires XDR connector. | Yes |
| DeviceFileCertificateInfo | Endpoint | Single source. File signing certificate data from MDE enrolled devices. Requires XDR connector. | Yes |
| DeviceNetworkInfo | Endpoint | Single source. Network adapter and configuration data from MDE enrolled devices. Requires XDR connector. | Yes |
| DeviceInfo | Endpoint | Single source. Device inventory and enrollment status from MDE. Requires XDR connector. | Yes |
| EmailEvents | Email | Single source. Email flow events from Microsoft Defender for Office 365. Requires XDR connector. | Yes |
| EmailAttachmentInfo | Email | Single source. Email attachment data from MDO. Requires XDR connector. | Yes |
| EmailUrlInfo | Email | Single source. URL data from emails processed by MDO. Requires XDR connector. | Yes |
| EmailPostDeliveryEvents | Email | Single source. Post-delivery threat detections from MDO including detonation results. Requires XDR connector. | Yes |
| UrlClickEvents | Email | Single source. User click events on URLs in email and Teams from MDO. Requires XDR connector. | Yes |
| IdentityLogonEvents | Identity | Single source. Authentication events against on-premises Active Directory from Microsoft Defender for Identity. Requires XDR connector. | Yes |
| StorageBlobLogs | Cloud Infrastructure | Single source. Azure Storage blob access logs. Configured via diagnostic setting on each storage account. | Yes |
| Perf | Endpoint | Single source. Performance counters from AMA-enrolled machines — CPU, memory, disk utilization. | Yes |
| Anomalies | Capabilities | Single source. ML-detected behavioral anomalies from Sentinel fusion and anomaly detection rules. | Yes |
| UserPeerAnalytics | Capabilities | Single source. UEBA peer group analysis data used to detect deviations from peer behavior. | Yes |
| Usage | Compliance and Audit | Single source. Log Analytics ingestion and billing data per table. Operational tool — use for cost analysis in workbook ingestion tab. | Yes |
| AWSCloudTrail | Cloud Infrastructure | Single source. AWS CloudTrail API activity logs. Requires AWS connector configured. | Yes |

---

## Step 6 — Usage Volume Query

Run this separately to get ingestion volume per table. Add the daily volume to the Notes column for each table row.

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

Add to Notes as: `Daily Vol: 2.4 GB`. High-volume tables with no detections are cost optimization candidates — flag them.

---

## Step 7 — Sort Alphabetically

Sort by Table A to Z. Groups all rows consistently and makes the watchlist predictable.

---

## Step 8 — Upload as Watchlist

**Before uploading — remove these columns:**
- CurrentStatus — calculated live, goes stale immediately
- DaysSinceLastLog — calculated live, goes stale immediately
- LastSeen — goes stale immediately

**The five columns to upload:**
```
Table, Category, Details, Vetted, Notes
```

**Upload steps:**
1. Export the five-column tab as CSV
2. Navigate to Sentinel → Watchlists → Create new
3. Name: `[mssname]-tables`
4. Upload CSV
5. Set SearchKey to `Table`
6. Validate:

```kql
_GetWatchlist('[mssname]-tables')
| summarize TotalRows = count()
```

Confirm row count matches your spreadsheet.

---

## Step 9 — Proceed to Sources Watchlist

The tables watchlist is now complete. The next step is building `[mssname]-sources` — the data source registry where all monitoring detail lives.

Follow the complete process in:
```
06-watchlist-management/sources-watchlist-build-guide.md
```

That guide covers the AI-assisted process for building the sources watchlist table by table using the detections CSV, generating KQL sub-functions simultaneously, and uploading the completed watchlist.

---

## Known Limitations

**Lighthouse access scope** — access scoped to Sentinel workspace and Log Analytics. May not have access to VM extensions, resource tags, Azure Policy, or Resource Graph.

**Tables that never received data** — Log Analytics only registers a table after at least one record is written. A connector enabled but never sending data will not appear in any query. Set Vetted = No and document in Notes.

**Shared table breakout queries** — full breakout query library for all shared tables is in `resources/scratch.md`. Use those queries to understand what sources write to each shared table before filling in the Details field.

**ASIM tables** — health depends entirely on source table health. If a source table goes inactive the ASIM table may still appear active. Always note this in the Details field for ASIM tables.

---

## Audit Sign-Off

```
Audit Completed By:      [Engineer Name]
Date Completed:          [YYYY-MM-DD]
Client:                  [Client Name]
Total Tables:            [Number]
Active:                  [Number]
Inactive:                [Number]
Review:                  [Number]
No Data:                 [Number]
Vetted = Yes:            [Number]
Vetted = No:             [Number]
Next Audit Due:          [YYYY-MM-DD]
Notes:                   [Overall observations]
Watchlist Uploaded:      Yes / No
SharePoint Backup:       Yes / No — [link]
Sources Build Started:   Yes / No
```

No table may be signed off with Vetted = No without a documented action item and owner in Notes.

---

## DetectionCatalog — Building the Watchlist Using AI Assistance

> **Note:** This section documents the process for building the `[mssname]-detections` watchlist. All custom detections use the prefix `OC` followed by numbers — for example `OC12345 - Cloud Alerts`. This is the company naming convention for all custom analytic rules delivered to clients.

---

### Why We Are Building This

The DetectionCatalog is a ledger of all custom content deployed for a client. Every custom analytic rule has a row here. It answers three questions:

- **What do we have?** — every OC-series rule in one place
- **What does it do?** — plain English description per rule
- **What does it depend on?** — which tables and watchlists each rule needs

The GetEnrichedInventory function joins [mssname]-sources against [mssname]-detections on the Table field. Detection counts per source are derived automatically. Without this watchlist that join is not possible.

---

### Field Definitions

Six fields in this exact order:

| Field | What Goes Here |
|---|---|
| RuleId | OC##### — unique identifier, no other text |
| AnalyticRule | Full rule name exactly as it appears in Sentinel |
| Table | Comma separated exact table names this rule queries. Use `All` if the rule uses search * or union *. |
| Watchlist | Comma separated watchlist names referenced via _GetWatchlist(). Write None if none. |
| AlertClass | Security or Operational |
| Description | Plain English — what does this rule detect or monitor |

**AlertClass:**
- Security — threat detections, TI matching, anomaly rules, behavioral detections
- Operational — silent log source detections, heartbeat monitoring, Logic App failures, data limits

---

### Naming Convention and Upload

**Watchlist name:**
```
[mssname]-detections
```

**Upload steps:**
1. Complete the spreadsheet — all six fields, no [Manual] placeholders
2. Export as CSV
3. Sentinel → Watchlists → Create new
4. Name: `[mssname]-detections`
5. Upload CSV
6. Set SearchKey to `RuleId`
7. Validate:
```kql
_GetWatchlist('[mssname]-detections')
| summarize TotalRows = count()
```

---

### The AI-Assisted Build Process

#### Step 1 — Train the AI

```
I am building a detection catalog watchlist for Microsoft Sentinel.
I will be providing you with analytic rule information one rule at a time
or in batches. For each rule I provide please extract and output the
following fields in CSV format with these exact column headers:

RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description

Field instructions:
- RuleId: The OC##### identifier only — no other text
- AnalyticRule: The full rule name exactly as provided
- Table: Comma separated list of exact Log Analytics table names
  queried in the KQL. If the rule uses search * or union * write All.
  If you cannot determine the table from the KQL write [Manual]
- Watchlist: Comma separated watchlist names referenced in the KQL
  using _GetWatchlist(). Write None if no watchlists are referenced.
- AlertClass: Write Security if this is a threat or anomaly detection.
  Write Operational if this monitors platform health — silent log
  sources, heartbeats, Logic App failures, data limits, connector health.
- Description: One to two sentences in plain English explaining what
  this rule detects or monitors. Write it so any engineer can
  understand it without reading the KQL.

Output only the CSV rows — no headers, no explanation, no markdown.
I will add the header row myself.

Confirm you understand this format before I begin providing rules.
```

#### Step 2 — Filter Rules in Sentinel

```
Sentinel → Analytics → Active rules
→ Search bar: type OC
→ Sort alphabetically
→ Start from the top
```

#### Step 3 — Extract Rule Information

For each rule copy into a text editor in this order:
```
Name: OC12345 - Rule Name
ID: OC12345
Description: [rule description]
Query:
[full KQL query]
---
```

#### Step 4 — Send to AI in Batches of 50

After each batch copy the AI output into your spreadsheet. Clear the text editor and start the next batch.

#### Step 5 — Review Output

- Table = [Manual] — open the rule and fill in yourself
- AlertClass — verify Security vs Operational for every row
- Description — verify accuracy

#### Step 6 — Gap Detection Query

Run periodically to find rules missing from the watchlist:

```kql
// ============================================================
// DETECTION CATALOG GAP DETECTION
// Finds OC-prefixed rules in SentinelHealth not in [mssname]-detections
// ============================================================
let catalog = _GetWatchlist('[mssname]-detections')
| summarize by RuleId;
SentinelHealth
| where SentinelResourceType == "Analytics Rule"
| where SentinelResourceName matches regex @"^OC\d+"
| summarize LastRun = max(TimeGenerated)
    by SentinelResourceName
| extend RuleId = extract(@"^(OC\d+)", 1, SentinelResourceName)
| join kind=leftanti (
    catalog
) on RuleId
| extend Status = "Missing from DetectionCatalog — add to watchlist"
| project
    SentinelResourceName,
    RuleId,
    LastRun,
    Status
| order by SentinelResourceName asc
```

---

### Ongoing Maintenance

| Task | When | Who |
|---|---|---|
| Add new rules | When new OC rules are deployed | Platform or detection team |
| Run gap detection query | After every detection team release | Platform team |
| Review AlertClass assignments | When rule purpose changes | Platform team |
| Update Table field when KQL changes | When detection team modifies a rule | Detection team notifies platform team |
| Re-upload updated CSV | After any changes | Platform team |

---

*This runbook is maintained in sentinel-mssp-playbook under 02-data-sources. Update when the audit process changes, new source types require guidance, or new shared tables are identified.*