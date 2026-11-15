# Module 2: Gathering Business Requirements
## Lesson 2.4 — Analyzing Query Patterns

> *"A data model optimized for the wrong queries is a data model that fails in production."* — Every data warehouse engineer learns this the hard way.

After identifying the business problem and decomposing the metrics, the next critical step is **understanding how the data will be queried**. The same set of tables can serve a reporting dashboard beautifully and a real-time personalization engine terribly. Query patterns dictate the *shape* of your model.

---

### 1. Why Query Patterns Matter in Data Model Design

The way people query data determines:

| Design Decision | Driven By | Example |
|---|---|---|
| **Fact table grain** | Query grouping level | "Revenue by product per month" → monthly grain may suffice |
| **Dimension design** | Slice-and-dice needs | "By region, channel, and segment" → each needs a dimension table |
| **Partitioning strategy** | Common filter predicates | "Last 12 months only" → partition by date |
| **Indexing / clustering** | Sort and join keys | "Join on user_id" → cluster the fact by user_id |
| **Denormalization level** | Join frequency | Heavy joins between product and category → merge into one dimension |
| **Pre-aggregation** | Query volume & latency needs | Dashboard hit 10K times/day → materialized view |

> **Rule of thumb:** Your data model should be shaped by the *actual queries users run*, not by the *theoretical relationships in the source data*.

In interviews, interviewers test whether you can **reverse-engineer design decisions from query requirements**. They will say:

> *"We need to answer: 'What was the conversion rate for each marketing campaign, broken down by device type, for the last 6 months?'"*

A junior hears: "Oh, I need a conversion rate table." A senior hears: "I need a fact table at the event level, joined to campaign, device, and date dimensions, likely partitioned by date."

---

### 2. Common Query Pattern Categories

Understanding these six categories will cover 90% of interview scenarios.

#### A. Aggregations

**What it is:** Summarizing data across groups — totals, averages, counts, percentiles.

```sql
SELECT
  product_category,
  region,
  SUM(revenue)  AS total_revenue,
  AVG(order_value)  AS avg_order_value
FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY product_category, region;
```

**Modeling implications:**
- Determines whether you need **pre-aggregated summary tables**
- If the aggregation is run repeatedly with the same group-by, a **materialized view** or **aggregated fact table** saves compute
- The granularity of GROUP BY columns tells you which **dimension tables** are needed

#### B. Filtering

**What it is:** Narrowing the result set by one or more conditions before aggregation.

```sql
WHERE order_date >= '2024-06-01'       -- Time-range filter
  AND region IN ('US', 'EU')           -- Categorical filter
  AND order_value > 100                -- Value-range filter
```

**Modeling implications:**
- **Time-range filters** → partition or cluster by date columns (e.g., `PARTITION BY DATE_TRUNC('month', order_date)`)
- **High-cardinality categorical filters** (e.g., user_id, product_id) → ensure those keys are indexed or clustered
- **Multi-dimensional filtering** → each filter column typically maps to a dimension table in a star schema

#### C. Joins

**What it is:** Combining data from multiple tables to enrich the fact with context.

```sql
SELECT o.*, p.product_name, p.category, c.customer_name, c.region
FROM orders o
JOIN products p ON o.product_id = p.product_id
JOIN customers c ON o.customer_id = c.customer_id;
```

**Modeling implications:**
- **Star schema:** Keep dimension tables separate for flexibility; accept the join cost
- **Denormalized / OneBigTable (OBT):** Flatten dimensions into the fact for speed; lose some flexibility
- The tradeoff: **flexibility vs. query performance**
- In cloud data warehouses (BigQuery, Snowflake, Redshift), joins are cheaper than on-premise — but not free at billion-row scale

#### D. Time-Series Analysis

**What it is:** Analyzing how metrics evolve over time — daily revenue, weekly active users, monthly churn.

```sql
SELECT
  DATE_TRUNC('week', order_date)  AS week,
  COUNT(DISTINCT customer_id)     AS weekly_active_users
FROM orders
GROUP BY week
ORDER BY week;
```

**Modeling implications:**
- Requires a robust **date dimension** or reliable time-truncation functions
- Common patterns: **moving averages**, **period-over-period comparisons**, **year-over-year growth**
- Consider a **periodic snapshot fact table** (e.g., one row per user per day) to track state over time

#### E. Cohort Analysis

**What it is:** Tracking a group of users who share a common characteristic over time — typically when they signed up.

```sql
SELECT
  DATE_TRUNC('month', signup_date)  AS cohort,
  date_diff_days,
  COUNT(DISTINCT user_id)  AS active_users
FROM user_activity
GROUP BY cohort, date_diff_days;
```

**Modeling implications:**
- Requires storing a **cohort identifier** (e.g., `signup_month`) on every user or event
- Often a **slowly changing dimension** problem — users may move between segments
- A **bridge table** (user_cohort mapping) can decouple user identity from cohort membership

#### F. Funnel Analysis

**What it is:** Tracking users as they progress through a multi-step process — view → add-to-cart → checkout → purchase.

```sql
-- Funnel: page_view → add_to_cart → checkout → purchase
WITH steps AS (
  SELECT
    user_id,
    MAX(CASE WHEN event_type = 'page_view' THEN 1 ELSE 0 END)  AS viewed,
    MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END)  AS added_cart,
    MAX(CASE WHEN event_type = 'checkout' THEN 1 ELSE 0 END)  AS checked_out,
    MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END)  AS purchased
  FROM user_events
  WHERE event_date >= '2024-01-01'
  GROUP BY user_id
)
SELECT
  COUNT(CASE WHEN viewed = 1 THEN 1 END)  AS step_1,
  COUNT(CASE WHEN added_cart = 1 THEN 1 END)  AS step_2,
  COUNT(CASE WHEN checked_out = 1 THEN 1 END)  AS step_3,
  COUNT(CASE WHEN purchased = 1 THEN 1 END)  AS step_4,
FROM steps;
```

**Modeling implications:**
- Requires an **event-level fact table** capturing every interaction
- A **step_sequence_id** or **session_id** ties events to a single funnel instance
- Each funnel step needs a distinct, queryable `event_type` column

---

### 3. How to Identify Query Patterns from Requirements

Business stakeholders rarely speak in SQL. They say:

> *"We want to understand which products are doing well in each region."*

Your job: **translate natural language into query patterns.**

#### The Translation Checklist

| Requirement Phrase | Query Pattern | What to Ask Next |
|---|---|---|
| "Revenue by X" | Aggregation + multi-dimensional filter | "What dimensions do you need to break it down by?" |
| "Trends over time" | Time-series | "What's the minimum time granularity — daily, weekly, monthly?" |
| "Compare groups" | Cohort analysis | "How do you define the groups? By signup date, plan type?" |
| "Drop-off at each step" | Funnel analysis | "What are the steps? Is there a unique session/process ID?" |
| "Top N items" | Aggregation + sort + limit | "Top N by what measure? Within what time window?" |
| "Year-over-year comparison" | Time-series + join self | "Do you need same-period last year, or 12-month rolling?" |
| "Drill-down from summary to detail" | Multi-grain model needed | "What's the deepest level of detail they'll drill to?" |

#### The "Five Ws" of Query Discovery

Always ask these before designing:

```
WHO  → Who runs the query? (analyst, dashboard, ML pipeline)
WHAT → What is the output? (a number, a table, a chart)
WHEN → When is the query run? (real-time, daily batch, ad-hoc)
WHERE → Where does the result go? (dashboard, report, alert)
WHY  → Why does this query matter? (drives a business decision)
```

> **Interview tip:** If the interviewer says "we need to track user engagement," immediately follow up: "What does 'engagement' mean to you, and how often will you query it?" This shows you're thinking about both the metric and the pattern.

---

### 4. Mapping Query Patterns to Design Decisions

This is the core skill interviewers are testing. For each query pattern, here's the design implication:

#### Heavy Aggregations → Pre-Aggregated Tables / Materialized Views

```
Query: "Monthly revenue by product category, run 5,000 times/day on a dashboard"

Design:
  ├── Source: event-level fact table (raw transactions)
  ├── Aggregated: monthly_revenue_by_category (materialized view)
  │     └── Partitioned by month
  ├── Refresh: incremental daily refresh
  └── Trade-off: slight staleness vs. 100x faster queries
```

**When to use:**
- The same aggregation is queried thousands of times
- Freshness tolerance is hours or days (not seconds)
- The source table is too large for on-demand aggregation

#### Time-Range Filters → Date Partitioning

```
Query: "All orders in the last 12 months" (run repeatedly)

Design:
  ├── Partition strategy: PARTITION BY DATE_TRUNC('month', order_date)
  ├── Query benefit: partition pruning scans only 12 partitions instead of all data
  ├── Alternative: CLUSTER BY order_date (sorts data for range scans)
```

**When to use:**
- 80%+ of queries include a date range filter
- The dataset is large enough that full scans are expensive
- Data has a natural time boundary (monthly, daily)

#### Multi-Dimensional Filtering → Dimension Design

```
Query: "Revenue by region, product category, channel, and customer segment"

Design (Star Schema):
  ┌──────────────┐       ┌──────────────────┐
  │  dim_region  │◄──────│                  │
  └──────────────┘       │                  │
  ┌──────────────┐       │  fact_orders     │
  │ dim_category  │◄─────│                  │
  └──────────────┘       │                  │
  ┌──────────────┐       │                  │
  │ dim_channel   │◄─────│                  │
  └──────────────┘       │                  │
  ┌──────────────┐       └──────────────────┘
  │dim_customer_seg│◄──────
  └──────────────┘
```

**When to use:**
- Users need to slice the same fact across many different dimensions
- Each dimension has its own hierarchy (e.g., region → country → city)
- Dimension tables are relatively small (thousands to millions of rows)

#### Joins → Star Schema vs. Denormalized

| Approach | Pros | Cons | Best For |
|---|---|---|---|
| **Star Schema** (normalized dimensions) | Flexible; easy to add new dimensions; smaller fact table | Joins add query complexity; multiple joins can slow queries | Traditional BI; many different query combinations |
| **Denormalized / OBT** (one flat table) | No joins; simple queries; fast aggregations | Redundant data; harder to maintain; large storage | Single-team dashboards; cloud DW with cheap storage |
| **Conformed Dimensions** (shared dims across facts) | Consistent metrics across business units; enables enterprise data warehouse | Requires governance; upfront design effort | Organizations with multiple fact tables sharing dimensions |

> **Interviewer favorite question:** "Would you normalize or denormalize?"
> **Strong answer:** "It depends on who consumes the data and how. If multiple teams query it with different slice-and-dice needs, I prefer a star schema with conformed dimensions. If it's a single dashboard with a fixed set of metrics, a denormalized table with strategic pre-aggregation may be faster and simpler."

---

### 5. Real Examples — End-to-End

#### Example 1: "Revenue by Product Category and Region for Last 12 Months"

**Query pattern:** Aggregation + multi-dimensional filter + time-range

```sql
SELECT
  d_date.month_year,
  d_cat.category_name,
  d_reg.region_name,
  SUM(f.revenue)  AS total_revenue,
  COUNT(f.order_id)  AS order_count
FROM fact_orders f
JOIN dim_date d_date ON f.date_key = d_date.date_key
JOIN dim_category d_cat ON f.category_key = d_cat.category_key
JOIN dim_region d_reg ON f.region_key = d_reg.region_key
WHERE d_date.month_year >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY d_date.month_year, d_cat.category_name, d_reg.region_name;
```

**Design decisions derived:**

| Element | Decision | Why |
|---|---|---|
| Date handling | `dim_date` table + partition by month | Efficient time-range pruning; enables time hierarchies |
| Category | `dim_category` with hierarchy (cat → subcat → product) | Supports drill-down: "Show me subcategories within Electronics" |
| Region | `dim_region` with hierarchy (continent → country → state → city) | Supports geographic drill-down |
| Fact grain | One row per order | Fine enough to support any aggregation level |
| Pre-aggregation | Optional: monthly revenue by category × region | If dashboard queries are very frequent |

---

#### Example 2: "User Retention Over 30/60/90 Days"

**Query pattern:** Cohort analysis + time-series

```sql
SELECT
  u_cohort.cohort_month,
  u_cohort.days_since_signup,
  COUNT(DISTINCT u_cohort.user_id)  AS active_users,
  COUNT(DISTINCT u_cohort.user_id) * 100.0 / u_cohort.cohort_size
    AS retention_pct
FROM (
  SELECT
    u.user_id,
    DATE_TRUNC('month', u.signup_date)  AS cohort_month,
    DATEDIFF(day, u.signup_date, e.event_date)  AS days_since_signup,
    COUNT(DISTINCT u.user_id) OVER (PARTITION BY DATE_TRUNC('month', u.signup_date))
      AS cohort_size
  FROM dim_user u
  JOIN fact_user_events e ON u.user_id = e.user_id
  WHERE e.event_date BETWEEN u.signup_date
    AND DATE_ADD(u.signup_date, INTERVAL 90 DAY)
) u_cohort
WHERE u_cohort.days_since_signup IN (30, 60, 90)
GROUP BY u_cohort.cohort_month, u_cohort.days_since_signup, u_cohort.cohort_size;
```

**Design decisions derived:**

| Element | Decision | Why |
|---|---|---|
| User dimension | `dim_user` with `signup_date` | Cohort assignment requires a stable origin date |
| Event fact | `fact_user_events` at user × event grain | Tracks every interaction for retention calculation |
| Cohort tracking | Pre-computed `cohort_month` column | Avoids re-calculating cohort on every query |
| Time window | 90-day look-forward | Determines how far into the future we need data |
| Snapshot table | Optional: `daily_user_retention_snapshot` | If this report runs frequently, pre-compute daily retention |

---

#### Example 3: "Conversion Funnel: View → Add-to-Cart → Checkout → Purchase"

**Query pattern:** Funnel analysis + event-level fact

```sql
WITH funnel AS (
  SELECT
    e.session_id,
    MAX(CASE WHEN e.event_type = 'product_view' THEN 1 ELSE 0 END)  AS viewed,
    MAX(CASE WHEN e.event_type = 'add_to_cart' THEN 1 ELSE 0 END)  AS cart,
    MAX(CASE WHEN e.event_type = 'checkout_start' THEN 1 ELSE 0 END)  AS checkout,
    MAX(CASE WHEN e.event_type = 'purchase_complete' THEN 1 ELSE 0 END)  AS purchased
  FROM fact_user_events e
  WHERE e.event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  GROUP BY e.session_id
)
SELECT
  SUM(viewed)  AS step_1_viewed,
  SUM(cart)  AS step_2_cart,
  SUM(checkout)  AS step_3_checkout,
  SUM(purchased)  AS step_4_purchased,
  SUM(cart) * 100.0 / NULLIF(SUM(viewed), 0)  AS view_to_cart_rate,
  SUM(checkout) * 100.0 / NULLIF(SUM(cart), 0)  AS cart_to_checkout_rate,
  SUM(purchased) * 100.0 / NULLIF(SUM(checkout), 0)  AS checkout_to_purchase_rate
FROM funnel;
```

**Design decisions derived:**

| Element | Decision | Why |
|---|---|---|
| Event fact | `fact_user_events` at event grain (one row per event) | Every funnel step is an event; must capture each individually |
| Session ID | `session_id` or `page_view_id` ties events in a single journey | Enables grouping events into a funnel instance |
| Event type | Categorical `event_type` column with controlled vocabulary | Each funnel step maps to a specific value |
| Time context | `event_date` or `event_timestamp` for ordering | Funnel steps must be temporally ordered |
| Pre-computation | Optional: `daily_funnel_summary` for dashboards | Avoids re-computing the funnel on every dashboard load |

---

### 6. Interviewer Questions You'll Face

| # | Question | What They're Testing |
|---|---|---|
| 1 | "We have a fact table with 2 billion rows. Queries filtering by date take 30 seconds. How do you improve it?" | Partitioning strategy; understanding of partition pruning |
| 2 | "Would you build a star schema or a flattened table for a marketing analytics model?" | Trade-off analysis: flexibility vs. performance |
| 3 | "How would you model a funnel analysis that needs to track users across multiple devices?" | Session/user identity resolution; event-level grain |
| 4 | "The business wants to see 'revenue by product by region by day,' but they also need 'revenue by product by region by quarter.' Can the same model serve both?" | Multi-grain design; aggregation flexibility |
| 5 | "How do you decide the grain of your fact table?" | Understanding of fact table granularity and its impact on all queries |

---

### 7. Questions YOU Should Ask the Interviewer

| Question | What It Reveals | Why Ask It |
|---|---|---|
| "What are the top 3 queries this model needs to support?" | Priority use cases | Ensures you optimize for the right workload |
| "Who are the primary consumers of this data?" | Dashboard users, analysts, ML pipelines | Different consumers need different grains and SLAs |
| "What's the expected data volume and growth rate?" | Scale requirements | Determines partitioning, archiving, and refresh strategy |
| "What's the acceptable query latency for the main dashboard?" | Performance requirements | Decides between pre-aggregation vs. on-demand |
| "Are there any queries that currently fail or time out?" | Known pain points | Shows you can address existing problems |
| "Will this model feed downstream systems or just BI tools?" | Integration scope | Downstream consumers may need different schemas |
| "What's the data freshness requirement — real-time, hourly, daily?" | SLA constraints | Determines batch vs. streaming, snapshot frequency |

---

### 8. Top-Up Questions (Senior-Level Probes)

| # | Question | What a Strong Answer Includes |
|---|---|---|
| 1 | "How would your design change if this query ran 100,000 times per day instead of 100?" | Discuss materialized views, result caching, query result caching in the DW, pushing aggregation to the app layer |
| 2 | "If the business adds a new dimension — say 'sales rep' — how do you handle that without breaking existing models?" | Discuss conformed dimensions, impact on star schema vs. OBT, backward compatibility, migration strategy |
| 3 | "How do you handle queries that need to join three different fact tables?" | Discuss bridge tables, common grain alignment, summary tables at a shared grain, or a unified OBT |
| 4 | "When would you choose an accumulating snapshot over a transactional fact for a process-tracking use case?" | Discuss order fulfillment tracking, trade-off between denormalization and denormalized state tracking, SCD implications |
| 5 | "How do you ensure query performance doesn't degrade as data grows?" | Discuss partitioning, clustering, archival strategies, incremental materialized views, and monitoring query plans |

---

### 9. Practice Exercise

#### Scenario

You're interviewing for a Senior Data Engineer role at **StreamFlix**, a streaming video platform. The interviewer says:

> *"We need a data model to help our content team decide which shows to produce, renew, or cancel. Here's what they need to query:*
>
> *1. Watch hours by genre and country for the last 6 months*
> *2. Subscriber retention by signup cohort (30/60/90 days)*
> *3. The conversion funnel: free trial started → first episode watched → trial completed → paid subscription*
> *4. Average completion rate per title, broken down by genre and subscriber plan*
> *5. Top 10 most-watched titles each week"*

**Your task (work through this on paper or in a notebook):**

**Step 1:** Identify the query pattern for each of the 5 requirements.

**Step 2:** For each requirement, list the key design decisions it drives (grain, dimensions, partitioning, pre-aggregation).

**Step 3:** Sketch a high-level schema (fact tables + dimension tables).

**Step 4:** Identify at least 3 questions you would ask the interviewer to refine the model.

<details>
<summary>🔑 Click to see a sample answer</summary>

**Step 1: Query Patterns**

| # | Requirement | Pattern |
|---|---|---|
| 1 | Watch hours by genre/country | Aggregation + multi-dimensional filter + time-range |
| 2 | Subscriber retention by cohort | Cohort analysis |
| 3 | Trial → paid funnel | Funnel analysis |
| 4 | Completion rate by genre/plan | Aggregation + multi-dimensional filter |
| 5 | Top 10 titles weekly | Aggregation + sort + limit + time-series |

**Step 2: Design Decisions**

| Requirement | Fact Grain | Dimensions Needed | Special Design |
|---|---|---|---|
| 1 | View session (user × title × timestamp) | dim_genre, dim_country, dim_date | Partition by month |
| 2 | Subscriber lifecycle events | dim_user (with signup_date), dim_date | Cohort_month pre-computed |
| 3 | Event-level (trial_start, ep_watch, trial_end, subscribe) | dim_user, dim_date | event_type column, session/process tracking |
| 4 | View session with completion flag | dim_genre, dim_plan_type | Completion ratio (episodes watched / total episodes) |
| 5 | View session (aggregated to title × week) | dim_title, dim_date | Weekly aggregation; top-N requires sort |

**Step 3: High-Level Schema**

```
┌──────────────┐       ┌──────────────────────────┐
│ dim_date     │◄──────│                          │
└──────────────┘       │  fact_view_sessions      │  ← One row per
┌──────────────┐       │  (user_id, title_id,     │    view event
│ dim_title    │◄──────│   view_start_time,       │
│ (incl. genre)│       │   minutes_watched,       │
└──────────────┘       │   episodes_watched,      │
┌──────────────┐       │   session_id)            │
│ dim_country  │◄──────│                          │
└──────────────┘       └──────────────────────────┘
┌──────────────┐       ┌──────────────────────────┐
│ dim_user     │◄──────│  fact_subscriber_events   │  ← One row per
│ (signup_date,│       │  (user_id, event_type,   │    subscriber
│  plan_type,  │◄──────│   event_date, plan_id)   │    lifecycle event
│  cohort)     │       └──────────────────────────┘
└──────────────┘
┌──────────────┐
│ dim_plan     │◄──────  (plan_id, plan_name,
└──────────────┘    monthly_price)
```

**Step 4: Questions to Ask**

1. "Do you need real-time watch data, or is a daily batch acceptable?" → Determines streaming vs. batch pipeline
2. "What counts as 'completing' a title — watching all episodes, or 90%?" → Defines the completion metric
3. "Do you need to compare subscriber behavior across countries, or is country mainly for content licensing?" → Determines if country is a primary dimension or just a filter
4. "How many titles and concurrent users are we talking about?" → Determines partitioning and volume strategy

</details>

---

### Key Takeaways

1. **Query patterns dictate model shape.** Always identify the dominant query patterns before designing tables.
2. **Aggregations drive pre-aggregation decisions.** High-frequency aggregations benefit from materialized views or summary fact tables.
3. **Filter patterns drive partitioning and indexing.** If 80% of queries filter by date, partition by date.
4. **Multi-dimensional filtering drives dimension design.** Each slice-and-dice dimension becomes a dimension table in a star schema.
5. **Joins are the cost of flexibility.** Star schemas support arbitrary combinations but add join overhead; denormalization reduces joins at the cost of flexibility.
6. **Cohort and funnel analysis require event-level grain.** You cannot reconstruct these from aggregated data — keep the raw events.

> **Final interview tip:** When the interviewer gives you a scenario, say out loud: *"Let me first identify the query patterns here, then I'll map them to design decisions."* This single sentence signals that you have a structured approach — and that's exactly what separates seniors from juniors.

---

### Related Lessons

- ⬅️ [Lesson 7: Analyzing Metrics](./07-analyzing-metrics.md) — metrics and queries go hand in hand
- ➡️ [Lesson 9: Defining Latency Requirements](./09-defining-latency-requirements.md) — latency drives partition and refresh strategy
- ➡️ [Lesson 13: Creating High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md) — map query patterns to a visual schema
