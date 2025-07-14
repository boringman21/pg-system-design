# API Gateway Pattern

**Tags**: #system-components #api-gateway #microservices #pattern
**Date**: 2024-01-01

## 📝 Overview

API Gateway là single entry point cho tất cả client requests trong microservices architecture. Nó acts như reverse proxy để route requests, aggregate responses, và enforce cross-cutting concerns như authentication, rate limiting, monitoring.

## 🎯 Core Functions

### **Request Routing**
```python
class APIGateway:
    def __init__(self):
        self.routes = {}
        self.load_balancers = {}
        self.middleware_stack = []
    
    def register_service(self, path_pattern, service_instances):
        self.routes[path_pattern] = service_instances
        self.load_balancers[path_pattern] = RoundRobinBalancer(service_instances)
    
    def route_request(self, request):
        # Find matching service
        for pattern, instances in self.routes.items():
            if self.match_pattern(request.path, pattern):
                # Select instance via load balancing
                instance = self.load_balancers[pattern].get_next()
                return self.forward_request(request, instance)
        
        raise ServiceNotFoundError(f"No service found for {request.path}")

# Example routing configuration
gateway = APIGateway()

# Register microservices
gateway.register_service("/api/users/*", [
    "http://user-service-1:8080",
    "http://user-service-2:8080"
])

gateway.register_service("/api/orders/*", [
    "http://order-service-1:8080",
    "http://order-service-2:8080"
])
```

### **Authentication & Authorization**
```python
class AuthenticationMiddleware:
    def __init__(self, jwt_secret):
        self.jwt_secret = jwt_secret
        self.public_paths = ['/api/auth/login', '/api/health']
    
    def process_request(self, request):
        # Skip auth for public endpoints
        if request.path in self.public_paths:
            return request
        
        # Extract and validate JWT token
        auth_header = request.headers.get('Authorization')
        if not auth_header or not auth_header.startswith('Bearer '):
            raise UnauthorizedError("Missing or invalid token")
        
        token = auth_header.split(' ')[1]
        try:
            payload = jwt.decode(token, self.jwt_secret, algorithms=['HS256'])
            request.user = payload  # Add user context
            return request
        except jwt.ExpiredSignatureError:
            raise UnauthorizedError("Token expired")
        except jwt.InvalidTokenError:
            raise UnauthorizedError("Invalid token")

class AuthorizationMiddleware:
    def __init__(self):
        self.permissions = {
            '/api/admin/*': ['admin'],
            '/api/users/*/profile': ['user', 'admin'],
            '/api/orders/*': ['user', 'admin']
        }
    
    def process_request(self, request):
        if not hasattr(request, 'user'):
            return request  # No auth required
        
        required_roles = self.get_required_roles(request.path)
        user_roles = request.user.get('roles', [])
        
        if required_roles and not any(role in user_roles for role in required_roles):
            raise ForbiddenError("Insufficient permissions")
        
        return request
```

## 🛡️ Cross-Cutting Concerns

### **Rate Limiting**
```python
class RateLimitingMiddleware:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.default_limit = 100  # requests per minute
        self.service_limits = {
            '/api/search': 20,  # More restrictive for expensive operations
            '/api/upload': 5
        }
    
    def process_request(self, request):
        # Determine rate limit for this endpoint
        limit = self.get_rate_limit(request.path)
        
        # Create rate limiting key
        client_id = self.get_client_identifier(request)
        key = f"rate_limit:{client_id}:{request.path}"
        
        # Check current request count
        current_count = self.redis.get(key) or 0
        if int(current_count) >= limit:
            raise RateLimitExceededError(f"Rate limit exceeded: {limit}/min")
        
        # Increment counter with expiration
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.expire(key, 60)  # 1 minute window
        pipe.execute()
        
        return request
    
    def get_client_identifier(self, request):
        # Use user ID if authenticated, otherwise IP address
        if hasattr(request, 'user') and 'user_id' in request.user:
            return f"user:{request.user['user_id']}"
        return f"ip:{request.remote_addr}"

# Circuit breaker for service resilience
class CircuitBreakerMiddleware:
    def __init__(self):
        self.circuits = {}
        self.failure_threshold = 5
        self.timeout_duration = 30  # seconds
    
    def process_request(self, request):
        service_key = self.get_service_key(request)
        circuit = self.get_circuit(service_key)
        
        if circuit.is_open():
            raise ServiceUnavailableError(f"Circuit breaker open for {service_key}")
        
        try:
            response = self.forward_request(request)
            circuit.record_success()
            return response
        except Exception as e:
            circuit.record_failure()
            raise
```

### **Request/Response Transformation**
```python
class RequestTransformationMiddleware:
    def __init__(self):
        self.transformations = {
            '/api/v1/users': self.transform_v1_to_v2,
            '/api/legacy/*': self.transform_legacy_format
        }
    
    def process_request(self, request):
        transformer = self.get_transformer(request.path)
        if transformer:
            return transformer(request)
        return request
    
    def transform_v1_to_v2(self, request):
        # Convert v1 API format to v2 internal format
        if request.method == 'POST' and 'name' in request.json:
            request.json['full_name'] = request.json.pop('name')
        return request

class ResponseAggregationMiddleware:
    """Combine responses from multiple services"""
    
    def process_response(self, request, response):
        if request.path == '/api/user-profile':
            return self.aggregate_user_profile(request, response)
        return response
    
    def aggregate_user_profile(self, request, user_response):
        user_id = user_response.json['id']
        
        # Fetch additional data from other services
        orders = self.call_service(f'/api/orders/user/{user_id}')
        preferences = self.call_service(f'/api/preferences/user/{user_id}')
        
        # Combine responses
        aggregated = {
            **user_response.json,
            'orders': orders.json if orders else [],
            'preferences': preferences.json if preferences else {}
        }
        
        return Response(json=aggregated, status=200)
```

## 📊 Load Balancing & Service Discovery

### **Load Balancing Strategies**
```python
class LoadBalancer:
    def get_next(self):
        raise NotImplementedError

class RoundRobinBalancer(LoadBalancer):
    def __init__(self, instances):
        self.instances = instances
        self.current = 0
    
    def get_next(self):
        instance = self.instances[self.current]
        self.current = (self.current + 1) % len(self.instances)
        return instance

class WeightedRoundRobinBalancer(LoadBalancer):
    def __init__(self, weighted_instances):
        # weighted_instances = [('host1', 3), ('host2', 1)]
        self.instances = []
        for host, weight in weighted_instances:
            self.instances.extend([host] * weight)
        self.current = 0
    
    def get_next(self):
        instance = self.instances[self.current]
        self.current = (self.current + 1) % len(self.instances)
        return instance

class LeastConnectionsBalancer(LoadBalancer):
    def __init__(self, instances):
        self.instances = {instance: 0 for instance in instances}
    
    def get_next(self):
        # Return instance with least active connections
        return min(self.instances.items(), key=lambda x: x[1])[0]
    
    def record_connection_start(self, instance):
        self.instances[instance] += 1
    
    def record_connection_end(self, instance):
        self.instances[instance] -= 1
```

### **Service Discovery Integration**
```python
class ServiceDiscoveryClient:
    def __init__(self, consul_client):
        self.consul = consul_client
        self.service_cache = {}
        self.cache_ttl = 30  # seconds
    
    def discover_services(self, service_name):
        # Check cache first
        cache_key = f"service:{service_name}"
        cached = self.service_cache.get(cache_key)
        if cached and time.time() - cached['timestamp'] < self.cache_ttl:
            return cached['instances']
        
        # Query service registry
        _, services = self.consul.health.service(service_name, passing=True)
        instances = []
        
        for service in services:
            instance = {
                'host': service['Service']['Address'],
                'port': service['Service']['Port'],
                'health': service['Checks'][0]['Status']
            }
            instances.append(f"http://{instance['host']}:{instance['port']}")
        
        # Update cache
        self.service_cache[cache_key] = {
            'instances': instances,
            'timestamp': time.time()
        }
        
        return instances

# Dynamic route updates
class DynamicRoutingGateway(APIGateway):
    def __init__(self, service_discovery):
        super().__init__()
        self.service_discovery = service_discovery
        self.refresh_routes_periodically()
    
    def refresh_routes_periodically(self):
        def refresh():
            while True:
                try:
                    self.update_routes()
                    time.sleep(30)  # Refresh every 30 seconds
                except Exception as e:
                    logger.error(f"Route refresh failed: {e}")
        
        thread = threading.Thread(target=refresh, daemon=True)
        thread.start()
    
    def update_routes(self):
        # Discover all registered services
        services = self.service_discovery.get_all_services()
        
        for service_name, config in services.items():
            instances = self.service_discovery.discover_services(service_name)
            if instances:
                self.register_service(config['path_pattern'], instances)
```

## 🔧 Performance Optimization

### **Caching Layer**
```python
class CachingMiddleware:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.cacheable_methods = ['GET']
        self.cache_ttl = {
            '/api/users/*': 300,  # 5 minutes
            '/api/products/*': 600,  # 10 minutes
            '/api/config': 3600   # 1 hour
        }
    
    def process_request(self, request):
        if request.method not in self.cacheable_methods:
            return request
        
        cache_key = self.generate_cache_key(request)
        cached_response = self.redis.get(cache_key)
        
        if cached_response:
            return CachedResponse(json.loads(cached_response))
        
        return request  # Continue to backend
    
    def process_response(self, request, response):
        if request.method in self.cacheable_methods and response.status == 200:
            cache_key = self.generate_cache_key(request)
            ttl = self.get_cache_ttl(request.path)
            
            self.redis.setex(
                cache_key,
                ttl,
                json.dumps(response.json)
            )
        
        return response
```

### **Connection Pooling**
```python
class ConnectionPoolManager:
    def __init__(self):
        self.pools = {}
        self.pool_config = {
            'max_connections': 100,
            'timeout': 30,
            'keepalive': True
        }
    
    def get_connection_pool(self, service_url):
        if service_url not in self.pools:
            self.pools[service_url] = urllib3.PoolManager(
                num_pools=10,
                maxsize=self.pool_config['max_connections'],
                timeout=self.pool_config['timeout']
            )
        return self.pools[service_url]
    
    def forward_request(self, request, target_url):
        pool = self.get_connection_pool(target_url)
        
        try:
            response = pool.request(
                request.method,
                f"{target_url}{request.path}",
                body=request.body,
                headers=request.headers,
                timeout=30
            )
            return response
        except Exception as e:
            logger.error(f"Request forwarding failed: {e}")
            raise ServiceUnavailableError("Backend service unavailable")
```

## 🚀 Real-World Implementation

### **Kong API Gateway Configuration**
```yaml
# Kong declarative configuration
_format_version: "3.0"

services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - name: user-routes
        paths:
          - /api/users
        methods:
          - GET
          - POST
          - PUT
          - DELETE

  - name: order-service
    url: http://order-service:8080
    routes:
      - name: order-routes
        paths:
          - /api/orders

plugins:
  - name: jwt
    service: user-service
    config:
      secret_is_base64: false
      
  - name: rate-limiting
    service: user-service
    config:
      minute: 100
      hour: 1000
      
  - name: prometheus
    config:
      per_consumer: true
```

### **AWS API Gateway with Lambda**
```python
import boto3

def setup_aws_api_gateway():
    apigateway = boto3.client('apigateway')
    
    # Create API
    api = apigateway.create_rest_api(
        name='MyMicroservicesAPI',
        description='API Gateway for microservices'
    )
    
    # Create resources and methods
    resources = apigateway.get_resources(restApiId=api['id'])
    root_id = resources['items'][0]['id']
    
    # Users resource
    users_resource = apigateway.create_resource(
        restApiId=api['id'],
        parentId=root_id,
        pathPart='users'
    )
    
    # Add CORS and authentication
    apigateway.put_method(
        restApiId=api['id'],
        resourceId=users_resource['id'],
        httpMethod='GET',
        authorization='AWS_IAM',
        apiKeyRequired=True
    )
    
    return api['id']
```

## 🔗 Related Topics

- [[Microservices Architecture]] - Service decomposition patterns
- [[Load Balancing]] - Traffic distribution strategies  
- [[Authentication-Authorization]] - Security implementations
- [[Rate Limiting]] - Traffic control mechanisms

## 📚 Further Reading

- Kong Gateway documentation
- AWS API Gateway best practices
- Netflix Zuul design patterns
- Istio service mesh integration

---

**Key Takeaway**: API Gateway centralizes cross-cutting concerns và simplifies client-service communication trong microservices. Choose between self-managed (Kong, Zuul) hoặc cloud-managed (AWS API Gateway, Azure APIM) based on requirements. 