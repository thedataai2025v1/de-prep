# Lesson 22: Slowly Changing Dimensions (Types 0–7, Implementation & Merge Patterns)

> **Principal Mentor Note:** Slowly Changing Dimensions (SCDs) are guaranteed to come up in every analytical data modeling interview. Senior candidates recite textbook definitions of Type 1 and Type 2. Principal candidates go much further: they **mix SCD types column-by-column within the same table (Hybrid Type 6/7)**, explain how to prevent **SCD Type 2 row explosion** on high-churn dimensions, handle **out-of-order / retroactive attribute changes**, and write bug-free **idempotent `MERGE` SQL** that avoids overlapping `[valid_from, valid_to)` intervals.

---

## 1. Complete Reference of SCD Types (0 through 7)

| SCD Type | Mechanism | Historical Truth Preserved? | Typical Use Case | Trade-Offs |
| :--- | :--- | :--- | :--- | :--- |
| **Type 0 (Retain Original)** | Never updated after initial insert. | Original value only. | `signup_date`, `first_acquisition_channel`, `birth_date`, `original_credit_score`. | Zero write overhead; ideal for immutable cohort attributes. |
| **Type 1 (Overwrite)** | Update attribute in-place (`UPDATE`). | **No** (restates history to current value). | Fixing typos (`custmer_name`), `email_address`, `last_login_ts`, `current_account_owner`. | Fast, zero table growth, but destroys historical state (`as-was`). |
| **Type 2 (Add New Row)** | Expire old row (`valid_to = change_ts`, `is_current = FALSE`) and insert new version row with a new `_sk`. | **Yes (Full Point-in-Time History)**. | `subscription_tier`, `merchant_commission_pct`, `shipping_country`, `sales_territory`. | Exact `as-was` history, but increases dimension row count and requires historical SK lookup on fact loads. |
| **Type 3 (Add Previous Column)** | Add `previous_<attr>` and `attr_changed_date` columns on the same row. | Only **1 prior state** (`Current` vs. `Previous`). | Annual sales territory realignment (*"Compare FY26 territory vs. FY25 territory"*). | Fixed schema width, no row growth, but loses changes older than 1 step. |
| **Type 4 (History Table / Mini-Dim)** | Keep `dim_entity` as current Type 1, and move history to `dim_entity_history` (or extract fast-changing attributes into a **Mini-Dimension**). | **Yes** (in separate history table or fact-stamped mini-dim). | 500M-row `dim_user` where `loyalty_tier` changes weekly; high-throughput streaming pipelines. | Keeps primary `dim_user` small and fast (`1 row/user`), avoiding `MERGE` contention. |
| **Type 6 (Hybrid: 1 + 2 + 3)** | SCD Type 2 rows **plus** a `current_<attr>` column updated via Type 1 across *all* historical rows of that entity! | **Both `As-Was` AND `As-Is` simultaneously!** | Sales Rep Territory / Merchant Tier where Finance needs historical tier and Sales Ops needs current tier in the same query. | Best query ergonomics, but updating `current_<attr>` touches all prior rows of that entity on change. |
| **Type 7 (Dual Foreign Keys on Fact)** | Fact table stores **both** `entity_sk` (historical SCD2 version) AND `entity_id` (natural key), joining to `dim_entity` (`is_current = TRUE` for `as-is`). | **Both `As-Was` AND `As-Is` without rewriting old dimension rows!** | Modern cloud warehouses (Snowflake/Iceberg) where multi-row Type 6 updates cause Parquet write amplification. | **Principal Favorite:** Zero historical dimension row updates while supporting both temporal views! |

---

## 2. Production-Grade SCD Type 2 Implementation (CDC Stream $\rightarrow$ Gold Dimension)

Writing an SCD Type 2 `MERGE` statement on a live whiteboard/coderpad is a classic Principal coding/modeling test. Notice the **three subtle traps** you must avoid:
1. **No-Op Updates:** Upstream CDC often emits full row snapshots even when *non-tracked* columns changed (e.g., `updated_at` bumped, but `tier` and `country` did not change). Always compare a **`row_hash = XXHASH64(tracked_col_1, tracked_col_2, ...)`** so you never create duplicate SCD2 rows with identical attributes!
2. **Half-Open Intervals `[valid_from_utc, valid_to_utc)`:** Always use `>= valid_from_utc AND < valid_to_utc` (never `BETWEEN` with inclusive timestamps, which causes duplicate matches at the exact boundary second!).
3. **Single-Pass `MERGE` Atomic Upsert:** Never run `UPDATE` followed by `INSERT` in two separate non-transactional steps (a failure in between leaves an entity with zero current rows!).

```sql
-- ============================================================================
-- IDEMPOTENT SINGLE-PASS SCD TYPE 2 MERGE (Snowflake / Databricks / Spark SQL)
-- Staged incoming CDC changes deduplicated to latest change per merchant_id
-- ============================================================================
MERGE INTO dim_merchant AS target
USING (
    -- Step 1: Identify true attribute changes by comparing tracked column hashes
    SELECT
        src.merchant_id                             AS merge_key,
        src.*
    FROM stg_merchant_cdc src
    LEFT JOIN dim_merchant cur
      ON src.merchant_id = cur.merchant_id
     AND cur.is_current = TRUE
    WHERE cur.merchant_id IS NULL                                -- Brand-new merchant
       OR src.tracked_attr_hash <> cur.tracked_attr_hash         -- Tracked SCD2 attribute changed!
       OR cur.is_inferred_member = TRUE                          -- Overwrite late-arriving stub!

    UNION ALL

    -- Step 2: Emit a second row with NULL merge_key to trigger the INSERT branch
    -- for existing merchants whose tracked SCD2 attributes changed
    SELECT
        NULL                                        AS merge_key,
        src.*
    FROM stg_merchant_cdc src
    JOIN dim_merchant cur
      ON src.merchant_id = cur.merchant_id
     AND cur.is_current = TRUE
    WHERE src.tracked_attr_hash <> cur.tracked_attr_hash
      AND cur.is_inferred_member = FALSE
) AS staged
ON target.merchant_id = staged.merge_key
AND target.is_current = TRUE

-- Case A: Overwrite Inferred Member stub in-place (Type 1)
WHEN MATCHED AND target.is_inferred_member = TRUE THEN
    UPDATE SET
        target.merchant_name       = staged.merchant_name,
        target.merchant_tier       = staged.merchant_tier,
        target.commission_rate_pct = staged.commission_rate_pct,
        target.country_code        = staged.country_code,
        target.tracked_attr_hash   = staged.tracked_attr_hash,
        target.is_inferred_member  = FALSE,
        target.dw_updated_at_utc   = CURRENT_TIMESTAMP()

-- Case B: Expire the existing current row when tracked SCD2 attributes changed
WHEN MATCHED AND target.is_inferred_member = FALSE THEN
    UPDATE SET
        target.valid_to_utc        = staged.cdc_event_ts_utc,
        target.is_current          = FALSE,
        target.dw_updated_at_utc   = CURRENT_TIMESTAMP()

-- Case C: Insert brand-new merchant OR new version row for changed merchant
WHEN NOT MATCHED THEN
    INSERT (
        merchant_sk, merchant_id, merchant_name, merchant_tier,
        commission_rate_pct, country_code, tracked_attr_hash,
        valid_from_utc, valid_to_utc, is_current, is_inferred_member, dw_updated_at_utc
    )
    VALUES (
        XXHASH64(CONCAT(staged.merchant_id, '|', CAST(staged.cdc_event_ts_utc AS STRING))),
        staged.merchant_id, staged.merchant_name, staged.merchant_tier,
        staged.commission_rate_pct, staged.country_code, staged.tracked_attr_hash,
        CASE WHEN staged.is_new_entity THEN TIMESTAMP '1900-01-01 00:00:00'
             ELSE staged.cdc_event_ts_utc END,
        TIMESTAMP '9999-12-31 23:59:59',
        TRUE,
        FALSE,
        CURRENT_TIMESTAMP()
    );
```

---

## 3. Handling Out-of-Order / Retroactive Dimension Changes

**The Curveball:** Suppose Merchant M1's commission rate changed from `15%` to `20%` on **Nov 10**, and our warehouse recorded that row (`valid_from = Nov 10, valid_to = 9999-12-31`). On **Nov 15**, a delayed CDC event arrives stating that M1's commission rate actually changed to `18%` back on **Nov 5**!
* **Why Standard `MERGE` Fails:** Standard `MERGE` only checks `WHERE is_current = TRUE`, so it would mistakenly expire the Nov 10 row and insert the Nov 5 row as `is_current = TRUE`—corrupting the timeline!
* **Principal Solution (Window-Based Rebuild on Affected Keys):**
  * For the specific `merchant_id`s that received retroactive historical updates (`cdc_event_ts_utc < current.valid_from_utc`), collect all version snapshots for those keys from the Silver CDC changelog (`slv_merchant_cdc_log`), sort chronologically by `cdc_event_ts_utc`, and recompute `[valid_from_utc, valid_to_utc)` cleanly using `LEAD(cdc_event_ts_utc, 1, TIMESTAMP '9999-12-31 23:59:59') OVER (PARTITION BY merchant_id ORDER BY cdc_event_ts_utc)`.
  * Then re-link affected fact rows in the trailing lookback window (`order_ts >= Nov 5`) to the newly minted `merchant_sk`!

---

## 4. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "Why did you set `valid_from_utc = '1900-01-01'` for a brand-new merchant's first row instead of `staged.cdc_event_ts_utc`?"**
   * **Expected Insight:** Clock skew between microservices! If the Order service's clock is 400ms behind the Merchant Onboarding service's clock, a merchant's very first order might have `ordered_at_utc = 10:00:00.100` while the merchant creation event has `cdc_event_ts_utc = 10:00:00.500`. Setting the initial version's `valid_from_utc = '1900-01-01'` guarantees the point-in-time range join (`ordered_at_utc >= valid_from_utc AND ordered_at_utc < valid_to_utc`) **always matches** the first version without falling into a 400ms gap!
2. **Top-Up 2: "How do you decide which attributes of `dim_customer` should be SCD Type 1 vs. SCD Type 2?"**
   * **Expected Insight:** Apply the **3-Filter Test**:
     1. *Analytical Impact:* Does slicing historical facts by the old vs. new value change a business KPI or financial calculation? (e.g., `country`, `pricing_plan`, `account_segment` $\rightarrow$ **Type 2**; `phone_number`, `avatar_url`, `name_typo_fix` $\rightarrow$ **Type 1**).
     2. *Change Velocity:* Does it change more than once a month? If it changes daily (`credit_score`, `follower_count`), move it to a **Mini-Dimension (Type 4)** rather than bloating `dim_customer` with SCD2 rows.
     3. *Compliance:* PII columns (`email`, `street_address`) belong in an isolated Type 1 PII vault so GDPR updates/deletions don't leave old PII lingering in expired SCD2 historical rows!

---

## 5. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Uses a `tracked_attr_hash` (`XXHASH64`)** to ignore no-op CDC updates and prevent duplicate SCD2 versions. | Triggers a new SCD2 version row whenever *any* column (including `updated_at`) changes in the source DB. |
| **Sets initial `valid_from_utc = '1900-01-01'`** and uses half-open `[valid_from, valid_to)` intervals to prevent clock-skew misses and boundary duplicates. | Uses `BETWEEN valid_from AND valid_to` with exact creation timestamps, causing missed joins on millisecond clock skew. |
| **Advocates SCD Type 7 (storing both `entity_sk` and `entity_id` on facts)** over multi-row Type 6 updates on Parquet/Iceberg lakehouses. | Runs multi-row historical `UPDATE`s across Iceberg dimension partitions every batch run. |