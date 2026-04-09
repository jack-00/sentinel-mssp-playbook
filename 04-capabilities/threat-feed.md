# Threat Intelligence Feed Management

> **Purpose:** This document is the complete reference for understanding, configuring, maintaining, and troubleshooting threat intelligence feeds across all managed client Sentinel environments. It is written for platform engineers and covers everything from first principles through to troubleshooting and detection design.
>
> **Why this document exists:** Threat intelligence is one of the highest-value capabilities in Microsoft Sentinel and one of the most commonly broken ones. Feeds break silently. Credentials expire without anyone noticing. Indicators go stale while the connector shows green. Clients ask why a known malicious IP did not trigger an alert and we have no good answer. This document exists so that never happens.
>
> **How this fits our program:** Every threat feed configured for a client must be documented in the DataSourceInventory watchlist, tracked in the IntegrationRegistry, monitored by a Tier 1 silent detection, and visible in the Client Security Operations Workbook. A feed that is not documented, monitored, and visible in the workbook does not exist as far as our program is concerned. This document tells you how to get it there.
>
> **Last Updated:** April 2026

---

## What Is a Threat Intelligence Feed

A threat intelligence feed is a stream of indicators — known malicious IP addresses, domains, URLs, file hashes, and email addresses — that gets ingested into Microsoft Sentinel and matched against log data in real time.

When a user visits a domain that appears in the threat feed an alert fires. When a device connects to an IP that is known C2 infrastructure an alert fires. When a file hash matches a known piece of malware an alert fires. Without a threat feed none of those alerts exist because Sentinel has no way to know that those specific indicators are malicious.

Think of it like this. Your log data is the raw evidence — who connected to what, when, from where. The threat feed is the intelligence layer that tells you which of those connections matter. Data without intelligence is noise. Intelligence without data is theory. Together they are detection.

**What a healthy threat feed does:**

- **Matching** — Live log data is compared against active indicators continuously. A match creates an alert. Without fresh indicators this matching either misses real threats or produces false positives on expired indicators.

- **Context** — Good indicators carry metadata. Threat actor name, campaign, confidence score, TLP classification, indicator type, and expiry date. This context arrives with the alert and helps analysts understand what they are dealing with in seconds rather than minutes.

- **Capability support** — Microsoft Fusion ML uses threat intelligence as one of its correlation inputs. Degraded TI quality silently degrades Fusion output even when Fusion appears healthy. UEBA enrichment also draws on TI context for entity scoring.

**What a broken threat feed looks like:**

The connector shows Connected. The ThreatIntelIndicators table has records in it. Everything looks fine. But the API key expired three weeks ago. No new indicators have arrived. The dataset is getting more stale every day. Your detections are matching against indicators that are weeks old. A new C2 IP spun up last week is completely invisible. Nobody notices until a client asks why it was not caught.

This is the core problem this document solves.

---

## The Threat Feed Landscape — Options and Use Cases

Not all threat feeds are the same. They differ in format, quality, cost, freshness, indicator type coverage, and integration method. Understanding what is available helps you match the right feed to the right client need.

### Microsoft Defender Threat Intelligence (MDTI)
**What it is:** Microsoft's built-in global threat intelligence feed. Sourced from Microsoft's security research, telemetry from billions of endpoints, and partnerships with law enforcement and industry groups.

**Cost:** Basic tier included with Sentinel at no additional cost. Premium tier requires additional licensing and provides richer context, more indicators, and threat actor profiles.

**Indicator types:** IPs, domains, URLs, file hashes.

**Freshness:** Continuous. Microsoft updates indicators in near real-time.

**Integration method:** Native connector — one click to enable.

**Best for:** Every client. This should be the baseline feed for all environments. It requires no credentials, no maintenance, and provides solid foundational coverage.

**Use case example:** A client's user visits a domain that Microsoft's global telemetry has identified as a phishing site active in the last 24 hours. Without MDTI this is just a DNS query. With MDTI it is an alert.

---

### TAXII Feeds — Commercial and Free
**What it is:** TAXII (Trusted Automated eXchange of Indicator Information) is the industry standard protocol for sharing structured threat intelligence. Most serious threat intelligence providers support TAXII 2.x. Sentinel has a native TAXII connector that polls a TAXII server on a schedule and ingests STIX 2.1 objects directly into ThreatIntelIndicators.

**Cost:** Ranges from free (CISA AIS, Abuse.ch, AlienVault OTX) to expensive commercial subscriptions (Recorded Future, CrowdStrike Falcon Intelligence, Mandiant).

**Indicator types:** Depends on provider. Best commercial feeds include IPs, domains, URLs, file hashes, threat actor profiles, and campaign relationships.

**Freshness:** Hourly for most commercial feeds. Daily for some free feeds.

**Integration method:** TAXII connector — configured with server URL, collection ID, and credentials.

**Best for:** Clients who need industry-specific or higher-confidence threat intelligence beyond what MDTI provides. Also the right method when a client brings their own commercial TI subscription.

**Free TAXII options worth knowing:**
- CISA AIS — US government threat sharing, government and critical infrastructure focus
- Abuse.ch — malware and botnet infrastructure, high quality, free
- AlienVault OTX — community-sourced, broad coverage, variable quality
- Any organization running MISP with a TAXII output configured

**Use case example:** A client in financial services subscribes to a commercial feed focused on financially motivated threat actors. The feed provides fresh indicators specific to the threats most relevant to their industry. TAXII delivers those indicators hourly directly into Sentinel.

---

### Client-Provided TIP Integration
**What it is:** Some clients have their own Threat Intelligence Platform — Anomali ThreatStream, ThreatConnect, OpenCTI, MISP, or similar. They want their existing curated intelligence to flow into Sentinel alongside other feeds.

**Cost:** Client already pays for their TIP. Integration effort falls on us.

**Indicator types:** Whatever the client's TIP contains — varies by client.

**Freshness:** Depends on how actively the client maintains their TIP.

**Integration method:** Depends on the TIP. Many support TAXII output — use the TAXII connector. Some have native Sentinel connectors in Content Hub. Some require a custom Logic App.

**Best for:** Clients who have an existing intelligence practice and want to operationalize it in Sentinel. Also clients in regulated industries who have compliance requirements around using approved intelligence sources.

**Use case example:** A healthcare client maintains a MISP instance shared across their regional health network. All member organizations contribute indicators. They want this community intelligence to flow into their Sentinel environment. MISP has a TAXII output — configure the TAXII connector against their MISP instance.

**Important:** The client owns the quality of their TIP content. We are responsible for the integration working. We are not responsible for the quality, accuracy, or freshness of what they put in their TIP. Document this boundary clearly.

---

### Our Proprietary Feed — AWS Lambda
**What it is:** Our company's internally curated threat intelligence feed. Indicators are researched, validated, and maintained by our team. Delivered to client workspaces via an AWS Lambda function that calls the Microsoft Sentinel Upload Indicators API.

**Cost:** Included in our managed service. No additional client cost.

**Indicator types:** IPs, domains, URLs, file hashes — curated for relevance across our client base.

**Freshness:** Daily delivery via scheduled Lambda execution.

**Integration method:** AWS Lambda → Sentinel Upload Indicators API → ThreatIntelIndicators. Requires an app registration in the client's Entra ID tenant with appropriate permissions.

**Best for:** Every managed client. This is a baseline deliverable for all environments. It represents direct value from our team's intelligence work and differentiates our service.

**Use case example:** Our research team identifies a new phishing campaign targeting mid-market companies using Microsoft 365. We add the phishing domains and associated IPs to our feed. The Lambda delivers them to all ten client workspaces within 24 hours. A client receives a phishing email the next morning. The domain matches our indicator. Alert fires before the user clicks.

---

### Direct API Upload
**What it is:** Indicators pushed directly to the Sentinel Upload Indicators API from any source — scripts, one-time bulk imports, or automation tools.

**Best for:** One-time uploads, bulk historical indicator imports, or custom integrations that do not fit the other methods.

**Not for:** Ongoing operational feeds. A Logic App or Lambda is more appropriate for anything that needs to run on a schedule.

---

## How This Fits Our Program

Every threat feed we configure for a client must be reflected in three places before we consider the work complete:

**DataSourceInventory watchlist** — one row per feed using ThreatIntelIndicators as the Table and the SourceSystem value as the LogSource differentiator. This is what the workbook reads to display feed health. A feed not in the watchlist is invisible to the workbook.

**IntegrationRegistry watchlist** — one row per feed with all credential details, expiry dates, and escalation contacts. This is what drives the credential expiry alerts in Tab 6 of the workbook and the 30-day notification to engineers before anything expires.

**Tier 1 Silent Detection** — an AB-series SLD rule that monitors the specific SourceSystem value in ThreatIntelIndicators and fires if indicators stop arriving within the expected window. This is non-negotiable for any feed. A feed without a silent detection can break silently for weeks.

The workbook then surfaces all of this in Tab 4 Capabilities — the threat intelligence card shows overall feed health, per-feed freshness, active indicator counts, and credential expiry dates. This is what you show the client in the audit review and what you use to demonstrate ongoing value from the TI capability.

The principle is simple: **if it is not documented, monitored, and visible in the workbook it does not exist.**

---

## The Client Conversation — Discovery Questions

When a client asks to add a new threat feed do not start configuring anything. Start asking questions. The answers determine which path to take, what you need from them, and where the line of responsibility sits. Work through these questions in order.

---

### Understanding the Feed

**What is the feed and who provides it?**

This is the most important question. The answer immediately narrows your options.

Listen for a vendor name — Recorded Future, CrowdStrike, Anomali, MISP, Abuse.ch. Listen for a description — "we have a list of bad IPs from our SOC," "we subscribe to a commercial service," "our ISAC provides indicators." Listen for a Microsoft product — "we have Defender Threat Intelligence" — this is already available natively and just needs to be enabled.

Why it matters: Some providers have native Sentinel connectors. Some support TAXII. Some only have proprietary APIs. Some are just files. The provider determines the method.

---

**What format does the feed come in?**

Listen for STIX/TAXII — structured, industry standard, cleanest integration. Listen for CSV or JSON — flat file, needs more work. Listen for REST API — provider has an endpoint you call. Listen for "I don't know" — you need to look it up or ask the provider before proceeding.

Why it matters: Format determines integration method. TAXII is always preferred. Everything else requires more engineering.

---

**How often does the feed update?**

Listen for real-time, hourly, daily, weekly, or on demand.

Why it matters: This directly determines your silent detection threshold. A daily feed that has not arrived in 25 hours is a problem. A weekly feed that has not arrived in 8 days is a problem. You need to know the expected cadence before you can design meaningful monitoring.

---

**What types of indicators does it contain?**

Listen for IPs, domains, URLs, file hashes, email addresses, or combinations.

Why it matters: Indicator type determines detection coverage. An IP-only feed cannot help Defender for Endpoint match malware. Understanding this helps you set accurate client expectations — a feed with only domains will not improve endpoint detection regardless of its quality.

---

**Does your organization already use a Threat Intelligence Platform?**

Listen for yes with a product name, no, or not sure.

Why it matters: If they have a TIP it may have TAXII output — cleanest path. It also tells you whether they have an existing intelligence practice or whether this is their first TI integration. First-time integrations need more education about what to expect.

---

### Access and Credentials

**What credentials does the feed require?**

Listen for API key, username and password, OAuth2 client ID and secret, certificate, or no credentials for public feeds.

Why it matters: Credentials expire. You need to know the type to document it correctly in IntegrationRegistry, set the right expiry alert, and plan the rotation process before anything is configured.

---

**Who owns the credentials and who can rotate them?**

This is one of the most important questions in a co-managed model. Listen for a specific person or team with a name and contact information.

Why it matters: Credential rotation requires action from whoever owns them. If you do not know who that is before configuration you will find out when the feed breaks and nobody knows what to do. Get a name and a contact before you proceed.

---

**Are there app registrations involved in Entra ID?**

Listen for yes with an existing registration, no we will need one, or not sure.

Why it matters: App registrations in the client's Entra ID tenant are outside our direct control in a Lighthouse model. We need to know early whether we need client IT involvement and who specifically can help. This is the most common blocker during configuration.

---

### Scope and Expectations

**What do you expect this feed to help detect?**

Listen for specific threats — ransomware groups, nation state actors, industry-specific threats. Listen for general coverage or a compliance requirement. Listen for "I don't know" — then educate them.

Why it matters: Sets accurate expectations. A feed with 500 indicators from a free source is not going to transform detection coverage. A high-quality commercial feed with 100,000 fresh indicators will. If a client expects magic from a free feed you need to have that conversation before configuration not after.

---

**Is this replacing an existing feed or adding to existing coverage?**

Listen for replacing — understand why the old one is going and plan the transition. Listen for adding — understand how it complements what exists. Listen for first feed — full baseline setup required including analytics rules.

Why it matters: Replacing requires running both feeds simultaneously during transition. Adding requires understanding indicator overlap. First feed requires the most work — analytics rules, silent detections, workbook updates, and client education all need to happen together.

---

### The Decision Matrix

After the discovery questions use this to select the configuration method:

| Scenario | Recommended Method | Why |
|---|---|---|
| Provider supports TAXII 2.x | TAXII connector | Open standard, clean integration, least maintenance |
| Provider has native Sentinel connector | Native connector | Check Content Hub first — Microsoft manages the connection |
| Client has TIP with TAXII output | TAXII connector against their TIP | Same as above — cleanest path |
| Provider has REST API only | Logic App — Polling | Flexible, handles most REST APIs |
| Our proprietary feed | AWS Lambda | Already architected for this |
| Microsoft Defender TI licensed | Enable MDTI connector | No credentials needed, native |
| One-time bulk upload | Upload Indicators API directly | Not for ongoing feeds |
| Client has CSV or JSON file | Logic App reading file | Automate if recurring, manual if one-time |

**Priority order — always prefer higher on the list:**
1. Native Sentinel connector
2. TAXII connector
3. Logic App — Polling
4. Custom Lambda or Function App
5. Manual upload — last resort only

---

## Configuration — Each Path in Detail

### Path A — Native Sentinel Connector

**Check first:** Before configuring anything search Content Hub for the provider name. Many commercial providers have packaged solutions that include the connector, analytics rules, and workbook templates together.

```
Sentinel → Content Hub → search provider name
Sentinel → Data Connectors → search provider name
```

**Our tasks:**
1. Install Content Hub solution if available — this gets you analytics rules and workbook templates too
2. Enable the data connector in Sentinel
3. Follow connector-specific configuration wizard
4. Verify ThreatIntelIndicators receives data — check within 24 hours
5. Run SourceSystem breakout query to confirm the feed appears correctly
6. Add row to DataSourceInventory watchlist
7. Add row to IntegrationRegistry
8. Create Tier 1 silent detection

**Client tasks:**
1. Provide any API keys or license keys required by the connector
2. Approve permissions if the connector requires app registration
3. Confirm data sharing agreement with provider if required by their terms

**Information to collect and document:**

```
From client / provider:
- API key or license key
- Account ID or workspace ID in provider portal
- Provider support contact for issues

For IntegrationRegistry:
- Connector name (exact as shown in Sentinel)
- Credential type and expiry date
- Provider support contact
- License reference
```

---

### Path B — TAXII Connector

**Our tasks:**
1. Navigate to Sentinel → Data Connectors → Threat Intelligence — TAXII
2. Click Add New and configure:
   - Friendly name — use provider name for clarity
   - TAXII server root URL
   - Collection ID
   - Username and API key or password
   - Polling frequency — match to provider's update cadence
3. Save and monitor — verify data flows within first polling interval
4. Run SourceSystem breakout to confirm feed appears with expected SourceSystem value
5. Add row to DataSourceInventory watchlist
6. Add row to IntegrationRegistry with credential expiry date
7. Create Tier 1 silent detection with threshold based on polling frequency

**Client tasks:**
1. Obtain TAXII server URL from provider portal or documentation
2. Obtain collection ID from provider portal
3. Obtain API key or credentials from provider account
4. Confirm polling is permitted under provider terms — some providers limit polling frequency
5. Confirm if provider requires IP allowlisting for our connector

**Information to collect and document:**

```
From provider portal or documentation:
- TAXII Server Root URL
- Collection ID
- Authentication type: API Key / Username + Password
- Credential value — store securely
- Credential expiry date
- Polling frequency allowed
- Provider IP allowlist requirements
- Provider support contact

For IntegrationRegistry:
- All of the above plus rotation owner and process
```

---

### Path C — Logic App Polling

**Our tasks:**
1. Create Logic App with recurrence trigger set to polling frequency
2. Add HTTP action to call provider API with authentication
3. Parse API response and transform to STIX indicator format if needed
4. Add Upload Indicators API action to write to ThreatIntelIndicators
5. Add error handling — failure should notify SOC via Teams not fail silently
6. Test end to end — trigger manually and verify indicators arrive
7. Add row to DataSourceInventory watchlist
8. Add row to IntegrationRegistry — include Logic App name and resource group
9. Create Tier 1 silent detection

**Client tasks:**
1. Provide API endpoint URL from provider
2. Provide API key or OAuth credentials
3. If OAuth2 required — coordinate with client IT team to create app registration
4. Approve Logic App creation in their Azure subscription if required by their governance process

**Information to collect and document:**

```
From provider:
- API endpoint URL
- Authentication method
- API key or OAuth credentials
- Rate limits and polling constraints
- Response format documentation — needed to build parse action

If OAuth2 app registration required:
- Client IT team contact who can create app registration
- Agreed app registration name
- Required permissions
- Client secret expiry policy

For IntegrationRegistry:
- Logic App name
- Logic App resource group
- API endpoint
- Credential type and expiry
- App registration name and client ID if applicable
- App registration owner in client IT
- Rotation process and agreed schedule
```

---

### Path D — Our Proprietary Lambda Feed

**Our tasks:**
1. Coordinate with client IT team to create app registration in their Entra ID tenant
2. Verify required permissions are assigned — Sentinel Contributor or TI write permissions
3. Obtain client secret from client IT — store securely, never in plain text
4. Configure Lambda function environment variables:
   - TENANT_ID
   - CLIENT_ID
   - CLIENT_SECRET
   - WORKSPACE_ID
   - RESOURCE_GROUP
5. Trigger Lambda manually — verify first delivery
6. Confirm our SourceSystem value appears in ThreatIntelIndicators
7. Add row to DataSourceInventory watchlist
8. Add row to IntegrationRegistry with secret expiry date and rotation owner
9. Brief client IT team — app registration name, purpose, do not modify without notifying our team
10. Create Tier 1 silent detection
11. Set 30-day expiry alert in IntegrationRegistry

**Client tasks:**
1. Provide point of contact in IT who can create and manage app registrations
2. Create app registration in Entra ID with agreed name
3. Assign required permissions
4. Generate client secret and share securely
5. Confirm app registration must not be modified or deleted without notifying our team
6. Participate in client secret rotation on agreed schedule

**Information to collect and document:**

```
From client IT team:
- Tenant ID: Entra ID → Overview
- Workspace ID: Log Analytics workspace → Overview
- App registration name
- App registration client ID once created
- Client secret value — store in secure vault, never plain text
- Agreed rotation schedule
- IT contact name and contact for future rotations

For IntegrationRegistry:
- All of the above
- Lambda function ARN
- Client secret expiry date
- Rotation owner and process
```

---

## Detections and Silent Monitoring

### What to Consider When Designing TI Detections

Before creating any detection against ThreatIntelIndicators understand what you actually have in the table. A detection written against indicators that do not exist in your feed will never fire. A detection written against stale indicators will produce false positives.

**Step 1 — Know your indicator types**

Run this before writing any detection:

```kql
ThreatIntelIndicators
| where TimeGenerated > ago(30d)
| where ExpirationDateTime > now()
| summarize Count = count() by SourceSystem, IndicatorType
| order by SourceSystem asc, Count desc
```

If you have no domain indicators do not write a detection that matches domains. If you have no file hashes do not write a detection that matches file hashes. Write detections that match what you have.

**Step 2 — Understand freshness before trusting matches**

An indicator that expired last month is still in the table. A detection that does not filter on ExpirationDateTime will match against expired indicators and create false positives.

Always include this filter in TI detection queries:

```kql
| where ExpirationDateTime > now()
```

**Step 3 — Understand confidence before acting on matches**

Different feeds have different confidence levels. Some indicators are high confidence — confirmed malicious infrastructure used by known threat actors. Some are low confidence — seen once in community telemetry, may be false positive.

If your feed provides confidence scores use them:

```kql
| where ConfidenceScore >= 75 // Only act on high confidence indicators
```

**Step 4 — Match at the right layer**

Different indicator types match against different tables:

| Indicator Type | Best Table to Match Against |
|---|---|
| IP address | CommonSecurityLog, AzureFirewallDnsProxy, DeviceNetworkEvents, SignInLogs |
| Domain | DNSQueryLogs, AzureFirewallDnsProxy, DeviceNetworkEvents |
| URL | UrlClickEvents, EmailUrlInfo, DeviceNetworkEvents |
| File hash | DeviceFileEvents, DeviceProcessEvents, EmailAttachmentInfo |
| Email address | EmailEvents |

Match indicator type to the table that actually contains that type of data in this client's environment. If DNSQueryLogs does not exist for this client a domain-matching detection will never fire.

---

### Silent Detection Design for Threat Feeds

Every configured feed is Tier 1. A feed going silent without detection means threat matching stops. This is a critical operational failure.

**The key principle:** Do not use TimeGenerated for freshness monitoring of batch feeds. Use `ingestion_time()` to understand when data actually arrived. For daily batch feeds like our Lambda the gap between the event timestamp and the arrival time can be significant.

**Silent detection template for any TI feed:**

```kql
// AB##### - SLD - [Feed Name] Silence
// Tier 1 — fires if feed has not delivered new indicators
// within expected window
// Threshold: set to 1.5x the expected delivery frequency
// Daily feed = 36 hour threshold
// Hourly feed = 2 hour threshold
let threshold = 36h; // Adjust per feed frequency
let lastDelivery = ThreatIntelIndicators
| where SourceSystem == "[FEED_SOURCE_SYSTEM_VALUE]"
| summarize LastSeen = max(TimeGenerated);
lastDelivery
| extend HoursSinceDelivery = datetime_diff('hour', now(), LastSeen)
| where HoursSinceDelivery > toint(threshold / 1h)
| project
    TimeGenerated = now(),
    AlertName = "AB##### - SLD - [Feed Name] Silence",
    Description = strcat(
        "[Feed Name] has not delivered new indicators in ",
        tostring(HoursSinceDelivery),
        " hours. Last delivery: ",
        tostring(LastSeen),
        ". Expected frequency: [frequency]. Investigate feed pipeline immediately."
    ),
    Severity = "High",
    SourceSystem = "[FEED_SOURCE_SYSTEM_VALUE]"
```

**What to set the threshold to:**

| Feed Frequency | Recommended Threshold | Reasoning |
|---|---|---|
| Real-time / Continuous | 2 hours | Short gap catches problems fast |
| Hourly | 3 hours | Two missed polls before alerting |
| Daily | 36 hours | Allows for minor delays, catches day-long outages |
| Weekly | 10 days | Buffer for planned maintenance |

**Additional check — indicator freshness not just delivery:**

A feed can deliver indicators but all of them are expired. A second silent detection catches this:

```kql
// AB##### - SLD - [Feed Name] Stale Indicators
// Fires if feed has no active unexpired indicators
ThreatIntelIndicators
| where SourceSystem == "[FEED_SOURCE_SYSTEM_VALUE]"
| summarize
    TotalIndicators = count(),
    ActiveIndicators = countif(ExpirationDateTime > now())
| where ActiveIndicators == 0 and TotalIndicators > 0
| project
    TimeGenerated = now(),
    AlertName = "AB##### - SLD - [Feed Name] Stale Indicators",
    Description = "All indicators from [Feed Name] have expired. Feed is delivering data but no active indicators exist for matching. Investigate indicator expiry configuration.",
    Severity = "Medium"
```

---

## Troubleshooting

For every feed there are two categories of problem — ours and theirs. Work through our side completely before escalating. An escalation to a client or provider should always come with evidence that our side is fully healthy.

### Universal First Step

Before anything else run this to understand exactly what is broken:

```kql
ThreatIntelIndicators
| where SourceSystem == "[FEED_SOURCE_SYSTEM]"
| summarize
    LastDelivery = max(TimeGenerated),
    TotalIndicators = count(),
    ActiveIndicators = countif(ExpirationDateTime > now()),
    StaleIndicators = countif(ExpirationDateTime <= now())
| extend HoursSinceDelivery = datetime_diff('hour', now(), LastDelivery)
```

| Result | Meaning | Next Step |
|---|---|---|
| LastDelivery recent, ActiveIndicators > 0 | Feed is healthy | Problem is elsewhere — check analytics rules |
| LastDelivery recent, ActiveIndicators = 0 | Delivering but all expired | Freshness issue — check indicator ExpirationDateTime config |
| LastDelivery old, TotalIndicators > 0 | Feed stopped updating | Pipeline issue — work through troubleshooting below |
| No results at all | Feed never worked or wrong SourceSystem | Check SourceSystem value — run without filter to see all values |

---

### Native Connector Troubleshooting

**Our side — check first:**

```kql
SentinelHealth
| where SentinelResourceType == "Data Connector"
| where SentinelResourceName contains "[CONNECTOR_NAME]"
| summarize LastStatus = arg_max(TimeGenerated, Status, Description)
    by SentinelResourceName
```

- Connector shows healthy but no indicators → Provider may have changed API — check for Content Hub updates
- Connector shows disconnected → Re-enable connector, check if permissions changed

**Their side — escalate if:**
- Connector is healthy on our side
- API is reachable
- No indicators arriving despite healthy connector
- Contact provider support with connector health evidence

---

### TAXII Connector Troubleshooting

**Our side — check first:**

```
Sentinel → Data Connectors → Threat Intelligence — TAXII
→ Find the feed entry
→ Check last successful poll timestamp
→ Look for error messages
→ Check IntegrationRegistry for credential expiry date
```

| Symptom | Likely Cause | Our Fix |
|---|---|---|
| No indicators since specific date | API key expired | Rotate key with provider, update connector |
| Connector showing error | URL or collection ID changed | Get updated details from provider, reconfigure |
| Partial indicators only | Collection ID changed or wrong | Verify collection ID with provider |
| Indicators arriving but all stale | Provider not updating their feed | Escalate to provider — their problem |

**Their side — escalate if:**
- Our credentials are valid
- URL and collection ID are correct and confirmed with provider
- Connector is polling but provider collection has no new data
- Provider's feed is not being updated — this is entirely their problem

---

### Logic App Troubleshooting

**Our side — check first:**

```
Azure portal → Logic Apps → [Logic App Name] → Run History
→ Are runs succeeding or failing?
→ Click a failed run → identify which step failed
```

```kql
AzureDiagnostics
| where ResourceType == "WORKFLOWS"
| where status_s == "Failed"
| where Resource contains "[LOGIC_APP_NAME]"
| summarize Count = count(), LastFailure = max(TimeGenerated)
    by error_message_s
| order by Count desc
```

| Failure Step | Likely Cause | Our Fix | Escalate If |
|---|---|---|---|
| HTTP 401 | API key expired | Rotate key, update Logic App | Key is valid but 401 persists — provider auth issue |
| HTTP 404 | Endpoint URL changed | Get new URL from provider | Provider changed API without notice |
| HTTP 429 | Polling too frequent | Reduce polling frequency | Provider rate limit too restrictive |
| Parse failed | Response format changed | Update parse action | Provider changed format without notice |
| Upload 403 | App registration permissions changed | Check permissions | Client IT modified app registration |
| Upload invalid workspace | Workspace ID changed | Update Logic App config | Client moved workspace without notifying us |

**Their side — escalate if:**
- Logic App runs are succeeding with 200 responses
- Upload to Sentinel is returning success
- But indicators are not appearing in ThreatIntelIndicators
- This is a Sentinel ingestion pipeline issue — open Microsoft support ticket

---

### Lambda Feed Troubleshooting

**Our side — check first:**

```
AWS Console → Lambda → [Function Name] → Monitor → CloudWatch Logs
→ Look for authentication errors
→ Look for HTTP error codes
→ Verify execution schedule is still active
```

```kql
ThreatIntelIndicators
| where SourceSystem == "[OUR_SOURCE_SYSTEM]"
| summarize LastDelivery = max(TimeGenerated)
| extend HoursSince = datetime_diff('hour', now(), LastDelivery)
```

| Symptom | Likely Cause | Our Fix | Escalate If |
|---|---|---|---|
| AADSTS70011 in CloudWatch | Client secret expired | Coordinate rotation with client IT | Client IT must execute rotation |
| 403 on Sentinel API | App registration permissions changed | Investigate what changed | Client IT modified app registration |
| Function not triggering | Lambda schedule issue | Fix schedule in AWS | N/A — this is our infrastructure |
| Function times out | Network or Sentinel API latency | Increase timeout limit | Sentinel API slow — Microsoft issue |
| Invalid workspace error | Workspace ID changed | Update Lambda config | Client changed workspace without notifying us |

**The client secret rotation process — our most common Lambda failure:**

```
Step 1 — Confirm secret is expired
  Check IntegrationRegistry for expiry date
  Confirm AADSTS70011 error in CloudWatch logs

Step 2 — Notify client IT rotation owner
  Use contact documented in IntegrationRegistry
  Provide app registration name and client ID
  Request new secret with agreed expiry period

Step 3 — Client IT generates new secret in Entra ID
  Client IT shares new secret value via secure method
  Never accept credentials via email or Teams chat

Step 4 — Update Lambda environment variables
  Update CLIENT_SECRET in Lambda configuration
  Do not restart or redeploy — environment variable update takes effect immediately

Step 5 — Verify
  Trigger Lambda manually
  Confirm indicators appear in ThreatIntelIndicators
  Update IntegrationRegistry with new expiry date
  Confirm 30-day alert is reset for new expiry date
```

**Their side — escalate if:**
- Lambda executes successfully with no errors
- Authentication returns valid token
- Sentinel API call returns 200
- Indicators are not appearing in ThreatIntelIndicators
- Open Microsoft support ticket — Sentinel ingestion pipeline issue

---

## Documentation Checklist — End of Feed Onboarding

Nothing is done until everything below is checked. A feed that is configured but not documented, monitored, and visible in the workbook is not complete.

**DataSourceInventory watchlist:**
- [ ] One row per feed in ThreatIntelIndicators — identified by SourceSystem breakout
- [ ] LogSource, Origin, Transport, Category, Purpose all filled in
- [ ] SilentDet field has the AB-series detection ID
- [ ] DateAdded set to today
- [ ] Vetted = Yes
- [ ] Notes include feed type, polling frequency, and any environment-specific context

**IntegrationRegistry watchlist:**
- [ ] One row per feed
- [ ] Credential type and expiry date documented
- [ ] Rotation owner documented with contact information
- [ ] Rotation process and agreed schedule noted
- [ ] Logic App name or Lambda ARN noted for operational reference

**Silent Detections:**
- [ ] Tier 1 delivery silence detection created — AB##### - SLD - [Feed Name] Silence
- [ ] Tier 1 stale indicators detection created — AB##### - SLD - [Feed Name] Stale Indicators
- [ ] Thresholds set based on feed polling frequency
- [ ] Both detections are enabled and showing Healthy in SentinelHealth
- [ ] Both AB numbers documented in DataSourceInventory SilentDet field

**Client Communication:**
- [ ] Client IT informed of any app registrations created — name, purpose, do not modify
- [ ] Rotation owner confirmed and documented
- [ ] Rotation schedule agreed and in IntegrationRegistry
- [ ] Client security team understands what the feed provides
- [ ] Client expectations set accurately around detection coverage and indicator quality

---

*This document is maintained in sentinel-mssp-playbook under 04-capabilities. Update it when new feed types are onboarded, when provider configurations change, when the Lambda architecture changes, or when troubleshooting reveals new failure patterns not documented here.*