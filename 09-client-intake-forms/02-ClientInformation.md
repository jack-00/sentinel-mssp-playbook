# 02-ClientInformation

## Overview

This file contains the client information that will be imported into Microsoft
Sentinel as a watchlist named **ClientInfo**. The watchlist drives the Client
Information section of Page 1 of the AuditDeck workbook. Each category is
displayed as a separate titled block on the page. Only fields with values will
appear — blank fields are automatically filtered out.

---

## Fields

**Client Overview**
- Client Name
- Industry
- Company Size
- Company URL
- Onboard Date
- Bio

**Azure / Sentinel**
- Tenant ID
- Subscription ID
- Subscription Name
- Workspace ID
- Workspace Name
- Resource Group
- Region

**Licensing**
- M365 License
- Entra ID Tier
- Defender for Endpoint Plan
- Defender for Identity Plan
- Defender for Office 365 Plan
- Defender for Cloud Plan
- Defender for Cloud Apps Plan
- Defender for Servers Plan
- Defender for SQL
- Defender for Storage
- Defender for Containers
- Defender for Key Vault
- Microsoft Sentinel License Type

**Sentinel / Cost**
- Pricing Model
- Commitment Tier GB/Day
- Daily Cap GB
- Data Lake Enabled
- Average Daily Ingestion GB

**Support / Ops**
- Microsoft Support Plan
- Optiv Ingest License

---

## Watchlist Structure

Each row follows the format: **Category | Field | Value**

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
| Azure / Sentinel | Region | Manual | Azure region |
| Licensing | M365 License | Manual | e.g. Business Premium, E3, E5 |
| Licensing | Entra ID Tier | Manual | Free, P1, or P2 |
| Licensing | Defender for Endpoint Plan | Manual | Plan 1 or Plan 2 |
| Licensing | Defender for Identity Plan | Manual | Licensed |
| Licensing | Defender for Office 365 Plan | Manual | Plan 1 or Plan 2 |
| Licensing | Defender for Cloud Plan | Manual | Foundational or Defender CSPM |
| Licensing | Defender for Cloud Apps Plan | Manual | Licensed |
| Licensing | Defender for Servers Plan | Manual | Plan 1 or Plan 2 |
| Licensing | Defender for SQL | Manual | Licensed |
| Licensing | Defender for Storage | Manual | Licensed |
| Licensing | Defender for Containers | Manual | Licensed |
| Licensing | Defender for Key Vault | Manual | Licensed |
| Licensing | Microsoft Sentinel License Type | Manual | Pay-As-You-Go or Commitment Tier |
| Sentinel / Cost | Pricing Model | Manual | Pay-As-You-Go or Commitment Tier |
| Sentinel / Cost | Commitment Tier GB/Day | Manual | e.g. 100, 200, 300 — leave blank if Pay-As-You-Go |
| Sentinel / Cost | Daily Cap GB | Manual | Hard ingestion cap if configured — leave blank if none |
| Sentinel / Cost | Data Lake Enabled | Manual | Yes or No |
| Sentinel / Cost | Average Daily Ingestion GB | Script | Auto-populated by collection script |
| Support / Ops | Microsoft Support Plan | Manual | e.g. Basic, Developer, Standard, Premier, Unified |
| Support / Ops | Optiv Ingest License | Manual | Internal — Optiv managed ingestion license details |

---

## Notes

- Leave any fields blank that do not apply — blank values will not appear in the workbook
- Average Daily Ingestion GB is the only script-populated field — all others are manually maintained
- Do not add or remove rows from the template

---

## Spreadsheet Tabs

The 02-ClientInformation.xlsx file has two tabs:

**Tab 1 — Template**
Blank template for the client to fill in. Category and Field columns are
pre-populated. Client fills in the Value column only.

**Tab 2 — Example**
Completed example using fictional dummy data for reference.

---

## Tab 1 — Template CSV

```csv
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
```

---

## Tab 2 — Example CSV

```csv
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
```

---

## Adding to the Workbook

Once the completed file is returned import it into Microsoft Sentinel as a
watchlist named **ClientInfo** using the completed Tab 1 as the source file
with **Field** as the SearchKey.

Each category is displayed as a separate titled block on Page 1 of the AuditDeck
workbook using the following KQL queries:

### Client Overview

```kql
_GetWatchlist('ClientInfo')
| where Category == "Client Overview"
| where isnotempty(Value)
| project Field, Value
```

### Azure / Sentinel

```kql
_GetWatchlist('ClientInfo')
| where Category == "Azure / Sentinel"
| where isnotempty(Value)
| project Field, Value
```

### Licensing

```kql
_GetWatchlist('ClientInfo')
| where Category == "Licensing"
| where isnotempty(Value)
| project Field, Value
```

### Sentinel / Cost

```kql
_GetWatchlist('ClientInfo')
| where Category == "Sentinel / Cost"
| where isnotempty(Value)
| project Field, Value
```

### Support / Ops

```kql
_GetWatchlist('ClientInfo')
| where Category == "Support / Ops"
| where isnotempty(Value)
| project Field, Value
```