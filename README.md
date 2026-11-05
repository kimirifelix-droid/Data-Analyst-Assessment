# Tunda Box Data Analyst Assessment

This repository contains a complete, reproducible response to the fictional Tunda Box assessment. All records and amounts are invented, as stated in the prompt.

## Files

- [Assessment_Responses.md](./Assessment_Responses.md) — completed, document-style copy of the assessment questions and responses.
- [Data_Analyst_Assessment_Solutions.ipynb](./Data_Analyst_Assessment_Solutions.ipynb) — runnable Google Colab notebook with the sample data, pandas analysis, SQL examples, and statistical calculations.
- [THOUGHT_PROCESS.md](./THOUGHT_PROCESS.md) — assumptions, analytical choices, validation notes, and submission checklist.

## Submission links

All submission materials are available in this repository:

- [Assessment response document](https://github.com/kimirifelix-droid/Data-Analyst-Assessment/blob/main/Assessment_Responses.md)
- [Colab notebook source](https://github.com/kimirifelix-droid/Data-Analyst-Assessment/blob/main/Data_Analyst_Assessment_Solutions.ipynb)
- [Thought process and assumptions](https://github.com/kimirifelix-droid/Data-Analyst-Assessment/blob/main/THOUGHT_PROCESS.md)
- [Project README](https://github.com/kimirifelix-droid/Data-Analyst-Assessment/blob/main/README.md)
- [Open the notebook in Google Colab](https://colab.research.google.com/github/kimirifelix-droid/Data-Analyst-Assessment/blob/main/Data_Analyst_Assessment_Solutions.ipynb)

**Google Drive folder:** [Tunda Box Data Analyst Assessment](https://drive.google.com/drive/folders/1H1493TUIYEzENX2Y6tMsBf22_lqHK6CJ)

The Drive folder contains the assessment responses, Colab notebook, README, and thought-process document. The folder is set to **Anyone with the link — Viewer**, so reviewers can open the files without requesting access.

## Responses

### Part 1: Your experience

**Analysis I'm proud of:** _Personalize before submission._ For a study or personal project, describe a specific question you investigated, the data and method you used, what you found, and what decision or action changed as a result. I would avoid claiming a work project or business impact that I cannot substantiate.

**Conflicting numbers:** I would first align on the metric definition, population, time window, timezone, status rules, and treatment of duplicates, refunds, and missing values. Then I would trace both numbers back to their source data and query logic, reproduce them on the same small sample, and identify the exact point of divergence. I would document the agreed definition, correct the query or source issue, and share the reconciled result and caveats with the colleague.

### Part 2: SQL and Python

**S1. Orders by status.** To avoid counting the exact repeated row for order `1006` twice, first remove exact duplicate records. The deduplicated sample has **16 completed**, **2 cancelled**, and **2 refunded** orders (**20 distinct order records** total). If counting literal input rows instead, completed would be 17 and the total would be 21.

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

**S2(a). Completed revenue by city.** Normalize city names with `TRIM` and case folding, deduplicate exact repeated rows, and include only completed orders. `SUM` ignores the missing amount. The largest total is **Kisumu, KES 187,100**. This is provisional because one Kisumu order is KES 185,000 and needs verification.

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

Eldoret has no orders in this sample and therefore has no revenue row in this query.

**S2(b). Ten-minute update:** I would send: “Using completed orders, removing the exact duplicate order row and normalizing city names, Kisumu currently leads at KES 187,100. This is provisional: it includes a KES 185,000 order that is far above the other sample orders, so I’m checking that record before treating the number as final.” I would provide the query and definition with the result, rather than silently excluding the outlier or delaying a clearly caveated update.

**S3. Customers who never ordered.** Define “ordered” as having any order record, regardless of status. Customers **109 (Eldoret)** and **110 (Kisumu)** have no matching order. The sample also has order customer ID **999**, which has no matching customer.

```sql
SELECT c.customer_id, c.city
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.customer_id IS NULL
ORDER BY c.customer_id;
```

**S4. Repeat purchase rate.** Define a repeat buyer as a customer with at least two distinct, completed orders; count completed orders even if the amount is missing, and remove exact repeated rows. Four of the ten registered customers qualify (101, 102, 104, 107), so the repeat purchase rate is **4 / 10 × 100 = 40%**. The denominator is all customers in the supplied customer table, not just customers who ordered.

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

**S5. Completed revenue month over month.** Deduplicate exact records and sum non-null amounts for completed orders. Months are calendar months in 2026; the first observed month has no previous-month comparison.

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

The sharp August change is driven by the KES 185,000 order. It should be verified before drawing a business conclusion.

### Part 3: Data quality and spreadsheets

**D1. Audit**

- **Exact duplicate:** order `1006` appears twice. Check the source/ingestion process; deduplicate exact repeats by order ID for this sample and alert on duplicates. Confirm whether conflicting versions of the same ID exist before deciding which record is authoritative.
- **Suspicious amount:** completed order `1013` is KES 185,000, far above the other values. Verify against the source, currency/unit, decimal placement, and receipt; do not silently cap or drop it.
- **Missing value:** completed order `1009` has a NULL amount. Keep it as an order for order-count metrics, exclude it from revenue sums, and resolve it with the source owner.
- **Broken relationship:** customer ID `999` in orders is absent from customers. Investigate the missing customer record or incorrect ID; retain it as an exception rather than inventing a customer.
- **City formatting:** `"nairobi"` and `"Nairobi "` vary in case/whitespace. Preserve the raw value, normalize for analysis, and establish a controlled canonical city value.
- **Status and refund semantics:** confirm what completed, cancelled, and refunded mean and whether refunded revenue is reversed in this extract. Revenue here is completed gross order amount, not necessarily net recognized revenue.
- **Coverage and completeness:** the provided extract spans June–September order dates, but it is a small sample. Confirm the full reporting period, late-arriving orders, and whether every city/channel is represented before generalizing.

**D2. Spreadsheet formulas** (assuming the orders tab is named `orders`, customer ID is in `B`, status in `D`, amount in `E`, and payment method in `F`):

- Total completed M-Pesa revenue: `=SUMIFS(orders!E:E,orders!D:D,"completed",orders!F:F,"M-Pesa")`
- Add the matching customer city to orders row 2 (for example, in `G2`): `=IFERROR(XLOOKUP(B2,customers!A:A,customers!B:B,"Unknown"),"Unknown")`  
  If XLOOKUP is unavailable: `=IFERROR(VLOOKUP(B2,customers!A:B,2,FALSE),"Unknown")`

These formulas operate on the literal sheet data; deduplicate order IDs first if the intended total is one record per order.

**D3. Mean or median?** I would report the **median** as the typical completed order value because the KES 185,000 value is an extreme outlier that pulls the mean upward, and there is also a missing completed amount. I would investigate the outlier and missing value, report the sample size and a clear definition, and optionally include the mean alongside the median once data quality is understood. In this deduplicated sample, 15 completed orders have non-null amounts; the median is **KES 2,400**, while the mean is **KES 14,540**.

### Part 4: Statistics

**T1(a). P-value:** A p-value is the probability, assuming there is truly no effect (and the test assumptions hold), of seeing results at least as far from no-effect as the result we observed. It is not the probability that the no-effect explanation is true, nor does it tell us whether an effect is important in practice.

**T1(b). SMS offers and spending:** The observation alone does not establish that SMS caused higher spending. Customers who receive offers may differ beforehand (for example, prior spending or engagement), and those differences could explain the gap. A randomized holdout with comparable eligibility, a pre-defined outcome/window, and checks for delivery and spillover would provide stronger causal evidence; otherwise, use careful adjustment and describe the result as an association.

**T2. A/B test recommendation:** Conversion is 5.0% (50/1,000) for the current checkout and 6.0% (60/1,000) for the new checkout: a **1 percentage-point absolute increase** and a **20% relative increase**. These counts do not give strong evidence that the new checkout is better: a two-sided two-proportion z-test gives approximately **p = 0.33**, and the approximate 95% confidence interval for the absolute change is **-1.0 to +3.0 percentage points**. I would not ship solely on this result; check randomization, instrumentation, and guardrails, then continue or design a sufficiently powered test with a pre-agreed decision rule.

### Part 5: Business case

**T1(a). Falling conversion analysis**

| Channel | September conversion | October conversion |
|---|---:|---:|
| Organic | 2,400 / 40,000 = 6.00% | 2,318 / 38,000 = 6.10% |
| Paid Social | 200 / 10,000 = 2.00% | 630 / 30,000 = 2.10% |
| All traffic | 2,600 / 50,000 = 5.20% | 2,948 / 68,000 = 4.34% |

Both channels improved slightly, while the overall rate fell because Paid Social (the lower-converting channel) grew from 20.0% to 44.1% of sessions. This is a channel-mix/composition effect, not evidence from these totals that Paid Social conversion deteriorated. I would check comparable attribution windows and tracking, campaign/ad/landing-page mix, device/geography/audience, new versus returning visitors, session quality, spend/CAC and incremental value, and statistical uncertainty.

**T1(b). Recommendation (two sentences):** I would not pause Paid Social based on this table alone: its conversion rate rose from 2.0% to 2.1%, and Organic rose from 6.0% to 6.1%; the blended decline is explained by a larger share of lower-converting Paid Social sessions. I recommend checking campaign-level quality, attribution/tracking, spend and incremental economics before changing budget, while monitoring the channel mix and conversion by segment.

### Part 6: Questions for the team

1. What are the most important decisions this analyst would help the team make in the first three months?
2. How do Product, Growth, and Operations agree on metric definitions and communicate analytical uncertainty?
3. What data tools and quality checks are currently in place, and where would you most like the new analyst to improve them?

## Submission note

The experience response is intentionally a prompt for your own truthful example. Personalize it, add your name/date if needed, upload [Assessment_Responses.md](./Assessment_Responses.md) to Google Drive, and add its share link above before submitting.
