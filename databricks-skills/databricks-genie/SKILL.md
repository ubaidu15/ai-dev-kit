---
name: databricks-genie
description: "Create, configure, and query Databricks Genie Spaces for natural language SQL exploration. Use when building Genie Spaces, configuring advanced instructions/joins/filters/measures via the Databricks Python SDK, or asking questions via the Genie Conversation API."
---

# Databricks Genie

Create and query Databricks Genie Spaces - natural language interfaces for SQL-based data exploration.

## Overview

Genie Spaces allow users to ask natural language questions about structured data in Unity Catalog. The system translates questions into SQL queries, executes them on a SQL warehouse, and presents results conversationally.

## When to Use This Skill

Use this skill when:
- Creating a new Genie Space for data exploration
- **Configuring a Genie Space** with instructions, joins, filters, dimensions, measures, and example queries via the Databricks Python SDK
- Adding sample questions to guide users
- Connecting Unity Catalog tables to a conversational interface
- Asking questions to a Genie Space programmatically (Conversation API)

## Approach Options

### Option 1: Databricks Python SDK (Recommended for Production)

Use the Databricks SDK for type-safe, production-ready implementations:

```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()

# Full type safety, CI/CD integration, version control
space = w.genie.create_space(...)
w.genie.update_space(serialized_space=...)
result = w.genie.start_conversation_and_wait(...)
```

**Benefits**: Type safety, IDE autocomplete, error handling, production-ready, CI/CD integration

**See**: [space-configuration.md](space-configuration.md) for complete SDK implementation guide

### Option 2: Databricks CLI

Use the CLI for quick operations and shell scripts:

```bash
databricks genie create --title "Sales Analytics" --warehouse-id abc123
databricks genie ask --space-id abc123 "What were total sales?"
databricks tables get my_catalog.sales.customers
```

**Benefits**: No code required, shell integration, quick commands

### Option 3: MCP Tools (Interactive Development)

For interactive development with Claude Code, MCP tools are available:

| Tool | Purpose |
|------|---------|
| `create_or_update_genie` | Create or update a Genie Space |
| `ask_genie` | Ask questions to a space |
| `get_table_details` | Inspect table schemas |
| `list_genie`, `get_genie`, `delete_genie` | Space management |

**Benefits**: No setup, works directly in Claude Code, fast prototyping

**See**: [mcp-tools-reference.md](mcp-tools-reference.md) for complete tool documentation

## Quick Start

### Using Databricks Python SDK (Recommended)

```python
from databricks.sdk import WorkspaceClient
from datetime import timedelta

w = WorkspaceClient()

# 1. Inspect tables
table = w.tables.get(full_name="my_catalog.sales.customers")
print(f"Table: {table.name}, Columns: {len(table.columns)}")

# 2. Create Genie Space
space = w.genie.create_space(
    warehouse_id="your-warehouse-id",
    title="Sales Analytics",
    description="Explore sales data with natural language",
    serialized_space='{"version": 2, "data_sources": {"tables": [{"identifier": "my_catalog.sales.customers"}]}, "config": {}, "instructions": {}}'
)

# 3. Ask questions
result = w.genie.start_conversation_and_wait(
    space_id=space.space_id,
    content="What were total sales last month?",
    timeout=timedelta(seconds=120)
)
print(f"SQL: {result.attachments[0].query.query}")
```

### Using MCP Tools (Alternative)

For interactive development with Claude Code, you can use MCP tools:

```python
# 1. Inspect tables
get_table_details(catalog="my_catalog", schema="sales", table_stat_level="SIMPLE")

# 2. Create space
create_or_update_genie(
    display_name="Sales Analytics",
    table_identifiers=["my_catalog.sales.customers"],
    description="Explore sales data with natural language"
)

# 3. Ask questions
ask_genie(space_id="your_space_id", question="What were total sales last month?")
```

## Workflow

### SDK/CLI Workflow (Recommended)

```
1. Inspect tables    → w.tables.get() or CLI: databricks tables get
2. Create space      → w.genie.create_space() or CLI: databricks genie create
3. Configure space   → w.genie.update_space(serialized_space=...) - see space-configuration.md
4. Query space       → w.genie.start_conversation_and_wait() or CLI: databricks genie ask
5. Iterate           → w.genie.update_space() - refine instructions/joins/measures
```

**Benefits**: Type safety, CI/CD integration, version control, production-ready

### MCP Tools Workflow (Alternative for Interactive Development)

```
1. Inspect tables    → get_table_details
2. Create space      → create_or_update_genie
3. Configure space   → Use SDK for advanced config (see space-configuration.md)
4. Query space       → ask_genie
5. Iterate           → Use SDK to refine configuration
```

**Benefits**: No code setup, works directly in Claude Code, fast prototyping

## Prerequisites

### For SDK/CLI Usage (Recommended)

```bash
# Install Databricks SDK
pip install databricks-sdk

# Configure authentication (~/.databrickscfg or environment variables)
export DATABRICKS_HOST="https://your-workspace.cloud.databricks.com"
export DATABRICKS_TOKEN="dapi..."
```

### Data Requirements

1. **Tables in Unity Catalog** - Bronze/silver/gold tables with the data
2. **SQL Warehouse** - A warehouse to execute queries (auto-detected if not specified)

**Creating tables**: Use `synthetic-data-generation` → `spark-declarative-pipelines` skills

## Reference Files

- **[space-configuration.md](space-configuration.md)** - Complete SDK implementation guide (joins, filters, dimensions, measures, examples)
- **[spaces.md](spaces.md)** - Creating and managing Genie Spaces
- **[conversation.md](conversation.md)** - Asking questions via the Conversation API
- **[mcp-tools-reference.md](mcp-tools-reference.md)** - MCP tools reference (alternative approach)

## Common Issues

| Issue | Solution |
|-------|----------|
| **No warehouse available** | Create a SQL warehouse or provide `warehouse_id` explicitly |
| **Poor query generation** | Configure instructions, joins, filters, dimensions, measures, and example queries via the [Databricks Python SDK](space-configuration.md) |
| **Slow queries** | Ensure warehouse is running; use OPTIMIZE on tables |
