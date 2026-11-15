# Module 3: High-Level Model Diagrams
## Lesson 3.3 — Practice: E-commerce Platform

> *"The best way to learn dimensional modeling is to build a complete warehouse from scratch."* This lesson walks through a full design exercise for an e-commerce platform like Amazon — requirements, diagram, table specs, and discussion — so you can replicate the approach in any interview.

---

### 1. The Interview Prompt

You're interviewing for a **Senior Data Engineer** role. The interviewer opens with:

> *"Design a data warehouse for an e-commerce platform like Amazon. Walk me through your thinking from requirements to model."*

This is deliberately vague. Your goal: **ask structured questions**, **capture requirements**, **draw a high-level diagram**, and **defend every design choice** — all within a 20–25 minute window.

---

### 2. Requirements Gathering

Below is a realistic back-and-forth. Your questions are in **bold**; the interviewer's responses are in *italics*.

#### Phase 1 — Business Context

**"What business decisions will this warehouse enable? Which stakeholder groups are the primary consumers?"**

*"We need to track sales performance, analyze customer buying behavior, optimize inventory levels, and support a recommendation engine. The main consumers are the executive team, marketing, supply chain, and data science."*

**"What are the top 3–4 KPIs the business cares about most?"**

*"Gross merchandise value (GMV), conversion rate, average order value (AOV), return rate, and inventory turnover."*

#### Phase 2 — Scale & Constraints

**"Let's quantify the scale. Volume of transactions, users, and products?"**

*"About 10 million transactions per month, 100 million registered users, and roughly 1 million active products. We're growing at about 20% year over year."*

**"What are your latency requirements?"**

*"Most analytics dashboards refresh daily. But inventory levels need to be near real-time — under 5 minutes — because our supply chain team makes restocking decisions throughout the day."*

**"Data retention?"**

*"We need 3 years of detailed transaction data for analysis and compliance. Beyond that, we'd keep monthly aggregates indefinitely."*

#### Phase 3 — Data Sources

**"Where does the data originate? What systems are we pulling from?"**

*"We have an orders system, a product catalog, a customer accounts database, a promotions/discounts engine, a returns management system, and an inventory warehouse management system. All PostgreSQL today."*

#### Structured Requirements Summary

| Area | Detail |
|------|--------|
| **Business Goals** | Track sales, analyze customer behavior, optimize inventory, support recommendations |
| **Key Metrics** | GMV, conversion rate, AOV, return rate, inventory turnover |
| **Volume** | 10M transactions/month, 100M users, 1M products |
| **Latency** | Daily batch for analytics; < 5 min for inventory |
| **Retention** | 3 years detailed; aggregates beyond |

---

### 3. High-Level Model Diagram

We'll use a **star schema** — a Kimball approach with fact tables at the center and dimension tables fanning out.

```
                         ┌─────────────────┐
                         │  dim_promotion  │
                         │ (promo_id,     │
                         │  code, type,   │
                         │  discount,     │
                         │  start/end)    │
                         └────────┬────────┘
                                  │
┌──────────────┐       ┌─────────────────┐
│ dim_location │       │   dim_customer  │
│ (location_id │       │ (customer_id,   │
│  region,     │       │  signup_date,   │
│  country,    │       │  segment,       │
│  timezone)   │       │  lifetime_value │
└──────┬───────┘       └────────┬────────┘
       │                        │
┌──────────────┐       ┌────────┴────────┐       ┌─────────────────┐
│ dim_category │       │   dim_product   │       │  dim_date       │
│ (category_id │       │ (product_id,    │       │ (date_key,      │
│  category,   │◄──────│  name, price,   │──────►│  day, week,     │
│  hierarchy)  │       │  brand, weight, │       │  month, quarter,│
└──────┬───────┘       │  is_active)     │       │  year, holiday) │
       │               └────────┬────────┘       └────────┬────────┘
       │                        │                        │
       │               ┌────────┴────────┐       ┌───────┴──────────┐
       │               │  fact_returns   │       │ fact_inventory_  │
       │               │ ┌─────────────┐ │       │ snapshot         │
       │               │ │ - return_id │ │       │ ┌───────────────┐│
       │               │ │ - order_id  │ │       │ │ - date_key    ││
       │               │ │ - product_id│ │       │ │ - product_id  ││
       │               │ │ - reason_id │ │       │ │ - warehouse   ││
       │               │ │ - qty       │ │       │ │ - qty_on_hand ││
       │               │ │ - refund    │ │       │ │ - reorder_pt  ││
       └──────────────►│ └─────────────┘ │       │ └───────────────┘│
                       │                 │       └──────────────────┘
                       └────────┬────────┘
                                │
                       ┌────────┴────────┐
                       │   fact_orders   │
                       │ ┌─────────────┐ │
                       │ │ - order_id  │ │
                       │ │ - date_key  │ │
                       │ │ - customer  │ │
                       │ │ - product   │ │
                       │ │ - category  │ │
                       │ │ - location  │ │
                       │ │ - promo_key │ │
                       │ │ - qty       │ │
                       │ │ - unit_price│ │
                       │ │ - discount  │ │
                       │ │ - total     │ │
                       │ │ - ship_date │ │
                       │ └─────────────┘ │
                       └─────────────────┘
```

#### Grain of Each Table

| Table | Grain | One row represents |
|-------|-------|--------------------|
| **fact_orders** | One row per product line item per order | A single product within an order (an order with 3 products = 3 rows) |
| **fact_returns** | One row per returned line item | A return event for a specific product from an order |
| **fact_inventory_snapshot** | One row per product per warehouse per day | The state of inventory for one product at one location on one day |

---

### 4. Detailed Table Definitions

#### Fact Tables

**fact_orders** *(Grain: one row per product line item per order)*

| Column | Type | Source / Notes |
|--------|------|---------------|
| `order_line_key` | PK (surrogate) | Surrogate key for the fact row |
| `order_id` | STRING | Natural key from orders system |
| `date_key` | FK → dim_date | Order date |
| `ship_date_key` | FK → dim_date | Shipment date (nullable until shipped) |
| `customer_key` | FK → dim_customer | Who placed the order |
| `product_key` | FK → dim_product | What was purchased |
| `category_key` | FK → dim_category | Product's primary category |
| `location_key` | FK → dim_location | Ship-to location |
| `promotion_key` | FK → dim_promotion | Applied promo (NULL if none) |
| `quantity` | INT | Units ordered |
| `unit_price` | DECIMAL | Price per unit at time of sale |
| `discount_amount` | DECIMAL | Promo or loyalty discount |
| `line_total` | DECIMAL | `quantity × unit_price - discount_amount` |
| `tax_amount` | DECIMAL | Sales tax |
| `shipping_cost` | DECIMAL | Allocated per line item |
| `order_status` | STRING | Degenerate dimension: placed, shipped, delivered, cancelled |

**fact_returns** *(Grain: one row per returned line item)*

| Column | Type | Source / Notes |
|--------|------|---------------|
| `return_key` | PK (surrogate) | Surrogate key |
| `return_id` | STRING | Natural key from returns system |
| `date_key` | FK → dim_date | Return request date |
| `order_key` | FK → fact_orders | Links back to original order line |
| `customer_key` | FK → dim_customer | Who initiated the return |
| `product_key` | FK → dim_product | Product being returned |
| `return_reason_code` | STRING | Degenerate dim: defective, wrong_size, changed_mind, etc. |
| `quantity_returned` | INT | Units returned |
| `refund_amount` | DECIMAL | Money returned to customer |
| `restocking_fee` | DECIMAL | Fee deducted from refund |
| `return_status` | STRING | Degenerate dim: pending, approved, rejected, refunded |

**fact_inventory_snapshot** *(Grain: one row per product per warehouse per day — periodic snapshot)*

| Column | Type | Source / Notes |
|--------|------|---------------|
| `snapshot_key` | PK (surrogate) | Surrogate key |
| `date_key` | FK → dim_date | Snapshot date |
| `product_key` | FK → dim_product | Product tracked |
| `warehouse_location_key` | FK → dim_location | Warehouse location |
| `quantity_on_hand` | INT | Current stock level |
| `quantity_reserved` | INT | Allocated to open orders |
| `quantity_in_transit` | INT | En route to warehouse |
| `reorder_point` | INT | Threshold triggering a reorder |
| `days_of_supply` | DECIMAL | Derived: `on_hand / avg_daily_demand` |
| `is_stockout` | BOOL | Flag for zero inventory |

#### Dimension Tables

**dim_date** *(Standard calendar dimension — static, no SCD)*

| Column | Type | Notes |
|--------|------|-------|
| `date_key` | PK | INT (YYYYMMDD) |
| `full_date` | DATE | Human-readable |
| `day_of_week` | INT | 1–7 |
| `day_name` | STRING | Monday, etc. |
| `week_number` | INT | ISO week |
| `month_number` | INT | 1–12 |
| `month_name` | STRING | January, etc. |
| `quarter` | INT | 1–4 |
| `year` | INT | 2024, etc. |
| `is_holiday` | BOOL | National holidays |
| `is_weekend` | BOOL | Sat/Sun |

**dim_customer** *(SCD Type 2 — track changes to segment, lifetime_value)*

| Column | Type | SCD? | Notes |
|--------|------|------|-------|
| `customer_key` | PK (surrogate) | — | Surrogate key |
| `customer_id` | STRING | SCD-2 | Natural key |
| `signup_date` | DATE | SCD-1 | Set at creation, never changes |
| `email_domain` | STRING | SCD-2 | For cohort analysis |
| `customer_segment` | STRING | SCD-2 | New, active, at-risk, churned |
| `lifetime_value` | DECIMAL | SCD-2 | Cumulative net revenue |
| `preferred_location_key` | FK → dim_location | SCD-2 | Primary ship-to |
| `effective_start` | DATE | SCD-2 | — |
| `effective_end` | DATE | SCD-2 | NULL = current |
| `is_current` | BOOL | SCD-2 | TRUE = active version |

**dim_product** *(SCD Type 2 — track price, brand, active status changes)*

| Column | Type | SCD? | Notes |
|--------|------|------|-------|
| `product_key` | PK (surrogate) | — | Surrogate key |
| `product_id` | STRING | SCD-2 | SKU or UPC |
| `product_name` | STRING | SCD-2 | Current name |
| `brand` | STRING | SCD-2 | Manufacturer brand |
| `category_key` | FK → dim_category | SCD-2 | Primary category |
| `base_price` | DECIMAL | SCD-2 | Manufacturer suggested price |
| `weight_kg` | DECIMAL | SCD-1 | For shipping calc |
| `is_active` | BOOL | SCD-2 | Active listing or discontinued |
| `effective_start` | DATE | SCD-2 | — |
| `effective_end` | DATE | SCD-2 | NULL = current |
| `is_current` | BOOL | SCD-2 | TRUE = active version |

**dim_category** *(SCD Type 1 — slow-changing hierarchy, overwrites OK)*

| Column | Type | Notes |
|--------|------|-------|
| `category_key` | PK | Surrogate key |
| `category_id` | STRING | Natural key |
| `category_name` | STRING | e.g. "Electronics" |
| `parent_category_key` | FK | Self-referencing for hierarchy |
| `category_level` | INT | 1 = top-level, 2 = sub, 3 = leaf |
| `display_order` | INT | For UI sorting |

**dim_location** *(SCD Type 1 — geography changes rarely)*

| Column | Type | Notes |
|--------|------|-------|
| `location_key` | PK | Surrogate key |
| `location_id` | STRING | Natural key |
| `country` | STRING | ISO 2-letter code |
| `region` | STRING | State/province |
| `city` | STRING | — |
| `timezone` | STRING | IANA timezone name |
| `is_warehouse` | BOOL | TRUE if used as fulfillment center |

**dim_promotion** *(SCD Type 1 — promotions are time-bounded, rarely updated)*

| Column | Type | Notes |
|--------|------|-------|
| `promotion_key` | PK | Surrogate key |
| `promotion_code` | STRING | e.g. "SUMMER25" |
| `promotion_type` | STRING | Percentage, fixed-amount, BOGO, free-shipping |
| `discount_value` | DECIMAL | Value of the discount |
| `start_date` | DATE | — |
| `end_date` | DATE | — |
| `applicable_category_key` | FK → dim_category | NULL = store-wide |

---

### 5. Discussion Points

#### Why This Grain?

| Table | Grain Rationale |
|-------|----------------|
| **fact_orders** | One row per line item (not per order) because a single order can contain multiple products with different prices, discounts, and categories. If we used order-level grain, we'd have to repeat data or use a bridge table. Line-item grain lets us roll up GMV, AOV, and category sales with simple `SUM` and `AVG`. |
| **fact_returns** | One row per returned line item mirrors the fact_orders grain, enabling a direct join between returns and orders. This lets us calculate return rate as `COUNT(returned line items) / COUNT(sold line items)`. |
| **fact_inventory_snapshot** | Periodic snapshot (daily) because inventory is a **stateful** measure — we need to know "how much was on hand on this day." A transaction fact can't capture levels; a snapshot captures the state at a point in time. |

#### How the Model Supports Key Metrics

| Metric | How the Model Delivers It |
|--------|--------------------------|
| **GMV** | `SUM(line_total)` from fact_orders, filtered by date_key and any dimension |
| **Conversion Rate** | `COUNT(DISTINCT customer_key with orders) / COUNT(DISTINCT customer_key in traffic source)` — requires joining with a traffic/web analytics fact |
| **AOV** | `SUM(line_total) / COUNT(DISTINCT order_id)` — rolls up across line items |
| **Return Rate** | `COUNT(return_key) / COUNT(order_line_key)` with matching product_key and date ranges |
| **Inventory Turnover** | `SUM(sold quantity) / AVG(quantity_on_hand)` joining fact_orders to fact_inventory_snapshot |

#### Trade-offs Made

| Trade-off | Choice | Why |
|-----------|--------|-----|
| **Star vs. Snowflake** | Star schema (denormalized dimensions) | Simpler queries, better performance at this scale (1M products). Snowflake adds joins for marginal storage savings. |
| **Line-item vs. order-level grain** | Line-item grain | One order can have 5–10 products with different categories and prices. Line-item grain avoids repetition and simplifies category-level analytics. |
| **SCD-2 on customer & product** | Full history tracking | Needed for accurate time-based analysis (e.g., "What was this product's price when it sold in 2022?"). SCD-1 would lose this history. |
| **Daily inventory snapshots vs. real-time** | Daily snapshots in the warehouse + separate near-real-time feed | Daily snapshots suffice for trend analysis; a parallel streaming layer (Kafka → materialized view) serves the < 5 min inventory requirement. |
| **Separate fact_returns vs. order status flag** | Separate fact table | Returns have their own lifecycle, reasons, and refunds. A status flag on fact_orders can't capture return-level detail or link to the returns processing timeline. |

---

### Interviewer Questions You'll Face

| # | Question | What They're Testing |
|---|----------|---------------------|
| 1 | *"Why did you choose line-item grain for fact_orders instead of order-level?"* | Understanding of grain: line-item captures multi-product orders; order-level would require repeating product data or complex bridge tables |
| 2 | *"How would you calculate AOV (average order value) from your model?"* | Knowledge of grain awareness: `SUM(line_total) / COUNT(DISTINCT order_id)`, not just `AVG(line_total)` which would be wrong |
| 3 | *"Your inventory needs real-time but your warehouse is daily — how do you reconcile this?"* | Architecture thinking: separate near-real-time pipeline (streaming) for inventory dashboards; daily snapshot for analytics; both backed by same source |
| 4 | *"How would you handle a product that moves from one category to another? Does that break historical sales by category?"* | SCD awareness: SCD-2 on dim_product preserves the category at time of sale; joining fact_orders on `product_key` + `date_key` range ensures historical accuracy |

---

### Top-Up Questions (Senior-Level Probes)

| # | Question | What It Demonstrates |
|---|----------|---------------------|
| 1 | *"If a customer places an order, ships it, then returns half the items — how does your model handle the relationship between the original order and the partial return?"* | Grain alignment: fact_returns links to specific order_line_key rows, not just order_id. This handles partial returns, multiple return events per order, and different return reasons per product. |
| 2 | *"We want to compute inventory turnover, but fact_orders is daily-aggregated in the warehouse while fact_inventory_snapshot is also daily — what happens if an order ships the same day inventory is counted?"* | Temporal join awareness: use the inventory snapshot from the day *before* the order ships, or use an end-of-day snapshot that accounts for that day's shipments. This is a classic point-in-time join problem. |
| 3 | *"We're expanding into a new country with different tax rules and a new currency. What changes does your model need?"* | Dimension extensibility: add `currency` to dim_location or fact_orders; add `tax_rate` or a `dim_tax` dimension; ensure all monetary metrics can be converted to a common currency at query time. |
| 4 | *"How would you model a 'bundle' sale where three products are sold together at a discount? Does your current grain handle this?"* | Edge-case awareness: the line-item grain can handle bundles by splitting the bundle discount proportionally across line items, or by adding a `bundle_id` degenerate dimension to group related line items. |

---

### Practice Exercise

**Scenario:** The interviewer continues:

> *"Great work on the base model. Now, we're launching a subscription service — customers can subscribe to get monthly deliveries of products (think Amazon Subscribe & Save). Subscribers get a 15% recurring discount. We need to track subscriber acquisition, retention, and revenue. How would you extend your model?"*

**1. Identify the new business metrics** needed for the subscription program. List at least 4.
2. **Design 1–2 new fact tables** and any new or modified dimension tables. Specify the grain of each.
3. **Explain how subscription orders relate to regular orders** in your model — should they share a fact table or be separate?
4. **Describe how you'd track subscriber retention** (e.g., monthly cohort analysis of subscribers who stay vs. cancel).
5. **What SCD strategy** would you use for tracking a customer's subscription status over time?

<details>
<summary>💡 Click to reveal a hint</summary>

Think about these before sketching: is a subscription a **new business process** (needs its own fact table) or just a **modifier on existing orders** (needs new columns/dimensions on fact_orders)? How do you track the **subscription lifecycle** separately from individual orders? Consider an **accumulating snapshot fact** for the subscription lifecycle alongside regular order line items.

</details>

<details>
<summary>💡 Click to reveal a model answer outline</summary>

**New metrics:**
- Monthly subscribers (new + active)
- Subscriber churn rate (cancelled / active at start of month)
- Subscriber revenue (sum of subscription orders, vs. one-time purchases)
- Orders per subscriber per month (frequency)

**New tables:**
- `fact_subscription_snapshot` (periodic snapshot, grain: one row per subscription per month) — tracks subscription status, products included, discount level, delivery frequency
- `dim_subscription` (SCD-2) — tracks subscription plan details
- Modify `fact_orders` with `subscription_key` FK to link regular order lines back to the subscription that generated them

**Retention tracking:**
- Cohort analysis on subscription start date, tracking month-over-month active status from fact_subscription_snapshot
- Calculate retention as: `active_subscribers_in_month_N / original_cohort_size`

</details>

---

### Key Takeaways

| Principle | Application |
|-----------|-------------|
| **Grain is the most important decision** | Line-item grain enables category-level GMV, AOV, and return-rate calculations |
| **Match fact tables to business processes** | Orders, returns, and inventory are distinct processes — each gets its own fact table |
| **SCD strategy follows business need** | Customer/product use SCD-2 (history matters); location/category use SCD-1 |
| **Near-real-time needs a separate path** | Daily warehouse + streaming layer for inventory avoids bloating the main model |
| **Extensions build on existing structure** | Subscriptions extend fact_orders via a new FK and add a periodic snapshot |

---

### Related Lessons

- [Creating High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md)
- [Evolving Models Based on Changing Requirements](./14-evolving-models-based-on-changing-requirements.md)
- [Practice: Social Media Analytics](./16-practice-social-media-analytics.md)
- [Slowly Changing Dimensions: Types 1-4](./21-slowly-changing-dimensions-types-1-4.md)
