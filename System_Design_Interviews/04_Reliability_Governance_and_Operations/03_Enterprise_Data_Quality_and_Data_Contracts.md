# Scenario 3 — Design an Enterprise Data Quality and Data Contracts Platform

**Interview level:** Staff / Principal Data Engineer  
**Suggested interview time:** 45–60 minutes  
**Core themes:** data contracts, schema compatibility, quality dimensions, quarantine, reconciliation, ownership, trust

## 1. Problem statement

An enterprise has hundreds of producers and consumers. Data defects are discovered late in dashboards and ML models. Producers change schemas without warning, consumers interpret fields differently, and data quality checks are inconsistent across teams.

Design a platform and operating model that establishes data contracts, validates quality at appropriate boundaries, prevents invalid data from silently propagating, and supports remediation without blocking every pipeline for every warning.

Assume 250 producer teams, 700 consumer teams, 2,000 important datasets, and both batch and streaming delivery.

## 2. Clarifying questions

1. Which datasets are business-critical or regulated?
2. Who owns schema, semantics, and quality expectations?
3. Are producers and consumers in the same organization or external?
4. What changes are backward-compatible?
5. Which checks can run at ingestion, transformation, publication, and consumption?
6. What is the acceptable response to bad data: reject, quarantine, warn, or publish with status?
7. How are late-arriving, duplicate, corrected, and deleted records handled?
8. Can quality be measured against source control totals or only internal invariants?
9. What is the cost of false positives blocking valid data?
10. How are contract changes communicated and tested?

## 3. Requirements

### Functional
- Versioned data contracts for schema, semantics, keys, freshness, and quality.
- Schema compatibility checks in CI/CD and at runtime.
- Reusable quality rules and domain-specific assertions.
- Validation at ingestion, transformation, and publication boundaries.
- Quarantine and replay for invalid records.
- Source-to-target reconciliation and quality scorecards.
- Ownership, alert routing, remediation workflows, and audit history.
- Consumer impact analysis for breaking changes.

### Non-functional
- Low validation overhead on high-throughput streams.
- Consistent rule semantics across batch and streaming.
- Configurable severity and fail-open/fail-closed behaviour.
- Scalable metadata and test execution.
- Explainable quality status to downstream consumers.
- Safe rule evolution and exemption governance.

## 4. Data contract model

A contract is more than a schema. Include:
- dataset identity and owner,
- schema and field descriptions,
- data types, nullability, and allowed values,
- primary/business keys and grain,
- event-time and timezone semantics,
- update/delete and deduplication behaviour,
- freshness and delivery expectations,
- quality invariants and thresholds,
- compatibility policy,
- classification and access constraints,
- retention and change notification,
- producer and consumer responsibilities.

Separate structural compatibility from semantic compatibility. Changing a `decimal` field from dollars to cents is a breaking change even if the data type is unchanged.

## 5. Quality dimensions

| Dimension | Example check | Typical response |
|---|---|---|
| Completeness | Required fields / expected partitions present | Block or quarantine |
| Uniqueness | Business key unique within defined grain | Deduplicate or quarantine |
| Validity | Values within domain/range | Reject invalid rows |
| Consistency | Cross-field/business invariants | Hold publication |
| Accuracy | Reconcile with source/golden truth | Investigate and correct |
| Timeliness | Data available within freshness contract | Alert or mark stale |
| Referential integrity | Foreign keys resolve | Defer, quarantine, or repair |
| Distribution | Unexpected null/volume/value shifts | Warn or investigate |

A quality score alone can hide critical failures. Keep individual critical checks visible and weight according to business risk.

## 6. High-level architecture

```mermaid
flowchart TB
  PROD[Producer code / schema] --> CI[Contract CI checks]
  CI --> REG[Contract registry]
  PROD --> ING[Ingestion gateway]
  REG --> VAL[Runtime validation]
  ING --> VAL
  VAL -->|valid| RAW[(Raw / accepted data)]
  VAL -->|invalid| QUAR[(Quarantine / DLQ)]
  RAW --> TRANS[Transformation]
  REG --> TRANS
  TRANS --> PUB[Publication quality gate]
  PUB --> CUR[(Certified data products)]
  QUAR --> FIX[Remediation / replay workflow]
  FIX --> ING
  CUR --> CONS[Consumers]
  REG --> IMP[Lineage / impact analysis]
  IMP --> CONS
  MET[Quality metrics / reconciliation] <-- VAL
  MET <-- PUB
  MET --> ALERT[Owner alerts / scorecards]
```

## 7. Validation placement

### Producer-side / CI
Catch breaking schema changes, missing descriptions, invalid types, and contract violations before deployment. Validate compatibility against registered versions and known consumers.

### Ingestion boundary
Validate envelope, required fields, schema version, key presence, event timestamp, and basic domain constraints. Keep high-throughput checks efficient. Route invalid records to quarantine with reason and original payload reference.

### Transformation boundary
Validate business rules, joins, aggregations, uniqueness, and referential integrity. Use deterministic tests and representative fixtures.

### Publication boundary
Apply critical completeness, reconciliation, and correctness checks before certifying/publishing. Distinguish “produced” from “certified.”

### Consumer boundary
Expose quality/freshness status and allow critical consumers to require certified data. Consumers should not each reimplement every upstream validation.

## 8. Schema evolution and compatibility

Define compatibility modes:
- backward compatible: new consumer can read old data;
- forward compatible: old consumer can read new data;
- full compatible: both directions;
- breaking: requires versioning or coordinated migration.

The exact compatibility meaning depends on the serialization format and registry. Validate using the actual schema registry and client libraries. Schema compatibility does not guarantee semantic compatibility.

A safe change process:
1. Propose change and update contract.
2. Run compatibility checks.
3. Identify consumers through lineage/registry.
4. Publish migration plan and deprecation date.
5. Support dual-read or dual-write if needed.
6. Migrate consumers and observe adoption.
7. Remove old field/version only after sign-off.

## 9. Quarantine and remediation

A quarantine record should preserve:
- source identity and original event/record,
- schema version,
- validation rule and failure reason,
- event/ingestion timestamps,
- pipeline run and code version,
- retry count and remediation status.

Do not silently drop invalid records. Do not endlessly retry deterministic bad data. Separate transient failures (network, dependency) from deterministic validation failures.

Replay must be idempotent and preserve ordering where required. Revalidate against the intended contract version and record whether repaired data was backfilled into published outputs.

## 10. Reconciliation strategy

Use layered reconciliation:
- source-to-landing counts and offsets,
- landing-to-curated counts and key coverage,
- aggregate control totals,
- partition-level row/byte counts,
- sampled record-level comparisons,
- downstream business totals.

A row-count match is not enough to prove correctness. For financial data, reconcile amounts and business identifiers. For event streams, reconcile source positions and event IDs. Define tolerance and explain why it is acceptable.

## 11. Quality policy and severity

Use severity classes:
- **Critical:** risk of financial misstatement, regulatory breach, privacy issue, or material business harm. Block publication and page.
- **High:** material completeness/correctness degradation. Quarantine affected scope and notify owner.
- **Medium:** quality regression with limited impact. Publish with warning only if contract allows.
- **Informational:** trend or nonblocking observation.

Fail-open versus fail-closed must be explicit per product and check. A missing optional descriptive field may not block; a duplicated financial transaction key likely should.

## 12. Failure modes

| Failure | Mitigation |
|---|---|
| Producer deploys breaking schema | CI compatibility gate and versioned rollout |
| Semantic meaning changes silently | Contract review and semantic versioning |
| False-positive quality check | Calibrated thresholds, sampling, owner review |
| Quarantine backlog grows | Backlog SLO, ownership, remediation queue |
| Replay duplicates records | Idempotency keys and deterministic merge |
| Quality service unavailable | Local schema cache; risk-based fail-open/closed |
| Source counts mismatch | Offset/key/control-total reconciliation |
| Consumer ignores quality status | Certified product interface and governance |
| Rules drift across teams | Shared rule library and contract registry |

## 13. Observability

Track by dataset, producer, consumer, rule, and severity:
- records validated/accepted/rejected,
- quarantine volume and oldest age,
- schema compatibility failures,
- quality pass rate by critical rule,
- freshness and completeness,
- reconciliation deltas,
- rule execution latency/cost,
- producer and consumer adoption of contract versions,
- mean time to detect and remediate.

Avoid unbounded labels such as record ID. Store detailed failure samples separately with controlled access.

## 14. Trade-offs

| Decision | Option A | Option B | Trade-off |
|---|---|---|---|
| Validate all rows | Stronger detection | Higher compute/latency | Use for critical invariants |
| Sample validation | Lower cost | May miss rare defects | Risk-based sampling |
| Fail closed | Prevent bad publication | Can block valid data | Critical correctness/privacy |
| Fail open | Higher continuity | Defects may propagate | Low-risk optional checks |
| Central rules | Consistency | Domain semantics may be lost | Shared framework, domain ownership |
| Strict schema | Predictability | Producer friction | Compatibility policy and migration support |

## 15. Sample interviewer–candidate conversation

**Interviewer:** A producer adds a nullable field. Is that safe?

**Candidate:** Structurally it is often backward-compatible, but I would check the serialization format's compatibility mode and the consumers' behaviour. I would also confirm whether the field changes any semantic assumptions, and whether schema registration and deployment order are safe.

**Interviewer:** The field type stays the same, but its unit changes from dollars to cents.

**Candidate:** That is a semantic breaking change. Schema compatibility will not catch it. The contract needs units and meaning, and the migration should use a new field or version with dual-read/dual-write and consumer migration.

**Interviewer:** A quality rule says 2% of rows have null customer IDs. Do you stop the pipeline?

**Candidate:** I would ask whether customer ID is mandatory for this product and what those rows represent. If it is a critical key, quarantine or block the affected scope. If it is legitimately absent for a documented event type, the rule is too broad and needs conditional semantics. I would not choose a threshold without understanding the business invariant.

**Interviewer:** The source is late. Should downstream tables publish yesterday's data?

**Candidate:** It depends on the product contract. I would preserve the last certified snapshot and mark it stale, rather than publishing a partial or misleading dataset. Consumers need an explicit freshness status and a defined stale-data policy.

**Interviewer:** How do you avoid every team writing its own checks?

**Candidate:** Provide a shared contract and rule framework, common libraries, and CI templates. But domain teams should own business definitions and thresholds. Central platform owns execution, metadata, alerting, and enforcement mechanisms.

## 16. Principal-level follow-up questions

**Q1. What is the difference between schema registry and data contract?**  
A schema registry manages structural schemas and compatibility. A data contract also captures semantics, ownership, keys, quality, freshness, and operational expectations.

**Q2. How do you handle a breaking change with 80 downstream consumers?**  
Use lineage to identify consumers, communicate, publish a new version, provide migration guidance, measure adoption, and retire only after agreed sign-off. Prioritize critical consumers and provide compatibility windows.

**Q3. How do you avoid blocking a stream because one record is malformed?**  
Use bounded validation, quarantine malformed records, preserve ordering requirements, and continue only where business semantics allow. Critical key or security violations may require partition or product-level hold.

**Q4. How do you prevent duplicate replay?**  
Use stable event/business keys, idempotent sink operations, deterministic batch identifiers, and offset/checkpoint coordination. Reconcile after replay.

**Q5. How do you measure accuracy?**  
Accuracy requires an independent source of truth, golden sample, or domain invariant. If none exists, state that the platform measures plausibility/consistency rather than true accuracy.

**Q6. Who owns quality: producer or platform?**  
Producer owns meaning and source correctness; platform provides validation, observability, and enforcement; consumer owns appropriate use and feedback. Ownership must be explicit in the contract.

**Q7. What happens when the contract service is unavailable?**  
Cache known contract versions for bounded periods. For high-risk datasets, fail closed if contract cannot be verified; for low-risk streams, use a cached compatible schema and emit a warning. Define this per product.

**Q8. How do you prioritize quality investment?**  
By business impact, consumer count, regulatory exposure, defect history, and cost of late detection. Focus first on high-impact contracts and high-leverage shared checks.

## 17. Common interview mistakes

- Treating schema validation as complete data quality.
- Using row counts as the only reconciliation.
- Blocking all data for every warning.
- Dropping malformed records without quarantine.
- Assuming nullable field additions are always safe.
- Ignoring semantic changes and units.
- Making platform team responsible for business meaning.
- Defining thresholds without understanding the data distribution.

## 18. Practice exercise

Design contracts for:
1. CDC customer table with deletes and schema evolution.
2. Financial transaction stream with duplicate/replay risk.
3. Marketing attribution table with late-arriving events.

For each, define schema, keys, event-time semantics, freshness, critical checks, quarantine policy, compatibility mode, and consumer migration process.

## 19. References

- [Confluent Schema Registry documentation](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [Apache Avro specification](https://avro.apache.org/docs/)
- [Great Expectations documentation](https://docs.greatexpectations.io/)
- [Google SRE — Data Processing Pipelines](https://sre.google/workbook/data-processing/)

Product-specific compatibility and validation behaviour depends on the format, registry, and library versions.
