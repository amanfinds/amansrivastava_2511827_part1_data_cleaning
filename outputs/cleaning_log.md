# Cleaning Log — Retail Order Dataset

This log documents every issue found in `raw_orders.xlsx`, the cleaning actions taken, the business rules applied, and the assumptions made while producing `cleaned_orders.xlsx`.

---

## 1. Issues Found

### 1.1 Text fields
- `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status` all contained:
  - Leading/trailing spaces (e.g. `"  Diya Patel"`, `"  Pending"`)
  - Double/extra internal spaces (e.g. `"Vikram  Iyer"`, `"Small  Business"`)
  - Inconsistent casing — UPPERCASE, lowercase, and mixed case versions of the same value (e.g. `"NORTH"`, `"north"`, `"North"`; `"COMPLETED"`, `"completed"`)

### 1.2 Dates
- `order_date` and `ship_date` were stored in **four different formats** mixed within the same column:
  - `DD Mon YYYY` (e.g. `21 Jul 2024`)
  - `MM/DD/YYYY` (e.g. `07/27/2024`)
  - `DD-MM-YYYY` (e.g. `28-11-2024`)
  - `YYYY-MM-DD` (e.g. `2024-05-24`)
- 22 records had a `ship_date` earlier than the `order_date` (logically invalid shipping record).
- No missing dates and no unparseable/invalid date text were found once all four formats were correctly detected.

### 1.3 Discount field
- 18 records had a missing discount value.
- 16 records had a **negative** discount (e.g. `-0.19`).
- 15 records had a discount **above the normal allowed range** (values like `0.55`, `0.65`, `0.70`, `0.85` — i.e. above 30%).
- 8 records stored discount as a **percentage string** (e.g. `"70%"`, `"85%"`) instead of a decimal.

### 1.4 Duplicates
- 40 rows were involved in exact full-row duplication (20 unique records duplicated once each).
- 31 `order_id` values appeared more than once across the dataset.
- Of those, 12 `order_id` groups contained **conflicting information** (same order_id but different sales/profit/status values) rather than being simple exact duplicates.

### 1.5 Sales/Profit calculation mismatches
- 59 records had a `sales` value that did not match `quantity × unit_price × (1 − discount)` within a tolerance of 1 unit. The large majority of these trace directly back to the invalid/missing discount values described above (1.3).

### 1.6 Missing values
- `region`: 26 missing values
- `ship_mode`: 22 missing values

---

## 2. Cleaning Actions Performed

| Action | Detail |
|---|---|
| Whitespace cleanup | Applied `TRIM`-equivalent logic to remove leading/trailing spaces and collapse multiple internal spaces to one, on all text fields. |
| Case standardization | Standardized all text fields to Title Case (e.g. `"north"`/`"NORTH"` → `"North"`). |
| Date standardization | Parsed all four date formats and converted both `order_date` and `ship_date` to a single consistent `YYYY-MM-DD` format. |
| Discount conversion | Converted percentage-string discounts (`"70%"`) to decimal form (`0.70`). |
| Discount validation | Flagged (did not silently fix) negative discounts and discounts above the 30% allowed maximum. These values are excluded from the `calculated_sales` formula (clamped to the valid 0–30% range only for the purpose of recalculation) but the original `discount` value is preserved unchanged in its own column for audit purposes. |
| Duplicate removal | Removed only exact, fully-identical duplicate rows (kept the first occurrence). |
| Duplicate flagging | `order_id` groups with conflicting data were **not deleted** — they are flagged via `is_conflicting_duplicate_id` = TRUE and `data_quality_flag` = `invalid`, for manual business review. |
| Missing value handling | `region` and `ship_mode` missing values filled with `"Unknown"` and flagged as a warning. `discount` missing values filled with `0` only when quantity and unit price were both present and valid. |

---

## 3. Business Rules Applied

| Rule | Applied As |
|---|---|
| Missing `region` | Filled as `"Unknown"`, flagged in quality report |
| Missing `ship_mode` | Filled as `"Unknown"`, flagged in quality report |
| Missing `discount` | Treated as `0` only when quantity and unit price were valid |
| Negative `discount` | Flagged as invalid; original value preserved, not used in recalculated sales |
| Discount above allowed range (> 30%) | Flagged as invalid; original value preserved, not used in recalculated sales |
| Cancelled orders | Excluded from `contributes_to_completed_sales` and therefore from the completed-sales pivot summaries |
| Failed payments | Excluded from `contributes_to_completed_sales` and therefore from the completed-sales pivot summaries |
| Refunded orders | Summarized separately (`is_refunded` flag; see "Refund-Cancel-Fail by Region" pivot) |
| `ship_date` earlier than `order_date` | Flagged as an invalid shipping record (`data_quality_flag` = `invalid`) |

---

## 4. Assumptions Made

1. The valid discount range for this business is **0% to 30%**. Any discount outside this range (negative, or above 30%) is treated as a data-entry error, not a genuine discount, and is excluded from sales recalculation while being preserved and flagged for review.
2. "Completed sales" for pivot/business-review purposes means orders where `order_status = Completed` **and** `payment_status = Paid`. Orders that are Cancelled or have a Failed payment do not count toward completed sales totals, per the assignment's business rules.
3. Where a discount was missing but quantity and unit price were present, it was assumed the discount was simply not entered (i.e., **0%**) rather than truly unknown — since all other sales fields needed to calculate the order were valid.
4. Title-Case standardization was used for all categorical text fields as the cleanest consistent convention, since the underlying values (region names, categories, etc.) are proper nouns/categories.
5. Exact duplicate rows are assumed to be system export errors (the same record exported twice) and were safely removed, keeping the first occurrence.

---

## 5. Records Removed

- **20 records removed** — these were 100% exact duplicate rows (every column identical to another row). The first occurrence of each was kept; the rest were removed.

---

## 6. Records Flagged (Not Removed)

- **24 records flagged** for manual review — these belong to 12 `order_id` groups where the same order_id appears more than once but with **conflicting data** (e.g. different `sales`, `profit`, or `order_status` values). These were intentionally **not deleted**, since deleting could silently discard a legitimate correction or a genuine data error that the business needs to investigate. They are marked with `is_conflicting_duplicate_id = TRUE` and `data_quality_flag = invalid` in `cleaned_orders.xlsx`.
- Conflicting order IDs: ORD-2025-10091, ORD-2024-10124, ORD-2024-10143, ORD-2025-10171, ORD-2025-10225, ORD-2024-10273, ORD-2025-10315, ORD-2024-10332, ORD-2025-10336, ORD-2025-10374, ORD-2024-10424, ORD-2025-10572.
- Additional records flagged as `warning` (63 total) for less severe issues: missing region/ship_mode filled as Unknown, or a sales/profit calculation mismatch.
- Records flagged as `invalid` (73 total, including the 24 conflicting-duplicate rows above): invalid discount, ship date before order date, or conflicting duplicate order_id.

---

## 7. Limitations of the Cleaning Process

1. **Conflicting duplicates were flagged, not resolved.** A human reviewer with business context (e.g. access to the original source system) is needed to determine which version of each conflicting record is correct.
2. **Invalid discounts were excluded from calculation but not corrected.** We don't know what the "true" intended discount was for these 31 records (16 negative + 15 above-range), so `calculated_sales` treats them as 0% rather than guessing a value.
3. **Date parsing assumed only four known formats.** If the original system used any other date format not seen in this sample, those values would not have been caught by this logic (none were found in this dataset, but this is a structural limitation of the approach).
4. **"Unknown" region/ship_mode records remain unresolved.** These could potentially be inferred from `state`/`city` (for region) or from related historical orders, but that level of inference was out of scope for this cleaning pass.
5. **No outlier detection was performed on `unit_price`, `quantity`, `cost`, or `profit`** beyond the sales-recalculation mismatch check — extreme but technically valid values were not separately investigated.
