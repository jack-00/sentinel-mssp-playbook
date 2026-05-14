Subject: MSS - Microsoft Sentinel AuditDeck — Phase 1 Kickoff

Hello [Client Name],

We are excited to introduce AuditDeck — a new audit and monitoring program being
built directly inside your Microsoft Sentinel environment. The goal is to establish
a clear and consistent baseline of your security monitoring environment — covering
data sources, detection coverage, ingestion trends, agent health, and much more.

We have deployed the first version of the AuditDeck workbook into your environment
today. You can find it in the Workbooks section under MSS - Microsoft Sentinel Audit.
This initial version displays all tables that have ingested logs over the past 30
days, the most recent log received for each, and days since last log. This is just
the beginning — as we continue building AuditDeck it will grow to cover data sources,
detection coverage, ingestion trends, agent health, and much more.

We have three action items for you outlined below. Tasks 1 and 2 are quick — please
prioritize those first. Task 3 is a more detailed intake process that can follow.
A detailed breakdown of each task is listed below.
Everything you need is in the attached zip file — start with README.docx.

---

WHAT IS ATTACHED

    README.docx
    Start here. A complete guide to everything in this package and what
    we need from you.

    01-ClientInformation-Guide.docx
    Field by field guidance for completing the intake spreadsheet.

    02-ClientInformation.xlsx
    The intake spreadsheet. Tab 1 is your template to fill in. Tab 2 is
    a completed example for reference.

    03-EnvironmentProfile-Guide.docx
    Guidance for completing the environment profile document.

    04-EnvironmentProfile-Template.docx
    The environment profile document for you to fill in.

    05-DefenderXDR-SetupGuide.docx
    Information about the Microsoft Defender XDR Data Connector and
    what we need to know about your current configuration.

---

TASK 1 — Review the AuditDeck Workbook

Navigate to Microsoft Sentinel and open the Workbooks section — look for
MSS - Microsoft Sentinel Audit. Review the table of data sources and confirm
that the sources listed align with what you expect to be logging and that nothing
looks unfamiliar or unexpectedly missing. Please reply in this ticket with your
confirmation or any questions.

---

TASK 2 — Confirm Defender XDR Data Connector Status

The Microsoft Defender XDR Data Connector is one of the most critical connections
between your Microsoft Defender security products and Microsoft Sentinel. It ensures
that security events from your endpoints, identity, email, and cloud applications
are flowing into Sentinel where our team can monitor and detect threats.

Please reply in this ticket with one of the following:

    It is already enabled — Reference the attached 05-DefenderXDR-SetupGuide.docx
    and let us know which event categories are currently enabled.

    It is not enabled or I am not sure — Let us know and we will open a separate
    ticket to work with you on configuration.

    I have questions — Reply in this ticket and we will help.

---

TASK 3 — Complete and Return the Intake Forms

Please work through the two intake documents in the attached zip.

    02-ClientInformation.xlsx
    Fill in the Value column on Tab 1 only. Tab 2 is a completed example
    for reference — do not edit it. Use '01-ClientInformation-Guide.docx' for field by field guidance.

    04-EnvironmentProfile-Template.docx
    Fill in each section describing your environment. Use
    03-EnvironmentProfile-Guide.docx for guidance. Leave anything blank
    that does not apply or that you are unsure about.

When ready to return files please reply with a contact email address and we
will send you a secure link to upload your completed documents via kiteworks.

---

Thank you for your partnership on this — we look forward to getting AuditDeck
fully underway for your environment.
