# Multi-Source Table Investigation Queries

> **Purpose:** Use these queries to deeply understand what is inside each
> shared table before documenting it in the audit spreadsheet. Two levels
> of investigation for every table:
>
> **Level 1 — Who is writing to this table?**
> Identifies distinct log sources within the table.
>
> **Level 2 — What are they actually sending?**
> Identifies the categories, event types, and content within each source.
>
> The Azure Firewall lesson: knowing ResourceType = AZUREFIREWALLS is not
> enough. You need to know which log categories are present —
> ApplicationRule, NetworkRule, DnsProxy, ThreatIntelLog — because each
> one represents completely different detection coverage. Going silent on
> one while others remain active creates a blind spot that is invisible
> at the surface level.
>
> This principle applies everywhere. Always run both levels.

---

## AzureDiagnostics

### Level 1 — What resource types are writing here?
```kql
AzureDiagnostics
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by ResourceType, ResourceProvider
| order by Count desc
```

### Level 2 — What categories does each resource type send?
```kql
// Run once per ResourceType found in Level 1
// Replace AZUREFIREWALLS with your ResourceType
AzureDiagnostics
| where TimeGenerated > ago(30d)
| where ResourceType == "AZUREFIREWALLS"
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by Category
| order by Count desc
```

### Known category breakdowns to look for:

**AZUREFIREWALLS** — each category is different detection coverage:
- AzureFirewallApplicationRule — layer 7 application traffic allow/deny
- AzureFirewallNetworkRule — layer 3/4 network traffic allow/deny
- AzureFirewallDnsProxy — DNS queries through firewall (only if DNS proxy enabled)
- AzureFirewallThreatIntelLog — connections matching Microsoft TI indicators

**VAULTS (Key Vault):**
- AuditEvent — secret access, key operations, certificate events (HIGH VALUE)
- AzurePolicyEvaluationDetails — policy compliance events

**NETWORKSECURITYGROUPS:**
- NetworkSecurityGroupEvent — rule change events
- NetworkSecurityGroupRuleCounter — rule hit counters
- NetworkSecurityGroupFlowEvent — actual flow logs (HIGH VALUE — requires NSG Flow Logs enabled)

**WORKFLOWS (Logic Apps):**
- WorkflowRuntime — covers runs, actions, and triggers

### Level 3 — Check for specific resources within a category
```kql
// Find which specific resources are sending a category
// Useful when you have many Key Vaults or Firewalls
AzureDiagnostics
| where TimeGenerated > ago(30d)
| where ResourceType == "VAULTS"
| where Category == "AuditEvent"
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by Resource
| order by Count desc
```

### Level 4 — Verify NSG Flow Logs are actually present
```kql
// NSG Flow Logs require Network Watcher to be enabled
// They are frequently missing even when NSG logging appears configured
AzureDiagnostics
| where TimeGenerated > ago(30d)
| where ResourceType == "NETWORKSECURITYGROUPS"
| where Category == "NetworkSecurityGroupFlowEvent"
| summarize Count = count(), LastSeen = max(TimeGenerated)
```
// If this returns zero — NSG flow logs are NOT configured
// Only rule events are present — much lower detection value

---

## AzureMetrics

### Level 1 — What resource types are sending metrics?
```kql
AzureMetrics
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by ResourceType, Namespace
| order by Count desc
```

### Level 2 — What metrics does each resource type send?
```kql
// Run once per ResourceType
AzureMetrics
| where TimeGenerated > ago(30d)
| where ResourceType == "VAULTS"
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by MetricName
| order by Count desc
```

### Level 3 — Check for security-relevant metric anomalies
```kql
// Key Vault API hits — spikes may indicate enumeration or attack
AzureMetrics
| where TimeGenerated > ago(30d)
| where ResourceType == "VAULTS"
| where MetricName == "ServiceApiHit"
| summarize
    DailyHits = count()
    by bin(TimeGenerated, 1d), Resource
| order by TimeGenerated desc
```

### Notes on AzureMetrics security value:
// Most metrics are operational not security-relevant
// Exceptions worth noting:
// - Key Vault ServiceApiHit — spikes can indicate enumeration
// - Firewall health metrics — detect if firewall is degraded
// - Storage availability — detect if storage is being DoS'd
// - VM CPU spikes — detect cryptomining
// Low priority for silent detection unless specific anomaly
// detections are written against metrics data

---

## CommonSecurityLog

### Level 1 — What vendors and products are writing here?
```kql
CommonSecurityLog
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by DeviceVendor, DeviceProduct
| order by Count desc
```

### Level 2 — What event categories does each vendor send?
```kql
// Run once per DeviceVendor
CommonSecurityLog
| where TimeGenerated > ago(30d)
| where DeviceVendor == "Palo Alto Networks"
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by DeviceEventCategory, Activity
| order by Count desc
```

### Known category breakdowns to look for:

**Palo Alto Networks:**
- TRAFFIC — network traffic logs (allow/deny)
- THREAT — threat detections (malware, C2, exploits)
- URL — URL filtering logs
- WILDFIRE — WildFire sandbox results
- SYSTEM — firewall system events
// Each category is different detection coverage
// THREAT and WILDFIRE are highest security value
// TRAFFIC is high volume — verify it is not creating noise

**Fortinet FortiGate:**
- utm — unified threat management events
- traffic — network traffic logs
- event — system and admin events
- virus — AV detections
// utm and virus are highest security value

### Level 3 — Check for allow vs deny distribution
```kql
// Understanding allow vs deny ratio helps identify noise vs signal
CommonSecurityLog
| where TimeGenerated > ago(7d)
| where DeviceVendor == "Palo Alto Networks"
| where DeviceEventCategory == "TRAFFIC"
| summarize Count = count() by DeviceAction
| order by Count desc
```

### Level 4 — Check source and destination distribution
```kql
// Identify top talkers — helps understand what this firewall is seeing
CommonSecurityLog
| where TimeGenerated > ago(7d)
| where DeviceVendor == "Palo Alto Networks"
| where DeviceEventCategory == "THREAT"
| summarize Count = count() by SourceIP, DestinationIP, Activity
| order by Count desc
| take 20
```

---

## Syslog

### Level 1 — What hosts are writing here?
```kql
Syslog
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by HostName
| order by Count desc
```

### Level 2 — What facilities and severities does each host send?
```kql
// Run once per host
// Facility tells you what component generated the log
Syslog
| where TimeGenerated > ago(30d)
| where HostName == "HOSTNAME"
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by Facility, SeverityLevel
| order by Count desc
```

### Known Syslog facilities and their security value:

| Facility | What It Is | Security Value |
|---|---|---|
| auth / authpriv | Authentication events | HIGH — login, sudo, PAM |
| daemon | Background service events | MEDIUM — service starts/stops |
| kern | Kernel messages | MEDIUM — driver issues, OOM |
| syslog | Syslog daemon itself | LOW — operational |
| cron | Scheduled task execution | MEDIUM — persistence detection |
| local0-local7 | Custom application logs | VARIES — depends on config |
| user | User-level messages | LOW-MEDIUM |
| mail | Mail system | LOW unless mail server |

### Level 3 — Check authentication events specifically
```kql
// Auth facility logs are the highest value in Syslog
// Verify they are present for hosts that should have them
Syslog
| where TimeGenerated > ago(24h)
| where HostName == "HOSTNAME"
| where Facility == "auth" or Facility == "authpriv"
| summarize Count = count(), LastSeen = max(TimeGenerated)
```
// If zero — authentication events are not being collected
// This is a significant gap for Linux machines especially DCs

### Level 4 — Check for CEF forwarder vs Linux host
```kql
// Some Syslog hosts are CEF forwarders not Linux machines
// CEF forwarders should write to CommonSecurityLog not Syslog
// If you see a forwarder VM writing to Syslog investigate
Syslog
| where TimeGenerated > ago(7d)
| where HostName == "HOSTNAME"
| summarize
    Count = count(),
    SampleMessages = make_set(SyslogMessage, 5)
    by ProcessName
| order by Count desc
```

---

## ThreatIntelIndicators

### Level 1 — What feeds are writing here?
```kql
ThreatIntelIndicators
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by SourceSystem
| order by Count desc
```

### Level 2 — What indicator types does each feed provide?
```kql
// Run once per SourceSystem
ThreatIntelIndicators
| where TimeGenerated > ago(30d)
| where SourceSystem == "Microsoft Sentinel"
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by IndicatorType
| order by Count desc
```

### Level 3 — Check indicator freshness per feed
```kql
// Stale indicators reduce TI effectiveness significantly
// Indicators older than 30 days are low confidence
ThreatIntelIndicators
| where TimeGenerated > ago(30d)
| summarize
    TotalIndicators = count(),
    FreshIndicators = countif(ExpirationDateTime > now()),
    StaleIndicators = countif(ExpirationDateTime <= now()),
    OldestExpiry = min(ExpirationDateTime),
    NewestExpiry = max(ExpirationDateTime)
    by SourceSystem
```

### Level 4 — Check indicator type coverage
```kql
// Different indicator types cover different detection scenarios
// IP only = network detection only
// URL + Domain = web and DNS detection
// FileHash = endpoint detection
// Full coverage = all types present
ThreatIntelIndicators
| where TimeGenerated > ago(30d)
| where ExpirationDateTime > now()
| summarize Count = count() by IndicatorType, SourceSystem
| order by SourceSystem asc, Count desc
```

### Level 5 — Verify TI analytics rules are referencing correct tables
```kql
// ThreatIntelligenceIndicator is the LEGACY table
// ThreatIntelIndicators is the CURRENT table
// Rules referencing the legacy table still work but should be updated
// Check if legacy table still has data
ThreatIntelligenceIndicator
| where TimeGenerated > ago(7d)
| summarize Count = count(), LastSeen = max(TimeGenerated)
```

---

## ThreatIntelObjects

### Level 1 — What feeds are writing STIX objects here?
```kql
ThreatIntelObjects
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by SourceSystem
| order by Count desc
```

### Level 2 — What STIX object types are present?
```kql
// STIX 2.1 supports rich object types beyond just indicators
// Threat actors, attack patterns, campaigns, relationships
ThreatIntelObjects
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by ObjectType, SourceSystem
| order by Count desc
```

### Level 3 — Check relationship objects
```kql
// Relationship objects connect threat actors to techniques
// High value for understanding adversary TTPs
ThreatIntelObjects
| where TimeGenerated > ago(30d)
| where ObjectType == "relationship"
| summarize Count = count() by SourceSystem
```

---

## ASimNetworkSessionLogs

### Level 1 — What sources are normalized into this table?
```kql
ASimNetworkSessionLogs
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by EventVendor, EventProduct
| order by Count desc
```

### Level 2 — What event types and actions are present?
```kql
ASimNetworkSessionLogs
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by EventVendor, EventType, DvcAction
| order by Count desc
```

### Level 3 — Verify normalization quality
```kql
// Check that key ASIM fields are populated
// Empty fields indicate parser issues
ASimNetworkSessionLogs
| where TimeGenerated > ago(24h)
| summarize
    TotalRecords = count(),
    HasSrcIP = countif(isnotempty(SrcIpAddr)),
    HasDstIP = countif(isnotempty(DstIpAddr)),
    HasDstPort = countif(isnotempty(DstPortNumber)),
    HasAction = countif(isnotempty(DvcAction))
| extend
    SrcIPCoverage = round(todouble(HasSrcIP) / TotalRecords * 100, 1),
    DstIPCoverage = round(todouble(HasDstIP) / TotalRecords * 100, 1)
```
// Low coverage percentages indicate parser misconfiguration

---

## ASimAuditEventLogs

### Level 1 — What sources are normalized here?
```kql
ASimAuditEventLogs
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by EventVendor, EventProduct
| order by Count desc
```

### Level 2 — What audit event types are present?
```kql
ASimAuditEventLogs
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by EventVendor, EventType, EventResult
| order by Count desc
```

### Level 3 — Verify normalization quality
```kql
ASimAuditEventLogs
| where TimeGenerated > ago(24h)
| summarize
    TotalRecords = count(),
    HasActorUsername = countif(isnotempty(ActorUsername)),
    HasObject = countif(isnotempty(Object)),
    HasEventResult = countif(isnotempty(EventResult))
| extend ActorCoverage = round(todouble(HasActorUsername) / TotalRecords * 100, 1)
```

---

## ASimWebSessionLogs

### Level 1 — What sources are normalized here?
```kql
ASimWebSessionLogs
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by EventVendor, EventProduct
| order by Count desc
```

### Level 2 — What web session types are present?
```kql
ASimWebSessionLogs
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by EventVendor, EventType, DvcAction, HttpStatusCode
| order by Count desc
```

### Level 3 — Check for blocked vs allowed traffic
```kql
ASimWebSessionLogs
| where TimeGenerated > ago(7d)
| summarize Count = count() by DvcAction
| order by Count desc
```

---

## SecurityEvent

### Level 1 — What machines are writing here?
```kql
SecurityEvent
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by Computer
| order by Count desc
```

### Level 2 — What Event IDs are present?
```kql
SecurityEvent
| where TimeGenerated > ago(30d)
| summarize Count = count() by EventID
| order by Count desc
```

### Level 3 — Check critical Event ID coverage
```kql
// These Event IDs should be present in any active Windows environment
// Missing IDs indicate DCR or audit policy gaps
let critical_ids = datatable(EventID:int, Description:string)
[
    4624, "Successful logon",
    4625, "Failed logon",
    4648, "Logon with explicit credentials",
    4672, "Special privileges assigned",
    4688, "New process created",
    4698, "Scheduled task created",
    4720, "User account created",
    4728, "Member added to global group",
    4732, "Member added to local group",
    4756, "Member added to universal group",
    4740, "Account locked out",
    4771, "Kerberos pre-auth failed",
    4776, "NTLM credential validation",
    4768, "Kerberos TGT requested",
    4769, "Kerberos service ticket requested"
];
let present_ids = SecurityEvent
    | where TimeGenerated > ago(24h)
    | summarize by EventID;
critical_ids
| join kind=leftanti present_ids on EventID
| project
    EventID,
    Description,
    Status = "MISSING — not being collected"
```
// Any row returned here is a gap in your Windows security coverage

### Level 4 — Check if process creation has command line
```kql
// 4688 without CommandLine is significantly less valuable
// CommandLine requires audit policy to be configured in Windows
SecurityEvent
| where TimeGenerated > ago(24h)
| where EventID == 4688
| summarize
    TotalProcessCreation = count(),
    HasCommandLine = countif(isnotempty(CommandLine))
| extend CommandLineCoverage = round(todouble(HasCommandLine) / TotalProcessCreation * 100, 1)
```
// If CommandLineCoverage is 0% — process creation auditing is
// enabled but CommandLine logging is not configured in Windows
// audit policy. Significant detection gap.

---

## Operation

### Level 1 — What operation categories are present?
```kql
Operation
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by OperationCategory, Solution
| order by Count desc
```

### Level 2 — Check for error patterns
```kql
// Operation table contains workspace errors and warnings
// Recurring errors may indicate ingestion or agent problems
Operation
| where TimeGenerated > ago(7d)
| where OperationStatus != "Succeeded"
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by OperationCategory, Detail
| order by Count desc
```

### Notes on Operation:
// Low security value but useful for troubleshooting
// Errors here often explain why other tables have gaps
// Check when a table suddenly goes inactive

---

## Salesforce_ListViewEvent and Salesforce_LoginEvent

### Level 1 — Check if multiple Salesforce orgs write here
```kql
// Replace table name as needed
Salesforce_ListViewEvent
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by OrganizationId
| order by Count desc
```

### Level 2 — Check event types within the table
```kql
Salesforce_LoginEvent
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by LoginType, LoginStatus
| order by Count desc
```

### Level 3 — Check for failed logins
```kql
// Failed Salesforce logins are high value for detecting
// credential attacks against the platform
Salesforce_LoginEvent
| where TimeGenerated > ago(30d)
| where LoginStatus != "Success"
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated)
    by LoginStatus, LoginType
| order by Count desc
```

---

## Heartbeat

### Level 1 — What machines are sending heartbeats?
```kql
Heartbeat
| where TimeGenerated > ago(30d)
| summarize
    Count = count(),
    LastSeen = max(TimeGenerated),
    LastHeartbeat = max(TimeGenerated)
    by Computer, OSType, Category, Version
| order by LastSeen desc
```

### Level 2 — Check for machines that recently stopped
```kql
// Machines that sent heartbeats in last 30 days but
// not in the last 24 hours — potential agent failure
Heartbeat
| where TimeGenerated > ago(30d)
| summarize LastSeen = max(TimeGenerated) by Computer
| where LastSeen < ago(24h)
| extend HoursSince = datetime_diff('hour', now(), LastSeen)
| order by HoursSince desc
```

### Level 3 — Check agent versions
```kql
// Outdated AMA versions can cause ingestion issues
Heartbeat
| where TimeGenerated > ago(24h)
| summarize LastSeen = max(TimeGenerated) by Computer, Version
| order by Version asc
```

---

## General — Verify Table Health After Investigation

After completing the investigation for any table run this
final health check to confirm current status:

```kql
// Universal table health check
// Replace TABLE_NAME with actual table
TABLE_NAME
| where TimeGenerated > ago(24h)
| summarize
    RecordCount = count(),
    LastRecord = max(TimeGenerated),
    HoursSinceLastRecord = datetime_diff('hour', now(), max(TimeGenerated))
```

---

## The Deep Dive Principle

Every shared table investigation should answer these questions
before you finalize the spreadsheet rows:

1. How many distinct sources write to this table?
2. What categories or event types does each source send?
3. Are all expected categories present or are some missing?
4. What is the security value of each category?
5. Should missing categories be flagged as gaps?
6. Does the table need one row or multiple rows?

The Azure Firewall example is the model:
- Surface level: one source, AZUREFIREWALLS
- Deep level: four categories, each different detection coverage
- DnsProxy missing = DNS blind spot not visible at surface level
- Correct documentation: four rows or one row with detailed Notes

Apply this thinking to every shared table. What looks like
one thing at the surface is often multiple things underneath.
Understanding what is underneath is what separates a real audit
from a checkbox exercise.
