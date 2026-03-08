---
title: Deployment & Service
description: Workspaces, gateways, apps, and CI/CD with Azure DevOps
tags: [powerbi, deployment, service]
---

# Deployment & Service

Building a report is only half the work. Deploying it reliably, keeping it updated, and making it accessible to the right people requires understanding Power BI Service and its ecosystem.

---

## Power BI Service Overview

Power BI Service is the cloud platform where reports are published, shared, and managed. Key components:

- **Workspaces** — containers for reports, datasets, and dashboards
- **Apps** — packaged collections of reports distributed to end users
- **Datasets (Semantic Models)** — the published data models that reports connect to
- **Gateways** — connectors between Power BI Service and on-premise data sources
- **Pipelines** — deployment pipelines for promoting content between environments

---

## Workspaces

Workspaces are where teams collaborate on Power BI content. There are two types:

- **My Workspace** — personal, not for production use
- **Shared Workspaces** — team workspaces used for development and production

### Workspace Roles

| Role | Can publish | Can edit | Can share | Can manage |
|---|---|---|---|---|
| Admin | ✅ | ✅ | ✅ | ✅ |
| Member | ✅ | ✅ | ✅ | ❌ |
| Contributor | ✅ | ✅ | ❌ | ❌ |
| Viewer | ❌ | ❌ | ❌ | ❌ |

---

## Deployment Pipelines

Deployment pipelines allow you to promote content across three environments:

```
Development → Test → Production
```

This separates active development from content that end users consume, reducing the risk of breaking live reports.

---

## Gateways

A gateway is required when your data source is on-premise (SQL Server, files on a local server, etc.) and Power BI Service needs to refresh it.

Two types:
- **On-premises data gateway** — shared gateway for multiple users and datasets
- **Personal gateway** — single-user, not recommended for production

Gateway best practices:
- Install the gateway on a stable, always-on server
- Use a service account, not a personal account
- Monitor gateway health in Power BI Service

---

## Scheduled Refresh

After publishing, configure a refresh schedule so the dataset stays up to date.

- **Import mode** — requires scheduled refresh (up to 8x/day on Pro, 48x/day on Premium)
- **Direct Query / Live Connection** — always live, no refresh needed

Always configure refresh failure notifications to catch broken connections early.

---

## Apps

Apps are the recommended way to distribute reports to end users. An app is a snapshot of workspace content packaged for consumption.

Benefits:
- Users see a clean, curated experience
- Developers can update the workspace without affecting the published app until they choose to update it
- Access is managed separately from the workspace

---

## CI/CD with Azure DevOps

For teams using version control and automated deployment:

1. Store `.pbix` files or PBIP format in a Git repository
2. Use Azure DevOps pipelines to automate publishing via the Power BI REST API
3. Use deployment pipelines in Power BI Service to promote between environments
4. Tag releases and document changes in pull requests

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

---

## Best Practices

- Never develop directly in production workspaces
- Use deployment pipelines even for small teams
- Use service principals instead of personal accounts for automated deployments
- Document dataset refresh schedules and gateway configurations
- Use Apps for end users — never share workspace access with non-developers
- Monitor usage metrics to identify unused reports and optimize resources
