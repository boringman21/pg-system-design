# Event-Driven Architecture

**Tags**: #pattern #event-driven #architecture #async #microservices
**Date**: 2024-01-01

## 📝 Overview

Event-Driven Architecture (EDA) là một kiến trúc pattern trong đó system components giao tiếp thông qua việc tạo ra và xử lý events. Pattern này cho phép loose coupling giữa các services và hỗ trợ scalability tốt hơn.

## 🎯 Core Concepts

### **Event**
```python
class Event:
    def __init__(self, event_type, data, timestamp=None, source=None):
        self.event_type = event_type
        self.data = data
        self.timestamp = timestamp or datetime.utcnow()
        self.source = source
        self.event_id = str(uuid.uuid4())
        self.version = "1.0"

# Example events
user_registered = Event(
    event_type="user.registered",
    data={"user_id": 123, "email": "user@example.com"},
    source="user_service"
)

order_placed = Event(
    event_type="order.placed", 
    data={"order_id": 456, "user_id": 123, "total": 99.99},
    source="order_service"
)
```

### **Event Producer**
```python
class EventProducer:
    def __init__(self, event_bus):
        self.event_bus = event_bus
    
    def publish_event(self, event_type, data):
        event = Event(event_type, data, source=self.__class__.__name__)
        
        # Add metadata
        event.correlation_id = self.get_correlation_id()
        event.causation_id = self.get_causation_id()
        
        # Publish to event bus
        self.event_bus.publish(event)
        
        # Log for audit
        logger.info(f"Published event: {event.event_type}")
        
        return event.event_id

# Example: User Service
class UserService(EventProducer):
    def register_user(self, user_data):
        # Business logic
        user = User.create(user_data)
        user.save()
        
        # Publish event
        self.publish_event("user.registered", {
            "user_id": user.id,
            "email": user.email,
            "registration_date": user.created_at
        })
        
        return user
```

### **Event Consumer**
```python
class EventConsumer:
    def __init__(self, event_bus):
        self.event_bus = event_bus
        self.event_handlers = {}
    
    def subscribe(self, event_type, handler):
        if event_type not in self.event_handlers:
            self.event_handlers[event_type] = []
        self.event_handlers[event_type].append(handler)
    
    def handle_event(self, event):
        handlers = self.event_handlers.get(event.event_type, [])
        
        for handler in handlers:
            try:
                handler(event)
            except Exception as e:
                logger.error(f"Error handling event {event.event_type}: {e}")
                # Send to dead letter queue
                self.send_to_dlq(event, e)

# Example: Email Service
class EmailService(EventConsumer):
    def __init__(self, event_bus):
        super().__init__(event_bus)
        self.subscribe("user.registered", self.send_welcome_email)
        self.subscribe("order.placed", self.send_order_confirmation)
    
    def send_welcome_email(self, event):
        user_data = event.data
        email_content = self.generate_welcome_email(user_data)
        self.send_email(user_data["email"], email_content)
        
        # Publish completion event
        self.publish_event("email.sent", {
            "email_type": "welcome",
            "recipient": user_data["email"],
            "user_id": user_data["user_id"]
        })
```

## 🏗️ Architecture Patterns

### **1. Event Bus Pattern**
```python
class EventBus:
    def __init__(self):
        self.subscribers = defaultdict(list)
        self.middleware = []
    
    def subscribe(self, event_type, handler):
        self.subscribers[event_type].append(handler)
    
    def publish(self, event):
        # Apply middleware
        for middleware in self.middleware:
            event = middleware.process(event)
        
        # Deliver to subscribers
        handlers = self.subscribers.get(event.event_type, [])
        for handler in handlers:
            try:
                handler(event)
            except Exception as e:
                self.handle_error(event, e)
    
    def add_middleware(self, middleware):
        self.middleware.append(middleware)

# Usage
event_bus = EventBus()

# Add logging middleware
event_bus.add_middleware(LoggingMiddleware())
event_bus.add_middleware(MetricsMiddleware())

# Subscribe services
user_service = UserService(event_bus)
email_service = EmailService(event_bus)
analytics_service = AnalyticsService(event_bus)
```

### **2. Event Store Pattern**
```python
class EventStore:
    def __init__(self):
        self.events = []
        self.snapshots = {}
    
    def append_event(self, stream_id, event):
        event.stream_id = stream_id
        event.sequence_number = self.get_next_sequence(stream_id)
        
        self.events.append(event)
        self.update_projections(event)
        
        return event.sequence_number
    
    def get_events(self, stream_id, from_sequence=0):
        return [e for e in self.events 
                if e.stream_id == stream_id and 
                e.sequence_number >= from_sequence]
    
    def replay_events(self, stream_id):
        events = self.get_events(stream_id)
        
        # Rebuild aggregate from events
        aggregate = self.create_aggregate()
        for event in events:
            aggregate.apply_event(event)
        
        return aggregate

# Example: Order Aggregate
class OrderAggregate:
    def __init__(self):
        self.order_id = None
        self.status = "pending"
        self.items = []
        self.total = 0
    
    def apply_event(self, event):
        if event.event_type == "order.created":
            self.order_id = event.data["order_id"]
            self.status = "created"
        elif event.event_type == "item.added":
            self.items.append(event.data["item"])
            self.total += event.data["price"]
        elif event.event_type == "order.confirmed":
            self.status = "confirmed"
```

### **3. CQRS + Event Sourcing**
```python
class CommandHandler:
    def __init__(self, event_store):
        self.event_store = event_store
    
    def handle_create_order(self, command):
        # Validate command
        self.validate_create_order(command)
        
        # Create events
        events = [
            Event("order.created", {
                "order_id": command.order_id,
                "user_id": command.user_id
            }),
            Event("items.added", {
                "order_id": command.order_id,
                "items": command.items
            })
        ]
        
        # Store events
        for event in events:
            self.event_store.append_event(
                stream_id=f"order-{command.order_id}",
                event=event
            )

class QueryHandler:
    def __init__(self, read_model):
        self.read_model = read_model
    
    def get_order(self, order_id):
        return self.read_model.get_order(order_id)
    
    def get_user_orders(self, user_id):
        return self.read_model.get_orders_by_user(user_id)

# Projection builder
class OrderProjection:
    def __init__(self):
        self.orders = {}
    
    def handle_event(self, event):
        if event.event_type == "order.created":
            self.orders[event.data["order_id"]] = {
                "id": event.data["order_id"],
                "user_id": event.data["user_id"],
                "status": "created",
                "items": [],
                "total": 0
            }
        elif event.event_type == "items.added":
            order_id = event.data["order_id"]
            if order_id in self.orders:
                self.orders[order_id]["items"].extend(event.data["items"])
                self.orders[order_id]["total"] = sum(
                    item["price"] for item in self.orders[order_id]["items"]
                )
```

## 📊 Implementation Patterns

### **Event Choreography**
```python
# Services react to events independently
class OrderService:
    def place_order(self, order_data):
        order = Order.create(order_data)
        
        # Publish event
        publish_event("order.placed", {
            "order_id": order.id,
            "user_id": order.user_id,
            "items": order.items,
            "total": order.total
        })

class PaymentService:
    def handle_order_placed(self, event):
        order_data = event.data
        
        # Process payment
        payment = self.process_payment(order_data)
        
        # Publish result
        if payment.success:
            publish_event("payment.completed", {
                "order_id": order_data["order_id"],
                "payment_id": payment.id
            })
        else:
            publish_event("payment.failed", {
                "order_id": order_data["order_id"],
                "reason": payment.error
            })

class InventoryService:
    def handle_order_placed(self, event):
        # Reserve inventory
        success = self.reserve_items(event.data["items"])
        
        if success:
            publish_event("inventory.reserved", {
                "order_id": event.data["order_id"]
            })
        else:
            publish_event("inventory.insufficient", {
                "order_id": event.data["order_id"]
            })
```

### **Event Orchestration**
```python
class OrderOrchestrator:
    def __init__(self):
        self.state_machine = OrderStateMachine()
    
    def handle_order_placed(self, event):
        order_id = event.data["order_id"]
        
        # Start saga
        saga = OrderSaga(order_id)
        saga.start()
        
        # First step: reserve inventory
        self.send_command("reserve_inventory", {
            "order_id": order_id,
            "items": event.data["items"]
        })
    
    def handle_inventory_reserved(self, event):
        order_id = event.data["order_id"]
        saga = self.get_saga(order_id)
        
        # Next step: process payment
        self.send_command("process_payment", {
            "order_id": order_id,
            "amount": saga.total_amount
        })
    
    def handle_payment_completed(self, event):
        order_id = event.data["order_id"]
        
        # Final step: ship order
        self.send_command("ship_order", {
            "order_id": order_id
        })
        
        # Complete saga
        saga = self.get_saga(order_id)
        saga.complete()
```

## 🔍 Best Practices

### **Event Design**
```python
class EventDesignPrinciples:
    """
    1. Events should be immutable
    2. Events should contain sufficient data
    3. Events should have clear semantics
    4. Events should be versionable
    """
    
    def create_good_event(self):
        return Event(
            event_type="order.placed",  # Clear, past tense
            data={
                "order_id": "12345",
                "user_id": "user123",
                "items": [{"id": 1, "name": "Product", "price": 10.0}],
                "total": 10.0,
                "currency": "USD",
                "timestamp": "2024-01-01T10:00:00Z"
            },
            version="1.0",  # Versioning for evolution
            schema_url="https://schemas.example.com/order-placed/v1"
        )

    def avoid_bad_event(self):
        # Bad: imperative, not enough data
        return Event(
            event_type="place_order",  # Should be past tense
            data={"order_id": "12345"}  # Missing critical data
        )
```

### **Error Handling**
```python
class EventErrorHandler:
    def __init__(self, dlq_client, retry_policy):
        self.dlq_client = dlq_client
        self.retry_policy = retry_policy
    
    def handle_processing_error(self, event, error):
        # Log error
        logger.error(f"Event processing failed: {error}")
        
        # Check retry policy
        if self.should_retry(event, error):
            # Retry with exponential backoff
            retry_delay = self.calculate_retry_delay(event.retry_count)
            self.schedule_retry(event, retry_delay)
        else:
            # Send to dead letter queue
            self.dlq_client.send_to_dlq(event, error)
    
    def should_retry(self, event, error):
        # Don't retry business logic errors
        if isinstance(error, ValidationError):
            return False
        
        # Don't retry if max attempts reached
        if event.retry_count >= self.retry_policy.max_retries:
            return False
        
        return True
```

## 🚀 Advantages

### **Benefits**
- **Loose Coupling**: Services không phụ thuộc trực tiếp vào nhau
- **Scalability**: Easy to scale individual components
- **Flexibility**: Easy to add new features without changing existing code
- **Resilience**: System có thể recover từ failures
- **Audit Trail**: Complete history of what happened

### **Use Cases**
- **E-commerce**: Order processing, inventory management
- **Banking**: Transaction processing, fraud detection
- **IoT**: Sensor data processing, real-time analytics
- **Gaming**: Player actions, leaderboards
- **Social Media**: Content updates, notifications

## ⚠️ Challenges

### **Complexity**
- **Debugging**: Harder to trace event flows
- **Testing**: More complex integration testing
- **Consistency**: Eventual consistency challenges
- **Ordering**: Event ordering issues

### **Solutions**
```python
class EventTracking:
    def add_correlation_id(self, event):
        # Track request across services
        event.correlation_id = self.get_correlation_id()
        return event
    
    def add_causation_id(self, event):
        # Track cause-effect relationships
        event.causation_id = self.get_current_event_id()
        return event

class EventualConsistency:
    def handle_inconsistency(self, event):
        # Implement compensation logic
        if self.detect_inconsistency(event):
            self.trigger_compensation(event)
```

## 🔗 Related Topics

- [[CQRS Pattern]] - Command Query Responsibility Segregation
- [[Saga Pattern]] - Distributed transaction management
- [[Message Queues]] - Infrastructure for events
- [[Microservices Architecture]] - Service decomposition

## 📚 Further Reading

- "Building Event-Driven Microservices" - Adam Bellemare
- "Event Sourcing" - Martin Fowler
- "Microservices Patterns" - Chris Richardson
- Apache Kafka documentation

---

**Key Takeaway**: Event-Driven Architecture cho phép xây dựng systems linh hoạt, scalable và resilient. Tuy nhiên, cần cẩn thận về complexity và consistency challenges khi implement.