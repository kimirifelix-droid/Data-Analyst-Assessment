# Data Analyst Assessment — Completed Responses

**Name:** _Add your name_  
**Date:** _Add submission date_

> All data is invented. Answers below are based only on the sample in the assessment. Assumptions are stated where a definition could vary.
>
> **Submission references:** [Project README](https://github.com/kimirifelix-droid/Data-Analyst-Assessment/blob/main/README.md) · [Colab notebook](https://colab.research.google.com/github/kimirifelix-droid/Data-Analyst-Assessment/blob/main/Data_Analyst_Assessment_Solutions.ipynb) · [Thought process](https://github.com/kimirifelix-droid/Data-Analyst-Assessment/blob/main/THOUGHT_PROCESS.md)

## Part 1: Your experience

### Your analysis

One analysis I’m proud of from this assessment project used the fictional Tunda Box traffic data to answer whether Paid Social was causing the blended conversion rate to fall. I calculated conversion separately by channel and compared session shares across September and October. Organic conversion rose from 6.0% to 6.1%, and Paid Social rose from 2.0% to 2.1%, while Paid Social’s share of sessions grew from 20.0% to 44.1%; that lower-converting channel mix explains the drop in overall conversion from 5.2% to 4.34%. The finding changed my recommendation in the exercise: rather than pausing Paid Social based on the aggregate rate, I recommended investigating campaign quality and unit economics first. This was an analysis of invented assessment data, not a real business intervention.

### Conflicting numbers

I would first align on the metric definition, population, time window, timezone, status rules, and treatment of duplicates, refunds, and missing values. Then I would trace both numbers back to their source data and query logic, reproduce them on the same small sample, and identify the exact point of divergence. I would document the agreed definition, correct the query or source issue, and share the reconciled result and caveats with the colleague.

## Part 2: SQL and Python

### S1. Orders by status

The sample has an exact duplicate row for order `1006`. After removing exact duplicate records, there are **16 completed**, **2 cancelled**, and **2 refunded** orders (20 distinct order records). Literal row counts before deduplication would show 17 completed orders and 21 rows total.

```sql
WITH clean_orders AS (
    SELECT DISTINCT order_id, customer_id, order_date, status,
                    amount_kes, payment_method
    FROM orders
)
SELECT status, COUNT(*) AS order_count
FROM clean_orders
GROUP BY status
ORDER BY status;
```

| Status | Orders |
|---|---:|
| cancelled | 2 |
| completed | 16 |
| refunded | 2 |

### S2. Revenue by city

#### (a)

Trim whitespace and normalize case, count completed orders only, and remove exact duplicate rows. SQL `SUM` ignores NULL amount. **Kisumu** has the highest observed completed revenue: **KES 187,100**. This is provisional because it includes a KES 185,000 order that is unusually large for this sample.

```sql
WITH clean_orders AS (
    SELECT DISTINCT order_id, customer_id, order_date, status,
                    amount_kes, payment_method
    FROM orders
)
SELECT LOWER(TRIM(c.city)) AS city,
       SUM(o.amount_kes) AS completed_revenue_kes
FROM clean_orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.status = 'completed'
GROUP BY LOWER(TRIM(c.city))
ORDER BY completed_revenue_kes DESC;
```

| Normalized city | Completed revenue (KES) |
|---|---:|
| Kisumu | 187,100 |
| Nairobi | 20,650 |
| Mombasa | 5,150 |
| Nakuru | 3,300 |

Eldoret has no orders in this sample and does not appear in this result.

#### (b)

I would send: “Using completed orders, removing the exact duplicate order row and normalizing city names, Kisumu currently leads at KES 187,100. This is provisional: it includes a KES 185,000 order that is far above the other sample orders, so I’m checking that record before treating the number as final.” I would provide the query and definition with the result.

### S3. Customers who never ordered

Assuming any order record counts as having ordered, customers **109 (Eldoret)** and **110 (Kisumu)** have never ordered.

```sql
SELECT c.customer_id, c.city
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.customer_id IS NULL
ORDER BY c.customer_id;
```

The reverse relationship also has an exception: order customer ID `999` is missing from the customers table.

### S4. Repeat purchase rate

Define a repeat buyer as a customer with at least two distinct completed orders. Count a completed order even if its amount is missing, remove exact repeated rows, and divide by all registered customers. Customers 101, 102, 104, and 107 meet the definition: **4 / 10 × 100 = 40%**.

```sql
WITH clean_orders AS (
    SELECT DISTINCT order_id, customer_id, order_date, status,
                    amount_kes, payment_method
    FROM orders
),
completed_per_customer AS (
    SELECT customer_id, COUNT(DISTINCT order_id) AS completed_orders
    FROM clean_orders
    WHERE status = 'completed'
    GROUP BY customer_id
)
SELECT 100.0 * COUNT(*) FILTER (WHERE COALESCE(x.completed_orders, 0) >= 2)
       / NULLIF(COUNT(*), 0) AS repeat_buyer_pct
FROM customers c
LEFT JOIN completed_per_customer x ON x.customer_id = c.customer_id;
```

### S5. Month-on-month revenue

Use calendar months, completed orders, deduplicated exact rows, and non-null amounts. The first observed month has no previous-month value.

```sql
WITH clean_orders AS (
    SELECT DISTINCT order_id, customer_id, order_date, status,
                    amount_kes, payment_method
    FROM orders
),
monthly AS (
    SELECT DATE_TRUNC('month', order_date)::date AS month,
           SUM(amount_kes) AS completed_revenue_kes
    FROM clean_orders
    WHERE status = 'completed'
    GROUP BY DATE_TRUNC('month', order_date)
),
with_previous AS (
    SELECT month, completed_revenue_kes,
           LAG(completed_revenue_kes) OVER (ORDER BY month) AS previous_month_revenue
    FROM monthly
)
SELECT month, completed_revenue_kes,
       100.0 * (completed_revenue_kes - previous_month_revenue)
         / NULLIF(previous_month_revenue, 0) AS mom_change_pct
FROM with_previous
ORDER BY month;
```

| Month | Completed revenue (KES) | Change vs. previous month |
|---|---:|---:|
| 2026-06 | 9,000 | — |
| 2026-07 | 6,800 | -24.44% |
| 2026-08 | 192,000 | +2,723.53% |
| 2026-09 | 10,300 | -94.64% |

August is dominated by the KES 185,000 order; validate it before interpreting the trend.

## Part 3: Data quality and spreadsheets

### D1. Audit the data

- **Duplicate order:** order `1006` is present twice with identical details. Investigate the source/ingestion cause and deduplicate exact repeats for this analysis. Flag same-ID records with differing details for review; do not choose a version arbitrarily.
- **Possible outlier:** order `1013` is KES 185,000, much higher than all other listed amounts. Verify the source, currency/unit, decimal placement, and receipt; do not silently drop it.
- **Missing amount:** order `1009` is completed but has NULL amount. Exclude it from revenue sums, keep it in completed-order counts, and resolve it with the source owner.
- **Orphan customer ID:** order `1020` refers to customer `999`, absent from the customer table. Investigate the ID or missing customer record.
- **Inconsistent city formatting:** `Nairobi`, `nairobi`, and `Nairobi ` should be normalized for grouping while preserving the original values and fixing upstream controlled values.
- **Status and refunds:** confirm status definitions and whether refunds are separately reversed. The analysis treats completed order amounts as gross completed revenue, not net recognized revenue.
- **Sample coverage:** verify completeness, reporting dates, late-arriving records, and coverage of all relevant customers/cities before making company-wide claims.

### D2. Spreadsheet formulas

Assuming the orders worksheet is named `orders`, with status in D, amount in E, payment method in F, and customer ID in B:

- Total completed M-Pesa revenue: `=SUMIFS(orders!E:E,orders!D:D,"completed",orders!F:F,"M-Pesa")`
- Add each order's customer city in `G2`: `=IFERROR(XLOOKUP(B2,customers!A:A,customers!B:B,"Unknown"),"Unknown")`
- If XLOOKUP is unavailable: `=IFERROR(VLOOKUP(B2,customers!A:B,2,FALSE),"Unknown")`

The revenue formula operates on raw rows; deduplicate order IDs first if the intended measure counts one record per order.

### D3. Mean or median?

I would use the **median** to represent a typical completed order value because the KES 185,000 observation greatly increases the mean. After removing the exact duplicate, 15 completed orders have a non-null amount: their median is **KES 2,400** and mean is **KES 14,540**. This overall mean includes order `1020` (customer ID `999`); city-level revenue omits that unmatched order because the query joins to customers. I would investigate the outlier and disclose the sample size and missing value, and consider showing both statistics once the amount is verified.

## Part 4: Statistics

### T1. Concepts in plain language

#### (a) P-value

A p-value is the probability, assuming there is truly no effect and the test assumptions hold, of seeing results at least as far from no-effect as the result observed. It is not the probability that the no-effect explanation is true, and it does not measure whether an effect matters commercially.

#### (b) SMS offers and spending

No. The observed difference alone does not show that SMS caused higher spending; people receiving offers may already differ in engagement or spending. Randomly assign eligible customers to offer and holdout groups, define the outcome and time window in advance, and check delivery and spillover to estimate causal impact more reliably.

### T2. Read an A/B test

Current conversion is **5.0%** (50/1,000), and new-checkout conversion is **6.0%** (60/1,000): **+1 percentage point absolute**, or **+20% relative**. With only 1,000 users per arm, the approximate two-sided two-proportion z-test gives **p ≈ 0.33** and a 95% confidence interval of roughly **-1.0 to +3.0 percentage points** for the absolute difference. I would not ship based on this result alone; verify randomization and instrumentation, review guardrails, and continue/design a sufficiently powered test with a pre-agreed stopping and decision rule.

## Part 5: Business case — Falling conversion

### (a) Work through the numbers

| Channel | September | October |
|---|---:|---:|
| Organic | 2,400 / 40,000 = **6.00%** | 2,318 / 38,000 = **6.10%** |
| Paid Social | 200 / 10,000 = **2.00%** | 630 / 30,000 = **2.10%** |
| Total | 2,600 / 50,000 = **5.20%** | 2,948 / 68,000 = **4.34%** |

Both channels improved slightly, but Paid Social (the lower-converting channel) grew from 20.0% to 44.1% of sessions. That mix shift explains the lower blended conversion; these aggregates do not show Paid Social conversion getting worse. I would check attribution and tracking comparability, campaign/ad/landing-page mix, device/geography/audience, new versus returning visitors, session quality, spend/CAC, incremental value, and statistical uncertainty.

### (b) Recommendation (no more than two sentences)

I would not pause Paid Social based on this table alone: its conversion rate rose from 2.0% to 2.1%, and Organic rose from 6.0% to 6.1%; the blended decline reflects a larger share of lower-converting Paid Social sessions. I recommend checking campaign-level quality, attribution/tracking, spend and incremental economics before changing budget, while monitoring conversion by channel and segment.

## Part 6: Your questions

1. What are the most important decisions this analyst would help the team make in the first three months?
2. How do Product, Growth, and Operations agree on metric definitions and communicate analytical uncertainty?
3. What data tools and quality checks are currently in place, and where would you most like the new analyst to improve them?
