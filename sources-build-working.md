# Sources Watchlist Build — Working Process

> **Status:** This is the working document for developing and testing the sources watchlist build process. Once the process is verified and stable it will be used to update the final guidance in `06-watchlist-management/sources-watchlist-build-guide.md`.
>
> **Last Updated:** April 2026

---

## Step 1 — Get the Full Table and Detection Map

Before doing anything else you need a complete picture of every table in the workspace and which detections depend on each one. This is your working map for everything that follows.

**You need two things ready:**
1. Your detections CSV — `[mssname]-detections` spreadsheet
2. Your tables inventory export — from the main table inventory query in the audit runbook

**Paste this prompt to your AI first. Wait for confirmation before sending any data:**

```
I am building a data source tracking system for Microsoft Sentinel.

I need you to help me create a complete table-first map of my environment.

I will give you two things:
1. A detection catalog CSV with these columns:
   RuleId, AnalyticRule, Table, Watchlist, AlertClass, Description
   The Table field contains comma separated table names — one detection
   may query multiple tables.

2. A list of all tables that exist in the workspace.

Your job is to produce a complete table map that includes every table.

Output format — one row per table:

Table | Detections
SignInLogs | OC00012 - Detection Name, OC00019 - Detection Name
AzureDiagnostics | OC00034 - Detection Name, OC00041 - Detection Name
AADManagedIdentitySignInLogs |

Rules:
- If a detection queries multiple tables list it under each table
- Group all detections for the same table on one row comma separated
- Include the RuleId and rule name for each detection
- Include ALL tables from the workspace list even if no detections use them
- For tables with no detections leave the Detections column blank
- Sort tables alphabetically

Confirm you understand before I provide the data.
```

**After confirmation — paste your detections CSV first, then paste your tables list.**

---

*More steps to be added as process is tested and verified.*