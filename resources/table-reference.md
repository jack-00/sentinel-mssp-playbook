# Table Reference

> **Purpose:** A permanent reference library for Microsoft Sentinel Log Analytics tables. One file per table. Built from real environment analysis — not theory. Use this during audits to quickly fill in spreadsheet values and during investigations to understand what a table contains and how to work with it.
>
> **Last Updated:** April 2026

---

## How to Use This Folder

**During an audit:**
1. Find the file for the table you are documenting
2. Copy the Spreadsheet Quick Reference values at the top directly into your spreadsheet
3. Done — no research needed for documented tables

**During an investigation:**
1. Open the table file
2. Read the key fields, dissection queries, and health check queries
3. Use the provided KQL to analyze the table in your environment

**Adding a new entry:**
1. Create a new file named exactly as the table appears in KQL — example: `SecurityEvent.md`
2. Follow the entry template below
3. Fill in every section — do not leave sections blank
4. Date stamp the entry at the bottom

---

## Entry Template

Copy this template when creating a new table file:

```markdown
# [TABLE_NAME]

## Spreadsheet Quick Reference
> Copy these values directly into the audit spreadsheet

| Field | Value |
|---|---|
| Table | [TABLE_NAME] |
| Log Source | [Category of thing writing to this table — not individual machines] |
| Source | [Plain English name a client would understand] |
| Vendor | [Company that produces this data] |
| Category | [Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure / SaaS Application / Threat Intelligence / Compliance and Audit / Vulnerability Management / Authentication / Capabilities / Other] |
| Collection | [Microsoft Connector / AMA Agent / CEF Syslog / REST API / Logic App / Manual Custom] |
| Purpose | [One sentence — what attack or risk does this help detect, written for a business audience] |

---

## What This Table Is

[Plain English description of the table — what it contains, where it comes from,
what produces it. Two to three sentences.]

---

## Log Source Identification

**Primary identifier field:** [The field used to distinguish sources within this table]

**Single or multiple sources:** [Single source / Multiple sources possible]

**When multiple sources exist:** [Which field to break out on and what to look for]

---

## Key Fields

| Field | What It Contains | Why It Matters |
|---|---|---|
| [field name] | [what it contains] | [why it matters for security] |
| [field name] | [what it contains] | [why it matters for security] |

---

## Security Value

[One paragraph explaining what attacks or risks this table helps detect.
Written for an engineer not a client.]

---

## MITRE ATT&CK Coverage

| Tactic | Technique |
|---|---|
| [Tactic] | [T#### — Technique name] |

---

## What Good Looks Like

- **Expected frequency:** [Continuous / Hourly / Daily / Event-driven]
- **Expected volume:** [Low / Medium / High — with rough MB/GB range if known]
- **Key fields that must be present:** [Fields that should always have values]
- **Health indicator:** [What tells you this table is fully healthy]

---

## Common Misconfigurations

- [Most common thing that goes wrong with this table]
- [Second most common issue]
- [Any gotchas specific to this table]

---

## Dissection Query

```kql
// [TABLE_NAME] — Log Source Breakout
// [Description of what this query does]
[QUERY]
```

---

## Health Check Query

```kql
// [TABLE_NAME] — Health Check
// Run to confirm table is healthy and key fields are present
[QUERY]
```

---

## Notes

[Anything else worth knowing about this table that does not fit above]

---

*First documented: [YYYY-MM-DD] — [environment type e.g. co-managed MSSP, mid-size enterprise]*
```

---

## Tables Documented

| Table | Category | Vendor | Date Added |
|---|---|---|---|
| SecurityEvent | Endpoint | Microsoft | April 2026 |

*Update this index when new tables are added.*