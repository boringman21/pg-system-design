# Video Streaming Platform Design (Netflix/YouTube style)

**Tags**: #case-study #video-streaming #system-design #scalability
**Date**: 2024-01-01

## 🎯 Problem Statement

Thiết kế một video streaming platform giống như Netflix hoặc YouTube với khả năng:
- Streaming video chất lượng cao cho hàng triệu người dùng đồng thời
- Upload và xử lý video content
- Personalized recommendations
- Global content delivery
- Multiple device support

## 📋 Requirements Gathering

### **Functional Requirements**
```
Core Features:
✅ User registration và authentication
✅ Video upload và processing
✅ Video streaming với multiple quality levels
✅ Search và browse videos
✅ Personalized recommendations
✅ Comments và ratings
✅ Subscription management
✅ Analytics và reporting

Advanced Features:
✅ Live streaming
✅ Offline download
✅ Subtitles và multiple languages
✅ Chromecast/AirPlay support
✅ Social features (sharing, playlists)
```

### **Non-Functional Requirements**
```
Scale:
- 500 million registered users
- 100 million daily active users
- 50 million concurrent viewers during peak
- 1 billion hours watched daily
- 500 hours of video uploaded every minute

Performance:
- Video start time < 2 seconds
- 99.9% uptime
- Support 4K/8K streaming
- Global latency < 100ms
- Adaptive bitrate streaming

Storage:
- 50 petabytes of video content
- Multiple quality versions per video
- Geographic replication
- 5 years retention policy
```

## 📊 Capacity Estimation

### **Traffic Estimation**
```
Daily Active Users: 100M
Average session duration: 60 minutes
Peak concurrent users: 50M (during prime time)

QPS Calculation:
- Video metadata requests: 50M users × 10 requests/session ÷ 86400s = ~6K QPS
- Video streaming: 50M concurrent × 1 stream = 50M concurrent streams
- Search requests: 100M × 5 searches/day ÷ 86400s = ~6K QPS
- Video uploads: 500 hours/minute = 30K video segments/second
```

### **Storage Estimation**
```
Video Storage:
- Average video length: 10 minutes
- Upload rate: 500 hours/minute = 3,000 videos/minute
- Multiple quality versions: 4 (360p, 720p, 1080p, 4K)
- Average file sizes:
  * 360p: 50MB per 10min video
  * 720p: 150MB per 10min video  
  * 1080p: 300MB per 10min video
  * 4K: 1GB per 10min video

Daily storage needed:
3,000 videos/min × 1,440 min/day × (50+150+300+1000)MB = 6.5 PB/day

Metadata Storage:
- Video metadata: 1KB per video
- User data: 1MB per user
- Total: 100M users × 1MB + daily videos × 1KB ≈ 100TB
```

### **Bandwidth Estimation**
```
Video Streaming:
- 50M concurrent users
- Average bitrate: 2 Mbps (adaptive streaming)
- Total bandwidth: 50M × 2 Mbps = 100 Tbps

Video Upload:
- 500 hours/minute of content
- Average upload bitrate: 10 Mbps
- Upload bandwidth: 500 × 60 min × 10 Mbps = 300 Gbps

CDN Requirements:
- 95% traffic served from CDN
- CDN bandwidth: 95 Tbps
- Origin bandwidth: 5 Tbps
```

## 🏗️ High-Level Architecture

```
[Mobile/Web Clients] → [CDN] → [Load Balancer] → [API Gateway]
                                                      ↓
                                              [User Service]
                                              [Video Service]
                                              [Recommendation Service]
                                              [Search Service]
                                              [Analytics Service]
                                                      ↓
                                              [Message Queue]
                                                      ↓
                                              [Video Processing Pipeline]
                                                      ↓
                                              [Video Storage] ← [Metadata DB]
```

## 🔧 Detailed Design

### **Video Upload và Processing Pipeline**
```python
# Video Upload Service
class VideoUploadService:
    def __init__(self):
        self.upload_queue = RedisQueue('video_upload')
        self.storage = S3Storage()
        self.processing_queue = KafkaProducer('video_processing')
    
    async def upload_video(self, user_id: str, video_file: bytes, metadata: dict):
        # 1. Validate user permissions
        if not await self.validate_user(user_id):
            raise UnauthorizedError()
        
        # 2. Generate unique video ID
        video_id = self.generate_video_id()
        
        # 3. Upload raw video to staging storage
        raw_video_path = f"raw/{video_id}/{metadata['filename']}"
        await self.storage.upload(raw_video_path, video_file)
        
        # 4. Save video metadata
        video_metadata = {
            'video_id': video_id,
            'user_id': user_id,
            'title': metadata['title'],
            'description': metadata.get('description', ''),
            'upload_time': datetime.utcnow(),
            'status': 'processing',
            'raw_path': raw_video_path
        }
        await self.save_metadata(video_metadata)
        
        # 5. Queue for processing
        processing_job = {
            'video_id': video_id,
            'raw_path': raw_video_path,
            'user_id': user_id,
            'priority': self.get_user_priority(user_id)
        }
        await self.processing_queue.send(processing_job)
        
        return {'video_id': video_id, 'status': 'uploaded'}

# Video Processing Service
class VideoProcessingService:
    def __init__(self):
        self.ffmpeg_pool = ProcessPool(workers=50)
        self.storage = S3Storage()
        self.cdn = CloudFlareAPI()
        
    async def process_video(self, job: dict):
        video_id = job['video_id']
        raw_path = job['raw_path']
        
        try:
            # 1. Download raw video
            raw_video = await self.storage.download(raw_path)
            
            # 2. Extract metadata
            video_info = await self.extract_video_info(raw_video)
            
            # 3. Generate thumbnails
            thumbnails = await self.generate_thumbnails(raw_video)
            
            # 4. Process multiple quality versions
            quality_versions = await self.process_qualities(raw_video)
            
            # 5. Upload processed videos
            processed_paths = {}
            for quality, video_data in quality_versions.items():
                path = f"processed/{video_id}/{quality}.mp4"
                await self.storage.upload(path, video_data)
                processed_paths[quality] = path
            
            # 6. Upload to CDN
            cdn_urls = await self.upload_to_cdn(processed_paths)
            
            # 7. Update metadata
            await self.update_video_status(video_id, {
                'status': 'ready',
                'processed_paths': processed_paths,
                'cdn_urls': cdn_urls,
                'video_info': video_info,
                'thumbnails': thumbnails,
                'processing_completed': datetime.utcnow()
            })
            
            # 8. Trigger content indexing for search
            await self.trigger_content_indexing(video_id)
            
        except Exception as e:
            await self.handle_processing_error(video_id, str(e))
    
    async def process_qualities(self, raw_video: bytes) -> dict:
        """Process video into multiple quality versions"""
        qualities = {
            '360p': {'width': 640, 'height': 360, 'bitrate': '800k'},
            '720p': {'width': 1280, 'height': 720, 'bitrate': '2500k'},
            '1080p': {'width': 1920, 'height': 1080, 'bitrate': '5000k'},
            '4k': {'width': 3840, 'height': 2160, 'bitrate': '15000k'}
        }
        
        processed_videos = {}
        
        # Process in parallel
        tasks = []
        for quality_name, params in qualities.items():
            task = self.ffmpeg_transcode(raw_video, params)
            tasks.append((quality_name, task))
        
        results = await asyncio.gather(*[task for _, task in tasks])
        
        for (quality_name, _), result in zip(tasks, results):
            processed_videos[quality_name] = result
        
        return processed_videos
    
    async def ffmpeg_transcode(self, input_video: bytes, params: dict) -> bytes:
        """Use FFmpeg to transcode video"""
        ffmpeg_cmd = [
            'ffmpeg', '-i', 'pipe:0',  # Input from stdin
            '-vf', f"scale={params['width']}:{params['height']}",
            '-b:v', params['bitrate'],
            '-c:v', 'libx264',
            '-preset', 'medium',
            '-crf', '23',
            '-f', 'mp4',
            'pipe:1'  # Output to stdout
        ]
        
        process = await asyncio.create_subprocess_exec(
            *ffmpeg_cmd,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        
        stdout, stderr = await process.communicate(input_video)
        
        if process.returncode != 0:
            raise Exception(f"FFmpeg error: {stderr.decode()}")
        
        return stdout
```

### **Video Streaming Service**
```python
# Adaptive Bitrate Streaming
class VideoStreamingService:
    def __init__(self):
        self.cdn_client = CDNClient()
        self.analytics = AnalyticsService()
        self.cache = RedisCache()
        
    async def get_video_manifest(self, video_id: str, user_context: dict):
        """Generate HLS/DASH manifest for adaptive streaming"""
        
        # 1. Get video metadata
        video_metadata = await self.get_video_metadata(video_id)
        if not video_metadata or video_metadata['status'] != 'ready':
            raise VideoNotAvailableError()
        
        # 2. Check user permissions
        if not await self.check_video_access(user_context['user_id'], video_id):
            raise UnauthorizedError()
        
        # 3. Get optimal CDN endpoint
        user_location = await self.get_user_location(user_context['ip'])
        cdn_endpoint = await self.get_nearest_cdn(user_location)
        
        # 4. Generate adaptive streaming manifest
        manifest = await self.generate_hls_manifest(
            video_metadata, 
            cdn_endpoint,
            user_context
        )
        
        # 5. Log streaming event
        await self.analytics.log_stream_start(user_context['user_id'], video_id)
        
        return manifest
    
    async def generate_hls_manifest(self, video_metadata: dict, cdn_endpoint: str, user_context: dict):
        """Generate HLS manifest with adaptive bitrate"""
        
        base_url = f"{cdn_endpoint}/video/{video_metadata['video_id']}"
        
        # Get available quality versions
        qualities = video_metadata['processed_paths'].keys()
        
        # Generate master playlist
        master_playlist = "#EXTM3U\n#EXT-X-VERSION:3\n"
        
        quality_info = {
            '360p': {'bandwidth': 800000, 'resolution': '640x360'},
            '720p': {'bandwidth': 2500000, 'resolution': '1280x720'},
            '1080p': {'bandwidth': 5000000, 'resolution': '1920x1080'},
            '4k': {'bandwidth': 15000000, 'resolution': '3840x2160'}
        }
        
        for quality in sorted(qualities):
            if quality in quality_info:
                info = quality_info[quality]
                master_playlist += f"""#EXT-X-STREAM-INF:BANDWIDTH={info['bandwidth']},RESOLUTION={info['resolution']}
{base_url}/{quality}/playlist.m3u8
"""
        
        return master_playlist
    
    async def get_video_segment(self, video_id: str, quality: str, segment_id: str):
        """Get video segment for streaming"""
        
        # 1. Check cache first
        cache_key = f"segment:{video_id}:{quality}:{segment_id}"
        cached_segment = await self.cache.get(cache_key)
        if cached_segment:
            return cached_segment
        
        # 2. Get from CDN
        cdn_url = f"https://cdn.example.com/video/{video_id}/{quality}/{segment_id}.ts"
        segment_data = await self.cdn_client.get(cdn_url)
        
        # 3. Cache for future requests
        await self.cache.set(cache_key, segment_data, ttl=3600)  # 1 hour
        
        return segment_data

# Video Analytics Service
class VideoAnalyticsService:
    def __init__(self):
        self.metrics_db = InfluxDB()
        self.real_time_metrics = RedisStream()
        
    async def log_stream_start(self, user_id: str, video_id: str):
        event = {
            'event_type': 'stream_start',
            'user_id': user_id,
            'video_id': video_id,
            'timestamp': datetime.utcnow().isoformat(),
            'session_id': self.generate_session_id()
        }
        
        await self.real_time_metrics.add('video_events', event)
        await self.metrics_db.write_point('video_streams', event)
    
    async def log_quality_change(self, session_id: str, old_quality: str, new_quality: str, reason: str):
        event = {
            'event_type': 'quality_change',
            'session_id': session_id,
            'old_quality': old_quality,
            'new_quality': new_quality,
            'reason': reason,  # 'bandwidth', 'user_selection', 'device_capability'
            'timestamp': datetime.utcnow().isoformat()
        }
        
        await self.real_time_metrics.add('video_events', event)
    
    async def log_buffer_event(self, session_id: str, buffer_duration: float, video_position: float):
        event = {
            'event_type': 'buffer',
            'session_id': session_id,
            'buffer_duration': buffer_duration,
            'video_position': video_position,
            'timestamp': datetime.utcnow().isoformat()
        }
        
        await self.real_time_metrics.add('video_events', event)
```

### **Recommendation System**
```python
# Collaborative Filtering + Content-Based Recommendations
class RecommendationService:
    def __init__(self):
        self.user_behavior_db = CassandraDB()
        self.ml_model_store = MLModelStore()
        self.cache = RedisCache()
        self.feature_store = FeatureStore()
        
    async def get_recommendations(self, user_id: str, context: dict) -> List[str]:
        """Get personalized video recommendations"""
        
        # 1. Check cache first
        cache_key = f"recommendations:{user_id}:{context.get('device_type', 'web')}"
        cached_recs = await self.cache.get(cache_key)
        if cached_recs:
            return json.loads(cached_recs)
        
        # 2. Get user features
        user_features = await self.get_user_features(user_id)
        
        # 3. Get collaborative filtering recommendations
        collaborative_recs = await self.collaborative_filtering(user_id, user_features)
        
        # 4. Get content-based recommendations
        content_recs = await self.content_based_filtering(user_id, user_features)
        
        # 5. Get trending videos
        trending_recs = await self.get_trending_videos(context)
        
        # 6. Ensemble different recommendation sources
        final_recs = await self.ensemble_recommendations([
            ('collaborative', collaborative_recs, 0.5),
            ('content', content_recs, 0.3),
            ('trending', trending_recs, 0.2)
        ])
        
        # 7. Apply business rules and filters
        filtered_recs = await self.apply_business_filters(user_id, final_recs)
        
        # 8. Cache results
        await self.cache.set(cache_key, json.dumps(filtered_recs), ttl=1800)  # 30 min
        
        return filtered_recs
    
    async def collaborative_filtering(self, user_id: str, user_features: dict) -> List[str]:
        """Matrix factorization based collaborative filtering"""
        
        # Get user-item interaction matrix
        user_interactions = await self.get_user_interactions(user_id)
        similar_users = await self.find_similar_users(user_id, user_features)
        
        # Get videos liked by similar users
        candidate_videos = []
        for similar_user_id, similarity_score in similar_users:
            similar_user_videos = await self.get_user_liked_videos(similar_user_id)
            for video_id, rating in similar_user_videos:
                if video_id not in user_interactions:
                    candidate_videos.append((video_id, rating * similarity_score))
        
        # Sort by predicted rating
        candidate_videos.sort(key=lambda x: x[1], reverse=True)
        
        return [video_id for video_id, _ in candidate_videos[:100]]
    
    async def content_based_filtering(self, user_id: str, user_features: dict) -> List[str]:
        """Content-based recommendations using video features"""
        
        # Get user's video watching history
        watched_videos = await self.get_user_watched_videos(user_id, limit=50)
        
        # Extract content preferences
        preferred_genres = Counter()
        preferred_creators = Counter()
        preferred_duration = []
        
        for video_id in watched_videos:
            video_metadata = await self.get_video_metadata(video_id)
            preferred_genres.update(video_metadata.get('genres', []))
            preferred_creators[video_metadata.get('creator_id')] += 1
            preferred_duration.append(video_metadata.get('duration', 0))
        
        # Find videos with similar content features
        avg_duration = sum(preferred_duration) / len(preferred_duration)
        
        candidate_query = {
            'genres': list(preferred_genres.keys()),
            'duration_range': (avg_duration * 0.5, avg_duration * 1.5),
            'exclude_watched': watched_videos
        }
        
        candidates = await self.search_videos_by_content(candidate_query)
        
        # Score based on content similarity
        scored_candidates = []
        for video_id, video_metadata in candidates:
            score = self.calculate_content_similarity(
                user_preferences={
                    'genres': preferred_genres,
                    'creators': preferred_creators,
                    'duration': avg_duration
                },
                video_features=video_metadata
            )
            scored_candidates.append((video_id, score))
        
        scored_candidates.sort(key=lambda x: x[1], reverse=True)
        return [video_id for video_id, _ in scored_candidates[:100]]
    
    async def real_time_recommendations(self, user_id: str, current_video_id: str):
        """Generate real-time recommendations based on current watching behavior"""
        
        # Get current video metadata
        current_video = await self.get_video_metadata(current_video_id)
        
        # Find videos with similar tags/categories
        similar_videos = await self.find_similar_videos(
            current_video_id, 
            similarity_threshold=0.7
        )
        
        # Get "watch next" recommendations
        watch_next = await self.get_autoplay_candidates(user_id, current_video_id)
        
        # Combine and rank
        recommendations = {
            'similar_videos': similar_videos[:10],
            'watch_next': watch_next[:5],
            'from_same_creator': await self.get_creator_videos(
                current_video['creator_id'], 
                exclude=[current_video_id]
            )[:5]
        }
        
        return recommendations
```

### **Search Service**
```python
# Video Search Service using Elasticsearch
class VideoSearchService:
    def __init__(self):
        self.elasticsearch = ElasticsearchClient()
        self.cache = RedisCache()
        self.analytics = SearchAnalyticsService()
        
    async def search_videos(self, query: str, user_context: dict, filters: dict = None):
        """Search videos with personalization and ranking"""
        
        # 1. Parse and clean query
        cleaned_query = self.clean_search_query(query)
        
        # 2. Build Elasticsearch query
        search_query = await self.build_search_query(
            cleaned_query, 
            user_context, 
            filters or {}
        )
        
        # 3. Execute search
        search_results = await self.elasticsearch.search(
            index='videos',
            body=search_query,
            size=50
        )
        
        # 4. Re-rank results based on user preferences
        personalized_results = await self.personalize_search_results(
            search_results['hits']['hits'],
            user_context
        )
        
        # 5. Log search event
        await self.analytics.log_search(
            user_context['user_id'],
            query,
            len(personalized_results)
        )
        
        return personalized_results
    
    async def build_search_query(self, query: str, user_context: dict, filters: dict):
        """Build Elasticsearch query with boosting and filtering"""
        
        user_id = user_context.get('user_id')
        user_preferences = await self.get_user_search_preferences(user_id)
        
        search_query = {
            "query": {
                "bool": {
                    "must": [
                        {
                            "multi_match": {
                                "query": query,
                                "fields": [
                                    "title^3",           # Title is most important
                                    "description^2",     # Description is important
                                    "tags^2",            # Tags are important
                                    "transcript^1",      # Transcript has lower weight
                                    "creator_name^2"     # Creator name is important
                                ],
                                "type": "best_fields",
                                "fuzziness": "AUTO"
                            }
                        }
                    ],
                    "should": [
                        # Boost videos from preferred genres
                        {
                            "terms": {
                                "genres": user_preferences.get('preferred_genres', []),
                                "boost": 1.5
                            }
                        },
                        # Boost recent videos
                        {
                            "range": {
                                "upload_date": {
                                    "gte": "now-30d",
                                    "boost": 1.2
                                }
                            }
                        },
                        # Boost popular videos
                        {
                            "range": {
                                "view_count": {
                                    "gte": 1000,
                                    "boost": 1.1
                                }
                            }
                        }
                    ],
                    "filter": []
                }
            },
            "sort": [
                {"_score": {"order": "desc"}},
                {"relevance_score": {"order": "desc"}},
                {"view_count": {"order": "desc"}}
            ]
        }
        
        # Apply filters
        if filters.get('genre'):
            search_query["query"]["bool"]["filter"].append({
                "term": {"genres": filters['genre']}
            })
        
        if filters.get('duration'):
            search_query["query"]["bool"]["filter"].append({
                "range": {"duration": filters['duration']}
            })
        
        if filters.get('upload_date'):
            search_query["query"]["bool"]["filter"].append({
                "range": {"upload_date": filters['upload_date']}
            })
        
        return search_query
    
    async def autocomplete_suggestions(self, partial_query: str, user_context: dict):
        """Provide search autocomplete suggestions"""
        
        cache_key = f"autocomplete:{partial_query}:{user_context.get('user_id', 'anon')}"
        cached_suggestions = await self.cache.get(cache_key)
        if cached_suggestions:
            return json.loads(cached_suggestions)
        
        # Query completion suggestions
        completion_query = {
            "suggest": {
                "video_suggest": {
                    "prefix": partial_query,
                    "completion": {
                        "field": "suggest",
                        "size": 10,
                        "contexts": {
                            "user_context": [user_context.get('country', 'US')]
                        }
                    }
                }
            }
        }
        
        suggestions = await self.elasticsearch.search(
            index='videos',
            body=completion_query
        )
        
        # Format suggestions
        formatted_suggestions = []
        for suggestion in suggestions['suggest']['video_suggest'][0]['options']:
            formatted_suggestions.append({
                'text': suggestion['text'],
                'score': suggestion['_score'],
                'video_count': suggestion.get('_source', {}).get('doc_count', 0)
            })
        
        # Cache suggestions
        await self.cache.set(cache_key, json.dumps(formatted_suggestions), ttl=300)
        
        return formatted_suggestions
```

## 📡 Content Delivery Network (CDN) Strategy

### **Multi-Tier CDN Architecture**
```
Global CDN Strategy:

Tier 1: Global CDN (CloudFlare/Akamai)
- Static content (thumbnails, metadata)
- Popular videos (top 20%)
- Global edge locations

Tier 2: Regional CDN 
- Regional popular content
- Live streaming origins
- Regional data centers

Tier 3: Origin Servers
- Complete video catalog
- Long-tail content
- Backup and disaster recovery

Content Placement Algorithm:
1. Hot content (>1M views) → Global CDN
2. Warm content (>100K views) → Regional CDN  
3. Cold content (<100K views) → Origin servers
4. Live content → Regional streaming servers
```

### **Adaptive Bitrate Streaming**
```python
# ABR Algorithm Implementation
class AdaptiveBitrateController:
    def __init__(self):
        self.quality_levels = [
            {'name': '360p', 'bitrate': 800, 'resolution': (640, 360)},
            {'name': '720p', 'bitrate': 2500, 'resolution': (1280, 720)},
            {'name': '1080p', 'bitrate': 5000, 'resolution': (1920, 1080)},
            {'name': '4k', 'bitrate': 15000, 'resolution': (3840, 2160)}
        ]
        
    def select_optimal_quality(self, 
                             available_bandwidth: int,  # kbps
                             buffer_level: float,       # seconds
                             device_capability: dict,
                             user_preference: str = 'auto'):
        """Select optimal video quality based on current conditions"""
        
        if user_preference != 'auto':
            return user_preference
        
        # Filter by device capability
        suitable_qualities = []
        for quality in self.quality_levels:
            if (quality['resolution'][0] <= device_capability['max_width'] and
                quality['resolution'][1] <= device_capability['max_height']):
                suitable_qualities.append(quality)
        
        # Select based on bandwidth and buffer
        if buffer_level < 5:  # Low buffer, be conservative
            target_bandwidth = available_bandwidth * 0.7
        elif buffer_level > 20:  # High buffer, can be aggressive
            target_bandwidth = available_bandwidth * 0.9
        else:
            target_bandwidth = available_bandwidth * 0.8
        
        # Find highest quality that fits bandwidth
        selected_quality = suitable_qualities[0]  # Default to lowest
        for quality in suitable_qualities:
            if quality['bitrate'] <= target_bandwidth:
                selected_quality = quality
            else:
                break
        
        return selected_quality['name']
    
    def should_switch_quality(self,
                            current_quality: str,
                            bandwidth_history: List[int],  # Last 10 measurements
                            buffer_level: float,
                            rebuffer_count: int):
        """Decide if we should switch video quality"""
        
        avg_bandwidth = sum(bandwidth_history) / len(bandwidth_history)
        current_bitrate = next(q['bitrate'] for q in self.quality_levels 
                             if q['name'] == current_quality)
        
        # Switch down if:
        # 1. Bandwidth is consistently lower than current bitrate
        # 2. Buffer is getting low
        # 3. Too many rebuffers
        if (avg_bandwidth < current_bitrate * 1.2 or 
            buffer_level < 10 or 
            rebuffer_count > 2):
            
            # Find lower quality
            for i, quality in enumerate(self.quality_levels):
                if quality['name'] == current_quality and i > 0:
                    return self.quality_levels[i-1]['name']
        
        # Switch up if:
        # 1. Bandwidth is consistently higher
        # 2. Buffer is healthy
        # 3. No recent rebuffers
        elif (avg_bandwidth > current_bitrate * 1.5 and 
              buffer_level > 20 and 
              rebuffer_count == 0):
            
            # Find higher quality
            for i, quality in enumerate(self.quality_levels):
                if (quality['name'] == current_quality and 
                    i < len(self.quality_levels) - 1):
                    return self.quality_levels[i+1]['name']
        
        return current_quality  # No change
```

## 🔐 Security và Content Protection

### **DRM và Content Security**
```python
# Digital Rights Management
class DRMService:
    def __init__(self):
        self.encryption_service = AESEncryption()
        self.license_server = WidevineLicenseServer()
        
    async def protect_video_content(self, video_id: str, content_policy: dict):
        """Apply DRM protection to video content"""
        
        # 1. Generate content encryption key
        content_key = self.encryption_service.generate_key()
        
        # 2. Encrypt video segments
        encrypted_segments = await self.encrypt_video_segments(video_id, content_key)
        
        # 3. Generate license for different DRM systems
        licenses = {
            'widevine': await self.license_server.generate_widevine_license(
                video_id, content_key, content_policy
            ),
            'fairplay': await self.generate_fairplay_license(
                video_id, content_key, content_policy
            ),
            'playready': await self.generate_playready_license(
                video_id, content_key, content_policy
            )
        }
        
        return {
            'encrypted_content': encrypted_segments,
            'licenses': licenses,
            'key_id': content_key.key_id
        }
    
    async def validate_playback_license(self, user_id: str, video_id: str, device_info: dict):
        """Validate user's license to play protected content"""
        
        # 1. Check user subscription status
        subscription = await self.check_user_subscription(user_id)
        if not subscription.is_active:
            raise LicenseValidationError("Invalid subscription")
        
        # 2. Check content access rights
        access_rights = await self.check_content_access(user_id, video_id)
        if not access_rights.can_play:
            raise LicenseValidationError("No access to this content")
        
        # 3. Check device limits
        active_devices = await self.get_active_devices(user_id)
        if len(active_devices) >= subscription.device_limit:
            raise LicenseValidationError("Device limit exceeded")
        
        # 4. Generate playback token
        playback_token = await self.generate_playback_token(
            user_id, video_id, device_info
        )
        
        return playback_token

# Content Security
class ContentSecurityService:
    def __init__(self):
        self.watermarking = VideoWatermarkingService()
        self.piracy_detection = PiracyDetectionService()
        
    async def apply_forensic_watermarking(self, video_id: str, user_id: str):
        """Apply invisible watermarking for piracy tracking"""
        
        watermark_data = {
            'user_id': user_id,
            'video_id': video_id,
            'timestamp': datetime.utcnow().isoformat(),
            'session_id': self.generate_session_id()
        }
        
        watermarked_segments = await self.watermarking.apply_watermark(
            video_id, watermark_data
        )
        
        return watermarked_segments
    
    async def detect_piracy(self, video_fingerprint: str):
        """Detect unauthorized distribution using content fingerprinting"""
        
        similar_content = await self.piracy_detection.find_similar_content(
            video_fingerprint
        )
        
        for match in similar_content:
            if match.similarity > 0.95:  # Likely piracy
                await self.report_piracy_incident(match)
        
        return similar_content
```

## 📊 Analytics và Monitoring

### **Real-time Video Analytics**
```python
# Video Analytics Pipeline
class VideoAnalyticsPipeline:
    def __init__(self):
        self.stream_processor = KafkaStreams()
        self.time_series_db = InfluxDB()
        self.dashboard = GrafanaDashboard()
        
    async def process_viewing_events(self):
        """Process real-time viewing events"""
        
        async for event in self.stream_processor.consume('video_events'):
            event_data = json.loads(event.value)
            
            # Real-time metrics calculation
            await self.update_real_time_metrics(event_data)
            
            # Store for historical analysis
            await self.store_historical_data(event_data)
            
            # Trigger alerts if needed
            await self.check_alerts(event_data)
    
    async def calculate_video_quality_metrics(self, video_id: str, time_window: str):
        """Calculate video quality metrics"""
        
        # Query recent events for this video
        events = await self.time_series_db.query(f"""
            SELECT 
                COUNT(*) as total_views,
                AVG(buffer_ratio) as avg_buffer_ratio,
                AVG(bitrate_switches) as avg_quality_switches,
                AVG(startup_time) as avg_startup_time,
                PERCENTILE(rebuffer_duration, 95) as p95_rebuffer_duration
            FROM video_events 
            WHERE video_id = '{video_id}' 
            AND time > now() - {time_window}
        """)
        
        # Calculate quality score
        quality_score = self.calculate_qoe_score(events[0])
        
        return {
            'video_id': video_id,
            'quality_score': quality_score,
            'metrics': events[0],
            'timestamp': datetime.utcnow()
        }
    
    def calculate_qoe_score(self, metrics: dict) -> float:
        """Calculate Quality of Experience score (0-100)"""
        
        # Startup time impact (0-25 points)
        startup_score = max(0, 25 - (metrics['avg_startup_time'] / 100))
        
        # Rebuffering impact (0-25 points)  
        rebuffer_score = max(0, 25 - (metrics['p95_rebuffer_duration'] / 1000))
        
        # Quality stability (0-25 points)
        stability_score = max(0, 25 - (metrics['avg_quality_switches'] * 2))
        
        # Buffer health (0-25 points)
        buffer_score = min(25, metrics['avg_buffer_ratio'] * 25)
        
        total_score = startup_score + rebuffer_score + stability_score + buffer_score
        
        return round(total_score, 2)

# A/B Testing for Video Features
class VideoABTestingService:
    def __init__(self):
        self.experiment_store = ExperimentStore()
        self.metrics_collector = MetricsCollector()
        
    async def run_recommendation_experiment(self, user_id: str, experiment_name: str):
        """Run A/B test for recommendation algorithms"""
        
        experiment = await self.experiment_store.get_experiment(experiment_name)
        if not experiment or not experiment.is_active:
            return None
        
        # Determine user's test group
        user_hash = hashlib.md5(f"{user_id}{experiment.seed}".encode()).hexdigest()
        group_id = int(user_hash, 16) % 100
        
        if group_id < experiment.control_percentage:
            variant = 'control'
            recommendations = await self.get_control_recommendations(user_id)
        else:
            variant = 'treatment'
            recommendations = await self.get_treatment_recommendations(user_id, experiment)
        
        # Log experiment exposure
        await self.metrics_collector.log_experiment_exposure(
            user_id, experiment_name, variant
        )
        
        return {
            'recommendations': recommendations,
            'experiment': experiment_name,
            'variant': variant
        }
```

## 💡 Optimizations và Best Practices

### **Performance Optimizations**
```
Video Delivery Optimization:
✅ Use HTTP/2 for multiplexed streaming
✅ Implement segment prefetching
✅ Use byte-range requests for seeking
✅ Compress video metadata
✅ Optimize thumbnail delivery

Caching Strategy:
✅ Cache popular videos at edge
✅ Use browser cache for static assets
✅ Implement smart cache invalidation
✅ Cache video metadata aggressively
✅ Use Redis for session data

Database Optimization:
✅ Partition user data by geography
✅ Use read replicas for analytics
✅ Denormalize for read-heavy workloads
✅ Index on commonly queried fields
✅ Use time-series DB for metrics
```

### **Cost Optimization**
```
Storage Costs:
- Archive old content to cheaper storage
- Delete unused quality versions
- Compress thumbnails and metadata
- Use lifecycle policies for automatic archiving

Bandwidth Costs:
- Optimize video encoding (HEVC/AV1)
- Use smart CDN routing
- Implement P2P delivery for popular content
- Compress API responses

Infrastructure Costs:
- Use spot instances for video processing
- Auto-scale based on demand
- Optimize database queries
- Use reserved instances for baseline capacity
```

---

**Checklist cho Video Streaming Platform:**
- [ ] Design scalable video upload pipeline
- [ ] Implement adaptive bitrate streaming
- [ ] Set up global CDN strategy
- [ ] Build recommendation system
- [ ] Implement search functionality
- [ ] Add DRM and content protection
- [ ] Set up comprehensive analytics
- [ ] Plan for live streaming
- [ ] Design mobile apps
- [ ] Implement cost optimization

**Trade-offs chính:**
- **Quality vs Bandwidth**: Higher quality = more bandwidth cost
- **Latency vs Quality**: Lower latency = potential quality compromise  
- **Cost vs Scale**: Global CDN = higher costs but better performance
- **Personalization vs Privacy**: Better recommendations = more data collection

**Remember**: Video streaming là một trong những use cases phức tạp nhất, cần cân bằng giữa technical performance và business requirements! 🎬 