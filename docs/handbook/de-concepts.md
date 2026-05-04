---
title: Data Engineering Competency Map
description: Core knowledge areas and skills for data engineering practice
tags: [handbook, data-engineering, skills]
---

# Data Engineering Competency Map

---

## Core Ingestion & Movement Patterns

- Full load vs Incremental load
- Change Data Capture (CDC)
- Batch processing vs Streaming (real-time)
- ETL vs ELT
- Data replication / synchronization
- Event-driven ingestion
- Idempotency in ingestion

---

## Change Tracking Concepts

- Watermarking (e.g., max date / ID)
- Slowly Changing Dimensions (SCD) (Type 1, Type 2, etc.)
- Upserts / MERGE patterns
- Soft deletes vs hard deletes
- Bi-temporal tracking (valid time vs transaction time)

---

## Transformation Concepts

- Data cleansing (null handling, deduplication)
- Schema transformation / mapping
- Joins and aggregations
- Normalization / denormalization
- Derived columns / business rules
- Data type casting and standardization
- Surrogate key generation

---

## Storage & Loading Strategies

- Medallion architecture (Bronze / Silver / Gold)
- Partitioning strategies
- File formats (columnar, row-based, semi-structured)
- Compression strategies
- Indexing
- Small files problem & optimization
- Table types (managed vs external)
- Write modes (overwrite, append, merge)

---

## Orchestration & Control

- Pipeline orchestration (dependencies, sequencing)
- Scheduling (time-based triggers)
- Parameterization
- Retry policies / error handling
- Parallelism and performance tuning
- Idempotency patterns (safe re-runs)
- Dead letter queues / poison message handling
- Cross-system pipeline dependencies
- Event-based triggers vs scheduled triggers

---

## Data Quality & Validation

- Validation rules (nulls, ranges, formats)
- Data profiling
- Anomaly detection
- Reconciliation (source vs target checks)
- Row count / checksum validation
- Completeness, accuracy, consistency checks
- Quality gates (blocking vs non-blocking)

---

## Testing & Observability

- Unit testing pipeline logic
- Integration testing (end-to-end pipeline tests)
- Data observability (monitoring data health over time)
- Alerting & SLA tracking
- Pipeline logging strategies
- Data freshness monitoring
- Schema drift detection

---

## CI/CD & Deployment (DataOps)

- Source control for pipelines
- Environment promotion (dev → staging → prod)
- Infrastructure as Code (IaC) for data resources
- Deployment pipelines for data artifacts
- Feature branching for data pipelines
- Rollback strategies

---

## Metadata & Lineage

- Data lineage (column-level and table-level)
- Impact analysis
- Schema evolution
- Data cataloging
- Business glossary
- Data ownership and stewardship

---

## Real-Time & Streaming Concepts

- Event streams
- Windowing (tumbling, sliding, session windows)
- Stream processing transformations
- Late-arriving data handling
- Exactly-once vs at-least-once delivery semantics
- Watermarking in streaming (event time vs processing time)
- Stream-to-batch (Lambda / Kappa architecture patterns)

---

## Security & Governance

- Access control during ingestion (RBAC / ABAC)
- Data masking & anonymization
- Sensitivity labels & classification
- Audit logs
- Encryption at rest vs in transit
- Data residency / sovereignty
- Row-level and column-level security

---

## Cost & Resource Management

- Compute optimization
- Storage cost management
- Query performance vs cost tradeoffs
- Reserved vs on-demand capacity
- Pipeline efficiency (reducing unnecessary runs)
- Data lifecycle management (archiving, tiering, deletion)