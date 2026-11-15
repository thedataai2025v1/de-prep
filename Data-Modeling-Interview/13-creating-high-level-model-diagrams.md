# Module 3: High-Level Model Diagrams
## Lesson 3.1 — Creating High-Level Model Diagrams

> *"A sketch on a whiteboard convinces an interviewer faster than a DDL script on a screen."* Before you write a single `CREATE TABLE`, you need to communicate your design at a level everyone can understand — business, engineering, and leadership. This lesson teaches you how to build high-level model diagrams that signal seniority, clarity, and interview readiness.

---

### 1. Why High-Level Diagrams Come Before Detailed Schema Design

In an interview, you'll rarely get time to specify every column and data type. **High-level diagrams are your first impression.** They demonstrate:

| What It Shows | Why It Matters |
|---|---|
| **Structural thinking** | You can decompose a business problem into facts and dimensions |
| **Volume awareness** | You estimate scale and flag storage/query risks early |
| **Communication skill** | Non-technical stakeholders can follow your reasoning |
| **Interview pacing** | A 5-minute diagram buys you 15 minutes to flesh out details |

Think of the process as three layers — just like the data modeling types you learned in Lesson 1.4:

```
Layer 1: High-Level Diagram     → "Here's the shape of the solution."
Layer 2: Logical Model          → "Here are every column, key, and relationship."
Layer 3: Physical DDL           → "Here's the exact SQL for our platform."
```

**The interview tests Layer 1 first.** Get the shape right, and everything else follows.

---

### 2. How to Sketch a Model During an Interview

Whether you're using a whiteboard, Miro, Lucidchart, or a shared Google Doc, follow these principles:

| Technique | Advice |
|---|---|
| **Start big** | Draw one large box for the fact table at the center first |
| **Add dimensions radially** | Place dimension boxes around the fact like a star |
| **Keep columns abbreviated** | List only key columns — full lists come later |
| **Label everything** | PK, FK, grain, and row counts — never assume |
| **Talk as you draw** | Narrate your reasoning: *"I'm starting with fact_orders because the business process is order fulfillment..."* |
| **Use consistent notation** | Stick to the conventions below so the interviewer can read quickly |

---

### 3. Elements to Include in Every High-Level Diagram

Every diagram should contain **five elements**:

| Element | Description | Example |
|---|---|---|
| **Fact tables** | The central measurement tables, prefixed `fact_` | `fact_orders` |
| **Dimension tables** | Context tables, prefixed `dim_` | `dim_date`, `dim_customer` |
| **Relationships** | Lines connecting FK → PK, labeled with cardinality | 1 ──< many |
| **Grain** | One-line annotation under each table explaining what a row represents | "One row = one order line item" |
| **Estimated row counts** | Scale estimates, often as ranges | "~2M rows/day, ~730M/year" |

---

### 4. Step-by-Step Approach to Building a Diagram

Follow these five steps in order. This is the exact sequence you should narrate in an interview.

#### Step 1 — Start with the Business Process → Identify the Fact Table

> *"The business process is online order fulfillment. Each time a customer completes an order, that's the atomic event I measure."*

The fact table captures **what happened, when, and how much**. Its grain is the most fundamental decision.

```
┌──────────────────────────────────────────┐
│              fact_orders                 │
│  grain: one row = one completed order   │
│                                          │
│  Columns:                                │
│  ─ order_id (PK)                         │
│    date_id (FK → dim_date)               │
│    customer_id (FK → dim_customer)       │
│    product_id (FK → dim_product)         │
│    location_id (FK → dim_location)       │
│    order_amount                          │
│    quantity                              │
│    discount_amount                       │
│    net_amount (derived)                  │
│                                          │
│  est. volume: ~2M rows/day              │
└──────────────────────────────────────────┘
```

#### Step 2 — Add Dimensions That Describe the Fact

Each dimension answers "who, what, where, when" about the fact.

```
  dim_date          →  WHEN the order happened
  dim_customer      →  WHO placed the order
  dim_product       →  WHAT was ordered
  dim_location      →  WHERE the order shipped to
```

#### Step 3 — Show Relationships with Foreign Keys

Draw lines from each FK in the fact to the PK in the dimension. Label cardinality.

```
  dim_date        ──┐
  dim_customer      ├───────<  fact_orders  (many-to-one in each direction)
  dim_product       ├───────<
  dim_location      ──┘
```

#### Step 4 — Note the Grain of Each Table

The grain tells the interviewer you understand **what level of detail** each table holds:

| Table | Grain Annotation |
|---|---|
| `fact_orders` | One row per completed order |
| `dim_date` | One row per calendar day |
| `dim_customer` | One row per unique customer (current attributes) |
| `dim_product` | One row per distinct product/SKU |
| `dim_location` | One row per geographic region/city |

#### Step 5 — Add Estimated Row Counts

Volume estimates show you think about scale. Even rough numbers are better than none:

| Table | Est. Rows | Reasoning |
|---|---|---|
| `fact_orders` | 2M/day × 365 ≈ 730M/year | Based on business stating 2M daily orders |
| `dim_date` | ~10,000 | 30 years of daily granularity |
| `dim_customer` | ~500K | Active customer base |
| `dim_product` | ~50K | SKU catalog |
| `dim_location` | ~500 | Cities and regions served |

---

### 5. Diagram Notation Conventions

Use these conventions **consistently** so interviewers can read your diagram without confusion.

| Convention | Symbol | Example |
|---|---|---|
| **Table prefix** | `fact_` or `dim_` | `fact_orders`, `dim_customer` |
| **Primary key** | `PK` or `(PK)` after column | `order_id (PK)` |
| **Foreign key** | `FK → table` after column | `customer_id (FK → dim_customer)` |
| **Relationship line** | `1 ──< many` (one-to-many) | Draw a line from dim PK to fact FK |
| **Grain** | Text line under table | `-- grain: one row per order` |
| **Volume** | `~N rows` or `~N rows/day` | `~2M rows/day` |
| **Derived column** | `(derived)` suffix | `net_amount (derived)` |

**Full notation example:**

```
  order_id (PK)              customer_id (PK)
  date_id  (FK→dim_date)     customer_name
  customer_id (FK→dim_cust)  email
  product_id  (FK→dim_prod)  city (FK→dim_location)
  order_amount               signup_date
  quantity                   -- grain: one row per customer
  discount_amount
  net_amount (derived)       -- ~500K rows
  -- grain: one row per order
  -- ~2M rows/day
```

---

### 6. Worked Example: E-Commerce Star Schema

Below is a complete high-level diagram for an e-commerce platform. This is the kind of diagram you'd draw on a whiteboard or shared canvas during an interview.

```
┌───────────────────┐        ┌───────────────────┐
│   dim_date        │        │   dim_customer    │
│───────────────────│        │───────────────────│
│ date_id  (PK)     │        │ customer_id (PK)  │
│ date              │        │ customer_name     │
│ year, month, week │        │ email             │
│ quarter           │        │ city_id (FK)      │
│ is_holiday        │        │ signup_date       │
│                   │        │ -- grain: 1/cust   │
│ -- grain: 1/day   │        │ -- ~500K rows     │
└─────────┬─────────┘        └────────┬──────────┘
          │ FK                         │ FK
          │                            │
          │        ┌───────────────────┤
          │        │                   │
          │        │                   │
┌─────────┴────────┐        ┌──────────┴──────────┐
│   dim_product    │        │   dim_location      │
│──────────────────│        │─────────────────────│
│ product_id (PK)  │        │ location_id  (PK)   │
│ product_name     │        │ city              │
│ category         │        │ state             │
│ subcategory      │        │ country           │
│ list_price       │        │ timezone          │
│                   │        │                   │
│ -- grain: 1/SKU  │        │ -- grain: 1/city  │
│ -- ~50K rows     │        │ -- ~500 rows      │
└────────┬─────────┘        └────────────────────┘
         │ FK                        │ FK
         │                           │
         │      ┌────────────────────┘
         │      │
         │      │
   ┌─────┴──────┴──────────────────────────────┐
   │              fact_orders                  │
   │───────────────────────────────────────────│
   │ order_id        (PK)                      │
   │ date_id         (FK → dim_date)           │
   │ customer_id     (FK → dim_customer)       │
   │ product_id      (FK → dim_product)        │
   │ location_id     (FK → dim_location)       │
   │ order_amount    (decimal)                 │
   │ quantity        (int)                     │
   │ discount_amount (decimal)                 │
   │ net_amount      (derived)                 │
   │                                           │
   │ -- grain: one row = one order line item   │
   │ -- ~2M rows/day, ~730M/year               │
   └───────────────────────────────────────────┘
```

**Reading the diagram left to right:**

1. **Center** — `fact_orders` is the heart of the star schema. Every query ultimately aggregates from this table.
2. **Surrounding dimensions** — Each dimension can be joined independently, enabling flexible filter-and-aggregate queries.
3. **Cardinality** — Every relationship is many-to-one: many orders belong to one date, one customer, one product, one location.
4. **Grain** — "One row = one order line item" means if a customer buys 3 products in one order, there are 3 rows. This is a **transaction fact**.
5. **Scale** — The row counts reveal that `fact_orders` is the dominant storage consumer (730M/year) while dimensions are small lookups.

---

### 7. Common Mistakes (And How to Avoid Them)

| Mistake | Why It Hurts | How to Fix |
|---|---|---|
| **Overcomplicating the diagram** | Drawing 15+ tables with every column confuses the interviewer | Start with 1 fact + 4–6 dimensions. Add more only if asked. |
| **Forgetting grain** | Without grain, the interviewer can't tell if you understand what a row means | Always annotate grain under every fact table. |
| **Missing key dimensions** | Skipping `dim_date` or `dim_customer` signals inexperience | Date and customer are almost always essential — include them by default. |
| **No volume estimates** | Suggests you haven't thought about scale | Give rough orders-of-magnitude numbers; they show architectural thinking. |
| **Confusing FK direction** | Drawing a line from the fact to the dimension without labeling FK/PK | Always label: `date_id (FK → dim_date)` points *from* fact *to* dimension PK. |
| **Not naming your diagram** | A floating diagram without context | Title it: "Star Schema — E-Commerce Order Fulfillment" and state the business process. |

---

### Interviewer Questions You'll Face

| # | Question | What They're Testing |
|---|----------|---------------------|
| 1 | *"Why did you put the grain at 'one row per order line item' instead of 'one row per order'?"* | Whether you understand that a single order with multiple products requires multiple fact rows — and the trade-off between grain and query simplicity. |
| 2 | *"How would this diagram change if we needed to track order status changes over time (placed → shipped → delivered)?"* | Recognition that a transaction fact alone can't track state transitions — you'd need an **accumulating snapshot fact** with multiple timestamp columns. |
| 3 | *"Where would you put product category? Inside `dim_product` or as a separate dimension?"* | Understanding of star vs. snowflake: star schema keeps category denormalized in `dim_product` for query simplicity. |
| 4 | *"If we get 10x more volume, which table becomes the bottleneck and what would you change?"* | Scale reasoning: `fact_orders` grows linearly with volume; consider partitioning by `date_id`, clustering by high-cardinality columns, or pre-aggregating. |

---

### Questions YOU Should Ask the Interviewer

Before you start drawing, ask these to scope the problem:

| # | Question | Why You're Asking |
|---|----------|------------------|
| 1 | *"What's the primary business process we're modeling?"* | Ensures you identify the correct fact table grain. |
| 2 | *"What are the key metrics the business needs to report on?"* | Reveals which columns must be in the fact table. |
| 3 | *"What dimensions do analysts slice and dice by most often?"* | Prioritizes which dimensions to include first. |
| 4 | *"Are there any data volume or latency constraints I should plan for?"* | Signals seniority — you're thinking about production realities. |
| 5 | *"Do you prefer a star schema, or have you committed to a specific architecture (e.g., OneBigTable, data vault)?"* | Aligns with the company's existing conventions. |

---

### Top-Up Questions (Senior-Level Probes)

| # | Question | What It Demonstrates |
|---|----------|---------------------|
| 1 | *"How would you handle slowly changing dimensions — for example, if a customer's city changes?"* | Knowledge of SCD types: Type 1 (overwrite), Type 2 (add new row with valid dates), Type 3 (add new column). For a dimension like `dim_customer` where location matters historically, SCD Type 2 is the standard choice. |
| 2 | *"What if two different business teams need to query the same dimensions but with different attributes?"* | Understanding of **conformed dimensions** — shared dimensions that can be joined to multiple fact tables, enabling enterprise-level reporting. |
| 3 | *"How would you extend this diagram to support a second fact table, like returns or refunds?"* | Ability to design a **bus matrix**: `dim_date`, `dim_customer`, and `dim_product` would be conformed dimensions shared between `fact_orders` and `fact_returns`. |
| 4 | *"Where would you draw the line between warehouse modeling and source system modeling?"* | Architectural maturity: the warehouse model should be **business-oriented**, not a mirror of the OLTP schema. Normalize in source, denormalize in warehouse for query performance. |

---

### Practice Exercise

**Scenario:** You're interviewing for a Data Engineer role at *StreamLine TV*, a video streaming service. The interviewer says:

> *"We have 50 million monthly active users who watch content from our library. We want to build a data warehouse to understand viewing behavior, content popularity, and subscriber churn. Sketch a high-level model diagram."*

**Your task:**

1. **Ask 3–5 scoping questions** before you start drawing (use the table above for inspiration).
2. **Sketch the diagram** on paper or in a tool, including:
   - At least one fact table with grain annotation
   - 3–4 dimension tables
   - Foreign key relationships with cardinality
   - Estimated row counts (make reasonable assumptions and state them)
3. **Explain your design decisions** out loud as you draw — what business process are you capturing, why did you choose this grain, and what metrics does this support?
4. **Be prepared to extend it** — the interviewer will likely ask: *"Now add a fact table for user subscriptions (new, renewal, cancellation). How does the diagram change?"*

<details>
<summary>💡 Click to reveal a hint</summary>

Think about these streaming-specific design decisions:
- **Fact grain**: One row per "viewing session" or one row per minute watched? Each has trade-offs.
- **Key dimensions**: `dim_content` (shows, movies, episodes), `dim_user` (subscriber info), `dim_date` (viewing date), `dim_device` (phone, TV, tablet).
- **Metrics**: watch time, completion rate, content views, bounce rate, subscriber LTV.
- **Second fact table**: `fact_subscriptions` (grain: one row per subscription event) shares `dim_user` and `dim_date` with `fact_viewing` — these are your conformed dimensions.

</details>

<details>
<summary>💡 Click to reveal a model answer outline</summary>

**Two-fact star schema:**

```
fact_viewing (grain: one row per viewing session)
  ├── session_id (PK)
  ├── date_id    (FK → dim_date)
  ├── user_id    (FK → dim_user)
  ├── content_id (FK → dim_content)
  ├── device_id  (FK → dim_device)
  ├── watch_duration_seconds
  ├── completion_pct
  └── ~500M sessions/month

fact_subscriptions (grain: one row per subscription event)
  ├── event_id   (PK)
  ├── date_id    (FK → dim_date)
  ├── user_id    (FK → dim_user)
  ├── event_type (new/renewal/cancellation)
  ├── plan_type
  └── ~2M events/month

Conformed dimensions (shared between both facts):
  dim_date, dim_user, dim_content, dim_device
```

</details>

---

> **Key takeaway from this lesson:** A high-level model diagram is your **first proof of competence** in a data modeling interview. The best diagrams are simple enough to draw in 5 minutes, clear enough for a non-technical stakeholder to read, and detailed enough (grain, FKs, volume) to prove you think like a senior engineer. Master this step, and the rest of the interview flows naturally.

---

### Related Lessons

- [Introduction to Data Modeling Questions](./01-introduction-to-data-modeling-questions.md)
- [How to Answer Data Modeling Questions](./02-how-to-answer-data-modeling-questions.md)
- [Data Modeling Fundamentals](./04-data-modeling-fundamentals.md)
- [Evolving Models Based on Changing Requirements](./14-evolving-models-based-on-changing-requirements.md)
