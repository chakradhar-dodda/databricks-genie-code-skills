# Incremental Processing Skill

## Purpose
This skill teaches Genie Code efficient incremental data processing patterns for batch and streaming workloads on Databricks. It covers watermarking, Change Data Feed (CDF), checkpointing, and merge strategies to process only new or changed data.

## When to Use
* Processing only new/changed records instead of full reloads
* Building efficient ETL pipelines with large datasets
* Implementing CDC (Change Data Capture) patterns
* Handling late-arriving data in streaming
* Reducing processing time and costs
* Maintaining historical state with SCD Type 2
* Syncing data from source systems incrementally

## Key Concepts

### 1. Incremental Patterns
* **Append-only**: New records only, no updates (e.g., logs, events)
* **Upsert/Merge**: New and updated records (e.g., customer data)
* **SCD Type 2**: Track historical changes with versioning
* **Delete**: Handle deletions from source systems

### 2. Change Data Feed (CDF)
* Tracks row-level changes (insert, update, delete) in Delta tables
* Enables incremental processing of downstream consumers
* More efficient than full table scans
* Available in DBR 9.1+

### 3. Watermarking
* Handles late-arriving data in streaming
* Defines how long to wait for late records
* Enables stateful aggregations
* Critical for event-time processing

### 4. Checkpointing
* Tracks processing progress
* Enables fault tolerance and recovery
* Required for streaming queries
* Prevents reprocessing of data

---

## Examples

### Example 1: Basic Incremental Load with Max Timestamp

```python
from pyspark.sql.functions import col, max as spark_max, current_timestamp

def incremental_load_by_timestamp(
    source_table: str,
    target_table: str,
    timestamp_column: str,
    checkpoint_table: str = None
):
    """
    Load only records newer than last processed timestamp
    """
    
    # Get last processed timestamp
    if checkpoint_table and spark.catalog.tableExists(checkpoint_table):
        last_timestamp = spark.table(checkpoint_table).select(spark_max("max_timestamp")).collect()[0][0]
        print(f"Last processed timestamp: {last_timestamp}")
    else:
        # First run - get earliest timestamp or use default
        last_timestamp = None
        print("First run - processing all data")
    
    # Read source data incrementally
    if last_timestamp:
        df = spark.table(source_table).filter(col(timestamp_column) > last_timestamp)
    else:
        df = spark.table(source_table)
    
    record_count = df.count()
    print(f"Processing {record_count} new records")
    
    if record_count > 0:
        # Write to target
        df.write.mode("append").saveAsTable(target_table)
        
        # Update checkpoint
        new_max_timestamp = df.select(spark_max(timestamp_column)).collect()[0][0]
        
        checkpoint_df = spark.createDataFrame([
            {
                'table_name': target_table,
                'max_timestamp': new_max_timestamp,
                'processed_at': current_timestamp(),
                'records_processed': record_count
            }
        ])
        
        if not checkpoint_table:
            checkpoint_table = f"{target_table}_checkpoint"
        
        checkpoint_df.write.mode("append").saveAsTable(checkpoint_table)
        
        print(f"✓ Loaded {record_count} records. New checkpoint: {new_max_timestamp}")
    else:
        print("No new records to process")
    
    return record_count


# Usage
records_loaded = incremental_load_by_timestamp(
    source_table="catalog.bronze.transaction_feed",
    target_table="catalog.silver.transactions",
    timestamp_column="created_at",
    checkpoint_table="catalog.metadata.load_checkpoints"
)
```

### Example 2: Delta Lake Change Data Feed (CDF)

```python
from delta.tables import DeltaTable

# Enable CDF on table (one-time setup)
spark.sql("""
    ALTER TABLE catalog.silver.customers
    SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
""")

def process_changes_with_cdf(
    source_table: str,
    last_version: int = None,
    last_timestamp: str = None
):
    """
    Process incremental changes using Delta Change Data Feed
    """
    
    # Build CDF query
    if last_version is not None:
        # Read changes since specific version
        changes_df = spark.read.format("delta") \
            .option("readChangeDataFeed", "true") \
            .option("startingVersion", last_version) \
            .table(source_table)
    elif last_timestamp:
        # Read changes since timestamp
        changes_df = spark.read.format("delta") \
            .option("readChangeDataFeed", "true") \
            .option("startingTimestamp", last_timestamp) \
            .table(source_table)
    else:
        # Get all changes
        changes_df = spark.read.format("delta") \
            .option("readChangeDataFeed", "true") \
            .table(source_table)
    
    # CDF columns: _change_type, _commit_version, _commit_timestamp
    # _change_type values: insert, update_preimage, update_postimage, delete
    
    # Separate by change type
    inserts = changes_df.filter(col("_change_type") == "insert")
    updates = changes_df.filter(col("_change_type").isin(["update_preimage", "update_postimage"]))
    deletes = changes_df.filter(col("_change_type") == "delete")
    
    print(f"Changes detected:")
    print(f"  Inserts: {inserts.count()}")
    print(f"  Updates: {updates.count() // 2}")  # Divide by 2 (pre + post images)
    print(f"  Deletes: {deletes.count()}")
    
    return {
        'inserts': inserts,
        'updates': updates,
        'deletes': deletes,
        'changes_df': changes_df
    }


# Usage
changes = process_changes_with_cdf(
    source_table="catalog.silver.customers",
    last_timestamp="2024-04-01 00:00:00"
)

# Process each change type appropriately
# e.g., propagate to downstream tables, external systems, etc.
```

### Example 3: MERGE for Upsert Pattern

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col, current_timestamp, lit

def upsert_with_merge(
    source_df: DataFrame,
    target_table: str,
    merge_keys: List[str],
    update_columns: List[str] = None,
    include_audit_columns: bool = True
):
    """
    Perform upsert using Delta MERGE operation
    
    Args:
        source_df: DataFrame with new/updated records
        target_table: Target Delta table name
        merge_keys: Columns to match for merge (e.g., ['customer_id'])
        update_columns: Columns to update (None = all columns)
        include_audit_columns: Add _updated_at metadata
    """
    
    # Add audit columns if requested
    if include_audit_columns:
        source_df = source_df.withColumn("_updated_at", current_timestamp())
        source_df = source_df.withColumn("_is_active", lit(True))
    
    # Get target Delta table
    target = DeltaTable.forName(spark, target_table)
    
    # Build merge condition
    merge_condition = " AND ".join([f"target.{key} = source.{key}" for key in merge_keys])
    
    # Determine update columns
    if update_columns is None:
        update_columns = [c for c in source_df.columns if c not in merge_keys]
    
    # Build update/insert expressions
    update_expr = {col: f"source.{col}" for col in update_columns}
    insert_expr = {col: f"source.{col}" for col in source_df.columns}
    
    # Perform merge
    merge_result = (
        target.alias("target")
        .merge(source_df.alias("source"), merge_condition)
        .whenMatchedUpdate(set=update_expr)
        .whenNotMatchedInsert(values=insert_expr)
        .execute()
    )
    
    print(f"Merge completed:")
    print(f"  Records updated: {merge_result.rows_updated if hasattr(merge_result, 'rows_updated') else 'N/A'}")
    print(f"  Records inserted: {merge_result.rows_inserted if hasattr(merge_result, 'rows_inserted') else 'N/A'}")
    
    return merge_result


# Usage example
new_customers = spark.table("catalog.bronze.customer_updates")

upsert_with_merge(
    source_df=new_customers,
    target_table="catalog.silver.customers",
    merge_keys=["customer_id"],
    update_columns=["email", "phone", "address", "last_modified"],
    include_audit_columns=True
)
```

### Example 4: SCD Type 2 Implementation

```python
from pyspark.sql.functions import col, current_timestamp, lit, md5, concat_ws, when

def scd_type2_merge(
    source_df: DataFrame,
    target_table: str,
    business_key: List[str],
    compare_columns: List[str]
):
    """
    Implement Slowly Changing Dimension Type 2 pattern
    Maintains full history of changes with effective dates
    """
    
    # Add hash for change detection
    source_df = source_df.withColumn(
        "_hash",
        md5(concat_ws("||", *[col(c) for c in compare_columns]))
    )
    
    source_df = source_df.withColumn("_ingestion_time", current_timestamp())
    
    # Get current active records from target
    target = DeltaTable.forName(spark, target_table)
    current_records = spark.table(target_table).filter(col("is_current") == True)
    
    # Join source with current records
    merge_condition = " AND ".join([f"source.{key} = target.{key}" for key in business_key])
    
    # Identify changes
    compare_df = (
        source_df.alias("source")
        .join(
            current_records.alias("target"),
            [col(f"source.{key}") == col(f"target.{key}") for key in business_key],
            "left"
        )
        .withColumn(
            "_change_detected",
            when(
                col("target._hash").isNull() | (col("source._hash") != col("target._hash")),
                True
            ).otherwise(False)
        )
    )
    
    # Records to insert (new or changed)
    new_records = (
        compare_df
        .filter(col("_change_detected"))
        .select("source.*")
        .withColumn("is_current", lit(True))
        .withColumn("effective_start_date", current_timestamp())
        .withColumn("effective_end_date", lit(None).cast("timestamp"))
    )
    
    # Keys of changed records (need to expire old versions)
    changed_keys = (
        compare_df
        .filter(col("_change_detected") & col("target._hash").isNotNull())
        .select([col(f"source.{key}").alias(key) for key in business_key])
    )
    
    # Expire old versions
    if changed_keys.count() > 0:
        expire_condition = " OR ".join([
            f"({' AND '.join([f'target.{key} = expired.{key}' for key in business_key])})"
        ])
        
        (
            target.alias("target")
            .merge(changed_keys.alias("expired"), expire_condition)
            .whenMatchedUpdate(
                condition="target.is_current = true",
                set={
                    "is_current": "false",
                    "effective_end_date": "current_timestamp()"
                }
            )
            .execute()
        )
    
    # Insert new versions
    new_records.write.mode("append").saveAsTable(target_table)
    
    print(f"SCD Type 2 processing complete:")
    print(f"  New versions created: {new_records.count()}")
    print(f"  Old versions expired: {changed_keys.count()}")


# Usage
source_customers = spark.table("catalog.bronze.customer_updates")

scd_type2_merge(
    source_df=source_customers,
    target_table="catalog.silver.customers_scd2",
    business_key=["customer_id"],
    compare_columns=["name", "email", "address", "phone", "tier"]
)
```

### Example 5: Streaming with Watermarking

```python
from pyspark.sql.functions import window, count

# Read streaming data with watermark
events_stream = (
    spark.readStream
    .format("delta")
    .table("catalog.bronze.user_events")
    .withWatermark("event_timestamp", "10 minutes")  # Wait 10 min for late data
)

# Windowed aggregation with watermark
windowed_stats = (
    events_stream
    .groupBy(
        window("event_timestamp", "5 minutes", "1 minute"),  # 5-min window, 1-min slide
        "event_type"
    )
    .agg(count("*").alias("event_count"))
)

# Write with checkpoint for fault tolerance
query = (
    windowed_stats.writeStream
    .format("delta")
    .outputMode("append")  # Watermark enables append mode for aggregations
    .option("checkpointLocation", "/mnt/checkpoints/windowed_events")
    .trigger(processingTime="1 minute")
    .toTable("catalog.silver.event_stats_1min")
)

# Monitor stream
query.awaitTermination()
```

### Example 6: Auto Loader for File Ingestion

```python
from pyspark.sql.functions import input_file_name, current_timestamp

def incremental_file_load_with_auto_loader(
    source_path: str,
    target_table: str,
    checkpoint_path: str,
    schema_location: str,
    file_format: str = "json"
):
    """
    Incrementally load files using Auto Loader (cloudFiles)
    Automatically handles new files and schema evolution
    """
    
    # Configure Auto Loader
    df = (
        spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", file_format)
        .option("cloudFiles.schemaLocation", schema_location)
        .option("cloudFiles.inferColumnTypes", "true")
        .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
        .load(source_path)
        .withColumn("_source_file", input_file_name())
        .withColumn("_ingestion_timestamp", current_timestamp())
    )
    
    # Write stream with checkpoint
    query = (
        df.writeStream
        .format("delta")
        .option("checkpointLocation", checkpoint_path)
        .option("mergeSchema", "true")
        .trigger(availableNow=True)  # Micro-batch mode
        .toTable(target_table)
    )
    
    # Wait for completion
    query.awaitTermination()
    
    print(f"✓ Incremental load complete")
    print(f"  Source: {source_path}")
    print(f"  Target: {target_table}")
    print(f"  Checkpoint: {checkpoint_path}")


# Usage
incremental_file_load_with_auto_loader(
    source_path="s3://my-bucket/data/transactions/",
    target_table="catalog.bronze.transactions",
    checkpoint_path="/mnt/checkpoints/transactions_bronze",
    schema_location="/mnt/schemas/transactions",
    file_format="json"
)
```

---

## Best Practices

### 1. Choose the Right Pattern
* **Append-only**: Simplest, for immutable event data
* **Timestamp-based**: Good for tables with update timestamps
* **CDF**: Best for tracking all changes with Delta tables
* **MERGE**: For upsert patterns, handles updates and inserts
* **Auto Loader**: Ideal for file-based ingestion

### 2. Checkpoint Management
* Store checkpoints in reliable storage (not local)
* Use separate checkpoint locations per stream
* Don't delete checkpoints unless reprocessing is desired
* Monitor checkpoint size growth

### 3. Watermark Configuration
* Set based on acceptable late data window
* Too short: Lose late data
* Too long: High memory usage for stateful operations
* Typical: 10 minutes to 1 hour

### 4. MERGE Performance
* Partition target tables appropriately
* Include partition filters in merge conditions
* Use OPTIMIZE and Z-ORDER on merge key columns
* Consider splitting large merges into batches

### 5. Error Handling
* Implement retry logic for transient failures
* Quarantine malformed records
* Monitor checkpoint progress
* Log processing metrics

### 6. Testing Incremental Logic
* Test with small batches first
* Verify idempotency (reprocessing gives same result)
* Test late-arriving data scenarios
* Validate checkpoint recovery

---

## Common Pitfalls

### ❌ Missing Idempotency
* **Problem**: Reprocessing creates duplicates
* **Solution**: Use MERGE with proper keys or Delta CDF

### ❌ No Watermarking in Streaming
* **Problem**: Unbounded state growth, memory issues
* **Solution**: Add watermark for event-time operations

### ❌ Checkpoint in Wrong Location
* **Problem**: Checkpoint deleted or inaccessible
* **Solution**: Use durable storage, backup checkpoints

### ❌ Full Scans Instead of Incremental
* **Problem**: Processing entire table every time
* **Impact**: Wasted resources, slow processing
* **Solution**: Use timestamp filtering, CDF, or Auto Loader

### ❌ Ignoring Late Data
* **Problem**: Losing events that arrive late
* **Solution**: Implement watermarking with appropriate delay

### ❌ Not Monitoring Processing Lag
* **Problem**: Don't know if pipeline is falling behind
* **Solution**: Track processing timestamps vs. event timestamps

---

## Performance Comparison

### Full Load (Anti-Pattern)
```python
# ❌ Processes everything every time
df = spark.table("source_table")
df.write.mode("overwrite").saveAsTable("target_table")
# 1 TB data, 30 minutes, $100/run
```

### Incremental with Timestamp
```python
# ✅ Processes only new data
df = spark.table("source_table").filter(col("date") > last_date)
df.write.mode("append").saveAsTable("target_table")
# 10 GB data, 2 minutes, $5/run (95% cost reduction)
```

---

## Monitoring Queries

```sql
-- Check CDF is enabled
SHOW TBLPROPERTIES catalog.schema.table_name;
-- Look for: delta.enableChangeDataFeed = true

-- View table history for incremental tracking
DESCRIBE HISTORY catalog.schema.table_name LIMIT 10;

-- Check streaming query status
SELECT * FROM system.streaming.queries;

-- Monitor checkpoint progress (via table properties)
DESCRIBE DETAIL catalog.schema.streaming_table;
```

---

## Related Skills

* [Streaming Pipelines](../streaming-pipelines/SKILL.md) - For real-time streaming patterns
* [Delta Lake Optimization](../delta-lake-optimization/SKILL.md) - For optimizing incremental writes
* [Data Quality Checks](../data-quality-checks/SKILL.md) - For validating incremental data

---

## Integration with Databricks Features

### Delta Live Tables
* Automatically handles incremental processing
* Built-in support for CDC and streaming
* Declarative pipeline definitions

### Auto Loader
* Cloud-native incremental file ingestion
* Automatic schema evolution
* Scales to millions of files

### Workflows
* Schedule incremental jobs
* Pass checkpoint info between tasks
* Monitor processing metrics

---

**Last Updated**: 2024-04-09
