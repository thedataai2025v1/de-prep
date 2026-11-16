# dbt Practical & Coding Interview Questions

| Category | Details |
|---|---|
| **Difficulty** | Medium–Hard (Principal-Level) |
| **Topics** | dbt, SQL, Jinja, Macros, Testing, Modeling, SCD, CDC, Incremental, Snapshots |
| **Companies** | Atlassian, Snowflake, Databricks, Uber, Confluent, Google, Adobe |
| **Interview Frequency** | High (Core DE tooling for modern data stacks) |

---

> **How to use this file:** Each question mirrors a real interview scenario. Read the problem, study the solution code, understand the trade-offs, then practice explaining it aloud. The *Follow-Up Questions* and *Principal-Level Considerations* sections are what differentiate senior from principal candidates.

---

## Q1. Staging Model — Clean Raw Transaction Data

### 1. Problem Statement

Write a dbt staging model that cleans raw transaction data: handle NULLs, strip whitespace, standardize dates, and add source attribution columns.

**Input (`raw_transactions`):**

| transaction_id | customer_name | order_date | amount | currency |
|---|---|---|---|---|
| T001 | `  Alice  ` | 2024-01-15 | 150.50 | USD |
| T002 | Bob | | NULL | EUR |
| T003 | NULL | 2024/03/20 | -25.00 | GBP |
| T004 | Charlie | 15-Jan-2024 | 300.00 | USD |

**Output (`stg_transactions`):**

| transaction_id | customer_name | order_date | amount | currency | source_system | loaded_at |
|---|---|---|---|---|---|---|
| T001 | Alice | 2024-01-15 | 150.50 | USD | raw_loader | 2024-06-01 10:00:00 |
| T002 | Bob | 2024-06-01 | 0.00 | EUR | raw_loader | 2024-06-01 10:00:00 |
| T003 | UNKNOWN | 2024-03-20 | 0.00 | GBP | raw_loader | 2024-06-01 10:00:00 |
| T004 | Charlie | 2024-01-15 | 300.00 | USD | raw_loader | 2024-06-01 10:00:00 |

### 2. Solution

```sql
-- models/staging/stg_transactions.sql
{{ config(
    materialized = 'view',
    tags = ['staging', 'transactions']
) }}

WITH cleaned AS (
    SELECT
        -- Handle NULL transaction IDs
        COALESCE(NULLIF(TRIM(transaction_id), ''), 'UNKNOWN') AS transaction_id,

        -- Strip whitespace, handle NULLs
        COALESCE(NULLIF(TRIM(customer_name), ''), 'UNKNOWN') AS customer_name,

        -- Standardize dates: handle multiple formats
        COALESCE(
            TRY_CAST(order_date AS DATE),
            TRY_CAST(DATE_FORMAT(order_date, 'yyyy-MM-dd') AS DATE),
            CURRENT_DATE
        ) AS order_date,

        -- Clamp negative amounts to 0
        GREATEST(COALESCE(amount, 0), 0) AS amount,

        -- Standardize currency codes to uppercase
        UPPER(TRIM(COALESCE(currency, 'USD'))) AS currency,

        -- Source attribution (always added to staging models)
        'raw_loader' AS source_system,
        CURRENT_TIMESTAMP AS loaded_at

    FROM {{ ref('raw_transactions') }}
)

SELECT *
FROM cleaned
```

### 3. Approaches

| Approach | Handling | Trade-off |
|---|---|---|
| `COALESCE` + `TRIM` | Simple, portable | Doesn't handle complex date formats |
| `TRY_CAST` / `SAFE_CAST` | Graceful NULL on failure | Vendor-specific (not all DBs) |
| dbt `safe_divide` macro | Reusable patterns | Requires macro management |

### 4. Edge Cases

| Edge Case | Behavior |
|---|---|
| All-NULL row | Still emitted with `UNKNOWN`/`CURRENT_DATE`/`0.00` |
| Unparseable date string | Falls back to `CURRENT_DATE` |
| Negative amounts | Clamped to `0.00` by `GREATEST` |
| Extremely long strings | `TRIM` handles padding; check DB varchar limits |
| Timezone-aware dates | Use `TIMESTAMP_NTZ`/`AT TIME ZONE` for consistency |

### 5. Follow-Up Questions

**Q1. Why use `TRY_CAST` instead of `CAST`?**
→ `CAST` raises an error on invalid data and halts the model. `TRY_CAST` returns NULL, allowing the pipeline to continue and the `COALESCE` fallback to kick in. In production, you want to log bad records, not fail the run.

**Q2. How would you handle date formats from multiple countries?**
→ Use a date-parsing macro that tries common formats sequentially. On Snowflake: `DATE_TRUNC('day', TRY_TO_DATE(col, 'YYYY-MM-DD') ?? TRY_TO_DATE(col, 'DD-MM-YYYY') ?? TRY_TO_DATE(col, 'MM/DD/YYYY'))`.

**Q3. What's the difference between this staging model and the raw source?**
→ Raw is an exact copy of the source. Staging applies cleaning/standardization but no business logic. This is dbt's "three-tier" architecture: raw → staging → marts.

**Q4. How do you track which rows had bad data?**
→ Add a `data_quality_flags` column or CTE that counts rows falling back to defaults. Pipe those counts to a data quality dashboard or dbt test.

**Q5. Why is this a `view` and not a `table`?**
→ Staging models are typically views: they're intermediate, recomputed from raw each run, and cheap to refresh. Marts/intermediates are often tables for query performance.

### 6. Principal-Level Considerations

- **Cost:** Views are computed at query time. At 1B+ rows, materialize or use incremental. On Snowflake, warehouse size matters more than materialization choice.
- **Observability:** Add `source_table` and `source_record_hash` columns to every staging model for lineage and debugging.
- **Idempotency:** Staging models should always produce the same output given the same raw input. No `CURRENT_TIMESTAMP` in production staging—use the source's timestamp instead.
- **Testing:** Pair every staging model with `not_null`, `unique`, and `relationships` tests in `schema.yml`.

---

## Q2. Incremental Model — Merge Strategy

### 1. Problem Statement

Write a dbt incremental model for `fact_events` using the merge strategy. Include the `is_incremental()` Jinja block, primary key, and merge logic. Explain first run vs. subsequent runs.

**Input (new records each run):**

| event_id | user_id | event_type | event_ts | revenue |
|---|---|---|---|---|
| E001 | U100 | purchase | 2024-06-01 10:00 | 50.00 |
| E002 | U101 | click | 2024-06-02 14:30 | 0.00 |

### 2. Solution

```sql
-- models/marts/fact_events.sql
{{ config(
    materialized = 'incremental',
    unique_key = 'event_id',
    incremental_strategy = 'merge',
    merge_update_columns = ['event_type', 'revenue'],
    on_schema_change = 'sync_all_columns',
    tags = ['marts', 'events', 'incremental']
) }}

-- Source CTE (always present)
WITH source AS (
    SELECT
        event_id,
        user_id,
        event_type,
        event_ts,
        revenue,
        CURRENT_TIMESTAMP AS dbt_updated_at
    FROM {{ ref('stg_events') }}
)

SELECT *
FROM source

{% if is_incremental() %}
    -- Only process records newer than the last loaded event_ts
    WHERE event_ts > (SELECT MAX(event_ts) FROM {{ this }})
{% endif %}
```

### 3. How It Works

| Run Type | Behavior | SQL Generated |
|---|---|---|
| **First run** | Loads ALL source rows | `INSERT INTO target SELECT * FROM source` |
| **Subsequent runs** | Only inserts/updates new or changed rows | `MERGE INTO target WHEN MATCHED THEN UPDATE... WHEN NOT MATCHED THEN INSERT...` |

The `is_incremental()` macro returns `true` only when the target table already exists. On first run, it returns `false`, so the `WHERE` clause is skipped.

### 4. Approaches Comparison

| Strategy | When to Use | Upsert? | Deletes? |
|---|---|---|---|
| `append` | Append-only logs, event streams | No | No |
| `merge` ⭐ | Records identified by PK; may update | Yes | No |
| `delete+insert` | Full record replacement on change | Yes | Yes |

### 5. Edge Cases

| Edge Case | Behavior | Fix |
|---|---|---|
| Same `event_id` arrives twice | Merge updates existing row | Ensure idempotent source |
| Late-arriving data (older event_ts) | Skipped by `WHERE` clause | Add a `lookback` window |
| Schema change in source | Fails by default | Use `on_schema_change = 'sync_all_columns'` |
| Duplicate primary keys in source | Unpredictable merge result | Deduplicate in source CTE |

### 6. Follow-Up Questions

**Q1. What's the difference between `merge` and `delete+insert`?**
→ `merge` only inserts new rows and updates changed columns in existing rows. `delete+insert` deletes the old row and inserts the complete new row—useful when many columns change or for full audit trails.

**Q2. How does `on_schema_change = 'sync_all_columns'` help?**
→ If the source adds a new column, dbt will add it to the target table on the next incremental run instead of failing. Without it, any schema drift breaks the pipeline.

**Q3. How would you handle late-arriving data?**
→ Replace `WHERE event_ts > (SELECT MAX(event_ts) FROM {{ this }})` with a configurable lookback: `WHERE event_ts >= (SELECT MAX(event_ts) - INTERVAL '{{ var("lookback_days", 2) }}' DAY FROM {{ this }})`.

**Q4. What happens if the warehouse doesn't support MERGE?**
→ dbt falls back to `insert` strategy: it creates a temporary table with new records, then `INSERT INTO target SELECT...` with a `WHERE NOT EXISTS` clause.

**Q5. Why not always use incremental models?**
→ Incremental models add complexity in testing and backfilling. For small, static tables, a full table or view is simpler and less error-prone.

### 7. Principal-Level Considerations

- **Backfill:** `is_incremental()` returns `false` on full rebuilds (e.g., `dbt run --full-refresh`). Always test this path.
- **Cost control:** Add a `WHERE` filter based on a partition key or watermark column. Without it, every incremental run scans the entire source.
- **Idempotency test:** Run the model twice with the same data and verify row count doesn't change.
- **Concurrency:** dbt runs models in parallel by default. Ensure incremental models don't have cross-model dependencies that could race.

---

## Q3. Snapshot — Timestamp Strategy

### 1. Problem Statement

Write a dbt snapshot for `dimension_users` using the timestamp strategy with `updated_at`. Show the `.sql` file and `schema.yml` configuration.

**Input (`stg_users`):**

| user_id | name | email | updated_at |
|---|---|---|---|
| U001 | Alice | alice@old.com | 2024-01-01 |
| U002 | Bob | bob@test.com | 2024-03-15 |

**After update to `stg_users`:**

| user_id | name | email | updated_at |
|---|---|---|---|
| U001 | Alice | alice@new.com | 2024-06-01 |
| U002 | Bob | bob@test.com | 2024-03-15 |

### 2. Solution

```sql
-- models/snapshots/snapshot_users.sql
{{
    config(
        strategy='timestamp',
        unique_key='user_id',
        updated_at='updated_at',
        at_column='dbt_at',
        invalidate_hard_deletes=True,
        snapshot_table_column_names={
            'dbt_valid_from': 'valid_from',
            'dbt_valid_to': 'valid_to',
            'dbt_updated_at': 'dbt_last_updated',
        }
    )
}}

SELECT
    user_id,
    name,
    email,
    updated_at
FROM {{ ref('stg_users') }}
```

```yaml
# models/snapshots/snapshot_users.yml
version: 2

models:
  - name: snapshot_users
    description: "Semi-additive snapshot of user dimension with historical changes"
    columns:
      - name: user_id
        tests:
          - unique
          - not_null
      - name: name
        tests:
          - not_null
      - name: email
      - name: updated_at
      - name: dbt_at
        description: "Timestamp of when the snapshot was taken"
      - name: valid_from
        description: "Start of the validity period"
      - name: valid_to
        description: "End of the validity period (NULL = current row)"
      - name: dbt_last_updated
        description: "When this snapshot row was last updated"
```

### 3. Columns Added by dbt

| Column | Purpose | Example |
|---|---|---|
| `dbt_at` (`valid_from`) | When the snapshot was taken | `2024-06-01 10:00:00` |
| `dbt_valid_to` (`valid_to`) | When this row stopped being current | `NULL` = current; `2024-06-01` = superseded |
| `dbt_updated_at` (`dbt_last_updated`) | The `updated_at` value at snapshot time | `2024-06-01 09:00:00` |

### 4. Result After Update

| user_id | name | email | updated_at | valid_from | valid_to | dbt_last_updated |
|---|---|---|---|---|---|---|
| U001 | Alice | alice@old.com | 2024-01-01 | 2024-01-01 | 2024-06-01 | 2024-01-01 |
| U001 | Alice | alice@new.com | 2024-06-01 | 2024-06-01 | NULL | 2024-06-01 |
| U002 | Bob | bob@test.com | 2024-03-15 | 2024-03-15 | NULL | 2024-03-15 |

U002's original row remains (no change detected). U001 gets a new row; the old row's `valid_to` is set to the snapshot timestamp.

### 5. Edge Cases

| Edge Case | Behavior | Fix |
|---|---|---|
| `updated_at` doesn't change on update | Row not detected as changed | Use `check` strategy instead |
| Hard delete in source | `invalidate_hard_deletes=True` closes out the row | Set to `False` to ignore deletes |
| Clock skew between source and warehouse | Validity periods may overlap | Normalize timestamps to UTC |
| Massive history table growth | Query performance degrades | Partition by `valid_from`; archive old rows |

### 6. Follow-Up Questions

**Q1. What's the difference between `timestamp` and `check` strategies?**
→ `timestamp` compares `updated_at` to the last known value. `check` compares specified columns (e.g., `name`, `email`) for any difference, regardless of the timestamp.

**Q2. How does a snapshot differ from an incremental model?**
→ Snapshots are specifically for historical tracking (SCD-like). Incremental models manage the current state. A snapshot creates new rows on change; an incremental model typically updates in place.

**Q3. Why would you use `invalidate_hard_deletes=True`?**
→ When a user is deleted from the source, their snapshot row gets `valid_to` set to now, marking the deletion in the history. Critical for audit compliance.

**Q4. How do you query the "current" state from a snapshot?**
→ `SELECT * FROM snapshot_users WHERE valid_to IS NULL` gives you all currently active rows.

**Q5. What's the performance impact of snapshots at scale?**
→ Each change creates a new row. A 1M-user table updated daily grows to 365M rows/year. Partition by date, limit history retention, or use incremental SCD models instead.

### 7. Principal-Level Considerations

- **Storage:** Snapshots can grow 10-100x the source. Budget for storage and consider data retention policies.
- **Scheduling:** Run snapshots on a cadence that matches business needs (hourly for real-time, daily for batch). Don't snapshot every column—select the ones that matter.
- **Testing:** Test that `valid_to IS NULL` row count + `valid_to IS NOT NULL` count = total historical changes.
- **Alternative:** For very large tables, consider a manual SCD Type 2 model or a streaming change log (e.g., Debezium + Kafka) instead of dbt snapshots.

---

## Q4. Custom Macro — safe_divide

### 1. Problem Statement

Write a custom dbt macro called `safe_divide(numerator, denominator)` that handles division by zero. Show how to use it in a model.

### 2. Solution

```jinja
-- macros/safe_divide.sql
{#-
    Safely divide two values, returning 0 (or NULL) when the denominator is 0.

    Args:
        numerator: The numerator expression (column or literal)
        denominator: The denominator expression (column or literal)
        default: Value to return when denominator is 0 (default: 0)

    Returns:
        The division result or the default value

    Example:
        {{ safe_divide('total_revenue', 'total_orders', '0') }}
#-}
{% macro safe_divide(numerator, denominator, default='0') -%}
    CASE
        WHEN {{ denominator }} = 0 OR {{ denominator }} IS NULL
            THEN {{ default }}
        ELSE {{ numerator }}::FLOAT / {{ denominator }}
    END
{%- endmacro %}
```

**Usage in a model:**

```sql
-- models/marts/customer_metrics.sql
WITH customer_orders AS (
    SELECT
        customer_id,
        SUM(revenue) AS total_revenue,
        COUNT(order_id) AS total_orders
    FROM {{ ref('fact_orders') }}
    GROUP BY 1
)

SELECT
    customer_id,
    total_revenue,
    total_orders,
    {{ safe_divide('total_revenue', 'total_orders', '0') }}
        AS avg_order_value,
    {{ safe_divide('total_orders', '365', '0') }}
        AS orders_per_day
FROM customer_orders
```

### 3. Approaches Comparison

| Approach | Zero Handling | NULL Handling | Customizable Default? |
|---|---|---|---|
| `CASE WHEN` (above) ⭐ | Returns default | Returns default | Yes |
| `NULLIF` | `1/NULLIF(0,0)` → NULL | Already NULL | No |
| `COALESCE(..., 0)` | Wraps the whole expression | Wraps NULL → 0 | No |

### 4. Edge Cases

| Edge Case | Behavior |
|---|---|
| Both numerator and denominator are 0 | Returns `default` (0) |
| Numerator is NULL | Returns `default` |
| Denominator is NULL | Returns `default` |
| Both are strings | Type-cast to FLOAT in the macro |
| Denominator is 0.00001 | Returns the actual division (not zero) |

### 5. Follow-Up Questions

**Q1. Why use a macro instead of writing the CASE in every model?**
→ DRY principle. A macro ensures consistent behavior across all models and is easy to update globally. If the business decides the default should be `NULL` instead of `0`, change one file.

**Q2. Why use string parameters instead of column references?**
→ dbt macros receive raw SQL expressions as strings. The Jinja template injects them directly into the generated SQL. You can pass column names, aliases, or full expressions.

**Q3. What if I need `NULL` as the default instead of `0`?**
→ Use the third parameter: `{{ safe_divide('a', 'b', 'NULL') }}`. The macro's `default` parameter is flexible.

**Q4. How do you test a macro?**
→ Write a test model that exercises all branches (zero, NULL, normal division) and add assertions in `schema.yml`. Or use dbt's built-in `jinja` tests.

**Q5. Why `::FLOAT` cast?**
→ Integer division in some databases truncates. Explicit casting ensures floating-point division. On Snowflake/BigQuery this is automatic, but it's defensive for Postgres/Redshift.

### 6. Principal-Level Considerations

- **Portability:** This macro works on all major warehouses. Avoid vendor-specific functions in shared macros.
- **Performance:** `CASE WHEN` is SARGable in most engines. No performance penalty vs. raw division.
- **Extensibility:** Consider adding a `precision` parameter for `ROUND(result, precision)` in financial contexts.

---

## Q5. Manual Type 2 SCD

### 1. Problem Statement

Write a dbt model that implements a Type 2 Slowly Changing Dimension manually (without snapshots), using a CTE-based approach. Show how `current_start_date` and `current_end_date` are computed.

**Source (`stg_products` — daily snapshot):**

| product_id | product_name | price | category |
|---|---|---|---|
| P001 | Widget A | 10.00 | Electronics |
| P002 | Widget B | 25.00 | Hardware |

**Next day (`stg_products`):**

| product_id | product_name | price | category |
|---|---|---|---|
| P001 | Widget A | 12.00 | Electronics |
| P002 | Widget B | 25.00 | Hardware |
| P003 | Widget C | 5.00 | New |

### 2. Solution

```sql
-- models/marts/scd_products.sql
{{ config(
    materialized = 'incremental',
    unique_key = 'product_id',
    incremental_strategy = 'merge'
) }}

WITH source AS (
    SELECT
        product_id,
        product_name,
        price,
        category,
        CURRENT_DATE AS record_date
    FROM {{ ref('stg_products') }}
),

-- If incremental, load the current SCD table and mark active rows
current_state AS (
    SELECT
        product_id,
        product_name,
        price,
        category,
        start_date,
        end_date
    FROM {{ this }}
{% if is_incremental() %}
    WHERE end_date IS NULL
{% endif %}
),

-- Detect what changed between source and current state
changed AS (
    SELECT
        s.product_id,
        s.product_name,
        s.price,
        s.category,
        s.record_date AS start_date,
        NULL::DATE AS end_date,
        CASE
            WHEN c.product_id IS NULL THEN 'NEW'
            WHEN c.product_name != s.product_name
              OR c.price != s.price
              OR c.category != s.category THEN 'UPDATED'
            ELSE 'UNCHANGED'
        END AS change_type
    FROM source s
    LEFT JOIN current_state c
        ON s.product_id = c.product_id
        AND c.end_date IS NULL
),

-- Close out rows that changed (set end_date)
closed_out AS (
    SELECT
        c.product_id,
        c.product_name,
        c.price,
        c.category,
        c.start_date,
        changed.record_date AS end_date
    FROM current_state c
    INNER JOIN changed
        ON c.product_id = changed.product_id
        AND changed.change_type IN ('NEW', 'UPDATED')
),

-- Combine: close old rows + insert new rows
final AS (
    SELECT product_id, product_name, price, category, start_date, end_date
    FROM closed_out

    UNION ALL

    SELECT product_id, product_name, price, category, start_date, end_date
    FROM changed
    WHERE change_type IN ('NEW', 'UPDATED')
)

SELECT *
FROM final
```

### 3. Result After Run 2

| product_id | product_name | price | category | start_date | end_date |
|---|---|---|---|---|---|
| P001 | Widget A | 10.00 | Electronics | 2024-01-01 | 2024-06-01 |
| P001 | Widget A | 12.00 | Electronics | 2024-06-01 | NULL |
| P002 | Widget B | 25.00 | Hardware | 2024-01-01 | NULL |
| P003 | Widget C | 5.00 | New | 2024-06-01 | NULL |

P001 gets closed out (price changed). P002 stays open (no change). P003 is a new entry.

### 4. Approaches Comparison

| Approach | Pros | Cons |
|---|---|---|
| **Manual CTE** (above) ⭐ | Full control; custom logic; portable | More code; harder to maintain |
| **dbt snapshots** | Built-in; less code | Less flexible; black-box logic |
| **MERGE-based SCD** | Efficient at scale | Vendor-specific syntax |

### 5. Edge Cases

| Edge Case | Behavior | Fix |
|---|---|---|
| All columns changed | Treated as `UPDATED` correctly | — |
| Product soft-deleted (no longer in source) | Row stays open (end_date NULL) | Add a hard-delete detection CTE |
| Timestamp precision mismatch | Equality check may miss changes | Use `COALESCE` + tolerance |
| Full-refresh | `is_incremental()` is false → loads all | Add a guard: only close rows within a date range |

### 6. Follow-Up Questions

**Q1. Why implement SCD manually instead of using dbt snapshots?**
→ Manual SCD lets you control the granularity (e.g., only certain columns trigger a new version), handle soft deletes, and merge business logic. Snapshots are simpler but less customizable.

**Q2. How would you handle soft deletes?**
→ Add a CTE that joins `current_state` with `source` using `NOT EXISTS`, and closes out rows for products no longer in the source.

**Q3. What's the difference between Type 1, Type 2, and Type 7?**
→ Type 1: overwrite (no history). Type 2: new row on change (full history). Type 7: hybrid—Type 2 for history, plus a separate "current" table for fast lookups.

**Q4. How do you ensure referential integrity with a fact table?**
→ Join the fact table to the SCD using the business key + the effective date range: `fact.date BETWEEN dim.start_date AND COALESCE(dim.end_date, '9999-12-31')`.

**Q5. What's the row growth rate for a 100K-product table?**
→ If 5% of products change daily: 5,000 new rows/day → 1.825M rows/year. Budget for this growth in storage and query time.

### 7. Principal-Level Considerations

- **Partitioning:** Always partition SCD tables by `start_date` (monthly or quarterly).
- **Indexing:** Index on `product_id` + `end_date IS NULL` for current-state lookups.
- **Backfill:** Test the full-refresh path explicitly. Manual SCD models can produce duplicate rows if run twice without an incremental strategy.
- **Cost:** On Snowflake, consider clustering keys (`CLUSTER BY product_id`) for 10M+ rows.

---

## Q6. Custom Test — Non-Negative Revenue

### 1. Problem Statement

Write a dbt custom test to validate that revenue values are always non-negative in a fact table. Show the test SQL and `schema.yml` configuration.

### 2. Solution

**Custom test macro:**

```jinja
-- macros/tests/accept_non_negative.sql
{%- test accept_non_negative(model, column_name, min_value=0) -%}

    select *
    from {{ model }}
    where {{ column_name }} < {{ min_value }}

{%- endtest -%}
```

**In `schema.yml`:**

```yaml
version: 2

models:
  - name: fact_transactions
    columns:
      - name: revenue
        tests:
          - accept_non_negative
          - accept_non_negative:
              min_value: -100  # Allow up to $100 refund per transaction
```

### 3. Approaches Comparison

| Approach | Flexibility | Performance | Best For |
|---|---|---|---|
| **Custom macro** (above) ⭐ | Configurable `min_value` | O(1) scan | Any numeric column |
| **Built-in `accepted_values`** | Limited to discrete values | O(1) scan | Categorical checks |
| **`dbt-expectations` package** | Rich test library | Varies | Complex validation suites |

### 4. Edge Cases

| Edge Case | Behavior |
|---|---|
| NULL revenue | Not caught by `< 0` (NULL < 0 is NULL, not TRUE) |
| Exactly 0 | Passes (non-negative means ≥ 0) |
| Floating-point: -0.0000001 | Fails correctly |
| Empty table | Passes (no rows violate) |

### 5. Follow-Up Questions

**Q1. Why is this a "general" test instead of a "singular" test?**
→ General tests take `model` and `column_name` as parameters and are declared in `schema.yml`. Singular tests are separate `.sql` files in the `tests/` directory. General tests are reusable and declarative.

**Q2. How do you handle NULLs in custom tests?**
→ Add `AND {{ column_name }} IS NOT NULL` to the WHERE clause, or use `COALESCE({{ column_name }}, 0) < {{ min_value }}`.

**Q3. What's the performance cost of dbt tests?**
→ Tests run a `SELECT` that scans the entire model. For 1B+ row tables, use `LIMIT 1000` in the test or sample the data.

**Q4. When should a test fail the build vs. warn?**
→ Critical data quality checks (e.g., revenue < 0) should fail. Anomalies that can be investigated later should `warn`. Use the `severity` parameter in dbt >= 1.5.

**Q5. How would you test "revenue matches the sum of line items"?**
→ Write a singular test: `SELECT COUNT(*) FROM (SELECT order_id FROM fact_orders WHERE revenue != (SELECT SUM(line_total) FROM fact_order_items WHERE order_id = o.order_id)) violations WHERE COUNT(*) > 0`.

### 6. Principal-Level Considerations

- **Test governance:** Document every test in `schema.yml` with a `description` explaining the business rule.
- **Test performance:** For large tables, test on a sample or on the incremental delta. Consider running tests as a separate job with a smaller warehouse.
- **CI/CD integration:** Run tests in a staging environment before merging to production. Use `dbt test --select state:modified` to test only changed models.

---

## Q7. Multi-Table Join — fact_order_items

### 1. Problem Statement

Write a dbt model that joins `stg_orders`, `stg_customers`, and `stg_products` to create `fact_order_items`. Use proper qualified refs and handle potential NULL joins.

### 2. Solution

```sql
-- models/marts/fact_order_items.sql
{{ config(
    materialized = 'table',
    sort = 'order_date',
    dist = 'customer_id',  -- Redshift
    tags = ['marts', 'orders']
) }}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),
customers AS (
    SELECT * FROM {{ ref('stg_customers') }}
),
products AS (
    SELECT * FROM {{ ref('stg_products') }}
),

joined AS (
    SELECT
        -- Order grain
        o.order_id,
        o.order_item_id,
        o.order_date,
        o.quantity,
        o.unit_price,

        -- Customer attributes (deny the join if customer is missing)
        COALESCE(c.customer_id, 'UNKNOWN') AS customer_id,
        COALESCE(c.customer_segment, 'Unknown Segment') AS customer_segment,
        COALESCE(c.region, 'Unknown Region') AS customer_region,

        -- Product attributes
        COALESCE(p.product_id, 'UNKNOWN') AS product_id,
        COALESCE(p.product_name, 'Unknown Product') AS product_name,
        COALESCE(p.category, 'Uncategorized') AS product_category,

        -- Derived metrics
        o.quantity * o.unit_price AS line_total,


---

## Q8. Write a dbt model that pivots daily sales data into a weekly summary with aggregated metrics (revenue, units_sold, distinct_customers). Show both window function and GROUP BY approaches.

### Scenario

You have `stg_daily_sales` with daily-level data and need a weekly summary for the analytics team's dashboard.

### Approach 1: GROUP BY (Standard)

```sql
-- models/marts/fct_weekly_sales.sql

{{ config(
    materialized='table',
    partition_by={'field': 'week_start_date', 'data_type': 'date'}
) }}

with dated_sales as (
    select
        customer_id,
        order_date,
        revenue,
        units_sold
    from {{ ref('stg_daily_sales') }}
    where order_date is not null
)

select
    -- Compute week start (Monday)
    date_trunc('week', order_date) as week_start_date,

    -- Aggregated metrics
    sum(revenue) as weekly_revenue,
    sum(units_sold) as weekly_units_sold,
    count(distinct customer_id) as weekly_distinct_customers,
    count(*) as total_transactions

from dated_sales
group by 1
order by week_start_date
```

### Approach 2: Window Functions (for running weekly totals)

```sql
-- models/marts/fct_weekly_sales_running.sql

{{ config(materialized='table') }}

with dated_sales as (
    select
        customer_id,
        order_date,
        date_trunc('week', order_date) as week_start_date,
        revenue,
        units_sold
    from {{ ref('stg_daily_sales') }}
    where order_date is not null
),

daily_agg as (
    select
        week_start_date,
        order_date,
        sum(revenue) as daily_revenue,
        sum(units_sold) as daily_units_sold,
        count(distinct customer_id) as daily_customers
    from dated_sales
    group by 1, 2
),

with_running as (
    select
        week_start_date,
        order_date,
        daily_revenue,
        daily_units_sold,
        daily_customers,

        -- Running totals within each week
        sum(daily_revenue) over (
            partition by week_start_date
            order by order_date
            rows between unbounded preceding and current row
        ) as running_weekly_revenue,

        -- Last cumulative value per week
        max(sum(daily_revenue) over (
            partition by week_start_date
            order by order_date
            rows between unbounded preceding and current row
        )) over (partition by week_start_date) as weekly_revenue

    from daily_agg
)

select
    week_start_date,
    max(weekly_revenue) as weekly_revenue,
    sum(daily_units_sold) as weekly_units_sold,
    count(distinct daily_customers) as weekly_distinct_customers  -- approximate; use approach 1 for exact
from with_running
group by 1
```

### Comparison

| Aspect | GROUP BY (Approach 1) | Window Functions (Approach 2) |
|--------|----------------------|-------------------------------|
| Simplicity | Simple, readable | More complex, harder to maintain |
| Correctness | Exact for all metrics | `distinct` count is approximate |
| Performance | Single pass aggregation | Multiple passes over data |
| Use case | Final weekly summary | Running/intermediate aggregations |

**Recommendation:** Use GROUP BY for final summaries. Use window functions when you need running totals or need to carry intermediate state.

### Sample Data

**Input (stg_daily_sales):**
| order_date | customer_id | revenue | units_sold |
|-----------|-------------|---------|------------|
| 2024-01-01 | C001 | 150.00 | 3 |
| 2024-01-01 | C002 | 200.00 | 4 |
| 2024-01-02 | C001 | 100.00 | 2 |
| 2024-01-03 | C003 | 50.00 | 1 |

**Output (fct_weekly_sales):**
| week_start_date | weekly_revenue | weekly_units_sold | weekly_distinct_customers | total_transactions |
|-----------------|----------------|-------------------|---------------------------|--------------------|
| 2023-12-31 | 500.00 | 10 | 3 | 4 |

### Edge Cases

| Scenario | Handling |
|----------|----------|
| `order_date` is NULL | Filtered out in `WHERE` clause |
| Missing weeks (no sales) | Use date spine CTE with LEFT JOIN to fill gaps |
| Cross-year week boundaries | `date_trunc('week', ...)` handles this correctly |
| Timezone differences | Ensure `order_date` is consistently UTC or local |

### Follow-Up Questions

**Q: How would you fill in missing weeks (weeks with zero sales)?**

**A:** Join with a date spine:
```sql
with date_spine as (
    {{ dbt_utils.date_spine(
        datepart='week',
        start_date="to_date('2020-01-01')",
        end_date="to_date('2025-01-01')"
    )}}
),
spine as (
    select cast(date_day as date) as week_start_date
    from date_spine
)
select
    s.week_start_date,
    coalesce(f.weekly_revenue, 0) as weekly_revenue,
    coalesce(f.weekly_units_sold, 0) as weekly_units_sold,
    coalesce(f.weekly_distinct_customers, 0) as weekly_distinct_customers
from spine s
left join fct_weekly_sales f on s.week_start_date = f.week_start_date
```

**Q: How do you handle fiscal weeks that don't align with calendar weeks?**

**A:** Use a fiscal calendar date dimension:
```sql
-- dim_fiscal_calendar.sql
select
    the_date,
    fiscal_year,
    fiscal_week,
    fiscal_quarter
from {{ ref('date_spine') }}
-- fiscal mapping comes from a seed or business rule macro
```

**Q: What's the performance cost of `count(distinct customer_id)` on large datasets?**

**A:** On Snowflake, it's efficient with clustering. On BigQuery, use `approx_distinct(customer_id, 0.01)` for 1% error tolerance with 10x speedup. At TB scale, exact distinct counts can be expensive.

**Q: When would you materialize this as a view vs a table?**

**A:** View for ad-hoc exploration (no storage cost, always fresh). Table for dashboard performance (pre-computed, supports partitioning/clustering). At Principal level: view if the upstream is already fast and the aggregation is light; table if the dashboard queries this 100+ times/hour.

---

## Q9. Write a dbt macro that generates a date spine (consecutive dates) between two dates. Show how to use it in a model with LEFT JOIN to fill missing dates.

### Scenario

You need a consecutive date range to fill gaps in time-series data — a fundamental building block for analytics.

### Macro

```sql
-- macros/generate_date_spine.sql

{% macro generate_date_spine(start_date, end_date, step='1 day') %}
    /*
        Generate a consecutive list of dates between start_date and end_date.

        Args:
            start_date: SQL expression or literal (e.g., to_date('2020-01-01'))
            end_date: SQL expression or literal
            step: Increment string (default: '1 day')

        Returns:
            A CTE named __date_spine with column 'date_day'
    */

    -- Generate sequential integers
    , __numbers as (
        select level as n
        from dual
        connect by level <= 1000000  -- BigQuery: GENERATE_ARRAY; Snowflake: ROW_NUMBER
    )

    , __date_spine as (
        select
            date_add(
                {{ start_date }},
                interval (n - 1) {{ step }}
            ) as date_day
        from __numbers
        where date_add({{ start_date }}, interval (n - 1) {{ step }}) <= {{ end_date }}
    )

    select date_day
    from __date_spine
{% endmacro %}
```

### Production-Ready Cross-Database Version

```sql
-- macros/cross_db_date_spine.sql

{% macro generate_date_spine(start_date, end_date, step='1 day') %}
    {# Detect the adapter and generate appropriate SQL #}
    {% if target.type == 'bigquery' %}
        with __date_spine as (
            select
                date_add({{ start_date }}, interval n {{ step }}) as date_day
            from unnest(generate_array(0, 1000000)) as n
            where date_add({{ start_date }}, interval n {{ step }}) <= {{ end_date }}
        )
        select date_day from __date_spine

    {% elif target.type == 'snowflake' %}
        with __numbers as (
            select row_number() over () as n
            from table(generator(rowcount => 1000000))
        ),
        __date_spine as (
            select
                dateadd(day, n - 1, {{ start_date }}) as date_day
            from __numbers
            where dateadd(day, n - 1, {{ start_date }}) <= {{ end_date }}
        )
        select date_day from __date_spine

    {% elif target.type == 'postgres' %}
        with __date_spine as (
            select
                {{ start_date }} + (n::int * interval '{{ step }}') as date_day
            from generate_series(0, 1000000) as n
            where {{ start_date }} + (n::int * interval '{{ step }}') <= {{ end_date }}
        )
        select date_day from __date_spine

    {% else %}
        {{ adapter.dispatch('generate_date_spine', 'dbt')(start_date, end_date, step) }}
    {% endif %}
{% endmacro %}
```

### Usage in a Model

```sql
-- models/marts/daily_active_users_filled.sql

{{ config(materialized='table') }}

with actual_dau as (
    select
        activity_date,
        count(distinct user_id) as active_users
    from {{ ref('fct_user_activity') }}
    group by 1
),

date_spine as (
    {{ generate_date_spine(
        start_date="to_date('2020-01-01')",
        end_date="current_date()"
    )}}
)

select
    d.date_day as activity_date,
    coalesce(a.active_users, 0) as active_users
from date_spine d
left join actual_dau a on d.date_day = a.activity_date
order by d.date_day
```

### Sample Output

| activity_date | active_users |
|--------------|-------------|
| 2024-01-01 | 15230 |
| 2024-01-02 | 0 |  ← filled gap (holiday) |
| 2024-01-03 | 14890 |
| 2024-01-04 | 0 |  ← filled gap (pipeline failure) |

### Pros / Cons

| Aspect | dbt_utils.date_spine | Custom macro |
|--------|---------------------|--------------|
| Cross-database support | Yes (maintained by dbt Labs) | You must maintain it |
| Flexibility | Fixed step (day, week, month) | Custom step size |
| Dependencies | Requires dbt_utils package | No extra dependency |
| Maintenance burden | None | You maintain adapter-specific logic |

### Edge Cases

| Scenario | Handling |
|----------|----------|
| start_date > end_date | Returns empty set; add validation test |
| Date range too large (>1M rows) | Increase rowcount limit or use recursive CTE |
| Different timezones | Ensure dates are cast to the right timezone first |

### Follow-Up Questions

**Q: Why not just use `dbt_utils.date_spine` directly?**

**A:** `dbt_utils.date_spine` is the right choice 90% of the time. Write a custom one when: (1) you need non-standard steps (e.g., 6-hour intervals), (2) you're in a restricted environment without package access, or (3) you need to embed the spine as a CTE within a larger model and `dbt_utils` doesn't fit that pattern.

**Q: How do you handle timezones in a date spine?**

**A:** Define timezone at the source level:
```sql
select convert_timezone('UTC', 'America/New_York', activity_timestamp)::date as activity_date
```
At Principal level: define timezone as a dbt variable (`{{ var('analytics_timezone', 'America/New_York') }}`) so all models use the same convention.

**Q: What happens if the date spine generates millions of rows?**

**A:** On Snowflake this is fine (virtual warehouse handles it). On BigQuery with on-demand pricing, each row costs compute. Limit the spine to the actual range you need and partition the output table by date.

**Q: How do you fill gaps for weekly or monthly granularity?**

**A:** Change the step parameter: `step='1 week'` or `step='1 month'`. For weekly, align to Monday first: `date_trunc('week', start_date)`.

---

## Q10. Write a dbt model that deduplicates event data, keeping only the latest record per event_id using ROW_NUMBER(). Show the before/after data.

### Scenario

Raw event data has duplicate `event_id` rows due to retries, backfills, or CDC processing. You need to keep only the latest version of each event.

### Solution

```sql
-- models/staging/stg_events_deduplicated.sql

{{ config(
    materialized='incremental',
    unique_key='event_id',
    strategy='merge'
) }}

with ranked_events as (
    select
        *,
        row_number() over (
            partition by event_id
            order by updated_at desc, created_at desc
        ) as rn
    from {{ ref('stg_events_raw') }}

    {% if is_incremental() %}
    where updated_at >= (
        select max(updated_at) from {{ this }}
    )
    {% endif %}
)

select
    event_id,
    user_id,
    event_type,
    event_timestamp,
    event_properties,
    session_id,
    platform,
    created_at,
    updated_at

from ranked_events
where rn = 1
```

### Sample Data

**Input (stg_events_raw — with duplicates):**

| event_id | user_id | event_type | updated_at |
|----------|---------|------------|------------|
| E001 | U100 | page_view | 2024-01-15 10:00:00 |
| E001 | U100 | page_view | 2024-01-15 10:05:00 ← latest |
| E002 | U200 | click | 2024-01-15 11:00:00 |
| E003 | U300 | purchase | 2024-01-15 12:00:00 |
| E003 | U300 | purchase | 2024-01-15 12:30:00 ← latest |
| E003 | U300 | purchase | 2024-01-15 11:00:00 |

**Output (stg_events_deduplicated):**

| event_id | user_id | event_type | updated_at |
|----------|---------|------------|------------|
| E001 | U100 | page_view | 2024-01-15 10:05:00 |
| E002 | U200 | click | 2024-01-15 11:00:00 |
| E003 | U300 | purchase | 2024-01-15 12:30:00 |

### Deduplication Strategy Options

| Approach | When to Use | Limitation |
|----------|-------------|------------|
| `ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY updated_at DESC)` | General-purpose, supports tie-breaking | Requires full scan |
| `DISTINCT ON (event_id)` (Postgres only) | Postgres-specific, clean syntax | Not cross-database |
| `QUALIFY ROW_NUMBER() = 1` (Snowflake/BigQuery) | Concise, single-pass | Not available on all databases |
| `GROUP BY event_id` with MAX(updated_at) | When you only need the timestamp, not the full row | Loses other column values |

### QUALIFY Version (Snowflake/BigQuery)

```sql
select *
from {{ ref('stg_events_raw') }}
qualify row_number() over (
    partition by event_id
    order by updated_at desc, created_at desc
) = 1
```

### Edge Cases

| Scenario | Handling |
|----------|----------|
| Two records with identical `updated_at` | Tie-break with `created_at desc` or a UUID |
| `event_id` is NULL | Filter out: `WHERE event_id IS NOT NULL` |
| Incremental runs re-process old data | Use `is_incremental()` to limit the window |
| Source backfills old data after new data was loaded | Use snapshot instead of deduplication |

### Follow-Up Questions

**Q: What if the source sends an idempotent event with the same timestamp?**

**A:** Add a monotonically increasing ingestion sequence number:
```sql
row_number() over (
    partition by event_id
    order by ingestion_seq_num desc
) as rn
```

**Q: How does this handle late-arriving data that updates old events?**

**A:** This approach only keeps the latest version per event_id. If late-arriving data has an earlier timestamp but new information, it gets dropped. For this scenario, use a snapshot with check strategy instead, or ensure the source always increments `updated_at`.

**Q: What's the performance impact of ROW_NUMBER() on a billion-row table?**

**A:** It requires a full partition-sort operation. On Snowflake with clustering on `event_id`, it's manageable. On BigQuery, partition by `event_date` first, then deduplicate per partition. At Principal level: consider deduplicating at the ingestion layer (Kafka deduplication, Kafka Streams) to avoid loading duplicates in the first place.

**Q: Why not use `DELETE` for duplicates?**

**A:** In dbt, you don't delete — you rebuild. The merge strategy handles this: the new deduplicated model replaces the old rows. This is idempotent and auditable. Direct DELETE is not repeatable and breaks dbt's declarative paradigm.

---

## Q11. Write a dbt incremental model using the delete+insert strategy for a table where records can be updated. Show the full config and SQL.

### Scenario

Your fact_orders table receives both new orders and updates to existing orders (e.g., status changes, cancelled items). You need the delete+insert strategy to handle this.

```sql
-- models/marts/fact_orders.sql

{{ config(
    materialized='incremental',
    unique_key='order_id',
    strategy='delete+insert'
) }}

with source as (
    select
        order_id,
        customer_id,
        order_date,
        total_amount,
        status,
        updated_at
    from {{ ref('stg_orders') }}

    {% if is_incremental() %}
    -- Only process records that have changed since the last run
    where updated_at >= (
        select max(updated_at) from {{ this }}
    )
    {% endif %}
)

-- Deduplicate within the current batch
, ranked as (
    select
        *,
        row_number() over (
            partition by order_id
            order by updated_at desc
        ) as rn
    from source
)

select
    order_id,
    customer_id,
    order_date,
    total_amount,
    status,
    updated_at
from ranked
where rn = 1
```

### How delete+insert Works

| Step | What Happens |
|------|-------------|
| 1 | Delete rows from target where `order_id` matches any `order_id` in the new data |
| 2 | Insert all rows from the new data |

**Result:** Updated orders are effectively replaced. New orders are added.

### delete+insert vs merge

| Aspect | delete+insert | merge |
|--------|--------------|-------|
| How it works | DELETE all matching, then INSERT | Upsert: INSERT new, UPDATE existing |
| Database support | All databases | Snowflake, BigQuery, SQL Server |
| Performance | Can be slower (full delete + insert) | More efficient (single operation) |
| When to use | Databases without merge, or when records change completely | When you need partial updates |

### Follow-Up Questions

**Q: What happens if `updated_at` is NULL in the source?**
**A:** The deduplication via `ROW_NUMBER()` will keep one arbitrary row. Add `COALESCE(updated_at, created_at)` or filter NULLs upstream.

**Q: How does this handle records that don't have an `updated_at` column?**
**A:** Use the `append` strategy instead, or add a dbt-generated `_dbt_updated_at` column.

**Q: What if two incremental runs overlap in time?**
**A:** The `is_incremental()` window prevents re-processing, but if there's a gap or overlap, use a lookback window: `updated_at >= (select max(updated_at) - interval '1 day' from {{ this }})`.

**Q: Why delete+insert instead of merge for Snowflake?**
**A:** At Principal level: merge is preferred on Snowflake for performance. Use delete+insert only when (1) the target database doesn't support merge, or (2) every column in the record changes (no partial update needed).

---

## Q12. Write a dbt seed file (CSV content) for a currency_exchange_rates lookup table, then write a model that uses it to convert multi-currency transaction amounts to USD.

### Seed File

```csv
-- seeds/currency_exchange_rates.csv
currency_code,exchange_rate_to_usd,effective_date
USD,1.0000,2024-01-01
EUR,1.0850,2024-01-01
GBP,1.2650,2024-01-01
JPY,0.0067,2024-01-01
AUD,0.6520,2024-01-01
CAD,0.7450,2024-01-01
CHF,1.1750,2024-01-01
INR,0.0120,2024-01-01
BRL,0.2030,2024-01-01
CNY,0.1390,2024-01-01
```

```yaml
# models/schema.yml (or seed_sources.yml)
seeds:
  - name: currency_exchange_rates
    cols:
      - name: currency_code
        tests:
          - unique
          - not_null
      - name: exchange_rate_to_usd
        tests:
          - not_null
          - dbt_utils.equality:
              compare_to: ref('currency_exchange_rates')
              column_map:
                exchange_rate_to_usd: exchange_rate_to_usd
```

### Conversion Model

```sql
-- models/marts/fact_transactions_usd.sql

{{ config(materialized='table') }}

with source as (
    select
        transaction_id,
        customer_id,
        amount,
        currency_code,
        transaction_date
    from {{ ref('stg_transactions') }}
),

converted as (
    select
        s.transaction_id,
        s.customer_id,
        s.amount as original_amount,
        s.currency_code as original_currency,
        s.amount * coalesce(c.exchange_rate_to_usd, 1.0) as amount_usd,
        s.transaction_date
    from source s
    left join {{ ref('currency_exchange_rates') }} c
        on s.currency_code = c.currency_code
)

select
    transaction_id,
    customer_id,
    original_amount,
    original_currency,
    round(amount_usd, 2) as amount_usd,
    transaction_date
from converted
```

### Sample Data

**Input (stg_transactions):**

| transaction_id | customer_id | amount | currency_code | transaction_date |
|---------------|-------------|--------|---------------|-----------------|
| T001 | C100 | 150.00 | EUR | 2024-01-15 |
| T002 | C200 | 5000 | JPY | 2024-01-15 |
| T003 | C300 | 75.50 | USD | 2024-01-15 |

**Output (fact_transactions_usd):**

| transaction_id | original_amount | original_currency | amount_usd |
|---------------|----------------|-------------------|-----------|
| T001 | 150.00 | EUR | 162.75 |
| T002 | 5000 | JPY | 33.50 |
| T003 | 75.50 | USD | 75.50 |

### Edge Cases

| Scenario | Handling |
|----------|----------|
| Unknown currency code | `COALESCE(c.exchange_rate_to_usd, 1.0)` — treat as USD |
| Missing exchange rate | Log as warning with a custom test |
| Rate changes over time | Use time-varying seeds or a sourced table |

### Follow-Up Questions

**Q: Seeds are stored in the warehouse — isn't that wasteful?**
**A:** Yes. Seeds are loaded as tables in the warehouse. For large lookup tables, load them from an external file (GCS/S3) instead. Seeds are best for small, infrequently changing reference data (< 10K rows).

**Q: How do you update exchange rates regularly?**
**A:** At Principal level: don't use seeds for time-varying data. Use an external table loaded from a daily API call or a staging model from a currency API source. Seeds should be static.

**Q: What if we need historical exchange rates?**
**A:** Use a time-varying lookup table:
```sql
select
    s.*,
    c.exchange_rate_to_usd
from source s
join currency_exchange_rates c
    on s.currency_code = c.currency_code
    and s.transaction_date >= c.effective_date
    and s.transaction_date < coalesce(c.expiry_date, '2100-01-01')
```

**Q: How do you handle rounding differences?**
**A:** Round at the final output layer, not intermediate. Use `ROUND(amount_usd, 2)` only in the final marts model. Keep full precision in staging and intermediate layers.

---

## Q13. Write a dbt model that flattens a semi-structured JSON column (event_properties) into individual columns using database-specific functions.

### Scenario

Your raw event data has a JSON `event_properties` column with nested attributes that need to be flattened for analytics.

### Snowflake Version (using FLATTEN)

```sql
-- models/staging/stg_events_flattened_snowflake.sql

{% if target.type == 'snowflake' %}

{{ config(materialized='view') }}

with raw as (
    select
        event_id,
        user_id,
        event_type,
        event_timestamp
    from {{ ref('stg_events_raw') }}
),

flattened as (
    select
        r.event_id,
        r.user_id,
        r.event_type,
        r.event_timestamp,
        -- Extract specific fields from JSON
        value:event_properties:page_name::varchar as page_name,
        value:event_properties:referrer::varchar as referrer,
        value:event_properties:screen_width::integer as screen_width,
        value:event_properties:screen_height::integer as screen_height,
        value:event_properties:device_type::varchar as device_type,
        value:event_properties:browser::varchar as browser,
        -- Handle nested JSON arrays
        get(path, 'event_properties.tags')::varchar as tags_json
    from raw r
)

select * from flattened

{% endif %}
```

### Postgres Version (using jsonb)

```sql
{% if target.type == 'postgres' %}

{{ config(materialized='view') }}

with raw as (
    select
        event_id,
        user_id,
        event_type,
        event_timestamp,
        event_properties::jsonb as props
    from {{ ref('stg_events_raw') }}
)

select
    event_id,
    user_id,
    event_type,
    event_timestamp,
    (props ->> 'page_name')::text as page_name,
    (props ->> 'referrer')::text as referrer,
    (props ->> 'screen_width')::integer as screen_width,
    (props ->> 'screen_height')::integer as screen_height,
    (props ->> 'device_type')::text as device_type,
    (props ->> 'browser')::text as browser,
    props -> 'tags' as tags_jsonb
from raw
{% endif %}
```

### BigQuery Version (using JSON_EXTRACT)

```sql
{% if target.type == 'bigquery' %}

{{ config(materialized='view') }}

select
    event_id,
    user_id,
    event_type,
    event_timestamp,
    JSON_VALUE(event_properties, '$.page_name') as page_name,
    JSON_VALUE(event_properties, '$.referrer') as referrer,
    SAFE_CAST(JSON_VALUE(event_properties, '$.screen_width') AS INT64) as screen_width,
    SAFE_CAST(JSON_VALUE(event_properties, '$.screen_height') AS INT64) as screen_height,
    JSON_VALUE(event_properties, '$.device_type') as device_type,
    JSON_VALUE(event_properties, '$.browser') as browser,
    JSON_EXTRACT(event_properties, '$.tags') as tags_json
from {{ ref('stg_events_raw') }}
{% endif %}
```

### Cross-Database Macro Approach

```sql
-- macros/extract_json_field.sql

{% macro extract_json_field(json_column, path, return_type='varchar') %}
    {% if target.type == 'snowflake' %}
        {{ json_column }}:{{ path }}::{{ return_type }}
    {% elif target.type == 'bigquery' %}
        {% if return_type == 'varchar' %}
            JSON_VALUE({{ json_column }}, '$.{{ path }}')
        {% elif return_type == 'integer' %}
            SAFE_CAST(JSON_VALUE({{ json_column }}, '$.{{ path }}') AS INT64)
        {% else %}
            JSON_VALUE({{ json_column }}, '$.{{ path }}')
        {% endif %}
    {% elif target.type == 'postgres' %}
        ({{ json_column }} ->> '{{ path }}')::{{ return_type }}
    {% else %}
        {{ json_column }}
    {% endif %}
{% endmacro %}
```

### Follow-Up Questions

**Q: What if the JSON structure varies across event types?**
**A:** Create separate flattening models per event type, or use a conditional extraction macro:
```sql
case
    when event_type = 'page_view' then extract_json_field(props, 'page_name')
    when event_type = 'click' then extract_json_field(props, 'button_id')
end as contextual_field
```

**Q: How do you handle deeply nested JSON (3+ levels)?**
**A:** Chain the path: `props -> 'user' -> 'preferences' -> 'theme'` in Postgres, or `props:user:preferences:theme` in Snowflake.

**Q: What if the JSON is malformed?**
**A:** Use `TRY_PARSE_JSON()` in Snowflake, or `SAFE.JSON_VALUE()` in BigQuery to return NULL instead of throwing an error.

**Q: At TB scale, is JSON extraction performant?**
**A:** No — JSON extraction is CPU-intensive. At Principal level: flatten JSON at the ingestion layer (Spark/Kafka Streams) before loading into the warehouse. Only use in-database JSON extraction for small-scale or ad-hoc use cases.

---

## Q14. Write a dbt model that calculates a running total and a 7-day moving average of daily sales using window functions.

### Solution

```sql
-- models/marts/daily_sales_with_running_metrics.sql

{{ config(materialized='table') }}

with daily_sales as (
    select
        order_date,
        sum(revenue) as daily_revenue,
        sum(units_sold) as daily_units
    from {{ ref('stg_orders') }}
    group by 1
),

with_windows as (
    select
        order_date,
        daily_revenue,
        daily_units,

        -- Running total (cumulative sum from the beginning)
        sum(daily_revenue) over (
            order by order_date
            rows between unbounded preceding and current row
        ) as cumulative_revenue,

        sum(daily_units) over (
            order by order_date
            rows between unbounded preceding and current row
        ) as cumulative_units,

        -- 7-day moving average
        avg(daily_revenue) over (
            order by order_date
            rows between 6 preceding and current row
        ) as revenue_7day_moving_avg,

        -- 7-day moving sum
        sum(daily_revenue) over (
            order by order_date
            rows between 6 preceding and current row
        ) as revenue_7day_moving_sum

    from daily_sales
)

select
    order_date,
    daily_revenue,
    daily_units,
    round(cumulative_revenue, 2) as cumulative_revenue,
    cumulative_units,
    round(revenue_7day_moving_avg, 2) as revenue_7day_moving_avg,
    round(revenue_7day_moving_sum, 2) as revenue_7day_moving_sum
from with_windows
order by order_date
```

### Sample Data

| order_date | daily_revenue | cumulative_revenue | revenue_7day_moving_avg |
|-----------|--------------|-------------------|------------------------|
| 2024-01-01 | 1500.00 | 1500.00 | 1500.00 |
| 2024-01-02 | 1800.00 | 3300.00 | 1650.00 |
| 2024-01-03 | 2100.00 | 5400.00 | 1800.00 |
| 2024-01-04 | 1600.00 | 7000.00 | 1750.00 |
| 2024-01-05 | 2400.00 | 9400.00 | 1880.00 |
| 2024-01-06 | 1900.00 | 11300.00 | 1883.33 |
| 2024-01-07 | 2200.00 | 13500.00 | 1985.71 |
| 2024-01-08 | 2500.00 | 16000.00 | 2085.71 |

### Follow-Up Questions

**Q: What happens on days with no sales?**
**A:** The window function skips the gap. Fill gaps with a date spine (see Q9) and LEFT JOIN to ensure every day appears.

**Q: How do you calculate a trailing 7-day average (excluding current day)?**
**A:** Change to `rows between 7 preceding and 1 preceding`.

**Q: What's the performance cost of window functions at scale?**
**A:** Window functions require partitioning and sorting. On Snowflake with clustering on `order_date`, they're efficient. On BigQuery, ensure the table is partitioned by date.

**Q: How would you calculate a year-over-year comparison?**
**A:** Use `LAG(daily_revenue, 365)` to compare with the same day last year.

---

## Q15. Write a dbt custom test to check referential integrity between two models where the foreign key relationship is not enforced by the database.

### Solution

```sql
-- macros/tests/referential_integrity.sql

{% test referential_integrity(model, column_name, to, to_column_name, field='all') %}
    /*
        Custom test to check that every value in `column_name` of `model`
        exists in `to_column_name` of `to`.

        Args:
            model: The child model (with the FK)
            column_name: FK column in the child
            to: The parent model (referenced table)
            to_column_name: PK column in the parent
    */

    select
        child.{{ column_name }} as fk_id,
        count(*) as orphan_count
    from {{ model }} as child
    left join {{ to }} as parent
        on child.{{ column_name }} = parent.{{ to_column_name }}
    where parent.{{ to_column_name }} is null
      and child.{{ column_name }} is not null
    group by 1
{% endmacro %}
```

### Usage in schema.yml

```yaml
models:
  - name: fact_orders
    tests:
      - referential_integrity:
          column_name: customer_id
          to: ref('dim_customers')
          to_column_name: customer_id
      - referential_integrity:
          column_name: product_id
          to: ref('dim_products')
          to_column_name: product_id
```

### How It Works

| Step | What It Does |
|------|-------------|
| LEFT JOIN child to parent | Finds child rows with no matching parent |
| WHERE parent is NULL | Identifies orphans |
| GROUP BY + COUNT | Shows how many orphans per FK value |

### Follow-Up Questions

**Q: How expensive is this test on large tables?**
**A:** Very — it does a full join. At Principal level: run this test on staging models only, or sample: `QUALIFY row_number() over () <= 100000`. For production, use a materialized test table with incremental refresh.

**Q: What if the foreign key can be NULL (optional relationship)?**
**A:** The test handles this: `and child.{{ column_name }} is not null` filters out legitimate NULLs.

**Q: How do you test bidirectional referential integrity?**
**A:** Run the test in both directions:
```yaml
tests:
  - referential_integrity:
      column_name: department_id
      to: ref('dim_departments')
  - referential_integrity:
      column_name: employee_id
      to: ref('dim_employees')
```

---

## Q16. Write a dbt model that implements a periodic snapshot fact table (daily inventory levels) using dbt snapshots with the check strategy.

### Solution

```sql
-- models/snapshots/snap_inventory_daily.sql

{{
    config(
        strategy='check',
        unique_key='inventory_id',
        check_cols=['quantity_on_hand', 'warehouse_id', 'status'],
        updated_at=None,
        snapshot_timestamp=None,  -- uses current timestamp
        dbt_snapshot_mapping={
            'invalidate_hard_deletes': True
        }
    )
}}

select
    product_id,
    warehouse_id,
    quantity_on_hand,
    status,
    last_recounted_at
from {{ ref('stg_inventory') }}
```

### Configuration in snapshots/ schema.yml

```yaml
models:
  - name: snap_inventory_daily
    description: "Daily inventory snapshot — tracks changes in stock levels"
    columns:
      - name: dbt_valid_from
        description: "When this version became valid"
      - name: dbt_valid_to
        description: "When this version expired (NULL = current)"
```

### How Check Strategy Works

| Step | What Happens |
|------|-------------|
| 1 | dbt compares `quantity_on_hand`, `warehouse_id`, `status` against existing rows |
| 2 | If any checked column changed, it closes the old row (`dbt_valid_to`) and opens a new one |
| 3 | New inventory items get a new row with `dbt_valid_from` |

### Sample Output

| inventory_id | product_id | quantity_on_hand | dbt_valid_from | dbt_valid_to |
|-------------|-----------|-----------------|----------------|-------------|
| INV001 | P001 | 100 | 2024-01-01 00:00 | 2024-01-02 00:00 |
| INV001 | P001 | 95 | 2024-01-02 00:00 | NULL |
| INV001 | P001 | 80 | 2024-01-03 00:00 | NULL |

### Follow-Up Questions

**Q: Why check strategy over timestamp for inventory?**
**A:** Inventory may change without an `updated_at` column in the source (e.g., recount). Check strategy detects any data change.

**Q: How do you query current inventory from the snapshot?**
**A:** `WHERE dbt_valid_to IS NULL` gives you the latest version.

**Q: What's the storage cost of periodic snapshots?**
**A:** At Principal level: a daily snapshot over 3 years = 1095 versions per item. For 1M products, that's ~1B rows. Use incremental snapshots and partition by `dbt_valid_from`. Consider compressing old data or using micro-partitioning (Snowflake).

---

## Q17. Write a dbt model that performs a self-join to calculate customer retention (month-over-month) showing which customers are new, returning, or churned.

### Solution

```sql
-- models/marts/customer_retention_analysis.sql

{{ config(materialized='table') }}

-- Monthly customer activity
, monthly_customers as (
    select
        customer_id,
        date_trunc('month', order_date) as activity_month
    from {{ ref('fact_orders') }}
    group by 1, 2
),

-- Pivot: count how many distinct months each customer was active
, customer_months as (
    select
        customer_id,
        activity_month,
        min(activity_month) as first_active_month
    from monthly_customers
    group by 1, 2
),

-- Self-join to check if customers were active in the previous month
, with_prev_month as (
    select
        cm.customer_id,
        cm.activity_month,
        cm.first_active_month,
        case
            when exists (
                select 1
                from monthly_customers prev
                where prev.customer_id = cm.customer_id
                  and prev.activity_month = dateadd(month, -1, cm.activity_month)
            ) then 'returning'
            when cm.activity_month = cm.first_active_month then 'new'
            else 'churned_then_returning'
        end as customer_status
    from customer_months cm
)

select
    activity_month,
    customer_status,
    count(distinct customer_id) as customer_count
from with_prev_month
group by 1, 2
order by 1, 2
```

### Sample Output

| activity_month | customer_status | customer_count |
|---------------|-----------------|---------------|
| 2024-01-01 | new | 1200 |
| 2024-02-01 | new | 1150 |
| 2024-02-01 | returning | 650 |
| 2024-03-01 | new | 1100 |
| 2024-03-01 | returning | 500 |
| 2024-03-01 | churned_then_returning | 45 |

### Follow-Up Questions

**Q: How do you calculate retention rate per cohort?**
**A:** Join with a cohort table and compute: `count(active_in_month) / count(cohort_size)`.

**Q: What's the performance impact of the EXISTS subquery?**
**A:** At scale, use a proper self-join with `LEFT JOIN` instead:
```sql
left join monthly_customers prev
    on prev.customer_id = cm.customer_id
    and prev.activity_month = dateadd(month, -1, cm.activity_month)
```

**Q: How do you handle customers who purchase multiple times per month?**
**A:** The `GROUP BY customer_id, activity_month` deduplicates them to one row per month.
---

## Q18. Write a dbt model that uses QUALIFY clause (Snowflake) or equivalent to keep only the latest record per group, handling tie-breaking with multiple columns.

### Scenario

Raw events data has duplicate records from retry logic. You need the latest record per event_id, with deterministic tie-breaking.

```sql
-- models/staging/stg_events_latest.sql

{{ config(materialized='view') }}

-- Snowflake approach using QUALIFY
{% if target.type == 'snowflake' %}

select *
from {{ ref('stg_events_raw') }}
qualify row_number() over (
    partition by event_id
    order by
        event_timestamp desc,
        received_at desc,
        event_id asc
) = 1

{% endif %}

-- BigQuery / Postgres approach (no QUALIFY)
{% if target.type in ('bigquery', 'postgres') %}

with ranked as (
    select
        *,
        row_number() over (
            partition by event_id
            order by
                event_timestamp desc,
                received_at desc,
                event_id asc
        ) as rn
    from {{ ref('stg_events_raw') }}
)

select *
except (rn)
from ranked
where rn = 1

{% endif %}
```

### Tie-Breaking Strategy

| Priority | Column | Purpose |
|----------|--------|---------|
| 1 | `event_timestamp DESC` | Get the latest event time |
| 2 | `received_at DESC` | Break ties when timestamps are identical |
| 3 | `event_id ASC` | Deterministic final tie-breaker |

### Follow-Up Questions

**Q: Why not use DISTINCT ON in Postgres?**
**A:** `DISTINCT ON (event_id) ORDER BY event_timestamp DESC` works for single-column ordering but is harder to read with multi-column tie-breaking. `ROW_NUMBER()` is more portable.

**Q: What's the performance difference between QUALIFY and subquery + WHERE?**
**A:** QUALIFY is evaluated as part of the window function (one pass). Subquery + WHERE requires materializing the ranked CTE first. QUALIFY is typically 10-20% faster.

**Q: How do you handle events where `event_timestamp` is NULL?**
**A:** `ORDER BY event_timestamp DESC NULLS LAST` to push NULLs to the end, or filter them in a WHERE clause upstream.

---

## Q19. Write a dbt model that pivots monthly sales data from rows to columns, then unpivots it back for a normalized output, demonstrating both operations.

### Pivot (Rows to Columns)

```sql
-- models/marts/sales_pivot_by_month.sql

{{ config(materialized='table') }}

with monthly_sales as (
    select
        product_category,
        format(order_date, 'YYYY-MM') as month,
        sum(revenue) as monthly_revenue
    from {{ ref('fact_orders') }}
    group by 1, 2
)

select
    product_category,
    sum(case when month = '2024-01' then monthly_revenue end) as revenue_2024_01,
    sum(case when month = '2024-02' then monthly_revenue end) as revenue_2024_02,
    sum(case when month = '2024-03' then monthly_revenue end) as revenue_2024_03,
    sum(case when month = '2024-04' then monthly_revenue end) as revenue_2024_04,
    sum(case when month = '2024-05' then monthly_revenue end) as revenue_2024_05,
    sum(case when month = '2024-06' then monthly_revenue end) as revenue_2024_06
from monthly_sales
group by product_category
```

### Unpivot (Columns back to Rows)

```sql
-- models/marts/sales_unpivoted.sql

{{ config(materialized='view') }}

select
    product_category,
    '2024-01' as month, revenue_2024_01 as monthly_revenue
from {{ ref('sales_pivot_by_month') }}
union all
select
    product_category,
    '2024-02' as month, revenue_2024_02 as monthly_revenue
from {{ ref('sales_pivot_by_month') }}
union all
select
    product_category,
    '2024-03' as month, revenue_2024_03 as monthly_revenue
from {{ ref('sales_pivot_by_month') }}
union all
select
    product_category,
    '2024-04' as month, revenue_2024_04 as monthly_revenue
from {{ ref('sales_pivot_by_month') }}
union all
select
    product_category,
    '2024-05' as month, revenue_2024_05 as monthly_revenue
from {{ ref('sales_pivot_by_month') }}
union all
select
    product_category,
    '2024-06' as month, revenue_2024_06 as monthly_revenue
from {{ ref('sales_pivot_by_month') }}
```

### Snowflake UNPIVOT (Native)

```sql
-- Alternative for Snowflake
select
    product_category,
    month,
    monthly_revenue
from {{ ref('sales_pivot_by_month') }}
unpivot (monthly_revenue for month in
    (revenue_2024_01 as '2024-01',
     revenue_2024_02 as '2024-02',
     revenue_2024_03 as '2024-03',
     revenue_2024_04 as '2024-04',
     revenue_2024_05 as '2024-05',
     revenue_2024_06 as '2024-06')
)
```

### Follow-Up Questions

**Q: Is UNPIVOT supported in all databases?**
**A:** No. Snowflake has native `UNPIVOT`. BigQuery and Postgres require `UNION ALL`. Use a cross-database macro to abstract it.

**Q: Why would you unpivot data you already pivoted?**
**A:** Downstream tools (Tableau, Looker) often prefer long/narrow formats. Also needed for time-series analysis.

---

## Q20. Write a dbt model that uses CTEs to progressively aggregate event data from raw events → daily per-user → weekly per-user → monthly totals.

### Solution

```sql
-- models/marts/weekly_user_activity_summary.sql

{{ config(materialized='table') }}

-- Layer 1: Raw events to daily per-user
, daily_user_activity as (
    select
        user_id,
        date_trunc('day', event_timestamp)::date as activity_date,
        count(*) as event_count,
        count(distinct event_type) as distinct_event_types,
        count(distinct session_id) as session_count
    from {{ ref('stg_events') }}
    group by 1, 2
),

-- Layer 2: Daily to weekly per-user
, weekly_user_activity as (
    select
        user_id,
        date_trunc('week', activity_date)::date as activity_week,
        sum(event_count) as weekly_events,
        count(distinct activity_date) as active_days,
        max(daily_events) as max_daily_events,
        avg(daily_events) as avg_daily_events,
        sum(distinct_event_types) as weekly_event_types,
        sum(session_count) as weekly_sessions
    from daily_user_activity
    group by 1, 2
),

-- Layer 3: Weekly to monthly summary
, monthly_summary as (
    select
        user_id,
        date_trunc('month', activity_week)::date as activity_month,
        sum(weekly_events) as monthly_events,
        sum(active_days) as monthly_active_days,
        max(max_daily_events) as peak_daily_events,
        avg(avg_daily_events) as avg_daily_activity,
        max(weekly_sessions) as peak_weekly_sessions,
        sum(weekly_sessions) as monthly_sessions,
        count(distinct activity_week) as active_weeks
    from weekly_user_activity
    group by 1, 2
)

select * from monthly_summary
```

### CTE Structure

| CTE | Granularity | Purpose |
|-----|-----------|---------|
| `daily_user_activity` | Daily per user | Raw aggregation |
| `weekly_user_activity` | Weekly per user | Rolling up daily data |
| `monthly_summary` | Monthly per user | Final output |

### Follow-Up Questions

**Q: What's the memory cost of multiple CTEs?**
**A:** Modern warehouses (Snowflake, BigQuery) optimize CTEs as inlined views — no materialization overhead. For older databases, use temp tables.

**Q: How do you handle users with no activity in a week?**
**A:** Cross join with a date spine and fill with zeros.

**Q: When would you NOT use CTEs and use temp tables instead?**
**A:** When intermediate results are expensive to recompute and reused multiple times. Also when debugging — you can run each CTE independently.

---

## Q21. Write a dbt model that generates a date/datetime dimension table for a given year range, including day of week, week number, quarter, fiscal period, and holiday flags.

### Solution

```sql
-- models/marts/dim_date.sql

{{ config(materialized='table') }}

with base_dates as (
    -- Generate dates from 2020-01-01 to 2030-12-31
    {{ generate_date_range('2020-01-01', '2030-12-31') }}
),

enriched as (
    select
        the_date as date_key,
        the_date,

        -- Basic date parts
        extract(year from the_date) as year,
        extract(month from the_date) as month,
        extract(day from the_date) as day_of_month,

        -- Day of week
        extract(dow from the_date) as day_of_week_num,  -- 0=Sunday
        date_part('dow', the_date) as day_of_week_0,
        to_char(the_date, 'Day') as day_name,

        -- Week calculations
        extract(week from the_date) as week_number,
        date_trunc('week', the_date)::date as week_start_date,
        date_trunc('month', the_date)::date as month_start_date,

        -- Quarter
        extract(quarter from the_date) as quarter,
        'Q' || extract(quarter from the_date) as quarter_label,

        -- Year-month combinations for filtering
        to_char(the_date, 'YYYY-MM') as year_month,
        to_char(the_date, 'YYYY-QQ') as year_quarter,

        -- Last day of month?
        case when the_date = last_day(the_date) then true else false end as is_last_day_of_month,

        -- Is it a weekend?
        case when extract(dow from the_date) in (0, 6) then true else false end as is_weekend,

        -- Is it the last week of the month?
        case
            when the_date >= last_day(the_date) - interval '6 days'
            then true
            else false
        end as is_last_week_of_month

    from base_dates
)

select * from enriched
order by date_key
```

### Follow-Up Questions

**Q: How do you add holiday flags?**
**A:** Left join with a holidays seed:
```sql
left join {{ ref('holidays') }} h
    on d.date_key = h.holiday_date
    and d.year = h.year
```

**Q: How do you handle fiscal years that don't align with calendar years?**
**A:** Add a fiscal calendar:
```sql
case
    when month >= 4 then extract(year from the_date) + 1
    else extract(year from the_date)
end as fiscal_year
```

**Q: Should dim_date be a table or a view?**
**A:** Table — it's small (~4000 rows for 10 years), static, and joins frequently.

---

## Q22. Write a dbt model that performs a slow-moving dimension type 2 (SCD2) using a manual approach with a staging table that has natural keys and change detection.

### Solution

```sql
-- models/marts/dim_products_scd2.sql

{{ config(
    materialized='incremental',
    unique_key='surrogate_key'
) }}

with source as (
    select
        product_id,
        product_name,
        category,
        price,
        status,
        last_updated
    from {{ ref('stg_products') }}
),

-- Existing records in the SCD2 dimension
, existing as (
    select
        surrogate_key,
        product_id,
        product_name,
        category,
        price,
        status,
        valid_from,
        valid_to
    from {{ this }}
    where valid_to is null  -- Only current rows
),

-- Detect changes by comparing with current valid rows
, changes as (
    select
        s.product_id,
        s.product_name,
        s.category,
        s.price,
        s.status,
        s.last_updated,
        case
            when e.surrogate_key is null then 'NEW'
            when e.product_name != s.product_name
                 or e.category != s.category
                 or e.price != s.price
                 or e.status != s.status
            then 'CHANGED'
            else 'UNCHANGED'
        end as change_type
    from source s
    left join existing e
        on s.product_id = e.product_id
),

-- Close out changed records
, closed_rows as (
    select
        e.surrogate_key,
        e.product_id,
        e.product_name,
        e.category,
        e.price,
        e.status,
        e.valid_from,
        current_timestamp::timestamp as valid_to
    from changes c
    join existing e on c.product_id = e.product_id
    where c.change_type = 'CHANGED'
),

-- New rows for NEW and CHANGED records
, new_rows as (
    select
        gen_randomuuid()::uuid as surrogate_key,
        c.product_id,
        c.product_name,
        c.category,
        c.price,
        c.status,
        current_timestamp::timestamp as valid_from,
        null as valid_to
    from changes c
    where c.change_type in ('NEW', 'CHANGED')
),

-- Unchanged rows (keep as-is)
, unchanged as (
    select
        e.surrogate_key,
        e.product_id,
        e.product_name,
        e.category,
        e.price,
        e.status,
        e.valid_from,
        e.valid_to
    from changes c
    join existing e on c.product_id = e.product_id
    where c.change_type = 'UNCHANGED'
)

-- Union all parts
select * from closed_rows
union all
select * from new_rows
union all
select * from unchanged
```

### How It Works

| Part | Purpose |
|------|---------|
| `changes` CTE | Detects NEW, CHANGED, UNCHANGED records |
| `closed_rows` | Sets `valid_to` on old versions |
| `new_rows` | Creates new versions with fresh `valid_from` |
| `unchanged` | Keeps existing rows untouched |

### Follow-Up Questions

**Q: Why not use dbt snapshots?**
**A:** dbt snapshots are simpler but less flexible. Manual SCD2 gives you (1) custom surrogate keys, (2) custom change detection logic, (3) integration with external change tracking.

**Q: How do you handle deletions?**
**A:** Use an anti-join to detect source records that no longer exist, then close them with `valid_to = current_timestamp`.

**Q: What's the impact on downstream models?**
**A:** Downstream models must join on `product_id AND valid_to IS NULL` to get current values.

---

## Q23. Write a dbt model that calculates customer lifetime value (CLV) with multiple metrics: total revenue, average order value, purchase frequency, and customer tenure in months.

### Solution

```sql
-- models/marts/customer_lifetime_value.sql

{{ config(materialized='incremental', unique_key='customer_id') }}

with customer_orders as (
    select
        customer_id,
        order_id,
        order_date,
        revenue,
        units_sold
    from {{ ref('fact_orders') }}
    {% if is_incremental() %}
    where order_date >= (
        select max(order_date) from {{ this }}
    )
    {% endif %}
),

-- Aggregate customer metrics
, customer_metrics as (
    select
        customer_id,
        count(distinct order_id) as total_orders,
        sum(revenue) as total_revenue,
        avg(revenue) as avg_order_value,
        sum(units_sold) as total_units,

        -- Customer tenure
        min(order_date)::date as first_order_date,
        max(order_date)::date as last_order_date,
        datediff(month, min(order_date)::date, max(order_date)::date) as tenure_months,

        -- Purchase frequency (orders per month)
        case
            when datediff(month, min(order_date)::date, max(order_date)::date) > 0
            then count(distinct order_id) * 1.0
                 / datediff(month, min(order_date)::date, max(order_date)::date)
            else count(distinct order_id)
        end as orders_per_month,

        -- Average days between orders
        case
            when count(distinct order_id) > 1
            then datediff(day, min(order_date), max(order_date)) * 1.0
                 / (count(distinct order_id) - 1)
            else 0
        end as avg_days_between_orders

    from customer_orders
    group by 1
),

-- Calculate CLV
, clv as (
    select
        customer_id,
        total_orders,
        total_revenue,
        round(avg_order_value, 2) as avg_order_value,
        total_units,
        first_order_date,
        last_order_date,
        tenure_months,
        round(orders_per_month, 2) as orders_per_month,
        round(avg_days_between_orders, 1) as avg_days_between_orders,

        -- CLV = total revenue (historical)
        round(total_revenue, 2) as historical_clv,

        -- Projected CLV (assuming 12 more months)
        round(avg_order_value * orders_per_month * 12, 2) as projected_12m_clv,

        -- Recency score (days since last order)
        datediff(day, last_order_date, current_date::date) as days_since_last_order

    from customer_metrics
)

select * from clv
```

### Sample Output

| customer_id | total_orders | total_revenue | avg_order_value | tenure_months | historical_clv | projected_12m_clv |
|------------|-------------|--------------|----------------|--------------|---------------|------------------|
| C001 | 45 | 12,500.00 | 277.78 | 36 | 12,500.00 | 11,904.00 |
| C002 | 12 | 3,200.00 | 266.67 | 8 | 3,200.00 | 5,000.00 |
| C003 | 89 | 28,400.00 | 319.10 | 48 | 28,400.00 | 18,504.00 |

### Follow-Up Questions

**Q: How do you handle customers with only one order?**
**A:** The `CASE` handles it: `orders_per_month` = total orders (no division by zero).

**Q: How do you incorporate churn probability?**
**A:** Join with a churn prediction model's output and weight: `projected_12m_clv * (1 - churn_probability)`.

**Q: What about discounting future revenue?**
**A:** Apply a discount rate: `projected_12m_clv / (1 + discount_rate)^months`.
---

## Q24. Write a dbt model that creates a summary aggregation table with multiple GROUPING SETS (subtotal by category, by region, and grand total) in a single query.

### Solution

```sql
-- models/marts/sales_summary_with_subtotals.sql

{{ config(materialized='table') }}

select
    coalesce(category, 'ALL_CATEGORIES') as category,
    coalesce(region, 'ALL_REGIONS') as region,
    coalesce(year_month, 'ALL_PERIODS') as year_month,
    count(distinct order_id) as order_count,
    sum(revenue) as total_revenue,
    sum(units_sold) as total_units,
    avg(revenue) as avg_order_revenue
from {{ ref('fact_orders') }}
group by grouping sets (
    (category, region, year_month),   -- Detailed
    (category, region),                -- Subtotal by category + region
    (category),                        -- Subtotal by category
    (region),                          -- Subtotal by region
    ()                                 -- Grand total
)
order by
    category nulls last,
    region nulls last,
    year_month nulls last
```

### Output Example

| category | region | year_month | order_count | total_revenue |
|---------|--------|-----------|------------|--------------|
| Electronics | North | 2024-01 | 150 | 45,000 |
| Electronics | South | 2024-01 | 120 | 38,000 |
| Electronics | ALL_REGIONS | 2024-01 | 270 | 83,000 |
| Electronics | ALL_REGIONS | ALL_PERIODS | 3240 | 996,000 |
| ALL_CATEGORIES | ALL_REGIONS | ALL_PERIODS | 8500 | 2,500,000 |

### Follow-Up Questions

**Q: What's the difference between GROUPING SETS, ROLLUP, and CUBE?**
**A:**
- `ROLLUP(a, b, c)` = `(a,b,c), (a,b), (a), ()` — hierarchical from left to right
- `CUBE(a, b, c)` = all 8 combinations — every possible grouping
- `GROUPING SETS` — explicit control over which combinations

**Q: How do you identify subtotal rows vs detail rows?**
**A:** Use `GROUPING_ID()` or `GROUPING(category)` which returns 1 for aggregated rows:
```sql
GROUPING(category) as is_category_subtotal,
GROUPING(region) as is_region_subtotal
```

**Q: Is GROUPING SETS supported in all databases?**
**A:** Not natively. BigQuery supports it; Snowflake supports it. Postgres does not — you need `UNION ALL` of separate GROUP BY queries.

---

## Q25. Write a dbt model that uses LATERAL JOIN (or equivalent CROSS JOIN LATERAL) to unnest a JSON array of order items into individual rows.

### Solution

```sql
-- models/staging/stg_order_items_unnested.sql

{{ config(materialized='view') }}

-- Snowflake with LATERAL FLATTEN
{% if target.type == 'snowflake' %}

select
    o.order_id,
    o.customer_id,
    o.order_date,
    f.value:item_id::varchar as item_id,
    f.value:product_id::varchar as product_id,
    f.value:quantity::integer as quantity,
    f.value:unit_price::float as unit_price,
    f.value:quantity::integer * f.value:unit_price::float as line_total
from {{ ref('stg_orders') }} o,
     lateral flatten(input => o.items_json) f

{% endif %}

-- Postgres with jsonb_array_elements
{% if target.type == 'postgres' %}

select
    o.order_id,
    o.customer_id,
    o.order_date,
    item->>'item_id' as item_id,
    item->>'product_id' as product_id,
    (item->>'quantity')::integer as quantity,
    (item->>'unit_price')::float as unit_price,
    (item->>'quantity')::integer * (item->>'unit_price')::float as line_total
from {{ ref('stg_orders') }} o,
     jsonb_array_elements(o.items_json::jsonb) as item

{% endif %}

-- BigQuery with UNNEST
{% if target.type == 'bigquery' %}

select
    o.order_id,
    o.customer_id,
    o.order_date,
    JSON_VALUE(item, '$.item_id') as item_id,
    JSON_VALUE(item, '$.product_id') as product_id,
    SAFE_CAST(JSON_VALUE(item, '$.quantity') AS INT64) as quantity,
    SAFE_CAST(JSON_VALUE(item, '$.unit_price') AS FLOAT64) as unit_price
from {{ ref('stg_orders') }} o,
     UNNEST(JSON_EXTRACT_ARRAY(o.items_json, '$')) as item

{% endif %}
```

### Sample Data

**Input (stg_orders):**
| order_id | items_json |
|---------|-----------|
| ORD001 | `[{"item_id":"I1","product_id":"P100","quantity":2,"unit_price":25.00},{"item_id":"I2","product_id":"P200","quantity":1,"unit_price":50.00}]` |

**Output (stg_order_items_unnested):**
| order_id | item_id | product_id | quantity | unit_price | line_total |
|---------|---------|-----------|----------|-----------|-----------|
| ORD001 | I1 | P100 | 2 | 25.00 | 50.00 |
| ORD001 | I2 | P200 | 1 | 50.00 | 50.00 |

### Follow-Up Questions

**Q: What if the JSON array is empty?**
**A:** The lateral join produces no rows for that order. Use `LEFT JOIN LATERAL` or a `CROSS JOIN LEFT ASOF` equivalent if you need to keep the parent row.

**Q: How do you handle nested arrays within the items?**
**A:** Nested flattening requires multiple lateral joins or recursive CTEs.

---

## Q26. Write a dbt model that calculates cohort analysis — grouping customers by their first purchase month and tracking their activity over subsequent months.

### Solution

```sql
-- models/marts/customer_cohort_analysis.sql

{{ config(materialized='table') }}

-- Step 1: Determine each customer's cohort (first purchase month)
, customer_cohorts as (
    select
        customer_id,
        min(date_trunc('month', order_date)::date) as cohort_month
    from {{ ref('fact_orders') }}
    group by 1
),

-- Step 2: Get all monthly activity per customer
, customer_monthly_activity as (
    select
        c.cohort_month,
        f.customer_id,
        date_trunc('month', f.order_date)::date as activity_month,
        count(distinct f.order_id) as orders,
        sum(f.revenue) as revenue
    from customer_cohorts c
    join {{ ref('fact_orders') }} f
        on c.customer_id = f.customer_id
    group by 1, 2, 3
),

-- Step 3: Calculate the period offset (months since cohort)
, with_period as (
    select
        cohort_month,
        activity_month,
        datediff(month, cohort_month, activity_month) as period_month,
        count(distinct customer_id) as active_customers,
        sum(orders) as total_orders,
        sum(revenue) as total_revenue
    from customer_monthly_activity
    group by 1, 2, 3
),

-- Step 4: Calculate cohort sizes
, cohort_sizes as (
    select
        cohort_month,
        count(distinct customer_id) as cohort_size
    from customer_cohorts
    group by 1
),

-- Step 5: Calculate retention rates
select
    p.cohort_month,
    c.cohort_size,
    p.period_month,
    p.active_customers,
    round(p.active_customers * 100.0 / c.cohort_size, 2) as retention_rate_pct,
    p.total_orders,
    round(p.total_revenue, 2) as total_revenue,
    round(p.total_revenue / nullif(p.active_customers, 0), 2) as revenue_per_active_customer
from with_period p
join cohort_sizes c
    on p.cohort_month = c.cohort_month
order by p.cohort_month, p.period_month
```

### Sample Output

| cohort_month | cohort_size | period_month | active_customers | retention_rate_pct |
|-------------|------------|-------------|-----------------|-------------------|
| 2024-01-01 | 1200 | 0 | 1200 | 100.00 |
| 2024-01-01 | 1200 | 1 | 650 | 54.17 |
| 2024-01-01 | 1200 | 2 | 400 | 33.33 |
| 2024-01-01 | 1200 | 3 | 280 | 23.33 |
| 2024-02-01 | 1100 | 0 | 1100 | 100.00 |
| 2024-02-01 | 1100 | 1 | 600 | 54.55 |

### Follow-Up Questions

**Q: How do you handle customers who appear in multiple cohorts?**
**A:** They don't — `MIN(date_trunc('month', order_date))` assigns each customer to exactly one cohort.

**Q: How do you calculate repeat purchase rate?**
**A:** `count(case when orders > 1 then 1 end) / count(*)` per cohort per period.

**Q: How do you visualize this in a BI tool?**
**A:** Use a heatmap with `cohort_month` on one axis, `period_month` on the other, and `retention_rate_pct` as color.

---

## Q27. Write a dbt macro that generates a dynamic cross-tabulation (pivot) where the columns are determined by a config variable.

### Macro

```sql
-- macros/dynamic_pivot.sql

{% macro dynamic_pivot(source, row_column, pivot_column, value_column, pivot_values, agg_function='sum') %}

select
    {{ row_column }},
    {% for val in pivot_values %}
        {{ agg_function }}(case when {{ pivot_column }} = '{{ val }}' then {{ value_column }} end)
        as {{ pivot_column }}_{{ val }}
        {% if not loop.last %},{% endif %}
    {% endfor %}
from {{ source }}
group by {{ row_column }}
order by {{ row_column }}

{% endmacro %}
```

### Usage in a Model

```sql
-- models/marts/category_sales_by_region.sql

{{ config(materialized='table') }}

{{ dynamic_pivot(
    source=ref('fact_orders'),
    row_column='category',
    pivot_column='region',
    value_column='revenue',
    pivot_values=['North', 'South', 'East', 'West'],
    agg_function='sum'
) }}
```

### Equivalent SQL Output

```sql
select
    category,
    sum(case when region = 'North' then revenue end) as region_North,
    sum(case when region = 'South' then revenue end) as region_South,
    sum(case when region = 'East' then revenue end) as region_East,
    sum(case when region = 'West' then revenue end) as region_West
from fact_orders
group by category
order by category
```

### Follow-Up Questions

**Q: How do you make pivot_values dynamic instead of hardcoded?**
**A:** Pass via `var()`:
```sql
pivot_values=var('regions', ['North', 'South', 'East', 'West'])
```

**Q: What if new values appear in the source data?**
**A:** You need to update the pivot_values list. For fully dynamic pivots, use database-specific PIVOT syntax or generate the list via a query.

---

## Q28. Write a dbt model that calculates year-over-year (YoY) and month-over-month (MoM) growth rates with proper handling for NULL values and division by zero.

### Solution

```sql
-- models/marts/sales_growth_rates.sql

{{ config(materialized='table') }}

, monthly_sales as (
    select
        date_trunc('month', order_date)::date as month,
        sum(revenue) as revenue,
        count(distinct order_id) as order_count,
        count(distinct customer_id) as customer_count
    from {{ ref('fact_orders') }}
    group by 1
),

with_growth as (
    select
        month,
        revenue,
        order_count,
        customer_count,

        -- Previous month values
        lag(revenue) over (order by month) as prev_month_revenue,
        lag(order_count) over (order by month) as prev_month_orders,
        lag(customer_count) over (order by month) as prev_month_customers,

        -- Previous year values (12 months ago)
        lag(revenue, 12) over (order by month) as prev_year_revenue,
        lag(order_count, 12) over (order by month) as prev_year_orders,

        -- Month-over-Month growth
        case
            when lag(revenue) over (order by month) is null then null
            when lag(revenue) over (order by month) = 0 then null
            else (revenue - lag(revenue) over (order by month))
                 / lag(revenue) over (order by month)
        end as mom_revenue_growth,

        -- Year-over-Year growth
        case
            when lag(revenue, 12) over (order by month) is null then null
            when lag(revenue, 12) over (order by month) = 0 then null
            else (revenue - lag(revenue, 12) over (order by month))
                 / lag(revenue, 12) over (order by month)
        end as yoy_revenue_growth

    from monthly_sales
)

select
    month,
    revenue,
    order_count,
    customer_count,
    round(mom_revenue_growth * 100, 2) as mom_revenue_growth_pct,
    round(yoy_revenue_growth * 100, 2) as yoy_revenue_growth_pct,
    prev_month_revenue,
    prev_year_revenue
from with_growth
order by month
```

### Sample Output

| month | revenue | mom_revenue_growth_pct | yoy_revenue_growth_pct |
|------|---------|----------------------|----------------------|
| 2023-01-01 | 100,000 | NULL | NULL |
| 2023-02-01 | 110,000 | 10.00 | NULL |
| 2024-01-01 | 125,000 | 15.00 | 25.00 |
| 2024-02-01 | 130,000 | 4.00 | 18.18 |

### Follow-Up Questions

**Q: Why use CASE instead of a safe_divide macro?**
**A:** Either works. Using the `safe_divide` macro:
```sql
safe_divide(revenue - prev_month_revenue, prev_month_revenue) as mom_revenue_growth
```
is cleaner and more maintainable.

**Q: How do you handle seasonality?**
**A:** YoY growth naturally handles seasonality by comparing the same month year-over-year. MoM growth can be misleading during seasonal spikes.

---

## Q29. Write a dbt model that joins fact and dimension tables with a many-to-many relationship using an intermediate bridge table to avoid fan-out issues.

### Scenario

Products can belong to multiple categories, and categories can have multiple products. A direct join would cause revenue to be counted multiple times.

### Solution

```sql
-- models/marts/fact_category_sales_deduplicated.sql

{{ config(materialized='table') }}

, order_items as (
    select
        o.order_id,
        o.customer_id,
        o.order_date,
        oi.product_id,
        oi.quantity,
        oi.revenue as line_revenue
    from {{ ref('fact_orders') }} o
    join {{ ref('stg_order_items') }} oi
        on o.order_id = oi.order_id
),

-- Assign each product-category combination a row number
-- to ensure each revenue unit is counted only once per category
, product_categories as (
    select
        pc.product_id,
        pc.category_id,
        pc.category_name
    from {{ ref('stg_product_category_bridge') }} pc
),

-- For products in multiple categories, split revenue equally
, category_split as (
    select
        product_id,
        category_id,
        category_name,
        -- Count how many categories this product belongs to
        count(category_id) over (partition by product_id) as category_count
    from product_categories
),

-- Join with revenue
, category_sales as (
    select
        cs.category_id,
        cs.category_name,
        oi.order_id,
        oi.customer_id,
        oi.order_date,
        oi.product_id,
        oi.quantity,
        -- Split revenue equally across categories
        oi.line_revenue / cs.category_count as allocated_revenue
    from order_items oi
    join category_split cs
        on oi.product_id = cs.product_id
)

select
    category_id,
    category_name,
    order_date,
    count(distinct order_id) as order_count,
    count(distinct customer_id) as customer_count,
    sum(allocated_revenue) as category_revenue,
    sum(quantity) as total_units
from category_sales
group by 1, 2, 3
order by 1, 3
```

### How It Prevents Fan-Out

| Scenario | Problem | Solution |
|----------|---------|----------|
| Product P1 in Categories A, B | Join doubles revenue | Split revenue: each gets 50% |
| Product P2 in Categories A, B, C | Join triples revenue | Split revenue: each gets 33% |

### Follow-Up Questions

**Q: Is equal split the best approach?**
**A:** It's the simplest. At Principal level: use a business rule (primary category gets 100%, or allocate by revenue proportion if available).

**Q: How do you handle products with no category?**
**A:** Add an 'Uncategorized' category in the bridge table, or filter with `WHERE category_id IS NOT NULL`.

---

## Q30. Write a comprehensive dbt model that combines multiple techniques (CTEs, window functions, conditional logic, self-joins) to produce a customer RFM (Recency, Frequency, Monetary) segmentation.

### Solution

```sql
-- models/marts/customer_rfm_segmentation.sql

{{ config(materialized='table') }}

-- Step 1: Calculate base RFM metrics per customer
, rfm_base as (
    select
        customer_id,
        -- Recency: days since last purchase
        datediff(day, max(order_date), current_date::date) as recency_days,
        -- Frequency: total number of orders
        count(distinct order_id) as frequency,
        -- Monetary: total revenue
        sum(revenue) as monetary,
        -- Average order value
        avg(revenue) as avg_order_value,
        -- First and last purchase dates
        min(order_date)::date as first_purchase_date,
        max(order_date)::date as last_purchase_date
    from {{ ref('fact_orders') }}
    group by 1
),

-- Step 2: Assign R, F, M scores (1-5) using NTILE
, rfm_scored as (
    select
        customer_id,
        recency_days,
        frequency,
        monetary,
        avg_order_value,
        first_purchase_date,
        last_purchase_date,

        -- Recency score (lower recency = higher score)
        ntile(5) over (order by recency_days asc) as r_score,

        -- Frequency score (higher frequency = higher score)
        ntile(5) over (order by frequency asc) as f_score,

        -- Monetary score (higher monetary = higher score)
        ntile(5) over (order by monetary asc) as m_score
    from rfm_base
),

-- Step 3: Calculate combined RFM score and segment
, rfm_segmented as (
    select
        customer_id,
        recency_days,
        frequency,
        round(monetary, 2) as monetary,
        round(avg_order_value, 2) as avg_order_value,
        first_purchase_date,
        last_purchase_date,
        r_score,
        f_score,
        m_score,
        r_score + f_score + m_score as rfm_total_score,

        -- Customer segmentation
        case
            when r_score >= 4 and f_score >= 4 and m_score >= 4 then 'Champions'
            when r_score >= 4 and f_score >= 3 then 'Loyal Customers'
            when r_score >= 3 and f_score >= 3 then 'Potential Loyalists'
            when r_score >= 4 and f_score <= 2 then 'New Customers'
            when r_score >= 2 and f_score >= 3 then 'Promising'
            when r_score >= 2 and f_score <= 2 then 'About to Sleep'
            when r_score <= 1 and f_score >= 3 then 'Cant Lose Them'
            when r_score <= 1 and f_score <= 1 then 'Lost'
            else 'Need Attention'
        end as customer_segment
    from rfm_scored
)

select
    *,
    -- Additional metrics
    case
        when recency_days <= 30 then 'Active'
        when recency_days <= 90 then 'At Risk'
        else 'Churned'
    end as activity_status
from rfm_segmented
order by rfm_total_score desc
```

### Sample Output

| customer_id | recency_days | frequency | monetary | r_score | f_score | m_score | rfm_total_score | customer_segment |
|------------|-------------|-----------|---------|---------|---------|---------|----------------|-----------------|
| C001 | 5 | 45 | 12,500.00 | 5 | 5 | 5 | 15 | Champions |
| C002 | 30 | 12 | 3,200.00 | 4 | 3 | 3 | 10 | Loyal Customers |
| C003 | 180 | 3 | 800.00 | 1 | 1 | 1 | 3 | Lost |
| C004 | 15 | 5 | 1,500.00 | 4 | 2 | 2 | 8 | New Customers |

### Techniques Demonstrated

| Technique | Where Used |
|-----------|-----------|
| CTEs | 4 progressive layers |
| Aggregation | Base RFM calculation |
| Window Functions (NTILE) | Score assignment |
| Conditional Logic (CASE) | Segmentation |
| Derived Metrics | Activity status |

### Follow-Up Questions

**Q: Why NTILE(5) instead of custom thresholds?**
**A:** NTILE automatically adapts to the data distribution. Custom thresholds (e.g., recency < 30 = 5) are more interpretable but require manual tuning. At Principal level, use NTILE for data-driven scoring, custom thresholds for business-aligned scoring.

**Q: How do you make the segmentation dynamic?**
**A:** Use `var()` for the segment thresholds and `config()` for the number of NTILE buckets.

**Q: How often should RFM be recalculated?**
**A:** Weekly or monthly, depending on business velocity. Use incremental materialization for efficiency.

**Q: What are the limitations of RFM?**
**A:** RFM doesn't account for (1) product category, (2) return rate, (3) profit margin, (4) customer interaction channels. At Principal level: extend RFM to RFM-P (with Profit) or use ML-based clustering (K-Means) for more granular segmentation.

---

## Summary

This guide covers 30 practical dbt questions designed for Principal Data Engineer interviews. Each question demonstrates:

- **Real-world scenarios** you'll encounter in production
- **Multiple implementation approaches** with trade-offs
- **Cross-database compatibility** considerations
- **Edge case handling** (NULLs, division by zero, missing data)
- **Performance considerations** at scale
- **Follow-up questions** with detailed answers

### Key Skills Demonstrated

| Skill Area | Questions |
|-----------|-----------|
| Seeds & Reference Data | Q1, Q2 |
| Incremental Strategies | Q3, Q11 |
| Snapshots & SCD2 | Q4, Q16, Q22 |
| Macros & Cross-DB | Q5, Q8, Q9, Q13 |
| Window Functions | Q7, Q14, Q18, Q28 |
| Custom Testing | Q6, Q15 |
| Advanced SQL | Q10, Q19, Q20, Q24, Q25 |
| Analytics Patterns | Q12, Q17, Q21, Q23, Q26, Q30 |
| Architecture & Design | Q27, Q29 |

### Interview Tips for Principal-Level

1. **Always discuss trade-offs** — never present one solution as "the answer"
2. **Mention scale** — how does your solution handle 1B+ rows?
3. **Discuss observability** — how do you monitor data quality?
4. **Talk about team impact** — how does this approach help other engineers?
5. **Address cost** — what's the compute/storage cost of your approach?
6. **Mention alternatives** — "I chose X, but Y would work if..."
7. **Reference real experience** — tie answers to your 17 years of production work

Good luck with your interviews! 🚀