# Web API設計ガイドライン

## 1. RESTful API設計原則

### リソース指向の設計
- リソースは名詞で表現（動詞は使わない）
  - ✅ `/users`, `/products`, `/orders`
  - ❌ `/getUser`, `/createProduct`, `/deleteOrder`

### HTTPメソッドの適切な使用
- `GET`: リソースの取得（冪等性あり）
- `POST`: リソースの新規作成
- `PUT`: リソースの完全な更新（冪等性あり）
- `PATCH`: リソースの部分的な更新
- `DELETE`: リソースの削除（冪等性あり）

## 2. API URLの設計

### バージョニング戦略

#### URLパスでのバージョニング（推奨）
```
https://api.example.com/v1/users
https://api.example.com/v2/users
```

### 基本的なリソース操作
```
# ユーザー管理
GET    https://api.example.com/v1/users              # 一覧取得
GET    https://api.example.com/v1/users/123          # 詳細取得
POST   https://api.example.com/v1/users              # 新規作成
PUT    https://api.example.com/v1/users/123          # 完全更新
PATCH  https://api.example.com/v1/users/123          # 部分更新
DELETE https://api.example.com/v1/users/123          # 削除

# 商品管理
GET    https://api.example.com/v1/products           # 一覧取得
GET    https://api.example.com/v1/products/456       # 詳細取得
POST   https://api.example.com/v1/products           # 新規作成
PUT    https://api.example.com/v1/products/456       # 完全更新
DELETE https://api.example.com/v1/products/456       # 削除
```

### 関連リソースへのアクセス
```
# ユーザーの注文
GET    https://api.example.com/v1/users/123/orders
GET    https://api.example.com/v1/users/123/orders/789

# 注文の商品一覧
GET    https://api.example.com/v1/orders/789/items

# ユーザーのお気に入り
GET    https://api.example.com/v1/users/123/favorites
POST   https://api.example.com/v1/users/123/favorites/456
DELETE https://api.example.com/v1/users/123/favorites/456
```

### 検索・フィルタリング
```
# 商品検索（カテゴリ、価格範囲、ソート）
GET https://api.example.com/v1/products?category=electronics&min_price=100&max_price=1000&sort=-price

# ユーザー検索（ページネーション付き）
GET https://api.example.com/v1/users?search=john&page=2&limit=20

# 注文履歴（日付範囲、ステータスフィルタ）
GET https://api.example.com/v1/orders?from=2024-01-01&to=2024-12-31&status=completed&sort=-created_at
```

### アクション系エンドポイント
```
# 認証関連
POST https://api.example.com/v1/auth/login
POST https://api.example.com/v1/auth/logout
POST https://api.example.com/v1/auth/refresh
POST https://api.example.com/v1/auth/password-reset
POST https://api.example.com/v1/auth/password-reset/confirm

# 支払い処理
POST https://api.example.com/v1/orders/789/payment
POST https://api.example.com/v1/orders/789/payment/refund

# 一括操作
POST https://api.example.com/v1/products/bulk-update
DELETE https://api.example.com/v1/users/bulk-delete
```

# 参考リンク
https://qiita.com/baby-degu/items/6f516189445d98ddbb7d