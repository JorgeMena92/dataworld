---
title: Incident Tracking & Documentation
description: How to write an incident record, structure a post-mortem, and contribute to the knowledge base.
tags: [operations, support, documentation, post-mortem, knowledge-base]
---

# Incident Tracking & Documentation

Good documentation is not a bureaucratic formality — it is how a support team gets faster over time. A well-written incident record means the next engineer who sees the same issue can resolve it in minutes instead of hours. A thorough post-mortem means the same incident does not happen again.

---

## The Incident Record

Every incident, regardless of severity, needs a written record. The record is created when the incident is detected and updated throughout the lifecycle — not written from memory after resolution.

### Required Fields

| Field | Description |
|-------|-------------|
| **Ticket ID** | Unique identifier from your ticketing system |
| **Title** | One clear sentence describing the user-visible symptom — not the technical cause |
| **Severity** | SEV-1 through SEV-4 (see [Incident Management](incident-management.md)) |
| **Detected At** | Timestamp when the issue was first identified |
| **Reported By** | User, automated alert, or proactive check |
| **Affected Component** | Power BI / Databricks / ADF / Fabric / ADLS / Source system |
| **Affected Users or Processes** | Who or what is impacted — be specific |
| **Status** | Open · In Progress · Resolved · Closed |
| **Owner** | The engineer currently responsible for the ticket |

### Timeline

The timeline is the most valuable part of the incident record. It captures what happened, when, and what was done in response — in chronological order.

```
[YYYY-MM-DD HH:MM UTC] Incident detected — [how it was detected]
[YYYY-MM-DD HH:MM UTC] Ticket created — Severity assigned: SEV-[X]
[YYYY-MM-DD HH:MM UTC] Initial acknowledgement sent to reporter
[YYYY-MM-DD HH:MM UTC] Investigation started — [layer being checked]
[YYYY-MM-DD HH:MM UTC] [Finding] — [what was found, what was ruled out]
[YYYY-MM-DD HH:MM UTC] [Action taken] — [result]
[YYYY-MM-DD HH:MM UTC] Root cause identified — [brief description]
[YYYY-MM-DD HH:MM UTC] Resolution applied — [fix / workaround / rollback]
[YYYY-MM-DD HH:MM UTC] User confirmed resolution
[YYYY-MM-DD HH:MM UTC] Ticket closed
```

!!! tip "Write as you go"
    Add timeline entries as events happen, not at the end. During a SEV-1, you will not remember the exact sequence after the fact. Three-word entries are fine — the timestamp is what matters most.

### Resolution Summary

The resolution summary is a short, self-contained paragraph written when the incident is closed. It should be readable by a non-technical stakeholder and should answer three questions:

- What was the issue and who was affected?
- What caused it?
- How was it resolved and what is the current status?

**Example:**
> The daily sales dashboard was showing data from two days prior due to a Databricks job that had been silently failing since the previous deployment. The job was restarted manually and completed successfully, restoring current data to the report. The pipeline failure alert was not configured for this job — an alert has been added as a follow-up action.

---

## Post-Mortem

A post-mortem is a structured review conducted after a SEV-1 or SEV-2 incident is resolved. The goal is to understand what happened deeply enough to prevent recurrence — not to assign blame.

Run the post-mortem within **48–72 hours** of resolution, while the details are still fresh.

### When to Write a Post-Mortem

| Severity | Post-Mortem Required |
|----------|---------------------|
| SEV-1 | Always |
| SEV-2 | Always |
| SEV-3 | If recurring (same issue 2+ times in 30 days) |
| SEV-4 | Not required |

### Post-Mortem Template

```
POST-MORTEM — [Ticket ID] — [Incident Title]
Date of Incident: [YYYY-MM-DD]
Date of Post-Mortem: [YYYY-MM-DD]
Author: [Name]
Participants: [Names / teams involved]
```

#### Summary

Two to three sentences covering what happened, how long it lasted, and what the user impact was.

#### Timeline

Full chronological timeline from detection to resolution (copy from the incident record and expand if needed).

#### Root Cause

A clear, specific explanation of the technical root cause. Avoid vague language like "human error" — describe the exact condition that caused the failure.

**Good:** *The Databricks job cluster was using a fixed instance type that became unavailable in the region due to capacity constraints, causing the job to queue indefinitely without raising an alert.*

**Not useful:** *The job failed due to a configuration issue.*

#### Contributing Factors

Secondary conditions that made the incident worse, harder to detect, or slower to resolve. These are as important as the root cause.

- No alert was configured for job queue timeout
- The job failure mode was silent — no error was raised, only a missed completion
- The monitoring dashboard did not cover this pipeline

#### Impact Assessment

| Dimension | Detail |
|-----------|--------|
| Duration | Total time from first impact to full resolution |
| Users affected | Number and teams |
| Processes affected | Pipelines, reports, downstream systems |
| Data impact | Data loss / stale data / incorrect data / none |
| SLA | Met / Breached — if breached, by how long |

#### What Went Well

Honest assessment of what worked during the response — detection speed, communication, tooling, team coordination.

#### What Could Be Improved

Specific, actionable observations — not general criticism. Each point should lead directly to an action item.

#### Action Items

| Action | Owner | Due Date | Ticket |
|--------|-------|----------|--------|
| Add job timeout alert to Databricks Workflow | [Name] | [Date] | [ID] |
| Update runbook for pipeline failure pattern | [Name] | [Date] | [ID] |
| Review all production jobs for missing alerting | [Name] | [Date] | [ID] |

!!! warning "Action items without owners and due dates are just wishes"
    Every action item must have a named owner and a due date. Track them as separate tickets or tasks in your backlog — not just in the post-mortem document.

---

## Knowledge Base Contribution

The knowledge base is the team's collective memory. Every resolved incident is an opportunity to make the next one faster.

### When to Write a Knowledge Base Article

Write an article when:

- You resolved an issue that took more than 2 hours to diagnose and the solution was not obvious
- You found that no documentation existed for a recurring issue
- You developed a diagnostic approach or workaround that others would benefit from
- A post-mortem produced a reusable finding

### Knowledge Base Article Structure

```
Title: [Short, searchable description of the problem — write it as someone would search for it]
Tags: [platform components, symptom keywords]
Last updated: [Date]
```

**Symptom**
What the user or engineer sees — the observable behaviour that leads someone to this article.

**Environment**
Which platform components, tools, or configurations this applies to.

**Root Cause**
What actually causes this symptom. Keep it precise.

**Diagnosis Steps**
How to confirm this is the issue you are dealing with — specific queries, log locations, UI checks.

**Resolution**
Step-by-step fix or workaround. Number each step. Include exact commands, paths, or UI navigation where relevant.

**Prevention**
What change reduces the likelihood of this happening again.

**Related Articles / Tickets**
Links to related knowledge base articles or the original incident ticket.

---

## Tool-Specific Notes

=== "ServiceNow"

    - Use **Incident > Notes** tab to maintain the running timeline — Work Notes for internal entries, Comments for stakeholder-facing updates
    - Use the **Problem** record type (linked from the Incident) to track recurring issues and their root cause investigations — Problems are separate from Incidents in ServiceNow's data model
    - Post-mortems can be documented as a **Problem record** with a **Root Cause Analysis** (RCA) attachment, or as a **Knowledge article** linked to the incident
    - To contribute to the knowledge base, create a **Knowledge > Article** and submit it for approval — most ServiceNow configurations require a knowledge manager to publish articles
    - Use **Related Records** to link the Incident to its Problem record, Change Request, and resulting Knowledge article so the full chain is traceable

=== "Jira Service Management"

    - Maintain the timeline in the ticket's **Activity** section using **Internal Notes** (not visible to reporters) for investigation entries
    - Jira SM has a native **Post-incident review** feature in some configurations — check if your project has it enabled under **Incident management settings**
    - Link the incident ticket to a **Jira Software issue** for any follow-up engineering work (bugs, improvements) using **Link Issue**
    - For knowledge base articles, use **Confluence** — create a page in a dedicated **Runbooks** or **Known Issues** space and link it back to the Jira ticket using the Confluence macro or a manual link
    - Tag knowledge articles with Jira issue keys so they are discoverable from both directions

=== "Azure DevOps Boards"

    - Maintain the timeline in the **Discussion** section of the work item — use `@mentions` to notify team members of key updates
    - For post-mortems, create a dedicated **Wiki page** in the Azure DevOps Wiki under a `Post-Mortems` section, named by date and incident ID
    - Link the incident work item to the Wiki page using the **Add Link > Wiki page** option (available in some configurations) or paste the Wiki URL in the Description
    - For follow-up action items, create child **Tasks** under the incident work item so all resulting work is traceable to the original incident
    - Maintain a **Known Issues** Wiki section with articles following the structure above — use the Wiki search to make articles discoverable across the team
