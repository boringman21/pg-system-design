# Rate Limiter Design - Practice Problem

**Tags**: #practice #rate-limiting #algorithm #system-design
**Date**: 2024-01-01
**Difficulty**: Medium

## 📝 Problem Statement

Design a rate limiter that can:
- Limit the number of requests per user per time window
- Support different rate limiting strategies
- Scale to handle millions of users
- Be used as both library and service

## 🎯 Requirements

### Functional Requirements
- [ ] Limit requests per user (e.g., 100 requests/hour)
- [ ] Different limits for different APIs
- [ ] Support multiple time windows (minute, hour, day)
- [ ] Return meaningful error messages when limit exceeded

### Non-functional Requirements
- [ ] Low latency (< 10ms overhead)
- [ ] High availability (99.9%)
- [ ] Distributed and scalable
- [ ] Memory efficient

## 🔧 Implementation Approaches

### **1. Token Bucket Algorithm**
```python
import time
from threading import Lock

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity      # Max tokens
        self.tokens = capacity        # Current tokens
        self.refill_rate = refill_rate # Tokens per second
        self.last_refill = time.time()
        self.lock = Lock()
    
    def consume(self, tokens=1):
        with self.lock:
            self._refill()
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False
    
    def _refill(self):
        now = time.time()
        tokens_to_add = (now - self.last_refill) * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now

# Usage
bucket = TokenBucket(capacity=100, refill_rate=10)  # 100 tokens, refill 10/sec
if bucket.consume():
    print("Request allowed")
else:
    print("Rate limited")
```

### **2. Sliding Window Log**
```python
import time
from collections import deque

class SlidingWindowLog:
    def __init__(self, limit, window_size):
        self.limit = limit           # Max requests
        self.window_size = window_size # Time window in seconds
        self.requests = deque()      # Request timestamps
    
    def is_allowed(self):
        now = time.time()
        
        # Remove expired requests
        while self.requests and self.requests[0] <= now - self.window_size:
            self.requests.popleft()
        
        # Check if under limit
        if len(self.requests) < self.limit:
            self.requests.append(now)
            return True
        
        return False

# Usage  
limiter = SlidingWindowLog(limit=100, window_size=3600)  # 100 req/hour
if limiter.is_allowed():
    print("Request allowed")
```

### **3. Sliding Window Counter**
```python
import time
import math

class SlidingWindowCounter:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.current_window = 0
        self.current_count = 0
        self.previous_count = 0
    
    def is_allowed(self):
        now = time.time()
        current_window = math.floor(now / self.window_size)
        
        # New window
        if current_window > self.current_window:
            self.previous_count = self.current_count
            self.current_count = 0
            self.current_window = current_window
        
        # Calculate weighted count
        elapsed_time = now - (current_window * self.window_size)
        weight = 1 - (elapsed_time / self.window_size)
        estimated_count = (self.previous_count * weight) + self.current_count
        
        if estimated_count < self.limit:
            self.current_count += 1
            return True
        
        return False
```

## 🚀 Distributed Rate Limiter

### **Redis-based Implementation**
```python
import redis
import time
import json

class DistributedRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        
    def is_allowed(self, user_id, limit, window_size):
        key = f"rate_limit:{user_id}"
        pipeline = self.redis.pipeline()
        now = time.time()
        
        # Sliding window log approach
        pipeline.zremrangebyscore(key, 0, now - window_size)
        pipeline.zcard(key)
        pipeline.zadd(key, {str(now): now})
        pipeline.expire(key, int(window_size) + 1)
        
        results = pipeline.execute()
        current_count = results[1]
        
        return current_count < limit

# Usage
redis_client = redis.Redis(host='localhost', port=6379, db=0)
limiter = DistributedRateLimiter(redis_client)

if limiter.is_allowed('user123', limit=100, window_size=3600):
    print("Request allowed")
```

### **With Lua Script for Atomicity**
```python
class RedisRateLimiterAtomic:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.lua_script = """
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        
        -- Remove expired entries
        redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
        
        -- Count current entries
        local current = redis.call('ZCARD', key)
        
        if current < limit then
            -- Add current request
            redis.call('ZADD', key, now, now)
            redis.call('EXPIRE', key, window + 1)
            return 1
        else
            return 0
        end
        """
    
    def is_allowed(self, user_id, limit, window_size):
        key = f"rate_limit:{user_id}"
        now = time.time()
        
        result = self.redis.eval(
            self.lua_script, 
            1, 
            key, 
            limit, 
            window_size, 
            now
        )
        
        return result == 1
```

## 🎛️ Rate Limiter Service

### **HTTP API**
```python
from flask import Flask, request, jsonify
import redis

app = Flask(__name__)
redis_client = redis.Redis()
limiter = DistributedRateLimiter(redis_client)

@app.route('/rate-limit', methods=['POST'])
def check_rate_limit():
    data = request.json
    user_id = data['user_id']
    limit = data['limit']
    window_size = data['window_size']
    
    allowed = limiter.is_allowed(user_id, limit, window_size)
    
    return jsonify({
        'allowed': allowed,
        'user_id': user_id,
        'limit': limit,
        'window_size': window_size
    })

# Rate limiter middleware
def rate_limit_middleware(user_id, limit=100, window=3600):
    if not limiter.is_allowed(user_id, limit, window):
        return jsonify({'error': 'Rate limit exceeded'}), 429
    return None
```

## 📊 Performance & Monitoring

### **Metrics Collection**
```python
import time
from collections import defaultdict

class RateLimiterMetrics:
    def __init__(self):
        self.allowed_requests = 0
        self.blocked_requests = 0
        self.latency_samples = []
        self.per_user_stats = defaultdict(lambda: {'allowed': 0, 'blocked': 0})
    
    def record_request(self, user_id, allowed, latency):
        if allowed:
            self.allowed_requests += 1
            self.per_user_stats[user_id]['allowed'] += 1
        else:
            self.blocked_requests += 1
            self.per_user_stats[user_id]['blocked'] += 1
        
        self.latency_samples.append(latency)
        
        # Keep only last 1000 samples
        if len(self.latency_samples) > 1000:
            self.latency_samples.pop(0)
    
    def get_stats(self):
        total_requests = self.allowed_requests + self.blocked_requests
        block_rate = self.blocked_requests / total_requests if total_requests > 0 else 0
        avg_latency = sum(self.latency_samples) / len(self.latency_samples) if self.latency_samples else 0
        
        return {
            'total_requests': total_requests,
            'allowed_requests': self.allowed_requests,
            'blocked_requests': self.blocked_requests,
            'block_rate': block_rate,
            'average_latency_ms': avg_latency * 1000
        }
```

## 🎯 Advanced Features

### **Hierarchical Rate Limiting**
```python
class HierarchicalRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.limiters = {}
    
    def check_limits(self, user_id, api_endpoint):
        limits_to_check = [
            f"user:{user_id}",           # Per user
            f"api:{api_endpoint}",       # Per API endpoint
            f"user_api:{user_id}:{api_endpoint}"  # Per user per API
        ]
        
        for limit_key in limits_to_check:
            config = self.get_limit_config(limit_key)
            if config and not self.is_allowed(limit_key, config['limit'], config['window']):
                return False, f"Rate limit exceeded for {limit_key}"
        
        return True, "Allowed"
```

## 🔗 Liên kết và Tham khảo

### **Related Topics**
- [[02-System-Components/Load-Balancers/Load-Balancer-Overview|Load Balancer]] - Request distribution
- [[02-System-Components/API-Gateway/API-Gateway-Pattern|API Gateway]] - Centralized rate limiting
- [[02-System-Components/Caching/Caching-Strategies|Caching]] - Performance optimization
- [[Redis]] - Distributed storage

### **Further Reading**
- "Designing Data-Intensive Applications" - Martin Kleppmann
- "High Performance Browser Networking" - Ilya Grigorik
- Redis documentation on rate limiting
- Kong API Gateway rate limiting plugins
- Rate limiting algorithms comparison
- Redis Lua scripting guide  
- API rate limiting best practices
- Distributed systems clock synchronization

---

**Practice Exercise**: Implement rate limiter với configurable algorithms, test performance với 10K concurrent users, và measure accuracy và latency. 