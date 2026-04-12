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

## How Silent Detections Work in This Program

We use two master silent detections plus optional custom detections per source. This is the variable-driven approach — the detection code never changes, only the watchlist configuration changes.

**Master Source Monitor**
One detection covers all sources. It reads every row in `[mssname]-sources` where MonitoringFrequency is not None. It calls the KQL sub-functions to get LastSeen per source. It fires per source that exceeds its own MonitoringFrequency threshold.

Adding a new source to monitor means updating `[mssname]-sources`. Zero detection changes needed.

**Custom SLS Detections**
For sources needing specific logic, specific alerting behavior, or specific thresholds beyond what the master handles. The OC##### identifier is stored in the SLS field in `[mssname]-sources`. The master detection is the floor — custom SLS detections are the ceiling.

**The SLS field in `[mssname]-sources`:**
- `OC#####` — a custom silent detection exists and is assigned to this source
- `Missing` — no custom detection yet, source is covered by the master detection only

---

## The Silent Detection Tier Framework

Every log source is assigned a tier that determines what level of monitoring it receives. The tier is based on answering one core question:

**What happens if this source goes silent and nobody notices for a week?**

---

### Tier 1 — Required. No Exceptions.

Silent detection must exist, must be enabled, and must be healthy. A Tier 1 source going silent without detection is an operational failure.

**Criteria — a source is Tier 1 if any of the following are true:**
- An OC-series detection actively queries this source — losing it means a detection goes blind
- The source feeds a capability — UEBA, Threat Intelligence, Fusion ML
- The source is part of the incident pipeline — SecurityIncident, SecurityAlert
- A client compliance obligation requires this data to be continuously collected
- Loss of this source creates a meaningful gap in MITRE ATT&CK coverage with no redundant coverage

**Alert behavior:** Tier 1 fires → High severity incident → Teams notification immediately

**MonitoringFrequency guidance for Tier 1:**
- Identity sources — 1h
- Network and Firewall sources — 5h
- Endpoint sources — 15h
- Pipeline sources — 1h
- Threat Intelligence sources — 24h

**Examples:**
- SignInLogs — password spray and impossible travel detections go blind
- SecurityEvent — authentication and lateral movement detections go blind
- CommonSecurityLog per vendor — perimeter detection goes blind
- ThreatIntelIndicators — all TI matching stops
- SecurityIncident — pipeline breaks, no tickets reach ServiceNow
- BehaviorAnalytics — UEBA output stops updating
- Any source with an OC-series detection mapped to it

---

### Tier 2 — Recommended. Strong Justification.

Silent detection should exist. A Tier 2 source going silent is important to know about but immediate security impact is lower than Tier 1.

**Criteria — a source is Tier 2 if any of the following are true:**
- Source has high detection value but no current OC-series detection depends on it yet
- Source is used for threat hunting and investigation context
- Source feeds compliance reporting or audit requirements
- Source is a custom integration — Logic App, API polling — that can break silently
- Source is expected by the client or referenced in their compliance documentation

**Alert behavior:** Tier 2 fires → Medium severity incident → Teams notification within standard review window

**MonitoringFrequency guidance for Tier 2:**
- Most Tier 2 sources — 24h
- Less frequent batch sources — 48h

**Examples:**
- AzureActivity — no immediate detection gap but critical for incident response
- OfficeActivity — compliance and investigation value
- Key Vault logs — high value, detections may be planned but not yet written
- Salesforce tables — custom integrations that break without obvious indication
- Keeper, Zoom, Drupal, SecureW2 — same reason
- StorageBlobLogs — investigation and exfiltration detection value

---

### Tier 3 — Health Dashboard Only. No Dedicated Silent Detection.

No analytics rule needed. Health is monitored via the workbook. Set MonitoringFrequency = None in `[mssname]-sources`.

**Criteria — a source is Tier 3 if any of the following are true:**
- Source is a capability output — health covered by Tier 1 on the source tables that feed it
- Source is an ASIM normalized table — source table coverage exists at Tier 1
- Source is an operational meta table — health monitored by platform health checks
- Source going silent is interesting to know about but does not require immediate action

**Examples:**
- BehaviorAnalytics — UEBA source table coverage covers this
- IdentityInfo — same
- Anomalies — same
- ASimNetworkSessionLogs, ASimAuditEventLogs, ASimWebSessionLogs — source tables covered
- Usage, Operation, Perf — operational meta tables

---

### Tier 4 — Document Only. No Monitoring Required.

Worth documenting in `[mssname]-sources` but no silent detection and no health monitoring. Set MonitoringFrequency = None and SLS = Missing.

**Criteria — all of the following are true:**
- No OC-series detections depend on it
- No compliance requirement mandates it
- No meaningful threat hunting value
- Going silent creates no detection gap

**Important:** A source moves from Tier 4 to a higher tier when a detection is written against it or a compliance requirement is identified. Tier assignments are not permanent.

---

## The Decision Tree

Work through these questions in order. Stop at the first YES.

```
Q1: Does an OC-series detection actively query this source?
    YES → Tier 1

Q2: Does this source feed a capability (UEBA, Threat Intel, Fusion)?
    YES → Tier 1

Q3: Is this source part of the incident pipeline?
    YES → Tier 1

Q4: Does a client compliance obligation require continuous collection?
    YES → Tier 1 or Tier 2 depending on urgency

Q5: Is this a custom integration that could break silently?
    YES → Tier 2

Q6: Does this source have high investigation or threat hunting value
    even without a current detection?
    YES → Tier 2

Q7: Is this an ASIM table or capability output table?
    YES → Tier 3

Q8: Is this an operational meta table?
    YES → Tier 3

Q9: None of the above apply
    → Tier 4
```

---

## Compliance and Legal Elevation

Client-specific compliance requirements can elevate any source to Tier 1 regardless of whether a detection depends on it. HIPAA, PCI-DSS, SOC 2, GDPR — specific log sources may be legally required to be continuously monitored. A source that is Tier 3 by default becomes Tier 1 for a compliant client if required by their regulatory framework.

Review compliance requirements alongside this tier framework during every client onboarding and license entitlement review.

---

## Setting MonitoringFrequency and SLS in `[mssname]-sources`

After assigning a tier set these two fields in the sources watchlist:

| Tier | MonitoringFrequency | SLS |
|---|---|---|
| Tier 1 | Set per guidance above — 1h, 5h, 15h, or 24h | OC##### if custom detection exists — Missing if master covers it |
| Tier 2 | 24h or 48h | OC##### if custom detection exists — Missing if master covers it |
| Tier 3 | None | Missing |
| Tier 4 | None | Missing |

The master source detection automatically monitors everything with MonitoringFrequency set. You only need to create a custom SLS detection when the source needs specific logic or alerting beyond what the master provides.

---

## Current State — Manual Process

Today silent detections are created and managed manually. The process is:

1. Engineer identifies a log source during the data source audit
2. Engineer assigns a tier using the decision tree above
3. Set MonitoringFrequency in `[mssname]-sources` — the master detection handles monitoring automatically
4. For sources needing custom logic — create an OC-series SLD detection and store the ID in the SLS field
5. Verify detection health via SentinelHealth during Phase 3 of the audit

---

## Silent Detection Naming Convention

All custom silent detections follow the OC-series naming convention with a consistent structure:

```
OC##### - SLD - [Source Description]
```

Examples:
- `OC00012 - SLD - SignInLogs Silence`
- `OC00013 - SLD - CommonSecurityLog Palo Alto Silence`
- `OC00014 - SLD - ThreatIntelIndicators Freshness`
- `OC00015 - SLD - Heartbeat DC01 Silence`

The `SLD` tag makes silent detections immediately identifiable in SentinelHealth queries:

```kql
// Find all silent detections
SentinelHealth
| where SentinelResourceType == "Analytics Rule"
| where SentinelResourceName contains "SLD"
| summarize LastStatus = arg_max(TimeGenerated, Status)
    by SentinelResourceName
```

In `[mssname]-detections` silent detections have AlertClass = Operational.

---

## Custom Silent Detection Template

When a source needs a custom SLD beyond what the master covers use this template:

```kql
// ============================================================
// OC##### - SLD - [Source Description]
// Tier: [1/2]
// Source: [Table name] — [LogSource]
// Purpose: Fires if [source] stops sending data beyond threshold
// MonitoringFrequency in [mssname]-sources: [value]
// Created: [Date]
// ============================================================
let threshold = [time period]; // matches MonitoringFrequency in watchlist
let hasData = [TABLE_NAME]
| where TimeGenerated > ago(threshold)
| summarize Count = count()
| project HasData = Count > 0;
hasData
| where HasData == false
| project
    TimeGenerated = now(),
    AlertName = "OC##### - SLD - [Source Description]",
    Description = "[Source] has not sent data in the last [threshold]. [Detection impact statement.]",
    Severity = "[High/Medium]",
    Tier = "[1/2]"
```

---

## Future Vision — Automated Silent Detection Management

### Phase 1 — Catalog and Standardize (Next)
Build a silent detection catalog in the repo. One file per custom SLD detection. Every detection documented with ID, name, what source it monitors, tier, threshold, and alert behavior.

### Phase 2 — Pre-Canned Detection Library
Build a library of pre-written custom SLD detections for common Tier 1 and Tier 2 sources. Stored in `08-silent-detections/library/` as KQL files deployable via Sentinel API.

### Phase 3 — Automated Coverage Verification
A scheduled analytics rule that reads `[mssname]-sources` for all sources where SLS = Missing and MonitoringFrequency is set, cross-references against SentinelHealth, and flags any gaps.

### Phase 4 — Multi-Tenant Content Distribution
Maintain canonical SLD detections centrally. Distribute updates to all client workspaces from one place via the Defender portal multi-tenant content distribution capability.

---

## Review and Maintenance

Silent detections are reviewed:
- During every data source audit — verify MonitoringFrequency and SLS fields are correct for all sources
- When a new OC-series detection is written — check if source tables need Tier 1 coverage
- When a client changes their data sources — new sources get tiered, decommissioned sources updated
- Quarterly — review all Tier 2 and Tier 3 assignments to see if anything should be elevated

---

*This document is maintained in sentinel-mssp-playbook under 08-silent-detections. Update it when the tier framework evolves, when new source types are identified, or when the automation phases are completed.*