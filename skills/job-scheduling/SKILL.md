# Job Scheduling Skill

## Purpose
This skill teaches Genie Code how to schedule Databricks jobs using cron expressions, triggers, continuous execution, file arrival triggers, and conditional scheduling patterns for automated data pipelines.

## When to Use
* Scheduling recurring data pipelines
* Setting up time-based job execution
* Implementing file-based triggers
* Creating continuous streaming jobs
* Coordinating dependent job schedules
* Managing timezone-aware schedules
* Implementing business day logic

## Key Concepts

### 1. Schedule Types
* **Cron-based**: Time-based schedules (daily, weekly, monthly)
* **File arrival**: Trigger on new files in storage
* **Continuous**: Run continuously for streaming
* **Manual**: On-demand execution only

### 2. Cron Expressions
* **Quartz format**: 7 fields (second, minute, hour, day, month, day-of-week, year)
* **Common patterns**: Daily, weekly, monthly, custom
* **Timezone**: Specify execution timezone

### 3. Triggers
* **File arrival**: Monitor cloud storage paths
* **API triggers**: External system triggers
* **Job completion**: Downstream job triggers

### 4. Execution Control
* **Max concurrent runs**: Limit parallel executions
* **Queue behavior**: Skip, queue, or replace runs
* **Pause/resume**: Temporarily disable schedules

---

## Examples

### Example 1: Common Cron Schedules

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import Task, NotebookTask, JobCluster, ClusterSpec, CronSchedule

w = WorkspaceClient()

# Daily at 2 AM Pacific Time
daily_job = w.jobs.create(
    name="Daily ETL Pipeline",
    tasks=[
        Task(
            task_key="daily_processing",
            notebook_task=NotebookTask(
                notebook_path="/Pipelines/daily_etl"
            ),
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
    schedule=CronSchedule(
        quartz_cron_expression="0 0 2 * * ? *",  # sec min hour day month dow year
        timezone_id="America/Los_Angeles",
        pause_status="UNPAUSED"
    )
)

# Every weekday at 8 AM UTC
weekday_job = w.jobs.create(
    name="Weekday Report Generation",
    tasks=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 0 8 ? * MON-FRI *",
        timezone_id="UTC"
    )
)

# First day of month at 3 AM
monthly_job = w.jobs.create(
    name="Monthly Aggregation",
    tasks=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 0 3 1 * ? *",  # Day 1 of every month
        timezone_id="America/New_York"
    )
)

# Every 6 hours
sixhourly_job = w.jobs.create(
    name="6-Hourly Data Refresh",
    tasks=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 0 */6 * * ? *",  # Every 6 hours
        timezone_id="UTC"
    )
)

# Every 15 minutes
frequent_job = w.jobs.create(
    name="Near Real-Time Processing",
    tasks=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 */15 * * * ? *",  # Every 15 minutes
        timezone_id="UTC"
    )
)

# Sundays at 1 AM (weekly maintenance)
weekly_job = w.jobs.create(
    name="Weekly Data Cleanup",
    tasks=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 0 1 ? * SUN *",  # Every Sunday
        timezone_id="America/Chicago"
    )
)

# Common cron patterns:
# "0 0 0 * * ? *"         - Daily at midnight
# "0 30 8 * * ? *"        - Daily at 8:30 AM
# "0 0 */4 * * ? *"       - Every 4 hours
# "0 0 9 ? * MON *"       - Every Monday at 9 AM
# "0 0 0 1,15 * ? *"      - 1st and 15th of month
# "0 0 2 L * ? *"         - Last day of month at 2 AM
```

### Example 2: File Arrival Triggers

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import Task, NotebookTask, FileArrivalTriggerConfiguration

w = WorkspaceClient()

# Trigger when files arrive in S3 path
file_trigger_job = w.jobs.create(
    name="File Arrival Processing",
    tasks=[
        Task(
            task_key="process_new_files",
            notebook_task=NotebookTask(
                notebook_path="/Pipelines/process_files",
                base_parameters={
                    "input_path": "{{trigger.file_path}}"  # Dynamic path from trigger
                }
            ),
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
    trigger=FileArrivalTriggerConfiguration(
        url="s3://my-bucket/incoming-data/",
        min_time_between_triggers_seconds=300,  # Wait 5 min between triggers
        wait_after_last_change_seconds=60  # Wait 60s after last file change
    )
)

# Notebook to process triggered files
processing_notebook = """
# Get file path from trigger
file_path = dbutils.widgets.get("input_path")

print(f"Processing files from: {file_path}")

# Read new files
df = spark.read.format("parquet").load(file_path)

# Process data
processed_df = df.transform(clean_data).transform(aggregate)

# Write results
processed_df.write.mode("append").saveAsTable("processed_data")

# Archive processed files
dbutils.fs.mv(file_path, f"s3://my-bucket/archive/{datetime.now().strftime('%Y-%m-%d')}/")

print("✅ Processing complete")
"""
```

### Example 3: Continuous Jobs for Streaming

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import Task, NotebookTask, ContinuousSchedule

w = WorkspaceClient()

# Continuous streaming job
streaming_job = w.jobs.create(
    name="Continuous Streaming Pipeline",
    tasks=[
        Task(
            task_key="stream_processing",
            notebook_task=NotebookTask(
                notebook_path="/Streaming/process_kafka_stream"
            ),
            job_cluster_key="streaming_cluster"
        )
    ],
    job_clusters=[
        JobCluster(
            job_cluster_key="streaming_cluster",
            new_cluster=ClusterSpec(
                spark_version="13.3.x-scala2.12",
                node_type_id="i3.xlarge",
                num_workers=4,
                autoscale=AutoScale(min_workers=2, max_workers=8)
            )
        )
    ],
    schedule=ContinuousSchedule(
        pause_status="UNPAUSED"
    ),
    max_concurrent_runs=1  # Only one instance at a time
)

# Streaming notebook
streaming_notebook = """
# Continuous streaming processing
from pyspark.sql.functions import col, window, count

# Read from Kafka
df = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "events")
    .load()
)

# Process stream
processed = (
    df
    .selectExpr("CAST(value AS STRING) as json")
    .select(from_json(col("json"), schema).alias("data"))
    .select("data.*")
    .withWatermark("timestamp", "10 minutes")
    .groupBy(
        window("timestamp", "5 minutes"),
        "event_type"
    )
    .agg(count("*").alias("count"))
)

# Write to Delta
query = (
    processed.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/checkpoints/events")
    .table("streaming_events")
)

# Keep stream running
query.awaitTermination()
"""
```

### Example 4: Conditional Scheduling Logic

```python
from databricks.sdk import WorkspaceClient

# Notebook with business day logic
business_day_notebook = """
from datetime import datetime
import pandas as pd
from pandas.tseries.holiday import USFederalHolidayCalendar

# Check if today is a business day
today = datetime.now().date()
cal = USFederalHolidayCalendar()
holidays = cal.holidays(start=today, end=today)

is_weekend = today.weekday() >= 5  # Saturday=5, Sunday=6
is_holiday = today in holidays.date

if is_weekend or is_holiday:
    print(f"⏭️  Skipping execution - not a business day")
    print(f"Today: {today}, Weekend: {is_weekend}, Holiday: {is_holiday}")
    dbutils.notebook.exit("SKIPPED")
else:
    print(f"✅ Running - business day: {today}")
    
    # Execute main processing
    df = spark.table("source_data")
    # ... process ...
    
    dbutils.notebook.exit("SUCCESS")
"""

w = WorkspaceClient()

# Job runs daily but skips non-business days
business_day_job = w.jobs.create(
    name="Business Days Only Pipeline",
    tasks=[
        Task(
            task_key="check_business_day",
            notebook_task=NotebookTask(
                notebook_path="/Scheduling/check_business_day"
            ),
            job_cluster_key="cluster"
        ),
        Task(
            task_key="run_processing",
            notebook_task=NotebookTask(
                notebook_path="/Pipelines/main_processing"
            ),
            depends_on=[TaskDependency(task_key="check_business_day")],
            job_cluster_key="cluster"
        )
    ],
    job_clusters=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 0 8 * * ? *",  # Daily at 8 AM
        timezone_id="America/New_York"
    )
)
```

### Example 5: Advanced Scheduling Patterns

```python
from databricks.sdk import WorkspaceClient

# Pattern 1: Staggered schedules for resource management
w = WorkspaceClient()

# High priority: Top of hour
high_priority = w.jobs.create(
    name="High Priority ETL",
    tasks=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 0 * * * ? *",  # Every hour at :00
        timezone_id="UTC"
    )
)

# Medium priority: 20 minutes past hour
medium_priority = w.jobs.create(
    name="Medium Priority ETL",
    tasks=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 20 * * * ? *",  # Every hour at :20
        timezone_id="UTC"
    )
)

# Low priority: 40 minutes past hour
low_priority = w.jobs.create(
    name="Low Priority ETL",
    tasks=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 40 * * * ? *",  # Every hour at :40
        timezone_id="UTC"
    )
)

# Pattern 2: Time-zone aware scheduling
# Process data for each region at local 2 AM
regions = [
    ("Americas ETL", "America/New_York"),
    ("Europe ETL", "Europe/London"),
    ("APAC ETL", "Asia/Singapore")
]

for name, timezone in regions:
    w.jobs.create(
        name=name,
        tasks=[...],
        schedule=CronSchedule(
            quartz_cron_expression="0 0 2 * * ? *",
            timezone_id=timezone  # Each runs at local 2 AM
        )
    )

# Pattern 3: Dependent job scheduling
# Upstream job
upstream = w.jobs.create(
    name="Upstream Data Preparation",
    tasks=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 0 1 * * ? *",  # 1 AM
        timezone_id="UTC"
    )
)

# Downstream job (waits for upstream via API)
downstream_notebook = """
from databricks.sdk import WorkspaceClient
import time

w = WorkspaceClient()

# Wait for upstream job to complete
upstream_job_id = 12345
max_wait_seconds = 7200  # 2 hours

print(f"Waiting for upstream job {upstream_job_id}...")

start_time = time.time()
while time.time() - start_time < max_wait_seconds:
    runs = w.jobs.list_runs(job_id=upstream_job_id, limit=1)
    
    if runs and runs[0].state.life_cycle_state == "TERMINATED":
        if runs[0].state.result_state == "SUCCESS":
            print("✅ Upstream job successful, proceeding...")
            break
        else:
            raise Exception("Upstream job failed")
    
    time.sleep(60)  # Check every minute
else:
    raise Exception("Timeout waiting for upstream job")

# Proceed with downstream processing
df = spark.table("upstream_output")
# ... process ...
"""

downstream = w.jobs.create(
    name="Downstream Processing",
    tasks=[
        Task(
            task_key="wait_and_process",
            notebook_task=NotebookTask(
                notebook_path="/Scheduling/wait_for_upstream"
            ),
            job_cluster_key="cluster",
            timeout_seconds=10800  # 3 hours
        )
    ],
    job_clusters=[...],
    schedule=CronSchedule(
        quartz_cron_expression="0 30 1 * * ? *",  # 1:30 AM (30 min after upstream)
        timezone_id="UTC"
    )
)
```

### Example 6: Schedule Management and Monitoring

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import PauseStatus
import pandas as pd

w = WorkspaceClient()

class JobScheduleManager:
    """Manage job schedules programmatically"""
    
    def __init__(self):
        self.w = WorkspaceClient()
    
    def pause_job(self, job_id):
        """Pause a scheduled job"""
        job = self.w.jobs.get(job_id)
        
        if job.settings.schedule:
            self.w.jobs.update(
                job_id=job_id,
                new_settings={
                    **job.settings,
                    "schedule": {
                        **job.settings.schedule,
                        "pause_status": PauseStatus.PAUSED
                    }
                }
            )
            print(f"✅ Paused job {job_id}: {job.settings.name}")
    
    def resume_job(self, job_id):
        """Resume a paused job"""
        self.w.jobs.update(
            job_id=job_id,
            new_settings={
                "schedule": {
                    "pause_status": PauseStatus.UNPAUSED
                }
            }
        )
        print(f"✅ Resumed job {job_id}")
    
    def get_schedule_summary(self):
        """Get summary of all scheduled jobs"""
        jobs = self.w.jobs.list()
        
        schedule_data = []
        for job in jobs:
            if job.settings.schedule:
                schedule_data.append({
                    'job_id': job.job_id,
                    'name': job.settings.name,
                    'schedule': job.settings.schedule.quartz_cron_expression,
                    'timezone': job.settings.schedule.timezone_id,
                    'status': job.settings.schedule.pause_status,
                    'max_concurrent': job.settings.max_concurrent_runs
                })
        
        return pd.DataFrame(schedule_data)
    
    def monitor_schedule_adherence(self, days=7):
        """Monitor if jobs are running on schedule"""
        from datetime import datetime, timedelta
        
        query = f"""
        SELECT 
            job_id,
            job_name,
            DATE(start_time) as run_date,
            COUNT(*) as executions,
            AVG(execution_duration) / 1000 / 60 as avg_duration_minutes,
            SUM(CASE WHEN state = 'SUCCESS' THEN 1 ELSE 0 END) as successful_runs,
            SUM(CASE WHEN state = 'FAILED' THEN 1 ELSE 0 END) as failed_runs
        FROM system.workflows.job_runs
        WHERE start_time >= CURRENT_DATE() - INTERVAL {days} DAYS
        GROUP BY job_id, job_name, DATE(start_time)
        ORDER BY run_date DESC, job_name
        """
        
        return spark.sql(query).toPandas()

# Usage
manager = JobScheduleManager()

# Get all schedules
schedules = manager.get_schedule_summary()
print(schedules)

# Pause/resume jobs
manager.pause_job(job_id=123)
manager.resume_job(job_id=123)

# Monitor adherence
adherence = manager.monitor_schedule_adherence(days=30)
print(adherence)
```

---

## Best Practices

### 1. Cron Expression Design
* Use standard patterns when possible
* Document complex cron expressions
* Test with online cron validators
* Consider timezone implications

### 2. Execution Control
* Set max_concurrent_runs=1 for sequential pipelines
* Use queue mode for critical jobs
* Set appropriate timeouts
* Monitor execution patterns

### 3. Resource Management
* Stagger schedules to avoid resource contention
* Use autoscaling for variable workloads
* Schedule heavy jobs during off-peak hours
* Monitor cluster utilization

### 4. Reliability
* Implement retry logic
* Set up failure notifications
* Test schedules before production
* Document schedule dependencies

### 5. Monitoring
* Track schedule adherence
* Monitor execution durations
* Alert on missed schedules
* Review and optimize regularly

---

## Cron Expression Reference

```
Format: second minute hour day month day-of-week year

Fields:
- second: 0-59
- minute: 0-59
- hour: 0-23
- day: 1-31
- month: 1-12 or JAN-DEC
- day-of-week: 1-7 or SUN-SAT (1=Sunday)
- year: 1970-2099

Special characters:
* = all values
? = no specific value (day or day-of-week)
- = range (MON-FRI)
, = list (1,15)
/ = increment (*/15)
L = last (day of month)
```

---

## Related Skills

* [Workflow Orchestration](../workflow-orchestration/SKILL.md) - For task orchestration
* [Error Handling](../error-handling/SKILL.md) - For failure handling
* [Monitoring Observability](../monitoring-observability/SKILL.md) - For schedule monitoring

---

**Last Updated**: 2024-04-09
