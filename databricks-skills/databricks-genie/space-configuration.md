# Genie Space Advanced Configuration

Configure Genie Space instructions, joins, filters, dimensions, measures, and example queries programmatically using the **Databricks Python SDK**.

This guide covers the advanced configuration workflow that goes beyond basic space creation. For basic operations (create, list, query), see [mcp-tools-reference.md](mcp-tools-reference.md).

---

## Overview

**Basic space creation** (via MCP tools) gives you a working Genie Space, but **advanced configuration** dramatically improves query quality by teaching Genie:

- **How tables relate** (join specifications)
- **What filters to use** (active employees, recent orders, etc.)
- **Common dimensions** (year, region, tenure buckets)
- **Key metrics** (headcount, revenue, conversion rate)
- **Example patterns** (question-to-SQL examples)

This configuration uses the `serialized_space` parameter, which contains a JSON structure defining all these elements.

---

## Prerequisites

```bash
pip install databricks-sdk
```

Ensure you have authentication configured:
- `.databrickscfg` file with profile, OR
- Environment variables: `DATABRICKS_HOST`, `DATABRICKS_TOKEN`

---

## Quick Start

```python
from databricks.sdk import WorkspaceClient
import json
import uuid

# Initialize SDK client (auto-authenticates)
w = WorkspaceClient()

# Get your space_id (from create_or_update_genie or list_genie MCP tool)
space_id = "your-space-id"

# Build configuration
config = {
    "version": 2,
    "config": {"sample_questions": []},
    "data_sources": {
        "tables": [
            {"identifier": "catalog.schema.table1"},
            {"identifier": "catalog.schema.table2"}
        ]
    },
    "instructions": {
        "text_instructions": [],
        "join_specs": [],
        "example_question_sqls": [],
        "sql_snippets": {
            "filters": [],
            "expressions": [],
            "measures": []
        }
    }
}

# Sort tables alphabetically (REQUIRED)
config["data_sources"]["tables"].sort(key=lambda x: x["identifier"])

# Update space with configuration
result = w.genie.update_space(
    space_id=space_id,
    serialized_space=json.dumps(config, indent=2)
)

print(f"✅ Configured space: {result.space_id}")
```

---

## Configuration Structure

The `serialized_space` parameter expects a JSON string with this structure:

```json
{
  "version": 2,
  "config": {
    "sample_questions": [...]
  },
  "data_sources": {
    "tables": [...]
  },
  "instructions": {
    "text_instructions": [...],
    "join_specs": [...],
    "example_question_sqls": [...],
    "sql_snippets": {
      "filters": [...],
      "expressions": [...],
      "measures": [...]
    }
  }
}
```

---

## Critical Requirements

Before implementing, understand these **non-negotiable requirements**:

1. ✅ **Every item needs an `id`** - Lowercase 32-hex UUID without hyphens. Use `uuid.uuid4().hex`
2. ✅ **All lists must be sorted by `id`** - Or you get `INVALID_PARAMETER_VALUE: instructions must be sorted by id`
3. ✅ **`data_sources.tables` must be sorted by `identifier`** - Alphabetical sort required
4. ✅ **String arrays for multi-line content** - SQL, instructions, and content fields use arrays of strings
5. ✅ **Join SQL uses backtick-quoted aliases** - e.g., `` `orders`.`customer_id` = `customers`.`id` ``
6. ✅ **Relationship type on separate line** - The `--rt=FROM_RELATIONSHIP_TYPE_*--` must be a separate array element
7. ✅ **SDK overwrites entirely** - If you update with partial config, existing joins/filters are wiped. Always send full config.
8. ✅ **Always save a backup** - Export the config to a file before updating

---

## Helper Functions

```python
import uuid

def hex_id():
    """Generate a 32-character hex UUID for config items."""
    return uuid.uuid4().hex

def sort_config(config):
    """Sort all lists in config by required fields.

    CRITICAL: This must be called before sending to SDK.
    """
    # Sort sample questions by id
    config["config"]["sample_questions"].sort(key=lambda x: x["id"])

    # Sort tables by identifier
    config["data_sources"]["tables"].sort(key=lambda x: x["identifier"])

    # Sort instruction sections by id
    for key in ["text_instructions", "example_question_sqls", "join_specs"]:
        config["instructions"][key].sort(key=lambda x: x["id"])

    # Sort sql_snippets by id
    for key in ["filters", "expressions", "measures"]:
        config["instructions"]["sql_snippets"][key].sort(key=lambda x: x["id"])

    return config
```

---

## Configuration Sections

### 1. Text Instructions

Markdown-formatted guidance that teaches Genie about your data model. This is the **most impactful section** for query quality.

```python
{
    "id": hex_id(),
    "content": [
        "## Data Model Overview\n",
        "\n",
        "- **dim_employee**: Master employee records with eid as primary key\n",
        "- **fact_payroll**: Payroll transactions joined via eid\n",
        "\n",
        "### Business Rules\n",
        "\n",
        "- For headcount queries, filter to status = 'Active'\n",
        "- The eid field is the join key for all fact tables\n",
        "- Hire dates before 2020 may have incomplete data\n"
    ]
}
```

**Tips**:
- Use markdown formatting for clarity
- Each array element is one line (include `\n` at the end)
- Cover: table descriptions, relationships, business rules, valid values
- Explain disambiguation (e.g., "revenue" could mean gross vs net)

---

### 2. Join Specifications

Define how tables relate to each other. Genie uses these to auto-generate JOINs.

```python
{
    "id": hex_id(),
    "left": {
        "identifier": "catalog.schema.fact_payroll",
        "alias": "fact_payroll"
    },
    "right": {
        "identifier": "catalog.schema.dim_employee",
        "alias": "dim_employee"
    },
    "sql": [
        "`fact_payroll`.`eid` = `dim_employee`.`eid`",
        "--rt=FROM_RELATIONSHIP_TYPE_MANY_TO_ONE--"
    ],
    "instruction": [
        "Join payroll to employee dimension to get employee details"
    ]
}
```

**Relationship types** (second element in `sql` array):
- `FROM_RELATIONSHIP_TYPE_ONE_TO_ONE` - e.g., employee to profile
- `FROM_RELATIONSHIP_TYPE_MANY_TO_ONE` - e.g., orders to customer
- `FROM_RELATIONSHIP_TYPE_MANY_TO_MANY` - e.g., students to courses

**Requirements**:
- Always use backtick-quoted format: `` `alias`.`column` ``
- Alias should match table name (last segment of identifier)
- Relationship type must be on separate line

---

### 3. Filters (sql_snippets.filters)

Predefined WHERE clause conditions with natural language synonyms.

```python
{
    "id": hex_id(),
    "sql": ["dim_employee.status = 'Active'"],
    "display_name": "Active Employees",
    "instruction": ["Use when user asks about current, active, or existing employees"],
    "synonyms": ["current employees", "active staff", "employed", "working"]
}
```

**Tips**:
- Cover the most common filter patterns users ask for
- Include 3-5 synonyms covering natural phrasing
- Use `alias.column` format in SQL
- Keep filters focused (one concept per filter)

---

### 4. Expressions / Dimensions (sql_snippets.expressions)

Computed columns for GROUP BY - date extractions, bucketing, categorization.

```python
# Date extraction
{
    "id": hex_id(),
    "sql": ["YEAR(dim_employee.hire_date)"],
    "display_name": "Hire Year",
    "instruction": ["Use to group employees by hire year"],
    "synonyms": ["hire year", "year hired", "hired in year"]
}

# Bucketing
{
    "id": hex_id(),
    "sql": [
        "CASE WHEN DATEDIFF(CURRENT_DATE(), dim_employee.hire_date) < 365 THEN '<1 Year' "
        "WHEN DATEDIFF(CURRENT_DATE(), dim_employee.hire_date) < 1095 THEN '1-3 Years' "
        "ELSE '3+ Years' END"
    ],
    "display_name": "Tenure Bucket",
    "instruction": ["Use to categorize employees by tenure"],
    "synonyms": ["tenure group", "tenure band", "seniority level"]
}

# Direct column reference
{
    "id": hex_id(),
    "sql": ["dim_employee.department"],
    "display_name": "Department",
    "instruction": ["Group by department"],
    "synonyms": ["dept", "department", "team"]
}
```

**Common patterns**:
- `YEAR(date_column)` - Group by year
- `DATE_TRUNC('month', date_column)` - Group by month
- `CASE WHEN ... END` - Bucketing
- Direct column references as named dimensions

---

### 5. Measures (sql_snippets.measures)

Aggregate calculations - SUM, COUNT, AVG applied to metrics.

```python
{
    "id": hex_id(),
    "sql": ["COUNT(DISTINCT dim_employee.eid)"],
    "display_name": "Employee Count",
    "instruction": ["Use to count distinct employees"],
    "synonyms": ["headcount", "number of employees", "staff count", "FTE"]
}

{
    "id": hex_id(),
    "sql": ["AVG(fact_payroll.salary)"],
    "display_name": "Average Salary",
    "instruction": ["Calculate average salary across employees"],
    "synonyms": ["avg salary", "mean salary", "average pay"]
}
```

**Tips**:
- Use `DISTINCT` where appropriate to avoid double-counting across joins
- Include 3-5 synonyms for each measure
- Always qualify columns with table alias

---

### 6. Example Question-SQL Pairs (example_question_sqls)

Complete question-to-SQL examples that teach Genie query patterns.

```python
{
    "id": hex_id(),
    "question": ["What is the turnover rate by department?"],
    "sql": [
        "SELECT department, period,\n",
        "  termination_count, avg_headcount,\n",
        "  ROUND(turnover_rate * 100, 1) AS turnover_pct\n",
        "FROM catalog.schema.metrics_turnover_by_dept\n",
        "ORDER BY turnover_rate DESC"
    ],
    "usage_guidance": ["Use for turnover rate analysis by department"]
}
```

**Tips**:
- Each SQL line is a separate array element ending with `\n`
- Use fully-qualified table names
- Include mix of simple (single-table) and complex (multi-join) examples
- Cover the most common question patterns

---

### 7. Sample Questions (config.sample_questions)

Questions that appear in the UI to guide users.

```python
{
    "id": hex_id(),
    "question": ["What is the total headcount by department?"]
}
```

**Tips**:
- Keep questions concise and clear
- Reference actual column names and business terms
- Cover different complexity levels

---

## Complete Implementation Example

```python
from databricks.sdk import WorkspaceClient
import json
import uuid

def hex_id():
    return uuid.uuid4().hex

def configure_genie_space(space_id: str, warehouse_id: str):
    """
    Configure a Genie Space with advanced instructions, joins, filters,
    dimensions, and measures.
    """
    w = WorkspaceClient()

    CAT = "my_catalog.my_schema"

    # Build full configuration
    config = {
        "version": 2,
        "config": {
            "sample_questions": [
                {"id": hex_id(), "question": ["What is the total headcount?"]},
                {"id": hex_id(), "question": ["Show me average salary by department"]},
                {"id": hex_id(), "question": ["What is the turnover rate?"]}
            ]
        },
        "data_sources": {
            "tables": [
                {"identifier": f"{CAT}.dim_employee"},
                {"identifier": f"{CAT}.fact_payroll"}
            ]
        },
        "instructions": {
            "text_instructions": [
                {
                    "id": hex_id(),
                    "content": [
                        "## HR Analytics Data Model\n",
                        "\n",
                        "This workspace contains employee and payroll data.\n",
                        "\n",
                        "### Tables\n",
                        "\n",
                        "- **dim_employee**: Master employee records\n",
                        "  - eid: Employee ID (primary key)\n",
                        "  - name, department, status, hire_date\n",
                        "  - status values: 'Active', 'Terminated', 'On Leave'\n",
                        "\n",
                        "- **fact_payroll**: Payroll transactions\n",
                        "  - eid: Foreign key to dim_employee\n",
                        "  - salary, bonus, period\n",
                        "\n",
                        "### Business Rules\n",
                        "\n",
                        "- For headcount, always filter to status = 'Active'\n",
                        "- Use DISTINCT counts to avoid duplication across joins\n",
                        "- Tenure is calculated from hire_date to current date\n"
                    ]
                }
            ],
            "join_specs": [
                {
                    "id": hex_id(),
                    "left": {
                        "identifier": f"{CAT}.fact_payroll",
                        "alias": "fact_payroll"
                    },
                    "right": {
                        "identifier": f"{CAT}.dim_employee",
                        "alias": "dim_employee"
                    },
                    "sql": [
                        "`fact_payroll`.`eid` = `dim_employee`.`eid`",
                        "--rt=FROM_RELATIONSHIP_TYPE_MANY_TO_ONE--"
                    ],
                    "instruction": [
                        "Join payroll facts to employee dimension for employee details"
                    ]
                }
            ],
            "example_question_sqls": [
                {
                    "id": hex_id(),
                    "question": ["What is the average salary by department?"],
                    "sql": [
                        "SELECT e.department,\n",
                        "  AVG(p.salary) as avg_salary\n",
                        "FROM my_catalog.my_schema.fact_payroll p\n",
                        "JOIN my_catalog.my_schema.dim_employee e ON p.eid = e.eid\n",
                        "WHERE e.status = 'Active'\n",
                        "GROUP BY e.department\n",
                        "ORDER BY avg_salary DESC"
                    ],
                    "usage_guidance": ["Use for salary analysis by department"]
                }
            ],
            "sql_snippets": {
                "filters": [
                    {
                        "id": hex_id(),
                        "sql": ["dim_employee.status = 'Active'"],
                        "display_name": "Active Employees",
                        "instruction": ["Filter to currently active employees"],
                        "synonyms": ["current employees", "active", "employed", "working"]
                    },
                    {
                        "id": hex_id(),
                        "sql": ["YEAR(dim_employee.hire_date) >= 2020"],
                        "display_name": "Hired Since 2020",
                        "instruction": ["Filter to employees hired in 2020 or later"],
                        "synonyms": ["recent hires", "new employees", "hired recently"]
                    }
                ],
                "expressions": [
                    {
                        "id": hex_id(),
                        "sql": ["YEAR(dim_employee.hire_date)"],
                        "display_name": "Hire Year",
                        "instruction": ["Group by year employee was hired"],
                        "synonyms": ["hire year", "year hired", "hired in"]
                    },
                    {
                        "id": hex_id(),
                        "sql": ["dim_employee.department"],
                        "display_name": "Department",
                        "instruction": ["Group by department"],
                        "synonyms": ["dept", "department", "team", "division"]
                    },
                    {
                        "id": hex_id(),
                        "sql": [
                            "CASE "
                            "WHEN DATEDIFF(CURRENT_DATE(), dim_employee.hire_date) < 365 THEN '<1 Year' "
                            "WHEN DATEDIFF(CURRENT_DATE(), dim_employee.hire_date) < 1095 THEN '1-3 Years' "
                            "ELSE '3+ Years' END"
                        ],
                        "display_name": "Tenure Bucket",
                        "instruction": ["Categorize employees by tenure"],
                        "synonyms": ["tenure", "tenure group", "seniority", "experience"]
                    }
                ],
                "measures": [
                    {
                        "id": hex_id(),
                        "sql": ["COUNT(DISTINCT dim_employee.eid)"],
                        "display_name": "Employee Count",
                        "instruction": ["Count distinct employees"],
                        "synonyms": ["headcount", "number of employees", "employee count", "FTE"]
                    },
                    {
                        "id": hex_id(),
                        "sql": ["AVG(fact_payroll.salary)"],
                        "display_name": "Average Salary",
                        "instruction": ["Calculate average salary"],
                        "synonyms": ["avg salary", "mean salary", "average pay"]
                    },
                    {
                        "id": hex_id(),
                        "sql": ["SUM(fact_payroll.salary)"],
                        "display_name": "Total Salary Cost",
                        "instruction": ["Sum of all salaries"],
                        "synonyms": ["total salary", "salary cost", "payroll cost"]
                    }
                ]
            }
        }
    }

    # CRITICAL: Sort all lists
    config["config"]["sample_questions"].sort(key=lambda x: x["id"])
    config["data_sources"]["tables"].sort(key=lambda x: x["identifier"])

    for key in ["text_instructions", "example_question_sqls", "join_specs"]:
        config["instructions"][key].sort(key=lambda x: x["id"])

    for key in ["filters", "expressions", "measures"]:
        config["instructions"]["sql_snippets"][key].sort(key=lambda x: x["id"])

    # Save backup before updating
    backup_file = f"/tmp/genie_config_{space_id}.json"
    with open(backup_file, "w") as f:
        json.dump(config, f, indent=2)
    print(f"💾 Saved backup to {backup_file}")

    # Update space using SDK
    result = w.genie.update_space(
        space_id=space_id,
        title="HR Analytics",
        description="Employee and payroll analytics workspace",
        serialized_space=json.dumps(config),
        warehouse_id=warehouse_id
    )

    print(f"✅ Updated Genie Space: {result.space_id}")
    print(f"   Title: {result.title}")
    print(f"   Warehouse: {result.warehouse_id}")

    return result

# Usage
if __name__ == "__main__":
    result = configure_genie_space(
        space_id="your-space-id",
        warehouse_id="your-warehouse-id"
    )
```

---

## Workflow

### SDK Workflow (Recommended)

#### Step 1: Create Space with Initial Configuration

```python
from databricks.sdk import WorkspaceClient
import json

w = WorkspaceClient()

# Create space with basic config
space = w.genie.create_space(
    warehouse_id="your-warehouse-id",
    title="HR Analytics",
    description="Employee and payroll analytics",
    serialized_space=json.dumps({
        "version": 2,
        "data_sources": {
            "tables": [
                {"identifier": "my_catalog.my_schema.dim_employee"},
                {"identifier": "my_catalog.my_schema.fact_payroll"}
            ]
        },
        "config": {},
        "instructions": {}
    })
)

space_id = space.space_id
```

#### Step 2: Test Baseline Queries

```python
from datetime import timedelta

result = w.genie.start_conversation_and_wait(
    space_id=space_id,
    content="What is the total headcount?",
    timeout=timedelta(seconds=120)
)

# Review the generated SQL
print(result.attachments[0].query.query if result.attachments else "No query")
```

#### Step 3: Add Advanced Configuration

Build full configuration with joins, filters, dimensions, and measures:

```python
config = {...}  # Build configuration as shown in "Complete Implementation Example"

# Update with advanced config
w.genie.update_space(
    space_id=space_id,
    serialized_space=json.dumps(config)
)
```

#### Step 4: Validate Improvements

```python
result = w.genie.start_conversation_and_wait(
    space_id=space_id,
    content="What is the total headcount?",
    timeout=timedelta(seconds=120)
)

# SQL should now be more accurate and use configured filters
print(result.attachments[0].query.query)
```

#### Step 5: Iterate

```python
# Load saved config
with open(f"/tmp/genie_config_{space_id}.json") as f:
    config = json.load(f)

# Add new filter/measure/dimension
config["instructions"]["sql_snippets"]["filters"].append({...})

# Sort and update
sort_config(config)
w.genie.update_space(space_id=space_id, serialized_space=json.dumps(config))
```

---

### MCP Tools Workflow (Alternative)

For interactive development with Claude Code:

#### Step 1: Create Basic Space

```python
result = create_or_update_genie(
    display_name="HR Analytics",
    table_identifiers=[
        "my_catalog.my_schema.dim_employee",
        "my_catalog.my_schema.fact_payroll"
    ]
)
space_id = result["space_id"]
```

#### Step 2: Test Queries

```python
result = ask_genie(space_id=space_id, question="What is the total headcount?")
print(result["sql"])
```

#### Step 3: Add Advanced Configuration

Switch to SDK for advanced configuration (see SDK workflow above).

#### Step 4: Validate with MCP

```python
result = ask_genie(space_id=space_id, question="What is the total headcount?")
print(result["sql"])  # Should now use configured filters
```

---

## Retrieving Current Configuration

To see what's currently configured (useful before updates):

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Get space details (basic info only)
space = w.genie.get_space(space_id=space_id)
print(f"Title: {space.title}")
print(f"Description: {space.description}")

# Note: The SDK's get_space() doesn't return serialized_space
# You must maintain your own backup/version control of configurations
```

**Important**: The `serialized_space` field is write-only. The SDK does not return it in GET responses. Always save your configuration to a file or version control.

---

## Best Practices

### Configuration Quality

1. **Start with instructions** - Text instructions are the most impactful
2. **Define all key joins** - Cover every relationship users might query
3. **Add common filters** - Cover the 80% use cases
4. **Include date dimensions** - YEAR, MONTH, QUARTER for every date column
5. **Define core metrics** - All key business metrics as measures
6. **Provide examples** - 6-10 example queries covering common patterns

### Development Workflow

1. **Version control** - Store configurations in git
2. **Backup before updates** - Always save current config before changing
3. **Test incrementally** - Add one section at a time, test, then add more
4. **Validate sorting** - Use the `sort_config()` helper function
5. **Monitor query quality** - Track whether configuration improves results

### Production Deployment

1. **Use CI/CD** - Deploy configurations via automated pipelines
2. **Separate environments** - Dev/staging/prod spaces
3. **Document changes** - Keep changelog of configuration updates
4. **Monitor usage** - Track which measures/filters are actually used

---

## Common Issues and Solutions

### Issue: "instructions must be sorted by id"

**Cause**: Lists are not sorted by `id` field

**Solution**: Always call `sort_config()` before updating:

```python
config = {...}

# Sort all lists
config["config"]["sample_questions"].sort(key=lambda x: x["id"])
config["data_sources"]["tables"].sort(key=lambda x: x["identifier"])
for key in ["text_instructions", "example_question_sqls", "join_specs"]:
    config["instructions"][key].sort(key=lambda x: x["id"])
for key in ["filters", "expressions", "measures"]:
    config["instructions"]["sql_snippets"][key].sort(key=lambda x: x["id"])
```

---

### Issue: Joins don't work correctly

**Causes**:
- Incorrect alias format (not backtick-quoted)
- Relationship type missing or on wrong line
- Alias doesn't match table name

**Solution**:

```python
# ✅ Correct
{
    "sql": [
        "`orders`.`customer_id` = `customers`.`id`",  # Backtick-quoted
        "--rt=FROM_RELATIONSHIP_TYPE_MANY_TO_ONE--"   # Separate line
    ]
}

# ❌ Wrong
{
    "sql": [
        "orders.customer_id = customers.id --rt=FROM_RELATIONSHIP_TYPE_MANY_TO_ONE--"
    ]
}
```

---

### Issue: Configuration gets wiped

**Cause**: SDK `update_space` with `serialized_space` overwrites entire config

**Solution**: Always fetch current config, merge changes, then update:

```python
# Load your saved configuration
with open(f"/tmp/genie_config_{space_id}.json") as f:
    config = json.load(f)

# Add new filter
new_filter = {
    "id": hex_id(),
    "sql": ["column = 'value'"],
    "display_name": "New Filter",
    "instruction": ["..."],
    "synonyms": ["..."]
}
config["instructions"]["sql_snippets"]["filters"].append(new_filter)

# Sort and update
config["instructions"]["sql_snippets"]["filters"].sort(key=lambda x: x["id"])
w.genie.update_space(space_id=space_id, serialized_space=json.dumps(config))
```

---

### Issue: Complex SQL doesn't fit in one array element

**Cause**: Very long SQL statements

**Solution**: Split across multiple array elements (each line is one element):

```python
{
    "sql": [
        "SELECT dept,\n",
        "  COUNT(*) as cnt,\n",
        "  AVG(salary) as avg_sal\n",
        "FROM employees\n",
        "WHERE status = 'Active'\n",
        "GROUP BY dept\n",
        "ORDER BY cnt DESC"
    ]
}
```

---

## Additional Resources

- **MCP Tools Reference**: [mcp-tools-reference.md](mcp-tools-reference.md) - Basic space operations
- **Space Creation Guide**: [spaces.md](spaces.md) - Initial space setup workflow
- **Conversation API**: [conversation.md](conversation.md) - Querying Genie Spaces

---

## Exporting for Reference

You can view configured spaces in the Databricks UI and use the UI as a reference for the expected format. However, note that the UI may use different terminology or structure than the programmatic API.

For a canonical reference, maintain your configurations in version control and use the examples in this guide as templates.
