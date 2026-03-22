# DCL — MCP 連携ガイド

> **ステータス：仕様のみ。MCPエンドポイントは開発中です。**
> このドキュメントはMCPの予定される動作を説明しています。接続はEarly Testerリリース時に利用可能になります。

このガイドは **DataClaweをMCPツールとして統合するAIエージェント開発者** 向けです（Claude・GPT・Cursor・その他MCP対応ホスト）。

フロントエンド・バックエンドからHTTP経由で呼び出す場合は [README.api.ja.md](README.api.ja.md) を参照してください。

DCLコマンド仕様については [README.ja.md](README.ja.md) を参照してください。

---

## 概要

DataClaweは **Model Context Protocol（MCP）** を **JSON-RPC 2.0** で実装しています。

AIエージェントは標準MCPの `tools/call` メソッドを使ってDataClaweツールを呼び出します。DCLペイロードは `arguments` フィールドに含めます。REST APIのペイロードと同一です。

```
AIエージェント
  → JSON-RPC 2.0（tools/call）
    → DataClawe MCPサーバー
      → DCLエンジン
        → MySQL / PostgreSQL
```

---

## プロトコル：JSON-RPC 2.0

MCPはトランスポート層としてJSON-RPC 2.0を使用します。すべてのツール呼び出しはこの構造に従います：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "dataclawe",
    "arguments": {
      <DCLペイロードをここに記述>
    }
  }
}
```

`arguments` フィールドには標準DCL JSONを含めます。REST APIにPOSTするものと同一です。

---

## MCPサーバーエンドポイント（予定）

```
wss://mcp.dataclawe.com/v1
```

認証はMCPハンドシェイク時にドメインアクセスキーで行います。

---

## ツール定義

以下の定義でDataClaweをMCPツールとして登録してください：

```json
{
  "name": "dataclawe",
  "description": "DCL JSONコマンドを使ってMySQLまたはPostgreSQLデータベースを操作します。SELECT・INSERT・UPDATE・DELETE・COUNT・EXISTS・SCHEMA・TABLESアクションに対応しています。",
  "inputSchema": {
    "type": "object",
    "properties": {
      "dcl": {
        "type": "string",
        "enum": ["1.0"],
        "description": "DCLプロトコルバージョン。常に '1.0'。"
      },
      "action": {
        "type": "string",
        "enum": ["SELECT", "INSERT", "UPDATE", "DELETE", "COUNT", "EXISTS", "SCHEMA", "TABLES"],
        "description": "実行するデータベース操作。"
      },
      "table": {
        "type": "string",
        "description": "対象テーブル名。TABLESアクション以外はすべて必須。"
      },
      "columns": {
        "type": "array",
        "items": { "type": "string" },
        "description": "返すカラム名の配列。省略するとすべてのカラムを返す。"
      },
      "where": {
        "description": "絞り込み条件。文字列の配列（AND）またはOR/ANDキーを持つオブジェクト。",
        "oneOf": [
          {
            "type": "array",
            "items": { "type": "string" },
            "description": "ANDで結合する条件の配列。例：[\"status = 'active'\", \"age >= 20\"]"
          },
          {
            "type": "object",
            "description": "複雑な条件のためのOR/ANDキーを持つオブジェクト。"
          }
        ]
      },
      "data": {
        "type": "object",
        "description": "INSERTまたはUPDATE操作のレコードデータ。"
      },
      "order": {
        "type": "string",
        "description": "ORDER BY句。例：'created_at DESC'"
      },
      "limit": {
        "type": "integer",
        "description": "返すレコードの最大件数。"
      },
      "offset": {
        "type": "integer",
        "description": "スキップするレコード数。"
      },
      "group": {
        "type": "string",
        "description": "集計クエリのGROUP BYカラム。"
      }
    },
    "required": ["dcl", "action"]
  }
}
```

---

## ツール呼び出し例

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

### DELETE（論理削除）

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

### SCHEMA — テーブル構造を確認する

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

### TABLES — 利用可能なテーブル一覧を取得する

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

## ツールレスポンス形式

MCPツールのレスポンスは標準MCPのcontent構造に従います。

### 成功

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

`text` フィールドにはDCLレスポンスJSONが文字列として含まれます。`status`・`count`・`data`・`meta` にアクセスするにはパースしてください。

### エラー

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

## MCPホストの設定

### Claude Desktop（claude_desktop_config.json）

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

### Cursor（.cursor/mcp.json）

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

> **注意：** `dataclawe-mcp` はMCPサーバーのバイナリです。インストール手順はEarly Testerリリース時に公開されます。

---

## AIエージェントへの推奨プロンプト

DataClaweをAIエージェントで使用する場合、システムプロンプトに以下のコンテキストを含めることを推奨します：

```
DataClaweデータベースツールが利用可能です。

データベースを探索する際は、以下の順序で始めてください：
1. TABLESアクション — 利用可能なテーブル一覧を取得する
2. SCHEMAアクション — 特定テーブルの構造を確認する
3. その後、SELECT・INSERT・UPDATE・DELETEを必要に応じて使用する

データを書き込む前に必ずテーブル構造を確認してください。
DELETEは論理削除です。レコードはフラグで管理され、物理削除されません。
```

---

## REST APIとの違い

| | REST API | MCP |
|---|---|---|
| トランスポート | HTTPS POST | JSON-RPC 2.0（WebSocket） |
| 呼び出し元 | フロントエンド / バックエンドコード | AIエージェント |
| 認証 | ヘッダーのBearerトークン | MCPハンドシェイク時のアクセスキー |
| DCLペイロード | リクエストボディに直接 | `params.arguments` の中 |
| レスポンス | DCL JSONを直接返す | MCPコンテンツの中にDCL JSON |

DCLペイロード自体（`dcl`・`action`・`table`・`where` など）は**両プロトコルで同一**です。

---

## 注意事項

- アクセスキーは**ドメイン単位**で発行されます。1ドメインにつき1キーです。
- DELETEは常に**論理削除**です。DCL経由でレコードが物理削除されることはありません。
- SCHEMAとTABLESは読み取り専用の探索アクションです。AIエージェントが自由に呼び出して安全です。
- MCPサーバーはテナント分離を強制します。エージェントは認可されたドメイン外のデータにアクセスできません。

---

## 関連ドキュメント

- [README.ja.md](README.ja.md) — DCLコマンド仕様
- [README.api.ja.md](README.api.ja.md) — REST API連携ガイド
- [dataclawe.com/early](https://dataclawe.com/early) — Early Testerプログラム（$20）

---

*DCL MCP 連携ガイド — v1.0 — 2026 — DataClawe Project*
