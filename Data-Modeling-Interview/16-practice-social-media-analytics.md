# Module 3: High-Level Model Diagrams
## Lesson 3.4 — Practice: Social Media Analytics

> *"If you can't draw it, you don't understand it."* — Every senior data architect, ever.

This is a full interview simulation. You'll design a data warehouse for a social media platform like Twitter/X — from requirements through a complete high-level data model.

---

## 1. The Interview Prompt

> **"Design a data warehouse for a social media platform like Twitter/X. Walk me through your approach from requirements to a high-level data model."**

| Skill Tested | What They Look For |
|---|---|
| Structured thinking | Requirements first, then model — not tables first |
| Scale awareness | Reasoning about 500M events/day, 300M users |
| Granularity judgment | Fact table grain decisions under pressure |
| Realism | SCD strategies, dedup, semi-structured data |

---

## 2. Requirements Gathering

### 2.1 Business Goals & Key Metrics

| Goal | KPIs | Refresh |
|---|---|---|
| Track engagement | Engagement rate (likes+retweets+replies / impressions) | Near-real-time |
| Analyse content trends | Hashtag velocity, trending topics by region | Hourly |
| Monitor user growth | DAU/MAU, follower growth, churn, reactivation | Daily |
| Detect abuse | Report rate, bot-pattern composite score | Near-real-time |

### 2.2 Scale Requirements

| Parameter | Value | Modelling Implication |
|---|---|---|
| Event volume | ~500M events/day | Partitioning mandatory on largest fact |
| User base | 300M users | Dimension manageable with slow growth |
| Post creation | 100K+/min (≈144M/day) | `fact_post_events` = largest table |
| Latency | Near-real-time (abuse); daily (trends) | Dual-path: streaming + batch |
| Retention | 2 yrs event detail; compressed aggregates beyond | Tiered storage / aggregation layers |

### Interviewer Questions You'll Face

1. **"Walk me through how you'd discover these requirements. Who would you talk to?"**
2. **"Which metric is hardest to compute at scale, and why?"**
3. **"If you could only deliver ONE metric in week one, which do you pick and why?"**

### Top-Up Questions (Senior-Level Probes)

1. **"How would this design change if the company pivots to short-form video?"**
2. **"Where does the line between data warehouse and data lakehouse blur for you?"**
3. **"How do you handle data quality when events arrive out of order?"**

---

## 3. High-Level Model Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     SOCIAL MEDIA DW — STAR SCHEMA                       │
│                                                                          │
│  ┌──────────┐     ┌────────────┐     ┌─────────────────────────────┐    │
│  │ dim_date │◄────│ dim_user   │◄────│  fact_post_events           │    │
│  │ (730 rows)│    │ (SCD-2)    │     │  Grain: 1 row = 1 event     │    │
│  └──────────┘     └─────┬──────┘     │  ~500M rows/day             │    │
│                         │            └──────────────▲──────────────┘    │
│  ┌──────────┐     ┌─────┴──────┐                     │ 1-to-many       │
│  │ dim_post │◄────│fact_content│                     │                 │
│  │ (daily   │     │_posts      │                     │                 │
│  │   grain) │     │ (144M/day) │                     │                 │
│  └────┬─────┘     └─────┬──────┘                     │                 │
│       │                 │                            │                 │
│  ┌────┴─────┐     ┌─────┴──────┐                     │                 │
│  │dim_hashtag│    │fact_user_  │                     │                 │
│  │(bridge)  │     │_snapshots  │                     │                 │
│  └──────────┘     │ (1/day/user│                     │                 │
│                   │   300M)    │                     │                 │
│                   └────────────┘                     │                 │
│  ┌──────────┐                                      │                 │
│  │dim_device │◄─────────────────────────────────────┤                 │
│  └──────────┘                                       │                 │
│  ┌──────────┐                                      │                 │
│  │dim_location│◄────────────────────────────────────┤                 │
│  └──────────┘                                       │                 │
│                                                     │                 │
│  ┌──────────────────────────────────────────────────┤                 │
│  │         fact_post_events (main fact)              │                 │
│  │  event_key(PK), post_key(FK), user_key(FK),      │                 │
│  │  event_type, event_date_key, device_key,         │                 │
│  │  location_key, event_timestamp, session_id       │                 │
│  └──────────────────────────────────────────────────┘                 │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Detailed Table Definitions

### 4.1 Fact Tables

#### `fact_post_events` — Event-Level Grain (Primary Fact)

| Column | Type | FK | Notes |
|---|---|---|---|
| `event_key` | BIGINT | — | Surrogate PK (assigned after dedup) |
| `post_key` | BIGINT | `dim_post` | Post the event relates to |
| `user_key` | BIGINT | `dim_user` | User who performed action |
| `event_type_key` | TINYINT | lookup | 1=like, 2=retweet, 3=reply, 4=share, 5=bookmark |
| `event_date_key` | DATE | `dim_date` | **Partition column** |
| `device_key` | BIGINT | `dim_device` | Client device |
| `location_key` | INT | `dim_location` | GeoIP location |
| `event_timestamp` | TIMESTAMP | — | Exact event time |
| `session_id` | VARCHAR(64) | — | Client session (dedup key) |
| `source_platform` | VARCHAR(20) | — | web, ios, android, api |

#### `fact_content_posts` — Post-Level Grain

| Column | Type | FK | Notes |
|---|---|---|---|
| `post_key` | BIGINT | — | Surrogate PK |
| `author_key` | BIGINT | `dim_user` | Post author |
| `created_at` | TIMESTAMP | — | Post creation |
| `text_content` | TEXT | — | Post body |
| `media_count` | TINYINT | — | Images/video attached |
| `lang_code` | CHAR(2) | — | ISO-639 |
| `visibility_flag` | TINYINT | — | 1=public, 2=followers-only |
| `deleted_flag` | BOOLEAN | — | Soft delete |

#### `fact_user_snapshots` — Daily User Snapshot

| Column | Type | FK | Notes |
|---|---|---|---|
| `snapshot_date_key` | DATE | `dim_date` | Daily snapshot (PK) |
| `user_key` | BIGINT | `dim_user` | (PK) one row per user/day |
| `follower_count` | INT | — | Midnight value |
| `following_count` | INT | — | Midnight value |
| `post_count` | INT | — | Total posts ever |
| `engagement_score` | DECIMAL(5,2) | — | Composite metric |

### 4.2 Dimension Tables

#### `dim_user` — SCD Type 2

| Column | Type | Notes |
|---|---|---|
| `user_key` | BIGINT | Surrogate PK |
| `user_id` | VARCHAR(32) | Natural key (external) |
| `display_name` | VARCHAR(100) | Changes → SCD-2 tracked |
| `bio` | TEXT | Changes → SCD-2 tracked |
| `signup_date` | DATE | Stable; not SCD-tracked |
| `scd_start_date` | DATE | Row effective from |
| `scd_end_date` | DATE | NULL = current row |
| `is_current` | BOOLEAN | Optimisation for current lookup |

> **SCD-2 strategy:** Daily batch checks `display_name` and `bio` for changes. Closes old row (`scd_end_date` = today), opens new row. `signup_date` is Type 1 (overwritten in-place) — it never changes, so SCD-2 overhead is wasted here.

#### `dim_hashtag` — Many-to-Many Bridge

| Column | Type | Notes |
|---|---|---|
| `hashtag_key` | BIGINT | Surrogate PK |
| `tag_text` | VARCHAR(50) | Normalised: lowercase, trimmed |
| `first_seen_date` | DATE | First appearance |
| `post_key` | BIGINT | FK to `dim_post` (bridge table) |

#### `dim_date` — Calendar

Standard: `date_key`, `year`, `month`, `day`, `day_of_week`, `is_weekend`, `is_holiday`, `quarter`. ~730 rows for 2-year retention.

#### `dim_device` — SCD Type 1 (Slowly changing, overwrite OK)

| Column | Type | Notes |
|---|---|---|
| `device_key` | BIGINT | Surrogate PK |
| `os` | VARCHAR(20) | iOS, Android, Windows |
| `browser` | VARCHAR(30) | Chrome, Safari |
| `app_version` | VARCHAR(16) | Version string |
| `device_type` | VARCHAR(10) | phone, tablet, desktop |

#### `dim_location` — SCD Type 1

| Column | Type | Notes |
|---|---|---|
| `location_key` | INT | Surrogate PK |
| `country_code` | CHAR(2) | ISO-3166 |
| `country_name` | VARCHAR(60) | — |
| `region` | VARCHAR(50) | State/province |
| `city` | VARCHAR(50) | — |
| `latitude` | DECIMAL(9,6) | Geo coordinate |
| `longitude` | DECIMAL(9,6) | Geo coordinate |

---

## 5. Discussion Points

### 5.1 Why Separate `fact_post_events` and `fact_content_posts`?

| Reason | Detail |
|---|---|
| **Grain mismatch** | Post-level attributes (text, media_count) vs. per-event actions |
| **Query performance** | "Trending posts" needs post table only; no 500M-row join |
| **Storage** | Post text stored once in `fact_content_posts`; events reference by key |
| **Partitioning** | Events partition by `event_date_key`; posts by `created_at` — different lifecycles |
| **Change frequency** | Post attributes stable; events purely append-only |

### 5.2 Event Deduplication

Events arrive from multiple channels (web, iOS, Android, partner APIs). Dedup uses a **composite natural key** *before* assigning surrogate keys:

```sql
INSERT INTO fact_post_events_dedup
SELECT * FROM fact_post_events_staging
WHERE (user_id, post_id, event_type, session_id, event_timestamp) IN (
    SELECT user_id, post_id, event_type, session_id, MIN(event_timestamp)
    FROM fact_post_events_staging
    GROUP BY user_id, post_id, event_type, session_id
);
```

> **Rule:** Always dedup in the staging layer before loading to the fact table. Assign surrogate `event_key` after dedup.

### 5.3 Semi-Structured Data: Hashtags as Arrays

Posts carry multiple hashtags. Two modelling approaches:

| Approach | Pro | Con |
|---|---|---|
| **Bridge table** (`dim_hashtag`) | Full SQL queryability, indexable, portable | Row multiplication; storage overhead |
| **ARRAY / JSON column** in `dim_post` | Compact; native in BigQuery/Snowflake | Harder to query; less portable |

**Recommendation:** Bridge table for the warehouse (standard SQL, indexable). Keep raw JSON array in the landing zone for ML pipelines.

### Interviewer Questions You'll Face

1. **"How do you handle a user changing their display name mid-day?"**
2. **"What happens when an event arrives hours late — out of order?"**
3. **"For 'top 10 posts by engagement in the last hour', which table do you hit?"**

### Top-Up Questions (Senior-Level Probes)

1. **"How does this model change for live video with per-second view counts?"**
2. **"Walk me through your CDC strategy for the source OLTP database."**
3. **"Estimate the storage cost for 2 years of event detail at 500M/day."**

---

## 6. Practice Exercise: Add Direct Messaging (DM / Chat)

The platform adds direct messaging between users.

### Part A — Extend the Model (15 min)

Identify **2 new fact tables** and **1 new dimension**. Define grain, key columns, and data types. Draw additions to the diagram.

### Part B — Asymmetric Relationships (10 min)

A DM conversation has two participants. Model it so you can query:
- "All messages between user A and user B" (order doesn't matter)
- "Message count per user per day"

### Part C — Senior Extension (10 min)

Extend for **group chats** (3+ participants). Consider:
- Group membership changes over time
- Per-participant read/unread tracking
- Encryption type as a dimension attribute

---

### Suggested Solution

**Part A — New Tables**

| Table | Type | Grain | Key Columns |
|---|---|---|---|
| `fact_dm_messages` | Fact | 1 row per message | `message_key`(PK), `conversation_key`(FK), `sender_key`, `recipient_key`, `message_date_key`, `sent_timestamp`, `message_text` |
| `fact_dm_conversations` | Fact | 1 row per conversation pair | `conversation_key`(PK), `user_a_key`, `user_b_key`, `created_at`, `last_message_at` |
| `dim_dm_media` | Dimension | 1 row per attachment | `media_key`(PK), `file_type`, `file_size_bytes`, `duration_seconds` |

```
┌──────────────────────┐     ┌──────────────────────────┐
│ fact_dm_messages     │     │ fact_dm_conversations    │
│ message_key (PK)     │◄────│ conversation_key (PK)    │
│ conversation_key(FK) │     │ user_a_key (FK→dim_user) │
│ sender_key  (FK)     │     │ user_b_key  (FK→dim_user)│
│ recipient_key (FK)   │     │ created_at               │
│ message_date_key(FK) │     │ last_message_at          │
│ sent_timestamp       │     └──────────────────────────┘
│ message_text         │
│ media_key(FK)        │
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│ dim_dm_media         │
│ media_key (PK)       │
│ file_type            │
│ file_size_bytes      │
└──────────────────────┘
```

**Part B — Symmetric Pair Normalisation**

```sql
-- Normalise user pair: always store smaller ID first
INSERT INTO fact_dm_conversations
SELECT NEXTVAL('conv_seq'),
       LEAST(user_a, user_b),
       GREATEST(user_a, user_b),
       NOW(), NULL;

-- Query: messages between any two users (order-agnostic)
SELECT * FROM fact_dm_messages
WHERE conversation_key IN (
    SELECT conversation_key FROM fact_dm_conversations
    WHERE user_a_key = LEAST(123, 456)
      AND user_b_key = GREATEST(123, 456)
);
```

**Part C — Group Chat Extension**

| New Table | Grain | Notes |
|---|---|---|
| `dim_group_chat` | 1 row per group | `group_key`, `created_at`, `creator_key`, `encryption_type` |
| `fact_group_membership` | 1 row per member | `group_key`, `user_key`, `joined_at`, `left_at` (SCD-2) |
| `fact_group_messages` | 1 row per message | `message_key`, `group_key`, `sender_key`, `sent_timestamp` |
| `fact_message_read_status` | 1 row per msg per user | `message_key`, `user_key`, `read_at` (NULL = unread) |

---

## 7. Key Takeaways

| Concept | Interview-Ready Answer |
|---|---|
| Start with requirements | Never draw tables before understanding KPIs, volume, latency |
| Grain is king | Every fact table has ONE grain — state it first |
| SCD-2 for user dims | Daily snapshot + slow-change columns (display_name, bio) |
| Dedup at load | Natural-key deduplication before surrogate key assignment |
| Bridge tables for arrays | Hashtags, mentions, media — explode to relational rows |
| Partition for scale | Time-based partitioning on the largest fact is non-negotiable |

> **Next:** Lesson 3.5 — Star Schema vs. Data Vault: Choosing Your Modelling Approach.

*© Euron Code — Data Modelling Interview Course, Module 3.*

---

### Related Lessons

- [Creating High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md)
- [Practice: E-Commerce Platform](./15-practice-ecommerce-platform.md)
- [Practice: Video Streaming Service](./17-practice-video-streaming-service.md)
- [Analyzing Query Patterns](./08-analyzing-query-patterns.md)
