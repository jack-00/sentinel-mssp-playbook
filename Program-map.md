# Program Map — Full Scope and Future State

> This document captures the complete scope of the sentinel-mssp-playbook program as it stands today and where it is going. It includes the full repository tree, the status of every file, ideas in progress, future work items, known limitations, and items that need to be updated in existing documents. This is the living reference for the entire program. Update it whenever new ideas are locked in or new work is completed.
>
> **Last Updated:** April 2026 — Updated after session 2. All Layer 1 and Layer 2 core documents complete. README rewritten. ARCHITECTURE updated. Runbook fully rewritten. Workbook vision documented.

---

## Repository Tree — Full Intended Structure

Legend:
- ✅ Built and committed
- 🔨 In progress — being built now
- 📋 Planned — design locked, not yet written
- 💡 Future — concept captured, design not yet started
- ⚠️ Needs update — exists but requires revision

```
sentinel-mssp-playbook/
│
├── README.md                                          ✅ Complete — rewritten with full story and vision
├── ARCHITECTURE.md                                    ✅ Complete — updated taxonomy, six-tab workbook, capabilities
├── PROGRAM-MAP.md                                     ✅ This document
│
├── 01-baseline/
│   ├── README.md                                      📋 Planned
│   ├── architect-checklist.md                         ✅ Complete — updated with capabilities, TI table fix, Layer 2 bridge
│   ├── data-source-baseline.md                        📋 Planned — industry standard data source requirements
│   ├── connector-runbook.md                           📋 Planned
│   └── audit-log-enablement.md                        📋 Planned
│
├── 02-data-sources/
│   ├── README.md                                      📋 Planned
│   ├── data-source-audit-runbook.md                   ✅ Complete — full rewrite with log source auditing, seven statuses, breakout queries, pre-populated reference
│   ├── workbook-vision-and-design.md                  ✅ Complete — six-tab design, client meeting narrative, version roadmap
│   ├── table-reference-watchlist.md                   📋 Planned
│   └── kql-validation-queries.md                      📋 Planned
│
├── 03-detection-library/
│   ├── README.md                                      📋 Planned
│   ├── catalog-index.md                               📋 Planned
│   ├── detection-template.md                          📋 Planned
│   └── detections/
│       └── AB00000.md                                 📋 Planned — one file per detection
│
├── 04-capabilities/
│   ├── README.md                                      📋 Planned
│   ├── ueba.md                                        📋 Planned
│   ├── threat-feed.md                                 📋 Planned
│   ├── fusion-ml.md                                   📋 Planned
│   └── soc-optimization.md                            📋 Planned
│
├── 05-xdr-migration/
│   ├── README.md                                      📋 Planned
│   ├── pre-migration-checklist.md                     📋 Planned
│   ├── post-migration-checklist.md                    📋 Planned
│   ├── schema-changes.md                              📋 Planned
│   └── new-capabilities.md                            📋 Planned
│
├── 06-watchlist-management/
│   ├── README.md                                      📋 Planned
│   ├── watchlist-schemas.md                           📋 Planned
│   ├── watchlist-health-queries.md                    💡 Future
│   ├── client-verification-report.md                  💡 Future
│   └── auto-discovery-queries.md                      💡 Future
│
├── 07-integrations/
│   ├── README.md                                      📋 Planned
│   ├── api-key-inventory-schema.md                    📋 Planned
│   └── credential-health-queries.md                   💡 Future
│
├── 08-silent-detections/
│   ├── README.md                                      📋 Planned
│   ├── silent-detection-standards.md                  📋 Planned
│   └── heartbeat-detection-guide.md                   📋 Planned
│
├── 09-audit-deck/
│   ├── README.md                                      📋 Planned
│   ├── workbook-design.md                             📋 Planned
│   ├── deck-template.md                               📋 Planned
│   ├── metrics-and-kpis.md                            📋 Planned
│   └── gap-framing-guide.md                           📋 Planned
│
├── 10-automation-pipeline/
│   ├── README.md                                      💡 Future
│   ├── snapshot-logic-app.md                          💡 Future
│   ├── teams-notification-playbook.md                 💡 Future
│   ├── change-detection-rules.md                      💡 Future
│   └── watchlist-verification-report.md               💡 Future
│
├── 11-access-and-permissions/
│   ├── README.md                                      💡 Future
│   ├── lighthouse-delegation-guide.md                 💡 Future
│   ├── graph-api-exploration.md                       💡 Future
│   └── permission-expansion-request.md                💡 Future
│
├── 12-tagging-strategy/
│   ├── README.md                                      💡 Future
│   ├── azure-policy-tagging.md                        💡 Future
│   ├── ama-arc-auto-tagging.md                        💡 Future
│   └── untagged-asset-discovery.md                    💡 Future
│
└── resources/
    ├── mitre-mapping-reference.md                     📋 Planned
    ├── kql-snippets.md                                📋 Planned
    ├── glossary.md                                    📋 Planned
    └── taxonomy.md                                    📋 Planned — see below
```

---

## Workspace Taxonomy

> The taxonomy is the classification system for everything that exists inside a Microsoft Sentinel workspace. Understanding these categories is what separates a surface-level audit from a complete one. Every document in this repo uses these terms consistently.

| Category | What It Is | Examples |
|---|---|---|
| **Log Sources** | Specific originators writing data to a table. One table can have many log sources. | Palo Alto PA-3200 → CommonSecurityLog, DC01 → Syslog |
| **Tables** | Log Analytics destinations where data lands. The ground truth layer. | SignInLogs, CommonSecurityLog, BehaviorAnalytics, Syslog |
| **Watchlists** | Static reference data used by detections, capabilities, and reports | VIP Users, Critical Assets, Domain Controllers, Network Ranges |
| **Capabilities** | Features that depend on multiple data sources and integrations to function. If any dependency breaks the capability degrades — often silently. | UEBA, Custom Threat Feed, Fusion ML, SOC Optimization |
| **Detections** | Analytics rules that consume table data and produce alerts and incidents | AB-series scheduled rules, NRT rules, Microsoft Content Hub rules |
| **Silent Detections** | A specific category of detection that monitors whether a log source or table is actively sending data. Every log source must have one. | Heartbeat miss detection, table silence detection |
| **Automation** | Logic Apps, playbooks, and automation rules that respond to incidents and alerts | Incident enrichment playbook, ServiceNow ticket creation, Teams notification |
| **Reports** | Scheduled outputs that run on a cadence and deliver information to stakeholders | Scheduled Logic App reports using watchlists, client verification reports |
| **Integrations** | API keys, service principals, OAuth tokens, and credentials that connect services. If these expire, dependent capabilities and automation fail silently. | VirusTotal API key, Threat Feed API key, Logic App service principal |
| **Workbooks** | Interactive live dashboards that visualize workspace data in real time | Data source audit workbook, MITRE coverage workbook, Watchlist Explorer |

---

## Workbook Design — Full Tab Structure

The primary audit workbook is called the **Client Security Operations Workbook**. It is the centerpiece of every client review. All tabs pull live data. No manual preparation required.

### Tab 1 — Summary
**Purpose:** The opening view. Rolled-up health at a glance. Client-readable in under ten seconds.

**Contents:**
- Header with client name and review date
- Summary metric cards:
  - Data Sources: X of Y healthy
  - Capabilities: X of X active
  - Detections Active: X of Y (detections with healthy data vs total deployed)
  - Silent Detections: X of Y healthy
  - Integrations: X of Y valid
- Source group health table — one row per category, X of Y log sources format
- Click to expand into Tab 2 detail for any row

**Technical note:** Expandable rows in Sentinel workbooks are achieved using parameters and conditional visibility sections. A parameter captures the selected row value. A linked section below filters the detail table to that selection. This is confirmed supported in Azure Monitor workbooks via the dropdown parameters feature.

### Tab 2 — Data Sources
**Purpose:** Full detail view. One row per log source. The working engineer view.

**Contents:**
- Filter bar — filter by Status, Category, Vendor, Table
- Full table with all columns:
  - Table, Log Source, Source, Vendor, Category, Status, Last Seen, Daily Vol, Purpose, Collection, Silent Detection, Detection Status, Detections, Watchlists, Notes
- Color coded by status
- Rows sortable by any column

### Tab 3 — Silent Detections
**Purpose:** Dedicated view for silent log source detection health. Every log source must have one. This tab makes it immediately visible if any are missing or disabled.

**Contents:**
- Summary: X of Y silent detections healthy
- Table — one row per log source:
  - Log Source, Table, Silent Detection ID, Detection Name, Enabled Status, Health Status, Last Fired, Last Result
- Flagged rows for:
  - Detection disabled
  - Detection unhealthy per SentinelHealth
  - Log source has no silent detection mapped (Missing)

### Tab 4 — Capabilities
**Purpose:** Health view for workspace capabilities. Each capability shows its full dependency chain.

**Contents:**
- One card per capability — UEBA, Threat Feed, Fusion ML, SOC Optimization
- Each card shows:
  - Overall capability status (derived from dependency health)
  - Dependency chain — each required data source with its own status
  - API key dependencies with expiry dates
  - Last verified date
- Click card to expand full dependency detail

**Capabilities documented:**
- **UEBA** — depends on: SignInLogs, AuditLogs, SecurityEvents, MDI sensors, Entra ID sync, BehaviorAnalytics output health
- **Custom Threat Feed** — depends on: Feed connector (custom API — AWS function), API key validity, ThreatIntelligenceIndicator table freshness, indicator count, last update timestamp. Note: new tables are ThreatIntelIndicators and ThreatIntelObjects — verify rules reference correct tables
- **Fusion ML** — depends on: Multiple alert sources connected, XDR connector active, multi-stage correlation enabled
- **SOC Optimization** — depends on: Analytics rules active, tables have data, Defender portal connected

### Tab 5 — Insights
**Purpose:** Historical and analytical view. Built from DataSourceAudit_CL custom table populated by snapshot Logic App.

**Contents:**
- Volume trend charts per source category — monthly and weekly views
- Status change history — what changed between snapshots
- New log sources detected since last snapshot — flagged 🟠
- Missing log sources — expected but not found — flagged 🔵
- Age of watchlist entries — entries not verified in 90+ days
- Decommissioned sources — sources marked Decom over time
- Cost analysis — high volume sources with no detections referencing them

**Note:** This tab is V2 — requires snapshot Logic App to be built first. Tab is planned but not buildable until DataSourceAudit_CL exists.

### Tab 6 — Integrations
**Purpose:** API key and credential health. Never let an expiring key silently break a capability.

**Contents:**
- Table — one row per integration:
  - Key Name, Service, What Depends On It, Expiry Date, Days Until Expiry, Status, Last Verified
- Status flags:
  - 🟢 Valid — more than 60 days to expiry
  - 🟡 Expiring Soon — less than 60 days
  - 🔴 Expired — past expiry date
  - ⚫ Unknown — no expiry date documented
- Alert threshold: Teams notification fires when any key is within 30 days of expiry

---

## Spreadsheet Structure — Final Column Design

The audit spreadsheet has one row per **log source** — not per table. For tables with a single dedicated source the Log Source and Source columns are the same. For shared tables like Syslog and CommonSecurityLog each originating source gets its own row.

### Column Headers (Short / Full Name)

| Short Header | Full Name | Assigned By |
|---|---|---|
| Table | Table Name | Query |
| Log Source | Specific Log Source | Query + Manual |
| Source | Data Source (plain English) | Manual |
| Vendor | Vendor | Manual |
| Category | Category | Manual |
| Status | Health Status | Query |
| Last Seen | Last Log Received | Query |
| Daily Vol | Avg Daily Volume (30d) | Query |
| Purpose | Why It Matters | Manual |
| Collection | How It Collects | Manual |
| Silent Det | Silent Detection ID and Name | Manual |
| Det Status | Silent Detection Health | Query |
| Detections | Other Detections That Rely On This Source | Manual |
| Watchlists | Watchlists Referencing This Source | Manual |
| Notes | Notes | Manual |

### Status Values

| Status | Emoji | Assigned By | Meaning |
|---|---|---|---|
| Active | 🟢 | System | Data flowing, healthy, within expected thresholds |
| Inactive | 🔴 | System | Had data, now silent. Something broke. |
| Review | 🟡 | System | Data flowing but volume anomaly or partial config detected |
| No Data | ⚫ | System | In watchlist, never received data |
| Missing | 🔵 | System | Expected based on reference list, not found in workspace |
| Flag | 🟠 | System | New or unexpected source appeared — needs human classification |
| Decom | 🗑️ | Manual only | Marked for retirement — no longer needed |

---

## Automation Pipeline — Future Design

> This section captures the design intent for the automation layer. None of this is built yet. It is documented here so the design is not lost and can be referenced when we are ready to build it.

### Snapshot Logic App (DataSourceAudit_CL)
- **Runs:** Weekly on a defined schedule
- **What it does:** Executes the joined audit query (live workspace data + watchlist context) and writes results to a custom Log Analytics table called `DataSourceAudit_CL`
- **Output:** One row per log source per snapshot, timestamped
- **Enables:** Tab 5 Insights, historical trending, status change detection

### Change Detection Analytics Rules
Rules that query `DataSourceAudit_CL` and fire when:
- A log source status changes from Active to Inactive
- A new log source appears that is not in the watchlist (Flag)
- A log source disappears that was previously Active (Missing)
- Volume drops more than 50% compared to previous snapshot
- Volume spikes more than 300% compared to previous snapshot
- A capability dependency goes unhealthy

### Teams Notification Playbook
- **Triggers:** Change detection analytics rules above
- **What it does:** Posts a formatted message to the SOC Teams channel
- **Message format:**
  ```
  🔴 INACTIVE — Client X
  Log Source: Palo Alto PA-3200
  Table: CommonSecurityLog
  Last Seen: 14 hours ago
  Impact: Perimeter traffic detections blind
  Detections Affected: AB00055, AB00061
  Action: Investigate Syslog forwarder
  ```

### Watchlist Verification Report
- **Runs:** Monthly
- **What it does:** Queries each watchlist for stale entries, compares against active log sources, identifies decommissioned assets still in watchlists and new assets not yet in watchlists
- **Delivered to:** Client IT team via email or ServiceNow ticket
- **Ask:** Review and update watchlist entries, confirm or remove stale entries

---

## Access and Permissions — Known Limitations

> This section documents what we know and do not know about our current access scope via Lighthouse. These limitations affect what is possible in the audit system today and what requires permission expansion to unlock.

### What We Can Access Today
- Log Analytics workspace — full query access ✅
- Sentinel data — all tables, analytics rules, watchlists, playbooks ✅
- SentinelHealth and SentinelAudit tables ✅
- Defender portal — varies by client migration status ✅

### What May Be Blocked by Lighthouse Delegation
- Azure VM extensions — needed to verify AMA agent health ⚠️
- Azure Resource tags — needed for tagging strategy ⚠️
- Azure Policy — needed for auto-tagging enforcement ⚠️
- Azure Resource Graph — needed for full resource inventory ⚠️
- Logic App management — may be limited depending on resource group access ⚠️

### What Requires Microsoft Graph API
- Full Entra ID device inventory beyond what SignInLogs shows 🔲
- Complete service principal and app registration details 🔲
- Conditional access policy status 🔲
- License assignment details 🔲

### Action Items
- [ ] Document exactly which roles are delegated per client via Lighthouse
- [ ] Identify which clients have broader subscription-level access already
- [ ] Investigate GDAP (Granular Delegated Admin Privileges) — confirmed available in Sentinel as of RSAC 2026 — may solve some Lighthouse limitations
- [ ] Explore Microsoft Graph Explorer for what Graph endpoints are accessible
- [ ] Build permission expansion request template for clients where broader access is needed
- [ ] Investigate Azure Resource Graph connector in workbooks — confirmed supported, may not require subscription-level access

**Important note from RSAC 2026:** Microsoft announced Granular Delegated Admin Privileges (GDAP) for Sentinel at RSAC 2026. This may significantly expand what is possible via Lighthouse delegation without requiring full subscription access. This needs to be investigated before building the tagging and resource discovery features.

---

## Tagging Strategy — Future Design

> Captured here as a future work item. Do not build until Lighthouse and Graph access limitations are fully understood.

### The Problem
Co-managed clients frequently have untagged or inconsistently tagged Azure resources. This makes it impossible to query for asset classification, ownership, or criticality in an automated way. Watchlists fill some of this gap but require manual maintenance.

### The Vision
- Azure Policy enforces default tags on all new resources automatically
- AMA and Arc deployments include standard tags in deployment templates
- A weekly Logic App queries for untagged or default-tagged resources and creates a client ticket to classify them
- Tagged resources flow into watchlists automatically reducing manual maintenance burden

### Default Tag Schema (proposed)
```
Classification:    Unreviewed | CriticalAsset | DomainController |
                   Workstation | Server | NetworkDevice | Other
Environment:       Production | Development | Test | DMZ
LoggingEnabled:    True | False
AgentType:         AMA | Arc | CEF | None
ManagedBy:         [Your company name]
CriticalityLevel:  Critical | High | Medium | Low
```

### Status
💡 Future — requires Lighthouse permission investigation first

---

## Regulatory Compliance Watchlist — Future Design

> Captured here as a future work item for the audit deck phase.

### The Concept
Some clients have regulatory obligations — HIPAA, PCI-DSS, SOC 2, GDPR. Specific tables and data sources are required to meet those obligations. If those sources go silent the client may be out of compliance without knowing it.

### The Design
A watchlist called `RegulatoryRequirements` that maps:
- Regulation name
- Required data sources and tables
- Specific log categories required
- Retention requirements
- Audit frequency

The workbook queries this watchlist and verifies that each required source is Active and that retention settings meet the documented requirement. A dedicated compliance section in the audit deck shows clients their regulatory coverage status.

### Status
💡 Future — design to be completed in audit deck phase (Layer 9)

---

## Watchlist Management System — Future Design

> Captured here as a future work item for Layer 6.

### The Problem
Watchlists in Sentinel have no native management layer. No expiry, no ownership tracking, no dependency mapping, no audit trail, no staleness detection. In a co-managed model the client owns the data but we own the system that depends on it. This creates a risk when watchlist data goes stale.

### The Design
- Every watchlist entry gets a `LastVerified` date field
- Staleness detection fires when entries exceed 90 days without verification
- Auto-discovery queries compare active log sources against watchlist contents
- Monthly client report identifies stale entries and new unclassified assets
- Watchlist health score (verified entries / total entries) tracked over time
- Watchlist Explorer workbook (already built) extended to show health scores

### Known Complexity
The Syslog table problem — many different sources write to Syslog. The watchlist and audit system must track at the log source level not just the table level. The `HostName` and `ProcessName` fields within Syslog identify individual sources. Same applies to CommonSecurityLog using `DeviceVendor` and `DeviceProduct`.

### Status
💡 Future — design to be completed in Layer 6

---

## Files Completed This Session

The following files were built or updated and are ready to commit:

| File | Action | Status |
|---|---|---|
| `README.md` | Full rewrite — story-driven, mailbomb narrative, workbook walkthrough, protect the eggs closing | ✅ Ready to commit |
| `ARCHITECTURE.md` | Updated — repo name, full tree, nine-category taxonomy, capabilities concept, six-tab workbook | ✅ Ready to commit |
| `01-baseline/architect-checklist.md` | Updated — TI table names corrected, UEBA capability dependency table added, Layer 2 bridge section, footer fixed | ✅ Ready to commit |
| `02-data-sources/data-source-audit-runbook.md` | Full rewrite — log source as audit unit, seven statuses, four queries, breakout queries, pre-populated reference, Phase 3 verification, known limitations, onboarding note | ✅ Ready to commit |
| `02-data-sources/workbook-vision-and-design.md` | New file — six-tab design, client meeting narrative, version roadmap, technical notes | ✅ Ready to commit |

## Files Still To Build — Next Priorities

All previous update items have been completed this session. The following are what comes next when work resumes.

### 01-baseline/ — Next Up
- [ ] `data-source-baseline.md` — New file. Industry standard data source requirements. Tier 1, Tier 2, Tier 3. Full entry per source with what it is, why it matters, MITRE coverage, how to configure, how to verify, and what good looks like. Defines the data standard every client environment must meet.
- [ ] `connector-runbook.md` — Step-by-step connector configuration guide
- [ ] `audit-log-enablement.md` — Diagnostic settings and UAL enablement guide
- [ ] `README.md` — Folder README explaining how to use the baseline documents

### 02-data-sources/ — Next Up
- [ ] `table-reference-watchlist.md` — Watchlist schema. How to structure DataSourceInventory, field definitions, how to populate from audit export, how to maintain going forward
- [ ] `kql-validation-queries.md` — Reusable validation queries per data source type
- [ ] `README.md` — Folder README

### 03-detection-library/ — Next Major Layer
- [ ] `catalog-index.md` — Master detection table
- [ ] `detection-template.md` — Blank template for new detection entries
- [ ] `README.md` — How to use the catalog
- [ ] `detections/` — One file per AB-series detection

### Ongoing
- [ ] Update PROGRAM-MAP status flags as each file is completed
- [ ] Review and update ARCHITECTURE.md when new layers are added

---

## Ideas Parking Lot

> Ideas that came up during brainstorming that are too good to lose but not yet designed or scheduled. Revisit these during future planning sessions.

**Watchlist-driven client asset verification ticket**
Monthly Logic App sends a formatted ServiceNow ticket to the client IT team listing watchlist entries that need verification. Client updates the ticket, your team processes the changes. Closes the co-management loop on data accuracy.

**Detection coverage score in summary header**
"Detections Active: 72 of 83" — detections with healthy data vs total deployed. The most powerful single number for demonstrating value and surfacing gaps simultaneously. V1 consideration for Tab 1 summary.

**Regulatory compliance tab in workbook**
Maps required data sources to regulatory frameworks. Shows clients their compliance coverage status in real time. HIPAA, PCI, SOC 2, GDPR.

**Graph API exploration**
Microsoft Graph provides access to device inventory, service principal details, conditional access policy status, and license assignments that Sentinel cannot see. Worth a dedicated investigation session to understand what is accessible via current Lighthouse delegation and what Graph endpoints are available.

**GDAP investigation**
Microsoft announced Granular Delegated Admin Privileges for Sentinel at RSAC 2026. This may be the key to unlocking broader Azure resource access without full subscription delegation. High priority investigation item.

**Custom table temporal audit (DataSourceAudit_CL)**
Snapshot Logic App writes audit state weekly. Enables historical trending, status change detection, and automated Teams alerts. Designed but not yet built. Prerequisite for Tab 5 Insights.

**Tagging automation via Azure Policy and AMA deployment templates**
Auto-tag all new resources with Classification: Unreviewed. Weekly Logic App finds untagged resources and creates client ticket. Reduces watchlist maintenance burden significantly. Blocked by Lighthouse permission investigation.

**Watchlist health scoring**
Verified entries / total entries per watchlist. Tracked over time. Shown in audit deck. Drives the watchlist verification report conversation with clients.

**Microsoft Sentinel data lake workbooks**
Confirmed at RSAC 2026 — Sentinel workbooks can now run directly on the data lake using KQL. Enables trend analysis and executive reporting from the data lake. Investigate for Tab 5 Insights and audit deck reporting.

**Custom graph powered by Fabric**
Announced at RSAC 2026 — custom security graphs using data from Sentinel data lake and non-Microsoft sources. Could be used to visualize dependency chains for capabilities, watchlist relationships, and detection coverage maps. Future investigation item.

---

*This document is the living map of the entire program. It is updated whenever new decisions are made, new ideas are locked in, or existing files need revision. It is the first document to read when returning to this project after any break.*

---

## Session 3 Updates — April 2026

### Files Updated This Session

| File | What Changed |
|---|---|
| `02-data-sources/data-source-audit-runbook.md` | Final column set locked — Origin and Transport replace Vendor and Source. Status plain text. Detections AB numbers only. Full approved value lists documented. Pre-populated reference expanded to 40+ tables. |
| `resources/table-reference/README.md` | Template updated with Origin and Transport fields replacing Vendor. Approved values reference added. |
| `resources/table-reference/SecurityEvent.md` | Quick reference updated with Origin and Transport. Vendor removed. |
| `01-baseline/license-entitlement-review.md` | New file — documents client license tier and what capabilities that unlocks |

### Final Locked Column Set — Tab 1

```
Table, Status, LastSeen_UTC, DaysSinceLastLog, LogSource,
Origin, Transport, Category, Purpose, SilentDet, Detections, Notes
```

### Key Decisions Made This Session
- Origin and Transport replace Vendor and Source — two fields answering two distinct questions
- Status is plain text only — no emojis — watchlist compatible
- Detections field is AB numbers only — no names
- Flag status reserved for automation pipeline — not used in manual audit
- Type field eliminated — Origin and Transport together answer what Type was trying to answer
- ASIM tables documented as Transport = ASIM Parser, Origin = Multiple Sources
- SecurityEvent is single log source category — one row covers whole table
- Tab 2 added for technical configuration reference — engineer facing only
- Purpose stays in Tab 1 — workbook needs it to display human-readable row context

### New Concepts Introduced This Session
- ASIM — normalized schema tables fed by parsers from raw tables
- Origin vs Transport — what generated the data vs how it got into Sentinel
- Log source category vs individual device — one row per source type not per machine
- License entitlement review — prerequisite before any baseline or audit work