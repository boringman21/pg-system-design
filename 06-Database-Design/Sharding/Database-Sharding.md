# Database Sharding

**Tags**: #database #sharding #scalability #horizontal-scaling
**Date**: 2024-01-01

## 📝 Overview

Database sharding là kỹ thuật chia nhỏ large database thành multiple smaller databases (shards) để improve performance và scalability. Mỗi shard chứa subset of data và có thể deployed trên different servers.

## 🎯 Sharding Fundamentals

### **Why Sharding?**
```
Traditional single database limitations:
- Storage capacity limits
- CPU/Memory bottlenecks  
- Network I/O constraints
- Backup/Recovery time increases

Sharding benefits:
- Horizontal scalability
- Improved query performance
- Better resource utilization
- Fault isolation
```

### **Sharding vs Other Scaling Techniques**
```python
# Comparison matrix
scaling_techniques = {
    "Read Replicas": {
        "pros": ["Easy to implement", "Improves read performance"],
        "cons": ["No write scaling", "Data consistency challenges"],
        "use_case": "Read-heavy workloads"
    },
    "Vertical Scaling": {
        "pros": ["Simple", "No application changes"],
        "cons": ["Hardware limits", "Single point of failure"],
        "use_case": "Small to medium applications"
    },
    "Sharding": {
        "pros": ["True horizontal scaling", "Linear performance improvement"],
        "cons": ["Complex implementation", "Cross-shard queries difficult"],
        "use_case": "Large-scale distributed applications"
    }
}
```

## 🔄 Sharding Strategies

### **1. Range-Based Sharding**
Data distributed dựa trên range of shard key values

```python
class RangeSharding:
    def __init__(self):
        self.shard_ranges = [
            {"shard_id": 1, "min": 0, "max": 25000},
            {"shard_id": 2, "min": 25001, "max": 50000},
            {"shard_id": 3, "min": 50001, "max": 75000},
            {"shard_id": 4, "min": 75001, "max": 100000}
        ]
    
    def get_shard_id(self, user_id):
        for range_config in self.shard_ranges:
            if range_config["min"] <= user_id <= range_config["max"]:
                return range_config["shard_id"]
        raise ValueError("User ID out of range")
    
    def rebalance_shards(self, new_shard_count):
        # Redistribute ranges when adding new shards
        total_range = 100000
        range_per_shard = total_range // new_shard_count
        
        new_ranges = []
        for i in range(new_shard_count):
            min_val = i * range_per_shard
            max_val = (i + 1) * range_per_shard - 1
            if i == new_shard_count - 1:  # Last shard gets remainder
                max_val = total_range
            
            new_ranges.append({
                "shard_id": i + 1,
                "min": min_val,
                "max": max_val
            })
        
        self.shard_ranges = new_ranges

# Example usage for e-commerce orders
class OrderSharding(RangeSharding):
    def __init__(self):
        # Shard by order date for temporal queries
        self.shard_ranges = [
            {"shard_id": 1, "start_date": "2023-01-01", "end_date": "2023-03-31"},
            {"shard_id": 2, "start_date": "2023-04-01", "end_date": "2023-06-30"},
            {"shard_id": 3, "start_date": "2023-07-01", "end_date": "2023-09-30"},
            {"shard_id": 4, "start_date": "2023-10-01", "end_date": "2023-12-31"}
        ]
    
    def get_shard_for_order(self, order_date):
        for range_config in self.shard_ranges:
            if range_config["start_date"] <= order_date <= range_config["end_date"]:
                return range_config["shard_id"]
        return self.create_new_shard(order_date)
```

**Pros**: Range queries efficient, easy to understand
**Cons**: Uneven distribution, hotspots possible

### **2. Hash-Based Sharding**
Use hash function để evenly distribute data

```python
import hashlib
import consistent_hashing

class HashSharding:
    def __init__(self, shard_count=4):
        self.shard_count = shard_count
    
    def get_shard_id(self, key):
        # Simple modulo hashing
        hash_value = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
        return (hash_value % self.shard_count) + 1
    
    def add_shard(self):
        # Problem: Adding shard requires massive data movement
        old_shard_count = self.shard_count
        self.shard_count += 1
        
        # Calculate data movement needed
        affected_keys = []
        for key in range(100000):  # Example key range
            old_shard = (hash(key) % old_shard_count) + 1
            new_shard = (hash(key) % self.shard_count) + 1
            if old_shard != new_shard:
                affected_keys.append(key)
        
        print(f"Need to move {len(affected_keys)} records")
        return affected_keys

class ConsistentHashSharding:
    """Improved hash sharding with consistent hashing"""
    
    def __init__(self, initial_shards):
        self.ring = ConsistentHashRing()
        for shard_id in initial_shards:
            self.add_shard(shard_id)
    
    def add_shard(self, shard_id):
        # Add multiple virtual nodes for better distribution
        for i in range(100):  # 100 virtual nodes per shard
            virtual_node = f"{shard_id}:virtual:{i}"
            self.ring.add_node(virtual_node, shard_id)
    
    def remove_shard(self, shard_id):
        self.ring.remove_nodes_for_shard(shard_id)
    
    def get_shard_id(self, key):
        return self.ring.get_node(key)
    
    def rebalance_data(self, old_shard, new_shard, key_range):
        """Move data between shards during rebalancing"""
        return {
            "source_shard": old_shard,
            "target_shard": new_shard,
            "keys_to_move": key_range,
            "estimated_size": len(key_range) * 1024  # Estimated bytes
        }

# Real-world example: User data sharding
class UserDataSharding:
    def __init__(self):
        self.shards = {
            1: "user_db_shard_1",
            2: "user_db_shard_2", 
            3: "user_db_shard_3",
            4: "user_db_shard_4"
        }
    
    def get_user_shard(self, user_id):
        # Hash based on user ID
        shard_id = (hash(str(user_id)) % len(self.shards)) + 1
        return self.shards[shard_id]
    
    def store_user(self, user_data):
        shard = self.get_user_shard(user_data["user_id"])
        return self.execute_on_shard(shard, "INSERT", user_data)
    
    def get_user(self, user_id):
        shard = self.get_user_shard(user_id)
        return self.execute_on_shard(shard, "SELECT", {"user_id": user_id})
```

**Pros**: Even distribution, no hotspots
**Cons**: Range queries difficult, rebalancing expensive

### **3. Directory-Based Sharding**
Lookup service tracks which shard contains which data

```python
class DirectorySharding:
    def __init__(self):
        self.directory = {}  # key -> shard_id mapping
        self.shard_metadata = {
            1: {"host": "db1.example.com", "capacity": 50000},
            2: {"host": "db2.example.com", "capacity": 50000},
            3: {"host": "db3.example.com", "capacity": 50000}
        }
    
    def register_key(self, key, shard_id):
        self.directory[key] = shard_id
    
    def get_shard_id(self, key):
        return self.directory.get(key)
    
    def migrate_key(self, key, new_shard_id):
        old_shard_id = self.directory.get(key)
        if old_shard_id:
            # Move data from old shard to new shard
            data = self.fetch_from_shard(old_shard_id, key)
            self.store_to_shard(new_shard_id, key, data)
            self.remove_from_shard(old_shard_id, key)
            
            # Update directory
            self.directory[key] = new_shard_id
    
    def auto_balance(self):
        """Automatically balance load across shards"""
        shard_loads = self.calculate_shard_loads()
        
        # Find overloaded and underloaded shards
        avg_load = sum(shard_loads.values()) / len(shard_loads)
        
        for shard_id, load in shard_loads.items():
            if load > avg_load * 1.2:  # 20% above average
                # Move some keys to less loaded shards
                keys_to_move = self.select_keys_to_move(shard_id, load - avg_load)
                target_shard = min(shard_loads.items(), key=lambda x: x[1])[0]
                
                for key in keys_to_move:
                    self.migrate_key(key, target_shard)

# Database router with directory sharding
class DatabaseRouter:
    def __init__(self, directory_service):
        self.directory = directory_service
        self.connections = {}  # shard_id -> database_connection
    
    def execute_query(self, key, query, params=None):
        shard_id = self.directory.get_shard_id(key)
        if not shard_id:
            raise ValueError(f"No shard found for key: {key}")
        
        connection = self.get_connection(shard_id)
        return connection.execute(query, params)
    
    def execute_cross_shard_query(self, query, params=None):
        """Execute query across all shards and aggregate results"""
        results = []
        
        for shard_id in self.directory.shard_metadata.keys():
            connection = self.get_connection(shard_id)
            shard_result = connection.execute(query, params)
            results.extend(shard_result)
        
        return self.aggregate_results(results)
```

**Pros**: Flexible, easy rebalancing
**Cons**: Directory is single point of failure, extra lookup overhead

## 🛠️ Implementation Challenges

### **Cross-Shard Queries**
```python
class CrossShardQueryHandler:
    def __init__(self, shard_router):
        self.router = shard_router
    
    def execute_join_query(self, user_id, order_filters):
        """Join user data with orders across shards"""
        
        # Get user data from user shard
        user_shard = self.router.get_user_shard(user_id)
        user_data = self.router.query_shard(user_shard, 
            "SELECT * FROM users WHERE id = ?", [user_id])
        
        # Query all order shards (since orders could be anywhere)
        order_results = []
        for shard_id in self.router.get_order_shards():
            orders = self.router.query_shard(shard_id,
                "SELECT * FROM orders WHERE user_id = ? AND status = ?",
                [user_id, order_filters["status"]])
            order_results.extend(orders)
        
        # Combine results in application layer
        return {
            "user": user_data[0] if user_data else None,
            "orders": order_results
        }
    
    def execute_aggregation_query(self, start_date, end_date):
        """Aggregate data across all shards"""
        
        total_revenue = 0
        order_count = 0
        
        # Query each shard
        for shard_id in self.router.get_order_shards():
            result = self.router.query_shard(shard_id, """
                SELECT COUNT(*) as orders, SUM(total) as revenue 
                FROM orders 
                WHERE created_at BETWEEN ? AND ?
            """, [start_date, end_date])
            
            if result:
                order_count += result[0]["orders"]
                total_revenue += result[0]["revenue"] or 0
        
        return {
            "total_orders": order_count,
            "total_revenue": total_revenue,
            "average_order_value": total_revenue / order_count if order_count > 0 else 0
        }
    
    def execute_distributed_transaction(self, user_id, order_data):
        """Two-phase commit across shards"""
        
        user_shard = self.router.get_user_shard(user_id)
        order_shard = self.router.get_order_shard(order_data["order_id"])
        
        transaction_id = self.generate_transaction_id()
        
        try:
            # Phase 1: Prepare
            self.router.prepare_transaction(user_shard, transaction_id, {
                "action": "update_user_balance",
                "user_id": user_id,
                "amount": -order_data["total"]
            })
            
            self.router.prepare_transaction(order_shard, transaction_id, {
                "action": "create_order",
                "order_data": order_data
            })
            
            # Phase 2: Commit
            self.router.commit_transaction(user_shard, transaction_id)
            self.router.commit_transaction(order_shard, transaction_id)
            
            return {"status": "success", "transaction_id": transaction_id}
            
        except Exception as e:
            # Rollback all shards
            self.router.rollback_transaction(user_shard, transaction_id)
            self.router.rollback_transaction(order_shard, transaction_id)
            raise
```

### **Shard Management**
```python
class ShardManager:
    def __init__(self):
        self.shards = {}
        self.rebalancing_lock = threading.Lock()
    
    def add_new_shard(self, shard_config):
        """Add new shard to cluster"""
        
        with self.rebalancing_lock:
            new_shard_id = len(self.shards) + 1
            
            # Create new shard
            self.shards[new_shard_id] = {
                "host": shard_config["host"],
                "status": "initializing",
                "capacity": shard_config["capacity"]
            }
            
            # Rebalance existing data
            self.rebalance_data_to_new_shard(new_shard_id)
            
            # Update routing configuration
            self.update_routing_config()
            
            self.shards[new_shard_id]["status"] = "active"
    
    def rebalance_data_to_new_shard(self, new_shard_id):
        """Move data from existing shards to new shard"""
        
        # Calculate how much data to move from each shard
        total_shards = len(self.shards)
        data_per_shard = 1.0 / total_shards
        
        for shard_id in self.shards:
            if shard_id == new_shard_id:
                continue
                
            # Move portion of data to new shard
            keys_to_move = self.select_keys_for_migration(
                shard_id, data_per_shard
            )
            
            self.migrate_keys(shard_id, new_shard_id, keys_to_move)
    
    def remove_shard(self, shard_id):
        """Remove shard and redistribute its data"""
        
        with self.rebalancing_lock:
            if shard_id not in self.shards:
                raise ValueError(f"Shard {shard_id} not found")
            
            # Mark shard as draining
            self.shards[shard_id]["status"] = "draining"
            
            # Move all data to other shards
            remaining_shards = [s for s in self.shards.keys() if s != shard_id]
            data_on_shard = self.get_all_keys_on_shard(shard_id)
            
            # Distribute data evenly among remaining shards
            keys_per_shard = len(data_on_shard) // len(remaining_shards)
            
            for i, target_shard in enumerate(remaining_shards):
                start_idx = i * keys_per_shard
                end_idx = start_idx + keys_per_shard
                if i == len(remaining_shards) - 1:  # Last shard gets remainder
                    end_idx = len(data_on_shard)
                
                keys_to_move = data_on_shard[start_idx:end_idx]
                self.migrate_keys(shard_id, target_shard, keys_to_move)
            
            # Remove shard
            del self.shards[shard_id]
```

## 📊 Monitoring & Maintenance

### **Shard Health Monitoring**
```python
class ShardMonitor:
    def __init__(self, shard_manager):
        self.shard_manager = shard_manager
        self.metrics = {}
    
    def collect_metrics(self):
        """Collect performance metrics from all shards"""
        
        for shard_id, shard_config in self.shard_manager.shards.items():
            try:
                connection = self.get_connection(shard_id)
                
                # Collect key metrics
                metrics = {
                    "cpu_usage": self.get_cpu_usage(shard_id),
                    "memory_usage": self.get_memory_usage(shard_id),
                    "disk_usage": self.get_disk_usage(shard_id),
                    "query_latency": self.get_avg_query_latency(shard_id),
                    "throughput": self.get_queries_per_second(shard_id),
                    "data_size": self.get_data_size(shard_id),
                    "connection_count": self.get_active_connections(shard_id)
                }
                
                self.metrics[shard_id] = metrics
                
                # Check for alerts
                self.check_alerts(shard_id, metrics)
                
            except Exception as e:
                logger.error(f"Failed to collect metrics for shard {shard_id}: {e}")
                self.metrics[shard_id] = {"status": "error", "error": str(e)}
    
    def check_alerts(self, shard_id, metrics):
        """Check if any metrics exceed thresholds"""
        
        alerts = []
        
        if metrics["cpu_usage"] > 80:
            alerts.append(f"High CPU usage: {metrics['cpu_usage']}%")
        
        if metrics["memory_usage"] > 85:
            alerts.append(f"High memory usage: {metrics['memory_usage']}%")
        
        if metrics["query_latency"] > 1000:  # ms
            alerts.append(f"High query latency: {metrics['query_latency']}ms")
        
        if metrics["disk_usage"] > 90:
            alerts.append(f"High disk usage: {metrics['disk_usage']}%")
        
        if alerts:
            self.send_alerts(shard_id, alerts)
    
    def recommend_actions(self):
        """Analyze metrics and recommend actions"""
        
        recommendations = []
        
        # Check for load imbalance
        cpu_usages = [m.get("cpu_usage", 0) for m in self.metrics.values()]
        if max(cpu_usages) - min(cpu_usages) > 30:
            recommendations.append("Consider rebalancing data across shards")
        
        # Check for scaling needs
        avg_cpu = sum(cpu_usages) / len(cpu_usages)
        if avg_cpu > 70:
            recommendations.append("Consider adding new shards")
        
        # Check for storage capacity
        disk_usages = [m.get("disk_usage", 0) for m in self.metrics.values()]
        if max(disk_usages) > 85:
            recommendations.append("Consider increasing storage capacity")
        
        return recommendations
```

## 🔗 Liên kết và Tham khảo

### **Related Topics**
- [[06-Database-Design/Replication/Database-Replication|Database Replication]] - Combining replication with sharding
- [[Consistent Hashing]] - Advanced sharding algorithms
- [[07-Microservices/Microservices-Architecture|Microservices Architecture]] - Per-service data sharding
- [[06-Database-Design/NoSQL-Databases/NoSQL-Database-Types|NoSQL Database Types]] - Sharding in different database systems

### **Sharding Tools**
- **MySQL**: MySQL Cluster, Vitess, ProxySQL
- **PostgreSQL**: Citus, PostgreSQL-XL, pg_shard
- **MongoDB**: Native sharding support
- **Cassandra**: Automatic data distribution

### **Implementation Considerations**
- Choose appropriate shard key
- Plan for hotspot prevention
- Implement cross-shard transactions carefully
- Monitor shard distribution và performance
- Plan for shard rebalancing

---

*Sharding is powerful scaling technique nhưng introduces complexity. Carefully design shard key và implement proper monitoring để avoid hotspots và ensure balanced load distribution.*

## 📚 Further Reading

- MySQL sharding best practices
- MongoDB sharding architecture
- Cassandra ring topology
- Vitess: YouTube's database sharding solution

---

**Key Takeaway**: Sharding enables horizontal database scaling but introduces complexity trong querying và transactions. Choose sharding strategy based on access patterns: range cho temporal data, hash cho even distribution, directory cho flexibility. 