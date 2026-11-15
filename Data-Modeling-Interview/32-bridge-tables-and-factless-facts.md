# Lesson 32 — Bridge Tables and Factless Fact Tables

## Learning Objectives
- Distinguish bridge tables from factless fact tables
- Model many-to-many relationships with effective-dated bridge tables
- Prevent double counting when joining bridge tables to facts
- Design allocation bridges for attribution analysis
- Use factless facts to track events and coverage (eligibility)
- Compare eligible populations to actual outcomes
- Handle fan-out risks in star schema joins

## Interview Dialogue

**Interviewer:** A customer can belong to several loyalty programmes. How do you model it?

**Candidate:** I would use a customer-programme bridge at one row per membership relationship, with effective dates if membership changes. If we need to attribute revenue, I would ask whether the business wants membership analysis or allocated revenue. The latter requires a defined allocation rule.

**Interviewer:** Why can't we just join the bridge to sales and sum revenue?

**Candidate:** Because a sale can match multiple memberships and be repeated once per bridge row. That creates fan-out. I would use distinct event counts, approved allocation weights, or a separate membership analysis depending on the question.

**Interviewer:** Give me an example of a factless fact table.

**Candidate:** Student attendance is a clear event fact: one row per student per class session, with no numeric measure. We can count attendance rows. Promotion eligibility is a coverage fact: one row per eligible customer and offer episode. Comparing eligibility to redemption lets us measure conversion and non-redemption.

**Interviewer:** Can a bridge also be factless?

**Candidate:** Yes. A relationship bridge may contain only keys and relationship attributes, so structurally it can resemble a factless fact. I distinguish them by purpose: the bridge resolves a multi-valued relationship for other facts, while a factless event or coverage fact is itself the record of an occurrence or opportunity being analysed.

## Bridge Tables

A bridge table resolves a relationship that is many-to-many or otherwise cannot be represented by a single foreign key in a conventional star schema. It may connect a dimension to another dimension or support a multivalued attribute.

### Example: Customer and Account

A customer may hold multiple accounts, and an account may have multiple owners. A simple `customer_sk` on `dim_account` cannot represent all owners without duplication or loss of information.

```text
dim_customer
  customer_sk
  customer_name

dim_account
  account_sk
  account_number

bridge_account_customer
  account_sk
  customer_sk
  relationship_type
  valid_from
  valid_to
  ownership_pct
```

**Grain:** one row per account-customer relationship version (if effective-dated).

A bridge may contain:
- Foreign keys to the related entities
- Relationship type
- Effective start/end dates
- Allocation weight or role
- Optional relationship attributes

### Effective Dating

Membership can change over time. If a customer is removed from an account, a historical transaction should not automatically be attributed to the new owner. Add validity intervals and join by event time when the business requires historical relationship context.

```sql
SELECT
    c.customer_name,
    SUM(f.transaction_amount * b.ownership_pct) AS attributed_amount
FROM fact_account_transaction f
JOIN bridge_account_customer b
  ON f.account_sk = b.account_sk
 AND f.event_ts >= b.valid_from
 AND f.event_ts <  b.valid_to
JOIN dim_customer c
  ON b.customer_sk = c.customer_sk
GROUP BY c.customer_name;
```

This query assumes ownership percentages are valid allocation weights and sum to 1 for each account and time interval. If the relationship is joint ownership but the business does not allocate value, do not multiply financial measures by an invented percentage.

### Fan-Out and Double Counting

Suppose one sale of $100 is associated with three campaign tags. Joining the sale fact to a three-row bridge produces three rows, each carrying $100. A naïve sum returns $300.

**Ways to handle fan-out:**
1. **Distinct entity/event counts** — use a distinct count of the business event when appropriate
2. **Allocation weights** — distribute measures according to an approved rule
3. **Separate analysis** — report membership counts without summing the original fact measure
4. **Pre-aggregate** — aggregate fact measures before joining, if the aggregation grain is valid
5. **Semantic-layer guardrails** — expose measures with explicit aggregation behaviour

Do not use `SUM(DISTINCT amount)` as a generic fix. Two different legitimate transactions may have the same amount, so it can undercount.

### Allocation Bridge

Example: one order can be attributed to multiple marketing campaigns. The business may approve a last-touch, first-touch, equal-share, or model-based allocation.

```text
bridge_order_campaign
  order_id
  campaign_sk
  attribution_model_sk
  allocation_pct
  valid_from
  valid_to
```

**Validation rules:**
- Every allocated order/model has the expected allocation coverage
- Weights sum to 1 within tolerance
- No overlapping duplicate relationship versions
- Attribution model version is retained for reproducibility

The attribution rule is a business definition, not a purely technical choice. Store the model/version used to produce the allocation.

### Hierarchy and Multivalued Bridges

Bridge tables can support:
- Customer ↔ loyalty programme
- Product ↔ multiple categories
- Employee ↔ multiple project teams
- Account ↔ multiple owners
- Campaign ↔ multiple channels
- Parent ↔ descendant hierarchy paths

For hierarchy closure tables, one row can represent an ancestor-descendant pair plus depth. For multi-parent hierarchies, define whether a descendant can appear through multiple paths and how rollups avoid double counting.

## Factless Fact Tables

A factless fact table records an event or relationship without a numeric measure. The row itself is evidence that something happened or was true.

### Event Fact: Class Attendance

**Grain:** one row per student, class session, and attendance outcome.

```sql
CREATE TABLE fact_class_attendance (
    student_sk       BIGINT NOT NULL,
    class_session_sk BIGINT NOT NULL,
    attendance_status_sk INTEGER NOT NULL,
    attendance_ts    TIMESTAMP,
    source_event_id  VARCHAR(100)
);
```

There may be no numeric measure. Attendance can be counted by counting rows, subject to the grain and duplicate policy.

### Coverage Fact: Promotion Eligibility

A factless fact can record which customers were eligible for a promotion on a given date. It supports questions such as:
- How many eligible customers redeemed the offer?
- Which eligible customers did not redeem?
- What percentage of eligible customers converted?

```text
fact_promotion_eligibility
  customer_sk
  promotion_sk
  eligibility_date_sk
  eligibility_reason_sk

fact_promotion_redemption
  customer_sk
  promotion_sk
  redemption_date_sk
  order_id
  discount_amount
```

To find eligible customers who did not redeem:

```sql
SELECT
    e.promotion_sk,
    COUNT(DISTINCT e.customer_sk) AS eligible_customers,
    COUNT(DISTINCT r.customer_sk) AS redeemed_customers
FROM fact_promotion_eligibility e
LEFT JOIN fact_promotion_redemption r
  ON e.customer_sk = r.customer_sk
 AND e.promotion_sk = r.promotion_sk
GROUP BY e.promotion_sk;
```

For a production conversion rate, define the eligibility cohort and redemption window. A customer eligible multiple times may need a unique eligibility episode key. Otherwise, one redemption can match multiple eligibility rows and distort counts.

### Event Fact vs. Coverage Fact

| Type | Purpose | Example |
|------|---------|---------|
| **Event-tracking fact** | Records that an event occurred | Student attendance, login, support interaction |
| **Coverage fact** | Records that an opportunity existed | Promotion eligibility, product-store availability |

Coverage facts are especially useful because absence of an event can be meaningful only if the opportunity was known to exist. You cannot reliably say "eligible customers who did not redeem" unless you have a trustworthy eligible population.

## Bridge vs. Factless Fact: Decision Table

| Question | Bridge Table | Factless Fact |
|----------|-------------|---------------|
| Primary purpose | Resolve a relationship | Record an event or condition |
| Typical grain | One row per relationship/version | One row per event or entity-opportunity |
| Numeric measures | May have allocation weight | Usually none |
| Example | Account-customer ownership | Student attended class |
| Temporal need | Often effective-dated | Event time or coverage interval |
| Main risk | Join fan-out | Duplicate events or incorrect absence inference |

A relationship bridge may itself be modelled as a factless fact in some dimensional designs, especially when the relationship is analytically meaningful. The distinction is about intended semantics and usage, not simply the presence of measures.

## Common Interview Questions

1. What is a bridge table?
2. What is a factless fact table?
3. What is the difference between a bridge and a factless fact?
4. How do you model many-to-many relationships in a star schema?
5. How do you prevent double counting after a bridge join?
6. What is an allocation bridge?
7. What is a coverage fact?
8. How do you find eligible customers who did not redeem?
9. How do you model effective-dated memberships?
10. Can a bridge table have measures?

**Follow-up / top-up questions:**
- Why is `SUM(DISTINCT amount)` not a safe fan-out fix?
- What if one customer has multiple eligibility episodes for the same promotion?
- How do you validate allocation percentages?
- What if a bridge relationship has no natural end date?
- How do you model a product assigned to multiple categories?
- How do you handle a hierarchy with multiple parent paths?
- How do you distinguish no event from missing event data?
- How would you build a factless fact for product-store availability?
- How do you model a relationship that changes retroactively?
- When would you use a bridge versus a separate transaction fact?

**Principal-level probes:**
- Design a reusable many-to-many modelling standard for an enterprise warehouse
- How do you make fan-out risks visible to BI users?
- How do you preserve attribution reproducibility when allocation rules change?
- How would you model eligibility and conversion for a campaign with repeated exposures?
- How do you govern effective-dated bridges across multiple domains?
- How do you test completeness when the expected population is itself changing?

## Data Quality and Performance Controls

**For bridges:**
- Uniqueness at relationship grain
- Validity interval non-overlap
- Allocation weights sum to expected total
- No orphan dimension keys
- Effective-dated join coverage
- Fan-out monitoring

**For factless facts:**
- Stable event/episode identity
- Duplicate detection
- Expected population completeness for coverage facts
- Event timestamp and timezone quality
- Clear distinction between no event, not eligible, and unknown
- Retention and privacy controls

**Performance techniques:**
- Partition large event facts by event date where appropriate
- Cluster/sort on frequently filtered or joined keys where supported
- Pre-aggregate common analysis paths
- Avoid exploding bridges into huge denormalized tables without a use case
- Maintain statistics and inspect query plans
- Test cardinality and skew on high-degree bridge keys

## Practice Exercise

Design a data warehouse model for a university:
- Students can enrol in multiple courses
- Courses have multiple instructors
- Students attend class sessions
- Students may be eligible for scholarships but may not receive them
- Course enrolment can be withdrawn and reinstated

Create:
1. A student-course bridge with temporal membership
2. A course-instructor bridge
3. A factless attendance event fact
4. A scholarship eligibility coverage fact
5. A scholarship award transaction fact

Explain the grain, keys, date handling, duplicate controls, and how you would calculate attendance rate and scholarship conversion without double counting.

**Evaluation rubric (10 points):**
- Correct bridge/factless distinction: 2
- Grain and temporal relationship design: 2
- Fan-out prevention: 2
- Coverage and absence semantics: 2
- Quality, governance, and performance: 2

## Key Takeaways

| Concept | Key Point |
|---------|-----------|
| Bridge tables | Resolve many-to-many or multi-valued relationships that a single FK cannot handle |
| Factless facts | Record events or opportunities without numeric measures — rows are evidence |
| Fan-out risk | A bridge join can multiply fact rows; aggregation semantics must be explicit |
| Allocation bridges | Distribute measures with business-approved weights; store the attribution model version |
| Coverage facts | Make absence analysis possible by defining the eligible population up front |
| Effective dating | Add valid_from/valid_to to bridges when relationships change over time |
| Distinct counts | Use `COUNT(DISTINCT event_id)` — not `SUM(DISTINCT amount)` — to prevent undercounting |
| Decision table | Distinguish bridge (resolve relationship) from factless fact (record occurrence) by purpose |

### Related Lessons
- [Lesson 10](./10-star-schema-vs-snowflake.md) — Star Schema vs Snowflake Schema
- [Lesson 22](./22-degenerate-and-junk-dimensions.md) — Degenerate and Junk Dimensions
- [Lesson 24](./24-fact-table-types.md) — Fact Table Types
- [Lesson 31](./31-advanced-dimension-design.md) — Advanced Dimension Design Techniques
