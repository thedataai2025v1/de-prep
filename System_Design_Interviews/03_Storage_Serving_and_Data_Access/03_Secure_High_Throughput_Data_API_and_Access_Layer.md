# Scenario 3 — Design a Secure, High-Throughput Data API and Access Layer

**Interview level:** Staff / Principal Data Engineer  
**Primary themes:** data APIs, REST/gRPC, pagination, authorization, caching, rate limiting, schema evolution, streaming responses  
**Suggested interview time:** 45–60 minutes

## 1. Problem statement

Design a governed data access layer that exposes curated enterprise datasets to internal applications, partner systems, and approved external consumers. Consumers need low-latency lookups, filtered queries, bulk extracts, and change notifications without direct access to the underlying lakehouse or warehouse.

Assume 20,000 client applications/service identities, 100,000 peak requests/sec across endpoints, 5 TB/day of bulk export traffic, and a mix of latency-sensitive point lookups and analytical queries. Some datasets contain sensitive information and require row-level entitlements, purpose limitation, audit, and revocation.

The core challenge is to avoid turning the data platform into an unbounded SQL proxy or a collection of bespoke endpoints with inconsistent security and semantics.

## 2. Clarifying questions

1. Are access patterns point lookup, bounded filtering, aggregations, bulk export, or all of these?
2. What are p50/p95/p99 latency and availability targets by endpoint?
3. Are consumers internal only, partner-facing, or public?
4. What is the authentication model: OAuth2, workload identity, mTLS, API keys?
5. How are tenant, user, purpose, and row-level entitlements represented?
6. What freshness is required and can clients tolerate stale cache results?
7. Are requests idempotent? Are writes or only reads exposed?
8. What are maximum response size, page size, and export size?
9. Do clients need stable pagination while underlying data changes?
10. What are data residency, retention, audit, and deletion requirements?

## 3. Requirements

### Functional
- Versioned APIs for approved datasets and use cases.
- Authentication, authorization, tenant resolution, and policy enforcement.
- Filtering, sorting, cursor pagination, and bounded projections.
- Bulk export jobs for large result sets.
- Rate limits, quotas, request validation, and query guardrails.
- Schema contracts, deprecation policy, and compatibility tests.
- Audit events, usage analytics, and cost attribution.
- Optional change feed/webhook delivery for supported datasets.

### Non-functional
- Predictable low latency for common reads.
- High availability and graceful degradation.
- Protection of source warehouses and operational databases.
- Strong tenant isolation and data minimization.
- Backpressure and bounded resource usage.
- Secure secrets, encryption, and traceable access.
- Safe schema evolution and backwards compatibility.

## 4. API product boundaries

Do not expose arbitrary SQL as the default API. Define products around consumer use cases and data contracts:
- entity lookup,
- bounded list/search,
- aggregated metrics,
- bulk export,
- incremental change feed.

Each API should declare owner, schema, freshness, supported filters, sort order, maximum page size, authorization scope, retention, and SLO. A stable API contract decouples consumers from physical table names and storage engines.

## 5. High-level architecture

```mermaid
flowchart TB
  C[Client applications] --> EDGE[DNS / WAF / API gateway]
  EDGE --> AUTH[Authentication and token validation]
  AUTH --> POL[Policy and entitlement service]
  POL --> ROUTER[Dataset API router]
  ROUTER --> CACHE[Identity-aware cache]
  ROUTER --> READ[Read service / query templates]
  READ --> SERVE[(Serving store / warehouse / lakehouse)]
  ROUTER --> EXPORT[Bulk export job service]
  EXPORT --> OBJ[(Encrypted export object storage)]
  OBJ --> C
  ROUTER --> AUDIT[Audit and usage events]
  READ --> AUDIT
  POLICYDB[(Entitlements / policy metadata)] --> POL
  SCHEMA[Schema registry / contract catalog] --> ROUTER
  OBS[Metrics / traces / SLOs] <-- EDGE
  OBS <-- READ
```

## 6. API and service design

### 6.1 Endpoint design

Prefer resource-oriented, task-specific endpoints with explicit supported parameters. Example:

`GET /v1/customers/{customer_id}/transactions?from=...&to=...&limit=100`

Avoid exposing arbitrary column names, unrestricted filters, or raw SQL. Validate:
- allowed fields and operators,
- filter combinations,
- maximum time range,
- sort keys,
- page size,
- request complexity,
- caller's dataset entitlement.

For high-throughput internal calls, gRPC can provide strongly typed contracts and efficient service-to-service communication. REST is broadly interoperable and easier for external consumers. The API contract should not depend on a particular transport.

### 6.2 Pagination and consistency

Offset pagination is simple but becomes expensive for deep pages and can skip/duplicate rows as data changes. Prefer keyset/cursor pagination using a deterministic ordering key, usually a unique tie-breaker such as `(event_time, record_id)`.

A cursor should be opaque, signed or authenticated, versioned, and bound to:
- query/filter fingerprint,
- tenant and authorization context,
- sort order,
- snapshot/version or consistency boundary,
- last-seen key,
- expiry.

For snapshot-stable pagination, bind the cursor to a warehouse/table snapshot or materialized result. For a live feed, document that pages represent moving data and may not provide a repeatable snapshot.

### 6.3 Query execution and source protection

Use predefined query templates or a constrained query builder. Route point lookups to an indexed serving store; route aggregates to an analytical engine; route large exports to asynchronous jobs. Do not run one warehouse query for every API call if a low-latency key-value lookup is required.

Apply:
- connection pools and concurrency caps,
- per-source circuit breakers,
- query timeouts,
- maximum scan/row limits,
- bounded retries with jitter,
- bulkheads between endpoint classes.

A retry on a read is usually safer than a retry on a mutation. For writes, require idempotency keys and explicit duplicate semantics.

### 6.4 Caching

Cache only where the freshness and security model permits. Cache key includes tenant, effective principal/entitlement scope, normalized filters, API version, and relevant policy/data version. Use short TTL for volatile datasets and invalidation events for policy changes.

Never let a shared cache bypass row-level policy. A safe pattern is to cache policy-independent results and apply authorization filtering after retrieval only if this does not expose unauthorized data to the application layer, logs, or cache operators. Prefer enforcing access before data leaves the trusted serving boundary.

### 6.5 Bulk export

Large responses should not be streamed through API gateway workers indefinitely. Create an asynchronous export:
1. Validate request and quota.
2. Persist immutable request definition and authorization context.
3. Execute with resource limits and snapshot semantics.
4. Write encrypted files to private object storage.
5. Publish a short-lived signed download link or authenticated retrieval endpoint.
6. Expire the export and record access/deletion evidence.

Re-check authorization at download time. Signed URLs should have narrow scope and short expiry; do not treat possession of a long-lived URL as a substitute for revocation.

### 6.6 Schema and versioning

Use additive evolution where possible. Breaking changes require a new API version or explicit migration window. Contract tests should validate producer output against published schemas and consumer expectations.

Define:
- field nullability and defaults,
- timestamp/timezone semantics,
- decimal precision,
- enum evolution,
- ordering guarantees,
- pagination semantics,
- error codes,
- deprecation timelines.

Do not silently change the meaning of a field while keeping its name and type.

## 7. Security model

- Authenticate callers with short-lived OAuth2/JWT or workload identity; use mTLS where mutual service identity is needed.
- Resolve tenant and principal from verified claims, not arbitrary request parameters.
- Authorize dataset, row, column, purpose, and operation.
- Keep policy evaluation close to the data access boundary.
- Rotate secrets and avoid long-lived API keys where possible.
- Encrypt data in transit and at rest.
- Redact sensitive values from logs, traces, and error messages.
- Audit who accessed which dataset, under what purpose and policy version.
- Rate-limit by identity, tenant, endpoint, and expensive operation.
- Test tenant boundary bypass, cursor tampering, replay, and privilege revocation.

## 8. Reliability and overload behaviour

Use bounded queues, deadlines, circuit breakers, bulkheads, and graceful degradation. Return meaningful status codes and retry guidance. A 429 should include retry hints where appropriate; a 503 should indicate temporary unavailability without leaking internals.

Avoid retry amplification: retries should be bounded, jittered, and coordinated with client behaviour. Idempotency keys should be stored for a defined duration and scoped to tenant/client/operation.

A service should not claim success until the underlying operation reaches its documented commit point. For asynchronous export, success means the job was accepted—not that the export is ready.

## 9. Observability and SLOs

Track:
- request rate, errors, and latency by endpoint/client/tenant,
- authorization denials and policy latency,
- cache hit rate and stale response incidents,
- source query latency, scans, and concurrency,
- rate-limit events and queue depth,
- export job age, duration, bytes, and failure rate,
- schema validation failures,
- cost per API product/client,
- data freshness at serving layer.

Example SLOs should be endpoint-specific, e.g. 99.9% availability and p95 under a defined threshold for point lookups; bulk exports have completion-time SLOs rather than request latency SLOs.

## 10. Failure modes

| Failure | Design response |
|---|---|
| Policy service unavailable | Fail closed for sensitive data; carefully defined cached policy only if safe |
| Serving store overloaded | Bulkhead, queue, rate limit, degrade optional endpoints |
| Client retry storm | Retry-after, jitter, idempotency, gateway throttling |
| Cursor replay/tampering | Signed opaque cursor, expiry, principal/tenant binding |
| Schema breaking change | Contract tests, versioning, deprecation and rollback |
| Export link leaked | Short TTL, narrow scope, reauthorization, audit |
| Cache leaks tenant data | Identity-aware key, policy version, negative tests |
| Slow client | Response limits, streaming backpressure, cursor expiry |
| Warehouse outage | Route to approved fallback or return explicit unavailable status |

## 11. Cost and performance

Separate point reads from analytical scans. Precompute popular aggregates, maintain indexed serving projections, and use asynchronous exports for large result sets. Monitor API fan-out: one API call that triggers dozens of warehouse queries can be more damaging than high request count suggests.

Use client quotas, query budgets, response compression where useful, connection reuse, and cache TTLs. Cost allocation should map API consumer to compute, storage, and transfer costs.

## 12. Trade-offs

| Choice | Advantage | Disadvantage |
|---|---|---|
| REST | Broad compatibility | Larger payload/protocol overhead |
| gRPC | Typed contracts and efficient RPC | Less direct browser/external interoperability |
| Offset pagination | Simple client model | Deep-page cost and unstable pages |
| Keyset pagination | Efficient and stable ordering | Requires deterministic keys |
| Synchronous export | Simple small responses | Ties up request workers |
| Async export | Scalable large results | Job lifecycle and secure download complexity |
| Shared cache | High reuse | Security/invalidation risk |
| Per-tenant cache | Isolation | Lower hit rate and higher cost |

## 13. Principal-level follow-up questions

**Q1. How would you prevent the API from becoming a SQL proxy?**  
Expose product-oriented contracts and constrained query templates. Maintain supported filters and query budgets. Route to fit-for-purpose serving systems and reserve ad hoc SQL for controlled analytical endpoints.

**Q2. How do you paginate while data is changing?**  
Use keyset pagination with a deterministic unique order. Bind cursors to a snapshot or watermark when repeatable traversal is required. Otherwise document live-feed semantics and possible movement.

**Q3. What if authorization changes halfway through a multi-page export?**  
Bind the export to an authorization snapshot for audit, but re-evaluate access at download and define revocation semantics. For sensitive data, cancel or regenerate the export if entitlement is revoked. Make policy version visible in the audit trail.

**Q4. How do you handle 100,000 requests/sec?**  
First segment by endpoint and cost. Scale stateless gateway/read services horizontally, cache hot reads safely, use serving stores for point lookups, apply quotas, and protect downstreams with bulkheads. Do not translate request rate directly into warehouse query rate.

**Q5. How do you make write APIs safe under retries?**  
Require idempotency keys, persist operation state, define deduplication scope and expiry, and return the original result for duplicate requests. Ensure the idempotency record and business mutation share a reliable transaction boundary or recovery protocol.

**Q6. What is your approach to API version deprecation?**  
Measure client usage, publish migration guides, offer parallel versions, communicate deadlines, provide contract tests, and retire only after owner sign-off and telemetry confirms migration.

**Q7. How do you avoid leaking PII through observability?**  
Use structured logs with identifiers or hashes, redact payloads, restrict trace attributes, apply retention controls, and audit access to telemetry. Debug sampling must not bypass data classification.

**Q8. When would you use change data capture or webhooks instead of polling?**  
Use change feeds for high-volume or freshness-sensitive consumers where delivery semantics can be made explicit. Webhooks need retry, ordering, deduplication, signature validation, and dead-letter handling. Polling remains appropriate for simpler, lower-frequency integrations.

## 14. Common mistakes

- Exposing arbitrary SQL to application consumers.
- Using offset pagination for deep/high-volume datasets.
- Treating API gateway authentication as complete data authorization.
- Caching without tenant and policy context.
- Serving multi-terabyte results synchronously.
- Retrying non-idempotent operations.
- Ignoring source-system protection and API fan-out.
- Logging raw PII or full query parameters.

## 15. Interview drills

**Whiteboard:** Show gateway, identity, policy, API router, cache, serving stores, export jobs, audit, and telemetry. Mark where authorization is evaluated and where a cursor is validated.

**Verbal:** Explain why the API contract is a data product boundary and not a thin wrapper around a warehouse.

**Assumption change:** Partner needs a 10 GB export every hour and row-level access can be revoked at any time. Redesign export lifecycle, snapshot semantics, download authorization, and audit.

## 16. References

- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [Google API design guide](https://cloud.google.com/apis/design)
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OAuth 2.0 Security Best Current Practice — RFC 9700](https://www.rfc-editor.org/rfc/rfc9700)

Use these as protocol and security references; endpoint contracts and SLOs must be tailored to the organization.
