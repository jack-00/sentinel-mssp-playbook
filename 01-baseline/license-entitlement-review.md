# License and Entitlement Review

> **Purpose:** Before configuring, auditing, or reviewing any client Sentinel environment, you must first understand what the client is licensed for. License tier determines what capabilities are available, what connectors can be enabled, and what features should be configured. An engineer who does not know the client's license tier cannot produce a complete baseline or audit.
>
> **When to complete this:** At onboarding — before the baseline checklist is started. Update whenever the client changes their licensing.
>
> **Where this information surfaces:** The license summary is documented here, referenced in the baseline checklist, and displayed in the Client Security Operations Workbook so any engineer working in the environment has immediate context.
>
> **Last Updated:** April 2026

---

## Why This Matters

Every Sentinel environment looks different on the surface. Some clients have UEBA producing rich behavioral insights. Others have UEBA enabled but no Identity Protection events because they are not licensed for Entra ID P2. Some clients have Defender for Identity sensors on every domain controller. Others have the connector enabled but no license to actually run the product.

The dangerous assumption is that if a connector is available it means the client has the license to use it fully. That is not always true. Microsoft's licensing model means that many features exist in the portal but produce no data or degraded data without the appropriate license.

If you do not know what a client is licensed for you will:

- Enable connectors that produce no data because the underlying product is not licensed
- Miss capabilities the client is paying for but that nobody enabled
- Recommend features the client cannot access
- Build detections that rely on data the client will never have
- Fail to advise clients on what their existing licenses unlock

Knowing the license tier upfront eliminates all of these problems. It tells you exactly what should exist in this environment and what you should be verifying.

---

## The License Review Process

Complete this document at the start of every new client engagement. Work through each section and document what the client has. Use the capability impact table in each section to understand what that license means for the Sentinel environment.

Sources for license information:
- Microsoft 365 Admin Center → Billing → Licenses
- Azure portal → Cost Management → Subscriptions
- Direct from the client IT or procurement team
- Microsoft Partner Center if you have access

---

## Section 1 — Microsoft 365 License Tier

The M365 license tier is the single most impactful licensing decision for a Sentinel environment. It determines which Defender products are included, what identity protection features are available, and what audit log data can be collected.

**Document the client's M365 license:**

| Field | Value |
|---|---|
| Primary M365 License | |
| Number of Seats | |
| Any Mix of License Tiers | |

**Capability impact by tier:**

| License | Sentinel Impact |
|---|---|
| Microsoft 365 Business Basic | No Defender products included. Limited audit log retention. Minimal Sentinel value without additional licensing. |
| Microsoft 365 Business Premium | Includes Defender for Business (simplified MDE), Entra ID P1, Defender for Office 365 Plan 1. Good baseline for SMB environments. |
| Microsoft 365 E3 | Includes Defender for Office 365 Plan 1, Entra ID P1, basic compliance features. No Identity Protection. No Defender for Identity. |
| Microsoft 365 E5 | Full suite — Defender for Office 365 Plan 2, Entra ID P2, Identity Protection, Defender for Identity, Defender for Cloud Apps, Compliance suite. Maximum Sentinel data availability. |
| Microsoft 365 F3 | Frontline worker license. Limited Sentinel relevance — document if present alongside other tiers. |

**What E5 specifically unlocks that E3 does not:**
- Entra ID Identity Protection — AADRiskyUsers and AADUserRiskEvents tables
- Defender for Identity — IdentityLogonEvents, IdentityQueryEvents, IdentityDirectoryEvents tables
- Defender for Office 365 Plan 2 — advanced threat hunting, attack simulation
- Microsoft Defender for Cloud Apps — CloudAppEvents table, OAuth app governance
- Compliance features — advanced audit, communication compliance
- Microsoft Purview — data classification and insider risk

**Engineer note:** If a client has E3 and you see AADRiskyUsers or AADUserRiskEvents in the workspace — investigate. Either they have an Entra ID P2 add-on or there is a licensing question to resolve.

---

## Section 2 — Microsoft Entra ID License

Entra ID licensing is separate from M365 in some environments. Clients may have standalone Entra ID P1 or P2 licenses rather than getting them through an M365 bundle.

**Document the client's Entra ID license:**

| Field | Value |
|---|---|
| Entra ID License Tier | Free / P1 / P2 |
| Included via M365 | Yes / No |
| Standalone Seats | |

**Capability impact by tier:**

| License | Sentinel Impact |
|---|---|
| Entra ID Free | Basic sign-in logs. No Identity Protection. No risk events. Limited UEBA value. |
| Entra ID P1 | Adds Conditional Access. Basic sign-in risk signals. No Identity Protection risk events in Sentinel. |
| Entra ID P2 | Adds Identity Protection — AADRiskyUsers and AADUserRiskEvents tables populate. Risk-based Conditional Access. PIM. Full UEBA identity baseline. |

**What P2 specifically unlocks:**
- `AADRiskyUsers` table — accounts flagged as compromised by Microsoft ML
- `AADUserRiskEvents` table — specific risk detections per sign-in event
- Risk-based Conditional Access signals visible in logs
- Privileged Identity Management audit events

---

## Section 3 — Microsoft Defender Products

Document which Defender products the client is licensed for and whether they are deployed. License and deployment are two different things — a client can be licensed for Defender for Identity but have no sensors deployed on domain controllers.

**Document each Defender product:**

| Product | Licensed | Deployed | Notes |
|---|---|---|---|
| Microsoft Defender for Endpoint P1 | | | |
| Microsoft Defender for Endpoint P2 | | | |
| Microsoft Defender for Office 365 Plan 1 | | | |
| Microsoft Defender for Office 365 Plan 2 | | | |
| Microsoft Defender for Identity | | | |
| Microsoft Defender for Cloud Apps | | | |
| Microsoft Defender for Cloud | | | |
| Microsoft Defender XDR | | | |

**Capability impact:**

| Product | Tables It Populates in Sentinel | What Goes Blind Without It |
|---|---|---|
| Defender for Endpoint P2 | DeviceEvents, DeviceLogonEvents, DeviceProcessEvents, DeviceNetworkEvents, DeviceFileEvents | All endpoint-based detections — malware execution, lateral movement on devices, process anomalies |
| Defender for Office 365 Plan 2 | EmailEvents, EmailAttachmentInfo, EmailUrlInfo, EmailPostDeliveryEvents | Email-based attack detection — phishing, malware delivery, post-delivery detonation |
| Defender for Identity | IdentityLogonEvents, IdentityQueryEvents, IdentityDirectoryEvents | On-premises AD attack detection — pass-the-hash, Kerberoasting, AD enumeration, DCSync |
| Defender for Cloud Apps | CloudAppEvents | SaaS application anomalies — mass download, impossible travel in apps, OAuth abuse |
| Defender for Cloud | SecurityAlert (Defender for Cloud alerts) | Azure workload protection — VM threats, Key Vault access, container threats |
| Defender XDR | SecurityIncident, SecurityAlert (unified) | Multi-stage incident correlation — the mailbomb scenario. Without XDR connector incidents are invisible to Sentinel |

---

## Section 4 — Microsoft Sentinel Pricing Tier

Sentinel pricing affects cost optimization decisions and retention configuration. Knowing the commitment tier helps you advise on whether the current setup is cost-effective.

**Document the Sentinel pricing configuration:**

| Field | Value |
|---|---|
| Pricing Tier | Pay-As-You-Go / Commitment Tier |
| Commitment Level (if applicable) | 100 GB / 200 GB / 300 GB / 400 GB / 500 GB / 1000 GB / 2000 GB / 5000 GB per day |
| Current Daily Ingestion Volume | |
| Hot Retention Period | |
| Archive Retention Period | |
| Estimated Monthly Cost | |

**Pricing guidance:**
- Pay-As-You-Go is appropriate for low volume environments under approximately 100 GB per day
- Commitment tiers provide significant discounts at higher volumes — typically 30-60% savings
- If daily ingestion consistently exceeds the PAYG breakeven point recommend a commitment tier review
- XDR-backed tables (DeviceEvents, EmailEvents etc) include 30 days retention in the Defender license — extending these to 90 days in Sentinel adds cost

---

## Section 5 — Azure Subscriptions and Resource Context

Understanding the client's Azure footprint helps identify what data sources should exist and what Azure Activity logs should be connected.

**Document Azure subscription details:**

| Subscription Name | Subscription ID | Purpose | Sentinel Connected |
|---|---|---|---|
| | | | |
| | | | |

**Add-on services to note:**

| Service | Present | Notes |
|---|---|---|
| Azure Key Vault | | Document if diagnostic logging is enabled |
| Azure Storage Accounts | | Note any security-relevant storage |
| Azure Kubernetes Service | | Note if AKS logging is configured |
| Azure SQL / Cosmos DB | | Note if query logging is enabled |
| Azure App Service | | Note if HTTP logging is configured |
| Azure Virtual Machines | | Document AMA enrollment status |
| Azure Arc Connected Machines | | Document Arc enrollment for on-prem |

---

## Section 6 — Additional Security Products

Document any third-party or additional Microsoft security products the client uses that may have Sentinel connectors or integration potential.

| Product | Vendor | Sentinel Connector Available | Connected | Notes |
|---|---|---|---|---|
| | | | | |
| | | | | |

**Common third-party products with Sentinel connectors:**
- Palo Alto Networks — NGFW, Panorama, Prisma
- Fortinet — FortiGate, FortiAnalyzer
- CrowdStrike Falcon
- Okta — Identity provider
- Salesforce
- ServiceNow
- AWS CloudTrail
- Google Cloud Platform
- Qualys / Tenable — vulnerability management
- CyberArk / BeyondTrust — PAM

---

## Section 7 — Copilot for Security

Microsoft Copilot for Security is an AI-powered security tool that integrates with Sentinel and the Defender portal. It is licensed separately and provides significant value when available.

**Document Copilot licensing:**

| Field | Value |
|---|---|
| Copilot for Security Licensed | Yes / No |
| Security Compute Units (SCUs) | |
| Embedded Copilot in Defender Portal | Yes / No — available to all Defender portal users |

**What Copilot unlocks:**
- Natural language KQL query generation
- Incident summarization and narrative generation
- Threat intelligence briefings
- Script and file deobfuscation
- Guided investigation recommendations
- Automated analyst notes

**Note:** Embedded Copilot features in the Defender portal (incident summaries, basic recommendations) are available to all customers using the Defender portal regardless of Copilot for Security licensing. Standalone Copilot for Security with full SCU capacity unlocks the complete feature set.

---

## Section 8 — License Summary and Capability Map

Complete this summary after filling in all sections above. This is the single-page reference that goes in the workbook and that any engineer reads first when picking up this client.

**Client:** _______________
**Date Reviewed:** _______________
**Reviewed By:** _______________

| Capability | Available | Enabled | Verified | Notes |
|---|---|---|---|---|
| Entra ID Sign-in Logs | | | | |
| Entra ID Identity Protection | | | | |
| Defender for Endpoint | | | | |
| Defender for Office 365 | | | | |
| Defender for Identity | | | | |
| Defender for Cloud Apps | | | | |
| Defender for Cloud | | | | |
| Defender XDR Connector | | | | |
| UEBA | | | | |
| Fusion ML | | | | |
| Threat Intelligence | | | | |
| SOC Optimization | | | | |
| Copilot for Security | | | | |
| Advanced Hunting | | | | |

**Overall license tier assessment:**
- [ ] Basic — E3 or equivalent. Limited capability set. Focus on Microsoft Connector sources and AMA agents.
- [ ] Standard — E5 or equivalent without full Defender suite. Good coverage with some gaps.
- [ ] Full — E5 with full Defender suite deployed. All capabilities available. Maximum Sentinel value.

**Capabilities licensed but not yet enabled:**
*(List anything the client is paying for that we have not turned on yet)*

**Capabilities not licensed but worth recommending:**
*(List any additions that would significantly improve coverage)*

---

## How This Feeds the Workbook

The license and entitlement summary feeds directly into the Client Security Operations Workbook in two ways.

**Tab 4 — Capabilities** uses the license summary to determine which capability cards should be displayed. A client without Defender for Identity does not get an MDI sensor health card. A client without Copilot for Security does not get a Copilot integration status card. The workbook reflects what the client actually has access to — not a generic template.

**Tab 1 — Summary** displays the license tier as context. When an engineer opens the workbook during a client review they immediately see the license tier which sets expectations for what should be present in all other tabs.

---

*This document is maintained in the sentinel-mssp-playbook repository under 01-baseline. Update it when the client changes their licensing or when new Microsoft products become relevant to Sentinel environments. Review minimum once per year or when Microsoft announces significant licensing changes.*