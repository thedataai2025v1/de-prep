# dbt Theory Interview Questions

| Category | Details |
|---|---|
| **Role Level** | Principal Data Engineer (17+ years) |
| **Target Companies** | Atlassian, Snowflake, Databricks, Uber, Confluent, Google, Adobe |
| **Interview Frequency** | High — core competency for modern data engineering roles |
| **Topics** | dbt architecture, materializations, incremental modeling, snapshots, macros, CI/CD, governance, data contracts |

---

## Q1. What is dbt and how does it differ from traditional ETL tools?

### Problem Statement

Explain what dbt is, the "T" in ELT vs traditional ETL, and articulate the philosophical and architectural differences between dbt and tools like Informatica, Talend, SSIS, or Fivetran.

### Answer

**dbt (data build tool)** is a transformation layer that runs **inside** the data warehouse or data lakehouse. It transforms raw, loaded data into analytics-ready models using **SQL and Jinja templating**. It inverts the traditional ETL paradigm to **ELT**:

- **Extract:** Data is loaded raw into the warehouse (by Fivetran, Airbyte, Stitch, etc.)
- **Load:** Land raw/normalized data directly in the warehouse
- **Transform:** dbt transforms data **in-warehouse** using SQL

Unlike traditional ETL tools that extract data from the warehouse, transform it in an external engine, and load it back, dbt leverages the **computational power of the warehouse itself** (Snowflake, BigQuery, Databricks) for transformations. This eliminates data movement bottlenecks and leverages the warehouse's query optimizer, partitioning, and parallelism.

**Key philosophical differences:**
- **Code as SQL:** dbt models are SQL files in a code repository — fully version-controlled, code-reviewable, and testable
- **Software engineering practices for data:** CI/CD, documentation, testing, and modular design applied to data pipelines
- **Declarative vs imperative:** You define *what* to build (SELECT queries); dbt figures out *how* and *when* to build it based on dependencies
- **No data movement:** Transformations happen where the data lives — reducing compute costs and latency

### Example

```sql
-- Traditional ETL: Extract → Transform in external engine → Load back
-- dbt ELT: Raw data is already in the warehouse; dbt transforms in place

-- dbt model: models/staging/stg_orders.sql
{{ config(
    materialized = 'view',
    tags = ['staging', 'orders']
) }}

WITH source AS (
    SELECT * FROM {{ source('raw_jaffle_shop', 'orders') }}
),

renamed AS (
    SELECT
        order_id,
        customer_id,
        CAST(status AS STRING) AS status,
        CAST(order_date AS DATE) AS order_date,
        CAST(shipped_date AS DATE) AS shipped_date
    FROM source
)

SELECT * FROM renamed
```

### Pros/Cons

| Aspect | dbt (ELT) | Traditional ETL (Informatica/SSIS) |
|---|---|---|
| **Compute location** | In-warehouse (leverages cloud DW scale) | External servers (bottlenecked) |
| **Code versioning** | Native (Git-based) | Limited / proprietary tools |
| **Testing** | Built-in (unique, not_null, relationships, custom) | Manual / external testing frameworks |
| **Documentation** | Auto-generated (dbt docs, catalog) | Manual / separate wikis |
| **Scalability** | Scales with warehouse compute | Requires horizontal scaling of ETL servers |
| **Cost model** | Warehouse compute credits + dbt Cloud | ETL server licenses + warehouse compute |
| **Data freshness** | Near real-time (run on-demand) | Batch windows (hours/days) |
| **Learning curve** | SQL + Jinja (data engineers) | Proprietary GUIs + custom scripting |

**When traditional ETL still wins:**
- Heavy data cleaning/masking before warehouse ingestion (PII scrubbing)
- Transformations that cannot be expressed in SQL (image processing, NLP)
- On-premise environments without a cloud warehouse

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Warehouse compute spike from back-to-back dbt runs | Snowflake credit burn / query timeouts | Stagger model runs, use incremental models, configure warehouse auto-suspend |
| Massive table scans in staging (no partitioning) | $10K+ single run on BigQuery | Add partition/clustering configs early, use ephemeral staging |
| Circular dependency in DAG | dbt fails with compilation error | Validate DAG with `dbt deps` and `dbt parse` in CI |
| Raw source schema change undetected | Downstream models silently produce wrong results | Source freshness tests, schema evolution strategies, data contracts (dbt 1.8+) |
| Incremental model fails mid-run with `merge` strategy | Partial data, inconsistent state | Use `--empty` flag for re-runs, implement idempotent merge logic |

### Follow-Up Questions

**Q1. Why did dbt choose ELT over ETL? Is it always better?**
→ ELT leverages the massive parallelism and storage of cloud warehouses. At scale (TB+), moving data out of the warehouse just to transform it is expensive and slow. However, ELT is not always better — if transformations are CPU-intensive non-SQL operations (image processing, regex-heavy parsing on small data), ETL may still be more cost-effective.

**Q2. Can dbt replace tools like Airflow or Dagster?**
→ No. dbt is a **transformation orchestrator** — it manages dependencies between SQL models. It is not a general-purpose workflow scheduler. Atlassian uses dbt **within** Airflow/Dagster pipelines: Airflow handles orchestration of external tasks (data ingestion, Python scripts, notifications), and dbt handles the SQL transformation layer. The two complement each other.

**Q3. What happens if your warehouse goes down during a dbt run?**
→ dbt depends on warehouse availability. If the warehouse is unavailable, dbt runs fail. At Principal level, you design for this: use dbt Cloud's retry policies, implement alerting on failed runs, and ensure that failed incremental models can be safely re-run idempotently.

**Q4. How does dbt handle data that doesn't fit in the warehouse (e.g., files > warehouse limits)?**
→ dbt itself doesn't handle data loading. For extremely large files, use the warehouse-native loading tools (Snowpipe, BigQuery load jobs, Databricks Autoloader). dbt then transforms the data once it's in the warehouse. For lakehouse architectures (Databricks), dbt can also run as Spark SQL on Delta Lake.

**Q5. Is dbt just for SQL transformations? Can it handle Python?**
→ dbt supports Python models (dbt 1.5+) that run as PySpark (Databricks), dbt Python (BigQuery/Snowflake), or Pandas (local). At scale, Python models should be used judiciously — they bypass the query optimizer and can be significantly slower than equivalent SQL. Use them only when SQL cannot express the logic (complex ML scoring, API calls).

**Q6. How does dbt compare to Spark SQL or Databricks notebooks for transformation?**
→ dbt brings software engineering discipline (version control, testing, documentation, dependency management) to transformations. Spark SQL/Databricks notebooks are more flexible but lack built-in dependency graphs, automated testing, and documentation. At Uber, dbt runs on top of Databricks Spark SQL, combining the best of both.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| "dbt loads data into the warehouse" | dbt does **no** data loading — that's Fivetran/Airbyte | Clarify: dbt is transformation-only (the "T") |
| "dbt replaces Airflow" | dbt manages SQL dependencies; Airflow orchestrates workflows | Use both: Airflow for orchestration, dbt for transformations |
| "dbt works without a warehouse" | dbt requires a SQL adapter (warehouse/lakehouse) | Understand that dbt is a query generator, not a compute engine |
| "All models should be tables" | Tables consume storage and refresh fully each run | Use views for lightweight queries, incremental for large tables |

### Principal-Level Considerations

- **Architecture decision:** At scale (1000+ models, TB of data), the choice of which layers are views vs tables vs incremental models has direct cost implications. At Snowflake, a full table refresh of a 100GB model costs compute credits proportional to data scanned. Incremental models scanning only new rows save 90%+ compute.
- **Governance:** dbt enables data teams to apply consistent governance — naming conventions, tagging, documentation, and testing — across hundreds of models. At Atlassian, this discipline is critical for self-serve analytics.
- **Cost optimization:** At the Principal level, you must architect the dbt project to minimize compute: use ephemeral models for intermediate transformations, views for frequently queried marts, and incremental materializations for large fact tables. Monitor warehouse spend per dbt model.
- **Lakehouse vs warehouse:** With Databricks (Delta Lake), dbt can materialize as Delta tables with time travel, ACID transactions, and schema evolution — capabilities beyond traditional warehouses.

---

## Q2. Explain the dbt project structure and key configuration files (dbt_project.yml, profiles.yml, packages.yml)

### Problem Statement

Walk through the anatomy of a dbt project: the required and optional files, the purpose of each, and how they interact. Explain the separation between the project definition, database connection, and package management.

### Answer

A dbt project is a directory with a specific structure that dbt uses to understand your data transformation project. The key files form a **three-layer configuration system**:

1. **`dbt_project.yml`** — The project definition (what, how to build)
2. **`profiles.yml`** — The database connection (where to run)
3. **`packages.yml`** — The dependency management (what reusable packages to use)

**Standard project structure:**

```
my_dbt_project/
├── dbt_project.yml          # Project name, version, config defaults
├── packages.yml             # dbt package dependencies
├── profiles.yml             # Database connection profiles (usually in ~/.dbt/)
├── models/                  # SQL model files
│   ├── staging/             # Raw data → normalized
│   │   ├── stg_orders.sql
│   │   └── schema.yml
│   ├── intermediate/        # Joining, aggregation
│   │   └── int_order_facts.sql
│   └── marts/               # Final analytics tables/views
│       ├── customer_orders.sql
│       └── schema.yml
├── seeds/                   # CSV files loaded as tables
├── snapshots/               # SCD tracking models
├── macros/                  # Reusable Jinja macros
├── tests/                   # Custom data tests
├── analysis/                # Scratchpad for exploration (ignored by dbt)
└── docs/                    # Additional documentation
```

### Example

```yaml
# dbt_project.yml
name: 'jaffle_shop_analytics'
version: '1.0.0'
config-version: 2

# This setting tells dbt which profiles to look for
profile: 'jaffle_shop'

# Paths — override defaults if needed
model-paths: ["models"]
analysis-paths: ["analysis"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

# Target-specific configuration
target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

# Global model defaults — cascade to all models
models:
  jaffle_shop_analytics:
      +materialized: view
      +docs:
        +show: true

      staging:
          +materialized: view
          +tags: ['staging']
          +post-hook: "COMMENT ON TABLE {{ this }} IS 'Staging: raw data normalized'"

      intermediate:
          +materialized: ephemeral
          +tags: ['intermediate']

      marts:
          +materialized: table
          +tags: ['marts']
          +persist_docs:
            relation: true
            columns: true

# Global test config
tests:
  +store_failures: true
  +severity: ERROR

seeds:
  +column_types:
    order_status: varchar

snapshots:
  +target_schema: 'snapshot_tables'

# Package imports
require-dbt-version: [">=1.8.0", "<2.0.0"]

# Dispatch for cross-database macro resolution
dispatch:
  - macro_namespace: dbt_utils
    search_order: ['dbt_utils', 'jaffle_shop_analytics']
```

```yaml
# profiles.yml (typically stored in ~/.dbt/profiles.yml — NEVER in repo)
jaffle_shop:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: abc12345.us-east-1
      user: dev_user
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      database: JAFFLE_DB
      schema: DEV_SCHEMA
      role: TRANSFORM_ROLE
      threads: 4
      warehouse: TRANSFORM_WH

    prod:
      type: snowflake
      account: abc12345.us-east-1
      user: prod_user
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      database: JAFFLE_DB
      schema: PROD_SCHEMA
      role: TRANSFORM_ROLE
      threads: 8
      warehouse: TRANSFORM_WH
```

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: "1.1.1"
  - package: dbt-labs/codegen
    version: "2.0.0"
  - package: calogica/dbt_expectations
    version: "0.10.3"
```

### Pros/Cons of Configuration Approach

| File | Purpose | In Repo? | Sensitive Data? |
|---|---|---|---|
| `dbt_project.yml` | Project name, version, global config, model defaults | ✅ Yes | No |
| `profiles.yml` | Database credentials and connection info | ❌ No (~/.dbt/) | Yes — use env vars |
| `packages.yml` | External package dependencies with versions | ✅ Yes | No |
| `schema.yml` (per folder) | Model properties, tests, descriptions, configs | ✅ Yes | No |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| `profiles.yml` committed to Git | Credentials leaked to public repo | Store in `~/.dbt/`, use env vars for passwords, add `.gitignore` |
| `config-version` mismatch | dbt fails to parse project | Always specify `config-version: 2` for dbt 1.0+ |
| Missing `require-dbt-version` | Project runs on incompatible dbt version | Pin `require-dbt-version: [">=1.8.0", "<2.0.0"]` |
| `dispatch` misconfigured | Wrong macro resolved (falls back to wrong namespace) | Test dispatch resolution with `dbt parse` and verify macro resolution order |
| Package version conflict | Two packages require different versions of a dependency | Pin exact versions, use `dbt deps` to resolve, upgrade packages together |

### Follow-Up Questions

**Q1. Why is `profiles.yml` stored outside the project?**
→ It contains sensitive database credentials. Storing it in `~/.dbt/profiles.yml` keeps credentials out of version control. In dbt Cloud, connections are managed in the UI — no `profiles.yml` needed.

**Q2. What happens if I don't specify a `profile` in `dbt_project.yml`?**
→ dbt looks for a profile matching the project name. Best practice is to always specify the profile explicitly to avoid ambiguity, especially in multi-project monorepo setups.

**Q3. How do environment-specific configurations work?**
→ Use the `--vars` flag or environment variables. For example: `dbt run --vars '{ "target_env": "prod" }'` or reference `{{ env_var('ENVIRONMENT') }}` in models. dbt Cloud environments (dev, prod) map to different connection profiles and variables.

**Q4. What is the `clean-targets` configuration?**
→ These are directories that `dbt clean` will delete. Always include `target/` (compiled SQL, manifest.json) and `dbt_packages/` (installed packages). Never clean your source models, seeds, or macros.

**Q5. How does `dispatch` interact with packages?**
→ `dispatch` tells dbt to look in specific namespaces when resolving macros. If your project defines a macro `safe_divide` and `dbt_utils` also defines it, `dispatch` controls which one is used first. Order matters: the first matching macro in the search order wins.

**Q6. Can you have multiple dbt projects in one repo?**
→ Yes. Each project has its own `dbt_project.yml` in a subdirectory. Use `dbt run --project-dir subdir/` to run a specific project. This is common at companies like Atlassian with multiple product lines.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Committing `profiles.yml` to Git | Credential exposure | Store in `~/.dbt/` or use dbt Cloud connections |
| Hardcoding credentials in `profiles.yml` | Security risk, fails in CI/CD | Use `{{ env_var('SNOWFLAKE_PASSWORD') }}` |
| Missing `config-version: 2` | dbt 1.0+ won't parse the file | Always include `config-version: 2` |
| Putting all models in one folder | No logical organization, hard to maintain | Use `staging/`, `intermediate/`, `marts/` structure |
| Not pinning package versions | Breaking changes from upstream | Pin exact versions: `"1.1.1"` not `">=1.0"` |

### Principal-Level Considerations

- **Monorepo vs multi-repo:** At scale, teams debate a single large dbt project vs multiple smaller projects. At Google, large monorepos with dbt sub-projects allow shared macros and packages across teams while maintaining clear boundaries. The trade-off is build times — a 2000-model project may take 30+ minutes to compile.
- **Configuration as infrastructure:** Treat `dbt_project.yml` as infrastructure code. Changes to global materializations or test severity should go through the same review process as code changes.
- **Multi-environment strategy:** Production-grade dbt projects need at least 3 environments (dev, staging, prod) with different schemas, warehouses, and thread counts. dbt Cloud environments abstract this; CLI uses `--target dev/prod`.
- **Secrets management:** At Principal level, integrate with HashiCorp Vault, AWS Secrets Manager, or GCP Secret Manager. Never store credentials in plaintext anywhere.

---

## Q3. What are dbt materializations? Compare table, view, incremental, ephemeral with trade-offs and production examples

### Problem Statement

Explain what materializations are in dbt, the four core types, and when to choose each. Provide production-grade reasoning for the choice at TB-scale data warehouses.

### Answer

**Materialization** defines **how dbt persists the result of a model** in the database. It is the single most important architectural decision for each model, directly impacting compute cost, storage cost, query performance, and pipeline latency.

dbt generates the appropriate DDL (CREATE TABLE, CREATE VIEW, etc.) based on the materialization type. The model SQL is the same — only the output differs.

### Example

```sql
-- Same model logic, different materializations:

-- 1. TABLE: Full materialization
{{ config(materialized='table') }}

SELECT
    order_id,
    customer_id,
    order_date,
    total_amount
FROM {{ ref('stg_orders') }}

-- dbt generates:
-- CREATE OR REPLACE TABLE schema.customer_orders AS (the SELECT above)


-- 2. VIEW: Virtual table
{{ config(materialized='view') }}

SELECT
    order_id,
    customer_id,
    order_date,
    total_amount
FROM {{ ref('stg_orders') }}

-- dbt generates:
-- CREATE OR REPLACE VIEW schema.customer_orders AS (the SELECT above)


-- 3. INCREMENTAL: Only process new/changed rows
{{ config(materialized='incremental', unique_key='order_id') }}

SELECT
    order_id,
    customer_id,
    order_date,
    total_amount
FROM {{ ref('stg_orders') }}
{% if is_incremental() %}
WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}

-- dbt generates (first run):
-- CREATE TABLE schema.customer_orders AS (SELECT * FROM stg_orders)
-- dbt generates (subsequent runs):
-- MERGE INTO schema.customer_orders USING (SELECT ... WHERE order_date > MAX)
-- ON target.order_id = source.order_id
-- WHEN MATCHED THEN UPDATE ...
-- WHEN NOT MATCHED THEN INSERT ...


-- 4. EPHEMERAL: CTE — no physical object
{{ config(materialized='ephemeral') }}

SELECT
    order_id,
    customer_id,
    order_date,
    total_amount
FROM {{ ref('stg_orders') }}

-- dbt generates: Injects the SQL as a CTE in any model that refs this
```

### Comparison

| Materialization | Storage | Compute per Run | Query Performance | Best Use Case |
|---|---|---|---|---|
| **Table** | High (stores data) | High (full rebuild) | Fast (pre-computed) | Final marts, small-medium fact tables |
| **View** | None (virtual) | None (no refresh) | Depends on underlying tables | Lightweight marts, aggregations, joins |
| **Incremental** | High (stores data) | Low (only new rows) | Fast (pre-computed) | Large fact tables, append-heavy data |
| **Ephemeral** | None (CTE) | None (inlined) | Depends on consumer model | Intermediate transformations, staging |

**Production rule of thumb at TB scale:**

| Layer | Recommended Materialization | Rationale |
|---|---|---|
| **Staging** | View or ephemeral | Minimal transformation; no need to materialize |
| **Intermediate** | Ephemeral | Avoids intermediate storage; inlined as CTEs |
| **Marts (small, <10M rows)** | Table | Pre-compute for fast BI queries |
| **Marts (large, >100M rows)** | Incremental | Avoid full rebuilds; scan only new data |
| **Aggregates on top of large marts** | View | If the underlying table is already optimized |

### Pros/Cons

**Table:**
- Pros: Fastest query performance; fully materialized; supports all warehouse features (partitioning, clustering)
- Cons: Storage cost; full rebuild on every run (expensive for large tables); slower pipeline runs

**View:**
- Pros: Zero storage cost; always up-to-date; no rebuild time; easy to maintain
- Cons: Query performance depends on underlying data; can cause expensive cross-database queries; no partitioning on the view itself

**Incremental:**
- Pros: Drastically reduces compute for large tables; only processes new/changed data
- Cons: Complex to implement correctly; requires `unique_key`; stateful (depends on previous run); partial failures can corrupt state

**Ephemeral:**
- Pros: Zero storage; zero rebuild time; SQL is inlined as CTE — single optimized query
- Cons: Every downstream model re-computes the logic; can cause repeated expensive computations; hard to debug (no physical table to query)

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Table model with 500M rows refreshed daily | $200-500/hr compute on Snowflake, pipeline takes 30+ min | Switch to incremental, or partition + cluster |
| View on top of unpartitioned 10TB table | Every query scans 10TB — $100+/query | Ensure underlying tables are partitioned; consider materializing |
| Incremental model with no `is_incremental()` guard | Full rebuild every run (wastes compute) | Always include the conditional filter logic |
| Ephemeral model referenced by 20 downstream models | Each of 20 models re-executes the expensive join | Materialize as a table if the join is expensive and shared |
| Incremental merge strategy on BigQuery without `_PARTITIONTIME` | Merge may scan entire table | Use `insert_overwrite` strategy on BigQuery instead |

### Follow-Up Questions

**Q1. When would you choose a view over a table for a final mart?**
→ When the underlying data is small-to-medium sized and the transformation is simple (a few joins or aggregations). Views are always up-to-date and cost zero storage. At Atlassian, we use views for marts that aggregate data across multiple large fact tables — the warehouse optimizer handles the pushdown efficiently.

**Q2. How does `is_incremental()` work?**
→ It's a built-in dbt macro that returns `true` when the model is running in incremental mode (i.e., the target table already exists) and `false` on the first run. You use it in a `WHERE` clause to filter only new data.

**Q3. Can you change a materialization from view to table without breaking downstream models?**
→ Yes — downstream models use `ref('model_name')` which resolves to the physical object regardless of materialization. However, query performance may change significantly, and you must update any partitioning/clustering configs.

**Q4. What is the performance impact of a long chain of ephemeral models?**
→ Each ephemeral model becomes a CTE in the final query. A chain of 10 ephemeral models becomes a 10-level CTE. Most warehouse optimizers handle this well, but very deep CTE chains can confuse the optimizer. At 5+ levels of ephemeral models, consider materializing an intermediate result.

**Q5. How do materialized views (dbt 1.5+) fit into this?**
→ Materialized views are a hybrid: stored like tables but refreshed incrementally by the warehouse (not by dbt). Use them when the warehouse supports them natively (Snowflake, BigQuery) and the data doesn't change frequently. They require less maintenance than dbt incremental models but offer less control over refresh timing.

**Q6. At what data volume does incremental become worthwhile vs full table refresh?**
→ Generally above 10M rows or ~10GB, depending on the volume of new data vs total data. If only 1% of rows change daily, incremental saves ~99% compute. Below 10M rows, the overhead of merge operations may make full refresh simpler and equally fast.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Making every model a table | Wastes storage, slow pipeline runs | Use views for staging, ephemeral for intermediate |
| Forgetting `is_incremental()` guard | Processes ALL data every run | Always filter with `WHERE` clause using `is_incremental()` |
| Using ephemeral for expensive joins | Every consumer re-computes the join | Materialize shared expensive joins as tables |
| Not setting `unique_key` for incremental | Duplicate rows or failed merges | Always specify a unique key for incremental models |
| Choosing table materialization for a 100TB table | Massive compute cost on every run | Use incremental with appropriate strategy |

### Principal-Level Considerations

- **Cost-aware architecture:** At Principal level, materialization decisions are cost decisions. A 100GB table fully rebuilt daily on Snowflake costs ~$15-30/day in compute. An incremental version processing 1GB/day costs ~$0.15-0.30/day — a 100x difference.
- **Data freshness SLAs:** Materialization affects pipeline end-to-end latency. Views have zero build time but query latency depends on underlying data freshness. Tables require full rebuild time. Incremental offers the best balance for large data.
- **Warehouse-specific optimizations:** On BigQuery, tables should be partitioned by date. On Snowflake, clustering keys matter. On Databricks Delta, Z-order optimization applies. Your materialization strategy must account for these warehouse-specific features.
- **Testing implications:** Views and ephemeral models test faster (no physical rebuild). Table and incremental models test slower (physical operations). Structure your test suite to prioritize fast tests on views and full tests on tables.

---

## Q4. How does incremental modeling work? Explain all incremental strategies (append, delete+insert, merge, insert_overwrite) with examples

### Problem Statement

Deep-dive into incremental models: how they work, the four strategies, when to use each, and the implementation patterns that prevent data corruption at scale.

### Answer

**Incremental models** are the workhorse of production dbt pipelines for large fact tables. Instead of rebuilding the entire table on every run, dbt processes only **new or changed rows** and applies them to the existing table. This reduces compute by orders of magnitude for append-heavy workloads.

The workflow:
1. **First run:** dbt creates the table with all data (full load)
2. **Subsequent runs:** dbt identifies new/changed rows, generates a temporary result set, and applies changes using the configured strategy

### Example

```sql
-- models/marts/fct_orders_incremental.sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    strategy='merge',
    on_schema_change='sync_all_columns'
) }}

-- Source data
WITH new_orders AS (
    SELECT
        order_id,
        customer_id,
        order_date,
        total_amount,
        status
    FROM {{ ref('stg_orders') }}
    {% if is_incremental() %}
    -- Only process orders since the last run
    WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM new_orders
```

**Four Incremental Strategies:**

### Strategy 1: Append (default, simplest)

```sql
-- dbt generates:
INSERT INTO target_table SELECT * FROM new_data

-- Configuration:
{{ config(
    materialized='incremental',
    strategy='append'
) }}
```

| Aspect | Detail |
|---|---|
| **How it works** | Simply appends new rows to the table |
| **Best for** | Append-only data (logs, events, CDC streams) |
| **unique_key needed?** | No |
| **Handles updates?** | No — duplicates possible |
| **Handles deletes?** | No |
| **Warehouse support** | All |

### Strategy 2: Merge (most common, most powerful)

```sql
-- dbt generates (Snowflake):
MERGE INTO target_table AS target
USING incremental_query AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET
    target.total_amount = source.total_amount,
    target.status = source.status
WHEN NOT MATCHED THEN INSERT (
    order_id, customer_id, order_date, total_amount, status
) VALUES (
    source.order_id, source.customer_id, source.order_date, source.total_amount, source.status
)

-- Configuration:
{{ config(
    materialized='incremental',
    unique_key='order_id',
    strategy='merge'
) }}
```

| Aspect | Detail |
|---|---|
| **How it works** | MERGE statement: updates existing, inserts new |
| **Best for** | Data with updates (orders, customers, products) |
| **unique_key needed?** | Yes |
| **Handles updates?** | Yes — matched rows are updated |
| **Handles deletes?** | No (source must include deleted rows with a flag, or use a separate delete strategy) |
| **Warehouse support** | Snowflake, Databricks, Postgres (via upsert), BigQuery (with merge strategy) |

### Strategy 3: Delete + Insert

```sql
-- dbt generates:
DELETE FROM target_table WHERE order_id IN (SELECT order_id FROM incremental_query)
INSERT INTO target_table SELECT * FROM incremental_query

-- Configuration:
{{ config(
    materialized='incremental',
    unique_key='order_id',
    strategy='delete+insert'
) }}
```

| Aspect | Detail |
|---|---|
| **How it works** | Deletes matching rows, then inserts the full new set |
| **Best for** | Data where updates are frequent and MERGE is not supported |
| **unique_key needed?** | Yes |
| **Handles updates?** | Yes (delete + re-insert) |
| **Handles deletes?** | No (same limitation as merge) |
| **Warehouse support** | BigQuery (primary use case), Postgres |

### Strategy 4: Insert Overwrite

```sql
-- dbt generates (BigQuery):
CREATE OR REPLACE TABLE target_table
PARTITION BY DATE(order_date)
CLUSTER BY customer_id AS
SELECT * FROM existing_table
UNION ALL
SELECT * FROM new_data

-- Configuration:
{{ config(
    materialized='incremental',
    strategy='insert_overwrite',
    partition_by={
        "field": "order_date",
        "data_type": "date"
    }
) }}
```

| Aspect | Detail |
|---|---|
| **How it works** | Re-creates the entire partition with existing + new data |
| **Best for** | BigQuery partitioned tables where partition boundaries are clear |
| **unique_key needed?** | No (relies on partition keys) |
| **Handles updates?** | Yes within the partition |
| **Handles deletes?** | Yes within the partition |
| **Warehouse support** | BigQuery only |

### Strategy Comparison

| Strategy | Updates | Deletes | unique_key | Warehouses | Complexity | Performance |
|---|---|---|---|---|---|---|
| **Append** | ❌ | ❌ | No | All | Low | Fast (just INSERT) |
| **Merge** | ✅ | ❌* | Yes | Most | Medium | Fast (single pass) |
| **Delete+Insert** | ✅ | ❌* | Yes | BQ, PG | Medium | Slower (two passes) |
| **Insert Overwrite** | ✅ | ✅ (per partition) | No | BigQuery | High | Varies (partition-sized scan) |

*All strategies except insert_overwrite cannot handle logical deletes unless the source explicitly signals deletions (e.g., `is_deleted = true`).

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Incremental run fails mid-merge (warehouse crash, timeout) | Partial updates — some rows updated, some not | Design idempotent logic; use `--empty` flag to reset; wrap in transaction if supported |
| Incremental run on BigQuery with wrong partition config | Scans all partitions instead of 1 — high cost | Set `require_partition_filter = true` on table |
| Data arrives late (events from yesterday processed today) | Missing data in incremental filter | Use a time buffer: `WHERE event_date > (SELECT MAX(event_date) FROM {{ this }}) - INTERVAL '3 DAYS'` |
| unique_key collisions (two rows with same key in source) | Merge may produce duplicate or fail | Deduplicate in source query: `ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) AS rn WHERE rn = 1` |
| Schema changes (new column added) | Incremental fails — target table has different schema | Use `on_schema_change='sync_all_columns'` or manual migration |

### Follow-Up Questions

**Q1. How do you handle deletes in incremental models?**
→ Most incremental strategies don't handle deletes natively. For soft deletes: include a `is_deleted` flag in the source, filter it in the model, and the merge strategy will handle updates (set `is_deleted=true`). For hard deletes: use a post-hook that deletes rows no longer in the source, or use `insert_overwrite` on BigQuery which replaces entire partitions.

**Q2. What's the cost difference between a full refresh and incremental on a 1TB table?**
→ On Snowflake, a full refresh of 1TB scans ~1TB of data (cost: ~$25-50). Incremental processing 10GB of new data scans ~10GB (cost: ~$0.25-0.50). That's a **50-100x** cost savings. Over a year, this difference is substantial.

**Q3. What happens if `is_incremental()` always returns false?**
→ If the target table doesn't exist (first run, truncated, or renamed), `is_incremental()` returns false, and dbt runs a full load. This is safe — it means the table will be fully rebuilt. The danger is the opposite: if `is_incremental()` always returns true, no new data is ever processed (the filter excludes everything).

**Q4. Can you combine multiple strategies in one model?**
→ No — each model has one strategy. However, you can use conditional logic in Jinja to apply different filters based on a variable. Example: use merge for normal runs, but delete+insert for GDPR compliance runs.

**Q5. How does BigQuery's `insert_overwrite` differ from merge?**
→ `insert_overwrite` replaces entire partitions in one atomic operation. It's more reliable than merge on BigQuery because merges on BigQuery can be slower and have concurrency issues. It handles both inserts and updates within the partition boundary. The trade-off is that you must have a partition key and define clear partitions.

**Q6. How do you handle historically late-arriving data (data that arrives 30 days late)?**
→ Extend the incremental window. Instead of `WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})`, use `WHERE order_date > (SELECT MAX(order_date) FROM {{ this }}) - INTERVAL '30 DAYS'`. This ensures late data is captured while still limiting most scans. Combine with a periodic full refresh for absolute correctness.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| No `is_incremental()` guard | Every run is a full refresh (no benefit) | Add `{% if is_incremental() %} ... {% endif %}` block |
| Missing `unique_key` for merge strategy | Merge fails or produces duplicates | Always set a unique key |
| Using append on data with updates | Duplicate rows accumulate indefinitely | Use merge or delete+insert |
| Not deduplicating source data | unique_key collisions during merge | Add `QUALIFY ROW_NUMBER() OVER (...) = 1` (Snowflake) or `WHERE rn = 1` |
| Overly wide incremental window | Process too much data, negating incremental benefit | Minimize the lookback window (e.g., 1-7 days not 30) |

### Principal-Level Considerations

- **Idempotency is non-negotiable:** At TB scale, pipeline failures happen. Every incremental model must be idempotent — running it twice should produce the same result. This means deduplicating source data, setting proper unique keys, and handling late-arriving data gracefully.
- **Cost observability:** Monitor incremental run costs per model. A poorly configured incremental model that scans too much data can cost more than a full refresh. At Databricks, use the Delta transaction log to verify that incremental runs process only the expected partitions.
- **Data contract enforcement:** With dbt 1.8+ data contracts, the incremental model's schema is contractually enforced. Any schema change upstream must be propagated before the model runs. This prevents silent failures.
- **Disaster recovery:** At Principal level, define a process for recovering from a corrupted incremental table: `dbt run --full-refresh --select fct_orders` to rebuild it from scratch. Schedule periodic full-refreshes (e.g., monthly on a Sunday) to ensure long-term data integrity.

---

## Q5. What are dbt snapshots and how do they differ from SCD Type 2 in traditional data warehousing? Explain check strategy vs timestamp strategy

### Problem Statement

Explain dbt snapshots — the mechanism for tracking historical changes to table records (Slowly Changing Dimension Type 2). Compare to traditional SCD Type 2 implementation, contrast the check and timestamp strategies, and discuss production considerations at scale.

### Answer

**dbt snapshots** implement **SCD Type 2** — they track the full history of changes to rows in a table. When a row's tracked attributes change, the snapshot closes the current record and opens a new one with a `valid_from` / `valid_to` range.

Unlike traditional SCD Type 2 (implemented via stored procedures, triggers, or manual SQL), dbt snapshots are **declarative** — you define which columns to track, and dbt generates the complex merge/update logic automatically.

**Snapshot structure:**

Each snapshot adds four columns to the target table:
- `dbt_scd_id` — Unique identifier for the SCD row (hash of the primary key + valid_from)
- `dbt_updated_at` — When the source row was last updated
- `dbt_valid_from` — When this version of the row became active
- `dbt_valid_to` — When this version was replaced (NULL = current version)

### Example

```sql
-- snapshots/customer_snapshot.sql
{% snapshot customer_snapshot %}

    {{
        config(
            target_schema='snapshots',
            unique_key='customer_id',
            strategy='timestamp',
            updated_at='updated_at',
            invalidate_hard_deletes=False,
            check_cols=['email', 'status', 'tier']
        )
    }}

    SELECT
        customer_id,
        first_name,
        last_name,
        email,
        status,
        tier,
        updated_at
    FROM {{ source('raw_jaffle_shop', 'customers') }}

{% endsnapshot %}
```

**Check strategy vs Timestamp strategy:**

| Strategy | Trigger | Configuration | Best For |
|---|---|---|---|
| **Timestamp** | A dedicated `updated_at` timestamp column | `strategy='timestamp', updated_at='last_modified'` | Tables with reliable `updated_at` timestamps |
| **Check** | Any/all columns being compared (hash of column values) | `strategy='check', check_cols=['col1', 'col2']` or `check_cols='all'` | Tables without timestamps, or when you want to compare all columns |

**How dbt's SCD Type 2 differs from traditional:**

| Aspect | dbt Snapshot | Traditional SCD Type 2 |
|---|---|---|
| **Trigger mechanism** | dbt run (declarative — run the snapshot model) | Stored procedure, trigger, or manual ETL step |
| **Implementation** | dbt generates DDL (MERGE, INSERT, UPDATE) automatically | Written as complex PL/SQL / T-SQL by the developer |
| **History reconstruction** | Automatic: `dbt_valid_from` and `dbt_valid_to` columns | Manual — developer must add date columns |
| **Hard deletes** | By default, snapshot tracks no hard delete (deleted rows remain with `dbt_valid_to` set) | Varies — typically requires a separate CDC process |
| **Overwriting existing rows** | Not allowed — snapshot refuses if source has duplicate `unique_key` | Developer must handle conflict resolution manually |
| **Warehouse support** | Snowflake, BigQuery, Databricks, Postgres adapter-specific | Any warehouse (but implementation is manual) |
| **Incremental performance** | Built-in incremental (only processes changes) | Must be built manually |

### Timestamp Strategy Example

```sql
-- Only tracks changes when `updated_at` changes
{% snapshot products_timestamp_snapshot %}

    {{
        config(
            target_schema='snapshots',
            unique_key='product_id',
            strategy='timestamp',
            updated_at='last_modified_date'
        )
    }}

    SELECT
        product_id,
        product_name,
        category,
        price,
        last_modified_date
    FROM {{ ref('stg_products') }}

{% endsnapshot %}
```

**Generated query (Snowflake, simplified):**

```sql
-- First, get new/updated rows from source
WITH snapshot_query AS (
    SELECT * FROM stg_products
    WHERE last_modified_date > (SELECT MAX(dbt_updated_at) FROM snapshots.products_timestamp_snapshot)
),
-- Close existing records that have changed
UPDATE snapshots.products_timestamp_snapshot
SET dbt_valid_to = snapshot_query.last_modified_date
WHERE dbt_valid_to IS NULL
  AND EXISTS (SELECT 1 FROM snapshot_query WHERE ...);
-- Insert new records
INSERT INTO snapshots.products_timestamp_snapshot
SELECT ..., snapshot_query.last_modified_date, NULL FROM snapshot_query;
```

### Check Strategy Example

```sql
-- Tracks changes in ALL columns
{% snapshot products_check_all_snapshot %}

    {{
        config(
            target_schema='snapshots',
            unique_key='product_id',
            strategy='check',
            check_cols='all'
        )
    }}

    SELECT * FROM {{ ref('stg_products') }}

{% endsnapshot %}
```

```sql
-- Tracks changes only in specific columns (more efficient)
{% snapshot products_check_specified %}

    {{
        config(
            target_schema='snapshots',
            unique_key='product_id',
            strategy='check',
            check_cols=['price', 'category']
        )
    }}

    SELECT * FROM {{ ref('stg_products') }}

{% endsnapshot %}
```

### Pros/Cons

| Aspect | Timestamp Strategy | Check Strategy |
|---|---|---|
| **Performance** | Fast — only checks one timestamp column | Slower — must compare column values |
| **Reliability** | Depends on accurate `updated_at` value | Independent of timestamps — detects ANY change |
| **Column granularity** | Full-row timestamp (all columns at once) | Fine-grained — can select specific columns |
| **False positives** | Timestamp can change without data change | No false positives (compares actual data) |
| **Schema changes** | Robust — adding columns doesn't affect it | Adding columns to `check_cols='all'` increases work |
| **Storage** | Same (reliable timestamp avoids extraneous versions) | Potentially more versions if irrelevant columns change |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Source `unique_key` duplicates | Snapshot fails with unique constraint violation | Deduplicate source first, or use `unique_key` as a composite key |
| Massive transaction log (`check_cols='all'` on 50M rows) | Snapshot runs for hours, may timeout | Switch to timestamp strategy or reduce `check_cols` |
| Hard deletes in source (row removed) | Snapshot keeps old record with `dbt_valid_to = NULL` (current) | Set `invalidate_hard_deletes=True` to close deleted rows |
| Non-append-only load patterns (warehouse full refreshes) | Snapshot may see all rows as "new" — creates many versions | Run snapshot before any destructive load operations |
| Snapshot of a snapshot | Creates SCD Type 3 (tracking tracking) — usually wrong | Never snapshot an already-snapshot table |

### Follow-Up Questions

**Q1. How does `invalidate_hard_deletes` work?**
→ When set to `True`, dbt compares the current set of unique keys in the source vs the snapshot. For any key present in the snapshot but absent in the source, dbt sets `dbt_valid_to = CURRENT_TIMESTAMP`. This closes the record. Default is `False` because hard deletes are rare in production — missing data often means a source error, not an intentional delete.

**Q2. Can you snapshot a view? An incremental model?**
→ Yes to both. Snapshotting an incremental model is common — the incremental model processes new data efficiently, and the snapshot tracks historical changes to the slowly-changing attributes. The snapshot will only see the final state from the incremental model, not the intermediate changes.

**Q3. What happens if `updated_at` has NULL values in the timestamp strategy?**
→ dbt treats NULL as "this row has never been updated." NULL rows will be snapshotted once (on the first run) but never re-processed because `MAX(dbt_updated_at)` from the snapshot will be NULL, and the comparison `source.updated_at > snapshot.dbt_updated_at` evaluates to unknown (NULL). Fix: COALESCE the timestamp in the snapshot query.

**Q4. How do you query a snapshot for a point-in-time view?**
→ 
```sql
-- Get the state as of a specific date
SELECT *
FROM snapshots.customer_snapshot
WHERE '2024-06-01' BETWEEN dbt_valid_from AND COALESCE(dbt_valid_to, '9999-12-31')
```

**Q5. What's the storage impact of snapshots at scale (e.g., 1B rows changing daily)?**
→ Significant. Each change creates a new row. A 100M-customer table with 10% changing daily would generate 10M new rows per day in the snapshot. Over a year, that's 3.65B rows. Implement snapshot retention policies (e.g., archive records where `dbt_valid_to < 90 days ago`) or use more selective `check_cols` to reduce churn.

**Q6. Should you snapshot staging tables or marts tables?**
→ Snapshot staging tables (the raw source). Marts should consume the snapshot for the point-in-time view. This avoids storing duplicate SCD data across multiple layers. If you snapshot a mart, you may inadvertently track changes that are artifacts of the transformation logic, not the source data.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using `check_cols='all'` on wide tables | Every column change triggers a new version — table grows fast | Use timestamp strategy or specific check_cols |
| Not setting `invalidate_hard_deletes` | Deleted rows remain as current forever | Assess whether hard deletes exist; set flag if needed |
| Snapshotting after a full-refresh | Snapshot sees every row as "new" — duplicates history | Snapshot before destructive operations, or use `dbt snapshot` on a stable schedule |
| Missing `unique_key` constraint | Duplicate rows, inconsistent SCD state | Always define a unique key that matches the source's natural key |

### Principal-Level Considerations

- **Snapshot lifecycle management:** At scale, snapshots produce massive data. At Atlassian, implement snapshot archiving — move snapshot data older than 90 days to cold storage. Monitor snapshot storage costs: a snapshot table can grow 100x vs its source over a year.
- **Snapshot scheduling:** Snapshot runs should precede mart model runs. If you build a mart from a snapshot but the snapshot hasn't run yet, the mart shows stale historical snapshots. Use dbt DAG dependencies: `{{ ref('customer_snapshot') }}` ensures proper order.
- **Compliance:** Snapshots are the audit trail. For GDPR/CCPA, you must be able to reconstruct customer state at any point in time. Ensure snapshot retention aligns with compliance requirements (commonly 7 years for financial data).
- **Multi-warehouse portability:** Snapshot implementation varies by adapter. On Snowflake, use MERGE. On BigQuery, use INSERT + UPDATE. On Databricks, use Delta MERGE. Test snapshot behavior when migrating between warehouses.

---

## Q6. Explain dbt macros — how they work, when to use them, and how they differ from SQL functions. Include Jinja templating examples

### Problem Statement

Explain what dbt macros are, how Jinja templating enables code reuse, the difference between SQL functions and dbt macros, and production patterns for macro usage at scale.

### Answer

**dbt macros** are reusable Jinja templates that return SQL strings. They are dbt's mechanism for **code reuse, abstraction, and logic sharing** across models. Macros are defined in `.sql` files under the `macros/` directory.

**Key distinction:** Macros run at **compile time** (when `dbt parse` or `dbt compile` runs), not at query execution time. A macro generates a SQL string that is then sent to the warehouse. SQL functions (e.g., `DATE_TRUNC`, `ROW_NUMBER`) run at **query execution time** inside the warehouse.

**Three categories of macros:**

1. **Utility macros** — Wrap complex SQL logic (pivot, union, date spine)
2. **Configuration macros** — Generate conditional DDL based on target database
3. **Model-building macros** — Generate complete model SQL through programmatic abstraction

### Example

```sql
-- macros/date_spine.sql
{% macro date_spine(start_date, end_date, datepart='day') %}
    {{ return(adapter.dispatch('date_spine', 'dbt_utils')(start_date, end_date, datepart)) }}
{% endmacro %}

-- Usage in a model:
-- SELECT * FROM {{ dbt_utils.date_spine('2024-01-01', '2024-12-31', 'day') }}


-- macros/calculate_discount.sql
{% macro calculate_discount(price, discount_pct) %}
    CASE
        WHEN {{ discount_pct }} >= 1.0 THEN 0  -- guard against 100%+
        ELSE {{ price }} * (1 - {{ discount_pct }})
    END
{% endmacro %}

-- Usage:
-- SELECT
--     product_id,
--     {{ calculate_discount('price', '0.1') }} AS discounted_price
-- FROM stg_products


-- macros/generate_schema_name.sql (override default behavior)
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ default_schema }}_{{ custom_schema_name }}
    {%- endif -%}
{%- endmacro %}


-- macros/clean_string.sql
{% macro clean_string(column_name) %}
    TRIM(
        REGEXP_REPLACE(
            COALESCE({{ column_name }}, ''),
            r'[\n\r\t]+',
            ' '
        )
    )
{% endmacro %}

-- Usage:
-- SELECT {{ clean_string('description') }} AS cleaned_description FROM raw_products
```

### Macros vs SQL Functions

| Aspect | dbt Macro | SQL Function (e.g., UDF) |
|---|---|---|
| **Execution time** | Compile time (before the query runs) | Query execution time |
| **Language** | Jinja (Python-like templating) | SQL or Python (warehouse-specific) |
| **Portability** | Cross-database (via dispatch) | Warehouse-specific (one dialect) |
| **What it returns** | A SQL string (that is then executed) | A computed value per row |
| **Debugging** | Harder — compile the model, check generated SQL | Easier — test with SELECT FUNCTION(...) |
| **Performance** | Zero runtime overhead (already compiled) | Warehouse execution cost |
| **Reusability** | Across models, across databases | Within one database only |
| **Use case** | Generate SQL, loops, conditionals | Transform values row-by-row |

### Pros/Cons

| Advantage | Disadvantage |
|---|---|
| **Code reuse:** Write once, use in 100 models | **Readability:** Over-abstracted macros make models hard to understand |
| **Cross-database compatibility:** Dispatch chooses the right SQL for each warehouse | **Debugging complexity:** Must `dbt compile` to see generated SQL |
| **Conditional logic:** `{% if target.name == 'prod' %}` | **Compile-time issues:** Errors surface late (at compile time, not edit time) |
| **Loops:** `{% for col in column_list %}` | **Performance naivety:** Users assume macros are "free" but the generated SQL must still be efficient |
| **Abstraction:** Hide complex SQL behind a simple name | **Testing burden:** Macros need their own unit tests (dbt 1.5+) |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Macro uses `return()` but returns non-string | dbt raises compilation error | Always wrap in `{{ return(...) }}` and return a SQL string |
| Macro references undefined variable | `UndefinedVariable` error at compile time | Pass all required args explicitly |
| Macro generates SQL with syntax error | Error surfaces only at run time | Unit test macro output with known inputs |
| Recursive macro (calls itself) | Stack overflow at compile time | Avoid recursion; use loops instead |
| Macro called with NULL args | NULL is passed as the string 'None' | Explicitly handle NULL: `{% if arg is not none %}` |

### Follow-Up Questions

**Q1. What is `adapter.dispatch` and why is it important in macros?**
→ `adapter.dispatch` is the dbt mechanism for **cross-database macro resolution**. A macro like `date_spine` may need different SQL for Snowflake vs BigQuery vs Postgres. `dispatch` searches namespaces in order and picks the adapter-specific implementation. Without dispatch, macros are tied to one database dialect.

**Q2. How do you test a macro?**
→ In dbt 1.5+, use unit tests:
```yaml
# tests/unit/calculate_discount.yml
unit_tests:
  - name: test_calculate_discount_normal
    model: macro_discount
    given:
      - input: ref('stg_products')
        rows:
          - {price: 100}
    expect:
      rows:
        - - discounted_price: 90
```
For earlier dbt versions, create a test model that exercises the macro and use `dbt test`.

**Q3. Can macros call other macros?**
→ Yes — deeply. This is the foundation of dbt's package ecosystem. `dbt_utils.date_spine()` calls internal macros for each database adapter. However, deeply nested macro calls (5+ levels) make debugging difficult. Aim for 1-2 levels of macro depth.

**Q4. When should you NOT use a macro?**
→ When the logic is simple and used in 1-2 places — inline SQL is clearer. When the macro abstraction hides expensive operations (e.g., a macro that generates a CROSS JOIN). When the macro is warehouse-specific and you use only one warehouse — a SQL UDF is simpler and faster at runtime.

**Q5. How does Jinja sandboxing work in dbt?**
→ dbt sandboxes Jinja — you cannot use `os`, `subprocess`, or arbitrary Python modules. You can use `env_var()`, `var()`, and basic Python constructs (loops, conditionals, lists, dicts). This is a security measure: models should not have arbitrary system access.

**Q6. What is the `generate_schema_name` macro pattern?**
→ This is a **project-level hook** that defines how dbt assigns schemas to models. The default behavior creates models in `target.schema` (e.g., `prod` → `prod.orders`). Custom overrides can create environment-prefixed schemas (e.g., `prod_analytics`, `dev_analytics`) to isolate environments.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using macros for trivial logic (e.g., `COALESCE`) | Adds indirection without benefit | Write inline SQL |
| Complex loops in macros (`for` over 1000 items) | Generates 1000 blocks of repeated SQL — huge compiled files | Use SQL-level operations (e.g., UNPIVOT) instead of Jinja loops |
| Not using `return()` in utility macros | Macro returns `None` — model fails | Always explicitly `return()` the generated SQL |
| Hardcoding warehouse-specific syntax in macros | Breaks cross-database compatibility | Use `adapter.dispatch` or warehouse-specific conditional blocks |

### Principal-Level Considerations

- **Macro governance:** At scale, establish a macro review process. Macros should be documented, tested, and version-controlled. Overgrown `macros/` directories (>50 macros) indicate a need to split into packages.
- **Performance monitoring:** Macros are free at compile time, but the SQL they generate is not. Review the compiled SQL (`target/compiled/`) to ensure macros don't produce expensive or cross-database-inefficient SQL.
- **Package vs DIY macros:** Before writing a new macro, check if `dbt_utils`, `dbt_expectations`, or `calogica` already provides it. At Google, teams share internal macro packages to avoid duplication and ensure consistency across 100+ model projects.
- **Security:** Macros executed with `env_var()` can expose secrets if the macro is shared outside the team. Never log or return environment variable values from macros.

---

## Q7. What are dbt seeds and when would you use them vs models? What are the limitations?

### Problem Statement

Explain what dbt seeds are, how they work, the use cases, and the significant limitations that make them unsuitable for production-scale reference data.

### Answer

**dbt seeds** are CSV files in the `seeds/` directory that dbt loads into tables in the warehouse. They are loaded via `dbt seed`, which converts CSV rows into database rows in a table named after the CSV file.

**Primary use cases:**
- Small reference/lookup tables (country codes, status mappings, holiday calendars)
- Small mapping tables for testing (dev/test datasets)
- Configuration data that lives in version control (currencies, timezone offsets)

### Example

```csv
# seeds/country_codes.csv
country_code,country_name,region,currency
US,United States,Americas,USD
GB,United Kingdom,Europe,GBP
DE,Germany,Europe,EUR
JP,Japan,Asia,JPY
AU,Australia,Oceania,AUD
```

```sql
-- models/reference/country_lookup.sql
-- Seed can be ref'd like a model:
{{ config(materialized='view') }}

SELECT
    o.order_id,
    o.customer_id,
    o.order_amount,
    cc.country_name,
    cc.region,
    cc.currency
FROM {{ ref('stg_orders') }} o
LEFT JOIN {{ ref('country_codes') }} cc
    ON o.country_code = cc.country_code
```

```bash
# Loading seeds
dbt seed              # Load all seeds
dbt seed --select country_codes  # Load specific seed
dbt seed --full-refresh  # Truncate and reload all seeds
```

### Seeds vs Models

| Aspect | Seeds (CSV) | Models (SQL) |
|---|---|---|
| **Source data** | Static CSV in repo | SQL query against warehouse |
| **Size limit** | ~1M rows (practical limit) | Unlimited (warehouse scale) |
| **Version control** | CSV diff is visible in Git | SQL diff is visible in Git |
| **Refreshing** | Manual (`dbt seed` or `--full-refresh`) | Automated per DAG |
| **Dependency tracking** | Can be `ref()`'d by models | `ref()` each other |
| **Type inference** | Automatic (may be wrong) | Explicit (SQL CAST) |
| **Production use** | Reference data only | All data transformations |

### Limitations

| Limitation | Detailed Explanation | Impact | Mitigation |
|---|---|---|---|
| **Scale** | dbt loads entire CSV into memory, then sends as INSERT to warehouse | Cannot handle >1M rows; memory crash | Use warehouse-native load (COPY INTO, Snowpipe) |
| **Type inference** | dbt scans entire CSV to guess column types (string, int, decimal) | Type mismatches: '0' as boolean, NULL handling, large ints as floats | Explicitly set `column_types` in `dbt_project.yml` |
| **Deployment order** | Seeds must be loaded before any model that `ref()`s them | Pipeline dependency `dbt seed && dbt run` | Ensure CI/CD runs seeds first |
| **No incremental** | Seeds always load the entire file; no change-tracking | Full re-insert on every `dbt seed` run | Use incremental models for production reference data |
| **Encoding issues** | CSV parsing assumes UTF-8; BOM, non-ASCII may fail | Garbled data, failed seed loads | Validate CSV encoding; use BOM detection |
| **Null handling** | Empty cells are interpreted as NULL, but empty string '' also becomes NULL | Data corruption for string columns where '' is valid | Use a sentinel value for empty strings, or handle in model |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| 500MB CSV seed file | dbt loads entire file into memory — OOM on small runners | Split into smaller files; or use SQL model with `UNION ALL` |
| Column type inferred incorrectly (e.g., `zip_code` as int) | Leading zeros stripped: '02134' → 2134 | Explicitly set `column_types` in `dbt_project.yml` |
| Large seed refreshed while models query it | Inconsistent state — models see partial load | Schedule seeds before model runs in CI/CD |
| CSV with commas in data (unquoted) | Parser splits on comma → wrong columns | Properly quote data: `"value, with comma"` |

### Follow-Up Questions

**Q1. How do you configure column types for a seed?**
→ In `dbt_project.yml`:
```yaml
seeds:
  jaffle_shop_analytics:
    country_codes:
      +column_types:
        country_code: varchar(5)
        zip_code: varchar(10)
    product_mapping:
      +column_types:
        product_id: int
        product_name: varchar(255)
```

**Q2. Can a seed reference another seed?**
→ No. Seeds are loaded directly from CSVs; they cannot reference each other. If you need cross-seed logic, create a model that joins seeds.

**Q3. When would a Principal Data Engineer choose a seed over an external table or an SQL model?**
→ Only for **truly static reference data that must be version-controlled with the codebase**: ISO country codes, config flags, small lookup tables shared across teams. For anything that changes weekly or larger than 10K rows, use a model that reads from a shared source (e.g., a dimensional table in the warehouse).

**Q4. How do you handle seed data that changes between environments (dev vs prod)?**
→ Use separate seed CSV files per environment, or use the `--vars` flag to conditionally load different seeds. The cleanest approach is to put environment-specific seed data into a model that reads from an environment-specific table.

**Q5. What happens if a seed column has all NULL values?**
→ dbt infers the type as `TEXT`. Explicitly define the type in `column_types` if you expect a different type and will fill data later.

**Q6. Why not use seeds for a 100MB reference table?**
→ The CSV would be loaded to memory, parsed, and sent as a single large INSERT. This is inefficient and memory-intensive. Use a model that reads from a warehouse table (loaded via COPY) for production-scale reference data.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using seeds for large reference data (100K+ rows) | Memory issues, slow `dbt seed`, Git repo bloat | Use a model reading from a warehouse table |
| Not setting `column_types` for numeric codes | Leading zeros truncated (zip codes, SSNs) | Explicitly define as `varchar` |
| Checking seed CSVs into big Git repos | Binary large objects slow down `git clone` | Use Git LFS for large CSVs, or avoid seeds altogether |
| Relying on seeds for data that changes frequently | Manual `dbt seed` required for every change | Use incremental model for dynamic reference data |

### Principal-Level Considerations

- **Seed as a configuration mechanism:** At Principal level, seeds are the **least important** dbt feature for production. Prefer loading reference data through the warehouse's native import tools (Snowpipe, BigQuery Data Transfer) and modelling it as staging models. Keep seeds only for tiny (<100 row), truly static configs.
- **CI/CD integration:** Seeds must be loaded before models. In production CI/CD:
  ```bash
  dbt seed --select country_codes currency_lookup
  dbt run --select +referencing_models
  ```
- **Column type auditing:** In larger teams, seeds without explicit `column_types` are a source of production bugs. Require all seeds to have explicit column type definitions in `dbt_project.yml`.
- **Alternatives:** For production reference data, consider:
  - Looker / LookML define lookup mappings
  - A `dbt_model` that reads from a `source()` (e.g., a dimensional table)
  - A `dbt_macro` that generates lookup CASE statements
---

## Q8. How does dbt's DAG (Directed Acyclic Graph) work? How do `ref()`, `source()`, and `config()` build the dependency graph?

### Problem Statement

Explain how dbt constructs the execution DAG from model files, how `ref()`, `source()`, and `config()` contribute to the graph, and how the DAG determines execution order, parallelism, and state.

### Answer

dbt builds a **Directed Acyclic Graph (DAG)** by parsing all `.sql` and `.py` model files at compile time. Each node in the DAG represents a model (or test, or snapshot). Each edge represents a dependency — a `ref()` or `source()` call from one model to another.

**How the DAG is built:**

1. **Parse phase:** dbt scans each model file for `{{ ref('model_name') }}`, `{{ source('source_name', 'table_name') }}`, and `config(...)` blocks.
2. **Graph construction:** Every `ref()` creates an edge from the current model to the referenced model. Every `source()` creates an edge from the current model to a source node (defined in `sources.yml`).
3. **Topological sort:** dbt sorts the DAG topologically — models are executed in dependency order. If model B depends on model A, A runs first.
4. **Parallelism:** Models at the same depth level (no dependencies on each other) run in parallel, up to `threads` limit.
5. **Execution:** dbt walks the sorted graph, running models and recording success/failure. Failures are cached so downstream models are skipped.

**The three core DAG-building functions:**

| Function | What it creates | SQL Generated | Example |
|---|---|---|---|
| `{{ ref('stg_orders') }}` | Edge to another model | `FROM "PROD_DB"."PROD_SCHEMA"."STG_ORDERS"` (with schema/database resolution) | `SELECT * FROM {{ ref('stg_orders') }}` |
| `{{ source('jaffle_shop', 'customers') }}` | Edge to a source node | `FROM "JAFFLE_DB"."RAW_JAFFLE"."CUSTOMERS"` | `SELECT * FROM {{ source('raw_jaffle_shop', 'customers') }}` |
| `config(materialized='view')` | Node metadata (materialization, schema, tags) | Generates CREATE VIEW or CREATE TABLE as DDL | `{{ config(materialized='table') }}` |

### Example

```sql
-- models/staging/stg_orders.sql
{{ config(materialized='view') }}

SELECT
    id AS order_id,
    customer_id,
    order_date,
    amount
FROM {{ source('jaffle_shop', 'orders') }}
```

```sql
-- models/marts/fct_orders.sql
{{ config(materialized='table') }}

SELECT
    o.order_id,
    c.customer_name,
    o.order_date,
    o.amount,
    CASE WHEN o.amount > 100 THEN 'high' ELSE 'standard' END AS order_tier
FROM {{ ref('stg_orders') }} o
LEFT JOIN {{ ref('dim_customers') }} c
    ON o.customer_id = c.customer_id
```

```sql
-- models/marts/dim_customers.sql
{{ config(materialized='table') }}

SELECT
    customer_id,
    customer_name,
    signup_date
FROM {{ ref('stg_customers') }}
```

**Resulting DAG (simplified):**
```
source:jaffle_shop.orders → stg_orders → fct_orders
source:jaffle_shop.customers → stg_customers → dim_customers → fct_orders
```

**Execution order:** `stg_orders`, `stg_customers` (parallel), then `dim_customers`, then `fct_orders`.

### DAG Visualization Commands

```bash
# Terminal text output
dbt ls --resource-type model  # List all nodes
dbt ls --select +stg_orders+  # Show upstream + downstream of a model

# Graphviz visualization
dbt docs generate
dbt docs serve  # Browser: green = model, blue = source, red = test

# YAML export (for custom tooling)
dbt ls --output json --resource-type model --selector some_selector
```

### Pros/Cons

| Advantage | Disadvantage |
|---|---|
| **Automatic dependency resolution** — no manual ordering | **Implicit graph** — dependencies from `ref()` in macros may be missed |
| **Parallel execution** — models at same depth run concurrently | **Deadlock detection** — only at compile time; a cycle causes bailout |
| **Selective execution** — `+model_name+` picks full DAG subset | **Graph memory** — very large projects (10K+ models) strain memory |
| **Lineage documentation** — `dbt docs` gives end-to-end lineage | **Circular dependencies** — `ref()` loops are caught only at compile (rare) |
| **Test integration** — tests follow the same DAG as models | **Cross-project refs** need dbt 1.6+ and explicit project dependencies |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Circular dependency (A refs B, B refs A) | dbt raises `CompilationError: circular dependency` | Break the cycle — create intermediate model or use `source()` for one side |
| Missing `ref()` in a macro argument | Dependency not registered — model runs out of order, may fail | Ensure every `ref()` is explicit in model body or use `depends_on` config |
| Model `ref()`s a non-existent model | dbt raises `CompilationError: model not found` | Check spelling, ensure model exists in DAG or install correct package |
| `source()` defined in YAML but not in any model | Source appears in `dbt docs` but no edge — orphan source | Add `source()` to at least one model |
| Cross-project ref with incompatible versions | Ref fails at compile time | Ensure both projects use compatible dbt versions (1.6+ for cross-project) |

### Follow-Up Questions

**Q1. How does `ref()` resolve to the correct table at runtime?**
→ dbt replaces `{{ ref('stg_orders') }}` with the fully-qualified name (`database.schema.table`) based on the target environment. In dev, it might resolve to `dev_dbt.stg_orders`. In prod, it resolves to `prod.stg_orders`. The resolution is controlled by the `generate_ref_name` macro or the default `{{ database }}.{{ schema }}.{{ identifier }}`.

**Q2. What is a "selector" and how does it interact with the DAG?**
→ Selectors (YAML in `selectors.yml`) define named filter expressions based on DAG properties: tags, fqn, resource type, etc. Example: `selectors: [name: "daily_run", definition: {union: [{tag: "daily"}, {tag: "core"}]}]`. Use: `dbt run --selector daily_run`.

**Q3. How do you handle a model that depends on a model in another dbt project?**
→ In dbt 1.6+, use cross-project references. In `dependencies.yml`:
```yaml
packages:
  - project: https://github.com/org/analytics-project
    version: ">=1.0.0"
```
Then use `{{ ref('analytics', 'fct_orders') }}` from the calling project.

**Q4. What happens to the DAG when a model fails?**
→ dbt marks that node as "error" and skips all downstream nodes that depend on it. Other branches of the DAG continue. This is managed by the `--defer` flag (if using state, upstream model can be reused from previous run).

**Q5. How does dbt decide which models to run when you use `--select`?**
→ `--select` uses a graph selector. `+model_name` selects the model and all upstream (ancestors). `model_name+` selects the model and all downstream (descendants). `+model_name+` selects ancestors + the model + descendants. These are combined with `,` (intersection) or ` ` (union).

**Q6. What is a "disconnected" model — one with no `ref()` or `source()` calls?**
→ A model with zero dependencies becomes a root node in the DAG — it runs first (alongside other root nodes). This is unusual for most models but valid for utility models or full-refresh-only aggregations.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using `source()` for a table that is also a model | Creates duplicate dependency edges; confusion in lineage | Use `ref()` for models, `source()` only for raw ingestion tables |
| Forgetting `{{ ref() }}` and using a hardcoded table name | dbt doesn't register the dependency → model runs in wrong order, fails | Always use `{{ ref() }}` — never hardcode table names |
| Creating circular dependencies via `config(depends_on=...)` | Compile error | Break the cycle — refactor logic or flatten the dependency |
| Adding a model that depends on nothing but a `source()` not defined in YAML | Source node doesn't exist; dbt raises source not found | Define the source in `schema.yml` files first |

### Principal-Level Considerations

- **Graph partitioning for scale:** At Uber scale (thousands of models), the DAG must be partitioned into sub-graphs. Use tags (`tag: hourly`, `tag: daily`, `tag: weekly`) to segment the DAG and run different DAG slices at different cadences. Monitor graph parse time — slow parse (>5 min) indicates too many nodes.
- **Selective CI/CD:** Use DAG selectors in CI/CD to run only impacted downstream models. When a PR modifies only `stg_orders`, run `dbt run --select stg_orders+` to test only downstream models. This reduces CI time from hours to minutes.
- **Defer for production stability:** In production, use `--defer --state target/prod/` to run only changed models and their downstream impacts, reusing pre-built upstream artifacts. This is the foundation of dbt's production deployment strategy.
- **Cross-team ownership:** With 100+ data engineers, the DAG must be modular. Define explicit `group` ownership per model group. Use dbt 1.8+ groups and access controls to restrict who can modify specific DAG segments.

---

## Q9. What is dbt testing? Explain the difference between singular and generic tests, where they live, and how to write custom tests

### Problem Statement

Explain dbt's testing framework — the two test types (singular and generic), how they are defined, how they run, and how to write custom tests for production data quality at scale.

### Answer

**dbt tests** are assertions about the data in your warehouse. They are not unit tests of the transformation logic (pre-1.5, at least); they are **data quality checks** that verify model outputs meet expectations.

**Two types of tests:**

1. **Singular tests** — One `.sql` file per test, asserted as a query that returns zero rows when passing. The file is placed in `tests/` and each file is one test.

2. **Generic tests** — Parameterized test logic (macros) defined in `macros/` or in a package. They are applied inline in `schema.yml` files with arguments. Built-in: `unique`, `not_null`, `accepted_values`, `relationships`. Custom ones can be written.

### Example

**Singular test:**

```sql
-- tests/assert_positive_order_amount.sql
-- Fails if any order amount is negative or zero
SELECT
    order_id,
    amount
FROM {{ ref('stg_orders') }}
WHERE amount <= 0
```

```bash
# dbt output when test passes:
PASS = 1, WARN = 0, ERR = 0, SKIP = 0

# dbt output when test fails:
FAIL = 1 (assert_positive_order_amount — returns 5 rows)
```

**Generic test (built-in, applied in YAML):**

```yaml
# models/staging/schema.yml
version: 2

models:
  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: status
        tests:
          - accepted_values:
              values: ['placed', 'shipped', 'completed', 'returned']
  - name: fct_orders
    tests:
      - dbt_utils.expression_is_true:
          expression: "amount >= 0"
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
          - relationships:
              to: ref('dim_customers')
              field: customer_id
```

**Custom generic test:**

```sql
-- macros/tests/is_positive.sql
{% test is_positive(model, column_name) %}

    SELECT *
    FROM {{ model }}
    WHERE {{ column_name }} <= 0

{% endtest %}
```

```yaml
# Usage in schema.yml
models:
  - name: fct_orders
    columns:
      - name: amount
        tests:
          - is_positive
```

**Custom generic test with arguments:**

```sql
-- macros/tests/column_value_between.sql
{% test column_value_between(model, column_name, min_value, max_value) %}

    SELECT *
    FROM {{ model }}
    WHERE {{ column_name }} < {{ min_value }}
       OR {{ column_name }} > {{ max_value }}

{% endtest %}
```

```yaml
# Usage
columns:
  - name: discount_pct
    tests:
      - column_value_between:
          min_value: 0
          max_value: 1
```

### Singular vs Generic Tests

| Aspect | Singular Test | Generic Test |
|---|---|---|
| **Definition** | One `.sql` file per test in `tests/` | Parameterized macro, applied in YAML |
| **Parameterization** | None — each test is hardcoded for one model/column | Highly parameterized — reuse across models |
| **Reusability** | Low — copy-paste across models | High — declare once, apply many times |
| **Maintenance** | High (100 tests = 100 files) | Low (one macro, many declarations) |
| **When to use** | Complex logic (multiple tables, window functions) | Simple column-level checks (unique, not_null, accepted_values) |
| **Discovery** | dbt scans `tests/` directory automatically | Applied manually in `schema.yml` |

### Pros/Cons

| Pro | Con |
|---|---|
| **Declarative** — test passes if query returns 0 rows | **Not unit tests** — tests run post-execution, not during development |
| **DAG integration** — tests run after their model(s) | **No test isolation** — one test fail can cascade (tests depend on model outputs) |
| **Parallel execution** — independent tests run concurrently | **No built-in test coverage report** — must use `dbt-coverage` or custom |
| **Native to dbt** — no external test runner needed | **No "warning" severity** — only pass/fail (workaround: `limit > 0` returns rows as warning) |
| **CI/CD ready** — `dbt test --select model_name` integrates with pipelines | **Large test suites** — slow if 1000+ tests; need parallel threads |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| `unique` test on a NULL column | Passes — NULLs are considered distinct | Add `not_null` test first (test order is independent but logic is sequential) |
| `relationships` test with mismatched data types | Fails — join error, not data error | Ensure foreign key and primary key columns have same data type |
| Test filter `--select model_name` misses downstream tests | Tests on models that depend on the model are not run | Use `--select model_name+` to include downstream tests |
| Custom test returns 0 rows due to SQL error (not data) | Test passes (false negative) | Use dbt 1.5+ unit tests or `dbt test --store-failures` to log results |
| Large test on TB-scale table | Full table scan — slow and expensive | Add WHERE clause to test query (e.g., `WHERE order_date >= CURRENT_DATE - 7`) |

### Follow-Up Questions

**Q1. What is the difference between a dbt test and a unit test (dbt 1.5+)?**
→ Pre-1.5, dbt tests are only data quality checks on model outputs. dbt 1.5+ introduced **unit tests** that test transformation logic (SQL-to-SQL) using mock inputs. Unit tests verify that a model's logic is correct; data quality tests verify the output data meets expectations. They serve different purposes: unit tests catch logic bugs, data tests catch data errors.

**Q2. How do you test a macro?**
→ Two approaches:
- Pre-1.5: Create a model that exercises the macro, then test that model.
- dbt 1.5+: Use `unit_tests` for macros directly (see Q6 macro section for example).

**Q3. What is `--store-failures` and when should you use it?**
→ `dbt test --store-failures` persists test results to a table (`test_audit` or `dbt_test__audit`). Use in CI/CD to audit test history and review failing rows. Not for production (adds write load) — use only in CI or dev.

**Q4. How do you test a model that depends on data that changes daily (e.g., a date spine)?**
→ Use a singular test that queries the current date:
```sql
-- tests/assert_date_spine_is_current.sql
SELECT *
FROM {{ ref('date_spine_model') }}
WHERE date_col = CURRENT_DATE  -- Should have today's row
```
This test adapts to changing data without hardcoding dates.

**Q5. How do you write a custom test that checks row count thresholds (e.g., "row count must be > 0")?**
```sql
-- macros/tests/row_count_threshold.sql
{% test row_count_threshold(model, min_count, max_count=None) %}

    WITH row_count AS (
        SELECT COUNT(*) AS cnt FROM {{ model }}
    )
    SELECT cnt
    FROM row_count
    WHERE cnt < {{ min_count }}
       {% if max_count %} OR cnt > {{ max_count }} {% endif %}

{% endtest %}
```

**Q6. How do tests integrate with dbt's freshness checks?**
→ Tests and source freshness (`dbt source freshness`) are independent. Freshness checks verify that source data was updated recently. Tests verify data content quality. Both are part of dbt's overall data quality framework but address different concerns.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Not adding tests to the `schema.yml` for staging models | Raw data quality issues propagate to marts | Always test staging models first (not_null, unique on PKs) |
| Writing a singular test that duplicates a generic test | Duplication, harder maintenance | Prefer generic tests; use singular only for complex logic |
| Running all tests with no `--select` filter on TB-scale models | Expensive full scans, long run times | Run tests selectively: `dbt test --select model_name+` |
| Ignoring failed tests in CI/CD | Tests reduce quality — treating them as optional | Make tests part of CI/CD pipeline with `--warn-error` to fail CI on failure |

### Principal-Level Considerations

- **Test strategy at scale:** Implement a **tiered test approach**: Tier 1 (must-pass): `unique`, `not_null` on marts PKs. Tier 2 (should-pass): `accepted_values`, `relationships` on facts. Tier 3 (informational): row count thresholds, freshness. Run Tier 1 in every CI, Tier 2 nightly, Tier 3 weekly.
- **Cost-aware testing:** At Uber scale, avoid full-table-scan tests on TB-scale models. Use `--store-failures` to persist test results and only scan changed partitions. Implement `where` clauses in tests: `tests: - unique: where: "order_date >= CURRENT_DATE - 30"`.
- **Test observability:** Export test results to a monitoring system (Datadog, PagerDuty). Alert on Tier 1 test failures immediately. Use `dbt test --store-failures` to populate an audit table and build a dbt dashboard over it.
- **Data contract enforcement:** With dbt 1.8 data contracts, tests become part of the contract. A failing test on a published model should block downstream consumers. This is the foundation of data mesh — each data product must pass its contract's tests.

---

## Q10. What is source freshness in dbt? How does it work and how do you set up a production freshness SLA?

### Problem Statement

Explain dbt's `source freshness` feature — how it tracks when source data was last updated, how to configure SLAs (Service Level Agreements), and how to operationalize freshness monitoring at scale.

### Answer

**Source freshness** in dbt is a mechanism to verify that source tables were loaded recently. It's defined in `sources.yml` and run with `dbt source freshness`. For each source table, you specify a `freshness` block that declares:

- `warn_after` — If data is older than this, emit a WARNING
- `error_after` — If data is older than this, emit an ERROR (non-zero exit code)
- `loaded_at_field` — The timestamp column that indicates when the row was loaded (usually `_loaded_at`, `_sync_timestamp`, `ingested_at`)

**How it works:**
1. dbt queries each source table for `MAX(loaded_at_field)`.
2. dbt compares the timestamp to the current time.
3. If the difference exceeds `warn_after`, a warning is logged.
4. If it exceeds `error_after`, an error is raised (exit code 1).

### Example

```yaml
# models/staging/sources.yml
version: 2

sources:
  - name: jaffle_shop
    database: JAFFLE_DB
    schema: RAW_JAFFLE
    tables:
      - name: orders
        loaded_at_field: _etl_loaded_at
        freshness:
          warn_after:
            count: 2
            period: hour
          error_after:
            count: 6
            period: hour
        columns:
          - name: id
            tests:
              - unique
              - not_null

      - name: customers
        loaded_at_field: _etl_loaded_at
        freshness:
          warn_after:
            count: 24
            period: hour
          error_after:
            count: 72
            period: hour

      - name: historical_products
        # No freshness defined — will not be checked
```

**Running freshness:**

```bash
# Check all sources
dbt source freshness

# Check specific source
dbt source freshness --select source:jaffle_shop

# Filter by tag
dbt source freshness --select tag:critical_sla

# Output example:
# Freshness of jaffle_shop.orders loaded at field _etl_loaded_at
#   - WARN: max is 2024-06-15 14:30:00, warn after 2 hours, error after 6 hours
#   - PASS: max is 2024-06-15 16:00:00 (fresh within 2 hours)
#   - ERROR: max is 2024-06-15 09:00:00 (stale — 7+ hours ago)
```

### JSON Output for Operational Use

```bash
dbt source freshness --output json --select source:jaffle_shop | ConvertFrom-Json
```

```json
{
  "meta": {"status": "error"},
  "results": [
    {
      "node": {"unique_id": "source:jaffle_shop.orders", "name": "orders"},
      "status": "pass",
      "max_loaded_at": "2024-06-15T14:30:00Z",
      "snapshotted_at": "2024-06-15T16:00:00Z",
      "max_loaded_at_ago_seconds": 5400,
      "criteria": {"warn_after": {"count": 2, "period": "hour"}, "error_after": {"count": 6, "period": "hour"}}
    }
  ]
}
```

### Pros/Cons

| Pro | Con |
|---|---|
| **Declarative SLA definitions** — in source YAML, not ETL tool | **Requires a `loaded_at_field`** — not all sources have one |
| **DAG integration** — freshness checks run in `dbt build` pipeline | **Single timestamp per table** — doesn't track per-partition freshness |
| **JSON output** — integrates with monitoring tools (Datadog, PagerDuty) | **Time-zone sensitive** — loaded_at_field must be UTC or config must match |
| **Selectively skippable** — sources without freshness block are ignored | **No built-in alerting** — dbt outputs exit code; you need external tooling to alert |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| `loaded_at_field` is NULL for all rows | dbt cannot calculate freshness — warns/errors immediately | Ensure the field is never NULL; populate with `CURRENT_TIMESTAMP` in load |
| Source table is empty (0 rows) | dbt treats this as "no data loaded" — errors immediately | Use a health-check source table that always has at least one row |
| Clock skew between warehouse and dbt runner | Freshness calculated incorrectly (seems older or newer than reality) | Use NTP-synchronized clocks; warehouse time takes precedence |
| Time zone mismatch (loaded_at in EST, dbt in UTC) | Freshness hours are off by 5 | Always use UTC for loaded_at fields, or explicitly set time zone |
| Source table has no `loaded_at_field` (historical snapshot with no timestamp) | Freshness cannot be defined | Add a `_dbt_loaded_at` timestamp to the staging model, or skip freshness for that source |

### Follow-Up Questions

**Q1. Can freshness be used for partitioned tables (e.g., BigQuery ingestion-time partitions)?**
→ Yes — but freshness checks the entire table's `MAX(loaded_at)`, not per-partition. For partitioned tables, define `loaded_at_field` as `_PARTITIONTIME` (BigQuery). To check per-partition freshness, you need a custom model that runs a freshness check per partition.

**Q2. How do you set up alerting from `dbt source freshness`?**
→ Integrate with monitoring:
```bash
dbt source freshness --output json > freshness_result.json
$exit_code = $LASTEXITCODE
if ($exit_code -ne 0) {
    # Parse JSON, send to PagerDuty / Datadog
    $results = Get-Content freshness_result.json | ConvertFrom-Json
    foreach ($r in $results.results) {
        if ($r.status -eq "error") {
            Send-Event -Source "dbt_freshness" -Severity Error -Message "$($r.node.name) is stale"
        }
    }
}
```

**Q3. What is the difference between `dbt source freshness` and `dbt test` on sources?**
→ `dbt source freshness` checks timeliness (how recent is the data). Tests on sources check data quality (nulls, uniqueness). They are complementary: fresh data can be bad data, and good data can be stale.

**Q4. How do you handle a source that loads only on weekdays (no weekend load)?**
→ Set `error_after` to 72 hours (covers Friday→Monday gap):
```yaml
freshness:
  warn_after:
    count: 48  # 2 days — may warn Monday morning for Friday load
    period: hour
  error_after:
    count: 72  # 3 days — Monday load within window
    period: hour
```

**Q5. Can you run freshness as part of `dbt build`?**
→ No — `dbt build` runs models, tests, and snapshots, but not source freshness. You must run `dbt source freshness` separately. This is by design: freshness checks the source, not a model output.

**Q6. What is the `snapshotted_at` field in freshness output?**
→ It's the timestamp when dbt ran the freshness query. Combined with `max_loaded_at`, it tells you how stale the data is relative to the check time: `max_loaded_at_ago_seconds` = `snapshotted_at - max_loaded_at`.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Not defining `loaded_at_field` | Freshness check returns no timestamp (always stale) | Always declare the timestamp column |
| Setting `error_after` tighter than actual load schedule | False alerts — team ignores alerts | Set SLA based on actual load cadence, not desired cadence |
| Assuming freshness in dev is the same as prod | Dev sources may be stale by design | Run freshness only in prod; skip in dev with `--exclude source:*` |
| Not monitoring freshness results | SLA is defined but never checked | Integrate freshness into CI/CD or monitoring tool |

### Principal-Level Considerations

- **SLA tiering:** Define SLA tiers: **Tier 1** (revenue-critical — warn 30 min, error 1 hour), **Tier 2** (operational — warn 2 hours, error 6 hours), **Tier 3** (analytical — warn 12 hours, error 24 hours). Sources loaded by batch ETL at 2 AM daily should not have a 30-minute SLA.
- **Compute cost:** Running freshness on 50 sources means 50 queries. At scale, this triggers warehouse costs. Optimize by running freshness checks only on sources with active downstream models (use `dbt source freshness --select source:critical`).
- **SLA compliance dashboard:** Build a dbt model that reads `dbt_source_freshness_results` (stored as a table or snowplow event) and visualize in a BI tool. This gives business stakeholders visibility into data timeliness SLAs.
- **Cross-team ownership:** Each source should have a designated owner. When freshness fails, the owner (not the data consumer) gets the alert. This is essential for data mesh architectures where sources are owned by domain teams.

---

## Q11. How do dbt snapshots work? Explain SCD Type 1 vs Type 2 and the underlying mechanics

### Problem Statement

Explain dbt snapshots — how they capture point-in-time state of source data, the difference between SCD Type 1 (overwrite) and SCD Type 2 (history tracking), and the underlying SQL mechanics.

### Answer

**dbt snapshots** capture the state of a source table at a point in time. Unlike models (which transform data), snapshots preserve historical state — essential for auditing, time-travel analysis, and accurate historical reporting.

**Two strategies:**

| Strategy | Behavior | Use Case |
|---|---|---|
| **SCD Type 1** (`strategy='overwrite'`) | Updates changed rows in-place — no history | When you only need the latest state |
| **SCD Type 2** (`strategy='check'`) | Preserves all history with `valid_from`/`valid_to` | When you need to answer "what did this row look like on X date?" |

### SCD Type 1 (Overwrite)

```yaml
# models/snapshots/customer_snapshot.yml
version: 2

snapshots:
  - name: customer_snapshot
    config:
      strategy: overwrite
      target_schema: SNAPSHOT_SCHEMA
    columns:
      - name: customer_id
      - name: email
      - name: phone
```

```sql
-- Underlying SQL (simplified)
-- 1. Find rows that changed
-- 2. DELETE old versions
-- 3. INSERT new versions
-- Result: snapshot table always mirrors current source state
```

### SCD Type 2 (Check — History Tracking)

```yaml
# models/snapshots/customer_history.yml
version: 2

snapshots:
  - name: customer_history
    config:
      strategy: check
      unique_key: customer_id
      check_cols:
        - email
        - phone
      updated_at: _dbt_updated_at
      valid_from_time: _dbt_valid_from
      valid_to_time: _dbt_valid_to
      target_schema: SNAPSHOT_SCHEMA
    columns:
      - name: customer_id
      - name: email
      - name: phone
      - name: _dbt_updated_at
      - name: _dbt_valid_from
      - name: _dbt_valid_to
```

```sql
-- Underlying SCD2 SQL (simplified)
-- 1. Identify rows where check_cols changed since last snapshot
-- 2. Set _dbt_valid_to = CURRENT_TIMESTAMP on old versions
-- 3. INSERT new version with _dbt_valid_from = CURRENT_TIMESTAMP

WITH diff AS (
    SELECT
        s.customer_id,
        s.email,
        s.phone,
        CURRENT_TIMESTAMP AS _dbt_valid_to
    FROM snapshot_table s
    INNER JOIN source_table n ON s.customer_id = n.customer_id
    WHERE s.email != n.email OR s.phone != n.phone
)
UPDATE snapshot_table
SET _dbt_valid_to = diff._dbt_valid_to
FROM diff WHERE snapshot_table.customer_id = diff.customer_id

INSERT INTO snapshot_table
SELECT n.customer_id, n.email, n.phone,
       CURRENT_TIMESTAMP AS _dbt_updated_at,
       CURRENT_TIMESTAMP AS _dbt_valid_from,
       NULL AS _dbt_valid_to
FROM source_table n
LEFT JOIN snapshot_table s ON n.customer_id = s.customer_id AND s._dbt_valid_to IS NULL
WHERE n.email != s.email OR n.phone != s.phone
```

**Valid-time query example:**
```sql
-- What was customer 42's email on 2024-01-15?
SELECT email FROM customer_history
WHERE customer_id = 42
  AND '2024-01-15' BETWEEN _dbt_valid_from AND COALESCE(_dbt_valid_to, '9999-12-31')
```

### `check` vs `timestamp` Strategies (SCD2)

| Strategy | How it detects changes | Column needed | Pros | Cons |
|---|---|---|---|---|
| `check` | Compares specified columns for equality | `check_cols: [col1, col2]` | Precise — only relevant columns trigger history | Slower — row-by-row column comparison |
| `timestamp` | Any row with `updated_at` > last snapshot time | `updated_at: updated_at` | Fast — single column check | Over-sensitive — minor updates create new versions |

### Example

**Source table (changes over time):**
| customer_id | email | phone |
|---|---|---|
| 1 | alice@a.com | 555-0001 |
| 2 | bob@b.com | 555-0002 |

**After customer 1 changes email → `alice2@a.com`:**
| customer_id | email | phone | _dbt_valid_from | _dbt_valid_to |
|---|---|---|---|---|
| 1 | alice@a.com | 555-0001 | 2024-01-01 00:00 | 2024-06-15 10:00 |
| 1 | alice2@a.com | 555-0001 | 2024-06-15 10:00 | NULL ← current version |
| 2 | bob@b.com | 555-0002 | 2024-01-01 00:00 | NULL ← no change |

### Pros/Cons

| Pro | Con |
|---|---|
| **Declarative** — YAML config, dbt generates all SQL | **Table grows indefinitely** — SCD2 history is unbounded |
| **Database-agnostic** — works on Snowflake, BigQuery, Redshift, Postgres | **No built-in partitioning control** — must set it via `config` |
| **Built-in valid-time columns** — `_dbt_valid_from`, `_dbt_valid_to` | **Full table scan on each run** — no delta detection in source |
| **Incremental by default** — only changed rows are processed | **Not suited for high-churn data** — e.g., session data with millions of daily changes |
| **Can snapshot from a model** — not just raw sources | **Snowflake limitations** — `MERGE` syntax requires specific configuration |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Source table grows to 10B rows | Full table scan on every snapshot run — prohibitively slow | Snapshot from a staging model with daily partitions, not the raw source |
| `_dbt_valid_to` set to same timestamp for multiple rows | Invalid history — overlapping valid-time windows | Ensure atomic transactions; check dbt snapshot SQL for race conditions |
| `check_cols` includes a NULL-able column | `NULL != NULL` is false → change missed | Use `COALESCE` in custom snapshot strategy or use `timestamp` strategy |
| Snapshot runs twice in same second | Both versions get same `_dbt_valid_from` → ambiguous history | Use database-specific timestamp (e.g., Snowflake `CURRENT_TIMESTAMP` with microsecond precision) |
| `unique_key` not unique in source | Snapshot fails or produces duplicate history rows | Add uniqueness test on source; fix upstream ETL |

### Follow-Up Questions

**Q1. What is the `invalidate_hard_deletes` config?**
→ When set to `'dbt_valid_to'`, rows that are deleted from the source but existed in the snapshot get their `_dbt_valid_to` set to the current time. This records that the row was deleted, preserving the full history including removal.

**Q2. How do you partition a snapshot table for performance?**
→ Add `config: partition_by: {'time': 'order_date', 'data_type': 'day'}`. For BigQuery, use `partition_by: 'date_col'`. For Redshift, use `partition_by: {'type': 'date', 'args': ['order_date']}`. This is adapter-specific.

**Q3. Can you snapshot a model (not just a source)?**
→ Yes — `{{ ref('stg_orders') }}` is valid inside a snapshot. This is common when you need a history of cleaned/staged data.

**Q4. What happens if a snapshot target table already exists?**
→ dbt appends/merges — it does not drop and recreate. This is essential for maintaining history.

**Q5. How do you handle a full-refresh of a snapshot (e.g., after a schema change)?**
→ Run `dbt run-operation drop_snapshot --args "{{ var('snapshot_name') }}"` or use `dbt snapshot --full-refresh` to recreate the snapshot table from scratch.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Snapshotting a 10B-row raw table | Full scan on every run — cost explodes | Snapshot from a filtered staging model or limit by date |
| Not using `invalidate_hard_deletes` for SCD2 | Deleted rows remain as "current" forever | Always enable: `invalidate_hard_deletes: 'dbt_valid_to'` |
| Forgetting `unique_key` on SCD2 | dbt cannot identify which row to update | Always set `unique_key` — this is the PK of the source |
| Using `timestamp` strategy on a table where `updated_at` changes every sync | Creates a new history row on every run even when no real data changed | Use `check` strategy with only business-meaningful columns |

### Principal-Level Considerations

- **Storage management:** SCD2 tables grow without bounds. Implement retention policies: archive old snapshot history to cold storage (Snowflake Time Travel + Fail-safe, BigQuery long-term storage) and trim active table to last N years.
- **Cost control:** Snapshot runs are the most expensive dbt operations. Partition snapshots by load time, use incremental merge strategies, and run only on high-value tables. Budget snapshot compute separately in your data cost model.
- **Data mesh alignment:** In a data mesh, each domain owns its source snapshots. The snapshot is the domain's "historical data product" — consumers query it for time-travel analysis. Domain teams are responsible for SLAs on snapshot freshness and completeness.

---

## Q12. What are dbt hooks? Explain `pre_hook` and `post_hook` — when to use them and production examples

### Problem Statement

Explain dbt hooks — how to run custom SQL before/after model execution, their lifecycle in the build process, and production-level use cases.

### Answer

**dbt hooks** allow you to execute raw SQL before (`pre_hook`) or after (`post_hook`) a model's compiled SQL runs. They are defined in `config()` blocks and execute in the same transaction (unless you disable it).

**Hook lifecycle per model:**
```
pre_hook 1 → pre_hook 2 → ... → MODEL_SQL → ... → post_hook N → post_hook 1
```
Pre-hooks run in order. Post-hooks run in reverse order (LIFO — like SQL transaction management).

### Example

**Transaction control (default behavior):**
```sql
-- models/marts/fct_orders_atomic.sql
{{ config(
    materialized='table',
    post_hook=[
        "GRANT SELECT ON {{ this }} TO ROLE analytics_reader",
        "COMMENT ON TABLE {{ this }} IS 'Updated by {{ env_var('CI_PIPELINE','manual') }}'"
    ],
    pre_hook=[
        "ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 7200"
    ]
) }}

SELECT
    order_id,
    customer_id,
    order_date,
    amount
FROM {{ ref('stg_orders') }}
```

**Non-transactional hooks (for operations that can't be in a transaction):**
```sql
{{ config(
    materialized='table',
    post_hook=[
        {
            "sql": "GRANT SELECT ON {{ this }} TO ROLE analytics_reader",
            "transaction": False
        }
    ]
) }}
```

**Multiple hooks in YAML (project-level or model-level):**
```yaml
# dbt_project.yml
models:
  jaffle_shop:
    marts:
      +post-hook:
        - "GRANT SELECT ON {{ this }} TO ROLE analytics_reader"

    staging:
      +pre-hook:
        - "ALTER SESSION SET TIMEZONE = 'UTC'"
```

### Production Hook Patterns

| Pattern | Hook Type | Example | Why |
|---|---|---|---|
| **Warehouse session config** | `pre_hook` | `ALTER SESSION SET QUERY_TAG = 'dbt_{{ model.name }}'` | Tag queries for cost tracking |
| **Post-build permissions** | `post_hook` | `GRANT SELECT ON {{ this }} TO ROLE analytics_reader` | Auto-grant on every build |
| **Table/view comments** | `post_hook` | `COMMENT ON TABLE {{ this }} IS 'Owner: data-team'` | Self-documenting tables |
| **Resource optimization** | `pre_hook` | `ALTER WAREHOUSE analytics_wh SET WAREHOUSE_SIZE = 'XLARGE'` | Scale warehouse for heavy model |
| **Data archival** | `post_hook` | Custom SQL to move old partitions to cold storage | Automated tiered storage |
| **Row-level security** | `post_hook` | `CREATE ROW ACCESS POLICY ...` | Security applied post-build |

### Pros/Cons

| Pro | Con |
|---|---|
| **Declarative** — hooks live in dbt config, not external scripts | **Tied to dbt lifecycle** — not reusable outside dbt |
| **Transaction-aware** — default transaction wraps hooks + model SQL | **Error handling is opaque** — hook failure may leave partial state |
| **Model-scoped** — different hooks per model as needed | **Hard to debug** — hook SQL errors intermingle with model errors |
| **Non-transactional option** — for DDL that can't be in a transaction | **Adapter-specific** — some hooks (GRANT syntax) are warehouse-specific |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Hook fails in a transaction | Entire model run rolls back — model not built | Use `transaction: False` for non-critical hooks |
| `post_hook` with `transaction: True` on DDL (CREATE TABLE) | DDL cannot be in transaction on some warehouses (Redshift) | Use `transaction: False` for DDL hooks |
| Hook references `{{ this }}` in a `view` materialization | `{{ this }}` resolves correctly — but GRANT on a view may need different syntax | Test GRANT syntax for views vs tables per adapter |
| Multiple `post_hook` entries with conflicting logic | Later hooks may undo earlier ones | Order hooks carefully; test hook interactions |
| Hook runs on every incremental build (not just full refresh) | Expensive hooks (e.g., GRANT) run unnecessarily | Add conditional logic: `pre_hook: ['{ "sql": "...", "transaction": false }']` |

### Follow-Up Questions

**Q1. What is the difference between hooks in `config()` vs in `dbt_project.yml`?**
→ `config()` in the model file applies only to that model. `dbt_project.yml` hooks apply to all models under the specified path (or globally with `models: +post-hook: ...`). Model-level `config()` overrides project-level.

**Q2. Can hooks run Python code?**
→ No — hooks execute raw SQL. For Python logic, use `dbt run-operation` with a custom macro.

**Q3. What is `{{ this }}` in a hook?**
→ It resolves to the fully qualified name of the model being built (`"database"."schema"."table"`). Essential for GRANT, COMMENT, and similar DDL.

**Q4. How do you disable hooks for a specific model?**
→ Override with empty: `{{ config(post_hook=[]) }}` or `{{ config(post_hook=[]) }}` to clear project-level hooks.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using hooks for logic that belongs in the model | Hooks are for infrastructure — not business logic | Move business logic into the model SQL |
| Forgetting `transaction: False` for DDL hooks on Redshift | DDL in transaction causes error on Redshift | Always use `transaction: False` for CREATE/ALTER DDL |
| Hook referencing `{{ ref('other_model') }}` | Resolved at compile time — but the model may not be built yet | Avoid cross-model refs in hooks; use `{{ this }}` instead |
| Not testing hooks in dev | Hooks may have different behavior in dev vs prod (different roles, schemas) | Test hooks in both environments |

### Principal-Level Considerations

- **Centralized permission management:** Instead of per-model hooks, use a single `post-hook` in `dbt_project.yml` to GRANT all models to the right roles. This ensures no model is ever left without proper access.
- **Cost attribution via hooks:** Every model's `pre_hook` can set `QUERY_TAG` or `STATEMENT_TAG` (Snowflake) with the model name, owner, and cost center. This enables precise compute cost attribution per model.
- **Audit trail hooks:** Post-hooks that log build events to an audit table (`INSERT INTO dbt_build_log (model_name, run_time, status, warehouse) VALUES ({{ model.name }}, CURRENT_TIMESTAMP, 'success', CURRENT_WAREHOUSE())`) provide a build-level audit trail for compliance.

---

## Q13. What is dbt's dispatch mechanism? How does `dbt_utils` override built-in macros, and how do you create custom dispatch paths?

### Problem Statement

Explain dbt's macro dispatch system — how dbt resolves which macro to call when multiple packages provide the same macro name, how `dbt_utils` works, and how to create custom dispatch paths for adapter-specific logic.

### Answer

**dbt dispatch** is the macro resolution system that allows you to **override built-in macros** with your own implementations. It works via a **search path** — dbt looks for a macro in order, using the first match it finds.

**How dispatch works:**

1. dbt checks your project's `macros/` directory first.
2. Then it checks each installed package's `macros/` directory (in the order listed in `dbt_project.yml`).
3. If no match is found, it falls back to dbt's internal built-in macros.

**Key config in `dbt_project.yml`:**
```yaml
dispatch:
  - macro_namespace: dbt_utils
    search_order: ['my_project', 'dbt_utils']
```

This tells dbt: "When resolving `dbt_utils.surrogate_key()`, first check `my_project` for a macro named `dbt_utils__surrogate_key`, then fall back to `dbt_utils`'s own implementation."

### Example

**Overriding a `dbt_utils` macro:**

```sql
-- macros/dbt_utils__safe_add.sql
-- Custom implementation that handles NULL differently than dbt_utils default

{% macro safe_add(expr1, expr2) %}
    COALESCE({{ expr1 }}, 0) + COALESCE({{ expr2 }}, 0)
{% endmacro %}
```

```yaml
# dbt_project.yml
dispatch:
  - macro_namespace: dbt_utils
    search_order: ['my_project', 'dbt_utils']
```

Now, when any model calls `{{ dbt_utils.safe_add(a, b) }}`, it uses your custom implementation instead of the one in `dbt_utils`.

**Creating a custom dispatch namespace:**

```sql
-- macros/jaffle_date.sql
{% macro jaffle_date__current_week_start() %}
    {{ adapter.dispatch('current_week_start', 'jaffle_date')() }}
{% endmacro %}

{% macro default__current_week_start() %}
    DATEADD(day, -EXTRACT(dow FROM CURRENT_DATE), DATE_TRUNC('week', CURRENT_DATE))
{% endmacro %}

{% macro snowflake__current_week_start() %}
    DATEADD(day, -(EXTRACT(DAYOFWEEK FROM CURRENT_TIMESTAMP()) % 7), DATE_TRUNC('week', CURRENT_TIMESTAMP()))
{% endmacro %}

{% macro bigquery__current_week_start() %}
    DATE_TRUNC(CURRENT_DATE(), WEEK(MONDAY))
{% endmacro %}
```

```sql
-- Usage in any model
SELECT * FROM orders
WHERE order_date >= {{ jaffle_date.current_week_start() }}
```

### Dispatch Resolution Order

```
1. my_project/dbt_utils__surrogate_key.sql  (your override)
2. dbt_utils/surrogate_key.sql               (package default)
3. dbt core built-in                         (fallback)
```

### Pros/Cons

| Pro | Con |
|---|---|
| **Adapter-specific macros** — write one macro that adapts per database | **Naming convention complexity** — must follow `{namespace}__{macro_name}` pattern |
| **Package override** — customize `dbt_utils` without forking it | **Resolution order is not obvious** — harder to debug which macro is being used |
| **Cross-database portability** — one dbt project works on Snowflake, BigQuery, Postgres | **Version coupling** — if `dbt_utils` changes its macro interface, your override may break |
| **Clean separation** — custom logic in your project, shared logic in packages | **Dispatch config is global** — cannot have different dispatch orders per model |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Two packages provide the same macro name (not in a namespace) | Whichever is listed first wins — silent behavior change | Use explicit dispatch config to control resolution order |
| `dispatch` config points to a package that doesn't have the macro | dbt falls back to next in `search_order` or built-in | Ensure fallback is correct; test in dev |
| Custom macro uses adapter-specific SQL not covered by dispatch | Works in one database, fails in another | Use `adapter.dispatch()` with per-adapter implementations |
| Package upgrade changes macro signature | Existing overrides may fail at compile time | Pin package versions; test after upgrades |

### Follow-Up Questions

**Q1. What is the difference between `adapter.dispatch()` and the `dispatch` config in `dbt_project.yml`?**
→ `adapter.dispatch()` is for **adapter-specific implementations** within a macro (e.g., `snowflake__foo`, `bigquery__foo`). The `dispatch` config in YAML is for **namespace resolution** (which package's macro takes precedence). They serve different purposes.

**Q2. How do you debug which macro is being called?**
→ Use `dbt --debug` — it logs macro resolution. Or add `{{ log('Using macro: ' ~ model.name, info=True) }}` in your custom macro.

**Q3. Can you dispatch to a macro in another package (not just your project)?**
→ Yes — list multiple packages in `search_order`: `search_order: ['my_project', 'team_shared', 'dbt_utils']`. dbt searches each in order.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Forgetting `adapter.dispatch()` in the wrapper macro | Only the `default__` implementation runs — adapter-specific code is ignored | Always use `{{ adapter.dispatch('macro_name', 'namespace')() }}` |
| Not following the `{namespace}__{macro_name}` naming convention | dbt cannot find your override | Name macros exactly: `dbt_utils__surrogate_key.sql` |
| Assuming dispatch order is alphabetical | It follows `search_order` in YAML — not file order | Explicitly set `search_order` in `dispatch` config |

### Principal-Level Considerations

- **Cross-database strategy:** Maintain a single dbt project that runs on multiple warehouses (Snowflake, BigQuery, Postgres) using `adapter.dispatch()` for all warehouse-specific SQL. This is the foundation of a "write once, run anywhere" data platform.
- **Shared macro library:** Create an internal package (`team_shared_macros`) with dispatch-configured overrides. All projects inherit your team's custom macros, ensuring consistency across teams.
- **Version pinning:** Always pin package versions in `packages.yml` (`package: dbt-labs/dbt_utils version: [">=1.0.0", "<2.0.0"]`) to prevent breaking changes in overridden macros.

---

## Q14. What is dbt governance? Explain groups, access controls, and metadata management at an organization level

### Problem Statement

Explain dbt governance features — model groups, access controls (`access`), and how to manage metadata (owners, descriptions, docs) across a multi-team organization.

### Answer

**dbt governance** encompasses features for managing ownership, permissions, and metadata at scale. Key features (introduced in dbt 1.8+):

1. **Groups** — Define ownership of models across teams
2. **Access** — Control who can consume models (`protected`, `restricted`, `private`, `public`)
3. **Metadata** — Owner, description, docs, and custom properties per model

### Groups

```yaml
# models/groups.yml
groups:
  - name: revenue_team
    owner:
      name: Revenue Analytics
      email: revenue-team@company.com
      slack: #revenue-analytics

  - name: data_engineering
    owner:
      name: Data Engineering
      email: data-eng@company.com

  - name: marketing_analytics
    owner:
      name: Marketing Analytics
      email: marketing-analytics@company.com
```

```yaml
# models/marts/schema.yml
version: 2

models:
  - name: fct_revenue
    group: revenue_team
    description: "Fact table for revenue calculations"
    access: protected
    columns:
      - name: revenue_usd
        tests:
          - dbt_utils.not_empty:
              group: revenue_team
```

### Access Levels

| Level | Who Can Use | Who Can Modify | Use Case |
|---|---|---|---|
| `public` | Anyone (any dbt project) | Group members | Core shared data marts |
| `protected` | Anyone in the same organization | Group members | Business-critical models |
| `restricted` | Only models in the same project | Group members | Intermediate/working tables |
| `private` | No `ref()` allowed from other groups/projects | Group members | Internal-only models |

**Default access:** `protected` (can be overridden per model).

### Metadata and Documentation

```yaml
# models/marts/schema.yml
models:
  - name: fct_orders
    group: revenue_team
    access: protected
    description: "Core order fact table — used by all downstream reporting"
    meta:
      owner: "revenue-team"
      cost_center: "revenue"
      sla: "daily"
    columns:
      - name: order_id
        description: "Unique order identifier"
        tests:
          - unique
          - not_null
      - name: order_date
        description: "Date the order was placed"
        meta:
          pii: false
      - name: customer_email
        description: "Customer's email address"
        meta:
          pii: true
        tests:
          - dbt_utils.accepted_range
```

### Example — Full Governance Block

```yaml
version: 2

groups:
  - name: revenue_team
    owner:
      name: Revenue Analytics
      email: revenue-team@company.com
      slack: #revenue-analytics

models:
  - name: fct_revenue_daily
    group: revenue_team
    access: protected
    description: "Daily revenue aggregated by product, region, and channel"
    meta:
      owner: revenue-team
      sla: daily
      refresh_schedule: "0 6 * * *"
      cost_center: finance
      dbt_version: "1.8.0"
    columns:
      - name: revenue_date
        description: "Date of revenue recognition"
        tests:
          - unique
          - not_null
      - name: gross_revenue_usd
        description: "Total revenue before returns and adjustments"
        tests:
          - dbt_utils.greater_than_equal:
              value: 0
    config:
      materialized: table
      tags: ['daily', 'revenue']
```

### Pros/Cons

| Pro | Con |
|---|---|
| **Clear ownership** — every model has a responsible team | **Requires discipline** — teams must update `groups.yml` and metadata |
| **Access control** — prevents unauthorized `ref()` access | **No runtime enforcement** — access is compile-time only (no warehouse-level enforcement) |
| **Custom `meta`** — extensible for any organizational need | **Tooling dependent** — meta properties need custom tooling to enforce |
| **Integration with dbt docs** — `dbt docs serve` shows ownership | **Migration effort** — existing projects need governance retrofitted |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Model has no `group` assigned | Falls to project default — ownership unclear | Enforce `group` on every model via CI lint rule or policy |
| `access: public` on a PII-containing model | Data leak — external projects can `ref()` the model | Audit `access` levels; use `meta.pii: true` to flag sensitive models |
| Owner email is outdated | Stale governance — wrong team gets alerts | Review `groups.yml` ownership quarterly |
| Cross-group `ref()` with `restricted` access | Compile error — unauthorized reference | Change access level or move model to shared scope |
| Multiple teams claim ownership of the same model | Conflicting governance — unclear who fixes issues | Define clear ownership rules; use `group` to assign single owner |

### Follow-Up Questions

**Q1. How does `access: protected` differ from `restricted`?**
→ `protected` allows any model in the same organization (any project) to `ref()` it. `restricted` allows only models in the same project to `ref()` it. `protected` is for sharing across projects; `restricted` is for internal-only models.

**Q2. Can access levels enforce warehouse-level permissions?**
→ No — dbt access is compile-time only. It controls which models can be referenced, not who can query the resulting tables. Use hooks (GRANT) or warehouse RBAC for runtime enforcement.

**Q3. How do you audit which models lack governance metadata?**
→ Use `dbt ls` with filters: `dbt ls --select config.group:none` to find unassigned models. Or write a custom Python script that parses `schema.yml` files.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Setting `access: public` on intermediate models | Increases coupling — external teams depend on your internal logic | Use `restricted` for intermediate models, `protected` or `public` only for published marts |
| Not assigning a `group` | No team owns the model — bugs go unfixed | Assign `group` to every model; use a default in `dbt_project.yml` |
| Leaving `meta` fields undocumented | Other teams don't know what they mean | Define a governance schema (what meta fields exist and their meaning) in a README |

### Principal-Level Considerations

- **Governance as code:** Treat `groups.yml` and `schema.yml` as infrastructure code. Review them in PRs alongside model SQL changes. Use CI to enforce that every new model has a group, access level, and description.
- **SLA enforcement:** Use `meta.sla` combined with dbt source freshness and test results to build a governance dashboard. Alert when a model's SLA is not met (e.g., daily model not refreshed within 8 hours).
- **Cost ownership:** Tie `meta.cost_center` to warehouse usage. Use pre-hooks to set query tags, then aggregate costs by cost_center. Each team pays for their own compute — a key data mesh principle.
- **Cross-org sharing:** `access: public` models become "published data products" — other dbt projects in the same organization can consume them. This enables data mesh: each domain publishes, others consume via `ref()`.
---

## Q15. How does dbt's `config` inheritance work across `dbt_project.yml` and model config blocks?

### Problem Statement

Explain how dbt resolves `config()` settings when defined at the project level, folder level, and model level — including precedence, merging, and the `+` prefix convention.

### Answer

dbt configures models at three levels of granularity, with a strict precedence order:

**Precedence (highest to lowest):**
1. **Model-level** `config()` block in `.sql` file
2. **Folder-level** config in `dbt_project.yml` (schema path matching)
3. **Project-level** config in `dbt_project.yml` (`models:` root key)
4. **Default** dbt defaults (e.g., materialized = 'view')

**Merge behavior:** A child config overrides the parent config for the **same key**. Different keys from different levels merge (complement each other).

**The `+` prefix:** In `dbt_project.yml`, the `+` prefix (e.g., `+materialized: table`) explicitly marks the config as applying to models, not to other resources. It's syntactic sugar and clarifies intent.

### Example

**Project-level config:**
```yaml
# dbt_project.yml
name: jaffle_shop

models:
  +materialized: view
  +schema: ANALYTICS
  +tags: ['core']

  jaffle_shop:
    staging:
      +materialized: view
      +schema: STAGING
      +tags: ['staging', 'core']

    marts:
      +materialized: table
      +schema: MARTS
      +tags: ['marts', 'core']
```

**Model-level config overrides:**
```sql
-- models/marts/fct_orders_daily.sql
{{ config(
    materialized='table',
    snowflake_warehouse='analytics_xl',
    schema='MARTS'
) }}

SELECT order_date, COUNT(DISTINCT order_id) AS orders
FROM {{ ref('stg_orders') }}
GROUP BY order_date
```

**Resolution:**
```
+materialized: table        (from model config — overrides dbt_project.yml)
+snowflake_warehouse: 'analytics_xl'  (from model config — merges)
+schema: MARTS              (from model config — overrides)
+tags: ['marts', 'core']    (from folder config in dbt_project.yml — merges with model tags)
```

### Merge Rules

| Scenario | Result |
|---|---|
| Model config sets `materialized='table'`, project sets `materialized='view'` | Model's `'table'` wins |
| Model config sets `tags=['a']`, project sets `tags=['b']` | Model's `['a']` wins (overwrites, does not merge) |
| Model config sets `materialized='table'`, project sets `schema='STAGING'` | Both apply: materialized from model, schema from project |
| No config in project or model | dbt default: `materialized='view'` |

### Pros/Cons

| Pro | Con |
|---|---|
| **DRY** — set common defaults once in project config | **Hidden overrides** — hard to see effective config for a model |
| **Overridable** — model config provides surgical control | **Inheritance is not intuitive** — `tags` overwrites, `schema` merges differently |
| **Multi-level** — project → folder → model is natural hierarchy | **No visual "effective config" viewer** — must read YAML + SQL |
| **`+` prefix is optional** — consistent behavior with or without it | **Merge vs override** difference for composite configs (tags, columns) is confusing |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| `tags` set at folder level and model level | Model-level `tags` overwrites folder tags | Always include all tags in model config if you override; use `+tags` in project to mean "add these tags" |
| `schema` config at folder level mixed with `schema` alias | Conflicting schema names — model may appear in unexpected schema | Use `+schema` consistently; avoid hardcoding schema in folder config |
| `config()` block in model uses `materialized='incremental'` but project sets `materialized='view'` | Model config wins — incremental model as expected | No issue — this is the intended behavior |
| No `+` prefix in project-level config | Works identically with or without `+` prefix | Use `+` for clarity but not required |

### Follow-Up Questions

**Q1. How do you see the effective config for a model at compile time?**
→ Use `dbt --debug run --select model_name` — dbt logs the effective `config` for each model. Or inspect `target/compiled/<model.sql>` for the compiled SQL (includes resolved config).

**Q2. Can you have a model that inherits config from a YAML `schema.yml` file?**
→ Yes — `config:` block in `schema.yml` sets folder-level config for all models listed under that path. This is less common than `config()` blocks but equally valid.

**Q3. What is `config.persist_docs` and how does it relate to this?**
→ `persist_docs` is a project-level config that pushes dbt descriptions (from `schema.yml`) to the warehouse as `COMMENT ON TABLE/COLUMN`. It's inherited like any other config: project → folder → model.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Setting `materialized: 'view'` in project, `materialized: 'table'` in model config | Works but confusing — hard to see which models override | Document overrides in model comments |
| Setting `+tags: ['staging']` at folder level, `tags: ['my_tag']` at model level | Model loses staging tag | Use model config: `tags: ['staging', 'my_tag']` to retain folder tags |
| Mixing `+` prefix and no prefix inconsistently | Works but hard to read | Use `+` prefix in all project-level configs for clarity |

### Principal-Level Considerations

- **Standardize config patterns:** Define a company-wide config hierarchy: Project level (materialized='view' as default), Team folder (e.g., `revenue/` has `materialized='table'`), Individual model (overrides only for exceptions). Enforce via CI lint rules.
- **Effective config visibility:** Build a report (via `dbt ls --output json`) that shows the resolved config for every model. Publish to a wiki or data catalog so all teams can see effective configs without reading YAML.
- **Cost cascade:** Configs for `snowflake_warehouse`, `cluster_by`, `partition_by` cascade. When a team sets an expensive warehouse at the project level, all models inherit it. Gate these cost-relevant configs at the folder level.

---

## Q16. How does dbt state and the `--defer` flag work? Explain the `target/` directory and state comparison

### Problem Statement

Explain dbt state management — how `dbt compile` and `dbt run` use the `target/` directory, how `--state` + `--defer` enable incremental builds in CI/CD, and how state comparison avoids rebuilding unchanged models.

### Answer

**dbt state** tracks the "shape" of your project — which models exist, their configs, their checksums. State is persisted as JSON files in the `target/` directory.

**Key state files in `target/`:**
- `target/manifest.json` — Full state (all nodes, edges, configs, checksums)
- `target/run_results.json` — Run outcomes (pass/fail, timing, bytes processed)
- `target/graph.gpickle` — Serialized DAG graph (for fast re-compilation)
- `target/sources.json` — Source freshness state

**State comparison flags:**
| Flag | What It Does |
|---|---|
| `--state path/to/state` | Point to a previous manifest.json to compare against |
| `--defer` | If a model hasn't changed, use the existing table from the previous run (don't rebuild) |
| `--defer --state target/prod/` | In production, skip unchanged models entirely |
| `--select state:modified` | Run only models whose checksum changed vs the state |
| `--select state:new` | Run only new models (not in the state manifest) |
| `--select result:error` | Re-run models that failed in the last run |

### State Comparison in CI/CD

```bash
# In CI: compare current PR branch to main branch state
# Step 1: Fetch main branch state (from artifact)
aws s3 cp s3://dbt-state-bucket/main/manifest.json target/prod/

# Step 2: Compile current branch, compare to main state
dbt compile --state target/prod/ --select state:modified+

# Step 3: Run only changed models and their downstream impacts
dbt run --state target/prod/ --defer --select state:modified+
```

**State comparison logic:**
```
state:modified = models whose checksum (based on SQL + config) differs from state
state:new      = models in current project but not in state
state:absent   = models in state but not in current project (deleted)
```

### Example

```yaml
# CI pipeline (GitHub Actions example)
  - name: Download prod state
    run: aws s3 cp s3://dbt-state/${{ github.base_ref }}/manifest.json target/prod/
    
  - name: Run changed models
    run: |
      dbt run --state target/prod/ --defer --select state:modified+ | tee dbt_run.log
      dbt test --state target/prod/ --defer --select state:modified+
      
  - name: Upload new state
    if: github.ref == 'refs/heads/main'
    run: aws s3 cp target/manifest.json s3://dbt-state/main/
```

### Pros/Cons

| Pro | Con |
|---|---|
| **Dramatically faster CI** — skip 90%+ of models on a typical PR | **State must be stored** — requires S3/cloud storage bucket |
| **No full refresh** — `--defer` avoids rebuilding upstream models | **State drift** — if warehouse data changes but model code doesn't, `--defer` misses it |
| **Fine-grained selects** — `state:modified+` pins impacted models | **State file size** — `manifest.json` can be 50MB+ for large projects |
| **Cross-environment consistency** — compare dev to prod state | **Checksum sensitivity** — whitespace changes trigger rebuild |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| `target/prod/manifest.json` is stale (not updated after last run) | State comparison is inaccurate — models may be re-run unnecessarily | Refresh state after every production run (automated pipeline) |
| A model changes only in a comment (no SQL change) | Checksum changes → model considered modified | Avoid comments in model files (use schema.yml for docs instead) |
| `--defer` with an upstream model that was physically dropped | Model relies on dropped table → model fails | Run `dbt run --full-refresh` periodically to catch physical state |
| State comparison between mismatched dbt versions | Manifest schema differs → comparison errors | Ensure state manifest was generated with same dbt version |

### Follow-Up Questions

**Q1. What is the `state:modified` selector vs `result:error`?**
→ `state:modified` compares checksums to a previous **manifest**. `result:error` uses the **run_results.json** from the last run. `state:modified` is for identifying what changed. `result:error` is for re-running failures.

**Q2. How do you persist state for multi-step pipeline (compile → run → test)?**
→ Run `dbt compile` first, which produces `target/manifest.json`. Then for subsequent steps, pass `--state target/` so each step uses the same state reference.

**Q3. What is the `source_status` field in state comparison?**
→ `source_status` tracks `fresher`, `status_match`, `staler` — it compares source freshness timestamps to detect data staleness changes.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Not storing state after production runs | `--defer --state` has no reference → all models rebuild | Always upload `manifest.json` to cloud storage post-run |
| Using `--defer` without `--state` | dbt doesn't know what to defer to | Always provide `--state path/` with a valid manifest |
| Assuming `--defer` skips all models it can | `--defer` only skips if the model is in the state manifest and hasn't changed | Use `state:modified+` for fine-grained selection |

### Principal-Level Considerations

- **State as an artifact:** Treat state as a deployable artifact, versioned just like code. Store in S3 with object versioning enabled. Every production run creates a new state version. Rollback is just pointing `--state` to an older version.
- **CI/CD pipeline with state:** In CI, download the main branch state, compare to the PR branch, and run only impacted models + their tests. This reduces CI from 60+ minutes to under 5 minutes for large projects.
- **State freshness SLAs:** If `--defer` is used, upstream models are NOT rebuilt. Schedule a weekly `--full-refresh` run to catch physical changes (dropped tables, schema migrations) that state comparison misses.

---

## Q17. What is dbt's lineage (column-level)? How does it work and how do you use it?

### Problem Statement

Explain dbt's column-level lineage — how it traces which source column(s) produce a given model column, how it's generated, and how to use it for impact analysis and debugging.

### Answer

**Column-level lineage** (introduced in dbt 1.5+) traces the flow of columns through the DAG — from source to staging to final mart. It answers: "If I change `source.orders.amount`, which downstream model columns will break?"

**How it works:**

1. dbt parses each `.sql` model file and tracks every column reference in `SELECT` statements.
2. It builds a graph of column dependencies: `model_X.column_A → model_Y.column_B`.
3. This graph is stored in `target/manifest.json` under the `column_depends_on` node metadata.
4. In `dbt docs`, click any column to see its lineage: upstream columns (where it came from) and downstream columns (where it propagates to).

### Example

**Staging model:**
```sql
-- models/staging/stg_orders.sql
SELECT
    id AS order_id,
    customer_id,
    amount_usd,
    status,
    created_at
FROM {{ source('jaffle_shop', 'orders') }}
```

**Intermediate model:**
```sql
-- models/marts/int_order_metrics.sql
SELECT
    customer_id,
    SUM(amount_usd) AS total_revenue,
    COUNT(DISTINCT order_id) AS order_count,
    MIN(created_at) AS first_order_date
FROM {{ ref('stg_orders') }}
GROUP BY customer_id
```

**Column lineage trace:**
```
source.orders.amount_usd → stg_orders.amount_usd → int_order_metrics.total_revenue
source.orders.created_at  → stg_orders.created_at  → int_order_metrics.first_order_date
source.orders.order_id    → stg_orders.order_id     → int_order_metrics.order_count
source.orders.customer_id → stg_orders.customer_id → int_order_metrics.customer_id
```

**Viewing lineage:**
```bash
dbt docs generate  # Builds lineage graph
dbt docs serve     # Open browser → click any model → click any column → see upstream/downstream
```

### Column-Level Lineage Query

```sql
-- Query the manifest.json directly (advanced)
-- Target: find all columns downstream of `source.orders.amount_usd`
SELECT node_key, column_name
FROM manifest.nodes
CROSS JOIN LATERAL FLATTEN(node.meta.column_depends_on)
WHERE column_depends_on.value:'source_node' = 'source:jaffle_shop.orders:amount_usd'
```

### Pros/Cons

| Pro | Con |
|---|---|
| **Impact analysis** — know exactly what breaks when a column changes | **Limited to direct SQL** — column transfer via CASE/IF statements is not traced |
| **Auto-generated** — no manual lineage effort | **Cross-model macros** — macro-injected columns are not traced |
| **Visual in dbt docs** — intuitive UI for lineage exploration | **Large project overload** — rendering column lineage for 500+ columns can be slow |
| **JSON exportable** — integrate with data catalog tools | **Column renaming** — lineage uses output column name, not source name |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Column is derived from a complex CASE statement | Lineage shows dependency on the source column but not the transformation logic | Review lineage for column-level mapping; use test to verify logic |
| Column passes through a macro (e.g., `{{ dbt_utils.surrogate_key() }}`) | Lineage traces the macro's inputs, not the output column | Check macro code for input → output column mapping |
| Column is added via `SELECT *` (not explicitly listed) | Lineage is incomplete — only columns explicitly referenced in SQL are traced | Avoid `SELECT *`; always list columns explicitly |
| Column lineage across multiple models (deep DAG) | Lineage is correct but UI can become crowded | Filter lineage to immediate upstream/downstream (click "one level up/down") |

### Follow-Up Questions

**Q1. Can you export column-level lineage to a third-party data catalog (e.g., Atlan, DataHub)?**
→ Yes — use `dbt docs generate --no-compile` (to skip re-compile) and parse `target/manifest.json`. Most data catalogs (Atlan, DataHub, Amundsen) have dbt integration that uses the manifest for column lineage.

**Q2. How do you do impact analysis before renaming a column?**
→ Use column lineage: look at `source.orders.amount` → find all downstream columns → see which would break if renamed. Also run `dbt test` after the change; failing tests indicate broken lineage.

**Q3. Is column-level lineage available for Python models?**
→ Partially — Python models (`materialized='table'`, type: python) do have column lineage if the column names are deducible from the final `DataFrame`. But complex transformations inside Python functions are not traced.

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using `SELECT *` — lineage trace is broken | Column lineage can't determine which columns flow through | Always list columns explicitly |
| Not running `dbt docs generate` after schema changes | Lineage is stale | Include `dbt docs generate` in CI/CD pipeline |
| Assuming column lineage traces via aggregate functions | It does — just the source column, not the aggregate logic | Use column lineage for source identification, not for debugging aggregation |

### Principal-Level Considerations

- **Impact analysis at scale:** Before a large migration (e.g., renaming 50 columns in the source schema), run column-level lineage to generate a "downstream impact report." Share this report with all downstream teams — they see exactly which of their columns will break.
- **Data catalog integration:** Export column-level lineage to a data catalog (Atlan, DataHub) daily. This provides business users self-serve impact analysis without reading dbt code.
- **Lineage as data contract:** Column-level lineage is the technical foundation of data contracts. When a domain publishes a data product, the contract includes the column-level lineage — consumers see exactly how each column was derived.

---

## Q18. What is dbt's package ecosystem? How do you create and manage packages?

### Problem Statement

Explain the dbt package ecosystem — the Hub, how to use packages in `packages.yml`, version management, and how to create and publish a custom package.

### Answer

**dbt packages** are reusable collections of macros, models, tests, and analyses. They are the dbt equivalent of libraries (pip packages, npm modules, ruby gems).

**Package ecosystem layers:**
1. **dbt Hub** (hub.getdbt.com) — Official package registry, 100+ packages
2. **GitHub-based packages** — From any public or private GitHub repo
3. **Local packages** — From a local file path (for monorepo setups)
4. **dbt Labs-maintained** — `dbt_utils`, `dbt_expectations`, `dbt_audit_helper`, `dbt_external_tables`
5. **Community packages** — `re_data`, `dbt_ml`, `dbt_profiler`, `dbt_linreg`, `dbt_artifacts`

### Using packages

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]

  - package: dbt-labs/dbt_expectations
    version: ">=0.8.0"

  - package: dbt-labs/audit_helper
    version: ">=0.9.0"

  - package: calogica/dbt_date
    version: [">=0.7.0", "<1.0.0"]

  - git: "https://github.com/your-org/your-private-package.git"
    revision: v1.2.3
    warn-unpinned: false  # Allow git-based packages

  # Local package (monorepo)
  - local: ../shared-dbt-macros
```

```bash
# Install all packages
dbt deps

# Pin a package version (in packages.yml)
dbt deps --lock   # Generates package-lock.yml

# List installed packages
dbt deps --status

# Update a single package
dbt deps --upgrade dbt_utils
```

### Creating a Package

**Package structure:**
```
my-dbt-package/
├── macros/
│   ├── my_custom_macros.sql
│   ├── dbt_utils__safe_add.sql   # Dispatch override example
│   └── my_custom_generic_tests.sql
├── models/
│   └── example_schema.yml
├── analyses/
│   └── my_analysis.sql
├── integration_tests/             # Test your own package
│   ├── dbt_project.yml
│   ├── models/
│   └── macros/
├── README.md
├── CHANGELOG.md
├── LICENSE
└── dbt_project.yml
```

**Package `dbt_project.yml`:**
```yaml
name: my_dbt_package
version: 1.0.0
config-version: 2

require-dbt-version: [">=1.5.0", "<2.0.0"]

macro-paths: ["macros"]
model-paths: ["models"]
analysis-paths: ["analyses"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_modules"
```

**Publishing to Hub:**
```
1. Create a GitHub repository with the package
2. Add `hub` tag in GitHub repo settings
3. Submit to hub.getdbt.com via PR
4. Version with Git tags (v1.0.0, v1.1.0)
5. Maintain changelog and upgrade guide
```

### Package Categories

| Category | Example Packages | What They Provide |
|---|---|---|
| **Utility** | `dbt_utils`, `dbt_expectations` | Macros (surrogate_key, date_spine, pivot) and tests |
| **Data quality** | `re_data`, `dbt_profiler` | Profiling, anomaly detection, data quality monitors |
| **Date/time** | `dbt_date` | Calendar tables, holidays, date functions |
| **Audit** | `audit_helper` | Compare two tables row-by-row |
| **External** | `dbt_external_tables` | Manage external tables (S3, GCS) |
| **ML** | `dbt_ml` (community) | Feature engineering macros |
| **Adapters** | `dbt-redshift`, `dbt-bigquery` | Database-specific functionality (installed separately) |

### Pros/Cons

| Pro | Con |
|---|---|
| **Reusable code** — don't reinvent common macros | **Version conflicts** — two packages may require different dbt versions |
| **Community vetted** — `dbt_utils` has 500+ stars, active maintenance | **Hub publication friction** — submitting a package to Hub is a manual process |
| **Compiled locally** — packages are pulled as source code, not binaries | **Duplication** — similar packages (dbt_utils vs dbt_expectations) overlap in functions |
| **Git support** — use any private or public GitHub repo | **Local package path** — absolute paths break across teammates' machines |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Two packages define the same macro name | One package's macro overwrites the other (order-dependent) | Use `dispatch` config to control resolution order |
| `dbt deps` pulls an incompatible dbt version (`require-dbt-version`) | Install fails — `dbt deps` errors with incompatibility | Pin dbt version in project; check `require-dbt-version` of packages |
| Git-based package uses `revision: main` (unpinned) | CI/CD builds are non-deterministic — different code each time | Always pin to a tag or commit SHA |
| Package has a model with a `source()` that doesn't exist in your project | dbt compiles but the source doesn't exist → runtime error | Review package docs; ensure all expected sources are defined |
| Multiple packages installed: different versions of same package | `dbt deps` resolves to the latest compatible version | Use `>=1.0.0, <2.0.0` version ranges; check `package-lock.yml` |

### Follow-Up Questions

**Q1. How do you debug a package that is not installing correctly?**
→ Use `dbt deps --verbose` to see download progress. Check `dbt_packages/` (or `dbt_modules/`) to see what was installed. Run `dbt --version` to check package version.

**Q2. What is the difference between `dbt_modules` and `dbt_packages`?**
→ `dbt_modules/` is the legacy directory (pre-1.5). `dbt_packages/` is the modern directory (dbt 1.5+). Both contain the unpacked package contents.

**Q3. How do you pin a package to a specific commit?**
```yaml
  - git: "https://github.com/your-org/your-package.git"
    revision: abc12345  # Git commit SHA
```

**Q4. How do you create a private package for internal use?**
→ Use a `git:` URL with an SSH deploy key or token. Store the token as a secret in CI/CD and reference it in `packages.yml`:
```yaml
  - git: "https://{{ env_var('GITHUB_TOKEN') }}@github.com/org/internal-package.git"
    revision: v1.0.0
```

### Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Not pinning package versions | Different versions in dev vs prod → inconsistent behavior | Pin with `[">=1.0.0", "<2.0.0"]` or exact version |
| Installing unnecessary packages | Slower `dbt deps`, more namespace conflicts | Install only what you need |
| Committing `dbt_packages/` to git | Bloated repo; `dbt deps` should be run by each dev | Add `dbt_packages/` to `.gitignore` |
| Using a `local:` path that doesn't exist on CI | CI build fails — package not found | Use `git:` for all packages shared across environments |

### Principal-Level Considerations

- **Internal package stewardship:** Maintain an internal dbt package (`org_utils`) that all teams share. It includes company-specific macros, custom generic tests, and dispatch overrides. Treat it as an internal open-source project with PR reviews, versioning, and changelogs.
- **Package version governance:** Pin all packages in `packages.yml` with `[">=X.Y", "<X+1.Z"]` version ranges. Require a PR review for any version bump. Use `package-lock.yml` (dbt 1.6+) for deterministic installs.
- **Security audit for packages:** Before using a community package, scan its code for security issues (data exfiltration, network calls). Prefer dbt Labs-maintained packages over unmaintained ones.
---

## Q19. What are the different dbt objects (models, sources, seeds, snapshots, exposures, metrics)? How do you choose between them?

### Problem Statement

Enumerate and explain all dbt object types, their purpose, when to use each, and the relationship between them in the DAG.

### Answer

dbt defines six main object types in `manifest.json`. Each serves a distinct role:

| Object | Keyword | Purpose | Materialized Output | Example |
|---|---|---|---|---|
| **Model** | `ref()` | Core transformation (SELECT statement) | Table / View / Incremental / Ephemeral | `stg_orders.sql` |
| **Source** | `source()` | Raw table imported into warehouse (external pipeline) | Not a dbt-run object — just metadata | A Fivetran-synced table |
| **Seed** | `ref()` | CSV file loaded as a table (small static data) | Table in warehouse | `country_codes.csv` |
| **Snapshot** | `snapshot()` | Type-2 slowly changing dimension (history capture) | Table with timestamp tracking | `customer_snapshot.sql` |
| **Exposure** | `ref()/source()` in YAML | Links dbt models to downstream BI/reporting | No warehouse object — metadata only | Tableau dashboard reference |
| **Metric** | `metrics.yml` | Defines a business metric (dbt 1.3+ / dbt MetricFlow) | No warehouse object — semantic layer | "Monthly Active Users" |

### When to Use Each

- **Sources:** Always define every raw table as a source — gives you freshness checks, docs, and DAG traceability.
- **Models:** The primary workhorse. Use staging → intermediate → mart pattern.
- **Seeds:** Only for small, rarely-changing reference data (< 1M rows). Avoid for large datasets.
- **Snapshots:** Use for dimension models that need change history (customer profiles, product attributes).
- **Exposures:** Define in `schema.yml` for every downstream dashboard, report, or ML feature. This prevents "orphan models."
- **Metrics:** Use `dbt_metrics` (pre-1.6) or MetricFlow (1.6+) for curated, governed business metrics.

### Pros/Cons

| Object | Pro | Con |
|---|---|---|
| **Sources** | Freshness checks, DAG traceability | Requires external pipeline (Fivetran, Airbyte) to populate |
| **Models** | Full transformation power | Need to maintain and test |
| **Seeds** | Easy static data loading | Not version-controlled; only CSV; bad for >1M rows |
| **Snapshots** | Built-in SCD Type-2 | Only Type-2; no Type-1 or Type-6 |
| **Exposures** | DAG completeness | Must be manually maintained; no auto-detection |
| **Metrics** | Consistent metrics | Complex to define; limited to simple aggregations |

### Edge Cases

| Scenario | Best Object |
|---|---|
| Loading a small mapping CSV (100 rows) | Seed |
| Loading a 10M-row lookup table from a CSV | Model with `materialized='table'` referencing the CSV as an external source |
| Referencing a Fivetran-synced table | Source |
| Tracking customer address history | Snapshot |
| Linking dbt model to a Tableau workbook | Exposure |
| Defining "Total Revenue" as a curated metric | Metric (dbt_metrics or MetricFlow) |

### Follow-Up Questions

**Q1. Can a model reference both a source and a seed?**
→ Yes — `stg_orders.sql` can `{{ source('raw', 'orders') }}` and also `{{ ref('country_codes') }}` (a seed).

**Q2. What object should I use for an API-sourced dataset that I want to transform?**
→ Model that reads from a source (if the API data lands in warehouse via Fivetran/Airbyte) or use `dbt_external_tables` for external S3/GCS files.

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Using seeds for large dataset >1M rows | Slow loading; CSV parsing issues | Use a model that reads from a source |
| Not defining sources | No DAG traceability, no freshness checks | Always define sources for every raw table |
| Creating an exposure but not maintaining it | Stale metadata — consumers trust wrong model | Automate exposure generation via API |

### Principal-Level Considerations

- **Exposure-driven development:** Require every dbt model deployed to production to be linked to at least one exposure. If a model has no downstream consumer, question whether it should exist.
- **Object count governance:** Large dbt projects (>500 objects) become hard to navigate. Set guidelines: max 70% models, 10% sources, 10% seeds, 5% snapshots, 5% exposures.

---

## Q20. How does dbt model versioning work? Explain `versions` in schema.yml and the `ref` version pinning

### Problem Statement

Explain dbt's model versioning feature (dbt 1.5+) — how to version models, use `@` in `ref()`, deprecated versions, and how to migrate consumers to new versions.

### Answer

**Model versioning** allows you to maintain multiple versions of the same logical model simultaneously. This enables gradual migrations: consumers can adopt the new version without breaking existing pipelines.

**Key concepts:**
- `latest_version` — marks which version is the default (when `ref()` doesn't specify a version)
- `@` version pin — `{{ ref('fct_orders', v='2') }}` pins to a specific version
- `deprecated` — marks a version as deprecated (emits warnings to consumers)
- `access: protected` — prevents other projects from ref'ing certain versions

### Example

```yaml
# models/schema.yml
version: 2
models:
  - name: fct_orders
    description: "Order facts — v2 adds currency normalization"
    latest_version: 2
    versions:
      - v: 1
        schema:
          # Column definitions for v1
          columns:
            - name: order_id
            - name: amount_usd
              data_type: decimal(18,2)
        deprecation_date: 2024-06-01
        access: protected  # Only this project can ref v1

      - v: 2
        columns:
          - name: order_id
          - name: amount
            data_type: decimal(18,2)
          - name: currency
            data_type: varchar(3)
```

```sql
-- Consumer model — pins to v2 explicitly
SELECT *
FROM {{ ref('fct_orders', v='2') }}
-- This breaks if v2 is removed

-- Consumer model — uses latest version (v2)
SELECT *
FROM {{ ref('fct_orders') }}
-- Automatically uses latest_version
```

### Pros/Cons

| Pro | Con |
|---|---|
| **Gradual migration** — consumers update one at a time | **Dead versions** — old versions must be maintained (warehouse tables) |
| **Backwards compatibility** — old consumers still work | **Complex YAML** — maintaining multiple version schemas is verbose |
| **Deprecation warnings** — alerts consumers before breaking | **Only table models** — not yet available for views or incremental (dbt 1.5) |
| **Access control** — restrict who can see deprecated versions | **Test overhead** — must test all active versions |

### Edge Cases

| Scenario | Impact | Mitigation |
|---|---|---|
| Consumer uses `ref('fct_orders', v='1')` but v1 is removed | Run fails | Set `deprecation_date` well in advance; monitor warnings |
| `latest_version` is updated from 1 to 2 | All consumers not pinning a version automatically get v2 | Ensure v2 is fully backwards-compatible; pin consumers that need migration |
| A model with versioning has columns that differ between versions | Tests must be version-aware | Use `config-version: 2` in tests to target specific model versions |

### Follow-Up Questions

**Q1. What happens to the warehouse table when a version is deprecated?**
→ The table stays (dbt doesn't drop it). You must manually drop or use `--full-refresh` after deprecation.

**Q2. Can I version seeds or snapshots?**
→ No — versioning is currently only for models (`materialized='table'` or `'view'`).

**Q3. How do I run tests only for the latest version?**
→ Use `dbt test --select fct_orders --vars '{"latest_version": true}'` or configure test selection with version-aware tags.

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Removing v1 immediately after releasing v2 | Breaks all consumers still pinning v1 | Set deprecation_date 3+ months in the future |
| Not informing consumers about new version | Warnings ignored; migration delayed | Send calendar invitation for version migration |
| Defining versions for an incremental model | dbt doesn't support versioning for incremental models yet | Keep a single incremental model; version only non-incremental models |

### Principal-Level Considerations

- **Version lifecycle policy:** Define a company-wide version lifecycle: v1 → v2 released → 3-month migration window → v1 deprecation warning → v1 dropped. Enforce via a dbt lint rule that blocks `ref()` to a version past its deprecation date.
- **Backward compatibility guarantees:** When the latest version changes, the old version's column names and types must remain stable. Only add columns, never remove or rename.
- **Automation:** Build a CI job that checks all `ref()` calls across the project: if any still point to a deprecated version, the PR fails with a list of models that need updating.

---

## Q21. What is the difference between dbt views vs tables vs incremental vs ephemeral? When do you use each?

### Problem Statement

Compare dbt's four materialization strategies: view, table, incremental, and ephemeral. Explain their performance characteristics, cost, freshness considerations, and when each is appropriate.

### Answer

| Materialization | SQL Built | Warehouse Object | When to Use | Freshness |
|---|---|---|---|---|
| **View** | `CREATE VIEW AS ...` | View (lightweight) | Staging models, reference views, frequent-refresh needs | Always latest data |
| **Table** | `CREATE TABLE AS SELECT ...` | Table (materialized) | Small- to medium-sized models (<10M rows), endpoint models, mart models | Stale between runs |
| **Incremental** | `INSERT/UPDATE ... WHERE ...` | Table (with append/merge) | Large models (>100M rows), append-only logs, fact tables | Fresh at each run (new records) |
| **Ephemeral** | CTE (common table expression) | None (compile-time only) | Intermediate transformations, reusable CTEs, no downstream reference needed | Same as upstream |

### Example

```sql
-- View: always current, no storage cost
{{ config(materialized='view') }}
SELECT * FROM {{ source('raw', 'orders') }}
WHERE status = 'active'

-- Table: explicit materialization snapshot
{{ config(materialized='table') }}
SELECT customer_id, SUM(amount) AS total_spend
FROM {{ ref('stg_orders') }}
GROUP BY customer_id

-- Incremental: append-only, for large log data
{{ config(materialized='incremental', unique_key='order_id') }}
SELECT * FROM {{ source('raw', 'orders') }}
{% if is_incremental() %}
    WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}

-- Ephemeral: pure CTE, no warehouse object
{{ config(materialized='ephemeral') }}
SELECT customer_id, LOWER(email) AS email_normalized
FROM {{ source('raw', 'customers') }}
```

### Pros/Cons

| Materialization | Pro | Con |
|---|---|---|
| **View** | Zero storage; always current; fast to create | Query performance on complex views; no indexing; cannot add custom partitioning |
| **Table** | Fast reads; can index, partition, cluster | Storage cost; full rebuild on each run |
| **Incremental** | Fast for new data; low storage overhead | Complexity in merge logic; backfills can be slow |
| **Ephemeral** | Zero storage; no overhead in warehouse | Re-computed every use; cannot be tested directly; cannot be used in `source()` or `exposure()` |

### Edge Cases

| Scenario | Best Choice | Why |
|---|---|---|
| Model used by 50 dashboards, refreshed hourly | View (or table with incremental) | View avoids stale cache; table is okay if rebuild is fast |
| Model that filters 1B records down to 100K | Table (materialized) | View would scan 1B each query; table scans 100K |
| Model used ONLY as a CTE in one other model | Ephemeral | No need for warehouse object |
| Fact table with 5B rows, daily new 10M rows | Incremental | Full rebuild would be too expensive |

### Follow-Up Questions

**Q1. Can you change a view to a table without breaking downstream consumers?**
→ Yes — consumers use `ref()`, which resolves to the object regardless of materialization. The `ref()` binds to the model name, not the materialization type.

**Q2. Why are ephemeral models controversial?**
→ They can't be tested (no table to test against), can't be exposed (no warehouse object), and their SQL is inlined — debugging compiled SQL is harder. Use sparingly.

**Q3. What is `materialized='dynamic_table'` (Databricks)?**
→ Databricks-specific: a table that magically keeps itself fresh (streaming + batch). dbt 1.8+ supports this as a native materialization.

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Materializing everything as tables | High storage cost, slow rebuilds | Use views for staging; incremental for large facts |
| Using ephemeral for a commonly-referenced model | Re-computed everywhere | Change to view or table |
| Using view for a model with complex joins | Dashboard queries are slow | Materialize as table or incremental |
| Not using incremental for 5B-row fact tables | Full rebuild takes hours | Switch to incremental with unique_key |

### Principal-Level Considerations

- **Materialization by tier:** Define a materialization policy per DAG tier: **Staging** = view (always current), **Intermediate** = ephemeral or view (fast to build, no persistence), **Marts** = table or incremental (fast reads, persistent). Enforce in CI.
- **Cost vs freshness trade-off:** Views cost per-query (warehouse compute). Tables cost storage + compute on rebuild. For high-query dashboards, materialize as table even for small data — reduces query cost by 10x.
- **Incremental as default for large data:** Any model with >100M rows should default to incremental. Full-table rebuilds for billion-row tables are anti-patterns. Set this expectation in team guidelines.

---

## Q22. How do you handle cross-database deployments with dbt? (Multi-platform support)

### Problem Statement

Explain how to manage a dbt project that targets multiple data platforms (Snowflake, BigQuery, Redshift, Databricks, Postgres) — adapter dispatch, platform-specific macros, conditional SQL, and migration strategies.

### Answer

**dbt cross-database support** is achieved through:

1. **Adapters** — `dbt-snowflake`, `dbt-bigquery`, `dbt-redshift`, `dbt-databricks`, `dbt-postgres` each speak the platform's SQL dialect
2. **Dispatch mechanism** — `{{ dbt_utils.date_trunc('day', 'created_at') }}` resolves to the correct SQL function for each platform
3. **Platform-specific models** — Different model files for different adapters (e.g., `models/marts/fct_orders__snowflake.sql`)
4. **Cross-db macros** — `dbt_utils` and `dbt_date` provide platform-agnostic date/time/string functions
5. **`target.type` variable** — `{% if target.type == 'snowflake' %}...{% endif %}` for conditional SQL

### Example

```yaml
# dbt_project.yml — multiple targets
targets:
  dev:
    type: postgres
    schema: dev_schema
    
  prod:
    type: snowflake
    database: PROD_DB
    schema: PROD_SCHEMA
```

```sql
-- Platform-agnostic model using dbt_utils
{{ config(materialized='incremental', unique_key='order_id') }}
SELECT
    order_id,
    customer_id,
    amount,
    {{ dbt_utils.date_trunc('day', 'created_at') }} AS order_date
FROM {{ source('raw', 'orders') }}
{% if is_incremental() %}
    WHERE created_at > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

```sql
-- Platform-specific macro dispatch
{% macro my_project__hash_this(value) %}
    {% if target.type == 'snowflake' %}
        SHA2({{ value }}, 256)
    {% elif target.type == 'bigquery' %}
        SHA256(CAST({{ value }} AS STRING))
    {% elif target.type == 'postgres' %}
        ENCODE(DIGEST({{ value }}, 'sha256'), 'hex')
    {% endif %}
{% endmacro %}
```

### Dispatch Configuration

```yaml
# dbt_project.yml — dispatch for cross-db
dispatch:
  - macro_namespace: dbt_utils
    search_order:
      - my_project  # Override dbt_utils date_trunc for custom logic
      - dbt_utils   # Fall back to dbt_utils default
```

### Pros/Cons

| Pro | Con |
|---|---|
| **Single codebase** for all platforms | **SQL dialect differences** — functions, window syntax, DDL differ |
| **`dbt_utils`** abstracts many cross-db functions | **Not all functions covered** — some need manual dispatch |
| **CI/CD matrix testing** — run tests on all platforms | **Slower CI** — multiple platform builds |
| **Dispatch framework** — clean override pattern | **Learning curve** — developers must understand adapter dispatch |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| `dbt_utils.date_trunc()` does not exist for Databricks | Compilation error (missing macro) | Write a custom dispatch for `date_trunc` targeting Databricks |
| `LISTAGG(DISTINCT ...)` works on Snowflake but not Redshift | SQL compilation error on Redshift | Use `dbt_utils.listagg()` which handles dialect differences |
| Cross-db model with `PARTITION BY` clause (Snowflake-only) | Fails on Postgres | Wrap in `{% if target.type == 'snowflake' %}...{% endif %}` |
| Warehouse type detected but adapter not installed | `dbt deps` fails — adapter not found | Install correct adapter per CI environment |

### Follow-Up Questions

**Q1. How do you test cross-database compatibility?**
→ Use a CI matrix: run `dbt compile` with each adapter's `profiles.yml`. A model that compiles on Snowflake may fail on Postgres. Catch failures before PR merge.

**Q2. What is the most common cross-database pain point?**
→ Date/time functions — each platform has different syntax for date_trunc, dateadd, datediff, extract. Use `dbt_date` package for these.

**Q3. Can you have the same model with different SQL per platform?**
→ Yes — `models/marts/fct_orders__snowflake.sql`, `models/marts/fct_orders__bigquery.sql`. dbt selects the correct file based on `target.type`.

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Writing raw Snowflake-specific SQL in a model used on BigQuery | Compilation/runtime error on BigQuery | Use `dbt_utils` or dispatch for platform-agnostic functions |
| Assuming `dbt_utils` covers all cross-db use cases | Some operations (e.g., `MERGE` syntax) are not abstracted | Write custom dispatch macros for unsupported operations |
| Not testing on target platform before deployment | Failure discovered in production | Add CI matrix testing for all target platforms |
| Using `target.type` in too many places | Hard-to-read, spaghetti SQL | Extract into macros; keep model SQL clean |

### Principal-Level Considerations

- **Cross-platform as a business requirement:** If your company supports Snowflake + Databricks (acquisition scenario), the dbt codebase must be platform-agnostic. Invest in a comprehensive dispatch library (100+ macros) covering all SQL dialect differences.
- **Migration strategy:** When migrating from Redshift to Snowflake, first make the dbt code cross-database (Redshift + Snowflake compatible with dispatch), then switch the target. Users see no downtime.
- **Adopter-specific features:** Don't sacrifice platform strengths for cross-db compatibility. Partitioning (Snowflake), clustering (BigQuery), and auto-optimize (Databricks) are valuable. Use platform-specific models for performance-critical marts.

---

## Q23. How does dbt testing strategy work? Explain generic tests, singular tests, and custom test macros

### Problem Statement

Describe dbt's testing framework — types of tests (generic vs singular), how to write custom generic tests as macros, how to organise tests in a production project, and how tests integrate with CI/CD.

### Answer

**dbt tests** are SQL queries that return rows on failure. A test passes if it returns 0 rows; it fails if it returns >0 rows.

**Three types of tests:**

| Type | What It Is | Where Defined | When to Use |
|---|---|---|---|
| **Generic test** | Pre-defined test macro (reusable) | `schema.yml` under a model's `columns:` or `tests:` | Standard checks: `not_null`, `unique`, `accepted_values`, `relationships` |
| **Custom generic test** | User-defined macro (reusable pattern) | `macros/tests/*.sql` | Reusable business rules: `positive_value`, `row_count_threshold`, `column_value_between` |
| **Singular test** | One-off SQL query | `tests/*.sql` | Ad-hoc, context-specific checks, complex logic not worth abstracting |

### Example

**Generic tests in schema.yml:**
```yaml
version: 2
models:
  - name: fct_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: amount
        tests:
          - not_null
          - positive_value:   # Custom generic test
              min_value: 0
      - name: customer_id
        tests:
          - relationships:
              to: ref('dim_customers')
              field: customer_id
    tests:
      - row_count_threshold:
          min_rows: 1000
```

**Custom generic test macro:**
```sql
-- macros/tests/positive_value.sql
{% test positive_value(model, column_name, min_value=0) %}
    SELECT *
    FROM {{ model }}
    WHERE {{ column_name }} < {{ min_value }}
{% endtest %}
```

**Singular test:**
```sql
-- tests/assert_date_spine_is_current.sql
WITH max_date AS (
    SELECT MAX(order_date) AS max_dt FROM {{ ref('fct_orders') }}
)
SELECT * FROM max_date WHERE max_dt < CURRENT_DATE - INTERVAL '1 day'
```

### Running Tests

```bash
# Run all tests
dbt test

# Run only tests for specific models
dbt test --select fct_orders+

# Run tests with store_failures (persist failure records)
dbt test --store-failures

# Run tests and fail fast (stop on first failure)
dbt test --fail-fast

# Run tests with limited scope in CI
dbt test --state target/prod/ --defer --select state:modified+
```

### Pros/Cons

| Aspect | Pro | Con |
|---|---|---|
| **Generic tests** | Reusable, declarative, 0 SQL code per test | Limited logic — only check one column at a time |
| **Custom generic tests** | Reusable business logic, parameterised | Must write and maintain macro code |
| **Singular tests** | Full SQL flexibility | Not reusable; easy to grow unmanageable |
| **store_failures** | Audit trail of failures | Storage overhead for failure records |
| **CI integration** | Catch data quality issues pre-merge | Slow if testing all models every time |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Test on empty model — `not_null` on empty table | Returns 0 rows → test passes (correct) | Use `row_count_threshold` test alongside |
| Test on incremental model — `unique` test on 5B rows | `SELECT ... GROUP BY ... HAVING COUNT(*) > 1` scans entire table, expensive | Run test only on new/updated partitions; use `where` clause in test |
| Custom generic test uses `{{ ref() }}` which resolves differently than expected | Test runs on wrong model | Use `{{ model }}` parameter in generic test, not `{{ ref() }}` |
| CI runs all tests on every PR (200 tests × 5 min each) | CI takes hours | Use state-based test selection: only test modified models |

### Follow-Up Questions

**Q1. How do you organise tests in a production project?**
→ Follow the "test pyramid": 80% generic tests (`schema.yml`), 15% custom generic tests (`macros/tests/`), 5% singular tests (`tests/`). Each model should have at least `unique` + `not_null` on its primary key.

**Q2. What is `dbt test --store-failures`?**
→ Persists test failure records to a table (`test_name__failures`). Useful for auditing data quality over time. Won't help you debug faster, but provides a history of failures.

**Q3. How do you test a model that uses `source()` with freshness checks?**
→ Add `freshness:` config in `schema.yml` for the source. Also add `not_null` and `unique` tests on the source's primary key columns — these tests run against the source table directly.

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Not testing primary keys | Duplicate rows go undetected | Add `unique` + `not_null` on every model's primary key |
| Running all tests in CI without state filtering | Slow pipeline | Use `state:modified+` to test only changed models + their downstream |
| Writing singular tests for every edge case | Test folder becomes unmanageable | Refactor frequent patterns into custom generic tests |
| Ignoring test results | Tests lose value | Set CI to fail the pipeline if any test fails |

### Principal-Level Considerations

- **Test coverage SLA:** Set a minimum test coverage for every model: primary key (`unique` + `not_null`), one business rule per fact column, one row-count threshold. Measure coverage via `dbt test --coverage` (dbt 1.6+).
- **Test as documentation:** A test is a living piece of documentation. When a business rule change occurs, the test changes first (before the model SQL). Treat test updates as code changes.
- **Data quality SLAs:** Define response time for test failures: P0 (critical) = fix within 1 hour, P1 = within 4 hours, P2 = within 1 day. Route test failures to the owning team via webhook (dbt Cloud + PagerDuty/Slack).
---

## Q24. How do you handle table partitioning and clustering in dbt?

### Problem Statement

Explain how to configure partitioning and clustering for dbt models on Snowflake (`partition_by`, `cluster_by`), BigQuery (`partition_by`, `cluster_by`), Databricks (`partition_by`), and how to manage partition pruning for incremental models.

### Answer

**Partitioning** splits a table into segments by a column (e.g., `order_date`). **Clustering** sorts data within partitions for better compression and query pruning.

**Platform-specific configs:**

| Platform | Partition Config | Cluster Config | Notes |
|---|---|---|---|
| **Snowflake** | `partition_by` (not directly — Snowflake uses automatic clustering) | `cluster_by` (one or more columns) | Snowflake auto-clusters; set `cluster_by` for large tables |
| **BigQuery** | `partition_by: {field: order_date, data_type: date, granularity: day}` | `cluster_by: ['customer_id', 'status']` | Required for tables >1GB |
| **Databricks** | `partition_by: ['order_date']` | `clustered_by: ['customer_id']` (Delta Lake) | Partition by high-cardinality date column |
| **Redshift** | Sort key + Distribution style (no dbt-native config) | Sort key via `sort_type` and `sort` config | Use `sort_type:'interleaved'` for multiple columns |

### Example

```sql
-- BigQuery partition + cluster
{{ config(
    materialized='incremental',
    unique_key='order_id',
    partition_by={
        'field': 'order_date',
        'data_type': 'date',
        'granularity': 'month',
        'copy_partitions': True
    },
    cluster_by=['customer_id', 'status'],
    require_partition_filter=True
) }}

SELECT
    order_id,
    customer_id,
    amount,
    status,
    CAST(order_date AS DATE) AS order_date
FROM {{ source('raw', 'orders') }}
{% if is_incremental() %}
    WHERE order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 2 DAY)
{% endif %}
```

```sql
-- Snowflake clustering
{{ config(
    materialized='table',
    cluster_by=['order_date', 'customer_id'],
    auto_clustering=True
) }}

SELECT * FROM {{ source('raw', 'orders') }}
```

```sql
-- Databricks partitioning
{{ config(
    materialized='incremental',
    partition_by=['order_date'],
    unique_key='order_id',
    file_format='delta'
) }}

SELECT * FROM {{ source('raw', 'orders') }}
{% if is_incremental() %}
    WHERE order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 2 DAY)
{% endif %}
```

### Pros/Cons

| Strategy | Pro | Con |
|---|---|---|
| **Partition by date** | Excellent partition pruning for date-range queries; cheap to maintain | Date column must be present and non-null; partition explosion for high-cardinality dates |
| **Cluster by non-date column** | Faster GROUP BY and FILTER on cluster columns | Maintenance overhead (BigQuery auto-clustering is free; Snowflake credits used) |
| **`require_partition_filter`** | Prevents full-table scans (cost savings) | Queries without partition filter fail — must include partition column in WHERE |
| **Auto-clustering (Snowflake)** | Zero-touch clustering | Costs Snowflake credits; can be slow for continuous incremental loads |
| **Delta partitioning (Databricks)** | Native file pruning; low latency | Partition column must be low-cardinality (date/region) |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Partition column is NULL for some rows | All NULL rows land in a single partition (hot partition) | Ensure partition column is NOT NULL; use COALESCE |
| `granularity: hour` on BigQuery (5B rows/hour) | 5,000+ partitions created; partition limit (4K) hit | Use `granularity: day` or `month` |
| Partition column changes (e.g., `order_date` → `event_date`) | All partitions must be recreated | Full refresh with `--full-refresh` flag |
| Incremental model with `partition_by` on BigQuery — new partitions not picked up | Incremental runs miss new data if `is_incremental()` filter doesn't cover new partitions | Ensure incremental filter includes a window beyond latest partition |

### Follow-Up Questions

**Q1. What is `copy_partitions` in BigQuery partition config?**
→ When set to `True`, dbt copies entire partitions from the incremental model's target table to a temp table before merging — reduces merge time for append-only incremental models (dbt 1.3+).

**Q2. How do you monitor partition pruning?**
→ In BigQuery: `SELECT * FROM table WHERE partition_date = '2024-01-01'` — check query execution details (bytes processed). If bytes processed > partition size, prune failed.

**Q3. Can you partition by multiple columns?**
→ BigQuery: one column (partition), multiple clustering columns. Databricks: multiple partition columns (but high number of partitions can hurt). Snowflake: clustering (not partitioning).

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Partitioning by high-cardinality column (e.g., `user_id`) | Thousands of tiny partitions — poor performance | Partition by date, cluster by user_id |
| Not setting `require_partition_filter` on BigQuery | Users can accidentally query entire table ($$$) | Always set `require_partition_filter=True` for large partitioned tables |
| Setting `cluster_by` on a column that is rarely filtered | Wasted clustering maintenance cost | Only cluster by columns used in WHERE, GROUP BY, or JOIN filters |
| Forgetting `partition_by` for a large incremental model | Full-table scan on each incremental run | Add partition config; use `--full-refresh` to reorganize |

### Principal-Level Considerations

- **Partition strategy as cost governance:** Partitioning is your primary cost-control tool. On BigQuery, a model without `partition_by` scans the entire table every query. Mandate `partition_by` for all tables >10GB. Enforce with a dbt test that inspects `manifest.json` for missing partition configs.
- **Cluster column selection:** Choose cluster columns based on the most common query patterns (75%+ of queries filter on customer_id). Use query audit logs to determine patterns.
- **Partition lifecycle management:** BigQuery limits to 4,000 partitions per table. For date-based partitioning, a daily partition for 11 years fills this limit. Use monthly granularity for long-lived tables.

---

## Q25. How does `persist_docs` work in dbt? When and why would you use it?

### Problem Statement

Explain dbt's `persist_docs` feature — how it pushes model and column descriptions to the database as COMMENT ON TABLE/COLUMN, when to use it, and considerations for different platforms.

### Answer

**`persist_docs`** pushes the descriptions defined in `schema.yml` to the data warehouse as persistent metadata. This makes descriptions visible in BI tools (Tableau, PowerBI, Looker) that read database comments, and in warehouse-native catalogs (SHOW TABLES, INFORMATION_SCHEMA).

### Configuration

```yaml
# dbt_project.yml
models:
  +persist_docs:
    relation: true   # Pushes model description as COMMENT ON TABLE
    columns: true    # Pushes column descriptions as COMMENT ON COLUMN

# Or per model/folder
jaffle_shop:
  marts:
    +persist_docs:
      relation: true
      columns: false  # Only push table docs for marts
```

### What Happens

```yaml
# models/schema.yml
version: 2
models:
  - name: dim_customers
    description: "Customer dimension with lifetime value"
    columns:
      - name: customer_id
        description: "Unique identifier from the CRM system"
      - name: lifetime_value
        description: "Lifetime revenue in USD, calculated from all orders"
```

```sql
-- After dbt run, persist_docs executes (Snowflake):
ALTER TABLE PROD_DB.PROD_SCHEMA.DIM_CUSTOMERS
  MODIFY COMMENT 'Customer dimension with lifetime value';

ALTER TABLE PROD_DB.PROD_SCHEMA.DIM_CUSTOMERS
  MODIFY COLUMN CUSTOMER_ID
  COMMENT 'Unique identifier from the CRM system';

ALTER TABLE PROD_DB.PROD_SCHEMA.DIM_CUSTOMERS
  MODIFY COLUMN LIFETIME_VALUE
  COMMENT 'Lifetime revenue in USD, calculated from all orders';
```

### Pros/Cons

| Pro | Con |
|---|---|
| **BI tool discoverability** — descriptions appear in Tableau/Looker | **DDL execution** — adds time to dbt run (one ALTER TABLE per model) |
| **Single source of truth** — descriptions defined once in dbt, pushed to warehouse | **Platform differences** — some platforms limit COMMENT length (e.g., 256 chars on some systems) |
| **Auditable** — schema.yml is version-controlled, descriptions are visible in warehouse | **Not all platforms supported** — e.g., early Postgres versions |
| **Automated** — no manual step to document warehouse objects | **Descriptions can become stale** if schema.yml changes without a run |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Description exceeds platform character limit | ALTER TABLE fails | Keep descriptions under 256 characters; use `description: "Summary..."` in schema.yml |
| Model is dropped and recreated | Description lost — must be re-pushed with `persist_docs` on next run | Always run with `persist_docs` after full refresh |
| `persist_docs` set on ephemeral model | No effect — ephemeral models have no warehouse object | Only set `persist_docs` on view/table/incremental models |
| Column renamed but description stays | Old column's description still exists in warehouse | Drop old column manually; dbt only pushes for current columns |

### Follow-Up Questions

**Q1. Does `persist_docs` work for seeds and snapshots too?**
→ Yes — seeds and snapshots also support `persist_docs` since they create warehouse objects.

**Q2. Can I selectively push descriptions only for certain models?**
→ Yes — use folder-level or model-level config: `+persist_docs: {relation: true, columns: false}`.

**Q3. How do descriptions appear in the warehouse INFORMATION_SCHEMA?**
→ On Snowflake: `SHOW TABLES` → `comment` column. `SHOW COLUMNS` → `comment` column. These are readable by BI tools natively.

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Not enabling `persist_docs` | Descriptions exist only in schema.yml, invisible to BI tools | Enable at project level: `+persist_docs: {relation: true, columns: true}` |
| Writing descriptions only in model SQL comments | Not visible in warehouse or BI | Move all descriptions to schema.yml and enable `persist_docs` |
| Using `persist_docs` without also maintaining schema.yml | Stale descriptions | Require PR review for any description change |

### Principal-Level Considerations

- **Description hygiene:** `persist_docs` turns schema.yml into the single source of truth for warehouse metadata. Mandate that every model and every column has a description — enforce via a dbt test that inspects `manifest.json` for missing descriptions.
- **BI tool integration:** Enable `persist_docs` before onboarding BI tools. Tableau's "Data Catalog" reads database comments. Without `persist_docs`, BI users see only column names, not their meaning.
- **Description standards:** Define a "good description" standard: business definition, example values, source system, and SLA. Example: "Customer lifetime value (LTV) in USD. Example: 1,250.00. Source: dim_customers.last_ltv. Updated daily at 6am ET."

---

## Q26. How do you handle exceptions, errors, and failures in dbt? (Incremental failure handling)

### Problem Statement

Explain how dbt handles runtime failures — model-level exceptions, incremental model failures (duplicate keys, merge failures), error propagation in the DAG, and strategies for robust pipeline execution.

### Answer

**dbt failure modes and handling strategies:**

| Failure Type | Cause | dbt Behavior | Mitigation |
|---|---|---|---|
| **SQL compilation error** | Invalid SQL syntax, missing column | dbt fails immediately; no model runs | Run `dbt compile` pre-merge |
| **Model failure** | SQL runtime error (e.g., division by zero, constraint violation) | dbt fails that model; downstream skipped | Use `ON_ERROR` in SQL; add tests |
| **Incremental merge failure** | Unique key violation, merge condition mismatch | dbt fails; no partial data | Use `merge_exclude_columns`, `on_schema_change` |
| **Test failure** | Test returns rows (data quality issue) | dbt continues (tests are warnings by default) | Set `--fail-fast` to stop on first test failure |
| **Snapshot failure** | `updated_at` column missing or wrong type | Snapshot fails | Validate snapshot config in CI |
| **Source freshness failure** | Source table older than `warn_after`/`error_after` | Freshness check fails | Set appropriate thresholds |

### Incremental Model Failure Handling

```sql
-- Robust incremental model with error handling
{{ config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='fail',       -- Schema changes cause explicit failure
    full_refresh=False,
    merge_exclude_columns=None
) }}

WITH source AS (
    SELECT *
    FROM {{ source('raw', 'orders') }}
    {% if is_incremental() %}
        WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
    {% endif %}
),

-- Handle potential null/divide-by-zero
validated AS (
    SELECT
        *,
        -- Safe division: avoid divide by zero
        CASE WHEN total_items > 0
            THEN amount / total_items
            ELSE 0
        END AS amount_per_item
    FROM source
    WHERE order_id IS NOT NULL   -- Filter out null primary keys
)

SELECT * FROM validated
```

### Error Propagation in DAG

```
Source → stg_orders (FAIL) → stg_order_items (SKIP, because upstream failed)
                              int_order_metrics (SKIP)
                              fct_orders (SKIP)
                              dim_customers (RUN, because no dependency on stg_orders)
```

```bash
# Re-run only failed models after fixing the issue
dbt run --select result:error
# Or re-run everything from the point of failure
dbt run --select +model_name+
```

### Pros/Cons

| Strategy | Pro | Con |
|---|---|---|
| **`--fail-fast` on tests** | Stops pipeline early on quality failure | May leave downstream models without data |
| **`on_schema_change: fail`** | Explicit failure when schema drifts | Blocks run entirely — must fix schema first |
| **`on_schema_change: append_new_columns`** | Auto-handles additive schema changes | Silently accepts unexpected columns |
| **`result:error` selector** | Re-run only failures | Must know the first failure point |
| **`--full-refresh` after failure** | Clean slate | Destroys existing table — risk of data loss |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Incremental model fails mid-merge (e.g., unique key violation on row 1000 of 10,000) | dbt rolls back the transaction (no partial insert on most platforms) | Use `MERGE` with `WHEN NOT MATCHED`; test unique keys upstream |
| Model fails, but data is partially loaded (no transaction support) | Duplicate or incomplete data in warehouse | Ensure warehouse platform supports DDL transactions (Snowflake does; BigQuery does not) |
| Test failure in CI, but PR still merged | Production data quality issue | Enforce `dbt test` as required check in GitHub branch protection |
| `dbt run` error: `Runtime Error: Database Error: relation already exists` | Concurrent runs from CI and scheduled job | Lock system: use `dbt run --exclusive` or singleton scheduler |
| `is_incremental()` returns `False` on first run (table doesn't exist) | Full refresh runs; incremental logic not applied | Always wrap incremental filter in `{% if is_incremental() %}` |

### Follow-Up Questions

**Q1. What is `on_schema_change` in incremental models?**
→ Controls behavior when the source schema changes: `ignore` (default), `fail` (fail on column change), `append_new_columns` (add new columns), `sync_all_columns` (add/drop/rename). Use `append_new_columns` for production to auto-handle new columns.

**Q2. How do you prevent concurrent dbt runs from colliding?**
→ Use a mutex/lock: `dbt run --exclusive` (dbt Cloud) or a custom lock table (e.g., `SELECT pg_try_advisory_lock()` on Postgres, a lock row in Snowflake).

**Q3. How do you handle a catastrophic failure where the incremental table must be recreated?**
→ `dbt run --full-refresh --select model_name`. This drops the existing table and recreates it from scratch. Plan for downtime.

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| No `on_schema_change` config | Schema drifts silently — column added, but model SQL doesn't use it | Set `on_schema_change: 'append_new_columns'` |
| Not using `{% if is_incremental() %}` correctly | Incremental query runs on first run (when table doesn't exist) | Always wrap incremental filter conditionally |
| Ignoring `result:error` selector | Run full project again instead of just failed models | Use `dbt run --select result:error --defer` |
| Setting `full_refresh: False` globally | Never recreates table — stuck with stale schema | Use `on_schema_change` instead to auto-evolve |

### Principal-Level Considerations

- **Explicit failure handling:** Document a failure response playbook: "If model X fails → check Y → run Z to recover." This eliminates panic-driven recovery steps.
- **`on_schema_change` policy:** Mandate `on_schema_change: 'append_new_columns'` for all production incremental models. This prevents pipeline breaks when source systems add columns (common in event-streaming architectures).
- **Transaction safety audit:** Audit which platforms support DDL transactions (Snowflake does; BigQuery does not). For platforms without transactional DDL, implement a "staging → swap" pattern: build the new table in a shadow schema, then rename.
---

## Q27. What are data contracts in dbt? How do you implement them?

### Problem Statement

Explain the concept of data contracts as implemented in dbt 1.8+ — column-level constraints (NOT NULL, PRIMARY KEY, FOREIGN KEY) enforced at the warehouse level, how to define them, and the operational implications.

### Answer

**Data contracts** (dbt 1.8+) enforce column-level constraints directly in the data warehouse. Previously, dbt tests checked quality after data was loaded. Data contracts prevent invalid data from being written in the first place.

**Key features:**
- `constraints` — Warehouse-native constraints (NOT NULL, PRIMARY KEY, UNIQUE, FOREIGN KEY)
- `data_type` — Explicit column type enforcement (contract runs before data load)
- `model_contract` — Enables or disables the contract for the model
- **Fail-fast** — If data violates a constraint, the model run fails immediately (no partial load)

### Example

```yaml
# models/schema.yml
version: 2
models:
  - name: dim_customers
    config:
      contract:
        enforced: true
    columns:
      - name: customer_id
        data_type: integer
        description: "Unique customer identifier"
        constraints:
          - type: not_null
          - type: primary_key
            name: pk_dim_customers
      - name: email
        data_type: varchar(255)
        constraints:
          - type: not_null
          - type: unique
      - name: signup_date
        data_type: date
        constraints:
          - type: not_null
      - name: country_code
        data_type: varchar(2)
        constraints:
          - type: foreign_key
            expression: "REFERENCES dim_countries(code)"
            name: fk_dim_customers_country
```

```sql
-- Generated DDL (Snowflake):
CREATE TABLE PROD_DB.PROD_SCHEMA.DIM_CUSTOMERS (
    customer_id INTEGER NOT NULL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    signup_date DATE NOT NULL,
    country_code VARCHAR(2) FOREIGN KEY REFERENCES dim_countries(code)
)
```

### Pros/Cons

| Pro | Con |
|---|---|
| **Prevent bad data** — constraints block writes, no stale data | **Platform support varies** — BigQuery has limited constraint support |
| **Fail-fast** — error at INSERT time, not at query time | **Change management** — adding NOT NULL to existing column requires backfill |
| **Self-documenting** — schema IS the contract | **Performance overhead** — constraint checks add write latency |
| **BI tool clarity** — tools see data types directly | **Migration cost** — migrating existing data to meet constraints |

### Edge Cases / Failure Scenarios

| Scenario | Impact | Mitigation |
|---|---|---|
| Existing table has NULLs, NOT NULL constraint added | Constraint violation → model fails | Backfill NULLs before enabling contract |
| Source data violates FK constraint (orphan records) | Foreign key violation → model fails | Clean source data first; use SOFT_ASSERT pattern |
| Platform does not support `PRIMARY KEY` (BigQuery) | Constraint ignored — no enforcement | Write dbt tests as fallback (PK is not enforced on BigQuery) |
| Constraint on `data_type` — column type in source differs from contract | Contract violation → model fails | Ensure source matches contract; type-cast in model SQL |

### Follow-Up Questions

**Q1. What platforms support which constraints?**
→ **NOT NULL** — All. **UNIQUE** — Most (not BigQuery). **PRIMARY KEY** — Snowflake, Postgres, Redshift, Databricks; not BigQuery. **FOREIGN KEY** — Postgres, Redshift; limited on Snowflake/BigQuery. dbt ignores unsupported constraints.

**Q2. How do contracts differ from tests?**
→ Contracts are **preventive** (enforced at write time). Tests are **detective** (check after write). Use both: contracts for hard guarantees (NOT NULL, PK), tests for business rules (positive value, accepted values).

**Q3. Can contracts be temporarily disabled?**
→ Yes — set `contract: {enforced: false}` in schema.yml to skip enforcement. This allows migration runs without rewriting data.

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Enabling contracts on a model with existing NULLs | Every run fails until data is cleaned | Clean data first, then enable contract |
| Defining `PRIMARY KEY` on BigQuery — it's silently ignored | Constraint does nothing | Use `unique` + `not_null` tests instead |
| Not specifying `data_type` in contract | Contract is loose — any type goes | Always declare `data_type:` for every column |
| Using contracts for ephemeral models (no warehouse object) | No effect — ephemeral models have no table | Only set contracts on table/incremental models |

### Principal-Level Considerations

- **Contract sensitivity:** Apply contracts selectively. Core marts (dim_customers, fct_orders) get strict contracts. Staging models get no contracts (source data is inherently messy). Future phase: apply contracts to all "data product" models.
- **Contract enforcement in CI:** In CI, run with contracts `enforced: false`, but in production, enforce: true. Catch contract violations before reaching prod.
- **Data contract as API:** A data contract is the API between the data producer (dbt model) and data consumer (BI dashboard, ML pipeline). Version the contract (v1 → v2) like any API. Breaking changes require consumer migration.

---

## Q28. dbt vs. Airflow: How do they complement each other? (Orchestration vs. Transformation)

### Problem Statement

Compare dbt and Airflow — the responsibilities of each, how they are commonly used together, where they overlap, and architectural patterns for a modern data stack.

### Answer

**dbt** = transformation layer (SELECT). **Airflow** = orchestration layer (WHEN + DEPENDENCIES + EXTERNAL STEPS). They complement, not compete.

| Aspect | dbt | Airflow |
|---|---|---|
| **Primary function** | Data transformation (SQL models) | Workflow orchestration (DAGs of tasks) |
| **What it manages** | Data transformations, testing, docs | Scheduling, dependencies, retries, alerts |
| **Language** | SQL (+ Jinja) | Python |
| **State management** | dbt state + manifest | Airflow DAG runs + task instances |
| **Execution** | `dbt run` (single command) | `DAG run → task instances → sensors` |
| **Granularity** | Model-level (per table) | Task-level (any Python function) |
| **Retries** | Built-in per model (`retries` config) | Built-in per task (`retries`, `retry_delay`) |
| **Alerting** | Via dbt Cloud or CI | Native (Slack, email, PagerDuty) |
| **CI/CD** | dbt Cloud, dbt CLI in CI | Airflow DAG testing (pytest/unit tests) |
| **External dependencies** | Source freshness checks | Sensors (S3, API, SQL, custom) |

### Common Architecture Patterns

**Pattern 1: Airflow orchestrates dbt**
```
Flow: Fivetran syncs raw data → Airflow sensor detects it's done → Airflow triggers dbt run → dbt transforms → Airflow triggers Looker export
```
```python
# Airflow DAG
with DAG('dbt_pipeline', schedule='0 6 * * *') as dag:
    wait_for_fivetran = ExternalTaskSensor('fivetran_sync_dag')
    
    dbt_run = BashOperator(
        task_id='dbt_run',
        bash_command='cd /dbt_project && dbt run --target prod',
    )
    
    dbt_test = BashOperator(
        task_id='dbt_test',
        bash_command='cd /dbt_project && dbt test',
    )
    
    looker_export = BashOperator(
        task_id='looker_export',
        bash_command='looker --export-dashboard',
    )
    
    wait_for_fivetran >> dbt_run >> dbt_test >> looker_export
```

**Pattern 2: dbt Cloud native scheduling (no Airflow)**
```
dbt Cloud → Scheduled runs → dbt triggers → Model DAG → dbt test
```
Simpler but less flexibility: no custom sensors, no Python logic between models.

**Pattern 3: Airflow + dbt Cloud API**
```
Airflow triggers dbt Cloud job via API → dbt Cloud runs the full pipeline → Airflow monitors status
```

### Pros/Cons

| Pattern | Pro | Con |
|---|---|---|
| **Airflow orchestrates dbt** | Full flexibility (sensors, retries, branching) | Operational overhead (Airflow cluster) |
| **dbt Cloud only** | Simpler, less ops overhead | Limited orchestration capabilities |
| **Hybrid** | Best of both (scheduling flexibility + dbt power) | Two systems to manage |

### Edge Cases

| Scenario | Best Approach |
|---|---|
| Need to wait for external file (S3) before running dbt | Airflow sensor → dbt run |
| Need per-model retries and alerts | dbt native `retries` + dbt Cloud alerts |
| Complex branching logic (A→B→C or A→D→E) | Airflow (dbt can't branch at model level) |
| Simple daily pipeline with no external deps | dbt Cloud scheduling (no Airflow needed) |

### Follow-Up Questions

**Q1. Can dbt replace Airflow?**
→ No — dbt is a transformation tool, not a general-purpose orchestrator. It can't wait for external files, run Python steps, or conditionally branch.

**Q2. Should I run dbt inside Airflow's KubernetesPodOperator?**
→ Yes — this isolates dbt from Airflow dependencies and allows versioned environments. Use the `dbt-runner` pattern.

**Q3. What about Dagster vs Airflow for dbt orchestration?**
→ Dagster has native dbt integration (`dagster-dbt`). It can run individual dbt models as Dagster assets with per-model observability — a superior developer experience.

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| Airflow runs `dbt run --select model_name` for each model | 100 tasks for 100 models → slow, hard to manage | Run `dbt run` once (dbt handles DAG ordering internally) |
| Not separating CI/CD from orchestration | CI runs production jobs directly | CI = `dbt compile` + `dbt test`. Orchestration = `dbt run` in Airflow |
| Running dbt in Airflow without state | Every run rebuilds everything | Use `--defer --select state:modified+` in Airflow triggers |

### Principal-Level Considerations

- **Separation of concerns:** dbt owns "what to build" (DAG, transformations, tests). Airflow owns "when to build" (schedule, retries, external dependencies). Don't mix model logic into Airflow DAGs.
- **Airflow as thin wrapper:** The Airflow DAG should be a single `dbt run` (or at most 3–5 dbt commands: build, test, source freshness). Don't micro-manage individual models from Airflow.
- **dbt Cloud as alternative:** For companies without an existing Airflow investment, dbt Cloud's native scheduling is sufficient. Only bring in Airflow when you need external step orchestration or complex branching.

---

## Q29. How do you scale dbt for large projects (500+ models, 20+ engineers)?

### Problem Statement

Explain architectural patterns and operational practices for scaling dbt in large organizations — project organization, CI/CD strategy, performance optimization, team governance, and security.

### Answer

**Scaling dimensions and solutions:**

| Scaling Dimension | Challenge | Solution |
|---|---|---|
| **Model count** | 500+ models → slow compilation, huge DAG | Modular project structure, clean DAG layering |
| **Team size** | 20+ engineers → merge conflicts, naming collisions | Namespaced model paths, semantic naming |
| **Build time** | Full rebuild takes hours | State-based CI/CD, incremental by default |
| **Testing** | 5,000+ tests → CI runs for hours | State-based test selection |
| **Governance** | No ownership clarity → orphan models | Ownership tags, exposures, model review process |
| **Security** | Raw data exposure | Column-level access via grants, schema separation |

### Best Practices

**1. Project structure:**
```
dbt/
├── domains/                    # Domain-based separation
│   ├── revenue/
│   │   ├── models/
│   │   ├── tests/
│   │   └── docs/
│   ├── marketing/
│   └── finance/
├── shared/                     # Shared macros, tests, packages
│   ├── macros/
│   └── packages.yml
└── docs/                       # Project-level docs
```

**2. Naming conventions:**
```yaml
# models/revenue/schema.yml
models:
  - name: revenue_dim_customers     # domain_name_prefix_model
  - name: revenue_fct_orders
    tags: ['revenue', 'core']
    access: public
```

**3. CI/CD for scale:**
```yaml
# .github/workflows/dbt-ci.yml
jobs:
  dbt_ci:
    steps:
      - name: Download prod state
        run: aws s3 cp s3://dbt-state/main/manifest.json target/prod/
      
      - name: Run modified models
        run: |
          dbt build --state target/prod/ --defer --select state:modified+
      
      - name: Upload new state
        run: aws s3 cp target/manifest.json s3://dbt-state/${{ github.ref_name }}/
```

**4. Performance optimization:**
- Use `--defer` + `state:modified+` to avoid full rebuild
- Incremental materialization for all large models
- `dbt compile` once, then `dbt run` (skip re-compile)
- Use `dbt build` instead of `dbt run + dbt test` (more efficient file handling)
- Set `threads: 8` or higher in `profiles.yml`

### Organizational Governance

| Practice | Description |
|---|---|
| **Model ownership** | Each model has a `owner` tag (team name). CI assigns code review based on owner tag. |
| **Change approval** | Cross-team model changes require both teams' approval |
| **SLA definitions** | Source freshness + test pass rates per domain |
| **Documentation reviews** | Quarterly docs audit: all models have descriptions |
| **Orphan model detection** | CI checks: any model without an exposure is flagged |

### Pros/Cons

| Strategy | Pro | Con |
|---|---|---|
| **Domain-based project** | Team autonomy; no merge conflicts | Cross-domain dependencies need coordination |
| **State-based CI/CD** | Fast CI (5 min vs 2 hours) | State management complexity |
| **Model ownership** | Clear accountability | Ownership drift over time |
| **Central packages** | Reusable code, consistent patterns | Package version management overhead |

### Follow-Up Questions

**Q1. What is the practical limit for models in a single dbt project?**
→ ~1,500–2,000 models in one project. Beyond that, compilation time slows significantly and manifest parsing becomes a bottleneck. Split into multiple projects (dbt multi-project architecture) if needed.

**Q2. How do you handle cross-domain dependencies without tight coupling?**
→ Use `cross_ref` (dbt 1.8+) or external models: `{{ ref('revenue', 'fct_orders') }}` to reference another project's model. This adds an explicit cross-project dependency edge.

**Q3. What monitoring and observability do you recommend?**
→ Export `manifest.json` and `run_results.json` to a monitoring DB. Track per-model: build time, bytes processed, test pass rates, freshness. Build dashboards in your BI tool.

### Common Mistakes

| Mistake | Why Wrong | Fix |
|---|---|---|
| One massive `schema.yml` for all models | Hard to review, frequent merge conflicts | One schema.yml per domain folder |
| Giving all engineers write access to all folders | Accidental cross-domain changes | Use folder-level permissions (dbt Cloud) or CODEOWNERS (GitHub) |
| Running full `dbt build` on every PR for 500+ models | CI takes >2 hours | Use state-based selection (`state:modified+`) |

### Principal-Level Considerations

- **Multi-project architecture:** At very large scale (>2K models), split into a "hub and spoke" model: one central dbt project (core dimensions, shared macros) and per-domain projects (revenue-marketing, finance) that `cross_ref` the hub. This gives each domain autonomy and fast CI.
- **dbt Cloud Enterprise features:** Use dbt Cloud's discovery API + environment-specific credentials + folder-level permissions. These are essential for 20+ person teams.
- **Cost tracking per domain:** Allocate Snowflake credits per domain tag. Build a dbt macro that logs `bytes_spilled_to_remote_storage`, `credit_usage` per model. Charge back to domains monthly.

---

## Q30. Top 5 Principal-Level dbt Interview Questions (with model answers)

### Problem Statement

Provide five advanced dbt interview questions that distinguish principal-level engineers from senior engineers, with model answers.

### Question 1: "Your dbt project has 500 models, 3 teams, and CI takes 2 hours. What do you do?"

**Model Answer:**
- **Root cause analysis:** Profile compilation time vs model run time. Likely 90% of CI time is on unchanged models.
- **Immediate fix:** Implement state-based CI/CD using `--state` + `--defer` + `--select state:modified+`. Download main branch manifest from S3, compare, run only modified models + downstream. Reduces CI from 2h → 5min.
- **Organizational fix:** Split into domain-specific projects (revenue, marketing, finance). Each domain runs its own CI. Cross-domain refs use `cross_ref` (dbt 1.8+). Weekly full build to catch physical drift.
- **Monitoring:** Export `run_results.json` to a dashboard. Track per-domain CI time, test pass rate, and model failure rate. Alert when CI time exceeds 30 min.

### Question 2: "A critical source table has a schema change (new column added). How do you handle this without breaking the pipeline?"

**Model Answer:**
- **Prevention:** Set `on_schema_change: 'append_new_columns'` on all incremental models. This auto-adds new columns without breaking the pipeline.
- **Detection:** Source freshness + schema drift detection: run `dbt source freshness` and compare `raw_orders` column count to expected column count.
- **Response:** CI alerts the team (Slack). The team updates the relevant staging models to reference the new column. The change goes through normal code review.
- **Contract layer:** For critical marts, data contracts enforce column types — new columns must match the contract after review.

### Question 3: "Which is better for your use case: dbt Cloud or dbt CLI with Airflow?"

**Model Answer:**
It depends on three factors:
1. **Team size:** 1–10 people → dbt Cloud (simpler). 10+ people → both work; CLI + Airflow gives more control.
2. **Existing infrastructure:** If you already run Airflow for orchestration, keep it and add dbt CLI. If you're starting fresh, dbt Cloud is faster to set up.
3. **Custom orchestration needs:** If you need complex branching (wait for external file → run dbt → run ML model → export), use Airflow. If dbt's own DAG scheduling is enough, use dbt Cloud.
- **For a mid-size company (50 engineers, existing Airflow):** Run dbt CLI in Airflow's KubernetesPodOperator. Use dbt Cloud for IDE + docs (not for scheduling).
- **For a startup (5 engineers, no orchestration):** dbt Cloud only. Add Airflow only when the 3rd orchestration requirement surfaces.

### Question 4: "How do you decide between incremental and table materialization for a 5-billion-row model?"

**Model Answer:**
- **Decision tree:**
  1. Does the model have a monotonically increasing column (date, ID)? → Incremental.
  2. Is the model a dimension (frequently updated, small row count)? → Table (full refresh).
  3. Is the model append-only (events, logs)? → Incremental with `unique_key`.
  4. Is the model recalculated daily (aggregations over entire history)? → Either: incremental with batch-window merge, or table with swap pattern (build in parallel, rename).
- **For 5B-row fact table:** Always incremental. Full rebuild would take 2+ hours and cost $500+ in compute. Set `unique_key`, `partition_by` (daily), and `on_schema_change: 'append_new_columns'`. Backfill using `--full-refresh` only when schema-breaking changes are required (quarterly).
- **Trade-off:** Incremental adds complexity (merge logic, backfill strategy). Accept this complexity for the dramatic cost/time savings.

### Question 5: "Your team wants to adopt dbt for a data mesh architecture. How do you structure it?"

**Model Answer:**
- **Domain-oriented:** Each domain (finance, marketing, revenue) owns its own dbt project. The project is a "data product" with its own tests, documentation, and SLA.
- **Cross-domain sharing:** Two patterns:
  1. **`cross_ref`** (dbt 1.8+) — Domain A's dbt project references Domain B's model via `{{ ref('domain_b', 'dim_customers') }}`. This creates a runtime dependency but keeps domain autonomy.
  2. **Published tables** — Domain B publishes `dim_customers` to a shared schema. Domain A reads it as a source (not a ref). No compile-time dependency.
- **Data contracts:** Each domain publishes its data product with a contract (column names, types, constraints). If the contract breaks, the downstream domain is notified.
- **Governance layer:** Central team maintains `shared-macros` package (naming conventions, testing standards, security policies). Each domain consumes the package but owns its domain-specific logic.
- **Federated CI/CD:** Each domain runs its own CI/CD pipeline. Cross-domain changes require a PR to both projects, with approvals from both domains' data owners.

---

### End of Theory Questions (Q1–Q30)

*This document covers 30 principal-level dbt theory questions for data engineering interviews at companies like Atlassian, Snowflake, and other top-tier organizations. Each question includes a problem statement, answer, example, Pros/Cons, Edge Cases, Follow-ups, Common Mistakes, and Principal Considerations.*