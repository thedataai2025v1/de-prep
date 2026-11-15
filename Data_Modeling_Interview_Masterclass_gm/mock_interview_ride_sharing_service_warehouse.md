# Lesson 31: Mock Interview 1 — Design a Data Warehouse Schema for a Ride-Sharing Service

> **Principal Mentor Note:** This is a complete, end-to-end **45-minute Principal Data Engineering mock interview simulation** for a global ride-sharing platform (Uber / Ola / Lyft). Every section mirrors the exact pacing, whiteboard diagrams, production DDLs, validation SQL, and "Minute-35 Principal Curveballs" you will face in a live loop.

---

## 1. Phase 1: Clarifying Questions & Scope Alignment (Minutes 0–6)

### Interviewer Prompt
> *"Design the analytical data warehouse for a global ride-sharing platform like Uber or Ola. We want to understand marketplace health, driver supply utilization, pricing/surge effectiveness, and unit economics."*

### Candidate's 4-Bucket Discovery Questions
1. **Business Scope & Processes:** *"Should we focus on core on-demand passenger rides (`Request -> Dispatch -> Pickup -> Dropoff -> Fare Settlement`), or also include Uber Eats delivery and scheduled airport rides? Also, do we need to track driver online availability when they are idle between trips?"*
   * *Interviewer:* *"Focus on passenger rides, including shared/pooled rides, and yes—driver online availability and idle time are critical for Operations."*
2. **Core Metrics & Edge Cases:** *"How should we model canceled trips (rider vs. driver cancellations vs. no-supply timeouts), cancellation fees, tolls, promo discounts, and multi-currency payouts?"*
   * *Interviewer:* *"Track all request outcomes so we can compute conversion and cancellation rates. Store both local currency and USD locked at trip time."*
3. **Historical Truth (SCD Type 2 vs. Type 1):** *"When a driver's loyalty tier (`Blue/Gold/Platinum`) or commission rate changes, does Finance need historical trips reported under the rate active at the time of the trip (`as-was`), while Ops also views current driver status (`as-is`)?"*
   * *Interviewer:* *"Yes, exact historical tier/commission at trip time is mandatory."*
4. **Scale, Latency & Compliance:** *"What scale are we designing for, what is our freshness SLA, and do we have GDPR/PII erasure requirements?"*
   * *Interviewer:* *"Assume 30M completed trips/day, 50M trip requests/day across 800 cities, 15-minute freshness for city Ops dashboards, 5-year retention, and strict GDPR compliance."*

---

## 2. Phase 2: Enterprise Bus Matrix & Grain Contract (Minutes 6–12)

### Enterprise Bus Matrix
```
Business Process (Fact Table)              | Date | Time | Rider | Driver | Vehicle | Origin H3 | Dest H3 | Product | Promo
-------------------------------------------+------+------+-------+--------+---------+-----------+---------+---------+------
fct_trip_requests (Accumulating Snapshot)  |  X   |  X   |   X   |   X    |    X    |     X     |    X    |    X    |  X
fct_trips (Transaction Fact)               |  X   |  X   |   X   |   X    |    X    |     X     |    X    |    X    |  X
fct_driver_supply_intervals (Transaction)  |  X   |  X   |       |   X    |    X    |     X     |         |         |
fct_pooled_trip_groups (Transaction)       |  X   |  X   |       |   X    |    X    |     X     |    X    |    X    |
fct_zone_liquidity_15min (Periodic Snap)   |  X   |  X   |       |        |         |     X     |         |    X    |
```

### Explicit Grain Declarations
1. **`fct_trip_requests` (Accumulating Snapshot):** One row per rider trip request (`request_id`), tracking every lifecycle milestone from upfront quote to terminal outcome (`COMPLETED`, `RIDER_CANCELED`, `DRIVER_CANCELED`, `NO_SUPPLY`).
2. **`fct_trips` (Transaction Fact):** One row per completed or fee-bearing canceled rider trip (`trip_id`), capturing exact financial unit economics (GBV, surge, tolls, promos, driver payout, net platform take).
3. **`fct_driver_supply_intervals` (Transaction Fact):** One row per contiguous driver state interval (`ONLINE_IDLE`, `EN_ROUTE_PICKUP`, `ON_TRIP`), enabling exact denominator calculation for Driver Utilization (`On-Trip Mins / Total Online Mins`).

---

## 3. Phase 3: High-Level Constellation Diagram & DDL (Minutes 12–26)

```
+---------------------+       +----------------------+       +-----------------------+
| dim_rider [SCD2]    |       |  dim_date [CONFORM]  |       | dim_driver [SCD2]     |
| PK: rider_sk        |       |  PK: date_sk         |       | PK: driver_sk         |
| NK: rider_id        |       +----------+-----------+       | NK: driver_id         |
+----------+----------+            ^            ^            | tier, commission_pct  |
           | 1                     |            |            +-----+-------------+---+
           |                       |            |                  | 1           | 1
           | N             +-------+----+  +----+--------------+   |             |
           +-------------->| fct_trip_  |  | fct_trips [TXN]   |<--+             | N
                           | requests   |  | Grain: 1 row /    |                 v
+---------------------+    | [ACC-SNAP] |  | financial trip_id |    +-----------------------+
| dim_h3_zone [SCD1]  |--->| Grain: 1 / |  +----+--------------+    | fct_driver_supply_    |
| PK: h3_res8_index   |1:N | request_id |       ^                   | intervals [TXN]       |
| city, country, tz,  |    +------------+       | 1:N               | Grain: 1 state slice  |
| zone_category       |                +--------+--------+          +-----------------------+
+---------------------+                | dim_vehicle     |
   ^ (Role-Playing:                    | [SCD2]          |          +-----------------------+
   |  origin_h3_index &                | make, model, ev |          | dim_rider_pii [VAULT] |
   |  dropoff_h3_index)                +-----------------+          | GDPR Crypto-Shredded  |
                                                                    +-----------------------+
```

### Production DDL (Apache Iceberg / Snowflake)

```sql
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE gold.dim_h3_zone (
    h3_res8_index           BIGINT PRIMARY KEY,    -- Uber H3 Hex (~0.74 km^2) for spatial grouping
    h3_res6_parent          BIGINT NOT NULL,       -- Neighborhood rollup (~36 km^2)
    city_id                 INT NOT NULL,
    city_name               VARCHAR(100) NOT NULL,
    country_code            CHAR(2) NOT NULL,
    timezone_name           VARCHAR(64) NOT NULL,  -- e.g., 'Asia/Kolkata', 'America/New_York'
    zone_category           VARCHAR(32) NOT NULL,  -- AIRPORT, CBD, SUBURB, TRANSIT_HUB
    is_surge_eligible       BOOLEAN NOT NULL
);

CREATE TABLE gold.dim_driver (
    driver_sk               BIGINT PRIMARY KEY,
    driver_id               VARCHAR(64) NOT NULL,  -- Durable Natural Key
    city_id                 INT NOT NULL,
    signup_cohort_month     DATE NOT NULL,
    driver_tier             VARCHAR(24) NOT NULL,  -- BLUE, GOLD, PLATINUM, DIAMOND (SCD2)
    commission_rate_pct     DECIMAL(5,4) NOT NULL, -- Historical platform take rate (SCD2)
    is_fleet_partner_driver BOOLEAN NOT NULL,
    -- PII (full_name, phone, license_num, bank_iban) isolated in dim_driver_pii!
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL,
    is_inferred_member      BOOLEAN DEFAULT FALSE
);

-- ============================================================================
-- FACT 1: TRIP REQUESTS FUNNEL (Accumulating Snapshot)
-- Grain: One row per ride request_id
-- ============================================================================
CREATE TABLE gold.fct_trip_requests (
    request_id              VARCHAR(64) PRIMARY KEY,
    requested_date_local_sk INT NOT NULL,          -- Local city date partition key
    requested_at_utc        TIMESTAMP NOT NULL,
    matched_at_utc          TIMESTAMP,
    driver_arrived_at_utc   TIMESTAMP,
    trip_started_at_utc     TIMESTAMP,
    trip_ended_at_utc       TIMESTAMP,
    canceled_at_utc         TIMESTAMP,
    rider_sk                BIGINT NOT NULL,
    driver_sk               BIGINT NOT NULL DEFAULT -1, -- -1 sentinel if NO_SUPPLY
    vehicle_sk              BIGINT NOT NULL DEFAULT -1,
    origin_h3_index         BIGINT NOT NULL,
    dropoff_h3_index        BIGINT NOT NULL,
    product_type_sk         INT NOT NULL,          -- GO, PREMIER, XL, AUTO, MOTO, POOL
    promo_sk                INT NOT NULL DEFAULT -1,
    terminal_status         VARCHAR(32) NOT NULL,  -- COMPLETED, RIDER_CANCELED, DRIVER_CANCELED, NO_SUPPLY
    is_terminal_state       BOOLEAN NOT NULL,      -- Prunes completed rows during 15m MERGE!
    -- Upfront Quote & Funnel Lag Measures
    upfront_quote_usd       DECIMAL(10,2) NOT NULL,
    surge_multiplier        DECIMAL(5,2) NOT NULL,
    quoted_pickup_eta_sec   INT NOT NULL,
    actual_pickup_wait_sec  INT,
    dispatch_offers_sent    INT NOT NULL
)
PARTITION BY (requested_date_local_sk)
CLUSTER BY (origin_h3_index, is_terminal_state);

-- ============================================================================
-- FACT 2: COMPLETED & FEE-BEARING TRIPS (Transaction Fact)
-- Grain: One row per financial trip_id
-- ============================================================================
CREATE TABLE gold.fct_trips (
    trip_id                 VARCHAR(64) PRIMARY KEY,
    request_id              VARCHAR(64) NOT NULL,  -- Degenerate link to funnel
    pooled_match_group_id   VARCHAR(64),           -- Degenerate link for UberPool shared routes
    completed_date_local_sk INT NOT NULL,
    completed_at_utc        TIMESTAMP NOT NULL,
    rider_sk                BIGINT NOT NULL,
    rider_id                VARCHAR(64) NOT NULL,  -- SCD Type 7 dual key for 'as-is' joins
    driver_sk               BIGINT NOT NULL,       -- Historical SCD2 SK (captures tier at trip time)
    driver_id               VARCHAR(64) NOT NULL,
    vehicle_sk              BIGINT NOT NULL,
    origin_h3_index         BIGINT NOT NULL,
    dropoff_h3_index        BIGINT NOT NULL,
    product_type_sk         INT NOT NULL,
    promo_sk                INT NOT NULL,
    currency_code           CHAR(3) NOT NULL,
    fx_rate_to_usd          DECIMAL(18,8) NOT NULL,
    -- Additive Measures
    trip_distance_meters    INT NOT NULL,
    trip_duration_seconds   INT NOT NULL,
    base_fare_usd           DECIMAL(10,2) NOT NULL,
    surge_fare_usd          DECIMAL(10,2) NOT NULL,
    cancellation_fee_usd    DECIMAL(10,2) NOT NULL,
    tolls_and_taxes_usd     DECIMAL(10,2) NOT NULL,
    promo_discount_usd      DECIMAL(10,2) NOT NULL,
    rider_tip_usd           DECIMAL(10,2) NOT NULL,
    gross_booking_value_usd DECIMAL(10,2) NOT NULL,
    driver_payout_usd       DECIMAL(10,2) NOT NULL,
    net_platform_revenue_usd DECIMAL(10,2) NOT NULL -- GBV - driver_payout - promo - tolls
)
PARTITION BY (completed_date_local_sk)
CLUSTER BY (origin_h3_index, driver_id);
```

---

## 4. Phase 4: Query Validation (Minutes 26–33)

```sql
-- Query 1: City Rush-Hour Liquidity, Surge Conversion & Net Take Rate by Historical Driver Tier
SELECT
    z.city_name,
    z.zone_category,
    drv.driver_tier                                                 AS tier_at_trip_time,
    COUNT(r.request_id)                                             AS total_requests,
    SUM(CASE WHEN r.terminal_status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_trips,
    ROUND(
        SUM(CASE WHEN r.terminal_status = 'COMPLETED' THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(r.request_id), 0),
    2)                                                              AS conversion_pct,
    APPROX_PERCENTILE(r.actual_pickup_wait_sec, 0.90) / 60.0        AS p90_pickup_wait_mins,
    SUM(COALESCE(t.gross_booking_value_usd, 0))                     AS total_gbv_usd,
    ROUND(
        SUM(COALESCE(t.net_platform_revenue_usd, 0)) * 100.0 /
        NULLIF(SUM(COALESCE(t.gross_booking_value_usd, 0)), 0),
    2)                                                              AS net_take_rate_pct
FROM gold.fct_trip_requests r
JOIN gold.dim_date d ON r.requested_date_local_sk = d.date_sk
JOIN gold.dim_h3_zone z ON r.origin_h3_index = z.h3_res8_index
LEFT JOIN gold.fct_trips t ON r.request_id = t.request_id
LEFT JOIN gold.dim_driver drv ON t.driver_sk = drv.driver_sk
WHERE d.calendar_date BETWEEN DATE '2026-11-08' AND DATE '2026-11-14'
  AND r.surge_multiplier > 1.25
GROUP BY 1, 2, 3
ORDER BY 8 DESC;
```

---

## 5. Phase 5: Principal Curveballs & "Top-Up" Questions (Minutes 33–45)

1. **Curveball 1 (Cross-Midnight & Late-Arriving Trips):** *"A rider requests a trip at 11:52 PM local time on Nov 14, finishes at 12:25 AM on Nov 15, and the driver goes through a tunnel so the trip completion event arrives at 2:10 AM. How does your model handle this?"*
   * **Principal Answer:** `fct_trip_requests` stays anchored on `requested_date_local_sk = 20261114` (cohort funnel view), while `fct_trips` records `completed_date_local_sk = 20261115` (financial settlement view). Because our 15-minute incremental `MERGE` job scans a trailing 24-hour partition window (`requested_date_local_sk >= CURRENT_DATE - 1` and `is_terminal_state = FALSE`), the 2:10 AM tunnel sync seamlessly updates the Nov 14 request row and inserts the Nov 15 financial trip row with zero duplicates.
2. **Curveball 2 (Shared/Pooled Rides Cost Allocation):** *"In a pooled trip (`pooled_match_group_id`), 3 riders share a single driver route. Rider A pays $12, Rider B pays $10, Rider C pays $8 ($30 total GBV), and the driver is paid $21 for the combined route. How do you compute per-trip unit economics without double-counting driver payout?"*
   * **Principal Answer:** Store the unallocated driver route metrics (`actual_route_miles`, `total_driver_payout_usd = $21`) at the route grain in `fct_pooled_trip_groups` (grain: 1 row per `pooled_match_group_id`), and allocate `driver_payout_usd` deterministically down to each rider row in `fct_trips` proportional to standalone trip distance or fare (`$12/$30 * $21 = $8.40`, `$7.00`, `$5.60`) so `SUM(driver_payout_usd)` in `fct_trips` equals exactly `$21.00`.
3. **Curveball 3 (GDPR Erasure on 50 Billion Rows):** *"A rider in Berlin invokes GDPR Article 17 Right to Erasure. How do you purge their PII without rewriting 5 years of Parquet partitions?"*
   * **Principal Answer:** All rider PII (`name`, `email`, `phone`, `home_address`) is isolated in `dim_rider_pii` encrypted with a per-rider DEK in KMS, while `fct_trips` and `fct_trip_requests` store only a non-reversible `rider_sk` and coarse `H3_res8` hex cells (not exact house GPS coordinates). Destroying the rider's DEK and deleting their single row in `dim_rider_pii` permanently de-identifies 5 years of historical trips in sub-second time without rewriting a single Parquet file.