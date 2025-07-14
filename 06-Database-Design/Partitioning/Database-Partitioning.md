# Database Partitioning

## Tổng quan

Database Partitioning là kỹ thuật chia nhỏ database thành các phần nhỏ hơn (partitions) để:
- **Cải thiện Performance**: Queries chỉ scan phần dữ liệu cần thiết
- **Parallel Processing**: Thực hiện operations song song trên các partitions
- **Maintenance**: Backup, indexing, archiving từng partition
- **Scalability**: Lưu trữ partitions trên các servers khác nhau

## Các loại Partitioning

### 1. Horizontal Partitioning (Sharding)

#### Range Partitioning
Phân chia dữ liệu dựa trên range của một column.

```python
class RangePartitioning:
    def __init__(self, column_name, ranges):
        self.column_name = column_name
        self.ranges = ranges  # [(start, end, partition_name)]
        self.partitions = {}
    
    def get_partition(self, value):
        for start, end, partition_name in self.ranges:
            if start <= value <= end:
                return partition_name
        return None
    
    def insert(self, data):
        partition_key = data[self.column_name]
        partition_name = self.get_partition(partition_key)
        
        if partition_name:
            if partition_name not in self.partitions:
                self.partitions[partition_name] = []
            self.partitions[partition_name].append(data)
        else:
            raise ValueError(f"No partition found for value {partition_key}")
    
    def query(self, condition):
        results = []
        
        # Determine which partitions to query
        target_partitions = self.get_target_partitions(condition)
        
        for partition_name in target_partitions:
            if partition_name in self.partitions:
                partition_data = self.partitions[partition_name]
                # Apply condition to partition data
                for row in partition_data:
                    if self.evaluate_condition(row, condition):
                        results.append(row)
        
        return results
    
    def get_target_partitions(self, condition):
        # Analyze condition to determine which partitions to scan
        if self.column_name in condition:
            value = condition[self.column_name]
            partition = self.get_partition(value)
            return [partition] if partition else []
        else:
            # Must scan all partitions
            return [partition_name for _, _, partition_name in self.ranges]

# Example usage
date_partitioning = RangePartitioning('date', [
    ('2023-01-01', '2023-03-31', 'q1_2023'),
    ('2023-04-01', '2023-06-30', 'q2_2023'),
    ('2023-07-01', '2023-09-30', 'q3_2023'),
    ('2023-10-01', '2023-12-31', 'q4_2023')
])
```

#### Hash Partitioning
Phân chia dữ liệu bằng hash function.

```python
import hashlib

class HashPartitioning:
    def __init__(self, column_name, num_partitions):
        self.column_name = column_name
        self.num_partitions = num_partitions
        self.partitions = {i: [] for i in range(num_partitions)}
    
    def get_partition(self, value):
        hash_value = hashlib.md5(str(value).encode()).hexdigest()
        return int(hash_value, 16) % self.num_partitions
    
    def insert(self, data):
        partition_key = data[self.column_name]
        partition_id = self.get_partition(partition_key)
        self.partitions[partition_id].append(data)
    
    def query(self, condition):
        results = []
        
        if self.column_name in condition:
            # Can target specific partition
            value = condition[self.column_name]
            partition_id = self.get_partition(value)
            partition_data = self.partitions[partition_id]
            
            for row in partition_data:
                if self.evaluate_condition(row, condition):
                    results.append(row)
        else:
            # Must scan all partitions
            for partition_id in self.partitions:
                partition_data = self.partitions[partition_id]
                for row in partition_data:
                    if self.evaluate_condition(row, condition):
                        results.append(row)
        
        return results
    
    def evaluate_condition(self, row, condition):
        # Simple condition evaluation
        for key, value in condition.items():
            if row.get(key) != value:
                return False
        return True

# Example usage
user_partitioning = HashPartitioning('user_id', 8)
```

#### List Partitioning
Phân chia dữ liệu dựa trên danh sách giá trị cụ thể.

```python
class ListPartitioning:
    def __init__(self, column_name, partition_lists):
        self.column_name = column_name
        self.partition_lists = partition_lists  # {partition_name: [values]}
        self.partitions = {name: [] for name in partition_lists.keys()}
        
        # Create reverse mapping
        self.value_to_partition = {}
        for partition_name, values in partition_lists.items():
            for value in values:
                self.value_to_partition[value] = partition_name
    
    def get_partition(self, value):
        return self.value_to_partition.get(value)
    
    def insert(self, data):
        partition_key = data[self.column_name]
        partition_name = self.get_partition(partition_key)
        
        if partition_name:
            self.partitions[partition_name].append(data)
        else:
            raise ValueError(f"No partition found for value {partition_key}")
    
    def query(self, condition):
        results = []
        
        if self.column_name in condition:
            value = condition[self.column_name]
            partition_name = self.get_partition(value)
            
            if partition_name:
                partition_data = self.partitions[partition_name]
                for row in partition_data:
                    if self.evaluate_condition(row, condition):
                        results.append(row)
        else:
            # Must scan all partitions
            for partition_name in self.partitions:
                partition_data = self.partitions[partition_name]
                for row in partition_data:
                    if self.evaluate_condition(row, condition):
                        results.append(row)
        
        return results

# Example usage
region_partitioning = ListPartitioning('region', {
    'north_america': ['US', 'CA', 'MX'],
    'europe': ['UK', 'DE', 'FR', 'IT'],
    'asia': ['JP', 'CN', 'KR', 'IN']
})
```

### 2. Vertical Partitioning

Chia table thành các columns khác nhau.

```python
class VerticalPartitioning:
    def __init__(self, table_name, column_groups):
        self.table_name = table_name
        self.column_groups = column_groups  # {partition_name: [columns]}
        self.partitions = {name: {} for name in column_groups.keys()}
        self.primary_key = None
    
    def set_primary_key(self, pk_column):
        self.primary_key = pk_column
    
    def insert(self, data):
        pk_value = data[self.primary_key]
        
        for partition_name, columns in self.column_groups.items():
            partition_data = {}
            partition_data[self.primary_key] = pk_value
            
            for column in columns:
                if column in data:
                    partition_data[column] = data[column]
            
            self.partitions[partition_name][pk_value] = partition_data
    
    def query(self, pk_value, columns=None):
        if columns is None:
            # Return all columns
            result = {self.primary_key: pk_value}
            for partition_name in self.partitions:
                if pk_value in self.partitions[partition_name]:
                    partition_data = self.partitions[partition_name][pk_value]
                    result.update(partition_data)
            return result
        else:
            # Return only requested columns
            result = {self.primary_key: pk_value}
            
            # Determine which partitions to query
            target_partitions = self.get_partitions_for_columns(columns)
            
            for partition_name in target_partitions:
                if pk_value in self.partitions[partition_name]:
                    partition_data = self.partitions[partition_name][pk_value]
                    for column in columns:
                        if column in partition_data:
                            result[column] = partition_data[column]
            
            return result
    
    def get_partitions_for_columns(self, columns):
        partitions = set()
        for column in columns:
            for partition_name, partition_columns in self.column_groups.items():
                if column in partition_columns:
                    partitions.add(partition_name)
        return list(partitions)

# Example usage
user_vertical_partitioning = VerticalPartitioning('users', {
    'basic_info': ['name', 'email', 'created_at'],
    'profile': ['bio', 'avatar', 'preferences'],
    'activity': ['last_login', 'login_count', 'status']
})
user_vertical_partitioning.set_primary_key('user_id')
```

## Database-Specific Implementations

### 1. PostgreSQL Partitioning

```python
import psycopg2

class PostgreSQLPartitioning:
    def __init__(self, connection_string):
        self.connection = psycopg2.connect(connection_string)
        self.cursor = self.connection.cursor()
    
    def create_range_partitioned_table(self, table_name, partition_column, datatype):
        """Create a range partitioned table"""
        create_table_sql = f"""
        CREATE TABLE {table_name} (
            id SERIAL,
            {partition_column} {datatype},
            data JSONB,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        ) PARTITION BY RANGE ({partition_column});
        """
        
        self.cursor.execute(create_table_sql)
        self.connection.commit()
    
    def create_range_partition(self, parent_table, partition_name, start_value, end_value):
        """Create a range partition"""
        partition_sql = f"""
        CREATE TABLE {partition_name} PARTITION OF {parent_table}
        FOR VALUES FROM ('{start_value}') TO ('{end_value}');
        """
        
        self.cursor.execute(partition_sql)
        self.connection.commit()
    
    def create_hash_partitioned_table(self, table_name, partition_column, num_partitions):
        """Create a hash partitioned table"""
        create_table_sql = f"""
        CREATE TABLE {table_name} (
            id SERIAL,
            {partition_column} VARCHAR(255),
            data JSONB,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        ) PARTITION BY HASH ({partition_column});
        """
        
        self.cursor.execute(create_table_sql)
        self.connection.commit()
        
        # Create hash partitions
        for i in range(num_partitions):
            partition_name = f"{table_name}_part_{i}"
            partition_sql = f"""
            CREATE TABLE {partition_name} PARTITION OF {table_name}
            FOR VALUES WITH (MODULUS {num_partitions}, REMAINDER {i});
            """
            self.cursor.execute(partition_sql)
        
        self.connection.commit()
    
    def get_partition_info(self, table_name):
        """Get partition information"""
        query = """
        SELECT 
            schemaname,
            tablename,
            partitiontype,
            partitionkey,
            partitionrangestart,
            partitionrangeend
        FROM pg_partitions 
        WHERE tablename = %s;
        """
        
        self.cursor.execute(query, (table_name,))
        return self.cursor.fetchall()

# Example usage
pg_partitioning = PostgreSQLPartitioning("postgresql://user:password@localhost/dbname")

# Create date-based partitioned table
pg_partitioning.create_range_partitioned_table('orders', 'order_date', 'DATE')
pg_partitioning.create_range_partition('orders', 'orders_2023_q1', '2023-01-01', '2023-04-01')
pg_partitioning.create_range_partition('orders', 'orders_2023_q2', '2023-04-01', '2023-07-01')
```

### 2. MySQL Partitioning

```python
import mysql.connector

class MySQLPartitioning:
    def __init__(self, connection_config):
        self.connection = mysql.connector.connect(**connection_config)
        self.cursor = self.connection.cursor()
    
    def create_range_partitioned_table(self, table_name, partition_column, partitions):
        """Create range partitioned table"""
        
        partition_clauses = []
        for partition_name, max_value in partitions:
            if max_value == 'MAXVALUE':
                partition_clauses.append(f"PARTITION {partition_name} VALUES LESS THAN MAXVALUE")
            else:
                partition_clauses.append(f"PARTITION {partition_name} VALUES LESS THAN ({max_value})")
        
        create_table_sql = f"""
        CREATE TABLE {table_name} (
            id INT AUTO_INCREMENT,
            {partition_column} INT,
            data JSON,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (id, {partition_column})
        )
        PARTITION BY RANGE ({partition_column}) (
            {','.join(partition_clauses)}
        );
        """
        
        self.cursor.execute(create_table_sql)
        self.connection.commit()
    
    def create_hash_partitioned_table(self, table_name, partition_column, num_partitions):
        """Create hash partitioned table"""
        create_table_sql = f"""
        CREATE TABLE {table_name} (
            id INT AUTO_INCREMENT,
            {partition_column} VARCHAR(255),
            data JSON,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (id, {partition_column})
        )
        PARTITION BY HASH ({partition_column})
        PARTITIONS {num_partitions};
        """
        
        self.cursor.execute(create_table_sql)
        self.connection.commit()
    
    def create_list_partitioned_table(self, table_name, partition_column, partitions):
        """Create list partitioned table"""
        partition_clauses = []
        for partition_name, values in partitions:
            value_list = ','.join([f"'{v}'" for v in values])
            partition_clauses.append(f"PARTITION {partition_name} VALUES IN ({value_list})")
        
        create_table_sql = f"""
        CREATE TABLE {table_name} (
            id INT AUTO_INCREMENT,
            {partition_column} VARCHAR(255),
            data JSON,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (id, {partition_column})
        )
        PARTITION BY LIST ({partition_column}) (
            {','.join(partition_clauses)}
        );
        """
        
        self.cursor.execute(create_table_sql)
        self.connection.commit()
    
    def get_partition_info(self, table_name):
        """Get partition information"""
        query = """
        SELECT 
            PARTITION_NAME,
            PARTITION_METHOD,
            PARTITION_EXPRESSION,
            PARTITION_DESCRIPTION,
            TABLE_ROWS,
            DATA_LENGTH
        FROM INFORMATION_SCHEMA.PARTITIONS 
        WHERE TABLE_NAME = %s AND PARTITION_NAME IS NOT NULL;
        """
        
        self.cursor.execute(query, (table_name,))
        return self.cursor.fetchall()

# Example usage
mysql_partitioning = MySQLPartitioning({
    'host': 'localhost',
    'user': 'user',
    'password': 'password',
    'database': 'test_db'
})

# Create year-based partitioned table
mysql_partitioning.create_range_partitioned_table('sales', 'year', [
    ('p2020', 2021),
    ('p2021', 2022),
    ('p2022', 2023),
    ('p2023', 2024),
    ('p_future', 'MAXVALUE')
])
```

## Advanced Partitioning Strategies

### 1. Composite Partitioning

```python
class CompositePartitioning:
    def __init__(self, primary_strategy, secondary_strategy):
        self.primary_strategy = primary_strategy
        self.secondary_strategy = secondary_strategy
        self.partitions = {}
    
    def get_partition(self, data):
        primary_partition = self.primary_strategy.get_partition(data)
        secondary_partition = self.secondary_strategy.get_partition(data)
        
        return f"{primary_partition}_{secondary_partition}"
    
    def insert(self, data):
        partition_key = self.get_partition(data)
        
        if partition_key not in self.partitions:
            self.partitions[partition_key] = []
        
        self.partitions[partition_key].append(data)
    
    def query(self, condition):
        results = []
        
        # Determine target partitions
        target_partitions = self.get_target_partitions(condition)
        
        for partition_key in target_partitions:
            if partition_key in self.partitions:
                partition_data = self.partitions[partition_key]
                for row in partition_data:
                    if self.evaluate_condition(row, condition):
                        results.append(row)
        
        return results
    
    def get_target_partitions(self, condition):
        primary_targets = self.primary_strategy.get_target_partitions(condition)
        secondary_targets = self.secondary_strategy.get_target_partitions(condition)
        
        target_partitions = []
        for primary in primary_targets:
            for secondary in secondary_targets:
                target_partitions.append(f"{primary}_{secondary}")
        
        return target_partitions

# Example: Range-Hash composite partitioning
date_strategy = RangePartitioning('date', [
    ('2023-01-01', '2023-06-30', 'h1_2023'),
    ('2023-07-01', '2023-12-31', 'h2_2023')
])

user_strategy = HashPartitioning('user_id', 4)

composite_partitioning = CompositePartitioning(date_strategy, user_strategy)
```

### 2. Partition Pruning

```python
class PartitionPruning:
    def __init__(self, partitioning_strategy):
        self.partitioning_strategy = partitioning_strategy
        self.statistics = {}
    
    def collect_statistics(self):
        """Collect partition statistics"""
        for partition_name, partition_data in self.partitioning_strategy.partitions.items():
            stats = {
                'row_count': len(partition_data),
                'min_values': {},
                'max_values': {},
                'null_counts': {}
            }
            
            if partition_data:
                # Calculate min/max for each column
                for column in partition_data[0].keys():
                    values = [row[column] for row in partition_data if row[column] is not None]
                    if values:
                        stats['min_values'][column] = min(values)
                        stats['max_values'][column] = max(values)
                    
                    null_count = sum(1 for row in partition_data if row[column] is None)
                    stats['null_counts'][column] = null_count
            
            self.statistics[partition_name] = stats
    
    def prune_partitions(self, condition):
        """Prune partitions based on condition"""
        eligible_partitions = []
        
        for partition_name, stats in self.statistics.items():
            if self.partition_matches_condition(stats, condition):
                eligible_partitions.append(partition_name)
        
        return eligible_partitions
    
    def partition_matches_condition(self, stats, condition):
        """Check if partition can contain matching rows"""
        for column, value in condition.items():
            if column in stats['min_values'] and column in stats['max_values']:
                min_val = stats['min_values'][column]
                max_val = stats['max_values'][column]
                
                # Simple range check
                if isinstance(value, (int, float)):
                    if value < min_val or value > max_val:
                        return False
                elif isinstance(value, str):
                    if value < min_val or value > max_val:
                        return False
        
        return True
```

### 3. Automatic Partition Management

```python
import schedule
import time
from datetime import datetime, timedelta

class AutoPartitionManager:
    def __init__(self, partitioning_strategy):
        self.partitioning_strategy = partitioning_strategy
        self.auto_management_enabled = True
    
    def start_auto_management(self):
        """Start automatic partition management"""
        schedule.every().day.at("02:00").do(self.daily_maintenance)
        schedule.every().week.do(self.weekly_maintenance)
        schedule.every().month.do(self.monthly_maintenance)
        
        while self.auto_management_enabled:
            schedule.run_pending()
            time.sleep(60)  # Check every minute
    
    def daily_maintenance(self):
        """Daily partition maintenance"""
        print("Running daily partition maintenance...")
        
        # Check for partitions that need splitting
        self.check_partition_sizes()
        
        # Update statistics
        self.update_partition_statistics()
    
    def weekly_maintenance(self):
        """Weekly partition maintenance"""
        print("Running weekly partition maintenance...")
        
        # Analyze partition usage
        self.analyze_partition_usage()
        
        # Rebalance if needed
        self.rebalance_partitions()
    
    def monthly_maintenance(self):
        """Monthly partition maintenance"""
        print("Running monthly partition maintenance...")
        
        # Archive old partitions
        self.archive_old_partitions()
        
        # Create new partitions if needed
        self.create_future_partitions()
    
    def check_partition_sizes(self):
        """Check if any partition is too large"""
        for partition_name, partition_data in self.partitioning_strategy.partitions.items():
            if len(partition_data) > 1000000:  # 1M rows threshold
                print(f"Partition {partition_name} is too large: {len(partition_data)} rows")
                # Implement splitting logic
                self.split_partition(partition_name)
    
    def split_partition(self, partition_name):
        """Split a large partition"""
        partition_data = self.partitioning_strategy.partitions[partition_name]
        
        # Split in half
        mid_point = len(partition_data) // 2
        
        # Create new partitions
        new_partition_1 = f"{partition_name}_split_1"
        new_partition_2 = f"{partition_name}_split_2"
        
        self.partitioning_strategy.partitions[new_partition_1] = partition_data[:mid_point]
        self.partitioning_strategy.partitions[new_partition_2] = partition_data[mid_point:]
        
        # Remove original partition
        del self.partitioning_strategy.partitions[partition_name]
        
        print(f"Split partition {partition_name} into {new_partition_1} and {new_partition_2}")
    
    def archive_old_partitions(self):
        """Archive partitions older than retention period"""
        cutoff_date = datetime.now() - timedelta(days=365)  # 1 year retention
        
        for partition_name in list(self.partitioning_strategy.partitions.keys()):
            if self.is_partition_old(partition_name, cutoff_date):
                print(f"Archiving old partition: {partition_name}")
                self.archive_partition(partition_name)
    
    def is_partition_old(self, partition_name, cutoff_date):
        """Check if partition is older than cutoff date"""
        # Implementation depends on partition naming convention
        # Example: partition names like "data_2023_01"
        if "_20" in partition_name:
            try:
                year_month = partition_name.split("_")[-2] + "_" + partition_name.split("_")[-1]
                partition_date = datetime.strptime(year_month, "%Y_%m")
                return partition_date < cutoff_date
            except:
                return False
        return False
    
    def archive_partition(self, partition_name):
        """Archive a partition to cold storage"""
        partition_data = self.partitioning_strategy.partitions[partition_name]
        
        # Write to archive storage (e.g., S3, tape)
        archive_path = f"archive/{partition_name}.json"
        
        with open(archive_path, 'w') as f:
            json.dump(partition_data, f)
        
        # Remove from active partitions
        del self.partitioning_strategy.partitions[partition_name]
        
        print(f"Archived partition {partition_name} to {archive_path}")
```

## Monitoring và Performance

### 1. Partition Monitoring

```python
class PartitionMonitor:
    def __init__(self, partitioning_strategy):
        self.partitioning_strategy = partitioning_strategy
        self.metrics = {
            'partition_sizes': {},
            'query_performance': {},
            'partition_access_patterns': {}
        }
    
    def collect_metrics(self):
        """Collect partition metrics"""
        for partition_name, partition_data in self.partitioning_strategy.partitions.items():
            self.metrics['partition_sizes'][partition_name] = len(partition_data)
        
        # Calculate partition distribution
        total_rows = sum(self.metrics['partition_sizes'].values())
        
        for partition_name, size in self.metrics['partition_sizes'].items():
            distribution = (size / total_rows) * 100 if total_rows > 0 else 0
            print(f"Partition {partition_name}: {size} rows ({distribution:.2f}%)")
    
    def monitor_query_performance(self, query_func, condition):
        """Monitor query performance across partitions"""
        start_time = time.time()
        
        # Execute query
        results = query_func(condition)
        
        end_time = time.time()
        execution_time = end_time - start_time
        
        # Log performance metrics
        self.metrics['query_performance'][str(condition)] = {
            'execution_time': execution_time,
            'result_count': len(results),
            'timestamp': datetime.now().isoformat()
        }
        
        return results
    
    def analyze_partition_skew(self):
        """Analyze partition skew"""
        sizes = list(self.metrics['partition_sizes'].values())
        
        if not sizes:
            return
        
        avg_size = sum(sizes) / len(sizes)
        max_size = max(sizes)
        min_size = min(sizes)
        
        skew_ratio = max_size / avg_size if avg_size > 0 else 0
        
        print(f"Partition skew analysis:")
        print(f"  Average partition size: {avg_size:.0f}")
        print(f"  Max partition size: {max_size}")
        print(f"  Min partition size: {min_size}")
        print(f"  Skew ratio: {skew_ratio:.2f}")
        
        if skew_ratio > 2.0:
            print("  WARNING: High partition skew detected!")
    
    def generate_report(self):
        """Generate partition report"""
        report = {
            'timestamp': datetime.now().isoformat(),
            'total_partitions': len(self.partitioning_strategy.partitions),
            'total_rows': sum(self.metrics['partition_sizes'].values()),
            'partition_details': []
        }
        
        for partition_name, size in self.metrics['partition_sizes'].items():
            report['partition_details'].append({
                'name': partition_name,
                'size': size,
                'percentage': (size / report['total_rows']) * 100 if report['total_rows'] > 0 else 0
            })
        
        return report
```

### 2. Performance Optimization

```python
class PartitionOptimizer:
    def __init__(self, partitioning_strategy):
        self.partitioning_strategy = partitioning_strategy
        self.query_log = []
    
    def log_query(self, condition, execution_time, partitions_scanned):
        """Log query for analysis"""
        self.query_log.append({
            'condition': condition,
            'execution_time': execution_time,
            'partitions_scanned': partitions_scanned,
            'timestamp': datetime.now()
        })
    
    def analyze_query_patterns(self):
        """Analyze query patterns to optimize partitioning"""
        if not self.query_log:
            return
        
        # Find most frequently queried columns
        column_usage = {}
        for query in self.query_log:
            for column in query['condition'].keys():
                column_usage[column] = column_usage.get(column, 0) + 1
        
        print("Column usage frequency:")
        for column, count in sorted(column_usage.items(), key=lambda x: x[1], reverse=True):
            print(f"  {column}: {count} queries")
        
        # Identify slow queries
        slow_queries = [q for q in self.query_log if q['execution_time'] > 1.0]
        
        if slow_queries:
            print(f"\nSlow queries (>1s): {len(slow_queries)}")
            for query in slow_queries[:5]:  # Show top 5
                print(f"  {query['condition']}: {query['execution_time']:.2f}s")
    
    def recommend_optimizations(self):
        """Recommend partition optimizations"""
        recommendations = []
        
        # Analyze partition sizes
        sizes = [len(data) for data in self.partitioning_strategy.partitions.values()]
        
        if sizes:
            avg_size = sum(sizes) / len(sizes)
            large_partitions = [name for name, data in self.partitioning_strategy.partitions.items() 
                              if len(data) > avg_size * 2]
            
            if large_partitions:
                recommendations.append({
                    'type': 'partition_splitting',
                    'description': f'Consider splitting large partitions: {large_partitions}'
                })
        
        # Analyze query patterns
        if self.query_log:
            full_scan_queries = [q for q in self.query_log if q['partitions_scanned'] == len(self.partitioning_strategy.partitions)]
            
            if len(full_scan_queries) > len(self.query_log) * 0.5:
                recommendations.append({
                    'type': 'partitioning_strategy',
                    'description': 'Many queries require full partition scans. Consider different partitioning key.'
                })
        
        return recommendations
```

Database Partitioning là kỹ thuật quan trọng để cải thiện performance và scalability. Việc chọn strategy phù hợp phụ thuộc vào data access patterns và business requirements. 