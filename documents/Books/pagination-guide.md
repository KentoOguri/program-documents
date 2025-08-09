# オフセットベース vs カーソルベース

## 目次
1. [ページネーションとは](#ページネーションとは)
2. [オフセットベースページネーション](#オフセットベースページネーション)
3. [カーソルベースページネーション](#カーソルベースページネーション)
4. [比較と使い分け](#比較と使い分け)
5. [実装例](#実装例)
6. [ベストプラクティス](#ベストプラクティス)

## ページネーションとは

ページネーションは、大量のデータを管理可能な小さなチャンクに分割する技術です。これにより：
- サーバーの負荷を軽減
- ネットワーク帯域幅を節約
- ユーザー体験を向上
- レスポンス時間を短縮

## オフセットベースページネーション

### 概要
オフセットベースページネーションは、データセットを固定サイズのページに分割し、ページ番号でアクセスする方式です。

### 仕組み
```
データセット: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
ページサイズ: 5

ページ1: [1, 2, 3, 4, 5]     (offset=0, limit=5)
ページ2: [6, 7, 8, 9, 10]    (offset=5, limit=5)
ページ3: [11, 12]            (offset=10, limit=5)
```

### APIリクエスト例
```
GET /api/users?page=2&limit=20
GET /api/users?offset=20&limit=20
```

### レスポンス例
```json
{
  "data": [
    { "id": 21, "name": "User 21" },
    { "id": 22, "name": "User 22" },
    // ... 18 more users
  ],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 100,
    "totalPages": 5,
    "hasNext": true,
    "hasPrev": true
  },
  "links": {
    "first": "/api/users?page=1&limit=20",
    "prev": "/api/users?page=1&limit=20",
    "next": "/api/users?page=3&limit=20",
    "last": "/api/users?page=5&limit=20"
  }
}
```

### メリット
1. **直感的**: ユーザーが特定のページに直接ジャンプ可能
2. **総数の把握**: 全体のデータ数とページ数が分かる
3. **実装が簡単**: SQLのLIMITとOFFSETで簡単に実装可能
4. **キャッシュ効率**: ページ単位でキャッシュしやすい

### デメリット
1. **パフォーマンス問題**: 大きなオフセット値で性能が劣化
2. **データの重複・欠落**: リアルタイムで更新されるデータでは問題が発生
3. **スケーラビリティ**: 大規模データセットには不向き

### SQL実装例
```sql
-- ページ2を取得（1ページ20件）
SELECT * FROM users 
ORDER BY id 
LIMIT 20 OFFSET 20;

-- より効率的な方法（インデックスを活用）
SELECT * FROM users 
WHERE id > 20 
ORDER BY id 
LIMIT 20;
```

## カーソルベースページネーション

### 概要
カーソルベースページネーションは、特定のポインタ（カーソル）を使用して、データセット内の位置を追跡する方式です。

### 仕組み
```
初回リクエスト: 最初から5件
レスポンス: [1, 2, 3, 4, 5] + nextCursor="eyJpZCI6NX0="

次のリクエスト: cursor="eyJpZCI6NX0="から5件
レスポンス: [6, 7, 8, 9, 10] + nextCursor="eyJpZCI6MTB9="
```

### APIリクエスト例
```
GET /api/users?limit=20
GET /api/users?cursor=eyJpZCI6MjB9&limit=20
```

### レスポンス例
```json
{
  "data": [
    { "id": 21, "name": "User 21" },
    { "id": 22, "name": "User 22" },
    // ... 18 more users
  ],
  "pagination": {
    "nextCursor": "eyJpZCI6NDB9",
    "prevCursor": "eyJpZCI6MjB9",
    "hasMore": true,
    "count": 20
  }
}
```

### カーソルの実装方法

#### 1. 単純なIDベース
```javascript
// エンコード
const cursor = Buffer.from(JSON.stringify({ id: 40 })).toString('base64');
// "eyJpZCI6NDB9"

// デコード
const decoded = JSON.parse(Buffer.from(cursor, 'base64').toString());
// { id: 40 }
```

#### 2. 複合キーベース
```javascript
// タイムスタンプとIDの組み合わせ
const cursor = Buffer.from(JSON.stringify({ 
  timestamp: "2024-01-01T00:00:00Z",
  id: 40 
})).toString('base64');
```

### メリット
1. **一貫性**: データの追加・削除があっても結果が安定
2. **高パフォーマンス**: 大規模データセットでも高速
3. **リアルタイム対応**: 頻繁に更新されるデータに最適
4. **無限スクロール**: UIの無限スクロールに最適

### デメリット
1. **ランダムアクセス不可**: 特定のページに直接ジャンプできない
2. **総数の取得困難**: 全体のデータ数を取得するには別途クエリが必要
3. **実装の複雑さ**: オフセットベースより実装が複雑
4. **双方向ナビゲーション**: 前のページへの移動が複雑

### SQL実装例
```sql
-- 初回リクエスト
SELECT * FROM users 
ORDER BY created_at DESC, id DESC 
LIMIT 20;

-- 次のページ（cursor = {created_at: '2024-01-01', id: 100}）
SELECT * FROM users 
WHERE (created_at, id) < ('2024-01-01', 100)
ORDER BY created_at DESC, id DESC 
LIMIT 20;
```

## 比較と使い分け

### 機能比較表

| 機能 | オフセットベース | カーソルベース |
|------|------------|--------------|
| 実装の容易さ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| パフォーマンス（大規模データ） | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| データ一貫性 | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| ランダムアクセス | ⭐⭐⭐⭐⭐ | ⭐ |
| 総数の取得 | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| キャッシュ効率 | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| リアルタイムデータ対応 | ⭐⭐ | ⭐⭐⭐⭐⭐ |

### 使用場面の推奨

#### オフセットベースが適している場合
- 管理画面やダッシュボード
- データの総数表示が必要
- ユーザーが特定のページにジャンプする必要がある
- データの更新頻度が低い
- 中小規模のデータセット

#### カーソルベースが適している場合
- ソーシャルメディアのフィード
- チャットメッセージの履歴
- 無限スクロールUI
- リアルタイムで更新されるデータ
- 大規模データセット
- モバイルアプリケーション

## 実装例

### Node.js/Express実装

#### オフセットベース実装
```javascript
// ルートハンドラー
app.get('/api/users', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const offset = (page - 1) * limit;

  // データ取得
  const users = await db.query(
    'SELECT * FROM users ORDER BY id LIMIT $1 OFFSET $2',
    [limit, offset]
  );

  // 総数取得
  const totalResult = await db.query('SELECT COUNT(*) FROM users');
  const total = parseInt(totalResult.rows[0].count);
  const totalPages = Math.ceil(total / limit);

  res.json({
    data: users.rows,
    pagination: {
      page,
      limit,
      total,
      totalPages,
      hasNext: page < totalPages,
      hasPrev: page > 1
    },
    links: {
      first: `/api/users?page=1&limit=${limit}`,
      prev: page > 1 ? `/api/users?page=${page - 1}&limit=${limit}` : null,
      next: page < totalPages ? `/api/users?page=${page + 1}&limit=${limit}` : null,
      last: `/api/users?page=${totalPages}&limit=${limit}`
    }
  });
});
```

#### カーソルベース実装
```javascript
// カーソルのエンコード/デコード
const encodeCursor = (data) => Buffer.from(JSON.stringify(data)).toString('base64');
const decodeCursor = (cursor) => JSON.parse(Buffer.from(cursor, 'base64').toString());

// ルートハンドラー
app.get('/api/users', async (req, res) => {
  const limit = parseInt(req.query.limit) || 20;
  const cursor = req.query.cursor;
  
  let query = 'SELECT * FROM users';
  let params = [limit + 1]; // +1 で次のページの存在を確認
  
  if (cursor) {
    const decoded = decodeCursor(cursor);
    query += ' WHERE id > $2';
    params.push(decoded.id);
  }
  
  query += ' ORDER BY id LIMIT $1';
  
  const users = await db.query(query, params);
  
  const hasMore = users.rows.length > limit;
  const data = hasMore ? users.rows.slice(0, -1) : users.rows;
  
  res.json({
    data,
    pagination: {
      nextCursor: hasMore ? encodeCursor({ id: data[data.length - 1].id }) : null,
      hasMore
    }
  });
});
```

### GraphQL実装例

```graphql
# スキーマ定義
type Query {
  # オフセットベース
  usersPage(page: Int, limit: Int): UserPage!
  
  # カーソルベース（Relay仕様準拠）
  users(first: Int, after: String, last: Int, before: String): UserConnection!
}

type UserPage {
  data: [User!]!
  pageInfo: PageInfo!
}

type PageInfo {
  page: Int!
  limit: Int!
  total: Int!
  totalPages: Int!
  hasNext: Boolean!
  hasPrev: Boolean!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: ConnectionPageInfo!
}

type UserEdge {
  node: User!
  cursor: String!
}

type ConnectionPageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

## ベストプラクティス

### 1. 適切な方式の選択
- ユースケースに応じて適切な方式を選択
- 必要に応じて両方の方式を提供することも検討

### 2. パフォーマンスの最適化
```sql
-- インデックスの作成
CREATE INDEX idx_users_created_at_id ON users(created_at DESC, id DESC);

-- カバリングインデックスの使用
CREATE INDEX idx_users_covering ON users(id, name, email) WHERE active = true;
```

### 3. レスポンスの一貫性
```javascript
// 統一されたレスポンス構造
const paginatedResponse = (data, pagination, links = null) => ({
  data,
  pagination,
  ...(links && { links }),
  meta: {
    timestamp: new Date().toISOString(),
    version: "1.0"
  }
});
```

### 4. エラーハンドリング
```javascript
// 不正なページ番号の処理
if (page < 1 || page > totalPages) {
  return res.status(400).json({
    error: {
      code: "INVALID_PAGE",
      message: "指定されたページが存在しません"
    }
  });
}
```

### 5. キャッシュ戦略
```javascript
// オフセットベースのキャッシュ
const cacheKey = `users:page:${page}:limit:${limit}`;
const cached = await redis.get(cacheKey);
if (cached) return JSON.parse(cached);

// カーソルベースのキャッシュ（より複雑）
const cacheKey = `users:cursor:${cursor || 'initial'}:limit:${limit}`;
```

### 6. セキュリティ考慮事項
```javascript
// リミットの上限設定
const MAX_LIMIT = 100;
const limit = Math.min(parseInt(req.query.limit) || 20, MAX_LIMIT);

// カーソルの検証
try {
  const decoded = decodeCursor(cursor);
  if (!decoded.id || typeof decoded.id !== 'number') {
    throw new Error('Invalid cursor');
  }
} catch (e) {
  return res.status(400).json({ error: 'Invalid cursor' });
}
```

### 7. ドキュメンテーション
```yaml
# OpenAPI仕様書の例
paths:
  /users:
    get:
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            minimum: 1
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
```

## まとめ

ページネーションの選択は、アプリケーションの要件によって決まります：

- **オフセットベース**: シンプルで直感的、管理画面に最適
- **カーソルベース**: 高性能でスケーラブル、フィードやタイムラインに最適

両方の方式を理解し、適切に使い分けることで、より良いユーザー体験を提供できます。