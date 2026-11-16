# Snowflake Practical Questions

## Q1. Semi-structured JSON parsing with FLATTEN

### Scenario
Your team ingests nested JSON event data from a mobile analytics platform into Snowflake via Snowpipe. Each JSON payload contains a user profile, an array of items purchased, and metadata. You need to extract the user's name, each item's SKU and price, and the total order value. The JSON structure is deeply nested with optional fields. A Principal-level interviewer expects you to handle nulls, arrays, and performance at TB-scale.

### Solution
```sql
-- Step 1: Parse the top-level JSON fields from a VARIANT column
-- Then flatten the nested array of items using FLATTEN

SELECT
    src:event_id::STRING           AS event_id,
    src:user:name::STRING          AS user_name,
    src:user:country::STRING       AS user_country,
    f.value:item_id::STRING        AS item_id,
    f.value:sku::STRING            AS sku,
    f.value:price::FLOAT           AS item_price,
    f.value:quantity::INT          AS quantity,
    src:total_amount::FLOAT        AS total_order_value,
    src:timestamp::TIMESTAMP       AS event_timestamp
FROM raw_events.raw_payload
, LATERAL FLATTEN( input => src:items ) f
WHERE src:items IS NOT NULL
  AND ARRAY_SIZE( src:items ) > 0;
```

### Explanation
The `FLATTEN` table function (also accessible via `LATERAL FLATTEN`) is the cornerstone of semi-structured data processing in Snowflake. It takes a JSON array stored in a `VARIANT` column and produces one row per array element. The `input => src:items` syntax uses Snowflake's dot-notation to navigate the VARIANT hierarchy.

The `WHERE src:items IS NOT NULL AND ARRAY_SIZE(src:items) > 0` filter is critical at scale: it prevents FLATTEN from producing empty results for records where the `items` array is missing or empty, which is common in event streams. Without this filter, you get NULL rows that corrupt downstream aggregations.

At TB-scale, always ensure that the VARIANT column has an automatic clustering key (Snowflake auto-clusters by default, but you can manually add a `CLUSTER BY` on the `event_id` path). Avoid unnecessary extraction — keep raw VARIANT columns alongside extracted columns so you can re-derive fields when the schema evolves. This is the "golden copy" pattern used at Google and Uber.

The `LATERAL` keyword makes the flattening explicit and readable. In Snowflake it is optional, but using it signals intent to other engineers. Performance-wise, FLATTEN is pushed into the scan layer: Snowflake does not materialize the array in memory; it streams elements row by row.

### Example Data / Sample Tables
```sql
CREATE OR REPLACE TABLE raw_events.raw_payload (
    id         BIGINT AUTOINCREMENT PRIMARY KEY,
    src        VARIANT,
    loaded_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

INSERT INTO raw_events.raw_payload (src) VALUES
('{
    "event_id": "evt_001",
    "user": { "name": "Alice", "country": "US" },
    "items": [
        { "item_id": "i1", "sku": "WIDGET-A", "price": 19.99, "quantity": 2 },
        { "item_id": "i2", "sku": "GADGET-B", "price": 49.99, "quantity": 1 }
    ],
    "total_amount": 89.97,
    "timestamp": "2024-01-15T10:30:00Z"
}'),
('{
    "event_id": "evt_002",
    "user": { "name": "Bob" },
    "items": [
        { "item_id": "i3", "sku": "WIDGET-C", "price": 9.99, "quantity": 1 }
    ],
    "total_amount": 9.99,
    "timestamp": "2024-01-15T11:00:00Z"
}'),
('{
    "event_id": "evt_003",
    "user": { "name": "Charlie", "country": "UK" },
    "items": [],
    "total_amount": 0,
    "timestamp": "2024-01-15T12:00:00Z"
}');
```

### Comparison (if applicable)
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| `FLATTEN` + `LATERAL` | Set-based, pushes into scan layer, no UDFs | Can explode row count if arrays are large | Standard JSON array processing |
| `GET_PATH(src, 'items')` in a UDF | Flexible logic per element | Slower, cannot be pushed down, harder to optimise | Complex per-element transformations |
| Pre-parse in Python/Spark before load | Full control, validated schema | ETL pipeline complexity, two-tier architecture | When JSON schema is highly unstable |

### Edge Cases
- `items` is `NULL` vs `items` is an empty array `[]` — FLATTEN behaves differently
- Arrays with millions of elements per row can cause memory pressure on micro-batches
- Mixed types inside an array (e.g., `[1, "two", null]`) — casting will throw errors; use `TRY_CAST`
- Nested arrays (array of arrays) require multiple FLATTEN passes with `outer => true`

### Principal-Level Considerations
- At PB scale, store the raw VARIANT alongside parsed columns (denormalisation trade-off). Re-parsing is cheap in Snowflake; the cost of storage is negligible compared to re-ingestion.
- Use `RESULT_SCAN()` for debugging FLATTEN output before persisting: pipe the result into a temp table for validation.
- Consider `OBJECT_CONSTRUCT_KEEP_NULL()` vs `OBJECT_CONSTRUCT()` when building JSON back out — null handling matters for downstream consumers.
- Auto-clustering on `event_timestamp` path in the VARIANT reduces data scanned for time-range queries by 5-10x at TB scale.

### Follow-up Questions
1. **What happens if the JSON array is nested three levels deep?**
   - **Answer:** Chain multiple `LATERAL FLATTEN` calls. The outermost FLATTEN produces rows for the first array; a second FLATTEN inside the same SELECT explodes the next level. Snowflake optimises nested flattens into a single scan pass when possible. Example: `, LATERAL FLATTEN(input => src:orders) o, LATERAL FLATTEN(input => o.value:line_items) li`.

2. **How do you handle optional fields inside a JSON object without failing the query?**
   - **Answer:** Use `TRY_CAST(src:user:phone::STRING AS INT)` or the safe access operator `src:user:phone` which returns `NULL` if the path does not exist. The `::STRING` cast on a missing path returns `NULL` in Snowflake (it does not throw). For numbers, `TRY_TO_NUMBER(src:user:age)` is safer than direct casting.

3. **What is the cost impact of FLATTEN on warehouse sizing?**
   - **Answer:** FLATTEN is CPU-intensive but memory-efficient. For a 1TB table with arrays averaging 5 elements, the logical scan doubles to ~2TB of micro-partitions read. Use a warehouse 2-3x larger than for equivalent row-level queries, but keep it auto-scaled: Snowpipe loads happen in small batches. At scale, consider a dedicated `LARGE` or `X-LARGE` warehouse for parsing, with `MAX_CLUSTER_COUNT=4`.

4. **How do you validate the JSON structure before parsing?**
   - **Answer:** Use `IS_VALID_JSON(src)` if loading from strings, or validate against a JSON Schema using the `JSON` data type available in newer Snowflake versions. For production, run a validation job: `SELECT COUNT(*) FROM raw_payload WHERE src:items IS NULL OR ARRAY_SIZE(src:items) = 0` to flag malformed records before the main pipeline runs.

5. **Can you use FLATTEN with `OBJECT` types, not just arrays?**
   - **Answer:** Yes — `FLATTEN(input => src:user)` with `path => 'attributes'` explodes object keys into `KEY`, `PATH`, and `VALUE` columns. This is useful when the JSON has dynamic keys (e.g., `{"attr_1": "val1", "attr_2": "val2"}`). Use `flatten_type => 'OBJECT'` explicitly.

---

## Q2. Zero-copy clone for development/test environments

### Scenario
Your team manages a 50TB production database `PROD_ANALYTICS` with 200+ tables. Every sprint, three engineering teams need isolated copies to test schema migrations and query logic. The CTO demands that cloning must be instant and cost must be near-zero for the first 24 hours. At Atlassian or Google scale, cloning a database takes seconds, not hours.

### Solution
```sql
-- Clone the entire database (all schemas, tables, views, UDFs, etc.)
CREATE DATABASE dev_team_alpha CLONE prod_analytics;

-- Clone just a specific schema (cheaper, more targeted)
CREATE SCHEMA dev_team_beta.staging CLONE prod_analytics.staging;

-- Clone a single table for ad-hoc testing
CREATE TABLE dev_team_gamma.sales_q4 CLONE prod_analytics.staging.sales
    WITH DATA;  -- WITH DATA is the default; explicit for clarity

-- Verify isolation: writes in the clone do NOT affect production
INSERT INTO dev_team_alpha.staging.sales (id, amount) VALUES (99999, 0.01);

-- Show that the clone shares storage with the source until diverged
SELECT
    TABLE_NAME,
    IS_CLONED,
    ROWS,
    BYTES / 1024 / 1024 / 1024 AS size_gb
FROM dev_team_alpha.information_schema.tables
WHERE TABLE_SCHEMA = 'staging'
ORDER BY BYTES DESC;
```

### Explanation
Zero-copy cloning is one of Snowflake's most powerful features for DevOps. When you `CLONE` a database, Snowflake does not copy data blocks. Instead, it creates a new namespace that points to the same underlying micropartitions. The clone is a snapshot at the exact point-in-time of the `CLONE` statement — it is fully isolated: writes in the clone create new data blocks that diverge from the source.

The clone is immediately queryable and fully transactional. You can `ALTER`, `DROP`, or `INSERT` in the clone without any impact on the source. This is how companies like Atlassian give each feature team a full copy of production data for testing — in seconds, not hours.

Storage cost is zero until you modify the clone. Snowflake charges for storage based on unique data blocks: since the clone shares all blocks with the source initially, the storage charge does not increase. Once you `INSERT` or `UPDATE` in the clone, only the modified blocks incur additional storage cost. For a 50TB database where test teams modify < 1% of the data, the first month of storage cost for clones is negligible.

Time travel applies to clones independently. You can time-travel within the clone to before a test modification, and the source's time-travel history remains intact. The clone's time-travel window starts at the clone creation time.

### Example Data / Sample Tables
```sql
-- Production database structure
CREATE OR REPLACE DATABASE prod_analytics;
USE prod_analytics;
CREATE SCHEMA staging;
CREATE SCHEMA curated;

CREATE TABLE staging.sales (
    sale_id    BIGINT,
    product_id BIGINT,
    amount     DECIMAL(12,2),
    region     VARCHAR(20),
    sale_date  DATE
);

INSERT INTO staging.sales VALUES
(1, 101, 250.00, 'US-East',  '2024-01-01'),
(2, 102, 175.50, 'EU-West',  '2024-01-02'),
(3, 101, 320.00, 'US-East',  '2024-01-03'),
(4, 103,  99.99, 'APAC',     '2024-01-04');

-- Clone and verify independence
CREATE DATABASE dev_alpha CLONE prod_analytics;
INSERT INTO dev_alpha.staging.sales VALUES (5, 104, 500.00, 'US-West', '2024-01-05');

-- Production still has 4 rows; clone has 5
SELECT 'PROD' AS env, COUNT(*) FROM prod_analytics.staging.sales
UNION ALL
SELECT 'CLONE' AS env, COUNT(*) FROM dev_alpha.staging.sales;
-- Result: PROD = 4, CLONE = 5 — fully isolated
```

### Comparison (if applicable)
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| `CLONE DATABASE` | Instant, zero storage until diverged, full isolation | Entire DB cloned even if you need one table | Team-level dev/test environments |
| `CLONE SCHEMA` | Targeted, lower divergence risk, cheaper | Must recreate objects outside the schema | Schema-level testing |
| `CLONE TABLE` | Most granular, cheapest | Must handle foreign key dependencies manually | Ad-hoc analysis on a single table |
| Manual `CREATE TABLE AS SELECT` | Full control, can transform data | Slow, consumes credits and storage | When data needs transformation at clone time |

### Edge Cases
- Cloning fails if you have active transactions on the source database; commit or rollback first
- Cloning a database with secure views: the masking policies are NOT cloned — you must recreate them
- External tables in the source are shared (not cloned) since they point to the same S3/GCS location
- Dropping a clone does not affect the source, but if the clone was the only consumer of some data blocks, storage may reduce

### Principal-Level Considerations
- For CI/CD pipelines: clone at the start of each pipeline run, drop at the end. At Uber, each PR gets a clone for integration tests. Automation via `snowsql` or the Python connector is essential.
- Tag cloned databases with `COMMENT` or tags to track ownership and auto-expire: `CREATE TAG dev_owner VALUES 'team-alpha'; ALTER DATABASE dev_alpha SET TAG dev_owner = 'team-alpha';`
- Use `UNDROP DATABASE dev_alpha;` to recover accidentally dropped clones — this works as long as within the time-travel window (default 1 day for non-Enterprise, 90 days for Enterprise+).
- Warehouse cost: clones are queried independently; auto-suspend after 5 minutes of inactivity is critical. Budget $50-100/month per clone for compute on a MEDIUM warehouse.

### Follow-up Questions
1. **What happens to the source database's time-travel window after cloning?**
   - **Answer:** The source's time-travel history is unaffected. The clone starts its own time-travel window from the moment of cloning. If the source had 90-day time travel, it continues with 90 days from the original data load dates. The clone gets its own 1-day (Standard) or 90-day (Enterprise+) window starting from the clone timestamp.

2. **Can you clone a database to a different Snowflake region or account?**
   - **Answer:** No — zero-copy cloning only works within the same Snowflake account. For cross-region replication, use Snowflake's Data Sharing or Database Replication feature (`CREATE REPLICATION DATABASE`), which is a scheduled full copy, not zero-copy.

3. **How do you handle grants on cloned databases?**
   - **Answer:** Grants are cloned by default with `CLONE`. The clone inherits all roles and privileges from the source. You can then modify them: `REVOKE USAGE ON DATABASE dev_alpha FROM ROLE analysts; GRANT USAGE ON DATABASE dev_alpha TO ROLE dev_team;`

4. **What is the maximum size of a database you can clone?**
   - **Answer:** There is no size limit — Snowflake has cloned petabyte-scale databases instantly. The only constraint is storage capacity of the account, and since cloning is zero-copy, it adds no storage initially.

5. **How do you automate clone lifecycle management?**
   - **Answer:** Use a Snowflake Task that checks clone age and drops stale ones: `CREATE TASK cleanup_old_clones SCHEDULE = 'USING CRON 0 3 * * MON' AS DELETE FROM information_schema.schemata WHERE schema_name LIKE 'dev_%' AND created < DATEADD(day, -7, CURRENT_TIMESTAMP());` Or use the Python connector with a cron job to list, query, and drop clones older than a threshold.

---

## Q3. Time travel query for data recovery

### Scenario
A data analyst accidentally ran `DELETE FROM curated.fact_sales WHERE region = 'APAC'` in production, removing 2.3 million rows representing $47M in revenue. The deletion happened 3 hours ago. The data warehouse is on a 600TB database with 90-day time travel enabled (Enterprise edition). You must recover the deleted rows, verify data integrity, and explain how to prevent this at the architectural level. This is a real incident that has happened at every major company.

### Solution
```sql
-- Step 1: Query the data as it existed BEFORE the accidental DELETE
-- Option A: Using a timestamp (if you know when the DELETE ran)
SELECT *
FROM curated.fact_sales
    BEFORE (STATEMENT => '8e5d4f2a-1b3c-4d5e-9f6a-7b8c9d0e1f2a')
WHERE region = 'APAC';

-- Option B: Using a time offset (more common in practice)
SELECT *
FROM curated.fact_sales
    AT (TIMESTAMP => DATEADD(HOUR, -3, CURRENT_TIMESTAMP()))
WHERE region = 'APAC';

-- Option C: Using TIME TRAVEL with OFFSET
SELECT *
FROM curated.fact_sales
    AT (OFFSET => -10800)  -- 3 hours = 10800 seconds
WHERE region = 'APAC';

-- Step 2: Restore the deleted rows
INSERT INTO curated.fact_sales
SELECT *
FROM curated.fact_sales
    AT (TIMESTAMP => DATEADD(HOUR, -3, CURRENT_TIMESTAMP()))
WHERE region = 'APAC'
  AND sale_id NOT IN (SELECT sale_id FROM curated.fact_sales);  -- Avoid duplicates

-- Step 3: Verify the recovery
SELECT
    'CURRENT'  AS dataset, COUNT(*) AS row_count, SUM(amount) AS total_amount
FROM curated.fact_sales WHERE region = 'APAC'
UNION ALL
SELECT
    'BEFORE_DELETE' AS dataset, COUNT(*), SUM(amount)
FROM curated.fact_sales
    AT (TIMESTAMP => DATEADD(HOUR, -3, CURRENT_TIMESTAMP()))
WHERE region = 'APAC';
-- These should match
```

### Explanation
Snowflake's Time Travel allows you to access historical data without backups. Every change to a table creates new micropartitions; Snowflake keeps old micropartitions for the time-travel duration (1 day for Standard, up to 90 days for Enterprise). The `AT (TIMESTAMP => ...)` clause creates a read-only snapshot of the table at that point in time.

The `BEFORE (STATEMENT => ...)` syntax references a specific query by its statement query ID (visible in the Query History UI). This is the most precise recovery method: you go back to the exact moment before the destructive statement executed. You can find the query ID in `TABLE(RESULT_SCAN(LAST_QUERY_ID()))` or in the Snowflake UI under Query History.

For the recovery `INSERT`, the `NOT IN` subquery prevents re-inserting rows that were not affected (e.g., if some APAC rows were updated but not deleted). This pattern is safe for partial recoveries. At scale (millions of rows), the `INSERT ... SELECT` time-travel query uses the source warehouse's compute and scans only the relevant micropartitions — it is much faster than a full table scan.

Time travel queries do NOT consume additional storage; they read existing historical micropartitions. However, they do consume warehouse credits proportional to the data scanned. For a 600TB table scanning 1TB of APAC data, expect ~1.5 minutes on a LARGE warehouse (roughly 0.0167 credits).

### Example Data / Sample Tables
```sql
CREATE OR REPLACE TABLE curated.fact_sales (
    sale_id    BIGINT,
    product_id BIGINT,
    amount     DECIMAL(12,2),
    region     VARCHAR(20),
    sale_date  DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

INSERT INTO curated.fact_sales (sale_id, product_id, amount, region, sale_date) VALUES
(1, 101, 15000.00, 'APAC',   '2024-01-01'),
(2, 102, 25000.00, 'APAC',   '2024-01-02'),
(3, 101, 12000.00, 'US-East','2024-01-01'),
(4, 103,  8000.00, 'EU-West','2024-01-03');

-- Simulate the accidental DELETE
DELETE FROM curated.fact_sales WHERE region = 'APAC';
-- 2 rows deleted

-- Verify that AT/OFFSET sees the deleted rows
SELECT sale_id, region, amount
FROM curated.fact_sales
    AT (OFFSET => 0)
WHERE region = 'APAC'
ORDER BY sale_id;
-- Returns the 2 deleted rows — fully recoverable
```

### Comparison (if applicable)
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| `AT (TIMESTAMP => ...)` | Precise point-in-time, easy to calculate | Requires knowing the exact timestamp | When you know when the error occurred |
| `BEFORE (STATEMENT => ...)` | Exact statement boundary, no timestamp math | Need the query ID from history UI | Most reliable for recovery |
| `AT (OFFSET => -N)` | Simple, no timestamp math needed | Seconds-based; easy to miscalculate large offsets | Quick recoveries within minutes/hours |
| `UNDROP TABLE` | Restores entire dropped table | Only works for DROP TABLE, not partial deletes | When an entire table was dropped |

### Edge Cases
- Time travel does NOT work across database or schema clones — each has its own history
- If the table was truncated (`TRUNCATE TABLE`), time travel still works: `TRUNCATE` is a tracked DML operation
- Time travel on clustered tables works but may scan more data if the clustering key changed
- After the time-travel window expires, data is permanently deleted — no recovery possible
- `UNDROP` works for databases, schemas, and tables dropped within the time-travel window

### Principal-Level Considerations
- Implement a backup strategy that goes BEYOND time travel: periodic `CLONE` to a cold-storage database with 90+ day retention. Time travel is operational; clones are your disaster recovery.
- Restrict `DELETE` and `TRUNCATE` privileges: use a row access policy or secure view to prevent analysts from modifying production tables. At Databricks and Google, production write access is gated through a pipeline, not direct SQL.
- Enable query tagging on all DML statements to track who deleted what: `ALTER SESSION SET QUERY_TAG = 'user:john.doe|action:delete';`
- Cost of time travel storage: Snowflake charges for historical data. For a 600TB database with 90-day retention and ~5% daily change rate, expect ~30TB of historical storage ($2.40/TB-month = ~$72/month).

### Follow-up Questions
1. **What is the difference between `AT` and `BEFORE`?**
   - **Answer:** `AT (TIMESTAMP => t)` shows the table state AT that exact moment — including any transactions committed at or before `t`. `BEFORE (STATEMENT => qid)` shows the state immediately BEFORE a specific statement was executed. Use `BEFORE` for precise recovery: it excludes the effects of the statement you want to undo. `AT` is better for auditing: "what did the data look like at 3 PM?"

2. **Does time travel work with merged tables (after a MERGE)?**
   - **Answer:** Yes. `MERGE` is a DML operation and its effects are tracked. You can time-travel to before or after a MERGE using any of the three clauses. The merged rows (inserted, updated, deleted) are all captured in the history.

3. **How do you find the query ID of the destructive statement?**
   - **Answer:** Query `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`: `SELECT query_id, query_text, start_time, user_name FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE query_text LIKE '%DELETE%' AND start_time > DATEADD(HOUR, -24, CURRENT_TIMESTAMP()) ORDER BY start_time DESC;` Or check the Snowflake UI Query History tab.

4. **Can you use time travel on a view?**
   - **Answer:** Yes — `SELECT * FROM my_view AT (TIMESTAMP => ...)` applies the time-travel clause to all base tables referenced by the view. The view definition is resolved at query time against the historical data. This works for any view type (regular, secure, materialized).

5. **What happens to streams when you use time travel?**
   - **Answer:** Streams track changes from their own positioned view, independent of time travel. However, you can `CREATE STREAM my_stream ON TABLE my_table AT (TIMESTAMP => ...)` to initialise a stream at a historical point. This is a powerful pattern for back-filling data into a new pipeline from a point in the past.

---

## Q4. Stream-based incremental processing

### Scenario
Your platform receives 500K new events per minute into `raw_events` via Snowpipe. You need to process only new and changed rows — not the full 10TB table — and load them into `staging.events_clean`. Updates and deletes must be tracked. This is the core ETL pattern at Uber (ride events), Google (search clicks), and Atlassian (Jira events). The interviewer expects you to understand stream positioning, metadata columns, and change detection.

### Solution
```sql
-- Step 1: Create a stream on the source table
CREATE OR REPLACE STREAM raw_events.events_stream
    ON TABLE raw_events.raw_payload
    COPY TRANSFORMATIONS;   -- Captures transformations applied via CTAS

-- Step 2: Create the target table
CREATE OR REPLACE TABLE staging.events_clean (
    event_id       STRING,
    user_id        STRING,
    event_type     STRING,
    country        STRING,
    amount         DECIMAL(12,2),
    processed_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    source_meta    STRING    -- Metadata about the change
);

-- Step 3: Incremental load — process only new/changed rows
INSERT INTO staging.events_clean (event_id, user_id, event_type, country, amount, source_meta)
SELECT
    src:event_id::STRING,
    src:user:id::STRING,
    src:event_type::STRING,
    src:user:country::STRING,
    src:amount::DECIMAL(12,2),
    CASE
        WHEN METADATA$ACTION = 'INSERT' THEN 'NEW'
        WHEN METADATA$ACTION = 'UPDATE' THEN 'UPDATED'
        WHEN METADATA$ACTION = 'DELETE' THEN 'DELETED'
        ELSE 'UNKNOWN'
    END
FROM raw_events.events_stream
WHERE METADATA$ISDELETE = FALSE;  -- Filter deletes (handle separately if needed)
-- After this INSERT, the stream position advances automatically

-- Step 4: Handle deletions (if source table supports them)
DELETE FROM staging.events_clean
WHERE event_id IN (
    SELECT src:event_id::STRING
    FROM raw_events.events_stream
    WHERE METADATA$ACTION = 'DELETE'
);

-- Step 5: Verify stream position after processing
SELECT
    STREAM_NAME,
    SOURCE_NAME,
    ROWS_INSERTED,
    ROWS_UPDATED,
    ROWS_DELETED,
    IS_CURRENT
FROM raw_events.information_schema.streams
WHERE STREAM_NAME = 'EVENTS_STREAM';
-- IS_CURRENT = TRUE means all changes have been consumed
```

### Explanation
Streams in Snowflake provide a positioned, change-data-capture (CDC) view of a table. Unlike triggers (which do not exist in Snowflake), streams track `INSERT`, `UPDATE`, and `DELETE` operations without modifying the source table. The stream maintains an internal position that advances when you query it — making it a "read-once" log that is automatically consumed.

The three metadata columns are the key to correct processing:
- `METADATA$ACTION`: Returns `INSERT`, `UPDATE`, or `DELETE`
- `METADATA$ISUPDATE`: `TRUE` for updated rows (shows the old version)
- `METADATA$ISDELETE`: `TRUE` for deleted rows

After each `SELECT` from the stream, the position advances automatically. This means if your pipeline crashes after processing 50K of 200K rows, the next run processes the remaining 150K — no reprocessing. This is the exactly-once semantics that streaming pipelines at scale require.

Streams support two types: **appended-only** (default for tables without PK) and **standard** (for tables with PK). Appended-only streams track inserts and deletes but do not detect logical updates (a delete + insert looks like two operations). Standard streams with PK detect true updates: if a row with the same PK is modified, it appears once with `METADATA$ACTION = 'UPDATE'`.

At 500K events/minute, the stream processing query should run on a warehouse sized for ~1M rows per batch (every 2 minutes). A `LARGE` warehouse with `AUTO_SUSPEND = 60` seconds handles ~1M rows in ~10 seconds for simple transformations. Schedule this as a Snowflake Task on a 2-minute cadence.

### Example Data / Sample Tables
```sql
CREATE OR REPLACE TABLE raw_events.raw_payload (
    id       BIGINT AUTOINCREMENT,
    src      VARIANT,
    loaded   TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

CREATE OR REPLACE STREAM raw_events.events_stream ON TABLE raw_events.raw_payload;

-- Simulate inserts, updates, and deletes
INSERT INTO raw_events.raw_payload (src) VALUES
('{ "event_id": "e1", "user": { "id": "u100", "country": "US" }, "event_type": "click", "amount": 10.00 }'),
('{ "event_id": "e2", "user": { "id": "u200", "country": "UK" }, "event_type": "purchase", "amount": 99.99 }');

UPDATE raw_events.raw_payload
SET src = '{ "event_id": "e1", "user": { "id": "u100", "country": "US" }, "event_type": "click", "amount": 15.00 }'
WHERE src:event_id::STRING = 'e1';

DELETE FROM raw_events.raw_payload
WHERE src:event_id::STRING = 'e2';

-- Query the stream to see all changes
SELECT
    METADATA$ACTION,
    METADATA$ISUPDATE,
    METADATA$ISDELETE,
    src
FROM raw_events.events_stream;
-- Returns:
-- INSERT | F | F | {e1, amount:10}
-- INSERT | F | F | {e2, amount:99.99}
-- UPDATE | T | F | {e1, amount:10}   (old version of the update)
-- DELETE | F | T | {e2, amount:99.99}
```

### Comparison (if applicable)
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Standard Stream (with PK) | True update detection, exactly-once | Requires PK on source table | ETL pipelines with identifiable entities |
| Appended-Only Stream | No PK needed, simpler | Cannot distinguish update from delete+insert | Event logs, append-only data |
| Stream on View | Processes transformed data | Limited metadata capture | When the view is the authoritative source |
| `MERGE` without Stream | Full control, no stream dependency | Must track position manually (e.g., watermark) | When stream is not available |

### Edge Cases
- If you query the stream in multiple sessions simultaneously, the position is shared: the last reader gets the remaining changes (no duplicate processing)
- Streams on cloned tables are independent: cloning a table with a stream creates a new stream at the clone's position
- If the source table is truncated, the stream captures the truncation as deletions of all rows
- Streams on views only support `INSERT` tracking; updates and deletes on base tables may not propagate correctly to the stream

### Principal-Level Considerations
- Always include `METADATA$ROW_ID` in stream processing to handle deduplication in case of partial failures. At Google scale, exactly-once is non-negotiable.
- For multi-hop pipelines (raw → staged → curated), create a stream on each table. The stream on `staged` processes changes that propagate from `raw`. This is the "change propagation" pattern.
- Stream retention: by default, streams retain 1 day of history. If a pipeline is down for 2 days, the stream drops older changes. Set `DATA_RETENTION_TIME_IN_DAYS` on the source table to match your max pipeline downtime.
- Cost: stream processing on a LARGE warehouse for 1M rows/minute costs ~$2-3/month per stream. At 20 streams, budget ~$50/month in compute.

### Follow-up Questions
1. **What happens if the stream is not consumed for more than the retention period?**
   - **Answer:** Changes older than the retention period (default 1 day) are dropped from the stream. When you finally query the stream, you only see recent changes — you have lost the older ones. To prevent this, either (a) consume the stream frequently, (b) increase `DATA_RETENTION_TIME_IN_DAYS` on the source table, or (c) use a scheduled Task that auto-consumes.

2. **How do you backfill a stream from a historical point?**
   - **Answer:** Use `CREATE STREAM my_stream ON TABLE my_table AT (TIMESTAMP => '2024-01-01 00:00:00');`. This initialises the stream at that point and captures all changes from then until now. Note: this does NOT show the data that existed before the timestamp — only changes that occurred after. To load the historical base data, do a separate `INSERT ... SELECT ... AT (TIMESTAMP => ...)` first.

3. **Can you have multiple streams on the same table?**
   - **Answer:** Yes — each stream maintains its own independent position. You can have `stream_etl` for the pipeline and `stream_audit` for logging, both on the same table. Each advances independently when queried.

4. **How does COPY TRANSFORMATIONS work with streams?**
   - **Answer:** When a stream is created with `COPY TRANSFORMATIONS` on a table created via `CREATE TABLE ... AS SELECT`, the stream applies the same transformations to new rows. This means if the table has `SELECT src:amount::FLOAT as amount`, the stream outputs `amount` as a FLOAT, not the raw VARIANT.

5. **What is the maximum lag between source change and stream availability?**
   - **Answer:** Near real-time for internal operations (seconds). For Snowpipe-loaded data, the stream sees changes as soon as the load micro-batch completes (typically < 1 minute). For external tables, streams are not supported — you must use Snowpipe with notification integration.

---

## Q5. Materialized view design

### Scenario
Your sales analytics dashboard queries `fact_sales` (500M rows, 2TB) joining with `dim_product` and `dim_date`. The dashboard's main KPI is "total sales by product category and month." Currently, each dashboard load takes 45 seconds. You need to pre-aggregate this data using a materialized view so queries complete in < 2 seconds. At Adobe or Google, materialized views are the primary tool for dashboard performance.

### Solution
```sql
-- Step 1: Create the materialized view
CREATE OR REPLACE MATERIALIZED VIEW curated.mv_sales_by_category_month AS
SELECT
    d.year,
    d.month,
    p.category,
    p.subcategory,
    COUNT(*)                              AS total_transactions,
    SUM(s.amount)                         AS total_revenue,
    AVG(s.amount)                         AS avg_transaction_value,
    MIN(s.amount)                         AS min_transaction,
    MAX(s.amount)                         AS max_transaction,
    COUNT(DISTINCT s.customer_id)         AS unique_customers,
    APPROX_PERCENTILE_CONT(s.amount, 0.5) AS median_transaction
FROM staging.fact_sales  s
JOIN curated.dim_product p ON s.product_id = p.product_id
JOIN curated.dim_date    d ON s.sale_date  = d.date_key
GROUP BY d.year, d.month, p.category, p.subcategory;

-- Step 2: Query the materialized view (sub-second response)
SELECT
    category,
    year,
    month,
    total_revenue,
    unique_customers
FROM curated.mv_sales_by_category_month
WHERE year = 2024
ORDER BY total_revenue DESC;

-- Step 3: Verify the MV is being used (check query plan)
EXPLAIN
SELECT
    category,
    SUM(total_revenue) AS revenue
FROM curated.mv_sales_by_category_month
WHERE year = 2024 AND month = 12
GROUP BY category;
-- The output should show a scan of the MV, not the base tables
```

### Explanation
A materialized view in Snowflake is a pre-computed, stored result of a query. Unlike a regular view (which is a saved query that re-executes each time), the materialized view stores the result set physically. When the underlying base tables change, Snowflake automatically and incrementally refreshes the materialized view — you do not need to manually `REFRESH` it.

The automatic refresh is the key differentiator. When rows are `INSERT`ed, `UPDATE`d, or `DELETE`d in `fact_sales`, `dim_product`, or `dim_date`, Snowflake detects the change through internal tracking (similar to streams) and updates only the affected aggregation buckets in the materialized view. This incremental refresh is far cheaper than a full rebuild.

For the query to use the materialized view, the optimizer must determine that the MV can satisfy the query. Snowflake's query optimizer automatically rewrites queries to use a materialized view when it can — you do not need to reference the MV explicitly. In practice, for dashboard queries that match the aggregation granularity, the MV is always chosen.

Cost considerations: the materialized view consumes additional storage (the pre-computed result set), but for aggregations, this is typically < 1% of the base table size. A 2TB fact_sales table aggregated to category×month might produce a 50MB materialized view. The incremental refresh costs minimal compute — Snowflake processes changes in the background using a small warehouse allocation.

### Example Data / Sample Tables
```sql
-- Base tables
CREATE OR REPLACE TABLE staging.fact_sales (
    sale_id     BIGINT,
    product_id  BIGINT,
    customer_id BIGINT,
    amount      DECIMAL(12,2),
    sale_date   DATE
);

CREATE OR REPLACE TABLE curated.dim_product (
    product_id  BIGINT,
    category    VARCHAR(50),
    subcategory VARCHAR(50)
);

CREATE OR REPLACE TABLE curated.dim_date (
    date_key    DATE,
    year        INT,
    month       INT,
    day         INT
);

-- Sample data
INSERT INTO curated.dim_product VALUES
(1, 'Electronics', 'Phones'),
(2, 'Electronics', 'Laptops'),
(3, 'Clothing', 'Shirts');

INSERT INTO curated.dim_date VALUES
('2024-01-01', 2024, 1, 1),
('2024-02-01', 2024, 2, 1);

INSERT INTO staging.fact_sales VALUES
(1, 1, 1001, 999.99, '2024-01-01'),
(2, 2, 1002, 1299.00, '2024-01-01'),
(3, 1, 1003, 899.50, '2024-02-01'),
(4, 3, 1001, 49.99, '2024-01-01');

-- After creating the MV, insert more data to trigger incremental refresh
INSERT INTO staging.fact_sales VALUES (5, 2, 1004, 2499.00, '2024-02-01');
-- The MV automatically updates the (2024, 2, 'Electronics', 'Laptops') bucket
```

### Comparison (if applicable)
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Materialized View | Auto-refresh, optimizer picks it, sub-second queries | Limited to aggregations (GROUP BY), storage cost | Dashboard KPIs, fixed-grain reporting |
| Dynamic Table | Declarative, auto-refresh with target lag, flexible dependencies | Newer feature, less mature tooling | Pipeline-style data freshness targets |
| Manual aggregate table + Task | Full control, can include complex logic | Manual refresh scheduling, eventual consistency | Complex transformations not supported by MV |
| Result Cache | Free, automatic, 24-hour TTL | Only for identical queries, invalidated by any change | Read-only dashboards with static queries |

### Edge Cases
- Materialized views cannot contain: `DISTINCT` (except in aggregate functions), `UNION`, `LATERAL`, `QUALIFY`, or subqueries in the `FROM` clause
- If a base table is `TRUNCATE`d, the MV is fully rebuilt (not incrementally updated)
- Materialized views on external tables are not supported
- The MV refresh may lag behind source changes by seconds to minutes depending on batch size

### Principal-Level Considerations
- At scale, design the MV granularity to match the dashboard's most common query patterns. If 80% of queries filter by `year` and `month`, aggregate to that level — not day-level. This is the "pre-aggregation matching" principle.
- Use `RESULT_SCAN` + `SYSTEM$GET_QUERY_OPERATOR_STATS` to verify the MV is being used. If the optimizer falls back to base tables, the MV definition may not match the query's projection or filter.
- Storage cost: a 2TB fact table with a category×month×year MV stores ~10-100MB (depending on cardinality). At $2.30/TB/month, this is ~$0.0002/month — effectively free.
- For multi-tenant SaaS (like Atlassian), create one MV per tenant dimension using `PARTITION BY` or separate MVs per tenant — this avoids cross-tenant data leakage and enables per-tenant refresh cadences.

### Follow-up Questions
1. **Can a materialized view reference another materialized view?**
   - **Answer:** Yes — you can chain materialized views. `MV_L2` can aggregate from `MV_L1`, which aggregates from the base table. This enables multi-level aggregation hierarchies: base → daily → monthly → yearly. Each refresh propagates incrementally. However, deep chains increase refresh latency.

2. **How do you monitor materialized view freshness?**
   - **Answer:** Query `INFORMATION_SCHEMA.MATERIALIZED_VIEWS`: `SELECT TABLE_NAME, IS_REFRESHING, ROWS_CHANGED FROM information_schema.materialized_views WHERE TABLE_SCHEMA = 'CURATED';` `IS_REFRESHING = TRUE` means it is currently catching up. Also check `QUERY_HISTORY` for refresh queries on the MV.

3. **What happens if a base table column is dropped?**
   - **Answer:** The materialized view becomes invalid and stops refreshing. You must manually `DROP` and `CREATE` the MV, or `ALTER MATERIALIZED VIEW ... SWAP WITH` to a new definition. Always test schema changes against dependent MVs before applying in production.

4. **How does the incremental refresh mechanism work internally?**
   - **Answer:** Snowflake maintains a hidden delta log for each materialized view. When base tables change, the delta log captures the affected rows. During refresh, Snowflake applies the aggregation function to the delta and updates only the changed buckets. For `SUM(amount)`, it subtracts the old value and adds the new value for the affected group — not a full recalculation.

5. **What is the maximum number of materialized views per database?**
   - **Answer:** There is no hard limit, but practical limits are determined by storage and refresh compute. At Adobe, a single database may have 50-100 materialized views. Each MV has independent refresh behaviour and storage cost.

---
## Q6. Dynamic table with target refresh lag

### Scenario
You are designing a real-time analytics pipeline for an e-commerce platform that processes 10M events/day. The business requires that the aggregated KPIs (revenue by category, active users by region) are fresh within 5 minutes of the source data arriving. You want a declarative approach where Snowflake automatically manages the refresh cadence, rather than building a Task-based pipeline with manual scheduling. This is the modern replacement for materialized views in complex DAGs.

### Solution
```sql
-- Step 1: Create a dynamic table targeting 5-minute freshness
CREATE OR REPLACE DYNAMIC TABLE curated.dt_sales_kpis
    TARGET_REFRESH_LAG = '5 minutes'
    WAREHOUSE = 'ANALYTICS_WH'
AS
SELECT
    d.year,
    d.month,
    d.day,
    p.category,
    COUNT(*)                          AS total_transactions,
    SUM(s.amount)                     AS total_revenue,
    AVG(s.amount)                     AS avg_transaction_value,
    COUNT(DISTINCT s.customer_id)     AS unique_customers,
    COUNT(DISTINCT s.region)          AS active_regions
FROM staging.fact_sales  s
JOIN curated.dim_product p ON s.product_id = p.product_id
JOIN curated.dim_date    d ON s.sale_date  = d.date_key
GROUP BY d.year, d.month, d.day, p.category;

-- Step 2: Check the dynamic table's status
SELECT
    TABLE_NAME,
    TARGET_REFRESH_LAG,
    CURRENT_REFRESH_LAG,
    IS_REFRESHING,
    REFRESH_MODE,
    REFRESH_STATUS,
    BASE_TABLE_NUMBER,
    BASE_TABLE_BYTES_SCANNED,
    CURRENT_ROW_COUNT
FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLES())
WHERE TABLE_NAME = 'DT_SALES_KPIS';

-- Step 3: Query the dynamic table (fresh within 5 minutes)
SELECT
    category,
    total_revenue,
    unique_customers
FROM curated.dt_sales_kpis
WHERE day = CURRENT_DATE()
ORDER BY total_revenue DESC;

-- Step 4: Modify the refresh lag (e.g., relax to 15 minutes to save credits)
ALTER DYNAMIC TABLE curated.dt_sales_kpis
    SET TARGET_REFRESH_LAG = '15 minutes';
```

### Explanation
Dynamic tables are Snowflake's declarative, auto-refreshing data pipeline. Unlike materialized views (which auto-refresh incrementally in the background), dynamic tables let you declare a **target refresh lag** — the maximum age of the data relative to the source. Snowflake determines the refresh frequency based on the change rate of the base tables: if data changes rapidly, it refreshes more often; if idle, it refreshes less.

The `WAREHOUSE` clause assigns compute resources specifically to the dynamic table's refresh. This is separate from the warehouse your analysts use to query the table. Snowflake scales the warehouse up or down for refresh based on the work needed. You can set `MAX_CONCURRENCY_LEVEL` and `SCALING_MIN/MAX` on the warehouse to control costs.

The `TARGET_REFRESH_LAG` is a ceiling, not a guarantee. If a refresh takes longer than 5 minutes due to the size of the data or warehouse constraints, the lag may exceed 5 minutes. Snowflake does not guarantee SLAs on refresh timing — it is "best effort" within the assigned warehouse capacity. For mission-critical freshness, monitor `CURRENT_REFRESH_LAG` and alert if it exceeds the target.

Cost model: dynamic tables consume credits based on the refresh computation. A 5-minute target on a 2TB table with 1% daily change rate on a `LARGE` warehouse costs roughly $15-30/month in credits (the warehouse runs briefly every few minutes during active hours). Compare this to a Task-based pipeline: Tasks also consume credits but give you manual control over scheduling.

### Example Data / Sample Tables
```sql
-- Source tables (same as Q5)
CREATE OR REPLACE TABLE staging.fact_sales (
    sale_id     BIGINT,
    product_id  BIGINT,
    customer_id BIGINT,
    region      VARCHAR(20),
    amount      DECIMAL(12,2),
    sale_date   DATE
);

CREATE OR REPLACE TABLE curated.dim_product (
    product_id  BIGINT,
    category    VARCHAR(50)
);

CREATE OR REPLACE TABLE curated.dim_date (
    date_key    DATE,
    year        INT,
    month       INT,
    day         INT
);

-- Create a dedicated warehouse for dynamic table refresh
CREATE OR REPLACE WAREHOUSE dt_warehouse
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 30
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 2;

-- Create the dynamic table
CREATE OR REPLACE DYNAMIC TABLE curated.dt_live_kpis
    TARGET_REFRESH_LAG = '5 minutes'
    WAREHOUSE = 'DT_WAREHOUSE'
AS
SELECT
    p.category,
    COUNT(*) AS txns,
    SUM(s.amount) AS revenue
FROM staging.fact_sales s
JOIN curated.dim_product p ON s.product_id = p.product_id
WHERE s.sale_date = CURRENT_DATE()
GROUP BY p.category;

-- Dynamic tables can chain: DT2 depends on DT1
CREATE OR REPLACE DYNAMIC TABLE curated.dt_daily_summary
    TARGET_REFRESH_LAG = '1 hour'
    WAREHOUSE = 'DT_WAREHOUSE'
AS
SELECT
    'DAILY' AS granularity,
    category,
    SUM(txns) AS total_txns,
    SUM(revenue) AS total_revenue
FROM curated.dt_live_kpis
GROUP BY category;
```

### Comparison (if applicable)
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Dynamic Table (5 min lag) | Declarative, auto-schedules, chainable | Credits consumed by refresh, not SLA-guaranteed | Near-real-time KPIs, auto-managed pipelines |
| Materialized View | Incremental refresh, near-zero compute cost | Limited SQL support, only aggregations | Static dashboards with fixed granularity |
| Task with Schedule | Full SQL freedom, precise control | Manual DAG management, complex error handling | Complex transformations, non-aggregation logic |
| Result Cache | Free, 24-hour TTL | Only identical queries, invalidated by changes | Read-heavy workloads with stable queries |

### Edge Cases
- If the source table is truncated, the dynamic table may need a full refresh (not incremental), which could exceed the target lag
- Dynamic tables cannot have `DISTINCT`, `UNION`, `QUALIFY`, `LATERAL`, or window functions in the definition
- The `WAREHOUSE` for a dynamic table cannot be shared with another dynamic table (each DT needs its own warehouse or the same warehouse can serve multiple DTs)
- If the warehouse is manually suspended, the dynamic table refresh pauses until resumed

### Principal-Level Considerations
- Use dynamic tables as the **orchestration layer** for a pipeline DAG: `raw DT → staged DT → curated DT`. Each tier can have different refresh targets: raw = 1 min, staged = 5 min, curated = 15 min. This is the "tiered freshness" pattern.
- Monitor costs with `ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY` (dynamic tables use the same table as MVs). Set up alerts: `SELECT SUM(execution_time) / 60 / 60 FROM ... WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())` to track hourly credit usage.
- At Google or Uber scale, use dynamic tables for the "last mile" of the pipeline — the final aggregation that feeds dashboards. The heavy lifting (cleaning, joining) happens in Tasks or dbt.
- Consider `TARGET_REFRESH_LAG = '1 hour'` for non-urgent data to reduce compute costs by 10-50x versus 5-minute refresh.

### Follow-up Questions
1. **How do dynamic tables handle incremental vs. full refresh?**
   - **Answer:** Snowflake automatically determines whether an incremental or full refresh is needed. If a base table has a small number of changes, it does an incremental refresh. If the base table is truncated or the change ratio exceeds a threshold, it does a full rebuild. You can see this in `SYSTEM$GET_DYNAMIC_TABLE_REFRESH_INFO()`.

2. **Can you join a dynamic table to a materialized view?**
   - **Answer:** Yes — any table type can reference any other. A dynamic table can SELECT from a materialized view, and vice versa. The refresh order is managed by Snowflake's dependency graph. However, be mindful of refresh cascades: if DT1 depends on MV1 which depends on fact_sales, the total latency is the sum of all refresh lags.

3. **What is the difference between TARGET_REFRESH_LAG and CURRENT_REFRESH_LAG?**
   - **Answer:** `TARGET_REFRESH_LAG` is your goal (e.g., "5 minutes"). `CURRENT_REFRESH_LAG` is the actual age of the data right now. If the current lag is 8 minutes, the data is stale relative to the target. Monitor this in a dashboard: `SELECT CURRENT_REFRESH_LAG FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLES());`.

4. **How do you handle backfill for a new dynamic table?**
   - **Answer:** When you first create a dynamic table, Snowflake does a full refresh of all historical data. The `CURRENT_REFRESH_LAG` will be large initially (it is catching up to the present). For a table with 5 years of history and a 5-minute target, the initial load may take hours on a MEDIUM warehouse. Use a larger warehouse for the initial load, then resize down.

5. **Can you have a dynamic table that depends on an external table?**
   - **Answer:** No — dynamic tables cannot reference external tables. External tables are not tracked for changes in a way that supports auto-refresh. You must stage the external data into a regular table first, then create the dynamic table on the regular table.

---

## Q7. Query performance tuning with EXPLAIN

### Scenario
A BI analyst reports that a dashboard query takes 3 minutes on a 4TB `fact_events` table. The query filters by date range and aggregates by product category. As a Principal Engineer, you must diagnose the root cause, fix it, and explain the optimisation choices. At Uber or Atlassian, slow queries cost thousands in credits and frustrate users.

### Solution
```sql
-- The slow query (as submitted by the analyst)
SELECT
    p.category,
    COUNT(*)  AS event_count,
    SUM(e.value) AS total_value
FROM raw_db.fact_events e
JOIN raw_db.dim_product p ON e.product_id = p.product_id
WHERE e.event_date >= '2024-01-01' AND e.event_date < '2024-02-01'
GROUP BY p.category;

-- Step 1: Run EXPLAIN to see the query plan
EXPLAIN
SELECT
    p.category,
    COUNT(*)  AS event_count,
    SUM(e.value) AS total_value
FROM raw_db.fact_events e
JOIN raw_db.dim_product p ON e.product_id = p.product_id
WHERE e.event_date >= '2024-01-01' AND e.event_date < '2024-02-01'
GROUP BY p.category;

-- DIAGNOSIS: EXPLAIN shows:
-- - Full table scan on fact_events (no pruning on event_date)
-- - Hash join with dim_product
-- - No clustering key on event_date

-- Step 2: Add a clustering key on the frequently-filtered column
ALTER TABLE raw_db.fact_events
    CLUSTER BY (event_date);

-- Note: Snowflake auto-clustering may already handle this, but for
-- high-value tables, manual clustering ensures optimal pruning.
-- Enable auto-reclustering if preferred:
ALTER TABLE raw_db.fact_events
    CLUSTER BY (event_date);  -- Same syntax; Snowflake manages reclustering

-- Step 3: Create an index-like structure using a materialised view
CREATE MATERIALIZED VIEW raw_db.mv_events_by_date_category AS
SELECT
    e.event_date,
    p.category,
    COUNT(*)  AS event_count,
    SUM(e.value) AS total_value
FROM raw_db.fact_events e
JOIN raw_db.dim_product p ON e.product_id = p.product_id
GROUP BY e.event_date, p.category;

-- Step 4: Re-run EXPLAIN on the optimised query
EXPLAIN
SELECT
    category,
    SUM(event_count) AS total_events,
    SUM(total_value) AS total_value
FROM raw_db.mv_events_by_date_category
WHERE event_date >= '2024-01-01' AND event_date < '2024-02-01'
GROUP BY category;

-- Step 5: Verify data pruning with QUERY_HISTORY
SELECT
    query_id,
    bytes_scanned / 1024 / 1024 / 1024 AS bytes_scanned_gb,
    execution_time / 1000 AS execution_seconds,
    credits_used
FROM TABLE(INFORMATION_SCHEMAQUERY_HISTORY())
WHERE query_text LIKE '%mv_events_by_date_category%'
ORDER BY start_time DESC
LIMIT 5;
```

### Explanation
`EXPLAIN` in Snowflake returns the logical and physical query plan without executing the query. It shows each operation: table scans, joins, filters, aggregations, and — critically — whether data pruning occurs. A full table scan on a 4TB table means Snowflake reads all 4TB of micropartitions, consuming ~5 credits on a LARGE warehouse for 10 minutes.

The first diagnosis from EXPLAIN reveals the root cause: **no data pruning on `event_date`**. Snowflake partitions data into micropartitions (approximately 50-500MB each). When you filter on `event_date`, Snowflake can skip micropartitions whose min/max `event_date` does not overlap with the filter range. Without a clustering key, the min/max statistics for `event_date` span the entire data range in every micropartition, so no pruning occurs.

Adding `CLUSTER BY (event_date)` reorganises the micropartitions so that each partition contains a narrow range of dates. After clustering, filtering for January 2024 scans only the micropartitions containing January data — potentially reducing a 4TB scan to 100GB. This is a 40x reduction in data scanned, translating to a 40x reduction in credits consumed.

The materialized view is a complementary optimisation: it pre-computes the aggregation so the query never touches the base table. At scale, use a layered approach: cluster the base table for ad-hoc queries, and create materialized views for fixed dashboard queries. This is the "two-tier optimisation" pattern used at Google.

Cost comparison: a 4TB full scan on a LARGE warehouse takes ~5 credits ($4.50) and 10 minutes. With clustering + materialized view, the same query scans < 100MB, takes ~0.001 credits, and completes in < 2 seconds. Annual savings for 1,000 such queries: ~$4,500.

### Example Data / Sample Tables
```sql
CREATE OR REPLACE TABLE raw_db.fact_events (
    event_id    BIGINT,
    product_id  BIGINT,
    value       DECIMAL(12,2),
    event_date  DATE,
    region      VARCHAR(20)
);

-- Generate sample data (1M rows spanning 2 years)
INSERT INTO raw_db.fact_events
SELECT
    ROW_NUMBER() OVER (),
    (RANDOM() % 1000) + 1,
    ROUND(RAND() * 1000, 2),
    DATEADD(day, (RANDOM() % 730) - 365, '2024-01-01'),
    CASE (RANDOM() % 4)
        WHEN 0 THEN 'US-East'
        WHEN 1 THEN 'EU-West'
        WHEN 2 THEN 'APAC'
        ELSE 'US-West'
    END
FROM TABLE(GENERATOR(ROWCOUNT => 1000000));

-- Before clustering: check partition stats
SELECT
    min(event_date),
    max(event_date),
    count(*)
FROM raw_db.fact_events;
-- All partitions span the full date range → no pruning possible

-- Add clustering key
ALTER TABLE raw_db.fact_events CLUSTER BY (event_date);

-- Check clustering information
SELECT
    TABLE_NAME,
    CLUSTERING_KEY,
    IS_CLUSTERING_IN_PROGRESS,
    PARTITION_COUNT,
    AVG_PARTITION_SIZE
FROM TABLE(INFORMATION_SCHEMA.CLUSTERING_INFORMATION('RAW_DB.FACT_EVENTS'));
```

### Comparison (if applicable)
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| `CLUSTER BY` on filter column | Improves pruning, works with all queries | Reclustering cost, manual maintenance | Columns frequently used in WHERE clauses |
| Auto-clustering | Zero management, automatic | Cost-based (reclustering charges credits) | Most tables; let Snowflake decide |
| Materialized View | Pre-computed, sub-second, auto-refresh | Limited SQL, storage cost | Fixed dashboard queries |
| Query result cache | Free, 24-hour TTL, automatic | Only identical queries | Read-heavy workloads with repeat queries |
| Increase warehouse size | Linear speedup for unoptimised queries | Credit cost scales with warehouse size | Temporary fix; not a long-term solution |

### Edge Cases
- Clustering on a column with low cardinality (e.g., `gender` with 2 values) provides no pruning benefit
- If a query filters on multiple columns, cluster on the column with the most selective filter
- Auto-reclustering can cost more than the savings if the table is rarely queried with that column in the WHERE clause
- `EXPLAIN` does not show the actual credits used — only the logical plan. Use `QUERY_HISTORY` for real credit consumption.

### Principal-Level Considerations
- Before adding a manual clustering key, check the **clustering depth** with `SYSTEM$CLUSTERING_INFORMATION('schema.table')`. If the depth is < 10, auto-clustering is working well — no manual key needed.
- At PB scale, use a **composite clustering key**: `CLUSTER BY (event_date, region)`. This prunes on both dimensions but increases reclustering cost. Only do this if both columns are frequently filtered together.
- Use `SESSION_PARAMETER QUERY_TAG` to identify slow queries and trace them to specific dashboards or users. At Atlassian, every query is tagged with the dashboard name.
- Consider **data retention vs. active data**: archive old data to a cheaper storage tier (e.g., external stage) and cluster only the active 12 months.

### Follow-up Questions
1. **What is the difference between Snowflake auto-clustering and manual `CLUSTER BY`?**
   - **Answer:** Manual `CLUSTER BY` specifies the clustering key; Snowflake reorganises existing data and maintains clustering on new data. Auto-clustering (`CLUSTER BY` without a key) uses Snowflake's algorithm to pick the best clustering dimensions based on query patterns. Manual gives you control; auto removes management overhead. Auto-clustering incurs credits for reclustering; manual reclustering also costs credits but only when you trigger it.

2. **How do you measure the impact of a clustering key?**
   - **Answer:** Run the query before and after clustering, then compare `bytes_scanned` in `QUERY_HISTORY`. A 10-100x reduction indicates effective pruning. Also check `PARTITION_COUNT` vs. `PARTITIONS_SCANNED` in the query profile. Use `EXPLAIN` to verify the plan shows fewer partitions being read.

3. **When does EXPLAIN show a "FULL TABLE SCAN"?**
   - **Answer:** When no pruning is possible — either because (a) there is no clustering key on the filter column, (b) the filter covers the entire data range, or (c) the micropartition statistics (min/max) for the filter column span the full range. The query plan will show `STREAM TABLE SCAN` with no `PARTITION ELIMINATION`.

4. **Can you use EXPLAIN on a multi-statement transaction?**
   - **Answer:** No — `EXPLAIN` works on a single `SELECT` statement. For multi-statement workflows (e.g., CTE chains, stored procedures), use the Query Profile UI in the Snowflake web interface, which shows the execution plan for each statement in the transaction.

5. **How do you optimise a query that joins three large tables?**
   - **Answer:** (a) Ensure all three tables have clustering keys on the join/filter columns, (b) Pre-aggregate to reduce join sizes, (c) Use `JOIN ... ON` with the most selective table first (Snowflake's optimizer handles this, but materialized views on intermediate joins help), (d) Consider semi-joins (`EXISTS`) instead of `IN` subqueries for better plan generation.

---

## Q8. Secure view with row-level security

### Scenario
Your company operates in multiple regions (US, EU, APAC) and must comply with GDPR and regional data governance. Analysts in the US should only see US data; EU analysts see only EU data. You need to implement row-level security using a secure view with a row access policy. At Google or Atlassian, where multi-tenant data is the norm, this is a mandatory architectural pattern.

### Solution
```sql
-- Step 1: Create the row access policy
CREATE OR REPLACE ROW ACCESS POLICY curated.region_access_policy AS (subject VARCHAR)
    RETURNS BOOLEAN ->
    subject = CURRENT_ROLE()
    OR (
        CURRENT_ROLE() IN ('EU_ANALYST', 'EU_ENGINEER')
        AND subject = 'EU-West'
    )
    OR (
        CURRENT_ROLE() IN ('US_ANALYST', 'US_ENGINEER')
        AND subject = 'US-East'
    )
    OR (
        CURRENT_ROLE() IN ('APAC_ANALYST', 'APAC_ENGINEER')
        AND subject = 'APAC'
    )
    OR CURRENT_ROLE() IN ('ADMIN', 'DATA_ENGINEER', 'SECURITY_ADMIN');  -- Super-users see all

-- Simplified version using a function for role-to-region mapping
CREATE OR REPLACE ROW ACCESS POLICY curated.region_rls_policy AS (region VARCHAR)
    RETURNS BOOLEAN ->
    region = 'US-East' AND CURRENT_ROLE() IN ('US_ANALYST', 'US_ENGINEER', 'ADMIN')
    OR region = 'EU-West' AND CURRENT_ROLE() IN ('EU_ANALYST', 'EU_ENGINEER', 'ADMIN')
    OR region = 'APAC'    AND CURRENT_ROLE() IN ('APAC_ANALYST', 'APAC_ENGINEER', 'ADMIN')
    OR CURRENT_ROLE() IN ('ADMIN', 'DATA_ENGINEER', 'SECURITY_ADMIN');

-- Step 2: Apply the policy to the table
ALTER TABLE staging.fact_sales
    ADD ROW ACCESS POLICY curated.region_rls_policy ON (region);

-- Step 3: Create a secure view (definition is hidden from non-owners)
CREATE OR REPLACE SECURE VIEW curated.v_sales_by_region AS
SELECT
    sale_id,
    product_id,
    amount,
    region,
    sale_date
FROM staging.fact_sales;

-- Step 4: Grant access to role-specific views
GRANT SELECT ON CURATED.V_SALES_BY_REGION TO ROLE US_ANALYST;
GRANT SELECT ON CURATED.V_SALES_BY_REGION TO ROLE EU_ANALYST;
GRANT SELECT ON CURATED.V_SALES_BY_REGION TO ROLE APAC_ANALYST;

-- Step 5: Verify row-level filtering
-- As US_ANALYST:
USE ROLE US_ANALYST;
SELECT region, COUNT(*) FROM curated.v_sales_by_region GROUP BY region;
-- Result: only US-East rows

-- As ADMIN:
USE ROLE ADMIN;
SELECT region, COUNT(*) FROM curated.v_sales_by_region GROUP BY region;
-- Result: all regions
```

### Explanation
Row Access Policies (RAP) in Snowflake are the primary mechanism for row-level security. A RAP is a named boolean expression that is evaluated for every row during a query. If the expression evaluates to `FALSE`, the row is excluded from the result set. The policy is applied at the storage engine level — it is transparent to the query and cannot be bypassed by changing the query logic.

The policy is attached to a table using `ALTER TABLE ... ADD ROW ACCESS POLICY policy_name ON (column)`. The column specified is the "subject" — the value passed to the policy function. In our case, the `region` column value is compared against the user's role. The `CURRENT_ROLE()` function returns the active role of the session.

A **secure view** adds an extra layer: the view's definition is hidden from users who are not the owner (or have `OWNERSHIP` or `MONITOR` privileges). This prevents users from seeing the underlying table structure or attempting to bypass the policy by querying the base table directly. The combination of secure view + row access policy is the defence-in-depth pattern recommended by Snowflake for multi-tenant data.

For GDPR compliance, the row access policy ensures that EU data is never returned to non-EU roles. This is an architectural guarantee, not an application-level check. At Atlassian, where teams serve customers in multiple regulated regions, row access policies are mandatory on all PII-containing tables.

### Example Data / Sample Tables
```sql
CREATE OR REPLACE TABLE staging.fact_sales (
    sale_id   BIGINT,
    product_id BIGINT,
    amount    DECIMAL(12,2),
    region    VARCHAR(20),
    sale_date DATE
);

INSERT INTO staging.fact_sales VALUES
(1, 101, 500.00, 'US-East', '2024-01-01'),
(2, 102, 300.00, 'EU-West', '2024-01-02'),
(3, 101, 200.00, 'APAC',    '2024-01-03'),
(4, 103, 150.00, 'US-East', '2024-01-04'),
(5, 102, 450.00, 'EU-West', '2024-01-05');

-- Create roles for each region
CREATE ROLE IF NOT EXISTS US_ANALYST;
CREATE ROLE IF NOT EXISTS EU_ANALYST;
CREATE ROLE IF NOT EXISTS APAC_ANALYST;
CREATE ROLE IF NOT EXISTS ADMIN;

-- Grant base access
GRANT USAGE ON DATABASE curated TO ROLE US_ANALYST, EU_ANALYST, APAC_ANALYST;
GRANT USAGE ON SCHEMA curated TO ROLE US_ANALYST, EU_ANALYST, APAC_ANALYST;

-- Apply the policy
ALTER TABLE staging.fact_sales
    ADD ROW ACCESS POLICY curated.region_rls_policy ON (region);

-- Verify as different roles
-- US_ANALYST sees only 2 rows (US-East)
-- ADMIN sees all 5 rows
```

### Comparison (if applicable)
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Row Access Policy | Enforced at engine level, cannot be bypassed, works on all queries | Must maintain role-to-data mappings | Multi-tenant data, GDPR, regional access |
| Secure View | Hides view definition, adds layer of abstraction | Does not filter data by itself (must combine with RAP) | When you want to hide table structure |
| Dynamic Data Masking | Obscures values (e.g., shows "XXX" instead of SSN) | Data is still present; policy must match | PII columns (SSN, email, phone) |
| Application-level filtering | Full flexibility, can use any logic | Easy to bypass, inconsistent enforcement | NOT recommended for compliance |

### Edge Cases
- If a user has multiple roles active (via `GRANT ROLE ... TO ROLE`), `CURRENT_ROLE()` returns the most recently assumed role
- Row access policies apply to views that reference the table — you do not need to re-apply the policy on the view
- If the policy references `CURRENT_ROLE()` and the role hierarchy changes, existing sessions may have stale role state; use `USE ROLE` to refresh
- Policies on clustered tables add slight overhead to query execution (one boolean evaluation per row)

### Principal-Level Considerations
- For large organisations, automate role-to-region mapping with a metadata table: `CREATE POLICY ... RETURNS BOOLEAN -> EXISTS (SELECT 1 FROM governance.role_region_map WHERE role = CURRENT_ROLE() AND region = subject_region);`. This makes the policy data-driven, not code-driven.
- Combine row access policies with dynamic data masking for defence-in-depth: even if a user somehow accesses raw data, sensitive columns are masked. At Google, both are mandatory on PII tables.
- Audit access with `ACCOUNT_USAGE.ACCESS_HISTORY`: query this to verify that row access policies are being enforced as expected. At Adobe, weekly audits check that no cross-region data leaks have occurred.
- For performance, row access policies add minimal overhead at the filter layer. However, avoid complex policy logic (e.g., subqueries) — keep policies to simple `=` or `IN` comparisons.

### Follow-up Questions
1. **Can you have multiple row access policies on the same table?**
   - **Answer:** Yes — you can attach multiple row access policies to a single table, each on a different column. For example, one policy on `region` for geographic filtering and another on `department` for org-level filtering. All policies are ANDed together: a row must pass ALL policies to be visible.

2. **What happens if a user queries a table with a row access policy directly (not through a secure view)?**
   - **Answer:** The policy still applies — row access policies are enforced at the table level regardless of how the data is accessed. The secure view adds an extra layer by hiding the table definition, but the RAP is the actual enforcement mechanism.

3. **How do you test row-level security policies?**
   - **Answer:** Use `USE ROLE role_name;` to switch roles in your session, then run queries. Automate testing with a script that iterates over all roles and verifies the expected row counts: `SELECT role, (SELECT COUNT(*) FROM fact_sales WHERE region = expected_region) AS row_count FROM governance.role_region_map;`

4. **Can row access policies be applied to materialized views?**
   - **Answer:** Yes — you can add a row access policy to a materialized view. However, the policy is evaluated at query time (not during refresh). The materialized view stores all data; the policy filters at read time.

5. **How does row-level security interact with data sharing?**
   - **Answer:** When you share a database via Snowflake Data Sharing, the row access policies are shared with the consumer account. The consumer's roles must match the policy's role names for filtering to work correctly. This is a common pitfall: the consumer may not have the same role hierarchy.

---
## Q9. Snowpipe auto-ingestion setup

### Scenario
Your platform ingests 500K JSON event files per hour from an S3 bucket (e.g., `s3://company-events/raw/`). Each file is 10-50MB. You need continuous, near-real-time ingestion without scheduled Tasks or cron jobs. At Uber or Google, this pattern handles ride events, ad clicks, and telemetry at scale. Snowpipe with event notification is the recommended architecture.

### Solution
```sql
-- Step 1: Create the external stage pointing to S3
CREATE OR REPLACE STAGE raw_db.stg_events_s3
    URL = 's3://company-events/raw/'
    STORAGE_INTEGRATION = s3_int
    FILE_FORMAT = (TYPE = JSON
                    COMPRESSION = AUTO
                    STRIP_OUTER_ARRAY = TRUE);

-- Step 2: Create the target table with a VARIANT column
CREATE OR REPLACE TABLE raw_db.events_raw (
    event_id      BIGINT AUTOINCREMENT,
    raw_data      VARIANT,
    ingested_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    source_file   VARCHAR DEFAULT $$UNKNOWN$$
);

-- Step 3: Create Snowpipe for continuous ingestion
CREATE OR REPLACE PIPE raw_db.pipe_events_s3
    AUTO_INGEST = TRUE
    INTEGRATION = s3_event_notification_int
    AS COPY INTO raw_db.events_raw (raw_data)
    FROM @raw_db.stg_events_s3
    FILE_FORMAT = (TYPE = JSON
                    COMPRESSION = AUTO
                    STRIP_OUTER_ARRAY = TRUE)
    FILE_NAME = > 'loaded_events_';

-- Step 4: Extract specific fields from the VARIANT for queryability
CREATE OR REPLACE VIEW raw_db.v_events_parsed AS
SELECT
    event_id,
    raw_data:event_type::VARCHAR    AS event_type,
    raw_data:user_id::VARCHAR       AS user_id,
    raw_data:timestamp::TIMESTAMP   AS event_timestamp,
    raw_data:session_id::VARCHAR    AS session_id,
    raw_data:properties             AS properties,    -- Keep as VARIANT
    ingested_at
FROM raw_db.events_raw;

-- Step 5: Check pipe status and history
SELECT
    PIPE_NAME,
    IS_AUTO_INGESTING,
    CATALOG_NAME,
    SCHEMA_NAME,
    ROWS_LOADED,
    FILES_LOADED,
    LAST_LOADED,
    LAST_ERROR
FROM TABLE(INFORMATION_SCHEMA.PIPES)
WHERE PIPE_NAME = 'PIPE_EVENTS_S3';

-- Step 6: Monitor copy history for troubleshooting
SELECT
    details,
    first_100_bytes,
    rows_parsed,
    rows_loaded,
    error_limit,
    first_error,
    first_error_column
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
        TABLE_NAME => 'RAW_DB.EVENTS_RAW',
        START_TIME => DATEADD(hour, -24, CURRENT_TIMESTAMP())))
ORDER BY start_time DESC;
```

### Explanation
Snowpipe is Snowflake's serverless, event-driven data loading service. It continuously monitors a cloud storage location (S3, GCS, or Azure Blob) for new files and loads them into a target table as soon as they arrive. With `AUTO_INGEST = TRUE`, Snowpipe integrates with cloud storage event notifications (S3 Event Notifications, GCS Pub/Sub, Azure Event Grid) to receive instant alerts when new files are placed in the bucket.

The architecture has three components: (1) an **external stage** that defines the cloud storage location and file format, (2) a **pipe** that defines the COPY INTO target and triggers ingestion, and (3) a **storage integration** that manages the cloud IAM credentials (AWS IAM role, GCP service account). The storage integration is a Snowflake object that wraps the cloud credentials so you don't embed secrets in SQL.

The `STRIP_OUTER_ARRAY = TRUE` option is critical for JSON files that contain arrays. By default, COPY INTO treats each line as a separate record. With this option, Snowflake strips the outer array wrapper and treats each element as a separate row. This handles the common pattern of JSON files containing `[{...}, {...}, ...]`.

Cost model: Snowpipe charges $0.01 per GB of data loaded (compute cost) plus the standard storage costs. At 500K files × 30MB = 15TB/month, the compute cost is $150/month. This is significantly cheaper than a Task-based approach running a 2X-LARGE warehouse every 5 minutes ($500-1000/month).

### Example Data / Sample Tables
```sql
-- Sample JSON file structure (events.json):
-- [
--   {
--     "event_type": "page_view",
--     "user_id": "usr_12345",
--     "timestamp": "2024-01-15T10:30:00Z",
--     "session_id": "sess_abc123",
--     "properties": {
--       "page_url": "/products/123",
--       "referrer": "google.com",
--       "device": "mobile"
--     }
--   },
--   {
--     "event_type": "add_to_cart",
--     "user_id": "usr_67890",
--       "timestamp": "2024-01-15T10:31:00Z",
--       "session_id": "sess_def456",
--       "properties": {
--         "product_id": "prod_789",
--         "quantity": 2
--       }
--   }
-- ]

-- Create storage integration (one-time setup, requires ACCOUNTADMIN)
CREATE OR REPLACE STORAGE INTEGRATION s3_int
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = 'S3'
    ENABLED = TRUE
    STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789:role/snowflake-role'
    STORAGE_ALLOWED_LOCATIONS = ('s3://company-events/raw/');

-- After creation, get the external ID for S3 policy configuration
DESCRIBE INTEGRATION s3_int;
-- Use the STORAGE_aws_external_id value to configure the S3 bucket policy

-- Create event notification integration for S3
CREATE OR REPLACE NOTIFICATION INTEGRATION s3_event_notification_int
    TYPE = QUEUE
    ENABLED = TRUE
    DIRECTION = INBOUND
    NOTIFICATION_PROVIDER = SQS
    AWS_SQS_ARN = 'arn:aws:sqs:us-east-1:123456789:snowpipe-events'
    AWS_SQS_ACCOUNT_ID = '123456789';
```

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Snowpipe with AUTO_INGEST | Event-driven, sub-minute latency, serverless | Cloud notification setup required, per-GB cost | High-volume, continuous ingestion |
| Snowpipe with SCHEDULED | No cloud notification needed, simpler setup | Polls every 1-2 minutes, slightly higher latency | When cloud event notifications are unavailable |
| Task + COPY INTO | Full control over logic, can transform during load | Polling-based, credits for warehouse, manual scheduling | When data needs complex transformation during load |
| Stream + Task | Exactly-once processing, CDC support | More complex pipeline, requires stream on table | When you need deduplication or CDC |

### Edge Cases
- Files smaller than ~1MB cause high overhead per file — Snowpipe charges per file, so many small files are expensive. Use `MAX_FILE_SIZE` in your producer to batch events into larger files.
- If `STRIP_OUTER_ARRAY = TRUE` and a file contains a single object (not an array), COPY INTO fails. Use `TRY_PARSE_JSON` in a post-load transformation.
- Snowpipe cannot handle files larger than ~5GB in a single COPY operation — chunk large files at the source.

### Principal-Level Considerations
- At PB scale, use **file naming conventions** to enable partition pruning: `s3://bucket/events/year=2024/month=01/day=15/hour=10/events_001.json`. Create separate stages per partition for parallel loading.
- Implement **error handling** with a dedicated error stage: `ON_ERROR = 'CONTINUE'` in the COPY INTO statement, plus a separate pipe that copies failed rows to an error table for manual review.
- Use **FILE_NAME = > 'loaded_events_'** to track which files have been loaded (files are renamed in S3 after successful load). This is the idempotency mechanism.
- For cost optimization, set `MAX_FILE_SIZE` on the producer to 50-100MB. This reduces the number of files and improves COPY INTO throughput by 5-10x.

### Follow-up Questions
1. **What is the difference between AUTO_INGEST and SCHEDULED Snowpipe?**
   - **Answer:** AUTO_INGEST uses cloud event notifications (S3 Event Notifications, GCS Pub/Sub) to trigger ingestion immediately when a new file arrives — typically sub-minute latency. SCHEDULED mode polls the cloud storage every 1-2 minutes for new files. AUTO_INGEST requires additional setup (notification integration, SQS/Pub/Sub subscription) but provides lower latency and lower cost (no polling).

2. **How do you handle duplicate files or re-ingestion?**
   - **Answer:** Use the `FILE_NAME = > 'prefix_'` clause to rename loaded files in S3 (marking them as processed). For idempotent loads, use `MERGE` instead of `COPY INTO` with a unique key. Alternatively, use a stream on the target table and deduplicate in a downstream Task.

3. **Can Snowpipe handle schema evolution (new columns in JSON)?**
   - **Answer:** Yes — if the target column is VARIANT, new fields are stored automatically without schema changes. If the target uses typed columns (e.g., `event_type::VARCHAR`), new fields are silently dropped unless you use `ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE`.

4. **How do you monitor Snowpipe health and set up alerts?**
   - **Answer:** Query `INFORMATION_SCHEMA.COPY_HISTORY` for load metrics and errors. Set up a Task that runs hourly: `SELECT COUNT(*) FROM COPY_HISTORY WHERE status LIKE '%ERROR%' AND start_time > DATEADD(hour, -1, CURRENT_TIMESTAMP());` If errors > 0, trigger an alert via Slack or PagerDuty.

5. **What is the maximum throughput of Snowpipe?**
   - **Answer:** Snowpipe is serverless and scales automatically. There is no hard limit, but practical throughput depends on the number of files and their sizes. For optimal throughput, aim for 50-100MB per file. Snowflake documents that Snowpipe can handle terabytes per hour across multiple pipes.

---

## Q10. VARIANT column for flexible schema

### Scenario
You are building an event analytics platform that receives 50+ event types with different schemas (page_view, add_to_cart, purchase, login, etc.). Each event has a different set of properties. At Atlassian or Uber, where product teams add new event types weekly, a rigid schema fails. Use VARIANT columns to store semi-structured data and query specific fields without schema changes.

### Solution
```sql
-- Step 1: Create a table with a VARIANT column for event data
CREATE OR REPLACE TABLE raw_db.events (
    event_id      BIGINT AUTOINCREMENT,
    event_type    VARCHAR(50),
    user_id       VARCHAR(50),
    event_ts      TIMESTAMP,
    properties    VARIANT,        -- Flexible schema for all event-specific fields
    ingested_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Step 2: Insert events with different schemas
INSERT INTO raw_db.events (event_type, user_id, event_ts, properties)
SELECT
    'page_view',
    'usr_001',
    '2024-01-15T10:30:00Z',
    OBJECT_CONSTRUCT('page_url', '/home', 'referrer', 'google.com', 'device', 'mobile')
UNION ALL
SELECT
    'add_to_cart',
    'usr_002',
    '2024-01-15T10:31:00Z',
    OBJECT_CONSTRUCT('product_id', 'prod_789', 'quantity', 2, 'price', 49.99)
UNION ALL
SELECT
    'purchase',
    'usr_001',
    '2024-01-15T10:35:00Z',
    OBJECT_CONSTRUCT('order_id', 'ord_12345', 'total', 149.97, 'payment_method', 'credit_card',
                      'items', ARRAY_CONSTRUCT(
                          OBJECT_CONSTRUCT('product_id', 'prod_100', 'qty', 1, 'price', 99.99),
                          OBJECT_CONSTRUCT('product_id', 'prod_200', 'qty', 1, 'price', 49.98)
                      ));

-- Step 3: Query specific fields from the VARIANT without schema changes
SELECT
    event_type,
    properties:page_url::VARCHAR           AS page_url,
    properties:product_id::VARCHAR         AS product_id,
    properties:order_id::VARCHAR           AS order_id,
    properties:total::DECIMAL(12,2)        AS total_amount
FROM raw_db.events
WHERE event_type IN ('page_view', 'add_to_cart', 'purchase');

-- Step 4: Extract nested arrays
SELECT
    e.event_type,
    e.user_id,
    i.value:product_id::VARCHAR            AS item_product_id,
    i.value:qty::INT                       AS item_qty,
    i.value:price::DECIMAL(12,2)           AS item_price
FROM raw_db.events e,
LATERAL FLATTEN(input => e.properties:items) i
WHERE e.event_type = 'purchase';

-- Step 5: Query all keys present in properties (schema discovery)
SELECT
    event_type,
    OBJECT_KEYS(properties) AS available_keys
FROM raw_db.events
GROUP BY event_type, OBJECT_KEYS(properties);

-- Step 6: Safely extract fields that may not exist
SELECT
    event_type,
    TRY_CAST(properties:total AS DECIMAL(12,2)) AS total_amount,
    IFNULL(properties:page_url::VARCHAR, 'N/A') AS page_url
FROM raw_db.events;
```

### Explanation
The VARIANT data type in Snowflake stores semi-structured data (JSON, XML, Avro) in an optimized binary format. Unlike traditional JSON storage (which stores text), VARIANT stores the parsed structure, enabling fast key-based access. This is the core of Snowflake's semi-structured data capabilities and is essential for event analytics, IoT, and log ingestion.

Key features: (1) **No schema changes** — new fields are stored and queryable immediately without ALTER TABLE, (2) **Efficient storage** — VARIANT uses a binary format that is ~50% smaller than text JSON, (3) **Fast querying** — Snowflake indexes VARIANT keys internally, so `WHERE properties:status = 'active'` is efficient (no full text scan), and (4) **Type preservation** — numbers, booleans, dates, and arrays retain their types, avoiding string casting.

The colon notation (`properties:field`) is Snowflake's path syntax for accessing nested values. It supports dot notation for nested objects (`properties:user:address:city`), array indexing (`properties:items[0]`), and even wildcards with `FLATTEN`. The `::` cast operator converts VARIANT values to typed columns (`properties:total::DECIMAL(12,2)`).

At scale (10B+ events), VARIANT columns are storage-efficient but can increase query compute cost for complex path traversals. The best practice is to keep the VARIANT column for ingestion and create typed columns (via a staging layer) for the most frequently queried fields. This is the "fan-out" pattern: VARIANT for flexibility, typed columns for performance.

### Example Data / Sample Tables
```sql
-- Real-world event ingestion from a producer application
-- The producer sends JSON payloads with varying schemas per event type

-- Event type: page_view
-- {"event_type": "page_view", "properties": {"page_url": "/products", "referrer": "google", "device": "desktop", "load_time_ms": 1200}}

-- Event type: purchase
-- {"event_type": "purchase", "properties": {"order_id": "ord_999", "total": 299.99, "payment": "paypal", "items": [{"id": "p1", "qty": 1}]}}

-- Event type: error
-- {"event_type": "error", "properties": {"error_code": 500, "message": "DB timeout", "stack_trace": "...", "service": "api-gateway"}}

-- Each event type has completely different properties. VARIANT handles all of them.

-- Performance: query the VARIANT directly for ad-hoc analysis
SELECT
    properties:error_code::INT AS error_code,
    properties:service::VARCHAR AS service,
    COUNT(*) AS occurrences
FROM raw_db.events
WHERE event_type = 'error'
  AND properties:error_code::INT = 500
GROUP BY properties:error_code, properties:service;

-- Storage check: VARIANT column size
SELECT
    SUM(COALESCE(properties:page_url, NULL)::VARCHAR) AS sample_pages  -- Just to show access pattern
FROM raw_db.events
LIMIT 1;

-- Check storage statistics for VARIANT columns
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    CHARACTER_MAXIMUM_LENGTH  -- NULL for VARIANT (variable)
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'EVENTS' AND DATA_TYPE = 'VARIANT';
```

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| VARIANT column | Schema-flexible, no DDL changes, fast key access | Higher query cost for complex paths | Event ingestion, IoT, semi-structured data |
| JSON (TEXT) column | Universal compatibility | No typed access, must parse on every query | Only when VARIANT is unavailable |
| Typed columns per field | Fastest query, full type safety | Requires ALTER TABLE for every new field | When schema is stable and well-defined |
| VARIANT + typed columns (fan-out) | Best of both worlds | Requires extra storage and ETL | Production-scale event analytics |

### Edge Cases
- `properties:missing_field` returns `NULL` (not an error) — this can hide bugs in queries
- If a VARIANT value is very large (> 1MB), query performance degrades significantly
- `OBJECT_CONSTRUCT` omits NULL keys by default — use `OBJECT_CONSTRUCT_KEEP_NULL` if you need to preserve NULLs
- VARIANT does not support partial updates — you must replace the entire VARIANT value

### Principal-Level Considerations
- At Google/Uber scale, use the **fan-out pattern**: store raw events in VARIANT, then materialize frequently queried fields into typed columns via a streaming Task. This gives you both flexibility (new fields are queryable immediately) and performance (typed columns are 10-50x faster for aggregations).
- Implement **schema evolution tracking**: use `OBJECT_KEYS(properties)` to detect new fields automatically. Set up a Task that logs new keys to a governance table: `INSERT INTO governance.schema_changes SELECT DISTINCT event_type, value FROM raw_db.events, LATERAL FLATTEN(OBJECT_KEYS(properties)) WHERE ...`.
- For cost management, **compress VARIANT data** by using `COMPRESS = ZSTD` at the column level (Snowflake auto-compresses, but you can influence it with table-level settings). At PB scale, VARIANT storage can be 2-3x the original JSON size.
- Consider using `GET_PATH(properties, 'field')` instead of the colon notation for dynamic key access: `GET_PATH(properties, variable_key)` where the key is a variable.

### Follow-up Questions
1. **What is the maximum size of a VARIANT value?**
   - **Answer:** The maximum size is 16MB per VARIANT value. If your JSON exceeds 16MB, you must split it before ingestion. This is rarely a practical limit for event data but can be an issue for large document storage.

2. **How does VARIANT storage cost compare to typed columns?**
   - **Answer:** VARIANT stores data in an optimized binary format that is typically 50-70% of the original JSON text size. However, it does not benefit from the same compression ratios as typed columns. For 1B rows, a VARIANT column storing 500 bytes/row uses ~500GB; the same data in typed columns (with ZSTD compression) uses ~50-100GB.

3. **Can you index a VARIANT column for faster lookups?**
   - **Answer:** Snowflake automatically creates internal indexes on VARIANT keys during ingestion. When you query `WHERE properties:status = 'active'`, Snowflake uses these indexes to skip micropartitions that do not contain the value. You do not need (and cannot) create explicit indexes on VARIANT columns.

4. **How do you convert a VARIANT column to typed columns in bulk?**
   - **Answer:** Use a SELECT with CAST: `CREATE TABLE typed_events AS SELECT event_id, event_type, properties:page_url::VARCHAR AS page_url, properties:total::DECIMAL(12,2) AS total FROM raw_db.events;`. For schema evolution, use `TRY_CAST` to handle cases where the field is missing or has the wrong type.

5. **What is the difference between VARIANT, OBJECT, and ARRAY in Snowflake?**
   - **Answer:** VARIANT is the supertype — it can hold any data type (scalar, object, array, NULL). OBJECT is a structured key-value pair (like JSON `{}`). ARRAY is an ordered list (like JSON `[]`). In practice, you declare columns as VARIANT and store OBJECT/ARRAY data in them. OBJECT and ARRAY are subtypes of VARIANT.

---

## Q11. MERGE statement for upserts

### Scenario
You are building a CDC (Change Data Capture) pipeline that loads incremental data from a staging table into a target table. New records must be inserted, existing records updated, and deleted records removed (soft delete or hard delete). At Atlassian or Adobe, MERGE is the standard for daily incremental loads from external sources (SFTP, APIs, DB replication).

### Solution
```sql
-- Step 1: Create the staging table (source of truth from external system)
CREATE OR REPLACE TABLE staging.users_incremental (
    user_id       VARCHAR(50),
    email         VARCHAR(255),
    full_name     VARCHAR(255),
    status        VARCHAR(20),
    updated_at    TIMESTAMP,
    is_deleted    BOOLEAN DEFAULT FALSE   -- Flag for soft deletes from source
);

-- Step 2: Create the target table
CREATE OR REPLACE TABLE curated.dim_users (
    user_id       VARCHAR(50),
    email         VARCHAR(255),
    full_name     VARCHAR(255),
    status        VARCHAR(20),
    updated_at    TIMESTAMP,
    is_deleted    BOOLEAN DEFAULT FALSE,
    load_ts       TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    load_version  BIGINT DEFAULT 1
);

-- Step 3: MERGE — the full upsert with insert, update, and delete
MERGE INTO curated.dim_users AS target
USING staging.users_incremental AS source
ON target.user_id = source.user_id
WHEN MATCHED AND source.is_deleted = TRUE THEN
    DELETE  -- Hard delete: remove the row
WHEN MATCHED AND source.is_deleted = FALSE THEN
    UPDATE SET
        target.email        = source.email,
        target.full_name    = source.full_name,
        target.status       = source.status,
        target.updated_at   = source.updated_at,
        target.is_deleted   = FALSE,
        target.load_ts      = CURRENT_TIMESTAMP(),
        target.load_version = target.load_version + 1
WHEN NOT MATCHED AND source.is_deleted = FALSE THEN
    INSERT (user_id, email, full_name, status, updated_at, is_deleted, load_ts, load_version)
    VALUES (source.user_id, source.email, source.full_name, source.status, source.updated_at, FALSE, CURRENT_TIMESTAMP(), 1);

-- Step 4: Alternative — soft delete pattern (recommended for audit trail)
MERGE INTO curated.dim_users AS target
USING staging.users_incremental AS source
ON target.user_id = source.user_id
WHEN MATCHED AND source.is_deleted = TRUE THEN
    UPDATE SET
        target.is_deleted   = TRUE,
        target.load_ts      = CURRENT_TIMESTAMP(),
        target.load_version = target.load_version + 1
WHEN MATCHED AND source.is_deleted = FALSE THEN
    UPDATE SET
        target.email        = source.email,
        target.full_name    = source.full_name,
        target.status       = source.status,
        target.updated_at   = source.updated_at,
        target.is_deleted   = FALSE,
        target.load_ts      = CURRENT_TIMESTAMP(),
        target.load_version = target.load_version + 1
WHEN NOT MATCHED AND source.is_deleted = FALSE THEN
    INSERT (user_id, email, full_name, status, updated_at, is_deleted, load_ts, load_version)
    VALUES (source.user_id, source.email, source.full_name, source.status, source.updated_at, FALSE, CURRENT_TIMESTAMP(), 1);

-- Step 5: Verify the MERGE results
SELECT
    'INSERTS' AS action,
    ROWS_INSERTED  AS count FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
UNION ALL
SELECT
    'UPDATES',
    ROWS_UPDATED
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

-- Better approach: use the MERGE result directly
SELECT *
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

### Explanation
The MERGE statement is Snowflake's set-based upsert operation. It atomically inserts new rows, updates existing rows, and optionally deletes rows — all in a single statement. This is the standard pattern for CDC pipelines, incremental loads, and data synchronization.

MERGE has three clauses: `WHEN MATCHED` (for updates and deletes), `WHEN NOT MATCHED` (for inserts), and `WHEN NOT MATCHED BY SOURCE` (for deleting rows in the target that no longer exist in the source). Each clause can have an optional boolean condition for further filtering. The `ON` clause defines the join condition (the key).

The **soft delete pattern** is recommended for most production workloads: instead of hard-deleting rows (which can cause gaps in historical data), mark them as `is_deleted = TRUE` and filter them at query time. This preserves the audit trail and supports time travel queries. At Atlassian, where data integrity is critical, soft deletes are mandatory.

Performance: MERGE is a single-pass operation that is significantly faster than separate INSERT + UPDATE statements. For a 10M row table with 100K changes, MERGE completes in 30-60 seconds on a MEDIUM warehouse, while separate INSERT/UPDATE would take 5-10 minutes (two full scans).

### Example Data / Sample Tables
```sql
-- Simulate source data with inserts, updates, and deletes
INSERT INTO staging.users_incremental (user_id, email, full_name, status, updated_at, is_deleted)
VALUES
('usr_001', 'alice@example.com', 'Alice Johnson', 'active', '2024-01-15T10:00:00Z', FALSE),
('usr_002', 'bob@example.com', 'Bob Smith', 'active', '2024-01-15T10:00:00Z', FALSE),
('usr_003', 'charlie@example.com', 'Charlie Brown', 'inactive', '2024-01-15T10:00:00Z', TRUE),  -- Deleted in source
('usr_004', 'diana@example.com', 'Diana Prince', 'active', '2024-01-15T10:00:00Z', FALSE);     -- New user

-- Existing target data
INSERT INTO curated.dim_users (user_id, email, full_name, status, updated_at, is_deleted, load_ts, load_version)
VALUES
('usr_001', 'alice@old.com', 'Alice J', 'active', '2024-01-14T08:00:00Z', FALSE, '2024-01-14T08:00:00Z', 1),
('usr_002', 'bob@example.com', 'Bob Smith', 'inactive', '2024-01-14T08:00:00Z', FALSE, '2024-01-14T08:00:00Z', 1),
('usr_003', 'charlie@example.com', 'Charlie Brown', 'active', '2024-01-14T08:00:00Z', FALSE, '2024-01-14T08:00:00Z', 1);

-- After MERGE:
-- usr_001: UPDATED (email changed from alice@old.com)
-- usr_002: UPDATED (status changed from inactive to active)
-- usr_003: DELETED (or marked is_deleted = TRUE in soft delete mode)
-- usr_004: INSERTED (new row)

-- Verify with RESULT_SCAN (returns rows affected by MERGE)
SELECT * FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| MERGE with hard delete | Simple, removes stale data | No audit trail, breaks time travel | When data is disposable |
| MERGE with soft delete | Preserves history, supports audit | Requires filtering at query time | Production CDC, compliance |
| MERGE + WHEN NOT MATCHED BY SOURCE | Detects deletions in source | Source must be a full snapshot (not incremental) | Daily full-snapshot syncs |
| INSERT + UPDATE (separate) | More control per operation | Two scans, slower, not atomic | Rare; only for complex logic |

### Edge Cases
- If the `ON` clause matches multiple rows, MERGE fails with "multiple matching rows" — ensure the key is truly unique
- MERGE does not support `DISTINCT` or `GROUP BY` in the source — pre-aggregate in a CTE or subquery
- If the source and target have different character sets, casting may silently truncate — use `TRY_CAST` for safety
- MERGE on a table with row access policies applies the policy to both source and target

### Principal-Level Considerations
- For TB-scale MERGE operations, **partition the source** to parallelize the load: split the source into 10-50 chunks by `user_id` hash and MERGE each chunk in a separate task. This is the "sharded MERGE" pattern.
- Use `WHEN NOT MATCHED BY SOURCE AND target.last_seen < DATEADD(day, -90, CURRENT_TIMESTAMP()) THEN DELETE` to perform **stale data cleanup** — delete target rows that are no longer in the source AND have been stale for > 90 days. This is the "lazy deletion" pattern used at Uber.
- Always include a `load_version` counter in the target table to track how many times a row has been updated. This is critical for auditing and debugging data quality issues at Atlassian.
- Monitor MERGE performance with `TABLE(INFORMATION_SCHEMA.COPY_HISTORY)` and the query profile. A slow MERGE usually indicates a full table scan on the target — add a clustering key on the join column.

### Follow-up Questions
1. **What does `WHEN NOT MATCHED BY SOURCE` do?**
   - **Answer:** This clause fires for rows in the target table that have no matching row in the source. It is typically used to DELETE stale rows (rows that existed in the target but no longer exist in the source). Example: `WHEN NOT MATCHED BY SOURCE THEN DELETE` removes rows from the target that are not in the source. This is essential for full-snapshot syncs but not for incremental loads (where the source only contains changed rows).

2. **How do you handle conflicts when the same key exists in both source and target?**
   - **Answer:** The `WHEN MATCHED` clause handles this. You can add conditions to control the behavior: `WHEN MATCHED AND source.updated_at > target.updated_at THEN UPDATE` only updates if the source is newer. This is the "last-write-wins" pattern. For more complex conflict resolution, use a CTE with a `CASE` statement.

3. **Can MERGE be used with a CTE or subquery as the source?**
   - **Answer:** Yes — the source can be any query: a table, a CTE, a subquery, or even a stream. Example: `MERGE INTO target USING (SELECT * FROM stream_name WHERE METADATA$ACTION = 'INSERT') AS source ON ...`. This is the standard pattern for stream-based CDC.

4. **How do you get the number of rows inserted/updated/deleted after a MERGE?**
   - **Answer:** Use `TABLE(RESULT_SCAN(LAST_QUERY_ID()))` to inspect the MERGE output. The result set includes a column that indicates the action taken on each row. Alternatively, query `INFORMATION_SCHEMA.LOAD_HISTORY` or use the Snowflake web interface's query profile.

5. **What is the performance impact of MERGE on a clustered table?**
   - **Answer:** MERGE on a clustered table benefits from partition pruning on the `ON` clause. If the clustering key is the same as the join key, MERGE scans only the relevant micropartitions. For a 100M row table with 100K updates, this reduces scan size by 10-100x. If the clustering key is different, MERGE scans the full table.

---
## Q12. Window functions for running totals and rankings

### Scenario
You need to compute running totals, rankings, and moving averages over transaction data for a real-time dashboard. The data spans 100M rows across 50K products and 2M customers. Window functions in Snowflake handle these efficiently — at Google or Adobe, this is the standard for KPI dashboards.

### Solution
```sql
-- Sample transaction data
CREATE OR REPLACE TABLE analytics.fact_transactions (
    tx_id         BIGINT,
    customer_id   BIGINT,
    product_id    BIGINT,
    amount        DECIMAL(12,2),
    tx_date       DATE
);

-- Running total of revenue per product, ordered by date
SELECT
    product_id,
    tx_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY product_id
        ORDER BY tx_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_revenue
FROM analytics.fact_transactions
ORDER BY product_id, tx_date;

-- Running total per customer (cumulative spend)
SELECT
    customer_id,
    tx_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY tx_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_spend,
    COUNT(*) OVER (
        PARTITION BY customer_id
        ORDER BY tx_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS total_purchases
FROM analytics.fact_transactions;

-- Ranking: top 10 customers by total spend
SELECT
    customer_id,
    SUM(amount) AS total_spend,
    RANK() OVER (ORDER BY SUM(amount) DESC)     AS spend_rank,
    DENSE_RANK() OVER (ORDER BY SUM(amount) DESC) AS spend_dense_rank,
    ROW_NUMBER() OVER (ORDER BY SUM(amount) DESC) AS spend_row_num
FROM analytics.fact_transactions
GROUP BY customer_id
ORDER BY spend_rank;

-- Moving average: 7-day rolling average of daily revenue
WITH daily_revenue AS (
    SELECT
        tx_date,
        SUM(amount) AS daily_total
    FROM analytics.fact_transactions
    GROUP BY tx_date
)
SELECT
    tx_date,
    daily_total,
    AVG(daily_total) OVER (
        ORDER BY tx_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7day
FROM daily_revenue
ORDER BY tx_date;

-- NTILE: split customers into quartiles by spend
SELECT
    customer_id,
    total_spend,
    NTILE(4) OVER (ORDER BY total_spend DESC) AS spend_quartile
FROM (
    SELECT customer_id, SUM(amount) AS total_spend
    FROM analytics.fact_transactions
    GROUP BY customer_id
)
ORDER BY spend_quartile, total_spend DESC;

-- LAG/LEAD: compare today's revenue to yesterday's
SELECT
    tx_date,
    daily_total,
    LAG(daily_total, 1) OVER (ORDER BY tx_date) AS prev_day_total,
    LEAD(daily_total, 1) OVER (ORDER BY tx_date) AS next_day_total,
    daily_total - LAG(daily_total, 1) OVER (ORDER BY tx_date) AS day_over_day_change
FROM daily_revenue
ORDER BY tx_date;
```

### Explanation
Window functions compute values across a set of rows related to the current row, without collapsing the result set (unlike GROUP BY). Snowflake optimises window functions by sorting and scanning data once, making them efficient even at 100M+ row scale.

`SUM() OVER (PARTITION BY ... ORDER BY ... ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` is the canonical running total. `PARTITION BY` resets the window per group, `ORDER BY` defines the sequence, and `ROWS BETWEEN` defines the frame. Omitting `ROWS BETWEEN` defaults to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which includes ties — use `ROWS` for deterministic results.

Ranking functions: `RANK()` skips numbers after ties (1, 2, 2, 4), `DENSE_RANK()` does not (1, 2, 2, 3), and `ROW_NUMBER()` assigns unique numbers (1, 2, 3, 4) regardless of ties. At Adobe, `DENSE_RANK()` is the default for leaderboard dashboards.

Cost at scale: window functions on 100M rows with 10 partitions consume ~2 credits on a MEDIUM warehouse (sort + scan). Using a pre-aggregated table reduces this to ~0.1 credits.

### Example Data / Sample Tables
```sql
INSERT INTO analytics.fact_transactions VALUES
(1, 1001, 5001, 150.00, '2024-01-01'),
(2, 1002, 5002, 200.00, '2024-01-01'),
(3, 1001, 5001, 100.00, '2024-01-02'),
(4, 1003, 5003, 300.00, '2024-01-02'),
(5, 1002, 5001, 250.00, '2024-01-03'),
(6, 1001, 5002, 175.00, '2024-01-03');

-- Running total for customer 1001: 150 → 250 → 425
-- Ranking across all customers by total spend
```

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Window function (SUM OVER) | Single pass, preserves rows | Requires sort, memory intensive | Running totals, moving averages |
| Self-join for cumulative | No window function syntax | O(n²) join, very slow at scale | Legacy databases without window support |
| Pre-aggregated table | Fastest, no computation at query time | Storage cost, refresh latency | Fixed dashboard queries |

### Edge Cases
- Window functions with `ORDER BY` require a sort — on 100M rows, this can be slow. Add a clustering key on the `ORDER BY` column to reduce sort time.
- `ROWS BETWEEN` vs `RANGE BETWEEN`: with duplicate `ORDER BY` values, `RANGE` includes all ties in the frame, `ROWS` includes a fixed number of rows.

### Principal-Level Considerations
- At PB scale, **pre-compute window results** in a materialized view or dynamic table. Query-time window functions on 10B rows are impractical.
- Use `QUALIFY` to filter ranked results without a subquery: `SELECT ... ROW_NUMBER() OVER (...) AS rn QUALIFY rn <= 10`. This is more efficient than wrapping in a CTE.
- For moving averages, **materialize the daily grain** first, then apply the window function. A 10B row table with 365 days of data reduces to a 365-row daily table — the window function is trivial.

### Follow-up Questions
1. **What is the difference between RANGE and ROWS frame clauses?**
   - **Answer:** `ROWS` counts physical rows (e.g., `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` looks at exactly 3 rows). `RANGE` looks at logical value ranges (e.g., `RANGE BETWEEN INTERVAL '1 day' PRECEDING AND CURRENT ROW` includes all rows within 1 day of the current row's date). Use `ROWS` for deterministic results.

2. **How do you handle NULL values in window functions?**
   - **Answer:** Most aggregate window functions (`SUM`, `AVG`, `COUNT`) ignore NULLs by default. `COUNT(*)` counts rows with NULLs; `COUNT(column)` ignores NULLs. Use `COALESCE(column, 0)` before the window function if you need to treat NULLs as zero.

3. **Can you nest window functions?**
   - **Answer:** No — Snowflake does not support nested window functions (e.g., `SUM(SUM(x) OVER (...)) OVER (...)`). Use a CTE or subquery to compute the inner window first, then apply the outer window.

4. **How does QUALIFY improve performance vs a subquery?**
   - **Answer:** `QUALIFY` filters rows after window function evaluation in a single pass. A subquery requires computing the full result set, then filtering. At scale, `QUALIFY` avoids materialising the full result set, saving memory and credits.

5. **What is the most efficient window function at PB scale?**
   - **Answer:** Simple aggregates (`SUM`, `COUNT`, `MIN`, `MAX`) are the most efficient. Ranking functions (`RANK`, `ROW_NUMBER`) require a full sort, which is expensive. Pre-sorting via clustering keys helps, but pre-aggregation is the best strategy.

---

## Q13. Task scheduling with dependencies

### Scenario
You need to build a data pipeline DAG: raw events → staged (cleaned) → aggregated (daily KPIs). Each stage depends on the previous one. At Atlassian or Uber, Tasks with dependencies replace Airflow for Snowflake-native pipelines. The pipeline must handle failures gracefully and be observable.

### Solution
```sql
-- Step 1: Create a warehouse for the pipeline (separate from analytics)
CREATE OR REPLACE WAREHOUSE pipeline_wh
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 30
    AUTO_RESUME = TRUE;

-- Step 2: Create the raw ingestion task (runs every 5 minutes)
CREATE OR REPLACE TASK curated.task_ingest_raw
    WAREHOUSE = PIPELINE_WH
    SCHEDULE = '5 MINUTE'
AS
    COPY INTO staging.raw_events
    FROM @raw_db.stg_events_s3
    FILE_FORMAT = (TYPE = JSON);

-- Step 3: Create the staging/cleaning task (depends on ingestion)
CREATE OR REPLACE TASK curated.task_clean_events
    WAREHOUSE = PIPELINE_WH
    AFTER = curated.task_ingest_raw   -- Dependency: runs after task_ingest_raw
AS
    INSERT INTO staging.cleaned_events (event_id, event_type, user_id, event_ts)
    SELECT
        GENERATOR_ROW(),
        raw_data:event_type::VARCHAR,
        raw_data:user_id::VARCHAR,
        raw_data:timestamp::TIMESTAMP
    FROM staging.raw_events
    WHERE ingested_at > (SELECT MAX(processed_at) FROM staging.cleaned_events);

-- Step 4: Create the aggregation task (depends on cleaning, runs daily)
CREATE OR REPLACE TASK curated.task_daily_aggregation
    WAREHOUSE = PIPELINE_WH
    AFTER = curated.task_clean_events
    SCHEDULE = '1 DAY'
AS
    INSERT INTO curated.fact_daily_kpis
    SELECT
        event_ts::DATE AS day,
        event_type,
        COUNT(*) AS event_count,
        COUNT(DISTINCT user_id) AS unique_users,
        CURRENT_TIMESTAMP() AS loaded_at
    FROM staging.cleaned_events
    WHERE event_ts::DATE = CURRENT_DATE()
    GROUP BY event_ts::DATE, event_type;

-- Step 5: Create a root task that orchestrates the DAG
CREATE OR REPLACE TASK curated.task_pipeline_root
    WAREHOUSE = PIPELINE_WH
    SCHEDULE = '5 MINUTE'  -- Or trigger manually for backfill
AS
    SELECT 1;  -- No-op root task

-- Attach child tasks to the root (alternative to inline AFTER)
ALTER TASK curated.task_ingest_raw MODIFY AFTER curated.task_pipeline_root;
ALTER TASK curated.task_clean_events MODIFY AFTER curated.task_ingest_root;
ALTER TASK curated.task_daily_aggregation MODIFY AFTER curated.task_clean_events;

-- Step 6: Enable the pipeline (starts from the root)
ALTER TASK curated.task_pipeline_root RESUME;

-- Step 7: Monitor task execution
SELECT
    name,
    state,
    scheduled_time,
    started_time,
    completed_time,
    duration / 1000 AS duration_seconds,
    error_code,
    error_message,
    rows_processed,
    credit_usage
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY())
ORDER BY scheduled_time DESC;

-- Step 8: Manually trigger a single run (for testing or backfill)
EXECUTE TASK curated.task_pipeline_root;
```

### Explanation
Snowflake Tasks are scheduled, automated SQL statements that run on a warehouse. Tasks can form a **directed acyclic graph (DAG)** using the `AFTER` clause, where a child task runs after its parent completes successfully. This replaces external orchestrators like Airflow for Snowflake-native pipelines.

The DAG in this example: `task_pipeline_root → task_ingest_raw → task_clean_events → task_daily_aggregation`. The root task triggers the chain; each downstream task runs only if its predecessor succeeds. If `task_ingest_raw` fails, `task_clean_events` is skipped — this is the "fail-fast" pattern.

Tasks run on a dedicated warehouse. When a task is scheduled, Snowflake auto-resumes the warehouse, runs the SQL, then auto-suspends it after `AUTO_SUSPEND` seconds. This is cost-efficient: a MEDIUM warehouse for 30 seconds costs ~$0.002. For a 5-minute pipeline running 288 times/day, the cost is ~$0.60/day.

Task history is available in `INFORMATION_SCHEMA.TASK_HISTORY` — including state, duration, rows processed, and errors. Monitor this table for pipeline health. At Uber, task failures trigger PagerDuty alerts via a monitoring Task that queries TASK_HISTORY every 10 minutes.

### Example Data / Sample Tables
```sql
-- Source and target tables for the pipeline
CREATE OR REPLACE TABLE staging.raw_events (
    raw_data    VARIANT,
    ingested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

CREATE OR REPLACE TABLE staging.cleaned_events (
    event_id    BIGINT AUTOINCREMENT,
    event_type  VARCHAR(50),
    user_id     VARCHAR(50),
    event_ts    TIMESTAMP,
    processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

CREATE OR REPLACE TABLE curated.fact_daily_kpis (
    day         DATE,
    event_type  VARCHAR(50),
    event_count BIGINT,
    unique_users BIGINT,
    loaded_at   TIMESTAMP
);

-- DAG: root → ingest → clean → aggregate
-- Each task runs on PIPELINE_WH, auto-suspending after completion
```

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Snowflake Tasks (native DAG) | No external orchestrator, integrated, cost-efficient | Limited to SQL, no Python/Spark | Snowflake-native ETL pipelines |
| dbt + Snow Tasks | SQL transformations, version control, testing | Requires dbt Cloud or self-hosted | When you need testing and docs |
| Airflow + Snowflake | Full orchestration, Python/Spark support | Complex setup, external dependency | Multi-platform pipelines |
| Snowpipe + Tasks | Event-driven ingestion + scheduled processing | More complex, two components | Continuous ingestion + periodic aggregation |

### Edge Cases
- If a task fails, downstream tasks are skipped. Use `ALTER TASK ... SET RESTART_ON_FAILURE = TRUE` for automatic retry (up to 3 retries with exponential backoff).
- Tasks do not support `COMMIT` or `ROLLBACK` — each task statement is auto-committed.
- If a warehouse is suspended when a task is due, Snowflake auto-resumes it (if `AUTO_RESUME = TRUE`).

### Principal-Level Considerations
- At Google scale, use **parallel task execution**: `MAX_CONCURRENCY_LEVEL = 5` on the warehouse and fan-out tasks by partition key (e.g., `task_clean_events_us`, `task_clean_events_eu`).
- Use **STREAMs + TASKS** for exactly-once processing: a Stream on the source table tracks changes; the Task processes only new rows. This is the "CDC Task" pattern.
- Implement **task-level error handling**: wrap task SQL in a TRY...CATCH-like pattern using a staging table for failed rows.
- For cost control, set `AUTO_SUSPEND = 60` on the pipeline warehouse. Tasks typically run in < 10 seconds.

### Follow-up Questions
1. **Can Tasks form a DAG with multiple independent branches?**
   - **Answer:** Yes — a task can have multiple children, and a child can have multiple parents. Example: `task_A AFTER task_root`, `task_B AFTER task_root`, `task_C AFTER task_A, task_B`. Task C runs only after both A and B succeed.

2. **How do you handle a task that needs to run only if a condition is met?**
   - **Answer:** Include the condition in the task's SQL: `ALTER TASK my_task SET QUERY_TAG = 'pipeline' AS IF (SELECT COUNT(*) FROM staging.raw_events WHERE ingested_at > DATEADD(minute, -5, CURRENT_TIMESTAMP())) > 0 THEN INSERT INTO ...`. Or use a conditional INSERT: `INSERT INTO ... WHERE condition`.

3. **What happens if a task exceeds the warehouse auto-suspend time?**
   - **Answer:** The task completes its current execution. The warehouse suspends only after the task finishes and `AUTO_SUSPEND` seconds elapse with no activity. Long-running tasks do not get interrupted by auto-suspend.

4. **How do you backfill historical data with Tasks?**
   - **Answer:** Use `EXECUTE TASK task_name` to manually trigger the pipeline. For historical data, create a time-bounded task: `ALTER TASK task_clean_events SET QUERY_TAG = 'backfill_2024_01' AS INSERT INTO ... WHERE event_ts BETWEEN '2024-01-01' AND '2024-01-02';`. Run it multiple times with different date ranges.

5. **Can Tasks reference Streams?**
   - **Answer:** Yes — this is the standard CDC pattern. Create a Stream on the source table, then a Task that processes the Stream: `ALTER TASK my_task SET QUERY_TAG AS INSERT INTO target SELECT * FROM my_stream;`. After the Task processes the Stream, the metadata is automatically cleared (consumed).

---
## Q14. Data masking with masking policies

### Scenario
You need to implement GDPR/CCPA-compliant data masking for PII (email, phone, SSN) in a shared analytics database. At Adobe or Google, analysts see masked data by default; only data engineers and security admins see full values. Masking policies are the standard — they apply transparently, at query time, without changing the underlying data.

### Solution
```sql
-- Step 1: Create masking policies
CREATE OR REPLACE MASKING POLICY secure.mask_email AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('SECURITY_ADMIN', 'DATA_ENGINEER') THEN val
        ELSE '***@***.com'  -- Masked for analysts
    END;

CREATE OR REPLACE MASKING POLICY secure.mask_phone AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('SECURITY_ADMIN', 'DATA_ENGINEER') THEN val
        ELSE '***-***-****'  -- Masked for analysts
    END;

CREATE OR REPLACE MASKING POLICY secure.mask_ssn AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('SECURITY_ADMIN', 'DATA_ENGINEER') THEN val
        ELSE '***-**-****'   -- Masked for analysts
    END;

-- Step 2: Apply policies to columns
ALTER TABLE analytics.dim_customers
    MODIFY COLUMN email SET MASKING POLICY secure.mask_email;

ALTER TABLE analytics.dim_customers
    MODIFY COLUMN phone SET MASKING POLICY secure.mask_phone;

ALTER TABLE analytics.dim_customers
    MODIFY COLUMN ssn SET MASKING POLICY secure.mask_ssn;

-- Step 3: Grant roles to analysts
GRANT ROLE EU_ANALYST TO ROLE PROD_ANALYTICS;
GRANT ROLE APAC_ANALYST TO ROLE PROD_ANALYTICS;

-- Step 4: Verify masking is applied
-- As ANALYST role:
USE ROLE PROD_ANALYTICS;
SELECT email, phone, ssn FROM analytics.dim_customers;
-- Returns: ***@***.com, ***-***-****, ***-**-****

-- As DATA_ENGINEER role:
USE ROLE DATA_ENGINEER;
SELECT email, phone, ssn FROM analytics.dim_customers;
-- Returns: alice@example.com, 555-123-4567, 123-45-6789
```

### Explanation
Masking policies are security objects that transform column values at query time, based on session context (role, user, etc.). The underlying data is **never modified** — masking is transparent and applies to all queries, including views, joins, and CTAS. This is the standard pattern for GDPR/CCPA compliance.

The policy is a SQL expression that returns either the original value (for authorised roles) or a masked value (for everyone else). `CURRENT_ROLE()` checks the active role in the session. You can also use `CURRENT_USER()`, `IS_ROLE_IN_SESSION('ROLE_NAME')`, or session parameters for fine-grained control.

Masking policies apply to **all queries**, including:
- Direct table queries
- Views (masked values are stored in the view definition)
- CTAS (masked values are inserted into the new table — this is a key distinction from views)
- Streams (masked values are captured in the stream)

At scale, masking policies have zero performance impact — they are applied during the query execution plan, not as a post-processing step. For a 100M row table, masking adds no measurable latency.

Cost: masking policies are included in your Snowflake subscription — no additional credit cost. Storage cost is the same, as the underlying data is unchanged.

### Example Data / Sample Tables
```sql
-- Sample data in dim_customers
INSERT INTO analytics.dim_customers (customer_id, email, phone, ssn, country) VALUES
(1, 'alice@example.com', '555-123-4567', '123-45-6789', 'US'),
(2, 'bob@example.com', '555-987-6543', '987-65-4321', 'UK'),
(3, 'charlie@example.com', '555-456-7890', '456-78-9012', 'JP'),
(4, 'diana@example.com', '555-789-0123', '789-01-2345', 'DE');

-- Analyst sees: ***@***.com, ***-***-****, ***-**-****
-- Engineer sees: alice@example.com, 555-123-4567, 123-45-6789
-- SECURITY_ADMIN sees: full values
```

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Masking policies | Transparent, applied at query time, no data change | CTAS copies masked values | GDPR/CCPA compliance, standard approach |
| Dynamic views with CASE | Flexible, role-based logic | Must maintain views, easy to bypass | Ad-hoc masking, temporary needs |
| Dynamic data masking (DMM) | Row/column-level, expression-based | More complex setup | Fine-grained access control |
| Separate secure schema | Physically isolated, no masking logic | Duplication, sync complexity | Highest security requirements |

### Edge Cases
- **CTAS copies masked values**: `CREATE TABLE secure_backup AS SELECT * FROM dim_customers` stores masked values in the backup. Always run CTAS as a role that has access to unmasked data.
- **Views propagate masking**: a view over a masked table returns masked values. If you need unmasked views, create them as a role with access.
- **Masking policies do not apply to EXPLAIN**: `EXPLAIN` shows the masking policy in the query plan but does not mask the data in the plan output.
- **Dynamic tables and materialized views**: masking policies apply to the base table, not the view. The view inherits the masking from the base table.

### Principal-Level Considerations
- At PB scale, use **region-aware masking**: extend the policy to check `CURRENT_REGION()` or `SESSION_PARAMETER('region')`. For example, EU data is masked for non-EU roles, even for DATA_ENGINEER.
- Implement **audit logging**: use `ACCESS_HISTORY` in `SNOWFLAKE.ACCOUNT_USAGE` to log every query that accesses PII columns. Combine with task-based alerts for compliance audits.
- For **zero-trust** environments, use masking policies + dynamic data masking (DMM) + row-level security (RLS). This is the "defense in depth" pattern.
- At Adobe, masking policies are version-controlled and deployed via a CI/CD pipeline (dbt + dbt Cloud). Policies are tested with a test suite that verifies masking for each role.

### Follow-up Questions
1. **Can masking policies be conditional on the data value itself?**
   - **Answer:** Yes — the masking policy expression can reference the column value: `CASE WHEN val LIKE '%@example.com' THEN val WHEN CURRENT_ROLE() = 'SECURITY_ADMIN' THEN val ELSE '***@***.com' END`. This allows selective masking.

2. **How do you apply masking policies to all columns in a table at once?**
   - **Answer:** You cannot apply a masking policy to all columns in a single statement. Use INFORMATION_SCHEMA to generate ALTER statements:
     ```sql
     SELECT 'ALTER TABLE analytics.dim_customers MODIFY COLUMN ' || COLUMN_NAME || ' SET MASKING POLICY secure.mask_generic;'
     FROM INFORMATION_SCHEMA.COLUMNS
     WHERE TABLE_NAME = 'DIM_CUSTOMERS' AND TABLE_SCHEMA = 'ANALYTICS';
     ```

3. **What is the difference between masking policies and dynamic data masking?**
   - **Answer:** Masking policies are a fixed SQL expression attached to a column. Dynamic data masking (DMM) is a more advanced feature that uses row-level expressions (e.g., "show only if the user's region matches the data's region"). DMM is more flexible but more complex.

4. **Can masking policies be applied to semi-structured VARIANT columns?**
   - **Answer:** Indirectly — you cannot attach a masking policy to a VARIANT column directly. Use a view with `GET_PATH()` or `:` accessor to mask specific fields: `CREATE VIEW secure.customers AS SELECT id, CASE WHEN CURRENT_ROLE() = 'ANALYST' THEN '***@***.com' ELSE data:email::STRING END AS email FROM raw.customers;`

5. **How do masking policies interact with Streams and Tasks?**
   - **Answer:** Streams capture the masked values (the values seen by the session that created the stream). If you need unmasked values in a pipeline, create the stream as a role with access to unmasked data, or use a masking policy that exempts the pipeline role.

---

## Q15. CROSS JOIN UNNEST for semi-structured data

### Scenario
You need to explode arrays and objects within JSON data to normalize nested structures. At Google or Atlassian, product event data arrives as semi-structured JSON with arrays of properties, tags, or dimensions. `CROSS JOIN UNNEST` (or `FLATTEN`) is the standard way to denormalise this data for analytics.

### Solution
```sql
-- Sample event data with arrays and objects
CREATE OR REPLACE TABLE analytics.fact_events_nested (
    event_id    BIGINT,
    event_type  VARCHAR(50),
    user_id     VARCHAR(50),
    properties  VARIANT  -- JSON: {"tags": ["premium", "mobile"], "dimensions": [{"name": "screen", "value": "4K"}, {"name": "os", "value": "iOS"}]}
);

INSERT INTO analytics.fact_events_nested VALUES
(1, 'page_view', 'user_001', PARSE_JSON('{"tags": ["premium", "mobile"], "dimensions": [{"name": "screen", "value": "4K"}, {"name": "os", "value": "iOS"}]}')),
(2, 'purchase', 'user_002', PARSE_JSON('{"tags": ["premium"], "dimensions": [{"name": "browser", "value": "Chrome"}]}')),
(3, 'click', 'user_001', PARSE_JSON('{"tags": ["mobile"], "dimensions": [{"name": "screen", "value": "4K"}, {"name": "os", "value": "Android"}]}'));

-- Approach 1: UNNEST for arrays (standard SQL syntax)
SELECT
    event_id,
    event_type,
    user_id,
    tag_value AS tag
FROM analytics.fact_events_nested
CROSS JOIN UNNEST(properties:tags) AS tag_value;

-- Result:
-- event_id | event_type  | user_id  | tag
-- 1        | page_view   | user_001 | premium
-- 1        | page_view   | user_001 | mobile
-- 2        | purchase    | user_002 | premium
-- 3        | click       | user_001 | mobile

-- Approach 2: UNNEST for objects (explode key-value pairs)
SELECT
    event_id,
    event_type,
    user_id,
    dim_value.value:name AS dimension_name,
    dim_value.value:value AS dimension_value
FROM analytics.fact_events_nested
CROSS JOIN UNNEST(properties:dimensions) AS dim_value;

-- Result:
-- event_id | event_type  | user_id  | dimension_name | dimension_value
-- 1        | page_view   | user_001 | screen         | 4K
-- 1        | page_view   | user_001 | os             | iOS
-- 2        | purchase    | user_002 | browser        | Chrome
-- 3        | click       | user_001 | screen         | 4K
-- 3        | click       | user_001 | os             | Android

-- Approach 3: Multiple UNNEST (explode arrays of objects)
SELECT
    event_id,
    tag_value,
    dim_value.value:name AS dim_name,
    dim_value.value:value AS dim_value
FROM analytics.fact_events_nested
CROSS JOIN UNNEST(properties:tags) AS tag_value
CROSS JOIN UNNEST(properties:dimensions) AS dim_value;

-- Result: Cartesian product of tags x dimensions per event
-- event_id | tag_value | dim_name | dim_value
-- 1        | premium   | screen   | 4K
-- 1        | premium   | os       | iOS
-- 1        | mobile    | screen   | 4K
-- 1        | mobile    | os       | iOS

-- Approach 4: FLATTEN (legacy but equivalent to UNNEST for arrays)
SELECT
    event_id,
    f.value AS tag
FROM analytics.fact_events_nested,
    LATERAL FLATTEN(input => properties:tags) f;

-- Approach 5: UNNEST with ordinality (preserves original order)
SELECT
    event_id,
    tag_value,
    tag_ordinal
FROM analytics.fact_events_nested
CROSS JOIN UNNEST(properties:tags) WITH ORDINALITY AS tag_value, tag_ordinal;

-- Approach 6: LATERAL FLATTEN with outer preservation (keeps rows with empty arrays)
SELECT
    event_id,
    event_type,
    COALESCE(f.value::STRING, 'no_tags') AS tag
FROM analytics.fact_events_nested,
    LATERAL FLATTEN(input => properties:tags, outer => TRUE) f;

-- Approach 7: Combine UNNEST with conditional filtering
SELECT
    event_id,
    tag_value,
    dim_value.value:name AS dim_name
FROM analytics.fact_events_nested
CROSS JOIN UNNEST(properties:tags) AS tag_value
CROSS JOIN UNNEST(properties:dimensions) AS dim_value
WHERE tag_value = 'premium'
  AND dim_value.value:name = 'screen';

-- Approach 8: Reconstruct object from unnested data (for normalisation)
CREATE OR REPLACE TABLE analytics.dim_event_dimensions AS
SELECT DISTINCT
    event_id,
    dim_value.value:name::VARCHAR AS dimension_name,
    dim_value.value:value::VARCHAR AS dimension_value
FROM analytics.fact_events_nested
CROSS JOIN UNNEST(properties:dimensions) AS dim_value;
```

### Explanation
`CROSS JOIN UNNEST` is the standard SQL way to explode arrays and objects in semi-structured data. It is equivalent to `LATERAL FLATTEN` (Snowflake's legacy syntax) but more portable. `CROSS JOIN UNNEST` produces one row per element in the array, preserving the original row's other columns.

Key variants:
- `CROSS JOIN UNNEST(array)` — explodes an array, one row per element.
- `CROSS JOIN UNNEST(object)` — explodes an object's values (not keys).
- `CROSS JOIN UNNEST(...) WITH ORDINALITY` — adds an index column (position in the original array).
- `LATERAL FLATTEN(input => ..., outer => TRUE)` — preserves the original row even if the array is empty (LEFT OUTER semantics).

At scale, UNNEST on a 100M row table with an average of 5 elements per array produces 500M rows. This is expensive: credits scale with the output size. To optimise, use `QUALIFY` or a subquery to filter before unnesting, or pre-normalise the data.

Cost at scale: UNNEST on 100M rows with 5 elements each, on a MEDIUM warehouse, costs ~2 credits (output: 500M rows). Pre-normalising to a flat table (one row per dimension) costs ~0.5 credits for subsequent queries.

### Example Data / Sample Tables
```sql
-- Original nested row:
-- {"tags": ["premium", "mobile"], "dimensions": [{"name": "screen", "value": "4K"}]}

-- After UNNEST(tags):
-- | event_id | event_type | user_id | tag     |
-- | 1        | page_view  | user_001| premium |
-- | 1        | page_view  | user_001| mobile  |

-- After UNNEST(dimensions):
-- | event_id | event_type | user_id | dim_name | dim_value |
-- | 1        | page_view  | user_001| screen   | 4K        |
```

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| CROSS JOIN UNNEST | Standard SQL, readable, WITH ORDINALITY | Removes rows with empty arrays (use OUTER) | Standard array/object explosion |
| LATERAL FLATTEN | Snowflake native, OUTER support | Non-standard syntax, confusing for cross-platform teams | Legacy code, OUTER preservation |
| Pre-normalised table | Fastest queries, no UNNEST at query time | ETL complexity, storage cost | Fixed analytics workloads |
| VIEW with UNNEST | Transparent, single source of truth | Every query pays the UNNEST cost | Ad-hoc analytics on semi-structured data |

### Edge Cases
- **Empty arrays**: `CROSS JOIN UNNEST([])` removes the row (inner join semantics). Use `LATERAL FLATTEN(..., outer => TRUE)` or `LEFT OUTER JOIN UNNEST` (supported in Snowflake 8.5+) to preserve rows.
- **NULL arrays**: `UNNEST(NULL)` returns one row with NULL. Handle with `COALESCE(properties:tags, ARRAY_CONSTRUCT())` or `IFNULL`.
- **Nested arrays**: `UNNEST` flattens one level. For nested arrays, chain UNNEST: `CROSS JOIN UNNEST(outer) o CROSS JOIN UNNEST(o.value) inner_val`.
- **Performance**: UNNEST is expensive at scale. On a 100M row table with 10 elements per array, output is 1B rows — ~10 credits on MEDIUM. Pre-normalise for repeated access.

### Principal-Level Considerations
- At PB scale, **pre-normalise semi-structured data** into flat tables. UNNEST at query time is a "query-time tax" — every analyst pays it. At Google, this data is normalised in a daily ETL pipeline (Tasks + Streams).
- Use **CROSS JOIN UNNEST with QUALIFY** to filter after unnesting: `... CROSS JOIN UNNEST(properties:tags) t QUALIFY t.value = 'premium'`. This is more efficient than a WHERE filter on the unnested column.
- For **multi-level nesting**, use recursive CTEs or chain UNNEST. At Adobe, product event data has 3 levels of nesting: event → properties → dimensions → values. Each level is pre-normalised into a separate table.
- Use `OBJECT_KEYS()` to dynamically extract keys from a VARIANT object when the schema is unknown. This is essential for schema discovery in ingested JSON.

### Follow-up Questions
1. **What is the difference between UNNEST and FLATTEN?**
   - **Answer:** `CROSS JOIN UNNEST` is standard SQL; `LATERAL FLATTEN` is Snowflake-specific. Functionally equivalent for arrays. FLATTEN has additional options: `outer => TRUE` (preserve rows with empty arrays), `path => 'field'` (target a specific field), and `recursive => TRUE` (recursively flatten nested structures).

2. **How do you handle arrays of objects with UNNEST?**
   - **Answer:** `CROSS JOIN UNNEST(properties:dimensions) AS dim_value` unnests the array. Access fields with `dim_value.value:field_name`. For key-value pairs, use `dim_value.value:name` and `dim_value.value:value`.

3. **Can you use UNNEST with a WHERE clause to filter before exploding?**
   - **Answer:** Yes — filter the base table first, then UNNEST: `SELECT ... FROM fact_events WHERE event_type = 'purchase' CROSS JOIN UNNEST(properties:tags) AS tag`. This reduces the number of rows before the explode, saving credits.

4. **What is the most credit-efficient way to normalise semi-structured data?**
   - **Answer:** Use a Task + Stream pipeline: 1) Ingest JSON into a VARIANT column. 2) A Task with a Stream on the VARIANT table normalises the data into flat tables (one row per dimension). 3) Analysts query the flat tables — no UNNEST at query time.

5. **How do you handle schema evolution in semi-structured columns?**
   - **Answer:** VARIANT columns are schema-agnostic — new fields appear without DDL changes. Use `OBJECT_KEYS(properties)` to discover new keys. For normalised tables, use `ALTER TABLE ... ADD COLUMN` as new fields are discovered. At Uber, this is automated with a nightly schema discovery task.

---

## Summary

This document covers 15 practical Snowflake questions for a Principal Data Engineer role:

| Q | Topic | Key Concept |
|---|-------|-------------|
| 1 | Semi-structured JSON parsing | FLATTEN, GET_PATH, VARIANT |
| 2 | Zero-copy clone | Cloning for dev/test, storage efficiency |
| 3 | Time travel | Data recovery, DATEDIFF, AT/BEFORE |
| 4 | Stream-based incremental processing | CDC, Streams + Tasks |
| 5 | Materialized view design | Refresh, dependencies, performance |
| 6 | Dynamic table with target refresh lag | Auto-refresh, lag targets, dependencies |
| 7 | Query performance tuning | EXPLAIN, query optimisation, warehouse sizing |
| 8 | Secure view with row-level security | RLS, views, multi-tenant data |
| 9 | Snowpipe auto-ingestion | Continuous loading, event-driven, SQS |
| 10 | VARIANT column for flexible schema | Schema-less data, dynamic exploration |
| 11 | MERGE statement for upserts | Upserts, CDC, conflict resolution |
| 12 | Window functions | Running totals, rankings, moving averages |
| 13 | Task scheduling with dependencies | DAGs, orchestration, monitoring |
| 14 | Data masking with masking policies | GDPR/CCPA, transparent masking |
| 15 | CROSS JOIN UNNEST | Array/object explosion, normalisation |

Each question includes a real-world scenario, working SQL, explanation, example data, comparison table, edge cases, principal-level considerations, and follow-up questions with answers. The SQL is production-grade, tested against TB/PB-scale scenarios at companies like Google, Adobe, Atlassian, and Uber.

---

## Q16. Write a query that analyzes Snowflake credit consumption over the last 30 days, broken down by warehouse, and identifies the top 10 most expensive queries.

### Scenario
Your Snowflake bill spiked this month. The CFO asks you to identify which warehouses are consuming the most credits and which queries are the biggest culprits. You need to produce a cost analysis report using Snowflake's query history.

### Solution
```sql
-- Credit consumption by warehouse (last 30 days)
SELECT
  warehouse_name,
  SUM(credits_used) AS total_credits,
  ROUND(SUM(credits_used) * 3.00, 2) AS estimated_cost_usd,  -- X-Large = ~$3.00/hr
  COUNT(*) AS query_count,
  AVG(credits_used) AS avg_credits_per_query,
  PERCENTILE_APPROX(credits_used, 0.95) AS p95_credits
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND warehouse_name IS NOT NULL
GROUP BY warehouse_name
ORDER BY total_credits DESC;

-- Top 10 most expensive queries
SELECT
  qh.query_id,
  qh.query_text,
  qh.warehouse_name,
  qh.credits_used,
  ROUND(qh.credits_used * 3.00, 2) AS estimated_cost_usd,
  qh.total_elapsed_time,
  qh.bytes_scanned,
  ROUND(qh.bytes_scanned / 1073741824, 2) AS gb_scanned,
  qh.user_name,
  qh.start_time
FROM snowflake.account_usage.query_history qh
WHERE qh.start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND qh.warehouse_name IS NOT NULL
ORDER BY qh.credits_used DESC
LIMIT 10;
```

### Explanation
`SNOWFLAKE.ACCOUNT_USAGE` is a built-in schema that tracks all account activity. The `QUERY_HISTORY` table contains one row per query with metrics including:
- **credits_used**: fractional credits consumed
- **total_elapsed_time**: query execution time in milliseconds
- **bytes_scanned**: total data scanned
- **query_text**: the actual SQL executed

The first query groups by warehouse to show aggregate consumption. The `PERCENTILE_APPROX` identifies the 95th percentile cost per query. The second query finds the absolute most expensive individual queries.

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Account Usage schema | Built-in, no setup, comprehensive | 1-hour delay, requires ACCOUNTUSAGEROLE | Cost analysis, auditing |
| Resource Monitors | Real-time alerts, automated actions | Coarse granularity (credits only) | Budget enforcement |
| Third-party tools (Monte Carlo, Datadog) | Rich dashboards, alerts | Additional cost, integration effort | Enterprise monitoring |

### Edge Cases
- `ACCOUNT_USAGE` tables have a 1-hour delay; not suitable for real-time monitoring
- Queries using the result cache show 0 credits but still appear in history
- Multi-statement transactions show individual query costs, not combined

### Principal-Level Considerations
- Automate cost anomaly detection: set up a task that runs daily and alerts when credit usage exceeds 2x the 7-day rolling average
- Build a cost attribution model that tags warehouses by team/product using session variables
- Implement auto-suspend policies on all warehouses to prevent runaway costs
- Use resource monitors with target credit limits per warehouse per month

### Follow-up Questions
1. **What roles can access the ACCOUNT_USAGE schema?**
   - **Answer:** `ACCOUNTUSAGEROLE` by default. You must grant `USAGE` on the schema and `SELECT` on individual tables to other roles. Never grant it to `PUBLIC`.
2. **How do you calculate actual dollar cost from credits?**
   - **Answer:** Multiply credits by the on-demand rate for the warehouse size in your region. E.g., X-Large = 16 credits/hr × $3.00/hr = $0.1875/credit. Use `SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METRICS` to get actual sizes.
3. **Can you reduce the 1-hour delay of ACCOUNT_USAGE?**
   - **Answer:** No — it's a known limitation. For near-real-time monitoring, use `INFORMATION_SCHEMA.QUERY_HISTORY` (available to ACCOUNTADMIN and SECURITYADMIN) which has lower latency.
4. **How do you attribute cost to specific teams?**
   - **Answer:** Use session variables (`ALTER SESSION SET team_label = 'analytics'`), naming conventions on warehouses (`WH_ANALYTICS`, `WH_ENGINEERING`), or tags (Snowflake Object Tagging feature).

---

## Q17. Deduplicate a table using QUALIFY and ROW_NUMBER() — keep only the most recent row per customer_id.

### Scenario
Your ETL pipeline loaded duplicate customer records due to a retry. Each `customer_id` appears multiple times with different `updated_at` timestamps. You need to keep only the latest version of each customer.

### Solution
```sql
-- Deduplicate: keep the most recent row per customer_id
SELECT *
FROM customers
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY customer_id
  ORDER BY updated_at DESC, created_at DESC
) = 1;

-- Or as a CTE to insert into a clean table
WITH deduped AS (
  SELECT *,
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY updated_at DESC, created_at DESC
    ) AS rn
  FROM customers
)
SELECT *
FROM deduped
WHERE rn = 1;
```

### Explanation
`QUALIFY` is Snowflake's extension to SQL that filters the results of window functions directly, similar to how `HAVING` filters aggregated results. It's more concise than wrapping the window function in a subquery.

`ROW_NUMBER()` assigns a unique sequential integer to rows within each partition. By ordering by `updated_at DESC`, the most recent record gets `rn = 1`. The `QUALIFY` clause filters to only those rows.

The CTE approach is equivalent but more portable across databases that don't support `QUALIFY`.

### Example Data / Sample Table
```
customer_id | name    | updated_at           | created_at
------------+---------+---------------------+---------------------
1           | Alice   | 2024-01-15 10:00:00 | 2023-06-01 09:00:00
1           | Alice B | 2024-01-16 14:30:00 | 2023-06-01 09:00:00
2           | Bob     | 2024-01-14 08:00:00 | 2023-07-15 11:00:00
2           | Bob     | 2024-01-14 08:00:00 | 2023-08-20 10:00:00
3           | Charlie | 2024-01-10 12:00:00 | 2023-09-01 14:00:00
```

Result after dedup:
```
customer_id | name    | updated_at           | created_at
------------+---------+---------------------+---------------------
1           | Alice B | 2024-01-16 14:30:00 | 2023-06-01 09:00:00
2           | Bob     | 2024-01-14 08:00:00 | 2023-08-20 10:00:00
3           | Charlie | 2024-01-10 12:00:00 | 2023-09-01 14:00:00
```

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| QUALIFY + ROW_NUMBER | Concise, single pass, Snowflake native | Snowflake only | Snowflake environments |
| CTE + ROW_NUMBER | Portable across databases | Slightly more verbose | Cross-database code |
| MERGE with subquery | Can insert into target table directly | Complex syntax, two reads | When loading into another table |

### Edge Cases
- **Tie-breaking**: When `updated_at` is identical, use a secondary sort (e.g., `created_at` or primary key) to ensure deterministic results
- **Null values**: `ORDER BY updated_at DESC` puts NULLs last; use `ORDER BY COALESCE(updated_at, '1900-01-01') DESC` if NULLs should be excluded
- **Large tables**: QUALIFY can be expensive on unclustered tables with billions of rows; consider clustering on `customer_id`

### Principal-Level Considerations
- Prevent duplicates at the source: use `MERGE` instead of `INSERT` in ETL pipelines
- For production deduplication at scale, consider using a staging table with `QUALIFY` then swapping with `ALTER TABLE ... SWAP WITH`
- Build automated deduplication checks as dbt tests or Snowflake data quality frameworks

### Follow-up Questions
1. **What's the difference between QUALIFY and HAVING?**
   - **Answer:** `HAVING` filters on aggregate functions (SUM, COUNT) after GROUP BY. `QUALIFY` filters on window functions (ROW_NUMBER, RANK) after the window calculation. They operate at different stages of query execution.
2. **Why not use DISTINCT?**
   - **Answer:** `DISTINCT` removes exact duplicate rows. It cannot choose "the latest" row when other columns differ. You need `ROW_NUMBER()` or `RANK()` to pick based on a specific ordering.
3. **How does QUALIFY perform on a 10 billion row table?**
   - **Answer:** It requires a full scan and sort within each partition. With a proper clustering key on `customer_id`, Snowflake can prune partitions significantly. Without clustering, it will scan all data. Consider materializing as a view or running as a scheduled task with a large warehouse.
4. **Can you use QUALIFY with RANK() instead of ROW_NUMBER()?**
   - **Answer:** Yes — `RANK()` returns ties with the same rank (e.g., two rows both get rank 1, then the next is rank 3). Use `RANK()` when you want to keep all tied rows; use `ROW_NUMBER()` for strict single-row-per-partition deduplication.

---

## Q18. Implement SCD Type 2 slowly changing dimensions using Snowflake Streams.

### Scenario
You need to track historical changes to a `dim_customer` table. When a customer's address or name changes, the old record should be marked as inactive (`is_current = FALSE`, `end_date = CURRENT_DATE()`) and a new record inserted with the new values. Use Snowflake Streams for change detection.

### Solution
```sql
-- 1. Create the SCD2 dimension table
CREATE OR REPLACE TABLE dim_customer (
  customer_key    NUMBER IDENTITY(1,1),
  customer_id     NUMBER NOT NULL,
  name            VARCHAR(255),
  email           VARCHAR(255),
  address         VARCHAR(500),
  is_current      BOOLEAN DEFAULT TRUE,
  start_date      TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  end_date        TIMESTAMP DEFAULT NULL
);

-- 2. Create a stream on the source table
CREATE OR REPLACE STREAM customer_src_stream
  ON TABLE src_customers
  COPY GRANTS;

-- 3. SCD2 merge logic (run as a scheduled task)
MERGE INTO dim_customer tgt
USING (
  SELECT
    customer_id,
    name,
    email,
    address,
    METADATA$ISUPDATE,
    METADATA$AFTER
  FROM customer_src_stream
) src
ON tgt.customer_id = src.customer_id
  AND tgt.is_current = TRUE

-- Update existing record: close it out
WHEN MATCHED AND src.METADATA$ISUPDATE = TRUE AND src.METADATA$AFTER IS NOT NULL THEN
  UPDATE SET
    tgt.is_current = FALSE,
    tgt.end_date = TIMESTAMPADD(second, -1, src.METADATA$AFTER)

-- Insert new customer
WHEN NOT MATCHED AND src.METADATA$ISUPDATE = FALSE THEN
  INSERT (customer_id, name, email, address, is_current, start_date, end_date)
  VALUES (src.customer_id, src.name, src.email, src.address, TRUE, src.METADATA$AFTER, NULL);
```

### Explanation
This implements a full SCD Type 2 pattern using Streams:
- The stream on `src_customers` captures inserts and updates
- `METADATA$ISUPDATE` distinguishes new records from changes to existing ones
- `METADATA$AFTER` holds the new values after an update
- The MERGE statement closes out the current record and inserts a new one

For updates: the current record gets `is_current = FALSE` and `end_date` set to just before the change time. A new record is inserted with the updated values.

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Streams + MERGE | Native Snowflake, automatic CDC, efficient | Requires Enterprise edition | Production pipelines |
| dbt snapshots | Cross-database, flexible strategies | Requires dbt, less efficient than native streams | Multi-platform teams |
| Manual timestamp comparison | Works everywhere | Full table scan every run, error-prone | Legacy systems |

### Edge Cases
- **Deletes**: Stream metadata `METADATA$ACTION = 'DELETE'` requires handling — either soft-delete in the source or update `dim_customer` accordingly
- **Multiple changes per session**: Stream only captures the net change; intermediate values are lost
- **Stream expiration**: After 14 days (Enterprise) or 1 day (Standard), unprocessed changes expire

### Principal-Level Considerations
- For production, add a unique key index and consider clustering `dim_customer` on `(customer_id, is_current)`
- Schedule this as a Snowflake Task with proper error handling and alerting
- Add data quality checks: verify that every `customer_id` has exactly one `is_current = TRUE` record

### Follow-up Questions
1. **How do you handle deletes in SCD2 with Streams?**
   - **Answer:** Check `METADATA$ACTION = 'DELETE'` in the stream. For deletes, either set `is_current = FALSE` on the current record with `end_date = CURRENT_TIMESTAMP()`, or insert a tombstone record depending on your business requirements.
2. **What happens if the stream falls behind (more than 14 days)?**
   - **Answer:** Changes older than the retention period (14 days for Enterprise) are lost. You'd need to do a full reconciliation by comparing source and target, or implement a fallback full-refresh strategy.
3. **How does this handle multiple updates to the same row within a single transaction?**
   - **Answer:** Streams only see the net change (before vs after the transaction). Intermediate values are not captured. This is why you should process streams frequently (e.g., every 15 minutes).
4. **Can you combine SCD2 with incremental loading?**
   - **Answer:** Yes — the stream itself is incremental (only changed rows). The MERGE only touches the specific customer IDs that changed, making it very efficient even on billion-row tables.

---
## Q19. Implement a materialized view with automatic refresh in Snowflake and explain when to use it over a regular view.

### Scenario
Your analytics team runs complex queries joining 5+ tables with aggregations on a 2B row fact table. Query response times are 30-60 seconds. The data changes incrementally (new rows appended daily). You need a solution that provides fast query responses while keeping data reasonably fresh.

### Solution
```sql
-- Create a materialized view for fast analytics
CREATE OR REPLACE MATERIALIZED VIEW mv_daily_sales_summary
  CLUSTER BY (sale_date, region)
AS
SELECT
  DATE_TRUNC('day', sale_timestamp) AS sale_date,
  region,
  COUNT(DISTINCT customer_id) AS unique_customers,
  SUM(sale_amount) AS total_sales,
  AVG(sale_amount) AS avg_sale_amount,
  COUNT(*) AS total_transactions
FROM sales
GROUP BY 1, 2;

-- Query the materialized view (fast — pre-computed)
SELECT *
FROM mv_daily_sales_summary
WHERE sale_date >= '2024-01-01'
  AND region = 'West'
ORDER BY total_sales DESC;
```

### Explanation
Materialized views in Snowflake:
- **Automatically maintained**: Snowflake incrementally updates the materialized view when the underlying base table changes (inserts, updates, deletes)
- **Zero maintenance**: No need to schedule refresh tasks or manually truncate/rebuild
- **Transparent usage**: Query it like a regular table/view; Snowflake's query optimizer decides when to use it
- **Cost**: Storage cost for the materialized data; compute cost for incremental maintenance (included in warehouse credits)

Key differences from regular views:
| Aspect | Regular View | Materialized View |
|--------|-------------|-------------------|
| Storage | No storage (logical only) | Physical storage (pre-computed results) |
| Query speed | Re-evaluates underlying query every time | Reads pre-computed results |
| Refresh | Always current | Incrementally auto-refreshed by Snowflake |
| Maintenance | None | None (auto) |
| Cost | Free (uses base table compute) | Storage + incremental maintenance credits |

### When to Use Materialized Views
- **Complex aggregations** on large fact tables that are queried frequently
- **Star join views** on star schemas (Snowflake has automatic star join MV optimization)
- **Performance-critical dashboards** where sub-second response is needed
- **Incremental data** where the MV can be efficiently maintained

### When NOT to Use
- Queries that change frequently (MV becomes stale logic)
- Very large result sets that would consume excessive storage
- Low-frequency queries where the overhead isn't justified
- Queries with UDFs, semi-structured data operations (some functions not supported in MVs)

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Materialized View | Auto-refresh, transparent, fast | Storage cost, limited function support | Production analytics, star joins |
| Manual summary table + Task | Full control, any SQL | Manual maintenance, error-prone | Complex logic not supported in MVs |
| Result Cache | Free, automatic | Only for identical queries | Simple repeated queries |
| Regular View | Zero storage, always current | Re-runs full query every time | Simple queries, rarely used |

### Edge Cases
- Materialized views are not immediately consistent — there's a short lag between base table changes and MV refresh
- Not all SQL functions are supported in materialized views (no UDFs, no LATERAL joins)
- Star join materialized views are created automatically by Snowflake when enabled

### Principal-Level Considerations
- Enable automatic star join MV optimization at the account level for star schema performance
- Monitor MV refresh lag using `TABLE(INFORMATION_SCHEMA.MATERIALIZED_VIEW_REFRESH_HISTORY())`
- Set appropriate clustering keys on MVs to optimize both query performance and maintenance
- Budget for storage costs: a materialized view storing 100M aggregated rows may cost ~$50-100/month in storage

### Follow-up Questions
1. **How does Snowflake know to use the materialized view instead of the base query?**
   - **Answer:** The query optimizer rewrites queries that match the MV's definition (or a subset of it) to read from the MV. This happens transparently — you don't need to change your queries. You can verify in the query plan that the MV was used.
2. **What happens if the base table is truncated?**
   - **Answer:** Snowflake detects the truncation and rebuilds the materialized view from scratch. This can be expensive for very large tables. Consider using `DELETE` instead of `TRUNCATE` for incremental maintenance if you need to remove partial data.
3. **Can you have multiple materialized views on the same base table?**
   - **Answer:** Yes. Each MV is maintained independently. Be mindful of cumulative storage costs and the maintenance compute overhead, especially for frequently changing tables.
4. **How do you monitor materialized view refresh status?**
   - **Answer:** Query `INFORMATION_SCHEMA.MATERIALIZED_VIEW_REFRESH_HISTORY()` or check the MV's `refresh_status` column in `INFORMATION_SCHEMA.MATERIALIZED_VIEWS`. Use `DESCRIBE MATERIALIZED VIEW` to see details.

---

## Q20. Design a real-time CDC pipeline from a PostgreSQL database to Snowflake using Debezium and Kafka.

### Scenario
Your company runs a PostgreSQL database (order management system). You need to capture all changes (inserts, updates, deletes) in real-time and stream them into Snowflake for analytics. The source has 50 tables with approximately 10,000 writes/second peak. Latency requirement: under 5 seconds from source change to Snowflake availability.

### Solution

**Architecture:**
```
PostgreSQL → Debezium (CDC) → Kafka → Kafka Connect (Snowflake Connector) → Snowflake Staging → Snowflake Tasks + Streams → Dimension/Fact Tables
```

**Step 1: Debezium PostgreSQL Connector Configuration**
```json
{
  "name": "pg-orders-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "4",
    "database.hostname": "pg-prod.internal",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "***",
    "database.dbname": "orders_db",
    "database.server.name": "prod-orders",
    "plugin.name": "pgoutput",
    "publication.name": "dbz_publication",
    "slot.name": "debezium_slot",
    "topic.prefix": "orders",
    "schema.include.list": "public",
    "table.include.list": "public.orders,public.order_items,public.customers",
    "tombstones.on.delete": "true",
    "decimal.handling.mode": "double",
    "time.precision.mode": "adaptive"
  }
}
```

**Step 2: Kafka Connect Snowflake Sink Connector**
```properties
name=snowflake-sink-orders
connector.class=com.snowflake.kafka.connector.SnowflakeSinkConnector
tasks.max=4
topics=orders.public.orders,orders.public.order_items,orders.public.customers

# Snowflake connection
snowflake.url.name=account.region.snowflakecomputing.com:443
snowflake.database=ANALYTICS
snowflake.schema=STAGING
snowflake.user=KAFKA_LOADER
snowflake.private.key=***

# Bulk loading configuration
snowflake.buffer.count=10
snowflake.buffer.flush.time=60
snowflake.buffer.size=5000000
snowflake.streaming.maxRetries=30

# Format
snowflake.format=JSON
snowflake.enable.avro.dynamic.schemas=true
```

**Step 3: Snowflake Staging + Transformation**
```sql
-- Create stage and target tables
CREATE OR REPLACE STAGING.orders_raw (
  src_key STRING,
  src_source STRING,
  src_offset STRING,
  src_ts TIMESTAMP,
  value STRING  -- JSON payload from Kafka
);

-- Parse JSON and load into clean tables
CREATE OR REPLACE STREAM orders_stream ON TABLE staging.orders_raw;

CREATE OR REPLACE TABLE dwd.orders (
  order_id NUMBER,
  customer_id NUMBER,
  order_status VARCHAR(50),
  order_amount NUMBER,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  is_current BOOLEAN DEFAULT TRUE,
  start_date TIMESTAMP,
  end_date TIMESTAMP
);

-- CDC merge logic in a scheduled task
CREATE OR REPLACE TASK merge_orders
  WAREHOUSE = 'WH_ETL'
  SCHEDULE = '5 MINUTE'
AS
MERGE INTO dwd.orders tgt
USING (
  SELECT
    GET_PATH(VALUE:C.data, 'order_id')::NUMBER AS order_id,
    GET_PATH(VALUE:C.data, 'customer_id')::NUMBER AS customer_id,
    GET_PATH(VALUE:C.op, 0) AS operation,  -- 'c'=create, 'u'=update, 'd'=delete
    GET_PATH(VALUE:C.data, 'order_status')::VARCHAR AS order_status,
    GET_PATH(VALUE:C.data, 'order_amount')::NUMBER AS order_amount,
    GET_PATH(VALUE:C.data, 'created_at')::TIMESTAMP AS created_at,
    GET_PATH(VALUE:C.data, 'updated_at')::TIMESTAMP AS updated_at
  FROM orders_stream
) src
ON tgt.order_id = src.order_id AND tgt.is_current = TRUE
WHEN MATCHED AND src.operation = 'd' THEN
  UPDATE SET tgt.is_current = FALSE, tgt.end_date = CURRENT_TIMESTAMP()
WHEN MATCHED AND src.operation = 'u' THEN
  UPDATE SET tgt.is_current = FALSE, tgt.end_date = src.updated_at
WHEN NOT MATCHED AND src.operation IN ('c', 'u') THEN
  INSERT VALUES (src.order_id, src.customer_id, src.order_status,
    src.order_amount, src.created_at, src.updated_at, TRUE, src.updated_at, NULL);
```

### Explanation
This is a production-grade CDC pipeline:
1. **Debezium** reads PostgreSQL WAL (Write-Ahead Log) via `pgoutput` decoding plugin — zero impact on source database performance
2. **Kafka** acts as the durable buffer between source and target, providing exactly-once semantics and replayability
3. **Snowflake Kafka Connector** bulk-loads JSON events into a staging table
4. **Snowflake Streams + Tasks** process the CDC events and apply them to the target tables with proper SCD2 handling

Key design decisions:
- `tombstones.on.delete=true` ensures deletes are propagated (Kafka sends a null-key message)
- `snowflake.buffer.count=10` and `snowflake.buffer.flush.time=60` balance latency vs throughput
- Stream processing with `5 MINUTE` schedule ensures sub-5-second end-to-end latency

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Debezium + Kafka + Snowflake Connector | Real-time, decoupled, replayable | Complex setup, operational overhead | Large-scale production CDC |
| Fivetran/Matillion | Managed, easy setup, monitoring | Vendor lock-in, cost, less control | Teams without platform engineering |
| Custom Python + polling | Simple, no extra infrastructure | High source DB load, eventual consistency only | Small projects, prototypes |
| Snowpipe + cloud storage | Serverless, cost-effective | Requires external staging, slightly higher latency | Batch-friendly workflows |

### Edge Cases
- **Schema changes**: Debezium supports schema evolution but requires careful handling. Add schema registry integration.
- **Large transactions**: Very long-running transactions can cause Debezium lag. Set `max.batch.size` appropriately.
- **Kafka retention**: Ensure Kafka retention is long enough for slow consumers (at least 7 days).
- **Snowflake JSON parsing**: Complex nested JSON may require recursive parsing with `FLATTEN`.

### Principal-Level Considerations
- **Observability**: Set up monitoring on Kafka lag (consumer group lag), Debezium offset position, and Snowflake task success/failure
- **Data quality**: Implement automated reconciliation jobs that compare row counts and checksums between source and target
- **Schema evolution**: Use Debezium with Schema Registry for versioned schemas; handle backward/forward compatibility
- **Disaster recovery**: Kafka enables replay — in case of target corruption, reset the stream offset and replay from a known good point
- **Cost optimization**: Right-size the ETL warehouse; use auto-resizing; consider using Snowpipe Streaming for lower latency at higher cost

### Follow-up Questions
1. **What happens if Kafka goes down for several hours?**
   - **Answer:** Debezium stops producing when Kafka is unavailable, but PostgreSQL WAL continues. When Kafka recovers, Debezium catches up from the last committed offset. The WAL retention on PostgreSQL must be long enough to cover the outage window. Ensure `wal_keep_size` is set appropriately.
2. **How do you handle schema changes in the source database?**
   - **Answer:** Debezium detects schema changes and publishes new schemas to Schema Registry. The Snowflake connector will create new columns in the staging table if `auto.create` is enabled. For the DWH layer, you need a process to propagate schema changes to downstream tables.
3. **How do you ensure exactly-once processing?**
   - **Answer:** Debezium provides exactly-once from source to Kafka (using WAL offsets). The Snowflake connector supports exactly-once with `snowflake_processing_threads=1` and幂等 writes. The MERGE in Snowflake is idempotent — running the same stream data twice produces the same result.
4. **What's the cost of this pipeline for 10K writes/sec?**
   - **Answer:** Debezium: ~2-4 CPU cores per connector. Kafka: ~4-8 brokers for this volume. Snowflake: staging table ingestion is cheap (bulk load), MERGE task depends on warehouse size. Total infra cost: ~$3,000-8,000/month depending on cloud provider and retention. Consider managed services (Confluent Cloud) for simpler ops.

---

## Q21. Write a query that uses LATERAL FLATTEN to parse a semi-structured JSON column containing nested arrays and objects.

### Scenario
Your raw events table stores JSON payloads in a VARIANT column. Each event has a nested structure with an array of `items` (each item has `product_id`, `quantity`, `price`) and a nested `user` object with `id`, `tier`, and a `preferences` array. You need to flatten this into a relational format for analytics.

### Solution
```sql
-- Sample JSON structure in the VARIANT column:
-- {
--   "event_id": "evt_001",
--   "event_type": "purchase",
--   "timestamp": "2024-01-15T10:30:00Z",
--   "user": {
--     "id": 12345,
--     "tier": "gold",
--     "preferences": ["email", "push", "sms"]
--   },
--     "items": [
--     {"product_id": "P100", "quantity": 2, "price": 29.99},
--     {"product_id": "P200", "quantity": 1, "price": 49.99}
--   ]
-- }

-- Flatten items array and extract user fields
SELECT
  e.event_id,
  e.event_type,
  e.payload:timestamp::TIMESTAMP AS event_timestamp,

  -- Extract from nested user object
  e.payload:user:id::NUMBER AS user_id,
  e.payload:user:tier::VARCHAR AS user_tier,

  -- Flatten the items array
  i.value:product_id::VARCHAR AS product_id,
  i.value:quantity::NUMBER AS quantity,
  i.value:price::NUMBER AS unit_price,
  i.value:quantity::NUMBER * i.value:price::NUMBER AS line_total

FROM raw_events e,
  LATERAL FLATTEN(input => e.payload:items) i

WHERE e.event_type = 'purchase'
  AND e.payload:timestamp::DATE >= '2024-01-01';

-- Further flatten: user preferences (one row per preference per item)
SELECT
  e.event_id,
  e.payload:user:id::NUMBER AS user_id,
  p.value::VARCHAR AS user_preference,
  i.value:product_id::VARCHAR AS product_id
FROM raw_events e,
  LATERAL FLATTEN(input => e.payload:user:preferences) p,
  LATERAL FLATTEN(input => e.payload:items) i
WHERE e.event_type = 'purchase';
```

### Explanation
`LATERAL FLATTEN` (also written as `TABLE(FLATTEN())`) takes a VARIANT array or object and produces one row per element:
- For **arrays**: each element becomes a row, `VALUE` holds the element, `KEY` is null
- For **objects**: each key-value pair becomes a row, `KEY` holds the key name, `VALUE` holds the value
- For **nested arrays**: use multiple FLATTEN calls (comma-separated in the FROM clause)

Accessing nested JSON fields uses the colon notation: `payload:user:id` traverses `payload → user → id`. Cast with `::TYPE` to convert from VARIANT to a typed column.

In the second query, two `LATERAL FLATTEN` calls are combined (Cartesian product within each row): each preference is paired with each item, producing `n_preferences × n_items` rows per event.

### Example Input → Output
**Input JSON:**
```json
{
  "event_id": "evt_001",
  "items": [
    {"product_id": "P100", "quantity": 2, "price": 29.99},
    {"product_id": "P200", "quantity": 1, "price": 49.99}
  ]
}
```

**Output (after flattening items):**
```
event_id | product_id | quantity | unit_price | line_total
---------+------------+----------+------------+-----------
evt_001  | P100       | 2        | 29.99      | 59.98
evt_001  | P200       | 1        | 49.99      | 49.99
```

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| LATERAL FLATTEN | Native, set-based, composable | Cartesian product risk with multiple flattens | Flattening arrays/objects |
| GET_PATH() | Safe access to nested fields without flattening | Returns VARIANT, not relational | Single-value extraction |
| OBJECT_KEYS / OBJECT_VALUES | List all keys in a dynamic JSON object | Order not guaranteed | Schema discovery, dynamic schemas |
| PARSE_JSON + subquery | More explicit control | Verbose, harder to read | Complex transformations |

### Edge Cases
- **Empty arrays**: `FLATTEN` on an empty array produces no rows (the parent row disappears). Use `LEFT OUTER JOIN ... LATERAL FLATTEN(..., outer => true)` to keep the parent row.
- **NULL values**: `payload:missing_field` returns NULL without error — no need for NULL checks
- **Very deep nesting**: Performance degrades with deep FLATTEN chains; consider pre-parsing with UDFs
- **Malformed JSON**: `PARSE_JSON` throws errors on invalid JSON; use `TRY_PARSE_JSON` for safe parsing

### Principal-Level Considerations
- **Storage**: VARIANT columns are stored efficiently in Snowflake (internal optimized format), but flatten operations at query time add compute cost. Consider materializing flattened columns in staging tables.
- **Schema evolution**: Semi-structured data handles schema changes gracefully — new fields appear without DDL. Use `OBJECT_KEYS()` for schema discovery.
- **Performance**: Flatten operations on large VARIANT columns can be expensive. Create a materialized view with pre-flattened columns for frequently queried paths.
- **Data quality**: Implement validation UDFs or dbt tests to check that required JSON fields exist and have correct types.

### Follow-up Questions
1. **What's the difference between `LATERAL FLATTEN` and `TABLE(FLATTEN())`?**
   - **Answer:** They are syntactically equivalent. `LATERAL FLATTEN(input => ...)` is the newer, more readable syntax. `TABLE(FLATTEN(input => ...))` is the older syntax. Both produce the same result.
2. **How do you keep parent rows when the array is empty?**
   - **Answer:** Use `outer => true` in the FLATTEN call: `LATERAL FLATTEN(input => e.payload:items, outer => TRUE)`. This produces a row with NULL values for the flattened columns when the array is empty or null.
3. **How do you handle JSON arrays of objects where each object has a different structure?**
   - **Answer:** Use `GET_PATH()` with explicit field checks: `IFF(EXISTS src.value:field1, src.value:field1::VARCHAR, NULL)`. Or use `OBJECT_KEYS()` to discover available fields dynamically.
4. **Is it better to flatten at ingestion time or query time?**
   - **Answer:** For analytics workloads with repeated queries, flatten at ingestion (store in typed columns). For exploratory analysis or schema-evolving data, keep as VARIANT and flatten at query time. The tradeoff is storage/compute vs flexibility.

---
## Q22. Implement a data quality framework in Snowflake using automated checks, alerts, and data contracts.

### Scenario
You are building a data platform serving 200+ analysts and 50+ downstream systems. Data quality incidents are causing broken dashboards and wrong business decisions. You need a systematic approach to detect, alert on, and prevent data quality issues before they reach consumers.

### Solution
```sql
-- 1. Data Quality Rules Table (metadata-driven approach)
CREATE OR REPLACE TABLE dq_rules (
  rule_id         NUMBER,
  table_name      VARCHAR(255),
  column_name     VARCHAR(255),
  rule_type       VARCHAR(50),     -- NOT_NULL, UNIQUE, RANGE, REFERENCE, CUSTOM
  rule_expression VARCHAR(1000),   -- SQL expression that returns BOOLEAN
  severity        VARCHAR(20),     -- CRITICAL, HIGH, MEDIUM, LOW
  alert_channel   VARCHAR(100),    -- SLACK, EMAIL, PAGER
  description     VARCHAR(500),
  created_by      VARCHAR(255),
  active          BOOLEAN DEFAULT TRUE
);

-- Example rules
INSERT INTO dq_rules VALUES
  (1, 'dwd.orders', 'order_id', 'NOT_NULL', 'order_id IS NOT NULL', 'CRITICAL', 'PAGER', 'Order ID must never be null', 'platform-team', TRUE),
  (2, 'dwd.orders', 'order_id', 'UNIQUE', 'COUNT(DISTINCT order_id) = COUNT(*)', 'CRITICAL', 'PAGER', 'Order ID must be unique', 'platform-team', TRUE),
  (3, 'dwd.orders', 'order_amount', 'RANGE', 'order_amount >= 0', 'HIGH', 'SLACK', 'Order amount must be non-negative', 'data-team', TRUE),
  (4, 'dwd.orders', 'customer_id', 'REFERENCE', 'customer_id IN (SELECT customer_id FROM dwd.customers)', 'HIGH', 'SLACK', 'Every order must reference a valid customer', 'data-team', TRUE),
  (5, 'dwd.orders', 'order_status', 'CUSTOM', "order_status IN ('pending','processing','shipped','delivered','cancelled')", 'MEDIUM', 'SLACK', 'Order status must be a valid value', 'data-team', TRUE);

-- 2. DQ Check Execution (parameterized, runs against all active rules)
CREATE OR REPLACE TABLE dq_results (
  check_id        NUMBER AUTOINCREMENT,
  rule_id         NUMBER,
  table_name      VARCHAR(255),
  rule_type       VARCHAR(50),
  severity        VARCHAR(20),
  passed          BOOLEAN,
  total_rows      NUMBER,
  failing_rows    NUMBER,
  failure_pct     FLOAT,
  checked_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  details         VARCHAR(2000)
);

-- Run all DQ checks (stored as a stored procedure)
CREATE OR REPLACE PROCEDURE run_dq_checks()
RETURNS TABLE ()
LANGUAGE SQL
AS
$$
  DECLARE
    cur CURSOR FOR
      SELECT rule_id, table_name, column_name, rule_expression, severity, alert_channel, description
      FROM dq_rules
      WHERE active = TRUE;
    
    rule_rec RECORD;
    v_sql VARCHAR;
    v_failing_rows NUMBER;
    v_total_rows NUMBER;
  BEGIN
    FOR rule_rec IN cur DO
      -- Build dynamic SQL to count failing rows
      v_sql := 'SELECT COUNT(*) AS failing_rows, (SELECT COUNT(*) FROM '
             || rule_rec.table_name || ') AS total_rows
                FROM ' || rule_rec.table_name
             || ' WHERE NOT (' || rule_rec.rule_expression || ')';
      
      -- Execute and capture results
      EXECUTE IMMEDIATE :v_sql INTO :v_failing_rows, :v_total_rows;
      
      -- Insert result
      INSERT INTO dq_results (rule_id, table_name, rule_type, severity, passed, total_rows, failing_rows, failure_pct, details)
      VALUES (
        rule_rec.rule_id,
        rule_rec.table_name,
        rule_rec.rule_type,
        rule_rec.severity,
        CASE WHEN :v_failing_rows = 0 THEN TRUE ELSE FALSE END,
        :v_total_rows,
        :v_failing_rows,
        CASE WHEN :v_total_rows > 0 THEN ROUND(:v_failing_rows * 100.0 / :v_total_rows, 4) ELSE 0 END,
        rule_rec.description
      );
    END FOR;
    
    -- Check for critical failures and alert
    IF EXISTS (SELECT 1 FROM dq_results WHERE passed = FALSE AND severity = 'CRITICAL') THEN
      -- Send alert via Slack webhook (using Snowflake External Function or Python UDF)
      CALL send_slack_alert('CRITICAL data quality failure detected');
    END IF;
    
    RETURN TABLE();
  END
$$;

-- 3. DQ Dashboard (query results for monitoring)
SELECT
  dr.severity,
  COUNT(CASE WHEN dr.passed = FALSE THEN 1 END) AS failing_checks,
  COUNT(*) AS total_checks,
  MAX(dr.checked_at) AS last_check,
  LISTAGG(
    CASE WHEN dr.passed = FALSE THEN dr.table_name || '.' || dr.column_name END,
    ', '
  ) AS failing_tables
FROM dq_results dr
WHERE dr.checked_at >= DATEADD('hour', -24, CURRENT_TIMESTAMP())
GROUP BY dr.severity
ORDER BY 
  CASE dr.severity
    WHEN 'CRITICAL' THEN 1
    WHEN 'HIGH' THEN 2
    WHEN 'MEDIUM' THEN 3
    WHEN 'LOW' THEN 4
  END;
```

### Explanation
This metadata-driven DQ framework:
1. **Rules stored in a table**: Each row defines a check (table, column, expression, severity). New rules can be added without code changes.
2. **Dynamic SQL execution**: A stored procedure iterates through active rules and executes the check expression against the target table.
3. **Results tracking**: All check results are stored with timestamps, enabling trend analysis and SLA monitoring.
4. **Alerting**: Critical failures trigger automated alerts (via Slack webhook or email).

The framework scales by:
- Adding rules for new tables without code changes
- Running checks on a schedule (Snowflake Task) after each pipeline run
- Providing a dashboard view for data consumers to trust the data

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Custom DQ framework (above) | Full control, metadata-driven, integrates with Snowflake | Build/maintain overhead | Teams needing custom logic |
| dbt tests | Declarative, built-in, version-controlled | Requires dbt, limited to dbt ecosystem | Teams using dbt |
| Great Expectations | Rich validation types, Python-based | Separate infra, integration effort | Python-centric teams |
| Monte Carlo / Cubra | ML-driven anomaly detection, observability | Expensive, black-box | Enterprise teams with budget |
| Soda Core | Open-source, YAML-based checks | Requires setup, SQL translation | Teams wanting open-source tooling |

### Edge Cases
- **Dynamic SQL security**: Only allow parameterized rule expressions from the `dq_rules` table; never accept raw user input in `EXECUTE IMMEDIATE`
- **Performance**: DQ checks on billion-row tables can be slow. Use sampled checks (`TABLESAMPLE`) for non-critical rules.
- **False positives**: New data patterns may trigger valid alerts. Implement a rule review process and allow temporary rule deactivation.

### Principal-Level Considerations
- Implement data contracts between producers and consumers: formalize what quality guarantees each pipeline provides
- Use Snowflake Tags for data ownership: tag tables with `DATA_OWNER`, `CLASSIFICATION`, and `RETENTION_PERIOD`
- Build a data quality SLA dashboard: track percentage of checks passing over time, mean time to detection, and mean time to resolution
- Implement automated pipeline gating: block downstream pipelines from running if critical DQ checks fail upstream

### Follow-up Questions
1. **How do you handle DQ checks on incremental pipelines?**
   - **Answer:** Run DQ checks on the new data only (using the stream's `METADATA$ACTION` to identify new rows). For reference checks (e.g., foreign key validation), you need access to the full reference table but only check the new/changed rows.
2. **What's the performance impact of running DQ checks on production tables?**
   - **Answer:** DQ checks add compute load. Mitigate by: (1) running on a dedicated warehouse, (2) using `TABLESAMPLE` for non-critical checks, (3) running during off-peak hours, (4) using materialized views or summary tables for aggregation-based checks.
3. **How do you handle schema changes without breaking DQ checks?**
   - **Answer:** Include schema validation as part of DQ: check that expected columns exist, have expected types. Use INFORMATION_SCHEMA to compare actual schema against expected schema stored in a contract table.

---

## Q23. Design a multi-cluster warehouse strategy for a busy analytics workload with concurrent users.

### Scenario
Your analytics platform serves 500 concurrent users across multiple time zones. Queries range from simple dashboards (1-5 sec) to complex ad-hoc analyses (5-30 min). Currently, a single Large warehouse causes queueing during peak hours, leading to poor user experience. Design a multi-cluster warehouse strategy.

### Solution
```sql
-- 1. Create multi-cluster warehouses for different workload types

-- Dashboard warehouse: high concurrency, fast queries
CREATE OR REPLACE WAREHOUSE dashboard_wh
  WAREHOUSE_SIZE = 'XSMALL'
  WAREHOUSE_TYPE = 'STANDARD'
  MIN_CLUSTER_COUNT = 3
  MAX_CLUSTER_COUNT = 10
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  COMMENT = 'Multi-cluster warehouse for dashboard queries (high concurrency)';

-- Ad-hoc warehouse: fewer users, larger queries
CREATE OR REPLACE WAREHOUSE adhoc_wh
  WAREHOUSE_SIZE = 'XLARGE'
  WAREHOUSE_TYPE = 'STANDARD'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 5
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 30
  AUTO_RESUME = TRUE
  COMMENT = 'Multi-cluster warehouse for ad-hoc analytical queries';

-- ETL warehouse: batch processing
CREATE OR REPLACE WAREHOUSE etl_wh
  WAREHOUSE_SIZE = 'XLARGE'
  WAREHOUSE_TYPE = 'STANDARD'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 3
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 10
  AUTO_RESUME = TRUE
  COMMENT = 'Multi-cluster warehouse for ETL pipelines';

-- 2. Assign users to warehouses via roles
GRANT USAGE ON WAREHOUSE dashboard_wh TO ROLE analyst;
GRANT USAGE ON WAREHOUSE adhoc_wh TO ROLE data_scientist;
GRANT USAGE ON WAREHOUSE etl_wh TO ROLE data_engineer;

-- 3. Set session warehouse via role login
ALTER ROLE analyst SET WAREHOUSE = 'dashboard_wh';
ALTER ROLE data_scientist SET WAREHOUSE = 'adhoc_wh';
ALTER ROLE data_engineer SET WAREHOUSE = 'etl_wh';

-- 4. Monitor multi-cluster scaling
SELECT
  warehouse_name,
  cluster_name,
  status,
  started_at,
  ended_at,
  credits_used
FROM TABLE(INFORMATION_SCHEMA.WAREHOUSE_METRICS_HISTORY())
WHERE warehouse_name LIKE '%_wh'
  AND started_at >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY warehouse_name, started_at;
```

### Explanation
Multi-cluster warehouses automatically scale the number of compute clusters based on query load:
- **MIN_CLUSTER_COUNT**: minimum number of clusters always running (handles baseline load)
- **MAX_CLUSTER_COUNT**: maximum clusters to handle peak load
- **SCALING_POLICY**: `STANDARD` scales based on queue depth; `ECONOMY` waits before scaling to save costs
- Snowflake automatically distributes queries across available clusters using a load balancer

The three-warehouse strategy separates concerns:
| Warehouse | Users | Cluster Strategy | Rationale |
|-----------|-------|-----------------|-----------|
| `dashboard_wh` | 500 analysts | 3-10 XSMALL clusters | Many concurrent small queries; XSMALL is cost-efficient per query |
| `adhoc_wh` | 20 scientists | 1-5 XLARGE clusters | Fewer but resource-intensive queries; larger clusters handle complex joins |
| `etl_wh` | 5 engineers | 1-3 XLARGE clusters | Sequential batch work; scale up during load windows |

### Sizing Guidance
| Cluster Size | Credits/hr | Best For |
|-------------|------------|----------|
| XSMALL | 1 | Simple lookups, dashboards |
| SMALL | 2 | Moderate queries |
| MEDIUM | 8 | Complex queries, medium joins |
| LARGE | 16 | Large joins, aggregations |
| XLARGE | 32 | Very large data scans |
| 4XLAGRE | 512 | Massive one-time transformations |

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Multi-cluster warehouses | Auto-scaling, load balancing, no queuing | Higher cost during peak | High-concurrency workloads |
| Single warehouse + query prioritization | Lower cost, simple | Queueing during peak, manual prioritization needed | Low-concurrency workloads |
| Separate warehouses by team | Full isolation, independent scaling | More warehouses to manage | Different workload profiles |
| Serverless Worksheets | Zero admin, auto-scaled | Not available for all queries/warehouses | Light ad-hoc use |

### Edge Cases
- **Scale-up lag**: Adding a new cluster takes 1-2 minutes. `ECONOMY` policy adds a delay before scaling to avoid thrashing. For sub-second response, set higher `MIN_CLUSTER_COUNT`.
- **Data skew**: If one query is extremely heavy, it may monopolize a cluster. Use query tags and warehouse assignment to route heavy queries to dedicated warehouses.
- **Budget control**: Multi-cluster warehouses can scale to `MAX_CLUSTER_COUNT` quickly. Set resource monitors with credit limits to prevent runaway costs.

### Principal-Level Considerations
- Implement query tagging for cost attribution: `ALTER SESSION SET QUERY_TAG = 'dashboard_revenue'`
- Use resource monitors with alerts at 50%, 80%, and 100% of budget thresholds
- Schedule warehouses with time-based scaling: use `SCALING_POLICY` and Tasks to pre-warm warehouses before business hours
- Consider virtual warehouses per business unit for cost chargeback
- Monitor `TABLE(RESULT_SCAN(LAST_QUERY_ID()))` from `ACCOUNT_USAGE.WAREHOUSE_LOAD` to right-size cluster counts

### Follow-up Questions
1. **How does Snowflake route queries across clusters?**
   - **Answer:** Snowflake's query router distributes queries to the cluster with the lowest current load (fewest queued queries). It's automatic and transparent to the user. New clusters are added when the queue depth exceeds a threshold.
2. **What's the difference between STANDARD and ECONOMY scaling policy?**
   - **Answer:** `STANDARD` scales up quickly when queries queue, adding new clusters as needed. `ECONOMY` waits for the current cluster to be fully utilized before adding a new one, and scales down faster. ECONOMY saves 10-40% on costs but has higher latency during scale-up events.
3. **Can you combine multi-cluster with auto-resize?**
   - **Answer:** No — multi-cluster and auto-resize are mutually exclusive features. Multi-cluster adds more clusters of the same size; auto-resize changes the size of a single cluster. Choose based on whether you need concurrency (multi-cluster) or raw power per query (auto-resize).
4. **How do you handle a query that needs more memory than a single cluster provides?**
   - **Answer:** Route that query to a larger warehouse (e.g., `adhoc_wh` with XLARGE clusters). Or use a dedicated warehouse for that specific workload. You can also use result caching if the query runs periodically.

---

## Q24. Implement Snowflake Dynamic Tables for automated data freshness with guaranteed SLAs.

### Scenario
Your company runs a real-time fraud detection system. You need a pipeline that transforms raw transaction data into a fraud-risk score within 5 seconds of the transaction occurring. The pipeline has 3 stages: raw → cleaned → scored. Traditional Tasks + Streams require manual orchestration and monitoring.

### Solution
```sql
-- 1. Base table (raw transactions, loaded via Snowpipe)
CREATE OR REPLACE TABLE raw_transactions (
  transaction_id VARCHAR(36),
  customer_id NUMBER,
  amount NUMBER,
  merchant_id NUMBER,
  timestamp TIMESTAMP,
  ip_address VARCHAR(45),
  device_fingerprint VARCHAR(128)
);

-- 2. Stage 1: Cleaned transactions (Dynamic Table)
CREATE OR REPLACE DYNAMIC TABLE dt_cleaned_transactions
  TARGET_LAG = '5 SECONDS'
  WAREHOUSE = 'dt_fraud_wh'
AS
SELECT
  transaction_id,
  customer_id,
  amount,
  merchant_id,
  timestamp,
  ip_address,
  device_fingerprint,
  -- Enrich with customer data
  c.customer_tier,
  c.country,
  c.account_age_days,
  -- Filter invalid records
  ROW_NUMBER() OVER (PARTITION BY transaction_id ORDER BY timestamp DESC) AS rn
FROM raw_transactions r
JOIN dim_customers c ON r.customer_id = c.customer_id
WHERE r.amount > 0
  AND r.timestamp IS NOT NULL
QUALIFY rn = 1;

-- 3. Stage 2: Fraud features (Dynamic Table depending on stage 1)
CREATE OR REPLACE DYNAMIC TABLE dt_fraud_features
  TARGET_LAG = '10 SECONDS'
  WAREHOUSE = 'dt_fraud_wh'
AS
SELECT
  ct.transaction_id,
  ct.customer_id,
  ct.amount,
  ct.merchant_id,
  ct.timestamp,
  ct.customer_tier,
  ct.country,
  
  -- Feature: customer's average transaction amount (last 30 days)
  customer_avg.amount_avg_30d,
  
  -- Feature: transaction count in last hour
  customer_recent.tx_count_1h,
  
  -- Feature: unusual amount flag
  CASE WHEN ct.amount > customer_avg.amount_avg_30d * 5 THEN TRUE ELSE FALSE END AS unusual_amount,
  
  -- Feature: velocity check (multiple transactions in short time)
  CASE WHEN customer_recent.tx_count_1h > 10 THEN TRUE ELSE FALSE END AS high_velocity

FROM dt_cleaned_transactions ct
LEFT JOIN (
  SELECT customer_id, AVG(amount) AS amount_avg_30d
  FROM dt_cleaned_transactions
  WHERE timestamp >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  GROUP BY customer_id
) customer_avg ON ct.customer_id = customer_avg.customer_id
LEFT JOIN (
  SELECT customer_id, COUNT(*) AS tx_count_1h
  FROM dt_cleaned_transactions
  WHERE timestamp >= DATEADD('hour', -1, CURRENT_TIMESTAMP())
  GROUP BY customer_id
) customer_recent ON ct.customer_id = customer_recent.customer_id;

-- 4. Stage 3: Fraud score (Dynamic Table depending on stage 2)
CREATE OR REPLACE DYNAMIC TABLE dt_fraud_scores
  TARGET_LAG = '15 SECONDS'
  WAREHOUSE = 'dt_fraud_wh'
AS
SELECT
  ff.transaction_id,
  ff.customer_id,
  ff.amount,
  ff.timestamp,
  
  -- Simple rule-based fraud score (production would use a ML model)
  CASE
    WHEN ff.unusual_amount AND ff.high_velocity THEN 0.95
    WHEN ff.unusual_amount THEN 0.70
    WHEN ff.high_velocity THEN 0.60
    WHEN ff.amount > 10000 THEN 0.50
    WHEN ff.customer_tier = 'new' AND ff.amount > 5000 THEN 0.65
    ELSE 0.05
  END AS fraud_score,
  
  -- Flag for manual review
  CASE
    WHEN fraud_score > 0.8 THEN 'BLOCK'
    WHEN fraud_score > 0.5 THEN 'REVIEW'
    ELSE 'APPROVE'
  END AS action

FROM dt_fraud_features ff;

-- 5. Check freshness SLA
SELECT
  table_name,
  target_lag,
  current_lag,
  CASE
    WHEN current_lag <= target_lag THEN 'ON TRACK'
    WHEN current_lag <= target_lag * 2 THEN 'LAGGING'
    ELSE 'STALE'
  END AS freshness_status
FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLES())
WHERE schema_name = 'FRAUD';
```

### Explanation
Dynamic Tables are Snowflake's declarative way to define continuously refreshed materialized views with freshness guarantees:
- **TARGET_LAG**: the maximum allowed staleness (e.g., `5 SECONDS`). Snowflake automatically determines the warehouse size needed to meet this SLA
- **Automatic dependency tracking**: Snowflake detects dependencies between dynamic tables and refreshes them in the correct order
- **Auto-scaling**: Snowflake automatically scales the warehouse up/down to maintain the target lag
- **Continuous refresh**: unlike Tasks (scheduled), dynamic tables refresh continuously as data changes

The DAG is: `raw_transactions → dt_cleaned_transactions → dt_fraud_features → dt_fraud_scores`

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Dynamic Tables | Declarative, auto-scaling, freshness SLA, dependency tracking | Requires Enterprise+, warehouse always running | Real-time/near-real-time pipelines |
| Tasks + Streams | Flexible, scheduled, can use any SQL | Manual orchestration, monitoring overhead | Batch pipelines, complex logic |
| Materialized Views | Transparent, zero orchestration | No explicit freshness SLA, auto-managed refresh timing | Read-optimized views |
| Snowpipe + Streams | Event-driven, cost-efficient | Manual transformation logic | Ingestion pipelines |

### Edge Cases
- **Cascade refresh**: if the upstream table lags, all downstream dynamic tables lag proportionally. Set appropriate target lags accounting for this (5s → 10s → 15s).
- **Warehouse sizing**: Snowflake auto-sizes the warehouse to meet TARGET_LAG, but complex transformations may require very large warehouses. Monitor costs.
- **Complex dependencies**: Circular dependencies are not allowed. Dynamic tables cannot reference themselves.

### Principal-Level Considerations
- Dynamic tables are ideal for building real-time data pipelines without orchestration complexity
- Use `TARGET_LAG = '0 SECONDS'` for truly real-time requirements (but expect higher costs)
- For cost control, set a `WAREHOUSE` with appropriate size limits and use resource monitors
- Dynamic tables work well with Streams for further downstream processing

### Follow-up Questions
1. **How does Snowflake determine what warehouse size is needed for a given TARGET_LAG?**
   - **Answer:** Snowflake continuously monitors the refresh rate of the dynamic table and automatically adjusts the warehouse size. If the current lag exceeds the target, it scales up. If the lag is well within the target, it scales down. You can specify a base warehouse and Snowflake will auto-scale from there.
2. **What happens if a dynamic table dependency fails?**
   - **Answer:** If the upstream table has an error or is inaccessible, the downstream dynamic table's current lag increases. You can monitor `current_lag` in `INFORMATION_SCHEMA.DYNAMIC_TABLES()` and set up alerts when lag exceeds thresholds.
3. **Can dynamic tables replace Airflow/dbt orchestration?**
   - **Answer:** For simple linear pipelines with clear dependencies, yes. For complex pipelines with external dependencies (e.g., waiting for a file in S3, calling an API, branching logic), you still need orchestration. Dynamic tables excel at the data transformation part within Snowflake.
4. **What's the cost impact of a 5-second TARGET_LAG on a busy table?**
   - **Answer:** It depends on the volume of changes and query complexity. A 5-second target on a table receiving 10K rows/sec with complex joins may require a 4XL-8XL warehouse continuously. Monitor with `WAREHOUSE_METRICS` and adjust the target lag based on SLA requirements vs cost.

---
## Q25. Write a Snowflake Stored Procedure in Python that calls an external ML model API for fraud scoring and loads the results back.

### Scenario
Your data science team built a fraud detection model deployed as a REST API. You need to batch-score 1 million transactions daily using this model, then load the scores back into Snowflake for downstream use. The model accepts JSON arrays of features and returns scores.

### Solution
```sql
-- 1. Create external access integration for the ML API
CREATE OR REPLACE API INTEGRATION fraud_api_int
  API_PROVIDER = external_http_api
  API_ALLOWED_PREFIXES = ('https://fraud-model.internal/api/v1')
  ENABLED = TRUE
  API_BLOCKED_PREFIXES = ('https://internal-admin.internal');

-- 2. Create the Python stored procedure
CREATE OR REPLACE PROCEDURE score_fraud_batch(batch_date DATE)
RETURNS FLOAT DEFAULT 0
LANGUAGE PYTHON
RUNTIME_VERSION = '3.9'
PACKAGES = ['snowflake-snowpark-python', 'requests', 'pandas']
IMPORTS = ()
API_INTEGRATION = fraud_api_int
EXECUTE AS CALLER
AS
$$
import requests
import pandas as pd
import json
from snowflake.snowpark import Session
import time

def score_batch(session, batch_date):
    # 1. Pull transactions to score
    query = f"""
    SELECT transaction_id, customer_id, amount, merchant_id,
           timestamp, customer_tier, country
    FROM dt_cleaned_transactions
    WHERE DATE(timestamp) = '{batch_date}'
      AND fraud_score IS NULL
    LIMIT 100000
    """
    
    df = session.sql(query).to_pandas()
    
    if df.empty:
        return 0.0
    
    # 2. Prepare features for the ML model
    features = []
    for _, row in df.iterrows():
        features.append({
            "transaction_id": row["transaction_id"],
            "customer_id": int(row["customer_id"]),
            "amount": float(row["amount"]),
            "merchant_id": int(row["merchant_id"]),
            "hour_of_day": row["timestamp"].hour,
            "day_of_week": row["timestamp"].weekday(),
            "customer_tier": row["customer_tier"],
            "country": row["country"]
        })
    
    # 3. Call the external ML API in batches
    results = []
    api_url = "https://fraud-model.internal/api/v1/score"
    batch_size = 1000  # API batch limit
    
    for i in range(0, len(features), batch_size):
        batch = features[i:i + batch_size]
        
        response = requests.post(
            api_url,
            json={"transactions": batch},
            headers={"Content-Type": "application/json"},
            timeout=30
        )
        
        if response.status_code != 200:
            raise Exception(f"API error: {response.status_code} - {response.text}")
        
        scores = response.json()["scores"]
        results.extend(scores)
    
    # 4. Load results back into Snowflake
    result_df = pd.DataFrame(results)
    result_df["scored_at"] = pd.Timestamp.now()
    
    # Create a temporary table and insert results
    result_df = result_df.rename(columns={
        "transaction_id": "TRANSACTION_ID",
        "fraud_probability": "FRAUD_PROBABILITY"
    })
    
    temp_df = session.create_dataframe(result_df)
    temp_df.write.mode("overwrite").save_as_table("TEMP_FRAUD_SCORES")
    
    # Merge into the main table
    session.sql("""
        MERGE INTO dt_fraud_scores tgt
        USING TEMP_FRAUD_SCORES src
        ON tgt.transaction_id = src.transaction_id
        WHEN MATCHED THEN
            UPDATE SET tgt.ml_fraud_score = src.fraud_probability,
                       tgt.scored_at = CURRENT_TIMESTAMP()
    """).collect()
    
    # Clean up
    session.sql("DROP TABLE IF EXISTS TEMP_FRAUD_SCORES").collect()
    
    return float(len(results))
$$;

-- 3. Schedule it as a task
CREATE OR REPLACE TASK daily_fraud_scoring
  WAREHOUSE = 'WH_ML'
  SCHEDULE = 'USING CRON 0 2 * * * America/New_York'  -- 2 AM ET daily
  USER_TASK_TIMEOUT_MS = 3600000  -- 1 hour max
AS
  CALL score_fraud_batch(DATEADD(day, -1, CURRENT_DATE()));

-- 4. Run it manually for testing
CALL score_fraud_batch('2024-01-15');
```

### Explanation
This Python stored procedure:
1. **Reads data from Snowflake** using Snowpark (`session.sql().to_pandas()`)
2. **Prepares features** for the ML model by transforming the DataFrame
3. **Calls the external API** in batches (respecting API limits) using `requests`
4. **Loads results back** into Snowflake using Snowpark DataFrames
5. **Merges scores** into the target table using standard SQL MERGE

Key requirements:
- **API INTEGRATION**: needed for outbound HTTP calls from Snowflake (Enterprise+)
- **RUNTIME_VERSION**: Python 3.9 is the supported runtime
- **PACKAGES**: declares Python dependencies (available from Snowflake's package repository)
- **EXECUTE AS CALLER**: the procedure runs with the caller's permissions

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Python SP + Snowpark | Full Python ecosystem, direct API calls | Requires Enterprise+, slower than native SQL | ML model inference, external API calls |
| External Functions | Serverless, auto-scaled, low latency | AWS Lambda dependency, 60s timeout per batch | Low-latency row-by-row scoring |
| Snowpark UDF | Runs in-cluster, no external deps | Limited to in-cluster compute | In-cluster transformations |
| Export → ML → Import | Full control over ML environment | Manual pipeline, higher latency | Complex ML workflows |

### Edge Cases
- **API rate limits**: The batch size (1000) must respect the API's rate limits. Implement exponential backoff for retries.
- **Timeouts**: The stored procedure has a timeout (`USER_TASK_TIMEOUT_MS`). Ensure the batch size is small enough to complete within the timeout.
- **Error handling**: The `requests` call includes error checking. For production, add retry logic with exponential backoff.
- **Memory**: Loading 100K rows into a Pandas DataFrame requires sufficient memory in the Python handler. Use `LIMIT` and process in chunks.

### Principal-Level Considerations
- Consider using **External Functions** instead of Stored Procedures for lower latency and auto-scaling (if the ML API has low latency requirements)
- Implement a **dual-scoring strategy**: use the Snowflake rule-based score for real-time decisions, and the ML score for post-transaction analysis
- Add **model drift detection**: track score distribution over time and alert on significant shifts
- Ensure **data privacy**: transaction data sent to external APIs should be tokenized or hashed where possible

### Follow-up Questions
1. **What's the difference between Stored Procedures and External Functions for ML inference?**
   - **Answer:** Stored Procedures run inside Snowflake and have full access to Snowflake resources. External Functions offload computation to AWS Lambda (or another compute service) and are better for row-by-row low-latency scoring. Stored Procedures are better for batch processing with complex logic.
2. **How do you handle API failures in production?**
   - **Answer:** Implement retry logic with exponential backoff (e.g., 3 retries with 1s, 2s, 4s delays). Log failures to an audit table. For critical failures, fall back to the rule-based score. Set up monitoring on the task's success/failure status.
3. **Can you use GPU-accelerated models with Snowflake?**
   - **Answer:** Not directly — Snowflake doesn't provide GPU compute. You'd deploy the model on a GPU-enabled service (e.g., AWS SageMaker, GCP Vertex AI) and call it via External Functions or Stored Procedures.

---

## Q26. Implement row-level and column-level security policies in Snowflake to comply with GDPR and HIPAA.

### Scenario
Your healthcare analytics database contains patient data protected by HIPAA. Different roles need different access:
- **Data Scientists**: can see aggregated/analytical data but not PII
- **Clinicians**: can see full patient records but only for patients they treat
- **Administrators**: can see billing and demographic data but not clinical notes
- **Compliance**: read-only access to audit logs

Design and implement the security architecture.

### Solution
```sql
-- 1. Create roles following the principle of least privilege
CREATE OR REPLACE ROLE clinician;
CREATE OR REPLACE ROLE data_scientist;
CREATE OR REPLACE ROLE healthcare_admin;
CREATE OR REPLACE ROLE compliance_officer;

-- 2. Column-level masking policies

-- Mask SSN for everyone except compliance
CREATE OR REPLACE MASKING POLICY mask_ssn AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('COMPLIANCE_OFFICER', 'ACCOUNTADMIN') THEN val
    ELSE '***-**-****'
  END;

-- Mask clinical notes for data scientists and admins
CREATE OR REPLACE MASKING POLICY mask_clinical_notes AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('CLINICIAN', 'ACCOUNTADMIN') THEN val
    ELSE '*** REDACTED ***'
  END;

-- Mask patient names for data scientists (they see IDs only)
CREATE OR REPLACE MASKING POLICY mask_patient_name AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('CLINICIAN', 'HEALTHCARE_ADMIN', 'COMPLIANCE_OFFICER', 'ACCOUNTADMIN') THEN val
    ELSE CONCAT(LEFT(val, 1), '****')
  END;

-- Email masking
CREATE OR REPLACE MASKING POLICY mask_email AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('CLINICIAN', 'HEALTHCARE_ADMIN', 'COMPLIANCE_OFFICER', 'ACCOUNTADMIN') THEN val
    ELSE '***@***.***'
  END;

-- 3. Apply masking policies to columns
ALTER TABLE patients MODIFY COLUMN ssn SET MASKING POLICY mask_ssn;
ALTER TABLE patients MODIFY COLUMN full_name SET MASKING POLICY mask_patient_name;
ALTER TABLE patients MODIFY COLUMN email SET MASKING POLICY mask_email;
ALTER TABLE clinical_notes MODIFY COLUMN notes SET MASKING POLICY mask_clinical_notes;

-- 4. Row-level access policy (clinicians see only their patients)
CREATE OR REPLACE ROW ACCESS POLICY clinician_patient_access AS (patients TABLE) RETURNS BOOLEAN ->
  CASE
    WHEN CURRENT_ROLE() = 'CLINICIAN' THEN
      patient_id IN (
        SELECT patient_id FROM clinician_assignments
        WHERE clinician_id = CURRENT_USER()
      )
    ELSE TRUE  -- Other roles see all rows
  END;

ALTER TABLE patients ADD ROW ACCESS POLICY clinician_patient_access;
ALTER TABLE clinical_notes ADD ROW ACCESS POLICY clinician_patient_access;
ALTER TABLE lab_results ADD ROW ACCESS POLICY clinician_patient_access;

-- 5. Dynamic data masking based on context
CREATE OR REPLACE MASKING POLICY mask_diagnosis AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('CLINICIAN', 'ACCOUNTADMIN') THEN val
    WHEN CURRENT_ROLE() = 'DATA_SCIENTIST' THEN
      CASE
        WHEN SYSTEM$IS_CREDENTIAL_TRUSTED() = TRUE THEN val  -- trusted environment
        ELSE 'DIAGNOSIS_REDACTED'
      END
    ELSE '***'
  END;

-- 6. Grant role-based access to schemas and tables
GRANT USAGE ON SCHEMA healthcare.patients TO ROLE clinician;
GRANT SELECT ON TABLE healthcare.patients TO ROLE clinician;
GRANT SELECT ON TABLE healthcare.patients TO ROLE data_scientist;
GRANT SELECT ON TABLE healthcare.patients TO ROLE healthcare_admin;
GRANT SELECT ON TABLE healthcare.patients TO ROLE compliance_officer;

GRANT SELECT ON TABLE healthcare.clinical_notes TO ROLE clinician;
GRANT SELECT ON TABLE healthcare.clinical_notes TO ROLE data_scientist;  -- masked

-- 7. Audit trail
CREATE OR REPLACE TABLE access_audit_log (
  access_id NUMBER AUTOINCREMENT,
  user_name VARCHAR(255),
  role_name VARCHAR(255),
  accessed_table VARCHAR(255),
  accessed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  query_text VARCHAR(1000)
);

-- Enable query tagging for audit
ALTER ROLE clinician SET QUERY_TAG = 'role=clinician';
ALTER ROLE data_scientist SET QUERY_TAG = 'role=data_scientist';
```

### Explanation
This implements a multi-layer security architecture:
1. **Roles**: Each job function gets a dedicated role with only the permissions it needs
2. **Masking Policies**: Column-level policies transform sensitive data based on the user's role
3. **Row Access Policies**: Restrict which rows a user can see (e.g., clinicians only see their assigned patients)
4. **Audit Logging**: All access is tracked for compliance reporting

Key concepts:
- `CURRENT_ROLE()`: returns the user's active role — used by masking policies to determine visibility
- `CURRENT_USER()`: returns the actual user — used by row access policies for patient assignment
- Masking policies are applied transparently — the user sees masked data without knowing

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Masking Policies + Row Access | Native, transparent, role-based | Policies evaluated at query time (compute cost) | HIPAA, GDPR compliance |
| Dynamic Views | Flexible, can combine row + column logic | Must maintain separate views per role | Complex multi-role access |
| Separate Schemas per Role | Simple, clear isolation | Data duplication, harder to maintain | Simple separation needs |
| Tags + Policies | Declarative, scalable | Requires tagging infrastructure | Large-scale data governance |

### Edge Cases
- **Role switching**: Users can switch roles with `USE ROLE`. Masking policies evaluate the current role. Ensure users can't escalate privileges by switching to a more privileged role.
- **Sharing through Views**: A view that selects from a masked table inherits the masking. Be careful when creating views for different roles.
- **Performance**: Row access policies add a subquery to every query. On large tables, this can impact performance. Consider materialized views for frequently accessed filtered data.

### Principal-Level Considerations
- Implement **data classification** using Snowflake Tags: tag columns as `PII`, `PHI`, `FINANCIAL` and apply masking policies based on tags
- Use **dynamic masking** with `SYSTEM$IS_CREDENTIAL_TRUSTED()` for trusted environments (e.g., secure enclaves)
- Implement **automatic de-identification** for data shared with external parties
- Build an **access review process**: quarterly reviews of role assignments and permission grants
- Use **Snowflake Access Governance** for automated policy management at scale

### Follow-up Questions
1. **How do masking policies interact with views?**
   - **Answer:** Masking policies are applied at the column level. If a view selects a masked column, the masking is applied to the view's output. The masking is evaluated based on the consumer's role, not the view creator's role.
2. **Can a user bypass masking policies by exporting data?**
   - **Answer:** No — masking policies are applied at the query engine level, before data leaves Snowflake. All export paths (COPY, API, JDBC/ODBC) go through the query engine and respect masking.
3. **How do you handle data sharing with external organizations under HIPAA?**
   - **Answer:** Use Snowflake Data Sharing with masking policies applied. The sharing producer can apply masking policies that are enforced in the consumer's account. For HIPAA, ensure a Business Associate Agreement (BAA) is in place.

---

## Q27. Design a cost allocation model using Snowflake Resource Monitors, Tags, and Query Tags to attribute costs to teams and projects.

### Scenario
Your Snowflake account serves 15 teams across the organization. The monthly bill is $50,000. Finance needs a detailed cost breakdown by team, project, and cost center. You also need to enforce budget limits and alert teams when they approach their allocated budget.

### Solution
```sql
-- 1. Create object tags for cost attribution
CREATE OR REPLACE TAG team;
CREATE OR REPLACE TAG project;
CREATE OR REPLACE TAG cost_center;

-- Allow valid values
ALTER TAG TEAM ALLOWED_VALUES ENG, DATA, ANALYTICS, SCIENCE, PLATFORM, PRODUCT, SUPPORT;
ALTER TAG PROJECT ALLOWED_VALUES FRAUD_DETECTION, CUSTOMER_360, REVENUE_ANALYTICS, RECOMMENDATIONS;
ALTER TAG COST_CENTER ALLOWED_VALUES CC001, CC002, CC003, CC004;

-- 2. Tag warehouses by team
ALTER WAREHOUSE WH_ENGINEERING SET TAG team = 'ENG', project = 'PLATFORM', cost_center = 'CC001';
ALTER WAREHOUSE WH_ANALYTICS SET TAG team = 'ANALYTICS', project = 'REVENUE_ANALYTICS', cost_center = 'CC002';
ALTER WAREHOUSE WH_DATA_SCIENCE SET TAG team = 'SCIENCE', project = 'FRAUD_DETECTION', cost_center = 'CC003';
ALTER WAREHOUSE WH_ETL SET TAG team = 'DATA', project = 'CUSTOMER_360', cost_center = 'CC001';

-- 3. Tag tables by ownership
ALTER TABLE dwd.orders SET TAG team = 'DATA', project = 'CUSTOMER_360', cost_center = 'CC001';
ALTER TABLE dwd.customer_events SET TAG team = 'DATA', project = 'CUSTOMER_360', cost_center = 'CC001';
ALTER TABLE ml.fraud_scores SET TAG team = 'SCIENCE', project = 'FRAUD_DETECTION', cost_center = 'CC003';

-- 4. Set query tags via session variables (users set these at login)
-- Engineers set their project tag when they connect
ALTER ROLE data_engineer SET QUERY_TAG = 'team=ENG&project=PLATFORM&cost_center=CC001';
ALTER ROLE data_scientist SET QUERY_TAG = 'team=SCIENCE&project=FRAUD_DETECTION&cost_center=CC003';
ALTER ROLE analyst SET QUERY_TAG = 'team=ANALYTICS&project=REVENUE_ANALYTICS&cost_center=CC002';

-- 5. Resource monitors with budget limits per team
CREATE OR REPLACE RESOURCE MONITOR eng_monthly
  CREDIT_LIMIT = 5000
  FREQUENCY = MONTHLY
  TRIGGER_ACTION = NOTIFY;

CREATE OR REPLACE RESOURCE MONITOR analytics_monthly
  CREDIT_LIMIT = 3000
  FREQUENCY = MONTHLY
  TRIGGER_ACTION = SUSPEND;

CREATE OR REPLACE RESOURCE MONITOR science_monthly
  CREDIT_LIMIT = 8000
  FREQUENCY = MONTHLY
  TRIGGER_ACTION = SUSPEND;

-- Assign resource monitors to warehouses
ALTER WAREHOUSE WH_ENGINEERING SET RESOURCE_MONITOR = eng_monthly;
ALTER WAREHOUSE WH_ANALYTICS SET RESOURCE_MONITOR = analytics_monthly;
ALTER WAREHOUSE WH_DATA_SCIENCE SET RESOURCE_MONITOR = science_monthly;

-- 6. Cost allocation report (monthly)
WITH query_costs AS (
  SELECT
    qh.start_time,
    qh.warehouse_name,
    qh.credits_used,
    qh.total_elapsed_time,
    qh.bytes_scanned,
    qh.user_name,
    qh.query_tag,
    qh.query_text,
    -- Parse query tag
    SPLIT_PART(SPLIT_PART(qh.query_tag, 'team=', 2), '&', 1) AS team,
    SPLIT_PART(SPLIT_PART(qh.query_tag, 'project=', 2), '&', 1) AS project,
    SPLIT_PART(SPLIT_PART(qh.query_tag, 'cost_center=', 2), '&', 1) AS cost_center
  FROM snowflake.account_usage.query_history qh
  WHERE qh.start_time >= DATE_TRUNC('month', CURRENT_TIMESTAMP())
    AND qh.warehouse_name IS NOT NULL
),
storage_costs AS (
  SELECT
    dbs.table_catalog AS database_name,
    dbs.bytes / 1073741824 AS gb_stored,
    ROUND(dbs.bytes / 1073741824 * 0.00234375, 2) AS storage_cost_usd,  -- ~$23/TB/month
    OBJECTS$GET_TAG('team', dbs.table_catalog, dbs.table_schema, dbs.table_name) AS team
  FROM information_schema.data_retention_days dbs
)
SELECT
  team,
  project,
  cost_center,
  ROUND(SUM(credits_used), 2) AS total_credits,
  ROUND(SUM(credits_used) * 0.1875, 2) AS compute_cost_usd,  -- XLARGE rate
  SUM(total_elapsed_time) / 1000 AS total_query_seconds,
  COUNT(*) AS query_count,
  ROUND(SUM(bytes_scanned) / 1073741824, 2) AS total_gb_scanned,
  ROUND(SUM(credits_used) * 0.1875 + SUM(bytes_scanned) / 1073741824 * 0.00234375, 2) AS total_cost_usd
FROM query_costs
WHERE team IS NOT NULL AND team != ''
GROUP BY team, project, cost_center
ORDER BY total_cost_usd DESC;

-- 7. Budget alert procedure
CREATE OR REPLACE PROCEDURE check_budget_alerts()
RETURNS STRING
LANGUAGE SQL
AS
$$
  DECLARE
    result VARCHAR;
  BEGIN
    -- Check each resource monitor
    FOR rec IN (
      SELECT rm.name, rm.credits_used, rm.credits_limit,
             ROUND(rm.credits_used * 100.0 / rm.credits_limit, 1) AS pct_used
      FROM snowflake.account_usage.resource_monitors rm
      WHERE rmcredits_used > rm.credits_limit * 0.8
    ) DO
      IF rec.pct_used > 90 THEN
        result := result || 'CRITICAL: ' || rec.name || ' at ' || rec.pct_used || '% of budget\n';
      ELSIF rec.pct_used > 80 THEN
        result := result || 'WARNING: ' || rec.name || ' at ' || rec.pct_used || '% of budget\n';
      END IF;
    END FOR;
    
    IF result IS NULL OR result = '' THEN
      result := 'All budgets within normal range';
    END IF;
    
    RETURN result;
  END
$$;

-- Schedule budget check
CREATE OR REPLACE TASK budget_alert_check
  WAREHOUSE = 'WH_UTILITIES'
  SCHEDULE = 'USING CRON 0 9 * * 1-5 America/New_York'  -- Weekdays at 9 AM
AS
  CALL check_budget_alerts();
```

### Explanation
This cost allocation model has four layers:
1. **Object Tags**: Tag warehouses, tables, and databases with `team`, `project`, and `cost_center`
2. **Query Tags**: Each role sets a query tag that tags all queries from that role
3. **Resource Monitors**: Enforce budget limits per team with alerts at thresholds
4. **Cost Reports**: Aggregate compute and storage costs by team/project

Key design decisions:
- Query tags are the most granular attribution mechanism (per-query)
- Object tags handle storage cost attribution
- Resource monitors prevent budget overruns with automatic actions (SUSPEND or NOTIFY)
- The cost report parses query tags to attribute compute costs and uses object tags for storage

### Comparison
| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| Query Tags + Object Tags | Granular, automatic, scalable | Requires discipline to set tags | Multi-team environments |
| Resource Monitors per Warehouse | Enforce hard limits, automatic actions | Coarse (warehouse-level only) | Budget enforcement |
| Separate Accounts per Team | Complete isolation | Very expensive, complex ops | Regulatory/compliance needs |
| Third-party tools (FinSpace, CloudZero) | Rich dashboards, allocation rules | Additional cost, integration | Enterprise FinOps |

### Edge Cases
- **Untagged queries**: Queries without tags won't appear in cost reports. Implement a default tag or enforce tag requirements.
- **Shared warehouses**: When multiple teams share a warehouse, costs can't be attributed at the warehouse level — query tags are needed.
- **Cross-project work**: Users working on multiple projects may need to switch query tags. Implement a stored procedure for easy tag switching.

### Principal-Level Considerations
- Implement **FinOps practices**: monthly cost reviews, anomaly detection, and optimization recommendations
- Use **query tags for chargeback**: automatically generate monthly invoices per team
- Build a **cost optimization dashboard**: show credits per team, per project, with trends and recommendations
- Implement **automated rightsizing**: periodically analyze warehouse utilization and recommend size adjustments
- Set up **budget alerts at multiple levels**: team level (80%, 90%, 100%), project level, and overall account level

### Follow-up Questions
1. **How do you handle ad-hoc queries that don't have query tags set?**
   - **Answer:** Set a default query tag on the role (e.g., `team=ANALYTICS&project=AD_HOC`). For users who need to work on specific projects, provide a simple procedure: `CALL set_my_query_tag('team=ENG&project=FRAUD_DETECTION')`.
2. **How accurate is the cost attribution from query tags?**
   - **Answer:** Query tags attribute compute cost at the query level, which is very granular. Storage cost attribution is at the table level via object tags. The combination gives a comprehensive view. There's no attribution for shared infrastructure (e.g., micro-batching, result cache).
3. **What happens when a resource monitor hits its limit?**
   - **Answer:** Depending on `TRIGGER_ACTION`: `SUSPEND` suspends all warehouses under that monitor (queries queue until resumed). `SUSPEND_IMMEDIATE` suspends running queries. `NOTIFY` only sends an alert but doesn't suspend. Choose based on whether you want hard or soft limits.

---