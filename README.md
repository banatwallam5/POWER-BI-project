# Eshopp — Power BI Analysis

An e-commerce analytics project built on a training dataset (customers, orders, products, stores, website sessions) while working through Alex Freberg's free Power BI course. Nine report pages covering a CEO-level summary dashboard, customer and session behavior, product rankings, and geographic population context.

## Why this is a `.pbip`, not a `.pbix`

This project is saved as a **Power BI Project** (`.pbip`) instead of the older `.pbix` format. A `.pbix` is a single zipped binary — GitHub can store it, but nobody can open it on the page, diff it, or see what's inside without launching Power BI Desktop.

A `.pbip` is a folder of plain text instead: the semantic model as TMDL and the report definition as PBIR/JSON. The DAX measures and Power Query M code in this repo are readable directly on GitHub, and changes show up as real diffs. Power BI's developer mode (which produces this format) reached general availability in September 2026, so this is the standard way analytics teams version-control BI projects now, not a workaround.

**What's committed:** `Eshopp.pbip` plus its `Eshopp.Report/` and `Eshopp.SemanticModel/` folders — all three have to travel together, since the `.pbip` file is just a pointer.

## Tools

- Power BI Desktop (developer mode / PBIP)
- Power Query (M)
- DAX

## Data model

Two fact tables at different grains, both tied back to shared dimensions:

- **`sales`** — one row per sale (`sale_id`), with `date_id`, `customer_id`, `product_id`, `store_id`
- **`website_sessions`** — one row per site visit (`session_id`), with `customer_id` and `store_id`
- **Dimensions:** `customers`, `products`, `stores`, `dates`

**`geographies`** (country/state/city, with a population figure) is deliberately **not** related to anything else in the model. There's no real foreign key tying it to the rest of this training dataset, so rather than force a fake join, it's left as a standalone table that only powers the population breakdown on the Geo page. It won't cross-filter with the rest of the report, and that's intentional, not an oversight.

### A real modeling bug I found and fixed

Power BI's "Auto date/time" setting was still on, which meant three different date columns (`customer_signup_date`, `store_open_date`, `session_date`) and even `dates[date]` itself each had their own hidden, auto-generated calendar table — invisible in the Fields pane, each with its own disconnected Year/Quarter/Month hierarchy. On top of the `dates` dimension I'd deliberately built and related to `sales`, that's five competing definitions of "what calendar is this." A chart built on one of those hidden hierarchies wouldn't react to a slicer built on `dates`, with no error — just a chart that silently ignores a filter it obviously should respond to.

Fix: turned off Auto date/time (removes the hidden tables automatically), then built an explicit `Year → Month → Day` hierarchy directly on `dates`. That required adding a `date_year` column first — `dates` had `date_week`, which resets to `1` every January with no year attached, so grouping by it alone silently folds different years' data together. I left `date_week` out of the hierarchy entirely rather than nesting it under Month, since ISO weeks regularly span a month boundary (the last days of December routinely fall in "week 1") — it doesn't nest cleanly, so it doesn't belong in a parent-child hierarchy.

## Power Query

- **`sales`** — de-duplicated on `sale_id` after the type conversion step, so the fact table can't silently double-count a row
- **`website_sessions`** — split a combined `browser_os` column into separate `browser` and `OS` fields, merged them back into a single `OS_on_Device` label, and replaced null `session_quality_score` values with a default of `0.5` rather than leaving them blank

## DAX measures

```dax
total_revenue = SUM(sales[Discounted_price])

total_units_sold = SUM(sales[sale_quantity])

Revenue_in_feb =
CALCULATE(
    sales[total_revenue],
    dates[date] >= DATE(2025, 2, 1),
    dates[date] <= DATE(2025, 2, 28)
)

customer_avg_age = AVERAGE(customers[customer_age])

oldest_store_open_date = MIN(stores[store_open_date])

session_quality_score_avg_scaled =
AVERAGEX(website_sessions, INT(website_sessions[session_quality_score] * 100))

Target_quality_score = 50
Max_quality_score = 100
```

`session_quality_score_avg_scaled` is the one worth a second look: it's an iterator (`AVERAGEX`), not a plain `AVERAGE`, because the raw score needs a row-by-row transform (scale to 0–100, truncate to an integer) *before* it gets averaged — a plain aggregator can't do that transform-then-aggregate step in one pass. `Target_quality_score` and `Max_quality_score` are constant benchmark measures, used as reference lines rather than aggregations of the data itself.

A couple of calculated columns on `sales` worth noting too: `price` and `Discounted_price` both use row context (`RELATED()` to pull in the product's list price per sale), and `OrderSize` buckets each sale into Tiny/Mid/Massive with a `SWITCH(TRUE, ...)` pattern.

## Report pages

Cards · KPIs · Trends · Analysis · Rankings · Customers · Sessions · Geo · CEO Dashboard (default landing page)

## Credit

Built while working through [Alex Freberg's](https://www.youtube.com/@AlexTheAnalyst) free Power BI course on YouTube.

## Opening this project

1. Clone the repo
2. Open `Eshopp.pbip` in Power BI Desktop (requires a version with PBIP/developer mode support)
3. Make sure the `Eshopp.Report/` and `Eshopp.SemanticModel/` folders are present alongside the `.pbip` file
