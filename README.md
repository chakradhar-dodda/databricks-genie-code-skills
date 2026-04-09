# Databricks Genie Code Skills Repository

A comprehensive collection of **22 production-ready skills** for **Genie Code**, designed to empower data professionals working on the Databricks platform. This repository teaches Genie Code domain-specific expertise in data engineering, analytics, ML operations, platform optimization, and governance.

> **📊 Repository Stats:** 22 skills | 14,841 lines of code | 8 categories | Production-tested patterns

---

## 📚 Overview

This repository enhances Genie Code's capabilities through four powerful configuration layers:

### 1. **User Instructions** (`~/.assistant_instructions.md`)
Personal, persistent instructions for Genie Code - your coding style, preferences, and project context.

### 2. **Workspace Instructions** (Admin-configured)
Organization-wide guidelines configured by workspace admins for consistency across teams.

### 3. **User Skills** (`~/.assistant/skills/`)
Custom skills you create to teach Genie Code new capabilities specific to your workflows.

### 4. **Workspace Skills** (Admin-configured)
Enterprise-level domain expertise available to all users in the workspace.

---

## 🗂️ Repository Structure

```
databricks-genie-code-skills/
├── README.md                                   # This file
├── SKILLS_INVENTORY.md                         # Detailed skill catalog
├── examples/
│   ├── user-instructions.md                    # Template for personal config
│   └── workspace-instructions.md               # Example workspace config
├── skills/                                     # 22 production skills
│   ├── auto-loader/                           # 593 lines
│   ├── cost-optimization/                     # 825 lines  
│   ├── data-contracts/                        # 824 lines
│   ├── data-modeling/                         # 489 lines
│   ├── data-quality-checks/                   # 552 lines
│   ├── delta-lake-optimization/               # 681 lines
│   ├── documentation-practices/               # 893 lines
│   ├── error-handling/                        # 677 lines
│   ├── feature-engineering/                   # 822 lines
│   ├── incremental-processing/                # 593 lines
│   ├── job-scheduling/                        # 631 lines
│   ├── medallion-architecture/                # 497 lines
│   ├── mlflow-tracking/                       # 702 lines
│   ├── model-deployment/                      # 758 lines
│   ├── monitoring-observability/              # 427 lines
│   ├── performance-tuning/                    # 819 lines
│   ├── spark-optimization/                    # 960 lines
│   ├── sql-best-practices/                    # 826 lines
│   ├── streaming-pipelines/                   # 644 lines
│   ├── testing-strategies/                    # 530 lines
│   ├── unity-catalog-governance/              # 348 lines
│   └── workflow-orchestration/                # 750 lines
└── docs/                                       # Additional documentation
    ├── installation.md
    ├── creating-skills.md
    └── best-practices.md
```

---

## 🚀 Quick Start

### Installation

**Option 1: Direct Copy (Recommended)**
```bash
# 1. Navigate to your Databricks workspace
# 2. Clone or download this repository to your workspace folder
# 3. Copy skills to Genie Code's skills folder

cp -r /Workspace/Users/your.email@company.com/databricks-genie-code-skills/skills/* ~/.assistant/skills/
```

**Option 2: Git Clone**
```bash
# Clone into your workspace
git clone <repository-url> /Workspace/Users/your.email@company.com/databricks-genie-code-skills

# Link skills to Genie Code
ln -s /Workspace/Users/your.email@company.com/databricks-genie-code-skills/skills/* ~/.assistant/skills/
```

**Option 3: Selective Skills**
```bash
# Copy only specific skills you need
cp -r /Workspace/Users/your.email@company.com/databricks-genie-code-skills/skills/delta-lake-optimization ~/.assistant/skills/
cp -r /Workspace/Users/your.email@company.com/databricks-genie-code-skills/skills/spark-optimization ~/.assistant/skills/
```

### Verify Installation
```bash
# List installed skills
ls -la ~/.assistant/skills/

# Each skill should have a SKILL.md file
cat ~/.assistant/skills/delta-lake-optimization/SKILL.md
```

### (Optional) Customize User Instructions
```bash
# Copy template and customize for your needs
cp /Workspace/Users/your.email@company.com/databricks-genie-code-skills/examples/user-instructions.md ~/.assistant_instructions.md

# Edit with your preferences
databricks workspace edit ~/.assistant_instructions.md
```

---

## 💡 Using Skills with Genie Code

Once installed, skills are **automatically available** to Genie Code. Simply interact naturally:

### Natural Language Queries
```
"Help me optimize this Delta table for better query performance"
→ Genie Code uses: delta-lake-optimization, spark-optimization

"Set up a medallion architecture pipeline with data quality checks"
→ Genie Code uses: medallion-architecture, data-quality-checks, incremental-processing

"Deploy this ML model with monitoring and versioning"
→ Genie Code uses: model-deployment, mlflow-tracking, monitoring-observability
```

### Skill Selection
Genie Code automatically selects relevant skills based on:
* **Keywords in your question** (Delta, Spark, MLflow, Unity Catalog, etc.)
* **Context from your notebook** (existing code, table references, imports)
* **Task type** (optimization, deployment, governance, orchestration)
* **Best practices** (automatically applies patterns from multiple skills)

### Advanced Usage
```python
# Reference specific skills explicitly
"Using the spark-optimization skill, improve this join operation"

# Combine multiple skills
"Apply sql-best-practices and performance-tuning to optimize this query"

# Request comprehensive solutions
"Build a complete streaming pipeline following best practices"
→ Uses: streaming-pipelines, auto-loader, data-quality-checks, 
        error-handling, monitoring-observability
```

---

## 📖 Key Skills Included (22 Skills, 14,841 Lines)

### 🔧 Data Engineering (5 skills | 3,063 lines)
**High-volume data processing, ingestion, and pipeline patterns**

| Skill | Lines | Description |
|-------|-------|-------------|
| **Delta Lake Optimization** | 681 | OPTIMIZE, VACUUM, Z-ORDER, Liquid Clustering, Predictive I/O |
| **Auto Loader** | 593 | Schema inference, Unity Catalog Volumes, serverless ingestion |
| **Streaming Pipelines** | 644 | Structured Streaming, Delta Live Tables, exactly-once semantics |
| **Incremental Processing** | 593 | Change Data Feed, merge patterns, watermarking |
| **Data Quality Checks** | 552 | Expectations, Lakehouse Monitoring, Great Expectations integration |

### 🏛️ Governance & Architecture (3 skills | 1,669 lines)
**Security, compliance, and architectural patterns**

| Skill | Lines | Description |
|-------|-------|-------------|
| **Unity Catalog Governance** | 348 | Access control, lineage, data discovery, audit logging |
| **Data Contracts** | 824 | Schema enforcement, validation, evolution patterns |
| **Medallion Architecture** | 497 | Bronze/Silver/Gold layers, multi-hop pipelines |

### ⚡ Performance & Optimization (3 skills | 2,604 lines)
**Cost reduction and query acceleration techniques**

| Skill | Lines | Description |
|-------|-------|-------------|
| **Spark Optimization** | 960 | Partitioning, caching, broadcast joins, Photon engine |
| **Cost Optimization** | 825 | Cluster sizing, spot instances, serverless economics |
| **Performance Tuning** | 819 | Query plans, bottleneck analysis, memory tuning |

### 📊 Analytics & Data Modeling (2 skills | 1,315 lines)
**SQL patterns and dimensional modeling**

| Skill | Lines | Description |
|-------|-------|-------------|
| **SQL Best Practices** | 826 | Query optimization, CTEs, window functions, anti-patterns |
| **Data Modeling** | 489 | Star schema, SCD Type 2, normalization techniques |

### 🤖 ML Operations (3 skills | 2,282 lines)
**End-to-end machine learning lifecycle**

| Skill | Lines | Description |
|-------|-------|-------------|
| **Feature Engineering** | 822 | Feature Store, real-time serving, transformations |
| **Model Deployment** | 758 | Batch inference, REST APIs, serverless endpoints |
| **MLflow Tracking** | 702 | Experiment tracking, model registry, versioning |

### 🔄 Orchestration & Workflow (3 skills | 2,058 lines)
**Job scheduling and dependency management**

| Skill | Lines | Description |
|-------|-------|-------------|
| **Workflow Orchestration** | 750 | Multi-task jobs, dependencies, conditional execution |
| **Job Scheduling** | 631 | Cron triggers, event-driven patterns, SLA management |
| **Error Handling** | 677 | Retry strategies, alerting, failure recovery |

### 🔍 Quality & Monitoring (2 skills | 957 lines)
**Observability and testing patterns**

| Skill | Lines | Description |
|-------|-------|-------------|
| **Monitoring & Observability** | 427 | Metrics collection, alerting, system.access.audit |
| **Testing Strategies** | 530 | Unit tests, integration tests, data validation |

### 📚 Best Practices (1 skill | 893 lines)
**Standards and documentation patterns**

| Skill | Lines | Description |
|-------|-------|-------------|
| **Documentation Practices** | 893 | Code documentation, notebook standards, knowledge sharing |

---

## ✍️ Creating Custom Skills

Extend this repository with your own domain-specific skills:

### Skill Template Structure
```markdown
# Skill Name

## Purpose
Brief description of what this skill teaches Genie Code

## When to Use
Situations where this skill applies

## Key Concepts
1. Core principle 1
2. Core principle 2
3. Core principle 3

## Examples

### Example 1: Basic Usage
```python
# Code example with detailed comments
# Explain the pattern and why it works
```

### Example 2: Advanced Pattern
```python
# More complex real-world example
# Include error handling and edge cases
```

## Best Practices
* Practice 1 with rationale
* Practice 2 with rationale
* Practice 3 with rationale

## Common Pitfalls
* ❌ What to avoid
* ✅ What to do instead
* 💡 Why it matters

## Related Skills
* [skill-name](../skill-name/SKILL.md)
```

### Adding New Skills

1. **Create skill folder:**
   ```bash
   mkdir -p skills/your-skill-name
   cd skills/your-skill-name
   ```

2. **Create SKILL.md file** using the template above

3. **Add comprehensive examples** (aim for 500+ lines)

4. **Test with Genie Code:**
   ```bash
   cp -r skills/your-skill-name ~/.assistant/skills/
   ```

5. **Update this README** in the Key Skills section

---

## 🤝 Contributing

We **welcome and encourage contributions!** This repository thrives on community knowledge and real-world patterns.

### How to Contribute

**✨ Share New Skills**
* Industry-specific patterns (healthcare, finance, retail, etc.)
* Cloud-specific optimizations (AWS, Azure, GCP)
* Advanced MLOps workflows
* Real-time analytics patterns
* Security and compliance patterns

**🐛 Improve Existing Skills**
* Add more examples and use cases
* Update with latest Databricks features
* Fix errors or clarify explanations
* Add edge cases and troubleshooting tips

**📚 Enhance Documentation**
* Better installation guides
* Video tutorials or notebooks
* Architecture diagrams
* Performance benchmarks

### Contribution Process

1. **Fork this repository**
2. **Create a feature branch** (`git checkout -b feature/your-skill-name`)
3. **Add your skill** following the template structure
4. **Test with Genie Code** to ensure it works
5. **Submit a Pull Request** with clear description
6. **Share your use case** - help others understand when to use it

### What Makes a Great Contribution?

✅ **Production-tested patterns** (real-world usage)  
✅ **Comprehensive examples** (500+ lines recommended)  
✅ **Clear explanations** (why, not just how)  
✅ **Best practices & pitfalls** (learn from experience)  
✅ **Related skills** (help Genie Code connect concepts)

---

## ⭐ Show Your Support

If this repository helps you or your team:

* **⭐ Star this repository** to help others discover it
* **🔗 Share with colleagues** working on Databricks
* **💬 Provide feedback** on what works and what doesn't
* **🤝 Contribute your own skills** from production use cases

**Your stars and contributions help grow this knowledge base for the entire Databricks community!**

---

## 💬 Support & Feedback

This is a **living repository** designed to evolve with the Databricks platform, Genie Code capabilities, and community needs.

### Get Help
* **Questions?** Open an issue with the `question` label
* **Bug found?** Report with detailed reproduction steps
* **Feature request?** Share your use case and requirements

### Share Feedback
* **What's working well?** Tell us which skills save you the most time
* **What's missing?** Suggest new skill categories or topics
* **What needs improvement?** Help us enhance existing content
* **Production examples?** Share your real-world usage patterns

### Join the Community
* Participate in discussions
* Review pull requests
* Share your learning journey
* Help other users

**We read every issue and pull request.** Your feedback directly shapes this repository's evolution.

---

## 🌐 Connect & Collaborate

### Open to Discussion

I'm excited to discuss on:

* **📈 Scaling patterns** - How you're using these skills in production
* **🎓 Training programs** - Onboarding teams to Databricks with these skills
* **🏢 Enterprise adoption** - Deploying skills across large organizations
* **🤝 Partnerships** - Collaborating on industry-specific skill libraries
* **📝 Content creation** - Writing blogs, tutorials, or courses together
* **🎤 Speaking opportunities** - Presenting at conferences or meetups

**Have an idea or opportunity?** Open a discussion or reach out through issues or connect with me in Linkedin!

---

## 📚 Additional Resources

### Databricks Documentation
* [Databricks Documentation](https://docs.databricks.com)
* [Delta Lake Guide](https://docs.delta.io)
* [Unity Catalog Documentation](https://docs.databricks.com/data-governance/unity-catalog)
* [MLflow Documentation](https://mlflow.org/docs/latest/index.html)
* [Databricks SQL](https://docs.databricks.com/sql)

### Community Resources
* [Databricks Community Forums](https://community.databricks.com)
* [Delta Lake Community](https://delta.io/community)
* [Stack Overflow - Databricks Tag](https://stackoverflow.com/questions/tagged/databricks)

### Learning Platforms
* [Databricks Academy](https://academy.databricks.com)
* [Partner Training](https://partner-academy.databricks.com)

---

## 🎓 Learning Paths

Whether you're just starting with Databricks or optimizing production workloads, follow these curated learning paths:

### 🌱 Beginner Path (New to Databricks)
**Goal:** Build foundation in data engineering and analytics

1. **Start:** [SQL Best Practices](skills/sql-best-practices/SKILL.md) (826 lines)
   * Learn Databricks SQL syntax and optimization patterns
   * Understand query execution and performance

2. **Next:** [Delta Lake Optimization](skills/delta-lake-optimization/SKILL.md) (681 lines)
   * Master the lakehouse storage layer
   * Learn OPTIMIZE, VACUUM, and Z-ORDER

3. **Then:** [Medallion Architecture](skills/medallion-architecture/SKILL.md) (497 lines)
   * Understand Bronze/Silver/Gold patterns
   * Build multi-hop data pipelines

4. **Practice:** [Data Quality Checks](skills/data-quality-checks/SKILL.md) (552 lines)
   * Implement validation in your pipelines
   * Use Lakehouse Monitoring

**Time Investment:** 2-3 weeks | **Lines to Study:** 2,556 lines

---

### 🚀 Intermediate Path (Building Production Pipelines)
**Goal:** Create reliable, scalable data pipelines

1. **Foundation:** [Incremental Processing](skills/incremental-processing/SKILL.md) (593 lines)
   * Efficient batch and streaming patterns
   * Change Data Feed and merge operations

2. **Ingestion:** [Auto Loader](skills/auto-loader/SKILL.md) (593 lines)
   * Schema inference and evolution
   * Unity Catalog Volumes integration

3. **Real-time:** [Streaming Pipelines](skills/streaming-pipelines/SKILL.md) (644 lines)
   * Structured Streaming and DLT
   * Exactly-once processing guarantees

4. **Governance:** [Unity Catalog Governance](skills/unity-catalog-governance/SKILL.md) (348 lines)
   * Access control and lineage
   * Data discovery and audit logging

5. **Quality:** [Data Contracts](skills/data-contracts/SKILL.md) (824 lines)
   * Schema enforcement and validation
   * Contract-driven development

6. **Operations:** [Workflow Orchestration](skills/workflow-orchestration/SKILL.md) (750 lines)
   * Multi-task job dependencies
   * Error handling and monitoring

**Time Investment:** 4-6 weeks | **Lines to Study:** 3,752 lines

---

### ⚡ Advanced Path (Performance & Scale Optimization)
**Goal:** Optimize for cost, performance, and scale

1. **Performance:** [Spark Optimization](skills/spark-optimization/SKILL.md) (960 lines)
   * Partitioning strategies and caching
   * Broadcast joins and shuffle optimization
   * Photon engine best practices

2. **Tuning:** [Performance Tuning](skills/performance-tuning/SKILL.md) (819 lines)
   * Query plan analysis
   * Memory and resource tuning
   * Bottleneck identification

3. **Cost:** [Cost Optimization](skills/cost-optimization/SKILL.md) (825 lines)
   * Cluster sizing and autoscaling
   * Serverless vs. classic compute
   * Spot instance strategies

4. **Reliability:** [Error Handling](skills/error-handling/SKILL.md) (677 lines)
   * Retry strategies and circuit breakers
   * Failure recovery patterns
   * Alert configuration

5. **Monitoring:** [Monitoring & Observability](skills/monitoring-observability/SKILL.md) (427 lines)
   * Metrics collection and dashboards
   * System audit logging
   * SLO tracking

**Time Investment:** 3-4 weeks | **Lines to Study:** 3,708 lines

---

### 🤖 ML Engineering Path (End-to-End ML Operations)
**Goal:** Deploy and manage production ML systems

1. **Features:** [Feature Engineering](skills/feature-engineering/SKILL.md) (822 lines)
   * Feature Store implementation
   * Real-time feature serving
   * Feature transformation pipelines

2. **Tracking:** [MLflow Tracking](skills/mlflow-tracking/SKILL.md) (702 lines)
   * Experiment tracking and comparison
   * Model registry and versioning
   * Artifact management

3. **Deployment:** [Model Deployment](skills/model-deployment/SKILL.md) (758 lines)
   * Batch vs. real-time inference
   * REST API endpoints
   * Serverless model serving

4. **Scheduling:** [Job Scheduling](skills/job-scheduling/SKILL.md) (631 lines)
   * Training job orchestration
   * Retraining triggers
   * Model monitoring jobs

5. **Quality:** [Testing Strategies](skills/testing-strategies/SKILL.md) (530 lines)
   * Model validation tests
   * Integration testing
   * A/B testing patterns

**Time Investment:** 3-4 weeks | **Lines to Study:** 3,443 lines

---

### 🏢 Enterprise Path (Team Standards & Best Practices)
**Goal:** Establish organizational standards and practices

1. **Documentation:** [Documentation Practices](skills/documentation-practices/SKILL.md) (893 lines)
   * Code documentation standards
   * Notebook organization
   * Knowledge management

2. **Modeling:** [Data Modeling](skills/data-modeling/SKILL.md) (489 lines)
   * Dimensional modeling (star/snowflake)
   * SCD Type 2 patterns
   * Normalization techniques

3. **Testing:** [Testing Strategies](skills/testing-strategies/SKILL.md) (530 lines)
   * Unit and integration testing
   * Data validation frameworks
   * CI/CD integration

4. **Monitoring:** [Monitoring & Observability](skills/monitoring-observability/SKILL.md) (427 lines)
   * Team dashboards and alerts
   * Audit logging and compliance
   * Performance tracking

**Time Investment:** 2-3 weeks | **Lines to Study:** 2,339 lines

---

### 📊 Complete Mastery Path (All Skills)
**Goal:** Comprehensive expertise across all domains

Complete all 22 skills in recommended order by category:
1. Data Engineering (5 skills, 3,063 lines)
2. Governance & Architecture (3 skills, 1,669 lines)
3. Analytics & Data Modeling (2 skills, 1,315 lines)
4. Performance & Optimization (3 skills, 2,604 lines)
5. Orchestration & Workflow (3 skills, 2,058 lines)
6. ML Operations (3 skills, 2,282 lines)
7. Quality & Monitoring (2 skills, 957 lines)
8. Best Practices (1 skill, 893 lines)

**Time Investment:** 12-16 weeks | **Total Content:** 14,841 lines

---

## 📄 License

This repository is provided as-is for **educational and productivity purposes**. 

* ✅ Free to use, modify, and distribute
* ✅ Customize freely for your organization's needs
* ✅ Share with your team and community
* ✅ Build upon and create derivatives

**No warranty provided.** Use at your own discretion and always test in development before production.

---

## 🙏 Acknowledgments

### Created By

**Chakradhar Dodda**  
*Senior Data Engineer | Azure Data Engineer | Databricks*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/chakradhar-dodda-8a46b2b0/)

A Senior Data Engineer with 8 years of experience designing enterprise-scale data solutions on Azure Cloud and Databricks. Specializing in ETL/ELT pipelines, Delta Lake architecture, and Medallion data lakehouse patterns. This repository represents production-tested patterns and best practices from real-world implementations across Telecom and Energy domains.

**Certifications:**  
🏅 Databricks Certified Data Engineer Associate  
🏅 Microsoft Certified: Azure Data Engineer Associate (DP-203)  
🏅 Microsoft Certified: Azure AI Engineer Associate (AI-102)

**Core Expertise:**  
Azure Databricks • Delta Lake • PySpark • Azure Data Factory • Azure Synapse Analytics • Data Lakehouse Architecture • Medallion Architecture • Unity Catalog • ETL/ELT Pipelines

---

### Special Thanks

* The **Databricks community** for sharing knowledge and best practices
* **Contributors** who enhance these skills with real-world experience
* **Genie Code team** for building an incredible AI assistant
* **Users** who provide feedback and star this repository

---

**Built with ❤️ for the Databricks community**

*Last Updated: 2026-04-09*  
*Repository Version: 1.0*  
*Total Skills: 22 | Total Lines: 14,841*  
*Created by: [Chakradhar Dodda](https://www.linkedin.com/in/chakradhar-dodda-8a46b2b0/)*
