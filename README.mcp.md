# DCL — MCP Integration Guide

> **Status: Live and operational.**
> The MCP endpoint is available at `https://mcp.dataclawe.com/sse`.

---

## DataClawe DCL is the Standard for AI-Native Database Access

DataClawe DCL (DataClawe Command Language) is designed as the standard specification for AI agents to interact with databases.

> **"If your AI agent needs to talk to a database, speak DCL."**

- Simple, readable JSON structure
- MySQL-familiar actions — no new syntax to learn
- AI-safety by design — destructive operations intentionally excluded
- MySQL and PostgreSQL compatible
- CC BY 4.0 License — free to implement, extend, and adopt

This guide is for **AI agent developers integrating DataClawe as an MCP tool** (Claude, GPT, Cursor, and other MCP-compatible hosts).

For HTTP-based integration from frontend or backend code, see [README.api.md](README.api.md).

For the full DCL command specification, see [README.md](README.md).

---

## Overview

DataClawe implements the **Model Context Protocol (MCP)** over **SSE (Server-Sent Events)** transport.

AI agents invoke DataClawe tools using the standard MCP `tools/call` method. The DCL payload is passed in the `arguments` field — identical to the REST API payload.

```
AI Agent
  → JSON-RPC 2.0 (SSE)
    → DataClawe MCP Server
      → DCL Engine
        → MySQL / PostgreSQL
```

---

## Protocol: JSON-RPC 2.0

MCP uses JSON-RPC 2.0 as the transport layer. All tool calls follow this structure:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      <DCL payload here>
    }
  }
}
```

The `arguments` field contains standard DCL JSON — identical to what you POST to the REST API.

---

## MCP Server Endpoint

```
https://mcp.dataclawe.com/sse
```

Transport: **SSE (Server-Sent Events)**

Authentication is handled during the MCP handshake using your domain access key.

---

## Tool Definition

Register DataClawe as an MCP tool using the following definition:

```json
{
  "name": "dataclawe",
  "description": "Interact with MySQL or PostgreSQL databases using DCL JSON commands. Supports SELECT, INSERT, UPDATE, DELETE, COUNT, EXISTS, SCHEMA, TABLES, STATS, TIMELINE, TABLE_COPY, TABLE_RENAME, and TABLE_DROP actions.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "dcl": {
        "type": "string",
        "enum": ["1.0"],
        "description": "DCL protocol version. Always '1.0'."
      },
      "action": {
        "type": "string",
        "enum": [
          "SELECT", "INSERT", "UPDATE", "DELETE",
          "COUNT", "EXISTS", "SCHEMA", "TABLES",
          "STATS", "TIMELINE",
          "TABLE_COPY", "TABLE_RENAME", "TABLE_DROP"
        ],
        "description": "The database operation to perform."
      },
      "table": {
        "type": "string",
        "description": "Target table name. Required for all actions except TABLES."
      },
      "columns": {
        "type": "array",
        "items": { "type": "string" },
        "description": "Array of column names to return. Omit to return all columns."
      },
      "where": {
        "description": "Filter conditions. Array of strings (AND) or object with OR/AND keys.",
        "oneOf": [
          {
            "type": "array",
            "items": { "type": "string" },
            "description": "AND-joined conditions. e.g. [\"status = 'active'\", \"age >= 20\"]"
          },
          {
            "type": "object",
            "description": "Object with OR/AND keys for complex conditions."
          }
        ]
      },
      "data": {
        "type": "object",
        "description": "Record data for INSERT or UPDATE operations."
      },
      "order": {
        "type": "string",
        "description": "ORDER BY clause. e.g. 'created_at DESC'"
      },
      "limit": {
        "type": "integer",
        "description": "Maximum number of records to return."
      },
      "offset": {
        "type": "integer",
        "description": "Number of records to skip."
      },
      "group": {
        "type": "string",
        "description": "GROUP BY column for aggregation queries."
      },
      "new_name": {
        "type": "string",
        "description": "New table name for TABLE_RENAME action."
      },
      "target_table": {
        "type": "string",
        "description": "Destination table name for TABLE_COPY action."
      }
    },
    "required": ["dcl", "action"]
  }
}
```

---

## Tool Call Examples

### SELECT

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "SELECT",
      "table": "users",
      "columns": ["id", "name", "email"],
      "where": ["status = 'active'", "age >= 20"],
      "order": "created_at DESC",
      "limit": 10
    }
  }
}
```

### INSERT

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "INSERT",
      "table": "users",
      "data": {
        "name": "kimura",
        "email": "kimura@example.com",
        "status": "active"
      }
    }
  }
}
```

### UPDATE

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "UPDATE",
      "table": "users",
      "data": { "status": "inactive" },
      "where": ["id = 123"]
    }
  }
}
```

### DELETE (soft delete)

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "DELETE",
      "table": "users",
      "where": ["id = 123"]
    }
  }
}
```

### COUNT — Get record count

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "COUNT",
      "table": "orders",
      "where": ["status = 'pending'"]
    }
  }
}
```

### EXISTS — Check record existence

```json
{
  "jsonrpc": "2.0",
  "id": 6,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "EXISTS",
      "table": "users",
      "where": ["email = 'kimura@example.com'"]
    }
  }
}
```

Example response:
```json
{ "dcl": "1.0", "status": "OK", "exists": true, "meta": { "table": "users" } }
```

### SCHEMA — Inspect table structure

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "SCHEMA",
      "table": "users"
    }
  }
}
```

### TABLES — List available tables

```json
{
  "jsonrpc": "2.0",
  "id": 8,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "TABLES"
    }
  }
}
```

### STATS — Table statistics

Returns record count, last updated timestamp, and storage size. Useful for AI agents to assess table state before operating.

```json
{
  "jsonrpc": "2.0",
  "id": 9,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "STATS",
      "table": "users"
    }
  }
}
```

Example response:
```json
{
  "dcl": "1.0",
  "status": "OK",
  "data": {
    "table": "users",
    "record_count": 1024,
    "last_updated": "2026-03-28T10:00:00Z",
    "size_kb": 512
  }
}
```

### TIMELINE — Record change history

Returns the change history (created, updated, deleted) of a record in chronological order. Useful for AI agents tracking data evolution.

```json
{
  "jsonrpc": "2.0",
  "id": 10,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "TIMELINE",
      "table": "users",
      "where": ["id = 123"]
    }
  }
}
```

### TABLE_COPY — Copy a table

```json
{
  "jsonrpc": "2.0",
  "id": 11,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "TABLE_COPY",
      "table": "users",
      "target_table": "users_backup_20260328"
    }
  }
}
```

### TABLE_RENAME — Rename a table

```json
{
  "jsonrpc": "2.0",
  "id": 12,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "TABLE_RENAME",
      "table": "users_old",
      "new_name": "users_archived"
    }
  }
}
```

### TABLE_DROP — Drop a table (soft delete)

```json
{
  "jsonrpc": "2.0",
  "id": 13,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      "dcl": "1.0",
      "action": "TABLE_DROP",
      "table": "users_temp"
    }
  }
}
```

> **Note:** TABLE_DROP is a soft delete. Tables are not physically removed immediately.

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
        "text": "{\"dcl\":\"1.0\",\"status\":\"OK\",\"count\":42,\"data\":[{\"id\":1,\"name\":\"kimura\"}],\"meta\":{\"table\":\"users\",\"elapsed_ms\":12}}"
      }
    ]
  }
}
```

The `text` field contains the DCL response JSON as a string. Parse it to access `status`, `count`, `data`, and `meta`.

### Error

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"dcl\":\"1.0\",\"status\":\"ERROR\",\"code\":\"TABLE_NOT_FOUND\",\"message\":\"Table 'users' does not exist\"}"
      }
    ],
    "isError": true
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
      "command": "dataclawe-mcp",
      "args": [],
      "env": {
        "DATACLAWE_ACCESS_KEY": "dcl_kimura_clipjp_live_xxxxxxxxxxxx"
      }
    }
  }
}
```

### Cursor (`.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "dataclawe": {
      "command": "dataclawe-mcp",
      "env": {
        "DATACLAWE_ACCESS_KEY": "dcl_kimura_clipjp_live_xxxxxxxxxxxx"
      }
    }
  }
}
```

### Claude Code CLI (SSE direct connection)

```json
{
  "mcpServers": {
    "dataclawe": {
      "type": "sse",
      "url": "https://mcp.dataclawe.com/sse",
      "env": {
        "DATACLAWE_ACCESS_KEY": "dcl_kimura_clipjp_live_xxxxxxxxxxxx"
      }
    }
  }
}
```

---

## Recommended System Prompt for AI Agents

When using DataClawe in an AI agent, include the following context in your system prompt:

```
The DataClawe database tool is available.

When exploring a database, follow this order:
1. TABLES action — list available tables
2. SCHEMA action — inspect the structure of a specific table
3. STATS action — check record count and last updated timestamp
4. Then use SELECT, INSERT, UPDATE, DELETE as needed

Always check the table structure before writing data.
DELETE and TABLE_DROP are soft deletes — records are managed by status flags, not physically removed.
Use TIMELINE to inspect the change history of a record.
```

---

## Action Reference

| Action | Purpose | Read-only |
|---|---|---|
| SELECT | Retrieve records | ✅ |
| COUNT | Get record count | ✅ |
| EXISTS | Check record existence | ✅ |
| SCHEMA | Inspect table structure | ✅ |
| TABLES | List available tables | ✅ |
| STATS | Table statistics | ✅ |
| TIMELINE | Record change history | ✅ |
| INSERT | Add records | ❌ |
| UPDATE | Modify records | ❌ |
| DELETE | Soft-delete records | ❌ |
| TABLE_COPY | Copy a table | ❌ |
| TABLE_RENAME | Rename a table | ❌ |
| TABLE_DROP | Soft-delete a table | ❌ |

---

## REST API vs MCP

| | REST API | MCP |
|---|---|---|
| Transport | HTTPS POST | JSON-RPC 2.0 (SSE) |
| Caller | Frontend / backend code | AI agents |
| Auth | Bearer token in header | Access key at MCP handshake |
| DCL payload | Directly in request body | Inside `params.arguments` |
| Response | DCL JSON directly | DCL JSON wrapped in MCP content |

The DCL payload itself (`dcl`, `action`, `table`, `where`, etc.) is **identical across both protocols**.

---

## Notes

- Access keys are issued **per domain** — one key per domain.
- DELETE and TABLE_DROP are always **soft deletes**. No data is physically removed via DCL.
- SCHEMA, TABLES, STATS, and TIMELINE are read-only exploration actions — safe for AI agents to call freely.
- The MCP server enforces tenant isolation. Agents cannot access data outside their authorized domain.

---

## Related Documents

- [README.md](README.md) — DCL command specification
- [README.api.md](README.api.md) — REST API integration guide
- [dataclawe.com/early](https://dataclawe.com/early) — Early Tester Program ($20)

---

*DCL MCP Integration Guide — v1.1 — 2026 — DataClawe Project*
