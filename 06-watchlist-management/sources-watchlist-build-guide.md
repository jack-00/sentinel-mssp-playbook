# Building the Sources Watchlist — Process and AI Guidance

> **Purpose:** This document guides an engineer through the complete process of building the `[mssname]-sources` watchlist and the associated KQL sub-functions. It covers the full workflow from organizing detection data through to uploading a validated watchlist and deploying functions to Sentinel.
>
> **Who uses this:** Any engineer building or updating the sources watchlist. Works with any capable LLM including internal company tools.
>
> **What you produce:**
> 1. A completed `[mssname]-sources` watchlist spreadsheet ready for upload to Sentinel
> 2. KQL sub-functions for all shared tables ready for deployment
>
> **Where outputs go:**
> - Completed spreadsheet → `02-Clients/[ClientName]/04-Watchlists/Current/`
> - KQL functions → `02-Clients/[ClientName]/06-Functions/` and `01-Program/Functions/` if generic
> - Master prompts → `01-Program/AI-Prompts/sources-watchlist-prompt.md`
>
> **Prerequisites:** Complete the data source audit runbook and upload `[mssname]-tables` before starting this process. You need to know what tables exist in the environment before documenting what sources write to them.
>
> **Last Updated:** April 2026

---

## Why This Watchlist Exists and Why It Is Built This Way

The sources watchlist is the most important watchlist in the entire system. Everything that makes the workbook valuable — health monitoring per source, detection coverage per source, silent detection coverage, the audit deck data sources tab — all of it reads from this watchlist.

Every table gets at least one row here. Single source tables get one row. Shared tables — AzureDiagnostics, CommonSecurityLog, Syslog — get one row per distinct sub-source within them.

**Why we build it from detections:**

We do not start by guessing what sources matter. We start from what we know for certain — our detections. Every detection queries specific tables and often filters on specific sub-sources within those tables. If a detection depends on a source then that source is important. If it goes silent the detection goes blind. That is the clearest possible justification for tracking a source.

Building from detections also gives the AI the context it needs to produce accurate descriptions, purposes, and monitoring frequency recommendations — because it can see exactly what the data is being used for.

**Why we think table-first not detection-first:**

The natural instinct is to go through each detection one by one. The problem with this approach is that multiple detections often query the same table and the same sub-source within it. If you go detection by detection you end up writing the same function multiple times and creating redundant watchlist rows.

The correct approach is to invert the thinking:

```
Table
  ↓
What unique sources write to this table
  ↓
What detections depend on each source
  ↓
One function per unique source
One watchlist row per unique source
Multiple detections can share the same source and function
```

This produces zero redundancy. One function per shared table covering all its sub-sources. Every detection that depends on those sub-sources calls the same function.

---

## The Complete Workflow — Overview

```
Step 1 — Reorganize detections by table using AI
Step 2 — Run breakout queries to see what is actually in the environment
Step 3 — Train the AI with the full prompt
Step 4 — Work through tables one by one — AI produces watchlist rows AND functions
Step 5 — Transfer watchlist rows to spreadsheet as you go
Step 6 — Save KQL functions to functions document as you go
Step 7 — Handle tables without detections
Step 8 — Handle pending client input sources
Step 9 — Final review before upload
Step 10 — Review and finalize functions
Step 11 — Upload watchlist and deploy functions
```

---

## Standards and Taxonomy

Before starting know the approved values for every field. The AI uses these — knowing them helps you verify its output.

**Category — choose one:**
Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other

**Origin — choose one:**
Microsoft Entra ID / Microsoft 365 / Microsoft Defender / Microsoft Azure / Microsoft Sentinel / Windows OS / Network Device / AWS / Salesforce / Zoom / Keeper Security / SecureW2 / Drupal / Multiple Sources / Custom / Unknown

**Transport — choose one:**
Microsoft Connector / XDR Connector / AMA — DCR / AMA — DCR — DCE / CEF — AMA — DCR / Cribl — AMA — DCR / Cribl — DCR / Diagnostic Setting / Logic App — Polling / Logic App — Webhook / REST API — Push / TAXII — Polling / Sentinel Native / ASIM Parser / Application Insights SDK

**SLA:** True / False

**MonitoringFrequency:** None / 1h / 5h / 15h / 24h / 48h

**SLS:** AB##### or Missing

---

## Prerequisites — Have These Ready

1. **Detections CSV** — your `[mssname]-detections` spreadsheet with RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description
2. **Sources spreadsheet** — open with correct column headers in this exact order:
```
Table, LogSource, Category, Origin, Transport, Description,
Purpose, SLA, DataConnector, DCRName, DCEName, FunctionName,
SLS, MonitoringFrequency, Notes
```
3. **Functions document** — a new file called `functions-in-progress.kql` in the client's `06-Functions` folder in SharePoint
4. **Text editor** — staging area for input before sending to AI
5. **A capable LLM** — use your internal company LLM for anything involving client-specific information

---

## Step 1 — Reorganize Detections by Table

Your detections spreadsheet has a Table field with comma separated table names. A detection that queries three tables appears once in the spreadsheet but belongs under all three tables. A simple Excel sort does not handle this cleanly.

**Why this step matters:**
Before you can work table by table you need a clear view of which detections belong to which table. This prevents redundancy — you will see that multiple detections share the same table, which tells you they may share the same source and therefore the same function.

**How to do it — feed the entire detections CSV to AI:**

Copy your entire detections CSV content and paste it to the AI with this prompt:

```
I have a detection catalog with these columns:
RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description

The Table field contains comma separated table names — one detection
may query multiple tables.

Please reorganize this into a table-first view where I can see
each table and which detections depend on it.

Output as a simple two column list:

Table | Detections
AzureDiagnostics | OC00034 - Detection Name, OC00041 - Detection Name
SignInLogs | OC00012 - Detection Name, OC00019 - Detection Name
CommonSecurityLog | OC00055 - Detection Name

Rules:
- If a detection queries multiple tables list it under each table
- Group all detections for the same table on one row comma separated
- Sort tables alphabetically
- Include the RuleId and rule name for each detection
```

The AI returns a clean table-first reference. Save this — it is your working map for the rest of this process. You will go through it table by table.

---

## Step 2 — Run Breakout Queries for Shared Tables

For shared tables — AzureDiagnostics, CommonSecurityLog, Syslog, ThreatIntelIndicators, and ASIM tables — you need to know what is actually writing to that table in this specific environment before the AI can produce accurate output.

**Why this step matters:**
The AI writes functions based on detection KQL. But the detection may only filter on one sub-source within a shared table. The environment may have other sub-sources that no detection currently covers but that still need to be tracked. Without running the breakout query you would miss those sources entirely.

**How to do it:**
For each shared table in your table-first reference run the appropriate breakout query from `resources/scratch.md`. Those queries are organized by table — AzureDiagnostics Level 1 and Level 2, CommonSecurityLog Level 1 and Level 2, Syslog, ThreatIntelIndicators, and all ASIM tables.

Note what you find. You will include this information when you feed the table to the AI in Step 4.

**For single source tables** — SignInLogs, AuditLogs, SecurityEvent, OfficeActivity and all the Device tables — skip this step. There is only one source writing to these tables. The detection KQL is sufficient context for the AI.

---

## Step 3 — Train the AI

Paste this entire prompt into the AI at the start of every session. Wait for it to confirm it understands. Do this before feeding any table data.

---

**TRAINING PROMPT:**

```
I am building a data source tracking system for Microsoft Sentinel.
I will provide you with information about one table at a time. For each
table I will give you:
- The table name
- All detections that query this table with their full KQL
- For shared tables — what the live environment breakout query shows
  is actually writing to this table

You will analyze everything together and produce two types of output:

═══════════════════════════════════════
OUTPUT TYPE 1 — WATCHLIST ROWS
═══════════════════════════════════════

Produce one watchlist row block per unique log source or data source
that the detections depend on within this table.

Important: Think table-first not detection-first. Multiple detections
may depend on the same source. Produce one row per unique source —
not one row per detection. If three detections all query Azure Firewall
data within AzureDiagnostics that is one source row — not three.

Use this exact format for each row:

---WATCHLIST ROW---
TABLE: [exact table name]
LOGSOURCE: [plain English name of this specific source within the table]
CATEGORY: [Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other]
ORIGIN: [Microsoft Entra ID / Microsoft 365 / Microsoft Defender / Microsoft Azure / Microsoft Sentinel / Windows OS / Network Device / AWS / Salesforce / Zoom / Keeper Security / SecureW2 / Drupal / Multiple Sources / Custom / Unknown]
TRANSPORT: [Microsoft Connector / XDR Connector / AMA — DCR / CEF — AMA — DCR / Diagnostic Setting / Logic App — Polling / REST API — Push / TAXII — Polling / Sentinel Native / ASIM Parser / other if none fit]
DESCRIPTION: [2-3 sentences describing what this specific data source is. Be specific and natural — use both the table name context and what the detections are doing with the data. Do not be generic.]
PURPOSE: [one sentence — what attack or risk does this specific data help detect based on what the detections are doing with it]
SLA: [True if this is a primary security data source a client would expect to be monitored — False if it is supporting or operational]
DATACONNECTOR: [name of the Sentinel data connector most likely used — make your best educated guess]
DCRNAME: [write Unknown — engineer will fill in from environment]
DCENAME: [write Unknown — engineer will fill in from environment]
FUNCTIONNAME: [for shared tables write Get[TableName]Health — for single source tables write None]
SLS: [write Missing — engineer will assign silent detection later]
MONITORINGFREQUENCY: [1h for continuous / 5h for near-continuous / 15h for business-hours active / 24h for daily batch / 48h for less frequent / None if not appropriate]
CONFIDENCE: [High / Medium / Low — how confident are you in MonitoringFrequency and why]
NOTES: [anything an engineer should know — common misconfigurations, gotchas, things to verify]
---END ROW---

═══════════════════════════════════════
OUTPUT TYPE 2 — KQL SUB-FUNCTION
═══════════════════════════════════════

After all watchlist rows write a KQL sub-function IF this is a shared
table — meaning multiple distinct sources write to it and are
identified by filtering on a specific field.

Shared tables:
- AzureDiagnostics — filtered by ResourceType and Category
- CommonSecurityLog — filtered by DeviceVendor and DeviceProduct
- Syslog — filtered by HostName
- ThreatIntelIndicators — filtered by SourceSystem
- ASimNetworkSessionLogs, ASimAuditEventLogs, ASimWebSessionLogs
  — filtered by EventVendor and EventProduct

Do NOT write a sub-function for single source tables — SignInLogs,
AuditLogs, SecurityEvent, OfficeActivity, Device* tables etc.
The master detection handles these directly.

Every sub-function MUST return exactly these four columns:
Table, LogSource, LastSeen, Status

Use this exact structure:

---FUNCTION---
// Get[TableName]Health()
// Sub-function for [TableName]
// Returns one row per expected sub-source
// Output contract: Table, LogSource, LastSeen, Status
union
(
    [TableName]
    | where TimeGenerated > ago(24h)
    | where [FilterField] == "[FilterValue]"
    | summarize LastSeen = max(TimeGenerated)
    | extend Table = "[TableName]"
    | extend LogSource = "[Plain English Source Name matching watchlist row]"
),
(
    // one block per additional sub-source
)
| extend DaysSince = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSince <= 1, "Active",
    DaysSince <= 7, "Review",
    DaysSince > 7,  "Inactive",
    "No Data"
)
| project Table, LogSource, LastSeen, Status
---END FUNCTION---

═══════════════════════════════════════
IMPORTANT RULES
═══════════════════════════════════════

1. One watchlist row per unique source — not one per detection.
   Multiple detections sharing the same source = one row.

2. The LogSource value in the watchlist row and in the function
   extend statement must match exactly — this is the join key.

3. If a detection uses search * or union * with no table filter:
   LOGSOURCE: All Tables — workspace wide detection
   FUNCTIONNAME: None
   Do not write a sub-function.

4. If the breakout query shows sources that no detection currently
   covers — still include them in the watchlist row and function.
   They may need monitoring even without a current detection.

5. Use table name AND detection KQL AND breakout results together
   to infer description and purpose. Do not rely on just one.

6. Write descriptions naturally — explaining to a fellow engineer
   what they are looking at, not filling in a form.

7. If uncertain about any field say so in NOTES rather than
   guessing silently.

Confirm you understand both output types and all rules before
I provide any data.
```

---

Wait for confirmation before proceeding.

---

## Step 4 — Work Through Tables One by One

Use your table-first reference from Step 1 as your guide. Work through each table in order.

**For each table:**

1. Open your text editor
2. Write the table name at the top
3. For each detection listed under this table add:
   - Detection name
   - Full KQL query
   - Blank line between detections
4. For shared tables add a section showing what the breakout query found:
   ```
   ENVIRONMENT BREAKOUT RESULTS:
   ResourceType = AZUREFIREWALLS — Categories: ApplicationRule, NetworkRule, DnsProxy, ThreatIntelLog
   ResourceType = VAULTS — Categories: AuditEvent
   ```
5. Copy everything and paste to the AI

**Input format example:**
```
AzureDiagnostics

OC12345 - Azure Firewall Threat Intel Match
AzureDiagnostics
| where ResourceType == "AZUREFIREWALLS"
| where Category == "AzureFirewallThreatIntelLog"
| where action_s == "Deny"

OC12346 - Key Vault Secret Access Anomaly
AzureDiagnostics
| where ResourceType == "VAULTS"
| where Category == "AuditEvent"
| where OperationName == "SecretGet"

ENVIRONMENT BREAKOUT RESULTS:
ResourceType = AZUREFIREWALLS — Categories seen: ApplicationRule, NetworkRule, DnsProxy, ThreatIntelLog
ResourceType = VAULTS — Categories seen: AuditEvent
```

**Why include all detections for a table together:**
The AI sees the full picture of what data is being used for and can identify distinct sub-sources accurately. It also avoids producing duplicate rows — if two detections both query Azure Firewall the AI sees both and produces one row, not two.

**Why include the breakout results:**
The detections may only cover some of the sub-sources present in the environment. The breakout results tell the AI what else is there. For example the detections might only reference AzureFirewallThreatIntelLog but the breakout shows ApplicationRule and NetworkRule are also present. The AI can then produce rows for all of them — not just the ones detections currently cover.

---

## Step 5 — Transfer Watchlist Rows to Spreadsheet

After receiving output for each table transfer the watchlist rows to your sources spreadsheet immediately. Do not let output accumulate — transfer as you go.

Map AI output fields to spreadsheet columns:

| AI Output Field | Spreadsheet Column |
|---|---|
| TABLE | Table |
| LOGSOURCE | LogSource |
| CATEGORY | Category |
| ORIGIN | Origin |
| TRANSPORT | Transport |
| DESCRIPTION | Description |
| PURPOSE | Purpose |
| SLA | SLA — review and confirm |
| DATACONNECTOR | DataConnector |
| DCRNAME | DCRName — fill in from environment |
| DCENAME | DCEName — fill in from environment |
| FUNCTIONNAME | FunctionName |
| SLS | SLS — fill in when detection is created |
| MONITORINGFREQUENCY | MonitoringFrequency — review CONFIDENCE |
| NOTES | Notes — include CONFIDENCE reasoning |

**Fields to always review before accepting:**
- **MonitoringFrequency** — AI makes an educated guess. If you know the actual ingestion pattern override it. Low CONFIDENCE means verify.
- **SLA** — AI guesses based on source nature. You decide based on service commitment.
- **DataConnector** — verify against what is actually configured in this environment.

---

## Step 6 — Save KQL Functions

Paste each sub-function into `functions-in-progress.kql` with a comment header:

```kql
// ============================================================
// GetAzureDiagnosticsHealth()
// Generated: [date]
// Verify filter conditions against actual environment
// ============================================================
[paste function here]


// ============================================================
// GetCommonSecurityLogHealth()
// Generated: [date]
// ============================================================
[paste function here]
```

**Important:** The AI writes functions based on detection KQL and breakout results. After completing all tables review each function and verify the filter conditions match what is actually in the environment. Use the breakout queries to confirm.

---

## Step 7 — Handle Tables Without Detections

Some sources need to be in the watchlist even though no current detection uses them. These come from:

- Client compliance requirements — log types mandated by PCI-DSS, HIPAA etc
- Capability dependencies — UEBA, Threat Intelligence, Fusion ML
- Baseline requirements — data that should be collected as good practice
- Client-specific requests from the intake questionnaire

For these use this prompt:

```
I need to document a data source in Microsoft Sentinel that does not
currently have a custom detection but is being tracked for other reasons.

Table: [table name]
Reason for tracking: [compliance / UEBA dependency / client request / baseline]
Any context: [anything you know about this source]

Please produce a watchlist row block using the same format as before.
For PURPOSE base it on what this data source is generally used for
in security monitoring even if no specific detection currently uses it.
Do not write a sub-function for this request unless it is a shared
table with multiple sources.
```

---

## Step 8 — Handle Pending Client Input

Some sources will only be known after the client returns their intake questionnaire. Mark these rows:

```
Notes: [Pending - client input]
```

Return to them after the questionnaire is received.

---

## Step 9 — Final Review Before Upload

Before uploading check every row:

- [ ] Table matches exactly what appears in `[mssname]-tables` watchlist
- [ ] LogSource is plain English and descriptive
- [ ] Category is from the approved list
- [ ] Origin is from the approved list
- [ ] Transport is from the approved list
- [ ] Description is 2-3 sentences — specific not generic
- [ ] Purpose is one sentence written for a business audience
- [ ] SLA is True or False — nothing blank
- [ ] MonitoringFrequency is set — nothing blank
- [ ] No [Manual] or Unknown placeholders in required fields
- [ ] DCRName and DCEName filled in from environment or N/A
- [ ] FunctionName matches the actual function name you will deploy

---

## Step 10 — Review and Finalize Functions

Before deploying functions to Sentinel review each one:

- [ ] Function name follows the convention: Get[TableName]Health()
- [ ] Returns exactly four columns: Table, LogSource, LastSeen, Status
- [ ] Filter conditions verified against live environment
- [ ] All expected sub-sources included — add any missing ones
- [ ] LogSource values in function match LogSource values in sources watchlist exactly

**Verify filter conditions against live data:**
```kql
// Example for AzureDiagnostics
AzureDiagnostics
| where TimeGenerated > ago(30d)
| summarize Count = count() by ResourceType, Category
| order by Count desc
```

Add any combinations found here that are not already in the function.

---

## Step 11 — Upload Watchlist and Deploy Functions

**Watchlist upload:**
1. Export spreadsheet as CSV
2. Sentinel → Watchlists → Create new
3. Name: `[mssname]-sources`
4. Upload CSV
5. Set SearchKey to `Table`
6. Validate:
```kql
_GetWatchlist('[mssname]-sources')
| summarize TotalRows = count(), Tables = dcount(Table)
```

**Function deployment:**
1. In Sentinel navigate to Logs
2. Run each function KQL to verify it executes without errors
3. Click Save → Save as function
4. Name exactly as defined — `GetAzureDiagnosticsHealth` etc
5. Save final KQL files to `01-Program/Functions/` in SharePoint

---

## Tips for Best Output

**Include breakout results for shared tables.** This is the most important tip. Without it the AI only knows what detections currently cover — not what else is in the environment.

**Include all detections for a table together.** The AI sees the complete picture and avoids redundant rows.

**One table at a time.** Do not batch multiple tables. Output stays clean and traceable.

**Trust your knowledge over AI for MonitoringFrequency.** If you know a source sends data hourly set 1h regardless of AI suggestion. Use CONFIDENCE as a guide — Low means verify.

**Review the function filter conditions against live data.** Always verify before deploying.

---

*This document lives in 06-watchlist-management and is maintained in the sentinel-mssp-playbook repository. Update when the process changes, the prompt is refined, or new field types are added.*