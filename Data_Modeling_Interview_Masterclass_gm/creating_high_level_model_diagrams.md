# Lesson 13: Creating High-Level Model Diagrams (Bus Matrix & Entity Relationships)

> **Principal Mentor Note:** In a 45-minute interview, candidates who jump straight into writing 20 `CREATE TABLE` scripts on a blank screen lose the interviewer within 3 minutes. Principal Data Engineers communicate **top-down**: first an **Enterprise Bus Matrix** (aligning business processes to conformed dimensions), then a **High-Level Conceptual Star/Constellation Diagram** (showing cardinality, grain, and join paths), and *only then* detailed column definitions and physical DDLs.

---

## 1. The Three-Tier Visual Communication Stack

Always draw your model in three progressive layers so both product-minded and infra-minded interviewers can follow effortlessly:

```
[Tier 1: Enterprise Bus Matrix]  --> Proves cross-domain governance & conformed dimensions (2 mins)
        |
        v
[Tier 2: Star / Constellation ERD] --> Shows fact grains, cardinality (1:N, M:N), & role-playing keys (4 mins)
        |
        v
[Tier 3: Physical DDL & Layout]   --> Specifies types, surrogate keys, SCD2 columns, & partitioning (8 mins)
```

---

## 2. Artifact 1: The Kimball Enterprise Bus Matrix

The **Bus Matrix** is the single highest-signal diagram you can draw at minute 8 of an interview. Rows represent **Core Business Processes** (which become Fact tables); columns represent **Conformed Dimensions**.

### Example: Multi-Sided Marketplace Bus Matrix
```
Business Process (Fact Table)        | Date | Time | User | Merchant | Courier | Geo | Promo | Device
-------------------------------------+------+------+------+----------+---------+-----+-------+-------
fct_search_impressions (Transaction) |  X   |  X   |  X   |    X     |         |  X  |       |   X
fct_orders (Transaction)             |  X   |  X   |  X   |    X     |         |  X  |   X   |   X
fct_order_line_items (Transaction)   |  X   |  X   |  X   |    X     |         |  X  |   X   |
fct_delivery_legs (Transaction)      |  X   |  X   |  X   |    X     |    X    |  X  |       |
fct_order_lifecycle (Accum. Snap)    |  X   |  X   |  X   |    X     |    X    |  X  |       |
fct_merchant_daily_snap (Periodic)   |  X   |      |      |    X     |         |  X  |       |
```

### Why Interviewers Love the Bus Matrix
1. **Prevents Siloed Data Marts:** Proves that `dim_merchant` used by Marketing in `fct_search_impressions` is the **exact same conformed table** used by Finance in `fct_orders`.
2. **Enables Drill-Across Queries:** Because `Date`, `Merchant`, and `Geo` conform across rows, you can safely aggregate impressions from Fact A and revenue from Fact B separately to `merchant_sk` and join them cleanly without a Chasm Trap.
3. **Scopes Iterative Delivery:** You can circle rows 2 and 3 and say: *"Let's build out `fct_orders` and `fct_order_line_items` in full detail first, and keep the other processes in view for phase 2."*

---

## 3. Artifact 2: High-Level Star & Constellation Diagram Notation

When sketching ERDs in text/whiteboard tools (Excalidraw, Miro, CoderPad, Google Docs), use crisp **Crow's Foot / Cardinality Notation** and explicitly label:
* **`[FACT - TXN]`** (Transaction Fact), **`[FACT - PER]`** (Periodic Snapshot), **`[FACT - ACC]`** (Accumulating Snapshot), **`[BRIDGE]`** (Many-to-Many Bridge)
* **`[DIM - SCD1]`**, **`[DIM - SCD2]`**, **`[DIM - MINI]`**
* **Role-Playing Arrows** (e.g., `ordered_date_sk` and `delivered_date_sk` both pointing to `dim_date`)

```
                       +----------------------+
                       |   dim_date [CONFORM] |
                       | PK: date_sk (INT)    |
                       +----------+-----------+
                            ^            ^
         (ordered_date_sk)  |            |  (delivered_date_sk)
                            |            |
+------------------+   1:N  |            |  N:1   +-----------------------+
| dim_user [SCD2]  |------->+------------+------->| dim_merchant [SCD2]   |
| PK: user_sk      |        | fct_orders |        | PK: merchant_sk       |
| NK: user_id      |        | [FACT-TXN] |        | NK: merchant_id       |
+------------------+        | Grain: 1   |        +-----------------------+
                            | row/order  |                    | 1
+------------------+   1:N  |            |                    |
| dim_promo [SCD1] |------->| PK:order_id|                    | N (M:N via Bridge)
| PK: promo_sk     |        +-----+------+        +-----------v-----------+
+------------------+              | 1             | bridge_merchant_tag   |
                                  |               | FK: merchant_sk,tag_sk|
                                  | N             +-----------------------+
                     +------------v------------+
                     | fct_order_line_items    |
                     | [FACT-TXN]              |
                     | Grain: 1 row/order_item |
                     +-------------------------+
```

---

## 4. Handling Diagramming Anti-Patterns

| Anti-Pattern | Why It Hurts Your Interview Score | Principal Fix |
| :--- | :--- | :--- |
| **Spiderweb / Hairball Diagram** | Drawing lines between every dimension and every other dimension (over-snowflaking). | Keep dimensions flat (Star Schema); only use an **Outrigger** when justified (e.g., `dim_user_pii` for GDPR). |
| **Unlabeled Lines** | Drawing lines without `1:N` or `M:N` cardinality markers or FK names. | Always annotate `1` and `N` on every edge, and label role-playing FKs explicitly. |
| **Fact-to-Fact Direct Joins** | Drawing a `1:N` line from `fct_orders` directly to `fct_payments` in BI diagrams. | Show that both facts connect via **Conformed Dimensions** (`dim_user`, `dim_date`) or share a **Degenerate Dimension** (`order_id`), and pre-aggregate before joining. |
| **Missing Grain Labels** | Naming a box `Orders` without specifying whether a row is an order header, line item, or status event. | Write `Grain: 1 row per <entity>` directly inside every fact box. |

---

## 5. Real Interview Questions & Model Answers

### Q1: "When drawing a high-level model diagram, should you ever draw a relationship directly between two Fact tables—for example, `fct_orders` and `fct_order_line_items`?"

**Principal-Level Answer:**
> *"In pure Kimball dimensional theory, fact tables don't join directly to one another; they drill-across via conformed dimensions. However, in modern analytical practice—especially when we have a **Header-Item** pair (`fct_orders` at the order grain and `fct_order_line_items` at the line-item grain)—both tables carry `order_id` as a **Degenerate Dimension**.*
>
> *On my diagram, I represent `order_id` as a shared degenerate key rather than a standard dimension foreign key, and I explicitly warn against joining `fct_orders` to `fct_order_line_items` in raw BI queries without pre-aggregating `fct_order_line_items` to `order_id` first—otherwise, order-level header metrics like `delivery_fee` will fan out and double-count."*

---

## 6. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "How do you represent Snowflake vs. Star Schema trade-offs on a high-level diagram when an interviewer asks why you flattened `Category -> Subcategory -> Product`?"**
   * **Expected Insight:** Show the 3-table normalized chain (`product -> subcategory -> category`) crossed out or labeled *"Silver 3NF"* feeding a single denormalized `dim_product` in *"Gold Star"*. Explain that flattening eliminates 2 runtime joins per query, simplifies BI tool metadata, and costs negligible storage since `dim_product` has 10M rows vs. 10B rows in `fct_order_line_items`.
2. **Top-Up 2: "How do you visually model a hierarchical self-referencing entity—like an Employee Org Chart or Parent/Child Product Categories—on a high-level diagram?"**
   * **Expected Insight:** Never rely on recursive `parent_id` self-joins in an analytical Star Schema (BI tools like Tableau/Looker struggle with recursive CTEs). Show either a **Fixed-Depth Flattened Hierarchy** (`level_1_vp`, `level_2_director`, `level_3_manager`) inside `dim_employee` or a **Closure / Hierarchy Bridge Table** (`bridge_org_hierarchy(ancestor_sk, descendant_sk, depth_levels, is_leaf)`).

---

## 7. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Draws an Enterprise Bus Matrix first** to align business processes and conformed dimensions. | Draws 15 boxes randomly across the canvas with tangled crossing arrows. |
| **Labels table archetypes (`[FACT-TXN]`, `[DIM-SCD2]`) and exact grain** inside every box. | Draws generic boxes with just column lists and no indication of grain or SCD behavior. |
| **Explicitly annotates role-playing foreign keys** (`ordered_date_sk`, `shipped_date_sk`). | Draws 4 separate `dim_date` tables or leaves ambiguous `date_id` FKs. |