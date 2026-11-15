# Lesson 35: Mock Interview 5 — Design a Data Warehouse Schema for Instagram

> **Principal Mentor Note:** Instagram is a quintessential **consumer social & short-form video (Reels) hyper-scale data modeling** interview. At `2B+ MAU` and `50B+ feed/Reels impressions/day`, naive Star Schemas collapse under **celebrity hot-key skew**, **multi-surface attribution** (Home Feed vs. Reels Tab vs. Explore vs. Stories vs. Direct Messages), **Reels watch-time loops (>100% completion)**, and **28-day rolling DAU/WAU/MAU retention math**.

---

## 1. Phase 1: Clarifying Questions & Scope Alignment (Minutes 0–6)

### Interviewer Prompt
> *"Design the core analytical data warehouse for Instagram. Product, Creator Growth, and Integrity teams want to measure DAU/WAU/MAU stickiness, Reels vs. Feed engagement, creator monetization/reach, and content virality."*

### Candidate's 4-Bucket Discovery Questions
1. **Product Surfaces & Content Types:** *"Should we cover all core surfaces (`Home Feed`, `Reels`, `Explore`, `Stories`, `Profile`) and content formats (`Photo`, `Carousel`, `Reel`, `Story`), plus social graph follow/unfollow events and Direct Message (DM) shares?"*
   * *Interviewer:* *"Yes—focus on Feed, Reels, Stories, and Explore impressions, engagements (likes, comments, saves, DM reshares), creator growth, and DAU/WAU/MAU retention."*
2. **Reels Watch-Time & Story Ephemerality:** *"For Reels, users frequently watch a 15-second loop 2.5 times (`250%` completion). Should we track raw watch milliseconds alongside loop count, and how should 24-hour expiring Stories be retained analytically?"*
   * *Interviewer:* *"Track exact watch milliseconds and loop completions for Reels, and retain Story metadata in the warehouse permanently even after the 24-hour user-facing expiry."*
3. **Creator Attributes & Scale:** *"Creator follower counts fluctuate hourly—should follower tiers be tracked as SCD Type 2 on `dim_user` or via a Mini-Dimension? Also, what scale and freshness SLA apply?"*
   * *Interviewer:* *"Assume 600M DAU, 2.5B registered users, 40B media impressions/day, 3B engagements/day, 1-hour freshness for Product/Creator dashboards, and strict GDPR/account-deletion compliance."*

---

## 2. Phase 2: Enterprise Bus Matrix & Grain Contract (Minutes 6–12)

### Enterprise Bus Matrix
```
Business Process (Fact Table)              | Date | Hour | Viewer User | Author/Creator | Creator Tier | Media/Post | Surface | Device
-------------------------------------------+------+------+-------------+----------------+--------------+------------+---------+-------
fct_media_impressions (Transaction - Hot)  |  X   |  X   |      X      |       X        |      X       |     X      |    X    |   X
agg_media_delivery_hourly (Aggregate)      |  X   |  X   |             |       X        |      X       |     X      |    X    |   X
fct_engagements (Transaction Fact)         |  X   |  X   |      X      |       X        |      X       |     X      |    X    |   X
fct_reels_watch_sessions (Transaction)     |  X   |  X   |      X      |       X        |      X       |     X      |    X    |   X
fct_user_daily_activity (Periodic Snap)    |  X   |      |      X      |                |              |            |         |   X
fct_follow_graph_events (Transaction)      |  X   |  X   |      X      |       X        |              |            |    X    |
```

### Explicit Grain Declarations
1. **`agg_media_delivery_hourly` (Primary BI Impression Serving Fact):** One row per `(delivery_date_sk, delivery_hour_utc, media_sk, surface_sk, device_sk, viewer_country_code)`—compressing `40B` raw impressions/day into `~120M` rows/day while retaining HyperLogLog unique viewer sketches. (Raw `fct_media_impressions` lives in 14-day hot Iceberg storage for ranking ML training & integrity forensics).
2. **`fct_engagements` (Consolidated Transaction Fact):** One row per discrete engagement action (`engagement_id`: `LIKE`, `UNLIKE`, `COMMENT`, `SAVE`, `REPOST`, `DM_SHARE`), carrying pre-pivoted signed delta flags (`+1` / `-1`).
3. **`fct_reels_watch_sessions` (Transaction Fact):** One row per Reels viewport playback session (`watch_session_id`), capturing `watch_duration_ms`, `media_duration_ms`, `loop_count`, and `is_3sec_qualified_view`.
4. **`fct_user_daily_activity` (Periodic Snapshot):** One row per active `(viewer_user_sk, activity_date_sk)`, carrying a **28-bit integer activity bitmask** (`active_28d_bitmask`) for single-partition DAU/WAU/MAU calculation.

---

## 3. Phase 3: High-Level Constellation Diagram & DDL (Minutes 12–26)

```
+----------------------+       +---------------------+       +-----------------------+
| dim_user [SCD2]      |       |  dim_date [CONFORM] |       | dim_media [SCD1]      |
| PK: user_sk          |       |  PK: date_sk        |       | PK: media_sk          |
| NK: user_id          |       +----------+----------+       | NK: media_id          |
| (Role: Viewer/Author)|            ^           ^            | author_user_sk,       |
| cohort_mo, country   |            |           |            | media_type (REEL,etc) |
+---+--------------+---+            |           |            | audio_track_id, dur_ms|
    | 1            | 1              |           |            +-----+-------------+---+
    |              |        +-------+---+  +----+--------------+   |             |
    | N            | N      | fct_user_ |  | fct_engagements   |<--+             | N
    |              +------->| daily_    |  | [FACT-TXN]        |                 v
    |                       | activity  |  | Grain: 1/action   |       +-------------------+
    | 1                     | [PER-SNAP]|  +----+--------------+       | agg_media_        |
    +---------------------->| 28d_mask  |       ^                      | delivery_hourly   |
                            +-----------+       | 1:N                  | [AGG-HRLY + HLL]  |
+----------------------+            +-----------+-----------+          +-------------------+
| dim_creator_tier     |            | dim_surface [JUNK/SCD1|
| [MINI-DIM - 8 ROWS]  |----------->| HOME_FEED, REELS_TAB, |
| NANO..MEGA_CELEBRITY |    1:N     | EXPLORE, STORIES, DM  |
+----------------------+            +-----------------------+
```

### Production DDL (Apache Iceberg / Snowflake)

```sql
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE gold.dim_user (
    user_sk                 BIGINT PRIMARY KEY,
    user_id                 VARCHAR(64) NOT NULL,
    signup_date_sk          INT NOT NULL,
    signup_cohort_month     DATE NOT NULL,
    country_code            CHAR(2) NOT NULL,
    account_category        VARCHAR(24) NOT NULL,  -- PERSONAL, CREATOR, BUSINESS
    is_verified_badge       BOOLEAN NOT NULL,
    is_private_account      BOOLEAN NOT NULL,
    -- PII (email, phone, full_name, bio_text) isolated in dim_user_pii for GDPR!
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

-- Static Mini-Dimension: Prevents 2.5B-row SCD2 explosion when follower counts change!
CREATE TABLE gold.dim_creator_tier (
    creator_tier_sk         TINYINT PRIMARY KEY,
    tier_code               VARCHAR(24) NOT NULL,  -- NANO(<1K), MICRO(1K-10K), MID(10K-100K), MACRO(100K-1M), MEGA(1M+)
    min_followers           BIGINT NOT NULL,
    max_followers           BIGINT NOT NULL
);

CREATE TABLE gold.dim_media (
    media_sk                BIGINT PRIMARY KEY,
    media_id                VARCHAR(64) NOT NULL,
    author_user_sk          BIGINT NOT NULL,
    media_type              VARCHAR(24) NOT NULL,  -- PHOTO, CAROUSEL, REEL, STORY
    carousel_item_count     TINYINT NOT NULL DEFAULT 1,
    audio_track_id          VARCHAR(64),           -- Links Reels using the same trending audio!
    original_media_id       VARCHAR(64),           -- Links Remixes / Reposts to root viral post
    media_duration_ms       INT,                   -- NULL for static photos
    created_at_utc          TIMESTAMP NOT NULL,
    expires_at_utc          TIMESTAMP,             -- +24h for STORIES; NULL for permanent posts
    is_paid_partnership     BOOLEAN NOT NULL DEFAULT FALSE
);

-- ============================================================================
-- FACT 1: UNIFIED ENGAGEMENTS (Transaction Fact)
-- Grain: One row per discrete user engagement event (engagement_id)
-- ============================================================================
CREATE TABLE gold.fct_engagements (
    engagement_id           VARCHAR(64) NOT NULL,
    occurred_date_sk        INT NOT NULL,
    occurred_at_utc         TIMESTAMP NOT NULL,
    viewer_user_sk          BIGINT NOT NULL,
    author_user_sk          BIGINT NOT NULL,
    author_tier_sk          TINYINT NOT NULL,      -- Stamped at engagement time!
    media_sk                BIGINT NOT NULL,
    surface_sk              INT NOT NULL,          -- HOME_FEED, REELS_TAB, EXPLORE, STORIES
    device_sk               INT NOT NULL,
    engagement_action       VARCHAR(24) NOT NULL,  -- LIKE, UNLIKE, COMMENT, SAVE, UNSAVE, DM_SHARE
    -- Signed Additive Delta Flags (+1 for action, -1 for undo action)
    net_like_delta          TINYINT NOT NULL,      -- +1, -1, or 0
    net_comment_delta       TINYINT NOT NULL,
    net_save_delta          TINYINT NOT NULL,
    dm_share_count          TINYINT NOT NULL       -- 1 or 0 (sends post via Direct Message)
)
PARTITION BY (occurred_date_sk)
CLUSTER BY (author_user_sk, media_sk);

-- ============================================================================
-- FACT 2: DAILY USER ACTIVITY & BITMASK RETENTION (Periodic Snapshot)
-- Grain: One row per active (viewer_user_sk, activity_date_sk)
-- ============================================================================
CREATE TABLE gold.fct_user_daily_activity (
    activity_date_sk        INT NOT NULL,
    viewer_user_sk          BIGINT NOT NULL,
    country_code            CHAR(2) NOT NULL,
    primary_surface_sk      INT NOT NULL,
    days_since_signup       INT NOT NULL,
    active_28d_bitmask      INT NOT NULL,          -- Bit 0 = today ... Bit 27 = 27 days ago!
    feed_impressions        INT NOT NULL,
    reels_watched_count     INT NOT NULL,
    stories_viewed_count    INT NOT NULL,
    total_watch_time_sec    INT NOT NULL,
    engagements_created     INT NOT NULL,
    PRIMARY KEY (activity_date_sk, viewer_user_sk)
)
PARTITION BY (activity_date_sk)
CLUSTER BY (viewer_user_sk);
```

---

## 4. Phase 4: Query Validation (Minutes 26–33)

```sql
-- Query 1: Reels Virality (DM Share Rate, Loop Depth & Net Engagement Rate) by Creator Tier
WITH delivery AS (
    SELECT
        author_tier_sk,
        SUM(valid_impressions)                              AS impressions,
        HLL_ESTIMATE(HLL_COMBINE(unique_viewer_hll))        AS unique_reach
    FROM gold.agg_media_delivery_hourly
    WHERE delivery_date_sk BETWEEN 20261108 AND 20261114
      AND media_type = 'REEL'
    GROUP BY 1
),
engagements AS (
    SELECT
        e.author_tier_sk,
        SUM(e.net_like_delta)                               AS net_likes,
        SUM(e.net_comment_delta)                            AS net_comments,
        SUM(e.net_save_delta)                               AS net_saves,
        SUM(e.dm_share_count)                               AS dm_shares
    FROM gold.fct_engagements e
    JOIN gold.dim_media m ON e.media_sk = m.media_sk
    WHERE e.occurred_date_sk BETWEEN 20261108 AND 20261114
      AND m.media_type = 'REEL'
    GROUP BY 1
)
SELECT
    ct.tier_code                                            AS creator_tier,
    d.impressions,
    d.unique_reach,
    e.dm_shares,
    -- DM Shares per 1,000 impressions ("Send Rate" — Instagram's #1 virality signal!)
    ROUND(e.dm_shares * 1000.0 / NULLIF(d.impressions, 0), 2) AS dm_shares_per_1k_impr,
    ROUND((e.net_likes + e.net_comments + e.net_saves + e.dm_shares) * 100.0 /
          NULLIF(d.impressions, 0), 3)                      AS total_engagement_rate_pct
FROM gold.dim_creator_tier ct
LEFT JOIN delivery d ON ct.creator_tier_sk = d.author_tier_sk
LEFT JOIN engagements e ON ct.creator_tier_sk = e.author_tier_sk
ORDER BY ct.creator_tier_sk;
```

---

## 5. Phase 5: Principal Curveballs & "Top-Up" Questions (Minutes 33–45)

1. **Curveball 1 (Celebrity Skew on `author_user_sk`):** *"When a mega-creator with 400M followers drops a Reel, `fct_engagements` and `fct_reels_watch_sessions` receive 80M rows for a single `author_user_sk` in 2 hours. How do you prevent join stragglers in downstream daily creator rollups?"*
   * **Principal Answer:** Two-pronged defense: (1) Denormalize `author_tier_sk` and `author_country_code` directly onto `fct_engagements` and `agg_media_delivery_hourly` so tier-level executive queries never join `dim_user` at all; (2) For per-creator rollups, use **Two-Stage Salted Pre-Aggregation**—group by `(author_user_sk, MOD(ABS(HASH(viewer_user_sk)), 64))` in Stage 1 across 64 parallel tasks, then sum the 64 partial buckets in Stage 2!
2. **Curveball 2 (Carousel Multi-Slide Engagement Attribution):** *"A user swipes through 7 slides of a 10-slide Carousel post and likes slide #4. How do you model Carousel slide depth without multiplying post-level metrics?"*
   * **Principal Answer:** Keep `fct_engagements` and `agg_media_delivery_hourly` at the **post grain (`media_sk`)** so total post impressions and likes never fan out, and add `max_slide_viewed` (`1..10`) on `fct_media_impressions` plus `engaged_slide_index` on `fct_engagements`.
3. **Curveball 3 (28-Day Bitmask Update Cost at 600M DAU):** *"How do you compute `active_28d_bitmask` for 600M daily active users without scanning 28 days of history every night?"*
   * **Principal Answer:** Incremental bitwise shift! Take yesterday's partition (`T - 1`, `~600M–1.8B` users active in the last 28 days), `FULL OUTER JOIN` with today's active users (`T`), and compute `((COALESCE(yesterday.active_28d_bitmask, 0) << 1) & 268435455) | (CASE WHEN today.user_sk IS NOT NULL THEN 1 ELSE 0 END)` (`268435455 = 2^28 - 1`). Only drop rows where `active_28d_bitmask = 0` (users inactive for 29 consecutive days).