# Performance Monitoring

**Tags**: #performance #monitoring #observability #metrics
**Date**: 2024-01-01

## 📝 Overview

Performance monitoring là critical component của distributed systems, cho phép teams detect issues, optimize performance, và maintain system reliability. Comprehensive monitoring strategy bao gồm metrics, logging, tracing, và alerting.

## 🎯 Monitoring Pillars

### **Three Pillars of Observability**
```
1. Metrics - Numerical data over time
   - CPU usage, memory, request rate, latency
   - Aggregated and queryable

2. Logging - Discrete events with context
   - Error messages, debug information
   - Searchable and filterable

3. Tracing - Request flow across services
   - End-to-end request path
   - Performance bottleneck identification
```

### **Key Performance Indicators (KPIs)**
```python
class PerformanceMetrics:
    def __init__(self):
        self.golden_signals = {
            "latency": "How long requests take",
            "traffic": "How many requests per second", 
            "errors": "Rate of failed requests",
            "saturation": "How full your service is"
        }
        
        self.system_metrics = {
            "cpu_utilization": "Percentage of CPU used",
            "memory_usage": "RAM consumption",
            "disk_io": "Read/write operations",
            "network_throughput": "Data transfer rate"
        }
        
        self.application_metrics = {
            "response_time": "API response latency",
            "throughput": "Requests processed per second",
            "error_rate": "Failed requests percentage",
            "apdex": "Application Performance Index"
        }
```

## 📊 Metrics Collection

### **Application Metrics**
```python
import time
import threading
from collections import defaultdict, deque

class MetricsCollector:
    def __init__(self):
        self.counters = defaultdict(int)
        self.histograms = defaultdict(list)
        self.gauges = defaultdict(float)
        self.timers = defaultdict(list)
        self.lock = threading.Lock()
    
    def increment_counter(self, metric_name, value=1, tags=None):
        """Track cumulative events"""
        with self.lock:
            key = self._build_key(metric_name, tags)
            self.counters[key] += value
    
    def record_gauge(self, metric_name, value, tags=None):
        """Track current state/value"""
        with self.lock:
            key = self._build_key(metric_name, tags)
            self.gauges[key] = value
    
    def record_histogram(self, metric_name, value, tags=None):
        """Track distribution of values"""
        with self.lock:
            key = self._build_key(metric_name, tags)
            self.histograms[key].append(value)
            
            # Keep only recent values for memory efficiency
            if len(self.histograms[key]) > 1000:
                self.histograms[key] = self.histograms[key][-1000:]
    
    def time_function(self, metric_name, tags=None):
        """Decorator to time function execution"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                start_time = time.time()
                try:
                    result = func(*args, **kwargs)
                    self.increment_counter(f"{metric_name}.success", tags=tags)
                    return result
                except Exception as e:
                    self.increment_counter(f"{metric_name}.error", tags=tags)
                    raise
                finally:
                    duration = (time.time() - start_time) * 1000  # milliseconds
                    self.record_histogram(f"{metric_name}.duration", duration, tags)
            return wrapper
        return decorator
    
    def _build_key(self, metric_name, tags):
        if not tags:
            return metric_name
        tag_str = ",".join([f"{k}={v}" for k, v in sorted(tags.items())])
        return f"{metric_name},{tag_str}"

# Usage example
metrics = MetricsCollector()

@metrics.time_function("api.user.get", tags={"version": "v1"})
def get_user(user_id):
    # Simulate API call
    time.sleep(0.1)
    return {"id": user_id, "name": "John Doe"}

# Track business metrics
def process_order(order_data):
    metrics.increment_counter("orders.created", tags={
        "region": order_data["region"],
        "payment_method": order_data["payment_method"]
    })
    
    metrics.record_gauge("orders.total_value", order_data["total_amount"])
```

### **System Resource Monitoring**
```python
import psutil
import time

class SystemMonitor:
    def __init__(self, metrics_collector):
        self.metrics = metrics_collector
        self.running = False
        self.collection_interval = 10  # seconds
    
    def start_monitoring(self):
        """Start background monitoring thread"""
        self.running = True
        monitor_thread = threading.Thread(target=self._monitor_loop)
        monitor_thread.daemon = True
        monitor_thread.start()
    
    def stop_monitoring(self):
        self.running = False
    
    def _monitor_loop(self):
        while self.running:
            try:
                self._collect_system_metrics()
                time.sleep(self.collection_interval)
            except Exception as e:
                print(f"Error collecting metrics: {e}")
    
    def _collect_system_metrics(self):
        # CPU metrics
        cpu_percent = psutil.cpu_percent(interval=1)
        self.metrics.record_gauge("system.cpu.usage_percent", cpu_percent)
        
        cpu_count = psutil.cpu_count()
        self.metrics.record_gauge("system.cpu.count", cpu_count)
        
        # Memory metrics
        memory = psutil.virtual_memory()
        self.metrics.record_gauge("system.memory.usage_percent", memory.percent)
        self.metrics.record_gauge("system.memory.available_bytes", memory.available)
        self.metrics.record_gauge("system.memory.used_bytes", memory.used)
        
        # Disk metrics
        for disk in psutil.disk_partitions():
            try:
                disk_usage = psutil.disk_usage(disk.mountpoint)
                tags = {"device": disk.device, "mountpoint": disk.mountpoint}
                
                self.metrics.record_gauge("system.disk.usage_percent", 
                                        disk_usage.percent, tags)
                self.metrics.record_gauge("system.disk.free_bytes", 
                                        disk_usage.free, tags)
            except PermissionError:
                continue
        
        # Network metrics
        network = psutil.net_io_counters()
        self.metrics.record_gauge("system.network.bytes_sent", network.bytes_sent)
        self.metrics.record_gauge("system.network.bytes_recv", network.bytes_recv)
        
        # Process metrics
        process = psutil.Process()
        self.metrics.record_gauge("process.memory.rss", process.memory_info().rss)
        self.metrics.record_gauge("process.cpu.percent", process.cpu_percent())
        self.metrics.record_gauge("process.threads.count", process.num_threads())

# Database monitoring
class DatabaseMonitor:
    def __init__(self, db_connection, metrics_collector):
        self.db = db_connection
        self.metrics = metrics_collector
    
    def monitor_connection_pool(self):
        """Monitor database connection pool health"""
        pool_stats = self.db.get_pool_stats()
        
        self.metrics.record_gauge("db.connections.active", pool_stats["active"])
        self.metrics.record_gauge("db.connections.idle", pool_stats["idle"])
        self.metrics.record_gauge("db.connections.total", pool_stats["total"])
        self.metrics.record_gauge("db.connections.max", pool_stats["max_size"])
    
    def monitor_query_performance(self):
        """Monitor slow queries and performance"""
        slow_queries = self.db.get_slow_queries()
        
        for query in slow_queries:
            self.metrics.record_histogram("db.query.duration", 
                                         query["duration"], 
                                         tags={"query_type": query["type"]})
            
            if query["duration"] > 1000:  # > 1 second
                self.metrics.increment_counter("db.slow_queries", 
                                             tags={"table": query["table"]})
```

## 🔍 Distributed Tracing

### **Request Tracing Implementation**
```python
import uuid
import time
import json
from contextlib import contextmanager

class Span:
    def __init__(self, operation_name, trace_id=None, parent_span_id=None):
        self.span_id = str(uuid.uuid4())
        self.trace_id = trace_id or str(uuid.uuid4())
        self.parent_span_id = parent_span_id
        self.operation_name = operation_name
        self.start_time = time.time()
        self.end_time = None
        self.tags = {}
        self.logs = []
        self.status = "ok"
    
    def set_tag(self, key, value):
        self.tags[key] = value
    
    def log(self, message, level="info"):
        self.logs.append({
            "timestamp": time.time(),
            "level": level,
            "message": message
        })
    
    def finish(self):
        self.end_time = time.time()
    
    def duration_ms(self):
        end = self.end_time or time.time()
        return (end - self.start_time) * 1000
    
    def to_dict(self):
        return {
            "span_id": self.span_id,
            "trace_id": self.trace_id,
            "parent_span_id": self.parent_span_id,
            "operation_name": self.operation_name,
            "start_time": self.start_time,
            "end_time": self.end_time,
            "duration_ms": self.duration_ms(),
            "tags": self.tags,
            "logs": self.logs,
            "status": self.status
        }

class Tracer:
    def __init__(self):
        self.spans = []
        self.current_span = None
    
    @contextmanager
    def start_span(self, operation_name, parent_span=None):
        parent_span = parent_span or self.current_span
        
        span = Span(
            operation_name,
            trace_id=parent_span.trace_id if parent_span else None,
            parent_span_id=parent_span.span_id if parent_span else None
        )
        
        old_span = self.current_span
        self.current_span = span
        
        try:
            yield span
        except Exception as e:
            span.set_tag("error", True)
            span.log(f"Exception: {str(e)}", level="error")
            span.status = "error"
            raise
        finally:
            span.finish()
            self.spans.append(span)
            self.current_span = old_span
    
    def inject_headers(self, headers):
        """Inject tracing headers for cross-service calls"""
        if self.current_span:
            headers.update({
                "X-Trace-ID": self.current_span.trace_id,
                "X-Span-ID": self.current_span.span_id
            })
    
    def extract_headers(self, headers):
        """Extract tracing context from headers"""
        trace_id = headers.get("X-Trace-ID")
        parent_span_id = headers.get("X-Span-ID")
        
        if trace_id:
            # Create span with extracted context
            span = Span("extracted_span", trace_id, parent_span_id)
            self.current_span = span
            return span
        return None

# Example usage in microservices
tracer = Tracer()

def user_service_handler(user_id, request_headers):
    # Extract tracing context from incoming request
    tracer.extract_headers(request_headers)
    
    with tracer.start_span("user_service.get_user") as span:
        span.set_tag("user_id", user_id)
        span.set_tag("service", "user-service")
        
        # Simulate database call
        with tracer.start_span("database.query") as db_span:
            db_span.set_tag("query", "SELECT * FROM users WHERE id = ?")
            db_span.set_tag("table", "users")
            
            # Simulate query time
            time.sleep(0.05)
            user_data = {"id": user_id, "name": "John Doe"}
        
        # Call another service
        headers = {}
        tracer.inject_headers(headers)
        
        with tracer.start_span("http.request") as http_span:
            http_span.set_tag("url", "http://profile-service/users/{user_id}/profile")
            http_span.set_tag("method", "GET")
            
            # Simulate HTTP call
            time.sleep(0.1)
            profile_data = call_profile_service(user_id, headers)
        
        return {"user": user_data, "profile": profile_data}

def call_profile_service(user_id, headers):
    # This would be actual HTTP call to another service
    # Headers contain tracing information
    return {"preferences": {"theme": "dark"}}
```

### **APM Integration**
```python
# Integration with popular APM tools

# New Relic integration
import newrelic.agent

@newrelic.agent.function_trace()
def process_payment(order_data):
    newrelic.agent.add_custom_attribute("order_id", order_data["order_id"])
    newrelic.agent.add_custom_attribute("amount", order_data["amount"])
    
    # Process payment logic
    result = payment_processor.charge(order_data)
    
    if result["success"]:
        newrelic.agent.record_custom_event("PaymentSuccess", {
            "order_id": order_data["order_id"],
            "amount": order_data["amount"],
            "processing_time": result["processing_time"]
        })
    else:
        newrelic.agent.notice_error()
    
    return result

# DataDog integration
from datadog import DogStatsd

statsd = DogStatsd(host="localhost", port=8125)

def track_user_action(user_id, action):
    # Increment counter
    statsd.increment("user.actions", 
                    tags=[f"action:{action}", f"user_id:{user_id}"])
    
    # Time the action
    with statsd.timed("user.action.duration", tags=[f"action:{action}"]):
        # Perform action
        result = perform_action(action)
    
    return result

# Prometheus integration
from prometheus_client import Counter, Histogram, Gauge, start_http_server

# Define metrics
REQUEST_COUNT = Counter('http_requests_total', 
                       'Total HTTP requests', 
                       ['method', 'endpoint', 'status'])

REQUEST_LATENCY = Histogram('http_request_duration_seconds',
                           'HTTP request latency',
                           ['method', 'endpoint'])

ACTIVE_CONNECTIONS = Gauge('active_connections',
                          'Active database connections')

def monitor_endpoint(method, endpoint):
    def decorator(func):
        def wrapper(*args, **kwargs):
            with REQUEST_LATENCY.labels(method=method, endpoint=endpoint).time():
                try:
                    result = func(*args, **kwargs)
                    REQUEST_COUNT.labels(method=method, endpoint=endpoint, status="200").inc()
                    return result
                except Exception as e:
                    REQUEST_COUNT.labels(method=method, endpoint=endpoint, status="500").inc()
                    raise
        return wrapper
    return decorator

# Start Prometheus metrics server
start_http_server(8000)
```

## 🚨 Alerting & Anomaly Detection

### **Alert Rules Configuration**
```python
class AlertRule:
    def __init__(self, name, metric, condition, threshold, severity="warning"):
        self.name = name
        self.metric = metric
        self.condition = condition  # "greater_than", "less_than", "equals"
        self.threshold = threshold
        self.severity = severity
        self.cooldown_period = 300  # 5 minutes
        self.last_triggered = None
    
    def evaluate(self, metric_value):
        should_alert = False
        
        if self.condition == "greater_than" and metric_value > self.threshold:
            should_alert = True
        elif self.condition == "less_than" and metric_value < self.threshold:
            should_alert = True
        elif self.condition == "equals" and metric_value == self.threshold:
            should_alert = True
        
        # Check cooldown period
        if should_alert and self.last_triggered:
            time_since_last = time.time() - self.last_triggered
            if time_since_last < self.cooldown_period:
                should_alert = False
        
        if should_alert:
            self.last_triggered = time.time()
        
        return should_alert

class AlertManager:
    def __init__(self):
        self.rules = []
        self.notification_channels = []
    
    def add_rule(self, rule):
        self.rules.append(rule)
    
    def add_notification_channel(self, channel):
        self.notification_channels.append(channel)
    
    def evaluate_metrics(self, metrics_data):
        triggered_alerts = []
        
        for rule in self.rules:
            metric_value = metrics_data.get(rule.metric)
            if metric_value is not None and rule.evaluate(metric_value):
                alert = {
                    "rule_name": rule.name,
                    "metric": rule.metric,
                    "current_value": metric_value,
                    "threshold": rule.threshold,
                    "severity": rule.severity,
                    "timestamp": time.time()
                }
                triggered_alerts.append(alert)
        
        # Send notifications
        for alert in triggered_alerts:
            self.send_alert(alert)
        
        return triggered_alerts
    
    def send_alert(self, alert):
        for channel in self.notification_channels:
            channel.send(alert)

# Example alert setup
alert_manager = AlertManager()

# Define alert rules
cpu_alert = AlertRule("High CPU Usage", "system.cpu.usage_percent", 
                     "greater_than", 80, "critical")
memory_alert = AlertRule("High Memory Usage", "system.memory.usage_percent", 
                        "greater_than", 85, "warning")
error_rate_alert = AlertRule("High Error Rate", "api.error_rate", 
                           "greater_than", 5, "critical")

alert_manager.add_rule(cpu_alert)
alert_manager.add_rule(memory_alert)
alert_manager.add_rule(error_rate_alert)

# Notification channels
class SlackNotification:
    def __init__(self, webhook_url):
        self.webhook_url = webhook_url
    
    def send(self, alert):
        message = f"""
        🚨 ALERT: {alert['rule_name']}
        Metric: {alert['metric']}
        Current Value: {alert['current_value']}
        Threshold: {alert['threshold']}
        Severity: {alert['severity']}
        """
        # Send to Slack webhook
        self.post_to_slack(message)

alert_manager.add_notification_channel(SlackNotification("https://hooks.slack.com/..."))
```

## 🔗 Related Topics

- [[System Architecture Patterns]] - Designing monitorable systems
- [[Load Balancing]] - Monitoring load balancer performance
- [[Database Design]] - Database performance monitoring
- [[Microservices Architecture]] - Service-level monitoring

## 📚 Further Reading

- "Site Reliability Engineering" by Google
- Prometheus monitoring best practices
- OpenTelemetry specification
- Grafana dashboard design principles

---

**Key Takeaway**: Effective performance monitoring requires comprehensive coverage of metrics, logs, và traces. Implement proactive alerting và use standardized observability tools để maintain system reliability và performance. 