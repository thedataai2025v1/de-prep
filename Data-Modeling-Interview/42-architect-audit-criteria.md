# Lesson 42 — Architect Audit Criteria

**Level:** Principal / Architect
**Purpose:** Checklist for auditing a data model against enterprise-grade standards. Use this to evaluate any data model (your own or in an interview).

---

## 1. Core Principles Audit

| # | Criterion | Check |
|---|-----------|-------|
| 1 | **Data is a product** | Is there an owner, SLA, documentation, and consumer feedback loop? |
| 2 | **Source system boundaries** | Does Bronze have a 1:1 mapping to source systems? No cross-source merges in Bronze? |
| 3 | **Dimensional integrity** | Does every fact table have a clearly stated grain? No mixed grain? |
| 4 | **Conformed dimensions** | Are shared dimensions (date, customer, product) consistent across all marts? |
| 5 | **SDLC discipline** | Are changes versioned? Is there a migration plan for schema changes? |

**Pass:** 5/5. **Review:** 3-4/5. **Fail:** <3/5.

---

## 2. Advanced Architecture Patterns

### Multi-Mesh / Domain-Driven Design

| Aspect | What to Look For |
|--------|-----------------|
| **Domain boundaries** | Each domain (sales, marketing, support) has its own data product |
| **Conformed dimensions** | Shared dimensions are consumed, not duplicated, across domains |
| **Contract-driven** | Each domain publishes a schema contract; consumers depend on it |
| **Independent deployment** | A change in one domain doesn't break other domains |

### Event-Driven Architecture

| Aspect | What to Look For |
|--------|-----------------|
| **CDC from source** | Change Data Capture feeds Bronze with full history |
| **Idempotent MERGE** | Silver uses deterministic keys for exactly-once processing |
| **Late-arriving data** | Model handles events that arrive out of order |
| **Watermark-based processing** | Clear boundary between "current" and "historical" data |

---

## 3. Security & Governance

| Aspect | Check |
|--------|-------|
| **Row-level security** | Can access be restricted by region, org, or sensitivity? |
| **Column-level masking** | Are PII columns masked for non-privileged users? |
| **Audit trail** | Is there a log of who accessed what data and when? |
| **Data classification** | Are columns tagged as PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED? |
| **GDPR compliance** | Is there a process for right-to-be-forgotten requests? |

---

## 4. Performance & Cost Audit

### Query Performance Checklist

| Check | How to Verify |
|-------|---------------|
| **Partition filter required** | `require_partition_filter = TRUE` on large facts |
| **Cluster keys aligned** | Are cluster keys on the most common filter/join columns? |
| **Aggregate coverage** | Do the top 20 queries hit aggregates instead of raw facts? |
| **Non-algebraic MV handling** | Are non-algebraic views refreshed manually or rewritten? |
| **Skew detection** | Is there monitoring for data skew (e.g., one partition much larger than others)? |

### Cost Quantification

| Metric | How to Calculate |
|--------|-----------------|
| **Monthly scan cost** | `total_TB_scanned_per_month × cost_per_TB` |
| **Monthly storage cost** | `total_TB_stored × cost_per_TB_month` |
| **Compute cost per query** | `warehouse_size × query_duration × cost_per_second` |
| **Cost per dashboard user** | `total_monthly_cost / active_dashboard_users` |
| **Cost to serve** | `cost_per_query × queries_per_month` |

---

## 5. Data Quality & Testing

### Core DQ Checks (Every Pipeline)

| Check | Tool/Approach |
|-------|--------------|
| **Row count validation** | `COUNT(*)` matches source ± tolerance |
| **Null key check** | `COUNT(*) WHERE foreign_key IS NULL = 0` |
| **Duplicate detection** | `COUNT(*) - COUNT(DISTINCT primary_key) = 0` |
| **Value range check** | Numeric columns within expected bounds |
| **Freshness check** | `MAX(loaded_at)` within expected window |

### Pipeline Testing Strategy

```
Unit Tests → DQ checks on each Bronze/Silver/Gold table
Integration Tests → End-to-end flow with test data
Regression Tests → Schema change impact on downstream consumers
Performance Tests → Query latency on aggregates vs raw facts
```

---

## 6. Schema Evolution Strategy

### Breaking vs Non-Breaking Changes

| Change Type | Example | Impact | Strategy |
|-------------|---------|--------|----------|
| **Add column** | New metric in fact | Non-breaking | Add to schema, NULL for existing rows |
| **Drop column** | Remove unused metric | Breaking | Deprecate → warn → drop after 90 days |
| **Rename column** | Fix naming convention | Breaking | Create alias → migrate consumers → drop alias |
| **Change data type** | STRING → TIMESTAMP | Breaking | Dual-write → backfill → switch read path |
| **Change grain** | Add dimension to fact | Breaking | Create new table → deprecate old |

### Migration Playbook

1. **Announce** — Notify all consumers of the change with a timeline
2. **Dual-write** — Write to both old and new schema for overlap period
3. **Validate** — Run comparison queries between old and new
4. **Switch** — Redirect consumers to new schema
5. **Decommission** — Remove old schema after grace period

---

## 7. Anti-Patterns to Catch

| Anti-Pattern | Symptom | Fix |
|--------------|---------|-----|
| **God table** | One fact with 50+ columns, used for everything | Split by grain/use case |
| **Null-key dimension** | Placeholder row with `unknown` for missing keys | Add proper handling upstream |
| **Fat dimension** | Dimension with fact-like metrics | Move metrics to fact table |
| **Thin fact** | Fact with only keys, no measures | Consider a factless fact or bridge table |
| **Duplicate aggregations** | Multiple tables computing the same metric | Consolidate to one canonical aggregate |
| **Over-indexing** | Every column has a separate index | Index only high-cardinality filter/join columns |
| **Unbounded MERGE** | Full table scan on every load | Add `WHERE is_terminal_state = FALSE` |

---

## 8. Scorecard Summary

| Category | Weight | Max Score |
|----------|--------|-----------|
| Core Principles | 20% | 20 |
| Architecture Patterns | 15% | 15 |
| Security & Governance | 15% | 15 |
| Performance & Cost | 20% | 20 |
| Data Quality & Testing | 15% | 15 |
| Schema Evolution | 5% | 5 |
| Anti-Patterns | 10% | 10 |
| **Total** | **100%** | **100** |

**Rating:**
- **90-100:** Enterprise-grade, production-ready
- **70-89:** Good, with minor gaps to address
- **50-69:** Significant work needed
- **<50:** Needs redesign

---

## Related Lessons

- [Lesson 41 — Principal-Level Cheat Sheet](./41-principal-level-cheat-sheet.md)
- [Lesson 10 — Data Volume and Scalability](./10-data-volume-and-scalability-considerations.md)
- [Lesson 28 — Date-Based Partitioning](./28-date-based-partitioning.md)
- [Lesson 30 — Aggregates and Materialized Views](./30-aggregates-and-materialized-views.md)
