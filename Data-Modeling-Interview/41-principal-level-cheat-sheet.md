# Lesson 41 — Principal-Level Cheat Sheet

**Level:** Principal / Architect
**Purpose:** Quick-reference card for the most commonly asked advanced questions at principal-level interviews

---

## 1. Core Schema Decision Tree

```
Star vs Snowflake?
├─ Star → default for analytics warehouses (wide denormalized dims, flat facts)
└─ Snowflake → only if dimension values are reused across multiple fact tables
              AND the shared sub-dimension is < 5% of total dimension storage

Fact Grain?
├─ Transaction → one row per business event
├─ Periodic → one row per entity per period (e.g., daily user activity)
├─ Cumulative → one row per business process, updated as it progresses (e.g., order lifecycle)
└─ Accumulating snapshot → fixed columns for milestones, updated in place (e.g., shipped_at, delivered_at)
```

---

## 2. Key Interview Questions & Principal Answers

### "Explain Kimball vs Data Vault"

**Kimball (Dimensional Modeling):**
- Bottom-up, business-driven, incremental delivery
- Star schemas with conformed dimensions across data marts
- Business-friendly grain, denormalized dimensions
- Best for analytics, BI, dashboards

**Data Vault 2.0:**
- Top-down, source-driven, parallel load
- Hubs (business keys), Links (relationships), Satellites (attributes + history)
- Survivable architecture: handles source system changes without re-modeling
- Best for enterprise data warehouses with many changing source systems

**Principal Answer:**
> "I start with Kimball for the gold/consumption layer — business users need star schemas. If the source landscape is complex (10+ systems, frequent schema changes), I use Data Vault as the silver layer for auditability and survivability, then transform to Kimball star schemas for the gold layer. The two are complementary, not competing."

---

### "What are the Top 10 Red Flags in a Data Model?"

1. **No explicit grain definition** — every fact table must state its grain
2. **Mixed grain in one table** — e.g., order header + line items in the same table
3. **No surrogate keys** — using source system keys that can change
4. **Missing SCD handling** — dimensions change but model doesn't track history
5. **Non-conformed dimensions** — the same entity has different definitions across marts
6. **Missing slow query optimization** — no aggregates, no partitioning on large facts
7. **No data quality checks** — no row count validation, no null checks on keys
8. **Over-normalized facts** — facts that are actually dimensions disguised as facts
9. **No cost controls** — missing partition filters, unbounded full scans
10. **No documentation** — no data dictionary, no lineage, no business glossary

---

### "Top 1 Aggregate Table to Make"

For any platform, the most impactful aggregate is:

```
agg_<entity>_daily
- entity_key, date_key
- core_metrics (revenue, count, conversion, etc.)
```

**Why?**
- 90% of dashboard queries filter by entity + date range
- Reduces scan from billions of rows to millions
- Can be refreshed incrementally with partition-overwrite
- Quantifiable savings: if raw table is 10B rows and aggregate is 10M rows → 99.9% scan reduction

---

### "Non-Algebraic Materialized Views — The Pitfalls"

**What is non-algebraic?**
- Uses `DISTINCT`, `QUALIFY`, `RANK()`, or window functions
- Cannot be automatically refreshed by the warehouse engine

**Practical Impact:**
- `RANK() OVER(PARTITION BY date_key ORDER BY revenue DESC)` → **non-algebraic**
- `SUM(revenue) GROUP BY date_key` → **algebraic** (can be auto-refreshed)

**Workarounds (ranked):**
1. **Rewrite as algebraic equivalent** — e.g., replace `RANK() = 1` with `revenue = MAX(revenue)`
2. **Two-layer refresh** — materialize the algebraic aggregation, then apply `RANK()` on top
3. **Manual refresh** — schedule a full refresh via cron (acceptable for low-frequency queries)
4. **Accept no refresh** — use `REPLICA` for low-volume non-algebraic views

**Cost of ignoring:**
- A non-algebraic MV with `INCREMENTAL` refresh policy silently returns stale data
- Example: a top-10-vendors view cached at 3 AM shows yesterday's ranking all day
- Fix: rewrite as algebraic + apply ranking in the BI tool or application layer

---

## 3. FinOps Quantification Framework

**How to prove the cost of a data model decision:**

| Decision | Metric | Impact Formula |
|----------|--------|----------------|
| Partitioning | Scan rows reduced | `(rows_without_partition - rows_with_partition) × cost_per_TB_scanned` |
| Aggregates | Query time reduced | `(query_time_raw - query_time_agg) × compute_cost_per_second` |
| SCD Type 2 | Storage increase | `(version_count_per_entity - 1) × entity_count × row_size` |
| Retention policy | Storage reduced | `(days_retained - days_needed) × daily_ingest × cost_per_TB_month` |

**Example:** A non-partitioned fact table at 10B rows scanned 100×/day at $6/TB:
- Without partition: $600/day in scan cost
- With daily partition + filter: $6/day → **$216,000/year savings**

---

## 4. GDPR & Compliance — Three Patterns

| Pattern | How | When |
|---------|-----|------|
| **Hard delete** | `DELETE FROM fact WHERE user_key = X` | Full right-to-be-forgotten request |
| **Anonymize** | Set PII columns to NULL, keep `user_id` | Retain analytics while removing personal data |
| **Expire snapshots** | Warehouse retention policy + `CLUSTER BY` | Automatic cleanup of old data versions |

**Audit trail:** Always maintain a deletion log (`deletion_log(user_id, deleted_at, reason)`) for compliance proof.

---

## 5. Source System Modeling Patterns

| Pattern | Source → Silver | Silver → Gold |
|---------|----------------|---------------|
| **Transactional** | `MERGE` on `event_id` with `is_terminal_state` | Conformed dimensions + star schema |
| **Event streaming** | CDC → Kafka → Bronze (append) | Dedup + SCD merge in Silver |
| **Batch file** | Staging table → `MERGE` on natural key | Same as transactional |
| **API-based** | Staging table with `extracted_at` | Same as batch |

**Golden Rule:** Every source gets its own Bronze table. Never merge two sources in Bronze.

---

## 6. Interview Answer Framework (STAR-Plus)

1. **Situation** — "The business needed X with Y constraints..."
2. **Task** — "My role was to design Z..."
3. **Action** — "I chose [approach] because [reason]. I considered [alternative] but rejected it because [tradeoff]."
4. **Result** — "This delivered [quantifiable outcome]: [metric] improved by [number]."
5. **Plus** — "I documented the decision in an ADR. The model was audited against [criteria]. A key lesson: [retrospective insight]."

---

## Related Lessons

- [Lesson 10 — Data Volume and Scalability](./10-data-volume-and-scalability-considerations.md)
- [Lesson 28 — Date-Based Partitioning](./28-date-based-partitioning.md)
- [Lesson 30 — Aggregates and Materialized Views](./30-aggregates-and-materialized-views.md)
- [Lesson 42 — Architect Audit Criteria](./42-architect-audit-criteria.md)
