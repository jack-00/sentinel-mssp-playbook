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





let c = _GetWatchlist('[mssname]-client');
c | project Field = "Client Name", Info = ClientName, Order = 1
| union (c | project Field = "Industry", Info = Industry, Order = 2)
| union (c | project Field = "Website", Info = CompanyURL, Order = 3)
| union (c | project Field = "Onboarding Date", Info = OnboardingDate, Order = 4)
| union (c | project Field = "Bio", Info = Bio, Order = 5)
| union (c | project Field = "Tenant ID", Info = TenantID, Order = 6)
| union (c | project Field = "Subscription ID", Info = SubscriptionID, Order = 7)
| union (c | project Field = "Workspace ID", Info = WorkspaceID, Order = 8)
| union (c | project Field = "Workspace Name", Info = WorkspaceName, Order = 9)
| union (c | project Field = "Resource Group", Info = ResourceGroup, Order = 10)
| union (c | project Field = "M365 License", Info = M365License, Order = 11)
| union (c | project Field = "Defender License", Info = DefenderLicense, Order = 12)
| union (c | project Field = "Sentinel Commitment Tier", Info = SentinelCommitmentTier, Order = 13)
| union (c | project Field = "Entra ID Tier", Info = EntraIDTier, Order = 14)
| union (c | project Field = "Support Plan", Info = MicrosoftSupportPlan, Order = 15)
| union (c | project Field = "Notes", Info = Notes, Order = 16)
| sort by Order asc
| project Field, Info