# Lesson 20: Practice — Online Advertising Platform Data Model (AdTech / RTB)

> **Principal Mentor Note:** Online Advertising (Google Ads, Meta Ads, The Trade Desk) is the most demanding high-throughput data modeling problem in industry. You must handle **100B+ ad auction/impression events/day**, **exact financial precision in micro-dollars** (`CPM` / `CPC` / `CPA`), **click-to-conversion attribution windows** (linking a purchase 7 days later back to an earlier impression/click), and **invalid traffic (IVT) / bot clawbacks**.

---

## 1. Requirements & Scope Assumptions

* **Scale:** `50B ad impressions/day`, `800M ad clicks/day`, `40M attributed conversions/day`, `5M advertisers`, `80M active ad creatives`.
* **Core Business Processes:**
  1. **Ad Impressions & Clicks** (`fct_ad_events` or `agg_ad_delivery_hourly` — captures auction clearing price, placement, user context, bot/IVT filtering).
  2. **Attributed Conversions** (`fct_ad_conversions` — links downstream purchase/signup events back to the winning `impression_id` or `click_id` under configurable attribution windows: `1-day view`, `7-day click`, `28-day click`).
  3. **Advertiser Budget & Pacing Snapshot** (`fct_campaign_daily_spend` — tracks daily budget caps, actual spend, overdelivery credits, and ROAS).
* **Key Analytical Queries:**
  * Campaign Performance Funnel: Impressions, Clicks, CTR, Spend, Conversions, CPA (`Spend / Conversions`), and ROAS (`Conversion Value / Spend`) sliced by Campaign, Ad Group, Creative Format, Publisher Placement, and Geo.
  * Multi-Window Attribution Comparison: Comparing `7-day click-through` vs. `1-day view-through` conversions for the same campaign.
  * Invalid Traffic (IVT) Adjustment Impact: Gross billed spend vs. Net post-bot-clawback spend.

---

## 2. Enterprise Bus Matrix

```
Business Process (Fact Table)           | Date | Hour | Advertiser | Campaign/AdGroup | Creative | Publisher/Slot | Audience Segment | Attr Model
----------------------------------------+------+------+------------+------------------+----------+----------------+------------------+-----------
fct_ad_impressions (Transaction - Raw)  |  X   |  X   |     X      |        X         |    X     |       X        |        X         |
agg_ad_delivery_hourly (Aggregate)      |  X   |  X   |     X      |        X         |    X     |       X        |        X         |
fct_ad_conversions (Transaction)        |  X   |  X   |     X      |        X         |    X     |       X        |        X         |     X
fct_campaign_daily_spend (Periodic)     |  X   |      |     X      |        X         |          |                |                  |
```

---

## 3. High-Level Constellation Diagram

```
+------------------------+       +---------------------+       +------------------------+
| dim_campaign_hierarchy |       |  dim_date [CONFORM] |       | dim_creative [SCD2]    |
| [SCD2 - Flattened]     |       |  PK: date_sk        |       | PK: creative_sk        |
| PK: ad_group_sk        |       +----------+----------+       | NK: creative_id        |
| advertiser_id,         |            ^            ^           | format (VIDEO, BANNER) |
| campaign_id,           |            |            |           | landing_page_domain    |
| bid_strategy (CPA/CPC) |            |            |           +-----------+------------+
+-----------+------------+            |            |                       | 1
            | 1                       |            |                       |
            |                 +-------+----+  +----+---------------+       | N
            | N               | agg_ad_    |  | fct_ad_conversions |<------+
            +---------------->| delivery_  |  | [FACT-TXN]         |
                              | hourly     |  | Grain: 1 /         |
+------------------------+    | [AGG-HRLY] |  | (conv_id, attr_mod)|    +------------------------+
| dim_publisher_slot     |--->| Grain: 1 / |  +----+---------------+    | dim_attribution_model  |
| [SCD1]                 |1:N | (hr,adgrp, |       ^                    | [SCD1 - Static]        |
| publisher_id, app_id,  |    | creat,slot)|       | 1:N                | LAST_CLICK_7D,         |
| placement_position     |    +------------+  +----+---------------+    | VIEW_THROUGH_1D,       |
+------------------------+                    | dim_audience_seg   |    | DATA_DRIVEN_MULTI      |
                                              | [MINI-DIM]         |    +------------------------+
                                              +--------------------+
```

---

## 4. Core Production DDL & AdTech Modeling Patterns

```sql
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE dim_campaign_hierarchy (
    ad_group_sk             BIGINT PRIMARY KEY,
    ad_group_id             VARCHAR(64) NOT NULL,
    campaign_id             VARCHAR(64) NOT NULL,
    campaign_name           VARCHAR(255) NOT NULL,
    advertiser_id           VARCHAR(64) NOT NULL,
    advertiser_name         VARCHAR(255) NOT NULL,
    industry_vertical       VARCHAR(64),
    billing_model           VARCHAR(16) NOT NULL,  -- CPM, CPC, CPA
    bid_strategy            VARCHAR(32) NOT NULL,  -- TARGET_ROAS, MAX_CONVERSIONS, MANUAL_CPC
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

-- ============================================================================
-- HOURLY DELIVERY AGGREGATE (Primary BI Serving Fact — compresses 50B -> 80M rows/day)
-- Grain: One row per (delivery_date_sk, delivery_hour, ad_group_sk, creative_sk, slot_sk, audience_sk)
-- Note: Raw impression/click stream lives in Silver/Hot-30d Iceberg for ML & Attribution joins.
-- ============================================================================
CREATE TABLE agg_ad_delivery_hourly (
    delivery_date_sk        INT NOT NULL,
    delivery_hour_utc       TINYINT NOT NULL,      -- 0..23
    ad_group_sk             BIGINT NOT NULL,
    advertiser_id           VARCHAR(64) NOT NULL,  -- Denormalized for fast advertiser portal pruning
    creative_sk             BIGINT NOT NULL,
    slot_sk                 INT NOT NULL,
    audience_sk             INT NOT NULL,
    -- Additive Constituent Counts (NEVER store pre-divided CTR or CPM!)
    gross_impressions       BIGINT NOT NULL,
    valid_impressions       BIGINT NOT NULL,       -- Post-IVT / bot filtration
    viewable_impressions    BIGINT NOT NULL,       -- MRC standard (>=50% pixels for >=1s)
    valid_clicks            BIGINT NOT NULL,
    -- Exact Micro-Dollar Precision (1 USD = 1,000,000 micros; eliminates float rounding drift!)
    gross_spend_micros_usd  BIGINT NOT NULL,
    ivt_credit_micros_usd   BIGINT NOT NULL,       -- Automated bot clawback credits
    net_billed_micros_usd   BIGINT NOT NULL,       -- gross_spend - ivt_credit
    -- HyperLogLog Sketch for Exact-ish Unique Reach across arbitrary rollups!
    unique_user_hll_sketch  VARBINARY
)
PARTITION BY (delivery_date_sk)
CLUSTER BY (advertiser_id, ad_group_sk);

-- ============================================================================
-- FACT: ATTRIBUTED CONVERSIONS (Transaction Fact)
-- Grain: One row per (conversion_id, attribution_model_sk)
-- Why include attribution_model_sk in grain? Advertisers compare 7d-Click vs 1d-View
-- side-by-side; filtering WHERE attribution_model_code = 'LAST_CLICK_7D' avoids double-counting.
-- ============================================================================
CREATE TABLE fct_ad_conversions (
    conversion_id           VARCHAR(64) NOT NULL,
    attribution_model_sk    INT NOT NULL,          -- 1=LAST_CLICK_7D, 2=VIEW_THROUGH_1D, 3=LINEAR_28D
    conversion_date_sk      INT NOT NULL,          -- Date conversion occurred
    impression_date_sk      INT NOT NULL,          -- Role-playing FK: Date original ad was shown!
    conversion_at_utc       TIMESTAMP NOT NULL,
    ad_exposure_at_utc      TIMESTAMP NOT NULL,
    seconds_to_conversion   INT NOT NULL,          -- Lag between ad click/view and purchase
    ad_group_sk             BIGINT NOT NULL,
    advertiser_id           VARCHAR(64) NOT NULL,
    creative_sk             BIGINT NOT NULL,
    slot_sk                 INT NOT NULL,
    audience_sk             INT NOT NULL,
    conversion_type         VARCHAR(32) NOT NULL,  -- PURCHASE, LEAD_SIGNUP, APP_INSTALL
    -- Additive Attribution Measures (supports fractional credit e.g. 0.5000 for multi-touch)
    attributed_conv_weight  DECIMAL(7,4) NOT NULL, -- 1.0000 for last-click; fractional for multi-touch
    conversion_value_micros_usd BIGINT NOT NULL,
    PRIMARY KEY (conversion_id, attribution_model_sk)
)
PARTITION BY (conversion_date_sk)
CLUSTER BY (advertiser_id, ad_group_sk);
```

---

## 5. Validation Query (Full Campaign ROAS, CTR, CPM & Viewability)

```sql
-- Query: Advertiser Campaign Performance (Last 7 Days, 7-Day Click Attribution)
WITH delivery AS (
    SELECT
        ad_group_sk,
        SUM(valid_impressions)                              AS impressions,
        SUM(viewable_impressions)                           AS viewable_impressions,
        SUM(valid_clicks)                                   AS clicks,
        SUM(net_billed_micros_usd) / 1000000.0              AS net_spend_usd
    FROM agg_ad_delivery_hourly
    WHERE advertiser_id = 'ADV-44102'
      AND delivery_date_sk BETWEEN 20261108 AND 20261114
    GROUP BY 1
),
conversions AS (
    SELECT
        c.ad_group_sk,
        SUM(c.attributed_conv_weight)                       AS conversions,
        SUM(c.conversion_value_micros_usd) / 1000000.0      AS conv_value_usd
    FROM fct_ad_conversions c
    JOIN dim_attribution_model m ON c.attribution_model_sk = m.attribution_model_sk
    WHERE c.advertiser_id = 'ADV-44102'
      AND c.conversion_date_sk BETWEEN 20261108 AND 20261114
      AND m.model_code = 'LAST_CLICK_7D'
    GROUP BY 1
)
SELECT
    ch.campaign_name,
    COALESCE(d.impressions, 0)                              AS impressions,
    COALESCE(d.clicks, 0)                                   AS clicks,
    ROUND(COALESCE(d.clicks, 0) * 100.0 / NULLIF(d.impressions, 0), 3) AS ctr_pct,
    ROUND(COALESCE(d.net_spend_usd, 0), 2)                  AS spend_usd,
    ROUND(COALESCE(d.net_spend_usd, 0) * 1000.0 / NULLIF(d.impressions, 0), 2) AS ecpm_usd,
    COALESCE(c.conversions, 0)                              AS conversions,
    ROUND(COALESCE(c.conv_value_usd, 0) / NULLIF(d.net_spend_usd, 0), 2) AS roas
FROM dim_campaign_hierarchy ch
LEFT JOIN delivery d ON ch.ad_group_sk = d.ad_group_sk
LEFT JOIN conversions c ON ch.ad_group_sk = c.ad_group_sk
WHERE ch.advertiser_id = 'ADV-44102'
  AND ch.is_current = TRUE
ORDER BY spend_usd DESC;
```

---

## 6. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "Why use `BIGINT` micro-dollars (`spend_micros_usd`) instead of `FLOAT` or `DOUBLE` in AdTech?"**
   * **Expected Insight:** At `50B` impressions/day billed on a `$2.45 CPM` (`$0.00245` per impression), IEEE-754 floating-point rounding errors across billions of rows create thousands of dollars in penny-drift discrepancies during monthly advertiser invoicing. Storing integer micro-dollars (`$1 = 1,000,000 micros`) in `BIGINT` guarantees **exact integer arithmetic** and faster CPU vectorization than wide `DECIMAL(38,12)` columns.
2. **Top-Up 2: "A click happens on Nov 1, and the user purchases on Nov 6 (within the 7-day click window). Should `fct_ad_conversions` be partitioned by `conversion_date_sk` (Nov 6) or `impression_date_sk` (Nov 1)?"**
   * **Expected Insight:** Partition by `conversion_date_sk` (Nov 6) so each day's new conversions append cleanly into today's partition without constantly rewriting historical partitions from 1–28 days ago! Store `impression_date_sk` as a role-playing foreign key column so analysts can still report conversions by *impression cohort date* (`GROUP BY impression_date_sk`) when measuring cohort ROAS maturity curves.

---

## 7. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Pre-aggregates delivery (`agg_ad_delivery_hourly`) with HyperLogLog reach sketches** and keeps raw 50B/day events in short-TTL storage for attribution/ML. | Exposes a 50B-row/day atomic impression table directly to BI dashboards. |
| **Uses integer micro-dollars (`BIGINT`)** for exact CPM/CPC billing math. | Uses `FLOAT` / `DOUBLE` for monetary bid and spend columns. |
| **Pre-aggregates delivery and conversions independently by `ad_group_sk` before joining** (avoiding Chasm Traps). | Joins `agg_ad_delivery_hourly` directly to `fct_ad_conversions` on `ad_group_sk`, multiplying spend by conversion count! |