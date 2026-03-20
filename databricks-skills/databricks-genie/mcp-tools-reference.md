# Databricks Genie MCP Tools Reference

Complete reference for all MCP tools available for working with Genie Spaces.

---

## Space Management Tools

### `list_genie`

List all Genie Spaces accessible to you.

**Parameters**: None

**Returns**:
```python
{
    "spaces": [
        {
            "space_id": "abc123...",
            "title": "Sales Analytics",
            "description": "Explore sales data"
        },
        ...
    ]
}
```

**Example**:
```python
list_genie()
```

---

### `create_or_update_genie`

Create a new Genie Space or update an existing one.

**Parameters**:
- `display_name` (str, required): Display name for the Genie space
- `table_identifiers` (List[str], required): Tables to include (e.g., `["catalog.schema.customers"]`)
- `warehouse_id` (str, optional): SQL warehouse ID. Auto-detects if not provided
- `description` (str, optional): Description of the space
- `sample_questions` (List[str], optional): Sample questions to guide users
- `space_id` (str, optional): Existing space_id to update instead of create

**Returns**:
```python
{
    "space_id": "abc123...",
    "display_name": "Sales Analytics",
    "operation": "created",  # or "updated"
    "warehouse_id": "warehouse-id",
    "table_count": 2
}
```

**Example (Create)**:
```python
create_or_update_genie(
    display_name="Sales Analytics",
    table_identifiers=[
        "my_catalog.sales.customers",
        "my_catalog.sales.orders"
    ],
    description="Explore sales data with natural language",
    sample_questions=[
        "What were total sales last month?",
        "Who are our top 10 customers?"
    ]
)
```

**Example (Update)**:
```python
create_or_update_genie(
    space_id="abc123...",
    display_name="Sales Analytics",
    table_identifiers=["my_catalog.sales.customers"],
    description="Updated description"
)
```

---

### `get_genie`

Get details of a specific Genie Space.

**Parameters**:
- `space_id` (str, required): The Genie space ID

**Returns**:
```python
{
    "space_id": "abc123...",
    "display_name": "Sales Analytics",
    "description": "Explore sales data",
    "warehouse_id": "warehouse-id",
    "table_identifiers": ["catalog.schema.table1", "catalog.schema.table2"],
    "sample_questions": ["Question 1", "Question 2"]
}
```

**Example**:
```python
get_genie(space_id="abc123...")
```

---

### `delete_genie`

Delete a Genie Space permanently.

**Parameters**:
- `space_id` (str, required): The Genie space ID to delete

**Returns**:
```python
{
    "success": True,
    "space_id": "abc123..."
}
```

**Example**:
```python
delete_genie(space_id="abc123...")
```

**Warning**: This action cannot be undone.

---

## Conversation API Tools

### `ask_genie`

Ask a natural language question to a Genie Space and get SQL-generated results.

**Parameters**:
- `space_id` (str, required): The Genie Space ID to query
- `question` (str, required): The natural language question
- `conversation_id` (str, optional): Continue an existing conversation (for follow-ups)
- `timeout_seconds` (int, optional): Maximum wait time (default: 120)

**Returns**:
```python
{
    "question": "What were total sales last month?",
    "conversation_id": "conv-123",
    "message_id": "msg-456",
    "status": "COMPLETED",
    "sql": "SELECT SUM(amount) FROM ...",
    "description": "Calculating total sales for last month",
    "columns": ["total_sales"],
    "data": [[1234567.89]],
    "row_count": 1,
    "text_response": "Total sales last month were $1,234,567.89"
}
```

**Example (New conversation)**:
```python
result = ask_genie(
    space_id="abc123",
    question="What were total sales last month?"
)
```

**Example (Follow-up question)**:
```python
# Continue the conversation from previous result
ask_genie(
    space_id="abc123",
    question="Break that down by region",
    conversation_id=result["conversation_id"]
)
```

---

### `ask_genie_followup`

Ask a follow-up question in an existing conversation. This is a convenience wrapper around `ask_genie` with `conversation_id`.

**Parameters**:
- `space_id` (str, required): The Genie Space ID
- `conversation_id` (str, required): Conversation ID from previous ask_genie response
- `question` (str, required): Follow-up question
- `timeout_seconds` (int, optional): Maximum wait time (default: 120)

**Returns**: Same structure as `ask_genie`

**Example**:
```python
# First question
result = ask_genie(space_id="abc123", question="Show me sales by month")

# Follow-up
ask_genie_followup(
    space_id="abc123",
    conversation_id=result["conversation_id"],
    question="Now show only Q4"
)
```

---

## Supporting Tools

### `get_table_details`

Inspect table schemas before creating a Genie Space. This helps you understand the data structure and plan your space configuration.

**Parameters**:
- `catalog` (str, required): Catalog name
- `schema` (str, required): Schema name
- `table` (str, optional): Specific table name. If omitted, lists all tables in schema
- `table_stat_level` (str, optional): Detail level - "SIMPLE" (default) or "DETAILED"

**Returns**:
```python
{
    "tables": [
        {
            "name": "customers",
            "row_count": 10000,
            "columns": [
                {"name": "customer_id", "type": "BIGINT", "nullable": False},
                {"name": "name", "type": "STRING", "nullable": True},
                {"name": "region", "type": "STRING", "nullable": True}
            ]
        }
    ]
}
```

**Example**:
```python
# Inspect all tables in a schema
get_table_details(
    catalog="my_catalog",
    schema="sales",
    table_stat_level="SIMPLE"
)

# Inspect a specific table
get_table_details(
    catalog="my_catalog",
    schema="sales",
    table="customers"
)
```

---

### `execute_sql`

Test SQL queries directly on a warehouse. Useful for validating queries before adding them to Genie Space configuration.

**Parameters**:
- `warehouse_id` (str, required): SQL warehouse ID
- `sql` (str, required): SQL query to execute
- `catalog` (str, optional): Default catalog
- `schema` (str, optional): Default schema

**Returns**:
```python
{
    "status": "SUCCEEDED",
    "columns": ["customer_name", "total_sales"],
    "data": [
        ["John Doe", 10000.00],
        ["Jane Smith", 15000.00]
    ],
    "row_count": 2
}
```

**Example**:
```python
execute_sql(
    warehouse_id="warehouse-123",
    sql="SELECT name, SUM(amount) as total FROM orders GROUP BY name",
    catalog="my_catalog",
    schema="sales"
)
```

---

## Common Patterns

### Pattern 1: Create Space from Scratch

```python
# Step 1: Inspect tables
tables = get_table_details(
    catalog="my_catalog",
    schema="sales",
    table_stat_level="SIMPLE"
)

# Step 2: Review table structure
print(tables)

# Step 3: Create Genie Space
result = create_or_update_genie(
    display_name="Sales Analytics",
    table_identifiers=[
        "my_catalog.sales.customers",
        "my_catalog.sales.orders"
    ],
    description="Explore sales data with natural language",
    sample_questions=[
        "What were total sales last month?",
        "Who are our top 10 customers?"
    ]
)

print(f"Created space: {result['space_id']}")
```

---

### Pattern 2: Test Query Before Adding to Space

```python
# Test the SQL query first
result = execute_sql(
    warehouse_id="warehouse-123",
    sql="SELECT region, COUNT(*) as customer_count FROM customers GROUP BY region",
    catalog="my_catalog",
    schema="sales"
)

# If successful, add to space configuration
# (Use SDK approach from space-configuration.md)
```

---

### Pattern 3: Conversational Q&A

```python
# Start conversation
result = ask_genie(
    space_id="abc123",
    question="What were total sales last month?"
)

# Follow up with more questions
result2 = ask_genie(
    space_id="abc123",
    question="Break that down by region",
    conversation_id=result["conversation_id"]
)

result3 = ask_genie(
    space_id="abc123",
    question="Which region had the highest growth?",
    conversation_id=result["conversation_id"]
)
```

---

### Pattern 4: List and Inspect All Spaces

```python
# List all spaces
spaces = list_genie()

# Inspect each space
for space in spaces["spaces"]:
    details = get_genie(space_id=space["space_id"])
    print(f"{details['display_name']}: {len(details['table_identifiers'])} tables")
```

---

## Error Handling

All MCP tools return error information in the response when operations fail:

```python
result = create_or_update_genie(...)

if "error" in result:
    print(f"Operation failed: {result['error']}")
else:
    print(f"Success: {result['space_id']}")
```

Common error scenarios:

- **No warehouse available**: Provide explicit `warehouse_id` or create a SQL warehouse
- **Space not found**: Verify `space_id` is correct and you have access
- **Table not found**: Ensure table identifiers use fully-qualified names (`catalog.schema.table`)
- **Timeout**: Increase `timeout_seconds` for complex queries

---

## Best Practices

### When Using MCP Tools

1. **Always inspect tables first** - Use `get_table_details` before creating spaces
2. **Use specific table names** - Always use fully-qualified names (`catalog.schema.table`)
3. **Test SQL queries** - Use `execute_sql` to validate queries before adding to configuration
4. **Keep conversations short** - Long conversation threads may lose context
5. **Handle errors** - Check for `error` key in responses

### When to Use SDK vs MCP Tools

**Use MCP Tools for**:
- ✅ Basic space creation and management
- ✅ Asking questions to Genie Spaces
- ✅ Inspecting tables and testing queries
- ✅ Interactive development and exploration

**Use Databricks Python SDK for**:
- ✅ Advanced space configuration (joins, filters, measures)
- ✅ Production deployments with infrastructure as code
- ✅ Bulk operations across multiple spaces
- ✅ Integration with CI/CD pipelines

See [space-configuration.md](space-configuration.md) for SDK-based advanced configuration.

---

## Tool Implementation

These tools are implemented in the Databricks MCP server at:
```
databricks-mcp-server/databricks_mcp_server/tools/genie.py
```

The tools use the Databricks Python SDK (`databricks.sdk`) and custom `AgentBricksManager` wrapper for underlying API calls.
