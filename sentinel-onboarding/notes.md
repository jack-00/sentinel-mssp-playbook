Phase 0 — Access & Discovery (days 1–7): Lighthouse delegation executed by their team, environment intake form completed, run your audit workbook process against their workspace to inventory tables/sources/existing content

Phase 1 — Assess & Baseline (days 8–15): Map their ingesting sources to your detection catalog, identify gaps against your (soon-to-exist) baseline checklist, review their existing analytics rules for keep/kill/replace

Phase 2 — Deploy & Integrate (days 16–24): Push your managed detections, deploy the triage logic app + automation rules, wire into Kong/StreamSets → Elastic/ServiceNow, watchlists built

Phase 3 — Validate & Cutover (days 25–30): End-to-end incident test fires, silent log detections live, client portal access confirmed, go-live and hypercare

---

establish trust ("here's how my practice works")

take history ("tell me about the patient")

set expectations ("here's your care plan and your next appointment")

---

Intros & relationship map (10 min): who's who, and explicitly: who is our day-to-day technical contact, who approves access, who owns the ticket flow on their side
Our service model

(10 min): the elevator version of how you operate — Lighthouse access, managed detections, automated triage to ServiceNow, client portal. Short, confident, no internals

Their environment story (20 min): the discovery questions — this is the meat, and topic 3 on our list gives you the question bank

The plan (10 min): walk the four phases, give the 30-45 day range, name what's yours vs. theirs

Next actions & cadence (10 min): their homework (Lighthouse delegation, intake form, named contacts), agree on a weekly sync, set the next meeting date before you hang up

---

Prepare or send the intake form with the meeting invite follow-up today or first thing tomorrow if you can

Second, end the meeting by scheduling the next one. Momentum stays yours.

---

Relationship map (5 min): who's our technical counterpart, who approves access
The plan (10 min): four phases, 30-45 days, what's ours vs. theirs
Next actions & cadence (15 min): their homework, weekly sync, next meeting booked

---

# Sentinel Client Onboarding — Planning Doc
**Status:** Pre-kickoff | **Meeting:** Tomorrow (proxy client only — end client not present)
**Timeline commitment:** 30 days (45-day runway)

---

## Situation Summary

- Onboarding an **existing, healthy Sentinel workspace** (adopt-and-verify, not greenfield)
- Three-party structure: **Us (MSSP platform team, 3 techs) → Proxy client (existing client, brings us this deal) → End client**
- Pro-services is gone — platform team owns onboarding end-to-end for the first time
- No internal onboarding standards/baselines yet — this project builds them as deliverables
- Proxy client is evaluating us for future client referrals — execution here protects the pipeline

## Key Technical Facts (verified)

- **Azure Lighthouse delegations cannot be re-delegated.** Proxy's access to end client's
  tenant cannot pass through to us. End client's tenant admin must execute a delegation
  **directly to our tenant.**
- **Lighthouse alone cannot deploy data connectors** in a managed workspace — connector-side
  actions require GDAP or hands in the client's tenant. Existing workspace means most
  connectors already exist; client-side hands needed only for changes/fixes.
- **Sentinel Azure portal retirement: March 31, 2027** (extended from July 2026).
  Lighthouse remains the supported MSSP pattern for now. Our Defender portal process is
  on our roadmap — not a blocker for this onboarding.
- Workspaces newly onboarded after July 1, 2025 may be **auto-onboarded to the Defender
  portal** — need to confirm which portal end client's team uses day-to-day.

---

## 30-Day Timeline Skeleton (Adopt-and-Verify Model)

| Phase | Days | Focus |
|-------|------|-------|
| **0 — Access & Discovery** | 1–7 | Lighthouse delegation executed by end client, intake form completed, run audit workbook against workspace (tables/sources/content inventory) |
| **1 — Assess & Baseline** | 8–15 | Map ingesting sources to detection catalog, gap analysis vs. baseline checklist, keep/kill/replace review of existing analytics rules |
| **2 — Deploy & Integrate** | 16–24 | Push managed detections, deploy triage logic app + automation rules, wire Kong/StreamSets → Elastic/ServiceNow, build watchlists |
| **3 — Validate & Cutover** | 25–30 | End-to-end incident test fires, silent-log detections live, client portal access confirmed, go-live + hypercare |

**Critical path:** Lighthouse delegation. Everything starts when it executes.
**Reality check:** Industry norm is 2–3 days actual technical effort; calendar time is
mostly waiting on client approvals. 30 days is comfortable; 45 covers proxy-relay delays.

---

## Meeting Agenda — Proxy-Only Edition (1 hour)

> **Reframe:** This is an **alignment meeting**, not a discovery meeting. The environment
> answers aren't in the room. Settle the operating model, the plan, and the machinery to
> get the end client into a technical session next week.

1. **Intros (5 min)** — one-liner: "We're the platform team managing Sentinel day-to-day
   for you; glad to extend that."
2. **Block Zero — Relationship & Operating Model (20 min)** — the headliner. See below.
3. **Service Model & The Plan (15 min)** — pitch (Lighthouse, curated detections,
   automated triage → ServiceNow, client portal), then walk the 4 phases. Teach the
   Lighthouse no-re-delegation fact calmly, doc in hand.
4. **About the Client (10 min)** — light discovery only: size, industry/compliance, why
   now, what they've been promised, who their technical people are.
5. **Next Actions & Cadence (10 min)** — homework close (below), weekly sync agreed,
   technical discovery session target booked, recap email promised in 24h.

### 30-Minute Fallback
Intros (3) → Block Zero (10) → The Plan (7) → Next Actions (10).
Discovery travels by document; relationship clarity and momentum cannot.

---

## Block Zero — Relationship & Operating Model

**Opener:** *"Before the technical side — walk us through how this came together and how
you picture the day-to-day working with us in the mix."*

**Play-it-through scenarios (all five must get answered):**

- **Access chain:** "When we send the Lighthouse onboarding doc — who receives it, and
  who in the end client's tenant executes it?"
- **Communication flow:** "Tuesday, we spot a broken connector needing tenant-side hands.
  Who do we contact, and what's realistic turnaround?" *(calibrates Phase 0 padding)*
- **Ticket & incident flow:** "Incident fires at 2am, lands in ServiceNow. Whose T1 works
  it? Who's in the client portal?"
- **Authority & money:** "We recommend a connector that adds ingest cost. Who says yes?"
- **Scope boundary:** "What does the end client believe you're doing vs. what we're doing?"

**Proxy-only bonus questions (candid, end client not listening):**

- "What does the client expect this to feel like — have they been through a SOC
  onboarding before?"
- "Anything we should know about how they operate — responsive? Slow approvals?
  Technical depth on their side?"

**Watch for:** the glance — proxy contacts looking at each other before answering means
they haven't worked it out. Response: "No rush — that goes in the recap as an open item
for you two." Their ambiguity becomes a tracked action item, not our future problem.

**Exit criteria:** We know who executes Lighthouse in the end client's tenant, and who
our day-to-day technical counterpart is.

---

## Next Actions Handoff (Two-Column Close)

### Their homework
- [ ] Intro us to end client's technical contact
- [ ] Relay Lighthouse doc + intake form + discovery questionnaire to end client
- [ ] Book technical discovery session **with end client's Sentinel admin present**
      (target: within 5 business days)
- [ ] Named contacts: technical counterpart, access approver, escalation
- [ ] Confirm end client's Microsoft support plan is above Basic

### Our homework (say these out loud)
- [ ] Send Lighthouse documentation + intake form by end of day
- [ ] Run environment audit (workbook process) within 5 business days of access
- [ ] Deliver findings + gap report at end of Phase 1 (first "wow" deliverable)
- [ ] Recap email with owners + dates within 24 hours
      *(Recap restates timeline in writing — it becomes the contract and the
      scope-creep shield: "per our kickoff recap, that's a Phase 2 item.")*

---

## Discovery Question Bank

> Deployment: asked live at next week's technical session and/or sent as questionnaire.
> Deep-dive "why" completed for Environment & Access; remaining categories pending.

### Environment & Access

**Q1. How many Entra tenants and Azure subscriptions are in scope? Which
subscription/resource group hosts the Sentinel workspace?**
- *Plumbing:* Tenant = identity universe → Subscriptions = billing containers →
  Resource groups = folders → Log Analytics workspace = the log database Sentinel
  bolts onto.
- *Why:* (1) Lighthouse is scoped to subscription/RG — wrong scope = access requests in
  week two. (2) Many built-in connectors only work within their own tenant — a second
  tenant may mean logs that **cannot** flow in via canned connectors. (3) The hosting
  subscription owns the ingest bill.
- *Use case:* "Two tenants — corporate and a subsidiary" → immediate follow-up: "Does
  subsidiary data flow into this workspace today, and how?" Instant visibility-gap find.

**Q2. Who are the named owners: workspace owner, tenant admin contact, and who can
execute the Lighthouse delegation?**
- *Why:* Delegation must be deployed by someone in **their** tenant with Owner rights —
  we cannot do it for them. Some later needs (connector enablement, tenant diagnostic
  settings) require Global/Security Admin roles we never hold via Lighthouse. In the
  three-party structure, tenant-side turnaround = proxy delay + end-client delay.
- *Use case:* Recap email assigns "execute Lighthouse delegation" to a **named person
  with a date**. Tasks assigned to companies never get done.

**Q3. What's your Microsoft support plan level?**
- *Why:* Basic = forums and prayer. Platform-side breakage (silent connector failure,
  ingestion lag) is only fixable via Microsoft ticket. We can file support requests
  against a delegated subscription only if their plan supports it. Silent log source
  with no support path = detection gap with no repair path.
- *Use case:* Firewall logs stop ingesting (Microsoft-side). Standard+ = sev-B ticket
  same day. Basic = weeks of blindness that lands on our SOC's reputation.

**Q4. Any compliance or data residency constraints?**
- *Why:* Workspace region is fixed — can't move it. Retention beyond included 90 days
  costs real money and may be legally mandated. Compliance frameworks may constrain
  *our* team's access. Existing workspace = inherited decisions; know which were
  compliance-driven before "optimizing" them.
- *Use case:* Suggesting a retention cut from 365→90 days to save money, then hearing
  "PCI requires one year" = looking like we skipped discovery.

**Q5. Is the workspace onboarded to the Defender portal, or Azure portal only?**
- *Why:* Our toolchain is Azure-portal/Lighthouse-based; Defender portal process is
  roadmap. Post-July-2025 onboarded workspaces may already be in the Defender portal.
  Defender XDR correlation can merge alerts into incidents differently — relevant to
  incident-triggered automation. Not a blocker; a surprise-preventer.
- *Use case:* "Our admin uses the Defender portal" → "We manage via Azure/Lighthouse
  today with a Defender transition on our roadmap ahead of the 2027 retirement —
  nothing about that affects your onboarding." Gotcha converted to roadmap evidence.

### Data Sources & Ingest *(deep-dive pending — priority next)*
- Connectors enabled: actively ingesting vs. connected-but-silent?
- Daily ingest volume (GB/day), monthly spend, commitment tier?
- Canned vs. custom connectors? For custom: who built, who maintains, docs? *(8-hour rule)*
- Agents/forwarders: AMA + DCRs, legacy MMA remnants, syslog/CEF collectors — who owns?
- Critical sources — the "goes silent at 2am, wake someone" list *(seeds silent-log watchlist)*
- Planned source additions/retirements next 90 days?
- Retention setup: analytics vs. basic/auxiliary tiers, archive?

### Existing Content (keep/kill/replace) *(deep-dive pending)*
- How many analytics rules live? Built by whom?
- Existing automation: playbooks, automation rules, logic apps? Any that *act*
  (isolate/disable) vs. notify?
- Watchlists, workbooks, hunting queries? UEBA/anomaly rules enabled?
- Anything currently paging/emailing a human? *(don't orphan alert paths at cutover)*

### Operations & Ticketing *(deep-dive pending)*
- Who responds to incidents today? Walk an incident fire-to-close.
- SLA expectations, escalation paths, after-hours coverage?
- Does their team retain hands-on workspace access, or full handoff?

### Business Context *(deep-dive pending)*
- Tenant-side actions: through proxy or direct? Expected turnaround?
- Who approves spend-affecting changes?
- What outcomes matter most in first 90 days? Why now?

---

## Gap Capture — Internal Deliverables Built During This Onboarding

| Gap | Deliverable | Notes |
|-----|-------------|-------|
| No onboarding methodology | This doc → onboarding runbook | Pro-services replacement |
| No Sentinel baseline | Baseline checklist (auditing on, UEBA, audit logs, etc.) | **Fable 5 task** |
| Watchlist sprawl | Unified watchlist design (type/threshold as fields; HA edge cases separate) | **Fable 5 task** |
| Inconsistent silent-log detection | Standard silent-log flow driven by critical-source watchlist | **Fable 5 task** |
| Intake ↔ audit tooling not integrated | Fold audit workbook (tables/sources/ingest/health/detections tabs + Environment tab) into onboarding flow | **Fable 5 task** |
| No Defender portal process | Transition plan ahead of Mar 31, 2027 | Roadmap item, not this project |

---

## Open Items
- [ ] Confirm meeting length (30 vs 60 min) and final attendee list
- [ ] Does end client's workspace carry its own analytics rules/automation to inherit,
      or mostly raw ingest? *(shapes Phase 1 weight — ask proxy tomorrow)*
- [ ] Which portal does end client's team work in day-to-day?
- [ ] Deep-dive "why" treatment: remaining 4 question categories