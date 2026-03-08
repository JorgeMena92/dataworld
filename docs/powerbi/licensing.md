---
title: Licensing
description: Power BI license types, capacity, and cost considerations
tags: [powerbi, licensing]
---

# Licensing

Understanding Power BI licensing is essential for planning deployments, managing costs, and knowing what features are available to your team.

---

## License Types

### Free
- Access to Power BI Desktop (always free)
- Publish to My Workspace only
- Cannot share content with others
- Useful for personal development and learning

### Pro
- Publish to shared workspaces
- Share reports and datasets with other Pro users
- Access to deployment pipelines (limited)
- Required for collaboration in most organizations
- ~$10/user/month

### Premium Per User (PPU)
- All Pro features plus Premium features
- Access to advanced AI visuals, larger datasets, paginated reports, and deployment pipelines
- Content can only be consumed by other PPU or Premium users
- ~$20/user/month

### Premium Per Capacity (P SKUs)
- Dedicated cloud capacity — not tied to individual users
- End users can consume content with a **free license**
- Suitable for large-scale deployments with many viewers
- Starts at ~$5,000/month (P1)
- Includes paginated reports, AI features, and higher refresh rates

### Fabric (F SKUs)
- Microsoft's unified analytics platform
- Includes Power BI Premium features plus Data Engineering, Data Science, and Data Warehouse workloads
- Replaces P SKUs for new purchases
- Priced by Capacity Units (CUs)

---

## Feature Comparison

| Feature | Free | Pro | PPU | Premium/Fabric |
|---|---|---|---|---|
| Power BI Desktop | ✅ | ✅ | ✅ | ✅ |
| Publish to shared workspace | ❌ | ✅ | ✅ | ✅ |
| Share with other users | ❌ | ✅ | ✅ | ✅ |
| Paginated reports | ❌ | ❌ | ✅ | ✅ |
| Deployment pipelines | ❌ | Limited | ✅ | ✅ |
| Larger dataset storage | ❌ | ❌ | ✅ | ✅ |
| Refresh up to 48x/day | ❌ | ❌ | ✅ | ✅ |
| Free viewer consumption | ❌ | ❌ | ❌ | ✅ |
| AI visuals | ❌ | ❌ | ✅ | ✅ |

---

## Embedding Scenarios

| Scenario | License needed |
|---|---|
| Embed in SharePoint (internal) | Pro or PPU for publisher, free for viewers if Premium capacity |
| Embed in Teams | Pro or PPU |
| Embed in external app (Embed for customers) | Premium or Fabric capacity |
| Embed in internal app (Embed for organization) | Pro or PPU per user |

---

## Practical Guidance

- **Small teams (< 20 users)** — Pro licenses for developers and active users, free for occasional viewers if Premium capacity is available
- **Large organizations** — Premium Per Capacity or Fabric is more cost-effective when you have many viewers
- **Report Builder / Paginated Reports** — requires PPU or Premium
- **Fabric workloads** (Lakehouse, Pipelines, Notebooks) — require Fabric capacity

!!! tip
    If your organization already has Microsoft 365 E5 or certain Azure agreements, Power BI Pro may be included. Check your Microsoft licensing agreement before purchasing separately.

---

## Licensing for Report Builder (Paginated Reports)

Paginated reports (SSRS-style, pixel-perfect, print-ready) require:
- **PPU** or **Premium/Fabric** capacity to publish and share
- Power BI Report Builder (free desktop tool) for development

These are ideal for invoices, statements, operational reports, and any content that needs to be exported to PDF with exact formatting.
