# Medallion Architecture Skill

## Purpose
This skill teaches Genie Code the medallion architecture pattern (Bronze/Silver/Gold layers) for organizing data lakes on Databricks. It covers layer responsibilities, data flow patterns, quality gates, and best practices for building scalable, maintainable data platforms.

## When to Use
* Designing data lake architecture from scratch
* Organizing existing data into structured layers
* Implementing data quality gates between layers
* Building incremental processing pipelines
* Establishing data governance patterns
* Creating consumption-ready analytics tables
* Separating raw, curated, and business-level data

## Key Concepts

### 1. Three-Layer Architecture

**Bronze Layer** (Raw/Landing)
* Store raw, unprocessed data exactly as received
* Minimal transformations (type casting, basic validation)
* Full historical data with audit trails
* Source of truth for reprocessing
* Append-only, immutable

**Silver Layer** (Cleansed/Conformed)
* Deduplicated and validated data
* Schema standardization
* Data quality checks enforced
* Business key defined
* SCD Type 2 for slowly changing dimensions

**Gold Layer** (Business/Analytics)
* Business-level aggregations
* Star/snowflake schemas
* Optimized for analytics and BI
* Department or use-case specific views
* Pre-calculated metrics

### 2. Data Flow Pattern
```
Sources → Bronze → Silver → Gold → Consumption
           ↓        ↓        ↓
        Quarantine Quarantine Audit
```

### 3. Quality Gates
* Bronze → Silver: Schema validation, completeness
* Silver → Gold: Business rule validation, referential integrity

---

## Examples

### Example 1: Bronze Layer Implementation

```python
from pyspark.sql.functions import current_timestamp, input_file_name, col

def ingest_to_bronze(
    source_path: str,
    bronze_table: str,
    file_format: str = "json",
    checkpoint_path: str = None
):
    """
    Ingest raw data to Bronze layer with minimal processing
    
    Principles:
    - Store data exactly as received
    - Add audit columns only
    - Never filter or transform business data
    - Enable full data lineage
    """
    
    # Read with Auto Loader (cloud-native, scalable)
    df = (
        spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", file_format)
        .option("cloudFiles.schemaLocation", f"{checkpoint_path}/schema")
        .option("cloudFiles.inferColumnTypes", "true")
        .option("cloudFiles.schemaEvolutionMode", "rescue")  # Capture unexpected columns
        .load(source_path)
    )
    
    # Add audit columns only
    bronze_df = (
        df
        .withColumn("_bronze_ingestion_timestamp", current_timestamp())
        .withColumn("_source_file", input_file_name())
        .withColumn("_bronze_batch_id", col("_metadata.file_modification_time"))
    )
    
    # Write to Bronze (append-only)
    query = (
        bronze_df.writeStream
        .format("delta")
        .outputMode("append")
        .option("checkpointLocation", f"{checkpoint_path}/checkpoint")
        .option("mergeSchema", "true")  # Allow schema evolution
        .trigger(availableNow=True)
        .toTable(bronze_table)
    )
    
    return query


# Usage
bronze_query = ingest_to_bronze(
    source_path="s3://bucket/raw/transactions/",
    bronze_table="catalog.bronze.transactions_raw",
    file_format="json",
    checkpoint_path="/mnt/checkpoints/bronze_transactions"
)
```

### Example 2: Silver Layer with Data Quality

```python
from pyspark.sql.functions import col, md5, concat_ws, when, row_number
from pyspark.sql.window import Window
from delta.tables import DeltaTable

def bronze_to_silver(
    bronze_table: str,
    silver_table: str,
    business_keys: list,
    required_columns: list,
    validation_rules: dict = None
):
    """
    Transform Bronze to Silver with quality checks
    
    Responsibilities:
    - Deduplicate records
    - Standardize schema
    - Apply data quality rules
    - Create business keys
    - Track lineage
    """
    
    # Read from Bronze
    bronze_df = spark.table(bronze_table)
    
    # 1. Schema Standardization
    silver_df = bronze_df.select(
        *business_keys,
        *[col(c) for c in bronze_df.columns if c not in ['_rescued_data', '_bronze_ingestion_timestamp']],
        col("_bronze_ingestion_timestamp")
    )
    
    # 2. Data Quality Checks
    silver_df = silver_df.withColumn(
        "_is_valid",
        (
            # Check required columns are not null
            reduce(lambda x, y: x & y, [col(c).isNotNull() for c in required_columns])
        )
    )
    
    # Apply custom validation rules
    if validation_rules:
        for rule_name, rule_condition in validation_rules.items():
            silver_df = silver_df.withColumn(
                f"_valid_{rule_name}",
                when(expr(rule_condition), True).otherwise(False)
            )
    
    # 3. Deduplication (keep latest by timestamp)
    window = Window.partitionBy(*business_keys).orderBy(col("_bronze_ingestion_timestamp").desc())
    silver_df = (
        silver_df
        .withColumn("_row_num", row_number().over(window))
        .filter(col("_row_num") == 1)
        .drop("_row_num")
    )
    
    # 4. Add Silver metadata
    silver_df = (
        silver_df
        .withColumn("_silver_processing_timestamp", current_timestamp())
        .withColumn("_record_hash", md5(concat_ws("|", *business_keys)))
    )
    
    # 5. Split valid/invalid records
    valid_df = silver_df.filter(col("_is_valid") == True)
    invalid_df = silver_df.filter(col("_is_valid") == False)
    
    # Write valid records to Silver
    valid_df.write.mode("overwrite").saveAsTable(silver_table)
    
    # Write invalid records to quarantine
    if invalid_df.count() > 0:
        quarantine_table = silver_table.replace(".silver.", ".quarantine.")
        invalid_df.write.mode("append").saveAsTable(quarantine_table)
        print(f"⚠️  {invalid_df.count()} invalid records written to {quarantine_table}")
    
    print(f"✅ Silver table {silver_table} created with {valid_df.count()} valid records")


# Usage
bronze_to_silver(
    bronze_table="catalog.bronze.transactions_raw",
    silver_table="catalog.silver.transactions",
    business_keys=["transaction_id"],
    required_columns=["transaction_id", "customer_id", "amount", "transaction_date"],
    validation_rules={
        "positive_amount": "amount > 0",
        "valid_date": "transaction_date <= current_date()"
    }
)
```

### Example 3: Gold Layer Aggregations

```python
def silver_to_gold(
    silver_table: str,
    gold_table: str,
    aggregation_level: list,
    metrics: dict,
    filters: str = None
):
    """
    Create Gold layer business aggregations
    
    Responsibilities:
    - Business-level aggregations
    - Pre-calculated metrics
    - Star schema facts and dimensions
    - Optimized for BI tools
    """
    
    # Read from Silver
    silver_df = spark.table(silver_table)
    
    # Apply filters if specified
    if filters:
        silver_df = silver_df.filter(filters)
    
    # Build aggregation
    agg_exprs = [expr(f"{agg}({col}) as {alias}") for col, (agg, alias) in metrics.items()]
    
    gold_df = (
        silver_df
        .groupBy(*aggregation_level)
        .agg(*agg_exprs)
        .withColumn("_gold_created_timestamp", current_timestamp())
    )
    
    # Write to Gold (overwrite or merge depending on pattern)
    gold_df.write.mode("overwrite").saveAsTable(gold_table)
    
    # Optimize for queries
    spark.sql(f"OPTIMIZE {gold_table}")
    spark.sql(f"ANALYZE TABLE {gold_table} COMPUTE STATISTICS")
    
    print(f"✅ Gold table {gold_table} created with {gold_df.count()} records")


# Usage - Daily customer metrics
silver_to_gold(
    silver_table="catalog.silver.transactions",
    gold_table="catalog.gold.daily_customer_metrics",
    aggregation_level=["customer_id", "transaction_date"],
    metrics={
        "*": ("count", "transaction_count"),
        "amount": ("sum", "total_amount"),
        "amount": ("avg", "avg_amount"),
        "amount": ("max", "max_amount")
    }
)
```

### Example 4: End-to-End Medallion Pipeline

```python
from pyspark.sql import DataFrame
from typing import Dict, List, Callable

class MedallionPipeline:
    """
    Complete medallion architecture pipeline orchestrator
    """
    
    def __init__(self, catalog: str):
        self.catalog = catalog
        self.bronze_schema = f"{catalog}.bronze"
        self.silver_schema = f"{catalog}.silver"
        self.gold_schema = f"{catalog}.gold"
        self.quarantine_schema = f"{catalog}.quarantine"
    
    def ingest_bronze(
        self,
        source_path: str,
        table_name: str,
        file_format: str = "json"
    ):
        """Bronze ingestion with minimal processing"""
        df = (
            spark.readStream
            .format("cloudFiles")
            .option("cloudFiles.format", file_format)
            .option("cloudFiles.schemaLocation", f"/mnt/schemas/{table_name}")
            .load(source_path)
            .withColumn("_bronze_timestamp", current_timestamp())
            .withColumn("_source_file", input_file_name())
        )
        
        query = (
            df.writeStream
            .format("delta")
            .option("checkpointLocation", f"/mnt/checkpoints/bronze_{table_name}")
            .trigger(availableNow=True)
            .toTable(f"{self.bronze_schema}.{table_name}")
        )
        
        return query
    
    def transform_silver(
        self,
        bronze_table: str,
        silver_table: str,
        transform_func: Callable[[DataFrame], DataFrame],
        quality_checks: List[str]
    ):
        """Silver transformation with quality gates"""
        bronze_df = spark.table(f"{self.bronze_schema}.{bronze_table}")
        
        # Apply transformation
        transformed_df = transform_func(bronze_df)
        
        # Apply quality checks
        quality_df = transformed_df
        for check in quality_checks:
            quality_df = quality_df.withColumn(
                f"_check_{check}",
                expr(check)
            )
        
        # Split valid/invalid
        valid_df = quality_df.filter(
            reduce(lambda x, y: x & y, [col(f"_check_{c}") for c in quality_checks])
        )
        
        invalid_df = quality_df.filter(
            ~reduce(lambda x, y: x & y, [col(f"_check_{c}") for c in quality_checks])
        )
        
        # Write to Silver
        valid_df.write.mode("overwrite").saveAsTable(f"{self.silver_schema}.{silver_table}")
        
        # Write to quarantine
        if invalid_df.count() > 0:
            invalid_df.write.mode("append").saveAsTable(f"{self.quarantine_schema}.{silver_table}")
        
        return valid_df
    
    def build_gold(
        self,
        silver_table: str,
        gold_table: str,
        aggregation_func: Callable[[DataFrame], DataFrame]
    ):
        """Gold aggregation for business consumption"""
        silver_df = spark.table(f"{self.silver_schema}.{silver_table}")
        
        # Apply aggregation
        gold_df = aggregation_func(silver_df)
        
        # Write to Gold
        gold_df.write.mode("overwrite").saveAsTable(f"{self.gold_schema}.{gold_table}")
        
        # Optimize
        spark.sql(f"OPTIMIZE {self.gold_schema}.{gold_table}")
        
        return gold_df


# Usage Example
pipeline = MedallionPipeline(catalog="prod_analytics")

# Bronze ingestion
bronze_query = pipeline.ingest_bronze(
    source_path="s3://bucket/raw/orders/",
    table_name="orders_raw",
    file_format="json"
)

# Silver transformation
def clean_orders(df: DataFrame) -> DataFrame:
    return (
        df
        .dropDuplicates(["order_id"])
        .withColumn("amount", col("amount").cast("double"))
        .withColumn("order_date", col("order_date").cast("date"))
    )

silver_df = pipeline.transform_silver(
    bronze_table="orders_raw",
    silver_table="orders",
    transform_func=clean_orders,
    quality_checks=[
        "order_id IS NOT NULL",
        "amount > 0",
        "order_date <= current_date()"
    ]
)

# Gold aggregation
def aggregate_daily_orders(df: DataFrame) -> DataFrame:
    return (
        df.groupBy("order_date", "customer_id")
        .agg(
            count("*").alias("order_count"),
            sum("amount").alias("total_amount")
        )
    )

gold_df = pipeline.build_gold(
    silver_table="orders",
    gold_table="daily_customer_orders",
    aggregation_func=aggregate_daily_orders
)
```

---

## Best Practices

### 1. Bronze Layer
* Store data exactly as received (raw)
* Never delete or filter business data
* Add audit columns only (_ingestion_timestamp, _source_file)
* Use append-only mode
* Enable schema evolution
* Keep full history for reprocessing

### 2. Silver Layer
* Enforce data quality gates
* Deduplicate on business keys
* Standardize schema and types
* Apply business rules
* Track data lineage
* Use MERGE for updates (SCD Type 1/2)

### 3. Gold Layer
* Pre-aggregate for performance
* Create business-focused views
* Use star/snowflake schemas
* Optimize with Z-ORDER
* Add documentation/comments
* Consider BI tool requirements

### 4. General Principles
* Data flows one direction (Bronze → Silver → Gold)
* Each layer is independently queryable
* Layers can be rebuilt from previous layer
* Use Delta Lake for all layers
* Implement quality quarantine at each gate

---

## Common Pitfalls

### ❌ Filtering Data in Bronze
* **Problem**: Loses source data, can't reprocess
* **Solution**: Store all raw data, filter in Silver

### ❌ Complex Transformations in Bronze
* **Problem**: Difficult to debug, reprocess
* **Solution**: Keep Bronze simple, do transformations in Silver

### ❌ No Quality Gates
* **Problem**: Bad data propagates to Gold
* **Solution**: Implement validation between layers

### ❌ Skipping Layers
* **Problem**: Bronze → Gold directly loses flexibility
* **Solution**: Use all three layers for maintainability

### ❌ Not Using Delta Lake
* **Problem**: No ACID, difficult updates
* **Solution**: Use Delta Lake for all layers

---

## Related Skills

* [Incremental Processing](../incremental-processing/SKILL.md) - For efficient layer processing
* [Data Quality Checks](../data-quality-checks/SKILL.md) - For quality gates
* [Unity Catalog Governance](../../governance/unity-catalog-governance/SKILL.md) - For layer permissions

---

**Last Updated**: 2024-04-09
