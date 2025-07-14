# Consistency Models

**Tags**: #fundamental #distributed-systems #consistency #cap-theorem
**Date**: 2024-01-01
**Related**: [[CAP Theorem]], [[ACID Properties]], [[Distributed Systems]], [[Database Replication]]

## 🎯 Tổng quan

Consistency models định nghĩa các rules về thứ tự và visibility của operations trong distributed systems. Đây là trade-off quan trọng giữa consistency, availability và performance.

## 📊 Phân loại Consistency Models

### **1. Strong Consistency Models**

#### **Linearizability (Strongest)**
```
Định nghĩa: Mọi operation xuất hiện như thể thực hiện tức thì tại một thời điểm
- Real-time ordering preserved
- Once a write completes, all subsequent reads see that value
- Feels like single machine
```

**Ví dụ:**
```
Client A: Write(x, 1) completed at 10:00:01
Client B: Read(x) at 10:00:02 must return 1
Client C: Read(x) at 10:00:03 must return 1
```

**Use cases:** Banking systems, atomic operations
**Trade-off:** Highest consistency, lowest availability & performance

#### **Sequential Consistency**
```
Định nghĩa: Mọi operations xuất hiện như thể thực hiện theo một thứ tự tuần tự
- Preserves program order for each client
- Same global ordering seen by all clients
- No real-time constraints
```

**Ví dụ:**
```
Client A: Write(x, 1), Write(y, 2)
Client B: Read(x), Read(y)

Valid outcomes:
- B sees: x=1, y=2 (A's writes happened first)
- B sees: x=0, y=0 (A's writes happened after)

Invalid outcomes:
- B sees: x=0, y=2 (violates A's program order)
```

### **2. Weak Consistency Models**

#### **Eventual Consistency**
```
Định nghĩa: Nếu không có updates mới, eventually tất cả replicas sẽ converge
- No guarantees about when convergence happens
- Conflicts must be resolved
- High availability và performance
```

**Ví dụ Amazon DynamoDB:**
```
Write to region US-East: User profile updated
Read from region EU-West: Might see old data temporarily
After propagation delay: All regions consistent
```

**Conflict Resolution:**
- Last-write-wins (timestamp)
- Vector clocks
- Application-specific logic
- CRDT (Conflict-free Replicated Data Types)

#### **Causal Consistency**
```
Định nghĩa: Operations that are causally related must be seen in same order
- Preserves cause-and-effect relationships
- Concurrent operations can be seen in different orders
```

**Ví dụ Chat System:**
```
Alice: "How are you?" (message A)
Bob: "I'm fine, thanks!" (message B, caused by A)

All users must see: A before B
But concurrent messages can appear in any order
```

**Implementation:**
```python
# Vector clocks example
class VectorClock:
    def __init__(self, node_id, nodes):
        self.node_id = node_id
        self.clock = {node: 0 for node in nodes}
    
    def tick(self):
        self.clock[self.node_id] += 1
        return self.clock.copy()
    
    def update(self, other_clock):
        for node in self.clock:
            self.clock[node] = max(self.clock[node], other_clock[node])
        self.clock[self.node_id] += 1
```

#### **Session Consistency**
```
Định nghĩa: Trong một session, reads see writes từ chính session đó
- Read your own writes
- Monotonic reads
- Monotonic writes
```

**Guarantees:**
```
Read Your Writes: Client luôn thấy writes của chính mình
Monotonic Reads: Subsequent reads không thấy older values
Monotonic Writes: Writes from same client applied in order
```

### **3. Hybrid Models**

#### **Bounded Staleness**
```
Định nghĩa: Data có thể stale nhưng trong giới hạn time/version bounds
- Max staleness: 5 minutes hoặc 100 versions
- Predictable consistency lag
```

**Azure Cosmos DB Example:**
```
Write to primary region: timestamp T
Read from secondary: guaranteed within 5 minutes of T
```

#### **Strong Eventual Consistency**
```
Định nghĩa: Replicas nhận same updates sẽ have same state
- Deterministic conflict resolution
- CRDTs (Conflict-free Replicated Data Types)
```

## 🔧 Implementation Patterns

### **Synchronous Replication (Strong Consistency)**
```python
def write_with_strong_consistency(key, value):
    # Write to primary
    primary.write(key, value)
    
    # Synchronously replicate to all replicas
    for replica in replicas:
        replica.write(key, value)
    
    # Only return success when all replicas confirm
    return "SUCCESS"

# Trade-off: High latency, lower availability
```

### **Asynchronous Replication (Eventual Consistency)**
```python
def write_with_eventual_consistency(key, value):
    # Write to primary immediately
    primary.write(key, value)
    
    # Asynchronously replicate to replicas
    async_replicate_to_all(key, value)
    
    # Return success immediately
    return "SUCCESS"

def async_replicate_to_all(key, value):
    for replica in replicas:
        background_task(replica.write, key, value)
```

### **Quorum-based (Tunable Consistency)**
```python
def write_with_quorum(key, value, W):
    """Write to W replicas before returning success"""
    successful_writes = 0
    
    for replica in replicas:
        try:
            replica.write(key, value)
            successful_writes += 1
            if successful_writes >= W:
                return "SUCCESS"
        except Exception:
            continue
    
    return "FAILURE"

def read_with_quorum(key, R):
    """Read from R replicas and return most recent"""
    values = []
    
    for replica in replicas[:R]:
        try:
            values.append(replica.read(key))
        except Exception:
            continue
    
    # Return value with highest timestamp/version
    return max(values, key=lambda v: v.timestamp)

# Strong consistency when R + W > N
# W=3, R=2, N=3 → Strong consistency
# W=1, R=1, N=3 → Eventual consistency
```

## 🎯 Choosing Consistency Models

### **Application Requirements Matrix**

| Application Type | Consistency Model | Reasoning |
|------------------|-------------------|-----------|
| **Banking** | Linearizability | Critical correctness, money can't be lost |
| **E-commerce Cart** | Session | User sees their own changes, eventual for others |
| **Social Media Feed** | Eventual | High availability, stale posts acceptable |
| **Collaborative Editing** | Causal | Preserve cause-effect, handle conflicts |
| **Gaming Leaderboard** | Bounded Staleness | Fresh enough, but not critical |
| **DNS** | Eventual | Global consistency eventually, high availability |

### **Decision Framework**
```
1. Can you tolerate stale reads? → Eventual Consistency
2. Need real-time global ordering? → Linearizability  
3. Need cause-effect preservation? → Causal Consistency
4. Per-user data consistency? → Session Consistency
5. Time-bounded staleness acceptable? → Bounded Staleness
```

## 📈 CAP Theorem Impact

### **CP Systems (Consistency + Partition Tolerance)**
```
Examples: MongoDB, HBase, Redis Cluster
- Strong consistency during partitions
- May become unavailable
- Good for: Financial systems, configuration data
```

### **AP Systems (Availability + Partition Tolerance)**
```  
Examples: Cassandra, DynamoDB, CouchDB
- Always available during partitions
- Eventual consistency
- Good for: Social media, content delivery, IoT
```

### **CA Systems (Consistency + Availability)**
```
Examples: Traditional RDBMS (MySQL, PostgreSQL)
- Strong consistency when no partitions
- Cannot handle network partitions gracefully
- Good for: Single-datacenter applications
```

## 🛠️ Database Examples

### **Strong Consistency**
```sql
-- PostgreSQL with synchronous replication
postgresql.conf:
synchronous_standby_names = 'standby1,standby2'
synchronous_commit = 'on'

-- Guarantees: Write not committed until replicas confirm
-- Trade-off: Higher latency, potential unavailability
```

### **Eventual Consistency**  
```javascript
// DynamoDB example
const params = {
    TableName: 'Users',
    Item: { id: '123', name: 'John' },
    ConsistentRead: false  // Eventually consistent
};

// Read might return stale data temporarily
// But eventually all replicas will be consistent
```

### **Tunable Consistency**
```python
# Cassandra example
session.execute(
    "INSERT INTO users (id, name) VALUES (%s, %s)",
    (123, 'John'),
    consistency_level=ConsistencyLevel.QUORUM  # W=2 out of N=3
)

result = session.execute(
    "SELECT * FROM users WHERE id = %s",
    (123,),
    consistency_level=ConsistencyLevel.ONE  # R=1
)
```

## 🔍 Interview Questions

### **Q1: Explain eventual consistency with real example**
```
A: Amazon S3 example:
- Upload file to US-East bucket
- File available immediately in US-East
- Takes seconds/minutes to replicate to other regions
- Eventually all regions have the file
- Users might temporarily see "file not found" from other regions
```

### **Q2: When would you choose strong vs eventual consistency?**
```
Strong Consistency:
✅ Banking transactions
✅ Inventory management
✅ Authentication systems
❌ High-scale reads
❌ Global distribution

Eventual Consistency:
✅ Social media feeds
✅ Product catalogs
✅ User profiles
✅ Commenting systems
❌ Financial operations
❌ Real-time collaboration
```

### **Q3: How do you handle consistency in microservices?**
```
A: Patterns:
1. Event Sourcing: Store events, replay for consistency
2. Saga Pattern: Coordinated transactions across services
3. CQRS: Separate read/write models, eventual consistency
4. Two-Phase Commit: Strong consistency (expensive)
5. Compensating Transactions: Reverse operations on failure
```

## 📚 Related Topics

- [[CAP Theorem]]
- [[Database Replication]]
- [[Distributed Transactions]]
- [[Vector Clocks]]
- [[Conflict Resolution]]
- [[CRDT]]
- [[Event Sourcing]]

---

*Consistency is about trade-offs. Choose the weakest consistency model that still meets your application requirements for better performance and availability.* 