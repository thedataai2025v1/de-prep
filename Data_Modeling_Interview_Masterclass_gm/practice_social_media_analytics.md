# Lesson 16: Practice — Social Media Analytics Data Model

> **Principal Mentor Note:** Social media platforms (Instagram, LinkedIn, X/Twitter, TikTok) are extreme stress tests for **high-volume event telemetry** (`10B+ impressions/day`), **asymmetric social graphs** (100M-follower celebrities causing severe join skew), **multi-type polymorphic engagements** (likes, comments, shares, saves, dwell time), and **DAU/WAU/MAU cohort retention**.

---

## 1. Requirements & Scope Assumptions

* **Scale:** `300M DAU`, `1B total users`, `12B feed impressions/day`, `1.5B engagements/day`, `20M new posts/day`.
* **Core Business Processes:**
  1. Content Creation (`fct_posts` / `dim_post`)
  2. Feed Impressions & Dwell Telemetry (`fct_feed_impressions` + hourly rollups)
  3. User Engagements (`fct_engagements`: Likes, Comments, Shares, Saves, Profile Clicks)
  4. Social Graph Edge Changes (`fct_follow_events` + current `bridge_user_follows`)
  5. Daily User Activity & Retention (`fct_user_daily_activity` — powers DAU/WAU/MAU)
* **Key Analytical Queries:**
  * Creator engagement rate (`(Likes + Comments + Shares + Saves) / Impressions`) by content format (Reel, Carousel, Photo) and creator follower tier.
  * D1, D7, D30 retention cohorts by acquisition country and platform.
  * Viral cascade depth (how many reshares originated from a root post).

---

## 2. Enterprise Bus Matrix

```
Business Process (Fact Table)           | Date | Time | Actor User | Author/Creator | Post/Content | Surface/Feed | Device
----------------------------------------+------+------+------------+----------------+--------------+--------------+-------
fct_feed_impressions (Transaction)      |  X   |  X   |     X      |       X        |      X       |      X       |   X
fct_engagements (Transaction)           |  X   |  X   |     X      |       X        |      X       |      X       |   X
fct_follow_events (Transaction)         |  X   |  X   |     X      |       X        |              |              |
fct_user_daily_activity (Periodic Snap) |  X   |      |     X      |                |              |              |   X
agg_post_daily_metrics (Aggregate)      |  X   |      |            |       X        |      X       |      X       |
```

---

## 3. High-Level Constellation Diagram

```
+----------------------+       +---------------------+       +----------------------+
| dim_user [SCD2/MINI] |       |  dim_date [CONFORM] |       | dim_post [SCD1]      |
| PK: user_sk          |       |  PK: date_sk        |       | PK: post_sk          |
| NK: user_id          |       +----------+----------+       | NK: post_id          |
| (Role: Actor & Author)|            ^           ^           | root_post_id (viral) |
+---+--------------+---+            |           |           +----------+-----------+
    | 1            | 1              |           |                      | 1
    |              |        +-------+---+  +----+--------------+       |
    | N            | N      | fct_feed_ |  | fct_engagements   |       | N
    |              +------->|impressions|  | [FACT-TXN]        |<------+
    |                       | [FACT-TXN]|  | Grain: 1/action   |
    |                       +-----------+  +-------------------+
    |                                                ^
    | 1       +---------------------------+          | 1:N
    +-------->| fct_user_daily_activity   |   +------+---------------+
              | [FACT-PERIODIC SNAPSHOT]  |   | dim_engagement_type  |
              | Grain: 1/(user_sk,date_sk)|   | (LIKE,COMMENT,SHARE) |
              +---------------------------+   +----------------------+
```

---

## 4. Core Production DDL & Architectural Choices

```sql
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE dim_user (
    user_sk                 BIGINT PRIMARY KEY,
    user_id                 VARCHAR(64) NOT NULL,
    signup_date_sk          INT NOT NULL,
    signup_cohort_month     DATE NOT NULL,
    country_code            CHAR(2),
    account_type            VARCHAR(24),           -- PERSONAL, CREATOR, BUSINESS
    -- Volatile counts (follower_count) are NOT SCD2 tracked here to prevent row explosion!
    -- Instead, bucketed into a Mini-Dimension (dim_creator_tier) stamped on facts.
    is_verified             BOOLEAN,
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

CREATE TABLE dim_creator_tier (
    creator_tier_sk         INT PRIMARY KEY,       -- Mini-dimension (~10 static rows!)
    tier_code               VARCHAR(32) NOT NULL,  -- NANO(<1K), MICRO(1K-50K), MACRO(50K-1M), MEGA(1M+)
    min_followers           BIGINT NOT NULL,
    max_followers           BIGINT NOT NULL
);

CREATE TABLE dim_post (
    post_sk                 BIGINT PRIMARY KEY,
    post_id                 VARCHAR(64) NOT NULL,
    author_user_sk          BIGINT NOT NULL,
    root_post_id            VARCHAR(64),           -- Self-reference to original post for viral reshare trees
    media_type              VARCHAR(24) NOT NULL,  -- PHOTO, CAROUSEL, REEL, TEXT
    has_audio               BOOLEAN,
    language_code           CHAR(2),
    created_at_utc          TIMESTAMP NOT NULL,
    is_deleted              BOOLEAN DEFAULT FALSE
);

-- ============================================================================
-- FACT 1: UNIFIED ENGAGEMENTS (Consolidated Transaction Fact)
-- Grain: One row per discrete user engagement action (engagement_id)
-- Why Consolidated? Likes, Comments, Shares, and Saves share 95% identical
-- dimensions. Consolidating avoids 4-way UNION ALLs for total engagement KPIs.
-- ============================================================================
CREATE TABLE fct_engagements (
    engagement_id           VARCHAR(64) NOT NULL,
    occurred_date_sk        INT NOT NULL,
    occurred_at_utc         TIMESTAMP NOT NULL,
    actor_user_sk           BIGINT NOT NULL,       -- Role-playing FK -> dim_user (who engaged)
    author_user_sk          BIGINT NOT NULL,       -- Role-playing FK -> dim_user (content creator)
    author_tier_sk          INT NOT NULL,          -- FK -> dim_creator_tier (at time of engagement!)
    post_sk                 BIGINT NOT NULL,
    engagement_type_sk      INT NOT NULL,          -- 1=LIKE, 2=COMMENT, 3=SHARE, 4=SAVE
    surface_sk              INT NOT NULL,          -- HOME_FEED, EXPLORE, REELS_TAB, PROFILE
    device_sk               INT NOT NULL,
    -- Additive Measures (Pre-pivoted flags make conditional sums effortless)
    engagement_count        INT DEFAULT 1,
    like_count              TINYINT NOT NULL,      -- 1 or 0
    comment_count           TINYINT NOT NULL,      -- 1 or 0
    share_count             TINYINT NOT NULL,      -- 1 or 0
    save_count              TINYINT NOT NULL,      -- 1 or 0
    comment_char_length     INT                    -- NULL for non-comment events
)
PARTITION BY (occurred_date_sk)
CLUSTER BY (author_user_sk, post_sk);

-- ============================================================================
-- FACT 2: DAILY USER ACTIVITY (Periodic Snapshot for DAU / WAU / MAU & Retention)
-- Grain: One row per active (user_sk, activity_date_sk)
-- ============================================================================
CREATE TABLE fct_user_daily_activity (
    activity_date_sk        INT NOT NULL,
    user_sk                 BIGINT NOT NULL,
    country_code            CHAR(2) NOT NULL,
    primary_device_sk       INT NOT NULL,
    days_since_signup       INT NOT NULL,          -- Pre-computed cohort offset (0, 1, 7, 30...)
    -- Bitmask for ultra-fast rolling 7d/28d WAU/MAU computation!
    active_last_28d_bitmask INT NOT NULL,          -- Bit 0 = today, Bit 1 = yesterday ... Bit 27 = 27d ago
    impressions_viewed      INT NOT NULL,
    posts_created           INT NOT NULL,
    engagements_given       INT NOT NULL,
    active_dwell_seconds    INT NOT NULL,
    PRIMARY KEY (activity_date_sk, user_sk)
)
PARTITION BY (activity_date_sk)
CLUSTER BY (user_sk);
```

---

## 5. Principal Trick: Bitmask Rolling WAU/MAU & Cohort Retention SQL

Notice `active_last_28d_bitmask` in `fct_user_daily_activity`—a classic Meta/Instagram data engineering pattern! Instead of scanning 28 days of partitions (`1.5B * 28 = 42B rows`) every day to compute MAU, you scan **1 single day's partition (`300M rows`)**:

```sql
-- Compute DAU, WAU, and MAU on '2026-11-15' by scanning ONLY ONE DAY's partition!
SELECT
    d.calendar_date,
    u.country_code,
    COUNT(CASE WHEN (a.active_last_28d_bitmask & 1) > 0 THEN 1 END)       AS dau,
    COUNT(CASE WHEN (a.active_last_28d_bitmask & 127) > 0 THEN 1 END)     AS wau, -- 2^7 - 1 = 127
    COUNT(CASE WHEN a.active_last_28d_bitmask > 0 THEN 1 END)             AS mau,
    ROUND(
        COUNT(CASE WHEN (a.active_last_28d_bitmask & 1) > 0 THEN 1 END) * 1.0 /
        NULLIF(COUNT(CASE WHEN a.active_last_28d_bitmask > 0 THEN 1 END), 0),
    4) AS dau_mau_stickiness_ratio
FROM fct_user_daily_activity a
JOIN dim_date d ON a.activity_date_sk = d.date_sk
JOIN dim_user u ON a.user_sk = u.user_sk
WHERE d.calendar_date = DATE '2026-11-15'
GROUP BY 1, 2;
```

---

## 6. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "How do you prevent 'Celebrity Skew' when joining `fct_engagements` to `dim_user` on `author_user_sk` (e.g., Cristiano Ronaldo or Taylor Swift receives 50M engagements in an hour)?"**
   * **Expected Insight:** First, pre-denormalize the top 3 high-frequency creator attributes (`author_country`, `author_tier_sk`, `author_account_type`) onto `agg_post_daily_metrics` or `fct_engagements` so 90% of queries don't even need the `dim_user` join. Second, when joining to `dim_user` at scale, enable **Skew Join Optimization (AQE)** or apply **Deterministic Key Salting (`0..31`)** on known mega-creator IDs (`follower_count > 1M`).
2. **Top-Up 2: "A user likes a post at 10:01 AM and unlikes it at 10:03 AM. How do you model 'Unlikes' and 'Unfollows'?"**
   * **Expected Insight:** In the raw event log (`fct_engagements`), keep both `LIKE (+1)` and `UNLIKE (-1)` as immutable audit events (so Product can analyze accidental-tap/unlike UX friction), while in `agg_post_daily_metrics` sum the net delta (`SUM(net_like_delta)`). For the current social graph (`bridge_user_follows`), maintain current active edges via CDC `MERGE` with `followed_at_utc` and soft-delete `unfollowed_at_utc`.