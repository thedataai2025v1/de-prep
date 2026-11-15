# Module 3: High-Level Model Diagrams
## Lesson 3.8 — Practice: IoT Sensor Data

> *"The hardest models aren't the ones with the most tables — they're the ones where every row is a heartbeat."*

---

### 1. Interview Prompt

> **"Design a data warehouse for an IoT platform that collects sensor data from smart factories. The platform monitors equipment health, predicts failures, optimizes production, and tracks energy usage."**

This is an open-ended design prompt. Interviewers are testing your ability to:
- **Translate business goals into data requirements**
- **Handle extreme scale** (billions of rows/day)
- **Choose the right grain strategy** for multi-speed analytics
- **Anticipate real-world complexities** (late data, gaps, normalization)

---

### 2. Requirements Gathering

#### Business Goals & Key Metrics

| Goal | Key Metrics | Consumers |
|------|-------------|-----------|
| Monitor equipment health | Sensor readings/sec, anomaly count | Engineers, floor managers |
| Predict failures | Downtime hours, maintenance frequency | Reliability teams |
| Optimize production | Production throughput, batch yield | Operations managers |
| Track energy usage | kWh per machine, peak draw | Sustainability team |

#### Volume & Performance

| Parameter | Value | Implication |
|-----------|-------|-------------|
| Sensors | 50,000 | Wide dimension table |
| Machines | 500 | Hierarchical dim_sensor → dim_machine |
| Readings/day | ~1 billion | Raw fact needs partitioning + time-based retention |
| Latency: alerts | < 5 seconds | Real-time stream processing (not DW) |
| Latency: dashboards | hourly | Materialized aggregates |
| Latency: analytics | daily | Standard ELT pipeline |
| Raw retention | 90 days | ~36B rows — prune or archive |
| Hourly aggregates | 2 years | ~50K × 24 × 730 ≈ 876M rows |
| Daily aggregates | 5 years | ~50K × 365 × 5 ≈ 91M rows |

---

### 3. High-Level Model Diagram (ASCII)

```
┌─────────────────────────────────────────────────────────────────┐
│                    FACT TABLES (Star Schema)                    │
│                                                                 │
│  ┌──────────────────────────┐   ┌──────────────────────────┐    │
│  │  fact_sensor_readings    │   │  fact_hourly_aggregates  │    │
│  │  grain: 1 reading        │   │  grain: 1 sensor/hour    │    │
│  │  ~1B rows/day            │   │  derived from raw        │    │
│  │  retained: 90 days       │   │  retained: 2 years       │    │
│  └────────┬─────────────────┘   └────────┬─────────────────┘    │
│           │                              │                       │
│  ┌────────┴─────────────────┐   ┌────────┴──────────────────┐   │
│  │  fact_machine_downtime   │   │  fact_production_batches  │   │
│  │  grain: 1 downtime event │   │  grain: 1 production batch│   │
│  └──────────────────────────┘   └──────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       DIMENSION TABLES                          │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐       │
│  │  dim_date    │   │  dim_sensor  │   │  dim_machine │       │
│  │  (shared)    │   │  1:1 reading │   │  1:50,000    │       │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘       │
│         │                  │                   │                │
│         │           ┌──────┴───────┐  ┌───────┴───────┐       │
│         │           │ dim_sensor_  │  │  dim_location  │       │
│         │           │    type      │  │  (factory,    │       │
│         │           │  (temp,      │  │   zone, room)  │       │
│         │           │   vibration) │  └────────────────┘       │
│         │           └──────────────┘                           │
│         │                                                      │
│         │           ┌──────────────────┐                       │
│         │           │ dim_maintenance_ │                       │
│         │           │     events       │                       │
│         │           └──────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘

RELATIONSHIPS (→ means fact references dimension):
  fact_sensor_readings  → dim_date, dim_sensor, dim_sensor_type, dim_location, dim_machine
  fact_hourly_aggregates→ dim_date, dim_sensor, dim_sensor_type, dim_location, dim_machine
  fact_machine_downtime → dim_date, dim_machine, dim_location, dim_maintenance_events
  fact_production_batches → dim_date, dim_machine, dim_location

```

---

### 4. Detailed Table Definitions

#### 4.1 `fact_sensor_readings` (Raw)

```sql
-- Grain: One row per sensor reading
-- Volume: ~1B rows/day | Retention: 90 days
CREATE TABLE fact_sensor_readings (
    reading_id        BIGINT       NOT NULL,   -- surrogate key
    sensor_key        INT          NOT NULL,   -- FK → dim_sensor
    reading_date_key  INT          NOT NULL,   -- FK → dim_date (YYYYMMDD)
    reading_timestamp TIMESTAMP    NOT NULL,   -- event time (for gaps, late data)
    reading_value     DOUBLE       NOT NULL,   -- normalized to SI units
    unit_of_measure   VARCHAR(10)  NOT NULL,   -- °C, kPa, Hz, V, etc.
    quality_flag      TINYINT      DEFAULT 0,  -- 0=good, 1=flagged, 2=calculated
    anomaly_score     DOUBLE,                    -- null unless anomaly detected
    PARTITION BY      reading_date_key           -- prune old partitions
);
```

#### 4.2 `fact_hourly_aggregates`

```sql
-- Grain: One row per sensor per hour
-- Derived via: SELECT sensor_key, DATE_TRUNC('hour', ts),
--   AVG(value), MAX(value), MIN(value), COUNT(*)
-- Retention: 2 years
CREATE TABLE fact_hourly_aggregates (
    agg_id            BIGINT       NOT NULL,
    sensor_key        INT          NOT NULL,
    date_key          INT          NOT NULL,
    hour_number       TINYINT      NOT NULL,   -- 0-23
    avg_value         DOUBLE,
    max_value         DOUBLE,
    min_value         DOUBLE,
    std_dev           DOUBLE,
    reading_count     INT,                    -- expected: ~3600 for 1/sec sensors
    gap_seconds       INT          DEFAULT 0,  -- missing seconds in window
    quality_pct       DOUBLE       DEFAULT 100
);
```

#### 4.3 `fact_machine_downtime`

```sql
-- Grain: One row per downtime event
CREATE TABLE fact_machine_downtime (
    downtime_id       BIGINT       NOT NULL,
    machine_key       INT          NOT NULL,
    start_date_key    INT          NOT NULL,
    end_date_key      INT          NOT NULL,
    start_timestamp   TIMESTAMP,
    end_timestamp     TIMESTAMP,
    duration_minutes  INT          NOT NULL,
    downtime_code_key INT,               -- FK → dim_downtime_codes
    root_cause        VARCHAR(255),
    maintenance_key   INT,               -- FK → dim_maintenance_events (nullable)
    planned_flag      BOOLEAN      DEFAULT FALSE
);
```

#### 4.4 `fact_production_batches`

```sql
-- Grain: One row per production batch
CREATE TABLE fact_production_batches (
    batch_id          BIGINT       NOT NULL,
    machine_key       INT          NOT NULL,
    date_key          INT          NOT NULL,
    batch_start_ts    TIMESTAMP,
    batch_end_ts      TIMESTAMP,
    product_sku       VARCHAR(50),
    quantity_produced INT,
    quantity_defective INT,
    yield_pct         DOUBLE,
    energy_kwh        DOUBLE
);
```

#### 4.5 Key Dimension Tables

| Dimension | Key Columns | Notes |
|-----------|-------------|-------|
| `dim_date` | date_key, date, day_of_week, month, quarter, year | Shared across all facts |
| `dim_sensor` | sensor_key, sensor_id, sensor_name, machine_key, location_key, type_key, install_date, status | Links sensor → machine → location hierarchy |
| `dim_machine` | machine_key, machine_id, machine_name, location_key, model, install_date, status | SCDC Type 2 if machines change location |
| `dim_location` | location_key, factory_id, zone, room, coordinates | Hierarchical (factory → zone → room) |
| `dim_sensor_type` | type_key, type_name, measurement_unit, valid_range_min, valid_range_max, sample_rate_hz | Reference table — rarely changes |
| `dim_maintenance_events` | event_key, event_id, date_key, machine_key, event_type (preventive/corrective), duration, cost | SCDSC Type 2 for history |

---

### 5. Multi-Grain Aggregation Strategy

```
   ┌──────────────────────────────┐
   │    Raw (1 reading/row)       │  ← 1B rows/day, 90-day retention
   │    Streaming ingestion       │     Partitioned by date
   └──────────┬───────────────────┘
              │  Hourly roll-up (every hour, streaming + batch)
              ▼
   ┌──────────────────────────────┐
   │  Hourly Aggregates            │  ← ~876M rows/2yr → Dashboard queries
   │  (AVG, MAX, MIN, COUNT)      │     Materialized view or separate table
   └──────────┬───────────────────┘
              │  Daily roll-up (nightly batch job)
              ▼
   ┌──────────────────────────────┐
   │  Daily Aggregates             │  ← ~91M rows/5yr → Analytics, BI
   │  (Same measures, daily grain) │     Small enough for interactive tools
   └──────────────────────────────┘
```

**Why this works:** Dashboards read from hourly aggregates (< 1 sec). Ad-hoc analytics reads daily aggregates (fast). Raw data is available for reprocessing or ML feature engineering — but only for 90 days.

---

### 6. Discussion Points

#### 6.1 Why a Single Massive Fact Table Doesn't Work

| Problem | Consequence | Solution |
|---------|-------------|----------|
| 36B+ rows in one table | Full scans are impossible | Partition by date + prune old partitions |
| Different queries need different grains | One grain can't serve dashboards AND deep analytics | Materialized aggregates at hourly/daily |
| Read/write contention | Analytics queries slow down ingestion | Separate tables; write-only raw, read-only aggregates |
| Cost of storage | Storing 36B rows long-term is expensive | Time-based retention with tiered storage |

#### 6.2 Handling Late-Arriving Data

Sensors may send data 5–30 minutes late due to network issues or batching.

```
STRATEGY:
  1. Raw fact: Accept data with reading_timestamp != ingestion_timestamp.
     Window: accept if (ingestion_time - reading_time) < 2 hours.

  2. Hourly aggregates: Rebuild the completed hour when late data arrives.
     Use a "finalized" flag: hour is FINAL until a late reading invalidates it.

  3. Daily aggregates: Rebuild on a schedule (e.g., rebuild previous day at 06:00).

  4. Dim tables: Use SCD Type 2 to track sensor status changes over time.
```

#### 6.3 Time-Series Specific Challenges

| Challenge | Impact | Solution |
|-----------|--------|----------|
| Gaps in readings | Missing averages, false anomalies | `gap_seconds` column; flag hours with < 95% coverage |
| Varying frequencies | Some sensors send 1/sec, others 1/min | Normalize to a common time bucket; use `reading_count` to validate |
| Sensor calibration drift | Values shift over time | `quality_flag`; periodic recalibration events in dim_maintenance |
| Timezone handling | Global factories in different zones | Store all timestamps in UTC; `dim_date` handles conversion |
| Anomaly detection | Need real-time scoring | Compute `anomaly_score` on ingestion; raw table stores it |

---

### 7. Interviewer Questions You'll Face

1. **"Walk me through your model. Why did you choose these fact tables and not one big one?"**
   - *Expected:* Explain multi-grain strategy, partitioning, retention tiers.

2. **"How would you handle a sensor that sends data every 10 seconds vs. one that sends every second?"**
   - *Expected:* `dim_sensor_type` stores sample_rate_hz; aggregates use COUNT to detect gaps.

3. **"A factory manager complains their dashboard shows yesterday's data. How do you fix this?"**
   - *Expected:* Discuss pipeline latency, materialized view refresh schedules, streaming vs. batch trade-offs.

4. **"How would you model a sensor that measures multiple metrics (e.g., temperature AND humidity)?"**
   - *Expected:* Two options — one row per reading per metric (add metric_type column) or separate fact tables.

---

### 8. Top-Up Questions (Senior-Level Probes)

1. **"We need to compare sensor readings across factories. How do you handle different calibration standards?"**
   - *Think:* Normalization in `reading_value`; store raw_value + calibrated_value; dim_sensor_type has conversion formulas.

2. **"Your ML team wants to train a failure-prediction model. What features would you pre-compute in the warehouse?"**
   - *Think:* Rolling averages (1h, 6h, 24h), rate-of-change, stddev windows, gap frequency, maintenance recency — stored in a feature store or denormalized wide table.

3. **"Regulators require 7 years of energy data. How do you reconcile this with our 5-year retention policy?"**
   - *Think:* Tiered storage (hot → warm → cold/archive); daily aggregates are small enough for 7 years; raw data archived to object store with metadata in DW.

4. **"Two sensors fail simultaneously on the same machine. How do you correlate downtime events?"**
   - *Think:* `fact_machine_downtime` is at machine grain (not sensor); multiple sensor anomalies → one downtime event via temporal join.

---

### 9. Practice Exercise: Vehicle Telematics Fleet Tracking

> **Scenario:** Extend this IoT model to also track a fleet of 10,000 commercial vehicles. Each vehicle sends GPS coordinates, engine RPM, fuel consumption, speed, and brake temperature every 5 seconds. Business needs: route optimization, fuel efficiency tracking, driver behavior scoring, and maintenance scheduling.

**Your task — add these to the existing model:**

#### A. New Fact Tables (design grain and key columns)

1. `fact_vehicle_telematics` (raw)
2. `fact_vehicle_trips` (one row per trip — a trip starts when the vehicle moves > 50m for > 60 seconds)
3. `fact_driver_behavior` (aggregated scoring per driver per day)

#### B. New/Extended Dimension Tables

1. `dim_vehicle` (vehicle_id, make, model, fuel_type, assigned_driver_key, fleet_key)
2. `dim_driver` (driver_key, name, license_expiry, training_records, safety_score)
3. `dim_fleet` (fleet_key, fleet_name, region)
4. `dim_route` (route_key, origin, destination, expected_duration, expected_distance)

#### C. Think About

- How does `dim_sensor` on a vehicle differ from a factory sensor?
- Do you need a separate fact for GPS trajectories vs. engine metrics?
- How do you define a "trip" in the data model? (hint: state machine)
- What's the aggregation strategy? (per trip? per driver/day? per fleet/month?)

#### D. Draw the Extended Model

Sketch how the new tables connect to the existing factory model. Hint: a shared `dim_location` and `dim_date` serve both domains.

---

> **💡 Pro Tip for the Interview:** Always end your model design with a **data flow diagram** showing ingestion → raw → aggregates → consumers. It shows you think about the entire lifecycle, not just schemas.

---

*Next: Lesson 3.9 — Review & Common Mistakes in IoT Modeling*

---

### Related Lessons

- [Creating High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md)
- [Practice: Financial Transactions](./19-practice-financial-transactions.md)
- [Slowly Changing Dimensions: Types 1-4](./21-slowly-changing-dimensions-types-1-4.md)
- [Data Volume and Scalability Considerations](./10-data-volume-and-scalability-considerations.md)
