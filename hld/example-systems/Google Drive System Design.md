# Google Drive System Design

## System Overview
A cloud file storage and sync service (think Google Drive / Dropbox / OneDrive) where users upload, store, organize, and share files — with chunked parallel uploads, real-time sync across devices, collaborative access, and version history.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Upload, download, and delete files and folders
- Organize files in folder hierarchy
- File sync across multiple devices (desktop, mobile, web)
- Share files/folders with other users (view / edit permissions)
- Version history — restore previous versions
- Search files by name and content
- Offline access with sync on reconnect

### Non-Functional Requirements
- Availability: 99.99%
- Latency: <200ms for metadata operations; throughput-optimized for file transfers
- Scalability: 1B+ users, exabytes of storage
- Durability: 99.999999999% (11 nines) — files must never be lost
- Consistency: Strong read-after-write for file metadata; eventual for sync

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1B users, 200M DAU
- Average storage per user: 15GB
- 10M file uploads/day, average file size: 5MB
- 100M file downloads/day
- Read:Write ratio = 10:1

### Traffic
```
Uploads/sec         = 10M / 86400 ≈ 116/sec
Downloads/sec       = 100M / 86400 ≈ 1157/sec → CDN

Upload bandwidth    = 116 × 5MB = 580MB/sec
Download bandwidth  = 1157 × 5MB = 5.8GB/sec → CDN
```

### Storage
```
Total storage       = 1B × 15GB = 15EB
Uploads/day         = 10M × 5MB = 50TB/day
With dedup (30%)    = 35TB/day net new
With 3× replication = 105TB/day
```

## 3. Client-Side Architecture

The desktop/mobile client has four internal components that work together:

**Watcher** — monitors the local file system for changes (new files, modifications, deletions); triggers the upload pipeline on change

**Local Metadata Index** — local DB of file state (name, size, last modified, chunk hashes); used to detect what changed and avoid re-uploading unchanged chunks

**Chunker** — splits files into fixed-size chunks (configurable, e.g., 5MB); computes SHA256 of each chunk; enables parallel upload and delta sync

**Upload Manager** — manages the upload state machine; tracks which chunks are uploaded; handles retries; calls File Uploader Service APIs

**Sync Engine** — handles bidirectional sync:
- Pull: on reconnect, fetches changes since last sync cursor from server
- Push (Fanout): receives server-side change notifications and applies locally

## 4. Core Server Components

**LB + API Gateway** — Auth, rate limiting, routing

**User Onboarding Service** — Registration, login, JWT issuance; writes to User DB

**File Uploader Service** — Orchestrates the multi-step upload protocol; validates permissions; generates pre-signed S3 URLs per chunk; on commit, calls File Metadata Service and publishes to Kafka

**Validator Service** — Validates upload permissions before generating signed URLs; checks quota, file type restrictions, ownership

**File Metadata Service** — Stores and manages file/folder metadata, versions, chunk mappings; writes to MetaDB; publishes change events to Kafka for Sync Service

**Read/Download Service** — Handles file download requests; validates permissions; generates pre-signed S3 URLs for download; serves via CDN for public/cached files

**Sync Service** — Consumes Kafka file change events; pushes delta updates to connected devices (WebSocket/long-poll); handles pull requests from reconnecting devices using sync cursor

**Share Service** — Manages sharing permissions; generates share link tokens; access control

**Search Service** — File name and content search; Elasticsearch; CDC from MetaDB

**S3** — Stores actual chunk data; content-addressed by chunk hash (`s3://bucket/{chunkHash}`)

**CDN** — Caches frequently downloaded files at edge

**MetaDB (PostgreSQL)** — All file/folder metadata, versions, chunk mappings, permissions

**Redis** — Upload state (uploadId → chunkId bitmap + retry count), sync cursors, permission cache, session store

**Kafka** — File change events for Sync Service and Search indexing

## 5. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (MetaDB) | File hierarchy, permissions, versions, chunk mappings — ACID, relational |
| S3 | Exabyte-scale chunk storage, 11 nines durability, CDN integration |
| Redis | Upload state (chunk bitmap), sync cursors, permission cache |
| Elasticsearch | Full-text search on file names and content |

### PostgreSQL — folders

| Field | Type |
|---|---|
| folder_id | UUID (PK) |
| parent_folder_id | UUID, nullable (null = root) |
| owner_id | UUID |
| name | VARCHAR |
| created_at | TIMESTAMP |
| metadata | JSONB |

### PostgreSQL — files

| Field | Type |
|---|---|
| file_id | UUID (PK) |
| parent_folder_id | UUID (FK → folders) |
| owner_id | UUID |
| name | VARCHAR |
| size_bytes | BIGINT |
| current_version_id | UUID |
| is_deleted | BOOLEAN |
| metadata | JSONB |

### PostgreSQL — file_versions

| Field | Type |
|---|---|
| version_id | UUID (PK) |
| file_id | UUID (FK → files) |
| size_bytes | BIGINT |
| created_by | UUID |
| created_at | TIMESTAMP |

### PostgreSQL — file_version_chunks
Links a file version to its ordered list of chunks

| Field | Type |
|---|---|
| version_id | UUID (FK → file_versions) |
| chunk_index | INT (ordering of chunks) |
| chunk_id | UUID (FK → chunks) |

### PostgreSQL — chunks
Content-addressed; one row per unique chunk (deduplication at chunk level)

| Field | Type |
|---|---|
| chunk_id | UUID (PK) |
| object_key | TEXT (`s3://bucket/path`) |
| size_bytes | INT |
| checksum | VARCHAR (SHA256 — content hash) |
| created_at | TIMESTAMP |

### PostgreSQL — permissions

| Field | Type |
|---|---|
| permission_id | UUID (PK) |
| resource_type | ENUM (file / folder) |
| resource_id | UUID |
| user_id | UUID |
| role | ENUM (owner / editor / viewer) |
| created_at | TIMESTAMP |
| metadata | JSONB |

### PostgreSQL — users

| Field | Type |
|---|---|
| user_id | UUID (PK) |
| email | VARCHAR, unique |
| password_hash | VARCHAR |
| created_date | TIMESTAMP |
| metadata | JSONB |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `upload:{uploadId}` | Hash | `{chunkId_bitmap, retry_count, total_chunks}` | 86400s |
| `sync:cursor:{userId}:{deviceId}` | String | last sync timestamp | 30 days |
| `perm:cache:{fileId}:{userId}` | String | role | 300s |
| `session:{sessionId}` | String | userId | 86400s |

## 6. Key Flows

### 6.1 Auth

1. User registers/logs in → User Onboarding Service → write to User DB → return JWT
2. API Gateway validates JWT on every request
3. File Metadata Service checks permissions on every file operation

### 6.2 File Upload Protocol (6-Step)

The upload is a multi-step protocol that enables parallel chunk uploads, resumability, and deduplication.

**Step 1 — Initiate Upload**
```
POST /files/upload/init
{
  fileName: "video.mp4",
  fileSize: 52428800,
  chunkSize: 5MB,
  parentFolderId: "root121413",
  existingChunks: []   // client sends known chunk hashes; server skips those
}
→ returns { uploadId: "U456", fileId: "F123" }
```

**Step 2 — Client Chunks the File**
- Chunker splits file into N chunks of `chunkSize`
- Computes SHA256 of each chunk: `[sha0, sha1, ..., sha9]`
- Chunks with matching hash in `existingChunks` response are skipped (dedup)

**Step 3 — Request Signed URL per Chunk**
```
POST /upload/chunk-url
{
  uploadId: "U456",
  chunkId: 3,
  chunkHash: "xyz23"
}
→ Validator checks upload permission
→ returns { signedUrl: "https://s3...." }
```

**Step 4 — Upload Chunks Directly to S3 (Parallel, 3 at a time)**
- Client uploads each chunk directly to S3 using the pre-signed URL
- Bypasses app servers — reduces load and latency
- Parallel uploads (3 concurrent) for speed
- Redis tracks progress: `upload:{uploadId}` bitmap marks completed chunks
- On failure: retry individual chunk using same signed URL (idempotent)

**Step 5 — Commit Upload**
```
POST /uploads/{uploadId}/commit
```

**Step 6 — File Uploader calls File Metadata Service**
- File Metadata Service saves all metadata to MetaDB:
  - Creates `file_versions` record
  - Creates `file_version_chunks` records (chunk_index → chunk_id)
  - Creates/updates `chunks` records (chunk_id → S3 object_key)
  - Updates `files.current_version_id`
- Publishes `FILE_CREATED` / `FILE_UPDATED` event to Kafka
- Sync Service consumes → notifies other devices

### 6.3 File Download

1. Client `GET /files/{fileId}/download`
2. Read/Download Service validates permission (Redis cache → MetaDB)
3. Fetches `file_versions` → `file_version_chunks` → `chunks` to get all S3 object_keys in order
4. Returns pre-signed S3 URLs per chunk (or CDN URL for public files)
5. Client downloads chunks in parallel, reassembles in order

### 6.4 File Sync Across Devices

**Push (server → client):**
1. File Metadata Service publishes `FILE_UPDATED` to Kafka
2. Sync Service consumes, checks which devices are connected for that user
3. Pushes delta notification via WebSocket: `{fileId, newVersionId, changedChunks}`
4. Client Sync Engine receives, triggers download of changed chunks only

**Pull (client reconnects):**
1. Client sends last sync cursor (timestamp) to Sync Service
2. Sync Service queries MetaDB for all changes since cursor
3. Returns list of changed files with version info
4. Client downloads only changed chunks (delta sync)
5. Updates local cursor

**Conflict resolution:**
- Both devices modify same file offline
- On sync: detect conflict (both have newer version than last sync)
- Create conflict copy: `filename (Device A's copy).ext`
- User manually resolves

### 6.5 Delta Sync (Chunk-Level)

When a file is modified, only changed chunks need to be re-uploaded:
1. Watcher detects file change
2. Chunker re-chunks the file, computes SHA256 per chunk
3. Compares with Local Metadata Index (previous chunk hashes)
4. Only chunks with different hashes are uploaded
5. Server creates new `file_version` with updated `file_version_chunks` (reuses unchanged chunk_ids)

Example: 100MB file, user edits 1MB in the middle → only 1 chunk re-uploaded instead of 20.

### 6.6 Sharing

1. User shares file → Share Service writes to `permissions` table
2. For link sharing: generate `share_link_token = HMAC(fileId + role + expiry)`
3. Send email notification to recipient
4. On access: validate token → grant access

**Inherited permissions:**
- Sharing a folder grants access to all files inside
- Permission check: check file's direct permissions first; if none, check parent folder; walk up to root
- Cache results in Redis (TTL 5min)

### 6.7 Version History

1. Every upload commit creates a new `file_versions` record
2. Previous version's chunks retained in S3 (not deleted)
3. User views history → list of versions with timestamps
4. User restores version V: create new version pointing to V's `file_version_chunks`
5. Old versions purged after retention period (30 days free, unlimited paid)

## 7. Key Interview Concepts

### Chunked Upload Protocol
The 6-step protocol (init → chunk → signed URL → parallel S3 upload → commit → metadata) is the core design. Key benefits:
- Resumable: Redis bitmap tracks completed chunks; resume from last successful chunk on failure
- Parallel: 3 concurrent chunk uploads saturate available bandwidth
- Dedup: client sends existing chunk hashes on init; server skips already-stored chunks
- Direct S3 upload: bypasses app servers, reduces load

### Separate Chunks Table (Content-Addressed Storage)
Chunks stored by SHA256 hash in S3 (`s3://bucket/{chunkHash}`). MetaDB has a `chunks` table with `object_key`. Benefits:
- Deduplication at chunk level: two files sharing a 5MB chunk store it once
- Integrity: re-hash on download, compare to stored checksum
- Delta sync: unchanged chunks reused across versions without re-upload

### file_version_chunks as the Link Table
`file_version_chunks` maps `(version_id, chunk_index) → chunk_id`. This is what enables:
- Ordered chunk reassembly on download
- Delta sync: new version reuses unchanged chunk_ids, only adds new ones
- Storage efficiency: multiple versions share chunks

### Client-Side Components
- Watcher detects changes → triggers upload pipeline
- Chunker splits + hashes → enables dedup and delta
- Upload Manager tracks state → enables resumability
- Sync Engine handles pull/push → keeps devices in sync
- Local Metadata Index stores chunk hashes → enables delta detection without re-reading file

### Sync Cursor
Each device maintains a cursor (timestamp of last sync). On reconnect: "give me all changes since my cursor." Server returns delta. No need to compare full file lists. Cursor stored in Redis (per user per device).

### Permission Inheritance
Sharing a folder shares all contents. Permission check walks up the folder hierarchy. Cached in Redis (TTL 5min) to avoid repeated traversal. On permission change: actively invalidate cache.

### Pre-signed S3 URLs
Client uploads directly to S3 using time-limited pre-signed URLs. App servers never handle file bytes — only metadata and URL generation. Scales to any upload volume without app server bottleneck.

## 8. Failure Scenarios

### Upload Interrupted Mid-Chunk
- Recovery: client retries individual chunk using same signed URL (idempotent — same chunk hash = same S3 object); Redis bitmap shows which chunks completed; resume from there
- Prevention: Upload Manager persists state locally; Redis tracks server-side state

### S3 Region Outage
- Impact: uploads and downloads fail
- Recovery: cross-region S3 replication; CDN serves cached files; uploads queued and retried
- Prevention: multi-region S3; CDN absorbs most download traffic

### Sync Conflict
- Scenario: two devices modify same file offline
- Recovery: conflict copy created; user notified; manual resolution
- Prevention: last-write-wins for non-conflicting changes; conflict detection via version comparison

### MetaDB Failure
- Impact: file operations fail; downloads via CDN still work (cached)
- Recovery: promote replica (<30s); Kafka events retained; operations retry after recovery
- Prevention: synchronous replication; automated failover

### Permission Cache Stale
- Scenario: user's access revoked but Redis cache still shows access
- Recovery: short TTL (5min) limits window; on permission change, actively invalidate cache
- Prevention: cache invalidation on permission change; TTL as safety net

### Chunk Dedup Collision (SHA256)
- Scenario: two different files produce same SHA256 (extremely unlikely but theoretically possible)
- Recovery: use SHA256 + file size as composite key; collision probability negligible in practice
- Prevention: SHA256 collision resistance is sufficient for this use case
