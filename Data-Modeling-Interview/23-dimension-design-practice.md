# Module 4: Dimension Design
## Lesson 4.3 — Dimension Design Practice

> *"You can know every SCD type by heart, but the real test is knowing which one to use — and when to mix them."* This lesson puts you in the driver's seat with five realistic scenarios. Work through each one, then compare your answers with the solutions below.

---

### Scenario 1: E-commerce Customer Dimension

**Given:** Customers can change address, phone, preferred payment method, and loyalty tier.

**Business asks:** *"We need to track where each order was placed from, even if the customer later moved."*

**Task:** Design `dim_customer` with appropriate SCD type. Show table structure, rationale, and join to `fact_orders`.

#### Solution

**Mixed SCD — Type 2 for critical history, Type 1 for low-priority attributes:**

| Attribute | SCD Type | Rationale |
|---|---|---|
| `street_address`, `city`, `state` | **Type 2** | Critical — order origin must be preserved |
| `loyalty_tier` | **Type 2** | Segmenting customers by tier at order time |
| `customer_name` | **Type 2** | Corrections and life events need history |
| `phone_number` | **Type 1** | History never matters; overwrite is fine |
| `preferred_payment` | **Type 1** | Low impact; overwrite is fine |
| `signup_date` | **Type 0** | Immutable |

```sql
CREATE TABLE dim_customer (
    customer_key      BIGINT PRIMARY KEY,       -- surrogate key
    customer_id       VARCHAR(32) NOT NULL,      -- natural key
    customer_name     VARCHAR(128) NOT NULL,
    street_address    VARCHAR(256), city VARCHAR(64), state VARCHAR(64),
    loyalty_tier      VARCHAR(32),
    phone_number      VARCHAR(20),               -- Type 1
    preferred_payment VARCHAR(32),               -- Type 1
    signup_date       DATE NOT NULL,             -- Type 0
    effective_date    TIMESTAMP DEFAULT NOW(),
    expiry_date       TIMESTAMP,                 -- NULL = current row
    is_current        BOOLEAN DEFAULT TRUE
);

-- How it joins: fact_orders stores the surrogate key active at order time
SELECT c.city, c.loyalty_tier, SUM(o.order_amount) AS total_revenue
FROM fact_orders o
JOIN dim_customer c ON o.customer_key = c.customer_key
JOIN dim_date d ON o.date_key = d.date_key
WHERE d.date >= '2024-01-01' AND d.date < '2025-01-01'
GROUP BY c.city, c.loyalty_tier;
```

> **Why surrogate key and not `customer_id`?** The surrogate key points to the *exact version* of the customer row that existed at order time. Joining on `customer_id` alone returns the current address — destroying historical accuracy.

#### Interviewer Questions You'll Face

| # | Question | What They're Probing |
|---|----------|---------------------|
| 1 | "Why Type 2 for address but Type 1 for phone?" | SCD choice is per-attribute, driven by business need |
| 2 | "What happens when address and loyalty tier change on the same day?" | One Type 2 row captures all changes together |
| 3 | "What happens to old fact_orders rows when you close out an old dimension row?" | Facts are never updated — they keep the historical surrogate key |
| 4 | "The table is growing 5% per month. What would you do?" | Archive expired rows, partition by effective_date |

#### Top-Up Questions (Senior-Level Probes)

| # | Question |
|---|----------|
| 1 | "How would you implement this SCD logic in a streaming pipeline (Kafka → Iceberg) vs. batch ETL?" |
| 2 | "Source timestamp vs. load timestamp — how do you distinguish *when* a change happened vs. *when* we discovered it?" |
| 3 | "How do you handle a customer deleted from the source system?" |

---

### Scenario 2: Ride-Sharing Driver Dimension

**Given:** Drivers can change status (active/inactive/suspended), vehicle, and rating.

**Business asks:** *"We need to analyze how driver status affects ride quality over time."*

**Task:** Design `dim_driver` with appropriate SCD strategy.

#### Solution

**Mixed SCD — Type 2 for critical time-varying attributes, Type 1 for name:**

| Attribute | SCD Type | Rationale |
|---|---|---|
| `driver_status` | **Type 2** | Core question — correlate status at ride time with outcomes |
| `rating` | **Type 2** | Changes every ride; must know rating at ride time |
| `vehicle_make`, `vehicle_model` | **Type 2** | Vehicle changes affect ride quality |
| `background_check_expiry` | **Type 2** | Compliance — prove driver was compliant at ride time |
| `driver_name` | **Type 1** | Rarely changes |
| `signup_date` | **Type 0** | Immutable |

> **Senior note:** Because `rating` changes on every ride, a pure Type 2 `dim_driver` grows rapidly. Alternative: store rating history in `fact_rides` as a measure, and only use Type 2 for `driver_status` and `vehicle`. Raise this debate in an interview.

```sql
CREATE TABLE dim_driver (
    driver_key              BIGINT PRIMARY KEY,
    driver_id               VARCHAR(32) NOT NULL,
    driver_name             VARCHAR(128) NOT NULL,     -- Type 1
    vehicle_make            VARCHAR(64), vehicle_model VARCHAR(64),  -- Type 2
    driver_status           VARCHAR(32) NOT NULL,      -- Type 2
    rating                  DECIMAL(3, 2),             -- Type 2
    background_check_expiry DATE,                      -- Type 2
    signup_date             DATE NOT NULL,             -- Type 0
    effective_date          TIMESTAMP DEFAULT NOW(),
    expiry_date             TIMESTAMP, is_current BOOLEAN DEFAULT TRUE
);

SELECT d.driver_status, AVG(r.ride_rating) AS avg_ride_rating, COUNT(*) AS ride_count
FROM fact_rides r JOIN dim_driver d ON r.driver_key = d.driver_key
GROUP BY d.driver_status;
```

#### Interviewer Questions You'll Face

| # | Question | What They're Probing |
|---|----------|---------------------|
| 1 | "Rating changes every ride. Is Type 2 the right call?" | Knowing that high-frequency changes may belong in the fact table |
| 2 | "Show the dimension rows for a driver going active → suspended → active." | Tracing multi-state transitions in Type 2 |
| 3 | "What's the impact on existing facts when a driver's vehicle changes?" | Existing facts are untouched |
| 4 | "Could you put driver_status in the fact table instead?" | Duplicating dims into facts for query performance |

#### Top-Up Questions (Senior-Level Probes)

| # | Question |
|---|----------|
| 1 | "With 5M drivers, dim_driver grows to 2B+ rows. How do you keep this manageable?" |
| 2 | "A driver's vehicle changes mid-shift. How do you model this precisely?" |
| 3 | "Should rating be an average or a point-in-time snapshot in the dimension?" |

---

### Scenario 3: Social Media Post Dimension

**Given:** Posts have content_type, language, visibility, hashtags, and a soft-delete flag.

**Task:** Identify which attributes go in `dim_post`, which become degenerate dimensions, and which go in a junk dimension.

#### Solution

| Attribute | Placement | Why |
|---|---|---|
| `post_id` | **Degenerate dimension** in fact | No separate dimension needed — the ID is the grain |
| `content_type`, `language` | **dim_post_type** | Categorical, never change — perfect for grouping |
| `hashtags` | **Bridge table** `fact_post_hashtag` | M:N — unpack so each hashtag can be joined independently |
| `visibility`, `is_deleted` | **Junk dimension** `dim_post_flags` | Two low-cardinality flags; bundling reduces join count |
| `author_id` | FK → `dim_user` | Standard dimension join |
| `created_at` | FK → `dim_date` | Standard time dimension |

```
fact_post_events (grain: 1 row per engagement event)
├─── post_id               (degenerate — no separate dim needed)
├─── date_key              ──→ dim_date
├─── author_key            ──→ dim_user
├─── post_type_key         ──→ dim_post_type (content_type, language)
├─── post_flags_key        ──→ dim_post_flags (visibility, is_deleted)
└─── post_hashtag_key      ──→ fact_post_hashtag (bridge)
                              └─── hashtag_key ──→ dim_hashtag
```

```sql
-- Junk dimension: bundles low-cardinality flags
CREATE TABLE dim_post_flags (
    post_flags_key INT PRIMARY KEY,
    visibility     VARCHAR(16),   -- 'public', 'friends', 'private'
    is_deleted     BOOLEAN
);

-- Bridge table for M:N hashtags
CREATE TABLE fact_post_hashtag (
    bridge_key  BIGINT PRIMARY KEY,
    post_id     VARCHAR(64),
    hashtag_key INT REFERENCES dim_hashtag(hashtag_key)
);
```

> **Why a bridge table, not a comma-separated column?** Delimited strings cannot be joined, grouped, or indexed — the bridge table unpacks each hashtag independently.

#### Interviewer Questions You'll Face

| # | Question | What They're Probing |
|---|----------|---------------------|
| 1 | "Why not put everything in one big dim_post?" | Degenerate dimensions, junk dims, and bridges are distinct patterns |
| 2 | "When is a junk dimension inappropriate?" | Reduces joins but increases the cost of a single attribute change |
| 3 | "If a post's visibility changes, how does the fact reflect that?" | Old events keep the old flags_key; new events get the new one |

#### Top-Up Questions (Senior-Level Probes)

| # | Question |
|---|----------|
| 1 | "If a post has 100 hashtags, does the bridge table become a problem? How do you cap it?" |
| 2 | "The fact table gets billions of rows. What's your partition strategy?" |
| 3 | "Can the junk dimension answer 'which posts were public on Jan 1 but private on Mar 1'?" |

---

### Scenario 4: Role-Playing Date Dimension (Financial Transactions)

**Given:** Transactions have `created_at`, `settled_at`, and `reversed_at` (all dates).

**Task:** Show how `dim_date` works as a role-playing dimension. Show the join pattern.

#### Solution

One physical `dim_date` table referenced multiple times with different semantic meanings.

```sql
CREATE TABLE dim_date (
    date_key    INT PRIMARY KEY,   -- e.g., 20240115
    date        DATE NOT NULL, month INT, quarter INT,
    day_of_week VARCHAR(10), is_weekend BOOLEAN, is_holiday BOOLEAN
);

-- Three FKs to the SAME physical table, each with a different "role"
CREATE TABLE fact_transactions (
    txn_key           BIGINT PRIMARY KEY,
    created_date_key  INT REFERENCES dim_date(date_key),   -- Role 1
    settled_date_key  INT REFERENCES dim_date(date_key),   -- Role 2
    reversed_date_key INT REFERENCES dim_date(date_key),   -- Role 3 (nullable)
    amount            DECIMAL(14, 2), currency_code CHAR(3)
);

-- Settlement lag: join dim_date twice with different aliases
SELECT created.month AS creation_month,
       AVG(DATEDIFF('day', created.date, settled.date)) AS avg_settlement_lag
FROM fact_transactions t
JOIN dim_date created  ON t.created_date_key  = created.date_key
JOIN dim_date settled  ON t.settled_date_key  = settled.date_key
WHERE t.settled_date_key IS NOT NULL
GROUP BY created.month;
```

> **Key point:** There is only ONE `dim_date` table. The "roles" are a *logical* concept enforced by which foreign key column references it.

#### Interviewer Questions You'll Face

| # | Question | What They're Probing |
|---|----------|---------------------|
| 1 | "Why not create three separate date dimension tables?" | Avoids duplication and maintenance |
| 2 | "What happens if settled_date_key is NULL?" | INNER JOIN drops those rows — use LEFT JOIN for optional dates |
| 3 | "Can you join dim_date four times? Performance concerns?" | Star joins are highly optimized in columnar engines |

#### Top-Up Questions (Senior-Level Probes)

| # | Question |
|---|----------|
| 1 | "dim_date has ~10K rows. Should you materialize it or compute on the fly?" |
| 2 | "How do you handle fiscal calendars that differ across countries?" |
| 3 | "In Snowflake/BigQuery, does role-playing still have the same benefit?" |

---

### Scenario 5: Bridge Table for Many-to-Many (Video Genres)

**Given:** A video can have multiple genres; a genre has multiple videos.

**Task:** Design a bridge table (`dim_video_genres`). Show the structure and how it sits between fact and dimensions.

#### Solution

A **bridge table** (factless fact table) sits between fact and dimension to resolve M:N.

```
fact_video_views                        dim_genre
┌──────────────────┐                    ┌──────────────────┐
│ view_key (PK)    │                    │ genre_key (PK)   │
│ date_key         │                    │ genre_name       │
│ user_key         │                    │                  │
│ video_key ──────→│─→ dim_video_genres │←─────────────    │
│ duration_sec     │    (bridge)        │                  │
└──────────────────┘    ┌──────────────┐ └────────────────┘
                        │ bridge_key(PK)│
                        │ video_key     │ ←→ dim_video
                        │ genre_key     │ ←→ dim_genre
                        └──────────────┘
```

```sql
CREATE TABLE dim_genre (
    genre_key   INT PRIMARY KEY,
    genre_name  VARCHAR(64) NOT NULL UNIQUE
);

-- Bridge: one row per (video, genre) pairing
CREATE TABLE dim_video_genres (
    bridge_key  BIGINT PRIMARY KEY,
    video_key   BIGINT NOT NULL REFERENCES dim_video(video_key),
    genre_key   INT    NOT NULL REFERENCES dim_genre(genre_key),
    UNIQUE (video_key, genre_key)
);

-- Fact table has no genre_key — genres are reached via the bridge
CREATE TABLE fact_video_views (
    view_key      BIGINT PRIMARY KEY,
    video_key     BIGINT NOT NULL,
    date_key      INT NOT NULL,
    user_key      BIGINT NOT NULL,
    duration_sec  INT
);

-- Views by genre: a video in 3 genres appears 3 times (once per genre)
SELECT g.genre_name, COUNT(v.view_key) AS total_views
FROM fact_video_views v
JOIN dim_video_genres bg ON v.video_key = bg.video_key
JOIN dim_genre g ON bg.genre_key = g.genre_key
GROUP BY g.genre_name ORDER BY total_views DESC;

-- Videos that belong to BOTH 'Action' AND 'Sci-Fi'
SELECT bg1.video_key
FROM dim_video_genres bg1 JOIN dim_genre g1 ON bg1.genre_key = g1.genre_key
JOIN dim_video_genres bg2 ON bg1.video_key = bg2.video_key
JOIN dim_genre g2 ON bg2.genre_key = g2.genre_key
WHERE g1.genre_name = 'Action' AND g2.genre_name = 'Sci-Fi'
  AND bg1.genre_key < bg2.genre_key;
```

#### Interviewer Questions You'll Face

| # | Question | What They're Probing |
|---|----------|---------------------|
| 1 | "Why not store genres as a delimited string?" | Delimited strings cannot be joined, grouped, or indexed |
| 2 | "Is dim_video_genres a dimension or a fact table?" | It's a "factless fact table" — no measures |
| 3 | "If a video gains a new genre, do you update existing fact rows?" | No — INSERT a new bridge row; existing views keep old genres |

#### Top-Up Questions (Senior-Level Probes)

| # | Question |
|---|----------|
| 1 | "What happens to performance if a video has 20 genres? How do you mitigate?" |
| 2 | "In Iceberg/Delta Lake, how would you model the M:N differently?" |
| 3 | "How do you handle temporal genres — genres at view time vs. current genres?" |

---

### Practice Exercise

> **Design your own dimension model.** You're building the warehouse for a food-delivery platform (like DoorDash).

**Given:**
- Restaurants can change name, location, cuisine type, and delivery fee
- Drivers can change vehicle, status (active/inactive), and rating
- An order has a `placed_at`, `picked_up_at`, `delivered_at`, and optionally `cancelled_at`
- A restaurant can serve multiple cuisine types; a cuisine type covers many restaurants
- The business needs: revenue by cuisine over time, average delivery time by driver status at delivery time, and the ability to trace which location a restaurant was at when an order was placed

**Tasks:**
1. Sketch the star schema (fact table(s) and all dimension tables).
2. For each dimension, state the SCD type per attribute and justify it.
3. Identify any role-playing dimensions.
4. Identify any bridge tables or junk dimensions.
5. Write the SQL `CREATE TABLE` for your fact table and two dimensions.

> *When you're done, ask yourself: "Could I reconstruct the state of every entity at the time of any historical order?" If not, re-examine your SCD choices.*
---

### Related Lessons

- [Practice: E-Commerce Platform](./15-practice-ecommerce-platform.md)
- [Slowly Changing Dimensions: Types 1-4](./21-slowly-changing-dimensions-types-1-4.md)
- [Degenerate and Junk Dimensions](./22-degenerate-and-junk-dimensions.md)
- [Fact Table Types](./24-fact-table-types.md)
