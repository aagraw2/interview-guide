# Google Drive System Design

## System Overview
A cloud file storage and sync service (think Google Drive / Dropbox / OneDrive) where users upload, store, organize, and share files — with real-time sync across devices, collaborative access, and version history.

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
- Real-time collaboration on supported file types (Docs, Sheets)
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

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing

**Metadata Service** — File and folder metadata CRUD; permissions; version records; writes to Metadata DB

**Upload Service** — Handles file uploads; chunking for large files; deduplication; stores chunks to S3; updates Metadata Service

**Download Service** — Serves file downloads; validates permissions; returns CDN URLs or streams from S3

**Sync Service** — Manages device sync state; detects changes; pushes delta updates to connected devices via WebSocket/long-poll

**Share Service** — Manages sharing permissions; generates share links; access control

**Search Service** — File name and content search; Elasticsearch; CDC from Metadata DB

**Version Service** — Manages file version history; stores version metadata; retrieves previous versions

**Notification Service** — Notifies users of shared file changes, comments, access requests

**Metadata DB (MySQL/PostgreSQL)** — File/folder hierarchy, permissions, version records

**Chunk Store (S3)** — Actual file data stored as content-addressed chunks

**CDN** — Caches frequently downloaded files at edge

**Redis** — Active sync sessions, upload state, permission cache, session store

**Kafka** — File change events, sync notifications, search indexing

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL (Metadata DB) | File hierarchy (adjacency list), permissions, versions — ACID, relational |
| S3 (Chunk Store) | Exabyte-scale file storage, 11 nines durability, CDN integration |
| Redis | Upload state (resumable), sync session state, permission cache |
| Elasticsearch | Full-text search on file names and content |

### PostgreSQL — files

| Field | Type |
|---|---|
| file_id | UUID (PK) |
| owner_id | UUID |
| parent_folder_id | UUID, nullable (null = root) |
| name | VARCHAR |
| size_bytes | BIGINT |
| mime_type | VARCHAR |
| content_hash | VARCHAR (SHA-256, for dedup) |
| current_version_id | UUID |
| is_deleted | BOOLEAN |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

### PostgreSQL — file_versions

| Field | Type |
|---|---|
| version_id | UUID (PK) |
| file_id | UUID (FK → files) |
| version_number | INT |
| size_bytes | BIGINT |
| content_hash | VARCHAR |
| s3_key | TEXT (location in S3) |
| created_by | UUID |
| created_at | TIMESTAMP |

### PostgreSQL — permissions

| Field | Type |
|---|---|
| resource_id | UUID (file_id or folder_id) |
| resource_type | ENUM (file / folder) |
| grantee_id | UUID, nullable |
| share_link_token | VARCHAR, nullable |
| access_level | ENUM (owner / editor / viewer) |
| created_at | TIMESTAMP |

### PostgreSQL — folders

| Field | Type |
|---|---|
| folder_id | UUID (PK) |
| owner_id | UUID |
| parent_folder_id | UUID, nullable |
| name | VARCHAR |
| is_deleted | BOOLEAN |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `upload:state:{uploadId}` | String | `{chunks_uploaded, total_chunks}` | 86400s |
| `sync:cursor:{userId}:{deviceId}` | String | last sync timestamp | 30 days |
| `perm:cache:{fileId}:{userId}` | String | access_level | 300s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Auth

1. User logs in → JWT (1hr) + refresh token → session in Redis
2. API Gateway validates JWT on every request
3. Metadata Service checks permissions on every file operation

### 5.2 File Upload

**Small file (<5MB, single-part):**
1. Client `POST /upload` with file data
2. Upload Service computes SHA-256 hash of file content
3. Deduplication check: does `content_hash` already exist in S3?
   - If yes: reuse existing S3 object (no re-upload) — just create new metadata record pointing to same chunk
   - If no: upload to S3
4. Metadata Service creates file record + version record
5. Publish `FILE_CREATED` to Kafka → Search Service indexes, Sync Service notifies devices

**Large file (>5MB, chunked upload):**
1. Client initiates: `POST /upload/init` → returns `uploadId`
2. Client splits file into 5MB chunks, uploads each: `PUT /upload/{uploadId}/chunk/{n}`
3. Upload Service stores each chunk to S3 with key `chunks/{contentHash}`
4. Client completes: `POST /upload/{uploadId}/complete`
5. Upload Service assembles manifest, creates file record
6. Resumable: if upload interrupted, client resumes from last uploaded chunk (tracked in Redis)

### 5.3 File Download

1. Client `GET /files/{fileId}/download`
2. Download Service validates permission (Redis cache → PostgreSQL)
3. Fetch `s3_key` from Metadata DB
4. Return pre-signed S3 URL (valid 1hr) or CDN URL for public files
5. Client downloads directly from S3/CDN

### 5.4 File Sync Across Devices

**Change detection:**
1. User modifies file on Device A → Upload Service processes new version
2. Publishes `FILE_UPDATED` event to Kafka with `{fileId, userId, newVersionId, timestamp}`
3. Sync Service consumes event
4. Checks which of user's devices are connected (Redis `sync:cursor:{userId}:{deviceId}`)
5. Pushes delta notification to connected devices via WebSocket
6. Offline devices: delta queued; delivered on reconnect

**Device sync on reconnect:**
1. Device connects, sends `sync:cursor` (last sync timestamp)
2. Sync Service queries Metadata DB for all changes since cursor
3. Returns list of changed files (added, modified, deleted)
4. Device downloads changed files
5. Updates cursor to current timestamp

**Conflict resolution:**
- Both devices modify same file while offline
- On sync: detect conflict (both have newer version than last sync)
- Create conflict copy: `filename (Device A's copy).ext`
- User manually resolves
- Same approach as Dropbox

### 5.5 Sharing

1. User shares file → Share Service
2. Write permission record to PostgreSQL
3. Generate share link token (if link sharing): `HMAC(fileId + accessLevel + expiry)`
4. Send email notification to recipient
5. Recipient accesses file: Share Service validates token → grants access

**Inherited permissions:**
- Sharing a folder grants access to all files inside
- Permission check: walk up folder hierarchy until permission found
- Cache permission results in Redis (TTL 5min)

### 5.6 Version History

1. Every file upload creates a new version record
2. Previous version's S3 object retained (not deleted)
3. User views history: `GET /files/{fileId}/versions` → list of versions with timestamps
4. User restores version V: Upload Service creates new version with V's content_hash
5. Old versions deleted after retention period (e.g., 30 days for free tier, unlimited for paid)

### 5.7 Deduplication

Content-addressed storage: S3 key = `chunks/{SHA256_hash}`. If two users upload the same file, only one copy stored in S3. Both users' metadata records point to the same S3 object. Saves significant storage (common files: PDFs, images, videos).

## 6. Key Interview Concepts

### Content-Addressed Storage
Files stored by their content hash (SHA-256), not by name. Same content = same key = one copy in S3. Benefits:
- Deduplication: 30–50% storage savings
- Integrity verification: re-hash on download, compare to stored hash
- Efficient sync: if hash unchanged, file unchanged — no re-upload needed

### Chunked Upload for Large Files
5GB file as single upload: network interruption = restart from scratch. Chunked upload: each 5MB chunk uploaded independently. On interruption: resume from last successful chunk. Also enables parallel chunk upload (faster for large files).

### Delta Sync
Instead of re-uploading entire file on change, sync only changed chunks. Client computes rolling hash of file chunks, compares with server's chunk hashes, uploads only changed chunks. Rsync algorithm. Reduces bandwidth significantly for large files with small changes.

### Folder Hierarchy in DB
Adjacency list model: each file/folder has `parent_folder_id`. Simple, works for most queries. For deep hierarchy traversal (find all files in folder tree): use recursive CTE in PostgreSQL or materialized path pattern.

### Permission Inheritance
Sharing a folder shares all contents. Permission check: check file's direct permissions first; if none, check parent folder; walk up to root. Cache results in Redis to avoid repeated hierarchy traversal.

### Sync Cursor
Each device maintains a cursor (timestamp of last sync). On reconnect: "give me all changes since my cursor." Server returns delta. Efficient — no need to compare full file lists. Cursor stored in Redis (per user per device).

## 7. Failure Scenarios

### Upload Service Crash Mid-Upload
- Recovery: resumable upload — client retries from last successful chunk; Redis tracks upload state; idempotent chunk upload (same chunk hash = same S3 object)
- Prevention: upload state persisted to Redis before each chunk ACK

### S3 Region Outage
- Impact: uploads and downloads fail
- Recovery: cross-region replication; CDN serves cached files; uploads queued and retried
- Prevention: multi-region S3 replication; CDN absorbs most download traffic

### Sync Conflict
- Scenario: two devices modify same file offline
- Recovery: conflict copy created; user notified; manual resolution
- Prevention: last-write-wins for non-conflicting changes; conflict detection via version comparison

### Metadata DB Failure
- Impact: file operations fail; downloads via CDN still work (cached)
- Recovery: promote replica (<30s); Kafka events retained; operations retry after recovery
- Prevention: synchronous replication; automated failover

### Permission Cache Stale
- Scenario: user's access revoked but Redis cache still shows access
- Recovery: short TTL (5min) limits window; on permission change, actively invalidate cache
- Prevention: cache invalidation on permission change; TTL as safety net
