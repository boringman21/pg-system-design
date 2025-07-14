# Video Streaming Platform Design

**Tags**: #case-study #video-streaming #scalability #cdn
**Date**: 2024-01-01

## 📝 Overview

Design một video streaming platform có khả năng serve millions of users, handle video uploads, encoding, storage, và deliver content globally với low latency. Platform cần support multiple video qualities, live streaming, và recommendations.

## 🎯 Requirements Analysis

### **Functional Requirements**
```
Content Management:
- Video upload (multiple formats)
- Video encoding & transcoding
- Thumbnail generation
- Metadata management

User Features:
- Video browsing & search
- Video playback (multiple qualities)
- User subscriptions & channels
- Comments & ratings
- Playlists & watch history

Creator Features:
- Channel management
- Video analytics
- Monetization features
- Live streaming

Admin Features:
- Content moderation
- Analytics dashboard
- System monitoring
```

### **Non-Functional Requirements**
```python
scale_requirements = {
    "users": {
        "daily_active_users": "2 billion",
        "concurrent_viewers": "100 million",
        "peak_traffic": "150% of average"
    },
    "content": {
        "videos_uploaded_per_day": "500 hours/minute",
        "total_storage": "1 exabyte+",
        "supported_formats": ["MP4", "AVI", "MOV", "WebM"]
    },
    "performance": {
        "video_start_time": "< 2 seconds",
        "buffering_ratio": "< 0.1%",
        "availability": "99.9%",
        "global_latency": "< 150ms"
    },
    "bandwidth": {
        "peak_egress": "100+ Tbps",
        "encoding_capacity": "Millions of hours/day",
        "cdn_coverage": "Global"
    }
}
```

## 🏗️ High-Level Architecture

### **System Overview**
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Mobile Apps   │    │   Web Clients   │    │   Smart TVs     │
│                 │    │                 │    │                 │
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌─────────────▼─────────────┐
                    │       CDN Network         │
                    │    (Global Distribution)  │
                    └─────────────┬─────────────┘
                                 │
                    ┌─────────────▼─────────────┐
                    │      API Gateway          │
                    │   (Load Balancing)        │
                    └─────────────┬─────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                       │                        │
┌───────▼────────┐    ┌─────────▼────────┐    ┌─────────▼────────┐
│ Video Services │    │  User Services   │    │ Search Services  │
│                │    │                  │    │                  │
└────────────────┘    └──────────────────┘    └──────────────────┘
        │                       │                        │
┌───────▼────────┐    ┌─────────▼────────┐    ┌─────────▼────────┐
│ Video Storage  │    │  User Database   │    │ Search Database  │
│   + Encoding   │    │                  │    │ (Elasticsearch)  │
└────────────────┘    └──────────────────┘    └──────────────────┘
```

### **Service Architecture**
```python
class VideoStreamingArchitecture:
    def __init__(self):
        self.services = {
            "upload_service": {
                "responsibilities": ["Video upload", "Validation", "Initial processing"],
                "components": ["Upload API", "File validation", "Virus scanning"],
                "storage": "Temporary storage + Queue"
            },
            "encoding_service": {
                "responsibilities": ["Video transcoding", "Quality variants", "Thumbnails"],
                "components": ["FFmpeg workers", "GPU encoding", "Thumbnail generator"],
                "storage": "Encoded video storage"
            },
            "metadata_service": {
                "responsibilities": ["Video metadata", "Search indexing", "Categories"],
                "database": "PostgreSQL + Elasticsearch",
                "cache": "Redis"
            },
            "user_service": {
                "responsibilities": ["User management", "Subscriptions", "Preferences"],
                "database": "PostgreSQL",
                "cache": "Redis"
            },
            "recommendation_service": {
                "responsibilities": ["Content recommendations", "ML models"],
                "components": ["ML pipeline", "Collaborative filtering", "Content-based"],
                "storage": "Feature store + Model store"
            },
            "analytics_service": {
                "responsibilities": ["View tracking", "Performance metrics", "User behavior"],
                "database": "ClickHouse/BigQuery",
                "streaming": "Kafka"
            },
            "content_delivery": {
                "responsibilities": ["Video streaming", "Adaptive bitrate", "Caching"],
                "components": ["CDN", "Edge servers", "Streaming protocols"],
                "protocols": ["HLS", "DASH", "WebRTC"]
            }
        }
```

## 📹 Video Processing Pipeline

### **Upload & Encoding Service**
```python
import asyncio
import subprocess
import boto3
from typing import List, Dict
import uuid

class VideoUploadService:
    def __init__(self):
        self.s3_client = boto3.client('s3')
        self.sqs_client = boto3.client('sqs')
        self.temp_bucket = "video-uploads-temp"
        self.processed_bucket = "video-processed"
        self.encoding_queue = "video-encoding-queue"
    
    async def upload_video(self, file_data, metadata):
        """
        Handle video upload with validation
        """
        upload_id = str(uuid.uuid4())
        
        try:
            # Validate file
            validation_result = await self.validate_video_file(file_data)
            if not validation_result["valid"]:
                raise ValueError(f"Invalid video file: {validation_result['reason']}")
            
            # Upload to temporary storage
            temp_key = f"temp/{upload_id}/{metadata['filename']}"
            upload_url = self.generate_presigned_upload_url(temp_key)
            
            # Store metadata
            video_metadata = {
                "upload_id": upload_id,
                "filename": metadata["filename"],
                "title": metadata["title"],
                "description": metadata.get("description", ""),
                "category": metadata.get("category", "general"),
                "user_id": metadata["user_id"],
                "upload_time": datetime.utcnow(),
                "file_size": metadata["file_size"],
                "duration": validation_result.get("duration"),
                "resolution": validation_result.get("resolution"),
                "status": "uploaded"
            }
            
            await self.store_video_metadata(video_metadata)
            
            # Queue for encoding
            await self.queue_for_encoding(upload_id, temp_key, video_metadata)
            
            return {
                "upload_id": upload_id,
                "upload_url": upload_url,
                "status": "success"
            }
            
        except Exception as e:
            await self.handle_upload_error(upload_id, str(e))
            raise
    
    async def validate_video_file(self, file_data) -> Dict:
        """
        Validate uploaded video file
        """
        # Check file size (max 10GB)
        if file_data.size > 10 * 1024 * 1024 * 1024:
            return {"valid": False, "reason": "File too large"}
        
        # Check file type
        allowed_types = ['.mp4', '.avi', '.mov', '.webm', '.mkv']
        if not any(file_data.filename.lower().endswith(ext) for ext in allowed_types):
            return {"valid": False, "reason": "Unsupported file format"}
        
        # Use FFprobe to validate video
        try:
            probe_result = subprocess.run([
                'ffprobe', '-v', 'quiet', '-print_format', 'json',
                '-show_format', '-show_streams', file_data.filepath
            ], capture_output=True, text=True)
            
            if probe_result.returncode != 0:
                return {"valid": False, "reason": "Invalid video file"}
            
            video_info = json.loads(probe_result.stdout)
            
            # Extract video metadata
            video_stream = next(
                (s for s in video_info['streams'] if s['codec_type'] == 'video'),
                None
            )
            
            if not video_stream:
                return {"valid": False, "reason": "No video stream found"}
            
            return {
                "valid": True,
                "duration": float(video_info['format']['duration']),
                "resolution": f"{video_stream['width']}x{video_stream['height']}",
                "codec": video_stream['codec_name'],
                "bitrate": int(video_info['format'].get('bit_rate', 0))
            }
            
        except Exception as e:
            return {"valid": False, "reason": f"Validation error: {str(e)}"}

class VideoEncodingService:
    def __init__(self):
        self.encoding_profiles = {
            "480p": {"width": 854, "height": 480, "bitrate": "1000k"},
            "720p": {"width": 1280, "height": 720, "bitrate": "2500k"},
            "1080p": {"width": 1920, "height": 1080, "bitrate": "5000k"},
            "1440p": {"width": 2560, "height": 1440, "bitrate": "9000k"},
            "2160p": {"width": 3840, "height": 2160, "bitrate": "18000k"}
        }
    
    async def process_video(self, upload_id, source_path, metadata):
        """
        Encode video into multiple qualities
        """
        try:
            # Update status
            await self.update_video_status(upload_id, "encoding")
            
            # Generate thumbnails
            thumbnails = await self.generate_thumbnails(source_path, upload_id)
            
            # Encode video variants
            encoded_variants = await self.encode_video_variants(
                source_path, upload_id, metadata
            )
            
            # Create HLS/DASH manifests
            manifests = await self.create_streaming_manifests(
                upload_id, encoded_variants
            )
            
            # Update metadata with encoded info
            encoding_result = {
                "variants": encoded_variants,
                "thumbnails": thumbnails,
                "manifests": manifests,
                "status": "ready"
            }
            
            await self.update_video_metadata(upload_id, encoding_result)
            
            # Cleanup temporary files
            await self.cleanup_temp_files(source_path)
            
            # Notify completion
            await self.notify_encoding_complete(upload_id)
            
        except Exception as e:
            await self.handle_encoding_error(upload_id, str(e))
            raise
    
    async def encode_video_variants(self, source_path, upload_id, metadata):
        """
        Create multiple quality variants using FFmpeg
        """
        variants = {}
        original_resolution = metadata.get("resolution", "1920x1080")
        original_width = int(original_resolution.split('x')[0])
        
        # Determine which qualities to encode based on source resolution
        target_qualities = []
        for quality, profile in self.encoding_profiles.items():
            if profile["width"] <= original_width:
                target_qualities.append((quality, profile))
        
        # Encode each quality variant
        encoding_tasks = []
        for quality, profile in target_qualities:
            task = self.encode_single_variant(
                source_path, upload_id, quality, profile
            )
            encoding_tasks.append(task)
        
        # Process encodings in parallel (with limits)
        semaphore = asyncio.Semaphore(3)  # Max 3 concurrent encodings
        
        async def encode_with_limit(task):
            async with semaphore:
                return await task
        
        results = await asyncio.gather(*[
            encode_with_limit(task) for task in encoding_tasks
        ])
        
        for quality, result in zip([q[0] for q in target_qualities], results):
            variants[quality] = result
        
        return variants
    
    async def encode_single_variant(self, source_path, upload_id, quality, profile):
        """
        Encode single quality variant
        """
        output_path = f"encoded/{upload_id}/{quality}.mp4"
        
        # FFmpeg command for encoding
        cmd = [
            'ffmpeg', '-i', source_path,
            '-c:v', 'libx264',  # Video codec
            '-preset', 'medium',  # Encoding speed vs compression
            '-crf', '23',  # Quality factor
            '-maxrate', profile["bitrate"],
            '-bufsize', f"{int(profile['bitrate'][:-1]) * 2}k",
            '-vf', f'scale={profile["width"]}:{profile["height"]}',
            '-c:a', 'aac',  # Audio codec
            '-b:a', '128k',  # Audio bitrate
            '-movflags', '+faststart',  # Web optimization
            '-f', 'mp4',
            output_path
        ]
        
        # Run encoding
        process = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        
        stdout, stderr = await process.communicate()
        
        if process.returncode != 0:
            raise Exception(f"Encoding failed for {quality}: {stderr.decode()}")
        
        # Upload encoded file to storage
        s3_key = f"videos/{upload_id}/{quality}.mp4"
        await self.upload_to_s3(output_path, self.processed_bucket, s3_key)
        
        # Get file info
        file_stats = os.stat(output_path)
        
        return {
            "quality": quality,
            "resolution": f"{profile['width']}x{profile['height']}",
            "bitrate": profile["bitrate"],
            "file_size": file_stats.st_size,
            "s3_key": s3_key,
            "cdn_url": f"https://cdn.videoplatform.com/{s3_key}"
        }
    
    async def generate_thumbnails(self, source_path, upload_id):
        """
        Generate video thumbnails at different timestamps
        """
        thumbnails = []
        
        # Get video duration first
        probe_cmd = [
            'ffprobe', '-v', 'quiet', '-show_entries',
            'format=duration', '-of', 'csv=p=0', source_path
        ]
        
        process = await asyncio.create_subprocess_exec(
            *probe_cmd,
            stdout=asyncio.subprocess.PIPE
        )
        
        stdout, _ = await process.communicate()
        duration = float(stdout.decode().strip())
        
        # Generate thumbnails at 10%, 25%, 50%, 75% of video duration
        timestamps = [duration * p for p in [0.1, 0.25, 0.5, 0.75]]
        
        for i, timestamp in enumerate(timestamps):
            thumbnail_path = f"thumbnails/{upload_id}/thumb_{i}.jpg"
            
            cmd = [
                'ffmpeg', '-i', source_path,
                '-ss', str(timestamp),
                '-vframes', '1',
                '-vf', 'scale=320:180',
                '-q:v', '2',
                thumbnail_path
            ]
            
            process = await asyncio.create_subprocess_exec(
                *cmd,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE
            )
            
            await process.communicate()
            
            if process.returncode == 0:
                # Upload to S3
                s3_key = f"thumbnails/{upload_id}/thumb_{i}.jpg"
                await self.upload_to_s3(thumbnail_path, self.processed_bucket, s3_key)
                
                thumbnails.append({
                    "index": i,
                    "timestamp": timestamp,
                    "s3_key": s3_key,
                    "url": f"https://cdn.videoplatform.com/{s3_key}"
                })
        
        return thumbnails
```

## 🌐 Content Delivery Network

### **CDN Strategy**
```python
class CDNManagement:
    def __init__(self):
        self.edge_locations = self.setup_edge_locations()
        self.cache_policies = self.define_cache_policies()
        self.origin_servers = self.setup_origin_servers()
    
    def setup_edge_locations(self):
        """
        Global edge server configuration
        """
        return {
            "north_america": {
                "locations": ["us-east-1", "us-west-1", "ca-central-1"],
                "capacity": "50 Tbps",
                "cache_size": "1 PB per location"
            },
            "europe": {
                "locations": ["eu-west-1", "eu-central-1", "eu-north-1"],
                "capacity": "40 Tbps",
                "cache_size": "800 TB per location"
            },
            "asia_pacific": {
                "locations": ["ap-southeast-1", "ap-northeast-1", "ap-south-1"],
                "capacity": "30 Tbps",
                "cache_size": "600 TB per location"
            }
        }
    
    def define_cache_policies(self):
        """
        Caching strategies for different content types
        """
        return {
            "video_segments": {
                "ttl": "7 days",
                "cache_key": "video_id + quality + segment_number",
                "cache_behavior": "cache_everything"
            },
            "thumbnails": {
                "ttl": "30 days", 
                "cache_key": "video_id + thumbnail_index",
                "cache_behavior": "cache_everything"
            },
            "manifests": {
                "ttl": "1 hour",
                "cache_key": "video_id + user_location",
                "cache_behavior": "cache_with_validation"
            },
            "api_responses": {
                "ttl": "5 minutes",
                "cache_key": "endpoint + query_params",
                "cache_behavior": "cache_selected_responses"
            }
        }
    
    def get_optimal_cdn_node(self, user_location, video_id):
        """
        Select best CDN node for user request
        """
        # Calculate distance to edge locations
        distances = {}
        for region, config in self.edge_locations.items():
            for location in config["locations"]:
                distance = self.calculate_distance(user_location, location)
                distances[location] = distance
        
        # Select closest location
        closest_location = min(distances.items(), key=lambda x: x[1])[0]
        
        # Check if video is cached at closest location
        if self.is_video_cached(closest_location, video_id):
            return closest_location
        
        # If not cached, find next best location with cache
        for location in sorted(distances.items(), key=lambda x: x[1]):
            if self.is_video_cached(location[0], video_id):
                return location[0]
        
        # If not cached anywhere, use closest and trigger cache warming
        self.trigger_cache_warming(closest_location, video_id)
        return closest_location

class AdaptiveBitrateStreaming:
    """
    Implement adaptive bitrate streaming (HLS/DASH)
    """
    def __init__(self):
        self.quality_ladder = [
            {"name": "480p", "bandwidth": 1000000},
            {"name": "720p", "bandwidth": 2500000}, 
            {"name": "1080p", "bandwidth": 5000000},
            {"name": "1440p", "bandwidth": 9000000},
            {"name": "2160p", "bandwidth": 18000000}
        ]
    
    def generate_hls_manifest(self, video_id, available_qualities):
        """
        Generate HLS master playlist
        """
        manifest_lines = ["#EXTM3U", "#EXT-X-VERSION:6"]
        
        for quality in available_qualities:
            quality_info = next(
                q for q in self.quality_ladder if q["name"] == quality["quality"]
            )
            
            manifest_lines.extend([
                f"#EXT-X-STREAM-INF:BANDWIDTH={quality_info['bandwidth']},"
                f"RESOLUTION={quality['resolution']},CODECS=\"avc1.64001f,mp4a.40.2\"",
                f"{quality['quality']}/playlist.m3u8"
            ])
        
        return "\n".join(manifest_lines)
    
    def generate_quality_playlist(self, video_id, quality, segment_duration=10):
        """
        Generate playlist for specific quality
        """
        segments = self.get_video_segments(video_id, quality)
        
        playlist_lines = [
            "#EXTM3U",
            "#EXT-X-VERSION:3",
            f"#EXT-X-TARGETDURATION:{segment_duration}",
            "#EXT-X-PLAYLIST-TYPE:VOD"
        ]
        
        for i, segment in enumerate(segments):
            playlist_lines.extend([
                f"#EXTINF:{segment['duration']:.2f},",
                f"{video_id}/{quality}/segment_{i:04d}.ts"
            ])
        
        playlist_lines.append("#EXT-X-ENDLIST")
        
        return "\n".join(playlist_lines)
    
    def select_initial_quality(self, user_bandwidth, device_info):
        """
        Select appropriate initial quality based on user conditions
        """
        # Consider user's bandwidth
        suitable_qualities = [
            q for q in self.quality_ladder 
            if q["bandwidth"] <= user_bandwidth * 0.8  # 80% of available bandwidth
        ]
        
        if not suitable_qualities:
            return self.quality_ladder[0]  # Lowest quality
        
        # Consider device capabilities
        if device_info.get("is_mobile"):
            # Prefer lower quality on mobile to save battery/data
            mobile_qualities = [q for q in suitable_qualities if q["bandwidth"] <= 2500000]
            return mobile_qualities[-1] if mobile_qualities else suitable_qualities[0]
        
        return suitable_qualities[-1]  # Highest suitable quality
```

## 🔍 Search & Discovery

### **Video Search Service**
```python
from elasticsearch import Elasticsearch
import numpy as np

class VideoSearchService:
    def __init__(self):
        self.es = Elasticsearch(['localhost:9200'])
        self.index_name = "video_index"
        self.setup_index()
    
    def setup_index(self):
        """
        Create Elasticsearch index with proper mappings
        """
        mapping = {
            "mappings": {
                "properties": {
                    "video_id": {"type": "keyword"},
                    "title": {
                        "type": "text",
                        "analyzer": "standard",
                        "fields": {
                            "exact": {"type": "keyword"},
                            "suggest": {"type": "completion"}
                        }
                    },
                    "description": {"type": "text", "analyzer": "standard"},
                    "tags": {"type": "keyword"},
                    "category": {"type": "keyword"},
                    "duration": {"type": "integer"},
                    "upload_date": {"type": "date"},
                    "view_count": {"type": "long"},
                    "like_count": {"type": "long"},
                    "channel_id": {"type": "keyword"},
                    "channel_name": {"type": "text"},
                    "language": {"type": "keyword"},
                    "quality_levels": {"type": "keyword"},
                    "popularity_score": {"type": "float"},
                    "embedding_vector": {
                        "type": "dense_vector",
                        "dims": 512
                    }
                }
            }
        }
        
        if not self.es.indices.exists(index=self.index_name):
            self.es.indices.create(index=self.index_name, body=mapping)
    
    def index_video(self, video_data):
        """
        Index video for search
        """
        # Generate content embedding for semantic search
        content_text = f"{video_data['title']} {video_data['description']}"
        embedding = self.generate_content_embedding(content_text)
        
        doc = {
            "video_id": video_data["video_id"],
            "title": video_data["title"],
            "description": video_data["description"],
            "tags": video_data.get("tags", []),
            "category": video_data["category"],
            "duration": video_data["duration"],
            "upload_date": video_data["upload_date"],
            "view_count": video_data.get("view_count", 0),
            "like_count": video_data.get("like_count", 0),
            "channel_id": video_data["channel_id"],
            "channel_name": video_data["channel_name"],
            "language": video_data.get("language", "en"),
            "quality_levels": video_data.get("quality_levels", []),
            "popularity_score": self.calculate_popularity_score(video_data),
            "embedding_vector": embedding
        }
        
        self.es.index(index=self.index_name, id=video_data["video_id"], body=doc)
    
    def search_videos(self, query, filters=None, page=1, size=20, sort_by="relevance"):
        """
        Search videos with multiple ranking factors
        """
        # Build search query
        if query:
            search_query = {
                "bool": {
                    "should": [
                        {
                            "multi_match": {
                                "query": query,
                                "fields": [
                                    "title^3",      # Title boost
                                    "description^1",
                                    "tags^2",       # Tags boost
                                    "channel_name^1.5"
                                ],
                                "type": "best_fields"
                            }
                        },
                        {
                            "match": {
                                "title.exact": {
                                    "query": query,
                                    "boost": 5  # Exact title match boost
                                }
                            }
                        }
                    ],
                    "minimum_should_match": 1
                }
            }
        else:
            search_query = {"match_all": {}}
        
        # Apply filters
        if filters:
            filter_clauses = []
            
            if "category" in filters:
                filter_clauses.append({"term": {"category": filters["category"]}})
            
            if "duration_range" in filters:
                filter_clauses.append({
                    "range": {
                        "duration": {
                            "gte": filters["duration_range"]["min"],
                            "lte": filters["duration_range"]["max"]
                        }
                    }
                })
            
            if "upload_date" in filters:
                filter_clauses.append({
                    "range": {
                        "upload_date": {
                            "gte": filters["upload_date"]["from"]
                        }
                    }
                })
            
            if filter_clauses:
                search_query = {
                    "bool": {
                        "must": [search_query],
                        "filter": filter_clauses
                    }
                }
        
        # Define sorting
        sort_options = {
            "relevance": [{"_score": {"order": "desc"}}],
            "recent": [{"upload_date": {"order": "desc"}}],
            "popular": [{"view_count": {"order": "desc"}}],
            "rating": [{"like_count": {"order": "desc"}}],
            "duration": [{"duration": {"order": "asc"}}]
        }
        
        sort = sort_options.get(sort_by, sort_options["relevance"])
        
        # Add popularity boost to relevance
        if sort_by == "relevance":
            search_query = {
                "function_score": {
                    "query": search_query,
                    "functions": [
                        {
                            "field_value_factor": {
                                "field": "popularity_score",
                                "factor": 1.2,
                                "modifier": "log1p"
                            }
                        },
                        {
                            "field_value_factor": {
                                "field": "view_count",
                                "factor": 0.0001,
                                "modifier": "log1p"
                            }
                        }
                    ],
                    "score_mode": "multiply",
                    "boost_mode": "multiply"
                }
            }
        
        # Execute search
        response = self.es.search(
            index=self.index_name,
            body={
                "query": search_query,
                "sort": sort,
                "from": (page - 1) * size,
                "size": size,
                "highlight": {
                    "fields": {
                        "title": {"pre_tags": ["<mark>"], "post_tags": ["</mark>"]},
                        "description": {"pre_tags": ["<mark>"], "post_tags": ["</mark>"]}
                    }
                }
            }
        )
        
        # Format results
        videos = []
        for hit in response["hits"]["hits"]:
            video = hit["_source"]
            video["score"] = hit["_score"]
            
            if "highlight" in hit:
                video["highlights"] = hit["highlight"]
            
            videos.append(video)
        
        return {
            "videos": videos,
            "total": response["hits"]["total"]["value"],
            "page": page,
            "total_pages": (response["hits"]["total"]["value"] + size - 1) // size,
            "took": response["took"]
        }
    
    def get_search_suggestions(self, prefix, size=10):
        """
        Get search suggestions using completion suggester
        """
        response = self.es.search(
            index=self.index_name,
            body={
                "suggest": {
                    "video_suggest": {
                        "prefix": prefix,
                        "completion": {
                            "field": "title.suggest",
                            "size": size
                        }
                    }
                }
            }
        )
        
        suggestions = []
        for option in response["suggest"]["video_suggest"][0]["options"]:
            suggestions.append({
                "text": option["text"],
                "video_id": option["_source"]["video_id"],
                "score": option["_score"]
            })
        
        return suggestions
    
    def calculate_popularity_score(self, video_data):
        """
        Calculate video popularity score based on multiple factors
        """
        view_count = video_data.get("view_count", 0)
        like_count = video_data.get("like_count", 0)
        comment_count = video_data.get("comment_count", 0)
        
        # Time decay factor (newer videos get slight boost)
        upload_date = video_data.get("upload_date", datetime.utcnow())
        days_since_upload = (datetime.utcnow() - upload_date).days
        time_factor = max(0.1, 1 / (1 + days_since_upload / 30))  # Decay over 30 days
        
        # Engagement rate
        engagement_rate = (like_count + comment_count) / max(view_count, 1)
        
        # Combined score
        popularity = (
            np.log1p(view_count) * 0.6 +
            np.log1p(like_count) * 0.3 +
            engagement_rate * 100 * 0.1
        ) * time_factor
        
        return min(popularity, 1000)  # Cap at 1000
```

## 🔗 Related Topics

- [[CDN Design]] - Content delivery optimization
- [[Database Sharding]] - Scaling video metadata storage
- [[Microservices Architecture]] - Service decomposition
- [[Caching Strategies]] - Video content caching

## 📚 Further Reading

- YouTube architecture deep dive
- Netflix streaming technology
- Video encoding best practices
- CDN optimization strategies

---

**Key Takeaway**: Video streaming platforms require sophisticated infrastructure để handle massive scale. Key components include efficient video encoding pipelines, global CDN distribution, adaptive bitrate streaming, intelligent search systems, và robust recommendation engines. 