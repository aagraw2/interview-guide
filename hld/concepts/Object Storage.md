## 1. What is Object Storage?

Object storage is a data storage architecture that manages data as objects rather than files (file storage) or blocks (block storage). Each object contains the data, metadata, and a unique identifier.

```
Examples:
  - Amazon S3
  - Google Cloud Storage
  - Azure Blob Storage
  - MinIO (self-hosted)
  - Cloudflare R2
```

**Core use case:** Store and retrieve large amounts of unstructured data (images, videos, backups, logs, data lakes).

---

## 2. Storage Architectures Comparison

### Block storage

```
Raw storage blocks (like a hard drive)
  - Used by: EBS, SAN
  - Access: Direct block-level access
  - Use case: Database storage, VM disks
  - Performance: Very fast (low latency)
  - Cost: Expensive
```

### File storage

```
Hierarchical file system (directories and files)
  - Used by: NFS, EFS
  - Access: File paths (/home/user/file.txt)
  - Use case: Shared file systems
  - Performance: Fast (network latency)
  - Cost: Moderate
```

### Object storage

```
Flat namespace (no directories)
  - Used by: S3, GCS, Azure Blob
  - Access: HTTP API (GET, PUT, DELETE)
  - Use case: Unstructured data, backups, data lakes
  - Performance: Moderate (HTTP latency)
  - Cost: Very cheap
```

---

## 3. Object Storage Data Model

### Object structure

```
Object:
  - Key (unique identifier): "images/2024/01/photo.jpg"
  - Data (binary blob): [image bytes]
  - Metadata:
      - System metadata: size, last-modified, content-type
      - User metadata: custom key-value pairs
  - Version ID (if versioning enabled)
```

### Flat namespace

Unlike file systems, object storage has no true directories.

```
File system:
  /images/
    /2024/
      /01/
        photo.jpg

Object storage:
  Key: "images/2024/01/photo.jpg"
  (The "/" is just part of the key name, not a directory)

Listing objects with prefix "images/2024/01/" gives the illusion of directories
```

---

## 4. S3 API (Industry Standard)

Most object storage systems implement the S3 API.

### Basic operations

```
PUT /bucket/key
  Upload an object

GET /bucket/key
  Download an object

DELETE /bucket/key
  Delete an object

HEAD /bucket/key
  Get object metadata (without downloading data)

LIST /bucket?prefix=images/
  List objects with a given prefix
```

### Multipart upload

For large files (>100 MB), split into parts and upload in parallel.

```
1. Initiate multipart upload
   POST /bucket/key?uploads
   → Returns upload_id

2. Upload parts (in parallel)
   PUT /bucket/key?partNumber=1&uploadId=xyz
   PUT /bucket/key?partNumber=2&uploadId=xyz
   PUT /bucket/key?partNumber=3&uploadId=xyz

3. Complete multipart upload
   POST /bucket/key?uploadId=xyz
   → Assemble parts into final object

Benefits:
  - Parallel uploads (faster)
  - Resume on failure (re-upload only failed parts)
  - Upload files larger than 5 GB (S3 limit for single PUT)
```

### Presigned URLs

Generate a temporary URL that grants access to a private object.

```
Server:
  url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'photo.jpg'},
    ExpiresIn=3600  # 1 hour
  )
  → Returns: https://s3.amazonaws.com/my-bucket/photo.jpg?X-Amz-Signature=...

Client:
  GET https://s3.amazonaws.com/my-bucket/photo.jpg?X-Amz-Signature=...
  → Downloads photo.jpg (no authentication needed)

Use case:
  - Direct upload/download from client (bypass server)
  - Share private files temporarily
  - Reduce server bandwidth
```

---

## 5. Consistency Model

### S3 consistency (as of Dec 2020)

```
Strong consistency for all operations:
  - PUT → GET: Immediately see new object
  - DELETE → GET: Immediately see 404
  - PUT (overwrite) → GET: Immediately see new version

Before Dec 2020:
  - Eventual consistency for overwrites and deletes
  - Could read stale data for a few seconds
```

### Implications

```
Write object:
  PUT /bucket/key → 200 OK

Immediately read:
  GET /bucket/key → Returns the object (guaranteed)

No need to wait or retry for consistency
```

---

## 6. Durability and Availability

### S3 Standard

```
Durability: 99.999999999% (11 nines)
  - Probability of losing an object: 0.000000001% per year
  - If you store 10 million objects, expect to lose 1 object every 10,000 years

Availability: 99.99%
  - Uptime: 99.99% (52 minutes downtime per year)

How it's achieved:
  - Replicate across multiple availability zones (AZs)
  - Erasure coding (like RAID, but distributed)
  - Automatic healing (detect and repair corrupted data)
```

### Storage classes

```
S3 Standard:
  - Frequent access
  - High availability (99.99%)
  - Cost: $0.023/GB/month

S3 Infrequent Access (IA):
  - Infrequent access (once a month)
  - Lower storage cost, higher retrieval cost
  - Cost: $0.0125/GB/month + $0.01/GB retrieval

S3 Glacier:
  - Archive storage (rarely accessed)
  - Retrieval time: minutes to hours
  - Cost: $0.004/GB/month + retrieval cost

S3 Glacier Deep Archive:
  - Long-term archive (once a year)
  - Retrieval time: 12 hours
  - Cost: $0.00099/GB/month + retrieval cost
```

---

## 7. Versioning

Keep multiple versions of an object.

```
Enable versioning on bucket:
  PUT /bucket?versioning
  <VersioningConfiguration>
    <Status>Enabled</Status>
  </VersioningConfiguration>

Upload object:
  PUT /bucket/key (version 1) → version_id: v1
  PUT /bucket/key (version 2) → version_id: v2
  PUT /bucket/key (version 3) → version_id: v3

Get latest version:
  GET /bucket/key → Returns version 3

Get specific version:
  GET /bucket/key?versionId=v1 → Returns version 1

Delete object:
  DELETE /bucket/key → Adds a delete marker (version 4)
  GET /bucket/key → 404 (but versions 1-3 still exist)

Permanently delete:
  DELETE /bucket/key?versionId=v1 → Deletes version 1
```

**Use case:** Protect against accidental deletion, audit trail, rollback.

---

## 8. Lifecycle Policies

Automatically transition or delete objects based on age.

```
Lifecycle rule:
  - After 30 days: Transition to S3 IA
  - After 90 days: Transition to S3 Glacier
  - After 365 days: Delete

Example:
  Day 0: Upload log file to S3 Standard
  Day 30: Automatically moved to S3 IA
  Day 90: Automatically moved to S3 Glacier
  Day 365: Automatically deleted

Cost savings:
  S3 Standard (30 days): $0.023/GB × 30/30 = $0.023/GB
  S3 IA (60 days): $0.0125/GB × 60/30 = $0.025/GB
  S3 Glacier (275 days): $0.004/GB × 275/30 = $0.037/GB
  Total: $0.085/GB (vs $0.276/GB for S3 Standard all year)
```

---

## 9. Access Control

### Bucket policies

JSON-based access control for the entire bucket.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/public/*"
    }
  ]
}

Allows anyone to read objects under "public/" prefix
```

### IAM policies

Control access for AWS users and roles.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}

Allows user to read and write objects in "my-bucket"
```

### Access Control Lists (ACLs)

Legacy method (use bucket policies instead).

```
Object ACL:
  - Owner: full control
  - Public: read access
  - Authenticated users: read access
```

---

## 10. Performance Optimization

### Request rate limits (S3)

```
Old limits (before 2018):
  - 100 PUT/LIST/DELETE requests per second per prefix
  - 300 GET requests per second per prefix

Current limits (after 2018):
  - 3,500 PUT/COPY/POST/DELETE requests per second per prefix
  - 5,500 GET/HEAD requests per second per prefix

Prefix:
  my-bucket/images/2024/01/ → prefix = "images/2024/01/"

To scale beyond limits:
  - Use multiple prefixes (shard by user_id, date, etc.)
  - Example: user-123/file.jpg, user-456/file.jpg
```

### Transfer acceleration

Use CloudFront edge locations to speed up uploads.

```
Normal upload:
  Client (Tokyo) → S3 (us-east-1)
  Latency: 200ms per request

Transfer acceleration:
  Client (Tokyo) → CloudFront (Tokyo) → S3 (us-east-1)
  Latency: 20ms to CloudFront + fast AWS backbone to S3
  Total: ~50ms (4x faster)

Enable:
  PUT /bucket?accelerate
  <AccelerateConfiguration>
    <Status>Enabled</Status>
  </AccelerateConfiguration>

Upload to:
  https://my-bucket.s3-accelerate.amazonaws.com/key
```

### Multipart upload

Already covered above — use for files >100 MB.

---

## 11. Object Storage Internals

### Metadata separation

```
Object = Metadata + Data

Metadata:
  - Stored in distributed key-value store (e.g., DynamoDB)
  - Key: object key
  - Value: {size, location, checksum, metadata}

Data:
  - Stored in distributed file system
  - Split into chunks (e.g., 64 MB chunks)
  - Replicated across nodes

Lookup:
  1. Query metadata store for object key
  2. Get chunk locations
  3. Read chunks from storage nodes
  4. Assemble and return object
```

### Erasure coding

Like RAID, but distributed across nodes.

```
Erasure coding (6+3):
  - Split object into 6 data chunks
  - Generate 3 parity chunks
  - Store 9 chunks across 9 nodes
  - Can lose any 3 chunks and still reconstruct the object

Storage overhead:
  9 chunks / 6 data chunks = 1.5x (vs 3x for 3-way replication)

Durability:
  Can tolerate 3 node failures (vs 2 for 3-way replication)

Trade-off:
  - Lower storage cost (1.5x vs 3x)
  - Higher CPU cost (encoding/decoding)
```

### Consistent hashing

Distribute objects across storage nodes.

```
Hash(object_key) → node

Example:
  Hash("images/photo.jpg") → node 5
  Store chunks on nodes 5, 6, 7 (with replication)

When nodes are added/removed:
  - Only a fraction of objects need to be moved
  - Consistent hashing minimizes remapping
```

---

## 12. Common Use Cases

### 1. Static website hosting

```
S3 bucket:
  - Enable static website hosting
  - Upload HTML, CSS, JS files
  - Set bucket policy to allow public read

Access:
  http://my-bucket.s3-website-us-east-1.amazonaws.com

Use case: Landing pages, documentation, SPAs
```

### 2. Data lake

```
Store raw data in S3:
  - Logs, events, sensor data
  - Parquet, Avro, JSON files

Query with:
  - Athena (SQL queries on S3)
  - Spark (distributed processing)
  - Redshift Spectrum (query from data warehouse)

Use case: Analytics, machine learning, data science
```

### 3. Backup and disaster recovery

```
Backup database to S3:
  - Daily snapshots
  - Lifecycle policy: Glacier after 30 days
  - Cross-region replication for disaster recovery

Use case: Database backups, file backups, VM snapshots
```

### 4. Media storage

```
Store images, videos, audio:
  - Upload to S3
  - Serve via CloudFront (CDN)
  - Presigned URLs for private content

Use case: User-generated content, video streaming, photo sharing
```

---

## 13. Common Interview Questions + Answers

### Q: What's the difference between object storage and file storage?

> "Object storage uses a flat namespace with HTTP API access, optimized for unstructured data like images and backups. File storage uses a hierarchical file system with file paths, optimized for shared file access. Object storage is much cheaper and scales to petabytes, but has higher latency. File storage is faster but more expensive. Use object storage for static assets and backups, file storage for shared file systems."

### Q: How does S3 achieve 11 nines of durability?

> "S3 replicates data across multiple availability zones and uses erasure coding to tolerate multiple node failures. It continuously monitors for data corruption and automatically repairs damaged data. With erasure coding like 6+3, you can lose any 3 chunks and still reconstruct the object. This makes the probability of data loss extremely low — 11 nines means you'd expect to lose 1 object out of 10 million every 10,000 years."

### Q: What are presigned URLs and when would you use them?

> "Presigned URLs are temporary URLs that grant access to private S3 objects without requiring authentication. The server generates a URL with a signature and expiration time, and the client can use it to upload or download directly from S3. This is useful for direct client uploads (bypassing the server), sharing private files temporarily, or reducing server bandwidth. The URL expires after the specified time, so access is time-limited."

### Q: How would you optimize S3 performance for high request rates?

> "Use multiple prefixes to shard objects — S3 supports 3,500 PUT and 5,500 GET requests per second per prefix. For example, shard by user_id or date. Use multipart upload for large files to parallelize uploads. Enable transfer acceleration to route uploads through CloudFront edge locations. Use CloudFront as a CDN to cache frequently accessed objects and reduce S3 requests."

---

## 14. Interview Tricks & Pitfalls

### ✅ Trick 1: Know the consistency model

S3 has strong consistency (as of Dec 2020). Mentioning this shows you're up-to-date with recent changes.

### ✅ Trick 2: Explain erasure coding

Erasure coding is how S3 achieves high durability with lower storage overhead than replication. Mentioning it shows you understand the internals.

### ✅ Trick 3: Discuss presigned URLs

Presigned URLs are a common pattern for direct client uploads. Mentioning them shows you understand practical use cases.

### ❌ Pitfall 1: Confusing object storage with file storage

Object storage has a flat namespace and HTTP API. File storage has directories and file paths. Don't mix them up.

### ❌ Pitfall 2: Thinking S3 is a database

S3 is for storing blobs, not structured data with queries. You can't run SQL queries on S3 directly (use Athena for that).

### ❌ Pitfall 3: Forgetting about storage classes

S3 has multiple storage classes (Standard, IA, Glacier) with different costs and retrieval times. Mentioning lifecycle policies shows you understand cost optimization.

---

## 15. Quick Reference

```
What is object storage?
  Store data as objects (data + metadata + unique ID)
  Flat namespace, HTTP API access
  Examples: S3, GCS, Azure Blob, MinIO

Object structure:
  Key: "images/2024/01/photo.jpg"
  Data: [binary blob]
  Metadata: {size, content-type, custom key-value pairs}

S3 API:
  PUT /bucket/key → Upload object
  GET /bucket/key → Download object
  DELETE /bucket/key → Delete object
  LIST /bucket?prefix=images/ → List objects

Multipart upload:
  For large files (>100 MB)
  Upload parts in parallel
  Resume on failure

Presigned URLs:
  Temporary URL for private objects
  Direct client upload/download (bypass server)
  Expires after specified time

Consistency:
  S3: Strong consistency (as of Dec 2020)
  PUT → GET: Immediately see new object

Durability and availability:
  S3 Standard: 99.999999999% durability, 99.99% availability
  Replicate across multiple AZs
  Erasure coding (6+3: 1.5x overhead, tolerate 3 failures)

Storage classes:
  S3 Standard: Frequent access ($0.023/GB/month)
  S3 IA: Infrequent access ($0.0125/GB/month)
  S3 Glacier: Archive ($0.004/GB/month)
  S3 Glacier Deep Archive: Long-term ($0.00099/GB/month)

Versioning:
  Keep multiple versions of an object
  Protect against accidental deletion
  Rollback to previous version

Lifecycle policies:
  Automatically transition or delete objects
  Example: Standard (30d) → IA (90d) → Glacier (365d) → Delete

Performance:
  3,500 PUT/sec per prefix
  5,500 GET/sec per prefix
  Use multiple prefixes to scale
  Transfer acceleration: Use CloudFront edge locations

Internals:
  Metadata: Distributed key-value store
  Data: Distributed file system (chunks)
  Erasure coding: 1.5x overhead vs 3x for replication
  Consistent hashing: Distribute objects across nodes

Use cases:
  Static website hosting
  Data lake (analytics, ML)
  Backup and disaster recovery
  Media storage (images, videos)

Block vs File vs Object:
  Block: Raw blocks, very fast, expensive (EBS, SAN)
  File: Hierarchical, fast, moderate cost (NFS, EFS)
  Object: Flat namespace, moderate speed, cheap (S3, GCS)
```
