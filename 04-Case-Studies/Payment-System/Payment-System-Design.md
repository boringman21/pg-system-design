# Payment System Design

**Tags**: #case-study #payment #fintech #security #compliance
**Date**: 2024-01-01
**Difficulty**: Hard

## 📝 Problem Statement

Thiết kế một payment system như Stripe, PayPal, hoặc VNPay có thể:
- Xử lý transactions an toàn và reliable
- Hỗ trợ multiple payment methods
- Đảm bảo compliance với regulations
- Scale đến millions of transactions per day

## 🎯 Requirements Clarification

### Functional Requirements
- [ ] **Payment Processing**: Credit cards, bank transfers, digital wallets
- [ ] **Multi-currency**: Support various currencies và exchange rates
- [ ] **Merchant Integration**: APIs cho merchants
- [ ] **Refunds & Chargebacks**: Reverse transactions
- [ ] **Recurring Payments**: Subscriptions, recurring billing
- [ ] **Fraud Detection**: Real-time fraud prevention
- [ ] **Compliance**: PCI DSS, AML, KYC requirements

### Non-functional Requirements
- [ ] **Scalability**: 10M transactions/day, 10K TPS peak
- [ ] **Availability**: 99.99% uptime (52 minutes downtime/year)
- [ ] **Security**: PCI DSS Level 1 compliance
- [ ] **Latency**: <200ms for payment authorization
- [ ] **Consistency**: Strong consistency cho financial data
- [ ] **Audit**: Complete audit trail for all transactions

### Scale Estimates
- **Transactions**: 10M per day = 115 TPS average, 1K TPS peak
- **Merchants**: 100K active merchants
- **Users**: 50M registered users
- **Storage**: 10M transactions * 365 days * 5 years = 18B records
- **Data Size**: ~2KB per transaction = 36TB total

## 🏗️ High-Level Architecture

```
[Mobile/Web Apps] → [API Gateway] → [Load Balancer] → [Payment Service]
                                                            ↓
[Card Networks] ← [Payment Processor] ← [Fraud Detection] ← [Auth Service]
                                                            ↓
[Notification] ← [Event Bus] ← [Transaction DB] → [Audit Service]
                                    ↓
                            [Analytics/Reporting]
```

### Core Components
1. **Payment Gateway**: Entry point cho all payment requests
2. **Authorization Service**: Validate và authorize payments
3. **Fraud Detection**: Real-time fraud analysis
4. **Payment Processor**: Interface với card networks
5. **Transaction Database**: Store payment records
6. **Audit Service**: Compliance và logging
7. **Notification Service**: Real-time updates

## 🔍 Detailed Design

### API Design
```python
# Payment Processing
POST /api/v1/payments
{
    "amount": 1000,  # cents
    "currency": "USD",
    "payment_method": {
        "type": "card",
        "card_number": "4111111111111111",
        "exp_month": 12,
        "exp_year": 2025,
        "cvv": "123"
    },
    "merchant_id": "merchant_123",
    "customer_id": "customer_456",
    "description": "Order #12345",
    "metadata": {"order_id": "12345"}
}

# Response
{
    "id": "payment_789",
    "status": "succeeded",
    "amount": 1000,
    "currency": "USD",
    "created": "2024-01-01T10:00:00Z",
    "fees": 30,  # Processing fee
    "net": 970   # Amount to merchant
}

# Refund
POST /api/v1/refunds
{
    "payment_id": "payment_789",
    "amount": 1000,  # Full or partial refund
    "reason": "requested_by_customer"
}

# Webhook for merchants
POST /merchant/webhook
{
    "event": "payment.succeeded",
    "data": {
        "payment_id": "payment_789",
        "amount": 1000,
        "status": "succeeded"
    }
}
```

### Database Schema
```sql
-- Payments table
CREATE TABLE payments (
    id VARCHAR(50) PRIMARY KEY,
    merchant_id VARCHAR(50) NOT NULL,
    customer_id VARCHAR(50),
    amount BIGINT NOT NULL,  -- In cents
    currency VARCHAR(3) NOT NULL,
    status ENUM('pending', 'succeeded', 'failed', 'canceled') NOT NULL,
    payment_method_id VARCHAR(50),
    description TEXT,
    metadata JSON,
    fees BIGINT,
    net BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_merchant_created (merchant_id, created_at),
    INDEX idx_customer_created (customer_id, created_at),
    INDEX idx_status (status)
);

-- Payment methods table
CREATE TABLE payment_methods (
    id VARCHAR(50) PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL,
    type ENUM('card', 'bank_account', 'digital_wallet') NOT NULL,
    
    -- Card data (encrypted)
    card_brand VARCHAR(20),
    card_last4 VARCHAR(4),
    card_exp_month INT,
    card_exp_year INT,
    card_fingerprint VARCHAR(64),
    
    -- Bank account data
    bank_name VARCHAR(100),
    account_last4 VARCHAR(4),
    
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_customer (customer_id)
);

-- Transactions table (for audit)
CREATE TABLE transactions (
    id VARCHAR(50) PRIMARY KEY,
    payment_id VARCHAR(50) NOT NULL,
    type ENUM('payment', 'refund', 'chargeback', 'fee') NOT NULL,
    amount BIGINT NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(20) NOT NULL,
    gateway_response JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_payment (payment_id),
    INDEX idx_created (created_at),
    FOREIGN KEY (payment_id) REFERENCES payments(id)
);

-- Merchants table
CREATE TABLE merchants (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    api_key VARCHAR(64) NOT NULL,
    secret_key VARCHAR(64) NOT NULL,
    webhook_url VARCHAR(500),
    fee_percentage DECIMAL(5,4) DEFAULT 0.0290,  -- 2.9%
    fee_fixed BIGINT DEFAULT 30,  -- 30 cents
    status ENUM('active', 'suspended', 'pending') DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE KEY uk_api_key (api_key),
    INDEX idx_status (status)
);
```

### Payment Processing Flow
```python
class PaymentService:
    def __init__(self, fraud_detector, payment_processor, event_bus):
        self.fraud_detector = fraud_detector
        self.payment_processor = payment_processor
        self.event_bus = event_bus
    
    async def process_payment(self, payment_request):
        # 1. Validate request
        self.validate_payment_request(payment_request)
        
        # 2. Create payment record
        payment = Payment(
            id=generate_payment_id(),
            merchant_id=payment_request.merchant_id,
            amount=payment_request.amount,
            currency=payment_request.currency,
            status='pending'
        )
        payment.save()
        
        try:
            # 3. Fraud detection
            fraud_score = await self.fraud_detector.analyze(payment_request)
            if fraud_score > 0.8:
                payment.status = 'failed'
                payment.failure_reason = 'fraud_detected'
                payment.save()
                
                await self.event_bus.publish('payment.failed', {
                    'payment_id': payment.id,
                    'reason': 'fraud_detected'
                })
                return payment
            
            # 4. Process with payment processor
            result = await self.payment_processor.authorize(payment_request)
            
            if result.success:
                payment.status = 'succeeded'
                payment.processor_transaction_id = result.transaction_id
                payment.fees = self.calculate_fees(payment.amount)
                payment.net = payment.amount - payment.fees
            else:
                payment.status = 'failed'
                payment.failure_reason = result.error_code
            
            payment.save()
            
            # 5. Publish event
            await self.event_bus.publish(f'payment.{payment.status}', {
                'payment_id': payment.id,
                'merchant_id': payment.merchant_id,
                'amount': payment.amount,
                'currency': payment.currency
            })
            
            # 6. Send webhook to merchant
            await self.send_webhook(payment)
            
            return payment
            
        except Exception as e:
            payment.status = 'failed'
            payment.failure_reason = str(e)
            payment.save()
            
            await self.event_bus.publish('payment.failed', {
                'payment_id': payment.id,
                'error': str(e)
            })
            
            raise
    
    def calculate_fees(self, amount):
        # Stripe-like fee structure: 2.9% + $0.30
        percentage_fee = amount * 0.029
        fixed_fee = 30  # cents
        return int(percentage_fee + fixed_fee)
```

### Fraud Detection System
```python
class FraudDetector:
    def __init__(self, ml_model, rules_engine):
        self.ml_model = ml_model
        self.rules_engine = rules_engine
    
    async def analyze(self, payment_request):
        # Extract features
        features = self.extract_features(payment_request)
        
        # Rule-based checks
        rule_score = self.rules_engine.evaluate(features)
        if rule_score > 0.9:
            return 1.0  # High fraud score
        
        # ML-based scoring
        ml_score = self.ml_model.predict(features)
        
        # Combine scores
        combined_score = max(rule_score, ml_score)
        
        # Log for analysis
        await self.log_fraud_check(payment_request, combined_score)
        
        return combined_score
    
    def extract_features(self, payment_request):
        return {
            'amount': payment_request.amount,
            'currency': payment_request.currency,
            'card_country': payment_request.card_country,
            'merchant_category': payment_request.merchant_category,
            'time_of_day': payment_request.timestamp.hour,
            'day_of_week': payment_request.timestamp.weekday(),
            'velocity_1h': self.get_velocity(payment_request.customer_id, hours=1),
            'velocity_24h': self.get_velocity(payment_request.customer_id, hours=24),
            'new_merchant': self.is_new_merchant(payment_request.customer_id, 
                                              payment_request.merchant_id),
            'device_fingerprint': payment_request.device_fingerprint
        }

class FraudRulesEngine:
    def __init__(self):
        self.rules = [
            self.check_velocity_limits,
            self.check_amount_limits,
            self.check_location_mismatch,
            self.check_blacklist
        ]
    
    def evaluate(self, features):
        max_score = 0
        
        for rule in self.rules:
            score = rule(features)
            max_score = max(max_score, score)
        
        return max_score
    
    def check_velocity_limits(self, features):
        # Too many transactions in short time
        if features['velocity_1h'] > 10:
            return 0.9
        if features['velocity_24h'] > 50:
            return 0.8
        return 0.1
    
    def check_amount_limits(self, features):
        # Unusually large transaction
        if features['amount'] > 10000_00:  # $10,000
            return 0.7
        return 0.1
```

## 🔒 Security & Compliance

### PCI DSS Compliance
```python
class PCICompliance:
    def __init__(self):
        self.encryption_service = EncryptionService()
        self.token_vault = TokenVault()
    
    def handle_card_data(self, card_data):
        # Never store raw card data
        # Immediately tokenize
        token = self.token_vault.tokenize(card_data)
        
        # Clear sensitive data from memory
        self.clear_sensitive_data(card_data)
        
        return token
    
    def encrypt_pii(self, data):
        # Encrypt personally identifiable information
        return self.encryption_service.encrypt(data)
    
    def audit_access(self, user_id, action, resource):
        # Log all access to sensitive data
        audit_log = {
            'user_id': user_id,
            'action': action,
            'resource': resource,
            'timestamp': datetime.utcnow(),
            'ip_address': self.get_client_ip()
        }
        self.audit_logger.log(audit_log)

class TokenVault:
    def __init__(self):
        self.hsm = HardwareSecurityModule()
    
    def tokenize(self, card_data):
        # Generate random token
        token = secrets.token_urlsafe(32)
        
        # Encrypt card data with HSM
        encrypted_data = self.hsm.encrypt(json.dumps(card_data))
        
        # Store token -> encrypted data mapping
        self.store_token_mapping(token, encrypted_data)
        
        return token
    
    def detokenize(self, token):
        # Retrieve encrypted data
        encrypted_data = self.get_token_mapping(token)
        
        # Decrypt with HSM
        card_data = self.hsm.decrypt(encrypted_data)
        
        return json.loads(card_data)
```

## 📊 Monitoring & Observability

### Key Metrics
```python
class PaymentMetrics:
    def __init__(self, metrics_client):
        self.metrics = metrics_client
    
    def record_payment_attempt(self, payment):
        self.metrics.counter('payment.attempts').increment(
            tags={'merchant_id': payment.merchant_id}
        )
    
    def record_payment_success(self, payment):
        self.metrics.counter('payment.success').increment(
            tags={'merchant_id': payment.merchant_id}
        )
        
        # Record processing time
        self.metrics.histogram('payment.processing_time').record(
            payment.processing_time_ms
        )
    
    def record_payment_failure(self, payment, reason):
        self.metrics.counter('payment.failure').increment(
            tags={
                'merchant_id': payment.merchant_id,
                'failure_reason': reason
            }
        )
    
    def record_fraud_detection(self, score):
        self.metrics.histogram('fraud.score').record(score)
        
        if score > 0.8:
            self.metrics.counter('fraud.blocked').increment()
```

## 🎯 Trade-offs

| Decision | Pros | Cons |
|----------|------|------|
| **Strong Consistency** | Accurate financial data | Higher latency, reduced availability |
| **Microservices** | Scalability, fault isolation | Complexity, network latency |
| **Real-time Fraud Detection** | Better security | Higher processing cost |
| **Card Tokenization** | PCI compliance | Additional complexity |
| **Multi-processor Support** | Redundancy, better rates | Integration complexity |

## 🔗 Related Topics

- [[Event-Driven Architecture]] - Async payment processing
- [[Message Queue Systems]] - Reliable event handling
- [[Database Sharding]] - Scaling payment data
- [[Authentication Authorization]] - API security
- [[Performance Monitoring]] - Payment observability

## 📚 Further Reading

- PCI DSS Standards Documentation
- Stripe Engineering Blog
- "Building Secure and Reliable Systems" - Google SRE
- OWASP Payment Application Security Guidelines

---

**Key Takeaway**: Payment systems require exceptional attention to security, compliance, và reliability. Trade-offs between consistency và availability must be carefully considered, với security always being the top priority.

---

**Next Steps**: 
- [ ] Implement basic payment flow
- [ ] Add fraud detection rules
- [ ] Set up monitoring dashboards
- [ ] Ensure PCI DSS compliance