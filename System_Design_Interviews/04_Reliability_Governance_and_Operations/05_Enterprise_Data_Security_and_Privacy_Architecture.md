# Scenario 5 — Design Enterprise Data Security and Privacy for a Multi-Tenant Data Platform

**Interview level:** Staff / Principal Data Engineer  
**Suggested interview time:** 45–60 minutes  
**Core themes:** IAM, least privilege, encryption, key management, row/column security, privacy, retention, deletion, audit, threat modelling

## 1. Problem statement

Design a security and privacy architecture for a multi-tenant enterprise data platform containing customer, employee, financial, product, and operational data. Data flows through CDC, Kafka, object storage, lakehouse tables, warehouses, APIs, ML feature stores, and BI tools.

Assume 30 domains, 5,000 engineers/analysts, 2,000 service identities, external partners, and multiple data residency obligations. The platform must prevent unauthorized access, reduce sensitive data exposure, support audit and deletion requests, and remain usable for legitimate analytics.

The goal is defence in depth: identity, policy, storage, compute, APIs, governance, and operational controls must work together.

## 2. Clarifying questions

1. What data classifications exist and which regulations/contracts apply?
2. What are the tenant boundaries and isolation requirements?
3. Which identities are human, workload, partner, or emergency/break-glass?
4. Where must data reside and where may it be processed?
5. Are row-level, column-level, purpose-based, or attribute-based policies required?
6. How are encryption keys owned, rotated, revoked, and recovered?
7. What is the expected deletion SLA and how do backups/replicas behave?
8. Are raw data, query text, logs, metadata, and ML features also sensitive?
9. What audit evidence is required and how long must it be retained?
10. What is the availability behaviour if the policy or identity service is unavailable?

## 3. Security requirements

### Functional
- Central identity federation and workload identity.
- Authentication and authorization across all data planes.
- Least-privilege access to storage, catalogs, compute, APIs, and secrets.
- Row/column masking and purpose-based access.
- Encryption in transit and at rest with managed keys.
- Data classification, retention, residency, and deletion workflows.
- Auditable access and policy changes.
- Secrets rotation and emergency access.
- Tenant isolation and periodic access review.

### Non-functional
- Low-latency policy evaluation.
- High availability of authentication and authorization paths.
- No uncontrolled bypass through direct object-store or connector access.
- Key recovery and rotation without data loss.
- Traceable and testable policy propagation.
- Minimal sensitive data in logs and telemetry.

## 4. Threat model

Consider:
- compromised user credentials,
- compromised service account,
- overly broad IAM role,
- malicious insider,
- cross-tenant query bug,
- leaked signed URL or export,
- public object/bucket misconfiguration,
- stolen encryption key,
- query/log/sample-data leakage,
- policy synchronization failure,
- malicious or accidental deletion,
- supply-chain compromise in pipeline code or dependencies.

For each threat, identify preventive, detective, and recovery controls. Security is not a single gateway.

## 5. High-level architecture

```mermaid
flowchart TB
  USER[Users / services / partners] --> IDP[Identity provider / workload identity]
  IDP --> GW[API / SQL / platform gateways]
  GW --> PDP[Policy decision point]
  PDP --> PEP[Policy enforcement points]
  PEP --> CAT[Catalog / table access]
  PEP --> OBJ[(Object storage)]
  PEP --> WH[(Warehouse / OLAP)]
  PEP --> API[Data APIs / feature serving]
  KMS[Key management service] --> ENC[Encryption services]
  ENC --> OBJ
  ENC --> WH
  SEC[Classification / privacy metadata] --> PDP
  SEC --> RET[Retention / deletion workflow]
  RET --> OBJ
  RET --> WH
  RET --> LOG[Replicas / exports / caches / backups]
  AUDIT[Immutable audit pipeline] <-- GW
  AUDIT <-- PDP
  AUDIT <-- PEP
  SIEM[Security monitoring / incident response] <-- AUDIT
  VAULT[Secrets manager] --> PEP
```

## 6. Identity architecture

Separate:
- human identities federated through enterprise SSO;
- workload identities for pipelines and services;
- partner identities with scoped access;
- emergency/break-glass identities.

Prefer short-lived credentials and workload identity over static access keys. Bind service identity to workload, environment, and approved purpose. Avoid shared service accounts across unrelated pipelines because they destroy attribution and enlarge blast radius.

Authorization should use verified identity claims and policy context. Do not trust tenant IDs supplied in request parameters without server-side verification.

## 7. Authorization model

Use a combination of:
- RBAC for roles and broad responsibilities;
- ABAC for attributes such as tenant, environment, data classification, purpose, region, and device/workload context;
- row-level policies for tenant/entity restrictions;
- column masking or projection controls for sensitive fields.

Define a policy decision point (PDP) and policy enforcement points (PEPs). The PDP evaluates identity and resource attributes; PEPs enforce the decision in query engines, APIs, storage, and export workflows.

Critical rule: users must not bypass the PEP by obtaining direct object-store credentials or using an alternate connector path. Validate every access route.

## 8. Data classification and minimization

Create a classification taxonomy, for example:
- Public,
- Internal,
- Confidential,
- Restricted / highly sensitive.

Classification should be assigned at dataset and column level, inherited where appropriate, and reviewed. Minimize data before it enters broad analytical zones:
- tokenize or pseudonymize identifiers,
- drop unnecessary fields,
- aggregate where individual-level data is not required,
- separate identity mapping tables,
- restrict raw zones to ingestion and security-approved roles.

Pseudonymized data can remain personal data if re-identification is possible. Do not call it anonymous without a documented assessment.

## 9. Encryption and key management

Use TLS for service-to-service and client connections. Encrypt storage, backups, exports, and temporary files. Use managed key services and separate key administration from data access administration.

Consider:
- key hierarchy and envelope encryption,
- rotation schedule and emergency revocation,
- per-environment and possibly per-tenant keys,
- cross-region key recovery,
- key policy audit,
- separation of duties,
- data re-encryption or key rewrapping requirements.

Encryption at rest does not protect data from an authorized but overprivileged query principal. Authorization and minimization remain essential.

## 10. Tenant isolation

Isolation options:
- logical row-level tenant policy,
- schema/database per tenant,
- bucket/prefix per tenant,
- dedicated account/project/cluster.

Use stronger isolation when contractual, regulatory, or blast-radius requirements demand it. Shared infrastructure can be cost-efficient but needs mandatory tenant context, policy tests, and resource isolation.

Test negative cases:
- tenant A cannot query tenant B;
- cache cannot return another tenant's result;
- exports cannot be downloaded by another principal;
- feature store keys cannot collide across tenants;
- logs and metadata do not expose cross-tenant details.

## 11. Privacy lifecycle: collection to deletion

Design privacy controls across the full lifecycle:
1. Collection: purpose and minimization.
2. Ingestion: classification, tokenization, validation.
3. Storage: access, encryption, residency, retention.
4. Processing: approved purpose and controlled joins.
5. Serving: masking, row policies, audit.
6. Sharing: partner controls, export expiry, data-use terms.
7. Retention: expiry and legal hold.
8. Deletion: propagation and evidence.

Deletion is complex because data may exist in raw logs, snapshots, replicas, derived tables, caches, exports, feature stores, and backups. Maintain a deletion ledger with subject identifier mapping, affected assets, action status, exceptions, and evidence. Define how immutable backups age out and how deleted data is prevented from reappearing during restore.

## 12. Secrets and pipeline supply chain

- Store credentials in a secrets manager, not code or notebooks.
- Use short-lived tokens and rotation.
- Scope source credentials to required tables/operations.
- Scan code, containers, dependencies, and infrastructure configuration.
- Sign or verify deployment artifacts where supported.
- Restrict who can modify production DAGs, jobs, and policy definitions.
- Separate developer, deployer, and runtime permissions.
- Prevent untrusted code from exfiltrating credentials through logs or network access.

## 13. Audit and detection

Record:
- principal and workload identity,
- resource and operation,
- tenant and purpose,
- policy decision and policy version,
- timestamp and source,
- query/job/export identifier,
- result status and relevant data classification.

Avoid storing raw sensitive values or full query payloads unless required and protected. Protect audit records against alteration and define retention. Alert on unusual access volume, privilege escalation, bulk exports, denied-access spikes, policy changes, and access outside expected patterns.

## 14. Failure modes and fail-safe behaviour

| Failure | Risk | Response |
|---|---|---|
| Identity provider outage | New authentication unavailable | Short-lived validated sessions only within policy; documented outage mode |
| PDP unavailable | Authorization uncertain | Fail closed for restricted data |
| Policy sync lag | Stale permissions | Drift detection; runtime enforcement version check |
| Key service unavailable | Data inaccessible | Multi-region key recovery and tested procedures |
| Leaked service credential | Broad data access | Short TTL, narrow scope, revoke, investigate audit |
| Direct storage bypass | Policy circumvention | Private network, scoped roles, deny-by-default |
| Deletion misses derived data | Privacy noncompliance | Lineage-driven deletion workflow and ledger |
| Cache leaks result | Cross-tenant exposure | Identity-aware cache scope and purge |
| Audit pipeline outage | Evidence gap | Local durable buffer and alert on audit loss |

## 15. Sample interviewer–candidate conversation

**Interviewer:** We have row-level security in the BI tool. Is that sufficient?

**Candidate:** Not if users or services can access the warehouse, object storage, APIs, or exports through other paths. I would enforce policy at trusted data access boundaries and verify every route. BI filtering can improve user experience, but it cannot be the sole security control.

**Interviewer:** A service account needs access to all customer data for a daily pipeline.

**Candidate:** I would challenge whether it needs all columns and all tenants. Scope it to required datasets, fields, and operations, use a dedicated workload identity, short-lived credentials, and audit. If broad access is genuinely required, document owner approval and compensating controls.

**Interviewer:** A user requests deletion of their data. We delete it from the main table. Are we done?

**Candidate:** Not necessarily. We need lineage to find raw events, snapshots, replicas, derived tables, feature stores, caches, exports, and downstream products. We need to account for backups and legal holds, prevent deleted data from reappearing during restore, and retain completion evidence.

**Interviewer:** The policy engine is unavailable. Should we keep serving cached authorization decisions?

**Candidate:** It depends on risk and cache lifetime. For restricted data, I would fail closed when the decision is stale or cannot be validated. For lower-risk internal data, a short-lived signed decision may be acceptable if policy and security approve it. This must be explicit, not an accidental fallback.

**Interviewer:** How do you stop developers from copying production data to local machines?

**Candidate:** Use data minimization, controlled non-production datasets, masking/tokenization, restricted egress, endpoint controls, audit, and least-privilege access. Make safe test data easy to obtain so developers are not incentivized to bypass controls.

## 16. Principal-level follow-up questions

**Q1. RBAC versus ABAC — which do you choose?**  
Use RBAC for stable roles and ABAC for context such as tenant, purpose, classification, region, and workload. Most enterprise systems need a combination.

**Q2. How do you prevent direct object-store access bypassing row policies?**  
Use private storage, narrowly scoped workload roles, deny broad user access, and mediate access through trusted engines or services. If direct access is needed, enforce equivalent controls at that boundary.

**Q3. Is pseudonymization anonymization?**  
No. If a mapping or auxiliary information can re-identify a person, the data may remain personal data. Treat it according to applicable legal and organizational requirements.

**Q4. How do you revoke access quickly across many engines?**  
Centralize policy source, version policy, propagate through adapters, use short-lived credentials, and monitor enforcement lag. High-risk access should be denied if policy freshness cannot be established.

**Q5. How do you handle deletion in immutable backups?**  
Define backup retention and legal basis, prevent restore from reintroducing deleted records, and use tombstone/deletion ledgers to reapply deletions after restore. Document exceptions and expiry timelines with legal/privacy teams.

**Q6. Should every tenant have a separate account and cluster?**  
No. Choose isolation based on risk, contract, residency, noisy-neighbour impact, and cost. Logical isolation is acceptable for many workloads if enforcement is robust and tested.

**Q7. How do you secure query logs?**  
Normalize or redact literals, restrict query text access, apply retention, and avoid sensitive parameters in logs. Store execution metadata separately from raw SQL where possible.

**Q8. How do you prove controls work?**  
Automated negative authorization tests, policy simulation, access reviews, audit sampling, penetration testing, deletion drills, key rotation exercises, and incident response exercises.

## 17. Common interview mistakes

- Assuming encryption solves access control.
- Treating BI-level row filtering as sufficient.
- Using shared long-lived credentials.
- Ignoring direct object-store and export paths.
- Calling pseudonymized data anonymous.
- Forgetting caches, snapshots, replicas, and backups in deletion.
- Making the policy engine a single point of failure without a defined mode.
- Collecting audit logs that themselves expose sensitive data.

## 18. Practice exercise

Design access for:
1. An analyst who can view aggregated customer metrics but not direct identifiers.
2. A fraud model service that needs selected customer features at low latency.
3. A partner who can access only its own tenant's records.
4. A data engineer debugging a failed pipeline.
5. A privacy request requiring deletion across downstream products.

For each, state identity, authorization attributes, enforcement points, audit events, and failure behaviour.

## 19. References

- [NIST SP 800-53 Security and Privacy Controls](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [AWS IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS KMS best practices](https://docs.aws.amazon.com/kms/latest/developerguide/best-practices.html)

This is architecture interview guidance, not legal advice. Map controls to the jurisdictions, contracts, and regulatory obligations applicable to the organization.
