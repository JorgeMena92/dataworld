---
title: Microsoft Fabric
description: Microsoft Fabric notes — lakehouse, pipelines, notebooks, warehouse, and real-time intelligence
---

# Microsoft Fabric

Practical Fabric notes organized by workload — from platform fundamentals to lakehouse architecture, Spark notebooks, pipelines, and Direct Lake semantic models. Built from real project experience on top of existing Power BI and SQL knowledge.

Pages follow the data flow order — how data moves through the platform from ingestion to consumption. Fundamentals first, then storage, transformation, serving, real-time, and finally how Power BI connects on top.

<div class="card-grid">

  <a href="fundamentals/" class="card">
    <div class="card__title">Fundamentals</div>
    <div class="card__desc">What Fabric is, OneLake, workspaces, capacities, and the medallion architecture.</div>
  </a>

  <a href="lakehouse/" class="card">
    <div class="card__title">Lakehouse</div>
    <div class="card__desc">Delta Lake storage, SQL analytics endpoint, shortcuts, and table management.</div>
  </a>

  <a href="pipelines/" class="card">
    <div class="card__title">Pipelines</div>
    <div class="card__desc">Data Factory pipelines, activities, orchestration, and scheduling patterns.</div>
  </a>

  <a href="notebooks/" class="card">
    <div class="card__title">Notebooks</div>
    <div class="card__desc">Spark notebooks, PySpark patterns, and lakehouse integration.</div>
  </a>

  <a href="warehouse/" class="card">
    <div class="card__title">Warehouse</div>
    <div class="card__desc">T-SQL warehouse, SQL analytics endpoint, and cross-lakehouse queries.</div>
  </a>

  <a href="real-time/" class="card">
    <div class="card__title">Real-Time Intelligence</div>
    <div class="card__desc">Eventstream, KQL databases, and real-time analytics patterns.</div>
  </a>

  <a href="direct-lake/" class="card">
    <div class="card__title">Direct Lake</div>
    <div class="card__desc">Direct Lake mode, semantic models on OneLake, and performance considerations.</div>
  </a>

  <a href="deployment/" class="card">
    <div class="card__title">Deployment & ALM</div>
    <div class="card__desc">Git integration, deployment pipelines, and CI/CD across Fabric environments.</div>
  </a>

</div>