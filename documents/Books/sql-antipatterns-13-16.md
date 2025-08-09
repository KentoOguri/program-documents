# SQLアンチパターン 第13章〜第16章

## 目次
1. [第13章 フィア・オブ・ジ・アンノウン（恐怖のunknown）](#第13章-フィア・オブ・ジ・アンノウンknown)
2. [第14章 アンビギュアスグループ（曖昧なグループ）](#第14章-アンビギュアスグループ曖昧なグループ)
3. [第15章 ランダムセレクション](#第15章-ランダムセレクション)
4. [第16章 プアマンズ・サーチエンジン（貧者のサーチエンジン）](#第16章-プアマンズサーチエンジン貧者のサーチエンジン)

---

# 第13章 フィア・オブ・ジ・アンノウン（恐怖のunknown）

## 問題と解決策

### ❌ 問題：NULL値の不適切な扱い

```sql
-- ❌ 悪い例：NULL値を避けようとする設計
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    middle_name VARCHAR(50) NOT NULL DEFAULT '',  -- NULL回避で空文字
    birth_date DATE NOT NULL DEFAULT '1900-01-01', -- NULL回避でダミー値
    salary DECIMAL(10,2) NOT NULL DEFAULT -1       -- NULL回避で-1
);

-- ❌ 問題のあるクエリ
SELECT AVG(salary) FROM users WHERE salary != -1;  -- -1を除外する必要
SELECT COUNT(*) FROM users WHERE middle_name != ''; -- 空文字チェック
```

**主な問題**：
- ダミー値による計算結果の歪み
- 意味のないデフォルト値による混乱
- 不必要な条件分岐の増加
- データの意味が不明確

### ✅ 解決策：NULL値の正しい処理方法

```sql
-- ✅ 正しい例：適切なNULL使用
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    middle_name VARCHAR(50) NULL,           -- ミドルネームは任意
    birth_date DATE NULL,                   -- 生年月日は不明の場合あり
    salary DECIMAL(10,2) NULL               -- 給与情報は非公開の場合あり
);

-- ✅ 正しいNULL処理
SELECT AVG(salary) FROM users;              -- NULLは自動的に除外
SELECT COUNT(*) FROM users WHERE middle_name IS NOT NULL; -- 適切なNULL判定
```

## フィア・オブ・ジ・アンノウンアンチパターンとは？

NULL値を「悪いもの」として避け、ダミー値や空文字で代替することで、かえってデータの整合性や処理の複雑さを増してしまう設計パターンです。

## ❌ 問題点の詳細

### 1. ダミー値による計算エラー

```sql
-- ❌ ダミー値による統計の歪み
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    salary DECIMAL(10,2) NOT NULL DEFAULT -1,  -- 未設定は-1
    bonus DECIMAL(10,2) NOT NULL DEFAULT 0     -- 未設定は0
);

INSERT INTO employees VALUES
(1, 50000.00, 5000.00),
(2, -1, 0),         -- 給与未設定
(3, 60000.00, 0),   -- ボーナス本当に0
(4, -1, 0);         -- 両方未設定

-- ❌ 間違った統計結果
SELECT 
    AVG(salary) as avg_salary,      -- -1が計算に含まれる
    SUM(bonus) as total_bonus,      -- 0とNULLの区別ができない
    MIN(salary) as min_salary       -- -1が最小値になる
FROM employees;
-- 結果: avg_salary = 27249.75 (実際は55000), min_salary = -1

-- ✅ 正しい設計での統計
CREATE TABLE employees_correct (
    emp_id INT PRIMARY KEY,
    salary DECIMAL(10,2) NULL,     -- 未設定はNULL
    bonus DECIMAL(10,2) NULL       -- 未設定はNULL
);

SELECT 
    AVG(salary) as avg_salary,      -- NULLは自動除外
    SUM(COALESCE(bonus, 0)) as total_bonus, -- NULL=0として計算
    MIN(salary) as min_salary       -- NULLは除外
FROM employees_correct;
-- 結果: avg_salary = 55000 (正確), min_salary = 50000
```

### 2. 空文字による検索問題

```sql
-- ❌ 空文字での代替による問題
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    middle_name VARCHAR(50) NOT NULL DEFAULT '',
    last_name VARCHAR(50) NOT NULL,
    phone VARCHAR(20) NOT NULL DEFAULT 'UNKNOWN'
);

-- ❌ 検索時の混乱
-- 「ミドルネームがある顧客」を検索したい
SELECT * FROM customers WHERE middle_name != '';
-- → 空文字と実際の値の区別が曖昧

-- 「電話番号がわかっている顧客」を検索
SELECT * FROM customers WHERE phone != 'UNKNOWN';
-- → 'UNKNOWN'という実際の値との区別ができない

-- ✅ 正しい設計
CREATE TABLE customers_correct (
    customer_id INT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    middle_name VARCHAR(50) NULL,    -- ないならNULL
    last_name VARCHAR(50) NOT NULL,
    phone VARCHAR(20) NULL           -- 不明ならNULL
);

-- ✅ 明確な検索
SELECT * FROM customers_correct WHERE middle_name IS NOT NULL;
SELECT * FROM customers_correct WHERE phone IS NOT NULL;
```

### 3. 三値論理の回避による複雑さ

```sql
-- ❌ NULLを避けることで生じる複雑性
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL DEFAULT -1,     -- 価格未定は-1
    discount_rate DECIMAL(5,2) NOT NULL DEFAULT -1, -- 割引なしは-1
    is_discontinued CHAR(1) NOT NULL DEFAULT 'U' -- Unknown
);

-- ❌ 複雑な条件分岐
SELECT 
    name,
    CASE 
        WHEN price = -1 THEN 'Price TBD'
        ELSE CONCAT('$', price)
    END as display_price,
    CASE 
        WHEN discount_rate = -1 THEN 'No discount'
        ELSE CONCAT(discount_rate, '%')
    END as discount_info,
    CASE
        WHEN is_discontinued = 'Y' THEN 'Discontinued'
        WHEN is_discontinued = 'N' THEN 'Active'
        ELSE 'Status Unknown'
    END as status
FROM products;

-- ✅ NULLを使った簡潔な表現
CREATE TABLE products_correct (
    product_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NULL,
    discount_rate DECIMAL(5,2) NULL,
    is_discontinued BOOLEAN NULL
);

SELECT 
    name,
    COALESCE(CONCAT('$', price), 'Price TBD') as display_price,
    COALESCE(CONCAT(discount_rate, '%'), 'No discount') as discount_info,
    CASE is_discontinued
        WHEN TRUE THEN 'Discontinued'
        WHEN FALSE THEN 'Active'
        ELSE 'Status Unknown'
    END as status
FROM products_correct;
```

## ✅ NULL値の正しい処理方法

### 1. 集約関数での適切なNULL処理

```sql
-- ✅ 集約関数はNULLを自動的に除外
CREATE TABLE sales (
    sale_id INT PRIMARY KEY,
    amount DECIMAL(10,2) NULL,    -- 返品等でNULLの場合あり
    commission DECIMAL(8,2) NULL, -- 歩合なしの場合はNULL
    sale_date DATE NOT NULL
);

-- 正しい統計計算
SELECT 
    COUNT(*) as total_records,           -- 全レコード数
    COUNT(amount) as valid_sales,        -- NULLでない売上数
    AVG(amount) as avg_sale_amount,      -- NULLは除外して平均
    SUM(COALESCE(commission, 0)) as total_commission -- NULLを0として合計
FROM sales;
```

### 2. COALESCE関数による代替値指定

```sql
-- ✅ COALESCEで適切なデフォルト値設定
SELECT 
    customer_id,
    first_name,
    COALESCE(middle_name, '') as middle_name_display,
    last_name,
    COALESCE(phone, 'No phone on file') as phone_display,
    COALESCE(email, 'No email on file') as email_display
FROM customers;

-- ✅ 複数の代替候補
SELECT 
    employee_id,
    COALESCE(work_phone, mobile_phone, home_phone, 'No contact') as contact_phone
FROM employees;
```

### 3. NULL安全な比較

```sql
-- ✅ NULL安全な等価比較
-- MySQL
SELECT * FROM users 
WHERE middle_name <=> @search_middle_name;  -- NULLでも正しく比較

-- 標準SQL
SELECT * FROM users 
WHERE middle_name IS NOT DISTINCT FROM @search_middle_name;

-- ✅ CASE式でのNULL処理
SELECT 
    user_id,
    CASE 
        WHEN middle_name IS NULL THEN first_name || ' ' || last_name
        ELSE first_name || ' ' || middle_name || ' ' || last_name
    END as full_name
FROM users;
```

## 実装パターン

### パターン1: オプショナル属性の管理

```sql
-- ✅ ユーザープロフィール管理
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    
    -- 必須でない情報はNULLを許可
    first_name VARCHAR(50) NULL,
    last_name VARCHAR(50) NULL,
    birth_date DATE NULL,
    phone VARCHAR(20) NULL,
    bio TEXT NULL,
    avatar_url VARCHAR(500) NULL,
    
    -- アカウント状態（明確な3状態）
    email_verified BOOLEAN NULL,  -- NULL=未確認, TRUE=確認済み, FALSE=確認失敗
    
    -- タイムスタンプ
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP NULL     -- ログインしたことがない場合はNULL
);

-- プロフィール完成度の計算
SELECT 
    user_id,
    username,
    (
        (CASE WHEN first_name IS NOT NULL THEN 1 ELSE 0 END) +
        (CASE WHEN last_name IS NOT NULL THEN 1 ELSE 0 END) +
        (CASE WHEN birth_date IS NOT NULL THEN 1 ELSE 0 END) +
        (CASE WHEN phone IS NOT NULL THEN 1 ELSE 0 END) +
        (CASE WHEN bio IS NOT NULL THEN 1 ELSE 0 END) +
        (CASE WHEN avatar_url IS NOT NULL THEN 1 ELSE 0 END)
    ) * 100 / 6 as profile_completion_percentage
FROM user_profiles;
```

### パターン2: 外部キー関係での適切なNULL使用

```sql
-- ✅ オプショナルな関係性
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT NOT NULL,
    
    -- 割引クーポンは任意
    coupon_id INT NULL,
    
    -- 配送先は注文者と異なる場合のみ
    shipping_address_id INT NULL,
    
    order_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (coupon_id) REFERENCES coupons(coupon_id),
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(address_id)
);

-- クーポン使用状況の分析
SELECT 
    COUNT(*) as total_orders,
    COUNT(coupon_id) as orders_with_coupon,
    COUNT(coupon_id) * 100.0 / COUNT(*) as coupon_usage_rate
FROM orders;
```

### パターン3: 段階的データ収集

```sql
-- ✅ 段階的にデータを収集するフォーム
CREATE TABLE job_applications (
    application_id INT PRIMARY KEY,
    
    -- ステップ1: 基本情報（必須）
    email VARCHAR(100) NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    
    -- ステップ2: 職歴（任意、後で入力可能）
    current_company VARCHAR(100) NULL,
    current_position VARCHAR(100) NULL,
    experience_years INT NULL,
    
    -- ステップ3: 詳細情報（任意）
    cover_letter TEXT NULL,
    resume_file_id INT NULL,
    expected_salary DECIMAL(10,2) NULL,
    
    -- ステップ4: 面接設定（後で設定）
    interview_date TIMESTAMP NULL,
    interviewer_id INT NULL,
    
    -- 進捗管理
    application_step ENUM('basic', 'experience', 'details', 'interview', 'complete') DEFAULT 'basic',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 完了率の分析
SELECT 
    application_step,
    COUNT(*) as count,
    AVG(CASE WHEN current_company IS NOT NULL THEN 1 ELSE 0 END) as company_completion_rate,
    AVG(CASE WHEN cover_letter IS NOT NULL THEN 1 ELSE 0 END) as cover_letter_completion_rate
FROM job_applications
GROUP BY application_step;
```

### パターン4: 履歴データでのNULL活用

```sql
-- ✅ 価格履歴管理（終了日がNULLの場合は現在有効）
CREATE TABLE price_history (
    price_id INT PRIMARY KEY,
    product_id INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    effective_from DATE NOT NULL,
    effective_until DATE NULL,     -- NULLの場合は現在も有効
    
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    
    -- 同一商品で有効期間が重複しないように
    INDEX idx_product_dates (product_id, effective_from, effective_until)
);

-- 現在有効な価格の取得
SELECT p.product_name, ph.price, ph.effective_from
FROM products p
JOIN price_history ph ON p.product_id = ph.product_id
WHERE ph.effective_until IS NULL;

-- 特定日時点での価格取得
SET @target_date = '2024-03-15';
SELECT p.product_name, ph.price
FROM products p
JOIN price_history ph ON p.product_id = ph.product_id
WHERE ph.effective_from <= @target_date
  AND (ph.effective_until IS NULL OR ph.effective_until > @target_date);
```

## NULL処理のベストプラクティス

### 1. インデックス設計での考慮

```sql
-- ✅ NULLを含むインデックスの注意点
-- NULLは通常のインデックスに含まれない場合が多い

-- 部分インデックス（PostgreSQL）
CREATE INDEX idx_active_users ON users(last_login) WHERE last_login IS NOT NULL;

-- 関数インデックス（MySQL）
CREATE INDEX idx_has_phone ON users((phone IS NOT NULL));

-- NULLs FIRST/LAST での並び替え（PostgreSQL, Oracle）
SELECT * FROM users ORDER BY last_login DESC NULLS LAST;
```

### 2. 制約設計

```sql
-- ✅ CHECK制約でのビジネスルール
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    full_time BOOLEAN NOT NULL,
    hourly_rate DECIMAL(8,2) NULL,
    annual_salary DECIMAL(10,2) NULL,
    
    -- ビジネスルール: フルタイムは年俸、パートタイムは時給
    CONSTRAINT chk_salary_type CHECK (
        (full_time = TRUE AND annual_salary IS NOT NULL AND hourly_rate IS NULL) OR
        (full_time = FALSE AND hourly_rate IS NOT NULL AND annual_salary IS NULL)
    )
);
```

### 3. アプリケーション層での処理

```sql
-- ✅ ビューでのNULL処理の集約
CREATE VIEW user_display_view AS
SELECT 
    user_id,
    username,
    COALESCE(
        CONCAT(first_name, ' ', middle_name, ' ', last_name),
        CONCAT(first_name, ' ', last_name),
        username
    ) as display_name,
    
    COALESCE(phone, email, 'No contact info') as primary_contact,
    
    CASE 
        WHEN last_login IS NULL THEN 'Never logged in'
        WHEN last_login < DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 'Inactive'
        ELSE 'Active'
    END as activity_status,
    
    birth_date IS NOT NULL as has_birth_date,
    phone IS NOT NULL as has_phone
    
FROM users;
```

## まとめ

### フィア・オブ・ジ・アンノウンアンチパターンの問題点
1. **統計の歪み**: ダミー値による集計結果の不正確性
2. **検索の複雑化**: 不要な条件分岐の発生
3. **意味の曖昧性**: 実際の値とダミー値の区別困難
4. **メンテナンス困難**: 複雑な条件処理の増加

### NULL値活用による利点
1. **明確な意味**: 「未設定」「不明」の明確な表現
2. **自動処理**: 集約関数での適切な除外
3. **シンプルな条件**: IS NULL/IS NOT NULL による明確な判定
4. **データ整合性**: ビジネスルールの適切な表現

**重要**: NULL値は「悪いもの」ではありません。適切に使用することで、データの意味を明確に表現し、処理を簡潔にできます。

---

# 第14章 アンビギュアスグループ（曖昧なグループ）

## 問題と解決策

### ❌ 問題：GROUP BY時の非グループ化列参照

```sql
-- ❌ 悪い例：非決定的な結果
CREATE TABLE sales (
    sale_id INT PRIMARY KEY,
    salesperson_id INT,
    customer_id INT,
    amount DECIMAL(10,2),
    sale_date DATE
);

-- ❌ 問題のあるクエリ（MySQLのSQL_MODE=ONLYFULLGROUPBYで実行)
SELECT 
    salesperson_id,
    customer_id,        -- GROUP BYに含まれていない列
    MAX(amount) as max_sale,
    COUNT(*) as sale_count
FROM sales
GROUP BY salesperson_id;
-- エラーまたは予期しない結果
```

**主な問題**：
- 非決定的な結果
- SQLの標準規則違反
- 予期しない結果による業務エラー
- ポータビリティの問題

### ✅ 解決策：ウィンドウ関数、適切な集約

```sql
-- ✅ 正しい例：ウィンドウ関数使用
-- 各営業担当者の最大売上額とその詳細を取得
SELECT 
    salesperson_id,
    customer_id,
    amount,
    sale_date,
    RANK() OVER (PARTITION BY salesperson_id ORDER BY amount DESC) as rank_in_group
FROM sales
QUALIFY rank_in_group = 1;

-- または標準的なサブクエリ方式
SELECT s1.*
FROM sales s1
WHERE s1.amount = (
    SELECT MAX(s2.amount)
    FROM sales s2
    WHERE s2.salesperson_id = s1.salesperson_id
);
```

## アンビギュアスグループアンチパターンとは？

GROUP BY句で集約する際に、グループ化されていない列を SELECT 句に含めることで、どの行の値を返すか決まらない（非決定的）な結果を生じさせるアンチパターンです。

## ❌ 問題点の詳細

### 1. 非決定的な結果

```sql
-- ❌ どの顧客IDが返されるか不明
SELECT 
    salesperson_id,
    customer_id,     -- salesperson_idでGROUPしているのに customer_id を取得
    SUM(amount) as total_sales
FROM sales
GROUP BY salesperson_id;

-- 営業担当者ID=1 の場合:
-- - 顧客A, B, C との取引がある
-- - どの顧客IDが結果に含まれるかは実装依存
-- - 実行するたびに異なる結果の可能性
```

### 2. SQL標準違反とポータビリティ問題

```sql
-- ❌ データベースエンジンによる動作の違い

-- MySQL (sql_mode = '' の場合): エラーなし、予期しない結果
-- MySQL (sql_mode = ONLY_FULL_GROUP_BY): エラー
-- PostgreSQL: エラー
-- Oracle: エラー  
-- SQL Server: エラー

SELECT 
    category_id,
    product_name,    -- GROUP BY に含まれていない
    AVG(price) as avg_price
FROM products
GROUP BY category_id;
```

### 3. 業務ロジックエラー

```sql
-- ❌ 危険な集計例
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(10,2),
    shipping_address VARCHAR(500)
);

-- "各顧客の最新注文の配送先を取得したい"
-- ❌ 間違った実装
SELECT 
    customer_id,
    shipping_address,    -- どの注文の住所？
    MAX(order_date) as latest_order_date,
    SUM(total_amount) as total_spent
FROM orders
GROUP BY customer_id;

-- 問題: shipping_address は MAX(order_date) に対応していない可能性
```

### 4. パフォーマンス問題の隠蔽

```sql
-- ❌ 非効率なクエリの隠蔽
-- この書き方だと、実際には全行スキャンが必要だが
-- 一見シンプルに見えるため問題に気づきにくい

SELECT 
    department_id,
    employee_name,   -- 任意の従業員名
    COUNT(*) as employee_count,
    AVG(salary) as avg_salary
FROM employees
GROUP BY department_id;

-- 実際には各部署の全従業員情報が必要
```

## ✅ 適切な解決方法

### 1. ウィンドウ関数による解決

```sql
-- ✅ 各営業担当者の最高売上記録を取得
WITH sales_ranked AS (
    SELECT 
        sale_id,
        salesperson_id,
        customer_id,
        amount,
        sale_date,
        ROW_NUMBER() OVER (
            PARTITION BY salesperson_id 
            ORDER BY amount DESC, sale_date DESC
        ) as rn
    FROM sales
)
SELECT 
    salesperson_id,
    customer_id,
    amount,
    sale_date
FROM sales_ranked
WHERE rn = 1;
```

### 2. 集約関数とJOINの組み合わせ

```sql
-- ✅ 各カテゴリの最高価格商品を取得
-- ステップ1: カテゴリ別最高価格を算出
WITH category_max_price AS (
    SELECT 
        category_id,
        MAX(price) as max_price
    FROM products
    GROUP BY category_id
)
-- ステップ2: 元のテーブルとJOINして詳細取得
SELECT 
    p.category_id,
    p.product_id,
    p.product_name,
    p.price
FROM products p
INNER JOIN category_max_price cmp 
    ON p.category_id = cmp.category_id 
    AND p.price = cmp.max_price;
```

### 3. 相関サブクエリによる解決

```sql
-- ✅ 各部署の最新入社員を取得
SELECT 
    e1.department_id,
    e1.employee_id,
    e1.employee_name,
    e1.hire_date
FROM employees e1
WHERE e1.hire_date = (
    SELECT MAX(e2.hire_date)
    FROM employees e2
    WHERE e2.department_id = e1.department_id
);
```

### 4. GROUP_CONCAT/STRING_AGG による集約

```sql
-- ✅ 適切な集約表示
-- MySQL
SELECT 
    department_id,
    COUNT(*) as employee_count,
    AVG(salary) as avg_salary,
    GROUP_CONCAT(employee_name ORDER BY salary DESC SEPARATOR ', ') as employees,
    MIN(hire_date) as oldest_hire_date,
    MAX(hire_date) as newest_hire_date
FROM employees
GROUP BY department_id;

-- PostgreSQL/SQL Server
SELECT 
    department_id,
    COUNT(*) as employee_count,
    AVG(salary) as avg_salary,
    STRING_AGG(employee_name, ', ' ORDER BY salary DESC) as employees,
    MIN(hire_date) as oldest_hire_date,
    MAX(hire_date) as newest_hire_date
FROM employees
GROUP BY department_id;
```

## 実装パターン

### パターン1: TOP-N クエリ

```sql
-- ✅ 各カテゴリのトップ3商品を取得
SELECT 
    category_id,
    product_name,
    price,
    rank_in_category
FROM (
    SELECT 
        category_id,
        product_name,
        price,
        DENSE_RANK() OVER (
            PARTITION BY category_id 
            ORDER BY price DESC
        ) as rank_in_category
    FROM products
) ranked_products
WHERE rank_in_category <= 3
ORDER BY category_id, rank_in_category;
```

### パターン2: 最新レコードの取得

```sql
-- ✅ 各顧客の最新注文情報
CREATE VIEW latest_orders AS
SELECT 
    o.customer_id,
    o.order_id,
    o.order_date,
    o.total_amount,
    o.status
FROM orders o
WHERE o.order_date = (
    SELECT MAX(o2.order_date)
    FROM orders o2
    WHERE o2.customer_id = o.customer_id
);

-- または、ウィンドウ関数版
CREATE VIEW latest_orders_window AS
SELECT 
    customer_id,
    order_id,
    order_date,
    total_amount,
    status
FROM (
    SELECT 
        customer_id,
        order_id,
        order_date,
        total_amount,
        status,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id 
            ORDER BY order_date DESC
        ) as rn
    FROM orders
) ranked_orders
WHERE rn = 1;
```

### パターン3: 集約レポート

```sql
-- ✅ 部署別業績サマリー
SELECT 
    d.department_name,
    
    -- 基本統計
    COUNT(e.employee_id) as employee_count,
    AVG(e.salary) as avg_salary,
    MIN(e.salary) as min_salary,
    MAX(e.salary) as max_salary,
    
    -- 最高給与従業員の情報（サブクエリで取得）
    (SELECT employee_name 
     FROM employees e2 
     WHERE e2.department_id = d.department_id 
     ORDER BY salary DESC LIMIT 1) as highest_paid_employee,
    
    -- 在籍期間統計
    AVG(DATEDIFF(CURRENT_DATE, e.hire_date)) as avg_tenure_days,
    MIN(e.hire_date) as oldest_hire_date,
    MAX(e.hire_date) as newest_hire_date,
    
    -- 年齢統計（birth_dateがある場合）
    AVG(TIMESTAMPDIFF(YEAR, e.birth_date, CURRENT_DATE)) as avg_age
    
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name
ORDER BY employee_count DESC;
```

### パターン4: 条件付き集約

```sql
-- ✅ 月別売上統計（条件付き集約）
SELECT 
    DATE_FORMAT(sale_date, '%Y-%m') as sale_month,
    
    -- 基本統計
    COUNT(*) as total_sales,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_sale_amount,
    
    -- 条件付きカウント
    COUNT(CASE WHEN amount >= 1000 THEN 1 END) as large_sales_count,
    COUNT(CASE WHEN amount < 100 THEN 1 END) as small_sales_count,
    
    -- 条件付き合計
    SUM(CASE WHEN customer_type = 'premium' THEN amount ELSE 0 END) as premium_revenue,
    SUM(CASE WHEN customer_type = 'regular' THEN amount ELSE 0 END) as regular_revenue,
    
    -- パーセンタイル
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) as median_amount,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY amount) as p95_amount
    
FROM sales
GROUP BY DATE_FORMAT(sale_date, '%Y-%m')
ORDER BY sale_month;
```

## 複雑なケースの解決

### 1. 複数条件での最適値取得

```sql
-- ✅ 各商品カテゴリで最も人気（販売数量が多い）かつ最新の商品
WITH product_stats AS (
    SELECT 
        p.product_id,
        p.category_id,
        p.product_name,
        p.release_date,
        COALESCE(SUM(oi.quantity), 0) as total_sold
    FROM products p
    LEFT JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.product_id, p.category_id, p.product_name, p.release_date
),
ranked_products AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY category_id 
            ORDER BY total_sold DESC, release_date DESC
        ) as popularity_rank
    FROM product_stats
)
SELECT 
    category_id,
    product_name,
    release_date,
    total_sold
FROM ranked_products
WHERE popularity_rank = 1;
```

### 2. 階層データでの集約

```sql
-- ✅ 組織階層での集約（再帰CTEを使用）
WITH RECURSIVE org_hierarchy AS (
    -- トップレベルの部署
    SELECT 
        department_id,
        department_name,
        parent_department_id,
        1 as level
    FROM departments
    WHERE parent_department_id IS NULL
    
    UNION ALL
    
    -- 子部署
    SELECT 
        d.department_id,
        d.department_name,
        d.parent_department_id,
        oh.level + 1
    FROM departments d
    INNER JOIN org_hierarchy oh ON d.parent_department_id = oh.department_id
),
department_stats AS (
    SELECT 
        oh.department_id,
        oh.department_name,
        oh.level,
        COUNT(e.employee_id) as direct_employees,
        SUM(e.salary) as total_salary
    FROM org_hierarchy oh
    LEFT JOIN employees e ON oh.department_id = e.department_id
    GROUP BY oh.department_id, oh.department_name, oh.level
)
SELECT 
    department_name,
    level,
    direct_employees,
    total_salary,
    -- 部署レベルでの累計（子部署含む）
    SUM(total_salary) OVER (
        PARTITION BY SUBSTRING(department_name, 1, 
            CASE WHEN level = 1 THEN LENGTH(department_name) ELSE 1 END)
    ) as hierarchical_total
FROM department_stats
ORDER BY level, department_name;
```

## まとめ

### アンビギュアスグループアンチパターンの問題点
1. **非決定的結果**: 実行ごとに異なる結果の可能性
2. **SQL標準違反**: ポータビリティの欠如
3. **業務ロジックエラー**: 期待した結果と異なる出力
4. **デバッグ困難**: 問題の原因特定が困難

### 適切な解決方法による利点
1. **決定的結果**: 常に同じ条件で同じ結果
2. **標準準拠**: すべてのDBエンジンで動作
3. **明確な意図**: ビジネス要件が明確に表現
4. **保守性**: コードの意図が明確で保守しやすい

**重要**: GROUP BY使用時は、SELECT句に含める列は必ずGROUP BY句に含めるか、集約関数を使用してください。複雑な要件の場合は、ウィンドウ関数や適切なJOINを活用しましょう。

---

# 第15章 ランダムセレクション

## 問題と解決策

### ❌ 問題：ORDER BY RAND()の性能問題

```sql
-- ❌ 悪い例：ORDER BY RAND()
-- 100万件のテーブルから10件をランダム取得
SELECT * FROM products 
ORDER BY RAND() 
LIMIT 10;

-- 問題: 全100万件をメモリに読み込んでソートが発生
-- 実行時間: 数秒〜数分
```

**主な問題**：
- 全件スキャンによる性能劣化
- メモリ使用量の増大
- スケールしない設計
- レプリケーション環境での問題

### ✅ 解決策：効率的なランダム抽出手法

```sql
-- ✅ 正しい例：効率的なランダム抽出
-- 方法1: ランダムID範囲での抽出
SELECT * FROM products 
WHERE product_id >= (
    SELECT FLOOR(RAND() * (SELECT MAX(product_id) FROM products))
) 
LIMIT 10;

-- 方法2: サンプリングテーブル使用
SELECT p.* FROM products p
INNER JOIN (
    SELECT product_id FROM product_sample 
    WHERE RAND() < 0.1  -- 10%サンプリング
    LIMIT 10
) s ON p.product_id = s.product_id;
```

## ランダムセレクションアンチパターンとは？

ORDER BY RAND() (またはRANDOM()) を使用してランダムな結果を取得しようとすることで、大きな性能問題を引き起こすアンチパターンです。

## ❌ 問題点の詳細

### 1. 全件スキャンによる性能劣化

```sql
-- ❌ ORDER BY RAND() の実行計画
EXPLAIN SELECT * FROM large_table ORDER BY RAND() LIMIT 5;
/*
実行計画:
1. 全行を読み込み (Full Table Scan)
2. 各行にRAND()値を付与
3. 全行をメモリ上でソート
4. 上位5件を返却

100万行のテーブルで:
- 読み込み: 100万行
- ソート: O(n log n) = 約2000万回の比較
- メモリ使用量: テーブルサイズ全体
*/
```

### 2. メモリ使用量の爆発的増大

```sql
-- ❌ 大容量テーブルでのRAND()
CREATE TABLE large_products (
    product_id BIGINT PRIMARY KEY,
    product_name VARCHAR(200),
    description TEXT,
    image_data LONGBLOB,  -- 1MBの画像データ
    created_at TIMESTAMP
);

-- 100万件のデータで各レコード平均1.2MB
-- ORDER BY RAND() 実行時:
-- - 必要メモリ: 1.2TB (100万 × 1.2MB)
-- - 実行時間: 数時間
-- - システムクラッシュの可能性

SELECT * FROM large_products ORDER BY RAND() LIMIT 1;
```

### 3. レプリケーション環境での問題

```sql
-- ❌ レプリケーション不整合
-- マスター実行
INSERT INTO random_samples 
SELECT product_id FROM products ORDER BY RAND() LIMIT 1000;

-- 問題:
-- - マスターとスレーブで異なるRAND()値が生成される
-- - レプリケーション不整合が発生
-- - バイナリログで再現不可能
```

### 4. 一貫性のない結果

```sql
-- ❌ 同一クエリで異なる結果
-- 1回目実行
SELECT product_id FROM products ORDER BY RAND() LIMIT 5;
-- 結果: 15, 234, 567, 89, 1000

-- 2回目実行（同じクエリ）
SELECT product_id FROM products ORDER BY RAND() LIMIT 5;
-- 結果: 45, 123, 678, 234, 999

-- 問題: テスト環境で再現困難
-- キャッシュやページング処理で問題発生
```

## ✅ 効率的なランダム抽出手法

### 1. IDベースのランダム抽出

```sql
-- ✅ 連続したIDの場合
CREATE PROCEDURE GetRandomProducts(IN limit_count INT)
BEGIN
    DECLARE max_id INT;
    DECLARE min_id INT;
    
    SELECT MIN(product_id), MAX(product_id) 
    INTO min_id, max_id FROM products;
    
    SELECT * FROM products 
    WHERE product_id >= min_id + FLOOR(RAND() * (max_id - min_id + 1))
    LIMIT limit_count;
END;

-- ✅ 歯抜けIDの場合（削除されたレコードがある）
SELECT p1.* FROM products p1
JOIN (
    SELECT CEIL(RAND() * (SELECT MAX(product_id) FROM products)) as random_id
) p2
WHERE p1.product_id >= p2.random_id
LIMIT 10;
```

### 2. 事前サンプリングテーブル手法

```sql
-- ✅ サンプリング専用テーブル作成
CREATE TABLE product_sample (
    sample_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    weight DECIMAL(5,4) DEFAULT 1.0000,  -- 重み付け用
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    INDEX idx_weight (weight)
);

-- 定期的なサンプル更新（日次バッチ）
TRUNCATE product_sample;
INSERT INTO product_sample (product_id, weight)
SELECT 
    product_id,
    -- 人気商品ほど高い重み
    (view_count + purchase_count * 5) / 1000.0 as weight
FROM products 
WHERE is_active = TRUE;

-- 高速なランダム抽出
SELECT p.* FROM products p
INNER JOIN (
    SELECT product_id 
    FROM product_sample 
    WHERE RAND() <= weight 
    ORDER BY RAND() 
    LIMIT 10
) s ON p.product_id = s.product_id;
```

### 3. インデックス活用手法

```sql
-- ✅ タイムスタンプベースのランダム抽出
-- created_at にインデックスが存在する場合

-- 1. ランダムな時間範囲を選択
SET @random_date = DATE_SUB(NOW(), 
    INTERVAL FLOOR(RAND() * 365) DAY
);

-- 2. その時間以降の最初のN件
SELECT * FROM products 
WHERE created_at >= @random_date 
ORDER BY created_at 
LIMIT 10;

-- または時間範囲の両端からランダム選択
SET @start_date = DATE_SUB(NOW(), INTERVAL 365 DAY);
SET @end_date = DATE_SUB(NOW(), INTERVAL 30 DAY);

SELECT * FROM products 
WHERE created_at BETWEEN @start_date AND @end_date
ORDER BY product_id  -- インデックス使用
LIMIT 10 OFFSET FLOOR(RAND() * 100);  -- 小さなOFFSET
```

### 4. 階層化サンプリング手法

```sql
-- ✅ カテゴリ別均等サンプリング
WITH category_samples AS (
    SELECT 
        category_id,
        COUNT(*) as total_products
    FROM products 
    GROUP BY category_id
),
random_per_category AS (
    SELECT 
        p.product_id,
        p.category_id,
        ROW_NUMBER() OVER (
            PARTITION BY p.category_id 
            ORDER BY 
                -- 簡易ランダム（IDベース）
                ABS(p.product_id * 12345 + 67890) % 100000
        ) as rn
    FROM products p
    INNER JOIN category_samples cs ON p.category_id = cs.category_id
)
SELECT p.* FROM products p
INNER JOIN random_per_category rpc ON p.product_id = rpc.product_id
WHERE rpc.rn <= 2  -- 各カテゴリから2個ずつ
ORDER BY p.category_id, rpc.rn;
```

## 実装パターン

### パターン1: アプリケーション制御ランダム

```sql
-- ✅ アプリケーション側でランダムID生成
-- PHP例での実装
/*
$min_max = $pdo->query("SELECT MIN(id) as min_id, MAX(id) as max_id FROM products")->fetch();
$random_ids = [];

for($i = 0; $i < 10; $i++) {
    $random_ids[] = mt_rand($min_max['min_id'], $min_max['max_id']);
}

$placeholders = str_repeat('?,', count($random_ids) - 1) . '?';
$stmt = $pdo->prepare("SELECT * FROM products WHERE id >= ? LIMIT 1");
*/

-- 対応するSQLクエリ（準備済みステートメント）
SELECT * FROM products WHERE product_id >= ? LIMIT 1;
```

### パターン2: 重み付きランダム抽出

```sql
-- ✅ 商品人気度による重み付きサンプリング
CREATE TABLE product_weights (
    product_id INT PRIMARY KEY,
    popularity_score DECIMAL(10,4) NOT NULL DEFAULT 1.0,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- 人気度計算（日次更新）
INSERT INTO product_weights (product_id, popularity_score)
SELECT 
    p.product_id,
    -- 重み計算: ビュー数 + 購入数×5 + レビュー数×2
    GREATEST(
        (COALESCE(pv.view_count, 0) + 
         COALESCE(po.purchase_count, 0) * 5 + 
         COALESCE(pr.review_count, 0) * 2) / 100.0,
        0.1  -- 最小重み
    ) as popularity_score
FROM products p
LEFT JOIN product_views pv ON p.product_id = pv.product_id
LEFT JOIN purchase_orders po ON p.product_id = po.product_id  
LEFT JOIN product_reviews pr ON p.product_id = pr.product_id
ON DUPLICATE KEY UPDATE
    popularity_score = VALUES(popularity_score),
    last_updated = CURRENT_TIMESTAMP;

-- 重み付きランダム抽出
SELECT p.* FROM products p
INNER JOIN product_weights pw ON p.product_id = pw.product_id
WHERE RAND() * 10 <= pw.popularity_score  -- 重み適用
LIMIT 20;
```

### パターン3: Redis/Memcachedを活用した高速化

```sql
-- ✅ キャッシュ活用パターン
-- 1. 事前計算したランダムIDリストをキャッシュに保存
-- Redis例: 
-- LPUSH random_products:2024-03-15 123 456 789 101112
-- EXPIRE random_products:2024-03-15 3600

-- 2. バッチでキャッシュ更新
CREATE EVENT update_random_cache
ON SCHEDULE EVERY 1 HOUR
DO
BEGIN
    SET @cache_key = CONCAT('random_products:', DATE_FORMAT(NOW(), '%Y-%m-%d-%H'));
    
    -- ランダムIDリスト生成（大量のIDから効率的に抽出）
    CREATE TEMPORARY TABLE temp_random_ids AS
    SELECT product_id 
    FROM (
        SELECT 
            product_id,
            ROW_NUMBER() OVER (ORDER BY RAND()) as rn
        FROM (
            SELECT product_id 
            FROM products 
            WHERE is_active = TRUE 
            ORDER BY product_id  -- インデックス使用
            LIMIT 10000  -- 母集団を制限
        ) sample_pool
    ) random_ordered
    WHERE rn <= 1000;  -- 1000個のランダムIDを準備
    
    -- アプリケーション側でキャッシュに格納
END;

-- 3. キャッシュからランダムID取得してクエリ実行
SELECT * FROM products 
WHERE product_id IN (?, ?, ?, ?, ?)  -- キャッシュから取得したID
LIMIT 10;
```

### パターン4: 時系列データでの効率的ランダム抽出

```sql
-- ✅ ログデータからのランダムサンプリング
CREATE TABLE access_logs (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    page_url VARCHAR(500),
    access_time TIMESTAMP,
    ip_address VARCHAR(45),
    
    INDEX idx_access_time (access_time),
    INDEX idx_user_time (user_id, access_time)
);

-- 日付範囲を指定したランダム抽出
DELIMITER //
CREATE PROCEDURE GetRandomAccessLogs(
    IN start_date DATE,
    IN end_date DATE,
    IN sample_count INT
)
BEGIN
    DECLARE time_range_seconds INT;
    DECLARE i INT DEFAULT 0;
    
    -- 時間範囲を秒単位で計算
    SET time_range_seconds = TIMESTAMPDIFF(SECOND, start_date, end_date);
    
    -- 一時テーブルでランダム時刻生成
    CREATE TEMPORARY TABLE temp_random_times (
        random_time TIMESTAMP,
        INDEX idx_time (random_time)
    );
    
    -- ランダム時刻を生成
    WHILE i < sample_count DO
        INSERT INTO temp_random_times (random_time)
        VALUES (
            FROM_UNIXTIME(
                UNIX_TIMESTAMP(start_date) + 
                FLOOR(RAND() * time_range_seconds)
            )
        );
        SET i = i + 1;
    END WHILE;
    
    -- 各ランダム時刻以降の最初のレコードを取得
    SELECT al.* FROM access_logs al
    INNER JOIN (
        SELECT 
            trt.random_time,
            (SELECT MIN(log_id) 
             FROM access_logs al2 
             WHERE al2.access_time >= trt.random_time) as target_log_id
        FROM temp_random_times trt
    ) targets ON al.log_id = targets.target_log_id
    WHERE targets.target_log_id IS NOT NULL;
    
    DROP TEMPORARY TABLE temp_random_times;
END //
DELIMITER ;
```

## パフォーマンス比較

```sql
-- ✅ パフォーマンス測定例
-- 100万件のテーブルで10件ランダム取得の比較

-- 1. ORDER BY RAND() (アンチパターン)
-- 実行時間: 2.3秒, CPU: 高, メモリ: 500MB
SELECT * FROM products ORDER BY RAND() LIMIT 10;

-- 2. IDベースランダム (改善版)  
-- 実行時間: 0.01秒, CPU: 低, メモリ: 1MB
SELECT * FROM products WHERE product_id >= FLOOR(RAND() * 1000000) LIMIT 10;

-- 3. サンプリングテーブル (改善版)
-- 実行時間: 0.005秒, CPU: 低, メモリ: 0.5MB  
SELECT p.* FROM products p
INNER JOIN product_sample ps ON p.product_id = ps.product_id
WHERE RAND() <= ps.weight LIMIT 10;

-- パフォーマンス改善: 230倍高速化
```

## まとめ

### ランダムセレクションアンチパターンの問題点
1. **全件スキャン**: ORDER BY RAND() による性能劣化
2. **メモリ爆発**: 大容量テーブルでのメモリ不足
3. **レプリケーション問題**: マスター・スレーブ間の不整合
4. **非決定的**: テストやデバッグが困難

### 効率的手法による利点  
1. **高速処理**: インデックス活用による高速化
2. **メモリ効率**: 必要最小限のメモリ使用
3. **スケーラブル**: データ量に比例しない処理時間
4. **一貫性**: レプリケーション環境でも安定

**重要**: ランダム抽出が必要な場合は、ORDER BY RAND()を避け、IDベースやサンプリングテーブルなどの効率的な手法を選択してください。

---

# 第16章 プアマンズ・サーチエンジン（貧者のサーチエンジン）

## 問題と解決策

### ❌ 問題：LIKE '%keyword%'による全文検索

```sql
-- ❌ 悪い例：LIKE による検索
SELECT * FROM articles 
WHERE title LIKE '%データベース%' 
   OR content LIKE '%データベース%'
   OR tags LIKE '%データベース%';

-- 複数キーワード検索（さらに悪化）
SELECT * FROM articles 
WHERE (title LIKE '%データベース%' OR content LIKE '%データベース%')
  AND (title LIKE '%性能%' OR content LIKE '%性能%')
  AND (title LIKE '%最適化%' OR content LIKE '%最適化%');
```

**主な問題**：
- インデックスが使用されない
- 全文スキャンによる性能劣化
- 曖昧検索の不正確性
- 関連度による並び替え不可

### ✅ 解決策：専用の全文検索機能

```sql
-- ✅ MySQL全文検索
ALTER TABLE articles ADD FULLTEXT(title, content, tags);

SELECT *, MATCH(title, content, tags) AGAINST ('データベース 性能 最適化') as relevance
FROM articles 
WHERE MATCH(title, content, tags) AGAINST ('データベース 性能 最適化')
ORDER BY relevance DESC;

-- ✅ PostgreSQL全文検索  
ALTER TABLE articles ADD COLUMN search_vector tsvector;
UPDATE articles SET search_vector = to_tsvector('japanese', title || ' ' || content || ' ' || tags);
CREATE INDEX idx_search ON articles USING gin(search_vector);

SELECT *, ts_rank(search_vector, query) as rank
FROM articles, to_tsquery('japanese', 'データベース & 性能 & 最適化') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

## プアマンズ・サーチエンジンアンチパターンとは？

LIKE演算子の部分一致（%keyword%）を使用して全文検索機能を実装しようとすることで、性能問題や検索精度の低下を引き起こすアンチパターンです。

## ❌ 問題点の詳細

### 1. インデックス使用不可による性能劣化

```sql
-- ❌ LIKE '%keyword%' はインデックスを使用できない
CREATE TABLE documents (
    doc_id INT PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    created_at TIMESTAMP,
    
    INDEX idx_title (title),        -- 使用されない
    INDEX idx_created (created_at)  -- これは使用される
);

-- このクエリの実行計画
EXPLAIN SELECT * FROM documents 
WHERE title LIKE '%MySQL%'
  AND created_at >= '2024-01-01';

/*
実行計画:
1. created_at インデックスで範囲スキャン (良い)
2. 各行で title LIKE '%MySQL%' を全文チェック (悪い)
3. title のインデックスは使用されない
*/

-- 100万行のテーブルで:
-- - 'MySQL%' (前方一致): 0.01秒 (インデックス使用)
-- - '%MySQL%' (部分一致): 2.5秒 (フルスキャン)
-- - '%MySQL' (後方一致): 2.5秒 (フルスキャン)
```

### 2. 複数キーワードの組み合わせ爆発

```sql
-- ❌ 複数キーワードでの複雑なクエリ
-- 「データベース」「性能」「チューニング」すべてを含む記事検索
SELECT * FROM articles 
WHERE (
    title LIKE '%データベース%' OR content LIKE '%データベース%'
) AND (
    title LIKE '%性能%' OR content LIKE '%性能%'  
) AND (
    title LIKE '%チューニング%' OR content LIKE '%チューニング%'
);

-- 問題:
-- - 6個のLIKE条件 (3キーワード × 2列)
-- - 各行で6回の文字列検索処理
-- - インデックス使用不可でフルスキャン
-- - キーワード数に比例して性能劣化

-- さらに悪い例: OR条件での検索
SELECT * FROM articles
WHERE title LIKE '%データベース%' 
   OR title LIKE '%DB%'
   OR title LIKE '%MySQL%' 
   OR title LIKE '%PostgreSQL%'
   OR title LIKE '%Oracle%'
   OR content LIKE '%データベース%'
   OR content LIKE '%DB%'
   OR content LIKE '%MySQL%'
   -- ... 16個のLIKE条件
```

### 3. 検索精度の問題

```sql
-- ❌ 不正確な検索結果
-- 「データベース管理」を検索
SELECT title FROM articles WHERE content LIKE '%データベース管理%';

-- 問題のある結果例:
-- ○ "データベース管理システムの構築" (正解)
-- × "データベースは管理が重要" (誤ヒット - 「は管理」が含まれる)
-- × "データベース・管理・セキュリティ" (誤ヒット - 中点で区切られている)
-- ○ "データベース管理者の仕事" (正解だが関連度は低い)

-- 関連度による並び替えも不可能
-- すべての結果が同等の重みで表示される
```

### 4. 特殊文字や言語の問題

```sql
-- ❌ 特殊文字での問題
-- 検索語: "C++"
SELECT * FROM articles WHERE content LIKE '%C++%';
-- 問題: "C++"が正規表現として解釈される可能性

-- 日本語の問題
SELECT * FROM articles WHERE content LIKE '%データーベース%';  -- "データーベース"
-- "データベース"(長音なし)は検索されない

-- 英語の語形変化
SELECT * FROM articles WHERE content LIKE '%database%';
-- "databases", "Database", "DATABASE" 等のバリエーション対応が困難
```

## ✅ 専用の全文検索機能

### 1. MySQL 全文検索 (FULLTEXT)

```sql
-- ✅ FULLTEXT インデックス作成
CREATE TABLE articles (
    article_id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content LONGTEXT NOT NULL,
    tags VARCHAR(500),
    author_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 全文検索インデックス
    FULLTEXT ft_title (title),
    FULLTEXT ft_content (content), 
    FULLTEXT ft_all (title, content, tags)
);

-- 基本的な全文検索
SELECT 
    article_id, 
    title,
    MATCH(title, content, tags) AGAINST ('データベース 性能') as relevance_score
FROM articles
WHERE MATCH(title, content, tags) AGAINST ('データベース 性能')
ORDER BY relevance_score DESC
LIMIT 20;

-- ブール検索 (AND, OR, NOT)
SELECT * FROM articles
WHERE MATCH(title, content, tags) AGAINST (
    '+データベース +性能 -Oracle' IN BOOLEAN MODE
);

-- フレーズ検索
SELECT * FROM articles  
WHERE MATCH(title, content, tags) AGAINST (
    '"データベース管理システム"' IN BOOLEAN MODE
);

-- ワイルドカード検索
SELECT * FROM articles
WHERE MATCH(title, content, tags) AGAINST (
    'データベース*' IN BOOLEAN MODE
);
```

### 2. PostgreSQL 全文検索 (FTS)

```sql
-- ✅ PostgreSQL 全文検索設定
-- 日本語対応の設定
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE TABLE articles (
    article_id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    tags VARCHAR(500),
    author_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 全文検索用ベクトル列
    search_vector_ja tsvector,
    search_vector_en tsvector
);

-- 検索ベクトル更新
UPDATE articles SET 
    search_vector_ja = to_tsvector('japanese', title || ' ' || content || ' ' || COALESCE(tags, '')),
    search_vector_en = to_tsvector('english', title || ' ' || content || ' ' || COALESCE(tags, ''));

-- GINインデックス作成
CREATE INDEX idx_search_ja ON articles USING gin(search_vector_ja);
CREATE INDEX idx_search_en ON articles USING gin(search_vector_en);

-- 全文検索クエリ
SELECT 
    article_id, 
    title,
    ts_rank(search_vector_ja, query) as rank
FROM articles, to_tsquery('japanese', 'データベース & 性能') query
WHERE search_vector_ja @@ query
ORDER BY rank DESC
LIMIT 20;

-- 複雑な検索 (OR条件)
SELECT * FROM articles, to_tsquery('japanese', 'データベース | MySQL | PostgreSQL') query
WHERE search_vector_ja @@ query;

-- フレーズ検索
SELECT * FROM articles, phraseto_tsquery('japanese', 'データベース管理システム') query
WHERE search_vector_ja @@ query;

-- 自動更新トリガー
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector_ja = to_tsvector('japanese', NEW.title || ' ' || NEW.content || ' ' || COALESCE(NEW.tags, ''));
    NEW.search_vector_en = to_tsvector('english', NEW.title || ' ' || NEW.content || ' ' || COALESCE(NEW.tags, ''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_search_vector_trigger
    BEFORE INSERT OR UPDATE ON articles
    FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```

### 3. Elasticsearch連携パターン

```sql
-- ✅ ハイブリッドアプローチ (DB + Elasticsearch)

-- 1. 基本データはRDBMSで管理
CREATE TABLE articles (
    article_id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content LONGTEXT NOT NULL,
    author_id INT NOT NULL,
    category_id INT NOT NULL,
    status ENUM('draft', 'published', 'archived') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Elasticsearch同期用
    es_synced_at TIMESTAMP NULL,
    es_document_version INT DEFAULT 1,
    
    FOREIGN KEY (author_id) REFERENCES users(user_id),
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

-- 2. Elasticsearch同期用ビュー
CREATE VIEW articles_for_es AS
SELECT 
    a.article_id,
    a.title,
    a.content,
    a.status,
    a.created_at,
    a.updated_at,
    u.username as author_name,
    c.category_name,
    GROUP_CONCAT(t.tag_name) as tags
FROM articles a
JOIN users u ON a.author_id = u.user_id
JOIN categories c ON a.category_id = c.category_id
LEFT JOIN article_tags at ON a.article_id = at.article_id
LEFT JOIN tags t ON at.tag_id = t.tag_id
WHERE a.status = 'published'
GROUP BY a.article_id;

-- 3. 検索はElasticsearchで実行後、RDBMSから詳細取得
-- (アプリケーション層での実装)
-- GET /articles/_search
-- {
--   "query": {
--     "multi_match": {
--       "query": "データベース 性能",
--       "fields": ["title^2", "content", "tags"]
--     }
--   }
-- }

-- 検索結果のIDでRDBMSから取得
SELECT a.*, u.username, c.category_name
FROM articles a
JOIN users u ON a.author_id = u.user_id  
JOIN categories c ON a.category_id = c.category_id
WHERE a.article_id IN (?, ?, ?, ?)  -- Elasticsearchの結果ID
ORDER BY FIELD(a.article_id, ?, ?, ?, ?);  -- ES結果の順序維持
```

## 実装パターン

### パターン1: 段階的検索

```sql
-- ✅ 複数段階での検索精度向上
-- 1段階目: 全文検索で候補を絞り込み
WITH fulltext_results AS (
    SELECT 
        article_id,
        title,
        MATCH(title, content) AGAINST ('データベース 性能 最適化') as ft_score
    FROM articles
    WHERE MATCH(title, content) AGAINST ('データベース 性能 最適化')
    ORDER BY ft_score DESC
    LIMIT 1000  -- 候補を1000件に絞る
),
-- 2段階目: より詳細な条件で絞り込み
filtered_results AS (
    SELECT 
        fr.*,
        a.created_at,
        a.view_count,
        a.like_count
    FROM fulltext_results fr
    JOIN articles a ON fr.article_id = a.article_id
    WHERE a.status = 'published'
      AND a.created_at >= DATE_SUB(NOW(), INTERVAL 2 YEAR)  -- 2年以内
),
-- 3段階目: 複合スコア計算
final_ranking AS (
    SELECT 
        *,
        -- 複合スコア: 全文検索 + 人気度 + 新しさ
        (
            ft_score * 0.6 +  -- 関連度60%
            LOG10(view_count + 1) * 0.2 +  -- 人気度20% 
            (DATEDIFF(NOW(), created_at) / -365.0) * 0.2  -- 新しさ20%
        ) as final_score
    FROM filtered_results
)
SELECT * FROM final_ranking
ORDER BY final_score DESC
LIMIT 20;
```

### パターン2: 検索ログ分析

```sql
-- ✅ 検索品質向上のためのログ分析
CREATE TABLE search_logs (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NULL,
    search_query VARCHAR(500) NOT NULL,
    search_type ENUM('fulltext', 'tag', 'category') NOT NULL,
    results_count INT NOT NULL,
    clicked_article_id INT NULL,
    click_position INT NULL,  -- 何番目の結果をクリックしたか
    search_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    execution_time_ms INT,  -- クエリ実行時間
    
    INDEX idx_query (search_query),
    INDEX idx_search_time (search_time),
    FOREIGN KEY (clicked_article_id) REFERENCES articles(article_id)
);

-- 人気の検索クエリ分析
SELECT 
    search_query,
    COUNT(*) as search_count,
    AVG(results_count) as avg_results,
    COUNT(clicked_article_id) * 100.0 / COUNT(*) as click_through_rate,
    AVG(click_position) as avg_click_position
FROM search_logs
WHERE search_time >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY search_query
HAVING search_count >= 10
ORDER BY search_count DESC;

-- 検索結果の品質分析
SELECT 
    CASE 
        WHEN click_position IS NULL THEN 'No Click'
        WHEN click_position <= 3 THEN 'Top 3'
        WHEN click_position <= 10 THEN 'Top 10'
        ELSE 'Beyond 10'
    END as click_group,
    COUNT(*) as search_count,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() as percentage
FROM search_logs
WHERE search_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY click_group
ORDER BY click_position;
```

### パターン3: 動的インデックス管理

```sql
-- ✅ 検索パフォーマンス監視と最適化
CREATE TABLE search_performance (
    perf_id INT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    index_name VARCHAR(100) NOT NULL,
    avg_execution_time_ms DECIMAL(10,3),
    query_count INT,
    last_optimized TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- インデックス使用状況の監視
INSERT INTO search_performance (table_name, index_name, avg_execution_time_ms, query_count)
SELECT 
    'articles' as table_name,
    'ft_all' as index_name,
    AVG(execution_time_ms) as avg_execution_time_ms,
    COUNT(*) as query_count
FROM search_logs
WHERE search_time >= DATE_SUB(NOW(), INTERVAL 1 DAY)
  AND search_type = 'fulltext';

-- 自動インデックス最適化
CREATE EVENT optimize_fulltext_index
ON SCHEDULE EVERY 1 DAY
DO
BEGIN
    -- 統計情報更新
    ANALYZE TABLE articles;
    
    -- フラグメンテーション状況確認
    SET @fragmentation = (
        SELECT data_free / (data_length + index_length) * 100
        FROM information_schema.tables 
        WHERE table_schema = DATABASE() 
        AND table_name = 'articles'
    );
    
    -- 10%以上フラグメンテーションがある場合に最適化
    IF @fragmentation > 10 THEN
        OPTIMIZE TABLE articles;
        
        INSERT INTO search_performance (table_name, index_name, last_optimized)
        VALUES ('articles', 'fulltext_optimization', NOW());
    END IF;
END;
```

## パフォーマンス比較

```sql
-- ✅ 検索方式の性能比較 (100万件のarticlesテーブル)

-- 1. LIKE検索 (アンチパターン)
-- 実行時間: 3.2秒, CPU使用率: 90%
SELECT * FROM articles 
WHERE content LIKE '%データベース%' AND content LIKE '%性能%'
LIMIT 20;

-- 2. MySQL FULLTEXT検索
-- 実行時間: 0.08秒, CPU使用率: 15%
SELECT *, MATCH(content) AGAINST('データベース 性能') as score
FROM articles
WHERE MATCH(content) AGAINST('データベース 性能')
ORDER BY score DESC
LIMIT 20;

-- 3. PostgreSQL 全文検索
-- 実行時間: 0.05秒, CPU使用率: 10%
SELECT *, ts_rank(search_vector, query) as rank
FROM articles, to_tsquery('japanese', 'データベース & 性能') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;

-- 性能改善: 40-60倍高速化
-- メモリ使用量: 80%削減
-- CPU使用率: 75%削減
```

## まとめ

### プアマンズ・サーチエンジンアンチパターンの問題点
1. **性能劣化**: LIKE '%keyword%'による全文スキャン
2. **検索精度低下**: 関連度順ソート不可、誤ヒット発生
3. **スケーラビリティ欠如**: データ量に比例した性能劣化
4. **機能制限**: 複雑な検索条件への対応困難

### 専用全文検索による利点
1. **高速検索**: 専用インデックスによる高速化
2. **高精度**: 関連度スコア、言語処理対応
3. **豊富な機能**: ブール検索、フレーズ検索、ファジー検索
4. **スケーラブル**: データ量に比例しない検索性能

**重要**: 全文検索が必要な場合は、LIKE '%keyword%'を避け、データベースの全文検索機能やElasticsearch等の専用検索エンジンを活用してください。

---

SQLアンチパターン第13章から第16章の詳細な解説を作成しました。各章では具体的な問題例、詳細な解説、そして実践的な解決策を含めています。