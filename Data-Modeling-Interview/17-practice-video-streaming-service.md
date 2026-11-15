# Module 3: High-Level Model Diagrams
## Lesson 3.5 — Practice: Video Streaming Service

> **"Design a data warehouse for a video streaming platform like Netflix."**

This practice exercise walks you through a complete interview-style data modeling scenario. You'll gather requirements, sketch a high-level model, define tables, and discuss advanced patterns.

---

### 1. Requirements Gathering

| Category | Detail |
|----------|--------|
| **Business Goals** | Track viewing behavior, optimize content investment, reduce churn, personalize recommendations |
| **Key Metrics** | Hours watched, completion rate, subscriber churn, content ROI, search-to-play ratio |
| **Volume** | 100M sessions/month, 200M users, 15K+ titles |
| **Latency** | Daily for content analytics; near-real-time (minutes) for personalization |
| **Retention** | 5 years for churn analysis; 3 years for detailed event data |
| **Regulatory** | GDPR/CCPA compliance for user data; content rights expiry tracking |

#### Interviewer Questions You'll Face
1. *How would you decide which metrics to model at the session level versus the aggregate level?*
2. *What data sources would you expect to integrate (e.g., web, mobile app, smart TVs)?*
3. *How do you handle time zones and session boundaries for users across different regions?*

#### Top-Up Questions (Senior-Level Probes)
1. *If the personalization engine needs sub-minute latency but the warehouse is daily, what architecture would you propose?*
2. *How would you model "binge sessions" where a user watches multiple episodes consecutively?*
3. *What's your strategy for backfilling data when a new metric is requested retroactively?*

---

### 2. High-Level Model Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       VIDEO STREAMING DATA WAREHOUSE                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │  dim_date        │    │  dim_user        │    │  dim_content     │  │
│  │                  │    │  - user_sk (PK)  │    │  - content_sk(PK)│  │
│  │  - date_sk (PK)  │    │  - user_id       │    │  - content_id    │  │
│  │  - date_key      │    │  - signup_date   │    │  - title         │  │
│  │  - day_of_week   │    │  - region         │    │  - type          │  │
│  │  - month, year   │    │  - tier           │    │  - release_date  │  │
│  │  - is_weekend    │    │  - churn_flag    │    │  - runtime_min   │  │
│  └────────┬─────────┘    └────────┬─────────┘    └────────┬──────────┘  │
│           │                       │                       │             │
│           │ 1:M                   │ 1:M                   │ 1:M         │
│           ▼                       ▼                       ▼             │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │ fact_viewing_    │    │ fact_subscription│    │ fact_content_    │  │
│  │ sessions         │    │ events           │    │ search           │  │
│  │                  │    │                  │    │                  │  │
│  │ Grain: one row   │    │ Grain: one row   │    │ Grain: one row   │  │
│ │ per session per   │    │ per subscription │    │ per search event │  │
│ │ episode/title     │    │ state change     │    │                  │  │
│ │                  │    │                  │    │ - search_sk (PK) │  │
│ │ - session_sk (PK)│    │ - sub_sk (PK)    │    │ - date_sk (FK)   │  │
│ │ - date_sk (FK)   │    │ - date_sk (FK)   │    │ - user_sk (FK)   │  │
│ │ - user_sk (FK)   │    │ - user_sk (FK)   │    │ - search_term    │  │
│ │ - content_sk(FK) │    │ - prev_tier      │    │ - results_count  │  │
│ │ - device_sk (FK) │    │ - new_tier       │    │ - click_position │  │
│ │ - country_sk(FK) │    │ - amount         │    │ - played_flag    │  │
│ │ - start_time     │    │ - event_type     │    │ - play_latency   │  │
│ │ - pause_durations│    │ - cancel_reason  │    │                  │  │
│ │ - completion_%   │    │ - effective_date │    └──────────────────┘  │
│ │ - completed_flag │    │                  │                          │
│ │ - binge_session_ │    └──────────────────┘    ┌──────────────────┐  │
│ │   id             │                            │  dim_genre       │  │
│ └──────────────────┘                            │  - genre_sk (PK) │  │
│                                                  │  - genre_name    │  │
│  ┌──────────────────┐    ┌──────────────────┐    │  - parent_sk    │  │
│  │  dim_device      │    │  dim_country     │    └────────┬─────────┘  │
│  │                  │    │                  │             │ 1:M        │
│  │  - device_sk(PK) │    │ - country_sk(PK)│             ▼            │
│  │  - device_type   │    │ - country_code  │  ┌──────────────────┐  │
│  │  - os, version   │    │ - country_name  │  │  (link table for  │  │
│  │  - platform      │    │ - currency      │  │  content→genre)   │  │
│  │  - manufacturer  │    │ - timezone      │  └──────────────────┘  │
│  └──────────────────┘    └──────────────────┘                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Grain Annotations:**
- `fact_viewing_sessions`: One row per viewing session (user + content + start time). Updated as session progresses (accumulating snapshot).
- `fact_subscription_events`: One row per subscription state change (sign-up, upgrade, downgrade, cancellation).
- `fact_content_search`: One row per search event. Includes whether the user clicked and played a result.

**Key Relationships:**
- Every fact table links to `dim_date` for time-series analysis.
- `dim_genre` uses a many-to-many link table with `dim_content` (each title can have multiple genres).
- `dim_country` supports geo-based content rights and localization analysis.

---

### 3. Detailed Table Definitions

#### `dim_date`
| Column | Data Type | Notes |
|--------|-----------|-------|
| date_sk | INT (PK) | Surrogate key |
| date_key | DATE | Calendar date |
| day_of_week | VARCHAR(10) | 'Monday'...'Sunday' |
| is_weekend | BOOLEAN | |
| month | INT | 1–12 |
| quarter | INT | 1–4 |
| year | INT | |
| is_holiday | BOOLEAN | Regional holidays flag |

#### `dim_user`
| Column | Data Type | Notes |
|--------|-----------|-------|
| user_sk | INT (PK) | Surrogate key |
| user_id | VARCHAR(64) | Natural key |
| signup_date | DATE | |
| current_tier | VARCHAR(20) | 'Basic', 'Standard', 'Premium' |
| country_sk | INT (FK) | |
| churn_flag | BOOLEAN | Active/inactive |
| effective_start | DATE | SCD Type 2 |
| effective_end | DATE | NULL = current |

#### `dim_content` (SCD Type 2 — metadata changes)
| Column | Data Type | Notes |
|--------|-----------|-------|
| content_sk | INT (PK) | Surrogate key |
| content_id | VARCHAR(64) | Natural key |
| title | VARCHAR(500) | |
| content_type | VARCHAR(20) | 'Movie', 'Series', 'Documentary', 'Short' |
| release_date | DATE | |
| runtime_minutes | INT | |
| language | VARCHAR(10) | Primary audio language |
| rating | VARCHAR(5) | 'PG-13', 'R', etc. |
| is_original | BOOLEAN | Platform-produced content |
| effective_start | DATE | SCD Type 2 |
| effective_end | DATE | NULL = current |

> **SCD Strategy for `dim_content`:** Title metadata (runtime, rating, language availability) changes infrequently. Use **SCD Type 2** to preserve history: when metadata changes, close the current row (set `effective_end`) and insert a new row. This ensures we can answer "what metadata did users see when they watched this in 2022?"

#### `fact_viewing_sessions` (Accumulating Snapshot)
| Column | Data Type | Notes |
|--------|-----------|-------|
| session_sk | BIGINT (PK) | Surrogate key |
| date_sk | INT (FK) | Date session started |
| user_sk | INT (FK) | |
| content_sk | INT (FK) | |
| device_sk | INT (FK) | |
| country_sk | INT (FK) | |
| start_time | TIMESTAMP | |
| first_pause_time | TIMESTAMP | NULL if not paused |
| last_pause_time | TIMESTAMP | Updated on each pause |
| completion_time | TIMESTAMP | NULL until session ends |
| watch_duration_sec | INT | Updated incrementally |
| completion_pct | DECIMAL(5,2) | 0.00–100.00 |
| completed_flag | BOOLEAN | TRUE = 100% watched |
| binge_session_id | VARCHAR(64) | Groups consecutive episodes |
| row_status | VARCHAR(10) | 'IN_PROGRESS', 'COMPLETED', 'DROPPED' |

#### `fact_subscription_events`
| Column | Data Type | Notes |
|--------|-----------|-------|
| sub_sk | BIGINT (PK) | Surrogate key |
| date_sk | INT (FK) | Date of event |
| user_sk | INT (FK) | |
| event_type | VARCHAR(20) | 'SIGNUP', 'UPGRADE', 'DOWNGRADE', 'CANCEL', 'REACTIVATE' |
| previous_tier | VARCHAR(20) | NULL for new signups |
| new_tier | VARCHAR(20) | NULL for cancellations |
| monthly_amount | DECIMAL(8,2) | |
| cancel_reason | VARCHAR(200) | NULL unless event_type='CANCEL' |
| effective_date | DATE | |

#### `fact_content_search`
| Column | Data Type | Notes |
|--------|-----------|-------|
| search_sk | BIGINT (PK) | Surrogate key |
| date_sk | INT (FK) | |
| user_sk | INT (FK) | |
| device_sk | INT (FK) | |
| search_term | VARCHAR(500) | |
| results_count | INT | |
| click_position | INT | Position clicked; NULL = no click |
| played_content_sk | INT (FK) | Content played from search; NULL = no play |
| play_latency_sec | DECIMAL(5,2) | Time from search to play |

#### `dim_genre`
| Column | Data Type | Notes |
|--------|-----------|-------|
| genre_sk | INT (PK) | Surrogate key |
| genre_name | VARCHAR(100) | 'Action', 'Drama', 'Sci-Fi' |
| parent_genre_sk | INT (FK) | Hierarchical (nullable) |

#### `link_content_genre` (Many-to-Many)
| Column | Data Type | Notes |
|--------|-----------|-------|
| content_sk | INT (FK) | |
| genre_sk | INT (FK) | |

---

### 4. Discussion Points

#### Accumulating Snapshot for Viewing Sessions

An **accumulating snapshot fact table** captures a process that unfolds over time. A viewing session has multiple milestones:

```
Time →  |---start---|---pause---|---resume---|---pause---|---complete---|
Event:    START         PAUSE_1       RESUME_1      PAUSE_2      COMPLETED
Row updates each milestone until the session is complete.
```

- **Start time** is populated when the session begins.
- **Pause times** are updated each time the user pauses (we store the last pause, and optionally track total pause duration).
- **Completion time** is NULL until the user finishes or explicitly stops.
- **Completion %** updates incrementally as the user watches.

This pattern lets us answer: *"What's the average time to first pause?"*, *"What % of sessions start but never finish?"*, and *"How does completion rate vary by content type?"*

#### Handling Partial Views vs. Completions

- A session is **completed** when `completion_pct >= 100` and `row_status = 'COMPLETED'`.
- A session is **dropped** when the user stops before 100% and doesn't resume within 24 hours (`row_status = 'DROPPED'`).
- **Partial views** (`row_status = 'IN_PROGRESS'` but `completion_pct > 0`) feed into "watch momentum" metrics for personalization.

#### Tracking Binge Behavior

- A **binge session** groups consecutive episodes of the same series watched within a short gap (e.g., < 30 minutes between episodes).
- The `binge_session_id` links episodes in `fact_viewing_sessions`.
- Aggregation: `COUNT(DISTINCT binge_session_id)` per user per month tracks binge frequency.
- This supports business questions like: *"Do users who binge have lower churn?"*

#### Interviewer Questions You'll Face
1. *Why an accumulating snapshot instead of separate event tables for start, pause, and complete?*
2. *How do you handle a user resuming a session on a different device?*
3. *What's the trade-off between row updates in an accumulating snapshot and append-only event logs?*

#### Top-Up Questions (Senior-Level Probes)
1. *How would you design this model to support A/B testing of recommendation algorithms?*
2. *If a content provider demands privacy-isolated analytics for their shows, how would you structure that?*
3. *How do you ensure data quality when ingesting from 10+ client SDKs with different event schemas?*

---

### 5. Practice Exercise

**Scenario:** The platform now launches **live streaming** (sports events, concerts, awards shows) alongside on-demand content. Extend the model to support:

| New Requirement | Detail |
|-----------------|--------|
| Live events | Scheduled start/end times, multiple simultaneous streams |
| Live chat | User messages during live streams (moderation analytics) |
| Concurrent viewers | Real-time viewer count for each live stream |
| DVR/replay | Users can pause, rewind, and replay live content afterward |
| Regional blackout | Some live content is geo-restricted |

**Your Task:**

1. **Add or modify fact/dimension tables** to capture live streaming data. Sketch the extensions to the diagram.
2. **Define grain** for any new fact tables. How does a "live viewing session" differ from an on-demand session?
3. **Identify new SCD needs.** Does `dim_content` need changes for live events?
4. **Propose 3–5 new metrics** unique to live streaming (e.g., peak concurrent viewers, chat engagement rate).
5. **Draft sample SQL** for a query: *"For the last 30 days, show the top 10 live events by average concurrent viewers, broken down by country."*

> **Tip:** Consider whether live streaming requires a separate fact table (`fact_live_viewing`) or if `fact_viewing_sessions` can be extended with a `content_delivery_mode` column ('ON_DEMAND' vs. 'LIVE'). There are valid arguments for both approaches — be ready to discuss trade-offs.

---

*Ready to tackle the exercise? Sketch your model first, then refine. The best answers show your reasoning, not just a diagram.*

---

### Related Lessons

- [Creating High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md)
- [Practice: Social Media Analytics](./16-practice-social-media-analytics.md)
- [Practice: Ridesharing Platform](./18-practice-ridesharing-platform.md)
- [Data Volume and Scalability Considerations](./10-data-volume-and-scalability-considerations.md)
