# SQLアンチパターン 第22章～第25章

## 目次
1. [第22章 シー・ノー・エビル（臭いものに蓋）](#第22章-シー・ノー・エビル臭いものに蓋)
2. [第23章 ディプロマティック・イミュニティ（外交特権）](#第23章-ディプロマティック・イミュニティ外交特権)
3. [第24章 マジックビーンズ（魔法の豆）](#第24章-マジックビーンズ魔法の豆)
4. [第25章 砂の城](#第25章-砂の城)

---

## 第22章 シー・ノー・エビル（臭いものに蓋）

### 問題と解決策

#### ❌ 問題：エラーハンドリングの無視
```sql
-- ❌ エラーを無視するコード
BEGIN
    INSERT INTO users (email) VALUES ('invalid-email');
    -- エラーが発生してもそのまま続行
EXCEPTION
    WHEN OTHERS THEN
        NULL; -- 何もしない
END;

-- ❌ アプリケーション側でのエラー無視
try {
    executeQuery("DELETE FROM users WHERE id = ?", userId);
    // エラーチェックなし
} catch (SQLException e) {
    // エラーログなし、ユーザーに通知なし
}
```

**主な問題**：
- データの不整合が発生しても気づかない
- デバッグが困難になる
- システムの信頼性が低下する
- セキュリティリスクが増大する

#### ✅ 解決策：適切なエラーハンドリング
```sql
-- ✅ 適切なエラーハンドリング
BEGIN
    INSERT INTO users (email, created_at) VALUES (@email, NOW());
    COMMIT;
EXCEPTION
    WHEN constraint_violation THEN
        ROLLBACK;
        INSERT INTO error_log (error_type, message, occurred_at) 
        VALUES ('CONSTRAINT_VIOLATION', 'Invalid email format: ' + @email, NOW());
        RAISE;
    WHEN OTHERS THEN
        ROLLBACK;
        INSERT INTO error_log (error_type, message, occurred_at) 
        VALUES ('UNKNOWN_ERROR', ERROR_MESSAGE(), NOW());
        RAISE;
END;
```

**解決方法**：
1. **エラーログの記録**: 全てのエラーを適切にログ出力
2. **ユーザーへの通知**: 適切なエラーメッセージでユーザーに状況を説明
3. **トランザクション管理**: エラー時の適切なロールバック
4. **監視とアラート**: 重要なエラーの監視体制構築

### エラーハンドリングのベストプラクティス

#### 1. 構造化されたエラーログ
```sql
-- ✅ エラーログテーブル
CREATE TABLE error_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    error_code VARCHAR(50) NOT NULL,
    error_message TEXT NOT NULL,
    table_name VARCHAR(100),
    operation_type ENUM('INSERT', 'UPDATE', 'DELETE', 'SELECT'),
    user_id INT,
    session_id VARCHAR(100),
    sql_state VARCHAR(10),
    error_details JSON,
    occurred_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP NULL,
    severity ENUM('LOW', 'MEDIUM', 'HIGH', 'CRITICAL') DEFAULT 'MEDIUM',
    
    INDEX idx_occurred_at (occurred_at),
    INDEX idx_error_code (error_code),
    INDEX idx_severity (severity)
);

-- エラーログ挿入のストアドプロシージャ
DELIMITER //
CREATE PROCEDURE log_error(
    IN p_error_code VARCHAR(50),
    IN p_error_message TEXT,
    IN p_table_name VARCHAR(100),
    IN p_operation_type VARCHAR(10),
    IN p_user_id INT,
    IN p_severity VARCHAR(10)
)
BEGIN
    INSERT INTO error_log (
        error_code, error_message, table_name, 
        operation_type, user_id, severity
    ) VALUES (
        p_error_code, p_error_message, p_table_name, 
        p_operation_type, p_user_id, p_severity
    );
END //
DELIMITER ;
```

#### 2. 制約違反の適切な処理
```sql
-- ✅ 制約違反時の処理例
DELIMITER //
CREATE PROCEDURE safe_user_insert(
    IN p_email VARCHAR(255),
    IN p_username VARCHAR(100),
    OUT p_result VARCHAR(20),
    OUT p_message TEXT
)
BEGIN
    DECLARE EXIT HANDLER FOR 1062 -- Duplicate entry
    BEGIN
        SET p_result = 'ERROR';
        SET p_message = 'ユーザー名またはメールアドレスが既に使用されています';
        CALL log_error('DUPLICATE_USER', p_message, 'users', 'INSERT', NULL, 'MEDIUM');
    END;
    
    DECLARE EXIT HANDLER FOR 1406 -- Data too long
    BEGIN
        SET p_result = 'ERROR';
        SET p_message = '入力データが長すぎます';
        CALL log_error('DATA_TOO_LONG', p_message, 'users', 'INSERT', NULL, 'LOW');
    END;
    
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        SET p_result = 'ERROR';
        SET p_message = 'データベースエラーが発生しました';
        CALL log_error('SQL_EXCEPTION', p_message, 'users', 'INSERT', NULL, 'HIGH');
    END;
    
    INSERT INTO users (email, username, created_at) 
    VALUES (p_email, p_username, NOW());
    
    SET p_result = 'SUCCESS';
    SET p_message = 'ユーザーが正常に作成されました';
END //
DELIMITER ;
```

#### 3. アプリケーション層での適切な処理
```java
// ✅ Java での適切なエラーハンドリング
public class UserService {
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);
    
    public UserResult createUser(String email, String username) {
        try (Connection conn = dataSource.getConnection()) {
            CallableStatement stmt = conn.prepareCall("{CALL safe_user_insert(?, ?, ?, ?)}");
            stmt.setString(1, email);
            stmt.setString(2, username);
            stmt.registerOutParameter(3, Types.VARCHAR);
            stmt.registerOutParameter(4, Types.VARCHAR);
            
            stmt.execute();
            
            String result = stmt.getString(3);
            String message = stmt.getString(4);
            
            if ("SUCCESS".equals(result)) {
                logger.info("User created successfully: {}", email);
                return UserResult.success(message);
            } else {
                logger.warn("User creation failed: {} - {}", email, message);
                return UserResult.error(message);
            }
            
        } catch (SQLException e) {
            logger.error("Database error during user creation", e);
            return UserResult.error("システムエラーが発生しました。管理者にお問い合わせください。");
        } catch (Exception e) {
            logger.error("Unexpected error during user creation", e);
            return UserResult.error("予期しないエラーが発生しました。");
        }
    }
}
```

---

## 第23章 ディプロマティック・イミュニティ（外交特権）

### 問題と解決策

#### ❌ 問題：SQL品質管理の軽視
```sql
-- ❌ 品質管理を軽視したコード例
-- 1. コードレビューなし
CREATE VIEW monthly_sales AS
SELECT user_id, SUM(amount), AVG(amount)  -- ❌ カラム名が不明確
FROM orders 
WHERE status = 'completed'  -- ❌ ハードコーディング
GROUP BY user_id;  -- ❌ 月の概念がない

-- 2. テストなし
-- 3. ドキュメントなし
-- 4. パフォーマンステストなし
```

**主な問題**：
- SQLコードの品質が低下する
- メンテナンスが困難になる
- バグの発見が遅れる
- パフォーマンス問題が見過ごされる

#### ✅ 解決策：SQL品質管理の実装
```sql
-- ✅ 品質を考慮したコード例
CREATE VIEW monthly_sales_summary AS
SELECT 
    u.user_id,
    u.username,
    DATE_FORMAT(o.created_at, '%Y-%m') as sales_month,
    COUNT(*) as order_count,
    SUM(o.total_amount) as total_sales,
    AVG(o.total_amount) as average_order_value,
    MIN(o.total_amount) as min_order_value,
    MAX(o.total_amount) as max_order_value
FROM orders o
JOIN users u ON o.user_id = u.user_id
WHERE o.status = (SELECT status_code FROM order_statuses WHERE status_name = 'completed')
    AND o.created_at >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
GROUP BY u.user_id, u.username, DATE_FORMAT(o.created_at, '%Y-%m')
ORDER BY sales_month DESC, total_sales DESC;

-- インデックス最適化
CREATE INDEX idx_orders_status_created ON orders(status, created_at);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);
```

### SQL品質管理のベストプラクティス

#### 1. SQLコードレビューチェックリスト
```sql
-- ✅ レビュー項目の例

-- a) 命名規則の確認
CREATE TABLE user_profiles (  -- ✅ 分かりやすいテーブル名
    user_profile_id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- ✅ 一意で分かりやすいID
    user_id BIGINT NOT NULL,  -- ✅ 外部キーの命名一貫性
    first_name VARCHAR(100) NOT NULL,  -- ✅ 適切なサイズ設定
    last_name VARCHAR(100) NOT NULL,
    date_of_birth DATE,  -- ✅ 適切なデータ型
    profile_image_url TEXT,  -- ✅ URL用にTEXT型
    is_public BOOLEAN DEFAULT FALSE,  -- ✅ デフォルト値設定
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  -- ✅ 監査カラム
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- ✅ 適切な制約
    CONSTRAINT fk_user_profiles_user_id 
        FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    CONSTRAINT chk_date_of_birth 
        CHECK (date_of_birth <= CURRENT_DATE),
    
    -- ✅ 必要なインデックス
    INDEX idx_user_profiles_user_id (user_id),
    INDEX idx_user_profiles_is_public (is_public)
);
```

#### 2. SQLテストの実装
```sql
-- ✅ SQLテストの例（テストデータ準備）
CREATE TEMPORARY TABLE test_users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(100),
    email VARCHAR(255),
    created_at TIMESTAMP
);

CREATE TEMPORARY TABLE test_orders (
    order_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    created_at TIMESTAMP
);

-- テストデータ挿入
INSERT INTO test_users VALUES 
(1, 'testuser1', 'test1@example.com', '2024-01-15 10:00:00'),
(2, 'testuser2', 'test2@example.com', '2024-01-20 11:00:00'),
(3, 'testuser3', 'test3@example.com', '2024-02-01 12:00:00');

INSERT INTO test_orders VALUES 
(1, 1, 100.00, 'completed', '2024-01-16 10:00:00'),
(2, 1, 150.00, 'completed', '2024-01-17 10:00:00'),
(3, 2, 200.00, 'completed', '2024-01-21 10:00:00'),
(4, 3, 75.00, 'pending', '2024-02-02 10:00:00');

-- テストクエリ実行
SELECT 
    'User Sales Test' as test_name,
    user_id,
    total_sales,
    CASE 
        WHEN user_id = 1 AND total_sales = 250.00 THEN 'PASS'
        WHEN user_id = 2 AND total_sales = 200.00 THEN 'PASS'
        WHEN user_id = 3 AND total_sales = 0.00 THEN 'PASS'  -- pending orders excluded
        ELSE 'FAIL'
    END as test_result
FROM (
    SELECT 
        u.user_id,
        COALESCE(SUM(o.total_amount), 0) as total_sales
    FROM test_users u
    LEFT JOIN test_orders o ON u.user_id = o.user_id AND o.status = 'completed'
    GROUP BY u.user_id
) sales_data;
```

#### 3. パフォーマンス管理
```sql
-- ✅ パフォーマンス監視クエリ
CREATE TABLE query_performance_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    query_hash VARCHAR(64) NOT NULL,  -- クエリのハッシュ値
    query_text TEXT NOT NULL,
    execution_time_ms INT NOT NULL,
    rows_examined INT,
    rows_returned INT,
    index_usage JSON,  -- 使用されたインデックス情報
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_query_hash (query_hash),
    INDEX idx_execution_time (execution_time_ms),
    INDEX idx_executed_at (executed_at)
);

-- スロークエリの監視
SELECT 
    query_hash,
    COUNT(*) as execution_count,
    AVG(execution_time_ms) as avg_time,
    MAX(execution_time_ms) as max_time,
    AVG(rows_examined) as avg_rows_examined
FROM query_performance_log
WHERE executed_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
    AND execution_time_ms > 1000  -- 1秒以上
GROUP BY query_hash
ORDER BY avg_time DESC;
```

---

## 第24章 マジックビーンズ（魔法の豆）

### 問題と解決策

#### ❌ 問題：ActiveRecordパターンの過度な依存
```ruby
# ❌ ActiveRecordパターンの問題例
class Order < ActiveRecord::Base
  has_many :order_items
  belongs_to :user
  
  # ❌ N+1クエリ問題
  def calculate_total
    total = 0
    order_items.each do |item|  # 各アイテムごとにクエリが実行される
      total += item.price * item.quantity
    end
    total
  end
  
  # ❌ ビジネスロジックがモデルに集中
  def apply_discount(discount_code)
    discount = Discount.find_by(code: discount_code)  # 毎回DB検索
    if discount && discount.valid?
      self.discount_amount = calculate_discount(discount)
    end
  end
end

# ❌ 非効率なデータ取得
orders = Order.includes(:user).limit(10)
orders.each do |order|
  puts "#{order.user.name}: #{order.calculate_total}"  # 各注文で複数クエリ
end
```

**主な問題**：
- N+1クエリ問題が発生しやすい
- SQLの複雑なクエリが書きにくい
- パフォーマンスが悪化する
- ビジネスロジックがモデルに集中する

#### ✅ 解決策：適切なデータアクセス設計
```ruby
# ✅ 改善されたアプローチ
class OrderService
  def self.orders_with_totals(limit = 10)
    # 1回のSQLで必要なデータを全て取得
    sql = <<~SQL
      SELECT 
        o.id,
        o.user_id,
        u.name as user_name,
        o.created_at,
        SUM(oi.price * oi.quantity) as total_amount,
        COUNT(oi.id) as item_count
      FROM orders o
      JOIN users u ON o.user_id = u.id
      JOIN order_items oi ON o.id = oi.order_id
      WHERE o.created_at >= ?
      GROUP BY o.id, o.user_id, u.name, o.created_at
      ORDER BY o.created_at DESC
      LIMIT ?
    SQL
    
    ActiveRecord::Base.connection.exec_query(
      sql, 
      'orders_with_totals',
      [1.month.ago, limit]
    ).to_a
  end
  
  def self.apply_bulk_discount(order_ids, discount_code)
    # バルク更新でパフォーマンス改善
    discount = DiscountService.find_valid_discount(discount_code)
    return false unless discount
    
    ActiveRecord::Base.transaction do
      # 1つのUPDATE文で一括更新
      Order.where(id: order_ids).update_all([
        "discount_amount = total_amount * ?", 
        discount.percentage / 100.0
      ])
      
      # 割引適用履歴をバルクで挿入
      discount_applications = order_ids.map do |order_id|
        {
          order_id: order_id,
          discount_id: discount.id,
          applied_at: Time.current
        }
      end
      
      DiscountApplication.insert_all(discount_applications)
    end
  end
end
```

### ActiveRecordの適切な使用方法

#### 1. クエリ最適化
```ruby
# ✅ Eager Loadingの活用
class OrderController < ApplicationController
  def index
    # 必要な関連データを事前読み込み
    @orders = Order.includes(:user, :order_items => :product)
                  .where(created_at: 1.month.ago..Time.current)
                  .order(created_at: :desc)
                  .limit(20)
    
    # ビューで使用する集計データを事前計算
    @order_summaries = @orders.map do |order|
      {
        id: order.id,
        user_name: order.user.name,
        total_amount: order.order_items.sum { |item| item.price * item.quantity },
        item_count: order.order_items.size
      }
    end
  end
  
  def sales_report
    # 複雑な集計は生SQLで実行
    @monthly_sales = ActiveRecord::Base.connection.exec_query(<<~SQL)
      SELECT 
        DATE_FORMAT(created_at, '%Y-%m') as month,
        COUNT(*) as order_count,
        SUM(total_amount) as total_sales,
        AVG(total_amount) as average_order_value
      FROM orders
      WHERE created_at >= DATE_SUB(CURRENT_DATE, INTERVAL 12 MONTH)
      GROUP BY DATE_FORMAT(created_at, '%Y-%m')
      ORDER BY month DESC
    SQL
  end
end
```

#### 2. データアクセス層の分離
```ruby
# ✅ Repository パターンの導入
class OrderRepository
  def self.find_with_items(order_id)
    Order.includes(:order_items => [:product])
         .find(order_id)
  end
  
  def self.recent_orders_with_totals(days = 30, limit = 50)
    sql = <<~SQL
      SELECT 
        o.*,
        u.name as user_name,
        u.email as user_email,
        order_totals.total_amount,
        order_totals.item_count
      FROM orders o
      JOIN users u ON o.user_id = u.id
      JOIN (
        SELECT 
          order_id,
          SUM(price * quantity) as total_amount,
          COUNT(*) as item_count
        FROM order_items
        GROUP BY order_id
      ) order_totals ON o.id = order_totals.order_id
      WHERE o.created_at >= DATE_SUB(CURRENT_DATE, INTERVAL ? DAY)
      ORDER BY o.created_at DESC
      LIMIT ?
    SQL
    
    ActiveRecord::Base.connection.exec_query(sql, 'recent_orders', [days, limit])
  end
  
  def self.bulk_update_status(order_ids, new_status)
    Order.where(id: order_ids).update_all(
      status: new_status,
      updated_at: Time.current
    )
    
    # ステータス変更履歴を記録
    status_changes = order_ids.map do |order_id|
      {
        order_id: order_id,
        new_status: new_status,
        changed_at: Time.current
      }
    end
    
    OrderStatusHistory.insert_all(status_changes)
  end
end
```

#### 3. パフォーマンス監視
```ruby
# ✅ クエリ監視の実装
class QueryMonitor
  def self.monitor_slow_queries
    ActiveSupport::Notifications.subscribe 'sql.active_record' do |*args|
      event = ActiveSupport::Notifications::Event.new(*args)
      
      if event.duration > 1000  # 1秒以上のクエリ
        Rails.logger.warn(
          "Slow Query Detected: #{event.duration}ms - #{event.payload[:sql]}"
        )
        
        # パフォーマンス改善のため詳細ログを記録
        SlowQueryLog.create!(
          sql: event.payload[:sql],
          duration_ms: event.duration,
          connection_id: event.payload[:connection_id],
          occurred_at: Time.current
        )
      end
    end
  end
end
```

---

## 第25章 砂の城

### 問題と解決策

#### ❌ 問題：運用計画の不備
```sql
-- ❌ 運用を考慮していないテーブル設計例
CREATE TABLE user_activity_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    activity_type VARCHAR(50),
    activity_data TEXT,  -- JSON形式のデータ
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- ❌ パーティショニングなし
-- ❌ アーカイブ戦略なし
-- ❌ インデックス戦略が不明確
-- ❌ データ増加の見積もりなし
```

**主な問題**：
- データ量の急激な増加への対応ができない
- パフォーマンスが時間とともに劣化する
- バックアップ時間が増大する
- ストレージコストが制御不能になる

#### ✅ 解決策：運用を考慮した設計
```sql
-- ✅ 運用を考慮したテーブル設計
CREATE TABLE user_activity_log (
    id BIGINT AUTO_INCREMENT,
    user_id INT NOT NULL,
    activity_type ENUM('login', 'logout', 'page_view', 'purchase', 'search') NOT NULL,
    activity_data JSON,
    ip_address VARCHAR(45),  -- IPv6対応
    user_agent TEXT,
    created_at TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),  -- ミリ秒精度
    created_date DATE GENERATED ALWAYS AS (DATE(created_at)) STORED,  -- パーティションキー用
    
    PRIMARY KEY (id, created_date),  -- パーティションキーを含める
    INDEX idx_user_activity_user_id (user_id, created_date),
    INDEX idx_user_activity_type (activity_type, created_date),
    INDEX idx_user_activity_created (created_at)
) 
-- 日別パーティショニング
PARTITION BY RANGE (TO_DAYS(created_date)) (
    PARTITION p_2024_01 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p_2024_02 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    PARTITION p_2024_03 VALUES LESS THAN (TO_DAYS('2024-04-01')),
    PARTITION p_2024_04 VALUES LESS THAN (TO_DAYS('2024-05-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### 運用計画のベストプラクティス

#### 1. データライフサイクル管理
```sql
-- ✅ アーカイブテーブルの設計
CREATE TABLE user_activity_log_archive (
    id BIGINT NOT NULL,
    user_id INT NOT NULL,
    activity_type ENUM('login', 'logout', 'page_view', 'purchase', 'search') NOT NULL,
    activity_data JSON,
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMP(3) NOT NULL,
    archived_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (id, created_at),  -- 複合主キー
    INDEX idx_archive_user_id (user_id, created_at),
    INDEX idx_archive_archived (archived_at)
) 
ENGINE=MyISAM  -- 圧縮効率のためMyISAM使用
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p_2022 VALUES LESS THAN (2023),
    PARTITION p_2023 VALUES LESS THAN (2024),
    PARTITION p_2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- アーカイブ処理のストアドプロシージャ
DELIMITER //
CREATE PROCEDURE archive_old_activity_logs(IN days_to_keep INT DEFAULT 90)
BEGIN
    DECLARE archive_cutoff_date DATE;
    DECLARE rows_archived INT DEFAULT 0;
    
    -- アーカイブ対象日の計算
    SET archive_cutoff_date = DATE_SUB(CURRENT_DATE, INTERVAL days_to_keep DAY);
    
    -- トランザクション開始
    START TRANSACTION;
    
    -- 古いデータをアーカイブテーブルに移動
    INSERT INTO user_activity_log_archive (
        id, user_id, activity_type, activity_data, 
        ip_address, user_agent, created_at
    )
    SELECT 
        id, user_id, activity_type, activity_data, 
        ip_address, user_agent, created_at
    FROM user_activity_log
    WHERE created_date < archive_cutoff_date
    LIMIT 10000;  -- バッチサイズ制限
    
    SET rows_archived = ROW_COUNT();
    
    -- 元テーブルから削除
    DELETE FROM user_activity_log 
    WHERE created_date < archive_cutoff_date 
    LIMIT 10000;
    
    COMMIT;
    
    -- アーカイブ処理のログ
    INSERT INTO maintenance_log (
        operation_type, 
        table_name, 
        rows_affected, 
        operation_date
    ) VALUES (
        'ARCHIVE', 
        'user_activity_log', 
        rows_archived, 
        NOW()
    );
    
    SELECT CONCAT(rows_archived, ' rows archived from user_activity_log') as result;
END //
DELIMITER ;
```

#### 2. パフォーマンス監視と最適化
```sql
-- ✅ パフォーマンス監視テーブル
CREATE TABLE table_performance_metrics (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    operation_type ENUM('SELECT', 'INSERT', 'UPDATE', 'DELETE') NOT NULL,
    avg_execution_time_ms DECIMAL(8,2) NOT NULL,
    max_execution_time_ms DECIMAL(8,2) NOT NULL,
    query_count INT NOT NULL,
    rows_examined BIGINT,
    rows_affected BIGINT,
    index_usage_ratio DECIMAL(5,2),  -- インデックス使用率
    measured_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_table_metrics_table (table_name, measured_at),
    INDEX idx_table_metrics_performance (avg_execution_time_ms DESC)
);

-- 定期的な統計情報更新
DELIMITER //
CREATE PROCEDURE update_table_statistics()
BEGIN
    -- テーブルサイズの監視
    INSERT INTO table_size_history (table_name, size_mb, row_count, recorded_at)
    SELECT 
        table_name,
        ROUND(((data_length + index_length) / 1024 / 1024), 2) AS size_mb,
        table_rows,
        NOW()
    FROM information_schema.TABLES 
    WHERE table_schema = DATABASE()
        AND table_type = 'BASE TABLE';
    
    -- インデックスの効果測定
    INSERT INTO index_effectiveness_log (
        table_name, index_name, cardinality, 
        usage_count, last_used, effectiveness_score
    )
    SELECT 
        s.table_name,
        s.index_name,
        s.cardinality,
        COALESCE(usage_stats.usage_count, 0),
        usage_stats.last_used,
        CASE 
            WHEN s.cardinality > 0 AND usage_stats.usage_count > 0 
            THEN (usage_stats.usage_count / s.cardinality) * 100
            ELSE 0 
        END as effectiveness_score
    FROM information_schema.STATISTICS s
    LEFT JOIN (
        SELECT table_name, index_name, 
               COUNT(*) as usage_count,
               MAX(last_query_time) as last_used
        FROM performance_schema.table_io_waits_summary_by_index_usage
        GROUP BY table_name, index_name
    ) usage_stats ON s.table_name = usage_stats.table_name 
                 AND s.index_name = usage_stats.index_name
    WHERE s.table_schema = DATABASE();
    
END //
DELIMITER ;

-- 毎日実行するイベント
CREATE EVENT daily_maintenance
ON SCHEDULE EVERY 1 DAY
STARTS '2024-01-01 02:00:00'
DO
BEGIN
    CALL update_table_statistics();
    CALL archive_old_activity_logs(90);
    ANALYZE TABLE user_activity_log;
    OPTIMIZE TABLE user_activity_log_archive;
END;
```

#### 3. 障害対策とリカバリ計画
```sql
-- ✅ バックアップ戦略の実装
CREATE TABLE backup_schedule (
    id INT AUTO_INCREMENT PRIMARY KEY,
    backup_type ENUM('FULL', 'INCREMENTAL', 'DIFFERENTIAL') NOT NULL,
    table_name VARCHAR(100),
    backup_frequency VARCHAR(50) NOT NULL,  -- 'daily', 'weekly', 'monthly'
    retention_days INT NOT NULL,
    backup_command TEXT,
    last_backup TIMESTAMP NULL,
    next_backup TIMESTAMP NULL,
    is_active BOOLEAN DEFAULT TRUE,
    
    INDEX idx_backup_schedule_next (next_backup, is_active)
);

-- バックアップスケジュール設定
INSERT INTO backup_schedule (backup_type, table_name, backup_frequency, retention_days, backup_command) VALUES
('FULL', NULL, 'weekly', 30, 'mysqldump --all-databases --single-transaction --routines --triggers'),
('INCREMENTAL', 'user_activity_log', 'daily', 7, 'mysqlbinlog --start-datetime="yesterday"'),
('FULL', 'user_activity_log_archive', 'monthly', 365, 'mysqldump --single-transaction');

-- 災害復旧手順の文書化
CREATE TABLE disaster_recovery_procedures (
    id INT AUTO_INCREMENT PRIMARY KEY,
    scenario_name VARCHAR(200) NOT NULL,
    severity_level ENUM('LOW', 'MEDIUM', 'HIGH', 'CRITICAL') NOT NULL,
    estimated_rto_minutes INT,  -- Recovery Time Objective
    estimated_rpo_minutes INT,  -- Recovery Point Objective
    procedure_steps TEXT NOT NULL,
    required_resources TEXT,
    contact_list TEXT,
    last_tested TIMESTAMP NULL,
    test_results TEXT,
    
    INDEX idx_recovery_severity (severity_level)
);

INSERT INTO disaster_recovery_procedures (
    scenario_name, severity_level, estimated_rto_minutes, estimated_rpo_minutes,
    procedure_steps, required_resources
) VALUES (
    'Primary Database Server Failure', 'CRITICAL', 30, 5,
    '1. Confirm primary server status\n2. Promote standby to master\n3. Update application configuration\n4. Verify data consistency\n5. Monitor system stability',
    'Standby server, Network access, Administrative privileges'
);
```

#### 4. 容量計画と拡張戦略
```sql
-- ✅ 容量計画テーブル
CREATE TABLE capacity_planning (
    id INT AUTO_INCREMENT PRIMARY KEY,
    resource_type ENUM('DATABASE_SIZE', 'TABLE_ROWS', 'INDEX_SIZE', 'QUERY_VOLUME') NOT NULL,
    resource_name VARCHAR(100) NOT NULL,
    current_value BIGINT NOT NULL,
    growth_rate_per_month DECIMAL(5,2),  -- 月間成長率（%）
    predicted_6_months BIGINT,
    predicted_12_months BIGINT,
    warning_threshold BIGINT,
    critical_threshold BIGINT,
    recommended_action TEXT,
    measured_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_capacity_planning_resource (resource_type, resource_name),
    INDEX idx_capacity_planning_thresholds (current_value, warning_threshold, critical_threshold)
);

-- 容量予測の更新プロシージャ
DELIMITER //
CREATE PROCEDURE update_capacity_predictions()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE current_size BIGINT;
    DECLARE growth_rate DECIMAL(5,2);
    DECLARE resource_name VARCHAR(100);
    
    DECLARE capacity_cursor CURSOR FOR
        SELECT resource_name, current_value, growth_rate_per_month
        FROM capacity_planning
        WHERE resource_type = 'DATABASE_SIZE';
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN capacity_cursor;
    
    read_loop: LOOP
        FETCH capacity_cursor INTO resource_name, current_size, growth_rate;
        IF done THEN
            LEAVE read_loop;
        END IF;
        
        -- 6ヶ月後と12ヶ月後の予測値を計算
        UPDATE capacity_planning 
        SET 
            predicted_6_months = ROUND(current_value * POWER(1 + (growth_rate/100), 6)),
            predicted_12_months = ROUND(current_value * POWER(1 + (growth_rate/100), 12))
        WHERE resource_name = resource_name AND resource_type = 'DATABASE_SIZE';
        
    END LOOP;
    
    CLOSE capacity_cursor;
    
    -- 閾値を超える項目をアラート
    INSERT INTO system_alerts (alert_type, message, severity, created_at)
    SELECT 
        'CAPACITY_WARNING',
        CONCAT('Resource ', resource_name, ' is approaching warning threshold'),
        'MEDIUM',
        NOW()
    FROM capacity_planning
    WHERE current_value >= warning_threshold
        AND current_value < critical_threshold;
    
    INSERT INTO system_alerts (alert_type, message, severity, created_at)
    SELECT 
        'CAPACITY_CRITICAL',
        CONCAT('Resource ', resource_name, ' has exceeded critical threshold'),
        'HIGH',
        NOW()
    FROM capacity_planning
    WHERE current_value >= critical_threshold;
    
END //
DELIMITER ;
```

### まとめ

各章で紹介した問題とその解決策は、実際の開発・運用現場でよく遭遇する課題です。

**第22章 シー・ノー・エビル**：
- エラーを無視することの危険性
- 適切なエラーハンドリングとログ記録の重要性

**第23章 ディプロマティック・イミュニティ**：
- SQL品質管理の軽視による長期的な問題
- コードレビュー、テスト、監視の体制化

**第24章 マジックビーンズ**：
- ActiveRecordパターンの過度な依存問題
- パフォーマンスと保守性のバランス

**第25章 砂の城**：
- 運用計画の不備による長期的な問題
- データライフサイクル、容量計画、災害復旧の重要性

これらの問題を回避するためには、設計段階から運用面を考慮し、継続的な改善を行うことが重要です。