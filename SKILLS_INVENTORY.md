# Databricks Genie Code Skills Inventory

## Complete Skills Reference Guide

This document provides a comprehensive inventory of all 22 skills available in the databricks-genie-code-skills repository, organized by category with detailed coverage, dependencies, and integration patterns.

> **📊 Quick Stats:** 22 complete skills | 14,841 lines | 8 categories | Production-tested

---

## ✅ All Skills Complete (22 Skills)

### 🔧 Data Engineering (5 skills | 3,063 lines)
**High-volume data processing, ingestion, and pipeline patterns**

#### 1. Delta Lake Optimization
**Path**: `skills/delta-lake-optimization/SKILL.md`  
**Lines**: 681 | **Examples**: 6+  
**Coverage**: OPTIMIZE, VACUUM, Z-ORDER, Liquid Clustering, Predictive I/O, file size analysis, auto-optimization  
**Dependencies**: None (foundational)  
**Links To**: → All data pipeline skills, Performance Tuning, Cost Optimization  
**Use Cases**: Table maintenance, query acceleration, storage optimization

#### 2. Auto Loader
**Path**: `skills/auto-loader/SKILL.md`  
**Lines**: 593 | **Examples**: 7+  
**Coverage**: Schema inference/evolution, Unity Catalog Volumes, serverless ingestion, file notification mode, multi-format support, partition discovery, rescue columns  
**Dependencies**: None (foundational)  
**Links To**: → Incremental Processing, Streaming Pipelines, Medallion Architecture  
**Use Cases**: Cloud file ingestion, CDC from files, bulk data loads

#### 3. Streaming Pipelines
**Path**: `skills/streaming-pipelines/SKILL.md`  
**Lines**: 644 | **Examples**: 7+  
**Coverage**: Structured Streaming, Delta Live Tables (DLT), exactly-once semantics, watermarking, stateful processing, stream-stream joins, Kafka integration, serverless DLT  
**Dependencies**: Auto Loader, Incremental Processing  
**Links To**: → Monitoring & Observability, Error Handling, Data Quality Checks  
**Use Cases**: Real-time analytics, event processing, CDC streaming

#### 4. Incremental Processing
**Path**: `skills/incremental-processing/SKILL.md`  
**Lines**: 593 | **Examples**: 6+  
**Coverage**: Change Data Feed (CDF), Unity Catalog CDC, watermarking, MERGE operations, SCD Type 2, timestamp-based loads, serverless incremental jobs  
**Dependencies**: Delta Lake Optimization  
**Links To**: → Streaming Pipelines, Data Quality Checks, Medallion Architecture  
**Use Cases**: Efficient updates, CDC patterns, historical tracking

#### 5. Data Quality Checks
**Path**: `skills/data-quality-checks/SKILL.md`  
**Lines**: 552 | **Examples**: 5+  
**Coverage**: Validation framework, Lakehouse Monitoring, completeness/accuracy checks, DLT expectations, anomaly detection, drift detection, metrics tracking, custom quality metrics  
**Dependencies**: Delta Lake Optimization  
**Links To**: → Data Contracts, Monitoring & Observability, Testing Strategies  
**Use Cases**: Data validation, quality monitoring, anomaly detection

---

### 🏛️ Governance & Architecture (3 skills | 1,669 lines)
**Security, compliance, and architectural patterns**

#### 6. Unity Catalog Governance
**Path**: `skills/unity-catalog-governance/SKILL.md`  
**Lines**: 348 | **Examples**: 6+  
**Coverage**: Access control, row/column-level security, data classification, audit logs (system.access.audit), lineage, data discovery, compliance patterns  
**Dependencies**: None (foundational)  
**Links To**: → Medallion Architecture, Data Contracts, Monitoring & Observability  
**Use Cases**: Data governance, security enforcement, compliance tracking

#### 7. Data Contracts
**Path**: `skills/data-contracts/SKILL.md`  
**Lines**: 824 | **Examples**: 5+  
**Coverage**: Schema enforcement, validation, evolution patterns, SLAs, quality contracts, versioning, contract testing, producer-consumer agreements  
**Dependencies**: Data Quality Checks, Data Modeling  
**Links To**: → Monitoring & Observability, Testing Strategies, Documentation Practices  
**Use Cases**: Schema validation, API contracts, team agreements

#### 8. Medallion Architecture
**Path**: `skills/medallion-architecture/SKILL.md`  
**Lines**: 497 | **Examples**: 4+  
**Coverage**: Bronze/Silver/Gold layers, multi-hop pipelines, quality gates, orchestration patterns, quarantine handling  
**Dependencies**: Data Quality Checks, Incremental Processing  
**Links To**: → Workflow Orchestration, Data Contracts, Unity Catalog Governance  
**Use Cases**: Lakehouse design, data layering, quality enforcement

---

### ⚡ Performance & Optimization (3 skills | 2,604 lines)
**Cost reduction and query acceleration techniques**

#### 9. Spark Optimization
**Path**: `skills/spark-optimization/SKILL.md`  
**Lines**: 960 | **Examples**: 8+  
**Coverage**: Partitioning strategies, caching, broadcast joins, Photon engine best practices, Catalyst optimizer, AQE, shuffle optimization, data skew handling  
**Dependencies**: Delta Lake Optimization  
**Links To**: → Performance Tuning, SQL Best Practices, Cost Optimization  
**Use Cases**: Query acceleration, large-scale joins, shuffle optimization

#### 10. Cost Optimization
**Path**: `skills/cost-optimization/SKILL.md`  
**Lines**: 825 | **Examples**: 7+  
**Coverage**: Cluster sizing, spot instances, serverless economics, Photon engine, cost monitoring, budget alerts, ROI analysis, autoscaling  
**Dependencies**: Performance Tuning, Spark Optimization  
**Links To**: → Monitoring & Observability, Job Scheduling  
**Use Cases**: Cost reduction, resource efficiency, budget management

#### 11. Performance Tuning
**Path**: `skills/performance-tuning/SKILL.md`  
**Lines**: 819 | **Examples**: 7+  
**Coverage**: Query plan analysis, memory tuning, bottleneck identification, profiling, GC tuning, resource monitoring, spill reduction  
**Dependencies**: Spark Optimization  
**Links To**: → Cost Optimization, Monitoring & Observability  
**Use Cases**: Troubleshooting slow queries, memory optimization, performance analysis

---

### 📊 Analytics & Data Modeling (2 skills | 1,315 lines)
**SQL patterns and dimensional modeling**

#### 12. SQL Best Practices
**Path**: `skills/sql-best-practices/SKILL.md`  
**Lines**: 826 | **Examples**: 7+  
**Coverage**: Query optimization, CTEs, window functions, Liquid Clustering (replaces Z-ORDER), Unity Catalog SQL patterns (row-level security, masking), anti-patterns, predicate pushdown  
**Dependencies**: None (foundational)  
**Links To**: → Spark Optimization, Performance Tuning, Data Modeling  
**Use Cases**: Query optimization, SQL development, analytics

#### 13. Data Modeling
**Path**: `skills/data-modeling/SKILL.md`  
**Lines**: 489 | **Examples**: 3+  
**Coverage**: Star schema, SCD Type 2, normalization techniques, dimensional modeling (star/snowflake), Data Vault (Hubs/Links/Satellites)  
**Dependencies**: Delta Lake Optimization  
**Links To**: → Data Contracts, Incremental Processing, SQL Best Practices  
**Use Cases**: Warehouse design, dimensional modeling, reporting layers

---

### 🤖 ML Operations (3 skills | 2,282 lines)
**End-to-end machine learning lifecycle**

#### 14. Feature Engineering
**Path**: `skills/feature-engineering/SKILL.md`  
**Lines**: 822 | **Examples**: 8+  
**Coverage**: Feature Store, real-time feature serving (<10ms latency), transformations, categorical encoding, time-based features, scaling, interactions, feature selection  
**Dependencies**: Data Quality Checks, Incremental Processing  
**Links To**: → MLflow Tracking, Model Deployment  
**Use Cases**: Feature pipelines, feature serving, ML data preparation

#### 15. Model Deployment
**Path**: `skills/model-deployment/SKILL.md`  
**Lines**: 758 | **Examples**: 6+  
**Coverage**: Batch inference, REST APIs, serverless model serving, Model Registry, scale-to-zero, GPU support, A/B testing, drift detection  
**Dependencies**: MLflow Tracking  
**Links To**: → Monitoring & Observability, Error Handling, Job Scheduling  
**Use Cases**: Model serving, inference at scale, production deployment

#### 16. MLflow Tracking
**Path**: `skills/mlflow-tracking/SKILL.md`  
**Lines**: 702 | **Examples**: 8+  
**Coverage**: Experiment tracking, MLflow 2.x features, Unity Catalog models, model aliases (champion/challenger), hyperparameter tuning, enhanced autologging, artifact management  
**Dependencies**: Feature Engineering  
**Links To**: → Model Deployment, Monitoring & Observability  
**Use Cases**: Experiment management, model versioning, training tracking

---

### 🔄 Orchestration & Workflow (3 skills | 2,058 lines)
**Job scheduling and dependency management**

#### 17. Workflow Orchestration
**Path**: `skills/workflow-orchestration/SKILL.md`  
**Lines**: 750 | **Examples**: 7+  
**Coverage**: Multi-task jobs, dependencies, serverless workflows, parameter passing, conditional execution, fan-out/fan-in patterns  
**Dependencies**: Medallion Architecture  
**Links To**: → Job Scheduling, Error Handling, Monitoring & Observability  
**Use Cases**: Pipeline orchestration, task dependencies, complex workflows

#### 18. Job Scheduling
**Path**: `skills/job-scheduling/SKILL.md`  
**Lines**: 631 | **Examples**: 7+  
**Coverage**: Cron triggers, serverless job scheduling, event-driven patterns, continuous jobs, conditional execution, SLA management, job pools  
**Dependencies**: Workflow Orchestration  
**Links To**: → Error Handling, Monitoring & Observability, Cost Optimization  
**Use Cases**: Scheduled pipelines, trigger-based jobs, automated workflows

#### 19. Error Handling
**Path**: `skills/error-handling/SKILL.md`  
**Lines**: 677 | **Examples**: 6+  
**Coverage**: Retry strategies, circuit breakers, alerting, try-catch patterns, dead letter queues, failure recovery patterns  
**Dependencies**: Workflow Orchestration  
**Links To**: → Monitoring & Observability, Testing Strategies  
**Use Cases**: Fault tolerance, error recovery, alerting

---

### 🔍 Quality & Monitoring (2 skills | 957 lines)
**Observability and testing patterns**

#### 20. Monitoring & Observability
**Path**: `skills/monitoring-observability/SKILL.md`  
**Lines**: 427 | **Examples**: 6+  
**Coverage**: Metrics collection, Unity Catalog audit logging (system.access.audit, system.query.history), Lakehouse Monitoring integration, alerting, dashboards, SLO tracking, compliance monitoring  
**Dependencies**: Data Contracts, Error Handling  
**Links To**: → Testing Strategies, Documentation Practices  
**Use Cases**: System monitoring, audit tracking, performance tracking

#### 21. Testing Strategies
**Path**: `skills/testing-strategies/SKILL.md`  
**Lines**: 530 | **Examples**: 6+  
**Coverage**: Unit tests, integration tests, data validation, E2E pipeline testing, regression testing, CI/CD integration  
**Dependencies**: Data Quality Checks, Monitoring & Observability  
**Links To**: → Documentation Practices, Error Handling, Data Contracts  
**Use Cases**: Quality assurance, automated testing, validation

---

### 📚 Best Practices (1 skill | 893 lines)
**Standards and documentation patterns**

#### 22. Documentation Practices
**Path**: `skills/documentation-practices/SKILL.md`  
**Lines**: 893 | **Examples**: 5+  
**Coverage**: Code documentation, notebook standards, knowledge sharing, README patterns, API docs, runbooks, data dictionaries, ADRs (Architecture Decision Records)  
**Dependencies**: Data Contracts, Testing Strategies  
**Links To**: → Monitoring & Observability (for runbooks)  
**Use Cases**: Knowledge management, team documentation, standards

---

## 📊 Repository Statistics

| Metric | Value |
|--------|-------|
| **Total Skills** | 22 |
| **Complete Skills** | 22 |
| **Total Lines** | 14,841 |
| **Total Examples** | 130+ |
| **Average Lines/Skill** | 674 |
| **Categories** | 8 |
| **Last Updated** | 2026-04-09 |

### Lines by Category

| Category | Skills | Lines | % of Total |
|----------|--------|-------|------------|
| Data Engineering | 5 | 3,063 | 20.6% |
| Performance & Optimization | 3 | 2,604 | 17.5% |
| ML Operations | 3 | 2,282 | 15.4% |
| Orchestration & Workflow | 3 | 2,058 | 13.9% |
| Governance & Architecture | 3 | 1,669 | 11.2% |
| Analytics & Data Modeling | 2 | 1,315 | 8.9% |
| Quality & Monitoring | 2 | 957 | 6.4% |
| Best Practices | 1 | 893 | 6.0% |

---

## 🔗 Skill Dependency Graph

```
Foundation Layer (No Dependencies)
├── Auto Loader (593 lines)
├── Delta Lake Optimization (681 lines)
├── SQL Best Practices (826 lines)
└── Unity Catalog Governance (348 lines)

Data Pipeline Layer
├── Incremental Processing (593 lines)
│   ├── Depends on: Delta Lake Optimization
│   └── Enables: Streaming Pipelines, Data Quality Checks, Medallion Architecture
│
├── Streaming Pipelines (644 lines)
│   ├── Depends on: Auto Loader, Incremental Processing
│   └── Enables: Monitoring & Observability, Error Handling
│
└── Data Quality Checks (552 lines)
    ├── Depends on: Delta Lake Optimization
    └── Enables: Data Contracts, Medallion Architecture, Feature Engineering

Architecture Layer
├── Medallion Architecture (497 lines)
│   ├── Depends on: Data Quality Checks, Incremental Processing
│   └── Enables: Workflow Orchestration
│
├── Data Modeling (489 lines)
│   ├── Depends on: Delta Lake Optimization
│   └── Enables: Data Contracts
│
└── Data Contracts (824 lines)
    ├── Depends on: Data Quality Checks, Data Modeling
    └── Enables: Monitoring & Observability, Testing Strategies, Documentation

Optimization Layer
├── Spark Optimization (960 lines)
│   ├── Depends on: Delta Lake Optimization
│   └── Enables: Performance Tuning, SQL Best Practices
│
├── Performance Tuning (819 lines)
│   ├── Depends on: Spark Optimization
│   └── Enables: Cost Optimization
│
└── Cost Optimization (825 lines)
    ├── Depends on: Performance Tuning, Spark Optimization
    └── Enables: Job Scheduling

ML Operations Layer
├── Feature Engineering (822 lines)
│   ├── Depends on: Data Quality Checks, Incremental Processing
│   └── Enables: MLflow Tracking
│
├── MLflow Tracking (702 lines)
│   ├── Depends on: Feature Engineering
│   └── Enables: Model Deployment
│
└── Model Deployment (758 lines)
    ├── Depends on: MLflow Tracking
    └── Enables: Monitoring & Observability, Job Scheduling

Orchestration Layer
├── Workflow Orchestration (750 lines)
│   ├── Depends on: Medallion Architecture
│   └── Enables: Job Scheduling, Error Handling
│
├── Job Scheduling (631 lines)
│   ├── Depends on: Workflow Orchestration, Cost Optimization
│   └── Enables: Error Handling, Monitoring & Observability
│
└── Error Handling (677 lines)
    ├── Depends on: Workflow Orchestration
    └── Enables: Monitoring & Observability, Testing Strategies

Operations Layer
├── Monitoring & Observability (427 lines)
│   ├── Depends on: Data Contracts, Error Handling
│   └── Enables: Testing Strategies, Documentation Practices
│
├── Testing Strategies (530 lines)
│   ├── Depends on: Data Quality Checks, Monitoring & Observability
│   └── Enables: Documentation Practices
│
└── Documentation Practices (893 lines)
    └── Depends on: Data Contracts, Testing Strategies
```

---

## 🎓 Learning Paths

### 🌱 Path 1: Beginner - Data Engineering Foundation
**Duration**: 2-3 weeks | **Lines**: 2,556  
**Goal**: Build foundation in data engineering and analytics

1. **SQL Best Practices** (826 lines) - Query fundamentals and optimization
2. **Delta Lake Optimization** (681 lines) - Lakehouse storage layer
3. **Medallion Architecture** (497 lines) - Bronze/Silver/Gold patterns
4. **Data Quality Checks** (552 lines) - Validation and monitoring

**Project**: Build a bronze→silver→gold pipeline with quality checks

---

### 🚀 Path 2: Intermediate - Production Pipelines
**Duration**: 4-6 weeks | **Lines**: 3,752  
**Goal**: Create reliable, scalable data pipelines

1. **Incremental Processing** (593 lines) - Efficient batch and streaming patterns
2. **Auto Loader** (593 lines) - Schema inference and UC Volumes
3. **Streaming Pipelines** (644 lines) - Structured Streaming and DLT
4. **Unity Catalog Governance** (348 lines) - Access control and lineage
5. **Data Contracts** (824 lines) - Schema enforcement and validation
6. **Workflow Orchestration** (750 lines) - Multi-task job dependencies

**Project**: Streaming pipeline with SLAs, monitoring, and alerting

---

### ⚡ Path 3: Advanced - Performance & Cost Optimization
**Duration**: 3-4 weeks | **Lines**: 3,708  
**Goal**: Optimize for cost, performance, and scale

1. **Spark Optimization** (960 lines) - Partitioning, caching, broadcast joins
2. **Performance Tuning** (819 lines) - Query plan analysis, memory tuning
3. **Cost Optimization** (825 lines) - Cluster sizing, serverless economics
4. **Error Handling** (677 lines) - Retry strategies, circuit breakers
5. **Monitoring & Observability** (427 lines) - Metrics, audit logs, SLO tracking

**Project**: Optimize existing pipeline to reduce costs by 30%

---

### 🤖 Path 4: ML Engineering - End-to-End ML Operations
**Duration**: 3-4 weeks | **Lines**: 3,443  
**Goal**: Deploy and manage production ML systems

1. **Feature Engineering** (822 lines) - Feature Store, real-time serving
2. **MLflow Tracking** (702 lines) - Experiment tracking, model registry
3. **Model Deployment** (758 lines) - Batch/real-time inference, serverless
4. **Job Scheduling** (631 lines) - Training orchestration, retraining triggers
5. **Testing Strategies** (530 lines) - Model validation, integration testing

**Project**: End-to-end ML pipeline with monitoring and retraining

---

### 🏢 Path 5: Enterprise - Platform Architecture
**Duration**: 2-3 weeks | **Lines**: 2,339  
**Goal**: Establish organizational standards and practices

1. **Documentation Practices** (893 lines) - Code docs, notebooks, knowledge
2. **Data Modeling** (489 lines) - Dimensional modeling, SCD Type 2
3. **Testing Strategies** (530 lines) - Unit/integration testing, CI/CD
4. **Monitoring & Observability** (427 lines) - Team dashboards, compliance

**Project**: Design multi-team data platform with governance

---

### 📊 Path 6: Complete Mastery - All Skills
**Duration**: 12-16 weeks | **Lines**: 14,841  
**Goal**: Comprehensive expertise across all domains

Complete all 22 skills in recommended order by category:
1. Data Engineering (5 skills, 3,063 lines)
2. Governance & Architecture (3 skills, 1,669 lines)
3. Analytics & Data Modeling (2 skills, 1,315 lines)
4. Performance & Optimization (3 skills, 2,604 lines)
5. Orchestration & Workflow (3 skills, 2,058 lines)
6. ML Operations (3 skills, 2,282 lines)
7. Quality & Monitoring (2 skills, 957 lines)
8. Best Practices (1 skill, 893 lines)

**Project**: Enterprise lakehouse platform from scratch

---

## 🔄 Cross-Skill Integration Patterns

### Pattern 1: Complete Data Pipeline
```
Auto Loader (593) → Data Quality Checks (552) → 
Incremental Processing (593) → Delta Lake Optimization (681) → 
Medallion Architecture (497) → Workflow Orchestration (750) → 
Monitoring & Observability (427)
```
**Total**: 4,093 lines | **Use Case**: Production ETL/ELT pipeline

---

### Pattern 2: ML Feature Pipeline
```
Data Quality Checks (552) → Feature Engineering (822) → 
MLflow Tracking (702) → Model Deployment (758) → 
Monitoring & Observability (427)
```
**Total**: 3,261 lines | **Use Case**: ML model lifecycle

---

### Pattern 3: Streaming Analytics
```
Auto Loader (593) → Streaming Pipelines (644) → 
Data Quality Checks (552) → Error Handling (677) → 
Monitoring & Observability (427)
```
**Total**: 2,893 lines | **Use Case**: Real-time event processing

---

### Pattern 4: Cost-Optimized Production Pipeline
```
Spark Optimization (960) → Performance Tuning (819) → 
Cost Optimization (825) → Job Scheduling (631) → 
Monitoring & Observability (427)
```
**Total**: 3,662 lines | **Use Case**: Efficient resource utilization

---

### Pattern 5: Enterprise Data Platform
```
Unity Catalog Governance (348) → Data Contracts (824) → 
Medallion Architecture (497) → Testing Strategies (530) → 
Documentation Practices (893) → Monitoring & Observability (427)
```
**Total**: 3,519 lines | **Use Case**: Governed multi-team platform

---

## 💡 Using Skills with Genie Code

### Automatic Skill Selection
Once skills are installed in `~/.assistant/skills/`, Genie Code automatically references relevant skills based on:

* **Keywords in your question** (Delta, Spark, MLflow, Unity Catalog, etc.)
* **Context from your notebook** (existing code, table references, imports)
* **Task type** (optimization, deployment, governance, orchestration)
* **Best practices** (automatically applies patterns from multiple skills)

### Example Questions by Category

#### Data Ingestion
* "Set up Auto Loader for JSON files from cloud storage"
* "How do I handle schema evolution with rescue columns?"
* "Configure file notification mode for AWS S3"

#### Data Quality
* "Validate data completeness and accuracy in my pipeline"
* "Implement DLT expectations for data quality"
* "Create anomaly detection rules using Lakehouse Monitoring"

#### Performance Optimization
* "Optimize Delta table with Liquid Clustering"
* "Tune Spark job to reduce shuffle spill"
* "Reduce pipeline costs by 30% using serverless compute"

#### ML Operations
* "Build a feature engineering pipeline with Feature Store"
* "Track experiments with MLflow and Unity Catalog models"
* "Deploy model to serverless endpoint with scale-to-zero"

#### Orchestration
* "Create multi-task workflow with conditional execution"
* "Schedule job with cron trigger and retry logic"
* "Implement circuit breaker pattern for failures"

#### Governance
* "Set up Unity Catalog with row-level security"
* "Implement data contracts for schema validation"
* "Create audit logging with system.access.audit"

#### Operations
* "Set up monitoring dashboard with SLO tracking"
* "Create data quality tests with pytest"
* "Write runbook documentation for on-call"

---

## 📋 Skill Quality Standards

All 22 skills meet these production-ready criteria:

* ✅ **400+ lines** of comprehensive content
* ✅ **5-8 complete** working code examples
* ✅ **Clear dependency mapping** with other skills
* ✅ **Cross-skill references** and integration patterns
* ✅ **Best practices** and anti-patterns documented
* ✅ **Modern Databricks features** (Unity Catalog, Serverless, Photon, Liquid Clustering, Lakehouse Monitoring)
* ✅ **Last updated** within current year (2026)
* ✅ **Tested** with real Databricks scenarios

---

## 👨‍💻 Maintained By

**Chakradhar Dodda**  
*Senior Data Engineer | Azure Data Engineer | Databricks*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/chakradhar-dodda-8a46b2b0/)

8 years of experience in enterprise-scale data solutions on Azure Cloud and Databricks. This inventory represents production-tested patterns from real-world implementations across Telecom and Energy domains.

**Certifications:**  
🏅 Databricks Certified Data Engineer Associate  
🏅 Microsoft Certified: Azure Data Engineer Associate (DP-203)  
🏅 Microsoft Certified: Azure AI Engineer Associate (AI-102)

---

**Last Updated**: 2026-04-09  
**Repository**: databricks-genie-code-skills  
**Version**: 2.0  
**Total Skills**: 22 complete  
**Total Lines**: 14,841

---

*For installation instructions, see [README.md](README.md#quick-start)*  
*For skill creation guide, see [README.md](README.md#creating-custom-skills)*  
*For repository overview, see [README.md](README.md)*
