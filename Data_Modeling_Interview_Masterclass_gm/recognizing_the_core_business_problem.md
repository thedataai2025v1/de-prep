# Lesson 6: Recognizing the Core Business Problem & Grain Discovery

> **Principal Mentor Note:** Every company on earth runs on a handful of **core operational value loops** (e.g., *List $\rightarrow$ Search $\rightarrow$ Book $\rightarrow$ Stay $\rightarrow$ Review* at Airbnb). If you identify the exact business process at the heart of the prompt and lock down its **atomic grain**, the rest of the star schema practically designs itself.

---

## 1. Decomposing Any Business into Core Value Loops

When given an unfamiliar industry prompt (e.g., *Fleet Logistics*, *AdTech*, *Telemedicine*, *B2B SaaS*), immediately break the business down into **4 universal stages**:

```
[1. Supply / Catalog Onboarding] ──> [2. Discovery & Intent] ──> [3. Core Transaction / Fulfillment] ──> [4. Settlement & Retention]
```

### Pattern Recognition Across Domains

| Domain | Stage 1: Supply | Stage 2: Intent / Funnel | Stage 3: Core Transaction | Stage 4: Settlement / Lifecycle |
| :--- | :--- | :--- | :--- | :--- |
| **Ride-Sharing (Uber/Lyft)** | Driver shifts & online supply hours | Ride quote & trip request | Trip pickup $\rightarrow$ drop-off | Fare capture, surge split, driver payout |
| **E-Commerce (Amazon)** | Seller SKU listing & warehouse inventory | Search, product view, add-to-cart | Order checkout & line-item shipment | Payment settlement, returns, refunds |
| **Fintech / Payments (Stripe)** | Merchant onboarding & KYC | PaymentIntent creation & auth attempt | Capture, clearing & settlement | Interchange fees, payouts, chargebacks |
| **Video Streaming (Netflix)** | Content catalog & licensing rights | Impression, browse, play start | Playback heartbeat / watch session | Subscription billing, renewal, churn |
| **Cloud Infra (AWS)** | Region/AZ resource capacity | Instance provisioning / API request | Hourly/second resource metering | Invoice generation, reserved credits, billing |

---

## 2. The Golden Rule of Grain Discovery

Kimball's most sacred rule—and the #1 interview differentiator—is **defining the grain before choosing dimensions or facts**.

> **Definition of Grain:** What a **single physical row** in a fact table represents in the real world.

### How to Test If Your Grain Is Rock-Solid
Ask yourself: *"What is the natural primary key of this fact table?"*
* If you cannot name a clean, unambiguous natural key (e.g., `(order_id, line_item_number)` or `(user_id, session_id, event_seq)`), your grain is blurry.
* **Always aim for the finest atomic grain available** at ingestion. You can always aggregate atomic rows *upward* into daily or order-level summaries, but you can **never** disaggregate an order-level summary back down into individual product SKUs.

### Common Grain Traps & How Principal Engineers Solve Them

#### Trap 1: Header-Level vs. Line-Item-Level Metrics (E-Commerce)
* **Scenario:** An order (`order_101`) has 3 line items (`item_A: $50`, `item_B: $30`, `item_C: $20`), a `$10` shipping fee, and a `$15` order-level promo discount.
* **Wrong Approach:** Putting `order_shipping_fee = 10` and `order_discount = 15` on every row of `fct_order_items`. Any `SUM(order_shipping_fee)` grouped by `product_category` will triple-count shipping!
* **Principal Solutions (Pick One Explicitly):**
  1. **Proportional Allocation:** Allocate header fees/discounts to each line item deterministically (`item_price / total_order_item_price`), storing `allocated_shipping_usd` and `allocated_discount_usd` on `fct_order_items`, and document the allocation rule.
  2. **Dual-Grain Fact Tables:** Maintain `fct_orders` (grain: 1 row per order) for order-level fees/shipping/conversion, and `fct_order_items` (grain: 1 row per order line item) for product-level unit sales and gross merchandise value (GMV).

#### Trap 2: Mutable Lifecycle vs. Immutable Events (Order Fulfillment)
* **Scenario:** Stakeholders ask: *"How many orders were placed today?"* AND *"What is the average time from Order Placed $\rightarrow$ Packed $\rightarrow$ Shipped $\rightarrow$ Delivered?"*
* **Principal Solution:** Recognize these are **two different business processes**:
  * **Process A (Event Stream):** Every status change (`PLACED`, `PACKED`, `SHIPPED`, `DELIVERED`, `CANCELED`) is an immutable row in `fct_order_status_events` (grain: 1 row per order status transition).
  * **Process B (Lifecycle Funnel):** `fct_order_fulfillment` (grain: 1 row per `order_id`, Accumulating Snapshot) stores milestone timestamps (`placed_at`, `packed_at`, `shipped_at`, `delivered_at`) and pre-computed lag durations (`placed_to_shipped_minutes`), updated via `MERGE` as the order progresses.

#### Trap 3: Continuous Telemetry vs. Discrete Sessions (Streaming / IoT)
* **Scenario:** A video player emits a telemetry ping every 10 seconds (`playback_heartbeat`), plus discrete `play`, `pause`, `buffer`, and `stop` events.
* **Principal Solution:**
  * Raw heartbeats stay in Silver (`slv_playback_heartbeats`, grain: 1 row per 10s ping) for QoS/buffering engineering diagnostics.
  * Gold analytics uses **Sessionized Fact** (`fct_watch_sessions`, grain: 1 row per continuous viewing session `watch_session_id`), aggregating `total_watch_seconds`, `buffer_count`, and `max_bitrate`.

---

## 3. Real Interview Questions & Model Answers

### Q1: "We run a food delivery marketplace (like DoorDash). A customer places one order that gets split across two couriers because the first courier's bike broke down mid-delivery. What is the grain of your core delivery fact table?"

**Principal-Level Answer:**
> *"This is a classic grain fork between the **Customer Order** and the **Courier Dispatch Leg**. If we model only at the `order_id` grain, we lose courier-level metrics like acceptance rate, distance driven per courier, and split payout accounting. Conversely, if we model only at the `dispatch_leg_id` grain and attach the customer's `$25.00` food subtotal to both rows, we double-count GMV.*
>
> *Therefore, I separate the two core business processes into distinct fact tables:*
> 1. ***`fct_orders` (Grain: 1 row per customer `order_id`):** Captures customer-facing financials (`food_subtotal`, `delivery_fee`, `tip_amount`, `promo_discount`) and end-to-end order timestamps (`ordered_at`, `delivered_at`).*
> 2. ***`fct_delivery_legs` (Grain: 1 row per courier assignment `delivery_leg_id`, with `order_id` as a foreign/degenerate key):** Captures operational execution (`courier_sk`, `leg_sequence_num`, `leg_status` = `REASSIGNED` vs. `COMPLETED`, `leg_distance_meters`, `courier_payout_usd`).*
>
> *This preserves exact financial reconciliation on `fct_orders` while giving Operations full visibility into courier handoffs and reliability."*

---

## 4. Conversational & "Top-Up" Follow-Up Questions

1. **Top-Up 1: "When is it acceptable to store pre-aggregated grain in the primary fact table instead of atomic events?"**
   * **Expected Insight:** When the raw atomic stream is orders of magnitude larger than any analytical use case requires and lacks business entity context—for example, network router packets (100B/day) aggregated into **1-minute flow summaries** (`(src_ip, dst_ip, minute_bucket)`), or ad-server raw pixels rolled up into **1-minute campaign/placement aggregates** after bot filtering. Always keep raw logs in cold object storage (Bronze) for replay/audit.
2. **Top-Up 2: "How do you handle a 'Zero-Event' business question—like 'Which drivers were online and available in Zone A at 5 PM, even if they completed zero trips'?"**
   * **Expected Insight:** A transaction fact table (`fct_trips`) **cannot answer questions about events that didn't happen**. You must recognize a **Coverage / Availability business process** and model a **Periodic Snapshot Fact** (`fct_driver_supply_15min` or `fct_driver_daily_availability`) with one row per `(driver_sk, time_bucket, zone_sk)` recording `online_minutes`, `idle_minutes`, and `trips_completed`.

---

## 5. Principal-Level Signals vs. Red Flags

| Signal (Strong Hire / Principal) | Red Flag (Down-level / No Hire) |
| :--- | :--- |
| **Writes the grain statement in bold** at the top of every fact table definition. | Leaves grain implicit, forcing the interviewer to guess whether a row is an order, item, or event. |
| **Splits multi-process entities** (Order vs. Delivery Leg; Order vs. Fulfillment Lifecycle) cleanly. | Shoves 60 columns from 4 lifecycle stages into one bloated, half-NULL mega-table. |
| **Identifies "non-event" analytical questions** (inventory stockouts, idle driver supply, inactive users) and introduces snapshot facts. | Tries to answer driver utilization or churn using only a `fct_completed_trips` transaction table. |