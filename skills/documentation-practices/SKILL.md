# Documentation Practices Skill

## Purpose
This skill teaches Genie Code how to create comprehensive documentation for Databricks projects including code comments, README patterns, API documentation, data dictionaries, runbook creation, and knowledge management for maintainable data platforms.

## When to Use
* Starting new data projects
* Onboarding team members
* Documenting pipelines and workflows
* Creating operational runbooks
* Building data catalogs
* Establishing team standards
* Reducing technical debt

## Key Concepts

### 1. Documentation Strategy Framework
* **Situation**: Current state and stakeholders
* **Motivation**: Business context and purpose
* **Restrictions**: Technical constraints and requirements (R)
* **Evaluation**: Success metrics and monitoring (E)
* **Decisions**: Approval workflows and responsibilities (D)

### 2. Documentation Layers
* **Code-level**: Docstrings, comments, type hints
* **Project-level**: README, architecture docs
* **API-level**: Function/class documentation
* **Data-level**: Schema, lineage, quality rules
* **Operational**: Runbooks, troubleshooting guides

### 3. Documentation Types
* **How-to guides**: Step-by-step instructions
* **Tutorials**: Learning-oriented walkthroughs
* **Reference**: Technical specifications
* **Explanations**: Conceptual understanding

### 4. Living Documentation
* Version controlled with code
* Updated with changes
* Reviewed in PRs
* Automatically generated where possible

---

## Examples

### Example 1: Comprehensive README Template

```markdown
# Customer Analytics Pipeline

## Situation
**Owner**: Data Engineering Team  
**Stakeholders**: Analytics, Marketing, Product  
**Dependencies**: CRM system, Orders database, Customer support tickets

## Motivation (Business Purpose)
This pipeline provides unified customer analytics to:
* Enable marketing segmentation and targeting
* Power churn prediction models
* Support customer success dashboards
* Drive product recommendations

**Business Value**: $2M annual revenue impact through improved retention

## Overview
This pipeline processes customer data from multiple sources, applies transformations, and produces analytics-ready datasets following the medallion architecture (Bronze → Silver → Gold).

## Architecture

```
┌──────────┐     ┌────────┐     ┌────────┐     ┌──────┐
│  Source  │────▶│ Bronze │────▶│ Silver │────▶│ Gold │
│ Systems  │     │  Raw   │     │ Clean  │     │Curated│
└──────────┘     └────────┘     └────────┘     └──────┘
   CRM              Delta          Delta         Delta
   Orders           Lake           Lake          Lake
   Support
```

## Data Flow

### Bronze Layer
* **Tables**: `bronze.crm_customers`, `bronze.orders`, `bronze.support_tickets`
* **Update Frequency**: Real-time (streaming)
* **Retention**: 90 days
* **Purpose**: Raw data ingestion with minimal transformation

### Silver Layer
* **Tables**: `silver.customers`, `silver.orders_cleaned`, `silver.tickets_cleaned`
* **Update Frequency**: Every 15 minutes
* **Retention**: 2 years
* **Purpose**: Cleaned, validated, deduplicated data

### Gold Layer
* **Tables**: `gold.customer_360`, `gold.customer_metrics`, `gold.churn_features`
* **Update Frequency**: Hourly
* **Retention**: Indefinite
* **Purpose**: Business-ready aggregations and features

## Restrictions (Technical Requirements)

### Schema Contracts
See [data-contracts/customer_360_contract.py](contracts/customer_360_contract.py)

### Data Quality Rules
* Customer ID: NOT NULL, UNIQUE
* Email: Valid format, NOT NULL
* Lifetime value: >= 0
* Last order date: <= CURRENT_DATE

### SLAs
* **Latency**: Data available within 60 minutes of source update
* **Availability**: 99.5% uptime
* **Freshness**: No data older than 4 hours in gold layer

## Evaluation (Monitoring)

### Key Metrics
* Pipeline success rate: Target 99.9%
* Average execution time: Target < 30 minutes
* Data quality score: Target > 95%

### Monitoring Dashboards
* [Pipeline Health Dashboard](https://databricks.com/dashboards/pipeline-health)
* [Data Quality Dashboard](https://databricks.com/dashboards/data-quality)

### Alerts
* **Critical**: Pipeline failure, SLA violation
* **Warning**: High error rate (>5%), data quality degradation
* **Info**: Long-running jobs, unusual data volumes

## Getting Started

### Prerequisites
* Databricks Runtime 13.3 LTS+
* Unity Catalog access to `main` catalog
* Service principal with contributor role

### Installation

```bash
# Clone repository
git clone https://github.com/company/customer-analytics-pipeline.git
cd customer-analytics-pipeline

# Install dependencies
pip install -r requirements.txt

# Configure
cp config/config.example.yaml config/config.yaml
# Edit config.yaml with your settings
```

### Running the Pipeline

```python
# Run complete pipeline
python src/main.py --env production

# Run specific layer
python src/main.py --layer silver --env production

# Backfill historical data
python src/main.py --backfill --start-date 2024-01-01 --end-date 2024-01-31
```

## Development

### Project Structure
```
customer-analytics-pipeline/
├── src/
│   ├── bronze/          # Bronze layer ingestion
│   ├── silver/          # Silver layer transformations
│   ├── gold/            # Gold layer aggregations
│   ├── utils/           # Shared utilities
│   └── main.py          # Entry point
├── tests/
│   ├── unit/            # Unit tests
│   ├── integration/     # Integration tests
│   └── fixtures/        # Test data
├── config/              # Configuration files
├── docs/                # Additional documentation
├── contracts/           # Data contracts
└── requirements.txt
```

### Testing

```bash
# Run all tests
pytest tests/ -v

# Run unit tests only
pytest tests/unit/ -v

# Run with coverage
pytest tests/ --cov=src --cov-report=html
```

### Code Quality

```bash
# Format code
black src/ tests/

# Lint
pylint src/
flake8 src/

# Type checking
mypy src/
```

## Decisions (Change Management)

### Deployment Process
1. Create feature branch
2. Implement changes + tests
3. Submit PR with documentation updates
4. Code review (2 approvals required)
5. Merge to main
6. Automated tests run
7. Deploy to dev → staging → production

### Breaking Changes
* Require 30-day notice to consumers
* Document in CHANGELOG.md
* Provide migration guide
* Maintain backward compatibility when possible

### On-Call Rotation
* Primary: Data Engineering Team (rotating weekly)
* Secondary: Platform Team
* Escalation: Engineering Manager

## Troubleshooting

### Common Issues

**Pipeline fails with "Table not found"**
* Check Unity Catalog permissions
* Verify table names in config/config.yaml
* Check if source tables exist

**Data quality checks failing**
* Review DLQ table: `monitoring.dlq_customer_events`
* Check source data quality
* Review quality rules in contracts/

**Performance degradation**
* Check cluster size and autoscaling settings
* Review query plans with EXPLAIN
* Check for data skew in join keys

## Related Documentation
* [API Reference](docs/api.md)
* [Data Dictionary](docs/data_dictionary.md)
* [Runbook](docs/runbook.md)
* [Architecture Decision Records](docs/adr/)

## Changelog
See [CHANGELOG.md](CHANGELOG.md)

## Contributors
* John Doe - Tech Lead
* Jane Smith - Data Engineer
* Bob Jones - Data Engineer

## License
MIT License - See [LICENSE](LICENSE)

---
**Last Updated**: 2026-04-09
```

### Example 2: Function and Class Documentation

```python
from typing import List, Optional, Dict
from pyspark.sql import DataFrame
from pyspark.sql.functions import col, when, lit
from datetime import datetime

class CustomerDataProcessor:
    """
    Process customer data through bronze → silver → gold layers.
    
    This class implements the customer analytics pipeline following
    the medallion architecture pattern. It handles data ingestion,
    cleaning, validation, and aggregation.
    
    Attributes:
        spark (SparkSession): Active Spark session
        catalog (str): Unity Catalog name
        environment (str): Deployment environment (dev/staging/prod)
    
    Example:
        >>> processor = CustomerDataProcessor(spark, catalog="main", environment="prod")
        >>> processor.run_bronze_to_silver()
        >>> processor.run_silver_to_gold()
    
    See Also:
        * Data contract: contracts/customer_contract.py
        * Monitoring: dashboards/pipeline_health.py
    """
    
    def __init__(self, spark, catalog: str, environment: str = "production"):
        """
        Initialize customer data processor.
        
        Args:
            spark: Spark session instance
            catalog: Unity Catalog name for tables
            environment: Deployment environment
        
        Raises:
            ValueError: If catalog doesn't exist
            PermissionError: If user lacks catalog access
        """
        self.spark = spark
        self.catalog = catalog
        self.environment = environment
        self._validate_access()
    
    def run_bronze_to_silver(
        self,
        source_table: str,
        target_table: str,
        validation_rules: Optional[List[Dict]] = None
    ) -> DataFrame:
        """
        Transform bronze raw data to silver cleaned data.
        
        Applies data cleaning, validation, and deduplication following
        data quality rules defined in the data contract.
        
        Args:
            source_table: Fully qualified bronze table name
                          (e.g., "main.bronze.customers")
            target_table: Fully qualified silver table name
                          (e.g., "main.silver.customers")
            validation_rules: Optional custom validation rules.
                            If None, uses default contract rules.
        
        Returns:
            DataFrame: Cleaned and validated data
        
        Raises:
            DataQualityError: If validation failure rate > 5%
            TableNotFoundError: If source table doesn't exist
        
        Example:
            >>> result = processor.run_bronze_to_silver(
            ...     source_table="main.bronze.customers",
            ...     target_table="main.silver.customers"
            ... )
            >>> print(f"Processed {result.count()} records")
        
        Business Context:
            * Motivation: Clean data for reliable analytics
            * SLA: Complete within 15 minutes
            * Quality threshold: >95% pass rate
        
        Technical Notes:
            * Uses Delta merge for upserts
            * Partitioned by signup_date
            * Z-ordered by customer_id for query performance
        """
        # Implementation...
        pass
    
    def calculate_customer_lifetime_value(
        self,
        customer_df: DataFrame,
        orders_df: DataFrame,
        lookback_days: int = 365
    ) -> DataFrame:
        """
        Calculate customer lifetime value (CLV) metrics.
        
        Computes total spend, order frequency, and average order value
        for each customer over a specified time window.
        
        Args:
            customer_df: Customer dimension data
            orders_df: Orders fact data
            lookback_days: Number of days to include (default: 365)
        
        Returns:
            DataFrame with columns:
                - customer_id: Unique customer identifier
                - lifetime_value: Total amount spent
                - order_count: Number of orders
                - avg_order_value: Average order amount
                - first_order_date: Date of first order
                - last_order_date: Date of most recent order
        
        Business Rules:
            * Only includes completed orders (status = 'completed')
            * Excludes returns and cancellations
            * Values in USD
        
        Performance:
            * Optimized for broadcast join on customer_df
            * Expected runtime: <5 minutes for 10M orders
        
        Example:
            >>> customers = spark.table("silver.customers")
            >>> orders = spark.table("silver.orders")
            >>> clv = processor.calculate_customer_lifetime_value(
            ...     customers, orders, lookback_days=730
            ... )
            >>> clv.filter(col("lifetime_value") > 10000).count()
            1523
        """
        # Implementation...
        pass
```

### Example 3: Data Dictionary

```markdown
# Data Dictionary: Customer 360 View

## Table: `gold.customer_360`

**Purpose**: Unified customer view combining profile, behavior, and metrics  
**Owner**: Data Engineering Team  
**Update Frequency**: Hourly  
**Retention**: Indefinite  
**Row Count**: ~2.5M (as of 2024-04-01)

### Schema

| Column Name | Data Type | Nullable | Description | Business Definition | Sample Values |
|-------------|-----------|----------|-------------|---------------------|---------------|
| `customer_id` | BIGINT | No | Unique customer identifier | Internal customer ID from CRM system | 12345, 67890 |
| `email` | STRING | No | Customer email address | Primary contact email, validated format | user@example.com |
| `name` | STRING | No | Customer full name | Legal name or preferred name | "John Doe" |
| `signup_date` | DATE | No | Account creation date | UTC date when account was created | 2023-01-15 |
| `status` | STRING | No | Account status | Current account state | "active", "inactive", "suspended" |
| `lifetime_value` | DECIMAL(10,2) | No | Total customer spend | Sum of all completed orders in USD | 1250.50 |
| `order_count` | BIGINT | No | Number of orders | Count of completed orders | 15 |
| `avg_order_value` | DECIMAL(10,2) | Yes | Average order amount | Mean order value in USD | 83.37 |
| `first_order_date` | DATE | Yes | Date of first order | NULL if no orders | 2023-01-20 |
| `last_order_date` | DATE | Yes | Date of last order | NULL if no orders | 2024-03-28 |
| `days_since_last_order` | INT | Yes | Days since last order | CURRENT_DATE - last_order_date | 12 |
| `churn_risk_score` | DOUBLE | Yes | Predicted churn probability | ML model score 0-1, NULL if no model | 0.23 |
| `customer_segment` | STRING | Yes | Marketing segment | Rule-based segmentation | "high_value", "at_risk", "new" |
| `preferred_category` | STRING | Yes | Top product category | Category with most orders | "electronics" |
| `support_ticket_count` | INT | No | Number of support tickets | Count of all tickets | 3 |
| `nps_score` | INT | Yes | Net Promoter Score | Last survey response 0-10, NULL if no response | 9 |
| `updated_at` | TIMESTAMP | No | Last update timestamp | UTC timestamp of last pipeline run | 2024-04-01 14:30:00 |

### Constraints

* **Primary Key**: `customer_id`
* **Foreign Keys**: 
  * References `silver.customers(customer_id)`
  * References `silver.orders(customer_id)`
* **Check Constraints**:
  * `lifetime_value >= 0`
  * `order_count >= 0`
  * `churn_risk_score BETWEEN 0 AND 1`
  * `status IN ('active', 'inactive', 'suspended')`

### Partitioning & Optimization

* **Partitioned By**: DATE(signup_date)
* **Z-Ordered By**: customer_id, status
* **Liquid Clustering**: customer_segment (coming soon)

### Data Lineage

```
crm.customers ────┐
                  ├──▶ bronze.customers ──▶ silver.customers ────┐
                  │                                               │
orders_db.orders ─┤                                               ├──▶ gold.customer_360
                  ├──▶ bronze.orders ──▶ silver.orders ───────────┤
                  │                                               │
support.tickets ──┘                                               │
ml_models.churn_predictions ──────────────────────────────────────┘
```

### Usage Examples

```sql
-- High-value customers at risk of churn
SELECT customer_id, name, lifetime_value, churn_risk_score
FROM gold.customer_360
WHERE lifetime_value > 5000
  AND churn_risk_score > 0.7
  AND days_since_last_order > 60
ORDER BY churn_risk_score DESC;

-- Customer segmentation summary
SELECT 
    customer_segment,
    COUNT(*) as customer_count,
    AVG(lifetime_value) as avg_ltv,
    AVG(order_count) as avg_orders
FROM gold.customer_360
WHERE status = 'active'
GROUP BY customer_segment;
```

### Change Log

| Date | Version | Changes | Author |
|------|---------|---------|--------|
| 2024-04-01 | 2.1.0 | Added `nps_score` column | J. Doe |
| 2024-03-15 | 2.0.0 | Added `churn_risk_score`, updated schema | J. Smith |
| 2024-01-10 | 1.0.0 | Initial version | B. Jones |

---
**Last Updated**: 2026-04-09
```

### Example 4: Operational Runbook

```markdown
# Runbook: Customer Analytics Pipeline

## Purpose
This runbook provides operational procedures for monitoring, troubleshooting, and maintaining the customer analytics pipeline.

## On-Call Information
* **Primary**: Data Engineering Team (rotating weekly)
* **Secondary**: Platform Team  
* **Escalation**: Engineering Manager
* **Slack Channel**: #data-eng-oncall
* **PagerDuty**: data-eng-oncall@company.pagerduty.com

## Monitoring & Alerts

### Key Dashboards
1. [Pipeline Health](https://databricks.com/dashboards/pipeline-health)
2. [Data Quality](https://databricks.com/dashboards/data-quality)
3. [SLA Compliance](https://databricks.com/dashboards/sla-compliance)

### Alert Severity Levels

**CRITICAL** (Page on-call immediately)
* Pipeline failure for > 1 hour
* Data freshness SLA violation (>4 hours old)
* Data quality score < 90%

**WARNING** (Notify team channel)
* High error rate (>5%)
* Performance degradation (>2x normal duration)
* Data quality score 90-95%

**INFO** (Log only)
* Pipeline completed
* Unusual data volume (±50% of normal)

## Common Issues & Resolution

### Issue: Pipeline Failure - "Table Not Found"

**Symptoms**:
* Alert: "Pipeline failed with TableNotFoundException"
* Log message: "Table 'main.bronze.customers' not found"

**Diagnosis**:
```sql
-- Check if table exists
SHOW TABLES IN main.bronze LIKE 'customers';

-- Check Unity Catalog permissions
SHOW GRANTS ON TABLE main.bronze.customers;
```

**Resolution**:
1. Verify table exists in Unity Catalog
2. Check service principal has SELECT permission
3. If table missing, check upstream ingestion job
4. If permissions issue, request access via ServiceNow

**Prevention**:
* Monitor upstream table availability
* Add pre-flight checks before pipeline runs

---

### Issue: Data Quality Degradation

**Symptoms**:
* Alert: "Data quality score below threshold"
* Many records in DLQ table

**Diagnosis**:
```sql
-- Check DLQ for common errors
SELECT 
    error_type,
    COUNT(*) as error_count,
    MIN(ingestion_timestamp) as first_seen
FROM monitoring.dlq_customer_events
WHERE DATE(ingestion_timestamp) = CURRENT_DATE()
GROUP BY error_type
ORDER BY error_count DESC;

-- Check source data quality
SELECT 
    COUNT(*) as total_records,
    SUM(CASE WHEN email IS NULL THEN 1 ELSE 0 END) as null_emails,
    SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) as null_ids
FROM bronze.customers
WHERE DATE(ingestion_timestamp) = CURRENT_DATE();
```

**Resolution**:
1. Identify root cause from DLQ analysis
2. If source data issue:
   * Contact source system owner
   * Document issue in incident ticket
   * Decide if pipeline should continue or block
3. If pipeline logic issue:
   * Create hotfix PR
   * Deploy to production after testing
4. Reprocess failed records:
```python
# Reprocess DLQ records
dlq_df = spark.table("monitoring.dlq_customer_events") \\
    .filter(col("ingestion_timestamp") >= "2024-04-01")

# Apply fixes and reprocess
fixed_df = apply_fixes(dlq_df)
write_to_silver(fixed_df)
```

**Prevention**:
* Implement data quality checks at source
* Add schema evolution support
* Set up source data monitoring

---

### Issue: Performance Degradation

**Symptoms**:
* Alert: "Pipeline duration exceeded threshold"
* Job running >2x normal duration

**Diagnosis**:
```sql
-- Check recent execution times
SELECT 
    DATE(start_time) as date,
    AVG(duration_minutes) as avg_duration,
    MAX(duration_minutes) as max_duration
FROM monitoring.pipeline_runs
WHERE pipeline = 'customer_analytics'
  AND start_time >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY DATE(start_time)
ORDER BY date DESC;

-- Check data volumes
SELECT 
    DATE(timestamp) as date,
    COUNT(*) as record_count
FROM bronze.customers
WHERE timestamp >= CURRENT_DATE() - INTERVAL 7 DAYS
GROUP BY DATE(timestamp)
ORDER BY date DESC;
```

**Resolution**:
1. Check for data skew:
```python
# Analyze partition sizes
df = spark.table("bronze.customers")
df.groupBy("signup_date").count().orderBy(col("count").desc()).show()
```

2. Check cluster configuration:
   * Verify autoscaling is enabled
   * Check if cluster is hitting resource limits
   * Consider increasing cluster size

3. Optimize queries:
```python
# Review query plans
spark.conf.set("spark.sql.adaptive.enabled", "true")
df.explain("cost")
```

4. Apply optimizations:
   * Add OPTIMIZE and ZORDER commands
   * Adjust partition strategy
   * Cache intermediate results if needed

**Prevention**:
* Regular table optimization
* Monitor data growth trends
* Implement partition pruning

---

## Maintenance Procedures

### Weekly Tasks
* Review error logs and DLQ records
* Check pipeline performance trends
* Verify backup completion
* Update documentation if needed

### Monthly Tasks
* Review and optimize Delta tables:
```sql
OPTIMIZE main.gold.customer_360 ZORDER BY (customer_id);
VACUUM main.gold.customer_360 RETAIN 168 HOURS; -- 7 days
```
* Analyze cost and resource usage
* Review and update SLA thresholds
* Conduct runbook review with team

### Quarterly Tasks
* Full DR (Disaster Recovery) test
* Review and update data contracts
* Security audit
* Performance benchmarking

---

##Escalation Matrix

| Issue Severity | Response Time | Escalation Path |
|----------------|---------------|-----------------|
| CRITICAL | 15 minutes | On-call → Secondary → Manager → VP |
| WARNING | 2 hours | On-call → Secondary → Manager |
| INFO | Next business day | On-call handles |

---

## Contact Information

| Role | Name | Email | Slack | Phone |
|------|------|-------|-------|-------|
| Tech Lead | John Doe | john@company.com | @johndoe | +1-555-0001 |
| Data Engineer | Jane Smith | jane@company.com | @janesmith | +1-555-0002 |
| Platform Lead | Bob Jones | bob@company.com | @bobjones | +1-555-0003 |

---
**Last Updated**: 2026-04-09
```

### Example 5: Architecture Decision Record (ADR)

```markdown
# ADR-003: Adopt Medallion Architecture for Customer Analytics

## Status
**Accepted** - 2024-01-15

## Context
We need to design a scalable, maintainable data architecture for our growing customer analytics needs.

**Current Challenges**:
* Data quality issues in downstream analytics
* Difficulty debugging pipeline failures
* Lack of reusability across teams
* Unclear data lineage

**Stakeholders**:
* Data Engineering Team
* Analytics Team
* ML Platform Team
* BI Team

## Decision

We will adopt the **Medallion Architecture** pattern with three layers:

### Bronze Layer (Raw Data)
* **Purpose**: Ingest raw data with minimal transformation
* **Restrictions**: Schema-on-read, no filtering, full history
* **Format**: Delta Lake
* **Retention**: 90 days

### Silver Layer (Cleaned Data)
* **Purpose**: Cleaned, deduplicated, validated data
* **Restrictions**: Schema enforcement, quality checks, business rules
* **Format**: Delta Lake with constraints
* **Retention**: 2 years

### Gold Layer (Business-Ready)
* **Purpose**: Aggregated, enriched, feature-engineered data
* **Restrictions**: Optimized for analytics/ML consumption
* **Format**: Delta Lake with Z-ordering
* **Retention**: Indefinite

## Consequences

### Positive
* ✅ **Data Quality**: Clear separation of raw vs. clean data
* ✅ **Reusability**: Silver layer shared across teams
* ✅ **Debugging**: Easy to trace issues to specific layer
* ✅ **Compliance**: Raw data retained for audit
* ✅ **Performance**: Gold layer optimized for queries

### Negative
* ❌ **Storage Costs**: 3x data storage (bronze/silver/gold)
* ❌ **Complexity**: More pipeline stages to maintain
* ❌ **Latency**: Additional processing hops

### Mitigation Strategies
* Use retention policies to manage storage costs
* Automate layer transitions with orchestration
* Implement streaming where low latency needed

## Evaluation Metrics
* **Data Quality Score**: Target >95% (vs. 78% current)
* **Pipeline Reliability**: Target 99.9% uptime
* **Query Performance**: Target <5s p95 (vs. 45s current)
* **Development Velocity**: Target 50% reduction in time to analytics

## Alternatives Considered

### Option 1: Single Table Architecture
* **Pros**: Simple, low storage cost
* **Cons**: Poor separation of concerns, hard to debug
* **Rejected**: Doesn't solve current challenges

### Option 2: Data Vault
* **Pros**: Highly normalized, audit trail
* **Cons**: Complex, steep learning curve, query performance
* **Rejected**: Overengineered for our needs

## Related Decisions
* [ADR-001: Adopt Delta Lake](adr-001-delta-lake.md)
* [ADR-002: Unity Catalog Migration](adr-002-unity-catalog.md)

## References
* [Databricks Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
* Team discussion: #data-arch channel, 2024-01-10

---
**Author**: John Doe  
**Reviewers**: Jane Smith, Bob Jones  
**Last Updated**: 2026-04-09
```

---

## Best Practices

### 1. Documentation Strategy
* **Situation**: Document current state
* **Motivation**: Explain business value
* **Restrictions**: Technical requirements (R)
* **Evaluation**: Success metrics (E)
* **Decisions**: Approval processes (D)

### 2. Code Documentation
* Write docstrings for all public functions/classes
* Include type hints
* Document parameters, returns, and exceptions
* Provide usage examples

### 3. Project Documentation
* README first approach
* Keep docs next to code
* Update docs with code changes
* Review docs in PRs

### 4. Living Documentation
* Version control all documentation
* Generate docs from code when possible
* Keep documentation DRY (Don't Repeat Yourself)
* Archive outdated docs

### 5. Knowledge Sharing
* Document decisions in ADRs
* Maintain runbooks for operations
* Create onboarding guides
* Record tribal knowledge

---

## Related Skills

* [Data Contracts](../data-contracts/SKILL.md) - For contract documentation
* [Testing Strategies](../testing-strategies/SKILL.md) - For test documentation
* [Monitoring Observability](../monitoring-observability/SKILL.md) - For operational docs

---

**Last Updated**: 2026-04-09
