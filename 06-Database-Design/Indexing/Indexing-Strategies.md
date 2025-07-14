# Indexing Strategies

## Tổng quan

Database Indexing là kỹ thuật tạo ra các cấu trúc dữ liệu phụ trợ để tăng tốc độ truy vấn dữ liệu:
- **Tăng tốc độ SELECT**: Giảm thời gian tìm kiếm từ O(n) xuống O(log n)
- **Tối ưu hóa JOIN**: Cải thiện performance của join operations
- **Hỗ trợ sorting**: Tăng tốc ORDER BY queries
- **Unique constraints**: Đảm bảo tính duy nhất của dữ liệu

## Các loại Index

### 1. B-Tree Index

B-Tree là cấu trúc index phổ biến nhất, phù hợp cho range queries.

```python
class BTreeNode:
    def __init__(self, is_leaf=False):
        self.keys = []
        self.values = []  # For leaf nodes
        self.children = []  # For internal nodes
        self.is_leaf = is_leaf
        self.next_leaf = None  # For leaf nodes linking

class BTreeIndex:
    def __init__(self, order=3):
        self.root = BTreeNode(is_leaf=True)
        self.order = order  # Maximum number of children
        self.min_keys = (order - 1) // 2
        self.max_keys = order - 1
    
    def search(self, key):
        """Search for a key in the B-tree"""
        return self._search_recursive(self.root, key)
    
    def _search_recursive(self, node, key):
        """Recursive search implementation"""
        i = 0
        while i < len(node.keys) and key > node.keys[i]:
            i += 1
        
        if i < len(node.keys) and key == node.keys[i]:
            if node.is_leaf:
                return node.values[i]
            else:
                return self._search_recursive(node.children[i + 1], key)
        
        if node.is_leaf:
            return None
        else:
            return self._search_recursive(node.children[i], key)
    
    def insert(self, key, value):
        """Insert a key-value pair"""
        root = self.root
        
        # If root is full, split it
        if len(root.keys) == self.max_keys:
            new_root = BTreeNode()
            new_root.children.append(root)
            self._split_child(new_root, 0)
            self.root = new_root
        
        self._insert_non_full(self.root, key, value)
    
    def _insert_non_full(self, node, key, value):
        """Insert into a non-full node"""
        i = len(node.keys) - 1
        
        if node.is_leaf:
            # Insert into leaf node
            node.keys.append(None)
            node.values.append(None)
            
            while i >= 0 and key < node.keys[i]:
                node.keys[i + 1] = node.keys[i]
                node.values[i + 1] = node.values[i]
                i -= 1
            
            node.keys[i + 1] = key
            node.values[i + 1] = value
        else:
            # Find child to insert into
            while i >= 0 and key < node.keys[i]:
                i -= 1
            i += 1
            
            # Split child if full
            if len(node.children[i].keys) == self.max_keys:
                self._split_child(node, i)
                if key > node.keys[i]:
                    i += 1
            
            self._insert_non_full(node.children[i], key, value)
    
    def _split_child(self, parent, index):
        """Split a full child node"""
        full_child = parent.children[index]
        new_child = BTreeNode(is_leaf=full_child.is_leaf)
        
        mid = self.max_keys // 2
        
        # Move keys to new child
        new_child.keys = full_child.keys[mid + 1:]
        full_child.keys = full_child.keys[:mid]
        
        if full_child.is_leaf:
            new_child.values = full_child.values[mid + 1:]
            full_child.values = full_child.values[:mid]
            
            # Link leaf nodes
            new_child.next_leaf = full_child.next_leaf
            full_child.next_leaf = new_child
        else:
            new_child.children = full_child.children[mid + 1:]
            full_child.children = full_child.children[:mid + 1]
        
        # Promote middle key to parent
        parent.keys.insert(index, full_child.keys[mid])
        parent.children.insert(index + 1, new_child)
        
        if full_child.is_leaf:
            # Keep middle key in leaf for range queries
            full_child.keys.append(full_child.keys[mid])
            full_child.values.append(full_child.values[mid])
    
    def range_query(self, start_key, end_key):
        """Perform range query"""
        results = []
        
        # Find starting leaf node
        leaf_node = self._find_leaf(start_key)
        
        # Traverse leaf nodes
        while leaf_node:
            for i, key in enumerate(leaf_node.keys):
                if key >= start_key and key <= end_key:
                    results.append((key, leaf_node.values[i]))
                elif key > end_key:
                    return results
            
            leaf_node = leaf_node.next_leaf
        
        return results
    
    def _find_leaf(self, key):
        """Find leaf node that could contain the key"""
        node = self.root
        
        while not node.is_leaf:
            i = 0
            while i < len(node.keys) and key > node.keys[i]:
                i += 1
            node = node.children[i]
        
        return node

# Usage
btree_index = BTreeIndex(order=5)

# Insert data
btree_index.insert(10, "Record 10")
btree_index.insert(20, "Record 20")
btree_index.insert(5, "Record 5")
btree_index.insert(15, "Record 15")

# Search
result = btree_index.search(15)
print(f"Found: {result}")

# Range query
range_results = btree_index.range_query(10, 20)
print(f"Range query results: {range_results}")
```

### 2. Hash Index

Hash Index sử dụng hash function để tìm kiếm nhanh với O(1) average time.

```python
import hashlib

class HashIndex:
    def __init__(self, initial_size=1000):
        self.size = initial_size
        self.buckets = [[] for _ in range(self.size)]
        self.count = 0
        self.load_factor_threshold = 0.75
    
    def _hash(self, key):
        """Hash function"""
        hash_value = hashlib.md5(str(key).encode()).hexdigest()
        return int(hash_value, 16) % self.size
    
    def insert(self, key, value):
        """Insert key-value pair"""
        # Check load factor
        if self.count >= self.size * self.load_factor_threshold:
            self._resize()
        
        bucket_index = self._hash(key)
        bucket = self.buckets[bucket_index]
        
        # Check if key already exists
        for i, (k, v) in enumerate(bucket):
            if k == key:
                bucket[i] = (key, value)  # Update existing
                return
        
        # Insert new key-value pair
        bucket.append((key, value))
        self.count += 1
    
    def search(self, key):
        """Search for key"""
        bucket_index = self._hash(key)
        bucket = self.buckets[bucket_index]
        
        for k, v in bucket:
            if k == key:
                return v
        
        return None
    
    def delete(self, key):
        """Delete key"""
        bucket_index = self._hash(key)
        bucket = self.buckets[bucket_index]
        
        for i, (k, v) in enumerate(bucket):
            if k == key:
                del bucket[i]
                self.count -= 1
                return True
        
        return False
    
    def _resize(self):
        """Resize hash table when load factor is too high"""
        old_buckets = self.buckets
        self.size *= 2
        self.buckets = [[] for _ in range(self.size)]
        old_count = self.count
        self.count = 0
        
        # Rehash all elements
        for bucket in old_buckets:
            for key, value in bucket:
                self.insert(key, value)
        
        print(f"Hash table resized from {self.size // 2} to {self.size}")
        print(f"Rehashed {old_count} elements")
    
    def get_stats(self):
        """Get hash table statistics"""
        non_empty_buckets = sum(1 for bucket in self.buckets if bucket)
        max_bucket_size = max(len(bucket) for bucket in self.buckets)
        avg_bucket_size = self.count / non_empty_buckets if non_empty_buckets else 0
        
        return {
            'size': self.size,
            'count': self.count,
            'load_factor': self.count / self.size,
            'non_empty_buckets': non_empty_buckets,
            'max_bucket_size': max_bucket_size,
            'avg_bucket_size': avg_bucket_size
        }

# Usage
hash_index = HashIndex()

# Insert data
hash_index.insert("user_123", {"name": "John", "age": 30})
hash_index.insert("user_456", {"name": "Jane", "age": 25})
hash_index.insert("user_789", {"name": "Bob", "age": 35})

# Search
user = hash_index.search("user_456")
print(f"Found user: {user}")

# Statistics
stats = hash_index.get_stats()
print(f"Hash index stats: {stats}")
```

### 3. Bitmap Index

Bitmap Index hiệu quả cho columns có ít giá trị distinct (low cardinality).

```python
class BitmapIndex:
    def __init__(self, column_name):
        self.column_name = column_name
        self.bitmaps = {}  # {value: bitarray}
        self.row_count = 0
    
    def add_value(self, value, row_id):
        """Add value for a specific row"""
        # Ensure we have enough bits
        while row_id >= self.row_count:
            self._extend_bitmaps()
        
        # Set bit for this value
        if value not in self.bitmaps:
            self.bitmaps[value] = [False] * self.row_count
        
        self.bitmaps[value][row_id] = True
        
        # Unset bit for other values (assuming single value per row)
        for other_value, bitmap in self.bitmaps.items():
            if other_value != value and len(bitmap) > row_id:
                bitmap[row_id] = False
    
    def _extend_bitmaps(self):
        """Extend all bitmaps with new row"""
        for bitmap in self.bitmaps.values():
            bitmap.append(False)
        self.row_count += 1
    
    def query_equals(self, value):
        """Query rows where column equals value"""
        if value not in self.bitmaps:
            return []
        
        return [i for i, bit in enumerate(self.bitmaps[value]) if bit]
    
    def query_in(self, values):
        """Query rows where column in values"""
        result_bitmap = [False] * self.row_count
        
        for value in values:
            if value in self.bitmaps:
                for i, bit in enumerate(self.bitmaps[value]):
                    result_bitmap[i] = result_bitmap[i] or bit
        
        return [i for i, bit in enumerate(result_bitmap) if bit]
    
    def query_not_equals(self, value):
        """Query rows where column not equals value"""
        if value not in self.bitmaps:
            return list(range(self.row_count))
        
        return [i for i, bit in enumerate(self.bitmaps[value]) if not bit]
    
    def and_operation(self, other_index):
        """AND operation with another bitmap index"""
        result = {}
        
        for value1 in self.bitmaps:
            for value2 in other_index.bitmaps:
                result_key = f"{value1}_{value2}"
                result[result_key] = []
                
                for i in range(min(len(self.bitmaps[value1]), len(other_index.bitmaps[value2]))):
                    result[result_key].append(
                        self.bitmaps[value1][i] and other_index.bitmaps[value2][i]
                    )
        
        return result
    
    def or_operation(self, other_index):
        """OR operation with another bitmap index"""
        result = {}
        
        for value1 in self.bitmaps:
            for value2 in other_index.bitmaps:
                result_key = f"{value1}_{value2}"
                result[result_key] = []
                
                for i in range(min(len(self.bitmaps[value1]), len(other_index.bitmaps[value2]))):
                    result[result_key].append(
                        self.bitmaps[value1][i] or other_index.bitmaps[value2][i]
                    )
        
        return result
    
    def get_cardinality(self):
        """Get number of distinct values"""
        return len(self.bitmaps)
    
    def get_stats(self):
        """Get bitmap index statistics"""
        total_bits = sum(len(bitmap) for bitmap in self.bitmaps.values())
        set_bits = sum(sum(bitmap) for bitmap in self.bitmaps.values())
        
        return {
            'distinct_values': len(self.bitmaps),
            'total_rows': self.row_count,
            'total_bits': total_bits,
            'set_bits': set_bits,
            'compression_ratio': (total_bits - set_bits) / total_bits if total_bits > 0 else 0
        }

# Usage
# Create bitmap index for gender column
gender_index = BitmapIndex('gender')

# Add data
gender_index.add_value('Male', 0)
gender_index.add_value('Female', 1)
gender_index.add_value('Male', 2)
gender_index.add_value('Female', 3)
gender_index.add_value('Male', 4)

# Query
male_rows = gender_index.query_equals('Male')
print(f"Male rows: {male_rows}")

female_rows = gender_index.query_equals('Female')
print(f"Female rows: {female_rows}")

# Statistics
stats = gender_index.get_stats()
print(f"Bitmap index stats: {stats}")
```

### 4. Inverted Index

Inverted Index thường dùng cho full-text search.

```python
import re
from collections import defaultdict

class InvertedIndex:
    def __init__(self):
        self.index = defaultdict(set)  # {term: set(doc_ids)}
        self.documents = {}  # {doc_id: document_text}
        self.term_frequencies = defaultdict(lambda: defaultdict(int))  # {doc_id: {term: frequency}}
    
    def add_document(self, doc_id, text):
        """Add document to index"""
        self.documents[doc_id] = text
        terms = self._tokenize(text)
        
        # Build inverted index
        for term in terms:
            self.index[term].add(doc_id)
            self.term_frequencies[doc_id][term] += 1
    
    def _tokenize(self, text):
        """Tokenize text into terms"""
        # Convert to lowercase and split by non-alphanumeric characters
        terms = re.findall(r'\b\w+\b', text.lower())
        return terms
    
    def search(self, query):
        """Search for documents containing query terms"""
        query_terms = self._tokenize(query)
        
        if not query_terms:
            return []
        
        # Get documents containing all query terms (AND operation)
        result_docs = set(self.index[query_terms[0]])
        
        for term in query_terms[1:]:
            result_docs = result_docs.intersection(self.index[term])
        
        return list(result_docs)
    
    def search_or(self, query):
        """Search for documents containing any query terms (OR operation)"""
        query_terms = self._tokenize(query)
        result_docs = set()
        
        for term in query_terms:
            result_docs = result_docs.union(self.index[term])
        
        return list(result_docs)
    
    def search_phrase(self, phrase):
        """Search for exact phrase"""
        phrase_terms = self._tokenize(phrase)
        
        if not phrase_terms:
            return []
        
        # Get documents containing all terms
        candidate_docs = self.search(' '.join(phrase_terms))
        
        # Check for exact phrase match
        matching_docs = []
        for doc_id in candidate_docs:
            if self._contains_phrase(doc_id, phrase_terms):
                matching_docs.append(doc_id)
        
        return matching_docs
    
    def _contains_phrase(self, doc_id, phrase_terms):
        """Check if document contains exact phrase"""
        text = self.documents[doc_id].lower()
        phrase = ' '.join(phrase_terms)
        return phrase in text
    
    def get_term_frequency(self, doc_id, term):
        """Get frequency of term in document"""
        return self.term_frequencies[doc_id][term]
    
    def get_document_frequency(self, term):
        """Get number of documents containing term"""
        return len(self.index[term])
    
    def calculate_tf_idf(self, doc_id, term):
        """Calculate TF-IDF score"""
        tf = self.get_term_frequency(doc_id, term)
        df = self.get_document_frequency(term)
        
        if tf == 0 or df == 0:
            return 0
        
        import math
        
        # TF-IDF = TF * log(N/DF)
        tf_score = 1 + math.log(tf) if tf > 0 else 0
        idf_score = math.log(len(self.documents) / df)
        
        return tf_score * idf_score
    
    def search_ranked(self, query, limit=10):
        """Search and return ranked results"""
        query_terms = self._tokenize(query)
        candidate_docs = self.search_or(query)
        
        # Calculate scores
        doc_scores = []
        for doc_id in candidate_docs:
            score = 0
            for term in query_terms:
                score += self.calculate_tf_idf(doc_id, term)
            
            doc_scores.append((doc_id, score))
        
        # Sort by score
        doc_scores.sort(key=lambda x: x[1], reverse=True)
        
        return doc_scores[:limit]
    
    def get_stats(self):
        """Get index statistics"""
        return {
            'total_documents': len(self.documents),
            'total_terms': len(self.index),
            'avg_terms_per_doc': sum(len(terms) for terms in self.term_frequencies.values()) / len(self.documents) if self.documents else 0,
            'most_common_terms': sorted(self.index.items(), key=lambda x: len(x[1]), reverse=True)[:10]
        }

# Usage
inverted_index = InvertedIndex()

# Add documents
inverted_index.add_document(1, "The quick brown fox jumps over the lazy dog")
inverted_index.add_document(2, "A quick brown fox is very fast")
inverted_index.add_document(3, "The lazy dog sleeps all day")
inverted_index.add_document(4, "Quick silver fox runs fast")

# Search
results = inverted_index.search("quick fox")
print(f"Search results: {results}")

# Ranked search
ranked_results = inverted_index.search_ranked("quick fox")
print(f"Ranked results: {ranked_results}")

# Phrase search
phrase_results = inverted_index.search_phrase("quick brown fox")
print(f"Phrase search results: {phrase_results}")

# Statistics
stats = inverted_index.get_stats()
print(f"Inverted index stats: {stats}")
```

## Composite Indexes

### Multi-Column Index

```python
class CompositeIndex:
    def __init__(self, columns):
        self.columns = columns
        self.index = {}  # {composite_key: [row_ids]}
    
    def add_row(self, row_id, values):
        """Add row to composite index"""
        if len(values) != len(self.columns):
            raise ValueError("Values must match number of columns")
        
        # Create composite key
        composite_key = tuple(values)
        
        if composite_key not in self.index:
            self.index[composite_key] = []
        
        self.index[composite_key].append(row_id)
    
    def search_exact(self, values):
        """Search for exact match on all columns"""
        composite_key = tuple(values)
        return self.index.get(composite_key, [])
    
    def search_prefix(self, prefix_values):
        """Search using prefix of columns"""
        matching_rows = []
        
        for composite_key, row_ids in self.index.items():
            if self._matches_prefix(composite_key, prefix_values):
                matching_rows.extend(row_ids)
        
        return matching_rows
    
    def _matches_prefix(self, composite_key, prefix_values):
        """Check if composite key matches prefix"""
        if len(prefix_values) > len(composite_key):
            return False
        
        for i, value in enumerate(prefix_values):
            if composite_key[i] != value:
                return False
        
        return True
    
    def range_search(self, column_index, start_value, end_value):
        """Range search on specific column"""
        matching_rows = []
        
        for composite_key, row_ids in self.index.items():
            if column_index < len(composite_key):
                column_value = composite_key[column_index]
                if start_value <= column_value <= end_value:
                    matching_rows.extend(row_ids)
        
        return matching_rows
    
    def get_column_stats(self, column_index):
        """Get statistics for specific column"""
        if column_index >= len(self.columns):
            return None
        
        column_values = [key[column_index] for key in self.index.keys()]
        
        return {
            'distinct_values': len(set(column_values)),
            'min_value': min(column_values) if column_values else None,
            'max_value': max(column_values) if column_values else None,
            'most_common': max(set(column_values), key=column_values.count) if column_values else None
        }

# Usage
# Create composite index on (last_name, first_name, age)
composite_index = CompositeIndex(['last_name', 'first_name', 'age'])

# Add data
composite_index.add_row(1, ['Smith', 'John', 30])
composite_index.add_row(2, ['Smith', 'Jane', 25])
composite_index.add_row(3, ['Johnson', 'Bob', 35])
composite_index.add_row(4, ['Smith', 'John', 40])

# Exact search
exact_results = composite_index.search_exact(['Smith', 'John', 30])
print(f"Exact search results: {exact_results}")

# Prefix search
prefix_results = composite_index.search_prefix(['Smith'])
print(f"Prefix search results: {prefix_results}")

# Range search on age
age_range_results = composite_index.range_search(2, 25, 35)  # Column index 2 is age
print(f"Age range search results: {age_range_results}")
```

## Index Optimization

### 1. Index Usage Analyzer

```python
class IndexUsageAnalyzer:
    def __init__(self):
        self.query_log = []
        self.index_usage = defaultdict(int)
        self.slow_queries = []
    
    def log_query(self, query, indexes_used, execution_time):
        """Log query execution"""
        self.query_log.append({
            'query': query,
            'indexes_used': indexes_used,
            'execution_time': execution_time,
            'timestamp': time.time()
        })
        
        # Track index usage
        for index_name in indexes_used:
            self.index_usage[index_name] += 1
        
        # Track slow queries
        if execution_time > 1.0:  # > 1 second
            self.slow_queries.append({
                'query': query,
                'execution_time': execution_time,
                'indexes_used': indexes_used
            })
    
    def analyze_index_usage(self):
        """Analyze index usage patterns"""
        total_queries = len(self.query_log)
        
        usage_report = {
            'total_queries': total_queries,
            'index_usage': dict(self.index_usage),
            'unused_indexes': [],
            'most_used_indexes': [],
            'slow_queries_count': len(self.slow_queries)
        }
        
        # Find most used indexes
        if self.index_usage:
            sorted_usage = sorted(self.index_usage.items(), key=lambda x: x[1], reverse=True)
            usage_report['most_used_indexes'] = sorted_usage[:10]
        
        return usage_report
    
    def recommend_indexes(self):
        """Recommend new indexes based on query patterns"""
        recommendations = []
        
        # Analyze slow queries
        for slow_query in self.slow_queries:
            if not slow_query['indexes_used']:
                # Query without indexes
                recommendations.append({
                    'type': 'missing_index',
                    'query': slow_query['query'],
                    'execution_time': slow_query['execution_time'],
                    'recommendation': 'Consider adding index on frequently queried columns'
                })
        
        # Analyze query patterns
        where_clauses = []
        for query in self.query_log:
            where_clause = self._extract_where_clause(query['query'])
            if where_clause:
                where_clauses.append(where_clause)
        
        # Find common patterns
        from collections import Counter
        common_patterns = Counter(where_clauses)
        
        for pattern, count in common_patterns.most_common(5):
            if count > len(self.query_log) * 0.1:  # > 10% of queries
                recommendations.append({
                    'type': 'common_pattern',
                    'pattern': pattern,
                    'frequency': count,
                    'recommendation': f'Consider composite index for pattern: {pattern}'
                })
        
        return recommendations
    
    def _extract_where_clause(self, query):
        """Extract WHERE clause from query"""
        # Simple extraction - in practice, use proper SQL parser
        query_lower = query.lower()
        where_index = query_lower.find('where')
        
        if where_index != -1:
            return query[where_index:].strip()
        
        return None

# Usage
analyzer = IndexUsageAnalyzer()

# Log some queries
analyzer.log_query(
    "SELECT * FROM users WHERE last_name = 'Smith'",
    ['idx_last_name'],
    0.05
)

analyzer.log_query(
    "SELECT * FROM orders WHERE user_id = 123 AND status = 'active'",
    ['idx_user_id', 'idx_status'],
    0.12
)

analyzer.log_query(
    "SELECT * FROM products WHERE category = 'electronics' AND price > 100",
    [],  # No indexes used
    2.5   # Slow query
)

# Analyze usage
usage_report = analyzer.analyze_index_usage()
print(f"Index usage report: {usage_report}")

# Get recommendations
recommendations = analyzer.recommend_indexes()
print(f"Index recommendations: {recommendations}")
```

### 2. Index Maintenance

```python
class IndexMaintenance:
    def __init__(self, index):
        self.index = index
        self.maintenance_log = []
    
    def rebuild_index(self):
        """Rebuild index from scratch"""
        start_time = time.time()
        
        # Save current index
        old_index = self.index.copy() if hasattr(self.index, 'copy') else None
        
        try:
            # Clear and rebuild
            self.index.clear()
            
            # Rebuild logic depends on index type
            if isinstance(self.index, BTreeIndex):
                self._rebuild_btree()
            elif isinstance(self.index, HashIndex):
                self._rebuild_hash()
            
            end_time = time.time()
            
            self.maintenance_log.append({
                'operation': 'rebuild',
                'start_time': start_time,
                'end_time': end_time,
                'duration': end_time - start_time,
                'success': True
            })
            
            return True
            
        except Exception as e:
            # Restore old index on failure
            if old_index:
                self.index = old_index
            
            self.maintenance_log.append({
                'operation': 'rebuild',
                'start_time': start_time,
                'end_time': time.time(),
                'success': False,
                'error': str(e)
            })
            
            return False
    
    def _rebuild_btree(self):
        """Rebuild B-tree index"""
        # In practice, this would reload data from base table
        print("Rebuilding B-tree index...")
        # Implementation depends on data source
    
    def _rebuild_hash(self):
        """Rebuild hash index"""
        # In practice, this would reload data from base table
        print("Rebuilding hash index...")
        # Implementation depends on data source
    
    def analyze_fragmentation(self):
        """Analyze index fragmentation"""
        if isinstance(self.index, BTreeIndex):
            return self._analyze_btree_fragmentation()
        elif isinstance(self.index, HashIndex):
            return self._analyze_hash_fragmentation()
        
        return None
    
    def _analyze_btree_fragmentation(self):
        """Analyze B-tree fragmentation"""
        # Calculate fragmentation metrics
        total_nodes = self._count_btree_nodes(self.index.root)
        leaf_nodes = self._count_leaf_nodes(self.index.root)
        
        # Calculate fill factor
        total_capacity = total_nodes * self.index.max_keys
        actual_keys = self._count_total_keys(self.index.root)
        
        fill_factor = actual_keys / total_capacity if total_capacity > 0 else 0
        
        return {
            'total_nodes': total_nodes,
            'leaf_nodes': leaf_nodes,
            'fill_factor': fill_factor,
            'fragmentation_level': 1 - fill_factor,
            'needs_rebuild': fill_factor < 0.5
        }
    
    def _analyze_hash_fragmentation(self):
        """Analyze hash index fragmentation"""
        stats = self.index.get_stats()
        
        return {
            'load_factor': stats['load_factor'],
            'max_bucket_size': stats['max_bucket_size'],
            'avg_bucket_size': stats['avg_bucket_size'],
            'needs_rebuild': stats['max_bucket_size'] > stats['avg_bucket_size'] * 3
        }
    
    def _count_btree_nodes(self, node):
        """Count total B-tree nodes"""
        if not node:
            return 0
        
        count = 1
        if not node.is_leaf:
            for child in node.children:
                count += self._count_btree_nodes(child)
        
        return count
    
    def _count_leaf_nodes(self, node):
        """Count leaf nodes"""
        if not node:
            return 0
        
        if node.is_leaf:
            return 1
        
        count = 0
        for child in node.children:
            count += self._count_leaf_nodes(child)
        
        return count
    
    def _count_total_keys(self, node):
        """Count total keys in B-tree"""
        if not node:
            return 0
        
        count = len(node.keys)
        if not node.is_leaf:
            for child in node.children:
                count += self._count_total_keys(child)
        
        return count
    
    def schedule_maintenance(self, maintenance_type, schedule):
        """Schedule regular maintenance"""
        import schedule
        
        if maintenance_type == 'rebuild':
            schedule.every().week.do(self.rebuild_index)
        elif maintenance_type == 'analyze':
            schedule.every().day.do(self.analyze_fragmentation)
        
        print(f"Scheduled {maintenance_type} maintenance: {schedule}")
    
    def get_maintenance_history(self):
        """Get maintenance history"""
        return self.maintenance_log

# Usage
btree_index = BTreeIndex()
maintenance = IndexMaintenance(btree_index)

# Analyze fragmentation
fragmentation_report = maintenance.analyze_fragmentation()
print(f"Fragmentation report: {fragmentation_report}")

# Rebuild if needed
if fragmentation_report and fragmentation_report.get('needs_rebuild', False):
    success = maintenance.rebuild_index()
    print(f"Index rebuild success: {success}")

# Get maintenance history
history = maintenance.get_maintenance_history()
print(f"Maintenance history: {history}")
```

Indexing Strategies là yếu tố quan trọng để tối ưu performance database. Việc chọn đúng loại index và maintain chúng properly sẽ đảm bảo queries chạy nhanh và hiệu quả. 