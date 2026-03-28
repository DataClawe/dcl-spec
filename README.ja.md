# DCL — DataClawe Command Language

**AIエージェント・フロントエンドエンジニア・SQLが難しいと感じるすべての人のための、JSONベースのデータベースコマンド標準仕様。**

> 「MySQLとMSX-BASICを知っていれば、DCLはすでに理解できる。」

---

## DCLとは

DCL（DataClawe Command Language）は、シンプルなJSONを使ってデータベースと通信するためのオープン標準仕様です。

以下を対象として設計されています：

- **AIエージェント**（Claude・GPT・Cursor など）が安全に構造化データを読み書きする
- **フロントエンドエンジニア**がSQLを書かずにデータベースにアクセスする
- **MCP（Model Context Protocol）**ツールとの統合
- MySQLまたはPostgreSQLに接続する**あらゆるアプリケーション**（レガシー・新規問わず）

DCLは内部でネイティブSQLに変換されます。既存のデータベースは変更不要。既存のアプリケーションも変更不要。

---

## 設計思想

DCLは3つのルールに従っています：

**1. 読めば理解できる。**
難解な演算子なし。フレームワーク固有の構文なし。`:`チェーンなし。

**2. MySQLの言葉。JSONの構造。**
`SELECT`・`INSERT`・`UPDATE`・`DELETE` などのアクションは、まさにそのままの意味です。WHERE条件はMySQLエンジニアが書き慣れた書き方そのままです。

**3. MSX-BASICレベルのシンプルさ。**
`age >= 20` という条件に説明が必要なら、設計が失敗しています。

**4. デフォルトはシンプル。必要なときだけ強力に。**
標準仕様は現実の大半のニーズをカバーします。高度な機能は必要になったときだけ。複雑さのコストは、必要になるまで払わなくていい。

---

## クイックスタート

### データを取得する

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

### レコードを追加する

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

### レコードを更新する

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

### レコードを削除する（論理削除）

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

## アクション一覧

| アクション | 説明 | 読み取り専用 |
|------------|------|---|
| `SELECT` | 1件または複数件のレコードを取得 | ✅ |
| `COUNT` | 条件に一致するレコード数を取得 | ✅ |
| `EXISTS` | レコードが存在するか確認 | ✅ |
| `SCHEMA` | テーブルのカラム定義を取得 | ✅ |
| `TABLES` | 利用可能なテーブル一覧を取得 | ✅ |
| `STATS` | テーブルのレコード数・更新日時・容量を取得 | ✅ |
| `TIMELINE` | レコードの変更履歴を時系列で取得 | ✅ |
| `INSERT` | 新しいレコードを作成 | ❌ |
| `UPDATE` | 既存のレコードを更新 | ❌ |
| `DELETE` | レコードを論理削除（ステータスフラグで管理） | ❌ |
| `TABLE_CREATE` | テーブルを新規作成する（空テーブル宣言） | ❌ |
| `TABLE_COPY` | テーブルをコピーする | ❌ |
| `TABLE_RENAME` | テーブル名を変更する | ❌ |
| `TABLE_DROP` | テーブルを論理削除する | ❌ |

---

## WHERE条件

WHEREは条件文字列の配列です。複数の条件はデフォルトでAND結合されます。

### 基本的な比較

```json
"where": [
  "status = 'active'",
  "age >= 20",
  "age <= 60",
  "status != 'deleted'"
]
```

### OR条件

```json
"where": {
  "OR": [
    "status = 'active'",
    "status = 'pending'"
  ]
}
```

### AND + OR の組み合わせ

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

### NULLチェック

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

## 集計

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

## TABLE_CREATE — テーブルの作成

空のテーブルを宣言します。内部的に `status=8` のスキーマ宣言行を1件作成します。

```json
{
  "dcl": "1.0",
  "action": "TABLE_CREATE",
  "table": "users"
}
```

スキーマを事前に定義する場合（オプション）：

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

> **スキーマ未指定の場合：** `schema: null` で作成されます。初回INSERT时に自動的にカラム型が `text` で登録されます。

---

## status=8 スキーマ宣言行の設計

DataClaweは各テーブルに `status=8` の宣言行を1件保持します。

| status値 | 用途 |
|---|---|
| `status=1` | 通常レコード（アクティブ） |
| `status=8` | テーブル宣言行 ＋ スキーマキャッシュ |
| `status=9` | 論理削除済み |

**status=8行の内容例：**

```json
{
  "schema": {
    "name":  "text",
    "age":   "text",
    "email": "text"
  }
}
```

**スキーマ自動進化（Auto Schema Evolution）：**

INSERTで存在しないテーブルへ書き込むと、status=8行が自動作成されます。
新しいカラムが追加されるたびにstatus=8行のschemaも自動追記されます（型は `text`）。

```
初回INSERT: {"name": "kimura", "age": 25}
  → status=8行を自動作成: {"name":"text", "age":"text"}
  → レコードをINSERT

2回目INSERT: {"name": "suzuki", "phone": "090-xxxx"}
  → "phone"が新カラム → status=8行に自動追記
  → schema: {"name":"text", "age":"text", "phone":"text"}
```

レガシーMySQL/PostgreSQL接続時は、実際のカラム型（`VARCHAR(255)`・`INT`等）がstatus=8行に記録されます。

---

## STATS — テーブル統計情報

レコード数・最終更新日時・使用容量などのメタ情報を取得します。AIエージェントがテーブルの状態を把握するのに有用です。

```json
{
  "dcl": "1.0",
  "action": "STATS",
  "table": "users"
}
```

レスポンス例：

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

## TIMELINE — レコード変更履歴

指定したレコードの変更履歴（作成・更新・論理削除）を時系列で取得します。AIエージェントがデータの変化を追跡するのに有用です。

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

## TABLE_COPY — テーブルのコピー

テーブルを別名でコピーします。バックアップ・マイグレーション・テスト用途に使用します。

```json
{
  "dcl": "1.0",
  "action": "TABLE_COPY",
  "table": "users",
  "target_table": "users_backup_20260328"
}
```

---

## TABLE_RENAME — テーブル名の変更

テーブルの名前を変更します。

```json
{
  "dcl": "1.0",
  "action": "TABLE_RENAME",
  "table": "users_old",
  "new_name": "users_archived"
}
```

---

## TABLE_DROP — テーブルの論理削除

テーブルを論理削除します。データは即座に物理削除されません。

```json
{
  "dcl": "1.0",
  "action": "TABLE_DROP",
  "table": "users_temp"
}
```

> **注意：** TABLE_DROPは論理削除です。SQLの `DROP TABLE` とは異なり、データは即座に消去されません。

---

## レスポンス形式

### 成功

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

### エラー

```json
{
  "dcl": "1.0",
  "status": "ERROR",
  "code": "TABLE_NOT_FOUND",
  "message": "Table 'users' does not exist"
}
```

### エラーコード一覧

| コード | 説明 |
|--------|------|
| `TABLE_NOT_FOUND` | 指定したテーブルが存在しない |
| `COLUMN_NOT_FOUND` | 指定したカラムが存在しない |
| `INVALID_ACTION` | 不明なアクションが指定された |
| `INVALID_WHERE` | WHERE条件を解析できなかった |
| `PERMISSION_DENIED` | テナントにアクセス権限がない |
| `CONNECTION_ERROR` | データベース接続に失敗した |

---

## 連携方式

DCLは2つのプロトコルで動作します。DCLコマンド自体はどちらでも同一です。

| プロトコル | 利用者 | ガイド |
|-----------|--------|--------|
| REST API（HTTPS POST） | フロントエンド・バックエンド・HTTPクライアント | [README.api.ja.md](README.api.ja.md) を参照 |
| MCP（JSON-RPC 2.0 / SSE） | AIエージェント（Claude・GPT・Cursor） | [README.mcp.ja.md](README.mcp.ja.md) を参照 |

### DCLペイロード（両プロトコル共通）

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

このDCL JSONは、REST APIでもMCPでも同一です。異なるのは外側のプロトコル層だけです。

---

## 対応データベース

| データベース | 状態 |
|---|---|
| MySQL 5.x / 8.x | ✅ 対応済み |
| PostgreSQL 9〜18 | ✅ 対応済み |
| その他 | 📋 対応予定 |

DCLはMySQLとPostgreSQLの差異を吸収します。DCLコマンドを1つ書けば、DataClaweが残りを処理します。

---

## 複雑さのレイヤー

DCLは2つのレイヤーで設計されています。必要なレイヤーを選択してください。

---

### DCL Standard — この仕様

AIエージェント・フロントエンドエンジニア・シンプルなデータ操作を行うすべての人向け。

- JOINなし。サブクエリなし。
- MySQLの基礎知識がある人なら誰でも読める。
- 現実のCRUDニーズの大半をカバー。

Standardで十分なら、それ以上は必要ありません。

---

### DCL Advanced — v1.1で追加予定

より高度な表現が必要なバックエンドエンジニア向け。

同じJSON構造。同じDataClaweエンジン。必要なときだけ追加の機能が使えます。

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

**サブクエリ**

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

**複数JOIN**

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

**Advancedで対応するJOINの種類**

| 種類 | 説明 |
|------|------|
| `INNER` | 両テーブルに一致するレコード |
| `LEFT` | 左テーブルの全レコード＋右テーブルの一致分 |
| `RIGHT` | 右テーブルの全レコード＋左テーブルの一致分 |

---

### なぜ2層構造なのか

```
SQLはすべてを1つの仕様で解決した。
だからSQLは30年経っても難しい。

DCL Standardはすべての人のためにある。
DCL Advancedは、それ以上が必要なときのためにある。
常にシンプルから始める。必要なときだけ深く入る。
```

同じエンジンが両方を処理します。同じJSON構造。StandardからAdvancedへ移行しても新しい構文を覚える必要はありません。新しいキーが増えるだけです。

---

### 意図的にサポートしないもの

以下はStandard・Advanced両方においてスコープ外です。DCLをAIエージェントにとって安全・予測可能に保つためです：

- ストアドプロシージャ
- スキーマ作成・`ALTER TABLE`
- 生SQLのパススルー
- 物理的なスキーマ破壊操作（`DROP TABLE` 相当の即時削除）

> **注意：** `TABLE_DROP` はDCL独自の**論理削除**であり、SQLの `DROP TABLE` とは異なります。

DCLはデータ操作言語であり、スキーマ管理言語ではありません。

---

## バージョニング

すべてのリクエストとレスポンスに `"dcl": "1.0"` フィールドが必須です。

将来のバージョンは後方互換性を維持します。DCL 1.0のリクエストはDCL 2.0のサーバーでも常に動作します。

---

## コントリビュート

DCLはオープン仕様です。フィードバック・提案・プルリクエストを歓迎します。

- 新しいアクションや演算子を提案する場合はIssueを作成
- ドキュメントや例の改善はプルリクエストで
- すべてのコントリビュートは設計思想に従うこと：**読みやすく、MySQL親しみやすく、BASICレベルのシンプルさ**

---

## ロードマップ

**v1.0 — DCL Standard**
- [ ] 仕様確定
- [ ] バリデーション用JSONスキーマ（`dcl.schema.json`）
- [ ] Goによるリファレンス実装（DataClaweエンジン）
- [ ] MCPサーバーリファレンス実装
- [ ] MySQLワイヤープロトコル対応
- [ ] マルチテナント分離仕様

**v1.1 — DCL Advanced**
- [ ] JOIN仕様（INNER / LEFT / RIGHT）
- [ ] サブクエリ対応
- [ ] ネストした集計

**SDK**
- [ ] JavaScript / TypeScript
- [ ] PHP
- [ ] Python
- [ ] Go

---

## 料金体系

DataClaweは段階的な料金体系を採用しており、個人開発者からエンタープライズまでスケールします。

**初期登録料：$20（全プラン共通・一回限り）**

| プラン | 月額 | レコード上限 | セッション上限 | SLA |
|--------|------|-------------|---------------|-----|
| **Free** | $0 | 2,000件 | 100回/日 | なし |
| **Personal** | $20 | 〜20,000件 | 〜20,000回/月 | なし |
| **Business** | $20〜 | 無制限 | 無制限 | なし |
| **Enterprise** | お問い合わせ | 無制限 | 無制限 | あり |

**従量単価（Business）**

| 項目 | 単価 |
|------|------|
| レコード保管 | $0.001 / 件 / 月 |
| セッション | $0.001 / 回 |

**1レコードのサイズ上限：100KB**
IoTセンサーデータ・CRM顧客データ・医療カルテ（テキスト）・WordPress投稿・金融取引データ等をカバーします。
画像・動画ファイルは対象外です。CDN・オブジェクトストレージに保存し、URLをDataClaweに保管してください。

Enterpriseプランの詳細は [dataclawe.com/enterprise](https://dataclawe.com/enterprise) からお問い合わせください。

---

## 背景

DCLは [DataClawe](https://dataclawe.com) プロジェクトの一部として開発されています。

DataClaweは、レガシーなMySQLおよびPostgreSQLシステムを、AIエージェント・LLMパイプライン・クラウドネイティブサービスと接続するデータベース変換エンジンです。元のシステムを書き直すことなく接続できます。

DCLは、この接続をデータベースエンジニアだけでなく、すべての人に可能にするコマンド言語です。

> レガシーシステムは破壊するものではない。翻訳するものだ。

---

## ライセンス

DCL仕様は [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) のもとで公開されています。

この仕様を実装・拡張・応用することは自由です。DataClaweへのクレジット表記を歓迎します。

---

*DCL v1.1 — 2026 — DataClawe Project*
