# Mock Interview 6 — Amazon-Like E-commerce Marketplace Data Warehouse

**Interview level:** Senior / Staff / Principal Data Engineer  
**Suggested duration:** 90 minutes  
**Primary challenge:** Model catalog identity, seller offers, order headers and lines, split shipments, inventory, promotions, returns, refunds, and marketplace finance across a complex multi-seller retail ecosystem.

## 1. Interview brief

> Design an analytical platform for a global e-commerce marketplace. Teams need sales, conversion, product performance, seller performance, inventory availability, fulfillment speed, returns, refunds, customer lifetime value, and contribution margin.

The key traps are confusing product identity with seller offer identity, order grain with order-line grain, ordered quantity with shipped quantity, and gross order value with recognized revenue or settled cash.

## 2. Discovery questions

- Is this first-party retail, third-party marketplace, or both? Assume both.
- Can one product have many sellers/offers? Yes.
- Is ASIN/product identity the same as seller SKU? No.
- Can an order contain multiple sellers and ship in multiple packages? Yes.
- Can a line be partially shipped, cancelled, returned, or refunded? Yes.
- Is inventory tracked by SKU, fulfillment center, condition, and time? Yes.
- Are promotions applied at order or line level? Both, with allocation rules.
- Are taxes, shipping, discounts, gift cards, and marketplace fees separate? Yes.
- What does "sales" mean? Define ordered, shipped, delivered, net of returns, and finance-recognized measures separately.

## 3. Business processes and grains

| Process | Fact | Grain |
|---|---|---|
| Order header | `fact_order` | One row per customer order |
| Order line | `fact_order_line` | One row per order × line |
| Order lifecycle | `fact_order_event` | One row per order/line status event |
| Shipment | `fact_shipment` | One row per shipment/package |
| Shipment line | `fact_shipment_line` | One row per shipment × order line |
| Inventory | `fact_inventory_snapshot` | One row per SKU/offer × fulfillment center × condition × snapshot time |
| Inventory movement | `fact_inventory_movement` | One row per stock movement |
| Return | `fact_return_line` | One row per return authorization × order line × returned item/quantity |
| Refund | `fact_refund_transaction` | One row per refund financial transaction |
| Offer price | `fact_offer_price_history` | One row per seller offer × effective interval/version |
| Promotion | `bridge_order_line_promotion` | One row per order line × applied promotion |

## 4. Identity model

```text
PRODUCT / ASIN ──< SELLER_OFFER >── SELLER
       │                 │
       │                 ├──< OFFER_PRICE_HISTORY
       │                 └──< INVENTORY_POSITION >── FULFILLMENT_CENTER
       │
CUSTOMER ──< ORDER ──< ORDER_LINE ──< SHIPMENT_LINE >── SHIPMENT
                         │
                         ├──< ORDER_LINE_EVENT
                         ├──< RETURN_LINE ──< REFUND_TRANSACTION
                         └──< ORDER_LINE_PROMOTION >── PROMOTION
```

**Product/ASIN** describes the catalog product. **Seller SKU** identifies a seller's own inventory/catalog item. **Offer** is the seller's sellable offer for a product, potentially with price, condition, fulfillment method, and availability. These keys must not be collapsed.

## 5. Dimensions

- `dim_product`: catalog identity, brand/category, attributes; Type 2 where product classification history matters.
- `dim_seller`: seller account, cohort, region, status.
- `dim_offer`: offer identity, condition, fulfillment channel, seller-product mapping.
- `dim_customer`: pseudonymous customer key, cohort/segment.
- `dim_fulfillment_center`, `dim_marketplace`, `dim_geography`.
- `dim_promotion`, `dim_shipping_method`, `dim_return_reason`.
- `dim_date`: order, ship, delivery, return, refund, inventory snapshot date roles.
- `dim_device`, `dim_channel`.

A seller can list the same product in multiple conditions or fulfillment modes. Model offer identity separately from product identity and preserve effective-dated offer attributes.

## 6. Core fact designs

### `fact_order`
**Grain:** one row per customer order.  
Measures: order-level shipping, tax, gift wrap, order-level discount, total payable. This is not the place to repeat line-level quantity and product revenue.

### `fact_order_line`
**Grain:** one row per order × order line.  
Fields: order ID, line ID, product key, seller key, offer key, ordered quantity, unit price, allocated discount, allocated tax, ordered amount, cancellation quantity. One order with five products has five line rows.

### `fact_shipment_line`
**Grain:** one row per shipment × order line.  
Supports split shipments and partial quantities. `shipped_qty` can sum across multiple shipment lines for an order line. Do not put one shipment ID on the order line if split shipments are allowed.

### `fact_inventory_snapshot`
**Grain:** SKU/offer × fulfillment center × inventory condition × snapshot timestamp.  
Measures: sellable, reserved, damaged, in-transit, safety-stock, available-to-promise quantities. Inventory is semi-additive across time: sum across locations may be valid at one snapshot, but summing daily ending balances across dates is usually not.

### `fact_inventory_movement`
**Grain:** one inventory movement event.  
Types: receipt, pick, pack, ship, transfer, adjustment, return-to-stock, damage. Reconcile movement deltas to snapshots.

### `fact_return_line` and `fact_refund_transaction`
Returns and refunds are separate. An item can be returned without an immediate refund; a refund may occur without physical return (policy exception). Partial returns require quantity-level modeling.

## 7. Metrics and SQL

Define:
- Ordered GMV: value of orders placed, before or after discounts—name exact version.
- Shipped sales: value of shipped quantities.
- Delivered sales: value of delivered quantities.
- Net sales: finance-approved revenue measure after defined returns/discounts/tax treatment.
- Return rate: returned quantity / eligible shipped quantity, by cohort.
- In-stock rate: sellable offer-time / eligible offer-time, not a count of current stock rows.
- Inventory turns: cost of goods sold / average inventory, with accounting period and valuation policy.

### Example: ordered units and value by product
```sql
SELECT
  p.category_name,
  SUM(l.ordered_qty) AS ordered_units,
  SUM(l.ordered_amount_minor) AS ordered_value_minor
FROM fact_order_line l
JOIN dim_product p ON l.product_key = p.product_key
WHERE l.order_ts >= :start_ts AND l.order_ts < :end_ts
GROUP BY p.category_name;
```
If the order-level tax or shipping is allocated to lines, document the allocation method. Never add order-header totals once per line.

### Example: shipped quantity by order line
```sql
SELECT order_id, order_line_id,
       SUM(shipped_qty) AS total_shipped_qty
FROM fact_shipment_line
GROUP BY order_id, order_line_id;
```

## 8. Returns, refunds, and financial attribution

A customer orders 3 units, 2 ship, 1 is cancelled, then 1 shipped unit is returned and refunded. The model should preserve:
- ordered quantity = 3;
- shipped quantity = 2;
- cancelled quantity = 1;
- returned quantity = 1;
- retained delivered quantity = 1;
- refund amount and transaction date;
- return disposition (restock, damage, refurbish, discard);
- seller/platform responsibility and fee reversal policy.

A refund is a monetary event. Return is a logistics/quality event. They are related but not identical.

## 9. Data pipelines and architecture

- Catalog and seller/offer systems: CDC plus effective-dated snapshots.
- Orders: transactional events and CDC for corrections.
- Fulfillment: shipment, package, tracking, and delivery events.
- Inventory: movement stream plus periodic reconciliation snapshots.
- Returns/refunds: independent workflows and financial transaction events.
- Bronze stores raw immutable source envelopes.
- Silver resolves product/offer/seller keys, normalizes event taxonomies, deduplicates, and reconciles versions.
- Gold creates order, line, shipment, inventory, returns, seller, and finance marts.

At high volume, use event time and ingestion time distinctly. Backfills should be bounded by affected entity/date ranges. Maintain source-to-warehouse lineage and reconciliation counts.

## 10. Interview dialogue

**Interviewer:** "An order contains three products from two sellers. How many rows?"  
**Candidate:** "One order header, three order lines, and potentially multiple shipment and shipment-line rows depending on split fulfillment. Seller attribution belongs at the offer/order-line level."

**Interviewer:** "Can I join order total to order lines and sum it?"  
**Candidate:** "No. That repeats the header amount once per line. Either aggregate lines to order grain before joining or allocate order-level amounts to lines with a documented method."

**Interviewer:** "A product has five sellers. What is the product key?"  
**Candidate:** "The catalog product key identifies the product. Each seller's offer has a separate offer key and seller SKU. Product-level demand can aggregate across offers; seller performance must remain at offer/seller grain."

**Interviewer:** "Inventory is 10 at midnight and 8 the next midnight. Can I sum it to 18?"  
**Candidate:** "Not for on-hand balance. Inventory snapshots are semi-additive over time. Use an end-of-period or average-balance calculation. Movement facts are additive events."

**Interviewer:** "A return arrives 30 days after purchase."  
**Candidate:** "Report it by return event date for operations and by original order cohort for product quality/return-rate analysis. Keep both date roles and do not rewrite the original order date."

## 11. Principal-level probes
1. How do you establish canonical product identity across catalog merges?
2. How do you handle seller SKU reuse or seller account changes?
3. How do you support both first-party and third-party financial semantics?
4. How do you reconcile inventory movement streams with periodic stock counts?
5. How do you prevent double counting across order, line, shipment, and refund facts?
6. How do you handle multi-country tax and currency rules?
7. How do you design a migration from legacy order tables to event-based fulfillment?
8. How do you align catalog, retail, marketplace, supply chain, and finance ownership?

## 12. Common mistakes
- Treating ASIN/product, seller SKU, and offer as the same entity.
- Modeling only order header and ignoring line grain.
- Assuming one order has one shipment.
- Putting one shipment ID on an order line when partial shipments exist.
- Summing order totals after joining to lines.
- Summing inventory snapshots across dates.
- Treating returns and refunds as identical.
- Reporting ordered GMV as revenue without finance definitions.

## 13. Practice exercise

An order has four lines from three sellers. Two lines ship immediately, one ships later in two packages, and one is cancelled. One delivered item is returned and refunded; another is refunded without a physical return. Inventory is transferred between fulfillment centers.

Design:
- order/header/line/shipment grains;
- offer/product/seller identities;
- inventory movement and snapshot facts;
- returns/refunds;
- promotion and shipping allocation;
- metrics that reconcile across customer service, operations, and finance.

### Scoring rubric (10 points)
| Dimension | Points |
|---|---:|
| Requirement discovery and metric definitions | 1.5 |
| Product/offer/order identity and grains | 2.0 |
| Fulfillment, inventory, returns, refunds | 2.0 |
| Financial correctness and reconciliation | 1.5 |
| Scale, CDC, and physical design | 1.0 |
| Governance and quality | 0.5 |
| Principal-level leadership | 1.5 |

## 14. Further reading
- AWS, [Dimensional modeling in Amazon Redshift](https://aws.amazon.com/blogs/big-data/dimensional-modeling-in-amazon-redshift/).
- AWS, [Redshift table design best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-tables-best-practices.html).
