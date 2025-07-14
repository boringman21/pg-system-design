# URL Shortener - System Design

**Tags**: #case-study #design #url-shortener
**Date**: 2024-01-01
**Difficulty**: Easy

## 📝 Problem Statement

Thiết kế một URL shortening service như bit.ly hoặc tinyurl.com. System này cho phép users:
- Shorten long URLs thành short URLs
- Redirect từ short URLs về original URLs
- Analytics về clicks và usage

## 🎯 Requirements Clarification

### Functional Requirements
- [ ] **Shorten URL**: Convert long URL thành short URL
- [ ] **Redirect**: Short URL redirect về original URL
- [ ] **Custom URLs**: Users có thể tạo custom short URLs
- [ ] **Expiration**: URLs có thể có expiration date
- [ ] **Analytics**: Track clicks, locations, devices

### Non-functional Requirements
- [ ] **Scalability**: 100M URLs per day
- [ ] **Performance**: Redirect latency < 100ms
- [ ] **Availability**: 99.9% uptime
- [ ] **Consistency**: Eventually consistent OK

### Scale Estimates
- **Users**: 100M active users
- **URL creations**: 100M per day = 1150 per second
- **Redirects**: 10B per day = 115K per second (100:1 read/write ratio)
- **Storage**: 100M * 365 * 5 years = 182B URLs

## 🏗️ High-Level Design

```
[Client] → [Load Balancer] → [Web Servers] → [Cache] → [Database]
                                         ↓
                                   [Analytics DB]
```

### Core Components
1. **Load Balancer**: Distribute traffic
2. **Web Servers**: Handle API requests
3. **URL Encoding Service**: Generate short URLs
4. **Database**: Store URL mappings
5. **Cache**: Fast lookups for popular URLs
6. **Analytics Service**: Track metrics

## 🔍 Detailed Design

### API Design
```python
# Create short URL
POST /api/v1/shorten
{
    "originalUrl": "https://example.com/very/long/url",
    "customAlias": "mylink",  # optional
    "expirationDate": "2024-12-31"  # optional
}

# Redirect
GET /{shortUrl}
→ HTTP 302 redirect

# Get analytics
GET /api/v1/analytics/{shortUrl}
```

### Database Schema
```sql
-- URLs table
CREATE TABLE urls (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_url VARCHAR(7) NOT NULL UNIQUE,
    original_url TEXT NOT NULL,
    user_id INT,
    created_at TIMESTAMP,
    expires_at TIMESTAMP,
    click_count INT DEFAULT 0,
    INDEX(short_url),
    INDEX(user_id)
);

-- Analytics table
CREATE TABLE analytics (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_url VARCHAR(7),
    timestamp TIMESTAMP,
    ip_address VARCHAR(45),
    user_agent TEXT,
    country VARCHAR(3),
    device_type VARCHAR(20),
    INDEX(short_url, timestamp)
);
```

### URL Encoding Algorithm

#### **Approach 1: Base62 Encoding**
```python
import random
import string

BASE62 = string.ascii_letters + string.digits  # a-z, A-Z, 0-9

def encode_base62(number):
    if number == 0:
        return BASE62[0]
    
    result = []
    while number:
        number, remainder = divmod(number, 62)
        result.append(BASE62[remainder])
    
    return ''.join(reversed(result))

def generate_short_url():
    # Use database auto-increment ID
    db_id = get_next_id()  # e.g., 12345
    return encode_base62(db_id)  # returns "dnh"
```

#### **Approach 2: Random Generation**
```python
def generate_random_short_url(length=7):
    while True:
        short_url = ''.join(random.choices(BASE62, k=length))
        if not exists_in_database(short_url):
            return short_url
```

#### **Approach 3: MD5 Hash**
```python
import hashlib

def generate_hash_short_url(original_url):
    hash_value = hashlib.md5(original_url.encode()).hexdigest()
    return hash_value[:7]  # Take first 7 characters
```

## 📈 Scalability

### Bottlenecks
- **Database writes**: 1150 writes/second cho URL creation
- **Database reads**: 115K reads/second cho redirects
- **Single point of failure**: Database

### Solutions

#### **Database Scaling**
```
Master-Slave Replication:
- Master: Handle writes (URL creation)
- Multiple Slaves: Handle reads (redirects)
- Read/Write ratio: 100:1

Sharding:
- Shard by short_url hash
- Distribute load across multiple databases
```

#### **Caching Strategy**
```python
# Cache popular URLs
cache_key = f"url:{short_url}"
cache.set(cache_key, original_url, ttl=3600)

def redirect(short_url):
    # Try cache first
    original_url = cache.get(f"url:{short_url}")
    if original_url:
        return redirect_to(original_url)
    
    # Cache miss - query database
    original_url = database.get_original_url(short_url)
    if original_url:
        cache.set(f"url:{short_url}", original_url)
        return redirect_to(original_url)
    
    return not_found()
```

#### **CDN & Geographic Distribution**
- Deploy servers in multiple regions
- Use CDN cho static content
- Geo-DNS routing cho closest server

## 🔒 Security Considerations

- **Rate Limiting**: Prevent abuse (max 100 URLs/hour per user)
- **URL Validation**: Check malicious/spam URLs
- **Analytics Privacy**: Hash IP addresses
- **Custom URLs**: Prevent offensive/trademark conflicts

## 📊 Monitoring & Metrics

### Key Metrics
- **Creation Rate**: URLs created per second
- **Redirect Rate**: Redirects per second
- **Cache Hit Rate**: Percentage of cache hits
- **Error Rate**: 4xx/5xx responses
- **Latency**: P95/P99 response times

### Alerts
- Creation rate > 2000/sec
- Redirect latency > 200ms
- Cache hit rate < 80%
- Database connection errors

## 🎯 Trade-offs

| Decision | Pros | Cons |
|----------|------|------|
| Base62 vs Random | Predictable, shorter URLs | Enumerable, security risk |
| SQL vs NoSQL | ACID, relationships | Limited horizontal scaling |
| Cache vs No Cache | Fast reads, low DB load | Added complexity, stale data |
| Analytics Real-time | Up-to-date insights | Higher DB load |

## 🔗 Related Topics

- [[Load Balancer]]
- [[Database Sharding]]
- [[Caching Strategies]]
- [[CDN]]
- [[Rate Limiting]]

## 📚 Further Reading

- System Design Interview - Chapter 8
- Designing bit.ly architecture
- Base62 encoding algorithms
- Database sharding strategies

---

**Next Steps**: 
- [ ] Implement basic prototype
- [ ] Add custom URL feature
- [ ] Design analytics dashboard
- [ ] Plan for 10x traffic growth 