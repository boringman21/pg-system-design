# Consensus Algorithms in Distributed Systems

**Tags**: #distributed-systems #consensus #raft #paxos #blockchain #advanced
**Date**: 2024-01-01

## 📝 Overview

Consensus algorithms cho phép multiple nodes trong distributed system đồng thuận về một giá trị hoặc quyết định, ngay cả khi một số nodes fail hoặc network partitions xảy ra. Đây là nền tảng của nhiều hệ thống phân tán hiện đại.

## 🎯 Consensus Problem

### **The Problem**
Trong distributed system, làm sao để ensure tất cả nodes agree về:
- Thứ tự của operations
- Current state của system
- Leadership selection
- Configuration changes

### **Challenges**
- **Network Partitions**: Nodes có thể bị isolated
- **Node Failures**: Nodes có thể crash hoặc become unresponsive
- **Message Delays**: Network latency không predictable
- **Byzantine Failures**: Nodes có thể behave maliciously

## 🔄 Raft Consensus Algorithm

### **Core Concepts**
```python
class RaftNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.state = 'follower'  # follower, candidate, leader
        self.current_term = 0
        self.voted_for = None
        self.log = []
        self.commit_index = 0
        self.last_applied = 0
        
        # Leader-specific state
        self.next_index = {}
        self.match_index = {}
        
        # Timers
        self.election_timeout = random.uniform(150, 300)  # ms
        self.heartbeat_interval = 50  # ms
        self.last_heartbeat = time.time()

    def start_election(self):
        """Start leader election when timeout occurs"""
        self.state = 'candidate'
        self.current_term += 1
        self.voted_for = self.node_id
        self.last_heartbeat = time.time()
        
        # Vote for self
        votes = 1
        
        # Request votes from other nodes
        for peer in self.peers:
            if self.request_vote(peer):
                votes += 1
        
        # Become leader if majority votes
        if votes > len(self.peers) // 2:
            self.become_leader()
        else:
            self.state = 'follower'

    def request_vote(self, peer):
        """Request vote from peer node"""
        request = {
            'term': self.current_term,
            'candidate_id': self.node_id,
            'last_log_index': len(self.log) - 1,
            'last_log_term': self.log[-1]['term'] if self.log else 0
        }
        
        response = peer.handle_vote_request(request)
        return response.get('vote_granted', False)

    def handle_vote_request(self, request):
        """Handle vote request from candidate"""
        term = request['term']
        candidate_id = request['candidate_id']
        
        # Update term if newer
        if term > self.current_term:
            self.current_term = term
            self.voted_for = None
            self.state = 'follower'
        
        # Grant vote if conditions met
        vote_granted = (
            term == self.current_term and
            (self.voted_for is None or self.voted_for == candidate_id) and
            self.log_is_up_to_date(request['last_log_index'], request['last_log_term'])
        )
        
        if vote_granted:
            self.voted_for = candidate_id
            self.last_heartbeat = time.time()
        
        return {
            'term': self.current_term,
            'vote_granted': vote_granted
        }

    def become_leader(self):
        """Become leader and start sending heartbeats"""
        self.state = 'leader'
        
        # Initialize leader state
        for peer in self.peers:
            self.next_index[peer.node_id] = len(self.log)
            self.match_index[peer.node_id] = 0
        
        # Start sending heartbeats
        self.send_heartbeats()

    def send_heartbeats(self):
        """Send heartbeat to all followers"""
        for peer in self.peers:
            self.send_append_entries(peer)
```

### **Log Replication**
```python
def append_entries(self, peer):
    """Append entries to peer's log"""
    prev_log_index = self.next_index[peer.node_id] - 1
    prev_log_term = self.log[prev_log_index]['term'] if prev_log_index >= 0 else 0
    
    # Entries to send
    entries = self.log[self.next_index[peer.node_id]:]
    
    request = {
        'term': self.current_term,
        'leader_id': self.node_id,
        'prev_log_index': prev_log_index,
        'prev_log_term': prev_log_term,
        'entries': entries,
        'leader_commit': self.commit_index
    }
    
    response = peer.handle_append_entries(request)
    
    if response['success']:
        # Update next_index and match_index
        self.next_index[peer.node_id] = prev_log_index + len(entries) + 1
        self.match_index[peer.node_id] = prev_log_index + len(entries)
        
        # Update commit index if majority replicated
        self.update_commit_index()
    else:
        # Decrement next_index and retry
        self.next_index[peer.node_id] = max(0, self.next_index[peer.node_id] - 1)

def handle_append_entries(self, request):
    """Handle append entries from leader"""
    term = request['term']
    leader_id = request['leader_id']
    prev_log_index = request['prev_log_index']
    prev_log_term = request['prev_log_term']
    entries = request['entries']
    leader_commit = request['leader_commit']
    
    # Update term if newer
    if term > self.current_term:
        self.current_term = term
        self.voted_for = None
    
    # Reset election timeout
    self.last_heartbeat = time.time()
    self.state = 'follower'
    
    # Check log consistency
    if prev_log_index >= len(self.log) or \
       (prev_log_index >= 0 and self.log[prev_log_index]['term'] != prev_log_term):
        return {'term': self.current_term, 'success': False}
    
    # Append entries
    self.log = self.log[:prev_log_index + 1] + entries
    
    # Update commit index
    if leader_commit > self.commit_index:
        self.commit_index = min(leader_commit, len(self.log) - 1)
    
    return {'term': self.current_term, 'success': True}

def update_commit_index(self):
    """Update commit index based on majority replication"""
    for i in range(len(self.log) - 1, self.commit_index, -1):
        if self.log[i]['term'] == self.current_term:
            # Count how many nodes have replicated this entry
            count = 1  # Leader always has the entry
            for peer_id in self.match_index:
                if self.match_index[peer_id] >= i:
                    count += 1
            
            # If majority replicated, update commit index
            if count > len(self.peers) // 2:
                self.commit_index = i
                break
```

## 🌐 Paxos Algorithm

### **Basic Paxos**
```python
class PaxosNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.proposal_number = 0
        self.promised_proposals = {}
        self.accepted_proposals = {}
        self.chosen_values = {}

    def propose(self, value):
        """Propose a value using Paxos"""
        # Phase 1: Prepare
        self.proposal_number += 1
        proposal_id = f"{self.proposal_number}-{self.node_id}"
        
        prepare_responses = []
        for peer in self.peers:
            response = peer.handle_prepare(proposal_id)
            prepare_responses.append(response)
        
        # Count promises
        promises = [r for r in prepare_responses if r.get('promised')]
        
        if len(promises) <= len(self.peers) // 2:
            return False  # Not enough promises
        
        # Phase 2: Accept
        # Use highest-numbered accepted value, or our value if none
        highest_accepted = max(
            [p for p in promises if 'accepted_value' in p],
            key=lambda x: x['accepted_proposal'],
            default=None
        )
        
        value_to_propose = highest_accepted['accepted_value'] if highest_accepted else value
        
        accept_responses = []
        for peer in self.peers:
            response = peer.handle_accept(proposal_id, value_to_propose)
            accept_responses.append(response)
        
        # Count acceptances
        acceptances = [r for r in accept_responses if r.get('accepted')]
        
        if len(acceptances) > len(self.peers) // 2:
            # Value is chosen
            self.chosen_values[proposal_id] = value_to_propose
            return True
        
        return False

    def handle_prepare(self, proposal_id):
        """Handle prepare request"""
        if proposal_id > self.promised_proposals.get('id', ''):
            # Promise not to accept lower-numbered proposals
            self.promised_proposals['id'] = proposal_id
            
            # Return previously accepted value if any
            response = {'promised': True}
            if 'accepted_id' in self.accepted_proposals:
                response['accepted_proposal'] = self.accepted_proposals['accepted_id']
                response['accepted_value'] = self.accepted_proposals['accepted_value']
            
            return response
        else:
            return {'promised': False}

    def handle_accept(self, proposal_id, value):
        """Handle accept request"""
        if proposal_id >= self.promised_proposals.get('id', ''):
            # Accept the proposal
            self.accepted_proposals = {
                'accepted_id': proposal_id,
                'accepted_value': value
            }
            return {'accepted': True}
        else:
            return {'accepted': False}
```

### **Multi-Paxos**
```python
class MultiPaxosNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.leader = None
        self.log = []
        self.next_slot = 0
        self.decisions = {}
        self.proposals = {}

    def propose(self, command):
        """Propose a command in the next available slot"""
        if self.leader != self.node_id:
            # Forward to leader
            return self.forward_to_leader(command)
        
        # Find next available slot
        slot = self.next_slot
        while slot in self.decisions:
            slot += 1
        
        # Run Paxos for this slot
        proposal_id = f"{slot}-{self.node_id}"
        
        if self.run_paxos(slot, proposal_id, command):
            self.decisions[slot] = command
            self.execute_commands()
            return True
        
        return False

    def run_paxos(self, slot, proposal_id, command):
        """Run Paxos protocol for a specific slot"""
        # Similar to basic Paxos but for specific slot
        # ... (implementation details)
        pass

    def execute_commands(self):
        """Execute commands in order"""
        while self.next_slot in self.decisions:
            command = self.decisions[self.next_slot]
            self.execute_command(command)
            self.next_slot += 1
```

## 🔐 Byzantine Fault Tolerance

### **PBFT (Practical Byzantine Fault Tolerance)**
```python
class PBFTNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.view = 0
        self.sequence_number = 0
        self.primary = 0
        self.prepared_messages = {}
        self.committed_messages = {}
        self.executed_requests = set()

    def client_request(self, request):
        """Handle client request"""
        if self.node_id != self.primary:
            # Forward to primary
            return self.forward_to_primary(request)
        
        # Primary creates pre-prepare message
        self.sequence_number += 1
        msg = {
            'type': 'pre-prepare',
            'view': self.view,
            'sequence': self.sequence_number,
            'request': request,
            'primary': self.primary
        }
        
        # Send to all backups
        for peer in self.peers:
            if peer.node_id != self.node_id:
                peer.handle_pre_prepare(msg)
        
        # Process locally
        self.handle_pre_prepare(msg)

    def handle_pre_prepare(self, msg):
        """Handle pre-prepare message"""
        if not self.validate_pre_prepare(msg):
            return
        
        # Store message
        key = (msg['view'], msg['sequence'])
        self.prepared_messages[key] = msg
        
        # Send prepare to all nodes
        prepare_msg = {
            'type': 'prepare',
            'view': msg['view'],
            'sequence': msg['sequence'],
            'request_digest': self.hash_request(msg['request']),
            'node_id': self.node_id
        }
        
        self.broadcast_message(prepare_msg)

    def handle_prepare(self, msg):
        """Handle prepare message"""
        key = (msg['view'], msg['sequence'])
        
        if key not in self.prepared_messages:
            return
        
        # Count prepare messages
        prepare_count = self.count_prepare_messages(key)
        
        # If 2f+1 prepare messages, send commit
        if prepare_count >= 2 * self.byzantine_fault_tolerance() + 1:
            commit_msg = {
                'type': 'commit',
                'view': msg['view'],
                'sequence': msg['sequence'],
                'request_digest': msg['request_digest'],
                'node_id': self.node_id
            }
            
            self.broadcast_message(commit_msg)

    def handle_commit(self, msg):
        """Handle commit message"""
        key = (msg['view'], msg['sequence'])
        
        # Count commit messages
        commit_count = self.count_commit_messages(key)
        
        # If 2f+1 commit messages, execute request
        if commit_count >= 2 * self.byzantine_fault_tolerance() + 1:
            request = self.prepared_messages[key]['request']
            self.execute_request(request)

    def byzantine_fault_tolerance(self):
        """Maximum number of Byzantine failures tolerated"""
        return (len(self.peers) - 1) // 3
```

## 🔄 Practical Implementations

### **Raft in etcd**
```python
class EtcdRaftIntegration:
    def __init__(self):
        self.raft_node = RaftNode()
        self.state_machine = KeyValueStore()
        self.snapshot_manager = SnapshotManager()
    
    def put(self, key, value):
        """Put key-value pair"""
        if not self.raft_node.is_leader():
            return self.forward_to_leader({'put': {'key': key, 'value': value}})
        
        # Propose to Raft
        entry = {
            'type': 'put',
            'key': key,
            'value': value,
            'term': self.raft_node.current_term
        }
        
        return self.raft_node.propose_entry(entry)
    
    def apply_entry(self, entry):
        """Apply committed entry to state machine"""
        if entry['type'] == 'put':
            self.state_machine.put(entry['key'], entry['value'])
        elif entry['type'] == 'delete':
            self.state_machine.delete(entry['key'])
        
        # Create snapshot periodically
        if self.should_create_snapshot():
            self.snapshot_manager.create_snapshot(self.state_machine)
```

### **Consensus in Blockchain**
```python
class BlockchainConsensus:
    def __init__(self):
        self.blockchain = []
        self.pending_transactions = []
        self.difficulty = 4
        self.mining_reward = 50
    
    def proof_of_work(self, block):
        """Proof of Work consensus"""
        nonce = 0
        while True:
            block['nonce'] = nonce
            block_hash = self.calculate_hash(block)
            
            if block_hash.startswith('0' * self.difficulty):
                return block_hash
            
            nonce += 1
    
    def proof_of_stake(self, validators):
        """Proof of Stake consensus"""
        # Select validator based on stake
        total_stake = sum(v['stake'] for v in validators)
        selection_point = random.randint(0, total_stake - 1)
        
        current_stake = 0
        for validator in validators:
            current_stake += validator['stake']
            if current_stake > selection_point:
                return validator
    
    def delegated_proof_of_stake(self, delegates):
        """Delegated Proof of Stake"""
        # Round-robin among elected delegates
        current_slot = int(time.time()) // 3  # 3-second slots
        delegate_index = current_slot % len(delegates)
        return delegates[delegate_index]
```

## 📊 Performance Comparison

### **Throughput & Latency**
```python
class ConsensusMetrics:
    def __init__(self):
        self.metrics = {
            'raft': {
                'throughput': '10K-100K ops/sec',
                'latency': '1-10ms',
                'fault_tolerance': 'f < n/2',
                'network_complexity': 'O(n)'
            },
            'paxos': {
                'throughput': '5K-50K ops/sec',
                'latency': '2-20ms',
                'fault_tolerance': 'f < n/2',
                'network_complexity': 'O(n²)'
            },
            'pbft': {
                'throughput': '1K-10K ops/sec',
                'latency': '10-100ms',
                'fault_tolerance': 'f < n/3',
                'network_complexity': 'O(n²)'
            }
        }
    
    def benchmark_consensus(self, algorithm, num_nodes, num_operations):
        """Benchmark consensus algorithm"""
        start_time = time.time()
        
        # Run operations
        for i in range(num_operations):
            algorithm.propose(f"operation_{i}")
        
        end_time = time.time()
        
        return {
            'throughput': num_operations / (end_time - start_time),
            'latency': (end_time - start_time) / num_operations,
            'success_rate': algorithm.get_success_rate()
        }
```

## 🎯 Choosing the Right Algorithm

### **Decision Matrix**
```python
def choose_consensus_algorithm(requirements):
    """Choose appropriate consensus algorithm"""
    
    if requirements.get('byzantine_faults'):
        if requirements.get('performance') == 'high':
            return 'HotStuff'  # High-performance BFT
        else:
            return 'PBFT'
    
    if requirements.get('consistency') == 'strong':
        if requirements.get('partition_tolerance') == 'high':
            return 'Raft'  # Strong consistency, partition tolerant
        else:
            return 'Paxos'  # Strong consistency, less partition tolerant
    
    if requirements.get('availability') == 'high':
        return 'Gossip'  # Eventually consistent, highly available
    
    return 'Raft'  # Default choice for most applications

# Usage examples
web_app_requirements = {
    'byzantine_faults': False,
    'consistency': 'strong',
    'partition_tolerance': 'high',
    'performance': 'high'
}
# Result: Raft

blockchain_requirements = {
    'byzantine_faults': True,
    'consistency': 'strong',
    'partition_tolerance': 'high',
    'performance': 'medium'
}
# Result: PBFT or HotStuff
```

## 🔗 Related Topics

- [[CAP Theorem]] - Fundamental constraints
- [[Distributed Systems Fundamentals]] - Basic concepts
- [[Event-Driven Architecture]] - Async distributed systems
- [[Database Replication]] - Consensus applications

## 📚 Further Reading

- "Designing Data-Intensive Applications" - Martin Kleppmann
- Original Raft paper - Diego Ongaro
- "Paxos Made Simple" - Leslie Lamport
- "Practical Byzantine Fault Tolerance" - Castro & Liskov
- etcd Raft implementation analysis

---

**Key Takeaway**: Consensus algorithms are crucial for building reliable distributed systems. Choose based on your fault tolerance requirements, performance needs, và network conditions. Raft is typically a good default choice for most applications due to its simplicity và strong consistency guarantees.

---

**Next Steps**:
- [ ] Implement basic Raft algorithm
- [ ] Study etcd/Consul implementations
- [ ] Practice consensus algorithm trade-offs
- [ ] Explore blockchain consensus mechanisms