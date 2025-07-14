# File Storage System Design

## Tổng quan hệ thống

### Yêu cầu chức năng
- **Upload file**: Tải lên file với kích thước khác nhau (1KB - 10GB)
- **Download file**: Tải xuống file với tốc độ cao
- **File versioning**: Quản lý phiên bản file
- **File sharing**: Chia sẻ file với quyền hạn
- **Search**: Tìm kiếm file theo metadata
- **Synchronization**: Đồng bộ file giữa devices

### Yêu cầu phi chức năng
- **Availability**: 99.9% uptime
- **Durability**: 99.99999999% (11 9s)
- **Consistency**: Eventually consistent
- **Scalability**: Hàng tỷ file, PB storage
- **Performance**: Upload/download < 100ms latency

## Kiến trúc hệ thống

### High-Level Architecture
```
Client Apps → Load Balancer → API Gateway → Metadata Service
                                        → File Service
                                        → Storage Service
```

### Core Components

#### 1. Metadata Service
```python
class MetadataService:
    def __init__(self):
        self.db = CassandraDB()
        self.cache = RedisCache()
    
    def store_file_metadata(self, file_info):
        metadata = {
            'file_id': file_info.id,
            'user_id': file_info.user_id,
            'filename': file_info.name,
            'size': file_info.size,
            'content_type': file_info.type,
            'created_at': datetime.now(),
            'updated_at': datetime.now(),
            'version': 1,
            'storage_locations': file_info.locations,
            'checksum': file_info.checksum
        }
        
        # Store in database
        self.db.insert('file_metadata', metadata)
        
        # Cache for quick access
        self.cache.set(f"file:{file_info.id}", metadata, ttl=3600)
        
        return metadata
    
    def get_file_metadata(self, file_id):
        # Try cache first
        cached = self.cache.get(f"file:{file_id}")
        if cached:
            return cached
        
        # Fallback to database
        metadata = self.db.get('file_metadata', {'file_id': file_id})
        if metadata:
            self.cache.set(f"file:{file_id}", metadata, ttl=3600)
        
        return metadata
```

#### 2. File Service
```python
class FileService:
    def __init__(self):
        self.metadata_service = MetadataService()
        self.storage_service = StorageService()
        self.chunk_size = 4 * 1024 * 1024  # 4MB chunks
    
    def upload_file(self, file_data, user_id):
        # Generate file ID
        file_id = self.generate_file_id()
        
        # Calculate checksum
        checksum = hashlib.sha256(file_data).hexdigest()
        
        # Check for deduplication
        existing_file = self.check_duplicate(checksum)
        if existing_file:
            return self.create_reference(existing_file, user_id)
        
        # Split file into chunks
        chunks = self.split_into_chunks(file_data)
        
        # Upload chunks to storage
        storage_locations = []
        for chunk in chunks:
            locations = self.storage_service.store_chunk(chunk)
            storage_locations.extend(locations)
        
        # Store metadata
        file_info = FileInfo(
            id=file_id,
            user_id=user_id,
            size=len(file_data),
            checksum=checksum,
            locations=storage_locations
        )
        
        metadata = self.metadata_service.store_file_metadata(file_info)
        
        return {'file_id': file_id, 'status': 'uploaded'}
    
    def download_file(self, file_id, user_id):
        # Check permissions
        if not self.check_permission(file_id, user_id):
            raise PermissionError("Access denied")
        
        # Get metadata
        metadata = self.metadata_service.get_file_metadata(file_id)
        if not metadata:
            raise FileNotFoundError("File not found")
        
        # Retrieve chunks from storage
        chunks = []
        for location in metadata['storage_locations']:
            chunk = self.storage_service.retrieve_chunk(location)
            chunks.append(chunk)
        
        # Reconstruct file
        file_data = b''.join(chunks)
        
        # Verify integrity
        if hashlib.sha256(file_data).hexdigest() != metadata['checksum']:
            raise IntegrityError("File corruption detected")
        
        return file_data
    
    def split_into_chunks(self, file_data):
        chunks = []
        for i in range(0, len(file_data), self.chunk_size):
            chunk = file_data[i:i + self.chunk_size]
            chunks.append(chunk)
        return chunks
```

#### 3. Storage Service (Distributed)
```python
class StorageService:
    def __init__(self):
        self.storage_nodes = [
            'storage-node-1.example.com',
            'storage-node-2.example.com',
            'storage-node-3.example.com'
        ]
        self.replication_factor = 3
    
    def store_chunk(self, chunk_data):
        chunk_id = self.generate_chunk_id()
        
        # Select storage nodes using consistent hashing
        selected_nodes = self.select_nodes(chunk_id)
        
        # Store chunk on multiple nodes for redundancy
        storage_locations = []
        for node in selected_nodes:
            location = self.store_on_node(node, chunk_id, chunk_data)
            storage_locations.append(location)
        
        return storage_locations
    
    def retrieve_chunk(self, location):
        try:
            return self.retrieve_from_node(location)
        except NodeUnavailableError:
            # Try alternative locations
            alternative_locations = self.get_alternative_locations(location)
            for alt_location in alternative_locations:
                try:
                    return self.retrieve_from_node(alt_location)
                except NodeUnavailableError:
                    continue
            raise ChunkUnavailableError("Chunk cannot be retrieved")
    
    def select_nodes(self, chunk_id):
        # Consistent hashing implementation
        hash_value = hashlib.md5(chunk_id.encode()).hexdigest()
        hash_int = int(hash_value, 16)
        
        selected_nodes = []
        for i in range(self.replication_factor):
            node_index = (hash_int + i) % len(self.storage_nodes)
            selected_nodes.append(self.storage_nodes[node_index])
        
        return selected_nodes
```

### Data Storage Schema

#### Metadata Database (Cassandra)
```sql
CREATE TABLE file_metadata (
    file_id UUID PRIMARY KEY,
    user_id UUID,
    filename TEXT,
    size BIGINT,
    content_type TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    version INT,
    storage_locations LIST<TEXT>,
    checksum TEXT,
    tags SET<TEXT>,
    is_deleted BOOLEAN DEFAULT FALSE
);

CREATE INDEX ON file_metadata (user_id);
CREATE INDEX ON file_metadata (checksum);
```

#### File Versioning
```python
class FileVersioning:
    def create_version(self, file_id, new_content):
        current_metadata = self.metadata_service.get_file_metadata(file_id)
        
        # Create new version
        new_version = current_metadata['version'] + 1
        
        # Store new content
        new_file_id = self.generate_version_id(file_id, new_version)
        self.upload_file(new_content, current_metadata['user_id'])
        
        # Update metadata with version info
        version_metadata = {
            'file_id': new_file_id,
            'original_file_id': file_id,
            'version': new_version,
            'previous_version': current_metadata['version']
        }
        
        self.metadata_service.store_file_metadata(version_metadata)
        
        return new_file_id
    
    def get_version_history(self, file_id):
        versions = self.db.query(
            'file_metadata', 
            {'original_file_id': file_id}
        )
        return sorted(versions, key=lambda x: x['version'])
```

## Các tính năng nâng cao

### 1. Deduplication
```python
class DeduplicationService:
    def __init__(self):
        self.fingerprint_db = BloomFilter(capacity=1000000, error_rate=0.1)
    
    def check_duplicate(self, file_checksum):
        # Check if file already exists
        if file_checksum in self.fingerprint_db:
            existing_file = self.metadata_service.get_by_checksum(file_checksum)
            if existing_file:
                return existing_file
        
        # Add to fingerprint database
        self.fingerprint_db.add(file_checksum)
        return None
    
    def create_reference(self, existing_file, user_id):
        # Create reference instead of storing duplicate
        reference = {
            'file_id': self.generate_file_id(),
            'user_id': user_id,
            'reference_to': existing_file['file_id'],
            'created_at': datetime.now()
        }
        
        return reference
```

### 2. File Compression
```python
class CompressionService:
    def compress_file(self, file_data, content_type):
        if content_type.startswith('image/'):
            # Use specific image compression
            return self.compress_image(file_data)
        elif content_type.startswith('video/'):
            # Use video compression
            return self.compress_video(file_data)
        else:
            # Use general compression (gzip)
            return gzip.compress(file_data)
    
    def decompress_file(self, compressed_data, compression_type):
        if compression_type == 'gzip':
            return gzip.decompress(compressed_data)
        elif compression_type == 'image':
            return self.decompress_image(compressed_data)
        elif compression_type == 'video':
            return self.decompress_video(compressed_data)
```

### 3. CDN Integration
```python
class CDNService:
    def __init__(self):
        self.cdn_endpoints = [
            'https://cdn1.example.com',
            'https://cdn2.example.com',
            'https://cdn3.example.com'
        ]
    
    def cache_popular_files(self, file_id):
        # Cache frequently accessed files
        download_count = self.get_download_count(file_id)
        if download_count > 1000:  # Threshold for popular files
            file_data = self.file_service.download_file(file_id)
            
            for endpoint in self.cdn_endpoints:
                self.upload_to_cdn(endpoint, file_id, file_data)
    
    def get_download_url(self, file_id):
        # Check if file is cached in CDN
        if self.is_cached_in_cdn(file_id):
            return f"{self.cdn_endpoints[0]}/files/{file_id}"
        else:
            return f"/api/files/{file_id}/download"
```

### 4. Security & Permissions
```python
class SecurityService:
    def encrypt_file(self, file_data, user_key):
        # Encrypt file with user's key
        cipher = AES.new(user_key, AES.MODE_GCM)
        ciphertext, tag = cipher.encrypt_and_digest(file_data)
        
        return {
            'ciphertext': ciphertext,
            'nonce': cipher.nonce,
            'tag': tag
        }
    
    def decrypt_file(self, encrypted_data, user_key):
        cipher = AES.new(user_key, AES.MODE_GCM, nonce=encrypted_data['nonce'])
        plaintext = cipher.decrypt_and_verify(
            encrypted_data['ciphertext'], 
            encrypted_data['tag']
        )
        return plaintext
    
    def check_permission(self, file_id, user_id):
        permissions = self.get_file_permissions(file_id)
        return user_id in permissions['read_users'] or \
               user_id == permissions['owner']
```

## Monitoring & Metrics

### Key Metrics
```python
class MetricsCollector:
    def __init__(self):
        self.metrics = {
            'upload_latency': Histogram('file_upload_duration_seconds'),
            'download_latency': Histogram('file_download_duration_seconds'),
            'storage_usage': Gauge('total_storage_bytes'),
            'error_rate': Counter('file_operation_errors_total')
        }
    
    def record_upload(self, duration, size):
        self.metrics['upload_latency'].observe(duration)
        self.metrics['storage_usage'].inc(size)
    
    def record_download(self, duration):
        self.metrics['download_latency'].observe(duration)
    
    def record_error(self, operation, error_type):
        self.metrics['error_rate'].labels(
            operation=operation, 
            error_type=error_type
        ).inc()
```

### Health Checks
```python
class HealthChecker:
    def check_storage_nodes(self):
        healthy_nodes = 0
        for node in self.storage_nodes:
            if self.ping_node(node):
                healthy_nodes += 1
        
        health_ratio = healthy_nodes / len(self.storage_nodes)
        
        if health_ratio < 0.5:
            raise CriticalError("More than 50% storage nodes down")
        elif health_ratio < 0.8:
            self.alert_manager.send_warning("Storage nodes degraded")
        
        return health_ratio
```

## Scalability Patterns

### Horizontal Scaling
- **Storage nodes**: Thêm node khi storage đầy
- **Metadata sharding**: Phân chia theo user_id
- **Load balancing**: Round-robin cho upload/download

### Caching Strategy
- **Metadata cache**: Redis cho metadata thường truy cập
- **File cache**: CDN cho file popular
- **Chunk cache**: Local cache cho chunk gần đây

### Database Optimization
- **Indexing**: Index trên user_id, checksum, created_at
- **Partitioning**: Partition theo thời gian
- **Replication**: Master-slave cho read scaling

## Trade-offs & Considerations

### Consistency vs Availability
- **Eventually consistent**: Metadata có thể lag
- **Strong consistency**: Cho file operations quan trọng

### Storage vs Performance
- **Compression**: Giảm storage, tăng CPU
- **Replication**: Tăng availability, tăng cost

### Cost vs Durability
- **Replication factor**: 3 replicas cho 99.99% durability
- **Cross-region backup**: Cho disaster recovery

## Implementation Tips

### 1. Chunking Strategy
```python
def optimal_chunk_size(file_size):
    if file_size < 1024 * 1024:  # < 1MB
        return file_size  # No chunking
    elif file_size < 100 * 1024 * 1024:  # < 100MB
        return 1024 * 1024  # 1MB chunks
    else:
        return 4 * 1024 * 1024  # 4MB chunks
```

### 2. Error Handling
```python
class FileOperationError(Exception):
    pass

class ChunkCorruptionError(FileOperationError):
    pass

class InsufficientStorageError(FileOperationError):
    pass

def handle_upload_error(error):
    if isinstance(error, InsufficientStorageError):
        # Scale up storage
        storage_manager.add_storage_node()
    elif isinstance(error, ChunkCorruptionError):
        # Retry with different nodes
        return retry_with_different_nodes()
```

### 3. Performance Optimization
```python
class PerformanceOptimizer:
    def optimize_upload(self, file_data):
        # Parallel chunk upload
        chunks = self.split_into_chunks(file_data)
        
        with ThreadPoolExecutor(max_workers=10) as executor:
            futures = [
                executor.submit(self.upload_chunk, chunk) 
                for chunk in chunks
            ]
            
            results = [future.result() for future in futures]
        
        return results
    
    def optimize_download(self, file_id):
        # Parallel chunk download
        metadata = self.get_file_metadata(file_id)
        
        with ThreadPoolExecutor(max_workers=10) as executor:
            futures = [
                executor.submit(self.download_chunk, location)
                for location in metadata['storage_locations']
            ]
            
            chunks = [future.result() for future in futures]
        
        return b''.join(chunks)
```

File Storage System này cung cấp foundation mạnh mẽ cho việc lưu trữ file phân tán với high availability, durability và scalability. 