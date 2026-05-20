# File Storage Service API and DB Design

## Problem Statement

Design the API and metadata schema for a file storage service where users upload, download, share, and organize files.

**Core requirements:**
- Upload/download files reliably
- Store metadata and permissions
- Support folder hierarchy and sharing links
- Version files on updates

**Scale:** 500M files, heavy read/download traffic, large object sizes

---

## API Design

### Core Endpoints
```http
POST /api/v1/files/upload-init
PUT /api/v1/files/{fileId}/parts/{partNumber}
POST /api/v1/files/{fileId}/complete
GET /api/v1/files/{fileId}/download-url
POST /api/v1/files/{fileId}/share
GET /api/v1/folders/{folderId}/children?cursor=...&limit=50
```

### Example: Upload Init
```json
POST /api/v1/files/upload-init
{
  "name": "design-doc.pdf",
  "sizeBytes": 2432456,
  "contentType": "application/pdf",
  "folderId": "fld_12"
}
```

```json
201 Created
{
  "fileId": "fil_890",
  "uploadId": "up_221",
  "partSizeBytes": 5242880
}
```

### Design Decisions
- Use multipart upload for large files
- Return pre-signed URLs for direct object store upload/download
- Keep file metadata in DB, blob data in object storage

---

## Database Schema

```sql
CREATE TABLE files (
    id                  VARCHAR(64) PRIMARY KEY,
    owner_id            VARCHAR(64) NOT NULL,
    folder_id           VARCHAR(64),
    name                VARCHAR(255) NOT NULL,
    content_type        VARCHAR(120),
    size_bytes          BIGINT NOT NULL,
    storage_key         VARCHAR(400) NOT NULL UNIQUE,
    latest_version      INTEGER NOT NULL DEFAULT 1,
    is_deleted          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_files_owner_folder ON files (owner_id, folder_id);
CREATE INDEX idx_files_folder_name ON files (folder_id, name);

CREATE TABLE file_versions (
    file_id             VARCHAR(64) NOT NULL REFERENCES files(id),
    version_no          INTEGER NOT NULL,
    storage_key         VARCHAR(400) NOT NULL,
    size_bytes          BIGINT NOT NULL,
    checksum_sha256     VARCHAR(64),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (file_id, version_no)
);

CREATE TABLE file_shares (
    id                  VARCHAR(64) PRIMARY KEY,
    file_id             VARCHAR(64) NOT NULL REFERENCES files(id),
    created_by          VARCHAR(64) NOT NULL,
    access_level        VARCHAR(16) NOT NULL, -- VIEW, EDIT
    expires_at          TIMESTAMPTZ,
    token_hash          VARCHAR(128) NOT NULL UNIQUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Trade-offs and Considerations

- Blob store + metadata DB is standard for scale and cost.
- Strong consistency for metadata; eventual consistency acceptable for analytics.
- Soft delete with delayed purge helps recovery and compliance workflows.
