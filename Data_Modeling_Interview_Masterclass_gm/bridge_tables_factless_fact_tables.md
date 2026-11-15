# Lesson 27: Bridge Tables & Factless Fact Tables (Many-to-Many & Coverage)

> **Principal Mentor Note:** Two of the sharpest tools in a Principal Data Engineer's toolkit—and the ones most often botched by mid-level candidates—are **Factless Fact Tables** (recording events or eligibility coverage that carry *zero numeric measures*) and **Bridge Tables** (resolving many-to-many `M:N` relationships between facts and dimensions without corrupting `SUM()` aggregations).

## 1. Factless Fact Tables: Modeling Events & Coverage Without Measures

A **Factless Fact Table** consists *entirely of foreign keys* (plus optional degenerate dimensions and timestamps) with **no numeric measurement columns**. Every row simply records the **intersection of dimensions at a point in time**.

### Two Distinct Types of Factless Fact Tables

#### Type A: Event-Tracking Factless Facts (Something Happened)

Records discrete business occurrences where the count of rows *is* the metric:

* **Examples:**

  * Student class attendance (`fct_student_attendance(attendance_date_sk, student_sk, course_sk, professor_sk, classroom_sk)`).

  * User-to-User social graph follow/block events (`fct_user_follows(followed_date_sk, follower_user_sk, followee_user_sk)`).

  * Security badge swipe / door access logs (`fct_badge_swipes(swipe_date_sk, swipe_time_sk, employee_sk, building_sk, door_sk)`).

* **Interview Tip:** You can optionally add a constant `event_count = 1` column so BI tools like Tableau/PowerBI have an explicit additive measure to `SUM(event_count)` rather than relying on `COUNT(*)`.

#### Type B: Coverage / Eligibility Factless Facts (What *Could* Have Happened)

Records valid combinations or scheduled eligibility so you can detect **negative / non-events** (what *didn't* happen!):

* **Classic Retail Interview Scenario:** *"Which promoted products in our Thanksgiving circular had **ZERO sales** in Chicago stores yesterday?"*

  * Why `fct_sales` alone fails: If Product P-404 sold 0 units in Store S-12, there is **no row** in `fct_sales` for `(Nov 26, P-404, S-12)`! Filtering `WHERE units_sold = 0` returns nothing.

  * **Principal Solution:** Model a **Coverage Factless Fact** (`fct_promotion_coverage(promo_date_sk, promo_sk, product_sk, store_sk)`) loaded from the marketing campaign schedule, and `LEFT ANTI JOIN` (or `LEFT JOIN ... WHERE sales.product_sk IS NULL`) against `fct_sales`:

```
-- Find Promoted Products with ZERO Sales (Negative Event Detection via Coverage Fact)
SELECT
    d.calendar_date,
    s.store_name,
    pr.promo_name,
    p.product_name
FROM fct_promotion_coverage cov
JOIN dim_date d     ON cov.promo_date_sk = d.date_sk
JOIN dim_store s    ON cov.store_sk = s.store_sk
JOIN dim_promo pr   ON cov.promo_sk = pr.promo_sk
JOIN dim_product p  ON cov.product_sk = p.product_sk
LEFT JOIN fct_sales sal
  ON cov.promo_date_sk = sal.sale_date_sk
 AND cov.store_sk      = sal.store_sk
 AND cov.product_sk    = sal.product_sk
WHERE s.city_name = 'Chicago'
  AND d.calendar_date = DATE '2026-11-26'
  AND sal.product_sk IS NULL;                      -- Exposes exact zero-sale items!

```

## 2. Bridge Tables: Solving Many-to-Many (`M:N`) Dimensions Without Fan-Out

In a strict Star Schema, every fact row points to **exactly one** row per dimension (`N:1`). When a single fact row maps to **multiple dimension members simultaneously** (`M:N`), joining directly multiplies the fact table's additive measures (**Fan-Out Double-Counting**).

### Real-World `M:N` Scenarios

* **Fintech / Banking:** A joint checking account (`account_sk`) is owned by 2 or 3 co-owners (`customer_sk`).

* **E-Commerce / AdTech:** A single order conversion is attributed across 3 marketing touchpoints (`campaign_sk`) via multi-touch attribution.

* **Healthcare:** A single hospital claim (`claim_id`) has 4 diagnosis codes (`diagnosis_sk`).

* **Cloud / SaaS:** A single GitHub repository or AWS project is tagged with 6 team/cost-center labels.

### Production Pattern: Weighted Bridge Table with Group Key

```
-- ============================================================================
-- DIMENSION & BRIDGE FOR JOINT BANK ACCOUNTS (M:N Account <-> Customer)
-- ============================================================================
CREATE TABLE bridge_account_customer (
    account_group_sk        BIGINT NOT NULL,       -- Links directly to fct_account_transactions!
    customer_sk             BIGINT NOT NULL,       -- FK -> dim_customer
    ownership_role          VARCHAR(24) NOT NULL,  -- PRIMARY_OWNER, JOINT_OWNER, BENEFICIARY
    ownership_weight        DECIMAL(5,4) NOT NULL, -- Sums to 1.0000 per account_group_sk!
    is_primary_owner        BOOLEAN NOT NULL,
    PRIMARY KEY (account_group_sk, customer_sk)
);

-- ============================================================================
-- HOW TO QUERY WITHOUT DOUBLE-COUNTING (Two Explicit Modes)
-- ============================================================================
-- Mode 1: Weighted Financial Allocation (Preserves exact bank-wide Total Deposits!)
SELECT
    c.customer_segment,
    SUM(t.deposit_amount_usd * b.ownership_weight) AS weighted_deposits_usd
FROM fct_account_transactions t
JOIN bridge_account_customer b ON t.account_group_sk = b.account_group_sk
JOIN dim_customer c            ON b.customer_sk = c.customer_sk
GROUP BY 1;

-- Mode 2: Unweighted "Impact / Exposure" Reporting (Explicitly labeled as Non-Additive across customers)
-- e.g., "Total deposit volume across accounts where Jane Doe is an owner (primary or joint)"
SELECT
    c.customer_id,
    SUM(t.deposit_amount_usd)                      AS total_associated_account_deposits_usd
FROM fct_account_transactions t
JOIN bridge_account_customer b ON t.account_group_sk = b.account_group_sk
JOIN dim_customer c            ON b.customer_sk = c.customer_sk
WHERE c.customer_id = 'CUST-7788'
GROUP BY 1;

```

### Principal Design Tricks for Bridge Tables

1. **Always Include `is_primary_flag` + `primary_entity_sk` on the Fact Table:**

   * 80% of executive dashboards want a simple `1:1` view by Primary Account Owner or Primary Diagnosis without worrying about bridge table weights. Denormalizing `primary_customer_sk` onto the fact table makes those queries fast and foolproof, while `bridge_account_customer` handles full `M:N` weighted attribution.

2. **Group-Key Deduplication (`account_group_sk`):**

   * Instead of keying the bridge table by `transaction_id` (which would create billions of bridge rows!), key it by **`account_group_sk`** (a hash of the sorted set of `(customer_sk, ownership_role)` pairs). Even if an account executes 10,000 transactions, the bridge table stores those 2 co-owner rows **once**!

## 3. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "In modern cloud warehouses (Snowflake, BigQuery, Databricks), why not just store an `ARRAY<BIGINT>` of `customer_sk`s directly on the fact table instead of a separate Bridge Table?"**

   * **Expected Insight:** Storing `ARRAY<BIGINT>` (or `ARRAY<STRUCT<customer_sk, weight>>`) on the fact table is a valid modern **Nested / Semi-Structured Denormalization** trade-off when the array is small (`< 10` elements) and read-only. However, if co-owners or tags change over time, updating an array inside 5 billion fact rows is vastly more expensive than updating 2 rows in a compact `bridge_account_customer` table—and most standard BI tools (Tableau, PowerBI, Looker) generate much cleaner SQL against relational bridge tables than `LATERAL FLATTEN` / `UNNEST` arrays.

2. **Top-Up 2: "How do you prevent an analyst from joining a Bridge Table without filtering `is_primary_owner = TRUE` or multiplying by `ownership_weight`, and accidentally inflating company revenue?"**

   * **Expected Insight:** Never expose raw un-weighted bridge joins in self-service BI explores without guardrails. Expose **two governed semantic views/explores**: (1) `fct_transactions_primary_owner` (1:1 join via `primary_customer_sk`, safe for standard `SUM()`), and (2) `fct_transactions_weighted_attribution` (pre-multiplies measures by `ownership_weight` inside the view definition).

## 4. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) | 
| ----- | ----- | 
| **Introduces Coverage Factless Facts** to answer "negative event" questions (promoted items with 0 sales, scheduled drivers who didn't log in). | Claims "you can't query things that didn't happen" or runs expensive cross-joins across all products $\times$ stores $\times$ dates. | 
| **Keys Bridge Tables by Group Hash (`group_sk`)** rather than duplicating bridge rows per transaction, and includes `weighting_factor` + `is_primary` guardrails. | Joins a `1:N` bridge table directly to a financial fact table without weights, silently doubling or tripling revenue totals. | 
