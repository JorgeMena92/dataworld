# Jorge Mena — Data World

Personal knowledge base built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).  
Covers SQL, Power BI, data engineering patterns, and platform operations — combining theoretical foundations with practical experience from real projects.

🌐 **Live site:** https://jorgemena92.github.io/dataworld/

---

## Local Development

**Requirements:** Python 3.9+

```bash
# Install dependencies
pip install mkdocs-material mkdocs-minify-plugin

# Serve locally with live reload
python -m mkdocs serve --watch docs/stylesheets/
```

Then open http://127.0.0.1:8000

---

## Deploy to GitHub Pages

```bash
python -m mkdocs gh-deploy
```

This builds the site and pushes it to the `gh-pages` branch automatically.

> **Note:** Never edit the `gh-pages` branch directly — it is managed by `gh-deploy`.

---

## Versioning

Tag releases before deploying:

```bash
git add .
git commit -m "v2.6.1 - description"
git tag v2.6.1
git push origin main --tags
python -m mkdocs gh-deploy
```

---

## Adding Content

1. Create a `.md` file inside the relevant folder (`docs/sql/`, `docs/powerbi/`, `docs/operations/`, `docs/handbook/`, etc.)
2. Add frontmatter at the top:

```yaml
---
title: Page Title
description: Short description for SEO
tags: [topic, subtopic]
---
```

3. Add the file to the `nav:` section of `mkdocs.yml`

> **Note:** `hide: toc` and `hide: navigation` are not needed — both are handled globally via CSS.

---

## Content Structure

Every page follows the same structure to keep the site consistent and easy to navigate.

### Page Template

```yaml
---
title: Page Title
description: One sentence for SEO and the nav tooltip
tags: [topic, subtopic]
---
```

```markdown
# Page Title

One or two sentences explaining what this topic is and why it matters.

---

## What It Is

Clear definition. No assumed knowledge — explain the concept before showing code or configuration.

---

## How It Works

Core syntax, mechanics, or configuration. Minimal and focused on the essential pattern.

---

## Common Patterns

Real-world variations and use cases — the situations you actually encounter in practice.

---

## Best Practices

Short, actionable rules. What to do, what to avoid, and why.
```

### Content Principles

- **Definition first** — always explain what something is before showing how to use it
- **Standard approach first, platform-specific second** — lead with the canonical or most widely applicable way, then note tool-specific variations
- **Real examples** — use realistic names that reflect actual scenarios (`orders`, `customers`, `sales_report`) — not generic placeholders like `table1` or `col_a`
- **One concept per section** — if a section is getting long, it probably needs its own page
- **Cross-reference related pages** — link to related topics instead of repeating content
- **Use callouts consistently** — `!!! tip` for good practices, `!!! warning` for common mistakes, `!!! note` for platform-specific differences
- **Best practices always last** — short, scannable, actionable

---

## Project Structure

```
.
├── mkdocs.yml
└── docs/
    ├── index.md
    ├── about.md
    ├── stylesheets/
    │   └── extra.css
    ├── assets/
    │   ├── logo.png
    │   └── images/
    │       └── favicon.png
    │       └── star-schema.svg
    ├── sql/
    │   ├── index.md
    │   ├── introduction.md
    │   ├── tools.md
    │   ├── ansi-sql.md
    │   ├── sql-command-categories.md
    │   ├── fundamentals.md
    │   ├── filtering-conditions.md
    │   ├── sorting-limiting.md
    │   ├── data-types-casting.md
    │   ├── scalar-functions.md
    │   ├── aggregations.md
    │   ├── joins.md
    │   ├── set-operations.md
    │   ├── subqueries.md
    │   ├── ctes.md
    │   ├── temporary-tables.md
    │   ├── window-functions.md
    │   ├── dml-fundamentals.md
    │   ├── insert.md
    │   ├── update.md
    │   ├── delete.md
    │   ├── merge.md
    │   ├── transactions.md
    │   ├── bulk-operations.md
    │   ├── incremental-loading.md
    │   ├── ddl-fundamentals.md
    │   ├── create.md
    │   ├── alter.md
    │   ├── drop.md
    │   ├── truncate.md
    │   ├── constraints.md
    │   ├── data-integrity.md
    │   ├── indexes-ddl.md
    │   ├── naming-conventions.md
    │   ├── schema-design.md
    │   ├── views.md
    │   ├── materialized-views.md
    │   ├── functions.md
    │   ├── stored-procedures.md
    │   ├── triggers.md
    │   ├── execution-plans.md
    │   ├── query-optimization.md
    │   ├── indexes.md
    │   ├── partitioning.md
    │   ├── patterns-gaps-islands.md
    │   ├── patterns-running-totals.md
    │   ├── patterns-deduplication.md
    │   ├── patterns-latest-record.md
    │   ├── patterns-top-n-per-group.md
    │   ├── patterns-sessionization.md
    │   ├── patterns-pivot.md
    │   ├── patterns-scd.md
    │   ├── patterns-date-spine.md
    │   └── patterns-idempotent.md
    ├── powerbi/
    │   ├── index.md
    │   ├── fundamentals.md
    │   ├── power-query.md
    │   ├── data-modeling.md
    │   ├── dax.md
    │   ├── dax-iterators.md
    │   ├── dax-time-intelligence.md
    │   ├── dax-patterns.md
    │   ├── report-design.md
    │   ├── deployment.md
    │   └── licensing.md
    ├── operations/
    │   ├── index.md
    │   ├── support/
    │   │   ├── index.md
    │   │   ├── layered-fw.md
    │   │   ├── incident-management.md
    │   │   ├── daily-operations.md
    │   │   ├── incident-tracking.md
    │   │   ├── communication-standards.md
    │   │   └── runbooks/
    │   │       └── index.md
    │   └── monitoring/
    │       └── index.md
    └── handbook/
        ├── index.md
        ├── references.md
        └── data-roles.md
```

---

## Changelog

| Version | Description |
|---------|-------------|
| v2.6.1 | CSS refactor — generic color tokens replacing brand-specific variable names, section numbering fixed, dark mode overrides consolidated |
| v2.6.0 | Power BI section fully reviewed — fundamentals expanded with Fabric intro, Desktop Views, and connection modes; Power Query deepened with M language, parameters, Reference vs Duplicate; Data Modeling with role-playing dimensions, inactive relationships, DAX date table examples; DAX split into four pages (Fundamentals, Iterators, Time Intelligence, Patterns); Report Design expanded with navigation, visual interactions, conditional formatting, slicer panel pattern; Deployment updated with RLS, service principals, Fabric Git, VNet gateway; Licensing rebuilt with F SKU table, P SKU retirement, Fabric Free clarification, trials |
| v2.5.0 | Operations refinements — incident flow HTML/CSS component, collapsible layered triage, runbooks moved inside Support Framework, references page rebuilt with links, CSS cleanup and dark mode consistency pass |
| v2.4.0 | Operations section — Support Framework (Layered Triage, Incident Management, Daily Operations, Incident Tracking, Communication Standards), Runbooks and Monitoring stubs; CSS cleanup and consistency pass |
| v2.3.0 | DDL, Database Objects, Performance, and SQL Patterns sections reviewed — ANSI fixes, cross-references, vendor notes, structural consistency across 29 pages |
| v2.2.0 | DML section reviewed — ANSI fixes across INSERT, UPDATE, DELETE, MERGE, Transactions, Bulk Operations, Incremental Loading; broken relative links fixed |
| v2.1.0 | DQL section reviewed — ANSI fixes, new scalar-functions.md, EXISTS coverage, non-equi joins, named windows, NTH_VALUE; Handbook layered framework generalized |
| v2.0.1 | Handbook: Layered Framework — interactive accordion diagram replacing Mermaid flowchart |
| v2.0.0 | SQL section fully rebuilt — ANSI-first, 50+ pages across DQL, DML, DDL, Database Objects, Performance, and SQL Patterns |
| v1.3.0 | Handbook: Layered Data Platform Support Framework |
| v1.2.0 | Handbook section: Data Roles, References |
| v1.1.0 | Power BI section complete |
| v1.0.0 | Initial release — SQL section |