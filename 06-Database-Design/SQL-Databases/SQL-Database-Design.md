# SQL Database Design

**Tags**: #database #sql #schema-design #relational
**Date**: 2024-01-01

## 📝 Overview

SQL databases (RDBMS) là foundation của nhiều applications, đặc biệt cho systems cần strong consistency, complex relationships, và ACID transactions. Hiểu cách design efficient schema và optimize queries là crucial cho system performance.

## 🎯 RDBMS Fundamentals

### **ACID Properties**
- **Atomicity**: All-or-nothing transactions
- **Consistency**: Data integrity maintained
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed data persists

### **Relational Model**
```sql
-- Tables với relationships
CREATE TABLE users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    total_amount DECIMAL(10,2),
    status ENUM('pending', 'confirmed', 'shipped', 'delivered'),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

## 🏗️ Schema Design Principles

### **1. Normalization**

#### **1NF (First Normal Form)**
```sql
-- ❌ Violates 1NF (repeating groups)
CREATE TABLE bad_users (
    user_id INT,
    name VARCHAR(100),
    phone1 VARCHAR(20),
    phone2 VARCHAR(20),
    phone3 VARCHAR(20)
);

-- ✅ 1NF compliant
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE user_phones (
    phone_id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    phone_number VARCHAR(20),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

#### **2NF (Second Normal Form)**
```sql
-- ❌ Violates 2NF (partial dependency)
CREATE TABLE order_items_bad (
    order_id INT,
    product_id INT,
    quantity INT,
    product_name VARCHAR(100),  -- Depends only on product_id
    product_price DECIMAL(10,2), -- Depends only on product_id
    PRIMARY KEY (order_id, product_id)
);

-- ✅ 2NF compliant
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    price DECIMAL(10,2)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

#### **3NF (Third Normal Form)**
```sql
-- ❌ Violates 3NF (transitive dependency)
CREATE TABLE employees_bad (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100), -- Depends on department_id
    department_budget DECIMAL(12,2) -- Depends on department_id
);

-- ✅ 3NF compliant
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100),
    budget DECIMAL(12,2)
);

CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
```

### **2. Denormalization for Performance**

```sql
-- Denormalized table cho reporting/analytics
CREATE TABLE order_summary (
    order_id INT PRIMARY KEY,
    user_id INT,
    user_email VARCHAR(255), -- Denormalized from users table
    user_name VARCHAR(100),  -- Denormalized from users table
    total_amount DECIMAL(10,2),
    item_count INT,         -- Calculated field
    order_date DATE,
    status VARCHAR(20),
    INDEX(user_id),
    INDEX(order_date),
    INDEX(status)
);
```

## 🔧 Popular SQL Databases

### **MySQL**
```sql
-- MySQL-specific features
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    data JSON, -- MySQL 5.7+ JSON support
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Partitioning
CREATE TABLE sales (
    id INT,
    sale_date DATE,
    amount DECIMAL(10,2)
) 
PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);
```

### **PostgreSQL**
```sql
-- PostgreSQL advanced features
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    tags TEXT[], -- Array type
    metadata JSONB, -- Binary JSON
    price NUMERIC(10,2),
    search_vector tsvector -- Full-text search
);

-- Indexes
CREATE INDEX idx_products_tags ON products USING GIN(tags);
CREATE INDEX idx_products_metadata ON products USING GIN(metadata);
CREATE INDEX idx_products_search ON products USING GIN(search_vector);

-- Advanced queries
SELECT * FROM products 
WHERE tags && ARRAY['electronics', 'mobile']; -- Array overlap

SELECT * FROM products 
WHERE metadata @> '{"brand": "Apple"}'; -- JSON contains
```

## 📊 Indexing Strategies

### **Types of Indexes**

#### **B-Tree Index (Default)**
```sql
-- Primary key (clustered index)
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100)
);

-- Secondary indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_name ON users(name);

-- Composite index
CREATE INDEX idx_users_email_name ON users(email, name);
```

#### **Hash Index**
```sql
-- PostgreSQL hash index for equality lookups
CREATE INDEX idx_users_id_hash ON users USING HASH(id);
```

#### **Partial Index**
```sql
-- Index only active users
CREATE INDEX idx_active_users ON users(email) 
WHERE status = 'active';
```

#### **Functional Index**
```sql
-- Index on function result
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
```

### **Index Design Best Practices**

```sql
-- ✅ Good: Selective columns first
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- ❌ Bad: Non-selective columns first  
CREATE INDEX idx_orders_date_status ON orders(created_at, status);

-- ✅ Good: Cover common query patterns
-- Query: SELECT user_id, total FROM orders WHERE status = 'pending'
CREATE INDEX idx_orders_status_covering ON orders(status) 
INCLUDE (user_id, total);
```

## ⚡ Query Optimization

### **Explain Plans**
```sql
-- MySQL
EXPLAIN SELECT u.name, COUNT(o.order_id) 
FROM users u 
LEFT JOIN orders o ON u.user_id = o.user_id 
WHERE u.created_at > '2024-01-01'
GROUP BY u.user_id;

-- PostgreSQL  
EXPLAIN (ANALYZE, BUFFERS) 
SELECT u.name, COUNT(o.order_id)
FROM users u 
LEFT JOIN orders o ON u.user_id = o.user_id 
WHERE u.created_at > '2024-01-01'
GROUP BY u.user_id;
```

### **Query Optimization Techniques**

#### **Use Appropriate Joins**
```sql
-- ✅ INNER JOIN when you need matching records
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id;

-- ✅ LEFT JOIN when you need all users
SELECT u.name, COALESCE(COUNT(o.order_id), 0) as order_count
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id;

-- ❌ Avoid unnecessary JOINs
-- Bad: JOIN just to filter
SELECT u.name FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE o.status = 'active';

-- Good: Use EXISTS instead
SELECT u.name FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.user_id = u.user_id AND o.status = 'active'
);
```

#### **Efficient WHERE Clauses**
```sql
-- ✅ Use indexes effectively
SELECT * FROM orders WHERE status = 'pending'; -- Uses index

-- ❌ Functions prevent index usage
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-01';

-- ✅ Range queries use indexes
SELECT * FROM orders 
WHERE created_at >= '2024-01-01' 
AND created_at < '2024-01-02';
```

## 🔄 Database Scaling

### **Read Replicas**
```sql
-- Master-slave configuration
-- Master: Handles writes
-- Slaves: Handle reads

-- Application routing
class DatabaseRouter:
    def route_query(self, query):
        if query.is_write():
            return self.master_db
        else:
            return random.choice(self.slave_dbs)
```

### **Vertical Scaling**
```
Scale Up Options:
├── More CPU cores
├── More RAM
├── Faster SSD storage
└── Better network bandwidth

Limits:
├── Hardware limits
├── Single point of failure
└── Expensive at scale
```

### **Connection Pooling**
```python
# Database connection pool
import sqlalchemy.pool as pool

engine = create_engine(
    'mysql://user:pass@localhost/db',
    pool_size=20,           # Number of connections to maintain
    max_overflow=30,        # Additional connections if needed
    pool_pre_ping=True,     # Validate connections
    pool_recycle=3600       # Recreate connections after 1 hour
)
```

## 📈 Performance Monitoring

### **Key Metrics**
```sql
-- MySQL performance metrics
SHOW GLOBAL STATUS LIKE 'Slow_queries';
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Uptime';

-- Query performance
SELECT 
    query,
    total_time,
    mean_time,
    rows_examined,
    rows_sent
FROM mysql.slow_log
WHERE start_time > DATE_SUB(NOW(), INTERVAL 1 DAY);

-- PostgreSQL stats
SELECT 
    schemaname,
    tablename,
    seq_scan,        -- Sequential scans
    seq_tup_read,    -- Rows read by sequential scans
    idx_scan,        -- Index scans
    idx_tup_fetch    -- Rows fetched by index scans
FROM pg_stat_user_tables;
```

### **Query Analysis**
```sql
-- Find slow queries
SELECT 
    query_time,
    lock_time,
    rows_sent,
    rows_examined,
    sql_text
FROM mysql.slow_log
WHERE query_time > 1.0
ORDER BY query_time DESC
LIMIT 10;

-- Index usage analysis
SELECT 
    table_name,
    index_name,
    cardinality,
    non_unique
FROM information_schema.statistics
WHERE table_schema = 'your_database'
ORDER BY cardinality DESC;
```

## 🎯 Trade-offs & Considerations

### **SQL vs NoSQL Decision Matrix**

| Factor | SQL (RDBMS) | NoSQL |
|--------|-------------|-------|
| **ACID** | Strong ACID guarantees | Eventually consistent |
| **Schema** | Fixed schema, migrations | Flexible schema |
| **Scaling** | Vertical, limited horizontal | Easy horizontal scaling |
| **Queries** | Complex SQL queries | Simple queries |
| **Consistency** | Strong consistency | Tunable consistency |
| **Joins** | Efficient joins | Limited/no joins |

### **When to Choose SQL**
- ✅ Strong consistency requirements
- ✅ Complex relationships
- ✅ ACID transactions needed
- ✅ Complex queries and reporting
- ✅ Well-defined schema

### **Common Pitfalls**
- ❌ Over-normalization hurting performance
- ❌ Missing indexes on frequently queried columns
- ❌ N+1 query problems
- ❌ Not using connection pooling
- ❌ Ignoring query execution plans

## 🔗 Liên kết và Tham khảo

### **Related Topics**
- [[06-Database-Design/Sharding/Database-Sharding|Database Sharding]] - Horizontal scaling strategies
- [[06-Database-Design/Replication/Database-Replication|Database Replication]] - Master-slave setup
- [[01-Fundamentals/ACID-Properties/ACID-Properties|ACID Properties]] - Transaction guarantees
- [[06-Database-Design/NoSQL-Databases/NoSQL-Database-Types|NoSQL Databases]] - Alternative approaches

### **SQL Best Practices**
- Use appropriate data types và constraints
- Implement proper indexing strategy
- Design for query performance
- Plan for data growth và scaling
- Maintain referential integrity

### **Popular SQL Databases**
- **MySQL**: Web applications, e-commerce
- **PostgreSQL**: Complex queries, data analytics
- **SQL Server**: Enterprise applications
- **Oracle**: Large-scale enterprise systems
- **SQLite**: Embedded applications, prototyping

---

*SQL databases excel at structured data và complex relationships. Choose the right database engine based on workload characteristics và scaling requirements.*

## 📚 Further Reading

- "High Performance MySQL" by Baron Schwartz
- "PostgreSQL: Up and Running" by Regina Obe
- Database normalization theory
- SQL query optimization techniques

---

**Key Takeaway**: SQL databases excel at complex relationships và strong consistency, nhưng require careful schema design và query optimization cho performance at scale. 