---
title: Communication Standards
description: How to communicate during incidents, in handovers, and with stakeholders across a data platform support role.
tags: [operations, support, communication, stakeholders]
---

# Communication Standards

Clear, consistent communication is one of the most important skills in a support role. During an incident, stakeholders want to know what is happening, what is being done, and when it will be resolved. In a handover, the incoming engineer needs to operate independently from the first minute. In both cases, the quality of your communication directly affects trust in the team.

---

## Core Principles

These principles apply to all communication in a support context — incident updates, tickets, handovers, and stakeholder messages.

- **Acknowledge first** — confirm receipt before you have an answer. Silence is worse than "I'm looking into it."
- **Be specific about what you know and what you don't** — "I have identified the layer where the issue is occurring" is more useful than "still investigating"
- **Avoid technical jargon with business stakeholders** — translate symptoms and impact into business terms
- **State the next action and the next update time** — never leave a stakeholder without a clear "what happens next"
- **Use consistent timestamps** — always include the time zone, especially in global teams. UTC is the safest default
- **Keep records** — all significant communications during an incident should be traceable in the ticket

---

## Incident Communication

### Initial Acknowledgement

Send within the SLA response window (see [Incident Management](incident-management.md)). It does not need to contain a diagnosis — only confirmation that the issue is being investigated.

**Template:**
```
Hi [Name / Team],

Thank you for raising this. I have logged a ticket ([ID]) and I am currently 
investigating. I will provide an update by [time].

[Your name]
```

For automated alerts or proactively detected incidents, send the acknowledgement to the relevant stakeholder group rather than waiting for someone to ask.

### Progress Update

Send at the frequency defined by severity (every 30 minutes for SEV-1, every 2 hours for SEV-2). Even if there is no new information, send an update — it prevents escalation and builds trust.

**Template:**
```
Update — [Ticket ID] — [HH:MM UTC]

Current status: [One sentence on where investigation stands]
What has been ruled out: [Layers or causes confirmed not to be the issue]
Current focus: [What is being investigated right now]
Next update: [Time]

[Your name]
```

!!! tip "The update does not need to be long"
    A three-line update sent on time is worth more than a detailed update sent late. Stakeholders prioritise timeliness over depth during an active incident.

### Resolution Notification

Sent when the incident is resolved and the user impact has been eliminated.

**Template:**
```
Resolved — [Ticket ID] — [HH:MM UTC]

The issue has been resolved. [One sentence describing what was wrong and what was done.]

[If a workaround is in place rather than a permanent fix:]
Note: A temporary workaround is currently in place. A permanent fix is scheduled 
for [date/time]. We will notify you when this is completed.

Please confirm on your end that everything is working as expected. If you experience 
any further issues, reply to this message or raise a new ticket.

[Your name]
```

### Outage Communication (SEV-1)

For a SEV-1 affecting multiple users or business-critical processes, a broader outage communication is required. This is sent to a wider audience — typically a distribution list, a Teams or Slack channel, or a status page update.

**Template:**
```
[PLATFORM ALERT] [Affected Service] — [HH:MM UTC]

We are currently experiencing an issue with [service/component]. 
[One sentence describing the user-visible impact — e.g. "Power BI dashboards 
are currently unavailable for all users."]

Our team is actively investigating. We will provide updates every 30 minutes.

Ticket reference: [ID]
Next update: [HH:MM UTC]
```

Follow-up updates for outage communications follow the same format as progress updates above. When resolved, send a clear **All Clear** message to the same audience.

**All Clear Template:**
```
[RESOLVED] [Affected Service] — [HH:MM UTC]

The issue with [service/component] has been resolved. Full service has been restored.

[One sentence root cause summary if appropriate for the audience.]

Duration: [Start time] to [End time] ([total duration])
Ticket reference: [ID]

We apologise for the disruption. A full post-mortem will be completed and 
shared with the relevant teams.
```

---

## Handover Communication

The written handover (see [Daily Operations](daily-operations.md)) should be complemented by a **verbal or chat-based handover** for active SEV-1 and SEV-2 incidents. A written document alone is not sufficient when an active incident requires immediate action from the incoming shift.

For active incidents at shift change:

- Brief the incoming engineer verbally (or via a live call) on the current state
- Walk through the investigation timeline and what has been ruled out
- Confirm ownership transfer — the incoming engineer explicitly acknowledges they are now the owner
- Update the ticket with a handover entry: `[HH:MM UTC] Ownership transferred to [Name] at shift change`

!!! warning "Do not leave an active SEV-1 unattended during shift change"
    The outgoing engineer should remain available for at least 15 minutes after handing over a SEV-1, in case the incoming engineer needs context that is not in the written record.

---

## Stakeholder Communication

### Knowing Your Audience

Different stakeholders need different levels of detail.

| Audience | What They Need | What to Avoid |
|----------|---------------|---------------|
| End users / report consumers | Is it fixed? When will it be fixed? | Technical root cause details |
| BI team / data team | Technical detail, root cause, data impact | Oversimplification that loses accuracy |
| Management / leadership | Business impact, duration, resolution status | Raw technical logs or stack traces |
| Vendor / external support | Full technical detail, error messages, logs | Vague symptom descriptions |

### Managing Expectations During Long Incidents

When a SEV-1 or SEV-2 extends beyond the initial resolution target, proactive expectation management is essential.

- Acknowledge the delay explicitly — do not wait for the stakeholder to ask
- Provide a revised estimate with reasoning: *"We have identified the root cause in the pipeline layer and are working on a fix. We expect resolution by [time]. The delay is due to [brief reason]."*
- If no estimate is possible, say so directly: *"We do not yet have an estimated resolution time. I will update you by [time] with either a resolution or a revised estimate."*
- Never give an estimate you are not confident in — a missed estimate damages trust more than no estimate

### Escalation Communication

When escalating to another team, provide everything they need to act without asking follow-up questions.

**Escalation message template:**
```
Hi [Team],

I need your support on a [SEV-X] incident currently affecting [service/users].

Ticket: [ID]
Summary: [One sentence symptom description]
Impact: [Who and what is affected]
What we have investigated: [Layers checked, what was ruled out]
Where we are stuck: [Specific blocker — access, expertise, system ownership]
SLA status: [Time remaining before breach / already breached]

Please can you [specific ask — e.g. check the ADLS container ACLs / review the 
gateway configuration / confirm source system availability].

I am available on [channel] if you need to discuss.

[Your name]
```

---

## Writing Quality

Support communication reflects the team's professionalism. Apply these standards consistently.

- **Use plain English** — write short sentences. Avoid passive voice where possible.
- **Lead with the most important information** — status first, detail second
- **Proofread before sending** — especially for outage communications that go to a wide audience
- **Use consistent terminology** — refer to platform components by their correct names (Databricks Workflow, not "the job thing"; Power BI semantic model, not "the dataset" in technical contexts)
- **Avoid hedging language in resolutions** — "it seems to be working now" is not a resolution. Confirm it is working before closing.

---

## Tool-Specific Notes

=== "ServiceNow"

    - Use **Work Notes** for all internal communication and investigation notes — these are not visible to the reporter
    - Use **Comments** (or the customer-facing reply) for all stakeholder-facing communication — these are visible to the reporter and trigger email notifications
    - For major incidents, use the **Major Incident Communication** feature to send broadcast updates to affected user groups from within the incident record
    - ServiceNow's **Notify** module (if enabled) supports SMS and voice notifications for SEV-1 escalations
    - All emails sent from ServiceNow are automatically logged against the ticket — avoid communicating via personal email during incidents as it breaks the audit trail

=== "Jira Service Management"

    - Use **Internal Notes** for team-facing communication — reporters cannot see these
    - Use standard **Comments** for reporter-facing communication — these trigger email notifications to the reporter and any watchers
    - Add stakeholders as **Watchers** on high-severity tickets so they receive automatic notifications on all updates
    - For outage communications to a broad audience, use a connected **Statuspage** (Atlassian's status page product) if your team has configured one — updates can be posted from within Jira SM
    - Jira SM does not have a native broadcast/mass notification feature — use your team's chat platform (Teams, Slack) for wide audience outage communications and link back to the ticket

=== "Azure DevOps Boards"

    - All communication in Azure DevOps Boards is in the **Discussion** section — there is no native internal/external split, so be mindful of what is visible if external users have access to your project
    - Use `@mentions` to notify specific team members or teams of critical updates
    - For broad stakeholder communication, Azure DevOps does not have a native notification broadcast — use your team's email distribution list or chat channel and reference the ticket ID
    - If your organisation uses **Microsoft Teams**, consider setting up a Teams channel notification from Azure DevOps using the Azure DevOps Teams app — this pushes work item updates to a channel automatically
    - For post-incident communication and status updates, maintain a **Wiki page** for the incident that stakeholders can reference, linked from the work item
