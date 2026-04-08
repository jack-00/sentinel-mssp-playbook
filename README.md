# sentinel-mssp-playbook

A structured operational framework for managing Microsoft Sentinel and Defender XDR environments at scale. Built for security engineers and MSSP teams who want a repeatable, documented standard for standing up, auditing, and continuously improving client Sentinel deployments.

## What This Is

This playbook is organized into layers that build on each other. Each layer produces a real, usable artifact — not just documentation for its own sake.

## Structure

```
sentinel-mssp-playbook/
├── 01-baseline/          # Universal Sentinel configuration checklist
├── 02-data-sources/      # Data source audit framework and runbook
├── 03-detection-library/ # Detection catalog with MITRE mapping
├── 04-xdr-migration/     # Defender XDR migration guidance
└── 05-audit-deck/        # Client review templates and workbook design
```

## The Four Layers

**Layer 1 — Baseline**
A non-negotiable checklist of every setting, connector, and capability that must be configured in every Sentinel workspace. Covers connectors, audit log enablement, UEBA, analytics rules, watchlists, automation, and health monitoring.

**Layer 2 — Data Source Audit**
A repeatable process for documenting and validating every data source in a Sentinel workspace. Organized by table rather than connector — because tables tell the truth. Includes KQL queries, field definitions, and verification guidance.

**Layer 3 — Detection Library**
A structured catalog of detections mapped to MITRE ATT&CK tactics and techniques. Each entry documents what the detection does, what data it requires, and why it matters — in terms both engineers and stakeholders can understand.

**Layer 4 — Audit Deck**
A repeatable client review system built on living workbooks. Coverage gaps are tracked over time and presented as improvement opportunities with clear recommendations.

## Key Principles

- Audit by table, not by connector — connector status is misleading, table data is ground truth
- Every gap is a documented decision — nothing is simply forgotten
- The workbook does the work — client reviews require narration, not research
- MITRE ATT&CK is the measurement framework — coverage is tracked and visualized consistently

## Status

Active development. Layers 1 and 2 are in progress.

---

*Maintained by a Microsoft Sentinel engineering team. Contributions and feedback welcome.*