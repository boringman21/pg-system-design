# Cache Invalidation Strategies

**Tags**: #caching #performance #invalidation #consistency
**Ngày tạo**: 2024-01-01
**Độ khó**: Intermediate
**Thời gian đọc**: 10-12 phút

## 🎯 Cache Invalidation Overview

Cache invalidation là one of the hardest problems in computer science. Strategies để ensure cached data remains consistent với underlying data source.

### **Core Challenges**
- **Stale Data**: Cached data becomes outdated
- **Consistency**: Cache vs database synchronization  
- **Performance**: Invalidation overhead
- **Complexity**: Managing dependencies

---

## 🔄 Invalidation Strategies

### **1. Time-Based Expiration (TTL)**
```python
# Simple TTL-based cache
class TTLCache:
    def __init__(self, default_ttl=300):  # 5 minutes
        self.cache = {}
        self.expiry_times = {}
        self.default_ttl = default_ttl
    
    def set(self, key, value, ttl=None):
        ttl = ttl or self.default_ttl
        expiry_time = time.time() + ttl
        
        self.cache[key] = value
        self.expiry_times[key] = expiry_time
    
    def get(self, key):
        if key not in self.cache:
            return None
        
        # Check if expired
        if time.time() > self.expiry_times[key]:
            self.delete(key)
            return None
        
        return self.cache[key]
    
    def delete(self, key):
        self.cache.pop(key, None)
        self.expiry_times.pop(key, None)
```

**Pros**: Simple, predictable
**Cons**: May serve stale data, wasteful refreshing

### **2. Write-Through Invalidation**
```python
# Invalidate immediately on write
class WriteThroughCache:
    def __init__(self, database, cache):
        self.db = database
        self.cache = cache
    
    def read(self, key):
        # Try cache first
        value = self.cache.get(key)
        if value is not None:
            return value
        
        # Cache miss - read from database
        value = self.db.get(key)
        if value is not None:
            self.cache.set(key, value)
        
        return value
    
    def write(self, key, value):
        # Write to database first
        self.db.set(key, value)
        
        # Update cache immediately
        self.cache.set(key, value)
    
    def delete(self, key):
        # Delete from database
        self.db.delete(key)
        
        # Invalidate cache
        self.cache.delete(key)
```

**Pros**: Strong consistency
**Cons**: Write latency, cache pollution

### **3. Write-Behind Invalidation**
```python
# Batch invalidation for performance
class WriteBehindCache:
    def __init__(self, database, cache):
        self.db = database
        self.cache = cache
        self.dirty_keys = set()
        self.invalidation_queue = Queue()
        self.start_background_invalidator()
    
    def read(self, key):
        return self.cache.get(key) or self.db.get(key)
    
    def write(self, key, value):
        # Write to cache immediately
        self.cache.set(key, value)
        
        # Mark as dirty for later database sync
        self.dirty_keys.add(key)
        
        # Queue for invalidation
        self.invalidation_queue.put({
            'key': key,
            'value': value,
            'timestamp': time.time()
        })
    
    def start_background_invalidator(self):
        def invalidator():
            while True:
                try:
                    # Batch process invalidations
                    self.process_invalidation_batch()
                    time.sleep(1)  # Process every second
                except Exception as e:
                    self.log_error(e)
        
        threading.Thread(target=invalidator, daemon=True).start()
    
    def process_invalidation_batch(self):
        batch = []
        
        # Collect batch of updates
        while not self.invalidation_queue.empty() and len(batch) < 100:
            batch.append(self.invalidation_queue.get())
        
        if batch:
            # Batch write to database
            self.db.batch_write(batch)
            
            # Remove from dirty set
            for item in batch:
                self.dirty_keys.discard(item['key'])
```

**Pros**: High write performance
**Cons**: Risk of data loss, eventual consistency

---

## 🎯 Event-Driven Invalidation

### **Cache Invalidation via Events**
```python
# Event-driven cache invalidation
class EventDrivenCache:
    def __init__(self, cache, event_bus):
        self.cache = cache
        self.event_bus = event_bus
        self.dependency_graph = {}
        
        # Subscribe to data change events
        self.event_bus.subscribe('data_updated', self.handle_data_update)
        self.event_bus.subscribe('data_deleted', self.handle_data_delete)
    
    def set_with_dependencies(self, key, value, depends_on=None):
        self.cache.set(key, value)
        
        # Track dependencies
        if depends_on:
            for dep in depends_on:
                if dep not in self.dependency_graph:
                    self.dependency_graph[dep] = set()
                self.dependency_graph[dep].add(key)
    
    def handle_data_update(self, event):
        updated_key = event['key']
        
        # Invalidate directly affected cache
        self.cache.delete(updated_key)
        
        # Invalidate dependent caches
        if updated_key in self.dependency_graph:
            for dependent_key in self.dependency_graph[updated_key]:
                self.cache.delete(dependent_key)
                
                # Publish cascade invalidation event
                self.event_bus.publish('cache_invalidated', {
                    'key': dependent_key,
                    'reason': f'dependency_on_{updated_key}'
                })
    
    def handle_data_delete(self, event):
        deleted_key = event['key']
        self.invalidate_cascade(deleted_key)
    
    def invalidate_cascade(self, key):
        # Recursive dependency invalidation
        self.cache.delete(key)
        
        if key in self.dependency_graph:
            for dependent in self.dependency_graph[key]:
                self.invalidate_cascade(dependent)
```

### **Smart Cache Tags**
```python
# Tag-based invalidation for complex dependencies
class TaggedCache:
    def __init__(self, cache):
        self.cache = cache
        self.tag_to_keys = {}  # tag -> set of keys
        self.key_to_tags = {}  # key -> set of tags
    
    def set(self, key, value, tags=None, ttl=None):
        self.cache.set(key, value, ttl)
        
        if tags:
            self.key_to_tags[key] = set(tags)
            
            for tag in tags:
                if tag not in self.tag_to_keys:
                    self.tag_to_keys[tag] = set()
                self.tag_to_keys[tag].add(key)
    
    def invalidate_by_tag(self, tag):
        """Invalidate all cache entries with given tag"""
        if tag in self.tag_to_keys:
            keys_to_invalidate = self.tag_to_keys[tag].copy()
            
            for key in keys_to_invalidate:
                self.delete(key)
            
            return len(keys_to_invalidate)
        return 0
    
    def delete(self, key):
        # Remove from cache
        self.cache.delete(key)
        
        # Clean up tag mappings
        if key in self.key_to_tags:
            tags = self.key_to_tags[key]
            
            for tag in tags:
                if tag in self.tag_to_keys:
                    self.tag_to_keys[tag].discard(key)
                    
                    # Clean up empty tag sets
                    if not self.tag_to_keys[tag]:
                        del self.tag_to_keys[tag]
            
            del self.key_to_tags[key]

# Usage example
tagged_cache = TaggedCache(redis_cache)

# Cache user profile with tags
tagged_cache.set(
    'user:123:profile',
    user_profile_data,
    tags=['user:123', 'profiles', 'active_users']
)

# Cache user posts with tags
tagged_cache.set(
    'user:123:posts',
    user_posts_data,
    tags=['user:123', 'posts', 'user_content']
)

# When user data changes, invalidate all related caches
tagged_cache.invalidate_by_tag('user:123')
```

---

## 📊 Invalidation Patterns

### **1. Lazy Invalidation**
```python
class LazyInvalidationCache:
    def __init__(self, cache, database):
        self.cache = cache
        self.db = database
        self.version_tracker = {}
    
    def get(self, key):
        # Get cached value với version
        cached_item = self.cache.get(key)
        if cached_item is None:
            return self.load_and_cache(key)
        
        # Check if version is still valid
        current_version = self.db.get_version(key)
        if cached_item['version'] != current_version:
            # Stale data - reload
            return self.load_and_cache(key)
        
        return cached_item['value']
    
    def load_and_cache(self, key):
        value = self.db.get(key)
        version = self.db.get_version(key)
        
        cached_item = {
            'value': value,
            'version': version,
            'cached_at': time.time()
        }
        
        self.cache.set(key, cached_item)
        return value
```

### **2. Proactive Refresh**
```python
class ProactiveRefreshCache:
    def __init__(self, cache, database, refresh_threshold=0.8):
        self.cache = cache
        self.db = database
        self.refresh_threshold = refresh_threshold
        self.refresh_executor = ThreadPoolExecutor(max_workers=5)
    
    def get(self, key):
        cached_item = self.cache.get(key)
        
        if cached_item is None:
            # Cache miss
            return self.load_and_cache(key)
        
        # Check if needs proactive refresh
        age = time.time() - cached_item['cached_at']
        ttl = cached_item.get('ttl', 300)
        
        if age > (ttl * self.refresh_threshold):
            # Trigger background refresh
            self.refresh_executor.submit(self.background_refresh, key)
        
        return cached_item['value']
    
    def background_refresh(self, key):
        try:
            fresh_value = self.db.get(key)
            
            cached_item = {
                'value': fresh_value,
                'cached_at': time.time(),
                'ttl': 300
            }
            
            self.cache.set(key, cached_item)
            
        except Exception as e:
            self.log_refresh_error(key, e)
```

---

## ⚠️ Anti-Patterns & Pitfalls

### **Cache Stampede**
```python
# Problem: Multiple threads refresh same cache simultaneously
class StampedeProtection:
    def __init__(self, cache, database):
        self.cache = cache
        self.db = database
        self.refresh_locks = {}
        self.lock = threading.Lock()
    
    def get_with_stampede_protection(self, key):
        cached_value = self.cache.get(key)
        
        if cached_value is not None:
            return cached_value
        
        # Try to acquire refresh lock
        with self.lock:
            if key in self.refresh_locks:
                # Another thread is refreshing, wait briefly
                time.sleep(0.01)
                return self.cache.get(key)  # Try cache again
            
            # Acquire refresh lock
            self.refresh_locks[key] = threading.Lock()
        
        try:
            with self.refresh_locks[key]:
                # Double-check cache (another thread might have updated)
                cached_value = self.cache.get(key)
                if cached_value is not None:
                    return cached_value
                
                # Load from database
                fresh_value = self.db.get(key)
                self.cache.set(key, fresh_value)
                return fresh_value
                
        finally:
            # Release refresh lock
            with self.lock:
                if key in self.refresh_locks:
                    del self.refresh_locks[key]
```

### **Cascading Failures**
```python
class CircuitBreakerCache:
    def __init__(self, cache, database, failure_threshold=5):
        self.cache = cache
        self.db = database
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.last_failure_time = 0
        self.circuit_open = False
    
    def get(self, key):
        cached_value = self.cache.get(key)
        
        if cached_value is not None:
            return cached_value
        
        # Check circuit breaker
        if self.circuit_open:
            if time.time() - self.last_failure_time > 60:  # 1 minute
                self.circuit_open = False
                self.failure_count = 0
            else:
                # Return stale data if available
                return self.get_stale_data(key)
        
        try:
            fresh_value = self.db.get(key)
            self.cache.set(key, fresh_value)
            
            # Reset failure count on success
            self.failure_count = 0
            return fresh_value
            
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.circuit_open = True
            
            # Return stale data as fallback
            return self.get_stale_data(key)
```

---

## 🚀 Best Practices

### **1. Choose Right Strategy**
```python
def choose_invalidation_strategy(use_case):
    if use_case['consistency'] == 'strong':
        return 'write_through'
    elif use_case['write_heavy']:
        return 'write_behind'
    elif use_case['read_heavy']:
        return 'ttl_with_refresh'
    else:
        return 'event_driven'
```

### **2. Monitor Cache Health**
```python
class CacheMetrics:
    def __init__(self):
        self.hit_count = 0
        self.miss_count = 0
        self.invalidation_count = 0
        self.refresh_count = 0
    
    def record_hit(self):
        self.hit_count += 1
    
    def record_miss(self):
        self.miss_count += 1
    
    def get_hit_ratio(self):
        total = self.hit_count + self.miss_count
        return self.hit_count / total if total > 0 else 0
    
    def get_stats(self):
        return {
            'hit_ratio': self.get_hit_ratio(),
            'total_requests': self.hit_count + self.miss_count,
            'invalidations': self.invalidation_count,
            'refreshes': self.refresh_count
        }
```

---

**🎯 There are only two hard things in Computer Science: cache invalidation và naming things. Choose your strategy wisely!**

## 📚 Related Topics
- [[02-System-Components/Caching/Caching-Strategies|Caching Strategies]]
- [[02-System-Components/CDN/CDN-Architecture|CDN Architecture]]
- [[06-Database-Design/Consistency-Models/Data-Consistency-Models|Data Consistency Models]] 