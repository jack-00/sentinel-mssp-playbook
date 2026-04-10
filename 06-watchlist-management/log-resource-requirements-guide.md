# Log Source Requirements — Process and Registry

> **Purpose:** This document defines the process for determining which log sources within shared tables need to be tracked, why they need to be tracked, and how that decision gets recorded in the LogSourceRequirements watchlist. It covers the full end to end process from requirements gathering through to populating the LogSourceRegistry watchlist that drives the workbook and functions.
>
> **Why this exists:** Knowing that a log source exists in the workspace is not enough. You need to know why it matters. A log source being tracked without a documented reason is maintenance burden with no clear value. A log source that matters but is not tracked is a silent gap waiting to cause a problem. This process ensures every tracking decision is deliberate, justified, and documented.
>
> **How it fits the program:** The LogSourceRequirements watchlist sits between the discovery process and the LogSourceRegistry watchlist. It captures the why. LogSourceRegistry captures the what. Together they give any engineer a complete answer to two critical questions — what are we tracking and why does it matter.
>
> **Last Updated:** April 2026

---

## Why Requirements Matter

Every log source tracked in this program creates work. It needs a row in LogSourceRegistry. It may need a silent detection. It needs to be monitored for health. It shows up in the workbook and the audit deck. If a source goes inactive someone needs to investigate and fix it.

That is a real operational cost. Tracking a log source that nothing depends on wastes time and creates noise. Not tracking a log source that a detection depends on creates blind spots.

The requirements process answers the question that should be asked before any log source is added to LogSourceRegistry:

**Why does this specific log source need to be tracked?**

If you cannot answer that question the log source should not be in LogSourceRegistry. If you can answer it the answer should be documented so anyone coming after you understands the decision.

This also protects against the most common drift scenario — a requirement changes, a detection is retired, a compliance framework no longer applies, but the log source stays in the watchlist because nobody remembers why it was added. With requirements documented you can periodically review them and confidently remove sources that no longer have justification.

---

## The Four Requirement Types

Every log source tracked in this program must have at least one requirement from this list. If a log source does not meet any of these criteria it should not be tracked.

---

### Requirement Type 1 — Detection Dependency

A custom detection queries this specific log source. If this source goes inactive the detection goes blind. The client loses security coverage without knowing it.

This is the most important requirement type. Detection dependencies are objective — they come directly from the detection KQL. There is no ambiguity. Either the detection queries this source or it does not.

**How to identify:** Analyze every OC-series detection KQL. Extract every table queried and every filter condition that narrows to a specific sub-source within a shared table. The detection's dependency on that sub-source is the requirement.

**Examples:**
- Detection OC12345 queries AzureDiagnostics where ResourceType == AZUREFIREWALLS and Category == AzureFirewallThreatIntelLog → Azure Firewall Threat Intel Log has a Detection Dependency requirement
- Detection OC67890 queries CommonSecurityLog where DeviceVendor == Palo Alto Networks → Palo Alto Networks in CommonSecurityLog has a Detection Dependency requirement

---

### Requirement Type 2 — Capability Dependency

A Sentinel capability — UEBA, Threat Intelligence matching, Fusion ML, SOC Optimization — depends on this log source being healthy. Capability dependencies are different from detection dependencies because capabilities degrade silently rather than going completely blind. A capability can continue running against stale or incomplete data without any obvious error.

**How to identify:** Review the capability dependency chains documented in 04-capabilities/. Any source listed as a dependency for an active capability has a Capability Dependency requirement.

**Examples:**
- UEBA depends on SignInLogs being healthy → SignInLogs has a Capability Dependency requirement
- Threat Intelligence matching depends on ThreatIntelIndicators having fresh active indicators → ThreatIntelIndicators has a Capability Dependency requirement

---

### Requirement Type 3 — Compliance and Retention

A regulatory framework, legal obligation, or contractual requirement mandates that this log source be collected and retained for a defined period. This requirement exists independent of whether any detection currently uses the data.

**How to identify:** Complete the client questionnaire below. The client's compliance team or legal team must confirm which frameworks apply and what they require.

**Common frameworks and what they typically require:**

| Framework | Common Log Requirements |
|---|---|
| PCI-DSS | Firewall logs, authentication logs, system logs — 12 months retention, 3 months immediately available |
| HIPAA | Access logs for systems containing PHI — 6 years retention |
| SOC 2 | Logical access logs, change management logs — typically 1 year |
| ISO 27001 | Security event logs — retention period defined by risk assessment |
| GDPR | Access logs for personal data systems — duration of processing plus defined period |
| NIST 800-171 | Audit logs for CUI systems — 3 years |

**Examples:**
- Client has PCI-DSS scope — firewall logs covering cardholder data environment have a Compliance requirement regardless of whether detections use them
- Client has HIPAA obligations — access logs for systems containing PHI have a Compliance requirement

---

### Requirement Type 4 — Report Dependency

A scheduled report that the client receives references data from this log source. If the source goes inactive the report produces incomplete or missing data. The client may not notice immediately but the report loses integrity.

**How to identify:** Review all scheduled reports configured for this client. Identify which log sources each report reads from. Any report that references a specific sub-source within a shared table creates a Report Dependency requirement for that source.

**Examples:**
- Monthly firewall traffic summary report reads from AzureDiagnostics NSG flow logs → NSG Flow Events has a Report Dependency requirement
- Weekly privileged access report reads from AuditLogs → AuditLogs has a Report Dependency requirement

---

## The Client Questionnaire

Complete this questionnaire with the client before building LogSourceRegistry. The answers feed directly into requirement identification. Share this with the client security team and have them complete it with input from their legal and compliance team where needed.

---

**Section 1 — Compliance Frameworks**

1. Which compliance frameworks apply to your organization?
   - PCI-DSS — if yes, which systems are in scope
   - HIPAA — if yes, which systems contain PHI
   - SOC 2 — if yes, Type 1 or Type 2, which trust principles
   - ISO 27001 — if yes, certification scope
   - GDPR — if yes, which systems process personal data
   - NIST 800-171 — if yes, which systems handle CUI
   - Other — specify

2. What log retention periods are required by your compliance obligations?

3. Are there any specific log types explicitly required by your compliance framework?

4. Do you have a current compliance audit scheduled? If yes when?

---

**Section 2 — Audit and Legal Requirements**

1. Have you ever needed to produce log evidence for a legal matter, regulatory audit, or internal investigation?

2. If yes — what types of logs were required?

3. Are there any systems where log collection is required by contract — for example a cyber insurance policy or a client contract?

4. What is your current log retention period and is it meeting your obligations?

---

**Section 3 — Reports**

1. What scheduled security reports do you currently receive?

2. For each report — what data does it show and where does that data come from?

3. Are there any reports you wish you had but do not currently receive?

4. Who are the stakeholders who receive security reports?

---

**Section 4 — Business Context**

1. Are there any systems or data sources that are particularly sensitive or high value from a business perspective?

2. Have you had any security incidents in the past where specific log data was critical to the investigation?

3. Are there any upcoming changes to your environment — new systems, new cloud services, new vendors — that should be factored into log collection planning?

4. Are there any systems where you have had visibility gaps in the past that you want to address?

---

## The AI-Assisted Detection Analysis Process

The fastest and most accurate way to identify detection dependencies is to use AI to analyze every detection KQL. This process extracts every table and sub-source filter condition from the detection logic systematically.

---

### Step 1 — Prepare the Detection List

Export all OC-series detections from Sentinel with their KQL queries. This can be done using the same process used to build the DetectionCatalog — filter analytics rules by OC prefix, copy name, ID, and KQL for each detection.

---

### Step 2 — Train the AI

Use this prompt to establish context before analyzing detections:

```
I am analyzing Microsoft Sentinel analytic rule KQL queries to identify
log source dependencies for a data source tracking system.

For each detection I provide please analyze the KQL and identify:

1. Every Log Analytics table the query reads from — list exact table names

2. For shared tables that can have multiple sub-sources — identify the
   specific filter conditions that narrow to a particular sub-source.
   Shared tables to watch for:
   - AzureDiagnostics — look for ResourceType and Category filters
   - CommonSecurityLog — look for DeviceVendor and DeviceProduct filters
   - Syslog — look for HostName or ProcessName filters
   - ThreatIntelIndicators — look for SourceSystem filters
   - ASimNetworkSessionLogs — look for EventVendor and EventProduct
   - ASimAuditEventLogs — look for EventVendor and EventProduct

3. Every watchlist the query references via _GetWatchlist()

Output format for each detection — one row per table dependency:

DetectionID | DetectionName | Table | SubSourceFilter | Watchlists

Where SubSourceFilter is the specific filter condition in plain English
— for example "ResourceType = AZUREFIREWALLS and Category = AzureFirewallThreatIntelLog"
or "DeviceVendor = Palo Alto Networks" or "None" if no sub-source filter.

Output only the rows — no headers, no explanation.
I will add headers myself.

Confirm you understand before I provide detections.
```

---

### Step 3 — Analyze Detections in Batches

Feed detections in batches of 20-30. For each batch provide:
- Detection name
- Detection ID
- Full KQL query

The AI outputs one row per table dependency per detection. A detection that queries three tables produces three rows.

---

### Step 4 — Consolidate Results

After all detections are processed consolidate the results:

1. Group by Table and SubSourceFilter
2. For each unique combination list all detections that depend on it
3. This gives you the complete dependency map — every sub-source and which detections rely on it

This consolidated output directly populates the RequirementType and RequirementSource fields in the LogSourceRequirements watchlist.

---

## The LogSourceRequirements Watchlist

This watchlist captures the why behind every log source tracking decision. It is the requirements layer that sits between the discovery process and LogSourceRegistry.

**Watchlist name:**
```
[CompanyName]-LogSourceRequirements
```

**Fields:**

| Field | What Goes Here |
|---|---|
| Table | Exact table name |
| LogSource | The specific log source or sub-source being justified |
| RequirementType | Detection / Capability / Compliance / Report |
| RequirementSource | What specifically creates this requirement — detection ID, capability name, framework name, or report name |
| RequirementDetail | Plain English explanation of why this source needs to be tracked |
| Priority | High / Medium / Low — based on requirement type and business impact |
| ReviewDate | When this requirement should be reviewed — annually for compliance, when detection is retired for detection dependencies |
| Notes | Any additional context |

**Priority guidance:**

| Requirement Type | Default Priority |
|---|---|
| Detection Dependency — High severity detection | High |
| Detection Dependency — Medium severity detection | Medium |
| Capability Dependency — UEBA, TI, Fusion | High |
| Compliance — regulatory mandate | High |
| Report Dependency | Medium |
| Detection Dependency — Low severity | Low |

---

### Example Rows

```
Table:             AzureDiagnostics
LogSource:         Azure Firewall — Threat Intel Log
RequirementType:   Detection
RequirementSource: OC12345 - TI Match Outbound Connection
RequirementDetail: Detection queries AzureDiagnostics filtered to
                   AZUREFIREWALLS and AzureFirewallThreatIntelLog.
                   Going inactive means this detection goes blind.
Priority:          High
ReviewDate:        Review if OC12345 is retired
Notes:             Client enabled Azure Firewall TI feed April 2026

Table:             AzureDiagnostics
LogSource:         Azure Key Vault — Audit Events
RequirementType:   Compliance
RequirementSource: PCI-DSS Requirement 10.2
RequirementDetail: PCI-DSS requires logging of all access to audit
                   trails including Key Vault where secrets related
                   to cardholder data environment are stored.
Priority:          High
ReviewDate:        Annual compliance review
Notes:             Confirmed with client compliance team April 2026

Table:             CommonSecurityLog
LogSource:         Palo Alto Networks — PAN-OS
RequirementType:   Detection
RequirementSource: OC67890 - Outbound C2 Detection
RequirementDetail: Detection queries CommonSecurityLog filtered to
                   Palo Alto Networks. Primary network perimeter
                   detection source.
Priority:          High
ReviewDate:        Review if OC67890 is retired
Notes:             Primary firewall for client network perimeter
```

---

## How LogSourceRequirements Feeds LogSourceRegistry

Once LogSourceRequirements is complete it directly drives what goes into LogSourceRegistry. The process is:

1. Every unique Table + LogSource combination in LogSourceRequirements gets a row in LogSourceRegistry
2. The Priority field in LogSourceRequirements informs the SilentDet tier assignment in LogSourceRegistry — High priority sources get Tier 1 silent detections
3. The RequirementDetail informs the Purpose field in LogSourceRegistry
4. Sources with no entry in LogSourceRequirements should not be in LogSourceRegistry

This creates a clean traceable chain:

```
Requirements gathering
        ↓
LogSourceRequirements watchlist — why we track it
        ↓
LogSourceRegistry watchlist — what it is and how to monitor it
        ↓
KQL sub-functions — health check per shared table
        ↓
GetEnrichedInventory() — complete unified view
        ↓
Workbook and audit deck
```

---

## Periodic Review Process

Requirements change over time. Detections get retired. Compliance frameworks change. Reports are discontinued. The LogSourceRequirements watchlist must be reviewed periodically to ensure tracked sources still have valid justification.

**Review triggers:**
- A detection is retired → review all sources where RequirementSource = that detection ID
- A compliance audit is completed → review all Compliance requirement rows
- A report is discontinued → review all Report requirement rows
- Annual review → check all rows with ReviewDate in the past

**The review question for each row:**
Is the requirement in RequirementSource still active? If yes — keep the row. If no — assess whether another requirement justifies keeping the source. If no other requirement exists — remove from LogSourceRequirements and LogSourceRegistry.

This prevents watchlist bloat and ensures monitoring effort is always justified.

---

## What Needs to Be Updated in the Repo

Adding this process affects several existing documents. Update these alongside adding this guide:

**ARCHITECTURE.md** — add LogSourceRequirements to the Data Layer watchlist list. Update the system diagram to show it sitting between the requirements gathering process and LogSourceRegistry.

**PROGRAM-MAP.md** — add LogSourceRequirements as a new planned watchlist. Update session notes.

**data-source-audit-runbook.md** — add a step before building LogSourceRegistry that references this requirements process. Engineers should complete requirements gathering before populating LogSourceRegistry.

**dev-notes.md** — add a note that LogSourceRegistry rows must have a corresponding LogSourceRequirements entry. The join between them is how you validate that everything being tracked has a documented reason.

**silent-detection-standards.md** — reference LogSourceRequirements Priority field as an input to tier assignment. High priority = Tier 1, Medium = Tier 2, Low = Tier 3.

---

*This document is maintained in sentinel-mssp-playbook under 06-watchlist-management. Update it when new requirement types are identified, when the client questionnaire is refined based on real engagements, or when the AI analysis process changes.*