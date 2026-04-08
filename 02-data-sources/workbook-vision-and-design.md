# Client Security Operations Workbook — Vision and Design

> This document captures the vision, design, and intended experience of the Client Security Operations Workbook. It exists so that every engineer building or using this workbook understands not just what it does but why it was built this way and what the experience should feel like for both the engineer and the client.
>
> **Status:** Design phase — workbook not yet built
> **Lives in:** Microsoft Sentinel — one deployed per client workspace
> **Last Updated:** April 2026

---

## What This Workbook Is

The Client Security Operations Workbook is the operational centerpiece of the audit program. It is a live, always-current dashboard that runs inside Microsoft Sentinel and answers the most important questions about a client's security data environment automatically — without any manual preparation, report generation, or data gathering.

It is the thing you open during a client review and the story is already there.

It is also the thing your engineers check between reviews to catch problems before the client does.

---

## What Problem It Solves

Before this workbook existed, preparing for a client review meant:
- Manually checking connector status in the portal
- Running ad-hoc KQL queries and copying results somewhere
- Trying to remember what changed since last time
- Scrambling to explain why something broke or was missing
- Having no consistent way to show clients what they have, what is working, and what could be better

The workbook eliminates all of that. The data is always current. The story is always ready. The engineer's job in a client review shifts from researcher to narrator.

---

## The Foundation — How It Works

The workbook is powered by two data sources working together:

**1. Live workspace queries**
KQL queries run against the client's Log Analytics workspace in real time. These return current table health, log source status, detection health, and capability dependency status. This data is always current — it reflects the state of the environment right now.

**2. The data source watchlist**
A watchlist loaded into the client's Sentinel workspace that contains all the curated context that cannot be queried — plain English source names, vendor, category, purpose, collection method, silent detection mapping, and notes. This watchlist is populated once from the first-run audit export and maintained going forward.

The workbook joins these two sources. Live data provides the facts. The watchlist provides the context. Together they produce a complete, human-readable picture of the environment.

---

## The Six Tabs

---

### Tab 1 — Summary

**What it is:** The opening view. The first thing a client sees when the workbook is opened during a review. Designed to communicate overall health in under ten seconds without reading a single label.

**What it shows:**

A header row with the client name and the current date.

A row of summary metric cards:

| Card | Example | What It Means |
|---|---|---|
| Data Sources | 47 of 49 healthy | 47 log sources Active, 2 need attention |
| Capabilities | 4 of 4 active | All capabilities fully operational |
| Detections Active | 72 of 83 | 72 detections have healthy data, 11 are blind |
| Silent Detections | 49 of 49 healthy | Every log source has a working silent detection |
| Integrations | 6 of 7 valid | One API key needs attention |

Below the cards, a source group health table — one row per category. Each row shows the category name and an X of Y count for log sources within it. Any row that is not 100% is immediately visible.

| Category | Log Sources | Status |
|---|---|---|
| Identity | 8 of 8 | 🟢 |
| Endpoint | 12 of 12 | 🟢 |
| Firewall | 5 of 5 | 🟢 |
| Network | 3 of 4 | 🟡 |
| Email | 6 of 6 | 🟢 |
| Cloud Infrastructure | 9 of 9 | 🟢 |
| Threat Intelligence | 1 of 1 | 🟢 |
| Capabilities | 4 of 4 | 🟢 |

Clicking any row expands into the detail view for that category in Tab 2.

**What the client experiences:** They see immediately that the environment is mostly healthy. One thing needs attention. Before a single word is spoken they understand the shape of the conversation.

**Technical note:** Row expansion is achieved using workbook parameters and conditional visibility. Selecting a row sets a parameter value. A linked section below filters the Tab 2 table to that category. This is supported in Azure Monitor workbooks via dropdown parameters.

---

### Tab 2 — Data Sources

**What it is:** The full detail view. One row per log source. Every column visible. The working engineer view and the drill-down destination from Tab 1.

**What it shows:**

A filter bar at the top — filter by Status, Category, Vendor, Table. Filters update the table in real time.

A full table with all columns:

| Column | Source |
|---|---|
| Table | Query |
| Log Source | Query + Watchlist |
| Source | Watchlist |
| Vendor | Watchlist |
| Category | Watchlist |
| Status | Query |
| Last Seen | Query |
| Daily Vol | Query |
| Purpose | Watchlist |
| Collection | Watchlist |
| Silent Det | Watchlist |
| Det Status | Query |
| Detections | Watchlist |
| Watchlists | Watchlist |
| Notes | Watchlist |

Rows are color coded by status. Red rows are immediately visible. Yellow rows draw attention. Green rows recede into the background.

**What the client experiences:** If they ask "what data do you have on us?" — this is the answer. Every source, organized, labeled, with a plain English explanation of why each one matters.

**What the engineer uses it for:** Daily health checking between reviews. During a review, drilling into any flagged rows to explain what is happening and what the plan is.

---

### Tab 3 — Silent Detections

**What it is:** A dedicated view for silent log source detection health. Silent detections are the immune system of the environment — they verify that each log source is actively sending data. Every log source must have one. This tab makes it immediately visible if any are missing, disabled, or unhealthy.

**What it shows:**

A summary at the top — X of Y silent detections healthy.

A table with one row per log source:

| Column | What It Shows |
|---|---|
| Log Source | The source being monitored |
| Table | Which table it writes to |
| Silent Detection ID | The AB-series detection ID |
| Detection Name | Full detection name |
| Enabled | Yes / No |
| Health Status | From SentinelHealth table |
| Last Fired | When the detection last ran successfully |
| Coverage | What the detection monitors for |

Flagged rows:
- 🔴 Detection is disabled
- 🔴 Detection is unhealthy per SentinelHealth
- 🟠 Log source has no silent detection mapped — Missing coverage

**Why this tab matters:** A log source can go silent and nobody knows because the connector shows connected and the table still exists in the schema. The silent detection is what catches this automatically. If the silent detection itself is broken or missing — the safety net has a hole. This tab finds those holes.

**What the client experiences:** "We have a detection for every single data source that alerts us automatically if it stops sending data. This tab shows you that every one of those detections is healthy and working."

---

### Tab 4 — Capabilities

**What it is:** Health view for the workspace capabilities — features that depend on multiple data sources and integrations working together. If any dependency breaks, the capability degrades — often silently. This tab surfaces that degradation before it causes a missed detection.

**Current capabilities documented for every client:**

**UEBA — User and Entity Behavior Analytics**
Learns what normal looks like for every user and entity. Flags deviations. Depends on:
- SignInLogs → must be Active
- AuditLogs → must be Active
- SecurityEvents → must be Active
- MDI Sensors → must be deployed and reporting
- Entra ID sync → must be current
- BehaviorAnalytics output → must have recent data

**Custom Threat Feed**
Brings in known malicious indicators — IPs, domains, file hashes — and matches them against all ingested log data. Depends on:
- Feed connector (custom API — AWS function) → must be Active
- API key → must be valid and not expired
- ThreatIntelIndicators table → must have recent data
- Indicator freshness → must be updated within vendor's defined cadence
- Analytics rules referencing TI → must reference current table names (ThreatIntelIndicators / ThreatIntelObjects — not legacy ThreatIntelligenceIndicator)

**Fusion ML — Multi-Stage Attack Detection**
Correlates signals across multiple alert sources to detect multi-stage attacks that no single rule would catch. Depends on:
- Multiple alert sources connected and feeding the correlation engine
- XDR connector active and syncing
- Multi-stage incident correlation enabled in workspace settings

**SOC Optimization**
Microsoft's built-in recommendations engine that analyzes your environment and suggests improvements to detection coverage, data ingestion, and UEBA configuration. Depends on:
- Analytics rules active
- Tables have data
- Defender portal connected

**What each capability card shows:**
- Overall status — derived automatically from dependency health
- Dependency chain — each dependency with its own status indicator
- API key status where applicable — including expiry date
- Last verified date

**What the client experiences:** "These are the advanced features of your security platform. Each one depends on multiple things working correctly underneath it. This tab shows you that every dependency is healthy — or flags it immediately when something breaks."

A client who asks "what is UEBA?" gets the plain English answer: "It learns what normal looks like for every person in your organization. If your CFO's account suddenly logs in from a country they have never been to at 3am and starts downloading files — UEBA flags it as highly anomalous within minutes, even if the password was correct."

---

### Tab 5 — Insights

**What it is:** The historical and analytical view. Built from the DataSourceAudit_CL custom table that is populated by the snapshot Logic App on a weekly schedule. This tab answers questions about trends, changes, and patterns over time.

**What it shows:**

- Volume trend charts per source category — how much data each category is producing week over week and month over month
- Status change history — what changed between snapshots. A source that was Active last week and is Inactive this week is immediately visible
- New log sources — sources that appeared since the last snapshot, flagged 🟠 for classification
- Missing log sources — expected sources that disappeared, flagged 🔵
- Age of watchlist entries — entries that have not been verified in 90 or more days
- Cost analysis — high volume sources with no detections or watchlists referencing them. Data being ingested that nothing is using

**What the client experiences:** "This tab shows us the history of your environment. We can see exactly when something changed, how your data volumes trend over time, and whether anything new appeared that we need to classify."

The moment when you show a client the volume chart and point to the exact day a data source went silent — and then show them the Teams notification your team received at 3am that same night — that is when the relationship changes permanently.

**Important note:** This tab is Version 2. It requires the snapshot Logic App to be built and DataSourceAudit_CL to have at least two weeks of data before it is meaningful. Tab is planned in the workbook structure but the queries that power it depend on the automation pipeline being in place first.

---

### Tab 6 — Integrations

**What it is:** API key and credential health. A dedicated view for every integration credential that keeps the environment running. Expiring API keys are one of the most common causes of silent capability failure.

**What it shows:**

A table with one row per integration credential:

| Column | What It Shows |
|---|---|
| Key Name | Plain English name of the credential |
| Service | What service it authenticates to |
| What Depends On It | Capabilities, playbooks, or Logic Apps that use it |
| Expiry Date | When the credential expires |
| Days Until Expiry | Calculated automatically |
| Status | Valid / Expiring Soon / Expired / Unknown |
| Last Verified | When an engineer last confirmed it was working |
| Notes | Rotation process, owner, any context |

Status thresholds:
- 🟢 Valid — more than 60 days to expiry
- 🟡 Expiring Soon — 30 to 60 days
- 🔴 Critical — less than 30 days
- 🔴 Expired — past expiry date
- ⚫ Unknown — no expiry date documented

**Automation:** When any key reaches the Expiring Soon threshold a Teams notification fires — "Client X: [Key Name] expires in [N] days. Rotate before [date]. Affects: [what depends on it]."

**What the client experiences:** "Every credential that keeps your security tools connected is tracked here. We know when they expire and we rotate them before they cause a problem. If a key expired silently your threat feed would stop updating, your enrichment playbooks would stop working, and you would not know until an incident response failed."

---

## The Audit Experience — Running a Client Review

This is what it looks and feels like to use this workbook in a client review meeting.

### Before the Meeting
Open the workbook. Spend two minutes reviewing Tab 1. Note anything that is not green. That is your talking points list. No other preparation needed.

### Opening the Meeting
Share your screen. Open the workbook to Tab 1. Let the client see it for five seconds before you say anything. The visual does the work.

"This is a live view of your security data environment right now. Everything here is current as of this moment — not a report we prepared last night."

### Walking Through the Summary
Point to the metric cards. Call out the numbers. If everything is green — "Everything is healthy. Let me walk you through what that means." If something is not green — "We have one thing to discuss. Let me show you what it is and what we are doing about it."

Work through the source group table row by row. Keep it high level. Save the detail for the next tab.

### Drilling Into Detail
Click on any row in the summary table. Tab 2 loads filtered to that category. Now you are in the detail. Walk through any flagged rows. Explain what each one means in plain English using the Purpose column. Do not read the technical fields to the client — translate them.

For any Inactive or Review source: "This went quiet on [date]. Here is what we know about why. Here is what we are doing. Here is the timeline."

For any Flag source: "This is new since our last review. We have investigated it and here is what it is. We have added it to the watchlist."

### The Capabilities Conversation
Navigate to Tab 4. Walk through each capability card. For any capability that is fully green — explain what it does and what it is protecting against in plain English. This is not a status check — this is a value demonstration.

"Your UEBA has been running for six months now. It has built behavioral profiles for every user in your organization. Last month it flagged three accounts with unusual access patterns. Two were false positives we tuned out. One became an incident. Here is that story."

### The Silent Detections Moment
Navigate to Tab 3. Show the X of Y count. "We have a detection for every single data source that tells us automatically if it stops sending data. Every one of them is healthy right now. This is how we knew about the firewall issue before you did."

This is one of the most powerful moments in the review. It demonstrates that your monitoring is proactive and systematic — not reactive.

### The Detection Value Conversation
Navigate to Tab 7. This is where the ROI story lives.

"This tab shows you exactly which data sources are actively contributing to security incidents — and which ones are ingesting data that nothing is using yet. Over the last 30 days your identity logs contributed to 14 incidents, your endpoint data contributed to 9, and your threat intelligence feed flagged 3 known malicious IPs that showed up in your traffic. That is direct security value from data you are already paying to collect."

Then flip it. "These three sources here — they are ingesting data but produced zero incident contributions this month. That is either a cost optimization opportunity or a detection gap we need to close. We will talk about which it is."

That is a conversation no other MSSP is having with their clients. You are not just showing what you caught — you are showing that you understand the economics of their security program.

---

## What Makes This Different

Most MSSP client reviews are retrospective — here is what happened last month. This workbook makes reviews prospective — here is where we are right now, here is what changed, here is where we are going.

Most MSSP reporting is prepared manually — someone spends hours pulling data together before a meeting. This workbook requires zero preparation — the data is always current.

Most MSSP gap conversations are uncomfortable — "we don't have visibility into that." This workbook makes gap conversations productive — "here are eleven detections that would activate if you onboarded this data source. Here is what each one catches. Here is how we get there."

The workbook turns the audit from a report delivery into a strategic conversation. That is what builds long-term client relationships.

---

## Tab 7 — Detection Value

**What it is:** The ROI tab. Closes the loop between data being ingested and security value being produced. Shows which data sources are actively contributing to incidents and detections, and which are ingesting data that nothing is using. Powers two critical client conversations — proving value and identifying waste.

**What it shows:**

**Section 1 — 30-day incident contribution by data source**
Which log sources produced or contributed to incidents in the last 30 days. Sorted by incident count descending. Color coded — green for active contributors, yellow for low contribution, gray for zero contribution.

```kql
// 30-day incident summary by source product
SecurityIncident
| where TimeGenerated > ago(30d)
| where Status != "Closed" or ClosedTime > ago(30d)
| summarize
    TotalIncidents = count(),
    Critical = countif(Severity == "Critical"),
    High = countif(Severity == "High"),
    Medium = countif(Severity == "Medium"),
    FalsePositives = countif(Classification == "FalsePositive")
    by ProductName
| extend TruePositiveRate = round((1.0 - todouble(FalsePositives) / TotalIncidents) * 100, 1)
| order by TotalIncidents desc
```

**Section 2 — Top firing detections**
The AB-series and Content Hub detections that fired the most in the last 30 days. Shows detection name, total fires, true positive rate, and which data source feeds it. This is the "here is what we built and here is it working" section.

```kql
// Top firing detections — last 30 days
SecurityAlert
| where TimeGenerated > ago(30d)
| summarize
    TotalFired = count(),
    HighSeverity = countif(AlertSeverity == "High"),
    Confirmed = countif(Status == "Resolved")
    by AlertName, ProductName
| order by TotalFired desc
| take 20
```

**Section 3 — Zero contribution sources**
Data sources that ingested data in the last 30 days but contributed to zero incidents and zero detection fires. This is the waste and gap identification section. Each row is either a cost optimization candidate or an unwritten detection opportunity.

```kql
// Tables with ingestion but zero alert contribution — last 30 days
let active_tables = SecurityAlert
    | where TimeGenerated > ago(30d)
    | summarize by ProductName;
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize TotalGB = round(sum(Quantity) / 1000, 2) by DataType
| where TotalGB > 0
| join kind=leftanti (active_tables | project DataType = ProductName) on DataType
| order by TotalGB desc
```

**Section 4 — Detection value summary card**
A single summary card at the top of the tab showing:
- Total incidents this month
- Data sources actively contributing
- Data sources with zero contribution
- Estimated monthly ingestion cost for zero-contribution sources
- Detection true positive rate overall

**What the client experiences:**
This tab answers the question every client eventually asks — "what are we actually getting from all of this?" The answer is right there in front of them. Not anecdotal. Not approximate. Exact numbers from their own data.

And for the zero contribution sources — you are not presenting a problem. You are presenting a choice. "This source is costing approximately X per month and has not contributed to any detection this month. We can either reduce ingestion on it or we can write detections that use it. Here is what we could catch if we did the latter."

That is the detection library conversation happening naturally, driven by real data, at the right moment.

---

## Version Roadmap

**Version 1 — Build this first**
- Tab 1 Summary
- Tab 2 Data Sources
- Tab 3 Silent Detections
- Tab 4 Capabilities
- Tab 6 Integrations
- Tab 7 Detection Value — data is already available, queries are straightforward, client value is immediate

Tab 5 Insights requires the snapshot automation pipeline which is a separate build.

**Version 2 — After automation pipeline is built**
- Tab 5 Insights — historical trends, status change history, volume anomalies

**Version 3 — Future**
- Regulatory compliance section
- Watchlist health scoring
- Detection coverage score with MITRE heatmap integration
- Custom graph visualization for capability dependency chains

---

*This document is maintained in sentinel-mssp-playbook under 02-data-sources. Update it as the workbook design evolves. When the workbook is built, add a section documenting the actual JSON template location and deployment instructions.*