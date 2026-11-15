# Lesson 4: Data Modeling Fundamentals (Normalization, Kimball, Inmon, Data Vault & OBT)

> **Principal Mentor Note:** Religious wars over *"Kimball vs. Inmon vs. Data Vault vs. One Big Table"* are a junior trap. At the Principal level, every modeling paradigm is simply a **different optimization point along the axes of Write Flexibility, Read Ergonomics, Governance, and Engineering Velocity**. Modern cloud data platforms almost always compose multiple paradigms across Medallion layers (`Bronze -> Silver -> Gold`).

---

## 1. Normalization Forms (1NF $\rightarrow$ 3NF / BCNF)

Normalization eliminates **data redundancy** and **update, insert, and delete anomalies** in operational (OLTP) databases.

| Normal Form | Core Rule | Violation Example | How to Fix |
| :--- | :--- | :--- | :--- |
| **1NF (First Normal Form)** | Atomic cell values; no repeating groups or unindexed arrays; unique primary key. | `orders` table has `item_ids = "101, 104, 209"` in one VARCHAR column. | Explode into `order_items(order_id, item_id, quantity)`. |
| **2NF (Second Normal Form)** | Must be in 1NF + no **partial dependencies** (non-key columns must depend on the *entire* composite PK, not just part of it). | `order_items(order_id, product_id, product_name, quantity)` — `product_name` depends only on `product_id`. | Move `product_name` to `products(product_id, product_name)`. |
| **3NF (Third Normal Form)** | Must be in 2NF + no **transitive dependencies** (non-key columns cannot depend on other non-key columns). | `orders(order_id, customer_id, customer_zip, customer_city)` — `customer_city` depends on `customer_zip`, not `order_id`. | Separate `customers` and `zip_codes` lookup tables. |
| **BCNF (Boyce-Codd NF)** | Every determinant must be a candidate key (stricter 3NF for overlapping composite candidate keys). | Instructor teaches only one subject, but a subject has multiple instructors; `(student_id, subject)` $\rightarrow$ `instructor`, while `instructor` $\rightarrow$ `subject`. | Decompose into `(student_id, instructor_id)` and `(instructor_id, subject)`. |

### Why We Intentionally De-Normalize in Analytics
In a 3NF schema, answering *"Total revenue by product category and customer state"* requires joining 6–10 tables (`order_items -> orders -> customers -> addresses -> zip_codes -> products -> subcategories -> categories`). In columnar analytical engines, **denormalizing dimensions** (flattening `product -> subcategory -> category` into `dim_product`) trades minor storage redundancy for:
* Massively fewer shuffles/joins at query time.
* Simpler, bug-free SQL for analysts and BI tools.

---

## 2. Comparing the 4 Major Analytical Paradigms

```
+-------------------+-------------------+-------------------+-------------------+
|   KIMBALL (Star)  |  INMON (CIF/3NF)  |    DATA VAULT     |  ONE BIG TABLE    |
+-------------------+-------------------+-------------------+-------------------+
| Bottom-up by      | Top-down          | Audit-first       | Consumption-first |
| business process  | enterprise 3NF    | insert-only graph | wide flat table   |
|                   | core + data marts | (Hub/Link/Sat)    |                   |
+-------------------+-------------------+-------------------+-------------------+
| Best for:         | Best for:         | Best for:         | Best for:         |
| Core BI / Gold    | Strict enterprise | Regulated / multi-| ML training sets, |
| warehouse layer   | integration       | source ingestion  | high-speed BI     |
+-------------------+-------------------+-------------------+-------------------+
```

### Detailed Trade-Off Matrix

| Architecture | Core Building Blocks | Write / Ingestion Speed | BI Query Performance | Schema Evolution | Best Fit & Modern Lakehouse Role |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Kimball (Dimensional / Star)** | **Fact Tables** (events/metrics) + **Conformed Dimensions** (context) | Moderate (requires surrogate key lookups & SCD `MERGE`s) | **High** (1-hop star joins; optimized by MPP engines) | Moderate | **Gold / Curated Core** for BI, executive reporting, and governed metrics. |
| **Inmon (Corporate Information Factory)** | Enterprise **3NF** warehouse feeding downstream dimensional Data Marts | Slow (complex cross-domain normalization rules) | Low on 3NF core; High on marts | Low (rigid upfront enterprise model) | Rare as a greenfield choice; concepts survive in **Silver canonical entity layers**. |
| **Data Vault 2.0** | **Hubs** (business keys), **Links** (relationships), **Satellites** (attributes + history) | **Very High** (parallel, insert-only, hash-keyed, zero upstream dependencies) | **Low** (requires many joins across Hubs/Links/Sats; needs downstream Star/OBT layer) | **Highest** (add new Satellites without touching existing tables) | Regulated industries (Finance, Healthcare) & **Silver Integration Layer** when merging 15+ messy ERPs. |
| **One Big Table (OBT)** | Single wide table joining fact + all dimension attributes | Fast for append-only; **Very Slow** for retroactive dimension updates | **Highest** for single-table BI engines (ClickHouse, Druid, Tableau extracts) | High (add columns easily), but high duplication cost | **Serving / Consumption Layer** built *on top* of Kimball Gold tables. |

---

## 3. Modern Lakehouse Synthesis: How Principal Engineers Combine Them

In modern architectures (Databricks / Iceberg / Snowflake + dbt), you rarely pick just one. You layer them intentionally:

1. **Bronze (Raw Immutable Log):** Append-only raw events and CDC payloads (`VARIANT` / Parquet) partitioned by `ingestion_date`.
2. **Silver (Integrated Entity & Event Layer):** Deduplicated, typed, cleaned canonical tables. In complex multi-source enterprises, lightweight **Data Vault** or normalized entity models live here.
3. **Gold Core (Kimball Dimensional Model):** Conformed dimensions (`dim_customer`, `dim_product`) and granular fact tables (`fct_orders`, `fct_payments`) acting as the **single source of truth**.
4. **Gold Serving / Marts (OBT & Rollups):** Wide denormalized tables (`obt_order_analytics`) and pre-aggregated summary tables tailored for specific BI tools or ML feature stores.

---

## 4. Real Interview Questions & Model Answers

### Q1: "Our company just acquired two competitors, so we now have three different ERP systems with conflicting customer and product schemas. Should we model the warehouse using Kimball, Data Vault, or OBT?"

**Principal-Level Answer:**
> *"I would use a **hybrid two-tier architecture: Data Vault in the Silver integration layer, feeding a Kimball Star Schema in the Gold layer**.*
>
> *Here is why: During post-merger integration, source schemas change weekly and business rules for deduplicating customers across the three ERPs will be debated for months. If we force a Kimball model directly from raw ingestion, every rule change forces us to rebuild surrogate keys and rewrite historical fact tables. With **Data Vault 2.0** in Silver, each ERP loads independently in parallel into hash-keyed **Hubs** (`hub_customer`), **Links** (`link_customer_order`), and **Satellites** (`sat_customer_erp_a`, `sat_customer_erp_b`) using insert-only patterns—giving us 100% auditability and lineage.*
>
> *Then, we isolate all master data management (MDM) survivorship rules in a **Business Vault / Same-As Link**, and project a clean, unified **Kimball Star Schema** (`dim_customer`, `fct_orders`) in Gold so analysts never have to write 12-way Data Vault joins."*

---

### Q2: "Give me a concrete SQL example showing why mixing grains in a single table causes silent metric corruption."

**Principal-Level Answer:**
> *"Suppose we join `orders` (Order grain) and `order_items` (Line-item grain) into a single wide table `obt_order_items` and include `order_shipping_fee` ($10.00) on every row:*

```sql
-- TABLE: obt_order_items (Grain: 1 row per order_item)
-- order_id | item_id | item_price | order_shipping_fee
-- 1001     | A1      | 50.00      | 10.00
-- 1001     | A2      | 30.00      | 10.00

-- THE TRAP: Analyst runs a standard SUM()
SELECT
    SUM(item_price)         AS total_item_revenue, -- 80.00 (Correct)
    SUM(order_shipping_fee) AS total_shipping      -- 20.00 (WRONG! Double-counted $10)
FROM obt_order_items
WHERE order_id = 1001;
```

> *To fix this at the Principal level, you have two choices:*
> 1. *Keep `order_shipping_fee` strictly in an order-grain fact table (`fct_orders`), OR*
> 2. *Allocate shipping down to the line-item grain deterministically (e.g., proportional to item price: `$6.25` and `$3.75`) so that `SUM(allocated_shipping_fee)` equals `$10.00` at every slice."*

---

## 5. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "In Data Vault, why do we use deterministic hashes (e.g., SHA-256 of business key) instead of auto-incrementing integer surrogate keys?"**
   * **Expected Insight:** Auto-incrementing keys require central sequence coordination and parent-first loading (you must load `dim_customer` before `fct_orders` to look up the integer SK). Hash keys allow **parallel, asynchronous loading** of Hubs, Links, and Satellites simultaneously across distributed compute engines (Spark/Snowflake) without lookup bottlenecks. The trade-off is larger key size (16–32 bytes vs. 4–8 bytes) and slightly slower join/index performance in BI engines—which is why we convert back to compact integer/bigint keys or clean Star Schemas in Gold.
2. **Top-Up 2: "What is the 'Chasm Trap' vs. 'Fan Trap' in dimensional modeling?"**
   * **Expected Insight:**
     * **Fan Trap:** A `1 : Many` relationship leads to another `1 : Many` relationship (`Customer -> Orders -> Order_Items`), causing measures on the middle entity (`Orders.shipping_fee`) to be multiplied when joined all the way down.
     * **Chasm Trap:** Two independent `Many : 1` fact tables converge on a single shared dimension (`fct_impressions -> dim_campaign <- fct_clicks`), and joining both facts directly through the dimension creates a Cartesian product per campaign. Fix: Pre-aggregate each fact to the `campaign_id` grain *before* joining, or use a `UNION ALL` / consolidated fact pattern.

---

## 6. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Treats modeling paradigms as composable layers** across Bronze/Silver/Gold rather than mutually exclusive religions. | Argues dogmatically that *"Kimball is dead because of BigQuery"* or *"OBT is always bad"*. |
| **Spots Fan Traps and Chasm Traps immediately** when reviewing multi-fact joins. | Joins two fact tables together directly on a shared dimension key without pre-aggregation. |
| **Understands the operational cost of hash keys vs. integer surrogate keys** at multi-billion-row scale. | Uses 64-character hex string hashes as join keys across 50-billion-row BI tables without considering join shuffle overhead. |