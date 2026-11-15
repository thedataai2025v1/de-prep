# Lesson 14: Evolving Models Based on Changing Requirements & Schema Evolution

> **Principal Mentor Note:** Every data model you design in an interview will be tested against a **"Curveball at Minute 30"**—e.g., *"Now we're launching subscriptions,"* *"Now one order can have multiple payment methods,"* or *"Upstream engineering just renamed a nested JSON struct."* Principal engineers design **extensible schemas** from Day 1 and manage evolution using **backward-compatible migrations, dual-write shadow views, and Iceberg/Delta schema evolution primitives** so zero downstream dashboards break.

---

## 1. The 5 Classic Model Evolution Scenarios (And Exact Refactoring Patterns)

| Evolution Trigger | Before (v1 Schema) | Naive Breaking Change | Principal Zero-Downtime Refactor |
| :--- | :--- | :--- | :--- |
| **1. `1:1` Relationship Becomes `1:Many`**<br>*(e.g., Order had 1 payment method; now supports split payments: Gift Card + Credit Card)* | `payment_method_sk` FK directly on `fct_orders`. | Overwrite `payment_method_sk` with `'MULTIPLE'` (loses card breakdown) or duplicate `fct_orders` rows (corrupts GMV!). | Keep `primary_payment_method_sk` on `fct_orders` for backward compatibility, and introduce **`fct_order_payments`** (grain: 1 row per `(order_id, payment_attempt_seq)`). |
| **2. Single Entity Becomes Multi-Entity / Marketplace**<br>*(e.g., Single-brand store opens up to 3rd-party sellers)* | `fct_orders` has `user_sk` and `product_sk` only; seller is implicitly "Us". | Rewrite `fct_orders` from scratch and force all analysts to change queries. | Add `seller_sk` to `fct_orders` and `dim_product`, backfill historical rows with `seller_sk = 1` (`'First-Party Retail'`), and expose a view alias. |
| **3. Scalar Attribute Becomes Multi-Valued**<br>*(e.g., A video had 1 `genre`; now has up to 5 tags)* | `genre VARCHAR` column on `dim_video`. | Store comma-separated strings `'Action,Sci-Fi'` (violates 1NF; breaks `GROUP BY`). | Keep `primary_genre` on `dim_video` + introduce **`bridge_video_tags`** + `dim_tag` (with `tag_weight = 1.0 / tag_count`). |
| **4. Business Metric Definition Changes**<br>*(e.g., "Active User" changes from `app_open` to `meaningful_action`)* | `is_active_user` boolean computed at ETL write time. | Overwrite historical `is_active_user` silently (causes inexplicable YoY jumps). | Version the metric column (`is_active_user_v1`, `is_active_user_v2`) during a 30-day transition window in the semantic layer, then deprecate `v1`. |
| **5. Upstream Event Adds Rapidly Changing Metadata**<br>*(e.g., Mobile teams add 15 new A/B experiment flags weekly)* | Adding 15 new nullable columns to `fct_events` every sprint. | Constant `ALTER TABLE` churn and wide sparse tables with 400 mostly-NULL columns. | Keep SLA-backed core columns typed, and capture dynamic experiment/feature flags in a typed **`MAP<STRING, STRING>` / `VARIANT`** column (`experiment_context`). |

---

## 2. Physical Schema Evolution on Modern Lakehouses (Iceberg / Delta / Snowflake)

When an interviewer asks *"How do you evolve a 100TB production table safely?"*, demonstrate command of modern table format internals:

### Safe vs. Unsafe Physical Operations
1. **Metadata-Only Evolution (Zero Data File Rewrites — Instant & Safe):**
   * Adding a new nullable column (`ALTER TABLE fct_orders ADD COLUMN tip_usd DECIMAL(10,2)`). Old Parquet files lack the column; Iceberg/Delta readers automatically return `NULL` (or default value in Iceberg v3) without touching a single byte of historical Parquet!
   * Renaming a column or reordering columns in **Apache Iceberg** (because Iceberg tracks columns by immutable **integer field IDs**, not column names or ordinal positions—unlike legacy Hive tables!).
   * Widening numeric types safely (`INT -> BIGINT`, `FLOAT -> DOUBLE`, `DECIMAL(10,2) -> DECIMAL(18,2)`).
2. **Breaking Operations (Requires Expand-Contract Migration):**
   * Narrowing types (`BIGINT -> INT`), changing `STRING -> TIMESTAMP`, or changing a table's **grain** or **partitioning spec** (note: Iceberg supports **Partition Evolution** without rewriting old files, e.g., evolving from `MONTH(ts)` to `DAY(ts)` going forward!).
   * **Expand-Contract Protocol:**
     1. **Expand:** Add new column `amount_usd_v2` (or create new table `fct_orders_v2`), dual-populate both old and new fields in the ETL pipeline.
     2. **Backfill:** Populate historical partitions for `amount_usd_v2` asynchronously.
     3. **Migrate Consumers:** Point BI semantic layer / downstream dbt models to `v2` and monitor query logs (`QUERY_HISTORY`) until `v1` usage hits zero.
     4. **Contract:** Drop `v1` column/table after deprecation SLA expires.

---

## 3. Real Interview Questions & Model Answers

### Q1: "Six months after you launch your E-Commerce Star Schema (`fct_orders`, `fct_order_line_items`), Product launches 'Buy Now, Pay Later' (BNPL) installments AND allows customers to split a single order across multiple shipping addresses. How does your model evolve without breaking existing executive dashboards?"

**Principal-Level Answer:**
> *"Both product launches break a foundational 1:1 assumption in our v1 `fct_orders` table: one payment per order and one shipping address per order. If I simply join `fct_orders` to a new `payments` or `shipments` table in-place, every `SUM(order_gmv)` query on existing dashboards will fan out and double-count revenue.*
>
> *I will evolve the model using **Additive Grain Extension** plus **Backward-Compatible Rollup Fields**:*
> 1. ***Preserve `fct_orders` Grain (1 row per `order_id`):* Keep `fct_orders` intact so 100% of existing GMV, order count, and conversion dashboards continue working with zero SQL changes. Add summary columns (`payment_split_count`, `shipment_count`, `is_bnpl_order`) populated via backward-compatible defaults (`1`, `1`, `FALSE`) for historical rows.*
> 2. ***Introduce `fct_order_payments` (Grain: 1 row per `payment_transaction_id`):* Captures each installment or split tender (`CREDIT_CARD`, `GIFT_CARD`, `KLARNA_BNPL`), `installment_number`, `installment_total`, `authorized_amount_usd`, and `settled_amount_usd`.*
> 3. ***Introduce `fct_shipments` & `fct_shipment_items` (Grain: 1 row per `(shipment_id, order_line_item_id)`):* Connects individual line items to their respective destination `shipping_address_sk`, carrier, and tracking lifecycle without altering the financial grain of `fct_order_line_items`."*

---

## 4. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "How does Apache Iceberg's 'Hidden Partitioning' and 'Partition Evolution' help when data volume grows 20x over 2 years?"**
   * **Expected Insight:** In legacy Hive/Parquet tables, changing partitioning from `YEAR/MONTH` to `YEAR/MONTH/DAY` required creating a brand-new table and rewriting terabytes of historical data, plus forcing every analyst to rewrite their `WHERE` clauses to include the new partition columns. In **Apache Iceberg**, partition transforms (`month(order_ts)` $\rightarrow$ `day(order_ts)`) are stored in metadata. You run `ALTER TABLE fct_orders REPLACE PARTITION FIELD month(order_ts) WITH day(order_ts)`—new data writes into daily partitions immediately while historical data stays untouched in monthly partitions, and queries filtering on `WHERE order_ts >= '2026-01-01'` automatically prune both old and new partitions seamlessly!
2. **Top-Up 2: "Upstream software engineers change a Kafka event field from `discount_cents (INT)` to `discount_amount (STRING '12.50')` without warning. How does your pipeline absorb this without losing data?"**
   * **Expected Insight:** Enforce **Schema Registry contracts** (Avro/Protobuf with `BACKWARD` compatibility) at the producer boundary. In Bronze ingestion, capture raw payloads safely (`rescue` / `_corrupt_record` column in Spark/Snowflake) so malformed events quarantine into a Dead Letter Queue (DLQ) rather than crashing the batch, and normalize both `discount_cents` and `discount_amount` into canonical `discount_usd DECIMAL(12,2)` inside Silver.

---

## 5. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Protects existing fact table grains** by adding new child/sibling fact tables rather than breaking 1:1 assumptions in-place. | Adds a `1:N` join directly into a core fact table and causes silent fan-out across existing BI dashboards. |
| **Applies the Expand-Contract pattern** with query-log deprecation checks for breaking schema changes. | Renames or drops production columns in-place (`ALTER TABLE DROP COLUMN`) assuming nobody uses them. |
| **Leverages modern table format features** (Iceberg field-ID column evolution, partition evolution, `MAP`/`VARIANT` extension columns). | Treats every schema tweak as requiring a full multi-terabyte table drop-and-reload. |