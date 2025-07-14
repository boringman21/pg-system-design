# DevOps Best Practices

**Tags**: #devops #deployment #automation #infrastructure
**Date**: 2024-01-01

## 📝 Overview

DevOps practices là essential cho successful system design và operation. Chúng enable rapid, reliable deployments; infrastructure automation; và continuous monitoring để maintain system health và performance.

## 🚀 CI/CD Pipeline Design

### **Continuous Integration**
```yaml
# .github/workflows/ci.yml
name: Continuous Integration

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:6
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-dev.txt
    
    - name: Run linting
      run: |
        flake8 src/ tests/
        black --check src/ tests/
        isort --check-only src/ tests/
    
    - name: Run type checking
      run: mypy src/
    
    - name: Run unit tests
      run: |
        pytest tests/unit/ --cov=src --cov-report=xml
    
    - name: Run integration tests
      run: |
        pytest tests/integration/
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost/test_db
        REDIS_URL: redis://localhost:6379
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
    
    - name: Build Docker image
      run: |
        docker build -t myapp:${{ github.sha }} .
        docker tag myapp:${{ github.sha }} myapp:latest
    
    - name: Run security scan
      run: |
        docker run --rm -v $(pwd):/app -w /app securecodewarrior/docker-security-scanner
```

### **Continuous Deployment**
```yaml
# .github/workflows/cd.yml
name: Continuous Deployment

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2
    
    - name: Build and push Docker image
      run: |
        # Build image
        docker build -t myapp:${{ github.sha }} .
        
        # Tag for ECR
        docker tag myapp:${{ github.sha }} ${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}
        docker tag myapp:${{ github.sha }} ${{ secrets.ECR_REGISTRY }}/myapp:latest
        
        # Login to ECR
        aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
        
        # Push images
        docker push ${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}
        docker push ${{ secrets.ECR_REGISTRY }}/myapp:latest
    
    - name: Deploy to staging
      run: |
        # Update Kubernetes deployment
        kubectl set image deployment/myapp-staging myapp=${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}
        kubectl rollout status deployment/myapp-staging --timeout=300s
    
    - name: Run smoke tests
      run: |
        python scripts/smoke_tests.py --environment staging
    
    - name: Deploy to production
      if: success()
      run: |
        # Blue-green deployment
        kubectl set image deployment/myapp-production myapp=${{ secrets.ECR_REGISTRY }}/myapp:${{ github.sha }}
        kubectl rollout status deployment/myapp-production --timeout=600s
    
    - name: Notify deployment
      if: always()
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: 'Deployment ${{ job.status }} for commit ${{ github.sha }}'
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### **Multi-Environment Pipeline**
```python
# deploy.py - Deployment automation script
import os
import subprocess
import yaml
import time

class DeploymentManager:
    def __init__(self, environment):
        self.environment = environment
        self.config = self.load_config()
        self.kubectl = "kubectl"
    
    def load_config(self):
        with open(f"config/{self.environment}.yaml", 'r') as f:
            return yaml.safe_load(f)
    
    def deploy_application(self, image_tag):
        """
        Deploy application with zero-downtime strategy
        """
        print(f"Deploying {image_tag} to {self.environment}")
        
        # Update deployment manifest
        self.update_deployment_manifest(image_tag)
        
        # Apply Kubernetes manifests
        self.apply_manifests()
        
        # Wait for rollout
        self.wait_for_rollout()
        
        # Run health checks
        if not self.run_health_checks():
            self.rollback_deployment()
            raise Exception("Health checks failed, deployment rolled back")
        
        # Update traffic routing (if using blue-green)
        if self.config.get("deployment_strategy") == "blue_green":
            self.switch_traffic()
        
        print(f"Deployment to {self.environment} completed successfully")
    
    def update_deployment_manifest(self, image_tag):
        """
        Update Kubernetes deployment with new image
        """
        cmd = [
            self.kubectl, "set", "image",
            f"deployment/{self.config['app_name']}",
            f"{self.config['app_name']}={self.config['image_registry']}/{self.config['app_name']}:{image_tag}",
            f"--namespace={self.config['namespace']}"
        ]
        subprocess.run(cmd, check=True)
    
    def wait_for_rollout(self, timeout=600):
        """
        Wait for deployment rollout to complete
        """
        cmd = [
            self.kubectl, "rollout", "status",
            f"deployment/{self.config['app_name']}",
            f"--namespace={self.config['namespace']}",
            f"--timeout={timeout}s"
        ]
        subprocess.run(cmd, check=True)
    
    def run_health_checks(self):
        """
        Run health checks against deployed application
        """
        health_url = f"{self.config['base_url']}/health"
        
        for attempt in range(10):
            try:
                response = requests.get(health_url, timeout=10)
                if response.status_code == 200:
                    health_data = response.json()
                    if health_data.get("status") == "healthy":
                        return True
            except Exception as e:
                print(f"Health check attempt {attempt + 1} failed: {e}")
            
            time.sleep(30)
        
        return False
    
    def rollback_deployment(self):
        """
        Rollback to previous deployment
        """
        cmd = [
            self.kubectl, "rollout", "undo",
            f"deployment/{self.config['app_name']}",
            f"--namespace={self.config['namespace']}"
        ]
        subprocess.run(cmd, check=True)
        
        print("Deployment rolled back to previous version")

# Usage
if __name__ == "__main__":
    import sys
    
    environment = sys.argv[1]
    image_tag = sys.argv[2]
    
    deployer = DeploymentManager(environment)
    deployer.deploy_application(image_tag)
```

## 🏗️ Infrastructure as Code

### **Terraform Infrastructure**
```hcl
# main.tf - AWS infrastructure setup
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "myapp-terraform-state"
    key    = "infrastructure/terraform.tfstate"
    region = "us-west-2"
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC and networking
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "${var.project_name}-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["${var.aws_region}a", "${var.aws_region}b", "${var.aws_region}c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  
  tags = {
    Project     = var.project_name
    Environment = var.environment
  }
}

# EKS Cluster
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  
  cluster_name    = "${var.project_name}-${var.environment}"
  cluster_version = "1.27"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  # Node groups
  eks_managed_node_groups = {
    main = {
      desired_capacity = 3
      max_capacity     = 10
      min_capacity     = 1
      
      instance_types = ["t3.medium"]
      
      k8s_labels = {
        Environment = var.environment
        NodeGroup   = "main"
      }
    }
  }
  
  tags = {
    Project     = var.project_name
    Environment = var.environment
  }
}

# RDS Database
resource "aws_db_subnet_group" "main" {
  name       = "${var.project_name}-${var.environment}-db-subnet"
  subnet_ids = module.vpc.private_subnets
  
  tags = {
    Name = "${var.project_name}-${var.environment}-db-subnet"
  }
}

resource "aws_db_instance" "main" {
  identifier = "${var.project_name}-${var.environment}-db"
  
  engine         = "postgres"
  engine_version = "13.7"
  instance_class = "db.t3.micro"
  
  allocated_storage     = 20
  max_allocated_storage = 100
  storage_encrypted     = true
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  skip_final_snapshot = true
  
  tags = {
    Name        = "${var.project_name}-${var.environment}-db"
    Project     = var.project_name
    Environment = var.environment
  }
}

# ElastiCache Redis
resource "aws_elasticache_subnet_group" "main" {
  name       = "${var.project_name}-${var.environment}-cache-subnet"
  subnet_ids = module.vpc.private_subnets
}

resource "aws_elasticache_replication_group" "main" {
  replication_group_id       = "${var.project_name}-${var.environment}-redis"
  description                = "Redis cluster for ${var.project_name}"
  
  port               = 6379
  parameter_group_name = "default.redis7"
  node_type          = "cache.t3.micro"
  num_cache_clusters = 2
  
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  
  tags = {
    Name        = "${var.project_name}-${var.environment}-redis"
    Project     = var.project_name
    Environment = var.environment
  }
}
```

### **Kubernetes Manifests**
```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myregistry.com/myapp:latest
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: redis-url
        - name: ENVIRONMENT
          value: "production"
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - api.myapp.com
    secretName: myapp-tls
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

## 📊 Monitoring & Observability

### **Prometheus Monitoring Setup**
```yaml
# monitoring/prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
      - "/etc/prometheus/rules/*.yml"
    
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              - alertmanager:9093
    
    scrape_configs:
      # Kubernetes API server
      - job_name: 'kubernetes-apiserver'
        kubernetes_sd_configs:
          - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: default;kubernetes;https
      
      # Application metrics
      - job_name: 'myapp'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            action: keep
            regex: myapp
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
      
      # Node exporter
      - job_name: 'node-exporter'
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)

---
# monitoring/alert-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring
data:
  app-rules.yml: |
    groups:
    - name: application.rules
      rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors per second"
      
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "95th percentile latency is {{ $value }} seconds"
      
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod is crash looping"
          description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is crash looping"
```

### **Application Monitoring**
```python
# monitoring.py - Application metrics
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time
import psutil
import threading

# Define metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

ACTIVE_CONNECTIONS = Gauge(
    'active_database_connections',
    'Number of active database connections'
)

MEMORY_USAGE = Gauge(
    'process_memory_usage_bytes',
    'Process memory usage in bytes'
)

CPU_USAGE = Gauge(
    'process_cpu_usage_percent',
    'Process CPU usage percentage'
)

class MetricsCollector:
    def __init__(self, port=8000):
        self.port = port
        self.running = False
    
    def start(self):
        """Start Prometheus metrics server"""
        start_http_server(self.port)
        
        # Start system metrics collection
        self.running = True
        metrics_thread = threading.Thread(target=self._collect_system_metrics)
        metrics_thread.daemon = True
        metrics_thread.start()
    
    def _collect_system_metrics(self):
        """Collect system metrics periodically"""
        while self.running:
            try:
                # Memory usage
                process = psutil.Process()
                memory_info = process.memory_info()
                MEMORY_USAGE.set(memory_info.rss)
                
                # CPU usage
                cpu_percent = process.cpu_percent()
                CPU_USAGE.set(cpu_percent)
                
                time.sleep(15)  # Collect every 15 seconds
            except Exception as e:
                print(f"Error collecting system metrics: {e}")

def monitor_request(method, endpoint):
    """Decorator to monitor HTTP requests"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            start_time = time.time()
            status = "200"
            
            try:
                result = func(*args, **kwargs)
                return result
            except Exception as e:
                status = "500"
                raise
            finally:
                duration = time.time() - start_time
                REQUEST_COUNT.labels(method=method, endpoint=endpoint, status=status).inc()
                REQUEST_DURATION.labels(method=method, endpoint=endpoint).observe(duration)
        
        return wrapper
    return decorator

# Health check endpoint
@monitor_request("GET", "/health")
def health_check():
    """Application health check"""
    health_status = {
        "status": "healthy",
        "timestamp": time.time(),
        "checks": {}
    }
    
    # Database connectivity check
    try:
        db_connection.execute("SELECT 1")
        health_status["checks"]["database"] = "healthy"
    except Exception as e:
        health_status["checks"]["database"] = f"unhealthy: {str(e)}"
        health_status["status"] = "unhealthy"
    
    # Redis connectivity check
    try:
        redis_client.ping()
        health_status["checks"]["redis"] = "healthy"
    except Exception as e:
        health_status["checks"]["redis"] = f"unhealthy: {str(e)}"
        health_status["status"] = "unhealthy"
    
    return health_status
```

## 🔐 Security & Compliance

### **Security Scanning Pipeline**
```bash
#!/bin/bash
# security-scan.sh

echo "Starting security scans..."

# 1. Static Application Security Testing (SAST)
echo "Running SAST scan..."
bandit -r src/ -f json -o reports/sast-report.json

# 2. Dependency vulnerability scanning
echo "Scanning dependencies..."
safety check --json --output reports/dependency-scan.json

# 3. Docker image vulnerability scanning
echo "Scanning Docker image..."
trivy image --format json --output reports/image-scan.json myapp:latest

# 4. Infrastructure security scanning
echo "Scanning Terraform configurations..."
tfsec . --format json --out reports/terraform-scan.json

# 5. Kubernetes security scanning
echo "Scanning Kubernetes manifests..."
kubesec scan k8s/*.yaml > reports/k8s-security.json

# 6. Secret detection
echo "Scanning for secrets..."
truffleHog --regex --entropy=False --json src/ > reports/secrets-scan.json

echo "Security scans completed. Check reports/ directory for results."

# Evaluate results and fail if critical issues found
python scripts/evaluate_security_results.py
```

### **GitOps Security**
```yaml
# .github/workflows/security.yml
name: Security Scanning

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  security-scan:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0  # Fetch full history for secret scanning
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Run Snyk to check for vulnerabilities
      uses: snyk/actions/python@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high
    
    - name: Run GitGuardian scan
      uses: GitGuardian/ggshield-action@v1
      env:
        GITHUB_PUSH_BEFORE_SHA: ${{ github.event.before }}
        GITHUB_PUSH_BASE_SHA: ${{ github.event.base }}
        GITHUB_PULL_BASE_SHA: ${{ github.event.pull_request.base.sha }}
        GITHUB_DEFAULT_BRANCH: ${{ github.event.repository.default_branch }}
        GITGUARDIAN_API_KEY: ${{ secrets.GITGUARDIAN_API_KEY }}
```

## 🔗 Liên kết và Tham khảo

### **Related Topics**
- [[07-Microservices/Microservices-Architecture|Microservices Architecture]] - Deployment strategies for microservices
- [[Container Orchestration]] - Kubernetes best practices  
- [[08-Performance/Monitoring/Performance-Monitoring|Monitoring Systems]] - Observability implementation
- [[09-Security/Authentication-Authorization|Security Patterns]] - Security in DevOps pipelines

### **Further Reading**
- "The DevOps Handbook" - Gene Kim
- "Accelerate" - Nicole Forsgren
- "Site Reliability Engineering" - Google
- "Infrastructure as Code" - Kief Morris

### **Tools và Platforms**
- **CI/CD**: Jenkins, GitLab CI, GitHub Actions, CircleCI
- **Infrastructure**: Terraform, Ansible, CloudFormation
- **Containers**: Docker, Kubernetes, OpenShift
- **Monitoring**: Prometheus, Grafana, DataDog, New Relic

---

*Continuous improvement là key principle trong DevOps. Luôn measure, analyze, và optimize processes để đạt được better outcomes.* 