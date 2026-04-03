# S3-like Object Storage System Design

## System Overview
A distributed object storage service (think AWS S3 / Google Cloud Storage) that stores arbitrary files (objects) in buckets, provides high durability and availability, and serves objects at scale via a simple HTTP API.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Create and delete buckets
- Upload, download, and delete objects (files of any size)
- List objects in a bucket
- Object versioning
- Access control (public / private / IAM policies)
- Pre-signed URLs for temporary access
- Multipart upload for large files

### Non-Functional Requirements
- Durability: 99.999999999% (11 nines) — data must never be lost
- Availability: 99.99%
- Latency: <100ms for small object reads; throughput-optimized for large files
- Scalability: Exabytes of storage, billions of objects
- Consistency: Strong read-after-write consistency

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1B objects stored
- 100M uploads/day, 1B downloads/day
- Average object size: 1MB
- Read:Write ratio = 10:1

### Traffic
```
Uploads/sec     = 100M / 86400 ≈ 1160/sec
Downloads/sec   = 1B / 86400   ≈ 11.6K/sec
Peak (3×)       ≈ 35K downloads/sec

Bandwidth       = 35K × 1MB = 35GB/sec → CDN + distributed storage nodes
```

### Storage
```
Total objects   = 1B × 1MB = 1PB
With 3× replication = 3PB
Growth/day      = 100M × 1MB = 100TB/day
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing; validates bucket/object access permissions

**Metadata Service** — Stores object metadata (bucket, key, size, checksum, location, version); does NOT store object data; backed by a distributed metadata DB

**Storage Service** — Manages actual object data storage across storage nodes; handles chunking, replication, and retrieval

**Storage Nodes** — Physical/virtual machines with large disks; store object chunks; replicated across availability zones

**Chunk Manager** — Splits large objects into fixed-size chunks (e.g., 64MB); tracks which chunks belong to which object; manages replication

**Replication Service** — Ensures each chunk is replicated to N nodes (typically 3) across different AZs; handles re-replication on node failure

**Garbage Collection Service** — Cleans up deleted objects and orphaned chunks asynchronously

**Access Control Service** — Evaluates bucket policies, IAM permissions, pre-signed URL validity

**Metadata DB (Cassandra)** — Object metadata: bucket, key, version, chunk locations, checksum, size

**Bucket DB (PostgreSQL)** — Bucket metadata, ownership, policies, versioning config

**Redis** — Pre-signed URL cache, session store, rate limiting

**CDN** — Caches frequently accessed public objects at edge

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| Cassandra (Metadata DB) | Billions of object metadata records, high read/write throughput, partition by bucket+key |
| PostgreSQL (Bucket DB) | Small number of buckets, ACID for policy/config changes |
| Redis | Pre-signed URL validation, hot metadata cache |
| Raw disk (Storage Nodes) | Actual object data — custom storage engine, not a DB |

### Cassandra — object_metadata

Partition key: `bucket_id`, Clustering: `object_key`

| Field | Type |
|---|---|
| bucket_id | UUID (partition key) |
| object_key | VARCHAR (clustering) |
| version_id | UUID |
| size_bytes | BIGINT |
| checksum_md5 | VARCHAR |
| content_type | VARCHAR |
| chunk_ids | LIST\<UUID\> |
| storage_class | VARCHAR (standard / infrequent / archive) |
| created_at | TIMESTAMP |
| deleted_at | TIMESTAMP, nullable |

### PostgreSQL — buckets

| Field | Type |
|---|---|
| bucket_id | UUID (PK) |
| owner_id | UUID |
| bucket_name | VARCHAR, unique |
| region | VARCHAR |
| versioning_enabled | BOOLEAN |
| access_policy | JSONB |
| created_at | TIMESTAMP |

### Storage Node — chunk storage

Each chunk stored as a flat file on disk:
- Filename: `{chunkId}.dat`
- Replicated to 3 nodes across different AZs
- Chunk size: 64MB (configurable)
- Checksum stored alongside for integrity verification

## 5. Key Flows

### 5.1 Object Upload

**Small object (<5MB, single-part):**
1. Client `PUT /bucket/key` → API Gateway → Access Control (permission check)
2. Metadata Service generates `objectId`, `versionId`
3. Storage Service receives data, splits into chunks (if >64MB)
4. Chunk Manager assigns chunk to 3 storage nodes (across AZs)
5. Data written to primary node, replicated to 2 replicas
6. Checksum computed and verified
7. Metadata Service writes object metadata to Cassandra (chunk locations, checksum, size)
8. Return 200 OK with `ETag` (checksum)

**Large object (multipart upload):**
1. Client initiates: `POST /bucket/key?uploads` → returns `uploadId`
2. Client uploads parts in parallel: `PUT /bucket/key?partNumber=N&uploadId=X`
3. Each part stored as chunks independently
4. Client completes: `POST /bucket/key?uploadId=X` with part list
5. Storage Service assembles manifest; Metadata Service writes final object record
6. Parts cleaned up after assembly

### 5.2 Object Download

1. Client `GET /bucket/key` → API Gateway → Access Control
2. Metadata Service fetches object metadata from Cassandra (chunk IDs + node locations)
3. Storage Service fetches chunks from storage nodes in parallel
4. Reassembles chunks in order, streams to client
5. Checksum verified on reassembly

**CDN path for public objects:**
- CDN edge serves cached copy
- On miss: CDN pulls from Storage Service, caches at edge

### 5.3 Pre-signed URLs

1. Client requests pre-signed URL: `GET /presign?bucket=X&key=Y&expiry=3600`
2. Access Control Service generates signed URL: `HMAC(bucket + key + expiry + secret)`
3. URL valid for specified duration
4. Anyone with URL can access object without auth (within expiry)
5. On access: API Gateway validates signature and expiry

### 5.4 Replication & Durability

- Each chunk replicated to 3 nodes across 3 AZs
- Write quorum: 2/3 nodes must acknowledge before returning success
- Read: fetch from nearest available node
- On node failure: Replication Service detects (heartbeat), re-replicates affected chunks to a new node
- Cross-region replication: optional, async copy to another region for disaster recovery

## 6. Key Interview Concepts

### Why 11 Nines Durability
11 nines = 0.000000001% annual data loss probability. Achieved through:
- 3× replication across AZs
- Checksums on every chunk (detect corruption)
- Erasure coding for archive storage (more space-efficient than 3× replication)
- Cross-region replication for disaster recovery

### Erasure Coding vs Replication
- Replication (3×): store 3 full copies. Simple, fast reads. 200% storage overhead.
- Erasure coding (e.g., 6+3): split object into 6 data chunks + 3 parity chunks. Can reconstruct from any 6 of 9. 50% storage overhead. More CPU for encode/decode. Used for infrequently accessed (cold) storage.

### Consistent Hashing for Storage Nodes
Chunk Manager uses consistent hashing to assign chunks to storage nodes. On node addition/removal, only chunks on the affected hash range need to move — minimizes data movement during scaling.

### Metadata vs Data Separation
Metadata (bucket, key, size, chunk locations) is tiny and query-heavy — stored in Cassandra. Actual data is large and sequential — stored on raw disk. This separation allows metadata to scale independently and be queried fast without touching data.

### Multipart Upload
For files >5GB, single-part upload is impractical (network interruption = restart from scratch). Multipart allows parallel upload of parts, resume on failure, and efficient large file handling. Each part is independently checksummed.

## 7. Failure Scenarios

### Storage Node Failure
- Detection: heartbeat timeout
- Recovery: Replication Service identifies chunks with <3 replicas, re-replicates to healthy nodes
- Data available from remaining 2 replicas during recovery (read quorum = 1)

### Metadata DB (Cassandra) Failure
- Impact: object lookups fail; uploads fail
- Recovery: RF=3, QUORUM reads/writes continue on remaining nodes
- Prevention: multi-datacenter replication

### Chunk Corruption
- Detection: checksum mismatch on read
- Recovery: fetch chunk from another replica; re-replicate from healthy copy
- Prevention: periodic background scrubbing — read all chunks, verify checksums

### Partial Upload Failure
- Recovery: multipart upload allows resume from last successful part
- Incomplete multipart uploads cleaned up by GC after 7 days
