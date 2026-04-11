# The Story So Far

> Read this before anything else in the repo. It will tell you why we are building this, what we have built, and where we are going. Everything else in this repository is a chapter. This is the book.

---

## It Started With a Mailbomb

A client got hit. A mailbomb attack — hundreds of emails flooding their inbox in seconds. Defender XDR caught it and created an incident. But Sentinel never saw it. The XDR connector was not configured. The client asked why we did not catch it.

There was no good answer.

That moment exposed something bigger than a missing connector. There was no system. No standard. No documented definition of what good looked like. Every client environment was different. Every audit review required hours of manual preparation. The team was reactive, flying blind, and had no way to scale.

So we started building the operating system.

---

## What We Are Building

Not a workbook. Not a runbook. Not a set of watchlists.

A managed security program operating system — a complete, standardized, automated system for delivering world-class Sentinel MSSP services across all clients simultaneously. When it is complete an engineer can run a single command and know exactly where every client environment stands, what is working, what is broken, what is missing, and what the next move is.

The audit deck is the output — the thing you show the client. But the operating system is the foundation that makes the audit deck possible, accurate, and effortless to produce.

---

## The Moment Everything Clicks

Imagine this scenario.

A new client signs with the MSSP. Within 48 hours they receive a clean guided spreadsheet. It has every field they need to fill in — their data sources, their critical assets, their key contacts, their compliance requirements. Dropdowns for approved values. Instructions built in. No technical knowledge required.

They fill it in. The team reviews it, adds the technical fields — connector names, DCR references, detection assignments. Five watchlists get uploaded to their Sentinel workspace.

The workbook opens.

Every data source they documented is now visible with a live health status. Every detection is mapped to the sources it depends on. Every SLA commitment is tracked and measurable. Every API key and credential has an expiry date and an owner. The MITRE coverage map shows exactly where they are protected and where the gaps are.

The onboarding progress tab shows 73% complete. Exactly what is missing is visible — three sources need silent detections, two connectors need to be configured, one capability needs a dependency resolved. No manual checklist. No spreadsheet someone forgot to update. The data tells the truth.

Two weeks later everything is green. Onboarding is complete. Not because someone ticked a box — because the system says so.

A quarter later the team sits down with the client for their first audit review. The workbook opens. Everything is live and current. Nothing was prepared. No reports were pulled. No one was asked for status updates. Just open it and start talking.

The client sees their data sources, their detection coverage, their MITRE map, their costs, their dependencies. They see that three issues were caught before they noticed and resolved. They see that their Salesforce API key expires in 23 days and a rotation is already scheduled. They see that the detection covering their Azure Firewall fired six times last month and all six were legitimate threats that were triaged.

They are not asking if we are doing our job. They are asking what is next.

That is what this program builds.

---

## What We Have Built

We started with the foundation — a complete taxonomy of what a Sentinel workspace actually contains. Nine categories. A shared language for everything that follows.

Then we designed the data layer. Five watchlists that together hold every piece of curated knowledge about a client environment.

**`[mssname]-tables`** — the pipe registry. One row per table. Its only job is to know what tables exist and catch new ones appearing. Five fields. Lean and purposeful.

**`[mssname]-sources`** — where the real work lives. Every table gets at least one row here. Single source tables get one row. Shared tables like AzureDiagnostics and CommonSecurityLog get one row per sub-source. Fifteen fields covering transport, origin, silent detection assignment, monitoring frequency threshold, connector details, and more. This watchlist drives the workbook, the functions, and the master silent detection.

**`[mssname]-detections`** — the detection ledger. Every custom detection documented with its rule ID, what tables it queries, what watchlists it references, its alert class, and a plain English description. Detection team owns it. Already built and uploaded for the first client.

**`[mssname]-dependencies`** — the credential tracker. Every API key, client secret, and OAuth token that keeps data sources running. Expiry dates, rotation owners, what breaks when something expires.

**`[mssname]-client`** — the client profile. Everything you reach for when opening a support ticket or starting an audit review. Tenant ID, workspace ID, contacts, links, bio, compliance frameworks. Feeds the audit deck splash screen automatically.

---

## The Architecture

On top of the watchlists sit KQL functions — the intelligence layer. They join watchlists against live workspace data and return enriched results. The workbook never touches raw watchlists directly. It always calls a function.

Every shared table gets its own sub-function. AzureDiagnostics, CommonSecurityLog, Syslog, ThreatIntelIndicators, each ASIM table — one function each. Different filter logic inside each one but every function returns the same four columns. Standardized output regardless of how the data is filtered.

A master function unions all the sub-functions and joins against the sources watchlist for metadata. GetEnrichedInventory() goes one level higher — joining tables, sources, detections, and live health data into one complete unified view.

One call. Everything. That is the goal.

---

## The Detection Layer

Two master silent detections cover all monitoring. Both read entirely from watchlists. No hardcoded values anywhere.

The master source detection reads every row in the sources watchlist where MonitoringFrequency is set. It calls the sub-functions to get LastSeen per specific source. It fires per source that exceeds its own threshold. One detection covers every source in every environment.

Change a threshold — update the watchlist.
Add a source — update the watchlist.
The detection never changes.

For sources that need more precision — specific logic, specific alerting beyond what the master handles — custom silent detections are created and their identifier stored in the sources watchlist. The master is the floor. Custom detections are the ceiling.

---

## The Audit Deck

Nine tabs. Built from live data. Zero manual preparation.

**Splash** — client name, bio, status, contacts, onboarding date.
**SLS and Capabilities** — what was committed to and are we delivering it.
**Table Health** — what pipes exist and are they flowing.
**Data Sources** — what is flowing through each pipe with detection coverage.
**Dependencies** — credentials and expiry dates — what could silently break.
**Detections** — every detection, enabled status, 30 day fire count.
**MITRE Coverage** — heatmap, gaps, what is available but not enabled.
**Ingestion and Cost** — what is being collected, what it costs, what has no detection value.
**Opportunities** — open items, recommendations, client conversation.

The client sees proof that the team knows what they are doing. The team sees a system they can trust.

---

## The Core Principles

**Watchlists are configuration. Functions are logic. The workbook is visualization.** Nothing crosses those boundaries.

**Code never changes for configuration updates.** Adding a new source to monitor means updating the watchlist. The detection automatically adapts.

**The logs tell the truth. Connectors lie.** A connector can show green while data has silently stopped. Always query tables directly.

**Repeatable, standardized, scalable.** Every solution must pass all three tests before it is considered complete.

**Automate where the value justifies the overhead.** Not everything should be automated. Only automate when the manual alternative is genuinely unsustainable.

**Engineer for the destination.** Every decision made today should be compatible with the repository vision — version controlled, deployable with a command, AI-assisted review.

---

## Where We Are Now

The detections watchlist is built and uploaded to the first client environment. The tables watchlist is in progress. The sources watchlist build process is fully documented with an AI-assisted guide so any engineer can build it using any LLM including internal company tools.

**What comes next:**

1. Finish the tables watchlist — complete and upload
2. Build the sources watchlist — AI-assisted, table by table from the detections CSV
3. Build the dependencies watchlist — every credential documented
4. Build the client watchlist — client profile complete
5. Upload all five watchlists to Sentinel
6. Write the KQL sub-functions — one per shared table
7. Write GetSharedTableHealth() and GetEnrichedInventory()
8. Build the workbook tabs on top of the functions
9. Deliver something that makes the client's jaw drop

---

## The Vision Beyond

The same spreadsheet engineers fill in for operations becomes the onboarding intake form for new clients. Fill it in once. Upload the watchlists. The system comes alive.

Eventually everything lives in a repository. Detections, watchlists, workbooks, baseline configurations — all version controlled, all deployable with a command, all reviewable before deployment. New clients are onboarded in days not weeks. AI assists with triage and audit checks.

One client's problem becomes every client's solution. That is how you scale.

---

*This document lives at the root of the sentinel-mssp-playbook repository alongside readme.md, vision.md, architecture.md, and program-map.md. Read it first. Read everything else after.*