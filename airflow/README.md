# Airflow Interview Preparation

> **Target:** Principal Data Engineer (17 YOE)
> **Companies:** Atlassian, Snowflake, Databricks, Uber, Google, Adobe, Confluent
> **Focus:** Architectural depth, production-scale orchestration, failure modes, cost optimization, cross-team governance

## Files

| File | Description |
|------|-------------|
| [01-Theory-Questions.md](01-Theory-Questions.md) | 30 theory questions with detailed answers, examples, follow-ups |
| [02-Practical-Questions.md](02-Practical-Questions.md) | 30 practical/scenario questions with solutions, edge cases, follow-ups |

## Topics Covered

### Theory Questions (01-Theory-Questions.md)
1. Airflow Architecture & Components
2. DAGs & Task Dependencies
3. Operators vs Sensors vs Hooks
4. Task States & Lifecycle
5. Executor Types Comparison
6. Celery Executor Architecture
7. Kubernetes Executor Architecture
8. XComs & Cross-Task Communication
9. Backfill & Catchup
10. Task Concurrency & Pool Management
11. Logging & Monitoring
12. Plugin System
13. Airflow Configuration Management
14. Scheduling & Timing
15. Task Retry & Recovery
16. Airflow Metadata Database
17. Airflow at Scale
18. Airflow as Orchestrator vs Runner
19. Dynamic Task Generation & Task Mapping
20. Airflow Security & RBAC
21. Custom Operators Development
22. Airflow vs Alternatives (Prefect, Dagster, etc.)
23. Airflow Upgrade & Migration
24. Airflow 2.x Breaking Changes
25. Connections & Variables Management
26. External Triggering & Callbacks
27. SLA Misses & Alerting
28. Airflow on Kubernetes
29. Operational Best Practices
30. Disaster Recovery & High Availability

### Practical Questions (02-Practical-Questions.md)
1. Building a production ETL pipeline
2. Dynamic task generation from external config
3. Handling task failures with retries and alerts
4. Implementing data quality checks with sensors
5. Building a CDC pipeline orchestrator
6. Cross-database data synchronization
7. Implementing idempotent DAGs
8. Building a multi-tenant Airflow deployment
9. Dynamic task mapping for parameterized jobs
10. Implementing custom operators for cloud services
11. Building a scalable data lake ingestion pipeline
12. Implementing branching logic with BranchPythonOperator
13. Cross-DAG dependencies with ExternalTaskSensor
14. Building a real-time streaming pipeline orchestrator
15. Implementing custom XCom backends
16. Building a cost-optimized DAG with auto-scaling
17. Implementing data partition-aware processing
18. Building a multi-region data replication pipeline
19. Implementing custom hooks for proprietary systems
20. Building a ML pipeline orchestrator with Airflow
21. Implementing custom sensors for event-driven processing
22. Building a complex dependency graph with subDAGs/task groups
23. Implementing SLA monitoring and alerting
24. Building a data quality framework with Great Expectations
25. Implementing custom callbacks and error handling
26. Building a multi-cluster Kubernetes deployment
27. Implementing dynamic resource allocation
28. Building a complex ETL with data validation at each step
29. Implementing custom plugins for enterprise integration
30. Designing a production Airflow deployment at scale

## How to Use

1. **Read theory questions first** to understand the architecture and concepts
2. **Practice practical questions** to prepare for hands-on interview scenarios
3. **Study follow-up questions** — they reveal what interviewers probe for at Principal level
4. **Focus on Principal-level considerations** — trade-offs, architecture decisions, and leadership aspects

## Prerequisites for This Interview Prep

- Working knowledge of Python, SQL, distributed systems
- Understanding of cloud platforms (AWS, GCP, Azure)
- Familiarity with data engineering concepts (ETL, ELT, CDC, streaming)
- Understanding of Kubernetes and container orchestration
