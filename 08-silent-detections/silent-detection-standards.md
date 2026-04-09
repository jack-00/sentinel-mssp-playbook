# Silent Detection Standards and Management

> **Purpose:** This document captures the methodology, tiers, decision framework, and future vision for silent log source detection management across all managed Sentinel environments. Silent detections are the immune system of the environment — they catch failures before clients do. This document standardizes how we think about them, how we prioritize them, and where we are going with automation.
>
> **Current State:** Silent detections are created and managed manually. This document establishes the standards that will govern both the current manual process and the future automated system.
>
> **Last Updated:** April 2026

---

## What Is a Silent Detection

A silent detection is an analytics rule that monitors a specific log source to confirm it is actively sending data. It does not detect attacks. It detects silence — the absence of expected data.

When a log source goes silent:
- Detections that depend on it go blind
- Capabilities that feed from it degrade
- Compliance obligations may be violated
- The client's security posture deteriorates without anyone knowing

The most dangerous gap in any Sentinel environment is not a missing detection rule. It is a missing data source that nobody noticed went offline. Silent detections are the safety net that catches this before a client does.

---

## The Core Principle

**Not every log source needs a silent detection.**

Having a silent detection for every single table and every single log source would create alert fatigue, maintenance burden, and noise. The goal is not maximum coverage — it is intelligent coverage. Silent detections should exist where their absence would create a meaningful undetected gap.

The framework below provides a consistent, repeatable way to make that decision for every log source in every client environment.

---

## The Silent Detection Tier Framework

Every log source is assigned a tier that determines what level of monitoring it receives. The tier is based on answering one core question:

**What happens if this source goes silent and nobody notices for a week?**

---

### Tier 1 — Required. No Exceptions.

Silent detection must exist, must be enabled, and must be healthy. A Tier 1 source going silent without detection is an operational failure.

**Criteria — a source is Tier 1 if any of the following are true:**
- An AB-series detection actively queries this source — losing it means a detection goes blind
- The source feeds a capability — UEBA, Threat Intelligence, Fusion ML
- The source is part of the incident pipeline — SecurityIncident, SecurityAlert
- A client compliance obligation requires this data to be continuously collected
- Loss of this source creates a meaningful gap in MITRE ATT&CK coverage that has no redundant coverage from another source

**Alert behavior:** Tier 1 silent detection fires → High severity incident → Teams notification to SOC immediately

**Examples:**
- SignInLogs — password spray and impossible travel detections go blind
- SecurityEvent — authentication and lateral movement detections go blind
- CommonSecurityLog per vendor — perimeter detection goes blind
- ThreatIntelIndicators — all TI matching stops
- SecurityIncident — pipeline breaks, no tickets reach ServiceNow
- BehaviorAnalytics — UEBA output stops updating
- Any source with an AB-series detection mapped to it

---

### Tier 2 — Recommended. Strong Justification.

Silent detection should exist. A Tier 2 source going silent is important to know about but the immediate security impact is lower than Tier 1. Worth having a silent detection but the urgency when it fires is medium not critical.

**Criteria — a source is Tier 2 if any of the following are true:**
- Source has high detection value but no current AB-series detection depends on it yet
- Source is used for threat hunting and investigation context — losing it impairs IR capability
- Source feeds compliance reporting or audit requirements
- Source is a custom integration — Logic App, API polling — that can break silently without obvious indication
- Source is expected by the client or referenced in their compliance documentation

**Alert behavior:** Tier 2 silent detection fires → Medium severity incident → Teams notification to SOC within standard review window

**Examples:**
- AzureActivity — no immediate detection gap but critical for incident response
- OfficeActivity — compliance and investigation value
- Key Vault logs — high value, detections may be planned but not yet written
- Salesforce tables — custom integrations that break without obvious indication
- Keeper, Zoom, Drupal, SecureW2 — same reason
- AADProvisioningLogs — compliance and audit value
- StorageBlobLogs — investigation and exfiltration detection value

---

### Tier 3 — Health Dashboard Only. No Dedicated Silent Detection.

No analytics rule needed. Health is monitored via the workbook Tab 3 and the snapshot pipeline. A source going silent here is caught by the workbook not by a firing incident.

**Criteria — a source is Tier 3 if any of the following are true:**
- Source is a capability output not a raw input — its health is covered by Tier 1 silent detections on the source tables that feed it
- Source is an ASIM normalized table — source table coverage already exists at Tier 1
- Source is an operational meta table — its health is monitored by platform health checks
- Source has low security value and no detections depend on it
- Source going silent is interesting to know about but does not require immediate action

**Monitoring method:** Workbook Tab 3 shows health status. Snapshot comparison catches changes between audits. No incident created.

**Examples:**
- BehaviorAnalytics — UEBA source table coverage covers this
- IdentityInfo — same
- Anomalies — same
- UserPeerAnalytics — same
- ASimNetworkSessionLogs, ASimAuditEventLogs, ASimWebSessionLogs — source tables covered by Tier 1
- Usage — operational meta table
- Operation — operational meta table
- Perf — low security value
- AppMetrics, AppPerformanceCounters — low security value
- DeviceInfo, DeviceNetworkInfo — context tables, investigation value only
- AlertInfo, AlertEvidence — pipeline covered by SecurityIncident at Tier 1

---

### Tier 4 — Document Only. No Monitoring Required.

Worth documenting in the audit spreadsheet but no silent detection and no health monitoring needed. These sources being present or absent does not meaningfully affect security detection, compliance, or investigation capability.

**Criteria — a source is Tier 4 if all of the following are true:**
- No AB-series detections depend on it
- No compliance requirement mandates it
- No meaningful threat hunting value
- Going silent creates no detection gap

**Examples:**
- AppTraces — depends entirely on what application sends this and whether detections use it
- Salesforce_LoginHistory_CL — historical reference, Salesforce_LoginEvent covers active monitoring
- StorageQueueLogs, StorageTableLogs — low detection value unless specific detections exist
- AzureMetrics — low security value unless specific anomaly detections are written against it

**Important note:** A source can move from Tier 4 to a higher tier when a detection is written against it or when a compliance requirement is identified. Tier assignments are not permanent — they are reviewed when the detection library grows or client requirements change.

---

## The Decision Tree

When assigning a tier to any log source work through these questions in order. Stop at the first YES.

```
Q1: Does an AB-series detection actively query this source?
    YES → Tier 1

Q2: Does this source feed a capability (UEBA, Threat Intel, Fusion)?
    YES → Tier 1

Q3: Is this source part of the incident pipeline?
    (SecurityIncident, SecurityAlert, BehaviorAnalytics output)
    YES → Tier 1

Q4: Does a client compliance obligation require continuous collection
    of this source?
    YES → Tier 1 (compliance-driven) or Tier 2 depending on urgency

Q5: Is this a custom integration that could break silently?
    (Logic App polling, REST API, custom pipeline)
    YES → Tier 2

Q6: Does this source have high investigation or threat hunting value
    even without a current detection?
    YES → Tier 2

Q7: Is this an ASIM table or capability output table?
    YES → Tier 3

Q8: Is this an operational meta table?
    (Usage, Operation, Perf, AppMetrics)
    YES → Tier 3

Q9: None of the above apply
    → Tier 4
```

---

## Compliance and Legal Elevation

The tier framework above represents the baseline recommendation based on detection and operational need. Client-specific compliance requirements can elevate any source to Tier 1 regardless of whether a detection depends on it.

If a client has regulatory obligations — HIPAA, PCI-DSS, SOC 2, GDPR — specific log sources may be legally required to be continuously collected and monitored. A source that is Tier 3 by default becomes Tier 1 for a compliant client if that source is required by their regulatory framework.

This is why the license-entitlement-review and the future regulatory-requirements watchlist are important. When those documents are complete for a client they should be reviewed alongside this tier framework to identify any compliance-driven elevations.

---

## Current State — Manual Process

Today silent detections are created and managed manually. The process is:

1. Engineer identifies a log source during the data source audit
2. Engineer assigns a tier using the decision tree above
3. For Tier 1 and Tier 2 sources — engineer creates an AB-series silent detection in Sentinel
4. Detection ID and name are documented in the SilentDet column of the spreadsheet
5. Detection health is verified via SentinelHealth during Phase 3 of the audit

**The problems with the manual process:**
- No centralized catalog of which silent detections exist and what they cover
- No consistent naming convention for silent detections
- No automated way to verify that every Tier 1 source has a detection
- No way to know when a new source appears that needs a detection
- Silent detections are not version controlled or documented outside the spreadsheet
- If an engineer leaves, the institutional knowledge of what each detection does goes with them

---

## Future Vision — Automated Silent Detection Management

The goal is a fully managed, version-controlled, automated silent detection system. Here is where we are going:

### Phase 1 — Catalog and Standardize (Next)

Build a silent detection catalog in the repo. One file per silent detection following a standard template. Every detection documented with:
- Detection ID and name
- What source it monitors
- What table it queries
- What field and threshold it uses
- What tier it is
- What alert it creates when it fires
- Date created and last reviewed

The catalog lives in `08-silent-detections/` and becomes the source of truth for all silent detections across all clients.

### Phase 2 — Pre-Canned Detection Library

Build a library of pre-written silent detections for all Tier 1 and Tier 2 sources we commonly see across environments. These are ready-made KQL rules that can be deployed to any new client workspace during onboarding.

Common pre-canned silent detections:
- SignInLogs silence
- SecurityEvent silence
- CommonSecurityLog silence per vendor
- ThreatIntelIndicators silence and freshness check
- Heartbeat silence per machine
- OfficeActivity silence
- AzureActivity silence
- Custom integration silence — generic template for Logic App sources

The library is stored in `08-silent-detections/library/` as KQL files that can be deployed via ARM template, Bicep, or the Sentinel API.

### Phase 3 — Automated Coverage Verification

A scheduled analytics rule or Logic App that:
1. Queries the data source watchlist for all Tier 1 and Tier 2 sources
2. Cross-references against the SentinelHealth table for active AB-series rules
3. Identifies any Tier 1 or Tier 2 source that has no corresponding healthy silent detection
4. Creates an incident and Teams notification — "Missing silent detection — [Source] — Tier [N]"

This makes the gap between what should have a silent detection and what actually has one visible and actionable automatically.

### Phase 4 — Snapshot-Driven New Source Detection

When the DataSourceAudit_CL snapshot pipeline is running:
1. New tables appearing between snapshots are automatically assigned Flag status
2. An incident is created — "New log source detected — [Table] — [Source]"
3. Engineer investigates, assigns a tier, creates a silent detection if Tier 1 or 2
4. Watchlist is updated, Flag status resolved

This closes the loop — no new source can appear without being noticed, classified, and monitored.

### Phase 5 — Multi-Tenant Content Distribution

Microsoft confirmed in February 2026 that multi-tenant content distribution is available from the Defender portal. Once the silent detection library is built:
1. Maintain the canonical set of silent detections centrally
2. Distribute updates to all client workspaces from one place
3. New pre-canned detection added to the library — deployed to all clients automatically
4. No more manually deploying the same rule to ten different workspaces

---

## Silent Detection Naming Convention

All silent detections follow the AB-series naming convention with a consistent structure:

```
AB##### - SLD - [Source Description]
```

Examples:
- `AB00012 - SLD - SignInLogs Silence`
- `AB00013 - SLD - CommonSecurityLog Palo Alto Silence`
- `AB00014 - SLD - ThreatIntelIndicators Freshness`
- `AB00015 - SLD - Heartbeat DC01 Silence`

The `SLD` tag in the name makes silent detections immediately identifiable in the analytics rules list and in SentinelHealth queries:

```kql
// Find all silent detections
SentinelHealth
| where SentinelResourceType == "Analytics Rule"
| where SentinelResourceName contains "SLD"
| summarize LastStatus = arg_max(TimeGenerated, Status)
    by SentinelResourceName
```

---

## Silent Detection Template

Every silent detection follows this structure:

```kql
// ============================================================
// AB##### - SLD - [Source Description]
// Tier: [1/2]
// Source: [Table name]
// Purpose: Fires if [source] stops sending data for [threshold]
// Created: [Date]
// ============================================================
let threshold = [time period]; // e.g. 4h for Tier 1, 24h for Tier 2
let hasData = [TABLE_NAME]
| where TimeGenerated > ago(threshold)
| summarize Count = count()
| project HasData = Count > 0;
hasData
| where HasData == false
| project
    TimeGenerated = now(),
    AlertName = "AB##### - SLD - [Source Description]",
    Description = "[Source] has not sent data in the last [threshold]. [Detection impact statement.]",
    Severity = "[High/Medium]",
    Tier = "[1/2]"
```

**Threshold guidelines by tier:**
- Tier 1 Identity sources — 4 hours
- Tier 1 Endpoint sources — 12 hours
- Tier 1 Network and Firewall sources — 6 hours
- Tier 1 Pipeline sources (SecurityIncident) — 1 hour
- Tier 2 sources — 24 hours

---

## Review and Maintenance

Silent detections are reviewed:
- During every data source audit — verify all Tier 1 and Tier 2 sources have a healthy detection
- When a new AB-series detection is written — check if the source tables it depends on have Tier 1 silent detections
- When a client changes their data sources — new sources get tiered and monitored, decommissioned sources get marked Decom
- Quarterly — review all Tier 2 and Tier 3 assignments to see if anything should be elevated based on new detections written

---

*This document is maintained in sentinel-mssp-playbook under 08-silent-detections. It governs the silent detection methodology across all managed environments. Update it when the tier framework evolves, when new source types are identified, or when the automation phases are completed.*