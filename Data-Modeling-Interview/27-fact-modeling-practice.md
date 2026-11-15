
---

### 💡 Answer Key — Food Delivery Platform

#### A. Fact Table Type Justifications

| # | Need | Fact Type | Justification |
|---|---|---|---|
| 1 | Order lifecycle tracking | **Accumulating Snapshot** | Tracks a single order through multiple defined milestones; measures elapsed time between stages. |
| 2 | Daily revenue by restaurant | **Periodic Snapshot** | Summarizes revenue at a fixed daily frequency; answers "what was total revenue on this date?" |
| 3 | Rider performance metrics | **Periodic Snapshot** | Aggregated daily metrics per rider (average delivery time, count of deliveries); not a transaction log. |
| 4 | Restaurant ratings over time | **Periodic Snapshot** | Captures a running state (average rating) at a daily frequency; the rating is non-additive. |

> **Discussion point:** Some could argue that #2 (revenue) could be transactional. But the business question is "daily revenue by restaurant" — an aggregate. If the business also needs transaction-level detail, create *both*: a transactional `fact_orders` for drill-down, and a periodic `fact_daily_revenue_snapshots` for fast dashboards.

#### B. CREATE TABLE Statements

```sql
-- 1. Order lifecycle (accumulating snapshot)
CREATE TABLE fact_order_lifecycle (
    order_lifecycle_key   BIGINT          PRIMARY KEY,
    customer_key          INT             NOT NULL,
    restaurant_key        INT             NOT NULL,
    rider_key             INT,
    order_date_key        INT             NOT NULL,
    -- Milestone date keys (NULL until the event occurs)
    order_placed_date_key INT             NOT NULL,
    restaurant_accept_date_key INT,
    food_ready_date_key   INT,
    rider_assigned_date_key INT,
    picked_up_date_key    INT,
    delivered_date_key    INT,
    -- Measures
    order_total           DECIMAL(10,2),
    tip_amount            DECIMAL(10,2),
    delivery_fee          DECIMAL(10,2),
    platform_commission   DECIMAL(10,2),
    -- Derived
    prep_time_minutes     INT,                    -- food_ready - restaurant_accept
    delivery_time_minutes INT,                    -- delivered - picked_up
    total_time_minutes    INT,                    -- delivered - order_placed
    order_status          VARCHAR(20),             -- 'placed', 'preparing', 'in_transit', 'delivered', 'cancelled'
    etl_updated_at        TIMESTAMP       DEFAULT CURRENT_TIMESTAMP
);

-- 2. Daily revenue by restaurant (periodic snapshot)
CREATE TABLE fact_daily_revenue (
    daily_revenue_key     BIGINT          PRIMARY KEY,
    date_key              INT             NOT NULL,
    restaurant_key        INT             NOT NULL,
    city_key              INT,
    total_orders          INT,
    total_food_revenue    DECIMAL(14,2),
    total_tips            DECIMAL(14,2),
    total_delivery_fees   DECIMAL(14,2),
    total_commissions     DECIMAL(14,2),
    gross_revenue         DECIMAL(14,2),         -- all revenue columns summed
    avg_order_value       DECIMAL(10,2)          -- gross_revenue / total_orders
);

-- 3. Rider performance (periodic snapshot — daily per rider)
CREATE TABLE fact_rider_performance (
    rider_perf_key        BIGINT          PRIMARY KEY,
    date_key              INT             NOT NULL,
    rider_key             INT             NOT NULL,
    zone_key              INT,
    orders_delivered      INT,
    avg_prep_time_min     DECIMAL(6,2),
    avg_delivery_time_min DECIMAL(6,2),
    total_distance_km     DECIMAL(8,2),
    total_earnings        DECIMAL(10,2),
    on_time_pct           DECIMAL(5,2),          -- % of deliveries within SLA
    avg_customer_rating   DECIMAL(3,2)
);

-- 4. Restaurant ratings (periodic snapshot — daily per restaurant)
CREATE TABLE fact_restaurant_ratings (
    rating_snapshot_key   BIGINT          PRIMARY KEY,
    date_key              INT             NOT NULL,
    restaurant_key        INT             NOT NULL,
    running_avg_rating    DECIMAL(3,2),
    total_reviews         INT,
    new_reviews_today     INT,
    rating_5_star_pct     DECIMAL(5,2),
    rating_1_star_pct     DECIMAL(5,2)
);
```

#### C. Conformed Dimensions & Connection Diagram

```
              ┌──────────────────┐
              │   dim_date       │
              │  (shared by ALL)  │
              └────────┬─────────┘
                       │ date_key
     ┌─────────────────┼────────────────────┐
     │                 │                    │
┌────▼──────┐   ┌──────▼───────┐   ┌───────▼───────┐
│  dim_     │   │   dim_       │   │   dim_        │
│  customer │   │   restaurant │   │   rider       │
│           │   │              │   │               │
│ ─── ───── │   │ ─── ─────   │   │  ─── ─────   │
└────┬──────┘   └──────┬───────┘   └───────┬───────┘
     │                 │                    │
     │          ┌──────┼───────┐            │
     │          │      │       │            │
     │     ┌────▼──┐ ┌─▼────┐ ┌▼────────┐  │
     │     │fact_  │ │fact_ │ │fact_    │  │
     │     │order  │ │daily │ │rider_   │  │
     │     │_lifecycle│ │rev  │ │perf    │  │
     │     └───┬───┘ └──┬───┘ └──┬─────┘  │
     │         │        │        │         │
     │         └────────┼────────┘         │
     │                  │                  │
     │         ┌───────▼──────┐            │
     │         │fact_restaurant│            │
     │         │_ratings       │            │
     └─────────┴──────────────┘            │
                                           │
                              (rider assigned to order
                               joins here)
```

**Conformed dimensions used across multiple fact tables:**

| Dimension | Used By |
|---|---|
| `dim_date` | **All four** fact tables — the universal conformed dimension |
| `dim_restaurant` | `fact_order_lifecycle`, `fact_daily_revenue`, `fact_restaurant_ratings` |
| `dim_rider` | `fact_order_lifecycle`, `fact_rider_performance` |
| `dim_customer` | `fact_order_lifecycle` (could extend to `fact_daily_revenue` for customer-level revenue) |
| `dim_city` | `fact_daily_revenue` (outrigger of `dim_restaurant`) |

> **Interview takeaway:** `dim_date` is the most critical conformed dimension — it's the single field that lets you join all four fact tables for a "company-wide daily dashboard." `dim_restaurant` is the second most important, bridging lifecycle, revenue, and ratings.

---

## Summary: What You've Practiced

| Scenario | Skill Tested |
|---|---|
| 1 — Ride-sharing | Identifying the correct fact table type (transactional, periodic snapshot, accumulating snapshot) |
| 2 — E-commerce | Classifying measures as additive, semi-additive, or non-additive |
| 3 — Multi-fact query | Pre-aggregating at shared grain before joining across fact tables |
| 4 — Loan lifecycle | Designing an accumulating snapshot with milestone date keys |
| 5 — Warehouse inventory | Combining multiple fact types around conformed dimensions |
| Final exercise | End-to-end design: choosing types, writing schemas, identifying conformed dimensions |

### Key Takeaways

1. **Fact type follows the business question** — events → transactional, states → periodic snapshot, lifecycles → accumulating snapshot.
2. **Additive measures are the gold standard** — decompose non-additive measures into additive components whenever possible.
3. **Conformed dimensions are the glue** — shared dimension keys across fact tables enable multi-fact queries without fan-out.
4. **Accumulating snapshots are updated, not appended** — they are the only fact table type where UPDATE is the primary load operation (after the initial INSERT).
5. **Periodic snapshots trade storage for simplicity** — you can answer "what was the state on this day?" without complex date-range joins.

> **Next step:** Move on to [Module 6: Data Modeling Interview Questions](./28-data-modeling-interview-questions.md) to practice answering real interview prompts under time pressure.

---

### Related Lessons

- [Practice: Ridesharing Platform](./18-practice-ridesharing-platform.md)
- [Fact Table Types](./24-fact-table-types.md)
- [Additive, Semi-Additive, Non-Additive Measures](./25-additive-semi-additive-non-additive-measures.md)
- [Multi-Fact Query Strategies](./26-multi-fact-query-strategies.md)
