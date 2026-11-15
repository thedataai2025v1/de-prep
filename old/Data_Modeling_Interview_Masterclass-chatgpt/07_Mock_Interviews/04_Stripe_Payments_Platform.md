# Mock Interview 4 — Stripe-Like Payments and Financial Data Platform

**Interview level:** Staff / Principal Data Engineer  
**Suggested duration:** 90 minutes  
**Primary challenge:** Build a financially correct analytical model for payment intent, authorization, capture, refunds, disputes, balance transactions, fees, transfers, and payouts. The key test is whether you distinguish a payment's business lifecycle from actual money movement and accounting evidence.

## 1. Interview brief

> Design a global payments data platform supporting merchant analytics, payment success, authorization/capture funnels, refunds, disputes, fees, settlement reconciliation, balances, payouts, and finance reporting.

This is a correctness-first interview. A model that is fast but cannot reconcile money is not acceptable.

## 2. Discovery questions

| Question | Assumption |
|---|---|
| Is one customer checkout always one payment? | No; retries and multiple payment attempts exist |
| Can authorization and capture be separate? | Yes; partial/multiple capture may exist depending on method/product |
| Can one payment be partially refunded multiple times? | Yes |
| Are disputes separate from refunds? | Yes |
| Are funds settled immediately? | No; pending/available balance timing matters |
| Are there multiple currencies? | Yes, with explicit conversion events/rates |
| Can one merchant payout aggregate many transactions? | Yes |
| Are connected accounts/transfers supported? | Yes |
| What is the accounting source of truth? | Immutable balance/ledger entries plus reconciled external settlement records |
| Can events be duplicated or delivered out of order? | Yes |

## 3. Domain model: do not collapse the state machines

```text
MERCHANT ──< PAYMENT_INTENT ──< PAYMENT_ATTEMPT
                    │                  │
                    │                  └── authorization / charge / capture
                    ├──< REFUND
                    ├──< DISPUTE ──< DISPUTE_EVENT
                    └──< BALANCE_TRANSACTION ──< FEE_DETAIL

MERCHANT ──< TRANSFER ──< TRANSFER_REVERSAL
BALANCE_TRANSACTION ──< PAYOUT_ALLOCATION >── PAYOUT
CURRENCY / PAYMENT_METHOD / CARD_NETWORK / COUNTRY
```

Names and exact object relationships differ by processor and API version. Treat the diagram as a conceptual model; validate source semantics before mapping IDs.

## 4. Business processes and grains

| Process | Fact | Grain |
|---|---|---|
| Payment intent lifecycle | `fact_payment_intent` | One row per payment intent |
| Attempt funnel | `fact_payment_attempt` | One row per attempt |
| Authorization/capture | `fact_authorization_capture_event` | One row per authorization/capture event |
| Money movement | `fact_balance_transaction` | One row per processor balance transaction |
| Fee detail | `fact_fee_component` | One row per balance transaction × fee component |
| Refund | `fact_refund` | One row per refund object/financial event |
| Dispute | `fact_dispute` | One row per dispute |
| Transfer | `fact_transfer` | One row per transfer |
| Payout | `fact_payout` | One row per payout |
| Payout allocation | `bridge_payout_balance_transaction` | One row per payout × balance transaction allocation |
| Ledger | `fact_ledger_entry` | One row per journal line / debit-credit posting |

The **balance transaction** or internal ledger posting is the critical money-movement grain. Payment intent status alone cannot prove that cash settled.

## 5. Dimensional model and core keys

Dimensions:
- `dim_merchant` (SCD2 for tier, country, risk segment where permitted).
- `dim_customer` (pseudonymous analytics identity).
- `dim_payment_method`, `dim_card_network`, `dim_country`, `dim_currency`.
- `dim_failure_reason`, `dim_dispute_reason`, `dim_transaction_type`.
- `dim_date` with event, created, available, settlement, and payout date roles.
- `dim_product` / `dim_payment_channel`.

Retain source object IDs as durable degenerate identifiers. Surrogate keys are useful for warehouse dimensions, but do not discard processor IDs needed for reconciliation and support.

## 6. Money and accounting correctness

### Amount representation
- Preserve original integer minor-unit amount and currency.
- Do not convert to floating point for financial calculations.
- Store currency exponent/scale metadata where needed; not every currency uses two decimal places.
- Store FX rate, rate source, quote timestamp, source and target currencies, and both amounts.
- Do not aggregate multiple currencies before conversion to a documented reporting currency.

### Immutable postings
Use append-only ledger/balance transaction records. Corrections are reversals or compensating entries, not destructive updates. Model debit/credit direction explicitly and enforce a balancing invariant at the appropriate journal scope.

Illustrative ledger:
```sql
CREATE TABLE fact_ledger_entry (
  journal_id          VARCHAR(64) NOT NULL,
  journal_line_id     VARCHAR(64) NOT NULL,
  account_id          VARCHAR(64) NOT NULL,
  merchant_key        BIGINT,
  source_object_id    VARCHAR(64),
  posting_ts          TIMESTAMP NOT NULL,
  currency_code       CHAR(3) NOT NULL,
  debit_minor         BIGINT DEFAULT 0,
  credit_minor        BIGINT DEFAULT 0,
  entry_type          VARCHAR(40),
  reversal_of_line_id VARCHAR(64),
  source_version      BIGINT,
  ingested_at         TIMESTAMP
);
```
This is an illustrative analytical ledger representation, not a claim about a processor's internal ledger schema. A journal must balance according to the accounting design and currency rules.

### Example: partial refund
Original capture: +10,000 minor units. Refund 1: -2,500. Refund 2: -1,000. Net customer charge before fees/other adjustments: 6,500. Preserve all three transactions. Never overwrite the original capture amount to 6,500.

## 7. Payment lifecycle and event ingestion

A typical payment intent can move through states such as requires payment method, requires confirmation/action, processing, succeeded, or canceled. An attempt may fail while the intent remains retryable. Authorization may occur before capture; capture can be partial depending on product and payment method. Refund and dispute processes continue after payment success.

Ingestion pattern:
1. Receive webhook/event/API/CDC data.
2. Persist raw envelope, event ID, object ID, event creation time, API version, received time, and payload.
3. Deduplicate on provider event ID; retain delivery attempts separately if operationally useful.
4. Fetch/reconcile canonical object state when events are missing or out of order.
5. Normalize into append-only lifecycle events and financial transactions.
6. Derive snapshots and marts.
7. Reconcile internal totals against provider balance reports and bank settlement.

Do not rely only on client redirects/callbacks to declare payment success. Asynchronous provider events and server-side reconciliation are essential.

## 8. Example SQL: merchant daily payment funnel

```sql
WITH intent_level AS (
  SELECT
    merchant_key,
    CAST(created_ts AS DATE) AS payment_date,
    payment_intent_id,
    MAX(CASE WHEN status = 'SUCCEEDED' THEN 1 ELSE 0 END) AS succeeded
  FROM fact_payment_intent
  GROUP BY merchant_key, CAST(created_ts AS DATE), payment_intent_id
)
SELECT merchant_key, payment_date,
       COUNT(*) AS intents,
       SUM(succeeded) AS successful_intents,
       1.0 * SUM(succeeded) / NULLIF(COUNT(*), 0) AS intent_success_rate
FROM intent_level
GROUP BY merchant_key, payment_date;
```
For attempt success rate, query `fact_payment_attempt` and use attempts as denominator. Do not mix intent success and attempt success.

### Example: net settled amount by currency
```sql
SELECT merchant_key, currency_code,
       SUM(net_amount_minor) AS net_balance_movement_minor
FROM fact_balance_transaction
WHERE status = 'AVAILABLE'
  AND available_ts >= :start_ts
  AND available_ts < :end_ts
GROUP BY merchant_key, currency_code;
```
A production definition must specify transaction types included/excluded and reconcile against the provider's balance reporting categories.

## 9. Reconciliation design

Reconcile at multiple layers:
1. Intent/charge: business payment state vs capture records.
2. Processor balance: captures, refunds, fees, disputes, transfers, adjustments.
3. Payout: payout amount vs allocated balance transactions.
4. Bank: payout paid/failed status vs bank statement/settlement file.
5. General ledger: journal totals by account, currency, and accounting date.

Build a reconciliation table with:
- source system and extract batch;
- currency and accounting period;
- expected amount, observed amount, delta;
- transaction count and unmatched count;
- tolerance and reason code;
- owner, status, resolution timestamp.

Never "fix" a reconciliation delta by silently adjusting a dashboard total. Investigate missing, duplicate, delayed, reversed, FX, fee, and timing differences.

## 10. Interview dialogue

**Interviewer:** "A PaymentIntent succeeded. Can we report the merchant has received the money?"  
**Candidate:** "No. It indicates payment completion at that object level, not necessarily funds availability or bank payout. I'd inspect capture, balance transaction, fee, transfer, payout, and settlement states."

**Interviewer:** "A webhook arrives twice."  
**Candidate:** "Persist the raw delivery, deduplicate the business event by provider event ID, and make downstream writes idempotent. If the same object update arrives with a newer version, apply version-aware logic."

**Interviewer:** "A refund is created but later fails."  
**Candidate:** "Refund requested and refund financially completed are separate states. Keep refund lifecycle events and associated balance movements; don't subtract the amount until the relevant financial state is reached according to the reporting definition."

**Interviewer:** "A dispute is lost after a payout."  
**Candidate:** "The dispute creates a subsequent financial movement and possibly a negative balance/adjustment. Preserve the original payout and add the dispute debit/related entries. Reconcile the timeline rather than rewriting historical payout records."

**Interviewer:** "Can we calculate revenue as captured amount minus refunds?"  
**Candidate:** "Not safely. We need to distinguish gross payment volume, merchant funds, processor fees, platform revenue, taxes, refunds, disputes, FX, and accounting recognition. Finance owns the revenue definition."

## 11. Principal-level probes

1. How would you guarantee traceability from a dashboard amount to provider and bank records?
2. What is the canonical financial event contract across payment methods?
3. How do you handle API version changes and provider schema evolution?
4. How do you design backfills without duplicating postings?
5. What does end-to-end exactly-once mean in a webhook-to-warehouse pipeline?
6. How do you isolate merchant data and enforce least privilege?
7. How do you operate reconciliation exceptions across Finance and Engineering?
8. How would you migrate a legacy mutable payment table to an immutable event/ledger model?

A strong principal answer proposes control totals, replay safety, lineage, ownership, auditability, exception workflows, and a staged migration with parallel reconciliation.

## 12. Common mistakes
- Treating payment intent, attempt, charge, balance transaction, and payout as synonyms.
- Overwriting capture amounts after refunds.
- Using floating point for money.
- Ignoring currency exponent and FX timestamp.
- Reporting succeeded payments as settled cash.
- Treating dispute as a simple status field on a payment.
- Assuming one payout equals one payment.
- Claiming webhook delivery is exactly once.

## 13. Practice exercise

A merchant captures 100.00 USD, receives a 20.00 partial refund, incurs a 3.20 fee, receives a 15.00 dispute debit later, and has a payout that includes this payment plus 12 other transactions.

Design:
- payment and attempt facts;
- immutable financial postings;
- payout allocation bridge;
- multi-currency treatment;
- reconciliation controls;
- a merchant dashboard that distinguishes gross volume, net balance movement, fees, and cash paid out.

### Scoring rubric (10 points)
| Dimension | Points |
|---|---:|
| Domain discovery and lifecycle separation | 1.5 |
| Correct financial grains and ledger thinking | 2.5 |
| Currency, refunds, disputes, payouts | 2.0 |
| Idempotency, replay, and reconciliation | 1.5 |
| Security, audit, and data quality | 1.0 |
| Principal ownership and trade-offs | 1.5 |

## 14. Further reading
- Stripe, [Events API](https://docs.stripe.com/api/events) — asynchronous lifecycle events.
- Stripe, [Capture a PaymentIntent](https://docs.stripe.com/api/payment_intents/capture) — authorization/capture semantics.
- Stripe, [Dispute object](https://docs.stripe.com/api/disputes/object) — dispute and balance transaction relationships.
- Stripe, [Payouts API](https://docs.stripe.com/api/payouts) — payout lifecycle.
