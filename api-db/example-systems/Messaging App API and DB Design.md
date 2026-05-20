# Messaging App API and DB Design

## System Overview
A real-time messaging application (think WhatsApp or Slack) supporting one-on-one and group conversations, delivery and read receipts, media attachments, and message ordering guarantees. The core challenge is ensuring messages are delivered in order and receipts are tracked efficiently at scale.

## 1. Requirements

### Functional Requirements
- One-on-one and group conversations (up to 256 members)
- Send text messages and media attachments (images, files)
- Delivery receipts (sent, delivered, read) per message per recipient
- Message ordering within a conversation using sequence numbers
- Edit and delete messages (soft delete with "message deleted" placeholder)
- Search messages within a conversation
- Online/last-seen presence indicators

### Non-Functional Requirements
- Availability: 99.99% — messages must never be lost
- Latency: <100ms for message delivery (P99)
- Ordering: Messages within a conversation must be delivered in send order
- Scalability: 500M users, 100B messages/day
- Durability: Messages stored for 5 years

## 2. Scale Estimation

```
DAU                     = 100M users
Messages/day            = 100B
Messages/sec (avg)      = 100B / 86400 ≈ 1.16M/sec
Peak messages/sec       ≈ 5M/sec

Message size (avg)      = 200B (text) + metadata
Media messages          = 10% of total → 10B/day
Media storage/day       = 10B × 100KB avg = 1PB/day (CDN + object store)

Message metadata/day    = 100B × 200B = 20TB/day
5-year storage          ≈ 36PB (messages + media)
```

## 3. API Design

### Key Endpoints

#### Send Message
```
POST /api/v1/conversations/{conversationId}/messages
Authorization: Bearer <token>

Request:
{
  "content": "Hey, are you free tonight?",
  "content_type": "TEXT",
  "client_message_id": "client-uuid-abc123",   // idempotency key
  "reply_to_message_id": null
}

Response 201:
{
  "message_id": "msg_xyz",
  "conversation_id": "conv_abc",
  "sequence_number": 1042,
  "sender_id": "user_123",
  "content": "Hey, are you free tonight?",
  "status": "SENT",
  "created_at": "2025-06-01T10:00:00Z"
}
```

#### Get Messages (with cursor pagination)
```
GET /api/v1/conversations/{conversationId}/messages?before_seq=1042&limit=50

Response 200:
{
  "messages": [
    {
      "message_id": "msg_xyz",
      "sequence_number": 1042,
      "sender": { "id": "user_123", "name": "Alice" },
      "content": "Hey, are you free tonight?",
      "content_type": "TEXT",
      "receipts": { "delivered_count": 1, "read_count": 0 },
      "created_at": "2025-06-01T10:00:00Z"
    }
  ],
  "has_more": true,
  "oldest_seq": 993
}
```

#### Create Conversation
```
POST /api/v1/conversations
Authorization: Bearer <token>

Request:
{
  "type": "GROUP",
  "name": "Weekend Plans",
  "member_ids": ["user_456", "user_789"]
}

Response 201:
{
  "conversation_id": "conv_abc",
  "type": "GROUP",
  "name": "Weekend Plans",
  "members": [...],
  "created_at": "2025-06-01T09:00:00Z"
}
```

#### Mark Messages as Read
```
POST /api/v1/conversations/{conversationId}/read
Authorization: Bearer <token>

Request: { "up_to_sequence_number": 1042 }
Response 204
```

#### Upload Media Attachment
```
POST /api/v1/media/upload
Authorization: Bearer <token>

Request: multipart/form-data with file

Response 201:
{
  "media_id": "media_abc",
  "url": "https://cdn.example.com/media/media_abc.jpg",
  "content_type": "image/jpeg",
  "size_bytes": 245000
}
```

### Endpoint Summary

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/conversations | Create conversation |
| GET | /api/v1/conversations | List user's conversations |
| GET | /api/v1/conversations/{id}/messages | Get messages (paginated) |
| POST | /api/v1/conversations/{id}/messages | Send message |
| PATCH | /api/v1/messages/{messageId} | Edit message |
| DELETE | /api/v1/messages/{messageId} | Delete message (soft) |
| POST | /api/v1/conversations/{id}/read | Mark messages as read |
| POST | /api/v1/media/upload | Upload media |
| GET | /api/v1/conversations/{id}/search | Search messages |

## 4. Database Schema

```sql
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    phone           VARCHAR(30) NOT NULL UNIQUE,
    username        VARCHAR(50) UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      VARCHAR(500),
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE conversations (
    id              BIGSERIAL PRIMARY KEY,
    type            VARCHAR(20) NOT NULL,   -- DIRECT, GROUP
    name            VARCHAR(255),           -- NULL for DIRECT
    avatar_url      VARCHAR(500),
    created_by      BIGINT REFERENCES users(id),
    last_message_id BIGINT,                 -- denormalized for conversation list
    last_message_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE conversation_members (
    conversation_id BIGINT NOT NULL REFERENCES conversations(id),
    user_id         BIGINT NOT NULL REFERENCES users(id),
    role            VARCHAR(20) NOT NULL DEFAULT 'MEMBER',  -- ADMIN, MEMBER
    last_read_seq   BIGINT NOT NULL DEFAULT 0,  -- highest sequence number read
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    left_at         TIMESTAMPTZ,
    PRIMARY KEY (conversation_id, user_id)
);

CREATE INDEX idx_conv_members_user_id ON conversation_members (user_id)
    WHERE left_at IS NULL;

-- Messages with per-conversation sequence numbers
CREATE TABLE messages (
    id                  BIGSERIAL PRIMARY KEY,
    conversation_id     BIGINT NOT NULL REFERENCES conversations(id),
    sender_id           BIGINT NOT NULL REFERENCES users(id),
    sequence_number     BIGINT NOT NULL,        -- monotonic per conversation
    content             TEXT,
    content_type        VARCHAR(20) NOT NULL DEFAULT 'TEXT',
                        -- TEXT, IMAGE, VIDEO, FILE, AUDIO, SYSTEM
    media_id            BIGINT REFERENCES media_attachments(id),
    reply_to_message_id BIGINT REFERENCES messages(id),
    is_deleted          BOOLEAN NOT NULL DEFAULT FALSE,
    edited_at           TIMESTAMPTZ,
    client_message_id   VARCHAR(100),           -- idempotency key
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (conversation_id, sequence_number),
    UNIQUE (sender_id, client_message_id)       -- dedup per sender
);

CREATE INDEX idx_messages_conversation_seq ON messages (conversation_id, sequence_number DESC);
CREATE INDEX idx_messages_sender_id ON messages (sender_id);
-- Full-text search within conversations
CREATE INDEX idx_messages_fts ON messages
    USING GIN (to_tsvector('english', COALESCE(content, '')))
    WHERE content IS NOT NULL AND is_deleted = FALSE;

-- Per-recipient delivery/read receipts
CREATE TABLE message_receipts (
    message_id      BIGINT NOT NULL REFERENCES messages(id),
    user_id         BIGINT NOT NULL REFERENCES users(id),
    delivered_at    TIMESTAMPTZ,
    read_at         TIMESTAMPTZ,
    PRIMARY KEY (message_id, user_id)
);

CREATE INDEX idx_receipts_user_id ON message_receipts (user_id, read_at);

CREATE TABLE media_attachments (
    id              BIGSERIAL PRIMARY KEY,
    uploader_id     BIGINT NOT NULL REFERENCES users(id),
    url             VARCHAR(500) NOT NULL,
    thumbnail_url   VARCHAR(500),
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    width           INT,
    height          INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Sequence number generator per conversation (prevents gaps)
CREATE TABLE conversation_sequences (
    conversation_id BIGINT PRIMARY KEY REFERENCES conversations(id),
    last_sequence   BIGINT NOT NULL DEFAULT 0
);
```

### Schema Summary

| Table | Purpose |
|-------|---------|
| conversations | Conversation metadata (direct or group) |
| conversation_members | Membership with last-read tracking |
| messages | Message content with sequence numbers |
| message_receipts | Per-recipient delivery and read status |
| media_attachments | Media metadata (content in object store) |
| conversation_sequences | Sequence number generator per conversation |

## 5. Key Design Decisions

### 5.1 Per-Conversation Sequence Numbers

Global message IDs don't guarantee ordering within a conversation (gaps, out-of-order delivery). Per-conversation sequence numbers solve this:

```sql
-- Atomic sequence increment
UPDATE conversation_sequences
SET last_sequence = last_sequence + 1
WHERE conversation_id = $1
RETURNING last_sequence;
```

This runs inside the same transaction as the message insert. The sequence is monotonically increasing per conversation, enabling clients to detect gaps and request missing messages.

### 5.2 Read Receipts at Scale

Storing one row per message per recipient in `message_receipts` is expensive for group chats (256 members × millions of messages). The optimization:

- `conversation_members.last_read_seq` tracks the highest sequence number a user has read
- For "read by N" counts, query: `SELECT COUNT(*) FROM conversation_members WHERE last_read_seq >= $seq`
- Individual `message_receipts` rows are only written for the most recent N messages (configurable)

### 5.3 Idempotent Message Delivery

Network retries can cause duplicate messages. The `client_message_id` (unique per sender) prevents duplicates:

```sql
INSERT INTO messages (conversation_id, sender_id, content, client_message_id, ...)
VALUES ($1, $2, $3, $4, ...)
ON CONFLICT (sender_id, client_message_id) DO NOTHING
RETURNING id, sequence_number;
```

If the message already exists, the existing record is returned without creating a duplicate.

### 5.4 Media Storage Separation

Media content is stored in object storage (S3), not the database. The `media_attachments` table stores only metadata (URL, size, dimensions). This keeps the database lean and allows CDN delivery of media. Presigned URLs are used for direct client-to-S3 uploads.

### 5.5 Conversation List Performance

The conversation list shows the most recent message per conversation. `conversations.last_message_at` is denormalized for fast sorting:

```sql
SELECT c.*, cm.last_read_seq
FROM conversations c
JOIN conversation_members cm ON c.id = cm.conversation_id
WHERE cm.user_id = $1 AND cm.left_at IS NULL
ORDER BY c.last_message_at DESC
LIMIT 20;
```

Without denormalization, this would require a correlated subquery or window function over all messages.

## 6. Failure Scenarios

### Message Ordering Gap
- **Impact**: Client receives message with sequence 1044 but never received 1043; gap in conversation
- **Recovery**: Client detects gap (seq 1044 when expecting 1043); requests missing messages via REST API
- **Prevention**: Sequence numbers make gaps detectable; WebSocket reconnect triggers gap-fill request

### Duplicate Message on Retry
- **Impact**: User sees the same message twice
- **Recovery**: `ON CONFLICT DO NOTHING` on `client_message_id` prevents DB duplicates; client deduplicates by `client_message_id`
- **Prevention**: Always include `client_message_id` in send requests; client generates UUID per send attempt

### Group Chat Fan-Out at Scale
- **Impact**: Sending a message to a 256-member group requires 256 WebSocket pushes; slow for large groups
- **Recovery**: Fan-out via message queue (Kafka); each member's connection server consumes their messages
- **Prevention**: Async fan-out for groups >10 members; direct push for small groups

### Media Upload Failure Mid-Send
- **Impact**: Message sent with broken media reference
- **Recovery**: Upload media first, get `media_id`, then send message; never send message before media upload confirms
- **Prevention**: Two-step flow: upload → get media_id → send message with media_id; validate media_id exists before message insert
