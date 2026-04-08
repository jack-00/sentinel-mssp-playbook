# sentinel-mssp-playbook

> *The operational playbook for managing Microsoft Sentinel and Defender XDR environments like a world-class security team.*

---

## The Incident That Started It All

It was a regular Tuesday. A client's Microsoft Defender XDR portal lit up with a critical incident — a mailbomb attack. Alerts firing. Correlation engine working exactly as designed. XDR saw it, flagged it, and created an incident.

We did not.

No ticket in ServiceNow. No alert in our SOC queue. No Teams message. Nothing. The client's XDR portal was screaming and our Sentinel environment was completely silent — because the Microsoft Defender XDR data connector was never configured. XDR and Sentinel were not talking to each other. An entire product's worth of detection capability was invisible to our pipeline.

The client noticed before we did.

That is the kind of moment that makes you question everything. Not because it was catastrophic — it was caught and handled — but because it was *silent*. Nobody broke anything. Nobody made a mistake. A configuration gap that nobody knew about let a critical incident slip through a crack that should never have existed.

This playbook is the answer to that moment. Not just a fix for one connector — a complete, systematic approach to making sure that kind of silence never happens again.

---

## What We Are Building

This is not a collection of documentation for documentation's sake. This is a living operational system — built in layers, each one depending on the one before it, designed to take any managed Sentinel environment from "running" to "exceptional."

Four layers. Built in order. No skipping ahead.

**Layer 1** gets every workspace configured correctly and verified with proof. Every connector. Every audit log. Every capability. Every health check. Signed off by an engineer who actually ran the queries.

**Layer 2** establishes the complete data source inventory — not by connector status, but by log source. Because a connector showing green means nothing if the data stopped flowing three weeks ago.

**Layer 3** catalogs every detection we have built — what it does, what data it needs, what it catches, what goes blind without it. The invisible made visible.

**Layer 4** turns all of that into a client experience that speaks for itself.

And along the way — an automated pipeline that watches the environment continuously, takes weekly snapshots, and alerts us the moment anything changes. More on that in a moment.

---

## Why Baselines Change Everything

Here is the truth about most managed Sentinel environments: they were set up once, during onboarding, by a team that was moving fast and had a lot of clients to get through. Most things got configured. Some things did not. A few things got missed entirely. And because Sentinel does not yell at you when something is not configured — it just stays quiet — nobody ever found out.

The baseline checklist in this playbook is the antidote to that. It is opinionated, specific, and non-negotiable. Every workspace gets validated against it. Every item either passes or gets documented with a reason why it does not apply. Nothing is simply forgotten.

But the baseline is not just a checklist — it introduces a concept that changes how you think about Sentinel environments entirely: **Capabilities**.

UEBA is not a setting you toggle. It is a capability that depends on SignInLogs, AuditLogs, SecurityEvents, MDI sensors, and Entra ID sync all working correctly at the same time. You can enable UEBA and it will appear active. But if SignInLogs stop flowing three weeks later, UEBA keeps running — against stale data — and the behavioral baselines quietly degrade. Everything looks green. Nothing is actually working.

The baseline makes you verify not just that capabilities are enabled — but that every dependency is healthy. Because a capability is only as strong as its weakest dependency.

Once baseline is achieved, you have something you have probably never had before: **solid ground**. A documented, verified, signed-off foundation that every future improvement is built on top of. You know what you have. You know it works. And you can prove it.

---

## The Audit — From the Driver's Seat

It is a Tuesday afternoon. You are on a Teams call with the client who watched their XDR portal fire a critical incident while your pipeline sat silent. The relationship has been tense. They are professional about it but you can feel the doubt. They are wondering if they made the right choice.

You share your screen.

You open Microsoft Sentinel. You navigate to Workbooks. You open one called **Client Security Operations Review**.

No spreadsheet. No slides. No report you stayed up assembling the night before. Just a live dashboard — pulling real data from their workspace, right now, in real time.

---

**The Summary Tab loads first.**

Before you say a single word the client can see it. A clean header. Their company name. Today's date. And a row of metric cards:

- **Data Sources: 47 of 49 healthy**
- **Capabilities: 4 of 4 active**
- **Detections Active: 72 of 83**
- **Silent Detections: 49 of 49 healthy**
- **Integrations: 6 of 7 valid**

Below the cards — a source group health table. Identity: 8 of 8. Endpoint: 12 of 12. Firewall: 5 of 5. Network: 3 of 4. One row is yellow.

You let them look at it for a moment. Then you say:

*"This is a live view of your environment right now. Not a report we prepared. Not a snapshot from last week. This is what your security data looks like at this exact moment."*

---

**You click the yellow Network row.**

Tab 2 loads — filtered to Network sources. Four rows green. One row red.

`CommonSecurityLog — Fortinet VPN — 🔴 Inactive — Last seen 11 days ago`

You do not wait for them to ask. You already know.

*"This one we caught automatically. Your Fortinet VPN stopped sending logs eleven days ago. Our system flagged it, we investigated, and we identified the cause — a configuration change on the VPN concentrator during last month's maintenance window shifted the Syslog destination IP. We have a fix scheduled for tomorrow morning. Your perimeter firewall is still healthy — this is specifically the VPN authentication log feed."*

The client nods. They were not going to bring it up but they had noticed something felt off with VPN logging. You got there first.

---

**You navigate to Tab 3 — Silent Detections.**


49 of 49. Every single log source has a corresponding detection that monitors it for silence. If any source stops sending data — even a source nobody is actively watching — a detection fires, an incident is created, and your SOC gets a Teams message within the hour.

*"We did not find out about your Fortinet issue because someone happened to check a dashboard. We found out because a detection we built specifically for that log source fired and told us. That is how this works now."*

The client leans forward slightly. Something shifted.

---

**Tab 4 — Capabilities.**

Four cards. UEBA. Custom Threat Feed. Fusion ML. SOC Optimization. All green except UEBA which is showing yellow.

You click it. The dependency chain expands:

- SignInLogs → 🟢
- AuditLogs → 🟢
- SecurityEvents → 🟢
- MDI Sensors → 🟢
- BehaviorAnalytics Output → 🟡 Review

*"UEBA itself is running. All the data feeding it is healthy. But we flagged the output volume as Review because it dropped slightly compared to last month. We are investigating whether a configuration change affected the anomaly sensitivity settings. It is not broken — it is being watched. That is the difference."*

You click through the Threat Feed card. API key valid. 187 days until expiry. 14,847 indicators loaded. Last updated four hours ago.

*"Your threat feed is pulling nearly fifteen thousand known malicious indicators — IPs, domains, file hashes — updated every few hours. Every log we collect gets checked against this list automatically."*

---

**Tab 6 — Integrations.**

One row amber. A VirusTotal API key expiring in 28 days.

*"This is on our radar. We will rotate it before it expires. If it expired without being replaced your incident enrichment playbook would stop working — every new ticket would be missing the malware reputation check. We catch these before they become problems."*

The client's IT director glances across the table at the CISO. Something passes between them.

---

**You navigate back to Tab 1.**

*"Three months ago we did not have this level of visibility into your environment. Today every data source is tracked at the log source level — not just the table level. Every capability is monitored with its full dependency chain. Every credential has an expiry date we are watching. Every detection has a silent monitor behind it. When something breaks — we know before you do."*

*"The XDR connector situation you experienced — that cannot happen now. The connector is configured, validated, and monitored. If it ever stopped syncing a detection would fire within the hour and our team would be on it before you ever saw an impact."*

You pause.

*"This is the foundation. From here we start building upward — more data sources, more detections, more automation. Every quarter when we sit in this meeting you will see how far we have come."*

The client who doubted you three months ago sits back and says:

*"This is exactly what we needed."*

**The ball is in their court now.**

---

## The Possibilities — What Comes Next

Once the foundation is solid the real possibilities open up. Here is a taste of what this system enables:

**Detection coverage gaps become conversations, not embarrassments.** The workbook shows 72 of 83 detections active. The other 11 are not broken — they are waiting for data. Each one links to a detection library entry that explains what it catches, what data it needs, and how to onboard that data. A gap is not a failure. It is a recommendation with a path forward.

**The MITRE heatmap tells the maturity story.** Six tactics covered at onboarding. Twelve tactics covered six months later. A visual that shows a client their security program growing — not just running.

**Watchlist health keeps the client honest.** A monthly report goes to their IT team — here are the watchlist entries that have not been verified in 90 days, here are machines sending logs that are not in any watchlist, here are decommissioned assets still listed as active. They verify. You process the updates. The data stays accurate.

**API key expiry never catches anyone off guard.** 30 days out — Teams message. Key rotated. Done. No capability silently fails because someone forgot a renewal date.

**And then — the trick up our sleeve.**

---

## We Wrangled the Beast

The client adds a new server to their environment. Nobody tells us. Why would they — it is their environment. But three days later something interesting happens.

The weekly snapshot Logic App runs on schedule. It queries the workspace, captures the state of every table and every log source, and writes the results to a custom table called `DataSourceAudit_CL`. It compares this week's snapshot to last week's.

A new entry appears in `CommonSecurityLog`. A log source that was not there before. The analytics rule fires.

An incident is created in Sentinel.

A Teams message lands in our SOC channel at 7:43am:

> 🟠 **NEW LOG SOURCE DETECTED — Client X**
> Table: CommonSecurityLog
> Log Source: Fortinet — New Branch Firewall
> First Seen: 2026-04-08 07:41 UTC
> Status: Flag — not in watchlist
> Action: Investigate and classify

We look into it. New branch office. New firewall. IT spun it up last week. Makes sense. We add it to the watchlist, assign it a category, map it to the relevant detections, create a silent detection for it, and close the incident.

At the next client review we open the Insights tab. There it is — the new source appearing on the volume trend chart, the exact date it came online, the incident that caught it, the resolution.

*"You added a firewall last month. We caught it within 72 hours, classified it, and it is now fully monitored. You did not have to tell us — the system told us."*

---

## Baseline Achieved. Sleep Well.

This is what it means to have a real foundation.

Not a collection of connectors that somebody enabled during onboarding and nobody has verified since. Not a dashboard that shows green because nothing is explicitly broken. A documented, verified, signed-off, continuously monitored environment where every data source is known, every capability is healthy, every detection has a safety net, and every change gets noticed.

The beast is wrangled.

And now that we have solid ground to stand on — every improvement we make is additive. Every new detection, every new data source, every new capability gets built on top of something that actually works. We are not patching gaps we did not know existed. We are growing a program we fully understand.

Baseline is not the ceiling. Baseline is the floor.

**Protect the eggs.**

---

## Repository Structure

```
sentinel-mssp-playbook/
├── README.md                        ← You are here
├── ARCHITECTURE.md                  ← How everything connects
├── PROGRAM-MAP.md                   ← Full scope, status, and future roadmap
│
├── 01-baseline/                     ← Layer 1 — Configuration standard
├── 02-data-sources/                 ← Layer 2 — Audit system and workbook
├── 03-detection-library/            ← Layer 3 — Detection catalog
├── 04-capabilities/                 ← Capability dependency documentation
├── 05-xdr-migration/                ← Defender XDR migration guidance
├── 06-watchlist-management/         ← Watchlist health and verification
├── 07-integrations/                 ← API key and credential management
├── 08-silent-detections/            ← Silent detection standards
├── 09-audit-deck/                   ← Layer 4 — Client review system
├── 10-automation-pipeline/          ← Snapshot, alerts, Teams notifications
├── 11-access-and-permissions/       ← Lighthouse scope and Graph API
└── resources/                       ← Taxonomy, KQL snippets, glossary
```

---

## Status

Active development. Layers 1 and 2 are complete. Detection library, XDR migration, and audit deck phases in progress. See `PROGRAM-MAP.md` for full status of every file and the complete future roadmap.

---

*Built for Microsoft Sentinel engineers who want to operate at a higher standard. Contributions and ideas welcome.*