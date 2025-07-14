# Vector Clocks in Distributed Systems

**Tags**: #distributed-systems #vector-clocks #causality #ordering #lamport
**Date**: 2024-01-01

## 📝 Overview

Vector clocks là một mechanism để track causal relationships và establish partial ordering của events trong distributed systems. Chúng giúp xác định whether events happened concurrently hay có causal relationship.

## 🎯 Problem: Ordering Events in Distributed Systems

### **The Challenge**
Trong distributed systems:
- **No global clock**: Mỗi node có local clock riêng
- **Network delays**: Messages có thể arrive out of order
- **Causal relationships**: Cần track what happened before what
- **Concurrent events**: Identify truly concurrent operations

### **Why Physical Clocks Fail**
```python
# Example of why physical clocks don't work

# Node A (timestamp: 10:00:01)
send_message("Hello", to=NodeB)

# Node B (timestamp: 10:00:00 - clock skew!)
receive_message("Hello", from=NodeA)

# Logical problem: receive happened "before" send according to timestamps
```

## 🕐 Lamport Timestamps (Background)

### **Lamport Clock Algorithm**
```python
class LamportClock:
    def __init__(self):
        self.counter = 0
    
    def tick(self):
        """Increment clock on local event"""
        self.counter += 1
        return self.counter
    
    def send_message(self, message, to_node):
        """Send message with timestamp"""
        timestamp = self.tick()
        message_with_timestamp = {
            'content': message,
            'timestamp': timestamp,
            'from': self.node_id
        }
        self.network.send(message_with_timestamp, to_node)
        return timestamp
    
    def receive_message(self, message_with_timestamp):
        """Receive message and update clock"""
        received_timestamp = message_with_timestamp['timestamp']
        
        # Update clock: max(local_clock, received_timestamp) + 1
        self.counter = max(self.counter, received_timestamp) + 1
        
        return self.counter
    
    def compare_events(self, event1, event2):
        """Compare two events using Lamport timestamps"""
        if event1.timestamp < event2.timestamp:
            return "event1 happened before event2"
        elif event1.timestamp > event2.timestamp:
            return "event2 happened before event1"
        else:
            return "concurrent or same event"
```

### **Lamport Clock Limitations**
- **Cannot detect concurrency**: Same timestamp doesn't mean concurrent
- **False causality**: Lower timestamp doesn't guarantee "happened before"
- **No causal information**: Cannot determine if events are causally related

## 🔢 Vector Clocks

### **Vector Clock Structure**
```python
class VectorClock:
    def __init__(self, node_id, all_nodes):
        self.node_id = node_id
        self.all_nodes = all_nodes
        # Initialize vector with zeros for all nodes
        self.vector = {node: 0 for node in all_nodes}
    
    def tick(self):
        """Increment own counter on local event"""
        self.vector[self.node_id] += 1
        return self.vector.copy()
    
    def send_message(self, message, to_node):
        """Send message with vector clock"""
        # Increment counter for send event
        timestamp = self.tick()
        
        message_with_vector = {
            'content': message,
            'vector_clock': timestamp,
            'from': self.node_id
        }
        
        self.network.send(message_with_vector, to_node)
        return timestamp
    
    def receive_message(self, message_with_vector):
        """Receive message and update vector clock"""
        received_vector = message_with_vector['vector_clock']
        
        # Update vector: element-wise max + increment own counter
        for node in self.all_nodes:
            if node == self.node_id:
                # Increment own counter
                self.vector[node] += 1
            else:
                # Take max with received vector
                self.vector[node] = max(self.vector[node], received_vector[node])
        
        return self.vector.copy()
    
    def compare_vectors(self, vector1, vector2):
        """Compare two vector clocks"""
        
        # Check if vector1 <= vector2 (happened before)
        v1_before_v2 = all(vector1[node] <= vector2[node] for node in self.all_nodes)
        v1_strictly_before_v2 = v1_before_v2 and any(vector1[node] < vector2[node] for node in self.all_nodes)
        
        # Check if vector2 <= vector1 (happened before)
        v2_before_v1 = all(vector2[node] <= vector1[node] for node in self.all_nodes)
        v2_strictly_before_v1 = v2_before_v1 and any(vector2[node] < vector1[node] for node in self.all_nodes)
        
        if v1_strictly_before_v2:
            return "vector1 → vector2"  # vector1 happened before vector2
        elif v2_strictly_before_v1:
            return "vector2 → vector1"  # vector2 happened before vector1
        elif vector1 == vector2:
            return "vector1 = vector2"  # same event
        else:
            return "vector1 || vector2"  # concurrent events
    
    def is_concurrent(self, vector1, vector2):
        """Check if two events are concurrent"""
        return self.compare_vectors(vector1, vector2) == "vector1 || vector2"
    
    def happened_before(self, vector1, vector2):
        """Check if vector1 happened before vector2"""
        return self.compare_vectors(vector1, vector2) == "vector1 → vector2"
```

### **Vector Clock Example**
```python
# Example with 3 nodes: A, B, C
nodes = ['A', 'B', 'C']

# Initialize vector clocks
clock_A = VectorClock('A', nodes)  # [0, 0, 0]
clock_B = VectorClock('B', nodes)  # [0, 0, 0]
clock_C = VectorClock('C', nodes)  # [0, 0, 0]

# Sequence of events:

# 1. A sends message to B
clock_A.tick()  # A: [1, 0, 0]
msg_A_to_B = {'content': 'Hello', 'vector_clock': clock_A.vector}

# 2. B receives message from A
clock_B.receive_message(msg_A_to_B)  # B: [1, 1, 0]

# 3. C performs local event
clock_C.tick()  # C: [0, 0, 1]

# 4. B sends message to C
clock_B.tick()  # B: [1, 2, 0]
msg_B_to_C = {'content': 'Hi', 'vector_clock': clock_B.vector}

# 5. C receives message from B
clock_C.receive_message(msg_B_to_C)  # C: [1, 2, 2]

# Analysis:
# - A's event [1, 0, 0] happened before B's final state [1, 2, 0]
# - C's local event [0, 0, 1] was concurrent with A's event [1, 0, 0]
# - B's final state [1, 2, 0] happened before C's final state [1, 2, 2]
```

## 🔄 Practical Implementation

### **Efficient Vector Clock**
```python
import json
from collections import defaultdict

class EfficientVectorClock:
    """Optimized vector clock implementation"""
    
    def __init__(self, node_id):
        self.node_id = node_id
        self.vector = defaultdict(int)
        self.version = 0
    
    def tick(self):
        """Increment own counter"""
        self.vector[self.node_id] += 1
        self.version += 1
        return self.get_vector()
    
    def update(self, other_vector):
        """Update with another vector clock"""
        if isinstance(other_vector, dict):
            other_vector = EfficientVectorClock.from_dict(other_vector)
        
        # Update each component
        for node_id, timestamp in other_vector.vector.items():
            if node_id != self.node_id:
                self.vector[node_id] = max(self.vector[node_id], timestamp)
        
        # Increment own counter
        self.vector[self.node_id] += 1
        self.version += 1
        
        return self.get_vector()
    
    def get_vector(self):
        """Get vector as dictionary"""
        return dict(self.vector)
    
    def to_string(self):
        """Serialize to string"""
        return json.dumps(self.get_vector(), sort_keys=True)
    
    @classmethod
    def from_string(cls, vector_string, node_id):
        """Deserialize from string"""
        vector_dict = json.loads(vector_string)
        clock = cls(node_id)
        clock.vector = defaultdict(int, vector_dict)
        return clock
    
    @classmethod
    def from_dict(cls, vector_dict):
        """Create from dictionary"""
        clock = cls(None)
        clock.vector = defaultdict(int, vector_dict)
        return clock
    
    def causally_before(self, other_vector):
        """Check if this vector is causally before another"""
        if isinstance(other_vector, dict):
            other_vector = EfficientVectorClock.from_dict(other_vector)
        
        # Check if all components are ≤ and at least one is <
        all_nodes = set(self.vector.keys()) | set(other_vector.vector.keys())
        
        all_less_equal = True
        some_strictly_less = False
        
        for node in all_nodes:
            self_val = self.vector[node]
            other_val = other_vector.vector[node]
            
            if self_val > other_val:
                all_less_equal = False
                break
            elif self_val < other_val:
                some_strictly_less = True
        
        return all_less_equal and some_strictly_less
    
    def concurrent_with(self, other_vector):
        """Check if this vector is concurrent with another"""
        return (not self.causally_before(other_vector) and 
                not EfficientVectorClock.from_dict(other_vector).causally_before(self.get_vector()))
    
    def merge(self, other_vector):
        """Merge with another vector clock (element-wise max)"""
        if isinstance(other_vector, dict):
            other_vector = EfficientVectorClock.from_dict(other_vector)
        
        all_nodes = set(self.vector.keys()) | set(other_vector.vector.keys())
        
        merged = EfficientVectorClock(self.node_id)
        for node in all_nodes:
            merged.vector[node] = max(self.vector[node], other_vector.vector[node])
        
        return merged

class VectorClockManager:
    """Manages vector clocks for distributed system"""
    
    def __init__(self, node_id):
        self.node_id = node_id
        self.clock = EfficientVectorClock(node_id)
        self.event_log = []
        self.causal_history = {}
    
    def log_local_event(self, event_type, data=None):
        """Log local event"""
        timestamp = self.clock.tick()
        
        event = {
            'event_id': f"{self.node_id}_{self.clock.version}",
            'event_type': event_type,
            'node_id': self.node_id,
            'timestamp': timestamp,
            'data': data,
            'local_time': time.time()
        }
        
        self.event_log.append(event)
        self.causal_history[event['event_id']] = timestamp
        
        return event
    
    def send_message(self, message, to_node):
        """Send message with vector clock"""
        # Log send event
        send_event = self.log_local_event('message_send', {
            'to_node': to_node,
            'message': message
        })
        
        # Attach vector clock to message
        message_with_clock = {
            'content': message,
            'vector_clock': send_event['timestamp'],
            'from_node': self.node_id,
            'event_id': send_event['event_id']
        }
        
        return message_with_clock
    
    def receive_message(self, message_with_clock):
        """Receive message and update clock"""
        # Update clock with received vector
        self.clock.update(message_with_clock['vector_clock'])
        
        # Log receive event
        receive_event = self.log_local_event('message_receive', {
            'from_node': message_with_clock['from_node'],
            'message': message_with_clock['content'],
            'causal_dependency': message_with_clock['event_id']
        })
        
        return receive_event
    
    def get_causal_history(self, event_id):
        """Get all events that causally precede given event"""
        if event_id not in self.causal_history:
            return []
        
        event_vector = self.causal_history[event_id]
        causal_events = []
        
        for other_event_id, other_vector in self.causal_history.items():
            if other_event_id != event_id:
                other_clock = EfficientVectorClock.from_dict(other_vector)
                if other_clock.causally_before(event_vector):
                    causal_events.append(other_event_id)
        
        return causal_events
    
    def get_concurrent_events(self, event_id):
        """Get all events concurrent with given event"""
        if event_id not in self.causal_history:
            return []
        
        event_vector = self.causal_history[event_id]
        concurrent_events = []
        
        for other_event_id, other_vector in self.causal_history.items():
            if other_event_id != event_id:
                other_clock = EfficientVectorClock.from_dict(other_vector)
                if other_clock.concurrent_with(event_vector):
                    concurrent_events.append(other_event_id)
        
        return concurrent_events
```

## 🔍 Applications of Vector Clocks

### **1. Consistent Snapshots**
```python
class DistributedSnapshot:
    """Chandy-Lamport snapshot algorithm using vector clocks"""
    
    def __init__(self, node_id, all_nodes):
        self.node_id = node_id
        self.all_nodes = all_nodes
        self.clock_manager = VectorClockManager(node_id)
        self.snapshot_state = {}
        self.snapshot_in_progress = False
    
    def initiate_snapshot(self):
        """Initiate global snapshot"""
        if self.snapshot_in_progress:
            return
        
        self.snapshot_in_progress = True
        
        # Record own state
        snapshot_event = self.clock_manager.log_local_event('snapshot_start')
        self.snapshot_state = {
            'node_state': self.get_local_state(),
            'vector_clock': snapshot_event['timestamp'],
            'channel_states': {}
        }
        
        # Send marker messages to all other nodes
        for other_node in self.all_nodes:
            if other_node != self.node_id:
                marker_msg = self.clock_manager.send_message(
                    {'type': 'snapshot_marker'}, 
                    other_node
                )
                self.send_to_node(marker_msg, other_node)
        
        return self.snapshot_state
    
    def handle_snapshot_marker(self, marker_msg):
        """Handle snapshot marker from another node"""
        # Update clock
        self.clock_manager.receive_message(marker_msg)
        
        if not self.snapshot_in_progress:
            # First marker received - record state
            self.initiate_snapshot()
        
        # Record channel state
        from_node = marker_msg['from_node']
        self.snapshot_state['channel_states'][from_node] = \
            self.get_channel_state(from_node)
    
    def get_local_state(self):
        """Get current local state"""
        return {
            'data': self.local_data,
            'timestamp': time.time(),
            'event_log_size': len(self.clock_manager.event_log)
        }
    
    def get_channel_state(self, from_node):
        """Get state of channel from specific node"""
        # Return messages received from node since snapshot started
        return {
            'messages_in_transit': [],
            'last_message_timestamp': None
        }

class CausallyOrderedMulticast:
    """Causally ordered message delivery using vector clocks"""
    
    def __init__(self, node_id, all_nodes):
        self.node_id = node_id
        self.all_nodes = all_nodes
        self.clock_manager = VectorClockManager(node_id)
        self.pending_messages = []
        self.delivered_messages = []
    
    def multicast(self, message):
        """Multicast message to all nodes"""
        # Create message with vector clock
        msg_with_clock = self.clock_manager.send_message(message, 'all')
        
        # Send to all other nodes
        for other_node in self.all_nodes:
            if other_node != self.node_id:
                self.send_to_node(msg_with_clock, other_node)
        
        # Deliver to self immediately
        self.deliver_message(msg_with_clock)
    
    def receive_multicast(self, message_with_clock):
        """Receive multicast message"""
        # Update clock
        self.clock_manager.receive_message(message_with_clock)
        
        # Add to pending messages
        self.pending_messages.append(message_with_clock)
        
        # Try to deliver pending messages
        self.attempt_delivery()
    
    def attempt_delivery(self):
        """Attempt to deliver pending messages in causal order"""
        delivered_any = True
        
        while delivered_any:
            delivered_any = False
            
            for i, pending_msg in enumerate(self.pending_messages):
                if self.can_deliver(pending_msg):
                    # Deliver message
                    self.deliver_message(pending_msg)
                    
                    # Remove from pending
                    self.pending_messages.pop(i)
                    delivered_any = True
                    break
    
    def can_deliver(self, message):
        """Check if message can be delivered"""
        msg_vector = message['vector_clock']
        from_node = message['from_node']
        
        # Check if all causal dependencies are satisfied
        for node in self.all_nodes:
            if node == from_node:
                # From sender: should be exactly one more than last delivered
                last_delivered_count = self.get_last_delivered_count(from_node)
                if msg_vector[node] != last_delivered_count + 1:
                    return False
            else:
                # From other nodes: should be ≤ what we've delivered
                last_delivered_count = self.get_last_delivered_count(node)
                if msg_vector[node] > last_delivered_count:
                    return False
        
        return True
    
    def get_last_delivered_count(self, node):
        """Get count of last delivered message from node"""
        for msg in reversed(self.delivered_messages):
            if msg['from_node'] == node:
                return msg['vector_clock'][node]
        return 0
    
    def deliver_message(self, message):
        """Deliver message to application"""
        self.delivered_messages.append(message)
        
        # Notify application
        self.on_message_delivered(message)
    
    def on_message_delivered(self, message):
        """Callback when message is delivered"""
        print(f"Delivered message: {message['content']} from {message['from_node']}")
```

### **2. Conflict Detection**
```python
class ConflictDetector:
    """Detect conflicts using vector clocks"""
    
    def __init__(self):
        self.operations = {}
        self.conflicts = []
    
    def add_operation(self, operation_id, vector_clock, operation_type, resource):
        """Add operation with vector clock"""
        operation = {
            'id': operation_id,
            'vector_clock': vector_clock,
            'type': operation_type,
            'resource': resource,
            'timestamp': time.time()
        }
        
        self.operations[operation_id] = operation
        
        # Check for conflicts
        self.detect_conflicts(operation)
    
    def detect_conflicts(self, new_operation):
        """Detect conflicts with existing operations"""
        for op_id, existing_op in self.operations.items():
            if (op_id != new_operation['id'] and
                existing_op['resource'] == new_operation['resource']):
                
                # Check if operations are concurrent
                if self.are_concurrent(existing_op['vector_clock'], 
                                    new_operation['vector_clock']):
                    
                    # Check if operations conflict
                    if self.operations_conflict(existing_op, new_operation):
                        conflict = {
                            'operation1': existing_op,
                            'operation2': new_operation,
                            'resource': existing_op['resource'],
                            'detected_at': time.time()
                        }
                        self.conflicts.append(conflict)
    
    def are_concurrent(self, vector1, vector2):
        """Check if two vector clocks are concurrent"""
        clock1 = EfficientVectorClock.from_dict(vector1)
        return clock1.concurrent_with(vector2)
    
    def operations_conflict(self, op1, op2):
        """Check if two operations conflict"""
        # Write-write conflicts
        if op1['type'] == 'write' and op2['type'] == 'write':
            return True
        
        # Read-write conflicts
        if (op1['type'] == 'read' and op2['type'] == 'write') or \
           (op1['type'] == 'write' and op2['type'] == 'read'):
            return True
        
        # Read-read operations don't conflict
        return False
    
    def get_conflicts_for_resource(self, resource):
        """Get all conflicts for a specific resource"""
        return [conflict for conflict in self.conflicts 
                if conflict['resource'] == resource]
    
    def resolve_conflict(self, conflict, resolution):
        """Resolve conflict with given resolution"""
        conflict['resolution'] = resolution
        conflict['resolved_at'] = time.time()
        
        # Apply resolution logic
        if resolution == 'last_writer_wins':
            # Choose operation with latest timestamp
            if conflict['operation1']['timestamp'] > conflict['operation2']['timestamp']:
                winner = conflict['operation1']
            else:
                winner = conflict['operation2']
            
            conflict['winner'] = winner
        
        elif resolution == 'merge':
            # Merge operations if possible
            conflict['merged_operation'] = self.merge_operations(
                conflict['operation1'], 
                conflict['operation2']
            )
```

## 📊 Performance Optimizations

### **Compressed Vector Clocks**
```python
class CompressedVectorClock:
    """Compressed vector clock to reduce size"""
    
    def __init__(self, node_id, compression_threshold=10):
        self.node_id = node_id
        self.vector = {}
        self.compression_threshold = compression_threshold
    
    def compress(self):
        """Compress vector clock by removing old entries"""
        if len(self.vector) > self.compression_threshold:
            # Keep only most recent entries
            sorted_entries = sorted(self.vector.items(), 
                                  key=lambda x: x[1], 
                                  reverse=True)
            
            # Keep top N entries
            compressed_vector = dict(sorted_entries[:self.compression_threshold])
            
            # Always keep own entry
            if self.node_id not in compressed_vector:
                compressed_vector[self.node_id] = self.vector[self.node_id]
            
            self.vector = compressed_vector
    
    def tick(self):
        """Increment and compress if needed"""
        self.vector[self.node_id] = self.vector.get(self.node_id, 0) + 1
        
        if len(self.vector) > self.compression_threshold:
            self.compress()
        
        return self.vector.copy()

class AdaptiveVectorClock:
    """Adaptive vector clock that adjusts based on communication patterns"""
    
    def __init__(self, node_id):
        self.node_id = node_id
        self.vector = {}
        self.communication_frequency = {}
        self.last_communication = {}
        self.inactive_threshold = 3600  # 1 hour
    
    def update_communication_frequency(self, node):
        """Update communication frequency with node"""
        self.communication_frequency[node] = \
            self.communication_frequency.get(node, 0) + 1
        self.last_communication[node] = time.time()
    
    def prune_inactive_nodes(self):
        """Remove nodes that haven't communicated recently"""
        current_time = time.time()
        
        inactive_nodes = []
        for node, last_comm_time in self.last_communication.items():
            if current_time - last_comm_time > self.inactive_threshold:
                inactive_nodes.append(node)
        
        for node in inactive_nodes:
            if node in self.vector:
                del self.vector[node]
            if node in self.communication_frequency:
                del self.communication_frequency[node]
            if node in self.last_communication:
                del self.last_communication[node]
    
    def get_relevant_nodes(self):
        """Get nodes that are still relevant for tracking"""
        self.prune_inactive_nodes()
        
        # Sort by communication frequency
        relevant_nodes = sorted(
            self.communication_frequency.items(),
            key=lambda x: x[1],
            reverse=True
        )
        
        return [node for node, _ in relevant_nodes]
```

## 🎯 Best Practices

### **Vector Clock Guidelines**
```python
class VectorClockBestPractices:
    """Best practices for using vector clocks"""
    
    @staticmethod
    def minimize_vector_size():
        """Keep vector clocks as small as possible"""
        
        # Good: Only include active nodes
        active_nodes = get_active_nodes()
        clock = EfficientVectorClock('node1')
        
        # Bad: Including all possible nodes
        all_possible_nodes = get_all_possible_nodes()  # Could be huge
        
    @staticmethod
    def use_compression():
        """Use compression for large systems"""
        
        # Compress vectors periodically
        compressed_clock = CompressedVectorClock('node1', threshold=50)
        
        # Use adaptive pruning
        adaptive_clock = AdaptiveVectorClock('node1')
        
    @staticmethod
    def handle_node_failures():
        """Handle node failures gracefully"""
        
        def cleanup_failed_node(failed_node_id):
            # Remove from vector clock
            if failed_node_id in clock.vector:
                del clock.vector[failed_node_id]
            
            # Update node list
            active_nodes.remove(failed_node_id)
    
    @staticmethod
    def batch_updates():
        """Batch vector clock updates for efficiency"""
        
        # Good: Batch multiple updates
        updates = []
        for message in message_batch:
            updates.append(message['vector_clock'])
        
        # Apply all updates at once
        clock.batch_update(updates)
        
        # Bad: Update for each message separately
        for message in message_batch:
            clock.update(message['vector_clock'])
```

## 🔗 Related Topics

- [[Consensus Algorithms]] - Achieving agreement about event ordering
- [[Distributed Locking]] - Coordinating concurrent access
- [[CAP Theorem]] - Consistency guarantees in distributed systems
- [[Event-Driven Architecture]] - Causal event processing

## 📚 Further Reading

- "Time, Clocks, and the Ordering of Events" - Leslie Lamport
- "Distributed Systems: Principles and Paradigms" - Tanenbaum & van Steen
- "Designing Data-Intensive Applications" - Martin Kleppmann
- Vector Clock implementation in Apache Cassandra

---

**Key Takeaway**: Vector clocks provide a way to track causal relationships in distributed systems without relying on synchronized physical clocks. Chúng essential cho building systems that need to understand event ordering và detect concurrent operations.

---

**Next Steps**: 
- [ ] Implement basic vector clock
- [ ] Add compression optimizations
- [ ] Test with distributed system
- [ ] Practice conflict detection
- [ ] Study real-world applications