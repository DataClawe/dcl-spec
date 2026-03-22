# DCL — REST API ガイド

> **ステータス：仕様のみ。エンドポイントは開発中です。**
> このドキュメントはAPIの予定される動作を説明しています。接続はEarly Testerリリース時に利用可能になります。

このガイドは **HTTPでDataClaweを呼び出すフロントエンドエンジニア・バックエンドエンジニア・あらゆる開発者** 向けです。

AIエージェント（Claude・GPT・Cursor）からの連携は [README.mcp.ja.md](README.mcp.ja.md) を参照してください。

DCLコマンド仕様については [README.ja.md](README.ja.md) を参照してください。

---

## 概要

DataClaweは標準的なHTTPS POSTリクエストでDCL JSONコマンドを受け付けます。

```
POST https://api.dataclawe.com/v1/dcl
Content-Type: application/json
Authorization: Bearer <your-access-key>

{ DCL JSON ペイロード }
```

特別なSDKは不要です。あらゆるHTTPクライアントから利用できます。

---

## 認証

すべてのリクエストにAuthorizationヘッダーのBearerトークンが必要です。

```
Authorization: Bearer dcl_kimura_clipjp_live_a1b2c3d4e5f6
```

アクセスキーはDataClaweコンソールからドメイン（テナント）単位で発行されます。

**アクセスキーの形式：**

```
dcl_{ユーザー名}_{ドメイン}_{環境}_{ランダム}

例：
dcl_kimura_clipjp_live_a1b2c3d4e5f6      （本番環境）
dcl_kimura_visitpass_dev_x9y8z7w6v5u4    （開発環境）
```

---

## リクエスト形式

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
  "code": "PERMISSION_DENIED",
  "message": "Invalid access key"
}
```

---

## HTTPステータスコード

| HTTPステータス | 意味 |
|---|---|
| `200` | リクエスト正常処理 |
| `400` | 不正なDCL JSON（リクエスト形式エラー） |
| `401` | Authorizationヘッダーが存在しないか無効 |
| `404` | テーブルが見つからない |
| `429` | レート制限超過 |
| `500` | 内部サーバーエラー |

注意：DCLレベルのエラー（`TABLE_NOT_FOUND` など）はHTTP `200` を返し、レスポンスボディに `"status": "ERROR"` が含まれます。

---

## コード例

### JavaScript（fetch）

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
console.log(data.data); // レコードの配列
```

### JavaScript（axios）

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

### PHP（curl）

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

### PHP（file_get_contents）

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

### Python（requests）

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

### curl（コマンドライン）

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

## よくある使い方

### INSERTして新規レコードのIDを取得する

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
// result.data[0].table_id — 新規作成されたレコードのID
```

### INSERT前に存在確認する

```javascript
// Step 1: 存在確認
const exists = await dcl({
  action: 'EXISTS',
  table: 'users',
  where: ["email = 'kimura@example.com'"]
});

// Step 2: 存在しない場合のみINSERT
if (!exists.data[0].exists) {
  await dcl({
    action: 'INSERT',
    table: 'users',
    data: { email: 'kimura@example.com', status: 'active' }
  });
}
```

### シンプルなラッパー関数（JavaScript）

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

// 使用例
const users = await dcl({
  action: 'SELECT',
  table: 'users',
  where: ["status = 'active'"],
  limit: 20
});
```

---

## レート制限

| プラン | リクエスト数/分 |
|--------|---------------|
| Early Tester | 60 rpm / ドメイン |
| Pro | 600 rpm / ドメイン |
| Enterprise | カスタム |

レート制限に達した場合はHTTP `429` と `Retry-After` ヘッダーが返されます。

---

## 注意事項

- アクセスキーは**ドメイン単位**で発行されます。1ドメインにつき1キーです。
- アクセスキーは秘密にしてください。フロントエンドコードに直接埋め込まないでください。バックエンドプロキシまたは環境変数を使用してください。
- DELETEは常に**論理削除**（ステータスフラグ）です。DCL経由でレコードが物理削除されることはありません。
- すべてのタイムスタンプはUTC ISO 8601形式で返されます。

---

## 関連ドキュメント

- [README.ja.md](README.ja.md) — DCLコマンド仕様
- [README.mcp.ja.md](README.mcp.ja.md) — MCP連携ガイド
- [dataclawe.com/early](https://dataclawe.com/early) — Early Testerプログラム（$20）

---

*DCL REST API ガイド — v1.0 — 2026 — DataClawe Project*
