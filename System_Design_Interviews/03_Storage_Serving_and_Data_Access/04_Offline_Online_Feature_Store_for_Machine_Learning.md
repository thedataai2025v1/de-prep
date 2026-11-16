# Scenario 4 — Design an Offline/Online Feature Store for Machine Learning

**Interview level:** Staff / Principal Data Engineer  
**Primary themes:** feature engineering, offline/online consistency, point-in-time correctness, materialization, low-latency serving, freshness, ML governance  
**Suggested interview time:** 45–60 minutes

## 1. Problem statement

Design a feature platform that enables ML teams to define, compute, discover, validate, train on, and serve reusable features. The platform must support historical training datasets and low-latency online inference while reducing training-serving skew and preventing temporal data leakage.

Assume 200 ML models, 20,000 feature definitions, 5 million active entities, 50,000 online reads/sec peak, 10,000 feature updates/sec peak, p99 online feature retrieval target below 20 ms for a typical feature vector, and offline history spanning 2 years.

The most important design issue is not merely storing features twice. It is preserving point-in-time semantics, freshness, entity identity, transformation consistency, and operational ownership.

## 2. Clarifying questions

1. What are the online inference latency and availability targets?
2. What are entity types and join keys?
3. Are features batch, streaming, request-time, or hybrid?
4. What freshness is required per feature?
5. Are feature transformations shared between training and serving?
6. How are late events, corrections, deletes, and backfills handled?
7. What is the required historical lookback and point-in-time semantics?
8. What is the feature vector size and online key/value access pattern?
9. Do models require strongly consistent feature groups or can values be eventually consistent?
10. What are data privacy, lineage, access, and retention requirements?

## 3. Requirements

### Functional
- Feature registry with ownership, schema, entities, descriptions, version, and freshness contract.
- Offline historical feature storage and point-in-time training retrieval.
- Online low-latency retrieval by entity key.
- Batch and streaming feature computation.
- Materialization from offline to online stores.
- Feature validation, freshness monitoring, lineage, and access control.
- Backfills and feature versioning with reproducible training datasets.
- Online feature logging and monitoring for training-serving skew.

### Non-functional
- Low p99 latency and high availability for online inference.
- Point-in-time correct historical joins.
- Defined consistency and freshness semantics.
- Scalable write/read throughput.
- Isolation between teams/models and protection of sensitive attributes.
- Reproducibility and auditability.
- Safe feature schema evolution and rollback.

## 4. High-level architecture

```mermaid
flowchart TB
  SRC[CDC / events / batch sources] --> TRANS[Feature transformations]
  TRANS --> OFF[(Offline feature store: lakehouse / warehouse)]
  TRANS --> REG[Feature registry and metadata]
  OFF --> PIT[Point-in-time retrieval]
  PIT --> TRAIN[Training dataset / model training]
  OFF --> MAT[Materialization service]
  MAT --> ON[(Online key-value store)]
  APP[Online inference service] --> FS[Feature serving API]
  FS --> ON
  REG --> FS
  APP --> LOG[Online feature / prediction logs]
  LOG --> OFF
  VALID[Freshness / schema / distribution validation] --> REG
  VALID --> OFF
  IAM[Identity, policy, lineage] --> REG
  IAM --> FS
  OBS[Latency / freshness / skew monitoring] <-- FS
```

## 5. Core architecture decisions

### 5.1 Feature registry and contracts

A feature definition should include:
- stable feature name and version,
- entity type and join key,
- data type and nullability,
- transformation/source,
- event-time and creation-time semantics,
- TTL/freshness expectation,
- owner and support contact,
- privacy classification and access policy,
- online/offline availability,
- validation rules and lineage.

Treat feature definitions as versioned contracts. A change in units, aggregation window, null handling, or time semantics is a breaking semantic change even if the physical type remains the same.

### 5.2 Offline store

The offline store holds historical feature values, usually in a lakehouse or analytical warehouse. It supports large scans, training set generation, backfills, and reproducibility.

Store event timestamp and, where necessary, created/ingested timestamp separately. Point-in-time joins must select the latest feature value available at or before each entity observation time, subject to TTL. Do not join the latest feature value to every historical training row; that leaks future information.

A robust retrieval operation:
1. Start with entity rows and their observation timestamps.
2. Find candidate feature rows with matching entity key.
3. Filter feature event time to `<= observation_time`.
4. Apply TTL and deterministic tie-breaking.
5. Select the latest valid feature value.
6. Preserve lineage and feature version.

### 5.3 Online store

The online store serves latest values by entity key with predictable latency. A key-value database is a common choice. It should support atomic update of a feature record or feature group where needed, TTL, replication, and horizontal scaling.

Key design options:
- entity key + feature view/group key,
- entity key with packed feature vector,
- per-feature keys.

Packed vectors reduce request fan-out but can create larger writes and more contention. Per-feature keys support independent updates but may increase online read round trips. Benchmark the actual feature retrieval pattern.

Define missing-feature behaviour explicitly: default value, null, fallback, or fail prediction. The model should know feature freshness and missingness where it affects prediction quality.

### 5.4 Materialization and consistency

Materialization copies selected historical/latest values from offline to online. It is a pipeline with checkpoints, watermarks, retries, and lag monitoring—not a one-time bulk copy.

Track:
- source snapshot or watermark,
- entity key,
- feature event timestamp,
- write timestamp,
- feature version,
- materialization job/run ID.

Use idempotent upserts and reject stale writes when an older event arrives after a newer one. A monotonic version or event-time comparison can prevent late events from overwriting newer values, but tie-breaking must be deterministic.

Do not claim atomic consistency between the offline lakehouse and online key-value store unless a real cross-system protocol provides it. Usually there is a freshness window and a measurable materialization lag.

### 5.5 Streaming and late-arriving data

For streaming features, use event-time windows and stateful processing. Define allowed lateness and correction behaviour. A late event may update offline history and online latest value; it may also require retractions or recomputation for windowed features.

For each feature, document whether the online value represents:
- latest observed event,
- latest event-time window aggregate,
- latest successfully materialized offline value,
- request-time calculation.

These semantics are not interchangeable.

### 5.6 Training-serving consistency

Prefer a shared transformation definition or tested equivalent implementations. Where online and offline transformations differ for performance reasons, run parity tests over the same input fixtures and compare outputs.

Log online feature values (or a safe reference to them) with prediction/request IDs so that production predictions can be reconstructed. Be careful with sensitive data retention and cost.

Monitor:
- feature distribution drift,
- null/default rate,
- freshness,
- training-serving skew,
- unseen entity rate,
- online/offline parity,
- feature retrieval timeout and fallback rate.

### 5.7 Backfills and versioning

A backfill can be expensive and can overwrite current online state if not isolated. Run backfills into a new feature version or controlled staging namespace, validate, then promote. Do not let a historical backfill blindly overwrite newer online values.

A safe migration:
1. Publish v2 contract and compute offline history.
2. Validate data quality and point-in-time retrieval.
3. Shadow materialize to a separate online namespace.
4. Compare v1/v2 feature values and model impact.
5. Switch model traffic gradually.
6. Retain rollback path and deprecate v1.

## 6. Data correctness and temporal semantics

Distinguish:
- **event time:** when the underlying fact happened;
- **ingestion time:** when the platform received it;
- **created/available time:** when the feature became usable;
- **observation time:** when the model would have made the prediction.

For leakage prevention, historical training must not use a feature that was not available at the observation time. In some domains, event time alone is insufficient: a transaction may occur on Monday but only arrive on Wednesday. The training row from Tuesday must not see it.

Use deterministic tie-breaking for equal timestamps and define deletion/correction semantics. Preserve source lineage and transformation version to reproduce training sets.

## 7. Failure modes

| Failure | Mitigation |
|---|---|
| Offline-to-online lag | Freshness SLO, catch-up capacity, alert and fallback |
| Stale event overwrites new value | Version/event-time guard and deterministic tie-break |
| Backfill corrupts online latest values | Separate namespace/version, controlled promotion |
| Point-in-time leakage | PIT join tests with event and availability timestamps |
| Online store outage | Multi-AZ, bounded fallback, prediction degradation policy |
| Feature schema mismatch | Registry validation, compatibility gates |
| Hot entity key | Shard strategy, caching, key distribution monitoring |
| Missing feature | Explicit default/null/fail policy and model training alignment |
| Transformation skew | Shared code or parity test suite and online logging |

## 8. Security and governance

- Feature-level ownership and data classification.
- Entity-level access controls where sensitive data requires it.
- Encryption, secret rotation, and least-privilege service identities.
- Avoid exposing raw PII through feature names, logs, or model metadata.
- Retention policies for historical and online features.
- Audit feature access and materialization.
- Enforce purpose restrictions and lineage from source to model.
- Support deletion propagation through offline history, online store, logs, and derived datasets.

## 9. SLOs and observability

Online:
- p50/p95/p99 feature retrieval latency,
- availability and timeout rate,
- request fan-out and payload size,
- missing feature rate,
- hot-key and throttling rate.

Offline/materialization:
- feature freshness lag,
- materialization completion and failure rate,
- point-in-time retrieval correctness,
- backfill duration,
- offline/online parity,
- data quality and schema drift.

Example contracts:
- critical online feature group p99 retrieval below 20 ms under specified load.
- 99% of streaming features materialized within 60 seconds.
- 99.9% availability for the online feature read API.
- zero tolerated future-feature leakage in validated training datasets.

SLOs must be set per feature criticality and measured under representative vector sizes.

## 10. Capacity and cost

Estimate online footprint:
`entity_count × average_serialized_feature_bytes × replication_factor × overhead`

Estimate read throughput from inference QPS × feature lookups per request. Reduce round trips by batching features per entity. Account for write amplification, TTL churn, replication, backup, and hot-key skew.

Offline cost includes storage history, training scans, backfills, compaction, and logging. Online cost is often driven by memory/IOPS, replication, network, and operational availability requirements.

## 11. Trade-offs

| Choice | Benefit | Trade-off |
|---|---|---|
| Separate offline and online stores | Fit-for-purpose performance | Materialization consistency gap |
| Single shared transformation | Reduced skew | May not suit low-latency runtime |
| Packed feature vector | Fewer reads | Larger writes and update contention |
| Per-feature key | Flexible updates | More read fan-out |
| Event-time latest | Correct temporal ordering | Late-event handling complexity |
| Ingestion-time latest | Simple availability semantics | May not reflect event chronology |
| Strong online consistency | Easier reasoning | Higher latency/cost constraints |
| Eventual consistency | Scale and availability | Staleness must be modelled |

## 12. Principal-level follow-up questions

**Q1. How do you prevent future data leakage in training?**  
Use point-in-time joins keyed by entity and observation time, and include availability/created time where ingestion is delayed. Apply TTL and deterministic tie-breaking. Test with synthetic future events and delayed-arrival cases.

**Q2. What if an old event arrives after a newer event?**  
Do not blindly apply last-write-wins by arrival time. Compare event time/version and use deterministic correction rules. The feature definition must specify whether late data can revise aggregates and how online state is repaired.

**Q3. Can offline and online stores be transactionally consistent?**  
Usually not with independent systems absent a distributed transaction protocol. Design an explicit consistency window, watermark, idempotent materialization, and observable lag. Expose feature freshness to consumers.

**Q4. How do you safely backfill two years of data?**  
Compute into a versioned/staging namespace, validate point-in-time correctness and distribution, then promote. Keep online latest state isolated from historical backfill writes. Use checkpoints and throttling.

**Q5. How do you handle a 20 ms p99 target with 100 features?**  
Batch feature retrieval by entity/group, co-locate related data where appropriate, minimize network round trips, use efficient serialization, and benchmark p99 under realistic concurrency. Cache only when freshness and policy allow.

**Q6. What happens when a feature is missing at inference?**  
Define per-feature behaviour: default, null with model support, fallback, or reject. Train with the same missingness semantics and monitor missing rate. Do not silently use a semantically incorrect default.

**Q7. How do you detect training-serving skew?**  
Log online feature values or reproducible references, compare distributions and sampled predictions with offline recomputation, and run parity tests. Segment by model, feature version, and traffic cohort.

**Q8. What belongs in a central feature platform versus an ML team?**  
Central platform owns registry, storage interfaces, materialization, serving reliability, security, lineage, and standard validation. ML teams own feature semantics, business validity, model-specific thresholds, and approval of changes.

## 13. Common mistakes

- Treating the online store as a cache of the latest offline table without consistency semantics.
- Using event time only when late ingestion changes availability.
- Training on latest values rather than point-in-time values.
- Allowing historical backfills to overwrite current online features.
- Ignoring entity key skew and read fan-out.
- Defining features without owner, TTL, or semantic version.
- Failing to log online feature values for debugging.
- Promising strong cross-store consistency without a protocol.

## 14. Interview drills

**Whiteboard:** Draw source events, transformation, offline store, point-in-time retrieval, materialization, online store, serving API, and prediction logs. Label event time, availability time, and observation time.

**Verbal:** Explain with a delayed-arrival transaction why joining only on event timestamp can still leak future information.

**Assumption change:** A model's p99 latency target drops from 20 ms to 5 ms and feature updates increase 20x. Revisit batching, key layout, cache, consistency, and cost.

## 15. References

- [Feast overview and architecture](https://docs.feast.dev/)
- [Feast offline store overview](https://docs.feast.dev/reference/offline-stores/overview)
- [Feast point-in-time joins](https://docs.feast.dev/getting-started/concepts/point-in-time-joins)
- [Feast quickstart](https://docs.feast.dev/getting-started/quickstart)

Feature store implementations differ. Verify supported store capabilities, materialization semantics, and point-in-time behaviour for the chosen version.
