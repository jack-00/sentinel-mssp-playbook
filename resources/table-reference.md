# Table Reference

> **Purpose:** A permanent reference library for Microsoft Sentinel Log Analytics tables. One file per table. Built from real environment analysis — not theory. Each entry documents what the table is, what fields matter, how to identify log sources within it, and the exact values needed to fill in the audit spreadsheet.
>
> **How entries get added:** Through the table dissection process during audit sessions. When a table is fully analyzed and understood its entry is written here. This saves time on every future audit by eliminating the need to re-analyze tables that have already been worked through.
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
3. Use the provided KQL to analyze the table in your specific environment

**Adding a new entry:**
1. Create a new file named exactly as the table appears in KQL — example: `SecurityEvent.md`
2. Follow the entry template below
3. Fill in every section
4. Date stamp the entry
5. Update the index table at the bottom of this README

---

## Entry Template

```markdown
# [TABLE_NAME]

## Spreadsheet Quick Reference
> Copy these values directly into the audit spreadsheet

| Field | Value |
|---|---|
| Table | [TABLE_NAME] |
| LogSource | [Plain English description of what writes to this table] |
| Origin | [From approved Origin list] |
| Transport | [From approved Transport list] |
| Category | [From approved Category list] |
| Purpose | [One sentence — what attack or risk does this detect] |
| Data Connector | [Sentinel connector name or None] |
| Single Source | [Yes — one log source type / No — multiple sources possible] |

---

## What It Is
[Two to three sentences. Plain English. What is this table, where does it
come from, what produces it.]

## Log Source
[Single or multiple source types. Which field identifies individual sources
within this table if multiple exist.]

## Notes
[Anything critical to know when filling in the spreadsheet. Common
misconfigurations. Important caveats.]

---

*Phase 2 — detailed queries, field reference, MITRE mapping, and
misconfiguration analysis coming*
```

---

## Approved Field Values Reference

### Origin
Microsoft Entra ID / Microsoft 365 / Microsoft Defender / Microsoft Azure /
Microsoft Sentinel / Windows OS / Network Device / AWS / Salesforce / Zoom /
Keeper Security / SecureW2 / Drupal / Multiple Sources / Custom / Unknown

### Transport
Microsoft Connector / XDR Connector / AMA — DCR / AMA — DCR — DCE /
CEF — AMA — DCR / Cribl — AMA — DCR / Cribl — DCR / Diagnostic Setting /
Logic App — Polling / Logic App — Webhook / REST API — Push / TAXII — Polling /
Sentinel Native / ASIM Parser

### Category
Identity / Endpoint / Email / Network / Firewall / Cloud Infrastructure /
SaaS Application / Threat Intelligence / Compliance and Audit /
Vulnerability Management / Authentication / Capabilities / Other

---

## Tables Documented

| Table | Category | Origin | Transport | Date Added |
|---|---|---|---|---|
| SecurityEvent | Endpoint | Windows OS | AMA — DCR | April 2026 |

*Update this index every time a new table file is added.*