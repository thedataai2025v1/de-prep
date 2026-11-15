# Lesson 37 — Mock Interview: Airbnb-Style Short-Term Rental Platform

**Level:** Senior / Staff / Principal Data Engineer
**Duration:** 60-90 minutes
**Case type:** End-to-end source-to-warehouse data modeling

## How to use this case in an interview

Treat this as a full interview exercise. Focus on modeling a two-sided marketplace with listings, bookings, reviews, and payments across a global platform.

## Learning Objectives

- Model a two-sided marketplace (hosts and guests)
- Handle listing lifecycle and pricing changes
- Design booking lifecycle with multi-step state transitions
- Track reviews and ratings with temporal context
- Handle geographic and temporal patterns

## Interview Dialogue

**Interviewer:** We're building a data platform for a short-term rental marketplace like Airbnb. We need to model listings, bookings, reviews, payments and host analytics. Design the data warehouse.

**Candidate:** I'd start by identifying the core business processes:

1. **Listing Management** — hosts create, update, deactivate listings with pricing and availability
2. **Booking Lifecycle** — search → request → confirmation → check-in → check-out → review
3. **Payments & Settlements** — guest payments, host payouts, platform fees
4. **Reviews & Ratings** — post-stay reviews from both guests and hosts
5. **Search & Discovery** — search queries, impressions, click-through

**Interviewer:** Walk me through the source system design.

**Candidate:** The operational systems likely have:

**Source Systems:**

**Listings Table**
- listing_id, host_id, title, description
- property_type, room_type, bedrooms, bathrooms, max_guests
- city, country, latitude, longitude
- status (active, inactive, suspended), created_at, updated_at

**Hosts Table**
- host_id, email, first_name, last_name
- registration_date, status, superhost_flag
- country, response_rate

**Pricing Rules Table**
- listing_id, base_price_night, currency
- weekend_price, seasonal_adjustment
- minimum_stay, maximum_stay
- effective_from, effective_to

**Bookings Table**
- booking_id, listing_id, guest_id, check_in_date, check_out_date
- status (requested, confirmed, checked_in, checked_out, cancelled)
- total_price, cleaning_fee, service_fee, host_fee
- created_at, updated_at

**Payments Table**
- payment_id, booking_id, guest_id, amount, currency
- payment_method, paid_at, status
- host_payout_amount, platform_fee

**Reviews Table**
- review_id, booking_id, reviewer_id (guest or host), reviewee_id
- rating (1-5), comment, created_at

**Search Events Table**
- search_id, guest_id, search_params (JSON), timestamp
- impressions_count, clicks_count, resulting_booking_id

**Interviewer:** How would you design the warehouse fact and dimension tables?

**Candidate:** I'd model each process at its natural grain:

**Core Dimensions:**

**Dim_Listing**
- listing_sk (surrogate key), listing_id, host_sk
- title, property_type_sk, room_type_sk
- bedrooms, bathrooms, max_guests
- city_sk, country_sk
- status, created_date_sk
- current_flag, start_date, end_date (SCD Type 2 for listing changes)

**Dim_Host**
- host_sk, host_id, email, first_name, last_name
- registration_date_sk, status, superhost_flag
- country_sk, response_rate

**Dim_Guest**
- guest_sk, guest_id, email, first_name, last_name
- registration_date_sk, country_sk

**Dim_Date** — standard date dimension (for check-in, check-out dates)
**Dim_Location** — geographic hierarchies (country → region → city → neighborhood)

**Core Fact Tables:**

**Fact_Bookings** (transactional fact, one row per booking)
- booking_sk, listing_sk, guest_sk, host_sk
- check_in_date_sk, check_out_date_sk
- nights_count, total_price, cleaning_fee
- service_fee, host_fee, status_sk
- created_date_sk

**Fact_Bookings_Daily** (periodic snapshot fact, one row per booking per day of stay)
- booking_sk, date_sk, listing_sk, guest_sk
- day_of_stay (1, 2, 3, ...), price_per_night
- occupancy_flag (1 if listing occupied on this date)

**Fact_Bookings_Daily** enables occupancy analysis without complex date arithmetic. It's a degenerate fact table expanded across the stay period.

**Fact_Payments** (transactional fact)
- payment_sk, booking_sk, guest_sk, date_sk
- amount, currency_sk, payment_method_sk
- host_payout_amount, platform_fee, status_sk

**Fact_Reviews** (transactional fact)
- review_sk, booking_sk, reviewer_sk, reviewee_sk
- date_sk, rating_score
- review_type_sk (guest_review, host_review)

**Fact_Search_Events** (transactional fact)
- search_sk, guest_sk, date_sk, time_sk
- impressions_count, clicks_count
- resulting_booking_sk (nullable), conversion_flag

**Interviewer:** How do you handle listing pricing changes over time?

**Candidate:** Pricing is one of the most critical dimensions. I'd use:

**Dim_Pricing** (SCD Type 2)
- pricing_sk, listing_sk, base_price_night, currency_sk
- weekend_price, minimum_stay, maximum_stay
- effective_from_sk, effective_to_sk, current_flag

This captures pricing history so we can analyze:
- Revenue impact of price changes
- Seasonal pricing patterns
- Price competitiveness in the market

**Interviewer:** How do you handle the search-to-booking conversion?

**Candidate:** This requires connecting search events to bookings:
- **Fact_Search_Events** tracks searches with impressions and clicks
- **resulting_booking_sk** links to the booking if one resulted
- **conversion_flag** marks whether a search led to a booking
- Time-to-booking can be calculated from search timestamp to booking creation

For deeper analysis, we'd need to attribute bookings to the right search, handling cases where a guest searches multiple times before booking.

**Interviewer:** What about the geographic dimension?

**Candidate:** Geographic data has multiple hierarchies:
- **Dim_Location** for standard hierarchies (country → region → city → neighborhood)
- **Dim_Timezone** for scheduling (check-in/check-out in local time)
- **Dim_Market** for business units (groups of cities for regional reporting)

The listing dimension joins to location for geographic reporting. Hosts are also geographically distributed, which affects host analytics.

## Key Takeaways

| Concept | Application in this case |
|---------|--------------------------|
| Two-sided marketplace | Hosts and guests as separate dimensions |
| SCD Type 2 | Track listing changes, pricing changes over time |
| Periodic snapshot facts | Fact_Bookings_Daily for occupancy analysis |
| Transactional facts | Bookings, payments, reviews at their natural grain |
| Degenerate facts | Search events without traditional dimension keys |
| Time intelligence | Occupancy rates, length of stay, seasonality |
| Geographic hierarchies | Country → city → neighborhood for listings |
| Conversion tracking | Search → booking attribution |
| Financial reconciliation | Guest payment = host payout + platform fee + fees |
| Multi-currency support | Currency dimension for global operations |

## Interview Scoring Rubric

| Level | What to look for |
|-------|------------------|
| **Junior** | Can identify basic fact and dimension tables |
| **Senior** | Separates business processes, uses SCD for pricing, understands periodic snapshots |
| **Staff** | Handles conversion tracking, geographic hierarchies, financial reconciliation |
| **Principal** | Addresses multi-currency, cross-market analysis, search attribution |

### Related Lessons

- [Lesson 3](./03-identifying-the-right-business-processes.md) — Identifying business processes
- [Lesson 4](./04-determining-grain-of-fact-table.md) — Determining grain of fact tables
- [Lesson 14](./14-evolving-models-based-on-changing-requirements.md) — Evolving models
- [Lesson 21](./21-slowly-changing-dimensions-types-1-4.md) — SCD Types 1-4
- [Lesson 24](./24-fact-table-types.md) — Fact table types
- [Lesson 31](./31-advanced-dimension-design.md) — Advanced dimension design
