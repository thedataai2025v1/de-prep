# Module 4: Dimension Design
## Lesson 4.1 — Slowly Changing Dimensions (Types 0–4)

---

### What are Slowly Changing Dimensions (SCDs)?

In dimensional modeling, **dimensions describe the "who, what, where, when, and why" of business facts**. But real-world data changes: customers move, employees change departments, products get repriced. **Slowly Changing Dimensions (SCDs)** are strategies for handling dimension attributes that change over time — and deciding how much of that history to preserve.

| Concept | Explanation |
|---|---|
| **Dimension change** | A descriptive attribute changes (e.g., customer's address) |
| **Fact change** | A measurable event is recorded (e.g., a sale amount) |
| **SCD scope** | Only applies to dimension attributes, not facts |

**Why SCDs are critical:** Financial reporting, regulatory compliance, trend analysis, and accurate cohort tracking all depend on knowing what a dimension looked like *at a specific point in time*. Choosing the wrong SCD strategy can make your data warehouse silently wrong.

> **Key insight for interviews:** Always ask *"Does the business need to know what this looked like at a prior date?"* before designing any dimension.

---

### Type 0 — No Change

**Strategy:** Keep the original value forever. The attribute is loaded once and never updated.

| Attribute | Behavior |
|---|---|
| Original value | Preserved indefinitely |
| New value | Ignored — no update ever |
| SQL pattern | `INSERT` only; no `UPDATE` |

**When to use:** Immutable attributes such as birth date, Social Security Number, employee ID, or date of incorporation.

**Trade-offs:**
- ✅ Simplest strategy; zero maintenance overhead
- ✅ Guaranteed referential integrity
- ❌ Useless for any attribute that legitimately changes
- ❌ Often bundled with other SCD types on the same table

**Example:**
```sql
-- Employee dimension, Type 0 columns mixed with others
INSERT INTO dim_employee (emp_id, emp_name, hire_date, ssn)
VALUES (101, 'Jane Smith', '2020-01-15', '123-45-6789');

-- Even if SSN is corrected later, we NEVER update it (Type 0)
-- (In reality, SSNs don't change — making Type 0 the correct choice)
```

---

### Type 1 — Overwrite

**Strategy:** Simply replace the old value with the new one. No history is kept.

| Attribute | Behavior |
|---|---|
| Old value | Permanently lost |
| New value | Replaces old in-place |
| SQL pattern | Standard `UPDATE` statement |

**When to use:** Corrections of erroneous data, or attributes where history genuinely doesn't matter (e.g., a customer's phone number).

**Example:**
```sql
-- Customer address updated — old address disappears
UPDATE dim_customer
SET street_address = '456 New Avenue', city = 'Chicago'
WHERE customer_key = 1001;
```

**Trade-offs:**
- ✅ Fast, simple, minimal storage
- ✅ Good for corrections and non-critical attributes
- ❌ No historical trace — you cannot answer "Where did this customer live in 2022?"
- ❌ Can cause misleading aggregations if historical facts reference the overwritten dimension

---

### Type 2 — Full History ⭐ (Most Important for Interviews)

**Strategy:** Add a **new row** for each change. Mark the old row as inactive using effective/expiry dates and a current flag.

| Column | Purpose |
|---|---|
| `dim_key` (surrogate key) | Unique ID for each version (auto-increment / sequence) |
| `natural_key` | Business identifier (e.g., customer number) — stays the same across versions |
| `attribute` | The changing value (e.g., state, address) |
| `effective_date` | When this version became valid |
| `expiry_date` | When it was superseded (NULL = still current) |
| `is_current` | Flag (1/0 or TRUE/FALSE) for quick current-row lookup |

**SQL Pattern — Detecting and Loading a Change:**
```sql
-- Step 1: Mark the old row as expired
UPDATE dim_customer
SET expiry_date = '2024-06-01',
    is_current = 0
WHERE customer_natural_key = 1001
  AND is_current = 1;

-- Step 2: Insert the new version
INSERT INTO dim_customer (
    dim_key, customer_natural_key, customer_name, state,
    effective_date, expiry_date, is_current
)
VALUES (
    5002, 1001, 'Jane Smith', 'Illinois',
    '2024-06-01', NULL, 1
);
```

**Querying a point-in-time snapshot:**
```sql
-- "What state was customer 1001 in on 2024-03-15?"
SELECT customer_name, state
FROM dim_customer
WHERE customer_natural_key = 1001
  AND '2024-03-15' BETWEEN effective_date AND COALESCE(expiry_date, '9999-12-31');
```

**When to use:** Regulatory requirements (SOX, HIPAA), trend analysis, any scenario where you must reconstruct the dimension at any prior date.

**Trade-offs:**
- ✅ Complete, accurate history — can reconstruct any point in time
- ✅ Industry standard — every senior data engineer expects you to know this
- ❌ Largest table growth (1+ rows per change per dimension)
- ❌ More complex joins — facts must join on effective_date ranges
- ❌ Requires robust ETL logic to detect changes

**Surrogate Key vs Natural Key — The Interview Cornerstone:**
```
Natural Key (customer_natural_key = 1001)  → Identifies the entity
   ├─ Row 1: dim_key=5001, state='Texas',    effective=2020-01-01, is_current=0
   ├─ Row 2: dim_key=5002, state='California',effective=2022-06-15, is_current=0
   └─ Row 3: dim_key=5003, state='Illinois', effective=2024-06-01, is_current=1   ← Current

Fact tables join on dim_key (surrogate), NOT natural key.
This ensures each fact is linked to the EXACT dimension version that existed at transaction time.
```

---

### Type 3 — Limited History

**Strategy:** Add extra columns to store a limited number of prior values (typically just the previous one).

| Column | Purpose |
|---|---|
| `current_department` | Present value |
| `previous_department` | Value at last change |
| `change_date` | When the last change occurred |

**When to use:** The business only cares about "What changed from the last period?" — not full history.

**Example:**
```sql
UPDATE dim_employee
SET previous_department = current_department,
    current_department = 'Engineering',
    change_date = '2024-06-01'
WHERE emp_key = 101;
```

**Trade-offs:**
- ✅ Simple, only one row per entity (no row multiplication)
- ✅ Supports "before/after" analysis
- ❌ Only one prior value — you lose all history before that
- ❌ Adding more columns (prev_prev_dept, etc.) is fragile and not scalable

---

### Type 4 — Historical Table (Helper Table)

**Strategy:** Keep the main dimension table current-only (like Type 1), and maintain a **separate history table** that records every change.

```
dim_employee (current only)          emp_history (append-only log)
┌──────────────────────┐             ┌──────────────────────────────┐
│ emp_key │ dept │ ... │             │ hist_key │ emp_key │ old_dept
│    101  │ Eng │ ... │             │    1001  │   101   │  Sales
└──────────────────────┘             │    1002  │   101   │  Marketing
                                     └──────────────────────────────┘
```

**When to use:** History is infrequently queried; you want fast current-dimension lookups for everyday queries.

**SQL Pattern:**
```sql
-- Before updating, log the old value
INSERT INTO emp_history (emp_key, previous_dept, change_date)
SELECT emp_key, department, '2024-06-01'
FROM dim_employee
WHERE emp_key = 101;

-- Then update the current dimension (Type 1 behavior)
UPDATE dim_employee
SET department = 'Engineering'
WHERE emp_key = 101;
```

**Trade-offs:**
- ✅ Main dimension stays lean and fast
- ✅ Complete history available in the separate table
- ❌ Two tables to manage and query
- ❌ Joins across both tables are more complex

---

### Type 5 — Additive Changes

**Strategy:** Track cumulative/additive changes (e.g., running totals, credits added to a balance) alongside regular slowly changing attributes. Combines Type 2 with accumulating fact tracking.

| Column | Purpose |
|---|---|
| `dim_key` | Surrogate key |
| `emp_key` | Natural key |
| `current_balance` | Additive measure (grows over time) |
| `department` | Regular SCD attribute |
| `effective_date`, `expiry_date`, `is_current` | Type 2 controls |

**When to use:** When a dimension needs to track both historical attribute changes (Type 2) and cumulative values that only go up (e.g., cumulative discounts, reward points, lifetime spend).

**Example:**
```sql
-- Employee gets a raise AND a department change
UPDATE dim_employee
SET expiry_date = CURRENT_DATE, is_current = 0
WHERE emp_key = 101 AND is_current = 1;

INSERT INTO dim_employee (dim_key, emp_key, department, current_balance, effective_date, expiry_date, is_current)
VALUES ((SELECT MAX(dim_key) + 1 FROM dim_employee), 101, 'Engineering', 15000, CURRENT_DATE, NULL, 1);
-- current_balance went from 10000 → 15000 (additive), department also changed (Type 2)
```

**Trade-offs:**
- ✅ Captures both additive and non-additive changes in one table
- ❌ Blurs the line between dimension and fact; some purists prefer separating them
- ❌ Harder to reason about when the additive and SCD changes happen independently

---

### Type 6 — Hybrid SCD (Type 1 + Type 2 + Type 3)

**Strategy:** Store current values in the row's main columns (Type 1 behavior), keep full history through row versioning (Type 2), and also store the previous value in extra columns (Type 3). You get all three behaviors in one table.

```
dim_employee (Type 6 — Hybrid)
┌──────────────────────────────────────────────────────────────────────┐
│ dim_key │ emp_key │ dept │ prev_dept │ current_dept │ eff_date │ exp│
│   1001  │   101   │ Sales│    NULL   │   Sales      │ 2024-01-01│NULL│
│   1002  │   101   │ Eng  │   Sales   │     Eng      │ 2024-06-01│NULL│
└──────────────────────────────────────────────────────────────────────┘
```

**When to use:** When you need fast current-value lookups (read the column directly), a quick before/after comparison (prev\_dept vs current\_dept), AND full historical traceability (query by effective/expiry dates). Common in financial and healthcare domains where regulators demand all three views.

**Example:**
```sql
-- Expire the current row, insert new row with old value captured
UPDATE dim_employee
SET expiry_date = CURRENT_DATE, is_current = 0
WHERE emp_key = 101 AND is_current = 1;

INSERT INTO dim_employee (dim_key, emp_key, dept, prev_dept, current_dept, effective_date, expiry_date, is_current)
SELECT (SELECT MAX(dim_key) + 1 FROM dim_employee),
       e.emp_key, 'Engineering', e.dept, 'Engineering',
       CURRENT_DATE, NULL, 1
FROM dim_employee e
WHERE e.emp_key = 101 AND e.is_current = 1;
```

**Trade-offs:**
- ✅ Best of all three SCD types in one table
- ✅ Supports any analytical query without extra joins
- ❌ More columns to maintain, higher write complexity
- ❌ ETL logic is the most complex of all SCD types

---

### Type 7 — Double SCD (Type 2 + Type 3)

**Strategy:** Maintain Type 2 full history AND Type 3 previous-value columns simultaneously. Similar to Type 6 but simpler — no Type 1 component. The main columns change with each new row (Type 2), and a separate column always holds the prior value (Type 3).

```
dim_employee (Type 7 — Double SCD)
┌────────────────────────────────────────────────────────────────────┐
│ dim_key │ emp_key │ dept │ prev_dept │ eff_date │ exp_date │ cur │
│   1001  │   101   │Sales│   NULL    │2024-01-01│   NULL   │  1  │
│   1002  │   101   │ Eng │  Sales    │2024-06-01│   NULL   │  1  │
│   1003  │   101   │ Fin │   Eng     │2024-12-01│   NULL   │  1  │
└────────────────────────────────────────────────────────────────────┘
```

**When to use:** When you need both a full audit trail (query by date range) and a quick "what was the previous value?" lookup on any row. Useful in compliance-heavy environments.

**Trade-offs:**
- ✅ Full history + prior-value lookup in the same row
- ❌ Extra column storage and write complexity
- ❌ Rarely needed — most teams can satisfy requirements with Type 2 alone

---

### How to Choose the Right SCD Type — Decision Framework

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SCD Type Decision Framework                         │
├─────────────────────────────────────────────────────────────────────────┤
│ QUESTION  →  ANSWER              →  SCD TYPE                            │
├─────────────────────────────────────────────────────────────────────────┤
│ "Does this attribute ever change?"                                     │
│   No  →  It's immutable                      →  Type 0                  │
│   Yes →  Continue to next question           →  ↓                       │
├─────────────────────────────────────────────────────────────────────────┤
│ "Does the business need to know what it was before?"                   │
│   No  →  Just keep the latest                →  Type 1 (Overwrite)      │
│   Yes →  Continue to next question           →  ↓                       │
├─────────────────────────────────────────────────────────────────────────┤
│ "Do they need the FULL history, or just the last change?"              │
│   Last change only  →  Limited history        →  Type 3                 │
│   Full history      →  Continue to next       →  ↓                     │
├─────────────────────────────────────────────────────────────────────────┤
│ "How often is history queried?"                                         │
│   Frequently  →  Inline with dimension      →  Type 2 (Full History)    │
│   Rarely      →  Separate history table      →  Type 4                  │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key Questions to Ask (and be ready to answer in an interview):**

| Question | Why It Matters |
|---|---|
| Do you need full history, or just the last change? | Determines Type 2 vs Type 3 |
| How far back do you need to track? | Affects retention policy, partitioning |
| How often is historical data queried? | Determines Type 2 (inline) vs Type 4 (separate) |
| Is there a regulatory or audit requirement? | Almost always mandates Type 2 |
| What is the expected change frequency? | High frequency → Type 2 table growth concern |

---

### SCDs in the Interview

#### How to Explain SCD Types Concisely (60-Second Pitch)

> *"Slowly Changing Dimensions are strategies for handling changes in dimension data over time. **Type 0** keeps the original value forever for immutable attributes. **Type 1** overwrites without history. **Type 2** creates a new row for every change, giving full history — it's the most common and the one I default to unless there's a specific reason not to. **Type 3** stores a limited number of prior values in extra columns. **Type 4** keeps the main dimension current and stores history in a separate table. My choice depends on whether the business needs history, how far back, and how often it's queried."*

#### Common Interview Question

> **Interviewer:** "A customer's state changes from Texas to California. How would you track this?"

**Model Answer:**
> *"I'd default to **Type 2 SCD**. I'd mark the existing row for that customer as expired with an expiry date and set `is_current = 0`, then insert a new row with the new state, today's date as `effective_date`, `expiry_date = NULL`, and `is_current = 1`. This gives us a complete audit trail — if the business later needs to know the customer's state on any historical date, we can query the effective/expiry date range. I'd only consider Type 1 if the business explicitly said history didn't matter, or Type 3 if they only needed the previous state."*

#### Follow-Up: Show the SQL

```sql
-- The interviewer will often ask you to write this on a whiteboard
UPDATE dim_customer
SET expiry_date = CURRENT_DATE,
    is_current = 0
WHERE customer_natural_key = 1001
  AND is_current = 1;

INSERT INTO dim_customer (
    dim_key, customer_natural_key, customer_name, state,
    effective_date, expiry_date, is_current
)
VALUES (
    (SELECT MAX(dim_key) + 1 FROM dim_customer),
    1001, 'Jane Smith', 'California',
    CURRENT_DATE, NULL, 1
);
```

---

### Interviewer Questions You'll Face

**Q1: What's the difference between a surrogate key and a natural key in a Type 2 SCD?**
> **A:** The natural key (e.g., customer number) identifies the business entity and stays the same across all version rows. The surrogate key (e.g., an auto-increment `dim_key`) uniquely identifies each version/row. Fact tables join on the surrogate key so each fact is tied to the exact dimension version that existed at transaction time. Without a surrogate key, you cannot distinguish which version of the dimension a fact references.

**Q2: How would you handle a dimension where multiple attributes change at different times?**
> **A:** In Type 2, a change to *any* tracked attribute triggers a new row. If attributes change independently and need separate history tracking, you can create separate slowly changing dimensions — e.g., a `dim_customer_state` with its own SCD logic, and a `dim_customer_address` with its own — and join both to the fact table. This is called a **conformal dimension** or **split dimension** strategy.

**Q3: What happens if a row needs to be corrected (not changed) in a Type 2 dimension?**
> **A:** If it's a true correction (e.g., a typo), we have two options: (1) Update the row in-place if no facts have yet referenced it, or (2) If facts already reference it, we must close the incorrect row, insert a corrected version back-dated to the original effective date, and re-point the affected facts to the new surrogate key. Option 2 is safer for audit integrity.

**Q4: How do you efficiently detect changes in a source system for Type 2?**
> **A:** Compare the current source data against the latest (`is_current = 1`) rows in the dimension table. Common approaches: (1) Hash the relevant columns and compare checksums, (2) Compare each attribute individually, (3) Use CDC (Change Data Capture) tools like Debezium, or (4) Check `updated_at` timestamps in the source system. The hash/checksum approach is the most common in production.

---

### Top-Up Questions (Senior-Level Probes)

**Q1: How does a Type 2 SCD interact with fact table grain?**
> **A:** Every fact row must store the surrogate key of the dimension version valid at transaction time, not the latest. At ETL load time, you look up the dimension row where `effective_date ≤ transaction_date AND (expiry_date IS NULL OR expiry_date > transaction_date)`. If you always join on `is_current = 1`, you get current snapshots — not point-in-time accuracy.

**Q2: What's the performance impact of Type 2 on large fact joins?**
> **A:** Type 2 inflates dimension row count, which can slow joins. Mitigations: (1) Partition the dimension table by `effective_date` or `is_current`, (2) Use a Type 4 pattern if history is rarely queried, (3) Create a materialized view or denormalized snapshot for current-only queries, (4) In cloud warehouses (Snowflake, BigQuery), the row count impact is less severe than in on-prem systems due to columnar storage and pruning.

**Q3: How would you implement a Type 2 SCD in a streaming/real-time pipeline (e.g., Kafka + Spark)?**
> **A:** Use a watermark-based approach: maintain the current state in a stateful store (e.g., Redis, Kafka Streams state store), detect changes against it, write expired rows and new rows to the dimension, and publish the changes. In Spark Structured Streaming, use `mapGroupsWithState` to track the latest version per natural key, and write both the UPDATE and INSERT in a single transaction.

**Q4: When would you combine SCD types on the same dimension table?**
> **A:** Absolutely — most real dimensions use a mix. For example, a customer dimension might use: Type 0 for customer_id, Type 1 for phone_number (no history needed), Type 2 for state (regulatory requirement), and Type 3 for department (only need the previous value). Bill Inmon and Ralph Kimball both advocate this mixed-SCD approach.

---

### Practice Exercise

**Scenario:** A retail company's `dim_product` table has the following attributes:

| Attribute | Change Frequency | Business Requirement |
|---|---|---|
| `product_id` | Never changes | Unique identifier |
| `product_name` | Rarely (corrections only) | No history needed |
| `product_category` | Every 6 months | Must track full history for trend analysis |
| `product_price` | Weekly | Need current and previous price |
| `product_launch_date` | Never changes | Immutable |

**Task 1:** Assign the correct SCD type to each attribute and justify your choice.

**Task 2:** Write the DDL for `dim_product` including all necessary columns for your chosen SCD types.

**Task 3:** Write the ETL logic (pseudocode or SQL) to process a change where product `P-200` moves from category "Electronics" to "Home Goods" and its price changes from $29.99 to $24.99 on `2024-07-01`.

**Task 4:** Write a query to show all categories product `P-200` has been in, in chronological order.

---

> **Final Interview Tip:** When asked about SCDs, lead with Type 2. It's the most defensible default. Only propose alternatives when you can articulate a clear business reason. Senior interviewers want to hear you weigh trade-offs, not just recite definitions.

---

### Related Lessons

- [Degenerate and Junk Dimensions](./22-degenerate-and-junk-dimensions.md)
- [Dimension Design Practice](./23-dimension-design-practice.md)
- [Fact Table Types](./24-fact-table-types.md)
- [Data Retention Policies and Historical Data Management](./11-data-retention-policies-and-historical-data-management.md)
