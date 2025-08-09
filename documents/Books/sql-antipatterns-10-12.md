# SQLアンチパターン 第10章〜第12章

## 目次
1. [第10章 サーティワンフレーバー（31のフレーバー）](#第10章-サーティワンフレーバー31のフレーバー)
2. [第11章 ファントムファイル（幻のファイル）](#第11章-ファントムファイル幻のファイル)
3. [第12章 インデックスショットガン（闇雲インデックス）](#第12章-インデックスショットガン闇雲インデックス)

---

# 第10章 サーティワンフレーバー（31のフレーバー）

## 問題と解決策

### ❌ 問題：ENUMやCHECK制約の過度な使用

```sql
-- ❌ 悪い例：ENUM型での制限
CREATE TABLE bugs (
    bug_id INT PRIMARY KEY,
    status ENUM('NEW', 'IN_PROGRESS', 'RESOLVED', 'CLOSED'),
    priority ENUM('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')
);

-- ❌ 新しいステータス追加時の問題
-- ALTER TABLEが必要で、既存データに影響する可能性
ALTER TABLE bugs MODIFY status ENUM('NEW', 'IN_PROGRESS', 'RESOLVED', 'CLOSED', 'REOPENED');
```

**主な問題**：
- 値の変更・追加が困難
- ポータビリティが低い（MySQL特有）
- 国際化対応が困難
- 値の順序に依存した比較が発生

### ✅ 解決策：参照テーブルの活用

```sql
-- ✅ 正しい例：参照テーブル使用
CREATE TABLE bug_status (
    status_id INT PRIMARY KEY,
    status_name VARCHAR(50) NOT NULL UNIQUE,
    status_order INT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE priorities (
    priority_id INT PRIMARY KEY,
    priority_name VARCHAR(50) NOT NULL UNIQUE,
    priority_level INT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE bugs (
    bug_id INT PRIMARY KEY,
    status_id INT NOT NULL,
    priority_id INT NOT NULL,
    FOREIGN KEY (status_id) REFERENCES bug_status(status_id),
    FOREIGN KEY (priority_id) REFERENCES priorities(priority_id)
);
```

## ENUMアンチパターンとは？

ENUMアンチパターンとは、値を限定したい場合にENUM型やCHECK制約を過度に使用し、後の変更や拡張を困難にしてしまう設計パターンです。

## ❌ 問題点の詳細

### 1. 値の追加・変更の困難さ

```sql
-- ❌ ENUM値の変更は ALTER TABLE が必要
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    category ENUM('electronics', 'clothing', 'books')
);

-- 新カテゴリ追加時
ALTER TABLE products 
MODIFY category ENUM('electronics', 'clothing', 'books', 'toys');
-- → テーブルロック発生、ダウンタイムの可能性
```

### 2. 国際化の問題

```sql
-- ❌ ハードコードされた値
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    status ENUM('pending', 'shipped', 'delivered')
);

-- 日本語対応時の問題
-- ENUMの値自体を変更する必要がある
```

### 3. ポータビリティの問題

```sql
-- ❌ MySQL特有のENUM
-- PostgreSQL, Oracle, SQL Serverでは使用不可

-- CHECK制約版も同様
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    status VARCHAR(20) CHECK (status IN ('active', 'inactive'))
);
-- → 値の変更時にCHECK制約を再定義する必要
```

### 4. クエリの複雑化

```sql
-- ❌ ENUMでの複雑な条件
SELECT * FROM bugs 
WHERE status IN ('IN_PROGRESS', 'RESOLVED') 
   OR (status = 'NEW' AND priority = 'CRITICAL');

-- ステータス順序での並び替えが困難
SELECT * FROM bugs ORDER BY status;  -- アルファベット順になってしまう
```

## ✅ 参照テーブルによる解決

### 1. 柔軟な値管理

```sql
-- ✅ 参照テーブルでの管理
CREATE TABLE bug_status (
    status_id INT PRIMARY KEY,
    status_name VARCHAR(50) NOT NULL,
    status_code VARCHAR(10) NOT NULL UNIQUE,
    display_order INT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 初期データ
INSERT INTO bug_status (status_id, status_name, status_code, display_order) VALUES
(1, 'New', 'NEW', 1),
(2, 'In Progress', 'PROGRESS', 2),
(3, 'Resolved', 'RESOLVED', 3),
(4, 'Closed', 'CLOSED', 4);

-- 新しいステータス追加は簡単
INSERT INTO bug_status (status_id, status_name, status_code, display_order)
VALUES (5, 'Reopened', 'REOPENED', 3);
```

### 2. 国際化対応

```sql
-- ✅ 多言語対応テーブル
CREATE TABLE status_translations (
    status_id INT NOT NULL,
    language_code CHAR(2) NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    PRIMARY KEY (status_id, language_code),
    FOREIGN KEY (status_id) REFERENCES bug_status(status_id)
);

INSERT INTO status_translations VALUES
(1, 'en', 'New'),
(1, 'ja', '新規'),
(2, 'en', 'In Progress'),
(2, 'ja', '進行中'),
(3, 'en', 'Resolved'),
(3, 'ja', '解決済み');
```

### 3. 高度なクエリ機能

```sql
-- ✅ 参照テーブルを活用したクエリ
-- 適切な順序での表示
SELECT b.bug_id, bs.status_name, p.priority_name
FROM bugs b
JOIN bug_status bs ON b.status_id = bs.status_id
JOIN priorities p ON b.priority_id = p.priority_id
ORDER BY bs.display_order, p.priority_level DESC;

-- アクティブなステータスのみ
SELECT b.*
FROM bugs b
JOIN bug_status bs ON b.status_id = bs.status_id
WHERE bs.is_active = TRUE;

-- ステータス遷移の管理
CREATE TABLE status_transitions (
    from_status_id INT NOT NULL,
    to_status_id INT NOT NULL,
    is_allowed BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (from_status_id, to_status_id),
    FOREIGN KEY (from_status_id) REFERENCES bug_status(status_id),
    FOREIGN KEY (to_status_id) REFERENCES bug_status(status_id)
);
```

## 実装パターン

### パターン1: 基本的な参照テーブル

```sql
-- カテゴリ管理
CREATE TABLE categories (
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    category_name VARCHAR(50) NOT NULL UNIQUE,
    parent_category_id INT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (parent_category_id) REFERENCES categories(category_id)
);

-- 使用例
CREATE TABLE products (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    category_id INT NOT NULL,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);
```

### パターン2: コード付き参照テーブル

```sql
-- システム内部用コードと表示名を分離
CREATE TABLE user_roles (
    role_id INT AUTO_INCREMENT PRIMARY KEY,
    role_code VARCHAR(20) NOT NULL UNIQUE,  -- システム内部用
    role_name VARCHAR(50) NOT NULL,         -- 表示用
    description TEXT,
    permissions JSON,  -- 権限情報
    is_active BOOLEAN DEFAULT TRUE
);

INSERT INTO user_roles (role_code, role_name, description, permissions) VALUES
('ADMIN', 'Administrator', 'System administrator', '["read", "write", "delete"]'),
('EDITOR', 'Editor', 'Content editor', '["read", "write"]'),
('VIEWER', 'Viewer', 'Read-only user', '["read"]');
```

### パターン3: 階層構造の管理

```sql
-- 組織階層の管理
CREATE TABLE departments (
    dept_id INT AUTO_INCREMENT PRIMARY KEY,
    dept_code VARCHAR(10) NOT NULL UNIQUE,
    dept_name VARCHAR(100) NOT NULL,
    parent_dept_id INT NULL,
    dept_level INT NOT NULL DEFAULT 1,
    dept_path VARCHAR(255),  -- 階層パス（例：/1/3/7/）
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (parent_dept_id) REFERENCES departments(dept_id)
);

-- 階層クエリ
-- 特定部署の全下位部署を取得
SELECT d2.*
FROM departments d1
JOIN departments d2 ON d2.dept_path LIKE CONCAT(d1.dept_path, '%')
WHERE d1.dept_code = 'SALES';
```

## まとめ

### ENUMアンチパターンの問題点
1. **拡張性の欠如**: 値の追加・変更が困難
2. **ポータビリティの問題**: データベース固有の機能
3. **国際化の困難**: ハードコードされた値
4. **クエリの制限**: 柔軟な条件指定が困難

### 参照テーブルによる利点
1. **柔軟性**: 値の追加・変更が容易
2. **拡張性**: 追加属性（説明、順序、アクティブフラグ等）
3. **国際化対応**: 翻訳テーブルとの組み合わせ
4. **データ整合性**: 外部キー制約による保証

**重要**: 値の変更可能性がある場合は、ENUMよりも参照テーブルを選択することを強く推奨します。

---

# 第11章 ファントムファイル（幻のファイル）

## 問題と解決策

### ❌ 問題：ファイルパスのみをDBに保存

```sql
-- ❌ 悪い例：ファイルパスのみ保存
CREATE TABLE documents (
    doc_id INT PRIMARY KEY,
    title VARCHAR(255),
    file_path VARCHAR(500),  -- ファイルパスのみ
    uploaded_at TIMESTAMP
);

INSERT INTO documents VALUES 
(1, 'プレゼン資料', '/uploads/2024/presentation.pptx', NOW());
```

**主な問題**：
- ファイルが存在しない可能性（ファントムファイル）
- データ不整合（ファイル削除時）
- バックアップの複雑性
- セキュリティリスク

### ✅ 解決策：BLOB型の適切な使用

```sql
-- ✅ 正しい例：ファイル内容をDB内に保存
CREATE TABLE documents (
    doc_id INT PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    filename VARCHAR(255) NOT NULL,
    content_type VARCHAR(100) NOT NULL,
    file_size BIGINT NOT NULL,
    file_data LONGBLOB NOT NULL,  -- ファイル内容
    uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    checksum VARCHAR(64)  -- 整合性確認用
);
```

## ファントムファイルアンチパターンとは？

ファントムファイルアンチパターンとは、データベースにファイルの実体ではなくファイルパス（参照）のみを保存することで、データの整合性やセキュリティに問題を生じさせる設計パターンです。

## ❌ 問題点の詳細

### 1. データ整合性の問題

```sql
-- ❌ ファイルが実際に存在するかわからない
CREATE TABLE user_avatars (
    user_id INT PRIMARY KEY,
    avatar_path VARCHAR(500)
);

-- ファイルシステム上で削除されても DB には残る
-- → 参照エラー、リンク切れが発生
SELECT user_id, avatar_path FROM user_avatars;
-- 結果のパスが実際には存在しない可能性
```

### 2. バックアップ・復旧の複雑性

```sql
-- ❌ DBとファイルシステムの同期問題
-- データベースバックアップ
mysqldump mydb > backup.sql  

-- しかしファイルは別途バックアップが必要
tar -czf files_backup.tar.gz /var/www/uploads/

-- 復旧時の整合性保証が困難
-- - DBは昨日の状態
-- - ファイルは今日の状態
-- → データ不整合が発生
```

### 3. セキュリティリスク

```sql
-- ❌ パストラバーサル攻撃のリスク
CREATE TABLE attachments (
    file_id INT PRIMARY KEY,
    file_path VARCHAR(500)
);

-- 悪意あるファイルパス
INSERT INTO attachments VALUES 
(1, '../../../etc/passwd'),
(2, '..\\..\\windows\\system32\\config\\sam');

-- アプリケーションでの危険な実装
-- SELECT file_path FROM attachments WHERE file_id = ?
-- → システムファイルへの不正アクセス可能
```

### 4. 同期問題

```sql
-- ❌ ファイル操作とDB操作の非同期
BEGIN;
INSERT INTO documents (title, file_path) VALUES ('重要資料', '/uploads/doc123.pdf');
-- この時点でアプリがクラッシュ
-- → DBにはレコードあり、ファイルは未作成
-- → ファントムファイル状態
```

### 5. スケーラビリティの問題

```sql
-- ❌ 複数サーバー環境での問題
-- サーバーA: /uploads/file1.jpg に保存
-- サーバーB: /uploads/file1.jpg が存在しない
-- → ロードバランサー環境で問題発生

CREATE TABLE images (
    image_id INT,
    local_path VARCHAR(500)  -- ローカルパス保存
);
-- → サーバー間でファイル共有が必要
```

## ✅ BLOB型による解決

### 1. 完全なデータ整合性

```sql
-- ✅ ファイル内容をDB内に保存
CREATE TABLE files (
    file_id INT AUTO_INCREMENT PRIMARY KEY,
    original_name VARCHAR(255) NOT NULL,
    content_type VARCHAR(100) NOT NULL,
    file_size BIGINT NOT NULL,
    content LONGBLOB NOT NULL,
    md5_hash VARCHAR(32) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ファイル保存例（PHPでの例）
INSERT INTO files (original_name, content_type, file_size, content, md5_hash)
VALUES (?, ?, ?, ?, MD5(?));
```

### 2. バックアップ・復旧の簡素化

```sql
-- ✅ DBバックアップのみで完了
mysqldump --single-transaction mydb > complete_backup.sql

-- 復旧も簡単
mysql mydb < complete_backup.sql
-- → すべてのファイルも含めて完全復旧
```

### 3. セキュリティの向上

```sql
-- ✅ パストラバーサル攻撃の防止
-- ファイル内容がBLOB型で保存されているため
-- 悪意あるパス指定は不可能

CREATE TABLE secure_files (
    file_id INT AUTO_INCREMENT PRIMARY KEY,
    filename VARCHAR(255) NOT NULL,  -- 表示用のみ
    content LONGBLOB NOT NULL,       -- 実際の内容
    access_token VARCHAR(64) UNIQUE, -- アクセス制御
    owner_id INT NOT NULL,
    is_public BOOLEAN DEFAULT FALSE
);

-- セキュアなアクセス制御
SELECT content, content_type 
FROM secure_files 
WHERE access_token = ? AND (is_public = TRUE OR owner_id = ?);
```

## 実装パターン

### パターン1: 基本的なファイル管理

```sql
-- ✅ 包括的なファイル管理テーブル
CREATE TABLE file_storage (
    file_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    file_uuid CHAR(36) NOT NULL UNIQUE,     -- UUID for external reference
    original_filename VARCHAR(255) NOT NULL,
    file_extension VARCHAR(10),
    mime_type VARCHAR(100) NOT NULL,
    file_size BIGINT NOT NULL,
    content LONGBLOB NOT NULL,
    
    -- メタデータ
    uploaded_by INT NOT NULL,
    upload_ip VARCHAR(45),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 整合性チェック
    sha256_hash VARCHAR(64) NOT NULL,
    
    -- アクセス制御
    is_public BOOLEAN DEFAULT FALSE,
    access_count INT DEFAULT 0,
    last_accessed TIMESTAMP NULL,
    
    INDEX idx_uuid (file_uuid),
    INDEX idx_uploaded_by (uploaded_by),
    INDEX idx_created_at (created_at)
);
```

### パターン2: バージョン管理付きファイル

```sql
-- ✅ ファイルバージョン管理
CREATE TABLE documents (
    doc_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    doc_uuid CHAR(36) NOT NULL UNIQUE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    current_version_id BIGINT,
    created_by INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE document_versions (
    version_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    doc_id BIGINT NOT NULL,
    version_number INT NOT NULL,
    filename VARCHAR(255) NOT NULL,
    content_type VARCHAR(100) NOT NULL,
    file_size BIGINT NOT NULL,
    content LONGBLOB NOT NULL,
    checksum VARCHAR(64) NOT NULL,
    
    -- バージョン管理
    version_notes TEXT,
    created_by INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (doc_id) REFERENCES documents(doc_id) ON DELETE CASCADE,
    UNIQUE KEY unique_version (doc_id, version_number),
    INDEX idx_doc_version (doc_id, version_number)
);

-- 最新バージョン更新のトリガー
DELIMITER //
CREATE TRIGGER update_current_version 
AFTER INSERT ON document_versions
FOR EACH ROW
BEGIN
    UPDATE documents 
    SET current_version_id = NEW.version_id,
        updated_at = CURRENT_TIMESTAMP
    WHERE doc_id = NEW.doc_id;
END //
DELIMITER ;
```

### パターン3: 大容量ファイル対応

```sql
-- ✅ 大容量ファイル用のチャンク分割
CREATE TABLE large_files (
    file_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    file_uuid CHAR(36) NOT NULL UNIQUE,
    filename VARCHAR(255) NOT NULL,
    total_size BIGINT NOT NULL,
    chunk_size INT NOT NULL DEFAULT 1048576,  -- 1MB chunks
    total_chunks INT NOT NULL,
    mime_type VARCHAR(100),
    status ENUM('uploading', 'complete', 'failed') DEFAULT 'uploading',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE file_chunks (
    chunk_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    file_id BIGINT NOT NULL,
    chunk_number INT NOT NULL,
    chunk_data MEDIUMBLOB NOT NULL,
    chunk_size INT NOT NULL,
    checksum VARCHAR(32) NOT NULL,
    
    FOREIGN KEY (file_id) REFERENCES large_files(file_id) ON DELETE CASCADE,
    UNIQUE KEY unique_chunk (file_id, chunk_number)
);

-- ファイル完整性チェック用プロシージャ
DELIMITER //
CREATE PROCEDURE CheckFileIntegrity(IN file_uuid CHAR(36))
BEGIN
    DECLARE expected_chunks INT;
    DECLARE actual_chunks INT;
    
    SELECT total_chunks INTO expected_chunks
    FROM large_files WHERE file_uuid = file_uuid;
    
    SELECT COUNT(*) INTO actual_chunks
    FROM file_chunks fc
    JOIN large_files lf ON fc.file_id = lf.file_id
    WHERE lf.file_uuid = file_uuid;
    
    IF expected_chunks = actual_chunks THEN
        UPDATE large_files 
        SET status = 'complete' 
        WHERE file_uuid = file_uuid;
    END IF;
END //
DELIMITER ;
```

## 実用的な実装例

### アプリケーションでの使用例

```sql
-- ✅ ファイルアップロード処理
INSERT INTO file_storage (
    file_uuid, original_filename, file_extension, 
    mime_type, file_size, content, uploaded_by, sha256_hash
) VALUES (
    UUID(), 'document.pdf', 'pdf', 
    'application/pdf', 2048576, ?, 123, SHA2(?, 256)
);

-- ✅ ファイルダウンロード処理
SELECT 
    original_filename,
    mime_type,
    file_size,
    content
FROM file_storage 
WHERE file_uuid = ? AND (is_public = TRUE OR uploaded_by = ?);

-- アクセス履歴更新
UPDATE file_storage 
SET access_count = access_count + 1,
    last_accessed = CURRENT_TIMESTAMP
WHERE file_uuid = ?;
```

## パフォーマンス考慮事項

```sql
-- ✅ 大容量BLOB用の最適化
-- ファイル情報のみ取得（BLOB除外）
SELECT 
    file_id, file_uuid, original_filename, 
    mime_type, file_size, created_at
FROM file_storage
WHERE uploaded_by = ?
ORDER BY created_at DESC;

-- ✅ ファイル内容は必要時のみ取得
SELECT content 
FROM file_storage 
WHERE file_uuid = ?;

-- ✅ インデックスの最適化
-- BLOB列にはインデックス不可のため、検索用の別列を用意
ALTER TABLE file_storage 
ADD COLUMN content_preview TEXT,  -- 検索用テキスト
ADD FULLTEXT(content_preview);
```

## まとめ

### ファントムファイルアンチパターンの問題点
1. **データ整合性の欠如**: ファイルとDBの同期問題
2. **バックアップの複雑性**: ファイルシステムとDBの分離
3. **セキュリティリスク**: パストラバーサル等の脆弱性
4. **スケーラビリティ問題**: 複数サーバー環境での課題

### BLOB型使用による利点
1. **完全な整合性**: ファイルとメタデータの一体管理
2. **簡単なバックアップ**: DBバックアップのみで完了
3. **セキュリティ向上**: ファイルシステムアクセスの制限
4. **トランザクション対応**: ACIDプロパティの保証

**重要**: 中小規模のファイルはBLOB型での保存を検討し、大容量ファイルの場合はチャンク分割や外部ストレージとの組み合わせを検討してください。

---

# 第12章 インデックスショットガン（闇雲インデックス）

## 問題と解決策

### ❌ 問題：インデックスの不適切な管理

```sql
-- ❌ 悪い例：闇雲なインデックス作成
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    age INT,
    city VARCHAR(100),
    created_at TIMESTAMP
);

-- 思いつくままにインデックス作成
CREATE INDEX idx_username ON users(username);
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_first_name ON users(first_name);
CREATE INDEX idx_last_name ON users(last_name);
CREATE INDEX idx_age ON users(age);
CREATE INDEX idx_city ON users(city);
CREATE INDEX idx_created_at ON users(created_at);
CREATE INDEX idx_full_name ON users(first_name, last_name);
CREATE INDEX idx_name_age ON users(first_name, last_name, age);
-- 大量の冗長なインデックス！
```

**主な問題**：
- 過剰なインデックスによる書き込み性能劣化
- ストレージ容量の無駄
- メンテナンス負荷の増加
- 冗長なインデックスの存在

### ✅ 解決策：MENTOR原則に基づく管理

```sql
-- ✅ 正しい例：戦略的なインデックス設計
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,  -- 自然にインデックスが作成される
    email VARCHAR(100) NOT NULL UNIQUE,    -- 自然にインデックスが作成される
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    age INT,
    city VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 実際のクエリパターンに基づいたインデックス
CREATE INDEX idx_name_search ON users(last_name, first_name);  -- 名前検索用
CREATE INDEX idx_city_age ON users(city, age);                 -- 地域・年齢検索用
CREATE INDEX idx_created_recent ON users(created_at DESC);     -- 最新ユーザー取得用
```

## インデックスショットガンアンチパターンとは？

インデックスショットガンアンチパターンとは、性能問題に対して計画性なく大量のインデックスを作成し、かえって性能劣化やメンテナンス問題を引き起こす設計パターンです。

## ❌ 問題点の詳細

### 1. 書き込み性能の劣化

```sql
-- ❌ 過剰なインデックスによる書き込み負荷
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(200),
    category_id INT,
    price DECIMAL(10,2),
    stock_count INT,
    created_at TIMESTAMP
);

-- 10個のインデックス
CREATE INDEX idx1 ON products(product_name);
CREATE INDEX idx2 ON products(category_id);
CREATE INDEX idx3 ON products(price);
CREATE INDEX idx4 ON products(stock_count);
CREATE INDEX idx5 ON products(created_at);
CREATE INDEX idx6 ON products(category_id, price);
CREATE INDEX idx7 ON products(price, stock_count);
CREATE INDEX idx8 ON products(product_name, category_id);
CREATE INDEX idx9 ON products(category_id, price, stock_count);
CREATE INDEX idx10 ON products(created_at DESC, category_id);

-- INSERT時に10個のインデックスを更新する必要
INSERT INTO products VALUES (1, 'Laptop', 1, 999.99, 50, NOW());
-- → 大幅な書き込み性能劣化
```

### 2. ストレージ容量の浪費

```sql
-- ❌ 冗長なインデックスの例
CREATE INDEX idx_user_email ON users(email);           -- 3MB
CREATE INDEX idx_email_name ON users(email, first_name); -- 4MB  
CREATE INDEX idx_email_full ON users(email, first_name, last_name); -- 5MB

-- 12MBのインデックス容量が必要
-- しかし、実際に必要なのは最後の複合インデックスのみの場合が多い
```

### 3. インデックスの衝突

```sql
-- ❌ オプティマイザーの混乱
CREATE INDEX idx_created_date ON orders(created_at);
CREATE INDEX idx_created_status ON orders(created_at, status);
CREATE INDEX idx_status_created ON orders(status, created_at);

-- 同様のクエリでも異なるインデックスが使用される可能性
EXPLAIN SELECT * FROM orders WHERE created_at > '2024-01-01' AND status = 'shipped';
-- どのインデックスを使うかオプティマイザーが迷う
```

### 4. メンテナンス負荷

```sql
-- ❌ 大量のインデックス統計情報更新
-- MySQL
ANALYZE TABLE products;  -- 全インデックスの統計更新が必要

-- PostgreSQL  
VACUUM ANALYZE products;  -- 同様に全インデックス対象

-- インデックス数に比例してメンテナンス時間が増加
```

## MENTOR原則とは？

**M**easure（測定）、**E**xplain（実行計画）、**N**ominate（候補選定）、**T**est（テスト）、**O**ptimize（最適化）、**R**ebuild（再構築）の6つの原則です。

## ✅ MENTOR原則に基づく解決

### M: Measure（測定）

```sql
-- ✅ パフォーマンス測定
-- スロークエリの特定
SET long_query_time = 1;  -- 1秒以上のクエリをログ

-- クエリ実行頻度の調査
SELECT 
    sql_text,
    exec_count,
    avg_timer_wait/1000000000000 as avg_duration_sec,
    sum_timer_wait/1000000000000 as total_duration_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE schema_name = 'mydb'
ORDER BY sum_timer_wait DESC
LIMIT 20;
```

### E: Explain（実行計画）

```sql
-- ✅ 実行計画の分析
EXPLAIN FORMAT=JSON
SELECT u.username, p.product_name, o.total_amount
FROM users u
JOIN orders o ON u.user_id = o.user_id  
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE u.city = 'Tokyo' 
  AND o.created_at >= '2024-01-01'
  AND p.category_id = 1;

-- key_len, rows, filtered を確認してインデックス効果を測定
```

### N: Nominate（候補選定）

```sql
-- ✅ 戦略的インデックス候補選定

-- 1. WHERE句での検索条件
CREATE INDEX idx_user_city ON users(city);
CREATE INDEX idx_order_date ON orders(created_at);
CREATE INDEX idx_product_category ON products(category_id);

-- 2. JOIN条件（既に外部キーでインデックス存在の場合は不要）
-- users.user_id (PRIMARY KEY - 既存)
-- orders.user_id (外部キー)
-- orders.order_id (PRIMARY KEY - 既存)
-- order_items.order_id (外部キー)
-- order_items.product_id (外部キー)
-- products.product_id (PRIMARY KEY - 既存)

-- 3. 複合インデックスの検討（カーディナリティを考慮）
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);
CREATE INDEX idx_items_order_product ON order_items(order_id, product_id);
```

### T: Test（テスト）

```sql
-- ✅ インデックス効果のテスト

-- ベースライン測定
SELECT BENCHMARK(1000,
    (SELECT COUNT(*) FROM users WHERE city = 'Tokyo')
);

-- インデックス作成前後の比較
-- 作成前
EXPLAIN SELECT * FROM users WHERE city = 'Tokyo';
-- type: ALL, rows: 1000000

-- インデックス作成
CREATE INDEX idx_city ON users(city);

-- 作成後
EXPLAIN SELECT * FROM users WHERE city = 'Tokyo';
-- type: ref, rows: 5000

-- パフォーマンス改善の定量測定
```

### O: Optimize（最適化）

```sql
-- ✅ インデックス最適化

-- カーディナリティ分析
SELECT 
    column_name,
    cardinality,
    cardinality / table_rows as selectivity
FROM information_schema.statistics s
JOIN information_schema.tables t ON s.table_name = t.table_name
WHERE s.table_schema = 'mydb' AND s.table_name = 'users';

-- 低選択性カラムのインデックス削除検討
-- gender (M/F) → selectivity: 0.5 → インデックス効果低い
DROP INDEX idx_gender ON users;

-- 複合インデックスの列順序最適化
-- 選択性の高い列を先頭に配置
CREATE INDEX idx_optimized ON users(email, city, age);  -- email が最も選択性が高い
```

### R: Rebuild（再構築）

```sql
-- ✅ 定期的なインデックス再構築

-- インデックスの断片化状況確認
SELECT 
    table_name,
    index_name,
    stat_value as pages,
    stat_value * @@innodb_page_size / 1024 / 1024 as size_mb
FROM mysql.innodb_index_stats 
WHERE stat_name = 'n_leaf_pages' 
  AND database_name = 'mydb';

-- 断片化したインデックスの再構築
ALTER TABLE users DROP INDEX idx_city;
CREATE INDEX idx_city ON users(city);

-- または
OPTIMIZE TABLE users;  -- MyISAM
```

## 実装パターン

### パターン1: クエリパターン分析ベース

```sql
-- ✅ よく実行されるクエリパターンから逆算

-- パターン1: ユーザー検索
-- 頻度: 1000回/分
SELECT * FROM users WHERE username = ? OR email = ?;
-- 対応: username, email は UNIQUE制約で自動インデックス化

-- パターン2: 商品一覧表示  
-- 頻度: 500回/分
SELECT * FROM products 
WHERE category_id = ? AND price BETWEEN ? AND ?
ORDER BY created_at DESC LIMIT 20;
-- 対応: 複合インデックス
CREATE INDEX idx_category_price_created ON products(category_id, price, created_at DESC);

-- パターン3: 注文履歴表示
-- 頻度: 200回/分  
SELECT * FROM orders 
WHERE user_id = ? AND status IN ('completed', 'shipped')
ORDER BY created_at DESC;
-- 対応: 複合インデックス
CREATE INDEX idx_user_status_created ON orders(user_id, status, created_at DESC);
```

### パターン2: インデックス使用率モニタリング

```sql
-- ✅ 使用されていないインデックスの検出

-- MySQL 5.7+
SELECT 
    object_schema,
    object_name,
    index_name,
    count_read,
    count_write,
    count_read/count_write as read_write_ratio
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'mydb'
  AND count_read = 0  -- 読み取り使用回数が0
ORDER BY count_write DESC;

-- PostgreSQL
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes 
WHERE idx_scan = 0  -- 使用されていないインデックス
ORDER BY pg_relation_size(indexrelid) DESC;
```

### パターン3: 段階的インデックス構築

```sql
-- ✅ 段階的なインデックス最適化

-- フェーズ1: 必須インデックス（UNIQUE制約等）
ALTER TABLE users ADD UNIQUE KEY uk_username (username);
ALTER TABLE users ADD UNIQUE KEY uk_email (email);

-- フェーズ2: 高頻度クエリ対応
-- 測定結果: WHERE city 条件のクエリが多い
CREATE INDEX idx_city ON users(city);

-- フェーズ3: 複合条件対応  
-- 測定結果: city + age の複合条件が多い
-- 既存 idx_city を削除して複合インデックスに変更
DROP INDEX idx_city ON users;
CREATE INDEX idx_city_age ON users(city, age);

-- フェーズ4: ORDER BY 最適化
-- 測定結果: created_at DESC での並び替えが多い
CREATE INDEX idx_city_age_created ON users(city, age, created_at DESC);
DROP INDEX idx_city_age ON users;  -- 冗長なインデックスを削除
```

## インデックス選択の指針

```sql
-- ✅ インデックス作成判断基準

-- 1. カーディナリティチェック
SELECT 
    COUNT(DISTINCT column_name) / COUNT(*) as selectivity
FROM table_name;
-- selectivity < 0.1 → インデックス効果が低い

-- 2. 複合インデックスの列順序
-- a. 等価条件（=）を先に
-- b. 範囲条件（>, <, BETWEEN）を後に  
-- c. ORDER BY 列を最後に

-- 例: WHERE category_id = ? AND price > ? ORDER BY created_at
CREATE INDEX idx_optimal ON products(category_id, price, created_at);

-- 3. カバリングインデックス検討
-- SELECT product_name, price FROM products WHERE category_id = ?
CREATE INDEX idx_covering ON products(category_id, product_name, price);
-- → テーブルアクセス不要（Index Only Scan）
```

## まとめ

### インデックスショットガンアンチパターンの問題点
1. **書き込み性能劣化**: INSERT/UPDATE時の負荷増大
2. **ストレージ浪費**: 冗長なインデックスによる容量増大  
3. **メンテナンス負荷**: 統計情報更新等の作業増加
4. **オプティマイザー混乱**: 不適切なインデックス選択

### MENTOR原則による利点
1. **測定ベース**: データに基づいた意思決定
2. **計画的構築**: 戦略的なインデックス設計
3. **継続的最適化**: 定期的な見直しと改善
4. **効果測定**: 定量的な改善効果の確認

**重要**: インデックスは「多ければ良い」ではありません。実際のクエリパターンを分析し、測定に基づいた最適な数と構成を維持することが重要です。

---

SQLアンチパターン第10章から第12章の詳細な解説を作成しました。各章では問題の具体的な例、詳細な解説、そして実践的な解決策を含めています。
