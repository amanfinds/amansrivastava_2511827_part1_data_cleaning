# Business Data Cleaning, Validation & Excel Reporting

## Problem Summary

A retail company exports order-level sales data from multiple internal systems. The raw export (`raw_orders.xlsx`) contains a wide range of data quality problems — inconsistent text formatting, mixed date formats, duplicate records, missing values, invalid discounts, sales/profit calculation mismatches, and inconsistent order status values.

This project cleans and validates that dataset, documents every issue found, applies the business's stated data rules, and produces summary reports (data quality report + pivot summaries) that the business can use for review — all done in Excel.

## Dataset Description

- **Source file:** `data/raw_orders.xlsx`
- **Rows:** 932 order-level records
- **Columns:** 21 — order identifiers, dates, customer info, location, product category, ship mode, quantities/prices/discounts, sales/cost/profit, and payment/order status.
- **Cleaned output:** `data/cleaned_orders.xlsx` — 912 records (after exact-duplicate removal) with 35 columns including all required calculated and flag columns.

## Tools Used

- Microsoft Excel / `.xlsx` workbooks (openpyxl-generated, fully formula-driven for all calculated numeric columns)
- Excel functions: `TRIM`, text standardization, `DATEVALUE`, `MONTH`, `YEAR`, `ROUND`, `MIN`/`MAX`/`IF` for validation logic
- PivotTable-style grouped summaries with sorting and Excel Tables (filterable)

## Cleaning Steps Performed

1. **Text fields** — trimmed extra/leading/trailing spaces, collapsed double spaces, standardized casing to Title Case across `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status`.
2. **Dates** — detected and parsed four different raw date formats (`DD Mon YYYY`, `MM/DD/YYYY`, `DD-MM-YYYY`, `YYYY-MM-DD`) in both `order_date` and `ship_date`, and standardized both to `YYYY-MM-DD`. Added `shipping_delay_days`.
3. **Discounts** — converted percentage-string discounts (e.g. `"70%"`) to decimal form, flagged negative and above-30%-range discounts as invalid, and filled missing discounts with `0` where the rest of the order's sales fields were valid.
4. **Duplicates** — removed 20 exact full-row duplicates (kept first occurrence); flagged (but did **not** delete) 24 records belonging to 12 `order_id` groups with genuinely conflicting data.
5. **Calculated columns** — added `cleaned_discount`, `calculated_sales`, `calculated_profit`, `profit_margin`, `shipping_delay_days`, `order_month`, `order_year`, and `data_quality_flag` (clean / warning / invalid), all built as live Excel formulas.

Full detail is in [`outputs/cleaning_log.md`]

## Business Rules Applied

| Rule | Applied As |
|---|---|
| Missing `region` | Filled `"Unknown"`, flagged |
| Missing `ship_mode` | Filled `"Unknown"`, flagged |
| Missing `discount` | Filled `0` only if other sales fields valid |
| Negative / above-range `discount` | Flagged invalid, excluded from recalculated sales |
| Cancelled orders | Excluded from completed-sales summaries |
| Failed payments | Excluded from completed-sales summaries |
| Refunded orders | Summarized separately |
| Ship date before order date | Flagged as invalid shipping record |

## Summary of Data Quality Issues Found

- 26 missing `region`, 22 missing `ship_mode`, 18 missing `discount`
- 16 negative discounts, 15 discounts above the allowed 30% range, 8 discounts stored as percent-strings
- 22 records with `ship_date` earlier than `order_date`
- 40 rows involved in exact duplication (20 records removed), 12 `order_id` groups with conflicting data (24 records flagged, not removed)
- 59 records where the original `sales` value didn't match the recalculated sales (mostly explained by the invalid discounts above)
- **Final record classification:** 776 clean, 63 warning, 73 invalid (out of 912 cleaned records)

Full breakdown is in [`outputs/data_quality_report.xlsx`]

## Summary of Final Pivot Reports

`outputs/pivot_summary.xlsx` contains six summary sheets (completed & paid orders only, unless noted):

1. **Sales & Profit by Region** — sorted descending by sales
2. **Sales & Profit by Category & Sub-Category** — sorted by category then sales
3. **Order Count by Ship Mode** — all cleaned records, sorted by count
4. **Profit Margin by Customer Segment** — sorted descending by margin
5. **Refunded / Cancelled / Failed Orders by Region** — all affected records
6. **Monthly Sales Trend** — by year and month

## Key Business Insights

- **Technology is the top-performing category**, generating roughly ₹21.8L in completed sales and ₹6.4L in profit — ahead of both Furniture and Office Supplies.
- **Home Office is the most profitable customer segment by margin** (~29.8%), narrowly ahead of Corporate, Consumer, and Small Business — all of which sit within a tight 28–30% band.
- **South and West regions lead on sales volume**, each contributing over ₹15L in completed sales; the "Unknown" region bucket (records with missing region data) still accounts for a non-trivial ~₹1.9L, reinforcing the value of fixing the missing-region data issue at the source.
- **Order disruption is meaningful**: 145 cancelled orders, 71 refunded orders, and 69 failed payments exist in the cleaned dataset — together representing a sizeable share of total order volume that does not convert to completed revenue.
- **Average shipping delay is ~3.8 days** across all orders, with 22 records showing a logically invalid negative delay (ship date before order date) that warrants investigation with the shipping/logistics system.

## Assumptions and Limitations

- Valid discount range assumed to be 0%–30%; values outside this were treated as data errors, not real discounts.
- "Completed sales" = `order_status = Completed` **and** `payment_status = Paid`.
- Conflicting duplicate order_id records were flagged for manual business review rather than auto-resolved, since there is no reliable way to determine the "correct" version without more context.
- Date parsing covers the four formats observed in this dataset; other formats would need additional logic.



