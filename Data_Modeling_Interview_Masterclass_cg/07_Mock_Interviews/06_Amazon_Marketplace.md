# Principal Data Engineer Mock Interview — E-commerce Marketplace

**Level:** Senior / Staff / Principal Data Engineer  
**Duration:** 90 minutes (plus 30–45 minutes for SQL exercise)  
**Case type:** End-to-end source-to-warehouse data modeling  

## How to use this case in an interview

Treat this as a 75–90 minute interview, not a schema memorization exercise. First state assumptions, then ask clarifying questions, then inventory business processes and source systems. Do not design a single universal wide table.

**Important:** The source schemas below are realistic, intentionally simplified teaching fixtures. They are not representations of private production schemas at the named companies. Source-system tables are usually normalized for transactional writes; warehouse facts and dimensions are designed for analytical access.

## 1. Interview prompt and success criteria

> A multi-country marketplace needs consistent reporting for catalog, sellers/offers, orders, order lines, split shipments, delivery, inventory, promotions, returns, refunds, seller settlement, GMV and net sales.

A strong Principal-level answer must demonstrate:
- ability to clarify ambiguous business language and establish metric contracts;
- understanding of operational source entities, keys, cardinalities and update semantics;
- grain-first modeling and separation of independent business processes;
- correct historical, event-time, financial and temporal semantics;
- robust ingestion, CDC, deduplication, late-arrival and replay strategy;
- scalable physical design, observability, governance and cross-team ownership.

## 2. Interview conversation: requirement discovery

Use this as a natural conversation, not a checklist recitation.

| Candidate asks | Interviewer response / working assumption | Design consequence |
|---|---|---|
| Who consumes the data? | Product, operations, finance, leadership and data science. | Separate operational freshness marts from certified financial reporting. |
| What decisions are made? | Funnel, operational performance, customer outcomes and financial reconciliation. | Model multiple business processes, not one fact. |
| What is the authoritative definition of the core entity? | The source domain owner owns the canonical identifier and lifecycle. | Preserve source IDs and define cross-system identity resolution. |
| Are source rows overwritten or append-only? | Both patterns exist. | Combine CDC current-state capture with immutable event history. |
| Can one parent have multiple children or attempts? | Yes; retries, reassignment, partial fulfillment and adjustments occur. | Explicit one-to-many / many-to-many tables; never assume one-to-one. |
| Do attributes change historically? | Yes, some dimensions and policies change. | SCD Type 2 or effective-dated reference data where past attribution matters. |
| What are freshness and availability expectations? | Operational aggregates in minutes; certified finance in daily close windows. | Distinct SLAs, pipelines and quality gates. |
| What is the expected scale? | High-volume events plus lower-volume entities. | Separate hot event facts, snapshots, aggregates and raw retention tiers. |
| What is the meaning of a reporting day? | Local business day for user-facing operations; UTC for event transport. | Store UTC instants plus business timezone/date derivations. |
| What corrections/deletions are required? | Late corrections, privacy deletion and financial reversals occur. | Versioned corrections, tombstones, lineage and policy-driven retention. |

**Candidate synthesis:** “I will model the core entities and lifecycle events independently, preserve source identifiers and event history, then publish conformed dimensions and facts at explicit grains. Before finalizing, I will validate every metric against its source-of-truth definition and reconciliation owner.”

## 3. Source-system inventory and source table structures

The following tables are the starting operational layer. For each source, identify: primary key, foreign keys, row grain, mutable columns, event timestamps, ingestion/update timestamps, deletion semantics and expected CDC behavior.

### `catalog.products`
**Grain / purpose:** One row per catalog product/ASIN; product identity independent of seller.

```text
product_id PK | title | brand_id | category_id | marketplace_id | created_at | status | updated_at
P100 | Wireless Headphones | B8 | C12 | IN | 2025-01-01 | active | 2026-11-14
```

### `marketplace.seller_skus`
**Grain / purpose:** One row per seller-owned SKU mapping to catalog product.

```text
seller_sku_id PK | seller_id FK | product_id FK | seller_sku_code | condition | created_at | status
SS1 | S10 | P100 | WH-001 | new | 2025-05-01 | active
```

### `marketplace.offers`
**Grain / purpose:** One row per seller offer; current price/fulfillment attributes.

```text
offer_id PK | seller_sku_id FK | marketplace_id | fulfillment_type | current_price_minor | currency | offer_status | updated_at
OF1 | SS1 | IN | FBA | 499900 | INR | buyable | 2026-11-14
```

### `orders.order_headers`
**Grain / purpose:** One row per customer order; header-level amounts and status.

```text
order_id PK | customer_id | marketplace_id | ordered_at | order_status | currency | shipping_minor | order_tax_minor | order_discount_minor | order_total_minor
O900 | CU1 | IN | 2026-11-14 10:00 | partially_shipped | INR | 5000 | 45000 | 10000 | 539000
```

### `orders.order_lines`
**Grain / purpose:** One row per order line; product/offer and quantity.

```text
order_line_id PK | order_id FK | line_no | offer_id FK | ordered_qty | unit_price_minor | line_discount_minor | line_tax_minor | line_status
OL1 | O900 | 1 | OF1 | 1 | 499900 | 10000 | 45000 | shipped
OL2 | O900 | 2 | OF2 | 2 | 20000 | 0 | 0 | unshipped
```

### `fulfillment.shipments`
**Grain / purpose:** One row per shipment/package.

```text
shipment_id PK | order_id FK | fulfillment_center_id | carrier | shipped_at | delivered_at | shipment_status
SH1 | O900 | FC1 | CARR1 | 2026-11-14 18:00 | NULL | in_transit
SH2 | O900 | FC2 | CARR2 | 2026-11-15 09:00 | NULL | created
```

### `fulfillment.shipment_lines`
**Grain / purpose:** One row per shipment × order line allocation.

```text
shipment_line_id PK | shipment_id FK | order_line_id FK | shipped_qty | unit_cost_minor
SL1 | SH1 | OL1 | 1 | 300000
```

### `inventory.inventory_movements`
**Grain / purpose:** One row per stock movement event.

```text
movement_id PK | offer_id FK | fulfillment_center_id | movement_type | quantity_delta | occurred_at | reference_type | reference_id
IM1 | OF1 | FC1 | receipt | 20 | 2026-11-10 | po | PO1
IM2 | OF1 | FC1 | pick | -1 | 2026-11-14 | shipment | SH1
```

### `inventory.inventory_snapshots`
**Grain / purpose:** One row per offer × FC × condition × snapshot time.

```text
snapshot_id PK | offer_id FK | fulfillment_center_id | snapshot_ts | sellable_qty | reserved_qty | damaged_qty | in_transit_qty
IS1 | OF1 | FC1 | 2026-11-14 23:59 | 19 | 0 | 0 | 0
```

### `returns.return_lines`
**Grain / purpose:** One row per return authorization × order line.

```text
return_line_id PK | order_line_id FK | requested_qty | received_qty | disposition | reason | requested_at | received_at
RL1 | OL1 | 1 | 1 | restock | defective | 2026-11-20 | 2026-11-22
```

### `finance.refund_transactions`
**Grain / purpose:** One row per refund transaction; may be partial or no physical return.

```text
refund_id PK | order_id FK | order_line_id nullable | amount_minor | currency | reason | status | created_at | settled_at
RF1 | O900 | OL1 | 499900 | INR | return | succeeded | 2026-11-22 | 2026-11-22
```

### Source profiling questions the interviewer may ask

1. Which source is authoritative when two systems disagree?
2. Is the primary key globally unique or only unique within tenant/marketplace/country?
3. Can a natural key be reused after deletion or account closure?
4. Does `updated_at` change on every mutation, and is it reliable enough for incremental extraction?
5. Are deletes hard deletes, soft deletes, tombstones or status transitions?
6. Are timestamps UTC, local time or timezone-naive?
7. Is a child table append-only, mutable, or both?
8. What happens when the parent arrives after the child due to asynchronous ingestion?
9. Which fields are PII, payment-sensitive, location-sensitive or otherwise restricted?
10. What source-side constraints are real guarantees versus application conventions?

## 4. Source ER model and cardinality

```text
Catalog product, seller, seller SKU, offer, customer, order, order line, shipment, shipment line, fulfillment center, inventory movement, inventory snapshot, promotion, return line, refund, seller settlement.
```

State cardinality explicitly. A parent-to-child relationship is not a reason to flatten the child into the parent. For every relationship, discuss optionality, lifecycle, deletion behavior and whether the child can exist before the parent is visible in the warehouse.

## 5. Source-to-warehouse mapping

| Source object | Source grain | Target model | Transformation / modeling decision |
|---|---|---|---|
| `products` | One row per catalog product/ASIN | Silver `products` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `seller_skus` | One row per seller-owned SKU mapping to catalog product. | Silver `seller_skus` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `offers` | One row per seller offer | Silver `offers` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `order_headers` | One row per customer order | Silver `order_headers` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `order_lines` | One row per order line | Silver `order_lines` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `shipments` | One row per shipment/package. | Silver `shipments` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `shipment_lines` | One row per shipment × order line allocation. | Silver `shipment_lines` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `inventory_movements` | One row per stock movement event. | Silver `inventory_movements` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `inventory_snapshots` | One row per offer × FC × condition × snapshot time. | Silver `inventory_snapshots` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `return_lines` | One row per return authorization × order line. | Silver `return_lines` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |
| `refund_transactions` | One row per refund transaction | Silver `refund_transactions` → case-specific facts/dimensions | Preserve source key, timestamps, operation/version; normalize enums and validate references. |

Mapping rules:
- Preserve the source natural key and the source system identifier in the integration layer.
- Generate warehouse surrogate keys for dimensions when historical versions are needed.
- Do not silently drop orphan records; route them to an inferred/unknown member and reconcile later.
- Preserve raw payloads and source metadata in the bronze layer; do not treat a parsed current-state table as the only history.
- Keep header, line, event, snapshot and financial posting grains separate.

## 6. Target dimensional model

### Proposed facts

| Fact | Grain | Key attributes / measures |
|---|---|---|
| `fact_order` | One row per order header. | order ID, customer, marketplace, order timestamp, header shipping/tax/discount/total; never sum after line fanout. |
| `fact_order_line` | One row per order × line. | line ID, order, offer/product/seller, quantity, unit price, line discount/tax, line amount. |
| `fact_shipment_line` | One row per shipment × order line allocation. | shipment, order line, shipped quantity, ship/delivery timestamps; supports partial and split shipments. |
| `fact_inventory_movement` | One row per inventory movement. | movement ID, offer, FC, movement type, signed quantity, timestamp, reference. |
| `fact_inventory_snapshot` | One row per offer × FC × condition × snapshot time. | sellable/reserved/damaged/in-transit balances; semi-additive across time. |
| `fact_return_line` | One row per return authorization × order line. | requested/received quantities, disposition, reason, event dates. |
| `fact_refund_transaction` | One row per refund transaction. | refund ID, order/line, signed amount, currency, refund status and dates. |

### Conformed dimensions

dim_product (Type 2 where category/brand reporting needs historical attribution); dim_seller; dim_offer; dim_customer; dim_marketplace/country; dim_fulfillment_center; dim_carrier; dim_promotion; dim_return_reason; dim_date with order/ship/delivery/return/refund roles; dim_currency.

### Star-schema relationship sketch

```text
                dim_date (role-playing)
                      |
dim_actor ---- fact_core_process ---- dim_product_or_service
     |                |               |
dim_geo ------------- | -------------- dim_status
                      |
                dim_reason

Independent child/event/financial facts share conformed dimensions
but are not joined raw at incompatible grains.
```

The sketch is deliberately generic: use the case-specific fact list above to draw separate stars. A conformed dimension shares meaning and keys across facts; it does not mean all facts should be physically joined into one table.

## 7. Physical source DDL: representative operational schema

Below is an illustrative relational source definition for the main entity and one child table. It is not the warehouse DDL.

```sql
-- Operational source example; syntax is PostgreSQL-like.
CREATE TABLE core_entity (
    entity_id       VARCHAR(64) PRIMARY KEY,
    actor_id        VARCHAR(64) NOT NULL,
    business_status VARCHAR(32) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    source_version  BIGINT NOT NULL DEFAULT 1
);

CREATE TABLE core_entity_event (
    event_id        VARCHAR(64) PRIMARY KEY,
    entity_id       VARCHAR(64) NOT NULL,
    event_type      VARCHAR(64) NOT NULL,
    event_ts        TIMESTAMPTZ NOT NULL,
    source_sequence BIGINT,
    payload         JSONB,
    CONSTRAINT fk_event_entity
      FOREIGN KEY (entity_id) REFERENCES core_entity(entity_id)
);
```

Discuss why the OLTP schema may be normalized and why the warehouse should not blindly copy its table layout. At production scale, source constraints may not be enforced in every system; validate rather than assume.

## 8. Warehouse DDL: fact grain and dimensional keys

```sql
-- Illustrative analytical target. Adapt types, identity strategy and syntax
-- to Snowflake, Redshift, Databricks/Delta or the chosen warehouse.
CREATE TABLE dim_actor (
    actor_key       BIGINT NOT NULL,
    actor_id        VARCHAR(64) NOT NULL,
    segment         VARCHAR(64),
    region          VARCHAR(64),
    valid_from      TIMESTAMP NOT NULL,
    valid_to        TIMESTAMP NOT NULL,
    is_current      BOOLEAN NOT NULL,
    source_system   VARCHAR(64),
    record_hash     VARCHAR(128)
);

CREATE TABLE fact_core_process (
    process_key       BIGINT NOT NULL,
    process_id        VARCHAR(64) NOT NULL,
    actor_key         BIGINT,
    process_date_key  INTEGER,
    created_ts        TIMESTAMP NOT NULL,
    terminal_ts       TIMESTAMP,
    process_status    VARCHAR(32),
    source_updated_ts TIMESTAMP,
    warehouse_load_ts TIMESTAMP NOT NULL
);
```

**Do not copy this generic DDL as your answer.** Replace `actor`, `process` and the generic measures with the case entities, and state the grain in a sentence before adding columns. The key design is: one fact row corresponds to exactly one declared business event/process instance at that grain.

## 9. Transformation and loading strategy

### Layered architecture

```text
Operational DBs / APIs / Event producers
        |
        +--> CDC (snapshot + change log) ----+
        |                                    |
        +--> Domain event streams -----------+--> Bronze: immutable raw + metadata
        |                                    |
        +--> Periodic source snapshots ------+ 
                                             |
                                  Silver: typed, deduped, conformed
                                  - source key normalization
                                  - event ordering and version resolution
                                  - data quality and quarantine
                                  - identity and SCD processing
                                             |
                                  Gold: facts, dimensions, aggregates
                                  - certified metric definitions
                                  - finance/operations marts
                                  - access-controlled data products
```

### Required ingestion metadata

Every landed record should carry, where available:
`source_system`, `source_table_or_topic`, `source_primary_key`, `source_event_id`, `source_version`, `source_commit_ts`, `event_ts`, `ingested_at`, `operation_type` (insert/update/delete), `schema_version`, `kafka_topic`, `kafka_partition`, `kafka_offset`, `batch_id` and `payload_hash`.

Do not confuse Kafka offset with a business event ID. Offsets are transport coordinates scoped to a topic-partition. Use source event IDs, entity versions or deterministic business keys for idempotency.

### Incremental loading pattern

1. Bootstrap with a consistent snapshot or source-supported snapshot/CDC handoff.
2. Capture changes with transaction/commit metadata where supported.
3. Land raw changes append-only.
4. Deduplicate by source identity and version; quarantine conflicting duplicates.
5. Apply deletes as tombstones or governed deletion actions, not accidental disappearance.
6. Build current-state silver tables using deterministic latest-version logic.
7. Build event facts from immutable event sources; never synthesize historical events from a current row unless clearly labeled as inferred.
8. Process late arrivals with event-time lookback/replay windows.
9. Reconcile source counts, key counts, financial totals and state distributions.
10. Publish gold tables only after critical quality checks pass.

## 10. Worked data problem: from source rows to facts

Use the sample source records above. Explain, step by step:
- which source records represent separate business events;
- which IDs are natural keys and which are warehouse surrogate keys;
- which source rows are updates versus new facts;
- which attributes need Type 2 history;
- which measures can be summed and which cannot;
- what happens when a child row arrives before its parent;
- how corrections and reversals are represented;
- how the target fact rows are produced without fanout.

### SQL pattern: latest source state

```sql
WITH ranked AS (
  SELECT s.*,
         ROW_NUMBER() OVER (
           PARTITION BY source_system, source_pk
           ORDER BY source_version DESC, source_commit_ts DESC, ingested_at DESC
         ) AS rn
  FROM silver_source_changes s
)
SELECT *
FROM ranked
WHERE rn = 1
  AND operation_type <> 'DELETE';
```

This pattern is valid only if the ordering columns represent a trustworthy source version order. If source versions can tie or are absent, define a deterministic tie-breaker and document the limitation.

### SQL pattern: prevent fanout

```sql
WITH child_agg AS (
  SELECT parent_id, COUNT(*) AS child_count
  FROM fact_child_event
  GROUP BY parent_id
)
SELECT p.parent_id,
       p.parent_amount,
       COALESCE(c.child_count, 0) AS child_count
FROM fact_parent p
LEFT JOIN child_agg c
  ON p.parent_id = c.parent_id;
```

Never sum a parent-grain measure after joining to multiple child rows unless it has first been aggregated to a compatible grain.

## 11. Metric contracts and reconciliation

| Metric | Grain / numerator | Denominator / exclusions | Time basis | Owner / reconciliation |
|---|---|---|---|---|
| Primary volume metric | Distinct qualifying business IDs | Eligible population, not raw rows | Event/business date | Product or operations |
| Conversion | Distinct entities reaching defined outcome | Eligible entities entering funnel | Cohort or event date | Product analytics |
| Time-to-outcome | Outcome timestamp − start timestamp | Exclude invalid/negative durations; define pause policy | Elapsed or business time | Operations |
| Financial amount | Signed postings or allocated transaction amounts | Currency and status scope explicit | Posting/settlement date | Finance |
| Current balance/state | Latest valid snapshot at an as-of time | No summing across snapshot dates | Snapshot timestamp | Operations/finance |

For every metric, document: business definition, SQL owner, source-of-truth table, numerator, denominator, filters, timezone, currency, late-arrival policy, correction policy, refresh SLA and expected reconciliation tolerance.

## 12. Data quality, failure modes and recovery

| Failure mode | Detection | Handling |
|---|---|---|
| Duplicate delivery | Duplicate source event ID / same entity version | Idempotent merge; retain raw evidence |
| Out-of-order event | Sequence/time regression | Reorder by source sequence; recompute affected entity |
| Late-arriving dimension | Fact references unknown natural key | Inferred member, then resolve |
| Parent/child mismatch | Orphan foreign key | Quarantine or inferred parent; alert owner |
| Schema evolution | Registry/contract incompatibility | Compatibility gate, versioned parser, quarantine |
| Source delete | CDC tombstone or deletion request | Apply retention/privacy policy; preserve audit metadata where allowed |
| Backfill overlap | Duplicate keys across batch ranges | Idempotent deterministic merge |
| Metric drift | Source-to-gold reconciliation delta | Stop certification, identify changed definition or missing data |
| Partial pipeline failure | Checkpoint/manifest mismatch | Replay from durable offset/watermark; no partial publish |

## 13. Interviewer–candidate deep-dive dialogue

**Interviewer:** “Why not create one denormalized table with all attributes and child details?”

**Candidate:** “Because the source has independent grains and cardinalities. Flattening creates row multiplication, duplicates parent measures, increases update complexity and makes lifecycle history ambiguous. I would publish a core fact plus separate event/child facts, then expose curated views or aggregates for specific use cases.”

**Interviewer:** “What if the source has only a mutable current-state table and no event log?”

**Candidate:** “I would first ask whether CDC can capture row changes from the database log. If only periodic snapshots are available, I can infer observed state changes between snapshots, but I must label the history as sampled/inferred and cannot claim to know intermediate transitions.”

**Interviewer:** “A fact arrives before its dimension. Do you drop it?”

**Candidate:** “No. I preserve the fact and use an inferred dimension member keyed by the natural key, then enrich or remap when the full dimension arrives. The approach depends on whether the surrogate key can be updated safely and whether downstream tables have already consumed it.”

**Interviewer:** “How do you handle a late correction to a completed process?”

**Candidate:** “Keep raw change history, identify whether the correction changes descriptive state or an additive financial/event fact, and apply a versioned correction. For finance, use reversal/adjustment postings rather than overwriting the original movement. Recompute only impacted aggregates and expose both event time and posting time.”

**Interviewer:** “How would you prove the warehouse is correct?”

**Candidate:** “Define reconciliation at the same grain as the source: row counts, distinct business IDs, status distributions, amounts by currency and posting date, and selected entity-level traces. Use freshness, completeness, uniqueness, referential integrity and metric-level checks. Certification should be blocked on material unexplained variance.”

## 14. Principal-level follow-up questions

1. **What is the difference between catalog product, seller SKU and seller offer? Give a concrete example.**
2. **An order has four lines and three shipments. How do you prevent header totals being multiplied?**
3. **How do you represent partial shipment, partial cancellation and backorder for one order line?**
4. **How do you reconcile inventory movement events with end-of-day inventory snapshots?**
5. **Why are inventory balances semi-additive? How do you calculate average inventory and inventory turns?**
6. **How do you model returns where requested quantity, received quantity, restocked quantity and refunded quantity differ?**
7. **How do you allocate order-level shipping, tax and discounts to lines for product profitability?**
8. **How do you model marketplace versus first-party retail revenue and seller settlement?**
9. **How do you support multi-country currencies, tax regimes, marketplace calendars and product identity merges?**
10. **What controls ensure order, shipment, return, refund and seller settlement facts reconcile across domains?**

For each follow-up, answer in this order: clarify the requirement, identify the affected grain, state the design choice, explain an alternative, discuss failure modes, and define how you would validate it.

## 15. Common weak answers

- Drawing a star schema before understanding source entities and business processes.
- Listing columns without declaring grain and keys.
- Treating source `updated_at` as a complete event history.
- Joining independent facts directly and multiplying measures.
- Using one timestamp for event time, ingestion time, business date and posting date.
- Summing balances or distinct counts across time.
- Treating a status field as an immutable event log.
- Assuming primary/foreign-key constraints are enforced in every source.
- Omitting currency, timezone, deletion, retention and late-arrival policies.
- Giving a technology answer (“use Kafka/Spark”) without explaining semantics and recovery.

## 16. Candidate practice assignment

Complete these tasks on a whiteboard or in a Markdown file:

1. Draw the source ER diagram and label PK/FK/cardinality/optionality.
2. Identify all source grains and mutable versus append-only tables.
3. Write the target fact grain statement for each fact.
4. Draw separate star schemas and identify conformed dimensions.
5. Write DDL for the three most important target facts and dimensions.
6. Write two analytical SQL queries and one reconciliation query.
7. Explain CDC bootstrap, replay, deduplication, late data and delete handling.
8. Identify one metric that is easy to miscalculate and show how you prevent it.
9. Explain physical design for a selected platform (Snowflake, Redshift or Databricks).
10. Present a migration, ownership and adoption plan as the Principal Engineer.

### Scoring rubric (100 points)

| Area | Points | What excellent performance looks like |
|---|---:|---|
| Discovery and metric semantics | 10 | Asks targeted questions; resolves ambiguous definitions |
| Source model and lineage | 15 | Correct source grains, keys, cardinalities and mutation semantics |
| Fact/dimension design | 20 | Grain-first, conformed dimensions, history and no fanout |
| SQL and correctness | 10 | Correct joins, time logic, deduplication and reconciliation |
| Ingestion and CDC | 15 | Snapshot/CDC handoff, idempotency, replay, late data and deletes |
| Scale and performance | 10 | Workload-driven partitioning/clustering/aggregation and cost controls |
| Governance and quality | 10 | Privacy, lineage, contracts, ownership and SLOs |
| Principal leadership | 10 | Trade-offs, migration, stakeholder alignment and measurable adoption |

**Interpretation:** 85–100 = strong Principal-level signal; 70–84 = strong Senior/Staff with gaps to close; 55–69 = design fundamentals need strengthening; below 55 = revisit source modeling and grain fundamentals.

## 17. Further reading and validation

- AWS, [Dimensional modeling in Amazon Redshift](https://aws.amazon.com/blogs/big-data/dimensional-modeling-in-amazon-redshift/).
- AWS, [Redshift table design best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-tables-best-practices.html).
- Kimball Group, [Fact tables](https://www.kimballgroup.com/2008/11/fact-tables/).

Also validate warehouse-specific syntax and capabilities against the current vendor documentation. The SQL here is illustrative and intentionally not tied to one engine.

## 18. Case-specific relationship diagram

```text
products 1 ─── N seller_skus N ─── 1 sellers
seller_skus 1 ─── N offers
customers 1 ─── N orders 1 ─── N order_lines N ─── 1 offers
order_lines 1 ─── N shipment_lines N ─── 1 shipments
offers 1 ─── N inventory_movements
offers 1 ─── N inventory_snapshots
order_lines 1 ─── N return_lines
orders/order_lines 1 ─── N refund_transactions
```

A shipment can contain lines from multiple orders depending on fulfillment design; validate the real cardinality. The teaching model uses shipment-to-order-line allocations so split and partial shipment are explicit.

## 19. Case-specific target DDL

```sql
CREATE TABLE fact_order_line (
  order_line_key BIGINT NOT NULL,
  order_id VARCHAR(64) NOT NULL,
  order_line_id VARCHAR(64) NOT NULL,
  product_key BIGINT,
  seller_key BIGINT,
  offer_key BIGINT,
  order_date_key INTEGER NOT NULL,
  ordered_qty INTEGER NOT NULL,
  unit_price_minor BIGINT NOT NULL,
  line_discount_minor BIGINT NOT NULL,
  line_tax_minor BIGINT NOT NULL,
  currency_code CHAR(3) NOT NULL
);

CREATE TABLE fact_shipment_line (
  shipment_line_id VARCHAR(64) NOT NULL,
  shipment_id VARCHAR(64) NOT NULL,
  order_line_id VARCHAR(64) NOT NULL,
  shipped_qty INTEGER NOT NULL,
  shipped_ts TIMESTAMP,
  delivered_ts TIMESTAMP
);
```

## 20. Case-specific SQL exercises

**A. Order value by line without header fanout**

```sql
SELECT marketplace_key, order_date_key,
       SUM(ordered_qty * unit_price_minor - line_discount_minor + line_tax_minor) AS line_value_minor
FROM fact_order_line
GROUP BY marketplace_key, order_date_key;
```

Shipping and header-level discounts/taxes are excluded here unless allocated to lines under a documented method.

**B. Shipped quantity per order line**

```sql
SELECT order_line_id, SUM(shipped_qty) AS shipped_qty
FROM fact_shipment_line
GROUP BY order_line_id;
```

**C. Inventory movement roll-forward**

```sql
SELECT offer_key, fulfillment_center_key,
       SUM(quantity_delta) AS net_movement_qty
FROM fact_inventory_movement
WHERE occurred_ts >= :period_start
  AND occurred_ts < :period_end
GROUP BY offer_key, fulfillment_center_key;
```

Compare opening balance + net movements to closing snapshot by offer, FC and inventory condition. Explain timing cutoffs and in-transit treatment.

## 21. Detailed source-to-target mapping matrix

| Source table | Source columns used | Target model | Transformation / semantic rule |
|---|---|---|---|
| `products` | product_id, brand/category, marketplace, status | dim_product (SCD2 where needed) | Catalog identity is not seller SKU or offer. |
| `seller_skus` | seller_sku_id, seller, product, seller SKU code, condition | dim_seller_sku; dim_seller/product | Seller SKU may map to canonical product; validate reuse. |
| `offers` | offer_id, seller SKU, fulfillment, price, currency, status | dim_offer; fact_offer_price_history | Price/status history is effective-dated, not only current. |
| `order_headers` | order_id, customer, marketplace, order date, header amounts | fact_order; dim_customer/marketplace | Header measures stay at order grain. |
| `order_lines` | line_id, order, offer, quantity, unit price, discounts/tax | fact_order_line; dim_product/seller/offer | Line grain; do not repeat header total on each line. |
| `shipments` | shipment_id, order, FC, carrier, ship/delivery times | dim_shipment or shipment attributes on shipment fact | One order can have multiple shipments. |
| `shipment_lines` | shipment_line_id, shipment, order line, shipped qty | fact_shipment_line | Allocation grain supports partial shipments. |
| `inventory_movements` | movement_id, offer, FC, movement type, signed qty | fact_inventory_movement | Additive movement event; reconcile to stock snapshots. |
| `inventory_snapshots` | snapshot_id, offer, FC, snapshot time, quantities | fact_inventory_snapshot | Semi-additive balance at point in time. |
| `return_lines` | return line, order line, requested/received qty, disposition | fact_return_line; dim_return_reason/disposition | Return logistics is distinct from refund cash event. |
| `refund_transactions` | refund_id, order/line, amount, status, dates | fact_refund_transaction | Can occur with or without physical return. |
