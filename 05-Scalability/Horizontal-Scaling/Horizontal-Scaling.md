# Horizontal Scaling

**Tags**: #scalability #horizontal-scaling #distributed
**Date**: 2024-01-01

## 📝 Overview

Horizontal scaling (scale out) là process của adding more servers để handle increased load, thay vì upgrade existing server (vertical scaling). Đây là key strategy cho large-scale systems.

## 🎯 Horizontal vs Vertical Scaling

### **Horizontal Scaling (Scale Out)**
```
Before: [Server 1]
After:  [Server 1] [Server 2] [Server 3]
```
- Add more machines
- Distribute load across multiple servers
- Linear scalability potential
- Higher availability

### **Vertical Scaling (Scale Up)**
```
Before: [4 CPU, 8GB RAM]
After:  [8 CPU, 16GB RAM]
```
- Upgrade existing machine
- Limited by hardware constraints
- Single point of failure
- Simpler to implement

## 🔧 Implementation Strategies

### **Load Balancing**
```python
class LoadBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.current = 0
    
    def get_server(self):
        # Round robin
        server = self.servers[self.current]
        self.current = (self.current + 1) % len(self.servers)
        return server
```

### **Database Sharding**
```sql
-- Shard by user_id
SELECT * FROM users_shard_1 WHERE user_id % 4 = 1;
SELECT * FROM users_shard_2 WHERE user_id % 4 = 2;
-- ...
```

### **Stateless Services**
```python
# ❌ Stateful - hard to scale
class StatefulService:
    def __init__(self):
        self.user_sessions = {}  # In-memory state
    
    def handle_request(self, user_id, data):
        session = self.user_sessions.get(user_id)
        # Process with session state

# ✅ Stateless - easy to scale
class StatelessService:
    def handle_request(self, user_id, data):
        session = self.session_store.get(user_id)  # External storage
        # Process with session data
```

## 📊 Benefits & Challenges

### **Benefits**
- ✅ **Linear Scaling**: Add capacity by adding servers
- ✅ **Fault Tolerance**: Single server failure doesn't bring down system
- ✅ **Cost Effective**: Use commodity hardware
- ✅ **Geographic Distribution**: Servers in multiple regions

### **Challenges**
- ❌ **Complexity**: Distributed system complexity
- ❌ **Data Consistency**: Maintain consistency across servers
- ❌ **Network Latency**: Communication between servers
- ❌ **Operational Overhead**: More servers to manage

## 🎯 When to Use

**Use Horizontal Scaling When:**
- Traffic is growing rapidly
- Need high availability
- Cost optimization important
- Workload can be partitioned

**Use Vertical Scaling When:**
- Simple application
- Strong consistency required
- Limited operational resources
- Quick short-term solution needed

## 🔗 Liên kết và Tham khảo

### **Related Topics**
- [[02-System-Components/Load-Balancers/Load-Balancer-Overview|Load Balancer]] - Distribute traffic
- [[06-Database-Design/Sharding/Database-Sharding|Database Sharding]] - Partition data
- [[07-Microservices/Microservices-Architecture|Microservices]] - Service decomposition
- [[Auto Scaling]] - Dynamic scaling

### **Scaling Strategies**
- **Application Layer**: Stateless services, load balancing
- **Database Layer**: Read replicas, sharding, caching
- **Storage Layer**: Distributed file systems, object storage
- **Network Layer**: CDN, edge computing

### **Tools và Platforms**
- **Cloud Platforms**: AWS Auto Scaling, Google Cloud, Azure Scale Sets
- **Container Orchestration**: Kubernetes HPA, Docker Swarm
- **Load Balancers**: NGINX, HAProxy, AWS ALB/NLB
- **Monitoring**: Prometheus, Grafana, CloudWatch

---

*Horizontal scaling provides better fault tolerance và cost efficiency so với vertical scaling. Design applications to be stateless và distributed from the start.* 