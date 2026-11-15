# Lesson 15: Practice — E-Commerce Platform Data Model

> **Principal Mentor Note:** E-commerce is the quintessential data modeling interview domain because it tests **multi-grain modeling** (Order vs. Line Item vs. Shipment vs. Return), **inventory snapshots**, **promo/discount allocation**, and **SCD Type 2 product/customer history**. Master this blueprint and you can adapt it to any retail, B2B procurement, or digital goods prompt.

---

## 1. Requirements & Scope Assumptions

* **Business Model:** Global B2C + 3P Marketplace E-Commerce platform (`~15M orders/day`, `~40M line items/day`, `50M SKU catalog`, `200M customers`).
* **Core Business Processes:**
  1. Checkout & Order Line-Item Sales (GMV, Discounts, Taxes, Shipping, Net Revenue)
  2. Fulfillment & Multi-Package Shipping Lifecycle
  3. Returns & Refunds (Booking refunds accurately on return date while linking to original order)
  4. Daily Warehouse Inventory Snapshot (Semi-additive stock-on-hand & stockout tracking)
* **Key Analytical Queries:**
  * Net Revenue & Margin by Product Category *as of the order date* (SCD Type 2) vs. *current* category hierarchy.
  * Order fulfillment funnel SLA (`Ordered -> Paid -> Packed -> Shipped -> Delivered`).
  * Daily inventory turnover and stockout rate by fulfillment center.

---

## 2. Enterprise Bus Matrix

```
Business Process (Fact Table)           | Date | Time | Customer | Product | Seller | Warehouse | Promo
----------------------------------------+------+------+----------+---------+--------+-----------+------
fct_orders (Transaction - Header)       |  X   |  X   |    X     |         |        |           |  X
fct_order_line_items (Transaction)      |  X   |  X   |    X     |    X    |   X    |     X     |  X
fct_order_fulfillment (Accum. Snapshot) |  X   |  X   |    X     |         |   X    |     X     |
fct_returns (Transaction)               |  X   |  X   |    X     |    X    |   X    |     X     |
fct_inventory_daily (Periodic Snapshot) |  X   |      |          |    X    |   X    |     X     |
```

---

## 3. High-Level Constellation Diagram

```
+---------------------+       +----------------------+       +---------------------+
| dim_customer [SCD2] |       |  dim_date [CONFORM]  |       | dim_product [SCD2]  |
| PK: customer_sk     |       |  PK: date_sk (INT)   |       | PK: product_sk      |
| NK: customer_id     |       +----------+-----------+       | NK: sku_id          |
+----------+----------+            ^            ^            +----------+----------+
           | 1                     |            |                     | 1
           |                       |            |                     |
           | N             +-------+----+  +----+-------+             | N
           +-------------->| fct_orders |  | fct_order_ |<------------+
                           | [TXN-HDR]  |  | line_items |
+---------------------+    | Grain:     |  | [TXN-ITEM] |    +---------------------+
| dim_promo [SCD1]    |--->| 1/order_id |  | Grain:     |<---| dim_seller [SCD2]   |
| PK: promo_sk        | 1:N+------------+  | 1/line_item|1:N +---------------------+
+---------------------+         | 1        +-----+------+
                                |                | 1         +---------------------+
                                | 1              |      1:N  | dim_warehouse [SCD1]|
                    +-----------v-------+        |     +---->| PK: warehouse_sk    |
                    | fct_order_        |        |     |     +---------------------+
                    | fulfillment [ACC] |        v     |
                    | Grain: 1/order_id |   +----------+----------+
                    +-------------------+   | fct_inventory_daily |
                                            | [PERIODIC SNAPSHOT] |
                                            | Grain: 1/sku/wh/day |
                                            +---------------------+
```

---

## 4. Core Production DDL (Snowflake / Iceberg SQL)

```sql
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE dim_product (
    product_sk          BIGINT PRIMARY KEY,
    sku_id              VARCHAR(64) NOT NULL,      -- Durable Natural Key
    product_name        VARCHAR(255) NOT NULL,
    brand_name          VARCHAR(128),
    category_l1         VARCHAR(100),              -- Flattened hierarchy (eliminates snowflake joins)
    category_l2         VARCHAR(100),
    category_l3         VARCHAR(100),
    unit_cost_usd       DECIMAL(12,2),
    is_perishable       BOOLEAN,
    -- SCD Type 2 Audit Columns
    valid_from_utc      TIMESTAMP NOT NULL,
    valid_to_utc        TIMESTAMP NOT NULL,        -- '9999-12-31 23:59:59' for current
    is_current          BOOLEAN NOT NULL
);

CREATE TABLE dim_customer (
    customer_sk         BIGINT PRIMARY KEY,
    customer_id         VARCHAR(64) NOT NULL,
    signup_cohort_month DATE NOT NULL,             -- Conformed cohort attribute
    loyalty_tier        VARCHAR(32),               -- SCD2 tracked
    country_code        CHAR(2),                   -- SCD2 tracked
    region_code         VARCHAR(32),
    -- Note: PII (email, full_name, street_address) isolated in dim_customer_pii for GDPR
    valid_from_utc      TIMESTAMP NOT NULL,
    valid_to_utc        TIMESTAMP NOT NULL,
    is_current          BOOLEAN NOT NULL
);

-- ============================================================================
-- FACT 1: ORDER LINE ITEMS (Atomic Transaction Fact)
-- Grain: One row per line item within a customer order (order_id, line_item_seq)
-- ============================================================================
CREATE TABLE fct_order_line_items (
    order_id                VARCHAR(64) NOT NULL,  -- Degenerate Dimension
    line_item_seq           INT NOT NULL,          -- Degenerate Dimension
    ordered_date_sk         INT NOT NULL,          -- FK -> dim_date
    ordered_at_utc          TIMESTAMP NOT NULL,
    customer_sk             BIGINT NOT NULL,       -- Pre-resolved historical SCD2 SK
    customer_id             VARCHAR(64) NOT NULL,  -- Natural key for fast 'as-is' joins
    product_sk              BIGINT NOT NULL,       -- Pre-resolved historical SCD2 SK
    seller_sk               BIGINT NOT NULL,       -- Pre-resolved historical SCD2 SK
    warehouse_sk            INT NOT NULL,          -- Fulfillment origin warehouse
    promo_sk                INT NOT NULL,          -- -1 sentinel for 'No Promo'
    currency_code           CHAR(3) NOT NULL,
    fx_rate_to_usd          DECIMAL(18,8) NOT NULL,
    -- Additive Measures
    quantity_ordered        INT NOT NULL,
    gross_item_amount_usd   DECIMAL(12,2) NOT NULL,
    item_discount_usd       DECIMAL(12,2) NOT NULL,
    allocated_order_disc_usd DECIMAL(12,2) NOT NULL, -- Proportional share of cart-level promo
    allocated_shipping_usd  DECIMAL(12,2) NOT NULL, -- Proportional share of order shipping fee
    tax_amount_usd          DECIMAL(12,2) NOT NULL,
    net_item_revenue_usd    DECIMAL(12,2) NOT NULL, -- gross - discounts
    cogs_usd                DECIMAL(12,2) NOT NULL, -- quantity * historical unit_cost_usd
    PRIMARY KEY (order_id, line_item_seq)
)
PARTITION BY (ordered_date_sk)
CLUSTER BY (seller_sk, product_sk);

-- ============================================================================
-- FACT 2: ORDER FULFILLMENT LIFECYCLE (Accumulating Snapshot Fact)
-- Grain: One row per customer order_id (updated via bounded MERGE)
-- ============================================================================
CREATE TABLE fct_order_fulfillment (
    order_id                VARCHAR(64) PRIMARY KEY,
    ordered_date_sk         INT NOT NULL,
    paid_date_sk            INT,
    packed_date_sk          INT,
    shipped_date_sk         INT,
    delivered_date_sk       INT,
    customer_sk             BIGINT NOT NULL,
    warehouse_sk            INT NOT NULL,
    current_order_status    VARCHAR(32) NOT NULL,  -- PLACED, PAID, PACKED, SHIPPED, DELIVERED, CANCELED
    ordered_at_utc          TIMESTAMP NOT NULL,
    paid_at_utc             TIMESTAMP,
    packed_at_utc           TIMESTAMP,
    shipped_at_utc          TIMESTAMP,
    delivered_at_utc        TIMESTAMP,
    -- Pre-computed Milestone Lag Measures (Seconds)
    order_to_ship_sec       BIGINT,
    ship_to_deliver_sec     BIGINT,
    total_fulfillment_sec   BIGINT,
    is_sla_breached         BOOLEAN
)
PARTITION BY (ordered_date_sk);

-- ============================================================================
-- FACT 3: DAILY INVENTORY SNAPSHOT (Periodic Snapshot Fact)
-- Grain: One row per (snapshot_date_sk, warehouse_sk, product_sk)
-- ============================================================================
CREATE TABLE fct_inventory_daily (
    snapshot_date_sk        INT NOT NULL,
    warehouse_sk            INT NOT NULL,
    product_sk              BIGINT NOT NULL,
    seller_sk               BIGINT NOT NULL,
    -- Semi-Additive Stock Measures (DO NOT SUM ACROSS TIME)
    eod_on_hand_units       INT NOT NULL,
    eod_reserved_units      INT NOT NULL,
    eod_available_units     INT NOT NULL,
    eod_inventory_value_usd DECIMAL(14,2) NOT NULL,
    -- Additive Flow Measures for the Day
    units_received_today    INT NOT NULL,
    units_shipped_today     INT NOT NULL,
    is_stockout_eod         BOOLEAN NOT NULL,
    PRIMARY KEY (snapshot_date_sk, warehouse_sk, product_sk)
)
PARTITION BY (snapshot_date_sk)
CLUSTER BY (warehouse_sk, product_sk);
```

---

## 5. Validation Query (Proving Multi-Grain & SCD2 Correctness)

```sql
-- Query: Monthly Net Margin & Return-Adjusted Revenue by Historical Product Category
SELECT
    d.year_month,
    p.category_l1                                       AS historical_category,
    SUM(li.net_item_revenue_usd)                        AS gross_sales_usd,
    COALESCE(SUM(r.refund_amount_usd), 0)               AS refunds_usd,
    SUM(li.net_item_revenue_usd) - COALESCE(SUM(r.refund_amount_usd), 0) AS net_revenue_usd,
    SUM(li.net_item_revenue_usd - li.cogs_usd)          AS gross_margin_usd
FROM fct_order_line_items li
JOIN dim_date d
  ON li.ordered_date_sk = d.date_sk
JOIN dim_product p
  ON li.product_sk = p.product_sk                       -- Fast equi-join to historical SCD2 version!
LEFT JOIN (
    -- Pre-aggregate returns to line-item grain to prevent fan-out if partial returns occur twice
    SELECT order_id, line_item_seq, SUM(refund_amount_usd) AS refund_amount_usd
    FROM fct_returns
    GROUP BY 1, 2
) r
  ON li.order_id = r.order_id
 AND li.line_item_seq = r.line_item_seq
WHERE d.calendar_year = 2026
GROUP BY 1, 2
ORDER BY 1, 5 DESC;
```

---

## 6. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "A customer buys a 3-item bundle SKU (`BUNDLE_99`) that physically ships as 3 individual warehouse SKUs (`ITEM_A`, `ITEM_B`, `ITEM_C`). How do you model this so both Marketing (bundle sales) and Warehouse Ops (component inventory depletion) get accurate numbers?"**
   * **Expected Insight:** Use a **Bill-of-Materials (BOM) Bridge Table** (`bridge_product_bundle(bundle_product_sk, component_product_sk, component_qty, revenue_allocation_pct)`), or record the customer-facing purchase at the bundle SKU level in `fct_order_line_items` while exploding component picks in `fct_warehouse_picks` linked via `(order_id, line_item_seq)`.
2. **Top-Up 2: "At 50M SKUs across 100 warehouses (`5 billion` possible combinations/day), `fct_inventory_daily` would generate 5B rows/day if stored densely! How do you avoid sparse table bloat?"**
   * **Expected Insight:** Store **sparse snapshots** (only insert rows where `eod_on_hand_units > 0` OR `units_received_today > 0` OR `units_shipped_today > 0` OR `is_stockout_eod = TRUE` for an active listed SKU). Never store millions of rows for discontinued SKUs with zero stock and zero movement!