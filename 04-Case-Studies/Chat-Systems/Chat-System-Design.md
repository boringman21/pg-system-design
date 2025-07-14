# Chat System - System Design

**Tags**: #case-study #design #real-time #messaging
**Date**: 2024-01-01
**Difficulty**: Medium

## 📝 Problem Statement

Thiết kế một chat system như WhatsApp, Slack, hoặc Facebook Messenger. System cần support:
- Real-time messaging giữa users
- Group chats
- Online status
- Message history
- File sharing
- Push notifications

## 🎯 Requirements Clarification

### Functional Requirements
- [ ] **1-on-1 Chat**: Direct messaging giữa 2 users
- [ ] **Group Chat**: Support 100 members per group
- [ ] **Online Status**: Show online/offline/last seen
- [ ] **Message History**: Persistent storage
- [ ] **File Sharing**: Images, documents (max 10MB)
- [ ] **Push Notifications**: Mobile & desktop
- [ ] **Message Status**: Sent, delivered, read receipts

### Non-functional Requirements
- [ ] **Scalability**: 10M active users, 1B messages/day
- [ ] **Performance**: Message delivery < 100ms
- [ ] **Availability**: 99.9% uptime
- [ ] **Consistency**: Messages delivered in order
- [ ] **Security**: End-to-end encryption

### Scale Estimates
- **Users**: 10M daily active users
- **Messages**: 1B per day = 11.5K per second
- **Connections**: 1M concurrent connections
- **Storage**: 1B messages * 100 bytes = 100GB per day

## 🏗️ High-Level Design

```
[Mobile/Web Client] ↔ [Load Balancer] ↔ [Chat Service] ↔ [Message Queue]
                                           ↓
                                      [WebSocket Manager] 
                                           ↓
                                      [User Database]
                                      [Message Database]
                                      [Media Storage]
```

### Core Components
1. **Chat Service**: Handle message routing
2. **WebSocket Manager**: Maintain persistent connections
3. **Message Queue**: Reliable message delivery
4. **Notification Service**: Push notifications
5. **User Service**: User management, presence
6. **Media Service**: File upload/download

## 🔍 Detailed Design

### API Design
```python
# WebSocket Events
{
    "type": "send_message",
    "data": {
        "recipient_id": "user123",
        "content": "Hello!",
        "message_type": "text",
        "timestamp": 1641234567
    }
}

{
    "type": "receive_message", 
    "data": {
        "message_id": "msg456",
        "sender_id": "user789",
        "content": "Hi there!",
        "timestamp": 1641234567
    }
}

# REST APIs
POST /api/v1/chats
GET /api/v1/chats/{chat_id}/messages
POST /api/v1/chats/{chat_id}/messages
PUT /api/v1/users/{user_id}/status
```

### Database Schema
```sql
-- Users table
CREATE TABLE users (
    user_id VARCHAR(36) PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    email VARCHAR(100),
    last_seen TIMESTAMP,
    status ENUM('online', 'offline', 'away'),
    created_at TIMESTAMP
);

-- Chats table (1-on-1 and groups)
CREATE TABLE chats (
    chat_id VARCHAR(36) PRIMARY KEY,
    chat_type ENUM('direct', 'group'),
    chat_name VARCHAR(100), -- for groups
    created_by VARCHAR(36),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Chat participants
CREATE TABLE chat_participants (
    chat_id VARCHAR(36),
    user_id VARCHAR(36),
    joined_at TIMESTAMP,
    role ENUM('admin', 'member'),
    PRIMARY KEY (chat_id, user_id)
);

-- Messages table
CREATE TABLE messages (
    message_id VARCHAR(36) PRIMARY KEY,
    chat_id VARCHAR(36),
    sender_id VARCHAR(36),
    content TEXT,
    message_type ENUM('text', 'image', 'file', 'system'),
    file_url VARCHAR(500),
    timestamp TIMESTAMP,
    INDEX(chat_id, timestamp),
    INDEX(sender_id)
);

-- Message status tracking
CREATE TABLE message_status (
    message_id VARCHAR(36),
    user_id VARCHAR(36),
    status ENUM('sent', 'delivered', 'read'),
    timestamp TIMESTAMP,
    PRIMARY KEY (message_id, user_id)
);
```

### Real-time Communication

#### **WebSocket Connection Management**
```python
class WebSocketManager:
    def __init__(self):
        self.connections = {}  # user_id -> websocket
        self.user_servers = {}  # user_id -> server_id
    
    async def connect(self, user_id, websocket):
        # Store connection
        self.connections[user_id] = websocket
        self.user_servers[user_id] = self.server_id
        
        # Update user status
        await self.update_user_status(user_id, 'online')
        
        # Notify friends about online status
        await self.broadcast_status_update(user_id, 'online')
    
    async def disconnect(self, user_id):
        # Clean up connection
        del self.connections[user_id]
        del self.user_servers[user_id]
        
        # Update status
        await self.update_user_status(user_id, 'offline')
        await self.broadcast_status_update(user_id, 'offline')
    
    async def send_message(self, recipient_id, message):
        if recipient_id in self.connections:
            # User connected to this server
            await self.connections[recipient_id].send_text(message)
        else:
            # User on different server - use message queue
            await self.publish_to_queue(recipient_id, message)
```

#### **Message Flow**
```python
async def handle_send_message(user_id, data):
    # 1. Validate message
    if not validate_message(data):
        return {"error": "Invalid message"}
    
    # 2. Save to database
    message = {
        "message_id": generate_uuid(),
        "chat_id": data["chat_id"],
        "sender_id": user_id,
        "content": data["content"],
        "timestamp": datetime.utcnow()
    }
    await save_message(message)
    
    # 3. Get chat participants
    participants = await get_chat_participants(data["chat_id"])
    
    # 4. Send to online users
    for participant in participants:
        if participant != user_id:
            await send_to_user(participant, message)
    
    # 5. Send push notifications to offline users
    offline_users = await get_offline_participants(participants)
    await send_push_notifications(offline_users, message)
    
    return {"status": "sent", "message_id": message["message_id"]}
```

## 📈 Scalability Solutions

### **Horizontal Scaling**
```python
# Chat Service Scaling
class ChatServiceCluster:
    def __init__(self):
        self.servers = []  # List of chat server instances
        self.load_balancer = ConsistentHashingLB()
    
    def route_user(self, user_id):
        # Consistent hashing for connection stickiness
        server = self.load_balancer.get_server(user_id)
        return server
    
    def route_message(self, recipient_id, message):
        # Find which server has the user's connection
        target_server = self.get_user_server(recipient_id)
        if target_server:
            target_server.send_message(recipient_id, message)
        else:
            # User offline - store for delivery
            self.store_offline_message(recipient_id, message)
```

### **Database Sharding**
```python
# Shard messages by chat_id
def get_message_shard(chat_id):
    shard_id = hash(chat_id) % NUM_SHARDS
    return f"messages_shard_{shard_id}"

# Shard users by user_id  
def get_user_shard(user_id):
    shard_id = hash(user_id) % NUM_SHARDS
    return f"users_shard_{shard_id}"
```

### **Message Queue Architecture**
```python
# Use message queue for inter-server communication
class MessageBroker:
    def __init__(self):
        self.kafka_producer = KafkaProducer()
        self.kafka_consumer = KafkaConsumer()
    
    async def publish_message(self, topic, message):
        # Topic: user_messages_{user_id}
        await self.kafka_producer.send(topic, message)
    
    async def consume_messages(self, user_id):
        topic = f"user_messages_{user_id}"
        async for message in self.kafka_consumer.subscribe(topic):
            await self.deliver_message(user_id, message)
```

## 🔒 Security & Privacy

### **End-to-End Encryption**
```python
# Signal Protocol implementation
class E2EEncryption:
    def __init__(self):
        self.key_server = KeyServer()
    
    def encrypt_message(self, message, recipient_public_key):
        # Generate session key
        session_key = generate_random_key()
        
        # Encrypt message with session key
        encrypted_message = AES.encrypt(message, session_key)
        
        # Encrypt session key with recipient's public key
        encrypted_session_key = RSA.encrypt(session_key, recipient_public_key)
        
        return {
            "encrypted_message": encrypted_message,
            "encrypted_key": encrypted_session_key
        }
    
    def decrypt_message(self, encrypted_data, private_key):
        # Decrypt session key
        session_key = RSA.decrypt(encrypted_data["encrypted_key"], private_key)
        
        # Decrypt message
        message = AES.decrypt(encrypted_data["encrypted_message"], session_key)
        return message
```

### **Authentication & Authorization**
```python
# JWT-based authentication
def authenticate_websocket(token):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
        user_id = payload['user_id']
        return user_id
    except jwt.InvalidTokenError:
        raise AuthenticationError("Invalid token")

# Rate limiting
class RateLimiter:
    def __init__(self):
        self.redis = redis.Redis()
    
    def check_rate_limit(self, user_id, limit=100, window=60):
        key = f"rate_limit:{user_id}"
        current = self.redis.incr(key)
        if current == 1:
            self.redis.expire(key, window)
        return current <= limit
```

## 📱 Push Notifications

```python
class NotificationService:
    def __init__(self):
        self.fcm = FCMClient()
        self.apns = APNSClient()
    
    async def send_push_notification(self, user_id, message):
        # Get user's device tokens
        devices = await get_user_devices(user_id)
        
        notification = {
            "title": message["sender_name"],
            "body": message["content"][:50] + "...",
            "data": {
                "chat_id": message["chat_id"],
                "message_id": message["message_id"]
            }
        }
        
        for device in devices:
            if device["platform"] == "android":
                await self.fcm.send(device["token"], notification)
            elif device["platform"] == "ios":
                await self.apns.send(device["token"], notification)
```

## 📊 Monitoring & Metrics

### **Key Metrics**
```python
# Message delivery metrics
message_sent_count = Counter('messages_sent_total')
message_delivery_latency = Histogram('message_delivery_seconds')
connection_count = Gauge('websocket_connections_active')
message_queue_size = Gauge('message_queue_size')

# System health metrics
cpu_usage = Gauge('cpu_usage_percent')
memory_usage = Gauge('memory_usage_percent')
database_connection_pool = Gauge('db_connections_active')
```

## 🎯 Trade-offs

| Decision | Pros | Cons |
|----------|------|------|
| WebSocket vs Polling | Real-time, efficient | Complex, connection management |
| Message Queue vs Direct | Reliable, scalable | Added latency, complexity |
| E2E Encryption | Security, privacy | Performance overhead, key management |
| Sharding vs Replication | Horizontal scaling | Cross-shard queries difficult |

## 🔗 Related Topics

- [[WebSocket]] - Real-time communication
- [[Message Queue]] - Kafka, RabbitMQ
- [[Database Sharding]] - Horizontal scaling
- [[Load Balancer]] - Connection distribution
- [[Push Notifications]] - Mobile notifications

## 📚 Further Reading

- WhatsApp system architecture blog posts
- Signal Protocol specification
- Apache Kafka for real-time messaging
- WebSocket scaling patterns

---

**Next Steps**: 
- [ ] Implement basic WebSocket chat
- [ ] Add group chat functionality
- [ ] Design offline message delivery
- [ ] Implement end-to-end encryption 