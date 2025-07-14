# Load Balancer - Overview

**Tags**: #component #load-balancer #scalability
**Date**: 2024-01-01

## 📝 Overview

Load Balancer là một component quan trọng trong system architecture, có nhiệm vụ phân phối requests đến multiple servers để đảm bảo:
- **Performance**: Tránh overload một server
- **Availability**: Failover khi server down
- **Scalability**: Dễ dàng thêm/bớt servers

## 🎯 Purpose & Use Cases

### Primary Functions
- Distribute incoming requests across multiple servers
- Health checking và failover
- SSL termination
- Session persistence
- Rate limiting

### Common Use Cases
- Web applications với multiple app servers
- API gateways
- Database connection pooling
- Microservices traffic routing

## 🏗️ How It Works

### Architecture
```
Client Requests → Load Balancer → Server Pool
                                    ├── Server 1
                                    ├── Server 2
                                    └── Server 3
```

### Key Concepts
- **Upstream**: Servers behind load balancer
- **Health Check**: Monitor server availability
- **Sticky Session**: Route user to same server
- **Round Robin**: Distribute requests evenly

### Workflow
1. Client gửi request đến Load Balancer
2. Load Balancer chọn server dựa trên algorithm
3. Forward request đến server được chọn
4. Return response về client

## 🔧 Types & Variants

| Type | Layer | Description | Use Case | Pros | Cons |
|------|-------|-------------|----------|------|------|
| **L4 (Transport)** | TCP/UDP | Route based on IP & port | High performance | Fast, low latency | Less intelligent routing |
| **L7 (Application)** | HTTP/HTTPS | Route based on content | Advanced routing | Feature-rich, content-aware | Higher latency, more CPU |
| **Hardware** | Physical | Dedicated appliances | Enterprise | High performance | Expensive, less flexible |
| **Software** | Application | Software-based | Cloud, startups | Flexible, cost-effective | Requires maintenance |

## ⚙️ Load Balancing Algorithms

### **Round Robin**
```python
servers = [server1, server2, server3]
current = 0

def get_server():
    global current
    server = servers[current]
    current = (current + 1) % len(servers)
    return server
```

### **Weighted Round Robin**
- Assign weights dựa trên server capacity
- High-spec servers nhận nhiều requests hơn

### **Least Connections**
- Route đến server có ít active connections nhất
- Good cho long-lived connections

### **IP Hash**
- Hash client IP để determine server
- Đảm bảo same client → same server

### **Least Response Time**
- Combine least connections + response time
- Route đến server fastest & least loaded

## 📊 Performance Characteristics

### Metrics
- **Throughput**: Requests per second
- **Latency**: Added delay (usually <1ms for L4)
- **Availability**: Uptime percentage
- **Connection Handling**: Concurrent connections

### L4 vs L7 Performance
```
L4 Load Balancer:
- Throughput: 1M+ RPS
- Latency: <1ms
- Memory: Low

L7 Load Balancer:  
- Throughput: 100K+ RPS
- Latency: 1-5ms
- Memory: Higher (content inspection)
```

## 🎯 Trade-offs

### Advantages
- ✅ **Scalability**: Easy horizontal scaling
- ✅ **Availability**: Automatic failover
- ✅ **Performance**: Distribute load evenly
- ✅ **Flexibility**: Traffic shaping & routing

### Disadvantages
- ❌ **Single Point of Failure**: Need HA setup
- ❌ **Complexity**: Additional infrastructure
- ❌ **Cost**: Hardware/software costs
- ❌ **Latency**: Additional network hop

## 🔧 Popular Load Balancers

### **Cloud Load Balancers**
- **AWS ALB/NLB**: Application & Network Load Balancer
- **Google Cloud Load Balancer**: Global load balancing
- **Azure Load Balancer**: L4 load balancing

### **Software Load Balancers**
- **NGINX**: High-performance web server + LB
- **HAProxy**: Open-source TCP/HTTP load balancer
- **Envoy**: Service mesh proxy
- **Cloudflare**: Global CDN + load balancing

### **Hardware Load Balancers**
- **F5 BIG-IP**: Enterprise-grade appliances
- **Citrix NetScaler**: Application delivery controller

## 🚨 Common Pitfalls

- **No Health Checks**: Routing to failed servers
- **Session Stickiness**: Can cause uneven load
- **SSL Termination**: Certificate management complexity
- **Single LB**: No redundancy/failover

## 🔗 Integration

### Works Well With
- [[Auto Scaling Groups]]
- [[CDN]]
- [[API Gateway]]
- [[Service Discovery]]

### Alternatives
- [[DNS Round Robin]]
- [[Client-side Load Balancing]]
- [[Service Mesh]]

## 📚 Learning Resources

### Documentation
- NGINX Load Balancing Guide
- AWS Elastic Load Balancing
- HAProxy Configuration Manual

### Best Practices
- Implement health checks
- Use multiple AZs/regions
- Monitor key metrics
- Plan for failover scenarios

## 🧪 Hands-on Practice

### Lab Exercises
- [ ] Setup NGINX load balancer với 3 backend servers
- [ ] Implement health checks và failover
- [ ] Compare L4 vs L7 performance
- [ ] Configure SSL termination

### Projects to Try
- Build a simple web app với load balancer
- Implement session persistence
- Set up multi-region load balancing
- Create custom health check endpoints

---

**Study Notes**: 
- Load balancers là building block quan trọng cho scalable systems
- Chọn L4 vs L7 dựa trên requirements
- Always implement health checks và redundancy 