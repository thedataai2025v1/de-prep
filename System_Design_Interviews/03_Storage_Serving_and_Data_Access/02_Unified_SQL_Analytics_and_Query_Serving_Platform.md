# Scenario 2 — Design a Unified SQL Analytics and Query-Serving Platform

**Interview level:** Staff / Principal Data Engineer  
**Primary themes:** Trino, Spark SQL, federation, workload management, concurrency, caching, governance, query resilience  
**Suggested interview time:** 45–60 minutes

## 1. Problem statement

Design a SQL access platform through which analysts, BI tools, data scientists, and scheduled workloads query data across a lakehouse, warehouse, operational databases, and selected SaaS sources. The platform must provide a consistent identity and policy model, isolate noisy workloads, control concurrency and cost, and deliver predictable performance.

Assume 2,000 analysts and service identities, 8,000 peak concurrent submitted queries, 500–1,000 concurrently running queries across multiple clusters, 15 PB of lakehouse data, and 40% of queries expected to finish within 10 seconds. Some workloads are interactive; others scan terabytes.

The key design question is whether to build one universal query cluster or a governed fleet of workload-specific query services behind a common access layer.

## 2. Clarifying questions

1. Which SQL engines and data sources must be supported?
2. Are joins across sources required, and what latency is acceptable?
3. What are p50/p95/p99 query latency targets by workload class?
4. What is the ratio of interactive BI, ad hoc, scheduled batch, and data science?
5. Are users allowed to run arbitrary SQL, or only certified semantic models?
6. What are the tenant, row-level, column-level, and purpose-based access rules?
7. How are query costs attributed and charged back?
8. Are result caching and data caching permitted for sensitive datasets?
9. Must long-running queries survive worker failures?
10. What are the freshness and consistency expectations across sources?

## 3. Requirements

### Functional
- SQL endpoint with authentication, authorization, query submission, cancellation, and status.
- Federated access to approved catalogs and connectors.
- Workload classification, resource groups/queues, quotas, and admission control.
- Query history, lineage, audit, explain plans, and cost attribution.
- Result and metadata caching where safe.
- Support for BI, notebooks, batch transformations, and large scans.
- Centralized policy and configuration with cluster-level execution isolation.

### Non-functional
- High availability for query submission and metadata access.
- Bounded queueing and predictable resource isolation.
- Graceful degradation under overload rather than cluster collapse.
- Strong security and auditability.
- Defined latency, availability, freshness, and cost SLOs.
- Independent scaling of interactive and batch workloads.

## 4. Capacity and workload model

Do not size only by concurrent users. Size by query resource demand:
- arrival rate by query class,
- active execution count,
- CPU-seconds/query,
- peak memory/query,
- bytes scanned/query,
- shuffle/exchange volume,
- connector/source limits,
- result size and client fetch rate.

Little's Law is a useful sanity check:
`average_in_flight = arrival_rate_per_second × average_query_duration_seconds`

Example: 100 query submissions/sec with a 20-second mean duration implies ~2,000 in-flight queries if every request is admitted. That is not a safe execution concurrency target; admission control must queue, reject, or route work.

Use separate workload pools:
- interactive short queries,
- BI dashboards,
- scheduled ELT,
- exploratory/ad hoc,
- large batch scans,
- administrative operations.

## 5. High-level architecture

```mermaid
flowchart TB
  CLIENT[BI / notebooks / services / jobs] --> GW[SQL gateway and identity]
  GW --> CLASS[Workload classifier and admission control]
  CLASS --> POOL1[Interactive query cluster]
  CLASS --> POOL2[BI query cluster]
  CLASS --> POOL3[Batch / resilient query cluster]
  POOL1 --> CAT[Catalog and metadata services]
  POOL2 --> CAT
  POOL3 --> CAT
  CAT --> LAKE[(Lakehouse tables)]
  CAT --> WH[(Warehouse)]
  CAT --> SRC[(Approved source connectors)]
  IAM[Identity and policy engine] --> GW
  IAM --> CAT
  OBS[Query telemetry / audit / lineage / cost] <-- GW
  OBS <-- POOL1
  OBS <-- POOL2
  OBS <-- POOL3
  CACHE[Result / metadata cache] <--> GW
```

## 6. Core design

### 6.1 Gateway and query classification

The gateway authenticates the caller, resolves tenant and workload identity, validates endpoint permissions, applies request limits, and routes to an appropriate engine pool. Classification can use identity, source application, catalog, query tags, SQL shape, expected scan, and explicit workload labels.

Do not rely solely on SQL parsing to infer query cost. A simple-looking query can trigger a huge scan or join. Use historical fingerprints, table statistics, limits, and runtime feedback. Permit users to declare a workload class but enforce quotas independently.

### 6.2 Cluster topology and isolation

A single shared cluster is operationally simple but vulnerable to noisy neighbours. A fleet of dedicated clusters gives stronger isolation but increases idle capacity, configuration drift, and operational burden.

A practical pattern:
- common gateway, identity, policy, catalog, and telemetry;
- separate execution pools for interactive and batch classes;
- per-tenant or per-domain resource groups within shared pools;
- dedicated clusters for regulated, mission-critical, or exceptionally heavy workloads.

Trino resource groups can limit resource usage and queue queries when capacity is exhausted. Use these as one layer of admission control, not as the entire governance architecture. Keep cluster-level limits and source-system limits too.

### 6.3 Query lifecycle

1. Authenticate user/service and resolve groups/tenant.
2. Apply endpoint and catalog authorization.
3. Classify workload and enforce rate/concurrency limits.
4. Parse, analyze, optimize, and plan.
5. Estimate scan/resource requirements and admit, queue, route, or reject.
6. Execute distributed stages and exchanges.
7. Enforce runtime limits and cancellation.
8. Stream results with backpressure and client fetch limits.
9. Emit query metrics, audit record, lineage, and cost allocation.

### 6.4 Federation versus data movement

Federation avoids copying data and can simplify access, but it couples query performance and availability to remote sources. Cross-source joins may be expensive, and connector pushdown varies.

Use federation for selective access and bounded operational queries. For repeated analytics, high-volume joins, or stable reporting, replicate/ingest data into a governed analytical store. Make data freshness and source protection explicit.

A federation policy should define:
- allowed source and connector list,
- query timeout and concurrency caps per source,
- pushdown requirements,
- maximum rows/bytes returned,
- masking and audit behaviour,
- fallback when source is unavailable.

### 6.5 Caching

Potential cache layers:
- metadata/statistics cache,
- result cache,
- BI semantic-layer cache,
- object/data cache near compute,
- connector-level cache.

Cache keys must include all relevant identity and query context. A result computed under one user's row-level policy must not be reused for another user unless the cache proves policy equivalence. Invalidate on data commits, policy changes, schema changes, and relevant session settings. Define TTL and staleness tolerance.

Caching is not a correctness substitute. For financial or operationally critical data, users may require explicit freshness guarantees or bypass controls.

### 6.6 Optimizer and data layout collaboration

Query engine performance is strongly affected by table statistics, partitioning, clustering, file size, and join cardinality. Build a feedback loop:
- collect query plans and scan metrics;
- identify recurring expensive patterns;
- improve table layout/statistics;
- recommend materialized views or aggregate tables;
- re-measure end-user latency and cost.

Do not tune only engine memory settings when the underlying data layout causes avoidable scans.

### 6.7 Fault tolerance and query classes

Long batch queries benefit from task-level retries and external exchange storage where supported. Short interactive queries may prefer fast failure and retry at the client. The query engine's retry mode, connector support, result size, and external exchange configuration matter.

Separate retry policy by cluster/workload class. A retry that helps a multi-hour batch query may increase tail latency for a high-volume interactive endpoint.

## 7. Workload management policy

Define:
- max running queries per tenant/user/workload,
- max queued queries and queue wait,
- memory and CPU limits,
- scan/bytes limits,
- query timeout,
- per-source concurrency,
- result size/fetch rate,
- priority and fairness rules,
- runaway query detection and kill policy.

Admission control must be applied before resource exhaustion. Use bounded queues; unbounded queues convert overload into delayed failure and memory pressure.

## 8. Security and governance

- Central identity propagation; never trust a user-supplied tenant header alone.
- Row/column masking and policy enforcement at a trusted layer.
- Separate service identities for scheduled jobs.
- TLS and secure internal engine communication.
- Secret management for connector credentials.
- Audit query text or normalized fingerprints according to privacy requirements.
- Prevent connector credentials from exposing unrestricted source access.
- Enforce data classification and purpose restrictions.
- Test cache isolation and cross-tenant access explicitly.

## 9. Observability and SLOs

Measure by workload class and tenant:
- query submission success and gateway availability,
- queue wait p50/p95/p99,
- planning and execution latency,
- query success/failure/cancel rates,
- CPU, memory, spill, exchange bytes,
- bytes scanned and returned,
- source connector latency/error rate,
- cache hit rate and stale-result incidents,
- resource-group saturation,
- cost per successful query and per workload.

Example SLOs:
- 99.9% monthly query submission availability.
- Interactive query p95 under a workload-specific target for a defined query cohort.
- 95% of BI dashboard queries complete within their agreed budget.
- No tenant exceeds its configured resource quota beyond a documented burst allowance.

Never promise one latency SLO for arbitrary SQL across petabytes.

## 10. Failure modes

| Failure | Mitigation |
|---|---|
| Coordinator/gateway failure | Multiple gateway instances, health-based routing, retry-safe submission IDs |
| Worker loss | Task/query retry where supported; otherwise clear failure and client retry |
| Source DB overload | Per-source concurrency, timeout, circuit breaker, query allowlist |
| Queue explosion | Bounded queue, admission control, reject with retry guidance |
| Bad query plan | Statistics, query limits, plan inspection, workload routing |
| Cache policy leak | Identity-aware cache keys, policy-version invalidation, tests |
| Catalog outage | HA catalog, cached metadata only where safe, clear degradation mode |
| Result consumer too slow | Backpressure, result-size limits, cursor expiry |
| Cluster noisy neighbour | Resource groups, separate pools, tenant quotas |

## 11. Cost and capacity strategy

Cost includes compute, coordinator overhead, object reads, cross-region/network transfer, exchange storage, source-system load, cache, idle capacity, and support burden.

Use:
- autoscaling with warm capacity for interactive traffic,
- scheduled scale-up/down for known batch windows,
- workload-specific instance profiles,
- query budgets and alerts,
- materialized aggregates for recurring heavy workloads,
- cost attribution by user, team, domain, and query fingerprint.

Do not autoscale solely on CPU; queue wait, memory pressure, query arrival rate, and source bottlenecks are also relevant.

## 12. Trade-offs

| Choice | Benefit | Cost / risk |
|---|---|---|
| One shared cluster | High utilization, simple operations | Noisy neighbours, shared failure domain |
| Dedicated workload pools | Predictability and isolation | Idle capacity, fleet complexity |
| Federation | Fresh access, less duplication | Source coupling and variable performance |
| Replication into lakehouse | Stable analytics and source protection | Freshness lag and duplicated storage |
| Result cache | Low latency and lower compute | Invalidation and security complexity |
| Fast-fail interactive | Protects tail latency | User-visible retries |
| Fault-tolerant batch | Resilience for long jobs | Exchange storage and retry overhead |

## 13. Principal-level follow-up questions

**Q1. How do you prevent a dashboard from starving data engineering jobs?**  
Use workload classification, resource groups, separate pools where needed, bounded queues, and reserved capacity. Define fairness and priority explicitly. Monitor queue wait by class and test peak contention.

**Q2. How do you handle federated joins across a lake and a production database?**  
First challenge whether the join belongs in the federated path. If necessary, push down selective predicates, cap source concurrency, and protect the operational database. For recurring or broad joins, replicate into an analytical store.

**Q3. What if a user changes their session identity after submitting a query?**  
Bind authenticated identity and authorization context server-side at submission. Do not trust mutable client headers during execution. Audit effective principal and policy version.

**Q4. How would you support 8,000 concurrent submitted queries?**  
Separate submission concurrency from execution concurrency. Use gateway scaling, bounded admission queues, workload routing, per-tenant quotas, and backpressure. Size execution based on measured resource demand and latency targets.

**Q5. Should every long query be retried?**  
No. Retry only when the operation is safe and the connector/execution mode supports it. Use task retry for appropriate batch workloads; interactive queries may fast-fail. Avoid retry storms and duplicate side effects for writes.

**Q6. How do you control cost for exploratory SQL?**  
Use scan estimates and runtime scan limits, per-user budgets, quotas, query timeout, and cost visibility. Offer sampled or curated datasets and explain-plan education. Escalate limits through an auditable process.

**Q7. How do you guarantee cached results respect row-level security?**  
Include effective identity/entitlement and policy version in cache scope, or cache only policy-independent intermediate data. Invalidate on policy changes and test with adversarial cross-user scenarios.

**Q8. When do you build a semantic layer?**  
When metric definitions, joins, and business logic are repeatedly reimplemented across BI tools. Keep semantic definitions versioned, governed, and tested; do not use the semantic layer to hide poor physical design.

## 14. Common mistakes

- Treating user count as query concurrency.
- One cluster for all workload types without admission control.
- Federation without source protection.
- Unbounded queues and retries.
- Caching without identity-aware invalidation.
- Claiming global query consistency across heterogeneous sources.
- Ignoring client fetch rate and result size.
- Measuring only average latency rather than queue and tail latency.

## 15. Interview drills

**Whiteboard:** Draw gateway, identity, classifier, workload pools, catalog, sources, policy, cache, and telemetry. Show where admission control occurs.

**Verbal:** Explain why a query platform is a fleet and policy problem, not simply a Trino deployment.

**Assumption change:** Increase interactive concurrency tenfold while keeping batch throughput constant. Explain isolation, reserved capacity, autoscaling, fairness, and cost.

## 16. References

- [Trino documentation](https://trino.io/docs/current/)
- [Trino resource groups](https://trino.io/docs/current/admin/resource-groups.html)
- [Trino fault-tolerant execution](https://trino.io/docs/current/admin/fault-tolerant-execution.html)
- [Trino query resiliency](https://trino.io/docs/current/installation/query-resiliency.html)

Check connector-specific retry support and configuration against the exact deployed release.
