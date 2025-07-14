# Saga Pattern

**Tags**: #saga #distributed-transactions #microservices
**Ngày tạo**: 2024-01-01

## 🎯 Saga Overview

Saga Pattern manages distributed transactions across microservices by breaking them into compensable steps.

### Core Concepts
- **Saga**: Sequence of local transactions
- **Compensation**: Rollback actions for failed steps
- **Coordination**: Choreography or Orchestration

## 🏗️ Implementation Types

### Choreography-based Saga
```python
class OrderSaga:
    def handle_order_created(self, event):
        # Step 1: Process payment
        self.event_bus.publish('ProcessPayment', event)
    
    def handle_payment_failed(self, event):
        # Compensate: Cancel order
        self.event_bus.publish('CancelOrder', event)
```

### Orchestration-based Saga
```python
class OrderSagaOrchestrator:
    def execute_order_saga(self, order_data):
        try:
            payment_result = self.payment_service.process(order_data)
            inventory_result = self.inventory_service.reserve(order_data)
            return self.complete_order(order_data)
        except Exception:
            return self.compensate(order_data)
```

## 📊 Choreography vs Orchestration

| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| Coupling | Loose | Tight |
| Monitoring | Difficult | Easy |
| Fault Tolerance | Resilient | Single point of failure |

## ⚠️ Challenges
- Isolation issues
- Compensation complexity  
- Timeout handling
- Debugging distributed flows

## 🎯 When to Use
- Distributed transactions across services
- Long-running business processes
- Need eventual consistency
- Complex compensation required

**Use Saga for reliable distributed transactions with compensation!** 