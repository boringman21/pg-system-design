# CQRS Pattern

**Tags**: #cqrs #design-patterns #architecture #event-sourcing
**Ngày tạo**: 2024-01-01

## 🎯 CQRS Overview

Command Query Responsibility Segregation (CQRS) separates read and write operations into different models, allowing each to be optimized independently.

### Core Concepts
- **Commands**: Change system state (Write operations)
- **Queries**: Read data without changes (Read operations)  
- **Separate Models**: Different optimization strategies for reads vs writes
- **Event Sourcing**: Often paired with CQRS for complete audit trail

## 🏗️ Detailed Implementation

### Basic CQRS Structure

```python
from abc import ABC, abstractmethod
from typing import List, Optional
from dataclasses import dataclass
from datetime import datetime

# Command Side - Write Model
@dataclass
class CreateUserCommand:
    email: str
    name: str
    age: int

@dataclass
class UpdateUserCommand:
    user_id: str
    name: Optional[str] = None
    age: Optional[int] = None

class UserCommandHandler:
    def __init__(self, repository, event_bus):
        self.repository = repository
        self.event_bus = event_bus
    
    def create_user(self, command: CreateUserCommand):
        # Validate business rules
        if self.repository.user_exists(command.email):
            raise ValueError("User already exists")
        
        # Create domain object
        user = User(
            id=generate_id(),
            email=command.email,
            name=command.name,
            age=command.age
        )
        
        # Save to write store
        self.repository.save(user)
        
        # Publish event for read models
        self.event_bus.publish('UserCreated', {
            'user_id': user.id,
            'email': user.email,
            'name': user.name,
            'age': user.age,
            'timestamp': datetime.utcnow()
        })
    
    def update_user(self, command: UpdateUserCommand):
        user = self.repository.get(command.user_id)
        if not user:
            raise ValueError("User not found")
        
        # Update domain object
        if command.name:
            user.name = command.name
        if command.age:
            user.age = command.age
        
        # Save changes
        self.repository.save(user)
        
        # Publish event
        self.event_bus.publish('UserUpdated', {
            'user_id': user.id,
            'name': user.name,
            'age': user.age,
            'timestamp': datetime.utcnow()
        })

# Query Side - Read Model
@dataclass
class UserView:
    id: str
    email: str
    name: str
    age: int
    created_at: datetime
    updated_at: datetime

@dataclass
class UserSummaryView:
    total_users: int
    average_age: float
    users_by_age_group: dict

class UserQueryHandler:
    def __init__(self, read_repository):
        self.read_repository = read_repository
    
    def get_user(self, user_id: str) -> Optional[UserView]:
        return self.read_repository.get_user_view(user_id)
    
    def get_users_by_age_range(self, min_age: int, max_age: int) -> List[UserView]:
        return self.read_repository.get_users_by_age_range(min_age, max_age)
    
    def get_user_summary(self) -> UserSummaryView:
        return self.read_repository.get_user_summary()
    
    def search_users(self, query: str) -> List[UserView]:
        return self.read_repository.search_users(query)
```

### Event Handlers for Read Models

```python
class UserReadModelHandler:
    def __init__(self, read_repository):
        self.read_repository = read_repository
    
    def handle_user_created(self, event):
        user_view = UserView(
            id=event['user_id'],
            email=event['email'],
            name=event['name'],
            age=event['age'],
            created_at=event['timestamp'],
            updated_at=event['timestamp']
        )
        self.read_repository.create_user_view(user_view)
        
        # Update summary statistics
        self.update_user_summary()
    
    def handle_user_updated(self, event):
        user_view = self.read_repository.get_user_view(event['user_id'])
        if user_view:
            user_view.name = event['name']
            user_view.age = event['age']
            user_view.updated_at = event['timestamp']
            self.read_repository.update_user_view(user_view)
            
            # Update summary statistics
            self.update_user_summary()
    
    def update_user_summary(self):
        # Recalculate summary statistics
        summary = self.read_repository.calculate_user_summary()
        self.read_repository.update_user_summary(summary)
```

### Database Schema Design

```sql
-- Write Model (Normalized)
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    age INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Read Model (Denormalized for queries)
CREATE TABLE user_views (
    id UUID PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    age INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    -- Denormalized fields for efficient queries
    age_group VARCHAR(50), -- 'teen', 'adult', 'senior'
    search_tokens TEXT     -- For full-text search
);

-- Materialized view for summary
CREATE TABLE user_summary (
    id INTEGER PRIMARY KEY DEFAULT 1,
    total_users INTEGER NOT NULL,
    average_age DECIMAL(5,2) NOT NULL,
    users_by_age_group JSON NOT NULL,
    last_updated TIMESTAMP NOT NULL
);

-- Indexes for read optimization
CREATE INDEX idx_user_views_age ON user_views(age);
CREATE INDEX idx_user_views_age_group ON user_views(age_group);
CREATE INDEX idx_user_views_search ON user_views USING gin(to_tsvector('english', search_tokens));
```

## 🔄 CQRS with Event Sourcing

```python
from typing import List
from dataclasses import dataclass

@dataclass
class Event:
    event_id: str
    aggregate_id: str
    event_type: str
    data: dict
    timestamp: datetime
    version: int

class EventStore:
    def append_events(self, aggregate_id: str, events: List[Event], expected_version: int):
        # Optimistic concurrency control
        current_version = self.get_current_version(aggregate_id)
        if current_version != expected_version:
            raise ConcurrencyException("Version mismatch")
        
        # Store events
        for event in events:
            self.store_event(event)
    
    def get_events(self, aggregate_id: str, from_version: int = 0) -> List[Event]:
        return self.query_events(aggregate_id, from_version)

class UserAggregate:
    def __init__(self, user_id: str):
        self.id = user_id
        self.email = None
        self.name = None
        self.age = None
        self.version = 0
        self.uncommitted_events = []
    
    def create_user(self, email: str, name: str, age: int):
        if self.email is not None:
            raise ValueError("User already exists")
        
        event = Event(
            event_id=generate_id(),
            aggregate_id=self.id,
            event_type="UserCreated",
            data={"email": email, "name": name, "age": age},
            timestamp=datetime.utcnow(),
            version=self.version + 1
        )
        self.apply_event(event)
        self.uncommitted_events.append(event)
    
    def update_user(self, name: str = None, age: int = None):
        if self.email is None:
            raise ValueError("User does not exist")
        
        changes = {}
        if name and name != self.name:
            changes["name"] = name
        if age and age != self.age:
            changes["age"] = age
        
        if changes:
            event = Event(
                event_id=generate_id(),
                aggregate_id=self.id,
                event_type="UserUpdated",
                data=changes,
                timestamp=datetime.utcnow(),
                version=self.version + 1
            )
            self.apply_event(event)
            self.uncommitted_events.append(event)
    
    def apply_event(self, event: Event):
        if event.event_type == "UserCreated":
            self.email = event.data["email"]
            self.name = event.data["name"]
            self.age = event.data["age"]
        elif event.event_type == "UserUpdated":
            if "name" in event.data:
                self.name = event.data["name"]
            if "age" in event.data:
                self.age = event.data["age"]
        
        self.version = event.version
    
    def get_uncommitted_events(self) -> List[Event]:
        return self.uncommitted_events
    
    def mark_events_as_committed(self):
        self.uncommitted_events = []
```

## 📊 Benefits & Trade-offs

### **Benefits**
- ✅ **Performance**: Optimized read and write models
- ✅ **Scalability**: Independent scaling of reads and writes
- ✅ **Flexibility**: Multiple read models for different use cases
- ✅ **Separation of Concerns**: Clear boundaries between commands and queries
- ✅ **Auditability**: Complete event history with Event Sourcing

### **Trade-offs**
- ❌ **Complexity**: More moving parts and infrastructure
- ❌ **Eventual Consistency**: Read models may lag behind writes
- ❌ **Data Duplication**: Same data stored in multiple formats
- ❌ **Development Overhead**: More code to maintain
- ❌ **Debugging Difficulty**: Distributed data flows

## 🎯 When to Use CQRS

### **Good Candidates**
- **High Read/Write Ratio**: 100:1 or higher
- **Complex Business Logic**: Different validation rules for commands vs queries
- **Performance Requirements**: Need sub-second read performance
- **Multiple Read Models**: Different views of same data
- **Audit Requirements**: Need complete change history

### **Not Suitable For**
- **Simple CRUD Applications**: Overhead not justified
- **Small Scale**: Additional complexity not needed
- **Strong Consistency Requirements**: Eventual consistency not acceptable
- **Limited Team Experience**: Requires architectural expertise

## 🔧 Implementation Patterns

### **1. Simple CQRS**
```python
# Separate read and write models, same database
class UserService:
    def __init__(self, write_repo, read_repo):
        self.write_repo = write_repo  # Normalized tables
        self.read_repo = read_repo    # Denormalized views
    
    def create_user(self, command):
        # Write to normalized tables
        user = self.write_repo.create(command)
        
        # Update denormalized views
        self.read_repo.update_user_view(user)
```

### **2. CQRS with Event Sourcing**
```python
# Complete event sourcing with projections
class UserCommandHandler:
    def __init__(self, event_store):
        self.event_store = event_store
    
    def handle_create_user(self, command):
        aggregate = UserAggregate(command.user_id)
        aggregate.create_user(command.email, command.name, command.age)
        
        # Store events
        self.event_store.append_events(
            aggregate.id,
            aggregate.get_uncommitted_events(),
            aggregate.version - 1
        )
```

### **3. CQRS with Microservices**
```python
# Commands and queries in separate services
class UserCommandService:
    def create_user(self, command):
        # Process command
        user = self.repository.create(command)
        
        # Publish event to message bus
        self.event_bus.publish('UserCreated', user.to_dict())

class UserQueryService:
    def __init__(self):
        # Subscribe to events
        self.event_bus.subscribe('UserCreated', self.handle_user_created)
    
    def handle_user_created(self, event):
        # Update read model
        self.read_repository.create_user_view(event)
```

## 🛠️ Technology Stack

### **Event Store Options**
- **EventStore**: Purpose-built event store
- **PostgreSQL**: JSON events in relational DB
- **Apache Kafka**: Streaming event log
- **AWS DynamoDB**: NoSQL event storage

### **Read Model Storage**
- **Elasticsearch**: Full-text search and analytics
- **Redis**: In-memory caching
- **MongoDB**: Document-based read models
- **PostgreSQL**: Materialized views

### **Message Bus**
- **RabbitMQ**: Reliable message delivery
- **Apache Kafka**: High-throughput streaming
- **AWS SQS/SNS**: Managed messaging
- **Azure Service Bus**: Enterprise messaging

## 🔗 Related Patterns

- [[Event Sourcing]] - Complete audit trail
- [[Saga Pattern]] - Distributed transactions
- [[Event-Driven Architecture]] - Loose coupling
- [[Microservices Pattern]] - Service decomposition
- [[Database Sharding]] - Data partitioning

## 📚 Further Reading

- **Books**:
  - "Implementing Domain-Driven Design" - Vaughn Vernon
  - "Building Event-Driven Microservices" - Adam Bellemare
  - "Event Sourcing and CQRS" - Alexey Zimarev

- **Articles**:
  - [CQRS by Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
  - [Event Sourcing by Greg Young](https://eventstore.com/blog/event-sourcing-basics)

---

**Key Takeaway**: CQRS shines when read and write patterns differ significantly. Use it for complex domains with performance requirements, but be prepared for increased complexity and eventual consistency challenges. 