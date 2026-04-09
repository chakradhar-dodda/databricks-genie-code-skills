# Error Handling Skill

## Purpose
This skill teaches Genie Code how to implement robust error handling in Databricks pipelines using try-catch patterns, dead letter queues, retry strategies, circuit breakers, logging, and alerting for production-grade data systems.

## When to Use
* Building production data pipelines
* Handling data quality issues
* Implementing fault-tolerant workflows
* Managing external system failures
* Logging and tracking errors
* Setting up error notifications
* Implementing graceful degradation
* Creating error recovery mechanisms

## Key Concepts

### 1. Error Types
* **Transient errors**: Temporary failures (network, timeout)
* **Data quality errors**: Bad data, schema mismatches
* **System errors**: Out of memory, cluster failures
* **Business logic errors**: Validation failures, constraint violations

### 2. Error Handling Patterns
* **Try-catch blocks**: Explicit error handling
* **Dead letter queue**: Isolate bad records
* **Retry with backoff**: Automatic retry with delays
* **Circuit breaker**: Prevent cascading failures
* **Fail fast**: Stop on critical errors

### 3. Recovery Strategies
* **Retry**: Attempt operation again
* **Fallback**: Use alternative data/logic
* **Skip**: Continue with remaining data
* **Alert and stop**: Notify and halt pipeline

### 4. Logging and Monitoring
* **Error logging**: Capture error details
* **Metrics tracking**: Count and categorize errors
* **Alerting**: Notify stakeholders
* **Error dashboards**: Visualize error patterns

---

## Examples

### Example 1: Try-Catch with Detailed Logging

```python
from pyspark.sql.functions import col, current_timestamp, lit
from datetime import datetime
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

class DataPipelineWithErrorHandling:
    """Pipeline with comprehensive error handling"""
    
    def __init__(self, pipeline_name):
        self.pipeline_name = pipeline_name
        self.error_table = "monitoring.pipeline_errors"
    
    def run_pipeline(self, input_table, output_table):
        """Execute pipeline with error handling"""
        
        try:
            logger.info(f"Starting pipeline: {self.pipeline_name}")
            
            # Read input data
            try:
                df = spark.table(input_table)
                logger.info(f"Read {df.count():,} records from {input_table}")
            except Exception as e:
                self.log_error("DATA_READ_ERROR", f"Failed to read {input_table}", e)
                raise
            
            # Validate data
            try:
                self._validate_data(df)
            except ValueError as e:
                self.log_error("DATA_VALIDATION_ERROR", "Data validation failed", e)
                raise
            
            # Transform data
            try:
                transformed_df = self._transform(df)
                logger.info(f"Transformed {transformed_df.count():,} records")
            except Exception as e:
                self.log_error("TRANSFORMATION_ERROR", "Data transformation failed", e)
                raise
            
            # Write output
            try:
                (
                    transformed_df
                    .write
                    .mode("overwrite")
                    .option("mergeSchema", "true")
                    .saveAsTable(output_table)
                )
                logger.info(f"✅ Successfully wrote to {output_table}")
                
            except Exception as e:
                self.log_error("DATA_WRITE_ERROR", f"Failed to write to {output_table}", e)
                raise
            
            # Log success
            self._log_success(df.count(), transformed_df.count())
            
            return transformed_df
            
        except Exception as e:
            logger.error(f"❌ Pipeline failed: {str(e)}")
            self._send_failure_notification(str(e))
            raise
    
    def _validate_data(self, df):
        """Validate input data"""
        # Check for required columns
        required_cols = ["customer_id", "event_date", "amount"]
        missing_cols = [c for c in required_cols if c not in df.columns]
        
        if missing_cols:
            raise ValueError(f"Missing required columns: {missing_cols}")
        
        # Check for nulls in key columns
        null_counts = df.select([
            count(when(col(c).isNull(), 1)).alias(c) 
            for c in required_cols
        ]).collect()[0]
        
        null_cols = [c for c in required_cols if null_counts[c] > 0]
        if null_cols:
            logger.warning(f"Null values found in: {null_cols}")
    
    def _transform(self, df):
        """Transform data with error handling"""
        return (
            df
            .filter(col("amount") > 0)  # Business logic
            .withColumn("processed_at", current_timestamp())
        )
    
    def log_error(self, error_type, message, exception):
        """Log error to monitoring table"""
        error_data = spark.createDataFrame([
            (
                datetime.now(),
                self.pipeline_name,
                error_type,
                message,
                str(exception),
                str(type(exception).__name__)
            )
        ], ["timestamp", "pipeline", "error_type", "message", "details", "exception_class"])
        
        error_data.write.mode("append").saveAsTable(self.error_table)
        logger.error(f"{error_type}: {message} - {exception}")
    
    def _log_success(self, input_count, output_count):
        """Log successful execution"""
        success_data = spark.createDataFrame([
            (
                datetime.now(),
                self.pipeline_name,
                "SUCCESS",
                input_count,
                output_count
            )
        ], ["timestamp", "pipeline", "status", "input_records", "output_records"])
        
        success_data.write.mode("append").saveAsTable("monitoring.pipeline_runs")
    
    def _send_failure_notification(self, error_message):
        """Send notification on failure"""
        # Implement notification logic (email, Slack, PagerDuty)
        logger.error(f"🚨 ALERT: Pipeline {self.pipeline_name} failed: {error_message}")

# Usage
pipeline = DataPipelineWithErrorHandling("customer_etl")
result = pipeline.run_pipeline(
    input_table="bronze.customers",
    output_table="silver.customers"
)
```

### Example 2: Dead Letter Queue Pattern

```python
from pyspark.sql.functions import col, struct, to_json, current_timestamp

def process_with_dead_letter_queue(input_df, good_table, dlq_table):
    """
    Process records, send bad records to dead letter queue
    """
    
    # Add row ID for tracking
    df_with_id = input_df.withColumn("row_id", monotonically_increasing_id())
    
    # Define validation function
    def is_valid_record(record):
        """Validate individual record"""
        try:
            # Business validation rules
            if record.customer_id is None:
                return False, "Missing customer_id"
            if record.amount <= 0:
                return False, "Invalid amount (must be > 0)"
            if record.event_date is None:
                return False, "Missing event_date"
            return True, None
        except Exception as e:
            return False, str(e)
    
    # Register UDF
    from pyspark.sql.types import StructType, StructField, BooleanType, StringType
    validation_schema = StructType([
        StructField("is_valid", BooleanType()),
        StructField("error_reason", StringType())
    ])
    
    validate_udf = udf(is_valid_record, validation_schema)
    
    # Apply validation
    df_validated = df_with_id.withColumn("validation", validate_udf(struct(*df_with_id.columns)))
    
    # Split good and bad records
    good_records = (
        df_validated
        .filter(col("validation.is_valid") == True)
        .drop("row_id", "validation")
    )
    
    bad_records = (
        df_validated
        .filter(col("validation.is_valid") == False)
        .withColumn("error_reason", col("validation.error_reason"))
        .withColumn("dlq_timestamp", current_timestamp())
        .withColumn("original_record", to_json(struct(*[c for c in input_df.columns])))
        .drop("validation")
    )
    
    # Write good records
    good_count = good_records.count()
    good_records.write.mode("append").saveAsTable(good_table)
    print(f"✅ Processed {good_count:,} valid records")
    
    # Write bad records to DLQ
    bad_count = bad_records.count()
    if bad_count > 0:
        bad_records.write.mode("append").saveAsTable(dlq_table)
        print(f"⚠️ Sent {bad_count:,} invalid records to DLQ: {dlq_table}")
        
        # Alert if too many bad records
        bad_pct = (bad_count / (good_count + bad_count)) * 100
        if bad_pct > 5:
            alert_message = f"High error rate: {bad_pct:.1f}% records failed validation"
            send_alert(alert_message)
    
    return good_records, bad_records

# Usage
input_df = spark.table("bronze.raw_events")
good_df, bad_df = process_with_dead_letter_queue(
    input_df,
    good_table="silver.validated_events",
    dlq_table="monitoring.dlq_events"
)
```

### Example 3: Retry with Exponential Backoff

```python
import time
from functools import wraps

def retry_with_backoff(max_retries=3, initial_delay=1, backoff_factor=2, max_delay=60):
    """
    Decorator for retry logic with exponential backoff
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            delay = initial_delay
            last_exception = None
            
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                
                except Exception as e:
                    last_exception = e
                    
                    if attempt < max_retries - 1:
                        # Log retry attempt
                        print(f"⚠️ Attempt {attempt + 1} failed: {str(e)}")
                        print(f"Retrying in {delay} seconds...")
                        
                        # Wait before retry
                        time.sleep(delay)
                        
                        # Exponential backoff
                        delay = min(delay * backoff_factor, max_delay)
                    else:
                        # Final attempt failed
                        print(f"❌ All {max_retries} attempts failed")
                        raise last_exception
            
            raise last_exception
        
        return wrapper
    return decorator

# Usage example
@retry_with_backoff(max_retries=3, initial_delay=2, backoff_factor=2)
def read_external_api(endpoint):
    """Read data from external API with retry"""
    import requests
    
    response = requests.get(endpoint, timeout=30)
    response.raise_for_status()
    
    return response.json()

@retry_with_backoff(max_retries=5, initial_delay=1)
def write_to_database(df, table_name):
    """Write to database with retry"""
    (
        df.write
        .format("jdbc")
        .option("url", "jdbc:postgresql://...")
        .option("dbtable", table_name)
        .mode("append")
        .save()
    )

# Execute with automatic retry
try:
    data = read_external_api("https://api.example.com/data")
    df = spark.createDataFrame(data)
    write_to_database(df, "target_table")
    print("✅ Operation successful")
except Exception as e:
    print(f"❌ Operation failed after retries: {e}")
```

### Example 4: Circuit Breaker Pattern

```python
from datetime import datetime, timedelta
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"  # Normal operation
    OPEN = "open"      # Blocking requests
    HALF_OPEN = "half_open"  # Testing recovery

class CircuitBreaker:
    """
    Circuit breaker to prevent cascading failures
    """
    
    def __init__(self, 
                 failure_threshold=5, 
                 timeout_seconds=60,
                 half_open_max_calls=3):
        self.failure_threshold = failure_threshold
        self.timeout_seconds = timeout_seconds
        self.half_open_max_calls = half_open_max_calls
        
        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
        self.half_open_calls = 0
    
    def call(self, func, *args, **kwargs):
        """Execute function with circuit breaker protection"""
        
        if self.state == CircuitState.OPEN:
            # Check if timeout has passed
            if (datetime.now() - self.last_failure_time).seconds >= self.timeout_seconds:
                print("🔄 Circuit breaker entering HALF_OPEN state")
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
            else:
                raise Exception("Circuit breaker is OPEN - service unavailable")
        
        if self.state == CircuitState.HALF_OPEN:
            if self.half_open_calls >= self.half_open_max_calls:
                raise Exception("Circuit breaker HALF_OPEN limit reached")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        
        except Exception as e:
            self._on_failure()
            raise e
    
    def _on_success(self):
        """Handle successful call"""
        if self.state == CircuitState.HALF_OPEN:
            self.half_open_calls += 1
            
            if self.half_open_calls >= self.half_open_max_calls:
                print("✅ Circuit breaker CLOSED - service recovered")
                self.state = CircuitState.CLOSED
                self.failure_count = 0
        
        elif self.state == CircuitState.CLOSED:
            self.failure_count = 0
    
    def _on_failure(self):
        """Handle failed call"""
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        
        if self.state == CircuitState.HALF_OPEN:
            print("❌ Circuit breaker OPEN - service still failing")
            self.state = CircuitState.OPEN
        
        elif self.failure_count >= self.failure_threshold:
            print(f"❌ Circuit breaker OPEN - {self.failure_count} failures")
            self.state = CircuitState.OPEN

# Usage
circuit_breaker = CircuitBreaker(failure_threshold=3, timeout_seconds=60)

def unreliable_external_service():
    """Simulate unreliable external service"""
    import requests
    response = requests.get("https://flaky-api.example.com/data")
    return response.json()

# Use circuit breaker
for i in range(10):
    try:
        data = circuit_breaker.call(unreliable_external_service)
        print(f"✅ Call {i+1} successful")
    except Exception as e:
        print(f"❌ Call {i+1} failed: {str(e)}")
    
    time.sleep(1)
```

### Example 5: Comprehensive Error Monitoring Dashboard

```python
# Create error monitoring tables
spark.sql("""
CREATE TABLE IF NOT EXISTS monitoring.pipeline_errors (
    timestamp TIMESTAMP,
    pipeline STRING,
    error_type STRING,
    message STRING,
    details STRING,
    exception_class STRING,
    resolved BOOLEAN DEFAULT FALSE
) USING DELTA
PARTITIONED BY (DATE(timestamp))
""")

# Error analysis queries
error_summary_query = """
SELECT 
    DATE(timestamp) as error_date,
    pipeline,
    error_type,
    COUNT(*) as error_count,
    COUNT(DISTINCT exception_class) as unique_exceptions,
    MAX(timestamp) as last_occurrence
FROM monitoring.pipeline_errors
WHERE timestamp >= CURRENT_DATE() - INTERVAL 7 DAYS
  AND resolved = FALSE
GROUP BY DATE(timestamp), pipeline, error_type
ORDER BY error_date DESC, error_count DESC
"""

error_summary = spark.sql(error_summary_query)
error_summary.display()

# Identify chronic errors
chronic_errors_query = """
WITH error_frequency AS (
    SELECT 
        pipeline,
        error_type,
        COUNT(*) as occurrences,
        MIN(timestamp) as first_seen,
        MAX(timestamp) as last_seen,
        DATEDIFF(MAX(timestamp), MIN(timestamp)) as days_active
    FROM monitoring.pipeline_errors
    WHERE timestamp >= CURRENT_DATE() - INTERVAL 30 DAYS
    GROUP BY pipeline, error_type
)

SELECT *,
    occurrences / (days_active + 1) as errors_per_day
FROM error_frequency
WHERE days_active >= 3  -- Error persisting for 3+ days
ORDER BY errors_per_day DESC
LIMIT 20
"""

chronic_errors = spark.sql(chronic_errors_query)
chronic_errors.display()

# Alert on error spikes
def check_error_spikes(threshold_pct=50):
    """Alert if error rate increases significantly"""
    
    query = """
    WITH daily_errors AS (
        SELECT 
            DATE(timestamp) as date,
            pipeline,
            COUNT(*) as error_count
        FROM monitoring.pipeline_errors
        GROUP BY DATE(timestamp), pipeline
    ),
    error_trends AS (
        SELECT 
            date,
            pipeline,
            error_count,
            LAG(error_count, 1) OVER (PARTITION BY pipeline ORDER BY date) as prev_error_count
        FROM daily_errors
    )
    
    SELECT *,
        ((error_count - prev_error_count) / prev_error_count) * 100 as pct_change
    FROM error_trends
    WHERE date = CURRENT_DATE()
      AND prev_error_count > 0
      AND ((error_count - prev_error_count) / prev_error_count) * 100 > :threshold
    ORDER BY pct_change DESC
    """
    
    spikes = spark.sql(query, threshold=threshold_pct)
    
    if spikes.count() > 0:
        print("🚨 ERROR SPIKES DETECTED:")
        spikes.display()
        
        # Send alerts
        for row in spikes.collect():
            send_alert(
                f"Error spike in {row.pipeline}: "
                f"{row.error_count} errors ({row.pct_change:.1f}% increase)"
            )

check_error_spikes(threshold_pct=50)
```

### Example 6: Graceful Degradation Strategy

```python
def process_with_fallback(primary_source, fallback_source, output_table):
    """
    Process data with graceful degradation
    """
    
    try:
        # Attempt primary processing
        print("Attempting primary data source...")
        df = spark.table(primary_source)
        
        # Validate primary data quality
        if df.count() < 1000:  # Minimum record threshold
            raise ValueError("Insufficient records in primary source")
        
        null_pct = df.filter(col("customer_id").isNull()).count() / df.count()
        if null_pct > 0.1:  # Max 10% nulls
            raise ValueError(f"High null rate: {null_pct:.1%}")
        
        print("✅ Using primary data source")
        
    except Exception as e:
        print(f"⚠️ Primary source failed: {str(e)}")
        print("Falling back to secondary source...")
        
        try:
            # Use fallback source
            df = spark.table(fallback_source)
            
            if df.count() == 0:
                raise ValueError("Fallback source also empty")
            
            print("✅ Using fallback data source")
            
            # Log fallback usage
            log_fallback_usage(primary_source, fallback_source, str(e))
            
        except Exception as fallback_error:
            print(f"❌ Fallback also failed: {str(fallback_error)}")
            
            # Last resort: use cached data
            try:
                df = spark.table(f"{output_table}_cache")
                print("⚠️ Using cached data as last resort")
                
                send_critical_alert(
                    f"Both primary and fallback failed. Using cached data. "
                    f"Primary error: {str(e)}, Fallback error: {str(fallback_error)}"
                )
            except:
                print("❌ No cached data available - pipeline cannot proceed")
                raise
    
    # Process and write
    processed_df = df.transform(apply_transformations)
    processed_df.write.mode("overwrite").saveAsTable(output_table)
    
    # Update cache for future fallback
    processed_df.write.mode("overwrite").saveAsTable(f"{output_table}_cache")
    
    return processed_df

# Usage
result = process_with_fallback(
    primary_source="realtime.customer_events",
    fallback_source="batch.customer_events_daily",
    output_table="processed.customer_metrics"
)
```

---

## Best Practices

### 1. Error Classification
* Distinguish transient vs. permanent errors
* Categorize errors by severity
* Track error patterns over time

### 2. Recovery Strategies
* Implement retry for transient errors
* Use dead letter queues for bad data
* Provide fallback options
* Cache data for emergencies

### 3. Logging
* Log all errors with context
* Include timestamps and trace IDs
* Capture input data causing errors
* Store in searchable format

### 4. Alerting
* Set up severity-based alerts
* Alert on error rate increases
* Notify appropriate teams
* Include actionable information

### 5. Testing
* Test error handling paths
* Simulate failure scenarios
* Validate retry logic
* Test circuit breaker behavior

---

## Related Skills

* [Workflow Orchestration](../workflow-orchestration/SKILL.md) - For task-level error handling
* [Job Scheduling](../job-scheduling/SKILL.md) - For schedule error handling
* [Monitoring Observability](../monitoring-observability/SKILL.md) - For error monitoring
* [Data Quality Checks](../data-quality-checks/SKILL.md) - For data validation

---

**Last Updated**: 2024-04-09
