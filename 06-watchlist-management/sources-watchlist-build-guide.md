# Building the Sources Watchlist — Process and AI Guidance

> **Purpose:** This document guides an engineer through the complete process of building the `[mssname]-sources` watchlist and the associated KQL sub-functions. It covers the full workflow from understanding why we are doing this, through gathering all the inputs we need, to producing a complete working spreadsheet that becomes both a monitoring system and an audit deck data source.
>
> **Who uses this:** Any engineer building or updating the sources watchlist. Works with any capable LLM including internal company tools.
>
> **What you produce:**
> 1. A completed `[mssname]-sources` watchlist spreadsheet ready for upload to Sentinel
> 2. KQL sub-functions for all shared tables ready for deployment
> 3. A client intake questionnaire to gather what the data alone cannot tell us
>
> **Where outputs go:**
> - Completed spreadsheet → `02-Clients/[ClientName]/04-Watchlists/Current/`
> - KQL functions → `02-Clients/[ClientName]/06-Functions/` and `01-Program/Functions/` if generic
> - AI prompts → `01-Program/AI-Prompts/sources-watchlist-prompt.md`
>
> **Prerequisites:** Complete the data source audit runbook and upload `[mssname]-tables` before starting this process.
>
> **Last Updated:** April 2026

---

## Why We Are Doing This — Read This First

Before touching a spreadsheet or running a query understand what we are actually trying to accomplish here and why the approach we are taking is the right one.

---

### The Problem With How Most MSSPs Work

Most managed security providers track data sources the wrong way. They look at connectors. A connector shows green so they assume everything is fine. Or they look at tables — if a table has data something must be working. Neither of these is granular enough. A connector can show green while silently sending nothing. A table can appear healthy because one source is sending while another source that a critical detection depends on has been silent for a week.

The other problem is manual audit decks. Before every client review someone spends hours pulling screenshots, running queries, copying numbers into a PowerPoint. The data is stale before the meeting starts. If something changed yesterday nobody knows.

This program solves both problems simultaneously.

---

### What The Sources Watchlist Actually Does

The `[mssname]-sources` watchlist is not just a list of data sources. It is the configuration layer that drives two completely separate and equally important systems:

**System 1 — Automated monitoring**

Every row in the sources watchlist with a MonitoringFrequency set is automatically monitored by the master source detection. That detection reads the watchlist, calls the appropriate KQL function for each source, checks when data last arrived, and fires an incident if any source exceeds its threshold. No manual checking. No connector widgets. The logs tell the truth.

This is why creating one detection per data source the old way is the wrong approach. If you have thirty data sources you would write thirty separate detection rules. Each one hardcoded to a specific table and threshold. Change a threshold means editing a rule. Add a new source means writing a new rule. Thirty rules to maintain, thirty rules that can drift.

Under this approach you write two master detections and one sub-function per shared table. Everything else is watchlist configuration. Add a new source — update the watchlist. Change a threshold — update the watchlist. The code never changes.

**Here is a concrete example of why functions are powerful:**

AzureDiagnostics has four sub-sources — Azure Firewall Application Rule, Azure Firewall Network Rule, Azure Firewall DNS Proxy, and Azure Firewall Threat Intel Log. Without functions you write four separate detection rules. With a function you write one:

```kql
// GetAzureDiagnosticsHealth() — one function, four sources monitored
union
(
    AzureDiagnostics
    | where ResourceType == "AZUREFIREWALLS"
    | where Category == "AzureFirewallApplicationRule"
    | summarize LastSeen = max(TimeGenerated)
    | extend LogSource = "Azure Firewall — Application Rule"
),
(
    AzureDiagnostics
    | where ResourceType == "AZUREFIREWALLS"
    | where Category == "AzureFirewallNetworkRule"
    | summarize LastSeen = max(TimeGenerated)
    | extend LogSource = "Azure Firewall — Network Rule"
),
(
    AzureDiagnostics
    | where ResourceType == "AZUREFIREWALLS"
    | where Category == "AzureFirewallDnsProxy"
    | summarize LastSeen = max(TimeGenerated)
    | extend LogSource = "Azure Firewall — DNS Proxy"
),
(
    AzureDiagnostics
    | where ResourceType == "AZUREFIREWALLS"
    | where Category == "AzureFirewallThreatIntelLog"
    | summarize LastSeen = max(TimeGenerated)
    | extend LogSource = "Azure Firewall — Threat Intel Log"
)
| extend Table = "AzureDiagnostics"
| extend DaysSince = datetime_diff('day', now(), LastSeen)
| extend Status = case(
    DaysSince <= 1, "Active",
    DaysSince <= 7, "Review",
    DaysSince > 7,  "Inactive",
    "No Data"
)
| project Table, LogSource, LastSeen, Status
```

The master detection calls this one function and gets back four rows — one per source — each with its own health status. Add a fifth Azure Firewall category to monitor — add one block to the function, add one row to the watchlist. Nothing else changes.

**System 2 — The audit deck**

The same watchlist that drives monitoring also powers the workbook. The GetEnrichedInventory function joins the sources watchlist against the detections watchlist and live workspace data to produce the complete picture — every source, its current health, how many detections depend on it, when it last sent data. The audit deck data sources tab reads from this function. No manual preparation. Open the workbook and the truth is right there.

This is the win-win. One piece of work — building and maintaining the sources watchlist — produces two completely separate benefits. Monitoring is automated. The audit deck builds itself.

---

### What We Need to Track and Why

The goal of this process is to identify every data source that matters and document it with enough context to monitor it properly. A source matters if any of the following are true:

**Detections depend on it** — if this source goes silent a detection goes blind. The client loses security coverage without knowing it. This is the most important reason to track a source and it is entirely objective — either a detection queries this source or it does not.

**Reports reference it** — scheduled reports delivered to the client read from this source. If it goes silent the report produces incomplete data. The client may not notice but the report loses integrity. This includes any workbook queries that display data from this source.

**Workbook queries use it** — the audit deck and internal workbook both query specific tables and sources. If those sources go silent the workbook shows stale or missing data. The workbook is only as good as the data behind it.

**Capabilities depend on it** — UEBA, Threat Intelligence, Fusion ML each have specific data source dependencies. If any dependency goes silent the capability degrades — often silently.

**Compliance or retention requires it** — regulatory frameworks like PCI-DSS, HIPAA, and SOC 2 mandate specific log collection and retention periods. A source may need to be tracked and retained even if no current detection uses it.

**The client identified it** — the client may have business-specific reasons to track certain sources. Systems that are critical to their operations, data that they have been asked to produce for audits in the past, sources that have caused problems before.

---

### The Three Inputs

We gather information from three sources to build the watchlist. No single input is sufficient on its own.

**Input 1 — Detections, reports, and workbook KQL**
This tells us what data is currently being used for security purposes. Feed this to AI to identify tables and sub-sources.

**Input 2 — Live environment breakout queries**
This tells us what is actually writing to each table right now. The detection may only filter on one sub-source but the environment may have others that also need tracking. Run these queries in `resources/scratch.md` for every shared table.

**Input 3 — Client intake questionnaire**
This tells us what the data alone cannot tell us. Compliance requirements, retention needs, business-critical systems, legal obligations, past incidents. The client fills this in and we incorporate their answers into the watchlist.

All three inputs together give us the complete picture.

---

## The Complete Workflow — Overview

```
Phase 1 — Gather all inputs
  Step 1 — Collect detections, report KQL, and workbook KQL
  Step 2 — Reorganize by table using AI
  Step 3 — Run environment breakout queries for shared tables
  Step 4 — Send client intake questionnaire

Phase 2 — Build the watchlist with AI assistance
  Step 5 — Train the AI
  Step 6 — Work through tables one by one
  Step 7 — Incorporate client questionnaire responses
  Step 8 — Handle tables without any KQL dependency

Phase 3 — Finalize and deploy
  Step 9 — Final review of spreadsheet
  Step 10 — Review and finalize KQL functions
  Step 11 — Upload watchlist and deploy functions
```

---

## Standards and Taxonomy

Know the approved values for every field before starting. The AI uses these — knowing them helps you verify its output.

**Category:** Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other

**Origin:** Microsoft Entra ID / Microsoft 365 / Microsoft Defender / Microsoft Azure / Microsoft Sentinel / Windows OS / Network Device / AWS / Salesforce / Zoom / Keeper Security / SecureW2 / Drupal / Multiple Sources / Custom / Unknown

**Transport:** Microsoft Connector / XDR Connector / AMA — DCR / AMA — DCR — DCE / CEF — AMA — DCR / Cribl — AMA — DCR / Cribl — DCR / Diagnostic Setting / Logic App — Polling / Logic App — Webhook / REST API — Push / TAXII — Polling / Sentinel Native / ASIM Parser / Application Insights SDK

**SLA:** True / False

**MonitoringFrequency:** None / 1h / 5h / 15h / 24h / 48h

**SLS:** OC##### or Missing

**Tier:** 1 / 2 / 3 / 4

---

## Prerequisites — Have These Ready

1. **Detections spreadsheet** — `[mssname]-detections` with RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description
2. **Report KQL** — KQL queries from any scheduled reports configured for this client
3. **Workbook KQL** — KQL queries from the audit deck and internal workbook tabs that reference specific tables
4. **Sources investigation spreadsheet** — open with column headers in this exact order:
```
Table, LogSource, Category, Origin, Transport, Description,
Purpose, SLA, DataConnector, DCRName, DCEName, FunctionName,
SLS, MonitoringFrequency, Notes, Tier, Verified, ActionItems
```
5. **Functions document** — `functions-in-progress.kql` in the client's `06-Functions` folder
6. **Text editor** — staging area before sending to AI
7. **A capable LLM** — use your internal company LLM for client-specific information

**Spreadsheet naming:**
```
[ClientName]-sources-investigation-[YYYY-MM-DD].xlsx
```

When investigation is complete and ready for watchlist upload:
1. Remove Tier, Verified, and ActionItems columns
2. Save a copy as `[ClientName]-sources-[YYYY-MM-DD].xlsx`
3. Upload the sources copy as the watchlist
4. Keep the investigation copy in SharePoint for reference

---

## Phase 1 — Gather All Inputs

### Step 1 — Collect Detections, Report KQL, and Workbook KQL

Gather three types of KQL input:

**Detections** — your `[mssname]-detections` spreadsheet. Already have this.

**Report KQL** — open any scheduled Logic App reports configured for this client. Copy the KQL queries they use. These queries reference specific tables and sources that the report depends on. If a source goes silent the report breaks — it needs to be tracked.

**Workbook KQL** — open the audit deck workbook and internal workbook JSON. Find every query that references a specific table by name. These are sources the workbook depends on. If they go silent the workbook shows incorrect data.

Compile all of this into one document — detections first, then report queries, then workbook queries. Label each section clearly so you know what type each query is when you feed it to the AI.

---

### Step 2 — Reorganize by Table Using AI

Your detections have comma separated table names. A detection that queries three tables appears once but belongs under all three. Report and workbook queries may also reference multiple tables. A simple sort does not handle this.

Feed everything to AI with this prompt to get a clean table-first view:

```
I have KQL queries from three sources — detection rules, scheduled
reports, and workbook queries. I need you to reorganize all of this
into a table-first view showing which queries depend on each table.

For detection rules the input format is:
RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description

For report and workbook queries I will provide the query name and
full KQL.

Please produce a table-first reference in this format:

TABLE: [exact table name]
DETECTIONS: [RuleId - Name, RuleId - Name]
REPORTS: [Report name if any report query uses this table]
WORKBOOK: [Workbook tab name if any workbook query uses this table]
DEPENDENCY SUMMARY: [one sentence — why this table matters based on
what is querying it]

Rules:
- If a query references multiple tables list it under each table
- Sort tables alphabetically
- If a table is referenced by detections AND reports AND workbook
  queries list all of them
- If a table is only referenced by one type still list it

[paste your detections CSV and all report and workbook KQL here]
```

Save this table-first reference. It is your working map for Phase 2.

---

### Step 3 — Run Environment Breakout Queries

For every shared table in your table-first reference run the appropriate breakout query from `resources/scratch.md`. These queries show you what is actually writing to each table right now in this specific environment.

**Shared tables that need breakout queries:**
- AzureDiagnostics — ResourceType and Category breakout
- CommonSecurityLog — DeviceVendor and DeviceProduct breakout
- Syslog — HostName breakout
- ThreatIntelIndicators — SourceSystem breakout
- ThreatIntelObjects — SourceSystem breakout
- ASimNetworkSessionLogs — EventVendor and EventProduct breakout
- ASimAuditEventLogs — same
- ASimWebSessionLogs — same

Note the results for each shared table. You will include these when you feed each table to the AI in Phase 2. This is how you catch sub-sources that exist in the environment but no detection currently covers — they may still need monitoring for other reasons.

For single source tables — SignInLogs, AuditLogs, SecurityEvent, all Device tables — skip this step.

---

### Step 4 — Send Client Intake Questionnaire

While you are building the watchlist from data the client has context you cannot get from queries. Send them this questionnaire and incorporate their responses in Step 7.

---

**CLIENT INTAKE QUESTIONNAIRE**

*Please complete this with input from your security, compliance, and legal teams where needed. This information helps us ensure we are collecting and monitoring all the data that matters to your organization.*

**Section 1 — Compliance and Legal**

1. Which compliance frameworks apply to your organization?
   - PCI-DSS — if yes, which systems are in scope
   - HIPAA — if yes, which systems contain PHI
   - SOC 2 — if yes, which trust principles
   - ISO 27001 — if yes, certification scope
   - GDPR — if yes, which systems process personal data
   - NIST 800-171 — if yes, which systems handle CUI
   - Other — please specify

2. What log retention periods are required by your compliance obligations?

3. Have you ever needed to produce log evidence for a legal matter, regulatory audit, or internal investigation? If yes — what types of logs were required?

4. Are there any systems where log collection is required by contract — for example a cyber insurance policy or client contract?

**Section 2 — Business Critical Systems**

1. Are there any systems or applications that are particularly sensitive or critical to your business operations?

2. If any of these systems experienced a security incident would you expect to have detailed logs available for investigation?

3. Are there upcoming changes to your environment — new systems, cloud services, or vendors — that we should factor into our monitoring?

**Section 3 — Past Issues**

1. Have you had any security incidents in the past where log data was missing or insufficient for the investigation?

2. Are there any systems where you have experienced visibility gaps that concerned you?

3. Are there any systems you specifically want us to ensure are monitored?

**Section 4 — Reports and Visibility**

1. What security reports do you currently receive and what data do they show?

2. Are there any reports you wish you had but do not currently receive?

3. Are there specific metrics or visibility needs you want to see in your quarterly audit review?

---

## Phase 2 — Build the Watchlist With AI Assistance

### Step 5 — Train the AI

Paste this entire prompt at the start of every session. Wait for confirmation before proceeding.

---

**TRAINING PROMPT:**

```
I am building a data source tracking system for Microsoft Sentinel.
I will provide you with information about one table at a time. For each
table I will give you:
- The table name
- All detections that query this table with their full KQL
- Any report or workbook queries that reference this table
- For shared tables — what the live environment breakout query shows
  is actually writing to this table

You will analyze everything together and produce two types of output
followed by a CSV summary row.

═══════════════════════════════════════
OUTPUT TYPE 1 — WATCHLIST ROWS
═══════════════════════════════════════

Produce one watchlist row block per unique log source that any of
the provided queries depend on within this table.

Think table-first not query-first. Multiple queries may depend on
the same source. Produce one row per unique source — not one row
per query. If three detections all query Azure Firewall data that
is one source row — not three.

Also consider the breakout results. If the environment has a source
that no current query covers but that source is present and active
it may still need a row — flag it in NOTES for the engineer to decide.

Use this exact format for each row:

---WATCHLIST ROW---
TABLE: [exact table name]
LOGSOURCE: [plain English name of this specific source]
CATEGORY: [Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other]
ORIGIN: [Microsoft Entra ID / Microsoft 365 / Microsoft Defender / Microsoft Azure / Microsoft Sentinel / Windows OS / Network Device / AWS / Salesforce / Zoom / Keeper Security / SecureW2 / Drupal / Multiple Sources / Custom / Unknown]
TRANSPORT: [Microsoft Connector / XDR Connector / AMA — DCR / CEF — AMA — DCR / Diagnostic Setting / Logic App — Polling / REST API — Push / TAXII — Polling / Sentinel Native / ASIM Parser / other if none fit]
DESCRIPTION: [2-3 sentences — specific, natural, uses table context and what the queries do with the data]
PURPOSE: [one sentence — what attack, risk, or business need does this source serve]
DEPENDENCYTYPE: [Detection / Report / Workbook / Capability / Compliance / Multiple — what type of dependency justifies tracking this source]
SLA: [True if primary security source client expects monitored — False otherwise]
DATACONNECTOR: [best educated guess at Sentinel connector name]
DCRNAME: [Unknown — engineer fills in]
DCENAME: [Unknown — engineer fills in]
FUNCTIONNAME: [Get[TableName]Health for shared tables — None for single source]
SLS: [Missing — engineer assigns later]
MONITORINGFREQUENCY: [1h / 5h / 15h / 24h / 48h / None]
CONFIDENCE: [High / Medium / Low — explain MonitoringFrequency reasoning]
TIER: [1 / 2 / 3 / 4 — based on dependency type and what breaks if silent]
NOTES: [gotchas, things to verify, sources found in breakout but not covered by queries]
---END ROW---

═══════════════════════════════════════
OUTPUT TYPE 2 — KQL SUB-FUNCTION
═══════════════════════════════════════

After all watchlist rows write a KQL sub-function IF this is a shared
table — multiple distinct sources write to it identified by a filter.

Shared tables:
- AzureDiagnostics — ResourceType and Category
- CommonSecurityLog — DeviceVendor and DeviceProduct
- Syslog — HostName
- ThreatIntelIndicators — SourceSystem
- ASimNetworkSessionLogs, ASimAuditEventLogs, ASimWebSessionLogs
  — EventVendor and EventProduct

Do NOT write a sub-function for single source tables.

Every sub-function MUST return exactly these four columns:
Table, LogSource, LastSeen, Status

The LogSource value in each extend statement must exactly match
the LOGSOURCE value in the corresponding watchlist row above.
This is the join key — if they do not match exactly the join fails.

Use this structure:

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
    | extend LogSource = "[must match LOGSOURCE in watchlist row exactly]"
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
OUTPUT TYPE 3 — CSV SUMMARY ROW
═══════════════════════════════════════

After the watchlist rows and function output one CSV row per source
using exactly these column headers in this exact order:

Table,LogSource,Category,Origin,Transport,Description,Purpose,SLA,DataConnector,DCRName,DCEName,FunctionName,SLS,MonitoringFrequency,Notes,Tier,Verified,ActionItems

Rules for CSV output:
- Wrap any field containing commas in double quotes
- Description and Purpose should be concise for CSV readability
- Verified = No for all rows — engineer confirms
- ActionItems = any follow up needed — empty if none
- Output all rows for this table together before moving on

═══════════════════════════════════════
IMPORTANT RULES
═══════════════════════════════════════

1. One watchlist row per unique source — not one per query.
   Multiple queries sharing the same source = one row.

2. LogSource in watchlist row and in function extend must match exactly.

3. If a query uses search * or union * with no table filter:
   LOGSOURCE: All Tables — workspace wide
   FUNCTIONNAME: None
   No sub-function.

4. Include sources found in breakout results even if no query covers
   them — flag in NOTES for engineer to decide whether to track.

5. DEPENDENCYTYPE helps the engineer understand why this source matters:
   Detection = a rule queries it
   Report = a scheduled report queries it
   Workbook = a workbook tab queries it
   Capability = UEBA, TI, Fusion depends on it
   Multiple = more than one of the above

6. Use all inputs together — table name, all queries, breakout results
   — to infer description and purpose accurately.

7. If uncertain about any field say so in NOTES.

Confirm you understand all three output types before I provide data.
```

---

Wait for confirmation before proceeding.

---

### Step 6 — Work Through Tables One by One

Use your table-first reference from Step 2 as your guide. Work through each table in order.

**For each table:**

1. Open your text editor
2. Write the table name at the top
3. Add all detections for this table — name and full KQL
4. Add any report queries that reference this table — report name and KQL
5. Add any workbook queries that reference this table — tab name and KQL
6. For shared tables add the breakout results:
```
ENVIRONMENT BREAKOUT RESULTS:
ResourceType = AZUREFIREWALLS — Categories: ApplicationRule, NetworkRule, DnsProxy, ThreatIntelLog
ResourceType = VAULTS — Categories: AuditEvent
```
7. Copy everything and paste to the AI

**After receiving output for each table:**
1. Copy the CSV rows into your sources investigation spreadsheet immediately
2. Paste the KQL function into `functions-in-progress.kql` immediately
3. Review the watchlist rows — override any fields you know better than the AI
4. Clear the text editor and move to the next table

Do not let output accumulate. Transfer as you go.

---

### Step 7 — Incorporate Client Questionnaire Responses

After receiving the completed questionnaire from the client review their responses and add any sources not already in the spreadsheet.

For each new source identified from client responses add a row with:
- DEPENDENCYTYPE = Compliance or Client Request
- TIER = 1 if compliance mandated, 2 if client requested
- NOTES = the specific compliance requirement or client reason

Use this prompt for client-identified sources with no KQL dependency:

```
I need to document a data source that a client has identified as
important for compliance or business reasons. No detection currently
queries it.

Table: [table name]
Source: [what the client told us about it]
Reason: [compliance requirement / retention need / business critical]

Please produce a watchlist row block and CSV row using the same
format as before. For PURPOSE base it on what this source is
generally used for in security monitoring and the client reason.
DEPENDENCYTYPE = Compliance or Client Request as appropriate.
Do not write a sub-function unless it is a shared table.
```

---

### Step 8 — Handle Tables With No KQL Dependency

Some tables may appear in `[mssname]-tables` with no detection, report, or workbook query referencing them. Before skipping them ask:

- Does a capability depend on this table?
- Did the client questionnaire mention it?
- Is there a compliance or retention requirement?
- Is there meaningful investigation value even without a current detection?

If yes to any of these — add a row. If no to all — leave it out. Document your decision in the `[mssname]-tables` Notes field so future engineers know it was considered.

---

## Phase 3 — Finalize and Deploy

### Step 9 — Final Review of Spreadsheet

Check every row before upload:

- [ ] Table matches exactly what appears in `[mssname]-tables`
- [ ] LogSource is plain English and descriptive
- [ ] Category, Origin, Transport are from approved lists
- [ ] Description is 2-3 sentences — specific not generic
- [ ] Purpose is one sentence written for a business audience
- [ ] DEPENDENCYTYPE explains why this source is tracked
- [ ] SLA is True or False — nothing blank
- [ ] MonitoringFrequency is set — nothing blank
- [ ] Tier is assigned — 1 / 2 / 3 / 4
- [ ] No Unknown placeholders in DCRName or DCEName — fill from environment or write N/A
- [ ] FunctionName matches the actual function name you will deploy
- [ ] ActionItems reviewed and resolved or documented
- [ ] Verified = Yes for any row you have confirmed against the live environment

---

### Step 10 — Review and Finalize Functions

Before deploying each function:

- [ ] Function name follows the convention: Get[TableName]Health()
- [ ] Returns exactly four columns: Table, LogSource, LastSeen, Status
- [ ] LogSource values in the function exactly match LogSource values in the watchlist
- [ ] Filter conditions verified against live environment breakout results
- [ ] All expected sub-sources included — add any missing ones

**Verify filter conditions:**
```kql
// Example for AzureDiagnostics
AzureDiagnostics
| where TimeGenerated > ago(30d)
| summarize Count = count() by ResourceType, Category
| order by Count desc
```

---

### Step 11 — Upload Watchlist and Deploy Functions

**Prepare the upload copy:**
1. Make a copy of the investigation spreadsheet
2. Remove these columns: Tier, Verified, ActionItems, DEPENDENCYTYPE
3. Save as `[ClientName]-sources-[YYYY-MM-DD].xlsx`
4. Upload to `02-Clients/[ClientName]/04-Watchlists/Current/`

**Upload to Sentinel:**
1. Export the upload copy as CSV
2. Sentinel → Watchlists → Create new
3. Name: `[mssname]-sources`
4. Upload CSV
5. Set SearchKey to `Table`
6. Validate:
```kql
_GetWatchlist('[mssname]-sources')
| summarize TotalRows = count(), Tables = dcount(Table)
```

**Deploy functions:**
1. In Sentinel navigate to Logs
2. Run each function KQL — verify it executes without errors
3. Click Save → Save as function
4. Name exactly as defined — `GetAzureDiagnosticsHealth` etc
5. Save final KQL files to `01-Program/Functions/` in SharePoint if generic enough to reuse

---

## The Result — What You Have When This Is Done

At the end of this process you have:

**A complete sources watchlist** — every tracked data source documented with full context, monitoring configuration, and dependency information. Uploaded to Sentinel and immediately driving the master source detection.

**A set of KQL sub-functions** — one per shared table, all following the same output contract, all ready for GetSharedTableHealth() and GetEnrichedInventory() to call.

**An investigation spreadsheet** — the full working reference including tier assignments, verification status, and action items. Stays in SharePoint as the audit trail.

**Automated monitoring** — the master source detection immediately starts monitoring everything with MonitoringFrequency set. If any source goes silent beyond its threshold an incident fires. No manual checking required.

**A live audit deck** — the workbook now has everything it needs to show the client their data source health, detection coverage, and SLA status. No manual preparation before reviews.

One process. Two systems. Both working automatically from the day the watchlist is uploaded.

---

*This document lives in 06-watchlist-management and is maintained in the sentinel-mssp-playbook repository. Update when the process changes, the prompt is refined, new field types are added, or new input sources are identified.*