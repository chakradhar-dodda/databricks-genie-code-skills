# My Databricks Genie Code Instructions

This file provides persistent instructions for Genie Code to understand your preferences, coding style, and project context.

## Quick Start
The fastest way to add instructions is to start your input with the `#` character in Genie Code chat.

---

## Code Style & Preferences

### Language Preferences
- **Primary language**: PySpark (Python)
- **SQL dialect**: Spark SQL
- **Notebook style**: Modular with clear section headers

### Naming Conventions
- **Tables**: `snake_case` (e.g., `customer_orders_silver`)
- **Columns**: `snake_case` (e.g., `order_date`, `customer_id`)
- **Notebooks**: Title Case with context (e.g., "Customer Analytics - ETL Pipeline")
- **Jobs**: Descriptive with env prefix (e.g., `prod_daily_orders_etl`)
- **Functions**: `snake_case` for Python, `camelCase` for Scala

### Code Structure
- Always include docstrings for functions and classes
- Add type hints for Python functions
- Use descriptive variable names (avoid single letters except in lambdas)
- Maximum line length: 100 characters
- Prefer explicit over implicit

---

## Project Context

### Current Projects
**Project 1: Customer Analytics Platform**
- **Catalog**: `prod_analytics`
- **Schemas**: `bronze`, `silver`, `gold`
- **Purpose**: Real-time customer behavior tracking
- **Data Sources**: Kafka streams, REST APIs, S3 buckets
- **Target SLA**: < 5 minute latency for streaming, daily batch for historical

**Project 2: ML Feature Store**
- **Catalog**: `ml_features`
- **Purpose**: Centralized feature engineering and serving
- **Tech Stack**: Feature Store, MLflow, Delta Lake
- **Models**: Customer churn, product recommendations

### Team Standards
- Use medallion architecture (bronze/silver/gold)
- All production tables must be Delta format
- Enable Change Data Feed on transactional tables
- Implement data quality checks at each layer
- Tag all resources: `team`, `project`, `env`, `cost_center`

---

## Data Engineering Preferences

### Delta Lake
- Always use `OPTIMIZE` with `ZORDER` for frequently queried columns
- Run `VACUUM` on schedule (7-day retention minimum)
- Enable Auto Optimize for write-heavy tables
- Use liquid clustering for high-cardinality columns

### Schema Management
- Define schemas explicitly (avoid schema inference in production)
- Use `mergeSchema` option cautiously
- Document schema changes in table properties
- Version control schema definitions

### Incremental Processing
- Prefer Auto Loader for file ingestion
- Use Delta Lake CDF for incremental updates
- Implement watermarking for late-arriving data
- Checkpoint streaming queries every 5 minutes

### Data Quality
- Implement expectations at ingestion (bronze layer)
- Add validation rules in silver layer
- Calculate data quality metrics and store them
- Alert on quality threshold violations

---

## Unity Catalog & Governance

### Security
- Follow principle of least privilege
- Use service principals for production jobs
- Implement row-level and column-level security where needed
- Mask PII in non-production environments

### Catalog Organization
- **Development**: `dev_<team>` catalogs
- **Staging**: `stg_<team>` catalogs
- **Production**: `prod_<team>` catalogs
- Use schemas to separate domains (e.g., `finance`, `marketing`, `operations`)

### Metadata Management
- Add comprehensive table and column comments
- Tag tables with data classification (public, internal, confidential, restricted)
- Document lineage for critical datasets
- Maintain data dictionaries

---

## Performance & Optimization

### Spark Tuning
- Partition tables by date columns (e.g., `PARTITIONED BY (date)`)
- Use broadcast joins for small lookup tables (< 10MB)
- Enable adaptive query execution (AQE)
- Cache DataFrames when reused multiple times

### Cluster Configuration
- **Dev**: Single-node clusters for exploratory work
- **Production**: Autoscaling clusters with appropriate min/max
- Use Photon for SQL-heavy workloads
- Enable spot instances for batch jobs (80% spot + 20% on-demand)

### Cost Optimization
- Schedule jobs during off-peak hours when possible
- Use job compute pools for batch workloads
- Implement automatic cluster termination (15 min idle)
- Monitor DBU consumption by project/team

---

## ML Operations

### Model Development
- Track all experiments in MLflow
- Version datasets used for training
- Log hyperparameters, metrics, and artifacts
- Use reproducible random seeds

### Model Deployment
- Register production models in Model Registry
- Use model aliases (Champion/Challenger)
- Implement A/B testing framework
- Monitor model drift and performance

### Feature Engineering
- Build features as reusable transformations
- Store features in Delta tables with timestamps
- Document feature definitions and logic
- Version feature pipelines

---

## Workflow & Orchestration

### Job Design
- Break complex pipelines into modular tasks
- Implement retry logic with exponential backoff
- Use task dependencies to control execution flow
- Add timeout configurations

### Error Handling
- Log errors with full context (timestamp, job ID, input params)
- Send alerts to Slack/PagerDuty for critical failures
- Implement graceful degradation where possible
- Create runbooks for common failure scenarios

### Testing
- Write unit tests for transformation functions
- Implement data validation tests
- Test with sample data before full production runs
- Use staging environment for integration testing

---

## Documentation Practices

### Notebook Documentation
- Start with an overview markdown cell
- Explain the purpose and business context
- Document input/output datasets
- Include example usage and expected runtime

### Code Comments
- Explain "why" not "what" (code should be self-documenting)
- Document assumptions and edge cases
- Add TODO/FIXME markers for technical debt
- Link to relevant tickets/documentation

### Table Documentation
- Add table comments explaining purpose
- Document column meanings and valid values
- Include update frequency and SLA
- Specify data retention policies

---

## Collaboration & Git

### Version Control
- Commit frequently with descriptive messages
- Use feature branches for new development
- Tag production releases
- Keep notebooks in Repos for Git integration

### Code Review
- Peer review all production code changes
- Check for security vulnerabilities
- Validate performance implications
- Ensure documentation is updated

---

## Preferred Patterns

### When Reading Data
```python
# Read with schema and options
df = (spark.read
    .format("delta")
    .option("mergeSchema", "false")
    .table("catalog.schema.table"))
```

### When Writing Data
```python
# Write with optimizations
(df.write
    .format("delta")
    .mode("append")
    .option("mergeSchema", "false")
    .option("optimizeWrite", "true")
    .partitionBy("date")
    .saveAsTable("catalog.schema.table"))
```

### When Merging Data
```python
# Use Delta Lake MERGE with predicates
(target.alias("t")
    .merge(source.alias("s"), "t.id = s.id")
    .whenMatchedUpdate(set={"col": "s.col"})
    .whenNotMatchedInsert(values={"col": "s.col"})
    .execute())
```

---

## Anti-Patterns to Avoid

- ❌ Using `collect()` on large DataFrames
- ❌ Skipping schema validation
- ❌ Not partitioning large tables
- ❌ Hardcoding credentials
- ❌ Ignoring data quality issues
- ❌ Not monitoring job performance
- ❌ Overusing `cache()` without unpersist
- ❌ Writing to Parquet instead of Delta

---

## Agent Memories

### Learned Preferences
_This section is auto-updated by Genie Code as it learns your patterns_

### Frequent Tasks
_Common operations you perform regularly_

### Project-Specific Notes
_Context that Genie Code has learned about your specific projects_

---

## Communication Style

- Be concise but thorough in explanations
- Provide working code examples, not pseudocode
- Explain trade-offs when multiple approaches exist
- Highlight potential issues and edge cases
- Suggest optimizations when relevant

---

_Last Updated: 2024-04-09_
_This is a living document - update as your preferences and projects evolve_
