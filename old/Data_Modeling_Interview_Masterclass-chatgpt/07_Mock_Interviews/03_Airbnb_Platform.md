# Mock Interview 3 — Airbnb-Like Marketplace Data Warehouse

**Interview level:** Senior / Staff / Principal Data Engineer  
**Suggested duration:** 90 minutes  
**Primary challenge:** Model a two-sided accommodation marketplace with time-dependent availability, reservations, guests, hosts, listings, pricing, cancellations, stays, payments, payouts, reviews, and marketplace funnels.

## 1. Interview brief

> Design an analytical platform for a global accommodation marketplace. Teams need to understand search-to-book conversion, listing availability, occupancy, host performance, guest experience, cancellation, gross booking value, refunds, host payouts, and market health.

The central modeling trap: a listing is not simply "available" or "unavailable". Availability is date-specific, may be blocked by host or platform, may be held during checkout, and may be consumed by reservations. Booking creation, payment, check-in, stay completion, cancellation, refund, and payout are distinct processes.

## 2. Requirement discovery

Ask:
- Is a reservation for a single listing and continuous date range? Assume yes for this case.
- Are instant bookings and host-approved requests both supported? Yes.
- Can guests modify dates, guests, or price? Yes; retain versions.
- Can a reservation have multiple payment attempts, partial refunds, or multiple payouts? Yes.
- Is inventory modeled per night? Yes, but source systems may expose calendar-day availability.
- Can a listing be temporarily blocked, suspended, or unlisted? Yes.
- Are host and guest the same person in different transactions? Yes; model person identity separately from role in a transaction.
- Do we need search impressions and ranking exposure? Yes, but only approved event instrumentation.
- What is the occupancy denominator? Available nights, sellable nights, or all calendar nights? Publish explicit alternatives.
- What are reporting freshness needs? Near-real-time funnel/availability monitoring; daily certified finance and occupancy.

## 3. Business processes and grain

| Process | Fact | Grain |
|---|---|---|
| Search activity | `fact_search_event` | One row per search event/session |
| Listing exposure | `fact_listing_impression` | One row per listing impression |
| Availability calendar | `fact_listing_night_state` | One row per listing × local night × state version |
| Reservation lifecycle | `fact_reservation` | One row per reservation (accumulating snapshot) |
| Reservation changes | `fact_reservation_event` | One row per lifecycle/change event |
| Stay nights | `fact_stay_night` | One row per reservation × occupied night |
| Payment movement | `fact_payment_transaction` | One row per processor financial transaction |
| Refund | `fact_refund` | One row per refund event/transaction |
| Host payout | `fact_host_payout` | One row per payout transaction |
| Review | `fact_review` | One row per review submission |

Availability and stay-night facts are not interchangeable. A reservation may be booked today for nights months later; booking date and stay date answer different questions.

## 4. Conceptual model

```text
PERSON ──< RESERVATION >── LISTING ──< LISTING_NIGHT_STATE
  │              │            │
  ├── guest role │            ├── HOST_LISTING_ASSIGNMENT >── HOST (PERSON)
  └── host role  │            └── LISTING_ATTRIBUTE_HISTORY
                 ├──< RESERVATION_EVENT
                 ├──< RESERVATION_GUEST
                 ├──< STAY_NIGHT
                 ├──< PAYMENT / REFUND
                 ├──< HOST_PAYOUT_ALLOCATION
                 └──< REVIEW

SEARCH_SESSION ──< LISTING_IMPRESSION ── LISTING
```

Use a person entity with role-playing guest/host keys or a conformed `dim_person` where policy allows. Do not duplicate the same human as unrelated guest and host identities unless source identity resolution requires it.

## 5. Dimensions

- `dim_listing`: listing type, room type, capacity, property characteristics; Type 2 for meaningful changes.
- `dim_host`: host cohort, account type, region, quality tier; carefully govern sensitive attributes.
- `dim_guest`: cohort/segment and approved profile attributes.
- `dim_market`, `dim_city`, `dim_neighborhood`: geographic hierarchy.
- `dim_booking_channel`, `dim_device`, `dim_search_context`.
- `dim_cancellation_reason`, `dim_reservation_status`.
- `dim_date`: booking date, check-in date, check-out date roles.
- `dim_currency`, `dim_payment_method`, `dim_promotion`.

A listing's host can change. Use an effective-dated host-listing relationship, not a static `host_id` if historical attribution matters.

## 6. Core model design

### `fact_reservation` — accumulating snapshot
**Grain:** one row per canonical reservation.  
Milestones: created, requested, accepted/confirmed, modified, cancelled, check-in, checkout, completed. Measures: booked nights, guest count, quoted amount, confirmed amount, cancellation fee. Keep status history in the event fact. If reservation modifications change dates or amounts, retain versioned reservation events or a reservation-version fact; do not silently overwrite values used in prior reports.

### `fact_stay_night`
**Grain:** one row per reservation × local night actually occupied.  
Measures: allocated nightly room revenue, tax, cleaning fee allocation (if policy permits), occupancy flag. This fact supports stay-date reporting and nightly occupancy. A single reservation with five nights produces five rows.

### `fact_listing_night_state`
**Grain:** one row per listing × local night × state version/effective interval.  
States: available, host-blocked, booked, held, platform-blocked, not-sellable. The current calendar can be derived from effective-dated state changes. Do not store only a current boolean if historical availability is a business KPI.

### Payments and payouts
Model payment attempts, captures, refunds, chargebacks, and payout allocations separately. A guest payment is not identical to host earnings or payout. Reconcile gross booking value, platform revenue, taxes, fees, refunds, host earnings, and cash movement.

## 7. Availability modeling: deep dive

For a listing-night:
- `available` means a guest could book it under the defined rules at that instant;
- `sellable` may exclude dates blocked for operational/legal reasons;
- `booked` means a confirmed reservation occupies it;
- `held` means temporary checkout/approval hold;
- `occupied` means the guest actually stayed.

These are different states. Define occupancy, utilization, and availability metrics carefully.

Example metrics:
- Booking conversion: confirmed reservations / eligible search sessions (session-level denominator).
- Listing conversion: listings booked / distinct listings exposed, with exposure cohort definition.
- Occupancy rate: occupied sellable nights / sellable nights. Alternative denominator should be separately named.
- Cancellation rate: cancelled confirmed reservations / confirmed reservations in a cohort.
- Booking lead time: check-in date minus confirmation timestamp, using local market dates.
- Average daily rate: room revenue / occupied nights; define treatment of fees and taxes.

### SQL: occupancy by market and stay month
```sql
SELECT
  market_key,
  DATE_TRUNC('month', stay_date) AS stay_month,
  SUM(CASE WHEN occupied_flag THEN 1 ELSE 0 END) AS occupied_nights,
  SUM(CASE WHEN sellable_flag THEN 1 ELSE 0 END) AS sellable_nights,
  1.0 * SUM(CASE WHEN occupied_flag THEN 1 ELSE 0 END)
      / NULLIF(SUM(CASE WHEN sellable_flag THEN 1 ELSE 0 END), 0) AS occupancy_rate
FROM fact_listing_night_state
GROUP BY 1, 2;
```
Use the actual stay-night fact for occupied nights and a correctly versioned availability state for the denominator. Avoid joining raw reservation rows to calendar rows without a unique listing-night key.

## 8. Reservation changes and event semantics

Example lifecycle:
`SEARCHED → VIEWED → REQUESTED → CONFIRMED → MODIFIED → CHECKED_IN → COMPLETED`

Alternative paths include declined, expired, cancelled by guest, cancelled by host, platform-cancelled, no-show, and dispute. A modification may:
- change dates and therefore nightly inventory;
- change guest count;
- change price and taxes;
- trigger additional payment or refund;
- alter cancellation terms.

Persist event payload/version, previous/new values or a change reference, actor, effective time, event time, and ingestion time. Build a canonical current reservation snapshot from events. Reconcile availability after reservation changes and cancellation.

## 9. Data pipeline architecture

- Ingest search/impression events, reservation domain events, listing/host CDC, payment provider events, and calendar state changes.
- Bronze: immutable raw events, API snapshots, CDC envelopes, source metadata.
- Silver: canonical identities, normalized status codes, deduplication, effective-dated listing/host records, event validation.
- Gold: reservation snapshot, stay-night, availability, funnel, finance, host performance marts.
- Use streaming for operational availability/funnel freshness where needed; use batch reconciliation for certified occupancy and finance.
- Reprocess affected listing-night ranges when a booking is modified or cancelled.
- Use idempotent event processing and source version ordering.

Airbnb's engineering discussion of its multi-product data architecture notes that high-volume interaction data can justify separate product-facing models, while cross-cutting concepts such as messaging and payments may benefit from shared product-agnostic models. Use this as a design principle, not a claim that your proposed schema matches Airbnb's internal schema.

## 10. Interview dialogue

**Interviewer:** "A guest books 10 nights, cancels after 4, and receives a partial refund. How many rows?"  
**Candidate:** "One reservation snapshot, multiple reservation events, up to ten reservation-night allocations or state transitions depending on the model, one or more payment transactions, and one or more refund transactions. The stay-night fact has only nights actually occupied. The remaining nights return to availability only if the cancellation policy and operational state permit."

**Interviewer:** "The listing is unavailable now. Was it unavailable last month?"  
**Candidate:** "A current listing flag cannot answer historical availability. I'd maintain effective-dated listing-night state changes or periodic snapshots with clear semantics."

**Interviewer:** "A reservation is confirmed but the payment is still processing."  
**Candidate:** "Reservation state and payment state are separate state machines. The model must allow confirmed/pending-payment or other business-defined combinations rather than deriving one state from the other."

**Interviewer:** "A host manages several listings and changes account."  
**Candidate:** "Use a host-listing assignment relationship with validity intervals and a stable host identity mapping. Historical booking attribution follows the relationship effective at the relevant business event."

## 11. Principal-level probes
1. How do you support multiple product lines with shared payment and identity models?
2. How do you migrate from a calendar snapshot to event-sourced availability?
3. How do you reconcile availability under concurrent booking attempts?
4. How do you prevent search event instrumentation changes from invalidating conversion metrics?
5. How do you distinguish host quality from listing quality?
6. How do you handle legal restrictions and data retention across countries?
7. How do you make occupancy comparable across markets with different policies?
8. What is the data contract between marketplace, payments, trust/safety, and finance?

## 12. Common mistakes
- Treating reservation status as payment status.
- Treating a listing's current availability as historical inventory.
- Calculating occupancy using reservations rather than occupied/sellable nights.
- Ignoring local midnight, timezone, and inclusive/exclusive check-out date semantics.
- Assuming a reservation always has one payment and one payout.
- Overwriting reservation modifications and losing historical booking economics.
- Counting search impressions as sessions or unique users.

## 13. Practice exercise

A host blocks two nights, a guest books five nights, modifies the stay to seven nights, then cancels three days before check-in. A second guest books the released dates. Explain:
- listing-night state history;
- reservation version/event model;
- payment/refund and payout effects;
- booking-date versus stay-date reporting;
- how you avoid double-booking and double-counting.

### Scoring rubric (10 points)
| Dimension | Points |
|---|---:|
| Requirements and metric definitions | 1.5 |
| Availability and reservation grains | 2.0 |
| Lifecycle, payments, and stay economics | 2.0 |
| Event/CDC architecture and corrections | 1.0 |
| Performance and temporal modeling | 1.0 |
| Governance and quality | 1.0 |
| Principal-level trade-offs | 1.5 |

## 14. Further reading
- Airbnb Engineering, [Scaling beyond one: evolving data architecture for a multi-product world](https://airbnb.tech/uncategorized/scaling-beyond-one-how-airbnb-evolved-its-data-architecture-for-a-multi-product-world/).
- AWS, [Dimensional modeling in Amazon Redshift](https://aws.amazon.com/blogs/big-data/dimensional-modeling-in-amazon-redshift/).
