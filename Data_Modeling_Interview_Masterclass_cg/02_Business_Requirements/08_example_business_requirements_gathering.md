# Lesson 8: Example --- Business Requirements Gathering

## Scenario

A multi-city ride-sharing company wants a data warehouse to improve
marketplace efficiency, understand trip economics, monitor service
quality, and support executive reporting.

The initial request is:

> "Build a single analytics model for rides, drivers, customers,
> revenue, and operations."

This is intentionally broad. The task is to conduct discovery before
designing the schema.

## 1. Step one: clarify the business outcomes

Potential outcomes: 1. Improve successful trip completion. 2. Reduce
rider wait time and cancellations. 3. Improve driver utilization and
retention. 4. Understand gross bookings, platform revenue, and driver
earnings. 5. Monitor service quality by city, zone, and time. 6.
Reconcile operational trip activity with payment and settlement records.

Confirm which outcomes are in the first release and who owns each KPI.

## 2. Step two: identify users and decisions

  -----------------------------------------------------------------------
  User                    Decisions               Likely data needs
  ----------------------- ----------------------- -----------------------
  Executive team          City growth,            Daily/weekly KPIs
                          marketplace health      

  City operations         Where to rebalance      Zone/time demand and
                          supply                  availability

  Finance                 Revenue and payout      Payment, refund,
                          reconciliation          adjustment, settlement

  Driver operations       Driver utilization and  Driver-day activity and
                          retention               lifecycle

  Product analytics       Funnel and experience   Request, matching,
                          improvements            trip, rating events

  Support                 Investigate complaints  Trip and support-case
                                                  history
  -----------------------------------------------------------------------

This immediately suggests that one physical table is unlikely to be the
right serving structure for every consumer.

## 3. Step three: define the core processes

Candidate processes: - Ride request. - Driver offer and acceptance. -
Matching and assignment. - Trip lifecycle. - Cancellation. - Payment
attempt and capture. - Refund and adjustment. - Driver availability. -
Driver payout. - Rating and support interaction.

Confirm whether the source systems record each event reliably and
whether identifiers link the processes.

## 4. Step four: clarify metric definitions

  -----------------------------------------------------------------------
  Metric                              Questions to resolve
  ----------------------------------- -----------------------------------
  Completed trips                     Which terminal statuses count? Are
                                      test trips excluded?

  Cancellation rate                   Denominator: requests, matched
                                      rides, or accepted trips?

  Wait time                           From request to match, driver
                                      arrival, or pickup?

  Gross bookings                      Fare before or after discounts?
                                      Include taxes/tips?

  Platform revenue                    Which fees and incentives are
                                      included?

  Driver earnings                     Gross fare share, incentives, tips,
                                      adjustments, or net payout?

  Utilization                         Online time, engaged time, or trip
                                      time?
  -----------------------------------------------------------------------

Do not assume these definitions. Get agreement from business and finance
owners.

## 5. Step five: establish fact grains

Initial candidate grains:

  -----------------------------------------------------------------------
  Fact                                Grain statement
  ----------------------------------- -----------------------------------
  `fact_ride_request`                 One row per ride request

  `fact_trip`                         One row per trip

  `fact_payment_attempt`              One row per payment attempt

  `fact_payment_transaction`          One row per
                                      captured/refunded/adjusted payment
                                      transaction, depending on event
                                      semantics

  `fact_driver_availability`          One row per driver per defined time
                                      interval or snapshot

  `fact_driver_day`                   One row per driver per service date

  `fact_rating`                       One row per rating submission

  `fact_support_interaction`          One row per support interaction
  -----------------------------------------------------------------------

The exact set depends on the source event model and reporting
requirements. A trip request and a completed trip are not
interchangeable.

## 6. Step six: identify dimensions

Potential shared dimensions: - Date and time (including role-playing
request, pickup, drop-off, and payment dates). - Rider. - Driver. - City
and service zone. - Vehicle. - Service type. - Cancellation reason. -
Payment method. - Promotion or campaign. - Platform/channel.

Clarify whether rider and driver are separate business roles or whether
a person can occupy both roles. Decide how historical attributes such as
driver tier, city assignment, and rider segment should be tracked.

## 7. Step seven: query patterns

Example queries: 1. Completed trips by city, zone, and hour. 2.
Request-to-match and request-to-pickup latency percentiles. 3.
Cancellation rate by lifecycle stage and reason. 4. Gross bookings,
platform revenue, and driver earnings by service date. 5. Payment
success rate by payment provider. 6. Driver online time versus engaged
time. 7. Repeat rider rate by acquisition cohort. 8. Ratings by service
type and city. 9. Support contacts per thousand completed trips.

Map each query to the appropriate fact grain. For example, payment
success rate belongs to payment attempts, not directly to completed
trips.

## 8. Step eight: latency, volume, and scale

Ask: - How many requests and trips occur daily? - What are peak events
per second during major commute periods? - How many GPS points are
generated per trip? - Does operations require seconds-level freshness? -
Can finance reporting be daily and reconciled? - How many years of
detailed trip and location history are needed? - What are the query
concurrency expectations?

Separate high-volume telemetry from trip-level facts if their grain,
retention, and query patterns differ.

## 9. Step nine: history, quality, and governance

History: - Preserve trip lifecycle events where auditability is
required. - Track selected driver/rider attributes historically if
past-state analysis is needed. - Define event-time and ingestion-time
semantics. - Define late event and correction windows.

Quality: - Unique request and trip identifiers. - Valid lifecycle
transitions. - Referential integrity or controlled unknown-member
handling. - Fare and payment reconciliation. - Timestamp ordering and
timezone consistency. - Duplicate event detection.

Governance: - Restrict personal and precise location data. - Define
purpose-based access. - Apply approved retention and deletion
policies. - Record ownership and metric definitions.

## 10. Illustrative high-level model

``` text
                 dim_date / dim_time
                         |
dim_rider ------ fact_ride_request ------ dim_zone
                         |
                      dim_city

dim_driver -------- fact_trip -------- dim_vehicle
                         |
                    dim_service_type
                         |
                     dim_date (role-playing)

dim_payment_method -- fact_payment_attempt -- dim_date
                         |
                    payment_provider

dim_driver -------- fact_driver_day -------- dim_date
```

This is a conceptual illustration, not a final physical schema. The
payment and trip processes remain separate because their grains differ.

## 11. Interviewer--candidate conversation

**Interviewer:** The CEO wants one model for all ride-sharing analytics.
What would you build?

**Candidate:** I'd first clarify the executive decisions and the
priority use cases. I would not start by forcing every process into one
fact table. Ride requests, completed trips, payment attempts, driver
availability, and payouts have different grains and lifecycle semantics.

**Interviewer:** The CEO wants daily revenue by city and driver. Can you
join trip and payment facts?

**Candidate:** I'd clarify the revenue definition and the attribution
relationship. I'd aggregate trip-level measures and payment transactions
independently to a compatible reporting grain, then combine them using
conformed dimensions or a well-defined bridge where needed. A raw join
could multiply rows if a trip has multiple payment attempts or
adjustments.

**Interviewer:** Operations wants data within 30 seconds, but finance
wants reconciled daily figures.

**Candidate:** I'd propose separate serving paths with shared
identifiers and metric definitions: a low-latency operational view and a
reconciled finance dataset. The latter can incorporate late events and
settlement adjustments. I'd make freshness and finalization status
visible to consumers.

**Interviewer:** How do you keep driver city assignment historically
accurate?

**Candidate:** I'd determine whether city assignment is a slowly
changing attribute and whether analysis needs the historical city at
trip time. If yes, I'd preserve effective-dated versions and associate
the trip to the correct version, or retain an explicit trip-time city
key where that matches the business semantics.

## 12. Requirements document template

Use this template in future case studies:

### Business objective

-   Desired outcome:
-   Business owner:
-   Success criteria:

### Users and use cases

-   Consumer groups:
-   Decisions:
-   Priority questions:

### Metrics

-   Metric name:
-   Definition:
-   Formula:
-   Inclusion/exclusion:
-   Time basis:
-   Owner:

### Data and model

-   Business processes:
-   Candidate entities:
-   Fact grains:
-   Dimensions:
-   Identifiers and relationships:
-   Historical requirements:

### Non-functional requirements

-   Freshness SLO:
-   Query latency:
-   Daily and peak volume:
-   Growth:
-   Concurrency:
-   Availability/recovery:

### Governance

-   Data classification:
-   Access:
-   Retention:
-   Deletion:
-   Audit:
-   Ownership:

### Assumptions and open questions

-   Assumption:
-   Risk if wrong:
-   Owner to confirm:
-   Decision date:

## 13. Practice exercise

Repeat the same discovery process for an online marketplace. Do not draw
tables until you have written: 1. Three business outcomes. 2. Five
stakeholder questions. 3. Five precise metric definitions. 4. At least
four distinct fact grains. 5. Three freshness or scale constraints. 6.
Two history requirements. 7. Three data-quality checks.

## 14. Principal-level follow-ups

-   How would you make city, driver, and rider dimensions consistent
    across regions?
-   How would you support new ride types without breaking downstream
    contracts?
-   How would you separate operational telemetry from governed
    analytical data?
-   How would you define ownership for gross bookings, platform revenue,
    and driver earnings?
-   How would you manage a 10x growth in trip and location events?
-   How would you prove that financial measures reconcile to settlement?

## 15. Final checklist

-   [ ] Business outcomes and decision-makers are known.
-   [ ] Metrics have precise definitions and owners.
-   [ ] Each fact has a clear grain.
-   [ ] Query patterns map to facts and dimensions.
-   [ ] Latency and scale targets are measurable.
-   [ ] Historical and retention requirements are explicit.
-   [ ] Quality, privacy, and reconciliation are included.
-   [ ] Assumptions and unresolved questions are documented.

## Key takeaway

**Requirements gathering turns an ambiguous request into a testable
contract for the data model, its consumers, and its operating
characteristics.**
