# Module 3: High-Level Model Diagrams — Lesson 3.6: Practice: Ride-Sharing Platform

> **Goal:** Walk through a complete data warehouse design for a ride-sharing platform. This mirrors exactly what a senior data engineering interviewer expects you to produce on a whiteboard or shared doc.

---

## 1. Interview Prompt

> *"Design a data warehouse for a ride-sharing platform like Uber. You need to support ride analytics, driver supply optimization, dynamic pricing analysis, and safety monitoring. Walk me through requirements, the model, and the key design decisions."*

### Interviewer Questions You'll Face
| # | Question | What They're Probing |
|---|----------|---------------------|
| 1 | "Walk me through your fact tables. What's the grain of each?" | Fact-grain discipline — can you avoid mixing grains in one table? |
| 2 | "How would you track the full ride lifecycle from search to completion?" | Event modeling vs. state-transition modeling. |
| 3 | "Driver status changes over time — how do you handle that in dimensions?" | SCD knowledge (Type 2 preferred for auditability). |
| 4 | "What's different about analyzing surge pricing vs. flat-rate rides?" | Time-varying dimensions and bridge tables. |

### Top-Up Questions (Senior-Level Probes)
| # | Question |
|---|----------|
| 1 | "How do you partition your fact tables to keep daily rollups performant at 20M rides/month?" |
| 2 | "If a rider requests multiple searches in one sitting before accepting a match, how is that modeled?" |
| 3 | "How would you join ride data with geospatial zones without losing query performance?" |
| 4 | "Where would you put driver earnings — as a derived metric or a separate fact?" |

---

## 2. Requirements Gathering

### 2.1 Business Goals
| Goal | Description | Primary Consumer |
|------|-------------|-----------------|
| Track ride metrics | Completions, cancellations, duration, distance | Product & Ops |
| Optimize driver supply | Match driver availability to demand hotspots | Supply Operations |
| Analyze pricing | Base fare, surge multipliers, discounts | Pricing Team |
| Monitor safety | Driver incidents, rider complaints, trip anomalies | Trust & Safety |

### 2.2 Key Metrics (with formulas)
| Metric | Formula | Refresh |
|--------|---------|---------|
| Rides Completed | `COUNT(ride_id WHERE status = 'completed')` | Near-real-time |
| Acceptance Rate | `matches_accepted / matches_sent` | Near-real-time |
| ETA Accuracy | `ABS(eta_seconds - actual_seconds) / eta_seconds` | Near-real-time |
| Driver Earnings | `SUM(fare_revenue) - platform_fee - incentives` | Daily |
| Cancellation Rate | `cancellations / rides_started` | Near-real-time |
| Revenue | `SUM(gross_fare) - refunds` | Daily |

### 2.3 Scale & Latency Profile
| Parameter | Value | Implication |
|-----------|-------|-------------|
| Volume | **20M rides/month** (~670K/day) | Partition fact_rides by date |
| Drivers | **5M** active drivers | dim_driver fits in memory; use SCD2 |
| Riders | **100M** registered riders | dim_rider large; archive inactive |
| Latency (dispatch) | < 5 seconds | Separate operational data store (ODS) |
| Latency (analytics) | Daily batch (T+1) | Main warehouse load |
| Retention | 2 years detailed, 3 years compliance | Archive to cold storage at 24 months |

---

## 3. High-Level Model Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        RIDE-SHARING DATA WAREHOUSE                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│   FACT TABLES (additive measures)                                       │
│                                                                        │
│  ┌─────────────────┐     ┌────────────────────┐   ┌──────────────┐    │
│  │  fact_rides     │     │ fact_driver_activity│   │ fact_fares  │    │
│  │ grain: 1 row    │     │ grain: 1 row per    │   │ grain: 1 row │    │
│  │   per ride      │     │   activity event    │   │   per ride   │    │
│  │                 │     │   per driver        │   │              │    │
│  │  ride_id (PK)   │     │  activity_id (PK)  │   │  ride_id (FK)│    │
│  │  rider_sk (FK)  │     │  driver_sk (FK)    │   │  base_fare   │    │
│  │  driver_sk (FK) │     │  activity_type     │   │  surge_mult  │    │
│  │  date_sk (FK)   │     │  date_sk (FK)      │   │  tip_amount  │    │
│  │  pickup_loc_sk  │     │  duration_seconds  │   │  platform_fee│    │
│  │  dropoff_loc_sk │     │  earnings_earned   │   │  total_fare  │    │
│  │  status_cd      │     │  location_sk (FK)  │   │  driver_pay  │    │
│  │  duration_sec   │     │                    │   │              │    │
│  │  distance_km    │     │                    │   │              │    │
│  └───────┬─────────┘     └────────┬───────────┘   └───────┬────────┘   │
│          │                        │                        │            │
├──────────┼────────────────────────┼────────────────────────┼────────────┤
│          │                        │                        │            │
│   DIMENSIONS (conformed)          │                        │            │
│                                   │                        │            │
│  ┌──────────┐ ┌──────────┐ ┌─────┴────────┐ ┌──────────┐ │ ┌────────┐│
│  │dim_date  │ │dim_rider │ │ dim_driver   │ │dim_vehicle│ │ │dim_srv ││
│  │(cal)     │ │(pers)    │ │(SCD2: status,│ │(specs)    │ │ │type    ││
│  └────┬─────┘ └────┬─────┘ │  car, status)│ └────┬─────┘ │ └───┬────┘│
│       │            │       └──────┬───────┘       │        │     │    │
│       │            │              │                │        │     │    │
│       │            │       ┌──────┴───────┐  ┌────┴─────┐  │     │    │
│       │            │       │dim_surge_     │  │dim_loc   │  │     │    │
│       │            │       │(time-varying) │  │(zones)  │  │     │    │
│       │            │       └───────────────┘  └─────────┘  │     │    │
│       │            │                                       │     │    │
│       └────────────┴───────────────────────────────────────┴─────┘    │
│                                                                        │
│   Relationships: fact_rides → dim_driver (FK), fact_fares → dim_driver │
│   dim_driver uses SCD2 with effective_date/end_date/is_current         │
│   dim_surge is a time-varying dimension keyed to (location_sk, date_sk)│
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Detailed Table Definitions

### 4.1 Fact: `fact_rides`

```
-- Grain: ONE ROW PER RIDE REQUEST (from search to final state)
-- Partitioned BY: ride_date_sk  |  Clustering: status_cd, driver_sk

ride_id           BIGINT        PK          Unique ride transaction ID
ride_date_sk      INT           FK → dim_date   Date ride was requested
rider_sk          BIGINT        FK → dim_rider
driver_sk         BIGINT        FK → dim_driver  NULL if search never matched
pickup_loc_sk     INT           FK → dim_location
dropoff_loc_sk    INT           FK → dim_location  NULL if cancelled before start
service_type_sk   INT           FK → dim_service_type
vehicle_sk        INT           FK → dim_vehicle  NULL if not assigned

status_cd         VARCHAR(20)   'search','matched','accepted','in_progress',
                                'completed','cancelled_rider','cancelled_driver'
duration_seconds  INT           Actual trip duration (NULL if cancelled)
distance_km       DECIMAL(8,2)  Route distance

search_timestamp  TIMESTAMP
match_timestamp   TIMESTAMP     When driver was matched
accept_timestamp  TIMESTAMP     When driver accepted
start_timestamp   TIMESTAMP     When trip began
complete_timestamp TIMESTAMP    When trip ended

eta_seconds       INT           Estimated time at search time
surge_multiplier  DECIMAL(4,2)  Captured at search time (also see dim_surge)
```

### 4.2 Fact: `fact_driver_activity`

```
-- Grain: ONE ROW PER DRIVER ACTIVITY EVENT (login, logout, online, offline, trip)
-- Partitioned BY: activity_date_sk

activity_id       BIGINT        PK
activity_date_sk  INT           FK → dim_date
driver_sk         BIGINT        FK → dim_driver
location_sk       INT           FK → dim_location  Where activity occurred
activity_type     VARCHAR(20)   'login','logout','online','offline','trip_start',
                                'trip_complete','idle'
duration_seconds  INT           Duration of the activity period
earnings_earned   DECIMAL(10,2) Earnings accrued during this period
```

### 4.3 Fact: `fact_fares`

```
-- Grain: ONE ROW PER COMPLETED RIDE (financial breakdown)
-- Partitioned BY: fare_date_sk

fare_id           BIGINT        PK
ride_id           BIGINT        FK → fact_rides
fare_date_sk      INT           FK → dim_date
service_type_sk   INT           FK → dim_service_type
surge_sk          INT           FK → dim_surge   Time-varying pricing dimension

base_fare         DECIMAL(10,2)
distance_charge   DECIMAL(10,2)
time_charge       DECIMAL(10,2)
surge_upcharge    DECIMAL(10,2)
tip_amount        DECIMAL(10,2)
platform_fee      DECIMAL(10,2)
driver_pay        DECIMAL(10,2)
total_fare        DECIMAL(10,2)
currency_cd       CHAR(3)
```

### 4.4 Dimension: `dim_driver` (SCD Type 2)

```
-- SCD2: Track every status, vehicle, and tier change
-- Query current state: WHERE is_current = 1
-- Query historical state: WHERE effective_date <= as_of_date AND end_date > as_of_date

driver_sk         BIGINT        PK, Surrogate Key
driver_id         VARCHAR(32)   Business key
effective_date    TIMESTAMP
end_date          TIMESTAMP     NULL = current record
is_current        BOOLEAN       1 = current
status_cd         VARCHAR(20)   'active','suspended','deactivated','pending'
tier_cd           VARCHAR(10)   'gold','platinum','standard','new'
vehicle_sk        INT           FK → dim_vehicle  Assigned vehicle at this time
rating_score      DECIMAL(3,2)  Driver rating at this time
date_joined       DATE
home_city_sk      INT           FK → dim_location
```

### 4.5 Other Dimensions (Key Columns Only)

| Dimension | Key Columns | Notes |
|-----------|------------|-------|
| `dim_date` | `date_sk`, `date`, `day_of_week`, `month`, `year`, `is_weekend`, `is_holiday` | Calendar table, 10 years pre-loaded |
| `dim_rider` | `rider_sk`, `rider_id`, `signup_date`, `account_status`, `total_rides`, `avg_rating` | Slowly changing (Type 1 for status) |
| `dim_location` | `location_sk`, `city`, `state`, `country`, `latitude`, `longitude`, `zone_id`, `zone_type` | `zone_type`: 'airport','downtown','suburb','highway' |
| `dim_vehicle` | `vehicle_sk`, `make`, `model`, `year`, `color`, `seats`, `vehicle_class` | Type 1 — rarely changes |
| `dim_service_type` | `service_type_sk`, `name`, `description`, `base_rate` | 'UberX','UberXL','Uber Black','Uber Pool' |
| `dim_surge` | `surge_sk`, `location_sk`, `date_sk`, `hour_sk`, `surge_multiplier`, `demand_index` | **Time-varying** — one row per zone per time slot |

---

## 5. Discussion Points

### 5.1 Tracking Ride Lifecycle (State-Transition Modeling)

A ride moves through **six states**. We capture each transition with a timestamp:

```
  search ──► matched ──► accepted ──► in_progress ──► completed
   │           │            │              │
   └──────────► cancelled_rider ───────────┘
               │
               └──────────► cancelled_driver
```

**Design decision:** Store all transition timestamps in `fact_rides` as nullable columns (denormalized for performance). This avoids a separate event-sourcing fact table for the analytics layer. For full audit, the **operational data store (ODS)** keeps every event row.

> **Interview talking point:** *"I denormalize the lifecycle states into wide fact rows for the analytics warehouse because BI queries ask 'how long did rides spend in each state?' — and having all timestamps on one row lets us compute that with simple CASE/COALESCE logic instead of self-joins."*

### 5.2 Surge Pricing as a Time-Varying Dimension

Surge pricing changes **per location, per time slot**. This makes it a **time-varying dimension** (not a classic slowly changing dimension):

```
dim_surge (time-varying, refreshed every 5 minutes)
┌────────┬───────────────┬────────────┬──────────────────┬──────────────┐
│ surge_sk│ location_sk   │ date_sk    │ hour_sk          │ surge_mult   │
├────────┼───────────────┼────────────┼──────────────────┼──────────────┤
│  1001  │  5001         │ 2024-01-15 │ 08:00-08:05      │ 1.00         │
│  1002  │  5001         │ 2024-01-15 │ 08:05-08:10      │ 1.45         │
│  1003  │  5001         │ 2024-01-15 │ 08:10-08:15      │ 1.80         │
└────────┴───────────────┴────────────┴──────────────────┴──────────────┘
```

- `fact_fares` joins to `dim_surge` via `surge_sk` to capture the **exact multiplier** that applied.
- We also store `surge_multiplier` directly in `fact_rides` (snapshotted at search time) to avoid join issues when the surge dimension is retroactively corrected.

> **Interview talking point:** *"I dual-store surge in both the fact (frozen snapshot) and the dimension (authoritative time series). This lets analysts compare actual vs. historical pricing policies."*

### 5.3 Geospatial Analysis

Ride analysis frequently involves **pickup and dropoff zones** (not just point coordinates). `dim_location` supports both:

| Analysis | Join Strategy |
|----------|---------------|
| "Rides from airport zones" | `fact_rides.pickup_loc_sk = dim_location.location_sk WHERE zone_type = 'airport'` |
| "Supply-demand heatmaps" | Pre-compute zone aggregates into a **fact_zone_activity** summary table |
| "Cross-zone trips (city to suburb)" | Join `fact_rides` to `dim_location` **twice** — once via `pickup_loc_sk`, once via `dropoff_loc_sk` |

> **Interview talking point:** *"For heavy geospatial queries, I'd create a zone-level summary fact table pre-aggregated by (date, pickup_zone, dropoff_zone) so analysts don't scan 670K daily ride rows for every geographic query."*

---

## 6. Interview Recap: What to Say Out Loud

When you present this model, use this structure:

1. **"I start with the business questions."** — Name the metrics and who needs them.
2. **"I define my facts by grain."** — One fact table per measurement context.
3. **"I normalize shared attributes into conformed dimensions."** — `dim_driver`, `dim_location`.
4. **"I handle time-varying attributes with the right SCD strategy."** — SCD2 for driver status; time-varying dim for surge.
5. **"I design for partition and query patterns."** — Partition by date; duplicate snapshot columns to avoid fragile joins.

---

## 7. Practice Exercise: Adding Food Delivery to the Same Platform

The platform now wants to **add food delivery** (like Uber Eats) using the same data warehouse. You need to extend the model without breaking existing ride-sharing queries.

### Task

Design the changes. Address:

| # | Question |
|---|----------|
| 1 | **Should `fact_rides` be split, or should you use a conformed dimension to distinguish rides from deliveries?** Explain your choice. |
| 2 | **What new fact tables** are needed? (Consider: order placement, restaurant acceptance, driver pickup, delivery completion, refund.) |
| 3 | **What new dimensions** are needed? (Consider: `dim_restaurant`, `dim_menu_item`, `dim_order_type`.) |
| 4 | **How do you reuse** `dim_driver`, `dim_location`, and `dim_date` without duplication? |
| 5 | **How would you handle** the case where the same driver does both rides and deliveries in one day? |

### Suggested Approach (Sketch Your Answer First)

Before looking at a model answer, sketch:
1. A new `dim_order_type` with values `'ride'` and `'delivery'` — does this help?
2. Whether `fact_rides` becomes `fact_transitions` (generic) — what's lost?
3. A `fact_orders` table at grain "one row per order" that captures both rides and deliveries.

### Model Answer Summary

| Decision | Rationale |
|----------|-----------|
| **Keep `fact_rides` for rides only.** Add `fact_deliveries` for food orders. | Different grain semantics: a ride has exactly one passenger; a delivery has restaurant → driver → customer, and may involve multiple menu items. Mixing them forces many NULLs. |
| **Add `dim_order_type`** as a bridge dimension. | Lets cross-analysis queries (e.g., "driver earnings by order type") work via a unified conformed driver key. |
| **Add `dim_restaurant`** (name, location, rating, cuisine, active flag). | New entity — not a ride concept. |
| **Add `dim_menu_item`** (item_id, restaurant_sk, name, category, base_price). | Supports item-level revenue breakdown. |
| **Reuse `dim_driver`, `dim_date`, `dim_location`** with no changes. | These are already conformed dimensions — deliveries join the same surrogate keys. |
| **A driver doing both** is handled naturally. | `fact_driver_activity` already aggregates across activity types; add `'delivery_pickup'` and `'delivery_complete'` to `activity_type`. |

---

> **Next:** Lesson 3.7 — Review & Common Mistakes in Model Diagrams

---

### Related Lessons

- [Creating High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md)
- [Practice: Video Streaming Service](./17-practice-video-streaming-service.md)
- [Practice: Financial Transactions](./19-practice-financial-transactions.md)
- [Data Quality Rules and Validation Strategies](./12-data-quality-rules-and-validation-strategies.md)
