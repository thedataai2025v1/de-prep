# Lesson 33: Mock Interview 3 — Design a Data Warehouse Schema for Airbnb

> **Principal Mentor Note:** Airbnb is a premier Principal-level data modeling prompt because it combines **two-sided marketplace dynamics** (Guests & Hosts), **inventory calendar snapshots** (Listing Night Availability: *Available vs. Booked vs. Host-Blocked*), **cross-month GAAP revenue recognition** (a 10-night stay booked in October spanning Nov 27 – Dec 7 cannot recognize all revenue on the booking date!), **cancellations & mid-stay alterations**, and **multi-valued listing amenities**.

---

## 1. Phase 1: Clarifying Questions & Scope Alignment (Minutes 0–6)

### Interviewer Prompt
> *"Design the analytical data warehouse for Airbnb's core Stays marketplace. Leadership wants to track search-to-book conversion, Gross Booking Value (GBV), recognized revenue, listing occupancy rates, Average Daily Rate (ADR), and host quality/Superhost retention."*

### Candidate's 4-Bucket Discovery Questions
1. **Business Processes & Funnel Scope:** *"Should we cover the full guest journey from search impressions/listing views through booking request/instant book, stay check-in/check-out, cancellations/alterations, and post-stay two-sided reviews, as well as daily listing calendar availability?"*
   * *Interviewer:* *"Yes—include search-to-book funnel, bookings/stays, calendar occupancy, cancellations, and guest reviews."*
2. **Revenue Recognition & Multi-Currency:** *"When a guest books in October for a December stay—and pays in EUR for a listing priced in JPY—do we need both **Booked GBV** (on booking date) and **Recognized Revenue / Stay Nights** (allocated per night stayed across month boundaries), plus locked FX rates?"*
   * *Interviewer:* *"Yes! Finance strictly requires night-by-night recognized revenue for monthly close, while Product tracks booked GBV on the reservation date."*
3. **Historical Truth (SCD Type 2):** *"When a host achieves or loses `Superhost` status, or a listing changes its `room_type`, `base_nightly_price`, or `cancellation_policy`, do we need point-in-time historical state at the time of booking (`as-was`)?"*
   * *Interviewer:* *"Yes, historical Superhost status and cancellation policy at booking time are critical."*
4. **Scale, Latency & Compliance:** *"What daily scale, freshness SLA, retention, and GDPR/privacy requirements apply?"*
   * *Interviewer:* *"Assume 150M active users, 8M active listings across 220 countries, 400M search events/day, 2M bookings/day, 1-hour freshness for marketplace dashboards, 7-year financial retention, and strict GDPR PII compliance."*

---

## 2. Phase 2: Enterprise Bus Matrix & Grain Contract (Minutes 6–12)

### Enterprise Bus Matrix
```
Business Process (Fact Table)              | Date | Guest | Host | Listing | Geo/Market | Amenity (Bridge) | Cancel Policy
-------------------------------------------+------+-------+------+---------+------------+------------------+--------------
fct_search_funnel_daily (Periodic/Funnel)  |  X   |   X   |      |    X    |     X      |                  |
fct_bookings (Accumulating Snapshot)       |  X   |   X   |  X   |    X    |     X      |        X         |       X
fct_booking_nights (Transaction/Allocated) |  X   |   X   |  X   |    X    |     X      |                  |
fct_listing_calendar_daily (Periodic Snap) |  X   |       |  X   |    X    |     X      |        X         |       X
fct_stay_reviews (Transaction)             |  X   |   X   |  X   |    X    |     X      |                  |
```

### Explicit Grain Declarations (The Core Architectural Insight)
1. **`fct_bookings` (Accumulating Snapshot):** One row per reservation (`booking_id`), tracking lifecycle milestones (`requested_at`, `confirmed_at`, `checked_in_at`, `checked_out_at`, `canceled_at`), total booking financials (GBV, guest service fee, host fee, cleaning fee, taxes), and cancellation/alteration status.
2. **`fct_booking_nights` (Allocated Transaction Fact):** One row per **`(booking_id, stay_date_sk)`** for every night of a reservation. Solves **cross-month GAAP revenue recognition** (a Nov 28 – Dec 3 stay splits cleanly into 3 Nov nights and 2 Dec nights) and calculates exact **Average Daily Rate (ADR)** per night stayed!
3. **`fct_listing_calendar_daily` (Periodic Snapshot):** One row per **`(listing_sk, calendar_date_sk)`** for active listings within the forward 365-day booking window + historical dates, tracking whether each listing-night is `AVAILABLE`, `BOOKED`, or `HOST_BLOCKED`—enabling exact **Occupancy Rate** (`Booked Nights / (Booked Nights + Available Nights)`).

---

## 3. Phase 3: High-Level Constellation Diagram & DDL (Minutes 12–26)

```
+---------------------+       +----------------------+       +-----------------------+
| dim_user [SCD2]     |       |  dim_date [CONFORM]  |       | dim_listing [SCD2]    |
| PK: user_sk         |       |  PK: date_sk         |       | PK: listing_sk        |
| NK: user_id         |       +----------+-----------+       | NK: listing_id        |
| (Role: Guest & Host)|            ^            ^            | host_sk, room_type,   |
| is_superhost, cohort|            |            |            | bedrooms, instant_book|
+---+-------------+---+            |            |            +-----+-------------+---+
    | 1           | 1              |            |                  | 1           | 1
    |             |        +-------+----+  +----+--------------+   |             |
    | N           | N      | fct_       |  | fct_booking_      |<--+             | N (M:N)
    |             +------->| bookings   |  | nights [TXN-ALLOC]|                 v
    |                      | [ACC-SNAP] |  | Grain: 1 /        |       +-------------------+
    |                      | Grain: 1 / |  | (booking_id,night)|       | bridge_listing_   |
    |                      | booking_id |  +-------------------+       | amenities         |
    |                      +----+-------+                              | amenity_group_sk  |
    |                           ^                                      +-------------------+
    | 1       +-----------------+--------+
    +-------->| fct_listing_calendar_    |   +-----------------------+
              | daily [PERIODIC SNAP]    |   | dim_geo_market [SCD1] |
              | Grain: 1/(listing,date)  |   | h3_res7, city, region,|
              +--------------------------+   | market_tier, country  |
                                             +-----------------------+
```

### Production DDL (Apache Iceberg / Snowflake)

```sql
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE gold.dim_listing (
    listing_sk              BIGINT PRIMARY KEY,
    listing_id              VARCHAR(64) NOT NULL,  -- Durable Natural Key
    host_user_sk            BIGINT NOT NULL,       -- Conformed FK -> dim_user
    geo_market_sk           INT NOT NULL,
    amenity_group_sk        BIGINT NOT NULL,       -- Hash of sorted amenities -> bridge_listing_amenities
    property_type           VARCHAR(32) NOT NULL,  -- APARTMENT, HOUSE, VILLA, CABIN, BOUTIQUE_HOTEL
    room_type               VARCHAR(32) NOT NULL,  -- ENTIRE_HOME, PRIVATE_ROOM, SHARED_ROOM
    bedrooms_count          TINYINT NOT NULL,
    max_guests_capacity     TINYINT NOT NULL,
    is_instant_book_enabled BOOLEAN NOT NULL,
    cancellation_policy     VARCHAR(24) NOT NULL,  -- FLEXIBLE, MODERATE, STRICT (SCD2 tracked!)
    -- Note: Exact street address & lat/lon isolated in dim_listing_pii for host safety/GDPR!
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

CREATE TABLE gold.dim_user (
    user_sk                 BIGINT PRIMARY KEY,
    user_id                 VARCHAR(64) NOT NULL,
    signup_cohort_month     DATE NOT NULL,
    country_code            CHAR(2) NOT NULL,
    is_host                 BOOLEAN NOT NULL,
    is_superhost            BOOLEAN NOT NULL,      -- SCD2 tracked (captures status at booking time!)
    identity_verified_flag  BOOLEAN NOT NULL,
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

-- ============================================================================
-- FACT 1: BOOKINGS LIFECYCLE (Accumulating Snapshot)
-- Grain: One row per reservation (booking_id)
-- ============================================================================
CREATE TABLE gold.fct_bookings (
    booking_id              VARCHAR(64) PRIMARY KEY,
    booked_date_sk          INT NOT NULL,          -- Partition key (date reservation was created)
    check_in_date_sk        INT NOT NULL,          -- Role-playing FK -> dim_date
    check_out_date_sk       INT NOT NULL,          -- Role-playing FK -> dim_date
    canceled_date_sk        INT NOT NULL DEFAULT -1,
    booked_at_utc           TIMESTAMP NOT NULL,
    confirmed_at_utc        TIMESTAMP,
    canceled_at_utc         TIMESTAMP,
    guest_user_sk           BIGINT NOT NULL,
    host_user_sk            BIGINT NOT NULL,       -- Historical SCD2 SK (captures Superhost status!)
    listing_sk              BIGINT NOT NULL,       -- Historical SCD2 SK (captures policy/specs!)
    geo_market_sk           INT NOT NULL,
    booking_channel         VARCHAR(16) NOT NULL,  -- IOS, ANDROID, WEB
    is_instant_booked       BOOLEAN NOT NULL,
    booking_status          VARCHAR(24) NOT NULL,  -- PENDING, CONFIRMED, COMPLETED, CANCELED_GUEST, CANCELED_HOST
    is_terminal_state       BOOLEAN NOT NULL,      -- TRUE once check_out_date passed or canceled
    -- Stay & Guest Counts
    nights_booked           INT NOT NULL,
    adults_count            TINYINT NOT NULL,
    children_count          TINYINT NOT NULL,
    lead_time_days          INT NOT NULL,          -- DATEDIFF(day, booked_date, check_in_date)
    -- Multi-Currency & Total Reservation Financials (Booked View)
    guest_currency_code     CHAR(3) NOT NULL,
    host_currency_code      CHAR(3) NOT NULL,
    fx_rate_to_usd          DECIMAL(18,8) NOT NULL,
    nightly_subtotal_usd    DECIMAL(12,2) NOT NULL,
    cleaning_fee_usd        DECIMAL(10,2) NOT NULL,
    guest_service_fee_usd   DECIMAL(10,2) NOT NULL,
    host_service_fee_usd    DECIMAL(10,2) NOT NULL,
    occupancy_taxes_usd     DECIMAL(10,2) NOT NULL,
    gross_booking_value_usd DECIMAL(12,2) NOT NULL,-- Subtotal + cleaning + guest fee + taxes
    net_platform_rev_usd    DECIMAL(12,2) NOT NULL,-- guest_service_fee + host_service_fee
    refunded_amount_usd     DECIMAL(12,2) NOT NULL DEFAULT 0.00
)
PARTITION BY (booked_date_sk)
CLUSTER BY (geo_market_sk, is_terminal_state);

-- ============================================================================
-- FACT 2: BOOKING NIGHTS (Allocated Transaction Fact for GAAP Revenue & ADR)
-- Grain: One row per (booking_id, stay_date_sk)
-- ============================================================================
CREATE TABLE gold.fct_booking_nights (
    booking_id              VARCHAR(64) NOT NULL,
    stay_date_sk            INT NOT NULL,          -- Exact calendar night stayed (Partition key!)
    booked_date_sk          INT NOT NULL,
    listing_sk              BIGINT NOT NULL,
    host_user_sk            BIGINT NOT NULL,
    guest_user_sk           BIGINT NOT NULL,
    geo_market_sk           INT NOT NULL,
    night_sequence_num      INT NOT NULL,          -- 1, 2, ... N
    is_weekend_night        BOOLEAN NOT NULL,
    is_canceled_night       BOOLEAN NOT NULL,
    -- Allocated Per-Night Measures (Sums cleanly to fct_bookings totals!)
    nightly_room_rate_usd   DECIMAL(10,2) NOT NULL,
    allocated_cleaning_usd  DECIMAL(10,2) NOT NULL,-- cleaning_fee / nights_booked
    allocated_net_rev_usd   DECIMAL(10,2) NOT NULL,-- net_platform_rev / nights_booked
    PRIMARY KEY (booking_id, stay_date_sk)
)
PARTITION BY (stay_date_sk)
CLUSTER BY (geo_market_sk, listing_sk);

-- ============================================================================
-- FACT 3: LISTING CALENDAR DAILY SNAPSHOT (Periodic Snapshot for Occupancy)
-- Grain: One row per (calendar_date_sk, listing_sk)
-- ============================================================================
CREATE TABLE gold.fct_listing_calendar_daily (
    calendar_date_sk        INT NOT NULL,
    listing_sk              BIGINT NOT NULL,
    listing_id              VARCHAR(64) NOT NULL,
    geo_market_sk           INT NOT NULL,
    availability_status     VARCHAR(24) NOT NULL,  -- AVAILABLE, BOOKED, HOST_BLOCKED
    listed_nightly_price_usd DECIMAL(10,2) NOT NULL,
    booking_id              VARCHAR(64),           -- Populated when BOOKED
    -- Additive Unit Flags for Trivial Occupancy Math
    available_night_flag    TINYINT NOT NULL,      -- 1 if AVAILABLE, else 0
    booked_night_flag       TINYINT NOT NULL,      -- 1 if BOOKED, else 0
    blocked_night_flag      TINYINT NOT NULL,      -- 1 if HOST_BLOCKED, else 0
    PRIMARY KEY (calendar_date_sk, listing_sk)
)
PARTITION BY (calendar_date_sk)
CLUSTER BY (geo_market_sk, listing_id);
```

---

## 4. Phase 4: Query Validation (Minutes 26–33)

```sql
-- Query 1: Monthly GAAP Recognized Revenue, Occupancy Rate & ADR by Market and Superhost Status
WITH nights_recognized AS (
    SELECT
        d.year_month,
        bn.geo_market_sk,
        u.is_superhost                                          AS was_superhost_at_booking,
        COUNT(*)                                                AS stayed_nights,
        SUM(bn.nightly_room_rate_usd + bn.allocated_cleaning_usd) AS room_revenue_usd,
        SUM(bn.allocated_net_rev_usd)                           AS recognized_net_rev_usd
    FROM gold.fct_booking_nights bn
    JOIN gold.dim_date d ON bn.stay_date_sk = d.date_sk
    JOIN gold.dim_user u ON bn.host_user_sk = u.user_sk
    WHERE d.calendar_year = 2026
      AND bn.is_canceled_night = FALSE
    GROUP BY 1, 2, 3
),
market_occupancy AS (
    SELECT
        d.year_month,
        c.geo_market_sk,
        SUM(c.booked_night_flag)                                AS total_booked_nights,
        SUM(c.available_night_flag + c.booked_night_flag)       AS total_bookable_nights
    FROM gold.fct_listing_calendar_daily c
    JOIN gold.dim_date d ON c.calendar_date_sk = d.date_sk
    WHERE d.calendar_year = 2026
    GROUP BY 1, 2
)
SELECT
    nr.year_month,
    gm.city_name,
    nr.was_superhost_at_booking,
    nr.stayed_nights,
    ROUND(nr.room_revenue_usd / NULLIF(nr.stayed_nights, 0), 2) AS adr_usd, -- True Average Daily Rate
    nr.recognized_net_rev_usd,
    ROUND(mo.total_booked_nights * 100.0 / NULLIF(mo.total_bookable_nights, 0), 2) AS market_occupancy_pct
FROM nights_recognized nr
JOIN gold.dim_geo_market gm ON nr.geo_market_sk = gm.geo_market_sk
LEFT JOIN market_occupancy mo
  ON nr.year_month = mo.year_month
 AND nr.geo_market_sk = mo.geo_market_sk
ORDER BY 1, 6 DESC;
```

---

## 5. Phase 5: Principal Curveballs & "Top-Up" Questions (Minutes 33–45)

1. **Curveball 1 (Mid-Stay Alterations & Extensions):** *"A guest books 5 nights (Nov 10–15). On Nov 12, they extend their stay by 2 nights (Nov 15–17) at a higher nightly rate. How do your tables handle alterations without breaking GAAP accounting?"*
   * **Principal Answer:** On `fct_bookings`, update `check_out_date_sk`, `nights_booked` (`5 -> 7`), and financial totals via `MERGE` (while recording an immutable audit row in `fct_booking_alterations`). On `fct_booking_nights`, **never alter already-elapsed nights (`Nov 10–11`)**—insert 2 new rows for `stay_date_sk = Nov 15` and `Nov 16` at the new rate, keeping prior recognized days 100% immutable!
2. **Curveball 2 (Taming `fct_listing_calendar_daily` Size):** *"8M listings $\times$ 365 forward days = 2.92 billion rows generated every single day if you snapshot the entire 365-day forward booking window nightly! How do you cut storage/compute by 90%?"*
   * **Principal Answer:** Separate **Historical Calendar** (`calendar_date <= CURRENT_DATE`, written once per day: only `8M` rows/day!) from **Forward Pacing Snapshots** (`fct_forward_pacing_weekly`, sampled weekly or stored as compressed run-length intervals `[available_from_date, available_to_date]` rather than dense daily rows).
3. **Curveball 3 (Multi-Valued Amenities Filter Without Fan-Out):** *"Listings have 30+ amenities (`Pool`, `WiFi`, `EV Charger`, `Hot Tub`). Analysts want to compare ADR for listings with BOTH `Pool` AND `EV Charger`."*
   * **Principal Answer:** Promote the top 10 highest-signal filter amenities (`has_wifi`, `has_pool`, `has_ev_charger`, `has_ac`, `is_pet_friendly`) as boolean columns directly on `dim_listing` (or a compact `dim_top_amenities` Junk Dimension) for zero-join filtering, while maintaining `bridge_listing_amenities` keyed by `amenity_group_sk` for the full 100+ long-tail amenity catalog.