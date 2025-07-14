# System Design Quick Reference

**Tags**: #reference #cheatsheet #quick-guide
**Date**: 2024-01-01

## 🚀 Interview Framework

### **1. Requirements (5-10 minutes)**
```
Functional Requirements:
- What features do we need?
- What are the core use cases?

Non-functional Requirements:  
- How many users?
- How much data?
- What's the read/write ratio?
- Latency requirements?
- Availability requirements?
```

### **2. Estimation (5 minutes)**
```
Traffic: QPS = Daily Active Users * Avg requests per user / 86400
Storage: Daily data * 365 * 5 years
Bandwidth: QPS * Average request size
Memory: 80/20 rule - cache 20% of daily data
```

### **3. High-Level Design (10-15 minutes)**
```
[Client] → [CDN] → [Load Balancer] → [App Servers]
                                           ↓
                                    [Cache] ← [Database]
                                           ↓
                                    [Message Queue]
```

### **4. Detailed Design (15-20 minutes)**
- API design
- Database schema  
- Algorithm details
- Handle bottlenecks

### **5. Scale & Optimize (5-10 minutes)**
- Database sharding
- Read replicas
- Caching strategies
- Load balancing

## 📊 Capacity Planning

### **Common Numbers**
```
L1 cache reference: 0.5 ns
L2 cache reference: 7 ns
RAM access: 100 ns
SSD random read: 150 μs
Network round trip (same DC): 0.5 ms
Network round trip (different coast): 150 ms
```

### **Scale Estimates**
```
Small: 1K users, 100 QPS
Medium: 100K users, 10K QPS  
Large: 10M users, 1M QPS
Massive: 1B users, 100M QPS
```

## 🗃️ Database Patterns

### **SQL vs NoSQL**
```
Use SQL when:
✅ ACID transactions needed
✅ Complex relationships
✅ Strong consistency
✅ Complex queries

Use NoSQL when:  
✅ Massive scale
✅ Flexible schema
✅ Simple queries
✅ Eventual consistency OK
```

### **Sharding Strategies**
```
By User ID: hash(user_id) % num_shards
By Geographic: users_west, users_east
By Feature: user_data, user_messages
```

## 🔄 Caching

### **Cache Levels**
```
L1: Browser (seconds)
L2: CDN (hours)
L3: Load Balancer (minutes)  
L4: Application (hours)
L5: Database (persistent)
```

### **Cache Patterns**
```
Cache-Aside: App manages cache
Write-Through: Update cache + DB
Write-Behind: Cache first, DB async
Refresh-Ahead: Proactive refresh
```

## ⚖️ Load Balancing

### **Algorithms**
```
Round Robin: Next server in sequence
Weighted RR: Based on server capacity
Least Connections: Fewest active connections
IP Hash: Route by client IP
```

### **Health Checks**
```
HTTP: GET /health → 200 OK
TCP: Port connectivity check
Custom: Application-specific logic
```

## 📡 Communication

### **Sync vs Async**
```
Synchronous:
✅ Simple, immediate response
❌ Tight coupling, cascade failures

Asynchronous:  
✅ Loose coupling, better scale
❌ Complex, eventual consistency
```

### **Message Queues**
```
Kafka: High throughput, streaming
RabbitMQ: Reliable, feature-rich
SQS: Managed, auto-scaling
Redis: In-memory, pub/sub
```

## 🔐 Security Essentials

### **Authentication**
```
JWT: Stateless, scalable
OAuth: Third-party integration
Sessions: Server-side state
API Keys: Service-to-service
```

### **Rate Limiting**
```
Token Bucket: Burst traffic OK
Fixed Window: Simple implementation
Sliding Window: Accurate but complex
```

## 🎯 Common Architectures

### **Microservices**
```
✅ Independent deployment
✅ Technology diversity
✅ Fault isolation
❌ Network complexity
❌ Data consistency
```

### **Event-Driven**
```
✅ Loose coupling
✅ Async processing
✅ Easy to scale
❌ Complex debugging
❌ Event ordering
```

## 📈 Scaling Patterns

### **Database Scaling**
```
Read Replicas: Scale reads
Sharding: Scale writes
Partitioning: Vertical split
Federation: Split by function
```

### **Application Scaling**
```
Horizontal: More servers
Vertical: Bigger servers
Auto-scaling: Dynamic capacity
Load balancing: Distribute traffic
```

## 🚨 Common Mistakes

### **Interview Mistakes**
- ❌ Jump to solution without requirements
- ❌ Over-engineer from the start
- ❌ Ignore data consistency
- ❌ Forget about monitoring
- ❌ Not consider failure scenarios

### **Design Mistakes**
- ❌ Single point of failure
- ❌ Not planning for scale
- ❌ Ignoring security
- ❌ Poor data modeling
- ❌ No error handling

## 🔧 Technology Choices

### **Databases**
```
MySQL: ACID, relations, OLTP
PostgreSQL: Advanced SQL, JSON
MongoDB: Document, flexible schema
Cassandra: Wide column, high write
Redis: In-memory, caching
```

### **Message Systems**
```
Kafka: Event streaming, high throughput
RabbitMQ: Message broker, reliability
Apache Pulsar: Multi-tenant, geo-replication
Amazon SQS: Managed, serverless
```

### **Caching**
```
Redis: Data structures, persistence
Memcached: Simple, fast
CDN: Geographic, static content
Application: In-process, fastest
```

## 💡 Problem-Solving Tips

### **Start Simple**
1. Identify core features
2. Basic architecture
3. Add complexity gradually
4. Address bottlenecks

### **Think Trade-offs**
- Consistency vs Availability
- Performance vs Cost
- Complexity vs Maintainability
- Security vs Usability

### **Ask Questions**
- "What if...?" scenarios
- Failure cases
- Edge cases
- Future growth

---

**Quick Checklist cho Interview:**
- [ ] Clarify requirements upfront
- [ ] Estimate capacity needs
- [ ] Start với simple design
- [ ] Identify bottlenecks
- [ ] Scale the design
- [ ] Consider failure scenarios
- [ ] Discuss monitoring & alerting

**Remember**: There's no perfect solution, chỉ có trade-offs! 🎯 