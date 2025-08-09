# SQLアンチパターン 第17章〜第21章

## 目次
1. [第17章 スパゲッティクエリ](#第17章-スパゲッティクエリ)
2. [第18章 インプリシットカラム（暗黙の列）](#第18章-インプリシットカラム暗黙の列)
3. [第19章 リーダブルパスワード（読み取り可能パスワード）](#第19章-リーダブルパスワード読み取り可能パスワード)
4. [第21章 シュードキー・ニートフリーク（疑似キー潔癖症）](#第21章-シュードキーニートフリーク疑似キー潔癖症)

---

# 第17章 スパゲッティクエリ

## 問題と解決策

### ❌ 問題：複雑すぎる単一クエリ

```sql
-- ❌ 悪い例：巨大で理解困難な単一クエリ
SELECT 
    u.user_id,
    u.username,
    u.email,
    (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.user_id AND o.status = 'completed') as completed_orders,
    (SELECT SUM(o.total_amount) FROM orders o WHERE o.user_id = u.user_id AND o.status = 'completed') as total_spent,
    (SELECT AVG(r.rating) FROM reviews r JOIN orders o ON r.order_id = o.order_id WHERE o.user_id = u.user_id) as avg_rating,
    (SELECT p.product_name FROM orders o JOIN order_items oi ON o.order_id = oi.order_id 
     JOIN products p ON oi.product_id = p.product_id 
     WHERE o.user_id = u.user_id ORDER BY o.order_date DESC LIMIT 1) as latest_product,
    CASE 
        WHEN (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.user_id AND o.order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)) > 5 THEN 'VIP'
        WHEN (SELECT SUM(o.total_amount) FROM orders o WHERE o.user_id = u.user_id AND o.status = 'completed') > 10000 THEN 'Premium'
        ELSE 'Regular'
    END as customer_tier,
    (SELECT GROUP_CONCAT(DISTINCT c.category_name) 
     FROM orders o JOIN order_items oi ON o.order_id = oi.order_id 
     JOIN products p ON oi.product_id = p.product_id 
     JOIN categories c ON p.category_id = c.category_id 
     WHERE o.user_id = u.user_id) as purchased_categories
FROM users u
WHERE u.registration_date >= '2023-01-01'
ORDER BY total_spent DESC;
```

**主な問題**：
- 可読性の著しい低下
- メンテナンス困難
- 性能問題（N+1クエリ）
- デバッグ困難

### ✅ 解決策：分割統治、CTE、段階的処理

```sql
-- ✅ 正しい例：CTEによる段階的処理
WITH user_orders AS (
    SELECT 
        user_id,
        COUNT(*) FILTER (WHERE status = 'completed') as completed_orders,
        SUM(total_amount) FILTER (WHERE status = 'completed') as total_spent,
        COUNT(*) FILTER (WHERE order_date >= CURRENT_DATE - INTERVAL '30 days') as recent_orders
    FROM orders
    GROUP BY user_id
),
user_ratings AS (
    SELECT 
        o.user_id,
        AVG(r.rating) as avg_rating
    FROM orders o
    JOIN reviews r ON o.order_id = r.order_id
    GROUP BY o.user_id
),
latest_purchases AS (
    SELECT DISTINCT ON (o.user_id)
        o.user_id,
        p.product_name as latest_product
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    ORDER BY o.user_id, o.order_date DESC
),
user_categories AS (
    SELECT 
        o.user_id,
        string_agg(DISTINCT c.category_name, ', ') as purchased_categories
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    JOIN categories c ON p.category_id = c.category_id
    GROUP BY o.user_id
)
SELECT 
    u.user_id,
    u.username,
    u.email,
    COALESCE(uo.completed_orders, 0) as completed_orders,
    COALESCE(uo.total_spent, 0) as total_spent,
    ur.avg_rating,
    lp.latest_product,
    CASE 
        WHEN uo.recent_orders > 5 THEN 'VIP'
        WHEN uo.total_spent > 10000 THEN 'Premium'
        ELSE 'Regular'
    END as customer_tier,
    uc.purchased_categories
FROM users u
LEFT JOIN user_orders uo ON u.user_id = uo.user_id
LEFT JOIN user_ratings ur ON u.user_id = ur.user_id
LEFT JOIN latest_purchases lp ON u.user_id = lp.user_id
LEFT JOIN user_categories uc ON u.user_id = uc.user_id
WHERE u.registration_date >= '2023-01-01'
ORDER BY uo.total_spent DESC NULLS LAST;
```

## スパゲッティクエリアンチパターンとは？

一つのクエリに過度に多くの処理を詰め込み、複雑で理解困難、メンテナンス不可能なクエリを作成してしまうアンチパターンです。まるでスパゲッティのように絡み合った複雑な構造になることからこの名称で呼ばれます。

## ❌ 問題点の詳細

### 1. 可読性の著しい低下

```sql
-- ❌ 理解困難な複雑クエリ
SELECT 
    p.product_id,
    p.product_name,
    p.price,
    (SELECT AVG(r1.rating) FROM reviews r1 WHERE r1.product_id = p.product_id) as avg_rating,
    (SELECT COUNT(*) FROM reviews r2 WHERE r2.product_id = p.product_id) as review_count,
    (SELECT COUNT(DISTINCT oi.order_id) FROM order_items oi WHERE oi.product_id = p.product_id) as order_count,
    (SELECT SUM(oi2.quantity) FROM order_items oi2 WHERE oi2.product_id = p.product_id) as total_sold,
    (SELECT c.category_name FROM categories c WHERE c.category_id = p.category_id) as category_name,
    (SELECT b.brand_name FROM brands b WHERE b.brand_id = p.brand_id) as brand_name,
    (SELECT COUNT(*) FROM products p2 WHERE p2.category_id = p.category_id AND p2.price < p.price) + 1 as price_rank_in_category,
    CASE 
        WHEN (SELECT COUNT(*) FROM order_items oi3 WHERE oi3.product_id = p.product_id AND oi3.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)) > 100 THEN 'Hot'
        WHEN (SELECT AVG(r3.rating) FROM reviews r3 WHERE r3.product_id = p.product_id) > 4.5 THEN 'Recommended' 
        WHEN p.created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY) THEN 'New'
        ELSE 'Regular'
    END as product_status,
    (SELECT GROUP_CONCAT(t.tag_name) FROM product_tags pt JOIN tags t ON pt.tag_id = t.tag_id WHERE pt.product_id = p.product_id) as tags
FROM products p
WHERE p.is_active = 1
ORDER BY (SELECT COUNT(*) FROM order_items oi4 WHERE oi4.product_id = p.product_id) DESC;

-- 問題：
-- - 12個のサブクエリ
-- - 同一テーブルへの重複アクセス
-- - ネストした条件分岐
-- - 600行を超える長大なクエリ
```

### 2. 性能問題（N+1クエリパターン）

```sql
-- ❌ 非効率なサブクエリによる性能劣化
-- 上記クエリを1000件の商品で実行した場合:

-- メインクエリ: products テーブル全体スキャン (1回)
-- サブクエリ1: reviews テーブルスキャン × 1000回 (AVG計算)
-- サブクエリ2: reviews テーブルスキャン × 1000回 (COUNT計算) 
-- サブクエリ3: order_items テーブルスキャン × 1000回
-- サブクエリ4: order_items テーブルスキャン × 1000回
-- ...
-- 合計: 約10,000回のテーブルアクセス

-- 実行統計:
-- - 実行時間: 45秒
-- - 論理読み取り: 50,000ページ
-- - CPU時間: 30秒
```

### 3. メンテナンスの困難さ

```sql
-- ❌ 変更が困難なクエリ例
-- 要件変更: "最近30日間の売上も表示したい"

-- 現在のクエリに以下を追加する必要:
(SELECT SUM(oi.quantity * oi.unit_price) 
 FROM order_items oi 
 JOIN orders o ON oi.order_id = o.order_id 
 WHERE oi.product_id = p.product_id 
   AND o.order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
   AND o.status = 'completed') as recent_revenue

-- 問題:
-- - 既に複雑なクエリがさらに複雑化
-- - 新しいサブクエリ追加で性能がさらに劣化  
-- - テスト範囲の拡大
-- - 既存ロジックへの影響懸念
```

### 4. デバッグとテストの困難

```sql
-- ❌ エラー箇所の特定困難
-- エラーメッセージ例:
-- "Subquery returns more than 1 row"

-- 問題: 12個のサブクエリのどれが原因か不明
-- - reviews テーブルの重複データ？
-- - categories テーブルの結合問題？
-- - 条件分岐のどの部分？

-- デバッグのために全体を分解する必要
-- → 開発効率の著しい低下
```

## ✅ 分割統治による解決

### 1. CTE（Common Table Expression）による段階的処理

```sql
-- ✅ 段階的な処理による可読性向上
WITH product_stats AS (
    -- 商品の基本統計
    SELECT 
        product_id,
        AVG(rating) as avg_rating,
        COUNT(*) as review_count
    FROM reviews
    GROUP BY product_id
),
sales_stats AS (
    -- 販売統計
    SELECT 
        oi.product_id,
        COUNT(DISTINCT oi.order_id) as order_count,
        SUM(oi.quantity) as total_sold,
        SUM(CASE 
            WHEN o.order_date >= CURRENT_DATE - INTERVAL '30 days' 
            THEN oi.quantity * oi.unit_price 
            ELSE 0 
        END) as recent_revenue
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY oi.product_id
),
product_rankings AS (
    -- カテゴリ内価格ランキング
    SELECT 
        product_id,
        category_id,
        RANK() OVER (PARTITION BY category_id ORDER BY price ASC) as price_rank
    FROM products
),
product_tags AS (
    -- タグ情報の集約
    SELECT 
        pt.product_id,
        string_agg(t.tag_name, ', ') as tags
    FROM product_tags pt
    JOIN tags t ON pt.tag_id = t.tag_id
    GROUP BY pt.product_id
)
SELECT 
    p.product_id,
    p.product_name,
    p.price,
    c.category_name,
    b.brand_name,
    COALESCE(ps.avg_rating, 0) as avg_rating,
    COALESCE(ps.review_count, 0) as review_count,
    COALESCE(ss.order_count, 0) as order_count,
    COALESCE(ss.total_sold, 0) as total_sold,
    COALESCE(ss.recent_revenue, 0) as recent_revenue,
    pr.price_rank,
    CASE 
        WHEN ss.total_sold > 100 AND ss.recent_revenue > 10000 THEN 'Hot'
        WHEN ps.avg_rating > 4.5 THEN 'Recommended'
        WHEN p.created_at >= CURRENT_DATE - INTERVAL '7 days' THEN 'New'
        ELSE 'Regular'
    END as product_status,
    pt.tags
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN brands b ON p.brand_id = b.brand_id
LEFT JOIN product_stats ps ON p.product_id = ps.product_id
LEFT JOIN sales_stats ss ON p.product_id = ss.product_id
LEFT JOIN product_rankings pr ON p.product_id = pr.product_id
LEFT JOIN product_tags pt ON p.product_id = pt.product_id
WHERE p.is_active = TRUE
ORDER BY ss.total_sold DESC NULLS LAST;
```

### 2. 一時テーブルによる処理分割

```sql
-- ✅ 複雑な処理を一時テーブルで分割
-- ステップ1: 基本統計の計算
CREATE TEMPORARY TABLE temp_product_base_stats AS
SELECT 
    product_id,
    COUNT(*) as total_reviews,
    AVG(rating) as avg_rating,
    COUNT(CASE WHEN rating >= 4 THEN 1 END) as positive_reviews
FROM reviews
WHERE created_at >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY product_id;

CREATE INDEX idx_temp_product ON temp_product_base_stats(product_id);

-- ステップ2: 販売統計の計算
CREATE TEMPORARY TABLE temp_sales_analysis AS
SELECT 
    oi.product_id,
    COUNT(DISTINCT o.order_id) as unique_orders,
    SUM(oi.quantity) as total_quantity,
    SUM(oi.quantity * oi.unit_price) as total_revenue,
    COUNT(DISTINCT o.user_id) as unique_customers,
    MIN(o.order_date) as first_sale_date,
    MAX(o.order_date) as last_sale_date
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
WHERE o.status = 'completed'
  AND o.order_date >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY oi.product_id;

CREATE INDEX idx_temp_sales ON temp_sales_analysis(product_id);

-- ステップ3: 最終結果の組み立て
SELECT 
    p.product_id,
    p.product_name,
    p.price,
    c.category_name,
    COALESCE(tpbs.avg_rating, 0) as avg_rating,
    COALESCE(tpbs.total_reviews, 0) as review_count,
    COALESCE(tsa.total_revenue, 0) as revenue,
    COALESCE(tsa.unique_customers, 0) as customer_count,
    CASE 
        WHEN tsa.total_revenue > 50000 THEN 'Top Seller'
        WHEN tpbs.avg_rating > 4.5 THEN 'Highly Rated'
        WHEN p.created_at >= CURRENT_DATE - INTERVAL '30 days' THEN 'New Arrival'
        ELSE 'Standard'
    END as product_category
FROM products p
JOIN categories c ON p.category_id = c.category_id
LEFT JOIN temp_product_base_stats tpbs ON p.product_id = tpbs.product_id
LEFT JOIN temp_sales_analysis tsa ON p.product_id = tsa.product_id
WHERE p.is_active = TRUE
ORDER BY tsa.total_revenue DESC NULLS LAST;

-- 一時テーブルの削除
DROP TEMPORARY TABLE temp_product_base_stats;
DROP TEMPORARY TABLE temp_sales_analysis;
```

### 3. ストアドプロシージャによる処理分割

```sql
-- ✅ 複雑なビジネスロジックの処理分割
DELIMITER //
CREATE PROCEDURE GetProductAnalysis(
    IN p_category_id INT,
    IN p_date_from DATE,
    IN p_date_to DATE
)
BEGIN
    -- 1. 基本データの取得
    SELECT 
        p.product_id,
        p.product_name,
        p.price,
        p.created_at
    FROM products p
    WHERE (p_category_id IS NULL OR p.category_id = p_category_id)
      AND p.is_active = TRUE
    INTO @products_cursor;
    
    -- 2. レビュー統計の計算
    CALL CalculateReviewStats(p_category_id, p_date_from, p_date_to);
    
    -- 3. 売上統計の計算  
    CALL CalculateSalesStats(p_category_id, p_date_from, p_date_to);
    
    -- 4. 競合分析
    CALL AnalyzeCompetition(p_category_id);
    
    -- 5. 最終結果の出力
    SELECT 
        p.product_id,
        p.product_name,
        p.price,
        rs.avg_rating,
        rs.review_count,
        ss.total_sales,
        ss.market_share,
        ca.competitive_position
    FROM products p
    LEFT JOIN review_stats rs USING(product_id)
    LEFT JOIN sales_stats ss USING(product_id)  
    LEFT JOIN competitive_analysis ca USING(product_id)
    ORDER BY ss.total_sales DESC;
END //

CREATE PROCEDURE CalculateReviewStats(
    IN p_category_id INT,
    IN p_date_from DATE, 
    IN p_date_to DATE
)
BEGIN
    CREATE TEMPORARY TABLE review_stats AS
    SELECT 
        r.product_id,
        AVG(r.rating) as avg_rating,
        COUNT(*) as review_count,
        STDDEV(r.rating) as rating_stddev
    FROM reviews r
    JOIN products p ON r.product_id = p.product_id
    WHERE (p_category_id IS NULL OR p.category_id = p_category_id)
      AND r.created_at BETWEEN p_date_from AND p_date_to
    GROUP BY r.product_id;
END //
DELIMITER ;
```

## 実装パターン

### パターン1: 段階的絞り込みクエリ

```sql
-- ✅ フィルタ条件を段階的に適用
WITH active_products AS (
    -- フェーズ1: 基本フィルタ
    SELECT product_id, product_name, category_id, price
    FROM products
    WHERE is_active = TRUE
      AND price > 0
      AND created_at >= CURRENT_DATE - INTERVAL '2 years'
),
popular_products AS (
    -- フェーズ2: 人気商品の絞り込み  
    SELECT 
        ap.product_id,
        ap.product_name,
        ap.category_id,
        ap.price,
        COUNT(DISTINCT oi.order_id) as order_count
    FROM active_products ap
    JOIN order_items oi ON ap.product_id = oi.product_id
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
      AND o.order_date >= CURRENT_DATE - INTERVAL '6 months'
    GROUP BY ap.product_id, ap.product_name, ap.category_id, ap.price
    HAVING COUNT(DISTINCT oi.order_id) >= 10
),
rated_products AS (
    -- フェーズ3: レビューありの商品
    SELECT 
        pp.*,
        AVG(r.rating) as avg_rating,
        COUNT(r.rating) as review_count
    FROM popular_products pp
    JOIN reviews r ON pp.product_id = r.product_id
    WHERE r.rating IS NOT NULL
    GROUP BY pp.product_id, pp.product_name, pp.category_id, pp.price, pp.order_count
    HAVING AVG(r.rating) >= 3.0
)
-- フェーズ4: 最終結果とランキング
SELECT 
    rp.*,
    RANK() OVER (PARTITION BY rp.category_id ORDER BY rp.avg_rating DESC) as rating_rank,
    RANK() OVER (PARTITION BY rp.category_id ORDER BY rp.order_count DESC) as popularity_rank
FROM rated_products rp
ORDER BY rp.category_id, rp.avg_rating DESC;
```

### パターン2: 条件付き集約の分離

```sql
-- ✅ 複雑な条件分岐を分離
WITH order_metrics AS (
    SELECT 
        o.user_id,
        -- 期間別の注文統計
        COUNT(CASE WHEN o.order_date >= CURRENT_DATE - INTERVAL '30 days' THEN 1 END) as orders_30d,
        COUNT(CASE WHEN o.order_date >= CURRENT_DATE - INTERVAL '90 days' THEN 1 END) as orders_90d,
        COUNT(CASE WHEN o.order_date >= CURRENT_DATE - INTERVAL '1 year' THEN 1 END) as orders_1y,
        
        -- 金額別の統計
        SUM(CASE WHEN o.status = 'completed' THEN o.total_amount ELSE 0 END) as total_spent,
        AVG(CASE WHEN o.status = 'completed' THEN o.total_amount END) as avg_order_value,
        MAX(o.total_amount) as max_order_value,
        
        -- ステータス別統計
        COUNT(CASE WHEN o.status = 'completed' THEN 1 END) as completed_orders,
        COUNT(CASE WHEN o.status = 'cancelled' THEN 1 END) as cancelled_orders
    FROM orders o
    GROUP BY o.user_id
),
user_segments AS (
    -- セグメント分類ロジック
    SELECT 
        om.*,
        CASE 
            WHEN om.orders_30d >= 5 AND om.total_spent >= 10000 THEN 'VIP'
            WHEN om.orders_90d >= 3 AND om.total_spent >= 5000 THEN 'Premium'  
            WHEN om.orders_1y >= 2 THEN 'Regular'
            WHEN om.orders_1y = 1 THEN 'New Customer'
            ELSE 'Inactive'
        END as customer_segment,
        
        CASE
            WHEN om.cancelled_orders * 100.0 / NULLIF(om.orders_1y, 0) > 20 THEN 'High Risk'
            WHEN om.avg_order_value > 1000 THEN 'High Value'
            ELSE 'Standard'
        END as risk_category
    FROM order_metrics om
)
SELECT 
    u.user_id,
    u.username,
    u.email,
    us.customer_segment,
    us.risk_category,
    us.total_spent,
    us.orders_30d,
    us.avg_order_value
FROM users u
LEFT JOIN user_segments us ON u.user_id = us.user_id
ORDER BY us.total_spent DESC NULLS LAST;
```

### パターン3: ウィンドウ関数による効率化

```sql
-- ✅ サブクエリをウィンドウ関数で置き換え
SELECT 
    p.product_id,
    p.product_name,
    p.price,
    c.category_name,
    
    -- レビュー統計（ウィンドウ関数使用）
    AVG(r.rating) OVER (PARTITION BY p.product_id) as avg_rating,
    COUNT(r.rating) OVER (PARTITION BY p.product_id) as review_count,
    
    -- カテゴリ内ランキング
    RANK() OVER (PARTITION BY p.category_id ORDER BY p.price DESC) as price_rank_in_category,
    PERCENT_RANK() OVER (PARTITION BY p.category_id ORDER BY p.price) as price_percentile,
    
    -- 売上ランキング
    DENSE_RANK() OVER (ORDER BY sales_summary.total_sales DESC) as sales_rank,
    
    -- 移動平均（直近5商品の平均価格）
    AVG(p.price) OVER (
        PARTITION BY p.category_id 
        ORDER BY p.created_at 
        ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
    ) as moving_avg_price
    
FROM products p
JOIN categories c ON p.category_id = c.category_id
LEFT JOIN reviews r ON p.product_id = r.product_id
LEFT JOIN (
    SELECT 
        product_id,
        SUM(quantity * unit_price) as total_sales
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY product_id
) sales_summary ON p.product_id = sales_summary.product_id
WHERE p.is_active = TRUE
ORDER BY sales_rank NULLS LAST;
```

## まとめ

### スパゲッティクエリアンチパターンの問題点
1. **可読性の低下**: 複雑すぎて理解困難
2. **性能問題**: N+1クエリによる非効率性
3. **メンテナンス困難**: 変更や拡張が困難
4. **デバッグ困難**: エラー箇所の特定が困難

### 分割統治による利点
1. **可読性向上**: 段階的処理による理解しやすさ
2. **性能改善**: 効率的な結合とインデックス活用
3. **保守性**: 個別機能の独立したテスト・修正
4. **再利用性**: CTEや一時テーブルの再利用

**重要**: 複雑なクエリは分割統治の原則に従い、CTE、一時テーブル、ストアドプロシージャなどを活用して段階的に処理することが重要です。

---

# 第18章 インプリシットカラム（暗黙の列）

## 問題と解決策

### ❌ 問題：SELECT *使用の問題

```sql
-- ❌ 悪い例：SELECT * の使用
SELECT * FROM users u
JOIN orders o ON u.user_id = o.user_id
JOIN order_items oi ON o.order_id = oi.order_id;

-- INSERT時のSELECT *
INSERT INTO user_backup 
SELECT * FROM users WHERE last_login < '2023-01-01';

-- VIEW作成でのSELECT *
CREATE VIEW active_users AS 
SELECT * FROM users WHERE is_active = 1;
```

**主な問題**：
- カラムの明示性欠如
- スキーマ変更による破綻
- 不要データの転送
- セキュリティリスク

### ✅ 解決策：明示的な列指定

```sql
-- ✅ 正しい例：明示的な列指定
SELECT 
    u.user_id,
    u.username,
    u.email,
    u.created_at,
    o.order_id,
    o.order_date,
    o.total_amount,
    oi.product_id,
    oi.quantity,
    oi.unit_price
FROM users u
JOIN orders o ON u.user_id = o.user_id  
JOIN order_items oi ON o.order_id = oi.order_id;

-- 必要な列のみを明示的に指定
INSERT INTO user_backup (user_id, username, email, last_login, created_at)
SELECT user_id, username, email, last_login, created_at
FROM users 
WHERE last_login < '2023-01-01';
```

## インプリシットカラムアンチパターンとは？

SELECT *（アスタリスク）を使用してすべての列を暗黙的に取得することで、明示性の欠如、性能問題、保守性の低下を引き起こすアンチパターンです。

## ❌ 問題点の詳細

### 1. スキーマ変更による破綻

```sql
-- ❌ 初期状態のテーブル
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    price DECIMAL(10,2)
);

-- ❌ SELECT * を使用したビュー
CREATE VIEW product_list AS
SELECT * FROM products WHERE price > 100;

-- ❌ SELECT * を使用したアプリケーションコード
-- SELECT * FROM products ORDER BY price DESC;
-- 期待される結果順序: product_id, product_name, price

-- スキーマ変更: 新しい列の追加
ALTER TABLE products ADD COLUMN description TEXT AFTER product_name;
ALTER TABLE products ADD COLUMN category_id INT AFTER product_id;

-- 結果: 列順序の変更
-- 新しい順序: product_id, category_id, product_name, description, price

-- 問題:
-- 1. アプリケーションコードで列の位置（インデックス）を使用している場合に破綻
-- 2. INSERT SELECT * 文で列数不一致エラー
-- 3. ビューの結果列が予期しない順序に変更
```

### 2. 不要なデータ転送による性能問題

```sql
-- ❌ 不要な大容量データの転送
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    bio TEXT,
    avatar_image LONGBLOB,      -- 1-5MB の画像データ
    settings JSON,              -- 複雑な設定情報
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    last_login TIMESTAMP
);

-- ❌ ユーザーリスト表示でSELECT *
-- 実際に必要: username, email, last_login のみ
SELECT * FROM user_profiles 
WHERE last_login >= DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY last_login DESC;

-- 問題:
-- - 1000ユーザーで 1-5GB のavatar_imageを不要に転送
-- - ネットワーク帯域の浪費
-- - メモリ使用量の増大
-- - レスポンス時間の悪化

-- 実行時間比較:
-- SELECT * : 15秒, 転送量: 2.3GB
-- SELECT username, email, last_login : 0.2秒, 転送量: 50KB
```

### 3. セキュリティリスク

```sql
-- ❌ 機密情報の意図しない露出
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    password_hash VARCHAR(255),    -- 機密情報
    salt VARCHAR(100),            -- 機密情報  
    api_key VARCHAR(128),         -- 機密情報
    credit_card_hash VARCHAR(255), -- 機密情報
    ssn_encrypted VARBINARY(100), -- 機密情報
    profile_data JSON,
    created_at TIMESTAMP
);

-- ❌ APIでSELECT * を使用
-- 内部的な管理者用クエリのつもりが...
SELECT * FROM users WHERE user_id = ?;

-- 問題: すべての機密情報がアプリケーション層に送信される
-- - password_hash
-- - salt  
-- - api_key
-- - credit_card_hash
-- - ssn_encrypted

-- ログ出力、エラー画面、デバッグ情報で機密データが露出する危険
```

### 4. JOIN時の列名衝突

```sql
-- ❌ 複数テーブルでの列名重複問題
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    created_at TIMESTAMP,  -- 重複
    updated_at TIMESTAMP   -- 重複
);

CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    created_at TIMESTAMP,  -- 重複
    updated_at TIMESTAMP   -- 重複
);

-- ❌ SELECT * でJOIN
SELECT * FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id;

-- 問題:
-- - created_at が2つ存在（どちらがどのテーブルか不明）
-- - updated_at が2つ存在
-- - アプリケーションでアクセス時に曖昧性エラー
-- - MySQL: "Duplicate column name 'created_at'"
-- - PostgreSQL: "column name 'created_at' appears more than once"
```

### 5. INSERT SELECT での列順序依存

```sql
-- ❌ 危険な INSERT SELECT * パターン
-- ソーステーブル
CREATE TABLE temp_users (
    user_id INT,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP
);

-- デスティネーションテーブル  
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100), 
    created_at TIMESTAMP
);

-- ❌ 初期状態では正常動作
INSERT INTO users SELECT * FROM temp_users;  -- 成功

-- スキーマ変更後の問題
ALTER TABLE temp_users ADD COLUMN phone VARCHAR(20) AFTER email;

-- temp_users の新構造: user_id, username, email, phone, created_at
-- users の構造: user_id, username, email, created_at

-- ❌ 同じクエリが列数不整合で失敗
INSERT INTO users SELECT * FROM temp_users;  
-- ERROR: Column count doesn't match value count at row 1

-- さらに悪いケース: 列数は一致するが型が異なる場合
-- phone の値が created_at に、created_at の値が存在しない列にマッピング
-- データ破損の可能性
```

## ✅ 明示的な列指定による解決

### 1. 必要な列のみの明示的指定

```sql
-- ✅ ユースケースに応じた列選択
-- ユーザーリスト表示用
SELECT 
    user_id,
    username,
    email,
    last_login,
    created_at
FROM user_profiles
WHERE is_active = TRUE
ORDER BY last_login DESC;

-- プロフィール詳細表示用（画像除外）
SELECT 
    user_id,
    username,
    email,
    first_name,
    last_name,
    bio,
    settings,
    created_at,
    updated_at
FROM user_profiles
WHERE user_id = ?;

-- 画像データが必要な場合のみ
SELECT 
    user_id,
    avatar_image
FROM user_profiles
WHERE user_id = ? AND avatar_image IS NOT NULL;
```

### 2. JOIN時の明示的列指定

```sql
-- ✅ JOIN時の曖昧性回避
SELECT 
    o.order_id,
    o.user_id,
    o.order_date,
    o.total_amount,
    o.status as order_status,
    o.created_at as order_created_at,
    
    oi.item_id,
    oi.product_id,
    oi.quantity,
    oi.unit_price,
    oi.created_at as item_created_at,
    
    p.product_name,
    p.price as current_price,
    
    u.username,
    u.email
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN users u ON o.user_id = u.user_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '30 days';
```

### 3. 安全なデータ転送パターン

```sql
-- ✅ セキュリティを考慮したクエリパターン
-- パブリックAPI用（機密情報除外）
CREATE VIEW public_user_info AS
SELECT 
    user_id,
    username,
    email,
    profile_data->'$.display_name' as display_name,
    profile_data->'$.bio' as bio,
    created_at
FROM users
WHERE is_active = TRUE;

-- 管理者用（必要最小限の機密情報）
CREATE VIEW admin_user_summary AS
SELECT 
    user_id,
    username,
    email,
    last_login,
    login_count,
    account_status,
    created_at
FROM users;

-- 認証専用（認証情報のみ）
CREATE VIEW auth_user_data AS
SELECT 
    user_id,
    username,
    password_hash,
    salt,
    failed_login_attempts,
    account_locked_until
FROM users;
```

### 4. 型安全なデータ移行

```sql
-- ✅ 明示的な列指定によるデータ移行
-- テーブル構造の確認
SELECT 
    COLUMN_NAME,
    DATA_TYPE,
    IS_NULLABLE,
    COLUMN_DEFAULT
FROM information_schema.COLUMNS
WHERE TABLE_NAME = 'users'
ORDER BY ORDINAL_POSITION;

-- 安全なデータコピー
INSERT INTO users (user_id, username, email, created_at)
SELECT 
    user_id,
    username,  
    email,
    created_at
FROM temp_users
WHERE email IS NOT NULL
  AND username IS NOT NULL;

-- 列追加後も安全
ALTER TABLE temp_users ADD COLUMN phone VARCHAR(20);

-- 同じクエリが引き続き動作（phone列は無視される）
INSERT INTO users (user_id, username, email, created_at)
SELECT 
    user_id,
    username,
    email,
    created_at  -- phone列は明示的に除外
FROM temp_users;
```

## 実装パターン

### パターン1: レイヤー別列選択

```sql
-- ✅ プレゼンテーション層用
CREATE VIEW user_list_view AS
SELECT 
    user_id,
    username,
    CONCAT(first_name, ' ', last_name) as full_name,
    email,
    last_login,
    CASE 
        WHEN last_login >= DATE_SUB(NOW(), INTERVAL 7 DAY) THEN 'Active'
        WHEN last_login >= DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 'Recent'
        ELSE 'Inactive'
    END as activity_status
FROM users
WHERE is_active = TRUE;

-- ✅ データ処理層用
CREATE VIEW user_analytics_view AS
SELECT 
    user_id,
    username,
    registration_date,
    last_login,
    login_count,
    total_orders,
    total_spent,
    avg_order_value,
    customer_segment
FROM users
WHERE registration_date >= DATE_SUB(NOW(), INTERVAL 2 YEAR);

-- ✅ レポート層用
CREATE VIEW user_report_view AS
SELECT 
    user_id,
    username,
    email,
    registration_date,
    total_orders,
    total_spent,
    last_order_date,
    days_since_last_order
FROM users
WHERE total_orders > 0;
```

### パターン2: 動的列選択（ストアドプロシージャ）

```sql
-- ✅ 用途に応じた動的列選択
DELIMITER //
CREATE PROCEDURE GetUserData(
    IN user_id INT,
    IN data_scope ENUM('basic', 'profile', 'admin', 'full')
)
BEGIN
    CASE data_scope
        WHEN 'basic' THEN
            SELECT 
                user_id,
                username,
                email,
                created_at
            FROM users WHERE user_id = user_id;
            
        WHEN 'profile' THEN
            SELECT 
                user_id,
                username,
                email,
                first_name,
                last_name,
                bio,
                settings,
                avatar_url
            FROM users WHERE user_id = user_id;
            
        WHEN 'admin' THEN
            SELECT 
                user_id,
                username,
                email,
                last_login,
                login_count,
                failed_attempts,
                account_status,
                created_at,
                updated_at
            FROM users WHERE user_id = user_id;
            
        WHEN 'full' THEN
            -- 管理者のみ、機密情報を含む全データ
            SELECT 
                user_id,
                username,
                email,
                first_name,
                last_name,
                bio,
                settings,
                last_login,
                login_count,
                account_status,
                created_at,
                updated_at
            FROM users WHERE user_id = user_id;
    END CASE;
END //
DELIMITER ;
```

### パターン3: 列指定の標準化

```sql
-- ✅ 標準的な列セットの定義
-- 基本ユーザー情報
CREATE VIEW user_basic AS
SELECT 
    user_id,
    username,
    email,
    created_at
FROM users;

-- ユーザーアクティビティ
CREATE VIEW user_activity AS  
SELECT
    user_id,
    last_login,
    login_count,
    total_sessions,
    avg_session_duration
FROM user_statistics;

-- 注文履歴サマリー
CREATE VIEW user_orders AS
SELECT
    user_id,
    total_orders,
    total_spent,
    avg_order_value,
    last_order_date,
    first_order_date
FROM order_statistics;

-- 結合用の統合ビュー
CREATE VIEW user_dashboard AS
SELECT 
    ub.user_id,
    ub.username,
    ub.email,
    ub.created_at,
    ua.last_login,
    ua.login_count,
    uo.total_orders,
    uo.total_spent
FROM user_basic ub
LEFT JOIN user_activity ua ON ub.user_id = ua.user_id
LEFT JOIN user_orders uo ON ub.user_id = uo.user_id;
```

### パターン4: パフォーマンス最適化

```sql
-- ✅ インデックス効率を考慮した列選択
-- カバリングインデックスを活用
CREATE INDEX idx_user_list_covering ON users(
    is_active, last_login, user_id, username, email
);

-- インデックスのみでクエリ完結（テーブルアクセス不要）
SELECT 
    user_id,
    username,
    email
FROM users
WHERE is_active = TRUE
  AND last_login >= DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY last_login DESC;

-- 大きなBLOB/TEXT列を避ける最適化
SELECT 
    p.product_id,
    p.product_name,
    p.price,
    p.category_id,
    -- description(TEXT)は除外してパフォーマンス向上
    p.created_at
FROM products p
WHERE p.is_active = TRUE
ORDER BY p.created_at DESC
LIMIT 100;

-- 詳細が必要な場合のみ追加クエリ
SELECT description 
FROM products 
WHERE product_id = ?;
```

## まとめ

### インプリシットカラムアンチパターンの問題点
1. **スキーマ変更への脆弱性**: 列追加・順序変更による破綻
2. **性能問題**: 不要データの転送によるオーバーヘッド
3. **セキュリティリスク**: 機密情報の意図しない露出
4. **保守性の低下**: 列の依存関係が不明確

### 明示的列指定による利点
1. **スキーマ変更耐性**: 列構造変更に対する堅牢性
2. **性能最適化**: 必要データのみの効率的転送
3. **セキュリティ向上**: 意図的な情報制御
4. **可読性向上**: 使用する列の明確化

**重要**: SELECT *は開発中の一時的な使用に留め、本番環境では必要な列のみを明示的に指定することを強く推奨します。

---

# 第19章 リーダブルパスワード（読み取り可能パスワード）

## 問題と解決策

### ❌ 問題：パスワードの平文保存

```sql
-- ❌ 悪い例：パスワードの平文保存
CREATE TABLE users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,  -- 平文パスワード！
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ❌ パスワードが丸見えで保存
INSERT INTO users (username, email, password) VALUES
('john_doe', 'john@example.com', 'mypassword123'),
('jane_smith', 'jane@example.com', 'secretpass456'),
('admin', 'admin@company.com', 'admin123');

-- ❌ 認証時の平文比較
SELECT * FROM users 
WHERE username = 'john_doe' AND password = 'mypassword123';
```

**主な問題**：
- データ漏洩時の完全な情報露出
- 内部関係者による悪用可能性
- 法規制・コンプライアンス違反
- ユーザーの信頼失墜

### ✅ 解決策：適切なハッシュ化

```sql
-- ✅ 正しい例：ハッシュ化されたパスワード保存
CREATE TABLE users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,  -- ハッシュ化済み
    salt VARCHAR(32) NOT NULL,            -- ソルト
    hash_algorithm VARCHAR(20) DEFAULT 'bcrypt',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    password_updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ✅ ハッシュ化されたパスワードの保存例
INSERT INTO users (username, email, password_hash, salt) VALUES
('john_doe', 'john@example.com', '$2y$12$LQv3c1yqBWVHxkd0LQ4YYu.aNTmw9VC/Fb7dh2WYNyJdYmD3L8pTC', 'a1b2c3d4'),
('jane_smith', 'jane@example.com', '$2y$12$3H8R6Zx9P2kL1N5Q7Yc8MnP4Bd6wQ9Tr2LxN5Kb3Df8Gh9Ms1Vp7E', 'e5f6g7h8');
```

## リーダブルパスワードアンチパターンとは？

パスワードを平文（読み取り可能な形式）でデータベースに保存することで、深刻なセキュリティリスクを招くアンチパターンです。これは現代のシステムにおいて最も危険なアンチパターンの一つです。

## ❌ 問題点の詳細

### 1. データ漏洩時の壊滅的影響

```sql
-- ❌ 平文パスワードテーブルの例
CREATE TABLE user_accounts (
    account_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    password VARCHAR(100),  -- 平文保存！
    security_question VARCHAR(200),
    security_answer VARCHAR(100),  -- これも平文！
    backup_email VARCHAR(100),
    phone_number VARCHAR(20)
);

-- 実際のデータ例
INSERT INTO user_accounts VALUES
(1, 'ceo_johnson', 'ceo@company.com', 'Company2024!', 'What is your pet name?', 'Fluffy', 'personal@ceo.com', '+1234567890'),
(2, 'hr_manager', 'hr@company.com', 'HumanRes#789', 'Where were you born?', 'New York', 'hr.personal@gmail.com', '+1234567891'),
(3, 'john_employee', 'john@company.com', 'john_password', 'Your favorite color?', 'blue', 'john.personal@yahoo.com', '+1234567892');

-- ❌ SQL Injection や内部漏洩で全情報が露出
SELECT * FROM user_accounts;
-- 結果：全ユーザーのパスワード、セキュリティ回答が平文で表示

-- 問題の深刻度:
-- 1. パスワード使い回しによる他サービスへの被害拡大
-- 2. 企業の機密情報アクセス権限の悪用
-- 3. 法的責任（GDPR、個人情報保護法等）
-- 4. 企業評判・信頼の失墜
```

### 2. 内部不正・権限悪用のリスク

```sql
-- ❌ 開発者・管理者による悪用可能性
-- 運用中のシステムでのパスワード検索
SELECT username, password FROM users 
WHERE email LIKE '%@company.com'
ORDER BY created_at DESC;

-- 特定ユーザーのパスワード取得
SELECT password FROM users WHERE username = 'target_user';

-- パスワード傾向分析（悪意ある目的）
SELECT 
    password,
    COUNT(*) as usage_count
FROM users 
GROUP BY password 
ORDER BY usage_count DESC;
-- 結果：よく使われるパスワードパターンが判明

-- 問題：
-- - データベース管理者が全ユーザーのパスワードを見放題
-- - 開発者がデバッグ名目で機密情報取得
-- - バックアップファイルからの情報漏洩
-- - ログファイルへの平文パスワード記録
```

### 3. パスワードポリシーの無効化

```sql
-- ❌ 平文保存による弱いパスワードの放置
-- パスワード強度チェック（平文でないと困難）
SELECT 
    username,
    password,
    LENGTH(password) as length,
    CASE 
        WHEN password REGEXP '^[a-z]+$' THEN 'Weak: Only lowercase'
        WHEN password REGEXP '^[0-9]+$' THEN 'Weak: Only numbers'  
        WHEN LENGTH(password) < 8 THEN 'Weak: Too short'
        ELSE 'Unknown strength'
    END as password_strength
FROM users
ORDER BY LENGTH(password);

-- 結果例:
-- john_doe | password | 8 | Unknown strength  
-- admin | admin123 | 8 | Weak: predictable
-- user1 | 12345678 | 8 | Weak: Only numbers

-- 問題：
-- - 弱いパスワードの特定・強制変更が可能（セキュリティ上は良いが、平文前提）
-- - 類似パスワードの検出
-- - パスワード履歴の平文管理
-- - 一時パスワードの平文保存
```

### 4. ログ・バックアップファイルでの露出

```sql
-- ❌ ログファイルでの平文パスワード記録
-- アプリケーションログの例
-- 2024-03-15 10:30:15 DEBUG: User login attempt - Username: john_doe, Password: mypassword123
-- 2024-03-15 10:30:16 INFO: Login successful for john_doe
-- 2024-03-15 10:35:22 ERROR: Login failed - Username: hacker, Password: admin123
-- 2024-03-15 10:40:30 DEBUG: Password reset for user jane_doe, New password: temp456789

-- バックアップファイルでの露出
mysqldump production_db > backup_2024_03_15.sql
-- backup_2024_03_15.sql の内容:
INSERT INTO `users` VALUES 
(1,'admin','admin@company.com','supersecret2024'),
(2,'manager','manager@company.com','ManagerPass!23'),
(3,'developer','dev@company.com','DevCode789');

-- 問題：
-- - ログファイルが第三者に渡る可能性
-- - バックアップファイルの不適切な保存・送信
-- - 監査ログでの平文パスワード記録
-- - エラー情報での意図しない露出
```

## ✅ 適切なハッシュ化による解決

### 1. bcryptによる堅牢なハッシュ化

```sql
-- ✅ セキュアなユーザーテーブル設計
CREATE TABLE secure_users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    
    -- セキュアなパスワード管理
    password_hash VARCHAR(255) NOT NULL,  -- bcrypt等のハッシュ
    salt VARCHAR(64) NOT NULL,            -- ランダムソルト
    hash_algorithm VARCHAR(20) DEFAULT 'bcrypt',
    hash_cost INT DEFAULT 12,             -- bcryptのコストファクタ
    
    -- パスワード管理
    password_created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    password_expires_at TIMESTAMP NULL,   -- パスワード期限
    password_history JSON,                -- 過去のハッシュ（再利用防止）
    
    -- セキュリティ情報
    failed_login_attempts INT DEFAULT 0,
    account_locked_until TIMESTAMP NULL,
    last_login_at TIMESTAMP NULL,
    last_password_change TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 監査情報
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_username (username),
    INDEX idx_email (email),
    INDEX idx_account_locked (account_locked_until)
);
```

### 2. パスワードハッシュ化の実装例

```sql
-- ✅ パスワード登録用ストアドプロシージャ
DELIMITER //
CREATE PROCEDURE RegisterUser(
    IN p_username VARCHAR(50),
    IN p_email VARCHAR(100),
    IN p_plain_password VARCHAR(255)
)
BEGIN
    DECLARE v_salt VARCHAR(64);
    DECLARE v_password_hash VARCHAR(255);
    DECLARE v_user_exists INT DEFAULT 0;
    
    -- ユーザー重複チェック
    SELECT COUNT(*) INTO v_user_exists
    FROM secure_users
    WHERE username = p_username OR email = p_email;
    
    IF v_user_exists > 0 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Username or email already exists';
    END IF;
    
    -- ランダムソルト生成（アプリケーション側で生成推奨）
    SET v_salt = SHA2(CONCAT(p_username, NOW(), RAND()), 256);
    
    -- パスワードハッシュ化（実際はアプリケーション側で実行）
    -- bcrypt(p_plain_password + v_salt, 12)
    SET v_password_hash = CONCAT('$2y$12$', v_salt, 'mock_hash_placeholder');
    
    -- セキュアなユーザー作成
    INSERT INTO secure_users (
        username, 
        email, 
        password_hash, 
        salt, 
        hash_algorithm,
        hash_cost
    ) VALUES (
        p_username,
        p_email,
        v_password_hash,
        v_salt,
        'bcrypt',
        12
    );
    
    SELECT 'User registered successfully' as result;
END //
DELIMITER ;
```

### 3. 安全な認証処理

```sql
-- ✅ 認証用ストアドプロシージャ
DELIMITER //
CREATE PROCEDURE AuthenticateUser(
    IN p_username VARCHAR(50),
    IN p_plain_password VARCHAR(255),
    OUT p_user_id INT,
    OUT p_auth_result VARCHAR(50)
)
BEGIN
    DECLARE v_stored_hash VARCHAR(255);
    DECLARE v_salt VARCHAR(64);
    DECLARE v_failed_attempts INT;
    DECLARE v_locked_until TIMESTAMP;
    DECLARE v_found INT DEFAULT 0;
    
    -- ユーザー情報取得
    SELECT 
        user_id,
        password_hash,
        salt,
        failed_login_attempts,
        account_locked_until
    INTO 
        p_user_id,
        v_stored_hash,
        v_salt,
        v_failed_attempts,
        v_locked_until
    FROM secure_users
    WHERE username = p_username AND account_locked_until < NOW();
    
    SET v_found = ROW_COUNT();
    
    IF v_found = 0 THEN
        SET p_auth_result = 'USER_NOT_FOUND_OR_LOCKED';
        SET p_user_id = NULL;
    ELSE
        -- パスワード検証（アプリケーション側で実行）
        -- bcrypt_verify(p_plain_password + v_salt, v_stored_hash)
        
        -- 模擬的な検証（実際はアプリケーション側）
        IF CONCAT('hash_of_', p_plain_password) = 'mock_condition' THEN
            -- 認証成功
            UPDATE secure_users 
            SET 
                failed_login_attempts = 0,
                last_login_at = NOW()
            WHERE user_id = p_user_id;
            
            SET p_auth_result = 'SUCCESS';
        ELSE
            -- 認証失敗
            UPDATE secure_users 
            SET 
                failed_login_attempts = failed_login_attempts + 1,
                account_locked_until = CASE 
                    WHEN failed_login_attempts + 1 >= 5 
                    THEN DATE_ADD(NOW(), INTERVAL 30 MINUTE)
                    ELSE account_locked_until
                END
            WHERE user_id = p_user_id;
            
            SET p_auth_result = 'INVALID_PASSWORD';
            SET p_user_id = NULL;
        END IF;
    END IF;
END //
DELIMITER ;
```

### 4. パスワード変更とセキュリティ強化

```sql
-- ✅ セキュアなパスワード変更
DELIMITER //
CREATE PROCEDURE ChangePassword(
    IN p_user_id INT,
    IN p_current_password VARCHAR(255),
    IN p_new_password VARCHAR(255)
)
BEGIN
    DECLARE v_current_hash VARCHAR(255);
    DECLARE v_current_salt VARCHAR(64);
    DECLARE v_password_history JSON;
    DECLARE v_new_salt VARCHAR(64);
    DECLARE v_new_hash VARCHAR(255);
    
    -- 現在のパスワード情報取得
    SELECT password_hash, salt, password_history
    INTO v_current_hash, v_current_salt, v_password_history
    FROM secure_users
    WHERE user_id = p_user_id;
    
    -- 現在のパスワード検証（省略）
    
    -- 新しいソルト生成
    SET v_new_salt = SHA2(CONCAT(p_user_id, NOW(), RAND()), 256);
    
    -- パスワード履歴の更新（過去5個のハッシュを保持）
    SET v_password_history = JSON_ARRAY_APPEND(
        COALESCE(v_password_history, JSON_ARRAY()),
        '$',
        JSON_OBJECT(
            'hash', v_current_hash,
            'created_at', NOW()
        )
    );
    
    -- 履歴を5個に制限
    IF JSON_LENGTH(v_password_history) > 5 THEN
        SET v_password_history = JSON_REMOVE(v_password_history, '$[0]');
    END IF;
    
    -- 新しいパスワードハッシュ（実際はアプリケーション側で生成）
    SET v_new_hash = CONCAT('$2y$12$', v_new_salt, 'new_hash_placeholder');
    
    -- パスワード更新
    UPDATE secure_users
    SET 
        password_hash = v_new_hash,
        salt = v_new_salt,
        password_history = v_password_history,
        last_password_change = NOW(),
        failed_login_attempts = 0,
        account_locked_until = NULL
    WHERE user_id = p_user_id;
    
    SELECT 'Password changed successfully' as result;
END //
DELIMITER ;
```

## セキュリティ強化パターン

### パターン1: 多層防御システム

```sql
-- ✅ 包括的なセキュリティテーブル設計
CREATE TABLE user_security (
    user_id INT PRIMARY KEY,
    
    -- パスワード管理
    password_hash VARCHAR(255) NOT NULL,
    salt VARCHAR(64) NOT NULL,
    hash_algorithm VARCHAR(20) DEFAULT 'argon2id',
    
    -- 2FA (Two-Factor Authentication)
    totp_secret VARCHAR(32),              -- TOTP秘密鍵（暗号化済み）
    backup_codes JSON,                    -- バックアップコード（ハッシュ化済み）
    is_2fa_enabled BOOLEAN DEFAULT FALSE,
    
    -- セッション管理
    session_token_hash VARCHAR(255),
    session_expires_at TIMESTAMP,
    concurrent_sessions_limit INT DEFAULT 3,
    
    -- セキュリティポリシー
    password_min_length INT DEFAULT 12,
    require_special_chars BOOLEAN DEFAULT TRUE,
    password_expiry_days INT DEFAULT 90,
    
    -- アクセス制御
    allowed_ip_ranges JSON,              -- 許可IPレンジ
    login_time_restrictions JSON,        -- ログイン時間制限
    
    -- 監査・ログ
    security_events JSON,               -- セキュリティイベント履歴
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- セキュリティイベントログテーブル
CREATE TABLE security_audit_log (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    event_type ENUM('login_success', 'login_fail', 'password_change', 'account_lock', '2fa_enable', '2fa_disable', 'suspicious_activity'),
    ip_address VARCHAR(45),
    user_agent TEXT,
    details JSON,
    risk_score INT,                     -- リスクスコア（0-100）
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_events (user_id, created_at),
    INDEX idx_event_type (event_type, created_at),
    INDEX idx_risk_score (risk_score, created_at)
);
```

### パターン2: パスワード強度管理

```sql
-- ✅ パスワード強度チェック
DELIMITER //
CREATE FUNCTION CheckPasswordStrength(p_password VARCHAR(255))
RETURNS JSON
READS SQL DATA
DETERMINISTIC
BEGIN
    DECLARE v_score INT DEFAULT 0;
    DECLARE v_feedback JSON DEFAULT JSON_ARRAY();
    
    -- 長さチェック
    IF LENGTH(p_password) >= 12 THEN
        SET v_score = v_score + 25;
    ELSE
        SET v_feedback = JSON_ARRAY_APPEND(v_feedback, '$', 'Password must be at least 12 characters long');
    END IF;
    
    -- 大文字チェック
    IF p_password REGEXP '[A-Z]' THEN
        SET v_score = v_score + 15;
    ELSE
        SET v_feedback = JSON_ARRAY_APPEND(v_feedback, '$', 'Include at least one uppercase letter');
    END IF;
    
    -- 小文字チェック
    IF p_password REGEXP '[a-z]' THEN
        SET v_score = v_score + 15;
    ELSE
        SET v_feedback = JSON_ARRAY_APPEND(v_feedback, '$', 'Include at least one lowercase letter');
    END IF;
    
    -- 数字チェック
    IF p_password REGEXP '[0-9]' THEN
        SET v_score = v_score + 15;
    ELSE
        SET v_feedback = JSON_ARRAY_APPEND(v_feedback, '$', 'Include at least one number');
    END IF;
    
    -- 特殊文字チェック
    IF p_password REGEXP '[!@#$%^&*(),.?":{}|<>]' THEN
        SET v_score = v_score + 20;
    ELSE
        SET v_feedback = JSON_ARRAY_APPEND(v_feedback, '$', 'Include at least one special character');
    END IF;
    
    -- 一般的なパスワードチェック（簡易版）
    IF p_password IN ('password', '123456', 'admin', 'qwerty') THEN
        SET v_score = 0;
        SET v_feedback = JSON_ARRAY_APPEND(v_feedback, '$', 'Password is too common');
    END IF;
    
    RETURN JSON_OBJECT(
        'score', v_score,
        'strength', CASE 
            WHEN v_score >= 80 THEN 'Very Strong'
            WHEN v_score >= 60 THEN 'Strong'  
            WHEN v_score >= 40 THEN 'Medium'
            WHEN v_score >= 20 THEN 'Weak'
            ELSE 'Very Weak'
        END,
        'feedback', v_feedback
    );
END //
DELIMITER ;

-- 使用例
SELECT CheckPasswordStrength('MySecureP@ssw0rd2024!') as password_analysis;
```

### パターン3: 侵害検知システム

```sql
-- ✅ 異常活動検知
CREATE TABLE threat_detection (
    detection_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    threat_type ENUM('brute_force', 'unusual_location', 'unusual_time', 'multiple_failures', 'suspicious_ip'),
    risk_level ENUM('low', 'medium', 'high', 'critical'),
    details JSON,
    auto_response ENUM('none', 'alert', 'lock_account', 'require_2fa'),
    resolved_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_threats (user_id, created_at),
    INDEX idx_risk_level (risk_level, created_at)
);

-- 異常ログイン検知トリガー
DELIMITER //
CREATE TRIGGER detect_suspicious_login
AFTER INSERT ON security_audit_log
FOR EACH ROW
BEGIN
    DECLARE v_recent_failures INT DEFAULT 0;
    DECLARE v_different_locations INT DEFAULT 0;
    
    -- ブルートフォース攻撃検知
    IF NEW.event_type = 'login_fail' THEN
        SELECT COUNT(*) INTO v_recent_failures
        FROM security_audit_log
        WHERE user_id = NEW.user_id
          AND event_type = 'login_fail'
          AND created_at >= DATE_SUB(NOW(), INTERVAL 15 MINUTE);
        
        IF v_recent_failures >= 5 THEN
            INSERT INTO threat_detection (user_id, threat_type, risk_level, details, auto_response)
            VALUES (
                NEW.user_id,
                'brute_force',
                'high',
                JSON_OBJECT('failure_count', v_recent_failures, 'ip_address', NEW.ip_address),
                'lock_account'
            );
            
            -- アカウントロック
            UPDATE secure_users 
            SET account_locked_until = DATE_ADD(NOW(), INTERVAL 1 HOUR)
            WHERE user_id = NEW.user_id;
        END IF;
    END IF;
    
    -- 異常な地理的位置からのアクセス検知（簡易版）
    IF NEW.event_type = 'login_success' THEN
        -- 過去の正常なログイン場所と比較（実装省略）
        -- 地理的情報は外部サービスと連携
    END IF;
END //
DELIMITER ;
```

## まとめ

### リーダブルパスワードアンチパターンの問題点
1. **データ漏洩時の壊滅的被害**: 全パスワードが平文で露出
2. **内部不正のリスク**: 管理者・開発者による悪用可能性
3. **法規制違反**: GDPR、個人情報保護法等への抵触
4. **信頼失墜**: 企業・サービスの評判損失

### 適切なハッシュ化による利点
1. **データ保護**: 漏洩時も原パスワードが判明しない
2. **内部不正防止**: 管理者でもパスワードを知ることができない
3. **コンプライアンス対応**: セキュリティ基準への準拠
4. **信頼性向上**: ユーザーの安心・信頼獲得

**重要**: パスワードの平文保存は絶対に避けるべきです。bcrypt、Argon2、scrypt等の適切なハッシュアルゴリズムを使用し、ソルトと組み合わせた堅牢なセキュリティを実装してください。

---

# 第21章 シュードキー・ニートフリーク（疑似キー潔癖症）

## 問題と解決策

### ❌ 問題：主キーの欠番を埋める問題

```sql
-- ❌ 悪い例：欠番を埋めようとする処理
CREATE TABLE products (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- データ挿入
INSERT INTO products (product_name, price) VALUES 
('Product A', 100.00),
('Product B', 200.00),
('Product C', 300.00);
-- 結果: product_id = 1, 2, 3

-- product_id = 2 を削除
DELETE FROM products WHERE product_id = 2;
-- 結果: product_id = 1, 3 (欠番発生)

-- ❌ 欠番を埋めようとする危険な処理
-- 新しい商品を挿入時に欠番(2)を使用しようとする
SET @next_id = (
    SELECT MIN(t1.product_id + 1) 
    FROM products t1 
    LEFT JOIN products t2 ON t1.product_id + 1 = t2.product_id 
    WHERE t2.product_id IS NULL
);

INSERT INTO products (product_id, product_name, price) 
VALUES (@next_id, 'Product D', 400.00);
```

**主な問題**：
- 一意性保証の複雑化
- 同時実行時の競合状態
- 既存参照の破綻リスク
- パフォーマンスの劣化

### ✅ 解決策：欠番の許容

```sql
-- ✅ 正しい例：自然な主キー生成を許容
CREATE TABLE products (
    product_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- シンプルな挿入（自動採番に任せる）
INSERT INTO products (product_name, price) VALUES 
('Product A', 100.00),
('Product B', 200.00),
('Product C', 300.00);

-- 削除後も自然に継続
DELETE FROM products WHERE product_id = 2;

-- 新規挿入（欠番を気にしない）
INSERT INTO products (product_name, price) VALUES 
('Product D', 400.00);
-- 結果: product_id = 4（欠番は無視）
```

## シュードキー・ニートフリークアンチパターンとは？

主キー（特に人工的な代理キー）の欠番を「汚い」「不完全」と感じ、欠番を埋めようとすることで、システムの複雑性とリスクを増大させるアンチパターンです。

## ❌ 問題点の詳細

### 1. 同時実行時の競合状態

```sql
-- ❌ 欠番検索・挿入での競合条件
-- セッション1とセッション2が同時実行される場合

-- 両セッションが同じ欠番を検出
-- セッション1:
SELECT MIN(p1.product_id + 1) as next_available
FROM products p1
LEFT JOIN products p2 ON p1.product_id + 1 = p2.product_id
WHERE p2.product_id IS NULL;
-- 結果: 2 (欠番)

-- セッション2（同時実行）:
SELECT MIN(p1.product_id + 1) as next_available  
FROM products p1
LEFT JOIN products p2 ON p1.product_id + 1 = p2.product_id
WHERE p2.product_id IS NULL;
-- 結果: 2 (同じ欠番)

-- 両セッションが同じIDで挿入を試行
-- セッション1:
INSERT INTO products (product_id, product_name, price) 
VALUES (2, 'Product X', 100.00);  -- 成功

-- セッション2:
INSERT INTO products (product_id, product_name, price) 
VALUES (2, 'Product Y', 200.00);  -- ERROR: Duplicate entry '2'

-- 問題:
-- - デッドロック発生の可能性
-- - 重複キーエラーのハンドリング必要
-- - 再試行ロジックの実装必要
-- - アプリケーションの複雑化
```

### 2. 既存参照関係の破綻リスク

```sql
-- ❌ 外部キー参照がある場合の危険性
CREATE TABLE categories (
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    category_name VARCHAR(100)
);

CREATE TABLE products (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(100),
    category_id INT,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

CREATE TABLE order_items (
    item_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT,
    quantity INT,
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- 初期データ
INSERT INTO categories VALUES (1, 'Electronics'), (2, 'Books'), (3, 'Clothing');
INSERT INTO products VALUES (1, 'Laptop', 1), (2, 'Novel', 2), (3, 'T-Shirt', 3);
INSERT INTO order_items VALUES (1, 2, 5);  -- Novel を5個注文

-- category_id = 2 (Books) を削除
DELETE FROM categories WHERE category_id = 2;
-- これに伴い product_id = 2 (Novel) も削除される（カスケード削除）

-- 欠番(2)を埋めようとして新しいカテゴリを挿入
INSERT INTO categories (category_id, category_name) VALUES (2, 'Sports');

-- しかし order_items テーブルの参照が残っている場合
-- item_id = 1 は存在しない product_id = 2 を参照
-- データ整合性の問題が発生
```

### 3. パフォーマンス問題

```sql
-- ❌ 欠番検索の高コスト処理
-- 100万件のテーブルで欠番を検索
SELECT MIN(t1.user_id + 1) as next_gap
FROM users t1
LEFT JOIN users t2 ON t1.user_id + 1 = t2.user_id  
WHERE t2.user_id IS NULL
  AND t1.user_id < (SELECT MAX(user_id) FROM users);

-- 実行計画分析:
-- 1. users テーブル全体スキャン (100万行)
-- 2. 自己結合による N×N 比較
-- 3. MAX値の別途計算
-- 実行時間: 約15秒 (大規模テーブル)

-- ❌ さらに悪い例: 全ての欠番を検索
WITH RECURSIVE missing_ids AS (
    SELECT 1 as id
    UNION ALL
    SELECT id + 1 FROM missing_ids 
    WHERE id < (SELECT MAX(user_id) FROM users)
)
SELECT m.id as missing_id
FROM missing_ids m
LEFT JOIN users u ON m.id = u.user_id
WHERE u.user_id IS NULL;

-- 問題:
-- - 再帰CTEによる高負荷
-- - メモリ使用量の増大  
-- - ロック時間の延長
```

### 4. 業務ロジックとの不整合

```sql
-- ❌ 時系列データでの問題
CREATE TABLE daily_reports (
    report_id INT AUTO_INCREMENT PRIMARY KEY,
    report_date DATE NOT NULL UNIQUE,
    sales_amount DECIMAL(15,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 日次レポートデータ
INSERT INTO daily_reports (report_date, sales_amount) VALUES
('2024-01-01', 10000.00),  -- report_id = 1
('2024-01-02', 15000.00),  -- report_id = 2  
('2024-01-03', 12000.00);  -- report_id = 3

-- 1月2日のデータに問題があり削除
DELETE FROM daily_reports WHERE report_date = '2024-01-02';

-- 欠番(2)を埋めて新しいデータを挿入
INSERT INTO daily_reports (report_id, report_date, sales_amount) 
VALUES (2, '2024-01-04', 18000.00);

-- 問題:
-- - report_id の順序と日付の順序が不一致
-- - 時系列分析での混乱
-- - 監査ログとの不整合
-- ORDER BY report_id と ORDER BY report_date の結果が異なる
```

## ✅ 欠番の許容による解決

### 1. シンプルな主キー設計

```sql
-- ✅ 自然な主キー生成
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- 十分な範囲
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_username (username),
    INDEX idx_email (email)
);

-- シンプルな挿入
INSERT INTO users (username, email) VALUES 
('john_doe', 'john@example.com'),
('jane_smith', 'jane@example.com');

-- 削除しても欠番は気にしない
DELETE FROM users WHERE username = 'john_doe';

-- 新規ユーザー（欠番無視）
INSERT INTO users (username, email) VALUES 
('mike_johnson', 'mike@example.com');  -- user_id = 3
```

### 2. UUID使用による根本的解決

```sql
-- ✅ UUIDによる主キー（欠番概念が不要）
CREATE TABLE orders (
    order_id CHAR(36) PRIMARY KEY DEFAULT (UUID()),
    user_id BIGINT NOT NULL,
    order_date DATE NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_orders (user_id, order_date),
    INDEX idx_order_date (order_date)
);

-- UUID自動生成での挿入
INSERT INTO orders (user_id, order_date, total_amount) VALUES
(123, '2024-03-15', 2500.00);  -- order_id は自動生成される

-- 削除しても「欠番」の概念がない
DELETE FROM orders WHERE order_id = 'f47ac10b-58cc-4372-a567-0e02b2c3d479';

-- 利点:
-- - グローバル一意性
-- - 欠番の概念が存在しない
-- - 分散システムでの安全性
-- - マージ・レプリケーションが容易
```

### 3. ビジネス識別子の分離

```sql
-- ✅ 代理キーと自然キーの分離
CREATE TABLE invoices (
    -- 代理キー（システム内部用）
    invoice_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    
    -- ビジネス識別子（ユーザー向け）
    invoice_number VARCHAR(20) NOT NULL UNIQUE,
    
    -- その他のデータ
    customer_id BIGINT NOT NULL,
    issue_date DATE NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    
    INDEX idx_invoice_number (invoice_number),
    INDEX idx_customer_date (customer_id, issue_date)
);

-- ビジネス識別子生成ロジック
DELIMITER //
CREATE FUNCTION GenerateInvoiceNumber(p_date DATE)
RETURNS VARCHAR(20)
READS SQL DATA
DETERMINISTIC
BEGIN
    DECLARE v_sequence INT;
    DECLARE v_year_month CHAR(6);
    
    SET v_year_month = DATE_FORMAT(p_date, '%Y%m');
    
    -- その月の最大連番を取得
    SELECT COALESCE(MAX(
        CAST(SUBSTRING(invoice_number, 8) AS UNSIGNED)
    ), 0) + 1 INTO v_sequence
    FROM invoices
    WHERE invoice_number LIKE CONCAT('INV-', v_year_month, '%');
    
    RETURN CONCAT('INV-', v_year_month, '-', LPAD(v_sequence, 4, '0'));
END //
DELIMITER ;

-- 使用例
INSERT INTO invoices (invoice_number, customer_id, issue_date, total_amount) VALUES
(GenerateInvoiceNumber('2024-03-15'), 123, '2024-03-15', 15000.00);
-- 結果: invoice_number = 'INV-202403-0001'

-- 内部的な invoice_id は欠番があっても問題なし
-- ビジネス側は invoice_number で連番を管理
```

### 4. ソフトデリート導入

```sql
-- ✅ 論理削除による欠番回避
CREATE TABLE products (
    product_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    
    -- ソフトデリート用
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMP NULL,
    deleted_by BIGINT NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_active_products (is_deleted, created_at),
    INDEX idx_product_name (product_name, is_deleted)
);

-- 論理削除の実装
UPDATE products 
SET 
    is_deleted = TRUE,
    deleted_at = NOW(),
    deleted_by = 456  -- 削除実行者のID
WHERE product_id = 123;

-- アクティブな商品のみ取得
SELECT product_id, product_name, price
FROM products
WHERE is_deleted = FALSE
ORDER BY created_at DESC;

-- 利点:
-- - 物理削除による欠番が発生しない
-- - 削除データの復旧が可能
-- - 監査証跡の保持
-- - 参照整合性の維持
```

## 実装パターン

### パターン1: シーケンステーブル管理

```sql
-- ✅ 業務用シーケンス管理（欠番なし保証）
CREATE TABLE business_sequences (
    sequence_name VARCHAR(50) PRIMARY KEY,
    current_value BIGINT NOT NULL DEFAULT 0,
    increment_by INT NOT NULL DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 初期シーケンス設定
INSERT INTO business_sequences (sequence_name, current_value) VALUES
('customer_code', 1000),
('order_number', 10000),
('invoice_number', 100000);

-- 次の値を取得する関数
DELIMITER //
CREATE FUNCTION GetNextSequence(p_sequence_name VARCHAR(50))
RETURNS BIGINT
MODIFIES SQL DATA
BEGIN
    DECLARE v_next_value BIGINT;
    
    UPDATE business_sequences 
    SET current_value = current_value + increment_by
    WHERE sequence_name = p_sequence_name;
    
    SELECT current_value INTO v_next_value
    FROM business_sequences
    WHERE sequence_name = p_sequence_name;
    
    RETURN v_next_value;
END //
DELIMITER ;

-- 使用例（欠番なし保証）
INSERT INTO customers (customer_code, customer_name) VALUES
(CONCAT('CUST', LPAD(GetNextSequence('customer_code'), 6, '0')), 'ABC Company');
-- 結果: customer_code = 'CUST001001'
```

### パターン2: 階層的ID管理

```sql
-- ✅ 部門別・年月別の階層ID
CREATE TABLE document_registry (
    document_id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- システム内部ID
    department_code CHAR(3) NOT NULL,
    year_month CHAR(6) NOT NULL,
    sequence_number INT NOT NULL,
    document_title VARCHAR(200) NOT NULL,
    
    -- 複合一意制約
    UNIQUE KEY uk_dept_yearmonth_seq (department_code, year_month, sequence_number),
    
    INDEX idx_dept_yearmonth (department_code, year_month)
);

-- 階層ID生成プロシージャ
DELIMITER //
CREATE PROCEDURE CreateDocument(
    IN p_dept_code CHAR(3),
    IN p_title VARCHAR(200),
    OUT p_document_id BIGINT,
    OUT p_document_number VARCHAR(20)
)
BEGIN
    DECLARE v_year_month CHAR(6);
    DECLARE v_sequence INT;
    
    SET v_year_month = DATE_FORMAT(NOW(), '%Y%m');
    
    -- その部門・月の最大連番を取得
    SELECT COALESCE(MAX(sequence_number), 0) + 1 INTO v_sequence
    FROM document_registry
    WHERE department_code = p_dept_code 
      AND year_month = v_year_month;
    
    INSERT INTO document_registry 
    (department_code, year_month, sequence_number, document_title)
    VALUES (p_dept_code, v_year_month, v_sequence, p_title);
    
    SET p_document_id = LAST_INSERT_ID();
    SET p_document_number = CONCAT(p_dept_code, '-', v_year_month, '-', LPAD(v_sequence, 3, '0'));
END //
DELIMITER ;

-- 使用例
CALL CreateDocument('HR', 'Employee Handbook', @doc_id, @doc_number);
-- 結果: @doc_number = 'HR-202403-001'
```

### パターン3: 分散ID生成

```sql
-- ✅ 分散環境でのID競合回避
CREATE TABLE id_generators (
    generator_name VARCHAR(50) PRIMARY KEY,
    node_id INT NOT NULL,                    -- サーバーノードID (1-1023)
    sequence BIGINT NOT NULL DEFAULT 0,     -- 連番 (0-4095)
    last_timestamp BIGINT NOT NULL,         -- 最後の生成時刻
    
    INDEX idx_node (node_id)
);

-- Snowflake型ID生成関数
DELIMITER //
CREATE FUNCTION GenerateSnowflakeId(p_node_id INT)
RETURNS BIGINT
MODIFIES SQL DATA
BEGIN
    DECLARE v_timestamp BIGINT;
    DECLARE v_last_timestamp BIGINT;
    DECLARE v_sequence BIGINT;
    DECLARE v_snowflake_id BIGINT;
    
    -- 現在時刻（ミリ秒、カスタムエポックからの経過時間）
    SET v_timestamp = UNIX_TIMESTAMP() * 1000 - 1640995200000;  -- 2022-01-01からの経過
    
    -- 前回の状態を取得・更新
    SELECT last_timestamp, sequence INTO v_last_timestamp, v_sequence
    FROM id_generators 
    WHERE generator_name = 'snowflake' AND node_id = p_node_id
    FOR UPDATE;
    
    -- 同一ミリ秒内での連番管理
    IF v_timestamp = v_last_timestamp THEN
        SET v_sequence = v_sequence + 1;
        IF v_sequence >= 4096 THEN
            -- シーケンス番号が上限に達した場合は次のミリ秒まで待機
            SET v_timestamp = v_timestamp + 1;
            SET v_sequence = 0;
        END IF;
    ELSE
        SET v_sequence = 0;
    END IF;
    
    -- 状態更新
    UPDATE id_generators 
    SET last_timestamp = v_timestamp, sequence = v_sequence
    WHERE generator_name = 'snowflake' AND node_id = p_node_id;
    
    -- Snowflake ID構築 (64bit)
    -- [1bit予約][41bitタイムスタンプ][10bitノードID][12bitシーケンス]
    SET v_snowflake_id = 
        (v_timestamp << 22) |       -- タイムスタンプ (41bit)
        (p_node_id << 12) |         -- ノードID (10bit)  
        v_sequence;                 -- シーケンス (12bit)
    
    RETURN v_snowflake_id;
END //
DELIMITER ;
```

## まとめ

### シュードキー・ニートフリークアンチパターンの問題点
1. **複雑性の増大**: 欠番検索・埋め込み処理の複雑化
2. **競合状態**: 同時実行時のデッドロック・重複エラー
3. **パフォーマンス劣化**: 高コストな欠番検索処理
4. **データ整合性リスク**: 既存参照関係の破綻可能性

### 欠番許容による利点
1. **シンプルさ**: 自動採番機能の自然な利用
2. **高性能**: 不要な検索処理の排除
3. **高い並行性**: 競合状態の回避
4. **安定性**: 予期しないデータ破損の防止

**重要**: 代理キー（サロゲートキー）の欠番は技術的に問題ありません。ビジネス要件で連番が必要な場合は、別途ビジネス識別子を設計し、システム内部のIDとは分離して管理してください。