# Message Queue Systems

**Tags**: #system-components #message-queue #async #communication
**Date**: 2024-01-01

## 📝 Overview

Message queues là backbone của distributed systems, cho phép asynchronous communication giữa services. Chúng decouple producers và consumers, cung cấp reliability và scalability.

## 🎯 Core Concepts

### **Message Queue Pattern**
```
Producer -> Message Queue -> Consumer

Benefits:
- Decoupling: Producer không cần biết consumer
- Reliability: Messages persist if consumer offline
- Scalability: Multiple consumers process messages
- Load Balancing: Work distributed across consumers
```

### **Key Components**
```python
# Message structure
class Message:
    def __init__(self, id, payload, headers=None, timestamp=None):
        self.id = id
        self.payload = payload
        self.headers = headers or {}
        self.timestamp = timestamp or time.time()
        self.retry_count = 0
        self.delivery_tag = None

# Queue interface
class MessageQueue:
    def publish(self, message, routing_key=None):
        pass
    
    def consume(self, queue_name, callback):
        pass
    
    def acknowledge(self, delivery_tag):
        pass
    
    def reject(self, delivery_tag, requeue=True):
        pass
```

## 🔄 Queue Types & Patterns

### **1. Point-to-Point (P2P)**
```python
# Single consumer receives each message
class PointToPointQueue:
    def __init__(self):
        self.queue = deque()
        self.lock = threading.Lock()
    
    def send(self, message):
        with self.lock:
            self.queue.append(message)
    
    def receive(self):
        with self.lock:
            return self.queue.popleft() if self.queue else None

# Example: Order processing
def process_orders():
    order_queue = PointToPointQueue()
    
    # Producer
    order_queue.send({"order_id": 123, "amount": 99.99})
    
    # Single consumer processes
    order = order_queue.receive()
    if order:
        process_payment(order)
```

### **2. Publish-Subscribe (Pub/Sub)**
```python
# Multiple subscribers receive same message
class PubSubQueue:
    def __init__(self):
        self.subscribers = defaultdict(list)
        self.messages = defaultdict(list)
    
    def subscribe(self, topic, callback):
        self.subscribers[topic].append(callback)
    
    def publish(self, topic, message):
        # Deliver to all subscribers
        for callback in self.subscribers[topic]:
            callback(message)

# Example: User activity notifications
pubsub = PubSubQueue()

# Multiple services subscribe
pubsub.subscribe("user_registered", send_welcome_email)
pubsub.subscribe("user_registered", create_user_profile)
pubsub.subscribe("user_registered", track_analytics)

# Single publish triggers all
pubsub.publish("user_registered", {"user_id": 123, "email": "user@example.com"})
```

### **3. Work Queues (Task Distribution)**
```python
# Multiple workers process tasks in parallel
class WorkQueue:
    def __init__(self, num_workers=4):
        self.queue = Queue()
        self.workers = []
        self.start_workers(num_workers)
    
    def start_workers(self, num_workers):
        for i in range(num_workers):
            worker = threading.Thread(target=self.worker_loop)
            worker.daemon = True
            worker.start()
            self.workers.append(worker)
    
    def worker_loop(self):
        while True:
            task = self.queue.get()
            try:
                task['function'](*task['args'])
            except Exception as e:
                logger.error(f"Task failed: {e}")
            finally:
                self.queue.task_done()
    
    def submit_task(self, function, *args):
        self.queue.put({'function': function, 'args': args})

# Example: Image processing
work_queue = WorkQueue(num_workers=8)

for image_path in image_paths:
    work_queue.submit_task(resize_image, image_path, "200x200")
```

## 🚀 Popular Message Queue Systems

### **RabbitMQ (AMQP)**
```python
import pika

class RabbitMQClient:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host=host)
        )
        self.channel = self.connection.channel()
    
    def create_queue(self, queue_name, durable=True):
        self.channel.queue_declare(queue=queue_name, durable=durable)
    
    def publish_message(self, queue_name, message):
        self.channel.basic_publish(
            exchange='',
            routing_key=queue_name,
            body=json.dumps(message),
            properties=pika.BasicProperties(delivery_mode=2)  # Persistent
        )
    
    def consume_messages(self, queue_name, callback):
        def wrapper(ch, method, properties, body):
            try:
                message = json.loads(body)
                callback(message)
                ch.basic_ack(delivery_tag=method.delivery_tag)
            except Exception as e:
                logger.error(f"Processing failed: {e}")
                ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
        
        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=wrapper
        )
        self.channel.start_consuming()

# Usage example
rabbitmq = RabbitMQClient()
rabbitmq.create_queue('email_queue')

# Producer
rabbitmq.publish_message('email_queue', {
    'to': 'user@example.com',
    'subject': 'Welcome!',
    'body': 'Thank you for signing up'
})

# Consumer
def process_email(message):
    send_email(message['to'], message['subject'], message['body'])

rabbitmq.consume_messages('email_queue', process_email)
```

### **Apache Kafka (Event Streaming)**
```python
from kafka import KafkaProducer, KafkaConsumer

class KafkaClient:
    def __init__(self, bootstrap_servers=['localhost:9092']):
        self.bootstrap_servers = bootstrap_servers
    
    def create_producer(self):
        return KafkaProducer(
            bootstrap_servers=self.bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            acks='all',  # Wait for all replicas
            retries=3
        )
    
    def create_consumer(self, topics, group_id):
        return KafkaConsumer(
            *topics,
            bootstrap_servers=self.bootstrap_servers,
            group_id=group_id,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest'
        )

# High-throughput event processing
kafka_client = KafkaClient()

# Producer: Track user events
producer = kafka_client.create_producer()

def track_user_event(user_id, event_type, data):
    event = {
        'user_id': user_id,
        'event_type': event_type,
        'data': data,
        'timestamp': time.time()
    }
    producer.send('user_events', value=event)

# Consumer: Process events for analytics
consumer = kafka_client.create_consumer(['user_events'], 'analytics_group')

for message in consumer:
    event = message.value
    update_user_analytics(event)
```

### **Amazon SQS (Managed Service)**
```python
import boto3

class SQSClient:
    def __init__(self):
        self.sqs = boto3.client('sqs')
    
    def send_message(self, queue_url, message_body, delay_seconds=0):
        response = self.sqs.send_message(
            QueueUrl=queue_url,
            MessageBody=json.dumps(message_body),
            DelaySeconds=delay_seconds
        )
        return response['MessageId']
    
    def receive_messages(self, queue_url, max_messages=10):
        response = self.sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=max_messages,
            WaitTimeSeconds=20  # Long polling
        )
        return response.get('Messages', [])
    
    def delete_message(self, queue_url, receipt_handle):
        self.sqs.delete_message(
            QueueUrl=queue_url,
            ReceiptHandle=receipt_handle
        )

# Serverless task processing
sqs_client = SQSClient()
queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789/task-queue'

# Send task
task = {'action': 'resize_image', 'image_url': 'http://example.com/image.jpg'}
message_id = sqs_client.send_message(queue_url, task)

# Process tasks
messages = sqs_client.receive_messages(queue_url)
for message in messages:
    task = json.loads(message['Body'])
    process_task(task)
    sqs_client.delete_message(queue_url, message['ReceiptHandle'])
```

## 🛡️ Reliability Patterns

### **Message Acknowledgment**
```python
class ReliableMessageProcessor:
    def __init__(self, queue_client):
        self.queue_client = queue_client
        self.processing_timeout = 300  # 5 minutes
    
    def process_with_ack(self, queue_name):
        message = self.queue_client.receive_message(queue_name)
        if not message:
            return
        
        try:
            # Process message with timeout
            with timeout(self.processing_timeout):
                result = self.process_message(message.payload)
            
            # Acknowledge successful processing
            self.queue_client.ack_message(message.delivery_tag)
            return result
            
        except TimeoutError:
            # Reject and requeue for retry
            self.queue_client.nack_message(message.delivery_tag, requeue=True)
            raise ProcessingTimeoutError()
            
        except Exception as e:
            # Decide whether to retry or send to dead letter queue
            message.retry_count += 1
            if message.retry_count < 3:
                self.queue_client.nack_message(message.delivery_tag, requeue=True)
            else:
                self.send_to_dead_letter_queue(message)
                self.queue_client.ack_message(message.delivery_tag)
            raise
```

### **Dead Letter Queues**
```python
class DeadLetterQueueHandler:
    def __init__(self, main_queue, dlq_queue):
        self.main_queue = main_queue
        self.dlq_queue = dlq_queue
    
    def process_with_dlq(self, message):
        max_retries = 3
        retry_count = getattr(message, 'retry_count', 0)
        
        try:
            return self.process_message(message)
        except Exception as e:
            if retry_count >= max_retries:
                # Send to dead letter queue for manual inspection
                dlq_message = {
                    'original_message': message.payload,
                    'error': str(e),
                    'retry_count': retry_count,
                    'failed_at': time.time()
                }
                self.dlq_queue.send(dlq_message)
                return None
            else:
                # Retry with exponential backoff
                delay = 2 ** retry_count  # 1s, 2s, 4s
                message.retry_count = retry_count + 1
                self.main_queue.send(message, delay=delay)
                raise
```

### **Idempotent Processing**
```python
class IdempotentProcessor:
    def __init__(self, cache_client):
        self.cache = cache_client
        self.processed_messages = set()
    
    def process_idempotent(self, message):
        message_id = message.id
        
        # Check if already processed
        if self.cache.exists(f"processed:{message_id}"):
            logger.info(f"Message {message_id} already processed, skipping")
            return self.cache.get(f"result:{message_id}")
        
        try:
            # Process message
            result = self.process_message(message.payload)
            
            # Store result and mark as processed
            self.cache.set(f"result:{message_id}", result, ttl=3600)
            self.cache.set(f"processed:{message_id}", True, ttl=3600)
            
            return result
            
        except Exception as e:
            # Don't mark as processed if failed
            raise
```

## 📊 Performance Considerations

### **Batching for Throughput**
```python
class BatchMessageProcessor:
    def __init__(self, queue_client, batch_size=100):
        self.queue_client = queue_client
        self.batch_size = batch_size
        self.batch = []
    
    def process_batch(self):
        messages = self.queue_client.receive_messages_batch(
            count=self.batch_size
        )
        
        if not messages:
            return
        
        try:
            # Process entire batch together
            results = self.process_messages_batch([m.payload for m in messages])
            
            # Acknowledge all messages
            for message in messages:
                self.queue_client.ack_message(message.delivery_tag)
                
            return results
            
        except Exception as e:
            # Handle batch failure
            for message in messages:
                self.queue_client.nack_message(message.delivery_tag, requeue=True)
            raise
```

### **Backpressure Handling**
```python
class BackpressureAwareConsumer:
    def __init__(self, queue_client, max_processing=10):
        self.queue_client = queue_client
        self.semaphore = asyncio.Semaphore(max_processing)
        self.processing_count = 0
    
    async def consume_with_backpressure(self):
        while True:
            # Check if can accept more work
            if self.processing_count >= self.semaphore._value:
                await asyncio.sleep(0.1)  # Back off
                continue
            
            message = await self.queue_client.receive_message_async()
            if message:
                # Process with concurrency limit
                asyncio.create_task(self.process_message_async(message))
    
    async def process_message_async(self, message):
        async with self.semaphore:
            self.processing_count += 1
            try:
                await self.process_message(message.payload)
                await self.queue_client.ack_message_async(message.delivery_tag)
            except Exception as e:
                await self.queue_client.nack_message_async(
                    message.delivery_tag, requeue=True
                )
            finally:
                self.processing_count -= 1
```

## 🔗 Related Topics

- [[Microservices Architecture]] - Inter-service communication
- [[Event-Driven Architecture]] - Asynchronous system design
- [[Load Balancing]] - Distributing work across consumers
- [[Caching Strategies]] - Message caching and deduplication

## 📚 Further Reading

- RabbitMQ documentation and patterns
- Apache Kafka design principles
- AWS SQS best practices
- Enterprise Integration Patterns book

---

**Key Takeaway**: Message queues enable scalable, reliable communication trong distributed systems. Choose the right pattern (P2P, Pub/Sub, Work Queue) và implementation based on your consistency, throughput và durability requirements. 