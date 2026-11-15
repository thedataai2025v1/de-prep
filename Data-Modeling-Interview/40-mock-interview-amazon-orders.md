# Lesson 40 — Mock Interview: Amazon-Style Marketplace Orders

**Level:** Senior / Staff / Principal Data Engineer
**Duration:** 60-90 minutes
**Case type:** End-to-end source-to-warehouse data modeling

## How to Use This

This mock interview covers a large marketplace platform like Amazon, focusing on:
- Multi-grain order lifecycle (placement → fulfillment → delivery → returns)
- Supply chain and shipment tracking across distribution centers
- Product catalog with hierarchical categories
- Bridge tables for seller-to-category mappings
- Advanced partitioning and aggregate strategies

---

## Scenario

You are the lead data engineer for a global e-commerce marketplace (like Amazon) where third-party sellers list products alongside first-party inventory. The platform handles millions of orders per day across multiple countries.

### Business Requirements

1. **Track the full order lifecycle** — placed, paid, fulfilled, shipped, delivered, returned
2. **Seller analytics** — sales by seller, by category, by region
3. **Product catalog management** — hierarchical categories, product attributes, price history
4. **Shipment tracking** — per-package delivery status across distribution centers
5. **Return & refund analysis** — return reasons, refund amounts, seller liability
6. **Promotion & coupon tracking** — discounts applied at order or item level

### Key Entities

- **Customers** — buyers on the platform (SCD Type 2 for address changes)
- **Sellers** — third-party and first-party sellers
- **Products** — catalog items with categories, attributes, pricing
- **Orders** — header-level order information
- **Order Lines** — individual line items within an order
- **Shipments** — physical shipment events (can split one order into multiple shipments)
- **Returns** — return requests with reasons and status
- **Promotions** — coupons, discounts, lightning deals

---

## Phase 1: Requirements Clarification (10 min)

### Questions to Ask

**Business / Scope:**
- What are the key business metrics? (GMV, order count, return rate, delivery time?)
- Do first-party and third-party sellers have different data flows?
- How do promotions work? (order-level discount, item-level coupon, lightning deal?)
- What is the return window? (30 days, seasonal variations?)

**Technical:**
- What are the source systems? (order service, fulfillment service, payment service — each separate?)
- Target warehouse? (Databricks, Snowflake, Redshift?)
- Required latency? (real-time order tracking vs. daily sales reports)
- Data volume? (e.g., 5M orders/day, 20M line items/day)

---

## Phase 2: Source-to-Warehouse Modeling (20 min)

### Dimension Tables

| Dimension | Grain | Key Columns | SCD Type |
|-----------|-------|-------------|----------|
| `dim_customer` | One row per customer version | `customer_id`, `email`, `country`, `signup_date` | Type 2 (address changes) |
| `dim_seller` | One row per seller version | `seller_id`, `name`, `country`, `seller_type`, `rating` | Type 2 (rating, status changes) |
| `dim_product` | One row per product version | `product_id`, `name`, `brand`, `price`, `seller_key` | Type 2 (price changes) |
| `dim_category` | Hierarchical categories | `category_id`, `parent_category_id`, `name`, `level` | Type 1 |
| `dim_shipment_method` | One row per method | `method_id`, `name`, `expected_days` | Type 1 |
| `dim_date` | Standard date dimension | `date_key`, `day`, `month`, `quarter`, `year` | — |
| `dim_promotion` | One row per promotion | `promotion_id`, `code`, `discount_type`, `discount_value`, `valid_from`, `valid_to` | Type 1 |

### Fact Tables

| Fact Table | Grain | Key Columns |
|------------|-------|-------------|
| `fct_orders` | One row per order | `order_id`, `customer_key`, `date_key`, `total_amount`, `total_tax`, `total_discount`, `status`, `created_at`, `paid_at`, `delivered_at` |
| `fct_order_lines` | One row per order line | `order_line_id`, `order_key`, `product_key`, `seller_key`, `quantity`, `unit_price`, `line_total`, `discount_amount` |
| `fct_shipments` | One row per shipment | `shipment_id`, `order_line_key`, `date_key`, `shipment_method_key`, `status`, `shipped_at`, `delivered_at` |
| `fct_returns` | One row per return | `return_id`, `order_line_key`, `date_key`, `reason`, `refund_amount`, `status` |
| `fct_promotion_usage` | One row per promotion application | `usage_id`, `order_key`, `promotion_key`, `date_key`, `discount_amount` |

### Key Design Decisions

**1. Order Lines as Separate Fact:**
- `fct_order_lines` is a **dependent fact** of `fct_orders`
- One order → multiple line items → multiple sellers
- Line-level grain enables seller-specific and product-specific analytics

**2. Order-to-Shipment Split:**
- One order can split into multiple shipments (different warehouses)
- `fct_shipments` links to `order_line_key` (not `order_key`) since shipments are per-line-item
- A single line item can also be split into multiple shipments (rare, but possible)

**3. Category Hierarchy:**
- `dim_category` with `parent_category_id` enables roll-up at any level
- `dim_product` links to a leaf category; roll up via recursive CTE or category bridge
- For performance, pre-compute `product_category_path` in `dim_product`

**4. Seller-to-Category Bridge:**
- Sellers sell across many categories; use `bridge_seller_category` for category-level seller analytics
- Many-to-many: one seller → many categories, one category → many sellers

---

## Phase 3: Advanced Questions (15 min)

### Q: How do you handle order line splits across shipments?

**Answer (Principal Level):**
- When one line item is split (e.g., 5 units, 3 from Warehouse A, 2 from Warehouse B):
  - `fct_shipments` has two rows, each with `quantity_shipped`
  - Link back to `order_line_key` to trace which order line each shipment belongs to
  - Total shipped = `SUM(quantity_shipped)` across shipments for a given `order_line_key`
- This is a case for **grain awareness** — the shipment fact is finer-grain than the order line fact

### Q: How do you calculate on-time delivery rate?

**Answer:**
- Compare `delivered_at` (from `fct_shipments`) against `expected_delivery_date` (from `shipment_method_key` + `shipped_at`)
- On-time = `CASE WHEN delivered_at <= expected_delivery_date THEN 1 ELSE 0 END`
- Aggregate at seller/day grain for dashboards

### Q: How do you model returns that are partial (some line items, not all)?

**Answer:**
- `fct_returns` is at **order line grain** — each return links to a specific `order_line_key`
- Partial return = only some order lines have corresponding rows in `fct_returns`
- Return rate at order level = `COUNT(DISTINCT order_lines_with_returns) / COUNT(DISTINCT order_lines)`

### Q: How do you handle product price changes over time?

**Answer:**
- `dim_product` is SCD Type 2 — price changes create a new version row
- `fct_order_lines` stores `unit_price` at time of sale (snapshot) — so historical sales always show the price at purchase time
- `dim_product.current_price` reflects the latest version for current catalog queries

---

## Phase 4: Architecture & Optimization (10 min)

### Medallion Architecture

| Layer | Purpose | Key Operations |
|-------|---------|----------------|
| **Bronze** | Raw events from order/fulfillment/payment services | `MERGE` by event ID, append-only |
| **Silver** | Conformed dimensions + order facts | SCD Type 2 for product/customer/seller |
| **Gold** | Aggregates for business dashboards | `agg_seller_daily_sales`, `agg_category_performance`, `agg_delivery_metrics` |

### Key Gold Aggregates

```
agg_seller_daily_sales
- seller_key, date_key
- order_count, line_count, gross_revenue, net_revenue, return_count, refund_total
```

```
agg_category_performance
- category_key, date_key
- order_count, revenue, avg_delivery_days, return_rate
```

### Performance

- **Partition** `fct_order_lines` by `date_key`
- **Cluster** `fct_shipments` by `shipment_method_key` for delivery analytics
- **Materialize** gold aggregates with daily incremental refresh
- **Bloom filter** on `order_id` for order-level lookups

---

## Evaluation Rubric

| Level | What They Show |
|-------|---------------|
| **Senior** | Correct star schema with order/order_line/shipment separation, SCD Type 2 for product price |
| **Staff** | Bridge table for seller-category, handles partial returns at line grain, gold aggregate design |
| **Principal** | Handles line-item shipment splits, quantifies scale, explains cluster/partition strategy, connects to FinOps |

---

## Related Lessons

- [Lesson 24 — Fact Table Types](./24-fact-table-types.md) — dependent fact for order lines
- [Lesson 31 — Advanced Dimension Design](./31-advanced-dimension-design.md) — category hierarchy
- [Lesson 32 — Bridge Tables and Factless Facts](./32-bridge-tables-and-factless-facts.md)
- [Lesson 10 — Data Volume and Scalability](./10-data-volume-and-scalability-considerations.md)
- [Lesson 28 — Date-Based Partitioning](./28-date-based-partitioning.md)
- [Lesson 41 — Principal-Level Cheat Sheet](./41-principal-level-cheat-sheet.md)
