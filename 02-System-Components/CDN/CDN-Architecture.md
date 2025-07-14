# Content Delivery Network (CDN) Architecture

**Tags**: #cdn #content-delivery #system-components #performance
**Date**: 2024-01-01

## 🎯 Tổng quan

Content Delivery Network (CDN) là hệ thống phân phối địa lý của các servers được thiết kế để cung cấp nội dung web với hiệu suất cao và độ khả dụng cao bằng cách phân phối dịch vụ tương đối với end users.

### **Mục đích chính của CDN:**
- Giảm latency bằng cách đưa nội dung gần hơn với người dùng
- Giảm tải cho origin servers
- Cải thiện availability và fault tolerance
- Tăng tốc độ tải trang web
- Tiết kiệm băng thông

## 🏗️ CDN Architecture Components

### **Edge Locations/Points of Presence (PoPs)**
```
Định nghĩa: Data centers đặt ở các vị trí địa lý khác nhau

Chức năng:
✅ Cache và serve content to end users
✅ Handle DNS queries
✅ Terminate SSL/TLS connections
✅ Perform content optimization
✅ Execute edge computing functions

Locations:
- Major cities worldwide
- Internet exchange points
- Near major ISPs
- Strategic geographical locations
```

### **Origin Servers**
```
Định nghĩa: Source servers chứa original content

Responsibilities:
✅ Store master copy of all content
✅ Handle cache misses from edge servers
✅ Serve dynamic content
✅ Handle API requests
✅ Manage content updates

Types:
- Primary origin (main server)
- Secondary origin (backup/failover)
- Shield origins (intermediate cache layer)
```

### **CDN Control Plane**
```
Components:
✅ DNS management system
✅ Load balancing algorithms
✅ Health monitoring
✅ Analytics and reporting
✅ Cache invalidation system
✅ Configuration management

Functions:
- Route users to optimal edge location
- Monitor edge server health
- Manage cache policies
- Handle content purging
- Collect performance metrics
```

## 🔄 CDN Working Mechanism

### **Request Flow**
```
1. User Request
   ↓
2. DNS Resolution → CDN DNS
   ↓
3. Edge Location Selection (based on proximity, load, health)
   ↓
4. Edge Server Check Cache
   ↓
5a. Cache Hit → Serve Content
   ↓
5b. Cache Miss → Fetch from Origin
   ↓
6. Cache Content at Edge
   ↓
7. Serve Content to User
```

### **Edge Location Selection Algorithm**
```python
class EdgeLocationSelector:
    def __init__(self):
        self.geo_database = GeoLocationDB()
        self.health_monitor = HealthMonitor()
        self.load_balancer = LoadBalancer()
    
    def select_optimal_edge(self, user_ip: str, content_type: str) -> str:
        """Select best edge location for user request"""
        
        # 1. Get user's geographical location
        user_location = self.geo_database.get_location(user_ip)
        
        # 2. Find candidate edge locations
        nearby_edges = self.find_nearby_edges(
            user_location, 
            max_distance_km=500
        )
        
        # 3. Filter healthy edge locations
        healthy_edges = [
            edge for edge in nearby_edges 
            if self.health_monitor.is_healthy(edge.id)
        ]
        
        # 4. Score edges based on multiple factors
        scored_edges = []
        for edge in healthy_edges:
            score = self.calculate_edge_score(
                edge, user_location, content_type
            )
            scored_edges.append((edge, score))
        
        # 5. Select best edge
        scored_edges.sort(key=lambda x: x[1], reverse=True)
        
        if scored_edges:
            return scored_edges[0][0].id
        else:
            # Fallback to any available edge
            return self.get_fallback_edge(user_location)
    
    def calculate_edge_score(self, edge, user_location, content_type):
        """Calculate score for edge location selection"""
        
        # Distance factor (closer is better)
        distance = self.calculate_distance(user_location, edge.location)
        distance_score = max(0, 100 - (distance / 10))  # 100 points max
        
        # Load factor (less loaded is better)
        current_load = self.get_current_load(edge.id)
        load_score = max(0, 100 - current_load)  # 100 points max
        
        # Cache hit ratio for content type
        hit_ratio = self.get_cache_hit_ratio(edge.id, content_type)
        cache_score = hit_ratio * 100  # 100 points max
        
        # Network latency factor
        latency = self.get_average_latency(edge.id, user_location)
        latency_score = max(0, 100 - latency)  # 100 points max
        
        # Weighted score
        total_score = (
            distance_score * 0.3 +
            load_score * 0.2 +
            cache_score * 0.3 +
            latency_score * 0.2
        )
        
        return total_score

# CDN DNS Resolution
class CDNDNSResolver:
    def __init__(self):
        self.edge_selector = EdgeLocationSelector()
        self.dns_cache = DNSCache()
    
    def resolve_cdn_domain(self, domain: str, user_ip: str, record_type: str):
        """Resolve CDN domain to optimal edge location"""
        
        # Check if we have cached result for this user
        cache_key = f"{domain}:{user_ip}:{record_type}"
        cached_result = self.dns_cache.get(cache_key)
        if cached_result and not cached_result.is_expired():
            return cached_result.ip_address
        
        # Extract content information from domain
        content_type = self.extract_content_type(domain)
        
        # Select optimal edge location
        optimal_edge = self.edge_selector.select_optimal_edge(
            user_ip, content_type
        )
        
        # Get edge IP address
        edge_ip = self.get_edge_ip(optimal_edge)
        
        # Cache the result
        self.dns_cache.set(
            cache_key, 
            edge_ip, 
            ttl=300  # 5 minutes
        )
        
        return edge_ip
    
    def extract_content_type(self, domain: str) -> str:
        """Extract content type from domain/subdomain"""
        
        subdomain_mapping = {
            'images': 'image',
            'video': 'video',
            'static': 'static',
            'api': 'api',
            'js': 'javascript',
            'css': 'stylesheet'
        }
        
        for subdomain, content_type in subdomain_mapping.items():
            if subdomain in domain:
                return content_type
        
        return 'web'  # default
```

## 📦 CDN Caching Strategies

### **Cache Levels và TTL**
```
CDN Cache Hierarchy:

L1: Browser Cache (Client-side)
- TTL: 1 hour - 1 day
- Controls: Cache-Control headers
- Content: Static assets, images

L2: CDN Edge Cache
- TTL: 1 hour - 7 days  
- Controls: CDN configuration
- Content: Popular content, static files

L3: CDN Regional Cache (Shield)
- TTL: 1 day - 30 days
- Controls: CDN shield settings
- Content: Less popular but still cached content

L4: Origin Cache
- TTL: Varies by content
- Controls: Application logic
- Content: Dynamic content with caching headers
```

### **Cache Invalidation Strategies**
```python
class CDNCacheManager:
    def __init__(self):
        self.cache_store = DistributedCache()
        self.invalidation_queue = MessageQueue('cache_invalidation')
        self.edge_servers = EdgeServerManager()
    
    async def invalidate_content(self, 
                               content_paths: List[str], 
                               invalidation_type: str = 'purge'):
        """Invalidate content across all edge locations"""
        
        invalidation_id = self.generate_invalidation_id()
        
        invalidation_request = {
            'id': invalidation_id,
            'paths': content_paths,
            'type': invalidation_type,  # 'purge' or 'refresh'
            'timestamp': datetime.utcnow().isoformat(),
            'priority': self.calculate_priority(content_paths)
        }
        
        # Send invalidation request to all edge locations
        edge_locations = await self.edge_servers.get_all_locations()
        
        tasks = []
        for edge in edge_locations:
            task = self.send_invalidation_to_edge(edge.id, invalidation_request)
            tasks.append(task)
        
        # Wait for all invalidations to complete
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Track invalidation status
        success_count = sum(1 for r in results if not isinstance(r, Exception))
        
        return {
            'invalidation_id': invalidation_id,
            'total_edges': len(edge_locations),
            'successful_edges': success_count,
            'completion_time': datetime.utcnow().isoformat()
        }
    
    async def smart_cache_warming(self, content_path: str, target_regions: List[str]):
        """Proactively warm cache for expected popular content"""
        
        # Predict which edge locations will need this content
        predicted_edges = await self.predict_high_demand_edges(
            content_path, target_regions
        )
        
        # Warm cache at predicted locations
        warming_tasks = []
        for edge_id in predicted_edges:
            task = self.warm_cache_at_edge(edge_id, content_path)
            warming_tasks.append(task)
        
        await asyncio.gather(*warming_tasks)
        
        return {
            'content_path': content_path,
            'warmed_edges': len(predicted_edges),
            'regions': target_regions
        }
    
    def calculate_cache_efficiency(self, edge_id: str, time_window: str):
        """Calculate cache performance metrics"""
        
        metrics = self.cache_store.get_metrics(edge_id, time_window)
        
        hit_ratio = metrics['cache_hits'] / (metrics['cache_hits'] + metrics['cache_misses'])
        byte_hit_ratio = metrics['bytes_served_from_cache'] / metrics['total_bytes_served']
        
        efficiency_score = (hit_ratio * 0.6) + (byte_hit_ratio * 0.4)
        
        return {
            'edge_id': edge_id,
            'hit_ratio': hit_ratio,
            'byte_hit_ratio': byte_hit_ratio,
            'efficiency_score': efficiency_score,
            'total_requests': metrics['cache_hits'] + metrics['cache_misses'],
            'cache_size_gb': metrics['cache_size_bytes'] / (1024**3)
        }

# Cache Policy Management
class CachePolicyManager:
    def __init__(self):
        self.content_analyzer = ContentAnalyzer()
        self.usage_predictor = UsagePredictor()
    
    def determine_cache_policy(self, content_info: dict) -> dict:
        """Determine optimal caching policy for content"""
        
        content_type = content_info['content_type']
        content_size = content_info['size_bytes']
        popularity_score = content_info.get('popularity_score', 0)
        
        # Base TTL by content type
        base_ttl_map = {
            'image': 86400 * 7,      # 7 days
            'css': 86400 * 30,       # 30 days
            'js': 86400 * 30,        # 30 days
            'video': 86400 * 1,      # 1 day
            'html': 3600,            # 1 hour
            'api': 300,              # 5 minutes
            'font': 86400 * 365      # 1 year
        }
        
        base_ttl = base_ttl_map.get(content_type, 3600)
        
        # Adjust TTL based on popularity
        if popularity_score > 0.8:
            ttl = base_ttl * 2  # Popular content cached longer
        elif popularity_score < 0.2:
            ttl = base_ttl * 0.5  # Unpopular content cached shorter
        else:
            ttl = base_ttl
        
        # Adjust for content size
        if content_size > 100 * 1024 * 1024:  # > 100MB
            cache_tier = 'shield_only'  # Only cache at shield/regional level
        elif content_size > 10 * 1024 * 1024:  # > 10MB
            cache_tier = 'regional_and_edge'
        else:
            cache_tier = 'all_levels'
        
        return {
            'ttl_seconds': int(ttl),
            'cache_tier': cache_tier,
            'cache_control': f'public, max-age={int(ttl)}',
            'vary_headers': self.determine_vary_headers(content_info),
            'compression': self.should_compress(content_info)
        }
    
    def determine_vary_headers(self, content_info: dict) -> List[str]:
        """Determine which headers to vary caching on"""
        
        vary_headers = []
        
        # Mobile vs desktop content
        if content_info.get('responsive', False):
            vary_headers.append('User-Agent')
        
        # Internationalization
        if content_info.get('multilingual', False):
            vary_headers.extend(['Accept-Language', 'Accept-Encoding'])
        
        # Content negotiation
        if content_info['content_type'] in ['image', 'video']:
            vary_headers.append('Accept')
        
        return vary_headers
```

## 🔧 CDN Optimization Techniques

### **Content Optimization**
```python
class ContentOptimizer:
    def __init__(self):
        self.image_optimizer = ImageOptimizer()
        self.compressor = ContentCompressor()
        self.minifier = CodeMinifier()
    
    async def optimize_content(self, content: bytes, content_type: str, user_agent: str):
        """Optimize content before serving to user"""
        
        optimizations = []
        
        # Image optimization
        if content_type.startswith('image/'):
            optimized_content = await self.optimize_image(content, user_agent)
            optimizations.append('image_optimization')
        
        # Text compression
        elif content_type in ['text/html', 'text/css', 'application/javascript']:
            # Minify first
            if content_type == 'text/css':
                content = self.minifier.minify_css(content.decode())
            elif content_type == 'application/javascript':
                content = self.minifier.minify_js(content.decode())
            elif content_type == 'text/html':
                content = self.minifier.minify_html(content.decode())
            
            # Then compress
            optimized_content = self.compressor.gzip_compress(content.encode())
            optimizations.extend(['minification', 'gzip_compression'])
        
        # General compression for other content
        else:
            if len(content) > 1024:  # Only compress if > 1KB
                optimized_content = self.compressor.compress(content, user_agent)
                optimizations.append('compression')
            else:
                optimized_content = content
        
        return {
            'content': optimized_content,
            'optimizations_applied': optimizations,
            'original_size': len(content),
            'optimized_size': len(optimized_content),
            'compression_ratio': len(optimized_content) / len(content)
        }
    
    async def optimize_image(self, image_data: bytes, user_agent: str):
        """Optimize images based on client capabilities"""
        
        # Detect client capabilities
        supports_webp = 'webp' in user_agent.lower()
        supports_avif = 'avif' in user_agent.lower()
        is_mobile = self.is_mobile_device(user_agent)
        
        # Choose optimal format
        if supports_avif:
            target_format = 'avif'
        elif supports_webp:
            target_format = 'webp'
        else:
            target_format = 'jpeg'
        
        # Adjust quality for mobile
        quality = 75 if is_mobile else 85
        
        # Resize for mobile if needed
        if is_mobile:
            max_width = 800
        else:
            max_width = 1920
        
        optimized_image = await self.image_optimizer.optimize(
            image_data,
            format=target_format,
            quality=quality,
            max_width=max_width
        )
        
        return optimized_image

# HTTP/2 and HTTP/3 Optimization
class ProtocolOptimizer:
    def __init__(self):
        self.http2_settings = HTTP2Settings()
        self.http3_settings = HTTP3Settings()
    
    def configure_http2_optimization(self):
        """Configure HTTP/2 optimizations"""
        
        return {
            'server_push': {
                'enabled': True,
                'push_critical_resources': True,
                'max_concurrent_pushes': 10
            },
            'multiplexing': {
                'max_concurrent_streams': 100,
                'initial_window_size': 65535,
                'max_frame_size': 16384
            },
            'header_compression': {
                'hpack_enabled': True,
                'hpack_table_size': 4096
            }
        }
    
    def configure_http3_optimization(self):
        """Configure HTTP/3 (QUIC) optimizations"""
        
        return {
            'quic_settings': {
                'initial_max_data': 1048576,  # 1MB
                'initial_max_stream_data': 262144,  # 256KB
                'max_idle_timeout': 30000,  # 30 seconds
                'max_udp_payload_size': 1472
            },
            'congestion_control': 'bbr',
            'connection_migration': True,
            '0rtt_enabled': True
        }

# Edge Computing Functions
class EdgeComputingService:
    def __init__(self):
        self.function_registry = FunctionRegistry()
        self.execution_engine = EdgeExecutionEngine()
    
    async def execute_edge_function(self, function_name: str, request_context: dict):
        """Execute serverless function at edge location"""
        
        # Get function code and configuration
        function_config = await self.function_registry.get_function(function_name)
        
        if not function_config:
            raise FunctionNotFoundError(f"Function {function_name} not found")
        
        # Prepare execution context
        execution_context = {
            'request': request_context,
            'geo_location': request_context.get('geo_location'),
            'user_agent': request_context.get('user_agent'),
            'timestamp': datetime.utcnow().isoformat()
        }
        
        # Execute function with timeout
        try:
            result = await asyncio.wait_for(
                self.execution_engine.execute(
                    function_config['code'],
                    execution_context
                ),
                timeout=function_config.get('timeout', 30)
            )
            
            return {
                'success': True,
                'result': result,
                'execution_time_ms': result.get('execution_time', 0)
            }
            
        except asyncio.TimeoutError:
            return {
                'success': False,
                'error': 'Function execution timeout',
                'timeout_seconds': function_config.get('timeout', 30)
            }
        except Exception as e:
            return {
                'success': False,
                'error': str(e),
                'error_type': type(e).__name__
            }

# Example edge functions
edge_functions = {
    'image_resize': '''
async function imageResize(context) {
    const { width, height, quality } = context.request.query;
    const imageUrl = context.request.path;
    
    // Resize image on the fly
    const resizedImage = await resizeImage(imageUrl, {
        width: parseInt(width) || 800,
        height: parseInt(height) || 600,
        quality: parseInt(quality) || 80
    });
    
    return {
        contentType: 'image/jpeg',
        body: resizedImage,
        headers: {
            'Cache-Control': 'public, max-age=86400',
            'Content-Length': resizedImage.length
        }
    };
}
''',
    
    'geo_redirect': '''
async function geoRedirect(context) {
    const userCountry = context.geo_location.country;
    const countryRedirects = {
        'US': '/us',
        'GB': '/uk', 
        'DE': '/de',
        'JP': '/jp'
    };
    
    const redirectPath = countryRedirects[userCountry] || '/global';
    
    return {
        status: 302,
        headers: {
            'Location': `https://example.com${redirectPath}`,
            'Cache-Control': 'no-cache'
        }
    };
}
''',
    
    'ab_test': '''
async function abTest(context) {
    const userId = context.request.headers['x-user-id'];
    const testId = context.request.query.test_id;
    
    // Simple hash-based assignment
    const hash = await crypto.subtle.digest('SHA-256', 
        new TextEncoder().encode(userId + testId)
    );
    const variant = new Uint8Array(hash)[0] % 2 === 0 ? 'A' : 'B';
    
    return {
        headers: {
            'X-Test-Variant': variant,
            'Set-Cookie': `test_${testId}=${variant}; Path=/; Max-Age=86400`
        }
    };
}
'''
}
```

## 📊 CDN Performance Monitoring

### **Key Metrics và KPIs**
```python
class CDNMonitoringService:
    def __init__(self):
        self.metrics_collector = MetricsCollector()
        self.alerting_system = AlertingSystem()
        self.dashboard = CDNDashboard()
    
    async def collect_cdn_metrics(self):
        """Collect comprehensive CDN performance metrics"""
        
        metrics = {
            'performance_metrics': await self.collect_performance_metrics(),
            'availability_metrics': await self.collect_availability_metrics(),
            'traffic_metrics': await self.collect_traffic_metrics(),
            'cost_metrics': await self.collect_cost_metrics(),
            'security_metrics': await self.collect_security_metrics()
        }
        
        # Store metrics for historical analysis
        await self.metrics_collector.store_metrics(metrics)
        
        # Check for alerts
        await self.check_performance_alerts(metrics)
        
        return metrics
    
    async def collect_performance_metrics(self):
        """Collect CDN performance metrics"""
        
        all_edges = await self.get_all_edge_locations()
        
        performance_data = {
            'global_metrics': {
                'avg_response_time_ms': 0,
                'p95_response_time_ms': 0,
                'p99_response_time_ms': 0,
                'cache_hit_ratio': 0,
                'bandwidth_utilization': 0
            },
            'edge_metrics': {}
        }
        
        total_response_times = []
        total_cache_hits = 0
        total_requests = 0
        
        for edge in all_edges:
            edge_data = await self.get_edge_performance(edge.id)
            
            performance_data['edge_metrics'][edge.id] = {
                'location': edge.location,
                'response_time_ms': edge_data['avg_response_time'],
                'cache_hit_ratio': edge_data['cache_hit_ratio'],
                'requests_per_second': edge_data['rps'],
                'bandwidth_mbps': edge_data['bandwidth'],
                'cpu_utilization': edge_data['cpu_usage'],
                'memory_utilization': edge_data['memory_usage']
            }
            
            # Aggregate for global metrics
            total_response_times.extend(edge_data['response_times'])
            total_cache_hits += edge_data['cache_hits']
            total_requests += edge_data['total_requests']
        
        # Calculate global metrics
        if total_response_times:
            performance_data['global_metrics']['avg_response_time_ms'] = \
                sum(total_response_times) / len(total_response_times)
            performance_data['global_metrics']['p95_response_time_ms'] = \
                self.percentile(total_response_times, 95)
            performance_data['global_metrics']['p99_response_time_ms'] = \
                self.percentile(total_response_times, 99)
        
        if total_requests > 0:
            performance_data['global_metrics']['cache_hit_ratio'] = \
                total_cache_hits / total_requests
        
        return performance_data
    
    async def collect_availability_metrics(self):
        """Collect CDN availability and uptime metrics"""
        
        all_edges = await self.get_all_edge_locations()
        
        availability_data = {
            'global_uptime_percentage': 0,
            'total_edges': len(all_edges),
            'healthy_edges': 0,
            'degraded_edges': 0,
            'unhealthy_edges': 0,
            'edge_status': {}
        }
        
        healthy_count = 0
        degraded_count = 0
        unhealthy_count = 0
        
        for edge in all_edges:
            health_status = await self.check_edge_health(edge.id)
            
            availability_data['edge_status'][edge.id] = {
                'status': health_status['status'],
                'uptime_percentage': health_status['uptime_24h'],
                'last_health_check': health_status['last_check'],
                'response_time_ms': health_status['health_check_time'],
                'error_rate': health_status['error_rate']
            }
            
            if health_status['status'] == 'healthy':
                healthy_count += 1
            elif health_status['status'] == 'degraded':
                degraded_count += 1
            else:
                unhealthy_count += 1
        
        availability_data['healthy_edges'] = healthy_count
        availability_data['degraded_edges'] = degraded_count
        availability_data['unhealthy_edges'] = unhealthy_count
        availability_data['global_uptime_percentage'] = \
            (healthy_count / len(all_edges)) * 100 if all_edges else 0
        
        return availability_data
    
    async def generate_performance_report(self, time_period: str):
        """Generate comprehensive CDN performance report"""
        
        report_data = {
            'report_period': time_period,
            'generated_at': datetime.utcnow().isoformat(),
            'summary': {},
            'detailed_metrics': {},
            'recommendations': []
        }
        
        # Collect data for the specified period
        historical_data = await self.get_historical_metrics(time_period)
        
        # Calculate summary statistics
        report_data['summary'] = {
            'total_requests': sum(d['requests'] for d in historical_data),
            'total_bandwidth_gb': sum(d['bandwidth_bytes'] for d in historical_data) / (1024**3),
            'avg_cache_hit_ratio': sum(d['cache_hit_ratio'] for d in historical_data) / len(historical_data),
            'avg_response_time_ms': sum(d['response_time'] for d in historical_data) / len(historical_data),
            'uptime_percentage': sum(d['uptime'] for d in historical_data) / len(historical_data),
            'cost_usd': sum(d['cost'] for d in historical_data)
        }
        
        # Identify performance trends
        trends = self.analyze_performance_trends(historical_data)
        report_data['trends'] = trends
        
        # Generate recommendations
        recommendations = await self.generate_optimization_recommendations(
            report_data['summary'], trends
        )
        report_data['recommendations'] = recommendations
        
        return report_data
    
    async def generate_optimization_recommendations(self, summary: dict, trends: dict):
        """Generate actionable optimization recommendations"""
        
        recommendations = []
        
        # Cache hit ratio recommendations
        if summary['avg_cache_hit_ratio'] < 0.85:
            recommendations.append({
                'category': 'cache_optimization',
                'priority': 'high',
                'title': 'Improve Cache Hit Ratio',
                'description': f"Current cache hit ratio is {summary['avg_cache_hit_ratio']:.1%}. Target should be >85%.",
                'actions': [
                    'Review and optimize cache TTL settings',
                    'Implement cache warming for popular content',
                    'Analyze cache invalidation patterns',
                    'Consider adding more edge locations'
                ]
            })
        
        # Response time recommendations
        if summary['avg_response_time_ms'] > 200:
            recommendations.append({
                'category': 'performance',
                'priority': 'high',
                'title': 'Reduce Response Time',
                'description': f"Average response time is {summary['avg_response_time_ms']:.0f}ms. Target should be <200ms.",
                'actions': [
                    'Enable HTTP/2 and HTTP/3',
                    'Optimize content compression',
                    'Implement connection keep-alive',
                    'Review edge location coverage'
                ]
            })
        
        # Cost optimization recommendations
        if trends.get('cost_trend', 0) > 0.1:  # Cost increasing by >10%
            recommendations.append({
                'category': 'cost_optimization',
                'priority': 'medium',
                'title': 'Optimize CDN Costs',
                'description': 'CDN costs are trending upward. Consider optimization opportunities.',
                'actions': [
                    'Review traffic patterns for anomalies',
                    'Optimize cache policies to reduce origin fetches',
                    'Implement intelligent content compression',
                    'Consider traffic-based pricing tiers'
                ]
            })
        
        return recommendations

# Real-time CDN Dashboard
class CDNDashboard:
    def __init__(self):
        self.websocket_manager = WebSocketManager()
        self.data_processor = RealTimeDataProcessor()
    
    async def stream_realtime_metrics(self):
        """Stream real-time CDN metrics to dashboard"""
        
        while True:
            try:
                # Collect current metrics
                current_metrics = await self.collect_current_metrics()
                
                # Process and format for dashboard
                dashboard_data = self.format_for_dashboard(current_metrics)
                
                # Broadcast to connected clients
                await self.websocket_manager.broadcast(dashboard_data)
                
                # Wait before next update
                await asyncio.sleep(10)  # Update every 10 seconds
                
            except Exception as e:
                print(f"Error streaming metrics: {e}")
                await asyncio.sleep(30)  # Wait longer on error
    
    def format_for_dashboard(self, metrics: dict):
        """Format metrics for dashboard consumption"""
        
        return {
            'timestamp': datetime.utcnow().isoformat(),
            'global_stats': {
                'requests_per_second': metrics['global_rps'],
                'bandwidth_gbps': metrics['global_bandwidth'] / (1024**3),
                'cache_hit_ratio': metrics['global_cache_hit_ratio'],
                'avg_response_time_ms': metrics['global_response_time'],
                'active_edges': metrics['healthy_edge_count']
            },
            'top_edge_locations': metrics['top_performing_edges'][:10],
            'alerts': metrics['active_alerts'],
            'traffic_map': metrics['traffic_by_region']
        }
```

## 💡 CDN Best Practices

### **Configuration Best Practices**
```
Cache Headers:
✅ Set appropriate Cache-Control headers
✅ Use ETags for cache validation
✅ Implement Last-Modified headers
✅ Set Vary headers correctly
✅ Use immutable for static assets

Security:
✅ Enable HTTPS everywhere
✅ Implement HSTS headers
✅ Use CSP headers
✅ Enable DDoS protection
✅ Implement rate limiting

Performance:
✅ Enable Brotli/Gzip compression
✅ Use HTTP/2 and HTTP/3
✅ Implement resource hints (preload, prefetch)
✅ Optimize image formats (WebP, AVIF)
✅ Minimize DNS lookups
```

### **Cost Optimization Strategies**
```
Traffic Management:
- Use tiered pricing effectively
- Implement smart origin offloading
- Optimize cache hit ratios
- Monitor bandwidth usage patterns

Content Strategy:
- Compress all text content
- Optimize images and videos
- Remove unused assets
- Implement lazy loading

Architecture:
- Use regional shields
- Implement pull zones efficiently
- Monitor traffic patterns
- Right-size cache storage
```

---

**Checklist cho CDN Implementation:**
- [ ] Plan edge location strategy
- [ ] Configure cache policies
- [ ] Set up origin servers
- [ ] Implement health monitoring
- [ ] Configure SSL/TLS
- [ ] Set up analytics and reporting
- [ ] Plan failover strategies
- [ ] Test performance improvements
- [ ] Monitor costs and usage
- [ ] Optimize based on data

**Key Trade-offs:**
- **Performance vs Cost**: More edge locations = better performance but higher costs
- **Cache vs Freshness**: Longer TTL = better performance but potential stale content
- **Complexity vs Control**: Managed CDN vs self-hosted solution

**Remember**: CDN là foundation quan trọng cho global scale applications! 🌍 