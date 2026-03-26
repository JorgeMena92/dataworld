---
title: Deployment & ALM
description: Git integration, deployment pipelines, and CI/CD for Microsoft Fabric
tags: [fabric, deployment, alm, git]
---

# Deployment & ALM

Application Lifecycle Management (ALM) in Microsoft Fabric covers how you version, promote, and automate the deployment of workspaces, semantic models, notebooks, pipelines, and reports across environments. Fabric has native tooling for this — Git integration, deployment pipelines, and a REST API — that works across all Fabric item types, not just Power BI.

---

## Environments

A typical Fabric setup uses three separate workspaces as environments:

```
Development → Test → Production
```

Each workspace is assigned to a Fabric capacity. Using separate capacities per environment gives you billing isolation and ensures a broken development workload cannot consume production compute.

| Environment | Purpose | Capacity recommendation |
|-------------|---------|------------------------|
| **Development** | Active development, experimentation | Smallest F SKU or trial |
| **Test** | QA, UAT, integration testing | Same SKU as production or smaller |
| **Production** | End users and business operations | Sized for actual workload |

!!! warning
    Never develop directly in a production workspace. Even read-only access to production data from a development notebook is risky — a misconfigured write operation can corrupt live data.

---

## Git Integration

Fabric supports native Git integration at the workspace level. A workspace can be connected to a branch in Azure DevOps or GitHub, making all supported items version-controlled automatically.

### Setting it up

1. Go to **Workspace settings → Git integration**
2. Connect to an Azure DevOps or GitHub repository
3. Select the branch and folder
4. Choose whether to sync existing workspace content to Git or initialize from the repository

### What gets committed

Fabric items are stored as open, human-readable formats in the repository:

| Item type | Format in Git |
|-----------|--------------|
| Semantic model | TMDL (folder of `.tmdl` files) |
| Report | PBIR (folder of JSON files per visual) |
| Notebook | `.ipynb` (Jupyter format) |
| Pipeline | JSON definition |
| Dataflow Gen2 | JSON definition |
| Lakehouse | Metadata only (data stays in OneLake) |

!!! note
    Not all Fabric item types support Git integration yet. Microsoft is progressively adding support. Check the current list in the Fabric documentation before planning your branching strategy around a specific item type.

### Branching strategy

A simple branching strategy for Fabric:

```
main         ← production state, protected branch
  └── dev    ← integration branch, synced to Dev workspace
        └── feature/my-feature  ← individual work branches
```

Developers work in feature branches, merge to `dev` (which syncs to the Development workspace), and promote to `main` via pull request after review.

!!! tip
    Use branch policies in Azure DevOps or GitHub to require pull request reviews before merging to `main`. This gives you a review gate before anything reaches production.

---

## Deployment Pipelines

Deployment pipelines in Fabric allow you to promote content from one workspace stage to the next with a single click — or via API for automated deployments.

### Setup

1. In Fabric, go to **Workspaces → Deployment pipelines → New pipeline**
2. Assign the Development, Test, and Production workspaces to the three stages
3. Configure deployment rules for each stage (data source bindings, parameter overrides)

### Deployment rules

The most critical configuration — deployment rules let each stage point to its own data sources without changing the item definitions:

```
Development stage → dev-lakehouse / dev-database
Test stage        → test-lakehouse / test-database
Production stage  → prod-lakehouse / prod-database
```

Set rules in **Pipeline → Stage settings → Deployment rules** for each semantic model, dataflow, or pipeline that connects to a data source.

!!! warning
    Without deployment rules, promoting a semantic model to production keeps it pointing at the development data source. Always configure rules before the first production deployment.

### What can be deployed

Deployment pipelines support most Fabric item types — semantic models, reports, notebooks, pipelines, dataflows, lakehouses, warehouses, and more. The pipeline shows a diff of what has changed between stages before you deploy, so you can review what will be promoted.

---

## CI/CD with Azure DevOps

For teams that want fully automated deployments triggered by Git events (merges to `main`, pull request approvals), the Fabric REST API enables scripted deployments from Azure DevOps pipelines.

### Typical workflow

```
Developer merges PR to main
        │
        ▼
Azure DevOps pipeline triggered
        │
        ├── Update workspace from Git branch
        │
        ├── Run deployment pipeline (Dev → Test)
        │
        └── After approval gate: run deployment pipeline (Test → Prod)
```

### Fabric REST API

The Fabric REST API covers workspace management, item deployment, and pipeline execution:

```yaml
# Example Azure DevOps pipeline step — trigger Fabric deployment pipeline
- task: PowerShell@2
  displayName: 'Deploy to Test'
  inputs:
    script: |
      $token = (Get-AzAccessToken -ResourceUrl "https://api.fabric.microsoft.com").Token
      $headers = @{ Authorization = "Bearer $token"; "Content-Type" = "application/json" }
      $body = @{ sourceStageOrder = 0 } | ConvertTo-Json
      Invoke-RestMethod `
        -Uri "https://api.fabric.microsoft.com/v1/pipelines/$(pipelineId)/deploy" `
        -Method POST `
        -Headers $headers `
        -Body $body
```

### Service principals

Always use a service principal for automated deployments — never a personal account:

1. Register an app in **Azure Entra ID → App registrations**
2. Generate a client secret
3. In Fabric Admin portal: **Admin settings → Developer settings → Allow service principals to use Fabric APIs**
4. Add the service principal to each workspace as a **Member** or **Admin**

→ See [Deployment & Service](../powerbi/deployment.md) for the full service principal setup walkthrough.

---

## PBIP and Git-Friendly Formats

Fabric's Git integration works best with open file formats. For Power BI content specifically:

- **PBIP** (Power BI Project) — stores the semantic model as TMDL and the report as PBIR, both fully diff-able in Git
- **PBIX** — binary format, can be stored in Git but diffs are not human-readable

For any workspace connected to Git, use PBIP format. PBIR became the default for new reports in January 2026 (Service) and March 2026 (Desktop).

→ See [Fundamentals](../powerbi/fundamentals.md) for the full PBIP / PBIR explanation.

---

## Workspace and Capacity Management

### Capacity per environment

Assign each environment workspace to its own capacity:

```
Dev workspace    → F8  capacity  (dev)
Test workspace   → F16 capacity  (test)
Prod workspace   → F64 capacity  (prod)
```

This ensures:
- Development Spark jobs or heavy refreshes do not throttle production queries
- Billing is isolated per environment
- You can pause the dev capacity overnight to reduce cost

### Pausing capacity

Fabric capacities can be paused when not in use — useful for development and test environments that are only needed during business hours:

- Pause via **Azure Portal → Fabric capacity → Pause**
- Automate with a Logic App or Azure Automation runbook on a schedule
- Data in OneLake is preserved when capacity is paused — only compute stops

!!! tip
    Pausing a dev F8 capacity overnight and on weekends can reduce its cost by 60-70%. Production capacity should never be paused — it serves live users.

---

## Best Practices

- Use separate workspaces for dev, test, and production — never share a workspace across environments
- Connect all workspaces to Git from the start — retrofitting version control is always harder
- Configure deployment rules before the first production deploy — especially data source bindings
- Use service principals for all automated deployments — never personal accounts
- Use PBIP format for Power BI content in Git-connected workspaces
- Protect the `main` branch with pull request policies — no direct pushes
- Pause dev and test capacities outside business hours to reduce cost
- Monitor capacity usage with the **Fabric Capacity Metrics app** — throttling is silent and hard to debug without it
