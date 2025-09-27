# E-commerce Customer Behaviour — Analysis & Dashboard

This repository contains a lightweight pipeline (Python) and a Power BI model to analyze an e-commerce “events vs. purchases” dataset, with a focus on **customer behavior**, **conversion timing**, and **demand mix**. The project is designed to remain honest about missing monetary fields while still extracting actionable insight.

---

## Contents

* `Python_analysis.ipynb` — notebook to load, clean, and engineer features (events, purchases, RF flags, pre-purchase metrics).
* `E-commerce Customer Behaviour Data Analysis.pdf` — presentation summarizing methods, charts, and recommendations. 
* (Optionally exported) CSVs for Power BI:

  * `user_features_pbi.csv` (one row per user)
  * `fact_activity.csv` (all rows incl. non-purchase events)
  * `fact_purchases.csv` (purchase rows only)
  * `dim_date.csv` (daily calendar)
  * `dim_users.csv` (slim user dimension)

---

## Quick Summary

* **Two row types**

  * **Event** = one activity row (timestamped interaction).
  * **Purchase row** = event where **Quantity + Rate + Total Price** are all present.
* **User segments**

  * **Converter** = user with ≥ 1 purchase row.
  * **Non-converter** = user with 0 purchase rows.
* **Key observations**

  * Activity (all users) and purchases peak around **Q4**.
  * **October** has the **most activity and purchases (volume)**.
  * **Nov–Dec** show the **highest purchase probability (rate)**.
  * A few sub-categories dominate quantity; weekday/hour patterns favor **Mondays** and **13–15 / 19–21** windows.
* **Recommendations**

  * Pre-peak (Sep–Oct) first-purchase nudges; capture contacts and remarket into Nov–Dec.
  * Align promos and staffing to **Mon + evening** peaks.
  * Feature top sellers; protect inventory in Q4. 

---

## Data & Assumptions

* Source: E-commerce interaction logs (Kaggle dataset referenced in the deck).
* High missingness in monetary columns; **no imputation** of price/qty. We use **only true purchase rows** for revenue/qty and treat other rows as **behavioral events**. 

---

## How to Reproduce (Python → PBI)

### 1) Run the notebook

1. Open `Python_analysis.ipynb`.
2. Point the loader to your Excel/CSV file (the dataset you’re using).
3. Execute all cells to:

   * Standardize datetime, derive `weekday`, `hour`, etc.
   * Flag purchase rows (`is_purchase_row`), mark user-level converters.
   * Build `user_features` (first/last seen, purchases, recency, frequency, **events_before_first_purchase**, **days_to_first_purchase**).
   * Export CSVs for Power BI (optional).

> The notebook was prepared to output a small “star” model: `fact_activity`, `fact_purchases`, `user_features_pbi`, and `dim_date`.

### 2) Model in Power BI (minimal)

**Relationships**

* `Date[Date]` 1—* `fact_activity[Date]`
* `Date[Date]` 1—* `fact_purchases[Date]`
* `user_features_pbi[User_id]` 1—* `fact_activity[User_id]`
* `user_features_pbi[User_id]` 1—* `fact_purchases[User_id]`

**Measures (paste into a “Measures” table)**

```DAX
Total Events = COUNTROWS ( 'fact_activity' )
Total Purchases (rows) = COUNTROWS ( 'fact_purchases' )

Events (Converters) =
CALCULATE ( [Total Events], TREATAS ( {1}, user_features_pbi[has_purchased_ever] ) )

Events (Non-Converters) =
CALCULATE ( [Total Events], TREATAS ( {0}, user_features_pbi[has_purchased_ever] ) )

Purchase Rate =
DIVIDE ( [Total Purchases (rows)], [Total Events] )

Purchasing Users =
DISTINCTCOUNT ( 'fact_purchases'[User_id] )

-- First-time converters by month (Option A: TREATAS; create user_features_pbi[FirstPurchaseDate] as DATE from first_purchase)
First-time Converters =
CALCULATE (
    DISTINCTCOUNT ( user_features_pbi[User_id] ),
    FILTER ( user_features_pbi, NOT ISBLANK ( user_features_pbi[FirstPurchaseDate] ) ),
    TREATAS ( VALUES ( 'Date'[Date] ), user_features_pbi[FirstPurchaseDate] )
)
```

**Date table (DAX)**

```DAX
Date =
ADDCOLUMNS (
    CALENDAR ( MIN ( 'fact_activity'[Date] ), MAX ( 'fact_activity'[Date] ) ),
    "Year", YEAR ( [Date] ),
    "Month", FORMAT ( [Date], "MMMM" ),
    "MonthNum", MONTH ( [Date] ),
    "YearMonth", FORMAT ( [Date], "YYYY-MM" ),
    "Weekday", FORMAT ( [Date], "dddd" ),
    "WeekdayNum", WEEKDAY ( [Date], 2 )
)
```

Mark as Date table (Modeling → Mark as date table).

---

Got it—here’s a short, skimmable version you can drop into your README.

# Process 

**1) Data preprocessing**

* Load file, parse `DateTime`, keep rows with `User_id` + `DateTime`.
* Quick checks: row count, unique users, % missing by column.

**2) Split row types**

* **Purchase rows** = `Quantity` + `Rate` + `Total Price` all present.
* **Events** = other activity rows (no monetary fields).
* Tag `is_purchase_row`; mark users with any purchase as **converters**.

**3) Time features**

* Derive `Date`, `YearMonth`, `Weekday` (plus sort key), `Hour` from `DateTime`.

**4) User features (per user)**

* From all rows: `first_seen`, `last_seen`, `days_active`, `total_events`, `events_per_day_active`.
* From purchase rows only: `purchases`, `first_purchase`, `last_purchase`, `total_quantity`, `total_revenue`, `median_rate`.
* Derived: `has_purchased_ever`, `recency_days`, `frequency_purchases`.

**5) Funnel signals**

* `events_before_first_purchase` (how much activity before first buy).
* `days_to_first_purchase` (time from first_seen to first buy).

**6) RF segmentation**

* **R_quartile** (more recent = higher R), **F_quartile** (more orders = higher F).
* Optionally use rule-based buckets (e.g., R0/F0 for never purchased).

**7) BI modeling (light)**

* Facts: **fact_activity** (all rows), **fact_purchases** (purchase rows).
* Dim: **user_features** (one row per user), **Date** (calendar).
* Build relationships on `Date` and `User_id`. Use Date fields on time axes.

**8) Core metrics**

* **Total Events**, **Total Purchases (rows)**, **Purchase Rate = Purchases ÷ Events** (propensity).
* **Purchasing Users** (distinct buyers), **First-time Converters** (buyers whose first purchase is in period).
* Demand = top items/subcategories by **total_quantity** (from purchase rows only).

---

## Feature Engineering (high-value fields)

* **Row level**

  * `is_purchase_row` (all 3 value cols present)
  * `weekday`, `weekday_num`, `hour`, `Date`
* **User level (in `user_features_pbi`)**

  * `first_seen`, `last_seen`, `days_active`, `events_per_day_active`
  * `purchases`, `total_quantity`, `total_revenue` (true purchases only)
  * `has_purchased_ever` (0/1), `recency_days`, `frequency_purchases`
  * **`events_before_first_purchase`**, **`days_to_first_purchase`**
  * **RF**: `R_quartile` (smaller recency → higher score), `F_quartile`, `RF_segment`

> Non-converters have no `last_purchase` → **R is blank**. For a business-friendly matrix, use rule-based buckets like `R0: No Purchase Yet`, `R4: ≤7d`, etc.

---

## Core Questions this Project Answers

1. **When do we peak (seasonality & cadence)?**

   * Events by **month**, **weekday**, **hour** (all users)
   * **Active Non-Converter Users** by month
   * **Median events before 1st purchase** (funnel warm-up)

2. **When and what converts (buying patterns & demand)?**

   * **Purchase Rate** by month (probability = purchases ÷ events)
   * Purchases by **weekday** / **hour**
   * **Top SubCategory** (or product proxy) by quantity

---

## Interpreting “Purchase Rate”

* **Definition:** `Purchase Rate = purchases ÷ total events` within the filter context (e.g., a month).
* **Implication:** a higher rate means visitors were **more likely** to turn actions into purchases.
  It does **not** mean there were more buyers; pair it with **Purchasing Users** and **First-time Converters** for the “people” view.

This explains why **October** can have the **most purchases (volume)**, while **Nov–Dec** show a **higher purchase rate (propensity)**.

---

## Known Limitations

* Monetary analysis is limited to **true purchase rows** (no imputation of price/qty).
* Product detail may be limited; if SKU/catalog data is added later, demand analysis becomes richer.
* Timezone alignment may be needed if raw timestamps are UTC.

---

## What to Do Next

* **Pre-peak (Sep–Oct)**: on-site nudges after ~3–4 actions, capture emails/push for remarketing.
* **Peak (Nov–Dec)**: time-boxed promos; simplify checkout; staff for **Mon** and **evening** windows.
* **Merch/ops**: feature top sellers; protect inventory in Q4; bundle long-tail items.

---

## Credits / Contact

* Notebook & dashboard wiring by **Rizky Habibie**.
* Presentation: *E-commerce Customer Behaviour — Data Analysis*. 
* Contact: `rizkyfebriibrahabibie@gmail.com`

---
