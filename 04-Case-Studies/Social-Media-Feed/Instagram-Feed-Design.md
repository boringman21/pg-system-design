# Instagram Feed System Design

**Tags**: #case-study #social-media #feed #instagram #scalability
**Date**: 2024-01-01
**Difficulty**: Medium-Hard

## 📝 Problem Statement

Thiết kế hệ thống newsfeed cho Instagram với khả năng:
- Hiển thị posts từ users mà user follow
- Upload và share photos/videos
- Like, comment, share posts
- Support 1 billion users với high availability

## 🎯 Requirements Clarification

### Functional Requirements
- [ ] **User Management**: Register, login, follow/unfollow users
- [ ] **Content Creation**: Upload photos/videos với captions, hashtags
- [ ] **Feed Generation**: Personalized timeline cho mỗi user
- [ ] **Interactions**: Like, comment, share posts
- [ ] **Search**: Search users, hashtags, locations
- [ ] **Notifications**: Real-time notifications cho interactions

### Non-functional Requirements
- [ ] **Scalability**: 1B users, 500M daily active users
- [ ] **Performance**: Feed load time < 200ms
- [ ] **Availability**: 99.9% uptime
- [ ] **Consistency**: Eventually consistent feed (acceptable delay)
- [ ] **Storage**: Petabytes of photos/videos

### Scale Estimates
- **Users**: 1B registered, 500M daily active
- **Posts**: 100M photos/videos uploaded per day
- **Feed Generation**: 500M * 50 posts viewed = 25B feed requests/day
- **Read/Write Ratio**: 100:1 (more reads than writes)
- **Storage**: 100M posts * 2MB avg = 200TB per day

## 🏗️ High-Level Design

```
[Mobile Apps] → [CDN] → [Load Balancer] → [API Gateway]
                                               ↓
[Web Servers] → [Cache Layer] → [Feed Generation Service]
      ↓              ↓                    ↓
[User Service] [Media Service] [Notification Service]
      ↓              ↓                    ↓
[User DB] [Media Storage] [Message Queue] [Timeline DB]
```

### Core Components
1. **Load Balancer**: Route traffic, health checking
2. **API Gateway**: Rate limiting, authentication, request routing
3. **Web Servers**: Handle HTTP requests, business logic
4. **User Service**: User profiles, relationships, authentication
5. **Media Service**: Upload, processing, storage của photos/videos
6. **Feed Generation Service**: Create personalized timelines
7. **Notification Service**: Real-time notifications
8. **Cache Layer**: Redis cho hot data
9. **CDN**: Global content delivery
10. **Databases**: User data, metadata, timelines

## 🔍 Detailed Design

### API Design
```python
# User Management APIs
POST /api/v1/users/register
POST /api/v1/users/login
GET /api/v1/users/{user_id}
POST /api/v1/users/{user_id}/follow
DELETE /api/v1/users/{user_id}/follow

# Content APIs
POST /api/v1/posts
GET /api/v1/posts/{post_id}
DELETE /api/v1/posts/{post_id}
POST /api/v1/posts/{post_id}/like
POST /api/v1/posts/{post_id}/comment

# Feed APIs
GET /api/v1/feed/timeline?user_id={user_id}&limit=20&offset=0
GET /api/v1/feed/explore?user_id={user_id}

# Search APIs
GET /api/v1/search/users?q={query}
GET /api/v1/search/hashtags?q={query}
```

### Database Schema
```sql
-- User Management
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    profile_image_url TEXT,
    bio TEXT,
    follower_count INTEGER DEFAULT 0,
    following_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User Relationships
CREATE TABLE user_follows (
    follower_id UUID NOT NULL,
    following_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, following_id),
    FOREIGN KEY (follower_id) REFERENCES users(user_id),
    FOREIGN KEY (following_id) REFERENCES users(user_id)
);

-- Posts
CREATE TABLE posts (
    post_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    media_url TEXT NOT NULL,
    media_type VARCHAR(10) NOT NULL, -- 'photo', 'video'
    caption TEXT,
    hashtags TEXT[], -- Array of hashtags
    location VARCHAR(255),
    like_count INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    INDEX idx_user_posts (user_id, created_at),
    INDEX idx_hashtags USING gin(hashtags)
);

-- Likes
CREATE TABLE post_likes (
    user_id UUID NOT NULL,
    post_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, post_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (post_id) REFERENCES posts(post_id)
);

-- Comments
CREATE TABLE post_comments (
    comment_id UUID PRIMARY KEY,
    post_id UUID NOT NULL,
    user_id UUID NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (post_id) REFERENCES posts(post_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    INDEX idx_post_comments (post_id, created_at)
);

-- Timeline Storage (Fan-out on write)
CREATE TABLE user_timelines (
    user_id UUID NOT NULL,
    post_id UUID NOT NULL,
    post_created_at TIMESTAMP NOT NULL,
    timeline_created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, post_created_at, post_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (post_id) REFERENCES posts(post_id)
);
```

### Feed Generation Algorithm

#### **Approach 1: Push Model (Fan-out on Write)**
```python
class FeedGenerationService:
    def __init__(self, user_service, timeline_service, cache):
        self.user_service = user_service
        self.timeline_service = timeline_service
        self.cache = cache
    
    async def generate_feed_on_post_creation(self, post):
        """Push new post to all followers' timelines"""
        # Get all followers of the user
        followers = await self.user_service.get_followers(post.user_id)
        
        # Handle celebrities differently (too many followers)
        if len(followers) > 1000000:  # 1M followers threshold
            # Use pull model for celebrities
            await self.cache.set(f"celebrity_post:{post.user_id}", post, ttl=3600)
            return
        
        # Push to followers' timelines
        timeline_tasks = []
        for follower_id in followers:
            timeline_tasks.append(
                self.timeline_service.add_to_timeline(
                    user_id=follower_id,
                    post_id=post.post_id,
                    post_created_at=post.created_at
                )
            )
        
        # Execute in parallel
        await asyncio.gather(*timeline_tasks)
        
        # Cache hot timelines
        await self.cache_hot_timelines(followers[:1000])  # Cache top 1000 active users
    
    async def get_user_feed(self, user_id, limit=20, offset=0):
        """Get user's personalized feed"""
        # Try cache first
        cache_key = f"feed:{user_id}:{offset}:{limit}"
        cached_feed = await self.cache.get(cache_key)
        if cached_feed:
            return cached_feed
        
        # Get from timeline table
        timeline_posts = await self.timeline_service.get_timeline(
            user_id=user_id,
            limit=limit,
            offset=offset
        )
        
        # Enrich with post details
        feed = []
        for timeline_item in timeline_posts:
            post = await self.get_post_details(timeline_item.post_id)
            feed.append(post)
        
        # Cache the result
        await self.cache.set(cache_key, feed, ttl=300)  # 5 minutes
        return feed
```

#### **Approach 2: Pull Model (Fan-out on Read)**
```python
class PullBasedFeedService:
    def __init__(self, user_service, post_service, cache):
        self.user_service = user_service
        self.post_service = post_service
        self.cache = cache
    
    async def get_user_feed(self, user_id, limit=20):
        """Generate feed on-demand by pulling from following users"""
        # Get users that current user follows
        following_users = await self.user_service.get_following(user_id)
        
        # Get recent posts from each followed user
        all_posts = []
        for following_user_id in following_users:
            recent_posts = await self.post_service.get_recent_posts(
                user_id=following_user_id,
                limit=10
            )
            all_posts.extend(recent_posts)
        
        # Sort by creation time and apply ranking
        sorted_posts = sorted(all_posts, key=lambda p: p.created_at, reverse=True)
        
        # Apply ML-based ranking
        ranked_posts = await self.apply_ml_ranking(user_id, sorted_posts)
        
        return ranked_posts[:limit]
    
    async def apply_ml_ranking(self, user_id, posts):
        """Apply machine learning ranking algorithm"""
        # Simplified ranking based on engagement
        scored_posts = []
        for post in posts:
            score = self.calculate_engagement_score(post)
            scored_posts.append((post, score))
        
        # Sort by score
        scored_posts.sort(key=lambda x: x[1], reverse=True)
        return [post for post, score in scored_posts]
    
    def calculate_engagement_score(self, post):
        """Calculate engagement score for ranking"""
        # Simple engagement formula
        age_hours = (datetime.utcnow() - post.created_at).total_seconds() / 3600
        engagement_rate = (post.like_count + post.comment_count * 2) / max(age_hours, 1)
        
        return engagement_rate
```

#### **Hybrid Approach (Recommended)**
```python
class HybridFeedService:
    def __init__(self, push_service, pull_service, user_service):
        self.push_service = push_service
        self.pull_service = pull_service
        self.user_service = user_service
    
    async def get_user_feed(self, user_id, limit=20):
        """Combine push and pull models"""
        # Get pre-computed timeline (push model)
        push_feed = await self.push_service.get_user_feed(user_id, limit=15)
        
        # Get fresh content from celebrities (pull model)
        celebrity_posts = await self.pull_service.get_celebrity_posts(user_id, limit=5)
        
        # Merge and rank
        combined_feed = push_feed + celebrity_posts
        ranked_feed = await self.rank_posts(user_id, combined_feed)
        
        return ranked_feed[:limit]
    
    async def rank_posts(self, user_id, posts):
        """Apply personalized ranking"""
        # Get user preferences
        user_preferences = await self.user_service.get_preferences(user_id)
        
        # Apply ML model for personalization
        ranked_posts = await self.ml_ranking_service.rank(
            user_id=user_id,
            posts=posts,
            preferences=user_preferences
        )
        
        return ranked_posts
```

## 📈 Scalability Solutions

### **Database Scaling**
```python
class DatabaseSharding:
    def __init__(self):
        self.shards = {
            'users': ['user_shard_1', 'user_shard_2', 'user_shard_3'],
            'posts': ['post_shard_1', 'post_shard_2', 'post_shard_3'],
            'timelines': ['timeline_shard_1', 'timeline_shard_2', 'timeline_shard_3']
        }
    
    def get_user_shard(self, user_id):
        return self.shards['users'][hash(user_id) % len(self.shards['users'])]
    
    def get_post_shard(self, post_id):
        return self.shards['posts'][hash(post_id) % len(self.shards['posts'])]
    
    def get_timeline_shard(self, user_id):
        return self.shards['timelines'][hash(user_id) % len(self.shards['timelines'])]
```

### **Caching Strategy**
```python
class CachingStrategy:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    async def cache_user_feed(self, user_id, feed, ttl=300):
        """Cache user feed for 5 minutes"""
        key = f"feed:{user_id}"
        await self.redis.setex(key, ttl, json.dumps(feed))
    
    async def cache_post_details(self, post_id, post_data, ttl=3600):
        """Cache post details for 1 hour"""
        key = f"post:{post_id}"
        await self.redis.setex(key, ttl, json.dumps(post_data))
    
    async def cache_user_timeline(self, user_id, timeline, ttl=1800):
        """Cache user timeline for 30 minutes"""
        key = f"timeline:{user_id}"
        await self.redis.setex(key, ttl, json.dumps(timeline))
    
    async def invalidate_user_cache(self, user_id):
        """Invalidate user-related caches"""
        keys_to_delete = [
            f"feed:{user_id}",
            f"timeline:{user_id}",
            f"following:{user_id}",
            f"followers:{user_id}"
        ]
        await self.redis.delete(*keys_to_delete)
```

### **Media Storage Architecture**
```python
class MediaStorageService:
    def __init__(self, s3_client, cloudfront_client):
        self.s3 = s3_client
        self.cloudfront = cloudfront_client
    
    async def upload_media(self, user_id, media_file, media_type):
        """Upload media with multiple resolutions"""
        # Generate unique filename
        filename = f"{user_id}/{uuid.uuid4()}.{media_type}"
        
        # Upload original
        original_url = await self.upload_to_s3(filename, media_file)
        
        # Generate thumbnails/resized versions
        if media_type in ['jpg', 'jpeg', 'png']:
            thumbnails = await self.generate_image_thumbnails(media_file)
            thumbnail_urls = {}
            for size, thumbnail in thumbnails.items():
                thumb_filename = f"{filename}_{size}"
                thumbnail_urls[size] = await self.upload_to_s3(thumb_filename, thumbnail)
        
        # Video processing
        elif media_type in ['mp4', 'mov']:
            video_formats = await self.process_video(media_file)
            video_urls = {}
            for format_name, video_data in video_formats.items():
                video_filename = f"{filename}_{format_name}"
                video_urls[format_name] = await self.upload_to_s3(video_filename, video_data)
        
        return {
            'original_url': original_url,
            'thumbnails': thumbnail_urls if media_type in ['jpg', 'jpeg', 'png'] else {},
            'video_formats': video_urls if media_type in ['mp4', 'mov'] else {}
        }
    
    async def get_media_url(self, media_key, size='original'):
        """Get CDN URL for media"""
        return f"https://cdn.instagram.com/{media_key}_{size}"
```

## 🔄 Real-time Features

### **WebSocket for Real-time Updates**
```python
class RealtimeService:
    def __init__(self, websocket_manager, notification_service):
        self.websocket_manager = websocket_manager
        self.notification_service = notification_service
    
    async def handle_post_like(self, post_id, user_id):
        """Handle real-time like notification"""
        # Update like count
        await self.update_like_count(post_id, increment=1)
        
        # Get post owner
        post = await self.get_post(post_id)
        post_owner_id = post.user_id
        
        # Send real-time notification
        if post_owner_id != user_id:  # Don't notify self
            notification = {
                'type': 'like',
                'post_id': post_id,
                'user_id': user_id,
                'message': f'Someone liked your post'
            }
            await self.websocket_manager.send_to_user(post_owner_id, notification)
    
    async def handle_new_comment(self, post_id, user_id, comment_text):
        """Handle real-time comment notification"""
        # Save comment
        comment = await self.save_comment(post_id, user_id, comment_text)
        
        # Get post owner
        post = await self.get_post(post_id)
        post_owner_id = post.user_id
        
        # Send real-time notification
        if post_owner_id != user_id:
            notification = {
                'type': 'comment',
                'post_id': post_id,
                'user_id': user_id,
                'comment_id': comment.comment_id,
                'message': f'Someone commented on your post'
            }
            await self.websocket_manager.send_to_user(post_owner_id, notification)
```

## 🔒 Security Considerations

### **Authentication & Authorization**
```python
class AuthenticationService:
    def __init__(self, jwt_secret):
        self.jwt_secret = jwt_secret
    
    def generate_access_token(self, user_id):
        """Generate JWT access token"""
        payload = {
            'user_id': user_id,
            'exp': datetime.utcnow() + timedelta(hours=1),
            'iat': datetime.utcnow()
        }
        return jwt.encode(payload, self.jwt_secret, algorithm='HS256')
    
    def verify_token(self, token):
        """Verify JWT token"""
        try:
            payload = jwt.decode(token, self.jwt_secret, algorithms=['HS256'])
            return payload['user_id']
        except jwt.ExpiredSignatureError:
            raise AuthenticationError("Token expired")
        except jwt.InvalidTokenError:
            raise AuthenticationError("Invalid token")
```

### **Rate Limiting**
```python
class RateLimitingService:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    async def check_rate_limit(self, user_id, action, limit, window):
        """Check if user exceeded rate limit"""
        key = f"rate_limit:{user_id}:{action}"
        current_count = await self.redis.get(key)
        
        if current_count is None:
            await self.redis.setex(key, window, 1)
            return True
        
        if int(current_count) >= limit:
            return False
        
        await self.redis.incr(key)
        return True
    
    async def apply_rate_limits(self, user_id):
        """Apply various rate limits"""
        limits = {
            'post_creation': (10, 3600),     # 10 posts per hour
            'like_action': (1000, 3600),     # 1000 likes per hour
            'comment_action': (100, 3600),   # 100 comments per hour
            'follow_action': (200, 3600),    # 200 follows per hour
        }
        
        for action, (limit, window) in limits.items():
            if not await self.check_rate_limit(user_id, action, limit, window):
                raise RateLimitExceededError(f"Rate limit exceeded for {action}")
```

## 📊 Monitoring & Metrics

### **Key Performance Indicators**
```python
class MetricsService:
    def __init__(self, metrics_client):
        self.metrics = metrics_client
    
    async def track_feed_performance(self, user_id, latency, cache_hit):
        """Track feed generation performance"""
        self.metrics.timing('feed.generation.latency', latency)
        self.metrics.increment('feed.generation.count')
        
        if cache_hit:
            self.metrics.increment('feed.cache.hit')
        else:
            self.metrics.increment('feed.cache.miss')
    
    async def track_user_engagement(self, user_id, action, post_id):
        """Track user engagement metrics"""
        self.metrics.increment(f'engagement.{action}')
        self.metrics.increment(f'user.{user_id}.{action}')
        self.metrics.increment(f'post.{post_id}.{action}')
```

### **Health Checks**
```python
class HealthCheckService:
    def __init__(self, db_client, redis_client, s3_client):
        self.db = db_client
        self.redis = redis_client
        self.s3 = s3_client
    
    async def check_database_health(self):
        """Check database connectivity"""
        try:
            await self.db.execute("SELECT 1")
            return {'status': 'healthy', 'service': 'database'}
        except Exception as e:
            return {'status': 'unhealthy', 'service': 'database', 'error': str(e)}
    
    async def check_cache_health(self):
        """Check Redis connectivity"""
        try:
            await self.redis.ping()
            return {'status': 'healthy', 'service': 'cache'}
        except Exception as e:
            return {'status': 'unhealthy', 'service': 'cache', 'error': str(e)}
    
    async def check_storage_health(self):
        """Check S3 connectivity"""
        try:
            await self.s3.head_bucket(Bucket='instagram-media')
            return {'status': 'healthy', 'service': 'storage'}
        except Exception as e:
            return {'status': 'unhealthy', 'service': 'storage', 'error': str(e)}
```

## 🎯 Trade-offs Analysis

### **Feed Generation Strategy**
| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| **Push (Fan-out on Write)** | Fast reads, pre-computed | Slow writes, storage overhead | Regular users |
| **Pull (Fan-out on Read)** | Fast writes, fresh content | Slow reads, CPU intensive | Celebrities |
| **Hybrid** | Balanced performance | Complex implementation | Large scale systems |

### **Consistency vs Performance**
| Aspect | Strong Consistency | Eventual Consistency |
|--------|-------------------|---------------------|
| **Feed Updates** | Immediate, accurate | Delayed but scalable |
| **Like Counts** | Real-time | Approximate |
| **Comments** | Instant visibility | Propagation delay |

## 🔗 Related Topics

- [[02-System-Components/Caching/Caching-Strategies|Caching Strategies]]
- [[06-Database-Design/Sharding/Database-Sharding|Database Sharding]]
- [[02-System-Components/Message-Queues/Message-Queue-Systems|Message Queues]]
- [[03-Design-Patterns/Event-Driven-Architecture|Event-Driven Architecture]]
- [[08-Performance/Monitoring/Performance-Monitoring|Performance Monitoring]]

## 📚 Further Reading

- **Instagram Engineering Blog**: Architecture deep dives
- **"Designing Instagram" by Educative**: Comprehensive guide
- **Facebook's TAO Paper**: Social graph data store
- **Twitter's Timeline Architecture**: Alternative approaches

---

**Key Takeaways**: 
- Instagram's scale requires hybrid feed generation approaches
- Heavy caching at multiple levels is essential for performance
- Database sharding is crucial for handling write-heavy workloads
- Real-time features add complexity but enhance user experience
- Monitoring and rate limiting are essential for reliability 