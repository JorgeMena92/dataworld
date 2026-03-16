---
title: Monitoring & Alerting
description: What to monitor, how to set up alerts, and how to interpret signals across a Microsoft Azure data platform.
tags: [operations, monitoring, alerting, azure]
---

# Monitoring & Alerting

Effective monitoring is the difference between finding out about a problem from a user and finding it yourself before anyone notices. In a data platform built on Power BI, Azure Databricks, ADF, and Fabric, monitoring spans multiple services — each with its own signals, alert mechanisms, and failure patterns.

This section covers what to monitor across the platform, how to configure alerts, and how to interpret signals during an incident.

---

## Coverage Plan

| Topic | Status |
|-------|--------|
| Power BI — Refresh monitoring and gateway health | Coming soon |
| Azure Databricks — Job alerts and cluster monitoring | Coming soon |
| Azure Data Factory — Pipeline monitoring and alert rules | Coming soon |
| Microsoft Fabric — Capacity monitoring and pipeline health | Coming soon |
| Azure Monitor — Unified alerting across platform services | Coming soon |
| ADLS — Storage diagnostics and access monitoring | Coming soon |

---

## Monitoring Philosophy

A well-monitored data platform surfaces three categories of signal:

**Availability** — is the component running? Did the job complete? Did the refresh succeed?

**Correctness** — is the data arriving on time? Are volumes within expected ranges? Are there unexpected nulls or duplicates?

**Performance** — is the job taking longer than usual? Is the cluster under-resourced? Is a query regressing?

Most out-of-the-box monitoring covers availability. Correctness and performance monitoring typically require custom rules, data quality checks, or platform-specific configuration — and they catch the issues that availability monitoring misses entirely.

!!! note "This section is growing"
    Pages covering each platform component will be added progressively. The monitoring topics align directly with the layers in the [Layered Triage Framework](../support/layered-fw.md) — each layer has observable signals that can be monitored proactively.
