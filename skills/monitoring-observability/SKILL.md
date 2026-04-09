# Monitoring & Observability Skill

## Purpose
This skill teaches Genie Code how to implement comprehensive monitoring and observability in Databricks using metrics collection, alerting, dashboards, log aggregation, tracing, and SLO/SLI tracking for production data platforms.

## When to Use
* Monitoring pipeline health and performance
* Tracking data quality metrics
* Setting up operational alerting
* Debugging production issues
* Measuring SLA compliance
* Creating operational dashboards
* Implementing incident response

## Key Concepts

### 1. Monitoring Strategy Framework
* **Situation**: Current system state and baseline
* **Motivation**: Why specific metrics matter
* **Restrictions**: What to monitor and measure
* **Evaluation**: SLOs, SLIs, error budgets
* **Decisions**: Escalation paths and responses

### 2. Observability Pillars
* **Metrics**: Quantitative measurements (latency, throughput, errors)
* **Logs**: Event records with context
* **Traces**: Request/job flow tracking
* **Alerts**: Proactive problem notification

### 3. Monitoring Levels
* **Infrastructure**: Cluster CPU, memory, disk
* **Pipeline**: Job duration, success rate, data volume
* **Data quality**: Completeness, accuracy, freshness
* **Business**: KPIs, SLA compliance

### 4. Alert Strategy
* **Severity levels**: Critical, warning, info
* **Alert channels**: Email, Slack, PagerDuty
* **On-call rotation**: Escalation procedures
* **Runbooks**: Response documentation

---

## Examples

### Example 1: Comprehensive Pipeline Metrics

```python
from pyspark.sql.functions import *
from datetime import datetime
import json

class PipelineMonitor:
    """Track metrics for data quality and SLA compliance"""
    
    def __init__(self, pipeline_name):
        self.pipeline_name = pipeline_name
        self.metrics_table = "monitoring.pipeline_metrics"
        self.run_start = datetime.now()
    
    def collect_metrics(self, df, stage_name):
        """Collect detailed metrics for a pipeline stage"""
        
        metrics = {
            'record_count': df.count(),
            'column_count': len(df.columns),
            'partition_count': df.rdd.getNumPartitions(),
            'duplicate_count': df.count() - df.distinct().count(),
            'estimated_size_mb': self._estimate_size(df)
        }
        
        self._save_metrics(stage_name, metrics)
        return metrics
    
    def _estimate_size(self, df):
        """Estimate DataFrame size"""
        sample_size = min(1000, df.count())
        if sample_size == 0:
            return 0
        
        sample_json = df.limit(sample_size).toJSON().collect()
        sample_bytes = sum(len(s.encode('utf-8')) for s in sample_json)
        estimated_mb = (sample_bytes / sample_size) * df.count() / (1024 * 1024)
        return round(estimated_mb, 2)
    
    def _save_metrics(self, stage_name, metrics):
        """Save metrics to monitoring table"""
        spark.createDataFrame([
            (datetime.now(), self.pipeline_name, stage_name,
             metrics['record_count'], metrics['estimated_size_mb'])
        ], ["timestamp", "pipeline", "stage", "record_count", "size_mb"]
        ).write.mode("append").saveAsTable(self.metrics_table)
    
    def track_sla(self, sla_name, actual_value, target_value, unit="minutes"):
        """Track SLA compliance and alert on violations"""
        compliance = actual_value <= target_value
        
        spark.createDataFrame([
            (datetime.now(), self.pipeline_name, sla_name,
             actual_value, target_value, unit, compliance)
        ], ["timestamp", "pipeline", "sla_name", "actual", "target", "unit", "compliant"]
        ).write.mode("append").saveAsTable("monitoring.sla_tracking")
        
        if not compliance:
            print(f"🚨 SLA VIOLATION: {sla_name} - {actual_value} {unit} (target: {target_value})")
        
        return compliance

# Usage
monitor = PipelineMonitor("customer_etl")

# Collect metrics
df = spark.table("bronze.customers")
metrics = monitor.collect_metrics(df, "bronze_read")

# Track SLAs
monitor.track_sla("processing_latency", actual_value=45, target_value=60, unit="minutes")
monitor.track_sla("data_freshness", actual_value=2, target_value=4, unit="hours")
```

### Example 2: Data Quality Monitoring

```python
def monitor_table_quality(table_name, quality_checks):
    """Run quality checks and store results"""
    
    df = spark.table(table_name)
    results = []
    
    for check in quality_checks:
        if check['type'] == 'completeness':
            null_pct = df.filter(col(check['column']).isNull()).count() / df.count()
            passed = null_pct <= check['threshold']
            results.append({
                'timestamp': datetime.now(),
                'table_name': table_name,
                'metric_name': f"completeness_{check['column']}",
                'metric_value': 1 - null_pct,
                'passed': passed
            })
        
        elif check['type'] == 'uniqueness':
            total = df.count()
            distinct = df.select(check['column']).distinct().count()
            uniqueness = distinct / total
            passed = uniqueness >= check['threshold']
            results.append({
                'timestamp': datetime.now(),
                'table_name': table_name,
                'metric_name': f"uniqueness_{check['column']}",
                'metric_value': uniqueness,
                'passed': passed
            })
        
        elif check['type'] == 'freshness':
            max_timestamp = df.agg(max(check['column'])).collect()[0][0]
            hours_old = (datetime.now() - max_timestamp).total_seconds() / 3600
            passed = hours_old <= check['threshold']
            results.append({
                'timestamp': datetime.now(),
                'table_name': table_name,
                'metric_name': f"freshness_{check['column']}",
                'metric_value': hours_old,
                'passed': passed
            })
    
    # Save results
    spark.createDataFrame(results).write.mode("append").saveAsTable("monitoring.data_quality_metrics")
    
    return results

# Define quality checks
quality_checks = [
    {'type': 'completeness', 'column': 'customer_id', 'threshold': 0.01},  # Max 1% nulls
    {'type': 'uniqueness', 'column': 'customer_id', 'threshold': 0.99},    # Min 99% unique
    {'type': 'freshness', 'column': 'updated_at', 'threshold': 24}          # Max 24 hours old
]

# Run monitoring
results = monitor_table_quality("gold.customers", quality_checks)
print(f"✅ {sum(1 for r in results if r['passed'])}/{len(results)} quality checks passed")
```

### Example 3: Alerting System

```python
from enum import Enum

class AlertSeverity(Enum):
    CRITICAL = "critical"  # Page on-call immediately
    WARNING = "warning"    # Notify team channel
    INFO = "info"          # Log only

class AlertManager:
    """Manage alerts with severity-based routing"""
    
    def __init__(self):
        self.alert_history_table = "monitoring.alert_history"
    
    def send_alert(self, alert_name, message, severity, metadata=None):
        """Send alert through appropriate channels"""
        
        alert_id = str(uuid.uuid4())
        
        # Log alert
        spark.createDataFrame([
            (alert_id, datetime.now(), alert_name, severity.value, message)
        ], ["alert_id", "timestamp", "alert_name", "severity", "message"]
        ).write.mode("append").saveAsTable(self.alert_history_table)
        
        # Route by severity
        if severity == AlertSeverity.CRITICAL:
            self._page_oncall(alert_name, message)
        elif severity == AlertSeverity.WARNING:
            self._notify_team(alert_name, message)
        else:
            print(f"[INFO] {alert_name}: {message}")
    
    def _page_oncall(self, alert_name, message):
        """Send critical alert to on-call"""
        print(f"🚨 PAGING ON-CALL: {alert_name} - {message}")
        # Send to PagerDuty, Slack with @channel, etc.
    
    def _notify_team(self, alert_name, message):
        """Send warning to team channel"""
        print(f"⚠️ TEAM ALERT: {alert_name} - {message}")
        # Send to Slack team channel, email, etc.
    
    def check_alert_rules(self):
        """Check monitoring data against alert rules"""
        
        # Rule: Pipeline failure rate > 5%
        failure_rate = spark.sql("""
            SELECT pipeline,
                   COUNT(*) as total_runs,
                   SUM(CASE WHEN status = 'FAILED' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as failure_rate_pct
            FROM monitoring.pipeline_runs
            WHERE timestamp >= CURRENT_TIMESTAMP() - INTERVAL 1 HOUR
            GROUP BY pipeline
            HAVING failure_rate_pct > 5
        """)
        
        for row in failure_rate.collect():
            self.send_alert(
                alert_name=f"high_failure_rate_{row.pipeline}",
                message=f"Pipeline {row.pipeline} has {row.failure_rate_pct:.1f}% failure rate",
                severity=AlertSeverity.WARNING
            )

# Usage
alert_mgr = AlertManager()

# Manual alert
alert_mgr.send_alert(
    alert_name="data_pipeline_failure",
    message="Customer ETL pipeline failed after 3 retries",
    severity=AlertSeverity.CRITICAL
)

# Automated alert checks
alert_mgr.check_alert_rules()
```

### Example 4: Operational Dashboard Queries

```python
# Pipeline Health Overview
pipeline_health = spark.sql("""
SELECT 
    pipeline,
    DATE(timestamp) as date,
    COUNT(*) as total_runs,
    SUM(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END) as successful_runs,
    SUM(CASE WHEN status = 'FAILED' THEN 1 ELSE 0 END) as failed_runs,
    AVG(duration_minutes) as avg_duration_minutes
FROM monitoring.pipeline_runs
WHERE timestamp >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY pipeline, DATE(timestamp)
ORDER BY date DESC, pipeline
""")

# Data Quality Trends
quality_trends = spark.sql("""
SELECT 
    table_name,
    metric_name,
    DATE(timestamp) as date,
    AVG(metric_value) as avg_value,
    AVG(CASE WHEN passed THEN 100.0 ELSE 0.0 END) as pass_rate_pct
FROM monitoring.data_quality_metrics
WHERE timestamp >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY table_name, metric_name, DATE(timestamp)
ORDER BY date DESC
""")

# SLA Compliance
sla_compliance = spark.sql("""
SELECT 
    pipeline,
    sla_name,
    DATE(timestamp) as date,
    AVG(actual) as avg_actual,
    MAX(target) as target,
    AVG(CASE WHEN compliant THEN 100.0 ELSE 0.0 END) as compliance_pct
FROM monitoring.sla_tracking
WHERE timestamp >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY pipeline, sla_name, DATE(timestamp)
ORDER BY date DESC
""")

print("=" * 80)
print("MONITORING DASHBOARD")
print("=" * 80)
print("\n1. PIPELINE HEALTH")
pipeline_health.display()

print("\n2. DATA QUALITY TRENDS")
quality_trends.display()

print("\n3. SLA COMPLIANCE")
sla_compliance.display()
```

### Example 5: Pipeline Tracing

```python
import uuid

class PipelineTracer:
    """Distributed tracing for pipeline observability"""
    
    def __init__(self, pipeline_name):
        self.pipeline_name = pipeline_name
        self.trace_id = str(uuid.uuid4())
        self.spans = []
    
    def trace_operation(self, operation_name):
        """Context manager for traced operations"""
        span_id = str(uuid.uuid4())
        start_time = datetime.now()
        
        class SpanContext:
            def __enter__(self):
                return self
            
            def __exit__(self, exc_type, exc_val, exc_tb):
                end_time = datetime.now()
                duration_ms = (end_time - start_time).total_seconds() * 1000
                status = 'success' if exc_type is None else 'error'
                
                # Save span
                spark.createDataFrame([
                    (trace_id, span_id, pipeline_name, operation_name,
                     start_time, end_time, duration_ms, status)
                ], ["trace_id", "span_id", "pipeline", "operation",
                    "start_time", "end_time", "duration_ms", "status"]
                ).write.mode("append").saveAsTable("monitoring.pipeline_traces")
        
        return SpanContext()

# Usage
tracer = PipelineTracer("customer_etl")

with tracer.trace_operation("read_source"):
    df = spark.read.parquet("s3://bucket/data/")

with tracer.trace_operation("transformation"):
    transformed_df = df.transform(apply_transformations)

with tracer.trace_operation("write_target"):
    transformed_df.write.mode("overwrite").saveAsTable("gold.customers")

print(f"✅ Trace ID: {tracer.trace_id}")

# Query trace performance
trace_analysis = spark.sql("""
SELECT 
    pipeline,
    operation,
    AVG(duration_ms) as avg_duration_ms,
    PERCENTILE(duration_ms, 0.95) as p95_duration_ms
FROM monitoring.pipeline_traces
WHERE start_time >= CURRENT_TIMESTAMP() - INTERVAL 7 DAYS
GROUP BY pipeline, operation
ORDER BY avg_duration_ms DESC
""")
```

---

## Best Practices

### 1. Metrics Strategy
* Define **what to measure** based on business impact
* Set **realistic thresholds** (don't alert on everything)
* Track **trends** not just point-in-time values
* Monitor **leading indicators** (predict problems)

### 2. Alert Design
* Make alerts **actionable** (include runbook links)
* Use **severity levels** appropriately
* Avoid **alert fatigue** (tune thresholds)
* Include **context** in alert messages

### 3. Dashboard Design
* Show **most important metrics** at the top
* Use **time-series visualizations** for trends
* Highlight **anomalies** and **violations**
* Link to **detailed logs** and traces

### 4. SLO/SLI Tracking
* Define **Service Level Objectives** (targets)
* Measure **Service Level Indicators** (actual)
* Calculate **error budgets**
* Review and adjust quarterly

---

## Related Skills

* [Data Contracts](../data-contracts/SKILL.md) - For SLA definitions
* [Error Handling](../error-handling/SKILL.md) - For error logging
* [Testing Strategies](../testing-strategies/SKILL.md) - For test monitoring

---

**Last Updated**: 2026-04-09
