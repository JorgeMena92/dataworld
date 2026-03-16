---
title: Daily Operations & Handover
description: Shift start checklist, platform health checks, and handover template for data platform support.
tags: [operations, support, daily-ops, handover]
---

# Daily Operations & Handover

Daily operations in a data platform support role means starting every shift with a clear picture of platform health, knowing what is running, what has failed, and what was left unresolved by the previous shift. A structured handover ensures nothing is lost between teams.

---

## Shift Start Checklist

Run this checklist at the beginning of every shift before picking up any new tickets.

### 1 · Review the Handover

- Read the handover notes from the outgoing shift (see [Handover Template](#handover-template) below)
- Identify any open SEV-1 or SEV-2 incidents that require immediate attention
- Note any ongoing investigations or pending actions that belong to you now
- Confirm there are no SLA breaches already in progress

### 2 · Platform Health Check

Check each platform component in order. The goal is to identify failures or anomalies **before** users report them.

#### Power BI

- [ ] Open the **Power BI Admin Portal** and check for service-level alerts
- [ ] Review the **Refresh History** for all critical datasets — confirm last successful refresh and check for failures
- [ ] Check the **Gateway status** — confirm all on-premises data source connections are healthy
- [ ] Review any scheduled refreshes that should have run overnight — confirm they completed on time

#### Azure Databricks

- [ ] Open **Databricks Workflows** and review the run history for all production jobs
- [ ] Identify any failed, cancelled, or abnormally long runs from the overnight window
- [ ] Check cluster health — confirm that job clusters terminated cleanly and no interactive clusters are left running unexpectedly
- [ ] Review **Databricks alerts** or notification emails for any job failures

#### Azure Data Factory / Microsoft Fabric Pipelines

- [ ] Open **ADF Monitor** or **Fabric Monitor** and review pipeline run history
- [ ] Check for failed or skipped pipeline runs in the overnight window
- [ ] Confirm trigger schedules fired as expected — look for missing runs, not just failed ones
- [ ] Review any pipelines in a **waiting** state that may be blocked on dependencies

#### Azure Data Lake Storage

- [ ] Spot-check expected file arrivals in the bronze zone for overnight ingestion jobs
- [ ] Confirm that landing zone files have been processed and moved to the appropriate tier
- [ ] Check for any storage access errors in the diagnostic logs if anomalies were flagged

#### Azure Platform Health

- [ ] Check the **Azure Service Health** dashboard for any active incidents or planned maintenance in your regions
- [ ] Check the **Databricks status page** for any ongoing cloud provider issues
- [ ] Check the **Power BI service status page** for any active degradations

### 3 · Open Ticket Review

- Review the ticket queue and sort by severity and SLA deadline
- Identify tickets where SLA response or update is due within the next 2 hours
- Assign unassigned tickets appropriately
- Add an acknowledgement or update to any ticket that has been waiting more than 24 hours without a response

---

## Shift End Checklist

Before handing over, complete the following:

- [ ] Update all open tickets with a current status and next action
- [ ] Resolve or close any tickets that were completed during the shift
- [ ] Document anything that was investigated but not resolved — include findings, steps taken, and what was ruled out
- [ ] Complete the handover note (see below)
- [ ] Flag any upcoming scheduled maintenance, deployments, or known risks for the incoming shift

---

## Handover Template

The handover note is the single most important communication in a support role. A good handover means the incoming engineer can operate independently from the first minute of their shift.

Keep it factual, brief, and actionable. Use the structure below consistently — it makes scanning fast.

---

```
HANDOVER — [Date] [Shift: Morning / Afternoon / Night]
Engineer: [Your name]
```

### Platform Status

| Component | Status | Notes |
|-----------|--------|-------|
| Power BI | ✅ Healthy / ⚠️ Degraded / 🔴 Incident | |
| Databricks | ✅ Healthy / ⚠️ Degraded / 🔴 Incident | |
| ADF / Fabric Pipelines | ✅ Healthy / ⚠️ Degraded / 🔴 Incident | |
| Azure Infrastructure | ✅ Healthy / ⚠️ Degraded / 🔴 Incident | |

### Active Incidents

List all open SEV-1 and SEV-2 incidents. For each:

```
Ticket: [ID]
Severity: SEV-[1/2]
Summary: [One sentence description of the issue]
Current status: [What is known, what has been ruled out, where investigation stands]
Next action: [What needs to happen next, and who owns it]
SLA deadline: [Timestamp]
```

### Resolved During This Shift

Brief summary of tickets closed during the shift — one line each is enough.

```
[Ticket ID] — [Issue summary] — Resolved by [fix/workaround/rollback]
```

### Pending Actions

Tasks that were not completed and need follow-up from the incoming shift:

```
- [Action description] — Owner: [team/person] — Due: [time/date]
```

### Known Risks & Watch Items

Anything that is not yet an incident but warrants monitoring:

```
- [Description of risk or anomaly] — Being watched because [reason]
```

### Notes for Incoming Shift

Anything else the incoming engineer needs to know — upcoming deployments, maintenance windows, stakeholder commitments, pending change requests.

---

## Tool-Specific Notes

=== "ServiceNow"

    - Use **My Work** or the **Support Queue** view to review open tickets at shift start
    - Filter by **Assignment Group** and sort by **Priority** then **SLA Due**
    - Post the handover content as a **Work Note** on each active SEV-1 / SEV-2 ticket so the incoming engineer can see the current state directly on the ticket
    - Use the **Major Incident Workbench** for active SEV-1s — it centralises timeline, updates, and team communication in one view
    - ServiceNow's **On-Call Scheduling** module manages shift assignments if your team uses it — confirm you are listed as active for your shift

=== "Jira Service Management"

    - Use the **Queues** view filtered by your team's assignment and sorted by Priority and SLA remaining time
    - At shift end, update all active tickets with a **Work Log** entry summarising status and next action
    - Post the handover note as an **Internal Note** on each active SEV-1 / SEV-2 ticket
    - Use **Jira's SLA panel** on each ticket to confirm how much time remains before breach
    - If your team uses Confluence alongside Jira SM, maintain a shared **Handover** page in Confluence that the incoming shift updates at the start of each shift

=== "Azure DevOps Boards"

    - Use the **Boards > Queries** view with a saved query filtered to Active, In Progress, and your team's Area Path, sorted by Severity
    - Post shift-end updates in the **Discussion** section of each active ticket
    - For handover documentation, maintain a shared **Wiki page** (Azure DevOps Wiki) with the handover template — update it at the end of each shift
    - Azure DevOps has no native SLA management — if your team tracks SLA via a Power BI report on the DevOps connector, review that dashboard at shift start to identify at-risk tickets
