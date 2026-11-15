# Lesson 18: Practice — Ride-Sharing Platform Data Model

> **Principal Mentor Note:** Ride-sharing (Uber, Lyft, Ola) is a classic **two-sided geospatial marketplace** interview problem. To hit the Principal bar, you must model **both sides of the marketplace**—Demand (`fct_trip_requests`, `fct_trips`) and Supply (`fct_driver_supply_intervals`)—handle **H3 hexagonal spatial indexing** instead of raw lat/lon floats, capture **surge pricing & split payouts**, and support **multi-stop / pooled trips**.

---

## 1. Requirements & Scope Assumptions

* **Scale:** `25M completed trips/day`, `40M trip requests/day` (including unfulfilled/rider-canceled), `6M active drivers`, `150M riders` across `700+ cities`.
* **Core Business Processes:**
  1. **Ride Quote & Request Funnel** (`fct_trip_requests` — upfront price quote, surge multiplier, match outcome: `COMPLETED`, `RIDER_CANCELED`, `DRIVER_CANCELED`, `NO_SUPPLY`).
  2. **Completed Trip Execution & Unit Economics** (`fct_trips` — pickup/dropoff timestamps, H3 spatial zones, distance, duration, gross booking fare, surge revenue, tolls, driver payout, net platform take).
  3. **Driver Supply & Utilization** (`fct_driver_online_intervals` — tracks `ONLINE_IDLE`, `EN_ROUTE_TO_PICKUP`, and `ON_TRIP` minutes so Operations can compute true supply utilization).
  4. **Marketplace Balance Periodic Snapshot** (`fct_zone_supply_demand_15min` — powers surge pricing effectiveness and ETA heatmaps).
* **Key Analytical Queries:**
  * Marketplace Liquidity: Request-to-Completion Conversion Rate & `P90` Pickup ETA by City, H3 Zone, and Rush Hour.
  * Unit Economics: Net Take Rate (`(Gross Fare - Driver Payout - Promos) / Gross Fare`) by Product Type (UberX, Black, Pool) and Surge Tier.
  * Driver Utilization: `On-Trip Minutes / Total Online Minutes` by Driver Tenure Cohort and Vehicle Type.

---

## 2. Enterprise Bus Matrix

```
Business Process (Fact Table)              | Date | Time | Rider | Driver | Vehicle | Origin H3 | Dest H3 | Product | Promo
-------------------------------------------+------+------+-------+--------+---------+-----------+---------+---------+------
fct_trip_requests (Accumulating Snapshot)  |  X   |  X   |   X   |   X    |    X    |     X     |    X    |    X    |  X
fct_trips (Transaction Fact)               |  X   |  X   |   X   |   X    |    X    |     X     |    X    |    X    |  X
fct_driver_online_intervals (Transaction)  |  X   |  X   |       |   X    |    X    |     X     |         |         |
fct_zone_supply_demand_15min (Periodic)    |  X   |  X   |       |        |         |     X     |         |    X    |
```

---

## 3. High-Level Constellation Diagram

```
+---------------------+       +----------------------+       +-----------------------+
| dim_rider [SCD2]    |       |  dim_date [CONFORM]  |       | dim_driver [SCD2]     |
| PK: rider_sk        |       |  PK: date_sk         |       | PK: driver_sk         |
| NK: rider_id        |       +----------+-----------+       | NK: driver_id         |
+----------+----------+            ^            ^            | tier, signup_cohort   |
           | 1                     |            |            +-----+-------------+---+
           |                       |            |                  | 1           | 1
           | N             +-------+----+  +----+--------------+   |             |
           +-------------->| fct_trip_  |  | fct_trips [TXN]   |<--+             | N
                           | requests   |  | Grain: 1 row /    |                 v
+---------------------+    | [ACC-SNAP] |  | completed trip_id |    +-----------------------+
| dim_h3_zone [SCD1]  |--->| Grain: 1 / |  +----+--------------+    | fct_driver_online_    |
| PK: h3_res8_index   |1:N | request_id |       ^                   | intervals [TXN]       |
| city, neighborhood, |    +------------+       | 1:N               | Grain: 1 state slice  |
| is_airport_zone     |                +--------+--------+          +-----------------------+
+---------------------+                | dim_vehicle     |
   ^ (Role-Playing:                    | [SCD2]          |
   |  origin_h3_index &                | make, model, ev |
   |  dropoff_h3_index)                +-----------------+
```

---

## 4. Core Production DDL & Geospatial Modeling Choices

```sql
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE dim_h3_zone (
    h3_res8_index           BIGINT PRIMARY KEY,    -- Uber H3 Hexagonal Index (~0.74 km^2 per cell)
    h3_res6_parent          BIGINT NOT NULL,       -- Coarser neighborhood rollup (~36 km^2)
    city_id                 INT NOT NULL,
    city_name               VARCHAR(100) NOT NULL,
    country_code            CHAR(2) NOT NULL,
    timezone_name           VARCHAR(64) NOT NULL,  -- e.g., 'America/New_York', 'Asia/Kolkata'
    zone_category           VARCHAR(32) NOT NULL,  -- AIRPORT, DOWNTOWN_CORE, SUBURB, TRANSIT_HUB
    is_congested_zone       BOOLEAN NOT NULL
);

CREATE TABLE dim_driver (
    driver_sk               BIGINT PRIMARY KEY,
    driver_id               VARCHAR(64) NOT NULL,
    city_id                 INT NOT NULL,
    signup_cohort_month     DATE NOT NULL,
    driver_tier             VARCHAR(24) NOT NULL,  -- BLUE, GOLD, PLATINUM, DIAMOND (SCD2 tracked!)
    is_ev_driver            BOOLEAN NOT NULL,
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

-- ============================================================================
-- FACT 1: TRIP REQUESTS & FUNNEL (Accumulating Snapshot Fact)
-- Grain: One row per ride request (request_id) — captures BOTH completed & canceled!
-- ============================================================================
CREATE TABLE fct_trip_requests (
    request_id              VARCHAR(64) PRIMARY KEY,
    requested_date_local_sk INT NOT NULL,          -- Local city date for operational reporting
    requested_at_utc        TIMESTAMP NOT NULL,
    matched_at_utc          TIMESTAMP,
    driver_arrived_at_utc   TIMESTAMP,
    trip_started_at_utc     TIMESTAMP,
    trip_ended_at_utc       TIMESTAMP,
    canceled_at_utc         TIMESTAMP,
    rider_sk                BIGINT NOT NULL,
    driver_sk               BIGINT,                -- NULL if NO_SUPPLY or canceled before match
    vehicle_sk              BIGINT,
    origin_h3_index         BIGINT NOT NULL,       -- Role-playing FK -> dim_h3_zone
    dropoff_h3_index        BIGINT NOT NULL,       -- Role-playing FK -> dim_h3_zone
    product_type_sk         INT NOT NULL,          -- UBER_X, BLACK, XL, MOTO, AUTO
    promo_sk                INT NOT NULL,
    terminal_status         VARCHAR(32) NOT NULL,  -- COMPLETED, RIDER_CANCELED, DRIVER_CANCELED, NO_SUPPLY
    cancellation_stage      VARCHAR(32),           -- PRE_MATCH, EN_ROUTE_PICKUP, AT_PICKUP
    -- Upfront Quote & Funnel Lag Measures
    upfront_quote_usd       DECIMAL(10,2) NOT NULL,
    surge_multiplier        DECIMAL(5,2) NOT NULL, -- 1.00 = no surge; 2.40 = 2.4x surge
    quoted_eta_seconds      INT NOT NULL,
    actual_pickup_wait_sec  INT,                   -- trip_started_at - requested_at
    dispatch_attempt_count  INT NOT NULL           -- How many drivers declined/timed out before match
)
PARTITION BY (requested_date_local_sk)
CLUSTER BY (origin_h3_index, terminal_status);

-- ============================================================================
-- FACT 2: COMPLETED TRIPS & UNIT ECONOMICS (Transaction Fact)
-- Grain: One row per completed financial trip (trip_id)
-- ============================================================================
CREATE TABLE fct_trips (
    trip_id                 VARCHAR(64) PRIMARY KEY,
    request_id              VARCHAR(64) NOT NULL,  -- Links back to funnel request
    completed_date_local_sk INT NOT NULL,
    completed_at_utc        TIMESTAMP NOT NULL,
    rider_sk                BIGINT NOT NULL,
    driver_sk               BIGINT NOT NULL,       -- Historical SCD2 SK (captures driver tier on trip)
    vehicle_sk              BIGINT NOT NULL,       -- Separate from driver because drivers rent/swap cars!
    origin_h3_index         BIGINT NOT NULL,
    dropoff_h3_index        BIGINT NOT NULL,
    product_type_sk         INT NOT NULL,
    promo_sk                INT NOT NULL,
    currency_code           CHAR(3) NOT NULL,
    fx_rate_to_usd          DECIMAL(18,8) NOT NULL,
    -- Additive Physical & Financial Measures
    trip_distance_meters    INT NOT NULL,
    trip_duration_seconds   INT NOT NULL,
    base_fare_usd           DECIMAL(10,2) NOT NULL,
    surge_fare_usd          DECIMAL(10,2) NOT NULL,
    tolls_and_surcharges_usd DECIMAL(10,2) NOT NULL,
    promo_discount_usd      DECIMAL(10,2) NOT NULL,
    rider_tip_usd           DECIMAL(10,2) NOT NULL,
    gross_booking_value_usd DECIMAL(10,2) NOT NULL,
    driver_payout_usd       DECIMAL(10,2) NOT NULL,
    net_platform_revenue_usd DECIMAL(10,2) NOT NULL -- gross - driver_payout - promo_discount - tolls
)
PARTITION BY (completed_date_local_sk)
CLUSTER BY (origin_h3_index, driver_sk);
```

---

## 5. Validation Query (Marketplace Liquidity & Surge Unit Economics)

```sql
-- Query: Airport vs. Downtown Surge Efficiency & Net Take Rate by Driver Tier
SELECT
    z.city_name,
    z.zone_category,
    drv.driver_tier,
    COUNT(r.request_id)                                              AS total_requests,
    COUNT(t.trip_id)                                                 AS completed_trips,
    ROUND(COUNT(t.trip_id) * 1.0 / NULLIF(COUNT(r.request_id), 0), 4) AS fulfillment_rate,
    ROUND(AVG(r.actual_pickup_wait_sec) / 60.0, 2)                   AS avg_pickup_wait_mins,
    SUM(t.gross_booking_value_usd)                                   AS gbv_usd,
    SUM(t.net_platform_revenue_usd) /
        NULLIF(SUM(t.gross_booking_value_usd), 0)                    AS net_take_rate
FROM fct_trip_requests r
JOIN dim_date d ON r.requested_date_local_sk = d.date_sk
JOIN dim_h3_zone z ON r.origin_h3_index = z.h3_res8_index
LEFT JOIN fct_trips t ON r.request_id = t.request_id
LEFT JOIN dim_driver drv ON t.driver_sk = drv.driver_sk
WHERE d.calendar_date BETWEEN DATE '2026-11-01' AND DATE '2026-11-07'
  AND r.surge_multiplier >= 1.50
GROUP BY 1, 2, 3
ORDER BY 8 DESC;
```

---

## 6. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "Why did you separate `dim_driver` and `dim_vehicle` instead of putting `vehicle_make` and `license_plate` inside `dim_driver`?"**
   * **Expected Insight:** In many markets (fleet rentals, EV leasing partnerships, shared family cars), a single driver drives multiple vehicles within the same week, and multiple drivers share a single fleet vehicle across day/night shifts. Separating `dim_driver` and `dim_vehicle` avoids unnecessary SCD Type 2 churn on `dim_driver` every time a driver swaps vehicles and allows Fleet Ops to track maintenance/utilization per `vehicle_sk`.
2. **Top-Up 2: "How do you model UberPool / Shared Rides where 2 or 3 independent riders share overlapping segments of a single driver's route?"**
   * **Expected Insight:** Separate the **Rider Trip** (`fct_trips`, grain: 1 row per rider booking, storing rider fare & pickup/dropoff H3) from the **Driver Shared Dispatch Match** (`fct_pooled_matches`, grain: 1 row per `match_group_id`, storing actual total road miles driven by the driver, `matched_rider_count`, and pooled driver payout). Link them via `match_group_id` as a degenerate key on `fct_trips`.

---

## 7. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Uses discrete H3 hexagonal grid IDs (`BIGINT`)** joined to `dim_h3_zone` instead of raw `FLOAT` lat/lon coordinates. | Stores only raw `(lat, lon)` floats on fact rows, making spatial `GROUP BY` and clustering impossible. |
| **Models both Demand Funnel (`fct_trip_requests`) and Supply Availability (`fct_driver_online_intervals`)**. | Only models completed trips, leaving no way to measure `NO_SUPPLY` lost demand or idle driver hours. |
| **Uses Local City Date (`requested_date_local_sk`)** for partitioning/reporting alongside UTC timestamps. | Groups rush-hour commute patterns using UTC dates (splitting evening rush in California across two days!). |