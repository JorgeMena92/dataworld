# Jorge Mena — Data World

Personal knowledge base built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).  
Covers SQL, Power BI, Azure Databricks, and data engineering patterns — combining theoretical foundations with practical experience from real projects.

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
git commit -m "v1.3.0 - description"
git tag v1.3.0
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
tags: [sql, analytics]
---
```

3. Add the file to the `nav:` section of `mkdocs.yml`

> **Note:** `hide: toc` and `hide: navigation` are not needed — both are handled globally via CSS.

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
    │   ├── ddl.md
    │   ├── dml.md
    │   ├── dcl.md
    │   ├── ansi-sql.md
    │   ├── fundamentals.md
    │   ├── basic.md
    │   ├── intermediate.md
    │   ├── advanced.md
    │   └── ansi-features.md
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
| v1.3.0 | Handbook: Layered Data Platform Support Framework |
| v1.2.0 | Handbook section: Data Roles, References |
| v1.1.0 | Power BI section complete |
| v1.0.0 | Initial release — SQL section |