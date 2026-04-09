# Delta Lake Optimization Skill

## Purpose
This skill teaches Genie Code comprehensive Delta Lake optimization techniques including OPTIMIZE, Z-ORDER, VACUUM, liquid clustering, and performance tuning strategies for large-scale Delta tables on Databricks.

## When to Use
* Optimizing Delta tables with small file problems
* Improving query performance on large Delta tables
* Managing storage costs and retention
* Implementing maintenance schedules
* Migrating to liquid clustering
* Troubleshooting slow Delta table reads/writes
* Designing partition strategies

## Key Concepts

### 1. Small File Problem
* Each file has overhead for metadata and I/O operations
* Too many small files degrade read performance
* Target file size: 128MB - 1GB per file
* Caused by: streaming writes, frequent small batches, high-cardinality partitions

### 2. OPTIMIZE Command
* Compacts small files into larger files
* Reduces metadata overhead
* Improves read performance significantly
* Can be combined with Z-ORDER
* Supported modes: Full table or partition-level

### 3. Z-ORDER Clustering
* Co-locates related data in files using space-filling curves
* Improves data skipping effectiveness
* Best for columns frequently used in WHERE/JOIN clauses
* Effective for multiple columns (up to 4 recommended)
* Different from traditional partitioning

### 4. Liquid Clustering (DBR 13.3+)
* Next-generation clustering technique
* Replaces both partitioning and Z-ORDER
* Automatic incremental clustering
* Supports changing clustering keys without rewriting
* Better for high-cardinality columns

### 5. VACUUM
* Removes old file versions to save storage
* Required for GDPR/compliance (permanent deletion)
* Default retention: 7 days
* Cannot undo after completion


### 6. Predictive Optimization (2023+)
* **Automated optimization** without manual OPTIMIZE commands
* **Predictive I/O** learns from query patterns
* **Auto-compaction** runs in the background
* **Intelligent VACUUM** with safe retention management
* **Per-table or per-catalog enablement**
* **Serverless compute integration** - automatic on serverless
* Eliminates maintenance overhead

---

## Examples

### Example 1: Basic OPTIMIZE and Z-ORDER

```python
from pyspark.sql.functions import col, count

# Analyze table file statistics before optimization
def analyze_table_files(table_name: str):
    """
    Analyze file size distribution and count
    """
    files_df = spark.sql(f"DESCRIBE DETAIL {table_name}")
    num_files = files_df.select("numFiles").collect()[0][0]
    size_in_bytes = files_df.select("sizeInBytes").collect()[0][0]
    
    avg_file_size_mb = (size_in_bytes / num_files / 1024 / 1024) if num_files > 0 else 0
    
    print(f"Table: {table_name}")
    print(f"Number of files: {num_files}")
    print(f"Total size: {size_in_bytes / 1024 / 1024:.2f} MB")
    print(f"Average file size: {avg_file_size_mb:.2f} MB")
    
    return {
        'num_files': num_files,
        'size_mb': size_in_bytes / 1024 / 1024,
        'avg_file_size_mb': avg_file_size_mb
    }


# Before optimization
stats_before = analyze_table_files("catalog.schema.customer_transactions")

# OPTIMIZE without Z-ORDER (simple compaction)
spark.sql("""
    OPTIMIZE catalog.schema.customer_transactions
""")

# OPTIMIZE with Z-ORDER for frequently queried columns
spark.sql("""
    OPTIMIZE catalog.schema.customer_transactions
    ZORDER BY (customer_id, transaction_date)
""")

# After optimization
stats_after = analyze_table_files("catalog.schema.customer_transactions")

print(f"File reduction: {stats_before['num_files'] - stats_after['num_files']} files")
print(f"Space savings: {stats_before['size_mb'] - stats_after['size_mb']:.2f} MB")
```

### Example 2: Partition-Level Optimization

```python
from datetime import datetime, timedelta

def optimize_recent_partitions(
    table_name: str,
    partition_column: str,
    days_back: int = 7,
    zorder_columns: list = None
):
    """
    Optimize only recent partitions to save time and resources
    """
    
    # Calculate date range
    end_date = datetime.now().date()
    start_date = end_date - timedelta(days=days_back)
    
    # Build OPTIMIZE command
    optimize_cmd = f"OPTIMIZE {table_name}"
    
    # Add WHERE clause for partition filter
    optimize_cmd += f" WHERE {partition_column} >= '{start_date}' AND {partition_column} <= '{end_date}'"
    
    # Add Z-ORDER if specified
    if zorder_columns:
        zorder_cols = ', '.join(zorder_columns)
        optimize_cmd += f" ZORDER BY ({zorder_cols})"
    
    print(f"Executing: {optimize_cmd}")
    
    # Run optimization
    result = spark.sql(optimize_cmd)
    
    # Parse results
    metrics = result.collect()[0]
    print(f"Files added: {metrics['num_added_files']}")
    print(f"Files removed: {metrics['num_removed_files']}")
    print(f"Partitions optimized: {metrics['num_affected_partitions']}")
    
    return result


# Usage: Optimize last 7 days of transaction data
optimize_recent_partitions(
    table_name="catalog.schema.transactions",
    partition_column="transaction_date",
    days_back=7,
    zorder_columns=["customer_id", "merchant_id"]
)
```

### Example 3: Auto-Optimize Configuration

```python
# Enable Auto Optimize for new table
spark.sql("""
    CREATE TABLE catalog.schema.high_frequency_writes (
        id BIGINT,
        event_time TIMESTAMP,
        user_id STRING,
        event_type STRING,
        payload STRING
    )
    USING DELTA
    PARTITIONED BY (DATE(event_time))
    TBLPROPERTIES (
        'delta.autoOptimize.optimizeWrite' = 'true',
        'delta.autoOptimize.autoCompact' = 'true'
    )
""")

# Enable Auto Optimize for existing table
spark.sql("""
    ALTER TABLE catalog.schema.existing_table
    SET TBLPROPERTIES (
        'delta.autoOptimize.optimizeWrite' = 'true',
        'delta.autoOptimize.autoCompact' = 'true'
    )
""")

# Check current Auto Optimize settings
def check_auto_optimize(table_name: str):
    """
    Check if Auto Optimize is enabled
    """
    properties_df = spark.sql(f"SHOW TBLPROPERTIES {table_name}")
    
    optimize_write = properties_df.filter(
        col("key") == "delta.autoOptimize.optimizeWrite"
    ).select("value").collect()
    
    auto_compact = properties_df.filter(
        col("key") == "delta.autoOptimize.autoCompact"
    ).select("value").collect()
    
    return {
        'optimizeWrite': optimize_write[0][0] if optimize_write else 'false',
        'autoCompact': auto_compact[0][0] if auto_compact else 'false'
    }


settings = check_auto_optimize("catalog.schema.high_frequency_writes")
print(f"Auto Optimize settings: {settings}")
```

### Example 4: VACUUM for Storage Management

```python
from delta.tables import DeltaTable

def vacuum_table_with_safety_checks(
    table_name: str,
    retention_hours: int = 168  # 7 days default
):
    """
    VACUUM table with safety checks and monitoring
    """
    
    # Get table details
    detail_df = spark.sql(f"DESCRIBE DETAIL {table_name}")
    location = detail_df.select("location").collect()[0][0]
    size_before = detail_df.select("sizeInBytes").collect()[0][0]
    
    print(f"Table: {table_name}")
    print(f"Location: {location}")
    print(f"Size before VACUUM: {size_before / 1024 / 1024:.2f} MB")
    print(f"Retention: {retention_hours} hours ({retention_hours / 24:.1f} days)")
    
    # Safety check: ensure retention is at least 7 days for production
    if retention_hours < 168:
        print("⚠️  WARNING: Retention less than 7 days may break time travel!")
        response = input("Continue? (yes/no): ")
        if response.lower() != 'yes':
            return None
    
    # Perform dry run first
    print("\n🔍 Performing dry run...")
    delta_table = DeltaTable.forName(spark, table_name)
    files_to_delete = delta_table.vacuum(retention_hours, dry_run=True)
    
    print(f"Files to be deleted: {files_to_delete.count()}")
    
    # Execute VACUUM
    print("\n🧹 Executing VACUUM...")
    result = delta_table.vacuum(retention_hours)
    
    # Check size after
    detail_df_after = spark.sql(f"DESCRIBE DETAIL {table_name}")
    size_after = detail_df_after.select("sizeInBytes").collect()[0][0]
    
    space_freed = size_before - size_after
    print(f"\n✅ VACUUM completed")
    print(f"Size after VACUUM: {size_after / 1024 / 1024:.2f} MB")
    print(f"Space freed: {space_freed / 1024 / 1024:.2f} MB ({space_freed / size_before * 100:.1f}%)")
    
    return {
        'size_before_mb': size_before / 1024 / 1024,
        'size_after_mb': size_after / 1024 / 1024,
        'space_freed_mb': space_freed / 1024 / 1024,
        'space_freed_pct': space_freed / size_before * 100
    }


# Usage
vacuum_result = vacuum_table_with_safety_checks(
    "catalog.schema.transactions",
    retention_hours=168  # 7 days
)
```

### Example 5: Liquid Clustering (DBR 13.3+)

```python
# Create table with liquid clustering
spark.sql("""
    CREATE TABLE catalog.schema.events_clustered (
        event_id STRING,
        user_id STRING,
        event_timestamp TIMESTAMP,
        event_type STRING,
        country STRING,
        device_type STRING
    )
    USING DELTA
    CLUSTER BY (event_type, country, user_id)
""")

# Convert existing table to liquid clustering
spark.sql("""
    ALTER TABLE catalog.schema.existing_events
    CLUSTER BY (event_type, country, user_id)
""")

# Change clustering columns (doesn't require full rewrite)
spark.sql("""
    ALTER TABLE catalog.schema.events_clustered
    CLUSTER BY (user_id, event_timestamp)
""")

# Optimize with liquid clustering (incremental clustering)
spark.sql("""
    OPTIMIZE catalog.schema.events_clustered
""")

# Check clustering information
def analyze_clustering(table_name: str):
    """
    Analyze liquid clustering metrics
    """
    # Get table properties
    properties_df = spark.sql(f"SHOW TBLPROPERTIES {table_name}")
    
    clustering_cols = properties_df.filter(
        col("key") == "delta.clustering.columns"
    ).select("value").collect()
    
    if clustering_cols:
        print(f"Clustering columns: {clustering_cols[0][0]}")
    else:
        print("Table is not using liquid clustering")
    
    # Get table details
    detail_df = spark.sql(f"DESCRIBE DETAIL {table_name}")
    detail_df.select("name", "numFiles", "sizeInBytes").show(truncate=False)


analyze_clustering("catalog.schema.events_clustered")
```

### Example 6: Optimization Scheduling

```python
from datetime import datetime

def create_optimization_schedule(
    catalog: str,
    schema: str,
    optimize_interval_days: int = 1,
    vacuum_interval_days: int = 7
):
    """
    Create Databricks job for regular Delta table maintenance
    """
    
    optimization_notebook = f"""
# Delta Optimization Notebook
# Scheduled maintenance for {catalog}.{schema}

from pyspark.sql.functions import col
from datetime import datetime
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def optimize_schema_tables(catalog_name, schema_name):
    # Get all Delta tables in schema
    tables = spark.sql(f"SHOW TABLES IN {{catalog_name}}.{{schema_name}}")
    
    results = []
    
    for row in tables.collect():
        table_name = f"{{catalog_name}}.{{schema_name}}.{{row['tableName']}}"
        
        try:
            logger.info(f"Optimizing {{table_name}}...")
            
            # Get table details
            detail = spark.sql(f"DESCRIBE DETAIL {{table_name}}").collect()[0]
            num_files_before = detail['numFiles']
            
            # Optimize with Z-ORDER if table has common filter columns
            # (This would need to be customized per table)
            spark.sql(f"OPTIMIZE {{table_name}}")
            
            # Get files after
            detail_after = spark.sql(f"DESCRIBE DETAIL {{table_name}}").collect()[0]
            num_files_after = detail_after['numFiles']
            
            results.append({{
                'table': table_name,
                'files_before': num_files_before,
                'files_after': num_files_after,
                'status': 'SUCCESS'
            }})
            
            logger.info(f"✓ {{table_name}}: {{num_files_before}} → {{num_files_after}} files")
            
        except Exception as e:
            logger.error(f"✗ Failed to optimize {{table_name}}: {{str(e)}}")
            results.append({{
                'table': table_name,
                'status': 'FAILED',
                'error': str(e)
            }})
    
    return results

# Run optimization
results = optimize_schema_tables("{catalog}", "{schema}")

# Summary
success_count = sum(1 for r in results if r['status'] == 'SUCCESS')
print(f"Optimization complete: {{success_count}}/{{len(results)}} tables optimized successfully")
"""
    
    return optimization_notebook


# Generate notebook code
notebook_code = create_optimization_schedule("prod_analytics", "silver")
print(notebook_code)
```

---


### Example 7: Predictive Optimization (Recommended for 2026)

```sql
-- Enable Predictive Optimization at catalog level
ALTER CATALOG prod_analytics
  SET TBLPROPERTIES ('delta.autoOptimize.optimizeWrite' = 'true',
                     'delta.autoOptimize.autoCompact' = 'true',
                     'delta.enablePredictiveIO' = 'true');

-- Enable for specific table
ALTER TABLE prod_analytics.silver.customer_events
  SET TBLPROPERTIES ('delta.autoOptimize.optimizeWrite' = 'true',
                     'delta.autoOptimize.autoCompact' = 'true');

-- Check Predictive Optimization status
DESCRIBE DETAIL prod_analytics.silver.customer_events;
```

```python
# Enable Predictive Optimization with Python
from pyspark.sql.functions import *

# For new table
(df
 .write
 .format("delta")
 .option("delta.autoOptimize.optimizeWrite", "true")
 .option("delta.autoOptimize.autoCompact", "true")
 .saveAsTable("prod_analytics.silver.customer_events"))

# For existing table
spark.sql("""
  ALTER TABLE prod_analytics.silver.customer_events
  SET TBLPROPERTIES (
    'delta.autoOptimize.optimizeWrite' = 'true',
    'delta.autoOptimize.autoCompact' = 'true',
    'delta.targetFileSize' = '128MB'
  )
""")

# Monitor auto-optimization activity
spark.sql("""
  SELECT 
    timestamp,
    operation,
    operationMetrics.*
  FROM (
    DESCRIBE HISTORY prod_analytics.silver.customer_events
  )
  WHERE operation IN ('OPTIMIZE', 'AUTO COMPACT')
  ORDER BY timestamp DESC
  LIMIT 20
""").display()

# Check table properties
spark.sql("""
  SHOW TBLPROPERTIES prod_analytics.silver.customer_events
""").display()
```

**Predictive Optimization Benefits:**
* ✅ No manual OPTIMIZE commands needed
* ✅ Automatic background compaction
* ✅ Learns from query patterns
* ✅ Reduces operational overhead
* ✅ Intelligent file sizing
* ✅ Works with liquid clustering

**When to Use:**
* Production tables with frequent writes
* Tables on serverless compute (automatic)
* Unity Catalog managed tables
* High-volume streaming tables
* Tables accessed by many concurrent queries

**Cost Considerations:**
* Runs on background compute (minimal cost)
* Automatic on serverless (included)
* Can reduce storage costs via auto-compaction
* Reduces manual job scheduling overhead

## Best Practices

### 1. OPTIMIZE Frequency
* **High-write tables**: Daily or after every large batch
* **Medium-write tables**: Weekly
* **Low-write tables**: Monthly or on-demand
* **Streaming tables**: Use Auto Optimize

### 2. Z-ORDER Column Selection
* Choose columns frequently used in WHERE clauses
* Prioritize high-cardinality columns (e.g., user_id, transaction_id)
* Limit to 3-4 columns maximum
* Avoid low-cardinality columns (e.g., boolean flags)
* Consider query patterns, not just column types

### 3. Partition Strategy
* Partition by date/timestamp for time-series data
* Avoid high-cardinality partition columns (>1000 partitions)
* Target 1GB+ per partition
* Consider liquid clustering instead for complex access patterns

### 4. VACUUM Retention
* **Production**: Minimum 7 days (default)
* **Compliance**: Adjust based on data retention policies
* **Time Travel**: Longer retention = longer history
* **Storage Costs**: Balance retention vs. storage costs

### 5. Auto Optimize Usage
* **Enable for**: Streaming workloads, high-frequency writes
* **Disable for**: Large batch jobs (optimize manually instead)
* **Monitor**: Can increase write latency slightly

### 6. Liquid Clustering (When to Use)
* Tables with multiple query patterns
* High-cardinality partition candidates
* Need to change clustering strategy over time
* DBR 13.3+ available

---

## Common Pitfalls

### ❌ Over-Optimizing
* **Problem**: Running OPTIMIZE too frequently on stable tables
* **Impact**: Wasted compute, unnecessary file rewrites
* **Solution**: Schedule based on write patterns, not arbitrary frequency

### ❌ Z-ORDER on Low-Cardinality Columns
* **Problem**: Ordering by status (3 values) or boolean flags
* **Impact**: Minimal benefit, wasted resources
* **Solution**: Choose high-cardinality columns with >1000 distinct values

### ❌ Aggressive VACUUM
* **Problem**: Setting retention < 7 days without understanding impact
* **Impact**: Breaks concurrent readers, limits time travel
* **Solution**: Keep 7+ day retention, understand query patterns

### ❌ Ignoring Small File Problem
* **Problem**: Letting small files accumulate indefinitely
* **Impact**: Severe read performance degradation
* **Solution**: Monitor file counts, set up regular optimization

### ❌ Too Many Partition Columns
* **Problem**: Partitioning by multiple dimensions (date, region, product)
* **Impact**: Partition explosion, worse performance
* **Solution**: Partition by one column, use Z-ORDER or liquid clustering for others

### ❌ Optimizing During Peak Hours
* **Problem**: Running heavy OPTIMIZE during business hours
* **Impact**: Resource contention, slow queries
* **Solution**: Schedule maintenance during off-peak hours

---

## Optimization Decision Tree

```
Is the table experiencing slow read performance?
├─ Yes → Check file count (DESCRIBE DETAIL)
│   ├─ >1000 files → Run OPTIMIZE
│   └─ <1000 files → Check query patterns
│       └─ Frequent WHERE clauses? → Add Z-ORDER
│
└─ No → Is it a write-heavy table?
    ├─ Yes → Enable Auto Optimize
    └─ No → Schedule periodic maintenance
```

---

## Performance Impact Examples

### Before Optimization
* Files: 5,000 small files (10MB each)
* Query time: 45 seconds
* Metadata overhead: High

### After OPTIMIZE
* Files: 50 files (1GB each)
* Query time: 8 seconds
* Improvement: **82% faster**

### After OPTIMIZE + Z-ORDER
* Files: 50 files (1GB each)
* Query time: 3 seconds
* Improvement: **93% faster**
* Data skipping: 90% of files skipped

---

## Monitoring Queries

```sql
-- Check table file statistics
DESCRIBE DETAIL catalog.schema.table_name;

-- Check table properties
SHOW TBLPROPERTIES catalog.schema.table_name;

-- View table history
DESCRIBE HISTORY catalog.schema.table_name;

-- Check partition sizes
SELECT 
    partition_column,
    COUNT(*) as file_count,
    SUM(size_in_bytes) / 1024 / 1024 as size_mb
FROM (
    SELECT 
        input_file_name() as file_name,
        partition_column,
        COUNT(*) as row_count
    FROM catalog.schema.table_name
    GROUP BY ALL
) 
GROUP BY partition_column
ORDER BY file_count DESC;
```

---

## Related Skills

* [Incremental Processing](../incremental-processing/SKILL.md) - For efficient write patterns
* [Spark Optimization](../../optimization/spark-optimization/SKILL.md) - For query-level optimization
* [Cost Optimization](../../governance/cost-optimization/SKILL.md) - For balancing performance vs. cost

---

## Integration with Databricks Features

### Delta Live Tables
* Automatically manages optimization for streaming tables
* Configure via pipeline settings
* Monitor via pipeline UI

### Databricks Workflows
* Schedule optimization jobs during off-peak hours
* Use task dependencies for multi-table optimization
* Monitor job metrics for optimization effectiveness

### Unity Catalog
* Track optimization history via table audit logs
* Document optimization strategy in table comments
* Use tags to identify tables needing frequent optimization

---

**Last Updated**: 2024-04-09
