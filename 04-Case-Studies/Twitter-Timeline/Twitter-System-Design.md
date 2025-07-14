# Twitter System Design

**Tags**: #case-study #social-media #timeline #scale
**Ngày tạo**: 2024-01-01
**Độ khó**: Hard

## 🎯 Problem Statement

Design Twitter with 300M users, 500M tweets/day, real-time timeline generation.

### Requirements
- Post tweets (280 chars)
- Follow users
- Home & user timeline
- Search tweets
- Trending topics

## 📊 Scale Estimates

```python
daily_active_users = 300_000_000
tweets_per_day = 500_000_000
timeline_reads_per_day = 15_000_000_000

# Peak traffic
peak_tweets_per_second = 29_000
peak_timeline_reads_per_second = 520_000

# Storage
daily_tweet_storage = 240_GB  # 500M * 480B
yearly_tweet_storage = 87_TB
```

## 🏗️ Architecture

### Core Services
- **Tweet Service**: Create, store tweets
- **Timeline Service**: Generate user feeds
- **User Service**: User profiles, follow relationships
- **Search Service**: Tweet search, trending
- **Notification Service**: Real-time updates

### Database Design
```sql
-- Tweets (sharded by tweet_id)
CREATE TABLE tweets (
    tweet_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    content TEXT,
    created_at TIMESTAMP,
    like_count INT DEFAULT 0
);

-- Follows (sharded by follower_id)
CREATE TABLE follows (
    follower_id BIGINT,
    following_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (follower_id, following_id)
);
```

## 📱 Timeline Generation

### Fan-out Strategies

#### Push Model (Pre-compute)
```python
def handle_new_tweet(tweet):
    followers = get_followers(tweet.user_id)
    
    for follower_id in followers:
        add_to_timeline_cache(follower_id, tweet)
```

#### Pull Model (On-demand)
```python
def get_timeline(user_id):
    following = get_following(user_id)
    tweets = []
    
    for followed_user in following:
        user_tweets = get_recent_tweets(followed_user)
        tweets.extend(user_tweets)
    
    return sort_by_timestamp(tweets)
```

#### Hybrid Model
```python
def get_timeline(user_id):
    # Pre-computed for regular users
    timeline = get_precomputed_timeline(user_id)
    
    # On-demand for celebrities
    celebrity_tweets = get_celebrity_tweets(user_id)
    
    return merge_and_sort(timeline, celebrity_tweets)
```

## 🔍 Search & Trending

### Tweet Search
```python
class SearchService:
    def search_tweets(self, query):
        es_query = {
            'query': {
                'multi_match': {
                    'query': query,
                    'fields': ['content', 'hashtags', 'username']
                }
            },
            'sort': [{'created_at': {'order': 'desc'}}]
        }
        
        return elasticsearch.search(es_query)
```

### Trending Topics
```python
def calculate_trending():
    current_counts = get_hashtag_counts(last_hour)
    historical_avg = get_historical_average()
    
    trending = []
    for hashtag, count in current_counts:
        velocity = count / historical_avg[hashtag]
        if velocity > 2.0:  # Trending threshold
            trending.append((hashtag, velocity))
    
    return sort_by_velocity(trending)
```

## ⚡ Performance Optimizations

### Caching Strategy
- **Timeline Cache**: 30 minutes TTL
- **Tweet Cache**: 1 hour TTL  
- **User Profile**: 2 hours TTL
- **Trending Topics**: 5 minutes TTL

### Sharding
- **Tweets**: Shard by tweet_id
- **Users**: Shard by user_id
- **Timelines**: Co-locate with user data

### Rate Limiting
- Tweet creation: 300/15min
- Timeline reads: 1500/15min
- Search: 300/15min

## 🎯 Key Decisions

1. **Eventual Consistency**: Acceptable for social media
2. **Hybrid Fan-out**: Optimize for different user types
3. **Denormalized Storage**: Faster reads over write complexity
4. **Aggressive Caching**: Handle massive read traffic

**Twitter success = Right trade-offs for social media use case!** 