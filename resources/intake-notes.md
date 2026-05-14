# Client Intake Form — Watchlist Schema

## Watchlist Structure
Each row follows the format: **Category | Field | Value**

---

## Fields by Category

| Category | Field | Source | Notes |
|---|---|---|---|
| Client Overview | Client Name | Manual | Legal or trading name |
| Client Overview | Industry | Manual | e.g. Healthcare, Financial Services, Manufacturing |
| Client Overview | Company Size | Manual | e.g. SMB, Mid-Market, Enterprise or rough employee count |
| Client Overview | Company URL | Manual | Primary website |
| Client Overview | Onboard Date | Manual | Date client was onboarded to the program |
| Client Overview | Bio | Manual | Short description of the company and their environment |
| Azure / Sentinel | Tenant ID | Manual | Azure AD / Entra tenant ID |
| Azure / Sentinel | Subscription ID | Manual | Azure subscription ID |
| Azure / Sentinel | Subscription Name | Manual | Friendly name of the subscription |
| Azure / Sentinel | Workspace ID | Manual | Log Analytics workspace ID |
| Azure / Sentinel | Workspace Name | Manual | Friendly name of the workspace |
| Azure / Sentinel | Resource Group | Manual | Resource group containing the workspace |
| Azure / Sentinel | Region | Manual | Azure region — relevant for data residency conversations |
| Licensing | M365 License | Manual | e.g. Business Premium, E3, E5 |
| Licensing | Entra ID Tier | Manual | Free, P1, or P2 |
| Licensing | Defender for Endpoint Plan | Manual | Not Licensed, Plan 1, or Plan 2 |
| Licensing | Defender for Identity Plan | Manual | Not Licensed or Licensed |
| Licensing | Defender for Office 365 Plan | Manual | Not Licensed, Plan 1, or Plan 2 |
| Licensing | Defender for Cloud Plan | Manual | Not Licensed, Foundational, or Defender CSPM |
| Licensing | Defender for Cloud Apps Plan | Manual | Not Licensed or Licensed |
| Licensing | Defender for Servers Plan | Manual | Not Licensed, Plan 1, or Plan 2 |
| Licensing | Defender for SQL | Manual | Not Licensed or Licensed |
| Licensing | Defender for Storage | Manual | Not Licensed or Licensed |
| Licensing | Defender for Containers | Manual | Not Licensed or Licensed |
| Licensing | Defender for Key Vault | Manual | Not Licensed or Licensed |
| Licensing | Microsoft Sentinel License Type | Manual | Pay-As-You-Go or Commitment Tier |
| Sentinel / Cost | Pricing Model | Manual | Pay-As-You-Go or Commitment Tier |
| Sentinel / Cost | Commitment Tier GB/Day | Manual | e.g. 100, 200, 300 — leave blank if Pay-As-You-Go |
| Sentinel / Cost | Daily Cap GB | Manual | Hard ingestion cap if configured — leave blank if none |
| Sentinel / Cost | Data Lake Enabled | Manual | Yes or No |
| Sentinel / Cost | Average Daily Ingestion GB | Script | Auto-populated by collection script |
| Support / Ops | Microsoft Support Plan | Manual | e.g. Basic, Developer, Standard, Premier |
| Support / Ops | Optiv Ingest License | Manual | Internal — Optiv managed ingestion license details |

---

## Notes
- Rows where Value is **Not Licensed** or blank will be filtered out by workbook KQL and will not display on Page 1
- **Average Daily Ingestion GB** is the only script-populated field in this watchlist — all others are manually maintained
- This watchlist is the source of truth for Page 1 of the audit workbook
- Dynamic intake questions (e.g. credential expiration dates) are generated separately based on what the collection script finds in the client environment

You're right — if you're leaving it blank rather than typing "Not Licensed" then the `isnotempty(Value)` filter already handles it. The `Value != "Not Licensed"` check is redundant.

Here are the corrected queries:

```markdown
## KQL Queries — Page 1 Workbook Blocks

All queries reference the watchlist named **ClientInfo** — update the watchlist name to match yours.
Each query filters to its category and excludes empty values.

---

### Client Overview

```kql
_GetWatchlist('ClientInfo')
| where Category == "Client Overview"
| where isnotempty(Value)
| project Field, Value
```

---

### Azure / Sentinel

```kql
_GetWatchlist('ClientInfo')
| where Category == "Azure / Sentinel"
| where isnotempty(Value)
| project Field, Value
```

---

### Licensing

```kql
_GetWatchlist('ClientInfo')
| where Category == "Licensing"
| where isnotempty(Value)
| project Field, Value
```

---

### Sentinel / Cost

```kql
_GetWatchlist('ClientInfo')
| where Category == "Sentinel / Cost"
| where isnotempty(Value)
| project Field, Value
```

---

### Support / Ops

```kql
_GetWatchlist('ClientInfo')
| where Category == "Support / Ops"
| where isnotempty(Value)
| project Field, Value
```

---

## Notes
- Replace **ClientInfo** with your actual watchlist name in all queries
- Each query block maps to one titled section on Page 1 of the workbook
- Empty fields simply will not appear — no extra filtering needed
- Add `| sort by Field asc` to any query if you want fields displayed alphabetically within a category
- Average Daily Ingestion GB updates automatically each time the collection script refreshes the watchlist
```

# Client Environment Profile — Categories

1. Environment Summary
2. Network
3. Identity
4. Endpoints
5. Cloud
6. Virtualization
7. Containers
8. SaaS and Applications
9. Public Facing Infrastructure
10. Notes

```markdown
# Client Intake — Watchlist CSV Files

These two CSV files are used to collect persistent client information that populates the Client Information watchlist in Microsoft Sentinel. The watchlist drives Page 1 of the client audit workbook.

## Files

- **template.csv** — Blank template for the client to fill out. All Category and Field values are pre-populated. The client only fills in the Value column.
- **example.csv** — Fully completed example using fictional dummy data. Use this as a reference when filling out the template.

## How to Use

1. Provide the client with both CSV files along with the accompanying Word document that explains each field in detail
2. The client fills in the Value column on the template CSV only
3. Once returned, the completed CSV is imported into Microsoft Sentinel as a watchlist named **ClientInfo**
4. The workbook queries the watchlist to automatically populate Page 1

## Notes

- Leave any fields blank that do not apply — blank values will not appear in the workbook
- The **Average Daily Ingestion GB** field does not need to be filled in — it is auto-populated by the collection script
- Field values should match the options described in the accompanying Word document to ensure consistent formatting across all clients
- Do not add or remove rows from the template — contact your Optiv engineer if you believe a field is missing
```

Tab 1 — Template (blank)

Category,Field,Value
Client Overview,Client Name,
Client Overview,Industry,
Client Overview,Company Size,
Client Overview,Company URL,
Client Overview,Onboard Date,
Client Overview,Bio,
Azure / Sentinel,Tenant ID,
Azure / Sentinel,Subscription ID,
Azure / Sentinel,Subscription Name,
Azure / Sentinel,Workspace ID,
Azure / Sentinel,Workspace Name,
Azure / Sentinel,Resource Group,
Azure / Sentinel,Region,
Licensing,M365 License,
Licensing,Entra ID Tier,
Licensing,Defender for Endpoint Plan,
Licensing,Defender for Identity Plan,
Licensing,Defender for Office 365 Plan,
Licensing,Defender for Cloud Plan,
Licensing,Defender for Cloud Apps Plan,
Licensing,Defender for Servers Plan,
Licensing,Defender for SQL,
Licensing,Defender for Storage,
Licensing,Defender for Containers,
Licensing,Defender for Key Vault,
Licensing,Microsoft Sentinel License Type,
Sentinel / Cost,Pricing Model,
Sentinel / Cost,Commitment Tier GB/Day,
Sentinel / Cost,Daily Cap GB,
Sentinel / Cost,Data Lake Enabled,
Sentinel / Cost,Average Daily Ingestion GB,
Support / Ops,Microsoft Support Plan,
Support / Ops,Optiv Ingest License,

Tab 2 — Example (dummy data)

Category,Field,Value
Client Overview,Client Name,Contoso Medical Group
Client Overview,Industry,Healthcare
Client Overview,Company Size,Mid-Market — approximately 450 employees
Client Overview,Company URL,https://www.contosomed.com
Client Overview,Onboard Date,2024-03-15
Client Overview,Bio,Contoso Medical Group is a regional urgent care provider operating 12 clinics across the Pacific Northwest. Their environment is hybrid with on-premises Active Directory synced to Entra ID. Primary endpoints are Windows 11 managed via Intune with strict HIPAA compliance requirements.
Azure / Sentinel,Tenant ID,a1b2c3d4-e5f6-7890-abcd-ef1234567890
Azure / Sentinel,Subscription ID,b2c3d4e5-f6a7-8901-bcde-f12345678901
Azure / Sentinel,Subscription Name,Contoso-Medical-Production
Azure / Sentinel,Workspace ID,c3d4e5f6-a7b8-9012-cdef-123456789012
Azure / Sentinel,Workspace Name,Contoso-Sentinel-Workspace
Azure / Sentinel,Resource Group,rg-contoso-sentinel-prod
Azure / Sentinel,Region,East US
Licensing,M365 License,Microsoft 365 E3
Licensing,Entra ID Tier,P2
Licensing,Defender for Endpoint Plan,Plan 2
Licensing,Defender for Identity Plan,Licensed
Licensing,Defender for Office 365 Plan,Plan 1
Licensing,Defender for Cloud Plan,Defender CSPM
Licensing,Defender for Cloud Apps Plan,Licensed
Licensing,Defender for Servers Plan,Plan 2
Licensing,Defender for SQL,
Licensing,Defender for Storage,Licensed
Licensing,Defender for Containers,
Licensing,Defender for Key Vault,
Licensing,Microsoft Sentinel License Type,Commitment Tier
Sentinel / Cost,Pricing Model,Commitment Tier
Sentinel / Cost,Commitment Tier GB/Day,100
Sentinel / Cost,Daily Cap GB,150
Sentinel / Cost,Data Lake Enabled,Yes
Sentinel / Cost,Average Daily Ingestion GB,
Support / Ops,Microsoft Support Plan,Standard
Support / Ops,Optiv Ingest License,OIL-CMG-2024


```markdown
# Client Information Intake Guide

## Why We Are Asking for This Information

As part of our co-managed Microsoft Sentinel service, we are building a comprehensive audit and monitoring workbook that lives directly inside your Sentinel environment. This workbook will serve as a living reference for both your team and ours — giving anyone who opens it an immediate, clear picture of your environment, your security coverage, and the health of your monitoring platform.

To build this properly we need to collect some foundational information about your organization, your Microsoft licensing, and your Azure environment. This information will be stored securely inside your own Sentinel workspace as a watchlist and will only be visible to those who already have access to your environment.

This is a one-time collection effort. Once the data is in place your workbook will be live and we will maintain it going forward as part of our ongoing service.

---

## About the Spreadsheet

The spreadsheet we have provided has two tabs:

**Tab 1 — Template**
This is the tab you fill out. The Category and Field columns are already populated — you only need to fill in the Value column. Not every field will apply to your organization. If a field does not apply simply leave it blank and it will not appear in the workbook.

**Tab 2 — Example**
This tab shows a fully completed example using a fictional company called Contoso Medical Group. Use it as a reference to understand what kind of information we are looking for and what format to use. Do not edit this tab.

---

## Field Guide

The following section explains every field in the template, what it means, why we need it, and what to put there. Fields are grouped by category matching the spreadsheet.

---

### Client Overview

**Client Name**
Your organization's full legal or trading name.
*Example: Contoso Medical Group*

**Industry**
The industry or sector your organization operates in. This helps us understand your regulatory environment and the types of threats most relevant to your business.
*Examples: Healthcare, Financial Services, Legal, Manufacturing, Retail, Education, Non-Profit, Technology*

**Company Size**
A general description of your organization's size. This helps us understand the scale of your environment.
*Examples: Small — under 50 employees, Mid-Market — 50 to 500 employees, Enterprise — over 500 employees. You can also provide a rough employee count if you prefer.*

**Company URL**
Your organization's primary public website address.
*Example: https://www.contosomed.com*

**Onboard Date**
The date your organization began working with us on this Sentinel deployment. If you are unsure your Optiv engineer can provide this.
*Format: YYYY-MM-DD — Example: 2024-03-15*

**Bio**
A short 3 to 5 sentence description of your organization and your technology environment. Think of this as what a new security engineer would need to know in their first 60 seconds working with your environment. Include what your organization does, how many locations and employees you have, and anything notable about your technology setup.
*Example: Contoso Medical Group is a regional urgent care provider operating 12 clinics across the Pacific Northwest with approximately 450 employees. Their environment is hybrid with on-premises Active Directory synced to Entra ID. Primary endpoints are Windows 11 managed via Intune. They operate under strict HIPAA compliance requirements and rely heavily on Epic EHR for patient data management.*

---

### Azure / Sentinel

These fields identify your Azure and Microsoft Sentinel environment. Most of this information can be found in the Azure portal. If you are unsure where to find any of these values your Optiv engineer can assist.

**Tenant ID**
Your Microsoft Entra ID tenant identifier. This is a unique ID that identifies your organization in Microsoft's cloud.
*Where to find it: Azure Portal → Microsoft Entra ID → Overview → Tenant ID*
*Format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx*

**Subscription ID**
The unique identifier for your Azure subscription.
*Where to find it: Azure Portal → Subscriptions → select your subscription → Subscription ID*
*Format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx*

**Subscription Name**
The friendly name of your Azure subscription.
*Where to find it: Azure Portal → Subscriptions → Name column*
*Example: Contoso-Medical-Production*

**Workspace ID**
The unique identifier for your Microsoft Sentinel Log Analytics workspace.
*Where to find it: Azure Portal → Log Analytics Workspaces → select your workspace → Overview → Workspace ID*
*Format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx*

**Workspace Name**
The friendly name of your Log Analytics workspace.
*Where to find it: Azure Portal → Log Analytics Workspaces → Name column*
*Example: Contoso-Sentinel-Workspace*

**Resource Group**
The name of the Azure resource group that contains your Sentinel workspace.
*Where to find it: Azure Portal → Log Analytics Workspaces → select your workspace → Resource Group*
*Example: rg-contoso-sentinel-prod*

**Region**
The Azure region where your Sentinel workspace is hosted.
*Where to find it: Azure Portal → Log Analytics Workspaces → select your workspace → Overview → Location*
*Examples: East US, West Europe, UK South, Australia East*

---

### Licensing

These fields capture your Microsoft security product licensing. This information is critical for understanding what security data is available in your environment and what detections we can run on your behalf.

For each Defender product enter the plan level if licensed or leave blank if not licensed. If you are unsure about any of these contact your Microsoft account team or check the Microsoft 365 Admin Center under Billing → Licenses.

**M365 License**
Your primary Microsoft 365 license tier.
*Options: Microsoft 365 Business Basic, Microsoft 365 Business Standard, Microsoft 365 Business Premium, Microsoft 365 E3, Microsoft 365 E5*

**Entra ID Tier**
Your Microsoft Entra ID license tier. This was previously called Azure Active Directory.
*Options: Free, P1, P2*
*Note: Entra ID P2 includes Privileged Identity Management and Identity Protection which are important for advanced identity monitoring*

**Defender for Endpoint Plan**
Microsoft's endpoint detection and response solution for workstations and servers.
*Options: Plan 1, Plan 2*
*Note: Plan 2 provides significantly more telemetry and is required for advanced endpoint detections*

**Defender for Identity Plan**
Microsoft's identity threat detection solution. Monitors Active Directory and Entra ID for identity based attacks.
*Options: Licensed*
*Note: Requires on-premises Active Directory domain controllers to have the MDI sensor installed*

**Defender for Office 365 Plan**
Microsoft's email and collaboration security solution covering Exchange Online, Teams, and SharePoint.
*Options: Plan 1, Plan 2*
*Note: Plan 2 includes attack simulation training and advanced hunting capabilities*

**Defender for Cloud Plan**
Microsoft's cloud security posture management solution for Azure workloads.
*Options: Foundational CSPM, Defender CSPM*
*Note: Only relevant if you have Azure hosted workloads*

**Defender for Cloud Apps Plan**
Microsoft's cloud access security broker solution for monitoring and controlling SaaS application usage.
*Options: Licensed*

**Defender for Servers Plan**
Microsoft's server protection solution for Azure and on-premises servers.
*Options: Plan 1, Plan 2*
*Note: Only relevant if you have Azure virtual machines or on-premises servers enrolled in Defender for Cloud*

**Defender for SQL**
Microsoft's protection solution for SQL databases hosted in Azure.
*Options: Licensed*
*Note: Only relevant if you have Azure SQL databases*

**Defender for Storage**
Microsoft's protection solution for Azure Storage accounts.
*Options: Licensed*
*Note: Only relevant if you have Azure Storage accounts*

**Defender for Containers**
Microsoft's protection solution for containerized workloads running in Azure Kubernetes Service or other container platforms.
*Options: Licensed*
*Note: Only relevant if you run containers or Kubernetes*

**Defender for Key Vault**
Microsoft's protection solution for Azure Key Vault resources.
*Options: Licensed*
*Note: Only relevant if you use Azure Key Vault*

**Microsoft Sentinel License Type**
How your Microsoft Sentinel usage is billed.
*Options: Pay-As-You-Go, Commitment Tier*
*Note: If you are unsure your Optiv engineer can confirm this*

---

### Sentinel / Cost

These fields capture how your Sentinel workspace is configured from a cost and capacity perspective.

**Pricing Model**
Your current Sentinel billing model.
*Options: Pay-As-You-Go, Commitment Tier*

**Commitment Tier GB/Day**
If you are on a Commitment Tier enter the daily GB commitment level here. Leave blank if you are on Pay-As-You-Go.
*Where to find it: Azure Portal → Microsoft Sentinel → Settings → Pricing*
*Examples: 100, 200, 300, 500*

**Daily Cap GB**
If your workspace has a hard daily ingestion cap configured enter it here. This is a safety limit that stops data ingestion once the cap is reached each day. Leave blank if no cap is configured.
*Where to find it: Azure Portal → Log Analytics Workspace → Usage and estimated costs → Daily cap*
*Note: If a daily cap is set and ingestion reaches it your security monitoring will stop for the remainder of that day. Let your Optiv engineer know if you have a cap configured so we can monitor for it.*

**Data Lake Enabled**
Whether the Microsoft Sentinel Data Lake tier has been enabled on your workspace. This is a newer feature that provides low cost long term data retention.
*Options: Yes, No*
*Where to find it: Azure Portal → Microsoft Sentinel → Settings*

**Average Daily Ingestion GB**
Leave this blank. This field is automatically populated by our collection scripts and does not need to be filled in manually.

---

### Support / Ops

**Microsoft Support Plan**
Your organization's Microsoft support plan tier. This determines how quickly Microsoft responds to support requests and what level of assistance is available.
*Options: Basic, Developer, Standard, Premier, Unified*
*Note: Basic support does not include technical support cases. If you are unsure which plan you have check with your IT team or Microsoft account manager.*

**Optiv Ingest License**
Your Optiv managed ingestion license identifier. Your Optiv engineer will provide this value — you do not need to look this up yourself.

---

## Questions or Need Help?

If you are unsure about any field or cannot locate a value please reach out to your Optiv engineer before submitting. It is better to leave a field blank than to enter incorrect information. We would rather follow up on a few gaps than work from inaccurate data.

Once completed please return the filled in template CSV to your Optiv engineer. Do not modify the Category or Field columns — only fill in the Value column.

Thank you for taking the time to complete this. The information you provide here directly improves the quality of your security monitoring and the value of your audit workbook.
```

