# Workflow Orchestration Skill

## Purpose
This skill teaches Genie Code how to orchestrate complex data pipelines using Databricks Workflows, including job creation, task dependencies, parameter passing, retry logic, notifications, and multi-task orchestration patterns.

## When to Use
* Orchestrating multi-step data pipelines
* Creating dependencies between tasks
* Passing parameters between tasks
* Implementing retry and error handling
* Scheduling complex workflows
* Coordinating notebook, Python, SQL, and DLT tasks
* Setting up notifications and alerts
* Managing production workflows

## Key Concepts

### 1. Jobs and Tasks
* **Job**: Collection of tasks with dependencies
* **Task**: Single unit of work (notebook, Python, SQL, JAR, DLT)
* **Task dependencies**: Define execution order
* **Task types**: Notebook, Python wheel, JAR, SQL, DLT pipeline

### 2. Task Dependencies
* **Sequential**: Tasks run one after another
* **Parallel**: Tasks run simultaneously
* **Conditional**: Tasks run based on conditions
* **Fan-out/fan-in**: Multiple parallel tasks converge

### 3. Parameter Passing
* **Job parameters**: Set at job level
* **Task parameters**: Override job parameters
* **Dynamic parameters**: Pass outputs between tasks
* **Widgets**: UI for parameter input

### 4. Retry and Error Handling
* **Retry policies**: Automatic retry on failure
* **Max retries**: Limit retry attempts
* **Timeout**: Task execution limits
* **Error notifications**: Alert on failures

### 5. Notifications
* **Email**: Send to individuals or groups
* **Webhooks**: Integrate with external systems
* **On start/success/failure**: Flexible notification triggers

---

## Examples

### Example 1: Basic Multi-Task Workflow

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import (
    Task, NotebookTask, PythonWheelTask, SqlTask,
    JobCluster, ClusterSpec, TaskDependency
)

w = WorkspaceClient()

# Create multi-task job
job = w.jobs.create(
    name="ETL Pipeline - Customer Analytics",
    tasks=[
        # Task 1: Data Ingestion
        Task(
            task_key="ingest_data",
            notebook_task=NotebookTask(
                notebook_path="/Pipelines/01_ingest_raw_data",
                base_parameters={
                    "source_table": "raw.customer_events",
                    "target_table": "bronze.customer_events"
                }
            ),
            job_cluster_key="etl_cluster",
            timeout_seconds=3600,
            max_retries=2,
            min_retry_interval_millis=60000  # 1 minute
        ),
        
        # Task 2: Data Cleaning (depends on Task 1)
        Task(
            task_key="clean_data",
            notebook_task=NotebookTask(
                notebook_path="/Pipelines/02_clean_data",
                base_parameters={
                    "input_table": "bronze.customer_events",
                    "output_table": "silver.customer_events"
                }
            ),
            depends_on=[TaskDependency(task_key="ingest_data")],
            job_cluster_key="etl_cluster",
            timeout_seconds=7200
        ),
        
        # Task 3: Feature Engineering (depends on Task 2)
        Task(
            task_key="engineer_features",
            notebook_task=NotebookTask(
                notebook_path="/Pipelines/03_feature_engineering",
                base_parameters={
                    "input_table": "silver.customer_events",
                    "output_table": "gold.customer_features"
                }
            ),
            depends_on=[TaskDependency(task_key="clean_data")],
            job_cluster_key="etl_cluster"
        ),
        
        # Task 4: Aggregations (parallel with Task 3)
        Task(
            task_key="aggregate_metrics",
            sql_task=SqlTask(
                query=SqlTaskQuery(
                    query_id="abc123"  # Reference to saved SQL query
                )
            ),
            depends_on=[TaskDependency(task_key="clean_data")],
            sql_warehouse_id="warehouse123"
        ),
        
        # Task 5: Final Report (depends on Tasks 3 and 4)
        Task(
            task_key="generate_report",
            notebook_task=NotebookTask(
                notebook_path="/Pipelines/05_generate_report"
            ),
            depends_on=[
                TaskDependency(task_key="engineer_features"),
                TaskDependency(task_key="aggregate_metrics")
            ],
            job_cluster_key="etl_cluster"
        )
    ],
    
    # Define cluster configuration
    job_clusters=[
        JobCluster(
            job_cluster_key="etl_cluster",
            new_cluster=ClusterSpec(
                spark_version="13.3.x-scala2.12",
                node_type_id="i3.xlarge",
                num_workers=4,
                autotermination_minutes=15,
                spark_conf={
                    "spark.sql.adaptive.enabled": "true",
                    "spark.databricks.delta.optimizeWrite.enabled": "true"
                }
            )
        )
    ],
    
    # Email notifications
    email_notifications={
        "on_start": ["team@company.com"],
        "on_success": ["team@company.com"],
        "on_failure": ["team@company.com", "oncall@company.com"],
        "no_alert_for_skipped_runs": False
    },
    
    # Webhook notifications
    webhook_notifications={
        "on_failure": [
            {
                "id": "webhook123"  # Configured webhook ID
            }
        ]
    },
    
    # Job settings
    max_concurrent_runs=1,
    timeout_seconds=28800,  # 8 hours
    
    tags={
        "team": "data-engineering",
        "project": "customer-analytics",
        "environment": "production"
    }
)

print(f"Created job: {job.job_id}")
print(f"Job URL: {job.settings.url}")
```

### Example 2: Dynamic Parameter Passing Between Tasks

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import Task, NotebookTask, TaskDependency

w = WorkspaceClient()

# Task 1 outputs a value that Task 2 consumes
task1_notebook = """
# Task 1: Calculate date range
from datetime import datetime, timedelta

# Calculate processing window
end_date = datetime.now()
start_date = end_date - timedelta(days=7)

# Set output values for downstream tasks
dbutils.jobs.taskValues.set(key="start_date", value=start_date.strftime("%Y-%m-%d"))
dbutils.jobs.taskValues.set(key="end_date", value=end_date.strftime("%Y-%m-%d"))
dbutils.jobs.taskValues.set(key="record_count", value=10000)

print(f"Processing window: {start_date} to {end_date}")
"""

# Task 2 reads values from Task 1
task2_notebook = """
# Task 2: Process data using values from Task 1

# Get values from upstream task
start_date = dbutils.jobs.taskValues.get(
    taskKey="calculate_dates",
    key="start_date",
    default="2024-01-01"
)

end_date = dbutils.jobs.taskValues.get(
    taskKey="calculate_dates",
    key="end_date",
    default="2024-01-07"
)

record_count = dbutils.jobs.taskValues.get(
    taskKey="calculate_dates",
    key="record_count",
    default=0,
    debugValue=0  # Used in interactive runs
)

print(f"Processing {record_count} records from {start_date} to {end_date}")

# Process data
df = spark.sql(f'''
    SELECT * FROM events
    WHERE event_date BETWEEN '{start_date}' AND '{end_date}'
''')

# Continue processing...
"""

# Create job with dynamic parameters
job = w.jobs.create(
    name="Dynamic Parameter Pipeline",
    tasks=[
        Task(
            task_key="calculate_dates",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/calculate_dates"
            ),
            job_cluster_key="shared_cluster"
        ),
        Task(
            task_key="process_data",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/process_data"
            ),
            depends_on=[TaskDependency(task_key="calculate_dates")],
            job_cluster_key="shared_cluster"
        )
    ],
    job_clusters=[
        JobCluster(
            job_cluster_key="shared_cluster",
            new_cluster=ClusterSpec(
                spark_version="13.3.x-scala2.12",
                node_type_id="i3.xlarge",
                num_workers=2
            )
        )
    ]
)
```

### Example 3: Conditional Task Execution

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import Task, NotebookTask, TaskDependency, Condition

w = WorkspaceClient()

# Validation task that sets success/failure
validation_notebook = """
# Validation Task
df = spark.table("bronze.daily_data")

# Perform validation
record_count = df.count()
null_count = df.filter(col("customer_id").isNull()).count()

# Determine if validation passed
validation_passed = record_count > 1000 and null_count < 100

# Set task value for conditional logic
dbutils.jobs.taskValues.set(key="validation_passed", value=validation_passed)
dbutils.jobs.taskValues.set(key="record_count", value=record_count)

if not validation_passed:
    raise Exception(f"Validation failed: {record_count} records, {null_count} nulls")

print("✅ Validation passed")
"""

# Success processing task
success_notebook = """
print("Running success path - data quality is good")
# Process data normally
"""

# Failure handling task
failure_notebook = """
print("Running failure path - data quality issues detected")
# Send alerts, skip processing, or use backup data
"""

job = w.jobs.create(
    name="Conditional Workflow",
    tasks=[
        # Validation task
        Task(
            task_key="validate_data",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/validate"
            ),
            job_cluster_key="cluster"
        ),
        
        # Success path - runs if validation passes
        Task(
            task_key="process_good_data",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/process_success"
            ),
            depends_on=[
                TaskDependency(
                    task_key="validate_data",
                    outcome="success"  # Only run if validation succeeds
                )
            ],
            job_cluster_key="cluster"
        ),
        
        # Failure path - runs if validation fails
        Task(
            task_key="handle_bad_data",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/handle_failure"
            ),
            depends_on=[
                TaskDependency(
                    task_key="validate_data",
                    outcome="failure"  # Only run if validation fails
                )
            ],
            job_cluster_key="cluster"
        ),
        
        # Cleanup task - runs regardless of validation outcome
        Task(
            task_key="cleanup",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/cleanup"
            ),
            depends_on=[
                TaskDependency(task_key="process_good_data"),
                TaskDependency(task_key="handle_bad_data")
            ],
            job_cluster_key="cluster"
        )
    ],
    job_clusters=[
        JobCluster(
            job_cluster_key="cluster",
            new_cluster=ClusterSpec(
                spark_version="13.3.x-scala2.12",
                node_type_id="i3.xlarge",
                num_workers=2
            )
        )
    ]
)
```

### Example 4: Complex Fan-Out/Fan-In Pattern

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import Task, NotebookTask, TaskDependency

w = WorkspaceClient()

job = w.jobs.create(
    name="Fan-Out Fan-In Pipeline",
    tasks=[
        # Initial data preparation
        Task(
            task_key="prepare_data",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/00_prepare"
            ),
            job_cluster_key="cluster"
        ),
        
        # Fan-out: Process multiple regions in parallel
        Task(
            task_key="process_north_america",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/process_region",
                base_parameters={"region": "north_america"}
            ),
            depends_on=[TaskDependency(task_key="prepare_data")],
            job_cluster_key="cluster"
        ),
        
        Task(
            task_key="process_europe",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/process_region",
                base_parameters={"region": "europe"}
            ),
            depends_on=[TaskDependency(task_key="prepare_data")],
            job_cluster_key="cluster"
        ),
        
        Task(
            task_key="process_asia",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/process_region",
                base_parameters={"region": "asia"}
            ),
            depends_on=[TaskDependency(task_key="prepare_data")],
            job_cluster_key="cluster"
        ),
        
        # Fan-in: Combine results from all regions
        Task(
            task_key="combine_results",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/combine_regions"
            ),
            depends_on=[
                TaskDependency(task_key="process_north_america"),
                TaskDependency(task_key="process_europe"),
                TaskDependency(task_key="process_asia")
            ],
            job_cluster_key="cluster"
        ),
        
        # Final aggregation
        Task(
            task_key="final_aggregation",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/aggregate_global"
            ),
            depends_on=[TaskDependency(task_key="combine_results")],
            job_cluster_key="cluster"
        )
    ],
    job_clusters=[
        JobCluster(
            job_cluster_key="cluster",
            new_cluster=ClusterSpec(
                spark_version="13.3.x-scala2.12",
                node_type_id="i3.xlarge",
                num_workers=4
            )
        )
    ],
    max_concurrent_runs=1
)
```

### Example 5: Mixing Task Types (Notebook, SQL, DLT, Python)

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import (
    Task, NotebookTask, SqlTask, SqlTaskQuery, SqlTaskDashboard,
    PipelineTask, PythonWheelTask, TaskDependency
)

w = WorkspaceClient()

job = w.jobs.create(
    name="Multi-Type Task Pipeline",
    tasks=[
        # Task 1: DLT Pipeline for CDC ingestion
        Task(
            task_key="run_dlt_pipeline",
            pipeline_task=PipelineTask(
                pipeline_id="abc123-pipeline-id",
                full_refresh=False
            ),
            timeout_seconds=7200
        ),
        
        # Task 2: Notebook for feature engineering
        Task(
            task_key="engineer_features",
            notebook_task=NotebookTask(
                notebook_path="/Pipelines/feature_engineering"
            ),
            depends_on=[TaskDependency(task_key="run_dlt_pipeline")],
            job_cluster_key="ml_cluster"
        ),
        
        # Task 3: Python wheel for ML training
        Task(
            task_key="train_model",
            python_wheel_task=PythonWheelTask(
                package_name="ml_training",
                entry_point="train",
                parameters=["--config", "/configs/model_config.yaml"]
            ),
            depends_on=[TaskDependency(task_key="engineer_features")],
            job_cluster_key="ml_cluster"
        ),
        
        # Task 4: SQL query for aggregations
        Task(
            task_key="run_sql_aggregations",
            sql_task=SqlTask(
                query=SqlTaskQuery(
                    query_id="sql-query-123"
                )
            ),
            depends_on=[TaskDependency(task_key="engineer_features")],
            sql_warehouse_id="warehouse-id-456"
        ),
        
        # Task 5: Refresh SQL dashboard
        Task(
            task_key="refresh_dashboard",
            sql_task=SqlTask(
                dashboard=SqlTaskDashboard(
                    dashboard_id="dashboard-789"
                )
            ),
            depends_on=[TaskDependency(task_key="run_sql_aggregations")],
            sql_warehouse_id="warehouse-id-456"
        ),
        
        # Task 6: Notebook for reporting
        Task(
            task_key="send_report",
            notebook_task=NotebookTask(
                notebook_path="/Reporting/send_daily_report"
            ),
            depends_on=[
                TaskDependency(task_key="train_model"),
                TaskDependency(task_key="refresh_dashboard")
            ],
            job_cluster_key="reporting_cluster"
        )
    ],
    
    job_clusters=[
        JobCluster(
            job_cluster_key="ml_cluster",
            new_cluster=ClusterSpec(
                spark_version="13.3.x-ml-scala2.12",
                node_type_id="g4dn.xlarge",  # GPU instance
                num_workers=2
            )
        ),
        JobCluster(
            job_cluster_key="reporting_cluster",
            new_cluster=ClusterSpec(
                spark_version="13.3.x-scala2.12",
                node_type_id="i3.xlarge",
                num_workers=1
            )
        )
    ]
)
```

### Example 6: Error Handling and Retry Logic

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import Task, NotebookTask, TaskDependency

# Notebook with built-in error handling
error_handling_notebook = """
import time
from datetime import datetime

# Task configuration
MAX_RETRIES = 3
RETRY_DELAY = 60  # seconds

def process_with_retry():
    for attempt in range(MAX_RETRIES):
        try:
            # Attempt processing
            df = spark.table("source_table")
            
            # Validate data
            if df.count() == 0:
                raise ValueError("No data found in source table")
            
            # Process
            result_df = df.transform(clean_data).transform(aggregate_metrics)
            
            # Write results
            result_df.write.mode("overwrite").saveAsTable("target_table")
            
            print(f"✅ Processing successful on attempt {attempt + 1}")
            return True
            
        except Exception as e:
            print(f"❌ Attempt {attempt + 1} failed: {str(e)}")
            
            if attempt < MAX_RETRIES - 1:
                print(f"Retrying in {RETRY_DELAY} seconds...")
                time.sleep(RETRY_DELAY)
            else:
                # Final failure - log and alert
                error_msg = f"Processing failed after {MAX_RETRIES} attempts: {str(e)}"
                
                # Log to error table
                error_df = spark.createDataFrame([
                    (datetime.now(), "process_task", error_msg, str(e))
                ], ["timestamp", "task", "message", "details"])
                
                error_df.write.mode("append").saveAsTable("monitoring.job_errors")
                
                # Set failure status
                dbutils.jobs.taskValues.set(key="status", value="failed")
                dbutils.jobs.taskValues.set(key="error", value=error_msg)
                
                raise

# Run with retry logic
process_with_retry()
"""

w = WorkspaceClient()

job = w.jobs.create(
    name="Pipeline with Error Handling",
    tasks=[
        Task(
            task_key="critical_processing",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/process_with_retry"
            ),
            job_cluster_key="cluster",
            
            # Job-level retry configuration
            max_retries=3,
            min_retry_interval_millis=60000,  # 1 minute between retries
            retry_on_timeout=True,
            
            # Timeout
            timeout_seconds=3600
        ),
        
        # Error notification task (runs on failure)
        Task(
            task_key="notify_on_error",
            notebook_task=NotebookTask(
                notebook_path="/Workflows/send_error_notification"
            ),
            depends_on=[
                TaskDependency(
                    task_key="critical_processing",
                    outcome="failure"
                )
            ],
            job_cluster_key="cluster"
        )
    ],
    
    job_clusters=[
        JobCluster(
            job_cluster_key="cluster",
            new_cluster=ClusterSpec(
                spark_version="13.3.x-scala2.12",
                node_type_id="i3.xlarge",
                num_workers=2
            )
        )
    ],
    
    # Job-level notifications
    email_notifications={
        "on_failure": ["oncall@company.com", "team-lead@company.com"],
        "alert_on_last_attempt": True
    }
)
```

---

## Best Practices

### 1. Job Design
* Keep tasks focused and single-purpose
* Use clear, descriptive task names
* Limit job complexity (< 20 tasks)
* Group related workflows in separate jobs

### 2. Dependencies
* Minimize task dependencies for parallelism
* Use fan-out/fan-in for independent processing
* Avoid circular dependencies
* Test dependency logic thoroughly

### 3. Parameters
* Use job parameters for configuration
* Pass dynamic values via taskValues
* Document parameter requirements
* Validate parameters at task start

### 4. Error Handling
* Set appropriate retry policies
* Implement timeout values
* Create error notification tasks
* Log errors to monitoring tables

### 5. Performance
* Reuse clusters with job_cluster_key
* Right-size cluster configurations
* Enable autoscaling where appropriate
* Monitor task duration and optimize

### 6. Monitoring
* Set up email and webhook notifications
* Log task outputs and metrics
* Track job run history
* Create dashboards for job monitoring

---

## Related Skills

* [Job Scheduling](../job-scheduling/SKILL.md) - For scheduling workflows
* [Error Handling](../error-handling/SKILL.md) - For comprehensive error strategies
* [Monitoring Observability](../monitoring-observability/SKILL.md) - For production monitoring

---

**Last Updated**: 2024-04-09
