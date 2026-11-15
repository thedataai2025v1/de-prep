# Lesson 4.3 — Advanced Dimension Design Techniques

## 1. Why advanced patterns exist

A simple star schema works well when entities have stable attributes, one-to-many hierarchies, and manageable change rates. Real businesses are messier:
- Customers have hundreds of behavioural attributes.
- Products belong to multiple categories.
- Accounts have multiple owners.
- A profile attribute changes far more often than the rest of the dimension.
- The same entity plays different roles in a process.
- Analysts need both current and historical representations.

Advanced dimension patterns help preserve analytical usability without turning the model into an ungoverned maze of tables.

### Learning outcomes
- Choose between a wide dimension, mini-dimension, and outrigger.
- Model multi-valued attributes with bridge tables.
- Design hierarchy bridges and role-playing dimensions.
- Handle rapidly changing attributes and high-cardinality identifiers.
- Explain performance, governance, and temporal trade-offs.

---

## 2. Mini-dimensions for rapidly changing attributes

A mini-dimension separates a group of attributes that change frequently from a larger, slower-changing dimension.

Example: a customer dimension contains stable profile information such as birth date band, acquisition channel, and original signup date. A behavioural profile may change frequently:
- Engagement band
- Credit-risk band
- Affinity segment
- Spending band

Instead of creating a new Type 2 customer row for every behavioural change, create a `dim_customer_profile` mini-dimension. The fact can reference both the durable customer identity and the profile that applied at event time.

```text
fact_order
  order_date_sk
  customer_sk
  customer_profile_sk
  net_amount

dim_customer
  customer_sk
  customer_bk
  signup_date
  acquisition_channel

dim_customer_profile
  customer_profile_sk
  engagement_band
  risk_band
  spend_band
  valid_from
  valid_to
```

The profile key can represent a combination of banded attributes. This is useful when many customers share the same profile combination.

**Benefits**
- Prevents the main customer dimension from growing with every volatile attribute update.
- Makes profile-based segmentation efficient.
- Separates stable identity from changing classification.

**Costs and risks**
- Adds another join and another key-resolution process.
- Profile combinations can still become high-cardinality.
- Facts must capture the correct profile at the event time.
- Attribute definitions and bands require governance.

Do not create a mini-dimension for every changing column. Group attributes that change at similar rates and are commonly analysed together.

## 3. Outrigger dimensions

An outrigger is a dimension referenced by another dimension rather than directly by the fact. For example, `dim_customer` may reference `dim_geography` for detailed geographic attributes.

```text
fact_order -> dim_customer -> dim_geography
```

This is a snowflaked portion of a dimensional model. It can reduce duplicated descriptive data or centralize a shared hierarchy, but it introduces extra joins and can make the model less intuitive for BI users.

Use an outrigger when:
- The referenced entity has independent meaning and lifecycle.
- It is shared across dimensions.
- Its attributes are large or separately governed.
- The platform and semantic layer can handle the additional join cleanly.

Prefer a flattened dimension when simplicity and predictable query performance matter more than normalization. The decision should be based on actual consumer patterns and platform behaviour, not a blanket rule that snowflaking is always bad.

## 4. Bridge tables for multi-valued dimensions

A bridge table resolves a many-to-many relationship between a dimension and another entity.

Example: one customer can have multiple loyalty programmes, and one programme has many customers.

```text
dim_customer
  customer_sk

dim_loyalty_program
  program_sk

bridge_customer_program
  customer_sk
  program_sk
  valid_from
  valid_to
  allocation_pct
```

A fact joined through this bridge can multiply rows. If a customer belongs to three programmes, a sale may appear three times unless the query applies a defined allocation or distinct-count strategy.

### Allocation weights

If revenue must be allocated across programmes, store a governed allocation percentage. Validate that weights sum to 1 (or 100%) for each applicable customer and validity interval.

```sql
SELECT
    p.program_name,
    SUM(f.net_amount * b.allocation_pct) AS allocated_net_sales
FROM fact_order f
JOIN bridge_customer_program b
  ON f.customer_sk = b.customer_sk
 AND f.order_ts >= b.valid_from
 AND f.order_ts <  b.valid_to
JOIN dim_loyalty_program p
  ON b.program_sk = p.program_sk
GROUP BY p.program_name;
```

If no allocation rule exists, do not imply that summing the repeated measure across programmes produces a valid company total. Provide a distinct-count or membership analysis instead.

## 5. Hierarchy bridge tables

A hierarchy bridge represents parent-child relationships, especially when hierarchies are ragged, recursive, or many-to-many.

Example: an employee hierarchy where an employee may report through several matrix-management paths, or a product can appear in multiple category paths.

A closure-table style bridge may contain:

```text
bridge_org_hierarchy
  ancestor_employee_sk
  descendant_employee_sk
  hierarchy_depth
  valid_from
  valid_to
```

A row can represent each ancestor-descendant pair. `hierarchy_depth = 0` may represent the employee itself, depending on convention.

Advantages:
- Simplifies recursive rollups at query time.
- Supports variable-depth hierarchies.
- Can represent multiple paths when designed carefully.

Risks:
- The bridge can grow substantially.
- Updates to parent-child relationships may require many bridge changes.
- Multiple paths can double-count descendants unless the reporting rule is explicit.
- Historical hierarchy reporting requires effective dating.

For a strict tree, a parent key on the dimension may be enough. Use a bridge when query patterns or hierarchy complexity justify it.

## 6. Role-playing dimensions and role semantics

A single date dimension can support order date, ship date, delivery date, and cancellation date. The physical table is shared, but each foreign key has a distinct business role.

Similarly, an employee dimension may play the roles of sales representative, approver, and account manager. Role names should be explicit in the fact schema and semantic layer.

A common mistake is to reuse a dimension merely because two columns contain the same data type. Reuse is valid only when the entity and semantics match.

## 7. Junk dimensions and degenerate dimensions — when to use them

### Junk dimension

Combine several low-cardinality, independent flags into a compact dimension. Example: `is_expedited`, `is_gift`, and `is_subscription_order`.

Do not place attributes together simply because they are small. Their lifecycle, ownership, and usage should make the grouping sensible.

### Degenerate dimension

Keep an operational identifier such as `order_number` or `case_number` in the fact table when it has no meaningful descriptive dimension of its own.

A degenerate identifier is useful for drill-through and reconciliation. It is not a measure and should not be summed.

## 8. Rapidly changing dimensions and high-cardinality attributes

A classic Type 2 row for every change may be costly when attributes change frequently. Options include:
- Mini-dimension for banded or grouped attributes.
- Separate event/history fact for every attribute change.
- Snapshot fact for periodic state analysis.
- Current-state operational table plus an analytical history table.
- Type 1 for attributes where historical values are not required.

High-cardinality attributes such as free-form URLs, device fingerprints, or raw user-agent strings may be poor dimension columns. Consider:
- Keeping the raw value in a lower-level event store.
- Normalizing into useful derived categories.
- Tokenizing or hashing identifiers when appropriate.
- Using a separate reference table only if it provides reusable descriptive value.

Do not use hashing as a substitute for privacy controls. Hashes can still be personal or linkable data.

## 9. Temporal and privacy considerations

Advanced dimensions often hold sensitive attributes: customer address, risk band, health-related category, or identity links. Apply:
- Data minimization: store only what the analytical use case needs.
- Purpose and access controls.
- Retention and deletion policies.
- Masking or tokenization where suitable.
- Audit trails for sensitive changes.
- Careful handling of “right to erasure” and legally required retention.

For temporal accuracy, distinguish:
- **Valid time:** when the attribute was true in the business.
- **System time:** when the platform recorded the value.

A bitemporal structure may be required when a correction is received after the fact and the organization must answer both “what do we now believe was true then?” and “what did we believe at that time?”

## 10. Pattern selection guide

| Requirement | Candidate pattern | Watch out for |
|---|---|---|
| Frequently changing profile bands | Mini-dimension | Profile combination cardinality |
| Shared separately governed reference entity | Outrigger | Extra joins and BI complexity |
| One entity has many memberships | Bridge table | Measure fan-out and allocation |
| Recursive or ragged hierarchy | Hierarchy bridge | Bridge growth and multiple paths |
| Same entity used in multiple process roles | Role-playing dimension | Role semantics must be explicit |
| Several low-cardinality flags | Junk dimension | Avoid unrelated attribute bundling |
| Identifier with no descriptive attributes | Degenerate dimension | Do not treat as a measure |
| Very high-frequency attribute changes | Event/history or snapshot model | Temporal query complexity |

## 11. Worked example: customer analytics platform

A retailer needs to analyse orders by:
- Customer's stable acquisition channel.
- Customer's engagement band at order time.
- Loyalty programmes the customer belonged to at order time.
- Current customer profile for operational dashboards.

A possible model:
1. `dim_customer` stores durable identity and stable attributes.
2. `dim_customer_profile` stores combinations of engagement/risk/spend bands.
3. `bridge_customer_program` stores dated programme memberships and allocation weights if financial attribution is required.
4. `fact_order` stores `customer_sk`, `customer_profile_sk`, event timestamp, and measures.
5. A current-state view exposes the latest profile for operational dashboards.

Important: membership joins can multiply order rows. Revenue attribution requires an approved allocation rule. Engagement-band history requires the profile key resolved at order event time.

## 12. Interview conversation

**Interviewer:** A customer has 150 behavioural attributes, and many change daily. Would you put all of them in a Type 2 customer dimension?

**Candidate:** Not automatically. I would classify attributes by analytical purpose, change frequency, and historical requirement. Stable identity belongs in the durable customer dimension. Frequently changing banded attributes could be grouped into a mini-dimension, while raw behavioural changes may be better represented as events or periodic snapshots.

**Interviewer:** What if a customer belongs to multiple loyalty programmes?

**Candidate:** That is a many-to-many relationship, so I would model it with a bridge containing customer key, programme key, and effective dates. If we need to allocate revenue across programmes, we need a business-approved allocation rule and weights. Otherwise, a join can multiply revenue and produce misleading totals.

**Interviewer:** Why not snowflake every dimension to reduce duplication?

**Candidate:** Normalization reduces repeated attributes, but it increases joins and complexity for analysts. I would snowflake only where the referenced entity has independent lifecycle or governance value, or where duplication is genuinely problematic. I would test query patterns and the BI semantic layer before standardizing the pattern.

**Interviewer:** How would you represent an organization hierarchy that changes over time?

**Candidate:** If it is a simple tree and consumers need only current parent relationships, a parent key may suffice. For historical rollups, variable depth, or multiple reporting paths, I would use an effective-dated hierarchy bridge. I would explicitly define how multiple paths affect aggregation to prevent double counting.

## 13. Interview questions and follow-ups

### Core questions
1. What is a mini-dimension, and when is it useful?
2. What is an outrigger dimension?
3. How do you model a many-to-many relationship in a star schema?
4. What is a hierarchy bridge?
5. When should you use a junk dimension?
6. What is the difference between a degenerate dimension and a regular dimension?
7. What is a role-playing dimension?
8. How do you prevent double counting when joining through a bridge?

### Follow-up / “top-up” questions
- How do you design allocation weights for multi-valued memberships?
- What if bridge membership changes over time?
- How do you handle a hierarchy with multiple parents?
- When does a mini-dimension become too large?
- How do you model a customer profile with thousands of possible attribute combinations?
- Would you store raw device IDs in a dimension?
- How do you handle privacy deletion in historical dimensional data?
- How would you optimize a bridge table with billions of rows?
- When would a snapshot fact be better than a Type 2 dimension?

### Principal-level probes
- Define a reusable enterprise pattern for multi-valued dimensions.
- How would you govern shared bridges across domains?
- What controls prevent bridge joins from silently inflating metrics?
- How would you evolve a hierarchy model while maintaining historical reporting?
- How do you balance analyst usability, storage cost, query performance, and semantic correctness?

## 14. Practice exercise

Design the dimensions for a video-streaming platform:
- A household may have multiple profiles.
- A profile can belong to multiple content-preference segments over time.
- A title may be assigned to multiple genres.
- Viewer engagement bands change frequently.
- Content can be available in different countries and licensing windows.

Explain:
1. Which entities deserve separate dimensions?
2. Where would you use a bridge table?
3. Would you use a mini-dimension for engagement bands?
4. How would you represent effective-dated title availability by country?
5. How would you prevent viewing minutes from being double-counted across genres?

**Evaluation rubric (10 points):**
- Correct entity and grain boundaries: 2
- Appropriate advanced pattern selection: 2
- Temporal validity: 2
- Fan-out/double-counting controls: 2
- Governance, privacy, and performance trade-offs: 2

## 15. Key takeaways

- Mini-dimensions isolate rapidly changing analytical profiles from durable identity.
- Outriggers centralize separately governed entities but add joins.
- Bridges model many-to-many relationships; measures need explicit fan-out and allocation semantics.
- Hierarchy bridges support recursive, ragged, and historical rollups.
- Advanced patterns should solve a demonstrated business or scale problem, not add abstraction for its own sake.
