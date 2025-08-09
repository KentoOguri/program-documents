# 第9章 ラウンディングエラー（丸め誤差）詳解

## 目次
1. [問題と解決策](#問題と解決策)
2. [ラウンディングエラーとは？](#ラウンディングエラーとは？)
3. [なぜ浮動小数点で誤差が発生するのか？](#なぜ浮動小数点で誤差が発生するのか？)
4. [具体的な問題例](#具体的な問題例)
5. [❌ 問題点の詳細](#❌-問題点の詳細)
6. [✅ 解決策：DECIMAL/NUMERIC型の使用](#✅-解決策：decimalnumeric型の使用)
7. [実用的な計算例](#実用的な計算例)
8. [データベース別の対応](#データベース別の対応)
9. [移行とテスト](#移行とテスト)
10. [パフォーマンス考慮事項](#パフォーマンス考慮事項)
11. [まとめ](#まとめ)

## 問題と解決策

### ❌ 問題：FLOAT/DOUBLE型による計算誤差

```sql
-- 金融システムで発生する深刻な問題
CREATE TABLE accounts (
    balance FLOAT  -- ❌ 危険！
);

SELECT 0.1 + 0.2;  -- 結果: 0.30000000000000004 (期待値: 0.3)
SELECT * FROM accounts WHERE balance = 0.11;  -- ヒットしない可能性
```

**主な問題**：
- 基本的な算術演算で誤差が発生
- 等値比較が期待通りに動作しない
- 累積誤差により大きなズレが生じる
- 会計システムで監査上の問題となる

### ✅ 解決策：DECIMAL/NUMERIC型の使用

```sql
-- 正確な金融計算を実現
CREATE TABLE accounts (
    balance DECIMAL(15,2)  -- ✅ 正しい！
);

SELECT 0.1 + 0.2;  -- 結果: 0.3 (正確)
SELECT * FROM accounts WHERE balance = 0.11;  -- 期待通りに動作
```

**解決方法**：
1. **DECIMAL/NUMERIC型必須**: 金額・価格は必ずDECIMAL型
2. **適切な精度設定**: 業務要件に応じた桁数決定
3. **四捨五入の統一**: ROUNDによる一貫した丸め処理
4. **移行時の検証**: FLOAT→DECIMAL移行時の差異確認
5. **パフォーマンス考慮**: 精度と性能のバランス

**重要**: 金額を扱うシステムでFLOAT/DOUBLE型を使用するのは非常に危険です。必ずDECIMAL/NUMERIC型を使用してください。

## ラウンディングエラーとは？

ラウンディングエラー（丸め誤差）とは、金額や正確な小数値を扱う際にFLOAT型やDOUBLE型（浮動小数点型）を使用することで発生する予期しない計算誤差のことです。特に金融システムでは致命的な問題となります。

## なぜ浮動小数点で誤差が発生するのか？

### IEEE 754標準の仕組み
```
FLOAT (32bit):  [符号1bit][指数8bit][仮数23bit]
DOUBLE (64bit): [符号1bit][指数11bit][仮数52bit]

例：0.1 を2進数で表現すると
0.1 (10進) = 0.0001100110011001100110011... (2進、循環小数)
```

### 根本的な問題
多くの10進小数は2進数では正確に表現できないため、近似値が格納されます。

## 具体的な問題例

### シチュエーション1：金融システムの口座管理

```sql
-- ❌ 悪い例：FLOAT型での金額管理
CREATE TABLE accounts (
    account_id INT PRIMARY KEY,
    account_number VARCHAR(20) UNIQUE,
    balance FLOAT,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE transactions (
    transaction_id INT PRIMARY KEY,
    account_id INT,
    amount FLOAT,
    transaction_type ENUM('deposit', 'withdrawal'),
    transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(account_id)
);
```

### データ例と問題の発生

```sql
-- 初期データ
INSERT INTO accounts VALUES (1, 'ACC001', 1000.00, NOW());

-- 取引データ
INSERT INTO transactions VALUES 
(1, 1, 999.99, 'withdrawal', NOW()),
(2, 1, 0.10, 'deposit', NOW());

-- ❌ 期待値：残高 = 1000.00 - 999.99 + 0.10 = 0.11
-- 実際の計算
SELECT 1000.00 - 999.99 + 0.10 as calculated_balance;
-- 結果：0.10999999046325684 (MySQL FLOAT)

-- ❌ 残高更新
UPDATE accounts 
SET balance = balance - 999.99 + 0.10
WHERE account_id = 1;

SELECT balance FROM accounts WHERE account_id = 1;
-- 結果：0.109999990463257 (期待値：0.11)
```

### シチュエーション2：商品価格と税金計算

```sql
-- ❌ 悪い例：商品価格にFLOAT使用
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    unit_price FLOAT,
    tax_rate FLOAT DEFAULT 0.10  -- 消費税10%
);

CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price FLOAT,
    total_amount FLOAT
);

-- データ例
INSERT INTO products VALUES (1, 'ノートPC', 89999.00, 0.10);

-- ❌ 税込価格計算の問題
SELECT 
    product_name,
    unit_price,
    unit_price * (1 + tax_rate) as price_with_tax,
    unit_price * (1 + tax_rate) - unit_price as tax_amount
FROM products WHERE product_id = 1;

-- 結果例：
-- price_with_tax: 98998.9010009765625 (期待値：98999.00)
-- tax_amount: 9999.90100097655 (期待値：9000.00)
```

## ❌ 問題点の詳細

### 1. 基本的な算術演算の誤差

```sql
-- ❌ 単純な加算でも誤差が発生
SELECT 0.1 + 0.2 as result;
-- MySQL: 0.30000000000000004
-- 期待値: 0.3

SELECT 0.1 + 0.1 + 0.1 as result;  
-- MySQL: 0.30000000000000004
-- 期待値: 0.3

-- ❌ 減算でも誤差
SELECT 1.0 - 0.9 as result;
-- MySQL: 0.09999999999999998  
-- 期待値: 0.1
```

### 2. 等値比較の失敗

```sql
-- ❌ 期待通りに動作しない比較
SELECT * FROM accounts WHERE balance = 0.11;
-- 結果：0件（実際の値は0.10999999046325684のため）

-- ❌ 条件分岐の予期しない結果
SELECT 
    account_id,
    balance,
    CASE 
        WHEN balance = 0.11 THEN 'Exact Match'
        WHEN balance > 0.11 THEN 'Greater'
        ELSE 'Less or Different'
    END as comparison_result
FROM accounts WHERE account_id = 1;
-- 結果：'Less or Different' (期待値：'Exact Match')
```

### 3. 累積誤差の拡大

```sql
-- ❌ 利息計算での累積誤差
CREATE TABLE interest_calculation (
    month_num INT,
    principal FLOAT,
    interest_rate FLOAT,
    monthly_interest FLOAT,
    new_balance FLOAT
);

-- 月利0.5%の複利計算
SET @principal = 100000.0;
SET @rate = 0.005;

INSERT INTO interest_calculation VALUES 
(1, @principal, @rate, @principal * @rate, @principal * (1 + @rate));

-- 12ヶ月分の計算をシミュレート
-- 最終的に期待値から大きくずれる可能性
```

### 4. 集計での問題

```sql
-- ❌ SUM集計での誤差
INSERT INTO transactions VALUES
(3, 1, 0.01, 'deposit', NOW()),
(4, 1, 0.01, 'deposit', NOW()),
(5, 1, 0.01, 'deposit', NOW());
-- 0.01を100回繰り返すと...

SELECT SUM(amount) FROM transactions WHERE transaction_type = 'deposit';
-- 期待値：1.03（0.10 + 0.01×3）
-- 実際値：1.0299999713897705
```

### 5. 会計システムでの致命的問題

```sql
-- ❌ 決算書の不整合
CREATE TABLE financial_summary (
    period VARCHAR(10),
    revenue FLOAT,
    expenses FLOAT,
    profit FLOAT
);

INSERT INTO financial_summary VALUES
('2024Q1', 1000000.50, 750000.25, 250000.25);

-- 検算
SELECT 
    period,
    revenue,
    expenses, 
    profit,
    revenue - expenses as calculated_profit,
    profit - (revenue - expenses) as difference
FROM financial_summary;

-- difference が 0 にならない可能性
-- 監査で問題となる
```

### 6. 通貨換算での問題

```sql
-- ❌ 為替レート計算
CREATE TABLE exchange_rates (
    from_currency CHAR(3),
    to_currency CHAR(3),
    rate FLOAT,
    effective_date DATE
);

INSERT INTO exchange_rates VALUES ('USD', 'JPY', 149.25, '2024-01-15');

-- 100.33 USD → JPY変換
SELECT 100.33 * 149.25 as jpy_amount;
-- 結果：14974.2324999999981 (期待値：14974.2325)

-- 逆変換での不整合
SELECT (100.33 * 149.25) / 149.25 as back_to_usd;
-- 結果：100.329999999999984 (期待値：100.33)
```

## ✅ 解決策：DECIMAL/NUMERIC型の使用

### 正しいテーブル設計

```sql
-- ✅ 正しい例：DECIMAL型での金額管理
CREATE TABLE accounts (
    account_id INT PRIMARY KEY,
    account_number VARCHAR(20) UNIQUE,
    balance DECIMAL(15,2) NOT NULL DEFAULT 0.00,  -- 15桁、小数点以下2桁
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE transactions (
    transaction_id INT PRIMARY KEY,
    account_id INT,
    amount DECIMAL(12,2) NOT NULL,  -- 12桁、小数点以下2桁
    transaction_type ENUM('deposit', 'withdrawal') NOT NULL,
    transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    description VARCHAR(255),
    FOREIGN KEY (account_id) REFERENCES accounts(account_id)
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,  -- 99,999,999.99まで
    tax_rate DECIMAL(5,4) NOT NULL DEFAULT 0.1000,  -- 100.0000%まで
    cost DECIMAL(10,2),
    profit_margin DECIMAL(5,4)
);
```

### 正確な計算の実現

```sql
-- ✅ 正確な算術演算
SELECT 0.1 + 0.2 as result;
-- 結果：0.3 (正確)

SELECT 1000.00 - 999.99 + 0.10 as calculated_balance;
-- 結果：0.11 (正確)

-- ✅ 正確な等値比較
SELECT * FROM accounts WHERE balance = 0.11;
-- 期待通りに動作

-- ✅ 税金計算
SELECT 
    product_name,
    unit_price,
    ROUND(unit_price * (1 + tax_rate), 2) as price_with_tax,
    ROUND(unit_price * tax_rate, 2) as tax_amount
FROM products WHERE product_id = 1;
-- 正確な結果
```

### 精度の適切な設定

```sql
-- ✅ 用途別の精度設定

-- 一般的な金額（円、ドル等）
price DECIMAL(10,2)         -- 99,999,999.99まで

-- 高精度金融計算
interest_rate DECIMAL(8,6)  -- 99.999999%まで
exchange_rate DECIMAL(12,8) -- 9999.99999999まで

-- 仮想通貨（小数点以下が多い）
crypto_amount DECIMAL(20,8) -- 999兆.99999999まで

-- 単価×数量の計算結果保存
total_amount DECIMAL(15,4)  -- 中間計算の精度確保

-- 統計・分析用
percentage DECIMAL(5,2)     -- 999.99%まで
ratio DECIMAL(10,6)         -- 9999.999999まで
```

## 実用的な計算例

### 1. 複利計算システム

```sql
-- ✅ 正確な複利計算
CREATE TABLE compound_interest (
    calculation_id INT AUTO_INCREMENT PRIMARY KEY,
    principal DECIMAL(15,2) NOT NULL,
    annual_rate DECIMAL(6,4) NOT NULL,  -- 99.9999%まで
    compound_frequency INT NOT NULL,    -- 年間複利回数
    years DECIMAL(4,2) NOT NULL,
    final_amount DECIMAL(18,4),
    total_interest DECIMAL(18,4)
);

-- 複利計算の実装
DELIMITER //
CREATE FUNCTION calculate_compound_interest(
    principal DECIMAL(15,2),
    annual_rate DECIMAL(6,4),
    compound_frequency INT,
    years DECIMAL(4,2)
) RETURNS DECIMAL(18,4)
READS SQL DATA
DETERMINISTIC
BEGIN
    DECLARE final_amount DECIMAL(18,4);
    
    -- A = P(1 + r/n)^(nt)
    SET final_amount = principal * POWER(
        1 + (annual_rate / compound_frequency), 
        compound_frequency * years
    );
    
    RETURN final_amount;
END //
DELIMITER ;

-- 使用例
INSERT INTO compound_interest (principal, annual_rate, compound_frequency, years)
VALUES (1000000.00, 0.0350, 12, 5.00);

UPDATE compound_interest 
SET final_amount = calculate_compound_interest(principal, annual_rate, compound_frequency, years),
    total_interest = calculate_compound_interest(principal, annual_rate, compound_frequency, years) - principal
WHERE calculation_id = LAST_INSERT_ID();
```

### 2. 注文システム

```sql
-- ✅ 正確な注文計算
CREATE TABLE orders (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    subtotal DECIMAL(12,2) DEFAULT 0.00,
    tax_amount DECIMAL(12,2) DEFAULT 0.00,
    discount_amount DECIMAL(12,2) DEFAULT 0.00,
    shipping_cost DECIMAL(8,2) DEFAULT 0.00,
    total_amount DECIMAL(12,2) DEFAULT 0.00
);

CREATE TABLE order_items (
    item_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity DECIMAL(8,3) NOT NULL,  -- 小数点での注文も可能
    unit_price DECIMAL(10,2) NOT NULL,
    line_total DECIMAL(12,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- 注文計算のストアドプロシージャ
DELIMITER //
CREATE PROCEDURE calculate_order_total(IN order_id_param INT)
BEGIN
    DECLARE subtotal_calc DECIMAL(12,2) DEFAULT 0.00;
    DECLARE tax_calc DECIMAL(12,2) DEFAULT 0.00;
    DECLARE total_calc DECIMAL(12,2) DEFAULT 0.00;
    DECLARE tax_rate DECIMAL(5,4) DEFAULT 0.1000;  -- 10%
    
    -- 小計の計算
    SELECT COALESCE(SUM(line_total), 0.00) INTO subtotal_calc
    FROM order_items 
    WHERE order_id = order_id_param;
    
    -- 税額の計算（四捨五入）
    SET tax_calc = ROUND(subtotal_calc * tax_rate, 2);
    
    -- 合計の計算
    SELECT subtotal_calc + tax_calc + COALESCE(shipping_cost, 0.00) - COALESCE(discount_amount, 0.00)
    INTO total_calc
    FROM orders 
    WHERE order_id = order_id_param;
    
    -- 注文テーブルの更新
    UPDATE orders 
    SET subtotal = subtotal_calc,
        tax_amount = tax_calc,
        total_amount = total_calc
    WHERE order_id = order_id_param;
END //
DELIMITER ;
```

### 3. 在庫評価システム

```sql
-- ✅ 移動平均法による在庫評価
CREATE TABLE inventory_movements (
    movement_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    movement_type ENUM('in', 'out') NOT NULL,
    quantity DECIMAL(10,3) NOT NULL,
    unit_cost DECIMAL(10,4),  -- 仕入単価（高精度）
    movement_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    running_quantity DECIMAL(12,3),
    running_value DECIMAL(15,4),
    average_cost DECIMAL(10,4)
);

-- 移動平均計算のトリガー
DELIMITER //
CREATE TRIGGER calculate_moving_average 
BEFORE INSERT ON inventory_movements
FOR EACH ROW
BEGIN
    DECLARE prev_qty DECIMAL(12,3) DEFAULT 0;
    DECLARE prev_value DECIMAL(15,4) DEFAULT 0;
    DECLARE new_qty DECIMAL(12,3);
    DECLARE new_value DECIMAL(15,4);
    
    -- 前回の数量と価値を取得
    SELECT COALESCE(running_quantity, 0), COALESCE(running_value, 0)
    INTO prev_qty, prev_value
    FROM inventory_movements
    WHERE product_id = NEW.product_id
    ORDER BY movement_id DESC
    LIMIT 1;
    
    IF NEW.movement_type = 'in' THEN
        -- 入庫の場合
        SET new_qty = prev_qty + NEW.quantity;
        SET new_value = prev_value + (NEW.quantity * NEW.unit_cost);
    ELSE
        -- 出庫の場合
        SET new_qty = prev_qty - NEW.quantity;
        SET new_value = prev_value - (NEW.quantity * COALESCE(prev_value / NULLIF(prev_qty, 0), 0));
    END IF;
    
    SET NEW.running_quantity = new_qty;
    SET NEW.running_value = new_value;
    SET NEW.average_cost = CASE 
        WHEN new_qty > 0 THEN new_value / new_qty 
        ELSE 0 
    END;
END //
DELIMITER ;
```

## データベース別の対応

### MySQL
```sql
-- DECIMAL と NUMERIC は同じ
CREATE TABLE prices (
    item_id INT PRIMARY KEY,
    price DECIMAL(10,2),     -- 推奨
    old_price NUMERIC(10,2)  -- DECIMAL と同等
);

-- 計算関数
SELECT 
    ROUND(price * 1.08, 2) as price_with_tax,
    TRUNCATE(price * 1.08, 2) as truncated_price,
    CEILING(price * 1.08) as rounded_up,
    FLOOR(price * 1.08) as rounded_down
FROM prices;
```

### PostgreSQL
```sql
-- NUMERIC型（推奨）
CREATE TABLE financial_data (
    id SERIAL PRIMARY KEY,
    amount NUMERIC(15,2),    -- 推奨
    rate DECIMAL(8,4)        -- NUMERICのエイリアス
);

-- 高精度計算
SELECT 
    amount,
    amount * 1.08::NUMERIC(5,4) as with_tax,
    ROUND(amount * 1.08, 2) as rounded_tax
FROM financial_data;

-- 任意精度（注意：パフォーマンス影響）
amount NUMERIC  -- 精度制限なし
```

### Oracle
```sql
-- NUMBER型
CREATE TABLE accounts (
    account_id NUMBER PRIMARY KEY,
    balance NUMBER(15,2),    -- 15桁、小数点以下2桁
    interest_rate NUMBER(5,4) -- 5桁、小数点以下4桁
);

-- 計算例
SELECT 
    balance,
    ROUND(balance * 1.08, 2) as with_tax,
    TRUNC(balance * 1.08, 2) as truncated
FROM accounts;
```

### SQL Server
```sql
-- DECIMAL / NUMERIC型
CREATE TABLE transactions (
    transaction_id INT IDENTITY PRIMARY KEY,
    amount DECIMAL(18,2),    -- 推奨
    tax_amount NUMERIC(10,4)
);

-- MONEY型（SQL Server独有）
amount MONEY  -- 固定小数点、4桁精度、-922兆～922兆

-- 計算例
SELECT 
    amount,
    ROUND(amount * 1.08, 2) as with_tax,
    CAST(amount * 1.08 AS DECIMAL(18,2)) as casted_amount
FROM transactions;
```

## 移行とテスト

### 既存データの移行

```sql
-- ❌ 既存のFLOATデータ
CREATE TABLE old_accounts (
    account_id INT PRIMARY KEY,
    balance FLOAT
);

-- ✅ 新しいDECIMALテーブル
CREATE TABLE new_accounts (
    account_id INT PRIMARY KEY,
    balance DECIMAL(15,2)
);

-- 移行時の注意点
INSERT INTO new_accounts (account_id, balance)
SELECT 
    account_id,
    ROUND(balance, 2)  -- 四捨五入で精度調整
FROM old_accounts;

-- 移行前後の検証
SELECT 
    o.account_id,
    o.balance as old_balance,
    n.balance as new_balance,
    ABS(o.balance - n.balance) as difference
FROM old_accounts o
JOIN new_accounts n ON o.account_id = n.account_id
WHERE ABS(o.balance - n.balance) > 0.01;  -- 1円以上の差異をチェック
```

### テストケース

```sql
-- ✅ 精度テスト
SELECT 
    -- 基本演算
    1.23 + 4.56 as addition,
    10.00 - 3.33 as subtraction,
    2.50 * 3.00 as multiplication,
    10.00 / 3.00 as division,
    
    -- 四捨五入テスト
    ROUND(10.00 / 3.00, 2) as rounded_division,
    ROUND(2.505, 2) as rounded_half,
    
    -- 累積計算テスト
    (0.1 + 0.1 + 0.1 + 0.1 + 0.1 + 0.1 + 0.1 + 0.1 + 0.1 + 0.1) as ten_times_point_one;
```

## パフォーマンス考慮事項

### DECIMAL型の特徴

```sql
-- DECIMALの内部構造
-- DECIMAL(15,2) の場合：
-- - 整数部：13桁
-- - 小数部：2桁
-- - 格納：可変長（必要な桁数のみ）

-- インデックス効率
CREATE INDEX idx_account_balance ON accounts(balance);
-- DECIMAL型でもインデックスは効率的

-- 計算性能
-- DECIMAL: 正確だが計算コストは高め
-- FLOAT: 高速だが精度に問題
-- 金融システムでは精度を優先
```

### 最適化のテクニック

```sql
-- ✅ 中間計算の精度管理
SELECT 
    product_id,
    quantity,
    unit_price,
    -- 中間計算は高精度で、最終結果で丸める
    ROUND(quantity * unit_price * tax_rate, 2) as tax_amount,
    ROUND(quantity * unit_price * (1 + tax_rate), 2) as total_with_tax
FROM order_items;

-- ✅ バッチ処理での効率化
-- 大量データの集計時は、適切な精度設定で最適化
SELECT 
    DATE(transaction_date) as transaction_day,
    SUM(amount) as daily_total,  -- SUM後も DECIMAL を維持
    AVG(amount) as daily_average,
    COUNT(*) as transaction_count
FROM transactions
WHERE transaction_date >= '2024-01-01'
GROUP BY DATE(transaction_date);
```

## まとめ

### ラウンディングエラーの問題点
1. **算術演算の誤差**: 基本的な加減乗除で誤差発生
2. **等値比較の失敗**: 期待した条件分岐が動作しない
3. **累積誤差**: 計算を重ねるごとに誤差が拡大
4. **会計システムでの致命的影響**: 監査・決算で問題
5. **通貨換算の不整合**: 往復変換で元の値に戻らない

### DECIMAL型による解決
1. **正確な10進計算**: 期待通りの算術演算
2. **確実な等値比較**: 条件分岐が正常動作
3. **誤差の累積防止**: 繰り返し計算でも正確
4. **監査対応**: 会計システムで安心
5. **精度の制御**: 用途に応じた精度設定

### チェックリスト
- [ ] 金額・価格のカラムにDECIMAL型を使用しているか
- [ ] 税率・利率にも適切な精度のDECIMAL型を使用しているか
- [ ] 計算結果の丸め処理が一貫しているか
- [ ] 等値比較でイプシロン比較を使用しているか（アプリケーション側）
- [ ] 移行時のデータ検証を実施したか
- [ ] パフォーマンステストを実施したか

ラウンディングエラーは金融システムにとって致命的な問題ですが、DECIMAL型の適切な使用により完全に回避できます。システム設計時から必ず考慮し、一貫した精度管理を行うことが重要です。