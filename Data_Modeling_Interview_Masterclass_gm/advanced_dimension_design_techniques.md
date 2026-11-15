# Lesson 23: Advanced Dimension Design (Junk, Mini, Outrigger, Role-Playing & Hierarchies)

> **Principal Mentor Note:** Standard Star Schemas handle clean `1:N` entities effortlessly. Real-world enterprise warehouses get messy: dozens of low-cardinality boolean flags cluttering fact tables, 500M-row dimensions with fast-changing attributes, self-referencing org charts, and dates playing 6 different roles on the same transaction. Mastering **Junk, Mini, Outrigger, Role-Playing, and Hierarchical Dimensions** is what allows you to model complex domains without sacrificing query speed or BI usability.

---

## 1. Quick-Reference Taxonomy of Advanced Dimension Patterns

| Pattern | Core Problem Solved | Structural Implementation | Scale & Query Benefit |
| :--- | :--- | :--- | :--- |
| **1. Role-Playing Dimension** | A single conformed dimension (`dim_date`, `dim_user`, `dim_h3_zone`) is referenced multiple times with different meanings in the same fact table. | One physical table (`dim_date`), multiple foreign keys on the fact (`ordered_date_sk`, `shipped_date_sk`, `delivered_date_sk`), aliased in SQL or semantic views. | Single source of truth for calendar/geo attributes; zero duplicate dimension maintenance. |
| **2. Junk Dimension** | 10–25 unrelated low-cardinality flags and indicators (`is_gift`, `is_express`, `payment_type`, `channel`) cluttering a wide fact table. | Combine Cartesian product (or observed combinations) of flags into a single compact `dim_order_flags` table keyed by one `INT` `order_flags_sk`. | Replaces 15 `VARCHAR`/`BOOLEAN` columns on a 20B-row fact table with a single 4-byte `INT` FK! |
| **3. Mini-Dimension (SCD Type 4)** | A huge dimension (`dim_customer`, 300M rows) has a few rapidly changing analytical attributes (`age_band`, `income_tier`, `rfm_segment`, `credit_band`). | Band/bucket the volatile attributes into a static lookup table `dim_customer_demographics` (~5,000 rows) and stamp `demographics_sk` directly onto the fact row at event time. | Eliminates SCD Type 2 row explosion on the 300M-row `dim_customer` while preserving exact point-in-time demographic history! |
| **4. Outrigger Dimension** | A secondary dimension attached to a primary dimension (a controlled 1-hop Snowflake), used *only* for strict security/compliance isolation or wildly different grain. | `dim_user` links to `dim_user_pii` (GDPR isolation) or `dim_country_macro_indicators` (annual GDP/inflation). | Isolates PII encryption/erasure without slowing down 99% of analytical queries. |
| **5. Bridge / Closure Table** | Multi-valued (`M:N`) relationships (e.g., multiple diagnoses per hospital visit) or variable-depth recursive hierarchies (Org chart, Category tree). | Intermediate table `(parent_sk, child_sk, depth, weight_factor)` between fact and dimension. | Solves variable-depth rollups and `M:N` attribution without fan-out double-counting. |

---

## 2. Deep Dive: Implementing Each Pattern with Production Examples

### Pattern A: Junk Dimensions (Cleaning Up "Flag Soup" on Fact Tables)
Suppose `fct_orders` has 12 operational flags: `is_first_order` (2), `is_gift_wrapped` (2), `is_subscription_renewal` (2), `order_channel` (`WEB`, `IOS`, `ANDROID`, `API` = 4), `delivery_speed` (`STANDARD`, `EXPRESS`, `SAME_DAY` = 3), `tax_exempt_status` (2).
* Total possible combinations: $2 \times 2 \times 2 \times 4 \times 3 \times 2 = 192\text{ rows}$!
* Instead of storing 12 columns on 10 billion rows of `fct_orders`, create `dim_order_flags` (192 rows total, completely broadcast-cached in worker RAM):

```sql
CREATE TABLE dim_order_flags (
    order_flags_sk          INT PRIMARY KEY,       -- Deterministic hash or integer ID (1..192)
    is_first_order          BOOLEAN NOT NULL,
    is_gift_wrapped         BOOLEAN NOT NULL,
    is_subscription_renewal BOOLEAN NOT NULL,
    order_channel           VARCHAR(16) NOT NULL,  -- WEB, IOS, ANDROID, API
    delivery_speed          VARCHAR(16) NOT NULL,  -- STANDARD, EXPRESS, SAME_DAY
    tax_exempt_status       VARCHAR(16) NOT NULL   -- TAXABLE, EXEMPT_RESALE, EXEMPT_GOV
);
```
> **Principal Caveat:** Only group **low-cardinality** flags (`2–10` distinct values each) into a Junk Dimension. If you include a high-cardinality string like `promo_code` (`50,000` values), the Cartesian product explodes to $192 \times 50,000 = 9.6\text{M rows}$—keep `promo_code` in its own `dim_promo`!

---

### Pattern B: Mini-Dimensions (Taming High-Churn Attributes on Massive Dimensions)
In Fintech (Stripe/Robinhood) or E-Commerce (Amazon), `dim_customer` has `250M` rows, and attributes like `ltv_decile` (1–10), `churn_risk_band` (`LOW`, `MED`, `HIGH`), `account_tenure_bucket` (`0-30d`, `31-90d`, `91-365d`, `1yr+`), and `kyc_tier` (`TIER_1`, `TIER_2`, `TIER_3`) change constantly.

```sql
-- Static Mini-Dimension: 10 * 3 * 4 * 3 = 360 rows total! Never needs SCD2 updates!
CREATE TABLE dim_customer_segment_mini (
    customer_segment_sk     INT PRIMARY KEY,
    ltv_decile              TINYINT NOT NULL,      -- 1..10
    churn_risk_band         VARCHAR(16) NOT NULL,  -- LOW, MEDIUM, HIGH
    account_tenure_bucket   VARCHAR(24) NOT NULL,  -- 0_30D, 31_90D, 91_365D, 365D_PLUS
    kyc_tier                VARCHAR(16) NOT NULL   -- TIER_1, TIER_2, TIER_3
);

-- fct_transactions stamps BOTH customer_sk (current/slowly changing) AND customer_segment_sk (volatile)!
-- Querying "Revenue by Churn Risk Band at the time of transaction" requires ZERO range joins!
```

---

### Pattern C: Variable-Depth Hierarchies (Flattened vs. Closure Bridge Table)
When modeling an **Enterprise Sales Org Hierarchy** (`CEO -> SVP -> VP -> Director -> Manager -> Account Executive`) or **Multi-Level Product Taxonomy**:

#### Option 1: Fixed-Depth Flattened Hierarchy (Best when depth is bounded, e.g., $\le 6$ levels)
Store all levels directly on `dim_sales_rep`:
* `rep_sk`, `rep_name`, `manager_name`, `director_name`, `vp_name`, `svp_name`.
* *Trick for ragged hierarchies (where a Rep reports directly to a VP, skipping Director):* Repeat the VP's name down into the `director_name` column (`ragged-to-balanced` padding) so drill-down charts in Tableau/Looker never show ugly `NULL` buckets!

#### Option 2: Hierarchy Bridge / Closure Table (Best when depth is unbounded, e.g., 15-level corporate subsidiaries)
Store **every ancestor-descendant path** in `bridge_org_hierarchy`:

```sql
CREATE TABLE bridge_org_hierarchy (
    ancestor_rep_sk         BIGINT NOT NULL,       -- Top/intermediate manager
    descendant_rep_sk       BIGINT NOT NULL,       -- Reporting rep (includes self: ancestor = descendant, depth = 0)
    depth_levels            INT NOT NULL,          -- 0 = self, 1 = direct report, 2 = skip-level...
    is_direct_report        BOOLEAN NOT NULL,      -- TRUE when depth_levels = 1
    is_leaf_node            BOOLEAN NOT NULL,
    PRIMARY KEY (ancestor_rep_sk, descendant_rep_sk)
);

-- Instant Subtree Rollup Query (No slow Recursive CTEs!):
-- "Total Q4 bookings rolled up under VP 'Sarah Jenkins' (rep_sk = 5001) across ALL levels below her"
SELECT
    mgr.rep_name                                   AS vp_name,
    SUM(b.booking_amount_usd)                      AS total_org_bookings_usd
FROM fct_bookings b
JOIN bridge_org_hierarchy h
  ON b.sales_rep_sk = h.descendant_rep_sk          -- Joins all descendants below Sarah!
JOIN dim_sales_rep mgr
  ON h.ancestor_rep_sk = mgr.rep_sk
WHERE h.ancestor_rep_sk = 5001
GROUP BY 1;
```

---

## 3. Real Interview Questions & Model Answers

### Q1: "In a healthcare data warehouse, a patient encounter (`fct_encounters`) has 12 boolean clinical indicators (e.g., `is_emergency`, `is_readmission_30d`, `is_telehealth`, `has_insurance_preauth`, `is_icu_stay`), AND each encounter can be associated with 1 to 15 diagnosis codes (`ICD-10`). How do you model both cleanly without bloating the fact table or double-counting claim amounts?"

**Principal-Level Answer:**
> *"These are two distinct dimensional modeling challenges that require two different patterns:*
> 1. ***For the 12 boolean clinical indicators:** I combine them into a **Junk Dimension** (`dim_encounter_indicators`, at most $2^{12} = 4,096$ rows) keyed by a single 4-byte `encounter_indicators_sk` on `fct_encounters`. This eliminates 12 boolean columns from the multi-billion-row fact table while keeping every slice-and-dice filter one fast broadcast join away.*
> 2. ***For the 1 to 15 diagnosis codes:** This is a true many-to-many multivalued dimension. I create a **Bridge Table** (`bridge_encounter_diagnoses`) keyed by `(encounter_id, diagnosis_sk)` with `diagnosis_rank` (`1 = Primary`, `2+ = Secondary`) and `attribution_weight` (summing to `1.0` per encounter, e.g., `1.0 / diagnosis_count` or clinically weighted). Additionally, I stamp `primary_diagnosis_sk` directly on `fct_encounters` so executive dashboards reporting by primary diagnosis run a simple `1:1` star join with zero risk of fan-out double-counting."*

---

## 4. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "What is the biggest danger when analysts query a Closure / Hierarchy Bridge Table (`bridge_org_hierarchy`), and how do you guard against it?"**
   * **Expected Insight:** If an analyst queries `bridge_org_hierarchy` **without filtering `WHERE ancestor_rep_sk = <specific_manager>`** (e.g., grouping across *all* ancestors simultaneously), every deal gets counted once for the Rep (`depth=0`), once for their Manager (`depth=1`), once for their Director (`depth=2`), and once for the VP (`depth=3`)—inflating company revenue by 4x! Guard against this by exposing the bridge table through governed semantic views or requiring an explicit `ancestor_rep_sk` filter in BI tools.
2. **Top-Up 2: "When does an Outrigger Dimension become an anti-pattern (over-snowflaking)?"**
   * **Expected Insight:** When you normalize lookup tables purely to save disk space—for example, splitting `dim_product -> dim_brand -> dim_manufacturer -> dim_country` into 4 chained tables. That forces 4 sequential joins on every query and complicates BI metadata for negligible storage savings. Reserve Outriggers strictly for **PII security/GDPR isolation** (`dim_user_pii`) or **drastically slower-changing external reference data** (`dim_currency_fx_monthly`).

---

## 5. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Consolidates 10+ boolean/status flags into a compact Junk Dimension** to shrink fact row width. | Leaves 25 sparse `is_*` boolean/string flags directly on a 50B-row fact table. |
| **Extracts fast-changing attributes on massive dimensions into a Mini-Dimension (Type 4)** instead of triggering SCD2 explosion. | Tracks daily-changing risk scores or follower tiers as SCD Type 2 rows on a 300M-row user dimension. |
| **Uses Closure / Bridge tables (`ancestor_sk`, `descendant_sk`, `depth`)** for variable-depth hierarchies instead of slow recursive CTEs. | Forces BI tools to execute `WITH RECURSIVE` self-joins on `dim_employee` at dashboard query time. |