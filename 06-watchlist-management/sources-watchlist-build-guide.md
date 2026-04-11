# Building the Sources Watchlist — Process and AI Guidance

> **Purpose:** This document guides an engineer through the process of building the `[mssname]-sources` watchlist using detection data and AI assistance. It includes the exact prompt to use with any LLM, the input format, and what to do with the output.
>
> **Who uses this:** Any engineer building or updating the sources watchlist. Does not require access to any specific AI tool — works with any capable LLM including internal company tools.
>
> **What you produce:** A completed sources watchlist spreadsheet ready for upload to Sentinel.
>
> **Last Updated:** April 2026

---

## Why We Do It This Way

The sources watchlist is the single place where all monitoring lives. Every table gets at least one row here — single source tables get one row describing the whole table as a source, shared tables get one row per sub-source. This means the sources watchlist drives everything — health monitoring, silent detections, workbook data sources tab, audit deck.

The sources watchlist needs rich information about each data source — what it is, where it comes from, what it detects, how often data arrives, and what connector or configuration is involved. This information exists in two places:

**Detection KQL** — tells us exactly what data a detection needs to function. If a detection filters on `DeviceVendor == "Palo Alto Networks"` we know Palo Alto logs are a required source. The KQL is objective truth.

**Table name context** — tells us the broader picture. Combined with the detection logic an AI can infer the data connector, the typical ingestion frequency, the security purpose, and the appropriate monitoring threshold.

By feeding both together we get richer output than either alone would produce.

---

## Prerequisites

Before starting have these ready:

1. **Your DetectionCatalog CSV** — `[mssname]-detections` spreadsheet. This is your source list. Work through it table by table.

2. **Access to a capable LLM** — any large language model works. Use your company internal LLM for anything involving client-specific information.

3. **A text editor** — Notepad, VS Code, TextEdit — anything you can paste into and copy from. This is your staging area before sending to the AI.

4. **Your sources watchlist spreadsheet** — open and ready with the correct column headers:
```
Table, LogSource, Category, Origin, Transport, Description,
Purpose, SLA, DataConnector, DCRName, DCEName, FunctionName,
SLS, MonitoringFrequency, Notes
```

---

## Step 1 — Train the AI

Before feeding any data paste this prompt into the AI and wait for it to confirm it understands. Use this exact prompt every time you start a new session — the AI has no memory between sessions.

---

**TRAINING PROMPT — paste this first:**

```
I am building a data source tracking watchlist for Microsoft Sentinel.
I will be providing you with information about analytics rules (detections)
that run in Sentinel. For each table I provide you will receive:

- The table name at the top
- One or more detection names followed by their KQL queries
- Multiple detections for the same table will all be grouped under
  that table name

Your job is to analyze the table name and all detection KQL queries
together and produce one output block per unique log source or data
source category that the detections depend on.

For each log source produce output in this exact format:

---
TABLE: [exact table name]
LOGSOURCE: [plain English name of the specific source within this table]
CATEGORY: [choose one: Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other]
ORIGIN: [what system or product generates this data]
TRANSPORT: [how it gets into Sentinel — e.g. Microsoft Connector / AMA Agent / CEF Syslog / XDR Connector / Logic App / Diagnostic Setting / REST API]
DESCRIPTION: [2-3 sentences describing what this data source is. Be specific — use the table name and detection context together. Do not be generic or robotic. Infer from both what the data actually represents in this environment.]
PURPOSE: [one sentence — what attack or risk does this data help detect based on what the detections are doing with it]
DATACONNECTOR: [name of the Sentinel data connector most likely used to configure this source. If unsure make your best guess based on the table name and origin.]
MONITORINGFREQUENCY: [your best guess for how often this data source typically sends data. Choose from: 1h / 5h / 15h / 24h / 48h. Base this on the nature of the data — authentication logs are continuous, daily batch exports are 24h, weekly feeds are 48h.]
CONFIDENCE: [High / Medium / Low — how confident are you in the MonitoringFrequency recommendation]
DATAINFO: [2-3 sentences about the type of data this source provides — what fields are typically present, what events it captures, anything an engineer should know about working with this data in Sentinel]
NOTES: [anything else worth knowing — common misconfigurations, gotchas, things to verify in the environment]
---

Important rules:
- If multiple detections use the same table but filter on different
  sub-sources produce one output block per sub-source
- Use the detection KQL filter conditions to identify sub-sources
  within shared tables like CommonSecurityLog, AzureDiagnostics, Syslog
- If a detection uses search * or union * with no table filter write
  LOGSOURCE as: All Tables — workspace wide detection
- Do not invent information you are not confident about — use NOTES
  to flag uncertainty
- Write descriptions naturally — you are helping an engineer understand
  what they are working with, not filling in a form

Confirm you understand this format before I provide any detections.
```

---

Wait for the AI to confirm. If it does not confirm clearly paste the prompt again and ask it to confirm.

---

## Step 2 — Prepare Your Input

Group your detections by table from the DetectionCatalog. For each table:

1. Open a text editor
2. Write the table name at the very top on its own line
3. For each detection that uses this table add:
   - Detection name on its own line
   - The full KQL query directly below it
   - A blank line between detections
4. If multiple detections use the same table include all of them under the same table name — only write the table name once at the top

**Input format example:**
```
AzureDiagnostics

OC12345 - Azure Firewall Threat Intel Match
AzureDiagnostics
| where ResourceType == "AZUREFIREWALLS"
| where Category == "AzureFirewallThreatIntelLog"
| where action_s == "Deny"
| join kind=inner (
    _GetWatchlist('critical-assets')
) on $left.msg_s == $right.IPAddress

OC12346 - Key Vault Secret Access Anomaly
AzureDiagnostics
| where ResourceType == "VAULTS"
| where Category == "AuditEvent"
| where OperationName == "SecretGet"
| where ResultType != "Success"
```

**Why include all detections for a table:**
Two detections may use the same table but filter on completely different sub-sources. Sending both together lets the AI identify that AzureDiagnostics has two distinct sources — Azure Firewall and Key Vault — each needing its own row in the sources watchlist. If you only sent one detection you might miss the other source entirely.

---

## Step 3 — Send to AI and Collect Output

Copy your text editor content and paste it into the AI after the training prompt. The AI will produce one output block per unique log source it identifies.

**Work through tables one at a time.** Do not batch multiple tables in one paste — it creates confusion about which source belongs to which table.

After receiving output for one table copy it into a staging area — a separate text editor or a notes section — before moving to the next table.

---

## Step 4 — Transfer to Spreadsheet

After completing all tables transfer the AI output into your sources watchlist spreadsheet. For each output block:

1. Create a new row in the spreadsheet
2. Map each AI output field to the corresponding spreadsheet column
3. Fields the AI cannot know — leave blank for now:
   - **SLA** — you decide this based on SLS commitment
   - **DCRName** — only you know the actual DCR name in this environment
   - **DCEName** — same
   - **FunctionName** — filled in when sub-functions are built
   - **SLS** — filled in when silent detections are created

4. Review every row — the AI makes educated guesses. Verify:
   - LOGSOURCE makes sense for this environment
   - CATEGORY is correct
   - DATACONNECTOR matches what is actually configured
   - MONITORINGFREQUENCY is appropriate — override if you know better

---

## Step 5 — Handle Sources Without Detections

Not every source in the watchlist will have a detection. Some sources are tracked for:
- Compliance and retention requirements
- Capability dependencies — UEBA, Threat Intelligence, Fusion ML
- Client-specific requests

For these sources you still need to fill in the watchlist. Use the AI to help:

**Prompt for sources without detections:**
```
I need to document a data source in Microsoft Sentinel that does not
currently have a custom detection but is being tracked for other reasons.

Table: [table name]
Source: [what you know about the source]
Reason for tracking: [compliance / UEBA dependency / client request / etc]

Please provide the same output format as before — LOGSOURCE, CATEGORY,
ORIGIN, TRANSPORT, DESCRIPTION, PURPOSE, DATACONNECTOR,
MONITORINGFREQUENCY, CONFIDENCE, DATAINFO, NOTES.

For PURPOSE — base it on what this data source is generally used for
in security monitoring even if no specific detection currently uses it.
```

---

## Step 6 — Client Input Sources

Some sources will only be known after the client provides their input. These come from:

- The client questionnaire — compliance requirements, audit needs, specific systems
- Client-specific integrations — Salesforce, Keeper, Zoom, custom tools
- Business context — what systems are most critical to their operations

Leave these rows blank in the spreadsheet with a note — `[Pending client input]`. Fill them in after the client questionnaire is returned.

---

## Step 7 — Final Review Before Upload

Before uploading as a watchlist review every row against this checklist:

- [ ] Table matches exactly what appears in `[mssname]-tables`
- [ ] LogSource is plain English and descriptive
- [ ] Category is from the approved list
- [ ] Purpose is one sentence written for a business audience
- [ ] SLA is set — True or False, nothing blank
- [ ] MonitoringFrequency is set — None or a time value, nothing blank
- [ ] No [Manual] placeholders remaining
- [ ] No AI-generated text that is clearly wrong or generic

---

## Naming and Upload

**Watchlist name:**
```
[mssname]-sources
```

**Upload steps:**
1. Export the spreadsheet as CSV
2. Navigate to Sentinel → Watchlists → Create new
3. Name: `[mssname]-sources`
4. Upload CSV
5. Set SearchKey to `Table`
6. Validate:

```kql
_GetWatchlist('[mssname]-sources')
| summarize TotalRows = count()
```

---

## Tips for Getting the Best AI Output

**Be specific in your table name.** Just writing `AzureDiagnostics` is less useful than `AzureDiagnostics — client environment has Azure Firewall and Key Vault` if you know that context.

**Include all detections for a table.** The more detection context the AI has the better it can identify distinct sub-sources and infer purpose.

**If the output looks generic ask it to be more specific.** Tell it what you know about the environment — "this is a financial services client" or "the firewall is a Palo Alto PA-3200" — and ask it to revise.

**Trust your own knowledge over the AI for MONITORINGFREQUENCY.** The AI makes an educated guess. If you know a source sends data every hour set it to 1h regardless of what the AI suggests. The AI confidence rating tells you when to trust it and when to verify.

**Batch size.** One table at a time keeps output clean and traceable. Resist the temptation to paste multiple tables at once.

---

*This document is maintained in sentinel-mssp-playbook under 06-watchlist-management. Update it when the process changes, when the AI prompt is refined, or when new field types are added to the sources watchlist.*