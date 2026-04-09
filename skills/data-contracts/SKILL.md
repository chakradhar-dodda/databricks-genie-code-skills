# Data Contracts Skill

## Purpose
This skill teaches Genie Code how to define, enforce, and manage data contracts in Databricks using schema enforcement, SLAs, data quality contracts, API contracts, contract testing, and versioning for reliable data products.

## When to Use
* Defining data product interfaces
* Establishing producer-consumer agreements
* Enforcing schema evolution policies
* Setting data quality expectations
* Managing breaking changes
* Implementing data mesh architecture
* Creating reusable data assets
* Documenting data dependencies

## Key Concepts

### 1. Data Contract Elements 
* **Situation**: Current data landscape and stakeholders
* **Motivation**: Business value and use cases
* **Restrictions**: Schema, quality rules, SLAs (R = Restrictions)
* **Evaluation**: Quality metrics and monitoring (E = Evaluation)
* **Decisions**: Approval workflows and change management (D = Decisions)

### 2. Contract Types
* **Schema contracts**: Column names, types, constraints
* **Quality contracts**: Completeness, accuracy, timeliness
* **SLA contracts**: Availability, freshness, latency
* **Semantic contracts**: Business rules, transformations
* **API contracts**: Input/output specifications

### 3. Enforcement Mechanisms
* **Schema evolution**: Controlled changes
* **Check constraints**: Delta table constraints
* **Expectations**: Great Expectations integration
* **Validation**: Pre-write checks

### 4. Contract Lifecycle
* **Definition**: Document requirements
* **Implementation**: Enforce in code
* **Testing**: Validate compliance
* **Monitoring**: Track violations
* **Evolution**: Manage changes

---

## Examples

### Example 1: Comprehensive Data Contract Definition

```python
from dataclasses import dataclass
from typing import List, Dict
from enum import Enum

class SLATier(Enum):
    CRITICAL = "critical"  # 99.9% availability, <15 min freshness
    HIGH = "high"          # 99.5% availability, <1 hour freshness
    STANDARD = "standard"  # 99% availability, <4 hours freshness

@dataclass
class ColumnContract:
    """Contract for individual column"""
    name: str
    data_type: str
    nullable: bool
    description: str
    constraints: List[str]  # e.g., ["CHECK (amount >= 0)"]
    business_definition: str

@dataclass
class QualityRule:
    """Data quality expectation"""
    rule_name: str
    rule_type: str  # "completeness", "accuracy", "consistency"
    expectation: str  # e.g., "customer_id must be unique"
    threshold: float  # e.g., 0.99 for 99% compliance
    severity: str  # "blocker", "warning", "info"

@dataclass
class DataContract:
    """
    Complete data contract specification
    
    Framework:
    - Situation: Captures current state and stakeholders
    - Motivation: Documents business value
    - Restrictions: Enforces schema and quality (R)
    - Evaluation: Defines SLAs and metrics (E)
    - Decisions: Specifies approval process (D)
    """
    
    # Situation
    contract_name: str
    version: str
    owner_team: str
    consumers: List[str]
    
    # Motivation
    business_purpose: str
    use_cases: List[str]
    
    # Restrictions (R)
    table_name: str
    schema: List[ColumnContract]
    partition_columns: List[str]
    quality_rules: List[QualityRule]
    
    # Evaluation (E)
    sla_tier: SLATier
    freshness_sla_minutes: int
    availability_sla_pct: float
    
    # Decisions (D)
    approval_required_for_changes: bool
    breaking_change_notification_days: int
    
    # Metadata
    created_date: str
    last_updated: str


# Example contract
customer_events_contract = DataContract(
    # Situation
    contract_name="customer_events",
    version="2.1.0",
    owner_team="data-engineering",
    consumers=["analytics-team", "ml-platform", "bi-dashboards"],
    
    # Motivation
    business_purpose="Track customer interactions for analytics and ML models",
    use_cases=[
        "Customer segmentation for marketing campaigns",
        "Churn prediction models",
        "Real-time dashboards for customer success"
    ],
    
    # Restrictions (R)
    table_name="gold.customer_events",
    schema=[
        ColumnContract(
            name="event_id",
            data_type="STRING",
            nullable=False,
            description="Unique event identifier",
            constraints=["NOT NULL", "PRIMARY KEY"],
            business_definition="UUID v4 generated at event time"
        ),
        ColumnContract(
            name="customer_id",
            data_type="BIGINT",
            nullable=False,
            description="Customer identifier",
            constraints=["NOT NULL", "FOREIGN KEY customers(id)"],
            business_definition="Internal customer ID from CRM system"
        ),
        ColumnContract(
            name="event_type",
            data_type="STRING",
            nullable=False,
            description="Type of customer event",
            constraints=["CHECK (event_type IN ('login', 'purchase', 'support_ticket'))"],
            business_definition="Categorizes customer interaction type"
        ),
        ColumnContract(
            name="event_timestamp",
            data_type="TIMESTAMP",
            nullable=False,
            description="When the event occurred",
            constraints=["NOT NULL"],
            business_definition="UTC timestamp of event occurrence"
        ),
        ColumnContract(
            name="amount",
            data_type="DECIMAL(10,2)",
            nullable=True,
            description="Transaction amount for purchase events",
            constraints=["CHECK (amount IS NULL OR amount >= 0)"],
            business_definition="USD amount for purchase events, NULL for others"
        )
    ],
    partition_columns=["DATE(event_timestamp)"],
    
    quality_rules=[
        QualityRule(
            rule_name="unique_event_ids",
            rule_type="accuracy",
            expectation="event_id must be unique across all records",
            threshold=1.0,
            severity="blocker"
        ),
        QualityRule(
            rule_name="valid_customer_ids",
            rule_type="consistency",
            expectation="customer_id must exist in customers table",
            threshold=0.999,
            severity="blocker"
        ),
        QualityRule(
            rule_name="no_future_events",
            rule_type="accuracy",
            expectation="event_timestamp must be <= current_timestamp",
            threshold=1.0,
            severity="blocker"
        ),
        QualityRule(
            rule_name="completeness",
            rule_type="completeness",
            expectation="No NULL values in required columns",
            threshold=0.999,
            severity="warning"
        )
    ],
    
    # Evaluation (E)
    sla_tier=SLATier.HIGH,
    freshness_sla_minutes=60,
    availability_sla_pct=99.5,
    
    # Decisions (D)
    approval_required_for_changes=True,
    breaking_change_notification_days=30,
    
    created_date="2024-01-15",
    last_updated="2024-04-01"
)

# Generate contract documentation
def generate_contract_docs(contract: DataContract) -> str:
    """Generate markdown documentation from contract"""
    
    docs = f"""# Data Contract: {contract.contract_name} v{contract.version}

## Situation
**Owner**: {contract.owner_team}  
**Consumers**: {', '.join(contract.consumers)}

## Motivation
**Purpose**: {contract.business_purpose}

**Use Cases**:
{chr(10).join(f'- {uc}' for uc in contract.use_cases)}

## Restrictions (Contract Terms)

### Schema
Table: `{contract.table_name}`

| Column | Type | Nullable | Constraints | Business Definition |
|--------|------|----------|-------------|---------------------|
"""
    
    for col in contract.schema:
        docs += f"| {col.name} | {col.data_type} | {'Yes' if col.nullable else 'No'} | {', '.join(col.constraints)} | {col.business_definition} |\n"
    
    docs += f"\n### Quality Rules\n\n"
    for rule in contract.quality_rules:
        docs += f"- **{rule.rule_name}** ({rule.rule_type}): {rule.expectation}\n"
        docs += f"  - Threshold: {rule.threshold*100}%\n"
        docs += f"  - Severity: {rule.severity}\n\n"
    
    docs += f"""## Evaluation (SLAs)
- **Tier**: {contract.sla_tier.value}
- **Freshness**: Data available within {contract.freshness_sla_minutes} minutes
- **Availability**: {contract.availability_sla_pct}% uptime

## Decisions (Change Management)
- Breaking changes require **{contract.breaking_change_notification_days} days notice**
- Schema changes require approval: **{'Yes' if contract.approval_required_for_changes else 'No'}**

---
*Last Updated*: {contract.last_updated}
"""
    
    return docs

# Save contract documentation
contract_docs = generate_contract_docs(customer_events_contract)
dbutils.fs.put(
    "/contracts/customer_events_v2.1.0.md",
    contract_docs,
    overwrite=True
)
print("✅ Contract documentation generated")
```

### Example 2: Schema Enforcement with Delta Constraints

```python
from pyspark.sql.types import *
from pyspark.sql.functions import col, current_timestamp

def create_table_with_contract(contract: DataContract):
    """
    Create Delta table with enforced schema and constraints
    """
    
    # Build schema from contract
    schema = StructType([
        StructField(
            col.name,
            eval(col.data_type + "()"),  # e.g., StringType()
            col.nullable
        )
        for col in contract.schema
    ])
    
    # Create table with schema
    spark.sql(f"""
        CREATE TABLE IF NOT EXISTS {contract.table_name} (
            {', '.join(f"{col.name} {col.data_type}" + (" NOT NULL" if not col.nullable else "")
                      for col in contract.schema)},
            _contract_version STRING,
            _ingestion_timestamp TIMESTAMP
        )
        USING DELTA
        PARTITIONED BY ({', '.join(contract.partition_columns)})
        TBLPROPERTIES (
            'delta.dataSkippingNumIndexedCols' = '5',
            'delta.enableChangeDataFeed' = 'true',
            'delta.columnMapping.mode' = 'name',
            'contract.version' = '{contract.version}',
            'contract.owner' = '{contract.owner_team}'
        )
    """)
    
    # Add check constraints from contract
    for col_contract in contract.schema:
        for constraint in col_contract.constraints:
            if constraint.startswith("CHECK"):
                constraint_name = f"{col_contract.name}_constraint"
                
                spark.sql(f"""
                    ALTER TABLE {contract.table_name}
                    ADD CONSTRAINT {constraint_name}
                    {constraint}
                """)
                
                print(f"✅ Added constraint: {constraint_name}")
    
    print(f"✅ Created table with contract: {contract.table_name}")

# Apply contract
create_table_with_contract(customer_events_contract)

# Write data with validation
def write_with_contract_validation(df, contract: DataContract):
    """Write data only if it meets contract requirements"""
    
    # Validate schema
    expected_cols = {col.name for col in contract.schema}
    actual_cols = set(df.columns)
    
    missing_cols = expected_cols - actual_cols
    if missing_cols:
        raise ValueError(f"Missing required columns: {missing_cols}")
    
    extra_cols = actual_cols - expected_cols
    if extra_cols:
        print(f"⚠️ Extra columns will be ignored: {extra_cols}")
        df = df.select(*expected_cols)
    
    # Add metadata
    df_with_metadata = (
        df
        .withColumn("_contract_version", lit(contract.version))
        .withColumn("_ingestion_timestamp", current_timestamp())
    )
    
    # Write with automatic constraint checking
    try:
        (
            df_with_metadata
            .write
            .format("delta")
            .mode("append")
            .saveAsTable(contract.table_name)
        )
        print(f"✅ Data written successfully to {contract.table_name}")
    
    except Exception as e:
        if "CHECK constraint" in str(e):
            print(f"❌ Data violates contract constraints: {e}")
            # Send to DLQ for investigation
            send_to_dlq(df, contract.table_name, str(e))
        raise

# Usage
new_events_df = spark.read.parquet("s3://incoming/events/")
write_with_contract_validation(new_events_df, customer_events_contract)
```

### Example 3: Quality Contract Validation

```python
class ContractValidator:
    """Validate data against contract quality rules"""
    
    def __init__(self, contract: DataContract):
        self.contract = contract
        self.results = []
    
    def validate(self, df):
        """Run all quality validations"""
        
        print(f"Validating against contract: {self.contract.contract_name} v{self.contract.version}")
        
        for rule in self.contract.quality_rules:
            result = self._validate_rule(df, rule)
            self.results.append(result)
            
            # Block if blocker rule fails
            if rule.severity == "blocker" and not result['passed']:
                raise ValueError(
                    f"Blocker quality rule failed: {rule.rule_name}\n"
                    f"Expected: {rule.expectation}\n"
                    f"Actual compliance: {result['compliance']:.2%}"
                )
        
        return self._generate_report()
    
    def _validate_rule(self, df, rule: QualityRule):
        """Validate single quality rule"""
        
        if rule.rule_type == "completeness":
            return self._check_completeness(df, rule)
        elif rule.rule_type == "accuracy":
            return self._check_accuracy(df, rule)
        elif rule.rule_type == "consistency":
            return self._check_consistency(df, rule)
        else:
            raise ValueError(f"Unknown rule type: {rule.rule_type}")
    
    def _check_completeness(self, df, rule):
        """Check for completeness (no nulls)"""
        total_rows = df.count()
        
        # Check all non-nullable columns
        non_nullable_cols = [
            col.name for col in self.contract.schema 
            if not col.nullable
        ]
        
        null_counts = df.select([
            count(when(col(c).isNull(), 1)).alias(c) 
            for c in non_nullable_cols
        ]).collect()[0]
        
        total_nulls = sum(null_counts[c] for c in non_nullable_cols)
        compliance = 1 - (total_nulls / (total_rows * len(non_nullable_cols)))
        
        return {
            'rule_name': rule.rule_name,
            'compliance': compliance,
            'threshold': rule.threshold,
            'passed': compliance >= rule.threshold,
            'details': f"{total_nulls} nulls found in {len(non_nullable_cols)} required columns"
        }
    
    def _check_accuracy(self, df, rule):
        """Check for accuracy rules"""
        if "unique" in rule.expectation:
            # Check uniqueness
            col_name = rule.expectation.split()[0]  # Extract column name
            total_rows = df.count()
            distinct_rows = df.select(col_name).distinct().count()
            compliance = distinct_rows / total_rows
            
            return {
                'rule_name': rule.rule_name,
                'compliance': compliance,
                'threshold': rule.threshold,
                'passed': compliance >= rule.threshold,
                'details': f"{total_rows - distinct_rows} duplicate values"
            }
        
        elif "future" in rule.expectation:
            # Check no future timestamps
            future_count = df.filter(col("event_timestamp") > current_timestamp()).count()
            total_rows = df.count()
            compliance = 1 - (future_count / total_rows)
            
            return {
                'rule_name': rule.rule_name,
                'compliance': compliance,
                'threshold': rule.threshold,
                'passed': compliance >= rule.threshold,
                'details': f"{future_count} future timestamps found"
            }
    
    def _check_consistency(self, df, rule):
        """Check referential integrity"""
        if "exist in" in rule.expectation:
            # Extract table name from expectation
            ref_table = rule.expectation.split("exist in ")[1].split()[0]
            
            # Check foreign key validity
            invalid_count = (
                df.join(
                    spark.table(ref_table),
                    df.customer_id == spark.table(ref_table).id,
                    "left_anti"
                )
                .count()
            )
            
            total_rows = df.count()
            compliance = 1 - (invalid_count / total_rows)
            
            return {
                'rule_name': rule.rule_name,
                'compliance': compliance,
                'threshold': rule.threshold,
                'passed': compliance >= rule.threshold,
                'details': f"{invalid_count} invalid references to {ref_table}"
            }
    
    def _generate_report(self):
        """Generate validation report"""
        report = {
            'contract': self.contract.contract_name,
            'version': self.contract.version,
            'timestamp': datetime.now().isoformat(),
            'rules_passed': sum(1 for r in self.results if r['passed']),
            'rules_failed': sum(1 for r in self.results if not r['passed']),
            'results': self.results
        }
        
        # Log to monitoring table
        self._log_validation(report)
        
        return report
    
    def _log_validation(self, report):
        """Log validation results"""
        spark.createDataFrame([
            (
                report['timestamp'],
                report['contract'],
                report['version'],
                report['rules_passed'],
                report['rules_failed'],
                str(report['results'])
            )
        ], ["timestamp", "contract", "version", "rules_passed", "rules_failed", "details"]
        ).write.mode("append").saveAsTable("monitoring.contract_validations")

# Usage
validator = ContractValidator(customer_events_contract)

# Validate incoming data
incoming_df = spark.table("bronze.raw_customer_events")
validation_report = validator.validate(incoming_df)

print(f"✅ Validation complete: {validation_report['rules_passed']}/{len(validation_report['results'])} rules passed")
```

### Example 4: Contract Versioning and Evolution

```python
class ContractVersionManager:
    """Manage contract versions and schema evolution"""
    
    def __init__(self):
        self.contracts_table = "governance.data_contracts"
    
    def register_contract(self, contract: DataContract):
        """Register new contract version"""
        
        contract_record = spark.createDataFrame([
            (
                contract.contract_name,
                contract.version,
                contract.owner_team,
                json.dumps([c.name for c in contract.schema]),
                json.dumps([{
                    'name': c.name,
                    'type': c.data_type,
                    'nullable': c.nullable
                } for c in contract.schema]),
                datetime.now(),
                "active"
            )
        ], ["name", "version", "owner", "columns", "schema_json", "registered_at", "status"])
        
        contract_record.write.mode("append").saveAsTable(self.contracts_table)
        print(f"✅ Registered contract: {contract.contract_name} v{contract.version}")
    
    def check_breaking_changes(self, old_contract: DataContract, new_contract: DataContract):
        """Detect breaking schema changes"""
        
        old_cols = {col.name: col for col in old_contract.schema}
        new_cols = {col.name: col for col in new_contract.schema}
        
        breaking_changes = []
        
        # Check for removed columns
        removed = set(old_cols.keys()) - set(new_cols.keys())
        if removed:
            breaking_changes.append(f"Removed columns: {removed}")
        
        # Check for type changes
        for col_name in old_cols.keys() & new_cols.keys():
            old_col = old_cols[col_name]
            new_col = new_cols[col_name]
            
            if old_col.data_type != new_col.data_type:
                breaking_changes.append(
                    f"Type changed: {col_name} ({old_col.data_type} -> {new_col.data_type})"
                )
            
            if not old_col.nullable and new_col.nullable:
                # Making nullable is non-breaking
                pass
            elif old_col.nullable and not new_col.nullable:
                breaking_changes.append(
                    f"Nullability changed: {col_name} (nullable -> non-nullable)"
                )
        
        return breaking_changes
    
    def propose_schema_change(self, new_contract: DataContract):
        """Propose contract update with approval workflow"""
        
        # Get current version
        current_version = (
            spark.table(self.contracts_table)
            .filter(col("name") == new_contract.contract_name)
            .filter(col("status") == "active")
            .orderBy(col("version").desc())
            .first()
        )
        
        if current_version:
            old_contract = self._load_contract(current_version)
            breaking_changes = self.check_breaking_changes(old_contract, new_contract)
            
            if breaking_changes:
                print("⚠️ BREAKING CHANGES DETECTED:")
                for change in breaking_changes:
                    print(f"  - {change}")
                
                # Notify consumers
                for consumer in new_contract.consumers:
                    notify_team(
                        consumer,
                        f"Breaking changes proposed for {new_contract.contract_name}",
                        breaking_changes,
                        notification_days=new_contract.breaking_change_notification_days
                    )
                
                return "approval_required"
            else:
                print("✅ No breaking changes - safe to deploy")
                return "auto_approved"
        else:
            print("New contract - no previous version to compare")
            return "new_contract"

# Usage
version_manager = ContractVersionManager()

# Register initial contract
version_manager.register_contract(customer_events_contract)

# Propose schema update
updated_contract = customer_events_contract
updated_contract.version = "2.2.0"
updated_contract.schema.append(
    ColumnContract(
        name="device_type",
        data_type="STRING",
        nullable=True,
        description="Device used for event",
        constraints=[],
        business_definition="Mobile, desktop, or tablet"
    )
)

status = version_manager.propose_schema_change(updated_contract)
print(f"Change status: {status}")
```

### Example 5: SLA Monitoring (Evaluation)

```python
# Monitor contract SLAs
def monitor_contract_slas(contract: DataContract):
    """
    Monitor SLA compliance for data contract 
    """
    
    # Check freshness SLA
    freshness_query = f"""
    SELECT 
        MAX(event_timestamp) as latest_event,
        TIMESTAMPDIFF(MINUTE, MAX(event_timestamp), CURRENT_TIMESTAMP()) as minutes_old
    FROM {contract.table_name}
    """
    
    freshness_result = spark.sql(freshness_query).collect()[0]
    minutes_old = freshness_result['minutes_old']
    
    freshness_sla_met = minutes_old <= contract.freshness_sla_minutes
    
    print(f"Freshness SLA: {contract.freshness_sla_minutes} minutes")
    print(f"Actual freshness: {minutes_old} minutes")
    print(f"Status: {'✅ MET' if freshness_sla_met else '❌ VIOLATED'}")
    
    # Check availability (uptime)
    availability_query = """
    WITH hourly_checks AS (
        SELECT 
            DATE_TRUNC('hour', timestamp) as hour,
            COUNT(*) as successful_writes
        FROM monitoring.pipeline_runs
        WHERE pipeline = :pipeline_name
          AND timestamp >= CURRENT_TIMESTAMP() - INTERVAL 30 DAYS
          AND status = 'SUCCESS'
        GROUP BY DATE_TRUNC('hour', timestamp)
    ),
    all_hours AS (
        SELECT SEQUENCE(
            DATE_TRUNC('hour', CURRENT_TIMESTAMP() - INTERVAL 30 DAYS),
            DATE_TRUNC('hour', CURRENT_TIMESTAMP()),
            INTERVAL 1 HOUR
        ) as hour
    )
    
    SELECT 
        COUNT(CASE WHEN h.successful_writes > 0 THEN 1 END) * 100.0 / COUNT(*) as availability_pct
    FROM all_hours a
    LEFT JOIN hourly_checks h ON a.hour = h.hour
    """
    
    availability_result = spark.sql(availability_query, pipeline_name=contract.table_name).collect()[0]
    availability_pct = availability_result['availability_pct']
    
    availability_sla_met = availability_pct >= contract.availability_sla_pct
    
    print(f"Availability SLA: {contract.availability_sla_pct}%")
    print(f"Actual availability: {availability_pct:.2f}%")
    print(f"Status: {'✅ MET' if availability_sla_met else '❌ VIOLATED'}")
    
    # Log SLA compliance
    sla_record = spark.createDataFrame([
        (
            datetime.now(),
            contract.contract_name,
            contract.version,
            "freshness",
            freshness_sla_met,
            minutes_old,
            contract.freshness_sla_minutes
        ),
        (
            datetime.now(),
            contract.contract_name,
            contract.version,
            "availability",
            availability_sla_met,
            availability_pct,
            contract.availability_sla_pct
        )
    ], ["timestamp", "contract", "version", "sla_type", "met", "actual_value", "sla_value"])
    
    sla_record.write.mode("append").saveAsTable("monitoring.sla_compliance")
    
    return freshness_sla_met and availability_sla_met

# Run SLA monitoring
all_slas_met = monitor_contract_slas(customer_events_contract)

if not all_slas_met:
    send_alert("SLA violation detected for customer_events contract")
```

---

## Best Practices

### 1. Contract Design 
* Document **Situation** (stakeholders, dependencies)
* Clarify **Motivation** (business value)
* Define **Restrictions** (schema, quality, constraints)
* Specify **Evaluation** metrics (SLAs, monitoring)
* Establish **Decision** workflows (approvals, changes)

### 2. Schema Management
* Use semantic versioning (major.minor.patch)
* Require approval for breaking changes
* Provide migration guides for consumers
* Support backward compatibility when possible

### 3. Quality Rules
* Start with critical rules (blockers)
* Set realistic thresholds (not 100%)
* Monitor rule violations over time
* Adjust rules based on data patterns

### 4. SLA Definition
* Align SLAs with business needs
* Make SLAs measurable and monitorable
* Document SLA calculation methods
* Review SLAs quarterly

### 5. Communication
* Notify consumers of changes early
* Maintain contract documentation
* Provide self-service contract discovery
* Hold regular contract review meetings

---

## Related Skills

* [Monitoring Observability](../monitoring-observability/SKILL.md) - For contract monitoring
* [Testing Strategies](../testing-strategies/SKILL.md) - For contract testing
* [Documentation Practices](../documentation-practices/SKILL.md) - For contract docs

---

**Last Updated**: 2026-04-09
