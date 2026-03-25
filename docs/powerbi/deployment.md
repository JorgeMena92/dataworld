---
title: Deployment & Service
description: Workspaces, gateways, apps, RLS, and CI/CD with Azure DevOps
tags: [powerbi, deployment, service]
---

# Deployment & Service

Building a report is only half the work. Deploying it reliably, keeping it updated, securing it properly, and making it accessible to the right people requires understanding Power BI Service and its ecosystem.

---

## Power BI Service Overview

Power BI Service is the cloud platform where reports are published, shared, and managed. Key components:

- **Workspaces** — containers for reports, semantic models, and dashboards where teams collaborate
- **Apps** — packaged collections of reports distributed to end users
- **Semantic Models** — the published data models that reports connect to (formerly called datasets)
- **Gateways** — connectors between Power BI Service and on-premise or private network data sources
- **Deployment Pipelines** — tools for promoting content between development, test, and production environments

---

## Workspaces

Workspaces are where teams collaborate on Power BI content. There are two types:

- **My Workspace** — personal, not for production use. Content here cannot be shared properly and has no role-based access.
- **Shared Workspaces** — team workspaces used for development, testing, and production. All production content must live here.

### Workspace Roles

| Role | Can publish | Can edit | Can share | Can manage access |
|------|-------------|----------|-----------|------------------|
| Admin | ✅ | ✅ | ✅ | ✅ |
| Member | ✅ | ✅ | ✅ | ❌ |
| Contributor | ✅ | ✅ | ❌ | ❌ |
| Viewer | ❌ | ❌ | ❌ | ❌ |

!!! tip
    Grant the minimum role needed. Most business users consuming reports through an App do not need any workspace role at all — App access is managed separately from workspace access.

---

## Deployment Pipelines

Deployment pipelines allow you to promote content across three environments in sequence:

```
Development → Test → Production
```

Each stage is a separate workspace. Content is promoted stage by stage — developers work in Development, QA validates in Test, and end users consume from Production. This prevents development work from breaking live reports.

### Stage behavior

- **Deploy** — copies reports and semantic models from one stage to the next
- **Compare** — shows a diff of what has changed between stages before deploying
- **Rules** — per-stage configuration overrides, such as pointing the Production semantic model at the production database instead of the development one

### Dataset binding (deployment rules)

The most important use of rules is **dataset binding** — configuring each stage to connect to its own data source without changing the report file:

```
Development stage → dev-server / dev_database
Test stage        → test-server / test_database
Production stage  → prod-server / prod_database
```

Set this up in **Deployment pipeline → Stage rules → Data source rules** for each semantic model. This means the same report file can be promoted all the way to production without any manual connection changes.

!!! warning
    Without deployment rules, promoting a report to production will keep it pointing at the development data source. Always configure stage rules before deploying to production for the first time.

---

## Gateways

A gateway is required when Power BI Service needs to reach a data source that is not publicly accessible — on-premise databases, files on internal servers, or resources inside a private network.

### Gateway types

| Type | Use case |
|------|---------|
| **On-premises data gateway** | Shared gateway for multiple users and semantic models. Installed on a server inside the network. |
| **Personal gateway** | Single-user only. Not suitable for production — if the user's machine is off, refresh fails. |
| **VNet data gateway** | Cloud-based gateway for Azure-hosted sources inside a Virtual Network. No on-prem server required. |

!!! note
    The VNet gateway is the modern alternative for organizations with data sources hosted in Azure but inside a private Virtual Network (Azure SQL, Synapse, etc.). It eliminates the need to maintain a physical gateway server while keeping the data source private.

### Gateway best practices

- Install the on-premises gateway on a stable, always-on server — not a developer's laptop
- Use a **service account**, not a personal account, to register and run the gateway
- Monitor gateway health and cluster status in **Power BI Service → Settings → Manage gateways**
- Set up gateway clusters (multiple gateway nodes) for high availability in production

---

## Scheduled Refresh

After publishing a semantic model in Import mode, configure a refresh schedule so the data stays up to date.

| Connection mode | Refresh needed |
|-----------------|---------------|
| Import | ✅ Scheduled (up to 8×/day on Pro, 48×/day on Premium/Fabric) |
| Direct Query | ❌ Always live |
| Live Connection | ❌ Always live |

Always configure **refresh failure notifications** — Power BI can email the dataset owner when a refresh fails, which is the fastest way to catch broken connections before users notice.

### Incremental refresh

For large tables, incremental refresh loads only new or changed rows instead of reloading the entire table on every refresh. This significantly reduces refresh time and resource usage.

Set it up in Power Query by defining `RangeStart` and `RangeEnd` parameters, then configuring the incremental refresh policy in Power BI Desktop under **Table tools → Incremental refresh**.

!!! note
    Incremental refresh requires Premium or Fabric capacity for the full feature set. On Pro, basic incremental refresh is available but with limitations. It is most valuable for fact tables with millions of rows that grow daily.

---

## Apps

Apps are the recommended way to distribute reports to end users. An app packages workspace content into a clean, curated experience separate from the workspace itself.

### App vs workspace access

| | Workspace access | App access |
|--|-----------------|------------|
| Who it's for | Developers and contributors | End users and consumers |
| What they see | All content including drafts | Only what the app publisher chooses to include |
| Affected by workspace changes | Immediately | Only when the app is republished |
| Required license | Pro or PPU | Free (if Premium capacity) or Pro |

### App audiences

A single app can be published to multiple **audiences** — each audience sees a different subset of reports and has different permissions. This allows one app to serve different teams without creating multiple separate apps.

```
App: Sales Analytics
├── Audience: Sales Team      → sees Sales Dashboard + Regional Breakdown
├── Audience: Finance Team    → sees Sales Dashboard + P&L Summary
└── Audience: Executives      → sees Executive Summary only
```

Configure audiences in **Power BI Service → Workspace → Create app → Audience**.

!!! tip
    Always distribute to end users via Apps rather than direct workspace access. Apps give you control over what users see, decouple the publishing cycle from the development cycle, and provide a cleaner navigation experience.

---

## Row-Level Security (RLS)

Row-Level Security restricts which rows of data a user can see based on their identity. The same report shows different data to different users — a regional manager sees only their region, a salesperson sees only their accounts.

### Static RLS

Define fixed filter rules in Power BI Desktop under **Modeling → Manage roles**:

```dax
-- Role: "Europe Region"
-- Table: dim_geography
-- Filter:
[region] = "Europe"
```

Assign users or groups to roles in Power BI Service under **Semantic model → Security**.

### Dynamic RLS

Dynamic RLS uses the logged-in user's identity to filter data automatically — no need to create one role per user.

```dax
-- Role: "My Region"
-- Table: dim_salesperson
-- Filter:
[email] = USERPRINCIPALNAME()
```

`USERPRINCIPALNAME()` returns the email address of the user currently viewing the report. The filter applies automatically based on who is logged in — one role covers all users.

!!! tip
    Prefer dynamic RLS over static RLS whenever the filtering logic can be expressed using the user's identity. Static roles require manual maintenance every time a user changes region or team — dynamic RLS handles it automatically as long as the underlying data is up to date.

!!! warning
    RLS only applies to **Import** and **Direct Query** modes. It does not apply to Live Connection reports — security for those is managed at the Analysis Services or Fabric level. Also, workspace Admins, Members, and Contributors **bypass RLS** — they always see all data. Only Viewers and App users are subject to RLS filters.

### Testing RLS

Always test RLS before publishing. In Power BI Desktop: **Modeling → View as → select a role**. In Power BI Service: **Semantic model → Security → Test as role**.

---

## Service Principals

A service principal is an application identity in Azure Active Directory used for automated, non-interactive authentication — deployments, scheduled tasks, and API calls that should not depend on a specific person's account.

### Why use service principals

- Personal accounts require the user to be active and licensed — if they leave, deployments break
- Service principals do not expire with employee turnover
- They can be granted the minimum permissions needed without a full user license

### Setting up a service principal for Power BI

1. Register an app in **Azure Active Directory → App registrations**
2. Generate a client secret or certificate for authentication
3. In Power BI Service: **Admin portal → Developer settings → Allow service principals to use Power BI APIs** — enable and scope to a security group
4. Add the service principal to the target workspace as a **Member** or **Admin**

```yaml
# Azure DevOps pipeline using a service principal
- task: PowerShell@2
  displayName: 'Deploy to Power BI'
  inputs:
    script: |
      Import-Module MicrosoftPowerBIMgmt
      Connect-PowerBIServiceAccount `
        -ServicePrincipal `
        -Credential $(clientSecret) `
        -TenantId $(tenantId)
      New-PowerBIReport -Path report.pbix -WorkspaceId $(workspaceId)
```

!!! warning
    Never use a personal account for automated deployments. If the account owner changes their password, enables MFA, or leaves the organization, every automated pipeline that uses that account will break.

---

## CI/CD with Azure DevOps and Fabric Git

### File formats

Power BI supports two file formats for version control:

| Format | Description |
|--------|-------------|
| `.pbix` | Binary file — the traditional format. Can be stored in Git but diffs are not human-readable. |
| `.pbip` (PBIP) | Project format — stores the report and semantic model as a folder of text files (JSON, TMDL). Fully diff-able and mergeable in Git. |

PBIP is the recommended format for teams using version control — it makes pull request reviews meaningful and enables proper branching strategies.

### Azure DevOps pipeline

```yaml
# Example Azure DevOps pipeline step
- task: PowerShell@2
  displayName: 'Deploy to Power BI'
  inputs:
    script: |
      Import-Module MicrosoftPowerBIMgmt
      Connect-PowerBIServiceAccount -ServicePrincipal
      New-PowerBIReport -Path report.pbix -WorkspaceId $(workspaceId)
```

### Fabric Git integration

Microsoft Fabric introduces native Git integration directly in the Service — no Azure DevOps pipeline needed for basic workflows:

1. Connect a workspace to a Git branch in **Workspace settings → Git integration**
2. Changes made in the workspace are committed to the branch
3. Pull requests and branch policies are managed in Azure DevOps or GitHub as normal
4. Merging to the main branch can trigger a deployment pipeline to promote to production

!!! note
    Fabric Git integration works natively with PBIP format. It is the most streamlined option for teams already using Microsoft's ecosystem and removes the need to manage custom deployment scripts for the publish step.

---

## Best Practices

- Never develop directly in production workspaces — always use a Development workspace
- Use deployment pipelines even for small teams — the stage separation protects production
- Configure deployment rules before the first production deploy — especially data source bindings
- Use service principals for all automated deployments — never personal accounts
- Distribute to end users via Apps, never by granting workspace access
- Use App audiences to serve different teams from a single app instead of duplicating reports
- Always configure refresh failure notifications on every semantic model
- Use PBIP format for new projects if the team uses version control
- Test RLS thoroughly before publishing — always use "View as role" before go-live
- Workspace Admins and Members bypass RLS — be deliberate about who gets those roles