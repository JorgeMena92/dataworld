# Jorge Mena — Data Engineering Notes

Personal site built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).  
Covers SQL, Power BI, Databricks, and data engineering in general.

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
git commit -m "v1.0.0 - first public release"
git tag v1.0.0
git push origin main --tags
python -m mkdocs gh-deploy
```

---

## Adding Content

1. Create a `.md` file inside the relevant folder (`docs/sql/`, `docs/powerbi/`, etc.)
2. Add frontmatter at the top:

```yaml
---
title: Window Functions
description: How to use OVER(), PARTITION BY, and frame specs in SQL
tags: [sql, analytics]
hide:
  - toc
  - navigation
---
```

3. Add the file to the `nav:` section of `mkdocs.yml`

---

## Project Structure

```
.
├── mkdocs.yml
├── overrides/
│   └── main.html           # custom nav + footer
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