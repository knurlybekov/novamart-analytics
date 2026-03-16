# NovaMart E-Commerce Bottleneck Analysis — Project Roadmap
### A Senior Data Analyst's Guide to Building a Portfolio-Grade Project

---

## How to Use This Document

This roadmap is structured as **7 phases**. Each phase builds on the previous one. Do NOT skip phases — the sequence matters because hiring managers evaluate your *process*, not just your charts.

For each task, I've marked:
- **[SQL]** = Write this in Snowflake SQL
- **[PY]** = Write this in Python (pandas/matplotlib/seaborn)
- **[VIZ]** = Build this in your dashboard tool (Tableau, Power BI, or Streamlit)
- **[DOC]** = Write this up as documentation or commentary
- **[PORTFOLIO]** = This specific output goes on your GitHub/portfolio

---

## PHASE 0: Project Setup & Repository Structure (Day 1)

Before you write a single query, set up your project like a professional. This is the first thing a hiring manager sees.

### 0.1 — GitHub Repository Structure [DOC] [PORTFOLIO]

Create a repo with this exact structure:

```
novamart-ecommerce-analysis/
│
├── README.md                    ← Project overview (write LAST, after analysis)
├── data/
│   └── data_dictionary.md       ← The dictionary I gave you
│
├── sql/
│   ├── 01_staging/              ← Raw → cleaned staging tables
│   ├── 02_intermediate/         ← Joins, calculated fields
│   ├── 03_marts/                ← Final analytical tables
│   └── 04_analysis/             ← Ad-hoc analysis queries
│
├── python/
│   ├── 01_data_quality.py       ← Data profiling & quality checks
│   ├── 02_eda.py                ← Exploratory data analysis
│   ├── 03_statistical_tests.py  ← Hypothesis testing
│   ├── 04_visualizations.py     ← Publication-quality charts
│   └── utils/                   ← Helper functions
│
├── dashboards/
│   └── screenshots/             ← Dashboard screenshots for README
│
├── reports/
│   └── executive_summary.md     ← Final findings document
│
└── notebooks/
    └── exploration.ipynb        ← Messy exploratory work (optional)
```

**Why this matters:** A hiring manager spends 30 seconds scanning your repo. Clear structure = immediate signal that you think like a professional, not a student.

### 0.2 — Snowflake Schema Setup [SQL]

You've already uploaded the raw data. Now create a proper schema hierarchy:

```sql
CREATE SCHEMA IF NOT EXISTS novamart_raw;       -- Raw uploaded data lives here
CREATE SCHEMA IF NOT EXISTS novamart_staging;    -- Cleaned, typed, validated
CREATE SCHEMA IF NOT EXISTS novamart_marts;      -- Final analytical tables
```

**Why three schemas?** This is called the **Medallion Architecture** (Bronze → Silver → Gold) or the **staging → intermediate → marts** pattern from dbt. Every modern data team uses some version of this. Showing it in your project tells employers you understand production data pipelines — not just ad-hoc querying.

---

## PHASE 1: Data Profiling & Quality Assessment (Days 2–3)

**Goal:** Before any analysis, you need to *know your data*. This phase answers: "Can I trust this data? Where can't I?"

A real analyst's first task on any new dataset is NEVER "start making charts." It's always "how broken is this data and what do I need to fix?"

### 1.1 — Completeness Profiling [SQL]

For EVERY table, run completeness checks. Write a query for each table like this:

```sql
-- Example for orders table
SELECT
    COUNT(*)                                           AS total_rows,
    COUNT(customer_id)                                 AS customer_id_present,
    ROUND(100.0 * COUNT(customer_id) / COUNT(*), 2)   AS customer_id_pct,
    COUNT(approval_date)                               AS approval_date_present,
    COUNT(shipped_date)                                AS shipped_date_present,
    COUNT(delivered_date)                               AS delivered_date_present,
    COUNT(estimated_delivery_date)                      AS est_delivery_present
FROM novamart_raw.orders;
```

Do this for ALL 12 tables. Document every column's null rate. Save this output — it becomes part of your write-up.

**What you'll find:** ~2% of orders have NULL customer_id. This is a real pattern — in production systems, orphaned records happen due to race conditions, failed lookups, or system migrations. You need to decide: exclude them or handle them? Document your decision and reasoning.

### 1.2 — Validity Checks [SQL]

Check for impossible or suspicious values:

```sql
-- Impossible dates: delivered before ordered
SELECT COUNT(*) AS impossible_delivery_dates
FROM novamart_raw.orders
WHERE delivered_date < order_date;

-- Negative or zero prices
SELECT COUNT(*) AS invalid_prices
FROM novamart_raw.order_items
WHERE unit_price <= 0 OR item_total <= 0;

-- Orders with status 'Delivered' but no delivered_date
SELECT COUNT(*) AS missing_delivery_timestamp
FROM novamart_raw.orders
WHERE order_status = 'Delivered' AND delivered_date IS NULL;

-- Duplicate primary keys
SELECT order_id, COUNT(*) AS dupes
FROM novamart_raw.orders
GROUP BY order_id
HAVING COUNT(*) > 1;

-- Referential integrity: items referencing non-existent orders
SELECT COUNT(*) AS orphan_items
FROM novamart_raw.order_items oi
LEFT JOIN novamart_raw.orders o ON oi.order_id = o.order_id
WHERE o.order_id IS NULL;
```

**Run similar checks for every join relationship in the dataset.** This is boring but essential. In interviews, when someone asks "How do you start a new analysis?" the answer is "data profiling and quality assessment" — and you'll have the project to prove it.

### 1.3 — Statistical Profiling [PY]

Use Python for distribution analysis that's harder in SQL:

```python
import pandas as pd

orders = pd.read_excel('01_orders.xlsx')

# Numeric distributions
print(orders[['order_total', 'shipping_cost', 'total_weight_kg']].describe())

# Check for outliers using IQR
for col in ['order_total', 'shipping_cost']:
    Q1, Q3 = orders[col].quantile([0.25, 0.75])
    IQR = Q3 - Q1
    outliers = orders[(orders[col] < Q1 - 1.5*IQR) | (orders[col] > Q3 + 1.5*IQR)]
    print(f"{col}: {len(outliers)} outliers ({len(outliers)/len(orders)*100:.1f}%)")
```

### 1.4 — Data Quality Report [DOC] [PORTFOLIO]

Write a 1-page summary documenting:
1. Total records per table
2. Null rates for key columns
3. Invalid/impossible records found
4. Outliers detected
5. **Your decisions** on how to handle each issue (exclude, impute, flag, etc.)

**This document alone demonstrates more professional maturity than 90% of portfolio projects.** Most students go straight to modeling. You're showing that you understand data governance.

---

## PHASE 2: Staging Layer — Clean, Type, Enrich (Days 4–6)

**Goal:** Transform raw data into clean, analysis-ready tables with calculated fields.

### 2.1 — Build Staging Tables [SQL]

Create cleaned versions of each key table. Here's the critical one for orders:

```sql
CREATE OR REPLACE TABLE novamart_staging.stg_orders AS
SELECT
    order_id,
    customer_id,

    -- Parse timestamps
    TO_TIMESTAMP(order_date)               AS order_ts,
    TO_TIMESTAMP(approval_date)            AS approval_ts,
    TO_TIMESTAMP(shipped_date)             AS shipped_ts,
    TO_TIMESTAMP(delivered_date)           AS delivered_ts,
    TO_TIMESTAMP(estimated_delivery_date)  AS estimated_delivery_ts,

    -- Date parts (incredibly useful for grouping)
    DATE_TRUNC('month', TO_TIMESTAMP(order_date))  AS order_month,
    DATE_TRUNC('week', TO_TIMESTAMP(order_date))   AS order_week,
    DAYOFWEEK(TO_TIMESTAMP(order_date))             AS order_dow,
    HOUR(TO_TIMESTAMP(order_date))                  AS order_hour,
    QUARTER(TO_TIMESTAMP(order_date))               AS order_quarter,
    YEAR(TO_TIMESTAMP(order_date))                  AS order_year,

    -- Lifecycle duration metrics (THE CORE OF YOUR BOTTLENECK ANALYSIS)
    DATEDIFF('hour', order_date, approval_date)      AS hours_to_approve,
    DATEDIFF('hour', approval_date, shipped_date)    AS hours_to_ship,
    DATEDIFF('hour', shipped_date, delivered_date)   AS hours_in_transit,
    DATEDIFF('hour', order_date, delivered_date)     AS hours_total_lifecycle,

    -- Late delivery flag
    CASE
        WHEN delivered_date > estimated_delivery_date THEN 1
        ELSE 0
    END AS is_late_delivery,

    DATEDIFF('day', estimated_delivery_date, delivered_date) AS days_late,

    -- Data quality flags
    CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END          AS flag_missing_customer,
    CASE WHEN delivered_date < order_date THEN 1 ELSE 0 END  AS flag_impossible_date,

    -- Original fields
    order_status,
    customer_region,
    customer_city,
    seller_id,
    carrier,
    warehouse_origin,
    num_items,
    order_total,
    shipping_cost,
    total_weight_kg,
    payment_method,
    is_weekend_order,
    order_channel

FROM novamart_raw.orders
WHERE order_status != 'Cancelled'  -- Filter out cancellations (analyzed separately)
  AND (delivered_date >= order_date OR delivered_date IS NULL);  -- Remove impossible dates
```

**Critical concept: You're creating the lifecycle duration columns here.** These four metrics — hours_to_approve, hours_to_ship, hours_in_transit, hours_total_lifecycle — are the foundation of your entire bottleneck analysis. Without them, you can't measure anything.

### 2.2 — Build Staging for All Other Tables [SQL]

Create similar staging tables for:

**stg_order_items** — Join with products to add category, margin, and cost:
```sql
CREATE OR REPLACE TABLE novamart_staging.stg_order_items AS
SELECT
    oi.*,
    p.category,
    p.subcategory,
    p.cost_price,
    p.weight_kg AS unit_weight_kg,
    p.is_fragile,
    p.is_hazardous,
    -- Profit calculation
    (oi.unit_price - p.cost_price) * oi.quantity AS item_profit,
    ROUND((oi.unit_price - p.cost_price) / NULLIF(oi.unit_price, 0) * 100, 2) AS margin_pct
FROM novamart_raw.order_items oi
LEFT JOIN novamart_raw.products p ON oi.product_id = p.product_id;
```

**stg_sellers** — Enriched with fulfillment type flags:
```sql
CREATE OR REPLACE TABLE novamart_staging.stg_sellers AS
SELECT
    *,
    CASE WHEN fulfillment_type = 'Dropship' THEN 1 ELSE 0 END AS is_dropship,
    CASE WHEN warehouse_location = 'WH-Southeast' THEN 1 ELSE 0 END AS is_southeast_wh
FROM novamart_raw.sellers;
```

**stg_returns** — Add return timing:
```sql
CREATE OR REPLACE TABLE novamart_staging.stg_returns AS
SELECT
    r.*,
    DATEDIFF('day', r.return_initiated_date, r.return_received_date) AS return_transit_days,
    p.category AS product_category,
    p.cost_price,
    (r.refund_amount + r.restocking_fee) AS total_return_cost
FROM novamart_raw.returns_cancellations r
LEFT JOIN novamart_raw.products p ON r.product_id = p.product_id;
```

Build similar staging for: stg_reviews, stg_inventory, stg_campaigns, stg_tickets, stg_shipping_events.

### 2.3 — Build Analytical Marts [SQL]

Now create the final, fully-joined tables optimized for your specific analyses:

**mart_order_complete** — THE master table joining everything:
```sql
CREATE OR REPLACE TABLE novamart_marts.mart_order_complete AS
SELECT
    o.*,
    s.seller_name,
    s.warehouse_location,
    s.quality_score        AS seller_quality_score,
    s.fulfillment_type,
    s.avg_processing_hours AS seller_avg_processing_hours,
    c.region               AS customer_region_detail,
    c.acquisition_channel,
    c.customer_segment,
    c.lifetime_value_estimate,
    -- Aggregated item-level info
    items.total_items_qty,
    items.total_profit,
    items.avg_margin_pct,
    items.primary_category,
    -- Return info
    ret.has_return,
    ret.return_reason,
    ret.total_refund_amount,
    -- Review info
    rev.rating,
    rev.delivery_rating,
    rev.was_delivery_late AS review_flagged_late
FROM novamart_staging.stg_orders o
LEFT JOIN novamart_staging.stg_sellers s ON o.seller_id = s.seller_id
LEFT JOIN novamart_raw.customers c ON o.customer_id = c.customer_id
LEFT JOIN (
    SELECT
        order_id,
        SUM(quantity) AS total_items_qty,
        SUM(item_profit) AS total_profit,
        AVG(margin_pct) AS avg_margin_pct,
        -- Most expensive item's category = "primary category"
        MAX_BY(category, item_total) AS primary_category
    FROM novamart_staging.stg_order_items
    GROUP BY order_id
) items ON o.order_id = items.order_id
LEFT JOIN (
    SELECT
        order_id,
        1 AS has_return,
        MAX(return_reason) AS return_reason,
        SUM(refund_amount) AS total_refund_amount
    FROM novamart_staging.stg_returns
    GROUP BY order_id
) ret ON o.order_id = ret.order_id
LEFT JOIN (
    SELECT order_id, rating, delivery_rating, was_delivery_late
    FROM novamart_raw.customer_reviews
    QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY review_date DESC) = 1
) rev ON o.order_id = rev.order_id;
```

**Why this matters:** You now have ONE table where each row is an order with seller context, customer context, item profitability, return status, and review data. This is the table you'll build 80% of your analysis on. Hiring managers LOVE seeing this kind of denormalized analytical table — it shows you understand dimensional modeling.

---

## PHASE 3: Exploratory Data Analysis (Days 7–10)

**Goal:** Develop intuition about what's happening before formal analysis. This is where you discover the stories in the data.

### 3.1 — Volume & Revenue Trends [SQL] [VIZ]

```sql
-- Monthly order volume and revenue
SELECT
    order_month,
    COUNT(*) AS order_count,
    SUM(order_total) AS gross_revenue,
    SUM(total_profit) AS gross_profit,
    AVG(order_total) AS avg_order_value,
    SUM(CASE WHEN has_return = 1 THEN 1 ELSE 0 END) AS return_orders,
    SUM(CASE WHEN is_late_delivery = 1 THEN 1 ELSE 0 END) AS late_orders
FROM novamart_marts.mart_order_complete
GROUP BY order_month
ORDER BY order_month;
```

**Chart this as a time series.** You should see a clear seasonal pattern with Q4 spikes.

### 3.2 — Lifecycle Stage Duration Analysis (THE CORE) [SQL] [VIZ]

This is the heart of your bottleneck analysis:

```sql
-- Median and P90 durations at each lifecycle stage
SELECT
    'Approval' AS stage,
    MEDIAN(hours_to_approve) AS median_hours,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY hours_to_approve) AS p90_hours,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY hours_to_approve) AS p95_hours
FROM novamart_marts.mart_order_complete
WHERE hours_to_approve IS NOT NULL

UNION ALL

SELECT
    'Processing & Shipping',
    MEDIAN(hours_to_ship),
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY hours_to_ship),
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY hours_to_ship)
FROM novamart_marts.mart_order_complete
WHERE hours_to_ship IS NOT NULL

UNION ALL

SELECT
    'In Transit',
    MEDIAN(hours_in_transit),
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY hours_in_transit),
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY hours_in_transit)
FROM novamart_marts.mart_order_complete
WHERE hours_in_transit IS NOT NULL;
```

**What to look for:** The gap between median and P90 tells you where the "tail" problems are. If median approval is 5 hours but P90 is 22 hours, that means 10% of orders are getting stuck somewhere — and that's your bottleneck.

### 3.3 — Segmented Bottleneck Discovery [SQL]

Now the powerful part — break down each stage by dimension:

```sql
-- Processing time by warehouse
SELECT
    warehouse_origin,
    COUNT(*) AS orders,
    MEDIAN(hours_to_ship) AS median_processing_hrs,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY hours_to_ship) AS p90_processing_hrs,
    AVG(CASE WHEN is_late_delivery = 1 THEN 1.0 ELSE 0 END) AS late_delivery_rate
FROM novamart_marts.mart_order_complete
WHERE hours_to_ship IS NOT NULL
GROUP BY warehouse_origin
ORDER BY median_processing_hrs DESC;

-- Transit time by carrier x season
SELECT
    carrier,
    CASE WHEN MONTH(order_ts) IN (12, 1, 2) THEN 'Winter'
         WHEN MONTH(order_ts) IN (3, 4, 5) THEN 'Spring'
         WHEN MONTH(order_ts) IN (6, 7, 8) THEN 'Summer'
         ELSE 'Fall' END AS season,
    COUNT(*) AS orders,
    MEDIAN(hours_in_transit) AS median_transit_hrs,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY hours_in_transit) AS p90_transit_hrs
FROM novamart_marts.mart_order_complete
WHERE hours_in_transit IS NOT NULL
GROUP BY carrier, season
ORDER BY carrier, season;

-- Approval time: weekday vs weekend
SELECT
    CASE WHEN is_weekend_order = 1 THEN 'Weekend' ELSE 'Weekday' END AS day_type,
    COUNT(*) AS orders,
    MEDIAN(hours_to_approve) AS median_approval_hrs,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY hours_to_approve) AS p90_approval_hrs
FROM novamart_marts.mart_order_complete
WHERE hours_to_approve IS NOT NULL
GROUP BY day_type;
```

**Visualize these as grouped bar charts or heatmaps.** The patterns will be dramatic.

### 3.4 — Return Rate Deep Dive [SQL]

```sql
-- Return rate by category x warehouse
SELECT
    primary_category,
    warehouse_origin,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN has_return = 1 THEN 1 ELSE 0 END) AS returned_orders,
    ROUND(100.0 * SUM(CASE WHEN has_return = 1 THEN 1 ELSE 0 END) / COUNT(*), 2) AS return_rate_pct,
    SUM(COALESCE(total_refund_amount, 0)) AS total_refund_cost
FROM novamart_marts.mart_order_complete
GROUP BY primary_category, warehouse_origin
HAVING COUNT(*) >= 30
ORDER BY return_rate_pct DESC;

-- Do late deliveries cause more returns?
SELECT
    is_late_delivery,
    primary_category,
    COUNT(*) AS orders,
    SUM(CASE WHEN has_return = 1 THEN 1 ELSE 0 END) AS returns,
    ROUND(100.0 * SUM(CASE WHEN has_return = 1 THEN 1 ELSE 0 END) / COUNT(*), 2) AS return_rate
FROM novamart_marts.mart_order_complete
WHERE primary_category IN ('Electronics', 'Clothing')
GROUP BY is_late_delivery, primary_category
ORDER BY primary_category, is_late_delivery;
```

### 3.5 — Marketing Channel Efficiency [SQL]

```sql
-- Channel performance comparison
SELECT
    channel,
    SUM(spend) AS total_spend,
    SUM(conversions) AS total_conversions,
    SUM(revenue_attributed) AS total_revenue,
    ROUND(SUM(revenue_attributed) / NULLIF(SUM(spend), 0), 2) AS roas,
    ROUND(SUM(spend) / NULLIF(SUM(conversions), 0), 2) AS cpa,
    ROUND(SUM(spend) / NULLIF(SUM(clicks), 0), 2) AS cpc,
    ROUND(100.0 * SUM(clicks) / NULLIF(SUM(impressions), 0), 4) AS ctr_pct
FROM novamart_raw.marketing_campaigns
WHERE spend > 0
GROUP BY channel
ORDER BY roas DESC;
```

---

## PHASE 4: Statistical Validation (Days 11–13)

**Goal:** Prove your bottleneck findings are statistically significant, not just visual patterns. This is what separates a junior analyst from someone ready for a senior role.

### 4.1 — Hypothesis Testing for Bottlenecks [PY]

Don't just say "weekends are slower." Prove it:

```python
from scipy import stats

# Test 1: Weekend vs Weekday approval time
weekend = df[df['is_weekend_order'] == 1]['hours_to_approve'].dropna()
weekday = df[df['is_weekend_order'] == 0]['hours_to_approve'].dropna()

# Mann-Whitney U test (non-parametric, doesn't assume normal distribution)
stat, pvalue = stats.mannwhitneyu(weekend, weekday, alternative='greater')
print(f"Weekend vs Weekday approval: U={stat:.0f}, p={pvalue:.6f}")
print(f"Weekend median: {weekend.median():.1f}h, Weekday median: {weekday.median():.1f}h")
# If p < 0.05, the difference is statistically significant

# Test 2: BudgetPost winter vs non-winter transit
bp = df[df['carrier'] == 'BudgetPost']
bp_winter = bp[bp['order_ts'].dt.month.isin([12, 1, 2])]['hours_in_transit'].dropna()
bp_other  = bp[~bp['order_ts'].dt.month.isin([12, 1, 2])]['hours_in_transit'].dropna()

stat, pvalue = stats.mannwhitneyu(bp_winter, bp_other, alternative='greater')
print(f"BudgetPost winter vs non-winter: U={stat:.0f}, p={pvalue:.6f}")

# Test 3: WH-Southeast vs other warehouses processing time
se = df[df['warehouse_origin'] == 'WH-Southeast']['hours_to_ship'].dropna()
other = df[df['warehouse_origin'] != 'WH-Southeast']['hours_to_ship'].dropna()

stat, pvalue = stats.mannwhitneyu(se, other, alternative='greater')
print(f"WH-Southeast vs others: U={stat:.0f}, p={pvalue:.6f}")
```

**Why Mann-Whitney and not t-test?** Fulfillment durations are right-skewed (some orders take much longer than average). The Mann-Whitney U test doesn't assume a normal distribution, making it more appropriate. Knowing this distinction is interview-grade knowledge.

### 4.2 — Chi-Square Test for Return Rates [PY]

Test whether late deliveries actually increase return rates (not just a coincidence):

```python
from scipy.stats import chi2_contingency

# Contingency table: late delivery × returned
contingency = pd.crosstab(df['is_late_delivery'], df['has_return'].fillna(0))
chi2, p, dof, expected = chi2_contingency(contingency)
print(f"Chi-square: {chi2:.2f}, p-value: {p:.6f}")
# p < 0.05 means late delivery is significantly associated with returns
```

### 4.3 — Effect Size Calculation [PY]

Statistical significance isn't enough — you need to show how BIG the effect is:

```python
# Cohen's d for approval time difference
def cohens_d(group1, group2):
    n1, n2 = len(group1), len(group2)
    var1, var2 = group1.var(), group2.var()
    pooled_std = ((n1-1)*var1 + (n2-1)*var2) / (n1+n2-2)
    return (group1.mean() - group2.mean()) / (pooled_std ** 0.5)

d = cohens_d(weekend, weekday)
print(f"Cohen's d: {d:.2f}")
# 0.2 = small, 0.5 = medium, 0.8 = large effect
```

### 4.4 — Statistical Summary Table [DOC] [PORTFOLIO]

Create a clean table summarizing all your tests. This goes in your report:

| Bottleneck | Test | Sample Sizes | p-value | Effect Size | Conclusion |
|---|---|---|---|---|---|
| Weekend approval delay | Mann-Whitney U | Weekend: n=X, Weekday: n=Y | <0.001 | d=0.XX (large) | Significant |
| BudgetPost winter | Mann-Whitney U | Winter: n=X, Other: n=Y | <0.001 | d=0.XX (large) | Significant |
| WH-Southeast processing | Mann-Whitney U | SE: n=X, Other: n=Y | <0.001 | d=0.XX (medium) | Significant |
| Late delivery → returns | Chi-square | n=X | <0.001 | Cramér's V=0.XX | Significant |

---

## PHASE 5: Financial Impact Quantification (Days 14–16)

**Goal:** Translate bottlenecks into dollar figures. THIS is what gets you hired. Any analyst can find a pattern. A great analyst says "this problem costs the company $X per year and here's the math."

### 5.1 — Cost of Late Deliveries [SQL]

```sql
-- Revenue lost to late-delivery-driven returns
WITH late_return_costs AS (
    SELECT
        CASE WHEN is_late_delivery = 1 THEN 'Late' ELSE 'On-Time' END AS delivery_status,
        COUNT(*) AS total_orders,
        SUM(CASE WHEN has_return = 1 THEN 1 ELSE 0 END) AS returns,
        ROUND(100.0 * SUM(CASE WHEN has_return = 1 THEN 1 ELSE 0 END) / COUNT(*), 2) AS return_rate,
        SUM(COALESCE(total_refund_amount, 0)) AS total_refunds
    FROM novamart_marts.mart_order_complete
    GROUP BY delivery_status
)
SELECT
    *,
    -- Calculate EXCESS return rate caused by lateness
    return_rate - LAG(return_rate) OVER (ORDER BY delivery_status DESC) AS excess_return_rate_pct,
    -- Estimated annual cost of excess returns
    total_refunds AS actual_refund_cost
FROM late_return_costs;
```

Then calculate: If the "excess" return rate from late deliveries is X%, and total late orders have Y revenue, the cost is approximately X% × Y in avoidable refunds.

### 5.2 — Cost of BudgetPost in Winter [SQL]

```sql
-- What would NovaMart save by switching BudgetPost orders to NationWide in winter?
WITH bp_winter AS (
    SELECT
        COUNT(*) AS orders_affected,
        AVG(hours_in_transit) AS avg_transit_hrs,
        SUM(shipping_cost) AS total_shipping_cost,
        SUM(CASE WHEN is_late_delivery = 1 THEN 1 ELSE 0 END) AS late_count,
        SUM(CASE WHEN has_return = 1 THEN COALESCE(total_refund_amount, 0) ELSE 0 END) AS return_cost
    FROM novamart_marts.mart_order_complete
    WHERE carrier = 'BudgetPost'
      AND MONTH(order_ts) IN (12, 1, 2)
),
nw_baseline AS (
    SELECT
        AVG(hours_in_transit) AS avg_transit_hrs,
        AVG(CASE WHEN is_late_delivery = 1 THEN 1.0 ELSE 0 END) AS late_rate
    FROM novamart_marts.mart_order_complete
    WHERE carrier = 'NationWide'
      AND MONTH(order_ts) IN (12, 1, 2)
)
SELECT
    bp.orders_affected,
    bp.avg_transit_hrs AS bp_avg_transit,
    nw.avg_transit_hrs AS nw_avg_transit,
    bp.late_count AS bp_late_orders,
    ROUND(bp.orders_affected * nw.late_rate) AS projected_nw_late_orders,
    bp.return_cost AS bp_winter_return_cost
FROM bp_winter bp, nw_baseline nw;
```

### 5.3 — Marketing Budget Reallocation Model [SQL]

```sql
-- If we moved TikTok's budget to Email Marketing, what would happen?
WITH channel_efficiency AS (
    SELECT
        channel,
        SUM(spend) AS total_spend,
        SUM(conversions) AS total_conversions,
        SUM(revenue_attributed) AS total_revenue,
        ROUND(SUM(revenue_attributed) / NULLIF(SUM(spend), 0), 2) AS roas,
        ROUND(SUM(conversions) / NULLIF(SUM(spend), 0) * 1000, 2) AS conversions_per_1k_spend
    FROM novamart_raw.marketing_campaigns
    WHERE spend > 0
    GROUP BY channel
)
SELECT
    channel,
    total_spend,
    roas,
    conversions_per_1k_spend,
    -- What if this channel's budget was reallocated to Email?
    ROUND(total_spend * (SELECT roas FROM channel_efficiency WHERE channel = 'Email Marketing'), 2)
        AS projected_revenue_if_email
FROM channel_efficiency
ORDER BY roas ASC;
```

### 5.4 — Build a Financial Impact Summary [DOC] [PORTFOLIO]

Create a table like this (fill in YOUR numbers):

| Bottleneck | Annual Revenue Impact | Fix Complexity | ROI Priority |
|---|---|---|---|
| Late deliveries → excess returns | $___K lost | Medium | HIGH |
| BudgetPost winter failures | $___K in avoidable refunds | Low (switch carrier) | HIGH |
| TikTok Ads budget waste | $___K underperforming | Low (reallocate) | HIGH |
| WH-Southeast slow processing | $___K delayed revenue | High (operational fix) | MEDIUM |
| Weekend approval staffing gap | $___K delayed revenue | Medium (hire/automate) | MEDIUM |
| Electronics Q4 stockouts | $___K lost sales | Medium (inventory planning) | HIGH |

**This is the slide a VP actually acts on.** Organize by ROI priority (high impact + low fix complexity = do first).

---

## PHASE 6: Dashboard & Visualization (Days 17–20)

**Goal:** Build an interactive dashboard that tells the story. Choose ONE tool: Tableau Public (free, great for portfolio), Power BI (great for Microsoft-focused employers), or Streamlit (great for Python-focused roles).

### 6.1 — Dashboard Architecture [VIZ]

Build 4 pages/tabs:

**Page 1: Executive KPI Overview**
- Total orders, revenue, profit (with YoY comparison)
- Fulfillment rate (% delivered on time)
- Average order lifecycle duration (with trend)
- Return rate (with trend)
- Filters: date range, region, category

**Page 2: Fulfillment Bottleneck Drill-Down**
- Stacked bar: time spent at each lifecycle stage (approval → processing → transit)
- Heatmap: warehouse × carrier → median delivery days
- Line chart: processing time by month (shows Q4 spike)
- Scatter: seller processing speed vs. their return rate

**Page 3: Revenue Leakage Analysis**
- Waterfall chart: gross revenue → shipping costs → returns → net revenue
- Bar chart: return rate by category and reason
- Comparison: late vs on-time delivery return rates
- Map: geographic revenue and late delivery concentration

**Page 4: Marketing ROI**
- ROAS by channel (bar chart, descending)
- Spend vs Revenue scatter (each dot = channel-month)
- Trend lines: monthly ROAS by channel (shows which channels are improving/declining)
- Budget reallocation scenario: "Move $X from TikTok to Email → projected gain"

### 6.2 — Design Principles [VIZ]

Follow these rules for a professional-looking dashboard:

1. **One insight per chart.** If you need two, use two charts.
2. **Title every chart as a finding**, not a description. Write "WH-Southeast processes orders 2.3x slower than average" not "Processing Time by Warehouse."
3. **Use consistent colors.** Pick a 5-color palette and stick to it.
4. **Annotate the key findings** directly on the charts (callout boxes, reference lines).
5. **Include a date filter** on every page.
6. **Left-to-right, top-to-bottom = most important to least important.**

---

## PHASE 7: Portfolio Presentation & Documentation (Days 21–23)

### 7.1 — README.md [DOC] [PORTFOLIO]

Your README is the landing page. Structure it like this:

```markdown
# NovaMart E-Commerce Operations: Bottleneck Analysis & Revenue Optimization

## The Problem
NovaMart, a mid-size US e-commerce company, is experiencing rising return rates, 
inconsistent delivery performance, and unclear marketing ROI. This project identifies 
the root causes, quantifies financial impact, and recommends prioritized fixes.

## Key Findings
1. **Weekend orders take 4x longer to approve** — staffing gap costs ~$XK/year
2. **BudgetPost carrier fails in winter** — 3+ extra transit days drive $XK in returns
3. **WH-Southeast processes 2.3x slower** — causing cascading delivery failures
4. **TikTok Ads ROAS is 0.XX** — worst channel, $XK/year opportunity cost
5. **Late deliveries increase clothing returns by 50%** — statistically significant (p<0.001)

## Tools & Skills Demonstrated
SQL (Snowflake) · Python (pandas, scipy, matplotlib) · Tableau · Statistical Testing · 
Dimensional Modeling · Data Quality Assessment · Financial Impact Quantification

## Dashboard
[Link or screenshot]

## How to Navigate This Repo
...
```

### 7.2 — Executive Summary Report [DOC] [PORTFOLIO]

Write a 2-3 page report structured like a consulting deliverable:

1. **Situation** (1 paragraph): What does NovaMart do, what data did you analyze?
2. **Key Findings** (5-6 bullet points with numbers): What did you discover?
3. **Root Cause Analysis** (1 paragraph per bottleneck): WHY is each problem happening?
4. **Financial Impact** (the summary table from Phase 5.4)
5. **Recommendations** (prioritized, numbered): What should the company do, in what order?
6. **Methodology & Limitations** (1 paragraph): How you did it and what could improve

### 7.3 — What Interviewers Will Ask [DOC]

Prepare answers for these questions:

- "Walk me through your approach." → Start with data quality, explain your staging pipeline, then findings.
- "How did you decide which bottleneck to focus on?" → Financial impact + fix complexity prioritization.
- "What was the hardest part?" → Data quality issues, deciding how to handle missing customer_ids.
- "What would you do differently with more time?" → A/B test recommendations, predictive model for at-risk orders, build an automated alerting pipeline.
- "Why didn't you use a t-test?" → The data is right-skewed, so I used the non-parametric Mann-Whitney U test.
- "How would you present this to a non-technical stakeholder?" → The executive summary focuses on dollars and actions, not statistics.

---

## Quick Reference: Skills This Project Demonstrates

| Skill | Where in Project | Why It Matters |
|---|---|---|
| SQL window functions | Staging layer, QUALIFY, ROW_NUMBER | Top SQL interview topic |
| CTEs and subqueries | Mart tables, financial analysis | Shows query organization |
| Data quality assessment | Phase 1 | Shows professional maturity |
| Dimensional modeling | Staging → Marts pipeline | Core analytics engineering |
| Statistical hypothesis testing | Phase 4 | Separates analysts from chart-makers |
| Financial impact estimation | Phase 5 | The #1 skill employers want |
| Dashboard design | Phase 6 | Required for every DA role |
| Technical documentation | README, exec summary | Communication = career growth |
| Git/version control | Repo structure | Basic but many candidates lack it |

---

## Estimated Timeline

| Phase | Duration | Output |
|---|---|---|
| 0: Setup | 1 day | Repo structure, Snowflake schemas |
| 1: Data Quality | 2 days | Quality report |
| 2: Staging & Marts | 3 days | Clean tables, calculated fields |
| 3: EDA | 4 days | Initial findings, visualizations |
| 4: Statistical Tests | 3 days | Validated hypotheses with p-values |
| 5: Financial Impact | 3 days | Dollar-impact summary table |
| 6: Dashboard | 4 days | Interactive 4-page dashboard |
| 7: Documentation | 3 days | README, exec summary, polished repo |
| **Total** | **~23 days** | **Complete portfolio project** |

---

## Final Advice From a Senior Analyst

**The project itself is not what gets you hired. It's the STORY you tell about it.**

Every piece of your project should answer: "So what?" Don't say "I found that WH-Southeast has a 36-hour median processing time." Say "WH-Southeast's processing bottleneck adds 20 hours to 18% of all orders, driving a 32% return rate on electronics that costs the company approximately $X per year. Switching Southeast electronics sellers to Warehouse Fulfilled operations would reduce this by an estimated 40%."

Numbers without context are noise. Numbers with business impact and a recommended action — that's analysis.

Good luck, Karen. You have everything you need.
