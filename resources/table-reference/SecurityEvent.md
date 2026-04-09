# SecurityEvent

## Spreadsheet Quick Reference
> Copy these values directly into the audit spreadsheet

| Field | Value |
|---|---|
| Table | SecurityEvent |
| LogSource | Windows Security Events via AMA |
| Origin | Windows OS |
| Transport | AMA — DCR |
| Category | Endpoint |
| Purpose | Detects authentication attacks, lateral movement, privilege escalation, and persistence mechanisms across Windows machines |
| Data Connector | Windows Security Events via AMA — available in Content Hub |
| Single Source | Yes — all records come from Windows machines via AMA. One row in the spreadsheet covers the whole table. Document enrolled machine count in Notes. |

---

## What It Is

SecurityEvent contains the Windows Security Event Log collected from Windows machines via the Azure Monitor Agent. It captures authentication events, privilege use, process creation, account management, and other security-relevant Windows activity. The Data Collection Rule (DCR) configuration controls which Event IDs are collected — this varies significantly between environments and is one of the most common sources of partial configuration.

## Log Source

Single source type — Windows machines via AMA. All records in this table come from the same collection mechanism regardless of how many machines are enrolled. Do not create separate spreadsheet rows per machine. If you need to document which machines are enrolled add that to the Notes column using the SecurityEvent breakout query in the runbook.

## Notes

The most important thing to verify for this table is not whether it exists but whether the right Event IDs are being collected. A SecurityEvent table missing critical IDs like 4625 (failed logon) or 4688 (process creation) provides significantly less detection coverage than it appears to. This is verified during Phase 3 of the audit process using the Event ID coverage check.

The DCR must explicitly collect from the Security event log — not just System or Application logs. This is the most common misconfiguration. Process creation auditing (EventID 4688) also requires Windows audit policy to be configured in addition to the DCR — the DCR cannot collect events that Windows never generated.

---

*Phase 2 — detailed queries, EventID reference table, MITRE mapping, and misconfiguration analysis coming*