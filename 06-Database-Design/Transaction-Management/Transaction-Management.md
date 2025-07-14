# Transaction Management

## Tổng quan

Transaction Management là hệ thống quản lý các operations trên database để đảm bảo:
- **Atomicity**: Tất cả operations trong transaction thành công hoặc rollback
- **Consistency**: Database luôn ở trạng thái consistent
- **Isolation**: Transactions không ảnh hưởng lẫn nhau
- **Durability**: Committed transactions được lưu trữ permanent

## ACID Properties

### 1. Atomicity

```python
import sqlite3
from contextlib import contextmanager

class AtomicTransaction:
    def __init__(self, db_connection):
        self.connection = db_connection
        self.cursor = db_connection.cursor()
    
    @contextmanager
    def transaction(self):
        """Context manager for atomic transactions"""
        try:
            self.connection.execute("BEGIN")
            yield self.cursor
            self.connection.execute("COMMIT")
        except Exception as e:
            self.connection.execute("ROLLBACK")
            raise e
    
    def transfer_money(self, from_account, to_account, amount):
        """Atomic money transfer"""
        with self.transaction() as cursor:
            # Check balance
            cursor.execute("SELECT balance FROM accounts WHERE id = ?", (from_account,))
            balance = cursor.fetchone()
            
            if not balance or balance[0] < amount:
                raise ValueError("Insufficient funds")
            
            # Debit from source account
            cursor.execute(
                "UPDATE accounts SET balance = balance - ? WHERE id = ?",
                (amount, from_account)
            )
            
            # Credit to destination account
            cursor.execute(
                "UPDATE accounts SET balance = balance + ? WHERE id = ?",
                (amount, to_account)
            )
            
            # Log transaction
            cursor.execute(
                "INSERT INTO transactions (from_account, to_account, amount) VALUES (?, ?, ?)",
                (from_account, to_account, amount)
            )

# Usage
conn = sqlite3.connect('bank.db')
atomic_tx = AtomicTransaction(conn)

try:
    atomic_tx.transfer_money(account_a=1, account_b=2, amount=100)
    print("Transfer successful")
except Exception as e:
    print(f"Transfer failed: {e}")
```

### 2. Consistency

```python
class ConsistencyManager:
    def __init__(self, db_connection):
        self.connection = db_connection
        self.constraints = []
    
    def add_constraint(self, constraint_func, error_message):
        """Add a consistency constraint"""
        self.constraints.append((constraint_func, error_message))
    
    def check_constraints(self):
        """Check all consistency constraints"""
        for constraint_func, error_message in self.constraints:
            if not constraint_func():
                raise ValueError(f"Consistency violation: {error_message}")
    
    def execute_with_consistency_check(self, operation):
        """Execute operation with consistency checks"""
        try:
            # Check pre-conditions
            self.check_constraints()
            
            # Execute operation
            result = operation()
            
            # Check post-conditions
            self.check_constraints()
            
            return result
        except Exception as e:
            # Rollback on consistency violation
            self.connection.rollback()
            raise e

# Example constraints
def check_account_balance_non_negative():
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM accounts WHERE balance < 0")
    return cursor.fetchone()[0] == 0

def check_total_balance_consistency():
    cursor = conn.cursor()
    cursor.execute("SELECT SUM(balance) FROM accounts")
    total_balance = cursor.fetchone()[0]
    cursor.execute("SELECT SUM(amount) FROM transactions")
    total_transactions = cursor.fetchone()[0] or 0
    
    return abs(total_balance - 1000000 + total_transactions) < 0.01  # Initial balance was 1M

# Usage
consistency_mgr = ConsistencyManager(conn)
consistency_mgr.add_constraint(check_account_balance_non_negative, "Account balance cannot be negative")
consistency_mgr.add_constraint(check_total_balance_consistency, "Total balance inconsistent")
```

### 3. Isolation

```python
import threading
import time
from enum import Enum

class IsolationLevel(Enum):
    READ_UNCOMMITTED = "READ UNCOMMITTED"
    READ_COMMITTED = "READ COMMITTED"
    REPEATABLE_READ = "REPEATABLE READ"
    SERIALIZABLE = "SERIALIZABLE"

class TransactionManager:
    def __init__(self, db_connection):
        self.connection = db_connection
        self.lock_manager = LockManager()
        self.active_transactions = {}
        self.transaction_counter = 0
    
    def begin_transaction(self, isolation_level=IsolationLevel.READ_COMMITTED):
        """Begin a new transaction"""
        tx_id = self.transaction_counter
        self.transaction_counter += 1
        
        transaction = Transaction(tx_id, isolation_level, self.connection, self.lock_manager)
        self.active_transactions[tx_id] = transaction
        
        return transaction
    
    def commit_transaction(self, tx_id):
        """Commit a transaction"""
        if tx_id in self.active_transactions:
            transaction = self.active_transactions[tx_id]
            transaction.commit()
            del self.active_transactions[tx_id]
    
    def rollback_transaction(self, tx_id):
        """Rollback a transaction"""
        if tx_id in self.active_transactions:
            transaction = self.active_transactions[tx_id]
            transaction.rollback()
            del self.active_transactions[tx_id]

class Transaction:
    def __init__(self, tx_id, isolation_level, connection, lock_manager):
        self.tx_id = tx_id
        self.isolation_level = isolation_level
        self.connection = connection
        self.lock_manager = lock_manager
        self.read_set = set()
        self.write_set = set()
        self.is_active = True
        self.start_time = time.time()
    
    def read(self, table, key):
        """Read with appropriate isolation level"""
        if not self.is_active:
            raise Exception("Transaction is not active")
        
        if self.isolation_level == IsolationLevel.READ_UNCOMMITTED:
            return self._read_uncommitted(table, key)
        elif self.isolation_level == IsolationLevel.READ_COMMITTED:
            return self._read_committed(table, key)
        elif self.isolation_level == IsolationLevel.REPEATABLE_READ:
            return self._repeatable_read(table, key)
        elif self.isolation_level == IsolationLevel.SERIALIZABLE:
            return self._serializable_read(table, key)
    
    def write(self, table, key, value):
        """Write with appropriate isolation level"""
        if not self.is_active:
            raise Exception("Transaction is not active")
        
        # Acquire write lock
        lock_key = f"{table}_{key}"
        if not self.lock_manager.acquire_write_lock(lock_key, self.tx_id):
            raise Exception("Could not acquire write lock")
        
        self.write_set.add(lock_key)
        
        # Perform write
        cursor = self.connection.cursor()
        cursor.execute(f"UPDATE {table} SET value = ? WHERE key = ?", (value, key))
    
    def _read_committed(self, table, key):
        """Read committed isolation level"""
        lock_key = f"{table}_{key}"
        
        try:
            # Acquire read lock
            if not self.lock_manager.acquire_read_lock(lock_key, self.tx_id):
                raise Exception("Could not acquire read lock")
            
            # Read value
            cursor = self.connection.cursor()
            cursor.execute(f"SELECT value FROM {table} WHERE key = ?", (key,))
            result = cursor.fetchone()
            
            return result[0] if result else None
        finally:
            # Release read lock immediately (READ COMMITTED)
            self.lock_manager.release_read_lock(lock_key, self.tx_id)
    
    def _repeatable_read(self, table, key):
        """Repeatable read isolation level"""
        lock_key = f"{table}_{key}"
        
        # Acquire read lock and keep it
        if not self.lock_manager.acquire_read_lock(lock_key, self.tx_id):
            raise Exception("Could not acquire read lock")
        
        self.read_set.add(lock_key)
        
        # Read value
        cursor = self.connection.cursor()
        cursor.execute(f"SELECT value FROM {table} WHERE key = ?", (key,))
        result = cursor.fetchone()
        
        return result[0] if result else None
    
    def _serializable_read(self, table, key):
        """Serializable isolation level"""
        # Same as repeatable read but with additional phantom read protection
        return self._repeatable_read(table, key)
    
    def commit(self):
        """Commit transaction"""
        if not self.is_active:
            return
        
        try:
            self.connection.commit()
            self.is_active = False
            
            # Release all locks
            for lock_key in self.read_set:
                self.lock_manager.release_read_lock(lock_key, self.tx_id)
            
            for lock_key in self.write_set:
                self.lock_manager.release_write_lock(lock_key, self.tx_id)
            
        except Exception as e:
            self.rollback()
            raise e
    
    def rollback(self):
        """Rollback transaction"""
        if not self.is_active:
            return
        
        self.connection.rollback()
        self.is_active = False
        
        # Release all locks
        for lock_key in self.read_set:
            self.lock_manager.release_read_lock(lock_key, self.tx_id)
        
        for lock_key in self.write_set:
            self.lock_manager.release_write_lock(lock_key, self.tx_id)

class LockManager:
    def __init__(self):
        self.read_locks = {}  # {resource: set(tx_ids)}
        self.write_locks = {}  # {resource: tx_id}
        self.lock = threading.Lock()
    
    def acquire_read_lock(self, resource, tx_id):
        """Acquire read lock on resource"""
        with self.lock:
            # Check if there's a conflicting write lock
            if resource in self.write_locks and self.write_locks[resource] != tx_id:
                return False
            
            # Acquire read lock
            if resource not in self.read_locks:
                self.read_locks[resource] = set()
            
            self.read_locks[resource].add(tx_id)
            return True
    
    def acquire_write_lock(self, resource, tx_id):
        """Acquire write lock on resource"""
        with self.lock:
            # Check if there's a conflicting write lock
            if resource in self.write_locks and self.write_locks[resource] != tx_id:
                return False
            
            # Check if there are conflicting read locks
            if resource in self.read_locks and len(self.read_locks[resource]) > 0:
                # Allow if this tx is the only reader
                if len(self.read_locks[resource]) > 1 or tx_id not in self.read_locks[resource]:
                    return False
            
            # Acquire write lock
            self.write_locks[resource] = tx_id
            return True
    
    def release_read_lock(self, resource, tx_id):
        """Release read lock on resource"""
        with self.lock:
            if resource in self.read_locks and tx_id in self.read_locks[resource]:
                self.read_locks[resource].remove(tx_id)
                if not self.read_locks[resource]:
                    del self.read_locks[resource]
    
    def release_write_lock(self, resource, tx_id):
        """Release write lock on resource"""
        with self.lock:
            if resource in self.write_locks and self.write_locks[resource] == tx_id:
                del self.write_locks[resource]
```

### 4. Durability

```python
import os
import json
import hashlib

class DurabilityManager:
    def __init__(self, db_connection, wal_file_path):
        self.connection = db_connection
        self.wal_file_path = wal_file_path
        self.wal_file = None
        self.checkpoint_interval = 100  # Checkpoint every 100 transactions
        self.transaction_count = 0
    
    def initialize_wal(self):
        """Initialize Write-Ahead Log"""
        self.wal_file = open(self.wal_file_path, 'a')
    
    def log_transaction(self, tx_id, operations):
        """Log transaction to WAL"""
        if not self.wal_file:
            self.initialize_wal()
        
        log_entry = {
            'tx_id': tx_id,
            'timestamp': time.time(),
            'operations': operations,
            'checksum': self._calculate_checksum(operations)
        }
        
        # Write to WAL
        self.wal_file.write(json.dumps(log_entry) + '\n')
        self.wal_file.flush()
        os.fsync(self.wal_file.fileno())  # Force write to disk
        
        self.transaction_count += 1
        
        # Checkpoint if needed
        if self.transaction_count % self.checkpoint_interval == 0:
            self.checkpoint()
    
    def _calculate_checksum(self, operations):
        """Calculate checksum for operations"""
        data = json.dumps(operations, sort_keys=True)
        return hashlib.md5(data.encode()).hexdigest()
    
    def checkpoint(self):
        """Checkpoint database"""
        try:
            # Flush all changes to database
            self.connection.commit()
            
            # Truncate WAL file
            self.wal_file.close()
            self.wal_file = None
            
            # Create new WAL file
            os.rename(self.wal_file_path, f"{self.wal_file_path}.old")
            self.initialize_wal()
            
            print(f"Checkpoint completed. WAL truncated after {self.transaction_count} transactions")
            
        except Exception as e:
            print(f"Checkpoint failed: {e}")
    
    def recover(self):
        """Recover from WAL after crash"""
        if not os.path.exists(self.wal_file_path):
            return
        
        with open(self.wal_file_path, 'r') as wal_file:
            for line in wal_file:
                try:
                    log_entry = json.loads(line.strip())
                    self._replay_transaction(log_entry)
                except json.JSONDecodeError:
                    print(f"Corrupted WAL entry: {line}")
                    continue
        
        print("Recovery completed")
    
    def _replay_transaction(self, log_entry):
        """Replay transaction from WAL"""
        tx_id = log_entry['tx_id']
        operations = log_entry['operations']
        
        # Verify checksum
        if log_entry['checksum'] != self._calculate_checksum(operations):
            print(f"Checksum mismatch for transaction {tx_id}")
            return
        
        # Replay operations
        cursor = self.connection.cursor()
        for operation in operations:
            cursor.execute(operation['sql'], operation['params'])
        
        self.connection.commit()
        print(f"Replayed transaction {tx_id}")

# Usage
durability_mgr = DurabilityManager(conn, 'database.wal')
durability_mgr.recover()  # Recover on startup
```

## Advanced Transaction Patterns

### 1. Distributed Transactions (2PC)

```python
class TwoPhaseCommitCoordinator:
    def __init__(self, participants):
        self.participants = participants
        self.transaction_id = None
    
    def execute_distributed_transaction(self, operations):
        """Execute distributed transaction using 2PC"""
        self.transaction_id = self._generate_transaction_id()
        
        try:
            # Phase 1: Prepare
            if not self._prepare_phase(operations):
                self._abort_transaction()
                return False
            
            # Phase 2: Commit
            self._commit_phase()
            return True
            
        except Exception as e:
            self._abort_transaction()
            raise e
    
    def _prepare_phase(self, operations):
        """Phase 1: Prepare"""
        prepared_participants = []
        
        for participant in self.participants:
            try:
                # Send prepare request
                if participant.prepare(self.transaction_id, operations):
                    prepared_participants.append(participant)
                else:
                    # Abort if any participant can't prepare
                    self._abort_prepared_participants(prepared_participants)
                    return False
            except Exception as e:
                self._abort_prepared_participants(prepared_participants)
                return False
        
        return True
    
    def _commit_phase(self):
        """Phase 2: Commit"""
        for participant in self.participants:
            try:
                participant.commit(self.transaction_id)
            except Exception as e:
                # Log error but continue with other participants
                print(f"Failed to commit on participant {participant}: {e}")
    
    def _abort_transaction(self):
        """Abort transaction on all participants"""
        for participant in self.participants:
            try:
                participant.abort(self.transaction_id)
            except Exception as e:
                print(f"Failed to abort on participant {participant}: {e}")
    
    def _abort_prepared_participants(self, prepared_participants):
        """Abort prepared participants"""
        for participant in prepared_participants:
            try:
                participant.abort(self.transaction_id)
            except Exception as e:
                print(f"Failed to abort prepared participant {participant}: {e}")
    
    def _generate_transaction_id(self):
        import uuid
        return str(uuid.uuid4())

class TransactionParticipant:
    def __init__(self, db_connection):
        self.connection = db_connection
        self.prepared_transactions = {}
    
    def prepare(self, transaction_id, operations):
        """Prepare phase implementation"""
        try:
            # Start transaction
            self.connection.execute("BEGIN")
            
            # Execute operations
            cursor = self.connection.cursor()
            for operation in operations:
                cursor.execute(operation['sql'], operation['params'])
            
            # Don't commit yet, just prepare
            self.prepared_transactions[transaction_id] = {
                'operations': operations,
                'prepared_at': time.time()
            }
            
            return True
            
        except Exception as e:
            self.connection.rollback()
            return False
    
    def commit(self, transaction_id):
        """Commit phase implementation"""
        if transaction_id in self.prepared_transactions:
            try:
                self.connection.commit()
                del self.prepared_transactions[transaction_id]
                return True
            except Exception as e:
                self.connection.rollback()
                raise e
        return False
    
    def abort(self, transaction_id):
        """Abort transaction"""
        if transaction_id in self.prepared_transactions:
            self.connection.rollback()
            del self.prepared_transactions[transaction_id]
        return True

# Usage
participant1 = TransactionParticipant(conn1)
participant2 = TransactionParticipant(conn2)

coordinator = TwoPhaseCommitCoordinator([participant1, participant2])

operations = [
    {'sql': 'UPDATE accounts SET balance = balance - 100 WHERE id = 1', 'params': []},
    {'sql': 'UPDATE accounts SET balance = balance + 100 WHERE id = 2', 'params': []}
]

success = coordinator.execute_distributed_transaction(operations)
```

### 2. Saga Pattern

```python
class SagaOrchestrator:
    def __init__(self):
        self.steps = []
        self.compensations = []
    
    def add_step(self, action, compensation):
        """Add a step to the saga"""
        self.steps.append(action)
        self.compensations.append(compensation)
    
    def execute(self):
        """Execute saga with compensations"""
        completed_steps = []
        
        try:
            # Execute all steps
            for i, step in enumerate(self.steps):
                result = step()
                completed_steps.append(i)
                
                if not result:
                    raise Exception(f"Step {i} failed")
            
            return True
            
        except Exception as e:
            # Compensate completed steps in reverse order
            self._compensate(completed_steps)
            raise e
    
    def _compensate(self, completed_steps):
        """Execute compensations for completed steps"""
        for step_index in reversed(completed_steps):
            try:
                compensation = self.compensations[step_index]
                compensation()
            except Exception as e:
                print(f"Compensation failed for step {step_index}: {e}")

# Example: E-commerce order saga
class OrderSaga:
    def __init__(self, order_id, user_id, amount):
        self.order_id = order_id
        self.user_id = user_id
        self.amount = amount
        self.saga = SagaOrchestrator()
        self._setup_saga()
    
    def _setup_saga(self):
        """Setup saga steps and compensations"""
        # Step 1: Reserve inventory
        self.saga.add_step(
            action=lambda: self._reserve_inventory(),
            compensation=lambda: self._release_inventory()
        )
        
        # Step 2: Process payment
        self.saga.add_step(
            action=lambda: self._process_payment(),
            compensation=lambda: self._refund_payment()
        )
        
        # Step 3: Create order
        self.saga.add_step(
            action=lambda: self._create_order(),
            compensation=lambda: self._cancel_order()
        )
        
        # Step 4: Send confirmation
        self.saga.add_step(
            action=lambda: self._send_confirmation(),
            compensation=lambda: self._send_cancellation()
        )
    
    def _reserve_inventory(self):
        # Reserve inventory logic
        print(f"Reserving inventory for order {self.order_id}")
        return True
    
    def _release_inventory(self):
        # Release inventory logic
        print(f"Releasing inventory for order {self.order_id}")
    
    def _process_payment(self):
        # Process payment logic
        print(f"Processing payment of ${self.amount} for user {self.user_id}")
        return True
    
    def _refund_payment(self):
        # Refund payment logic
        print(f"Refunding payment of ${self.amount} for user {self.user_id}")
    
    def _create_order(self):
        # Create order logic
        print(f"Creating order {self.order_id}")
        return True
    
    def _cancel_order(self):
        # Cancel order logic
        print(f"Cancelling order {self.order_id}")
    
    def _send_confirmation(self):
        # Send confirmation logic
        print(f"Sending confirmation for order {self.order_id}")
        return True
    
    def _send_cancellation(self):
        # Send cancellation logic
        print(f"Sending cancellation for order {self.order_id}")
    
    def execute(self):
        """Execute the order saga"""
        return self.saga.execute()

# Usage
order_saga = OrderSaga(order_id="ORDER123", user_id="USER456", amount=99.99)
try:
    order_saga.execute()
    print("Order completed successfully")
except Exception as e:
    print(f"Order failed: {e}")
```

## Transaction Monitoring

### 1. Deadlock Detection

```python
import networkx as nx
from collections import defaultdict

class DeadlockDetector:
    def __init__(self):
        self.wait_for_graph = defaultdict(set)
        self.lock_owners = {}  # {resource: tx_id}
        self.lock_waiters = defaultdict(set)  # {resource: set(tx_ids)}
    
    def add_lock_request(self, tx_id, resource):
        """Add a lock request"""
        if resource in self.lock_owners:
            owner = self.lock_owners[resource]
            if owner != tx_id:
                # Transaction is waiting for another transaction
                self.wait_for_graph[tx_id].add(owner)
                self.lock_waiters[resource].add(tx_id)
        else:
            # Grant lock immediately
            self.lock_owners[resource] = tx_id
    
    def release_lock(self, tx_id, resource):
        """Release a lock"""
        if resource in self.lock_owners and self.lock_owners[resource] == tx_id:
            del self.lock_owners[resource]
            
            # Remove from wait-for graph
            if tx_id in self.wait_for_graph:
                del self.wait_for_graph[tx_id]
            
            # Grant lock to next waiter
            if resource in self.lock_waiters and self.lock_waiters[resource]:
                next_tx = self.lock_waiters[resource].pop()
                self.lock_owners[resource] = next_tx
                
                # Remove from wait-for relationships
                for waiting_tx in self.wait_for_graph:
                    self.wait_for_graph[waiting_tx].discard(next_tx)
    
    def detect_deadlock(self):
        """Detect deadlocks using cycle detection"""
        # Create directed graph
        G = nx.DiGraph()
        
        for tx_id, waiting_for in self.wait_for_graph.items():
            for waiting_tx in waiting_for:
                G.add_edge(tx_id, waiting_tx)
        
        # Find cycles
        try:
            cycles = list(nx.simple_cycles(G))
            return cycles
        except nx.NetworkXNoCycle:
            return []
    
    def resolve_deadlock(self, cycle):
        """Resolve deadlock by aborting victim transaction"""
        # Choose victim (e.g., youngest transaction)
        victim = min(cycle)  # Simple strategy
        
        print(f"Deadlock detected: {cycle}")
        print(f"Aborting victim transaction: {victim}")
        
        # Abort victim transaction
        self._abort_transaction(victim)
        
        return victim
    
    def _abort_transaction(self, tx_id):
        """Abort transaction and release its locks"""
        # Release all locks held by this transaction
        resources_to_release = []
        for resource, owner in self.lock_owners.items():
            if owner == tx_id:
                resources_to_release.append(resource)
        
        for resource in resources_to_release:
            self.release_lock(tx_id, resource)
        
        # Remove from wait-for graph
        if tx_id in self.wait_for_graph:
            del self.wait_for_graph[tx_id]
        
        # Remove from all wait queues
        for resource in self.lock_waiters:
            self.lock_waiters[resource].discard(tx_id)

# Usage
deadlock_detector = DeadlockDetector()

# Simulate lock requests
deadlock_detector.add_lock_request(tx_id=1, resource='A')
deadlock_detector.add_lock_request(tx_id=2, resource='B')
deadlock_detector.add_lock_request(tx_id=1, resource='B')  # TX1 waits for TX2
deadlock_detector.add_lock_request(tx_id=2, resource='A')  # TX2 waits for TX1 - DEADLOCK!

# Detect and resolve deadlock
deadlocks = deadlock_detector.detect_deadlock()
if deadlocks:
    for cycle in deadlocks:
        deadlock_detector.resolve_deadlock(cycle)
```

### 2. Transaction Performance Monitoring

```python
class TransactionMonitor:
    def __init__(self):
        self.active_transactions = {}
        self.completed_transactions = []
        self.metrics = {
            'total_transactions': 0,
            'committed_transactions': 0,
            'aborted_transactions': 0,
            'avg_duration': 0,
            'deadlock_count': 0
        }
    
    def start_transaction(self, tx_id):
        """Start monitoring a transaction"""
        self.active_transactions[tx_id] = {
            'start_time': time.time(),
            'operations': [],
            'locks_held': set(),
            'status': 'active'
        }
    
    def log_operation(self, tx_id, operation_type, resource, duration):
        """Log an operation"""
        if tx_id in self.active_transactions:
            self.active_transactions[tx_id]['operations'].append({
                'type': operation_type,
                'resource': resource,
                'duration': duration,
                'timestamp': time.time()
            })
    
    def end_transaction(self, tx_id, status):
        """End transaction monitoring"""
        if tx_id in self.active_transactions:
            tx_info = self.active_transactions[tx_id]
            tx_info['end_time'] = time.time()
            tx_info['duration'] = tx_info['end_time'] - tx_info['start_time']
            tx_info['status'] = status
            
            # Move to completed transactions
            self.completed_transactions.append(tx_info)
            del self.active_transactions[tx_id]
            
            # Update metrics
            self.metrics['total_transactions'] += 1
            if status == 'committed':
                self.metrics['committed_transactions'] += 1
            elif status == 'aborted':
                self.metrics['aborted_transactions'] += 1
            
            # Update average duration
            total_duration = sum(tx['duration'] for tx in self.completed_transactions)
            self.metrics['avg_duration'] = total_duration / len(self.completed_transactions)
    
    def detect_long_running_transactions(self, threshold=30):
        """Detect long-running transactions"""
        current_time = time.time()
        long_running = []
        
        for tx_id, tx_info in self.active_transactions.items():
            duration = current_time - tx_info['start_time']
            if duration > threshold:
                long_running.append({
                    'tx_id': tx_id,
                    'duration': duration,
                    'operations': len(tx_info['operations'])
                })
        
        return long_running
    
    def generate_report(self):
        """Generate transaction report"""
        report = {
            'timestamp': time.time(),
            'active_transactions': len(self.active_transactions),
            'metrics': self.metrics,
            'long_running': self.detect_long_running_transactions()
        }
        
        if self.completed_transactions:
            # Calculate percentiles
            durations = [tx['duration'] for tx in self.completed_transactions[-1000:]]  # Last 1000 transactions
            durations.sort()
            
            report['performance'] = {
                'p50': durations[len(durations) // 2] if durations else 0,
                'p95': durations[int(len(durations) * 0.95)] if durations else 0,
                'p99': durations[int(len(durations) * 0.99)] if durations else 0
            }
        
        return report
```

Transaction Management là core component của database systems, đảm bảo data integrity và consistency trong môi trường concurrent access. 