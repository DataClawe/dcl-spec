# DCL — DataClawe Command Language

**A JSON-based database command standard for AI agents, frontend engineers, and anyone who finds SQL too complex.**

> "If you understand MySQL and MSX-BASIC, you already understand DCL."

---

## What is DCL?

DCL (DataClawe Command Language) is an open standard for communicating with databases using simple JSON.

It is designed for:

- **AI agents** (Claude, GPT, Cursor, and others) to safely read and write structured data
- **Frontend engineers** who need database access without writing SQL
- **MCP (Model Context Protocol)** tool integration
- **Any application** connecting to MySQL or PostgreSQL — legacy or new

DCL translates into native SQL internally. Your existing database does not change. Your existing application does not change.

---

## Design Philosophy

DCL follows three rules:

**1. If you can read it, you understand it.**
No cryptic operators. No framework-specific syntax. No `:` chaining.

**2. MySQL words. JSON structure.**
Actions like `SELECT`, `INSERT`, `UPDATE`, `DELETE` are exactly what you expect. WHERE conditions are written the way MySQL engineers already write them.

**3. MSX-BASIC level simplicity.**
If a condition like `age >= 20` needs an explanation, the design has failed.

**4. Simple by default. Powerful when needed.**
Standard covers most real-world needs. Advanced covers the rest. You never pay the complexity cost until you need it.

---

## Quick Start

### Fetch data

```json
{
  "dcl": "1.0",
  "action": "SELECT",
  "table": "users",
  "columns": ["id", "name", "email"],
  "where": [
    "status = 'active'",
    "age >= 20"
  ],
  "order": "created_at DESC",
  "limit": 10,
  "offset": 0
}
```

### Insert a record

```json
{
  "dcl": "1.0",
  "action": "INSERT",
  "table": "users",
  "data": {
    "name": "kimura",
    "email": "kimura@example.com",
    "status": "active"
  }
}
```

### Update records

```json
{
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
```

### Delete a record (logical delete)

```json
{
  "dcl": "1.0",
  "action": "DELETE",
  "table": "users",
  "where": [
    "id = 123"
  ]
}
```

---

## Actions

| Action | Description | Read-only |
|--------|-------------|-----------|
| `SELECT` | Fetch one or more records | ✅ |
| `COUNT` | Count matching records | ✅ |
| `EXISTS` | Check if a record exists | ✅ |
| `SCHEMA` | Get column definitions for a table | ✅ |
| `TABLES` | List all available tables | ✅ |
| `STATS` | Get record count, last updated, and storage size | ✅ |
| `TIMELINE` | Get chronological change history of a record | ✅ |
| `INSERT` | Create a new record | ❌ |
| `UPDATE` | Update existing records | ❌ |
| `DELETE` | Logical delete (status flag) | ❌ |
| `TABLE_CREATE` | Create a new table (empty declaration) | ❌ |
| `TABLE_COPY` | Copy a table | ❌ |
| `TABLE_RENAME` | Rename a table | ❌ |
| `TABLE_DROP` | Logical delete a table | ❌ |

---

## WHERE Conditions

WHERE is an array of condition strings. Multiple conditions are AND by default.

### Basic comparisons

```json
"where": [
  "status = 'active'",
  "age >= 20",
  "age <= 60",
  "status != 'deleted'"
]
```

### OR conditions

```json
"where": {
  "OR": [
    "status = 'active'",
    "status = 'pending'"
  ]
}
```

### AND + OR combined

```json
"where": {
  "AND": [
    "age >= 20",
    {
      "OR": [
        "status = 'active'",
        "status = 'pending'"
      ]
    }
  ]
}
```

### LIKE

```json
"where": [
  "name LIKE 'kimura%'"
]
```

### IN

```json
"where": [
  "status IN ('active', 'pending')"
]
```

### BETWEEN

```json
"where": [
  "age BETWEEN 20 AND 60"
]
```

### NULL checks

```json
"where": [
  "deleted_at IS NULL"
]
```

```json
"where": [
  "deleted_at IS NOT NULL"
]
```

---

## Aggregation

```json
{
  "dcl": "1.0",
  "action": "SELECT",
  "table": "orders",
  "columns": [
    "SUM(amount) AS total",
    "AVG(amount) AS average",
    "COUNT(*) AS count"
  ],
  "where": [
    "status = 'paid'"
  ],
  "group": "customer_id"
}
```

---

## COUNT

```json
{
  "dcl": "1.0",
  "action": "COUNT",
  "table": "users",
  "where": [
    "status = 'active'"
  ]
}
```

---

## EXISTS

```json
{
  "dcl": "1.0",
  "action": "EXISTS",
  "table": "users",
  "where": [
    "email = 'kimura@example.com'"
  ]
}
```

---

## SCHEMA

```json
{
  "dcl": "1.0",
  "action": "SCHEMA",
  "table": "users"
}
```

---

## TABLES

```json
{
  "dcl": "1.0",
  "action": "TABLES"
}
```

---

## TABLE_CREATE — Create a table

Declares a new empty table. Internally creates a single `status=8` schema declaration row.

```json
{
  "dcl": "1.0",
  "action": "TABLE_CREATE",
  "table": "users"
}
```

With optional schema definition:

```json
{
  "dcl": "1.0",
  "action": "TABLE_CREATE",
  "table": "users",
  "schema": {
    "name":  "text",
    "age":   "integer",
    "email": "text"
  }
}
```

> **Without schema:** Created with `schema: null`. Column types are auto-registered as `text` on first INSERT.

---

## status=8 Schema Declaration Row

DataClawe maintains one `status=8` declaration row per table.

| status value | Purpose |
|---|---|
| `status=1` | Normal record (active) |
| `status=8` | Table declaration + schema cache |
| `status=9` | Logically deleted |

**Example status=8 row content:**

```json
{
  "schema": {
    "name":  "text",
    "age":   "text",
    "email": "text"
  }
}
```

**Auto Schema Evolution:**

Writing to a non-existent table via INSERT auto-creates the status=8 row.
New columns are automatically appended to the status=8 schema on each INSERT (type defaults to `text`).

```
First INSERT: {"name": "kimura", "age": 25}
  → Auto-create status=8: {"name":"text", "age":"text"}
  → INSERT the record

Second INSERT: {"name": "suzuki", "phone": "090-xxxx"}
  → "phone" is new → auto-append to status=8 schema
  → schema: {"name":"text", "age":"text", "phone":"text"}
```

When connecting to legacy MySQL/PostgreSQL, actual column types (`VARCHAR(255)`, `INT`, etc.) are recorded in the status=8 row.

---

## STATS — Table statistics

Returns record count, last updated timestamp, and storage size. Useful for AI agents to assess table state before operating.

```json
{
  "dcl": "1.0",
  "action": "STATS",
  "table": "users"
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

---

## TIMELINE — Record change history

Returns the change history (created, updated, logical delete) of a record in chronological order. Useful for AI agents tracking data evolution.

```json
{
  "dcl": "1.0",
  "action": "TIMELINE",
  "table": "users",
  "where": [
    "id = 123"
  ]
}
```

---

## TABLE_COPY — Copy a table

Copies a table under a new name. Used for backups, migrations, and testing.

```json
{
  "dcl": "1.0",
  "action": "TABLE_COPY",
  "table": "users",
  "target_table": "users_backup_20260328"
}
```

---

## TABLE_RENAME — Rename a table

Renames a table.

```json
{
  "dcl": "1.0",
  "action": "TABLE_RENAME",
  "table": "users_old",
  "new_name": "users_archived"
}
```

---

## TABLE_DROP — Drop a table (logical delete)

Logically deletes a table. Data is not physically removed immediately.

```json
{
  "dcl": "1.0",
  "action": "TABLE_DROP",
  "table": "users_temp"
}
```

> **Note:** TABLE_DROP is a logical delete — not equivalent to SQL `DROP TABLE`. Data is not immediately destroyed.

---

## Response Format

### Success

```json
{
  "dcl": "1.0",
  "status": "OK",
  "count": 42,
  "data": [
    { "id": 1, "name": "kimura", "email": "kimura@example.com" }
  ],
  "meta": {
    "table": "users",
    "elapsed_ms": 12
  }
}
```

### Error

```json
{
  "dcl": "1.0",
  "status": "ERROR",
  "code": "TABLE_NOT_FOUND",
  "message": "Table 'users' does not exist"
}
```

### Error codes

| Code | Description |
|------|-------------|
| `TABLE_NOT_FOUND` | Specified table does not exist |
| `COLUMN_NOT_FOUND` | Specified column does not exist |
| `INVALID_ACTION` | Unknown action specified |
| `INVALID_WHERE` | WHERE condition could not be parsed |
| `PERMISSION_DENIED` | Tenant does not have access |
| `CONNECTION_ERROR` | Database connection failed |

---

## Integration

DCL works over two protocols. The DCL command itself is identical in both cases.

| Protocol | Used by | Guide |
|----------|---------|-------|
| REST API (HTTPS POST) | Frontend, Backend, any HTTP client | See [README.api.md](README.api.md) |
| MCP (JSON-RPC 2.0 / SSE) | AI agents (Claude, GPT, Cursor) | See [README.mcp.md](README.mcp.md) |

### DCL payload (identical in both protocols)

```json
{
  "dcl": "1.0",
  "action": "SELECT",
  "table": "orders",
  "where": [
    "status = 'pending'",
    "created_at >= '2026-01-01'"
  ],
  "order": "created_at DESC",
  "limit": 100
}
```

This DCL JSON is the same regardless of whether you call via REST API or MCP. Only the outer protocol layer differs.

---

## Supported Databases

| Database | Status |
|----------|--------|
| MySQL 5.x / 8.x | ✅ Supported |
| PostgreSQL 9–18 | ✅ Supported |
| Others | 📋 Planned |

DCL normalizes differences between MySQL and PostgreSQL. You write one DCL command. DataClawe handles the rest.

---

## Complexity Layers

DCL is designed in two layers. You choose the layer you need.

---

### DCL Standard — this specification

For AI agents, frontend engineers, and anyone doing straightforward data operations.

- No JOIN. No subquery.
- Readable by anyone who knows MySQL basics.
- Covers the vast majority of real-world CRUD needs.

If Standard covers your needs, you never have to go further.

---

### DCL Advanced — coming in v1.1

For backend engineers who need more expressive power.

Same JSON structure. Same DataClawe engine. More capability when you need it.

**JOIN**

```json
{
  "dcl": "1.0",
  "action": "SELECT",
  "table": "users",
  "columns": ["users.id", "users.name", "orders.amount"],
  "join": [
    {
      "table": "orders",
      "on": "users.id = orders.user_id",
      "type": "LEFT"
    }
  ],
  "where": [
    "users.status = 'active'"
  ],
  "order": "orders.amount DESC",
  "limit": 20
}
```

**Subquery**

```json
{
  "dcl": "1.0",
  "action": "SELECT",
  "table": "users",
  "where": [
    "id IN (SELECT user_id FROM orders WHERE status = 'paid')"
  ]
}
```

**Multiple JOINs**

```json
{
  "dcl": "1.0",
  "action": "SELECT",
  "table": "orders",
  "columns": [
    "orders.id",
    "users.name AS customer",
    "products.name AS product",
    "orders.amount"
  ],
  "join": [
    {
      "table": "users",
      "on": "orders.user_id = users.id",
      "type": "INNER"
    },
    {
      "table": "products",
      "on": "orders.product_id = products.id",
      "type": "LEFT"
    }
  ],
  "where": [
    "orders.status = 'paid'",
    "orders.created_at >= '2026-01-01'"
  ],
  "order": "orders.created_at DESC"
}
```

**JOIN types supported in Advanced**

| Type | Description |
|------|-------------|
| `INNER` | Records matching in both tables |
| `LEFT` | All left, matching right |
| `RIGHT` | All right, matching left |

---

### Why two layers?

```
SQL solved everything in one spec.
That is why SQL is still hard after 30 years.

DCL Standard is for everyone.
DCL Advanced is for when you need more.
You always start simple. You go deeper only when you must.
```

The same engine handles both. The same JSON structure. No new syntax to learn when you move from Standard to Advanced — just new keys.

---

### What remains intentionally unsupported

The following are out of scope in both Standard and Advanced, to keep DCL safe and predictable for AI agents:

- Stored procedures
- Schema creation or `ALTER TABLE`
- Raw SQL passthrough
- Physical schema destruction (immediate `DROP TABLE` equivalent)

> **Note:** `TABLE_DROP` in DCL is a **logical delete** — distinct from SQL `DROP TABLE`. It does not immediately destroy data.

DCL is a data operation language, not a schema management language.

---

## Versioning

The `"dcl": "1.0"` field is required in every request and response.

Future versions will remain backward compatible. A DCL 1.0 request will always work against a DCL 2.0 server.

---

## Contributing

DCL is an open specification. Feedback, proposals, and pull requests are welcome.

- Open an issue to propose a new action or operator
- Open a pull request to improve documentation or examples
- All contributions must follow the design philosophy: **readable, MySQL-familiar, BASIC-level simplicity**

---

## Roadmap

**v1.0 — DCL Standard**
- [ ] Spec finalization
- [ ] JSON Schema for validation (`dcl.schema.json`)
- [ ] Reference implementation in Go (DataClawe Engine)
- [ ] MCP server reference implementation
- [ ] MySQL wire protocol support
- [ ] Multi-tenant isolation specification

**v1.1 — DCL Advanced**
- [ ] JOIN specification (INNER / LEFT / RIGHT)
- [ ] Subquery support
- [ ] Nested aggregation

**SDKs**
- [ ] JavaScript / TypeScript
- [ ] PHP
- [ ] Python
- [ ] Go

---

## Pricing

DataClawe uses a tiered pricing model that scales from individual developers to enterprise.

**Initial registration fee: $20 (one-time, all plans)**

| Plan | Monthly | Records | Sessions | SLA |
|------|---------|---------|----------|-----|
| **Free** | $0 | 2,000 | 100/day | No |
| **Personal** | $20 | up to 20,000 | up to 20,000/month | No |
| **Business** | $20+ | Unlimited | Unlimited | No |
| **Enterprise** | Contact us | Unlimited | Unlimited | Yes |

**Usage rates (Business plan — overage)**

| Item | Unit price |
|------|-----------|
| Record storage | $0.001 / record / month |
| Session | $0.001 / session |

**Record size limit: 100KB per record.**
This covers IoT sensor data, CRM contacts, medical text records, WordPress posts, and financial transactions.
Images and video files are out of scope — store them in CDN/object storage and keep the URL in DataClawe.

For Enterprise pricing, contact us at [dataclawe.com/enterprise](https://dataclawe.com/enterprise).

---

## Background

DCL is developed as part of the [DataClawe](https://dataclawe.com) project.

DataClawe is a database translation engine that connects legacy MySQL and PostgreSQL systems to AI agents, LLM pipelines, and cloud-native services — without rewriting the original system.

DCL is the command language that makes this connection possible for everyone, not just database engineers.

> Legacy systems should not be destroyed. They should be translated.

---

## License

DCL specification is released under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

You are free to implement, extend, and build upon this specification. Attribution to DataClawe is appreciated.

---

*DCL v1.1 — 2026 — DataClawe Project*
