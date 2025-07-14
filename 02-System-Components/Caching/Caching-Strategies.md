# Caching Strategies

**Tags**: #component #caching #performance
**Date**: 2024-01-01

## 📝 Overview

Caching là technique quan trọng nhất để improve performance trong system design. Bằng cách store frequently accessed data ở location gần user hơn, caching có thể giảm latency từ seconds xuống milliseconds.

## 🎯 Purpose & Benefits

### Why Caching?
- **Performance**: Giảm response time từ 100ms xuống 1ms
- **Scalability**: Giảm load lên database/backend
- **Cost**: Cheaper than scaling database
- **User Experience**: Faster page loads, better UX

### Trade-offs
- **Complexity**: Cache invalidation, consistency issues
- **Memory Cost**: Additional infrastructure
- **Stale Data**: Cache có thể out-of-date

## 🏗️ Cache Levels

### **Browser Cache**
```
Client-side caching:
- Static assets (CSS, JS, images)
- API responses
- Controlled by HTTP headers
```

### **CDN (Content Delivery Network)**
```
Geographic caching:
- Static content closer to users
- Edge servers worldwide
- Examples: CloudFlare, AWS CloudFront
```

### **Reverse Proxy Cache**
```
Server-side caching:
- NGINX, Varnish
- Cache dynamic content
- Between load balancer and app servers
```

### **Application Cache**
```
In-memory caching:
- Redis, Memcached
- Database query results
- Session data, user profiles
```

### **Database Cache**
```
Database-level caching:
- Query result cache
- Buffer pool (InnoDB)
- Internal optimizations
```

## ⚙️ Caching Patterns

### **1. Cache-Aside (Lazy Loading)**
```python
def get_user(user_id):
    # Try cache first
    user = cache.get(f"user:{user_id}")
    if user:
        return user
    
    # Cache miss - fetch from database
    user = database.get_user(user_id)
    if user:
        cache.set(f"user:{user_id}", user, ttl=3600)
    
    return user

def update_user(user_id, data):
    # Update database
    database.update_user(user_id, data)
    # Invalidate cache
    cache.delete(f"user:{user_id}")
```

**Pros**: Simple, cache only what's needed
**Cons**: Cache miss penalty, possible stale data

### **2. Write-Through**
```python
def update_user(user_id, data):
    # Update database first
    database.update_user(user_id, data)
    # Update cache
    cache.set(f"user:{user_id}", data, ttl=3600)
    
def get_user(user_id):
    # Cache is always updated
    return cache.get(f"user:{user_id}")
```

**Pros**: Cache always fresh, no cache misses
**Cons**: Write penalty, cache churn

### **3. Write-Behind (Write-Back)**
```python
def update_user(user_id, data):
    # Update cache immediately
    cache.set(f"user:{user_id}", data, ttl=3600)
    # Queue database update (async)
    queue_db_update.delay('update_user', user_id, data)
```

**Pros**: Fast writes, high throughput
**Cons**: Risk of data loss, complex implementation

### **4. Refresh-Ahead**
```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    
    # If cache expires soon, refresh in background
    if cache.ttl(f"user:{user_id}") < 300:  # < 5 minutes
        refresh_cache.delay(user_id)
    
    return user

def refresh_cache(user_id):
    user = database.get_user(user_id)
    cache.set(f"user:{user_id}", user, ttl=3600)
```

**Pros**: Always fresh data, no cache miss penalty
**Cons**: Complex logic, prediction needed

## 🔧 Cache Technologies

### **Redis**
```python
# In-memory data structure store
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

# Basic operations
r.set('key', 'value', ex=3600)  # TTL = 1 hour
value = r.get('key')

# Advanced data structures
r.lpush('queue', 'item1', 'item2')  # List
r.hset('user:123', 'name', 'John')  # Hash
r.sadd('tags', 'python', 'redis')   # Set
```

**Use Cases**: Session store, pub/sub, leaderboards
**Pros**: Rich data types, persistence options
**Cons**: Single-threaded, memory bound

### **Memcached**
```python
# Simple key-value cache
import memcache

mc = memcache.Client(['127.0.0.1:11211'], debug=0)

mc.set('key', 'value', time=3600)
value = mc.get('key')
```

**Use Cases**: Simple caching, web applications
**Pros**: Simple, fast, multi-threaded
**Cons**: No persistence, limited data types

### **CDN Examples**
```javascript
// CloudFlare cache control
Cache-Control: public, max-age=3600, s-maxage=86400

// AWS CloudFront
{
  "CachePolicyId": "cache-policy-id",
  "DefaultTTL": 86400,
  "MaxTTL": 31536000
}
```

## 📊 Cache Performance Metrics

### **Key Metrics**
```python
# Cache hit ratio
hit_ratio = cache_hits / (cache_hits + cache_misses)
# Target: 90%+ for read-heavy workloads

# Average response time
avg_response_time = (cache_hits * cache_latency + 
                    cache_misses * db_latency) / total_requests

# Memory utilization
memory_usage = used_memory / total_memory
# Target: 70-80% (avoid memory pressure)

# Eviction rate
eviction_rate = evicted_keys / total_keys
# High eviction rate indicates need for more memory
```

### **Monitoring Dashboard**
```
Cache Performance Dashboard:
├── Hit Rate (line chart)
├── Response Time (histogram) 
├── Memory Usage (gauge)
├── Eviction Rate (counter)
└── Error Rate (alerts)
```

## 🚨 Cache Problems & Solutions

### **Cache Stampede**
```python
# Problem: Multiple requests for expired key
def get_with_lock(key):
    value = cache.get(key)
    if value:
        return value
    
    # Use distributed lock
    with cache.lock(f"lock:{key}", timeout=10):
        # Double-check after acquiring lock
        value = cache.get(key)
        if value:
            return value
        
        # Only one request fetches from DB
        value = database.get(key)
        cache.set(key, value, ttl=3600)
        return value
```

### **Cache Invalidation**
```python
# Tag-based invalidation
def cache_user(user_id, user_data):
    cache.set(f"user:{user_id}", user_data, ttl=3600)
    cache.sadd(f"tags:user:{user_id}", f"user:{user_id}")
    cache.sadd(f"tags:department:{user_data.dept}", f"user:{user_id}")

def invalidate_department(dept_id):
    # Invalidate all users in department
    keys = cache.smembers(f"tags:department:{dept_id}")
    for key in keys:
        cache.delete(key)
    cache.delete(f"tags:department:{dept_id}")
```

### **Hotkey Problem**
```python
# Distribute hotkeys across multiple cache instances
def get_hotkey(key):
    # Add random suffix to distribute load
    suffix = random.randint(1, 3)
    cache_key = f"{key}:copy{suffix}"
    
    value = cache.get(cache_key)
    if value:
        return value
    
    # Fetch and replicate to all copies
    value = database.get(key)
    for i in range(1, 4):
        cache.set(f"{key}:copy{i}", value, ttl=3600)
    
    return value
```

## 🎯 Cache Design Decisions

### **What to Cache?**
```
High Value Items:
✅ Frequently accessed data
✅ Expensive computations
✅ Database query results
✅ User sessions
✅ Static content

Low Value Items:
❌ Write-heavy data
❌ Large objects (>1MB)
❌ Frequently changing data
❌ One-time access data
```

### **Cache Hierarchy**
```
L1: Browser Cache (seconds)
L2: CDN (minutes to hours)  
L3: Load Balancer Cache (minutes)
L4: Application Cache (hours)
L5: Database Buffer (persistent)
```

## 🔗 Related Topics

- [[Load Balancer]] - Proxy caching
- [[CDN]] - Geographic caching
- [[Database Design]] - Query optimization
- [[Performance Optimization]] - Caching strategies

## 📚 Further Reading

- Redis documentation and best practices
- Memcached performance tuning
- CDN comparison and selection guide
- "High Performance Browser Networking" - Caching chapter

---

**Key Takeaway**: Effective caching can improve performance by 10-100x, nhưng cần careful design để tránh complexity và data consistency issues. 