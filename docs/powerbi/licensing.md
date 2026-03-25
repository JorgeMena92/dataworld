---
title: Licensing
description: Power BI license types, capacity, and cost considerations
tags: [powerbi, licensing]
---

# Licensing

Understanding Power BI licensing is essential for planning deployments, managing costs, and knowing what features are available to your team. The licensing model has two layers: **user licenses** (per person) and **capacity licenses** (per environment).

For current pricing, refer to the official Microsoft pages:
- [Power BI pricing](https://powerbi.microsoft.com/en-us/pricing/)
- [Microsoft Fabric pricing](https://azure.microsoft.com/en-us/pricing/details/microsoft-fabric/)

---

## User Licenses

User licenses are assigned per person and determine what each individual can do in Power BI.

### Free

- Access to Power BI Desktop (always free)
- Publish to My Workspace only
- Cannot share content with others or collaborate in shared workspaces
- Useful for personal development, learning, and consuming content on F64+ capacities

### Pro

- Publish to shared workspaces
- Share reports and semantic models with other Pro users
- Access to deployment pipelines (limited)
- Required for any developer or content creator in most organizations

### Premium Per User (PPU)

- All Pro features plus most Premium capacity features
- Access to paginated reports, larger dataset storage, advanced AI visuals, and full deployment pipelines
- Content can only be consumed by other PPU or Premium/Fabric capacity users
- Best for smaller teams that need Premium features without committing to a full capacity

!!! tip
    If your organization already has Microsoft 365 E5 or certain Azure agreements, Power BI Pro may already be included. Check your Microsoft licensing agreement before purchasing separately.

---

## Capacity Licenses

Capacity licenses are purchased at the organizational level and provide dedicated compute resources shared across users. They unlock features not available on per-user licenses and — at sufficient scale — eliminate the need for individual Pro licenses for report consumers.

### Fabric Capacity (F SKUs)

Microsoft Fabric is Microsoft's unified analytics platform. It includes Power BI Premium features plus Data Engineering, Data Science, Data Warehouse, Real-Time Intelligence, and more — all sharing the same capacity pool.

Fabric capacity is purchased through Azure as F SKUs, measured in Capacity Units (CUs):

| SKU | CUs | Equivalent | Typical use |
|-----|-----|-----------|-------------|
| F2 | 2 | — | Development, testing, small teams |
| F8 | 8 | — | Departmental analytics |
| F32 | 32 | — | Mid-size organizations |
| F64 | 64 | P1 | Production entry point — unlocks free viewer consumption |
| F128 | 128 | P2 | Large organizations, heavy workloads |
| F256 | 256 | P3 | Enterprise scale |

!!! note
    **F64 is the key licensing threshold.** Below F64, every user viewing Power BI content still needs a Pro or PPU license. At F64 and above, report consumers can view content with a free license — only content creators and publishers need Pro.

Two purchasing options are available:

- **Pay-as-you-go** — billed per second (one-minute minimum). Can be paused when not in use. Best for variable or unpredictable workloads.
- **Reserved capacity** — commit to one year for approximately 40% lower cost than pay-as-you-go. Best for stable, always-on production workloads.

### Premium Per Capacity (P SKUs)

!!! warning
    Microsoft is retiring P SKUs and consolidating on F SKUs. New customers should purchase Fabric capacity (F SKUs) instead. Existing P SKU customers should plan migration — P1 maps to F64, P2 to F128, P3 to F256.

P SKUs were the original dedicated capacity model for Power BI. They are still supported for existing customers but are no longer the recommended path for new purchases.

---

## Fabric Free vs Fabric Capacity

This is a common source of confusion:

- **Fabric (Free) user license** — automatically granted when a user signs into Fabric for the first time. It is a *user license*, not capacity. It allows creating and sharing non-Power BI Fabric items when the workspace runs on a paid F or Trial capacity. It does **not** provide free access to Power BI content on smaller capacities.
- **Fabric capacity (F SKU)** — a *capacity license* purchased through Azure. This is what provides the compute resources. Without a paid F SKU, there is no Fabric capacity.

A user with a Fabric Free license can view Power BI content only if the workspace is on an F64 or larger capacity. Below F64, they still need a Pro license.

---

## Trial Licenses

Before committing to a capacity purchase, Microsoft offers trials:

- **PPU trial** — 60-day individual trial. Gives access to most Premium Per User features. Available to any licensed user from their account settings.
- **Fabric capacity trial** — 60-day trial at F64 capacity level. Available at the tenant level. Ideal for evaluating Fabric workloads before purchasing.

!!! tip
    Use the Fabric trial to run a representative workload and monitor usage with the **Fabric Capacity Metrics app** before choosing an F SKU. This gives real data on CU consumption to size the capacity correctly.

---

## Feature Comparison

| Feature | Free | Pro | PPU | Fabric F64+ |
|---------|------|-----|-----|-------------|
| Power BI Desktop | ✅ | ✅ | ✅ | ✅ |
| Publish to shared workspace | ❌ | ✅ | ✅ | ✅ |
| Share with other users | ❌ | ✅ | ✅ | ✅ |
| Paginated reports | ❌ | ❌ | ✅ | ✅ |
| Deployment pipelines | ❌ | Limited | ✅ | ✅ |
| Larger dataset storage | ❌ | ❌ | ✅ | ✅ |
| Refresh up to 48×/day | ❌ | ❌ | ✅ | ✅ |
| Free viewer consumption | ❌ | ❌ | ❌ | ✅ |
| Fabric workloads (Lakehouse, Pipelines, Notebooks) | ❌ | ❌ | ❌ | ✅ |
| Git integration | ❌ | ❌ | ❌ | ✅ |
| DirectLake mode | ❌ | ❌ | ❌ | ✅ |

---

## Embedding Scenarios

Power BI content can be embedded in other applications and portals. The required license depends on the audience.

| Scenario | Description | License needed |
|----------|-------------|---------------|
| **Embed in SharePoint** | Internal portal embedding | Pro or PPU for publisher; free viewers if F64+ capacity |
| **Embed in Teams** | Reports as Teams tabs | Pro or PPU |
| **Embed for your organization** | Internal app using Power BI SDK — users authenticate with their own Microsoft account | Pro or PPU per user (or F64+ for free viewers) |
| **Embed for your customers** | External app — users do not have Microsoft accounts or Power BI licenses | Fabric F SKU or Azure Embedded (A SKU) capacity required |

!!! note
    **Embed for your organization** is for internal tools where users are authenticated members of the organization. **Embed for your customers** is for ISVs or public-facing apps where end users are external and have no Power BI license — the app itself holds the capacity, and the developer handles authentication.

---

## License Assignment

User licenses (Pro, PPU) are assigned through the **Microsoft 365 admin center**:

1. Go to [admin.microsoft.com](https://admin.microsoft.com)
2. Navigate to **Users → Active users → select a user → Licenses and apps**
3. Assign or remove Power BI Pro or PPU from the license list

Fabric capacity (F SKUs) is provisioned through the **Azure portal**:

1. Go to [portal.azure.com](https://portal.azure.com)
2. Search for **Microsoft Fabric** and create a new capacity
3. Assign the capacity to a workspace in Power BI Service under **Workspace settings → Premium**

---

## Practical Guidance

- **Small teams (under 20 users)** — Pro licenses for all developers and active users. PPU if paginated reports or advanced features are needed.
- **Growing organizations** — evaluate the F64 break-even point. If Pro licenses for consumers would exceed the F64 cost, capacity is more economical.
- **Large organizations** — Fabric capacity at F64 or above eliminates per-viewer licensing and adds the full Fabric workload portfolio.
- **Paginated reports** — require PPU or Premium/Fabric capacity to publish and share. Development uses the free Power BI Report Builder tool.
- **Fabric workloads** (Lakehouse, Pipelines, Notebooks, Data Warehouse) — require a paid Fabric F SKU. PPU does not cover these.

---

## Licensing for Paginated Reports

Paginated reports are pixel-perfect, print-ready reports similar to SSRS — ideal for invoices, statements, and operational documents that need exact formatting when exported to PDF.

Requirements:
- **Development** — Power BI Report Builder (free desktop tool)
- **Publish and share** — PPU or Fabric/Premium capacity
- **Consumption** — same license as the capacity or PPU requirement