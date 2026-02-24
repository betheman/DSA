# GCP Serverless & Large File Processing — Quick Reference

## Cloud Functions Overview

- Serverless, event-driven compute — no servers to manage
- Auto-scales from 0 to thousands, pay only for execution time
- Runtimes: Node.js, Python, Go, Java, .NET, Ruby, PHP

### Gen 1 vs Gen 2 (Use Gen 2)

- Gen 1: 9 min timeout, 8 GB memory, 2 vCPUs, 1 request/instance
- Gen 2: 60 min timeout, 32 GB memory, 8 vCPUs, up to 1000 requests/instance, built on Cloud Run

### Good Use Cases

- Lightweight REST/GraphQL APIs
- Event processing (file upload to GCS, Firestore document change)
- Webhooks (Stripe, GitHub)
- Scheduled/cron tasks
- Small data transformations, ETL
- IoT / Pub/Sub message processing

### Bad Use Cases

- Long-running jobs over 60 min — use Cloud Run Jobs or Compute Engine
- Large file processing over 32 MB payload — use direct GCS upload + event trigger
- Persistent connections — use Cloud Run or GKE
- Workloads needing more than 32 GB memory — use Compute Engine
- GPU/ML training — use Vertex AI or Compute Engine

### Pros

- Zero server management
- Auto-scaling (instant, milliseconds)
- Pay-per-use (100ms granularity)
- Fast deployment
- Built-in IAM, VPC connectors, secret management
- Native triggers from GCS, Pub/Sub, Firestore

### Cons

- Cold starts (100ms–2s on first invocation after idle)
- 60 min max timeout, 32 GB max memory
- 32 MB HTTP payload, 10 MB event payload
- Stateless — no persistent local storage
- Vendor lock-in to GCP
- Harder to debug than local apps
- Can get expensive at high volume

---

## Node.js Usage Examples

### HTTP Function

```javascript
const functions = require('@google-cloud/functions-framework');

functions.http('helloWorld', (req, res) => {
  const name = req.query.name || req.body.name || 'World';
  res.json({ message: `Hello, ${name}!` });
});
```

### Cloud Storage Trigger

```javascript
const functions = require('@google-cloud/functions-framework');
const { Storage } = require('@google-cloud/storage');

functions.cloudEvent('processFile', async (cloudEvent) => {
  const file = cloudEvent.data;
  const storage = new Storage();
  const [metadata] = await storage.bucket(file.bucket).file(file.name).getMetadata();
  console.log(`File: ${file.name}, Size: ${file.size}, Type: ${metadata.contentType}`);
});
```

### Pub/Sub Trigger

```javascript
const functions = require('@google-cloud/functions-framework');

functions.cloudEvent('processPubSub', (cloudEvent) => {
  const message = cloudEvent.data.message;
  const data = JSON.parse(Buffer.from(message.data, 'base64').toString());
  console.log('Received:', data);
});
```

### Firestore Trigger

```javascript
const functions = require('@google-cloud/functions-framework');

functions.cloudEvent('onUserCreated', async (cloudEvent) => {
  const document = cloudEvent.data;
  const userId = cloudEvent.subject.split('/').pop();
  console.log(`New user: ${userId}`, document.value.fields);
});
```

### Scheduled Function (Cron)

```javascript
const functions = require('@google-cloud/functions-framework');
const { Firestore } = require('@google-cloud/firestore');

functions.cloudEvent('dailyCleanup', async () => {
  const db = new Firestore();
  const cutoff = new Date();
  cutoff.setDate(cutoff.getDate() - 30);

  const snapshot = await db.collection('temp_data').where('createdAt', '<', cutoff).limit(500).get();
  const batch = db.batch();
  snapshot.docs.forEach((doc) => batch.delete(doc.ref));
  await batch.commit();
  console.log(`Deleted ${snapshot.size} old documents`);
});
```

### HTTP Function that Queues Background Work

```javascript
const functions = require('@google-cloud/functions-framework');
const { PubSub } = require('@google-cloud/pubsub');
const pubsub = new PubSub();

functions.http('queueWork', async (req, res) => {
  const { taskType, data } = req.body;
  if (!taskType || !data) return res.status(400).json({ error: 'Missing taskType or data' });

  const messageId = await pubsub.topic('background-tasks').publishMessage({
    data: Buffer.from(JSON.stringify({ taskType, data })),
    attributes: { taskType },
  });

  res.json({ message: 'Task queued', messageId, status: 'processing' });
});
```

### Function with Secret Manager

```javascript
const functions = require('@google-cloud/functions-framework');
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');
const client = new SecretManagerServiceClient();

async function getSecret(secretName) {
  const [version] = await client.accessSecretVersion({
    name: `projects/my-project/secrets/${secretName}/versions/latest`,
  });
  return version.payload.data.toString();
}

functions.http('secureEndpoint', async (req, res) => {
  const apiKey = await getSecret('external-api-key');
  const response = await fetch('https://api.example.com/data', {
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  res.json(await response.json());
});
```

### package.json

```json
{
  "name": "cloud-functions",
  "version": "1.0.0",
  "main": "index.js",
  "engines": { "node": ">=20" },
  "dependencies": {
    "@google-cloud/functions-framework": "^3.3.0",
    "@google-cloud/storage": "^7.7.0",
    "@google-cloud/firestore": "^7.3.0",
    "@google-cloud/pubsub": "^4.3.0",
    "@google-cloud/secret-manager": "^5.0.0"
  }
}
```

---

## Best Practices

- Initialize clients **outside** the handler — they get reused across invocations
- Use async/await everywhere
- Return proper HTTP status codes on error
- Don't set max timeout unnecessarily — use the smallest value that works
- Start with minimum memory, increase only if needed
- Set `min-instances=1` for critical functions to reduce cold starts
- Use structured logging for Cloud Logging

```javascript
const { Firestore } = require('@google-cloud/firestore');
const db = new Firestore(); // initialized once, reused across invocations

exports.handler = async (req, res) => {
  const doc = await db.collection('users').doc(req.params.id).get();
  res.json(doc.data());
};
```

---

## File Processing by Size — Examples

### Small Files (< 10 MB) — Process In-Memory in Cloud Function

Entire file fits in memory. Read it all at once, process, done. No streaming needed.

- **Service**: Cloud Function (Gen 2)
- **Memory**: 256 MB
- **Timeout**: 60 seconds
- **Strategy**: read entire file into memory, parse, save results

```javascript
const functions = require('@google-cloud/functions-framework');
const { Storage } = require('@google-cloud/storage');
const { Firestore } = require('@google-cloud/firestore');

const storage = new Storage();
const db = new Firestore();

functions.cloudEvent('processSmallFile', async (cloudEvent) => {
  const { bucket, name } = cloudEvent.data;
  const file = storage.bucket(bucket).file(name);

  // Safe to load entirely — file is under 10 MB
  const [contents] = await file.download();
  const rows = JSON.parse(contents.toString());

  const batch = db.batch();
  rows.forEach((row) => {
    const ref = db.collection('imported_data').doc();
    batch.set(ref, { ...row, importedAt: new Date(), source: name });
  });
  await batch.commit();

  console.log(`Processed ${rows.length} rows from ${name}`);
});
```

### Medium Files (10 MB – 256 MB) — Stream in a Single Cloud Function

Too large to load into memory safely. Stream line by line within one function invocation.

- **Service**: Cloud Function (Gen 2)
- **Memory**: 512 MB – 1 GB
- **Timeout**: 5–15 minutes
- **Strategy**: stream with readline, batch writes every N records

```javascript
const functions = require('@google-cloud/functions-framework');
const { Storage } = require('@google-cloud/storage');
const { Firestore } = require('@google-cloud/firestore');
const readline = require('readline');

const storage = new Storage();
const db = new Firestore();

functions.cloudEvent('processMediumFile', async (cloudEvent) => {
  const { bucket, name } = cloudEvent.data;
  const readStream = storage.bucket(bucket).file(name).createReadStream();
  const rl = readline.createInterface({ input: readStream, crlfDelay: Infinity });

  let batch = db.batch();
  let count = 0;
  const BATCH_SIZE = 500; // Firestore batch limit is 500

  for await (const line of rl) {
    if (!line.trim()) continue;
    const record = JSON.parse(line); // assumes JSONL (one JSON object per line)
    const ref = db.collection('imported_data').doc();
    batch.set(ref, { ...record, importedAt: new Date() });
    count++;

    if (count % BATCH_SIZE === 0) {
      await batch.commit();
      batch = db.batch();
    }
  }

  if (count % BATCH_SIZE !== 0) await batch.commit();
  console.log(`Streamed and saved ${count} records from ${name}`);
});
```

### Large Files (256 MB – 1 GB) — Fan-Out via Pub/Sub

Single function might timeout. Split the work: one function reads and publishes chunks, many workers process in parallel.

- **Service**: Cloud Function (orchestrator) + Cloud Function (workers)
- **Memory**: 1–2 GB for orchestrator, 512 MB for workers
- **Timeout**: 30 minutes for orchestrator
- **Strategy**: orchestrator reads file, publishes batches to Pub/Sub, workers process each batch independently

```javascript
// === ORCHESTRATOR: reads file, publishes batches to Pub/Sub ===

const functions = require('@google-cloud/functions-framework');
const { Storage } = require('@google-cloud/storage');
const { PubSub } = require('@google-cloud/pubsub');
const readline = require('readline');

const storage = new Storage();
const pubsub = new PubSub();
const topic = pubsub.topic('file-chunks');

functions.cloudEvent('orchestrateLargeFile', async (cloudEvent) => {
  const { bucket, name } = cloudEvent.data;
  const readStream = storage.bucket(bucket).file(name).createReadStream();
  const rl = readline.createInterface({ input: readStream, crlfDelay: Infinity });

  let batch = [];
  let batchIndex = 0;
  const CHUNK_SIZE = 500;

  for await (const line of rl) {
    if (!line.trim()) continue;
    batch.push(line);

    if (batch.length >= CHUNK_SIZE) {
      await topic.publishMessage({
        data: Buffer.from(JSON.stringify({ records: batch, batchIndex, source: name })),
      });
      batchIndex++;
      batch = [];
    }
  }

  if (batch.length > 0) {
    await topic.publishMessage({
      data: Buffer.from(JSON.stringify({ records: batch, batchIndex, source: name })),
    });
  }

  console.log(`Published ${batchIndex + 1} batches from ${name}`);
});
```

```javascript
// === WORKER: processes one batch from Pub/Sub ===

const functions = require('@google-cloud/functions-framework');
const { Firestore } = require('@google-cloud/firestore');

const db = new Firestore();

functions.cloudEvent('processChunk', async (cloudEvent) => {
  const message = cloudEvent.data.message;
  const { records, batchIndex, source } = JSON.parse(
    Buffer.from(message.data, 'base64').toString(),
  );

  const batch = db.batch();
  records.forEach((line) => {
    const record = JSON.parse(line);
    const ref = db.collection('imported_data').doc();
    batch.set(ref, { ...record, importedAt: new Date(), source, batchIndex });
  });
  await batch.commit();

  console.log(`Worker processed batch #${batchIndex} (${records.length} records)`);
});
```

### Very Large Files (> 1 GB) — Cloud Run Job with Checkpointing

Too large and too slow for Cloud Functions. Use Cloud Run Jobs with checkpoint-based resumability.

- **Service**: Cloud Run Job
- **Memory**: 2–4 GB
- **Timeout**: up to 24 hours
- **Strategy**: stream from last checkpoint, save progress every 10 MB, auto-retry on failure

```javascript
const { Storage } = require('@google-cloud/storage');
const { Firestore } = require('@google-cloud/firestore');
const readline = require('readline');

const storage = new Storage();
const db = new Firestore();

async function processVeryLargeFile(bucketName, fileName, jobId) {
  const checkpointRef = db.collection('processing_checkpoints').doc(jobId);
  const checkpoint = await checkpointRef.get();
  const startByte = checkpoint.exists ? checkpoint.data().lastByte : 0;

  const file = storage.bucket(bucketName).file(fileName);
  const [metadata] = await file.getMetadata();
  const totalSize = parseInt(metadata.size);

  console.log(`Resuming from byte ${startByte} of ${totalSize} (${Math.round((startByte / totalSize) * 100)}%)`);

  const readStream = file.createReadStream({ start: startByte, end: totalSize });
  const rl = readline.createInterface({ input: readStream });

  let processedBytes = startByte;
  let batchData = [];
  let lastCheckpointByte = startByte;
  const BATCH_SIZE = 1000;
  const CHECKPOINT_INTERVAL = 10 * 1024 * 1024; // 10 MB

  for await (const line of rl) {
    const extracted = JSON.parse(line);
    batchData.push(extracted);
    processedBytes += Buffer.byteLength(line, 'utf8') + 1;

    if (batchData.length >= BATCH_SIZE) {
      const writeBatch = db.batch();
      batchData.forEach((item) => {
        const ref = db.collection(`extracted_${jobId}`).doc();
        writeBatch.set(ref, item);
      });
      await writeBatch.commit();
      batchData = [];
    }

    if (processedBytes - lastCheckpointByte >= CHECKPOINT_INTERVAL) {
      await checkpointRef.set({
        lastByte: processedBytes,
        progress: Math.round((processedBytes / totalSize) * 100),
        status: 'processing',
        updatedAt: new Date(),
      });
      lastCheckpointByte = processedBytes;
    }
  }

  if (batchData.length > 0) {
    const writeBatch = db.batch();
    batchData.forEach((item) => {
      const ref = db.collection(`extracted_${jobId}`).doc();
      writeBatch.set(ref, item);
    });
    await writeBatch.commit();
  }

  await checkpointRef.set({
    lastByte: totalSize,
    progress: 100,
    status: 'completed',
    completedAt: new Date(),
  });

  console.log(`Completed processing ${totalSize} bytes`);
}

// Entry point — crashes resume from checkpoint automatically
async function main() {
  const { JOB_ID: jobId, BUCKET: bucket, FILE: file } = process.env;
  try {
    await processVeryLargeFile(bucket, file, jobId);
  } catch (error) {
    console.error('Failed (will retry from checkpoint):', error.message);
    process.exit(1);
  }
}

main();
```

### Quick Decision

- **< 10 MB**: load into memory, single Cloud Function, 256 MB RAM
- **10–256 MB**: stream line-by-line, single Cloud Function, 512 MB–1 GB RAM
- **256 MB–1 GB**: fan-out via Pub/Sub, orchestrator + worker Cloud Functions
- **> 1 GB**: Cloud Run Job with checkpointing, 2–4 GB RAM, up to 24 hours
- **> 10 GB**: consider Dataflow, GKE, or splitting the file upstream

---

## Large File Processing Architecture

### The Problem

Cloud Functions can fail due to:

- Timeout limits (60 min max for Gen 2)
- Memory exhaustion
- Cold start / scaling issues
- Transient infrastructure failures

For files over 1 GB with data extraction, you need **resumable processing with checkpointing**.

### Recommended Flow

- **Upload**: Client uploads directly to GCS via signed URL (resumable)
- **Trigger**: GCS event fires a Cloud Function, which publishes to Pub/Sub
- **Process**: Cloud Run Job picks up the message, processes the file with checkpointing
- **Failure Recovery**: Job crashes → Cloud Run auto-retries (up to 3x) → retried job reads checkpoint from Firestore → resumes from last position
- **Dead Letter**: After 3 failures, message goes to DLQ for manual investigation

### Processing Job with Checkpointing (Cloud Run Job)

Cloud Run Jobs are better than Cloud Functions for this:

- Up to 24-hour timeout
- Automatic retries
- No request/response overhead

```javascript
const { Storage } = require('@google-cloud/storage');
const { Firestore } = require('@google-cloud/firestore');
const readline = require('readline');

const storage = new Storage();
const db = new Firestore();

async function processLargeFile(bucketName, fileName, jobId) {
  const checkpointRef = db.collection('processing_checkpoints').doc(jobId);
  const checkpoint = await checkpointRef.get();
  const startByte = checkpoint.exists ? checkpoint.data().lastByte : 0;

  const file = storage.bucket(bucketName).file(fileName);
  const [metadata] = await file.getMetadata();
  const totalSize = parseInt(metadata.size);

  const readStream = file.createReadStream({ start: startByte, end: totalSize });
  const rl = readline.createInterface({ input: readStream });

  let processedBytes = startByte;
  let batchData = [];
  const BATCH_SIZE = 1000;
  const CHECKPOINT_INTERVAL = 10 * 1024 * 1024; // every 10 MB
  let lastCheckpointByte = startByte;

  for await (const line of rl) {
    const extracted = extractData(line);
    batchData.push(extracted);
    processedBytes += Buffer.byteLength(line, 'utf8') + 1;

    if (batchData.length >= BATCH_SIZE) {
      await saveBatch(batchData, jobId);
      batchData = [];
    }

    if (processedBytes - lastCheckpointByte >= CHECKPOINT_INTERVAL) {
      await checkpointRef.set({
        lastByte: processedBytes,
        progress: Math.round((processedBytes / totalSize) * 100),
        updatedAt: new Date(),
      });
      lastCheckpointByte = processedBytes;
    }
  }

  if (batchData.length > 0) await saveBatch(batchData, jobId);

  await checkpointRef.set({
    lastByte: totalSize,
    progress: 100,
    status: 'completed',
    completedAt: new Date(),
  });
}
```

### GCS Upload Trigger (Cloud Function → Pub/Sub)

```javascript
exports.onFileUploaded = async (event) => {
  const { PubSub } = require('@google-cloud/pubsub');
  const pubsub = new PubSub();

  await pubsub.topic('file-processing').publishMessage({
    data: Buffer.from(
      JSON.stringify({
        bucket: event.bucket,
        file: event.name,
        jobId: `job_${Date.now()}_${event.name}`,
        attempt: 1,
      }),
    ),
  });
};
```

### Retry Handler (Cloud Run Job Entry Point)

```javascript
async function main() {
  const { JOB_ID: jobId, BUCKET: bucket, FILE: file } = process.env;

  try {
    await processLargeFile(bucket, file, jobId);
    console.log('Processing completed');
  } catch (error) {
    console.error('Processing failed:', error);
    process.exit(1); // checkpoint was saved — next retry resumes from there
  }
}

main();
```

### Status API

```javascript
exports.getProcessingStatus = async (req, res) => {
  const checkpoint = await db.collection('processing_checkpoints').doc(req.params.jobId).get();

  if (!checkpoint.exists) return res.status(404).json({ error: 'Job not found' });

  res.json({
    jobId: req.params.jobId,
    progress: checkpoint.data().progress,
    status: checkpoint.data().status || 'processing',
    updatedAt: checkpoint.data().updatedAt,
  });
};
```

### Architecture Benefits

- **Checkpointing**: saves progress every 10 MB, resumes from last checkpoint
- **Cloud Run Jobs**: 24-hour timeout, automatic retries (up to 3)
- **Pub/Sub + DLQ**: messages retry automatically, dead letter queue catches repeated failures
- **Idempotent processing**: Job ID prevents duplicate data on reruns
- **Streaming reads**: never loads entire file into memory

### Project Structure

```
/project-root
├── functions/
│   ├── upload-url-generator/index.js
│   └── gcs-trigger/index.js
├── jobs/
│   └── file-processor/
│       ├── Dockerfile
│       ├── index.js
│       ├── extractor.js
│       └── package.json
├── infrastructure/
│   ├── pubsub.tf
│   └── cloudrun-job.yaml
└── package.json
```

---

## Choosing the Right Service

- **Cloud Functions**: event triggers, lightweight APIs, webhooks
- **Cloud Run**: containerized APIs, moderate workloads
- **Cloud Run Jobs**: batch processing up to 24 hours
- **Kubernetes**: long-running, stateful, large file workloads

### Cloud Functions vs Cloud Run vs Cloud Run Jobs

- **Cloud Functions**: max 60 min, max 32 GB, up to 1000 concurrent/instance, source-only (no container), scales 0→N, per-invocation pricing
- **Cloud Run**: max 60 min, max 32 GB, up to 1000 concurrent/instance, container support, scales 0→N, per-request pricing
- **Cloud Run Jobs**: max 24 hours, max 32 GB, batch (fixed task count), container support, manual/scheduled trigger, per-task pricing

### For Large File Upload Use Case

- **Generate upload URLs** → Cloud Functions (quick, stateless)
- **GCS upload trigger** → Cloud Functions (simple event handling)
- **File processing (>1 GB)** → Cloud Run Jobs (long timeout, checkpointing, retries)
- **Status API** → Cloud Functions or Cloud Run (lightweight)

---

## Solving Cold Starts

Cold starts happen when a new instance must be initialized (load runtime, dependencies, establish connections).

### Strategies (most to least effective)

- **Min instances = 1**: eliminates ~90–95% of cold starts, costs ~$10–15/month
- **Scheduled warm-up requests** (every 5 min via Cloud Scheduler): eliminates ~80–90%, costs ~$1/month
- **Use Gen 2 functions**: ~20–50% reduction, same cost
- **Optimize imports** (import only what you need, not full SDKs): ~20–40% reduction, free
- **Lazy loading** (require modules on first use): ~10–30% reduction, free

---

## Kubernetes vs Serverless

### When Kubernetes is Better

- No cold starts (pods always running)
- Stream large files directly through pods (no size limit)
- No timeout limits (can process for hours)
- Full networking control (VPC, service mesh)
- GPU/special hardware support
- Stateful processing with persistent volumes
- Easier debugging (kubectl logs, exec into pods)

### When Serverless is Better

- Sporadic/unpredictable traffic (pay nothing when idle)
- Instant scaling to thousands of instances
- Simple event handlers (no infra to manage)
- Prototyping / MVP (faster time to market)
- Cost-sensitive low-traffic apps ($0 base cost)

---

## Cost Reference

### Cloud Functions Pricing

- Invocations: $0.40 per million
- CPU: $0.000018 per GHz-second
- Memory: $0.0000025 per GB-second
- Egress: $0.12 per GB
- Free tier: 2M invocations, 400K GB-seconds/month

### Kubernetes (GKE) Pricing

- Cluster management (Standard): ~$73/month
- Nodes: ~$24/month (e2-medium) to ~$98/month (e2-standard-4)
- Load balancer: ~$18/month
- Autopilot: per-pod resource usage, no cluster fee

### Break-Even Point

- Below ~300K requests/day → Cloud Functions is cheaper
- Above ~300K requests/day → Kubernetes is cheaper
- Large file processing (>100 files/day at 1 GB each) → Kubernetes is cheaper

### Hidden Costs

- **Cloud Functions**: min instances ($10–30/month), egress ($0.12/GB), VPC connector (~$7/month), surprise spikes
- **Kubernetes**: DevOps time, over-provisioned idle capacity, load balancer (~$18/month), persistent disks, external monitoring

### Cost Tips

- **Cloud Functions**: use minimum memory needed, set appropriate (not max) timeout, use Gen 2, avoid min-instances unless critical
- **Kubernetes**: use Autopilot (no cluster fee), use spot/preemptible nodes for batch (60–91% cheaper), right-size pod requests, use HPA to scale down

---

## Load Balancer with Cloud Functions

### When You Need It

- Custom domain with SSL
- Multiple functions behind one domain
- DDoS protection (Cloud Armor)
- CDN/caching
- Path-based routing

### When You Don't

- Simple API using the default Cloud Functions URL

### Cost

- Load balancer forwarding rule: ~$18/month
- Managed SSL certificate: free
- Cloud CDN: per GB egress
- Cloud Armor: ~$5/month + per request

### Components

- **Load Balancer**: entry point with static IP and SSL termination
- **URL Map**: routes requests by path/host rules
- **Backend Service**: manages health checks, CDN, Cloud Armor policies
- **Serverless NEG**: Network Endpoint Group pointing to Cloud Functions
