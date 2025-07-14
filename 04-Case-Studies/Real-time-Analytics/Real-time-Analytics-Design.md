# Real-time Analytics System Design

**Tags**: #case-study #analytics #real-time #streaming #big-data
**Date**: 2024-01-01
**Difficulty**: Hard

## 📝 Problem Statement

Thiết kế một real-time analytics system như Google Analytics, Netflix Analytics, hoặc Uber's data platform có thể:
- Ingest millions of events per second
- Process và analyze data in real-time
- Provide dashboards với sub-second latency
- Support complex queries và aggregations
- Scale horizontally to handle growing data volume

## 🎯 Requirements Clarification

### Functional Requirements
- [ ] **Event Ingestion**: Collect events from multiple sources (web, mobile, IoT)
- [ ] **Real-time Processing**: Process events với latency <1 second
- [ ] **Aggregations**: Count, sum, average, percentiles, unique counts
- [ ] **Dashboards**: Real-time visualizations và metrics
- [ ] **Alerting**: Threshold-based alerts on metrics
- [ ] **Querying**: Ad-hoc queries on historical data
- [ ] **Multi-dimensional Analysis**: Filter by dimensions (time, geography, user segments)

### Non-functional Requirements
- [ ] **Scalability**: Handle 10M events/second
- [ ] **Latency**: <1 second end-to-end processing
- [ ] **Availability**: 99.9% uptime
- [ ] **Durability**: No data loss
- [ ] **Query Performance**: <100ms for dashboard queries
- [ ] **Cost Efficiency**: Optimize for cost per event processed

### Scale Estimates
- **Events**: 10M per second = 864B events per day
- **Event Size**: Average 1KB per event
- **Data Volume**: 864TB per day, 315PB per year
- **Users**: 10K concurrent dashboard users
- **Queries**: 100K queries per second
- **Retention**: 1 year hot data, 5 years cold data

## 🏗️ High-Level Architecture

```
[Data Sources] → [Event Ingestion] → [Stream Processing] → [Real-time Store]
                                                        → [Batch Processing] → [Historical Store]
                                                        → [Alerting System]
     ↓
[Dashboards] ← [Query Service] ← [Cache Layer] ← [Aggregation Engine]
```

### Core Components
1. **Event Ingestion**: Kafka, Kinesis cho high-throughput data collection
2. **Stream Processing**: Apache Flink, Storm cho real-time processing
3. **Real-time Store**: Apache Druid, ClickHouse cho fast analytics
4. **Batch Processing**: Apache Spark cho complex analytics
5. **Historical Store**: HDFS, S3 cho long-term storage
6. **Query Service**: Presto, Trino cho unified querying
7. **Cache Layer**: Redis, Memcached cho fast dashboard queries

## 🔍 Detailed Design

### Event Schema Design
```json
{
  "event_id": "uuid",
  "timestamp": "2024-01-01T10:00:00.000Z",
  "event_type": "page_view",
  "user_id": "user_123",
  "session_id": "session_456",
  "properties": {
    "page_url": "https://example.com/product/123",
    "page_title": "Product Page",
    "referrer": "https://google.com",
    "user_agent": "Mozilla/5.0...",
    "ip_address": "192.168.1.1",
    "country": "US",
    "device_type": "desktop"
  },
  "custom_properties": {
    "product_id": "123",
    "category": "electronics",
    "price": 99.99
  }
}
```

### Data Ingestion Pipeline
```python
class EventIngestionService:
    def __init__(self):
        self.kafka_producer = KafkaProducer()
        self.schema_registry = SchemaRegistry()
        self.validator = EventValidator()
        self.rate_limiter = RateLimiter()
    
    async def ingest_event(self, event_data):
        """Ingest single event"""
        
        # 1. Rate limiting
        if not await self.rate_limiter.allow_request(event_data.get('source')):
            raise RateLimitExceededException()
        
        # 2. Validate event schema
        if not self.validator.validate(event_data):
            raise InvalidEventSchemaException()
        
        # 3. Enrich event
        enriched_event = await self.enrich_event(event_data)
        
        # 4. Partition event
        partition_key = self.get_partition_key(enriched_event)
        
        # 5. Send to Kafka
        await self.kafka_producer.send(
            topic='events',
            key=partition_key,
            value=enriched_event,
            timestamp=enriched_event['timestamp']
        )
        
        return {'status': 'success', 'event_id': enriched_event['event_id']}
    
    async def ingest_batch(self, events):
        """Ingest batch of events"""
        
        # Process events in parallel
        tasks = []
        for event in events:
            task = asyncio.create_task(self.ingest_event(event))
            tasks.append(task)
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Collect results
        successful = sum(1 for r in results if not isinstance(r, Exception))
        failed = len(results) - successful
        
        return {
            'total': len(events),
            'successful': successful,
            'failed': failed
        }
    
    async def enrich_event(self, event):
        """Enrich event with additional data"""
        
        # IP geolocation
        if 'ip_address' in event.get('properties', {}):
            geo_data = await self.get_geo_location(event['properties']['ip_address'])
            event['properties'].update(geo_data)
        
        # User agent parsing
        if 'user_agent' in event.get('properties', {}):
            device_info = self.parse_user_agent(event['properties']['user_agent'])
            event['properties'].update(device_info)
        
        # Add processing timestamp
        event['processed_at'] = datetime.utcnow().isoformat()
        
        return event
    
    def get_partition_key(self, event):
        """Generate partition key for event distribution"""
        # Partition by user_id for session affinity
        if 'user_id' in event:
            return event['user_id']
        # Fallback to event_type for load balancing
        return event.get('event_type', 'unknown')

class EventValidator:
    def __init__(self):
        self.required_fields = ['event_id', 'timestamp', 'event_type']
        self.event_schemas = self.load_event_schemas()
    
    def validate(self, event):
        """Validate event against schema"""
        
        # Check required fields
        for field in self.required_fields:
            if field not in event:
                return False
        
        # Validate timestamp format
        try:
            datetime.fromisoformat(event['timestamp'].replace('Z', '+00:00'))
        except ValueError:
            return False
        
        # Validate against event-specific schema
        event_type = event.get('event_type')
        if event_type in self.event_schemas:
            return self.validate_against_schema(event, self.event_schemas[event_type])
        
        return True
    
    def validate_against_schema(self, event, schema):
        """Validate event against specific schema"""
        # Use jsonschema library for validation
        import jsonschema
        
        try:
            jsonschema.validate(event, schema)
            return True
        except jsonschema.exceptions.ValidationError:
            return False
```

### Stream Processing Engine
```python
class StreamProcessingEngine:
    def __init__(self):
        self.flink_client = FlinkClient()
        self.processing_jobs = {}
        self.metrics_collector = MetricsCollector()
    
    def create_real_time_aggregation_job(self, job_config):
        """Create real-time aggregation job"""
        
        job_code = f"""
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        // Configure checkpointing
        env.enableCheckpointing(5000); // 5 second checkpoints
        env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
        
        // Read from Kafka
        FlinkKafkaConsumer<Event> consumer = new FlinkKafkaConsumer<>(
            "events", 
            new EventDeserializationSchema(), 
            properties
        );
        
        DataStream<Event> events = env.addSource(consumer);
        
        // Process events
        DataStream<AggregatedMetric> aggregated = events
            .keyBy(event -> event.getEventType())
            .timeWindow(Time.seconds({job_config.window_size}))
            .aggregate(new {job_config.aggregation_function}());
        
        // Sink to real-time store
        aggregated.addSink(new DruidSink<>());
        
        env.execute("Real-time Aggregation Job");
        """
        
        job = self.flink_client.submit_job(job_code, job_config)
        self.processing_jobs[job.job_id] = job
        
        return job
    
    def create_session_analysis_job(self):
        """Create session analysis job"""
        
        job_code = """
        // Session analysis using event-time processing
        DataStream<Event> events = env.addSource(consumer);
        
        // Extract session events
        DataStream<SessionEvent> sessionEvents = events
            .filter(event -> event.getEventType().equals("page_view") || 
                           event.getEventType().equals("session_start") ||
                           event.getEventType().equals("session_end"))
            .keyBy(event -> event.getSessionId());
        
        // Session windowing
        DataStream<SessionMetrics> sessionMetrics = sessionEvents
            .keyBy(event -> event.getSessionId())
            .process(new SessionProcessFunction())
            .timeWindowAll(Time.minutes(1));
        
        // Calculate session metrics
        DataStream<AggregatedSessionMetrics> aggregated = sessionMetrics
            .timeWindowAll(Time.minutes(5))
            .aggregate(new SessionAggregationFunction());
        
        aggregated.addSink(new SessionMetricsSink());
        """
        
        return self.flink_client.submit_job(job_code)
    
    def create_anomaly_detection_job(self):
        """Create anomaly detection job"""
        
        job_code = """
        // Anomaly detection using CEP (Complex Event Processing)
        Pattern<Event, ?> pattern = Pattern.<Event>begin("start")
            .where(new SimpleCondition<Event>() {
                @Override
                public boolean filter(Event event) {
                    return event.getEventType().equals("error");
                }
            })
            .times(5).within(Time.minutes(1)); // 5 errors within 1 minute
        
        DataStream<Event> events = env.addSource(consumer);
        
        PatternStream<Event> patternStream = CEP.pattern(
            events.keyBy(event -> event.getUserId()),
            pattern
        );
        
        DataStream<Alert> alerts = patternStream.select(
            new PatternSelectFunction<Event, Alert>() {
                @Override
                public Alert select(Map<String, List<Event>> pattern) {
                    return new Alert("High error rate detected", pattern.get("start"));
                }
            }
        );
        
        alerts.addSink(new AlertSink());
        """
        
        return self.flink_client.submit_job(job_code)

class AggregationFunction:
    """Custom aggregation functions"""
    
    def __init__(self, metric_type):
        self.metric_type = metric_type
    
    def aggregate(self, events):
        """Aggregate events based on metric type"""
        
        if self.metric_type == 'count':
            return len(events)
        elif self.metric_type == 'sum':
            return sum(event.get('value', 0) for event in events)
        elif self.metric_type == 'avg':
            values = [event.get('value', 0) for event in events]
            return sum(values) / len(values) if values else 0
        elif self.metric_type == 'unique_count':
            return len(set(event.get('user_id') for event in events))
        elif self.metric_type == 'percentile':
            values = sorted([event.get('value', 0) for event in events])
            if not values:
                return 0
            index = int(0.95 * len(values))  # 95th percentile
            return values[index]
        
        return 0
```

### Real-time Data Store
```python
class RealTimeDataStore:
    def __init__(self):
        self.druid_client = DruidClient()
        self.clickhouse_client = ClickHouseClient()
        self.query_optimizer = QueryOptimizer()
    
    async def store_aggregated_metrics(self, metrics):
        """Store aggregated metrics"""
        
        # Prepare data for Druid
        druid_data = []
        for metric in metrics:
            druid_data.append({
                'timestamp': metric['timestamp'],
                'dimensions': metric['dimensions'],
                'metrics': metric['metrics']
            })
        
        # Batch insert to Druid
        await self.druid_client.batch_insert('analytics_metrics', druid_data)
        
        # Also store in ClickHouse for complex queries
        await self.clickhouse_client.insert('analytics_events', druid_data)
    
    async def query_real_time_metrics(self, query):
        """Query real-time metrics"""
        
        # Optimize query
        optimized_query = self.query_optimizer.optimize(query)
        
        # Choose appropriate storage based on query type
        if query.requires_exact_calculations():
            return await self.clickhouse_client.query(optimized_query)
        else:
            return await self.druid_client.query(optimized_query)
    
    async def get_dashboard_data(self, dashboard_config):
        """Get data for dashboard"""
        
        results = {}
        
        # Execute queries in parallel
        tasks = []
        for widget in dashboard_config['widgets']:
            task = asyncio.create_task(
                self.query_real_time_metrics(widget['query'])
            )
            tasks.append((widget['id'], task))
        
        # Collect results
        for widget_id, task in tasks:
            try:
                result = await task
                results[widget_id] = result
            except Exception as e:
                logger.error(f"Error querying widget {widget_id}: {e}")
                results[widget_id] = {'error': str(e)}
        
        return results

class DruidClient:
    def __init__(self):
        self.client = druid_client.DruidClient()
        self.datasource = 'analytics_metrics'
    
    async def query(self, query):
        """Execute Druid query"""
        
        druid_query = {
            'queryType': 'timeseries',
            'dataSource': self.datasource,
            'granularity': query.get('granularity', 'minute'),
            'intervals': [query['time_range']],
            'aggregations': self.build_aggregations(query['metrics']),
            'filter': self.build_filter(query.get('filters', {})),
            'context': {'timeout': 60000}
        }
        
        if query.get('group_by'):
            druid_query['queryType'] = 'groupBy'
            druid_query['dimensions'] = query['group_by']
        
        response = await self.client.query(druid_query)
        return self.parse_response(response)
    
    def build_aggregations(self, metrics):
        """Build Druid aggregations"""
        
        aggregations = []
        
        for metric in metrics:
            if metric['type'] == 'count':
                aggregations.append({
                    'type': 'longSum',
                    'name': metric['name'],
                    'fieldName': 'count'
                })
            elif metric['type'] == 'sum':
                aggregations.append({
                    'type': 'doubleSum',
                    'name': metric['name'],
                    'fieldName': metric['field']
                })
            elif metric['type'] == 'avg':
                aggregations.append({
                    'type': 'doubleSum',
                    'name': f"{metric['name']}_sum",
                    'fieldName': metric['field']
                })
                aggregations.append({
                    'type': 'longSum',
                    'name': f"{metric['name']}_count",
                    'fieldName': 'count'
                })
        
        return aggregations
    
    def build_filter(self, filters):
        """Build Druid filter"""
        
        if not filters:
            return None
        
        druid_filters = []
        
        for field, value in filters.items():
            if isinstance(value, list):
                druid_filters.append({
                    'type': 'in',
                    'dimension': field,
                    'values': value
                })
            else:
                druid_filters.append({
                    'type': 'selector',
                    'dimension': field,
                    'value': value
                })
        
        if len(druid_filters) == 1:
            return druid_filters[0]
        else:
            return {
                'type': 'and',
                'fields': druid_filters
            }

class ClickHouseClient:
    def __init__(self):
        self.client = clickhouse_client.Client()
        self.table = 'analytics_events'
    
    async def query(self, query):
        """Execute ClickHouse query"""
        
        sql = self.build_sql(query)
        
        result = await self.client.execute(sql)
        return self.parse_result(result)
    
    def build_sql(self, query):
        """Build SQL query"""
        
        select_fields = []
        
        # Add dimensions
        for dimension in query.get('group_by', []):
            select_fields.append(dimension)
        
        # Add metrics
        for metric in query['metrics']:
            if metric['type'] == 'count':
                select_fields.append(f"count(*) as {metric['name']}")
            elif metric['type'] == 'sum':
                select_fields.append(f"sum({metric['field']}) as {metric['name']}")
            elif metric['type'] == 'avg':
                select_fields.append(f"avg({metric['field']}) as {metric['name']}")
            elif metric['type'] == 'unique_count':
                select_fields.append(f"uniq({metric['field']}) as {metric['name']}")
        
        # Build SQL
        sql = f"SELECT {', '.join(select_fields)} FROM {self.table}"
        
        # Add WHERE clause
        if query.get('filters'):
            where_conditions = []
            for field, value in query['filters'].items():
                if isinstance(value, list):
                    where_conditions.append(f"{field} IN ({', '.join(repr(v) for v in value)})")
                else:
                    where_conditions.append(f"{field} = {repr(value)}")
            
            if where_conditions:
                sql += f" WHERE {' AND '.join(where_conditions)}"
        
        # Add time range
        if query.get('time_range'):
            start_time, end_time = query['time_range'].split('/')
            sql += f" AND timestamp >= '{start_time}' AND timestamp < '{end_time}'"
        
        # Add GROUP BY
        if query.get('group_by'):
            sql += f" GROUP BY {', '.join(query['group_by'])}"
        
        # Add ORDER BY
        if query.get('order_by'):
            sql += f" ORDER BY {query['order_by']}"
        
        # Add LIMIT
        if query.get('limit'):
            sql += f" LIMIT {query['limit']}"
        
        return sql
```

### Query Service và Caching
```python
class QueryService:
    def __init__(self):
        self.cache = CacheManager()
        self.real_time_store = RealTimeDataStore()
        self.historical_store = HistoricalDataStore()
        self.query_planner = QueryPlanner()
    
    async def execute_query(self, query):
        """Execute analytics query"""
        
        # Generate cache key
        cache_key = self.generate_cache_key(query)
        
        # Check cache first
        cached_result = await self.cache.get(cache_key)
        if cached_result:
            return cached_result
        
        # Plan query execution
        execution_plan = self.query_planner.plan(query)
        
        # Execute query
        result = await self.execute_plan(execution_plan)
        
        # Cache result
        await self.cache.set(cache_key, result, ttl=execution_plan.cache_ttl)
        
        return result
    
    async def execute_plan(self, execution_plan):
        """Execute query execution plan"""
        
        if execution_plan.requires_real_time_data():
            # Query real-time store
            real_time_result = await self.real_time_store.query_real_time_metrics(
                execution_plan.real_time_query
            )
            
            if execution_plan.requires_historical_data():
                # Combine with historical data
                historical_result = await self.historical_store.query(
                    execution_plan.historical_query
                )
                return self.combine_results(real_time_result, historical_result)
            else:
                return real_time_result
        else:
            # Query historical store only
            return await self.historical_store.query(execution_plan.historical_query)
    
    def generate_cache_key(self, query):
        """Generate cache key for query"""
        
        # Create deterministic hash of query
        query_str = json.dumps(query, sort_keys=True)
        cache_key = hashlib.md5(query_str.encode()).hexdigest()
        
        # Add time-based component for time-sensitive queries
        if query.get('time_range'):
            # Round to nearest minute for caching
            now = datetime.utcnow()
            rounded_time = now.replace(second=0, microsecond=0)
            cache_key += f"_{rounded_time.isoformat()}"
        
        return cache_key

class CacheManager:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.local_cache = {}
        self.cache_stats = CacheStats()
    
    async def get(self, key):
        """Get value from cache"""
        
        # Try local cache first
        if key in self.local_cache:
            self.cache_stats.record_hit('local')
            return self.local_cache[key]
        
        # Try Redis cache
        value = await self.redis_client.get(key)
        if value:
            self.cache_stats.record_hit('redis')
            # Store in local cache
            self.local_cache[key] = json.loads(value)
            return self.local_cache[key]
        
        self.cache_stats.record_miss()
        return None
    
    async def set(self, key, value, ttl=3600):
        """Set value in cache"""
        
        # Store in local cache
        self.local_cache[key] = value
        
        # Store in Redis
        await self.redis_client.setex(
            key, 
            ttl, 
            json.dumps(value, default=str)
        )
    
    async def invalidate_pattern(self, pattern):
        """Invalidate cache keys matching pattern"""
        
        # Clear local cache
        keys_to_remove = [k for k in self.local_cache.keys() if fnmatch.fnmatch(k, pattern)]
        for key in keys_to_remove:
            del self.local_cache[key]
        
        # Clear Redis cache
        redis_keys = await self.redis_client.keys(pattern)
        if redis_keys:
            await self.redis_client.delete(*redis_keys)

class QueryPlanner:
    def __init__(self):
        self.cost_model = QueryCostModel()
        self.data_freshness_requirements = DataFreshnessRequirements()
    
    def plan(self, query):
        """Plan query execution"""
        
        plan = ExecutionPlan()
        
        # Determine data freshness requirements
        freshness_requirement = self.data_freshness_requirements.get_requirement(
            query.get('query_type', 'default')
        )
        
        # Check if real-time data is needed
        if freshness_requirement <= 60:  # Less than 1 minute
            plan.requires_real_time = True
            plan.real_time_query = self.adapt_query_for_real_time(query)
        
        # Check if historical data is needed
        if query.get('time_range'):
            start_time = datetime.fromisoformat(query['time_range'].split('/')[0])
            if start_time < datetime.utcnow() - timedelta(hours=1):
                plan.requires_historical = True
                plan.historical_query = self.adapt_query_for_historical(query)
        
        # Set cache TTL based on freshness requirement
        plan.cache_ttl = min(freshness_requirement, 3600)
        
        return plan
```

### Alerting System
```python
class AlertingSystem:
    def __init__(self):
        self.rule_engine = AlertRuleEngine()
        self.notification_service = NotificationService()
        self.metrics_client = MetricsClient()
    
    async def evaluate_alerts(self, metrics):
        """Evaluate alert rules against metrics"""
        
        active_alerts = []
        
        # Get all alert rules
        rules = await self.rule_engine.get_active_rules()
        
        for rule in rules:
            try:
                # Evaluate rule against metrics
                if await self.evaluate_rule(rule, metrics):
                    alert = self.create_alert(rule, metrics)
                    active_alerts.append(alert)
                    
                    # Send notification
                    await self.notification_service.send_alert(alert)
                    
                    # Record alert metric
                    self.metrics_client.counter('alerts.fired').increment(
                        tags={'rule_id': rule.id, 'severity': rule.severity}
                    )
            
            except Exception as e:
                logger.error(f"Error evaluating alert rule {rule.id}: {e}")
        
        return active_alerts
    
    async def evaluate_rule(self, rule, metrics):
        """Evaluate single alert rule"""
        
        # Get metric value
        metric_value = self.get_metric_value(metrics, rule.metric_name)
        
        if metric_value is None:
            return False
        
        # Apply threshold condition
        if rule.condition == 'greater_than':
            return metric_value > rule.threshold
        elif rule.condition == 'less_than':
            return metric_value < rule.threshold
        elif rule.condition == 'equals':
            return metric_value == rule.threshold
        elif rule.condition == 'not_equals':
            return metric_value != rule.threshold
        
        return False
    
    def create_alert(self, rule, metrics):
        """Create alert object"""
        
        return Alert(
            rule_id=rule.id,
            rule_name=rule.name,
            severity=rule.severity,
            message=rule.message_template.format(
                metric_value=self.get_metric_value(metrics, rule.metric_name),
                threshold=rule.threshold
            ),
            timestamp=datetime.utcnow(),
            metrics=metrics
        )

class AlertRuleEngine:
    def __init__(self):
        self.rules = []
        self.rule_cache = {}
    
    async def get_active_rules(self):
        """Get all active alert rules"""
        
        # Return cached rules if available
        if self.rule_cache:
            return self.rule_cache.values()
        
        # Load rules from database
        rules = await self.load_rules_from_database()
        
        # Cache rules
        self.rule_cache = {rule.id: rule for rule in rules}
        
        return rules
    
    async def add_rule(self, rule_config):
        """Add new alert rule"""
        
        rule = AlertRule(
            id=str(uuid.uuid4()),
            name=rule_config['name'],
            metric_name=rule_config['metric_name'],
            condition=rule_config['condition'],
            threshold=rule_config['threshold'],
            severity=rule_config['severity'],
            message_template=rule_config['message_template'],
            active=True
        )
        
        # Save to database
        await self.save_rule_to_database(rule)
        
        # Update cache
        self.rule_cache[rule.id] = rule
        
        return rule
```

## 🎯 Trade-offs & Considerations

| Decision | Pros | Cons |
|----------|------|------|
| **Lambda Architecture** | Real-time + batch processing | Complexity, data consistency |
| **Kappa Architecture** | Simplified, stream-only | Limited batch processing |
| **Druid vs ClickHouse** | Better real-time, OLAP optimized | Higher complexity |
| **Exact vs Approximate** | Accuracy | Performance cost |
| **Hot vs Cold Storage** | Cost optimization | Query complexity |

## 📊 Performance Optimization

### Data Partitioning
```python
class DataPartitioningStrategy:
    def __init__(self):
        self.partition_schemes = {
            'time_based': TimeBasedPartitioning(),
            'hash_based': HashBasedPartitioning(),
            'range_based': RangeBasedPartitioning()
        }
    
    def partition_data(self, data, partition_scheme):
        """Partition data based on scheme"""
        
        partitioner = self.partition_schemes[partition_scheme]
        return partitioner.partition(data)
    
    def optimize_query_partitions(self, query):
        """Optimize query to use partitions efficiently"""
        
        # Add partition filters
        if query.get('time_range'):
            query['partition_filter'] = self.get_time_partitions(query['time_range'])
        
        return query

class PrecomputedAggregations:
    def __init__(self):
        self.aggregation_jobs = {}
    
    def create_aggregation_job(self, aggregation_config):
        """Create precomputed aggregation job"""
        
        job = {
            'id': str(uuid.uuid4()),
            'name': aggregation_config['name'],
            'source_table': aggregation_config['source_table'],
            'dimensions': aggregation_config['dimensions'],
            'metrics': aggregation_config['metrics'],
            'granularity': aggregation_config['granularity'],
            'schedule': aggregation_config['schedule']
        }
        
        # Create materialized view
        self.create_materialized_view(job)
        
        return job
    
    def create_materialized_view(self, job):
        """Create materialized view for precomputed aggregations"""
        
        sql = f"""
        CREATE MATERIALIZED VIEW {job['name']} AS
        SELECT 
            {', '.join(job['dimensions'])},
            {', '.join([f"{m['function']}({m['field']}) as {m['name']}" for m in job['metrics']])},
            toStartOfInterval(timestamp, INTERVAL {job['granularity']}) as time_bucket
        FROM {job['source_table']}
        GROUP BY {', '.join(job['dimensions'])}, time_bucket
        """
        
        # Execute SQL to create materialized view
        self.execute_sql(sql)
```

## 🔗 Related Topics

- [[Message Queue Systems]] - Event ingestion infrastructure
- [[Stream Processing]] - Real-time data processing
- [[Database Sharding]] - Scaling analytics storage
- [[Caching Strategies]] - Query performance optimization
- [[Monitoring Systems]] - System observability

## 📚 Further Reading

- Apache Kafka documentation
- Apache Flink streaming guide
- Apache Druid architecture
- ClickHouse performance tuning
- Stream processing patterns

---

**Key Takeaway**: Real-time analytics systems require careful balance between latency, accuracy, và cost. Use stream processing for real-time insights, precomputed aggregations for performance, và appropriate storage systems for different query patterns.

---

**Next Steps**: 
- [ ] Set up Kafka cluster
- [ ] Implement stream processing jobs
- [ ] Configure real-time storage
- [ ] Build dashboard APIs
- [ ] Set up alerting rules