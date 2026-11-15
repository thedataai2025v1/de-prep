# Lesson 6: Practice — Ride-Sharing Platform

## Interview prompt

Design a high-level analytical model for a ride-sharing platform. Riders request trips, drivers accept or reject offers, trips are completed or cancelled, payments are collected, and drivers receive payouts. Operations needs marketplace visibility; finance needs reconciled economics.

## 1. Clarify key definitions

Ask:
- What is a request versus a matched ride versus a trip?
- Can a request be offered to multiple drivers?
- Can a driver reject or time out?
- Can trips be reassigned?
- How are cancellations classified?
- Can one trip have multiple payment attempts?
- How are tips, tolls, incentives, refunds, and payouts represented?
- Does operations need seconds-level data?
- What is the location-data retention policy?

## 2. Candidate conceptual diagram

```text
RIDER 1 ─────< RIDE_REQUEST
RIDE_REQUEST 1 ─────< DRIVER_OFFER >──── 1 DRIVER
RIDE_REQUEST 0..1 ───── TRIP
TRIP 1 ─────< TRIP_EVENT
TRIP 1 ─────< PAYMENT_ATTEMPT
TRIP 1 ─────< PAYMENT_TRANSACTION
DRIVER 1 ─────< DRIVER_AVAILABILITY_INTERVAL
DRIVER 1 ─────< DRIVER_PAYOUT_LINE
TRIP ─────< TRIP_PAYOUT_ALLOCATION >──── DRIVER_PAYOUT_LINE
```

The request-to-trip relationship may be more complex if one request can be rematched, split, or converted into multiple legs. Confirm actual business semantics.

## 3. Candidate fact grains

| Fact | Grain |
|---|---|
| Ride request | One row per rider request |
| Driver offer | One row per driver offer for a request |
| Trip | One row per trip |
| Trip event | One row per trip lifecycle event |
| Payment attempt | One row per attempt |
| Payment transaction | One row per capture/refund/adjustment |
| Driver availability | One row per driver availability interval |
| GPS telemetry | One row per location ping |
| Driver payout line | One row per payout component/line |

GPS pings should not be placed at trip grain. Their volume and retention profile are different.

## 4. Metrics and model implications

- Request conversion: define which requests count and what event qualifies as conversion.
- Driver acceptance rate: accepted offers divided by eligible offers, with timeout/rejection rules.
- Rider wait time: specify request-to-match, request-to-arrival, or request-to-pickup.
- Trip cancellation rate: define denominator and cancellation ownership.
- Driver utilization: define engaged time relative to online or available time.
- Gross bookings: define fare components and discounts.
- Platform revenue: define fees, incentives, taxes, and adjustments.

## 5. Interviewer–candidate conversation

**Interviewer:** A request is offered to five drivers before one accepts. How do you model acceptance rate?

**Candidate:** I'd create an offer-level event/fact with one row per driver offer and its outcome. Acceptance rate can then be calculated using eligible offers as the denominator, with explicit treatment of timeout and withdrawn offers.

**Interviewer:** How do you calculate driver utilization?

**Candidate:** I'd define the numerator and denominator first. For example, engaged minutes divided by online minutes for a driver over a service interval. I'd model availability intervals and trip intervals separately, then handle overlap, breaks, and timezone boundaries consistently.

**Interviewer:** Why separate trip and payment?

**Candidate:** A trip can have multiple payment attempts, captures, refunds, or adjustments. They are different processes and grains. Combining raw rows can multiply trip measures and obscure financial reconciliation.

## 6. Follow-up questions
1. How do you model driver rematching?
2. How do you calculate cancellation rate at different funnel stages?
3. How do you model surge pricing?
4. How do you handle GPS data at scale?
5. How do you reconcile trip fare with settlement and payout?
6. How do you track a driver's city or service tier historically?

## 7. Principal-level discussion
- Design separate operational and reconciled analytical serving paths.
- Establish common definitions for request, trip, gross bookings, and driver earnings.
- Govern sensitive location and rider data.
- Plan for peak-hour bursts and replay/backfill.
- Support multi-city and multi-currency reporting.

## Evaluation rubric
- Process decomposition: 20%
- Grain and relationships: 25%
- Metric definitions: 20%
- Financial correctness: 15%
- Scale, latency, and privacy: 10%
- Communication: 10%

## Key takeaway
**Ride-sharing is a marketplace of interacting processes. Keep requests, offers, trips, telemetry, payments, and payouts at their own grains.**
