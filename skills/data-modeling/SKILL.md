# Data Modeling Skill

## Purpose
This skill teaches Genie Code best practices for data modeling on Databricks, including dimensional modeling, star/snowflake schemas, SCD (Slowly Changing Dimensions), data vault, and normalization strategies for lakehouse architectures.

## When to Use
* Designing data warehouse schemas
* Building analytics data models
* Implementing star or snowflake schemas
* Handling slowly changing dimensions
* Normalizing or denormalizing data
* Creating reusable dimensional models
* Optimizing for BI tool performance
* Implementing data vault patterns

## Key Concepts

### 1. Dimensional Modeling
* **Fact Tables**: Measurements, metrics, transactions
* **Dimension Tables**: Descriptive attributes, context
* **Star Schema**: Facts with denormalized dimensions
* **Snowflake Schema**: Facts with normalized dimensions

### 2. Slowly Changing Dimensions (SCD)
* **Type 0**: Retain original (immutable)
* **Type 1**: Overwrite (no history)
* **Type 2**: Add new row (full history)
* **Type 3**: Add new column (limited history)
* **Type 4**: Separate history table
* **Type 6**: Hybrid (1+2+3)

### 3. Data Vault
* **Hubs**: Business keys
* **Links**: Relationships between hubs
* **Satellites**: Descriptive attributes with history

### 4. Normalization Levels
* **1NF**: Atomic values, unique rows
* **2NF**: No partial dependencies
* **3NF**: No transitive dependencies
* **BCNF**: Every determinant is a candidate key

---

## Examples

### Example 1: Star Schema Implementation

```python
from pyspark.sql.functions import col, current_timestamp, md5, concat_ws

# Dimension: Customer (SCD Type 2)
def create_dim_customer():
    """
    Customer dimension with SCD Type 2
    """
    return spark.sql("""
        CREATE TABLE IF NOT EXISTS catalog.gold.dim_customer (
            customer_key BIGINT GENERATED ALWAYS AS IDENTITY,
            customer_id STRING NOT NULL,
            customer_name STRING,
            email STRING,
            phone STRING,
            address STRING,
            city STRING,
            state STRING,
            country STRING,
            customer_segment STRING,
            customer_tier STRING,
            -- SCD Type 2 columns
            effective_start_date TIMESTAMP NOT NULL,
            effective_end_date TIMESTAMP,
            is_current BOOLEAN NOT NULL,
            -- Audit columns
            created_timestamp TIMESTAMP NOT NULL,
            updated_timestamp TIMESTAMP,
            record_hash STRING
        )
        USING DELTA
        COMMENT 'Customer dimension with full history (SCD Type 2)'
    """)


# Dimension: Product
def create_dim_product():
    """
    Product dimension (SCD Type 1 - overwrite)
    """
    return spark.sql("""
        CREATE TABLE IF NOT EXISTS catalog.gold.dim_product (
            product_key BIGINT GENERATED ALWAYS AS IDENTITY,
            product_id STRING NOT NULL,
            product_name STRING NOT NULL,
            category STRING,
            subcategory STRING,
            brand STRING,
            unit_price DECIMAL(10, 2),
            unit_cost DECIMAL(10, 2),
            is_active BOOLEAN,
            -- Audit columns
            created_timestamp TIMESTAMP NOT NULL,
            updated_timestamp TIMESTAMP
        )
        USING DELTA
        COMMENT 'Product dimension with current values only'
    """)


# Dimension: Date
def create_dim_date(start_date: str, end_date: str):
    """
    Date dimension with date attributes
    """
    from pyspark.sql.functions import (
        year, month, dayofmonth, dayofweek, quarter,
        weekofyear, dayofyear, date_format
    )
    
    # Generate date range
    date_df = spark.sql(f"""
        SELECT sequence(
            to_date('{start_date}'),
            to_date('{end_date}'),
            interval 1 day
        ) as date_array
    """).selectExpr("explode(date_array) as date_value")
    
    # Add date attributes
    dim_date = (
        date_df
        .withColumn("date_key", col("date_value").cast("string").replace("-", "").cast("int"))
        .withColumn("year", year("date_value"))
        .withColumn("quarter", quarter("date_value"))
        .withColumn("month", month("date_value"))
        .withColumn("month_name", date_format("date_value", "MMMM"))
        .withColumn("day", dayofmonth("date_value"))
        .withColumn("day_of_week", dayofweek("date_value"))
        .withColumn("day_name", date_format("date_value", "EEEE"))
        .withColumn("week_of_year", weekofyear("date_value"))
        .withColumn("day_of_year", dayofyear("date_value"))
        .withColumn("is_weekend", when(col("day_of_week").isin(1, 7), True).otherwise(False))
        .withColumn("fiscal_year", year("date_value") + 1)  # Adjust for fiscal calendar
        .withColumn("fiscal_quarter", 
            when(col("month").isin(1, 2, 3), 4)
            .when(col("month").isin(4, 5, 6), 1)
            .when(col("month").isin(7, 8, 9), 2)
            .otherwise(3)
        )
    )
    
    dim_date.write.mode("overwrite").saveAsTable("catalog.gold.dim_date")
    return dim_date


# Fact: Sales Transactions
def create_fact_sales():
    """
    Sales fact table with foreign keys to dimensions
    """
    return spark.sql("""
        CREATE TABLE IF NOT EXISTS catalog.gold.fact_sales (
            sales_key BIGINT GENERATED ALWAYS AS IDENTITY,
            -- Foreign keys to dimensions
            customer_key BIGINT NOT NULL,
            product_key BIGINT NOT NULL,
            date_key INT NOT NULL,
            store_key BIGINT NOT NULL,
            -- Degenerate dimensions (transaction-level attributes)
            transaction_id STRING NOT NULL,
            line_item_number INT,
            -- Measures
            quantity INT NOT NULL,
            unit_price DECIMAL(10, 2) NOT NULL,
            discount_amount DECIMAL(10, 2),
            tax_amount DECIMAL(10, 2),
            total_amount DECIMAL(10, 2) NOT NULL,
            -- Audit columns
            created_timestamp TIMESTAMP NOT NULL,
            CONSTRAINT fk_customer FOREIGN KEY (customer_key) REFERENCES catalog.gold.dim_customer(customer_key),
            CONSTRAINT fk_product FOREIGN KEY (product_key) REFERENCES catalog.gold.dim_product(product_key),
            CONSTRAINT fk_date FOREIGN KEY (date_key) REFERENCES catalog.gold.dim_date(date_key)
        )
        USING DELTA
        PARTITIONED BY (date_key)
        COMMENT 'Sales transactions fact table'
    """)


# Create all tables
create_dim_customer()
create_dim_product()
create_dim_date(start_date="2020-01-01", end_date="2030-12-31")
create_fact_sales()
```

### Example 2: SCD Type 2 Implementation

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col, current_timestamp, lit, md5, concat_ws, when

def upsert_scd_type2(
    source_df: DataFrame,
    target_table: str,
    business_key: str,
    compare_columns: list
):
    """
    Implement SCD Type 2 pattern
    Tracks full history of changes
    """
    
    # Add hash for change detection
    source_df = source_df.withColumn(
        "_hash",
        md5(concat_ws("||", *[col(c) for c in compare_columns]))
    )
    
    # Get current records from target
    target = DeltaTable.forName(spark, target_table)
    current_df = spark.table(target_table).filter(col("is_current") == True)
    
    # Join source with current to identify changes
    changes_df = (
        source_df.alias("source")
        .join(
            current_df.alias("target"),
            source_df[business_key] == current_df[business_key],
            "left"
        )
        .select(
            "source.*",
            col("target._hash").alias("_target_hash"),
            col(f"target.{business_key}").alias("_exists")
        )
        .withColumn(
            "_change_type",
            when(col("_exists").isNull(), "INSERT")
            .when(col("_hash") != col("_target_hash"), "UPDATE")
            .otherwise("NOCHANGE")
        )
    )
    
    # Records to insert (new and changed)
    inserts_df = (
        changes_df
        .filter(col("_change_type").isin("INSERT", "UPDATE"))
        .drop("_target_hash", "_exists", "_change_type")
        .withColumn("effective_start_date", current_timestamp())
        .withColumn("effective_end_date", lit(None).cast("timestamp"))
        .withColumn("is_current", lit(True))
        .withColumn("created_timestamp", current_timestamp())
    )
    
    # Business keys of changed records (to expire)
    changed_keys = (
        changes_df
        .filter(col("_change_type") == "UPDATE")
        .select(business_key)
    )
    
    # Step 1: Expire old versions
    if changed_keys.count() > 0:
        (
            target.alias("target")
            .merge(
                changed_keys.alias("changes"),
                f"target.{business_key} = changes.{business_key} AND target.is_current = true"
            )
            .whenMatchedUpdate(
                set={
                    "effective_end_date": "current_timestamp()",
                    "is_current": "false",
                    "updated_timestamp": "current_timestamp()"
                }
            )
            .execute()
        )
    
    # Step 2: Insert new versions
    inserts_df.write.mode("append").saveAsTable(target_table)
    
    print(f"✅ SCD Type 2 complete: {inserts_df.count()} new versions created")


# Usage
source_customers = spark.table("catalog.silver.customers_staging")

upsert_scd_type2(
    source_df=source_customers,
    target_table="catalog.gold.dim_customer",
    business_key="customer_id",
    compare_columns=["customer_name", "email", "phone", "address", "city", "customer_tier"]
)
```

### Example 3: Data Vault Implementation

```python
# Hub: Customer
def create_hub_customer():
    """
    Hub table containing business keys only
    """
    return spark.sql("""
        CREATE TABLE IF NOT EXISTS catalog.vault.hub_customer (
            hub_customer_key STRING NOT NULL,  -- Hash of business key
            customer_id STRING NOT NULL,  -- Business key
            load_timestamp TIMESTAMP NOT NULL,
            record_source STRING NOT NULL,
            PRIMARY KEY (hub_customer_key)
        )
        USING DELTA
        COMMENT 'Customer hub - business keys'
    """)


# Satellite: Customer Details
def create_sat_customer():
    """
    Satellite table with descriptive attributes and history
    """
    return spark.sql("""
        CREATE TABLE IF NOT EXISTS catalog.vault.sat_customer (
            hub_customer_key STRING NOT NULL,  -- FK to hub
            load_timestamp TIMESTAMP NOT NULL,
            customer_name STRING,
            email STRING,
            phone STRING,
            address STRING,
            customer_segment STRING,
            hash_diff STRING NOT NULL,  -- Hash of all attributes
            record_source STRING NOT NULL,
            PRIMARY KEY (hub_customer_key, load_timestamp),
            FOREIGN KEY (hub_customer_key) REFERENCES catalog.vault.hub_customer(hub_customer_key)
        )
        USING DELTA
        COMMENT 'Customer satellite - descriptive attributes with history'
    """)


# Link: Customer-Product
def create_link_customer_product():
    """
    Link table capturing relationship between customer and product
    """
    return spark.sql("""
        CREATE TABLE IF NOT EXISTS catalog.vault.link_customer_product (
            link_customer_product_key STRING NOT NULL,  -- Hash of participating keys
            hub_customer_key STRING NOT NULL,
            hub_product_key STRING NOT NULL,
            load_timestamp TIMESTAMP NOT NULL,
            record_source STRING NOT NULL,
            PRIMARY KEY (link_customer_product_key),
            FOREIGN KEY (hub_customer_key) REFERENCES catalog.vault.hub_customer(hub_customer_key),
            FOREIGN KEY (hub_product_key) REFERENCES catalog.vault.hub_product(hub_product_key)
        )
        USING DELTA
        COMMENT 'Customer-Product purchase relationship'
    """)


# Load Hub from source
def load_hub_customer(source_df: DataFrame):
    """
    Load customer hub from source data
    """
    from pyspark.sql.functions import sha2, concat_ws, current_timestamp
    
    hub_df = (
        source_df
        .select("customer_id")
        .distinct()
        .withColumn("hub_customer_key", sha2(concat_ws("||", col("customer_id")), 256))
        .withColumn("load_timestamp", current_timestamp())
        .withColumn("record_source", lit("OLTP_SYSTEM"))
    )
    
    # Insert only new keys
    DeltaTable.forName(spark, "catalog.vault.hub_customer").alias("target").merge(
        hub_df.alias("source"),
        "target.customer_id = source.customer_id"
    ).whenNotMatchedInsertAll().execute()


# Load Satellite from source
def load_sat_customer(source_df: DataFrame):
    """
    Load customer satellite with change detection
    """
    from pyspark.sql.functions import sha2, concat_ws, current_timestamp, md5
    
    sat_df = (
        source_df
        .withColumn("hub_customer_key", sha2(concat_ws("||", col("customer_id")), 256))
        .withColumn("hash_diff", md5(concat_ws("||",
            col("customer_name"), col("email"), col("phone"), col("address"), col("customer_segment")
        )))
        .withColumn("load_timestamp", current_timestamp())
        .withColumn("record_source", lit("OLTP_SYSTEM"))
        .select("hub_customer_key", "load_timestamp", "customer_name", "email", 
                "phone", "address", "customer_segment", "hash_diff", "record_source")
    )
    
    # Insert only changed records (compare hash_diff)
    current_sat = spark.table("catalog.vault.sat_customer")
    
    new_records = (
        sat_df.alias("source")
        .join(
            current_sat.alias("target"),
            (col("source.hub_customer_key") == col("target.hub_customer_key")) &
            (col("source.hash_diff") == col("target.hash_diff")),
            "left_anti"
        )
    )
    
    new_records.write.mode("append").saveAsTable("catalog.vault.sat_customer")
```

---

## Best Practices

### 1. Choose the Right Model
* **Star Schema**: Best for BI tools, simple queries, good performance
* **Snowflake**: Normalized, storage efficient, complex queries
* **Data Vault**: Flexibility, auditability, enterprise data warehouse
* **Denormalized**: Lakehouse, analytical workloads, fast queries

### 2. SCD Type Selection
* **Type 1**: Current value only, no history needed
* **Type 2**: Full history tracking (most common for analytics)
* **Type 3**: Limited history (previous value column)
* Choose based on business requirements and query patterns

### 3. Dimension Design
* Use surrogate keys (auto-increment)
* Include effective dates for Type 2
* Add audit columns (created_date, updated_date)
* Use conformed dimensions across facts

### 4. Fact Table Design
* Store measurements and metrics
* Foreign keys to dimensions
* Consider grain carefully (transaction vs. aggregated)
* Partition by date for performance

### 5. Performance Optimization
* Partition fact tables by date
* Z-ORDER dimensions by frequently filtered columns
* Use appropriate data types
* Consider materialized aggregations

---

## Common Pitfalls

### ❌ Wrong Grain in Fact Table
* **Problem**: Mixing transaction and aggregate grain
* **Solution**: Define grain clearly, create separate facts if needed

### ❌ Not Using Surrogate Keys
* **Problem**: Natural keys change, joins break
* **Solution**: Always use surrogate keys for dimensions

### ❌ Over-Normalizing
* **Problem**: Complex queries, poor performance
* **Solution**: Denormalize for analytics use cases

### ❌ Missing SCD Type 2 for Changing Attributes
* **Problem**: Lose historical context
* **Solution**: Use SCD Type 2 for attributes that change over time

### ❌ No Date Dimension
* **Problem**: Complex date calculations in queries
* **Solution**: Always create a date dimension with attributes

---

## Related Skills

* [Medallion Architecture](../medallion-architecture/SKILL.md) - For overall data organization
* [Data Quality Checks](../data-quality-checks/SKILL.md) - For model validation
* [Performance Tuning](../../optimization/performance-tuning/SKILL.md) - For model optimization

---

**Last Updated**: 2024-04-09
