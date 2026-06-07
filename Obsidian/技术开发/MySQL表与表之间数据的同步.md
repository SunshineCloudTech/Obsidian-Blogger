---
title: MySQL表与表之间数据同步的完整指南
date: 2024-04-30
tags:
  - MySQL
  - 数据同步
  - 触发器
  - 主从复制
  - 数据一致性
description: 详细介绍MySQL中实现表与表之间数据同步的多种方法，包括触发器、复制、第三方工具等，并提供实际业务场景案例。
icon: arrows-rotate
category:
  - 运维笔记
  - 数据库
index: true
cover: https://picsum.photos/1200/600?random=20
---
# MySQL表与表之间数据同步的完整指南

## 概述

MySQL表与表之间的数据同步是数据库管理中的重要需求，涉及实时同步、批量同步、增量同步等多种场景。本文详细介绍各种数据同步方法的原理、实现和最佳实践。

## 目录

- [数据同步基础概念](#数据同步基础概念)
- [触发器实现同步](#触发器实现同步)
- [主从复制同步](#主从复制同步)
- [应用程序逻辑同步](#应用程序逻辑同步)
- [定时任务同步](#定时任务同步)
- [第三方工具同步](#第三方工具同步)
- [实际业务案例](#实际业务案例)
- [性能优化策略](#性能优化策略)
- [最佳实践](#最佳实践)

## 数据同步基础概念

### 同步类型分类

| 同步类型 | 特点 | 适用场景 | 优缺点 |
|----------|------|----------|--------|
| **实时同步** | 数据变更立即同步 | 高一致性要求 | 高性能开销，低延迟 |
| **准实时同步** | 秒级或分钟级延迟 | 一般业务场景 | 平衡性能与一致性 |
| **批量同步** | 定时批量处理 | 报表、分析场景 | 低开销，可接受延迟 |
| **增量同步** | 只同步变更数据 | 大数据表 | 高效率，复杂逻辑 |

### 数据一致性级别

```sql
-- 强一致性：事务级别同步
START TRANSACTION;
UPDATE source_table SET status = 'processed' WHERE id = 1;
INSERT INTO target_table (source_id, data) VALUES (1, 'processed_data');
COMMIT;

-- 最终一致性：异步同步，允许短暂不一致
-- 通过定时任务或消息队列实现

-- 弱一致性：无强制一致性保证
-- 通过日志回放或定期全量同步
```

## 触发器实现同步

### 基础触发器同步

#### 1. 单向同步触发器

```sql
-- 创建源表和目标表
CREATE TABLE source_products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2),
    stock_quantity INT,
    status ENUM('active', 'inactive') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE target_products_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    source_id INT,
    name VARCHAR(100),
    price DECIMAL(10,2),
    action ENUM('insert', 'update', 'delete'),
    sync_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- INSERT触发器
DELIMITER //
CREATE TRIGGER tr_products_after_insert
AFTER INSERT ON source_products
FOR EACH ROW
BEGIN
    INSERT INTO target_products_log (source_id, name, price, action)
    VALUES (NEW.id, NEW.name, NEW.price, 'insert');
END //

-- UPDATE触发器
CREATE TRIGGER tr_products_after_update
AFTER UPDATE ON source_products
FOR EACH ROW
BEGIN
    INSERT INTO target_products_log (source_id, name, price, action)
    VALUES (NEW.id, NEW.name, NEW.price, 'update');
END //

-- DELETE触发器
CREATE TRIGGER tr_products_after_delete
AFTER DELETE ON source_products
FOR EACH ROW
BEGIN
    INSERT INTO target_products_log (source_id, name, price, action)
    VALUES (OLD.id, OLD.name, OLD.price, 'delete');
END //
DELIMITER ;
```

#### 2. 条件触发器同步

```sql
-- 只有特定条件下才同步
DELIMITER //
CREATE TRIGGER tr_products_conditional_sync
AFTER UPDATE ON source_products
FOR EACH ROW
BEGIN
    -- 只有价格变化且状态为active时才同步
    IF OLD.price != NEW.price AND NEW.status = 'active' THEN
        INSERT INTO target_products_log (source_id, name, price, action)
        VALUES (NEW.id, NEW.name, NEW.price, 'price_update');
    END IF;
    
    -- 状态变为inactive时同步删除
    IF OLD.status = 'active' AND NEW.status = 'inactive' THEN
        DELETE FROM target_active_products WHERE source_id = NEW.id;
    END IF;
END //
DELIMITER ;
```

### 高级触发器功能

#### 1. 错误处理和回滚

```sql
DELIMITER //
CREATE TRIGGER tr_products_safe_sync
AFTER INSERT ON source_products
FOR EACH ROW
BEGIN
    DECLARE exit_code INT DEFAULT 0;
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION 
    BEGIN
        SET exit_code = 1;
        -- 记录错误日志
        INSERT INTO sync_error_log (table_name, operation, error_message, created_at)
        VALUES ('source_products', 'insert', 'Sync failed', NOW());
    END;
    
    -- 尝试同步
    INSERT INTO target_products_log (source_id, name, price, action)
    VALUES (NEW.id, NEW.name, NEW.price, 'insert');
    
    IF exit_code = 1 THEN
        -- 可以选择抛出异常或仅记录错误
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Sync operation failed';
    END IF;
END //
DELIMITER ;
```

#### 2. 批量操作优化

```sql
-- 创建临时同步表避免逐行触发
DELIMITER //
CREATE TRIGGER tr_products_batch_sync
AFTER INSERT ON source_products
FOR EACH ROW
BEGIN
    -- 将需要同步的数据插入临时表
    INSERT INTO temp_sync_queue (source_table, source_id, operation, data)
    VALUES ('source_products', NEW.id, 'insert', JSON_OBJECT(
        'id', NEW.id,
        'name', NEW.name,
        'price', NEW.price
    ));
END //
DELIMITER ;

-- 定时处理同步队列
CREATE EVENT ev_process_sync_queue
ON SCHEDULE EVERY 30 SECOND
DO
BEGIN
    -- 批量处理同步队列
    INSERT INTO target_products_log (source_id, name, price, action)
    SELECT 
        JSON_EXTRACT(data, '$.id'),
        JSON_EXTRACT(data, '$.name'),
        JSON_EXTRACT(data, '$.price'),
        operation
    FROM temp_sync_queue
    WHERE source_table = 'source_products';
    
    -- 清理已处理的队列
    DELETE FROM temp_sync_queue 
    WHERE source_table = 'source_products' 
      AND created_at < NOW() - INTERVAL 1 MINUTE;
END;
```

## 主从复制同步

### 传统主从复制

#### 1. 配置主服务器

```bash
# 主服务器配置 (my.cnf)
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-format=ROW
binlog-do-db=sync_database

# 重启MySQL服务
sudo systemctl restart mysql

# 创建复制用户
mysql -u root -p
```

```sql
-- 在主服务器上创建复制用户
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'strong_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;

-- 查看主服务器状态
SHOW MASTER STATUS;
-- 记录File和Position的值
```

#### 2. 配置从服务器

```bash
# 从服务器配置 (my.cnf)
[mysqld]
server-id=2
relay-log=mysql-relay-log
read-only=1

# 重启MySQL服务
sudo systemctl restart mysql
```

```sql
-- 在从服务器上配置主服务器信息
CHANGE MASTER TO
    MASTER_HOST='master_server_ip',
    MASTER_USER='repl_user',
    MASTER_PASSWORD='strong_password',
    MASTER_LOG_FILE='mysql-bin.000001',  -- 从主服务器获取
    MASTER_LOG_POS=154;                  -- 从主服务器获取

-- 启动复制
START SLAVE;

-- 检查复制状态
SHOW SLAVE STATUS\G
```

### 基于GTID的复制

```sql
-- 主服务器配置
[mysqld]
server-id=1
log-bin=mysql-bin
gtid-mode=ON
enforce-gtid-consistency=ON

-- 从服务器配置
[mysqld]
server-id=2
gtid-mode=ON
enforce-gtid-consistency=ON

-- 配置从服务器（基于GTID）
CHANGE MASTER TO
    MASTER_HOST='master_server_ip',
    MASTER_USER='repl_user',
    MASTER_PASSWORD='strong_password',
    MASTER_AUTO_POSITION=1;

START SLAVE;
```

## 应用程序逻辑同步

### 事务性同步

```sql
-- 方法1：单事务多表操作
DELIMITER //
CREATE PROCEDURE sp_sync_product_rental(
    IN p_product_id INT,
    IN p_customer_id INT,
    IN p_rental_date DATE
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    
    START TRANSACTION;
    
    -- 更新房源状态
    UPDATE properties 
    SET status = 'rented', rented_date = p_rental_date
    WHERE id = p_product_id AND status = 'available';
    
    -- 检查是否成功更新
    IF ROW_COUNT() = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Property not available for rent';
    END IF;
    
    -- 插入租赁记录
    INSERT INTO customer_rentals (customer_id, property_id, rental_date, status)
    VALUES (p_customer_id, p_product_id, p_rental_date, 'active');
    
    -- 插入日志记录
    INSERT INTO rental_logs (property_id, customer_id, action, created_at)
    VALUES (p_product_id, p_customer_id, 'rent_confirmed', NOW());
    
    COMMIT;
END //
DELIMITER ;

-- 调用存储过程
CALL sp_sync_product_rental(123, 456, '2024-03-05');
```

### 异步队列同步

```sql
-- 创建同步队列表
CREATE TABLE sync_queue (
    id INT AUTO_INCREMENT PRIMARY KEY,
    source_table VARCHAR(50),
    source_id INT,
    target_table VARCHAR(50),
    operation ENUM('insert', 'update', 'delete'),
    data JSON,
    status ENUM('pending', 'processing', 'completed', 'failed') DEFAULT 'pending',
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP NULL
);

-- 插入同步任务
DELIMITER //
CREATE PROCEDURE sp_queue_sync_task(
    IN p_source_table VARCHAR(50),
    IN p_source_id INT,
    IN p_target_table VARCHAR(50),
    IN p_operation VARCHAR(10),
    IN p_data JSON
)
BEGIN
    INSERT INTO sync_queue (source_table, source_id, target_table, operation, data)
    VALUES (p_source_table, p_source_id, p_target_table, p_operation, p_data);
END //
DELIMITER ;

-- 处理同步队列
DELIMITER //
CREATE PROCEDURE sp_process_sync_queue()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_id, v_source_id INT;
    DECLARE v_source_table, v_target_table, v_operation VARCHAR(50);
    DECLARE v_data JSON;
    DECLARE v_sql TEXT;
    
    DECLARE queue_cursor CURSOR FOR
        SELECT id, source_table, source_id, target_table, operation, data
        FROM sync_queue
        WHERE status = 'pending'
        ORDER BY created_at
        LIMIT 100;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN queue_cursor;
    
    queue_loop: LOOP
        FETCH queue_cursor INTO v_id, v_source_table, v_source_id, v_target_table, v_operation, v_data;
        
        IF done THEN
            LEAVE queue_loop;
        END IF;
        
        -- 更新状态为处理中
        UPDATE sync_queue SET status = 'processing' WHERE id = v_id;
        
        -- 根据操作类型构建SQL
        CASE v_operation
            WHEN 'insert' THEN
                SET v_sql = CONCAT('INSERT INTO ', v_target_table, ' SELECT * FROM ', v_source_table, ' WHERE id = ', v_source_id);
            WHEN 'update' THEN
                SET v_sql = CONCAT('UPDATE ', v_target_table, ' t1 JOIN ', v_source_table, ' t2 ON t1.source_id = t2.id SET t1.data = t2.data WHERE t2.id = ', v_source_id);
            WHEN 'delete' THEN
                SET v_sql = CONCAT('DELETE FROM ', v_target_table, ' WHERE source_id = ', v_source_id);
        END CASE;
        
        -- 执行同步操作（这里简化，实际应用中需要动态SQL）
        -- SET @sql = v_sql;
        -- PREPARE stmt FROM @sql;
        -- EXECUTE stmt;
        -- DEALLOCATE PREPARE stmt;
        
        -- 更新状态为完成
        UPDATE sync_queue 
        SET status = 'completed', processed_at = NOW() 
        WHERE id = v_id;
        
    END LOOP;
    
    CLOSE queue_cursor;
END //
DELIMITER ;
```

## 定时任务同步

### 基于时间戳的增量同步

```sql
-- 创建同步状态表
CREATE TABLE sync_status (
    id INT AUTO_INCREMENT PRIMARY KEY,
    source_table VARCHAR(50),
    target_table VARCHAR(50),
    last_sync_time TIMESTAMP,
    sync_type ENUM('full', 'incremental') DEFAULT 'incremental',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 增量同步存储过程
DELIMITER //
CREATE PROCEDURE sp_incremental_sync(
    IN p_source_table VARCHAR(50),
    IN p_target_table VARCHAR(50)
)
BEGIN
    DECLARE v_last_sync_time TIMESTAMP;
    DECLARE v_current_time TIMESTAMP DEFAULT NOW();
    
    -- 获取上次同步时间
    SELECT COALESCE(last_sync_time, '1970-01-01 00:00:00')
    INTO v_last_sync_time
    FROM sync_status
    WHERE source_table = p_source_table AND target_table = p_target_table;
    
    -- 如果没有记录，创建一条
    IF v_last_sync_time IS NULL THEN
        INSERT INTO sync_status (source_table, target_table, last_sync_time)
        VALUES (p_source_table, p_target_table, '1970-01-01 00:00:00');
        SET v_last_sync_time = '1970-01-01 00:00:00';
    END IF;
    
    -- 同步新增和修改的数据
    SET @sql = CONCAT(
        'INSERT INTO ', p_target_table, 
        ' (source_id, name, price, sync_time) ',
        'SELECT id, name, price, NOW() FROM ', p_source_table,
        ' WHERE updated_at > ''', v_last_sync_time, ''' ',
        'ON DUPLICATE KEY UPDATE ',
        'name = VALUES(name), price = VALUES(price), sync_time = VALUES(sync_time)'
    );
    
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    
    -- 更新同步时间
    UPDATE sync_status 
    SET last_sync_time = v_current_time, updated_at = v_current_time
    WHERE source_table = p_source_table AND target_table = p_target_table;
    
    -- 记录同步日志
    INSERT INTO sync_logs (source_table, target_table, sync_time, records_processed)
    VALUES (p_source_table, p_target_table, v_current_time, ROW_COUNT());
END //
DELIMITER ;
```

### 定时任务调度

```sql
-- 创建定时事件
CREATE EVENT ev_hourly_sync
ON SCHEDULE EVERY 1 HOUR
STARTS '2024-03-05 00:00:00'
DO
BEGIN
    CALL sp_incremental_sync('source_products', 'target_products');
    CALL sp_incremental_sync('source_orders', 'target_orders');
END;

-- 查看事件状态
SHOW EVENTS;

-- 启用事件调度器
SET GLOBAL event_scheduler = ON;
```

## 实际业务案例

### 案例：房屋租赁系统同步

```sql
-- 房源表
CREATE TABLE properties (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200),
    address VARCHAR(500),
    price DECIMAL(10,2),
    status ENUM('available', 'rented', 'maintenance') DEFAULT 'available',
    owner_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 租客已租房源表
CREATE TABLE customer_rentals (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    property_id INT,
    property_title VARCHAR(200),
    property_address VARCHAR(500),
    rental_price DECIMAL(10,2),
    rental_date DATE,
    status ENUM('active', 'terminated') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 租房操作的完整流程
DELIMITER //
CREATE PROCEDURE sp_rent_property(
    IN p_customer_id INT,
    IN p_property_id INT,
    IN p_rental_date DATE
)
BEGIN
    DECLARE v_property_title VARCHAR(200);
    DECLARE v_property_address VARCHAR(500);
    DECLARE v_rental_price DECIMAL(10,2);
    DECLARE v_property_status VARCHAR(20);
    
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;
    
    START TRANSACTION;
    
    -- 检查房源状态并获取信息
    SELECT title, address, price, status
    INTO v_property_title, v_property_address, v_rental_price, v_property_status
    FROM properties
    WHERE id = p_property_id FOR UPDATE;
    
    -- 验证房源可租
    IF v_property_status != 'available' THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Property is not available for rent';
    END IF;
    
    -- 更新房源状态
    UPDATE properties 
    SET status = 'rented', updated_at = NOW()
    WHERE id = p_property_id;
    
    -- 添加到租客已租房源表
    INSERT INTO customer_rentals (
        customer_id, property_id, property_title, 
        property_address, rental_price, rental_date
    ) VALUES (
        p_customer_id, p_property_id, v_property_title,
        v_property_address, v_rental_price, p_rental_date
    );
    
    -- 记录操作日志
    INSERT INTO rental_operation_logs (
        customer_id, property_id, operation, operation_time
    ) VALUES (
        p_customer_id, p_property_id, 'rent_confirmed', NOW()
    );
    
    COMMIT;
    
    -- 返回成功信息
    SELECT 'success' as result, LAST_INSERT_ID() as rental_id;
END //
DELIMITER ;
```

### 案例：电商订单同步

```sql
-- 订单同步到多个业务表
DELIMITER //
CREATE PROCEDURE sp_process_order_sync(
    IN p_order_id INT
)
BEGIN
    DECLARE v_customer_id, v_product_id, v_quantity INT;
    DECLARE v_total_amount DECIMAL(10,2);
    DECLARE v_order_status VARCHAR(20);
    
    -- 获取订单信息
    SELECT customer_id, total_amount, status
    INTO v_customer_id, v_total_amount, v_order_status
    FROM orders WHERE id = p_order_id;
    
    -- 同步到客户统计表
    INSERT INTO customer_statistics (customer_id, total_orders, total_amount, last_order_date)
    VALUES (v_customer_id, 1, v_total_amount, NOW())
    ON DUPLICATE KEY UPDATE
        total_orders = total_orders + 1,
        total_amount = total_amount + VALUES(total_amount),
        last_order_date = VALUES(last_order_date);
    
    -- 同步到财务报表
    INSERT INTO daily_sales_report (sale_date, order_count, total_revenue)
    VALUES (CURDATE(), 1, v_total_amount)
    ON DUPLICATE KEY UPDATE
        order_count = order_count + 1,
        total_revenue = total_revenue + VALUES(total_revenue);
    
    -- 同步库存扣减
    INSERT INTO inventory_changes (product_id, change_type, quantity, reference_id, reference_type)
    SELECT 
        oi.product_id, 'sale', -oi.quantity, p_order_id, 'order'
    FROM order_items oi
    WHERE oi.order_id = p_order_id;
END //
DELIMITER ;
```

## 性能优化策略

### 1. 批量处理优化

```sql
-- 批量同步优化
DELIMITER //
CREATE PROCEDURE sp_batch_sync_optimization(
    IN p_batch_size INT DEFAULT 1000
)
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_min_id, v_max_id INT DEFAULT 0;
    
    -- 获取待同步数据范围
    SELECT MIN(id), MAX(id) INTO v_min_id, v_max_id
    FROM source_table
    WHERE sync_status = 'pending';
    
    WHILE v_min_id <= v_max_id DO
        -- 批量处理
        INSERT INTO target_table (source_id, data, sync_time)
        SELECT id, data, NOW()
        FROM source_table
        WHERE id BETWEEN v_min_id AND v_min_id + p_batch_size - 1
          AND sync_status = 'pending'
        LIMIT p_batch_size;
        
        -- 更新同步状态
        UPDATE source_table
        SET sync_status = 'completed'
        WHERE id BETWEEN v_min_id AND v_min_id + p_batch_size - 1
          AND sync_status = 'pending';
        
        SET v_min_id = v_min_id + p_batch_size;
        
        -- 避免长时间锁定
        DO SLEEP(0.01);
    END WHILE;
END //
DELIMITER ;
```

### 2. 索引优化

```sql
-- 为同步相关字段创建索引
ALTER TABLE source_table ADD INDEX idx_sync_status (sync_status, updated_at);
ALTER TABLE source_table ADD INDEX idx_updated_at (updated_at);
ALTER TABLE target_table ADD INDEX idx_source_id (source_id);
ALTER TABLE sync_queue ADD INDEX idx_status_created (status, created_at);

-- 复合索引优化
ALTER TABLE properties ADD INDEX idx_status_updated (status, updated_at);
ALTER TABLE customer_rentals ADD INDEX idx_customer_status (customer_id, status);
```

## 最佳实践

### 1. 数据一致性保证

```sql
-- 使用校验和确保数据一致性
CREATE TABLE sync_checksum (
    table_name VARCHAR(50),
    sync_time TIMESTAMP,
    source_checksum CHAR(32),
    target_checksum CHAR(32),
    status ENUM('match', 'mismatch') DEFAULT 'match'
);

DELIMITER //
CREATE PROCEDURE sp_verify_sync_integrity(
    IN p_table_name VARCHAR(50)
)
BEGIN
    DECLARE v_source_checksum, v_target_checksum CHAR(32);
    
    -- 计算源表校验和
    SET @sql = CONCAT(
        'SELECT MD5(GROUP_CONCAT(MD5(CONCAT_WS(",", id, name, price)) ORDER BY id)) ',
        'FROM ', p_table_name
    );
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    -- 获取结果存储到变量中
    
    -- 计算目标表校验和
    SET @sql = CONCAT(
        'SELECT MD5(GROUP_CONCAT(MD5(CONCAT_WS(",", source_id, name, price)) ORDER BY source_id)) ',
        'FROM target_', p_table_name
    );
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    
    -- 比较并记录结果
    INSERT INTO sync_checksum (table_name, sync_time, source_checksum, target_checksum, status)
    VALUES (p_table_name, NOW(), v_source_checksum, v_target_checksum,
            IF(v_source_checksum = v_target_checksum, 'match', 'mismatch'));
END //
DELIMITER ;
```

### 2. 错误处理和监控

```sql
-- 同步错误监控
CREATE TABLE sync_error_alerts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    error_type VARCHAR(50),
    table_name VARCHAR(50),
    error_message TEXT,
    error_count INT DEFAULT 1,
    first_occurred TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_occurred TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    status ENUM('active', 'resolved') DEFAULT 'active'
);

-- 创建错误告警触发器
DELIMITER //
CREATE TRIGGER tr_sync_error_alert
AFTER INSERT ON sync_error_log
FOR EACH ROW
BEGIN
    INSERT INTO sync_error_alerts (error_type, table_name, error_message)
    VALUES (NEW.error_type, NEW.table_name, NEW.error_message)
    ON DUPLICATE KEY UPDATE
        error_count = error_count + 1,
        last_occurred = NOW();
END //
DELIMITER ;
```

### 3. 配置管理

```sql
-- 同步配置表
CREATE TABLE sync_config (
    id INT AUTO_INCREMENT PRIMARY KEY,
    config_key VARCHAR(100) UNIQUE,
    config_value TEXT,
    description TEXT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 插入配置
INSERT INTO sync_config (config_key, config_value, description) VALUES
    ('batch_size', '1000', '批量处理大小'),
    ('sync_interval', '300', '同步间隔（秒）'),
    ('max_retry_count', '3', '最大重试次数'),
    ('enable_checksum', 'true', '是否启用校验和验证');

-- 获取配置的函数
DELIMITER //
CREATE FUNCTION fn_get_sync_config(p_key VARCHAR(100))
RETURNS TEXT
READS SQL DATA
DETERMINISTIC
BEGIN
    DECLARE v_value TEXT;
    SELECT config_value INTO v_value
    FROM sync_config
    WHERE config_key = p_key;
    RETURN v_value;
END //
DELIMITER ;
```

---
通过本指南，您可以根据不同的业务需求和技术环境，选择最适合的数据同步方案，确保数据的一致性和系统的稳定性。