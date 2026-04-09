# Model Deployment Skill

## Purpose
This skill teaches Genie Code how to deploy machine learning models on Databricks using MLflow Model Registry, model serving endpoints, batch inference, A/B testing, and model monitoring for production ML systems.

## When to Use
* Deploying trained models to production
* Creating real-time model serving endpoints
* Implementing batch inference pipelines
* A/B testing multiple model versions
* Monitoring model performance in production
* Managing model lifecycle and governance
* Implementing CI/CD for ML models

## Key Concepts

### 1. MLflow Model Registry
* **Model versioning**: Track model evolution
* **Stage transitions**: Dev → Staging → Production
* **Model lineage**: Track training runs and data
* **Aliases**: Reference models by name
* **Webhooks**: Automate deployment workflows

### 2. Model Serving
* **Real-time endpoints**: Low-latency predictions
* **Serverless**: Auto-scaling, pay-per-use
* **GPU support**: For deep learning models
* **Authentication**: Token-based security
* **Rate limiting**: Protect endpoints from overload

### 3. Batch Inference
* **Scheduled jobs**: Process data in batches
* **Distributed**: Leverage Spark parallelism
* **Cost-effective**: For high-volume predictions
* **Delta integration**: Read/write to Delta tables

### 4. A/B Testing
* **Traffic splitting**: Route requests to versions
* **Champion/Challenger**: Compare model versions
* **Metrics tracking**: Monitor performance differences
* **Gradual rollout**: Minimize deployment risk

### 5. Model Monitoring
* **Drift detection**: Input and prediction drift
* **Performance metrics**: Accuracy, latency, throughput
* **Data quality**: Validate input data
* **Alerts**: Notify stakeholders on issues

---

## Examples

### Example 1: Register and Manage Models in Registry

```python
import mlflow
from mlflow.tracking import MlflowClient
from sklearn.ensemble import RandomForestClassifier
import pandas as pd

client = MlflowClient()

# Train and register model
def train_and_register_model():
    """Train model and register in MLflow Model Registry"""
    
    with mlflow.start_run(run_name="customer_churn_rf_v1") as run:
        # Load training data
        X_train, y_train = load_training_data()
        
        # Train model
        model = RandomForestClassifier(
            n_estimators=100,
            max_depth=10,
            random_state=42
        )
        model.fit(X_train, y_train)
        
        # Log parameters
        mlflow.log_param("n_estimators", 100)
        mlflow.log_param("max_depth", 10)
        mlflow.log_param("model_type", "RandomForest")
        
        # Evaluate and log metrics
        y_pred = model.predict(X_test)
        accuracy = accuracy_score(y_test, y_pred)
        auc = roc_auc_score(y_test, model.predict_proba(X_test)[:, 1])
        
        mlflow.log_metric("accuracy", accuracy)
        mlflow.log_metric("auc", auc)
        mlflow.log_metric("precision", precision_score(y_test, y_pred))
        mlflow.log_metric("recall", recall_score(y_test, y_pred))
        
        # Log model with signature and input example
        signature = infer_signature(X_train, model.predict(X_train))
        input_example = X_train.iloc[:5]
        
        mlflow.sklearn.log_model(
            sk_model=model,
            artifact_path="model",
            registered_model_name="customer_churn_classifier",
            signature=signature,
            input_example=input_example,
            pip_requirements=["scikit-learn==1.3.0", "pandas==2.0.3"]
        )
        
        return run.info.run_id

# Register model
run_id = train_and_register_model()

# Get latest version
model_name = "customer_churn_classifier"
latest_version = client.get_latest_versions(model_name, stages=["None"])[0]

print(f"Model registered: {model_name}, Version: {latest_version.version}")

# Add description and tags
client.update_model_version(
    name=model_name,
    version=latest_version.version,
    description="Random Forest model trained on Q1 2024 data. Accuracy: 92%, AUC: 0.88"
)

client.set_model_version_tag(
    name=model_name,
    version=latest_version.version,
    key="training_dataset",
    value="churn_data_2024_q1"
)

client.set_model_version_tag(
    name=model_name,
    version=latest_version.version,
    key="validation_date",
    value="2024-04-09"
)

# Transition to Staging
client.transition_model_version_stage(
    name=model_name,
    version=latest_version.version,
    stage="Staging",
    archive_existing_versions=True  # Archive old staging versions
)

# After validation in staging, promote to Production
client.transition_model_version_stage(
    name=model_name,
    version=latest_version.version,
    stage="Production",
    archive_existing_versions=False  # Keep old production for rollback
)

# Set alias for easy reference
client.set_registered_model_alias(
    name=model_name,
    alias="champion",
    version=latest_version.version
)

print(f"Model promoted to Production with alias 'champion'")
```

### Example 2: Create Model Serving Endpoint

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import (
    ServedEntityInput,
    EndpointCoreConfigInput,
    AutoCaptureConfigInput,
    TrafficConfig,
    Route
)

w = WorkspaceClient()

# Create serverless model serving endpoint
def create_serving_endpoint(model_name, model_version):
    """Create a serverless model serving endpoint"""
    
    endpoint_name = f"{model_name.replace('_', '-')}-endpoint"
    
    try:
        # Create endpoint
        endpoint = w.serving_endpoints.create(
            name=endpoint_name,
            config=EndpointCoreConfigInput(
                served_entities=[
                    ServedEntityInput(
                        entity_name=model_name,
                        entity_version=str(model_version),
                        scale_to_zero_enabled=True,  # Scale down when idle
                        workload_size="Small",        # Small, Medium, Large
                        workload_type="CPU"           # CPU or GPU
                    )
                ],
                # Enable inference table logging
                auto_capture_config=AutoCaptureConfigInput(
                    catalog_name="catalog",
                    schema_name="ml_monitoring",
                    table_name_prefix=model_name
                )
            )
        )
        
        print(f"Creating endpoint: {endpoint_name}")
        
        # Wait for endpoint to be ready
        w.serving_endpoints.wait_get_serving_endpoint_not_updating(endpoint_name)
        
        print(f"✅ Endpoint ready: {endpoint_name}")
        return endpoint
        
    except Exception as e:
        print(f"Error creating endpoint: {e}")
        return None

# Create endpoint
endpoint = create_serving_endpoint("customer_churn_classifier", 1)

# Query endpoint
def query_endpoint(endpoint_name, input_data):
    """Query model serving endpoint"""
    import requests
    import json
    
    # Get Databricks token
    token = dbutils.notebook.entry_point.getDbutils().notebook().getContext().apiToken().get()
    host = dbutils.notebook.entry_point.getDbutils().notebook().getContext().apiUrl().get()
    
    url = f"{host}/serving-endpoints/{endpoint_name}/invocations"
    
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    
    # Format input data
    data = {
        "dataframe_records": input_data
    }
    
    # Make prediction request
    response = requests.post(url, headers=headers, json=data, timeout=30)
    
    if response.status_code == 200:
        predictions = response.json()
        return predictions
    else:
        raise Exception(f"Error {response.status_code}: {response.text}")

# Test endpoint
test_data = [
    {"age": 35, "income": 75000, "tenure_months": 24, "support_calls": 3},
    {"age": 45, "income": 90000, "tenure_months": 48, "support_calls": 1}
]

predictions = query_endpoint("customer-churn-classifier-endpoint", test_data)
print(f"Predictions: {predictions}")
```

### Example 3: Batch Inference Pipeline

```python
import mlflow
from pyspark.sql.functions import struct, col, current_timestamp
from datetime import datetime

class BatchInferencePipeline:
    """
    Batch inference pipeline for model predictions
    """
    
    def __init__(self, model_name, model_stage="Production"):
        self.model_name = model_name
        self.model_stage = model_stage
        self.model_uri = f"models:/{model_name}/{model_stage}"
    
    def load_model(self):
        """Load model from registry"""
        print(f"Loading model: {self.model_uri}")
        self.model_udf = mlflow.pyfunc.spark_udf(
            spark, 
            model_uri=self.model_uri,
            result_type="double"  # Adjust based on model output
        )
        return self.model_udf
    
    def run_batch_inference(self, input_table, output_table, feature_cols):
        """
        Run batch inference on input table
        
        Args:
            input_table: Full table name (catalog.schema.table)
            output_table: Full table name for predictions
            feature_cols: List of feature column names
        """
        
        # Load input data
        df = spark.table(input_table)
        print(f"Loaded {df.count():,} rows from {input_table}")
        
        # Load model
        model_udf = self.load_model()
        
        # Apply model
        predictions_df = (
            df
            .withColumn(
                "prediction",
                model_udf(struct(*[col(c) for c in feature_cols]))
            )
            .withColumn("prediction_timestamp", current_timestamp())
            .withColumn("model_name", lit(self.model_name))
            .withColumn("model_version", lit(self.model_stage))
        )
        
        # Write predictions
        (
            predictions_df
            .write
            .mode("overwrite")
            .option("mergeSchema", "true")
            .saveAsTable(output_table)
        )
        
        print(f"✅ Predictions written to {output_table}")
        
        # Log metrics
        prediction_stats = predictions_df.select(
            count("*").alias("total_predictions"),
            avg("prediction").alias("avg_prediction"),
            sum(when(col("prediction") > 0.5, 1).otherwise(0)).alias("positive_predictions")
        ).collect()[0]
        
        print(f"\nPrediction Stats:")
        print(f"  Total: {prediction_stats.total_predictions:,}")
        print(f"  Average: {prediction_stats.avg_prediction:.4f}")
        print(f"  Positive: {prediction_stats.positive_predictions:,}")
        
        return predictions_df

# Usage
pipeline = BatchInferencePipeline(
    model_name="customer_churn_classifier",
    model_stage="Production"
)

feature_cols = ["age", "income", "tenure_months", "support_calls", "account_balance"]

predictions = pipeline.run_batch_inference(
    input_table="catalog.ml.customers_to_score",
    output_table="catalog.ml.churn_predictions",
    feature_cols=feature_cols
)

# Schedule batch inference job
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import Task, NotebookTask, JobCluster, ClusterSpec

w = WorkspaceClient()

job = w.jobs.create(
    name="Daily Churn Prediction Batch Job",
    tasks=[
        Task(
            task_key="batch_inference",
            notebook_task=NotebookTask(
                notebook_path="/ML/batch_inference_notebook",
                base_parameters={
                    "model_name": "customer_churn_classifier",
                    "input_table": "catalog.ml.customers_to_score",
                    "output_table": "catalog.ml.churn_predictions"
                }
            ),
            job_cluster_key="ml_cluster"
        )
    ],
    job_clusters=[
        JobCluster(
            job_cluster_key="ml_cluster",
            new_cluster=ClusterSpec(
                spark_version="13.3.x-ml-scala2.12",
                node_type_id="i3.xlarge",
                num_workers=4,
                autotermination_minutes=15
            )
        )
    ],
    schedule={
        "quartz_cron_expression": "0 0 2 * * ?",  # Daily at 2 AM
        "timezone_id": "America/Los_Angeles"
    }
)

print(f"Created batch inference job: {job.job_id}")
```

### Example 4: A/B Testing Model Versions

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import TrafficConfig, Route, ServedEntityInput

w = WorkspaceClient()

def setup_ab_test(endpoint_name, champion_version, challenger_version, traffic_split=90):
    """
    Set up A/B test between two model versions
    
    Args:
        endpoint_name: Name of serving endpoint
        champion_version: Current production model version
        challenger_version: New model version to test
        traffic_split: Percent traffic to champion (remainder goes to challenger)
    """
    
    print(f"Setting up A/B test:")
    print(f"  Champion (v{champion_version}): {traffic_split}%")
    print(f"  Challenger (v{challenger_version}): {100-traffic_split}%")
    
    # Update endpoint configuration
    w.serving_endpoints.update_config(
        name=endpoint_name,
        served_entities=[
            ServedEntityInput(
                entity_name="customer_churn_classifier",
                entity_version=str(champion_version),
                scale_to_zero_enabled=True,
                workload_size="Small",
                name="champion"
            ),
            ServedEntityInput(
                entity_name="customer_churn_classifier",
                entity_version=str(challenger_version),
                scale_to_zero_enabled=True,
                workload_size="Small",
                name="challenger"
            )
        ],
        traffic_config=TrafficConfig(
            routes=[
                Route(served_model_name="champion", traffic_percentage=traffic_split),
                Route(served_model_name="challenger", traffic_percentage=100-traffic_split)
            ]
        )
    )
    
    print("✅ A/B test configured")

# Set up A/B test
setup_ab_test(
    endpoint_name="customer-churn-classifier-endpoint",
    champion_version=3,
    challenger_version=4,
    traffic_split=90  # 90% champion, 10% challenger
)

# Monitor A/B test results
def analyze_ab_test(endpoint_name, days=7):
    """Analyze A/B test performance"""
    
    query = f"""
    SELECT 
        served_model_name as model_version,
        COUNT(*) as request_count,
        AVG(execution_time_ms) as avg_latency_ms,
        PERCENTILE(execution_time_ms, 0.95) as p95_latency_ms,
        PERCENTILE(execution_time_ms, 0.99) as p99_latency_ms,
        AVG(CAST(prediction as DOUBLE)) as avg_prediction,
        COUNT(CASE WHEN status_code = 200 THEN 1 END) * 100.0 / COUNT(*) as success_rate,
        SUM(CASE WHEN execution_time_ms > 1000 THEN 1 END) as slow_requests
    FROM system.serving.inference_logs
    WHERE endpoint_name = '{endpoint_name}'
      AND timestamp >= CURRENT_TIMESTAMP() - INTERVAL {days} DAYS
    GROUP BY served_model_name
    ORDER BY model_version
    """
    
    results = spark.sql(query)
    results.display()
    
    return results

# Analyze results
ab_results = analyze_ab_test("customer-churn-classifier-endpoint", days=7)

# Decide on promotion
def promote_challenger_if_better(ab_results, latency_threshold_ms=500):
    """Promote challenger to 100% if it performs better"""
    
    results_pd = ab_results.toPandas()
    
    if len(results_pd) == 2:
        champion = results_pd[results_pd['model_version'] == 'champion'].iloc[0]
        challenger = results_pd[results_pd['model_version'] == 'challenger'].iloc[0]
        
        # Compare metrics
        latency_improved = challenger['avg_latency_ms'] < champion['avg_latency_ms']
        success_rate_ok = challenger['success_rate'] >= 99.0
        latency_acceptable = challenger['p95_latency_ms'] < latency_threshold_ms
        
        if latency_improved and success_rate_ok and latency_acceptable:
            print("✅ Challenger performs better! Promoting to 100% traffic...")
            
            w.serving_endpoints.update_config(
                name="customer-churn-classifier-endpoint",
                traffic_config=TrafficConfig(
                    routes=[
                        Route(served_model_name="challenger", traffic_percentage=100)
                    ]
                )
            )
            
            print("✅ Challenger now receiving 100% traffic")
            return True
        else:
            print("⚠️ Challenger not performing better. Keeping champion.")
            return False
    
    return False

# Promote if better
promoted = promote_challenger_if_better(ab_results)
```

### Example 5: Model Monitoring and Drift Detection

```python
from pyspark.sql.functions import col, mean, stddev, min as spark_min, max as spark_max, count
from scipy import stats
import numpy as np

class ModelMonitor:
    """
    Monitor deployed models for performance and drift
    """
    
    def __init__(self, model_name, endpoint_name):
        self.model_name = model_name
        self.endpoint_name = endpoint_name
    
    def check_input_drift(self, current_data, reference_data, threshold=0.1):
        """
        Detect statistical drift in input features
        
        Uses Kolmogorov-Smirnov test for each feature
        """
        
        drift_results = []
        
        for column in current_data.columns:
            if column in reference_data.columns and column not in ['id', 'timestamp']:
                # Get distributions
                current_vals = current_data.select(column).toPandas()[column].dropna()
                reference_vals = reference_data.select(column).toPandas()[column].dropna()
                
                # KS test
                ks_statistic, p_value = stats.ks_2samp(current_vals, reference_vals)
                
                # Calculate mean drift
                current_mean = current_vals.mean()
                reference_mean = reference_vals.mean()
                reference_std = reference_vals.std()
                
                mean_drift = abs(current_mean - reference_mean) / (reference_std + 1e-10)
                
                drift_detected = p_value < 0.05  # Significant drift
                
                drift_results.append({
                    "feature": column,
                    "ks_statistic": ks_statistic,
                    "p_value": p_value,
                    "mean_drift": mean_drift,
                    "drift_detected": drift_detected,
                    "current_mean": current_mean,
                    "reference_mean": reference_mean
                })
        
        return pd.DataFrame(drift_results)
    
    def check_prediction_drift(self, endpoint_name, days=7):
        """Monitor prediction distribution changes"""
        
        query = f"""
        WITH daily_predictions AS (
            SELECT 
                DATE(timestamp) as date,
                AVG(CAST(prediction as DOUBLE)) as avg_prediction,
                STDDEV(CAST(prediction as DOUBLE)) as std_prediction,
                COUNT(*) as prediction_count
            FROM system.serving.inference_logs
            WHERE endpoint_name = '{endpoint_name}'
              AND timestamp >= CURRENT_DATE() - INTERVAL {days} DAYS
            GROUP BY DATE(timestamp)
        )
        SELECT 
            *,
            avg_prediction - LAG(avg_prediction) OVER (ORDER BY date) as prediction_drift
        FROM daily_predictions
        ORDER BY date DESC
        """
        
        return spark.sql(query)
    
    def monitor_performance_metrics(self, endpoint_name, days=30):
        """Monitor serving endpoint performance"""
        
        query = f"""
        SELECT 
            DATE(timestamp) as date,
            COUNT(*) as total_requests,
            AVG(execution_time_ms) as avg_latency,
            PERCENTILE(execution_time_ms, 0.50) as p50_latency,
            PERCENTILE(execution_time_ms, 0.95) as p95_latency,
            PERCENTILE(execution_time_ms, 0.99) as p99_latency,
            COUNT(CASE WHEN status_code = 200 THEN 1 END) * 100.0 / COUNT(*) as success_rate,
            COUNT(CASE WHEN execution_time_ms > 1000 THEN 1 END) as slow_requests,
            COUNT(CASE WHEN status_code >= 400 THEN 1 END) as error_count
        FROM system.serving.inference_logs
        WHERE endpoint_name = '{endpoint_name}'
          AND timestamp >= CURRENT_DATE() - INTERVAL {days} DAYS
        GROUP BY DATE(timestamp)
        ORDER BY date DESC
        """
        
        return spark.sql(query)
    
    def create_alerts(self, metrics):
        """Generate alerts based on performance metrics"""
        
        alerts = []
        
        latest = metrics.collect()[0]
        
        # Latency alerts
        if latest.p95_latency > 2000:
            alerts.append({
                "severity": "WARNING",
                "metric": "p95_latency",
                "value": latest.p95_latency,
                "threshold": 2000,
                "message": f"P95 latency ({latest.p95_latency:.0f}ms) exceeds threshold (2000ms)"
            })
        
        if latest.p99_latency > 5000:
            alerts.append({
                "severity": "CRITICAL",
                "metric": "p99_latency",
                "value": latest.p99_latency,
                "threshold": 5000,
                "message": f"P99 latency ({latest.p99_latency:.0f}ms) exceeds threshold (5000ms)"
            })
        
        # Success rate alerts
        if latest.success_rate < 99.5:
            alerts.append({
                "severity": "CRITICAL",
                "metric": "success_rate",
                "value": latest.success_rate,
                "threshold": 99.5,
                "message": f"Success rate ({latest.success_rate:.2f}%) below threshold (99.5%)"
            })
        
        # Error count alerts
        if latest.error_count > 100:
            alerts.append({
                "severity": "WARNING",
                "metric": "error_count",
                "value": latest.error_count,
                "threshold": 100,
                "message": f"High error count ({latest.error_count}) exceeds threshold (100)"
            })
        
        return alerts
    
    def send_alert(self, alert):
        """Send alert notification (implement with your alerting system)"""
        print(f"🚨 {alert['severity']}: {alert['message']}")
        # Integrate with Slack, PagerDuty, email, etc.

# Usage
monitor = ModelMonitor("customer_churn_classifier", "customer-churn-classifier-endpoint")

# Check input drift
current_data = spark.table("catalog.ml.current_week_customers")
reference_data = spark.table("catalog.ml.training_customers")

drift_report = monitor.check_input_drift(current_data, reference_data)
drift_report.display()

# Check prediction drift
prediction_drift = monitor.check_prediction_drift("customer-churn-classifier-endpoint", days=14)
prediction_drift.display()

# Monitor performance
performance_metrics = monitor.monitor_performance_metrics("customer-churn-classifier-endpoint", days=30)
performance_metrics.display()

# Generate and send alerts
alerts = monitor.create_alerts(performance_metrics)
for alert in alerts:
    monitor.send_alert(alert)
```

---

## Best Practices

### 1. Model Registry
* Version all models systematically
* Use stage transitions (None → Staging → Production)
* Document model changes and performance
* Archive old versions but keep for rollback
* Use aliases for stable references

### 2. Serving Endpoints
* Enable auto-scaling for variable load
* Monitor latency and throughput continuously
* Implement proper authentication
* Use appropriate instance sizes (GPU for deep learning)
* Set rate limits to prevent abuse

### 3. Batch Inference
* Schedule during off-peak hours
* Partition large datasets for parallelism
* Monitor job duration and failures
* Handle prediction errors gracefully
* Always log prediction timestamps

### 4. A/B Testing
* Start with small traffic split (10%)
* Monitor for at least 1 week
* Compare latency, accuracy, and business metrics
* Have rollback plan ready
* Document test results

### 5. Monitoring
* Monitor input drift weekly
* Track prediction distributions daily
* Set up performance alerts (latency, errors)
* Log all predictions for debugging
* Review metrics with stakeholders monthly

---

## Related Skills

* [MLflow Tracking](../mlflow-tracking/SKILL.md) - For experiment tracking
* [Feature Engineering](../feature-engineering/SKILL.md) - For feature preparation
* [Monitoring Observability](../monitoring-observability/SKILL.md) - For system monitoring
* [Error Handling](../error-handling/SKILL.md) - For production error handling

---

**Last Updated**: 2024-04-09
