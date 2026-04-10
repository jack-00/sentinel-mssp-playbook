# Program Vision — Sentinel MSSP Operating System

> **Purpose:** This document captures the full vision for what this program is building and why. It is the north star. Every technical decision, every watchlist field, every workbook tab, every automation exists to serve this vision. When you are unsure why something is being built read this document first.
>
> **Last Updated:** April 2026

---

## What We Are Actually Building

This is not a workbook project. This is not a detection project. This is not a reporting project.

This is a **managed security program operating system** — a complete, standardized, automated system for delivering world-class Sentinel MSSP services across all clients simultaneously. When it is complete an engineer can run a single command and know exactly where every client environment stands, what is working, what is broken, what is missing, and what the next move is. No tribal knowledge required. No manual preparation. No guesswork.

The audit deck is the output — the thing you show the client. But the operating system is the foundation that makes the audit deck possible, accurate, and effortless to produce.

---

## The Problem We Are Solving

We are new to this. We do not have a documented baseline of what good looks like. We do not have standardized systems. We are reactive. We are manually creating audit content before every client review. We do not have confidence that we know what is actually deployed, what is working, and what the client is entitled to.

This creates real problems:

- We cannot scale — each client requires custom manual effort
- We cannot be confident — we are not sure if we are meeting our commitments
- We cannot innovate — we spend all our time maintaining instead of improving
- We cannot grow — onboarding a new client means reinventing everything
- We are reactive — we find out about problems when clients do

**The goal of this program is to eliminate all of those problems permanently.**

---

## The Three Phases

### Phase 1 — Baseline and Standard

Define exactly what a good Sentinel environment looks like. Document every decision and why it was made. Create a repeatable, standardized baseline that every client environment is measured against.

**What this means:**
- A complete documented baseline checklist — every connector, every diagnostic setting, every capability, every optimization that should be enabled out of the box
- The why behind every decision — not just what to do but why it matters
- A standard we can defend to a client, to an auditor, to a new team member
- A client environment that is tight, optimized, and ready as a platform for data sources and detections

**What good looks like:**
When a new client is onboarded we run the baseline process and within a defined time window their environment meets every standard. No exceptions without documented justification. Any gap has an owner, a remediation plan, and a target date.

---

### Phase 2 — Visibility and Control

Build the systems that give us real-time visibility into every client environment simultaneously. Know exactly what is there, what is working, what is missing, and what has changed.

**What this means:**
- DataSourceInventory, DetectionCatalog, LogSourceRegistry — the watchlist backbone
- GetEnrichedInventory and supporting KQL functions
- Silent detection coverage for every tracked source
- The living workbook — open it and the truth is right there
- Automated change detection — something new appears, something goes inactive, something breaks — we know before the client does

**What good looks like:**
At any point in time an engineer can open the workbook for any client and immediately answer:
- What data sources are active and healthy
- What detections are running and what tables they depend on
- What capabilities are fully operational
- What has changed since last week
- What is our SLS commitment and are we meeting it
- What integrations are expiring and when

No preparation required. No manual data gathering. The system knows.

---

### Phase 3 — Value Delivery and Innovation

With the foundation solid and visibility complete we shift from reactive maintenance to proactive value delivery. This is where the real MSSP differentiation happens.

**What this means:**
- Audit deck that builds itself from live data — no manual preparation
- Detection coverage optimization — are we using all the data we are collecting
- Cost optimization — are we collecting data that nothing depends on
- MITRE coverage visibility — where are the gaps and what would close them
- Client-specific recommendations — based on their vertical, their risk profile, their business
- Repository-based management — push changes to all clients from one place
- AI-assisted triage and audit checks
- Automated onboarding for new clients

**What good looks like:**
The audit deck is generated in minutes not hours. The client meeting is a strategic conversation about improvement and innovation not a status update about what broke. The engineer is thinking about what is next not what is on fire.

---

## The Audit Deck Vision

The audit deck is the moment where all of this becomes visible to the client. It is not a report — it is a conversation. It is proof that we know what we are doing, that we are on top of everything, and that we are delivering real value.

### The Client Experience

A client sitting in an audit review should walk away knowing seven things:

**1. We are using everything they are paying for**
Every feature, every license entitlement, every capability that is available to them is either enabled and working or there is a documented decision about why it is not. Nothing is being left on the table. We have taken inventory of what they have access to and we have made deliberate choices.

**2. We know exactly what was agreed upon and we are delivering it**
Our SLS commitment is documented, visible, and measurable. Here is what we promised. Here is how we are doing against it. No ambiguity. No surprises. If we are meeting it we show the evidence. If we are not we have a plan.

**3. Their data is protected and monitoring is in place**
Every data source is tracked. Every tracked source has a health monitor. When something breaks we know before they do. Here is the silent detection coverage. Here is the last time something went inactive and how quickly we caught it. This is how the monitoring works and this is what we are responsible for versus what their team is responsible for.

**4. We are making full use of their data**
Every data source they are paying to collect has detections running against it. Here is the detection coverage per source. Here is what we are catching. Here is the MITRE coverage map — these are the tactics and techniques we can detect, these are the gaps, and here is what it would take to close them. We are not collecting data and ignoring it.

**5. Their money is being spent wisely**
Here is what is being ingested and what it costs. Here is what has detection coverage and what does not. Here are the optimization opportunities — sources that cost money but produce no detection value. Here are the noisy detections that might benefit from tuning. Here is what we recommend.

**6. We are evolving with the platform**
We are staying current with Microsoft's direction. Here is how we are leveraging the Defender portal migration. Here is what new capabilities we have evaluated or adopted. Here is what is on our roadmap. We are not standing still.

**7. We understand their business**
We know their vertical. We know what is most important to them. Our detection priorities and recommendations are aligned with their specific risk profile. We are not giving them a generic service — we are giving them a service that understands their business.

---

### The MSSP Experience

From our side the audit deck must be effortless to produce. Not because we are lazy but because manual preparation time is time stolen from actual security work.

**The audit deck is generated not created.**

Open the workbook. The data is live and current. Take the key metrics. Add the narrative. Done. Total preparation time — minutes not hours.

This is only possible because the operating system underneath is solid. The watchlists are current. The functions are running. The workbook is live. The data tells the truth automatically.

---

## The Audit Deck Structure

### Splash Screen — Client Overview
The first thing visible when the deck opens. Sets the context for everything that follows.

- Client name and company logo
- One paragraph bio — who they are, their industry, why security matters to them
- Onboarding date — how long we have been their partner
- Primary escalation contact — who to call when something is critical
- Key links — Sentinel workspace, SharePoint folder, ServiceNow, important portals
- Current overall status — one indicator: Healthy / Needs Attention / Critical
- Data sources tracked — total count
- Last audit date and next audit date

This slide answers: who is this client and are they fundamentally healthy right now.

---

### Section 1 — Platform Health
**Are we taking care of the platform?**

- Baseline compliance — X of Y baseline items complete. What is missing and why.
- Data source health — X of Y sources active. Inactive and review sources called out.
- Silent detection coverage — X of Y sources have active monitoring.
- Capability status — UEBA, Threat Intelligence, Fusion ML, SOC Optimization. Each one healthy or degraded with dependency chain visible.
- Integration health — all API keys and credentials current. Next expiry date.
- Recent incidents or issues — anything that broke, how quickly we caught it, how we resolved it.

This section answers: is the platform in good shape and are we doing our job.

---

### Section 2 — SLS Commitment
**Are we meeting what we promised?**

- List every item in our SLS commitment
- Current status against each item — meeting / at risk / not meeting
- Evidence for each item — not just a status, actual data
- Any items not meeting SLS — root cause, remediation plan, target date

This section answers: did we do what we said we would do.

---

### Section 3 — Detection Coverage
**Are we making full use of their data?**

- Total detections active — split by Security and Operational
- Detection coverage by data source — which sources have detections, which do not
- MITRE coverage map — tactics and techniques covered, gaps highlighted
- Available detections not yet enabled — what we could turn on and what data it would require
- Detection performance — top firing detections, true positive rate, tuning opportunities
- Noisy detections — anything firing excessively that may need tuning

This section answers: are we using their data well and where can we improve.

---

### Section 4 — Cost and Value
**Is their money being spent wisely?**

- Ingestion cost breakdown by data source — last 30 days
- Cost vs coverage — sources with high ingestion and zero detections flagged
- Month over month trend — is cost increasing, stable, or decreasing
- Optimization recommendations — specific sources where cost can be reduced without losing coverage
- Value delivered — incidents caught, threats detected, response time metrics

This section answers: are we being good stewards of their budget.

---

### Section 5 — Watchlist Review
**Do we have the right reference data?**

- Critical assets watchlist — current count, last reviewed date, opportunity for client to review and update
- Domain controllers watchlist — current list, any changes since last review
- VIP users watchlist — current list, any changes
- Any other client-specific watchlists — status and last review

This section answers: is our reference data current and accurate.

---

### Section 6 — Roadmap and Recommendations
**Where are we going?**

- Top three recommendations for this quarter — specific, actionable, with expected value
- Detections available to unlock — what data would enable them
- Platform improvements planned — what we are working on
- Industry context — relevant threats to their vertical, how our coverage addresses them
- Innovation updates — new Microsoft capabilities evaluated or adopted

This section answers: what is next and why does it matter.

---

## The Client Profile Watchlist

You mentioned this and it is exactly right. A watchlist that stores everything you need to know about a client that does not come from Sentinel. The information you reach for when opening a support ticket, writing a client email, or starting an audit review.

**`[mssname]-client`**

| Field | What Goes Here |
|---|---|
| ClientName | Full legal name |
| ShortName | What we call them day to day |
| Industry | Their vertical — financial services, healthcare, manufacturing etc |
| Bio | Two to three sentences about who they are and why security matters to them |
| OnboardingDate | When they became a client |
| TenantID | Azure tenant ID |
| WorkspaceID | Log Analytics workspace ID |
| WorkspaceName | Workspace name |
| SubscriptionID | Primary Azure subscription ID |
| SentinelURL | Direct link to their Sentinel workspace |
| SharePointURL | Link to their client folder in SharePoint |
| ServiceNowURL | Link to their ServiceNow instance or queue |
| PrimaryContact | Main security contact name and email |
| EscalationContact | Who to call for critical incidents |
| TechnicalContact | Client IT contact for things like app registrations |
| SLSDocument | Link to the signed SLS agreement |
| ComplianceFrameworks | Comma separated list of applicable frameworks |
| Notes | Anything else worth knowing |

This watchlist feeds the splash screen of the audit deck automatically. Open the deck and the client context is right there.

---

## The Logic App and ServiceNow Pipeline

You mentioned the Logic App feeding ServiceNow has issues. This is a critical piece of the operating system — it is how incidents become tickets, how alerts become action, how the SIEM connects to the service management layer.

Before we rebuild it we need to understand what is broken and what the ideal state looks like. Some things to consider:

- Should every Sentinel incident create a ServiceNow ticket or only certain severities
- What enrichment should happen before the ticket is created — client name, affected assets, recommended actions
- Should the ticket include a link back to the Sentinel incident
- Should ticket updates sync back to Sentinel — comment on the incident when the ticket is resolved
- Is the current Logic App approach the right architecture or should we consider the native Sentinel ServiceNow connector

This deserves its own document and its own session. Flag it as a priority item in PROGRAM-MAP.

---

## The Onboarding Process

When this program is complete onboarding a new client should be a defined repeatable process with a clear timeline and a clear definition of done.

The onboarding process will:
1. Deploy the baseline configuration — all standard connectors, diagnostic settings, capabilities
2. Run the data source audit — populate DataSourceInventory
3. Build the DetectionCatalog — catalog all custom detections deployed
4. Complete the client questionnaire — document requirements and SLS commitment
5. Upload all watchlists — client is now in the system
6. Verify silent detection coverage — every SLS source has monitoring
7. Deliver the first audit deck — baseline established, client knows where they stand
8. Schedule recurring audit cadence — quarterly reviews, monthly health checks

The onboarding timeline, checklist, and definition of done will be documented in a dedicated onboarding guide. This is a future document — build it once the baseline and visibility phases are complete.

---

## The Repository Vision

The endgame is managing everything from a repository. Detection changes, watchlist updates, workbook deployments, baseline configurations — all version controlled, all reviewable, all deployable with a command.

This means:
- Detection KQL stored as files in the repo — deployed via Bicep or Terraform to all client workspaces
- Watchlist CSV files stored in the repo — deployed via the Sentinel API
- Workbook JSON stored in the repo — deployed via ARM template
- Baseline configuration stored as Bicep — deployed to new clients during onboarding
- All changes reviewed via pull request — no undocumented changes to client environments
- AI-assisted review — flag potential issues before deployment

This is the long game. It requires the operating system to be solid first. But every decision we make now should be compatible with this future state.

---

## Guiding Principles

**Baseline first.** You cannot improve what you have not defined. Define what good looks like before trying to be great.

**Systems over heroics.** A repeatable system beats a brilliant engineer every time. Build systems that work even when the best engineer is not in the room.

**Earn the trust, then innovate.** The client needs to trust that the basics are covered before they care about innovation. Get the baseline right. Get the SLS right. Then bring the innovation.

**The workbook does not lie.** Manual reports can be massaged. Live data cannot. Everything in the audit deck comes from live data or it does not belong in the audit deck.

**Engineer for the destination.** Every decision made today should be compatible with the repository vision. Do not build things that will need to be thrown away.

**Automate what scales, document what does not.** Not everything can be automated yet. But everything can be documented. A documented manual process is one step away from an automated one.

**One client's problem is every client's opportunity.** When you solve a problem for one client build the solution so it works for all of them. That is how you scale.

---

*This document is the north star for the sentinel-mssp-playbook program. It does not contain implementation details — those live in their respective documents. This document captures the vision, the why, and the destination. Update it when the vision evolves. Reference it when decisions get hard.*