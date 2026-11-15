# Lesson 4.1 — Dimension Table Design

## 1. Why dimensions matter

A dimension describes the business context in which a measurable event occurs. Facts answer **what happened and how much**; dimensions answer **who, what, where, when, why, and how**.

Examples:
- A sales fact records a sale amount and quantity.
- Customer, product, store, date, promotion, and channel dimensions make that sale understandable and sliceable.

A useful dimensional model is not simply a collection of tables. It is a deliberately designed interface for business analysis. A strong design makes common questions easy to express, preserves historical meaning, and avoids ambiguous joins.

### Learning outcomes

By the end of this lesson, you should be able to:
1. Establish the grain of a dimension and its related fact tables.
2. Choose natural and surrogate keys deliberately.
3. Design attributes, hierarchies, and conformed dimensions.
4. Recognize role-playing, degenerate, junk, and date dimensions.
5. Explain design trade-offs in a senior/principal-level interview.

---

## 2. Dimension versus fact

| Dimension | Fact |
|---|---|
| Descriptive business context | Measurable business event or state |
| Usually wider, fewer rows | Usually narrower, many more rows |
| Attributes, labels, categories | Measures, foreign keys, event identifiers |
| Supports filtering and grouping | Supports aggregation and analysis |
| Example: `dim_customer` | Example: `fact_order_line` |

Do not classify a table based only on its name or whether it contains numeric columns. A dimension can contain numeric descriptive attributes (for example, product package weight), and a fact can contain no additive measure at all (for example, an attendance event).

## 3. Define the grain before columns

The grain states exactly what one row represents. For a customer dimension, a common grain is:

> One row represents one version of one customer entity.

That wording matters when the table uses Type 2 slowly changing dimensions: one customer can have multiple historical rows, but each row represents one version over a defined validity interval.

For a product dimension, the grain might be one row per sellable SKU, or one row per product variant. “One row per product” is not sufficient if the business distinguishes size, colour, pack, or market-specific SKU.

**Interview habit:** State the grain out loud before drawing the table. Ask whether the business entity is a person, account, household, subscription, SKU, listing, or some other level.

## 4. Natural keys and surrogate keys

A **natural/business key** comes from the source or business domain: `crm_customer_id`, `sku_code`, or `country_iso_code`.

A **surrogate key** is a warehouse-generated identifier, commonly an integer or other stable key: `customer_sk`.

Why use a surrogate key?
- It decouples the warehouse from source-system key changes.
- It supports multiple source systems that use different identifiers for the same entity.
- It enables multiple historical versions of one business entity in a Type 2 dimension.
- It gives facts a compact, stable join key in many warehouse engines.

Example:

```sql
CREATE TABLE dim_customer (
    customer_sk       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    source_system     VARCHAR(30) NOT NULL,
    customer_bk       VARCHAR(100) NOT NULL,
    customer_name     VARCHAR(200),
    customer_segment  VARCHAR(50),
    city              VARCHAR(100),
    country_code      CHAR(2),
    valid_from        TIMESTAMP NOT NULL,
    valid_to          TIMESTAMP NOT NULL,
    is_current        BOOLEAN NOT NULL,
    record_hash       VARCHAR(64),
    UNIQUE (source_system, customer_bk, valid_from)
);
```

The exact DDL varies by platform. Some cloud warehouses do not enforce primary-key constraints; the data pipeline must then validate uniqueness and referential integrity.

### Composite business keys

If source identifiers are unique only within a tenant or source, the business key is composite. For example, `(tenant_id, account_id)` may be required. Do not silently discard the tenant component.

### Unknown and special members

Many warehouses reserve dimension members such as:
- `Unknown` — the source value is genuinely unknown.
- `Not applicable` — the relationship does not apply.
- `Not yet received` — the fact arrived before the dimension data.
- `Error` — a known data-quality failure.

These should be explicit, governed members rather than arbitrary nulls or magic values scattered through queries. The exact key values are implementation-specific.

## 5. Attributes and hierarchies

Dimension attributes should be:
- Understandable to analysts.
- Stable enough to be useful at the chosen grain.
- Governed with clear definitions.
- Consistent in format, units, and null semantics.

A product dimension might contain:

```text
dim_product
  product_sk
  product_bk
  product_name
  brand
  category
  subcategory
  package_size
  package_uom
  launch_date
  discontinued_date
```

A hierarchy such as category → subcategory → product supports drill-down. However, not every hierarchy is a strict tree. A product can belong to multiple merchandising categories; that is a many-to-many relationship and may need a bridge table rather than duplicated product rows.

Avoid storing ambiguous labels such as `status` without defining the lifecycle and allowed values. Prefer domain-specific names such as `subscription_status` or `fulfilment_status`.

## 6. Conformed dimensions

A conformed dimension has consistent meaning and compatible attributes across multiple fact tables or data marts.

For example, `dim_date` and `dim_customer` may be shared by:
- `fact_orders`
- `fact_payments`
- `fact_support_cases`

This lets an analyst compare orders, payments, and support contacts by the same customer and calendar definitions.

Conformance is semantic, not merely physical. Two tables with the same column name are not conformed if “active customer” or “region” means something different in each.

Principal-level considerations:
- Define ownership and change control for shared dimensions.
- Version semantic definitions when business meaning changes.
- Avoid forcing unrelated entities into one oversized “customer” dimension.
- Support source-specific identifiers and mastered enterprise identities explicitly.

## 7. Common dimension patterns

### Role-playing dimensions

One physical date dimension can be used in several roles:

```sql
SELECT
    o.order_id,
    order_date.calendar_date AS order_date,
    ship_date.calendar_date  AS ship_date
FROM fact_order o
JOIN dim_date AS order_date
  ON o.order_date_sk = order_date.date_sk
JOIN dim_date AS ship_date
  ON o.ship_date_sk = ship_date.date_sk;
```

The same pattern applies to customer roles such as buyer, seller, and account owner, where the underlying entity domain is genuinely the same.

### Degenerate dimensions

A degenerate dimension is a business identifier stored directly in the fact table because it has no useful independent descriptive attributes. Examples include `order_number`, `invoice_number`, or `ticket_number`.

Do not create a one-column dimension table automatically just because an identifier exists.

### Junk dimensions

A junk dimension groups low-cardinality flags and indicators that do not warrant separate dimensions. For example, `is_gift`, `is_expedited`, and `is_first_order` might be combined into `dim_order_flags`.

Use this selectively. If attributes have independent lifecycles, high cardinality, or different ownership, a junk dimension may make governance harder.

### Date dimensions

A date dimension often contains calendar date, day, week, month, quarter, fiscal period, holiday indicator, and business-day indicator. A time-of-day dimension may be separate when hour/minute/shift analysis is important.

Store fiscal calendar rules explicitly. Do not assume every business uses the same fiscal year or week definition.

---

## 8. Worked example: retail order line

**Business question:** “Show net sales by customer segment, product category, store, and order month.”

Grain: one row per order line.

```sql
CREATE TABLE fact_order_line (
    order_line_id   VARCHAR(80) NOT NULL,
    order_date_sk   INTEGER NOT NULL,
    customer_sk     BIGINT NOT NULL,
    product_sk      BIGINT NOT NULL,
    store_sk        BIGINT NOT NULL,
    quantity        DECIMAL(18, 3) NOT NULL,
    gross_amount    DECIMAL(18, 2) NOT NULL,
    discount_amount DECIMAL(18, 2) NOT NULL,
    net_amount      DECIMAL(18, 2) NOT NULL
);
```

The fact stores dimension surrogate keys and measures. The order-line identifier can be degenerate if it is only needed for drill-through and has no independent descriptive attributes.

Example query:

```sql
SELECT
    d.calendar_year,
    d.calendar_month,
    c.customer_segment,
    p.category,
    s.store_region,
    SUM(f.net_amount) AS net_sales
FROM fact_order_line f
JOIN dim_date d
  ON f.order_date_sk = d.date_sk
JOIN dim_customer c
  ON f.customer_sk = c.customer_sk
JOIN dim_product p
  ON f.product_sk = p.product_sk
JOIN dim_store s
  ON f.store_sk = s.store_sk
GROUP BY
    d.calendar_year, d.calendar_month,
    c.customer_segment, p.category, s.store_region;
```

Check the additive behaviour of each measure. `net_amount` and `quantity` are usually additive across order lines. A unit price is not safely additive. Ratios such as margin percentage should generally be recalculated from aggregated numerator and denominator, not averaged blindly.

## 9. Common mistakes and how to avoid them

1. **Starting with columns instead of grain.** Write the row definition first.
2. **Using source IDs as the only warehouse key.** Consider source changes, multiple systems, and historical versions.
3. **Overusing dimensions.** Not every code or flag needs a standalone table.
4. **Mixing entities.** A person, organization, account, and household are not automatically the same dimension.
5. **Duplicating hierarchy labels in facts.** Keep descriptive context governed in dimensions unless a deliberate snapshot is required.
6. **Ignoring null semantics.** Distinguish unknown, not applicable, and not yet received.
7. **Assuming constraints are enforced.** Validate uniqueness and joins in the actual warehouse platform.
8. **Using inconsistent definitions across marts.** Establish conformed semantics and ownership.

---

## 10. Interview conversation

**Interviewer:** Design a customer dimension for an enterprise with a CRM, billing platform, and mobile application.

**Candidate:** I would first confirm the analytical entity and grain. Are we modelling an individual person, a business account, or both? Can one person have multiple accounts? I would also clarify whether customer identity is mastered across source systems or whether source identifiers must remain distinct.

**Interviewer:** Assume a person can have multiple accounts, and each source has its own ID.

**Candidate:** I would model person and account as separate business entities unless the required analytics explicitly call for a flattened reporting view. The person dimension would have a warehouse surrogate key and a source-system-qualified business key. A cross-reference or identity-resolution mapping would connect source IDs to the enterprise person identity. The account dimension would be separate and facts would reference the appropriate entity keys.

**Interviewer:** Why not use the CRM customer ID as the primary key?

**Candidate:** It may not be globally unique, may change, and cannot represent multiple historical versions cleanly. A warehouse surrogate key decouples fact history from source key management. I would still retain the source business key for lineage and reconciliation.

**Interviewer:** The customer's city changes. What happens?

**Candidate:** That is a business rule decision. If reports must preserve the city as it was when a transaction occurred, I would use Type 2 history for the city attribute or for the relevant customer profile version. If only the latest city matters, Type 1 overwrite may be appropriate. I would classify attributes by historical reporting requirement rather than applying one SCD policy to every column.

**Interviewer:** What if the same customer appears in two systems?

**Candidate:** I would not merge records based on a weak name match. I would use governed identity resolution with deterministic identifiers where possible, confidence thresholds for probabilistic matches, survivorship rules, and an auditable crosswalk. Facts should resolve to a consistent enterprise identity when the business has approved that mapping.

## 11. Interview questions and follow-ups

### Core questions
1. What is a dimension table, and how does it differ from a fact table?
2. Why do dimensional models use surrogate keys?
3. What is a conformed dimension?
4. Explain role-playing, degenerate, and junk dimensions.
5. How do you choose the grain of a dimension?
6. When would you use a separate date dimension?
7. Can a dimension contain numeric attributes?
8. How do you model a many-to-many product category relationship?

### Follow-up / “top-up” questions
- What if two source systems use the same customer ID for different people?
- What if a source business key is corrected after facts have loaded?
- How would you handle a customer merge or identity split?
- How do you prevent a Type 2 dimension join from duplicating facts?
- What is the difference between a conformed dimension and a shared physical table?
- How do you represent an unknown dimension member?
- When would a wide denormalized dimension become problematic?
- How would you validate dimension uniqueness in a warehouse that does not enforce constraints?

### Principal-level probes
- Who owns the definition of “customer” across domains?
- How do you evolve a shared dimension without breaking downstream data products?
- How would you model identity resolution while preserving auditability and privacy?
- What governance contract would you publish for a conformed dimension?
- How do you measure whether a dimension is actually useful to consumers?

## 12. Practice exercise

Design dimensions for a ride-sharing analytics warehouse. At minimum consider:
- Rider
- Driver
- Vehicle
- Pickup and drop-off location
- Trip request, accepted, pickup, and completion dates/times
- Payment method
- Promotion

For each dimension, state its grain, business key, surrogate key strategy, important attributes, and historical tracking policy. Explain which dimensions are conformed across trip, payment, and support facts.

**Evaluation rubric (10 points):**
- Grain and entity boundaries: 2
- Key strategy and source lineage: 2
- Historical semantics: 2
- Reuse/conformance and role-playing: 2
- Clear trade-offs and operational validation: 2

## 13. Key takeaways

- Grain is the foundation of a trustworthy dimension.
- Surrogate keys decouple analytics from source-system identity and enable versioned history.
- Conformed dimensions provide consistent business meaning across facts.
- Role-playing, degenerate, junk, and date dimensions solve distinct modelling needs.
- Choose history behaviour attribute by attribute, based on the business question.
