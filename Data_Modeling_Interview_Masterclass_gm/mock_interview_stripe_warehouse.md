# Lesson 34: Mock Interview 4 — Design a Data Warehouse Schema for Stripe

> **Principal Mentor Note:** Stripe is the gold standard **Fintech & Payments Infrastructure** data modeling interview. If you design Stripe like a generic e-commerce store, you will fail. At the Principal level, you must model **Stripe's canonical payment state machine** (`PaymentIntent -> Charge -> Capture -> Refund -> Dispute/Chargeback`), an **immutable double-entry balance transaction ledger** (`fct_balance_transactions`), **exact integer minor-unit arithmetic** (`cents` / `zero-decimal currencies` like JPY), **interchange++ vs. blended pricing tiers**, and **PCI-DSS / AML / GDPR regulatory isolation**.

## 1. Phase 1: Clarifying Questions & Scope Alignment (Minutes 0–6)

### Interviewer Prompt

> *"Design the core analytical and financial reconciliation data warehouse for Stripe. Leadership, Finance, and Risk teams need to track Gross Payment Volume (GPV), authorization conversion rates, net revenue after network interchange/scheme fees, dispute/chargeback ratios, and merchant payouts."*

### Candidate's 4-Bucket Discovery Questions

1. **Product Scope:** *"Should we focus on core Stripe Payments (`PaymentIntents`, card/wallet authorizations, captures, refunds, disputes, and merchant bank payouts), or also include Stripe Billing/Subscriptions and Stripe Connect multi-party marketplace splits?"*

   * *Interviewer:* *"Focus on core Payments through Settlement & Payouts, plus touch on how Connect marketplace platform fees fit in."*

2. **Financial Precision & Multi-Currency:** *"Merchants process payments in 135+ currencies (including zero-decimal currencies like `JPY` and three-decimal currencies like `KWD`) and settle in their local bank currency with FX conversion fees. Should we store exact minor-unit integers alongside normalized USD equivalents?"*

   * *Interviewer:* *"Yes—exact minor units in original currency plus FX-locked USD values are mandatory for zero penny-drift."*

3. **Pricing Models & Historical Truth:** *"Merchants move between `Standard Blended (2.9% + 30¢)` and custom `Enterprise Interchange++` contracts. Does Finance need exact historical fee structures at the moment of each transaction (`as-was`)?"*

   * *Interviewer:* *"Yes, historical merchant fee tier and MCC (Merchant Category Code) at transaction time are strictly required."*

4. **Scale, Latency & Regulatory Compliance:** *"What daily transaction volume, freshness SLA, and compliance mandates (PCI-DSS, 7-year AML retention, GDPR) apply?"*

   * *Interviewer:* *"Assume 250M payment attempts/day, 400M ledger balance entries/day, 15-minute freshness for Risk/Auth monitoring, 7-year immutable financial retention, strict PCI-DSS token isolation, and GDPR compliance."*

## 2. Phase 2: Enterprise Bus Matrix & Grain Contract (Minutes 6–12)

### Enterprise Bus Matrix

```
Business Process (Fact Table)              | Date | Time | Merchant | Payment Method | Card Network/BIN | Geo/Corridor | Fee Contract | Dispute Reason
-------------------------------------------+------+------+----------+----------------+------------------+--------------+--------------+---------------
fct_payment_intents (Accumulating Snap)    |  X   |  X   |    X     |       X        |        X         |      X       |      X       |
fct_payment_attempts (Transaction Fact)    |  X   |  X   |    X     |       X        |        X         |      X       |              |
fct_balance_transactions (Ledger Fact)     |  X   |  X   |    X     |       X        |        X         |      X       |      X       |
fct_disputes (Accumulating Snapshot)       |  X   |  X   |    X     |       X        |        X         |      X       |              |       X
fct_merchant_daily_balance (Periodic Snap) |  X   |      |    X     |                |                  |      X       |              |

```

### Explicit Grain Declarations (The Double-Entry Ledger Core)

1. **`fct_payment_intents` (Accumulating Snapshot):** One row per customer checkout intent (`payment_intent_id`), tracking lifecycle milestones (`created -> requires_action (3DS) -> authorized -> captured -> succeeded/failed`), retry counts, and 3DS authentication friction.

2. **`fct_balance_transactions` (Immutable Double-Entry Ledger Fact):** One row per financial movement affecting a merchant's Stripe balance (`balance_txn_id`)—covering `CHARGE`, `REFUND`, `DISPUTE_WITHDRAWAL`, `DISPUTE_REVERSAL`, `PAYOUT`, and `STRIPE_FEE_ADJUSTMENT`. **Every single dollar entering or leaving Stripe is accounted for here.**

3. **`fct_merchant_daily_balance` (Periodic Snapshot):** One row per `(merchant_sk, currency_code, snapshot_date_sk)`, enforcing the daily ledger invariant $\text{BOD Available} + \text{Net Ledger Movements} - \text{Payouts} = \text{EOD Available}$.

## 3. Phase 3: High-Level Constellation Diagram & DDL (Minutes 12–26)

```
+------------------------+       +----------------------+       +------------------------+
| dim_merchant [SCD2]    |       |  dim_date [CONFORM]  |       | dim_payment_method     |
| PK: merchant_sk        |       |  PK: date_sk         |       | [JUNK / LOOKUP DIM]    |
| NK: merchant_id        |       +----------+-----------+       | card_brand, funding,   |
| mcc_code, pricing_tier,|            ^            ^            | wallet_type, 3ds_ver   |
| fixed_fee_cents, pct   |            |            |            +-----------+------------+
+-----------+------------+            |            |                        | 1
            | 1                       |            |                        |
            |                 +-------+----+  +----+---------------+        | N
            | N               | fct_payment|  | fct_balance_       |<-------+
            +---------------->| _intents   |  | transactions [TXN] |
                              | [ACC-SNAP] |  | Grain: 1 row /     |
+------------------------+    | Grain: 1 / |  | balance_txn_id     |    +------------------------+
| dim_geo_corridor [SCD1]|--->| intent_id  |  +----+---------------+    | dim_card_bin [SCD1]    |
| merchant_country,      |1:N +------------+       ^                    | bin_prefix_6,          |
| issuer_country,        |                         | 1:N                | issuing_bank_name,     |
| is_cross_border_flag   |               +---------+----------+         | card_product_tier      |
+------------------------+               | fct_disputes       |         +------------------------+
                                         | [ACC-SNAP]         |
                                         | Grain: 1/dispute_id|
                                         +--------------------+

```

### Production DDL (Apache Iceberg / Snowflake)

```
-- ============================================================================
-- DIMENSIONS
-- ============================================================================
CREATE TABLE gold.dim_merchant (
    merchant_sk             BIGINT PRIMARY KEY,
    merchant_id             VARCHAR(64) NOT NULL,  -- acct_xxx
    connect_platform_id     VARCHAR(64),           -- Populated if Connect sub-merchant
    business_name           VARCHAR(255) NOT NULL,
    mcc_code                CHAR(4) NOT NULL,      -- ISO 18245 Merchant Category Code
    mcc_risk_tier           VARCHAR(16) NOT NULL,  -- LOW, STANDARD, HIGH_RISK
    country_code            CHAR(2) NOT NULL,
    pricing_model           VARCHAR(24) NOT NULL,  -- BLENDED_STD, INTERCHANGE_PLUS, CUSTOM_ENT
    blended_rate_bps        INT,                   -- e.g., 290 = 2.90% (SCD2 tracked!)
    blended_fixed_cents     INT,                   -- e.g., 30 cents (SCD2 tracked!)
    -- PII (beneficial owner SSN/EIN, bank routing/account) isolated in PCI/PII Vault!
    valid_from_utc          TIMESTAMP NOT NULL,
    valid_to_utc            TIMESTAMP NOT NULL,
    is_current              BOOLEAN NOT NULL
);

CREATE TABLE gold.dim_geo_corridor (
    corridor_sk             INT PRIMARY KEY,       -- Compact Junk/Geo Dim (~60K country pairs)
    merchant_country_code   CHAR(2) NOT NULL,
    issuer_country_code     CHAR(2) NOT NULL,
    is_cross_border         BOOLEAN NOT NULL,      -- Drives cross-border FX & scheme fees!
    is_intra_eea            BOOLEAN NOT NULL       -- European Economic Area interchange cap flag
);

-- ============================================================================
-- FACT 1: DOUBLE-ENTRY BALANCE LEDGER (Immutable Transaction Fact)
-- Grain: One row per balance movement (balance_txn_id: txn_xxx)
-- ============================================================================
CREATE TABLE gold.fct_balance_transactions (
    balance_txn_id          VARCHAR(64) PRIMARY KEY, -- txn_xxx (Idempotency PK)
    source_object_id        VARCHAR(64) NOT NULL,  -- ch_xxx, re_xxx, dp_xxx, po_xxx (Degenerate)
    payment_intent_id       VARCHAR(64),           -- pi_xxx (Links refunds/disputes to original payment!)
    payout_id               VARCHAR(64),           -- po_xxx (Populated when swept to merchant bank)
    created_date_utc_sk     INT NOT NULL,          -- Partition key
    available_on_date_utc_sk INT NOT NULL,         -- Settlement availability date
    created_at_utc          TIMESTAMP NOT NULL,
    merchant_sk             BIGINT NOT NULL,       -- Historical SCD2 SK (exact pricing contract!)
    merchant_id             VARCHAR(64) NOT NULL,  -- Natural key for fast merchant dashboard pruning
    payment_method_sk       INT NOT NULL,
    corridor_sk             INT NOT NULL,
    txn_category            VARCHAR(32) NOT NULL,  -- CHARGE, REFUND, DISPUTE, DISPUTE_REVERSAL, PAYOUT, FEE
    reporting_currency      CHAR(3) NOT NULL,      -- Merchant's settlement currency
    presentment_currency    CHAR(3) NOT NULL,      -- Customer's checkout currency
    fx_rate_present_to_settle DECIMAL(18,8) NOT NULL,
    fx_rate_settle_to_usd   DECIMAL(18,8) NOT NULL,
    -- EXACT INTEGER MINOR UNITS (Zero Float Drift!) + Normalized USD Equivalents
    gross_amount_minor      BIGINT NOT NULL,       -- Signed: +10000 for $100 charge, -10000 for refund
    interchange_fee_minor   BIGINT NOT NULL,       -- Pass-through bank interchange cost
    card_scheme_fee_minor   BIGINT NOT NULL,       -- Visa/Mastercard network assessment fee
    stripe_gross_fee_minor  BIGINT NOT NULL,       -- Total fee charged to merchant
    net_merchant_amount_minor BIGINT NOT NULL,     -- gross_amount - stripe_gross_fee
    -- USD Normalized Measures for Cross-Currency Executive Rollups
    gross_gpv_usd           DECIMAL(16,4) NOT NULL,-- Unsigned GPV (only >0 on CHARGE)
    net_cash_movement_usd   DECIMAL(16,4) NOT NULL,-- Signed net movement
    stripe_net_revenue_usd  DECIMAL(16,4) NOT NULL -- stripe_gross_fee - interchange - scheme_fee
)
PARTITION BY (created_date_utc_sk)
CLUSTER BY (merchant_id, txn_category);

-- ============================================================================
-- FACT 2: PAYMENT INTENTS FUNNEL (Accumulating Snapshot)
-- Grain: One row per checkout attempt (payment_intent_id: pi_xxx)
-- ============================================================================
CREATE TABLE gold.fct_payment_intents (
    payment_intent_id       VARCHAR(64) PRIMARY KEY,
    created_date_utc_sk     INT NOT NULL,
    created_at_utc          TIMESTAMP NOT NULL,
    succeeded_at_utc        TIMESTAMP,
    merchant_sk             BIGINT NOT NULL,
    payment_method_sk       INT NOT NULL,
    corridor_sk             INT NOT NULL,
    terminal_status         VARCHAR(32) NOT NULL,  -- SUCCEEDED, FAILED_DECLINED, CANCELED, REQUIRES_ACTION
    last_decline_code       VARCHAR(48),           -- 'insufficient_funds', 'do_not_honor', '3ds_failed'
    is_3ds_challenged       BOOLEAN NOT NULL,
    auth_attempt_count      INT NOT NULL,          -- Captures smart-retry recovery!
    is_recovered_by_retry   BOOLEAN NOT NULL,      -- TRUE if attempt 1 failed & attempt 2+ succeeded
    amount_usd              DECIMAL(16,4) NOT NULL,
    auth_to_capture_ms      BIGINT
)
PARTITION BY (created_date_utc_sk)
CLUSTER BY (merchant_sk, terminal_status);

```

## 4. Phase 4: Query Validation (Minutes 26–33)

```
-- Query 1: Cross-Border vs. Domestic GPV, Net Margin (bps) & Dispute Rate by MCC Industry
SELECT
    d.year_month,
    m.mcc_risk_tier,
    gc.is_cross_border,
    SUM(bt.gross_gpv_usd)                                           AS total_gpv_usd,
    SUM(bt.stripe_net_revenue_usd)                                  AS stripe_net_revenue_usd,
    -- Net Margin in Basis Points (1 bp = 0.01%)
    ROUND(
        SUM(bt.stripe_net_revenue_usd) * 10000.0 /
        NULLIF(SUM(bt.gross_gpv_usd), 0),
    2)                                                              AS net_margin_bps,
    ROUND(
        SUM(CASE WHEN bt.txn_category = 'DISPUTE' THEN ABS(bt.net_cash_movement_usd) ELSE 0 END) * 100.0 /
        NULLIF(SUM(bt.gross_gpv_usd), 0),
    4)                                                              AS dispute_volume_pct
FROM gold.fct_balance_transactions bt
JOIN gold.dim_date d ON bt.created_date_utc_sk = d.date_sk
JOIN gold.dim_merchant m ON bt.merchant_sk = m.merchant_sk
JOIN gold.dim_geo_corridor gc ON bt.corridor_sk = gc.corridor_sk
WHERE d.calendar_year = 2026
GROUP BY 1, 2, 3
ORDER BY 4 DESC;

```

## 5. Phase 5: Principal Curveballs & "Top-Up" Questions (Minutes 33–45)

1. **Curveball 1 (Delayed Interchange Clearing):** *"Card networks (Visa/Mastercard) settle actual interchange fees 24 to 72 hours AFTER Stripe captures a payment. At capture time (`T=0`), Stripe only has an estimated interchange fee. How do you model the true interchange adjustment 3 days later without mutating an immutable financial ledger?"*

   * **Principal Answer:** Never overwrite the original `CHARGE` row in `fct_balance_transactions`! Record `interchange_fee_minor` at `T=0` using the real-time estimated rate, and when the card network clearing file (EPRAW / IPM) arrives on `T+3`, insert a new immutable ledger adjustment row (`txn_category = 'INTERCHANGE_CLEARING_ADJ'`, linked via `payment_intent_id`) capturing the exact delta (`actual_interchange - estimated_interchange`).

2. **Curveball 2 (Zero-Decimal Currencies like JPY vs. USD):** *"A merchant processes `¥10,000 JPY` (`~$65 USD`) and `$100.00 USD` (`10000` cents). If an analyst runs `SUM(gross_amount_minor)` across currencies, they add `10,000 Yen` + `10,000 Cents`!"*

   * **Principal Answer:** Enforce a strict governance rule: `gross_amount_minor` may **only** be aggregated when `GROUP BY reporting_currency` (or filtered to a single currency), while all cross-currency rollups use `gross_gpv_usd` / `stripe_net_revenue_usd` computed at ingestion using `minor_unit_exponent` (`0` for JPY, `2` for USD, `3` for KWD) from `dim_currency`.

3. **Curveball 3 (PCI-DSS & GDPR Isolation):** *"How do you prevent raw credit card numbers (PANs) or merchant owner PII from ever entering your analytical lakehouse?"*

   * **Principal Answer:** Raw PANs never leave Stripe's isolated PCI Card Data Environment (CDE)—analytical streams receive only non-reversible surrogate tokens (`pm_xxx`, `card_fingerprint`) and the first 6/8 digits (`BIN`) joined to `dim_card_bin`. Merchant beneficial-owner PII is isolated in a KMS-encrypted `dim_merchant_pii` vault keyed by `merchant_sk`, satisfying both 7-year AML ledger immutability and GDPR crypto-shredding.