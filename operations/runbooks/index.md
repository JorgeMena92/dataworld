---
title: Runbooks
description: Step-by-step procedures for known and recurring data platform issues.
tags: [operations, runbooks]
---

# Runbooks

A runbook is a documented procedure for a specific, known scenario. Unlike the [Layered Triage Framework](../support/layered-fw.md), which guides you through an unknown incident, a runbook applies when you already know what the issue is and need the steps to resolve it quickly and correctly.

Runbooks reduce mean time to resolution (MTTR) for recurring issues, enable less experienced engineers to handle complex scenarios safely, and ensure consistency across the team.

---

## How to Use a Runbook

1. Confirm the symptom matches the runbook's described scenario before following the steps
2. Follow steps in order — do not skip steps, even if they seem unnecessary
3. If the steps do not resolve the issue, stop and escalate rather than improvising
4. Log which runbook was followed in the incident ticket timeline

---

## How to Write a Runbook

Every runbook follows the same structure:

```
Title: [Short, searchable problem description]
Applies to: [Platform component and specific scenario]
Severity: [Typical severity of this scenario]
Last validated: [Date it was last tested or used successfully]
```

**Symptom** — what the engineer or user observes

**Pre-conditions** — confirm these are true before following this runbook

**Steps** — numbered, specific, with exact commands, paths, or UI navigation

**Verification** — how to confirm the issue is resolved

**Rollback** — how to undo the steps if something goes wrong

**Escalation** — when to stop and escalate instead of continuing

---

## Runbook Index

!!! note "This section is growing"
    Runbooks are added as recurring issues are identified and documented. Each resolved incident is an opportunity to add one — see [Incident Tracking](../support/incident-tracking.md#knowledge-base-contribution) for guidance on when and how to write them.

*No runbooks yet. Add the first one after your next resolved recurring incident.*
