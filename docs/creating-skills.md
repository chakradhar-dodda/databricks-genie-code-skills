# Creating Custom Skills

This guide explains how to create your own custom skills for Genie Code.

## Skill Structure

Each skill is a directory containing a `SKILL.md` file:
```
my-custom-skill/
└── SKILL.md
```

## SKILL.md Template

```markdown
# Skill Name

## Purpose
Briefly describe what this skill teaches Genie Code to do.
Be specific about the capability being added.

## When to Use
List specific situations or use cases where this skill applies:
* Use case 1
* Use case 2
* Use case 3

## Key Concepts
Explain the core concepts and principles:

### Concept 1
Definition and importance

### Concept 2
How it works and when to apply

---

## Examples

### Example 1: Basic Usage
\`\`\`python
# Show a simple, working example
def example_function():
    # With clear comments
    pass
\`\`\`

Explain what the example does and why.

### Example 2: Advanced Pattern
\`\`\`python
# Show a more complex real-world example
class AdvancedExample:
    def method(self):
        # Production-ready code
        pass
\`\`\`

Explain the advanced pattern and its benefits.

### Example 3: Integration
Show how this integrates with Databricks features.

---

## Best Practices

### 1. Practice Name
* Guideline
* Why it matters
* When to apply

### 2. Practice Name
* Another best practice
* Rationale

---

## Common Pitfalls

### ❌ Anti-Pattern Name
* **Problem**: What goes wrong
* **Impact**: Why it's bad
* **Solution**: How to fix it

### ❌ Another Anti-Pattern
* Description and solution

---

## Related Skills

* [Related Skill 1](../path/to/skill/SKILL.md)
* [Related Skill 2](../path/to/skill/SKILL.md)

---

**Last Updated**: YYYY-MM-DD
```

## Creating Your First Skill

### Step 1: Choose a Topic
Pick something you want to teach Genie Code:
* A specific data pattern
* A workflow you use frequently
* A best practice for your team
* Integration with a tool or service

### Step 2: Create the Directory
```bash
mkdir -p ~/.assistant/skills/my-skill-name
```

### Step 3: Write SKILL.md
```bash
code ~/.assistant/skills/my-skill-name/SKILL.md
```

Use the template above as a starting point.

### Step 4: Include Working Examples
The most valuable part of a skill is **working code examples**.

**Good Example:**
```python
from pyspark.sql.functions import col, current_timestamp

def audit_table_changes(source_table: str, target_table: str):
    """
    Complete, working function with:
    - Clear purpose
    - Proper imports
    - Type hints
    - Docstring
    """
    df = spark.table(source_table)
    df = df.withColumn("audit_timestamp", current_timestamp())
    df.write.mode("append").saveAsTable(target_table)
    return df.count()
```

**Bad Example:**
```python
# Pseudocode that doesn't actually work
def some_function():
    # Do something
    pass
```

### Step 5: Add Context
Explain:
* **Purpose**: Why this pattern exists
* **Trade-offs**: When to use vs. not use
* **Integration**: How it fits with Databricks features

### Step 6: Test with Genie Code
Ask Genie Code questions related to your skill:
* "How do I [do the thing your skill covers]?"
* "Show me an example of [your skill topic]"

Genie Code should reference your skill in its response.

---

## Skill Best Practices

### 1. Be Specific
✅ "Delta Lake Z-ORDER Optimization"
❌ "Making tables faster"

### 2. Include Complete Examples
* Use actual imports and libraries
* Show real Databricks API calls
* Make examples copy-paste ready

### 3. Explain Why, Not Just How
* Why this pattern is valuable
* Trade-offs and alternatives
* When NOT to use it

### 4. Reference Databricks Features
* Link to Unity Catalog, Delta Lake, etc.
* Show integration patterns
* Mention version requirements

### 5. Keep It Updated
* Add "Last Updated" date
* Update when Databricks releases new features
* Remove deprecated patterns

---

## Advanced Skill Topics

### Multi-File Skills
For complex topics, you can include additional files:
```
my-advanced-skill/
├── SKILL.md          # Main skill definition
├── examples.py       # Extended code examples
└── reference.md      # Additional documentation
```

Reference them in SKILL.md:
```markdown
For more examples, see [examples.py](./examples.py)
```

### Domain-Specific Skills
Create skills for your industry:
* Healthcare: HIPAA compliance patterns
* Finance: PCI-DSS data handling
* Retail: Customer 360 architectures

### Team-Specific Skills
Document your team's patterns:
* Company naming conventions
* Standard data flows
* Approved architectures

---

## Contributing Skills

Want to share your skills?

1. **Test thoroughly**: Ensure examples work
2. **Document clearly**: Follow the template
3. **Submit for review**: Create a pull request
4. **Respond to feedback**: Iterate on suggestions

---

## Examples of Good Skills

Check these skills in the repository:
* `data-quality-checks` - Comprehensive validation patterns
* `delta-lake-optimization` - Performance tuning techniques
* `incremental-processing` - Efficient data processing

---

*Happy skill creating!*
