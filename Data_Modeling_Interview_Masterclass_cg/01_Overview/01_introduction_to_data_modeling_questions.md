# Lesson 1: Introduction to Data Modeling Questions

## Learning objectives

By the end of this lesson, you should be able to: - Explain what data
modeling is and why it matters to analytical systems. - Distinguish
business requirements, conceptual models, logical models, and physical
models. - Recognize the signals an interviewer is testing. - Establish a
repeatable approach to an open-ended modeling problem.

## 1. What is data modeling?

Data modeling is the process of representing business concepts, their
attributes, relationships, rules, and history in a structured form so
that data can be stored, integrated, queried, governed, and trusted.

A good model is not merely a collection of tables. It is an explicit
representation of: - **Business meaning:** What does a customer, order,
trip, payment, or active user mean? - **Grain:** What does one row
represent? - **Relationships:** How do entities and events relate? -
**Time:** Which facts are current, and which historical states must be
preserved? - **Workload:** Which queries and operational processes must
the model support? - **Quality and governance:** Which constraints,
ownership, privacy, and retention rules apply?

A technically valid schema can still be a poor model if it represents
the wrong business meaning or cannot answer the required questions.

## 2. Why interviewers ask data modeling questions

Interviewers are usually evaluating several abilities at once:

1.  **Requirement discovery:** Do you clarify the business objective
    before drawing tables?
2.  **Analytical thinking:** Can you identify entities, events,
    measures, dimensions, and relationships?
3.  **Grain discipline:** Can you define what one row means before
    choosing columns?
4.  **Temporal reasoning:** Can you represent changes, late-arriving
    data, and historical truth?
5.  **Trade-off judgment:** Can you explain why one model is better for
    a particular workload?
6.  **Communication:** Can you lead the discussion, make assumptions
    explicit, and respond to challenges?
7.  **Operational awareness:** Can you account for volume, latency,
    quality, security, and evolution?

At Senior Staff or Principal level, interviewers also look for the
ability to establish shared definitions across teams, identify risks,
and make a model evolvable.

## 3. The modeling layers

### Conceptual model

A business-level view of the major entities and their relationships. It
avoids implementation detail.

Example: Customer places Order; Order contains Order Lines; each Order
Line refers to a Product.

### Logical model

A more precise representation of entities, attributes, identifiers,
relationships, cardinality, and business constraints. It is usually
technology-independent.

### Physical model

The implementation in a particular database or platform: tables,
columns, data types, keys, constraints, partitioning, clustering,
indexes, file formats, and storage layout.

Do not jump to physical choices before the business meaning and logical
structure are clear.

## 4. OLTP and OLAP are different modeling contexts

  -----------------------------------------------------------------------
  Concern                 OLTP / operational      OLAP / analytical
  ----------------------- ----------------------- -----------------------
  Primary goal            Run business            Analyze business
                          transactions            performance

  Typical operations      Insert, update, point   Scan, aggregate, join
                          lookup                  

  Modeling tendency       Normalized              Often dimensional or
                                                  otherwise denormalized

  Data                    Current operational     Integrated, historical,
                          state and transaction   analytical
                          history                 

  Example                 Order service database  Sales warehouse
  -----------------------------------------------------------------------

Normalization is not automatically good or bad. The right design depends
on the system's purpose and workload.

## 5. A repeatable opening for an interview

A strong opening sounds like this:

> "Before drawing the schema, I'd like to clarify the business decisions
> and analytical questions this model must support. Then I'll identify
> the business processes and entities, define the grain of each fact,
> establish dimensions and relationships, and finally discuss history,
> scale, quality, and physical optimization."

This communicates structure without pretending that requirements are
already known.

## 6. Mini example: e-commerce sales

Suppose the request is: "Design a data model to analyze sales."

Clarify: - Is the goal order monitoring, financial reporting, product
analytics, or all three? - Is the grain one order or one order line? -
Are cancellations, returns, discounts, taxes, and shipping included? -
Do analysts need the product category as it was at the time of sale? -
What are the expected data volume and freshness requirements?

A common analytical starting point is one row per order line in a sales
fact table. Measures may include quantity, gross amount, discount
amount, tax amount, and net amount. Dimensions may include date,
customer, product, store/channel, and promotion.

This is only a starting hypothesis. The business definition of revenue
and the handling of returns must be clarified before implementation.

## 7. Interviewer--candidate conversation

**Interviewer:** Design a data model for an online marketplace.

**Candidate:** I'd first clarify whether we're modeling operational
transactions or analytical reporting. If the objective is analytics, I'd
identify the core business processes---orders, order lines, payments,
shipments, returns, and seller activity---and understand the questions
the business needs to answer.

**Interviewer:** Why not start with a single large table?

**Candidate:** A single wide table may simplify some queries, but it can
duplicate attributes, create inconsistent updates, and mix processes
with different grains. I'd model each business process at a clearly
defined grain, then use conformed dimensions where shared analysis is
required. A curated wide table could still be produced for a specific
workload.

**Interviewer:** What is the first modeling decision?

**Candidate:** The business process and grain. If the sales fact is one
row per order line, every measure and join must respect that grain.
Payment events or shipment events should not be joined blindly at that
grain because one order line can have multiple related events.

**Interviewer:** What if requirements change?

**Candidate:** I'd separate stable business concepts from implementation
choices, preserve source lineage, version important definitions, and
assess whether the change affects grain, history, downstream contracts,
or only a derived attribute.

## 8. Common mistakes

-   Drawing tables before clarifying the business problem.
-   Treating a business entity and an analytical fact as
    interchangeable.
-   Defining a fact table without stating its grain.
-   Mixing order-level and order-line-level measures.
-   Assuming all dimensions are static.
-   Optimizing indexes or partitions before understanding query
    patterns.
-   Giving a design without explaining assumptions or trade-offs.

## 9. Practice questions

1.  What is data modeling, and what business problems does it solve?
2.  Explain conceptual, logical, and physical data models.
3.  How does OLTP modeling differ from OLAP modeling?
4.  Why is grain a foundational decision?
5.  When might a denormalized model be preferable?
6.  How would you model an order that contains multiple products?
7.  What information would you clarify before designing a sales
    warehouse?
8.  How would you make a model resilient to changing business
    requirements?

## 10. Principal-level follow-ups

-   How do you align customer, product, and revenue definitions across
    business domains?
-   How do you govern a shared dimension used by multiple teams?
-   How do you introduce a breaking schema change without disrupting
    consumers?
-   How do you balance a canonical enterprise model with domain-specific
    data products?
-   How do you measure whether a model is successful beyond query
    performance?

## 11. Revision checklist

-   [ ] I can explain the purpose of data modeling.
-   [ ] I can distinguish conceptual, logical, and physical models.
-   [ ] I clarify business questions before designing.
-   [ ] I define grain before measures and joins.
-   [ ] I explain why the model fits the workload.
-   [ ] I can discuss evolution, history, quality, and governance.

## Key takeaway

**A strong data model starts with business meaning and workload, not
with tables.**
