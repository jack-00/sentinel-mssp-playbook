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

---

## What This Table Is

SecurityEvent contains the Windows Security Event Log collected from Windows machines via the Azure Monitor Agent. It captures authentication events, privilege use, process creation, account management, and other security-relevant Windows activity. Data flows from enrolled machines through a Data Collection Rule (DCR) into this table. The DCR configuration controls which Event IDs are collected — this is one of the most common sources of partial configuration across environments.

---

## Log Source Identification

**Primary identifier field:** `Computer` — identifies which machine sent each record

**Single or multiple sources:** Multiple machines write to this table but they are all the same type — Windows machines via AMA. This is treated as a single log source category in the audit spreadsheet, not one row per machine.

**When multiple sources exist:** Not applicable. All sources are Windows machines via AMA. Individual machine names go in the Notes column if needed — not as separate spreadsheet rows.

---

## Key Fields

| Field | What It Contains | Why It Matters |
|---|---|---|
| `Computer` | Machine name that generated the event | Log source identifier |
| `EventID` | Windows Event ID number | Determines what type of security event this is |
| `Account` | User account involved | Who performed the action |
| `AccountType` | User or Machine | Distinguishes user activity from service activity |
| `LogonType` | How the logon occurred | 2=Interactive, 3=Network, 10=RemoteInteractive — critical for attack pattern detection |
| `WorkstationName` | Source machine for network logons | Where the connection came from |
| `IpAddress` | Source IP for network logons | External vs internal origin |
| `ProcessName` | Process involved in the event | Used for process creation and privilege events |
| `SubjectUserName` | User who initiated the action | For events like account creation |
| `TargetUserName` | User the action was performed on | For events like account modification |
| `CommandLine` | Command line arguments for process creation | Only present if 4688 auditing includes command line — often not configured |

---

## Critical Event IDs

| EventID | Description | Security Value | MITRE Technique |
|---|---|---|---|
| 4624 | Successful logon | Authentication baseline — detects unusual logon patterns | T1078 Valid Accounts |
| 4625 | Failed logon | Brute force and password spray detection | T1110 Brute Force |
| 4648 | Logon with explicit credentials | Pass-the-hash, lateral movement | T1550 Use Alternate Auth Material |
| 4672 | Special privileges assigned | Privilege escalation detection | T1548 Abuse Elevation Control |
| 4688 | New process created | Malware execution, living off the land | T1059 Command Scripting |
| 4698 | Scheduled task created | Persistence mechanism detection | T1053 Scheduled Task |
| 4720 | User account created | Backdoor account creation | T1136 Create Account |
| 4726 | User account deleted | Account cleanup post-compromise | T1531 Account Access Removal |
| 4728 | Member added to global security group | Privilege escalation | T1098 Account Manipulation |
| 4732 | Member added to local security group | Local privilege escalation | T1098 Account Manipulation |
| 4756 | Member added to universal security group | Privilege escalation | T1098 Account Manipulation |
| 4776 | NTLM authentication attempt | Pass-the-hash detection | T1550.002 Pass the Hash |
| 4768 | Kerberos TGT requested | Kerberoasting baseline | T1558 Steal or Forge Kerberos Tickets |
| 4769 | Kerberos service ticket requested | Kerberoasting detection | T1558.003 Kerberoasting |

---

## Security Value

Windows Security Events are the foundation of on-premises identity and endpoint detection. Without this table you cannot detect authentication-based attacks, lateral movement through Windows environments, privilege escalation, persistence via scheduled tasks or account creation, or process-based attack patterns on Windows machines. For environments with on-premises Active Directory or hybrid identity this table is non-negotiable.

---

## MITRE ATT&CK Coverage

| Tactic | Technique |
|---|---|
| Initial Access | T1078 Valid Accounts |
| Credential Access | T1110 Brute Force, T1550 Pass the Hash, T1558 Kerberoasting |
| Lateral Movement | T1021 Remote Services, T1550 Use Alternate Auth Material |
| Privilege Escalation | T1548 Abuse Elevation, T1098 Account Manipulation |
| Persistence | T1136 Create Account, T1053 Scheduled Task |
| Execution | T1059 Command Scripting |
| Defense Evasion | T1562 Impair Defenses |

---

## What Good Looks Like

- **Expected frequency:** Continuous — data should arrive every few minutes in any active Windows environment
- **Expected volume:** Varies widely by environment size and DCR scope. A small environment may be under 100MB/day. A large enterprise with full auditing can exceed 10GB/day.
- **Key fields that must be present:** Computer, EventID, Account, TimeGenerated
- **Health indicator:** Critical Event IDs 4624 and 4625 should always be present in any active environment. If these are missing the DCR is not collecting Security log events.
- **Minimum viable EventID set:** 4624, 4625, 4648, 4672, 4688, 4720, 4728 — if any of these are absent flag as Review and investigate DCR configuration

---

## Common Misconfigurations

- **DCR collecting System log only — Security log missing entirely.** The DCR must explicitly include the Security event log. This is the most common misconfiguration and results in a table that appears active but contains no security-relevant events.
- **Process creation auditing (4688) not enabled in Windows audit policy.** Even if the DCR is correctly configured, 4688 events will not generate unless process creation auditing is enabled in Windows Local Security Policy or Group Policy. The DCR cannot collect events that Windows never generated.
- **CommandLine not captured for 4688 events.** Process creation is visible but command arguments are invisible. This significantly reduces the detection value of process creation events. Requires enabling "Include command line in process creation events" in audit policy.
- **Only domain controllers enrolled.** Workstations and member servers are missing. Lateral movement between workstations is completely invisible.
- **High volume of low-value events.** If EventID 4634 (logoff) or 4656 (object access) are being collected without filtering they generate enormous volume with minimal security value. Review DCR XPath queries for noise reduction.

---

## Dissection Query

```kql
// SecurityEvent — Log Source Breakout
// Shows each machine writing to this table with health status
// Used to understand scope of enrollment not to create separate rows
SecurityEvent
| where TimeGenerated > ago(30d)
| summarize
    LastSeen = max(TimeGenerated),
    TotalRecords = count(),
    UniqueEventIDs = dcount(EventID),
    EventIDs = make_set(EventID, 20)
    by Computer
| extend DaysSinceLastLog = datetime_diff('day', now(), LastSeen)
| extend MachineStatus = case(
    DaysSinceLastLog <= 1, "🟢 Active",
    DaysSinceLastLog <= 7, "🟡 Review",
    DaysSinceLastLog > 7,  "🔴 Inactive",
    "⚫ No Data"
)
| extend LastSeen_UTC = format_datetime(LastSeen, 'yyyy-MM-dd HH:mm')
| project Computer, MachineStatus, LastSeen_UTC, TotalRecords, UniqueEventIDs, EventIDs
| order by Computer asc
```

---

## Health Check Query

```kql
// SecurityEvent — Health Check
// Confirms table is healthy and critical Event IDs are present
SecurityEvent
| where TimeGenerated > ago(24h)
| summarize Count = count() by EventID
| extend EventDescription = case(
    EventID == 4624, "✅ Successful logon",
    EventID == 4625, "✅ Failed logon",
    EventID == 4648, "✅ Logon with explicit credentials",
    EventID == 4672, "✅ Special privileges assigned",
    EventID == 4688, "✅ New process created",
    EventID == 4720, "✅ User account created",
    EventID == 4728, "✅ Member added to global group",
    EventID == 4776, "✅ NTLM authentication",
    EventID == 4768, "✅ Kerberos TGT requested",
    EventID == 4769, "✅ Kerberos service ticket requested",
    strcat("EventID ", tostring(EventID))
)
| extend IsCritical = EventID in (4624, 4625, 4648, 4672, 4688, 4720, 4728, 4776, 4768, 4769)
| project EventID, EventDescription, Count, IsCritical
| order by IsCritical desc, Count desc
```

---

## Event ID Coverage Check Query

```kql
// SecurityEvent — Check which critical Event IDs are MISSING
// Any critical ID not appearing here is not being collected
let critical_ids = datatable(EventID:int, Description:string)
[
    4624, "Successful logon",
    4625, "Failed logon",
    4648, "Logon with explicit credentials",
    4672, "Special privileges assigned",
    4688, "New process created",
    4720, "User account created",
    4728, "Member added to global group",
    4776, "NTLM authentication",
    4768, "Kerberos TGT requested",
    4769, "Kerberos service ticket requested"
];
let present_ids = SecurityEvent
    | where TimeGenerated > ago(24h)
    | summarize by EventID;
critical_ids
| join kind=leftanti present_ids on EventID
| project EventID, Description, Status = "⚠️ Missing — not being collected"
```

---

## Notes

The single most important thing to verify for this table is not whether it exists — it is whether the right Event IDs are being collected. A SecurityEvent table with only logon events and no process creation, group membership changes, or account creation events provides significantly less detection coverage than it appears to on the surface.

Always run the Event ID coverage check query during Phase 3 verification. Do not mark this table Active without confirming the critical Event IDs are present.

---

*First documented: April 2026 — co-managed MSSP environment, mixed Windows server and workstation estate with hybrid Entra ID*