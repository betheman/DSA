Rate Limiter Algorithms – Guide
1️⃣ Fixed Window Rate Limiter

Concept:

Counts requests in fixed time windows (e.g., 1 min)

Resets counter at the start of each window

Example:

Limit: 5 requests/min

Window: 0–60s, 61–120s, …

Time	Request Count	Allowed?
10s	1	✅
20s	2	✅
59s	5	✅
61s	1 (new window)	✅

Redis Example (Node.js):

const key = `rate_limit:${userId}:${Math.floor(Date.now()/60000)}`;
const count = await redis.incr(key);
if (count === 1) await redis.expire(key, 60); // set TTL for window
return count <= 5;


Pros:

Simple, easy to implement

Cons:

Boundary spikes: user can send 5 requests at the end of one window + 5 at the start of next → 10 in ~1s

Use case:

Low-traffic APIs, non-critical limits

2️⃣ Sliding Window Rate Limiter

Concept:

Tracks all requests within last N seconds

Only requests in the current sliding window count

Example:

Limit: 5 requests / 60s

Requests at t = 0, 10, 20, 40, 50 → 6th request at t=55 → blocked

Sliding window moves continuously, no spikes at boundaries

Redis + Node.js Example (ZSET):

const now = Date.now();
const windowSize = 60000; // 60s
const limit = 5;
const key = `rate_limit:${userId}`;

// Remove old requests
await redis.zremrangebyscore(key, 0, now - windowSize);

// Count current requests
const count = await redis.zcard(key);

if (count < limit) {
    await redis.zadd(key, now, now);
    await redis.pexpire(key, windowSize);
    return true;
} else {
    return false;
}


Pros:

Accurate, prevents spikes at window boundaries

Works well for strict per-user limits

Cons:

Stores timestamps → higher memory usage

More Redis commands → higher latency

Use case:

APIs with strict rate limits, per-user or per-IP

3️⃣ Token Bucket Rate Limiter

Concept:

Bucket has capacity tokens

Tokens refill at a fixed rate

Each request consumes a token

If bucket empty → reject request

Allows bursts while maintaining average rate

Example:

Capacity = 5, Refill = 1 token/sec

Bucket starts full: 5 requests allowed immediately

Further requests wait for refill

Redis + Node.js + Lua (25k tokens/sec example):

-- Lua Script (atomic)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local cost = tonumber(ARGV[4])

local data = redis.call("HMGET", key, "tokens", "last_refill_ts")
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

local elapsed_ms = now - last_refill
if elapsed_ms >= 1000 then
    local refill = math.floor(elapsed_ms / 1000) * refill_rate
    tokens = math.min(capacity, tokens + refill)
    last_refill = now
end

local allowed = 0
if tokens >= cost then
    tokens = tokens - cost
    allowed = 1
end

redis.call("HMSET", key, "tokens", tokens, "last_refill_ts", last_refill)
redis.call("EXPIRE", key, 60)
return allowed


Node.js Call:

const allowed = await redis.eval(LUA_SCRIPT, 1, key, 25000, 25000, Date.now(), 1);


Pros:

Allows smooth bursts

Good for high-traffic APIs

Atomic with Lua

Cons:

Slightly approximate (fractional refill can allow minor bursts)

Harder to explain than Fixed Window

Use case:

API gateways, high-scale systems, traffic smoothing

Comparison Table
Feature	Fixed Window	Sliding Window	Token Bucket
Accuracy	Low	High	Medium
Burst Handling	❌ No	Limited	✅ Yes
Memory	Low	High (timestamps)	Very Low
Complexity	Low	Medium	Medium
Use Case	Simple limits	Strict per-user/IP	Smooth bursts / high RPS
Boundary Spikes	Yes	No	No (smooth)
