# DCL — MCP Integration Guide

> **Status: Live and operational.**
> The MCP endpoint is available at `https://mcp.dataclawe.com/sse`.

---

## DataClawe DCL is the Standard for AI-Native Database Access

DataClawe DCL (DataClawe Command Language) is designed as the standard specification for AI agents to interact with databases.

> **"If your AI agent needs to talk to a database, speak DCL."**

- Simple, readable JSON structure
- Schema-free — insert any data without pre-defining columns
- AI-safety by design — soft deletes, auto-schema evolution
- PostgreSQL backend with JSONB storage
- CC BY 4.0 License — free to implement, extend, and adopt

This guide is for **AI agent developers integrating DataClawe as an MCP tool** (Claude, GPT, Cursor, and other MCP-compatible hosts).

For HTTP-based integration from frontend or backend code, see [README.api.md](README.api.md).

For the full DCL command specification, see [README.md](README.md).

---

## Overview

DataClawe implements the **Model Context Protocol (MCP)** over **SSE (Server-Sent Events)** transport.

The MCP server exposes **18 individual tools** — each corresponding to a specific database operation. AI agents invoke these tools using the standard MCP `tools/call` method.

```
AI Agent
  → JSON-RPC 2.0 (SSE)
    → DataClawe MCP Server (18 tools)
      → PostgreSQL (JSONB)
```

---

## MCP Server Endpoint

```
https://mcp.dataclawe.com/sse
```

Transport: **SSE (Server-Sent Events)**

Authentication is handled during the MCP handshake using your API key.

---

## Data Model

DataClawe uses a two-level namespace: **database** → **table** → **records**.

- **db_name**: A logical database namespace (e.g., `"shop"`, `"crm"`)
- **table_name**: A table within a database (e.g., `"orders"`, `"customers"`)
- **table_id**: A unique integer ID for each record (returned as `_id` in query results)
- **data**: Record fields stored as schema-free JSON (any keys accepted)

Records are stored as JSONB in PostgreSQL. No schema definition is required — columns are auto-detected on first insert.

### Status Values

| Status | Meaning |
|--------|---------|
| `1` | Active (normal record) |
| `8` | Table schema declaration (internal) |
| `9` | Soft-deleted |

---

## Tools Overview

| Tool | Purpose | Read-only |
|------|---------|-----------|
| `dataclawe_query` | Search and retrieve records with filters | ✅ |
| `dataclawe_count` | Count matching records without fetching data | ✅ |
| `dataclawe_schema` | Get column definitions of a table | ✅ |
| `dataclawe_tables` | List tables and record counts | ✅ |
| `dataclawe_stats` | Get overall tenant statistics | ✅ |
| `dataclawe_timeline` | View recent data change history | ✅ |
| `dataclawe_inspect` | Inspect a single record by ID | ✅ |
| `dataclawe_db_list` | List all databases | ✅ |
| `dataclawe_insert` | Insert a new record | ❌ |
| `dataclawe_update` | Update fields of a record | ❌ |
| `dataclawe_delete` | Soft-delete a record | ❌ |
| `dataclawe_table_create` | Create an empty table | ❌ |
| `dataclawe_table_copy` | Copy a table | ❌ |
| `dataclawe_table_rename` | Rename a table | ❌ |
| `dataclawe_table_drop` | Soft-delete a table | ❌ |
| `dataclawe_db_create` | Create a database | ❌ |
| `dataclawe_db_rename` | Rename a database | ❌ |
| `dataclawe_db_drop` | Permanently drop a database | ❌ |

---

## Where Clause — Filter Operators

The `where` parameter is a JSON object used in `dataclawe_query` and `dataclawe_count`.

### Simple Equality

```json
{"status": "paid"}
```
→ `data->>'status' = 'paid'`

### Multiple Conditions (AND)

```json
{"status": "paid", "customer": "Alice"}
```
→ `data->>'status' = 'paid' AND data->>'customer' = 'Alice'`

### Comparison Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `$eq` | Equal | `{"status": {"$eq": "paid"}}` |
| `$ne` | Not equal | `{"status": {"$ne": "cancelled"}}` |
| `$gt` | Greater than | `{"amount": {"$gt": 1000}}` |
| `$gte` | Greater than or equal | `{"amount": {"$gte": 500}}` |
| `$lt` | Less than | `{"amount": {"$lt": 100}}` |
| `$lte` | Less than or equal | `{"amount": {"$lte": 2000}}` |
| `$in` | In array | `{"product": {"$in": ["Plan A", "Plan B"]}}` |
| `$nin` | Not in array | `{"status": {"$nin": ["cancelled", "refunded"]}}` |

### Range Query (combining operators)

```json
{"amount": {"$gte": 500, "$lte": 2000}}
```
→ Records where amount is between 500 and 2000

---

## Tool Call Examples

### dataclawe_query — Search records

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "dataclawe_query",
    "arguments": {
      "db_name": "shop",
      "table_name": "orders",
      "where": {"status": "paid"},
      "limit": 10
    }
  }
}
```

Example response:
```json
{
  "count": 2,
  "records": [
    {"customer": "Alice", "amount": 1500, "status": "paid", "_id": 42, "_created_at": "2026-03-28T10:00:00Z", "_updated_at": "2026-03-28T10:00:00Z"},
    {"customer": "Bob", "amount": 800, "status": "paid", "_id": 41, "_created_at": "2026-03-27T09:00:00Z", "_updated_at": "2026-03-27T09:00:00Z"}
  ]
}
```

**With operators:**
```json
{
  "db_name": "shop",
  "table_name": "orders",
  "where": {"amount": {"$gte": 1000}, "status": {"$in": ["paid", "shipped"]}},
  "limit": 50
}
```

### dataclawe_insert — Add a record

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "dataclawe_insert",
    "arguments": {
      "db_name": "shop",
      "table_name": "orders",
      "data": {
        "customer": "Alice",
        "amount": 1500,
        "status": "pending"
      }
    }
  }
}
```

Response:
```json
{"success": true, "table_id": 42}
```

> **Note:** If the table or database does not exist, they are auto-created. Schema evolves automatically — new fields can be added at any time.

### dataclawe_update — Modify a record

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "dataclawe_update",
    "arguments": {
      "db_name": "shop",
      "table_name": "orders",
      "table_id": 42,
      "data": {"status": "shipped", "shipped_at": "2026-03-30"}
    }
  }
}
```

Response:
```json
{"success": true, "affected": 1}
```

> **Note:** This is a partial update (PATCH semantics). Only the specified fields are overwritten. To find the `table_id`, first use `dataclawe_query` — each record has an `_id` field.

### dataclawe_delete — Soft-delete a record

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "tools/call",
  "params": {
    "name": "dataclawe_delete",
    "arguments": {
      "db_name": "shop",
      "table_name": "orders",
      "table_id": 42
    }
  }
}
```

Response:
```json
{"success": true, "affected": 1}
```

> **Note:** This is a logical (soft) delete. The record is marked as `status=9` and excluded from query results, but can still be viewed via `dataclawe_inspect` or `dataclawe_timeline`.

### dataclawe_count — Count records

```json
{
  "db_name": "shop",
  "table_name": "orders",
  "where": {"status": "paid"}
}
```

Response:
```json
{"db_name": "shop", "table_name": "orders", "count": 15}
```

### dataclawe_schema — Get table structure

```json
{
  "db_name": "shop",
  "table_name": "orders"
}
```

Response:
```json
{
  "found": true,
  "db_name": "shop",
  "table_name": "orders",
  "columns": [
    {"name": "customer", "type": "text"},
    {"name": "amount", "type": "text"},
    {"name": "status", "type": "text"}
  ],
  "column_count": 3
}
```

### dataclawe_tables — List tables

```json
{
  "db_name": "shop"
}
```

Response:
```json
{
  "tables": [
    {"table_name": "orders", "record_count": 42},
    {"table_name": "products", "record_count": 15}
  ]
}
```

### dataclawe_stats — Overall statistics

```json
{}
```

Response:
```json
{
  "active_records": "128",
  "deleted_records": "5",
  "declared_tables": "4",
  "databases": "2",
  "tables": "4",
  "oldest_record": "2026-03-01T00:00:00Z",
  "latest_update": "2026-03-30T12:00:00Z"
}
```

### dataclawe_timeline — Change history

```json
{
  "db_name": "shop",
  "table_name": "orders",
  "limit": 5
}
```

Response:
```json
{
  "events": [
    {"db_name": "shop", "table_name": "orders", "table_id": "42", "status": "9", "created_at": "2026-03-28T10:00:00Z", "updated_at": "2026-03-30T14:00:00Z"},
    {"db_name": "shop", "table_name": "orders", "table_id": "41", "status": "1", "created_at": "2026-03-27T09:00:00Z", "updated_at": "2026-03-27T09:00:00Z"}
  ]
}
```

### dataclawe_inspect — Inspect a single record

```json
{
  "db_name": "shop",
  "table_name": "orders",
  "table_id": 42
}
```

Response:
```json
{
  "found": true,
  "table_id": 42,
  "data": {"customer": "Alice", "amount": 1500, "status": "shipped"},
  "status": 1,
  "created_at": "2026-03-28T10:00:00Z",
  "updated_at": "2026-03-30T14:00:00Z"
}
```

### dataclawe_table_create — Create a table

```json
{
  "db_name": "shop",
  "table_name": "products",
  "schema": {"name": "text", "price": "integer", "in_stock": "boolean"}
}
```

> **Note:** Tables are also auto-created on first INSERT. Use `table_create` when you want to pre-declare a schema.

### dataclawe_table_copy — Copy a table

```json
{
  "db_name": "shop",
  "table_name": "orders",
  "new_table_name": "orders_backup"
}
```

### dataclawe_table_rename — Rename a table

```json
{
  "db_name": "shop",
  "table_name": "orders",
  "new_table_name": "sales_orders"
}
```

### dataclawe_table_drop — Drop a table (soft delete)

```json
{
  "db_name": "shop",
  "table_name": "old_orders"
}
```

> **Warning:** This soft-deletes ALL records in the table and the table schema definition.

### dataclawe_db_create — Create a database

```json
{
  "db_name": "shop"
}
```

> **Note:** Databases are also auto-created on first INSERT. Use `db_create` to explicitly register a namespace.

### dataclawe_db_list — List databases

```json
{}
```

Response:
```json
{"databases": ["shop", "crm", "analytics"], "count": 3}
```

### dataclawe_db_rename — Rename a database

```json
{
  "db_name": "shop",
  "new_db_name": "store"
}
```

### dataclawe_db_drop — Drop a database (permanent)

```json
{
  "db_name": "old_project"
}
```

> **Warning:** This permanently deletes ALL tables and records in the database. This cannot be undone.

---

## Tool Response Format

MCP tool responses follow the standard MCP content structure.

### Success

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"count\":2,\"records\":[{\"customer\":\"Alice\",\"amount\":1500,\"status\":\"paid\",\"_id\":42}]}"
      }
    ]
  }
}
```

The `text` field contains the tool response JSON as a string. Parse it to access the result data.

### Error

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32000,
    "message": "db_name and table_name are required"
  }
}
```

---

## MCP Host Configuration

### Claude Desktop (`claude_desktop_config.json`)

```json
{
  "mcpServers": {
    "dataclawe": {
      "type": "sse",
      "url": "https://mcp.dataclawe.com/sse"
    }
  }
}
```

### Cursor (`.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "dataclawe": {
      "type": "sse",
      "url": "https://mcp.dataclawe.com/sse"
    }
  }
}
```

---

## Recommended Workflow for AI Agents

When exploring a DataClawe database, follow this order:

1. **`dataclawe_db_list`** — List available databases
2. **`dataclawe_tables`** — List tables in the target database
3. **`dataclawe_schema`** — Inspect the structure of a specific table
4. **`dataclawe_stats`** — Check overall record counts
5. **`dataclawe_query`** / **`dataclawe_count`** — Search and retrieve data
6. **`dataclawe_insert`** / **`dataclawe_update`** / **`dataclawe_delete`** — Modify data as needed

Always check the table structure (`dataclawe_schema`) before writing data.

---

## Notes

- Access keys are issued **per domain** — one key per domain.
- `dataclawe_delete` and `dataclawe_table_drop` are always **soft deletes** (status=9). Records are preserved for audit.
- `dataclawe_db_drop` is a **permanent physical delete**. Use with extreme caution.
- Read-only tools (`query`, `count`, `schema`, `tables`, `stats`, `timeline`, `inspect`, `db_list`) are safe for AI agents to call freely.
- The MCP server enforces tenant isolation. Agents cannot access data outside their authorized domain.
- Schema evolves automatically — new fields added via INSERT are auto-registered.

---

## Related Documents

- [README.md](README.md) — DCL command specification
- [README.api.md](README.api.md) — REST API integration guide
- [dataclawe.com/early](https://dataclawe.com/early) — Early Tester Program ($20)

---

*DCL MCP Integration Guide — v2.0 — 2026 — DataClawe Project*
