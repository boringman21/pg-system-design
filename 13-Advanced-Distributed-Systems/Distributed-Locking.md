# Distributed Locking Mechanisms

**Tags**: #distributed-systems #locking #synchronization #concurrency #redis #zookeeper
**Date**: 2024-01-01

## 📝 Overview

Distributed locking cho phép multiple processes trên different machines coordinate access to shared resources. Đây là một primitive quan trọng trong distributed systems để ensure data consistency và avoid race conditions.

## 🎯 Problem Statement

### **Why Distributed Locking?**
- **Mutual Exclusion**: Chỉ một process có thể access resource tại một thời điểm
- **Coordination**: Synchronize operations across multiple services
- **Resource Protection**: Prevent concurrent modifications
- **Leader Election**: Choose a single leader trong cluster

### **Challenges**
- **Network Partitions**: Nodes có thể bị isolated
- **Node Failures**: Lock holder có thể crash
- **Clock Skew**: Different system clocks
- **False Sharing**: Multiple processes think they have the lock

## 🔒 Lock Requirements

### **Safety Properties**
1. **Mutual Exclusion**: Tại bất kỳ thời điểm nào, chỉ một client có thể hold lock
2. **Deadlock Freedom**: System không bao giờ enters deadlock state
3. **Fault Tolerance**: Lock service survives node failures

### **Liveness Properties**
1. **Lock Acquisition**: Clients eventually có thể acquire available locks
2. **Lock Release**: Locks are eventually released
3. **Progress**: System makes progress despite failures

## 🏗️ Implementation Approaches

### **1. Database-Based Locking**
```sql
-- Create locks table
CREATE TABLE distributed_locks (
    lock_name VARCHAR(255) PRIMARY KEY,
    holder_id VARCHAR(255) NOT NULL,
    acquired_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    version INT DEFAULT 1
);

-- Acquire lock
INSERT INTO distributed_locks (lock_name, holder_id, expires_at)
VALUES ('resource_lock', 'process_123', NOW() + INTERVAL 30 SECOND)
ON DUPLICATE KEY UPDATE 
    holder_id = CASE 
        WHEN expires_at < NOW() THEN VALUES(holder_id)
        ELSE holder_id
    END,
    expires_at = CASE 
        WHEN expires_at < NOW() THEN VALUES(expires_at)
        ELSE expires_at
    END,
    version = version + 1;

-- Release lock
DELETE FROM distributed_locks 
WHERE lock_name = 'resource_lock' AND holder_id = 'process_123';
```

```python
class DatabaseDistributedLock:
    def __init__(self, db_connection, lock_name, holder_id):
        self.db = db_connection
        self.lock_name = lock_name
        self.holder_id = holder_id
        self.acquired = False
        self.lock_timeout = 30  # seconds
    
    async def acquire(self, timeout=None):
        """Acquire distributed lock"""
        
        start_time = time.time()
        
        while True:
            try:
                # Try to acquire lock
                query = """
                INSERT INTO distributed_locks (lock_name, holder_id, expires_at)
                VALUES (%s, %s, %s)
                ON DUPLICATE KEY UPDATE 
                    holder_id = CASE 
                        WHEN expires_at < NOW() THEN VALUES(holder_id)
                        ELSE holder_id
                    END,
                    expires_at = CASE 
                        WHEN expires_at < NOW() THEN VALUES(expires_at)
                        ELSE expires_at
                    END,
                    affected_rows = ROW_COUNT()
                """
                
                expires_at = datetime.utcnow() + timedelta(seconds=self.lock_timeout)
                
                result = await self.db.execute(
                    query, 
                    (self.lock_name, self.holder_id, expires_at)
                )
                
                # Check if we got the lock
                if result.affected_rows > 0:
                    self.acquired = True
                    return True
                
                # Check timeout
                if timeout and (time.time() - start_time) > timeout:
                    return False
                
                # Wait before retry
                await asyncio.sleep(0.1)
                
            except Exception as e:
                logger.error(f"Error acquiring lock: {e}")
                return False
    
    async def release(self):
        """Release distributed lock"""
        
        if not self.acquired:
            return False
        
        try:
            query = """
            DELETE FROM distributed_locks 
            WHERE lock_name = %s AND holder_id = %s
            """
            
            result = await self.db.execute(query, (self.lock_name, self.holder_id))
            
            if result.affected_rows > 0:
                self.acquired = False
                return True
            
            return False
            
        except Exception as e:
            logger.error(f"Error releasing lock: {e}")
            return False
    
    async def extend_lock(self, additional_time=30):
        """Extend lock expiration time"""
        
        if not self.acquired:
            return False
        
        try:
            query = """
            UPDATE distributed_locks 
            SET expires_at = %s
            WHERE lock_name = %s AND holder_id = %s
            """
            
            new_expires_at = datetime.utcnow() + timedelta(seconds=additional_time)
            
            result = await self.db.execute(
                query, 
                (new_expires_at, self.lock_name, self.holder_id)
            )
            
            return result.affected_rows > 0
            
        except Exception as e:
            logger.error(f"Error extending lock: {e}")
            return False
    
    async def __aenter__(self):
        """Context manager entry"""
        success = await self.acquire()
        if not success:
            raise LockAcquisitionException(f"Could not acquire lock: {self.lock_name}")
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Context manager exit"""
        await self.release()
```

### **2. Redis-Based Locking**
```python
import redis
import time
import threading

class RedisDistributedLock:
    def __init__(self, redis_client, lock_name, holder_id=None):
        self.redis = redis_client
        self.lock_name = f"lock:{lock_name}"
        self.holder_id = holder_id or f"{socket.gethostname()}:{threading.current_thread().ident}"
        self.acquired = False
        self.lock_timeout = 30  # seconds
        self.renewal_thread = None
        self.stop_renewal = False
    
    async def acquire(self, timeout=None):
        """Acquire Redis distributed lock using SET with NX and EX"""
        
        start_time = time.time()
        
        while True:
            try:
                # Try to set lock with expiration
                result = await self.redis.set(
                    self.lock_name,
                    self.holder_id,
                    nx=True,  # Only set if not exists
                    ex=self.lock_timeout  # Set expiration
                )
                
                if result:
                    self.acquired = True
                    self.start_renewal_thread()
                    return True
                
                # Check timeout
                if timeout and (time.time() - start_time) > timeout:
                    return False
                
                # Wait before retry
                await asyncio.sleep(0.1)
                
            except Exception as e:
                logger.error(f"Error acquiring Redis lock: {e}")
                return False
    
    async def release(self):
        """Release Redis lock using Lua script for atomicity"""
        
        if not self.acquired:
            return False
        
        self.stop_renewal = True
        
        # Lua script to ensure atomic release
        release_script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        """
        
        try:
            result = await self.redis.eval(
                release_script,
                1,  # Number of keys
                self.lock_name,
                self.holder_id
            )
            
            if result:
                self.acquired = False
                return True
            
            return False
            
        except Exception as e:
            logger.error(f"Error releasing Redis lock: {e}")
            return False
    
    def start_renewal_thread(self):
        """Start background thread to renew lock"""
        
        def renewal_loop():
            while not self.stop_renewal and self.acquired:
                try:
                    # Renew lock before it expires
                    time.sleep(self.lock_timeout * 0.5)  # Renew at 50% of timeout
                    
                    if not self.stop_renewal:
                        self.extend_lock()
                        
                except Exception as e:
                    logger.error(f"Error in lock renewal: {e}")
                    break
        
        self.renewal_thread = threading.Thread(target=renewal_loop, daemon=True)
        self.renewal_thread.start()
    
    async def extend_lock(self):
        """Extend lock expiration using Lua script"""
        
        extend_script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("EXPIRE", KEYS[1], ARGV[2])
        else
            return 0
        end
        """
        
        try:
            result = await self.redis.eval(
                extend_script,
                1,  # Number of keys
                self.lock_name,
                self.holder_id,
                self.lock_timeout
            )
            
            return result == 1
            
        except Exception as e:
            logger.error(f"Error extending Redis lock: {e}")
            return False

class RedisRedlock:
    """Redlock algorithm implementation for Redis clusters"""
    
    def __init__(self, redis_instances, lock_name, holder_id=None):
        self.redis_instances = redis_instances
        self.lock_name = lock_name
        self.holder_id = holder_id or f"{socket.gethostname()}:{threading.current_thread().ident}"
        self.acquired_locks = []
        self.lock_timeout = 30000  # milliseconds
        self.drift_factor = 0.01
        self.retry_jitter = 200  # milliseconds
    
    async def acquire(self, retry_count=3):
        """Acquire Redlock across multiple Redis instances"""
        
        for attempt in range(retry_count):
            start_time = int(time.time() * 1000)
            acquired_count = 0
            
            # Try to acquire lock on each instance
            for redis_instance in self.redis_instances:
                try:
                    result = await redis_instance.set(
                        self.lock_name,
                        self.holder_id,
                        nx=True,
                        px=self.lock_timeout
                    )
                    
                    if result:
                        acquired_count += 1
                        self.acquired_locks.append(redis_instance)
                        
                except Exception as e:
                    logger.error(f"Error acquiring lock on Redis instance: {e}")
            
            # Calculate elapsed time
            elapsed_time = int(time.time() * 1000) - start_time
            
            # Check if we have majority and lock is still valid
            quorum = len(self.redis_instances) // 2 + 1
            
            if (acquired_count >= quorum and 
                elapsed_time < self.lock_timeout - self.drift_factor * self.lock_timeout):
                
                return True
            
            # Release acquired locks if we don't have quorum
            await self.release_partial_locks()
            
            # Wait before retry
            await asyncio.sleep(random.randint(0, self.retry_jitter) / 1000.0)
        
        return False
    
    async def release(self):
        """Release Redlock from all instances"""
        
        release_script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        """
        
        for redis_instance in self.acquired_locks:
            try:
                await redis_instance.eval(
                    release_script,
                    1,
                    self.lock_name,
                    self.holder_id
                )
            except Exception as e:
                logger.error(f"Error releasing lock on Redis instance: {e}")
        
        self.acquired_locks.clear()
    
    async def release_partial_locks(self):
        """Release partially acquired locks"""
        
        for redis_instance in self.acquired_locks:
            try:
                await redis_instance.delete(self.lock_name)
            except Exception as e:
                logger.error(f"Error releasing partial lock: {e}")
        
        self.acquired_locks.clear()
```

### **3. ZooKeeper-Based Locking**
```python
from kazoo.client import KazooClient
from kazoo.exceptions import NodeExistsError, NoNodeError

class ZooKeeperDistributedLock:
    def __init__(self, zk_client, lock_path):
        self.zk = zk_client
        self.lock_path = lock_path
        self.lock_node = None
        self.acquired = False
    
    async def acquire(self, timeout=None):
        """Acquire ZooKeeper distributed lock"""
        
        try:
            # Ensure lock path exists
            self.zk.ensure_path(self.lock_path)
            
            # Create ephemeral sequential node
            self.lock_node = self.zk.create(
                f"{self.lock_path}/lock-",
                ephemeral=True,
                sequence=True
            )
            
            # Extract sequence number
            sequence_num = int(self.lock_node.split('-')[-1])
            
            while True:
                # Get all children and sort
                children = self.zk.get_children(self.lock_path)
                children.sort(key=lambda x: int(x.split('-')[-1]))
                
                # Check if we have the smallest sequence number
                if children[0] == self.lock_node.split('/')[-1]:
                    self.acquired = True
                    return True
                
                # Find the node to watch (predecessor)
                predecessor = None
                for child in children:
                    child_seq = int(child.split('-')[-1])
                    if child_seq < sequence_num:
                        predecessor = child
                
                if predecessor:
                    # Watch predecessor node
                    predecessor_path = f"{self.lock_path}/{predecessor}"
                    
                    # Set watch and wait for notification
                    event = asyncio.Event()
                    
                    def watch_callback(event_obj):
                        event.set()
                    
                    if self.zk.exists(predecessor_path, watch=watch_callback):
                        # Wait for predecessor to be deleted
                        await asyncio.wait_for(event.wait(), timeout=timeout)
                    
                    # Check again
                    continue
                
                # If no predecessor, we should have the lock
                break
                
        except Exception as e:
            logger.error(f"Error acquiring ZooKeeper lock: {e}")
            return False
    
    async def release(self):
        """Release ZooKeeper lock"""
        
        if not self.acquired or not self.lock_node:
            return False
        
        try:
            self.zk.delete(self.lock_node)
            self.acquired = False
            self.lock_node = None
            return True
            
        except NoNodeError:
            # Node already deleted
            self.acquired = False
            self.lock_node = None
            return True
            
        except Exception as e:
            logger.error(f"Error releasing ZooKeeper lock: {e}")
            return False

class ZooKeeperLeaderElection:
    """Leader election using ZooKeeper"""
    
    def __init__(self, zk_client, election_path, candidate_id):
        self.zk = zk_client
        self.election_path = election_path
        self.candidate_id = candidate_id
        self.candidate_node = None
        self.is_leader = False
        self.leadership_callbacks = []
    
    async def start_election(self):
        """Start leader election process"""
        
        try:
            # Ensure election path exists
            self.zk.ensure_path(self.election_path)
            
            # Create ephemeral sequential node
            self.candidate_node = self.zk.create(
                f"{self.election_path}/candidate-",
                value=self.candidate_id.encode(),
                ephemeral=True,
                sequence=True
            )
            
            # Start monitoring for leadership
            await self.check_leadership()
            
        except Exception as e:
            logger.error(f"Error starting leader election: {e}")
            raise
    
    async def check_leadership(self):
        """Check if this node is the leader"""
        
        try:
            # Get all candidates
            candidates = self.zk.get_children(self.election_path)
            candidates.sort(key=lambda x: int(x.split('-')[-1]))
            
            # Check if we are the leader (smallest sequence number)
            if candidates[0] == self.candidate_node.split('/')[-1]:
                if not self.is_leader:
                    self.is_leader = True
                    await self.on_leadership_gained()
                return
            
            # Find predecessor to watch
            my_seq = int(self.candidate_node.split('-')[-1])
            predecessor = None
            
            for candidate in candidates:
                candidate_seq = int(candidate.split('-')[-1])
                if candidate_seq < my_seq:
                    predecessor = candidate
            
            if predecessor:
                predecessor_path = f"{self.election_path}/{predecessor}"
                
                # Watch predecessor
                event = asyncio.Event()
                
                def watch_callback(event_obj):
                    event.set()
                
                if self.zk.exists(predecessor_path, watch=watch_callback):
                    # Wait for predecessor to leave
                    await event.wait()
                    
                    # Check leadership again
                    await self.check_leadership()
                
        except Exception as e:
            logger.error(f"Error checking leadership: {e}")
    
    async def on_leadership_gained(self):
        """Called when this node becomes leader"""
        
        logger.info(f"Node {self.candidate_id} became leader")
        
        # Notify all callbacks
        for callback in self.leadership_callbacks:
            try:
                await callback(self.candidate_id)
            except Exception as e:
                logger.error(f"Error in leadership callback: {e}")
    
    def add_leadership_callback(self, callback):
        """Add callback to be called when leadership is gained"""
        self.leadership_callbacks.append(callback)
    
    async def resign_leadership(self):
        """Resign from leadership"""
        
        if self.candidate_node:
            try:
                self.zk.delete(self.candidate_node)
                self.candidate_node = None
                self.is_leader = False
                
            except NoNodeError:
                # Node already deleted
                pass
            
            except Exception as e:
                logger.error(f"Error resigning leadership: {e}")
                raise
```

## 🔧 Advanced Lock Features

### **Lock with Automatic Renewal**
```python
class AutoRenewingLock:
    def __init__(self, lock_implementation, renewal_interval=None):
        self.lock = lock_implementation
        self.renewal_interval = renewal_interval or (lock_implementation.lock_timeout * 0.5)
        self.renewal_task = None
        self.stop_renewal = False
    
    async def acquire(self, timeout=None):
        """Acquire lock with automatic renewal"""
        
        success = await self.lock.acquire(timeout)
        
        if success:
            self.start_renewal()
        
        return success
    
    async def release(self):
        """Release lock and stop renewal"""
        
        self.stop_renewal = True
        
        if self.renewal_task:
            self.renewal_task.cancel()
        
        return await self.lock.release()
    
    def start_renewal(self):
        """Start automatic renewal task"""
        
        async def renewal_loop():
            while not self.stop_renewal:
                try:
                    await asyncio.sleep(self.renewal_interval)
                    
                    if not self.stop_renewal:
                        await self.lock.extend_lock()
                        
                except asyncio.CancelledError:
                    break
                except Exception as e:
                    logger.error(f"Error in lock renewal: {e}")
                    break
        
        self.renewal_task = asyncio.create_task(renewal_loop())

class FairLock:
    """Fair lock implementation using queue"""
    
    def __init__(self, lock_implementation, queue_path):
        self.lock = lock_implementation
        self.queue_path = queue_path
        self.queue_position = None
    
    async def acquire(self, timeout=None):
        """Acquire lock fairly using queue"""
        
        # Join queue
        self.queue_position = await self.join_queue()
        
        # Wait for turn
        await self.wait_for_turn(timeout)
        
        # Acquire actual lock
        return await self.lock.acquire()
    
    async def join_queue(self):
        """Join the fair queue"""
        
        # Create queue entry
        queue_entry = {
            'id': str(uuid.uuid4()),
            'timestamp': time.time(),
            'holder': self.lock.holder_id
        }
        
        # Add to queue (implementation depends on backing store)
        return await self.add_to_queue(queue_entry)
    
    async def wait_for_turn(self, timeout=None):
        """Wait for turn in queue"""
        
        start_time = time.time()
        
        while True:
            # Check if it's our turn
            if await self.is_our_turn():
                break
            
            # Check timeout
            if timeout and (time.time() - start_time) > timeout:
                await self.leave_queue()
                raise TimeoutError("Timeout waiting for lock")
            
            # Wait before checking again
            await asyncio.sleep(0.1)
    
    async def is_our_turn(self):
        """Check if it's our turn in the queue"""
        
        queue = await self.get_queue()
        
        if not queue:
            return True
        
        # Check if we're first in queue
        return queue[0]['id'] == self.queue_position
    
    async def release(self):
        """Release lock and leave queue"""
        
        # Release actual lock
        result = await self.lock.release()
        
        # Leave queue
        await self.leave_queue()
        
        return result
```

### **Lock with Timeout và Retry**
```python
class RobustLock:
    def __init__(self, lock_implementation, max_retries=3, retry_delay=1.0):
        self.lock = lock_implementation
        self.max_retries = max_retries
        self.retry_delay = retry_delay
        self.backoff_factor = 1.5
    
    async def acquire(self, timeout=None):
        """Acquire lock with retry logic"""
        
        for attempt in range(self.max_retries):
            try:
                success = await self.lock.acquire(timeout)
                
                if success:
                    return True
                
                # Exponential backoff
                delay = self.retry_delay * (self.backoff_factor ** attempt)
                await asyncio.sleep(delay)
                
            except Exception as e:
                logger.warning(f"Lock acquisition attempt {attempt + 1} failed: {e}")
                
                if attempt == self.max_retries - 1:
                    raise
                
                # Wait before retry
                delay = self.retry_delay * (self.backoff_factor ** attempt)
                await asyncio.sleep(delay)
        
        return False
    
    async def acquire_with_circuit_breaker(self, timeout=None):
        """Acquire lock with circuit breaker pattern"""
        
        circuit_breaker = CircuitBreaker(
            failure_threshold=3,
            recovery_timeout=30,
            expected_exception=Exception
        )
        
        async def acquire_wrapper():
            return await self.lock.acquire(timeout)
        
        return await circuit_breaker.call(acquire_wrapper)

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60, expected_exception=Exception):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.expected_exception = expected_exception
        
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    async def call(self, func):
        """Call function with circuit breaker protection"""
        
        if self.state == 'OPEN':
            if self.should_attempt_reset():
                self.state = 'HALF_OPEN'
            else:
                raise CircuitBreakerOpenException("Circuit breaker is open")
        
        try:
            result = await func()
            self.on_success()
            return result
            
        except self.expected_exception as e:
            self.on_failure()
            raise
    
    def should_attempt_reset(self):
        """Check if we should attempt to reset circuit breaker"""
        
        if self.last_failure_time is None:
            return True
        
        return time.time() - self.last_failure_time >= self.recovery_timeout
    
    def on_success(self):
        """Handle successful operation"""
        
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def on_failure(self):
        """Handle failed operation"""
        
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
```

## 🔍 Lock Monitoring và Debugging

### **Lock Metrics**
```python
class LockMetrics:
    def __init__(self, metrics_client):
        self.metrics = metrics_client
    
    def record_lock_acquisition(self, lock_name, success, duration):
        """Record lock acquisition metrics"""
        
        self.metrics.counter('lock.acquisition.attempts').increment(
            tags={'lock_name': lock_name, 'success': success}
        )
        
        if success:
            self.metrics.histogram('lock.acquisition.duration').record(
                duration,
                tags={'lock_name': lock_name}
            )
    
    def record_lock_hold_time(self, lock_name, duration):
        """Record how long lock was held"""
        
        self.metrics.histogram('lock.hold.duration').record(
            duration,
            tags={'lock_name': lock_name}
        )
    
    def record_lock_contention(self, lock_name, waiting_count):
        """Record lock contention"""
        
        self.metrics.gauge('lock.contention.waiting').set(
            waiting_count,
            tags={'lock_name': lock_name}
        )

class LockDebugger:
    def __init__(self):
        self.lock_history = []
        self.active_locks = {}
    
    def log_lock_event(self, event_type, lock_name, holder_id, timestamp=None):
        """Log lock events for debugging"""
        
        event = {
            'type': event_type,
            'lock_name': lock_name,
            'holder_id': holder_id,
            'timestamp': timestamp or time.time(),
            'thread_id': threading.current_thread().ident
        }
        
        self.lock_history.append(event)
        
        # Keep only recent history
        if len(self.lock_history) > 1000:
            self.lock_history = self.lock_history[-1000:]
        
        # Update active locks
        if event_type == 'acquired':
            self.active_locks[lock_name] = event
        elif event_type == 'released':
            self.active_locks.pop(lock_name, None)
    
    def get_lock_timeline(self, lock_name):
        """Get timeline of events for specific lock"""
        
        return [event for event in self.lock_history 
                if event['lock_name'] == lock_name]
    
    def detect_deadlocks(self):
        """Detect potential deadlocks"""
        
        # Simple deadlock detection based on lock history
        # This is a simplified version
        
        waiting_for = {}
        
        for event in self.lock_history[-100:]:  # Recent events
            if event['type'] == 'waiting':
                waiting_for[event['holder_id']] = event['lock_name']
        
        # Check for cycles
        for holder_id, lock_name in waiting_for.items():
            if self.has_cycle(holder_id, lock_name, waiting_for):
                return True
        
        return False
    
    def has_cycle(self, start_holder, lock_name, waiting_for, visited=None):
        """Check for cycle in wait-for graph"""
        
        if visited is None:
            visited = set()
        
        if start_holder in visited:
            return True
        
        visited.add(start_holder)
        
        # Find who holds the lock we're waiting for
        lock_holder = self.active_locks.get(lock_name, {}).get('holder_id')
        
        if lock_holder and lock_holder in waiting_for:
            next_lock = waiting_for[lock_holder]
            return self.has_cycle(start_holder, next_lock, waiting_for, visited)
        
        return False
```

## 🎯 Best Practices

### **Lock Usage Guidelines**
```python
class LockBestPractices:
    """Best practices for using distributed locks"""
    
    @staticmethod
    def use_with_timeout():
        """Always use locks with timeout"""
        
        async def good_example():
            lock = RedisDistributedLock(redis_client, "resource_lock")
            
            # Good: Use timeout
            if await lock.acquire(timeout=30):
                try:
                    # Critical section
                    await do_work()
                finally:
                    await lock.release()
            else:
                # Handle timeout
                logger.warning("Could not acquire lock within timeout")
        
        async def bad_example():
            lock = RedisDistributedLock(redis_client, "resource_lock")
            
            # Bad: No timeout, could wait forever
            await lock.acquire()
            await do_work()
            await lock.release()
    
    @staticmethod
    def use_try_finally():
        """Always release locks in finally block"""
        
        async def good_example():
            lock = RedisDistributedLock(redis_client, "resource_lock")
            
            acquired = await lock.acquire(timeout=30)
            if acquired:
                try:
                    await do_work()
                finally:
                    await lock.release()  # Always release
        
        async def better_example():
            # Even better: Use context manager
            async with RedisDistributedLock(redis_client, "resource_lock"):
                await do_work()
    
    @staticmethod
    def keep_critical_section_small():
        """Keep critical sections as small as possible"""
        
        async def good_example():
            # Prepare data outside lock
            data = await prepare_data()
            
            async with RedisDistributedLock(redis_client, "resource_lock"):
                # Only critical operations inside lock
                await update_shared_resource(data)
        
        async def bad_example():
            async with RedisDistributedLock(redis_client, "resource_lock"):
                # Bad: Non-critical operations inside lock
                data = await prepare_data()  # This could take long
                await update_shared_resource(data)
    
    @staticmethod
    def use_hierarchical_locking():
        """Use consistent lock ordering to avoid deadlocks"""
        
        async def good_example():
            # Always acquire locks in same order
            lock_names = ["resource_a", "resource_b"]
            lock_names.sort()  # Consistent ordering
            
            locks = []
            try:
                for lock_name in lock_names:
                    lock = RedisDistributedLock(redis_client, lock_name)
                    await lock.acquire()
                    locks.append(lock)
                
                # Critical section
                await do_work_with_both_resources()
                
            finally:
                # Release in reverse order
                for lock in reversed(locks):
                    await lock.release()
```

## 🔗 Related Topics

- [[Consensus Algorithms]] - Achieving agreement in distributed systems
- [[CAP Theorem]] - Consistency vs availability trade-offs
- [[Database Replication]] - Coordinating distributed data
- [[Event-Driven Architecture]] - Coordination through events

## 📚 Further Reading

- "Designing Data-Intensive Applications" - Martin Kleppmann
- Redis distributed locking documentation
- ZooKeeper programming guide
- "Distributed Systems" - Maarten van Steen

---

**Key Takeaway**: Distributed locking is essential for coordination trong distributed systems, nhưng requires careful consideration of failure modes, performance, và consistency requirements. Choose implementation based on your specific needs: Redis cho simplicity, ZooKeeper cho strong consistency, databases cho existing infrastructure.

---

**Next Steps**: 
- [ ] Implement basic distributed lock
- [ ] Add lock monitoring
- [ ] Test failure scenarios
- [ ] Benchmark different implementations
- [ ] Practice deadlock detection