
# Design Twitter / News Feed - Complete Guide

## 1. Requirements

### Functional
- Post tweets (text, images)
- Follow/unfollow users
- View home timeline (tweets from followed users)
- View user profile timeline

### Non-Functional
- Read-heavy (100:1 read/write ratio)
- Low latency for feed (< 200ms)
- High availability
- Eventually consistent is OK

### Scale
- 500M users, 200M daily active
- 500M tweets/day (~6K tweets/sec)
- User follows avg 200 people

---

## 2. High-Level Architecture

```
┌─────────┐     ┌─────────────┐     ┌─────────────┐
│  User   │────▶│ Load Balancer│────▶│ API Gateway │
└─────────┘     └─────────────┘     └─────────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    ▼                      ▼                      ▼
             ┌───────────┐          ┌───────────┐          ┌───────────┐
             │ Tweet     │          │ Feed      │          │ User      │
             │ Service   │          │ Service   │          │ Service   │
             └─────┬─────┘          └─────┬─────┘          └───────────┘
                   │                      │
                   ▼                      ▼
             ┌───────────┐          ┌───────────┐
             │ Tweet DB  │          │ Feed Cache│
             └───────────┘          │ (Redis)   │
                                    └───────────┘
```

---

## 3. Data Model

### Users Table
```
user_id | name | email | followers_count | created_at
```

### Tweets Table
```
tweet_id | user_id | content | media_url | created_at
```

### Follows Table (One-Way Relationship)
```
follower_id | followee_id | created_at
```

**Example:**
```
┌─────────────┬─────────────┐
│ follower_id │ followee_id │
├─────────────┼─────────────┤
│ Alice       │ Bob         │  ← Alice follows Bob
│ Alice       │ Charlie     │  ← Alice follows Charlie
│ Bob         │ Charlie     │  ← Bob follows Charlie
└─────────────┴─────────────┘
```

**Queries:**
```sql
-- Get who I follow
SELECT followee_id FROM follows WHERE follower_id = 'Alice';

-- Get my followers
SELECT follower_id FROM follows WHERE followee_id = 'Charlie';
```

---

## 4. The Core Problem: Fan-Out

When User A posts a tweet, how do their followers see it?

### Approach 1: Fan-Out on Write (Push Model)

When user posts, immediately push tweet to all followers' cached timelines.

**Pros:** Fast reads (timeline pre-computed)
**Cons:** Slow writes for celebrities (millions of followers)

### Approach 2: Fan-Out on Read (Pull Model)

When user opens app, fetch tweets from all followed users in real-time.

**Pros:** Simple writes
**Cons:** Slow reads (must query 200+ users)

### Approach 3: Hybrid (Twitter's Actual Approach) ✅

```
Regular users (< 10K followers)  →  Fan-out on Write
Celebrities (> 10K followers)    →  Fan-out on Read
```

---

## 5. Timeline Cache Structure (Redis)

### Pre-Computed Timeline (Regular Users)

```
Key: timeline:{user_id}
Type: List
Value: [tweet_123, tweet_119, tweet_115, ...]  (newest → oldest)
Limit: 800 tweet IDs per user
```

### Celebrity Tweets (Sorted Set)

```
Key: tweets:{celebrity_id}
Type: Sorted Set
Score: Timestamp (milliseconds)
Value: tweet_id
```

**Example:**
```
Key: tweets:elonmusk

┌───────────────┬─────────────────┐
│ Score (time)  │ Value (tweet_id)│
├───────────────┼─────────────────┤
│ 1704067200000 │ tweet_501       │  ← oldest
│ 1704070800000 │ tweet_502       │
│ 1704074400000 │ tweet_503       │
│ 1704078000000 │ tweet_504       │  ← newest
└───────────────┴─────────────────┘
```

---

## 6. Fan-Out on Write Flow

When a regular user posts:

```
Bob (5K followers) posts "Hello World" at 10:00 AM
                │
                ▼
┌─────────────────────────────────────────────┐
│ Fan-Out Worker:                             │
│                                             │
│ For each follower of Bob:                   │
│   LPUSH timeline:{follower_id} tweet_123    │
│   LTRIM timeline:{follower_id} 0 799        │
└─────────────────────────────────────────────┘
```

**Architecture:**
```
┌───────────┐     ┌─────────────┐     ┌─────────────┐
│ Tweet     │────▶│ Message     │────▶│ Fan-Out     │
│ Created   │     │ Queue       │     │ Workers     │
└───────────┘     └─────────────┘     └─────────────┘
                                            │
                                            ▼
                                    ┌─────────────┐
                                    │ Update each │
                                    │ follower's  │
                                    │ cache       │
                                    └─────────────┘
```

---

## 7. Timeline Read Flow (Detailed)

```
User Alice opens app
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ 1. FETCH PRE-COMPUTED TIMELINE                          │
│    Redis: LRANGE timeline:alice 0 99                    │
│    Result: [tweet_123, tweet_119, tweet_115, ...]       │
│            (from regular users she follows)             │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ 2. GET LIST OF CELEBRITIES ALICE FOLLOWS                │
│    Query: SELECT followee_id FROM follows               │
│           WHERE follower_id = 'alice'                   │
│           AND followee_id IN (celebrity_list)           │
│    Result: [@elonmusk, @taylorswift]                    │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ 3. FETCH CELEBRITY TWEETS (Parallel)                    │
│    Redis: ZRANGEBYSCORE tweets:elonmusk (now-24h) now   │
│    Redis: ZRANGEBYSCORE tweets:taylorswift (now-24h) now│
│    Result: [celeb_tweet_501, celeb_tweet_499, ...]      │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ 4. MERGE & SORT BY TIMESTAMP                            │
│    Combined = regular_tweets + celebrity_tweets         │
│    Sort by created_at DESC                              │
│    Return top 20 for first page                         │
└─────────────────────────────────────────────────────────┘
```

---

## 8. ZRANGEBYSCORE Explained

**Command:**
```javascript
redis.zrangebyscore(`tweets:${celeb}`, Date.now() - 86400000, Date.now())
```

| Part | Meaning |
|------|---------|
| `zrangebyscore` | Get values where score is between min and max |
| `tweets:${celeb}` | The key (e.g., `tweets:elonmusk`) |
| `Date.now() - 86400000` | Min score = now minus 24 hours |
| `Date.now()` | Max score = now |

**86400000 = 24 hours in milliseconds** (24 × 60 × 60 × 1000)

**Visual:**
```
Timeline (24 hours):
─────────────────────────────────────────────────────►
│                                                    │
24h ago                                            Now
│                                                    │
│    [tweet_501]  [tweet_502]  [tweet_503]  [tweet_504]
│                                                    │
└──────────── ZRANGEBYSCORE returns these ───────────┘
```

---

## 9. Merge Algorithm (K-Way Merge)

**Input:**
```
Pre-computed timeline (already sorted):
[tweet_123 (10:05), tweet_119 (10:02), tweet_115 (9:58), ...]

Celebrity tweets (already sorted per celebrity):
Elon:   [tweet_501 (10:03), tweet_498 (9:55)]
Taylor: [tweet_601 (10:01), tweet_599 (9:50)]
```

**Using Max-Heap:**
```
Heap (max-heap by timestamp):
┌────────────────────────────────────────┐
│ (10:05, tweet_123, precomputed, idx=0) │ ← top
│ (10:03, tweet_501, elon, idx=0)        │
│ (10:02, tweet_119, precomputed, idx=1) │
│ (10:01, tweet_601, taylor, idx=0)      │
└────────────────────────────────────────┘

Pop top → tweet_123 (10:05) → Add to result
Pop top → tweet_501 (10:03) → Add to result
...continue until we have 20 tweets
```

**Pseudocode:**
```javascript
function getTimeline(userId, limit = 20) {
  // Step 1: Get pre-computed timeline
  const precomputedTweetIds = redis.lrange(`timeline:${userId}`, 0, 99);
  const precomputedTweets = fetchTweetsByIds(precomputedTweetIds);
  
  // Step 2: Get celebrities user follows
  const followedCelebrities = getFollowedCelebrities(userId);
  
  // Step 3: Fetch celebrity tweets (parallel)
  const celebrityTweets = await Promise.all(
    followedCelebrities.map(celeb => 
      redis.zrangebyscore(`tweets:${celeb}`, Date.now() - 86400000, Date.now())
    )
  );
  
  // Step 4: Merge using heap
  const merged = mergeKSortedLists([
    precomputedTweets,
    ...celebrityTweets
  ]);
  
  return merged.slice(0, limit);
}

function mergeKSortedLists(lists) {
  const heap = new MaxHeap(); // sorted by tweet.created_at
  
  // Initialize heap with first element from each list
  for (let i = 0; i < lists.length; i++) {
    if (lists[i].length > 0) {
      heap.push({ tweet: lists[i][0], listIdx: i, elementIdx: 0 });
    }
  }
  
  const result = [];
  
  while (!heap.isEmpty() && result.length < 100) {
    const { tweet, listIdx, elementIdx } = heap.pop();
    result.push(tweet);
    
    // Add next element from same list
    if (elementIdx + 1 < lists[listIdx].length) {
      heap.push({
        tweet: lists[listIdx][elementIdx + 1],
        listIdx,
        elementIdx: elementIdx + 1
      });
    }
  }
  
  return result;
}
```

**Merged Result:**
```
┌─────────────────────────┐
│ 10:05 - Bob: "Hi"       │
│ 10:03 - Elon: "..."     │
│ 10:02 - Carol: "..."    │
│ 10:01 - Taylor: "..."   │
│ 9:58  - Dave: "..."     │
│ 9:55  - Elon: "..."     │
│ 9:50  - Taylor: "..."   │
│ 9:45  - Eve: "..."      │
└─────────────────────────┘
```

---

## 10. Database Choices

| Data | Storage | Why |
|------|---------|-----|
| Tweets | MySQL/PostgreSQL + Sharding | Structured, need ACID for writes |
| Timeline Cache | Redis | Fast reads, sorted sets |
| Media | S3 + CDN | Blob storage, global distribution |
| User Graph | Graph DB or MySQL | Follow relationships |

---

## 11. Scaling Strategies

| Challenge | Solution |
|-----------|----------|
| Read-heavy | Cache timelines in Redis |
| Celebrity problem | Hybrid fan-out |
| Hot partitions | Shard by user_id |
| Global users | Multi-region, CDN |
| Spike handling | Message queue for async fan-out |

---

## 12. Performance Summary

| Step | Time |
|------|------|
| Fetch pre-computed (Redis) | ~5ms |
| Get celebrity list (cached) | ~2ms |
| Fetch celebrity tweets (parallel) | ~20ms |
| Merge (in-memory heap) | ~1ms |
| **Total** | **~30ms** ✅ |

---

## 13. Interview Tips

- **Clarify:** Read/write ratio, celebrities, real-time vs eventual consistency
- **Trade-offs:** Push vs Pull, Cache size limits
- **Numbers:** 500M tweets/day = ~6K tweets/sec
- **Deep dives:** Fan-out, ranking algorithm, cache invalidation

