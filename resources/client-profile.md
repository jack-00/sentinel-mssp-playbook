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


test