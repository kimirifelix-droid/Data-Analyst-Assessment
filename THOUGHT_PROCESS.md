# Thought process and assumptions

## Scope and reproducibility

The exercise gives only a small, invented extract. All numeric answers describe this sample, not the whole Tunda Box business. The notebook embeds the given rows so it runs in Colab without private data, a database connection, or file uploads.

## Data preparation decisions

1. **Exact duplicate handling:** order `1006` has two identical rows. For order-level reporting I collapse exact duplicate rows with `SELECT DISTINCT` / pandas `drop_duplicates()`. If records sharing an ID differed, the extract has no update timestamp or authoritative-version field, so I would flag and resolve those records rather than arbitrarily selecting one.
2. **City normalization:** trim whitespace and compare case-insensitively while preserving original values for audit. This combines `Nairobi`, `nairobi`, and `Nairobi ` for analysis.
3. **Order status:** completed revenue includes only `completed`; cancelled and refunded rows are excluded. This assumes `amount_kes` is a gross completed-order amount and that no separate refund ledger is needed.
4. **Missing amount:** NULL is excluded from revenue by SQL `SUM` and pandas numeric aggregation, but order `1009` still counts as a completed order for repeat-purchase calculations.
5. **Customer denominator:** repeat purchase rate is customers with at least two distinct completed orders divided by all 10 listed customers. This is one defensible definition; active-customer denominators or any-status repeat rates would answer different questions.
6. **Month-over-month:** use calendar months present in the sample, June through September 2026, and compare each month with the immediately previous observed month. The input has no May orders, so June has no comparison value.
7. **Statistical test:** use an unadjusted two-sided two-proportion z-test as a quick approximation. The result is not proof of no effect. A production decision should account for the experiment design, power, pre-specified metrics, guardrails, and practical impact.

## Validation and interpretation

- Deduplicated status counts reconcile to 20 orders: 16 completed, 2 cancelled, 2 refunded.
- The customer table has 10 unique IDs; order customer ID 999 is unmatched, and customer IDs 109 and 110 have no orders.
- Four customers have at least two distinct completed orders, giving 40%.
- The 15 completed records with non-null amounts sum to KES 218,100; their mean is KES 14,540 and median is KES 2,400. The KES 185,000 value explains the mean/median gap and should be investigated.
- The completed city sums and monthly totals use the same deduplicated, completed-only, non-null revenue logic.
- In the A/B example, 5% versus 6% is +1 percentage point (20% relative), but the approximate two-sided p-value is 0.33.
- In the channel example, October overall conversion is 2,948 / 68,000 = 4.3353%, which rounds to 4.3%. Paid Social share of sessions rises from 20.0% to 44.1%, while both segment conversion rates rise slightly.

## Before submitting

- Replace the Part 1 experience prompt with a genuine example from your study, work, or personal projects. Do not claim experience or impact that is not yours.
- Add your name and date to the response document if desired.
- Upload `Assessment_Responses.md` to your Google Drive and replace the Drive-link placeholder in `README.md`. The local environment does not have Drive authorization, so it cannot create or share that Drive copy.
- Open the notebook in Google Colab and run all cells top-to-bottom. Confirm that all displayed output agrees with the written answers.
