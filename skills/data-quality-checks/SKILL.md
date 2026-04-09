# Data Quality Checks Skill

## Purpose
This skill teaches Genie Code how to implement comprehensive data quality validation patterns for data pipelines on Databricks. It covers validation strategies, quality metrics, anomaly detection, and automated data quality frameworks.

## When to Use
* Building data validation into ETL pipelines
* Implementing data quality gates between pipeline stages
* Creating data quality dashboards and monitoring
* Detecting and handling data anomalies
* Implementing SLAs for data freshness and completeness
* Building automated data quality frameworks with Delta Live Tables expectations

## Key Concepts

### 1. Data Quality Dimensions
* **Completeness**: No missing required fields, expected row counts
* **Accuracy**: Values within expected ranges, business rule compliance
* **Consistency**: Referential integrity, cross-field validation
* **Timeliness**: Data freshness, SLA compliance
* **Validity**: Correct data types, format compliance
* **Uniqueness**: No duplicates where uniqueness is required

### 2. Validation Layers
* **Bronze**: Schema validation, basic format checks, quarantine bad records
* **Silver**: Business rule validation, referential integrity, deduplication
* **Gold**: Aggregation validation, historical consistency checks

### 3. Quality Metrics
Track quality KPIs over time:
* Null percentage by column
* Duplicate record percentage
* Outlier counts
* Schema drift events
* Data arrival delays
* Quality check pass/fail rates

---

## Examples

### Example 1: Basic Data Quality Checks with PySpark

```python
from pyspark.sql import DataFrame
from pyspark.sql.functions import col, count, when, sum as spark_sum, isnan, isnull
from pyspark.sql.types import *
from typing import Dict, List

class DataQualityValidator:
    """
    Comprehensive data quality validation framework
    """
    
    def __init__(self, df: DataFrame, table_name: str):
        self.df = df
        self.table_name = table_name
        self.quality_results = []
    
    def check_completeness(self, required_columns: List[str]) -> Dict:
        """
        Check for null values in required columns
        """
        total_records = self.df.count()
        completeness_results = {}
        
        for col_name in required_columns:
            null_count = self.df.filter(col(col_name).isNull()).count()
            null_percentage = (null_count / total_records) * 100
            
            completeness_results[col_name] = {
                'null_count': null_count,
                'null_percentage': round(null_percentage, 2),
                'status': 'PASS' if null_percentage < 1 else 'FAIL'
            }
        
        return completeness_results
    
    def check_value_ranges(self, range_rules: Dict[str, tuple]) -> Dict:
        """
        Validate values are within expected ranges
        
        Args:
            range_rules: {column_name: (min_value, max_value)}
        """
        range_results = {}
        
        for col_name, (min_val, max_val) in range_rules.items():
            out_of_range = self.df.filter(
                (col(col_name) < min_val) | (col(col_name) > max_val)
            ).count()
            
            range_results[col_name] = {
                'out_of_range_count': out_of_range,
                'min_expected': min_val,
                'max_expected': max_val,
                'status': 'PASS' if out_of_range == 0 else 'FAIL'
            }
        
        return range_results
    
    def check_uniqueness(self, unique_columns: List[str]) -> Dict:
        """
        Check for duplicate records based on key columns
        """
        total_records = self.df.count()
        distinct_records = self.df.select(unique_columns).distinct().count()
        duplicates = total_records - distinct_records
        duplicate_percentage = (duplicates / total_records) * 100
        
        return {
            'total_records': total_records,
            'distinct_records': distinct_records,
            'duplicate_count': duplicates,
            'duplicate_percentage': round(duplicate_percentage, 2),
            'status': 'PASS' if duplicates == 0 else 'FAIL'
        }
    
    def check_referential_integrity(
        self, 
        foreign_key: str, 
        reference_df: DataFrame, 
        reference_key: str
    ) -> Dict:
        """
        Check if foreign keys exist in reference table
        """
        # Get distinct foreign keys from main table
        fk_values = self.df.select(foreign_key).distinct()
        
        # Left anti join to find orphaned records
        orphaned = fk_values.join(
            reference_df.select(reference_key),
            fk_values[foreign_key] == reference_df[reference_key],
            'left_anti'
        ).count()
        
        return {
            'foreign_key': foreign_key,
            'reference_key': reference_key,
            'orphaned_records': orphaned,
            'status': 'PASS' if orphaned == 0 else 'FAIL'
        }
    
    def check_data_freshness(self, timestamp_column: str, max_age_hours: int) -> Dict:
        """
        Check if data is fresh enough based on timestamp
        """
        from pyspark.sql.functions import max as spark_max, current_timestamp, hour
        
        latest_timestamp = self.df.agg(spark_max(timestamp_column)).collect()[0][0]
        
        if latest_timestamp:
            hours_old = (
                self.df.select(
                    ((current_timestamp().cast("long") - col(timestamp_column).cast("long")) / 3600)
                ).agg({"((((CAST(current_timestamp() AS BIGINT) - CAST({} AS BIGINT)) / 3600))".format(timestamp_column): "min"}).collect()[0][0]
            )
            
            return {
                'latest_timestamp': latest_timestamp,
                'hours_old': round(hours_old, 2),
                'max_age_hours': max_age_hours,
                'status': 'PASS' if hours_old <= max_age_hours else 'FAIL'
            }
        else:
            return {
                'status': 'FAIL',
                'error': 'No timestamp data found'
            }
    
    def generate_quality_report(self) -> DataFrame:
        """
        Generate a quality report as DataFrame for storage
        """
        from pyspark.sql.functions import lit, current_timestamp
        
        # Calculate column-level metrics
        column_metrics = []
        
        for column in self.df.columns:
            null_count = self.df.filter(col(column).isNull()).count()
            total_count = self.df.count()
            
            column_metrics.append({
                'table_name': self.table_name,
                'column_name': column,
                'null_count': null_count,
                'null_percentage': round((null_count / total_count) * 100, 2),
                'check_timestamp': None  # Will be filled with current_timestamp
            })
        
        # Create DataFrame from metrics
        quality_df = spark.createDataFrame(column_metrics)
        quality_df = quality_df.withColumn('check_timestamp', current_timestamp())
        
        return quality_df


# Usage Example
df = spark.table("catalog.schema.transactions")
validator = DataQualityValidator(df, "transactions")

# Check completeness
completeness = validator.check_completeness(['customer_id', 'transaction_date', 'amount'])
print("Completeness Check:", completeness)

# Check value ranges
ranges = validator.check_value_ranges({
    'amount': (0, 1000000),
    'quantity': (1, 1000)
})
print("Range Check:", ranges)

# Check uniqueness
uniqueness = validator.check_uniqueness(['transaction_id'])
print("Uniqueness Check:", uniqueness)

# Generate and save quality report
quality_report = validator.generate_quality_report()
quality_report.write.mode("append").saveAsTable("catalog.monitoring.data_quality_metrics")
```

### Example 2: Delta Live Tables with Expectations

```python
import dlt
from pyspark.sql.functions import col, current_timestamp

# Bronze table with basic validation
@dlt.table(
    name="transactions_bronze",
    comment="Raw transaction data with schema validation"
)
@dlt.expect_all_or_drop({
    "valid_transaction_id": "transaction_id IS NOT NULL",
    "valid_amount": "amount >= 0",
    "valid_date": "transaction_date IS NOT NULL"
})
def transactions_bronze():
    return (
        spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .option("cloudFiles.schemaLocation", "/mnt/schema/transactions")
        .load("/mnt/raw/transactions/")
        .withColumn("ingestion_timestamp", current_timestamp())
    )


# Silver table with strict validation
@dlt.table(
    name="transactions_silver",
    comment="Validated and deduplicated transactions"
)
@dlt.expect_all({
    "valid_customer": "customer_id IS NOT NULL",
    "valid_amount_range": "amount BETWEEN 0 AND 1000000",
    "valid_currency": "currency IN ('USD', 'EUR', 'GBP')",
    "no_future_dates": "transaction_date <= current_date()"
})
@dlt.expect_or_fail("no_duplicates", "_row_number = 1")
def transactions_silver():
    from pyspark.sql.window import Window
    from pyspark.sql.functions import row_number
    
    bronze_df = dlt.read_stream("transactions_bronze")
    
    # Add row number for deduplication
    window = Window.partitionBy("transaction_id").orderBy(col("ingestion_timestamp").desc())
    
    return (
        bronze_df
        .withColumn("_row_number", row_number().over(window))
        .filter(col("_row_number") == 1)
        .drop("_row_number")
    )


# Gold table with business rules
@dlt.table(
    name="daily_transaction_summary",
    comment="Daily transaction aggregates with quality checks"
)
@dlt.expect_all_or_fail({
    "reasonable_daily_total": "total_amount < 10000000",
    "reasonable_count": "transaction_count > 0 AND transaction_count < 1000000",
    "no_negative_totals": "total_amount >= 0"
})
def daily_transaction_summary():
    return (
        dlt.read("transactions_silver")
        .groupBy("transaction_date", "currency")
        .agg(
            count("*").alias("transaction_count"),
            sum("amount").alias("total_amount"),
            avg("amount").alias("avg_amount")
        )
    )
```

### Example 3: Custom Quality Metrics Table

```python
from pyspark.sql.functions import current_timestamp, lit
from delta.tables import DeltaTable

def calculate_and_store_quality_metrics(
    source_table: str,
    target_catalog: str = "monitoring",
    target_schema: str = "data_quality"
):
    """
    Calculate comprehensive quality metrics and store in monitoring table
    """
    
    # Read source data
    df = spark.table(source_table)
    total_rows = df.count()
    
    # Create quality metrics table if not exists
    metrics_table = f"{target_catalog}.{target_schema}.quality_metrics"
    
    spark.sql(f"""
        CREATE TABLE IF NOT EXISTS {metrics_table} (
            table_name STRING,
            metric_name STRING,
            metric_value DOUBLE,
            threshold DOUBLE,
            status STRING,
            details STRING,
            check_timestamp TIMESTAMP,
            job_id STRING
        )
        USING DELTA
        PARTITIONED BY (DATE(check_timestamp))
    """)
    
    # Calculate metrics
    metrics = []
    
    # 1. Row count metric
    metrics.append({
        'table_name': source_table,
        'metric_name': 'row_count',
        'metric_value': float(total_rows),
        'threshold': 0.0,
        'status': 'PASS' if total_rows > 0 else 'FAIL',
        'details': f'Total rows: {total_rows}'
    })
    
    # 2. Null percentage per column
    for column in df.columns:
        null_count = df.filter(col(column).isNull()).count()
        null_pct = (null_count / total_rows * 100) if total_rows > 0 else 0
        
        metrics.append({
            'table_name': source_table,
            'metric_name': f'null_percentage_{column}',
            'metric_value': round(null_pct, 2),
            'threshold': 5.0,  # 5% threshold
            'status': 'PASS' if null_pct < 5.0 else 'WARN',
            'details': f'Column: {column}, Null count: {null_count}'
        })
    
    # 3. Duplicate percentage (if there's an ID column)
    id_columns = [c for c in df.columns if 'id' in c.lower()]
    if id_columns:
        id_col = id_columns[0]
        distinct_count = df.select(id_col).distinct().count()
        duplicate_pct = ((total_rows - distinct_count) / total_rows * 100) if total_rows > 0 else 0
        
        metrics.append({
            'table_name': source_table,
            'metric_name': f'duplicate_percentage_{id_col}',
            'metric_value': round(duplicate_pct, 2),
            'threshold': 1.0,  # 1% threshold
            'status': 'PASS' if duplicate_pct < 1.0 else 'FAIL',
            'details': f'Duplicate records: {total_rows - distinct_count}'
        })
    
    # Create DataFrame and write metrics
    metrics_df = spark.createDataFrame(metrics)
    metrics_df = (
        metrics_df
        .withColumn('check_timestamp', current_timestamp())
        .withColumn('job_id', lit(dbutils.notebook.entry_point.getDbutils().notebook().getContext().jobId().getOrElse(None)))
    )
    
    # Append to metrics table
    metrics_df.write.mode("append").saveAsTable(metrics_table)
    
    # Return summary
    failed_metrics = metrics_df.filter(col('status') == 'FAIL').count()
    
    return {
        'metrics_calculated': len(metrics),
        'failed_checks': failed_metrics,
        'status': 'PASS' if failed_metrics == 0 else 'FAIL'
    }


# Usage
result = calculate_and_store_quality_metrics("prod_analytics.silver.customer_orders")
print(result)
```

### Example 4: Anomaly Detection

```python
from pyspark.sql.functions import stddev, avg, col
from pyspark.sql.window import Window

def detect_anomalies(
    df: DataFrame,
    metric_column: str,
    group_by_columns: List[str],
    std_threshold: float = 3.0
) -> DataFrame:
    """
    Detect anomalies using z-score method
    
    Args:
        df: Input DataFrame
        metric_column: Column to check for anomalies
        group_by_columns: Columns to group by for calculating statistics
        std_threshold: Number of standard deviations to consider as anomaly
    """
    
    # Calculate statistics per group
    stats_df = df.groupBy(group_by_columns).agg(
        avg(metric_column).alias('mean_value'),
        stddev(metric_column).alias('std_value')
    )
    
    # Join back to original data
    result_df = (
        df.join(stats_df, group_by_columns)
        .withColumn(
            'z_score',
            (col(metric_column) - col('mean_value')) / col('std_value')
        )
        .withColumn(
            'is_anomaly',
            when(col('z_score').abs() > std_threshold, True).otherwise(False)
        )
    )
    
    return result_df


# Example: Detect anomalous transaction amounts by merchant
transactions_df = spark.table("catalog.schema.transactions")

anomalies = detect_anomalies(
    transactions_df,
    metric_column='amount',
    group_by_columns=['merchant_id', 'transaction_date'],
    std_threshold=3.0
)

# Filter and review anomalies
anomalous_transactions = anomalies.filter(col('is_anomaly'))
anomalous_transactions.write.mode("overwrite").saveAsTable("catalog.monitoring.anomalous_transactions")
```

---

## Best Practices

### 1. Layered Validation Strategy
* **Bronze**: Reject only malformed data, quarantine for review
* **Silver**: Enforce business rules, fail on critical violations
* **Gold**: Validate aggregations and consistency

### 2. Fail Fast vs. Warn
* **Fail**: Critical issues that should stop pipeline (missing keys, schema changes)
* **Warn**: Quality degradation that needs attention but shouldn't block (higher than usual nulls)
* **Quarantine**: Isolate bad records for investigation without blocking pipeline

### 3. Store Quality Metrics
* Create dedicated monitoring schema
* Track metrics over time to detect trends
* Alert on metric threshold violations
* Use metrics for SLA tracking

### 4. Implement Quality Gates
* Define quality thresholds per table/pipeline
* Automate pass/fail decisions
* Integrate with workflow orchestration
* Document quality requirements

### 5. Balance Performance vs. Thoroughness
* Run expensive checks (like full deduplication) less frequently
* Use sampling for exploratory quality checks
* Cache DataFrames if running multiple validations
* Optimize validation queries with proper partitioning

---

## Common Pitfalls

### ❌ Collecting Large DataFrames
* **Problem**: Using `.collect()` for counting on large datasets
* **Solution**: Use `.count()` or aggregate functions directly

### ❌ No Historical Tracking
* **Problem**: Only checking current quality, no trend analysis
* **Solution**: Store metrics with timestamps for historical analysis

### ❌ Blocking Pipelines on Minor Issues
* **Problem**: Failing entire pipeline on non-critical quality issues
* **Solution**: Implement severity levels (FAIL/WARN/INFO)

### ❌ Insufficient Metadata
* **Problem**: Quality failures without context about data source or timing
* **Solution**: Include job_id, timestamp, source info in quality logs

### ❌ One-Size-Fits-All Validation
* **Problem**: Same validation rules for all tables
* **Solution**: Customize validation based on table purpose and criticality

---

## Related Skills

* [Delta Lake Optimization](../delta-lake-optimization/SKILL.md) - For optimizing quality check performance
* [Medallion Architecture](../../governance/medallion-architecture/SKILL.md) - For layered validation approach
* [Streaming Pipelines](../streaming-pipelines/SKILL.md) - For quality checks in streaming context

---

## Integration with Databricks Features

### Unity Catalog
* Use table comments to document quality expectations
* Tag tables with quality tier (gold/silver/bronze)
* Track lineage to understand quality impact

### Delta Live Tables
* Use expectations for declarative quality rules
* Leverage automatic quarantine tables
* Monitor quality metrics in pipeline UI

### Workflows
* Add quality check tasks before critical downstream steps
* Use task values to pass quality metrics between tasks
* Implement conditional execution based on quality results

---

**Last Updated**: 2024-04-09
