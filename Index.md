# System Design Interview Prep - Index

**Tags**: #index #overview
**Date**: 2024-01-01

## 🏗️ Kiến thức nền tảng

### 01-Fundamentals
- [[01-Fundamentals/CAP-Theorem/CAP-Theorem|CAP Theorem]] - Consistency, Availability, Partition Tolerance
- [[01-Fundamentals/ACID-Properties/ACID-Properties|ACID Properties]] - Database transaction properties
- [[01-Fundamentals/Consistency-Models/Consistency-Models|Consistency Models]] - Strong, eventual, weak consistency
- [[01-Fundamentals/Distributed-Systems/Distributed-Systems-Fundamentals|Distributed Systems]] - Core concepts

### 02-System Components
- [[02-System-Components/Load-Balancers/Load-Balancer-Overview|Load Balancer]] - Traffic distribution and failover
- [[02-System-Components/Caching/Caching-Strategies|Caching]] - Redis, Memcached, CDN strategies
- [[02-System-Components/Message-Queues/Message-Queue-Systems|Message Queues]] - Kafka, RabbitMQ, SQS
- [[02-System-Components/API-Gateway/API-Gateway-Pattern|API Gateway]] - Request routing and management
- [[02-System-Components/CDN/CDN-Architecture|CDN]] - Content delivery networks
- [[02-System-Components/Search-Systems/Search-Architecture|Search Systems]] - Elasticsearch, search optimization

### 03-Design Patterns
- [[03-Design-Patterns/Microservices-Pattern|Microservices patterns]]
- [[03-Design-Patterns/Event-Driven-Architecture|Event-driven architecture]]
- [[03-Design-Patterns/CQRS-Pattern|CQRS (Command Query Responsibility Segregation)]]
- [[03-Design-Patterns/Saga-Pattern|Saga pattern]]

## 🎯 Case Studies

### 04-Case Studies
- [[04-Case-Studies/URL-Shortener/URL-Shortener-Design|URL Shortener]] - bit.ly, TinyURL design
- [[04-Case-Studies/Chat-Systems/Chat-System-Design|Chat System]] - WhatsApp, Slack architecture
- [[04-Case-Studies/Twitter-Timeline/Twitter-System-Design|Twitter Timeline]] - Social media timeline design
- [[04-Case-Studies/Video-Streaming/Video-Streaming-Design|Video Streaming]] - YouTube, Netflix design
- [[04-Case-Studies/E-Commerce/E-Commerce-Platform-Design|E-Commerce]] - Amazon, marketplace design
- [[04-Case-Studies/Search-Engine/Search-Engine-System-Design|Search Engine]] - Google search architecture
- [[04-Case-Studies/Payment-System/Payment-System-Design|Payment System]] - Stripe, PayPal design
- [[04-Case-Studies/Notification-System/Notification-System-Design|Notification System]] - Push notifications design
- [[04-Case-Studies/File-Storage/File-Storage-System-Design|File Storage]] - Dropbox, Google Drive design
- [[04-Case-Studies/Real-time-Analytics/Real-time-Analytics-Design|Real-time Analytics]] - Analytics pipeline design

## 🚀 Scaling & Performance

### 05-Scalability
- [[05-Scalability/Horizontal-Scaling/Horizontal-Scaling|Horizontal Scaling]] - Adding more servers
- [[05-Scalability/Vertical-Scaling/Vertical-Scaling|Vertical Scaling]] - Upgrading server specs
- [[05-Scalability/Auto-Scaling/Auto-Scaling|Auto Scaling]] - Dynamic resource allocation

### 06-Database Design
- [[06-Database-Design/SQL-Databases/SQL-Database-Design|SQL Databases]] - MySQL, PostgreSQL
- [[06-Database-Design/NoSQL-Databases/NoSQL-Database-Types|NoSQL Databases]] - MongoDB, Cassandra, DynamoDB
- [[06-Database-Design/Sharding/Database-Sharding|Sharding]] - Horizontal partitioning
- [[06-Database-Design/Replication/Database-Replication|Replication]] - Master-slave, master-master
- [[06-Database-Design/Partitioning/Database-Partitioning|Partitioning]] - Database partitioning strategies
- [[06-Database-Design/Indexing/Indexing-Strategies|Indexing]] - Database indexing strategies
- [[06-Database-Design/Transaction-Management/Transaction-Management|Transaction Management]] - ACID transactions

### 08-Performance
- [[08-Performance/Monitoring/Performance-Monitoring|Monitoring]] - Metrics, logging, alerting
- [[08-Performance/Optimization/Performance-Optimization|Optimization]] - Query optimization, indexing
- [[08-Performance/Profiling/Performance-Profiling|Profiling]] - Performance bottleneck identification

## 🔒 Advanced Topics

### 07-Microservices
- [[07-Microservices/Microservices-Architecture|Service decomposition]]
- Inter-service communication
- Service discovery
- Circuit breaker pattern

### 09-Security
- [[09-Security/Authentication-Authorization|Authentication & authorization]]
- Data encryption
- Security patterns
- Best practices

### 10-DevOps & Deployment
- [[10-DevOps-Deployment/DevOps-Best-Practices|CI/CD pipelines]]
- Container orchestration
- Infrastructure as code
- Monitoring & alerting

## 📝 Practice & Assessment

### 11-Interview Questions
- [[11-Interview-Questions/FAANG-Questions|FAANG Questions]] - Google, Amazon, Facebook style
- [[11-Interview-Questions/Startup-Questions|Startup Questions]] - Smaller company focus
- [[11-Interview-Questions/FAANG/Design-Twitter|Design Twitter]] - Twitter system design
- [[11-Interview-Questions/Startups/Startup-Common-Questions|Common Startup Questions]] - Startup-specific questions

### 12-Practice Problems
- [[12-Practice-Problems/Rate-Limiter-Design|Rate Limiter Design]] - Step-by-step solutions
- Alternative approaches
- Common mistakes to avoid
- Time complexity analysis

### 13-Advanced Distributed Systems
- [[13-Advanced-Distributed-Systems/Consensus-Algorithms|Consensus Algorithms]] - Raft, Paxos algorithms
- [[13-Advanced-Distributed-Systems/Distributed-Locking|Distributed Locking]] - Distributed locks and coordination
- [[13-Advanced-Distributed-Systems/Vector-Clocks|Vector Clocks]] - Distributed system ordering

## 🛠️ Templates

### Available Templates
- [[Templates/System-Design-Template|System Design Template]] - Structured approach
- [[Templates/Component-Template|Component Template]] - System component analysis
- [[Templates/Interview-Question-Template|Interview Question Template]] - Question breakdown
- [[Templates/Case-Study-Template|Case Study Template]] - Case study analysis framework
- [[Templates/Learning-Notes-Template|Learning Notes Template]] - Comprehensive learning notes

## 📈 Study Progress

### Learning Path
```
Week 1-2: Fundamentals
├── CAP Theorem
├── ACID Properties
├── Consistency Models
└── Distributed Systems Basics

Week 3-4: Core Components  
├── Load Balancers
├── Caching Strategies
├── Message Queues
└── API Design

Week 5-6: Database Design
├── SQL vs NoSQL
├── Sharding & Replication
├── Schema Design
└── Transactions

Week 7-8: Case Studies
├── URL Shortener
├── Chat System
├── Social Media Feed
└── Video Streaming

Week 9-10: Advanced Topics
├── Microservices
├── Security
├── Performance Optimization
└── DevOps Practices
```

### Milestones
- [ ] Complete fundamentals understanding
- [ ] Master core system components
- [ ] Design 5+ complete systems
- [ ] Practice mock interviews
- [ ] Understand trade-offs deeply

## 🔍 Quick Reference

### Common Patterns
- **Read-heavy systems**: Use caching, read replicas
- **Write-heavy systems**: Sharding, message queues
- **Real-time systems**: WebSockets, event streaming
- **Analytics systems**: Data warehouses, batch processing

### Scale Estimates
```python
# Quick calculations
def daily_to_second(daily_requests):
    return daily_requests / (24 * 3600)

def storage_estimate(records, size_per_record, years=5):
    return records * size_per_record * 365 * years

# Example: 1M daily users, 10 requests each
rps = daily_to_second(1_000_000 * 10)  # ~115 RPS
```

### Technology Choices
| Use Case | Technology Options |
|----------|--------------------|
| **Web Server** | NGINX, Apache, Node.js |
| **Load Balancer** | AWS ALB/NLB, HAProxy, NGINX |
| **Cache** | Redis, Memcached, CDN |
| **Database** | MySQL, PostgreSQL, MongoDB, Cassandra |
| **Message Queue** | Kafka, RabbitMQ, SQS |
| **Search** | Elasticsearch, Solr |
| **Monitoring** | Prometheus, Grafana, DataDog |

## 🏷️ Tag System

Sử dụng tags để organize và filter notes:

- `#fundamental` - Core concepts
- `#component` - System building blocks  
- `#case-study` - Real system designs
- `#scalability` - Scaling strategies
- `#database` - Data storage & retrieval
- `#performance` - Optimization techniques
- `#security` - Security considerations
- `#interview` - Interview-specific content
- `#practice` - Hands-on exercises
- `#template` - Templates and frameworks

## 🎯 Next Actions

1. **Start Learning**: Bắt đầu với [[01-Fundamentals/CAP-Theorem/CAP-Theorem|CAP Theorem]]
2. **Practice Daily**: Allocate 1-2 hours mỗi ngày
3. **Build Projects**: Implement simplified versions
4. **Mock Interviews**: Practice với peers hoặc mentors
5. **Stay Updated**: Follow system design blogs & papers

---

*Sử dụng Obsidian graph view để visualize connections giữa các concepts. Tags giúp filter và organize notes theo topics.* 