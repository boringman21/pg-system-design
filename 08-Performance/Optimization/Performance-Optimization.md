# Performance Optimization

**Tags**: #performance #optimization #system-design
**Date**: 2024-01-01

## 🎯 Tổng quan

Performance optimization là quá trình cải thiện hiệu suất hệ thống để đạt được:
- Thời gian phản hồi thấp hơn (Lower latency)
- Throughput cao hơn
- Sử dụng tài nguyên hiệu quả
- Trải nghiệm người dùng tốt hơn

## 📊 Performance Metrics

### **Core Metrics**
```
Latency: Thời gian xử lý một request
Throughput: Số lượng requests xử lý được/giây
Response Time: Thời gian từ khi gửi request đến khi nhận response
Availability: % thời gian hệ thống hoạt động
Error Rate: Tỷ lệ requests bị lỗi
```

### **Resource Metrics**
```
CPU Utilization: % CPU được sử dụng
Memory Usage: RAM đang sử dụng
Disk I/O: Tốc độ đọc/ghi disk
Network I/O: Băng thông mạng sử dụng
Connection Pool: Số connections active/idle
```

## 🔧 Frontend Optimization

### **Web Performance**
```
Critical Rendering Path:
1. HTML parsing
2. CSS parsing 
3. JavaScript execution
4. Layout calculation
5. Paint & Composite

Optimization Techniques:
✅ Minimize HTTP requests
✅ Compress assets (Gzip, Brotli)
✅ Use CDN
✅ Optimize images (WebP, AVIF)
✅ Lazy loading
✅ Code splitting
✅ Tree shaking
```

### **Browser Optimization**
```html
<!-- Resource Hints -->
<link rel="preload" href="critical.css" as="style">
<link rel="prefetch" href="next-page.js">
<link rel="preconnect" href="https://api.example.com">

<!-- Image Optimization -->
<picture>
  <source srcset="image.avif" type="image/avif">
  <source srcset="image.webp" type="image/webp">
  <img src="image.jpg" alt="Description" loading="lazy">
</picture>
```

## ⚡ Backend Optimization

### **Application Level**
```
Algorithm Optimization:
✅ Choose right data structures
✅ Optimize time complexity O(n) → O(log n)
✅ Space-time tradeoffs
✅ Memoization/Dynamic Programming

Code Optimization:
✅ Avoid N+1 queries
✅ Use connection pooling
✅ Implement circuit breakers
✅ Async/non-blocking I/O
✅ Batch operations
```

### **Database Optimization**
```sql
-- Index Optimization
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_order_date ON orders(created_at DESC);

-- Query Optimization
-- Bad: N+1 Query
SELECT * FROM orders WHERE user_id = ?;
SELECT * FROM users WHERE id = ?; -- Repeated for each order

-- Good: Join Query
SELECT o.*, u.name 
FROM orders o 
JOIN users u ON o.user_id = u.id 
WHERE o.created_at > '2024-01-01';

-- Use EXPLAIN to analyze query plans
EXPLAIN SELECT * FROM orders WHERE user_id = 123;
```

### **Caching Strategies**
```
Multi-Level Caching:

L1: Application Cache (In-Memory)
- Response caching
- Object caching
- Method result caching

L2: Distributed Cache (Redis/Memcached)
- Session data
- Frequently accessed data
- Computed results

L3: Database Query Cache
- Query result caching
- Prepared statement caching

L4: CDN (Content Delivery Network)
- Static assets
- Dynamic content at edge
```

## 🗄️ Database Performance

### **Indexing Strategies**
```sql
-- B-Tree Index (Default)
CREATE INDEX idx_btree ON table(column);

-- Hash Index (Equality lookups)
CREATE INDEX idx_hash ON table USING HASH(column);

-- Partial Index (Filtered)
CREATE INDEX idx_partial ON orders(user_id) 
WHERE status = 'active';

-- Composite Index (Multiple columns)
CREATE INDEX idx_composite ON users(age, city, status);

-- Covering Index (Include all needed columns)
CREATE INDEX idx_covering ON orders(user_id) 
INCLUDE (product_id, quantity, price);
```

### **Query Optimization**
```sql
-- Use EXISTS instead of IN for subqueries
-- Bad
SELECT * FROM users WHERE id IN (
    SELECT user_id FROM orders WHERE total > 1000
);

-- Good
SELECT * FROM users u WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.user_id = u.id AND o.total > 1000
);

-- Use LIMIT for pagination
SELECT * FROM products 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 100;

-- Use prepared statements
PREPARE stmt FROM 'SELECT * FROM users WHERE age > ? AND city = ?';
```

### **Database Partitioning**
```sql
-- Horizontal Partitioning (Sharding)
-- By Range
CREATE TABLE orders_2024_q1 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

-- By Hash
CREATE TABLE users_shard_1 PARTITION OF users
FOR VALUES WITH (modulus 4, remainder 0);

-- Vertical Partitioning
-- Split large tables by columns
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    bio TEXT,
    avatar_url VARCHAR(255)
);
```

## 🔄 Concurrency & Parallelism

### **Threading Models**
```python
# Thread Pool
import concurrent.futures

def process_data(data):
    # CPU-intensive work
    return expensive_computation(data)

with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(process_data, item) for item in data_list]
    results = [future.result() for future in futures]

# Async/Await (I/O intensive)
import asyncio
import aiohttp

async def fetch_data(session, url):
    async with session.get(url) as response:
        return await response.json()

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_data(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
```

### **Lock-Free Programming**
```python
# Using atomic operations
import threading
from queue import Queue

# Lock-free queue
queue = Queue()

# Compare-and-swap pattern
class Counter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()
    
    def increment(self):
        with self._lock:
            self._value += 1
    
    # Better: Use atomic operations
    def atomic_increment(self):
        import threading
        return threading.current_thread().ident
```

## 📡 Network Optimization

### **Protocol Optimization**
```
HTTP/2 Benefits:
✅ Multiplexing (Multiple requests per connection)
✅ Server Push
✅ Header compression (HPACK)
✅ Binary protocol

HTTP/3 (QUIC) Benefits:
✅ Reduced connection establishment time
✅ Better mobile performance
✅ Head-of-line blocking elimination
```

### **Connection Management**
```python
# Connection Pooling
import requests
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

session = requests.Session()
adapter = HTTPAdapter(
    pool_connections=100,  # Number of connection pools
    pool_maxsize=100,      # Max connections per pool
    max_retries=Retry(total=3, backoff_factor=0.1)
)
session.mount('http://', adapter)
session.mount('https://', adapter)

# Keep-alive connections
response = session.get('https://api.example.com/data')
```

### **Data Compression**
```python
# Gzip compression
import gzip
import json

# Compress data
data = {"key": "value" * 1000}
json_data = json.dumps(data).encode('utf-8')
compressed = gzip.compress(json_data)

print(f"Original: {len(json_data)} bytes")
print(f"Compressed: {len(compressed)} bytes")
print(f"Compression ratio: {len(compressed)/len(json_data):.2%}")
```

## 🎛️ System-Level Optimization

### **Memory Management**
```
Memory Optimization:
✅ Object pooling
✅ Memory-mapped files
✅ Garbage collection tuning
✅ Memory-efficient data structures

JVM Tuning:
-Xms2g -Xmx8g              # Heap size
-XX:+UseG1GC               # Use G1 garbage collector
-XX:MaxGCPauseMillis=100   # Max GC pause time
-XX:+PrintGCDetails        # GC logging
```

### **CPU Optimization**
```bash
# CPU affinity
taskset -c 0,1 ./my-application

# Process priority
nice -n -10 ./cpu-intensive-task

# Compiler optimizations
gcc -O3 -march=native -flto program.c
```

### **I/O Optimization**
```python
# Async I/O
import asyncio
import aiofiles

async def read_file_async(filename):
    async with aiofiles.open(filename, 'r') as f:
        return await f.read()

# Memory-mapped files for large files
import mmap

with open('large_file.txt', 'r') as f:
    with mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ) as mmapped_file:
        # Process file without loading into memory
        data = mmapped_file[1000:2000]
```

## 📈 Scaling Optimization

### **Horizontal Scaling**
```
Load Balancing Algorithms:
1. Round Robin - Simple distribution
2. Weighted Round Robin - Based on server capacity
3. Least Connections - Route to least busy server
4. IP Hash - Consistent routing per client
5. Health Check - Avoid unhealthy servers

Auto-scaling Triggers:
✅ CPU utilization > 70%
✅ Memory utilization > 80%
✅ Request queue length > 100
✅ Response time > 500ms
```

### **Vertical Scaling**
```
Scale-up Strategies:
✅ Increase CPU cores
✅ Add more RAM
✅ Faster storage (SSD → NVMe)
✅ Better network interfaces

Limitations:
❌ Hardware limits
❌ Single point of failure
❌ More expensive than horizontal scaling
```

## 🔍 Performance Testing

### **Load Testing**
```python
# Using locust for load testing
from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    wait_time = between(1, 3)
    
    @task(3)
    def view_homepage(self):
        self.client.get("/")
    
    @task(1)
    def view_profile(self):
        self.client.get("/profile")
    
    def on_start(self):
        self.client.post("/login", json={
            "username": "test",
            "password": "test"
        })

# Run: locust -f loadtest.py --host=http://localhost:8000
```

### **Benchmarking**
```bash
# Apache Bench
ab -n 10000 -c 100 http://localhost:8000/

# wrk (modern alternative)
wrk -t12 -c400 -d30s http://localhost:8000/

# Siege
siege -c 100 -t 60s http://localhost:8000/
```

## 🚨 Common Performance Antipatterns

### **Database Antipatterns**
```sql
-- N+1 Query Problem
-- Bad: Executes N+1 queries
users = get_all_users()  -- 1 query
for user in users:
    orders = get_orders_by_user(user.id)  -- N queries

-- Good: Single query with join
SELECT u.*, o.* FROM users u 
LEFT JOIN orders o ON u.id = o.user_id;

-- SELECT * Problem
-- Bad: Fetches unnecessary data
SELECT * FROM users WHERE id = 123;

-- Good: Select only needed columns
SELECT id, name, email FROM users WHERE id = 123;
```

### **Application Antipatterns**
```python
# Synchronous I/O in loops
# Bad
results = []
for url in urls:
    response = requests.get(url)  # Blocking call
    results.append(response.json())

# Good: Async batch processing
async def fetch_all(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_data(session, url) for url in urls]
        return await asyncio.gather(*tasks)

# Memory leaks
# Bad: Not closing resources
def read_files():
    files = []
    for filename in filenames:
        f = open(filename)  # Never closed
        files.append(f.read())
    return files

# Good: Using context managers
def read_files():
    contents = []
    for filename in filenames:
        with open(filename) as f:
            contents.append(f.read())
    return contents
```

## 📊 Performance Monitoring

### **Key Metrics to Track**
```
Application Metrics:
- Request rate (requests/second)
- Response time (p50, p95, p99)
- Error rate (%)
- Throughput (requests/minute)

Infrastructure Metrics:
- CPU utilization (%)
- Memory usage (GB)
- Disk I/O (IOPS, bandwidth)
- Network I/O (packets/sec, bandwidth)

Business Metrics:
- User satisfaction (Apdex score)
- Conversion rates
- Page load times
- Mobile vs Desktop performance
```

### **Monitoring Tools**
```yaml
# Prometheus configuration
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'my-app'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: /metrics
    scrape_interval: 5s
```

## 💡 Best Practices

### **Development Practices**
```
Performance Culture:
✅ Measure first, optimize second
✅ Set performance budgets
✅ Use performance CI/CD checks
✅ Regular performance reviews
✅ Document performance decisions

Code Review Checklist:
✅ Are database queries optimized?
✅ Is caching implemented appropriately?
✅ Are resources properly released?
✅ Is error handling efficient?
✅ Are there any performance bottlenecks?
```

### **Optimization Priorities**
```
1. Profile and identify bottlenecks
2. Optimize the biggest impact areas first
3. Measure improvements
4. Document changes
5. Monitor in production

Remember: 
- Premature optimization is the root of all evil
- Measure twice, optimize once
- User experience > Perfect code
```

---

**Checklist cho Performance Optimization:**
- [ ] Identify performance bottlenecks
- [ ] Set clear performance goals
- [ ] Implement monitoring and alerting
- [ ] Optimize database queries and indexes
- [ ] Implement appropriate caching
- [ ] Optimize application code
- [ ] Configure infrastructure properly
- [ ] Load test before production
- [ ] Monitor and iterate

**Nhớ rằng**: Performance optimization là một quá trình liên tục, không phải một lần làm xong! 🚀 