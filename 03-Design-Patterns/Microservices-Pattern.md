# Microservices Pattern

**Tags**: #pattern #microservices #architecture
**Date**: 2024-01-01

## 📝 Overview

Microservices pattern là architectural approach trong đó application được decompose thành nhiều small, independent services. Mỗi service có specific business responsibility và communicate qua well-defined APIs.

## 🎯 Core Principles

### **Single Responsibility**
- Mỗi service chỉ có một business capability
- High cohesion, loose coupling
- Dễ dàng understand và maintain

### **Decentralized**
- Services manage own data
- No shared databases
- Independent deployment

### **Technology Agnostic**
- Services có thể use different tech stacks
- Communication through APIs
- Platform independence

## 🏗️ Microservices vs Monolith

### **Monolithic Architecture**
```
┌─────────────────────────────────┐
│          Monolith App           │
├─────────────────────────────────┤
│  User Management   │  Orders    │
│  Product Catalog   │  Payment   │
│  Shopping Cart     │  Inventory │
└─────────────────────────────────┘
         Single Database
```

### **Microservices Architecture**
```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ User Service │  │Order Service │  │Product Service│
└──────────────┘  └──────────────┘  └──────────────┘
       │                 │                 │
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   User DB    │  │  Order DB    │  │ Product DB   │
└──────────────┘  └──────────────┘  └──────────────┘
```

## 🔧 Implementation Patterns

### **API Gateway Pattern**
```python
class APIGateway:
    def __init__(self):
        self.user_service = UserServiceClient()
        self.order_service = OrderServiceClient()
        self.product_service = ProductServiceClient()
    
    async def handle_request(self, request):
        # Authentication & Authorization
        user = await self.authenticate(request.token)
        
        # Route to appropriate service
        if request.path.startswith('/users'):
            return await self.user_service.handle(request)
        elif request.path.startswith('/orders'):
            return await self.order_service.handle(request)
        elif request.path.startswith('/products'):
            return await self.product_service.handle(request)
        
        # Rate limiting, logging, monitoring
        await self.log_request(request)
        return response
```

### **Service Discovery Pattern**
```python
class ServiceRegistry:
    def __init__(self):
        self.services = {}  # service_name -> [instances]
    
    def register_service(self, service_name, instance_info):
        if service_name not in self.services:
            self.services[service_name] = []
        self.services[service_name].append(instance_info)
    
    def discover_service(self, service_name):
        instances = self.services.get(service_name, [])
        # Load balancing - return healthy instance
        return self.select_healthy_instance(instances)
    
    def health_check(self):
        # Periodic health checks
        for service_name, instances in self.services.items():
            for instance in instances:
                if not self.is_healthy(instance):
                    self.remove_instance(service_name, instance)
```

### **Circuit Breaker Pattern**
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    async def call(self, service_func, *args, **kwargs):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'HALF_OPEN'
            else:
                raise CircuitBreakerOpenError()
        
        try:
            result = await service_func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
```

## 📡 Inter-Service Communication

### **Synchronous Communication**
```python
# REST API calls
class OrderService:
    def __init__(self):
        self.user_service = UserServiceClient()
        self.product_service = ProductServiceClient()
    
    async def create_order(self, user_id, items):
        # Validate user exists
        user = await self.user_service.get_user(user_id)
        if not user:
            raise UserNotFoundError()
        
        # Validate products
        for item in items:
            product = await self.product_service.get_product(item.product_id)
            if not product or product.stock < item.quantity:
                raise InsufficientStockError()
        
        # Create order
        order = Order(user_id=user_id, items=items)
        return await self.save_order(order)
```

### **Asynchronous Communication**
```python
# Event-driven communication
class OrderService:
    def __init__(self):
        self.event_bus = EventBus()
    
    async def create_order(self, order_data):
        # Create order
        order = await self.save_order(order_data)
        
        # Publish events
        await self.event_bus.publish('order.created', {
            'order_id': order.id,
            'user_id': order.user_id,
            'items': order.items,
            'total': order.total
        })
        
        return order

class InventoryService:
    def __init__(self):
        self.event_bus = EventBus()
        self.event_bus.subscribe('order.created', self.handle_order_created)
    
    async def handle_order_created(self, event_data):
        # Reduce inventory
        for item in event_data['items']:
            await self.reduce_stock(item['product_id'], item['quantity'])
        
        # Publish inventory updated event
        await self.event_bus.publish('inventory.updated', event_data)
```

## 💾 Data Management

### **Database per Service**
```python
# Each service has its own database
class UserService:
    def __init__(self):
        self.db = PostgreSQLConnection('user_db')
    
    async def get_user(self, user_id):
        return await self.db.query("SELECT * FROM users WHERE id = ?", user_id)

class OrderService:
    def __init__(self):
        self.db = MySQLConnection('order_db')
    
    async def get_orders(self, user_id):
        return await self.db.query("SELECT * FROM orders WHERE user_id = ?", user_id)
```

### **Saga Pattern for Distributed Transactions**
```python
class OrderSaga:
    def __init__(self):
        self.steps = []
        self.compensations = []
    
    async def execute(self, order_data):
        try:
            # Step 1: Reserve inventory
            reservation = await self.inventory_service.reserve_items(order_data.items)
            self.compensations.append(lambda: self.inventory_service.cancel_reservation(reservation.id))
            
            # Step 2: Process payment
            payment = await self.payment_service.charge_card(order_data.payment_info)
            self.compensations.append(lambda: self.payment_service.refund(payment.id))
            
            # Step 3: Create order
            order = await self.order_service.create_order(order_data)
            
            return order
            
        except Exception as e:
            # Execute compensations in reverse order
            for compensation in reversed(self.compensations):
                try:
                    await compensation()
                except:
                    # Log compensation failure
                    pass
            raise e
```

## 🚀 Deployment & DevOps

### **Containerization**
```dockerfile
# Dockerfile for microservice
FROM python:3.9-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .
EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### **Kubernetes Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: user-service:v1.0
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8000
  type: ClusterIP
```

## 📊 Monitoring & Observability

### **Distributed Tracing**
```python
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Setup tracing
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)

span_processor = BatchSpanProcessor(jaeger_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

class OrderService:
    async def create_order(self, order_data):
        with tracer.start_as_current_span("create_order") as span:
            span.set_attribute("user_id", order_data.user_id)
            span.set_attribute("item_count", len(order_data.items))
            
            # Call other services with tracing
            with tracer.start_as_current_span("validate_user"):
                user = await self.user_service.get_user(order_data.user_id)
            
            with tracer.start_as_current_span("process_payment"):
                payment = await self.payment_service.charge(order_data.payment)
            
            return order
```

### **Service Mesh**
```yaml
# Istio service mesh configuration
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: user-service
spec:
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: user-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: user-service
        subset: v1
      weight: 100
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: user-service
spec:
  host: user-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

## 🎯 Trade-offs

### **Advantages**
- ✅ **Independent Deployment**: Deploy services separately
- ✅ **Technology Diversity**: Different tech stacks per service
- ✅ **Scalability**: Scale services independently
- ✅ **Fault Isolation**: Failures don't cascade
- ✅ **Team Autonomy**: Teams can work independently

### **Disadvantages**
- ❌ **Complexity**: Distributed system complexity
- ❌ **Network Latency**: Inter-service communication overhead
- ❌ **Data Consistency**: No ACID transactions across services
- ❌ **Operational Overhead**: More services to monitor
- ❌ **Testing Complexity**: Integration testing challenges

## 🔗 Related Topics

- [[API Gateway]] - Single entry point pattern
- [[Service Discovery]] - Finding services dynamically
- [[Circuit Breaker]] - Fault tolerance pattern
- [[Event-Driven Architecture]] - Async communication
- [[Database Sharding]] - Data partitioning

## 📚 Further Reading

- "Building Microservices" by Sam Newman
- "Microservices Patterns" by Chris Richardson
- Martin Fowler's microservices articles
- Kubernetes and Docker documentation

---

**Key Takeaway**: Microservices offer flexibility và scalability nhưng introduce distributed system complexity. Start với monolith và decompose khi có clear business boundaries. 