# Building the Sources Watchlist — Process and AI Guidance

> **Purpose:** This document guides an engineer through the complete process of building the `[mssname]-sources` watchlist and the associated KQL sub-functions. It covers the full workflow from organizing detection data through to uploading a validated watchlist and deploying functions to Sentinel.
>
> **Who uses this:** Any engineer building or updating the sources watchlist. Works with any capable LLM including internal company tools.
>
> **What you produce:**
> 1. A sources investigation spreadsheet — your working reference during the process
> 2. A completed `[mssname]-sources` watchlist spreadsheet ready for upload
> 3. KQL sub-functions for all shared tables ready for deployment
>
> **Where outputs go:**
> - Investigation spreadsheet → `02-Clients/[ClientName]/04-Watchlists/Current/`
> - Sources watchlist spreadsheet → `02-Clients/[ClientName]/04-Watchlists/Current/`
> - KQL functions → `02-Clients/[ClientName]/06-Functions/` and `01-Program/Functions/` if generic
>
> **Prerequisites:** Complete the data source audit runbook and upload `[mssname]-tables` before starting. You also need your completed `[mssname]-detections` spreadsheet — this is the starting point for everything in this process.
>
> **Last Updated:** April 2026

---

## The Two Spreadsheets — Understanding the Relationship

Before starting understand how the two spreadsheets relate to each other. They serve different purposes and feed different watchlists.

**`[mssname]-detections` spreadsheet — already built:**
- One row per detection
- Fields: RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description
- Becomes the `[mssname]-detections` watchlist in Sentinel
- Used by GetEnrichedInventory() to show detection coverage per source
- Owned by detection team
- Your starting point for this process — do not modify it

**Sources investigation spreadsheet — what you build here:**
- One row per log source
- Built by feeding the detections spreadsheet to AI
- Your working reference during investigation
- Becomes the `[mssname]-sources` watchlist after investigation columns are removed
- Owned by platform team

**The relationship:**
```
[mssname]-detections spreadsheet
        ↓ feed to AI — reorganize by table
Sources investigation spreadsheet
        ↓ investigate, verify, complete
        ↓ remove investigation columns
[mssname]-sources watchlist
        ↓ GetEnrichedInventory() joins both watchlists
Workbook shows detection coverage per source automatically
```

---

## Why This Watchlist Exists and Why It Is Built This Way

The sources watchlist is the most important watchlist in the system. Everything that makes the workbook valuable — health monitoring per source, detection coverage per source, silent detection coverage, the audit deck data sources tab — all of it reads from this watchlist.

Every table gets at least one row here. Single source tables get one row. Shared tables get one row per distinct sub-source.

**Why we build it from detections:**
We start from what we know for certain — our detections. Every detection queries specific tables and often filters on specific sub-sources. If a detection depends on a source that source is important. If it goes silent the detection goes blind. That is the clearest possible justification for tracking a source.

**Why we think table-first not detection-first:**
Multiple detections often query the same table and the same sub-source. Going detection by detection creates redundant rows and functions. The correct approach is:

```
Table
  ↓
What unique sources write to this table
  ↓
What detections depend on each source
  ↓
One watchlist row per unique source
One function per unique source
Multiple detections share the same source and function
```

---

## Standards and Taxonomy

Approved values for every field. The AI uses these — know them so you can verify AI output.

**Category:** Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other

**Origin:** Microsoft Entra ID / Microsoft 365 / Microsoft Defender / Microsoft Azure / Microsoft Sentinel / Windows OS / Network Device / AWS / Salesforce / Zoom / Keeper Security / SecureW2 / Drupal / Multiple Sources / Custom / Unknown

**Transport:** Microsoft Connector / XDR Connector / AMA — DCR / AMA — DCR — DCE / CEF — AMA — DCR / Cribl — AMA — DCR / Cribl — DCR / Diagnostic Setting / Logic App — Polling / Logic App — Webhook / REST API — Push / TAXII — Polling / Sentinel Native / ASIM Parser / Application Insights SDK

**SLA:** True / False

**MonitoringFrequency:** None / 1h / 5h / 15h / 24h / 48h

**SLS:** OC##### or Missing

**Tier:** 1 / 2 / 3 / 4 — see silent-detection-standards.md for the decision tree

---

## Prerequisites — Have These Ready

1. **`[mssname]-detections` spreadsheet** — your existing detections catalog
2. **Sources investigation spreadsheet** — new, open with headers listed below
3. **Functions document** — new file called `functions-in-progress.kql` in `02-Clients/[ClientName]/06-Functions/`
4. **Text editor** — staging area before sending to AI
5. **A capable LLM** — use your internal company LLM for client-specific information

**Sources investigation spreadsheet headers — in this exact order:**
```
Table, LogSource, Category, Origin, Transport, Description,
Purpose, SLA, DataConnector, DCRName, DCEName, FunctionName,
SLS, MonitoringFrequency, Notes, Tier, Status, Verified, ActionItems
```

Save this immediately as:
```
[ClientName]-sources-investigation-[YYYY-MM-DD].xlsx
```

Format as an Excel Table with Ctrl+T before adding any data.

---

## The Complete Workflow

```
Step 1  — Reorganize detections by table using AI
Step 2  — Build the pivot table working reference
Step 3  — Run breakout queries for shared tables
Step 4  — Train the AI with the full prompt
Step 5  — Work through tables one by one
Step 6  — Transfer CSV output to investigation spreadsheet
Step 7  — Save KQL functions as you go
Step 8  — Handle tables without detections
Step 9  — Handle pending client input
Step 10 — Assign tiers and complete investigation fields
Step 11 — Final review before upload
Step 12 — Review and finalize functions
Step 13 — Create the upload-ready watchlist
Step 14 — Upload watchlist and deploy functions
```

---

## Step 1 — Reorganize Detections by Table

Your detections spreadsheet has comma separated table names per detection. A detection querying three tables appears once but belongs under all three. You need a table-first view before you can work systematically.

**Feed the entire detections CSV to AI with this prompt:**

```
I have a detection catalog with these columns:
RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description

The Table field contains comma separated table names — one detection
may query multiple tables.

Please reorganize this into a table-first view showing which
detections depend on each table.

Output as a simple two column list:

Table | Detections
AzureDiagnostics | OC00034 - Detection Name, OC00041 - Detection Name
SignInLogs | OC00012 - Detection Name, OC00019 - Detection Name

Rules:
- If a detection queries multiple tables list it under each table
- Group all detections for the same table on one row comma separated
- Include RuleId and rule name for each detection
- Sort tables alphabetically
- If a detection uses Table = All list it separately at the bottom
  under a heading called: All Tables — Workspace Wide Detections
```

Save this output — it is your working map for the entire process.

---

## Step 2 — Build the Pivot Table Working Reference

Take the two-column table-first output from Step 1 and paste it into a new Excel sheet. Format as a table with Ctrl+T. This is your visual reference as you work through each table.

You now have two views open:
- **Detections spreadsheet** — the detail per detection
- **Pivot table** — the table-first view showing which tables need to be processed

Work through the pivot table top to bottom. One table at a time.

---

## Step 3 — Run Breakout Queries for Shared Tables

For shared tables — AzureDiagnostics, CommonSecurityLog, Syslog, ThreatIntelIndicators, and ASIM tables — run the breakout queries from `resources/scratch.md` against the live environment before feeding anything to the AI.

**Why this matters:**
The AI writes based on detection KQL. But detections may only cover some sub-sources within a shared table. The environment may have additional sub-sources that no detection covers. Without the breakout query you would miss those sources entirely.

Note what you find. You will include this when feeding the table to the AI in Step 5.

For single source tables — SignInLogs, AuditLogs, SecurityEvent, all Device tables — skip this step.

---

## Step 4 — Train the AI

Paste this entire prompt at the start of every session. Wait for confirmation before proceeding.

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

You will analyze everything together and produce two types of output.

═══════════════════════════════════════
OUTPUT TYPE 1 — CSV ROWS
═══════════════════════════════════════

Produce one CSV row per unique log source the detections depend on.

Important: Think table-first not detection-first. Multiple detections
may depend on the same source. One row per unique source — not one
per detection. If three detections all query Azure Firewall data
within AzureDiagnostics that is one row — not three.

Output as CSV with these exact column headers in this exact order:
Table,LogSource,Category,Origin,Transport,Description,Purpose,SLA,DataConnector,DCRName,DCEName,FunctionName,SLS,MonitoringFrequency,Notes,Tier,Status,Verified,ActionItems

Rules for each field:

Table: exact table name as provided
LogSource: plain English name of this specific source within the table
Category: Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other
Origin: Microsoft Entra ID / Microsoft 365 / Microsoft Defender / Microsoft Azure / Microsoft Sentinel / Windows OS / Network Device / AWS / Salesforce / Zoom / Keeper Security / SecureW2 / Drupal / Multiple Sources / Custom / Unknown
Transport: Microsoft Connector / XDR Connector / AMA — DCR / CEF — AMA — DCR / Diagnostic Setting / Logic App — Polling / REST API — Push / TAXII — Polling / Sentinel Native / ASIM Parser / other if none fit
Description: 2-3 sentences describing what this source is. Be specific — use table name context and detection logic together. Not generic.
Purpose: one sentence — what attack or risk does this help detect based on what detections are doing with it
SLA: True if primary security source a client would expect monitored — False if supporting or operational
DataConnector: best guess at the Sentinel connector name based on table and origin
DCRName: Unknown
DCEName: Unknown
FunctionName: Get[TableName]Health for shared tables — None for single source tables
SLS: Missing
MonitoringFrequency: 1h for continuous identity/pipeline sources / 5h for network and firewall / 15h for endpoint / 24h for daily batch / 48h for less frequent / None if not appropriate
Notes: anything an engineer should know — misconfigurations, gotchas, things to verify. Include your reasoning for MonitoringFrequency choice.
Tier: 1 if an OC detection depends on this source or it feeds a capability — 2 if high investigation value or custom integration — 3 if ASIM or capability output — 4 if no dependencies
Status: [leave blank — engineer fills in from live environment]
Verified: No
ActionItems: [leave blank — engineer fills in]

Output the header row first then one data row per unique source.
No explanation before or after. Just the CSV.

═══════════════════════════════════════
OUTPUT TYPE 2 — KQL SUB-FUNCTION
═══════════════════════════════════════

After the CSV rows write a KQL sub-function IF this is a shared table.

Shared tables:
- AzureDiagnostics — filtered by ResourceType and Category
- CommonSecurityLog — filtered by DeviceVendor and DeviceProduct
- Syslog — filtered by HostName
- ThreatIntelIndicators — filtered by SourceSystem
- ASimNetworkSessionLogs, ASimAuditEventLogs, ASimWebSessionLogs
  — filtered by EventVendor and EventProduct

Do NOT write a sub-function for single source tables.

Every sub-function MUST return exactly these four columns:
Table, LogSource, LastSeen, Status

The LogSource value in the function must match the LogSource value
in the CSV rows exactly — this is the join key.

Use this exact structure:

---FUNCTION---
// Get[TableName]Health()
// Sub-function for [TableName]
// Output contract: Table, LogSource, LastSeen, Status
union
(
    [TableName]
    | where TimeGenerated > ago(24h)
    | where [FilterField] == "[FilterValue]"
    | summarize LastSeen = max(TimeGenerated)
    | extend Table = "[TableName]"
    | extend LogSource = "[LogSource value from CSV row]"
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

1. One CSV row per unique source — not one per detection

2. LogSource in CSV and in function extend must match exactly

3. If a detection uses search * or union * with no table filter:
   LogSource: All Tables — workspace wide detection
   FunctionName: None
   Do not write a sub-function

4. If breakout results show sources no detection currently covers
   still include them in the CSV and function

5. Use table name AND detection KQL AND breakout results together

6. Write descriptions naturally — not filling in a form

Confirm you understand before I provide any data.
```

---

## Step 5 — Work Through Tables One by One

Use your pivot table from Step 2 as your guide. For each table:

1. Open your text editor
2. Write the table name at the top
3. For each detection listed under this table add the detection name and full KQL
4. For shared tables add the breakout results below the detections
5. Copy everything and paste to the AI

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

ENVIRONMENT BREAKOUT RESULTS:
ResourceType = AZUREFIREWALLS — Categories: ApplicationRule, NetworkRule, DnsProxy, ThreatIntelLog
ResourceType = VAULTS — Categories: AuditEvent
```

---

## Step 6 — Transfer CSV Output to Investigation Spreadsheet

The AI outputs a CSV block. Copy it and paste directly into your investigation spreadsheet. Excel will ask how to split it — choose comma delimited.

Do this immediately after each table. Do not let output accumulate.

**Fields to always review before accepting:**
- **MonitoringFrequency** — AI makes an educated guess. Override if you know the actual pattern.
- **SLA** — AI guesses. You decide based on service commitment.
- **Tier** — AI guesses based on whether a detection depends on it. Verify against the decision tree in `08-silent-detections/silent-detection-standards.md`.
- **DataConnector** — verify against what is actually configured in this environment.

---

## Step 7 — Save KQL Functions As You Go

Paste each sub-function into `functions-in-progress.kql` with a comment header:

```kql
// ============================================================
// GetAzureDiagnosticsHealth()
// Generated: [date]
// Verify filter conditions against actual environment
// ============================================================
[paste function here]
```

---

## Step 8 — Handle Tables Without Detections

Some sources need to be tracked even though no current detection uses them. Compliance requirements, capability dependencies, baseline requirements, client requests.

For these use this prompt:

```
I need to document a data source in Microsoft Sentinel that does not
currently have a custom detection but is being tracked for other reasons.

Table: [table name]
Reason for tracking: [compliance / UEBA dependency / client request / baseline]
Any context: [anything you know]

Please produce a CSV row using the same format and column order as before.
Output the header row and one data row only.
For PURPOSE base it on what this source is generally used for in
security monitoring.
Do not write a sub-function unless it is a shared table.
```

---

## Step 9 — Handle Pending Client Input

Some sources only become known after the client returns their intake questionnaire. Mark these:

```
Verified: No
ActionItems: Pending client input — [what you need from them]
```

---

## Step 10 — Complete Investigation Fields

After all tables are processed go through every row and complete the investigation fields:

**Status** — run a quick query per table in the live environment and record what you see:
```kql
[TableName]
| where TimeGenerated > ago(24h)
| summarize LastSeen = max(TimeGenerated), Count = count()
```

**Verified** — change to Yes once you have confirmed the source exists in the environment and the details are accurate

**ActionItems** — document anything that still needs to be done:
- `SLS detection needed — Tier 1`
- `DCRName unknown — check Azure Monitor`
- `Pending client input on compliance requirement`
- `Breakout shows additional ResourceType not yet documented`

**DCRName and DCEName** — fill in from the Azure portal for AMA-based sources

---

## Step 11 — Final Review Before Upload

Check every row:

- [ ] Table matches exactly what appears in `[mssname]-tables`
- [ ] LogSource is plain English and descriptive
- [ ] All Category, Origin, Transport values from approved list
- [ ] Description is 2-3 sentences — specific not generic
- [ ] Purpose is one sentence for a business audience
- [ ] SLA is True or False — nothing blank
- [ ] MonitoringFrequency is set — nothing blank
- [ ] No Unknown placeholders in DCRName or DCEName — fill in or write N/A
- [ ] FunctionName matches actual function name you will deploy
- [ ] Tier is assigned for every row
- [ ] Verified = Yes for every row — or ActionItems explains why not

---

## Step 12 — Review and Finalize Functions

Before deploying functions verify each one:

- [ ] Returns exactly four columns: Table, LogSource, LastSeen, Status
- [ ] LogSource values match sources watchlist exactly
- [ ] Filter conditions verified against live environment
- [ ] All expected sub-sources included

**Verify filter conditions:**
```kql
// Example for AzureDiagnostics
AzureDiagnostics
| where TimeGenerated > ago(30d)
| summarize Count = count() by ResourceType, Category
| order by Count desc
```

---

## Step 13 — Create the Upload-Ready Watchlist

Once investigation is complete create a clean copy for upload:

1. Save the investigation spreadsheet — keep it as your reference
2. Create a copy named:
```
[ClientName]-sources-[YYYY-MM-DD].xlsx
```
3. In the copy delete these four columns:
   - Tier
   - Status
   - Verified
   - ActionItems
4. The remaining fifteen columns are the watchlist

---

## Step 14 — Upload Watchlist and Deploy Functions

**Watchlist upload:**
1. Export the upload-ready spreadsheet as CSV
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
2. Run each function KQL — verify no errors
3. Click Save → Save as function
4. Name exactly as defined — `GetAzureDiagnosticsHealth` etc
5. Save KQL files to `01-Program/Functions/` in SharePoint

---

## Tips for Best Output

**Include breakout results for shared tables.** Without them the AI only knows what detections cover — not what else is in the environment.

**Include all detections for a table together.** The AI sees the complete picture and avoids redundant rows.

**One table at a time.** Do not batch multiple tables.

**Paste CSV output directly into Excel.** Use Data → Text to Columns if Excel does not split automatically. Choose comma delimited.

**Review Tier carefully.** The AI assigns Tier based on whether a detection depends on the source. But compliance requirements, capability dependencies, and client requests can elevate a source. Cross reference with the decision tree in silent-detection-standards.md.

---

*This document lives in 06-watchlist-management and is maintained in the sentinel-mssp-playbook repository. Update when the process changes, the prompt is refined, or new fields are added.*