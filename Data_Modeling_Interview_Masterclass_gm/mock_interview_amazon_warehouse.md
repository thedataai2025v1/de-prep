# Lesson 36: Mock Interview 6 — Design a Data Warehouse Schema for Amazon

> **Principal Mentor Note:** Amazon is the ultimate **Retail + 3P Marketplace + Supply Chain Logistics + Prime Subscription** data modeling capstone. Unlike a simple single-brand e-commerce store, Amazon requires modeling **1P Retail vs. 3P FBA (Fulfilled by Amazon) vs. 3P FBM (Fulfilled by Merchant)** unit economics, **split multi-warehouse shipments** (one customer order of 3 items shipping from 2 different Fulfillment Centers in 3 packages), **Prime membership attribution**, and **daily Fulfillment Center inventory & stockout snapshots**.

## 1. Phase 1: Clarifying Questions & Scope Alignment (Minutes 0–6)

### Interviewer Prompt

> *"Design the analytical data warehouse for Amazon's core E-Commerce & Fulfillment business. Leadership wants to track Gross Merchandise Sales (GMS), 1P Retail Gross Margin vs. 3P Marketplace Seller Commission Revenue, Prime vs. Non-Prime order behavior, Click-to-Door delivery speed, and Fulfillment Center inventory health."*

### Candidate's 4-Bucket Discovery Questions

1. **Fulfillment Channels & Business Models:** *"Should we distinguish 1P Retail (Amazon buys wholesale and sells directly, tracking COGS and vendor rebates) from 3P Marketplace Sellers—specifically FBA (Fulfilled by Amazon, charging storage/pick-pack fees) vs. FBM (Fulfilled by Merchant)?"*

   * *Interviewer:* *"Yes—comparing 1P Retail margin against 3P FBA and FBM seller take-rates is a core executive priority."*

2. **Multi-Shipment Split Orders & Returns:** *"When a single customer order (`order_id`) contains 4 items that split into 2 packages from different Fulfillment Centers—and 1 item is later returned—how should we model order financials vs. shipment logistics vs. refunds?"*

   * *Interviewer:* *"Model order items, package shipments, and customer returns cleanly without double-counting item revenue or delivery fees."*

3. **Prime Attribution & Historical Truth:** *"When a customer joins or cancels Amazon Prime, or an ASIN moves between product categories or 1P/3P buy-box winners, do we need exact point-in-time state at order placement (`as-was`)?"*

   * *Interviewer:* *"Yes, Prime status at checkout and historical ASIN/Seller attributes at order time are mandatory."*

4. **Scale, Latency & Compliance:** *"What daily scale, freshness SLA, retention, and GDPR/PII rules apply?"*

   * *Interviewer:* *"Assume 25M orders/day, 65M order line items/day, 40M outbound packages/day, 300M ASIN catalog across 250 Fulfillment Centers, 1-hour freshness for Retail/Ops dashboards, 7-year financial retention, and strict GDPR compliance."*

## 2. Phase 2: Enterprise Bus Matrix & Grain Contract (Minutes 6–12)

### Enterprise Bus Matrix

```
Business Process (Fact Table)              | Date | Customer | ASIN/Product | Seller (1P/3P) | Fulfillment Ctr | Carrier | Promo/Deal
-------------------------------------------+------+----------+--------------+----------------+-----------------+---------+-----------
fct_order_line_items (Transaction Fact)    |  X   |    X     |      X       |       X        |        X        |         |     X
fct_shipment_packages (Accumulating Snap)  |  X   |    X     |              |       X        |        X        |    X    |
fct_shipment_package_items (Bridge/Txn)    |  X   |          |      X       |                |        X        |    X    |
fct_returns_refunds (Transaction Fact)     |  X   |    X     |      X       |       X        |        X        |         |
fct_fc_inventory_daily (Periodic Snapshot) |  X   |          |      X       |       X        |        X        |         |


```

### Explicit Grain Declarations (Eliminating the Order-vs-Shipment Fan-Out Trap)

1. **`fct_order_line_items` (Transaction Fact):** One row per `(order_id, line_item_id)`—captures customer checkout financials (`GMS`, discounts, allocated shipping, `1P COGS` vs. `3P Seller Referral/FBA Fees`, net contribution profit) and Prime status at checkout.

2. **`fct_shipment_packages` (Accumulating Snapshot):** One row per physical outbound parcel (`package_id`), tracking warehouse-to-doorstep milestones (`ordered_at -> fc_picked_at -> carrier_handed_off_at -> out_for_delivery_at -> delivered_at`) and **Click-to-Door (`C2D`)** SLA compliance (`Same-Day`, `Next-Day`, `2-Day`).

3. **`fct_shipment_package_items` (Allocation / Bridge Fact):** One row per `(package_id, order_id, line_item_id)` with `units_in_package`—resolving the `M:N` mapping when one order splits into multiple boxes or partial backorders without inflating `fct_order_line_items` revenue!

4. **`fct_fc_inventory_daily` (Periodic Snapshot):** One row per active `(snapshot_date_sk, fc_sk, asin_sk, seller_sk)`, tracking semi-additive `eod_sellable_units`, `eod_inbound_units`, `days_of_supply`, and `is_out_of_stock`.

## 3. Phase 3: High-Level Constellation Diagram & DDL (Minutes 12–26)

```
+---------------------+       +----------------------+       +-----------------------+
| dim_customer [SCD2] |       |  dim_date [CONFORM]  |       | dim_asin [SCD2]       |
| PK: customer_sk     |       |  PK: date_sk         |       | PK: asin_sk           |
| NK: customer_id     |       +----------+-----------+       | NK: asin_id           |
| is_prime_member,    |            ^            ^            | gl_product_group,     |
| prime_plan_type     |            |            |            | category_l1/l2/l3,    |
+----------+----------+            |            |            | brand, unit_weight_g  |
           | 1                     |            |            +-----+-------------+---+
           |                       |            |                  | 1           | 1
           | N             +-------+----+  +----+--------------+   |             |
           +-------------->| fct_order_ |  | fct_fc_inventory_ |<--+             | N
                           | line_items |  | daily [PER-SNAP]  |                 v
+---------------------+    | [TXN-ITEM] |  | Grain: 1 /        |       +-------------------+
| dim_seller [SCD2]   |--->| Grain: 1 / |  | (date,fc,asin,slr)|       | dim_fulfillment_  |
| PK: seller_sk       |1:N | line_item  |  +-------------------+       | center [SCD1]     |
| seller_type:        |    +-----+------+                              | fc_code, fc_type, |
| 1P_RETAIL, 3P_FBA,  |          | 1 (Degenerate order_id)             | region, timezone  |
| 3P_FBM, commission% |          v                                     +-------------------+
+---------------------+    +-----+--------------------+                        ^
                           | fct_shipment_packages    |                        | 1:N
                           | [ACC-SNAP]               |------------------------+
                           | Grain: 1 / package_id    |
                           | Click-to-Door (C2D) SLAs |
                           +--------------------------+


```

### Production DDL (Apache Iceberg / Snowflake)

```
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE gold.dim_asin (
    asin_sk                 BIGINT PRIMARY KEY,
    asin_id                 CHAR(10) NOT NULL,     -- Amazon Standard Identification Number
    gl_product_group        VARCHAR(64) NOT NULL,  -- General Ledger group (ELECTRONICS, APPAREL, GROCERY)
    category_l1             VARCHAR(100) NOT NULL,
    category_l2             VARCHAR(100) NOT NULL,
    brand_name              VARCHAR(128) NOT NULL,
    size_tier               VARCHAR(32) NOT NULL,  -- STANDARD_SMALL, STANDARD_LARGE, OVERSIZE
    unit_weight_grams       INT NOT NULL,
    is_hazmat               BOOLEAN NOT NULL,
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

CREATE TABLE gold.dim_seller (
    seller_sk               BIGINT PRIMARY KEY,
    seller_id               VARCHAR(64) NOT NULL,  -- 'AMAZON_1P' for first-party retail
    fulfillment_channel     VARCHAR(16) NOT NULL,  -- 1P_RETAIL, 3P_FBA, 3P_FBM
    seller_tier             VARCHAR(24) NOT NULL,  -- ENTERPRISE_BRAND, PRO_MERCHANT, INDIVIDUAL
    referral_fee_pct        DECIMAL(5,4) NOT NULL, -- Category commission rate (SCD2 tracked!)
    country_code            CHAR(2) NOT NULL,
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

-- ============================================================================
-- FACT 1: ORDER LINE ITEMS (Unified 1P + 3P Transaction Fact)
-- Grain: One row per (order_id, line_item_id)
-- ============================================================================
CREATE TABLE gold.fct_order_line_items (
    order_id                VARCHAR(64) NOT NULL,  -- Degenerate Dimension
    line_item_id            VARCHAR(64) NOT NULL,  -- Degenerate Dimension
    ordered_date_local_sk   INT NOT NULL,          -- Marketplace local date partition key
    ordered_at_utc          TIMESTAMP NOT NULL,
    customer_sk             BIGINT NOT NULL,       -- Historical SCD2 SK (captures Prime status!)
    asin_sk                 BIGINT NOT NULL,       -- Historical SCD2 SK
    seller_sk               BIGINT NOT NULL,       -- Historical SCD2 SK (1P vs 3P_FBA vs 3P_FBM)
    origin_fc_sk            INT NOT NULL,          -- -1 sentinel for 3P_FBM merchant-shipped items
    promo_sk                INT NOT NULL DEFAULT -1,
    marketplace_country     CHAR(2) NOT NULL,      -- US, UK, DE, IN, JP
    is_prime_order_item     BOOLEAN NOT NULL,      -- Stamped at checkout for cohort comparison
    promised_delivery_speed VARCHAR(24) NOT NULL,  -- SAME_DAY, NEXT_DAY, TWO_DAY, STANDARD
    currency_code           CHAR(3) NOT NULL,
    fx_rate_to_usd          DECIMAL(18,8) NOT NULL,
    -- Additive Unit & Financial Measures (Supports unified 1P + 3P P&L!)
    quantity_ordered        INT NOT NULL,
    gross_merchandise_sales_usd DECIMAL(14,2) NOT NULL, -- Total customer item spend (GMS)
    promo_discount_usd      DECIMAL(12,2) NOT NULL,
    allocated_shipping_rev_usd DECIMAL(12,2) NOT NULL,
    -- 1P Retail Cost vs. 3P Marketplace Fee Economics
    product_cogs_1p_usd     DECIMAL(14,2) NOT NULL,-- Wholesale cost for 1P; 0.00 for 3P!
    seller_commission_3p_usd DECIMAL(12,2) NOT NULL,-- Referral + FBA fees for 3P; 0.00 for 1P!
    est_fulfillment_cost_usd DECIMAL(12,2) NOT NULL,-- Pick/pack/linehaul cost (1P & 3P_FBA)
    net_contribution_profit_usd DECIMAL(14,2) NOT NULL,
    PRIMARY KEY (order_id, line_item_id)
)
PARTITION BY (ordered_date_local_sk)
CLUSTER BY (seller_sk, asin_sk);

-- ============================================================================
-- FACT 2: OUTBOUND SHIPMENT PACKAGES (Accumulating Snapshot for Click-to-Door SLA)
-- Grain: One row per physical outbound parcel (package_id)
-- ============================================================================
CREATE TABLE gold.fct_shipment_packages (
    package_id              VARCHAR(64) PRIMARY KEY,
    order_id                VARCHAR(64) NOT NULL,  -- Degenerate link to order
    ordered_date_local_sk   INT NOT NULL,
    shipped_date_local_sk   INT NOT NULL DEFAULT -1,
    delivered_date_local_sk INT NOT NULL DEFAULT -1,
    ordered_at_utc          TIMESTAMP NOT NULL,
    fc_picked_at_utc        TIMESTAMP,
    carrier_shipped_at_utc  TIMESTAMP,
    delivered_at_utc        TIMESTAMP,
    promised_delivery_utc   TIMESTAMP NOT NULL,
    customer_sk             BIGINT NOT NULL,
    origin_fc_sk            INT NOT NULL,
    carrier_sk              INT NOT NULL,          -- AMZL (Amazon Logistics), UPS, FEDEX, USPS
    is_terminal_delivered   BOOLEAN NOT NULL,
    -- Pre-Computed Logistics Bottleneck Lags (Minutes)
    click_to_ship_mins      INT,                   -- Warehouse processing speed
    ship_to_door_mins       INT,                   -- Carrier transit speed
    click_to_door_mins      INT,                   -- End-to-end C2D customer experience
    is_delivered_on_time    BOOLEAN                -- delivered_at_utc <= promised_delivery_utc
)
PARTITION BY (ordered_date_local_sk)
CLUSTER BY (origin_fc_sk, is_terminal_delivered);


```

## 4. Phase 4: Query Validation (Minutes 26–33)

```
-- Query 1: 1P Retail vs. 3P FBA vs. 3P FBM — GMS Share, Contribution Margin & Prime Penetration
SELECT
    d.year_quarter,
    s.fulfillment_channel,
    a.gl_product_group,
    SUM(li.gross_merchandise_sales_usd)                             AS total_gms_usd,
    ROUND(
        SUM(CASE WHEN li.is_prime_order_item THEN li.gross_merchandise_sales_usd ELSE 0 END) * 100.0 /
        NULLIF(SUM(li.gross_merchandise_sales_usd), 0),
    2)                                                              AS prime_gms_share_pct,
    SUM(li.net_contribution_profit_usd)                             AS contribution_profit_usd,
    ROUND(
        SUM(li.net_contribution_profit_usd) * 100.0 /
        NULLIF(SUM(li.gross_merchandise_sales_usd), 0),
    2)                                                              AS contribution_margin_pct
FROM gold.fct_order_line_items li
JOIN gold.dim_date d ON li.ordered_date_local_sk = d.date_sk
JOIN gold.dim_seller s ON li.seller_sk = s.seller_sk
JOIN gold.dim_asin a ON li.asin_sk = a.asin_sk
WHERE d.calendar_year = 2026
  AND li.marketplace_country = 'US'
GROUP BY 1, 2, 3
ORDER BY 1, 4 DESC;


```

## 5. Phase 5: Principal Curveballs & "Top-Up" Questions (Minutes 33–45)

1. **Curveball 1 (Buy-Box Switching Between 1P and 3P on the Same ASIN):** *"At 10:00 AM, Amazon 1P wins the Buy Box on `ASIN B09XYZ123` (`$49.99`). At 2:00 PM, Amazon 1P goes out of stock and `Seller_882` (3P FBA) wins the Buy Box (`$52.99`). How does your schema keep ASIN attributes clean without triggering SCD Type 2 churn on `dim_asin` every hour?"*

   * **Principal Answer:** Never store the current Buy Box winner or current price on `dim_asin`! `dim_asin` stores intrinsic product catalog attributes (`brand`, `category`, `weight`, `dimensions`), while `seller_sk` and `gross_merchandise_sales_usd` are stamped per transaction on `fct_order_line_items` at checkout time.

2. **Curveball 2 (Subscribe & Save Recurring Orders & Digital Kindle/Video Orders):** *"How does your model adapt when a customer orders a physical book + a digital Kindle eBook in the same cart, or receives a recurring monthly Subscribe & Save delivery?"*

   * **Principal Answer:** Add `order_source_type` (`ONE_TIME_CHECKOUT`, `SUBSCRIBE_AND_SAVE`, `BUY_NOW`) and `is_digital_fulfillment` on `fct_order_line_items`. Digital items have `origin_fc_sk = -2 ('Not Applicable - Digital')` and emit zero rows in `fct_shipment_packages`, keeping physical Fulfillment Center utilization metrics 100% pure.

3. **Curveball 3 (300M ASINs $\times$ 250 FCs Inventory Snapshot Scale):** *"`300M ASINs * 250 FCs = 75 billion` potential rows per day in `fct_fc_inventory_daily`! How do you store this economically?"*

   * **Principal Answer:** Enforce **Active-Node Sparsity**: a given ASIN is typically stocked in `5–25` regional FCs, not all 250 FCs globally. Only emit rows in `fct_fc_inventory_daily` where an ASIN has an active FC bin assignment (`eod_sellable_units > 0` OR `eod_inbound_units > 0` OR `is_out_of_stock_alert = TRUE`), reducing daily volume by ~95% (`~3.5B` compact rows/day), and retain daily granularity for 90 days before rolling up to weekly/month-end snapshots.