# Spark Optimization Skill

## Purpose
This skill teaches Genie Code how to optimize Apache Spark workloads on Databricks through intelligent use of Catalyst optimizer, Adaptive Query Execution (AQE), broadcast joins, partition tuning, data skew handling, and caching strategies. It covers techniques to improve query performance and resource utilization.

## When to Use
* Slow-running Spark queries or jobs
* High shuffle operations causing bottlenecks
* Data skew problems in joins or aggregations
* Memory pressure or spill to disk
* Inefficient broadcast joins
* Suboptimal partition sizes
* Redundant computations
* Wide transformations causing performance issues

## Key Concepts

### 1. Catalyst Optimizer
* **Logical optimization**: Rule-based query rewriting
* **Physical planning**: Choose best execution strategy
* **Code generation**: Compile to bytecode for performance
* **Cost-based optimization**: Statistics-driven decisions

### 2. Adaptive Query Execution (AQE)
* **Runtime optimization**: Adjust execution based on actual data
* **Coalesce partitions**: Reduce small partitions automatically
* **Optimize skew joins**: Handle data skew dynamically
* **Broadcast joins**: Convert to broadcast at runtime
* **Enabled by default** in Databricks Runtime 8.0+

### 3. Broadcast Joins
* **Small table replication**: Send small table to all executors
* **Avoid shuffle**: Eliminate expensive shuffle operations
* **Threshold**: Default 10MB, configurable
* **Explicit control**: Use broadcast() hint

### 4. Partitioning
* **Too many**: Small files, high overhead
* **Too few**: Insufficient parallelism, large tasks
* **Target**: 128MB-1GB per partition
* **Repartition**: Control partition count explicitly

### 5. Data Skew
* **Problem**: Uneven data distribution causes stragglers
* **Detection**: Check partition sizes, task durations
* **Solutions**: Salting, adaptive skew join, pre-aggregation
* **AQE**: Automatic skew handling

### 6. Caching Strategies
* **Cache frequently accessed data**: Avoid recomputation
* **Storage levels**: Memory, disk, or both
* **Delta cache**: SSD-based automatic caching
* **Memory management**: Unpersist when not needed

### 7. Shuffle Optimization
* **Minimize shuffles**: Expensive network operations
* **Partition tuning**: Right-size shuffle partitions
* **Co-located joins**: Pre-partition on join keys
* **Broadcast for small tables**: Avoid shuffle entirely

### 8. Catalyst Query Planning
* **Predicate pushdown**: Filter early in pipeline
* **Column pruning**: Read only needed columns
* **Constant folding**: Evaluate literals at compile time
* **Join reordering**: Optimize join sequence

### 9. Photon-Specific Optimizations (2026)
* **Vectorized execution**: SIMD instructions for 3-5x speedup
* **Optimized operators**: Aggregations, joins, filters, window functions
* **Automatic on serverless**: No configuration needed
* **CPU-intensive workloads**: String processing, aggregations, complex expressions
* **Best with**: Parquet, Delta Lake formats
* **Cost-benefit**: ~20% higher cost, 3-5x faster execution
* **Query patterns**: String manipulation, window functions, nested data

### 10. Serverless Compute Optimization (2026)
* **Instant start**: No cluster warmup overhead (<30 seconds)
* **Auto-scaling**: Scales per query, not per cluster
* **Resource isolation**: Queries don't interfere with each other
* **Photon included**: Always vectorized execution
* **Best for**: Ad-hoc analysis, variable workloads, development
* **Cost model**: Pay per DBU consumed, not cluster uptime
* **Zero configuration**: No cluster sizing decisions

---

## Examples

### Example 1: Enable and Configure Adaptive Query Execution

```python
from pyspark.sql import SparkSession

# Configure AQE for optimal performance
spark.conf.set("spark.sql.adaptive.enabled", "true")

# Coalesce partitions after shuffle
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.initialPartitionNum", "200")
spark.conf.set("spark.sql.adaptive.coalescePartitions.minPartitionSize", "1MB")

# Enable adaptive skew join optimization
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")

# Enable adaptive broadcast join
spark.conf.set("spark.sql.adaptive.autoBroadcastJoinThreshold", "10MB")

# Example query that benefits from AQE
large_df = spark.table("catalog.schema.large_fact_table")
small_df = spark.table("catalog.schema.small_dimension_table")

# AQE will automatically:
# 1. Convert to broadcast join if small_df < 10MB
# 2. Handle skew in join keys
# 3. Coalesce output partitions
result = (
    large_df
    .join(small_df, "customer_id", "left")
    .groupBy("region", "category")
    .agg({"revenue": "sum", "quantity": "sum"})
)

result.write.mode("overwrite").saveAsTable("catalog.schema.optimized_results")
```

### Example 2: Broadcast Joins Optimization

```python
from pyspark.sql.functions import broadcast, col

# Load tables
large_orders = spark.table("catalog.sales.orders")  # 10GB
small_products = spark.table("catalog.sales.products")  # 50MB

# ❌ BAD: Default join may cause shuffle on both sides
bad_join = large_orders.join(small_products, "product_id")

# ✅ GOOD: Explicit broadcast hint
good_join = large_orders.join(
    broadcast(small_products), 
    "product_id"
)

# Check execution plan
good_join.explain()
# Look for "BroadcastHashJoin" instead of "SortMergeJoin"

# For multiple small tables
dimensions = {
    "products": spark.table("catalog.sales.products"),  # 50MB
    "customers": spark.table("catalog.sales.customers"),  # 30MB
    "regions": spark.table("catalog.sales.regions")  # 5MB
}

# Broadcast all dimension tables
result = large_orders
for dim_name, dim_df in dimensions.items():
    join_key = f"{dim_name[:-1]}_id"  # products -> product_id
    result = result.join(broadcast(dim_df), join_key, "left")

# Configure broadcast threshold globally
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "100MB")

# Disable auto broadcast (force shuffle join)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")
```

### Example 3: Partition Tuning

```python
from pyspark.sql.functions import col, spark_partition_id, count

def analyze_partitions(df, name="dataframe"):
    """Analyze partition distribution"""
    partition_stats = (
        df
        .withColumn("partition_id", spark_partition_id())
        .groupBy("partition_id")
        .agg(count("*").alias("row_count"))
        .orderBy("partition_id")
    )
    
    stats = partition_stats.agg(
        count("*").alias("num_partitions"),
        avg("row_count").alias("avg_rows_per_partition"),
        min("row_count").alias("min_rows"),
        max("row_count").alias("max_rows")
    ).collect()[0]
    
    print(f"\n{name} Partition Analysis:")
    print(f"  Partitions: {stats.num_partitions}")
    print(f"  Avg rows/partition: {stats.avg_rows_per_partition:,.0f}")
    print(f"  Min rows: {stats.min_rows:,}")
    print(f"  Max rows: {stats.max_rows:,}")
    print(f"  Skew factor: {stats.max_rows / stats.avg_rows_per_partition:.2f}x")
    
    return partition_stats


# Load data
df = spark.table("catalog.schema.large_table")

# Analyze initial partitions
analyze_partitions(df, "Original")

# ❌ BAD: Too many partitions (small tasks)
too_many = df.repartition(1000)
analyze_partitions(too_many, "Too Many Partitions")

# ❌ BAD: Too few partitions (large tasks)
too_few = df.repartition(2)
analyze_partitions(too_few, "Too Few Partitions")

# ✅ GOOD: Right-size partitions (target 128MB-1GB per partition)
# Calculate optimal partition count
table_size_mb = spark.sql(f"""
    SELECT SUM(size_bytes) / 1024 / 1024 as size_mb 
    FROM catalog.schema.large_table
""").collect()[0].size_mb

target_partition_size_mb = 256  # Target 256MB per partition
optimal_partitions = int(table_size_mb / target_partition_size_mb)

print(f"\nTable size: {table_size_mb:.2f} MB")
print(f"Optimal partitions: {optimal_partitions}")

optimized = df.repartition(optimal_partitions)
analyze_partitions(optimized, "Optimized")

# Repartition by column for downstream operations
# Good for: Subsequent joins, aggregations on the column
partitioned_by_key = df.repartition(100, "customer_id")

# Coalesce to reduce partitions (no shuffle)
# Good for: Final output with many small files
coalesced = df.coalesce(50)

# Configure default shuffle partitions
spark.conf.set("spark.sql.shuffle.partitions", optimal_partitions)
```

### Example 4: Data Skew Handling

```python
from pyspark.sql.functions import col, lit, rand, concat, floor
from pyspark.sql.window import Window

# Load skewed data
orders = spark.table("catalog.sales.orders")
# Assume customer_id has skew (few customers with many orders)

# Method 1: Detect skew
skew_analysis = (
    orders
    .groupBy("customer_id")
    .agg(count("*").alias("order_count"))
    .orderBy(col("order_count").desc())
)

skew_analysis.show(10)
# Shows top customers with disproportionate order counts

# Method 2: Salting technique for skewed joins
customers = spark.table("catalog.sales.customers")

# Add salt to skewed data
salt_factor = 10

orders_salted = (
    orders
    .withColumn("salt", (rand() * salt_factor).cast("int"))
    .withColumn("customer_id_salted", concat(col("customer_id"), lit("_"), col("salt")))
)

# Replicate small table with salt
customers_replicated = (
    customers
    .crossJoin(
        spark.range(salt_factor).selectExpr("id as salt")
    )
    .withColumn("customer_id_salted", concat(col("customer_id"), lit("_"), col("salt")))
)

# Join on salted keys
result = (
    orders_salted
    .join(customers_replicated, "customer_id_salted")
    .drop("salt", "customer_id_salted")
)

# Method 3: Separate hot keys from normal data
# Identify hot keys (top 1% of customers)
hot_keys = (
    orders
    .groupBy("customer_id")
    .agg(count("*").alias("order_count"))
    .orderBy(col("order_count").desc())
    .limit(100)
    .select("customer_id")
)

# Separate hot and cold data
hot_orders = orders.join(broadcast(hot_keys), "customer_id", "inner")
cold_orders = orders.join(broadcast(hot_keys), "customer_id", "left_anti")

# Process hot keys with broadcast
hot_result = hot_orders.join(broadcast(customers), "customer_id")

# Process cold keys normally
cold_result = cold_orders.join(customers, "customer_id")

# Union results
final_result = hot_result.union(cold_result)

# Method 4: Use AQE automatic skew handling (recommended)
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")

# AQE will automatically detect and handle skew
auto_optimized = orders.join(customers, "customer_id")
```

### Example 5: Caching Strategies

```python
from pyspark import StorageLevel
from pyspark.sql.functions import col

# Load base data
base_data = spark.table("catalog.schema.large_table")

# Scenario 1: Multiple transformations on same base data
# ❌ BAD: Recompute base_data for each action
for category in ["A", "B", "C"]:
    filtered = base_data.filter(col("category") == category)
    count = filtered.count()
    print(f"Category {category}: {count} rows")
# This reads base_data from source 3 times!

# ✅ GOOD: Cache base data
base_data.cache()  # or .persist()
base_data.count()  # Materialize the cache

for category in ["A", "B", "C"]:
    filtered = base_data.filter(col("category") == category)
    count = filtered.count()
    print(f"Category {category}: {count} rows")
# Reads from cache, much faster!

# Unpersist when done
base_data.unpersist()

# Scenario 2: Different storage levels
# Memory only (fastest, but may evict)
df_memory = base_data.persist(StorageLevel.MEMORY_ONLY)

# Memory and disk (more reliable)
df_memory_disk = base_data.persist(StorageLevel.MEMORY_AND_DISK)

# Serialized (save memory, slower)
df_serialized = base_data.persist(StorageLevel.MEMORY_ONLY_SER)

# Disk only (for large datasets)
df_disk = base_data.persist(StorageLevel.DISK_ONLY)

# Scenario 3: Cache intermediate results
# ✅ GOOD: Cache expensive intermediate transformations
expensive_transform = (
    base_data
    .join(spark.table("catalog.schema.dim_table1"), "key1")
    .join(spark.table("catalog.schema.dim_table2"), "key2")
    .filter(col("date") > "2024-01-01")
    .withColumn("computed_metric", col("value") * 2 + col("other_value"))
)

expensive_transform.cache()
expensive_transform.count()  # Materialize

# Now use cached result multiple times
result1 = expensive_transform.groupBy("category").count()
result2 = expensive_transform.groupBy("region").sum("value")
result3 = expensive_transform.orderBy("computed_metric").limit(100)

expensive_transform.unpersist()

# Scenario 4: Delta caching (automatic on Databricks)
# Delta cache is automatic for Delta tables
# Configure Delta cache size
spark.conf.set("spark.databricks.io.cache.enabled", "true")
spark.conf.set("spark.databricks.io.cache.maxDiskUsage", "50g")

# Check cache metrics
cache_metrics = spark.sql("CACHE SELECT * FROM catalog.schema.delta_table")

# Scenario 5: Cache management
# Clear all caches
spark.catalog.clearCache()

# Check cached tables
cached_tables = spark.catalog.listCachedTables()
for table in cached_tables:
    print(f"Cached: {table.name}")
```

### Example 6: Shuffle Optimization

```python
from pyspark.sql.functions import col, sum as spark_sum, count, avg

# Load data
df = spark.table("catalog.schema.transactions")

# ❌ BAD: Multiple shuffles
bad_query = (
    df
    .groupBy("customer_id").agg(spark_sum("amount").alias("total"))  # Shuffle 1
    .join(
        df.groupBy("customer_id").agg(count("*").alias("count")),    # Shuffle 2
        "customer_id"
    )
)

# ✅ GOOD: Single shuffle with multiple aggregations
good_query = (
    df
    .groupBy("customer_id")
    .agg(
        spark_sum("amount").alias("total"),
        count("*").alias("count"),
        avg("amount").alias("avg_amount")
    )
)

# Configure shuffle partitions
# Default is 200, but should be based on data size
table_size_gb = 100  # Example: 100GB table
shuffle_partitions = max(200, int(table_size_gb / 0.128))  # Target 128MB per partition

spark.conf.set("spark.sql.shuffle.partitions", shuffle_partitions)

# Enable sort merge join optimization
spark.conf.set("spark.sql.join.preferSortMergeJoin", "true")

# Reduce shuffle for exchanges
spark.conf.set("spark.sql.shuffle.partitions", "auto")  # AQE will optimize

# Pre-shuffle and persist for multiple operations
pre_shuffled = (
    df
    .repartition(100, "customer_id")  # Partition by join/group key
    .cache()
)

# Multiple operations benefit from pre-shuffling
result1 = pre_shuffled.groupBy("customer_id").agg(spark_sum("amount"))
result2 = pre_shuffled.join(customers, "customer_id")  # Co-located join

# Monitor shuffle metrics
query = good_query.write.mode("overwrite").saveAsTable("catalog.schema.result")
# Check Spark UI for shuffle read/write bytes
```

### Example 7: Query Plan Analysis and Optimization

```python
from pyspark.sql.functions import col, sum as spark_sum

def analyze_query_plan(df, name="Query"):
    """Analyze and print query execution plan"""
    print(f"\n{'='*60}")
    print(f"{name} - Logical Plan")
    print(f"{'='*60}")
    df.explain(True)
    
    print(f"\n{'='*60}")
    print(f"{name} - Physical Plan (Cost)")
    print(f"{'='*60}")
    df.explain("cost")
    
    print(f"\n{'='*60}")
    print(f"{name} - Formatted Plan")
    print(f"{'='*60}")
    df.explain("formatted")


# Example query to optimize
orders = spark.table("catalog.sales.orders")
customers = spark.table("catalog.sales.customers")
products = spark.table("catalog.sales.products")

# Initial query (unoptimized)
query_v1 = (
    orders
    .join(customers, orders.customer_id == customers.id)
    .join(products, orders.product_id == products.id)
    .filter(col("order_date") >= "2024-01-01")
    .groupBy("customer_name", "product_category")
    .agg(spark_sum("order_amount").alias("total_amount"))
)

analyze_query_plan(query_v1, "Unoptimized Query")

# Optimized query
query_v2 = (
    orders
    .filter(col("order_date") >= "2024-01-01")  # Filter early (predicate pushdown)
    .join(broadcast(customers), orders.customer_id == customers.id)  # Broadcast small table
    .join(broadcast(products), orders.product_id == products.id)  # Broadcast small table
    .groupBy("customer_name", "product_category")
    .agg(spark_sum("order_amount").alias("total_amount"))
)

analyze_query_plan(query_v2, "Optimized Query")

# Look for in execution plan:
# ✅ GOOD signs:
#   - "Filter" appears early (predicate pushdown)
#   - "BroadcastHashJoin" for small tables
#   - "ColumnarToRow" (Photon optimization)
#   - Minimal "Exchange" (shuffle) operations
#
# ❌ BAD signs:
#   - "SortMergeJoin" for small tables
#   - Many "Exchange" operations
#   - Filter after join (should be before)
#   - No pruning of columns

# Enable Catalyst optimizer debugging
spark.conf.set("spark.sql.optimizer.excludedRules", "")  # Enable all rules

# Force specific optimization
from pyspark.sql.functions import broadcast
optimized_join = orders.join(
    broadcast(customers),
    "customer_id",
    "inner"
)

# Benchmark queries
import time

def benchmark_query(df, name="Query", iterations=3):
    """Benchmark query execution time"""
    times = []
    
    for i in range(iterations):
        start = time.time()
        df.count()  # or .write.mode("overwrite").saveAsTable(...)
        elapsed = time.time() - start
        times.append(elapsed)
        print(f"{name} - Run {i+1}: {elapsed:.2f}s")
    
    avg_time = sum(times) / len(times)
    print(f"{name} - Average: {avg_time:.2f}s\n")
    return avg_time

# Compare versions
time_v1 = benchmark_query(query_v1, "Unoptimized")
time_v2 = benchmark_query(query_v2, "Optimized")

improvement = ((time_v1 - time_v2) / time_v1) * 100
print(f"Performance improvement: {improvement:.1f}%")
```

### Example 8: Advanced Catalyst Optimization

```python
from pyspark.sql.functions import col, when, lit

# Enable cost-based optimization (CBO)
spark.conf.set("spark.sql.cbo.enabled", "true")
spark.conf.set("spark.sql.cbo.joinReorder.enabled", "true")
spark.conf.set("spark.sql.statistics.histogram.enabled", "true")

# Compute statistics for better optimization
spark.sql("ANALYZE TABLE catalog.sales.orders COMPUTE STATISTICS")
spark.sql("ANALYZE TABLE catalog.sales.orders COMPUTE STATISTICS FOR COLUMNS customer_id, product_id, order_date")

spark.sql("ANALYZE TABLE catalog.sales.customers COMPUTE STATISTICS")
spark.sql("ANALYZE TABLE catalog.sales.products COMPUTE STATISTICS")

# Now Catalyst can make better decisions about:
# - Join order
# - Join strategies (broadcast vs shuffle)
# - Partition pruning

# Check statistics
stats = spark.sql("DESCRIBE EXTENDED catalog.sales.orders").collect()
for row in stats:
    print(f"{row.col_name}: {row.data_type}")

# Catalyst will optimize this multi-join query automatically
optimized_query = (
    spark.table("catalog.sales.orders")
    .join(spark.table("catalog.sales.customers"), "customer_id")
    .join(spark.table("catalog.sales.products"), "product_id")
    .join(spark.table("catalog.sales.regions"), "region_id")
    .filter(col("order_date") >= "2024-01-01")
    .groupBy("region_name", "product_category")
    .agg(spark_sum("order_amount").alias("total"))
)

# Catalyst optimizations applied:
# 1. Predicate pushdown (filter before join)
# 2. Column pruning (only select needed columns)
# 3. Join reordering (join smallest tables first)
# 4. Constant folding (evaluate literals at compile time)
# 5. Expression simplification

# Disable specific optimization rules if needed
spark.conf.set(
    "spark.sql.optimizer.excludedRules",
    "org.apache.spark.sql.catalyst.optimizer.EliminateSorts"
)
```

### Example 9: Photon-Optimized Query Patterns (2026)

```python
# Enable Photon runtime (automatic on serverless, optional on classic)
spark.conf.set("spark.databricks.photon.enabled", "true")

# ✅ Photon excels at these patterns:

# 1. String-heavy aggregations (3-5x faster with Photon)
photon_optimized_strings = spark.sql("""
    SELECT 
        UPPER(customer_name) as customer_name,
        CONCAT(first_name, ' ', last_name) as full_name,
        REGEXP_REPLACE(email, '@.*', '') as email_prefix,
        COUNT(*) as transaction_count,
        SUM(amount) as total_amount
    FROM prod_analytics.gold.transactions
    WHERE date >= '2026-01-01'
        AND LOWER(status) = 'completed'
    GROUP BY customer_name, full_name, email_prefix
    HAVING COUNT(*) > 10
""")

# 2. Complex window functions (Photon accelerated)
photon_optimized_windows = spark.sql("""
    SELECT 
        customer_id,
        order_date,
        amount,
        -- Running totals (Photon optimized)
        SUM(amount) OVER (
            PARTITION BY customer_id 
            ORDER BY order_date 
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) as running_total,
        -- Moving average
        AVG(amount) OVER (
            PARTITION BY customer_id 
            ORDER BY order_date 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) as moving_avg_7day,
        -- Ranking
        ROW_NUMBER() OVER (
            PARTITION BY customer_id 
            ORDER BY amount DESC
        ) as amount_rank
    FROM prod_analytics.gold.orders
    WHERE order_date >= '2026-01-01'
""")

# 3. Large-scale joins (Photon + AQE)
photon_optimized_join = spark.sql("""
    SELECT 
        c.customer_id,
        c.customer_segment,
        COUNT(DISTINCT o.order_id) as order_count,
        SUM(o.amount) as total_spent,
        AVG(o.amount) as avg_order_value,
        MAX(o.order_date) as last_order_date
    FROM prod_analytics.gold.customers c
    INNER JOIN prod_analytics.gold.orders o 
        ON c.customer_id = o.customer_id
    WHERE o.order_date >= '2025-01-01'
    GROUP BY c.customer_id, c.customer_segment
    HAVING COUNT(DISTINCT o.order_id) > 5
""")

# 4. Nested data processing (Photon excels at arrays/structs)
photon_optimized_nested = spark.sql("""
    SELECT 
        order_id,
        customer_id,
        -- Array functions (Photon optimized)
        ARRAY_SIZE(line_items) as item_count,
        FILTER(line_items, x -> x.quantity > 1) as multi_quantity_items,
        TRANSFORM(line_items, x -> x.price * x.quantity) as item_totals,
        AGGREGATE(
            TRANSFORM(line_items, x -> x.price * x.quantity),
            0.0,
            (acc, x) -> acc + x
        ) as order_total
    FROM prod_analytics.gold.orders_nested
    WHERE order_date >= '2026-01-01'
""")

# Performance comparison: Photon vs Non-Photon
import time

def benchmark_query(query_name: str, sql_query: str, runs: int = 3):
    """Benchmark query performance"""
    times = []
    for i in range(runs):
        start = time.time()
        df = spark.sql(sql_query)
        df.count()  # Force execution
        elapsed = time.time() - start
        times.append(elapsed)
    
    avg_time = sum(times) / len(times)
    print(f"{query_name}: {avg_time:.2f}s (avg of {runs} runs)")
    return avg_time

# Disable Photon for comparison
spark.conf.set("spark.databricks.photon.enabled", "false")
time_without_photon = benchmark_query(
    "Without Photon",
    """
    SELECT customer_segment, COUNT(*), SUM(total_spent), AVG(total_spent)
    FROM prod_analytics.gold.customer_metrics
    GROUP BY customer_segment
    """
)

# Enable Photon
spark.conf.set("spark.databricks.photon.enabled", "true")
time_with_photon = benchmark_query(
    "With Photon",
    """
    SELECT customer_segment, COUNT(*), SUM(total_spent), AVG(total_spent)
    FROM prod_analytics.gold.customer_metrics
    GROUP BY customer_segment
    """
)

speedup = time_without_photon / time_with_photon
print(f"\n✅ Photon speedup: {speedup:.2f}x faster")

# Serverless compute configuration (in notebook/job settings)
serverless_config = {
    "num_workers": 0,  # Serverless indicator
    "spark_version": "14.3.x-scala2.12",
    "node_type_id": "Standard_DS3_v2",
    "data_security_mode": "USER_ISOLATION",
    "runtime_engine": "PHOTON"  # Always enabled on serverless
}

print("\n📊 Photon Best Practices:")
print("✅ Automatic on serverless compute")
print("✅ 3-5x faster for string operations")
print("✅ 2-3x faster for aggregations")
print("✅ 2-4x faster for joins")
print("✅ Best with Delta Lake and Parquet")
print("⚠️  ~20% higher cost, but much faster execution")
```

**When to Use Photon:**
* ✅ Ad-hoc analytics (serverless with Photon)
* ✅ String-heavy ETL
* ✅ Complex aggregations
* ✅ Large-scale joins
* ✅ Development/testing (serverless)
* ⚠️ Not necessary for simple queries (overhead)

**Serverless vs Classic with Photon:**

```python
# Serverless (Photon always enabled, instant start)
# - Cost: ~$0.50/DBU
# - Startup: <30 seconds
# - Scaling: Automatic per-query
# - Best for: Variable workloads

# Classic with Photon (manual config, cluster warmup)
# - Cost: ~$0.40/DBU (slightly cheaper)
# - Startup: 5-10 minutes
# - Scaling: Cluster-level
# - Best for: Steady workloads, 24/7 jobs
```

---

## Best Practices

### 1. Enable AQE by Default
* AQE is enabled by default in DBR 8.0+
* Provides automatic runtime optimizations
* Handles skew, coalesces partitions, converts joins
* No code changes required

### 2. Use Broadcast for Small Tables
* Explicitly broadcast tables < 100MB
* Avoid shuffle on large fact tables
* Monitor broadcast threshold
* Use `broadcast()` hint when certain

### 3. Optimize Partition Count
* Target 128MB-1GB per partition
* Too many = overhead, too few = no parallelism
* Use `repartition()` for even distribution
* Use `coalesce()` to reduce without shuffle

### 4. Handle Data Skew Proactively
* Monitor partition sizes and task durations
* Use salting for extreme skew
* Enable AQE skew join optimization
* Separate hot keys when necessary

### 5. Cache Strategically
* Cache data reused multiple times
* Don't cache data used once
* Unpersist when done to free memory
* Use appropriate storage level

### 6. Minimize Shuffles
* Combine multiple aggregations in single groupBy
* Pre-partition data by common key
* Use broadcast joins when possible
* Configure shuffle partitions appropriately

### 7. Analyze Query Plans
* Use `.explain()` to understand execution
* Look for expensive shuffles and scans
* Verify broadcast joins are applied
* Check for predicate pushdown

### 8. Leverage Photon for Performance (2026)
* **Use serverless** - Photon included automatically
* **String operations** - 3-5x speedup on text processing
* **Window functions** - 2-3x faster complex analytics
* **Nested data** - Optimized array/struct operations
* **Cost consideration** - ~20% higher, but much faster

### 9. Choose Compute Wisely (2026)
* **Serverless** - Ad-hoc, development, variable workloads
* **Serverless** - Instant start, zero config, Photon included
* **Classic** - 24/7 jobs, steady workloads, predictable load
* **Classic** - Manual tuning, longer startup, cost-optimized for always-on

---

## Common Pitfalls

### ❌ Shuffle Joins on Small Tables
* **Problem**: Expensive shuffle for small dimension tables
* **Solution**: Use `broadcast()` hint for tables < 100MB

### ❌ Too Many Partitions
* **Problem**: High overhead, slow task scheduling
* **Solution**: Target 128MB-1GB per partition

### ❌ Caching Everything
* **Problem**: Memory pressure, cache eviction
* **Solution**: Cache only reused data, unpersist when done

### ❌ Ignoring Data Skew
* **Problem**: Long-running tasks, stragglers
* **Solution**: Enable AQE skew join or use salting

### ❌ Multiple Shuffles for Same Data
* **Problem**: Redundant expensive operations
* **Solution**: Combine aggregations, pre-partition

### ❌ Not Computing Statistics
* **Problem**: Suboptimal Catalyst decisions
* **Solution**: Run ANALYZE TABLE for important tables

### ❌ Not Using Photon (2026)
* **Problem**: 3-5x slower query execution
* **Solution**: Use serverless compute (Photon automatic) or enable Photon on classic

---

## Performance Tuning Checklist

### Query Optimization
- [ ] Enable Adaptive Query Execution
- [ ] Use broadcast joins for small tables
- [ ] Filter early (predicate pushdown)
- [ ] Combine multiple aggregations
- [ ] Minimize shuffle operations

### Partitioning
- [ ] Right-size partitions (128MB-1GB)
- [ ] Configure shuffle partitions appropriately
- [ ] Use repartition for even distribution
- [ ] Coalesce for final output

### Data Skew
- [ ] Enable AQE skew join optimization
- [ ] Monitor partition size distribution
- [ ] Use salting for extreme skew
- [ ] Separate hot keys when needed

### Caching
- [ ] Cache frequently reused data
- [ ] Unpersist when done
- [ ] Use appropriate storage level
- [ ] Monitor cache memory usage

### Catalyst Optimizer
- [ ] Compute table statistics
- [ ] Enable cost-based optimization
- [ ] Analyze query plans
- [ ] Use explain() to verify optimizations

### Modern Features (2026)
- [ ] Use serverless for ad-hoc workloads (Photon automatic)
- [ ] Enable Photon on classic clusters
- [ ] Benchmark Photon speedup for your workload
- [ ] Choose serverless vs classic based on usage pattern

---

## Related Skills

* [Performance Tuning](../performance-tuning/SKILL.md) - For query profiling and bottleneck identification
* [SQL Best Practices](../sql-best-practices/SKILL.md) - For SQL-specific optimizations
* [Delta Lake Optimization](../delta-lake-optimization/SKILL.md) - For Delta-specific optimizations
* [Cost Optimization](../cost-optimization/SKILL.md) - For cost-effective configurations

---

## Integration with Databricks Features

### Photon Engine (2026)
* **Vectorized execution** - SIMD instructions
* **3-5x faster** - SQL/DataFrame operations
* **Automatic on serverless** - No configuration needed
* **Enable on classic** - `runtime_engine="PHOTON"`

### Delta Cache
* Automatically caches frequently accessed data
* SSD-based, faster than memory
* No code changes required
* Monitor with system tables

### Spark UI
* View query execution plans
* Analyze shuffle operations
* Identify bottlenecks
* Monitor task durations

### System Tables
* Query execution metrics
* Partition statistics
* Cache performance
* Resource utilization

---

**Last Updated**: 2026-04-09
