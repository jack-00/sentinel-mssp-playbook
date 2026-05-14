MSS - AuditDeck
Client Information Guide
Optiv Managed Security Services

================================================================================

INTRODUCTION

This document provides detailed guidance for completing the
02-ClientInformation.xlsx spreadsheet. Work through this guide alongside
the spreadsheet as you fill in each field.

The spreadsheet has two tabs:

    Tab 1 — Template
    This is the tab you fill out. The Category and Field columns are already
    populated — you only need to fill in the Value column. If a field does
    not apply to your organization simply leave it blank and it will not
    appear in the workbook.

    Tab 2 — Example
    This tab shows a fully completed example using a fictional company called
    Contoso Medical Group. Use it as a reference to understand what kind of
    information we are looking for and what format to use. Do not edit this tab.

If you are unsure about any field please leave it blank and reply in your
ServiceNow ticket — we would rather follow up on a few gaps than work from
inaccurate data.

================================================================================

CLIENT OVERVIEW

--------------------------------------------------------------------------------

Client Name
Your organization's full legal or trading name.
Example: Contoso Medical Group

--------------------------------------------------------------------------------

Industry
The industry or sector your organization operates in. This helps us understand
your regulatory environment and the types of threats most relevant to your
business.
Examples: Healthcare, Financial Services, Legal, Manufacturing, Retail,
Education, Non-Profit, Technology

--------------------------------------------------------------------------------

Company Size
A general description of your organization's size.
Examples: Small — under 50 employees, Mid-Market — 50 to 500 employees,
Enterprise — over 500 employees. You can also provide a rough employee
count if you prefer.

--------------------------------------------------------------------------------

Company URL
Your organization's primary public website address.
Example: https://www.contosomed.com

--------------------------------------------------------------------------------

Onboard Date
The date your organization began working with us on this Sentinel deployment.
If you are unsure your Optiv engineer can provide this.
Format: YYYY-MM-DD
Example: 2024-03-15

--------------------------------------------------------------------------------

Bio
A short 3 to 5 sentence description of your organization and your technology
environment. Think of this as what a new security engineer would need to know
in their first 60 seconds working with your environment. Include what your
organization does, how many locations and employees you have, and anything
notable about your technology setup.

Example: Contoso Medical Group is a regional urgent care provider operating
12 clinics across the Pacific Northwest with approximately 450 employees.
Their environment is hybrid with on-premises Active Directory synced to
Entra ID. Primary endpoints are Windows 11 managed via Intune. They operate
under strict HIPAA compliance requirements and rely heavily on Epic EHR for
patient data management.

================================================================================

AZURE / SENTINEL

These fields identify your Azure and Microsoft Sentinel environment. Most of
this information can be found in the Azure portal. If you are unsure where to
find any of these values your Optiv engineer can assist.

--------------------------------------------------------------------------------

Tenant ID
Your Microsoft Entra ID tenant identifier. This is a unique ID that identifies
your organization in Microsoft's cloud.
Where to find it: Azure Portal → Microsoft Entra ID → Overview → Tenant ID
Format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

--------------------------------------------------------------------------------

Subscription ID
The unique identifier for your Azure subscription.
Where to find it: Azure Portal → Subscriptions → select your subscription
→ Subscription ID
Format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

--------------------------------------------------------------------------------

Subscription Name
The friendly name of your Azure subscription.
Where to find it: Azure Portal → Subscriptions → Name column
Example: Contoso-Medical-Production

--------------------------------------------------------------------------------

Workspace ID
The unique identifier for your Microsoft Sentinel Log Analytics workspace.
Where to find it: Azure Portal → Log Analytics Workspaces → select your
workspace → Overview → Workspace ID
Format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

--------------------------------------------------------------------------------

Workspace Name
The friendly name of your Log Analytics workspace.
Where to find it: Azure Portal → Log Analytics Workspaces → Name column
Example: Contoso-Sentinel-Workspace

--------------------------------------------------------------------------------

Resource Group
The name of the Azure resource group that contains your Sentinel workspace.
Where to find it: Azure Portal → Log Analytics Workspaces → select your
workspace → Resource Group
Example: rg-contoso-sentinel-prod

--------------------------------------------------------------------------------

Region
The Azure region where your Sentinel workspace is hosted.
Where to find it: Azure Portal → Log Analytics Workspaces → select your
workspace → Overview → Location
Examples: East US, West Europe, UK South, Australia East

================================================================================

LICENSING

These fields capture your Microsoft security product licensing. This information
is critical for understanding what security data is available in your environment
and what detections we can run on your behalf.

For each Defender product enter the plan level if licensed or leave blank if
not licensed. If you are unsure about any of these contact your Microsoft account
team or check the Microsoft 365 Admin Center under Billing → Licenses.

--------------------------------------------------------------------------------

M365 License
Your primary Microsoft 365 license tier.
Options: Microsoft 365 Business Basic, Microsoft 365 Business Standard,
Microsoft 365 Business Premium, Microsoft 365 E3, Microsoft 365 E5

--------------------------------------------------------------------------------

Entra ID Tier
Your Microsoft Entra ID license tier. This was previously called Azure
Active Directory.
Options: Free, P1, P2
Note: Entra ID P2 includes Privileged Identity Management and Identity
Protection which are important for advanced identity monitoring.

--------------------------------------------------------------------------------

Defender for Endpoint Plan
Microsoft's endpoint detection and response solution for workstations
and servers.
Options: Plan 1, Plan 2
Note: Plan 2 provides significantly more telemetry and is required for
advanced endpoint detections.

--------------------------------------------------------------------------------

Defender for Identity Plan
Microsoft's identity threat detection solution. Monitors Active Directory
and Entra ID for identity based attacks.
Options: Licensed
Note: Requires on-premises Active Directory domain controllers to have the
MDI sensor installed.

--------------------------------------------------------------------------------

Defender for Office 365 Plan
Microsoft's email and collaboration security solution covering Exchange
Online, Teams, and SharePoint.
Options: Plan 1, Plan 2
Note: Plan 2 includes attack simulation training and advanced hunting
capabilities.

--------------------------------------------------------------------------------

Defender for Cloud Plan
Microsoft's cloud security posture management solution for Azure workloads.
Options: Foundational CSPM, Defender CSPM
Note: Only relevant if you have Azure hosted workloads.

--------------------------------------------------------------------------------

Defender for Cloud Apps Plan
Microsoft's cloud access security broker solution for monitoring and
controlling SaaS application usage.
Options: Licensed
Note: Only relevant if you have cloud applications you want monitored.

--------------------------------------------------------------------------------

Defender for Servers Plan
Microsoft's server protection solution for Azure and on-premises servers.
Options: Plan 1, Plan 2
Note: Only relevant if you have Azure virtual machines or on-premises
servers enrolled in Defender for Cloud.

--------------------------------------------------------------------------------

Defender for SQL
Microsoft's protection solution for SQL databases hosted in Azure.
Options: Licensed
Note: Only relevant if you have Azure SQL databases.

--------------------------------------------------------------------------------

Defender for Storage
Microsoft's protection solution for Azure Storage accounts.
Options: Licensed
Note: Only relevant if you have Azure Storage accounts.

--------------------------------------------------------------------------------

Defender for Containers
Microsoft's protection solution for containerized workloads running in
Azure Kubernetes Service or other container platforms.
Options: Licensed
Note: Only relevant if you run containers or Kubernetes.

--------------------------------------------------------------------------------

Defender for Key Vault
Microsoft's protection solution for Azure Key Vault resources.
Options: Licensed
Note: Only relevant if you use Azure Key Vault.

--------------------------------------------------------------------------------

Microsoft Sentinel License Type
How your Microsoft Sentinel usage is billed.
Options: Pay-As-You-Go, Commitment Tier
Note: If you are unsure your Optiv engineer can confirm this.

================================================================================

SENTINEL / COST

These fields capture how your Sentinel workspace is configured from a cost
and capacity perspective.

--------------------------------------------------------------------------------

Pricing Model
Your current Sentinel billing model.
Options: Pay-As-You-Go, Commitment Tier

--------------------------------------------------------------------------------

Commitment Tier GB/Day
If you are on a Commitment Tier enter the daily GB commitment level here.
Leave blank if you are on Pay-As-You-Go.
Where to find it: Azure Portal → Microsoft Sentinel → Settings → Pricing
Examples: 100, 200, 300, 500

--------------------------------------------------------------------------------

Daily Cap GB
If your workspace has a hard daily ingestion cap configured enter it here.
This is a safety limit that stops data ingestion once the cap is reached
each day. Leave blank if no cap is configured.
Where to find it: Azure Portal → Log Analytics Workspace → Usage and
estimated costs → Daily cap
Note: If a daily cap is set and ingestion reaches it your security monitoring
will stop for the remainder of that day. Let your Optiv engineer know if you
have a cap configured so we can monitor for it.

--------------------------------------------------------------------------------

Data Lake Enabled
Whether the Microsoft Sentinel Data Lake tier has been enabled on your
workspace. This is a newer feature that provides low cost long term
data retention.
Options: Yes, No
Where to find it: Azure Portal → Microsoft Sentinel → Settings

--------------------------------------------------------------------------------

Average Daily Ingestion GB
Leave this blank. This field is automatically populated by our collection
scripts and does not need to be filled in manually.

================================================================================

SUPPORT / OPS

--------------------------------------------------------------------------------

Microsoft Support Plan
Your organization's Microsoft support plan tier. This determines how quickly
Microsoft responds to support requests and what level of assistance is available.
Options: Basic, Developer, Standard, Premier, Unified
Note: Basic support does not include technical support cases. If you are
unsure which plan you have check with your IT team or Microsoft account manager.

--------------------------------------------------------------------------------

Optiv Ingest License
Your Optiv managed ingestion license identifier. Your Optiv engineer will
provide this value — you do not need to look this up yourself.

================================================================================

QUESTIONS

If you have any questions about any field please reply in your ServiceNow
ticket and we will be happy to help.

================================================================================