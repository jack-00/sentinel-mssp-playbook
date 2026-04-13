# Sources Watchlist Build — Working Process

> **Status:** This is the working document for developing and testing the sources watchlist build process. Once verified and stable it will be used to update the final guidance in `06-watchlist-management/sources-watchlist-build-guide.md`.
>
> **Last Updated:** April 2026

---

## Where We Are in the Process

At this point you should have completed the data source audit runbook and have these two files ready:

- `[client]-tables-[date].xlsx` — tables watchlist, already uploaded to Sentinel as `[mssname]-tables`
- `[client]-detections-[date].xlsx` — detections watchlist, already uploaded to Sentinel as `[mssname]-detections`

**The goal of this process** is to produce two more things:

1. `[client]-sources-[date].xlsx` — the sources watchlist that drives all monitoring
2. KQL sub-functions — one per shared table, saved per client and deployed to Sentinel

This process works table by table. By the time you finish the last table both outputs are complete.

---

## Step 1 — Create the Table Investigation Map

Before touching any individual table you need a complete map of every table in the environment and which detections depend on each one. This is your working reference for the entire process.

**You need both files open and ready:**
- Your detections CSV
- Your tables inventory export from the audit runbook

**Paste this prompt to your AI. Wait for confirmation before sending any data:**

```
I am building a data source tracking system for Microsoft Sentinel.

I need you to help me create a complete table investigation map.

I will give you two things:
1. A detection catalog CSV with these columns:
   RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description
   The Table field contains comma separated table names — one detection
   may query multiple tables.

2. A list of all tables that exist in the workspace.

Your job is to produce a complete table map that includes every table.

Output format — one row per table with these four columns:
Table, Detections, Verified, Notes

Rules:
- If a detection queries multiple tables list it under each table
- Group all detections for the same table on one row comma separated
- Include the RuleId and rule name for each detection
- Include ALL tables from the workspace list even if no detections use them
- For tables with no detections leave the Detections column blank
- Sort tables alphabetically
- Include column headers in the output

After processing confirm how many tables have detections and how many
have none so I can verify the output is complete.

Confirm you understand before I provide the data.
```

**After confirmation:**
1. Paste your detections CSV
2. Paste your tables list
3. When complete ask: `Please output this as raw CSV`
4. Copy the raw CSV output into a text editor
5. Save as `[client]-source-investigation-[date].csv`

**Then in Excel:**
1. Open the CSV
2. Ctrl+A to select all
3. Insert → PivotTable → OK
4. Set up and sort alphabetically by Table
5. Home → Format → AutoFit Column Width
6. Save As → `[client]-source-investigation-[date].xlsx`
7. Move the original CSV to Archive folder in SharePoint

You now have your working investigation map. Every table is listed. Detections are grouped by table. Tables with no detections have blank detection fields.

---

## Step 2 — Create the Empty Sources Spreadsheet

Before working through any tables create the empty sources spreadsheet with the correct column headers. You will paste AI output into this as you go through each table.

**Column headers — copy this exact line into a blank text file and save as CSV:**
```
Table,LogSource,Category,Origin,Transport,Description,Purpose,SLA,DataConnector,DCRName,DCEName,FunctionName,SLS,MonitoringFrequency,SentinelCapabilities,Notes
```

**Note on SentinelCapabilities:**
This field captures which Sentinel capabilities depend on this source — UEBA, Fusion ML, Threat Intelligence, SOC Optimization, or None. You will not know this upfront for every source. The AI analysis in Step 5 will tell you. Fill it in as you discover it. At the end of the sources build process use what you have learned here to go back and add CAP-series rows to the detections watchlist for any capabilities identified.

Save as:
```
[client]-sources-[date].xlsx
```

Keep this open alongside your investigation file throughout the process.

---

## Step 3 — Create a Text File Per Table

For each table in your investigation map create an individual text file. This is your working document for that table. Save all text files in a folder:

```
[client]-table-investigations/
    AzureDiagnostics.txt
    CommonSecurityLog.txt
    SignInLogs.txt
    ...
```

**The template for each text file:**

```
TABLE: [TableName]
FUNCTION NAME: Get[TableName]Health()

--- DISCOVERY ---

SCHEMA:
[paste output of: TableName | getschema]

SOURCE BREAKDOWN:
[for shared tables — list distinct source identifiers found]
[example: ResourceType = AZUREFIREWALLS, ResourceType = VAULTS]
[for single source tables — write: Single source]

CONSUMERS:
[Name and RuleId of anything that relies on this table]
[This includes detections, reports, workbook queries, capabilities, or client requirements]
[For each one paste the full KQL or description below the name]
[repeat for each consumer]
```

The text file gives AI everything it needs. When AI responds paste its output directly above this template at the top of the file. All detailed field values go into the sources spreadsheet — nothing else needed here.

---

## Step 4 — Discovery Queries Per Table

Before feeding anything to AI run these queries in Sentinel Logs to gather what you need. Paste results into the text file.

**For every table — get the schema:**
```kql
TableName
| getschema
```
This returns field names and data types only. No client data. Safe to share with AI.

**For shared tables only — get the source breakdown:**

These queries tell you what is actually writing to the table right now. Run the one that matches your table. Paste the field names and distinct values — not the full results.

```kql
// AzureDiagnostics
AzureDiagnostics
| where TimeGenerated > ago(30d)
| summarize count() by ResourceType, Category
| order by count_ desc

// CommonSecurityLog
CommonSecurityLog
| where TimeGenerated > ago(30d)
| summarize count() by DeviceVendor, DeviceProduct
| order by count_ desc

// Syslog
Syslog
| where TimeGenerated > ago(30d)
| summarize count() by HostName
| order by count_ desc

// ThreatIntelIndicators
ThreatIntelIndicators
| where TimeGenerated > ago(30d)
| summarize count() by SourceSystem
| order by count_ desc

// ASIM tables
ASimNetworkSessionLogs
| where TimeGenerated > ago(30d)
| summarize count() by EventVendor, EventProduct
| order by count_ desc
```

For any other table you are not sure about:
```kql
TableName
| where TimeGenerated > ago(30d)
| take 1
| project-away *_s, *_d, *_b, *_g
```
This shows you the main fields without exposing sensitive values.

---

## Step 5 — Feed Table to AI for Analysis

Once the text file is populated with schema and breakout results feed it to AI with this prompt. Do this one table at a time.

**Prompt:**

```
I am documenting data sources for a Microsoft Sentinel environment.
I will give you a table investigation file. It contains schema information,
source breakdown results, and a list of consumers that rely on this table.
Consumers may include custom detections, scheduled reports, workbook queries,
Sentinel capabilities, or client compliance requirements — each with their
KQL or description included. Analyze everything and give me:

SECTION 1 — TABLE SUMMARY
2-3 sentences on what this table is and what data lands here.
Then tell me how many distinct data sources you identify within it
and list each one by name.
For each source tell me:
- The plain English LogSource name
- The function name following this convention: Get[TableName]Health()
  Note: one function per table covers all sources within it
- Industry perspective on security importance of this data —
  how do most enterprise SOCs view this source
- Whether any Sentinel capabilities depend on it —
  UEBA, Threat Intelligence, Fusion ML, or None
- Whether any of the provided detections depend on this source

SECTION 2 — SOURCES WATCHLIST ROWS
One row per data source identified. Use these exact column headers:
Table, LogSource, Category, Origin, Transport, Description,
Purpose, SLA, DataConnector, DCRName, DCEName, FunctionName,
SLS, MonitoringFrequency, SentinelCapabilities, Notes

Rules for filling in fields:
- SLA: True if this is a primary security source — False if supporting
- DataConnector: best guess based on table and origin
- DCRName: write Unknown
- DCEName: write Unknown
- FunctionName: Get[TableName]Health() for all sources in this table
- SLS: write Missing
- MonitoringFrequency: your best estimate —
  1h for continuous real-time sources
  5h for near-continuous
  15h for business-hours active
  24h for daily batch sources
  48h for less frequent
  None if monitoring not appropriate
- SentinelCapabilities: list any Sentinel capabilities that depend
  on this source — UEBA, Fusion ML, Threat Intelligence, SOC Optimization.
  Write None if no capabilities depend on it.
  This is important — the engineer will use this to update the
  detections watchlist with CAP-series rows after the sources build
  is complete.
- Notes: include your confidence level on MonitoringFrequency
  and anything the engineer should verify

A source earns a row only if at least one of these is true:
- A detection queries it
- A Sentinel capability depends on it
- A workbook or report reads from it
- A compliance obligation requires it
- It is a custom integration that could break silently

Display the rows in a clean column format — not raw CSV.
I will copy and paste them into my spreadsheet manually.

Here is the investigation file:
[paste your text file content]
```

---

## Step 6 — Transfer Output to Your Files

After AI responds:

1. Copy the entire AI response and paste it into the text file under `--- AI OUTPUT ---`
2. Copy the sources rows from Section 2 and paste them into your sources spreadsheet
3. Fill in DCRName and DCEName from what you know about this environment
4. Note any SentinelCapabilities the AI identified — you will use these later to add CAP-series rows to the detections watchlist

The text file is now your complete record for this table. The sources spreadsheet has the row ready for upload. Move on to the next table.

Repeat Steps 3 through 6 for every table in your investigation map.

---

## Step 7 — Functions

Functions are written after all sources rows are complete. This is a separate pass. By the time you finish Step 6 for all tables you have everything the functions need — table name, LogSource names, filter conditions from the breakout results.

Function writing process will be documented here once the sources process is validated.

---

## Cleanup Step — Edge Cases and Validation

Before uploading the sources watchlist as final work through these checks:

**Master detection lookback:**
The master source detection lookback window must be set to match the longest MonitoringFrequency value in the sources watchlist. If any source has MonitoringFrequency = 48h the detection must look back at least 48h. If the lookback is shorter than the longest MonitoringFrequency the detection will never fire for that source even when it should.

Check: what is the longest MonitoringFrequency value in your sources watchlist? Set the master detection lookback to match.

**Master detection schedule:**
The master detection runs on a fixed schedule — recommended hourly. It checks each source against its own MonitoringFrequency threshold. A source with MonitoringFrequency = 1h fires after 1 hour of silence. A source with MonitoringFrequency = 48h fires after 48 hours. The hourly schedule means you find out within one hour of any threshold being breached.

**Edge cases that need a custom SLS detection instead of relying on master:**
- Source sends data less frequently than 48h — master cannot handle this
- Source needs logic beyond a simple time check — for example minimum record count, specific event IDs must be present, specific field values must appear
- Source needs its own schedule different from the master

For these cases create a custom OC-series SLD detection and store the ID in the SLS field in sources.

**MonitoringFrequency = None sources:**
These are excluded from the master detection entirely. Verify that every source with None is intentional — either Tier 3 or Tier 4 — and not an oversight.

---

## Notes and Open Questions

- Functions process to be documented after sources process is validated
- Client questionnaire input — CLT-series rows in detections — need to be incorporated into sources after client returns questionnaire
- Verify edge cases around custom integrations — Logic App sources, REST API sources — these may need custom SLS detections due to batch timing
- ASIM table sources — health depends on source tables not the ASIM table itself — ensure FunctionName points to source table function not ASIM function
