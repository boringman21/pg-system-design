# Data Consistency Models

**Tags**: #consistency #distributed-systems #database #cap-theorem
**Ngày tạo**: 2024-01-01
**Độ khó**: Intermediate
**Thời gian đọc**: 12-15 phút

## 🎯 Consistency Models Overview

Data consistency models định nghĩa rules về how và when data updates become visible across distributed systems.

### **Core Concepts**
- **Consistency**: All nodes see same data simultaneously  
- **Availability**: System remains operational
- **Partition Tolerance**: System continues despite network failures
- **Trade-offs**: CAP theorem constraints

---

## 🏗️ Consistency Models Spectrum

### **1. Strong Consistency**
```python
# Linearizable consistency - strongest model
class StrongConsistencyExample:
    def __init__(self):
        self.data = {}
        self.global_clock = 0
        self.lock = threading.Lock()
    
    def write(self, key, value):
        with self.lock:
            self.global_clock += 1
            # All reads after this point see new value
            self.data[key] = {
                'value': value,
                'timestamp': self.global_clock
            }
            # Synchronously replicate to all nodes
            self.replicate_to_all_nodes(key, value, self.global_clock)
    
    def read(self, key):
        with self.lock:
            # Always returns most recent write
            return self.data.get(key, {}).get('value')
```

**Characteristics:**
- ✅ Easy to reason about
- ✅ No data conflicts  
- ❌ High latency
- ❌ Lower availability

### **2. Eventual Consistency**
```python
# Amazon DynamoDB style eventual consistency
class EventualConsistencyExample:
    def __init__(self, node_id):
        self.node_id = node_id
        self.local_data = {}
        self.version_vector = {}
        self.pending_updates = []
    
    def write(self, key, value):
        # Write locally first
        self.version_vector[self.node_id] = self.version_vector.get(self.node_id, 0) + 1
        
        self.local_data[key] = {
            'value': value,
            'version': self.version_vector.copy(),
            'timestamp': time.time()
        }
        
        # Asynchronously propagate to other nodes
        self.async_replicate(key, value, self.version_vector.copy())
        return True  # Immediate response
    
    def read(self, key):
        # May return stale data
        return self.local_data.get(key, {}).get('value')
    
    def sync_with_peers(self, peer_updates):
        # Resolve conflicts using vector clocks
        for update in peer_updates:
            self.resolve_conflict(update)
```

**Characteristics:**
- ✅ High availability
- ✅ Low latency writes
- ❌ Temporary inconsistencies
- ❌ Complex conflict resolution

### **3. Causal Consistency**
```python
# Maintains causal relationships between operations
class CausalConsistencyExample:
    def __init__(self, node_id):
        self.node_id = node_id
        self.data = {}
        self.causal_history = []
        self.vector_clock = {}
    
    def write_with_causality(self, key, value, depends_on=None):
        # Update vector clock
        self.vector_clock[self.node_id] = self.vector_clock.get(self.node_id, 0) + 1
        
        operation = {
            'type': 'write',
            'key': key,
            'value': value,
            'clock': self.vector_clock.copy(),
            'depends_on': depends_on,
            'timestamp': time.time()
        }
        
        # Ensure causal dependencies are satisfied
        if depends_on:
            self.wait_for_dependencies(depends_on)
        
        self.data[key] = value
        self.causal_history.append(operation)
        
        return operation['clock']
    
    def wait_for_dependencies(self, dependencies):
        # Wait until all causal dependencies are satisfied
        for dep in dependencies:
            while not self.dependency_satisfied(dep):
                time.sleep(0.001)
```

---

## 📊 Consistency Models Comparison

| Model | Consistency | Availability | Performance | Use Cases |
|-------|-------------|--------------|-------------|-----------|
| **Strong** | Highest | Lowest | Slowest | Banking, Financial |
| **Sequential** | High | Medium | Medium | General RDBMS |
| **Causal** | Medium | High | Good | Social media, Chat |
| **Eventual** | Lowest | Highest | Fastest | DNS, CDN, Analytics |

---

## 🔄 Implementation Patterns

### **Read Quorum Strategy**
```python
class QuorumConsistency:
    def __init__(self, total_nodes=5):
        self.total_nodes = total_nodes
        self.write_quorum = (total_nodes // 2) + 1  # W
        self.read_quorum = (total_nodes // 2) + 1   # R
        # W + R > N ensures strong consistency
    
    def write(self, key, value):
        successful_writes = 0
        
        for node in self.get_available_nodes():
            try:
                node.write(key, value)
                successful_writes += 1
                
                if successful_writes >= self.write_quorum:
                    return True  # Success
                    
            except Exception as e:
                self.log_write_failure(node, e)
        
        return False  # Failed to achieve quorum
    
    def read(self, key):
        responses = []
        
        for node in self.get_available_nodes():
            try:
                response = node.read(key)
                responses.append(response)
                
                if len(responses) >= self.read_quorum:
                    # Return most recent version
                    return self.resolve_read_conflicts(responses)
                    
            except Exception as e:
                self.log_read_failure(node, e)
        
        raise Exception("Failed to achieve read quorum")
```

### **Vector Clocks for Conflict Detection**
```python
class VectorClock:
    def __init__(self, node_id):
        self.node_id = node_id
        self.clock = {}
    
    def increment(self):
        self.clock[self.node_id] = self.clock.get(self.node_id, 0) + 1
        return self.clock.copy()
    
    def update(self, other_clock):
        for node, timestamp in other_clock.items():
            self.clock[node] = max(self.clock.get(node, 0), timestamp)
        self.increment()
    
    def compare(self, other_clock):
        # Returns: 'before', 'after', or 'concurrent'
        self_less = all(
            self.clock.get(node, 0) <= other_clock.get(node, 0)
            for node in set(self.clock.keys()) | set(other_clock.keys())
        )
        
        other_less = all(
            other_clock.get(node, 0) <= self.clock.get(node, 0)
            for node in set(self.clock.keys()) | set(other_clock.keys())
        )
        
        if self_less and not other_less:
            return 'before'
        elif other_less and not self_less:
            return 'after'
        else:
            return 'concurrent'
```

---

## ⚠️ Consistency Challenges

### **Split-Brain Problem**
```python
class SplitBrainPrevention:
    def __init__(self, total_nodes):
        self.total_nodes = total_nodes
        self.majority = (total_nodes // 2) + 1
    
    def can_accept_writes(self, active_nodes):
        # Only accept writes if we have majority
        return len(active_nodes) >= self.majority
    
    def handle_network_partition(self, partition_nodes):
        if len(partition_nodes) >= self.majority:
            # This partition can continue operating
            return True
        else:
            # Enter read-only mode
            self.enter_read_only_mode()
            return False
```

### **Conflict Resolution Strategies**
```python
class ConflictResolver:
    def last_write_wins(self, conflicts):
        # Simple but may lose data
        return max(conflicts, key=lambda x: x['timestamp'])
    
    def multi_value_resolution(self, conflicts):
        # Return all conflicting values, let application decide
        return [conflict['value'] for conflict in conflicts]
    
    def application_specific_merge(self, conflicts):
        # Business logic determines resolution
        if all(isinstance(c['value'], int) for c in conflicts):
            # Sum for counters
            return sum(c['value'] for c in conflicts)
        else:
            # Concatenate for strings
            return ' | '.join(c['value'] for c in conflicts)
```

---

## 🎯 Choosing Consistency Model

### **Decision Framework**
```python
def choose_consistency_model(requirements):
    if requirements.get('financial_transactions'):
        return 'strong'
    
    if requirements.get('real_time_collaboration'):
        return 'causal'
    
    if requirements.get('high_availability') and requirements.get('global_scale'):
        return 'eventual'
    
    if requirements.get('simple_application'):
        return 'sequential'
    
    return 'eventual'  # Default for web scale
```

### **Use Case Examples**

| System | Consistency Model | Reasoning |
|--------|------------------|-----------|
| **Bank Account** | Strong | Money transfers must be atomic |
| **Social Media Feed** | Eventual | Likes/comments can be delayed |
| **Chat Application** | Causal | Message ordering matters |
| **DNS System** | Eventual | Global scale, eventual propagation OK |
| **Gaming Leaderboard** | Strong | Competitive integrity required |

---

## 🚀 Best Practices

### **1. Design for Consistency Requirements**
```python
class ConsistencyAwareDesign:
    def __init__(self):
        self.strong_consistency_data = {}  # Critical data
        self.eventual_consistency_data = {}  # Non-critical data
    
    def update_critical_data(self, key, value):
        # Use strong consistency for critical operations
        return self.strong_consistency_store.write(key, value)
    
    def update_analytics_data(self, key, value):
        # Use eventual consistency for analytics
        return self.eventual_consistency_store.write(key, value)
```

### **2. Monitor Consistency Lag**
```python
class ConsistencyMonitor:
    def measure_replication_lag(self, write_timestamp, read_timestamp):
        lag = read_timestamp - write_timestamp
        
        if lag > 5:  # 5 seconds threshold
            self.alert_high_lag(lag)
        
        self.metrics.record('replication_lag', lag)
```

### **3. Graceful Degradation**
```python
class GracefulConsistency:
    def read_with_fallback(self, key):
        try:
            # Try strong consistency first
            return self.strong_read(key)
        except TimeoutException:
            # Fallback to eventual consistency
            return self.eventual_read(key)
```

---

**🎯 Choose consistency model based on business requirements, not technical preferences. Different parts of system can use different models!**

## 📚 Related Topics
- [[01-Fundamentals/CAP-Theorem/CAP-Theorem|CAP Theorem]]
- [[06-Database-Design/Replication/Database-Replication|Database Replication]]
- [[01-Fundamentals/Distributed-Systems/Distributed-Systems-Fundamentals|Distributed Systems]] 