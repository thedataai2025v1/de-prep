# Lesson 4: Data Modeling Fundamentals

## Learning objectives

-   Understand entities, attributes, relationships, keys, constraints,
    and cardinality.
-   Distinguish normalization from dimensional modeling.
-   Understand fact and dimension concepts and measure behavior.
-   Avoid common modeling errors involving grain, keys, and joins.

## 1. Core building blocks

### Entity

A business object or concept about which the system stores information.

Examples: Customer, Product, Order, Driver, Trip, Payment, Account.

### Attribute

A property describing an entity.

Examples: Customer email, product category, order status, trip distance.

### Relationship

An association between entities.

Examples: - A customer places many orders. - An order contains many
order lines. - Each order line refers to one product.

### Identifier and key

A key identifies a row or connects related rows.

-   **Natural/business key:** A meaningful identifier from the business
    domain, such as an order number.
-   **Surrogate key:** A generated identifier used by the warehouse,
    often useful for versioned dimensions.
-   **Primary key:** The chosen unique row identifier.
-   **Foreign key:** A reference to a key in another table.
-   **Composite key:** A key made from multiple columns.

A natural key can change, be reused, or arrive late. A surrogate key can
provide stable warehouse identity, but it does not remove the need to
retain source business identifiers and enforce uniqueness rules.

## 2. Cardinality and optionality

Common relationship patterns: - **One-to-one:** One user has one
profile, if the business rule enforces it. - **One-to-many:** One
customer places many orders. - **Many-to-many:** Students enroll in many
courses and each course has many students.

A many-to-many relationship is often represented using an associative or
bridge table.

Also specify optionality: - Can an order exist before a payment is
made? - Can a customer have no orders? - Can a trip have no assigned
driver at request time?

Do not infer mandatory relationships simply because a foreign key exists
in a sample dataset.

## 3. Normalization

Normalization organizes relational data to reduce redundancy and update
anomalies.

### First Normal Form (1NF)

Attributes contain atomic values under the chosen relational
interpretation; repeating groups are not stored as multiple values in
one field.

### Second Normal Form (2NF)

The relation is in 1NF and every non-prime attribute depends on the
whole candidate key, not only part of a composite candidate key.

### Third Normal Form (3NF)

The relation is in 2NF and non-key attributes do not have inappropriate
transitive dependencies on a candidate key.

Normalization is useful for transactional consistency and controlled
updates. It is not a requirement that every analytical warehouse table
be fully normalized.

## 4. Dimensional modeling

Dimensional modeling structures analytical data around business
processes, facts, and descriptive dimensions.

-   **Fact table:** Stores measurements or events at a declared grain.
-   **Dimension table:** Stores descriptive context used to filter,
    group, and label facts.
-   **Star schema:** Fact table joins directly to denormalized
    dimensions.
-   **Snowflake schema:** Some dimension attributes are further
    normalized into related tables.

Example: - `fact_sales`: one row per order line. - `dim_date`: calendar
and fiscal attributes. - `dim_customer`: customer descriptors. -
`dim_product`: product descriptors. - `dim_store`: store descriptors.

A star schema often makes analytical queries and semantic definitions
easier to understand. The best design depends on the platform, workload,
governance needs, and data characteristics.

## 5. Grain: the most important fact-table decision

Grain defines exactly what one row represents.

Examples: - One row per order. - One row per order line. - One row per
payment attempt. - One row per customer per day. - One row per account
balance at end of day.

A fact table must have a consistent grain. Do not mix an order-level
total with order-line-level rows unless the semantics and aggregation
behavior are explicitly designed to prevent duplication.

### Grain and measure behavior

-   **Additive:** Can be summed across all relevant dimensions. Example:
    line quantity or line sales amount, subject to business definitions.
-   **Semi-additive:** Can be summed across some dimensions but not
    others. Example: account balance can be summed across accounts at a
    point in time, but summing daily balances across dates is usually
    meaningless.
-   **Non-additive:** Cannot be meaningfully summed. Example: ratios,
    percentages, and unit prices. Store the underlying components when
    possible and calculate the ratio at query time.

## 6. Fact table types

### Transaction fact

One row per event or transaction at a defined grain, such as each order
line or payment event.

### Periodic snapshot fact

One row per entity or measurement at a regular interval, such as account
balance per day.

### Accumulating snapshot fact

One row per process instance that is updated as it moves through
milestones, such as an order from placed to shipped to delivered.

These represent different business measurement patterns and should not
be confused.

## 7. Slowly Changing Dimensions (preview)

Dimension attributes may change over time.

-   **Type 1:** Overwrite the old value; history is not retained in that
    dimension.
-   **Type 2:** Add a new versioned row with effective dates or
    equivalent validity metadata.
-   **Type 3:** Retain a limited prior value in an additional column.

Choose based on the question: - "Show the customer's current segment"
may use Type 1. - "Show revenue by the segment the customer belonged to
at the time of sale" usually needs historical versioning, often Type 2.

Detailed implementation, surrogate-key assignment, late-arriving
dimensions, and advanced patterns are covered in the Dimension Design
module.

## 8. Example: a simple sales model

Assume the analytical requirement is product sales by day, customer
segment, and sales channel.

### Grain

One row in `fact_sales` represents one order line.

### Illustrative tables

`fact_sales` - sales_line_key - order_number (degenerate business
identifier) - order_date_key - customer_key - product_key -
channel_key - quantity - gross_amount - discount_amount - net_amount

`dim_date` - date_key - calendar_date - month - quarter - fiscal_period

`dim_customer` - customer_key - customer_business_id - segment - region

`dim_product` - product_key - product_business_id - product_name -
category

`dim_channel` - channel_key - channel_name

The example omits returns and payment events for clarity. Those should
be modeled as separate processes if the business needs their distinct
lifecycle and grain.

## 9. SQL illustration

``` sql
SELECT
    d.calendar_date,
    p.category,
    c.segment,
    SUM(f.net_amount) AS net_sales
FROM fact_sales f
JOIN dim_date d
  ON f.order_date_key = d.date_key
JOIN dim_product p
  ON f.product_key = p.product_key
JOIN dim_customer c
  ON f.customer_key = c.customer_key
GROUP BY
    d.calendar_date,
    p.category,
    c.segment;
```

This query assumes: - Each fact row is one order line. - Dimension keys
resolve to the intended dimension row. - `net_amount` is defined
consistently. - The joins do not multiply fact rows. - Returns are
either already reflected in the measure or reported separately according
to an agreed accounting rule.

## 10. Common modeling pitfalls

1.  **Unclear grain:** Leads to incorrect aggregation.
2.  **Fan-out joins:** Joining multiple one-to-many child tables can
    multiply rows.
3.  **Wrong measure additivity:** Summing balances across time produces
    misleading results.
4.  **Over-normalizing analytics:** Can make common reporting
    unnecessarily complex.
5.  **Over-denormalizing transactions:** Can create update anomalies.
6.  **Unstable business keys:** Can break historical joins if not
    handled.
7.  **Missing unknown-member strategy:** Facts may arrive before
    dimensions.
8.  **No temporal semantics:** Users cannot distinguish current truth
    from historical truth.
9.  **Ambiguous metric definitions:** Different teams calculate the same
    KPI differently.

## 11. Interviewer--candidate conversation

**Interviewer:** Explain the difference between normalization and
dimensional modeling.

**Candidate:** Normalization organizes relational data to reduce
redundancy and update anomalies, commonly in transactional systems.
Dimensional modeling organizes analytical data around business
processes, facts, and descriptive dimensions so that users can query and
aggregate it easily. They solve different problems; neither is
universally superior.

**Interviewer:** Can you store multiple products in one order row?

**Candidate:** We could store a nested collection in some platforms, but
for a conventional relational dimensional model I'd represent order
lines separately. That gives a clear order-line grain and supports
product-level measures and analysis.

**Interviewer:** Can I sum every numeric column in a fact table?

**Candidate:** No. I classify measures by additivity. Quantities and
monetary amounts may be additive under defined rules; balances are often
semi-additive; percentages and unit prices are non-additive. For ratios,
I generally retain numerator and denominator and calculate the ratio
after aggregation.

**Interviewer:** Why use surrogate keys in a dimension?

**Candidate:** They provide warehouse-controlled identity and help
distinguish historical versions of a business entity, particularly for
Type 2 dimensions. I'd still retain the natural business key for lineage
and matching.

## 12. Advanced follow-up questions

1.  What is the difference between a candidate key and a primary key?
2.  When is a composite key appropriate?
3.  What is a degenerate dimension?
4.  What is a role-playing dimension?
5.  What is a conformed dimension?
6.  How do you model a many-to-many relationship in a star schema?
7.  What is a factless fact table?
8.  How do you handle late-arriving facts and dimensions?
9.  How do you avoid double-counting when joining facts?
10. When would you choose a snowflake schema over a star schema?
11. How do you represent changing organizational hierarchies?
12. How would you model event time and ingestion time?

## 13. Principal-level discussion prompts

-   How would you define and govern a shared customer dimension across
    domains?
-   How do you decide between a canonical model and domain-owned data
    products?
-   How do you preserve historical semantics while changing a model's
    grain?
-   How do you introduce new dimensions without breaking existing
    consumers?
-   How do you enforce metric consistency across BI, ML, and operational
    use cases?

## 14. Revision checklist

-   [ ] I can explain entities, attributes, relationships, and
    cardinality.
-   [ ] I understand primary, foreign, natural, surrogate, and composite
    keys.
-   [ ] I can explain 1NF, 2NF, and 3NF at a practical level.
-   [ ] I can distinguish normalized and dimensional models.
-   [ ] I define fact-table grain precisely.
-   [ ] I can classify additive, semi-additive, and non-additive
    measures.
-   [ ] I can identify fan-out risks.
-   [ ] I can explain when historical dimension tracking is required.

## Key takeaway

**Grain establishes what a fact row means; keys and relationships
establish how it connects; measure behavior establishes how it can be
analyzed.**
