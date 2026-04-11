# Building the Sources Watchlist — Process and AI Guidance

> **Purpose:** This document guides an engineer through the process of building the `[mssname]-sources` watchlist and the associated KQL sub-functions using detection data and AI assistance. It includes the exact prompt to use with any LLM, the input format, the output format, and where everything gets saved.
>
> **Who uses this:** Any engineer building or updating the sources watchlist. Works with any capable LLM including internal company tools — no dependency on any specific AI system.
>
> **What you produce:**
> 1. A completed sources watchlist spreadsheet ready for upload to Sentinel
> 2. KQL sub-functions for all shared tables ready for deployment
>
> **Where outputs go:**
> - Completed spreadsheet → `02-Clients/[ClientName]/04-Watchlists/Current/`
> - KQL functions → `02-Clients/[ClientName]/06-Functions/` and `01-Program/Functions/` if generic
> - Master prompts → `01-Program/AI-Prompts/sources-watchlist-prompt.md`
>
> **Last Updated:** April 2026

---

## Goal and Why This Matters

The sources watchlist is the most important watchlist in the entire system. It is where all monitoring lives. Every table gets at least one row here. Single source tables get one row describing the whole table as a source. Shared tables — like AzureDiagnostics, CommonSecurityLog, Syslog — get one row per distinct sub-source.

This watchlist drives:
- The master source silent detection — reads MonitoringFrequency as a variable threshold
- The workbook data sources tab — shows health per source
- The audit deck data sources section — shows what we are collecting and its status
- The KQL sub-functions — built from the same source analysis

Getting this right is the foundation for everything above it. Take the time to do it properly.

---

## Standards and Taxonomy

Before starting understand the approved values for every field. The AI will use these — knowing them helps you verify its output.

**Category — choose one:**
Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other

**Origin — choose one:**
Microsoft Entra ID / Microsoft 365 / Microsoft Defender / Microsoft Azure / Microsoft Sentinel / Windows OS / Network Device / AWS / Salesforce / Zoom / Keeper Security / SecureW2 / Drupal / Multiple Sources / Custom / Unknown

**Transport — choose one:**
Microsoft Connector / XDR Connector / AMA — DCR / AMA — DCR — DCE / CEF — AMA — DCR / Cribl — AMA — DCR / Cribl — DCR / Diagnostic Setting / Logic App — Polling / Logic App — Webhook / REST API — Push / TAXII — Polling / Sentinel Native / ASIM Parser / Application Insights SDK

**SLA:** True / False

**MonitoringFrequency:** None / 1h / 5h / 15h / 24h / 48h

**SLS:** AB##### — the silent detection assigned to this source. Write Missing if none exists yet.

---

## Prerequisites

Before starting have these ready and open simultaneously:

1. **DetectionCatalog spreadsheet** — `[mssname]-detections`. This is your source list. Work through it table by table.

2. **Sources watchlist spreadsheet** — open with correct column headers in this exact order:
```
Table, LogSource, Category, Origin, Transport, Description,
Purpose, SLA, DataConnector, DCRName, DCEName, FunctionName,
SLS, MonitoringFrequency, Notes
```

3. **Functions document** — a new file called `functions-in-progress.kql` open in your client's 06-Functions folder in SharePoint. This is where you paste KQL sub-functions as they are generated.

4. **Text editor** — staging area for detection input before sending to the AI.

5. **A capable LLM** — use your internal company LLM for anything involving client-specific information.

---

## Step 1 — Train the AI

Paste this entire prompt into the AI at the start of every session. Wait for it to confirm it understands before proceeding. If it does not confirm clearly paste the prompt again.

---

**TRAINING PROMPT:**

```
I am building a data source tracking system for Microsoft Sentinel.
I will provide you with analytics rule (detection) information grouped
by table. For each table I provide you with the table name followed by
one or more detection names and their KQL queries.

You will analyze the table name and all detection KQL queries together
and produce two types of output:

═══════════════════════════════════════
OUTPUT TYPE 1 — WATCHLIST ROWS
═══════════════════════════════════════

Produce one watchlist row block per unique log source or data source
category that the detections depend on within that table.

Use this exact format for each row:

---WATCHLIST ROW---
TABLE: [exact table name as provided]
LOGSOURCE: [plain English name of this specific source within the table]
CATEGORY: [Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other]
ORIGIN: [Microsoft Entra ID / Microsoft 365 / Microsoft Defender / Microsoft Azure / Microsoft Sentinel / Windows OS / Network Device / AWS / Salesforce / Zoom / Keeper Security / SecureW2 / Drupal / Multiple Sources / Custom / Unknown]
TRANSPORT: [Microsoft Connector / XDR Connector / AMA — DCR / CEF — AMA — DCR / Diagnostic Setting / Logic App — Polling / REST API — Push / TAXII — Polling / Sentinel Native / ASIM Parser / Application Insights SDK / other if none fit]
DESCRIPTION: [2-3 sentences describing what this specific data source is. Be specific and natural — use both the table name context and what the detections are doing with the data. Do not be generic or robotic.]
PURPOSE: [one sentence — what attack or risk does this specific data help detect based on what the detections are doing with it]
SLA: [True if this is a primary security data source a client would expect to be monitored — False if it is supporting or operational data]
DATACONNECTOR: [name of the Sentinel data connector most likely used to configure this source — make your best educated guess based on table and origin]
DCRNAME: [write Unknown — engineer will fill in from environment]
DCENAME: [write Unknown — engineer will fill in from environment]
FUNCTIONNAME: [for shared tables write Get[TableName]Health — for single source tables write None]
SLS: [write Missing — engineer will assign silent detection later]
MONITORINGFREQUENCY: [your best estimate: 1h for continuous real-time sources / 5h for near-continuous / 15h for business-hours active sources / 24h for daily batch sources / 48h for less frequent sources / None if monitoring not appropriate]
CONFIDENCE: [High / Medium / Low — how confident are you in the MonitoringFrequency estimate and why]
NOTES: [anything an engineer should know — common misconfigurations, gotchas, things to verify, why you chose the MonitoringFrequency you did]
---END ROW---

═══════════════════════════════════════
OUTPUT TYPE 2 — KQL SUB-FUNCTION
═══════════════════════════════════════

After all watchlist rows for a table write a KQL sub-function IF the
table is a shared table — meaning multiple distinct sources write to
it and are identified by filtering on a specific field.

Examples of shared tables:
- AzureDiagnostics — filtered by ResourceType and Category
- CommonSecurityLog — filtered by DeviceVendor and DeviceProduct
- Syslog — filtered by HostName
- ThreatIntelIndicators — filtered by SourceSystem

Do NOT write a sub-function for single source tables like SignInLogs,
AuditLogs, SecurityEvent, OfficeActivity — the master detection
handles these directly.

Every sub-function MUST return exactly these four columns and no others:
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
    | extend LogSource = "[Plain English Source Name]"
),
(
    // additional sub-source blocks — one per identified source
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

1. If multiple detections use the same table but filter on different
   sub-sources produce one watchlist row per sub-source — they are
   different log sources even if they share a table

2. If a detection uses search * or union * with no specific table
   filter write:
   LOGSOURCE: All Tables — workspace wide detection
   FUNCTIONNAME: None
   And do not write a sub-function

3. Use both the table name AND the detection KQL together to infer
   purpose and description — do not rely on just one

4. Write descriptions naturally as if explaining to a fellow engineer
   what they are looking at — not filling in a form

5. If you are uncertain about any field say so in NOTES rather than
   guessing silently

6. The CONFIDENCE field for MonitoringFrequency is important — explain
   your reasoning briefly so the engineer can make an informed decision
   about whether to override your suggestion

Confirm you understand both output types before I provide any data.
```

---

Wait for confirmation before proceeding.

---

## Step 2 — Prepare Your Input

Group your detections by table from the DetectionCatalog. For each table:

1. Open your text editor
2. Write the table name at the very top on its own line
3. For each detection that uses this table add:
   - Detection name on its own line
   - Full KQL query directly below it
   - Blank line between detections
4. Include ALL detections for the same table — only write the table name once

**Input format:**
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
| where ResultType != "Success"
```

**Why include all detections for a table:**
Two detections may use the same table but filter on completely different sub-sources. Sending both together lets the AI identify that AzureDiagnostics has two distinct sources — Azure Firewall and Key Vault — each needing its own watchlist row and its own block in the sub-function.

---

## Step 3 — Send to AI and Collect Output

Copy your text editor content and paste it into the AI after the training prompt. Work one table at a time.

The AI will produce:
- One watchlist row block per sub-source identified
- One KQL sub-function if the table is a shared table

After receiving output for one table:
1. Transfer watchlist rows to the sources spreadsheet immediately
2. Paste the KQL function into `functions-in-progress.kql` immediately
3. Clear the text editor and move to the next table

Do not let output pile up — transfer as you go.

---

## Step 4 — Transfer Watchlist Rows to Spreadsheet

For each watchlist row block map the AI fields to spreadsheet columns:

| AI Output Field | Spreadsheet Column |
|---|---|
| TABLE | Table |
| LOGSOURCE | LogSource |
| CATEGORY | Category |
| ORIGIN | Origin |
| TRANSPORT | Transport |
| DESCRIPTION | Description |
| PURPOSE | Purpose |
| SLA | SLA |
| DATACONNECTOR | DataConnector |
| DCRNAME | DCRName — fill in from environment |
| DCENAME | DCEName — fill in from environment |
| FUNCTIONNAME | FunctionName |
| SLS | SLS — fill in when detection is created |
| MONITORINGFREQUENCY | MonitoringFrequency — review AI suggestion |
| NOTES | Notes — add CONFIDENCE reasoning here |

**Fields to always review before accepting:**
- **MonitoringFrequency** — the AI makes an educated guess. If you know the actual ingestion pattern override it. Check CONFIDENCE rating — Low means verify before accepting.
- **SLA** — the AI guesses based on the nature of the source. You decide based on your service commitment to the client.
- **DataConnector** — verify against what is actually configured in this environment.

---

## Step 5 — Save KQL Functions

As you receive sub-functions paste them into `functions-in-progress.kql` with a comment header:

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

**Important:** The AI writes functions based on detection KQL. It may not know every sub-source in the environment — only the ones detections currently cover. After completing all tables review the functions and add any additional sub-sources you know exist that no detection currently covers.

---

## Step 6 — Handle Tables Without Detections

Some tables need to be in the sources watchlist even though no current detection uses them. These come from:

- Client compliance requirements — specific log types mandated by PCI-DSS, HIPAA etc
- Capability dependencies — UEBA, Threat Intelligence, Fusion ML
- Client-specific requests from the intake questionnaire
- Baseline requirements — data that should be collected as part of good practice

For these use this targeted prompt after your main session:

```
I need to document a data source in Microsoft Sentinel that does not
currently have a custom detection but is being tracked for other reasons.

Table: [table name]
Reason for tracking: [compliance / UEBA dependency / client request / baseline]
Any context you have: [anything you know about this source]

Please produce a watchlist row block using the same format as before.
For PURPOSE — base it on what this data source is generally used for
in security monitoring even if no specific detection currently uses it.
Do not write a sub-function for this request.
```

---

## Step 7 — Sources Without Client Input Yet

Some sources will only be known after the client returns their intake questionnaire. These come from:
- Client-specific compliance obligations
- Client-specific integrations — custom _CL tables
- Business-critical systems the client identifies

Mark these rows in the spreadsheet with `[Pending - client input]` in the Notes column. Return to them after the questionnaire is received.

---

## Step 8 — Final Review Before Upload

Before uploading as a watchlist check every row:

- [ ] Table matches exactly what appears in `[mssname]-tables` watchlist
- [ ] LogSource is plain English and descriptive
- [ ] Category is from the approved list
- [ ] Origin is from the approved list
- [ ] Transport is from the approved list
- [ ] Description is 2-3 sentences — specific not generic
- [ ] Purpose is one sentence written for a business audience
- [ ] SLA is True or False — nothing blank
- [ ] MonitoringFrequency is set — not blank
- [ ] No [Manual] or Unknown placeholders in required fields
- [ ] DCRName and DCEName filled in from environment or N/A
- [ ] FunctionName matches the actual function name you will deploy

---

## Step 9 — Review and Finalize Functions

Before deploying functions to Sentinel review each one:

- [ ] Function name follows the convention: Get[TableName]Health()
- [ ] Returns exactly four columns: Table, LogSource, LastSeen, Status
- [ ] Filter conditions match what is actually in the environment — verify against live data
- [ ] All expected sub-sources are included — add any missing ones
- [ ] LogSource values in the function match LogSource values in the sources watchlist exactly — these must match for the join to work

**Verify filter conditions with this query per shared table:**
```kql
// Example for AzureDiagnostics — verify what is actually present
AzureDiagnostics
| where TimeGenerated > ago(30d)
| summarize Count = count() by ResourceType, Category
| order by Count desc
```

Add any ResourceType/Category combinations found here that are not already in the function.

---

## Step 10 — Upload Watchlist and Deploy Functions

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
2. Run each function KQL
3. Click Save → Save as function
4. Name it exactly as defined — `GetAzureDiagnosticsHealth` etc
5. Save the final KQL files to `01-Program/Functions/` in SharePoint if generic enough to reuse across clients

---

## Tips for Best AI Output

**Be specific about the environment when you can.** Tell the AI — "this is a financial services client" or "the firewall is a Palo Alto PA-3200" — it will produce more relevant descriptions and purpose statements.

**Include all detections for a table.** The more detection context the AI has the better it identifies distinct sub-sources and infers purpose accurately.

**Trust your own knowledge over the AI for MonitoringFrequency.** The AI makes an educated guess. If you know a source sends data hourly set it to 1h regardless of what the AI suggests. Use the CONFIDENCE field as a guide — Low means verify.

**One table at a time.** Resist batching multiple tables. Output stays clean and traceable and it is easier to transfer to the spreadsheet as you go.

**Review the function filter conditions against live data.** The AI writes functions based on detection KQL. The environment may have additional sub-sources that no detection currently covers. Always verify with a live query before deploying.

---

*This document lives in 01-Program/Processes/06-sources-watchlist-build.md in SharePoint and is maintained in the sentinel-mssp-playbook repository. Update when the process changes, the prompt is refined, or new field types are added.*