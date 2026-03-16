---
title: Support Framework
description: A structured approach to data platform support — incident triage, daily operations, documentation, and communication.
---

# Support Framework

A practical framework for supporting a production data platform built on Power BI, Azure Databricks, Azure Data Factory, and Microsoft Fabric.

The framework is organized around the four core responsibilities of a data platform support role: **investigating incidents**, **running daily operations**, **documenting and tracking work**, and **communicating clearly with stakeholders and teams**.

---

## Pages in This Section

| Page | Purpose |
|------|---------|
| [Layered Triage](layered-fw.md) | Work from the presentation layer inward — a structured method for isolating the root cause of any platform incident |
| [Incident Management](incident-management.md) | Severity classification, SLA tiers, response workflows, and escalation paths |
| [Daily Operations](daily-operations.md) | Shift start checklist, platform health checks, and handover template |
| [Incident Tracking](incident-tracking.md) | How to write an incident record, post-mortem structure, and knowledge base contribution |
| [Communication Standards](communication-standards.md) | How to communicate during outages, in handovers, and with stakeholders |

---

## How the Framework Fits Together
 
<div class="diagram-light" markdown>
```mermaid
flowchart TD
    INCIDENT(["🔍 Incident Detected"]):::start
 
    L1["📊 Layered Triage"]:::layer
    L2["🚨 Incident Management"]:::layer
    L3["📝 Incident Tracking"]:::layer
    L4["📣 Communication Standards"]:::layer
 
    D1["Work from the presentation layer inward.\nIsolate root cause before going deeper."]:::desc
    D2["Assign severity · Start SLA clock\nEscalate if needed"]:::desc
    D3["Write the incident record · Document timeline\nSchedule post-mortem"]:::desc
    D4["Update stakeholders · Complete handover\nContribute to knowledge base"]:::desc
 
    INCIDENT --> L1 --> L2 --> L3 --> L4
 
    L1 -.- D1
    L2 -.- D2
    L3 -.- D3
    L4 -.- D4
 
    classDef start fill:#e0e0e0,color:#1c1c1e,stroke:#e0e0e0,stroke-width:2px
    classDef layer fill:#EAF0FB,color:#1F3864,stroke:#2E75B6,stroke-width:1.5px
    classDef desc  fill:#ffffff,color:#3a3a3c,stroke:#e0e0e0,stroke-width:1px
 
    linkStyle 0,1,2,3 stroke:#7a7a7a,stroke-width:1.5px
    linkStyle 4,5,6,7 stroke:#c0c0c0,stroke-width:1px,stroke-dasharray:4
```
</div>
 
<div class="diagram-dark" markdown>
```mermaid
flowchart TD
    INCIDENT(["🔍 Incident Detected"]):::start
 
    L1["📊 Layered Triage"]:::layer
    L2["🚨 Incident Management"]:::layer
    L3["📝 Incident Tracking"]:::layer
    L4["📣 Communication Standards"]:::layer
 
    D1["Work from the presentation layer inward.\nIsolate root cause before going deeper."]:::desc
    D2["Assign severity · Start SLA clock\nEscalate if needed"]:::desc
    D3["Write the incident record · Document timeline\nSchedule post-mortem"]:::desc
    D4["Update stakeholders · Complete handover\nContribute to knowledge base"]:::desc
 
    INCIDENT --> L1 --> L2 --> L3 --> L4
 
    L1 -.- D1
    L2 -.- D2
    L3 -.- D3
    L4 -.- D4
 
    classDef start fill:#2a2a2a,color:#f0f0f0,stroke:#3a3a3a,stroke-width:2px
    classDef layer fill:#1e2a38,color:#a0b8d0,stroke:#2a3f55,stroke-width:1.5px
    classDef desc  fill:#1a1a1a,color:#a0b8d0,stroke:#2a2a2a,stroke-width:1px
 
    linkStyle 0,1,2,3 stroke:#606060,stroke-width:1.5px
    linkStyle 4,5,6,7 stroke:#404040,stroke-width:1px,stroke-dasharray:4
```
</div>
 
<style>
  [data-md-color-scheme="default"] .diagram-dark  { display: none; }
  [data-md-color-scheme="slate"]   .diagram-light { display: none; }
</style>
 
---

!!! tip "Start with Layered Triage"
    When an incident comes in, the [Layered Triage](layered-fw.md) page is your first reference. It tells you where to look and in what order — from the Power BI report surface all the way down to infrastructure and security.