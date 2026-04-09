# SQL Best Practices Skill

## Purpose
This skill teaches Genie Code how to write efficient, maintainable SQL queries on Databricks. It covers query optimization techniques, proper use of CTEs, window functions, predicate pushdown, avoiding anti-patterns, and leveraging Delta Lake capabilities for optimal SQL performance.

## When to Use
* Writing production SQL queries
* Optimizing slow-running SQL
* Building complex analytical queries
* Creating reusable SQL patterns
* Designing data transformation logic
* Implementing business logic in SQL
* Migrating SQL from other platforms

## Key Concepts

### 1. Query Structure
* **SELECT optimization**: Only select needed columns
* **WHERE clauses**: Filter early, use indexed columns
* **JOINs**: Proper join types and order
* **GROUP BY**: Aggregate efficiently
* **ORDER BY**: Use only when necessary

### 2. CTEs (Common Table Expressions)
* **Readability**: Break complex queries into steps
* **Maintainability**: Name intermediate results
* **Performance**: Materialized once, referenced multiple times
* **Debugging**: Easier to test each CTE

### 3. Window Functions
* **Ranking**: ROW_NUMBER, RANK, DENSE_RANK
* **Aggregates**: SUM, AVG, COUNT with OVER()
* **Offsets**: LAG, LEAD for time-series
* **Performance**: More efficient than self-joins

### 4. Predicate Pushdown
* **Partition pruning**: Filter on partition columns
* **Early filtering**: WHERE before JOIN
* **Column pruning**: Select only needed columns
* **Delta optimization**: Z-ORDER on filter columns

### 5. Joins
* **INNER**: Most restrictive first
* **LEFT**: Large table on left, small on right
* **BROADCAST**: Explicit for small tables
* **Avoid** CROSS JOIN unless necessary

### 6. Liquid Clustering Best Practices (2026)
* **Next-gen clustering**: Replaces Z-ORDER for better performance
* **Automatic maintenance**: No manual OPTIMIZE ZORDER needed
* **Flexible columns**: Change clustering keys without rewriting data
* **Better performance**: Up to 2x faster than Z-ORDER on large tables
* **Incremental clustering**: Automatically clusters new data
* **Multi-column optimization**: Handles multiple filter columns efficiently
* **When to use**: High-cardinality columns, frequent filters, large tables

### 7. Unity Catalog SQL Patterns (2026)
* **Row-level security**: Filter data based on user identity
* **Column masking**: Dynamic data masking with SQL functions
* **Fine-grained access**: Grant permissions at row/column level
* **Secure views**: Create views with access control
* **Audit tracking**: All queries logged in system tables
* **Data governance**: Centralized access control across all workspaces
* **Identity functions**: current_user(), is_member() for security rules

---

## Examples

### Example 1: Query Structure and CTEs

```sql
-- ❌ BAD: Nested subqueries, hard to read and maintain
SELECT 
    c.customer_name,
    c.customer_tier,
    o.total_orders,
    o.total_amount
FROM customers c
JOIN (
    SELECT 
        customer_id,
        COUNT(*) as total_orders,
        SUM(amount) as total_amount
    FROM (
        SELECT 
            customer_id,
            order_id,
            SUM(line_amount) as amount
        FROM (
            SELECT * FROM order_lines WHERE order_date >= '2024-01-01'
        )
        GROUP BY customer_id, order_id
    )
    GROUP BY customer_id
) o ON c.customer_id = o.customer_id
WHERE c.customer_tier = 'Premium';

-- ✅ GOOD: Clear CTEs, easy to understand and maintain
WITH filtered_lines AS (
    -- Step 1: Filter to relevant time period
    SELECT 
        customer_id,
        order_id,
        line_amount,
        order_date
    FROM order_lines
    WHERE order_date >= '2024-01-01'
),

order_totals AS (
    -- Step 2: Calculate order-level totals
    SELECT 
        customer_id,
        order_id,
        SUM(line_amount) as order_amount
    FROM filtered_lines
    GROUP BY customer_id, order_id
),

customer_summary AS (
    -- Step 3: Aggregate to customer level
    SELECT 
        customer_id,
        COUNT(DISTINCT order_id) as total_orders,
        SUM(order_amount) as total_amount,
        AVG(order_amount) as avg_order_amount
    FROM order_totals
    GROUP BY customer_id
)

-- Step 4: Join with customer data and filter
SELECT 
    c.customer_id,
    c.customer_name,
    c.customer_tier,
    cs.total_orders,
    cs.total_amount,
    cs.avg_order_amount
FROM customers c
JOIN customer_summary cs ON c.customer_id = cs.customer_id
WHERE c.customer_tier = 'Premium'
ORDER BY cs.total_amount DESC;
```

### Example 2: Window Functions for Analytics

```sql
-- Ranking and analytical queries with window functions

-- Example 1: Running totals and moving averages
SELECT 
    order_date,
    customer_id,
    order_amount,
    
    -- Running total
    SUM(order_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as running_total,
    
    -- 7-day moving average
    AVG(order_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7day,
    
    -- Rank within customer by amount
    ROW_NUMBER() OVER (
        PARTITION BY customer_id 
        ORDER BY order_amount DESC
    ) as order_rank,
    
    -- Previous and next order amounts
    LAG(order_amount, 1) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) as previous_order,
    
    LEAD(order_amount, 1) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) as next_order,
    
    -- Percent of customer total
    order_amount / SUM(order_amount) OVER (PARTITION BY customer_id) * 100 as pct_of_total

FROM orders
WHERE order_date >= '2024-01-01';

-- Example 2: Top N per group
WITH ranked_products AS (
    SELECT 
        category,
        product_name,
        revenue,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) as rank
    FROM product_sales
    WHERE sale_date >= CURRENT_DATE() - INTERVAL 30 DAYS
)

SELECT 
    category,
    product_name,
    revenue
FROM ranked_products
WHERE rank <= 5  -- Top 5 products per category
ORDER BY category, rank;

-- ❌ BAD: Self-join for previous value (slow!)
SELECT 
    a.customer_id,
    a.order_date,
    a.amount,
    b.amount as previous_amount
FROM orders a
LEFT JOIN orders b 
    ON a.customer_id = b.customer_id 
    AND b.order_date = (
        SELECT MAX(order_date) 
        FROM orders 
        WHERE customer_id = a.customer_id 
        AND order_date < a.order_date
    );

-- ✅ GOOD: Window function (much faster!)
SELECT 
    customer_id,
    order_date,
    amount,
    LAG(amount) OVER (PARTITION BY customer_id ORDER BY order_date) as previous_amount
FROM orders;
```

### Example 3: Predicate Pushdown and Partition Pruning

```sql
-- Optimize with proper filtering and partitioning

-- ❌ BAD: Filter after join, no partition pruning
SELECT 
    o.order_id,
    o.order_date,
    c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01'  -- Filter AFTER join
  AND c.customer_tier = 'Premium';

-- ✅ GOOD: Filter before join, leverage partition pruning
WITH filtered_orders AS (
    SELECT 
        order_id,
        customer_id,
        order_date,
        order_amount
    FROM orders
    WHERE order_date >= '2024-01-01'  -- Partition pruning (if partitioned by date)
),

premium_customers AS (
    SELECT 
        customer_id,
        customer_name
    FROM customers
    WHERE customer_tier = 'Premium'  -- Filter early
)

SELECT 
    o.order_id,
    o.order_date,
    c.customer_name,
    o.order_amount
FROM filtered_orders o
JOIN premium_customers c ON o.customer_id = c.customer_id;

-- Use EXPLAIN to verify partition pruning
EXPLAIN EXTENDED
SELECT * FROM orders
WHERE order_date >= '2024-01-01'
  AND order_date < '2024-02-01';
-- Look for "PartitionFilters" in the plan

-- Delta Lake optimization with Z-ORDER
OPTIMIZE catalog.schema.orders
ZORDER BY (customer_id, order_date);

-- Now queries filtering on these columns are faster
SELECT * FROM catalog.schema.orders
WHERE customer_id = 'C12345'
  AND order_date >= '2024-01-01';
```

### Example 4: Efficient Joins

```sql
-- Join best practices

-- ❌ BAD: Large table on right side of LEFT JOIN
SELECT 
    small.id,
    large.data
FROM small_table small
LEFT JOIN large_table large ON small.id = large.id;  -- Large table scanned entirely

-- ✅ GOOD: Large table on left
SELECT 
    large.id,
    large.data,
    small.info
FROM large_table large
LEFT JOIN small_table small ON large.id = small.id
WHERE large.date >= '2024-01-01';  -- Filter on large table

-- ❌ BAD: Multiple joins without optimization
SELECT 
    o.*,
    c.name,
    p.name,
    r.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
JOIN regions r ON c.region_id = r.id;

-- ✅ GOOD: Broadcast small dimension tables
SELECT 
    o.*,
    c.name as customer_name,
    p.name as product_name,
    r.name as region_name
FROM orders o
JOIN BROADCAST(customers) c ON o.customer_id = c.id  -- Broadcast hint
JOIN BROADCAST(products) p ON o.product_id = p.id
JOIN BROADCAST(regions) r ON c.region_id = r.id
WHERE o.order_date >= '2024-01-01';

-- ✅ GOOD: Join order - most restrictive first
WITH recent_orders AS (
    SELECT * FROM orders
    WHERE order_date >= '2024-01-01'  -- Filter first (most restrictive)
),

premium_customers AS (
    SELECT * FROM customers
    WHERE customer_tier = 'Premium'  -- Second filter
)

SELECT 
    ro.*,
    pc.customer_name
FROM recent_orders ro
JOIN premium_customers pc ON ro.customer_id = pc.customer_id;
```

### Example 5: Aggregation Best Practices

```sql
-- Efficient aggregation patterns

-- ❌ BAD: Multiple scans of same table
SELECT 
    (SELECT COUNT(*) FROM orders WHERE status = 'completed') as completed_count,
    (SELECT COUNT(*) FROM orders WHERE status = 'pending') as pending_count,
    (SELECT AVG(amount) FROM orders WHERE status = 'completed') as avg_amount;

-- ✅ GOOD: Single scan with conditional aggregation
SELECT 
    COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed_count,
    COUNT(CASE WHEN status = 'pending' THEN 1 END) as pending_count,
    AVG(CASE WHEN status = 'completed' THEN amount END) as avg_amount
FROM orders;

-- ✅ GOOD: Approximate aggregations for large datasets
SELECT 
    date,
    APPROX_COUNT_DISTINCT(customer_id) as unique_customers,  -- Fast approximation
    APPROX_PERCENTILE(order_amount, 0.5) as median_amount    -- Approximate median
FROM orders
WHERE date >= '2024-01-01'
GROUP BY date;

-- ❌ BAD: Redundant GROUP BY
SELECT 
    customer_id,
    region,
    SUM(amount) as total_amount
FROM (
    SELECT 
        customer_id,
        region,
        SUM(line_amount) as amount
    FROM order_lines
    GROUP BY customer_id, region
)
GROUP BY customer_id, region;

-- ✅ GOOD: Single aggregation
SELECT 
    customer_id,
    region,
    SUM(line_amount) as total_amount
FROM order_lines
GROUP BY customer_id, region;
```

### Example 6: Avoiding SQL Anti-Patterns

```sql
-- Common anti-patterns and their fixes

-- ❌ BAD: SELECT *
SELECT * FROM orders
JOIN customers ON orders.customer_id = customers.id;

-- ✅ GOOD: Select only needed columns
SELECT 
    orders.order_id,
    orders.order_date,
    orders.amount,
    customers.customer_name
FROM orders
JOIN customers ON orders.customer_id = customers.id;

-- ❌ BAD: Multiple ORs in WHERE
SELECT * FROM orders
WHERE status = 'completed' 
   OR status = 'shipped' 
   OR status = 'delivered';

-- ✅ GOOD: Use IN
SELECT * FROM orders
WHERE status IN ('completed', 'shipped', 'delivered');

-- ❌ BAD: Function on indexed column
SELECT * FROM orders
WHERE YEAR(order_date) = 2024 
  AND MONTH(order_date) = 3;

-- ✅ GOOD: Range predicate (enables partition pruning)
SELECT * FROM orders
WHERE order_date >= '2024-03-01' 
  AND order_date < '2024-04-01';

-- ❌ BAD: Unnecessary DISTINCT
SELECT DISTINCT customer_id, order_date, amount
FROM orders;  -- If combination is already unique

-- ✅ GOOD: Use GROUP BY if aggregating
SELECT customer_id, order_date, SUM(amount) as total_amount
FROM order_lines
GROUP BY customer_id, order_date;

-- ❌ BAD: Subquery in SELECT for each row
SELECT 
    order_id,
    customer_id,
    (SELECT customer_name FROM customers WHERE id = orders.customer_id) as name
FROM orders;

-- ✅ GOOD: JOIN instead
SELECT 
    o.order_id,
    o.customer_id,
    c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

### Example 7: Delta Lake Specific Optimizations

```sql
-- Leveraging Delta Lake features for SQL optimization

-- 1. MERGE for efficient upserts
MERGE INTO catalog.schema.target_table t
USING catalog.schema.source_updates s
ON t.id = s.id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- 2. Time travel for auditing
SELECT * FROM catalog.schema.orders
VERSION AS OF 100;  -- Specific version

SELECT * FROM catalog.schema.orders
TIMESTAMP AS OF '2024-01-01 00:00:00';  -- Point-in-time

-- 3. Optimize with Z-ORDER
OPTIMIZE catalog.schema.orders
ZORDER BY (customer_id, order_date);

-- 4. VACUUM to clean up old files
VACUUM catalog.schema.orders RETAIN 168 HOURS;  -- Keep 7 days

-- 5. Table constraints
ALTER TABLE catalog.schema.orders
ADD CONSTRAINT valid_amount CHECK (order_amount >= 0);

ALTER TABLE catalog.schema.orders
ADD CONSTRAINT customer_fk FOREIGN KEY (customer_id) REFERENCES customers(id);

-- 6. Change Data Feed for incremental processing
ALTER TABLE catalog.schema.orders
SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');

SELECT * FROM table_changes('catalog.schema.orders', 100, 200)
WHERE _change_type IN ('insert', 'update_postimage');

-- 7. Partition pruning with PARTITION BY
CREATE TABLE partitioned_orders
PARTITIONED BY (order_year, order_month)
AS SELECT *, YEAR(order_date) as order_year, MONTH(order_date) as order_month
FROM orders;

-- Query with partition pruning
SELECT * FROM partitioned_orders
WHERE order_year = 2024 AND order_month = 3;  -- Only scans relevant partitions
```

### Example 8: Liquid Clustering - Next-Gen Table Optimization (2026)

```sql
-- ✅ MODERN: Liquid Clustering (replaces Z-ORDER in 2026)
-- Create table with liquid clustering
CREATE TABLE catalog.schema.customer_orders_clustered
USING DELTA
CLUSTER BY (customer_id, order_date)  -- Liquid clustering keys
AS SELECT * FROM catalog.schema.customer_orders;

-- Query performance automatically optimized
SELECT 
    customer_id,
    order_date,
    SUM(order_amount) as total_amount
FROM catalog.schema.customer_orders_clustered
WHERE customer_id = 'C12345'  -- Efficient point lookup
  AND order_date >= '2024-01-01'  -- Efficient range scan
GROUP BY customer_id, order_date;

-- Enable liquid clustering on existing table
ALTER TABLE catalog.schema.existing_orders
CLUSTER BY (customer_id, order_date);

-- Change clustering keys (flexible! no data rewrite needed)
ALTER TABLE catalog.schema.customer_orders_clustered
CLUSTER BY (region, customer_tier, order_date);  -- New clustering keys

-- Automatic maintenance with Predictive Optimization
ALTER TABLE catalog.schema.customer_orders_clustered
SET TBLPROPERTIES ('delta.autoOptimize.autoCompact' = 'true');

-- ❌ OLD WAY: Z-ORDER (still works, but Liquid Clustering is better)
OPTIMIZE catalog.schema.orders
ZORDER BY (customer_id, order_date);
-- Requires manual re-running, can't change keys without rewrite

-- Performance comparison query
-- Liquid Clustering: 2x faster on large tables with multi-column filters
WITH liquid_perf AS (
    SELECT 
        customer_id,
        COUNT(*) as order_count,
        SUM(amount) as total_amount
    FROM catalog.schema.orders_liquid_clustered
    WHERE customer_tier IN ('Gold', 'Platinum')
      AND order_date >= '2024-01-01'
      AND region = 'US-WEST'
    GROUP BY customer_id
),

zorder_perf AS (
    SELECT 
        customer_id,
        COUNT(*) as order_count,
        SUM(amount) as total_amount
    FROM catalog.schema.orders_zorder
    WHERE customer_tier IN ('Gold', 'Platinum')
      AND order_date >= '2024-01-01'
      AND region = 'US-WEST'
    GROUP BY customer_id
)

SELECT * FROM liquid_perf
LIMIT 100;
```

**Liquid Clustering Benefits:**
* ✅ Automatic incremental clustering (new data automatically optimized)
* ✅ Flexible clustering keys (change without full table rewrite)
* ✅ Better multi-column performance (optimized for multiple filter columns)
* ✅ No manual OPTIMIZE needed (automatic with Predictive Optimization)
* ✅ Up to 2x faster on large tables with complex filters

**When to Use Liquid Clustering:**
* High-cardinality columns (customer_id, product_id)
* Multiple filter columns in queries
* Tables > 1TB where Z-ORDER is slow
* Tables with frequently changing access patterns
* Modern lakehouse architectures (2026+)


### Example 9: Unity Catalog SQL Patterns - Row-Level Security & Masking (2026)

```sql
-- ✅ Unity Catalog: Row-level security with SQL

-- 1. Row-Level Security: Users see only their region's data
CREATE OR REPLACE VIEW catalog.schema.regional_sales_secure AS
SELECT 
    order_id,
    customer_id,
    region,
    order_amount,
    order_date
FROM catalog.schema.all_sales
WHERE region IN (
    -- Dynamic filtering based on user's group membership
    CASE 
        WHEN is_member('us_east_team') THEN 'US-EAST'
        WHEN is_member('us_west_team') THEN 'US-WEST'
        WHEN is_member('eu_team') THEN 'EUROPE'
        WHEN is_member('admin') THEN region  -- Admins see all regions
        ELSE NULL  -- No access for others
    END
);

-- Grant access to the secure view
GRANT SELECT ON VIEW catalog.schema.regional_sales_secure TO `data-analysts@company.com`;

-- Query as a user: automatically filtered to their region
SELECT * FROM catalog.schema.regional_sales_secure
WHERE order_date >= '2024-01-01';

-- 2. Column-Level Masking: Dynamic PII masking
CREATE OR REPLACE FUNCTION catalog.schema.mask_email(email STRING)
RETURNS STRING
RETURN CASE 
    WHEN is_member('pii_readers') THEN email  -- Full access for PII readers
    WHEN is_member('data_analysts') THEN CONCAT(LEFT(email, 3), '***@***.com')  -- Partial mask
    ELSE '***@***.com'  -- Fully masked for others
END;

CREATE OR REPLACE VIEW catalog.schema.customers_masked AS
SELECT 
    customer_id,
    customer_name,
    catalog.schema.mask_email(email) as email,  -- Masked column
    phone,
    CASE 
        WHEN is_member('finance_team') THEN credit_score  -- Finance sees scores
        ELSE NULL  -- Others don't see scores
    END as credit_score,
    region,
    created_date
FROM catalog.schema.customers_raw;

-- 3. Multi-Level Access Control
CREATE OR REPLACE VIEW catalog.schema.orders_governed AS
SELECT 
    order_id,
    customer_id,
    
    -- Mask order amount based on user role
    CASE 
        WHEN is_member('finance_team') THEN order_amount  -- Exact amount
        WHEN is_member('sales_team') THEN ROUND(order_amount, -2)  -- Rounded to $100
        ELSE NULL  -- Hidden from others
    END as order_amount,
    
    -- Filter orders by region and user group
    region,
    order_date
    
FROM catalog.schema.orders_raw
WHERE 
    -- Regional filtering
    (is_member('us_team') AND region LIKE 'US-%')
    OR (is_member('eu_team') AND region = 'EUROPE')
    OR is_member('admin')  -- Admins see all
    
    -- Time-based access (analysts see last 90 days, finance sees all)
    AND (
        is_member('finance_team')
        OR (is_member('data_analysts') AND order_date >= CURRENT_DATE - INTERVAL 90 DAYS)
    );

-- 4. Audit Logging: Track who accessed what data
-- Query system tables to see access patterns
SELECT 
    user_identity.email as user_email,
    request_params.full_name_arg as table_accessed,
    event_time,
    request_params.operation as operation_type
FROM system.access.audit
WHERE action_name = 'getTable'
  AND event_date >= CURRENT_DATE - INTERVAL 7 DAYS
  AND request_params.full_name_arg LIKE 'catalog.schema.%'
ORDER BY event_time DESC
LIMIT 100;

-- 5. Grant Fine-Grained Permissions
-- Grant table-level access
GRANT SELECT ON TABLE catalog.schema.customer_orders TO `analysts@company.com`;

-- Grant column-level access (restrict PII columns)
GRANT SELECT (order_id, order_date, order_amount, region) 
ON TABLE catalog.schema.customer_orders 
TO `contractors@company.com`;

-- Grant row-level access via secure views
GRANT SELECT ON VIEW catalog.schema.regional_sales_secure TO `regional-managers@company.com`;

-- Revoke access
REVOKE SELECT ON TABLE catalog.schema.sensitive_data FROM `contractors@company.com`;
```

**Unity Catalog Security Benefits:**
* ✅ Row-level filtering with is_member() and user identity
* ✅ Column-level masking with SQL functions
* ✅ Dynamic access control based on user groups
* ✅ Audit logging in system.access.audit tables
* ✅ Fine-grained permissions (table, column, row level)
* ✅ Centralized governance across all workspaces

**Best Practices:**
* Use secure views for row-level security
* Create UDFs for reusable masking logic
* Test access patterns with different user roles
* Monitor system.access.audit for compliance
* Document security rules in view comments
* Use groups (not individual users) for permissions


---

## Best Practices

### 1. Column Selection
* Select only needed columns (avoid SELECT *)
* Use column pruning to reduce data read
* Project early in query execution

### 2. Filtering
* Filter as early as possible
* Use partition columns in WHERE clause
* Avoid functions on filter columns
* Use IN instead of multiple ORs

### 3. Joins
* Filter before joining
* Broadcast small tables explicitly
* Use appropriate join types
* Join order matters - most restrictive first

### 4. Aggregations
* Use conditional aggregation for multiple metrics
* APPROX functions for large datasets
* Avoid multiple scans of same table
* Pre-aggregate when possible

### 5. CTEs and Readability
* Use CTEs for complex queries
* Name CTEs descriptively
* Break complex logic into steps
* Comment complex business logic

### 6. Window Functions
* Prefer over self-joins
* Minimize partition columns
* Use appropriate frame specifications
* Consider performance on large windows

---

## Common Anti-Patterns

### ❌ SELECT *
* **Problem**: Reads all columns, slow and wasteful
* **Solution**: Select only needed columns

### ❌ OR in WHERE
* **Problem**: Can't use indexes effectively
* **Solution**: Use IN or UNION ALL

### ❌ Functions on Indexed Columns
* **Problem**: Prevents index/partition pruning
* **Solution**: Use range predicates

### ❌ Multiple Subqueries
* **Problem**: Multiple scans of same data
* **Solution**: Single CTE or JOIN

### ❌ DISTINCT Overuse
* **Problem**: Expensive sorting and deduplication
* **Solution**: Use GROUP BY or deduplicate earlier

---

## Query Optimization Checklist

- [ ] Select only necessary columns
- [ ] Filter early (WHERE before JOIN)
- [ ] Use partition pruning
- [ ] Broadcast small tables
- [ ] Use CTEs for readability
- [ ] Avoid SELECT *
- [ ] Use window functions instead of self-joins
- [ ] Leverage Delta Lake features (MERGE, Z-ORDER)
- [ ] Test with EXPLAIN to verify plan
- [ ] Monitor query performance in SQL warehouse

---

## Related Skills

* [Spark Optimization](../spark-optimization/SKILL.md) - For DataFrame optimizations
* [Performance Tuning](../performance-tuning/SKILL.md) - For bottleneck identification
* [Delta Lake Optimization](../delta-lake-optimization/SKILL.md) - For Delta-specific features

---

**Last Updated**: 2026-04-09
