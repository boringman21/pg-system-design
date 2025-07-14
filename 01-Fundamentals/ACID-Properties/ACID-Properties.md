# ACID Properties

**Tags**: #fundamental #database #transactions #acid
**Date**: 2024-01-01
**Related**: [[Database Transactions]], [[Consistency Models]], [[SQL Databases]]

## 🎯 Tổng quan

ACID là bộ 4 tính chất quan trọng đảm bảo độ tin cậy của database transactions trong SQL databases.

## 🔍 4 Tính chất ACID

### **A - Atomicity (Tính nguyên tử)**

**Định nghĩa**: Một transaction là một đơn vị công việc không thể chia nhỏ - hoặc tất cả thành công, hoặc tất cả thất bại.

```sql
-- Ví dụ: Chuyển tiền giữa 2 tài khoản
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- Nếu bất kỳ statement nào fail → rollback toàn bộ
```

**Cách thực hiện:**
- Write-Ahead Logging (WAL)
- Transaction log
- Rollback mechanisms
- Savepoints

**Vấn đề khi thiếu Atomicity:**
```
❌ Tiền đã trừ ở tài khoản A nhưng chưa cộng vào tài khoản B
❌ Partial updates → data inconsistency
❌ System crash giữa chừng → corrupt state
```

### **C - Consistency (Tính nhất quán)**

**Định nghĩa**: Database phải luôn ở trạng thái valid, tuân thủ tất cả constraints và business rules.

```sql
-- Business rule: Tổng số tiền trong hệ thống không đổi
-- Constraint: Balance không được âm
ALTER TABLE accounts ADD CONSTRAINT chk_balance 
CHECK (balance >= 0);

-- Foreign key constraints
ALTER TABLE orders ADD CONSTRAINT fk_customer
FOREIGN KEY (customer_id) REFERENCES customers(id);
```

**Loại constraints:**
- **Entity integrity**: Primary key not null
- **Referential integrity**: Foreign key constraints  
- **Domain integrity**: Data type, check constraints
- **Business rules**: Custom validation logic

**Ví dụ vi phạm Consistency:**
```
❌ Insert order với customer_id không tồn tại
❌ Update balance thành số âm
❌ Delete customer mà vẫn có orders
```

### **I - Isolation (Tính cô lập)**

**Định nghĩa**: Các transactions concurrent phải thực hiện như thể chúng chạy tuần tự, không ảnh hưởng lẫn nhau.

**Isolation Levels:**

#### 1. Read Uncommitted (Level 0)
```sql
-- T1: Dirty Read có thể xảy ra
BEGIN TRANSACTION;
UPDATE accounts SET balance = 500 WHERE id = 1;
-- T2 có thể đọc được giá trị 500 chưa commit
-- Nếu T1 rollback → T2 đọc dirty data
```

#### 2. Read Committed (Level 1)
```sql
-- T1: Chỉ đọc được committed data
-- Nhưng vẫn có Non-repeatable Read
SELECT balance FROM accounts WHERE id = 1; -- 1000
-- T2 update và commit
SELECT balance FROM accounts WHERE id = 1; -- 1500 (khác lần đọc trước)
```

#### 3. Repeatable Read (Level 2)
```sql
-- T1: Đảm bảo đọc lại cùng giá trị
-- Nhưng vẫn có Phantom Read
SELECT COUNT(*) FROM accounts WHERE balance > 1000; -- 5 records
-- T2 insert thêm account với balance > 1000
SELECT COUNT(*) FROM accounts WHERE balance > 1000; -- 6 records
```

#### 4. Serializable (Level 3)
```sql
-- Hoàn toàn isolated, như chạy tuần tự
-- Không có dirty, non-repeatable, phantom reads
-- Performance thấp nhất
```

**Concurrency Problems:**

| Problem | Read Uncommitted | Read Committed | Repeatable Read | Serializable |
|---------|------------------|----------------|-----------------|--------------|
| **Dirty Read** | ❌ | ✅ | ✅ | ✅ |
| **Non-repeatable Read** | ❌ | ❌ | ✅ | ✅ |
| **Phantom Read** | ❌ | ❌ | ❌ | ✅ |

### **D - Durability (Tính bền vững)**

**Định nghĩa**: Một khi transaction được commit, dữ liệu phải được lưu trữ bền vững, không mất mát ngay cả khi system crash.

**Cách thực hiện:**

#### 1. Write-Ahead Logging (WAL)
```
1. Write changes to log file first
2. Log file must be flushed to disk before commit
3. Data pages can be written to disk later
4. Recovery: replay log after crash
```

#### 2. Force/No-Force Policy
```
Force: Data pages written to disk at commit
No-Force: Data pages written later, rely on log for recovery
```

#### 3. Checkpointing
```sql
-- Periodic checkpoint: flush dirty pages to disk
-- Reduce recovery time after crash
CHECKPOINT;
```

#### 4. Backup & Recovery
```sql
-- Full backup
BACKUP DATABASE mydb TO 'backup.bak';

-- Transaction log backup
BACKUP LOG mydb TO 'log_backup.trn';

-- Point-in-time recovery
RESTORE DATABASE mydb FROM 'backup.bak' 
WITH STOPAT = '2024-01-01 14:30:00';
```

## ⚖️ Trade-offs

### **ACID vs Performance**
```
High ACID compliance → Lower performance
- More locks and logging overhead
- Synchronous writes to disk
- Complex concurrency control

Relaxed ACID → Higher performance  
- Eventual consistency
- Async replication
- Less locking
```

### **ACID vs Scalability**
```
Strong ACID → Harder to distribute
- Global transactions expensive
- 2PC (Two-Phase Commit) overhead
- CAP theorem conflicts

Relaxed ACID → Easier to scale
- BASE properties (Basic Availability, Soft state, Eventual consistency)
- Partition tolerance
- Local transactions
```

## 🛠️ Thực hiện trong các Database

### **MySQL**
```sql
-- InnoDB engine supports full ACID
-- MyISAM doesn't support transactions

SET autocommit = 0;
START TRANSACTION;
-- SQL statements
COMMIT; -- or ROLLBACK;
```

### **PostgreSQL**
```sql
-- Strong ACID compliance
-- MVCC (Multi-Version Concurrency Control)

BEGIN;
-- SQL statements  
COMMIT;
```

### **NoSQL & ACID**
```
MongoDB: ACID transactions từ version 4.0
Cassandra: Eventual consistency, không ACID
Redis: ACID cho single operations, MULTI/EXEC cho transactions
DynamoDB: ACID cho single items, transactions limited
```

## 🎯 Best Practices

### **1. Transaction Design**
```sql
-- Keep transactions short
BEGIN TRANSACTION;
    -- Minimal necessary operations
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Avoid long-running transactions
❌ BEGIN; SELECT * FROM huge_table; COMMIT;
```

### **2. Error Handling**
```sql
BEGIN TRY
    BEGIN TRANSACTION;
        -- Business logic
        UPDATE table1 SET col1 = value1;
        UPDATE table2 SET col2 = value2;
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    ROLLBACK TRANSACTION;
    THROW;
END CATCH
```

### **3. Deadlock Prevention**
```sql
-- Access resources in consistent order
-- Transaction 1
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Transaction 2 (same order)
UPDATE accounts SET balance = balance - 50 WHERE id = 1;
UPDATE accounts SET balance = balance + 50 WHERE id = 2;
```

## 🔍 Interview Questions

### **Q1: Explain ACID properties with banking example**
```
A: Atomicity - Transfer completes fully or not at all
C: Consistency - Total money remains same, no negative balances
I: Isolation - Concurrent transfers don't interfere
D: Durability - Completed transfer survives system crash
```

### **Q2: What happens if we remove each ACID property?**
```
Without A: Partial updates, inconsistent state
Without C: Invalid data, constraint violations
Without I: Race conditions, unexpected results
Without D: Data loss after crashes
```

### **Q3: ACID vs BASE comparison**
```
ACID: Strong consistency, lower availability
BASE: High availability, eventual consistency
Use ACID for: Financial systems, critical data
Use BASE for: Social media, content delivery, high-scale systems
```

## 📚 Related Topics

- [[Database Transactions]]
- [[Isolation Levels]]
- [[Concurrency Control]]
- [[Two-Phase Commit]]
- [[CAP Theorem]]
- [[BASE Properties]]
- [[MVCC]]

---

*ACID properties are fundamental to relational databases but come with performance and scalability trade-offs. Understanding when to relax ACID guarantees is key to designing scalable systems.* 