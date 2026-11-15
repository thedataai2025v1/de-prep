# Lesson 6.3 — Aggregates and Materialized Views

## 1. Why aggregates exist

An aggregate table or materialized view stores the result of a computation so recurring queries do not repeatedly scan and aggregate the same detailed data.

Examples:
- Daily sales by product, store, and channel.
- Monthly active users by country and plan.
- Daily trip metrics by city and driver category.
- Support SLA metrics by team and priority.
- Account balances at month end.

Aggregation can reduce scan volume and latency, but it introduces additional data products to build, refresh, validate, govern, and retire.

**Interview principle:** An aggregate is not merely a faster copy of a query. It is a contract with a specific grain, measure semantics, refresh policy, and freshness guarantee.

### Learning objectives
- Distinguish aggregate tables, materialized views, logical views, and cubes.
- Choose aggregate grain and preserve drill-down capability.
- Handle additive and non-additive measures correctly.
- Design incremental refresh and late-data correction.
- Understand query rewrite and materialized-view eligibility.
- Balance latency, cost, storage, and freshness.

## 2. Four related concepts

### Logical view
A stored query definition. Results are generally computed when queried (unless the engine provides additional optimization). It centralizes logic but does not necessarily persist the result.

### Materialized view (MV)
A persisted result derived from a query. The platform may refresh it incrementally or fully, and may automatically rewrite eligible queries to use it. Capabilities and restrictions differ by engine.

### Aggregate table
A physical table populated by a pipeline, often using SQL, Spark, dbt, or orchestration. The team controls its grain, refresh, backfill, and validation logic.

### OLAP cube / semantic aggregate
A multidimensional or semantic representation designed for analytical navigation. Modern semantic layers may provide measures and caching without a traditional cube engine.

These options overlap. The choice depends on whether the platform should manage refresh/query rewrite or the engineering team needs explicit control over the data product.

## 3. Aggregate grain

Example detailed fact:
> One row per order line.

Candidate aggregate:
> One row per business date, product, store, and sales channel.

The aggregate can support sales trends at that dimensional grain, but it cannot answer arbitrary order-line questions or drill down to a customer if customer is not retained.

Before building it, ask:
- Which dimensions must users filter or group by?
- Which dimensions are intentionally excluded?
- What is the lowest required drill-down level?
- Which measures can be safely rolled up?
- Does the aggregate need to support distinct counts?
- How fresh must it be?
- How will late changes be reflected?

An aggregate's grain is a permanent analytical constraint until the table is rebuilt or redesigned.

## 4. Additive and non-additive measures in aggregates

### Additive
Store sums and counts:
- `sales_amount_sum`
- `quantity_sum`
- `order_count`
- `discount_amount_sum`

These can generally be rolled up across dimensions when definitions and units are compatible.

### Ratios and averages
Do not store only a precomputed average if users need to roll it up. Store the components:
- `revenue_sum`
- `order_count`
- `discount_sum`
- `gross_sales_sum`

Then calculate average order value:

```sql
SUM(revenue_sum) / NULLIF(SUM(order_count), 0)
```

For conversion rate:

```sql
SUM(conversions) / NULLIF(SUM(eligible_sessions), 0)
```

### Distinct counts
Distinct counts are not generally additive. If one customer appears in two daily partitions, adding daily distinct customer counts double-counts that customer at the month level.

Options:
- Recompute exact distinct count at the requested grain.
- Use mergeable sketches (e.g., HyperLogLog) if approximation is acceptable and supported.
- Store a more detailed bridge or membership set for exact deduplication.
- Define the metric at the aggregate's fixed grain and do not imply it can be rolled up.

### Percentiles and medians
Averaging daily p95 values does not produce the monthly p95. Use raw distributions, mergeable quantile sketches, or compute from detailed data at the target grain.

### Semi-additive balances
Do not sum daily balances across dates. Use last-value/period-end logic or a defined average balance. An aggregate must preserve the temporal semantics.

## 5. Example: sales aggregate table

```sql
CREATE TABLE agg_daily_product_sales AS
SELECT
    order_date_sk,
    product_sk,
    store_sk,
    channel_sk,
    SUM(quantity)       AS quantity_sum,
    SUM(gross_amount)   AS gross_amount_sum,
    SUM(discount_amount) AS discount_amount_sum,
    SUM(net_amount)     AS net_amount_sum,
    COUNT(DISTINCT order_id) AS distinct_order_count
FROM fact_order_line
GROUP BY
    order_date_sk,
    product_sk,
    store_sk,
    channel_sk;
```

Caution: `COUNT(DISTINCT order_id)` is valid at this aggregate grain but is not additive across products if an order contains multiple products. If the business needs total orders by store/day, create a separate aggregate at date/store/channel grain or calculate from the appropriate order-level process.

This is a common principal-level issue: one aggregate may not safely answer every metric at every dimensional level.

## 6. Materialized view refresh strategies

### Full refresh
Recompute the complete result. Simple semantics, potentially expensive for large source tables.

### Incremental refresh
Process only changes since the prior refresh, where the engine supports it and the query definition is eligible. It can be faster, but correctness depends on change tracking, supported SQL constructs, and refresh semantics.

### Micro-batch refresh
A pipeline periodically recomputes recent partitions/windows. Common in lakehouse implementations where the team owns the aggregate table.

### Streaming aggregation
Continuously update state as events arrive. Requires explicit handling of event-time windows, watermarks, late data, state retention, and corrections.

### Hybrid strategy
Incrementally process the normal window, periodically reconcile or rebuild older periods, and provide a controlled backfill path.

Do not assume every materialized view supports every SQL expression or incremental update. Read the platform's eligibility rules and test with the exact definition.

## 7. Late-arriving data and correction strategy

Suppose a sale from 10 days ago arrives today. A daily aggregate for that date must change.

Possible strategies:
1. Recompute the affected date partition from the source fact.
2. Apply a signed delta if the event identity and correction semantics are reliable.
3. Use change data capture to update the aggregate.
4. Rebuild a rolling window and periodically reconcile older dates.

For updates/deletes, a simple append-only sum is insufficient. If an event amount changes from 100 to 120, the aggregate needs a +20 delta or a correct recomputation. If an event is deleted/reversed, the aggregate must remove or offset its contribution.

Keep:
- Aggregate refresh watermark.
- Source high-water mark/version.
- Last successful refresh timestamp.
- Affected business date range.
- Row counts and control totals.
- Reconciliation status.
- Definition/version of metric logic.

## 8. Query rewrite and semantic correctness

Some platforms can automatically rewrite a user query to use a materialized view if the view is compatible. The optimizer must prove that the stored result can answer the query without changing semantics.

Potential blockers:
- Missing dimensions or columns.
- Different filters.
- Unsupported functions or joins.
- Null-handling differences.
- Security policies or row-level access constraints.
- Stale or not-yet-refreshed view state.
- Non-additive measure roll-up.

Never assume an MV is being used simply because it exists. Inspect query plans or platform diagnostics.

## 9. Aggregate navigation and semantic layer

A mature analytics platform may have several aggregate grains:
- `agg_sales_daily_store`
- `agg_sales_daily_product_store`
- `agg_sales_monthly_region`
- `agg_customer_monthly_activity`

A semantic layer can select the right aggregate for a query, but it needs governed definitions:
- Metric name and formula.
- Grain and dimensional coverage.
- Valid roll-up dimensions.
- Refresh SLA and freshness timestamp.
- Currency/unit rules.
- Security and access policy.
- Owner and deprecation lifecycle.

Without this governance, aggregates become competing versions of the truth.

## 10. Cost and performance trade-offs

Aggregate tables and MVs add:
- Storage cost.
- Refresh compute.
- Pipeline complexity.
- Data freshness lag.
- Backfill and correction complexity.
- Potential duplication of business logic.
- More objects to monitor and govern.

They reduce:
- Repeated scans.
- Repeated joins and group-bys.
- Dashboard latency.
- Concurrency pressure on detailed facts.
- Sometimes overall warehouse compute cost.

Evaluate total cost of ownership:

```text
net value =
    query compute saved
  + latency / concurrency benefit
  - refresh compute
  - storage
  - engineering and operational maintenance
```

The equation is conceptual. Put real workload measurements behind each term.

## 11. Platform-specific notes

### Snowflake
Snowflake supports materialized views with platform-specific restrictions and maintenance costs. Dynamic Tables provide declarative pipeline management with target lag and refresh optimization; they are not identical to traditional materialized views. Automatic query optimization, clustering, and query acceleration may also affect the decision. Choose the object based on the exact use case and current feature rules.

### BigQuery
BigQuery supports materialized views with incremental maintenance for eligible queries and can use smart tuning/query rewrite in supported cases. Query shape, base-table changes, and supported SQL constructs affect eligibility. Aggregate tables built by scheduled queries or pipelines remain useful when logic is more complex or refresh control is needed.

### Databricks
Materialized views in supported Databricks SQL/Unity Catalog environments are managed objects with refresh behaviour and feature/runtime requirements. Many lakehouse teams also create aggregate Delta/Iceberg tables through dbt or Spark jobs. Liquid clustering can be applied to supported materialized views and tables; confirm the runtime and catalog support.

### Amazon Redshift
Redshift materialized views can store query results and may be incrementally or fully refreshed depending on query eligibility and changes. Refresh cost, query definition, and dependencies matter. Materialized views can be used for repeated joins and aggregations; inspect refresh behaviour and query plans.

### PostgreSQL
Materialized views persist a query result and are commonly refreshed explicitly. Standard refresh semantics can require substantial work; concurrent refresh has additional requirements. They are useful for moderate-scale summaries but are not automatically equivalent to incremental MVs in cloud warehouses.

## 12. Worked example: monthly active users

Definition: a user is active in a month if at least one qualifying event occurred during that calendar month.

Naïve aggregate:
```sql
SELECT
    month_start,
    country_sk,
    COUNT(DISTINCT user_sk) AS monthly_active_users
FROM fact_user_event
GROUP BY month_start, country_sk;
```

This is correct at month/country grain. But daily distinct active users cannot simply be summed to get monthly active users.

If the platform needs both daily active users and monthly active users:
- Keep separate aggregates at their defined grains.
- Or use a mergeable distinct-count sketch if approximate results are acceptable and the semantic layer exposes approximation.
- Or compute exact monthly distinct users from detailed data.

Define what counts as a qualifying event, exclude bots/test users consistently, and handle user identity merges or deletions.

## 13. Interview conversation

**Interviewer:** A dashboard takes 90 seconds because it scans a 10 TB sales fact. What do you do?

**Candidate:** I would inspect the query plan and workload first. If the dashboard repeatedly groups by date, store, and channel, I would evaluate an aggregate at that grain. I would preserve additive components such as sales sum and order count, not only precomputed ratios. I would compare a managed materialized view with a pipeline-owned aggregate table based on refresh support, SQL complexity, and operational control.

**Interviewer:** Can you sum daily distinct customers to get monthly distinct customers?

**Candidate:** No. A customer active on multiple days is counted repeatedly. I would compute the distinct count at month grain, use a mergeable sketch if approximate results are acceptable, or retain sufficient detail for exact calculation.

**Interviewer:** Sales from last week arrive late. How do you refresh the aggregate?

**Candidate:** I would track affected event dates and recompute those partitions or apply reliable deltas. I would update downstream refresh metadata, reconcile control totals, and expose the latest complete-through timestamp. For financial reporting, I would also apply the period-close and restatement policy.

**Interviewer:** How do you know the query uses the materialized view?

**Candidate:** I would inspect the actual query plan and platform diagnostics. The optimizer may reject rewrite because of unsupported constructs, missing dimensions, freshness, or semantic differences. I would benchmark the actual dashboard query rather than infer from the object's existence.

## 14. Common interview questions

1. What is an aggregate table?
2. What is a materialized view?
3. How does a logical view differ from an MV?
4. How do you choose aggregate grain?
5. Which measures can be rolled up?
6. Why are distinct counts difficult to aggregate?
7. Why can't daily p95 values be averaged into monthly p95?
8. How do you handle late-arriving facts?
9. What is incremental refresh?
10. What is query rewrite?
11. When would you choose an aggregate table over an MV?
12. How do you measure whether an aggregate is worth maintaining?

### Follow-up / top-up questions
- Can a precomputed average be safely rolled up?
- How do you calculate weighted conversion rate?
- How do you aggregate balances?
- How do you support exact versus approximate distinct counts?
- What happens when an underlying fact is updated or deleted?
- How do you handle a historical backfill?
- What if the MV refresh fails for two days?
- How do you avoid two aggregates defining revenue differently?
- What is the difference between target lag and guaranteed freshness?
- How do security policies affect aggregate reuse?
- How do you retire an aggregate that no longer has consumers?

### Principal-level probes
- Design a governed aggregate strategy for hundreds of dashboards and multiple business domains.
- How do you prevent metric duplication and inconsistent definitions?
- How would you balance exactness, latency, and cost for distinct-user analytics at petabyte scale?
- Design a refresh architecture for streaming facts with 30-day late corrections.
- How do you make aggregates resilient to schema and metric-definition evolution?
- How would you prove that an MV reduces total cost under high concurrency?
- What service-level indicators would you expose for aggregate freshness and correctness?

## 15. Practice exercise

A subscription business has a 6 TB event fact and 500 dashboards. Common dashboards ask for:
- Daily active subscribers by country and plan.
- Monthly recurring revenue by plan and region.
- 30-day retention cohorts.
- p95 API latency by endpoint.
- Exact monthly distinct paying customers for finance.

Design a set of aggregate tables or materialized views. For each, define grain, measures, refresh mode, late-data policy, exact/approximate semantics, ownership, and query routing. Explain which questions should continue to query detailed facts.

**Evaluation rubric (10 points):**
- Grain and query workload mapping: 2
- Correct measure aggregation semantics: 2
- Refresh and late-data strategy: 2
- Governance and query rewrite validation: 2
- Cost/freshness trade-offs: 2

## 16. Key takeaways

- Aggregates accelerate known query patterns but constrain dimensional drill-down.
- Define grain, measure semantics, and freshness before implementation.
- Ratios, distinct counts, percentiles, and balances need special treatment.
- Incremental refresh is platform- and query-dependent.
- Query plans, reconciliation, and total cost determine whether the optimization is successful.
