# Lesson 3: Analyzing Metrics

## Learning objectives

-   Convert KPI names into precise business definitions.
-   Identify numerator, denominator, filters, time basis, and grain.
-   Understand additive, semi-additive, and non-additive measures.
-   Design models that support consistent metric computation.

## 1. A metric is a contract

A metric definition should specify: - Business meaning. - Formula. -
Numerator and denominator, if applicable. - Inclusion and exclusion
rules. - Event date or reporting time basis. - Grain and aggregation
behavior. - Currency, unit, timezone, and rounding. - Treatment of
cancellations, refunds, duplicates, and late data. - Owner and version.

"Revenue" is not a sufficient definition. It may refer to gross
merchandise value, gross sales, net sales, cash collected, recognized
revenue, or platform fees. These are not interchangeable.

## 2. Decompose a metric

For conversion rate:

\[ `\text{Conversion Rate}`{=tex} =
`\frac{\text{Eligible converted sessions}}`{=tex}
{`\text{Eligible sessions}`{=tex}} \]

Clarify: - What is an eligible session? - What event counts as
conversion? - Is conversion attributed to the same session or a later
window? - Are bots and internal traffic excluded? - What timezone
defines the reporting day? - Are repeated conversions counted once or
multiple times?

Store the underlying counts where possible. A ratio should generally be
calculated after aggregating its numerator and denominator, not by
summing or blindly averaging row-level percentages.

## 3. Measure additivity

-   **Additive:** Can be summed across all relevant dimensions. Example:
    units sold.
-   **Semi-additive:** Can be summed across some dimensions but not all.
    Example: end-of-day account balance across accounts, but not across
    dates.
-   **Non-additive:** Cannot be meaningfully summed. Example:
    percentage, rate, average, unit price.

For averages, preserve sum and count: \[ `\text{Average}`{=tex} =
`\frac{\text{Sum}}{\text{Count}}`{=tex} \]

For weighted rates, aggregate the numerator and denominator rather than
averaging percentages without weights.

## 4. Metric grain and attribution

A metric's grain may differ from the grain of its source events.

Example: marketing conversion may be attributed to a campaign
touchpoint, user, session, or order. The attribution rule (first touch,
last touch, multi-touch, or another policy) is a business rule, not a
neutral technical detail.

Do not join orders to multiple campaign touchpoints and sum order value
without an attribution allocation rule; this can duplicate revenue.

## 5. Example: subscription churn

Possible definitions: - Customer churn: customers who leave divided by
customers eligible at period start. - Subscription churn: subscriptions
cancelled or expired divided by eligible subscriptions. - Revenue churn:
recurring revenue lost divided by starting recurring revenue.

Further decisions: - Voluntary versus involuntary churn. - Grace
periods. - Paused subscriptions. - Reactivations. - Upgrades and
downgrades. - New customers during the period. - Cohort and denominator
rules.

Different definitions answer different questions. A mature model may
publish multiple named measures rather than force one ambiguous churn
field.

## 6. Metric-to-model mapping

  -----------------------------------------------------------------------
  Metric                  Likely source process   Modeling considerations
  ----------------------- ----------------------- -----------------------
  Orders                  Order lifecycle         Order versus order-line
                                                  grain

  Net sales               Sales and returns       Revenue definition,
                                                  return linkage,
                                                  currency

  Active users            User activity events    Activity rule,
                                                  deduplication, time
                                                  window

  Average resolution time Case lifecycle          Start/end timestamps,
                                                  reopened cases

  Account balance         Balance snapshot        Semi-additive across
                                                  time

  Conversion rate         Funnel events           Session/user
                                                  attribution,
                                                  numerator/denominator
  -----------------------------------------------------------------------

## 7. Interviewer--candidate conversation

**Interviewer:** How would you model revenue?

**Candidate:** First I'd clarify which revenue concept the business
means---gross sales, net sales, recognized revenue, cash collected, or
another measure. I'd establish treatment of discounts, tax, shipping,
refunds, chargebacks, and currency conversion.

**Interviewer:** Can't we just sum order totals?

**Candidate:** Only if order totals have the agreed meaning and the
reporting use case is order-level. Product analysis may need order-line
measures. Returns and refunds may occur later and should be represented
with their own event grain or a clearly defined adjustment process.

**Interviewer:** How do you calculate average order value?

**Candidate:** I'd define the eligible order population and calculate
the sum of the chosen order value divided by the count of eligible
orders. I would avoid summing line-level values and dividing by line
count if the intended denominator is orders.

## 8. Common mistakes

-   Treating similarly named metrics as equivalent.
-   Averaging averages without weighting.
-   Summing ratios or balances across inappropriate dimensions.
-   Ignoring currency and timezone.
-   Duplicating measures through many-to-many joins.
-   Failing to define exclusions and restatements.

## 9. Practice questions

1.  Define net sales for an e-commerce business.
2.  Explain how to model conversion rate.
3.  Why is account balance semi-additive?
4.  How would you calculate average order value from order-line facts?
5.  Distinguish customer churn, subscription churn, and revenue churn.
6.  How do you handle multi-touch attribution without duplicating
    revenue?

## 10. Principal-level follow-ups

-   How do you establish a metric store or semantic layer?
-   Who owns metric definitions when finance and product disagree?
-   How do you version a KPI definition without rewriting history
    silently?
-   How do you reconcile warehouse metrics with financial systems?
-   How do you manage restatements when late data changes a published
    metric?

## Revision checklist

-   [ ] I define metric meaning, formula, grain, and filters.
-   [ ] I can identify numerator and denominator.
-   [ ] I understand measure additivity.
-   [ ] I can prevent fan-out and attribution duplication.
-   [ ] I account for currency, timezone, and late data.

## Key takeaway

**A metric is a governed business definition with explicit calculation
rules---not just a column or SQL expression.**
