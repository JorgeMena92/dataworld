# Jorge Mena — Data Engineering Notes

Personal site built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).  
Covers SQL, Power BI, Databricks, and data engineering in general.

🌐 **Live site:** https://jorgemena.github.io

---

## Local Development

**Requirements:** Python 3.9+

```bash
# Install dependencies
pip install mkdocs-material mkdocs-minify-plugin

# Serve locally with live reload
mkdocs serve
```

Then open http://127.0.0.1:8000

---

## Deploy to GitHub Pages

```bash
mkdocs gh-deploy
```

This builds the site and pushes it to the `gh-pages` branch automatically.

---

## Adding Content

1. Create a `.md` file inside the relevant folder (`docs/sql/`, `docs/powerbi/`, etc.)
2. Add frontmatter at the top:

```yaml
---
title: Window Functions
description: How to use OVER(), PARTITION BY, and frame specs in SQL
tags: [sql, analytics]
---
```

3. Uncomment the corresponding line in the `nav:` section of `mkdocs.yml`

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
    │   └── logo.svg
    ├── sql/
    │   └── index.md
    ├── powerbi/
    │   └── index.md
    ├── databricks/
    │   └── index.md
    ├── projects/
    │   └── index.md
    └── notes/
        └── index.md
```
