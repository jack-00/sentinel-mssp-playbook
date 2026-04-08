# SecurityEvent

## Spreadsheet Quick Reference
> Copy these values directly into the audit spreadsheet

| Field | Value |
|---|---|
| Table | SecurityEvent |
| Log Source | Windows Security Events via AMA |
| Source | Windows Security Event Logs |
| Vendor | Microsoft |
| Category | Endpoint |
| Collection | AMA Agent |
| Purpose | Detects authentication-based attacks, lateral movement, privilege escalation, and persistence mechanisms across Windows machines |
| Data Connector | Windows Security Events via AMA — available in Content Hub |
| Single Source | Yes — all records come from Windows machines via AMA Agent. One row in the spreadsheet covers the whole table. Individual machine names go in Notes if needed. |

---

## What It Is

SecurityEvent contains the Windows Security Event Log collected from Windows machines via the Azure Monitor Agent. It captures authentication events, privilege use, process creation, account management, and other security-relevant Windows activity. The DCR configuration controls which Event IDs are collected — this varies between environments and is one of the most common sources of partial configuration.

## Log Source

Single source type — Windows machines via AMA. All records in this table come from the same collection mechanism regardless of how many machines are enrolled. Do not create separate spreadsheet rows per machine. If you need to document how many machines are enrolled add that to the Notes column.

## Notes

The most important thing to verify for this table is not whether it exists but whether the right Event IDs are being collected. A SecurityEvent table missing critical IDs like 4625 or 4688 provides significantly less detection coverage than it appears to. This is verified during Phase 3 of the audit process.

---

*Phase 2 — detailed queries, EventID reference, MITRE mapping, and misconfiguration analysis coming*