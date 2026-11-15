# Lesson 5.2 — Periodic Snapshot Fact Tables

## 1. Purpose

A periodic snapshot fact table records the state or summarized performance of a business entity at regular, predictable intervals: daily inventory, month-end account balance, weekly subscription status, or daily active device count.

The defining feature is not merely that a table has a date column. Its grain includes a snapshot period or point in time, and the business expects a row for each relevant entity-period combination—even when there was no activity, depending on the population contract.

### Learning objectives
- Declare snapshot grain and population.
- Distinguish flow measures from stock/balance measures.
- Handle semi-additive facts correctly.
- Build dense snapshots and missing-period rows.
- Design opening/closing balances, restatements, and late-arriving events.
- Explain snapshot design and cost trade-offs in interviews.

## 2. Grain and population

Example grain:

> One row per product, warehouse, and business date, representing end-of-day inventory position.

The grain includes the snapshot date. Do not say “one row per product” when the table stores daily inventory.

A critical design question is the **population**:
- Every product ever stocked?
- Every active product at each warehouse?
- Only product-warehouse pairs with positive stock?
- Every account open at month end, including zero-balance accounts?

The population rule determines whether the snapshot is dense or sparse. A dense snapshot intentionally includes entities with no activity or zero values. A sparse snapshot includes only entities that meet a condition.

## 3. Periodic snapshot versus transaction fact

| Transaction fact | Periodic snapshot |
|---|---|
| One row per event | One row per entity-period |
| Event occurs at irregular times | Snapshot cadence is predictable |
| Captures what happened | Captures state or period summary |
| Often sparse | Often dense by design |
| Usually append-only | Usually append-only per period, but restatements can occur |
| Example: stock movement | Example: end-of-day stock balance |

A transaction fact can derive a snapshot, but the snapshot does not necessarily preserve the underlying event detail.

## 4. Example: daily inventory

```sql
CREATE TABLE fact_inventory_daily (
    snapshot_date_sk INTEGER NOT NULL,
    product_sk       BIGINT NOT NULL,
    warehouse_sk     BIGINT NOT NULL,
    opening_qty      DECIMAL(18, 3),
    received_qty     DECIMAL(18, 3),
    shipped_qty      DECIMAL(18, 3),
    adjustment_qty   DECIMAL(18, 3),
    closing_qty      DECIMAL(18, 3) NOT NULL,
    inventory_value  DECIMAL(18, 2),
    load_ts          TIMESTAMP NOT NULL
);
```

Grain: one row per product, warehouse, and snapshot date.

Reconciliation identity (subject to business definitions):

```text
closing_qty = opening_qty
            + received_qty
            - shipped_qty
            + adjustment_qty
```

If `shipped_qty` is already signed negative, the formula changes. Define sign conventions and units centrally.

## 5. Semi-additive measures

Balances are typically additive across entities at one point in time but not across time.

Valid:
```sql
SELECT snapshot_date_sk, SUM(closing_qty)
FROM fact_inventory_daily
GROUP BY snapshot_date_sk;
```

Usually invalid:
```sql
SELECT product_sk, SUM(closing_qty)
FROM fact_inventory_daily
GROUP BY product_sk;
```

The second query adds the same stock across multiple days. For a period-end report, select the closing snapshot for the period. For an average inventory metric, use an explicitly defined average over daily or sampled balances.

### Common balance aggregations

- **End-of-period balance:** latest valid snapshot in the period.
- **Average daily balance:** average of daily balances over the period.
- **Peak balance:** maximum balance in the period.
- **Days of inventory:** requires a defined demand/consumption denominator and time window.
- **Net movement:** sum of transaction movements, not sum of balances.

A measure's aggregation rule belongs in the semantic contract, not only in analyst tribal knowledge.

## 6. Zero activity versus missing data

A zero is a valid business value. A missing row may mean:
- Entity did not exist yet.
- Entity was inactive or out of scope.
- Snapshot job failed.
- Source data was unavailable.
- The balance is unknown.
- The balance is genuinely zero but no row was generated.

Do not automatically convert missing rows to zero. Establish the population and completeness contract first.

A common pattern is:
1. Generate the expected entity-period population.
2. Aggregate movements for the period.
3. Carry forward or calculate closing balances.
4. Join the results to the expected population.
5. Distinguish true zero from unknown/unavailable.

## 7. Building snapshots from transaction movements

Suppose a warehouse stores all inventory movements in `fact_inventory_movement`. To calculate closing stock:

```sql
WITH daily_movement AS (
    SELECT
        movement_date,
        product_sk,
        warehouse_sk,
        SUM(quantity_delta) AS net_movement
    FROM fact_inventory_movement
    GROUP BY movement_date, product_sk, warehouse_sk
),
running_balance AS (
    SELECT
        movement_date,
        product_sk,
        warehouse_sk,
        SUM(net_movement) OVER (
            PARTITION BY product_sk, warehouse_sk
            ORDER BY movement_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS closing_qty
    FROM daily_movement
)
SELECT *
FROM running_balance;
```

This simplified example assumes a reliable opening baseline and complete movement history. In production:
- Include opening balances at the start of the history.
- Generate dates with no movements.
- Define handling of late or corrected movements.
- Use a calendar table and expected product-warehouse population.
- Consider periodic checkpoints so every daily snapshot does not rescan all historical movements.

### Efficient incremental strategy

Maintain:
- A trusted prior closing balance.
- New movements since the last successful watermark.
- A recalculation window for late arrivals.
- A reconciliation check against the source inventory system.

If an old movement changes, all subsequent balances for the affected product/warehouse may need recomputation until the next trusted checkpoint.

## 8. Snapshot cadence and calendar semantics

Daily, weekly, monthly, and fiscal-period snapshots answer different questions. Clarify:
- Is the snapshot at calendar midnight, business close, or a source-defined cutoff?
- Which timezone defines the business day?
- Does “month-end” mean last calendar day or last business day?
- Are weekends and holidays included?
- How are entities opened or closed mid-period treated?

For global operations, a single UTC date may not match each region's local business day. Store the relevant business date key and preserve timestamp/timezone context when necessary.

## 9. Periodic snapshots for SaaS subscriptions

A monthly subscription snapshot might have grain:

> One row per subscription account and month-end date, representing the subscription state and balances at that month end.

Possible facts:
- Monthly recurring revenue (MRR) at month end.
- Active seat count.
- Outstanding balance.
- Usage to date in the month (a period flow, not an end-of-day stock).
- Days past due.

Be precise: MRR at month end is a point-in-time measure; usage during the month is a period measure. Both can live in one snapshot only if their semantics and grain are clearly defined, but separate fact tables may be clearer.

## 10. Restatements, late-arriving data, and snapshots

A snapshot can be restated when source events arrive late or are corrected. Choose:
- Immutable published snapshots plus correction/revision records.
- Rebuild affected periods.
- Maintain `snapshot_version` or `as_of_load_ts`.
- Publish current corrected view while retaining audit history.

Do not overwrite month-end financial snapshots without an agreed close/reopen policy. Financial periods may have formal locking and restatement rules.

## 11. Volume and storage trade-offs

A dense snapshot can grow quickly:

```text
rows = entity_count × number_of_periods × relevant dimensional combinations
```

For example, 5 million active accounts × 365 daily snapshots yields up to 1.825 billion rows per year before other dimensions.

Mitigation options:
- Choose the cadence that matches actual decisions.
- Snapshot only the required population.
- Use monthly rather than daily snapshots if daily history is not needed.
- Keep transaction movements as the detailed source of truth.
- Use partitioning/clustering aligned with common filters and platform capabilities.
- Retain detailed snapshots for a limited period and preserve longer-term aggregates if appropriate.
- Use sparse snapshots only when absence semantics are unambiguous.

## 12. Interview conversation

**Interviewer:** Design an inventory fact table for daily reporting.

**Candidate:** I would declare the grain as one row per product, warehouse, and business date, at a defined end-of-day cutoff. I would confirm the population: whether every active product-warehouse combination should appear even if no movement occurred. The measures could include opening, received, shipped, adjusted, and closing quantity, plus inventory value.

**Interviewer:** Can we sum closing quantity across dates?

**Candidate:** No. Closing quantity is semi-additive. We can sum across products and warehouses for one snapshot date, but across dates we need a point-in-time selection or an explicitly defined average/maximum calculation.

**Interviewer:** What if a product has no movement today?

**Candidate:** If it remains in the snapshot population, it should still have a row with the carried-forward closing balance. A missing row should not be interpreted as zero unless the population and completeness contract explicitly says so.

**Interviewer:** A receipt from three days ago arrives late. What happens?

**Candidate:** It changes the historical movement total and therefore may change the closing balance for that date and subsequent dates until the next checkpoint. I would recalculate the affected product-warehouse timeline, publish a controlled restatement, and reconcile it to the source.

## 13. Common interview questions

1. What is a periodic snapshot fact table?
2. What does dense versus sparse snapshot mean?
3. Why are balance measures semi-additive?
4. How do you model daily inventory?
5. How do you handle days with no activity?
6. How do you distinguish zero from missing?
7. How do you calculate month-end balance?
8. What is the difference between a periodic snapshot and a transaction fact?
9. How do you handle late-arriving movements?
10. How do you estimate snapshot table growth?

### Follow-up / top-up questions

- Should a snapshot contain every entity even if its balance is zero?
- How do you define active population at month end?
- What happens when an account opens halfway through a month?
- How do you calculate average daily balance?
- How do you support multiple time zones?
- How do you rebuild snapshots efficiently after a late correction?
- Would you store both transaction movements and daily snapshots?
- What if the source balance does not equal opening plus movements?
- How do you preserve closed financial periods?
- When is a monthly snapshot preferable to a daily snapshot?

### Principal-level probes

- Design a snapshot architecture for tens of millions of accounts.
- How do you balance dense coverage against storage and compute cost?
- How would you guarantee that all consumers see the same “as of” cutoff?
- How do you support both operational freshness and finance-certified month-end reporting?
- What is your backfill and restatement strategy?
- Which quality metrics would you monitor for snapshot completeness?

## 14. Practice exercise

Design a daily account-balance snapshot for a digital bank:
- Accounts can open and close during a month.
- Transactions can be backdated.
- Balances exist in multiple currencies.
- Finance closes monthly periods.
- Analysts need end-of-day and average daily balances.

Specify grain, population, measures, FX policy, cutoff/timezone, late-event strategy, restatement policy, and SQL for month-end balances.

**Evaluation rubric (10 points):**
- Grain and population: 2
- Correct semi-additive treatment: 2
- Zero/missing semantics: 2
- Late data and restatement design: 2
- Scale and governance trade-offs: 2

## 15. Key takeaways

- A periodic snapshot represents an entity's state or period performance at a regular cadence.
- Grain includes the snapshot period and entity population.
- Balances are generally semi-additive across time.
- Zero, missing, inactive, and unknown are different states.
- Late movements can affect multiple future snapshots; define recomputation and close policies.
