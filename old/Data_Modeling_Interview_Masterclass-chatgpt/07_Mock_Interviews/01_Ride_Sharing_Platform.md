# Mock Interview 1 — Ride-Sharing Platform Data Warehouse

**Interview level:** Senior / Staff / Principal Data Engineer  
**Suggested duration:** 75–90 minutes  
**Primary challenge:** Model a real-time, multi-sided marketplace where a single customer request can generate multiple dispatch attempts, driver assignments, trip lifecycle events, location telemetry, fare adjustments, and payment outcomes.

## 1. Interview brief

> Design an analytical data platform for a ride-sharing company operating in multiple countries. Product, marketplace operations, finance, and growth teams need reliable metrics for bookings, completed trips, cancellations, driver utilization, ETA accuracy, gross bookings, incentives, and contribution margin.

The interviewer is testing whether you can move from a vague business prompt to a governed, scalable model. Do not begin by drawing a star schema. Start by establishing business processes, grain, definitions, and workload requirements.

### What a strong candidate demonstrates
- Separates a rider's **request**, a **dispatch/offer attempt**, an accepted **assignment**, and a **trip**.
- Distinguishes event-level facts from a current-state snapshot.
- Models trip lifecycle and financial components without double counting.
- Treats location pings as a separate, high-volume telemetry domain.
- Handles multiple drivers offered a trip, driver/vehicle changes, cancellations, retries, and late events.
- Defines metrics precisely and makes trade-offs explicit.

## 2. Start with discovery, not tables

### Candidate opening

**Candidate:** "Before choosing entities, I want to identify the business processes and the decisions the warehouse must support. Is the main objective financial reporting, marketplace operations, product funnel analysis, or all three? These imply different grains and freshness requirements."

### Clarifying questions and plausible interviewer answers

| Area | Ask | Assumption for this exercise |
|---|---|---|
| Marketplace | Can one rider request receive multiple driver offers? | Yes; dispatch may fan out to many drivers |
| Lifecycle | Can a trip be reassigned after acceptance? | Yes, rarely |
| Trip definition | Does a trip begin at request, acceptance, pickup, or drop-off? | Each is a distinct milestone |
| Cancellation | Can rider, driver, or system cancel? | Yes; retain actor and reason |
| Pricing | Are quote, final fare, tips, tolls, taxes, and incentives separate? | Yes |
| Geography | Is city/country important? | Yes; city and service zone are conformed dimensions |
| Telemetry | Are GPS pings required for analytics? | Yes, but not in the core trip fact |
| Freshness | What does operations need? | 1–5 minute operational aggregates; finance daily certified reporting |
| Scale | What volume? | Illustrative: millions of requests/day, billions of location pings/month |
| Retention | How long? | Financial facts per policy; raw GPS shorter and purpose-limited |

**Follow-up:** "What if the business says 'booking' means request in one dashboard and completed trip in another?"  
**Strong response:** Establish a metric contract. Publish separate measures such as `ride_requests`, `accepted_requests`, `trips_started`, and `completed_trips`. Never hide semantic differences behind one ambiguous `booking_count`.

## 3. Business process inventory and grain

Use a bus matrix before detailed schema design.

| Business process | Proposed fact | Grain |
|---|---|---|
| Rider request funnel | `fact_ride_request` | One row per rider request |
| Dispatch | `fact_driver_offer` | One row per driver offer attempt for a request |
| Trip lifecycle | `fact_trip` | One row per canonical trip |
| Trip milestones | `fact_trip_event` | One row per lifecycle event |
| Trip financials | `fact_trip_charge_component` | One row per trip × financial component × posting/version |
| Driver supply | `fact_driver_supply_interval` | One row per driver × time interval × service zone |
| Location telemetry | `fact_driver_location_ping` | One row per device ping |
| Daily marketplace state | `fact_city_zone_daily_snapshot` | One row per date × city × zone × service category |

**Grain discipline:** `fact_trip` is not one row per status event. If you join it directly to 12 trip events, the trip-level fare will be repeated 12 times. Keep event facts separate and aggregate to the required grain before joining.

## 4. Conceptual model

```text
RIDER ──< RIDE_REQUEST ──< DRIVER_OFFER >── DRIVER
                  │                │
                  │                └── assignment / acceptance
                  ▼
                TRIP ──< TRIP_EVENT
                  │
                  ├──< TRIP_CHARGE_COMPONENT
                  ├──< TRIP_STOP / WAYPOINT
                  └──< TRIP_PAYMENT_REFERENCE

DRIVER ──< DRIVER_VEHICLE_ASSIGNMENT >── VEHICLE
CITY ──< SERVICE_ZONE
TRIP ── pickup_zone / dropoff_zone
```

This is a conceptual relationship view, not the final warehouse schema. A request may have zero, one, or many offers. A request can result in no trip, one trip, or—in exceptional split/pooled products—multiple service legs. Confirm product semantics before enforcing one-to-one assumptions.

## 5. Dimensional model

### Core dimensions
- `dim_date`: calendar, fiscal, holiday, and local-business-day attributes.
- `dim_time`: time-of-day bucket where useful; do not join every timestamp to a minute dimension by default.
- `dim_rider`: privacy-aware rider profile attributes; use a surrogate key and controlled PII access.
- `dim_driver`: driver attributes with SCD Type 2 for analytically meaningful changes such as tier or onboarding cohort.
- `dim_vehicle`: vehicle class, fuel type, capacity, accessibility attributes.
- `dim_city`, `dim_service_zone`: conformed geography dimensions.
- `dim_ride_product`: economy, premium, pooled, accessible, delivery, etc.
- `dim_cancellation_reason`: normalized reason taxonomy.
- `dim_device_app_version`: telemetry/app diagnostics where justified.

Role-playing dimensions: `pickup_date_key`, `dropoff_date_key`, `request_date_key`; `pickup_zone_key` and `dropoff_zone_key` both reference the service-zone dimension.

### `fact_trip` — accumulating snapshot

**Grain:** one row per canonical trip ID.  
**Purpose:** lifecycle duration, completion, cancellation, and trip-level outcome analysis.

Example columns:
```sql
CREATE TABLE fact_trip (
    trip_key              BIGINT,
    trip_id               VARCHAR(64) NOT NULL,
    request_id            VARCHAR(64) NOT NULL,
    rider_key             BIGINT,
    assigned_driver_key   BIGINT,
    vehicle_key           BIGINT,
    ride_product_key      BIGINT,
    city_key              BIGINT,
    pickup_zone_key       BIGINT,
    dropoff_zone_key      BIGINT,
    request_ts            TIMESTAMP,
    accepted_ts           TIMESTAMP,
    driver_arrived_ts     TIMESTAMP,
    pickup_ts             TIMESTAMP,
    dropoff_ts            TIMESTAMP,
    terminal_ts           TIMESTAMP,
    trip_status           VARCHAR(32),
    cancellation_actor    VARCHAR(32),
    cancellation_reason_key BIGINT,
    quoted_fare_minor     BIGINT,
    final_fare_minor      BIGINT,
    currency_code         CHAR(3),
    source_updated_at     TIMESTAMP,
    warehouse_updated_at  TIMESTAMP,
    record_version        BIGINT
);
```
This is a warehouse illustration; adapt types and constraints to the target engine. Store money in integer minor units or fixed-precision decimals with an explicit currency. Never sum unlike currencies without a documented FX conversion policy.

### `fact_trip_event` — immutable event fact

**Grain:** one row per unique trip lifecycle event.  
Fields: `event_id`, `trip_id`, `event_type`, `event_ts`, `ingested_at`, `actor_type`, `actor_id`, `reason_code`, `source_sequence`, `event_version`, `is_retracted`.

Examples: requested, offer accepted, driver arrived, pickup, dropoff, rider cancelled, driver cancelled, trip reopened/corrected. Preserve event history; derive current state deterministically.

### `fact_driver_offer`

**Grain:** one row per request × driver × offer attempt.  
Measures: offer response seconds, accepted flag, expired flag, offer distance, dispatch rank. A request-level acceptance rate must aggregate offers carefully: offers accepted / eligible offers, not accepted offers / requests unless that is the explicitly defined metric.

### Financial component fact

**Grain:** one row per trip × charge component × financial posting.  
Components may include base fare, distance, time, surge, toll, tax, tip, promotion, driver incentive, platform fee, refund, adjustment. A refund is a new negative financial event or reversal record—not an overwrite of the original charge.

### Location ping fact

**Grain:** one row per device ping. Keep it separate from the trip star because its volume and access patterns differ dramatically. Apply strict retention, access, aggregation, and privacy controls. Consider geospatial aggregation to zone/time buckets for most analytics.

## 6. Lifecycle and event processing

A common pipeline:
1. Application services publish domain events with stable IDs and entity IDs.
2. Kafka/MSK transports events; schema registry or equivalent enforces compatibility.
3. Bronze stores immutable raw payloads plus topic, partition, offset, event time, ingestion time.
4. Silver validates, deduplicates, normalizes event types, resolves identifiers, and handles corrections.
5. Gold builds trip snapshots, event facts, marketplace aggregates, and finance-certified marts.

### Deduplication and ordering
- Use a source event ID where available; otherwise construct a deterministic key only if the source contract guarantees its uniqueness.
- Kafka offsets identify transport position, not business uniqueness.
- Do not assume event arrival order equals event occurrence order.
- Use `event_ts`, source sequence/version, and explicit transition rules.
- Handle late events with watermarks for streaming views and correction-capable batch reconciliation.
- Make processing idempotent: replaying the same event must not create a second business event.

### Hard interviewer probe
**Interviewer:** "The `dropoff` event arrives before `pickup`. What do you do?"  
**Candidate:** "I don't discard it or force an invalid transition. I persist both immutable events, flag the entity as temporarily out of order, and derive the current snapshot using source sequence/version and valid transition logic. If no reliable sequence exists, I use event time with a documented tie-breaker and quarantine ambiguous histories. A later reconciliation can repair the snapshot without losing raw evidence."

## 7. Metrics and SQL

### Define metrics
- Request conversion = completed trips / eligible ride requests, with cohort/time-window definition.
- Driver acceptance rate = accepted offers / eligible offers.
- Completion rate = completed trips / accepted trips, or / requests—name the denominator explicitly.
- Driver utilization = engaged driver time / available online time; define treatment of idle, break, and repositioning time.
- Gross bookings = customer-facing trip charges before specified exclusions; finance must approve exact inclusions.
- Contribution margin = recognized revenue less defined variable costs and incentives; do not call gross fare "revenue" automatically.

### Example: daily completed trips and gross fare
```sql
SELECT
    CAST(t.dropoff_ts AS DATE) AS trip_date,
    c.city_name,
    p.product_name,
    COUNT(*) AS completed_trips,
    SUM(t.final_fare_minor) AS fare_minor_units
FROM fact_trip t
JOIN dim_city c ON t.city_key = c.city_key
JOIN dim_ride_product p ON t.ride_product_key = p.ride_product_key
WHERE t.trip_status = 'COMPLETED'
  AND t.dropoff_ts >= :start_ts
  AND t.dropoff_ts <  :end_ts
GROUP BY 1, 2, 3;
```

### Example: request funnel
```sql
WITH request_level AS (
  SELECT
      r.request_id,
      MAX(CASE WHEN e.event_type = 'OFFER_ACCEPTED' THEN 1 ELSE 0 END) AS accepted,
      MAX(CASE WHEN e.event_type = 'TRIP_COMPLETED' THEN 1 ELSE 0 END) AS completed
  FROM fact_ride_request r
  LEFT JOIN fact_trip_event e ON e.request_id = r.request_id
  WHERE r.request_ts >= :start_ts AND r.request_ts < :end_ts
  GROUP BY r.request_id
)
SELECT COUNT(*) AS requests,
       SUM(accepted) AS accepted_requests,
       SUM(completed) AS completed_requests
FROM request_level;
```
The aggregation first returns to request grain, preventing event fanout from inflating counts.

## 8. Physical design and scale

- Partition large event/ping tables by a date derived from event time or ingestion time according to dominant queries and late-arrival strategy.
- Avoid high-cardinality partitioning by rider or driver.
- Cluster/sort on common filters such as event date, city, and possibly trip ID based on measured workloads.
- Keep raw pings at a shorter retention than aggregated trip/zone metrics.
- Pre-aggregate city × zone × 5-minute supply/demand for operations; preserve drill-through to governed detail.
- Use approximate distinct counts only where the product owner accepts error bounds.
- Separate operational low-latency serving from finance-grade warehouse reporting where freshness and consistency requirements conflict.
- Benchmark data layout against representative queries; do not claim one partitioning design is universally optimal.

## 9. Data quality, governance, and reliability

| Check | Example |
|---|---|
| Uniqueness | One canonical `trip_id` in current trip snapshot |
| Referential integrity | Every trip references a valid product/city key or explicit unknown member |
| Temporal validity | `request_ts <= accepted_ts <= pickup_ts <= dropoff_ts` for completed trips |
| Financial reconciliation | Fare components reconcile to trip totals within currency precision |
| Completeness | Completed trip has required terminal timestamps |
| Freshness | Operational aggregates meet agreed lag SLO |
| Volume anomaly | Requests by city/product do not unexpectedly collapse |
| Privacy | GPS data access logged, minimized, and retention enforced |

Deletion requests must propagate through raw, curated, derived, cached, and downstream copies according to policy. Keep audit evidence without retaining prohibited personal content. Distinguish deletion from de-identification and legal retention exceptions.

## 10. Interview conversation: changing requirements

**Interviewer:** "The COO wants to know why a city has high demand but poor completion."  
**Candidate:** "I'd combine request funnel, dispatch offers, driver supply intervals, ETA, and cancellation facts at consistent city-zone-time grains. I'd separate supply shortage from dispatch inefficiency and rider/driver cancellation."

**Interviewer:** "A driver receives five offers for one request. How do you count that?"  
**Candidate:** "Five rows in the offer fact, one row in the request fact, and at most one canonical trip if one offer is accepted. I would not join offers to trip financials and sum fare without pre-aggregating."

**Interviewer:** "The driver changes vehicle during the trip."  
**Candidate:** "I would model vehicle assignment as a time-bounded relationship/event. If the product allows multiple vehicles per trip, a single `vehicle_key` on the trip snapshot is insufficient; use a trip-vehicle assignment bridge with effective timestamps, and define which vehicle is attributed to which measure."

**Interviewer:** "Finance says your fare number differs from settlement."  
**Candidate:** "I would reconcile operational fare, customer charge, payment processor balance movements, refunds, incentives, and settlement separately. A trip fare is not automatically cash collected or recognized revenue."

## 11. Principal-level probes

1. How would you define a canonical trip ID across service migrations?
2. How do you roll out a new event schema without breaking downstream consumers?
3. What is your strategy for replaying six months of events?
4. How do you partition ownership across Marketplace, Mobility, Payments, and Finance?
5. How do you make metrics consistent across city dashboards and executive reporting?
6. What are the SLOs for ingestion lag, metric freshness, completeness, and correction time?
7. How do you control cost when raw telemetry grows faster than business facts?
8. How would you migrate a legacy trip model while dashboards continue running?

A principal answer should describe contracts, ownership, migration phases, dual-run validation, observability, rollback, and decision-making—not only a technology choice.

## 12. Common weak answers

- Treating request, offer, assignment, and trip as synonyms.
- Putting all lifecycle events into one mutable row and losing history.
- Joining trip to event or offer facts before aggregating to compatible grain.
- Storing only a final fare and losing adjustments/refunds.
- Treating GPS pings as ordinary dimension attributes.
- Ignoring local time zones and city-specific operating calendars.
- Claiming exactly-once processing without defining end-to-end idempotency and sink semantics.

## 13. Practice exercise

Design a model for pooled rides where one request may share a vehicle with another rider and a trip contains multiple pickup/drop-off stops.

Your answer must define:
- request, trip, passenger-leg, stop, and driver assignment grains;
- how shared fare allocation works;
- how to represent a rider cancelling after pickup;
- how to avoid counting the same vehicle distance once per passenger;
- which metrics are trip-level versus passenger-leg-level.

### Evaluation rubric (10 points)
| Dimension | Points |
|---|---:|
| Requirement discovery and metric contracts | 1.5 |
| Correct process/grain separation | 2.0 |
| Lifecycle and financial correctness | 1.5 |
| Event processing and late-data strategy | 1.5 |
| Scale, physical design, and cost | 1.0 |
| Quality, privacy, governance | 1.0 |
| Principal-level leadership and trade-offs | 1.5 |

**Excellent answer:** explicitly states grain before schema, identifies fanout risks, preserves event and financial history, and explains how the design evolves.

## 14. Further reading
- Uber, [Fulfillment Platform re-architecture](https://www.uber.com/us/en/blog/fulfillment-platform-rearchitecture/) — domain modeling and event/API evolution.
- AWS, [Amazon Redshift table design best practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-tables-best-practices.html) — physical design decisions.
- AWS, [Redshift best practices](https://docs.aws.amazon.com/redshift/latest/dg/best-practices.html) — workload-aware warehouse design.
