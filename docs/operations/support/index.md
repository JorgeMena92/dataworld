---
title: Support Framework
description: A structured approach to data platform support — incident triage, daily operations, documentation, and communication.
---

# Support Framework

A practical framework for supporting a production data platform. Designed for roles that sit at the intersection of data engineering and platform operations — anyone responsible for keeping pipelines running, data fresh, and stakeholders informed.

---

## Pages in This Section

| Page | Purpose |
|------|---------|
| [Layered Triage](layered-fw.md) | Work from the presentation layer inward — a structured method for isolating the root cause of any platform incident |
| [Incident Management](incident-management.md) | Severity classification, SLA tiers, response workflows, and escalation paths |
| [Daily Operations](daily-operations.md) | Shift start checklist, platform health checks, and handover template |
| [Incident Tracking](incident-tracking.md) | How to write an incident record, post-mortem structure, and knowledge base contribution |
| [Communication Standards](communication-standards.md) | How to communicate during outages, in handovers, and with stakeholders |
| [Runbooks](../runbooks/index.md) | Step-by-step procedures for known and recurring platform issues |

---

## The Incident Flow

When something goes wrong, this is the end-to-end path from detection to closure. Each step links to the page that covers it in detail.

<div class="incident-flow">

  <div class="flow-step flow-step--alert">
    <div class="flow-step__icon">⚠️</div>
    <div class="flow-step__body">
      <div class="flow-step__title">Something is wrong</div>
      <div class="flow-step__sub">An incident has been detected</div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">1</div>
    <div class="flow-step__body">
      <div class="flow-step__title">🎫 Log it</div>
      <div class="flow-step__sub">Create ticket · Assign severity · <a href="incident-management.md">Incident Management</a></div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">2</div>
    <div class="flow-step__body">
      <div class="flow-step__title">🔎 Triage it</div>
      <div class="flow-step__sub">Confirm scope and impact · <a href="incident-management.md">Incident Management</a></div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">3</div>
    <div class="flow-step__body">
      <div class="flow-step__title">🗂️ Investigate it</div>
      <div class="flow-step__sub">Work through the layers · <a href="layered-fw.md">Layered Triage</a></div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">4</div>
    <div class="flow-step__body">
      <div class="flow-step__title">📋 Check for a known fix</div>
      <div class="flow-step__sub">Is there a procedure for this? · <a href="runbooks/index.md">Runbooks</a></div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">5</div>
    <div class="flow-step__body">
      <div class="flow-step__title">✅ Resolve it</div>
      <div class="flow-step__sub">Fix · Workaround · Rollback · <a href="incident-management.md">Incident Management</a></div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">6</div>
    <div class="flow-step__body">
      <div class="flow-step__title">📣 Communicate</div>
      <div class="flow-step__sub">Update stakeholders · <a href="communication-standards.md">Communication Standards</a></div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">7</div>
    <div class="flow-step__body">
      <div class="flow-step__title">📝 Document it</div>
      <div class="flow-step__sub">Incident record · Post-mortem · <a href="incident-tracking.md">Incident Tracking</a></div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">8</div>
    <div class="flow-step__body">
      <div class="flow-step__title">🔄 Hand over</div>
      <div class="flow-step__sub">Update the shift note · <a href="daily-operations.md">Daily Operations</a></div>
    </div>
  </div>

</div>