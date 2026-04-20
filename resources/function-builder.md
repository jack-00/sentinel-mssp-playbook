# MSSP Sentinel KQL Function Builder

## Overview

This document describes the architecture, design decisions, and build process for a watchlist-driven health monitoring and silent log detection system built on top of Microsoft Sentinel. It is intended as a living reference for engineers onboarding to the system or contributing to its development.

Think of this system as a living audit deck. Rather than static documentation that goes stale the moment it is written, everything here is self-validating. The watchlists define what *should* exist. The functions check what *actually* exists. The workbook and detections consume both. The documentation and the operational system are the same thing.

---

## The Problem This Solves

Microsoft Sentinel uses data connectors as a way to onboard log sources. However, data connectors are essentially deployment wizards — they help you get data flowing, but they are not a reliable way to monitor whether data is still flowing. A connector can show as connected while a source has been silent for days.

The standard approach of checking health at the table level is also incomplete. Many tables in Sentinel act as shared containers. A table like `CommonSecurityLog` might receive data from a Palo Alto firewall, a Fortinet device, a Cisco ASA, and a Sophos endpoint product — all in the same table. Checking that the table has data tells you nothing about whether each individual source is healthy.

This system solves both problems by:

- Moving health monitoring from the connector level to the **source level**
- Using **KQL functions** to check each individual data source within a table
- Storing the baseline definition of what should exist in **watchlists** that serve as the source of truth
- Driving **silent log detections**, **workbook displays**, and **deployment automation** all from the same foundation

---

## The Watchlists

The system is built around five watchlists. Together they form the control plane — the authoritative record of everything about a client environment. Each watchlist is named using the client's workspace alias as a prefix, referred to here as `[mssname]`.

### `[mssname]-client`
Client profile. Everything an engineer needs to understand the environment at a glance.

`ClientName, Industry, CompanyURL, OnboardingDate, Bio, TenantID, SubscriptionID, WorkspaceID, WorkspaceName, ResourceGroup, M365License, DefenderLicense, SentinelCommitmentTier, EntraIDTier, MicrosoftSupportPlan, DailyCapGB, Notes`

### `[mssname]-tables`
Table pipe registry. One row per table in the workspace. Used to track known tables, flag unknown ones, and drive the Tables page of the audit deck workbook.

`Table, Category, Details, Vetted, Notes`

### `[mssname]-sources`
Data source registry. The most important watchlist in the system. One row per tracked data source. Every function, detection, and workbook display for the Data Sources layer is driven from this watchlist.

`Table, LogSource, Category, Origin, Transport, Description, Purpose, SLA, DataConnector, DCRName, DCEName, FunctionName, SLS, Lookback, Schedule, SentinelCapabilities, Notes`

**Key fields explained:**

- `LogSource` — the unique name for this data source. This is the join key used everywhere. It must match exactly — character for character — between the watchlist and the function output.
- `FunctionName` — the name of the KQL function that covers this table. Used by deployment scripts to know which function to call.
- `Lookback` — how long a source can be silent before it is considered a problem. This drives both the KQL detection logic and the analytic rule lookback window. Examples: `1h`, `5h`, `24h`, `48h`, `None`.
- `Schedule` — how often the analytic rule runs. Calculated automatically as half of Lookback to ensure overlap between runs and prevent detection gaps. Can be overridden manually per source.
- `SLS` — whether a silent log detection exists for this source.

### `[mssname]-detections`
Content ledger. Every analytic rule, report query, workbook query, and client requirement tracked in one place.

`RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description`

### `[mssname]-dependencies`
Credential tracker. Every API key, client secret, and OAuth token with expiry and rotation information. Linked to sources so that when a source goes silent, an expiring credential can be identified as a possible cause.

`DependencyName, ServiceName, DependencyType, LinkedSource, ExpiryDate, RotationOwner, RotationContact, LastRotated, AppRegistrationName, Notes`

---

## The Plan

### How It All Fits Together

The system has three layers that all work together.

**Layer 1 — The Watchlists**
These are the baseline. They define what tables exist, what data sources live within them, what detections are in place, and what credentials keep everything running. When you onboard a new client you fill in these watchlists. From that point forward the watchlists drive everything else automatically.

**Layer 2 — The Functions**
One KQL function per table. Each function queries the table in real time and returns the health status of every data source tracked within it. The function also joins the watchlist at runtime to pull metadata like `Description` and `Lookback` so the output is immediately useful without needing to open anything else.

**Layer 3 — The Outputs**
Three outputs consume the function layer:
1. The **audit deck workbook** — a visual dashboard showing all data sources and whether they are logging
2. **Per-source analytic rules** — silent log detections that fire when a source goes quiet beyond its defined threshold
3. **Direct function runs** — a tech can call any function at any time to instantly see the health of all data sources for that table

---

## The Functions

### What a Function Is

A KQL function is a saved, reusable query with a name. Instead of writing the same query every time you want to check a table, you type the function name and it runs. Think of it as a command that encapsulates logic you want to reuse.

### The Output Contract

Every function in this system returns exactly seven columns. This contract is immutable. Every function, for every table, always returns the same shape.

```
Table | LogSource | Status | Lookback | LastSeen | DaysSince | Description
```

- `Table` — the Log Analytics table name
- `LogSource` — the name of the data source, must match the watchlist exactly
- `Status` — Active, Review, Inactive, or No Data
- `Lookback` — the monitoring threshold for this source, pulled from the watchlist at runtime
- `LastSeen` — the timestamp of the most recent log from this source
- `DaysSince` — the number of days since the last log was seen
- `Description` — a plain-language description of the source, pulled from the watchlist at runtime

**Status logic:**

| Status | Condition |
|--------|-----------|
| Active | Data seen within the last 24 hours |
| Review | Data seen within the last 7 days |
| Inactive | Data older than 7 days |
| No Data | No records found for this source |

### Two Types of Tables

**Single source tables** have one data source and always will. The function queries the table directly with no filtering and returns one row.

**Shared tables** act as containers for multiple different data sources. Examples include `CommonSecurityLog` which can hold data from many different firewall and endpoint vendors, or `AzureDiagnostics` which holds logs from many different Azure resource types. For shared tables, the function contains one block per data source, each filtered to identify that specific source, and the results are stacked together using the `union` operator.

### The Union Pattern

`union` is the KQL operator that stacks multiple query results into a single output. In our functions, each data source within a shared table is its own mini-query returning one row. Union combines all those rows into the final output.

```
// Source 1 block
let source1 = TableName
| where DeviceVendor == "Palo Alto Networks"
| summarize LastSeen = max(TimeGenerated)
| extend Table = "CommonSecurityLog", LogSource = "Palo Alto Firewall"
...

// Source 2 block
let source2 = TableName
| where DeviceVendor == "Fortinet"
| summarize LastSeen = max(TimeGenerated)
| extend Table = "CommonSecurityLog", LogSource = "Fortinet Firewall"
...

// Stack them together
union source1, source2
```

The key requirement is that every block returns the same columns in the same order. That is why the output contract is enforced strictly.

### Naming Convention

Function names follow this pattern exactly:

```
Get + TableName + Health()
```

- Use the exact table name as it appears in Log Analytics
- PascalCase throughout with no underscores or spaces
- Always include parentheses at the end

**Examples:**
- `GetSignInLogsHealth()`
- `GetCommonSecurityLogHealth()`
- `GetAzureDiagnosticsHealth()`
- `GetSecurityEventHealth()`

### How to Create a Function in Sentinel

1. Open Microsoft Sentinel and navigate to **Logs**
2. Write your KQL query and verify the output looks correct
3. Click **Save** and choose **Save as function**
4. Enter the function name following the naming convention above
5. Optionally add a description and category for organisation
6. Click **Save**

The function is now available in the workspace and can be called by name in any query, analytic rule, or workbook.

### Quality Checklist

Before saving any function, verify:

- [ ] Returns exactly the seven required columns in the correct order
- [ ] Every LogSource value matches the corresponding entry in the watchlist character for character
- [ ] Status logic is correct — Active for under 24h, Review for under 7 days, Inactive for 7 days or older, No Data for null
- [ ] Watchlist join brings back correct Description and Lookback values
- [ ] Running the function raw returns one row per expected source with no duplicates

---

## Building Functions with AI Assistance

Each table has a corresponding text file produced during the investigation phase. This file contains everything needed to write the function:

- Table name and function name
- Schema from `getschema`
- Source breakdown showing what sub-sources exist and how to identify them
- Consumer KQL from existing detections and workbook queries
- Analysis notes including filter conditions per source

To use AI to write a function, provide the briefing in two parts.

---

### Prompt: Function Builder Briefing

Use the following prompt to bring an AI assistant up to speed before building a function. Provide two follow-up messages — the first with the watchlist data for the relevant sources, the second with the full table text file.

---

**Post 1 of 2 — Watchlist data for this table**

> I am building KQL health functions for a Microsoft Sentinel MSSP program. Each function checks whether a table and its individual data sources are actively logging. Before I give you the table file I want to share the architecture so you understand exactly what to build.
>
> Every function returns exactly these seven columns in this order:
> `Table, LogSource, Status, Lookback, LastSeen, DaysSince, Description`
>
> Status values: Active = data within last 24h | Review = data within last 7 days | Inactive = data older than 7 days | No Data = no records found.
>
> The function joins the `[mssname]-sources` watchlist at runtime to pull `Lookback` and `Description`. The `LogSource` value in the function must match the watchlist exactly — it is the join key.
>
> For shared tables with multiple sources, use one block per source filtered by the appropriate field (DeviceVendor and DeviceProduct for CommonSecurityLog, ResourceType and Category for AzureDiagnostics, HostName for Syslog, SourceSystem for ThreatIntelIndicators). Stack all blocks using `union`.
>
> Naming convention: `Get + TableName + Health()` in PascalCase.
>
> Here are the relevant rows from the sources watchlist for this table:
> [paste watchlist rows here]

**Post 2 of 2 — Table investigation file**

> Here is the full investigation file for this table. Use the schema, source breakdown, filter conditions, and detection KQL to write the function following the contract above. Confirm the sources you identified, write the function, verify the filter conditions match the source breakdown, and flag any edge cases.
>
> [paste table text file contents here]

---

## Silent Log Detections

### How They Work

Each row in the `[mssname]-sources` watchlist with a `Lookback` value other than `None` gets its own analytic rule. The rule calls the table function, filters to the specific `LogSource` for that row, and fires an alert if that source is no longer Active.

The basic detection pattern looks like this:

```kql
GetCommonSecurityLogHealth()
| where LogSource == "Palo Alto Firewall"
| where Status != "Active"
```

If the function returns any results after that filter, something is wrong and an alert fires. The alert contains everything a triage engineer needs — the source name, its last seen timestamp, how many days it has been silent, and a plain-language description of what the source is.

### Avoiding Detection Gaps

A detection gap occurs when a rule runs so infrequently that a silent source falls between two lookback windows and is never caught. To prevent this, the rule schedule should always be shorter than the lookback window so that consecutive runs overlap.

The rule of thumb used in this system is:

> **Schedule = Lookback ÷ 2**

If a source has a Lookback of 24 hours, the rule runs every 12 hours. If Lookback is 1 hour, the rule runs every 30 minutes. This guarantees at least two chances to catch any silence event within the tolerance window.

### Analytic Rule Settings

Each analytic rule is configured from the watchlist row for that source:

| Rule Setting | Source |
|---|---|
| Query | Function call filtered to this LogSource |
| Lookback window | `Lookback` field from watchlist |
| Run frequency | `Schedule` field from watchlist (Lookback ÷ 2) |
| Alert threshold | Greater than 0 results |
| Event grouping | One alert per source row |

### Sources with Lookback = None

Sources with `Lookback` set to `None` are intentionally excluded from all silent log detections. This is used during onboarding before a source has been vetted, or for sources that are known to be intermittent and do not require automated monitoring.

---

## Programmatic Deployment

Because every piece of information needed to configure an analytic rule exists in the `[mssname]-sources` watchlist, the entire detection layer can be deployed automatically using a script.

The script reads the watchlist row by row. For each row where `Lookback` is not `None` and `Vetted` is true, it creates an analytic rule using:

- `FunctionName` to build the query
- `LogSource` to add the filter
- `Lookback` to set the lookback window
- `Schedule` to set the run frequency
- `Description` to populate the alert details
- `LogSource` and `Table` to name the rule

This means onboarding a new client requires only filling in the watchlists. Running the deployment script against a completed watchlist produces all detections automatically with no manual rule authoring required.

---

## The Audit Deck Workbook

The workbook is organised as a tab-style interface with three pages.

**Page 1 — Client**
Pulls from `[mssname]-client`. Static profile display showing everything about the client environment at a glance.

**Page 2 — Tables**
Pulls from `[mssname]-tables` joined with live table activity. Shows: `Table, Status, Category, Vetted, LastSeen, DaysSince, Details, Notes`

**Page 3 — Data Sources**
The most important page and the reason the functions exist. Pulls metadata from `[mssname]-sources` and joins live health output from the functions. Displays every tracked data source, its current status, when it was last seen, and all relevant context. This page cannot be completed until all functions are built.

The Data Sources page is the only place where all function outputs are joined together. Per-source analytic rules are always isolated — a problem with one detection never affects any other.

---

## Onboarding a New Client

1. Fill in all five watchlists with the client's environment details
2. Use the Tables page of the workbook as an observation deck while data begins flowing
3. As each source is confirmed healthy, mark it as `Vetted` in the watchlist
4. Run the deployment script to generate all analytic rules for vetted sources
5. Enable detections as sources are confirmed

New sources that appear in shared tables will not be automatically detected. During periodic audits, a validation query should be run to surface any `LogSource` values appearing in function output that have no matching row in the watchlist. Any new source found should be reviewed with the client, added to the watchlist, and processed through the standard onboarding checklist before detections are enabled.

---

*This document should be updated whenever the watchlist schema changes, a new function pattern is introduced, or the deployment process evolves.*