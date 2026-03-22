# DCL — REST API Guide

> **Status: Specification only. Endpoint under development.**
> This document describes the intended API behavior. Connection will be available in the Early Tester release.

This guide is for **frontend engineers, backend engineers, and anyone calling DataClawe via HTTP**.

If you are integrating via AI agent (Claude, GPT, Cursor), see [README.mcp.md](README.mcp.md) instead.

For the DCL command specification, see [README.md](README.md).

---

## Overview

DataClawe accepts DCL JSON commands over standard HTTPS POST requests.

```
POST https://api.dataclawe.com/v1/dcl
Content-Type: application/json
Authorization: Bearer <your-access-key>

{ DCL JSON payload }
```

No special SDK required. Any HTTP client works.

---

## Authentication

Every request requires a Bearer token in the Authorization header.

```
Authorization: Bearer dcl_kimura_clipjp_live_a1b2c3d4e5f6
```

Access keys are issued per domain (tenant) from your DataClawe console.

**Access key format:**

```
dcl_{username}_{domain}_{env}_{random}

Examples:
dcl_kimura_clipjp_live_a1b2c3d4e5f6      (production)
dcl_kimura_visitpass_dev_x9y8z7w6v5u4    (development)
```

---

## Request Format

```
POST https://api.dataclawe.com/v1/dcl
Content-Type: application/json
Authorization: Bearer dcl_kimura_clipjp_live_xxxxxxxxxxxx

{
  "dcl": "1.0",
  "action": "SELECT",
  "table": "users",
  "where": ["status = 'active'"],
  "limit": 10
}
```

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
  "code": "PERMISSION_DENIED",
  "message": "Invalid access key"
}
```

---

## HTTP Status Codes

| HTTP Status | Meaning                                      |
|-------------|----------------------------------------------|
| `200`       | Request processed successfully               |
| `400`       | Invalid DCL JSON (malformed request)         |
| `401`       | Missing or invalid Authorization header      |
| `404`       | Table not found                              |
| `429`       | Rate limit exceeded                          |
| `500`       | Internal server error                        |

Note: DCL-level errors (e.g. `TABLE_NOT_FOUND`) return HTTP `200` with `"status": "ERROR"` in the body.

---

## Code Examples

### JavaScript (fetch)

```javascript
const response = await fetch('https://api.dataclawe.com/v1/dcl', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer dcl_kimura_clipjp_live_xxxxxxxxxxxx'
  },
  body: JSON.stringify({
    dcl: '1.0',
    action: 'SELECT',
    table: 'users',
    where: ["status = 'active'", "age >= 20"],
    order: 'created_at DESC',
    limit: 10
  })
});

const data = await response.json();
console.log(data.data); // array of records
```

### JavaScript (axios)

```javascript
const { data } = await axios.post(
  'https://api.dataclawe.com/v1/dcl',
  {
    dcl: '1.0',
    action: 'SELECT',
    table: 'users',
    where: ["status = 'active'"],
    limit: 10
  },
  {
    headers: {
      'Authorization': 'Bearer dcl_kimura_clipjp_live_xxxxxxxxxxxx'
    }
  }
);
```

### PHP (curl)

```php
$payload = json_encode([
    'dcl'    => '1.0',
    'action' => 'SELECT',
    'table'  => 'users',
    'where'  => ["status = 'active'", "age >= 20"],
    'order'  => 'created_at DESC',
    'limit'  => 10,
]);

$ch = curl_init('https://api.dataclawe.com/v1/dcl');
curl_setopt_array($ch, [
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => [
        'Content-Type: application/json',
        'Authorization: Bearer dcl_kimura_clipjp_live_xxxxxxxxxxxx',
    ],
    CURLOPT_POSTFIELDS     => $payload,
    CURLOPT_RETURNTRANSFER => true,
]);

$response = json_decode(curl_exec($ch), true);
curl_close($ch);

if ($response['status'] === 'OK') {
    $records = $response['data'];
}
```

### PHP (file_get_contents)

```php
$context = stream_context_create([
    'http' => [
        'method'  => 'POST',
        'header'  => implode("\r\n", [
            'Content-Type: application/json',
            'Authorization: Bearer dcl_kimura_clipjp_live_xxxxxxxxxxxx',
        ]),
        'content' => json_encode([
            'dcl'    => '1.0',
            'action' => 'SELECT',
            'table'  => 'users',
            'where'  => ["status = 'active'"],
            'limit'  => 10,
        ]),
    ],
]);

$response = json_decode(
    file_get_contents('https://api.dataclawe.com/v1/dcl', false, $context),
    true
);
```

### Python (requests)

```python
import requests

response = requests.post(
    'https://api.dataclawe.com/v1/dcl',
    headers={
        'Authorization': 'Bearer dcl_kimura_clipjp_live_xxxxxxxxxxxx'
    },
    json={
        'dcl': '1.0',
        'action': 'SELECT',
        'table': 'users',
        'where': ["status = 'active'", "age >= 20"],
        'order': 'created_at DESC',
        'limit': 10
    }
)

data = response.json()
```

### curl (command line)

```bash
curl -X POST https://api.dataclawe.com/v1/dcl \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dcl_kimura_clipjp_live_xxxxxxxxxxxx" \
  -d '{
    "dcl": "1.0",
    "action": "SELECT",
    "table": "users",
    "where": ["status = '\''active'\''"],
    "limit": 10
  }'
```

---

## Common Patterns

### INSERT and get the new record ID

```javascript
const response = await fetch('https://api.dataclawe.com/v1/dcl', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer dcl_kimura_clipjp_live_xxxxxxxxxxxx'
  },
  body: JSON.stringify({
    dcl: '1.0',
    action: 'INSERT',
    table: 'users',
    data: {
      name: 'kimura',
      email: 'kimura@example.com',
      status: 'active'
    }
  })
});

const result = await response.json();
// result.data[0].table_id — newly created record ID
```

### Check existence before INSERT

```javascript
// Step 1: check
const exists = await dcl({
  action: 'EXISTS',
  table: 'users',
  where: ["email = 'kimura@example.com'"]
});

// Step 2: insert only if not found
if (!exists.data[0].exists) {
  await dcl({
    action: 'INSERT',
    table: 'users',
    data: { email: 'kimura@example.com', status: 'active' }
  });
}
```

### Minimal wrapper function (JavaScript)

```javascript
const DCL_KEY = 'dcl_kimura_clipjp_live_xxxxxxxxxxxx';

async function dcl(command) {
  const res = await fetch('https://api.dataclawe.com/v1/dcl', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${DCL_KEY}`
    },
    body: JSON.stringify({ dcl: '1.0', ...command })
  });
  const data = await res.json();
  if (data.status !== 'OK') throw new Error(data.message);
  return data;
}

// Usage
const users = await dcl({
  action: 'SELECT',
  table: 'users',
  where: ["status = 'active'"],
  limit: 20
});
```

---

## Rate Limits

| Plan          | Requests per minute |
|---------------|---------------------|
| Early Tester  | 60 rpm per domain   |
| Pro           | 600 rpm per domain  |
| Enterprise    | Custom              |

Rate limit responses return HTTP `429` with a `Retry-After` header.

---

## Notes

- Access keys are **domain-scoped**. One key per domain (tenant).
- Keep your access key private. Do not expose it in frontend code directly. Use a backend proxy or environment variable.
- DELETE is always a **logical delete** (status flag). Records are never physically removed via DCL.
- All timestamps are returned in UTC ISO 8601 format.

---

## See Also

- [README.md](README.md) — DCL command specification
- [README.mcp.md](README.mcp.md) — MCP integration guide
- [dataclawe.com/early](https://dataclawe.com/early) — Early Tester program ($20)

---

*DCL REST API Guide — v1.0 — 2026 — DataClawe Project*
