# Lesson 8: Practice — Online Advertising Platform

## Interview prompt

Design a high-level analytical model for an advertising platform. Advertisers create campaigns, define budgets and targeting, publish creatives, receive impressions and clicks, generate conversions, and are billed based on pricing models.

## 1. Clarify business questions

- What is the campaign hierarchy?
- Can campaigns contain multiple ad groups and creatives?
- Can one impression be served by multiple auctions or placements?
- What counts as a valid impression or click?
- How is conversion attribution defined?
- What is the attribution window?
- How are invalid traffic and click fraud handled?
- Is billing based on impressions, clicks, conversions, or fixed fees?
- How are budgets, pacing, and spend updated?
- What latency is required for campaign optimization?

## 2. Candidate conceptual diagram

```text
ADVERTISER 1 ─────< CAMPAIGN
CAMPAIGN 1 ─────< AD_GROUP
AD_GROUP 1 ─────< AD
AD 1 ─────< CREATIVE_VERSION
CAMPAIGN ─────< CAMPAIGN_TARGETING >──── AUDIENCE_SEGMENT

AD ─────< IMPRESSION
IMPRESSION 0..* ───── CLICK
CLICK ─────< CONVERSION_ATTRIBUTION >──── CONVERSION

CAMPAIGN 1 ─────< BUDGET_CHANGE
CAMPAIGN ─────< BILLING_EVENT
```

The conversion attribution relationship may be many-to-many when multiple touchpoints receive credit. The attribution model and allocation weights must be explicit.

## 3. Candidate fact grains

| Fact | Grain |
|---|---|
| Impression | One row per served impression event |
| Click | One row per click event |
| Conversion | One row per conversion event |
| Attribution | One row per conversion-touchpoint-model allocation |
| Budget change | One row per campaign budget change |
| Billing event | One row per billable event or adjustment |
| Campaign performance snapshot | One row per campaign per reporting interval |

Keep raw events separate from derived attribution and billing outputs.

## 4. Metric definitions

- CTR: eligible clicks / eligible impressions.
- Conversion rate: define click- or impression-based denominator and attribution window.
- CPA: attributed spend / attributed conversions.
- ROAS: attributed conversion value / advertising spend.
- Pacing: actual spend compared with expected spend over the campaign period.

Clarify invalid traffic filtering, deduplication, currency, time zone, and whether spend is estimated or billed.

## 5. Interviewer–candidate conversation

**Interviewer:** One conversion has three ad touchpoints. How do you assign revenue?

**Candidate:** I'd preserve the conversion and touchpoints separately, then create an attribution allocation fact keyed by conversion, touchpoint, and attribution model/version. The allocation weights should sum to the agreed total, normally one for a fully allocated conversion, unless the policy explicitly permits otherwise.

**Interviewer:** Can we calculate CTR by summing daily CTR values?

**Candidate:** No. I'd aggregate eligible clicks and impressions separately, then divide total clicks by total impressions. Daily CTRs have different denominators and cannot be summed or simply averaged without weighting.

**Interviewer:** How do campaign budgets change over time?

**Candidate:** I'd record budget-change events with effective timestamps and maintain a current-state view if needed. For historical pacing, we need the budget that was applicable at each point in time, not only the current budget.

## 6. Follow-up questions
1. How do you model ad auction events?
2. How do you prevent duplicate impression counting?
3. How do you handle conversion attribution windows?
4. How do you version attribution models?
5. How do you reconcile estimated spend with billed spend?
6. How do you model campaign hierarchy changes?

## 7. Principal-level discussion
- Govern attribution semantics across product and finance.
- Preserve raw event evidence for audit and reprocessing.
- Handle invalid traffic and late conversions.
- Support high-volume streaming optimization and reconciled billing.
- Prevent model-version changes from silently rewriting historical performance.

## Evaluation rubric
- Requirements and metric clarity: 20%
- Process and relationship design: 20%
- Fact grain and attribution: 25%
- Billing and temporal correctness: 15%
- Scale and latency: 10%
- Communication: 10%

## Key takeaway
**Advertising models must separate delivery events, clicks, conversions, attribution allocations, budgets, and billing—and make attribution rules reproducible.**
