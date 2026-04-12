# Data Source Audit Runbook

> **Purpose:** This runbook guides an engineer through a complete data source audit for a managed client Microsoft Sentinel environment. It covers what to audit, why it matters, and how to produce the two watchlists that power everything else in the program.
>
> **Who uses this:** Engineers conducting a first-time audit or a periodic re-audit of a client environment. No prior experience with this program required — everything you need to know is in this document.
>
> **What you produce:**
> 1. `[mssname]-tables` watchlist — the table pipe registry
> 2. `[mssname]-detections` watchlist — the complete content ledger
>
> **After this runbook:** Proceed to `06-watchlist-management/sources-watchlist-build-guide.md` to build `[mssname]-sources`.
>
> **Prerequisites:** Complete `01-baseline/license-entitlement-review.md` for this client before starting.
>
> **Last Updated:** April 2026

---

## What Is a Data Source Audit and Why Do We Do It

A data source audit is a systematic review of every source of data flowing into a client's Microsoft Sentinel environment. The goal is to know exactly what is there, classify it, understand why it matters, and put monitoring in place so we know immediately if anything stops working.

**The problem this solves:**

Without a data source audit you are flying blind. You do not know what data you have. You do not know what detections depend on what data. You do not know when something stops flowing. A detection can go completely blind because a log source went silent three weeks ago and nobody noticed. A client asks why a threat was not caught and there is no good answer.

The audit gives us the answer to four questions at any point in time:
- What data are we collecting
- Why are we collecting it
- Is it flowing right now
- What breaks if it stops

**What makes data important:**

Not all data is equally important. A source earns its place in the system if it serves at least one of these purposes:

| Purpose | What It Means |
|---|---|
| Detection dependency | A custom detection queries this source. If it goes silent the detection goes blind. |
| Capability dependency | A Sentinel capability depends on it — UEBA, Threat Intelligence, Fusion ML. |
| Workbook dependency | A workbook tab reads from this source to display information to engineers or clients. |
| Report dependency | A scheduled report reads from this source. If it stops the report produces incomplete data. |
| Compliance requirement | A legal or regulatory obligation requires this data to be continuously collected. |
| Client requirement | The client has specifically asked for this data to be collected and monitored. |

If a source does not serve any of these purposes it does not need to be tracked. Every row in the system earns its place.

**What we track — two levels:**

**Table level** — what pipes exist. SignInLogs, CommonSecurityLog, AzureDiagnostics. A table is just a container. We track tables to know what exists and to catch new unknown tables appearing.

**Source level** — what flows through each pipe. Some tables have one source. Some have many. A Palo Alto firewall and a Fortinet VPN both write to CommonSecurityLog — they are two different sources in the same table, each with different detection value. We track at the source level because that is the level where things go silent without anyone noticing.

**The end result:**

When this process is complete we can tell a client exactly what data we are collecting, why we are collecting it, that we have monitoring in place for every important source, and that we will know before they do if anything breaks. That conversation happens in the audit deck. This process is what makes it possible.

---

## The Two Watchlists This Process Produces

**`[mssname]-tables`** — one row per table. Five fields. The pipe registry. Catches new unknown tables. Built by this runbook.

**`[mssname]-detections`** — one row per item that consumes data. Every detection, every report, every workbook query, every capability dependency. Built by this runbook. This feeds the sources watchlist build process.

**`[mssname]-sources`** — one row per data source within each table. Built by the sources watchlist build guide using the detections watchlist as input. This is where all monitoring lives.

---

## Understanding the Workspace

A Microsoft Sentinel workspace contains nine distinct types of components. Know these terms — they are used consistently throughout all program documentation.

| Category | What It Is | Examples |
|---|---|---|
| **Log Sources** | Specific originators writing data to a table. One table can have many. | Palo Alto → CommonSecurityLog |
| **Tables** | Log Analytics destinations where data lands. | SignInLogs, CommonSecurityLog |
| **Watchlists** | Persistent reference data queryable in KQL. | [mssname]-tables, [mssname]-sources |
| **Capabilities** | Features depending on multiple sources working together. | UEBA, Threat Feed, Fusion ML |
| **Detections** | Analytics rules that consume data and produce alerts. | OC-series rules |
| **Silent Detections** | Rules that monitor whether a source is sending data. | OC##### - SLD - SignInLogs Silence |
| **Automation** | Logic Apps and playbooks that respond to incidents. | ServiceNow ticket creation |
| **Integrations** | API keys and credentials that keep services connected. | Threat feed API key |
| **Workbooks** | Interactive live dashboards. | Audit Deck Workbook |

---

## The Audit Workflow — Overview

```
Step 0  — Complete license entitlement review
Step 1  — Run the table inventory query
Step 2  — Export and format in Excel
Step 3  — Save to SharePoint
Step 4  — Fill in table fields using the reference
Step 5  — Run usage volume query
Step 6  — Sort and upload as [mssname]-tables watchlist
Step 7  — Build the detections watchlist
Step 8  — Proceed to sources watchlist build guide
```

---

## Step 0 — Complete License Entitlement Review

Before auditing anything complete the license entitlement review for this client. It tells you what data sources should exist based on what the client is licensed for. If a connector is licensed but no data is flowing that is a gap to document.

File location:
```
01-baseline/license-entitlement-review.md
```

---

## Step 1 — Run the Table Inventory Query

Run this in the Microsoft Sentinel Logs blade. It returns every table that has ever received data including tables that went silent months ago.

**First audit:** Remove the time filter line entirely. You want every table that has ever existed.
**Subsequent audits:** Keep the 30-day filter.

```kql
// ============================================================
// MAIN TABLE INVENTORY
// FIRST AUDIT: Remove the time filter line entirely
// SUBSEQUENT AUDITS: Keep as-is
// ============================================================
search *
| where TimeGenerated > ago(30d) // REMOVE THIS LINE ON FIRST AUDIT
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
| extend Category = "[Manual]"
| extend Details =  "[Manual]"
| extend Vetted =   "No"
| extend Notes =    "[Manual]"
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

**Reading the output:**
- Sorted by CurrentStatus — Inactive and Review appear first. Deal with those first — they are gaps.
- CurrentStatus and DaysSinceLastLog are for your reference during the audit only. Remove them before uploading as a watchlist.
- Every [Manual] field must be replaced before upload.

---

## Step 2 — Export and Format in Excel

1. Export the query output using the Export button in the Logs blade
2. Open the CSV in Excel
3. Click any cell in the data
4. Press **Ctrl+T** — formats as an Excel Table
5. Confirm headers are checked — click OK
6. Go to Home → Format → AutoFit Column Width

---

## Step 3 — Save to SharePoint

Save the unmodified export before filling in anything:

```
[ClientName]-tables-audit-[YYYY-MM-DD].xlsx
```

Upload to:
```
02-Clients/[ClientName]/04-Watchlists/Current/
```

This is your baseline snapshot. Always save before editing.

---

## Step 4 — Fill In Table Fields

Replace every [Manual] placeholder using the reference table below. For tables not in the reference use the field definitions to fill in manually.

**Field definitions:**

| Field | What Goes Here |
|---|---|
| Table | Exact Log Analytics table name — never modify |
| Category | Type of security data — from approved list below |
| Details | 2-3 sentences — what kind of data lands here, is it shared, is it ASIM |
| Vetted | Yes if fully classified. No if you are unsure — come back to it. |
| Notes | Volume, observations, action items, anything worth knowing |

**Approved Category values:**
Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other

**Details examples:**

Good: `Shared table. Receives CEF format logs from network devices via a Linux forwarder running AMA. Palo Alto Networks firewalls and Fortinet VPN appliances both write here.`

Good: `ASIM normalized network sessions. Health depends entirely on source tables feeding the parsers.`

Poor: `Firewall logs.`

**For shared tables — run breakout queries first:**
Before filling in Details for AzureDiagnostics, CommonSecurityLog, Syslog, ThreatIntelIndicators, or any ASIM table — run the appropriate breakout query from `resources/scratch.md`. This shows you what is actually writing to that table in this environment so you can write an accurate Details entry.

---

### Pre-Populated Reference — Common Tables

Use these values directly. No research needed for these tables.

| Table | Category | Details | Vetted |
|---|---|---|---|
| SignInLogs | Identity | Single source. Microsoft Entra ID interactive sign-in logs. Every user sign-in attempt — successful, failed, and MFA-challenged. | Yes |
| AADNonInteractiveUserSignInLogs | Identity | Single source. Non-interactive sign-ins — service-to-service authentication, token refresh, background authentication without user involvement. | Yes |
| AADServicePrincipalSignInLogs | Identity | Single source. Service principal and application sign-ins. Applications and automation authenticating to Azure AD resources. | Yes |
| AADManagedIdentitySignInLogs | Identity | Single source. Managed identity authentications from Azure resources authenticating to other services without stored credentials. | Yes |
| AuditLogs | Identity | Single source. Entra ID directory audit events — role assignments, group changes, app registrations, conditional access policy changes. | Yes |
| AADRiskyUsers | Identity | Single source. Entra ID Identity Protection risky user signals. Requires Entra ID P2. | Yes |
| AADUserRiskEvents | Identity | Single source. Individual risk event detections per sign-in from Identity Protection. Requires Entra ID P2. | Yes |
| OfficeActivity | Compliance and Audit | Single source. Microsoft 365 Unified Audit Log — SharePoint, OneDrive, Exchange, Teams, and admin operations. | Yes |
| AzureActivity | Cloud Infrastructure | Single source. Azure management plane activity — resource creation, deletion, role assignments, policy changes, ARM operations. | Yes |
| SecurityAlert | Cloud Infrastructure | Single source. Consolidated alert feed from all connected Microsoft security products. | Yes |
| SecurityIncident | Cloud Infrastructure | Single source. All Sentinel and XDR incidents including Fusion ML correlated incidents. Primary source for ServiceNow ticket pipeline. | Yes |
| BehaviorAnalytics | Capabilities | Single source. UEBA behavioral anomaly output. Requires 14-21 day baseline period before meaningful output begins. | Yes |
| IdentityInfo | Capabilities | Single source. Enriched user identity data from Entra ID and on-premises AD used for detection context enrichment. | Yes |
| ThreatIntelIndicators | Threat Intelligence | Shared table. Multiple feeds write here identified by SourceSystem. Each feed gets its own row in [mssname]-sources. Always filter ExpirationDateTime > now() in queries. | Yes |
| ThreatIntelObjects | Threat Intelligence | Shared table. STIX 2.1 structured threat intelligence objects. Multiple feeds write here identified by SourceSystem. | Yes |
| Heartbeat | Endpoint | Single source. Azure Monitor Agent heartbeat — one per minute per enrolled machine. Primary mechanism for detecting AMA agent availability. | Yes |
| SentinelHealth | Compliance and Audit | Single source. Sentinel operational health for analytics rules, connectors, and automation. Critical for detecting silent failures. | Yes |
| SentinelAudit | Compliance and Audit | Single source. All configuration changes to the Sentinel workspace — who changed what and when. | Yes |
| SecurityEvent | Endpoint | Single source. Windows Security Event Log from all enrolled machines via AMA. One sources row covers all machines. | Yes |
| CommonSecurityLog | Firewall | Shared table. CEF format logs from network devices via Linux forwarder running AMA. Each vendor and product gets its own sources row. Run the CommonSecurityLog breakout query. | Yes |
| Syslog | Endpoint | Shared table. Linux system logs and appliance logs. Each distinct host or appliance category gets its own sources row. Run the Syslog breakout query. | Yes |
| AzureDiagnostics | Cloud Infrastructure | Shared table. Diagnostic logs from multiple Azure resources — Key Vault, Azure Firewall, NSG, App Service. Each ResourceType and Category may need its own sources row. Run the AzureDiagnostics breakout queries. | Yes |
| AzureMetrics | Cloud Infrastructure | Shared table. Performance and health metrics from Azure resources. Multiple ResourceTypes write here. Assess detection dependency before adding to sources. | Yes |
| ASimNetworkSessionLogs | Network | Shared table. ASIM normalized network sessions. Health depends entirely on source tables. Run ASIM breakout query. | Yes |
| ASimAuditEventLogs | Compliance and Audit | Shared table. ASIM normalized audit events. Health depends entirely on source tables. | Yes |
| ASimWebSessionLogs | Network | Shared table. ASIM normalized web sessions. Health depends entirely on source tables. | Yes |
| CloudAppEvents | SaaS Application | Single source. Microsoft Defender for Cloud Apps events. Requires XDR connector. | Yes |
| DeviceEvents | Endpoint | Single source. MDE device events — process, network, file, registry activity from enrolled devices. Requires XDR connector. | Yes |
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
| IdentityLogonEvents | Identity | Single source. Authentication events against on-premises AD from Microsoft Defender for Identity. Requires XDR connector. | Yes |
| StorageBlobLogs | Cloud Infrastructure | Single source. Azure Storage blob access logs. Configured via diagnostic setting on each storage account. | Yes |
| Perf | Endpoint | Single source. Performance counters from AMA-enrolled machines — CPU, memory, disk utilization. | Yes |
| Anomalies | Capabilities | Single source. ML-detected behavioral anomalies from Sentinel fusion and anomaly detection rules. | Yes |
| UserPeerAnalytics | Capabilities | Single source. UEBA peer group analysis data used to detect deviations from peer behavior. | Yes |
| Usage | Compliance and Audit | Single source. Log Analytics ingestion and billing data per table. Use for cost analysis in the audit deck ingestion tab. | Yes |
| AWSCloudTrail | Cloud Infrastructure | Single source. AWS CloudTrail API activity logs. Requires AWS connector. | Yes |

---

## Step 5 — Run Usage Volume Query

Run this separately. Add daily volume to the Notes column for each table.

```kql
// USAGE VOLUME
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
| project Table = DataType, DailyVolume, TotalGB_30Days = TotalGB
| order by TotalGB_30Days desc
```

Add to Notes as: `Daily Vol: 2.4 GB`

High-volume tables with no detections are cost optimization candidates — flag them.

---

## Step 6 — Sort and Upload as Tables Watchlist

**Sort:** Table column A to Z.

**Before uploading remove these columns:**
- CurrentStatus
- DaysSinceLastLog
- LastSeen

These go stale immediately. The workbook calculates them live.

**Upload the five remaining columns:**
```
Table, Category, Details, Vetted, Notes
```

**Upload steps:**
1. Export the five-column spreadsheet as CSV
2. Sentinel → Watchlists → Create new
3. Name: `[mssname]-tables`
4. Upload CSV
5. Set SearchKey to `Table`
6. Validate:

```kql
_GetWatchlist('[mssname]-tables')
| summarize TotalRows = count()
```

---

## Step 7 — Build the Detections Watchlist

The detections watchlist — `[mssname]-detections` — is the ledger of everything that consumes data in this environment. This includes custom detections, workbook queries, reports, and capability dependencies. It is the input to the sources watchlist build process.

**Why this comes before sources:**
The sources build guide uses this watchlist to understand which tables matter and why. Without it the AI has no context for what each source is used for.

---

### What Goes in the Detections Watchlist

Everything that reads from Sentinel data and would break or degrade if a source went silent:

| Consumer Type | RuleId Prefix | Examples |
|---|---|---|
| Custom detection | OC##### | OC12345 - Impossible Travel |
| Workbook query | WB-[name] | WB-DataSources, WB-ThreatIntel |
| Scheduled report | RPT-[name] | RPT-MonthlyFirewall, RPT-WeeklyAccess |
| Capability dependency | CAP-[name] | CAP-UEBA, CAP-ThreatIntel, CAP-Fusion |
| Client requirement | CLT-[name] | CLT-PCI-Firewall, CLT-HIPAA-Access |

**AlertClass values:**
- Security — threat detections, TI matching, anomaly rules
- Operational — silent log sources, heartbeat monitoring, Logic App failures, platform health
- Compliance — legal or regulatory data requirements
- Reporting — workbook queries and scheduled reports
- Capability — UEBA, Threat Intelligence, Fusion ML dependencies

---

### Field Definitions

Six fields in this exact order:

| Field | What Goes Here |
|---|---|
| RuleId | Identifier using prefix convention above — OC#####, WB-name, RPT-name, CAP-name, CLT-name |
| AnalyticRule | Full name as it appears in Sentinel, or plain English name for non-detection items |
| Table | Comma separated exact table names this item reads from. Use `All` for workspace-wide. |
| Watchlist | Comma separated watchlist names referenced. Write None if none. |
| AlertClass | Security / Operational / Compliance / Reporting / Capability |
| Description | Plain English — what does this detect, display, or require |

---

### Building the Detections Watchlist — Full Process

#### Part A — Custom Detections

**Step 1 — Train the AI**

```
I am building a content catalog watchlist for Microsoft Sentinel.
I will provide analytic rule information in batches. For each rule
extract and output these fields in CSV format:

RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description

Field instructions:
- RuleId: The OC##### identifier only
- AnalyticRule: Full rule name exactly as it appears in Sentinel
- Table: Comma separated exact table names queried in the KQL.
  Write All if the rule uses search * or union *.
  Write [Manual] if you cannot determine from the KQL.
- Watchlist: Comma separated _GetWatchlist() references. Write None if none.
- AlertClass: Security for threat and anomaly detections.
  Operational for platform health — silent log sources, heartbeats,
  Logic App failures, data limits, connector health.
- Description: 1-2 sentences plain English — what does this detect.

Output CSV rows only — no headers, no explanation, no markdown.
I will add headers myself.

Confirm you understand before I provide rules.
```

**Step 2 — Filter rules in Sentinel**
```
Sentinel → Analytics → Active rules → Search: OC → Sort alphabetically
```

**Step 3 — Extract each rule into text editor**
```
Name: OC12345 - Rule Name
ID: OC12345
Description: [rule description]
Query:
[full KQL]
---
```

**Step 4 — Send to AI in batches of 50**
Copy AI CSV output into your spreadsheet after each batch.

**Step 5 — Review every row**
- Table = [Manual] → open the rule and fill in yourself
- AlertClass → verify Security vs Operational for every row
- Description → verify accuracy

---

#### Part B — Workbook Queries

For each workbook tab that reads from Sentinel data add a row:

```
RuleId:       WB-[TabName]
AnalyticRule: Workbook — [Tab Name] Tab
Table:        [comma separated tables this tab queries]
Watchlist:    [watchlists this tab reads from or None]
AlertClass:   Reporting
Description:  [what this tab displays and what data it needs]
```

Example:
```
WB-DataSources, Workbook — Data Sources Tab, [mssname]-sources, [mssname]-tables, Reporting, Displays all tracked data sources with live health status and detection coverage
```

---

#### Part C — Scheduled Reports

For each scheduled report add a row:

```
RuleId:       RPT-[ReportName]
AnalyticRule: Report — [Report Name]
Table:        [tables this report queries]
Watchlist:    [watchlists this report reads from or None]
AlertClass:   Reporting
Description:  [what this report shows, who receives it, cadence]
```

---

#### Part D — Capability Dependencies

For each active capability add a row documenting what data it depends on:

```
RuleId:       CAP-UEBA
AnalyticRule: Capability — UEBA
Table:        SignInLogs, AuditLogs, SecurityEvent, BehaviorAnalytics
Watchlist:    None
AlertClass:   Capability
Description:  UEBA requires SignInLogs, AuditLogs, and SecurityEvent to be healthy. BehaviorAnalytics is the output table. Degraded input = degraded behavioral scoring.
```

Common capability rows to add:
- CAP-UEBA — SignInLogs, AuditLogs, SecurityEvent, BehaviorAnalytics
- CAP-ThreatIntel — ThreatIntelIndicators
- CAP-Fusion — SecurityAlert, SecurityIncident plus multiple alert sources
- CAP-SOCOptimization — SentinelHealth, analytics rules active

---

#### Part E — Client Requirements

After receiving the completed client questionnaire add a row for each data requirement the client identified:

```
RuleId:       CLT-[RequirementName]
AnalyticRule: Client Requirement — [Plain English Name]
Table:        [tables that contain the required data]
Watchlist:    None
AlertClass:   Compliance
Description:  [what the requirement is, which framework or client request drives it]
```

The client questionnaire is sent separately as part of onboarding. It captures compliance obligations, audit requirements, and any specific data the client has requested be monitored.

---

### Upload the Detections Watchlist

1. Complete all five parts above
2. Export as CSV
3. Sentinel → Watchlists → Create new
4. Name: `[mssname]-detections`
5. Set SearchKey to `RuleId`
6. Validate:

```kql
_GetWatchlist('[mssname]-detections')
| summarize Count = count() by AlertClass
```

You should see rows across Security, Operational, Reporting, Capability, and Compliance classes.

---

### Gap Detection Query

Run periodically to find OC-series rules missing from the watchlist:

```kql
let catalog = _GetWatchlist('[mssname]-detections')
| summarize by RuleId;
SentinelHealth
| where SentinelResourceType == "Analytics Rule"
| where SentinelResourceName matches regex @"^OC\d+"
| summarize LastRun = max(TimeGenerated) by SentinelResourceName
| extend RuleId = extract(@"^(OC\d+)", 1, SentinelResourceName)
| join kind=leftanti (catalog) on RuleId
| project SentinelResourceName, RuleId, LastRun,
    Status = "Missing from detections watchlist"
| order by SentinelResourceName asc
```

---

## Step 8 — Proceed to Sources Watchlist Build Guide

Both watchlists are now complete. The next step is `[mssname]-sources` — the data source registry where all monitoring detail and silent detection assignments live.

Follow the complete process in:
```
06-watchlist-management/sources-watchlist-build-guide.md
```

That guide uses `[mssname]-detections` as its primary input. It reorganizes the detection data by table, runs breakout queries to identify what is actually in the environment, and uses AI to produce one sources row per unique data source plus a KQL sub-function for each shared table.

---

## Known Limitations

**Lighthouse scope** — access scoped to Sentinel workspace and Log Analytics. May not have access to VM extensions, resource tags, or Azure Policy.

**Tables that never received data** — a connector enabled but never sending data will not appear in the inventory query. Document as Missing with Vetted = No.

**Shared table breakout queries** — the full library is in `resources/scratch.md`. Run these before filling in Details for any shared table.

**ASIM tables** — health depends entirely on source tables. If a source goes inactive the ASIM table may still appear active. Always note this in Details.

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
Detections Watchlist:    [Row count]
  Security:              [Number]
  Operational:           [Number]
  Reporting:             [Number]
  Capability:            [Number]
  Compliance:            [Number]
Next Audit Due:          [YYYY-MM-DD]
Watchlists Uploaded:     Yes / No
SharePoint Backup:       Yes / No — [link]
Sources Build Started:   Yes / No
```

No table may be signed off with Vetted = No without a documented action item and owner in Notes.

---

*This runbook is maintained in sentinel-mssp-playbook under 02-data-sources. Update when the audit process changes, new source types require guidance, or new consumer types are identified.*