# Lesson 3: Practice — E-commerce Platform

## Interview prompt

Design a high-level data model for a multi-region e-commerce company. It sells physical and digital products, supports promotions, multiple payment methods, split shipments, returns, and refunds.

## 1. Clarify before modeling

Ask:
1. Are we modeling operational transactions, analytics, or both?
2. Does a cart become an order only after checkout?
3. Can one order contain multiple currencies or legal entities?
4. Can an order have multiple payment attempts or split tenders?
5. Can an order ship from multiple warehouses?
6. Are returns allowed at line or quantity level?
7. Can digital products have fulfillment events?
8. Do we need historical product price/category and customer attributes?

State assumptions if the interviewer does not provide answers.

## 2. Identify business processes

- Product catalog and product changes.
- Customer registration and profile changes.
- Cart and checkout.
- Order placement and order-line creation.
- Payment authorization, capture, failure, refund.
- Inventory reservation and adjustment.
- Shipment and delivery.
- Return and refund.
- Promotion application.

Keep distinct processes separate when their event grains differ.

## 3. Candidate conceptual diagram

```text
CUSTOMER 1 ─────< ORDER 1 ─────< ORDER_LINE >──── 1 PRODUCT
                     |
                     ├────< PAYMENT_ATTEMPT
                     |
                     ├────< SHIPMENT 1 ─────< SHIPMENT_LINE >──── ORDER_LINE
                     |
                     └────< RETURN 1 ─────< RETURN_LINE >─────── ORDER_LINE

ORDER ─────< ORDER_PROMOTION >──── PROMOTION
PRODUCT ───< PRODUCT_CATEGORY >─── CATEGORY
```

The diagram shows candidate relationships. Real implementations may require separate payment transactions, capture/refund events, inventory reservations, and product variants.

## 4. Grain statements

| Concept / fact candidate | Grain |
|---|---|
| Order | One row per accepted order |
| Order line | One row per product/variant line within an order |
| Payment attempt | One row per attempt to authorize or collect payment |
| Payment transaction | One row per capture, refund, reversal, or adjustment event |
| Shipment | One row per shipment |
| Shipment line | One row per order line allocation to a shipment |
| Return line | One row per returned order-line quantity/reason event |
| Inventory movement | One row per inventory movement event |

A split shipment means an order line may be allocated across multiple shipments. Do not assume one order line maps to exactly one shipment.

## 5. Dimensional modeling direction

Potential dimensions:
- Date and time (role-playing order, payment, shipment, return dates).
- Customer.
- Product and product variant.
- Promotion.
- Warehouse.
- Shipping method/carrier.
- Payment method/provider.
- Geography.
- Order channel.

Potential facts:
- Order-line transaction fact.
- Payment-attempt fact.
- Payment-transaction fact.
- Shipment-line fact.
- Return-line fact.
- Inventory movement fact.

Conform dimensions where semantics are genuinely shared.

## 6. Important metric traps

- Gross sales versus net sales.
- Discounts at order versus line level.
- Tax and shipping inclusion.
- Refund timing versus original sale date.
- Partial returns.
- Multiple currencies and conversion-date policy.
- Payment retries and duplicate callbacks.
- Digital delivery versus physical shipment.

For order-level measures, aggregate line values to order grain before joining to order-level facts. Avoid joining multiple child collections directly.

## 7. Interviewer–candidate conversation

**Interviewer:** One order contains three items and ships in two packages. How do you model it?

**Candidate:** I'd keep one order and three order lines, then model shipments separately. A shipment-line allocation links order lines to shipments and captures shipped quantities. That supports partial shipments and split fulfillment.

**Interviewer:** One item is returned twice in separate quantities. What changes?

**Candidate:** Return lines need to represent each return event or returned quantity against the original order line. The grain must distinguish multiple return events. I'd also keep refund transactions separate because a return and a refund are related but not identical business events.

**Interviewer:** How do you report net sales by product?

**Candidate:** I'd define the accounting/business rule first. Then aggregate sales and returns/adjustments to a compatible product and reporting-date grain, with an explicit policy for whether returns are reported by return date or attributed back to sale date.

## 8. Follow-up questions
1. How do you model product variants?
2. How do you represent bundles or kits?
3. How do you allocate order-level discounts to lines?
4. How do you avoid double counting with multiple payment attempts?
5. How do you handle product category changes historically?
6. How do you model inventory reservations versus physical stock movements?

## 9. Principal-level discussion
- How would you separate commerce, payments, and fulfillment data products?
- How would you support regional tax and currency rules?
- How would you govern product and customer identities across channels?
- How would you make financial measures reconcilable and auditable?

## Evaluation rubric
- Requirements clarification: 20%
- Correct entities and relationships: 20%
- Grain and cardinality: 25%
- Historical and edge-case handling: 15%
- Communication and trade-offs: 10%
- Validation and governance: 10%

## Key takeaway
**E-commerce is several related business processes—not one order table. Model each process at its natural grain and connect them with explicit keys and allocation rules.**
