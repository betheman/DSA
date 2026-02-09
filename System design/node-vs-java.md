# Node.js vs Java: Performance Comparison

---

# How Event Loop Works in Node.js

## Simple Analogy: Restaurant with 1 Waiter

```
┌─────────────────────────────────────────────────────────────────┐
│                     RESTAURANT (Node.js)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   👨‍🍳 Kitchen (Thread Pool)     👨‍💼 Waiter (Event Loop)           │
│   ┌─────────────────────┐       ┌─────────────────────┐        │
│   │ Cook 1: Making soup │       │ Takes orders        │        │
│   │ Cook 2: Grilling    │       │ Serves ready food   │        │
│   │ Cook 3: Baking      │       │ Never waits idle    │        │
│   │ Cook 4: Salad       │       │ Always moving       │        │
│   └─────────────────────┘       └─────────────────────┘        │
│                                                                 │
│   🪑 Tables (Requests)                                          │
│   Table 1: Waiting for soup                                     │
│   Table 2: Waiting for steak                                    │
│   Table 3: Just ordered                                         │
│   Table 4: Ready to pay                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Waiter NEVER stands and waits for food.
Takes order → Goes to next table → Serves when ready → Repeat
```

---

## Event Loop Phases

```
   ┌───────────────────────────────┐
┌─▶│           timers              │  ← setTimeout, setInterval
│  └─────────────┬─────────────────┘
│  ┌─────────────▼─────────────────┐
│  │     pending callbacks         │  ← I/O callbacks deferred
│  └─────────────┬─────────────────┘
│  ┌─────────────▼─────────────────┐
│  │       idle, prepare           │  ← Internal use only
│  └─────────────┬─────────────────┘
│  ┌─────────────▼─────────────────┐
│  │           poll                │  ← Retrieve new I/O events
│  └─────────────┬─────────────────┘     (most time spent here)
│  ┌─────────────▼─────────────────┐
│  │           check               │  ← setImmediate callbacks
│  └─────────────┬─────────────────┘
│  ┌─────────────▼─────────────────┐
│  │      close callbacks          │  ← socket.on('close')
│  └─────────────┬─────────────────┘
│                │
└────────────────┘
        ↑
    NEXT TICK & MICROTASKS
    (process.nextTick, Promises)
    Run between EVERY phase!
```

---

## Phase by Phase Explanation

### Complete Code Example: All Phases

```javascript
const fs = require('fs');
const net = require('net');

console.log('=== EVENT LOOP PHASES DEMO ===\n');

// ══════════════════════════════════════════════════════════
// PHASE 1: TIMERS
// Callbacks from: setTimeout(), setInterval()
// ══════════════════════════════════════════════════════════
setTimeout(() => {
    console.log('1. TIMERS PHASE: setTimeout callback');
}, 0);

setInterval(() => {
    console.log('1. TIMERS PHASE: setInterval callback (runs once)');
}, 0);


// ══════════════════════════════════════════════════════════
// PHASE 2: PENDING CALLBACKS (I/O Callbacks)
// Callbacks from: Some system operations (TCP errors, etc.)
// ══════════════════════════════════════════════════════════
// TCP connection error callback runs here
const socket = net.connect(9999, 'localhost', () => {});
socket.on('error', () => {
    console.log('2. PENDING CALLBACKS: TCP error callback');
});


// ══════════════════════════════════════════════════════════
// PHASE 3: IDLE, PREPARE
// Internal use only - no user callbacks
// ══════════════════════════════════════════════════════════
// (No user code runs here)


// ══════════════════════════════════════════════════════════
// PHASE 4: POLL
// Callbacks from: fs, http, database, most I/O operations
// ══════════════════════════════════════════════════════════
fs.readFile(__filename, () => {
    console.log('4. POLL PHASE: fs.readFile callback');
});

// HTTP server callbacks run in Poll phase
// const server = http.createServer((req, res) => {
//     console.log('4. POLL PHASE: HTTP request callback');
// });


// ══════════════════════════════════════════════════════════
// PHASE 5: CHECK
// Callbacks from: setImmediate()
// ══════════════════════════════════════════════════════════
setImmediate(() => {
    console.log('5. CHECK PHASE: setImmediate callback');
});


// ══════════════════════════════════════════════════════════
// PHASE 6: CLOSE CALLBACKS
// Callbacks from: socket.on('close'), server.on('close')
// ══════════════════════════════════════════════════════════
const tempSocket = net.connect(80, 'google.com');
tempSocket.on('close', () => {
    console.log('6. CLOSE CALLBACKS: socket close callback');
});
tempSocket.end();


// ══════════════════════════════════════════════════════════
// MICROTASKS (Run between EVERY phase)
// Callbacks from: process.nextTick(), Promise.then()
// ══════════════════════════════════════════════════════════
process.nextTick(() => {
    console.log('>> MICROTASK: process.nextTick (highest priority)');
});

Promise.resolve().then(() => {
    console.log('>> MICROTASK: Promise.then (after nextTick)');
});


console.log('\n=== SYNCHRONOUS CODE END ===\n');
```

**Expected Output Order:**
```
=== EVENT LOOP PHASES DEMO ===

=== SYNCHRONOUS CODE END ===

>> MICROTASK: process.nextTick (highest priority)
>> MICROTASK: Promise.then (after nextTick)
1. TIMERS PHASE: setTimeout callback
1. TIMERS PHASE: setInterval callback (runs once)
5. CHECK PHASE: setImmediate callback
4. POLL PHASE: fs.readFile callback
2. PENDING CALLBACKS: TCP error callback
6. CLOSE CALLBACKS: socket close callback
```

---

### Summary Table: Which Callback Runs Where?

| Order | Phase | Functions | Example |
|-------|-------|-----------|---------|
| 1 | **Timers** | `setTimeout()`, `setInterval()` | `setTimeout(() => {}, 1000)` |
| ↓ | *>> Microtasks* | `nextTick()`, `Promise.then()` | *Runs before next phase* |
| 2 | **Pending Callbacks** | Some system callbacks | TCP errors, DNS callbacks |
| ↓ | *>> Microtasks* | `nextTick()`, `Promise.then()` | *Runs before next phase* |
| 3 | **Idle, Prepare** | Internal only | (No user code) |
| ↓ | *>> Microtasks* | `nextTick()`, `Promise.then()` | *Runs before next phase* |
| 4 | **Poll** | `fs.*`, `http.*`, `net.*`, DB | `fs.readFile(cb)` |
| ↓ | *>> Microtasks* | `nextTick()`, `Promise.then()` | *Runs before next phase* |
| 5 | **Check** | `setImmediate()` | `setImmediate(() => {})` |
| ↓ | *>> Microtasks* | `nextTick()`, `Promise.then()` | *Runs before next phase* |
| 6 | **Close** | `*.on('close')` | `socket.on('close', cb)` |
| ↓ | *>> Microtasks* | `nextTick()`, `Promise.then()` | *Runs before loop repeats* |
| 🔄 | **Loop Repeats** | Back to Timers | |

**Key Point:** Microtasks (nextTick + Promises) run **between EVERY phase**, not just once!

---

### 1. Timers Phase
```javascript
// setTimeout - runs ONCE after delay
setTimeout(() => console.log('Timer 1'), 100);

// setInterval - runs REPEATEDLY
setInterval(() => console.log('Timer 2'), 100);

// Both execute in Timers phase when time >= delay
```

### 2. Poll Phase (Most Important)
```javascript
// This is where Node.js spends most time
// Waiting for:
// - Incoming HTTP requests
// - Database responses
// - File read completions

// File System
fs.readFile('file.txt', (err, data) => {
    console.log('Poll: File read complete');
});

// HTTP Server
server.on('request', (req, res) => {
    console.log('Poll: HTTP request received');
});

// Database
db.query('SELECT *', (err, rows) => {
    console.log('Poll: DB query complete');
});
```

### 3. Check Phase
```javascript
setImmediate(() => {
    console.log('Check: setImmediate callback');
});

// setImmediate vs setTimeout(0)
// Inside I/O callback: setImmediate ALWAYS runs first
fs.readFile('file.txt', () => {
    setTimeout(() => console.log('Timeout'), 0);
    setImmediate(() => console.log('Immediate')); // Runs FIRST!
});
```

### 4. Microtasks (Run Between Every Phase)

**Microtasks** are special callbacks that run **immediately after** the current operation finishes, **before** the event loop moves to the next phase.

#### Two Types of Microtasks

| Type | Code | Priority |
|------|------|----------|
| `process.nextTick()` | `process.nextTick(() => {...})` | **Highest** (runs first) |
| `Promise` | `Promise.resolve().then(() => {...})` | **High** (runs after nextTick) |

#### Example:
```javascript
Promise.resolve().then(() => {
    console.log('Microtask - runs ASAP');
});

process.nextTick(() => {
    console.log('nextTick - runs even before Promises');
});
```

#### Why "Between Every Phase"?

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Timers    │ ──▶ │  Microtasks │ ──▶ │    Poll     │
│   Phase     │     │   Queue     │     │   Phase     │
└─────────────┘     └─────────────┘     └─────────────┘
                          ↑
                    Runs HERE before
                    moving to next phase
```

**Before moving to the next phase, Node.js ALWAYS checks:**
1. Any `process.nextTick()` callbacks? → Run them
2. Any `Promise` callbacks? → Run them
3. Then proceed to next phase

#### Real-World Use Case

```javascript
// Reading file and processing data
fs.readFile('data.json', (err, data) => {
    // This runs in Poll phase
    
    process.nextTick(() => {
        // This runs IMMEDIATELY after current callback
        // Before any other I/O callbacks
        processData(data);
    });
});
```

> **Microtasks** = VIP callbacks that skip the line and run before anything else in the queue.

---

## Execution Order Example

```javascript
console.log('1: Start');

setTimeout(() => console.log('2: setTimeout'), 0);

setImmediate(() => console.log('3: setImmediate'));

Promise.resolve().then(() => console.log('4: Promise'));

process.nextTick(() => console.log('5: nextTick'));

console.log('6: End');
```

**Output:**
```
1: Start
6: End
5: nextTick      ← nextTick runs first (before microtasks)
4: Promise       ← Promise microtask
2: setTimeout    ← Timer phase (or after setImmediate, order varies)
3: setImmediate  ← Check phase
```

---

## Visual Flow: HTTP Request

```
Incoming HTTP Request
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│                      EVENT LOOP                              │
│                                                              │
│  1. Poll Phase detects new connection                        │
│         │                                                    │
│         ▼                                                    │
│  2. Callback: app.get('/users', handler)                     │
│         │                                                    │
│         ▼                                                    │
│  3. Handler calls: db.query('SELECT * FROM users')           │
│         │                                                    │
│         ├──────────────────────────────────────┐             │
│         │                                      │             │
│         ▼                                      ▼             │
│  4. Event loop continues              Thread Pool executes   │
│     (handles other requests)          database query         │
│         │                                      │             │
│         │                                      │             │
│         │◀─────────── Query complete ──────────┘             │
│         │                                                    │
│         ▼                                                    │
│  5. Callback queued to Poll phase                            │
│         │                                                    │
│         ▼                                                    │
│  6. Callback executes: res.json(results)                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Thread Pool (libuv)

```
┌─────────────────────────────────────────────────────────────┐
│                     NODE.JS PROCESS                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────┐      ┌─────────────────────────────┐  │
│   │   MAIN THREAD   │      │      THREAD POOL (libuv)    │  │
│   │   (Event Loop)  │      │  ┌─────┐ ┌─────┐ ┌─────┐   │  │
│   │                 │      │  │  T1 │ │  T2 │ │  T3 │   │  │
│   │  - JavaScript   │      │  └─────┘ └─────┘ └─────┘   │  │
│   │  - V8 Engine    │      │  ┌─────┐                    │  │
│   │  - Callbacks    │◀────▶│  │  T4 │  (Default: 4)     │  │
│   │                 │      │  └─────┘                    │  │
│   └─────────────────┘      └─────────────────────────────┘  │
│                                                              │
│   Thread Pool handles:                                       │
│   - fs.readFile, fs.writeFile                               │
│   - crypto.pbkdf2, crypto.randomBytes                       │
│   - dns.lookup                                              │
│   - zlib compression                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Increase thread pool:**
```javascript
process.env.UV_THREADPOOL_SIZE = 8; // Default is 4
```

---

## Blocking vs Non-Blocking

### ❌ Blocking (BAD)
```javascript
app.get('/data', (req, res) => {
    // Blocks event loop for 5 seconds!
    const start = Date.now();
    while (Date.now() - start < 5000) {} // CPU busy
    
    res.send('Done');
});
// ALL other requests wait 5 seconds!
```

### ✅ Non-Blocking (GOOD)
```javascript
app.get('/data', async (req, res) => {
    // Event loop is FREE during await
    const data = await db.query('SELECT ...');
    
    res.send(data);
});
// Other requests processed during await!
```

---

## Event Loop Key Takeaways

| Concept | Explanation |
|---------|-------------|
| **Single-threaded** | Only 1 thread runs JavaScript |
| **Non-blocking** | I/O operations don't block the thread |
| **Event-driven** | Callbacks execute when events occur |
| **Thread pool** | Heavy I/O offloaded to worker threads |
| **Microtasks** | Promises/nextTick run between phases |

> **Event Loop** = A single thread that continuously checks: "Any callbacks ready? Execute them. Any I/O complete? Queue their callbacks. Repeat forever."

---

## Overview

| Aspect | Node.js | Java |
|--------|---------|------|
| Model | Single-threaded, Event Loop | Multi-threaded |
| I/O | Non-blocking async | Blocking (by default) |
| Memory per connection | ~2-4 KB | ~1 MB (per thread) |

---

## Key Concept: CPU Context Switching

### What is it?

**Context switching** = CPU pausing one task to work on another task.

### Analogy: Chef in a Kitchen

```
1 Chef, 4 Dishes to cook

Without switching (ideal):
  Cook Dish 1 → Done → Cook Dish 2 → Done → ...

With switching (reality):
  Start Dish 1 → STOP → Start Dish 2 → STOP → Back to Dish 1 → STOP → ...
```

Every time the chef switches:
- Put down current ingredients
- Remember where they left off
- Pick up new ingredients
- Remember new recipe state

**This "switching" takes time and wastes effort!**

### What Happens During a Context Switch?

```
1. SAVE current thread state:
   - Register values
   - Program counter (where it was)
   - Stack pointer
   
2. LOAD new thread state:
   - Restore registers
   - Restore program counter
   - Restore stack pointer

Time taken: ~1-10 microseconds per switch
```

### Cost Calculation

```
Java with 1000 threads:
├── Context switches: ~10,000/second
├── Time per switch: ~5 microseconds
├── Time wasted: 10,000 × 5μs = 50ms per second
└── 5% CPU wasted just switching!

Node.js with 1 thread:
├── Context switches: ~0 (single thread)
├── Time wasted: ~0
└── 0% CPU wasted
```

---

## Why Java Needs More RAM & CPU

### RAM Comparison

| Java | Node.js |
|------|---------|
| **1 thread = ~1 MB** | **1 connection = ~4 KB** |
| 1000 requests = 1 GB | 1000 requests = 4 MB |

**Why:** Java creates a new thread per request. Each thread has its own memory stack.

### CPU Comparison

| Java | Node.js |
|------|---------|
| **Context switching** between 1000s of threads | **Single thread**, no switching |
| Threads compete for CPU | Event loop handles all |

**Why:** CPU wastes time switching between threads instead of doing actual work.

### Visual Summary

```
Java (Multi-threaded):
CPU: [T1][switch][T2][switch][T3][switch][T1][switch]...
      ↑           ↑           ↑           ↑
     work      overhead     work      overhead

Node.js (Single-threaded):
CPU: [event][event][event][event][event][event]...
      ↑      ↑      ↑      ↑      ↑      ↑
     work   work   work   work   work   work
     (no switching overhead)
```

---

## Case Studies

### Case 1: High-Concurrency I/O-Bound Applications

**Example: Chat Server with 10,000 Concurrent Connections**

**Java (Thread-per-request):**
```
Threads needed: 10,000
Memory per thread: ~1 MB (stack size)
Total memory: 10,000 × 1 MB = 10 GB RAM

CPU context switching: HIGH (10,000 threads)
```

**Node.js (Event Loop):**
```
Threads: 1 (+ worker pool)
Memory per connection: ~2-4 KB
Total memory: 10,000 × 4 KB = 40 MB RAM

CPU context switching: LOW (single thread)
```

**Result:** Node.js uses **250x less memory** for the same connections.

---

### Case 2: API Gateway / Proxy Server

**Scenario: 5,000 requests/sec, each waits 100ms for downstream service**

**Java Calculation:**
```
Concurrent requests = 5,000 req/sec × 0.1 sec = 500 concurrent
Threads needed = 500
Memory = 500 MB
Thread pool management overhead: HIGH
```

**Node.js Calculation:**
```
Event loop handles all 500 concurrent requests
Memory = ~50 MB
No thread management overhead
```

---

### Case 3: Real-time Applications

**WebSocket Server - 50,000 connections**

| Metric | Java (Netty) | Node.js |
|--------|--------------|---------|
| Memory | 2-3 GB | 200-400 MB |
| Latency (p99) | 5-10ms | 2-5ms |
| Code complexity | High | Low |

---

## Little's Law: Calculating Concurrent Requests

### Formula

```
Concurrent Requests = Arrival Rate × Average Response Time
```

### Example Calculation

| Value | Meaning |
|-------|---------|
| **10,000** | Requests arriving per second (arrival rate) |
| **0.25** | Each request takes 0.25 seconds (250ms) to complete |
| **2,500** | Number of requests being processed simultaneously |

```
Concurrent requests = 10,000 × 0.25 = 2,500
```

### Simple Analogy

**Restaurant Example:**
- **10,000 customers/hour** enter the restaurant
- Each customer **stays 15 minutes** (0.25 hours)
- How many customers are inside at any moment?

```
Customers inside = 10,000 × 0.25 = 2,500 customers
```

---

## Real-World Example: E-commerce Platform

### Order Processing API

**Traffic:** 10,000 requests/sec peak

**Each request:**
- DB read: 20ms
- External payment API: 200ms
- DB write: 30ms
- Total I/O wait: 250ms

**Java Calculation:**
```
Concurrent requests at peak: 10,000 × 0.25 = 2,500
Threads needed: 2,500
Memory: 2,500 MB = 2.5 GB (just for threads!)
Add JVM heap: +2 GB
Total: ~4.5 GB per instance
```

**Node.js Calculation:**
```
Single event loop handles all async I/O
Base memory: ~150 MB
Add heap for objects: ~500 MB
Total: ~650 MB per instance
```

### Cost Comparison (AWS)

```
Java: c5.xlarge (4 vCPU, 8 GB) = $0.17/hr
Node.js: t3.medium (2 vCPU, 4 GB) = $0.04/hr

Annual savings: (0.17 - 0.04) × 24 × 365 = $1,138/instance
```

---

## Benchmark: Simple HTTP Server

**Test:** 10,000 concurrent connections, 100,000 total requests

```
┌─────────────────────────────────────────────────────────┐
│                  Requests per Second                    │
├─────────────────────────────────────────────────────────┤
│ Node.js (Express)     ████████████████████  15,000 rps  │
│ Java (Spring Boot)    ████████████████      12,000 rps  │
│ Java (Netty)          █████████████████████ 18,000 rps  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    Memory Usage                         │
├─────────────────────────────────────────────────────────┤
│ Node.js               ██                    150 MB      │
│ Java (Spring Boot)    ████████████████      800 MB      │
│ Java (Netty)          ████████              400 MB      │
└─────────────────────────────────────────────────────────┘
```

---

## When to Use What

### When Node.js is BETTER

| Use Case | Why Node.js Wins |
|----------|------------------|
| **Real-time apps** (Chat, Gaming) | Low memory per connection |
| **API Gateways** | High I/O, low CPU |
| **Microservices** | Fast startup, low memory |
| **Streaming** | Non-blocking I/O |
| **Serverless** | Cold start: 100ms vs 2-5s |
| **Prototyping** | Faster development |

### When Java is BETTER

| Use Case | Why Java Wins |
|----------|---------------|
| **CPU-intensive** (ML, Data processing) | Multi-core utilization |
| **Enterprise** (Complex business logic) | Type safety, mature ecosystem |
| **Android apps** | Native support |
| **Large teams** | Strong typing, IDE support |
| **Long-running calculations** | Thread parallelism |

---

## Decision Formula

### Use Node.js when:

```
(I/O Wait Time / Total Request Time) > 0.7
AND
Concurrent Connections > 1,000
AND
CPU per request < 50ms
```

### Use Java when:

```
CPU processing time > I/O wait time
OR
Complex multi-threaded logic required
OR
Enterprise integration needed
```

---

## Summary

| Aspect | Node.js | Java |
|--------|---------|------|
| Memory per request | ~4 KB | ~1 MB |
| Context switching | None | High |
| I/O handling | Non-blocking | Blocking |
| Best for | I/O-bound, real-time | CPU-bound, enterprise |
| Startup time | ~100ms | ~2-5s |
| Cold start (serverless) | Fast | Slow |

### One-Line Summary

> **Node.js:** 1 request = 1 event = 4 KB RAM + no CPU switch
> **Java:** 1 request = 1 thread = 1 MB RAM + CPU context switch

---

# Interview Questions: Node.js vs Java

## Level 1: Conceptual Understanding

### Q1: Event Loop Blocking
> If Node.js is single-threaded, what happens if you write a CPU-intensive loop that takes 5 seconds? How does it affect other incoming requests?

<details>
<summary>Answer</summary>

**All other requests are BLOCKED for 5 seconds.**

The event loop cannot process any other events until the CPU-intensive operation completes. This is why Node.js is NOT suitable for CPU-heavy tasks.

**Solutions:**
- Use `Worker Threads` for CPU tasks
- Offload to separate microservice
- Break into smaller chunks with `setImmediate()`
</details>

---

### Q2: Thread Pool Mystery
> Node.js says it's single-threaded, but `fs.readFile()` doesn't block. How is that possible?

<details>
<summary>Answer</summary>

Node.js uses **libuv's thread pool** (default 4 threads) for:
- File system operations
- DNS lookups
- Crypto operations

The main event loop delegates I/O to the thread pool, continues processing other events, and gets notified when I/O completes.

```
Main Thread (Event Loop)
    ↓
    ├── HTTP handling
    ├── Timer callbacks
    └── Delegates I/O to Thread Pool (4 threads)
                ↓
            File read completes
                ↓
            Callback queued to Event Loop
```
</details>

---

## Level 2: Calculations

### Q3: Capacity Planning
> Your API receives **20,000 requests/second**. Each request makes a database call (50ms) and an external API call (150ms). 
> 
> Calculate:
> 1. How many concurrent requests at any moment?
> 2. How much RAM does Java need (thread-per-request)?
> 3. How much RAM does Node.js need?

<details>
<summary>Answer</summary>

**1. Concurrent requests:**
```
Total I/O time = 50ms + 150ms = 200ms = 0.2 seconds
Concurrent = 20,000 × 0.2 = 4,000 requests
```

**2. Java RAM:**
```
Threads needed = 4,000
Memory per thread = 1 MB
Thread memory = 4,000 MB = 4 GB
+ JVM heap (~2 GB)
Total = ~6 GB
```

**3. Node.js RAM:**
```
Memory per connection = ~4 KB
Connection memory = 4,000 × 4 KB = 16 MB
+ Heap (~500 MB)
Total = ~516 MB
```

**Difference: Java needs ~12x more RAM**
</details>

---

### Q4: Context Switch Cost
> A Java server has 2,000 threads on a 4-core CPU. If each context switch takes 5μs and CPU does 50,000 switches/second, what percentage of CPU is wasted?

<details>
<summary>Answer</summary>

```
Time wasted per second = 50,000 × 5μs = 250,000μs = 250ms
Total time available = 1000ms

CPU wasted = 250ms / 1000ms = 25%
```

**25% of CPU time is wasted just switching between threads!**
</details>

---

## Level 3: Architecture Decisions

### Q5: Hybrid Design
> You're building a system that:
> - Receives images via API
> - Compresses images (CPU heavy)
> - Stores in S3
> - Sends notification
>
> Would you use Node.js, Java, or both? Design the architecture.

<details>
<summary>Answer</summary>

**Best approach: Hybrid**

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Node.js API   │────▶│   Java Worker   │────▶│   Node.js       │
│  (Receive img)  │     │ (Compress - CPU)│     │  (S3 + Notify)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
     I/O bound            CPU bound              I/O bound
```

- **Node.js**: API gateway (high concurrency, I/O)
- **Java**: Image compression (CPU-intensive, multi-threaded)
- **Queue**: Message broker between services

Node.js handles 10,000+ connections with minimal RAM, Java uses all CPU cores for compression.
</details>

---

### Q6: Tricky Scenario
> Your Node.js server handles 10,000 WebSocket connections fine. You add a feature that calculates user statistics (takes 200ms CPU time per user). Now the server becomes unresponsive. Why? How do you fix it?

<details>
<summary>Answer</summary>

**Why:** 
The 200ms CPU calculation blocks the event loop. During that time, all 10,000 WebSocket connections can't receive messages.

**Fixes:**

1. **Worker Threads:**
```javascript
const { Worker } = require('worker_threads');
const worker = new Worker('./stats-calculator.js');
worker.postMessage(userId);
```

2. **Separate Microservice:**
```
WebSocket Server (Node.js) ──▶ Stats Service (Java/Go)
```

3. **Break into chunks:**
```javascript
async function calculateStats(users) {
    for (const user of users) {
        await calculateOne(user);
        await setImmediate(); // Let event loop breathe
    }
}
```
</details>

---

## Level 4: Deep Dive

### Q7: Memory Model
> Explain why Java's thread stack is ~1MB while Node.js connection overhead is ~4KB. What's stored in each?

<details>
<summary>Answer</summary>

**Java Thread Stack (1MB):**
```
├── Local variables for each method call
├── Method call frames (recursion depth)
├── Return addresses
├── Thread-local storage
└── JVM internal bookkeeping
```
Pre-allocated to handle deep recursion without stack overflow.

**Node.js Connection (~4KB):**
```
├── Socket file descriptor (~100 bytes)
├── Buffer for incoming data (~1-2 KB)
├── Callback reference (~100 bytes)
└── Connection state (~100 bytes)
```
Only stores connection metadata, no execution stack needed.

**Key difference:** Java needs execution context per thread. Node.js only needs data context per connection.
</details>

---

### Q8: Gotcha Question
> Node.js `cluster` module creates multiple processes. Java `ThreadPoolExecutor` creates multiple threads. 
>
> If both use 4 workers on a 4-core machine, which performs better for:
> 1. CPU-bound tasks?
> 2. I/O-bound tasks?

<details>
<summary>Answer</summary>

**1. CPU-bound tasks:**
```
Node.js Cluster: 4 processes × 1 core = 4 cores utilized ✅
Java ThreadPool: 4 threads × 1 core = 4 cores utilized ✅

Result: EQUAL (both fully utilize CPU)
```

**2. I/O-bound tasks (10,000 concurrent):**
```
Node.js Cluster: 4 processes × 2,500 connections = 10,000 ✅
                 Memory: 4 × 100MB = 400MB

Java ThreadPool: Only 4 threads, need more threads!
                 To handle 10,000: Need 10,000 threads
                 Memory: 10,000 × 1MB = 10GB ❌

Result: Node.js WINS (250x less memory for same load)
```

**Gotcha:** Thread pools limit concurrency. Event loops don't.
</details>

---

## Level 5: Real-World Problem

### Q9: Debug This
> Your Node.js API normally handles 5,000 req/sec. After adding a new feature, it drops to 500 req/sec. The feature:
> ```javascript
> app.get('/report', async (req, res) => {
>     const data = await db.query('SELECT * FROM large_table');
>     const result = JSON.stringify(data); // 100MB data
>     res.send(result);
> });
> ```
> What's wrong? How do you fix it?

<details>
<summary>Answer</summary>

**Problems:**

1. **Large data in memory:** 100MB × concurrent requests = memory exhaustion
2. **JSON.stringify blocks event loop:** Serializing 100MB takes 500ms+
3. **Single response:** All data sent at once, blocks others

**Fixes:**

```javascript
app.get('/report', async (req, res) => {
    // 1. Stream from database
    const cursor = db.queryCursor('SELECT * FROM large_table');
    
    // 2. Stream to response
    res.setHeader('Content-Type', 'application/json');
    res.write('[');
    
    let first = true;
    for await (const row of cursor) {
        if (!first) res.write(',');
        res.write(JSON.stringify(row));
        first = false;
    }
    
    res.write(']');
    res.end();
});
```

**Benefits:**
- Constant memory usage (~1MB)
- Event loop not blocked
- Response starts immediately
</details>

