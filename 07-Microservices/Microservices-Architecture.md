# Microservices Architecture

**Tags**: #microservices #architecture #distributed-systems #scalability
**Date**: 2024-01-01

## 📝 Overview

Microservices architecture là approach chia large applications thành multiple small, independent services. Mỗi service có own database, deployment process, và business logic riêng biệt, communicate qua well-defined APIs.

## 🎯 Core Principles

### **Service Characteristics**
```
1. Single Responsibility
   - Mỗi service handle một business capability
   - Autonomous team ownership
   - Independent deployment

2. Decentralized
   - Own data store
   - Independent technology stack
   - Distributed governance

3. Communication
   - API-first design
   - Asynchronous messaging
   - Event-driven architecture

4. Resilience
   - Failure isolation
   - Circuit breakers
   - Graceful degradation
```

### **Microservices vs Monolith**
```python
comparison = {
    "monolith": {
        "pros": [
            "Simple development/deployment",
            "Easy testing",
            "Good performance",
            "Simple monitoring"
        ],
        "cons": [
            "Limited scalability",
            "Technology lock-in", 
            "Large team coordination",
            "Deployment risk"
        ]
    },
    "microservices": {
        "pros": [
            "Independent scaling",
            "Technology diversity",
            "Team autonomy",
            "Fault isolation"
        ],
        "cons": [
            "Distributed system complexity",
            "Network latency",
            "Data consistency challenges",
            "Operational overhead"
        ]
    }
}
```

## 🏗️ Service Decomposition Strategies

### **Domain-Driven Design (DDD)**
```python
class DomainService:
    """
    Service được tổ chức theo business domains
    """
    def __init__(self, domain_name):
        self.domain = domain_name
        self.bounded_context = self.define_bounded_context()
        self.aggregates = self.identify_aggregates()
    
    def define_bounded_context(self):
        """
        Xác định boundaries của domain
        """
        return {
            "user_management": {
                "entities": ["User", "Profile", "Preferences"],
                "services": ["AuthenticationService", "UserProfileService"],
                "events": ["UserRegistered", "ProfileUpdated"]
            },
            "order_management": {
                "entities": ["Order", "OrderItem", "Payment"],
                "services": ["OrderService", "PaymentService"],
                "events": ["OrderCreated", "PaymentProcessed"]
            },
            "inventory_management": {
                "entities": ["Product", "Stock", "Reservation"],
                "services": ["InventoryService", "ReservationService"],
                "events": ["StockUpdated", "ItemReserved"]
            }
        }

# Example: E-commerce domain decomposition
class ECommerceDecomposition:
    def __init__(self):
        self.services = {
            "user_service": {
                "domain": "User Management",
                "responsibilities": [
                    "User registration/authentication",
                    "Profile management",
                    "Preferences"
                ],
                "data": ["users", "user_profiles", "user_preferences"],
                "apis": ["/users", "/auth", "/profiles"]
            },
            "product_service": {
                "domain": "Product Catalog",
                "responsibilities": [
                    "Product information",
                    "Category management", 
                    "Search functionality"
                ],
                "data": ["products", "categories", "product_images"],
                "apis": ["/products", "/categories", "/search"]
            },
            "order_service": {
                "domain": "Order Management",
                "responsibilities": [
                    "Order processing",
                    "Order history",
                    "Order status tracking"
                ],
                "data": ["orders", "order_items", "order_status"],
                "apis": ["/orders", "/order-history"]
            }
        }
```

### **Data Ownership Patterns**
```python
class DataOwnershipPattern:
    """
    Mỗi service owns its data completely
    """
    
    def __init__(self, service_name):
        self.service_name = service_name
        self.private_database = self.setup_database()
        self.api_endpoints = self.define_apis()
    
    def setup_database(self):
        """
        Database per service pattern
        """
        return {
            "type": "PostgreSQL",  # or MongoDB, MySQL, etc.
            "schema": f"{self.service_name}_db",
            "connection": f"postgresql://localhost/{self.service_name}_db",
            "migrations": f"migrations/{self.service_name}/",
            "access": "service_exclusive"  # Only this service can access
        }
    
    def define_apis(self):
        """
        Well-defined APIs for data access
        """
        return {
            "public_endpoints": [
                f"GET /{self.service_name}",
                f"POST /{self.service_name}",
                f"PUT /{self.service_name}/{{id}}",
                f"DELETE /{self.service_name}/{{id}}"
            ],
            "internal_endpoints": [
                f"GET /{self.service_name}/internal/health",
                f"GET /{self.service_name}/internal/metrics"
            ]
        }

# Anti-pattern: Shared database
class SharedDatabaseAntiPattern:
    """
    AVOID: Multiple services sharing same database
    """
    def __init__(self):
        self.problems = [
            "Tight coupling between services",
            "Schema changes affect multiple services",
            "Difficult to scale individual services",
            "Technology lock-in",
            "Deployment coordination required"
        ]
```

## 🔄 Communication Patterns

### **Synchronous Communication**
```python
import requests
import circuit_breaker
import asyncio

class SynchronousServiceClient:
    """
    Direct API calls between services
    """
    def __init__(self, base_url, timeout=5):
        self.base_url = base_url
        self.timeout = timeout
        self.circuit_breaker = circuit_breaker.CircuitBreaker(
            failure_threshold=5,
            timeout=30
        )
    
    @circuit_breaker.protected
    def call_service(self, endpoint, method="GET", data=None):
        """
        Make HTTP call to another service
        """
        url = f"{self.base_url}{endpoint}"
        
        try:
            response = requests.request(
                method=method,
                url=url,
                json=data,
                timeout=self.timeout,
                headers={"Content-Type": "application/json"}
            )
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.Timeout:
            raise ServiceTimeoutError(f"Service {self.base_url} timed out")
        except requests.exceptions.ConnectionError:
            raise ServiceUnavailableError(f"Service {self.base_url} unavailable")
    
    def get_user_profile(self, user_id):
        return self.call_service(f"/users/{user_id}/profile")
    
    def create_order(self, order_data):
        return self.call_service("/orders", method="POST", data=order_data)

# Service mesh implementation with Istio
class ServiceMeshCommunication:
    """
    Using service mesh for service-to-service communication
    """
    def __init__(self):
        self.service_registry = self.setup_service_registry()
        self.load_balancer = self.setup_load_balancer()
        self.security_policies = self.setup_security()
    
    def setup_service_registry(self):
        return {
            "discovery": "Consul/Eureka",
            "health_checks": "automatic",
            "load_balancing": "round_robin"
        }
    
    def call_service_via_mesh(self, service_name, endpoint, data=None):
        # Service mesh handles:
        # - Service discovery
        # - Load balancing  
        # - Circuit breaking
        # - Retries
        # - Authentication
        # - Monitoring
        
        service_url = self.service_registry.discover(service_name)
        return self.make_secure_call(service_url, endpoint, data)
```

### **Asynchronous Communication**
```python
import asyncio
import json
from abc import ABC, abstractmethod

class EventBus(ABC):
    """
    Abstract event bus for async communication
    """
    @abstractmethod
    def publish(self, event_type, data):
        pass
    
    @abstractmethod
    def subscribe(self, event_type, handler):
        pass

class RabbitMQEventBus(EventBus):
    """
    RabbitMQ implementation of event bus
    """
    def __init__(self, connection_string):
        self.connection = self.setup_connection(connection_string)
        self.channel = self.connection.channel()
        self.subscribers = {}
    
    def publish(self, event_type, data):
        """
        Publish event to exchange
        """
        event_data = {
            "event_type": event_type,
            "data": data,
            "timestamp": time.time(),
            "correlation_id": str(uuid.uuid4())
        }
        
        self.channel.basic_publish(
            exchange='events',
            routing_key=event_type,
            body=json.dumps(event_data),
            properties=pika.BasicProperties(
                delivery_mode=2,  # Persistent
                content_type='application/json'
            )
        )
    
    def subscribe(self, event_type, handler):
        """
        Subscribe to specific event type
        """
        queue_name = f"{event_type}_queue"
        self.channel.queue_declare(queue=queue_name, durable=True)
        self.channel.queue_bind(
            exchange='events',
            queue=queue_name,
            routing_key=event_type
        )
        
        def callback(ch, method, properties, body):
            try:
                event_data = json.loads(body)
                handler(event_data)
                ch.basic_ack(delivery_tag=method.delivery_tag)
            except Exception as e:
                # Handle error, possibly send to DLQ
                ch.basic_nack(
                    delivery_tag=method.delivery_tag,
                    requeue=False
                )
        
        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=callback
        )

# Event sourcing pattern
class EventSourcingService:
    """
    Service sử dụng event sourcing pattern
    """
    def __init__(self, event_store, event_bus):
        self.event_store = event_store
        self.event_bus = event_bus
    
    def handle_command(self, command):
        """
        Process command and generate events
        """
        # Validate command
        if not self.validate_command(command):
            raise InvalidCommandError()
        
        # Generate events
        events = self.process_command(command)
        
        # Store events
        for event in events:
            self.event_store.append(event)
            
            # Publish for other services
            self.event_bus.publish(event.type, event.data)
        
        return events
    
    def rebuild_state(self, entity_id):
        """
        Rebuild entity state from events
        """
        events = self.event_store.get_events(entity_id)
        state = {}
        
        for event in events:
            state = self.apply_event(state, event)
        
        return state

# Saga pattern for distributed transactions
class OrderSagaOrchestrator:
    """
    Orchestrate distributed transaction across multiple services
    """
    def __init__(self, event_bus):
        self.event_bus = event_bus
        self.saga_state = {}
    
    def start_order_saga(self, order_data):
        saga_id = str(uuid.uuid4())
        
        self.saga_state[saga_id] = {
            "step": "reserve_inventory",
            "order_data": order_data,
            "compensations": []
        }
        
        # Start with inventory reservation
        self.event_bus.publish("reserve_inventory", {
            "saga_id": saga_id,
            "order_id": order_data["order_id"],
            "items": order_data["items"]
        })
    
    def handle_inventory_reserved(self, event_data):
        saga_id = event_data["saga_id"]
        saga = self.saga_state[saga_id]
        
        # Add compensation action
        saga["compensations"].append({
            "action": "release_inventory",
            "data": event_data
        })
        
        # Next step: process payment
        saga["step"] = "process_payment"
        self.event_bus.publish("process_payment", {
            "saga_id": saga_id,
            "amount": saga["order_data"]["total"],
            "payment_method": saga["order_data"]["payment_method"]
        })
    
    def handle_payment_failed(self, event_data):
        saga_id = event_data["saga_id"]
        saga = self.saga_state[saga_id]
        
        # Execute compensations in reverse order
        for compensation in reversed(saga["compensations"]):
            self.event_bus.publish(compensation["action"], compensation["data"])
        
        # Mark saga as failed
        saga["status"] = "failed"
```

## 🛡️ Resilience Patterns

### **Circuit Breaker Pattern**
```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject calls
    HALF_OPEN = "half_open" # Testing if service recovered

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60, success_threshold=3):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.success_threshold = success_threshold
        
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
    
    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise CircuitBreakerOpenError("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        self.failure_count = 0
        
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.success_count = 0
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
    
    def _should_attempt_reset(self):
        return (
            self.last_failure_time and
            time.time() - self.last_failure_time >= self.timeout
        )

# Usage example
class PaymentServiceClient:
    def __init__(self):
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=3,
            timeout=30
        )
    
    def process_payment(self, payment_data):
        try:
            return self.circuit_breaker.call(
                self._make_payment_request,
                payment_data
            )
        except CircuitBreakerOpenError:
            # Fallback: queue payment for later processing
            return self._queue_payment_for_retry(payment_data)
```

### **Retry Pattern with Exponential Backoff**
```python
import random
import time
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=1, max_delay=60, 
                      exponential_base=2, jitter=True):
    """
    Decorator for retry with exponential backoff
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except (ConnectionError, TimeoutError) as e:
                    last_exception = e
                    
                    if attempt == max_retries:
                        break
                    
                    # Calculate delay
                    delay = min(
                        base_delay * (exponential_base ** attempt),
                        max_delay
                    )
                    
                    # Add jitter to prevent thundering herd
                    if jitter:
                        delay *= (0.5 + random.random() * 0.5)
                    
                    time.sleep(delay)
            
            raise last_exception
        return wrapper
    return decorator

# Usage
class ExternalServiceClient:
    @retry_with_backoff(max_retries=3, base_delay=1)
    def call_external_api(self, data):
        response = requests.post(
            "https://api.external.com/endpoint",
            json=data,
            timeout=5
        )
        response.raise_for_status()
        return response.json()
```

### **Bulkhead Pattern**
```python
import threading
import asyncio
from concurrent.futures import ThreadPoolExecutor

class BulkheadService:
    """
    Isolate resources to prevent cascading failures
    """
    def __init__(self):
        # Separate thread pools for different operations
        self.critical_pool = ThreadPoolExecutor(
            max_workers=10,
            thread_name_prefix="critical"
        )
        self.non_critical_pool = ThreadPoolExecutor(
            max_workers=5,
            thread_name_prefix="non_critical"
        )
        self.background_pool = ThreadPoolExecutor(
            max_workers=3,
            thread_name_prefix="background"
        )
    
    def process_critical_request(self, request):
        """
        High priority requests get dedicated resources
        """
        future = self.critical_pool.submit(self._handle_critical, request)
        return future.result(timeout=5)
    
    def process_non_critical_request(self, request):
        """
        Normal requests use separate pool
        """
        future = self.non_critical_pool.submit(self._handle_normal, request)
        return future.result(timeout=10)
    
    def process_background_task(self, task):
        """
        Background tasks won't affect user-facing operations
        """
        self.background_pool.submit(self._handle_background, task)
```

## 📊 Monitoring & Observability

### **Distributed Tracing**
```python
import opentelemetry.trace as trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

class DistributedTracing:
    def __init__(self, service_name):
        self.service_name = service_name
        self.setup_tracing()
    
    def setup_tracing(self):
        # Configure tracer
        trace.set_tracer_provider(TracerProvider())
        tracer = trace.get_tracer(self.service_name)
        
        # Configure Jaeger exporter
        jaeger_exporter = JaegerExporter(
            agent_host_name="localhost",
            agent_port=6831,
        )
        
        span_processor = BatchSpanProcessor(jaeger_exporter)
        trace.get_tracer_provider().add_span_processor(span_processor)
        
        self.tracer = tracer
    
    def trace_operation(self, operation_name):
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                with self.tracer.start_as_current_span(operation_name) as span:
                    # Add attributes
                    span.set_attribute("service.name", self.service_name)
                    span.set_attribute("operation.name", operation_name)
                    
                    try:
                        result = func(*args, **kwargs)
                        span.set_attribute("success", True)
                        return result
                    except Exception as e:
                        span.set_attribute("success", False)
                        span.set_attribute("error.message", str(e))
                        raise
            return wrapper
        return decorator

# Service implementation with tracing
class UserService:
    def __init__(self):
        self.tracing = DistributedTracing("user-service")
    
    @tracing.trace_operation("get_user_profile")
    def get_user_profile(self, user_id):
        # Implementation with automatic tracing
        return self.database.get_user(user_id)
    
    @tracing.trace_operation("update_user_profile")
    def update_user_profile(self, user_id, profile_data):
        # Implementation with automatic tracing
        return self.database.update_user(user_id, profile_data)
```

## 🔗 Liên kết và Tham khảo

### **Related Topics**
- [[02-System-Components/API-Gateway/API-Gateway-Pattern|API Gateway Pattern]] - Entry point for microservices
- [[02-System-Components/Message-Queues/Message-Queue-Systems|Message Queue Systems]] - Async communication
- [[06-Database-Design/SQL-Databases/SQL-Database-Design|Database Design]] - Data patterns in microservices
- [[10-DevOps-Deployment/DevOps-Best-Practices|Container Orchestration]] - Deployment strategies

### **Further Reading**
- "Building Microservices" - Sam Newman
- "Microservices Patterns" - Chris Richardson
- "Production-Ready Microservices" - Susan Fowler
- "The Microservices Book" - Eberhard Wolff

### **Best Practices**
- Start with monolith, extract services gradually
- Design for failure and resilience
- Implement comprehensive monitoring
- Use event-driven architecture
- Practice continuous delivery

---

*Microservices architecture requires careful design và disciplined implementation. Focus on business capabilities, maintain service autonomy, và ensure operational excellence.* 