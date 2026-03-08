---
title: Report Design
description: Bookmarks, drill through, KPIs, themes, and layout best practices
tags: [powerbi, report, design]
---

# Report Design

Good report design is not just about aesthetics — it directly affects how users understand and interact with data. A well-designed report communicates clearly and requires minimal explanation.

---

## Layout Principles

- **One message per page** — each report page should answer one business question
- **Top-left to bottom-right** — users read in this direction; place KPIs and filters at the top
- **Group related visuals** — use whitespace and alignment to create visual hierarchy
- **Limit visuals per page** — 5 to 7 visuals maximum; more creates cognitive overload
- **Consistent spacing** — use the alignment and distribution tools in Power BI Desktop

---

## KPIs and Summary Cards

Place the most important numbers at the top of the page as card visuals or KPI visuals. Users should be able to answer the key question within 5 seconds of opening the report.

```
[ Total Sales ]   [ Total Orders ]   [ Avg Ticket ]   [ YoY Growth ]
        ↓ supporting charts and breakdowns below
```

---

## Slicers and Filters

- Use slicers for dimensions users interact with frequently (date, region, category)
- Use the filter pane for technical or rarely-used filters
- Sync slicers across pages when the same filter should apply everywhere
- Place slicers consistently — same position on every page

---

## Bookmarks

Bookmarks capture the state of a report page — filters, visibility, and selections. Use them to:

- Create navigation buttons between views
- Show/hide visuals dynamically (toggle between chart and table)
- Build guided storytelling flows

```
Button "Show Table" → Bookmark A (table visible, chart hidden)
Button "Show Chart" → Bookmark B (chart visible, table hidden)
```

---

## Drill Through

Drill through allows users to right-click a data point and navigate to a detail page filtered to that context.

Setup:
1. Create a detail page
2. Add the drill through field to the **Drill through** well on that page
3. Power BI automatically adds a back button

Use drill through for:
- Product detail from a sales summary
- Customer profile from a customer list
- Order detail from an order summary

---

## Tooltips

Custom tooltip pages show additional context when a user hovers over a visual.

Setup:
1. Create a new page
2. Set page type to **Tooltip** in page settings
3. Assign the tooltip page to a visual in its format options

---

## Themes

Themes define the default colors, fonts, and visual formatting across the entire report. Always create a custom theme to match company branding.

A theme is a JSON file with this structure:

```json
{
  "name": "Company Theme",
  "dataColors": ["#2d8a57", "#3cb371", "#1c1c1e"],
  "background": "#f5f5f5",
  "foreground": "#1c1c1e",
  "tableAccent": "#3cb371"
}
```

Import it in Power BI Desktop via **View → Themes → Browse for themes**.

---

## Best Practices

- Use a consistent color palette — 2 to 3 primary colors maximum
- Avoid 3D charts, pie charts with many slices, and dual-axis charts
- Use conditional formatting to highlight exceptions, not decoration
- Always label axes and provide context in titles
- Test the report on the actual screen size it will be used on
- Remove gridlines and borders when they add noise without value
