# Distributed Systems Fundamentals

**Tags**: #distributed-systems #fundamentals #scalability #fault-tolerance
**Date**: 2024-01-01

## 📝 Overview

Distributed systems là collection of independent computers kết nối qua network, hoạt động như một unified system. Chúng là foundation của modern large-scale applications, enabling scalability, fault tolerance, và performance optimizations.

## 🎯 Core Characteristics

### **Fundamental Properties**
```
1. No shared memory
   - Components communicate via message passing
   - Each node has own local memory
   - Coordination requires explicit communication

2. No shared clock
   - Each node has own clock
   - Clock synchronization challenges
   - Causal ordering becomes critical

3. Independent failure modes
   - Nodes can fail independently
   - Network partitions possible
   - Partial system failures

4. Concurrency
   - Multiple nodes operate simultaneously
   - Race conditions across network
   - Distributed coordination needed
```

### **Benefits vs Challenges**
```python
distributed_systems_tradeoffs = {
    "benefits": {
        "scalability": "Handle more load by adding nodes",
        "fault_tolerance": "System continues despite individual failures",
        "performance": "Parallel processing and geographic distribution",
        "resource_sharing": "Utilize distributed computing resources",
        "geographic_distribution": "Serve users globally with low latency"
    },
    "challenges": {
        "complexity": "Much harder to design, implement, and debug",
        "consistency": "Maintaining data consistency across nodes",
        "network_issues": "Latency, packet loss, partitions",
        "partial_failures": "Some components fail while others continue",
        "concurrency_control": "Coordinating concurrent operations",
        "security": "Securing communication across untrusted networks"
    }
}
```

## 🌐 Distributed System Models

### **Communication Models**
```python
class DistributedCommunication:
    """
    Different communication patterns in distributed systems
    """
    
    def __init__(self):
        self.models = {
            "synchronous": {
                "description": "Nodes operate in lock-step",
                "benefits": ["Easier reasoning", "Strong consistency"],
                "drawbacks": ["Poor fault tolerance", "Slow as slowest node"],
                "use_cases": ["High-consistency requirements", "Small clusters"]
            },
            "asynchronous": {
                "description": "Nodes operate independently",
                "benefits": ["Better fault tolerance", "Higher performance"],
                "drawbacks": ["Complex coordination", "Eventual consistency"],
                "use_cases": ["Large-scale systems", "High availability"]
            }
        }
    
    def send_message_sync(self, sender, receiver, message):
        """
        Synchronous message passing
        """
        try:
            # Sender blocks until receiver acknowledges
            response = receiver.process_message(message)
            return response
        except TimeoutError:
            # In synchronous model, timeout indicates failure
            raise NodeFailureError("Receiver failed to respond")
    
    def send_message_async(self, sender, receiver, message):
        """
        Asynchronous message passing
        """
        # Sender doesn't wait for response
        message_queue.put((receiver.id, message))
        
        # Optional callback for response handling
        def handle_response(response):
            if response.success:
                self.process_success(response)
            else:
                self.handle_failure(response)
        
        return handle_response

class NetworkModel:
    """
    Network behavior assumptions
    """
    def __init__(self, model_type):
        self.model_type = model_type
        self.properties = self.get_model_properties()
    
    def get_model_properties(self):
        models = {
            "reliable": {
                "message_loss": False,
                "message_duplication": False,
                "message_reordering": False,
                "practical": False  # Idealized model
            },
            "fair_loss": {
                "message_loss": True,
                "message_duplication": False,
                "message_reordering": False,
                "retransmission": "Required for reliability"
            },
            "arbitrary": {
                "message_loss": True,
                "message_duplication": True,
                "message_reordering": True,
                "practical": True  # Real networks
            }
        }
        return models.get(self.model_type, models["arbitrary"])
```

### **Failure Models**
```python
class FailureModel:
    """
    Types of failures in distributed systems
    """
    
    def __init__(self):
        self.failure_types = {
            "crash_stop": {
                "description": "Node stops and never recovers",
                "detection": "Eventually detectable via timeouts",
                "handling": "Relatively straightforward",
                "examples": ["Server power failure", "Process crash"]
            },
            "crash_recovery": {
                "description": "Node stops but may restart",
                "detection": "Difficult to distinguish from slow node",
                "handling": "Need to handle restarts gracefully",
                "examples": ["Server reboot", "Network reconnection"]
            },
            "byzantine": {
                "description": "Node behaves arbitrarily/maliciously",
                "detection": "Very difficult, requires redundancy",
                "handling": "Complex protocols, voting mechanisms",
                "examples": ["Corrupted data", "Malicious actors", "Software bugs"]
            },
            "network_partition": {
                "description": "Network splits into isolated groups",
                "detection": "Nodes can't communicate across partition",
                "handling": "Choose consistency or availability",
                "examples": ["Router failure", "Network congestion"]
            }
        }
    
    def simulate_crash_stop_failure(self, node):
        """
        Simulate crash-stop failure
        """
        node.status = "failed"
        node.stop_processing()
        
        # Other nodes detect failure via timeout
        for other_node in self.cluster.nodes:
            if other_node != node:
                other_node.mark_peer_suspected(node.id)
    
    def simulate_network_partition(self, cluster, partition_groups):
        """
        Simulate network partition
        """
        for group1 in partition_groups:
            for group2 in partition_groups:
                if group1 != group2:
                    # Block communication between groups
                    for node1 in group1:
                        for node2 in group2:
                            cluster.block_communication(node1, node2)
    
    def handle_byzantine_behavior(self, suspicious_node, votes):
        """
        Handle potential Byzantine failure using voting
        """
        # Collect votes from majority of nodes
        honest_votes = 0
        total_votes = len(votes)
        
        for vote in votes:
            if self.verify_vote_authenticity(vote):
                honest_votes += 1
        
        # Byzantine fault tolerance: need >2/3 honest nodes
        if honest_votes > (2 * total_votes) / 3:
            return "consensus_reached"
        else:
            return "insufficient_honest_nodes"

class FailureDetector:
    """
    Detect node failures in distributed system
    """
    def __init__(self, timeout=5, retry_count=3):
        self.timeout = timeout
        self.retry_count = retry_count
        self.suspected_nodes = set()
        self.failed_nodes = set()
    
    def ping_node(self, node_id):
        """
        Send heartbeat to check if node is alive
        """
        for attempt in range(self.retry_count):
            try:
                response = self.send_ping(node_id, timeout=self.timeout)
                if response.success:
                    # Node is alive, remove from suspected
                    self.suspected_nodes.discard(node_id)
                    return True
            except TimeoutError:
                continue
        
        # All pings failed
        self.suspected_nodes.add(node_id)
        
        # If suspected for too long, mark as failed
        if self.is_long_suspected(node_id):
            self.failed_nodes.add(node_id)
            self.notify_failure(node_id)
        
        return False
    
    def periodic_failure_detection(self):
        """
        Periodically check all nodes in cluster
        """
        while True:
            for node_id in self.cluster.get_node_ids():
                if node_id not in self.failed_nodes:
                    self.ping_node(node_id)
            
            time.sleep(self.timeout)
```

## ⏰ Time & Ordering

### **Logical Clocks**
```python
class LamportClock:
    """
    Logical clock for ordering events in distributed systems
    """
    def __init__(self, node_id):
        self.node_id = node_id
        self.timestamp = 0
    
    def tick(self):
        """
        Increment clock on local event
        """
        self.timestamp += 1
        return self.timestamp
    
    def send_message(self, message, destination):
        """
        Send message with timestamp
        """
        self.tick()
        message.timestamp = self.timestamp
        message.sender_id = self.node_id
        
        self.network.send(message, destination)
        return message.timestamp
    
    def receive_message(self, message):
        """
        Update clock on message receipt
        """
        # Update clock: max(local_time, message_time) + 1
        self.timestamp = max(self.timestamp, message.timestamp) + 1
        
        # Process message
        self.process_message(message)
        return self.timestamp
    
    def compare_events(self, event1, event2):
        """
        Compare two events using happens-before relation
        """
        if event1.timestamp < event2.timestamp:
            return "event1_before_event2"
        elif event1.timestamp > event2.timestamp:
            return "event2_before_event1"
        else:
            # Concurrent events
            if event1.node_id < event2.node_id:
                return "event1_before_event2"  # Tie-breaking
            else:
                return "event2_before_event1"

class VectorClock:
    """
    Vector clock for better causality tracking
    """
    def __init__(self, node_id, cluster_size):
        self.node_id = node_id
        self.vector = [0] * cluster_size
        self.cluster_size = cluster_size
    
    def tick(self):
        """
        Increment own component
        """
        self.vector[self.node_id] += 1
        return self.vector.copy()
    
    def send_message(self, message, destination):
        """
        Send message with vector timestamp
        """
        self.tick()
        message.vector_timestamp = self.vector.copy()
        message.sender_id = self.node_id
        
        self.network.send(message, destination)
        return message.vector_timestamp
    
    def receive_message(self, message):
        """
        Update vector clock on message receipt
        """
        # Update each component: max(local[i], message[i])
        for i in range(self.cluster_size):
            if i == self.node_id:
                self.vector[i] += 1  # Increment own
            else:
                self.vector[i] = max(self.vector[i], message.vector_timestamp[i])
        
        self.process_message(message)
        return self.vector.copy()
    
    def happens_before(self, vector1, vector2):
        """
        Check if vector1 happens before vector2
        """
        # vector1 < vector2 if vector1[i] <= vector2[i] for all i
        # and vector1[j] < vector2[j] for at least one j
        
        less_equal_all = all(vector1[i] <= vector2[i] for i in range(self.cluster_size))
        strictly_less_one = any(vector1[i] < vector2[i] for i in range(self.cluster_size))
        
        return less_equal_all and strictly_less_one
    
    def concurrent(self, vector1, vector2):
        """
        Check if two events are concurrent
        """
        return (not self.happens_before(vector1, vector2) and 
                not self.happens_before(vector2, vector1))
```

### **Distributed Consensus**
```python
class RaftConsensus:
    """
    Raft consensus algorithm implementation
    """
    def __init__(self, node_id, cluster_nodes):
        self.node_id = node_id
        self.cluster_nodes = cluster_nodes
        self.state = "follower"  # follower, candidate, leader
        self.current_term = 0
        self.voted_for = None
        self.log = []
        self.commit_index = 0
        self.last_applied = 0
        
        # Leader state
        self.next_index = {}
        self.match_index = {}
    
    def start_election(self):
        """
        Start leader election process
        """
        self.state = "candidate"
        self.current_term += 1
        self.voted_for = self.node_id
        
        # Vote for self
        votes_received = 1
        
        # Request votes from other nodes
        for node_id in self.cluster_nodes:
            if node_id != self.node_id:
                vote_response = self.request_vote(node_id)
                if vote_response.vote_granted:
                    votes_received += 1
        
        # Check if won election
        majority = len(self.cluster_nodes) // 2 + 1
        if votes_received >= majority:
            self.become_leader()
        else:
            self.state = "follower"
    
    def request_vote(self, candidate_node):
        """
        Handle vote request from candidate
        """
        response = VoteResponse()
        response.term = self.current_term
        
        # Vote conditions
        if (candidate_node.term > self.current_term and
            (self.voted_for is None or self.voted_for == candidate_node.id) and
            self.is_log_up_to_date(candidate_node.last_log_index, candidate_node.last_log_term)):
            
            self.voted_for = candidate_node.id
            self.current_term = candidate_node.term
            response.vote_granted = True
        else:
            response.vote_granted = False
        
        return response
    
    def append_entries(self, entries, leader_id):
        """
        Handle append entries from leader (heartbeat/log replication)
        """
        response = AppendEntriesResponse()
        response.term = self.current_term
        
        # Reply false if term < currentTerm
        if entries.term < self.current_term:
            response.success = False
            return response
        
        # Update term and convert to follower
        if entries.term > self.current_term:
            self.current_term = entries.term
            self.voted_for = None
        
        self.state = "follower"
        self.leader_id = leader_id
        
        # Check log consistency
        if (entries.prev_log_index > 0 and
            (len(self.log) < entries.prev_log_index or
             self.log[entries.prev_log_index - 1].term != entries.prev_log_term)):
            response.success = False
            return response
        
        # Delete conflicting entries and append new ones
        if entries.entries:
            # Find insertion point
            insert_index = entries.prev_log_index
            
            # Delete conflicting entries
            self.log = self.log[:insert_index]
            
            # Append new entries
            self.log.extend(entries.entries)
        
        # Update commit index
        if entries.leader_commit > self.commit_index:
            self.commit_index = min(entries.leader_commit, len(self.log))
            self.apply_committed_entries()
        
        response.success = True
        return response
    
    def replicate_log_entry(self, entry):
        """
        Leader replicates log entry to followers
        """
        if self.state != "leader":
            return False
        
        # Add to own log
        entry.term = self.current_term
        entry.index = len(self.log)
        self.log.append(entry)
        
        # Replicate to followers
        success_count = 1  # Leader counts as success
        
        for follower_id in self.cluster_nodes:
            if follower_id != self.node_id:
                if self.send_append_entries(follower_id, [entry]):
                    success_count += 1
        
        # Check if majority replicated
        majority = len(self.cluster_nodes) // 2 + 1
        if success_count >= majority:
            self.commit_index = entry.index
            self.apply_committed_entries()
            return True
        
        return False

class PaxosConsensus:
    """
    Basic Paxos consensus algorithm
    """
    def __init__(self, node_id, acceptors):
        self.node_id = node_id
        self.acceptors = acceptors
        self.proposal_number = 0
        self.promised_proposals = {}
        self.accepted_proposals = {}
    
    def propose_value(self, value):
        """
        Propose a value using Paxos protocol
        """
        # Phase 1: Prepare
        self.proposal_number += 1
        proposal_id = (self.proposal_number, self.node_id)
        
        promises = []
        for acceptor in self.acceptors:
            promise = acceptor.prepare(proposal_id)
            if promise.success:
                promises.append(promise)
        
        # Check if majority promised
        if len(promises) < len(self.acceptors) // 2 + 1:
            return False  # Failed to get majority
        
        # Phase 2: Accept
        # Choose value: if any acceptor already accepted a value,
        # use the value from the highest-numbered proposal
        chosen_value = value
        highest_proposal = None
        
        for promise in promises:
            if (promise.accepted_proposal and 
                (highest_proposal is None or 
                 promise.accepted_proposal.number > highest_proposal.number)):
                highest_proposal = promise.accepted_proposal
                chosen_value = promise.accepted_proposal.value
        
        # Send accept requests
        accepts = []
        for acceptor in self.acceptors:
            accept = acceptor.accept(proposal_id, chosen_value)
            if accept.success:
                accepts.append(accept)
        
        # Check if majority accepted
        if len(accepts) >= len(self.acceptors) // 2 + 1:
            # Value chosen!
            self.notify_learners(chosen_value)
            return True
        
        return False
```

## 🔄 Distributed Transactions

### **Two-Phase Commit (2PC)**
```python
class TwoPhaseCommitCoordinator:
    """
    Coordinator for two-phase commit protocol
    """
    def __init__(self, transaction_id, participants):
        self.transaction_id = transaction_id
        self.participants = participants
        self.state = "active"
    
    def commit_transaction(self, operations):
        """
        Execute two-phase commit protocol
        """
        try:
            # Phase 1: Prepare/Vote
            prepare_responses = []
            
            for participant in self.participants:
                response = participant.prepare(self.transaction_id, operations[participant.id])
                prepare_responses.append((participant.id, response))
            
            # Check if all participants voted to commit
            all_prepared = all(response.vote == "commit" for _, response in prepare_responses)
            
            if all_prepared:
                # Phase 2: Commit
                self.state = "committing"
                
                for participant in self.participants:
                    participant.commit(self.transaction_id)
                
                self.state = "committed"
                return TransactionResult("committed")
            else:
                # Phase 2: Abort
                self.state = "aborting"
                
                for participant in self.participants:
                    participant.abort(self.transaction_id)
                
                self.state = "aborted"
                return TransactionResult("aborted")
                
        except Exception as e:
            # Handle coordinator failure
            self.state = "failed"
            
            # Abort transaction on all reachable participants
            for participant in self.participants:
                try:
                    participant.abort(self.transaction_id)
                except:
                    # Participant may be down, skip
                    pass
            
            return TransactionResult("failed", str(e))

class TwoPhaseCommitParticipant:
    """
    Participant in two-phase commit protocol
    """
    def __init__(self, participant_id):
        self.participant_id = participant_id
        self.transactions = {}  # transaction_id -> state
        self.prepared_operations = {}
    
    def prepare(self, transaction_id, operations):
        """
        Prepare phase: vote on whether to commit
        """
        try:
            # Validate operations
            if not self.validate_operations(operations):
                return PrepareResponse("abort", "Invalid operations")
            
            # Check resource availability
            if not self.check_resources(operations):
                return PrepareResponse("abort", "Insufficient resources")
            
            # Prepare resources (acquire locks, reserve space, etc.)
            self.prepare_resources(operations)
            
            # Save prepared state to durable storage
            self.prepared_operations[transaction_id] = operations
            self.transactions[transaction_id] = "prepared"
            self.persist_state(transaction_id, "prepared")
            
            return PrepareResponse("commit", "Ready to commit")
            
        except Exception as e:
            return PrepareResponse("abort", f"Preparation failed: {str(e)}")
    
    def commit(self, transaction_id):
        """
        Commit phase: actually apply the changes
        """
        if transaction_id not in self.transactions:
            raise ValueError(f"Unknown transaction: {transaction_id}")
        
        if self.transactions[transaction_id] != "prepared":
            raise ValueError(f"Transaction not prepared: {transaction_id}")
        
        try:
            # Apply the operations
            operations = self.prepared_operations[transaction_id]
            self.apply_operations(operations)
            
            # Update state
            self.transactions[transaction_id] = "committed"
            self.persist_state(transaction_id, "committed")
            
            # Release resources
            self.cleanup_transaction(transaction_id)
            
        except Exception as e:
            # This shouldn't happen if prepare was successful
            self.transactions[transaction_id] = "failed"
            raise
    
    def abort(self, transaction_id):
        """
        Abort phase: undo prepared changes
        """
        if transaction_id in self.transactions:
            # Undo any prepared changes
            if transaction_id in self.prepared_operations:
                self.undo_prepared_operations(transaction_id)
            
            # Update state
            self.transactions[transaction_id] = "aborted"
            self.persist_state(transaction_id, "aborted")
            
            # Release resources
            self.cleanup_transaction(transaction_id)
```

## 🔗 Liên kết và Tham khảo

### **Related Topics**
- [[01-Fundamentals/CAP-Theorem/CAP-Theorem|CAP Theorem]] - Fundamental trade-offs in distributed systems
- [[01-Fundamentals/Consistency-Models/Consistency-Models|Consistency Models]] - Different consistency guarantees
- [[07-Microservices/Microservices-Architecture|Microservices Architecture]] - Practical distributed system design
- [[06-Database-Design/Replication/Database-Replication|Database Replication]] - Distributing data across nodes

### **Distributed System Challenges**
- **Network**: Latency, bandwidth, partitions
- **Consensus**: Agreement in asynchronous systems
- **Coordination**: Distributed locks, leader election
- **Monitoring**: Observability in distributed environments

### **Key Algorithms**
- **Consensus**: Raft, PBFT, Paxos
- **Consistency**: Vector clocks, Lamport timestamps
- **Replication**: Primary-backup, chain replication
- **Load Balancing**: Consistent hashing, rendezvous hashing

---

*Distributed systems offer scalability và fault tolerance but introduce complexity. Understanding fundamental principles là crucial cho designing robust distributed applications.*

## 📚 Further Reading

- "Designing Data-Intensive Applications" by Martin Kleppmann
- "Distributed Systems: Concepts and Design" by Coulouris
- Leslie Lamport's papers on distributed systems
- Google's distributed systems papers (MapReduce, Bigtable, Spanner)

---

**Key Takeaway**: Distributed systems enable powerful scalability và fault tolerance nhưng introduce fundamental challenges around consistency, coordination, và failure handling. Success requires understanding trade-offs và choosing appropriate consistency models, consensus algorithms, và failure detection mechanisms for your specific use case. 