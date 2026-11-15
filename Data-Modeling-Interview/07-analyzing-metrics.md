# Module 2: Gathering Business Requirements
## Lesson 2.3 — Analyzing Metrics

> *"If you can't measure it, you can't model it."* In data modeling, every metric the business cares about eventually becomes a **measure** in a fact table. Learning to analyze metrics rigorously is the bridge between vague requirements and precise schema design.

---

### 1. Types of Metrics

Understanding the taxonomy of metrics is the first step in translating business goals into data models.

| Category | Definition | Example | Modeling Implication |
|---|---|---|---|
| **KPI (Key Performance Indicator)** | A critical metric aligned to a strategic objective | Quarterly revenue, monthly active users | Becomes a top-level measure in a fact table |
| **Leading Indicator** | Predicts future outcomes; actionable in real time | Number of demos booked, sign-up rate | Often tracked at a granular time grain |
| **Lagging Indicator** | Confirms past results; descriptive, not predictive | Annual churn rate, revenue growth | Aggregated over longer periods |
| **Input Metric** | Measures effort / activity the team can control | Emails sent, support tickets answered | Often the grain of a transactional fact table |
| **Output Metric** | Measures results / outcomes the team is accountable for | Conversion rate, customer retention | Often a derived/ratios measure in a fact table |

**Key Insight for Interviews:** Leading + Input metrics are *actionable*; lagging + Output metrics are *diagnostic*. A well-designed data model supports both.

---

### 2. Identifying the Right Metrics from Business Requirements

Business stakeholders rarely say "I need a fact table with these measures." They say:

> *"We want to reduce churn."* *"We need to understand what drives revenue."*

Your job is to **extract, clarify, and decompose** metrics from these statements.

#### The 4-Step Process

```
1. LISTEN    → Capture every metric mentioned (verbatim)
2. CLASSIFY  → Tag each as KPI / leading / lagging / input / output
3. DECOMPOSE → Break compound metrics into atomic components
4. VALIDATE  → Confirm with the stakeholder: "So if I understand correctly..."
```

#### Concrete Example

| Stakeholder Says | Extracted Metric | Type | Decomposed Into |
|---|---|---|---|
| "We need to track sales performance" | Total revenue | KPI, lagging, output | `SUM(unit_price × quantity)` per order |
| "We want to reduce customer churn" | Monthly churn rate | KPI, lagging, output | `cancelled_customers / active_customers` |
| "We need to see if our marketing works" | Conversion rate | Leading, output | `purchases / unique_visitors` |
| "We want to improve delivery speed" | Avg delivery time | Lagging, output | `SUM(delivery_time) / COUNT(trips)` |

---

### 3. Metric Decomposition

Complex metrics are **compound** — they are built from smaller, atomic pieces. Decomposing them is essential for:
- Determining the correct **fact table grain**
- Identifying which **dimension joins** are needed
- Supporting flexible analysis (ad-hoc slicing)

#### Decomposition Framework

```
Compound Metric
    │
    ├── Numerator Component    → What event is being counted/measured?
    ├── Denominator Component  → What population does it apply to?
    └── Time Dimension         → Over what period?
```

#### Example: Conversion Rate

```
Conversion Rate = Purchases / Unique Visitors

Numerator:  COUNT(order_id)  WHERE order.status = 'completed'
Denominator: COUNT(DISTINCT visitor_id)  for the same time period

Both require:
  - A shared time dimension (date)
  - The same channel / region dimensions for valid comparison
```

> ⚠️ **Interview Trap:** If the numerator and denominator have different grains (e.g., orders vs. sessions), you may need **two separate fact tables** joined at the dimension level, not at the row level.

---

### 4. Domain-Specific Metric Examples

#### E-Commerce

| Metric | Formula | Grain | Fact Table |
|---|---|---|---|
| **GMV** (Gross Merchandise Volume) | `SUM(unit_price × qty)` per order | Order line | `fact_order_line` |
| **Conversion Rate** | `purchases / unique_visitors` | Daily, per channel | Two facts + time dim |
| **AOV** (Average Order Value) | `total_revenue / order_count` | Daily | `fact_order` |
| **CAC** (Customer Acquisition Cost) | `marketing_spend / new_customers` | Monthly, per channel | `fact_marketing` + `fact_customer` |
| **LTV** (Lifetime Value) | `AVG(margin_per_order × orders_per_customer × lifetime_months)` | Per customer | `fact_customer_lifetime` |

#### Ride-Sharing

| Metric | Formula | Grain |
|---|---|---|
| **Trips Completed** | `COUNT(trip_id)` WHERE status = 'completed' | Per driver, per hour |
| **Driver Utilization** | `active_trip_minutes / total_shift_minutes` | Per driver, per day |
| **ETA Accuracy** | `AVG(ABS(eta_minutes - actual_minutes))` | Per trip |

#### Streaming Platform

| Metric | Formula | Grain |
|---|---|---|
| **Watch Time** | `SUM(watch_seconds)` per session | Per user, per title |
| **Completion Rate** | `completed_views / total_starts` | Per title, daily |
| **Churn Rate** | `cancellations / active_subs` at month end | Monthly (periodic snapshot) |

#### SaaS

| Metric | Formula | Grain |
|---|---|---|
| **MRR** (Monthly Recurring Revenue) | `SUM(active_subscription_revenue)` | Monthly (snapshot) |
| **ARR** | `MRR × 12` | Derived from MRR |
| **Churn** | `churned_customers / start_of_month_customers` | Monthly (periodic snapshot) |
| **NPS** | `%promoters - %detractors` | Per survey wave |

---

### 5. How Metrics Drive Fact Table Design

Every metric analysis leads to design decisions. Here's the mapping:

```
Metric → Measure in Fact Table
                          │
                          ├── What's the GRANULARITY? (one row = ?)
                          │     → "one row per order line"
                          │
                          ├── What DIMENSIONS apply?
                          │     → product, customer, channel, time
                          │
                          ├── Is it ADDITIVE, SEMI-ADDITIVE, or NON-ADDITIVE?
                          │     → determines if you can safely aggregate
                          │
                          └── What DEGENERATE DIMENSIONS are needed?
                                → order_number, transaction_id (no dimension table)
```

#### Design Pattern: One Fact Table Per Grain

```
fact_order_line          (grain: one row per line in an order)
├── order_line_key       (PK, surrogate)
├── date_key             (FK → dim_date)
├── product_key          (FK → dim_product)
├── customer_key         (FK → dim_customer)
├── quantity             (additive measure)
├── unit_price           (additive measure)
├── line_total           (additive = quantity × unit_price)
└── order_number         (degenerate dimension)
```

---

### 6. Additive, Semi-Additive, and Non-Additive Measures

This is one of the most frequently tested concepts in data modeling interviews.

| Measure Type | Can Sum Across All Dimensions? | Example | Storage Strategy |
|---|---|---|---|
| **Additive** | ✅ Yes — sums meaningfully across every dimension | Revenue, quantity, cost | Regular measure in any fact table |
| **Semi-Additive** | ✅ Across some (product, region); ❌ Across time | Bank account balance, inventory on hand, subscriber count | **Periodic snapshot fact table** (one row per entity per time period) |
| **Non-Additive** | ❌ No — aggregation loses meaning | Ratio, percentage, average, profit margin | **Compute at query time**: store numerator + denominator as additive measures, then divide in the query layer |

#### Visual Comparison

```
          ┌─────────────┐     ┌─────────────────┐     ┌──────────────────┐
          │ ADDITIVE    │     │ SEMI-ADDITIVE   │     │ NON-ADDITIVE     │
          │             │     │                 │     │                  │
Sales Jan │ $10,000     │ Bal Jan│ $5,000       │ Conv│ 15%              │
Sales Feb │ $12,000     │ Bal Feb│ $7,200       │ Rate│ (150/1000)       │
          │             │         │               │                  │
Sum       │ $22,000 ✓   │ Sum    │ $12,200 ✗    │ Sum  │ 30% ✗          │
          │             │ (use   │               │      │ (store 150,    │
          │             │ latest)│               │      │  store 1000)   │
          └─────────────┘     └─────────────────┘     └──────────────────┘
```

#### Modeling Strategies

| Scenario | Recommended Approach |
|---|---|
| Revenue, cost, quantity | Additive measures in a transaction fact |
| Subscriber count, inventory balance | Semi-additive in a **periodic snapshot fact** |
| Conversion rate, churn %, margin % | Store numerator + denominator as additive; compute ratio at query time |

> 💡 **Pro Tip:** In Kimball methodology, **never store a ratio as a physical measure** in the fact table. Always store the additive components and let the semantic layer (Power BI, Tableau, dbt) compute the ratio. This ensures users can re-aggregate and re-slice without breaking the math.

---

### Interviewer Questions You'll Face

| # | Question | What They're Testing |
|---|---|---|
| 1 | *"A stakeholder says 'I need to track revenue.' What follow-up questions do you ask?"* | Ability to decompose vague requirements |
| 2 | *"How would you model monthly churn rate in a star schema?"* | Understanding of semi-additive measures + snapshot facts |
| 3 | *"Conversion rate is 5% per channel. When summed across channels, it becomes 20% — which is wrong. How do you fix this?"* | Knowledge of non-additive measures; storing numerator/denominator |
| 4 | *"When would you use a periodic snapshot fact table instead of a transaction fact?"* | Knowledge of semi-additive measure modeling |

---

### Questions YOU Should Ask the Interviewer

| Question | Why It Impresses |
|---|---|
| *"What's the business's primary KPI — and what does 'success' look like numerically?"* | Shows you lead with business outcomes, not just data |
| *"Are there metrics the business currently tracks but hasn't asked for yet that would be useful?"* | Shows proactive analytical thinking |
| *"How granular do stakeholders need the analysis — daily, weekly, or real-time?"* | Shows you understand grain drives design |
| *"Which metrics are compared across time periods? Do we need period-over-period or cumulative views?"* | Shows awareness of semi-additive and snapshot patterns |

---

### Top-Up Questions (Senior-Level Probes)

| # | Question | Key Points to Hit |
|---|---|---|
| 1 | *"How do you handle a metric like 'active users' that's semi-additive across time but additive across regions?"* | Periodic snapshot fact; distinguish additive vs semi-additive axes |
| 2 | *"If churn is calculated monthly but marketing wants weekly trends, how do you reconcile the grain mismatch?"* | Multi-grain fact tables; bridge tables; acknowledge trade-offs |
| 3 | *"When should you denormalize metrics into a wide table vs. keep them in normalized star schema?"* | Performance vs. consistency; OLAP vs. data warehouse considerations |
| 4 | *"How do you model a metric that changes definition over time (e.g., 'revenue' shifted from gross to net)?"* | Versioned metrics; SCD-like approach for metric definitions; governance |

---

### Practice Exercise

**Scenario:** You're modeling data for a food delivery startup (like Uber Eats). The CEO says:

> *"I want to understand: (1) how much we're earning, (2) whether our restaurants are profitable, (3) if customers keep coming back, and (4) if drivers are happy."*

#### Your Task

1. **Extract** every metric from the CEO's statement.
2. **Classify** each as KPI / leading / lagging / input / output.
3. **Decompose** each metric into atomic numerator + denominator components.
4. **Classify** each as additive / semi-additive / non-additive.
5. **Sketch** the fact tables you'd create — what's the grain of each?

#### Suggested Approach

```
(1) "How much we're earning"
    → Metric: Platform Revenue (KPI, lagging, output)
    → Additive: SUM(commission + delivery_fee) per order
    → Fact: fact_order (grain: one row per order)

(2) "Are restaurants profitable"
    → Metric: Restaurant Margin % (KPI, lagging, output)
    → Non-additive: revenue − cost; store both additively, compute ratio
    → Fact: fact_restaurant_order (grain: one row per order per restaurant)

(3) "Do customers keep coming back"
    → Metric: Customer Retention Rate (KPI, lagging, output)
    → Semi-additive: track per customer per month in periodic snapshot
    → Fact: fact_customer_monthly (grain: one row per customer per month)

(4) "Are drivers happy"
    → Metric: Driver Satisfaction Score (KPI, leading, input)
    → Non-additive: average of survey scores
    → Fact: fact_driver_survey (grain: one row per survey response)
```

> 🎯 **Bonus Challenge:** How would you track "driver utilization" (minutes on delivery / minutes logged in)? What grain would this need, and would it be additive, semi-additive, or non-additive?

---

### Key Takeaways

1. **Every business metric maps to a measure in a fact table** — your job is to determine the right grain and type.
2. **Decompose compound metrics** into additive components before designing the schema.
3. **Additive measures** are easy — they sum everywhere. **Semi-additive** need snapshot facts. **Non-additive** ratios need numerator + denominator stored separately.
4. **Always ask clarifying questions** — the best data models come from the best conversations with stakeholders.
5. **Practice decomposing metrics on the fly** — this is one of the most common whiteboard interview exercises.

---

### Related Lessons

- ⬅️ [Lesson 6: Recognizing the Core Business Problem](./06-recognizing-the-core-business-problem.md) — peel back the layers
- ➡️ [Lesson 8: Analyzing Query Patterns](./08-analyzing-query-patterns.md) — how metrics guide table design
- ➡️ [Lesson 24: Fact Table Types](./24-fact-table-types.md) — which fact type fits each metric
