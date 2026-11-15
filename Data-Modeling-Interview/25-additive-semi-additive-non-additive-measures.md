# Module 5: Fact Data Modeling
## Lesson 5.2 — Additive, Semi-Additive, and Non-Additive Measures

---

### What Are Measure Additivity Classes?

In dimensional modeling, every numeric column in a fact table is a **measure** — a quantifiable piece of data about a business event. But not every measure behaves the same way when you aggregate it. The way a measure responds to **SUM**, **AVG**, **MIN/MAX**, or other aggregation functions depends on its **additivity class**.

| Class | Can Sum Across All Dimensions? | Can Sum Across Some Dimensions? | Can Sum Across Any Dimension? |
|---|---|---|---|
| **Additive** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Semi-Additive** | ❌ No | ✅ Yes | ✅ Yes (across non-temporal) |
| **Non-Additive** | ❌ No | ❌ No | ❌ No |

**Why interviewers care:** Identifying additivity correctly is the single best signal that you understand *what the data means*, not just how to write SQL. A candidate who blindly sums account balances or discount rates has demonstrated a fundamental modelling gap — one that produces wrong business answers in production.

> **Key insight for interviews:** Always ask *"What happens to this number when I SUM it across time, across products, across customers?"* before declaring any measure.

---

### 1. Additive Measures (Most Common)

**Definition:** An additive measure can be summed meaningfully across *all* dimension hierarchies in the fact table — time, product, customer, geography, channel, etc.

**Examples:**

| Measure | Why It's Additive |
|---|---|
| `revenue` / `sales_amount` | Sum across any dimension gives a valid total |
| `cost` / `total_cost` | Sums correctly at every level of granularity |
| `units_sold` / `quantity` | 10 units in January + 15 in February = 25 total |
| `clicks` / `impressions` | Each click is a countable, independent event |
| `duration_seconds` | 120s + 180s = 300s total duration |
| `refunded_amount` | Sums across any grouping |

**SQL verification:**

```sql
-- Additive measure: safe to SUM across any combination
SELECT
    d_year,
    d_month,
    d_product_category,
    SUM(f.revenue)      AS total_revenue,
    SUM(f.units_sold)   AS total_units
FROM fact_orders f
JOIN dim_date d ON f.date_sk = d.date_sk
JOIN dim_product p ON f.product_sk = p.product_sk
GROUP BY d_year, d_month, d_product_category;
```

Each subtotal and grand total is meaningful. Sum across months → quarterly total. Sum across categories → overall revenue. Sum everything → company-wide total. All correct.

**Interviewer Questions You'll Face**

> **Q1: "What makes a measure additive?"**
>
> **A:** A measure is additive when summing it across any combination of dimensions yields a business-meaningful result. Revenue is the classic example: whether you sum by product, by month, by customer, or by region, the total always represents a valid aggregate.

> **Q2: "Give me three additive measures from an e-commerce platform."**
>
> **A:** Revenue (total sales amount), units sold (quantity), and refund amount. Each one sums correctly across time, product, customer, and channel dimensions.

> **Q3: "Is profit an additive measure?"**
>
> **A:** Yes — total profit (revenue minus cost) is additive because both revenue and cost are additive, and their difference preserves additivity across any dimension.

**Top-Up Questions (Senior-Level Probes)**

> **Q1: "What happens when you multiply two additive measures?"**
>
> **A:** The result is generally *non-additive*. For example, `revenue × conversion_rate` cannot be summed directly because the product of two aggregates doesn't equal the aggregate of the products.

> **Q2: "Can you have a measure that's additive at one grain but not at another?"**
>
> **A:** Yes. For example, `order_total` is additive at the order-line grain, but if stored at the order-header grain and repeated per line, summing it across lines double-counts. Grain determines additivity.

> **Q3: "When would SUM be the wrong aggregation even for an additive measure?"**
>
> **A:** When the business question requires a different semantics — e.g., "how many orders had revenue > $100?" requires a COUNT with a filter, not SUM. The measure is additive, but the business question isn't about totals.

---

### 2. Semi-Additive Measures (Trickiest)

**Definition:** A semi-additive measure can be summed across *some* dimensions but **not all**. The most common case: summable across product, customer, or location — but **NOT across time**.

**Why not across time?** Because these measures represent a *state* or *balance* at a point in time, not a flow. Adding balances from two different days doesn't give you a meaningful "total balance" — it gives you garbage.

**Examples:**

| Measure | Additive Across... | NOT Additive Across... |
|---|---|---|
| `account_balance` | accounts, branches, currencies | time (days/months) |
| `inventory_quantity` | products, warehouses | time (daily snapshots) |
| `headcount` | departments, locations | time |
| `stock_price` / `closing_price` | symbols, exchanges | time |
| `temperature` | locations, sensors | time |

**The Anti-Pattern — what happens when you SUM a semi-additive measure across time:**

```sql
-- WRONG: This sums 30 daily balances and gives a meaningless number
SELECT SUM(closing_qty)
FROM fact_inventory_daily
WHERE product_sk = 101;

-- Returns 15,430 — but the warehouse never held 15,430 units!
-- Each day's closing_qty is a snapshot; summing them is like
-- adding up 30 thermometer readings to get a "total temperature"
```

**Correct Aggregation Strategies:**

| Business Question | Correct Aggregation |
|---|---|
| "What was the average inventory in January?" | `AVG(closing_qty)` for the month |
| "What was inventory at month-end?" | `LAST_VALUE(closing_qty) OVER (ORDER BY date)` |
| "What was the peak inventory?" | `MAX(closing_qty)` |
| "Total inventory across all warehouses *on a given day*?" | `SUM(closing_qty) WHERE date = '2025-01-31'` ✅ (same point in time) |

```sql
-- CORRECT: Average daily inventory for January 2025
SELECT
    p.product_name,
    AVG(f.closing_qty)  AS avg_daily_inventory
FROM fact_inventory_daily f
JOIN dim_product p ON f.product_sk = p.product_sk
WHERE f.snapshot_date_sk IN (
    SELECT date_sk FROM dim_date WHERE d_month = '2025-01'
)
GROUP BY p.product_name;

-- CORRECT: Inventory across all warehouses on a single day
SELECT
    w.warehouse_name,
    SUM(f.closing_qty) AS total_inventory_end_of_day
FROM fact_inventory_daily f
JOIN dim_warehouse w ON f.warehouse_sk = w.warehouse_sk
WHERE f.snapshot_date_sk = (
    SELECT date_sk FROM dim_date WHERE d_date = '2025-01-31'
)
GROUP BY w.warehouse_name;
```

**The periodic snapshot connection:** Semi-additive measures are the *hallmark* of periodic snapshot fact tables. That's why `fact_inventory_daily`, `fact_account_balance`, and `fact_headcount_daily` all live in snapshot tables — they track state over time.

**Interviewer Questions You'll Face**

> **Q1: "Is account balance additive? Explain."**
>
> **A:** Account balance is *semi-additive*. You can sum balances across multiple accounts to get a total portfolio value at a point in time, or across branches and locations. But you cannot sum balances across different dates — adding yesterday's balance to today's balance has no meaning. You need AVG, MAX, or LAST_VALUE when aggregating across time.

> **Q2: "How would you sum warehouse inventory across all warehouses for a given day?"**
>
> **A:** On a single date, inventory is additive across warehouses — so `SUM(closing_qty) WHERE snapshot_date = '2025-01-31'` is perfectly valid. The key is fixing the time dimension to one point so we're aggregating across space (warehouses) only.

> **Q3: "A user asks for the total number of employees over Q1. What's wrong with this request?"**
>
> **A:** Headcount is a semi-additive measure — it's a snapshot of how many people work here at a point in time. "Total employees over Q1" is ambiguous. I'd clarify: do they want the *average* headcount across Q1, the headcount *at the end* of Q1, or the *peak* headcount during Q1?

> **Q4: "You see a dashboard showing SUM of daily active users — is this correct?"**
>
> **A:** No. Daily Active Users (DAU) is a snapshot count — it tells us how many unique users were active *on that day*. Summing DAU across 30 days gives us "user-days," not "users." The correct aggregation depends on intent: average DAU for capacity planning, or rolling unique users for actual user count.

**Top-Up Questions (Senior-Level Probes)**

> **Q1: "How do you handle restatements in a semi-additive measure?"**
>
> **A:** If a prior month's account balance is corrected, the periodic snapshot must either: (a) delete and re-insert the corrected row (if append-only isn't strictly enforced), or (b) add a correction row with a sign reversal and net the values at query time. The design must ensure queries over any period see the corrected value.

> **Q2: "Can a measure be semi-additive across dimensions other than time?"**
>
> **A:** Yes. For example, `budget_allocation` is semi-additive across departments but not across fiscal years (each year's budget is a separate allocation). The non-additive dimension depends on the business semantics, not always time.

> **Q3: "What SQL window function would you use to get the latest balance per account?"**
>
> **A:** `LAST_VALUE(balance) OVER (PARTITION BY account_id ORDER BY snapshot_date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` — or more efficiently, `ROW_NUMBER() OVER (PARTITION BY account_id ORDER BY snapshot_date DESC)` and filter for `rn = 1`.

---

### 3. Non-Additive Measures (Never Sum)

**Definition:** A non-additive measure cannot be meaningfully summed across *any* dimension. These are typically **derived values** — ratios, percentages, averages, or prices — that make sense only at the grain at which they were calculated.

**Examples:**

| Measure | Why It's Non-Additive |
|---|---|
| `price_per_unit` | Summing $5 and $10 doesn't give a meaningful "total price" |
| `profit_margin_pct` | Summing 20% and 15% gives 35% — which is meaningless |
| `discount_rate` | Adding two discount percentages is mathematically nonsensical |
| `conversion_rate` | Summing 2.5% and 3.1% produces nothing useful |
| `average_session_duration` | Summing averages is wrong — need weighted average |
| `tax_rate` | 8% + 10% ≠ total tax |

**The Correct Approach: Recalculate from Base Measures**

Non-additive measures should *never* be stored as standalone facts if they need to be aggregated. Instead, store the *component* additive measures and recalculate at query time:

```sql
-- WRONG: Summing a non-additive measure
SELECT SUM(profit_margin_pct) FROM fact_orders;
-- Returns 487.3% — completely meaningless

-- CORRECT: Recalculate from additive components
SELECT
    d.d_year,
    d.d_month,
    SUM(f.revenue)                                           AS total_revenue,
    SUM(f.cost)                                              AS total_cost,
    (SUM(f.revenue) - SUM(f.cost)) / NULLIF(SUM(f.revenue), 0)
        AS weighted_profit_margin                            -- correct!
FROM fact_orders f
JOIN dim_date d ON f.date_sk = d.date_sk
GROUP BY d.d_year, d.d_month;
```

**Weighted Average Technique — the most important pattern for interviews:**

```sql
-- Scenario: What was the average price per unit this quarter?

-- WRONG: Simple average of unit prices (gives equal weight to every sale)
SELECT AVG(price_per_unit) FROM fact_orders;
-- A $1 pencil and a $10,000 laptop get equal weight — wrong!

-- CORRECT: Weighted average (revenue-weighted price)
SELECT
    SUM(revenue) / NULLIF(SUM(units_sold), 0) AS weighted_avg_price
FROM fact_orders
WHERE date_sk IN (
    SELECT date_sk FROM dim_date WHERE d_quarter = '2025-Q1'
);
-- Each unit sold is weighted equally — this is the real average price
```

**Decision table for aggregating non-additive measures:**

| Non-Additive Measure | Correct Aggregation | Derived From |
|---|---|---|
| `price_per_unit` | `SUM(revenue) / SUM(units_sold)` | revenue, units_sold |
| `profit_margin %` | `(SUM(revenue) - SUM(cost)) / SUM(revenue)` | revenue, cost |
| `conversion_rate` | `SUM(conversions) / SUM(visits)` | conversions, visits |
| `discount_pct` | `SUM(discount_amount) / SUM(list_price)` | discount_amount, list_price |
| `avg_session_duration` | `SUM(total_duration) / SUM(sessions)` | total_duration, sessions |

**Interviewer Questions You'll Face**

> **Q1: "A user wants daily average revenue per customer — how do you model this?"**
>
> **A:** I wouldn't store "average revenue per customer" as a measure in the fact table. Instead, I'd store `revenue` and `customer_sk` as separate additive measures at the transaction grain. Then at query time: `SUM(revenue) / COUNT(DISTINCT customer_sk) GROUP BY date`. This gives a true daily average.

> **Q2: "What's the problem with storing price in a fact table?"**
>
> **A:** Price is non-additive. If stored as a measure, any SUM or AVG across transactions gives wrong results. I'd either: (a) store `revenue` and `units_sold` separately and calculate price at query time, or (b) put price in `dim_product` where it's looked up by key, not aggregated.

> **Q3: "Can you make a non-additive measure additive?"**
>
> **A:** Not directly — but you can replace it with its additive components. Instead of storing `conversion_rate` as a single column, store `conversions` (additive) and `impressions` (additive). Then conversion_rate = SUM(conversions) / SUM(impressions) at any aggregation level.

**Top-Up Questions (Senior-Level Probes)**

> **Q1: "When might you choose to pre-compute a non-additive measure anyway?"**
>
> **A:** In high-performance scenarios where the weighted recalculation is too expensive and the reporting is always done at a fixed grain. For example, a pre-aggregated daily table might store `daily_margin_pct` because every dashboard query is at the day level — no further aggregation is needed. This is a deliberate trade-off: performance for flexibility.

> **Q2: "How do you handle currency conversion with non-additive measures?"**
>
> **A:** Convert the base additive measures first, then recalculate. Convert revenue to USD, convert cost to USD, *then* compute margin. Converting a percentage (non-additive) directly produces errors because the conversion factor differs across currencies.

> **Q3: "What if a dashboard tool only allows you to aggregate stored columns?"**
>
> **A:** This is a common real-world constraint. I'd store both the additive components and document clearly which measures are non-additive, adding metadata or a view layer that overrides the default aggregation. In a data warehouse, materialized views can pre-compute the correct weighted aggregations at each required level.

---

### 4. Summary: The Additivity Decision Matrix

Use this table during interviews to classify any measure on the fly:

| Question to Ask Yourself | Additive | Semi-Additive | Non-Additive |
|---|---|---|---|
| Can I SUM this across time? | ✅ | ❌ | ❌ |
| Can I SUM this across products? | ✅ | ✅ | ❌ |
| Can I SUM this across customers? | ✅ | ✅ | ❌ |
| Default aggregation | `SUM()` | `AVG()`, `MAX()`, `LAST_VALUE()` | Recalculate from components |
| Storage strategy | Store directly in fact table | Store in periodic snapshot table | Store components; compute at query |

---

### 5. Interview Question Patterns

These are the patterns you'll encounter in real interviews. Practice answering each until you can explain them in under 60 seconds.

**Pattern 1: Classification**

> *"Here's a fact table for a bank: `account_sk`, `date_sk`, `balance`, `transactions_count`, `interest_rate`, `overdraft_fee`. Classify each measure."*

| Measure | Classification | Reasoning |
|---|---|---|
| `balance` | Semi-additive | Sum across accounts ✅; sum across days ❌ |
| `transactions_count` | Additive | Sum across any dimension ✅ |
| `interest_rate` | Non-additive | A rate/percentage — never sum |
| `overdraft_fee` | Additive | Dollar amount of fees — sums anywhere |

**Pattern 2: The Trap Question**

> *"Write a query to get the total balance of all accounts in January 2025."*

The trap: A naive `SUM(balance)` gives a garbage number.

```sql
-- CORRECT: Total balance across all accounts at month-end
SELECT
    SUM(f.balance) AS total_portfolio_value
FROM fact_account_balances f
WHERE f.date_sk = (
    SELECT MAX(date_sk) FROM dim_date
    WHERE d_month = '2025-01'
);
-- We sum across accounts on a SINGLE day — this is correct because
-- balance IS additive across accounts (just not across time)
```

**Pattern 3: The Design Question**

> *"A user wants to track the monthly profit margin by product category. How do you model this?"*

```sql
-- Design: store additive components in the fact table
CREATE TABLE fact_sales (
    date_sk       INT NOT NULL,
    product_sk    INT NOT NULL,
    revenue       DECIMAL(18,2),    -- additive
    cost          DECIMAL(18,2),    -- additive
    units_sold    INT               -- additive
    -- NO profit_margin_pct column!
);

-- Query: recalculate at the required grain
SELECT
    d.d_year_month,
    cat.category_name,
    SUM(f.revenue)                                    AS total_revenue,
    SUM(f.cost)                                       AS total_cost,
    (SUM(f.revenue) - SUM(f.cost)) / NULLIF(SUM(f.revenue), 0)
        AS profit_margin                               -- derived, non-additive
FROM fact_sales f
JOIN dim_date d ON f.date_sk = d.date_sk
JOIN dim_product p ON f.product_sk = p.product_sk
JOIN dim_category cat ON p.category_sk = cat.category_sk
GROUP BY d.d_year_month, cat.category_name;
```

---

### 6. Practice Exercise

**Scenario:** You are designing a data warehouse for a grocery chain. The business has the following requirements:

1. Track daily inventory levels per product per store.
2. Track every sales transaction (product, store, date, units sold, revenue, cost, unit price, discount percentage).
3. Track the number of store employees by store by date.
4. Calculate daily, weekly, and monthly KPIs including:
   - Total revenue by store and category
   - Average inventory per store per month
   - Employee headcount per store per day
   - Average unit price across all sales
   - Profit margin by category per month

**Your tasks:**

**A.** For each measure below, classify as additive, semi-additive, or non-additive. Write one sentence of justification.

| Measure | Your Classification | Justification |
|---|---|---|
| `inventory_quantity` (daily snapshot) | | |
| `units_sold` (per transaction) | | |
| `revenue` (per transaction) | | |
| `cost` (per transaction) | | |
| `unit_price` (per transaction) | | |
| `discount_pct` (per transaction) | | |
| `headcount` (daily snapshot) | | |
| `profit_margin_pct` (derived) | | |

**B.** Design two fact tables — one transaction fact and one periodic snapshot. List the grain, dimensions, and measures for each.

**C.** Write SQL queries for each KPI. Pay special attention to the non-additive and semi-additive measures.

**D.** Bonus: If the business asks "What was the average headcount per store across the year?", explain the correct aggregation and why SUM would be wrong.

---

**Suggested Answers (review after attempting):**

**A.**
| Measure | Classification | Justification |
|---|---|---|
| `inventory_quantity` | Semi-additive | Sums across stores/products on one day, but not across days |
| `units_sold` | Additive | Each unit is an independent countable event |
| `revenue` | Additive | Dollar amounts sum correctly at every level |
| `cost` | Additive | Cost is additive just like revenue |
| `unit_price` | Non-additive | Price is a rate; summing prices is meaningless |
| `discount_pct` | Non-additive | Percentage; use `SUM(discount_amount) / SUM(list_price)` |
| `headcount` | Semi-additive | Sums across stores on a given day; not across time |
| `profit_margin_pct` | Non-additive | Ratio of two additive measures; recalculate from components |

**B. Fact Table Designs**

```sql
-- Transaction Fact (Additive measures)
-- Grain: One row per product per sale transaction
CREATE TABLE fact_grocery_sales (
    sale_sk          BIGINT PRIMARY KEY,
    date_sk          INT NOT NULL,
    store_sk         INT NOT NULL,
    product_sk       INT NOT NULL,
    units_sold       INT NOT NULL,
    revenue          DECIMAL(14,2),
    cost             DECIMAL(14,2)
);

-- Periodic Snapshot (Semi-additive measures)
-- Grain: One row per product per store per business day
CREATE TABLE fact_inventory_daily (
    snapshot_sk      BIGINT PRIMARY KEY,
    date_sk          INT NOT NULL,
    store_sk         INT NOT NULL,
    product_sk       INT NOT NULL,
    closing_qty      INT NOT NULL,
    inventory_value  DECIMAL(14,2)
);

-- Periodic Snapshot for headcount
-- Grain: One row per store per business day
CREATE TABLE fact_headcount_daily (
    snapshot_sk      BIGINT PRIMARY KEY,
    date_sk          INT NOT NULL,
    store_sk         INT NOT NULL,
    employee_count   INT NOT NULL
);
```

**C. KPI Queries**

```sql
-- 1. Total revenue by store and category
SELECT
    d.d_month,
    s.store_name,
    cat.category_name,
    SUM(f.revenue) AS total_revenue
FROM fact_grocery_sales f
JOIN dim_date d ON f.date_sk = d.date_sk
JOIN dim_store s ON f.store_sk = s.store_sk
JOIN dim_product p ON f.product_sk = p.product_sk
JOIN dim_category cat ON p.category_sk = cat.category_sk
GROUP BY d.d_month, s.store_name, cat.category_name;

-- 2. Average inventory per store per month
SELECT
    d.d_month,
    s.store_name,
    AVG(f.closing_qty) AS avg_monthly_inventory
FROM fact_inventory_daily f
JOIN dim_date d ON f.date_sk = d.date_sk
JOIN dim_store s ON f.store_sk = s.store_sk
GROUP BY d.d_month, s.store_name;

-- 3. Employee headcount per store per day (use LAST_VALUE or filter to latest)
SELECT
    d.d_date,
    s.store_name,
    f.employee_count
FROM fact_headcount_daily f
JOIN dim_date d ON f.date_sk = d.date_sk
JOIN dim_store s ON f.store_sk = s.store_sk;

-- 4. Average unit price (weighted — NON-additive, recalculate!)
SELECT
    SUM(revenue) / NULLIF(SUM(units_sold), 0) AS weighted_avg_price
FROM fact_grocery_sales;

-- 5. Profit margin by category per month (NON-additive, recalculate!)
SELECT
    d.d_month,
    cat.category_name,
    (SUM(f.revenue) - SUM(f.cost)) / NULLIF(SUM(f.revenue), 0)
        AS profit_margin
FROM fact_grocery_sales f
JOIN dim_date d ON f.date_sk = d.date_sk
JOIN dim_product p ON f.product_sk = p.product_sk
JOIN dim_category cat ON p.category_sk = cat.category_sk
GROUP BY d.d_month, cat.category_name;
```

**D. Bonus Answer:**

> "Average headcount per store across the year requires `AVG(employee_count)` grouped by store — because headcount is semi-additive. SUM would add up 365 daily headcount values, giving a number like 3,285 for a store that had 9 employees. That's 365 × 9 = 3,285 "employee-days," not a meaningful headcount. AVG gives us the correct daily average across the year."

---

> **Final takeaway:** In every data modelling interview, when you encounter a measure, immediately ask: *"Is this additive, semi-additive, or non-additive?"* This one question demonstrates deeper understanding than most candidates show in an entire session.

---

### Related Lessons

- [Fact Table Types](./24-fact-table-types.md)
- [Multi-Fact Query Strategies](./26-multi-fact-query-strategies.md)
- [Fact Modeling Practice](./27-fact-modeling-practice.md)
- [Analyzing Metrics](./07-analyzing-metrics.md)
