# Airflow — Practical Interview Questions (30 Q&As)

Principal Data Engineer level. Hands-on scenarios, code reviews, architecture decisions and debugging.

---

## Q1. Debug a DAG That Fails on the First Task Every Run

**Scenario:** A daily ETL DAG has been failing on the first task (extract) every morning since last week. The error log shows:

```
[2024-01-15 06:00:12,345] {taskinstance.py:214} ERROR - Failed to execute task
apache_airflow_exceptions_AirflowException: Task failed with exception
Traceback (most recent call last):
  File "/opt/airflow/dags/etl_pipeline.py", line 45, in extract_fn
    client = boto3.client('s3')
  File "/usr/lib/python3.10/site-packages/boto3/__init__.py", line 94, in client
    raise NoCredentialError(...)
botocore.exceptions.NoCredentialError: Unable to locate credentials
```

**What do you check? What's your debugging process?**

**Answer:**

**Step 1: Check the connection and credentials:**
```bash
# Is the AWS connection configured?
airflow connections list | grep aws

# Check if the connection has credentials set
airflow connections get aws_default
```

**Step 2: Check the Airflow connection used in the code:**
```python
# The code uses boto3 directly without Airflow's connection management
# Should use Airflow's AWSHook:
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

def extract_fn(**context):
    hook = S3Hook(aws_conn_id='aws_prod')  # Uses stored credentials
    client = hook.get_conn()
    # ... rest of logic
```

**Step 3: Check environment and IAM (if on AWS/ECS/EKS):**
```bash
# If running on EC2/ECS, check IAM role attachment
# If running on EKS, check IRSA (IAM Roles for Service Accounts)
kubectl get pod -n airflow <executor-pod> -o json | \
  jq '.spec.serviceAccountName'
```

**Root causes & fixes:**

| Root Cause | Fix |
|---|---|
| Missing Airflow connection | Create connection in UI/CLI with credentials |
| Using `boto3` directly instead of `S3Hook` | Refactor to use Airflow's hook |
| IAM role not attached (AWS) | Attach IAM role to executor instances |
| IRSA misconfigured (EKS) | Fix `serviceAccountAnnotations` in K8s config |
| Wrong `aws_conn_id` value | Correct the conn_id string in the operator |
| Connection credentials expired | Rotate credentials (STS, temporary creds) |

**Prevention:**
```python
# Use Airflow's connection system — credentials managed in UI, not in code
from airflow.providers.amazon.aws.hooks.base_aws import BaseAwsHook

def extract_fn(**context):
    hook = BaseAwsHook(conn_id='aws_prod')
    # Hook handles credential resolution (env vars, IAM, keys, etc.)
    session = hook.get_session()
    client = session.client('s3')
```

---

## Q2. Write a DAG That Processes a List of Tables Dynamically

**Scenario:** You have 50+ tables in a source database that need daily CDC-style extraction → S3 → Glue catalog. Each table has different schemas. Write a DAG that:
1. Discovers tables from a config table
2. Runs extraction for each table
3. Handles failures per-table (one failure shouldn't block all)
4. Sends a summary alert at the end

```python
import os
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.operators.email import EmailOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.exceptions import AirflowException

# Configuration
SOURCE_CONN_ID = "postgres_source"
AWS_CONN_ID = "aws_prod"
S3_BUCKET = "company-warehouse-raw"
SCHEMA = "source_db"
CONFIG_TABLE = "replication_config"
SLACK_WEBHOOK = os.getenv("SLACK_WEBHOOK_URL")
EMAIL_RECIPIENTS = ["data-platform@company.com"]

default_args = {
    "owner": "data_platform",
    "retries": 2,
    "retry_delay": timedelta(minutes=30),
    "retry_exponential_backoff": True,
}

dag = DAG(
    "multi_table_extract",
    default_args=default_args,
    description="Extract all configured tables from source DB to S3",
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["etl", "cdc", "multi-table"],
)

def get_tables_to_process(**context):
    """Query config table for tables that need processing."""
    hook = PostgresHook(postgres_conn_id=SOURCE_CONN_ID)
    conn = hook.get_conn()
    cursor = conn.cursor()
    
    # Get tables marked for replication
    cursor.execute(f"""
        SELECT table_name, source_schema, partition_column, watermark_column
        FROM {CONFIG_TABLE}
        WHERE active = true
        AND last_replicated_date < CURRENT_DATE
    """)
    
    tables = [
        {
            "table_name": row[0],
            "schema": row[1],
            "partition_column": row[2],
            "watermark_column": row[3],
        }
        for row in cursor.fetchall()
    ]
    
    cursor.close()
    conn.close()
    
    # Store list for downstream tasks
    context["ti"].xcom_push(key="tables_to_process", value=tables)
    return tables

def extract_table(**context):
    """Extract a single table from source DB to S3."""
    tables = context["ti"].xcom_pull(task_ids="list_tables")
    table = tables[context["ti"].try_number - 1]  # Simplified; use dynamic task mapping instead
    
    table_name = table["table_name"]
    schema = table["schema"]
    watermark_col = table["watermark_column"]
    
    # Get last watermark value
    last_watermark = context["ti"].xcom_pull(
        task_ids="extract_all", 
        key=f"watermark_{table_name}"
    ) or "1970-01-01"
    
    # Extract new data since last watermark
    hook = PostgresHook(postgres_conn_id=SOURCE_CONN_ID)
    conn = hook.get_conn()
    cursor = conn.cursor()
    
    cursor.execute(f"""
        SELECT * FROM {schema}.{table_name}
        WHERE {watermark_col} > %s
        ORDER BY {watermark_col}
    """, (last_watermark,))
    
    rows = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    
    cursor.close()
    conn.close()
    
    if not rows:
        context["ti"].xcom_push(key="status", value="no_new_data")
        return
    
    # Write to S3 as Parquet
    import pandas as pd
    from pyarrow import parquet
    
    df = pd.DataFrame(rows, columns=columns)
    s3_hook = S3Hook(aws_conn_id=AWS_CONN_ID)
    
    s3_key = f"{SCHEMA}/{table_name}/date={{{{ ds_nodash }}}}/{table_name}_{{{{ ds_nodash }}}}.parquet"
    buffer = BytesIO()
    parquet.write_table(pa.Table.from_pandas(df), buffer)
    buffer.seek(0)
    s3_hook.load_data(buffer, s3_key, bucket_name=S3_BUCKET, replace=True)
    
    # Update watermark
    new_watermark = rows[-1][columns.index(watermark_col)]
    context["ti"].xcom_push(key=f"watermark_{table_name}", value=new_watermark)
    context["ti"].xcom_push(key="status", value="success")

def summarize_results(**context):
    """Send summary of all table extractions."""
    ti = context["ti"]
    table_tasks = ti.xcom_pull(task_ids="extract_all", key="tables_to_process")
    
    results = {"success": [], "failed": [], "skipped": []}
    
    for table in table_tasks:
        task_id = f"extract_{table['table_name']}"
        try:
            status = ti.xcom_pull(task_ids=task_id, key="status")
            if status == "success":
                results["success"].append(table["table_name"])
            elif status == "no_new_data":
                results["skipped"].append(table["table_name"])
            else:
                results["failed"].append(table["table_name"])
        except:
            results["failed"].append(table["table_name"])
    
    # Send Slack notification
    summary = (
        f"ETL Summary:\n"
        f"✅ Success: {len(results['success'])}\n"
        f"❌ Failed: {len(results['failed'])}\n"
        f"⏭️  Skipped: {len(results['skipped'])}"
    )
    
    if results["failed"]:
        summary += f"\n\nFailed tables: {', '.join(results['failed'])}"
    
    send_slack_alert(summary, results["failed"])
    
    if results["failed"]:
        raise AirflowException(f"{len(results['failed'])} tables failed extraction")

# Tasks
list_tables = PythonOperator(
    task_id="list_tables",
    python_callable=get_tables_to_process,
    dag=dag,
)

# ... extract tasks would be created dynamically (see Q3 for better approach)

summarize = PythonOperator(
    task_id="summarize_results",
    python_callable=summarize_results,
    trigger_rule="all_done",  # Run even if some tasks failed
    dag=dag,
)

list_tables >> summarize
```

**Key points for the interview:**
- `trigger_rule="all_done"` ensures the summary task runs even if individual table tasks fail
- Each table extraction is independent — one failure doesn't block others
- XCom is used for passing data between tasks (watermark values, table list)
- Watermark-based CDC approach for incremental extraction
- Summary task aggregates results and alerts on failures

**What would the interviewer drill into?**

**"How do you handle 50+ tables efficiently?"**
- With static task creation, 50 tables = 50 task instances all running in parallel
- Resource pools can limit concurrency per category
- Better approach: use **Dynamic Task Mapping** (Q3) to avoid creating tasks at parse time

**"What happens if one table extraction takes 3 hours?"**
- It holds a worker slot for 3 hours
- Set `execution_timeout` on the task
- Consider breaking large tables into partitions (by date, by ID range)
- Use deferrable operators if waiting for external conditions

**"How do you ensure idempotency?"**
- The S3 key includes the date — re-running the same day overwrites the same file
- Watermark is updated only after successful write
- Consider writing to a temp key first, then moving to final key (atomic operation)

---

## Q3. Refactor the Above DAG to Use Dynamic Task mapping

**Scenario:** The DAG from Q2 creates 50 tasks at parse time, which slows down the scheduler. Refactor it using Airflow's Dynamic Task Mapping (2.3+).

```python
import os
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.providers.postgres.hooks.postgres import PostgresHook

default_args = {
    "owner": "data_platform",
    "retries": 2,
    "retry_delay": timedelta(minutes=30),
}

dag = DAG(
    "multi_table_extract_mapped",
    default_args=default_args,
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["etl", "dynamic-mapping"],
)

def get_tables_to_process(**context):
    """Return list of table configs for dynamic task mapping."""
    hook = PostgresHook(postgres_conn_id="postgres_source")
    conn = hook.get_conn()
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT table_name, source_schema, partition_column, watermark_column
        FROM replication_config
        WHERE active = true
    """)
    
    tables = [
        {
            "table_name": row[0],
            "schema": row[1],
            "partition_column": row[2],
            "watermark_column": row[3],
        }
        for row in cursor.fetchall()
    ]
    
    cursor.close()
    conn.close()
    return tables  # Return list → triggers dynamic task mapping

def extract_table(table_config):
    """Extract a single table. Called once per mapped task."""
    table_name = table_config["table_name"]
    schema = table_config["schema"]
    watermark_col = table_config["watermark_column"]
    
    hook = PostgresHook(postgres_conn_id="postgres_source")
    s3_hook = S3Hook(aws_conn_id="aws_prod")
    
    # Extract and write to S3
    # (Same logic as Q2, but parameterized by table_config)
    conn = hook.get_conn()
    cursor = conn.cursor()
    cursor.execute(f"SELECT * FROM {schema}.{table_name} WHERE {watermark_col} > ?", ("1970-01-01",))
    rows = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    cursor.close()
    conn.close()
    
    if rows:
        # Write to S3, update watermark, etc.
        pass
    
    return {"table": table_name, "rows": len(rows)}

# Task 1: Get table list
list_tables = PythonOperator(
    task_id="list_tables",
    python_callable=get_tables_to_process,
    dag=dag,
)

# Task 2: Dynamically mapped — one task instance per table
extract = PythonOperator(
    task_id="extract",
    python_callable=extract_table,
    expand={"table_config": list_tables.output},  # ← Dynamic task mapping
    dag=dag,
)

# Task 3: Summary runs after ALL mapped tasks complete
summarize = PythonOperator(
    task_id="summarize",
    python_callable=summarize_results,
    trigger_rule="all_done",
    dag=dag,
)

list_tables >> extract >> summarize
```

**What's different from Q2:**
| Q2 (Static) | Q3 (Dynamic Mapping) |
|---|---|
| 50 tasks created at parse time | 0 tasks at parse; created at runtime |
| Scheduler sees all 50 on every parse | Scheduler parses once, creates tasks when list_tables runs |
| DAG file must know tables ahead of time | Tables discovered at runtime |
| Hard to add tables without code change | Add to config table → next run picks it up |
| `{{ task.map_index }}` not available | Each mapped task has a map index |

**Principal considerations:**
- Dynamic task mapping is a **runtime** feature, not a parse-time feature
- The scheduler doesn't know how many tasks will be created until the upstream task runs
- You **cannot** set task-specific parameters on mapped tasks (they all share the same operator config)
- If you need per-table retry policies or timeouts, use a Python callable that checks the table config internally
- `expand=` spreads the list one item per task; `expand={"a": list_a, "b": list_b}` zips two lists together

---
## Q4. Design a DAG With Data Quality Checks That Can Gate Downstream Tasks

**Scenario:** Before loading data into the data warehouse, run data quality checks. If critical checks fail, block downstream tasks. If warning-level checks fail, allow tasks to proceed but alert.

```python
from airflow import DAG
from datetime import datetime, timedelta
from airflow.operators.python import PythonOperator
from airflow.operators.empty import EmptyOperator
from airflow.exceptions import AirflowException

default_args = {"owner": "data_platform", "retries": 1}

dag = DAG(
    "dq_gated_pipeline",
    default_args=default_args,
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
)

def run_data_quality_checks(**context):
    """Run DQ checks and categorize severity."""
    from airflow.providers.postgres.hooks.postgres import PostgresHook
    
    hook = PostgresHook(postgres_conn_id="warehouse")
    results = {"critical": [], "warning": [], "pass": []}
    
    # Example: row count check
    row_count = hook.get_records("SELECT COUNT(*) FROM staging.orders")[0][0]
    expected_min = 1000
    
    if row_count == 0:
        results["critical"].append(f"orders table is empty (got {row_count})")
    elif row_count < expected_min:
        results["warning"].append(f"orders count {row_count} below threshold {expected_min}")
    else:
        results["pass"].append(f"orders count: {row_count}")
    
    # Example: null check on primary key
    nulls = hook.get_records("SELECT COUNT(*) FROM staging.orders WHERE order_id IS NULL")[0][0]
    if nulls > 0:
        results["critical"].append(f"{nulls} null order_ids in orders table")
    
    # Example: freshness check
    max_date = hook.get_records("SELECT MAX(order_date) FROM staging.orders")[0][0]
    if max_date and (datetime.now().date() - max_date.date()).days > 1:
        results["warning"].append(f"orders data is stale (max date: {max_date})")
    
    # Store results for downstream
    context["ti"].xcom_push(key="dq_results", value=results)
    
    # Raise exception if critical checks failed
    if results["critical"]:
        raise AirflowException(
            f"DQ CRITICAL failures: {', '.join(results['critical'])}"
        )
    
    # Log warnings (don't fail)
    if results["warning"]:
        context["ti"].log.warning(f"DQ warnings: {', '.join(results['warning'])}")
    
    return results

def load_to_warehouse(**context):
    """Only runs if DQ checks pass (no critical failures)."""
    pass  # Loading logic here

def notify_dq_warnings(**context):
    """Send alert for warning-level DQ issues."""
    results = context["ti"].xcom_pull(task_ids="data_quality_checks", key="dq_results")
    if results and results.get("warning"):
        send_slack_alert(f"DQ Warnings: {', '.join(results['warning'])}")

dq_check = PythonOperator(
    task_id="data_quality_checks",
    python_callable=run_data_quality_checks,
    dag=dag,
)

load = PythonOperator(
    task_id="load_to_warehouse",
    python_callable=load_to_warehouse,
    dag=dag,
)

notify = PythonOperator(
    task_id="notify_warnings",
    python_callable=notify_dq_warnings,
    trigger_rule="all_done",  # Run even if dq_check failed
    dag=dag,
)

dq_check >> load
dq_check >> notify
```

**Interview drill:**

**"What if you want downstream to run but skip bad partitions?"**
- Use Apache Griffin or Great Expectations for partition-level checks
- Mark bad partitions instead of failing the whole DAG
- Use a "quarantine" table for bad rows

**"How do you integrate Great Expectations / dbt tests?"**
```python
from airflow.providers.dbt.cli.operators.dbt_cli import DbtTestOperator

dbt_tests = DbtTestOperator(
    task_id="dbt_data_tests",
    conn_id="postgres_warehouse",
    dbt_cmd_type="test",
    project_dir="/opt/airflow/dags/dbt/models/staging/",
    dag=dag,
)

# dbt tests fail the task on critical severity by default
# Configure severity in dbt/yml for warnings vs errors
```

---

## Q5. Build a DAG That Handles S3 File Arrival With Backoff and Timeout

**Scenario:** Wait for an S3 file to arrive, but don't hold a worker slot indefinitely. Use deferrable sensors with proper timeouts and alerting.

```python
import os
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

default_args = {
    "owner": "data_platform",
    "retries": 1,
}

dag = DAG(
    "s3_file_arrival",
    default_args=default_args,
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
)

# Deferrable sensor — releases worker slot while waiting
wait_for_file = S3KeySensor(
    task_id="wait_for_sales_file",
    bucket_name="incoming-data",
    bucket_key="sales/2024-01-15/sales.csv",
    aws_conn_id="aws_prod",
    deferrable=True,  # ← Key flag: uses triggerer, not worker slot
    timeout=7200,  # 2 hours max wait (in seconds)
    poke_interval=60,  # Check every 60 seconds
    mode="deferrable",  # Required with deferrable=True
    on_failure_callback=lambda context: send_slack_alert(
        f"File not found in bucket: {context['task_instance'].task_id}"
    ),
    dag=dag,
)

# ... downstream tasks

```

**What to explain:**
- `deferrable=True` moves the waiting to the **triggerer** process, freeing the worker slot
- Without `deferrable=True`, a sensor in `reschedule` mode still holds a worker slot for the `poke_interval` duration
- `timeout` is the absolute maximum wait time — after that, the task fails
- `poke_interval` is how often the trigger checks S3
- The triggerer process is lightweight and can handle hundreds of triggers concurrently

**Principal point:** At scale with 100+ daily files, non-deferrable sensors consume 100+ worker slots for hours. Deferrable sensors use ~10 triggerer workers.

---

## Q6. Implement Retry Logic With Exponential Backoff and Alerting After Final Failure

**Scenario:** A task calls an external API that occasionally times out. Implement smart retry logic: exponential backoff, max retries, and alert only after ALL retries are exhausted.

```python
import time
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.hooks.base import BaseHook
from airflow.exceptions import AirflowException, AirflowSkipException

default_args = {
    "owner": "data_platform",
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "retry_exponential_backoff": True,  # ← Exponential backoff
    "max_retry_delay": timedelta(hours=2),  # Cap backoff at 2 hours
    "execution_timeout": timedelta(hours=1),  # Kill if stuck
}

dag = DAG(
    "api_extraction_with_retry",
    default_args=default_args,
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
)

def extract_data_with_retry(**context):
    """Custom retry logic within the task (beyond Airflow's retry)."""
    import requests
    
    conn = BaseHook.get_connection("api_conn")
    url = conn.extra_fields.get("base_url") + "/v1/data"
    headers = {"Authorization": f"Bearer {conn.password}"}
    
    max_internal_retries = 3
    for attempt in range(1, max_internal_retries + 1):
        try:
            response = requests.get(url, headers=headers, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            context["ti"].xcom_push(key="rows", value=len(data))
            context["ti"].log.info(f"Success on attempt {attempt}: {len(data)} rows")
            return data
            
        except (requests.ConnectionError, requests.Timeout) as e:
            wait_time = 2 ** attempt  # Exponential backoff: 2s, 4s, 8s
            context["ti"].log.warning(
                f"Attempt {attempt}/{max_internal_retries} failed: {e}. "
                f"Retrying in {wait_time}s..."
            )
            if attempt < max_internal_retries:
                time.sleep(wait_time)
            else:
                # All internal retries exhausted
                raise AirflowException(
                    f"All {max_internal_retries} internal retries failed. "
                    f"Last error: {e}"
                )
        except requests.HTTPError as e:
            # Don't retry on 4xx errors (bad request)
            if response.status_code >= 500:
                raise AirflowException(f"Server error after retries: {e}")
            else:
                raise AirflowException(f"Client error (not retriable): {e}")

extract = PythonOperator(
    task_id="extract_api",
    python_callable=extract_data_with_retry,
    on_failure_callback=lambda context: send_slack_alert(
        f"Task {context['task_instance'].task_id} FAILED after all retries. "
        f"DAG: {context['dag'].dag_id}, Run: {context['run_id']}"
    ),
    on_retry_callback=lambda context: context['ti'].log.info(
        f"Airflow retrying (attempt {context['ti'].try_number})"
    ),
    dag=dag,
)
```

**Key layers of retry:**
1. **Internal retry** (in the callable): fast, for transient network issues
2. **Airflow retry** (`retries=3`): restarts the whole task with exponential backoff
3. **Alerting only on final failure**: `on_failure_callback` fires only after ALL retries are exhausted

**Interview drill:**
- **Why two layers?** Internal retry is fast (seconds between attempts). Airflow retry is slower but handles cases where the worker itself had issues.
- **When NOT to retry?** 4xx HTTP errors, bad data, schema mismatches — these won't be fixed by retrying.
- **Idempotency concern:** Each retry re-runs the task. Ensure the task is idempotent (same execution_date produces the same result).

---

## Q7. Troubleshoot a DAG Whose Tasks Are Stuck in "Queued" State

**Scenario:** Tasks show as "Queued" for hours but never transition to "Running". What's your diagnosis and fix?

```bash
# Step 1: Check executor status
airflow celery status  # If using Celery
# or
kubectl get pods -n airflow -l component=worker  # If using K8s

# Step 2: Check if workers are alive
airflow jobs check --local

# Step 3: Check the executor log
tail -f /opt/airflow/logs/scheduler/*/scheduler.log | grep -i "error\|exception"

# Step 4: Check database locks
psql -U airflow -d airflow -c "SELECT * FROM alembic_version;"
```

**Common causes:**

| Symptom | Root Cause | Fix |
|---|---|---|
| No workers running | Worker pods crashed | Check K8s events, fix image/config |
| Workers running but not picking up tasks | Celery broker (Redis/RabbitMQ) down | Restart broker, check connectivity |
| Tasks queued in DB but not in Celery | Scheduler → broker connection broken | Check `broker_url` in config |
| High DB load / locks | Too many DAGs, slow queries | Index metadata DB, reduce DAG count |
| `max_active_runs` exceeded | Too many concurrent DAG runs | Increase `max_active_runs` |
| Worker concurrency maxed out | `worker_concurrency` too low | Increase worker concurrency |

**Quick fix for stuck tasks:**
```bash
# Clear stuck task instances (careful — only for stuck tasks)
airflow tasks clear my_dag -s 2024-01-15 -e 2024-01-15 --include_upstream --include_downstream

# Or clear specific stuck state
airflow tasks clear my_dag my_task -s 2024-01-15 --reset_state
```

**Prevention at Principal level:**
- Set up health checks for all components (scheduler, workers, broker, DB)
- Monitor `dag_runs` table for stuck runs
- Use `executor.queued_task_instance()` metric
- Auto-scale workers based on queue depth
- Set `worker_concurrency` based on worker resources (CPU/memory per task)

---

## Q8. Build a DAG That Reads Config from S3 and Adjusts Behavior Per Environment

**Scenario:** Same DAG runs in dev, staging and prod. Each environment has different table names, thresholds and alert recipients. Read config from S3.

```python
import json
import os
from datetime import datetime
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

# Determine environment
AIRFLOW_ENV = os.getenv("AIRFLOW_ENV", "dev")  # dev, staging, prod

S3_CONFIG_BUCKET = "airflow-configs"
S3_CONFIG_KEY = f"dags/etl_pipeline/config_{AIRFLOW_ENV}.json"

def load_config(**context):
    """Load DAG configuration from S3."""
    hook = S3Hook(aws_conn_id="aws_prod")
    
    def _read_key(bucket, key):
        resp = hook.read_key(key, bucket_name=bucket)
        return json.loads(resp)
    
    config = _read_key(S3_CONFIG_BUCKET, S3_CONFIG_KEY)
    context["ti"].xcom_push(key="config", value=config)
    return config

# Example S3 config file (config_prod.json):
# {
#   "source_table": "prod.sales",
#   "target_table": "warehouse.fact_sales",
#   "row_threshold": 10000,
#   "alert_channel": "#data-alerts-prod",
#   "slack_webhook": "https://hooks.slack.com/prod/...",
#   "s3_bucket": "prod-data-lake",
#   "enable_dq_checks": true
# }

def process_with_config(**context):
    config = context["ti"].xcom_pull(task_id="load_config", key="config")
    
    source = config["source_table"]
    target = config["target_table"]
    threshold = config["row_threshold"]
    
    context["ti"].log.info(f"Processing {source} → {target}, threshold: {threshold}")
    
    # Process data using config values
    # ...

dag = DAG(
    "configurable_etl",
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
)

load_cfg = PythonOperator(
    task_id="load_config",
    python_callable=load_config,
    dag=dag,
)

process = PythonOperator(
    task_id="process_data",
    python_callable=process_with_config,
    dag=dag,
)

load_cfg >> process
```

**Principal considerations:**
- Config in S3 is version-controlled and auditable
- Use environment variable `AIRFLOW_ENV` set at the worker level (different per env)
- For production, consider using **AWS Secrets Manager** or **HashiCorp Vault** for sensitive values (passwords, webhooks)
- Cache the config in XCom to avoid multiple S3 reads per task
- Consider using Airflow's built-in `Variable` for simpler configs (stored in DB)

**Alternative: DAG factory pattern:**
```python
# configs/etl_config.py
ENV_CONFIGS = {
    "dev": {"source": "dev.sales", "threshold": 100, "alert": "#data-dev"},
    "prod": {"source": "prod.sales", "threshold": 10000, "alert": "#data-prod"},
}

def create_etl_dag(config_name):
    cfg = ENV_CONFIGS[config_name]
    return DAG(
        f"etl_{config_name}",
        schedule="@daily",
        # ... use cfg values in tasks
    )

# Generate DAGs at parse time
for env in ENV_CONFIGS:
    globals()[f"dag_{env}"] = create_etl_dag(env)
```

---

## Q9. Handle a Long-Running Spark Job From Airflow

**Scenario:** Submit a Spark job via Airflow. Handle job status tracking, timeouts, log collection and proper cleanup.

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from airflow.providers.apache.spark.hooks.spark_submit import SparkSubmitHook

default_args = {
    "owner": "data_platform",
    "retries": 1,
    "retry_delay": timedelta(minutes=10),
    "execution_timeout": timedelta(hours=6),  # Kill if > 6 hours
}

dag = DAG(
    "spark_processing",
    default_args=default_args,
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
)

spark_submit = SparkSubmitOperator(
    task_id="run_spark_job",
    application="/opt/airflow/dags/spark/jobs/transform.py",
    conn_id="spark_default",
    spark_submit_args=[
        "--deploy-mode", "cluster",
        "--master", "yarn",
        "--num-executors", "20",
        "--executor-memory", "8g",
        "--executor-cores", "4",
        "--driver-memory", "4g",
        "--conf", "spark.sql.adaptive.enabled=true",
        "--conf", "spark.dynamicAllocation.enabled=true",
        "--conf", f"spark.sql.sources.bucketing.enabled=true",
    ],
    name="daily_transform_{{ ds_nodash }}",
    verbose=True,
    dag=dag,
)
```

**For Databricks:**
```python
from airflow.providers.databricks.operators.databricks import DatabricksSubmitRunOperator

databricks_run = DatabricksSubmitRunOperator(
    task_id="databricks_transform",
    conn_id="databricks_prod",
    run_name=f"transform_{{{{ ds_nodash }}}}",
    tasks=[
        {
            "task_key": "transform",
            "spark_python_task": {
                "python_file": "dbfs:/dags/spark/transform.py",
                "parameters": [
                    f"--date={{{{{ ds_nodash }}}}}",
                    f"--source_bucket=s3://prod-data/",
                ],
            },
            "new_cluster": {
                "num_workers": 20,
                "node_type_id": "i3.8xlarge",
                "spark_version": "12.2.x-scala2.12",
            },
        }
    ],
    timeout_seconds=21600,  # 6 hours
    dag=dag,
)
```

**Principal considerations:**
- **Cluster management:** Use all-purpose clusters for dev, job clusters for prod (cost)
- **Dynamic allocation:** Enable `spark.dynamicAllocation.enabled` for cost efficiency
- **AQE (Adaptive Query Execution):** Enable for complex joins and skew
- **Monitoring:** Collect Spark UI metrics, link them in Airflow logs
- **Idempotency:** Spark jobs should write to temp paths, then atomically move to final path
- **Backpressure:** If Spark is slow, don't queue up too many Airflow DAG runs (`max_active_runs=1`)

---

## Q10. Write a DAG That Orchestrates a Multi-Step ML Pipeline (Train → Evaluate → Deploy)

**Scenario:** An ML pipeline that trains a model, evaluates it against a champion, and deploys only if metrics improve. Each step has different resource needs.

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.exceptions import AirflowSkipException

default_args = {
    "owner": "ml_platform",
    "retries": 1,
}

dag = DAG(
    "ml_training_pipeline",
    default_args=default_args,
    schedule="0 2 * * 1",  # Mondays at 2 AM
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["ml", "model-training"],
)

def train_model(**context):
    """Train model on latest data. Push metrics to XCom."""
    import subprocess
    
    result = subprocess.run(
        ["python", "/opt/airflow/dags/ml/train.py", "--date", "{{ ds_nodash }}"],
        capture_output=True, text=True,
    )
    
    if result.returncode != 0:
        raise AirflowException(f"Training failed: {result.stderr}")
    
    # Parse metrics from training output
    metrics = parse_metrics(result.stdout)
    context["ti"].xcom_push(key="metrics", value=metrics)
    context["ti"].xcom_push(key="model_path", value="/opt/models/challenger")
    return metrics

def evaluate_model(**context):
    """Compare challenger vs champion metrics."""
    challenger_metrics = context["ti"].xcom_pull(task_id="train_model", key="metrics")
    
    # Get champion metrics (from previous best)
    champion_metrics = get_champion_metrics()
    
    # Compare key metric (e.g., AUC)
    improvement = challenger_metrics["auc"] - champion_metrics["auc"]
    min_improvement = 0.01  # 1% improvement required
    
    context["ti"].log.info(
        f"Challenger AUC: {challenger_metrics['auc']:.4f} vs "
        f"Champion AUC: {champion_metrics['auc']:.4f} "
        f"(improvement: {improvement:.4f})"
    )
    
    if improvement >= min_improvement:
        context["ti"].xcom_push(key="deploy", value=True)
    else:
        context["ti"].log.info("No significant improvement. Skipping deployment.")
        context["ti"].xcom_push(key="deploy", value=False)

def deploy_model(**context):
    """Deploy challenger as new champion."""
    should_deploy = context["ti"].xcom_pull(task_id="evaluate_model", key="deploy")
    
    if not should_deploy:
        raise AirflowSkipException("Model does not meet deployment threshold")
    
    model_path = context["ti"].xcom_pull(task_id="train_model", key="model_path")
    # Deploy logic (e.g., update SageMaker endpoint, copy to production bucket)
    deploy_to_production(model_path)

train = PythonOperator(task_id="train_model", python_callable=train_model, dag=dag)
evaluate = PythonOperator(task_id="evaluate_model", python_callable=evaluate_model, dag=dag)
deploy = PythonOperator(task_id="deploy_model", python_callable=deploy_model, dag=dag)

train >> evaluate >> deploy
```

**Key patterns:**
- `AirflowSkipException` in deploy prevents false "failure" alerts when model doesn't improve
- XCom passes metrics between tasks
- Each step can run on different resources (train on GPU, evaluate on CPU)
- The DAG is idempotent: re-running on the same date retrains with the same data

---
## Q11. Build a Reusable Airflow Plugin for Custom Operators

**Scenario:** Your team has many custom operators (S3 → BigQuery load, CDC sync, etc.). You want a plugin so they are auto-discoverable in the UI, reusable across DAG files, and easy to version.

### Solution

```python
# plugins/custom_operators_plugin.py
from airflow.plugins_manager import AirflowPlugin
from airflow.models import BaseOperator
from airflow.hooks.base import BaseHook

class S3ToBigQueryOperator(BaseOperator):
    """Load a file from S3 to BigQuery with retry and monitoring."""

    def __init__(self, s3_conn_id, gcp_conn_id, s3_bucket, s3_key,
                 dataset_id, table_id, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.s3_conn_id = s3_conn_id
        self.gcp_conn_id = gcp_conn_id
        self.s3_bucket = s3_bucket
        self.s3_key = s3_key
        self.dataset_id = dataset_id
        self.table_id = table_id

    def execute(self, context):
        from airflow.providers.amazon.aws.hooks.s3 import S3Hook
        from airflow.providers.google.cloud.hooks.bigquery import BigQueryHook
        import tempfile, os

        self.log.info(f"Downloading s3://{self.s3_bucket}/{self.s3_key}")
        s3_hook = S3Hook(aws_conn_id=self.s3_conn_id)
        with tempfile.NamedTemporaryFile(delete=False) as tmp:
            s3_hook.download_file(self.s3_bucket, self.s3_key, tmp.name)
            local_path = tmp.name

        try:
            self.log.info(f"Loading into {self.dataset_id}.{self.table_id}")
            bq_hook = BigQueryHook(gcp_conn_id=self.gcp_conn_id)
            client = bq_hook.get_client()
            # Use BigQuery LoadJobConfig for the actual load
            # (simplified - in production use proper schema config)
            # client.load_table_from_file(...)
            self.log.info("Load complete")
        finally:
            os.unlink(local_path)

class S3ToBigQueryPlugin(AirflowPlugin):
    name = "s3_to_bigquery_plugin"
    operators = [S3ToBigQueryOperator]
```

Register the plugin:
```bash
# Place in $AIRFLOW_HOME/plugins/
# Airflow auto-discovers it on startup
```

Use in a DAG:
```python
from plugins.custom_operators_plugin import S3ToBigQueryOperator

load_task = S3ToBigQueryOperator(
    task_id="load_s3_to_bq",
    s3_conn_id="aws_prod",
    gcp_conn_id="gcp_prod",
    s3_bucket="my-etl-data",
    s3_key="raw/sales/2024-01-01.csv",
    dataset_id="analytics",
    table_id="sales_raw",
    retries=3,
    retry_delay=timedelta(minutes=5),
    dag=dag,
)
```

### What to Explain

1. **Plugin structure:** `AirflowPlugin` subclass with `name`, `operators`, and optionally `hooks`, `sensors`, `admin_views`.
2. **Auto-discovery:** Airflow scans `$AIRFLOW_HOME/plugins/` on startup.
3. **Why plugins over inline operators:** Reusability, versioning, separate packaging for CI/CD.
4. **Testing plugins:** Unit test the operator in isolation with mocked hooks.

### Principal-Level Considerations

- Package custom operators as a pip-installable library so teams can share them across Airflow deployments.
- Add integration tests that run against real S3/BQ (in a test account).
- Pin operator versions in DAG files if the operator has a versioned API.
- Consider OpenLineage integration for automatic lineage tracking from custom operators.

---

## Q12. Implement Dynamic Task Mapping for a Multi-Region ETL Pipeline

**Scenario:** You have ETL pipelines for 5 regions (US-EAST, US-WEST, EU-WEST, AP-SOUTH, AP-EAST). Each region has its own database. You want one DAG that dynamically creates tasks per region.

### Solution

```python
# dags/multi_region_etl.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.decorators import task

@dag(
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    doc_md="Process ETL for each region dynamically.",
    default_args={"owner": "data-engineering", "retries": 2},
)
def multi_region_etl():
    @task
    def get_regions():
        """This upstream task returns the list of regions to process."""
        # In production, this could come from a config table, API, etc.
        return ["us_east", "us_west", "eu_west", "ap_south", "ap_east"]

    @task(
        retries=3,
        retry_delay=timedelta(minutes=5),
        doc_md="Extract, transform, and load data for a specific region."
    )
    def process_region(region: str):
        from airflow.providers.postgres.hooks.postgres import PostgresHook

        conn_id_map = {
            "us_east": "pg_us_east",
            "us_west": "pg_us_west",
            "eu_west": "pg_eu_west",
            "ap_south": "pg_ap_south",
            "ap_east": "pg_ap_east",
        }
        conn_id = conn_id_map[region]
        hook = PostgresHook(postgres_conn_id=conn_id)
        conn = hook.get_conn()
        cursor = conn.cursor()
        cursor.execute("SELECT COUNT(*) FROM transactions WHERE extract_date = CURRENT_DATE")
        row_count = cursor.fetchone()[0]
        cursor.close()
        conn.close()

        import json
        result = {"region": region, "rows_processed": row_count}
        print(f"Processed {region}: {json.dumps(result)}")
        return result

    @task(trigger_rule="none_failed_min_one_success")
    def consolidate_results(results: list[dict]):
        total = sum(r["rows_processed"] for r in results)
        print(f"Total rows across {len(results)} regions: {total}")
        return {"total_regions": len(results), "total_rows": total}

    regions = get_regions()
    results = process_region.expand(region=regions)
    consolidate_results(results)

dag = multi_region_etl()
```

### What to Explain

1. **`.expand()`** vs `.map()`: `expand` takes a list and creates one task per element; `map` is for mapping between lists.
2. **Mapped tasks in UI:** Each mapped task shows as a separate task instance.
3. **Failure isolation:** If one region fails, others can succeed. Use `trigger_rule="none_failed_min_one_success"` for downstream consolidation.
4. **Dynamic vs static TaskGroups:** Mapping is dynamic at runtime; TaskGroups are static at DAG parse time.

### When Not to Use Dynamic Mapping

- When all regions share a single connection (no benefit to parallelism).
- When the number of regions changes frequently and you need strict SLAs per region.
- When you need custom retry logic per region.

### Principal-Level Considerations

- Combine dynamic mapping with external config (S3 config file, database table, API) so regions can be added without code changes.
- Add per-region metrics (row counts, duration) to a dashboard.
- Consider region-specific timezones for `schedule` if processing should happen at a local time.
- Use `max_active_tasks` to limit parallelism if regions have heavy workloads.

---

## Q13. Design a Backfill Strategy for a New Data Pipeline

**Scenario:** A new daily pipeline needs to backfill 2 years of historical data. The source system has daily partitions. The pipeline should be fast, idempotent, and not overwhelm the source database.

### Solution

```bash
# Option 1: Airflow CLI backfill with controlled parallelism
airflow dags backfill my_pipeline \
    --start-date 2022-01-01 \
    --end-date 2024-01-01 \
    --max-active-tasks 5 \
    --parallel 3 \
    --subdir dags/
```

```python
# dags/my_pipeline.py - Key considerations in the DAG
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
import boto3

default_args = {
    "owner": "data-engineering",
    "retries": 2,
    "retry_delay": timedelta(minutes=15),
    "retry_exponential_backoff": True,
    "max_retry_delay": timedelta(hours=2),
}

@dag(
    schedule="@daily",
    start_date=datetime(2022, 1, 1),
    catchup=False,  # Set to True only when ready to backfill
    default_args=default_args,
    max_active_tasks=5,
    doc_md="Backfill strategy: controlled parallelism with source protection.",
)
def my_pipeline():
    from airflow.decorators import task

    @task
    def extract_for_date(execution_date):
        # Use execution_date to extract only the partition for that day
        hook = PostgresHook(postgres_conn_id="source_pg")
        conn = hook.get_conn()
        cursor = conn.cursor()

        # Partitioned query: only touches one partition
        cursor.execute(
            "SELECT * FROM transactions "
            "WHERE transaction_date = %s",
            (execution_date.strftime("%Y-%m-%d"),)
        )
        rows = cursor.fetchall()
        cursor.close()
        conn.close()
        return rows

    @task
    def load_to_s3(rows, execution_date):
        s3_hook = S3Hook(aws_conn_id="aws_prod")
        date_str = execution_date.strftime("%Y/%m/%d")
        s3_hook.load_string(
            str(rows),
            f"raw/transactions/{date_str}.json",
            "my-etl-data"
        )
        return f"s3://my-etl-data/raw/transactions/{date_str}.json"

    # In production: use execution_date from context or map over dates
    from airflow.operators.python import get_current_date_string
    # For backfill, Airflow sets execution_date to the run's logical date
    extract = extract_for_date.expand(execution_date=get_current_date_string())
    load_to_s3(extract)

dag = my_pipeline()
```

### Backfill Best Practices

| Practice | Why |
|----------|-----|
| Partitioned queries | Only one partition per run; source DB isn't overwhelmed |
| `max_active_tasks` | Limits concurrent backfill runs |
| `--parallel` flag | Controls how many DAG runs execute simultaneously |
| Idempotent loads | `execution_date` in file path means re-runs are safe |
| Exponential backoff | Avoids thundering herd on retry |
| `catchup=False` initially | Prevents accidental backfill on DAG creation |

### What to Explain

1. **Backfill vs Catchup:** Backfill is manual; catchup is automatic when `start_date` is in the past and `catchup=True`.
2. **`--max-active-tasks`:** Limits total running tasks across all backfill runs.
3. **`--parallel`:** Limits concurrent DAG runs.
4. **Idempotency:** Each run writes to a date-specific location. Re-running is safe.

### Principal-Level Considerations

- **Staggered backfill:** Start with a small date range (1 month), verify data quality, then run the rest.
- **Rate limiting:** If the source DB has connection limits, add a `SimpleHttpSensor` or `PythonOperator` with a sleep between batches.
- **Monitoring:** Set up alerts specific to backfill runs using `AIRFLOW_IS_BACKFILL` environment variable.
- **Cost:** Backfilling 2 years with 5 parallel runs = ~730 * 5 task runs. Calculate compute cost upfront.
- **Data quality:** Run data quality checks on a sample of backfilled data before considering it complete.
- **Rollback plan:** If backfill corrupts data, you need a way to identify and re-process backfilled data (use `dag_run_id` or a `run_type` flag in the data).

---

## Q14. Build a Production-Grade Airflow Dashboard with Custom Metrics

**Scenario:** You need to monitor Airflow health beyond the built-in UI. Track custom metrics like ETL row counts, data freshness, pipeline duration, and error rates in a centralized dashboard.

### Solution

```python
# plugins/metrics_exporter.py
from airflow.plugins_manager import AirflowPlugin
from airflow.stats import stats
import time

# Custom metrics in operators
def track_etl_metrics(metric_name, value, tags=None):
    """Send custom metrics to Airflow's stats backend."""
    if tags:
        stats.incr(f"etl.{metric_name}", value, tags=tags)
    else:
        stats.incr(f"etl.{metric_name}", value)

def track_duration(metric_name, duration_seconds, tags=None):
    stats.timing(f"duration.{metric_name}", duration_seconds, tags=tags)
```

```python
# In your operator or task
@task
def etl_process(execution_date):
    start = time.time()
    # ... processing logic ...
    rows_processed = 150000

    from plugins.metrics_exporter import track_etl_metrics, track_duration
    track_etl_metrics("rows_processed", rows_processed,
                      tags={"dag": "etl_pipeline", "date": execution_date})
    track_duration("etl_duration", time.time() - start,
                   tags={"dag": "etl_pipeline"})
```

### Stats Backend Configuration

```python
# airflow.cfg
[statsd]
statsd_on = True
statsd_host = grafana-statsd.monitoring.svc
statsd_port = 8125
statsd_prefix = airflow
```

### Grafana Dashboard Panels

| Panel | Metric | Query |
|-------|--------|-------|
| DAG Success Rate | `airflow.scheduler.tasks.success` | `rate(airflow_scheduler_tasks_success[1h])` |
| Task Duration P95 | `airflow.duration.etl_duration` | `histogram_quantile(0.95, ...)` |
| Data Freshness | Custom metric | `max(etl_last_success{dag="etl_pipeline"})` |
| Error Rate | `airflow.scheduler.tasks.failed` | `rate(airflow_scheduler_tasks_failed[1h])` |
| Row Count | `etl.rows_processed` | `sum(etl_rows_processed{dag="etl_pipeline"})` |

### What to Explain

1. **Airflow Stats Backend:** Uses StatsD/InfluxDB/Prometheus. Configure in `airflow.cfg`.
2. **Custom Metrics:** Use `stats.incr()`, `stats.timing()`, `stats.gauge()` from any operator.
3. **Grafana Integration:** Pull metrics from the stats backend into Grafana panels.
4. **Data Freshness:** Track the latest successful execution date per DAG; alert if stale.

### Principal-Level Considerations

- **SLO tracking:** Define SLAs per pipeline (e.g., "data fresh by 6 AM") and track them in Grafana.
- **Cost tracking:** Add cost-per-run metrics based on cloud resource usage.
- **Cross-platform correlation:** Correlate Airflow metrics with infrastructure metrics (CPU, memory, DB load).
- **Automated runbook triggers:** Use metric thresholds to trigger automated remediation (e.g., restart a stuck DAG).
- **Security:** Don't expose Airflow's metrics endpoint publicly; put it behind an auth proxy.

---
## Q15. Implement a Production DAG With Complete Error Handling and Alerting

**Scenario:** Build a DAG that handles all failure modes gracefully: transient errors, data quality failures, and infrastructure issues, with appropriate alerting for each.

### Solution

```python
# dags/production_etl.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.exceptions import AirflowException
from airflow.providers.postgres.hooks.postgres import PostgresHook
import requests
import json

def send_alert(subject, body, severity="warning"):
    """Send alert via Slack/PagerDuty/email based on severity."""
    webhook_url = f"https://hooks.slack.com/services/xxx/yyy/zzz"
    payload = {
        "text": f"*[{severity.upper()}]* {subject}\n{body}",
        "username": "Airflow Alerts",
    }
    requests.post(webhook_url, json=payload)

def on_failure_callback(context):
    """Triggered on any task failure."""
    dag_id = context["dag"].dag_id
    task_id = context["task_instance"].task_id
    exec_date = context["execution_date"]
    send_alert(
        f"Task Failed: {dag_id}/{task_id}",
        f"Execution date: {exec_date}\nCheck Airflow UI for details.",
        severity="critical"
    )

def on_retry_callback(context):
    """Triggered on each retry attempt."""
    ti = context["task_instance"]
    send_alert(
        f"Task Retrying: {ti.dag_id}/{ti.task_id}",
        f"Attempt {ti.try_number + 1} of {ti.max_tries + 1}",
        severity="warning"
    )

default_args = {
    "owner": "data-engineering",
    "on_failure_callback": on_failure_callback,
    "on_retry_callback": on_retry_callback,
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "retry_exponential_backoff": True,
    "max_retry_delay": timedelta(hours=1),
    "email": ["data-platform@company.com"],
    "email_on_failure": True,
    "email_on_retry": False,
}

@dag(
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    default_args=default_args,
    sla_miss_callback=lambda context: send_alert(
        f"SLA Missed: {context['dag'].dag_id}",
        f"Tasks: {context['task_instances']}",
        severity="critical"
    ),
)
def production_etl():
    from airflow.decorators import task

    @task
    def validate_source():
        """Check source system is accessible before processing."""
        hook = PostgresHook(postgres_conn_id="source_pg")
        try:
            conn = hook.get_conn()
            cursor = conn.cursor()
            cursor.execute("SELECT 1")
            cursor.close()
            conn.close()
        except Exception as e:
            send_alert("Source DB Unreachable", str(e), "critical")
            raise AirflowException(f"Source system check failed: {e}")

    @task
    def extract_and_validate():
        """Extract data with quality checks."""
        hook = PostgresHook(postgres_conn_id="source_pg")
        conn = hook.get_conn()
        cursor = conn.cursor()
        cursor.execute("SELECT COUNT(*) FROM transactions WHERE date = CURRENT_DATE")
        count = cursor.fetchone()[0]
        cursor.close()
        conn.close()

        if count == 0:
            raise AirflowException("No data found for today - possible source issue")
        if count > 1000000:
            send_alert("Unusual Data Volume", f"Got {count} rows", "warning")
        return count

    @task(retries=5)  # Extra retries for transient network issues
    def load_data():
        """Load with comprehensive error handling."""
        # ... loading logic ...
        pass

    validate_source() >> extract_and_validate() >> load_data()

dag = production_etl()
```

### What to Explain

1. **Callback hierarchy:** `on_failure_callback` (task level), `sla_miss_callback` (DAG level), both can coexist.
2. **Retry strategy:** Exponential backoff prevents thundering herd; `max_retry_delay` caps it.
3. **Data validation as a gate:** Fail fast before expensive processing.
4. **Severity-based alerting:** Critical = PagerDuty, Warning = Slack, Info = email.

### Principal-Level Considerations

- Use a dedicated alerting service (PagerDuty, OpsGenie) with escalation policies.
- Deduplicate alerts: if 50 tasks fail due to one root cause, send one alert, not 50.
- Add runbook links to alerts so on-call engineers know how to respond.
- Track MTTD (Mean Time To Detect) and MTTR (Mean Time To Resolve) per pipeline.

---

## Q16. Build a Self-Healing Pipeline With Automatic Recovery

**Scenario:** A daily ETL pipeline occasionally fails due to transient issues (network blips, source DB lock, S3 timeout). Design it to recover automatically without manual intervention.

### Solution

```python
# dags/self_healing_etl.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.exceptions import AirflowException
from airflow.utils import state
import time

def smart_retry_logic(context):
    """Custom retry with exponential backoff and circuit breaker."""
    ti = context["task_instance"]
    max_tries = ti.max_tries
    try_number = ti.try_number

    if try_number >= 3:
        # After 3 failures, check if the issue is persistent
        raise AirflowException(
            f"Task failed {try_number} times - possible persistent issue. "
            f"Stopping retries to avoid wasting resources."
        )

    # Exponential backoff: 5min, 15min, 45min
    delay = min(5 * (3 ** (try_number - 1)), 60)
    return timedelta(minutes=delay)

def check_source_health():
    """Circuit breaker pattern: check if source is healthy before retrying."""
    import requests
    try:
        resp = requests.get("https://source-api/health", timeout=10)
        return resp.status_code == 200
    except:
        return False

default_args = {
    "owner": "data-engineering",
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "retry_exponential_backoff": True,
    "max_retry_delay": timedelta(hours=1),
    "execution_timeout": timedelta(hours=2),
    "on_failure_callback": lambda c: None,  # Custom logic in task
}

@dag(
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    default_args=default_args,
    max_active_runs=1,
    doc_md="Self-healing with circuit breaker and smart retries.",
)
def self_healing_etl():
    from airflow.decorators import task

    @task(
        retries=3,
        retry_delay=timedelta(minutes=5),
        retry_exponential_backoff=True,
    )
    def extract_with_retry():
        max_attempts = 3
        for attempt in range(1, max_attempts + 1):
            try:
                from airflow.providers.postgres.hooks.postgres import PostgresHook
                hook = PostgresHook(postgres_conn_id="source_pg")
                conn = hook.get_conn()
                cursor = conn.cursor()
                cursor.execute("SELECT * FROM transactions WHERE date = CURRENT_DATE")
                rows = cursor.fetchall()
                cursor.close()
                conn.close()
                return rows
            except Exception as e:
                if attempt == max_attempts:
                    raise
                wait = 5 * (2 ** attempt)
                print(f"Attempt {attempt} failed: {e}. Waiting {wait}s...")
                time.sleep(wait)

    @task(trigger_rule="none_failed_min_one_success")
    def partial_load_check(results):
        """If some partitions succeeded, load what we have."""
        return results

    extract_with_retry() >> partial_load_check()

dag = self_healing_etl()
```

### What to Explain

1. **Circuit breaker:** Check source health before retrying to avoid wasted attempts.
2. **Exponential backoff:** Built into Airflow's retry; configure `retry_exponential_backoff`.
3. **`trigger_rule="none_failed_min_one_success"`:** Allows partial success in mapped tasks.
4. **`execution_timeout`:** Hard stop to prevent zombie tasks.

### Principal-Level Considerations

- **Dead Letter Queue:** For failed records, write them to a separate location for manual review instead of failing the whole pipeline.
- **Auto-repair playbooks:** If a known issue is detected (e.g., source DB restart), auto-trigger a retry after the fix.
- **Cost of retries:** Each retry costs compute. Track retry rates and optimize the root cause.
- **Graceful degradation:** If non-critical data is missing, proceed with what you have and flag it.

---

## Q17. Design a Multi-Environment Airflow Deployment

**Scenario:** You need Airflow for dev, staging and prod with different configurations, isolated data, and a promotion path.

### Solution

```yaml
# docker-compose.override.yml - Environment-specific config
# .env.dev
AIRFLOW_ENV=dev
AIRFLOW__CORE__DAGS_FOLDER=/opt/airflow/dags
AIRFLOW__CORE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@dev-db:5432/airflow_dev
AIRFLOW__CELERY__BROKER_URL=redis://dev-redis:6379/0
AIRFLOW__CELERY__RESULT_BACKEND=db+postgresql://airflow:airflow@dev-db:5432/airflow_dev
AIRFLOW__CORE__EXECUTOR=CeleryExecutor

# .env.prod
AIRFLOW_ENV=prod
AIRFLOW__CORE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:secure_pass@prod-db:5432/airflow_prod
AIRFLOW__CELERY__BROKER_URL=redis://prod-redis:6379/0
AIRFLOW__CELERY__RESULT_BACKEND=db+postgresql://airflow:secure_pass@prod-db:5432/airflow_prod
AIRFLOW__CORE__EXECUTOR=KubernetesExecutor
AIRFLOW__API__AUTH_BACKENDS=airflow.api.auth.backend.basic_auth,airflow.api.auth.backend.kerberos
```

```python
# dags/config_loader.py
import os

ENV = os.getenv("AIRFLOW_ENV", "dev")

def get_config():
    configs = {
        "dev": {
            "source_conn_id": "pg_dev",
            "target_conn_id": "s3_dev",
            "alert_channel": "#dev-alerts",
            "max_active_tasks": 3,
        },
        "staging": {
            "source_conn_id": "pg_staging",
            "target_conn_id": "s3_staging",
            "alert_channel": "#staging-alerts",
            "max_active_tasks": 5,
        },
        "prod": {
            "source_conn_id": "pg_prod",
            "target_conn_id": "s3_prod",
            "alert_channel": "#prod-alerts",
            "max_active_tasks": 10,
        },
    }
    return configs[ENV]

# In DAG:
cfg = get_config()
@dag(max_active_tasks=cfg["max_active_tasks"])
def my_dag():
    pass
```

### Environment Isolation Strategy

| Aspect | Dev | Staging | Prod |
|--------|-----|---------|------|
| DB | Shared dev DB | Isolated staging DB | Prod DB (separate AZ) |
| Executor | LocalExecutor | CeleryExecutor | KubernetesExecutor |
| Auth | Basic | Basic | OAuth/Kerberos + RBAC |
| DAGs | Same codebase, env-specific config | Same | Same |
| Connections | Dev credentials | Staging credentials | Prod credentials (Vault) |
| Monitoring | Minimal | Full | Full + PagerDuty |
| CI/CD | Direct deploy | PR-triggered test | Approved deploy only |

### Promotion Path

```
1. Dev: Develop and test DAG locally or in dev Airflow
2. Staging: Merge to main → auto-deploy to staging
3. Staging validation: Run automated DAG tests, check data quality
4. Prod: Manual approval → deploy to prod
5. Prod validation: Monitor first run, verify metrics
```

### What to Explain

1. **Same DAG code, different config:** Use environment variables or external config to switch connections.
2. **Connection management:** Use Airflow Connections with environment-specific IDs.
3. **Secret management:** Dev uses plaintext; prod uses HashiCorp Vault or AWS Secrets Manager.
4. **CI/CD:** Validate DAGs in staging before prod deployment.

### Principal-Level Considerations

- **Parity:** Keep staging as close to prod as possible (same executor, same config pattern).
- **DAG testing in CI:** Parse all DAGs, run unit tests on tasks, validate connections exist.
- **Feature flags:** Enable new pipelines in staging first, then prod.
- **Rollback:** Keep previous DAG version available for quick rollback.
- **Audit trail:** Track who deployed what DAG version to which environment.

---

## Q18. Implement Airflow DAG Testing in CI/CD

**Scenario:** You need to validate DAGs before they reach production: syntax check, connection validation, and unit tests for custom logic.

### Solution

```python
# dags/test_dags.py
import unittest
from unittest.mock import patch, MagicMock
from airflow.models import DAG
from airflow.utils.dag_cycle_test import check_cycle
from airflow.models import DagBag

class TestDAGStructure(unittest.TestCase):
    """Test DAG parsing and structure."""

    def setUp(self):
        self.dagbag = DagBag(dag_folder="dags/", include_examples=False)

    def test_no_dag_cycle(self):
        """Ensure no circular dependencies."""
        for dag_id, dag in self.dagbag.dags.items():
            with self.assertRaises(Exception) if False else self.subTest(dag_id=dag_id):
                check_cycle(dag)

    def test_all_dags_have_owner(self):
        for dag_id, dag in self.dagbag.dags.items():
            self.assertIn("owner", dag.default_args)
            self.assertTrue(dag.default_args["owner"])

    def test_all_dags_have_schedule(self):
        for dag_id, dag in self.dagbag.dags.items():
            self.assertIsNotNone(dag.schedule_interval)

class TestCustomOperators(unittest.TestCase):
    """Test custom operator logic."""

    @patch("airflow.providers.postgres.hooks.postgres.PostgresHook")
    def test_extract_task(self, mock_hook):
        from dags.production_etl import extract_and_validate

        mock_conn = MagicMock()
        mock_cursor = MagicMock()
        mock_cursor.fetchone.return_value = (100,)
        mock_conn.cursor.return_value = mock_cursor
        mock_hook.return_value.get_conn.return_value = mock_conn

        # Test the function logic
        result = extract_and_validate.execute(context={})
        mock_cursor.close.assert_called()
        mock_conn.close.assert_called()

if __name__ == "__main__":
    unittest.main()
```

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
          python-version: "3.11"
      - name: Install dependencies
        run: pip install apache-airflow pytest
      - name: Parse DAGs
        run: |
          export AIRFLOW_HOME=$GITHUB_WORKSPACE
          airflow db init
          airflow dags list
      - name: Run tests
        run: pytest dags/test_dags.py -v
      - name: Lint DAGs
        run: pip install flake8 && flake8 dags/ --exclude dags/test_dags.py
```

### What to Explain

1. **DAG parsing test:** `DagBag` imports all DAGs; catches syntax errors.
2. **`check_cycle`:** Validates no circular dependencies.
3. **Unit tests for operators:** Mock hooks to test business logic.
4. **CI/CD pipeline:** Parse → test → lint → deploy.

### Principal-Level Considerations

- **Integration tests:** Run against a staging Airflow instance with real (but safe) connections.
- **Performance tests:** Check DAG parse time; slow DAGs block the scheduler.
- **Connection validation:** Verify all connections referenced in DAGs exist in the target environment.
- **DAG serialization:** Enable it in 2.7+ for faster parsing.
- **Test coverage:** Require minimum coverage for custom operators.

---
## Q19. Build a DAG That Triggers External Systems and Handles Their Responses

**Scenario:** After ETL completes, trigger a downstream ML training job, notify a dashboard service, and update a data catalog. Handle failures in each downstream system independently.

### Solution

```python
# dags/etl_with_downstream.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.http.hooks.http import HttpHook
from airflow.utils.trigger_rule import TriggerRule
import requests

def trigger_ml_training(context):
    """Trigger external ML training pipeline."""
    hook = HttpHook(method="POST", connector_id="ml_service")
    payload = {"dataset": "etl_output", "version": context["execution_date"].strftime("%Y%m%d")}
    try:
        response = hook.run(endpoint="train", data=payload)
        print(f"ML training triggered: {response}")
        return response
    except Exception as e:
        print(f"ML training trigger failed (non-blocking): {e}")
        return None

def notify_dashboard(context):
    """Notify dashboard that new data is available."""
    hook = HttpHook(method="POST", connector_id="dashboard_service")
    try:
        response = hook.run(endpoint="refresh")
        print(f"Dashboard notified: {response}")
    except Exception as e:
        print(f"Dashboard notification failed (non-blocking): {e}")

def update_data_catalog(context):
    """Update data catalog metadata."""
    from airflow.providers.openlineage.utils.utils import get_lineage_backends
    # Or use a custom catalog API
    hook = HttpHook(method="PUT", connector_id="data_catalog")
    try:
        response = hook.run(endpoint="datasets/etl_output/metadata")
        print(f"Catalog updated: {response}")
    except Exception as e:
        print(f"Catalog update failed (non-blocking): {e}")

default_args = {
    "owner": "data-engineering",
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
}

@dag(
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    default_args=default_args,
)
def etl_with_downstream():
    from airflow.decorators import task

    @task
    def run_etl():
        # ... main ETL logic ...
        pass

    @task(trigger_rule=TriggerRule.DOWNSTREAM_SUCCESS)
    def trigger_ml():
        trigger_ml_training(context=None)

    @task(trigger_rule=TriggerRule.DOWNSTREAM_SUCCESS)
    def notify():
        notify_dashboard(context=None)

    @task(trigger_rule=TriggerRule.DOWNSTREAM_SUCCESS)
    def catalog_update():
        update_data_catalog(context=None)

    etl = run_etl()
    # All downstream tasks run in parallel after ETL succeeds
    etl >> trigger_ml()
    etl >> notify()
    etl >> catalog_update()

dag = etl_with_downstream()
```

### What to Explain

1. **`trigger_rule`:** Controls when downstream tasks run relative to upstream.
2. **Independent failure handling:** Each downstream call catches its own exceptions.
3. **`HttpHook`:** Uses Airflow Connections for auth/config management.
4. **Parallel downstream tasks:** All three run simultaneously after ETL.

### Principal-Level Considerations

- Use a message queue (Kafka, SQS) instead of direct HTTP calls for reliability.
- Implement idempotency keys so downstream systems don't process duplicate triggers.
- Add dead letter queues for failed downstream messages.
- Consider using Airflow's `ExternalTaskMarker` if downstream work is in another DAG.

---

## Q20. Design an Airflow Deployment on Kubernetes With Production Hardening

**Scenario:** Deploy Airflow on Kubernetes for production with auto-scaling, high availability, and proper resource management.

### Solution

```yaml
# kubernetes/airflow-deployment.yaml
# Using Helm chart: https://github.com/apache/airflow/tree/main/chart

# values-prod.yaml
executor: KubernetesExecutor

webserver:
  replicas: 2
  ingress:
    enabled: true
    hosts:
      - airflow.company.com
    tls:
      - secretName: airflow-tls
        hosts:
          - airflow.company.com
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "1"

scheduler:
  replicas: 2  # Active/standby for HA
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "1"

workers:  # Not used with KubernetesExecutor but kept for fallback
  replicas: 0

kubernetes:
  worker_resources:
    requests:
      memory: "2Gi"
      cpu: "1"
    limits:
      memory: "4Gi"
      cpu: "2"
  namespace: airflow-prod
  imagePullSecrets:
    - name: registry-credentials
  env:
    - name: AIRFLOW_HOME
      value: /opt/airflow
    - name: AIRFLOW__CORE__SQL_ALCHEMY_CONN
      valueFrom:
        secretKeyRef:
          name: airflow-secrets
          key: database_url
    - name: AIRFLOW__KUBERNETES__NAMESPACE
      value: airflow-prod

redis:
  enabled: true
  master:
    persistence:
      enabled: true

postgresql:
  enabled: true
  primary:
    persistence:
      enabled: true
  auth:
    password:  # Use external secret manager in prod

logs:
  persistence:
    enabled: true
  storageClass: "gp3"
  accessMode: ReadWriteMany
  size: 100Gi

flower:
  enabled: true
```

### Architecture

```
┌─────────────────────────────────────────────────┐
│                 Ingress (TLS)                    │
└────────────────────┬────────────────────────────┘
                     │
         ┌───────────▼───────────┐
         │     Webserver (x2)     │  Active/standby
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │   Scheduler (x2)       │  Active/standby
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │  PostgreSQL (HA)       │  Metadata DB
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │    Redis (Cluster)     │  Lock + Flower
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │   K8s Namespace        │  Task Pods (ephemeral)
         │   airflow-prod         │
         └────────────────────────┘
```

### What to Explain

1. **KubernetesExecutor:** Each task runs in its own pod; pods are destroyed after completion.
2. **Scheduler HA:** Two schedulers with one active; Redis lock prevents conflicts.
3. **Webserver HA:** Two replicas behind a load balancer/ingress.
4. **Persistent logs:** Shared volume (RWX) so webserver can read task logs from any node.
5. **Resource limits:** Prevent runaway tasks from consuming all cluster resources.

### Principal-Level Considerations

- **Auto-scaling:** Use KEDA or HPA for worker pods based on queue depth.
- **Pod disruption budget:** Ensure scheduler and webserver have PDBs.
- **Network policies:** Restrict pod-to-pod communication.
- **Secret management:** Use External Secrets Operator with AWS Secrets Manager or HashiCorp Vault.
- **Cost optimization:** Use spot instances for worker pods; use on-demand for scheduler/webserver.
- **Backup strategy:** Regular PostgreSQL backups; log rotation with S3/GCS.
- **Monitoring:** Prometheus + Grafana for K8s metrics; custom Airflow metrics.

---
## Q21. Implement Cross-DAG Dependencies With Proper Error Handling

**Scenario:** DAG A must complete before DAG B runs. Handle cases where DAG A fails partially or runs late.

### Solution

```python
# dags/dag_a.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator

with DAG(
    dag_id="data_ingestion",
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
) as dag:
    ingest = PythonOperator(task_id="ingest", python_callable=ingest_data)

# dags/dag_b.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.trigger_rule import TriggerRule
from airflow.operators.bash import BashOperator
from airflow.exceptions import AirflowFailException

def check_upstream_success(ti):
    """Check if upstream DAG completed successfully."""
    from airflow.models import DagRun
    from airflow.utils.state import State
    dr = session.query(DagRun).filter(
        DagRun.dag_id == "data_ingestion",
        DagRun.execution_date == ti.execution_date
    ).first()
    if not dr or dr.state != State.SUCCESS:
        raise AirflowFailException("Upstream DAG did not complete successfully")

with DAG(
    dag_id="data_transformation",
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
) as dag:
    wait_for_ingestion = ExternalTaskMarker(
        task_id="wait_for_ingestion",
        external_dag_id="data_ingestion",
        external_task_id="ingest",
    )
    
    check_status = PythonOperator(
        task_id="check_upstream",
        python_callable=check_upstream_success,
        trigger_rule=TriggerRule.NONE_FAILED_MIN_ONE_SUCCESS,
    )
    
    transform = PythonOperator(task_id="transform", python_callable=transform_data)
    
    notify_on_failure = PythonOperator(
        task_id="notify_failure",
        python_callable=send_alert,
        trigger_rule=TriggerRule.ONE_FAILED,
        on_failure_callback=send_alert,
    )
    
    wait_for_ingestion >> check_status >> transform >> notify_on_failure
```

### What to Explain

1. **`ExternalTaskMarker`:** Creates a dependency marker without blocking resources.
2. **`ExternalTaskSensor`:** Alternative that actively polls the upstream DAG/task status.
3. **`trigger_rule`:** Controls downstream behavior when upstream fails.
4. **Manual status check:** For complex dependency logic beyond simple success/failure.

### Principal-Level Considerations

- Use `ExternalTaskMarker` (lighter) instead of `ExternalTaskSensor` (polls repeatedly) when possible.
- For complex inter-DAG workflows, consider consolidating into a single DAG with TaskGroups.
- Implement a DAG orchestration layer for cross-team dependencies.
- Use event-driven triggers (cloud events, Kafka) instead of schedule-based polling.

---

## Q22. Build a Multi-Tenant Data Pipeline Framework

**Scenario:** Support 50+ customers with similar ETL pipelines but different sources, schemas, and schedules without 50+ separate DAGs.

### Solution

```python
# dags/multi_tenant_factory.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.providers.http.hooks.http import HttpHook
from airflow.operators.python import BranchPythonOperator
from airflow.utils.trigger_rule import TriggerRule

def get_tenant_configs():
    """Load tenant configurations from a database or config store."""
    # In production, read from a config table in PostgreSQL
    return {
        "tenant_a": {
            "source_conn": "postgres_tenant_a",
            "target_table": "tenant_a.facts",
            "schedule": "@hourly",
            "regions": ["us-east-1", "eu-west-1"],
            "sla_minutes": 30,
        },
        "tenant_b": {
            "source_conn": "snowflake_tenant_b",
            "target_table": "tenant_b.facts",
            "schedule": "@daily",
            "regions": ["ap-southeast-1"],
            "sla_minutes": 60,
        },
    }

def create_tenant_dag(tenant_id: str, config: dict):
    """Factory function to create a DAG for a single tenant."""
    default_args = {
        "owner": f"data-eng-{tenant_id}",
        "retries": 2,
        "retry_delay": timedelta(minutes=5),
        "sla": timedelta(minutes=config["sla_minutes"]),
    }

    with DAG(
        dag_id=f"etl_{tenant_id}",
        schedule=config["schedule"],
        start_date=datetime(2024, 1, 1),
        catchup=False,
        default_args=default_args,
        tags=[tenant_id, "multi-tenant"],
        doc=f"""
        # ETL Pipeline for {tenant_id}
        
        Source: {config['source_conn']}
        Target: {config['target_table']}
        Regions: {', '.join(config['regions'])}
        """,
    ) as dag:
        from airflow.decorators import task, task_group
        
        @task
        def extract_data():
            hook = PostgresHook(conn_id=config["source_conn"])
            conn = hook.get_conn()
            cursor = conn.cursor()
            cursor.execute("SELECT * FROM source_table")
            rows = cursor.fetchall()
            cursor.close()
            conn.close()
            return rows

        @task_group
        def transform_by_region(region):
            @task
            def process_region():
                # Transform logic specific to region
                pass
            
            return process_region()
        
        @task
        def load_data(data, region_results):
            # Load transformed data to target
            pass
        
        @task
        def validate_data():
            # Per-tenant data quality checks
            pass
        
        data = extract_data()
        regions = config["regions"]
        
        # Use dynamic task mapping for regions
        region_results = transform_by_region.expand(regions)
        load_data(data, region_results) >> validate_data()

    globals()[f"dag_{tenant_id}"] = dag
    return dag

# Generate all tenant DAGs
configs = get_tenant_configs()
for tenant_id, config in configs.items():
    create_tenant_dag(tenant_id, config)
```

### What to Explain

1. **Factory pattern:** One function generates multiple DAGs from config.
2. **Config-driven:** Tenant-specific settings externalized from code.
3. **Task Groups:** Organize per-tenant logic visually.
4. **Dynamic task mapping:** Handle varying numbers of regions per tenant.

### Principal-Level Considerations

- Use a database config table with schema validation for tenant configs.
- Implement a self-service portal for tenants to configure their own pipelines.
- Add per-tenant resource quotas and rate limiting.
- Use separate K8s namespaces or resource pools per tenant tier.
- Implement cross-tenant monitoring dashboards.
- Consider cost allocation and chargeback per tenant.
- Use feature flags for gradual rollout of new pipeline features.

---

## Q23. Debug and Fix a DAG That Causes Scheduler Overload

**Scenario:** After adding a new DAG with 1000 backfill runs, the scheduler becomes unresponsive. The metadata database CPU spikes to 100%.

### Solution

```python
# Immediate mitigation: stop the problematic DAG
# Airflow CLI
# airflow dags pause my_new_dag

# Long-term fix: limit catchup and control backfill
from datetime import datetime, timedelta
from airflow import DAG

@dag(
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,  # CRITICAL: prevents backfill of all historical dates
    max_active_runs=3,  # Limit concurrent DAG runs
    concurrency=5,  # Limit concurrent tasks within a DAG run
    dagrun_timeout=timedelta(hours=4),  # Kill stuck DAG runs
)
def controlled_dag():
    from airflow.decorators import task
    
    @task(
        retries=2,
        retry_delay=timedelta(minutes=5),
        execution_timeout=timedelta(hours=1),  # Kill stuck tasks
    )
    def process():
        pass
    
    process()

dag = controlled_dag()

# For backfill, use controlled backfill command:
# airflow dags backfill controlled_dag \
#   --start-date 2024-01-01 \
#   --end-date 2024-01-31 \
#   --max-active-runs 2 \
#   --parallel 2
```

### What to Explain

1. **`catchup=False`:** Prevents scheduler from creating backfill runs for all missed dates.
2. **`max_active_runs`:** Limits concurrent DAG runs.
3. **`concurrency`:** Limits concurrent tasks per DAG.
4. **`dagrun_timeout`:** Automatically kills stuck DAG runs.
5. **`execution_timeout`:** Kills stuck individual tasks.

### Principal-Level Considerations

- Use `--parallel` flag with backfill to control how many runs start simultaneously.
- Monitor scheduler queue length; if it exceeds 100, pause non-critical DAGs.
- Implement a scheduler resource quota per team/department.
- Use `AIRFLOW__SCHEDULER__MAX_DAGRUNS_PER_LOOP` to limit scheduler throughput.
- Consider using multiple scheduler instances with proper locking.
- Set up alerts on scheduler queue depth and DB connection count.

---

## Q24. Implement a Data Pipeline With Checkpoint/Restart Capability

**Scenario:** A long-running multi-step pipeline (6+ hours) fails at step 4. You need to restart from step 4, not re-run everything.

### Solution

```python
# dags/checkpoint_pipeline.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.decorators import task
from airflow.utils.state import State
import json
import os

# Use a durable checkpoint store (S3, GCS, or PostgreSQL)
CHECKPOINT_TABLE = "pipeline_checkpoints"

@dag(
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    max_active_runs=1,
)
def checkpoint_pipeline():
    @task
    def step1_extract():
        # Extract data
        result = extract_data()
        # Save checkpoint
        save_checkpoint("step1", {"status": "done", "records": len(result)})
        return result

    @task
    def step2_clean(data):
        # Clean data
        result = clean_data(data)
        save_checkpoint("step2", {"status": "done", "records": len(result)})
        return result

    @task
    def step3_transform(data):
        result = transform_data(data)
        save_checkpoint("step3", {"status": "done", "records": len(result)})
        return result

    @task
    def step4_load(data):
        result = load_data(data)
        save_checkpoint("step4", {"status": "done", "records": len(result)})
        return result

    @task
    def step5_validate(data):
        validate_data(data)
        save_checkpoint("step5", {"status": "done"})

    @task
    def step6_notify(data):
        send_notification(data)
        save_checkpoint("step6", {"status": "done"})

    # Linear chain - Airflow naturally restarts from failed task
    data = step1_extract()
    data = step2_clean(data)
    data = step3_transform(data)
    data = step4_load(data)
    step5_validate(data)
    step6_notify(data)

dag = checkpoint_pipeline()

def save_checkpoint(step: str, state: dict):
    """Save checkpoint to a durable store."""
    from airflow.hooks.postgres_hook import PostgresHook
    hook = PostgresHook(conn_id="checkpoint_db")
    conn = hook.get_conn()
    cursor = conn.cursor()
    cursor.execute(
        """INSERT INTO pipeline_checkpoints 
           (step, state, updated_at) 
           VALUES (%s, %s, CURRENT_TIMESTAMP)
           ON CONFLICT (step) 
           DO UPDATE SET state = EXCLUDED.state, updated_at = CURRENT_TIMESTAMP""",
        (step, json.dumps(state))
    )
    conn.commit()
    cursor.close()
    conn.close()
```

### What to Explain

1. **Airflow's native retry:** Tasks retry from the failed step by default.
2. **XCom for data passing:** Intermediate results are stored in the metadata DB.
3. **Durable checkpoints:** For truly long-running pipelines, save state to external storage.
4. **`max_active_runs=1`:** Prevents parallel runs of the same logical pipeline.

### Principal-Level Considerations

- For pipelines > 4 hours, use intermediate landing zones (S3, GCS, Delta Lake).
- Implement idempotent writes so partial retries don't create duplicates.
- Use file-based checkpoints for large datasets (don't store data in XCom).
- Consider using Apache Beam or Spark for built-in checkpoint support.
- Add a "resume" endpoint that reads checkpoints and triggers from the right step.
- Use `AIRFLOW__CORE__XCOM_BACKEND` to change where XCom data is stored for large payloads.

---

## Q25. Design an Airflow-Based CI/CD Pipeline for DAG Development

**Scenario:** Set up a complete CI/CD pipeline for Airflow DAG development with testing, linting, and deployment.

### Solution

```yaml
# .github/workflows/dag-ci.yml
name: DAG CI/CD

on:
  pull_request:
    paths:
      - "dags/**"
      - "plugins/**"
  push:
    branches: [main]
    paths:
      - "dags/**"
      - "plugins/**"

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install dependencies
        run: |
          pip install flake8 pylint mypy
          pip install apache-airflow[provider] --constraint \
            https://raw.githubusercontent.com/apache/airflow/constraints-2.8.0/constraints-3.11.txt
      - name: Lint DAGs
        run: |
          flake8 dags/ --max-line-length=120
          pylint dags/ --disable=C0114,C0115,C0116
      - name: Check for common mistakes
        run: |
          # Check for globals() usage
          grep -rn "globals()" dags/ || true
          # Check for hardcoded credentials
          grep -rEn "(password|secret|key)\s*=\s*['\"]" dags/ || true

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install dependencies
        run: |
          pip install apache-airflow --constraint \
            https://raw.githubusercontent.com/apache/airflow/constraints-2.8.0/constraints-3.11.txt
          pip install pytest pytest-airflow
      - name: Run unit tests
        run: pytest tests/dags/ -v
      - name: Validate DAG structure
        run: |
          export AIRFLOW_HOME=/tmp/airflow
          export AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=sqlite:////tmp/airflow/airflow.db
          airflow db init
          python -c "
          from airflow.models import DagBag
          bag = DagBag(dag_folder='dags/', read_dags_from_db=False)
          assert len(bag.import_errors) == 0, f'Import errors: {bag.import_errors}'
          for dag_id, dag in bag.dags.items():
              print(f'✓ {dag_id}: {len(dag.tasks)} tasks')
          "

  integration-test:
    runs-on: ubuntu-latest
    needs: test
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: airflow
          POSTGRES_PASSWORD: airflow
          POSTGRES_DB: airflow
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      - name: Start Airflow
        run: |
          export AIRFLOW_HOME=/tmp/airflow
          export AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@localhost:5432/airflow
          export AIRFLOW__CORE__EXECUTOR=SequentialExecutor
          airflow db init
          airflow users create \
            --username admin \
            --firstname admin \
            --lastname admin \
            --role Admin \
            --email admin@test.com \
            --password admin
      - name: Run integration tests
        run: pytest tests/integration/ -v --airflow

  deploy:
    runs-on: ubuntu-latest
    needs: integration-test
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: |
          # Copy DAGs to production Airflow instance
          # Options: sync S3 bucket, update Git repo, or use Helm
          aws s3 sync dags/ s3://${{ secrets.AIRFLOW_DAGS_BUCKET }}/dags/
```

### Project Structure

```
project/
├── dags/
│   ├── __init__.py
│   ├── etl_pipeline.py
│   └── reporting_pipeline.py
├── plugins/
│   ├── __init__.py
│   └── custom_operators.py
├── tests/
│   ├── dags/
│   │   ├── test_etl_pipeline.py
│   │   └── test_reporting_pipeline.py
│   └── integration/
│       └── test_full_pipeline.py
├── .github/workflows/
│   └── dag-ci.yml
├── requirements.txt
├── pre-commit-config.yaml
└── docker-compose.test.yml
```

### What to Explain

1. **Multi-stage CI:** Lint → Unit Test → Integration Test → Deploy.
2. **`DagBag` validation:** Ensures DAGs parse correctly before deployment.
3. **`pytest-airflow`:** Airflow's official testing framework.
4. **`SequentialExecutor`:** Use in tests to avoid needing Celery/K8s.
5. **S3 sync deployment:** Simple DAG deployment strategy.

### Principal-Level Considerations

- Use pre-commit hooks for local linting before push.
- Implement a staging environment that mirrors production.
- Use feature flags for gradual rollout of new DAGs.
- Add performance regression tests for DAG import time.
- Implement automated DAG deprecation detection.
- Use infrastructure-as-code (Terraform) for managing Airflow deployment.
- Consider a DAG versioning system with rollback capability.

---

## Q26. Implement Custom Monitoring and Alerting for Data Pipelines

**Scenario:** Go beyond Airflow's built-in alerts to implement comprehensive monitoring with custom metrics, SLA tracking, and data quality dashboards.

### Solution

```python
# plugins/custom_monitoring.py
import time
import json
from typing import Any
from airflow.plugins_manager import AirflowPlugin
from airflow.utils.log.logging_mixin import LoggingMixin
from prometheus_client import Counter, Histogram, Gauge

# Prometheus metrics
TASK_DURATION = Histogram(
    "airflow_task_duration_seconds",
    "Duration of task execution",
    ["dag_id", "task_id"]
)

TASK_FAILURES = Counter(
    "airflow_task_failures_total",
    "Number of task failures",
    ["dag_id", "task_id", "error_type"]
)

RECORDS_PROCESSED = Counter(
    "airflow_records_processed_total",
    "Number of records processed",
    ["dag_id", "task_id"]
)

DATA_QUALITY_SCORE = Gauge(
    "airflow_data_quality_score",
    "Data quality score (0-100)",
    ["dag_id", "task_id"]
)

class MonitoringMixin(LoggingMixin):
    """Mixin for custom monitoring in tasks."""
    
    def __init__(self, dag_id: str, task_id: str):
        self.dag_id = dag_id
        self.task_id = task_id
        self.start_time = None
    
    def start(self):
        self.start_time = time.time()
    
    def end(self, records_processed: int = 0, quality_score: float = 100.0):
        duration = time.time() - self.start_time
        labels = {"dag_id": self.dag_id, "task_id": self.task_id}
        TASK_DURATION.labels(**labels).observe(duration)
        if records_processed > 0:
            RECORDS_PROCESSED.labels(**labels).inc(records_processed)
        DATA_QUALITY_SCORE.labels(**labels).set(quality_score)
        self.log.info(
            f"Task {self.task_id} completed in {duration:.2f}s, "
            f"processed {records_processed} records, quality={quality_score:.1f}"
        )
    
    def fail(self, error_type: str):
        TASK_FAILURES.labels(
            dag_id=self.dag_id,
            task_id=self.task_id,
            error_type=error_type
        ).inc()

# Custom on_failure_callback
def custom_on_failure(context: dict):
    """Enhanced failure callback with detailed logging and alerting."""
    ti = context["ti"]
    monitoring = MonitoringMixin(ti.dag_id, ti.task_id)
    
    error_type = "unknown"
    if context.get("exception"):
        error_type = type(context["exception"]).__name__
    monitoring.fail(error_type)
    
    # Determine severity
    severity = "P3"  # Default
    if ti.task_id in ["load", "transform"]:
        severity = "P1"  # Critical data tasks
    elif ti.max_retries and ti.try_number > ti.max_retries:
        severity = "P2"  # Exhausted retries
    
    # Send to alerting system
    send_alert(
        severity=severity,
        message=f"DAG {ti.dag_id} / Task {ti.task_id} failed",
        details={
            "execution_date": str(ti.execution_date),
            "try_number": ti.try_number,
            "error": error_type,
            "duration": ti.duration,
        }
    )

def send_alert(severity: str, message: str, details: dict):
    """Send alert to multiple channels."""
    import requests
    
    # Slack
    if severity in ["P1", "P2"]:
        requests.post(
            "https://hooks.slack.com/services/xxx/yyy/zzz",
            json={
                "text": f"🚨 {severity}: {message}",
                "blocks": [
                    {
                        "type": "section",
                        "text": {"type": "mrkdwn", "text": f"*{severity}: {message}*"},
                    },
                    {
                        "type": "context",
                        "elements": [{"type": "mrkdwn", "text": str(details)}],
                    },
                ],
            },
        )
    
    # PagerDuty for P1
    if severity == "P1":
        requests.post(
            "https://events.pagerduty.com/v2/enqueue",
            json={
                "routing_key": "YOUR_ROUTING_KEY",
                "event_action": "trigger",
                "payload": {
                    "summary": message,
                    "severity": "critical",
                    "source": "airflow",
                    "custom_details": details,
                },
            },
        )
```

### Grafana Dashboard Panels

```json
{
  "dashboard": {
    "title": "Airflow Data Pipeline Monitoring",
    "panels": [
      {
        "title": "Task Success Rate (24h)",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(airflow_task_failures_total[1h])) / sum(rate(airflow_task_duration_seconds_count[1h]))"
          }
        ]
      },
      {
        "title": "Task Duration P95",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(airflow_task_duration_seconds_bucket[1h]))"
          }
        ]
      },
      {
        "title": "Data Quality Score by DAG",
        "type": "gauge",
        "targets": [
          {
            "expr": "airflow_data_quality_score"
          }
        ]
      },
      {
        "title": "Records Processed (24h)",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(increase(airflow_records_processed_total[24h])) by (dag_id)"
          }
        ]
      }
    ]
  }
}
```

### What to Explain

1. **Prometheus metrics:** Standard metrics types (Counter, Histogram, Gauge) for different data.
2. **Custom callbacks:** `on_failure_callback` for alerting.
3. **Severity levels:** P1 (critical) → PagerDuty, P2 → Slack, P3 → email.
4. **Grafana panels:** Visualize task performance, quality, and volume.
5. **`MonitoringMixin`:** Reusable monitoring code for any task.

### Principal-Level Considerations

- Export Airflow's built-in metrics too (scheduler, executor, DB pool).
- Implement SLA violation detection with custom thresholds per DAG.
- Add data quality trends over time to detect degradation.
- Use correlation between task duration and data volume to detect anomalies.
- Implement a runbook system linked to each alert.
- Consider using OpenTelemetry for distributed tracing across pipeline stages.
- Set up alert fatigue management: suppress recurring non-critical alerts.

---
## Q27. Build a Reusable Pipeline Framework With Dynamic Configuration

**Scenario:** Create a framework that lets data analysts define pipelines using YAML/JSON config files without writing Python code, while engineers maintain the framework.

### Solution

```python
# plugins/pipeline_framework.py
"""
A reusable pipeline framework that reads pipeline definitions from YAML.
Analysts write config files; the framework generates Airflow DAGs.
"""
import yaml
from pathlib import Path
from datetime import datetime, timedelta
from airflow import DAG
from airflow.decorators import task
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.providers.snowflake.hooks.snowflake import SnowflakeHook
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

CONFIG_DIR = Path("/opt/airflow/pipeline_configs")

def load_pipeline_configs():
    """Load all pipeline config YAML files."""
    configs = []
    for config_file in CONFIG_DIR.glob("*.yaml"):
        with open(config_file) as f:
            config = yaml.safe_load(f)
            config["_source_file"] = config_file.name
            configs.append(config)
    return configs

def get_hook(source_type: str, conn_id: str):
    """Get the appropriate Airflow hook based on source type."""
    hooks = {
        "postgres": lambda: PostgresHook(conn_id=conn_id),
        "snowflake": lambda: SnowflakeHook(conn_id=conn_id),
        "s3": lambda: S3Hook(conn_id=conn_id),
    }
    if source_type not in hooks:
        raise ValueError(f"Unsupported source type: {source_type}")
    return hooks[source_type]()

@task
def extract_from_db(conn_id: str, query: str, source_type: str = "postgres"):
    """Generic database extraction."""
    hook = get_hook(source_type, conn_id)
    if source_type == "s3":
        return None  # S3 handled differently
    conn = hook.get_conn()
    cursor = conn.cursor()
    cursor.execute(query)
    rows = cursor.fetchall()
    cursor.close()
    conn.close()
    return rows

@task
def transform_data(data, transformations: list):
    """Apply a list of transformations to data."""
    import pandas as pd
    df = pd.DataFrame(data)
    for transform in transformations:
        if transform["type"] == "filter":
            df = df[df[transform["column"]] == transform["value"]]
        elif transform["type"] == "rename":
            df = df.rename(columns=transform["mapping"])
        elif transform["type"] == "add_column":
            df[transform["name"]] = df.eval(transform["expression"])
        elif transform["type"] == "drop_nulls":
            df = df.dropna(subset=transform.get("columns", None))
    return df.to_dict(orient="records")

@task
def load_to_db(conn_id: str, table: str, data: list, source_type: str = "postgres"):
    """Generic database loading."""
    hook = get_hook(source_type, conn_id)
    if source_type == "postgres":
        conn = hook.get_conn()
        cursor = conn.cursor()
        for row in data:
            columns = ", ".join(row.keys())
            placeholders = ", ".join(["%s"] * len(row))
            cursor.execute(
                f"INSERT INTO {table} ({columns}) VALUES ({placeholders})",
                list(row.values())
            )
        conn.commit()
        cursor.close()
        conn.close()

@task
def validate_data(data: list, rules: list):
    """Validate data against defined rules."""
    from airflow.exceptions import AirflowQualityException
    for rule in rules:
        if rule["type"] == "row_count":
            if len(data) < rule["min"]:
                raise AirflowQualityException(
                    f"Row count {len(data)} below minimum {rule['min']}"
                )
        elif rule["type"] == "null_check":
            nulls = sum(1 for row in data if row.get(rule["column"]) is None)
            if nulls > rule.get("max_nulls", 0):
                raise AirflowQualityException(
                    f"Nulls in {rule['column']}: {nulls} > max {rule['max_nulls']}"
                )

def create_pipeline_dag(config: dict):
    """Create a DAG from a pipeline configuration."""
    pipeline_id = config["pipeline_id"]
    schedule = config.get("schedule", "@daily")
    
    default_args = {
        "owner": config.get("owner", "data-team"),
        "retries": config.get("retries", 1),
        "retry_delay": timedelta(minutes=config.get("retry_delay_minutes", 10)),
    }
    
    with DAG(
        dag_id=f"auto_{pipeline_id}",
        schedule=schedule,
        start_date=datetime.fromisoformat(config["start_date"]),
        catchup=False,
        default_args=default_args,
        tags=[config.get("team", "data"), "auto-generated"],
        doc=config.get("description", ""),
    ) as dag:
        # Extract
        extract_task = extract_from_db(
            conn_id=config["source"]["conn_id"],
            query=config["source"]["query"],
            source_type=config["source"].get("type", "postgres"),
        )
        
        # Transform (if defined)
        if config.get("transformations"):
            transform_task = transform_data(
                extract_task,
                transformations=config["transformations"],
            )
            main_data = transform_task
        else:
            main_data = extract_task
        
        # Validate (if defined)
        if config.get("validation_rules"):
            validate_data(main_data, rules=config["validation_rules"])
        
        # Load
        load_to_db(
            conn_id=config["destination"]["conn_id"],
            table=config["destination"]["table"],
            data=main_data,
            source_type=config["destination"].get("type", "postgres"),
        )
    
    return dag

# Generate all DAGs from configs
for config in load_pipeline_configs():
    dag = create_pipeline_dag(config)
    globals()[f"auto_{config['pipeline_id']}"] = dag
```

### Example Pipeline Config

```yaml
# pipeline_configs/sales_report.yaml
pipeline_id: sales_daily
description: "Daily sales ETL from Postgres to Snowflake"
owner: analytics-team
schedule: "0 6 * * *"  # 6 AM daily
start_date: "2024-01-01"
retries: 2
retry_delay_minutes: 15

source:
  conn_id: postgres_sales
  type: postgres
  query: |
    SELECT 
      order_id, customer_id, amount, order_date, 
      region, product_category
    FROM sales.orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '1 day'

transformations:
  - type: filter
    column: amount
    value: 0
    operator: ">"
  - type: add_column
    name: amount_tax
    expression: "amount * 0.1"

validation_rules:
  - type: row_count
    min: 1
  - type: null_check
    column: order_id
    max_nulls: 0

destination:
  conn_id: snowflake_analytics
  type: snowflake
  table: analytics.sales_daily
```

### What to Explain

1. **Config-driven pipelines:** YAML defines the entire pipeline flow.
2. **Reusable tasks:** Generic extract/transform/load tasks work with any config.
3. **Factory pattern:** `create_pipeline_dag()` generates DAGs programmatically.
4. **Validation framework:** Declarative validation rules in config.

### Principal-Level Considerations

- Add a config validation service that checks YAML against a schema (JSON Schema).
- Implement a self-service web UI for analysts to create/edit configs.
- Add support for complex transformations (SQL-based, Python UDFs).
- Implement a preview mode to test configs without loading data.
- Add automatic schema detection and evolution handling.
- Consider versioning configs and supporting rollbacks.
- Implement a governance layer: approve configs before they become DAGs.

---

## Q28. Design a Disaster Recovery Plan for Airflow

**Scenario:** Design a complete disaster recovery strategy for your production Airflow deployment covering data loss, prolonged outages, and geographic failover.

### Solution

```yaml
# disaster-recovery/strategy.md

# Disaster Recovery Plan for Airflow

## 1. Data Backup Strategy

### Metadata Database (PostgreSQL)
- Continuous WAL archiving to S3 (every 5 minutes)
- Full daily backup at 2 AM UTC
- Point-in-time recovery to any second
- Retention: 30 days PITR, 1 year full backups
- Test restore monthly

### XCom Data
- Custom XCom backend writing to S3
- Critical XCom values (watermarks, offsets) replicated to secondary DB
- File-based checkpoints in Delta Lake for long-running pipelines

### Logs
- GCS/S3 remote logging enabled
- Logs retained for 90 days
- Structured JSON logs with correlation IDs

### DAG Code
- Git repository (already versioned)
- S3 mirror of DAG folder with versioning
- Helm chart values stored in Git

## 2. Backup Automation

# airflow-backup.sh
#!/bin/bash
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BUCKET="s3://company-airflow-backups"

# 1. PostgreSQL backup
pg_dump -U airflow -h primary.airflow.db \
  --format=custom --compress=9 \
  --file=/tmp/airflow_$BACKUP_DATE.dump airflow

aws s3 cp /tmp/airflow_$BACKUP_DATE.dump \
  "$BUCKET/db/$BACKUP_DATE/"

# 2. S3 mirror of DAGs
aws s3 sync /opt/airflow/dags/ "$BUCKET/dags/$BACKUP_DATE/"

# 3. XCom backup (if using custom backend)
aws s3 sync /opt/airflow/xcom/ "$BUCKET/xcom/$BACKUP_DATE/"

# 4. Config backup
tar czf /tmp/airflow_config_$BACKUP_DATE.tar.gz \
  /opt/airflow/airflow.cfg \
  /opt/airflow/pipeline_configs/
aws s3 cp /tmp/airflow_config_$BACKUP_DATE.tar.gz \
  "$BUCKET/config/$BACKUP_DATE/"
```

## 3. Recovery Procedures

### RTO (Recovery Time Objective): 1 hour
### RPO (Recovery Point Objective): 5 minutes (for DB)

### Scenario 1: Single AZ Failure
- Secondary scheduler takes over (Redis Sentinel)
- K8s reschedules pods in healthy AZ
- DB connection string updated via DNS failover
- Estimated recovery: 5-15 minutes

### Scenario 2: Database Complete Loss
1. Provision new RDS instance in standby region
2. Restore from S3 backup (PITR to 5 minutes before failure)
3. Update DNS/connection strings
4. Restart scheduler and workers
5. Verify DAG runs and resume backfill if needed
6. Estimated recovery: 30-60 minutes

### Scenario 3: Full Region Failure
1. Failover to secondary region (pre-provisioned standby)
2. Route traffic via global DNS (Route 53 health check)
3. Restore DB from cross-region backup
4. Verify all connections and external dependencies
5. Estimated recovery: 1-2 hours

## 4. Monitoring for DR

- CloudWatch alarms on RDS metrics (connections, IOPS, CPU)
- Custom metric: scheduler_heartbeat (fails if scheduler stops)
- Synthetic check: run a test DAG every 5 minutes in each region
- Automated runbook execution for common failure scenarios
```

### What to Explain

1. **RTO vs RPO:** Time to recover vs data loss tolerance.
2. **Backup layers:** DB, XCom, logs, code, config — each backed up separately.
3. **Recovery procedures:** Documented step-by-step for each failure scenario.
4. **Monitoring:** Synthetic checks detect failures before users do.

### Principal-Level Considerations

- Pre-provision a standby Airflow in a secondary region.
- Test DR quarterly with game day exercises.
- Automate failover where possible (reduces human error during crisis).
- Maintain a war room runbook with clear roles and escalation paths.
- Consider using managed services (CloudControl, Astronomer) for built-in DR.
- Cross-region replication for all data stores (DB, Redis, object storage).
- Business impact analysis: which pipelines are critical vs non-critical.

---

## Q29. Implement Airflow With Deferrable Operators for Cost Optimization

**Scenario:** You have 200+ DAGs that frequently use Sensors, which keep workers occupied waiting for external conditions. Optimize costs by making them deferrable.

### Solution

```python
# plugins/deferrable_operators.py
from typing import Any
from airflow.triggers.base import BaseTrigger, TriggerEvent
from airflow.utils.state import State
from airflow.models.baseoperator import BaseOperator
import asyncio
import aiohttp

class WaitForExternalTrigger(BaseTrigger):
    """Trigger that polls an external API until a condition is met."""
    
    def __init__(
        self,
        url: str,
        check_path: str,
        expected_value: Any,
        interval: float = 30.0,
        timeout: float = 3600.0,
        **kwargs
    ):
        super().__init__(**kwargs)
        self.url = url
        self.check_path = check_path
        self.expected_value = expected_value
        self.interval = interval
        self.timeout = timeout
    
    async def run(self):
        """Poll the external service until condition is met."""
        import json
        start_time = asyncio.get_event_loop().time()
        
        while True:
            elapsed = asyncio.get_event_loop().time() - start_time
            if elapsed > self.timeout:
                yield TriggerEvent({"status": "error", "message": "Timeout"})
                return
            
            try:
                async with aiohttp.ClientSession() as session:
                    async with session.get(f"{self.url}/{self.check_path}") as resp:
                        data = await resp.json()
                        # Navigate nested path like "status.data.ready"
                        value = data
                        for key in self.check_path.split("."):
                            value = value[key]
                        
                        if value == self.expected_value:
                            yield TriggerEvent({"status": "success", "data": data})
                            return
            except Exception as e:
                self.log.warning(f"Check failed: {e}")
            
            await asyncio.sleep(self.interval)

class WaitForExternalOperator(BaseOperator):
    """Deferrable operator that waits for an external condition."""
    
    def __init__(
        self,
        url: str,
        check_path: str,
        expected_value: Any,
        interval: float = 30.0,
        timeout: float = 3600.0,
        **kwargs
    ):
        super().__init__(**kwargs)
        self.url = url
        self.check_path = check_path
        self.expected_value = expected_value
        self.interval = interval
        self.timeout = timeout
    
    def execute(self, context):
        """Immediately defer to the trigger."""
        self.log.info(f"Deferring to WaitForExternalTrigger: {self.url}/{self.check_path}")
        self.defer(
            trigger=WaitForExternalTrigger(
                url=self.url,
                check_path=self.check_path,
                expected_value=self.expected_value,
                interval=self.interval,
                timeout=self.timeout,
            ),
            method_name="execute_complete",
        )
    
    def execute_complete(self, context, event):
        """Called when the trigger fires."""
        if event["status"] == "error":
            raise TimeoutError(f"Wait for external condition timed out: {event['message']}")
        self.log.info(f"External condition met: {event['data']}")
        return event["data"]

# Usage in a DAG
from airflow import DAG
from datetime import datetime

with DAG(
    dag_id="deferrable_pipeline",
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
) as dag:
    wait_for_upstream = WaitForExternalOperator(
        task_id="wait_for_data_ready",
        url="https://data-service.example.com/api/v1",
        check_path="jobs.ingestion.status",
        expected_value="complete",
        interval=30.0,
        timeout=3600.0,
    )
    
    # Subsequent tasks run normally after the condition is met
```

### Non-Deferrable vs Deferrable Comparison

```
Non-Deferrable Sensor (ExternalTaskSensor):
┌──────────────────────────────────────────────┐
│  Worker occupied for 2 hours                 │
│  ┌──────────────────────────────────────┐    │
│  │ Waiting (CPU+Memory consumed)         │    │
│  └──────────────────────────────────────┘    │
│  Worker cannot process other tasks            │
└──────────────────────────────────────────────┘

Deferrable Operator (WaitForExternalOperator):
┌──────────────────────────────────────────────┐
│  Worker releases itself after 1 line of code  │
│  ┌──────────────────────────────────────┐    │
│  │ Trigger running in triggerer (cheap)  │    │
│  └──────────────────────────────────────┘    │
│  Worker is free to process other tasks        │
└──────────────────────────────────────────────┘
```

### What to Explain

1. **Deferrable pattern:** `execute()` calls `self.defer()` with a trigger; trigger runs asynchronously; `execute_complete()` handles the result.
2. **`Triggerer`:** A lightweight process that runs triggers; doesn't need full worker resources.
3. **Cost savings:** One triggerer can handle hundreds of triggers; workers handle actual work.
4. **`aiohttp`:** Async HTTP client for non-blocking polling.

### Principal-Level Considerations

- Enable the Triggerer: `AIRFLOW__TRIGGERER__WORKER_CONCURRENCY=100`.
- Convert existing Sensors to deferrable equivalents.
- For AWS: use `WaitForS3KeyTrigger` (already deferrable in AWS provider).
- Monitor triggerer queue depth; scale triggerer workers if it grows.
- Consider timeout strategies per trigger type (short for APIs, long for data availability).
- Calculate cost savings: (number of sensors × average wait time × worker cost) vs triggerer cost.

---

## Q30. Build a Complete Data Platform Orchestration Layer

**Scenario:** Design the complete orchestration layer for a data platform serving 100+ pipelines across multiple teams, with governance, observability, and self-service capabilities.

### Solution

```python
# plugins/platform_orchestration.py
"""
Platform Orchestration Layer - Principal Level Architecture
Provides governance, observability, and self-service for Airflow.
"""

# 1. Pipeline Registry
class PipelineRegistry:
    """Central registry for all data pipelines."""
    
    def __init__(self, db_conn_id: str = "platform_db"):
        from airflow.providers.postgres.hooks.postgres import PostgresHook
        self.hook = PostgresHook(conn_id=db_conn_id)
    
    def register_pipeline(self, dag_id: str, metadata: dict) -> None:
        """Register a pipeline with governance metadata."""
        conn = self.hook.get_conn()
        cursor = conn.cursor()
        cursor.execute(
            """INSERT INTO pipeline_registry 
               (dag_id, owner, team, criticality, sla_minutes, 
                data_classification, description, tags, created_at)
               VALUES (%s, %s, %s, %s, %s, %s, %s, %s, CURRENT_TIMESTAMP)
               ON CONFLICT (dag_id) 
               DO UPDATE SET 
                   owner = EXCLUDED.owner,
                   team = EXCLUDED.team,
                   criticality = EXCLUDED.criticality,
                   sla_minutes = EXCLUDED.sla_minutes,
                   data_classification = EXCLUDED.data_classification,
                   description = EXCLUDED.description,
                   tags = EXCLUDED.tags""",
            (
                dag_id,
                metadata.get("owner"),
                metadata.get("team"),
                metadata.get("criticality", "medium"),
                metadata.get("sla_minutes", 60),
                metadata.get("data_classification", "internal"),
                metadata.get("description", ""),
                metadata.get("tags", "[]"),
            )
        )
        conn.commit()
        cursor.close()
        conn.close()
    
    def get_critical_pipelines(self) -> list:
        """Get all critical pipelines for priority scheduling."""
        conn = self.hook.get_conn()
        cursor = conn.cursor()
        cursor.execute(
            "SELECT dag_id, sla_minutes, owner FROM pipeline_registry WHERE criticality = 'critical'"
        )
        pipelines = cursor.fetchall()
        cursor.close()
        conn.close()
        return pipelines

# 2. Resource Governance
class ResourceGovernor:
    """Enforce resource quotas per team."""
    
    TEAM_QUOTAS = {
        "analytics": {"max_active_tasks": 20, "cpu_limit": "4", "memory_limit": "8Gi"},
        "ml-platform": {"max_active_tasks": 50, "cpu_limit": "8", "memory_limit": "16Gi"},
        "reporting": {"max_active_tasks": 10, "cpu_limit": "2", "memory_limit": "4Gi"},
    }
    
    def get_team_config(self, team: str) -> dict:
        """Get resource config for a team."""
        return self.TEAM_QUOTAS.get(team, {
            "max_active_tasks": 5,
            "cpu_limit": "1",
            "memory_limit": "2Gi",
        })

# 3. Self-Service Pipeline Creation
class PipelineFactory:
    """Allow teams to create standard pipelines via API."""
    
    @staticmethod
    def create_etl_pipeline(request: dict) -> dict:
        """Generate a standard ETL pipeline from a template."""
        pipeline_id = request["pipeline_id"]
        
        # Generate DAG code
        dag_code = f'''
from datetime import datetime, timedelta
from airflow import DAG
from airflow.decorators import task
from airflow.providers.postgres.hooks.postgres import PostgresHook

default_args = {{
    "owner": "{request["owner"]}",
    "retries": {request.get("retries", 2)},
    "retry_delay": timedelta(minutes={request.get("retry_delay_minutes", 10)}),
}}

with DAG(
    dag_id="auto_{pipeline_id}",
    schedule="{request["schedule"]}",
    start_date=datetime.fromisoformat("{request["start_date"]}"),
    catchup=False,
    default_args=default_args,
    tags=["{request["team"]}", "auto-generated"],
) as dag:

    @task
    def extract():
        hook = PostgresHook(conn_id="{request["source_conn_id"]}")
        conn = hook.get_conn()
        cursor = conn.cursor()
        cursor.execute("""{request["source_query"]}""")
        rows = cursor.fetchall()
        cursor.close()
        conn.close()
        return rows

    @task
    def transform(data):
        # Custom transformation logic
        return data

    @task
    def load(data):
        hook = PostgresHook(conn_id="{request["target_conn_id"]}")
        conn = hook.get_conn()
        cursor = conn.cursor()
        for row in data:
            cursor.execute(
                "INSERT INTO {request["target_table"]} VALUES (%s, %s, %s)",
                row
            )
        conn.commit()
        cursor.close()
        conn.close()

    extract() >> transform() >> load()
'''
        return {"dag_code": dag_code, "pipeline_id": pipeline_id}

# 4. Platform Dashboard API
class PlatformAPI:
    """REST API for platform observability and management."""
    
    @staticmethod
    def get_platform_health():
        """Get overall platform health metrics."""
        from airflow.models import DagRun, TaskInstance
        from airflow.utils.state import State
        from sqlalchemy import func
        
        session = Settings.session
        total_runs = session.query(func.count(DagRun.id)).filter(
            DagRun.execution_date >= datetime.now() - timedelta(hours=24)
        ).scalar()
        
        failed_runs = session.query(func.count(DagRun.id)).filter(
            DagRun.execution_date >= datetime.now() - timedelta(hours=24),
            DagRun.state == State.FAILURE
        ).scalar()
        
        return {
            "total_runs_24h": total_runs,
            "failed_runs_24h": failed_runs,
            "success_rate": ((total_runs - failed_runs) / total_runs * 100) if total_runs else 100,
            "critical_pipelines": PipelineRegistry().get_critical_pipelines(),
        }
```

### Architecture Overview

```
┌────────────────────────────────────────────────────────────┐
│                    Platform Orchestration Layer             │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Pipeline     │  │  Resource    │  │  Self-Service    │  │
│  │  Registry     │  │  Governor    │  │  Pipeline Factory│  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                  │                    │            │
│  ┌──────▼──────────────────▼────────────────────▼──────────┐ │
│  │              Governance & Policy Engine                  │ │
│  │  - Team quotas                                           │ │
│  │  - SLA enforcement                                       │ │
│  │  - Data classification                                   │ │
│  │  - Approval workflows                                    │ │
│  └────────────────────┬────────────────────────────────────┘ │
│                        │                                     │
│  ┌─────────────────────▼────────────────────────────────────┐ │
│  │              Airflow Core (Scheduling)                    │ │
│  └─────────────────────┬────────────────────────────────────┘ │
│                        │                                     │
│  ┌─────────────────────▼────────────────────────────────────┐ │
│  │           Observability & Platform API                    │ │
│  │  - Health dashboards                                      │ │
│  │  - Cost tracking                                          │ │
│  │  - SLA reporting                                          │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

### What to Explain

1. **Pipeline Registry:** Central source of truth for all pipelines with governance metadata.
2. **Resource Governor:** Enforces team quotas to prevent resource contention.
3. **Self-Service Factory:** Enables non-engineers to create standard pipelines.
4. **Platform API:** Provides observability and management endpoints.
5. **Governance Layer:** Ties together registration, quotas, and approval workflows.

### Principal-Level Considerations

- **Governance:** Implement a review process for new pipelines (automated checks + human review).
- **Cost allocation:** Track compute cost per team and show chargeback reports.
- **Data lineage:** Integrate with OpenLineage for end-to-end data lineage tracking.
- **Security:** Role-based access control (RBAC) for pipeline management.
- **Multi-tenancy:** Isolate teams with separate K8s namespaces and resource pools.
- **Documentation:** Auto-generate pipeline documentation from registry metadata.
- **Onboarding:** Create a self-service portal with templates, tutorials, and a sandbox environment.
- **Incident management:** Automated incident creation for critical pipeline failures with runbooks.

---