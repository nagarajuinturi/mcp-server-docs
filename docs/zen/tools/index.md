---
title: Tools
description: Built-in tools available when using the Actian MCP Server with Actian Zen.
---

# Tools

The Actian MCP Server for Actian Zen provides built-in tools for database access, ORM operations, blob handling, and server management. Six tools are registered at a time. Which six depends on `query_mode`, because the write tools replace the two administrative ones.

## Available Tools

Use the following tools to interact with the database:

| Tool | Description | `read-only` | `read-write` |
|------|-------------|:-----------:|:------------:|
| [`execute_query`](#execute_query) | Runs a `SELECT` query with automatic Zen dialect translation. | ✓ | ✓ |
| [`execute_write_query`](#execute_write_query) | Runs a validated `INSERT`, `UPDATE`, `DELETE`, or `MERGE` statement. | — | ✓ |
| [`list_tables`](#list_tables) | Lists all user tables from the Zen catalog. | ✓ | ✓ |
| [`describe_table`](#describe_table) | Returns column metadata, primary keys, and foreign keys for a table. | ✓ | ✓ |
| [`orm_operation`](#orm_operation) | Runs structured queries through SQLAlchemy. Also inserts, updates, and deletes single rows in `read-write`. | ✓ | ✓ |
| [`batch_operation`](#batch_operation) | Inserts, updates, or deletes many rows at once, and counts rows. | — | ✓ |
| [`blob_operation`](#blob_operation) | Lists and downloads file and blob data. | ✓ | — |
| [`database_manage`](#database_manage) | Queries server capabilities, lists DSNs, and releases locks. | ✓ | — |

!!! note "Writes use dedicated tools on Zen"
    Unlike the other Actian databases, `execute_query` on Zen never runs a write, in any mode. It accepts `SELECT` only, and rejects Data Manipulation Language (DML) with a pointer to `execute_write_query`. Set `query_mode` to `read-write` to register the write tools. Each write then requires the `mcp:write` scope and human approval. For more information, see [Write support](../../intro/write-support.md).

!!! warning "`blob_operation` and `database_manage` are not available in write mode"
    Enabling `read-write` registers `execute_write_query` and `batch_operation` and unregisters these two. A deployment that needs blob downloads or DSN management at the same time as writes is not supported in this release.

---

## execute_query

Executes a read-only SQL query against Actian Zen with automatic dialect translation. It supports complex queries like JOINs, subqueries, aggregations, and UNION.

Only `SELECT` is accepted, in both `read-only` and `read-write` mode. To modify data, use [`execute_write_query`](#execute_write_query).

!!! note "Auto-translations"
    The following translations are applied automatically before execution:

    - `LEN()` → `CHAR_LENGTH()` (Zen does not support `LEN()`)
    - `INFORMATION_SCHEMA` queries → `dbo.fSQL*()` catalog functions
    - Constraint names are truncated to 20 characters (Zen limit)

### Parameters

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `sql` | `string` | ✓ | Read-only SQL query to execute. |

### Output Schema

**On Success**

```json
{
  "results": [{"column": "value"}],
  "row_count": 2,
  "method": "execute_query"
}
```

**When results are truncated**

Returned when the result set exceeds `max_rows` (default: `1000`).

```json
{
  "results": [{"column": "value"}],
  "row_count": 1000,
  "truncated": true,
  "truncation_note": "Results limited to 1000 rows. Use WHERE to narrow results.",
  "method": "execute_query"
}
```

**When dialect translation is applied**

```json
{
  "results": [],
  "row_count": 0,
  "translated": true,
  "translation_note": "Translated LEN() to CHAR_LENGTH() for Zen compatibility",
  "original_sql": "SELECT LEN(name) FROM customers",
  "method": "execute_query"
}
```

### Example

**User Request**

```
Show me everyone whose last name starts with S
```

**Input**

```json
{
  "sql": "SELECT * FROM Person WHERE Last_Name LIKE 'S%' ORDER BY First_Name"
}
```

**Response**

```json
{
  "results": [
    {"ID": 101, "First_Name": "Alice", "Last_Name": "Smith"},
    {"ID": 102, "First_Name": "Bob", "Last_Name": "Sanders"}
  ],
  "row_count": 2,
  "method": "execute_query"
}
```

### Routing Errors

**A write was sent to this tool while `query_mode` is `read-write`**

The response names the tool to use instead, so the client can retry without help.

```json
{
  "error": "DML not allowed in execute_query. Use execute_write_query for INSERT/UPDATE/DELETE.",
  "alternative": "execute_write_query",
  "sql_classification": "DML"
}
```

**The statement is Data Definition Language**

```json
{
  "error": "DDL not allowed in this mode. Schema changes are blocked in 1.1 (deferred to full mode).",
  "alternative": null,
  "sql_classification": "DDL"
}
```

**Anything else**

Explicit transaction control, stored-procedure calls, and `SHOW` / `DESCRIBE` / `EXPLAIN` are not served by this tool. Use `list_tables`, `describe_table`, or the schema resource for metadata.

```json
{
  "error": "Statement is not supported via execute_query in read-write mode. Explicit transactions, stored-procedure CALL/EXEC and SHOW/DESCRIBE/EXPLAIN are not permitted; use the schema tools or resource API for metadata.",
  "sql_classification": "UNKNOWN"
}
```

In `read-only` mode a non-`SELECT` statement is rejected with `Only SELECT queries are permitted`.

---

## execute_write_query

Executes a single `INSERT`, `UPDATE`, `DELETE`, or `MERGE` statement. Registered only when `query_mode` is `read-write`.

Every statement passes validation, the `mcp:write` scope check, and human approval, in that order, before it reaches the database. See [Write support](../../intro/write-support.md).

!!! warning "Validation rules that reject a statement outright"
    These are enforced before anyone is asked to approve anything:

    - `UPDATE` and `DELETE` must carry a `WHERE` clause. A mass update is rejected.
    - Writes to the Zen system catalog (the `X$` tables) are blocked.
    - Multiple statements in one call are rejected.
    - Data Definition Language is rejected, including a `CREATE` or `DROP` nested inside the statement.

### Parameters

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `sql` | `string` | ✓ | The `INSERT`, `UPDATE`, `DELETE`, or `MERGE` statement to execute. |

### Output Schema

**On Success**

```json
{
  "sql": "<executed_statement>",
  "rows_affected": "<num_rows>",
  "success": true,
  "method": "execute_write_query"
}
```

When dialect translation is applied, the response also carries `translated` and `translation_note`.

**On Error**

```json
{
  "error": "<error_message>"
}
```

### Example: Writing a Row

**User Request**

```
Set Alice Smith's department to Sales
```

**Input**

```json
{
  "sql": "UPDATE Person SET Department = 'Sales' WHERE ID = 101"
}
```

Before running the statement, the server asks you to approve it in your client. The response depends on your answer.

**Response, when you approve**

```json
{
  "sql": "UPDATE Person SET Department = 'Sales' WHERE ID = 101",
  "rows_affected": 1,
  "success": true,
  "method": "execute_write_query"
}
```

**Response, when you decline, do not answer, or the client cannot show the prompt**

```json
{
  "error": "Write rejected by user",
  "success": false
}
```

### Write Errors

**The token lacks the `mcp:write` scope**

The server checks the scope before it asks anyone to approve the statement, so no prompt appears.

```json
{
  "error": "write operations require the 'mcp:write' scope, which the access token does not carry"
}
```

**`UPDATE` or `DELETE` without a `WHERE` clause**

```json
{
  "error": "UPDATE and DELETE require a WHERE clause"
}
```

**The statement targets the Zen system catalog**

```json
{
  "error": "Write access to system table 'X$File' is not permitted"
}
```

**The statement is not DML**

```json
{
  "error": "Only INSERT, UPDATE, DELETE and MERGE are permitted in write queries"
}
```

**More than one statement was passed**

```json
{
  "error": "Multi-statement queries are not permitted"
}
```

---

## list_tables

Returns all user tables in the connected database by querying the Zen `dbo.fSQLTables()` catalog function. System tables are excluded.

### Parameters

This tool takes no input parameters.

### Output Schema

**On Success**

```json
{
  "tables": ["<table_name>"],
  "count": "<num_tables>"
}
```

### Example

**User Request**

```
Show me all the tables in my database
```

**Response**

```json
{
  "tables": ["Person", "Department", "Billing", "Student", "Class", "Tuition", "Faculty"],
  "count": 7
}
```

---

## describe_table

Returns column metadata for a table, including names, types, precision, scale, nullability, defaults, primary keys, and foreign keys. Internally uses the `dbo.fSQLColumns()`, `dbo.fSQLPrimaryKeys()`, and `dbo.fSQLForeignKeys()` catalog functions.

### Parameters

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `table` | `string` | ✓ | Name of the table to describe. |

### Output Schema

**On Success**

```json
{
  "table_name": "<table_name>",
  "columns": [
    {
      "name": "<column_name>",
      "type": "<column_type>",
      "precision": "<precision>",
      "scale": "<scale>",
      "nullable": "<true|false>",
      "default": "<default_value>",
      "primary_key": "<true|false>"
    }
  ],
  "primary_keys": ["<primary_key_column>"],
  "foreign_keys": []
}
```

### Example

**User Request**

```
Show me schema information about the Person table
```

**Input**

```json
{
  "table": "Person"
}
```

**Response**

```json
{
  "table_name": "Person",
  "columns": [
    {
      "name": "ID",
      "type": "BIGIDENTITY",
      "precision": 19,
      "scale": 0,
      "nullable": false,
      "default": null,
      "primary_key": true
    }
  ],
  "primary_keys": ["ID"],
  "foreign_keys": []
}
```

---

## orm_operation

Performs structured operations through SQLAlchemy with dynamic model creation, handling the Zen SQL dialect automatically.

`select` works in every mode and supports JOINs (up to 3 tables), WHERE conditions, ORDER BY, GROUP BY, HAVING, LIMIT, OFFSET, and the aggregates `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX`.

`insert`, `update`, and `delete` act on a single row and require `query_mode` set to `read-write`. Like `execute_write_query`, each one is gated by the `mcp:write` scope and human approval. The approval prompt shows a preview of the statement the operation will run.

### Parameters

**Required**

| Field | Type | Description |
|-------|------|-------------|
| `operation` | `string` | One of `select`, `insert`, `update`, `delete`. |
| `table` | `string` | Target table name. |

**Optional**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `columns` | `list` | All columns | Columns to return. Supports aggregates like `COUNT(*) AS total`. `select` only. |
| `where` | `dict` | — | Filter conditions, for example, `{"field": "salary", "operator": ">", "value": 50000}`. `select` only. |
| `order_by` | `list` | — | Column names to sort by. `select` only. |
| `limit` | `integer` | `max_rows` | Maximum rows to return. Capped at `max_rows`. `select` only. |
| `offset` | `integer` | — | Number of rows to skip. `select` only. |
| `joins` | `list` | — | Join specs: `[{"table": "dept", "on": "p.dept_id = dept.id", "type": "LEFT"}]`. `select` only. |
| `group_by` | `list` | — | Columns to group by. `select` only. |
| `having` | `dict` | — | HAVING conditions for grouped queries. `select` only. |
| `data` | `dict` | — | Column and value pairs to write. Required for `insert` and `update`. |
| `entity_id` | `integer` | — | Primary key of the target row. Required for `update` and `delete`. |

### Output Schema

**On Success, for `select`**

```json
{
  "results": [{"column": "value"}],
  "row_count": 2,
  "method": "orm_operation"
}
```

**On Success, for `insert` and `update`**

The response carries the row as stored, so server-generated values such as an IDENTITY primary key come back without a second query.

```json
{
  "created": true,
  "entity": {"<column_name>": "<value>"}
}
```

An `update` returns the same shape with `updated` in place of `created`.

**On Success, for `delete`**

```json
{
  "deleted": true,
  "entity_id": "<primary_key>"
}
```

### Example: Reading

**User Request**

```
How many people have a last name starting with S?
```

**Input**

```json
{
  "operation": "select",
  "table": "Person",
  "columns": ["COUNT(*) AS total"],
  "where": {"field": "Last_Name", "operator": "LIKE", "value": "S%"}
}
```

### Example: Inserting a Row

This example needs `query_mode` set to `read-write`, and you are asked to approve the write before it runs.

**User Request**

```
Add Carol Jones to the Person table
```

**Input**

```json
{
  "operation": "insert",
  "table": "Person",
  "data": {"First_Name": "Carol", "Last_Name": "Jones"}
}
```

**Response**

```json
{
  "created": true,
  "entity": {"ID": 103, "First_Name": "Carol", "Last_Name": "Jones"}
}
```

### Write Errors

**The server is in `read-only` mode**

```json
{
  "error": "Operation 'insert' not allowed in readonly mode"
}
```

**The write was not approved**

```json
{
  "error": "Write rejected by user"
}
```

**The target row does not exist**

```json
{
  "updated": false,
  "error": "Entity not found"
}
```

---

## batch_operation

Inserts, updates, or deletes many rows in one call, and counts rows. Registered only when `query_mode` is `read-write`.

The three writing modes are gated by the `mcp:write` scope and human approval. For `batch_update` and `batch_delete` the server first runs a `COUNT(*)` with your `where_clause` and shows the number of matching rows in the approval prompt, so you see the blast radius before you agree to it.

`count` reads only and is not gated.

### Parameters

**Required**

| Field | Type | Description |
|-------|------|-------------|
| `mode` | `string` | One of `batch_insert`, `batch_update`, `batch_delete`, `count`. |
| `table` | `string` | Target table name. |

**Mode-dependent**

| Field | Type | Description |
|-------|------|-------------|
| `data` | `list` | Rows to insert, each a dict of column and value pairs. Required for `batch_insert`. |
| `updates` | `dict` | Column and value pairs to set. Required for `batch_update`. |
| `where_clause` | `string` | Predicate without the `WHERE` keyword, for example `Department = ?`. Required for `batch_update` and `batch_delete`, optional for `count`. |
| `where_params` | `list` | Values bound to the placeholders in `where_clause`. |

### Output Schema

**On Success**

Each mode names its own count field.

```json
{"table": "<table_name>", "rows_inserted": "<num_rows>", "success": true}
```

```json
{"table": "<table_name>", "rows_updated": "<num_rows>", "success": true}
```

```json
{"table": "<table_name>", "rows_deleted": "<num_rows>", "success": true}
```

```json
{"table": "<table_name>", "row_count": "<num_rows>"}
```

**On Error**

```json
{
  "error": "<error_message>"
}
```

### Example: Inserting Several Rows

**User Request**

```
Add these two people to the Person table
```

**Input**

```json
{
  "mode": "batch_insert",
  "table": "Person",
  "data": [
    {"First_Name": "Dan", "Last_Name": "Reed"},
    {"First_Name": "Erin", "Last_Name": "Vale"}
  ]
}
```

**Response**

```json
{
  "table": "Person",
  "rows_inserted": 2,
  "success": true
}
```

### Example: Updating Matching Rows

**User Request**

```
Move everyone in Support to the Sales department
```

**Input**

```json
{
  "mode": "batch_update",
  "table": "Person",
  "updates": {"Department": "Sales"},
  "where_clause": "Department = ?",
  "where_params": ["Support"]
}
```

The approval prompt reports how many rows match `Department = 'Support'` before you decide.

**Response**

```json
{
  "table": "Person",
  "rows_updated": 4,
  "success": true
}
```

### Errors

**A required parameter for the mode is missing**

```json
{
  "error": "updates and where_clause are required for batch_update mode"
}
```

`batch_delete` without a `where_clause` is rejected the same way, so a batch delete can never clear a whole table.

**The write was not approved**

```json
{
  "error": "Write rejected by user"
}
```

---

## blob_operation

Lists and downloads file or blob data from tables that store binary content. Available in `read-only` mode.

### Parameters

**Required**

| Field | Type | Description |
|-------|------|-------------|
| `action` | `string` | One of: `list`, `download`. |
| `table_name` | `string` | Table that stores blob data. |

**Optional**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `file_id` | `integer` | — | Row identifier. Required when `action` is `download`. |
| `output_path` | `string` | — | Destination file path. Required when `action` is `download`. |
| `id_column` | `string` | `id` | Name of the ID column. |
| `blob_column` | `string` | `file_data` | Name of the blob column. |

### Output Schema

**For `list`**

```json
{
  "files": [{"id": "<row_id>", "filename": "<filename>"}],
  "count": "<num_files>"
}
```

**For `download`**

```json
{
  "success": true,
  "output_path": "<destination_path>",
  "size": "<file_size_bytes>"
}
```

### Example

**Input**

```json
{
  "action": "list",
  "table_name": "documents"
}
```

**Response**

```json
{
  "files": [{"id": 1, "filename": "report.pdf"}],
  "count": 1
}
```

---

## database_manage

Provides server management operations: list available databases, list DSNs with details, query server capabilities, and release locks. Available in `read-only` mode.

### Parameters

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `action` | `string` | ✓ | One of: `list`, `list_dsns`, `switch`, `capabilities`, `release_locks`. |

### Output Schema

**For `capabilities`**

```json
{
  "server": "Actian Zen",
  "features": ["sql", "blobs"]
}
```

**For `list_dsns`**

```json
{
  "current_dsn": "<active_dsn>",
  "available_dsns": {
    "<dsn_name>": {"driver": "<driver_name>"}
  },
  "count": "<num_dsns>"
}
```

### Example

**Input**

```json
{
  "action": "capabilities"
}
```

**Response**

```json
{
  "server": "Actian Zen",
  "features": ["sql", "blobs"]
}
```

---

## Next Steps

<div class="grid cards" markdown>

- :material-pencil: **[Write support](../../intro/write-support.md)**  
  Enable writes, and understand the scope check and the approval prompt.

- :material-folder-open: **[Resources](../resources/index.md)**  
  Explore the resource types available through the Zen server.

- :material-message-text: **[Prompts](../prompts/index.md)**  
  Use the built-in prompt templates for common workflows.

- :material-puzzle: **[Extensions](../../extensions/index.md)**  
  Add your own tools to the server with a Python extension.

</div>
