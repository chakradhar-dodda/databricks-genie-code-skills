# Streaming Pipelines Skill

## Purpose
This skill teaches Genie Code how to build production-grade streaming data pipelines on Databricks using Structured Streaming, Auto Loader, and Delta Lake. It covers real-time data processing, state management, watermarking, and fault tolerance patterns for continuous data pipelines.

## When to Use
* Building real-time or near-real-time data pipelines
* Processing continuous streams from Kafka, Kinesis, Event Hubs
* Implementing CDC (Change Data Capture) patterns
* Handling late-arriving data with watermarking
* Building stateful stream processing applications
* Aggregating streaming data with windows
* Implementing exactly-once processing semantics
* Monitoring and troubleshooting streaming jobs

## Key Concepts

### 1. Structured Streaming
* Unified batch and streaming API
* Micro-batch and continuous processing modes
* Built on Spark SQL engine
* Fault-tolerant with checkpointing
* Exactly-once semantics with idempotent sinks

### 2. Streaming Sources
* **Kafka**: Distributed message broker
* **Kinesis**: AWS streaming service
* **Event Hubs**: Azure streaming service
* **Delta Lake**: Table as streaming source (CDF)
* **Auto Loader**: Cloud file ingestion
* **Rate Source**: Testing and development

### 3. Watermarking
* Handles late-arriving data
* Defines maximum lateness threshold
* Enables state cleanup
* Required for append mode aggregations
* Critical for windowed operations

### 4. Triggers
* **ProcessingTime**: Fixed interval micro-batches
* **Once**: Process available data once and stop
* **AvailableNow**: Process all available data, then stop
* **Continuous**: Low-latency continuous processing (experimental)

### 5. Output Modes
* **Append**: Only new rows (with watermark for aggregations)
* **Complete**: Full result table (for aggregations)
* **Update**: Only changed rows

---

## Examples

### Example 1: Basic Streaming from Kafka

```python
from pyspark.sql.functions import col, from_json, current_timestamp
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, TimestampType

# Define schema for incoming JSON data
event_schema = StructType([
    StructField("event_id", StringType(), True),
    StructField("user_id", StringType(), True),
    StructField("event_type", StringType(), True),
    StructField("event_timestamp", TimestampType(), True),
    StructField("properties", StringType(), True)
])

# Read from Kafka
streaming_df = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "kafka-broker:9092")
    .option("subscribe", "user-events")
    .option("startingOffsets", "latest")  # or "earliest"
    .option("failOnDataLoss", "false")  # Handle topic deletion gracefully
    .load()
)

# Parse Kafka message
parsed_df = (
    streaming_df
    .select(
        from_json(col("value").cast("string"), event_schema).alias("data"),
        col("timestamp").alias("kafka_timestamp"),
        col("topic"),
        col("partition"),
        col("offset")
    )
    .select("data.*", "kafka_timestamp", "topic", "partition", "offset")
    .withColumn("processing_timestamp", current_timestamp())
)

# Write to Delta Lake with checkpoint
query = (
    parsed_df.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/mnt/checkpoints/user_events")
    .trigger(processingTime="30 seconds")
    .toTable("catalog.bronze.user_events")
)

# Monitor the stream
print(f"Stream ID: {query.id}")
print(f"Status: {query.status}")
```

### Example 2: Windowed Aggregations with Watermark

```python
from pyspark.sql.functions import window, count, sum as spark_sum, avg, max as spark_max

# Read streaming data
events = (
    spark.readStream
    .format("delta")
    .table("catalog.bronze.user_events")
    .withWatermark("event_timestamp", "10 minutes")  # Handle 10 min late data
)

# Windowed aggregation
windowed_stats = (
    events
    .groupBy(
        window("event_timestamp", "5 minutes", "1 minute"),  # 5-min window, 1-min slide
        "event_type"
    )
    .agg(
        count("*").alias("event_count"),
        count("user_id").alias("unique_users"),
        spark_max("processing_timestamp").alias("max_processing_time")
    )
    .select(
        col("window.start").alias("window_start"),
        col("window.end").alias("window_end"),
        "event_type",
        "event_count",
        "unique_users",
        "max_processing_time"
    )
)

# Write aggregated results
query = (
    windowed_stats.writeStream
    .format("delta")
    .outputMode("append")  # Watermark enables append mode
    .option("checkpointLocation", "/mnt/checkpoints/windowed_stats")
    .trigger(processingTime="1 minute")
    .toTable("catalog.silver.event_stats_1min")
)
```

### Example 3: Stateful Stream Processing with mapGroupsWithState

```python
from pyspark.sql.functions import col
from pyspark.sql.streaming import GroupState, GroupStateTimeout
from typing import Iterator, Tuple

# Define state schema
class SessionState:
    def __init__(self, start_time, end_time, event_count, events):
        self.start_time = start_time
        self.end_time = end_time
        self.event_count = event_count
        self.events = events

def update_session_state(
    key: Tuple[str],
    events: Iterator[Row],
    state: GroupState
) -> Iterator[Row]:
    """
    Track user sessions with custom state management
    Session ends after 30 minutes of inactivity
    """
    user_id = key[0]
    
    # Get existing state or create new
    if state.exists:
        session = state.get
    else:
        session = SessionState(None, None, 0, [])
    
    # Process events
    event_list = list(events)
    
    if event_list:
        for event in event_list:
            session.event_count += 1
            session.events.append(event.event_type)
            
            if session.start_time is None:
                session.start_time = event.event_timestamp
            session.end_time = event.event_timestamp
        
        # Set timeout for 30 minutes
        state.setTimeoutDuration("30 minutes")
        state.update(session)
        
        # Don't output yet, session still active
        return iter([])
    
    elif state.hasTimedOut:
        # Session timed out, emit final result
        result = Row(
            user_id=user_id,
            session_start=session.start_time,
            session_end=session.end_time,
            session_duration_seconds=(session.end_time - session.start_time).total_seconds(),
            event_count=session.event_count,
            event_types=",".join(session.events)
        )
        state.remove()
        return iter([result])
    
    return iter([])


# Apply stateful transformation
events = spark.readStream.format("delta").table("catalog.bronze.user_events")

sessions = (
    events
    .groupBy("user_id")
    .flatMapGroupsWithState(
        update_session_state,
        outputMode="append",
        timeoutConf=GroupStateTimeout.ProcessingTimeTimeout
    )
)

# Write sessions
query = (
    sessions.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/mnt/checkpoints/user_sessions")
    .toTable("catalog.silver.user_sessions")
)
```

### Example 4: Stream-Stream Joins

```python
from pyspark.sql.functions import expr

# Stream 1: User events
events = (
    spark.readStream
    .format("delta")
    .table("catalog.bronze.user_events")
    .withWatermark("event_timestamp", "5 minutes")
)

# Stream 2: User profiles (slowly changing)
profiles = (
    spark.readStream
    .format("delta")
    .table("catalog.bronze.user_profiles")
    .withWatermark("updated_at", "1 hour")
)

# Join streams
enriched_events = (
    events.alias("e")
    .join(
        profiles.alias("p"),
        expr("""
            e.user_id = p.user_id AND
            e.event_timestamp >= p.updated_at AND
            e.event_timestamp <= p.updated_at + interval 1 hour
        """),
        "left"
    )
    .select(
        "e.*",
        "p.user_name",
        "p.user_tier",
        "p.user_segment"
    )
)

# Write enriched stream
query = (
    enriched_events.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/mnt/checkpoints/enriched_events")
    .trigger(processingTime="30 seconds")
    .toTable("catalog.silver.enriched_events")
)
```

### Example 5: Streaming Deduplication

```python
from pyspark.sql.functions import col, expr
from pyspark.sql.window import Window

# Read streaming data
events = (
    spark.readStream
    .format("delta")
    .table("catalog.bronze.user_events")
    .withWatermark("event_timestamp", "10 minutes")
)

# Deduplicate on event_id within watermark window
deduplicated = (
    events
    .dropDuplicates(["event_id"])  # Works with watermark
)

# Alternative: More complex deduplication with window function
# (Note: This works for batch, for streaming use dropDuplicates with watermark)
from pyspark.sql.functions import row_number

window = Window.partitionBy("event_id").orderBy(col("event_timestamp").desc())

deduplicated_advanced = (
    events
    .withColumn("row_num", row_number().over(window))
    .filter(col("row_num") == 1)
    .drop("row_num")
)

# Write deduplicated stream
query = (
    deduplicated.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/mnt/checkpoints/deduplicated_events")
    .toTable("catalog.silver.deduplicated_events")
)
```

### Example 6: Monitoring and Observability

```python
from pyspark.sql.functions import current_timestamp, lit

class StreamMonitor:
    """
    Monitor streaming query health and performance
    """
    
    def __init__(self, query):
        self.query = query
    
    def get_status(self):
        """Get current stream status"""
        status = self.query.status
        
        return {
            'stream_id': self.query.id,
            'is_active': self.query.isActive,
            'message': status.message,
            'is_trigger_active': status.isTriggerActive,
            'is_data_available': status.isDataAvailable
        }
    
    def get_progress(self):
        """Get last progress update"""
        progress = self.query.lastProgress
        
        if progress:
            return {
                'batch_id': progress['batchId'],
                'num_input_rows': progress['numInputRows'],
                'input_rows_per_second': progress['inputRowsPerSecond'],
                'process_rows_per_second': progress['processedRowsPerSecond'],
                'batch_duration_ms': progress['durationMs']['triggerExecution'],
                'sources': progress['sources'],
                'sink': progress['sink']
            }
        return None
    
    def write_metrics_to_table(self, metrics_table: str):
        """Write monitoring metrics to Delta table"""
        progress = self.get_progress()
        status = self.get_status()
        
        if progress:
            metrics_data = [{
                'stream_id': status['stream_id'],
                'timestamp': current_timestamp(),
                'batch_id': progress['batch_id'],
                'num_input_rows': progress['num_input_rows'],
                'input_rows_per_second': progress['input_rows_per_second'],
                'process_rows_per_second': progress['process_rows_per_second'],
                'batch_duration_ms': progress['batch_duration_ms'],
                'is_active': status['is_active']
            }]
            
            metrics_df = spark.createDataFrame(metrics_data)
            metrics_df.write.mode("append").saveAsTable(metrics_table)


# Usage
query = events.writeStream.format("delta").toTable("catalog.bronze.events")

monitor = StreamMonitor(query)

# Check status
status = monitor.get_status()
print(f"Stream is active: {status['is_active']}")

# Get progress metrics
progress = monitor.get_progress()
print(f"Processing rate: {progress['process_rows_per_second']} rows/sec")

# Write to monitoring table
monitor.write_metrics_to_table("catalog.monitoring.streaming_metrics")
```

### Example 7: Error Handling and Recovery

```python
from pyspark.sql.functions import col, when, current_timestamp

def create_fault_tolerant_stream(
    source_table: str,
    target_table: str,
    checkpoint_path: str,
    dead_letter_table: str = None
):
    """
    Create streaming pipeline with error handling and dead letter queue
    """
    
    # Read stream
    df = (
        spark.readStream
        .format("delta")
        .table(source_table)
    )
    
    # Add error handling
    processed_df = (
        df
        .withColumn("processing_timestamp", current_timestamp())
        .withColumn(
            "is_valid",
            when(
                (col("user_id").isNotNull()) &
                (col("event_timestamp").isNotNull()) &
                (col("event_type").isNotNull()),
                True
            ).otherwise(False)
        )
    )
    
    # Split valid and invalid records
    valid_df = processed_df.filter(col("is_valid"))
    invalid_df = processed_df.filter(~col("is_valid"))
    
    # Write valid records
    valid_query = (
        valid_df.writeStream
        .format("delta")
        .outputMode("append")
        .option("checkpointLocation", f"{checkpoint_path}/valid")
        .trigger(processingTime="30 seconds")
        .toTable(target_table)
    )
    
    # Write invalid records to dead letter queue
    if dead_letter_table:
        invalid_query = (
            invalid_df.writeStream
            .format("delta")
            .outputMode("append")
            .option("checkpointLocation", f"{checkpoint_path}/invalid")
            .trigger(processingTime="30 seconds")
            .toTable(dead_letter_table)
        )
        
        return valid_query, invalid_query
    
    return valid_query, None


# Usage
valid_query, invalid_query = create_fault_tolerant_stream(
    source_table="catalog.bronze.raw_events",
    target_table="catalog.silver.validated_events",
    checkpoint_path="/mnt/checkpoints/event_validation",
    dead_letter_table="catalog.quarantine.invalid_events"
)
```

---

## Best Practices

### 1. Checkpoint Management
* **Never delete checkpoints** unless reprocessing from scratch
* Store checkpoints in reliable, versioned storage (not local)
* Use separate checkpoint locations per stream
* Monitor checkpoint size growth
* Back up checkpoints for critical streams

### 2. Watermark Configuration
* Set based on acceptable data loss vs. memory tradeoff
* Too short: Risk losing late data
* Too long: High memory usage for state
* Typical: 10 minutes to 1 hour for most use cases
* Test with real data patterns

### 3. Trigger Selection
* **ProcessingTime**: Most common, good for near real-time (30s-5min)
* **AvailableNow**: Good for catching up after downtime
* **Once**: Good for testing
* Avoid continuous mode unless sub-second latency required

### 4. State Management
* Use watermarks to enable state cleanup
* Avoid unbounded state growth
* Monitor state size in checkpoints
* Consider flatMapGroupsWithState for complex state

### 5. Performance Optimization
* Partition streaming targets by date/time
* Enable Auto Optimize for streaming tables
* Use appropriate shuffle partitions
* Monitor processing rate vs. arrival rate

### 6. Monitoring
* Track input rate, processing rate, batch duration
* Set up alerts for stream failures
* Monitor checkpoint growth
* Track end-to-end latency

---

## Common Pitfalls

### ❌ No Watermarking with Aggregations
* **Problem**: Unbounded state growth, eventual OOM
* **Solution**: Always add watermark for stateful operations

### ❌ Changing Schema Without Plan
* **Problem**: Stream fails on schema changes
* **Solution**: Use mergeSchema option or version sources

### ❌ Deleting Checkpoints Accidentally
* **Problem**: Reprocesses all data from beginning
* **Solution**: Back up checkpoints, use lifecycle policies

### ❌ Not Monitoring Processing Lag
* **Problem**: Stream falls behind, users don't know
* **Solution**: Track and alert on processing lag metrics

### ❌ Single Large Trigger Interval
* **Problem**: High latency, bursty resource usage
* **Solution**: Use smaller intervals (30s-1min) for consistent load

### ❌ No Error Handling
* **Problem**: Bad records cause stream failure
* **Solution**: Implement dead letter queue pattern

---

## Performance Tuning

### Optimize Shuffle Partitions
```python
spark.conf.set("spark.sql.shuffle.partitions", "100")  # Adjust based on data volume
```

### Enable Adaptive Query Execution
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
```

### Configure Kafka Settings
```python
.option("maxOffsetsPerTrigger", "100000")  # Rate limiting
.option("kafka.max.poll.records", "10000")  # Records per poll
```

---

## Monitoring Queries

```sql
-- Check active streams
SELECT * FROM system.streaming.queries WHERE is_active = true;

-- View stream metrics
SELECT 
    stream_id,
    timestamp,
    batch_id,
    num_input_rows,
    input_rows_per_second,
    process_rows_per_second,
    batch_duration_ms
FROM catalog.monitoring.streaming_metrics
WHERE stream_id = '<your_stream_id>'
ORDER BY timestamp DESC
LIMIT 100;

-- Check checkpoint size
DESCRIBE DETAIL delta.`/mnt/checkpoints/your_stream`;
```

---

## Related Skills

* [Incremental Processing](../incremental-processing/SKILL.md) - For batch incremental patterns
* [Auto Loader](../auto-loader/SKILL.md) - For file-based streaming ingestion
* [Data Quality Checks](../data-quality-checks/SKILL.md) - For streaming data validation
* [Error Handling](../../orchestration/error-handling/SKILL.md) - For robust error patterns

---

## Integration with Databricks Features

### Delta Live Tables
* Declarative streaming pipelines
* Automatic checkpoint management
* Built-in quality checks
* Simplified monitoring

### Workflows
* Schedule streaming job restarts
* Monitor stream health
* Implement alerting

### Unity Catalog
* Track streaming table lineage
* Manage permissions on streams
* Audit streaming data access

---

**Last Updated**: 2024-04-09
