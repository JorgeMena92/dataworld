---
title: Incident Management
description: Severity classification, SLA tiers, response workflows, and escalation paths for data platform incidents.
tags: [operations, support, incident-management, sla]
---

# Incident Management

Incident management is the process of detecting, classifying, responding to, and resolving unplanned disruptions to a data platform. A consistent approach ensures that every incident is handled with the right urgency, the right people are involved at the right time, and nothing falls through the gaps.

---

## Severity Classification

Severity determines how fast you respond, who you involve, and how you communicate. Assign severity at the moment of detection — it can be revised as the situation becomes clearer.

| Severity | Label | Definition | Examples |
|----------|-------|------------|---------|
| **SEV-1** | Critical | Complete loss of a business-critical capability. Data is unavailable, incorrect, or actively misleading decision-makers. | BI reports unavailable for all users · Production pipeline fully stopped · Data loss confirmed |
| **SEV-2** | High | Significant degradation of a key capability. Core functionality is impaired but a workaround may exist. | Dashboard shows stale data · Key pipeline delayed by hours · Scheduled refresh failing for a major dataset |
| **SEV-3** | Medium | Partial impact on a non-critical capability or a low-visibility issue. Operations can continue. | One report visual incorrect · Non-critical pipeline delayed · Intermittent refresh failures |
| **SEV-4** | Low | Minor issue with no immediate operational impact. Informational or cosmetic. | Formatting issue in a report · Slow query with no user complaint · Documentation gap |

!!! warning "When in doubt, go higher"
    It is always better to open a SEV-2 and downgrade it than to open a SEV-3 and miss an SLA. Severity can be revised — missed response windows cannot.

---

## SLA Tiers

SLA (Service Level Agreement) targets define the maximum time allowed for each phase of the response. These are **guidelines** — actual SLA commitments depend on your organisation's contracts and policies.

| Severity | Initial Response | Update Frequency | Resolution Target |
|----------|-----------------|-----------------|------------------|
| SEV-1 | 15 minutes | Every 30 minutes | 4 hours |
| SEV-2 | 1 hour | Every 2 hours | 8 hours |
| SEV-3 | 4 hours | Once per business day | 3 business days |
| SEV-4 | 1 business day | As needed | Next sprint / backlog |

- **Initial Response** — acknowledge the ticket and confirm it is being investigated
- **Update Frequency** — post a progress update to the ticket and notify stakeholders
- **Resolution Target** — the incident is resolved or a permanent workaround is in place

!!! note "SLA clock"
    For most organisations, the SLA clock starts when the ticket is created or the alert fires — not when the engineer picks it up. Make sure you acknowledge tickets promptly even if investigation takes longer.

---

## Response Workflow

Every incident follows the same five-step lifecycle regardless of severity.

<div style="display:flex;align-items:center;justify-content:center;gap:0;margin:1.5rem 0;flex-wrap:wrap;">
  <div style="background:#EAF0FB;color:#1F3864;border:1.5px solid #2E75B6;border-radius:8px;padding:0.6rem 1.2rem;font-weight:600;font-size:0.8rem;text-align:center;">🔍 Detect</div>
  <div style="color:#7a7a7a;font-size:1.2rem;padding:0 0.25rem;">→</div>
  <div style="background:#EAF0FB;color:#1F3864;border:1.5px solid #2E75B6;border-radius:8px;padding:0.6rem 1.2rem;font-weight:600;font-size:0.8rem;text-align:center;">📋 Triage</div>
  <div style="color:#7a7a7a;font-size:1.2rem;padding:0 0.25rem;">→</div>
  <div style="background:#EAF0FB;color:#1F3864;border:1.5px solid #2E75B6;border-radius:8px;padding:0.6rem 1.2rem;font-weight:600;font-size:0.8rem;text-align:center;">🔎 Investigate</div>
  <div style="color:#7a7a7a;font-size:1.2rem;padding:0 0.25rem;">→</div>
  <div style="background:#EAF0FB;color:#1F3864;border:1.5px solid #2E75B6;border-radius:8px;padding:0.6rem 1.2rem;font-weight:600;font-size:0.8rem;text-align:center;">✅ Resolve</div>
  <div style="color:#7a7a7a;font-size:1.2rem;padding:0 0.25rem;">→</div>
  <div style="background:#EAF0FB;color:#1F3864;border:1.5px solid #2E75B6;border-radius:8px;padding:0.6rem 1.2rem;font-weight:600;font-size:0.8rem;text-align:center;">🔒 Close</div>
</div>

### 1 · Detect

An incident is detected through one of three channels:

- **User report** — a stakeholder raises an issue via ticket, email, or chat
- **Automated alert** — a monitoring rule fires (pipeline failure, refresh error, threshold breach)
- **Proactive check** — identified during the daily health check (see [Daily Operations](daily-operations.md))

As soon as an incident is detected, **create a ticket** and assign an initial severity. Do not investigate first — log first.

### 2 · Triage

Triage means confirming the scope and impact before diving into root cause analysis.

- Confirm the issue is real and reproducible
- Identify who is affected and how many users or processes are impacted
- Assign or confirm severity
- Notify the relevant team or on-call engineer if the issue warrants escalation
- Send an initial acknowledgement to the reporter

Use the [Layered Triage Framework](layered-fw.md) to structure your investigation.

### 3 · Investigate

Work through the platform layers systematically. Document your findings as you go — not after. This serves two purposes: it keeps stakeholders informed, and it builds the post-mortem record automatically.

- Start at the presentation layer and work inward
- Eliminate each layer before moving to the next
- Note timestamps, error messages, log excerpts, and configuration states
- If you reach a layer outside your access or expertise, escalate — do not guess

### 4 · Resolve

Resolution means the user impact has been eliminated. This can be achieved through:

- **Fix** — the root cause is corrected and the platform returns to normal operation
- **Workaround** — a temporary measure that restores user access while the permanent fix is prepared
- **Rollback** — a recent change is reverted to restore the previous stable state

Document the resolution action and the timestamp. Update the ticket status to **Resolved**.

### 5 · Close

An incident is closed when:

- The fix or workaround is confirmed stable
- The user or stakeholder has confirmed the issue is no longer present
- The ticket is updated with a complete timeline and resolution summary
- A post-mortem is scheduled (SEV-1 and SEV-2) or a brief root cause note is added (SEV-3)

---

## Escalation Paths

Escalate when the issue is beyond your current access, expertise, or authority to resolve — or when SLA is at risk and no progress is being made.

| Situation | Escalate To |
|-----------|------------|
| Root cause is in infrastructure or networking | Platform / Cloud infrastructure team |
| Security or access control issue | Security or IAM team |
| Source system unavailable or data not arriving | Source system owner or vendor |
| Requires a production deployment or hotfix | Engineering team or release manager |
| SLA breach imminent on a SEV-1 or SEV-2 | Support lead or manager |
| Business impact requires executive awareness | Account manager or service delivery manager |

!!! tip "Escalate early on SEV-1"
    On a SEV-1, loop in the relevant team within the first 30 minutes if you have not identified the root cause. Parallel investigation is faster than sequential handoffs.

---

## Tool-Specific Notes

=== "ServiceNow"

    - Create incidents via **Incident > Create New** or through the self-service portal
    - Set **Impact** and **Urgency** fields — ServiceNow calculates Priority automatically based on the Impact/Urgency matrix
    - Use **Assignment Group** to route to the correct team; avoid assigning directly to individuals unless escalating to a named person
    - Add **Work Notes** for internal updates (not visible to the reporter) and **Comments** for stakeholder-facing updates
    - Link related incidents using **Related Records** to identify recurring patterns
    - SEV-1 / P1 incidents automatically trigger major incident workflows in most ServiceNow configurations — confirm your organisation's setup

=== "Jira Service Management"

    - Create incidents from the **Queues** view or via email/portal submission
    - Set **Priority** directly on the ticket — Jira SM does not auto-calculate from impact/urgency unless configured with custom automation
    - Use **Internal Notes** for team-facing updates and standard comments for reporter-facing communication
    - Link tickets using **Link Issue > is caused by / blocks / relates to** to track dependencies between incidents and related problems
    - Escalation can be handled via **Escalation** field or by moving the ticket to a different queue with a higher SLA policy
    - Use **Automation Rules** in your project to trigger notifications on priority changes or SLA breach warnings

=== "Azure DevOps Boards"

    - Create incidents as **Work Items** of type **Bug** or a custom **Incident** type if your team has configured one
    - Set **Severity** and **Priority** fields; use **Area Path** to route to the correct team
    - Use **Discussion** section for all updates — both internal and stakeholder-facing (there is no native internal/external split unless using extensions)
    - Link work items using **Add Link > Related / Blocked By / Blocks** to track causal relationships
    - Azure DevOps does not have native SLA management — teams typically track SLA compliance via custom dashboards or reports built on the DevOps connector
    - For major incidents, use a dedicated **Epic** or **Feature** to group all related work items under a single incident record