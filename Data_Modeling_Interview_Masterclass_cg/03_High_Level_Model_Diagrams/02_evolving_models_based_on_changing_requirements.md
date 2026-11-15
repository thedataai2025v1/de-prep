# Lesson 2: Evolving Models Based on Changing Requirements

## Learning objectives
- Evolve a model without losing semantic clarity.
- Distinguish additive change from breaking change.
- Manage schema evolution, history, and downstream compatibility.
- Explain migration and governance choices at Principal level.

## 1. Change is normal

Requirements change because products, regulations, business processes, and analytical questions evolve. A robust model should accommodate change without becoming an uncontrolled collection of nullable columns and special cases.

Common changes:
- A new business process or product type.
- New attributes or dimensions.
- A relationship changing from one-to-one to one-to-many.
- A metric definition changing.
- New historical or audit requirements.
- A source system being replaced.
- A new region, currency, or legal entity.

## 2. Classify the change

| Change | Typical impact | Questions |
|---|---|---|
| Add optional attribute | Often additive | Is it meaningful for all entity types? |
| Add new event type | Process/model extension | Does it have a distinct grain? |
| Change relationship cardinality | Potentially breaking | Do consumers assume one-to-one? |
| Rename or remove field | Breaking contract | What consumers depend on it? |
| Change metric definition | Semantic breaking change | Is historical restatement needed? |
| Change grain | Major redesign | Can old and new facts be reconciled? |

A technically backward-compatible schema change can still be semantically breaking. For example, changing the meaning of `revenue` while retaining its name is dangerous.

## 3. Use stable business concepts and explicit versions

For important data products:
- Document the grain and business definitions.
- Version contracts when semantics or required fields change.
- Deprecate old fields through a communicated transition period.
- Keep lineage and ownership visible.
- Publish migration examples and validation checks.
- Avoid silent reinterpretation of historical records.

## 4. Example: subscription product expansion

Initial model: one customer has one subscription plan.

New requirement: a customer can have multiple subscriptions, add-ons, and seat-based licenses.

Do not simply add `plan_2`, `plan_3`, and `plan_4` columns. Revisit the relationship:
- Customer to subscription: one-to-many.
- Subscription to subscription item: one-to-many.
- Subscription item to product/plan: many-to-one.
- Subscription lifecycle events: separate event history if required.

This supports variable product composition and avoids a fixed-width design.

## 5. Preserve historical meaning

When a new attribute or relationship is introduced, decide:
- Is it known only from now onward?
- Can historical values be reconstructed?
- Should unknown history remain null/unknown?
- Does the new definition require backfill?
- Should prior metrics be restated?
- How are old and new versions exposed to consumers?

Never fabricate historical values simply to make a column appear complete.

## 6. Migration strategy

A safe migration may follow:
1. Discover impacted consumers using lineage and query logs.
2. Define target semantics and migration mapping.
3. Add new structures alongside old ones.
4. Backfill or dual-write where appropriate.
5. Validate counts, totals, and edge cases.
6. Run old and new models in parallel.
7. Migrate consumers in controlled waves.
8. Monitor adoption and correctness.
9. Deprecate and remove the old contract after agreement.

## 7. Interviewer–candidate conversation

**Interviewer:** The business changes from one subscription per customer to multiple subscriptions. What do you do?

**Candidate:** I'd treat this as a cardinality and potentially grain change, not just a column addition. I'd model subscription as its own entity with a customer foreign key and represent plan components as subscription items if needed.

**Interviewer:** Can we update the existing table in place?

**Candidate:** Possibly, but first I'd identify downstream assumptions. If consumers expect one row per customer, changing it to one row per subscription can multiply records and break metrics. I'd publish a new contract or view, migrate consumers, and validate the transition.

**Interviewer:** What about historical subscriptions?

**Candidate:** I'd determine whether the source retains history and whether we can reconstruct it. I'd distinguish genuine unknown history from values that can be backfilled, and document the effective date of the new model.

## 8. Common mistakes
- Treating all changes as harmless schema additions.
- Changing grain without notifying consumers.
- Reusing a field name after changing its meaning.
- Backfilling invented history.
- Ignoring downstream lineage and contract tests.

## 9. Practice questions
1. How would you evolve a one-to-one relationship into one-to-many?
2. What is a breaking data-contract change?
3. How would you migrate consumers from an old fact grain to a new one?
4. When should a metric definition be versioned?
5. How do you decide whether to backfill historical data?

## 10. Principal-level follow-ups
- How do you establish enterprise data-contract governance?
- How do you coordinate schema evolution across independently deployed teams?
- How do you prevent a central model from becoming a bottleneck?
- How do you measure migration readiness and consumer adoption?

## Revision checklist
- [ ] I classify additive, breaking, and semantic changes.
- [ ] I identify downstream consumers.
- [ ] I plan parallel migration and validation.
- [ ] I preserve historical truth and contract versions.

## Key takeaway
**Evolve the model deliberately: preserve meaning, protect consumers, and make semantic changes explicit.**
