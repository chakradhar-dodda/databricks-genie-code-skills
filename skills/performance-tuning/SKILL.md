# Performance Tuning Skill

## Purpose
This skill teaches Genie Code how to profile, diagnose, and tune Spark workload performance on Databricks. It covers query profiling, bottleneck identification, memory tuning, executor configuration, spill reduction, and systematic performance optimization methodologies.

## When to Use
* Slow-running queries or jobs need diagnosis
* Memory errors or OutOfMemoryError
* Excessive disk spill impacting performance
* GC pauses causing delays
* Underutilized cluster resources
* Task failures or stragglers
* High shuffle read/write times
* Need to establish performance baselines

## Key Concepts

### 1. Performance Profiling
* **Spark UI**: Primary tool for performance analysis
* **Query plans**: Understand execution strategy
* **Metrics**: Task duration, shuffle, memory, GC
* **Bottlenecks**: Identify slowest stages

### 2. Memory Management
* **Executor memory**: Heap space for task execution
* **Driver memory**: Query planning, collect operations
* **Storage memory**: Caching DataFrames/RDDs
* **Execution memory**: Shuffles, joins, aggregations
* **Memory fractions**: Configurable split

### 3. Executor Configuration
* **Executor cores**: Parallelism per executor
* **Executor memory**: RAM per executor
* **Number of executors**: Total parallelism
* **Balance**: Cores vs. memory vs. count

### 4. Disk Spill
* **Cause**: Insufficient memory for operations
* **Impact**: Significant performance degradation
* **Detection**: Spill metrics in Spark UI
* **Solutions**: Increase memory, reduce data, optimize

### 5. GC Tuning
* **Minor GC**: Young generation cleanup
* **Major GC**: Full heap cleanup (expensive)
* **GC time**: Should be < 10% of task time
* **Tuning**: Adjust heap size, GC algorithm


### 8. Photon Engine Optimization
* **C++ vectorized engine** - 3-5x faster than standard Spark
* **Automatic on serverless** - included by default
* **Can enable on classic clusters** - additional cost
* **Best for**: Aggregations, joins, filters, data transformations
* **Use when**: Query performance is critical
* **Transparent** - no code changes required

---

## Examples

### Example 1: Comprehensive Performance Profiling

```python
import time
from pyspark.sql.functions import col, sum as spark_sum
from datetime import datetime

class PerformanceProfiler:
    """
    Comprehensive performance profiling for Spark queries
    """
    
    def __init__(self, spark_context):
        self.sc = spark_context
    
    def profile_query(self, df, name="Query"):
        """Profile DataFrame query execution"""
        print(f"\n{'='*70}")
        print(f"Performance Profile: {name}")
        print(f"{'='*70}")
        
        # Get initial metrics
        start_time = time.time()
        initial_stages = len(self.sc._jsc.sc().statusTracker().getJobIdsForGroup(None))
        
        # Execute query
        result_count = df.count()
        
        # Calculate duration
        duration = time.time() - start_time
        
        # Get Spark metrics
        metrics = self._get_spark_metrics()
        
        # Print summary
        print(f"\nExecution Summary:")
        print(f"  Result rows: {result_count:,}")
        print(f"  Duration: {duration:.2f}s")
        print(f"  Stages: {metrics['num_stages']}")
        print(f"  Tasks: {metrics['num_tasks']}")
        print(f"  Avg task duration: {metrics['avg_task_duration']:.2f}s")
        
        print(f"\nMemory:")
        print(f"  Peak execution memory: {metrics['peak_execution_memory'] / 1024**3:.2f} GB")
        print(f"  Spill (memory): {metrics['memory_bytes_spilled'] / 1024**3:.2f} GB")
        print(f"  Spill (disk): {metrics['disk_bytes_spilled'] / 1024**3:.2f} GB")
        
        print(f"\nShuffle:")
        print(f"  Shuffle read: {metrics['shuffle_read_bytes'] / 1024**3:.2f} GB")
        print(f"  Shuffle write: {metrics['shuffle_write_bytes'] / 1024**3:.2f} GB")
        print(f"  Shuffle read time: {metrics['shuffle_read_time'] / 1000:.2f}s")
        
        print(f"\nGC:")
        print(f"  GC time: {metrics['gc_time'] / 1000:.2f}s ({metrics['gc_percent']:.1f}% of task time)")
        
        # Identify bottlenecks
        self._identify_bottlenecks(metrics, duration)
        
        return metrics
    
    def _get_spark_metrics(self):
        """Extract metrics from Spark status tracker"""
        status_tracker = self.sc.statusTracker()
        active_jobs = status_tracker.getActiveJobIds()
        
        metrics = {
            'num_stages': 0,
            'num_tasks': 0,
            'avg_task_duration': 0,
            'peak_execution_memory': 0,
            'memory_bytes_spilled': 0,
            'disk_bytes_spilled': 0,
            'shuffle_read_bytes': 0,
            'shuffle_write_bytes': 0,
            'shuffle_read_time': 0,
            'gc_time': 0,
            'total_task_time': 0
        }
        
        # Aggregate metrics from all stages
        for job_id in status_tracker.getJobIdsForGroup(None):
            job_info = status_tracker.getJobInfo(job_id)
            if job_info:
                for stage_id in job_info.stageIds():
                    stage_info = status_tracker.getStageInfo(stage_id)
                    if stage_info:
                        metrics['num_stages'] += 1
                        metrics['num_tasks'] += stage_info.numTasks()
        
        # Calculate derived metrics
        if metrics['num_tasks'] > 0:
            metrics['avg_task_duration'] = metrics['total_task_time'] / metrics['num_tasks']
        if metrics['total_task_time'] > 0:
            metrics['gc_percent'] = (metrics['gc_time'] / metrics['total_task_time']) * 100
        else:
            metrics['gc_percent'] = 0
        
        return metrics
    
    def _identify_bottlenecks(self, metrics, duration):
        """Identify performance bottlenecks"""
        print(f"\n{'='*70}")
        print("Bottleneck Analysis")
        print(f"{'='*70}")
        
        issues = []
        
        # Check for disk spill
        if metrics['disk_bytes_spilled'] > 0:
            issues.append(
                f"⚠️ DISK SPILL: {metrics['disk_bytes_spilled'] / 1024**3:.2f} GB spilled to disk. "
                "Increase executor memory or reduce data size."
            )
        
        # Check for memory spill
        if metrics['memory_bytes_spilled'] > metrics['peak_execution_memory'] * 0.5:
            issues.append(
                f"⚠️ MEMORY SPILL: Significant memory spill detected. "
                "Consider increasing executor memory or optimizing query."
            )
        
        # Check GC overhead
        if metrics['gc_percent'] > 10:
            issues.append(
                f"⚠️ GC OVERHEAD: {metrics['gc_percent']:.1f}% time spent in GC (>10% threshold). "
                "Increase executor memory or tune GC settings."
            )
        
        # Check shuffle overhead
        shuffle_time_percent = (metrics['shuffle_read_time'] / (duration * 1000)) * 100
        if shuffle_time_percent > 30:
            issues.append(
                f"⚠️ SHUFFLE BOTTLENECK: {shuffle_time_percent:.1f}% time spent in shuffle. "
                "Optimize joins, use broadcast, or reduce shuffle partitions."
            )
        
        # Check for task skew
        if metrics['avg_task_duration'] > 0:
            # Note: This is simplified; real implementation would check max vs avg
            issues.append(
                "💡 Check Spark UI for task skew (large variance in task durations)."
            )
        
        if issues:
            for issue in issues:
                print(f"\n{issue}")
        else:
            print("\n✅ No major bottlenecks detected!")
        
        print(f"{'='*70}\n")


# Usage
profiler = PerformanceProfiler(spark.sparkContext)

# Profile a query
df = (
    spark.table("catalog.schema.large_table")
    .join(spark.table("catalog.schema.other_table"), "key")
    .groupBy("category")
    .agg(spark_sum("amount"))
)

metrics = profiler.profile_query(df, "Join and Aggregate Query")
```

### Example 2: Memory Configuration Tuning

```python
# Calculate optimal executor memory configuration

def calculate_optimal_memory_config(
    cluster_node_memory_gb: int,
    num_cores_per_node: int,
    num_worker_nodes: int,
    overhead_fraction: float = 0.10
):
    """
    Calculate optimal memory configuration for Spark executors
    
    Args:
        cluster_node_memory_gb: Total memory per worker node
        num_cores_per_node: CPU cores per worker node
        num_worker_nodes: Number of worker nodes
        overhead_fraction: Memory overhead (default 10%)
    """
    
    # Reserve memory for OS and other processes (10%)
    available_memory_gb = cluster_node_memory_gb * 0.9
    
    # Executor configuration strategies
    
    # Strategy 1: Fat executors (1 executor per node, all cores)
    fat_config = {
        "strategy": "Fat Executors",
        "executors_per_node": 1,
        "executor_cores": num_cores_per_node,
        "executor_memory_gb": int(available_memory_gb),
        "total_executors": num_worker_nodes,
        "total_cores": num_cores_per_node * num_worker_nodes
    }
    
    # Strategy 2: Thin executors (many executors, few cores each)
    cores_per_executor = 2
    executors_per_node = num_cores_per_node // cores_per_executor
    memory_per_executor = available_memory_gb / executors_per_node
    
    thin_config = {
        "strategy": "Thin Executors",
        "executors_per_node": executors_per_node,
        "executor_cores": cores_per_executor,
        "executor_memory_gb": int(memory_per_executor),
        "total_executors": executors_per_node * num_worker_nodes,
        "total_cores": cores_per_executor * executors_per_node * num_worker_nodes
    }
    
    # Strategy 3: Balanced (recommended - 5 cores per executor)
    cores_per_executor = 5
    executors_per_node = num_cores_per_node // cores_per_executor
    memory_per_executor = available_memory_gb / executors_per_node
    
    balanced_config = {
        "strategy": "Balanced (Recommended)",
        "executors_per_node": executors_per_node,
        "executor_cores": cores_per_executor,
        "executor_memory_gb": int(memory_per_executor),
        "total_executors": executors_per_node * num_worker_nodes,
        "total_cores": cores_per_executor * executors_per_node * num_worker_nodes
    }
    
    # Print recommendations
    print(f"\n{'='*70}")
    print(f"Executor Memory Configuration")
    print(f"{'='*70}")
    print(f"Cluster: {num_worker_nodes} nodes x {cluster_node_memory_gb}GB x {num_cores_per_node} cores")
    print(f"Available per node: {available_memory_gb:.1f} GB\n")
    
    for config in [fat_config, thin_config, balanced_config]:
        print(f"{config['strategy']}:")
        print(f"  Executors per node: {config['executors_per_node']}")
        print(f"  Cores per executor: {config['executor_cores']}")
        print(f"  Memory per executor: {config['executor_memory_gb']} GB")
        print(f"  Total executors: {config['total_executors']}")
        print(f"  Total cores: {config['total_cores']}")
        print(f"  Parallelism: {config['total_cores'] * 2-4} (2-4x cores)")
        print()
    
    return balanced_config


# Example: r5d.4xlarge nodes (128 GB, 16 cores)
optimal_config = calculate_optimal_memory_config(
    cluster_node_memory_gb=128,
    num_cores_per_node=16,
    num_worker_nodes=10
)

# Apply configuration
spark.conf.set("spark.executor.memory", f"{optimal_config['executor_memory_gb']}g")
spark.conf.set("spark.executor.cores", str(optimal_config['executor_cores']))
spark.conf.set("spark.executor.instances", str(optimal_config['total_executors']))

# Configure memory fractions
spark.conf.set("spark.memory.fraction", "0.6")  # 60% for execution/storage
spark.conf.set("spark.memory.storageFraction", "0.5")  # 50% of memory.fraction for storage

# Configure off-heap memory (reduces GC pressure)
spark.conf.set("spark.memory.offHeap.enabled", "true")
spark.conf.set("spark.memory.offHeap.size", "2g")
```

### Example 3: Spill Reduction Techniques

```python
from pyspark.sql.functions import col, broadcast

# Identify and reduce disk spill

def analyze_spill(query_name="Query"):
    """Analyze spill metrics from last query"""
    # Note: In real implementation, use Spark UI REST API or listeners
    print(f"\nSpill Analysis for {query_name}:")
    print("Check Spark UI -> Stages -> Aggregated Metrics:")
    print("  - Spill (Memory)")
    print("  - Spill (Disk)")
    print("If spill > 0, apply techniques below\n")


# Scenario 1: Join causing spill
large_orders = spark.table("catalog.sales.orders")  # 100GB
small_products = spark.table("catalog.sales.products")  # 500MB

# ❌ BAD: Large shuffle join causes spill
bad_join = large_orders.join(small_products, "product_id")
analyze_spill("Bad Join")

# ✅ GOOD: Broadcast join eliminates shuffle
good_join = large_orders.join(broadcast(small_products), "product_id")
analyze_spill("Good Join")

# Scenario 2: Aggregation causing spill
# ❌ BAD: High cardinality groupBy with insufficient memory
bad_agg = (
    large_orders
    .groupBy("customer_id", "product_id", "order_date")  # High cardinality
    .agg(spark_sum("amount"))
)

# ✅ GOOD: Reduce cardinality or increase memory
good_agg = (
    large_orders
    .groupBy("customer_id", "product_id")  # Lower cardinality
    .agg(spark_sum("amount"))
)

# Or increase executor memory
spark.conf.set("spark.executor.memory", "16g")  # Was 8g

# Scenario 3: Window functions causing spill
from pyspark.sql.window import Window
from pyspark.sql.functions import row_number

# ❌ BAD: Window over entire dataset
bad_window = (
    large_orders
    .withColumn("row_num", row_number().over(
        Window.partitionBy("customer_id").orderBy("order_date")
    ))
)

# ✅ GOOD: Filter before window
good_window = (
    large_orders
    .filter(col("order_date") >= "2024-01-01")  # Reduce data first
    .withColumn("row_num", row_number().over(
        Window.partitionBy("customer_id").orderBy("order_date")
    ))
)

# Configuration to reduce spill
spark.conf.set("spark.sql.shuffle.partitions", "400")  # More partitions = smaller per-partition data
spark.conf.set("spark.executor.memory", "16g")  # More memory = less spill
spark.conf.set("spark.memory.fraction", "0.7")  # Increase execution memory
```

### Example 4: GC Tuning

```python
# Optimize garbage collection performance

# G1GC configuration (recommended for large heaps)
gc_configs = {
    # Use G1 collector
    "spark.executor.extraJavaOptions": 
        "-XX:+UseG1GC "
        "-XX:InitiatingHeapOccupancyPercent=35 "  # Start GC earlier
        "-XX:ConcGCThreads=4 "  # Concurrent GC threads
        "-XX:+PrintGCDetails "  # Log GC for analysis
        "-XX:+PrintGCTimeStamps "
        "-verbose:gc",
    
    # Increase young generation (reduce minor GC frequency)
    "spark.executor.memoryOverhead": "2g",
    
    # Increase memory to reduce GC pressure
    "spark.executor.memory": "16g",
    
    # Reduce cached data eviction
    "spark.memory.storageFraction": "0.3"
}

# Apply configurations
for key, value in gc_configs.items():
    spark.conf.set(key, value)

# Monitor GC in Spark UI:
# - GC time should be < 10% of task time
# - If GC time > 10%, increase executor memory
# - If minor GC frequent, increase young generation

# Alternative: Use off-heap memory to reduce GC
spark.conf.set("spark.memory.offHeap.enabled", "true")
spark.conf.set("spark.memory.offHeap.size", "4g")

print("GC Configuration Applied")
print("Monitor Spark UI -> Executors -> GC Time")
print("Target: GC Time < 10% of Task Time")
```

### Example 5: Query Plan Analysis and Optimization

```python
from pyspark.sql.functions import col, count, sum as spark_sum

def optimize_query_iteratively(initial_query, iterations=3):
    """
    Iteratively optimize query based on execution plan analysis
    """
    current_query = initial_query
    
    for i in range(iterations):
        print(f"\n{'='*70}")
        print(f"Optimization Iteration {i+1}")
        print(f"{'='*70}\n")
        
        # Analyze plan
        print("Execution Plan:")
        current_query.explain(True)
        
        # Benchmark
        start_time = time.time()
        count = current_query.count()
        duration = time.time() - start_time
        
        print(f"\nIteration {i+1} Results:")
        print(f"  Row count: {count:,}")
        print(f"  Duration: {duration:.2f}s")
        
        # Identify optimization opportunities
        plan_str = current_query._jdf.queryExecution().toString()
        
        optimizations = []
        
        if "SortMergeJoin" in plan_str and "small" in str(current_query):
            optimizations.append("Convert to broadcast join")
            
        if "Filter" not in plan_str.split("Join")[0] if "Join" in plan_str else False:
            optimizations.append("Push filter before join")
        
        if "Exchange" in plan_str:
            optimizations.append("Reduce shuffle operations")
        
        if optimizations:
            print(f"\n  Optimization opportunities:")
            for opt in optimizations:
                print(f"    - {opt}")
        else:
            print(f"\n  ✅ Query is well-optimized!")
            break
    
    return current_query


# Example usage
orders = spark.table("catalog.sales.orders")
products = spark.table("catalog.sales.products")
customers = spark.table("catalog.sales.customers")

initial_query = (
    orders
    .join(products, "product_id")
    .join(customers, "customer_id")
    .filter(col("order_date") >= "2024-01-01")
    .groupBy("customer_segment", "product_category")
    .agg(
        count("*").alias("order_count"),
        spark_sum("order_amount").alias("total_amount")
    )
)

optimized_query = optimize_query_iteratively(initial_query)
```

### Example 6: Task Skew Detection and Resolution

```python
from pyspark.sql.functions import spark_partition_id, count, col, lit, rand

def detect_task_skew(df, partition_key):
    """
    Detect skew in partition distribution
    """
    skew_analysis = (
        df
        .withColumn("partition_id", spark_partition_id())
        .groupBy("partition_id", partition_key)
        .agg(count("*").alias("record_count"))
        .groupBy("partition_id")
        .agg(
            count("*").alias("num_keys"),
            spark_sum("record_count").alias("total_records")
        )
        .orderBy(col("total_records").desc())
    )
    
    stats = skew_analysis.agg(
        avg("total_records").alias("avg_records"),
        max("total_records").alias("max_records"),
        min("total_records").alias("min_records")
    ).collect()[0]
    
    skew_factor = stats.max_records / stats.avg_records
    
    print(f"\nPartition Skew Analysis:")
    print(f"  Average records/partition: {stats.avg_records:,.0f}")
    print(f"  Max records/partition: {stats.max_records:,}")
    print(f"  Min records/partition: {stats.min_records:,}")
    print(f"  Skew factor: {skew_factor:.2f}x")
    
    if skew_factor > 3:
        print(f"  ⚠️ HIGH SKEW DETECTED (>{skew_factor:.1f}x)")
        print(f"  Recommendations:")
        print(f"    1. Enable AQE skew join")
        print(f"    2. Use salting technique")
        print(f"    3. Separate hot keys")
    else:
        print(f"  ✅ Skew is acceptable (<3x)")
    
    return skew_analysis


# Usage
df = spark.table("catalog.sales.orders")
skew_analysis = detect_task_skew(df, "customer_id")

# Resolution: Enable AQE skew handling
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")
```

### Example 7: Resource Utilization Monitoring

```python
def monitor_cluster_resources():
    """
    Monitor cluster resource utilization
    """
    # Get cluster metrics from system tables
    query = """
    SELECT 
        timestamp,
        cluster_id,
        -- CPU metrics
        AVG(cpu_percent) as avg_cpu_utilization,
        MAX(cpu_percent) as peak_cpu_utilization,
        
        -- Memory metrics
        AVG(memory_percent) as avg_memory_utilization,
        MAX(memory_percent) as peak_memory_utilization,
        
        -- Disk metrics
        AVG(disk_used_percent) as avg_disk_utilization,
        
        -- Network metrics
        SUM(network_in_bytes) / 1024 / 1024 / 1024 as network_in_gb,
        SUM(network_out_bytes) / 1024 / 1024 / 1024 as network_out_gb
        
    FROM system.compute.cluster_metrics
    WHERE timestamp >= CURRENT_TIMESTAMP() - INTERVAL 1 HOUR
    GROUP BY timestamp, cluster_id
    ORDER BY timestamp DESC
    LIMIT 100
    """
    
    metrics = spark.sql(query)
    
    # Analyze resource usage
    summary = metrics.agg(
        avg("avg_cpu_utilization").alias("avg_cpu"),
        avg("avg_memory_utilization").alias("avg_memory"),
        avg("avg_disk_utilization").alias("avg_disk")
    ).collect()[0]
    
    print("\nCluster Resource Utilization (Last Hour):")
    print(f"  CPU: {summary.avg_cpu:.1f}%")
    print(f"  Memory: {summary.avg_memory:.1f}%")
    print(f"  Disk: {summary.avg_disk:.1f}%")
    
    # Recommendations
    if summary.avg_cpu < 30:
        print("\n  💡 CPU underutilized. Consider smaller cluster or more parallelism.")
    
    if summary.avg_memory > 80:
        print("\n  ⚠️ High memory utilization. Consider increasing executor memory.")
    
    if summary.avg_disk > 70:
        print("\n  ⚠️ High disk utilization. Check for excessive spill or logging.")
    
    return metrics


# Monitor resources
resource_metrics = monitor_cluster_resources()

# Set up automated monitoring
def create_resource_alert(threshold_cpu=80, threshold_memory=85):
    """Create alerts for resource thresholds"""
    alert_query = f"""
    SELECT 
        timestamp,
        cluster_id,
        cpu_percent,
        memory_percent
    FROM system.compute.cluster_metrics
    WHERE 
        timestamp >= CURRENT_TIMESTAMP() - INTERVAL 5 MINUTES
        AND (cpu_percent > {threshold_cpu} OR memory_percent > {threshold_memory})
    """
    
    alerts = spark.sql(alert_query)
    
    if alerts.count() > 0:
        print(f"\n⚠️ RESOURCE ALERT:")
        alerts.show()
        # Send notification (email, Slack, PagerDuty, etc.)
    
    return alerts

# Run periodic checks
resource_alerts = create_resource_alert()
```

---

### Example 8: Photon Engine Performance

```python
# Photon is automatic on serverless, or enable on classic cluster
# Cluster configuration:
# {
#   "photon_enabled": true,
#   "runtime_version": "14.3.x-photon-scala2.12"
# }

# Photon excels at these operations:
from pyspark.sql.functions import *

# 1. Complex aggregations (3-5x faster with Photon)
df_aggregated = (
    spark.table("sales.transactions")
    .groupBy("customer_id", "product_category", "region")
    .agg(
        sum("amount").alias("total_sales"),
        avg("amount").alias("avg_transaction"),
        count("*").alias("transaction_count"),
        approx_percentile("amount", 0.95).alias("p95_amount")
    )
)

# 2. Large joins (2-4x faster with Photon)
result = (
    spark.table("sales.orders")
    .join(spark.table("sales.customers"), "customer_id")
    .join(spark.table("sales.products"), "product_id")
    .join(spark.table("sales.regions"), "region_id")
)

# 3. Window functions (4-6x faster with Photon)
from pyspark.sql.window import Window

window_spec = Window.partitionBy("customer_id").orderBy("order_date")
df_windowed = (
    spark.table("sales.orders")
    .withColumn("running_total", sum("amount").over(window_spec))
    .withColumn("order_rank", row_number().over(window_spec))
    .withColumn("prev_order_date", lag("order_date").over(window_spec))
)

# Monitor Photon usage in query plan
df_aggregated.explain("formatted")
# Look for "Photon" operators in the plan
```

**Photon Performance Comparison:**
```python
# Benchmark: Standard Spark vs Photon
import time

def benchmark_query(query_name, query_func, iterations=3):
    times = []
    for i in range(iterations):
        start = time.time()
        result = query_func()
        result.count()  # Force execution
        elapsed = time.time() - start
        times.append(elapsed)
    
    avg_time = sum(times) / len(times)
    return avg_time

# Example results (actual results may vary):
# Aggregation query: Standard=45s, Photon=12s (3.8x faster)
# Join query: Standard=32s, Photon=14s (2.3x faster)
# Window function: Standard=67s, Photon=11s (6.1x faster)
```

**When to Use Photon:**
* ✅ SQL-heavy workloads (aggregations, joins)
* ✅ BI and analytics queries
* ✅ Data transformation pipelines
* ✅ ETL with complex logic
* ⚠️ Not beneficial for: Pure Python/Scala UDFs, streaming with low latency requirements


## Best Practices

### 1. Profile Before Optimizing
* Use Spark UI to identify bottlenecks
* Establish baseline metrics
* Focus on biggest bottlenecks first
* Measure impact of each change

### 2. Right-Size Executors
* Use 5 cores per executor (sweet spot)
* Allocate 2-4GB memory per core
* Leave 10% memory overhead
* Balance parallelism and resource usage

### 3. Minimize Spill
* Increase executor memory if spilling
* Use broadcast joins for small tables
* Reduce data before expensive operations
* Optimize partition count

### 4. Tune GC Carefully
* Target < 10% GC time
* Use G1GC for large heaps (>32GB)
* Enable off-heap memory
* Monitor GC logs

### 5. Monitor Continuously
* Track query duration trends
* Set up resource utilization alerts
* Monitor spill and GC metrics
* Review Spark UI regularly

---

## Common Pitfalls

### ❌ Over-Provisioning Memory
* **Problem**: Waste money, longer GC pauses
* **Solution**: Right-size based on actual usage

### ❌ Ignoring Spill Metrics
* **Problem**: Severe performance degradation
* **Solution**: Monitor spill, increase memory or optimize

### ❌ Too Many/Few Cores
* **Problem**: Poor parallelism or resource contention
* **Solution**: Use 5 cores per executor

### ❌ Not Using Spark UI
* **Problem**: Guessing at optimizations
* **Solution**: Always check Spark UI first

---

## Related Skills

* [Spark Optimization](../spark-optimization/SKILL.md) - For Spark-specific optimizations
* [SQL Best Practices](../sql-best-practices/SKILL.md) - For SQL query optimization
* [Cost Optimization](../cost-optimization/SKILL.md) - For cost-effective configurations

---

**Last Updated**: 2024-04-09
