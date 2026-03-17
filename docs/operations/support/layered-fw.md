---
title: Layered Triage
description: Work from the presentation layer inward to the data source, then check infrastructure and security. Eliminate each layer before proceeding to the next.
---

# Layered Triage

Work from the **presentation layer inward** to the data source, then check infrastructure and security. Eliminate each layer before proceeding to the next.

---

## Context

This framework applies to any modern cloud data platform — regardless of the specific tools used. It was originally designed for a **Microsoft Azure** stack (Power BI, Databricks, ADF, Fabric, ADLS Gen2) and some examples reflect that context, but the layer structure and troubleshooting logic transfer to any equivalent platform.

The seven layers cover the full stack from what the end user sees down to infrastructure and security — each one a potential root cause for data incidents, refresh failures, or access issues.

---

## Troubleshooting Flow


<div class="incident-flow">

  <div class="flow-step flow-step--start">
    <div class="flow-step__icon">🔍</div>
    <div class="flow-step__body">
      <div class="flow-step__title">Incident Detected</div>
      <div class="flow-step__sub">An incident has been detected</div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">1</div>
    <div class="flow-step__body">
      <div class="flow-step__title">📊 Presentation Layer</div>
      <div class="flow-step__sub">Visuals · Reports · Dashboards</div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">2</div>
    <div class="flow-step__body">
      <div class="flow-step__title">🗂️ Semantic Model Layer</div>
      <div class="flow-step__sub">Measures · Relationships · Filters</div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">3</div>
    <div class="flow-step__body">
      <div class="flow-step__title">⚙️ Transformation Layer</div>
      <div class="flow-step__sub">Data Logic · Ingestion Pipelines</div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">4</div>
    <div class="flow-step__body">
      <div class="flow-step__title">🔄 Pipeline / Orchestration Layer</div>
      <div class="flow-step__sub">Jobs · Triggers · Scheduling</div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">5</div>
    <div class="flow-step__body">
      <div class="flow-step__title">🗄️ Source System Layer</div>
      <div class="flow-step__sub">APIs · Databases · External Feeds</div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">6</div>
    <div class="flow-step__body">
      <div class="flow-step__title">☁️ Infrastructure / Platform Layer</div>
      <div class="flow-step__sub">Compute · Networking · Services</div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step">
    <div class="flow-step__icon">7</div>
    <div class="flow-step__body">
      <div class="flow-step__title">🔒 Security / Access Layer</div>
      <div class="flow-step__sub">Permissions · Identity · Access Control</div>
    </div>
  </div>
  <div class="flow-arrow">↓</div>

  <div class="flow-step flow-step--alert">
    <div class="flow-step__icon">⚠️</div>
    <div class="flow-step__body">
      <div class="flow-step__title">Root Cause Identified</div>
      <div class="flow-step__sub">Confirm scope and impact</div>
    </div>
  </div>

</div>


## 1 · Presentation Layer

*Start here — this is what the user sees. Eliminate visual and report-level issues before going deeper.*

- Check report visuals for incorrect data, blank results, or rendering errors
- Validate report-level, page-level, and visual-level filters for unintended overrides
- Inspect filter controls and cross-filter behaviour between visuals
- Confirm correct fields and measures are bound to the right visuals
- Test row-level security behaviour across different user profiles
- Verify data refresh status — last successful run, failure message

---

## 2 · Semantic Model Layer

*Covers the dataset as a whole — measures, relationships, and filtering context.*

- Validate measures and calculated columns for logic regressions or context errors
- Inspect table relationships — cardinality, cross-filter direction, ambiguous paths
- Check filtering logic — bidirectional filters, inactive relationship overrides
- Review data preparation steps for errors or broken dependencies
- Verify data types and schema consistency (type mismatch = silent error)
- Check incremental refresh configuration and partition boundaries

---

## 3 · Transformation Layer

*Where data is shaped, cleaned, and prepared. Covers both SQL logic and processing transformations.*

- Review transformation logic for recent changes or regressions
- Inspect transformation scripts and processing jobs — execution logs, stack traces
- Validate ingestion pipelines — source extraction, landing zone delivery, file formats
- Check joins and aggregations for fan-out, duplicates, or data loss
- Verify null handling, data cleaning rules, and edge case coverage
- Confirm business rule implementation matches current specification
- Check for schema drift or structural changes in upstream tables

---

## 4 · Pipeline / Orchestration Layer

*How and when transformations run. Covers job execution, dependencies, and triggers.*

- Check pipeline execution status in orchestration tool
- Review job run logs — focus on the root error, not downstream cascade failures
- Inspect job dependency graph — identify the specific failing task
- Validate job dependencies — missing upstream output blocks all downstream steps
- Check scheduling and trigger configuration for missed or duplicated runs
- Verify upstream job completion and output availability before next stage
- Analyze abnormal job durations — data skew, resource contention, cold cluster start

---

## 5 · Source System Layer

*The origin of all data. Covers availability, schema, volume, and quality at the source.*

- Verify source system availability — APIs, relational databases, file shares, SFTP
- Check external data feed delivery status and arrival timestamps
- Test API responses — status codes, payload structure, authentication
- Validate source schema changes — new columns, renamed fields, dropped tables
- Inspect record volume anomalies — sudden drop or spike vs. historical baseline
- Confirm source credentials, connection strings, and firewall / IP allowlist rules
- Check source data quality — duplicates, nulls, encoding issues, unexpected values

---

## 6 · Infrastructure / Platform Layer

*The environment everything runs on. Check this layer to rule out platform-level failures.*

- Check platform status page and service health dashboard for active incidents
- Verify data gateway or connector service status and data source bindings
- Inspect compute cluster health — state, driver logs, autoscaling behaviour
- Validate compute capacity and runtime throttling
- Check network peering, private endpoints, DNS resolution
- Inspect authentication tokens and service account / managed identity expiry
- Verify object storage availability and tier configuration

---

## 7 · Security / Access Layer

*Cross-cutting layer — permission and credential issues can surface at any layer above.*

- Check workspace roles and report-level sharing settings
- Validate data asset access permissions and classification or sensitivity restrictions
- Verify service account credentials, secret or token expiry, and assigned roles
- Inspect storage container access policies
- Validate row-level security — role definitions, filter expressions, user membership
- Confirm identity provider group memberships and RBAC role assignments for affected users or service accounts
- Check data platform permissions and catalog grants