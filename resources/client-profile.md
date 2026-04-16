# AUDIT DECK

## [Company Name]

# 🛡️ [Client Name]

[Company URL]

*[Industry]*

---

[Bio — two to three sentences about who the client is and why security matters to their business.]

---

| | |
|---|---|
| **Onboarding Date** | [YYYY-MM-DD] |
| **Primary Contact** | [Name — Email] |
| **Sentinel Workspace** | [Link] |

---

## Escalation Path

| Escalation | Name | Title | Business Phone | Mobile Phone | Email |
|---|---|---|---|---|---|
| L0 | | | | | |
| L1 | | | | | |
| L2 | | | | | |
| L3 | | | | | |
| L4 | | | | | |
| L5 | | | | | |
| L6 | | | | | |
| L7 | | | | | |

---

## Notes

[Add any relevant notes here.]



# [mssname]-client — Watchlist CSV Header

Copy the line below into a blank text file and save as `[client]-client.csv`

```
ClientName,Industry,CompanyURL,OnboardingDate,Bio,TenantID,SubscriptionID,WorkspaceID,WorkspaceName,ResourceGroup,M365License,DefenderLicense,SentinelCommitmentTier,EntraIDTier,MicrosoftSupportPlan,Notes
```


_GetWatchlist('[mssname]-client')
| project ClientName, Industry, CompanyURL, Bio, OnboardingDate,
    TenantID, WorkspaceID, M365License, DefenderLicense,
    SentinelCommitmentTier, EntraIDTier,
    MicrosoftSupportPlan, MicrosoftSupportURL, Notes


let c = _GetWatchlist('[mssname]-client');
c | project Field = "Client Name", Value = ClientName, Order = 1
| union (c | project Field = "Industry", Value = Industry, Order = 2)
| union (c | project Field = "Website", Value = CompanyURL, Order = 3)
| union (c | project Field = "Onboarding Date", Value = OnboardingDate, Order = 4)
| union (c | project Field = "Bio", Value = Bio, Order = 5)
| union (c | project Field = "Tenant ID", Value = TenantID, Order = 6)
| union (c | project Field = "Subscription ID", Value = SubscriptionID, Order = 7)
| union (c | project Field = "Workspace ID", Value = WorkspaceID, Order = 8)
| union (c | project Field = "Workspace Name", Value = WorkspaceName, Order = 9)
| union (c | project Field = "Resource Group", Value = ResourceGroup, Order = 10)
| union (c | project Field = "M365 License", Value = M365License, Order = 11)
| union (c | project Field = "Defender License", Value = DefenderLicense, Order = 12)
| union (c | project Field = "Sentinel Commitment Tier", Value = SentinelCommitmentTier, Order = 13)
| union (c | project Field = "Entra ID Tier", Value = EntraIDTier, Order = 14)
| union (c | project Field = "Support Plan", Value = MicrosoftSupportPlan, Order = 15)
| union (c | project Field = "Notes", Value = Notes, Order = 16)
| sort by Order asc
| project Field, Value