---
title: Report Design
description: Bookmarks, drill through, KPIs, themes, and layout best practices
tags: [powerbi, report, design]
---

# Report Design

Good report design is not just about aesthetics — it directly affects how users understand and interact with data. A well-designed report communicates clearly, requires minimal explanation, and guides the user to the right answer without friction.

---

## The Report Canvas

A Power BI report is made up of **pages** — each page is an independent canvas containing visuals, slicers, buttons, and shapes. Pages are navigated via tabs at the bottom of the screen in Desktop, or via a page navigator visual in the published report.

Key canvas settings (found in **View → Page view** and **Format page**):

- **Canvas size** — the official default is 16:9 at 1280×720. Change for specific embed targets or mobile layouts.
- **Page background** — set a background color or image per page
- **Wallpaper** — the area outside the canvas; useful for full-bleed backgrounds
- **Page visibility** — hidden pages are not shown in navigation but can still be used for drill through and tooltips

!!! note
    1280×720 is increasingly considered too small for modern monitors. Setting the canvas to 1920×1080 is the practical recommendation for new reports — same 16:9 aspect ratio, significantly more space. A preview feature (as of early 2026) makes 1920×1080 the new default for new pages.

---

## Layout Principles

- **One message per page** — each page should answer one business question
- **Top-left to bottom-right** — users scan in this direction; place KPIs and filters at the top
- **Group related visuals** — use whitespace and alignment to create visual hierarchy
- **Limit visuals per page** — 5 to 7 visuals maximum; more creates cognitive overload
- **Consistent spacing** — use the alignment and distribution tools in Power BI Desktop
- **Consistent positioning** — slicers, titles, and navigation elements should be in the same position on every page
- **Use a three-color palette** — define a primary color (main data, key metrics), a secondary color (supporting elements, comparisons), and a neutral color (backgrounds, borders, labels). Stick to these three across the entire report. Additional colors should only appear for semantic purposes — red for negative, green for positive, amber for warning.

---

## KPIs and Summary Cards

Place the most important numbers at the top of the page as card or KPI visuals. Users should be able to answer the key question within 5 seconds of opening the report.

```
[ Total Sales ]   [ Total Orders ]   [ Avg Ticket ]   [ YoY Growth ]
        ↓ supporting charts and breakdowns below
```

Two visual types for KPIs:

- **Card** — displays a single value. Simple and reliable.
- **KPI visual** — displays a value, a trend, and a target. Requires a target measure and a date axis.

---

## Slicers and Filters

### Slicer types

| Type | Best for |
|------|---------|
| **Dropdown** | Long lists — saves space, expands on click |
| **List** | Short lists — all options visible at once |
| **Between** | Numeric ranges or date ranges with two bounds |
| **Relative date** | Dynamic periods — "last 30 days", "this month" |
| **Tile** | Small sets of discrete options — styled as buttons |

### Slicers vs filter pane

- Use **slicers** for dimensions users interact with frequently — date, region, category, product
- Use the **filter pane** for technical filters, developer-set constraints, or filters users should not change
- Slicers are visible and feel interactive; the filter pane is secondary and often hidden from end users

### Slicer panel pattern

A common UX pattern is to place all slicers inside a collapsible panel — a rectangle with a toggle button that shows and hides it using bookmarks. This keeps the canvas clean while keeping filters accessible.

```
[ ☰ Filters ]  ← button toggles panel visibility
    ┌─────────────────┐
    │ Date:    [____] │  ← panel shown via bookmark
    │ Region:  [____] │
    │ Category:[____] │
    └─────────────────┘
```

### Slicer sync

Use **View → Sync slicers** to apply the same slicer across multiple pages. This ensures that a date or region selection made on one page carries through to all related pages without the user having to reapply it.

---

## Visual Interactions

By default, clicking a data point in one visual filters all other visuals on the page. This behavior can be customized per visual pair using **Format → Edit interactions**.

Each visual can be set to one of three interaction modes with respect to another visual:

- **Filter** — the selected visual filters the target visual (default)
- **Highlight** — the selected visual highlights matching data in the target visual, dimming the rest
- **None** — the selected visual has no effect on the target visual

!!! tip
    Use **None** to prevent KPI cards from being filtered when a user clicks a chart — totals at the top of the page should usually remain unaffected by chart selections. Use **Highlight** instead of **Filter** when you want to show the full picture with context rather than a filtered subset.

!!! warning
    Edit interactions settings are not obvious to end users — if a visual appears unresponsive to selections, check whether its interaction has been set to **None** by a previous developer.

---

## Bookmarks

Bookmarks capture the state of a report page — filter context, visual visibility, slicer selections, and scroll position. They are the foundation of all interactive navigation and show/hide patterns in Power BI.

### What a bookmark captures

Each bookmark can independently capture or ignore three types of state:

| State | What it captures |
|-------|-----------------|
| **Data** | Current filter context and slicer selections |
| **Display** | Visibility of each visual (shown or hidden) |
| **Current page** | Which page is active |

When creating a bookmark, you can choose which of these to include — allowing fine-grained control over what changes when the bookmark is applied.

### Common patterns

**Toggle between chart and table:**
```
Button "Show Table" → Bookmark A (table visible, chart hidden, Data: off)
Button "Show Chart" → Bookmark B (chart visible, table hidden, Data: off)
```
Set **Data: off** so the toggle does not reset the user's filter selections.

**Collapsible slicer panel:**
```
Button "Open Filters"  → Bookmark A (panel visible)
Button "Close Filters" → Bookmark B (panel hidden)
```

**Reset filters button:**
```
Button "Reset" → Bookmark C (all slicers at default, Data: on)
```

### Bookmark groups

Bookmarks can be organized into groups in the Bookmarks panel. This is useful when you have multiple independent toggle patterns on the same page — grouping keeps them organized and prevents accidental conflicts between bookmark sets.

### Report vs personal bookmarks

- **Report bookmarks** — created by the developer, embedded in the report, visible to all users
- **Personal bookmarks** — created by end users in the Power BI Service to save their own view. Requires the feature to be enabled in report settings.

---

## Navigation

Power BI provides several ways to navigate between pages and states.

### Page navigator visual

A built-in visual that renders page tabs as styled buttons automatically. It updates when pages are added or removed — no manual maintenance needed.

Add it via **Insert → Buttons → Navigator → Page navigator**.

!!! tip
    Hide pages used for drill through and tooltips — the page navigator automatically excludes hidden pages, keeping the navigation clean.

### Buttons

Individual buttons can navigate to a specific page, trigger a bookmark, perform drill through, or open a URL. Buttons support hover and pressed states for visual feedback.

Common button actions:

| Action | Use |
|--------|-----|
| **Page navigation** | Go to a specific page |
| **Bookmark** | Apply a saved state |
| **Drill through** | Navigate to a detail page in context |
| **Back** | Return to the previous page |
| **URL** | Open an external link |

### Back button

Power BI adds a back button automatically to drill through pages. For custom navigation flows, add a back button manually via **Insert → Buttons → Back**.

---

## Drill Through

Drill through allows users to right-click a data point and navigate to a detail page filtered to that context — without setting up a slicer.

### Setup

1. Create a detail page and set its visibility to **Hidden**
2. Add the drill through field to the **Drill through** well in the Filters pane on that page
3. Power BI automatically adds a back button to return to the source page

### Use cases

- Product detail from a sales summary
- Customer profile from a customer list
- Order detail from an order overview

### Cross-report drill through

Drill through can navigate to a page in a **different report** in the same workspace. Enable it in **File → Options → Report settings → Allow visuals in this report to use drill-through targets from other reports**.

!!! note
    Cross-report drill through requires both reports to be published to the same workspace. A workspace in Power BI Service is a shared container where reports, datasets, and dashboards are stored and managed — think of it as a project folder in the cloud. Both the source report and the target report must live in the same workspace for the drill through connection to work. The target page must have the drill through field configured identically.

---

## Tooltips

Custom tooltip pages show additional context when a user hovers over a data point — replacing the default tooltip with a rich mini-report.

### Setup

1. Create a new page
2. Set page type to **Tooltip** in **Format page → Page information → Tooltip: On**
3. Set canvas size to **Tooltip** (320×240) — this prevents the tooltip from being too large
4. Build the visuals on the tooltip page
5. In the target visual, go to **Format visual → General → Tooltips → Type: Report page** and select the tooltip page

### Default vs report page tooltips

| Type | Description |
|------|-------------|
| **Default tooltip** | Auto-generated from the fields in the visual. No setup required. |
| **Report page tooltip** | A custom page with full visual control. Requires setup but much richer. |

!!! tip
    Keep tooltip pages simple — one or two visuals maximum. The tooltip appears on hover and should provide quick context, not a full analysis. Use a dark or contrasting background to visually distinguish it from the main canvas.

---

## Conditional Formatting

Conditional formatting applies dynamic colors, icons, and data bars to table and matrix visuals based on measure values — making exceptions and patterns immediately visible.

### Types

| Type | Description |
|------|-------------|
| **Background color** | Colors the cell background based on a scale or rules |
| **Font color** | Colors the text based on a scale or rules |
| **Data bars** | Adds an inline bar chart inside the cell |
| **Icons** | Adds status icons (arrows, circles, flags) based on thresholds |
| **Web URL** | Makes a cell a clickable link |

### Applying it

In a table or matrix visual: **Format visual → Cell elements → select the column → turn on the formatting type → configure via gradient, rules, or field value**.

### DAX-driven formatting

The most flexible approach is to drive formatting from a DAX measure that returns a color hex code:

```dax
Sales Color =
IF(
    [Total Sales] >= [Sales Target],
    "#2d8a57",  -- green
    "#c0392b"   -- red
)
```

Set the formatting type to **Field value** and select this measure. This gives full control — the color logic lives in DAX and can be as complex as needed.

!!! tip
    Use conditional formatting to communicate meaning, not decoration. Color every cell in a gradient because you can is noise. Color only the cells that require attention — exceptions, targets missed, thresholds crossed.

---

## Themes

Themes define the default colors, fonts, and visual formatting across the entire report. Always create a custom theme to match company branding rather than using the defaults.

### Applying a theme

- **Built-in themes** — available via **View → Themes**. Quick to apply, limited customization.
- **Custom theme JSON** — full control over colors, fonts, and visual defaults. Import via **View → Themes → Browse for themes**.

### Theme JSON structure

```json
{
    "name": "Company Theme",
    "dataColors": [
        "#2d8a57",
        "#3cb371",
        "#1c5c38",
        "#a8d5b5",
        "#1c1c1e",
        "#6c6c6e"
    ],
    "background":   "#f5f5f5",
    "foreground":   "#1c1c1e",
    "tableAccent":  "#2d8a57",
    "visualStyles": {
        "*": {
            "*": {
                "fontFamily": [{ "value": "DM Sans" }]
            }
        }
    }
}
```

!!! note
    Community theme galleries like [themes.powerbi.tips](https://themes.powerbi.tips) offer ready-made themes you can download and customize. Starting from an existing theme is faster than building from scratch.

---

## Mobile Layout

Power BI reports have a separate mobile layout that can be configured per page for phone-sized viewing. It is accessed via **View → Mobile layout** in Power BI Desktop.

The mobile layout is independent from the desktop layout — you choose which visuals to include and arrange them vertically for a single-column display.

!!! note
    Mobile layout is rarely a priority for internal business reports — most users consume reports in a browser on desktop. It becomes relevant when reports are embedded in mobile apps or when the audience is field-based and primarily uses phones. If mobile is not a requirement, skip it.

---

## Best Practices

- Use a consistent color palette — 2 to 3 primary colors maximum, defined in a theme
- Avoid 3D charts, pie charts with many slices, and dual-axis charts
- Use conditional formatting to highlight exceptions — not as decoration
- Always label axes and provide context in visual titles
- Test the report at the actual screen resolution it will be used on
- Remove gridlines and borders when they add noise without value
- Set KPI card interactions to **None** so totals are not filtered by chart clicks
- Hide drill through and tooltip pages — they should never appear in navigation
- Always set bookmark **Data** state to off for show/hide toggles — preserve the user's filter selections
- Use the page navigator visual instead of manual button navigation — it maintains itself automatically