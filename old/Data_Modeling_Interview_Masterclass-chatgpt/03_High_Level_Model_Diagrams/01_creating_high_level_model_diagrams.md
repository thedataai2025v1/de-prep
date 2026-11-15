# Lesson 1: Creating High-Level Model Diagrams

## Learning objectives
- Turn business requirements into a conceptual model.
- Identify entities, relationships, cardinality, and optionality.
- Distinguish conceptual, logical, and physical models.
- Explain a model clearly during a whiteboard interview.

## 1. What a high-level model should communicate

A high-level model is a compact representation of the important business concepts and how they relate. It is not a complete physical schema. It should help stakeholders and engineers agree on:
- The business entities and processes in scope.
- The relationships between entities.
- The important business identifiers.
- The cardinality and optionality of relationships.
- The boundaries between domains or processes.
- The major assumptions and unresolved questions.

Start with business meaning. Table names and data types come later.

## 2. Conceptual, logical, and physical models

| Model | Focus | Typical contents |
|---|---|---|
| Conceptual | Business understanding | Entities, relationships, business vocabulary |
| Logical | Detailed structure independent of platform | Attributes, keys, cardinality, normalization |
| Physical | Implementation | Tables, columns, data types, indexes, partitions, constraints |

A dimensional model is a logical/analytical design style. A star schema is not simply a conceptual ER diagram with different names.

## 3. A repeatable diagramming method

1. Restate the business process and scope.
2. Identify nouns that represent business entities.
3. Identify verbs that represent relationships or events.
4. Separate independent business processes.
5. State the grain of each event or fact.
6. Add identifiers and relationship cardinality.
7. Mark optional relationships and lifecycle states.
8. Identify many-to-many relationships and associative entities.
9. Validate the model against representative business questions.
10. Record assumptions and open decisions.

## 4. Example: online order process

A simplified conceptual view:

```text
CUSTOMER 1 ───────< ORDER 1 ───────< ORDER_LINE >────── 1 PRODUCT
                         |
                         | 0..*
                         v
                    PAYMENT
                         |
                         | 0..*
                         v
                    REFUND
```

Interpretation:
- A customer may place many orders.
- An order contains one or more order lines (confirm whether draft/empty orders exist).
- Each order line refers to a product or product variant.
- An order may have multiple payment attempts.
- A payment may have multiple refunds or adjustments depending on the payment provider and business process.

The diagram is deliberately simplified. It does not yet resolve shipment splits, bundles, promotions, taxes, or partial fulfillment.

## 5. Cardinality and optionality

Common cardinalities:
- One-to-one: one record relates to at most one record on the other side.
- One-to-many: one parent may relate to many children.
- Many-to-many: many records on each side can relate to many on the other side.

Always ask whether the relationship is optional. For example, an order may exist before payment authorization, so an order-to-payment relationship may be zero-to-many rather than exactly one-to-many.

Many-to-many relationships usually require an associative entity in a relational model. In analytical modeling, a bridge table may represent the relationship and its allocation semantics.

## 6. Entity versus event

A customer is an entity with identity and attributes. An order placed is a business event. Do not confuse a current-state entity table with an event history.

A useful question: "Can this concept happen repeatedly over time?" If yes, there may be a process/event that deserves its own model and grain.

## 7. Interviewer–candidate conversation

**Interviewer:** Draw a high-level model for an e-commerce platform.

**Candidate:** Before drawing, I'd confirm the scope: order placement only, or the complete lifecycle including payment, fulfillment, returns, and refunds? For an initial order-to-cash scope, I'd identify customer, order, order line, product, payment attempt, shipment, return, and refund as distinct concepts.

**Interviewer:** Why separate order and order line?

**Candidate:** They have different grains. An order represents the customer's purchase transaction; an order line represents one product or variant and its quantity and price. Product-level sales analysis requires line-level detail.

**Interviewer:** Can one order have multiple payments?

**Candidate:** It depends on split tender, retries, installments, and provider behavior. I'd model payment attempts or transactions separately rather than assume one payment row per order.

## 8. Common mistakes
- Drawing tables before clarifying the process.
- Omitting relationship cardinality.
- Assuming every child is mandatory.
- Mixing entities and events.
- Hiding many-to-many relationships.
- Making a high-level diagram too detailed to discuss.

## 9. Practice questions
1. Draw a conceptual model for a library lending process.
2. Explain the difference between an entity and an event.
3. Why might order-to-payment be one-to-many?
4. How do you represent a many-to-many relationship?
5. What information belongs in a conceptual diagram versus a physical schema?

## 10. Principal-level follow-ups
- How do you standardize modeling notation across teams?
- How do you keep a conceptual model useful as domains evolve?
- How do you communicate unresolved semantics without blocking early design?
- How do you decide when two apparently similar entities should remain separate?

## Revision checklist
- [ ] I can explain conceptual, logical, and physical models.
- [ ] I identify entities, events, keys, cardinality, and optionality.
- [ ] I state the process scope before drawing.
- [ ] I validate the diagram against business questions.

## Key takeaway
**A good high-level diagram makes business meaning, relationships, and assumptions visible before implementation details take over.**
