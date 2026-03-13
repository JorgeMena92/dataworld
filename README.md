# Jorge Mena — Data World

Personal knowledge base built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).  
Covers SQL, Power BI, and data engineering patterns — combining theoretical foundations with practical experience from real projects.

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
git commit -m "v2.0.0 - description"
git tag v2.0.0
git push origin main --tags
python -m mkdocs gh-deploy
```

---

## Adding Content

1. Create a `.md` file inside the relevant folder (`docs/sql/`, `docs/powerbi/`, `docs/handbook/`, etc.)
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
    │   ├── indexes-ddl.md
    │   ├── naming-conventions.md
    │   ├── schema-design.md
    │   ├── views.md
    │   ├── materialized-views.md
    │   ├── functions.md
    │   ├── stored-procedures.md
    │   ├── triggers.md
    │   ├── query-optimization.md
    │   ├── indexes.md
    │   ├── execution-plans.md
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
    │   ├── data-modeling.md
    │   ├── dax.md
    │   ├── report-design.md
    │   ├── power-query.md
    │   ├── deployment.md
    │   └── licensing.md
    └── handbook/
        ├── index.md
        ├── references.md
        ├── data-roles.md
        └── layered-fw.md
```

---

## Changelog

| Version | Description |
|---------|-------------|
| v2.0.0 | SQL section fully rebuilt — ANSI-first, 50+ pages across DQL, DML, DDL, Database Objects, Performance, and SQL Patterns |
| v1.3.0 | Handbook: Layered Data Platform Support Framework |
| v1.2.0 | Handbook section: Data Roles, References |
| v1.1.0 | Power BI section complete |
| v1.0.0 | Initial release — SQL section |