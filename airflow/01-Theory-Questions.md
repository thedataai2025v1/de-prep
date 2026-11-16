# Airflow Interview Preparation — Theory Questions (Q1–Q30)

> **Target:** Principal Data Engineer (17 YOE)
> **Companies:** Atlassian, Snowflake, Databricks, Uber, Google, Adobe, Confluent
> **Focus:** Architectural depth, production-scale decisions, failure modes, cost optimization

---

## Q1. Airflow Architecture & Core Components

### Question
Explain Apache Airflow's architecture. Core components (Scheduler, Executor, Webserver, DAG Processor, Metadata DB), how they interact, and why Airflow is the de facto production orchestrator at scale.

### Answer
Airflow is a DAG-centric workflow orchestration engine with loosely coupled components communicating via a shared metadata database.

**Core Components:**
1. **Webserver** — Flask-based UI; renders DAGs, task states, logs, variables, connections.
2. **Scheduler** — heart of Airflow; parses DAG files, creates task instances, sends ready tasks to the executor. Multiple schedulers support HA.
3. **Executor** — interface between scheduler and workers; determines how/where tasks run.
4. **Workers** — execute tasks (Celery workers, Kubernetes pods, or local threads).
5. **Metadata Database** — PostgreSQL/MySQL; stores DAG definitions, task states, logs, XComs, variables.
6. **DAG Processor** — parses Python DAG files; runs in the scheduler process.
7. **Plugin System** — custom operators, hooks, UI extensions via `plugins/` folder.

**Flow:** Scheduler parses DAGs → creates TaskInstances for scheduled intervals → marks ready tasks → Executor dispatches to workers → workers execute → results written back to DB.

**Why Airflow wins at scale:**
- Code-as-infrastructure (Python DAGs are versionable, testable, reviewable)
- Pluggable executors (Local, Celery, K8s, CeleryK8s)
- Battle-tested by Airbnb, Netflix, Lyft at massive scale
- Rich operator ecosystem (300+ providers)
- Strong community and CNCF graduation
- Native backfill, retries, SLAs, dynamic task generation

**Trade-offs:** DB-heavy (every task state change is a DB write), not a real-time engine (not for sub-second latency), Python-only DAG authoring.

---

## Q2. Executor Types and Selection at Scale

### Question
Compare Airflow executors. Which executor do you choose for production at enterprise scale, and why?

### Answer

| Executor | Scale | Use Case |
|---|---|---|
| Sequential | None | Development only; single-threaded |
| Local | Small | Dev/test; runs on same machine as scheduler |
| Celery | Large | Distributed workers via message broker (RabbitMQ/Redis) |
| Kubernetes | Large | Each task = pod; full isolation, auto-scaling |
| CeleryK8s | Large | Hybrid: short tasks via Celery, heavy tasks via K8s |
| CeleryExecutor (Dask) | Niche | GPU-heavy ML workloads |

**Principal-level recommendation:**

**KubernetesExecutor** is the production standard for greenfield deployments:
- Each task gets its own pod with exact resource specs (CPU, memory, GPU)
- Native auto-scaling via HPA
- Complete isolation between tasks
- No persistent worker fleet to manage
- Works natively with Databricks, Snowflake, Spark operators via ephemeral pods

**CeleryExecutor** remains dominant when:
- You already have a stable Celery/Kombu infrastructure
- Tasks are short-lived and lightweight (reduces pod startup overhead)
- You need consistent worker pools for GPU workloads
- Running on-prem without K8s

**CeleryK8s** when you need both: short tasks via Celery workers, heavy Spark/ML jobs via K8s pods.

**Key production configs:**
- `worker_concurrency` — parallel tasks per worker (default 16)
- `max_tasks_per_child` — restart worker after N tasks (prevent memory leaks)
- `celeryd_options.--pool=solo` — for dev
- `kubernetes_worker_pods_concurrency` — tasks per K8s worker pod

---

## Q3. DAG Design Patterns and Best Practices

### Question
What DAG design patterns do you use at Principal level? Cover task structure, error handling, idempotency, and anti-patterns.

### Answer

**Core Design Principles:**

1. **Idempotency** — every task must be safely re-runnable. Use overwrite semantics, transactional loads, or upserts. Never assume a task runs only once.

2. **Narrow DAGs** — prefer many small, focused DAGs over one monolithic DAG. Each DAG = one business domain or pipeline.

3. **Task Granularity** — tasks should be 1-5 minutes of work. Too fine-grained = scheduling overhead; too coarse = long retry times and blocked downstream tasks.

4. **Separation of Concerns** — one task = one operation. Don't combine extraction, transformation, and loading in one task.

**Production Patterns:**

```python
# Pattern 1: Trigger-only downstream DAGs
dag_a = DAG("data_ingestion")
dag_b = DAG("data_processing", trigger_rule="all_success")

# Pattern 2: Dynamic task generation from config
for table in TABLE_LIST:
    extract_task = ExtractOperator(task_id=f"extract_{table}")

# Pattern 3: Error boundary with catch-up task
failure_handler = PythonOperator(
    task_id="handle_failure",
    on_failure_callback=notify_slack,
    trigger_rule="none_failed_min_one_success"
)

# Pattern 4: Parameterized DAGs for multi-tenant
for tenant in TENANTS:
    dag = DAG(f"etl_{tenant}", ... )
```

**Anti-Patterns to Avoid:**
- Put all pipelines in one DAG (single point of failure)
- Use `PythonOperator` for I/O-bound work (use `BashOperator` with proper tools)
- Hardcode connections/credentials in DAG files
- Use `print()` for logging instead of `logging.info()`
- Long-running tasks without heartbeat/checkpointing
- Circular dependencies or implicit cross-DAG dependencies

**Idempotency Patterns:**
- Partition by `{{ ds }}` (data date) for daily pipelines
- Use `MERGE` / `UPSERT` for incremental loads
- Write to staging tables, then atomically swap
- Include `{{ run_id }}` for deduplication

---

## Q4. Connection and Secrets Management

### Question
How do you manage connections, credentials and secrets in production Airflow? Compare approaches and security implications.

### Answer

**Connection Types:** Airflow stores connections (database URLs, API endpoints) and variables (key-value config) in the metadata DB, encrypted at rest with Fernet encryption.

**Production Approaches (by security level):**

**Level 1: Airflow UI (Not for Production)**
- Connections stored encrypted in metadata DB
- Accessible to anyone with Airflow UI access
- Good for dev/test only

**Level 2: Environment Variables**
```bash
AIRFLOW_CONN_MY_POSTGRES=postgres://user:pass@host/db
```
- Simple, works in any deployment
- Risk: visible in process lists, container inspect
- Never store passwords in env vars at scale

**Level 3: Backend Secrets Providers (Production Standard)**
```python
secrets_backend = 'airflow.providers.amazon.aws.secrets.systems_manager.AWSSystemsManagerBackend'
secrets_backend_kwargs = {
    "connections_prefix": "/airflow/connections",
    "regions": ["us-east-1"],
}
```
Supported backends:
- **AWS Secrets Manager / SSM** — AWS-native, fine-granced IAM, audit trail
- **HashiCorp Vault** — vendor-neutral, dynamic secrets, lease management
- **GCP Secret Manager** — GCP-native, IAM-integrated
- **Azure Key Vault** — Azure-native with RBAC

**Level 4: External Secrets Operator (K8s + Vault)**
- Kubernetes-native: secrets mounted as volumes/env vars
- Auto-rotation of credentials
- Zero-secret-in-image policy

**Principal-level secrets strategy:**
```python
# airflow.cfg
[secrets]
secrets_backend = airflow.providers.apache.atlas.secrets.vault.AtlasVaultBackend
secrets_backend_kwargs = {
    "url": "https://vault.internal:8200",
    "token_path": "/run/secrets/vault-token",
    "connection_path": "secret/data/airflow/connections",
    "variable_path": "secret/data/airflow/variables",
}
```

**Critical security practices:**
- Never log connection strings or variable values
- Rotate credentials automatically (Vault leases)
- Encrypt metadata DB at rest (TDE)
- Restrict UI access to read-only for developers
- Audit all secret access via Vault/AWS CloudTrail
- Use short-lived tokens for downstream services

---

## Q5. Task Dependencies and Control Flow

### Question
Explain Airflow's dependency model. How do you set upstream/downstream relationships, use trigger rules, and implement complex conditional logic?

### Answer

**Setting Dependencies:**
```python
# Bitwise operators (chain style)
task_a >> task_b >> task_c

# Cross-chain (fan-out / fan-in)
task_a >> [task_b, task_c] >> task_d

# Explicit methods
task_b.set_upstream(task_a)
task_c.set_downstream(task_b)

# chain() and chain_linear() helpers
from airflow.utils.task_group import TaskGroup
from airflow.models.baseoperator import chain
chain(task_a, task_b, task_c)
```

**Trigger Rules:**
```python
# Default: all_success (all upstream must succeed)
PythonOperator(task_id="default_task")

# Run even if some upstream fail
PythonOperator(task_id="flexible_task", trigger_rule="none_failed")

# Run only if at least one succeeded
PythonOperator(task_id="catch_task", trigger_rule="one_success")

# Run regardless of upstream state (except upstream_failure)
PythonOperator(task_id="cleanup", trigger_rule="all_done")

# Do NOT run if any upstream failed
PythonOperator(task_id="strict", trigger_rule="all_success")
```

All trigger rules: `all_success`, `all_failed`, `all_done`, `one_success`, `one_failed`, `none_failed`, `none_failed_min_one_success`, `none_skipped`, `dummy`.

**Conditional Execution:**
```python
from airflow.utils.branch_datetime import BranchDateTimeOperator
from airflow.utils.branch_python import BranchPythonOperator

# Branch based on Python logic
def choose_path(ti):
    data_size = ti.xcom_pull(task_ids='check_size')
    if data_size > 1_000_000:
        return 'process_large'
    return 'process_small'

branch = BranchPythonOperator(
    task_id="branch",
    python_callable=choose_path,
    provide_context=True
)

# Join branches
join = EmptyOperator(task_id="join", trigger_rule="none_failed")
```

**Dynamic Dependencies (Task Mapping):**
```python
# Airflow 2.3+ dynamic task mapping
file_list = list_files_task.execute()
process_task.expand(input_file=file_list)
```

**Principal-level considerations:**
- Use `TaskGroup` for visual organization of related tasks
- Prefer explicit `>>` over `set_upstream()` for readability
- Use `trigger_rule="none_failed_min_one_success"` for cleanup tasks that should run even if some tasks are skipped
- Branch operators skip all unselected branches — downstream tasks need appropriate trigger rules
- Cross-DAG dependencies are not native; use ExternalTaskSensor, trigger DAG via REST API, or event-driven patterns (Kafka/SNS)

---

## Q6. XCom — Data Passing Between Tasks

### Question
How does XCom work? What are its limitations at scale? How do you handle large data exchanges between tasks?

### Answer

**XCom Basics:**
```python
# Push (return value is auto-pushed)
def my_func(**context):
    return {"result": 42}

# Explicit push
def my_func(**context):
    context['task_instance'].xcom_push(key='data', value='hello')

# Pull in downstream task
def my_func(**context):
    data = context['task_instance'].xcom_pull(
        task_ids='upstream_task', key='data'
    )
```

**Limitations:**
- Default value limit: **1 MB** (configurable via `max_exception_length`)
- Stored in metadata DB (PostgreSQL/MySQL) — every XCom is a DB write
- Values are serialized as JSON (must be JSON-serializable)
- No built-in compression
- Querying XComs across many tasks creates many DB queries

**Large Data Workarounds:**

**1. Store Reference, Not Data:**
```python
def write_to_s3(**context):
    s3_key = f"data/{context['ds']}/output.parquet"
    # Write actual data to S3
    context['task_instance'].xcom_push(key='s3_key', value=s3_key)
    # Downstream tasks read from S3 using the key
```

**2. Custom XCom Backend:**
```python
# Store XComs in S3 instead of DB
xcom_backend = 'airflow.providers.amazon.aws.xcom.s3.AWSS3XComBackend'
xcom_backend_kwargs = {"bucket": "airflow-xcoms", "base_path": "xcom"}
```

**3. Use File System or Object Storage for Large Payloads:**
- Write data to S3/GCS/local volume
- Pass only the path/reference via XCom
- Downstream tasks read from storage

**4. Database Table as Intermediate:**
```python
# Write results to a known table partition
# Downstream reads from that partition
```

**Principal-level XCom strategy:**
- Use XCom for small metadata only (file paths, row counts, flags)
- Never pass DataFrames, large lists, or binary data via XCom
- For task mapping (Airflow 2.3+), arguments are stored as XComs — be mindful of volume
- Monitor XCom table growth; implement retention policies
- Consider the custom XCom backend for high-throughput pipelines with many tasks

---

## Q7. Backfill and Catchup

### Question
Explain Airflow's backfill mechanism. What is `catchup`? How do you handle backfill at scale without impacting production DAGs?

### Answer

**Catchup:** When `catchup=True` (default in older versions; default is `False` in 2.3+), Airflow creates DAG runs for **every interval** between `start_date` and now. This can create thousands of queued runs.

```python
DAG(
    "my_dag",
    start_date=datetime(2024, 1, 1),
    catchup=False,  # Only run for the most recent interval
    schedule="@daily"
)
```

**Backfill:** Running a DAG for historical dates.

```bash
# CLI backfill (Airflow 1.x style, deprecated)
airflow backfill my_dag -s 2024-01-01 -e 2024-06-30

# Airflow 2.x: use backfill command
airflow dags backfill my_dag -s 2024-01-01 -e 2024-06-30
```

**Backfill vs Regular Runs:**
- Backfill runs have `external_trigger=True`
- They can use different resource allocations
- `depends_on_past=True` behaves differently during backfill

**Scale Challenges:**
- Backfill creates many parallel DAG runs, saturating the DB and workers
- Scheduler slows down with thousands of pending task instances
- Workers may exhaust memory/connections

**Principal-level backfill strategy:**
```bash
# Control concurrency
airflow dags backfill my_dag \
  -s 2024-01-01 -e 2024-06-30 \
  --parallel \
  --subdir dags/etl/ \
  --donot-pool

# Use a dedicated Airflow environment for large backfills
# Or use separate Airflow instances for backfill vs production
```

**Production-safe backfill patterns:**
1. **Separate Airflow Instance** for backfill workloads
2. **Staggered Backfill** — run in batches of days
3. **Lower Concurrency** — reduce `parallelism` and `dag_concurrency` during backfill
4. **Use `max_active_runs`** to limit concurrent DAG runs
5. **Backfill with different task resources** — use a task decorator or environment variable to allocate more resources during backfill

```python
import os
is_backfill = os.environ.get("AIRFLOW_IS_BACKFILL", "False") == "True"
spark_config = {
    "executor_cores": 8 if is_backfill else 4,
    "num_executors": 20 if is_backfill else 10,
}
```

---

## Q8. Concurrency Controls and Parallelism

### Question
How do you control concurrency in Airflow? Explain the hierarchy of parallelism settings and how they interact.

### Answer

**Concurrency Hierarchy (from broad to specific):**

```ini
# airflow.cfg - Global
[core]
parallelism = 1000              # Max tasks running across ALL DAGs
dag_concurrency = 50            # Max task instances per DAG
max_active_tasks_per_dag = 50   # (Airflow 2.3+) Max active tasks per DAG

[celery]
worker_concurrency = 16         # Tasks per worker
workers = 16                    # Number of worker processes

[scheduler]
max_tis_per_query = 512         # Task instances fetched per scheduler loop
```

**Per-DAG Controls:**
```python
DAG(
    "my_dag",
    max_active_tasks=20,        # Max tasks running across all runs of this DAG
    max_active_runs=3,          # Max concurrent DAG runs
    concurrency=10,             # (deprecated, use max_active_tasks)
)
```

**Per-Task Controls:**
```python
# Resource-based limiting via Pools
SparkSubmitOperator(
    task_id="heavy_spark",
    pool="spark_pool",          # Limit concurrent tasks in this pool
    queue="spark_queue",        # Route to specific workers (Celery)
)
```

**Resource Pools (Production Feature):**
```sql
-- Create pools in the DB or via UI
-- Pool: spark_pool, Slots: 5
-- Pool: snowflake_pool, Slots: 10
-- Pool: default_pool, Slots: -1 (unlimited)
```

**Principal-level concurrency strategy:**
- Set `parallelism` based on worker capacity: `workers × worker_concurrency × 0.8`
- Use `max_active_tasks` to prevent one DAG from starving others
- Use **Pools** to cap expensive resources (Spark, Snowflake warehouses)
- Monitor `airflow_scheduler_run_duration` — high values indicate scheduler overload
- For backfills, reduce global parallelism temporarily

**Kubernetes-specific:**
```ini
[kubernetes]
worker_pods_concurrency = 4
namespace = airflow
```
Each K8s worker pod can run multiple tasks; control total pods via HPA.

---

## Q9. Scheduler Internals and High Availability

### Question
How does the Airflow scheduler work internally? How do you achieve HA scheduling? What are the common scheduler bottlenecks?

### Answer

**Scheduler Loop:**
1. Parse/refresh DAG definitions (import DAG files)
2. Create DAG runs based on schedule
3. Create task instances for DAG runs
4. Identify ready tasks (all upstream satisfied)
5. Execute ready tasks via Executor

Each loop iteration takes ~1-5 seconds for small deployments, 10-60s for large ones.

**Multiple Scheduler HA:**
```bash
# Run multiple scheduler processes (Airflow 2.x)
airflow scheduler  # start multiple instances
```
Schedulers coordinate via DB-level locks (advisory locks in PostgreSQL). Only one scheduler processes a given DAG at a time.

**HA Configuration:**
```ini
[scheduler]
run_duration = 14400           # How long a scheduler runs before restarting
min_file_process_interval = 30 # Re-scan DAG folder every 30s
num_run_loop_iterations = 100  # Restart after 100 iterations
```

**Scheduler Bottlenecks:**
1. **DAG Parsing Time** — large DAG files with complex imports slow down every cycle. Mitigate with `dagbag_import_timeout` and lazy loading.
2. **Database Lock Contention** — multiple schedulers fighting for the same DB locks. Use PostgreSQL over MySQL for better locking.
3. **Too Many DAGs** — each DAG file adds parsing overhead. Target <500 active DAGs.
4. **Too Many Task Instances** — the scheduler queries `task_instance` table; millions of rows slow planning. Implement history cleanup.
5. **Executor Communication** — Celery broker lag or K8s API throttling.

**Principal-level optimizations:**
- Use `dag_processor_manager_class_name` with `AsyncFileProcessorManager` (Airflow 2.3+)
- Separate DAG parsing from scheduling via worker processes
- Implement DAG file organization: only parse changed files
- Monitor `scheduler_heartbeat` and `sched_loop` metrics
- Set `max_dagruns_per_query` to limit DB load
- Use `simple_schedule_table` (Airflow 2.4+) for faster schedule calculation

---

## Q10. Data Quality and Testing in Pipelines

### Question
How do you implement data quality checks in Airflow pipelines? Compare approaches and tool integrations.

### Answer

**Level 1: Inline Assertions (Basic)**
```python
def validate_row_count(**context):
    count = context['ti'].xcom_pull(task_ids='load_task', key='row_count')
    if count < 1000:
        raise AirflowException(f"Row count {count} below threshold")

validate = PythonOperator(task_id="validate", python_callable=validate_row_count)
```

**Level 2: Great Expectations Integration**
```python
from airflow.providers.apache.beam.operators.great_expectations import (
    GreatExpectationsOperator
)

ge_task = GreatExpectationsOperator(
    task_id="validate_data",
    dataset={"path": "/data/output.parquet"},
    suite={"path": "expectations/suite.json"},
    store_context=False,
)
```

**Level 3: SodaCL / Soda Core**
```python
soda_check = BashOperator(
    task_id="soda_check",
    bash_command="soda scan --vars-file vars.json datastore snowflake checks/gold.yml"
)
```

**Level 4: Dedicated Data Quality Framework (Principal-level)**
```python
# Separate DAG for data quality that runs after all ETL DAGs
dag = DAG("data_quality_checks", schedule="@daily")

checks = PythonOperator(
    task_id="run_all_checks",
    python_callable=lambda: run_quality_checks(
        tables=["orders", "customers", "products"],
        checks=["not_null", "unique", "freshness", "row_count_range"]
    ),
    on_failure_callback=escalate_to_team,
)
```

**Quality Check Types:**
- **Schema validation** — columns exist, types match
- **Constraint checks** — not null, unique, foreign key
- **Freshness** — data arrives within SLA
- **Volume** — row count within expected range
- **Distribution** — values within expected range
- **Cross-table** — joins and aggregations match

**Principal strategy:**
- Run data quality in a **separate DAG** to not block downstream pipelines
- Use **progressive quality**: basic checks inline (block pipeline), deep checks async (alert only)
- Integrate with **OpenLineage** for automated metadata-driven quality checks
- Use **DQML** (Data Quality Markup Language) for declarative checks
- Implement **quality gates** — stop pipeline if critical checks fail
- Store quality metrics for trend analysis and SLA reporting

---

## Q11. SLAs and Alerting

### Question
How do you implement and manage SLAs in Airflow? What's the difference between SLA, execution_timeout, and retry logic?

### Answer

**SLA Miss:**
```python
PythonOperator(
    task_id="etl_load",
    sla=timedelta(hours=2),  # Must complete within 2 hours of scheduled start
)
```
SLA is measured from the **scheduled start time** of the DAG run, not the task's actual start. It's a soft deadline — missing SLA does NOT fail the task.

**Timeouts (Hard Limits):**
```python
PythonOperator(
    task_id="etl_load",
    execution_timeout=timedelta(hours=1),  # Kill task after 1 hour of execution
    retry_timeout=timedelta(hours=0.5),    # Wait 30 min between retries
)
```

**Comparison:**

| Feature | When it starts | Effect on failure | Use case |
|---|---|---|---|
| SLA | Scheduled start time | Trigger callback only | Soft deadline, alerting |
| execution_timeout | Task start time | Kill & fail task | Hard limit, prevent zombies |
| retry | After task failure | Retry up to max_retries | Transient failures |
| retry_exponential_backoff | Between retries | Space out retries | Rate-limited APIs |

**Production Strategy:**
```python
dag = DAG(
    "etl_pipeline",
    schedule="@daily",
    default_args={
        "retries": 3,
        "retry_delay": timedelta(minutes=5),
        "retry_exponential_backoff": True,
        "max_retry_delay": timedelta(hours=1),
        "execution_timeout": timedelta(hours=2),
        "sla": timedelta(hours=3),
    },
    sla_miss_callback=notify_oncall,
)
```

**SLA Miss Callbacks:**
```python
def sla_miss_callback(dag, dilays, slugification_dag_params):
    for task, delay in dilays.items():
        send_slack(f"SLA missed: {dag.dag_id}.{task} by {delay}")
        create_jira_ticket(dag.dag_id, task, delay)
```

**Principal-level approach:**
- Use SLA for **monitoring and alerting only**; don't rely on it for logic
- Use `execution_timeout` as the **hard circuit breaker**
- Set `retry_delay` with exponential backoff for transient failures (network, API throttling)
- Use different retry counts for different failure types (network: 5 retries, data quality: 0 retries)
- Track SLA compliance as a KPI (percentage of tasks meeting SLA per month)
- Implement **adaptive SLAs** based on historical performance

---

## Q12. Callbacks and Lifecycle Hooks

### Question
Explain Airflow's callback system. What are on_success, on_failure, on_retry callbacks? How do you use them in production?

### Answer

**Task-level Callbacks:**
```python
PythonOperator(
    task_id="load_data",
    python_callable=load_data,
    on_success_callback=notify_success,
    on_failure_callback=notify_failure,
    on_retry_callback=notify_retry,
)
```

**DAG-level Callbacks:**
```python
dag = DAG(
    "my_dag",
    on_success_callback=dag_success,
    on_failure_callback=dag_failure,
    sla_miss_callback=sla_miss_handler,
)
```

**DAG-level with on_success_rule / on_failure_rule (Airflow 2.4+):**
```python
dag = DAG(
    "my_dag",
    on_success_rule=DagState.SUCCESS,
    on_failure_rule=DagState.FAILED,
)
```

**Production Callback Patterns:**

**1. Notification Callbacks:**
```python
def notify_failure(context):
    dag_id = context['dag'].dag_id
    task_id = context['task'].task_id
    execution_date = context['execution_date']
    
    message = (
        f"FAILED: {dag_id}.{task_id}\n"
        f"Run: {execution_date}\n"
        f"Worker: {context.get('worker', 'N/A')}\n"
        f"Error: {context.get('exception', '')}"
    )
    
    send_slack(alert_channel, message, priority="P1")
    if context.get('exception'):
        create_pagerduty_incident(message)
```

**2. State-based Retry Logic:**
```python
def smart_retry(context):
    attempt = context['ti'].try_number
    if attempt >= 3:
        # Escalate after 3 retries
        escalate_to_team(context['dag'].dag_id)

def notify_retry(context):
    attempt = context['ti'].try_number
    logging.warning(f"Retry {attempt} for {context['task'].task_id}")
```

**3. Cleanup on Failure:**
```python
def cleanup_on_failure(context):
    # Clean up temp files, stop Spark jobs, release resources
    s3_key = context['ti'].xcom_pull(task_ids='upload_task', key='s3_key')
    if s3_key:
        delete_from_s3(s3_key)
```

**Principal considerations:**
- Callbacks run **synchronously** in the worker process — don't block on long operations
- Use async callbacks or message queues for heavy notification logic
- Callbacks are NOT retried — make them idempotent and fault-tolerant
- Test callbacks in isolation — they run in a different context than the task
- Log everything in callbacks — they are the last source of truth for failures
- Use `dag_run_conf` for run-specific alerting preferences

---

## Q13. Logging and Observability

### Question
How do you implement logging in Airflow? What are the remote logging options? How do you troubleshoot failed tasks at scale?

### Answer

**Local Logging (default):**
- Logs stored on each worker at `BASE_LOG_FOLDER/dag_id/run_id/task_id/try_number.log`
- Default: `/tmp/airflow/logs`
- Problem: logs lost when worker restarts or pod is rescheduled

**Remote Logging (Production Standard):**
```ini
[core]
remote_logging = True
remote_log_conn_id = aws_logs_connection
remote_base_log_folder = s3://airflow-logs/prod
```

**Supported Remote Backends:**
- **S3** — most common for AWS deployments
- **GCS** — for GCP deployments  
- **HDFS** — for on-prem/Hadoop
- **Elasticsearch** — for centralized search
- **CloudWatch Logs** — for AWS-native observability

**Log Aggregation (Principal-level):**
```ini
# Ship logs to ELK stack
[core]
remote_logging = True
remote_log_class = airflow.providers.elasticsearch.log.elasticsearch_task_handler.ElasticsearchTaskHandler
remote_log_conn_id = elasticsearch_conn

# Alternatively use Fluentd/Filebeat sidecar in K8s
```

**Structured Logging:**
```python
import logging
import json

logger = logging.getLogger(__name__)

def log_structured(event, **kwargs):
    log_entry = {"event": event, "dag_id": "{{ dag.dag_id }}", **kwargs}
    logger.info(json.dumps(log_entry))
```

**Troubleshooting at Scale:**
1. **Airflow UI** — view task logs directly (if remote logging configured)
2. **ELK / Kibana** — search across all task logs with filters
3. **CloudWatch Logs Insights** — AWS-native log querying
4. **Datadog / New Relic** — APM integration for end-to-end traces
5. **OpenTelemetry** — distributed tracing across Airflow + downstream services

**Principal-level observability stack:**
- Remote logs (S3/GCS) as the source of truth
- ELK/Grafana Loki for log search
- Prometheus + Grafana for metrics
- OpenTelemetry for distributed tracing
- Custom dashboards: DAG success rate, avg task duration, SLA compliance

---

## Q14. Variables and Config Management

### Question
How do you manage configuration in Airflow? Compare Variables, Connections, config files, and environment variables.

### Answer

**Variables (Key-Value Config):**
```python
from airflow.models import Variable

# Set
Variable.set("snowflake_warehouse", "ETL_WH")
# Get
warehouse = Variable.get("snowflake_warehouse", default_var="SMALL_WH")
# Get with JSON parsing
config = Variable.get("pipeline_config", deserialize_as=dict)
```

**Connections (Structured Config):**
```python
from airflow.hooks.base import BaseHook

conn = BaseHook.get_connection("snowflake_prod")
config = {
    "account": conn.login,
    "password": conn.password,
    "database": conn.schema,
    "warehouse": conn.extra.get("warehouse"),
}
```

**Environment Variables:**
```bash
AIRFLOW_CONN_MY_CONN=postgres://user:pass@host/db
AIRFLOW_VAR_MY_VAR=value
```

**Config File (airflow.cfg):**
```ini
[core]
sql_alchemy_conn = postgresql+psycopg2://airflow:pass@db:5432/airflow
```

**Principal-level config strategy:**

| Type | Use For | Storage | Security |
|---|---|---|---|
| Connections | Service endpoints, DBs | Vault/AWS SM | Encrypted |
| Variables | Runtime config (thresholds, flags) | Vault/AWS SM | Encrypted |
| Env Vars | Deployment config (log level, env name) | K8s ConfigMap / Helm | Low sensitivity |
| airflow.cfg | Infrastructure config | Git (no secrets) | No secrets |

**Multi-environment pattern:**
```python
import os

env = os.getenv("AIRFLOW_ENV", "dev")
prefix = f"{env}."

warehouse = Variable.get(f"{prefix}snowflake_warehouse")
threshold = Variable.get(f"{prefix}row_count_threshold", default_var=100)
```

**Best practices:**
- Never store credentials as Variables — use Connections
- Use `default_var` to avoid KeyError
- Version all config changes alongside DAG code
- Use secrets backend for all sensitive values
- Document all variables/connections in a central README
- Use naming conventions: `env.component.key`
---

## Q15. REST API — Programmatic DAG Triggers and Management

### Question
How do you use the Airflow REST API to trigger DAGs, check status and manage runs programmatically?

### Answer

**Authentication:**
```bash
curl -X POST "http://airflow:8080/api/v1/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "api_user", "password": "pass"}'
# Returns JWT token
```

**Trigger a DAG Run:**
```bash
curl -X POST "http://airflow:8080/api/v1/dags/etl_pipeline/dagRuns" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"conf": {"source_system": "salesforce", "date_range": "2024-01-01"} }'
```

**Check DAG Run Status:**
```bash
curl -X GET "http://airflow:8080/api/v1/dags/etl_pipeline/dagRuns/{dag_run_id}" \
  -H "Authorization: Bearer <token>"
```

**Get Task Status:**
```bash
curl -X GET "http://airflow:8080/api/v1/dags/etl_pipeline/dagRuns/{dag_run_id}/tasks/{task_id}"
```

**Python Integration:**
```python
import requests

AIRFLOW_URL = "http://airflow:8080"

def trigger_dag(dag_id, conf=None):
    resp = requests.post(f"{AIRFLOW_URL}/api/v1/login", json={"username": "api", "password": os.getenv("AIRFLOW_API_PASS")})
    token = resp.json()["access_token"]
    headers = {"Authorization": f"Bearer {token}"}
    
    payload = {"conf": conf or {}}
    resp = requests.post(f"{AIRFLOW_URL}/api/v1/dags/{dag_id}/dagRuns", json=payload, headers=headers)
    resp.raise_for_status()
    return resp.json()
```

**Principal considerations:**
- Use basic auth with API tokens for service-to-service calls (not user login)
- Rate-limit external callers; Airflow API has no built-in throttling
- Cache tokens; re-authenticate on 401
- For high-volume triggers, consider **Kafka-based trigger** instead of REST API polling
- API v2 is the stable version in Airflow 2.x

---

## Q16. Sensors — Waiting for External Events

### Question
What are Airflow Sensors? How do they differ from regular operators? What are soft vs hard deadlines?

### Answer

**Sensors are operators that wait/poke for an external condition:**
```python
# Wait for a file to appear in S3
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

wait_for_file = S3KeySensor(
    task_id="wait_for_input",
    bucket_name="data-lake",
    bucket_key="incoming/sales.csv",
    wildcard_match=True,
    poke_interval=60,     # Check every 60s
    timeout=3600,         # Give up after 1 hour
)
```

**Sensor vs Operator:**
| Sensor | Operator |
|---|---|
| Runs in a loop (poke) until condition met | Runs once |
| Consumes worker slot while waiting | Consumes worker slot while running |
| Has `poke_interval` and `timeout` | No built-in waiting logic |

**Common Sensors:**
- `ExternalTaskSensor` — wait for a task in another DAG
- `TimeDeltaSensor` — wait a fixed duration
- `SqlSensor` — wait for a SQL query to return True
- `S3KeySensor` / `GCSObjectExistenceSensor` — wait for files
- `HttpSensor` — wait for an HTTP endpoint to return success

**Soft vs Hard Deadline (Airflow 2.x):**
```python
ExternalTaskSensor(
    task_id="wait_for_upstream",
    external_dag_id="source_dag",
    external_task_id="extract",
    execution_delta=timedelta(days=1),
    mode="reschedule",  # Release worker while waiting
    soft_fail=True,     # Skip downstream tasks instead of failing
    timeout=7200,
)
```

**`mode="reschedule"` vs `mode="poke"`:**
- **poke**: keeps a worker slot occupied while polling (wastes resources)
- **reschedule**: releases the worker; the task is rescheduled on each poke interval (saves resources)

**Principal strategy:**
- Always use `mode="reschedule"` for sensors that wait > a few minutes
- Use `soft_fail=True` when the upstream data might not arrive — skip gracefully instead of failing
- Avoid sensors where possible: event-driven triggers (SQS, Kafka, webhook) are more efficient
- For cross-DAG dependencies, prefer `ExternalTaskMarker` (Airflow 2.3+) over `ExternalTaskSensor`

---

## Q17. TaskGroups and DAG Organisation

### Question
What are TaskGroups? How do they improve DAG readability and manageability?

### Answer

**TaskGroups (Airflow 2.0+) group related tasks into a visual unit:**
```python
with DAG("etl_pipeline", schedule="@daily") as dag:
    
    with TaskGroup("extract_group") as extract:
        extract_s3 = PythonOperator(task_id="from_s3", python_callable=extract_s3)
        extract_api = PythonOperator(task_id="from_api", python_callable=extract_api)
    
    with TaskGroup("transform_group") as transform:
        clean = PythonOperator(task_id="clean", python_callable=clean_data)
        enrich = PythonOperator(task_id="enrich", python_callable=enrich_data)
    
    with TaskGroup("load_group") as load:
        load_warehouse = PythonOperator(task_id="to_snowflake", python_callable=load_data)
    
    extract >> transform >> load
```

**Benefits:**
- Visual grouping in the Airflow UI
- Share upstream/downstream relationships at group level
- Nested TaskGroups for complex DAGs
- Collapse/expand in the Graph View

**Cross-group dependencies:**
```python
# All tasks in extract_group are upstream of all tasks in transform_group
extract_group >> transform_group

# Or connect specific tasks across groups
extract_api >> clean
```

**Principal use:**
- Replace manual task naming conventions (`extract_`, `transform_`, `load_`) with actual visual groups
- Use TaskGroups to model **stages** or **sub-pipelines**
- Combine with `expand()` for dynamic tasks within a group
- Use nested TaskGroups when you have sub-stages within a stage

---

## Q18. Dynamic Task Mapping

### Question
How does dynamic task mapping work in Airflow 2.3+? How is it different from SubDAGs and dynamic DAG generation?

### Answer

**Dynamic Task Mapping (Airflow 2.3+):**
```python
class GetFiles(PythonOperator):
    def execute(self, context):
        files = list_s3_files("data-lake/incoming/")
        return files

class ProcessFile(PythonOperator):
    def execute(self, context):
        filepath = self.op_args[0]  # Mapped parameter
        process_file(filepath)

get_files = GetFiles(task_id="get_files")
process = ProcessFile.expand(
    task_id="process_file",
    filepath=get_files.output  # Maps to one task per file
)

get_files >> process
```

**Key differences from alternatives:**

| Feature | Task Mapping | SubDAGs (deprecated) | Dynamic DAGs |
|---|---|---|---|
| When tasks are created | At DAG run time | At parse time | At parse time |
| Scale | Per-task | Per-DAG | Per-DAG |
| UI visibility | Individual mapped tasks | Single SubDAG task | All rendered at parse |
| Concurrency control | Per-mapped-task | Per-SubDAG | Per-DAG |
| Status tracking | Per-mapped task | Aggregate | Per-task |

**Mapping with partial expansion:**
```python
# Map over a subset of parameters
process = ProcessFile.partial(
    task_id="process_file",
    transform="standard"  # Fixed value
).expand(
    filepath=get_files.output  # Mapped value
)
```

**Limitations:**
- Mapped tasks cannot be downstream of unmapped tasks within the same group without `set_upstream()` / `set_downstream()`
- Downstream tasks must use `expand()` too if they depend on mapped outputs
- XCom size limits apply to the list being mapped

**Principal guidance:**
- Use task mapping instead of SubDAGs (which are deprecated in Airflow 2.3+)
- Task mapping is better than dynamic DAG generation because it doesn't require re-parsing
- Set `max_map_length` to cap the number of mapped tasks
- Use `do_xcom_push=True` carefully — each mapped task stores its own XCom

---

## Q19. DAG CI/CD — Testing and Deployment

### Question
How do you set up CI/CD for Airflow DAGs? What tests should run? How do you deploy DAGs to multiple environments?

### Answer

**Test Suite:**

**1. DAG Validation Tests:**
```python
# tests/dags/test_dag_validation.py
import pytest
from airflow.models import BaseOperator

@pytest.mark.parametrize("dag_id", ["etl_pipeline", "data_quality"])
def test_dag_no_import_errors(dag_id):
    dagbag = get_dag_bag()
    assert dag_id in dagbag.dags
    dag = dagbag.get_dag(dag_id)
    assert dag is not None
```

**2. DAG Structure Tests:**
```python
# tests/dags/test_dag_structure.py
def test_etl_dag_task_count():
    dag = get_dag("etl_pipeline")
    assert len(dag.tasks) >= 5

def test_upstream_downstream():
    dag = get_dag("etl_pipeline")
    ti = dag.get_task("extract")
    assert dag.get_task("transform") in ti.downstream_list
```

**3. Unit Tests for Operators:**
```python
def test_custom_operator():
    op = MyCustomOperator(task_id="test", input_data={"key": "val"})
    result = op.execute(context={})
    assert result["status"] == "success"
```

**CI Pipeline (GitHub Actions):**
```yaml
# .github/workflows/dag-ci.yml
name: DAG CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: pip install -r requirements.txt pytest pytest-airflow
      - name: Test DAGs
        run: pytest tests/dags/
      - name: Lint DAGs
        run: pip install flake8 && flake8 dags/
```

**Deployment Strategy:**
```yaml
# Helm values for multi-environment
environments:
  dev:
    image: airflow:2.8.0-dev
    dag_folder: dags
    config: airflow-dev.cfg
  prod:
    image: airflow:2.8.0
    dag_folder: dags
    config: airflow-prod.cfg
```

**Principal-level CI/CD:**
- Parse all DAGs in CI to catch import errors before deployment
- Test with `@pytest.mark.airflow_db` for DB-dependent tests
- Use separate DAG repositories per environment if configs differ significantly
- Blue/green deployment: deploy to new worker pool, switch traffic
- Tag all DAG images with the git commit hash
- Use `astronomer registry publish` for DAG packages
- Run DAGs through a **staging environment** before production

---

## Q20. Testing Strategies for Airflow Pipelines

### Question
What are the different testing strategies for Airflow? How do you test DAGs, operators and sensors in isolation?

### Answer

**Test Pyramid for Airflow:**

**Level 1: Unit Tests (Operators):**
```python
def test_load_data_operator():
    context = {"ti": MagicMock(), "execution_date": datetime(2024, 1, 1)}
    context["ti"].xcom_pull = MagicMock(return_value=1000)
    
    op = LoadDataOperator(task_id="load", threshold=500)
    result = op.execute(context)
    
    assert result["rows_loaded"] > 500
```

**Level 2: DAG Structure Tests:**
```python
@pytest.mark.parametrize("dag_id,expected_tasks", [
    ("etl_pipeline", 8),
    ("data_quality", 3),
])
def test_dag_has_tasks(dag_id, expected_tasks):
    dag = get_dag(dag_id)
    assert len(dag.tasks) == expected_tasks
```

**Level 3: Integration Tests (with test DB):**
```python
@pytest.mark.airflow_db
def test_dag_runs_successfully():
    from airflow.utils.db import session
    with create_session() as session:
        # Test against a real (test) Airflow DB
        dr = get_dag_run("etl_pipeline", session)
        assert dr is not None
```

**Level 4: End-to-end Tests:**
```python
# Test with a real Airflow instance in Docker
def test_full_pipeline():
    trigger_dag("etl_pipeline")
    wait_for_dag_run("etl_pipeline", expected_state="success", timeout=600)
```

**Mocking External Services:**
```python
from unittest.mock import patch

@patch("airflow.providers.amazon.aws.hooks.s3.S3Hook.get_conn")
def test_s3_operator(mock_s3_conn):
    mock_s3_conn().download_file.return_value = True
    op = S3DownloadOperator(task_id="download", bucket="test", key="file.csv")
    op.execute({})
```

**Principal strategy:**
- Aim for >80% unit test coverage on custom operators
- Test DAG structure (task count, dependencies, schedule) in CI
- Use `pytest-airflow` plugin for Airflow-specific fixtures
- Mock all external hooks (S3, DB, HTTP) in unit tests
- Run full DAG tests against a local Airflow instance in staging
- Never test against production data; use synthetic or anonymised data
- Test failure paths: what happens when an upstream sensor times out?
---

## Q21. Monitoring and Metrics at Scale

### Question
How do you monitor Airflow at scale? What metrics matter? What dashboards do you build?

### Answer

**Built-in Metrics (StatsD):**
```ini
[metrics]
statsd_on = True
statsd_host = localhost
statsd_port = 8125
statsd_prefix = airflow
```

Airflow emits 50+ StatsD metrics: scheduler loop duration, task durations, queue depth, etc.

**Key Metrics to Track:**
| Metric | Meaning | Alert Threshold |
|---|---|---|
| `scheduler.slotbeats` | Scheduler health | Absence = down |
| `scheduler.dagbag.import_errors` | DAG parse failures | > 0 |
| `scheduler.tasks.running` | Active tasks | > available slots |
| `scheduler.tasks.failed` | Failed tasks | > 5 per hour |
| `executor.tasks.running` | Executor capacity | > 90% capacity |
| `task.duration` | Task execution time | > p95 historical |
| `dag.run.success_rate` | DAG success % | < 95% |
| `db.pool.usage` | DB connection pool | > 80% |

**Prometheus + Grafana Stack:**
```ini
[metrics]
statsd_on = True
statsd_host = prometheus-statsd-exporter
statsd_port = 9125
statsd_prefix = airflow
```

**Grafana Dashboard Panels:**
1. DAG Success Rate (by DAG, by day)
2. Task Duration (p50, p95, p99 by DAG)
3. Active Tasks / Slots Used / Available
4. Failed Tasks (by DAG, by error type)
5. Scheduler Loop Duration
6. DB Connection Pool Usage
7. SLA Misses Over Time

**Principal-level observability:**
- Airflow's built-in metrics cover infrastructure; build custom metrics for business logic
- Track data freshness as a metric: `last_success_timestamp` vs `current_time`
- Use **custom StatsD metrics** in operators:
```python
from airflow.utils.metrics import UsageTimeConfig
from timeit import default_timer

start = default_timer()
result = do_work()
duration = default_timer() - start
self.get_metrics_logger().increment("custom.etl.rows_processed", result.row_count)
self.get_metrics_logger().timing("custom.etl.duration", duration)
```
- Alert on **anomalies** (sudden increase in task duration) not just thresholds
- Correlate Airflow metrics with downstream system metrics (DB load, API latency)
- Build a **pipeline health score** dashboard: weighted combination of SLA compliance, data freshness, error rate

---

## Q22. Security — Authentication, Authorization, Secrets

### Question
How do you secure an Airflow deployment? Cover authentication, RBAC, secrets management and network security.

### Answer

**Authentication:**
```ini
[ldap]
bind_user = cn=manager,dc=example,dc=com
bind_password = from_secrets
basedn = dc=example,dc=com

# Or use OAuth (Google, GitHub, Okta) via FAB (Flask-AppBuilder)
[auth]
backend = airflow.providers.fab.auth.managers.fab_auth_manager.FabAuthManager
```

**RBAC (Role-Based Access Control):**
- Built into Airflow 2.x Web UI
- Roles: Admin, User, Viewer, Op, Public
- Custom roles with permission filters:
```python
# plugins/perm_filter.py
from flask_appbuilder.security.sqla.models import Permission, Role

def on_role_created(role):
    perms = {
        "etl_team": ["can_read", "can_edit", "can_delete"],
        "analytics_team": ["can_read"],
    }
```

**Per-DAG Permissions:**
```python
dag = DAG(
    "sensitive_pipeline",
    access_control=[
        {"role": "Admin", "perms": ["can_read", "can_edit", "can_delete"]},
        {"role": "ETL_Team", "perms": ["can_read", "can_edit"]},
    ],
)
```

**Secrets Backend (Production):**
```ini
[core]
# AWS Secrets Manager
secrets_backend = airflow.providers.amazon.aws.secrets.backends.secrets_manager.SecretsManagerBackend
secrets_backend_conn_id = aws_conn

# HashiCorp Vault
secrets_backend = airflow.providers.hashicorp.secrets.vault.AirflowVaultBackend
secrets_backend_kms_key = airflow
```

```bash
# Env var connection
AIRFLOW_CONN_MY_CONN=postgres://user:pass@host:5432/db

# Vault
AIRFLOW__CORE__SECRETS__BACKEND = vault
```

**Network Security:**
- Airflow Webserver behind a reverse proxy (NGINX, ALB) with TLS termination
- Internal network for worker-to-worker and worker-to-DB communication
- Restrict worker pods to only access required services (network policies)
- Disable the webserver in dev environments; use CLI only

**Principal-level security:**
- Use **short-lived credentials** via IAM roles for service accounts (IRSA on AWS)
- Rotate secrets automatically (secrets backend handles this)
- Enable audit logging for all DAG runs and user actions
- Implement **DAG-level permissions** so teams only see their own DAGs
- Run Airflow components with least-privilege OS users
- Encrypt the metadata database at rest
- Use mTLS for inter-service communication in K8s deployments
- Regularly audit connections and variables for hardcoded secrets
- Implement **DAG file signing** to prevent unauthorised DAG deployment

---

## Q23. KubernetesExecutor — Architecture and Tuning

### Question
How does KubernetesExecutor work? What are the performance tuning knobs? What are the common failure modes?

### Answer

**Architecture:**
- Each task → a Kubernetes Pod
- Pod runs the worker image with the task's command
- Pod terminates when the task completes
- Scheduler creates/deletes pods via the K8s API

**Configuration:**
```ini
[kubernetes]
worker_container_repository = my-airflow-worker
worker_container_tag = 2.8.0
worker_container_image_pull_policy = IfNotPresent
namespace = airflow-prod
airflow_configmap = airflow-config
airflow_local_settings_configmap = airflow-local-settings
```

**Pod Resource Tuning:**
```python
KubernetesPodOperator(
    task_id="spark_job",
    name="spark-etl",
    namespace="airflow-prod",
    image="spark:3.5",
    resources=ResourceRequest(
        memory="8Gi",
        cpu="4",
        limits=Resource(
            memory="8Gi",
            cpu="4",
        ),
    ),
    env_vars={"AIRFLOW_HOME": "/airflow"},
    in_cluster=True,
)
```

**Key Tuning Parameters:**
| Parameter | Default | Recommendation |
|---|---|---|
| `worker_pods_concurrency` | 16 | Scale to cluster capacity |
| `pod_cleanup_on_success` | True | Keep for debugging initially |
| `pod_template_file` | None | Use a custom template |
| `image_pull_secrets` | None | Set for private registries |
| `log_events_on_startup` | True | Useful for debugging |

**Common Failure Modes:**
1. **Pod Scheduling Failures** — insufficient cluster resources. Solution: cluster autoscaler, resource quotas
2. **Image Pull Errors** — private registry auth issues. Solution: imagePullSecrets
3. **OOMKilled** — insufficient memory limits. Solution: monitor and right-size
4. **CrashLoopBackOff** — task code errors. Solution: check pod logs
5. **API Rate Limiting** — too many pods created. Solution: throttle concurrency
6. **Stale Pods** — cleanup failures. Solution: set `worker_pods_parallelism` and `pod_mutation_timeout`

**Principal-level K8s deployment:**
- Use a **pod template file** for consistent environment setup
- Implement **HPA** (Horizontal Pod Autoscaler) for the scheduler
- Use **node affinity** to run compute-heavy tasks on GPU/large nodes
- Set up **pod disruption budgets** for scheduler pods
- Use **sidecar containers** for log shipping (Fluentd)
- Enable **resource quotas** per namespace to prevent runaway tasks
- Monitor K8s API latency; Airflow is sensitive to API delays
- Use `KubernetesPodOperator` for tasks that need custom environments (different language runtime, GPU)

---

## Q24. Data Lineage — Marquez and OpenLineage

### Question
How do you implement data lineage tracking in Airflow? What is OpenLineage? How do you integrate with Marquez?

### Answer

**OpenLineage — Standard Lineage Protocol:**
- Vendor-neutral standard for data lineage
- Captures: inputs, outputs, job name, run ID
- Supported by Airflow via the OpenLineage plugin

**Installation:**
```bash
pip install openlineage-airflow
```

**Configuration:**
```python
# airflow_local_settings.py
from openlineage.airflow.macros import dataset

def get_custom_facets(self):
    return {"schema": get_dataset_schema(self.operator)}
```

```ini
# airflow.cfg
[openlineage]
namespace = airflow-prod
```

**Marquez Integration (Self-hosted Lineage UI):**
```ini
[openlineage]
backend = http
backend_kwargs = {"url": "http://marquez:5000/lineage"}
```

**Airflow + Marquez Architecture:**
```
Task executes → OpenLineage facet extracted → POST to Marquez API → Marquez stores lineage → Marquez UI visualises
```

**Supported Operators (automatic lineage):**
- `S3ToS3Transfer`, `PostgresToMySqlTransfer`
- `BigQueryInsertJobOperator`
- `SnowflakeOperator` (via provider plugin)
- `SparkSubmitOperator`

**Custom Operators with Lineage:**
```python
from openlineage.airflow.extractor import OperatorLineage

class MyOperator(BaseOperator):
    def get_openlineage_facets_on_start(self):
        return OperatorLineage(
            inputs=[Dataset(namespace="s3", name="bucket/input", facets={})],
            outputs=[Dataset(namespace="snowflake", name="db.schema.table", facets={})],
        )
```

**Principal considerations:**
- OpenLineage is the industry standard; don't use proprietary lineage formats
- Marquez is open-source; Amundsen (Lyft) is another option
- Lineage enables impact analysis, compliance (GDPR), and data discovery
- Combine lineage with data quality for end-to-end pipeline health
- Use lineage to auto-generate data catalogs and documentation
- Track both technical lineage (transformations) and business lineage (metrics)

---

## Q25. Resource Pools and Concurrency Control

### Question
What are Resource Pools? How do they differ from concurrency settings? When do you use them?

### Answer

**Resource Pools limit concurrent execution of specific tasks across all DAGs:**
```python
dag = DAG("my_dag", schedule="@hourly")

task1 = PythonOperator(
    task_id="heavy_computation",
    python_callable=compute,
    resource_pool="gpu_pool",  # Only 4 of these can run at once
)

task2 = PythonOperator(
    task_id="another_heavy",
    python_callable=compute2,
    resource_pool="gpu_pool",  # Shares the pool with task1
)
```

**Define pools via UI or CLI:**
```bash
airflow pools create gpu_pool 4 "GPU pool - 4 slots"
airflow pools create db_write_pool 2 "DB write pool - 2 slots"
```

**Resource Pools vs Concurrency Settings:**

| Feature | Resource Pools | dag_concurrency | max_active_tasks |
|---|---|---|---|
| Scope | Across DAGs | Per DAG | Per DAG |
| Granularity | Per pool (manual assignment) | All tasks | All tasks |
| Use case | Shared scarce resources | DAG-level parallelism | Prevent overload |
| Flexibility | High — assign any task to any pool | Low | Low |

**Principal use cases:**
- **GPU pool**: limit tasks needing GPU to available GPU count
- **DB write pool**: limit concurrent writes to a single database
- **API rate limit pool**: limit calls to a rate-limited external API
- **Cost pool**: limit expensive tasks (Spark, big Snowflake warehouses)
- **Network bandwidth pool**: limit tasks that saturate network

**Dynamic pool management:**
```python
# Adjust pool size programmatically
from airflow.models import Pool
from airflow.utils.db import provide_session

@provide_session
def adjust_pool(session=None):
    pool = Pool.get_pool("gpu_pool", session=session)
    pool.slots = 8  # Increase during low-traffic hours
    session.merge(pool)
```

---

## Q26. Cross-DAG Dependencies

### Question
How do you manage dependencies between DAGs? What are the patterns and pitfalls?

### Answer

**Pattern 1: ExternalTaskMarker (Airflow 2.3+)**
```python
# In DAG A (upstream)
from airflow.operators.bash import BashOperator

extract = BashOperator(task_id="extract", bash_command="extract.sh")

# In DAG B (downstream)
from airflow.sensors.external_task import ExternalTaskSensor

wait_for_extract = ExternalTaskSensor(
    task_id="wait_for_extract",
    external_dag_id="dag_a",
    external_task_id="extract",
    execution_delta=timedelta(days=0),
    mode="reschedule",
)
```

**Pattern 2: Trigger DAG from Another DAG**
```python
from airflow.operators.trigger_dagrun import TriggerDagRunOperator

trigger = TriggerDagRunOperator(
    task_id="trigger_downstream",
    trigger_dag_id="downstream_dag",
    conf={"date": "{{ ds }}", "source": "dag_a"},
    wait_for_completion=True,  # Wait for downstream to finish
    poke_interval=30,
)
```

**Pattern 3: Shared State via XCom / Database**
```python
# DAG A writes a flag
Variable.set(f"dag_a_{ds}_completed", "true")

# DAG B checks the flag
class WaitForFlagSensor(BaseSensorOperator):
    def poke(self, context):
        return Variable.get(f"dag_a_{context['ds']}_completed", default_var="false") == "true"
```

**Pitfalls:**
- Circular dependencies between DAGs → `AirflowDagCycleException`
- Tight coupling between DAGs → harder to maintain independently
- Sensor timeouts → DAGs stuck waiting forever
- Timezone mismatches between DAGs with different schedules
- Backfill issues: running backfills of DAG A doesn't trigger DAG B

**Principal strategy:**
- Prefer **event-driven triggers** (SQS, Kafka, webhooks) over cross-DAG sensors
- Use `TriggerDagRunOperator` for explicit orchestration
- Use `ExternalTaskMarker` for implicit dependencies (Airflow 2.3+)
- Document cross-DAG dependencies in a diagram
- Implement **circuit breakers**: if upstream is down, skip gracefully
- Consider consolidating tightly-coupled DAGs into one DAG with TaskGroups
- Use `may_fail()` for non-critical cross-DAG dependencies

---

## Q27. Multi-Tenant Airflow Deployments

### Question
How do you run Airflow in a multi-tenant environment? How do you isolate teams, DAGs and resources?

### Answer

**Approach 1: Single Deployment with RBAC**
- One Airflow instance, multiple teams
- Use `access_control` per DAG
- Namespace DAG IDs by team: `team_a.etl`, `team_b.analytics`
- Resource pools shared across teams

```python
dag = DAG(
    "team_a.etl_pipeline",
    access_control=[
        {"role": "team_a_admin", "perms": ["can_read", "can_edit", "can_delete"]},
        {"role": "team_a_user", "perms": ["can_read"]},
    ],
)
```

**Approach 2: Multiple Airflow Deployments**
- Separate Airflow instances per team/business unit
- Separate metadata databases
- Separate worker pools
- More operational overhead but full isolation

**Approach 3: K8s Namespace per Team**
```yaml
# Each team gets a K8s namespace
apiVersion: v1
kind: Namespace
metadata:
  name: airflow-team-a
```

**Resource Isolation Patterns:**
- **Resource Pools** — limit shared resource usage per team
- **K8s Resource Quotas** — cap CPU/memory per namespace
- **Database Connection Pooling** — separate pools per team
- **Separate Log Storage** — prefix logs with team name

**Principal considerations:**
- For <5 teams with <500 DAGs total: single deployment with RBAC
- For 5+ teams or compliance requirements: separate deployments
- Implement **fair scheduling** — prevent one team from monopolising workers
- Use **cost allocation tags** to attribute compute costs per team
- Implement **self-service DAG deployment** with guardrails
- Separate **connections/secrets** per team (no cross-team access)
- Build a **shared library** of operators/sensors that all teams use
---

## Q28. Migrating from Airflow 1.x to 2.x

### Question
What are the major breaking changes between Airflow 1.x and 2.x? How do you plan and execute the migration?

### Answer

**Major Breaking Changes:**

| 1.x | 2.x | Impact |
|---|---|---|
| `execution_date` = next run | `execution_date` = actual run start | All schedules shift by one interval |
| `start_date` = first run | `start_date` = when DAG becomes active | DAGs don't run until start_date |
| No built-in RBAC UI | RBAC UI default | Auth changes |
| SequentialExecutor only | Celery/K8s/Local Executors | Deployment model changes |
| No TaskGroups | TaskGroups available | DAG organisation |
| Limited REST API | Full REST API v1 | Programmatic access |

**Schedule Semantics Change (Most Impactful):**
```python
# Airflow 1.x: execution_date was the "next" run time (end of period)
# Airflow 2.x: execution_date is the actual run start time (beginning of period)

# If you schedule daily at 6:00 AM:
# 1.x: execution_date=2024-01-02T06:00 means data for Jan 1
# 2.x: execution_date=2024-01-01T06:00 means data for Jan 1

# Fix: use data_interval_start / data_interval_end
def my_task(**context):
    # Airflow 2.x way
    start = context["data_interval_start"]
    end = context["data_interval_end"]
    # These correctly represent the data period regardless of version
```

**Migration Steps:**
1. **Audit all DAGs** — identify uses of `execution_date`, `{{ ds }}`, `{{ ds_nodash }}`
2. **Update schedule references** — switch to `data_interval_start` / `data_interval_end`
3. **Update operators** — migrate from deprecated operators to 2.x equivalents
4. **Update connections** — review all connections for deprecated fields
5. **Update plugins** — migrate custom plugins to 2.x API
6. **Test in isolation** — run each DAG in a test environment
7. **Database migration** — run `airflow db migrate`
8. **Cutover** — disable 1.x, enable 2.x, monitor closely

**Principal migration strategy:**
- Run 1.x and 2.x in parallel during the transition
- Use `AIRFLOW_IS_BACKFILL=true` for backfill runs
- Set `AIRFLOW_IGNORE_TIME` to control when DAGs start running post-migration
- Back up the metadata database before migration
- Test the migration on a clone of the production database first
- Plan for a **freeze window** where no new DAGs are added
- Communicate timeline clearly to all DAG owners

---

## Q29. Airflow Providers — Plugin Architecture

### Question
What are Airflow Providers? How do they work? How do you write a custom provider or extend existing ones?

### Answer

**Providers are packages that add connectors, operators, hooks, sensors and sensors for specific services:**
```bash
pip install apache-airflow-providers-amazon    # AWS
pip install apache-airflow-providers-google    # GCP
pip install apache-airflow-providers-snowflake # Snowflake
pip install apache-airflow-providers-apache-spark
pip install apache-airflow-providers-databricks
```

**Provider Contents:**
- **Hooks** — connect to the service (authentication, client setup)
- **Operators** — execute actions on the service
- **Sensors** — wait for conditions on the service
- **Transfers** — move data between services
- **Triggers** — async waiting (Airflow 2.2+)

**Custom Plugin:**
```python
# plugins/my_plugin.py
from airflow.models import BaseOperator
from airflow.plugins_manager import AirflowPlugin

class MyCustomOperator(BaseOperator):
    def __init__(self, conn_id, **kwargs):
        super().__init__(**kwargs)
        self.conn_id = conn_id
    
    def execute(self, context):
        from airflow.hooks.base import BaseHook
        conn = BaseHook.get_connection(self.conn_id)
        # Do custom work
        return {"status": "success"}

class MyPlugin(AirflowPlugin):
    name = "my_custom_plugin"
    operators = [MyCustomOperator]
```

**Extending Existing Providers:**
```python
from airflow.providers.amazon.aws.operators.s3 import S3CopyObjectOperator

class EnhancedS3CopyOperator(S3CopyObjectOperator):
    def execute(self, context):
        # Add custom logic before/after the base operator
        self.log.info("Starting enhanced S3 copy")
        result = super().execute(context)
        self.log.info("S3 copy completed, validating...")
        self.validate_copy()
        return result
```

**Principal strategy:**
- Prefer official providers over custom implementations
- Only write custom operators when no provider exists
- Extend existing providers rather than rewriting
- Follow provider naming conventions and structure
- Document custom plugins thoroughly
- Version plugins alongside DAG code
- Contribute useful operators back to the Apache Airflow Providers project

---

## Q30. Advanced Patterns — Triggers, Deferrable Operators and Dynamic Scheduling

### Question
What are deferrable operators and triggers? How do they solve the worker slot problem? What are advanced scheduling patterns?

### Answer

**Deferrable Operators (Airflow 2.2+):**
```python
from airflow.triggers.base import BaseTrigger, TriggerEvent
from airflow.utils.state import State

class S3KeyTrigger(BaseTrigger):
    def __init__(self, bucket, key, poke_interval=5):
        super().__init__()
        self.bucket = bucket
        self.key = key
        self.poke_interval = poke_interval
    
    async def run(self):
        while True:
            if await check_s3_key(self.bucket, self.key):
                yield TriggerEvent({"status": "success", "key": self.key})
            await asyncio.sleep(self.poke_interval)

class S3KeyDeferrableOperator(BaseOperator):
    def execute(self, context):
        # Instead of blocking, defer to trigger
        raise TaskDeferred(
            trigger=S3KeyTrigger(self.bucket, self.key),
            timeout=timedelta(hours=1),
        )
    
    def execute_complete(self, context, event):
        if event["status"] == "success":
            return f"Found: {event['key']}"
        raise AirflowException("Trigger timed out")
```

**Benefits of Deferrable Operators:**
- **Release worker slot** while waiting (vs sensors that hold slots)
- **Scale to thousands** of waiting tasks without needing workers
- **Async triggers** run in the triggerer process (lightweight)

**Triggerer Process:**
```ini
[core]
triggerer_worker_concurrency = 100  # Number of triggers that can run concurrently
```

Start the triggerer: `airflow triggers`

**Advanced Scheduling Patterns:**

**1. Condition-based Scheduling (Airflow 2.4+):**
```python
dag = DAG(
    "conditional_dag",
    schedule=DataIntervalTiming(),  # Custom schedule
)

class CustomSchedule( timetable.DataIntervalTiming):
    def infer_manual_data_interval(self, run_after):
        return DataInterval(
            start=run_after - timedelta(hours=1),
            end=run_after,
        )
    
    def should_run(self, att):
        # Only run if external condition is met
        return check_external_condition()
```

**2. Cron with Dynamic Adjustment:**
```python
from pendulum import cron_expression

dag = DAG(
    "dynamic_schedule",
    schedule="@daily",  # Base schedule
    # Override in code for specific runs
)

def adjust_schedule(context):
    # Skip weekends, adjust for holidays
    if context['execution_date'].weekday() >= 5:
        raise AirflowSkipException("Skipping weekend")
```

**3. Event-driven with Triggers:**
```python
# Use SQS/Kafka trigger to start DAGs on events
from airflow.providers.amazon.aws.triggers.sqs import SqsTrigger

class EventDrivenOperator(BaseOperator):
    def execute(self, context):
        raise TaskDeferred(
            trigger=SqsTrigger(
                queue_url=os.getenv("SQS_QUEUE_URL"),
                message_attribute_names=["event_type"],
            ),
        )
```

**Principal-level patterns:**
- Use deferrable operators for ALL long-wait scenarios (sensors, API polling)
- The triggerer process is lightweight — run it with high concurrency
- Replace `mode="reschedule"` sensors with deferrable equivalents
- Use event-driven triggers (SQS, Kafka, webhooks) instead of polling
- Implement **adaptive scheduling**: adjust frequency based on data volume
- Use **DAG bundles** (Airflow 2.9+) for version-controlled DAG deployments
- Consider **external orchestrators** (Temporal, Cadence) for mission-critical workflows where Airflow is overkill