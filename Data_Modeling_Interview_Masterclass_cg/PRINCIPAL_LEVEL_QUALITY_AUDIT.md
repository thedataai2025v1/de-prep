# Principal Data Engineer Quality Audit — Data Modeling Interview Masterclass

**Audit date:** 15 November 2026  
**Purpose:** Assess the current on-disk course against a Principal Data Engineer interview-preparation standard.  
**Status:** Reviewed baseline — **not yet the final upgraded course**.  

## Executive assessment

The course has all 36 planned lesson files, but lesson count is not the same as readiness. The foundational material is uneven: Modules 1–3 are brief and largely conceptual; Modules 4–6 have a stronger technical foundation but still need deeper implementation and measured exercises; five Module 7 cases remain materially below the Stripe benchmark.

Scores below assess interview-preparation depth, not the value of the topic or the completeness of the file inventory. The Stripe document is a stronger interview-preparation artifact, but it has not been certified as production-ready because its SQL and edge cases have not been executed against PostgreSQL and Snowflake.

## Module scores

| Module | Current score | Required work |
|---|---:|---|
| 01 Overview | 5.5/10 | Rework interview framework, normalization trade-offs, architecture context, realistic 60-minute walkthrough, and Senior/Staff/Principal evaluation rubric. |
| 02 Business Requirements | 6.0/10 | Add quantified workload sizing, metric contracts, freshness/completeness SLOs, cost modelling, retention/deletion/legal-hold requirements, and worked calculations. |
| 03 High Level Model Diagrams | 5.0/10 | Replace short sketches with complete conceptual, logical and physical models; show PK/FK, cardinality, optionality, temporal history, source ownership and model evolution. |
| 04 Dimension Design | 7.5/10 | Retain core material; add executable SCD1/2/6 labs, inferred members, retroactive corrections, late-arriving dimensions, effective-date fact lookup, hierarchy and privacy patterns. |
| 05 Fact Data Modeling | 7.5/10 | Retain core material; deepen multi-fact grains, fan-out prevention, allocations, semi-additive measures, financial reconciliation, incremental corrections and expected-result SQL. |
| 06 Performance Optimization | 7.5/10 | Add measured query-plan labs, join strategy, skew, file layout, compaction, concurrency, workload isolation, engine-specific tuning and cost/performance evidence. |
| 07 Mock Interviews | 4.5/10 excluding Stripe | Rebuild five cases with complete case-specific source/target DDL, diagrams, mappings, transformations, sample data, expected outputs, quality assertions and extended interviewer dialogues. |

## Module-specific findings

### 01 Overview — 5.5/10

Rework interview framework, normalization trade-offs, architecture context, realistic 60-minute walkthrough, and Senior/Staff/Principal evaluation rubric.

### 02 Business Requirements — 6.0/10

Add quantified workload sizing, metric contracts, freshness/completeness SLOs, cost modelling, retention/deletion/legal-hold requirements, and worked calculations.

### 03 High Level Model Diagrams — 5.0/10

Replace short sketches with complete conceptual, logical and physical models; show PK/FK, cardinality, optionality, temporal history, source ownership and model evolution.

### 04 Dimension Design — 7.5/10

Retain core material; add executable SCD1/2/6 labs, inferred members, retroactive corrections, late-arriving dimensions, effective-date fact lookup, hierarchy and privacy patterns.

### 05 Fact Data Modeling — 7.5/10

Retain core material; deepen multi-fact grains, fan-out prevention, allocations, semi-additive measures, financial reconciliation, incremental corrections and expected-result SQL.

### 06 Performance Optimization — 7.5/10

Add measured query-plan labs, join strategy, skew, file layout, compaction, concurrency, workload isolation, engine-specific tuning and cost/performance evidence.

### 07 Mock Interviews — 4.5/10 excluding Stripe

Rebuild five cases with complete case-specific source/target DDL, diagrams, mappings, transformations, sample data, expected outputs, quality assertions and extended interviewer dialogues.

## Module 7 case-by-case findings

| Case study | Current score | Main gaps |
|---|---:|---|
| 01 Ride Sharing Platform | 4.5/10 | Dispatch offer/retry/reassignment lifecycle; pooled-trip request bridge; driver supply intervals; effective-dated driver-vehicle relationship; fare versus ledger/earnings; incentive/toll/tip/refund treatment; telemetry privacy and retention; accounting perspective/sign convention. |
| 02 Customer Support Platform | 4.5/10 | SLA pause/resume intervals and business calendars; policy versioning; reopen cycles; assignment handoffs; multiple conversations; bot/human handling; agent concurrency; CSAT response bias; event-derived SLA reconstruction. |
| 03 Airbnb Platform | 4.5/10 | Calendar snapshot timestamp; distinguish calendar rules from observed availability; listing timezone; checkout-exclusive night expansion; booking modifications/cancellations; nightly price allocation; host payout allocation; eligible-night occupancy denominator. |
| 04 Stripe Payments Platform | 8.5/10 | Benchmark version is substantially improved. Still needs real PostgreSQL/Snowflake execution, executable assertions, complex financial edge-case tests, and verification of recovery/loading strategies. Interview-prep quality only; not production-certified. |
| 05 Instagram Social Platform | 4.5/10 | Define served/delivered/viewable impressions; deduplicate events; distinct reach; current graph versus graph history; video session derivation; experiment assignment versus exposure; deletion propagation; privacy taxonomy and high-volume ingestion. |
| 06 Amazon Marketplace | 4.0/10 | Separate ASIN/product, seller SKU and offer; contextual seller-SKU uniqueness; order header/line; split shipment allocations; inventory movement/snapshot; condition and fulfillment centre; return versus refund; settlement; distinguish hypothetical internal model from seller-scoped SP-API. |

## File inventory and depth signals

These counts are diagnostic signals, not automatic quality scores. Code-fence counts do not prove code correctness, and word counts do not prove technical depth.

| Lesson file | Words | Code fences | Headings |
|---|---:|---:|---:|
| `01_Overview/01_introduction_to_data_modeling_questions.md` | 1,144 | 0 | 17 |
| `01_Overview/02_how_to_answer_data_modeling_questions.md` | 1,226 | 0 | 22 |
| `01_Overview/03_rubric_for_data_modeling_questions.md` | 968 | 0 | 17 |
| `01_Overview/04_data_modeling_fundamentals.md` | 1,563 | 2 | 30 |
| `02_Business_Requirements/01_introduction_to_gathering_business_requirements.md` | 859 | 0 | 18 |
| `02_Business_Requirements/02_recognizing_the_core_business_problem.md` | 782 | 0 | 14 |
| `02_Business_Requirements/03_analyzing_metrics.md` | 825 | 0 | 14 |
| `02_Business_Requirements/04_analyzing_query_patterns.md` | 791 | 0 | 14 |
| `02_Business_Requirements/05_defining_latency_requirements.md` | 841 | 0 | 14 |
| `02_Business_Requirements/06_data_volume_and_scalability_considerations.md` | 851 | 0 | 15 |
| `02_Business_Requirements/07_data_retention_policies_and_historical_data_management.md` | 948 | 0 | 15 |
| `02_Business_Requirements/08_example_business_requirements_gathering.md` | 1,343 | 2 | 25 |
| `03_High_Level_Model_Diagrams/01_creating_high_level_model_diagrams.md` | 794 | 2 | 14 |
| `03_High_Level_Model_Diagrams/02_evolving_models_based_on_changing_requirements.md` | 775 | 0 | 14 |
| `03_High_Level_Model_Diagrams/03_practice_ecommerce_platform.md` | 745 | 2 | 13 |
| `03_High_Level_Model_Diagrams/04_practice_social_media_analytics.md` | 697 | 2 | 12 |
| `03_High_Level_Model_Diagrams/05_practice_video_streaming_service.md` | 619 | 2 | 12 |
| `03_High_Level_Model_Diagrams/06_practice_ride_sharing_platform.md` | 611 | 2 | 11 |
| `03_High_Level_Model_Diagrams/07_practice_cloud_services_platform.md` | 571 | 2 | 11 |
| `03_High_Level_Model_Diagrams/08_practice_online_advertising_platform.md` | 575 | 2 | 11 |
| `04_Dimension_Design/01_Dimension_Table_Design.md` | 2,077 | 10 | 24 |
| `04_Dimension_Design/02_Slowly_Changing_Dimensions.md` | 2,204 | 8 | 25 |
| `04_Dimension_Design/03_Advanced_Dimension_Design_Techniques.md` | 2,076 | 10 | 23 |
| `05_Fact_Data_Modeling/01_Transaction_Fact_Tables.md` | 2,249 | 8 | 25 |
| `05_Fact_Data_Modeling/02_Periodic_Snapshot_Fact_Tables.md` | 1,745 | 12 | 21 |
| `05_Fact_Data_Modeling/03_Accumulating_Snapshot_Fact_Tables.md` | 1,879 | 6 | 19 |
| `05_Fact_Data_Modeling/04_Bridge_Tables_and_Factless_Fact_Tables.md` | 2,020 | 12 | 23 |
| `06_Performance_Optimization/01_Date_Based_Partitioning.md` | 2,351 | 8 | 23 |
| `06_Performance_Optimization/02_Indexing_Strategies.md` | 2,035 | 6 | 30 |
| `06_Performance_Optimization/03_Aggregates_and_Materialized_Views.md` | 2,304 | 10 | 39 |
| `07_Mock_Interviews/01_Ride_Sharing_Platform.md` | 4,194 | 42 | 42 |
| `07_Mock_Interviews/02_Customer_Support_Platform.md` | 3,887 | 40 | 41 |
| `07_Mock_Interviews/03_Airbnb_Platform.md` | 3,907 | 40 | 41 |
| `07_Mock_Interviews/04_Stripe_Payments_Platform.md` | 11,872 | 46 | 123 |
| `07_Mock_Interviews/05_Instagram_Social_Platform.md` | 3,918 | 42 | 42 |
| `07_Mock_Interviews/06_Amazon_Marketplace.md` | 4,007 | 46 | 44 |
| `README.md` | 331 | 0 | 5 |
| `reference.md` | 1,379 | 0 | 10 |

## Mandatory acceptance gates for the upgraded course

### Gate 1 — Conceptual and modelling correctness
- Every entity and relationship has a clear business meaning, grain, cardinality and optionality.
- Natural keys, surrogate keys, primary keys, foreign keys, uniqueness and nullability are explicit.
- Temporal semantics are explicit: event time, effective time, ingestion time, update time and deletion time where relevant.
- Facts state their grain and additivity; multi-fact joins cannot silently fan out measures.
- Diagrams are real ER/model diagrams, not only lists of entity names.

### Gate 2 — Operational source realism
- Source schemas are labelled as educational fixtures or public API representations; no invented proprietary schema is presented as fact.
- Include source ownership, mutation semantics, CDC operation semantics, delete/tombstone handling, deduplication and replay assumptions.
- Include realistic sample rows, including malformed, late, duplicate and corrected records.

### Gate 3 — Source-to-target implementation
- Column-by-column mappings define transformations, type conversions, defaults, null handling, rejected records and lineage.
- Complete case-specific source and target DDL is supplied.
- SQL dialect is labelled for every non-portable script.
- Every transformation query refers to tables and columns actually defined in the lesson.
- Worked examples show source rows, target rows and expected query output.

### Gate 4 — Reliability and operational readiness
- Idempotency, retries, checkpointing, replay, backfills and late-arriving data are covered.
- Data-quality assertions cover duplicates, missing references, orphan records, invalid state transitions and reconciliation.
- Recovery and restatement behaviour is explicit.
- Privacy, access control, retention, deletion and auditability are addressed where applicable.

### Gate 5 — Principal-level interview readiness
- Include an extended candidate/interviewer dialogue with requirement changes and pushback.
- Provide strong model answers, alternative designs, trade-offs and failure modes.
- Include cost, scale, availability, freshness and operability decisions.
- Include evaluation rubrics that distinguish senior implementation answers from principal-level cross-team/platform judgment.

### Gate 6 — Validation honesty
- A script is labelled tested only if it has actually been run against the named engine and version.
- Unexecuted SQL is labelled illustrative or review-ready, not validated.
- Any unsupported assumption, public API limitation or inferred source entity is clearly identified.

## Recommended rebuild sequence

1. Module 1 — establish the interview framework and quality vocabulary.
2. Module 2 — quantitative requirements, metric contracts, SLOs and cost sizing.
3. Module 3 — full conceptual/logical/physical model diagrams and scenario labs.
4. Module 4 — executable dimension and temporal modelling labs.
5. Module 5 — multi-fact, allocation, semi-additive and reconciliation labs.
6. Module 6 — measured query-plan, storage-layout and engine-specific performance labs.
7. Module 7 — rebuild Amazon, ride-sharing, customer support, Airbnb and Instagram to the Stripe benchmark; then validate Stripe scripts and edge cases.

## Archive integrity note

The accompanying reviewed-baseline ZIP excludes the obsolete `_pre_rebuild.md` Stripe draft and includes this audit report. It preserves the current lesson contents; it does **not** claim that the outstanding upgrades above have already been implemented.