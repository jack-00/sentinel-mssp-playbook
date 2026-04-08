# Data Source Audit Runbook

## Overview

This runbook explains how to perform a complete data source audit for a managed client environment in Microsoft Sentinel. The goal is to produce a clear, accurate, and organized spreadsheet that documents every data source flowing into a client's Sentinel workspace — what it is, whether it is healthy, why it matters, and what depends on it.

### Why We Audit by Table, Not by Data Connector

Microsoft Sentinel presents data sources through a connector UI that makes it easy to assume that if a connector shows as connected, everything is working correctly. This assumption is dangerous and wrong.

A data connector in Sentinel is essentially a configuration wizard. It helps you enable a data source but it does not guarantee that data is actually flowing, that all log categories are enabled, that the correct tables are being written to, or that the data volume is healthy. A connector can show a green connected status while silently sending no data at all.

Tables do not lie. The Log Analytics tables in your workspace are the ground truth. If a table has data, something is sending it. If a table is empty or stale, something is broken or was never configured — regardless of what the connector UI shows.

Auditing by table gives you an accurate, unambiguous picture of what data actually exists in the workspace. It also makes it immediately clear which detections are running blind, which watchlists have no data to match against, and where the gaps are in MITRE ATT&CK coverage.

This is the foundation of everything. You cannot tune detections, measure coverage, or have an honest conversation with a client about their security posture without first knowing exactly what data you have.

---

## Audit Checklist

Work through these steps in order. Do not move to the next step until the current one is complete. Check each item off as you go.

### Phase 1 — Discovery
- [ ] Run the full table inventory query — get every table in the workspace
- [ ] Run the last log received query — get the most recent record timestamp per table
- [ ] Run the daily volume query — get average ingestion volume per table over the last 30 days
- [ ] Export or copy results into the audit spreadsheet — one row per table

### Phase 2 — Classification
- [ ] Assign Status to every row — Active, Inactive, Review, or No Data
- [ ] Fill in Source (plain English name) for every table
- [ ] Fill in Vendor for every table
- [ ] Assign Category for every table using the approved category list
- [ ] Fill in Purpose for every table
- [ ] Fill in Collection for every table
- [ ] Fill in Detections — reference the AB-series detection catalog
- [ ] Fill in Watchlists where applicable
- [ ] Add Notes for anything that needs attention, context, or follow-up

### Phase 3 — Verification
- [ ] For every table marked Active — verify configuration is complete and correct
- [ ] For every table marked Inactive — identify root cause and document in Notes
- [ ] For every table marked Review — document what specifically needs attention
- [ ] For every table marked No Data — confirm whether it is not applicable or not yet configured, document in Notes
- [ ] Final review — confirm no tables are missing from the inventory
- [ ] Sign off — document date completed and engineer name at the bottom of the spreadsheet

---

## Spreadsheet Structure

The audit spreadsheet has one row per table. Columns use short header names to keep the spreadsheet clean and readable. The full definition of what goes in each column is documented below. Fill in every field for every row. If a field is not applicable, write N/A — do not leave cells blank.

### Column Header Reference

| Short Header | Full Name | Description |
|---|---|---|
| Table | Table Name | Exact Log Analytics table name |
| Source | Data Source | Plain English description of what sends this data |
| Vendor | Vendor | Company or product that produces this data source |
| Category | Category | Type of data — select from approved list |
| Status | Status | Current health — Active, Inactive, Review, No Data |
| Last Seen | Last Log Received | Timestamp of most recent record |
| Daily Vol | Avg Daily Volume | Average ingestion volume over last 30 days |
| Purpose | Why It Matters | Plain English security impact statement |
| Collection | How It Collects | Mechanism by which data reaches the workspace |
| Detections | Detections That Rely On It | AB-series detection IDs and names that query this table |
| Watchlists | Watchlists That Use It | Watchlists referencing this table |
| Notes | Notes | Context, issues, action items, follow-up |

---

### Column Definitions

---

#### Table / Table Name
**What goes here:** The exact Log Analytics table name as it appears in the workspace. This is the technical name used in KQL queries.

**Format:** Exact string, no spaces, match case exactly as it appears in query results.

**Example:** `SignInLogs`

**Note:** Do not use friendly names here. The table name must be exact so that engineers can reference it directly in KQL without translation.

---

#### Source / Data Source
**What goes here:** A plain English description of what is sending data to this table. Write it so a client can understand it without technical knowledge.

**Format:** Free text. Capitalize properly. Do not use abbreviations unless they are universally understood.

**Example:** `Microsoft Entra ID Sign-in Logs`

**Note:** This is different from the table name. The table name is for engineers. The data source description is for everyone.

---

#### Vendor
**What goes here:** The company or product that produces this data source.

**Format:** Free text. Use the official vendor name.

**Examples:**
- `Microsoft`
- `Palo Alto Networks`
- `Fortinet`
- `CrowdStrike`
- `Okta`
- `Amazon Web Services`

**Note:** For Microsoft-native sources the vendor is always Microsoft. For third-party sources use the product vendor name, not the reseller or integrator.

---

#### Category / Category
**What goes here:** The category that best describes what type of data this source provides. Select from the approved list below.

**Approved categories:**
- `Identity` — user authentication, directory services, account management
- `Endpoint` — device telemetry, EDR, antivirus
- `Email` — mail flow, phishing, attachment scanning
- `Network` — traffic logs, DNS, DHCP, proxy
- `Firewall` — perimeter firewall and NGFW logs
- `Cloud Infrastructure` — Azure resource logs, management plane activity, cloud workload protection
- `SaaS Application` — third-party cloud applications such as Salesforce, ServiceNow, Okta
- `Threat Intelligence` — IOC feeds, indicator matching
- `Compliance and Audit` — M365 audit logs, regulatory compliance events
- `Vulnerability Management` — scanner results, CVE data, asset risk scoring
- `Authentication` — MFA, SSO, PAM, privileged access
- `Other` — anything that does not fit the above categories. Add a note explaining what it is.

---

#### Status / Status
**What goes here:** The current health status of this data source based on your review. Assign one of four values.

**Approved values:**

🟢 **Active** — Data is flowing within the expected timeframe for this source type. Volume looks normal. No issues identified. No action needed.

🔴 **Inactive** — The table has historical data but ingestion has stopped. Something broke at some point after initial configuration. This requires immediate investigation. Document the suspected cause in Notes.

🟡 **Review** — Data is flowing but something is not right. Volume looks abnormal, timestamps are inconsistent, only some expected log categories are present, or the source appears partially configured. An engineer needs to look closer before this can be marked Active.

⚫ **No Data** — The table has never received data, or the data source does not exist in this client's environment. Use Notes to clarify whether this is not applicable for this client or whether it was never configured and should be.

**Important:** Do not mark a source Active just because the connector shows as connected. Verify the table has recent data using the query results from Phase 1.

---

#### Last Seen / Last Log Received
**What goes here:** The timestamp of the most recent record in this table.

**Format:** `YYYY-MM-DD HH:MM UTC`

**Example:** `2026-04-07 22:14 UTC`

**Note:** This comes directly from the Phase 1 KQL query results. Copy the value exactly. If the table has no data this field should read `Never`.

---

#### Daily Vol / Avg Daily Volume
**What goes here:** The average daily ingestion volume for this table over the last 30 days.

**Format:** Use MB for smaller sources, GB for larger ones. Round to one decimal place.

**Examples:**
- `1.2 GB`
- `450 MB`
- `< 1 MB`

**Note:** This comes from the Phase 1 KQL query results. Volume gives you a baseline. If the volume drops significantly in a future audit it is an early warning that something is wrong — even if the source still appears Active.

---

#### Purpose / Why It Matters
**What goes here:** One or two sentences explaining why this data source is important from a security perspective. Write it so a client can understand it. This is the answer to the question "so what?"

**Format:** Plain English. No technical jargon. Client readable.

**Example:** `Sign-in logs are the primary source for detecting compromised accounts, impossible travel, password spray attacks, and unauthorized access attempts. Without this data, identity-based attacks are invisible.`

**Note:** Think about what an attacker can do that you would miss if this data source disappeared. That is your answer.

---

#### Collection / How It Collects
**What goes here:** The mechanism by which data gets from the source into the Log Analytics workspace. This tells engineers how the integration works at a high level.

**Approved values:**
- `Microsoft Connector` — native Microsoft service-to-service integration, no agent required
- `AMA Agent` — Azure Monitor Agent installed on a host or VM sending logs to a DCR
- `DCR / Data Collection Rule` — data collection rule routing logs from a source to the workspace
- `CEF / Syslog` — third-party device sending logs over Syslog or Common Event Format via a forwarder
- `REST API` — source pushing data via API, often through a Logic App or Function App
- `Logic App` — custom Logic App polling or receiving data from a third-party source
- `Manual / Custom` — custom ingestion pipeline not covered by the above

**Note:** If you are unsure how a source is collecting, the Phase 3 verification steps below will help you identify it.

---

#### Detections / Detections That Rely On It
**What goes here:** The AB-series detection IDs and names from the master detection catalog that query this table. If a detection goes blind when this table has no data, list it here.

**Format:** List each detection on a new line within the cell. Include both ID and name.

**Example:**
```
AB00012 - Possible Credential Stuffing
AB00034 - Impossible Travel - Successful Sign-In
AB00041 - Legacy Authentication Sign-In Detected
```

**Note:** If no custom detections query this table write `None - Microsoft Content Hub only` if Content Hub rules use it, or `None` if nothing queries it. A table with no detections depending on it may be a candidate for review — why is it being ingested if nothing uses it?

---

#### Watchlists / Watchlists That Use It
**What goes here:** Any watchlists that reference data from this table or are used in conjunction with detections that depend on this table.

**Format:** List watchlist names separated by commas, or write `None`.

**Example:** `VIP Users, Critical Assets, Domain Controllers`

---

#### Notes / Notes
**What goes here:** Anything that does not fit the other columns. This is a free-text field for context, issues, follow-up items, and client-specific information.

**Use this field for:**
- Root cause of an Inactive status
- What specifically needs attention for a Review status
- Whether a No Data source is not applicable or not yet configured
- Known false positive patterns for detections using this table
- Client-specific exceptions or context
- Outstanding action items with owner and target date

**Format:** Free text. Date-stamp any action items. Example: `2026-04-07 — Syslog forwarder stopped sending after network change. Ticket opened with client IT. Follow up by 2026-04-14.`

---

## Example Table

The following shows two example rows to illustrate how the spreadsheet should look when populated. Column headers use the short names. Values are fictional but representative.

| Table | Source | Vendor | Category | Status | Last Seen | Daily Vol | Purpose | Collection | Detections | Watchlists | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `SignInLogs` | Microsoft Entra ID Sign-in Logs | Microsoft | Identity | 🟢 Active | 2026-04-07 22:14 UTC | 2.4 GB | Primary source for detecting compromised accounts, password spray, impossible travel, and unauthorized access. Without this, identity-based attacks are invisible. | Microsoft Connector | AB00012 - Possible Credential Stuffing, AB00034 - Impossible Travel | VIP Users, Terminated Employees | Healthy. Non-interactive and service principal sign-in logs also confirmed flowing. |
| `CommonSecurityLog` | Palo Alto Networks Firewall | Palo Alto Networks | Firewall | 🟡 Review | 2026-04-06 03:42 UTC | 850 MB | Perimeter traffic logs used to detect outbound C2 communication, port scanning, and lateral movement across network segments. | CEF / Syslog | AB00055 - Outbound Connection to Known Malicious IP | Network Ranges, Critical Assets | Volume dropped 60% vs prior 30-day average. Last log over 19 hours ago. Possible Syslog forwarder issue. Marked Review pending investigation. |

---

## Phase 1 — Discovery Queries

Run these queries in the Microsoft Sentinel Logs blade. These are templates — substitute the placeholder values where indicated. All queries are designed to be copied and pasted directly.

---

### Query 1 — Full Table Inventory
**Purpose:** Get every table in the workspace that has ever received data. This is your starting list — one row per table in your spreadsheet.

**What it tells you:** Every table that exists in this workspace, when it last received data, and whether it has received anything in the last 30 days.

```kql
// Full Table Inventory
// Returns all tables with data, sorted by most recently active
// Run this first — this becomes your row list for the spreadsheet
search *
| summarize
    LastLogReceived = max(TimeGenerated),
    TotalRecords = count()
    by $table
| extend DaysSinceLastLog = datetime_diff('day', now(), LastLogReceived)
| extend Status = case(
    DaysSinceLastLog <= 1, "🟢 Active",
    DaysSinceLastLog <= 7, "🟡 Review",
    DaysSinceLastLog > 7, "🔴 Inactive",
    "⚫ No Data"
)
| project
    TableName = $table,
    LastLogReceived,
    DaysSinceLastLog,
    TotalRecords,
    Status
| order by LastLogReceived desc
```

**How to interpret results:**
- Every row is a table — add each one to your spreadsheet
- `LastLogReceived` goes directly into the Last Log Received column
- `DaysSinceLastLog` helps you assign Status — use it as a guide, not a rule. Use your judgment. A firewall that hasn't logged in 2 days may warrant Review even though the query marks it Active
- Tables with very low `TotalRecords` and old `LastLogReceived` are good candidates for Review or Inactive

---

### Query 2 — Average Daily Volume Per Table
**Purpose:** Get the average daily ingestion volume for each table over the last 30 days.

**What it tells you:** How much data each source is generating on a typical day. This becomes your volume baseline. Future audits that show significant drops are an early warning sign.

```kql
// Average Daily Volume Per Table — Last 30 Days
// Returns ingestion volume per table in MB
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize
    TotalGB = round(sum(Quantity) / 1000, 3),
    AvgDailyMB = round(avg(Quantity), 1)
    by DataType
| extend AvgDailyVolume = strcat(tostring(AvgDailyMB), " MB")
| project
    TableName = DataType,
    AvgDailyVolume,
    TotalGB_30Days = TotalGB
| order by TotalGB_30Days desc
```

**How to interpret results:**
- Match `TableName` in this query to `TableName` in Query 1 to fill in the Avg Daily Volume column
- Tables with `0` or near-zero volume despite showing a recent `LastLogReceived` may indicate a source that fires rarely — check Notes column to document this so it is not confused with an issue
- Tables with unexpectedly high volume may indicate a misconfigured source sending too much data — flag these as Review

---

### Query 3 — Last Log Per Table With Timestamp
**Purpose:** Get the precise last log timestamp per table with more detail than Query 1.

**What it tells you:** Exactly when the last record arrived for each table. Use this to fill in the Last Log Received column precisely.

```kql
// Last Log Received — Precise Timestamp Per Table
// Use this to fill in the Last Log Received column
// Replace TABLE_NAME with the specific table you want to check
// Or remove the where clause to check all tables
search *
| summarize LastLogReceived = max(TimeGenerated) by $table
| extend LastLogReceived_UTC = format_datetime(LastLogReceived, 'yyyy-MM-dd HH:mm')
| project
    TableName = $table,
    LastLogReceived_UTC
| order by LastLogReceived desc
```

**Template note:** To check a single table replace the search with the table name directly:
```kql
// Single table version — replace SignInLogs with your table name
SignInLogs
| summarize LastLogReceived = max(TimeGenerated)
| extend LastLogReceived_UTC = format_datetime(LastLogReceived, 'yyyy-MM-dd HH:mm')
```

---

### Query 4 — Tables With No Recent Data
**Purpose:** Quickly surface any table that has not received data in the last 48 hours.

**What it tells you:** Your immediate attention list. These are the tables to investigate first.

```kql
// Tables With No Recent Data — Last 48 Hours
// These tables need immediate attention
search *
| summarize LastLogReceived = max(TimeGenerated) by $table
| where LastLogReceived < ago(48h)
| extend HoursSinceLastLog = datetime_diff('hour', now(), LastLogReceived)
| project
    TableName = $table,
    LastLogReceived,
    HoursSinceLastLog
| order by HoursSinceLastLog desc
```

**How to interpret results:**
- Any table here that is a critical data source — identity, endpoint, firewall — should be flagged Inactive or Review immediately
- Some tables legitimately receive data infrequently — for example threat intelligence tables may only update daily or weekly. Use judgment and document the expected frequency in Notes

---

### Query 5 — Sentinel Health — Connector Status
**Purpose:** Check the health status of data connectors as reported by Sentinel itself. Use this alongside table queries — not instead of them.

**What it tells you:** Which connectors Sentinel considers unhealthy. Cross-reference with your table inventory to identify whether an unhealthy connector matches a table with no recent data.

```kql
// Sentinel Connector Health
// Cross-reference with table inventory results
SentinelHealth
| where TimeGenerated > ago(24h)
| where SentinelResourceType == "Data Connector"
| summarize
    LastStatus = arg_max(TimeGenerated, Status, Description)
    by SentinelResourceName
| project
    ConnectorName = SentinelResourceName,
    LastStatus = Status,
    LastChecked = TimeGenerated,
    Description
| order by LastStatus asc
```

**Important note:** A connector showing Healthy here does not mean data is flowing correctly. Always verify with the table queries above. Use this query to catch connectors that Sentinel itself has flagged — it is a supplement to table-level verification, not a replacement.

---

## Phase 2 — Classification Guidance

After running the Phase 1 queries and populating the table name, last log received, volume, and initial status fields, work through each row and fill in the remaining columns. The following guidance covers the fields that require human judgment.

---

### Assigning Status

Use the Phase 1 query results as your starting point but apply judgment. The query assigns status based on time thresholds — your assignment should factor in what the source is and what is normal for it.

**Guidance by source type:**

- **Identity sources** (SignInLogs, AuditLogs) — these should have data every few minutes in any active environment. If last log is more than 4 hours ago mark as Review. More than 24 hours mark Inactive.

- **Endpoint sources** (DeviceEvents, DeviceLogonEvents) — depend on device activity. Low volume on a weekend may be normal. No data for 48 hours in a business environment is Inactive.

- **Firewall and network sources** — should be near-continuous in any production environment. More than 6 hours with no data warrants Review. More than 24 hours is Inactive.

- **Threat intelligence tables** — may update once daily or less frequently. Check the vendor's update cadence before marking Inactive.

- **Compliance and audit tables** — may only populate when specific events occur. No data may mean no events, not a configuration problem. Document this distinction in Notes.

---

### Identifying How It Collects

If you are unsure how a source is getting data into the workspace, use the following approach to identify it:

**Step 1 — Check the Data Connectors page**
In the Sentinel or Defender portal navigate to Data Connectors. Find the connector associated with this table. The connector page will describe the collection method and show configuration status.

**Step 2 — Check for DCRs**
In the Azure portal navigate to Monitor → Data Collection Rules. If a DCR exists for this source it will be listed here. The DCR will show the source, the transformation if any, and the destination workspace.

**Step 3 — Check for AMA agents**
If the source is a Windows or Linux machine navigate to the machine in Azure portal → Extensions. If the Azure Monitor Agent extension is installed it is using AMA. Cross-reference with any DCRs that reference this machine.

**Step 4 — Check Logic Apps**
If the source is a third-party SaaS application or API-based source navigate to Logic Apps in the Azure portal filtered to the client's subscription. Look for Logic Apps with names referencing the vendor or data source.

**Use case example — Palo Alto firewall showing as Review:**
You see `CommonSecurityLog` with a last log received timestamp from 19 hours ago. The connector page shows connected. To investigate:
1. Check if a Syslog forwarder VM exists in the client's environment — this is typically a Linux VM that receives CEF logs from the firewall and forwards them to the workspace
2. Navigate to that VM in Azure portal and check if it is running
3. SSH to the VM and check the Syslog daemon status — `sudo systemctl status syslog` or `sudo systemctl status rsyslog`
4. Check the Azure Monitor Agent on the forwarder VM is healthy
5. Verify the firewall itself is still configured to send Syslog to the forwarder IP

This single example covers the most common reason a firewall source goes Inactive — the forwarder VM stopped, was rebooted, or the firewall Syslog destination was changed.

---

### Filling In Purpose (Why It Matters)

Write one to two sentences that answer the question: what would an attacker be able to do undetected if this data source disappeared?

**Good example:** `Firewall logs are the primary source for detecting outbound connections to known malicious infrastructure, port scanning activity, and unauthorized lateral movement across network segments. Without this data, network-based attack patterns are completely invisible.`

**Poor example:** `This table contains firewall logs.`

The good example tells the client what is at stake. The poor example just restates what the column header already implied. Always write for impact.

---

## Phase 3 — Verification Guidance

Phase 3 is about confirming that active sources are fully and correctly configured — not just that some data is flowing. Partial configurations are common and dangerous. A source that is streaming one log category but missing three others appears Active but has significant blind spots.

---

### Verification Approach by Collection Method

**Microsoft Connector sources (Entra ID, Azure Activity, M365)**
1. Navigate to the connector page in Sentinel
2. Confirm all log categories are enabled — for Entra ID this means interactive sign-ins, non-interactive sign-ins, audit logs, service principal sign-ins, managed identity sign-ins, provisioning logs, and identity protection events. Enabling only interactive sign-ins is a common partial configuration
3. Confirm diagnostic settings in the source service are routing all categories to the correct workspace — check Azure portal → Entra ID → Monitoring → Diagnostic Settings
4. Run a targeted query for each expected log category to confirm data exists:
```kql
// Template — replace TABLE_NAME with each expected table
// Run once per expected log category
TABLE_NAME
| where TimeGenerated > ago(24h)
| summarize Count = count()
```

**AMA Agent sources (Windows Security Events, Syslog)**
1. Confirm the Azure Monitor Agent is installed and healthy on the source machine — Azure portal → Virtual Machines → Extensions
2. Confirm a DCR exists and is associated with the machine — Monitor → Data Collection Rules
3. Confirm the DCR is routing to the correct workspace
4. Confirm the correct event IDs or Syslog facilities are configured in the DCR — a DCR that only collects System logs but not Security logs will appear healthy while missing the most important events
5. Validate with a targeted query:
```kql
// Windows Security Events verification
// Checks for critical event IDs that should always be present
SecurityEvent
| where TimeGenerated > ago(24h)
| where EventID in (4624, 4625, 4648, 4672, 4688, 4720, 4728)
| summarize Count = count() by EventID
| order by EventID asc
```
If any of these critical event IDs are missing from the results the DCR is not collecting them — this is a partial configuration and should be marked Review.

**CEF / Syslog sources (firewalls, network devices)**
1. Confirm the Syslog forwarder VM is running and healthy
2. Confirm the Azure Monitor Agent on the forwarder is healthy
3. Confirm the source device is configured to send Syslog to the forwarder IP on the correct port
4. Confirm the DCR on the forwarder is routing CEF data to the correct workspace
5. Validate with:
```kql
// CEF source verification
// Replace with your vendor's DeviceVendor value
CommonSecurityLog
| where TimeGenerated > ago(24h)
| where DeviceVendor == "Palo Alto Networks" // Replace with actual vendor
| summarize Count = count(), LastLog = max(TimeGenerated)
```

**REST API / Logic App sources**
1. Navigate to the Logic App in Azure portal
2. Check the run history — look for recent successful runs and confirm the trigger is firing on schedule
3. Check for failed runs — Logic App failures often show detailed error messages that identify the root cause
4. Confirm the API credentials used by the Logic App have not expired — this is the most common cause of API-based source failures
5. Validate with a targeted table query to confirm data is arriving at the expected frequency

---

### Common Issues and What They Mean

| Symptom | Likely Cause | Action |
|---|---|---|
| Connector shows connected, table has no data | Diagnostic settings not configured upstream | Enable diagnostic settings in source service |
| Table has data but volume is much lower than expected | Partial log category configuration | Review DCR or connector settings for missing categories |
| Table was active, now Inactive — no changes made | Agent stopped, VM rebooted, or credential expired | Check agent health, VM status, and API credentials |
| Table shows data but timestamps are hours old | Forwarder delay or batch ingestion lag | Check forwarder health and ingestion pipeline |
| New table appeared that was not there before | New connector enabled or agent updated | Document and classify in spreadsheet |
| Table exists but no detections query it | Detection gap or legacy table no longer used | Flag in Notes — evaluate whether ingestion is justified |

---

## Sign-Off

When all three phases are complete, add the following sign-off block to the bottom of the spreadsheet:

```
Audit Completed By:   [Engineer Name]
Date Completed:       [YYYY-MM-DD]
Environment:          [Client Name]
Total Tables Audited: [Number]
Active:               [Number]
Review:               [Number]
Inactive:             [Number]
No Data:              [Number]
Next Audit Due:       [YYYY-MM-DD]
Notes:                [Any overall observations]
```

Any table marked Review or Inactive must have a documented action item in its Notes field before sign-off is complete. A table cannot be left in Review or Inactive status without a next step and an owner.

---

*This runbook is maintained in the sentinel-mssp-playbook repository under 02-data-sources. Update it when the audit process changes or new source types require specific verification guidance. Version and date any significant changes.*