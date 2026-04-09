# Unity Catalog Governance Skill

## Purpose
This skill teaches Genie Code best practices for Unity Catalog governance, including access control, data classification, lineage tracking, audit logging, and compliance management on Databricks.

## When to Use
* Setting up Unity Catalog for the first time
* Implementing data governance policies
* Managing user access and permissions
* Tracking data lineage and dependencies
* Implementing data classification and tagging
* Compliance requirements (GDPR, HIPAA, SOC2)
* Auditing data access patterns
* Implementing row and column-level security

## Key Concepts

### 1. Three-Level Namespace
* **Catalog**: Top-level container (e.g., `prod`, `dev`, `staging`)
* **Schema**: Logical grouping within catalog (e.g., `finance`, `marketing`)
* **Table/View**: Actual data objects

### 2. Securable Objects
* Catalogs, Schemas, Tables, Views
* Volumes (for files), Functions, Models
* Connections (for external data sources)
* Each has independent permissions

### 3. Permission Model
* **USAGE**: Browse/discover objects
* **SELECT**: Read data
* **MODIFY**: Update/delete data
* **CREATE**: Create new objects
* **ALL PRIVILEGES**: Full control
* Inheritance from catalog → schema → table

### 4. Data Classification
* Tag sensitivity levels (PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED)
* Track PII (Personally Identifiable Information)
* Implement column-level security
* Mask sensitive data in non-production

---

## Examples

### Example 1: Catalog and Schema Setup

```sql
-- Create catalogs for different environments
CREATE CATALOG IF NOT EXISTS prod_analytics
  COMMENT 'Production analytics data';

CREATE CATALOG IF NOT EXISTS dev_analytics
  COMMENT 'Development analytics data';

-- Create schemas within catalog
CREATE SCHEMA IF NOT EXISTS prod_analytics.bronze
  COMMENT 'Raw ingested data';

CREATE SCHEMA IF NOT EXISTS prod_analytics.silver
  COMMENT 'Cleansed and validated data';

CREATE SCHEMA IF NOT EXISTS prod_analytics.gold
  COMMENT 'Business-level aggregates';

-- Add metadata tags
ALTER CATALOG prod_analytics
  SET TAGS ('env' = 'production', 'team' = 'data-engineering');

ALTER SCHEMA prod_analytics.gold
  SET TAGS ('tier' = 'business', 'sla' = 'high');
```

### Example 2: Grant Permissions

```sql
-- Grant catalog-level permissions
GRANT USAGE ON CATALOG prod_analytics TO `data-analysts`;
GRANT USAGE ON CATALOG dev_analytics TO `data-engineers`;

-- Grant schema-level permissions
GRANT USAGE, SELECT ON SCHEMA prod_analytics.gold TO `data-analysts`;
GRANT ALL PRIVILEGES ON SCHEMA dev_analytics.bronze TO `data-engineers`;

-- Grant table-level permissions
GRANT SELECT ON TABLE prod_analytics.gold.customer_metrics TO `marketing-team`;

-- Grant to service principals for jobs
GRANT USAGE ON CATALOG prod_analytics TO `sp-etl-job@company.com`;
GRANT SELECT, MODIFY ON SCHEMA prod_analytics.silver TO `sp-etl-job@company.com`;

-- View current grants
SHOW GRANTS ON CATALOG prod_analytics;
SHOW GRANTS ON TABLE prod_analytics.gold.customer_metrics;
```

### Example 3: Row-Level Security with Dynamic Views

```python
# Create view with row-level security based on user
spark.sql("""
    CREATE OR REPLACE VIEW prod_analytics.gold.customer_data_secure AS
    SELECT 
        customer_id,
        name,
        email,
        region,
        total_spend
    FROM prod_analytics.gold.customers
    WHERE 
        -- Analysts see only their region
        CASE 
            WHEN IS_MEMBER('regional-analysts') 
            THEN region = (
                SELECT region 
                FROM prod_analytics.metadata.user_regions 
                WHERE user_email = current_user()
            )
            -- Managers see all regions
            WHEN IS_MEMBER('data-managers') THEN TRUE
            ELSE FALSE
        END
""")

# Grant access to the secure view
spark.sql("""
    GRANT SELECT ON VIEW prod_analytics.gold.customer_data_secure 
    TO `regional-analysts`;
""")
```

### Example 4: Column-Level Security and Masking

```sql
-- Create masked view for PII data
CREATE OR REPLACE VIEW prod_analytics.gold.customers_masked AS
SELECT 
    customer_id,
    name,
    -- Mask email for non-privileged users
    CASE 
        WHEN IS_MEMBER('pii-readers') THEN email
        ELSE CONCAT('***@', SPLIT(email, '@')[1])
    END AS email,
    -- Mask phone number
    CASE 
        WHEN IS_MEMBER('pii-readers') THEN phone
        ELSE CONCAT('***-***-', RIGHT(phone, 4))
    END AS phone,
    region,
    total_spend
FROM prod_analytics.gold.customers;

-- Grant access to masked view
GRANT SELECT ON VIEW prod_analytics.gold.customers_masked 
TO `data-analysts`;
```

### Example 5: Data Classification Tagging

```python
from pyspark.sql import SparkSession

def classify_and_tag_tables(catalog: str, schema: str):
    """
    Automatically classify tables and add appropriate tags
    """
    
    # Get all tables in schema
    tables = spark.sql(f"SHOW TABLES IN {catalog}.{schema}")
    
    # PII column patterns
    pii_patterns = ['email', 'phone', 'ssn', 'passport', 'credit_card', 'address']
    
    for table_row in tables.collect():
        table_name = table_row['tableName']
        full_table = f"{catalog}.{schema}.{table_name}"
        
        # Get table columns
        columns_df = spark.sql(f"DESCRIBE TABLE {full_table}")
        columns = [row['col_name'].lower() for row in columns_df.collect()]
        
        # Detect PII columns
        pii_columns = [col for col in columns if any(pattern in col for pattern in pii_patterns)]
        
        # Classify table
        if pii_columns:
            classification = 'CONFIDENTIAL'
            has_pii = 'true'
        else:
            classification = 'INTERNAL'
            has_pii = 'false'
        
        # Apply tags
        spark.sql(f"""
            ALTER TABLE {full_table}
            SET TAGS (
                'classification' = '{classification}',
                'has_pii' = '{has_pii}',
                'pii_columns' = '{",".join(pii_columns) if pii_columns else "none"}'
            )
        """)
        
        # Add table comment if PII detected
        if pii_columns:
            spark.sql(f"""
                COMMENT ON TABLE {full_table} IS 
                'Contains PII data: {", ".join(pii_columns)}. Access restricted.'
            """)
        
        print(f"✓ Tagged {full_table}: {classification}, PII columns: {pii_columns}")


# Usage
classify_and_tag_tables("prod_analytics", "gold")
```

### Example 6: Audit Log Analysis

```sql
-- Query audit logs (requires admin access)
-- Note: Audit logs are in system tables

-- View recent access to sensitive tables
SELECT 
    event_time,
    user_identity.email as user_email,
    request_params.full_name_arg as table_name,
    action_name,
    response.status_code
FROM system.access.audit
WHERE 
    action_name IN ('getTable', 'createTable', 'deleteTable')
    AND request_params.full_name_arg LIKE 'prod_analytics.gold%'
    AND event_date >= CURRENT_DATE() - INTERVAL 7 DAYS
ORDER BY event_time DESC
LIMIT 100;

-- Track who accessed PII data
SELECT 
    user_identity.email,
    COUNT(*) as access_count,
    COUNT(DISTINCT request_params.full_name_arg) as unique_tables
FROM system.access.audit
WHERE 
    request_params.full_name_arg IN (
        SELECT CONCAT(table_catalog, '.', table_schema, '.', table_name)
        FROM system.information_schema.tables
        WHERE table_catalog = 'prod_analytics'
        -- Filter tables tagged with PII
    )
    AND event_date >= CURRENT_DATE() - INTERVAL 30 DAYS
GROUP BY user_identity.email
ORDER BY access_count DESC;
```

---

## Best Practices

### 1. Principle of Least Privilege
* Grant minimum permissions needed
* Use groups instead of individual users
* Review permissions regularly (quarterly)
* Remove access when no longer needed

### 2. Catalog Organization
* Separate catalogs by environment (prod, dev, staging)
* Separate catalogs by team or domain
* Use schemas to organize related data
* Implement naming conventions

### 3. Service Principals for Jobs
* Never use personal accounts for production jobs
* Create dedicated service principals per application
* Document purpose and scope in comments
* Rotate credentials regularly

### 4. Data Classification
* Tag all tables with classification level
* Identify and track PII columns
* Implement masking for non-production environments
* Document classification criteria

### 5. Audit and Monitoring
* Enable audit logging on all catalogs
* Monitor unusual access patterns
* Alert on sensitive data access
* Regular compliance reporting

### 6. Documentation
* Add comments to catalogs, schemas, tables
* Document permission model
* Maintain data dictionary
* Keep runbooks for common operations

---

## Common Pitfalls

### ❌ Granting ALL PRIVILEGES Too Broadly
* **Problem**: Users have more access than needed
* **Solution**: Grant specific permissions (SELECT, MODIFY) instead

### ❌ Not Using Groups
* **Problem**: Managing individual user permissions is unscalable
* **Solution**: Use groups for role-based access control

### ❌ Forgetting USAGE Permission
* **Problem**: Users can't see objects even with SELECT
* **Solution**: Always grant USAGE on catalog and schema

### ❌ No Service Principal for Jobs
* **Problem**: Jobs run as users, break when user leaves
* **Solution**: Use service principals for all automated workloads

### ❌ Not Tagging Data
* **Problem**: Can't identify sensitive data or track compliance
* **Solution**: Implement systematic tagging strategy

### ❌ Ignoring Audit Logs
* **Problem**: Can't investigate incidents or prove compliance
* **Solution**: Regularly review logs, set up alerts

---

## Permission Inheritance Example

```
Catalog: prod_analytics (User has USAGE)
├── Schema: bronze (User has USAGE, SELECT)
│   ├── Table: raw_events (Inherits: can read)
│   └── Table: raw_users (Inherits: can read)
└── Schema: gold (User has no explicit grant)
    └── Table: metrics (Cannot access - no schema permission)
```

---

## Related Skills

* [Medallion Architecture](../medallion-architecture/SKILL.md) - For organizing data by layer
* [Data Quality Checks](../../data-engineering/data-quality-checks/SKILL.md) - For governance implementation

---

**Last Updated**: 2024-04-09
