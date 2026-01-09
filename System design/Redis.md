# Redis List & ZSET Cheat Sheet

## 1. Redis Lists

### LPUSH (push to head)
```redis
LPUSH key value
LPUSH key v1 v2 v3
```
- Inserts at left (head)
- Newest first, O(1)

**Read latest N:** `LRANGE key 0 N-1`  
**Keep size bounded:** `LTRIM key 0 999`  
**Use:** Feeds (latest first), Stack (LIFO)

### RPUSH (push to tail)
```redis
RPUSH key value
RPUSH key v1 v2 v3
```
- Inserts at right (tail)
- Oldest first, O(1)

**Read last N:** `LRANGE key -N -1`  
**Use:** Queue (FIFO) → `RPUSH queue job` + `LPOP queue`

### POP commands
```redis
LPOP key              # pop from head
RPOP key              # pop from tail
BLPOP key timeout     # blocking pop (head)
BRPOP key timeout     # blocking pop (tail)
```

---

## 2. Redis Sorted Sets (ZSET)

### ZADD
```redis
ZADD key score member
ZADD key s1 m1 s2 m2
```
- Auto-sorted by score, O(log N)

### Read commands
```redis
ZRANGE key start stop [WITHSCORES]      # Lowest → Highest
ZREVRANGE key start stop [WITHSCORES]   # Highest → Lowest
ZREVRANGE key 0 N-1                     # Top N
```

### Range by score
```redis
ZRANGEBYSCORE key min max
ZRANGEBYSCORE key min max LIMIT offset count
```

### Update / increment
```redis
ZINCRBY key increment member
```

### Remove
```redis
ZREM key member
ZREMRANGEBYSCORE key min max
ZREMRANGEBYRANK key start stop
```

