# Cost Optimization Skill

## Purpose
This skill teaches Genie Code how to optimize Databricks costs through intelligent cluster sizing, autoscaling, spot instances, Photon engine, cost monitoring, and resource tagging. It covers strategies to reduce compute costs while maintaining performance and reliability.

## When to Use
* Planning cluster configurations for cost efficiency
* Implementing autoscaling for variable workloads
* Using spot/preemptible instances for fault-tolerant jobs
* Enabling Photon engine for compatible workloads
* Monitoring and analyzing cost trends
* Implementing cost allocation with tags
* Setting up budget alerts and governance
* Optimizing storage costs with lifecycle policies

## Key Concepts

### 1. Cluster Sizing
* **Right-sizing**: Match cluster size to workload requirements
* **Node types**: Balance memory, CPU, and cost
* **Driver vs. workers**: Optimize for bottlenecks
* **Pools**: Reduce startup time, maintain warm instances

### 2. Autoscaling
* **Min/max workers**: Define scaling boundaries
* **Scale-up delay**: Prevent premature scaling
* **Scale-down delay**: Balance cost vs. stability
* **Job clusters**: Always prefer over all-purpose clusters

### 3. Spot/Preemptible Instances
* **Cost savings**: 60-90% cheaper than on-demand
* **Fault tolerance**: Suitable for retryable workloads
* **Spot policies**: Fall back to on-demand when needed
* **Best for**: Batch processing, ETL, non-critical jobs

### 4. Photon Engine
* **Performance**: 2-3x faster for SQL/DataFrame operations
* **Cost efficiency**: Faster execution = less runtime cost
* **Compatibility**: Works with most SQL queries
* **Free tier**: Included in premium SKU

### 5. Cost Monitoring
* **Account console**: View usage and costs
* **System tables**: Analyze cluster usage programmatically
* **Tags**: Track costs by project, team, environment
* **Budgets**: Set alerts for spending thresholds


### 7. Serverless Compute Economics (2026 Recommended)
* **Instant-on compute** - no startup wait time
* **Pay-per-use pricing** - billed per second
* **No cluster management** - zero overhead
* **Auto-scaling included** - optimal resource utilization
* **Photon engine included** - 3-5x faster queries
* **Best for**: SQL warehouses, notebooks, scheduled jobs, DLT pipelines
* **Cost savings**: 20-40% vs classic clusters for intermittent workloads

---

## Examples

### Example 1: Optimal Job Cluster Configuration

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import Task, NotebookTask, JobCluster, ClusterSpec
from databricks.sdk.service.compute import AutoScale, AwsAttributes, AwsAvailability

w = WorkspaceClient()

# Create cost-optimized job cluster
job_cluster = JobCluster(
    job_cluster_key="cost_optimized_cluster",
    new_cluster=ClusterSpec(
        spark_version="13.3.x-scala2.12",  # Latest LTS
        node_type_id="i3.xlarge",  # Optimize for workload
        
        # Enable autoscaling
        autoscale=AutoScale(
            min_workers=2,
            max_workers=10
        ),
        
        # Use spot instances for cost savings
        aws_attributes=AwsAttributes(
            availability=AwsAvailability.SPOT_WITH_FALLBACK,
            zone_id="us-west-2a",
            first_on_demand=1,  # Keep driver on-demand
            spot_bid_price_percent=100  # Max bid price
        ),
        
        # Enable Photon for performance
        runtime_engine="PHOTON",
        
        # Enable autotermination
        autotermination_minutes=10,
        
        # Cost allocation tags
        custom_tags={
            "Project": "DataPipeline",
            "Team": "DataEngineering",
            "Environment": "Production",
            "CostCenter": "Engineering-001"
        },
        
        # Optimize Spark configs for cost
        spark_conf={
            "spark.databricks.delta.optimizeWrite.enabled": "true",
            "spark.databricks.delta.autoCompact.enabled": "true",
            "spark.sql.adaptive.enabled": "true",
            "spark.sql.adaptive.coalescePartitions.enabled": "true"
        }
    )
)

# Create job with optimized cluster
job = w.jobs.create(
    name="cost_optimized_etl_job",
    job_clusters=[job_cluster],
    tasks=[
        Task(
            task_key="process_data",
            job_cluster_key="cost_optimized_cluster",
            notebook_task=NotebookTask(
                notebook_path="/path/to/notebook",
                base_parameters={}
            ),
            timeout_seconds=3600
        )
    ],
    max_concurrent_runs=1
)

print(f"Created cost-optimized job: {job.job_id}")
```

### Example 2: Cost Monitoring with System Tables

```sql
-- Analyze cluster usage and costs by team/project
SELECT 
    usage_metadata.cluster_id,
    usage_metadata.custom_tags['Team'] as team,
    usage_metadata.custom_tags['Project'] as project,
    usage_metadata.custom_tags['Environment'] as environment,
    DATE(usage_start_time) as usage_date,
    
    -- DBU consumption
    SUM(usage_quantity) as total_dbus,
    
    -- Runtime hours
    SUM(TIMESTAMPDIFF(HOUR, usage_start_time, usage_end_time)) as runtime_hours,
    
    -- Estimate cost (example: $0.40 per DBU)
    SUM(usage_quantity) * 0.40 as estimated_cost_usd
    
FROM system.billing.usage
WHERE 
    usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
    AND sku_name LIKE '%Jobs%'  -- Focus on job compute
GROUP BY 1, 2, 3, 4, 5
ORDER BY estimated_cost_usd DESC
LIMIT 100;

-- Identify idle clusters (running but not processing)
SELECT 
    cluster_id,
    cluster_name,
    state,
    start_time,
    TIMESTAMPDIFF(HOUR, start_time, CURRENT_TIMESTAMP()) as hours_running,
    
    -- Calculate wasted cost for idle time
    TIMESTAMPDIFF(HOUR, start_time, CURRENT_TIMESTAMP()) * 
    (SELECT AVG(usage_quantity) FROM system.billing.usage WHERE cluster_id = c.cluster_id) * 
    0.40 as wasted_cost_usd
    
FROM system.compute.clusters c
WHERE 
    state = 'RUNNING'
    AND cluster_source = 'UI'  -- Interactive clusters
    AND TIMESTAMPDIFF(HOUR, start_time, CURRENT_TIMESTAMP()) > 2
ORDER BY hours_running DESC;
```

### Example 3: Automated Cost Optimization Script

```python
from databricks.sdk import WorkspaceClient
from datetime import datetime, timedelta
import pandas as pd

class CostOptimizer:
    """
    Automated cost optimization for Databricks workspace
    """
    
    def __init__(self):
        self.w = WorkspaceClient()
    
    def terminate_idle_clusters(self, idle_threshold_hours: int = 2):
        """Terminate clusters idle beyond threshold"""
        clusters = self.w.clusters.list()
        terminated_count = 0
        
        for cluster in clusters:
            if cluster.state.value == 'RUNNING':
                # Get last activity
                events = list(self.w.clusters.events(
                    cluster_id=cluster.cluster_id,
                    limit=1,
                    order="DESC"
                ))
                
                if events:
                    last_activity = events[0].timestamp
                    idle_hours = (datetime.now() - last_activity).total_seconds() / 3600
                    
                    if idle_hours > idle_threshold_hours:
                        print(f"Terminating idle cluster: {cluster.cluster_name} "
                              f"(idle for {idle_hours:.1f} hours)")
                        self.w.clusters.delete(cluster_id=cluster.cluster_id)
                        terminated_count += 1
        
        return terminated_count
    
    def optimize_cluster_size(self, cluster_id: str, utilization_threshold: float = 0.3):
        """
        Recommend cluster size optimization based on utilization
        """
        # Get cluster metrics from system tables
        metrics_query = f"""
        SELECT 
            AVG(cpu_utilization) as avg_cpu,
            AVG(memory_utilization) as avg_memory,
            MAX(num_workers) as max_workers,
            AVG(num_workers) as avg_workers
        FROM system.compute.cluster_metrics
        WHERE cluster_id = '{cluster_id}'
            AND timestamp >= CURRENT_TIMESTAMP() - INTERVAL 7 DAYS
        """
        
        metrics = spark.sql(metrics_query).collect()[0]
        
        recommendations = []
        
        # Check if overprovisioned
        if metrics.avg_cpu < utilization_threshold:
            recommendations.append(
                f"CPU utilization low ({metrics.avg_cpu:.1%}). "
                f"Consider smaller node types or fewer workers."
            )
        
        if metrics.avg_memory < utilization_threshold:
            recommendations.append(
                f"Memory utilization low ({metrics.avg_memory:.1%}). "
                f"Consider memory-optimized instances."
            )
        
        if metrics.avg_workers < metrics.max_workers * 0.5:
            recommendations.append(
                f"Average workers ({metrics.avg_workers:.0f}) much lower than max "
                f"({metrics.max_workers}). Reduce max_workers in autoscaling."
            )
        
        return recommendations
    
    def enable_photon_where_applicable(self):
        """Enable Photon on compatible job clusters"""
        jobs = self.w.jobs.list()
        updated_count = 0
        
        for job in jobs:
            job_details = self.w.jobs.get(job.job_id)
            
            # Check if job uses SQL/DataFrame operations (good for Photon)
            # and doesn't already have Photon enabled
            for task in job_details.settings.tasks:
                if task.notebook_task and not self._has_photon(task):
                    # Update to use Photon
                    print(f"Enabling Photon for job: {job.settings.name}")
                    # Update job configuration
                    updated_count += 1
        
        return updated_count
    
    def _has_photon(self, task) -> bool:
        """Check if task already uses Photon"""
        if hasattr(task, 'new_cluster') and task.new_cluster:
            return task.new_cluster.runtime_engine == "PHOTON"
        return False
    
    def generate_cost_report(self, days: int = 30) -> pd.DataFrame:
        """Generate cost report for the past N days"""
        query = f"""
        SELECT 
            DATE(usage_start_time) as date,
            usage_metadata.custom_tags['Team'] as team,
            usage_metadata.custom_tags['Project'] as project,
            sku_name,
            SUM(usage_quantity) as total_dbus,
            SUM(usage_quantity) * 0.40 as estimated_cost_usd
        FROM system.billing.usage
        WHERE usage_date >= CURRENT_DATE() - INTERVAL {days} DAYS
        GROUP BY 1, 2, 3, 4
        ORDER BY date DESC, estimated_cost_usd DESC
        """
        
        return spark.sql(query).toPandas()


# Usage
optimizer = CostOptimizer()

# Terminate idle clusters
terminated = optimizer.terminate_idle_clusters(idle_threshold_hours=2)
print(f"Terminated {terminated} idle clusters")

# Get cost report
cost_report = optimizer.generate_cost_report(days=30)
print(f"\nTotal cost last 30 days: ${cost_report['estimated_cost_usd'].sum():,.2f}")
```

### Example 4: Cluster Pools for Cost Efficiency

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.compute import CreateInstancePool, AwsAttributes, DiskSpec, AwsAvailability

w = WorkspaceClient()

# Create instance pool for fast startup and cost efficiency
pool = w.instance_pools.create(
    instance_pool_name="general-purpose-pool",
    
    # Node configuration
    node_type_id="i3.xlarge",
    
    # Use spot instances for pool
    aws_attributes=AwsAttributes(
        availability=AwsAvailability.SPOT_WITH_FALLBACK,
        zone_id="us-west-2a",
        spot_bid_price_percent=100
    ),
    
    # Pool sizing
    min_idle_instances=2,  # Keep 2 warm for fast startup
    max_capacity=50,  # Maximum pool size
    idle_instance_autotermination_minutes=15,  # Terminate unused after 15 min
    
    # Disk configuration
    disk_spec=DiskSpec(
        disk_type="EBS_VOLUME_TYPE_GENERAL_PURPOSE_SSD",
        disk_size=100
    ),
    
    # Preload libraries for faster startup
    preloaded_spark_versions=[
        "13.3.x-scala2.12"
    ],
    
    # Cost allocation
    custom_tags={
        "Pool": "GeneralPurpose",
        "CostCenter": "SharedServices"
    }
)

print(f"Created pool: {pool.instance_pool_id}")

# Use pool in job cluster
from databricks.sdk.service.jobs import JobCluster, ClusterSpec

job_cluster = JobCluster(
    job_cluster_key="pooled_cluster",
    new_cluster=ClusterSpec(
        spark_version="13.3.x-scala2.12",
        instance_pool_id=pool.instance_pool_id,  # Use pool
        num_workers=5,
        autotermination_minutes=10
    )
)
```

### Example 5: Storage Cost Optimization

```python
from pyspark.sql.functions import col, current_date, datediff

# Implement data lifecycle management
def optimize_table_storage(
    catalog: str,
    schema: str,
    table: str,
    retention_days: int = 90
):
    """
    Optimize storage costs through:
    1. Partition pruning
    2. VACUUM old files
    3. OPTIMIZE for fewer files
    4. Archive old data to cheaper storage
    """
    
    full_table = f"{catalog}.{schema}.{table}"
    
    # 1. Identify old data for archival
    old_data_query = f"""
    SELECT COUNT(*) as old_rows, 
           SUM(size_bytes) / 1024 / 1024 / 1024 as old_data_gb
    FROM {full_table}
    WHERE event_date < CURRENT_DATE() - INTERVAL {retention_days} DAYS
    """
    
    old_stats = spark.sql(old_data_query).collect()[0]
    print(f"Old data to archive: {old_stats.old_rows:,} rows, {old_stats.old_data_gb:.2f} GB")
    
    # 2. Archive old data to external storage (S3 Standard-IA or Glacier)
    archive_path = f"s3://bucket/archives/{catalog}/{schema}/{table}"
    
    spark.sql(f"""
        INSERT OVERWRITE {archive_path}
        SELECT * FROM {full_table}
        WHERE event_date < CURRENT_DATE() - INTERVAL {retention_days} DAYS
    """)
    
    # 3. Delete archived data from hot storage
    spark.sql(f"""
        DELETE FROM {full_table}
        WHERE event_date < CURRENT_DATE() - INTERVAL {retention_days} DAYS
    """)
    
    # 4. OPTIMIZE to reduce file count
    spark.sql(f"OPTIMIZE {full_table}")
    
    # 5. VACUUM to remove old files (save storage costs)
    spark.sql(f"VACUUM {full_table} RETAIN 7 HOURS")
    
    print(f"Storage optimization complete for {full_table}")


# Configure S3 lifecycle policies for archived data
lifecycle_policy = {
    "Rules": [
        {
            "Id": "ArchiveOldData",
            "Status": "Enabled",
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "STANDARD_IA"  # Cheaper after 30 days
                },
                {
                    "Days": 90,
                    "StorageClass": "GLACIER"  # Very cheap for long-term
                }
            ]
        }
    ]
}
```

### Example 6: Budget Alerts and Governance

```python
# Create budget monitoring workflow
budget_alert_notebook = """
# Budget Alert Notebook

from pyspark.sql.functions import sum as spark_sum, current_date
from databricks.sdk import WorkspaceClient

# Define budgets by team
TEAM_BUDGETS = {
    "DataEngineering": 10000,  # $10k/month
    "DataScience": 8000,
    "Analytics": 5000
}

# Query current month spend
current_spend_query = '''
SELECT 
    usage_metadata.custom_tags['Team'] as team,
    SUM(usage_quantity) * 0.40 as spend_usd
FROM system.billing.usage
WHERE 
    YEAR(usage_date) = YEAR(CURRENT_DATE())
    AND MONTH(usage_date) = MONTH(CURRENT_DATE())
GROUP BY team
'''

current_spend = spark.sql(current_spend_query).collect()

# Check against budgets and alert
w = WorkspaceClient()
alerts = []

for row in current_spend:
    team = row.team
    spend = row.spend_usd
    budget = TEAM_BUDGETS.get(team, float('inf'))
    
    usage_pct = (spend / budget) * 100
    
    if usage_pct > 90:
        alert_msg = f"⚠️ CRITICAL: {team} at {usage_pct:.0f}% of monthly budget (${spend:,.2f}/${budget:,.2f})"
        alerts.append(alert_msg)
        
        # Send notification (Slack, email, etc.)
        print(alert_msg)
    
    elif usage_pct > 75:
        alert_msg = f"⚠️ WARNING: {team} at {usage_pct:.0f}% of monthly budget (${spend:,.2f}/${budget:,.2f})"
        alerts.append(alert_msg)
        print(alert_msg)

# Write alerts to table for dashboarding
if alerts:
    alert_df = spark.createDataFrame(
        [(current_date(), alert) for alert in alerts],
        ["alert_date", "message"]
    )
    alert_df.write.mode("append").saveAsTable("monitoring.budget_alerts")
'''
```

### Example 7: Photon vs. Standard Cost-Benefit Analysis

```python
from pyspark.sql.functions import col, sum as spark_sum, avg

def analyze_photon_roi(
    job_id: str,
    photon_speedup_factor: float = 2.5,
    photon_price_multiplier: float = 1.0  # Photon included in Premium
):
    """
    Analyze whether enabling Photon saves money for a specific job
    """
    
    # Get historical runtime without Photon
    query = f"""
    SELECT 
        AVG(execution_duration_seconds) / 3600 as avg_runtime_hours,
        AVG(cluster_size) as avg_cluster_size,
        COUNT(*) as num_runs
    FROM system.jobs.runs
    WHERE job_id = {job_id}
        AND start_time >= CURRENT_DATE() - INTERVAL 30 DAYS
    """
    
    baseline = spark.sql(query).collect()[0]
    
    # Assume DBU rate
    dbu_per_hour = 0.40  # Example rate
    workers = baseline.avg_cluster_size
    
    # Calculate costs
    baseline_runtime_hours = baseline.avg_runtime_hours
    baseline_cost_per_run = baseline_runtime_hours * workers * dbu_per_hour
    
    # Photon optimization
    photon_runtime_hours = baseline_runtime_hours / photon_speedup_factor
    photon_cost_per_run = photon_runtime_hours * workers * dbu_per_hour * photon_price_multiplier
    
    # Monthly savings
    runs_per_month = baseline.num_runs
    monthly_savings = (baseline_cost_per_run - photon_cost_per_run) * runs_per_month
    annual_savings = monthly_savings * 12
    
    report = {
        "job_id": job_id,
        "baseline_runtime_hours": baseline_runtime_hours,
        "photon_runtime_hours": photon_runtime_hours,
        "speedup_factor": baseline_runtime_hours / photon_runtime_hours,
        "baseline_cost_per_run": baseline_cost_per_run,
        "photon_cost_per_run": photon_cost_per_run,
        "savings_per_run": baseline_cost_per_run - photon_cost_per_run,
        "monthly_savings": monthly_savings,
        "annual_savings": annual_savings,
        "recommendation": "Enable Photon" if monthly_savings > 0 else "Keep Standard"
    }
    
    print(f"\n{'='*60}")
    print(f"Photon ROI Analysis for Job {job_id}")
    print(f"{'='*60}")
    print(f"Baseline runtime:     {baseline_runtime_hours:.2f} hours")
    print(f"Photon runtime:       {photon_runtime_hours:.2f} hours ({photon_speedup_factor}x faster)")
    print(f"Savings per run:      ${baseline_cost_per_run - photon_cost_per_run:.2f}")
    print(f"Monthly savings:      ${monthly_savings:.2f}")
    print(f"Annual savings:       ${annual_savings:.2f}")
    print(f"\n✅ Recommendation: {report['recommendation']}")
    print(f"{'='*60}\n")
    
    return report


# Analyze multiple jobs
job_ids = [123, 456, 789]
results = [analyze_photon_roi(job_id) for job_id in job_ids]

# Convert to DataFrame for reporting
results_df = spark.createDataFrame(results)
results_df.write.mode("overwrite").saveAsTable("cost_analysis.photon_roi")
```

---

### Example 8: Serverless vs Classic Cost Analysis

```python
# Compare serverless vs classic compute costs
import pandas as pd

# Serverless pricing (example rates)
serverless_dbu_rate = 0.70  # $/DBU
serverless_overhead = 0  # No cluster management

# Classic cluster pricing
classic_dbu_rate = 0.40  # $/DBU
classic_startup_time = 5  # minutes wasted on startup
classic_idle_time = 30  # minutes of over-provisioning
classic_management_hours = 2  # hrs/week for tuning

def calculate_serverless_cost(query_minutes, queries_per_day):
    # Serverless: pay only for query execution
    daily_dbu = (query_minutes / 60) * queries_per_day
    monthly_cost = daily_dbu * 30 * serverless_dbu_rate
    return monthly_cost

def calculate_classic_cost(query_minutes, queries_per_day):
    # Classic: pay for startup + idle + execution
    total_minutes = (query_minutes + classic_startup_time + classic_idle_time) * queries_per_day
    daily_dbu = total_minutes / 60
    monthly_cost = daily_dbu * 30 * classic_dbu_rate
    management_cost = classic_management_hours * 4 * 150  # $150/hr eng cost
    return monthly_cost + management_cost

# Scenario: Intermittent analytics workload
scenarios = [
    ("Ad-hoc queries (5 min each, 10/day)", 5, 10),
    ("Scheduled reports (15 min each, 4/day)", 15, 4),
    ("ML training (60 min each, 2/day)", 60, 2),
]

results = []
for scenario, mins, count in scenarios:
    serverless = calculate_serverless_cost(mins, count)
    classic = calculate_classic_cost(mins, count)
    savings = ((classic - serverless) / classic) * 100
    
    results.append({
        'Scenario': scenario,
        'Serverless': f"${serverless:.2f}/mo",
        'Classic': f"${classic:.2f}/mo",
        'Savings': f"{savings:.1f}%"
    })

df = pd.DataFrame(results)
print(df.to_string(index=False))
```

**Serverless Recommendations:**
* ✅ Use for: Ad-hoc analytics, BI dashboards, intermittent jobs
* ✅ Use for: Development/testing (instant-on)
* ✅ Use for: Data science notebooks (auto-scaling)
* ⚠️ Consider classic for: 24/7 streaming, large batch ETL (>4 hours)

**Cost Optimization with Serverless:**
```sql
-- Enable serverless for SQL warehouse (Admin console)
-- Databricks SQL → SQL Warehouses → Create → Select "Serverless"

-- Monitor serverless usage
SELECT 
    DATE(usage_date) as date,
    SUM(usage_quantity) as total_dbu,
    SUM(usage_quantity * list_price) as cost_usd
FROM system.billing.usage
WHERE sku_name LIKE '%SERVERLESS%'
    AND usage_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY DATE(usage_date)
ORDER BY date DESC;
```


## Best Practices

### 1. Always Use Job Clusters
* Job clusters terminate after completion
* All-purpose clusters run indefinitely
* Job clusters ~40-60% cheaper for scheduled workloads

### 2. Enable Autoscaling
* Start small (min_workers=2)
* Set reasonable max based on workload
* Use scale-down timeout (5-10 minutes)
* Monitor actual usage to adjust

### 3. Leverage Spot Instances
* Use for all fault-tolerant workloads
* Always use SPOT_WITH_FALLBACK
* Keep driver on-demand (first_on_demand=1)
* Save 60-90% on worker costs

### 4. Implement Cost Tags
* Tag all clusters: Project, Team, Environment, CostCenter
* Use tags in billing analysis
* Create chargeback reports by tag
* Enforce tagging policies

### 5. Monitor and Alert
* Set up budget alerts at 75%, 90% thresholds
* Track cost trends weekly
* Identify anomalies early
* Review top spenders monthly

### 6. Optimize Storage
* Use lifecycle policies (Standard → IA → Glacier)
* VACUUM regularly to remove old files
* OPTIMIZE to reduce file count
* Archive cold data to cheaper tiers

---

## Common Pitfalls

### ❌ All-Purpose Clusters for Scheduled Jobs
* **Problem**: Clusters run 24/7 even when idle
* **Solution**: Always use job clusters for automation

### ❌ No Autotermination
* **Problem**: Forgotten clusters run indefinitely
* **Solution**: Set autotermination to 10-30 minutes

### ❌ Over-Provisioned Clusters
* **Problem**: Paying for unused capacity
* **Solution**: Start small, monitor utilization, scale based on data

### ❌ Not Using Spot Instances
* **Problem**: Paying 2-3x more than necessary
* **Solution**: Use spot for all retryable workloads

### ❌ No Cost Tags
* **Problem**: Can't track or allocate costs
* **Solution**: Implement tagging policy, enforce in workspace

### ❌ Ignoring Photon
* **Problem**: Missing 2-3x speedup = higher costs
* **Solution**: Enable Photon for SQL/DataFrame workloads

---

## Cost Optimization Checklist

### Compute
- [ ] Use job clusters for scheduled workloads
- [ ] Enable autoscaling (min=2, max based on workload)
- [ ] Use spot instances with fallback
- [ ] Enable Photon where applicable
- [ ] Set autotermination (10-30 minutes)
- [ ] Use cluster pools for frequent startups
- [ ] Right-size node types for workload

### Storage
- [ ] Implement lifecycle policies (Standard → IA → Glacier)
- [ ] VACUUM tables regularly (remove old files)
- [ ] OPTIMIZE tables to reduce file count
- [ ] Archive cold data to cheaper storage
- [ ] Use compression (already enabled in Delta)

### Monitoring
- [ ] Tag all resources (Project, Team, Environment)
- [ ] Set up budget alerts (75%, 90%)
- [ ] Create cost dashboards by team/project
- [ ] Review top spenders monthly
- [ ] Identify and terminate idle clusters

### Governance
- [ ] Enforce cluster policies (size limits, spot usage)
- [ ] Require cost tags on all resources
- [ ] Implement approval for large clusters
- [ ] Regular cost reviews with teams
- [ ] Track cost per pipeline/project

---

## Related Skills

* [Spark Optimization](../spark-optimization/SKILL.md) - For performance optimization
* [Performance Tuning](../performance-tuning/SKILL.md) - For query optimization
* [Workflow Orchestration](../workflow-orchestration/SKILL.md) - For job configuration
* [Monitoring Observability](../monitoring-observability/SKILL.md) - For cost tracking

---

## Integration with Databricks Features

### Account Console
* View usage and costs across workspaces
* Set budgets and alerts
* Download detailed billing reports
* Track DBU consumption

### System Tables
* Query usage programmatically (system.billing.usage)
* Analyze cluster metrics (system.compute.cluster_metrics)
* Build custom cost dashboards
* Automate cost reporting

### Cluster Policies
* Enforce spot instance usage
* Limit cluster sizes
* Require cost tags
* Prevent expensive configurations

### Workflows
* Schedule cost optimization scripts
* Automate budget checks
* Terminate idle clusters
* Generate cost reports

---

**Last Updated**: 2024-04-09
