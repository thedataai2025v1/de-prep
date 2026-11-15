# Lesson 4: Analyzing Query Patterns

## Learning objectives

-   Translate analytical questions into query patterns.
-   Identify dimensions, filters, joins, drill paths, and aggregation
    levels.
-   Use workload knowledge to guide modeling without overfitting.
-   Validate a model against representative queries.

## 1. Why query patterns matter

A model should support the questions users actually ask. Query patterns
influence: - Fact grain. - Dimension design. - Relationship structure. -
Historical tracking. - Pre-aggregation. - Physical layout and serving
choices.

Do not optimize for one sample query while ignoring the broader
workload.

## 2. Capture representative queries

For each user group, collect: - Common dashboard questions. - Filters
and group-by dimensions. - Drill-down and drill-across behavior. - Time
windows and comparison periods. - Join paths. - Expected row counts and
concurrency. - Interactive versus batch use. - Queries that must be
supported at launch versus later.

Example questions for an e-commerce warehouse: 1. Daily net sales by
product category. 2. Monthly revenue by customer acquisition cohort. 3.
Return rate by product and supplier. 4. Payment success rate by payment
method. 5. Order fulfillment time by warehouse.

These questions involve different processes and possibly different fact
grains.

## 3. Map queries to facts and dimensions

  -----------------------------------------------------------------------
  Query                   Likely fact grain       Important dimensions
  ----------------------- ----------------------- -----------------------
  Sales by product/day    Order line              Date, product, channel,
                                                  customer

  Payment success rate    Payment attempt         Date/time, method,
                                                  provider, outcome

  Fulfillment duration    Shipment or fulfillment Order, warehouse,
                          milestone               carrier, date

  Customer retention      Customer-period or      Customer, cohort date,
                          activity event          period

  Returns by reason       Return line/event       Product, customer,
                                                  reason, date
  -----------------------------------------------------------------------

Avoid assuming that one central fact table should answer every question.

## 4. Drill-across and conformed dimensions

Suppose sales and returns are separate facts. To compare them by product
and date, aggregate each fact independently to a common dimensional
level and then combine the aggregates.

This is safer than joining raw sales lines directly to raw return events
when multiple returns or adjustments can exist for a sale.

Conformed dimensions use consistent definitions and keys across facts.
Examples include shared date, product, customer, and geography
dimensions. Conformance is a semantic and governance commitment, not
merely reusing a table name.

## 5. Query-pattern inventory

A practical inventory includes: - Query name and business owner. -
Business question. - Required measures. - Filter and grouping
dimensions. - Time range and time basis. - Expected freshness. -
Frequency and concurrency. - Data volume scanned. - Performance
target. - Correctness and reconciliation rules.

Classify queries as: - High frequency / interactive. - Large analytical
scans. - Scheduled reporting. - Exploratory analysis. - Data science or
feature extraction. - Operational serving.

## 6. Example: customer support analytics

Queries: - Cases created per day by issue category. - Median first
response time by support tier. - SLA breach rate by team. - Number of
customer contacts per case. - Reopen rate by product.

A case-level accumulating snapshot can hold lifecycle milestones and
current status. A separate interaction fact can record each
message/contact. Joining these raw tables may multiply rows; aggregate
interactions to case grain before combining with case-level measures.

## 7. Interviewer--candidate conversation

**Interviewer:** You have designed a star schema. How do you know it
supports the requirements?

**Candidate:** I'd map each priority query to the fact grain, measures,
dimensions, and joins it needs. Then I'd test representative SQL and
reconcile outputs against known examples or source-system totals.

**Interviewer:** The dashboard needs both order and payment data.

**Candidate:** I'd first determine whether the relationship is
one-to-one or one-to-many. An order may have multiple payment attempts,
captures, or refunds. I'd retain separate process facts and aggregate
them to a common order or reporting grain before combining them.

**Interviewer:** What if one query is slow?

**Candidate:** I'd inspect the query plan, data volume, selectivity,
join behavior, partition pruning, file sizes, and concurrency. Depending
on evidence, I'd consider clustering, indexing where supported,
materialized aggregates, or a purpose-built serving table.

## 8. Common mistakes

-   Designing only from table names, not user queries.
-   Ignoring drill-down and historical comparisons.
-   Joining facts at incompatible grains.
-   Creating aggregates before understanding their refresh and
    correction behavior.
-   Assuming every workload belongs on the same storage/serving layer.

## 9. Practice questions

1.  Map five e-commerce queries to facts and dimensions.
2.  How do you support drill-across between sales and returns?
3.  What is a conformed dimension?
4.  How do you prevent fan-out when joining case and interaction data?
5.  How would query frequency and concurrency influence design?

## 10. Principal-level follow-ups

-   How do you balance reusable enterprise models with workload-specific
    marts?
-   How do you govern conformed dimensions across domains?
-   How do you identify a model that is overfit to current dashboards?
-   How do you handle a new high-concurrency use case without
    destabilizing existing consumers?

## Revision checklist

-   [ ] I collect representative queries.
-   [ ] I map each query to a fact grain and dimensions.
-   [ ] I understand drill-across and conformance.
-   [ ] I detect incompatible grains and fan-out.
-   [ ] I connect physical optimization to observed workload evidence.

## Key takeaway

**Design for a portfolio of real query patterns, and validate the model
by executing representative analytical questions.**
