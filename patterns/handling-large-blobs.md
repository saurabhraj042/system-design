# Handling Large Blobs

## The Core Idea

Files bigger than ~10MB don't belong in databases, and they shouldn't travel through your application servers either. Databases choke on binary blobs, and routing gigabytes through your API instances turns every upload or download into a bottleneck. The fix is to get your servers out of the data path entirely: servers hand out **temporary permission tokens**, and clients use those tokens to talk directly to cloud object storage or a CDN.

## Mental Model / Analogy

Your server is a nightclub bouncer. The bouncer doesn't personally escort every guest to the bar — that would create a massive queue at the door. Instead, the bouncer checks IDs, stamps wrists, and lets people walk in themselves. Same idea: your server validates identity and hands out a time-limited, scoped "wristband" (a presigned URL). From then on the client deals directly with storage.

---

## Why Object Storage, Not Databases?

Relational databases optimize for structured queries, small rows, and transactional consistency. A 200MB file stored as a blob:
- Kills query performance for every query on that table
- Inflates backup sizes and replication lag
- Eats memory buffers meant for indexes

Object stores (S3, GCS, Azure Blob) are designed for the opposite: unlimited capacity, streaming transfers, geo-replication, and per-byte pricing. General rule: anything over 10MB that doesn't need a SQL `WHERE` clause belongs in object storage.

---

## How It Works

### Upload: Presigned URLs

Instead of streaming the file through your server:

1. Client asks your server for upload permission
2. Server validates the request (auth, quota checks) and generates a **presigned URL** — a time-limited, cryptographically signed URL that lets the client PUT directly to your bucket
3. Server stores a `pending` metadata record in the database and returns the presigned URL + a file ID
4. Client PUTs the file bytes directly to the presigned URL (never touches your servers)
5. Object storage confirms the upload and fires an event notification

The signature is computed entirely in your server's memory using your cloud credentials — no network call to storage needed to generate it. You can embed constraints in the signature:
- Max file size (prevent someone uploading 50GB on a URL meant for a 5MB image)
- Allowed content type (ensure a "profile photo" upload can't receive an executable)

```
Client          Server           Object Storage
  |                |                    |
  |--POST /upload->|                    |
  |                |--generate presign--|
  |<--presign URL--|                    |
  |                                     |
  |--------PUT file bytes-------------->|
  |<--------201 Created-----------------|
  |                                     |
  |        [storage fires event]        |
  |                |<---ObjectCreated---|
  |                |--update DB: done---|
```

### Download: CDN + Signed URLs

For downloads, clients should pull from a **CDN**, not directly from origin storage. The CDN caches popular files at edge locations worldwide, so a user in Seoul isn't fetching from a Virginia S3 bucket on every request.

Two types of signed URLs exist:
- **Storage presigned URLs** (S3, GCS): validated by the storage service using your cloud credentials
- **CDN signed URLs** (CloudFront, Cloud CDN): validated at CDN edge nodes using public/private key crypto — no round-trip to origin needed

Use CDN signed URLs for anything accessed frequently; use direct storage URLs only for rare, one-time downloads.

### Resumable / Chunked Uploads for Large Files

A single 3GB upload over a flaky mobile connection failing at 95% means starting over — unacceptable. Cloud providers solve this with **multipart/chunked upload APIs**:

- **AWS S3**: multipart upload — each 5MB+ chunk gets its own presigned URL
- **GCS / Azure**: resumable session — a single session URL accepts `Range`-headed PUT requests

How it works:
1. Client initiates a multipart session, gets a session ID
2. Client splits the file into chunks (typically 5–10MB each), uploads them in parallel or sequentially
3. Each chunk returns a checksum/ETag
4. On connection drop, client queries which chunks already landed and resumes from the first missing one
5. After all chunks land, client calls the **complete** endpoint — storage assembles them into the final object

Progress tracking is free: completed_chunks / total_chunks × 100 = progress percentage.

> Remember to set lifecycle rules to clean up incomplete multipart uploads after 24–48 hours — partially uploaded chunks still cost money.

---

## State Synchronization: The Hard Part

Your database metadata and object storage now update at different times. Naively trusting the client to call "upload complete!" creates problems:
- Race conditions (DB shows "complete" before storage has the bytes)
- Orphaned files (client crashes after upload, before notifying you)
- Malicious clients (mark complete without uploading anything)
- Lost notifications (network failure drops the completion call)

### Solution: Event Notifications + Reconciliation

Most object stores can push events to your system when a file actually lands. S3 fires to SNS/SQS/Lambda, GCS publishes to Pub/Sub, Azure uses Event Grid. Your server listens for these events and updates the database only when storage itself confirms the file exists.

```
File arrives in S3
  → S3 fires ObjectCreated event
  → Your event handler receives it
  → Looks up the file record by storage key
  → Updates status: 'pending' → 'complete'
```

Events can still be lost (network hiccups, service outages). Add a **reconciliation job** that periodically scans for records stuck in `pending` status older than some threshold, checks whether the file actually exists in storage, and updates accordingly.

---

## Cloud Provider Quick Reference

| Concept | AWS | GCS | Azure |
|---|---|---|---|
| Direct upload | Presigned URL (PUT) | Signed URL | SAS token |
| Large file upload | Multipart Upload (5MB+ parts) | Resumable Upload | Block Blobs |
| Events on arrival | S3 → SNS/SQS/Lambda | Pub/Sub | Event Grid |
| CDN with auth | CloudFront signed URLs | Cloud CDN signed URLs | Azure CDN + SAS |
| Incomplete upload cleanup | Lifecycle Rules | Lifecycle Management | Lifecycle Policies |

---

## When to Use

Use this pattern when:
- Files exceed ~10MB (the threshold where server proxying becomes a real bottleneck)
- Your system handles media (videos, images, documents, backups)
- Uploads need to be resumable (mobile users, large files, unreliable connections)
- Downloads should be fast globally (CDN caching gives edge-local latency)

**Do NOT use when:**
- Files are small (< 10MB) — the extra round trip for a presigned URL adds more latency than it saves
- You need to validate content in real-time as bytes arrive (e.g., streaming CSV validation) — you must proxy those
- Compliance requires all data to pass through certified systems before storage (healthcare, some finance) — proxy through your servers, but still do it in chunks

---

## Common Gotchas

- **Never let clients choose their own storage keys** — they'll overwrite each other and create path-traversal risks. Generate keys server-side: `uploads/{user_id}/{uuid}/{filename}`
- **Incomplete multipart uploads cost money** — lifecycle rules are not optional
- **Signed URL expiry vs. upload duration** — a 15-minute presigned URL fails if a large upload takes 20 minutes. Set expiry to comfortably exceed the expected upload time
- **Metadata lives in your DB, not in object tags** — S3 tags max out at 10 tags × 256 chars. Use the `storage_key` as the join column between your metadata DB and the file in storage
- **The file doesn't exist until `CompleteMultipartUpload` succeeds** — your DB record can show `complete` while storage is still assembling parts if you update status on the last chunk rather than the completion response

---

## Practice Questions

1. A user starts uploading a 4GB video and their connection drops at 60%. How does your system know where to resume, and what does the client need to store locally to make this work?
2. A user uploads a profile photo. Describe the full flow — from the client hitting "Save" to the photo appearing in their profile — using presigned URLs and event notifications.
3. Your system generates presigned upload URLs with no content-length restriction. A malicious user uploads a 500GB file using a URL meant for a 2MB avatar. How do you prevent this?
4. When would you route a file upload through your application server instead of using direct-to-storage uploads? Give two concrete scenarios.
5. Explain the difference between how S3 presigned URLs and CloudFront signed URLs are validated. Which would you use for a frequently accessed resource and why?

---

## One-Line Summary

Get your servers out of the bytes business — hand out time-limited, scoped tokens, let clients talk directly to object storage for uploads and CDN for downloads, then reconcile state via storage events.
