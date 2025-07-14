# Performance Profiling

**Tags**: #performance #profiling #debugging #optimization
**Date**: 2024-01-01

## 🎯 Tổng quan

Performance Profiling là quá trình phân tích và đo lường hiệu suất ứng dụng để:
- Xác định bottlenecks và performance issues
- Hiểu cách ứng dụng sử dụng tài nguyên
- Tối ưu hóa hiệu suất dựa trên dữ liệu thực tế
- Theo dõi performance regression

## 📊 Types of Profiling

### **CPU Profiling**
```
Mục đích: Xác định functions/methods tốn nhiều CPU nhất

Metrics:
- CPU time spent in each function
- Call count and frequency
- Hot paths in code
- CPU utilization per thread

Tools:
✅ Java: JProfiler, YourKit, Java Flight Recorder
✅ Python: cProfile, py-spy, pyflame
✅ Node.js: clinic.js, 0x, V8 profiler
✅ .NET: PerfView, dotTrace, Application Insights
✅ Go: pprof, go tool trace
```

### **Memory Profiling**
```
Mục đích: Phân tích memory usage và detect memory leaks

Metrics:
- Heap size và growth
- Object allocation patterns
- Memory leaks
- Garbage collection behavior

Tools:
✅ Java: jmap, jhat, Eclipse MAT
✅ Python: memory_profiler, tracemalloc
✅ Node.js: heapdump, memwatch-next
✅ .NET: PerfView, DebugDiag
✅ Go: pprof heap profiling
```

### **I/O Profiling**
```
Mục đích: Theo dõi disk và network I/O performance

Metrics:
- File read/write operations
- Network request latency
- Database query performance
- Cache hit/miss ratios

Tools:
✅ Linux: iotop, iostat, strace
✅ Application: APM tools (New Relic, Datadog)
✅ Database: EXPLAIN ANALYZE, slow query logs
```

## 🔧 Profiling Tools và Techniques

### **Java Profiling**
```java
// Using Java Flight Recorder (JFR)
// JVM Arguments:
-XX:+FlightRecorder 
-XX:StartFlightRecording=duration=60s,filename=profile.jfr

// Code instrumentation
public class PerformanceProfiler {
    public static void main(String[] args) {
        // Start JFR recording programmatically
        FlightRecorder.start(
            Configuration.create("profile"),
            Duration.ofSeconds(60),
            Path.of("app-profile.jfr")
        );
        
        // Your application code
        runApplication();
    }
    
    @ProfiledMethod
    public void expensiveOperation() {
        long start = System.nanoTime();
        try {
            // Business logic
            performComplexCalculation();
        } finally {
            long duration = System.nanoTime() - start;
            System.out.printf("Method took: %.2f ms%n", 
                duration / 1_000_000.0);
        }
    }
}

// Using JProfiler API
import com.jprofiler.api.agent.Controller;

public class ProfiledService {
    public void criticalMethod() {
        Controller.startCPURecording(true);
        try {
            // Method implementation
        } finally {
            Controller.stopCPURecording();
        }
    }
}
```

### **Python Profiling**
```python
import cProfile
import pstats
import tracemalloc
from functools import wraps
import time

# CPU Profiling with cProfile
def profile_cpu():
    pr = cProfile.Profile()
    pr.enable()
    
    # Your code here
    expensive_function()
    
    pr.disable()
    stats = pstats.Stats(pr)
    stats.sort_stats('cumulative')
    stats.print_stats(10)  # Top 10 functions

# Memory Profiling
def profile_memory():
    tracemalloc.start()
    
    # Your code here
    memory_intensive_function()
    
    current, peak = tracemalloc.get_traced_memory()
    print(f"Current memory usage: {current / 1024 / 1024:.1f} MB")
    print(f"Peak memory usage: {peak / 1024 / 1024:.1f} MB")
    tracemalloc.stop()

# Custom profiling decorator
def profile_function(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        # Memory tracking
        tracemalloc.start()
        
        # CPU time tracking
        start_time = time.perf_counter()
        start_cpu = time.process_time()
        
        try:
            result = func(*args, **kwargs)
            return result
        finally:
            # Calculate metrics
            end_time = time.perf_counter()
            end_cpu = time.process_time()
            
            current, peak = tracemalloc.get_traced_memory()
            tracemalloc.stop()
            
            print(f"{func.__name__} Performance:")
            print(f"  Wall time: {(end_time - start_time)*1000:.2f} ms")
            print(f"  CPU time: {(end_cpu - start_cpu)*1000:.2f} ms")
            print(f"  Memory peak: {peak / 1024 / 1024:.1f} MB")
    
    return wrapper

# Usage example
@profile_function
def slow_function():
    # Simulate CPU-intensive work
    numbers = [i**2 for i in range(1000000)]
    return sum(numbers)

# Line-by-line profiling
from line_profiler import LineProfiler

def detailed_profiling():
    profiler = LineProfiler()
    profiler.add_function(target_function)
    profiler.enable_by_count()
    
    target_function()
    
    profiler.print_stats()
```

### **Node.js Profiling**
```javascript
// CPU Profiling with V8 Inspector
const inspector = require('inspector');
const fs = require('fs');

function startCPUProfiler() {
    const session = new inspector.Session();
    session.connect();
    
    session.post('Profiler.enable', () => {
        session.post('Profiler.start', () => {
            console.log('CPU profiler started');
        });
    });
    
    return session;
}

function stopCPUProfiler(session) {
    session.post('Profiler.stop', (err, { profile }) => {
        fs.writeFileSync('cpu-profile.json', JSON.stringify(profile));
        console.log('CPU profile saved');
    });
}

// Memory profiling
const v8 = require('v8');

function takeHeapSnapshot() {
    const heapSnapshot = v8.getHeapSnapshot();
    const fileName = `heap-${Date.now()}.heapsnapshot`;
    const fileStream = fs.createWriteStream(fileName);
    heapSnapshot.pipe(fileStream);
    console.log(`Heap snapshot saved to ${fileName}`);
}

// Custom performance monitoring
class PerformanceMonitor {
    constructor() {
        this.metrics = {};
    }
    
    startTimer(name) {
        this.metrics[name] = {
            start: process.hrtime.bigint(),
            memStart: process.memoryUsage()
        };
    }
    
    endTimer(name) {
        if (!this.metrics[name]) return;
        
        const end = process.hrtime.bigint();
        const memEnd = process.memoryUsage();
        const metric = this.metrics[name];
        
        const duration = Number(end - metric.start) / 1000000; // ms
        const memDiff = memEnd.heapUsed - metric.memStart.heapUsed;
        
        console.log(`${name}:`);
        console.log(`  Duration: ${duration.toFixed(2)} ms`);
        console.log(`  Memory delta: ${(memDiff / 1024 / 1024).toFixed(2)} MB`);
        
        delete this.metrics[name];
    }
}

// Usage
const monitor = new PerformanceMonitor();

async function expensiveOperation() {
    monitor.startTimer('operation');
    
    // Your code here
    await heavyComputation();
    
    monitor.endTimer('operation');
}

// Event loop lag monitoring
function measureEventLoopLag() {
    const start = process.hrtime.bigint();
    setImmediate(() => {
        const lag = Number(process.hrtime.bigint() - start) / 1000000;
        console.log(`Event loop lag: ${lag.toFixed(2)} ms`);
    });
}

setInterval(measureEventLoopLag, 1000);
```

### **Go Profiling**
```go
package main

import (
    "context"
    "fmt"
    "net/http"
    _ "net/http/pprof"
    "os"
    "runtime"
    "runtime/pprof"
    "runtime/trace"
    "time"
)

// CPU Profiling
func profileCPU() {
    f, err := os.Create("cpu.prof")
    if err != nil {
        panic(err)
    }
    defer f.Close()
    
    if err := pprof.StartCPUProfile(f); err != nil {
        panic(err)
    }
    defer pprof.StopCPUProfile()
    
    // Your code here
    expensiveFunction()
}

// Memory Profiling
func profileMemory() {
    f, err := os.Create("mem.prof")
    if err != nil {
        panic(err)
    }
    defer f.Close()
    
    runtime.GC() // Force garbage collection
    if err := pprof.WriteHeapProfile(f); err != nil {
        panic(err)
    }
}

// Trace Profiling
func profileTrace() {
    f, err := os.Create("trace.out")
    if err != nil {
        panic(err)
    }
    defer f.Close()
    
    if err := trace.Start(f); err != nil {
        panic(err)
    }
    defer trace.Stop()
    
    // Your code here
    expensiveFunction()
}

// Custom function timing
func timeFunction(name string, fn func()) {
    start := time.Now()
    var startMem runtime.MemStats
    runtime.ReadMemStats(&startMem)
    
    fn()
    
    duration := time.Since(start)
    var endMem runtime.MemStats
    runtime.ReadMemStats(&endMem)
    
    fmt.Printf("%s:\n", name)
    fmt.Printf("  Duration: %v\n", duration)
    fmt.Printf("  Allocs: %d\n", endMem.Mallocs-startMem.Mallocs)
    fmt.Printf("  Memory: %d bytes\n", endMem.TotalAlloc-startMem.TotalAlloc)
}

// HTTP profiling endpoint
func startProfilingServer() {
    go func() {
        fmt.Println("Profiling server started on :6060")
        fmt.Println("Visit http://localhost:6060/debug/pprof/")
        http.ListenAndServe(":6060", nil)
    }()
}

func main() {
    startProfilingServer()
    
    // Example usage
    timeFunction("ExpensiveOperation", func() {
        expensiveFunction()
    })
}
```

## 🛠️ Database Profiling

### **SQL Query Profiling**
```sql
-- PostgreSQL Query Profiling
-- Enable query logging
SET log_statement = 'all';
SET log_min_duration_statement = 100; -- Log queries > 100ms

-- Analyze query performance
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name
ORDER BY order_count DESC;

-- Query execution stats
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    stddev_time,
    rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- Index usage analysis
SELECT 
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE tablename = 'users';
```

### **MongoDB Profiling**
```javascript
// Enable profiling for slow operations (>100ms)
db.setProfilingLevel(1, { slowms: 100 });

// View profiling data
db.system.profile.find().limit(5).sort({ ts: -1 }).pretty();

// Analyze query performance
db.collection.explain("executionStats").find({
    "status": "active",
    "created_at": { $gte: new Date("2024-01-01") }
});

// Index usage statistics
db.collection.aggregate([
    { $indexStats: {} }
]);
```

## 🔍 Application Performance Monitoring (APM)

### **Custom Metrics Collection**
```python
import time
import psutil
import threading
from dataclasses import dataclass
from typing import Dict, List
from collections import defaultdict

@dataclass
class PerformanceMetric:
    name: str
    value: float
    timestamp: float
    tags: Dict[str, str] = None

class MetricsCollector:
    def __init__(self):
        self.metrics: List[PerformanceMetric] = []
        self.counters = defaultdict(int)
        self.timers = {}
        self.running = True
        
    def record_metric(self, name: str, value: float, tags: Dict[str, str] = None):
        metric = PerformanceMetric(
            name=name,
            value=value,
            timestamp=time.time(),
            tags=tags or {}
        )
        self.metrics.append(metric)
    
    def increment_counter(self, name: str, delta: int = 1):
        self.counters[name] += delta
    
    def start_timer(self, name: str):
        self.timers[name] = time.time()
    
    def end_timer(self, name: str) -> float:
        if name not in self.timers:
            return 0
        
        duration = time.time() - self.timers[name]
        self.record_metric(f"{name}_duration", duration * 1000)  # ms
        del self.timers[name]
        return duration
    
    def collect_system_metrics(self):
        # CPU usage
        cpu_percent = psutil.cpu_percent(interval=1)
        self.record_metric("cpu_usage_percent", cpu_percent)
        
        # Memory usage
        memory = psutil.virtual_memory()
        self.record_metric("memory_usage_percent", memory.percent)
        self.record_metric("memory_available_mb", memory.available / 1024 / 1024)
        
        # Disk usage
        disk = psutil.disk_usage('/')
        self.record_metric("disk_usage_percent", disk.percent)
        
    def start_background_collection(self):
        def collect_loop():
            while self.running:
                self.collect_system_metrics()
                time.sleep(10)  # Collect every 10 seconds
        
        thread = threading.Thread(target=collect_loop, daemon=True)
        thread.start()
    
    def get_metrics_summary(self) -> Dict:
        summary = {
            "total_metrics": len(self.metrics),
            "counters": dict(self.counters),
            "recent_metrics": self.metrics[-10:] if self.metrics else []
        }
        return summary

# Usage example
collector = MetricsCollector()
collector.start_background_collection()

def monitored_function():
    collector.start_timer("function_execution")
    collector.increment_counter("function_calls")
    
    try:
        # Your business logic
        time.sleep(0.1)  # Simulate work
        
        collector.increment_counter("function_success")
        return "success"
    
    except Exception as e:
        collector.increment_counter("function_errors")
        raise
    
    finally:
        collector.end_timer("function_execution")
```

### **Real-time Performance Dashboard**
```python
import asyncio
import json
from fastapi import FastAPI, WebSocket
from fastapi.responses import HTMLResponse
import psutil
import time

app = FastAPI()

class PerformanceDashboard:
    def __init__(self):
        self.connections = []
    
    async def add_connection(self, websocket: WebSocket):
        await websocket.accept()
        self.connections.append(websocket)
    
    async def remove_connection(self, websocket: WebSocket):
        if websocket in self.connections:
            self.connections.remove(websocket)
    
    async def broadcast_metrics(self, metrics: dict):
        disconnected = []
        for websocket in self.connections:
            try:
                await websocket.send_text(json.dumps(metrics))
            except:
                disconnected.append(websocket)
        
        # Remove disconnected clients
        for websocket in disconnected:
            await self.remove_connection(websocket)
    
    async def collect_and_broadcast(self):
        while True:
            try:
                metrics = {
                    "timestamp": time.time(),
                    "cpu_percent": psutil.cpu_percent(),
                    "memory_percent": psutil.virtual_memory().percent,
                    "disk_percent": psutil.disk_usage('/').percent,
                    "network_io": psutil.net_io_counters()._asdict(),
                    "process_count": len(psutil.pids())
                }
                
                await self.broadcast_metrics(metrics)
                await asyncio.sleep(1)  # Update every second
                
            except Exception as e:
                print(f"Error collecting metrics: {e}")
                await asyncio.sleep(5)

dashboard = PerformanceDashboard()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await dashboard.add_connection(websocket)
    try:
        while True:
            await websocket.receive_text()  # Keep connection alive
    except:
        await dashboard.remove_connection(websocket)

@app.on_event("startup")
async def startup_event():
    asyncio.create_task(dashboard.collect_and_broadcast())

@app.get("/")
async def get_dashboard():
    return HTMLResponse("""
    <!DOCTYPE html>
    <html>
    <head>
        <title>Performance Dashboard</title>
        <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    </head>
    <body>
        <div>
            <canvas id="cpuChart" width="400" height="200"></canvas>
            <canvas id="memoryChart" width="400" height="200"></canvas>
        </div>
        
        <script>
            const ws = new WebSocket("ws://localhost:8000/ws");
            
            const cpuCtx = document.getElementById('cpuChart').getContext('2d');
            const memoryCtx = document.getElementById('memoryChart').getContext('2d');
            
            const cpuChart = new Chart(cpuCtx, {
                type: 'line',
                data: {
                    labels: [],
                    datasets: [{
                        label: 'CPU Usage %',
                        data: [],
                        borderColor: 'rgb(75, 192, 192)',
                        tension: 0.1
                    }]
                },
                options: {
                    scales: {
                        y: { beginAtZero: true, max: 100 }
                    }
                }
            });
            
            const memoryChart = new Chart(memoryCtx, {
                type: 'line',
                data: {
                    labels: [],
                    datasets: [{
                        label: 'Memory Usage %',
                        data: [],
                        borderColor: 'rgb(255, 99, 132)',
                        tension: 0.1
                    }]
                },
                options: {
                    scales: {
                        y: { beginAtZero: true, max: 100 }
                    }
                }
            });
            
            ws.onmessage = function(event) {
                const data = JSON.parse(event.data);
                const time = new Date(data.timestamp * 1000).toLocaleTimeString();
                
                // Update CPU chart
                cpuChart.data.labels.push(time);
                cpuChart.data.datasets[0].data.push(data.cpu_percent);
                if (cpuChart.data.labels.length > 20) {
                    cpuChart.data.labels.shift();
                    cpuChart.data.datasets[0].data.shift();
                }
                cpuChart.update('none');
                
                // Update Memory chart
                memoryChart.data.labels.push(time);
                memoryChart.data.datasets[0].data.push(data.memory_percent);
                if (memoryChart.data.labels.length > 20) {
                    memoryChart.data.labels.shift();
                    memoryChart.data.datasets[0].data.shift();
                }
                memoryChart.update('none');
            };
        </script>
    </body>
    </html>
    """)
```

## 📊 Profiling Analysis và Interpretation

### **CPU Profile Analysis**
```
Flame Graph Interpretation:
1. Width = Time spent in function
2. Height = Call stack depth
3. Color = Different functions/modules

Key Patterns to Look For:
❌ Wide plateaus = CPU bottlenecks
❌ Deep stacks = Excessive recursion
❌ Unexpected functions = Hidden overhead
✅ Even distribution = Good performance
```

### **Memory Profile Analysis**
```
Memory Growth Patterns:
1. Linear growth = Potential memory leak
2. Sawtooth pattern = Normal GC behavior
3. Sudden spikes = Large allocations
4. Steady state = Healthy memory usage

Memory Leak Detection:
- Objects not being garbage collected
- Growing heap size over time
- High allocation rate without deallocation
- Retained references to unused objects
```

## 🚨 Common Profiling Pitfalls

### **Observer Effect**
```
Problem: Profiling overhead affects performance

Solutions:
✅ Use sampling profilers (not instrumentation)
✅ Profile in staging environment similar to production
✅ Use statistical profiling
✅ Minimize profiler overhead
✅ Profile representative workloads
```

### **Misleading Results**
```
Common Issues:
❌ Profiling debug builds instead of optimized
❌ Cold start effects (JIT compilation)
❌ Small sample sizes
❌ Artificial test data
❌ Different environment conditions

Best Practices:
✅ Warm up before profiling
✅ Profile production-like loads
✅ Use realistic data sets
✅ Profile multiple iterations
✅ Compare against baseline
```

## 💡 Best Practices

### **Profiling Strategy**
```
1. Establish Baseline
   - Measure current performance
   - Document normal behavior
   - Set performance goals

2. Systematic Approach
   - Profile incrementally
   - Focus on biggest bottlenecks first
   - Measure impact of changes

3. Continuous Profiling
   - Regular performance checks
   - Automated performance tests
   - Performance regression detection
```

### **Production Profiling**
```
Safe Production Profiling:
✅ Use low-overhead sampling profilers
✅ Limit profiling duration (5-10 minutes)
✅ Profile during low-traffic periods
✅ Have monitoring in place
✅ Quick rollback plan

Continuous Profiling:
✅ Always-on lightweight profiling
✅ Statistical sampling (1-10% overhead)
✅ Historical performance trends
✅ Automatic anomaly detection
```

---

**Checklist cho Performance Profiling:**
- [ ] Identify performance goals and metrics
- [ ] Choose appropriate profiling tools
- [ ] Set up profiling environment
- [ ] Collect baseline measurements
- [ ] Profile systematically (CPU, Memory, I/O)
- [ ] Analyze and interpret results
- [ ] Implement optimizations
- [ ] Verify improvements
- [ ] Set up continuous monitoring

**Nhớ rằng**: Profile first, optimize second - đừng đoán mò! 🔍 