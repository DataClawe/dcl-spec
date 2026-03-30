# DCL — MCP 連携ガイド

> **ステータス：実装済み・稼働中**
> MCPエンドポイントは `https://mcp.dataclawe.com/sse` で利用可能です。

---

## DataClawe DCL は AI ネイティブ・データベースアクセスの標準です

DataClawe DCL（DataClawe Command Language）は、AIエージェントがデータベースを操作するための標準仕様として設計されています。

> **「AIエージェントがデータベースを扱うなら、DCLで話しかけてください。」**

- JSON構造でシンプル・読みやすい
- スキーマフリー — カラム事前定義なしでデータを挿入可能
- AIエージェント安全設計 — 論理削除・スキーマ自動進化
- PostgreSQL バックエンド（JSONB ストレージ）
- CC BY 4.0 ライセンス — 実装・拡張・採用自由

このガイドは **DataClaweをMCPツールとして統合するAIエージェント開発者** 向けです（Claude・GPT・Cursor・その他MCP対応ホスト）。

フロントエンド・バックエンドからHTTP経由で呼び出す場合は [README.api.ja.md](README.api.ja.md) を参照してください。

DCLコマンド仕様については [README.ja.md](README.ja.md) を参照してください。

---

## 概要

DataClaweは **Model Context Protocol（MCP）** を **SSE（Server-Sent Events）** トランスポートで実装しています。

MCPサーバーは **18個の個別ツール** を公開しており、それぞれが特定のデータベース操作に対応しています。AIエージェントは標準MCPの `tools/call` メソッドを使ってこれらのツールを呼び出します。

```
AIエージェント
  → JSON-RPC 2.0（SSE）
    → DataClawe MCPサーバー（18ツール）
      → PostgreSQL（JSONB）
```

---

## MCPサーバーエンドポイント

```
https://mcp.dataclawe.com/sse
```

トランスポート：**SSE（Server-Sent Events）**

認証はMCPハンドシェイク時にAPIキーで行います。

---

## データモデル

DataClaweは2階層のネームスペースを使用します：**データベース** → **テーブル** → **レコード**

- **db_name**: 論理データベース名（例：`"shop"`、`"crm"`）
- **table_name**: データベース内のテーブル名（例：`"orders"`、`"customers"`）
- **table_id**: 各レコードの一意な整数ID（クエリ結果では `_id` として返される）
- **data**: スキーマフリーのJSONとして保存されるレコードフィールド（任意のキーを受け入れる）

レコードはPostgreSQLのJSONBとして保存されます。スキーマ定義は不要 — カラムは初回INSERT時に自動検出されます。

### ステータス値

| ステータス | 意味 |
|-----------|------|
| `1` | アクティブ（通常レコード） |
| `8` | テーブルスキーマ宣言行（内部用） |
| `9` | 論理削除済み |

---

## ツール一覧

| ツール | 用途 | 読み取り専用 |
|--------|------|-------------|
| `dataclawe_query` | フィルタ付きでレコードを検索・取得 | ✅ |
| `dataclawe_count` | データ取得なしでマッチするレコード数を取得 | ✅ |
| `dataclawe_schema` | テーブルのカラム定義を取得 | ✅ |
| `dataclawe_tables` | テーブル一覧とレコード数を取得 | ✅ |
| `dataclawe_stats` | テナント全体の統計情報を取得 | ✅ |
| `dataclawe_timeline` | データ変更履歴を取得 | ✅ |
| `dataclawe_inspect` | IDで単一レコードを詳細確認 | ✅ |
| `dataclawe_db_list` | データベース一覧を取得 | ✅ |
| `dataclawe_insert` | 新しいレコードを挿入 | ❌ |
| `dataclawe_update` | レコードのフィールドを更新 | ❌ |
| `dataclawe_delete` | レコードを論理削除 | ❌ |
| `dataclawe_table_create` | 空テーブルを作成 | ❌ |
| `dataclawe_table_copy` | テーブルをコピー | ❌ |
| `dataclawe_table_rename` | テーブル名を変更 | ❌ |
| `dataclawe_table_drop` | テーブルを論理削除 | ❌ |
| `dataclawe_db_create` | データベースを作成 | ❌ |
| `dataclawe_db_rename` | データベース名を変更 | ❌ |
| `dataclawe_db_drop` | データベースを永久削除 | ❌ |

---

## Where句 — フィルタ演算子

`where` パラメータは `dataclawe_query` と `dataclawe_count` で使用するJSONオブジェクトです。

### 単純な等値比較

```json
{"status": "paid"}
```
→ `data->>'status' = 'paid'`

### 複数条件（AND）

```json
{"status": "paid", "customer": "Alice"}
```
→ `data->>'status' = 'paid' AND data->>'customer' = 'Alice'`

### 比較演算子

| 演算子 | 意味 | 例 |
|--------|------|-----|
| `$eq` | 等しい | `{"status": {"$eq": "paid"}}` |
| `$ne` | 等しくない | `{"status": {"$ne": "cancelled"}}` |
| `$gt` | より大きい | `{"amount": {"$gt": 1000}}` |
| `$gte` | 以上 | `{"amount": {"$gte": 500}}` |
| `$lt` | より小さい | `{"amount": {"$lt": 100}}` |
| `$lte` | 以下 | `{"amount": {"$lte": 2000}}` |
| `$in` | 配列内に一致 | `{"product": {"$in": ["Plan A", "Plan B"]}}` |
| `$nin` | 配列内に不一致 | `{"status": {"$nin": ["cancelled", "refunded"]}}` |

### 範囲クエリ（演算子の組み合わせ）

```json
{"amount": {"$gte": 500, "$lte": 2000}}
```
→ amount が 500〜2000 のレコード

---

## ツール呼び出し例

### dataclawe_query — レコードの検索

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

レスポンス例：
```json
{
  "count": 2,
  "records": [
    {"customer": "Alice", "amount": 1500, "status": "paid", "_id": 42, "_created_at": "2026-03-28T10:00:00Z", "_updated_at": "2026-03-28T10:00:00Z"},
    {"customer": "Bob", "amount": 800, "status": "paid", "_id": 41, "_created_at": "2026-03-27T09:00:00Z", "_updated_at": "2026-03-27T09:00:00Z"}
  ]
}
```

**演算子の使用例：**
```json
{
  "db_name": "shop",
  "table_name": "orders",
  "where": {"amount": {"$gte": 1000}, "status": {"$in": ["paid", "shipped"]}},
  "limit": 50
}
```

### dataclawe_insert — レコードの追加

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

レスポンス：
```json
{"success": true, "table_id": 42}
```

> **注意：** テーブルやデータベースが存在しない場合は自動作成されます。スキーマは自動進化します — いつでも新しいフィールドを追加できます。

### dataclawe_update — レコードの更新

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

レスポンス：
```json
{"success": true, "affected": 1}
```

> **注意：** 部分更新（PATCHセマンティクス）です。指定したフィールドのみ上書きされます。`table_id` を見つけるには、まず `dataclawe_query` を使ってレコードを検索します — 各レコードに `_id` フィールドがあります。

### dataclawe_delete — レコードの論理削除

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

レスポンス：
```json
{"success": true, "affected": 1}
```

> **注意：** 論理（ソフト）削除です。レコードは `status=9` にマークされ、クエリ結果から除外されますが、`dataclawe_inspect` や `dataclawe_timeline` で引き続き確認できます。

### dataclawe_count — レコード数の取得

```json
{
  "db_name": "shop",
  "table_name": "orders",
  "where": {"status": "paid"}
}
```

レスポンス：
```json
{"db_name": "shop", "table_name": "orders", "count": 15}
```

### dataclawe_schema — テーブル構造の確認

```json
{
  "db_name": "shop",
  "table_name": "orders"
}
```

レスポンス：
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

### dataclawe_tables — テーブル一覧

```json
{
  "db_name": "shop"
}
```

レスポンス：
```json
{
  "tables": [
    {"table_name": "orders", "record_count": 42},
    {"table_name": "products", "record_count": 15}
  ]
}
```

### dataclawe_stats — 全体統計

```json
{}
```

レスポンス：
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

### dataclawe_timeline — 変更履歴

```json
{
  "db_name": "shop",
  "table_name": "orders",
  "limit": 5
}
```

レスポンス：
```json
{
  "events": [
    {"db_name": "shop", "table_name": "orders", "table_id": "42", "status": "9", "created_at": "2026-03-28T10:00:00Z", "updated_at": "2026-03-30T14:00:00Z"},
    {"db_name": "shop", "table_name": "orders", "table_id": "41", "status": "1", "created_at": "2026-03-27T09:00:00Z", "updated_at": "2026-03-27T09:00:00Z"}
  ]
}
```

### dataclawe_inspect — 単一レコードの詳細確認

```json
{
  "db_name": "shop",
  "table_name": "orders",
  "table_id": 42
}
```

レスポンス：
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

### dataclawe_table_create — テーブルの作成

```json
{
  "db_name": "shop",
  "table_name": "products",
  "schema": {"name": "text", "price": "integer", "in_stock": "boolean"}
}
```

> **注意：** テーブルは初回INSERT時にも自動作成されます。スキーマを事前定義したい場合に使用します。

### dataclawe_table_copy — テーブルのコピー

```json
{
  "db_name": "shop",
  "table_name": "orders",
  "new_table_name": "orders_backup"
}
```

### dataclawe_table_rename — テーブル名の変更

```json
{
  "db_name": "shop",
  "table_name": "orders",
  "new_table_name": "sales_orders"
}
```

### dataclawe_table_drop — テーブルの論理削除

```json
{
  "db_name": "shop",
  "table_name": "old_orders"
}
```

> **警告：** テーブル内の全レコードとスキーマ定義が論理削除されます。

### dataclawe_db_create — データベースの作成

```json
{
  "db_name": "shop"
}
```

> **注意：** データベースは初回INSERT時にも自動作成されます。明示的にネームスペースを登録したい場合に使用します。

### dataclawe_db_list — データベース一覧

```json
{}
```

レスポンス：
```json
{"databases": ["shop", "crm", "analytics"], "count": 3}
```

### dataclawe_db_rename — データベース名の変更

```json
{
  "db_name": "shop",
  "new_db_name": "store"
}
```

### dataclawe_db_drop — データベースの永久削除

```json
{
  "db_name": "old_project"
}
```

> **警告：** 指定データベースの全テーブル・全レコードが物理削除されます。この操作は取り消せません。

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
        "text": "{\"count\":2,\"records\":[{\"customer\":\"Alice\",\"amount\":1500,\"status\":\"paid\",\"_id\":42}]}"
      }
    ]
  }
}
```

`text` フィールドにツールレスポンスのJSONが文字列として含まれます。パースして結果データにアクセスしてください。

### エラー

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

## MCPホストの設定

### Claude Desktop（claude_desktop_config.json）

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

### Cursor（.cursor/mcp.json）

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

## AIエージェントの推奨ワークフロー

DataClaweのデータベースを操作する際は、以下の順序で始めてください：

1. **`dataclawe_db_list`** — 利用可能なDB一覧を取得
2. **`dataclawe_tables`** — 対象DBのテーブル一覧を取得
3. **`dataclawe_schema`** — テーブル構造を確認
4. **`dataclawe_stats`** — 全体のレコード数を確認
5. **`dataclawe_query`** / **`dataclawe_count`** — データの検索・取得
6. **`dataclawe_insert`** / **`dataclawe_update`** / **`dataclawe_delete`** — 必要に応じてデータを操作

データを書き込む前に必ずテーブル構造（`dataclawe_schema`）を確認してください。

---

## 注意事項

- アクセスキーは**ドメイン単位**で発行されます。1ドメインにつき1キーです。
- `dataclawe_delete` と `dataclawe_table_drop` は常に**論理削除**（status=9）です。データは監査用に保持されます。
- `dataclawe_db_drop` は**永久物理削除**です。使用には十分注意してください。
- 読み取り専用ツール（`query`・`count`・`schema`・`tables`・`stats`・`timeline`・`inspect`・`db_list`）はAIエージェントが自由に呼び出して安全です。
- MCPサーバーはテナント分離を強制します。エージェントは認可されたドメイン外のデータにアクセスできません。
- スキーマは自動進化します — INSERTで追加された新フィールドは自動的に登録されます。

---

## 関連ドキュメント

- [README.ja.md](README.ja.md) — DCLコマンド仕様
- [README.api.ja.md](README.api.ja.md) — REST API連携ガイド
- [dataclawe.com/early](https://dataclawe.com/early) — Early Testerプログラム（$20）

---

*DCL MCP 連携ガイド — v2.0 — 2026 — DataClawe Project*
