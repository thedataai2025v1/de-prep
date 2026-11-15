# Module 3: High-Level Model Diagrams — Lesson 3.7: Practice: Financial Transactions

> **Goal:** Given an open-ended interview prompt, gather requirements, design a high-level dimensional model, and defend every design choice. This lesson walks you through a complete example: a financial-services data warehouse.

---

## 1. Interview Prompt

> *"Design a data warehouse for a financial services platform — a bank that processes transactions across checking, savings, credit cards, and wire transfers."*

No schema is provided. You must **ask questions first**, then model.

---

## 2. Requirements Gathering

### 2a. Elicitation Checklist

Use this structure so nothing is missed:

| Category | What to Ask | Example Answer |
|----------|-------------|----------------|
| **Business goals** | What do executives need to decide? | Track transactions, detect fraud, manage risk, regulatory reporting |
| **Key metrics** | What KPIs drive dashboards? | Transaction volume, avg. value, fraud rate, risk exposure, daily balance changes |
| **Volume & scale** | Daily inserts, total entities | 50M txn/day, 50M accounts, 100K merchants |
| **Latency** | How fast must insights arrive? | Real-time for fraud; daily for regulatory reports |
| **Retention** | How long must data survive? | 7+ years (financial compliance); never delete |
| **Geo / currency** | One region or multi-national? | Multi-currency; FX conversion needed |

### 2b. Interviewer Questions You'll Face

1. *"Walk me through your model from scratch — start with the business requirements."*
2. *"Where do you draw the line between what goes in the data warehouse versus what stays in OLTP?"*
3. *"How would your design change if we had to support 20 currencies instead of one?"*
4. *"A regulator asks for every transaction over $10,000 in the last 5 years — can you answer that in seconds?"*

---

## 3. High-Level Model Diagram

```
                        ┌──────────────────┐
                        │   dim_date       │
                        │  (date_key,      │
                        │   date, quarter, │
                        │   fiscal_year)   │
                        └──────┬───────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
          ┌─────────┴──────┐    ┌────────┴──────────┐
          │ fact_transactions│    │ fact_account_      │
          │                  │    │ balances (SCD)    │
          │ grain: 1 row    │    │ grain: 1 row /    │
          │  per txn        │    │  account / day    │
          └───┬───┬───┬─────┘    └────────┬──────────┘
          ┌───┴───┴───┴─────┐             │
          │                  │             │
     ┌────┴────┐    ┌───────┴──────┐ ┌────┴────┐
     │dim_acct │    │  dim_customer │ │ dim_fx  │
     └───┬─────┘    └──────┬───────┘ └─────────┘
         │                 │
         │           ┌─────┴──────┐
         │           │ dim_channel│
         │           └────────────┘
    ┌────┴────┐
    │dim_merch│
    └────┬────┘
         │
    ┌────┴──────┐
    │dim_location│
    └───────────┘

          ┌──────────────────┐
          │ fact_fraud_alerts│
          │ grain: 1 row /   │
          │  fraud event     │
          └────┬───────┬─────┘
          ┌────┴────┐  ┌─────┴──────────┐
          │dim_acct │  │dim_transaction_type│
          └─────────┘  └────────────────────┘
```

**Key annotations:**
- `fact_transactions` — 1 row per transaction event (transactional fact table)
- `fact_account_balances` — daily snapshot (periodic snapshot fact table)
- `fact_fraud_alerts` — 1 row per fraud detection event (transactional)
- All foreign keys are surrogate keys (`_key` suffix) referencing dimension natural+surrogate pairs.

---

## 4. Detailed Table Definitions

### 4a. Fact Tables

#### `fact_transactions` (Transaction Fact)

| Column | Type | Notes |
|--------|------|-------|
| `transaction_key` | BIGINT (PK) | Surrogate key, auto-increment / hash |
| `transaction_id` | VARCHAR(64) | Business key, never overwritten |
| `date_key` | INT → dim_date | Transaction date |
| `account_key` | BIGINT → dim_account | Originating account |
| `merchant_key` | BIGINT → dim_merchant | Counterparty (NULL for ATM/transfer) |
| `customer_key` | BIGINT → dim_customer | Account holder |
| `transaction_type_key` | INT → dim_transaction_type | Deposit, withdrawal, transfer, POS, wire |
| `channel_key` | INT → dim_channel | Online, branch, ATM, mobile, POS |
| `location_key` | INT → dim_location | Where the transaction occurred |
| `fx_rate_key` | INT → dim_fx_rate | FX snapshot at time of txn |
| `amount_original` | DECIMAL(18,4) | Amount in local currency |
| `amount_usd_equivalent` | DECIMAL(18,4) | USD-equivalent for cross-border analysis |
| `fee_amount` | DECIMAL(18,4) | Fee charged |
| `reversal_flag` | BOOLEAN | TRUE if this row is a reversal of a prior txn |
| `reversed_transaction_key` | BIGINT | FK to the original txn being reversed |
| `status` | VARCHAR(16) | completed, pending, reversed, declined |
| `created_at` | TIMESTAMP | ETL insert timestamp (audit) |
| `processed_at` | TIMESTAMP | When settlement completed |

> **Grain:** One row per transaction event. A reversal inserts a *new row* with `reversal_flag=TRUE`; the original row remains untouched (append-only, immutable).

#### `fact_account_balances` (Periodic Snapshot Fact)

| Column | Type | Notes |
|--------|------|-------|
| `balance_key` | BIGINT (PK) | Surrogate |
| `date_key` | INT → dim_date | End-of-day (EOD) snapshot date |
| `account_key` | BIGINT → dim_account | Account ID |
| `customer_key` | BIGINT → dim_customer | Account holder |
| `opening_balance` | DECIMAL(18,4) | Balance at start of day |
| `closing_balance` | DECIMAL(18,4) | Balance at end of day |
| `daily_change` | DECIMAL(18,4) | Derived: closing - opening |
| `total_deposits` | DECIMAL(18,4) | Sum of deposits that day |
| `total_withdrawals` | DECIMAL(18,4) | Sum of withdrawals that day |
| `overdraft_flag` | BOOLEAN | TRUE if balance went negative at any point |
| `fx_rate_key` | INT → dim_fx_rate | FX rate on snapshot date |

> **Why a periodic snapshot?** Account balance is inherently *stateful* — it accumulates over time. A daily snapshot lets analysts answer "What was account X's balance on 2024-03-15?" and compute day-over-day changes without re-aggregating millions of transactions. This is a classic Kimball **periodic snapshot fact table**.

#### `fact_fraud_alerts` (Transactional Fact)

| Column | Type | Notes |
|--------|------|-------|
| `fraud_alert_key` | BIGINT (PK) | Surrogate |
| `transaction_key` | BIGINT → fact_transactions | Linked transaction |
| `date_key` | INT → dim_date | Alert date |
| `account_key` | BIGINT → dim_account | Flagged account |
| `risk_score` | DECIMAL(5,2) | 0.00–100.00 |
| `fraud_type` | VARCHAR(32) | Stolen card, identity theft, AML, structuring |
| `action_taken` | VARCHAR(16) | blocked, flagged, approved |
| `created_at` | TIMESTAMP | Detection timestamp |

> **Grain:** One row per fraud detection event (one transaction may trigger multiple alerts over time).

### 4b. Dimension Tables

#### `dim_date`

| Column | Type | Notes |
|--------|------|-------|
| `date_key` | INT (PK) | YYYYMMDD integer |
| `date` | DATE | Full date |
| `day_of_week` | TINYINT | 1–7 |
| `week_number` | TINYINT | ISO week |
| `month` | TINYINT | 1–12 |
| `quarter` | TINYINT | 1–4 |
| `fiscal_year` | SMALLINT | Fiscal calendar |
| `is_weekend` | BOOLEAN | |
| `is_holiday` | BOOLEAN | Country-specific holidays |

#### `dim_account`

| Column | Type | Notes |
|--------|------|-------|
| `account_key` | BIGINT (PK) | Surrogate |
| `account_id` | VARCHAR(32) | Business key |
| `account_type` | VARCHAR(16) | checking, savings, credit_card, wire |
| `opening_date` | DATE | |
| `closing_date` | DATE | NULL = active (SCD Type 2) |
| `currency_code` | CHAR(3) | ISO 4217 (USD, EUR, GBP…) |
| `status` | VARCHAR(16) | active, closed, frozen |
| `risk_rating` | VARCHAR(8) | low, medium, high |

#### `dim_customer`

| Column | Type | Notes |
|--------|------|-------|
| `customer_key` | BIGINT (PK) | Surrogate |
| `customer_id` | VARCHAR(32) | Business key |
| `first_name` | VARCHAR(64) | |
| `last_name` | VARCHAR(64) | |
| `email` | VARCHAR(128) | |
| `segment` | VARCHAR(32) | retail, premium, private_banking |
| `kyc_status` | VARCHAR(16) | verified, pending, rejected |

#### `dim_merchant`

| Column | Type | Notes |
|--------|------|-------|
| `merchant_key` | BIGINT (PK) | Surrogate |
| `merchant_id` | VARCHAR(32) | Business key |
| `name` | VARCHAR(128) | |
| `category` | VARCHAR(64) | retail, travel, online, food |
| `mcc_code` | VARCHAR(4) | Merchant Category Code |
| `country_code` | CHAR(2) | |

#### `dim_transaction_type`

| Column | Type | Notes |
|--------|------|-------|
| `transaction_type_key` | INT (PK) | Surrogate |
| `type_code` | VARCHAR(16) | deposit, withdrawal, transfer, pos_purchase, wire_in, wire_out |
| `description` | VARCHAR(64) | |
| `is_inflow` | BOOLEAN | TRUE for deposits/transfers in |
| `is_reversable` | BOOLEAN | Can the transaction be reversed? |

#### `dim_channel`

| Column | Type | Notes |
|--------|------|-------|
| `channel_key` | INT (PK) | Surrogate |
| `channel_code` | VARCHAR(16) | online, branch, atm, mobile, pos |
| `description` | VARCHAR(64) | |

#### `dim_location`

| Column | Type | Notes |
|--------|------|-------|
| `location_key` | INT (PK) | Surrogate |
| `latitude` | DECIMAL(9,6) | |
| `longitude` | DECIMAL(9,6) | |
| `city` | VARCHAR(64) | |
| `state_province` | VARCHAR(64) | |
| `country_code` | CHAR(2) | |

#### `dim_fx_rate`

| Column | Type | Notes |
|--------|------|-------|
| `fx_rate_key` | INT (PK) | Surrogate |
| `date_key` | INT → dim_date | Effective date |
| `from_currency` | CHAR(3) | Source |
| `to_currency` | CHAR(3) | Target (e.g. USD) |
| `rate` | DECIMAL(12,6) | Exchange rate |
| `effective_time` | TIMESTAMP | For intraday rates (fraud detection) |

---

## 5. Discussion Points

### 5a. Compliance & Immutability

Financial regulators (SOX, GDPR, PCI-DSS, Basel III) require **7-year retention** and **immutable audit trails**:
- **Never DELETE** from `fact_transactions`. Use `reversal_flag` to record corrections.
- Add an `etl_batch_id` column to every fact for lineage tracking.
- Archive cold data to object storage (S3/GCS) after 3 years; keep metadata in the DW.

### 5b. Fraud Detection as a Derived Dimension

Fraud is **not** stored as a column on the transaction fact. Instead, `fact_fraud_alerts` is a separate fact with its own grain. This lets analysts:
- Count alerts per account without skewing transaction aggregates.
- Join back to `fact_transactions` for root-cause analysis.
- Apply ML scores (`risk_score`) without mutating the core transaction log.

### 5c. Multi-Currency Support

| Concern | Solution |
|---------|----------|
| Different account currencies | `currency_code` on `dim_account` |
| Transaction in foreign currency | `amount_original` + `amount_usd_equivalent` on fact |
| FX rate lookup at time of txn | `dim_fx_rate` with intraday timestamps |
| Balance reporting in reporting currency | Pre-compute USD-equivalent on the snapshot fact |

### 5d. Handling Transaction Reversals

```
Original transaction (T1):  amount = +500,  reversal_flag = FALSE
Reversal transaction (T2):  amount = -500,  reversal_flag = TRUE,  reversed_transaction_key → T1
```

Both rows persist. Aggregates filter `WHERE reversal_flag = FALSE` unless the analyst explicitly wants reversal data.

---

## 6. Top-Up Questions (Senior-Level Probes)

| # | Question | What the interviewer is testing |
|---|----------|--------------------------------|
| 1 | *"How do you handle back-dated transactions that arrive weeks late?"* | Understanding of SCD Type 2, late-arriving facts, and how snapshots are regenerated. |
| 2 | *"Would you normalize the fraud-alert data into a separate schema? Why or why not?"* | Awareness of logical separation vs. physical separation; conformed dimensions. |
| 3 | *"How does your model support AML (Anti-Money Laundering) pattern detection like 'structuring'?"* | Ability to model for analytical joins and window-function queries across accounts. |
| 4 | *"The CFO wants a single 'customer revenue' number. Where does that live?"* | Understanding of aggregation layers (ODS → data marts → executive dashboards). |

---

## 7. Practice Exercise

> **Scenario:** The bank now offers **investment and portfolio management**. Customers can buy/sell stocks, bonds, and mutual funds through their brokerage accounts.

### Your Task

Extend the model above to support:

1. **New fact tables** for trades (buys/sells) and daily portfolio snapshots.
2. **New dimensions**: `dim_security` (stock/bond/mutual fund), `dim_portfolio`, `dim_brokerage_account`.
3. Show how a **daily portfolio snapshot** relates to the existing `fact_account_balances` model.
4. Add a column to `dim_customer` that tracks total investable assets (or explain why a derived fact is better).
5. Draw the updated high-level diagram (ASCII is fine).

### Hints

- Trades are analogous to transactions — same patterns apply.
- A portfolio snapshot is a **periodic snapshot** fact, just like `fact_account_balances`.
- The `dim_security` dimension should hold ticker, issuer, asset_class, and risk_category.
- Consider a **bridge table** if one portfolio holds many securities (one-to-many).

---

### ✅ Key Takeaways

| Principle | Application |
|-----------|-------------|
| **Append-only facts** | Never delete; use reversal flags |
| **Periodic snapshot facts** | Balance and portfolio state over time |
| **Conformed dimensions** | `dim_customer` shared across banking and investing |
| **Separate facts for separate grains** | `fact_fraud_alerts` ≠ a column on `fact_transactions` |
| **Compliance drives design** | 7-year retention, immutable logs, audit trail on every row |

---

### Related Lessons

- [Creating High-Level Model Diagrams](./13-creating-high-level-model-diagrams.md)
- [Practice: Ridesharing Platform](./18-practice-ridesharing-platform.md)
- [Practice: IoT Sensor Data](./20-practice-iot-sensor-data.md)
- [Data Retention Policies and Historical Data Management](./11-data-retention-policies-and-historical-data-management.md)
