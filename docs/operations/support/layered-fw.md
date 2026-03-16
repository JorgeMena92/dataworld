---
title: Layered Data Platform Support Framework
description: Work from the presentation layer inward to the data source, then check infrastructure and security. Eliminate each layer before proceeding to the next.
---

# Layered Triage

Work from the **presentation layer inward** to the data source, then check infrastructure and security. Eliminate each layer before proceeding to the next.

## Context

This framework applies to any modern cloud data platform — regardless of the specific tools used. It was originally designed for a **Microsoft Azure** stack (Power BI, Databricks, ADF, Fabric, ADLS Gen2) and some examples reflect that context, but the layer structure and troubleshooting logic transfer to any equivalent platform.

The seven layers cover the full stack from what the end user sees down to infrastructure and security — each one a potential root cause for data incidents, refresh failures, or access issues.

## Troubleshooting Flow

<div class="fw-diagram">

  <div class="fw-incident">
    <span class="fw-icon">🔍</span>
    <div>
      <div class="fw-incident-title">Incident Detected</div>
      <div class="fw-incident-sub">Start from the presentation layer and work inward</div>
    </div>
  </div>

  <div class="fw-connector"></div>

  <div class="fw-layers">

    <details class="fw-layer">
      <summary class="fw-layer-head">
        <span class="fw-info">
          <span class="fw-name">📊 1. Presentation Layer</span>
          <span class="fw-sub">Visuals · Reports · Dashboards</span>
        </span>
      </summary>
      <div class="fw-body">
        <ul>
          <li>Check report visuals for incorrect data, blank results, or rendering errors</li>
          <li>Validate report-level, page-level, and visual-level filters for unintended overrides</li>
          <li>Inspect filter controls and cross-filter behaviour between visuals</li>
          <li>Confirm correct fields and measures are bound to the right visuals</li>
          <li>Test row-level security behaviour across different user profiles</li>
          <li>Verify data refresh status — last successful run, failure message</li>
        </ul>
      </div>
    </details>

    <details class="fw-layer">
      <summary class="fw-layer-head">
        <span class="fw-info">
          <span class="fw-name">🗂️ 2. Semantic Model Layer</span>
          <span class="fw-sub">Measures · Relationships · Filters</span>
        </span>
      </summary>
      <div class="fw-body">
        <ul>
          <li>Validate measures and calculated columns for logic regressions or context errors</li>
          <li>Inspect table relationships — cardinality, cross-filter direction, ambiguous paths</li>
          <li>Check filtering logic — bidirectional filters, inactive relationship overrides</li>
          <li>Data preparation steps for errors or broken dependencies</li>
          <li>Verify data types and schema consistency (type mismatch = silent error)</li>
          <li>Check incremental refresh configuration and partition boundaries</li>
        </ul>
      </div>
    </details>

    <details class="fw-layer">
      <summary class="fw-layer-head">
        <span class="fw-info">
          <span class="fw-name">⚙️ 3. Transformation Layer</span>
          <span class="fw-sub">Data Logic · Ingestion Pipelines</span>
        </span>
      </summary>
      <div class="fw-body">
        <ul>
          <li>Review SQL transformation logic for recent changes or regressions</li>
          <li>Inspect transformation scripts and processing jobs — execution logs, stack traces</li>
          <li>Validate ingestion pipelines — source extraction, landing zone delivery, file formats</li>
          <li>Check joins and aggregations for fan-out, duplicates, or data loss</li>
          <li>Verify null handling, data cleaning rules, and edge case coverage</li>
          <li>Confirm business rule implementation matches current specification</li>
          <li>Check for schema drift or structural changes in upstream tables</li>
        </ul>
      </div>
    </details>

    <details class="fw-layer">
      <summary class="fw-layer-head">
        <span class="fw-info">
          <span class="fw-name">🔄 4. Pipeline / Orchestration Layer</span>
          <span class="fw-sub">Jobs · Triggers · Orchestration</span>
        </span>
      </summary>
      <div class="fw-body">
        <ul>
          <li>Check pipeline execution status in orchestration tool</li>
          <li>Review job run logs — focus on the root error, not downstream cascade failures</li>
          <li>Inspect job dependency graph — identify the specific failing task</li>
          <li>Validate job dependencies — missing upstream output blocks all downstream steps</li>
          <li>Check scheduling and trigger configuration for missed or duplicated runs</li>
          <li>Verify upstream job completion and output availability before next stage</li>
          <li>Analyze abnormal job durations — data skew, resource contention, cold cluster start</li>
        </ul>
      </div>
    </details>

    <details class="fw-layer">
      <summary class="fw-layer-head">
        <span class="fw-info">
          <span class="fw-name">🗄️ 5. Source System Layer</span>
          <span class="fw-sub">APIs · Databases · External Feeds</span>
        </span>
      </summary>
      <div class="fw-body">
        <ul>
          <li>Verify source system availability — APIs, relational databases, file shares, SFTP</li>
          <li>Check external data feed delivery status and arrival timestamps</li>
          <li>Test API responses — status codes, payload structure, authentication</li>
          <li>Validate source schema changes — new columns, renamed fields, dropped tables</li>
          <li>Inspect record volume anomalies — sudden drop or spike vs. historical baseline</li>
          <li>Confirm source credentials, connection strings, and firewall / IP allowlist rules</li>
          <li>Check source data quality — duplicates, nulls, encoding issues, unexpected values</li>
        </ul>
      </div>
    </details>

    <details class="fw-layer">
      <summary class="fw-layer-head">
        <span class="fw-info">
          <span class="fw-name">☁️ 6. Infrastructure / Platform Layer</span>
          <span class="fw-sub">Compute · Networking · Platform Services</span>
        </span>
      </summary>
      <div class="fw-body">
        <ul>
          <li>Check platform status page and service health dashboard for active incidents</li>
          <li>Verify data gateway or connector service status and data source bindings</li>
          <li>Inspect compute cluster health — state, driver logs, autoscaling behaviour</li>
          <li>Validate Azure compute capacity — compute capacity and runtime throttling</li>
          <li>Check network peering, private endpoints, DNS resolution</li>
          <li>Inspect authentication tokens and service account / managed identity expiry</li>
          <li>Verify object storage availability and tier configuration and tier configuration</li>
        </ul>
      </div>
    </details>

    <details class="fw-layer">
      <summary class="fw-layer-head">
        <span class="fw-info">
          <span class="fw-name">🔒 7. Security / Access Layer</span>
          <span class="fw-sub">Permissions · Identity · Access Control</span>
        </span>
      </summary>
      <div class="fw-body">
        <ul>
          <li>Check workspace roles and report-level sharing settings</li>
          <li>Validate data asset access permissions and classification or sensitivity restrictions</li>
          <li>Verify service account credentials, secret or token expiry, and assigned roles</li>
          <li>Inspect storage container access policies</li>
          <li>Validate row-level security — role definitions, filter expressions, user membership</li>
          <li>Confirm identity provider group memberships and RBAC role assignments for affected users or service accounts</li>
          <li>Check data platform permissions and catalog grants</li>
        </ul>
      </div>
    </details>

  </div>

  <div class="fw-connector"></div>

  <div class="fw-resolved">
    <span class="fw-icon">⚠️</span>
    <div>
      <div class="fw-resolved-title">Root Cause Identified</div>
      <div class="fw-resolved-sub">Eliminate each layer before proceeding to the next</div>
    </div>
  </div>

</div>

<style>
.fw-diagram {
  margin: 1.5rem 0;
  font-family: var(--md-text-font, "DM Sans", sans-serif);
}

.fw-incident, .fw-resolved {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 16px;
  border-radius: 10px;
  border: 1.5px solid;
}
.fw-incident {
  background: var(--md-default-bg-color, #f5f5f5);
  border-color: var(--brand-border, #e0e0e0);
}
.fw-resolved {
  background: #fdf0f0;
  border-color: #e08080;
}
[data-md-color-scheme="slate"] .fw-resolved {
  background: #3b1f1f;
  border-color: #6b3030;
}

.fw-icon { font-size: 1.1rem; flex-shrink: 0; }

.fw-incident-title, .fw-resolved-title {
  font-size: 0.85rem;
  font-weight: 600;
}
.fw-incident-title { color: var(--brand-ink, #1c1c1e); }
.fw-resolved-title { color: #922b21; }
[data-md-color-scheme="slate"] .fw-resolved-title { color: #f5a0a0; }


.fw-incident-sub, .fw-resolved-sub {
  font-size: 0.75rem;
  margin-top: 1px;
  color: var(--brand-muted, #7a7a7a);
}

.fw-layers {
  display: flex;
  flex-direction: column;
}

.fw-layer {
  border-left: 1.5px solid var(--brand-border, #e0e0e0);
  border-right: 1.5px solid var(--brand-border, #e0e0e0);
  border-bottom: 1px solid var(--brand-border, #e0e0e0);
  border-top: none;
  background: #ffffff;
}
.fw-layer:first-child {
  border-top: 1.5px solid var(--brand-border, #e0e0e0);
  border-radius: 10px 10px 0 0;
}
.fw-layer:last-child {
  border-radius: 0 0 10px 10px;
  border-bottom: 1.5px solid var(--brand-border, #e0e0e0);
}
[data-md-color-scheme="slate"] .fw-layer {
  background: #242424;
  border-color: #2a2a2a;
}

.fw-layer-head {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 14px;
  cursor: pointer;
  list-style: none;
  user-select: none;
}
.fw-layer-head::-webkit-details-marker { display: none; }
.fw-layer-head:hover { background: var(--brand-mint-glow, rgba(60,179,113,0.06)); }

.fw-num {
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: var(--brand-mint-dark, #2d8a57);
  color: #fff;
  font-size: 0.7rem;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.fw-info {
  display: flex;
  flex-direction: column;
  flex: 1;
}

.fw-name {
  font-size: 0.82rem;
  font-weight: 600;
  color: var(--brand-mint-dark, #2d8a57);
}
[data-md-color-scheme="slate"] .fw-name { color: var(--brand-mint, #3cb371); }

.fw-sub {
  font-size: 0.72rem;
  color: var(--brand-ink-soft, #3a3a3c);
  margin-top: 1px;
}
[data-md-color-scheme="slate"] .fw-sub { color: var(--brand-muted, #606060); }

.fw-body {
  padding: 6px 16px 12px 6px;
  border-top: 1px solid var(--brand-border, #e0e0e0);
}
[data-md-color-scheme="slate"] .fw-body {
  border-top-color: #2a2a2a;
}

.fw-body ul {
  margin: 4px 0 0;
  padding-left: 0.5rem;
  display: flex;
  flex-direction: column;
  gap: 3px;
}
.fw-body ul li {
  font-size: 0.78rem;
  color: var(--brand-ink, #1c1c1e);
  line-height: 1.5;
}
[data-md-color-scheme="slate"] .fw-body ul li { color: var(--brand-ink-soft, #a0b8d0); }


[data-md-color-scheme="slate"] .fw-incident-sub,
[data-md-color-scheme="slate"] .fw-resolved-sub {
  color: #f0f0f0;
}




</style>


