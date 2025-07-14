# Database Replication

## Tổng quan

Database Replication là quá trình sao chép dữ liệu từ một database (master) sang các database khác (replica/slave) để đảm bảo:
- **High Availability**: Hệ thống tiếp tục hoạt động khi một node bị lỗi
- **Load Distribution**: Phân tán read operations
- **Disaster Recovery**: Backup dữ liệu ở nhiều location
- **Geographic Distribution**: Giảm latency cho users ở xa

## Các loại Replication

### 1. Master-Slave Replication

#### Kiến trúc
```
Master (Write) → Slave 1 (Read)
              → Slave 2 (Read)
              → Slave 3 (Read)
```

#### Implementation (MySQL)
```python
import mysql.connector
from mysql.connector import Error

class MasterSlaveReplication:
    def __init__(self, master_config, slave_configs):
        self.master_config = master_config
        self.slave_configs = slave_configs
        self.master_connection = None
        self.slave_connections = []
        self.current_slave_index = 0
    
    def connect_to_master(self):
        try:
            self.master_connection = mysql.connector.connect(**self.master_config)
            return True
        except Error as e:
            print(f"Error connecting to master: {e}")
            return False
    
    def connect_to_slaves(self):
        for slave_config in self.slave_configs:
            try:
                connection = mysql.connector.connect(**slave_config)
                self.slave_connections.append(connection)
            except Error as e:
                print(f"Error connecting to slave: {e}")
    
    def write_operation(self, query, params=None):
        """All write operations go to master"""
        if not self.master_connection:
            self.connect_to_master()
        
        cursor = self.master_connection.cursor()
        try:
            cursor.execute(query, params)
            self.master_connection.commit()
            return cursor.lastrowid
        except Error as e:
            self.master_connection.rollback()
            raise e
        finally:
            cursor.close()
    
    def read_operation(self, query, params=None):
        """Read operations can go to any slave"""
        slave_conn = self.get_healthy_slave()
        if not slave_conn:
            # Fallback to master
            slave_conn = self.master_connection
        
        cursor = slave_conn.cursor()
        try:
            cursor.execute(query, params)
            return cursor.fetchall()
        except Error as e:
            # Try another slave or fallback to master
            return self.read_operation_fallback(query, params)
        finally:
            cursor.close()
    
    def get_healthy_slave(self):
        """Round-robin slave selection with health check"""
        for i in range(len(self.slave_connections)):
            slave_index = (self.current_slave_index + i) % len(self.slave_connections)
            slave_conn = self.slave_connections[slave_index]
            
            if self.is_slave_healthy(slave_conn):
                self.current_slave_index = (slave_index + 1) % len(self.slave_connections)
                return slave_conn
        
        return None
    
    def is_slave_healthy(self, connection):
        try:
            cursor = connection.cursor()
            cursor.execute("SELECT 1")
            cursor.fetchone()
            return True
        except:
            return False
    
    def check_replication_lag(self, slave_connection):
        """Check how far behind the slave is"""
        cursor = slave_connection.cursor()
        cursor.execute("SHOW SLAVE STATUS")
        status = cursor.fetchone()
        
        if status:
            # Seconds_Behind_Master column
            return status[32]  # May vary based on MySQL version
        return None
```

#### MySQL Master Configuration
```sql
-- /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-format = ROW
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1

-- Create replication user
CREATE USER 'replication'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
FLUSH PRIVILEGES;

-- Get master status
SHOW MASTER STATUS;
```

#### MySQL Slave Configuration
```sql
-- /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
server-id = 2
relay-log = mysql-relay-bin
log-slave-updates = 1
read-only = 1

-- Configure slave
CHANGE MASTER TO
  MASTER_HOST='master_host',
  MASTER_USER='replication',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;

START SLAVE;
SHOW SLAVE STATUS\G;
```

### 2. Master-Master Replication

#### Kiến trúc
```
Master A ←→ Master B
   ↓           ↓
Slave A1    Slave B1
Slave A2    Slave B2
```

#### Implementation
```python
class MasterMasterReplication:
    def __init__(self, master_configs):
        self.masters = master_configs
        self.connections = {}
        self.current_master = 0
    
    def connect_to_masters(self):
        for i, config in enumerate(self.masters):
            try:
                connection = mysql.connector.connect(**config)
                self.connections[i] = connection
            except Error as e:
                print(f"Error connecting to master {i}: {e}")
    
    def write_operation(self, query, params=None):
        """Try primary master first, fallback to secondary"""
        master_index = self.current_master
        
        try:
            return self.execute_on_master(master_index, query, params)
        except Exception as e:
            # Failover to other master
            other_master = (master_index + 1) % len(self.masters)
            try:
                result = self.execute_on_master(other_master, query, params)
                self.current_master = other_master  # Switch primary
                return result
            except Exception as e2:
                raise Exception(f"Both masters failed: {e}, {e2}")
    
    def execute_on_master(self, master_index, query, params):
        connection = self.connections[master_index]
        cursor = connection.cursor()
        try:
            cursor.execute(query, params)
            connection.commit()
            return cursor.lastrowid
        except Error as e:
            connection.rollback()
            raise e
        finally:
            cursor.close()
    
    def handle_conflict_resolution(self):
        """Resolve conflicts when both masters are updated"""
        # Implement conflict resolution strategy
        # Options: timestamp-based, application-specific logic
        pass
```

### 3. Multi-Master Replication

#### Galera Cluster Example
```python
class GaleraCluster:
    def __init__(self, cluster_nodes):
        self.cluster_nodes = cluster_nodes
        self.connections = {}
        self.current_node = 0
    
    def connect_to_cluster(self):
        for i, node_config in enumerate(self.cluster_nodes):
            try:
                connection = mysql.connector.connect(**node_config)
                self.connections[i] = connection
            except Error as e:
                print(f"Error connecting to node {i}: {e}")
    
    def write_operation(self, query, params=None):
        """Write to any available node"""
        available_nodes = self.get_available_nodes()
        
        if not available_nodes:
            raise Exception("No available nodes in cluster")
        
        node_index = available_nodes[self.current_node % len(available_nodes)]
        
        try:
            return self.execute_on_node(node_index, query, params)
        except Exception as e:
            # Try other nodes
            for node in available_nodes:
                if node != node_index:
                    try:
                        return self.execute_on_node(node, query, params)
                    except:
                        continue
            raise e
    
    def get_available_nodes(self):
        available = []
        for i, connection in self.connections.items():
            if self.is_node_healthy(connection):
                available.append(i)
        return available
    
    def is_node_healthy(self, connection):
        try:
            cursor = connection.cursor()
            cursor.execute("SHOW STATUS LIKE 'wsrep_cluster_status'")
            status = cursor.fetchone()
            return status[1] == 'Primary'
        except:
            return False
```

## Replication Strategies

### 1. Synchronous Replication

#### Characteristics
- **Consistency**: Strong consistency
- **Performance**: Higher latency
- **Reliability**: High durability

#### Implementation
```python
class SynchronousReplication:
    def __init__(self, master, slaves):
        self.master = master
        self.slaves = slaves
        self.min_slaves_for_commit = len(slaves) // 2 + 1
    
    def write_with_sync_replication(self, query, params=None):
        # Phase 1: Prepare
        transaction_id = self.generate_transaction_id()
        
        # Send to all slaves
        prepared_slaves = []
        for slave in self.slaves:
            if self.prepare_on_slave(slave, transaction_id, query, params):
                prepared_slaves.append(slave)
        
        # Check if we have enough slaves
        if len(prepared_slaves) < self.min_slaves_for_commit:
            self.abort_transaction(transaction_id, prepared_slaves)
            raise Exception("Not enough slaves for commit")
        
        # Phase 2: Commit
        try:
            # Commit on master
            self.commit_on_master(query, params)
            
            # Commit on slaves
            for slave in prepared_slaves:
                self.commit_on_slave(slave, transaction_id)
            
            return transaction_id
        except Exception as e:
            self.abort_transaction(transaction_id, prepared_slaves)
            raise e
    
    def prepare_on_slave(self, slave, transaction_id, query, params):
        try:
            cursor = slave.cursor()
            cursor.execute("START TRANSACTION")
            cursor.execute(query, params)
            # Don't commit yet
            return True
        except:
            return False
    
    def commit_on_slave(self, slave, transaction_id):
        cursor = slave.cursor()
        cursor.execute("COMMIT")
        cursor.close()
    
    def abort_transaction(self, transaction_id, slaves):
        for slave in slaves:
            cursor = slave.cursor()
            cursor.execute("ROLLBACK")
            cursor.close()
```

### 2. Asynchronous Replication

#### Implementation
```python
import asyncio
import queue
import threading

class AsynchronousReplication:
    def __init__(self, master, slaves):
        self.master = master
        self.slaves = slaves
        self.replication_queue = queue.Queue()
        self.replication_thread = threading.Thread(target=self.replication_worker)
        self.replication_thread.daemon = True
        self.replication_thread.start()
    
    def write_with_async_replication(self, query, params=None):
        # Write to master immediately
        cursor = self.master.cursor()
        try:
            cursor.execute(query, params)
            self.master.commit()
            
            # Queue for replication
            self.replication_queue.put({
                'query': query,
                'params': params,
                'timestamp': time.time()
            })
            
            return cursor.lastrowid
        except Exception as e:
            self.master.rollback()
            raise e
        finally:
            cursor.close()
    
    def replication_worker(self):
        while True:
            try:
                operation = self.replication_queue.get(timeout=1)
                self.replicate_to_slaves(operation)
                self.replication_queue.task_done()
            except queue.Empty:
                continue
            except Exception as e:
                print(f"Replication error: {e}")
    
    def replicate_to_slaves(self, operation):
        threads = []
        for slave in self.slaves:
            thread = threading.Thread(
                target=self.replicate_to_slave,
                args=(slave, operation)
            )
            threads.append(thread)
            thread.start()
        
        # Wait for all slaves
        for thread in threads:
            thread.join()
    
    def replicate_to_slave(self, slave, operation):
        try:
            cursor = slave.cursor()
            cursor.execute(operation['query'], operation['params'])
            slave.commit()
        except Exception as e:
            print(f"Failed to replicate to slave: {e}")
            # Add to retry queue
            self.handle_replication_failure(slave, operation)
```

### 3. Semi-Synchronous Replication

#### Implementation
```python
class SemiSynchronousReplication:
    def __init__(self, master, slaves, min_slaves_ack=1):
        self.master = master
        self.slaves = slaves
        self.min_slaves_ack = min_slaves_ack
        self.timeout = 10  # seconds
    
    def write_with_semi_sync_replication(self, query, params=None):
        # Write to master
        cursor = self.master.cursor()
        try:
            cursor.execute(query, params)
            master_result = cursor.lastrowid
            
            # Wait for acknowledgment from at least min_slaves_ack slaves
            ack_count = 0
            start_time = time.time()
            
            for slave in self.slaves:
                if time.time() - start_time > self.timeout:
                    break
                
                if self.send_to_slave_and_wait_ack(slave, query, params):
                    ack_count += 1
                    
                    if ack_count >= self.min_slaves_ack:
                        break
            
            if ack_count >= self.min_slaves_ack:
                self.master.commit()
                return master_result
            else:
                self.master.rollback()
                raise Exception("Not enough slave acknowledgments")
                
        except Exception as e:
            self.master.rollback()
            raise e
        finally:
            cursor.close()
    
    def send_to_slave_and_wait_ack(self, slave, query, params):
        try:
            cursor = slave.cursor()
            cursor.execute(query, params)
            slave.commit()
            return True
        except:
            return False
```

## Monitoring và Maintenance

### 1. Replication Monitoring
```python
class ReplicationMonitor:
    def __init__(self, master, slaves):
        self.master = master
        self.slaves = slaves
        self.metrics = {
            'replication_lag': {},
            'slave_status': {},
            'error_count': 0
        }
    
    def monitor_replication_lag(self):
        for i, slave in enumerate(self.slaves):
            try:
                lag = self.get_replication_lag(slave)
                self.metrics['replication_lag'][i] = lag
                
                if lag > 60:  # Alert if lag > 60 seconds
                    self.send_alert(f"High replication lag on slave {i}: {lag}s")
                    
            except Exception as e:
                self.metrics['error_count'] += 1
                self.send_alert(f"Error monitoring slave {i}: {e}")
    
    def get_replication_lag(self, slave):
        cursor = slave.cursor()
        cursor.execute("SHOW SLAVE STATUS")
        status = cursor.fetchone()
        
        if status:
            return status[32]  # Seconds_Behind_Master
        return None
    
    def monitor_slave_status(self):
        for i, slave in enumerate(self.slaves):
            try:
                status = self.get_slave_status(slave)
                self.metrics['slave_status'][i] = status
                
                if not status['io_running'] or not status['sql_running']:
                    self.send_alert(f"Slave {i} replication stopped")
                    
            except Exception as e:
                self.send_alert(f"Error checking slave {i} status: {e}")
    
    def get_slave_status(self, slave):
        cursor = slave.cursor()
        cursor.execute("SHOW SLAVE STATUS")
        result = cursor.fetchone()
        
        if result:
            return {
                'io_running': result[10] == 'Yes',
                'sql_running': result[11] == 'Yes',
                'last_error': result[19]
            }
        return None
    
    def send_alert(self, message):
        # Send to monitoring system
        print(f"ALERT: {message}")
```

### 2. Failover Management
```python
class FailoverManager:
    def __init__(self, master, slaves):
        self.master = master
        self.slaves = slaves
        self.current_master = master
        self.failover_in_progress = False
    
    def check_master_health(self):
        try:
            cursor = self.master.cursor()
            cursor.execute("SELECT 1")
            cursor.fetchone()
            return True
        except:
            return False
    
    def automatic_failover(self):
        if self.failover_in_progress:
            return
        
        self.failover_in_progress = True
        
        try:
            # Find best slave to promote
            best_slave = self.find_best_slave_for_promotion()
            
            if best_slave:
                self.promote_slave_to_master(best_slave)
                self.update_remaining_slaves(best_slave)
                self.current_master = best_slave
                
            self.failover_in_progress = False
            
        except Exception as e:
            self.failover_in_progress = False
            raise e
    
    def find_best_slave_for_promotion(self):
        best_slave = None
        min_lag = float('inf')
        
        for slave in self.slaves:
            if self.is_slave_healthy(slave):
                lag = self.get_replication_lag(slave)
                if lag < min_lag:
                    min_lag = lag
                    best_slave = slave
        
        return best_slave
    
    def promote_slave_to_master(self, slave):
        cursor = slave.cursor()
        
        # Stop slave processes
        cursor.execute("STOP SLAVE")
        
        # Reset slave configuration
        cursor.execute("RESET SLAVE ALL")
        
        # Enable writes
        cursor.execute("SET GLOBAL read_only = 0")
        
        # Enable binary logging for new master
        cursor.execute("SET GLOBAL log_bin = 1")
        
        cursor.close()
    
    def update_remaining_slaves(self, new_master):
        for slave in self.slaves:
            if slave != new_master:
                try:
                    self.reconfigure_slave(slave, new_master)
                except Exception as e:
                    print(f"Failed to reconfigure slave: {e}")
    
    def reconfigure_slave(self, slave, new_master):
        cursor = slave.cursor()
        
        # Stop current replication
        cursor.execute("STOP SLAVE")
        
        # Change master
        cursor.execute(f"""
            CHANGE MASTER TO
            MASTER_HOST='{new_master.host}',
            MASTER_USER='replication',
            MASTER_PASSWORD='password',
            MASTER_AUTO_POSITION=1
        """)
        
        # Start replication
        cursor.execute("START SLAVE")
        
        cursor.close()
```

## Best Practices

### 1. Configuration
```python
class ReplicationBestPractices:
    
    @staticmethod
    def recommended_mysql_settings():
        return {
            'master_settings': {
                'sync_binlog': 1,
                'innodb_flush_log_at_trx_commit': 1,
                'binlog_format': 'ROW',
                'expire_logs_days': 7,
                'max_binlog_size': '100M'
            },
            'slave_settings': {
                'read_only': 1,
                'relay_log_recovery': 1,
                'slave_net_timeout': 60,
                'slave_skip_errors': '1062,1053'
            }
        }
    
    @staticmethod
    def recommended_monitoring_queries():
        return [
            "SHOW SLAVE STATUS",
            "SHOW MASTER STATUS",
            "SHOW PROCESSLIST",
            "SELECT * FROM information_schema.REPLICA_HOST_STATUS"
        ]
```

### 2. Error Handling
```python
class ReplicationErrorHandler:
    def __init__(self):
        self.error_patterns = {
            1062: "Duplicate entry",
            1032: "Can't find record",
            1205: "Lock wait timeout"
        }
    
    def handle_replication_error(self, error_code, error_message):
        if error_code in self.error_patterns:
            return self.handle_known_error(error_code, error_message)
        else:
            return self.handle_unknown_error(error_code, error_message)
    
    def handle_known_error(self, error_code, error_message):
        if error_code == 1062:  # Duplicate entry
            return "SET GLOBAL sql_slave_skip_counter = 1; START SLAVE;"
        elif error_code == 1032:  # Can't find record
            return "SET GLOBAL sql_slave_skip_counter = 1; START SLAVE;"
        elif error_code == 1205:  # Lock wait timeout
            return "START SLAVE;"
    
    def handle_unknown_error(self, error_code, error_message):
        return f"Manual intervention required for error {error_code}: {error_message}"
```

Database Replication là foundation quan trọng cho high availability và scalability. Việc chọn strategy phù hợp phụ thuộc vào requirements cụ thể về consistency, performance, và complexity. 