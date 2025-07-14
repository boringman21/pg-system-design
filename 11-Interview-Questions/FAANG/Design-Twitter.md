# Design Twitter - Interview Question

**Tags**: #interview #hard #social-media #twitter
**Date**: 2024-01-01
**Company**: Meta, Google, Amazon
**Duration**: 45-60 minutes

## 📝 Question

"Design a simplified version of Twitter. Users can post tweets, follow other users, and see a timeline of tweets from people they follow. The system should handle 300M active users with 200M tweets per day."

## 🎯 Clarifying Questions

Ask these questions to understand requirements better:

1. What is the scale of the system?
   - How many users? (300M daily active)
   - How many tweets per day? (200M tweets)
   - How many follows per user? (Average 200)

2. What are the core features?
   - Post tweets (280 characters)
   - Follow/unfollow users
   - View timeline (home feed)
   - View user profile
   - Search tweets?
   - Retweets, likes, comments?

3. What are the constraints?
   - Read vs write ratio? (100:1 - more reads)
   - Timeline generation - push vs pull?
   - Real-time updates needed?
   - Media support (images, videos)?

## 🔍 Requirements Analysis

### Functional Requirements
- [ ] **Post Tweet**: Users can post 280-character tweets
- [ ] **Follow Users**: Users can follow/unfollow other users
- [ ] **Timeline**: View chronological timeline of followed users
- [ ] **User Profile**: View user's tweets and info
- [ ] **Search**: Search tweets by keywords

### Non-functional Requirements
- [ ] **Scalability**: 300M DAU, 200M tweets/day
- [ ] **Availability**: 99.9% uptime
- [ ] **Consistency**: Eventually consistent timeline OK
- [ ] **Performance**: Timeline load < 200ms

### Assumptions
- Average user follows 200 people
- Average user posts 0.67 tweets/day
- Read:Write ratio = 100:1
- Peak traffic 3x average

## 📐 Capacity Estimation

### Traffic Estimates
- **Daily Active Users**: 300M
- **Tweets per day**: 200M
- **Tweets per second**: 200M / (24 * 3600) = ~2.3K TPS
- **Peak TPS**: 2.3K * 3 = ~7K TPS
- **Timeline reads**: 2.3K * 100 = 230K reads/sec

### Storage Estimates
- **Tweet size**: 280 chars * 2 bytes + metadata = ~1KB
- **Daily storage**: 200M * 1KB = 200GB/day
- **5-year storage**: 200GB * 365 * 5 = ~365TB

### Bandwidth Estimates
- **Write bandwidth**: 7K TPS * 1KB = 7MB/sec
- **Read bandwidth**: 230K * 1KB = 230MB/sec

## 🏗️ System Design Approach

### Step 1: High-Level Design
```
[Client Apps] → [Load Balancer] → [API Gateway]
                                       ↓
[Tweet Service] ← [Timeline Service] ← [User Service]
       ↓               ↓                   ↓
[Tweet DB]    [Timeline Cache]      [User DB]
                     ↓
              [Message Queue]
```

### Step 2: Core Components
1. **User Service**: User registration, profiles, follow relationships
2. **Tweet Service**: Create, store, retrieve tweets
3. **Timeline Service**: Generate user timelines
4. **Search Service**: Search tweets by keywords
5. **Notification Service**: Real-time notifications
6. **Media Service**: Handle images/videos

### Step 3: API Design
```python
# Core APIs
POST /api/v1/tweets
{
    "content": "Hello Twitter!",
    "user_id": "user123"
}

GET /api/v1/users/{user_id}/timeline?limit=20&cursor=abc123

POST /api/v1/users/{user_id}/follow
{
    "target_user_id": "user456"
}

GET /api/v1/search/tweets?q=keyword&limit=20
```

### Step 4: Database Design
```sql
-- Users table
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    email VARCHAR(100),
    display_name VARCHAR(100),
    bio TEXT,
    followers_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    created_at TIMESTAMP
);

-- Tweets table
CREATE TABLE tweets (
    tweet_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    content VARCHAR(280),
    created_at TIMESTAMP,
    likes_count INT DEFAULT 0,
    retweets_count INT DEFAULT 0,
    INDEX(user_id, created_at),
    INDEX(created_at)
);

-- Follows table  
CREATE TABLE follows (
    follower_id BIGINT,
    followee_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    INDEX(follower_id),
    INDEX(followee_id)
);
```

## 📈 Detailed Design

### Timeline Generation Strategies

#### **Pull Model (Fan-out on Read)**
```python
def get_timeline(user_id, limit=20):
    # Get list of users that this user follows
    following = get_following_users(user_id)
    
    # Fetch recent tweets from all followed users
    tweets = []
    for followed_user in following:
        user_tweets = get_user_tweets(followed_user, limit)
        tweets.extend(user_tweets)
    
    # Sort by timestamp and return top N
    tweets.sort(key=lambda x: x.created_at, reverse=True)
    return tweets[:limit]
```

**Pros**: No storage overhead, good for celebrities
**Cons**: Slow timeline generation, high read latency

#### **Push Model (Fan-out on Write)**
```python
def post_tweet(user_id, content):
    # Create tweet
    tweet = create_tweet(user_id, content)
    
    # Get all followers
    followers = get_followers(user_id)
    
    # Push tweet to all followers' timelines
    for follower in followers:
        add_to_timeline(follower, tweet)
    
    return tweet

def get_timeline(user_id, limit=20):
    # Simply read from pre-computed timeline
    return get_precomputed_timeline(user_id, limit)
```

**Pros**: Fast timeline reads
**Cons**: High write amplification, storage overhead

#### **Hybrid Model**
```python
def post_tweet(user_id, content):
    tweet = create_tweet(user_id, content)
    
    # Check if user is celebrity (>1M followers)
    if is_celebrity(user_id):
        # Don't fan-out for celebrities
        return tweet
    else:
        # Fan-out to followers' timelines
        followers = get_followers(user_id)
        for follower in followers:
            add_to_timeline(follower, tweet)
    
    return tweet

def get_timeline(user_id, limit=20):
    # Get pre-computed timeline
    timeline = get_precomputed_timeline(user_id)
    
    # Merge with celebrity tweets on-the-fly
    celebrity_tweets = get_celebrity_tweets(user_id)
    
    # Merge and sort
    merged = merge_timelines(timeline, celebrity_tweets)
    return merged[:limit]
```

### Caching Strategy
```python
class TimelineCache:
    def __init__(self):
        self.redis = Redis()
    
    def get_timeline(self, user_id):
        cache_key = f"timeline:{user_id}"
        timeline = self.redis.get(cache_key)
        
        if timeline:
            return json.loads(timeline)
        return None
    
    def update_timeline(self, user_id, tweets):
        cache_key = f"timeline:{user_id}"
        # Cache top 1000 tweets, TTL 1 hour
        self.redis.setex(cache_key, 3600, json.dumps(tweets[:1000]))
    
    def invalidate_follower_timelines(self, user_id):
        # When user posts, invalidate followers' cached timelines
        followers = get_followers(user_id)
        for follower in followers:
            self.redis.delete(f"timeline:{follower}")
```

## ⚖️ Scale the Design

### Identify Bottlenecks
- **Timeline generation**: Expensive for users with many follows
- **Database writes**: 7K tweets/sec write load
- **Hot users**: Celebrities with millions of followers
- **Storage**: 365TB of tweet data

### Scaling Solutions

#### **Database Sharding**
```python
# Shard tweets by tweet_id
def get_tweet_shard(tweet_id):
    return tweet_id % NUM_SHARDS

# Shard users by user_id
def get_user_shard(user_id):
    return user_id % NUM_SHARDS

# Timeline can query multiple shards
def get_timeline(user_id):
    following = get_following_users(user_id)
    
    # Group by shard to minimize queries
    shard_users = defaultdict(list)
    for user in following:
        shard = get_user_shard(user)
        shard_users[shard].append(user)
    
    # Parallel queries to shards
    timeline_pieces = []
    for shard, users in shard_users.items():
        pieces = query_shard(shard, users)
        timeline_pieces.extend(pieces)
    
    return merge_and_sort(timeline_pieces)
```

#### **Read Replicas**
```python
# Separate read and write databases
class TwitterDB:
    def __init__(self):
        self.write_db = connect_to_master()
        self.read_dbs = [connect_to_replica(i) for i in range(5)]
    
    def write_tweet(self, tweet):
        return self.write_db.insert_tweet(tweet)
    
    def read_tweets(self, query):
        # Load balance across read replicas
        replica = random.choice(self.read_dbs)
        return replica.query_tweets(query)
```

#### **CDN for Media**
```python
# Store tweet media in CDN
def upload_media(file):
    # Upload to S3
    s3_url = s3.upload(file)
    
    # Return CDN URL
    cdn_url = f"https://cdn.twitter.com/{s3_url}"
    return cdn_url

def get_tweet_with_media(tweet_id):
    tweet = get_tweet(tweet_id)
    if tweet.media_url:
        # CDN will serve media globally
        tweet.media_url = transform_to_cdn_url(tweet.media_url)
    return tweet
```

## 🔒 Additional Considerations

### Security
- Rate limiting (300 tweets/hour per user)
- Content moderation and spam detection
- API authentication with OAuth
- Input validation and sanitization

### Monitoring
- Tweet creation rate and errors
- Timeline generation latency
- Cache hit ratios
- Database query performance

### Reliability
- Multi-region deployment
- Circuit breakers for service calls
- Dead letter queues for failed operations
- Backup and disaster recovery

## 🎯 Sample Solution

### Approach
1. Use hybrid fan-out model
2. Cache timelines in Redis
3. Shard databases by user_id
4. Use read replicas for timeline queries
5. CDN for media content

### Architecture Highlights
- **API Gateway**: Route requests, authentication, rate limiting
- **Microservices**: User, Tweet, Timeline, Search services
- **Event-driven**: Async processing for notifications
- **Caching**: Redis for timelines, memcached for user data

### Key Decisions
| Decision | Reasoning |
|----------|-----------|
| Hybrid fan-out | Balance between read/write performance |
| Sharding by user_id | Distribute user data and follows |
| Redis for timeline cache | Fast read access for hot data |
| Event-driven architecture | Decouple services, handle scale |

## 💡 Alternative Approaches

### Approach 1: Full Push Model
**Pros**: Extremely fast reads, simple timeline API
**Cons**: High storage cost, celebrity problem

### Approach 2: Full Pull Model  
**Pros**: No storage overhead, handles celebrities well
**Cons**: Slow timeline generation, complex caching

## 🚨 Common Mistakes

- ❌ Not considering the celebrity problem
- ❌ Underestimating storage requirements
- ❌ Ignoring cache invalidation complexity
- ❌ Not planning for read/write traffic patterns

## 📚 Related Questions

- [[Design Instagram Feed]]
- [[Design Facebook News Feed]]
- [[Design LinkedIn Feed]]

## 🎓 Key Takeaways

- Fan-out strategy is crucial for social media systems
- Hybrid approach balances read/write performance
- Caching is essential for timeline performance
- Database sharding needed for horizontal scaling
- Consider celebrity users as special case

---

**Practice Notes**: 
- Focus on timeline generation as core challenge
- Discuss trade-offs between push/pull models
- Consider consistency vs performance trade-offs

**Time to Complete**: 45-50 minutes
**Areas to Improve**: Database sharding strategy, caching invalidation 
