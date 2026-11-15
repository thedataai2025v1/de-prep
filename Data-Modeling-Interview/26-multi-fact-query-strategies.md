# Module 5: Fact Data Modeling — Lesson 5.3: Multi-Fact Query Strategies

---

### 1. What Are Multi-Fact Queries?

Each fact table captures one **business process** at one **grain**. Real questions span multiple facts:

| Question | Facts Needed |
|---|---|
| Revenue vs. inventory on hand | `fact_orders`, `fact_inventory` |
| Web traffic vs. purchases | `fact_page_views`, `fact_orders` |
| Ad spend vs. conversions | `fact_ad_impressions`, `fact_conversions` |

**Why interviewers ask:** Tests **grain alignment** and **conformed dimensions** — the glue that turns isolated facts into a unified truth.

---

### 2. Conformed Dimensions

A **conformed dimension** is shared by 2+ facts, using the same surrogate key and meaning everywhere.

| Concept | Explanation |
|---|---|
| **Conformed dimension** | Same key, same meaning across facts |
| **Why it matters** | Enables **drill-across** between processes |
| **Examples** | `dim_date`, `dim_product`, `dim_customer` |
| **Anti-pattern** | Separate `dim_date` per business process |

```sql
SELECT d.day_date, p.product_name,
    COALESCE(o.revenue, 0) AS revenue,
    COALESCE(i.qty_on_hand, 0) AS inventory
FROM dim_date d
JOIN fact_orders o ON d.date_key = o.date_key
JOIN dim_product p ON o.product_key = p.product_key
LEFT JOIN fact_inventory i ON d.date_key = i.date_key AND p.product_key = i.product_key;
-- Correct: date → orders (fact), orders → product (dim), inventory joined by date + product
```

**Interviewer Q&A**

1. **Q:** "What is a conformed dimension?" **A:** "A dimension shared by multiple facts with the same key and meaning. Enables cross-process analysis."

2. **Q:** "How do you know two facts can be queried together?" **A:** "Check they share at least one conformed dimension with matching keys."

3. **Q:** "When should a dimension NOT be conformed?" **A:** "When departments define it differently — household vs. individual 'customer.'"

**Senior Probes:** Retrofitting conformed dimensions? Data quality checks for consistency? Degenerate vs. conformed dimension?

---

### 3. Multiple Query Pattern (CTEs)

Run separate aggregations per fact, then join. Avoids Cartesian products and double-counting.

```sql
WITH revenue_cat AS (
    SELECT p.category, SUM(o.amount) AS revenue, COUNT(*) AS orders
    FROM fact_orders o JOIN dim_product p ON o.product_key = p.product_key
    GROUP BY p.category
),
inventory_cat AS (
    SELECT p.category, SUM(i.qty_on_hand) AS inventory
    FROM fact_inventory i JOIN dim_product p ON i.product_key = p.product_key
    GROUP BY p.category
)
SELECT COALESCE(r.category, i.category) AS category,
    COALESCE(r.revenue, 0) AS revenue,
    COALESCE(i.inventory, 0) AS inventory
FROM revenue_cat r FULL OUTER JOIN inventory_cat i ON r.category = i.category
ORDER BY revenue DESC;
```

| Benefit | Why |
|---|---|
| No double-counting | Aggregate before join |
| Flexible grain | Each CTE at its natural grain |
| Performance | Smaller intermediates; one concern per CTE |

**Interviewer Q&A**

1. **Q:** "Walk me through combining two sales facts." **A:** "Two CTEs — online and store — grouped by shared dimensions. Join on those keys."

2. **Q:** "Why FULL OUTER JOIN?" **A:** "Preserves categories in only one fact. INNER JOIN drops zero-sales or zero-inventory categories."

3. **Q:** "One fact has categories the other doesn't?" **A:** "FULL OUTER JOIN + COALESCE metrics to zero."

**Senior Probes:** When materialize CTEs as permanent tables? Scaling to 10+ facts? vs. dedicated aggregated fact?

---

### 4. Union Pattern

When facts share the **same grain** but different sources, stack them with `UNION ALL`.

```sql
SELECT date_key, product_key, customer_key, amount, quantity, 'online' AS channel
FROM fact_online_sales
UNION ALL
SELECT date_key, product_key, customer_key, amount, quantity, 'store' AS channel
FROM fact_store_sales;
```

| Use UNION ALL | Avoid |
|---|---|
| Same grain (per-transaction) | Different grains (daily vs. transactional) |
| Same dimension keys/meanings | Different business semantics |
| Channels, regions, system sources | Non-overlapping dimension references |

```sql
-- Materialize into unified fact
CREATE TABLE fact_all_sales AS
SELECT date_key, product_key, customer_key, amount, quantity, 'online' AS channel
FROM fact_online_sales UNION ALL
SELECT date_key, product_key, customer_key, amount, quantity, 'store' AS channel
FROM fact_store_sales;
```

**Interviewer Q&A**

1. **Q:** "UNION vs. JOIN?" **A:** "UNION stacks rows at the same grain. JOIN combines metrics side-by-side."

2. **Q:** "UNION vs. UNION ALL?" **A:** "Almost always UNION ALL. UNION sorts/deduplicates at high cost."

3. **Q:** "What metadata do you add?" **A:** "Source/channel column, ETL timestamps, source-specific ID if IDs differ."

**Senior Probes:** Schema drift handling? Indexing per cloud warehouse? Incremental loading on UNION-based fact?

---

### 5. Sub-Query / Correlated Pattern

When one fact **depends on another** — e.g., measuring time between events.

```sql
-- Direct JOIN
SELECT p.category, AVG(DATEDIFF(day, o.order_date, r.return_date)) AS avg_return_days
FROM fact_orders o JOIN dim_product p ON o.product_key = p.product_key
JOIN fact_returns r ON o.order_key = r.order_key
WHERE r.return_date IS NOT NULL GROUP BY p.category;

-- Correlated sub-query (scalar per row)
SELECT o.order_id, o.order_date,
    (SELECT MAX(r.return_date) FROM fact_returns r WHERE r.order_key = o.order_key)
        AS latest_return
FROM fact_orders o;
```

| Approach | When |
|---|---|
| **JOIN** | One-to-one or few-to-few |
| **Correlated sub-query** | One scalar per row (latest, max) |
| **CTE pre-aggregation** | Complex logic; avoids row-by-row |

**Interviewer Q&A**

1. **Q:** "Compute time between events in different facts?" **A:** "Join on shared key, DATEDIFF. For multiple matches, MIN/MAX or pre-aggregate in a CTE."

2. **Q:** "Correlated sub-query performance?" **A:** "Inner query per outer row = expensive. Prefer CTE + JOIN."

3. **Q:** "Dependent fact has no match?" **A:** "LEFT JOIN preserves primary rows. COALESCE date diffs."

**Senior Probes:** Modelling if dependent event is a status flag? Different refresh schedules? When should it be its own fact?

---

### 6. Drill Across vs. Drill Down

| Concept | Meaning | SQL |
|---|---|---|
| **Drill Across** | Query across multiple facts via shared dimension | JOIN/UNION through `dim_date`, `dim_product` |
| **Drill Down** | Aggregate → detail within one fact | Remove GROUP BY or change grain |

```sql
-- Drill ACROSS
WITH orders_d AS (SELECT date_key, SUM(amount) AS revenue FROM fact_orders GROUP BY date_key),
     inv_d   AS (SELECT date_key, SUM(qty_on_hand) AS stock FROM fact_inventory GROUP BY date_key)
SELECT d.day_date, o.revenue, i.stock
FROM dim_date d LEFT JOIN orders_d o ON d.date_key = o.date_key
LEFT JOIN inv_d i ON d.date_key = i.date_key;

-- Drill DOWN
SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount) FROM fact_orders GROUP BY 1;
SELECT order_id, order_date, amount FROM fact_orders WHERE month = '2024-01-01';
```

**Interviewer Q&A**

1. **Q:** "Difference?" **A:** "Drill across queries multiple facts through a shared dimension. Drill down goes aggregate → detail within one fact."

2. **Q:** "Which is harder?" **A:** "Drill across requires conformed dimensions. If dimensions aren't conformed, cross-fact analysis breaks."

3. **Q:** "Can you combine them?" **A:** "Yes. Drill across to compare revenue and returns per category, then drill into individual orders in the top category."

**Senior Probes:** How do BI tools handle each natively? Early modeling that enables drill across? Lakehouse vs. warehouse pattern?

---

### 7. Common Pitfalls

| Pitfall | Cause | Fix |
|---|---|---|
| **Double-counting** | Direct fact-to-fact JOIN inflates rows | CTE aggregate first |
| **Mismatched grains** | Transaction × daily joined | Aggregate both to same level |
| **Different time ranges** | One fact missing recent data | LEFT JOIN + COALESCE + document |
| **Non-conformed dimensions** | Same-name, different keys | Enforce conformance; mapping table |
| **NULL propagation** | NULLs inflate averages | `COALESCE(metric, 0)` for sums |

```sql
-- WRONG: Cartesian — 5 orders × 3 inventory rows = 15 rows, metrics tripled
SELECT p.category, SUM(o.amount), SUM(i.qty_on_hand)
FROM fact_orders o JOIN fact_inventory i
    ON o.date_key = i.date_key AND o.product_key = i.product_key
JOIN dim_product p ON o.product_key = p.product_key
GROUP BY p.category;

-- RIGHT: CTEs aggregate each fact first, then join
WITH order_agg AS (
    SELECT product_key, date_key, SUM(amount) AS revenue
    FROM fact_orders GROUP BY product_key, date_key
),
inv_agg AS (
    SELECT product_key, date_key, SUM(qty_on_hand) AS inventory
    FROM fact_inventory GROUP BY product_key, date_key
)
SELECT p.category, d.day_date,
    COALESCE(o.revenue, 0) AS revenue,
    COALESCE(i.inventory, 0) AS inventory
FROM dim_product p JOIN dim_date d ...
LEFT JOIN order_agg o ON p.product_key = o.product_key AND d.date_key = o.date_key
LEFT JOIN inv_agg i ON p.product_key = i.product_key AND d.date_key = i.date_key;
```

**Interviewer Q&A**

1. **Q:** "Most common mistake?" **A:** "Joining facts without aggregating first. Fix: CTEs per fact, then join on conformed keys."

2. **Q:** "Debug suspected double-counting?" **A:** "Count rows at each stage. If row count jumps at the join, grain mismatch."

3. **Q:** "Two facts with different time ranges?" **A:** "LEFT JOIN from `dim_date` to both aggregated facts, COALESCE NULLs, document periods."

**Senior Probes:** Architecture to make pitfalls impossible? Production checks for grain mismatches? Remediating diverged conformed dimensions? Multi-cloud lakehouse profile?

---

### Practice Exercise

**Rideshare quarterly review — Facts:**

```
fact_rides (ride_key, driver_key, rider_key, date_key, zone_key, fare_amount, trip_distance)
fact_cancellations (cancel_key, driver_key, date_key, zone_key, cancel_reason)
fact_incentives (incentive_key, driver_key, date_key, incentive_amount, incentive_type)
dim_driver (driver_key, driver_name, onboard_date, city)
dim_date (date_key, day_date, month, quarter, year)
dim_zone (zone_key, zone_name, city)
```

1. **Query:** Per city and month: total fare revenue, cancellation count, incentive payout. Identify conformed dimensions and explain approach.

2. **Add:** Ratio of cancellations to rides per city. Walk through SQL changes and pitfalls.

3. **Handle:** `fact_incentives` has data only from Q2; rides and cancellations go back two years. How does this affect your query?

4. **Senior:** This is now a daily ops dashboard. CTE approach or pre-aggregated summary table? Justify.

**Suggested approach:** "Use `dim_driver`, `dim_date`, `dim_zone` as conformed dimensions. Aggregate each fact independently — rides by city/month for revenue, cancellations for count, incentives for payout. FULL OUTER JOIN on (city, month) with COALESCE. Ratio: guard division by zero. Q2-only incentives: COALESCE to zero and document. For daily ops: materialize a summary table with incremental refresh."

---

*Next up: Lesson 5.4 — Bridge Tables and Many-to-Many Relationships*
---

### Related Lessons

- [Fact Table Types](./24-fact-table-types.md)
- [Additive, Semi-Additive, Non-Additive Measures](./25-additive-semi-additive-non-additive-measures.md)
- [Fact Modeling Practice](./27-fact-modeling-practice.md)
- [Analyzing Query Patterns](./08-analyzing-query-patterns.md)
