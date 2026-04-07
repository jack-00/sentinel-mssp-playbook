# Architecture — sentinel-blueprint

> This document explains the overall design of the sentinel-blueprint program — how the layers relate to each other, why each piece exists, and how everything connects into a single cohesive system. Read this before working in any individual layer folder.

---

## Repository Structure

```
sentinel-blueprint/
├── README.md                        ← Vision document — what this is and why it exists
├── ARCHITECTURE.md                  ← This document — how everything connects
│
├── 01-baseline/
│   ├── README.md                    ← How to use the baseline checklist
│   ├── architect-checklist.md       ← Universal baseline checklist (Layer 1)
│   ├── workspace-settings.md        ← Workspace, RBAC, and retention reference
│   ├── connector-runbook.md         ← Step-by-step connector configuration
│   └── audit-log-enablement.md      ← Diagnostic settings and UAL enablement guide
│
├── 02-data-sources/
│   ├── README.md                    ← How to use the data source audit system
│   ├── audit-design.md              ← The four-question audit framework
│   ├── tier1-critical.md            ← Critical data sources — must have
│   ├── tier2-recommended.md         ← Recommended data sources — should have
│   ├── tier3-enrichment.md          ← Enrichment sources — nice to have
│   ├── kql-validation-queries.md    ← Validation queries per data source
│   └── client-inventory-template.md ← Per-client data source inventory template
│
├── 03-detection-library/
│   ├── README.md                    ← How to use the detection catalog
│   ├── catalog-index.md             ← Master table — all detections, MITRE mapping, required tables
│   ├── detection-template.md        ← Blank template for new detection entries
│   └── detections/
│       ├── AB00001.md               ← One file per detection
│       ├── AB00002.md
│       └── ...
│
├── 04-xdr-migration/
│   ├── README.md                    ← XDR migration overview
│   ├── pre-migration-checklist.md   ← Validate before migrating
│   ├── post-migration-checklist.md  ← Validate after migrating
│   ├── schema-changes.md            ← Alert schema differences — standalone vs XDR connector
│   └── new-capabilities.md          ← What unlocks after migration
│
├── 05-audit-deck/
│   ├── README.md                    ← How to run a client audit review
│   ├── workbook-design.md           ← Workbook tab structure and KQL queries
│   ├── deck-template.md             ← Slide structure and section guidance
│   ├── metrics-and-kpis.md          ← What to measure and how to present it
│   └── gap-framing-guide.md         ← How to present gaps as improvement opportunities
│
└── resources/
    ├── mitre-mapping-reference.md   ← MITRE ATT&CK tactic and technique reference
    ├── watchlist-schemas.md         ← Standard watchlist field definitions
    ├── kql-snippets.md              ← Reusable KQL patterns across the program
    └── glossary.md                  ← Terms, acronyms, and concepts used in this repo
```

---

## The Four-Layer Program

The sentinel-blueprint program is built in four sequential layers. Each layer depends on the one before it. Do not skip ahead — a client is not ready for Layer 4 if Layer 1 is incomplete.

```
Layer 1 — Baseline        →    Layer 2 — Data Sources
     ↓                               ↓
Layer 4 — Audit Deck      ←    Layer 3 — Detection Library
```

### Layer 1 — Universal Sentinel Baseline
Every Sentinel workspace we manage is validated against a non-negotiable checklist before anything else happens. This covers connector configuration, audit log enablement, UEBA, analytics rules, watchlists, automation, workspace settings, and health monitoring. A client does not move to Layer 2 until Layer 1 is signed off.

**Key principle:** For every Microsoft capability that exists in the ecosystem, we make a deliberate decision — enabled and configured, or consciously excluded with a documented reason. Nothing is simply forgotten.

### Layer 2 — Data Source Audit
Once the baseline is solid, we establish a complete and accurate picture of what data each client has, what they should have, what the gaps are, and what those gaps cost them in terms of detection coverage. This is documented in a per-client inventory and visualized in the living workbook.

**Key principle:** Connector status is meaningless as an audit metric. What matters is whether tables have data, how recent it is, and what depends on it.

### Layer 3 — Detection Library
A master catalog of every detection we offer, documented in human-readable terms. Each entry explains what the detection does, what data it requires, why it matters, what MITRE technique it maps to, and how a client would configure the required data source if they wanted to enable it. The catalog is static documentation. Client-specific detection status is dynamic and lives in the workbook.

**Key principle:** One master list, one entry per detection, one status layer per client. The detection ID (AB-series prefix) is the thread that connects the repo documentation to the live Sentinel environment.

### Layer 4 — Audit Deck and Continuous Improvement
The workbook is always running. The detection library is always current. The audit deck is the act of narration — not research. When it is time for a client review, the engineer opens the workbook and the story is already there. Gaps are presented as improvement opportunities with recommendations attached, not as failures.

**Key principle:** The audit deck requires no prep work. The living workbook and detection library do the work continuously so the review is a presentation, not a scramble.

---

## The Living Workbook System

The workbook is the operational engine of the entire program. It is a custom Microsoft Sentinel workbook that runs against each client's workspace in real time and answers the four core audit questions automatically.

### The Four Questions the Workbook Answers

| Question | Workbook Section | Data Source |
|---|---|---|
| What data do we have? | Data inventory — tables, volume, freshness | `Usage`, `SentinelHealth` |
| What data should we have? | Source gap analysis — missing vs expected | Client inventory + `Usage` |
| What's the gap? | Coverage map — MITRE heatmap | Analytics rules + table status |
| What detections are affected? | Detection status — green / red per AB rule | Detection library + table health |

### Workbook Tab Structure

**Tab 1 — Data Health**
Every table in the workspace. Last ingestion timestamp. Volume over 30 days. Status flag — healthy, stale, or empty. This is the ground truth view. A table that has not received data in 72 hours is immediately flagged.

**Tab 2 — Detection Status**
Every AB-series detection mapped against its required tables. Green means the detection is active and data is flowing. Red means the required data is missing and the detection is running blind. Clicking a red detection links to its KB article and configuration guide.

**Tab 3 — MITRE Coverage Heatmap**
Visual heatmap of MITRE ATT&CK tactic and technique coverage. Three states: covered with healthy data, rule exists but no data, no coverage. This is the most powerful client-facing visual in the program.

**Tab 4 — Incident Activity**
Incident volume over time by severity. Top triggered detections. Notable incidents with brief narrative. MTTD and MTTR trends. This is the "here is what we caught" section.

**Tab 5 — Silent Log Source Health**
One row per configured data source. Confirms data is flowing from each source. Simple green or red status. Immediately readable during a client meeting.

**Tab 6 — Recommendations**
Automatically generated list of improvement opportunities based on red detection status and MITRE coverage gaps. Each item links to the relevant detection library entry and configuration guide.

### Two Outputs from One System

The workbook produces two outputs that serve different purposes:

**The living view** is what your engineers use every day and what you open during a client meeting. Always current, no preparation required.

**The audit snapshot** is a point-in-time export taken from the workbook at review time. This becomes the client-facing deliverable in the audit deck. No separate document to build — the work is already done.

---

## The Detection Library Architecture

### Master Catalog
One markdown file per detection stored in `03-detection-library/detections/`. Each file follows the standard detection template and contains: detection ID, name, plain English description, business impact, client value statement, MITRE mapping, required tables, validation KQL, and configuration guide links.

### Catalog Index
A single markdown table in `catalog-index.md` that lists every detection with its ID, name, MITRE tactic, required tables, and a link to the full entry. This is the at-a-glance reference for engineers and the source the workbook logic is modeled after.

### Client Status Layer
The workbook queries each client's workspace against the required tables documented in the catalog. Status is determined dynamically — not maintained manually. An engineer never updates a spreadsheet to mark a detection as active or inactive. The data tells the truth automatically.

### Detection ID Convention
All custom detections follow the `AB#####` naming prefix. This prefix is queryable in KQL and distinguishes custom content from Microsoft Content Hub rules in every table and every query across the program.

```kql
-- Find all custom detection rules
SentinelHealth
| where SentinelResourceName matches regex @"^AB\d+"
```

### Configuration Guide Links
Every detection entry includes links to an internal KB article explaining how to configure the required data source, and a Microsoft Docs link for the official connector or diagnostic setting setup. This means a client who asks "how do we get that data in?" has a documented path forward immediately available.

---

## The Audit Deck Philosophy

### Narration, Not Research
The audit deck is built from the workbook. There is no separate data gathering, no manual report preparation, no scrambling before a client meeting. The engineer opens the workbook, captures the current state, and narrates the story. Preparation time approaches zero.

### Gap Framing
Gaps in detection coverage are never presented as failures. They are always framed as improvement opportunities with a recommendation and a path forward attached. The language matters:

- ❌ "We are missing visibility into lateral movement"
- ✅ "By onboarding Windows Security Events we would gain coverage across five additional MITRE techniques including lateral movement — here is what that would look like"

### The Audit Deck Slide Structure
1. Program health snapshot — data source status, connector health, workspace summary
2. What we caught — incident metrics, top detections, notable incidents with narrative
3. Detection coverage — MITRE heatmap, active detections, detection trends over time
4. Where we can improve — gaps framed as opportunities, prioritized recommendations
5. Detections available to unlock — AB-series rules with no current data, what it would take to enable them
6. XDR and AI innovations — what is new, what we are evaluating, what we have adopted

---

## The MITRE ATT&CK North Star

MITRE ATT&CK is the measurement framework used throughout this program. Every data source, every detection, and every audit deck section is mapped to MITRE tactics and techniques. This creates a consistent language for talking about coverage that works internally for engineers and externally for clients.

The MITRE heatmap is the single most powerful visual in the program. It shows at a glance:
- Where coverage is strong — tactics with active detections and healthy data
- Where coverage exists but data is missing — tactics where rules are written but tables are empty
- Where there is no coverage at all — tactics with neither rules nor data

Progress is measured by watching the heatmap fill in over time as data sources are onboarded and detections are written. A client who started with coverage across six tactics and now has twelve can see that growth visually. That is the story of a maturing security program told in a single image.

---

## Key Design Principles

**Silent gaps are the enemy.** A broken detection fires incorrectly and you notice. A missing data source produces nothing and you do not notice — until a client does. Every system in this program is designed to surface silent gaps before they become client-visible failures.

**Tables do not lie. Connector widgets do.** A connector can show as connected while data has silently stopped flowing. The audit system always queries tables directly — never trusts connector status UI.

**One master list, dynamic client status.** The detection library is written once. Client-specific status is determined by live data, not maintained manually. This eliminates the most common source of documentation drift in MSSP environments.

**The workbook does the work. The deck tells the story.** Engineers should never spend significant time preparing for a client review. If preparation is taking hours, something is wrong with the system design.

**Constant improvement requires a clear baseline.** You cannot show a client how far they have come without knowing where they started. Every client gets a baseline snapshot at the beginning of the program. Every audit deck shows movement from that baseline.

---

*This document is maintained in the sentinel-blueprint repository. Update it when the program architecture changes — not when individual checklists or templates change. Those changes are tracked in their own files.*