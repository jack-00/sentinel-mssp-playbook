# Layer 1 — Universal Sentinel Baseline Checklist

> **Purpose:** Every Microsoft Sentinel workspace we manage must be validated against this checklist regardless of client size, industry, or onboarding date. This is the non-negotiable foundation. Nothing here is optional. If an item does not apply, it must be documented with a reason — not silently skipped.
>
> **Philosophy:** The most dangerous gaps are silent ones. A detection rule with bad logic fires incorrectly and you notice. A missing data source or disabled setting produces nothing — and you don't notice until a client does.
>
> **North Star Principle:** For every Microsoft capability and data source that exists in this ecosystem, we make a deliberate decision: *enabled and configured*, or *consciously excluded with a documented reason*. Nothing is simply forgotten.
>
> **Last Validated:** April 2026
> **Portal Reference:** Microsoft Defender Portal (security.microsoft.com) — Azure portal retirement deadline: **March 31, 2027**

---

## How to Use This Checklist

Each section follows a consistent structure:

- **What it is** — plain explanation of the feature or setting
- **What data it brings / enables** — the tables, signals, or capabilities it unlocks
- **Why it matters** — the detection and investigation impact
- **MITRE ATT&CK coverage** — the tactics this data source helps detect
- **Detections that depend on this** — the category of rules that go blind without it
- **Checklist items** — pass/fail verification steps

Work through sections in order. Complete one section fully before moving to the next. Document any exception with a date, reason, and owner.

---

## Section 1 — Defender Portal Migration Status

### What it is
Microsoft Sentinel is moving permanently to the Microsoft Defender portal (security.microsoft.com). The Azure portal experience retires March 31, 2027. This is not optional — it is a deadline. Clients onboarded to Sentinel after July 1, 2025 may already be auto-enrolled in the Defender portal.

### Why it matters
New features — including multi-tenant content distribution, AI-powered incident triage, Security Copilot integration, UEBA behaviors layer, and advanced multi-stage incident correlation — are shipping exclusively in the Defender portal. Clients still on the Azure portal are missing capabilities they are paying for and falling behind on the migration path.

### What changes at migration
- The XDR data connector is automatically enabled and configured
- Individual Defender product connectors (MDE, MDO, MDI, MDA) are automatically disconnected — they may still *appear* connected but data stops flowing through them
- Incident titles are controlled by the XDR correlation engine — automation rules using incident name as a condition will break (use tags instead)
- Alert schema changes — existing KQL analytics rules must be validated against XDR connector schema differences before migration
- The SecurityInsights API for incident management must migrate to Microsoft Graph API for unified incidents (impacts SOAR integrations and Logic Apps)

### Checklist
- [ ] Defender portal migration status documented for this client (migrated / in progress / not started)
- [ ] If migrated: XDR data connector confirmed active and syncing incidents bi-directionally
- [ ] If migrated: Individual Defender product connectors confirmed disconnected (not just appearing connected with no data flowing)
- [ ] If migrated: All existing KQL analytics rules reviewed against XDR connector alert schema differences
- [ ] If migrated: Automation rules audited — any using incident name as a trigger condition updated to use tags
- [ ] If migrated: Logic App pipeline validated — incidents still flowing correctly through to ServiceNow
- [ ] If not yet migrated: Target migration date documented and communicated

---

## Section 2 — Microsoft First-Party Data Connectors

> **Key principle:** These connectors exist inside the client's Microsoft ecosystem already. Most have zero additional licensing cost beyond what the client already pays. There is almost never a justifiable reason not to connect them. An unconnected Microsoft connector is a silent blind spot.

---

### 2.1 Microsoft Defender XDR Connector ⭐ CRITICAL

**What it is:** The unified connector that ingests all Microsoft Defender XDR incidents, alerts, and advanced hunting events into Sentinel. This single connector replaces all individual Defender product connectors for alert and incident data.

**What data it brings:**
- Unified XDR incidents (multi-stage, correlated across all Defender products)
- Alerts from Defender for Endpoint, Office 365, Identity, and Cloud Apps
- Entities enriched with cross-product context
- Advanced hunting events (raw telemetry) from Defender components
- Bi-directional incident sync between Sentinel and the Defender portal

**Why it matters:** This is the mailbomb lesson. Without this connector, any incident detected and correlated by XDR is completely invisible to Sentinel. Your Logic App never fires. No ticket reaches ServiceNow. The client sees a critical incident in their XDR portal and your SOC has no awareness of it whatsoever. As Microsoft continues shifting correlation and multi-stage detection into the XDR engine, this gap grows larger over time — not smaller.

**MITRE ATT&CK Coverage Enabled:**
| Tactic | Example Techniques |
|---|---|
| Initial Access | T1566 Phishing, T1078 Valid Accounts |
| Execution | T1059 Command and Scripting Interpreter |
| Persistence | T1098 Account Manipulation |
| Credential Access | T1110 Brute Force, T1556 Modify Auth Process |
| Lateral Movement | T1021 Remote Services, T1550 Pass the Hash/Ticket |
| Exfiltration | T1048 Exfiltration Over Alternative Protocol |
| Impact | T1486 Data Encrypted for Impact (Ransomware) |

**Detections that go blind without this:**
- Any detection relying on `SecurityAlert` or `SecurityIncident` from XDR products
- Multi-stage incident correlation rules
- Any custom content referencing Defender-sourced alerts

**Checklist:**
- [ ] Microsoft Defender XDR solution installed from Content Hub
- [ ] XDR connector enabled — incidents and alerts syncing
- [ ] Bi-directional sync confirmed — status updates in Sentinel reflect in Defender portal
- [ ] "Turn off all Microsoft incident creation rules for these products" checkbox enabled (prevents duplicate incidents)
- [ ] Entities connection enabled (syncs on-prem AD identities via Defender for Identity)
- [ ] Advanced hunting events connection evaluated and configured if raw telemetry is needed
- [ ] Validation query confirmed returning results:
  ```kql
  SecurityIncident
  | where ProviderName == "Microsoft XDR"
  | take 10
  ```

---

### 2.2 Microsoft Entra ID Connector ⭐ CRITICAL

**What it is:** The connector for the client's identity platform — the most attacked surface in any Microsoft environment. This is separate from the XDR connector. Entra ID logs are identity telemetry, not XDR alerts.

**What data it brings:**
- `SignInLogs` — interactive user sign-ins
- `AADNonInteractiveUserSignInLogs` — non-interactive sign-ins (service-to-service, background auth)
- `AuditLogs` — directory changes, group membership, role assignments, app registrations
- `AADProvisioningLogs` — account provisioning events
- `AADServicePrincipalSignInLogs` — service principal and managed identity authentication
- `AADManagedIdentitySignInLogs` — managed identity sign-ins
- `AADRiskyUsers` — Identity Protection risk flags
- `AADUserRiskEvents` — specific risk detections (leaked credentials, impossible travel, etc.)

**Why it matters:** Identity is the primary attack vector in cloud environments. Credential stuffing, password spray, MFA fatigue, impossible travel, token theft, and OAuth abuse are all invisible without Entra ID logs. The overwhelming majority of meaningful detection rules in the Microsoft Content Hub depend on `SignInLogs` or `AuditLogs`. This is also required for UEBA to function properly.

**Important:** The non-interactive and service principal sign-in logs are commonly missed. Attackers frequently abuse service principals and managed identities because they are often overlooked. Do not enable only interactive sign-in logs.

**MITRE ATT&CK Coverage Enabled:**
| Tactic | Example Techniques |
|---|---|
| Initial Access | T1078 Valid Accounts, T1566 Phishing (MFA fatigue) |
| Credential Access | T1110 Brute Force, T1606 Forge Web Credentials, T1528 Steal Application Access Token |
| Privilege Escalation | T1098 Account Manipulation, T1548 Abuse Elevation Control |
| Persistence | T1136 Create Account, T1098.003 Additional Cloud Roles |
| Defense Evasion | T1550 Use Alternate Auth Material |
| Discovery | T1087 Account Discovery, T1069 Permission Groups Discovery |

**Detections that go blind without this:**
- Password spray and brute force detections
- Impossible travel and sign-in anomaly rules
- MFA fatigue and legacy authentication detections
- Privileged role assignment alerting
- OAuth consent grant abuse detections
- All UEBA identity baselines

**Checklist:**
- [ ] Microsoft Entra ID connector enabled
- [ ] Interactive sign-in logs (`SignInLogs`) streaming
- [ ] Non-interactive sign-in logs (`AADNonInteractiveUserSignInLogs`) streaming
- [ ] Audit logs (`AuditLogs`) streaming
- [ ] Service principal sign-in logs (`AADServicePrincipalSignInLogs`) streaming
- [ ] Managed identity sign-in logs (`AADManagedIdentitySignInLogs`) streaming
- [ ] Identity Protection risk events streaming (`AADRiskyUsers`, `AADUserRiskEvents`)
- [ ] Provisioning logs streaming (`AADProvisioningLogs`)
- [ ] Validation — confirm data is flowing:
  ```kql
  SignInLogs
  | where TimeGenerated > ago(1h)
  | take 10
  ```

---

### 2.3 Microsoft Defender for Cloud Connector ⭐ CRITICAL

**What it is:** Cloud security posture management and workload protection for Azure resources. This connector is **not** replaced by the XDR connector. It must be enabled separately. Without it, Defender for Cloud incidents that flow through XDR will appear in Sentinel as empty shells — the incidents exist but contain no alerts or entities.

**What data it brings:**
- Security alerts for Azure resources (VMs, storage accounts, Key Vault, SQL, containers)
- Security recommendations and posture assessments
- Regulatory compliance assessments
- Defender for Cloud incidents (when paired with XDR connector)

**Why it matters:** Attackers who gain cloud access frequently pivot to Azure resources. Key Vault secret access, storage account enumeration, privilege escalation through Azure RBAC, and container escape events all generate Defender for Cloud alerts. Without this connector those alerts stay isolated in the Defender for Cloud portal and never reach your pipeline.

**MITRE ATT&CK Coverage Enabled:**
| Tactic | Example Techniques |
|---|---|
| Collection | T1530 Data from Cloud Storage |
| Credential Access | T1552 Unsecured Credentials (Key Vault access) |
| Privilege Escalation | T1098 Account Manipulation (RBAC abuse) |
| Discovery | T1526 Cloud Service Discovery |
| Exfiltration | T1537 Transfer Data to Cloud Account |
| Impact | T1485 Data Destruction, T1496 Resource Hijacking |

**Checklist:**
- [ ] Microsoft Defender for Cloud connector enabled (tenant-based, not legacy subscription-based)
- [ ] All relevant Azure subscriptions included
- [ ] Alerts streaming and visible in Sentinel incident queue
- [ ] Validation:
  ```kql
  SecurityAlert
  | where ProductName == "Azure Security Center"
  | where TimeGenerated > ago(24h)
  | take 10
  ```

---

### 2.4 Azure Activity Connector ⭐ CRITICAL

**What it is:** The management plane log for Azure — every action taken against Azure resources through the portal, CLI, PowerShell, or API. This covers administrative operations, not resource-level data.

**What data it brings (`AzureActivity` table):**
- Resource creation, modification, and deletion
- Role assignment changes (RBAC)
- Policy creation and modification
- Virtual machine start/stop/delete
- Network security group rule changes
- Key Vault access policy changes
- Subscription-level administrative actions

**Why it matters:** This is how you catch compromised accounts or attackers making changes to the Azure environment itself. Creating backdoor accounts, exporting VM disks, modifying firewall rules, assigning elevated roles — all of this is Azure Activity. It is also the primary log for answering "who changed what and when" during incident response.

**Important:** Connect all subscriptions, not just the one Sentinel lives in. Clients often have multiple subscriptions and activity in non-Sentinel subscriptions is invisible without explicitly connecting each one.

**MITRE ATT&CK Coverage Enabled:**
| Tactic | Example Techniques |
|---|---|
| Persistence | T1098 Account Manipulation, T1136 Create Account |
| Privilege Escalation | T1548 Abuse Elevation Control Mechanism |
| Defense Evasion | T1562 Impair Defenses (disabling policies/logging) |
| Discovery | T1526 Cloud Service Discovery, T1069 Permission Groups Discovery |
| Impact | T1485 Data Destruction, T1496 Resource Hijacking |

**Checklist:**
- [ ] Azure Activity connector enabled
- [ ] All client Azure subscriptions connected — not just the Sentinel subscription
- [ ] Administrative operations queryable in Sentinel
- [ ] Validation:
  ```kql
  AzureActivity
  | where TimeGenerated > ago(1h)
  | take 10
  ```

---

### 2.5 Microsoft 365 (Office Activity / Unified Audit Log) Connector ⭐ CRITICAL

**What it is:** The audit log for Microsoft 365 workloads — SharePoint, OneDrive, Exchange, Teams, and M365 admin operations. **Critical prerequisite: the Unified Audit Log must be explicitly enabled in the Microsoft 365 compliance portal before this connector will produce any data. It is not on by default in all tenants.**

**What data it brings (`OfficeActivity` table):**
- SharePoint and OneDrive file access, sharing, and download events
- Exchange mailbox access and forwarding rule creation
- Teams messages and meeting events (if configured)
- Admin privilege changes in M365 admin center
- OAuth application consent grants
- M365 compliance and DLP events

**Why it matters:** This is where you see data exfiltration through SharePoint and OneDrive, suspicious mailbox access and email forwarding rules (a classic persistence technique), OAuth consent attacks, and Teams-based social engineering. It is also a compliance requirement for many clients. The Unified Audit Log not being enabled is one of the most common silent gaps found in production environments.

**MITRE ATT&CK Coverage Enabled:**
| Tactic | Example Techniques |
|---|---|
| Initial Access | T1566 Phishing (via Teams/email) |
| Persistence | T1137 Office Application Startup, T1098 Account Manipulation (forwarding rules) |
| Collection | T1114 Email Collection, T1213 Data from Information Repositories |
| Exfiltration | T1048 Exfiltration Over Alternative Protocol, T1567 Exfiltration to Cloud Storage |
| Defense Evasion | T1550.001 Application Access Token (OAuth abuse) |

**Checklist:**
- [ ] Unified Audit Log confirmed **enabled** in M365 compliance portal (do not assume — verify)
- [ ] Microsoft 365 connector enabled in Sentinel
- [ ] SharePoint, Exchange, and Teams audit events flowing
- [ ] Validation:
  ```kql
  OfficeActivity
  | where TimeGenerated > ago(1h)
  | take 10
  ```

---

### 2.6 Microsoft Defender Threat Intelligence (MDTI) Connector

**What it is:** Microsoft's threat intelligence feed integrated directly into Sentinel, providing indicators of compromise (IOCs) — malicious IPs, domains, URLs, and file hashes — sourced from Microsoft's global threat research.

**What data it brings:**
- `ThreatIntelIndicators` — legacy table (being retired July 31, 2025 for new ingestion)
- `ThreatIntelligenceIndicator` — current primary table
- `ThreatIntelObjects` — new STIX 2.1 schema table supporting threat actors, attack patterns, relationships

**Why it matters:** Threat intelligence turns raw log data into actionable detections by matching known-bad indicators against your environment's network traffic, DNS, and authentication logs. Without a TI feed, your detections are entirely behavior-based and miss known infrastructure.

**Note:** As of mid-2025, Microsoft Sentinel ingests STIX objects into the new `ThreatIntelIndicators` and `ThreatIntelObjects` tables. Verify your analytics rules reference the correct current table.

**Checklist:**
- [ ] Microsoft Defender Threat Intelligence connector enabled (standard or premium tier based on license)
- [ ] TI indicators appearing in `ThreatIntelligenceIndicator` table
- [ ] Analytics rules referencing TI are using current table names (not deprecated legacy tables)
- [ ] Threat intelligence workbook deployed from Content Hub

---

## Section 3 — Audit and Diagnostic Log Enablement

> These are not Sentinel connectors — they are settings that must be manually enabled in Azure and Microsoft 365 before data can flow. This is the most commonly missed category in production environments. These settings are frequently off by default.

---

### 3.1 Sentinel Health and Audit Logging ⭐ CRITICAL

**What it is:** Sentinel's own health and audit logging — tracking the operational health of your analytics rules, data connectors, and automation, as well as recording who made what changes to the Sentinel environment itself.

**What data it brings:**
- `SentinelHealth` — health status of analytics rules, data connectors, and automation rules. Flags rules that stopped running, connectors that went silent, and playbooks that failed
- `SentinelAudit` — records every configuration change made to the Sentinel workspace: who created, modified, or deleted analytics rules, automation rules, watchlists, and connectors

**Why it matters:** Without this you have no visibility into silent failures. An analytics rule that stopped running due to a KQL timeout produces no errors and no alerts — it just silently stops working. A connector that went unhealthy three weeks ago shows as "connected" in the UI but has ingested nothing. This table is how you catch those failures before a client does.

**How to enable:** Sentinel → Settings → Settings tab → Auditing and health monitoring → Enable

**Checklist:**
- [ ] Auditing and health monitoring enabled in Sentinel settings
- [ ] `SentinelHealth` table confirmed populating
- [ ] `SentinelAudit` table confirmed populating
- [ ] Alert rule created to fire when analytics rules report unhealthy status:
  ```kql
  SentinelHealth
  | where SentinelResourceType == "Analytics Rule"
  | where Status !in ("Success", "Informational")
  ```
- [ ] Alert rule created to fire when connector health degrades significantly

---

### 3.2 Microsoft Entra ID Diagnostic Settings

**What it is:** Azure-level diagnostic settings that route Entra ID log categories to the Log Analytics workspace. These must be configured in Entra ID — enabling the Sentinel connector alone is not sufficient.

**Required log categories:**
- `SignInLogs`
- `AuditLogs`
- `NonInteractiveUserSignInLogs`
- `ServicePrincipalSignInLogs`
- `ManagedIdentitySignInLogs`
- `ProvisioningLogs`
- `ADFSSignInLogs` (if ADFS is in use)
- `RiskyUsers`
- `UserRiskEvents`

**Checklist:**
- [ ] Entra ID diagnostic settings configured in Azure portal → Entra ID → Monitoring → Diagnostic settings
- [ ] All required log categories routed to the correct Log Analytics workspace
- [ ] Each category confirmed flowing with a validation query

---

### 3.3 Azure Subscription Diagnostic Settings

**What it is:** Subscription-level diagnostic settings that route the Azure Activity log to Sentinel. Must be configured per subscription.

**Checklist:**
- [ ] Diagnostic settings configured at subscription level for each client subscription
- [ ] Activity log routed to Sentinel Log Analytics workspace
- [ ] Administrative, Security, and Policy log categories included

---

### 3.4 Microsoft 365 Unified Audit Log

**What it is:** M365 compliance portal setting that enables audit logging across all M365 workloads. Must be explicitly turned on — it is not enabled by default in all tenants.

**How to verify:** M365 Compliance Portal → Audit → check if audit log search is enabled. Or run in Exchange Online PowerShell: `Get-AdminAuditLogConfig | Select UnifiedAuditLogIngestionEnabled`

**Checklist:**
- [ ] Unified Audit Log confirmed enabled in M365 compliance portal
- [ ] Audit log retention period reviewed and aligned with client compliance requirements
- [ ] Exchange mailbox auditing enabled (separate from UAL — required for mailbox-level events)

---

### 3.5 Azure Resource Diagnostic Logs (Key Services)

**What it is:** Resource-level diagnostic settings for high-value Azure services that generate security-relevant logs.

**Priority resources:**
- **Azure Key Vault** — secret access, key operations, certificate events. Required to detect credential theft from Key Vault
- **Azure Storage Accounts** — blob access, anonymous access, unusual download patterns
- **Network Security Groups** — flow logs for network traffic visibility
- **Azure Kubernetes Service (AKS)** — control plane and audit logs if containers are in scope
- **Azure SQL / Cosmos DB** — query logs and access events if databases are in scope
- **Azure App Service** — HTTP logs if web apps are in scope

**Checklist:**
- [ ] Key Vault diagnostic logging enabled and routing to Sentinel workspace
- [ ] Storage Account logging enabled for security-relevant accounts
- [ ] NSG flow logs enabled (requires Network Watcher)
- [ ] Additional resource logging enabled based on client's Azure footprint (documented per client)

---

## Section 4 — UEBA (User and Entity Behavior Analytics)

### What it is
UEBA analyzes logs from connected data sources to build behavioral baselines for users, hosts, IP addresses, and applications using machine learning. It flags deviations from normal behavior — not rule-based alerts, but ML-identified anomalies.

**New as of 2025-2026:** Microsoft has introduced the **UEBA Behaviors Layer** — a new feature that aggregates and sequences events into meaningful patterns, provides explainability, and maps behaviors directly to the MITRE ATT&CK framework. This is a significant enhancement that makes UEBA output far more actionable.

### What data it brings
- `BehaviorAnalytics` — anomaly detections with investigation priority scores (0-10)
- `IdentityInfo` — enriched user identity data from Entra ID and on-prem AD (via MDI)
- `UserAccessAnalytics` — unusual resource access patterns
- UEBA enrichments on entity pages — activity timelines, peer comparisons, investigation priority scores

### Why it matters
Security alerts tell you something happened. UEBA tells you whether what happened is normal for that entity. A sign-in from an unusual location is noise. A sign-in from an unusual location *for that specific user, at that specific time of day, to a resource they have never accessed, while their peers show no similar activity* — that is a high-confidence signal. UEBA is what turns raw alerts into prioritized investigations.

**Important operational note:** UEBA requires 14-21 days to build behavioral baselines. Do not evaluate it in the first three weeks. Accuracy improves over time — the longer it runs, the better it gets.

### UEBA Data Sources to Enable
Enable all applicable sources:
- Microsoft Entra ID sign-in logs
- Microsoft Entra ID audit logs
- Azure Activity
- Security Events (Windows)
- Defender for Endpoint device logon events

### MITRE ATT&CK Coverage Enabled
| Tactic | Example Behaviors |
|---|---|
| Credential Access | Unusual authentication patterns, impossible travel |
| Lateral Movement | First-time access to new hosts, unusual remote service use |
| Collection | Unusual data access volume, first-time sensitive resource access |
| Exfiltration | Abnormal outbound data patterns |
| Privilege Escalation | Unusual role access, first-time admin action |
| Insider Threat | Peer group anomalies, off-hours access patterns |

### Checklist
- [ ] UEBA feature enabled: Settings → Entity behavior analytics → Turn on UEBA
- [ ] All applicable data sources connected to UEBA
- [ ] Directory services sync configured (Entra ID, and on-prem AD via MDI if applicable)
- [ ] Anomaly detection enabled: Settings → Anomalies → Detect Anomalies toggled on
- [ ] UEBA Behaviors Layer enabled (new 2025-2026 feature — map behaviors to MITRE ATT&CK)
- [ ] UEBA Essentials solution installed from Content Hub (pre-built hunting queries)
- [ ] `BehaviorAnalytics` table confirmed populating after baseline period (14-21 days)
- [ ] `IdentityInfo` table confirmed populating
- [ ] SOC optimization cards reviewed — UEBA recommends additional tables to onboard for better coverage
- [ ] Baseline period noted: UEBA enabled date documented so team knows when baselines become reliable

---

## Section 5 — Analytics Rules and Content Hub

### What it is
The Microsoft Sentinel Content Hub provides packaged security solutions containing analytics rules, workbooks, hunting queries, and playbooks maintained by Microsoft. These are the out-of-the-box detections that should be the baseline before any custom content is written.

### Why it matters
Starting with a blank analytics rule list means reinventing wheels that Microsoft has already built and maintains. Content Hub solutions are MITRE-mapped, regularly updated, and tied to the data sources you are connecting. They are the fastest path to detection coverage.

### Checklist

**Solutions to install for every client (based on connected data sources):**
- [ ] Microsoft Defender XDR solution installed
- [ ] Microsoft Entra ID solution installed
- [ ] Azure Activity solution installed
- [ ] Microsoft 365 solution installed
- [ ] Threat Intelligence solution installed
- [ ] UEBA Essentials solution installed
- [ ] Sentinel SOAR Essentials solution installed
- [ ] Microsoft Defender for Cloud solution installed (if applicable)
- [ ] Windows Security Events solution installed (if on-prem Windows in scope)

**Analytics rule configuration:**
- [ ] All installed solution analytics rules reviewed — enabled or explicitly disabled with documented reason
- [ ] Microsoft incident creation rules evaluated — if XDR connector is active, incident creation rules for XDR products should be turned off to prevent duplicates
- [ ] Near Real-Time (NRT) rules evaluated and enabled where applicable — NRT rules run every minute vs. scheduled rules which may run every 5 minutes or longer
- [ ] Scheduled analytics rules reviewed for appropriate run frequency and lookback periods
- [ ] Alert severity thresholds reviewed — default severities may not match client risk tolerance
- [ ] Entity mapping configured on all analytics rules — entities must be mapped for UEBA enrichment and investigation graph to work correctly
- [ ] MITRE ATT&CK mapping confirmed on all custom analytics rules

**Analytics rule health (requires Section 3.1 to be complete):**
- [ ] `SentinelHealth` monitored for any analytics rules in non-success state
- [ ] KQL schema validated post-XDR migration — rules using standalone connector schemas updated to XDR connector schemas where applicable

---

## Section 6 — Watchlist Standards

### What it is
Watchlists are structured reference data stored in Sentinel that analytics rules, hunting queries, and workbooks can query at runtime. They are the mechanism for making your detections context-aware — knowing the difference between a privileged admin signing in from an unusual location versus a regular user doing the same thing.

### Why it matters
Without watchlists, your detections treat all users, all hosts, and all IPs equally. A sign-in at 2am from a Domain Controller is expected behavior for a backup service account. Without a watchlist flagging that account as an exception, your detection fires every night and becomes noise. Watchlists are what separate a well-tuned environment from an alert fatigue factory.

### Standard Watchlist Schemas (deploy for every client)

**VIP and Privileged Users**
- Fields: `UserPrincipalName`, `DisplayName`, `Role`, `Department`, `RiskLevel`
- Purpose: Elevate alert severity and priority for actions involving high-value accounts

**Critical Assets / High-Value Hosts**
- Fields: `Hostname`, `IPAddress`, `AssetType`, `CriticalityLevel`, `Owner`
- Purpose: Elevate alert severity for events involving critical systems (Domain Controllers, PAM servers, finance systems)

**Domain Controllers**
- Fields: `Hostname`, `IPAddress`, `Domain`, `Site`
- Purpose: Required for many identity and lateral movement detections

**Network Ranges / Trusted IP Ranges**
- Fields: `IPRange`, `CIDR`, `Description`, `Type` (corporate/VPN/vendor)
- Purpose: Distinguish internal traffic from external, reduce false positives for known IP ranges

**Terminated Employees**
- Fields: `UserPrincipalName`, `TerminationDate`, `Department`
- Purpose: Alert on any authentication activity from accounts that should be disabled

**Service Accounts**
- Fields: `UserPrincipalName`, `AccountPurpose`, `ExpectedSourceIP`, `ExpectedLoginHours`
- Purpose: Flag service account behavior that deviates from expected baseline (different source IP, off-hours activity)

**Vendor / Third-Party Access**
- Fields: `VendorName`, `UserPrincipalName`, `ExpectedIPRange`, `AccessWindow`
- Purpose: Monitor and alert on vendor access outside expected parameters

### Checklist
- [ ] All standard watchlist schemas deployed
- [ ] Watchlists populated with current client data (not empty shells)
- [ ] Watchlists referenced in relevant analytics rules — confirm rules actually use them
- [ ] Watchlist data owner documented per client — someone responsible for keeping them current
- [ ] Watchlist review cadence established (minimum quarterly)
- [ ] Silent log source detections created for each watchlist-dependent data source

---

## Section 7 — Automation and SOAR Baseline

### What it is
Automation rules and playbooks (Logic Apps) are the SOAR layer of Sentinel — automating triage, enrichment, notification, and ticket creation so that analyst time is spent on decisions, not manual steps.

### Standard Playbooks (deploy for every client)

**Incident notification**
- Trigger: New incident created at High or Critical severity
- Action: Post to SOC Teams channel or send email alert with incident link and summary

**Entity enrichment**
- Trigger: New incident created
- Action: Geo-lookup on IP entities, VirusTotal hash check if licensed, WHOIS on domain entities
- Output: Enrichment comments added to incident

**ServiceNow ticket creation** *(already in place per current pipeline)*
- Validate: Logic App firing correctly, incident data populated in ticket, severity mapping correct

**Incident auto-tagging**
- Trigger: New incident created
- Action: Tag with client identifier, data source category, MITRE tactic if mapped

**Known false positive auto-close**
- Trigger: Specific analytics rules that generate high-volume known false positives
- Action: Auto-close with documented reason — requires SOC review and approval before deployment

### Checklist
- [ ] Incident notification playbook deployed and tested
- [ ] Entity enrichment playbook deployed and tested
- [ ] ServiceNow ticket creation Logic App validated — incidents flowing correctly
- [ ] Incident auto-tagging automation rule configured — client identifier tag applied to all incidents
- [ ] Automation rules audited for incident name dependencies (see Section 1 — XDR migration impact)
- [ ] Playbook permissions validated — managed identity or service principal has correct Sentinel roles
- [ ] Failed playbook alerts configured — monitor Logic App failure rate

---

## Section 8 — Workspace Settings and Access Control

### What it is
Foundational workspace configuration — retention, RBAC, cost settings, and workspace-level security that must be correct before anything else matters.

### Checklist

**Retention and cost:**
- [ ] Workspace data retention configured appropriately (90-day hot minimum recommended)
- [ ] Archive tier configured if client has compliance retention requirements beyond 90 days
- [ ] Commitment tier pricing evaluated — if ingestion volume justifies it, commitment tier saves cost vs. pay-as-you-go
- [ ] Per-table retention reviewed — XDR-backed tables include 30 days free, extending to 90 adds cost
- [ ] Data ingestion volume monitored — alert configured for unexpected spikes

**Access control:**
- [ ] Lighthouse delegation confirmed active and correct — verify access is scoped appropriately
- [ ] Role assignments audited — least privilege enforced (Sentinel Reader for read-only, Sentinel Responder for analysts, Sentinel Contributor for engineers)
- [ ] No overly broad Contributor or Owner role assignments at workspace level
- [ ] Service principal permissions reviewed if used for automation

**Workspace health:**
- [ ] No Azure resource locks applied to workspace (required for UEBA to function)
- [ ] Workspace in correct Azure region for data residency requirements
- [ ] Diagnostic settings on the workspace itself enabled (management plane activity)

---

## Section 9 — SOC Optimization and MITRE ATT&CK Coverage Review

### What it is
Microsoft Sentinel includes a built-in SOC Optimization feature that analyzes your connected data sources, active analytics rules, and detected threats to recommend improvements. The MITRE ATT&CK page in the Defender portal provides a visual heatmap of your current detection coverage mapped to tactics and techniques.

### Why it matters
This is your north star for measuring detection maturity. The MITRE heatmap shows you at a glance where you have coverage, where you have rules but no data, and where you have no coverage at all. SOC optimization recommendations surface specific actions to improve your posture — including recommending additional UEBA data sources, analytics rules to enable, and data sources to onboard.

### Checklist
- [ ] SOC Optimization page reviewed — all recommendations evaluated and either actioned or documented with reason for deferral
- [ ] MITRE ATT&CK coverage page reviewed in Defender portal — baseline coverage screenshot captured and stored in client folder
- [ ] Tactics with zero coverage identified and documented in client improvement roadmap
- [ ] Tactics with rules but no data identified — these become the data source ingestion conversation with the client
- [ ] Coverage baseline established so future audit decks can show improvement over time
- [ ] SOC Optimization review cadence established (minimum quarterly, aligns with audit deck cycle)

---

## Section 10 — Health Monitoring and Data Validation

### What it is
Ongoing operational health checks that ensure data is actually flowing, rules are actually running, and the pipeline is actually working. This section is about confirming the environment is healthy, not just configured.

### Key validation queries

**Connector ingestion health — last 24 hours by table:**
```kql
Usage
| where TimeGenerated > ago(24h)
| where IsBillable == true
| summarize TotalGB = round(sum(Quantity) / 1000, 2) by DataType
| order by TotalGB desc
```

**Analytics rule health — any non-success states:**
```kql
SentinelHealth
| where SentinelResourceType == "Analytics Rule"
| where Status !in ("Success", "Informational")
| project TimeGenerated, SentinelResourceName, Status, Description
```

**Connector health — last known status:**
```kql
SentinelHealth
| where SentinelResourceType == "Data Connector"
| summarize LastStatus = arg_max(TimeGenerated, Status, Description) by SentinelResourceName
| where LastStatus != "Success"
```

**Confirm XDR incidents are flowing:**
```kql
SecurityIncident
| where ProviderName == "Microsoft XDR"
| where TimeGenerated > ago(7d)
| summarize Count = count() by bin(TimeGenerated, 1d)
```

**Logic App / playbook failure check:**
```kql
SentinelHealth
| where SentinelResourceType == "Automation Rule"
| where Status != "Success"
| project TimeGenerated, SentinelResourceName, Description
```

### Checklist
- [ ] All validation queries run and returning expected results
- [ ] Silent connector alert configured — fires if a critical connector stops ingesting for more than 4 hours
- [ ] Ingestion spike alert configured — fires if daily ingestion exceeds expected volume by significant margin (cost control)
- [ ] Data Collection Health Monitoring workbook deployed from Content Hub
- [ ] Health check validation documented with date and results in client folder
- [ ] Health check review cadence established — minimum monthly

---

## Baseline Completion Sign-Off

| Section | Status | Engineer | Date | Notes |
|---|---|---|---|---|
| 1. Defender Portal Migration | | | | |
| 2. First-Party Data Connectors | | | | |
| 3. Audit and Diagnostic Logs | | | | |
| 4. UEBA | | | | |
| 5. Analytics Rules / Content Hub | | | | |
| 6. Watchlist Standards | | | | |
| 7. Automation and SOAR | | | | |
| 8. Workspace Settings / RBAC | | | | |
| 9. SOC Optimization / MITRE Review | | | | |
| 10. Health Monitoring | | | | |

**Overall Baseline Status:** ⬜ Not Started / 🟡 In Progress / ✅ Complete
**Completed by:**
**Date:**
**Next review date:**

---

*This document is maintained in the sentinel-blueprint repository. Any changes must be committed with a description of what changed and why. The checklist evolves as Microsoft releases new capabilities — review and update minimum once per quarter.*