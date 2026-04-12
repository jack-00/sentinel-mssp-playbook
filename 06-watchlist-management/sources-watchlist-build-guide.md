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

Paste this entire prompt into the AI at the start of every session. The prompt runs as a two-phase conversation. Phase 1 the AI receives your detections spreadsheet and outputs a reorganized CSV. Phase 2 it switches into table-by-table processing mode and tells you exactly what to send next.

---

**TRAINING PROMPT — paste this first, nothing else:**

```
I am helping build a data source tracking system for Microsoft Sentinel.
This is a two phase process. I will guide you through each phase.

═══════════════════════════════════════
PHASE 1 — RECEIVE DETECTIONS AND OUTPUT CSV
═══════════════════════════════════════

I will paste a detection catalog with these columns:
RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description

The Table field has comma separated table names. One detection
may query multiple tables.

When I paste the detection catalog:
1. Read every detection and every table it queries
2. Reorganize into a table-first view
3. Output a CSV with exactly these two columns:

Table,Detections

Rules:
- One row per unique table
- If a detection queries multiple tables list it under each table
- Detections column lists all detections for that table comma
  separated — include RuleId and name like:
  OC00034 - Detection Name, OC00041 - Detection Name
- Sort rows alphabetically by Table
- Output only the CSV — no explanation, no markdown
- After the CSV output say exactly:
  PHASE 1 COMPLETE. Copy the CSV above into a spreadsheet.
  When you are ready for Phase 2 type: READY

═══════════════════════════════════════
PHASE 2 — TABLE BY TABLE PROCESSING
═══════════════════════════════════════

When I type READY switch to Phase 2 mode and say:
"Ready for Phase 2. Send me one table at a time in this order:

  TABLE NAME

  Detection name
  Full KQL query

  Detection name
  Full KQL query

  ENVIRONMENT DATA:
  Results from live breakout query showing what is actually
  writing to this table in the environment.

  Send the table name first followed by each detection name
  and its full KQL query. Then add ENVIRONMENT DATA if you
  have breakout results. I will process one table at a time."

For each table I send produce two outputs:

───────────────────────────────────────
OUTPUT 1 — CSV ROWS
───────────────────────────────────────

One row per unique source — not one per detection. Multiple
detections may depend on the same source. If three detections
all query Azure Firewall data that is one row not three.

On the very first table only output the header row:
Table,LogSource,Category,Origin,Transport,Description,Purpose,SLA,DataConnector,DCRName,DCEName,FunctionName,SLS,MonitoringFrequency,Tier,Verified,ActionItems,Confidence,Notes

Then one data row per unique source found. Use these approved values:

Category: Identity / Endpoint / Email / Network / Firewall /
  Cloud Infrastructure / SaaS Application / Threat Intelligence /
  Compliance and Audit / Vulnerability Management / Authentication /
  Capabilities / Other

Origin: Microsoft Entra ID / Microsoft 365 / Microsoft Defender /
  Microsoft Azure / Microsoft Sentinel / Windows OS / Network Device /
  AWS / Salesforce / Zoom / Keeper Security / SecureW2 / Drupal /
  Multiple Sources / Custom / Unknown

Transport: Microsoft Connector / XDR Connector / AMA - DCR /
  CEF - AMA - DCR / Diagnostic Setting / Logic App - Polling /
  REST API - Push / TAXII - Polling / Sentinel Native / ASIM Parser

SLA: True or False
DataConnector: best educated guess from table name and origin
DCRName: Unknown
DCEName: Unknown
FunctionName: Get[TableName]Health for shared tables / None for single source
SLS: Missing

MonitoringFrequency:
  1h = continuous real-time
  5h = near-continuous
  15h = business-hours active
  24h = daily batch
  48h = less frequent
  None = monitoring not needed

Tier:
  1 = OC-series detection depends on this source
  2 = high value but no current detection
  3 = ASIM or capability output
  4 = no dependencies and no compliance requirement

Verified: No
ActionItems: flags for engineer — Unknown DCR / SLS needed /
  verify in environment / pending client input
Confidence: High / Medium / Low — how confident in MonitoringFrequency
  and Tier assignments. Briefly explain reasoning.
Notes: misconfigurations gotchas things to verify.

Special rules:
- If detection uses search * or union * with no table filter:
  LogSource = All Tables — workspace wide detection
  FunctionName = None
- If ENVIRONMENT DATA shows sources no detection covers
  still include them — they may need monitoring
- Wrap any field containing commas in double quotes

───────────────────────────────────────
OUTPUT 2 — KQL SUB-FUNCTION
───────────────────────────────────────

Write a sub-function ONLY for shared tables where multiple
distinct sources write and are identified by filtering on
a specific field:
- AzureDiagnostics — ResourceType and Category
- CommonSecurityLog — DeviceVendor and DeviceProduct
- Syslog — HostName
- ThreatIntelIndicators — SourceSystem
- ASimNetworkSessionLogs — EventVendor and EventProduct
- ASimAuditEventLogs — EventVendor and EventProduct
- ASimWebSessionLogs — EventVendor and EventProduct

Do NOT write a sub-function for single source tables like
SignInLogs AuditLogs SecurityEvent OfficeActivity Device* tables.

Every sub-function MUST return exactly these four columns:
Table, LogSource, LastSeen, Status

The LogSource value in the function must exactly match the
LogSource value in the CSV row — this is the join key.

Use this exact structure:

---FUNCTION START---
// Get[TableName]Health()
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
---FUNCTION END---

───────────────────────────────────────
AFTER EACH TABLE
───────────────────────────────────────

After each table say exactly:
"Table complete. Send the next table when ready."

Do not process the next table until I send new data.

═══════════════════════════════════════
CONFIRM
═══════════════════════════════════════

Confirm you understand the two phase process. Then say:
"Ready for Phase 1. Please paste your detection catalog now."
```

---

Wait for the AI to say it is ready for Phase 1 before pasting anything.

---

## Step 3B — Set Up Your Working Spreadsheet

After Phase 1 completes the AI outputs a two-column CSV showing each table and its detections. Before moving to Phase 2 set up your working spreadsheet.

**Name the spreadsheet:**
```
[ClientName]-sources-investigation-[YYYY-MM-DD].xlsx
```

**Open a new Excel workbook and set up two tabs:**

**Tab 1 — Sources** — this is where Phase 2 CSV output goes. As the AI produces rows for each table paste them here. By the end of Phase 2 this tab has one row per source across all tables.

**Tab 2 — Table Map** — paste the Phase 1 CSV output here. This is your working reference showing which detections belong to each table. You will refer to this as you work through Phase 2.

**After pasting Phase 2 output into Tab 1:**
1. Click any cell in the data
2. Press **Ctrl+T** to format as Excel Table
3. Click OK
4. This makes filtering and sorting work cleanly

**Upload to SharePoint immediately:**
Upload the workbook to SharePoint before filling in anything:
```
02-Clients/[ClientName]/04-Watchlists/Current/
```
This is your baseline snapshot before investigation.

**The investigation columns — Tab 1 will have these extra fields beyond the watchlist:**
- **Tier** — assign using the silent detection tier framework in `08-silent-detections/silent-detection-standards.md`
- **Verified** — change from No to Yes after you confirm the source against the live environment
- **ActionItems** — update as you investigate — missing DCR names, SLS detections needed, pending client input
- **Confidence** — AI's confidence in MonitoringFrequency and Tier — Low means you need to verify before accepting

**When investigation is complete and ready for watchlist upload:**
1. Make a copy of Tab 1
2. Delete the four investigation columns: Tier, Verified, ActionItems, Confidence
3. Save the copy as:
```
[ClientName]-sources-[YYYY-MM-DD].xlsx
```
4. Upload this as the `[mssname]-sources` watchlist
5. Keep the investigation workbook in SharePoint for reference

---

## Step 4 — Work Through Tables One by One

Once the AI confirms it is in Phase 2 mode it will tell you exactly what format to send. Work through the table-first CSV from Phase 1 in order — one table at a time.

**For each table type READY then send this in your text editor:**

```
TABLE NAME

Detection name
Full KQL query

Detection name
Full KQL query

ENVIRONMENT DATA:
[paste breakout query results here for shared tables]
```

**Example:**
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

ENVIRONMENT DATA:
ResourceType = AZUREFIREWALLS — Categories seen: ApplicationRule, NetworkRule, DnsProxy, ThreatIntelLog
ResourceType = VAULTS — Categories seen: AuditEvent
```

**For single source tables** — SignInLogs, AuditLogs, SecurityEvent etc — you do not need ENVIRONMENT DATA. Just the table name and detections.

**Why include all detections for a table together:**
The AI sees the full picture and can identify distinct sub-sources accurately. If two detections both query Azure Firewall it produces one row not two.

**Why include ENVIRONMENT DATA for shared tables:**
The detections may only cover some sub-sources present in the environment. The breakout results tell the AI what else exists so it can include those sources too — even if no detection currently covers them.

**After each table:**
The AI outputs the CSV rows and the KQL function if needed. Then says Table complete. You paste the next table. Repeat until all tables are done.

---

## Step 5 — Build the Working Spreadsheet from CSV Output

After Phase 1 completes copy the CSV output and paste it into a new spreadsheet. This is your working investigation spreadsheet — not the final upload yet.

**Spreadsheet name:**
```
[ClientName]-sources-investigation-[YYYY-MM-DD].xlsx
```

**After pasting Phase 1 CSV:**
1. Format as Excel Table — Ctrl+T
2. Save to SharePoint at `02-Clients/[ClientName]/04-Watchlists/Current/`

**As you work through each table in Phase 2:**
Copy the CSV rows the AI outputs for each table and paste them into this same spreadsheet. The header row was output on the first table — do not paste it again for subsequent tables. Just paste the data rows.

**Fields to always review before accepting:**
- **MonitoringFrequency** — AI makes an educated guess. Check the Notes field for its reasoning. If you know the actual ingestion pattern override it.
- **SLA** — AI guesses based on source nature. You decide based on service commitment to this client.
- **DataConnector** — verify against what is actually configured in this environment.
- **Tier** — review each assignment using the decision tree in `08-silent-detections/silent-detection-standards.md`.
- **DCRName and DCEName** — fill these in from the environment. Replace Unknown with actual values.

**The investigation columns — review and complete:**
- **Verified** — change to Yes after you have confirmed this source exists in the live environment
- **ActionItems** — work through each flag — missing DCR, SLS detection needed, pending client input

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

## Step 9 — Prepare Upload Version

Before uploading create a clean copy of the spreadsheet with the investigation columns removed.

**Remove these columns — investigation use only:**
- Tier
- Verified
- ActionItems

**Save the upload version as:**
```
[ClientName]-sources-[YYYY-MM-DD].xlsx
```

Keep the investigation version. The upload version is what goes to Sentinel.

---

## Step 10 — Final Review Before Upload

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