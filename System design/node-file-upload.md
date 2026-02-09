# System Design: Handling 10K File Uploads/Second in Node.js

## Overview

| Metric | Value |
|--------|-------|
| Requests | 10,000/second |
| File size (avg) | 5 MB |
| Storage | Local VM disk |
| Scaling | Kubernetes (horizontal) |

---

# How Node.js Handles 10K Requests Internally

## The Key Insight

```
10,000 requests arrive
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│                    SINGLE THREAD                              │
│                    (Event Loop)                               │
│                                                               │
│   Does NOT process 10K requests simultaneously!               │
│   Instead:                                                    │
│                                                               │
│   1. Accept request → Hand off to Thread Pool → NEXT!        │
│   2. Accept request → Hand off to Thread Pool → NEXT!        │
│   3. Accept request → Hand off to Thread Pool → NEXT!        │
│   ...                                                         │
│   10000. Accept request → Hand off → NEXT!                   │
│                                                               │
│   Time to accept 10K requests: ~10ms (just routing!)         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Node.js Does Very Little Work Per Request

```javascript
app.post('/upload', async (req, res) => {
    // What Node.js ACTUALLY does:
    
    // 1. Parse headers (CPU: ~0.1ms) ✅ Event Loop
    // 2. Call fs.writeFile() (CPU: ~0.01ms) ✅ Event Loop
    //    └── Actual write happens in Thread Pool!
    // 3. Wait... (CPU: 0ms) - Event Loop is FREE
    // 4. Callback fires (CPU: ~0.1ms) ✅ Event Loop
    // 5. Send response (CPU: ~0.1ms) ✅ Event Loop
    
    // Total CPU time: ~0.3ms per request
    // Event Loop can handle: 1000ms / 0.3ms = 3,333 requests/sec per core
});
```

---

## The Magic: Delegation

```
Event Loop's job: ACCEPT and DELEGATE (not PROCESS)

┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   REQUEST ARRIVES                                                │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              EVENT LOOP (Single Thread)                  │   │
│   │                                                          │   │
│   │   Time spent: 0.1ms                                      │   │
│   │   Work done: Parse request, create task, delegate        │   │
│   │                                                          │   │
│   └─────────────────────────┬────────────────────────────────┘   │
│                             │                                    │
│                             │ Delegate to OS/Thread Pool         │
│                             ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    THREAD POOL (4 threads)               │   │
│   │                                                          │   │
│   │   Thread 1: Writing file chunk 1                         │   │
│   │   Thread 2: Writing file chunk 2                         │   │
│   │   Thread 3: Writing file chunk 3                         │   │
│   │   Thread 4: Writing file chunk 4                         │   │
│   │                                                          │   │
│   │   Time spent: 100ms (but parallel!)                      │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                             │                                    │
│                             │ Callback when done                 │
│                             ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              EVENT LOOP                                  │   │
│   │                                                          │   │
│   │   Time spent: 0.1ms                                      │   │
│   │   Work done: Send response                               │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Event Loop CPU time: 0.2ms
Actual work time: 100ms (done by thread pool, NOT event loop)
```

---

## Timeline: 10K Requests in 1 Second

```
Time    Event Loop                         Thread Pool
────────────────────────────────────────────────────────────────
0ms     Accept req 1, delegate            Thread 1: Start file write
0.1ms   Accept req 2, delegate            Thread 2: Start file write
0.2ms   Accept req 3, delegate            Thread 3: Start file write
0.3ms   Accept req 4, delegate            Thread 4: Start file write
0.4ms   Accept req 5, delegate            (waiting for free thread)
...
10ms    Accept req 100, delegate          Threads still writing...
...
100ms   Callback: req 1 done, respond     Thread 1: FREE, takes req 5
100.1ms Callback: req 2 done, respond     Thread 2: FREE, takes req 6
...
1000ms  All 10K requests handled!

Event Loop: Spent ~200ms total (accepting + responding)
           FREE for ~800ms (doing nothing, just waiting)
```

---

## Why It Works: I/O is Slow, CPU is Fast

| Operation | Time | Who Does It |
|-----------|------|-------------|
| Parse HTTP headers | 0.1ms | Event Loop (CPU) |
| Validate request | 0.1ms | Event Loop (CPU) |
| Write 5MB to disk | 50-100ms | Thread Pool (I/O) |
| Send response | 0.1ms | Event Loop (CPU) |

**Node.js secret:** The slow part (disk write) is NOT done by Event Loop!

---

## The Math

```
Per request CPU work: 0.3ms
Requests per second (1 core): 1000ms / 0.3ms = 3,333 req/sec

With 8 cores (cluster): 3,333 × 8 = 26,664 req/sec

10K requests? Easy! ✅
```

---

# Code Implementation

## ❌ Wrong Code: Same Filename Bug

```javascript
app.post('/upload', (req, res) => {
    const writeStream = fs.createWriteStream('file.txt');  // ❌ SAME FILE!
    req.pipe(writeStream);
    writeStream.on('finish', () => res.send('OK'));
});
```

**What happens with 3 simultaneous requests:**

```
Request 1 ──▶ Write to file.txt ──▶ "Hello"
Request 2 ──▶ Write to file.txt ──▶ "World"     ← OVERWRITES!
Request 3 ──▶ Write to file.txt ──▶ "Test"      ← OVERWRITES!

Result: file.txt contains corrupted/mixed data!
```

---

## ✅ Correct Code: Unique Filenames with Streaming

```javascript
const express = require('express');
const fs = require('fs');
const path = require('path');
const { v4: uuidv4 } = require('uuid');

const app = express();
const UPLOAD_DIR = './uploads';

// Ensure upload directory exists
if (!fs.existsSync(UPLOAD_DIR)) {
    fs.mkdirSync(UPLOAD_DIR, { recursive: true });
}

app.post('/upload', (req, res) => {
    const fileId = uuidv4();
    const filePath = path.join(UPLOAD_DIR, `${fileId}.dat`);
    const writeStream = fs.createWriteStream(filePath);
    
    // Track progress
    let bytesReceived = 0;
    req.on('data', (chunk) => {
        bytesReceived += chunk.length;
    });
    
    // Stream directly from request to file - NO buffering!
    req.pipe(writeStream);
    
    // Handle completion
    writeStream.on('finish', () => {
        res.json({ 
            success: true, 
            fileId,
            size: bytesReceived 
        });
    });
    
    // Handle errors
    writeStream.on('error', (err) => {
        console.error('Write error:', err);
        res.status(500).json({ error: 'Upload failed' });
    });
    
    req.on('error', (err) => {
        console.error('Request error:', err);
        writeStream.destroy();
        res.status(500).json({ error: 'Upload failed' });
    });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## Memory Comparison: Buffering vs Streaming

```
WITHOUT streaming (BAD):
─────────────────────────
let data = [];
req.on('data', chunk => data.push(chunk));  // Accumulate in memory
req.on('end', () => {
    const buffer = Buffer.concat(data);     // 5MB in memory per request!
    fs.writeFile('file.txt', buffer, cb);
});

Memory: 10,000 × 5MB = 50GB ❌ CRASH!


WITH streaming (GOOD):
──────────────────────
req.pipe(writeStream);  // Stream directly, ~64KB buffer

Memory: 10,000 × 64KB = 640MB ✅
```

---

## How Multiple Requests Run in Parallel

```
Time 0ms:
┌─────────────────────────────────────────────────────────────────┐
│                      EVENT LOOP                                  │
│                                                                  │
│   Request 1 arrives → Create stream → pipe() → NEXT!            │
│   Request 2 arrives → Create stream → pipe() → NEXT!            │
│   Request 3 arrives → Create stream → pipe() → NEXT!            │
│                                                                  │
│   Time spent: ~0.3ms per request                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     THREAD POOL (4 threads)                      │
│                                                                  │
│   Thread 1: Writing file_1001.dat ████████░░░░░░                │
│   Thread 2: Writing file_1002.dat ██████░░░░░░░░                │
│   Thread 3: Writing file_1003.dat ████░░░░░░░░░░                │
│   Thread 4: Waiting for more work...                            │
│                                                                  │
│   All 3 files written IN PARALLEL!                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Proof: Requests Run Concurrently

```javascript
app.post('/upload', (req, res) => {
    const startTime = Date.now();
    const fileId = `file_${startTime}_${Math.random().toString(36).slice(2)}`;
    
    console.log(`[${startTime}] Request started: ${fileId}`);
    
    const writeStream = fs.createWriteStream(`./uploads/${fileId}.dat`);
    req.pipe(writeStream);
    
    writeStream.on('finish', () => {
        const endTime = Date.now();
        console.log(`[${endTime}] Request finished: ${fileId} (${endTime - startTime}ms)`);
        res.json({ success: true, fileId });
    });
});
```

**Send 3 requests simultaneously - Output:**
```
[1000] Request started: file_1000_abc123
[1001] Request started: file_1001_def456    ← Started BEFORE 1000 finished!
[1002] Request started: file_1002_ghi789    ← Started BEFORE 1000 finished!
[1050] Request finished: file_1000_abc123 (50ms)
[1055] Request finished: file_1001_def456 (54ms)
[1060] Request finished: file_1002_ghi789 (58ms)
```

**All 3 started within 2ms, ran in parallel!**

---

# Scaling with Kubernetes

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────────┐                                          │
│   │  Ingress / LB    │  ← Routes traffic to pods                │
│   └────────┬─────────┘                                          │
│            │                                                     │
│            ├─────────────────┬─────────────────┐                │
│            ▼                 ▼                 ▼                │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│   │    Pod 1     │  │    Pod 2     │  │    Pod N     │         │
│   │  (Node.js    │  │  (Node.js    │  │  (Node.js    │         │
│   │   Cluster)   │  │   Cluster)   │  │   Cluster)   │         │
│   │  8 workers   │  │  8 workers   │  │  8 workers   │         │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
│          │                 │                 │                  │
│          └─────────────────┼─────────────────┘                  │
│                            │                                     │
│                            ▼                                     │
│                   ┌─────────────────┐                           │
│                   │ Persistent Vol  │  ← Shared storage (NFS)   │
│                   │   /uploads      │                           │
│                   └─────────────────┘                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Capacity Planning

| Component | Capacity per Instance | Instances for 10K/sec |
|-----------|----------------------|----------------------|
| Node.js Pod (8 core) | ~8,000 req/sec | 2 pods minimum |
| With buffer (80% capacity) | ~6,400 req/sec | 2-3 pods |
| Disk I/O (SSD) | ~5,000 writes/sec | May need multiple volumes |

---

## Using PM2 Cluster Mode

```javascript
// server.js - Uses all CPU cores
const cluster = require('cluster');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
    console.log(`Master ${process.pid} starting ${numCPUs} workers`);
    
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }
    
    cluster.on('exit', (worker) => {
        console.log(`Worker ${worker.process.pid} died, restarting...`);
        cluster.fork();
    });
} else {
    require('./app.js');
    console.log(`Worker ${process.pid} started`);
}
```

**Or use PM2:**
```bash
pm2 start app.js -i max  # Uses all CPU cores
```

---

## Increase Thread Pool Size

```javascript
// Set BEFORE requiring any modules
process.env.UV_THREADPOOL_SIZE = 16;  // Default is 4

const express = require('express');
// ... rest of app
```

**Why:** More threads = more parallel file writes

---

## Production-Ready Code

```javascript
const express = require('express');
const fs = require('fs');
const path = require('path');
const { v4: uuidv4 } = require('uuid');
const cluster = require('cluster');
const numCPUs = require('os').cpus().length;

// Increase thread pool for more parallel I/O
process.env.UV_THREADPOOL_SIZE = 16;

const UPLOAD_DIR = process.env.UPLOAD_DIR || './uploads';
const PORT = process.env.PORT || 3000;
const MAX_FILE_SIZE = 100 * 1024 * 1024; // 100MB

if (cluster.isMaster) {
    console.log(`Master ${process.pid} starting ${numCPUs} workers`);
    
    // Ensure upload directory exists
    if (!fs.existsSync(UPLOAD_DIR)) {
        fs.mkdirSync(UPLOAD_DIR, { recursive: true });
    }
    
    for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
    }
    
    cluster.on('exit', (worker) => {
        console.log(`Worker ${worker.process.pid} died, restarting...`);
        cluster.fork();
    });
} else {
    const app = express();
    
    // Health check for Kubernetes
    app.get('/health', (req, res) => {
        res.json({ status: 'healthy', pid: process.pid });
    });
    
    // File upload endpoint
    app.post('/upload', (req, res) => {
        const fileId = uuidv4();
        const filePath = path.join(UPLOAD_DIR, `${fileId}.dat`);
        const writeStream = fs.createWriteStream(filePath);
        
        let bytesReceived = 0;
        
        req.on('data', (chunk) => {
            bytesReceived += chunk.length;
            
            // Check file size limit
            if (bytesReceived > MAX_FILE_SIZE) {
                req.destroy();
                writeStream.destroy();
                fs.unlink(filePath, () => {}); // Cleanup partial file
                return res.status(413).json({ error: 'File too large' });
            }
        });
        
        req.pipe(writeStream);
        
        writeStream.on('finish', () => {
            res.json({ 
                success: true, 
                fileId,
                size: bytesReceived,
                worker: process.pid
            });
        });
        
        writeStream.on('error', (err) => {
            console.error(`[Worker ${process.pid}] Write error:`, err);
            fs.unlink(filePath, () => {}); // Cleanup
            res.status(500).json({ error: 'Upload failed' });
        });
        
        req.on('error', (err) => {
            console.error(`[Worker ${process.pid}] Request error:`, err);
            writeStream.destroy();
            fs.unlink(filePath, () => {}); // Cleanup
        });
    });
    
    app.listen(PORT, () => {
        console.log(`Worker ${process.pid} listening on port ${PORT}`);
    });
}
```

---

# Summary

## How Node.js Handles 10K Uploads

| Step | What Happens | Time |
|------|--------------|------|
| 1 | Request arrives | 0ms |
| 2 | Event Loop parses headers | 0.1ms |
| 3 | Event Loop delegates I/O to Thread Pool | 0.01ms |
| 4 | Event Loop moves to NEXT request | 0ms |
| 5 | Thread Pool writes file to disk | 50-100ms |
| 6 | Callback queued when done | 0ms |
| 7 | Event Loop sends response | 0.1ms |

**Event Loop time per request: ~0.3ms**
**Actual I/O time: ~100ms (done by Thread Pool!)**

---

## Key Points

| Aspect | Solution |
|--------|----------|
| **Concurrency** | Event Loop delegates to Thread Pool |
| **Memory** | Streaming (64KB buffer vs full file) |
| **CPU Utilization** | Cluster mode (all cores) |
| **Scaling** | Kubernetes horizontal scaling |
| **Thread Pool** | Increase UV_THREADPOOL_SIZE |
| **Unique Files** | UUID for each upload |

---

## One-Line Summary

> "Node.js handles 10K file uploads by spending only 0.3ms per request on the Event Loop, streaming files to disk via Thread Pool, and scaling horizontally with Kubernetes pods."

