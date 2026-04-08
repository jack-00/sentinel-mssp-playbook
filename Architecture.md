# Architecture — sentinel-mssp-playbook

> This document explains the overall design of the sentinel-mssp-playbook program — how the layers relate to each other, why each piece exists, and how everything connects into a single cohesive system. Read this before working in any individual layer folder.
>
> For the full program scope, file status, future work items, and ideas parking lot see `PROGRAM-MAP.md`.

---

## Repository Structure

```
sentinel-mssp-playbook/
├── README.md                            ← Vision document — what this is and why it exists
├── ARCHITECTURE.md                      ← This document — how everything connects
├── PROGRAM-MAP.md                       ← Full scope, file status, future work, ideas
│
├── 01-baseline/
│   ├── README.md                        ← How to use the baseline checklist
│   ├── architect-checklist.md           ← Universal baseline checklist (Layer 1)
│   ├── data-source-baseline.md          ← Industry standard data source requirements
│   ├── connector-runbook.md             ← Step-by-step connector configuration
│   └── audit-log-enablement.md         ← Diagnostic settings and UAL enablement guide
│
├── 02-data-sources/
│   ├── README.md                        ← How to use the data source audit system
│   ├── data-source-audit-runbook.md     ← Full audit process — queries, fields, workflow
│   ├── workbook-vision-and-design.md    ← Workbook tab structure and client experience
│   ├── table-reference-watchlist.md     ← Watchlist schema and population guide
│   └── kql-validation-queries.md        ← Validation queries per data source
│
├── 03-detection-library/
│   ├── README.md                        ← How to use the detection catalog
│   ├── catalog-index.md                 ← Master table — all detections, MITRE mapping, required tables
│   ├── detection-template.md            ← Blank template for new detection entries
│   └── detections/
│       └── AB00000.md                   ← One file per detection
│
├── 04-capabilities/
│   ├── README.md                        ← What capabilities are and why they matter
│   ├── ueba.md                          ← UEBA dependency chain and health guide
│   ├── threat-feed.md                   ← Custom threat feed dependency chain
│   ├── fusion-ml.md                     ← Fusion ML requirements and health
│   └── soc-optimization.md             ← SOC optimization configuration and use
│
├── 05-xdr-migration/
│   ├── README.md                        ← XDR migration overview
│   ├── pre-migration-checklist.md       ← Validate before migrating
│   ├── post-migration-checklist.md      ← Validate after migrating
│   ├── schema-changes.md               ← Alert schema differences — standalone vs XDR connector
│   └── new-capabilities.md             ← What unlocks after migration
│
├── 06-watchlist-management/
│   ├── README.md                        ← Watchlist management overview
│   ├── watchlist-schemas.md             ← Standard watchlist field definitions
│   ├── watchlist-health-queries.md      ← Queries for staleness and coverage gaps
│   └── client-verification-report.md    ← Monthly client watchlist verification process
│
├── 07-integrations/
│   ├── README.md                        ← API key and credential management overview
│   ├── api-key-inventory-schema.md      ← Schema for the API key inventory watchlist
│   └── credential-health-queries.md     ← Queries for expiry and dependency tracking
│
├── 08-silent-detections/
│   ├── README.md                        ← Silent detection standards and requirements
│   ├── silent-detection-standards.md    ← Every log source must have one — rules and process
│   └── heartbeat-detection-guide.md     ← AMA heartbeat detection configuration
│
├── 09-audit-deck/
│   ├── README.md                        ← How to run a client audit review
│   ├── deck-template.md                 ← Slide structure and section guidance
│   ├── metrics-and-kpis.md             ← What to measure and how to present it
│   └── gap-framing-guide.md            ← How to present gaps as improvement opportunities
│
├── 10-automation-pipeline/
│   ├── README.md                        ← Automation pipeline overview
│   ├── snapshot-logic-app.md            ← DataSourceAudit_CL snapshot design
│   ├── teams-notification-playbook.md   ← SOC Teams alert design
│   └── change-detection-rules.md        ← Analytics rules for status change detection
│
├── 11-access-and-permissions/
│   ├── README.md                        ← Lighthouse and access scope documentation
│   ├── lighthouse-delegation-guide.md   ← What is and is not accessible via Lighthouse
│   └── graph-api-exploration.md         ← Microsoft Graph API capabilities investigation
│
└── resources/
    ├── taxonomy.md                      ← Full workspace taxonomy — nine categories defined
    ├── mitre-mapping-reference.md       ← MITRE ATT&CK tactic and technique reference
    ├── kql-snippets.md                  ← Reusable KQL patterns across the program
    └── glossary.md                      ← Terms, acronyms, and concepts used in this repo
```

---

## The Workspace Taxonomy

Before working in any layer it is important to understand that a Sentinel workspace contains distinct types of components. Each type plays a different role and has different dependencies. Using consistent terminology across all documents is what makes this program navigable at scale.

| Category | What It Is | Examples |
|---|---|---|
| **Log Sources** | Specific originators writing data to a table. One table can have many log sources writing to it. | Palo Alto PA-3200 → CommonSecurityLog, DC01 → Syslog |
| **Tables** | Log Analytics destinations where data lands. The ground truth layer. | SignInLogs, CommonSecurityLog, BehaviorAnalytics |
| **Watchlists** | Static reference data used by detections, capabilities, and reports | VIP Users, Critical Assets, Domain Controllers |
| **Capabilities** | Features that depend on multiple data sources and integrations working together. If any dependency breaks the capability degrades — often silently. | UEBA, Custom Threat Feed, Fusion ML, SOC Optimization |
| **Detections** | Analytics rules that consume table data and produce alerts and incidents | AB-series scheduled rules, NRT rules, Content Hub rules |
| **Silent Detections** | A specific category of detection that monitors whether a log source is actively sending data. Every log source must have one. | Heartbeat miss detection, table silence detection |
| **Automation** | Logic Apps, playbooks, and automation rules that respond to incidents and alerts | Enrichment playbook, ServiceNow ticket creation |
| **Reports** | Scheduled outputs that run on a cadence and deliver information to stakeholders | Scheduled Logic App reports using watchlists |
| **Integrations** | API keys, service principals, OAuth tokens, and credentials that connect services. If these expire, dependent capabilities fail silently. | VirusTotal API key, Threat Feed API key |
| **Workbooks** | Interactive live dashboards that visualize workspace data in real time | Client Security Operations Workbook |

---

## The Program Layers

The sentinel-mssp-playbook program is built in layers that depend on each other. Do not skip ahead — a client is not ready for Layer 4 if Layer 1 is incomplete.

```
Layer 1 — Baseline        →    Layer 2 — Data Sources
     ↓                               ↓
Layer 4 — Audit Deck      ←    Layer 3 — Detection Library
```

### Layer 1 — Universal Sentinel Baseline
Every Sentinel workspace we manage is validated against a non-negotiable checklist before anything else happens. This covers connector configuration, audit log enablement, UEBA and other capabilities, analytics rules, watchlists, automation, workspace settings, and health monitoring. A client does not move to Layer 2 until Layer 1 is signed off.

**Key principle:** For every Microsoft capability that exists in the ecosystem, we make a deliberate decision — enabled and configured, or consciously excluded with a documented reason. Nothing is simply forgotten.

### Layer 2 — Data Source Audit
Once the baseline is solid, we establish a complete and accurate picture of what data each client has, what they should have, what the gaps are, and what those gaps cost them in terms of detection coverage. We audit by log source — not by connector or table — because one table can have many sources and individual source failures are invisible at the table level.

**Key principle:** Connector status is meaningless as an audit metric. What matters is whether specific log sources are writing data, how recent that data is, and what depends on it.

### Layer 3 — Detection Library
A master catalog of every detection we offer, documented in human-readable terms. Each entry explains what the detection does, what data it requires, why it matters, what MITRE technique it maps to, and how a client would configure the required data source. The catalog is static documentation. Client-specific detection status is dynamic and lives in the workbook.

**Key principle:** One master list, one entry per detection, one status layer per client. The AB-series detection ID is the thread that connects repo documentation to the live Sentinel environment.

### Layer 4 — Audit Deck and Continuous Improvement
The workbook is always running. The detection library is always current. The audit deck is the act of narration — not research. When it is time for a client review, the engineer opens the workbook and the story is already there. Gaps are presented as improvement opportunities with recommendations attached, not as failures.

**Key principle:** The audit deck requires no prep work. The living workbook and detection library do the work continuously so the review is a presentation, not a scramble.

---

## The Capabilities Concept

Capabilities are features of the Sentinel workspace that depend on multiple data sources and integrations working together to function correctly. This is different from a simple connector or table — a capability has a dependency chain, and if any link in that chain breaks the capability degrades or fails entirely, often silently.

**Why this matters:** You can enable UEBA and it will appear active. But if the sign-in logs stop flowing three weeks later, UEBA keeps running against stale data. The BehaviorAnalytics table still writes. Everything looks green. But the behavioral baselines are no longer current and anomaly detection has quietly degraded. Without tracking capability dependencies explicitly this failure is invisible.

**Capabilities in every client environment:**

| Capability | Key Dependencies |
|---|---|
| UEBA | SignInLogs, AuditLogs, SecurityEvents, MDI sensors, Entra ID sync |
| Custom Threat Feed | Feed connector, API key, ThreatIntelIndicators freshness, indicator count |
| Fusion ML | Multiple alert sources, XDR connector, multi-stage correlation enabled |
| SOC Optimization | Active analytics rules, tables with data, Defender portal connected |

Capability health is monitored in Tab 4 of the Client Security Operations Workbook. Configuration of capabilities is covered in `01-baseline/architect-checklist.md`. Deep documentation for each capability lives in `04-capabilities/`.

---

## The Living Workbook System

The Client Security Operations Workbook is the operational engine of the entire program. It runs inside Microsoft Sentinel against each client's workspace in real time. It is powered by two sources working together — live KQL queries against the workspace and a curated data source watchlist that provides the context that cannot be queried.

### The Four Questions the Workbook Answers

| Question | Workbook Tab | Data Source |
|---|---|---|
| What data do we have and is it healthy? | Tab 2 — Data Sources | Live workspace query + watchlist |
| Do all log sources have silent detections? | Tab 3 — Silent Detections | SentinelHealth + watchlist |
| Are all capabilities fully operational? | Tab 4 — Capabilities | Dependency health per capability |
| Are credentials valid and current? | Tab 6 — Integrations | API key inventory watchlist |

### Six-Tab Workbook Structure

**Tab 1 — Summary**
The opening view. Rolled-up health in under ten seconds. Summary metric cards showing Data Sources X of Y, Capabilities X of X, Detections Active X of Y, Silent Detections X of Y, Integrations X of Y. Source group health table — one row per category, click to expand into Tab 2 detail.

**Tab 2 — Data Sources**
Full detail view. One row per log source. Every column visible. Color coded by status. Filterable by category, vendor, status. The working engineer view and the drill-down destination from Tab 1.

**Tab 3 — Silent Detections**
Dedicated view for silent log source detection health. Every log source must have a corresponding silent detection. This tab surfaces any that are missing, disabled, or unhealthy. Summary X of Y count at the top.

**Tab 4 — Capabilities**
One card per capability — UEBA, Threat Feed, Fusion ML, SOC Optimization. Each card shows overall capability status derived from dependency health, the full dependency chain with individual status per dependency, and API key validity where applicable.

**Tab 5 — Insights** *(Version 2 — requires automation pipeline)*
Historical and analytical view. Built from the DataSourceAudit_CL custom table populated by the snapshot Logic App. Volume trends, status change history, new and missing sources, watchlist entry age, cost analysis.

**Tab 6 — Integrations**
API key and credential health. One row per integration. Expiry dates, days until expiry, what depends on each key, status. Teams notification fires when any key is within 30 days of expiry.

### Full Workbook Design
See `02-data-sources/workbook-vision-and-design.md` for the complete tab-by-tab design, the client meeting narrative, version roadmap, and technical implementation notes.

---

## The Detection Library Architecture

### Master Catalog
One markdown file per detection stored in `03-detection-library/detections/`. Each file follows the standard detection template and contains: detection ID, name, plain English description, business impact, client value statement, MITRE mapping, required tables, validation KQL, and configuration guide links.

### Catalog Index
A single markdown table in `catalog-index.md` listing every detection with its ID, name, MITRE tactic, required tables, and a link to the full entry.

### Client Status Layer
The workbook queries each client's workspace against the required tables documented in the catalog. Status is determined dynamically — not maintained manually. An engineer never updates a spreadsheet to mark a detection as active or inactive. The data tells the truth automatically.

### Detection ID Convention
All custom detections follow the `AB#####` naming prefix. This prefix is queryable in KQL and distinguishes custom content from Microsoft Content Hub rules.

```kql
// Find all custom detection rules
SentinelHealth
| where SentinelResourceName matches regex @"^AB\d+"
```

---

## The Audit Deck Philosophy

### Narration, Not Research
The audit deck is built from the workbook. There is no separate data gathering, no manual report preparation, no scrambling before a client meeting. The engineer opens the workbook, captures the current state, and narrates the story. Preparation time approaches zero.

### Gap Framing
Gaps in detection coverage are never presented as failures. They are always framed as improvement opportunities with a recommendation and a path forward attached.

- ❌ "We are missing visibility into lateral movement"
- ✅ "By onboarding Windows Security Events we would gain coverage across five additional MITRE techniques including lateral movement — here is what that would look like and here is how we get there"

### The Audit Deck Slide Structure
1. Program health snapshot — summary metrics, data source status, workspace health
2. What we caught — incident metrics, top detections, notable incidents with narrative
3. Detection coverage — MITRE heatmap, active detections, detection trends over time
4. Where we can improve — gaps framed as opportunities, prioritized recommendations
5. Detections available to unlock — AB-series rules with no current data, what it would take
6. XDR and AI innovations — what is new, what we are evaluating, what we have adopted

---

## Key Design Principles

**Silent gaps are the enemy.** A broken detection fires incorrectly and you notice. A missing log source produces nothing and you do not notice — until a client does. Every system in this program is designed to surface silent gaps before they become client-visible failures.

**Audit by log source, not by table.** A table can have many sources writing to it. One source going silent while others remain active leaves the table appearing healthy. The audit system tracks at the log source level — not the table level.

**Tables do not lie. Connector widgets do.** A connector can show as connected while data has silently stopped flowing. The audit system always queries tables and log sources directly.

**Capabilities have dependency chains.** Enabling a capability is not the same as having a healthy capability. UEBA enabled with no sign-in logs flowing is not UEBA — it is a degraded system that appears active. Every capability's dependencies are tracked explicitly.

**One master list, dynamic client status.** The detection library is written once. Client-specific status is determined by live data, not maintained manually.

**The workbook does the work. The deck tells the story.** Engineers should never spend significant time preparing for a client review. If preparation is taking hours, something is wrong with the system design.

**Constant improvement requires a clear baseline.** You cannot show a client how far they have come without knowing where they started. Every client gets a baseline snapshot at the beginning of the program. Every audit deck shows movement from that baseline.

---

*This document is maintained in the sentinel-mssp-playbook repository. Update it when the program architecture changes — not when individual checklists or templates change. Those changes are tracked in their own files.*