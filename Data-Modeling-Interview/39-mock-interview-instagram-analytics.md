# Lesson 39 — Mock Interview: Instagram-Style Social Platform

**Level:** Senior / Staff / Principal Data Engineer
**Duration:** 60-90 minutes
**Case type:** End-to-end source-to-warehouse data modeling

## How to Use This

This mock interview covers a high-volume social platform like Instagram, focusing on:
- Multi-grain event modeling (likes, comments, shares, story views)
- Self-referencing dimensions (user-to-user follows)
- Bridge tables for many-to-many relationships (hashtags, audiences)
- Aggregate tables for timeline-style queries at scale

---

## Scenario

You are the lead data engineer for a social media platform with 500M+ monthly active users. Users post photos/videos, follow each other, and interact through likes, comments, shares, and story views.

### Business Requirements

1. **Track all user engagement events** — likes, comments, shares, story views, profile views
2. **Influencer analytics** — follower counts, engagement rates, content performance
3. **Content lifecycle tracking** — post creation, edits, deletion, visibility changes
4. **Hashtag & trending analytics** — which topics trend, by region and time
5. **User retention & cohort analysis** — sign-up cohorts, DAU/MAU, churn
6. **Story/ephemeral content** — expires in 24 hours, needs special retention handling

### Key Entities

- **Users** — accounts with profiles (SCD Type 2 for profile changes)
- **Posts** — photos/videos with metadata
- **Follows** — user-to-user relationship (self-referencing)
- **Hashtags** — many-to-many with posts
- **Engagement Events** — likes, comments, shares, views
- **Stories** — ephemeral content that expires
- **DM Conversations** — direct messages (private, limited analytics)

---

## Phase 1: Requirements Clarification (10 min)

### Questions to Ask

**Business / Scope:**
- What metrics drive decisions? (engagement rate, reach, retention, influencer growth?)
- What is the daily event volume? (billions of events/day)
- Do we need real-time dashboards or is T+1 acceptable?
- What is the retention period for story data vs. post data?
- Is DM data included in analytics or excluded for privacy?

**Technical:**
- What is the source system? (Kafka event streams, Postgres OLTP?)
- Target warehouse? (BigQuery, Snowflake, Databricks?)
- Are there compliance requirements? (GDPR, CCPA for user data)
- What is the required query latency for dashboards?

---

## Phase 2: Source-to-Warehouse Modeling (20 min)

### Dimension Tables

| Dimension | Grain | Key Columns | SCD Type |
|-----------|-------|-------------|----------|
| `dim_user` | One row per user version | `user_id`, `username`, `display_name`, `is_verified`, `country`, `signup_date`, `account_type` | Type 2 (profile changes, verification status) |
| `dim_post` | One row per post version | `post_id`, `user_key`, `post_type`, `caption`, `visibility`, `created_at` | Type 2 (edits, visibility changes) |
| `dim_hashtag` | One row per hashtag | `hashtag_id`, `hashtag_name`, `category` | Type 1 |
| `dim_story` | One row per story | `story_id`, `user_key`, `created_at`, `expires_at` | Type 1 |
| `dim_date` | Standard date dimension | `date_key`, `day`, `month`, `quarter`, `year` | — |
| `dim_hour` | Hour-level time dimension | `hour_key`, `date_key`, `hour_of_day` | — |

### Fact Tables

| Fact Table | Grain | Key Columns |
|------------|-------|-------------|
| `fct_engagement_events` | One row per engagement event | `event_id`, `post_key`, `user_key`, `date_key`, `hour_key`, `event_type`, `event_count` |
| `fct_follows` | One row per follow relationship | `follower_key`, `followee_key`, `date_key`, `follow_status`, `followed_at` |
| `fct_post_hashtags` | One row per post-hashtag link | `post_key`, `hashtag_key`, `date_key` |
| `fct_story_views` | One row per story view | `story_key`, `viewer_key`, `date_key`, `viewed_at` |
| `fct_user_daily_activity` | One row per user per day | `user_key`, `date_key`, `posts_created`, `stories_created`, `likes_given`, `comments_given`, `profile_views_received` |

### Key Design Decisions

**1. Self-Referencing Follows:**
- `fct_follows` stores `follower_key` and `followee_key` — both reference `dim_user`
- `follow_status` handles active, unfollowed, blocked states
- Follower count = `COUNT(DISTINCT follower_key)` WHERE `follow_status = 'active'`

**2. Engagement as Heavy Hitter:**
- `fct_engagement_events` is the largest table (billions of rows/day)
- `event_type` is a **degenerate dimension** (no separate dimension needed for like/comment/share)
- Consider aggregating to `fct_engagement_hourly` (post + hour + event_type) for dashboards

**3. Post-Hashtag Bridge:**
- `fct_post_hashtags` is effectively a **bridge table** — one post can have many hashtags, one hashtag can be on many posts
- Enables trending hashtag queries: `COUNT(DISTINCT post_key) GROUP BY hashtag_key`

**4. User Daily Activity as Aggregate Fact:**
- `fct_user_daily_activity` is a **periodic fact table** — pre-aggregated at user-day grain
- Avoids scanning billions of raw events for cohort/retention queries

---

## Phase 3: Advanced Questions (15 min)

### Q: How do you model the follower network for influencer analysis?

**Answer (Principal Level):**
- `fct_follows` provides the raw network; derive influencer metrics through aggregations:
  - `follower_count` = `COUNT(DISTINCT follower_key)` WHERE `followee_key = X`
  - `following_count` = `COUNT(DISTINCT followee_key)` WHERE `follower_key = X`
  - `engagement_rate` = `total_engagement_received / follower_count`
- For real-time follower counts, materialize a lightweight aggregate: `agg_user_follower_counts(user_key, date_key, follower_count, following_count)`

### Q: How do you handle hashtag trending across regions?

**Answer:**
- `fct_post_hashtags` links posts to hashtags; join `dim_user.country` for region
- Aggregate at `hashtag + region + date` grain:
```
agg_hashtag_trending
- hashtag_key, country_key, date_key
- post_count, unique_user_count, total_engagement
```
- Rank by `post_count DESC` within each region per day

### Q: Stories expire in 24 hours — how does this affect the model?

**Answer:**
- `dim_story` stores `expires_at`; the fact tables retain the events
- **Retention policy:** delete raw story view events after a defined period (e.g., 30 days) since stories are ephemeral
- Aggregate story metrics into `agg_user_daily_activity` before deleting raw events
- Use `is_expired` flag in `dim_story` if business needs historical awareness

### Q: How do you handle GDPR "right to be forgotten" for user data?

**Answer:**
- **Tiered deletion:** anonymize `dim_user` (replace PII with `user_id` + 'deleted')
- **Cascade delete:** remove `fct_engagement_events` where `user_key` is deleted (or anonymize)
- **Retention boundaries:** set `expire_snapshots` and `CLUSTER BY` with retention policies
- **Audit trail:** keep a separate `user_deletion_log` for compliance without PII

---

## Phase 4: Architecture & Optimization (10 min)

### Medallion Architecture

| Layer | Purpose | Key Operations |
|-------|---------|----------------|
| **Bronze** | Raw event streams from Kafka | Append-only, `MERGE` by `event_id` |
| **Silver** | Conformed dimensions + events | SCD Type 2 for users/posts, bridge for hashtags |
| **Gold** | Aggregates for dashboards | `agg_user_daily_activity`, `agg_hashtag_trending`, `agg_engagement_hourly` |

### Performance

- **Partition** `fct_engagement_events` by `date_key` with hourly sub-partitioning
- **Cluster** by `post_key` for post-scoped queries, `user_key` for user-scoped queries
- **Materialize** gold aggregates with incremental refresh every 15-60 min
- **Bloom filters** on `event_id` for deduplication

### Scale Considerations

- At 500M MAU with ~10 events/user/day = 5B events/day
- `fct_engagement_events` grows by ~5B rows/day → partition aggressively
- Pre-aggregate to hourly/daily grain for any query that doesn't need row-level detail
- Bound merge operations to active (non-expired) data only

---

## Evaluation Rubric

| Level | What They Show |
|-------|---------------|
| **Senior** | Correct star schema, identifies engagement as largest table, uses degenerate dimension for event_type |
| **Staff** | Medallion layers, self-referencing follows, bridge for hashtags, periodic fact for daily activity, GDPR handling |
| **Principal** | Quantifies scale (5B events/day), explains partition/cluster strategy, tiered deletion for GDPR, aggregate refresh cadence |

---

## Related Lessons

- [Lesson 10 — Data Volume and Scalability](./10-data-volume-and-scalability-considerations.md)
- [Lesson 21 — Slowly Changing Dimensions](./21-slowly-changing-dimensions-types-1-4.md)
- [Lesson 22 — Degenerate and Junk Dimensions](./22-degenerate-and-junk-dimensions.md) — event_type as degenerate
- [Lesson 24 — Fact Table Types](./24-fact-table-types.md) — periodic fact for daily activity
- [Lesson 32 — Bridge Tables and Factless Facts](./32-bridge-tables-and-factless-facts.md)
- [Lesson 41 — Principal-Level Cheat Sheet](./41-principal-level-cheat-sheet.md)
