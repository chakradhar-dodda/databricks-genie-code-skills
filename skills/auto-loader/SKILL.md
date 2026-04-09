# Auto Loader Skill

## Purpose
This skill teaches Genie Code how to use Databricks Auto Loader (cloudFiles) for efficient, scalable file ingestion from cloud storage. It covers schema inference, evolution, format handling, and production patterns for automatically processing millions of files.

## When to Use
* Incrementally loading files from S3, ADLS, or GCS
* Handling schema evolution automatically
* Processing millions of small files efficiently
* Setting up file-based data pipelines
* Avoiding manual file tracking
* Scaling file ingestion to petabytes
* Supporting multiple file formats (JSON, CSV, Parquet, Avro, XML)

## Key Concepts

### 1. Auto Loader Architecture
* Built on Structured Streaming
* Automatic schema inference and evolution
* Efficient file discovery (directory listing or file notification)
* Scales to millions of files
* Exactly-once processing guarantee

### 2. File Discovery Modes
* **Directory Listing**: Scans directory periodically (default)
* **File Notification**: Uses cloud events (SNS/SQS for AWS, Event Grid for Azure)
* Notification mode recommended for >10K files or high-velocity ingestion

### 3. Schema Evolution Modes
* **rescue**: Captures unexpected columns in _rescued_data
* **addNewColumns**: Adds new columns automatically
* **failOnNewColumns**: Fails on schema changes
* **none**: No schema evolution

### 4. File Formats Supported
* JSON (including multi-line)
* CSV (with headers, delimiters, quotes)
* Parquet
* Avro
* ORC
* Binary
* Text
* XML

### 5. Unity Catalog Volumes Integration (2026)
* **Governed file storage** - Files managed in Unity Catalog
* **Three-level namespace** - catalog.schema.volume
* **Access control** - UC permissions on volumes
* **Cloud-agnostic paths** - `/Volumes/catalog/schema/volume/` works everywhere
* **Lineage tracking** - Auto Loader reads tracked in UC
* **No cloud-specific credentials** - UC handles authentication
* **Best practice** - Store raw files in UC Volumes instead of external storage

### 6. Serverless Auto Loader (2026)
* **Zero configuration** - No cluster sizing needed
* **Instant start** - No cluster startup delay
* **Auto-scaling** - Automatically scales based on file volume
* **Cost-efficient** - Pay only for data processed
* **Photon-accelerated** - Faster parsing and transformation
* **Best for** - Development, ad-hoc ingestion, variable workloads

---

## Examples

### Example 1: Basic Auto Loader for JSON Files

```python
from pyspark.sql.functions import current_timestamp, input_file_name

# Simple Auto Loader configuration
df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/mnt/schemas/events")
    .option("cloudFiles.inferColumnTypes", "true")
    .load("s3://my-bucket/raw/events/")
    .withColumn("_ingestion_timestamp", current_timestamp())
    .withColumn("_source_file", input_file_name())
)

# Write to Delta Lake
query = (
    df.writeStream
    .format("delta")
    .option("checkpointLocation", "/mnt/checkpoints/events")
    .option("mergeSchema", "true")
    .trigger(availableNow=True)  # Process all available files
    .toTable("catalog.bronze.events")
)

query.awaitTermination()
```

### Example 2: Auto Loader with Schema Evolution

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, TimestampType

# Define initial schema hint (optional but recommended)
schema_hint = StructType([
    StructField("id", StringType(), True),
    StructField("timestamp", TimestampType(), True),
    StructField("user_id", StringType(), True),
    StructField("event_type", StringType(), True)
])

# Auto Loader with schema evolution
df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/mnt/schemas/events_v2")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")  # Auto-add new columns
    .option("cloudFiles.schemaHints", schema_hint.simpleString())  # Provide hint
    .option("cloudFiles.maxFilesPerTrigger", "1000")  # Rate limiting
    .load("s3://my-bucket/raw/events/")
)

# Write with schema merge enabled
query = (
    df.writeStream
    .format("delta")
    .option("checkpointLocation", "/mnt/checkpoints/events_v2")
    .option("mergeSchema", "true")  # Allow schema evolution in target
    .trigger(processingTime="5 minutes")
    .toTable("catalog.bronze.events_evolved")
)
```

### Example 3: CSV with Complex Options

```python
# Auto Loader for CSV files with custom options
df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("cloudFiles.schemaLocation", "/mnt/schemas/transactions")
    .option("cloudFiles.inferColumnTypes", "true")
    # CSV-specific options
    .option("header", "true")
    .option("delimiter", ",")
    .option("quote", '"')
    .option("escape", "\\")
    .option("multiLine", "false")
    .option("dateFormat", "yyyy-MM-dd")
    .option("timestampFormat", "yyyy-MM-dd HH:mm:ss")
    .option("mode", "PERMISSIVE")  # or DROPMALFORMED, FAILFAST
    .option("columnNameOfCorruptRecord", "_corrupt_record")
    .load("s3://my-bucket/raw/transactions/")
)

query = (
    df.writeStream
    .format("delta")
    .option("checkpointLocation", "/mnt/checkpoints/transactions_csv")
    .toTable("catalog.bronze.transactions")
)
```

### Example 4: File Notification Mode (Production Pattern)

```python
# AWS S3 with SNS/SQS file notifications (most scalable)
df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/mnt/schemas/high_volume")
    .option("cloudFiles.useNotifications", "true")  # Enable file notifications
    .option("cloudFiles.region", "us-east-1")  # AWS region
    # Optional: Use existing SQS queue
    .option("cloudFiles.queueUrl", "https://sqs.us-east-1.amazonaws.com/123456/my-queue")
    .option("cloudFiles.includeExistingFiles", "true")  # Process existing + new files
    .option("cloudFiles.maxFilesPerTrigger", "10000")
    .load("s3://my-bucket/high-volume/data/")
)

query = (
    df.writeStream
    .format("delta")
    .option("checkpointLocation", "/mnt/checkpoints/high_volume")
    .trigger(processingTime="1 minute")
    .toTable("catalog.bronze.high_volume_events")
)
```

### Example 5: Partition Discovery and Filtering

```python
from pyspark.sql.functions import col, input_file_name, regexp_extract

# Auto Loader with partition column extraction
df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/mnt/schemas/partitioned_data")
    .option("cloudFiles.partitionColumns", "year,month,day")  # Hive-style partitions
    .load("s3://my-bucket/data/year=*/month=*/day=*/")
)

# Or extract partition info from path manually
df_with_partitions = (
    df
    .withColumn("file_path", input_file_name())
    .withColumn("year", regexp_extract("file_path", r"year=(\d+)", 1))
    .withColumn("month", regexp_extract("file_path", r"month=(\d+)", 1))
    .withColumn("day", regexp_extract("file_path", r"day=(\d+)", 1))
)

# Filter to process only recent data
filtered_df = df_with_partitions.filter(
    (col("year") == "2024") & (col("month").isin("03", "04"))
)

query = (
    filtered_df.writeStream
    .format("delta")
    .option("checkpointLocation", "/mnt/checkpoints/partitioned")
    .partitionBy("year", "month", "day")
    .toTable("catalog.bronze.partitioned_data")
)
```

### Example 6: Multi-Format Pipeline

```python
def create_auto_loader_pipeline(
    source_path: str,
    file_format: str,
    target_table: str,
    schema_location: str,
    checkpoint_location: str,
    format_options: dict = None
):
    """
    Reusable Auto Loader pipeline factory
    
    Args:
        source_path: Cloud storage path (s3://, abfss://, gs://)
        file_format: json, csv, parquet, avro, etc.
        target_table: Fully qualified Delta table name
        schema_location: Path for schema tracking
        checkpoint_location: Path for checkpoints
        format_options: Dict of format-specific options
    """
    
    # Start with base options
    reader = (
        spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", file_format)
        .option("cloudFiles.schemaLocation", schema_location)
        .option("cloudFiles.inferColumnTypes", "true")
        .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    )
    
    # Add format-specific options
    if format_options:
        for key, value in format_options.items():
            reader = reader.option(key, value)
    
    # Load data
    df = reader.load(source_path)
    
    # Add metadata columns
    from pyspark.sql.functions import current_timestamp, input_file_name
    df = (
        df
        .withColumn("_ingestion_timestamp", current_timestamp())
        .withColumn("_source_file", input_file_name())
    )
    
    # Write to Delta
    query = (
        df.writeStream
        .format("delta")
        .option("checkpointLocation", checkpoint_location)
        .option("mergeSchema", "true")
        .trigger(availableNow=True)
        .toTable(target_table)
    )
    
    return query


# Usage examples
# JSON pipeline
json_query = create_auto_loader_pipeline(
    source_path="s3://bucket/json/",
    file_format="json",
    target_table="catalog.bronze.json_data",
    schema_location="/mnt/schemas/json",
    checkpoint_location="/mnt/checkpoints/json",
    format_options={"multiLine": "true"}
)

# CSV pipeline
csv_query = create_auto_loader_pipeline(
    source_path="s3://bucket/csv/",
    file_format="csv",
    target_table="catalog.bronze.csv_data",
    schema_location="/mnt/schemas/csv",
    checkpoint_location="/mnt/checkpoints/csv",
    format_options={
        "header": "true",
        "delimiter": "|",
        "inferSchema": "true"
    }
)

# Parquet pipeline (simplest)
parquet_query = create_auto_loader_pipeline(
    source_path="s3://bucket/parquet/",
    file_format="parquet",
    target_table="catalog.bronze.parquet_data",
    schema_location="/mnt/schemas/parquet",
    checkpoint_location="/mnt/checkpoints/parquet"
)
```

### Example 7: Error Handling with Rescue Column

```python
# Use _rescued_data column to capture schema mismatches
df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/mnt/schemas/with_rescue")
    .option("cloudFiles.schemaEvolutionMode", "rescue")  # Capture unexpected data
    .option("rescuedDataColumn", "_rescued_data")  # Column name for rescued data
    .load("s3://my-bucket/evolving-schema/")
)

# Filter and handle records with rescued data
from pyspark.sql.functions import col

valid_records = df.filter(col("_rescued_data").isNull())
rescued_records = df.filter(col("_rescued_data").isNotNull())

# Write valid records to main table
valid_query = (
    valid_records.writeStream
    .format("delta")
    .option("checkpointLocation", "/mnt/checkpoints/valid")
    .toTable("catalog.bronze.valid_records")
)

# Write rescued records to quarantine table for review
rescued_query = (
    rescued_records.writeStream
    .format("delta")
    .option("checkpointLocation", "/mnt/checkpoints/rescued")
    .toTable("catalog.quarantine.rescued_records")
)
```

### Example 8: Unity Catalog Volumes with Serverless Auto Loader (2026)

```python
from pyspark.sql.functions import current_timestamp, input_file_name, col

# ✅ Modern approach: Unity Catalog Volumes (cloud-agnostic)
# Files stored in: /Volumes/prod_analytics/bronze/landing_zone/customer_events/

# Serverless Auto Loader from Unity Catalog Volumes
df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    # Store schema in UC Volume
    .option("cloudFiles.schemaLocation", "/Volumes/prod_analytics/bronze/schemas/customer_events")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    .option("cloudFiles.maxFilesPerTrigger", "1000")
    # Read from UC Volume (no cloud-specific credentials needed!)
    .load("/Volumes/prod_analytics/bronze/landing_zone/customer_events/")
    .withColumn("_ingestion_timestamp", current_timestamp())
    .withColumn("_source_file", input_file_name())
)

# Serverless streaming write to UC table
# Run this on serverless compute (num_workers = 0 in cluster config)
query = (
    df.writeStream
    .format("delta")
    # Store checkpoints in UC Volume
    .option("checkpointLocation", "/Volumes/prod_analytics/bronze/checkpoints/customer_events")
    .option("mergeSchema", "true")
    .trigger(processingTime="5 minutes")  # Or availableNow=True
    .toTable("prod_analytics.bronze.customer_events")
)

query.awaitTermination()

# Monitor streaming query
from pyspark.sql.streaming import StreamingQuery

def monitor_auto_loader_stream(query: StreamingQuery):
    """Monitor Auto Loader streaming query metrics"""
    while query.isActive:
        progress = query.lastProgress
        if progress:
            print(f"Batch: {progress.batchId}")
            print(f"  Files processed: {progress.numInputRows}")
            print(f"  Processing time: {progress.batchDuration}ms")
            print(f"  Input rate: {progress.inputRowsPerSecond} rows/sec")
            
            # Check for schema evolution
            if "sources" in progress:
                for source in progress["sources"]:
                    if "description" in source:
                        print(f"  Source: {source['description']}")
        
        time.sleep(30)

# Run monitor in background
import time
monitor_auto_loader_stream(query)
```

**Unity Catalog Volumes Benefits:**
* ✅ Cloud-agnostic file paths (works on AWS, Azure, GCP)
* ✅ No credential management (UC handles auth)
* ✅ Governed storage (access control via GRANT)
* ✅ Lineage tracking (file reads tracked in UC)
* ✅ Easier debugging (consistent paths across environments)

**Comparison: Old vs New Approach**

```python
# ❌ OLD: Cloud-specific paths, manual credentials
df_old = (
    spark.readStream.format("cloudFiles")
    .option("cloudFiles.format", "json")
    .load("s3://my-bucket/data/")  # Cloud-specific
    # Requires IAM roles, keys, or instance profiles
)

# ✅ NEW: Unity Catalog Volumes, serverless
df_new = (
    spark.readStream.format("cloudFiles")
    .option("cloudFiles.format", "json")
    .load("/Volumes/catalog/schema/volume/data/")  # Cloud-agnostic
    # UC handles authentication automatically
)

# Serverless compute (in cluster configuration):
{
    "num_workers": 0,  # Serverless!
    "spark_version": "14.3.x-scala2.12",
    "node_type_id": "Standard_DS3_v2",
    "data_security_mode": "USER_ISOLATION",
    "runtime_engine": "PHOTON"  # Photon included
}
```

**Migration Pattern:**
```python
# 1. Create Unity Catalog Volume
spark.sql("""
    CREATE VOLUME IF NOT EXISTS prod_analytics.bronze.landing_zone
    COMMENT 'Raw files landing zone'
""")

# 2. Copy existing files to UC Volume (one-time)
dbutils.fs.cp(
    "s3://my-bucket/data/",
    "/Volumes/prod_analytics/bronze/landing_zone/",
    recurse=True
)

# 3. Update Auto Loader to read from UC Volume
# (see Example 8 above)

# 4. Grant access to users/groups
spark.sql("""
    GRANT READ, WRITE ON VOLUME prod_analytics.bronze.landing_zone 
    TO `data-engineers@company.com`
""")
```

---

## Best Practices

### 1. Choose Right File Discovery Mode
* **<10K files**: Directory listing (default) works fine
* **>10K files or high velocity**: Use file notifications
* File notifications reduce latency and cost

### 2. Schema Management
* Provide schema hints for critical fields
* Use addNewColumns mode for flexible evolution
* Store schema location in versioned storage
* Document schema changes

### 3. Performance Optimization
* Use `maxFilesPerTrigger` to control batch size
* Enable `useNotifications` for large-scale ingestion
* Partition target tables appropriately
* Use `availableNow` trigger for backfill

### 4. Monitoring
* Track files processed per batch
* Monitor schema evolution events
* Alert on rescued data volume
* Watch checkpoint size growth

### 5. File Organization
* Use consistent naming patterns
* Leverage Hive-style partitioning
* Avoid very small files (<128MB)
* Clean up processed files (optional)

### 6. Unity Catalog Integration (2026)
* **Store files in UC Volumes** - Governed, cloud-agnostic storage
* **Use UC Volumes for schemas** - Centralized schema management
* **Use UC Volumes for checkpoints** - Better governance
* **Run on serverless compute** - Zero configuration, auto-scaling
* **Grant UC permissions** - Control access at volume level

---

## Common Pitfalls

### ❌ No Schema Location
* **Problem**: Can't infer schema on restart
* **Solution**: Always specify cloudFiles.schemaLocation

### ❌ Wrong File Discovery Mode
* **Problem**: Slow discovery with millions of files
* **Solution**: Use file notifications for large-scale

### ❌ Not Handling Schema Evolution
* **Problem**: Pipeline fails on new columns
* **Solution**: Use addNewColumns or rescue mode

### ❌ Processing Files Multiple Times
* **Problem**: No checkpoint, reprocesses files
* **Solution**: Always set checkpointLocation

### ❌ Ignoring Rescued Data
* **Problem**: Silently losing data in _rescued_data
* **Solution**: Monitor and alert on rescued records

---

## Cloud-Specific Configuration

### AWS S3
```python
.option("cloudFiles.region", "us-east-1")
.option("cloudFiles.useNotifications", "true")
.load("s3://bucket/path/")
```

### Azure ADLS Gen2
```python
.option("cloudFiles.useNotifications", "true")
.option("cloudFiles.region", "eastus")
.load("abfss://container@account.dfs.core.windows.net/path/")
```

### Google Cloud Storage
```python
.option("cloudFiles.useNotifications", "true")
.load("gs://bucket/path/")
```

### Unity Catalog Volumes (Cloud-Agnostic, 2026)
```python
# Works on any cloud - no cloud-specific options needed
.load("/Volumes/catalog/schema/volume/path/")
```

---

## Related Skills

* [Streaming Pipelines](../streaming-pipelines/SKILL.md) - For general streaming patterns
* [Incremental Processing](../incremental-processing/SKILL.md) - For batch incremental patterns
* [Data Quality Checks](../data-quality-checks/SKILL.md) - For validating ingested data
* [Unity Catalog Governance](../unity-catalog-governance/SKILL.md) - For UC Volumes setup

---

**Last Updated**: 2026-04-09
