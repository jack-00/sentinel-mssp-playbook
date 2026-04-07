How Clients Are Set Up Today
Each client has a dedicated Log Analytics workspace with Microsoft Sentinel deployed and operational. Initial onboarding is handled by a separate team who works directly with the client to stand up or migrate their environment. During this phase, data connectors and data sources are brought online, watchlists are built for critical assets such as domain controllers and key hosts, and the foundational content layer is established.
That content layer consists of two things. First, silent log source detections — analytics rules that confirm data is flowing from each connected source. Second, custom detections built by our detection engineering team. These are scheduled KQL queries that run against the workspace on a defined interval. When results are returned an alert is created, which generates an incident. That incident triggers a Logic App that captures the relevant data and pushes it into an internal pipeline. From there StreamSets routes it into Elastic, and the incident ultimately lands in ServiceNow where it is triaged by the SOC.
In addition to detections, clients have scheduled reports that run on a defined cadence using watchlists and Logic Apps. A threat intelligence feed is also active across environments.
This is the foundation every client sits on today.
Where We Are Headed
Microsoft is migrating Sentinel into the Defender XDR unified portal. Some clients have already made this transition. Others are in progress. This migration is not just a UI change — it introduces new capabilities including multi-stage incident correlation, unified incident queues across Microsoft security products, advanced hunting across all XDR and Sentinel data in one place, and deeper AI-assisted investigation through Copilot for Security.
Preparing for this migration correctly, and taking full advantage of what it unlocks, requires deliberate planning and a tighter operational standard than we currently have documented.
What We Are Building
This repository is the operational playbook for getting every client environment to the highest possible standard and keeping it there. It is built in four layers that are designed to be completed in sequence.
Layer 1 — Universal Sentinel Baseline
A definitive checklist of every configuration, setting, and capability that must be enabled in every Sentinel workspace regardless of client size or industry. This eliminates the "why was that never turned on" problem. It covers workspace settings, connector configuration, audit log enablement, UEBA, analytics rules, watchlist standards, automation baselines, and health monitoring. Every environment is measured against this checklist. No exceptions.
Layer 2 — Data Source Documentation and Coverage
Every client's data sources are fully documented, validated, and understood. Silent log detections confirm what is and is not flowing. Custom detections are mapped to the tables they depend on, making it clear to clients exactly what they would gain by onboarding a missing source. This layer produces a living data source inventory for each client and a tiered ingestion roadmap that drives the ongoing conversation about coverage improvement.
Layer 3 — XDR Migration and Unified Portal Readiness
A structured approach to migrating each client into the Defender XDR unified portal. This includes pre-migration validation, configuration best practices specific to the unified experience, and a post-migration checklist to confirm nothing was lost and new capabilities are properly enabled. As new features become available in the unified portal this layer evolves to capture them.
Layer 4 — Audit Deck and Continuous Improvement
Once a client has a solid foundation, the focus shifts to demonstrating value and driving ongoing growth. This layer produces a repeatable client-facing audit deck that covers detection health, incident metrics, MITRE ATT&CK coverage visualization, data gaps, and a prioritized improvement roadmap. Workbooks support the data gathering. The audit cycle creates a feedback loop where findings flow back into improving the baseline and data coverage for every client.
The Standard We Are Working Toward
A client who has moved through all four layers would have the following:

A Sentinel environment configured to the highest standard with every relevant capability enabled and documented
A complete and validated data source inventory with silent log detections confirming health
A well-organized watchlist system with critical assets tagged, documented, and actively monitored
All custom detections deployed, performing, and mapped to their data dependencies
Full migration to the Defender XDR unified portal with all applicable settings configured
Incidents flowing correctly through the pipeline into ServiceNow
A scheduled audit cadence that gives clients clear visibility into their security posture, what is working, where gaps exist, and what comes next

Constant improvement starts from a great foundation. That is what this playbook is designed to build.
