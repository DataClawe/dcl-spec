# DCL — MCP Integration Guide

> **Status: Specification only. MCP endpoint under development.**
> This document describes the intended MCP behavior. Connection will be available in the Early Tester release.

This guide is for **AI agent developers integrating DataClawe as an MCP tool** (Claude, GPT, Cursor, and any MCP-compatible host).

If you are calling DataClawe from frontend or backend code via HTTP, see [README.api.md](README.api.md) instead.

For the DCL command specification, see [README.md](README.md).

---

## Overview

DataClawe implements the **Model Context Protocol (MCP)** using **JSON-RPC 2.0**.

AI agents call DataClawe tools using the standard MCP `tools/call` method. The DCL payload is passed inside the `arguments` field — identical to the REST API payload.

```
AI Agent
  → JSON-RPC 2.0 (tools/call)
    → DataClawe MCP Server
      → DCL Engine
        → MySQL / PostgreSQL
```

---

## Protocol: JSON-RPC 2.0

MCP uses JSON-RPC 2.0 as its transport layer. Every tool call follows this structure:

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

The `arguments` field contains the standard DCL JSON — identical to what you would POST to the REST API.

---

## MCP Server Endpoint (planned)

```
wss://mcp.dataclawe.com/v1
```

Authentication is performed during the MCP handshake using your domain access key.

---

## Tool Definition

Register DataClawe as an MCP tool using the following definition:

```json
{
  "name": "dataclawe",
  "description": "Query and manipulate MySQL or PostgreSQL databases using DCL JSON commands. Supports SELECT, INSERT, UPDATE, DELETE, COUNT, EXISTS, SCHEMA, and TABLES actions.",
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
        "enum": ["SELECT", "INSERT", "UPDATE", "DELETE", "COUNT", "EXISTS", "SCHEMA", "TABLES"],
        "description": "Database operation to perform."
      },
      "table": {
        "type": "string",
        "description": "Target table name. Required for all actions except TABLES."
      },
      "columns": {
        "type": "array",
        "items": { "type": "string" },
        "description": "Column names to return. Omit to return all columns."
      },
      "where": {
        "description": "Filter conditions. Array of strings (AND) or object with OR/AND keys.",
        "oneOf": [
          {
            "type": "array",
            "items": { "type": "string" },
            "description": "Array of conditions joined by AND. Example: [\"status = 'active'\", \"age >= 20\"]"
          },
          {
            "type": "object",
            "description": "Object with OR or AND key for complex conditions."
          }
        ]
      },
      "data": {
        "type": "object",
        "description": "Record data for INSERT or UPDATE operations."
      },
      "order": {
        "type": "string",
        "description": "ORDER BY clause. Example: 'created_at DESC'"
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
      "where": [
        "status = 'active'",
        "age >= 20"
      ],
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
      "data": {
        "status": "inactive"
      },
      "where": [
        "id = 123"
      ]
    }
  }
}
```

### DELETE (logical)

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
      "where": [
        "id = 123"
      ]
    }
  }
}
```

### COUNT

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
      "where": [
        "status = 'pending'"
      ]
    }
  }
}
```

### SCHEMA — discover table structure

```json
{
  "jsonrpc": "2.0",
  "id": 6,
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

### TABLES — list all available tables

```json
{
  "jsonrpc": "2.0",
  "id": 7,
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

### Claude Desktop (claude_desktop_config.json)

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

### Cursor (.cursor/mcp.json)

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

> **Note:** `dataclawe-mcp` is the MCP server binary. Installation instructions will be provided at Early Tester launch.

---

## Recommended Agent Prompt

When using DataClawe with an AI agent, include this context in your system prompt:

```
You have access to a DataClawe database tool.

To explore the database, start with:
1. TABLES action — to list all available tables
2. SCHEMA action — to understand the structure of a specific table
3. Then use SELECT, INSERT, UPDATE, DELETE as needed

Always check table structure before writing data.
DELETE is a logical delete — records are flagged, not removed.
```

---

## Key Differences from REST API

| | REST API | MCP |
|---|---|---|
| Transport | HTTPS POST | JSON-RPC 2.0 (WebSocket) |
| Called by | Frontend / Backend code | AI agents |
| Auth | Bearer token in header | Access key in MCP handshake |
| DCL payload | Request body directly | Inside `params.arguments` |
| Response | DCL JSON directly | DCL JSON inside MCP content |

The DCL payload itself (`dcl`, `action`, `table`, `where`, etc.) is **identical** in both protocols.

---

## Notes

- Access keys are **domain-scoped**. One key per domain (tenant).
- DELETE is always a **logical delete**. Records are never physically removed.
- SCHEMA and TABLES are read-only discovery actions — safe for AI agents to call freely.
- The MCP server enforces tenant isolation. An agent cannot access data outside its authorized domain.

---

## See Also

- [README.md](README.md) — DCL command specification
- [README.api.md](README.api.md) — REST API integration guide
- [dataclawe.com/early](https://dataclawe.com/early) — Early Tester program ($20)

---

*DCL MCP Integration Guide — v1.0 — 2026 — DataClawe Project*
