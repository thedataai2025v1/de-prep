# Lesson 17: Practice — Video Streaming Service Data Model

> **Principal Mentor Note:** Video streaming warehouses (Netflix, Prime Video, Disney+, YouTube) test three specific data modeling muscles: **(1) Sessionizing high-frequency playback heartbeats** (`10-second` player pings) into meaningful viewing sessions without losing QoS buffering telemetry, **(2) Modeling multi-valued content metadata** (genres, cast, audio tracks) without causing fan-out when summing watch hours, and **(3) Content Licensing & Subscription Unit Economics** (amortizing fixed content licensing costs across hours watched).

---

## 1. Requirements & Scope Assumptions

* **Scale:** `250M paid subscribers`, `120M DAU`, `6B raw playback heartbeats/day` $\rightarrow$ compressed into `450M viewing sessions/day`, catalog of `50,000` movies & series episodes.
* **Core Business Processes:**
  1. **Catalog Discovery & Impressions** (`fct_title_impressions` — homepage row placement, thumbnail A/B test variant, click-through).
  2. **Playback Session Telemetry** (`fct_watch_sessions` — aggregated from raw 10s player heartbeats in Silver; tracks watch duration, completion %, buffering ratio, bitrate, device).
  3. **Subscription Billing & Lifecycle** (`fct_subscription_events` & `fct_subscriber_monthly_snap` — trials, plan upgrades/downgrades, ad-tier vs. ad-free, churn).
  4. **Content Licensing & Title Daily Rollup** (`agg_title_daily_performance` — powers Content Acquisition ROI: `licensing_cost / total_qualified_hours`).
* **Key Analytical Queries:**
  * Title Completion Rate (`completed_sessions / qualified_starts`) by content genre, release cohort, and device type.
  * Quality of Experience (QoS) Rebuffering Ratio (`total_buffer_sec / total_play_sec`) by CDN provider, ISP, and app version.
  * Subscriber Churn Rate correlated with trailing 30-day watch hours and ad-load exposure.

---

## 2. Enterprise Bus Matrix

```
Business Process (Fact Table)           | Date | Time | Account | Profile | Title/Episode | Genre (Bridge) | Device/CDN | Plan
----------------------------------------+------+------+---------+---------+---------------+----------------+------------+-----
fct_title_impressions (Transaction)     |  X   |  X   |    X    |    X    |       X       |       X        |     X      |
fct_watch_sessions (Transaction/Sess)   |  X   |  X   |    X    |    X    |       X       |       X        |     X      |  X
fct_subscription_events (Transaction)   |  X   |  X   |    X    |         |               |                |            |  X
fct_subscriber_monthly_snap (Periodic)  |  X   |      |    X    |         |               |                |            |  X
```

---

## 3. High-Level Constellation Diagram

```
+----------------------+       +---------------------+       +-----------------------+
| dim_account [SCD2]   |       |  dim_date [CONFORM] |       | dim_title [SCD1/2]    |
| PK: account_sk       |       |  PK: date_sk        |       | PK: title_sk          |
| NK: account_id       |       +----------+----------+       | NK: title_id          |
| country, plan_tier   |            ^           ^            | series_id, season_num |
+----------+-----------+            |           |            | is_original, runtime_s|
           | 1                      |           |            +-----+-------------+---+
           |                        |           |                  | 1           | 1
           | N              +-------+----+  +---+---------------+  |             |
           +--------------->| fct_sub_   |  | fct_watch_sessions|  | N           | N (M:N)
                            | monthly_   |  | [FACT-SESSION]    |<-+    +--------v--------+
+----------------------+    | snap [PER] |  | Grain: 1 row /    |       | bridge_title_   |
| dim_profile [SCD1]   |--->| Grain:     |  | watch_session_id  |       | genre           |
| PK: profile_sk       |1:N | 1/acct/mo  |  +---+---------------+       | weighting_factor|
| maturity_rating, kids|    +------------+      ^                       +-----------------+
+----------------------+                        | 1:N
                                    +-----------+-----------+
                                    | dim_device_cdn [SCD1] |
                                    | device_type, os, cdn  |
                                    +-----------------------+
```

---

## 4. Core Production DDL & Modeling Decisions

```sql
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE dim_title (
    title_sk                BIGINT PRIMARY KEY,
    title_id                VARCHAR(64) NOT NULL,  -- Episode ID or Movie ID
    series_id               VARCHAR(64),           -- NULL for standalone movies; rolls up episodes!
    series_title            VARCHAR(255),
    episode_title           VARCHAR(255),
    season_number           INT,
    episode_number          INT,
    content_type            VARCHAR(24) NOT NULL,  -- MOVIE, EPISODE, TRAILER, LIVE_EVENT
    primary_genre           VARCHAR(64) NOT NULL,  -- Primary genre for fast non-fanout slicing
    release_date            DATE NOT NULL,
    is_original_production  BOOLEAN NOT NULL,
    runtime_seconds         INT NOT NULL,
    licensing_amort_daily_usd DECIMAL(12,2)        -- Daily amortized content cost for ROI math
);

CREATE TABLE bridge_title_genre (
    title_sk                BIGINT NOT NULL,
    genre_sk                INT NOT NULL,
    genre_weight            DECIMAL(5,4) NOT NULL, -- Sums to 1.0000 per title_sk (prevents fan-out)
    is_primary_genre        BOOLEAN NOT NULL,
    PRIMARY KEY (title_sk, genre_sk)
);

CREATE TABLE dim_account (
    account_sk              BIGINT PRIMARY KEY,
    account_id              VARCHAR(64) NOT NULL,
    signup_cohort_month     DATE NOT NULL,
    country_code            CHAR(2) NOT NULL,
    billing_plan_code       VARCHAR(32) NOT NULL,  -- AD_SUPPORTED, STANDARD_HD, PREMIUM_4K
    monthly_price_usd       DECIMAL(8,2) NOT NULL,
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

-- ============================================================================
-- FACT: WATCH SESSIONS (Sessionized Transaction Fact)
-- Grain: One row per continuous playback session (watch_session_id)
-- Built in Silver/Gold by sessionizing raw 10s heartbeats (30-min inactivity gap)
-- ============================================================================
CREATE TABLE fct_watch_sessions (
    watch_session_id        VARCHAR(64) PRIMARY KEY,
    session_start_date_sk   INT NOT NULL,
    session_start_utc       TIMESTAMP NOT NULL,
    session_end_utc         TIMESTAMP NOT NULL,
    account_sk              BIGINT NOT NULL,       -- Historical SCD2 SK (captures plan at watch time)
    profile_sk              BIGINT NOT NULL,       -- Sub-account viewer profile (Kids vs Adult)
    title_sk                BIGINT NOT NULL,
    device_cdn_sk           INT NOT NULL,
    entry_surface           VARCHAR(32) NOT NULL,  -- HOME_HERO, CONTINUE_WATCHING, SEARCH, AUTOPLAY
    -- Additive Duration & QoS Measures
    active_watch_seconds    INT NOT NULL,          -- Excludes paused & buffering wall-clock time
    paused_seconds          INT NOT NULL,
    rebuffering_seconds     INT NOT NULL,
    rebuffering_event_count INT NOT NULL,
    ad_impression_count     INT NOT NULL,          -- >0 on AD_SUPPORTED tier
    ad_watch_seconds        INT NOT NULL,
    -- Progress & Completion Measures
    start_position_seconds  INT NOT NULL,
    max_position_seconds    INT NOT NULL,
    is_qualified_start      BOOLEAN NOT NULL,      -- Business rule: active_watch_seconds >= 120
    is_completed_watch      BOOLEAN NOT NULL       -- Business rule: max_position >= 0.90 * runtime
)
PARTITION BY (session_start_date_sk)
CLUSTER BY (title_sk, account_sk);
```

---

## 5. Validation Query (Weighted Multi-Genre Watch Hours & QoS Rebuffering)

```sql
-- Query: Qualified Watch Hours & Rebuffering Ratio across ALL Genres (No Fan-Out!)
SELECT
    g.genre_name,
    SUM(ws.active_watch_seconds * b.genre_weight) / 3600.0 AS weighted_watch_hours,
    SUM(ws.rebuffering_seconds * b.genre_weight) * 1.0 /
        NULLIF(SUM(ws.active_watch_seconds * b.genre_weight), 0) AS rebuffering_ratio,
    COUNT(DISTINCT CASE WHEN ws.is_qualified_start THEN ws.account_sk END) AS unique_accounts_reached
FROM fct_watch_sessions ws
JOIN dim_date d ON ws.session_start_date_sk = d.date_sk
JOIN bridge_title_genre b ON ws.title_sk = b.title_sk
JOIN dim_genre g ON b.genre_sk = g.genre_sk
WHERE d.calendar_date BETWEEN DATE '2026-11-01' AND DATE '2026-11-07'
GROUP BY 1
ORDER BY 2 DESC;
```

---

## 6. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "A user starts watching a 2-hour movie at 11:30 PM on Nov 14 and finishes at 1:30 AM on Nov 15, or pauses for 45 minutes in the middle. How do you handle cross-midnight sessions and long pauses?"**
   * **Expected Insight:**
     * **Long Pauses:** In the Silver sessionization pipeline (Spark/Flink `SESSION WINDOW` or `LAG(heartbeat_ts)`), split a session if `heartbeat_gap > 30 minutes` of inactivity so a 4-hour dinner break doesn't inflate `session_wall_clock_duration`.
     * **Cross-Midnight:** Anchor the row in `fct_watch_sessions` by `session_start_date_sk` for session-level completion metrics, *and* emit a daily-allocated slice (`fct_title_daily_watch_time`) if Finance requires exact calendar-day watch-minute splits for content royalty payouts.
2. **Top-Up 2: "Now we introduce Live Sports Streaming (e.g., NFL / IPL Cricket) where 25 million concurrent viewers join at 7:00 PM sharp and need real-time QoS bitrate dashboards within 15 seconds. How does your model adapt?"**
   * **Expected Insight:** You cannot wait for a session to end (`session_end_utc`) during a 3-hour live game! Keep `fct_watch_sessions` for post-game VOD/replay analytics, and route the live 10-second heartbeats through a Flink tumbling-window aggregator (`1-minute buckets` keyed by `(title_id, cdn_provider, country_code, app_version)`) into **Apache Pinot / ClickHouse** for sub-15-second live QoS alerting.

---

## 7. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Separates raw 10s heartbeats (Silver) from sessionized watch facts (Gold)** and defines explicit business rules for `is_qualified_start` and `is_completed_watch`. | Attempts to run BI queries directly on 6B raw heartbeat pings/day or loses rebuffering telemetry completely. |
| **Models both `account_sk` (billing/SCD2) and `profile_sk` (viewer persona)** cleanly. | Conflates household billing accounts with individual viewer profiles (ruining Kids vs. Adult analytics). |
| **Uses `genre_weight` in `bridge_title_genre` + `primary_genre` on `dim_title`** to prevent multi-genre watch-hour fan-out. | Joins `dim_title` to a `1:N` genre table and inflates total platform watch hours by 2.4x. |