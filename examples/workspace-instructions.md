# Workspace Instructions for Genie Code
*Enterprise Databricks Standards - Admin Configured*

This file provides workspace-level instructions that apply to all users. These are configured by workspace administrators to ensure consistency, security, and compliance across the organization.

---

## Organization Overview

**Company**: [Your Company Name]
**Cloud Provider**: AWS / Azure / GCP
**Workspace Purpose**: Production data platform for analytics and ML
**Compliance**: SOC2, GDPR, HIPAA (as applicable)

---

## Required Practices (Mandatory)

### Unity Catalog Requirements
* All production tables **MUST** use Unity Catalog
* No direct access to S3/ADLS/GCS buckets - use Unity Catalog volumes
* All external locations must be registered and governed
* Tables without UC will be deprecated Q2 2024

### Security & Access Control
* **Never** hardcode credentials in notebooks or code
* Use Databricks Secrets for all sensitive data
* All production jobs must use service principals, not personal accounts
* Implement least-privilege access - grant only necessary permissions
* Enable audit logging on all production catalogs

### Resource Tagging (Required)
All resources (clusters, jobs, tables, warehouses) must include these tags:
* `team`: Team name (e.g., data-engineering, analytics, ml-platform)
* `project`: Project identifier (e.g., customer-360, fraud-detection)
* `env`: Environment (dev, stg, prod)
* `cost_center`: Finance cost center code
* `owner`: Primary contact email

### Change Management
* All production changes require pull request + approval
* Use Git integration for notebooks (Databricks Repos)
* Tag releases with semantic versioning
* Document breaking changes in commit messages

---

## Catalog & Schema Standards

### Catalog Naming Convention
```
<env>_<team>_<domain>
```

Examples:
* `prod_analytics_customer`
* `dev_ml_features`
* `stg_finance_reporting`

### Schema Organization
Each catalog should follow this schema structure:
* `bronze` - Raw, unprocessed data
* `silver` - Cleansed, deduplicated data
* `gold` - Business-level aggregates
* `sandbox` - Experimental/temporary tables (auto-cleanup enabled)

### Table Naming Convention
```
<domain>_<entity>_<layer>
```

Examples:
* `customer_orders_silver`
* `product_inventory_gold`
* `transaction_events_bronze`

---

## Approved Architectures & Patterns

### Medallion Architecture (Required for Production)
All data pipelines must implement medallion architecture:

1. **Bronze Layer**
   - Raw data ingestion
   - Minimal transformations
   - Preserve source schema
   - Include audit columns: `_ingestion_timestamp`, `_source_file`

2. **Silver Layer**
   - Data quality validation
   - Deduplication
   - Schema standardization
   - Type conversions

3. **Gold Layer**
   - Business aggregations
   - Star/snowflake schemas
   - Optimized for analytics
   - SCD Type 2 for historical tracking

### Data Ingestion Patterns
* **Batch Files**: Use Auto Loader (cloudFiles)
* **Streaming**: Use Delta Lake with checkpointing
* **APIs**: Implement incremental extraction patterns
* **Databases**: Use CDC where available

### Delta Lake Standards
* **All tables** must be Delta Lake format (no Parquet/CSV)
* Enable **Change Data Feed** on transactional tables
* Run **OPTIMIZE** weekly on frequently queried tables
* Configure **VACUUM** retention (30 days minimum for prod)
* Use **Liquid Clustering** for high-cardinality columns (DBR 13.3+)

---

## Compute & Cluster Standards

### Cluster Policies (Enforced)
* **Development**: Single-node or small clusters (max 4 workers)
* **Production**: Autoscaling with min=2, max=determined by workload
* **Spot instances**: Required for batch jobs (80% spot + 20% on-demand min)
* **Auto-termination**: 15 minutes idle for interactive, N/A for jobs

### Approved Runtimes
* **Databricks Runtime**: 13.3 LTS or later
* **ML Runtime**: 13.3 LTS ML or later
* **Photon**: Enabled for SQL-heavy workloads
* **Delta Live Tables**: Use for streaming pipelines

### SQL Warehouses
* **Classic Warehouses**: Being deprecated - migrate to Serverless
* **Serverless SQL**: Preferred for all SQL workloads
* **Warehouse Sizing**: Start with Small, scale up based on usage
* **Auto-stop**: 10 minutes for development, 30 minutes for production

---

## Data Quality & Validation

### Required Data Quality Checks
All pipelines must implement:

1. **Schema Validation**
   - Explicit schema definitions
   - Reject malformed records or quarantine
   - Log schema evolution changes

2. **Completeness Checks**
   - No nulls in required fields
   - Row count validation
   - Expected file/batch arrival times

3. **Accuracy Checks**
   - Value ranges and bounds
   - Referential integrity
   - Business rule validation

4. **Timeliness Checks**
   - Data freshness SLAs
   - Alert on delayed data
   - Monitor end-to-end latency

### Quality Metrics
Store quality metrics in: `prod_monitoring.data_quality.metrics`
```sql
CREATE TABLE IF NOT EXISTS prod_monitoring.data_quality.metrics (
    table_name STRING,
    metric_name STRING,
    metric_value DOUBLE,
    threshold DOUBLE,
    status STRING,
    check_timestamp TIMESTAMP,
    job_id STRING
)
```

---

## Workflow & Orchestration Standards

### Job Configuration
* Use **Databricks Workflows** (not external orchestrators) where possible
* Implement **task dependencies** for complex pipelines
* Configure **retry policies**: 3 attempts with exponential backoff
* Set **timeout limits** to prevent runaway jobs
* Use **job parameters** for environment-specific configs

### Scheduling
* Batch jobs: Schedule during off-peak hours (10 PM - 6 AM UTC)
* Streaming jobs: Continuous with trigger intervals
* Use **Cron expressions** for complex schedules
* Coordinate with other teams for resource-intensive jobs

### Alerting & Monitoring
* **Success notifications**: Log only (no email spam)
* **Failure notifications**: Slack channel + PagerDuty for critical
* **Performance degradation**: Alert if job duration > 150% of baseline
* **Cost alerts**: Alert if DBU spend > budget threshold

---

## ML Operations (MLOps)

### Model Development
* Track all experiments in **MLflow**
* Log hyperparameters, metrics, and datasets
* Use **reproducible environments** (MLflow Models with dependencies)
* Version training data with Delta Lake time travel

### Model Registry
* Register all production models in **Unity Catalog Model Registry**
* Use model **aliases**: `champion`, `challenger`, `archived`
* Document model purpose, features, and limitations
* Implement model **approval workflow** before production deployment

### Model Monitoring
* Track **prediction drift** and **data drift**
* Monitor **model performance** against baseline
* Log predictions for audit and debugging
* Implement **A/B testing** framework for model comparison

### Feature Store
* Use **Unity Catalog Feature Store** for centralized features
* Version feature definitions
* Document feature engineering logic
* Monitor feature quality and freshness

---

## Security & Compliance

### Data Classification
Tag all tables with classification level:
* **Public**: No restrictions
* **Internal**: Employee-only access
* **Confidential**: Restricted access, business-sensitive
* **Restricted**: Highly sensitive (PII, PHI, PCI)

### PII Handling
* Identify and tag PII columns
* Implement **column-level encryption** for sensitive data
* Use **dynamic views** to mask PII in non-production
* Retain PII only as long as legally required

### Audit & Compliance
* Enable **audit logs** on all Unity Catalog objects
* Monitor access patterns for anomalies
* Regular **access reviews** (quarterly)
* Document data lineage for compliance reporting

### Network Security
* Use **Private Link/VNet Injection** for production workspaces
* Restrict public internet access where possible
* Implement **IP allowlists** for workspace access
* Use VPN for sensitive operations

---

## Cost Management

### Spending Limits
* **Development**: $5K/month per team
* **Staging**: $10K/month per team
* **Production**: Budgeted by project, reviewed monthly

### Cost Optimization Requirements
* Use **spot instances** for all batch workloads
* Terminate idle clusters automatically
* Use **Photon** for SQL to reduce DBU consumption
* Implement **query result caching** in SQL warehouses
* Review and remove unused resources monthly

### Cost Monitoring
* Tag all resources for cost allocation
* Monthly cost reports by team/project
* Alert on anomalous spending (>20% increase week-over-week)
* Quarterly cost optimization reviews

---

## Documentation Requirements

### Table Documentation
All production tables must include:
* Table comment describing purpose and usage
* Column comments for all columns
* Data retention policy
* Update frequency and SLA
* Owner/maintainer contact

### Notebook Documentation
* Clear title and purpose
* Input/output datasets
* Dependencies and prerequisites
* Example usage
* Estimated runtime

### Pipeline Documentation
* End-to-end architecture diagram
* Data flow and transformations
* Error handling and recovery procedures
* Monitoring and alerting setup
* Runbook for common issues

---

## Approved Tools & Libraries

### Data Engineering
* **PySpark**: Primary language for transformations
* **Pandas on Spark**: For pandas-like API at scale
* **Delta Lake**: All table storage
* **Auto Loader**: File ingestion
* **Delta Live Tables**: Streaming pipelines

### Analytics
* **SQL**: Databricks SQL for queries
* **dbt**: Approved for transformation layer
* **Lakeview Dashboards**: Standard BI tool
* **Tableau/Power BI**: Connected via SQL Warehouse

### ML & Data Science
* **MLflow**: Experiment tracking and model registry
* **scikit-learn, XGBoost, LightGBM**: Approved ML libraries
* **TensorFlow, PyTorch**: Deep learning frameworks
* **Hyperopt**: Hyperparameter tuning

### Utilities
* **Delta Sharing**: External data sharing
* **Databricks Connect**: Local IDE development
* **Databricks CLI**: Automation and deployment

---

## Testing Requirements

### Development Testing
* Write **unit tests** for transformation functions
* Test with **sample data** before full runs
* Validate **schema compatibility**
* Check **data quality** rules

### Staging Validation
* Run pipeline on **production-like data**
* Validate **performance** meets SLA
* Test **error handling** and recovery
* Verify **monitoring** and alerts work

### Production Deployment
* Blue-green deployment for critical pipelines
* Canary testing for new models
* Rollback procedures documented
* Post-deployment validation

---

## Support & Escalation

### Support Channels
* **Slack**: `#databricks-help` for questions
* **Email**: data-platform-team@company.com
* **Office Hours**: Tuesdays 2-4 PM EST
* **Documentation**: Internal wiki at wiki.company.com/databricks

### Issue Escalation
1. **P3 (Low)**: Create Jira ticket, response within 2 business days
2. **P2 (Medium)**: Slack + Jira, response within 4 hours
3. **P1 (High)**: Slack + page on-call, response within 1 hour
4. **P0 (Critical)**: Page on-call immediately, all-hands response

---

## Workspace-Specific Context

### Common Data Sources
* **Kafka**: `kafka-prod.company.com:9092` (customer events, transactions)
* **S3 Buckets**: `s3://company-data-prod/` (external data catalog location)
* **RDS**: `mysql-prod.company.com` (transactional databases)
* **APIs**: Internal microservices at `api.company.com`

### Shared Catalogs
* `prod_shared_dimensions`: Dimension tables (customers, products, locations)
* `prod_shared_reference`: Reference data (country codes, categories)
* `prod_monitoring`: Logs, metrics, and observability data

### Service Principals
* `sp-data-engineering@company.com`: Batch ETL jobs
* `sp-streaming@company.com`: Real-time streaming jobs
* `sp-ml-training@company.com`: Model training jobs
* `sp-ml-serving@company.com`: Model inference

---

## Changelog

### 2026-04-09
* Added liquid clustering guidance
* Updated to DBR 13.3 LTS
* Mandated Unity Catalog for all new projects

### 2026-01-15
* Introduced workspace skills for Genie Code
* Updated security requirements for SOC2 compliance

### 2025-10-01
* Initial workspace instructions

---

_These instructions are maintained by the Data Platform Team._
_Questions or suggestions? Contact us in #databricks-help_
