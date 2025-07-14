# NoSQL Database Types

**Tags**: #database #nosql #scalability #data-modeling
**Date**: 2024-01-01

## 📝 Overview

NoSQL databases provide flexible, scalable alternatives to traditional relational databases. Chúng được thiết kế để handle large volumes of unstructured data và scale horizontally across commodity hardware.

## 🎯 NoSQL Categories

### **1. Document Databases**
Lưu trữ data như documents (JSON, BSON, XML)

```python
# MongoDB example
from pymongo import MongoClient

class DocumentStore:
    def __init__(self):
        self.client = MongoClient('mongodb://localhost:27017/')
        self.db = self.client.ecommerce
        self.products = self.db.products
    
    def create_product(self, product_data):
        # Flexible schema - no predefined structure
        product = {
            "name": product_data["name"],
            "price": product_data["price"],
            "category": product_data["category"],
            "specifications": product_data.get("specs", {}),  # Nested object
            "tags": product_data.get("tags", []),  # Array
            "reviews": [],
            "created_at": datetime.utcnow()
        }
        
        result = self.products.insert_one(product)
        return result.inserted_id
    
    def find_products_by_category(self, category):
        return list(self.products.find({"category": category}))
    
    def add_review(self, product_id, review):
        # Update nested array
        self.products.update_one(
            {"_id": ObjectId(product_id)},
            {"$push": {"reviews": review}}
        )
    
    def search_products(self, query):
        # Text search across multiple fields
        return list(self.products.find({
            "$text": {"$search": query}
        }))

# Complex queries with aggregation
def get_product_analytics(self):
    pipeline = [
        {"$match": {"category": "electronics"}},
        {"$unwind": "$reviews"},
        {"$group": {
            "_id": "$_id",
            "name": {"$first": "$name"},
            "avg_rating": {"$avg": "$reviews.rating"},
            "review_count": {"$sum": 1}
        }},
        {"$sort": {"avg_rating": -1}},
        {"$limit": 10}
    ]
    return list(self.products.aggregate(pipeline))
```

**Use Cases**: Content management, product catalogs, user profiles, configuration data

### **2. Key-Value Stores**
Simplest NoSQL model - key mapped to value

```python
# Redis example
import redis
import json

class KeyValueCache:
    def __init__(self):
        self.redis_client = redis.Redis(
            host='localhost', 
            port=6379, 
            decode_responses=True
        )
    
    def set_user_session(self, session_id, user_data, ttl=3600):
        # Store complex objects as JSON
        self.redis_client.setex(
            f"session:{session_id}",
            ttl,
            json.dumps(user_data)
        )
    
    def get_user_session(self, session_id):
        data = self.redis_client.get(f"session:{session_id}")
        return json.loads(data) if data else None
    
    def increment_page_views(self, page_url):
        # Atomic increment
        return self.redis_client.incr(f"views:{page_url}")
    
    def set_rate_limit(self, user_id, limit, window_seconds):
        key = f"rate_limit:{user_id}"
        current = self.redis_client.get(key) or 0
        
        if int(current) >= limit:
            return False  # Rate limit exceeded
        
        pipe = self.redis_client.pipeline()
        pipe.incr(key)
        pipe.expire(key, window_seconds)
        pipe.execute()
        return True
    
    def cache_computation_result(self, cache_key, computation_fn, ttl=300):
        # Cache expensive computations
        cached = self.redis_client.get(cache_key)
        if cached:
            return json.loads(cached)
        
        result = computation_fn()
        self.redis_client.setex(cache_key, ttl, json.dumps(result))
        return result

# Advanced Redis patterns
class DistributedQueue:
    def __init__(self, redis_client, queue_name):
        self.redis = redis_client
        self.queue_name = queue_name
    
    def push_task(self, task_data):
        self.redis.lpush(self.queue_name, json.dumps(task_data))
    
    def pop_task(self, timeout=0):
        # Blocking pop - wait for task
        result = self.redis.brpop(self.queue_name, timeout=timeout)
        if result:
            return json.loads(result[1])
        return None
```

**Use Cases**: Caching, session management, real-time recommendations, leaderboards

### **3. Column-Family (Wide-Column)**
Data stored in column families với flexible columns per row

```python
# Cassandra example
from cassandra.cluster import Cluster
from cassandra.policies import DCAwareRoundRobinPolicy

class TimeSeriesStore:
    def __init__(self):
        # Multi-datacenter setup
        self.cluster = Cluster(
            ['127.0.0.1'],
            load_balancing_policy=DCAwareRoundRobinPolicy()
        )
        self.session = self.cluster.connect()
        
        # Create keyspace with replication
        self.session.execute("""
            CREATE KEYSPACE IF NOT EXISTS analytics 
            WITH REPLICATION = {
                'class': 'SimpleStrategy',
                'replication_factor': 3
            }
        """)
        self.session.set_keyspace('analytics')
        
        # Time-series optimized table
        self.session.execute("""
            CREATE TABLE IF NOT EXISTS user_events (
                user_id UUID,
                event_date DATE,
                event_time TIMESTAMP,
                event_type TEXT,
                event_data MAP<TEXT, TEXT>,
                PRIMARY KEY ((user_id, event_date), event_time)
            ) WITH CLUSTERING ORDER BY (event_time DESC)
        """)
    
    def record_event(self, user_id, event_type, event_data):
        from datetime import datetime
        
        now = datetime.utcnow()
        self.session.execute("""
            INSERT INTO user_events 
            (user_id, event_date, event_time, event_type, event_data)
            VALUES (?, ?, ?, ?, ?)
        """, (user_id, now.date(), now, event_type, event_data))
    
    def get_user_timeline(self, user_id, start_date, end_date):
        # Efficient range query
        return self.session.execute("""
            SELECT * FROM user_events 
            WHERE user_id = ? AND event_date >= ? AND event_date <= ?
            ORDER BY event_time DESC
        """, (user_id, start_date, end_date))
    
    def get_recent_events(self, user_id, limit=100):
        # Latest events across partitions
        today = datetime.utcnow().date()
        return self.session.execute("""
            SELECT * FROM user_events 
            WHERE user_id = ? AND event_date = ?
            ORDER BY event_time DESC LIMIT ?
        """, (user_id, today, limit))

# Wide-column data modeling
class SocialMediaFeed:
    def __init__(self, session):
        self.session = session
        
        # Fan-out on write pattern
        self.session.execute("""
            CREATE TABLE IF NOT EXISTS user_timeline (
                user_id UUID,
                post_time TIMESTAMP,
                post_id UUID,
                author_id UUID,
                content TEXT,
                media_urls SET<TEXT>,
                PRIMARY KEY (user_id, post_time, post_id)
            ) WITH CLUSTERING ORDER BY (post_time DESC)
        """)
    
    def add_post_to_followers(self, post_data, follower_ids):
        # Denormalized approach for read performance
        batch = BatchStatement()
        
        for follower_id in follower_ids:
            batch.add(SimpleStatement("""
                INSERT INTO user_timeline 
                (user_id, post_time, post_id, author_id, content, media_urls)
                VALUES (?, ?, ?, ?, ?, ?)
            """), (
                follower_id, post_data['timestamp'], 
                post_data['post_id'], post_data['author_id'],
                post_data['content'], post_data['media_urls']
            ))
        
        self.session.execute(batch)
```

**Use Cases**: Time-series data, IoT sensors, activity feeds, content management

### **4. Graph Databases**
Optimized cho relationships và graph traversals

```python
# Neo4j example
from neo4j import GraphDatabase

class SocialNetworkGraph:
    def __init__(self, uri, username, password):
        self.driver = GraphDatabase.driver(uri, auth=(username, password))
    
    def close(self):
        self.driver.close()
    
    def create_user(self, user_id, properties):
        with self.driver.session() as session:
            session.run("""
                CREATE (u:User {id: $user_id, name: $name, email: $email})
            """, user_id=user_id, **properties)
    
    def create_friendship(self, user1_id, user2_id, relationship_data=None):
        with self.driver.session() as session:
            session.run("""
                MATCH (u1:User {id: $user1_id}), (u2:User {id: $user2_id})
                CREATE (u1)-[:FRIENDS_WITH {since: $since}]->(u2)
                CREATE (u2)-[:FRIENDS_WITH {since: $since}]->(u1)
            """, user1_id=user1_id, user2_id=user2_id, 
                 since=relationship_data.get('since'))
    
    def find_mutual_friends(self, user1_id, user2_id):
        with self.driver.session() as session:
            result = session.run("""
                MATCH (u1:User {id: $user1_id})-[:FRIENDS_WITH]-(mutual)
                      -[:FRIENDS_WITH]-(u2:User {id: $user2_id})
                RETURN mutual.name, mutual.id
            """, user1_id=user1_id, user2_id=user2_id)
            return [{"name": record["mutual.name"], 
                    "id": record["mutual.id"]} for record in result]
    
    def recommend_friends(self, user_id, limit=10):
        # Friends of friends recommendation
        with self.driver.session() as session:
            result = session.run("""
                MATCH (u:User {id: $user_id})-[:FRIENDS_WITH*2]-(suggested)
                WHERE NOT (u)-[:FRIENDS_WITH]-(suggested) AND u <> suggested
                WITH suggested, COUNT(*) as mutual_count
                ORDER BY mutual_count DESC
                LIMIT $limit
                RETURN suggested.name, suggested.id, mutual_count
            """, user_id=user_id, limit=limit)
            
            return [{
                "name": record["suggested.name"],
                "id": record["suggested.id"],
                "mutual_friends": record["mutual_count"]
            } for record in result]
    
    def shortest_path_between_users(self, user1_id, user2_id):
        with self.driver.session() as session:
            result = session.run("""
                MATCH path = shortestPath(
                    (u1:User {id: $user1_id})-[:FRIENDS_WITH*]-(u2:User {id: $user2_id})
                )
                RETURN [node in nodes(path) | node.name] as path_names,
                       length(path) as degrees_of_separation
            """, user1_id=user1_id, user2_id=user2_id)
            
            record = result.single()
            if record:
                return {
                    "path": record["path_names"],
                    "degrees": record["degrees_of_separation"]
                }
            return None
```

**Use Cases**: Social networks, recommendation engines, fraud detection, knowledge graphs

## 🔧 Data Modeling Strategies

### **Document Database Modeling**
```python
# Anti-pattern: Over-normalization
{
    "_id": "user123",
    "name": "John Doe",
    "address_id": "addr456"  # Reference to separate collection
}

# Better: Embed related data
{
    "_id": "user123",
    "name": "John Doe",
    "address": {
        "street": "123 Main St",
        "city": "New York",
        "postal_code": "10001"
    },
    "preferences": {
        "theme": "dark",
        "notifications": true
    }
}

# Handling one-to-many relationships
{
    "_id": "order123",
    "customer_id": "user123",
    "items": [
        {
            "product_id": "prod456",
            "name": "Laptop",
            "price": 999.99,
            "quantity": 1
        }
    ],
    "total": 999.99,
    "status": "shipped"
}
```

### **Key-Value Optimization**
```python
class OptimizedKeyValueStore:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def store_user_profile(self, user_id, profile_data):
        # Use hash for structured data within single key
        self.redis.hmset(f"user:{user_id}", {
            "name": profile_data["name"],
            "email": profile_data["email"],
            "last_login": profile_data["last_login"].isoformat()
        })
        
        # Set expiration
        self.redis.expire(f"user:{user_id}", 3600)
    
    def batch_operations(self, operations):
        # Use pipeline for multiple operations
        pipe = self.redis.pipeline()
        for op in operations:
            getattr(pipe, op["method"])(*op["args"])
        return pipe.execute()
    
    def implement_leaderboard(self, game_id):
        # Use sorted sets for rankings
        leaderboard_key = f"leaderboard:{game_id}"
        
        def add_score(user_id, score):
            self.redis.zadd(leaderboard_key, {user_id: score})
        
        def get_top_players(limit=10):
            return self.redis.zrevrange(
                leaderboard_key, 0, limit-1, withscores=True
            )
        
        def get_user_rank(user_id):
            rank = self.redis.zrevrank(leaderboard_key, user_id)
            return rank + 1 if rank is not None else None
        
        return {
            "add_score": add_score,
            "get_top_players": get_top_players,
            "get_user_rank": get_user_rank
        }
```

### **Column-Family Best Practices**
```python
# Partition key design for even distribution
class PartitioningStrategy:
    def design_partition_key(self, entity_type, natural_key):
        if entity_type == "time_series":
            # Time-based partitioning with bucketing
            return f"{natural_key}_{datetime.now().strftime('%Y%m%d')}"
        
        elif entity_type == "user_data":
            # Hash-based partitioning for even distribution
            import hashlib
            hash_value = hashlib.md5(natural_key.encode()).hexdigest()
            bucket = int(hash_value[:2], 16) % 100  # 100 buckets
            return f"bucket_{bucket:02d}_{natural_key}"
        
        return natural_key
    
    def design_clustering_key(self, access_pattern):
        # Design based on query patterns
        if access_pattern == "recent_first":
            return "timestamp DESC"
        elif access_pattern == "alphabetical":
            return "name ASC"
        elif access_pattern == "priority_first":
            return "priority DESC, timestamp DESC"
```

## ⚖️ Choosing the Right NoSQL Database

### **Decision Matrix**
```python
def choose_nosql_database(requirements):
    """
    Requirements assessment for NoSQL selection
    """
    
    if requirements.get("complex_relationships"):
        return "Graph Database (Neo4j, Amazon Neptune)"
    
    elif requirements.get("time_series_data"):
        return "Column-Family (Cassandra, InfluxDB)"
    
    elif requirements.get("flexible_schema") and requirements.get("complex_queries"):
        return "Document Database (MongoDB, CouchDB)"
    
    elif requirements.get("high_performance_cache") or requirements.get("simple_kv_ops"):
        return "Key-Value Store (Redis, DynamoDB)"
    
    elif requirements.get("horizontal_scaling") and requirements.get("write_heavy"):
        return "Column-Family (Cassandra, HBase)"
    
    else:
        return "Consider SQL database or multi-model approach"

# Example usage
requirements = {
    "complex_relationships": False,
    "time_series_data": True,
    "horizontal_scaling": True,
    "write_heavy": True,
    "eventual_consistency_ok": True
}

recommendation = choose_nosql_database(requirements)
# Output: "Column-Family (Cassandra, InfluxDB)"
```

### **Hybrid Approaches**
```python
class PolyglotPersistence:
    """Using multiple databases for different use cases"""
    
    def __init__(self):
        # Different databases for different needs
        self.user_cache = redis.Redis()  # Fast session data
        self.user_profiles = MongoClient()  # Flexible user data
        self.time_series = CassandraCluster()  # Analytics data
        self.social_graph = Neo4jDriver()  # Relationships
    
    def store_user_event(self, user_id, event_type, event_data):
        # Store in time-series DB for analytics
        self.time_series.record_event(user_id, event_type, event_data)
        
        # Cache recent activity
        self.user_cache.lpush(
            f"recent_activity:{user_id}", 
            json.dumps(event_data)
        )
        self.user_cache.ltrim(f"recent_activity:{user_id}", 0, 99)
        
        # Update user profile if needed
        if event_type == "profile_update":
            self.user_profiles.update_profile(user_id, event_data)
    
    def get_user_dashboard(self, user_id):
        # Aggregate data from multiple sources
        profile = self.user_profiles.get_profile(user_id)
        recent_activity = self.user_cache.lrange(f"recent_activity:{user_id}", 0, 10)
        friend_suggestions = self.social_graph.recommend_friends(user_id)
        
        return {
            "profile": profile,
            "recent_activity": [json.loads(a) for a in recent_activity],
            "friend_suggestions": friend_suggestions
        }
```

## 🔗 Related Topics

- [[SQL Database Design]] - Relational vs NoSQL comparison
- [[Database Sharding]] - Horizontal scaling strategies
- [[Caching Strategies]] - Performance optimization
- [[Consistency Models]] - Data consistency trade-offs

## 📚 Further Reading

- MongoDB schema design patterns
- Cassandra data modeling best practices
- Redis advanced patterns and use cases
- Neo4j graph algorithms and optimization

---

**Key Takeaway**: Choose NoSQL database type based on data structure, access patterns, và scalability requirements. Document databases cho flexible schemas, key-value cho high performance, column-family cho time-series, graph cho relationships. 