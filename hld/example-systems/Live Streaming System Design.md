# Live Streaming System Design

## System Overview
A live video streaming platform (think Twitch / YouTube Live) where streamers broadcast real-time video to potentially millions of concurrent viewers with low latency, adaptive quality, and real-time chat.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Streamer starts/stops a live stream
- Viewers watch live stream with <10s latency
- Adaptive bitrate based on viewer's network
- Real-time chat during stream
- Stream recording and VOD (video on demand) after stream ends
- Stream discovery (browse live streams by category, viewers)
- Notifications when followed streamer goes live

### Non-Functional Requirements
- Availability: 99.99%
- Latency: <10s end-to-end (streamer to viewer); <30s acceptable for large scale
- Scalability: 1M+ concurrent streams, 100M+ concurrent viewers
- Durability: Stream recordings must not be lost

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1M concurrent live streams
- Average 100 viewers per stream = 100M concurrent viewers
- Streamer bitrate: 6Mbps (1080p)
- Viewer bitrate: 4Mbps avg (mix of qualities)
- Stream duration: 2hr avg

### Traffic
```
Ingest bandwidth    = 1M streams × 6Mbps = 6 Tbps
Delivery bandwidth  = 100M viewers × 4Mbps = 400 Tbps → CDN

Segments/sec        = 1M streams × (1 segment/2s) = 500K segments/sec
Chat messages/sec   = 100M viewers × 0.1 msg/sec = 10M msg/sec
```

### Storage
```
Recording/stream    = 6Mbps × 7200s (2hr) = ~5GB raw
Encoded variants    = 5 qualities × 3GB avg = 15GB per stream
1M streams/day      = 1M × 15GB = 15PB/day (only record fraction)
```

## 3. Core Components

**RTMP Ingest Server** — Receives live stream from streamer's broadcasting software (OBS, etc.) via RTMP protocol; authenticates stream key

**Transcoder** — Converts incoming RTMP stream to HLS/DASH segments in real-time; produces multiple quality variants (360p, 720p, 1080p); publishes segments to S3 and CDN origin

**Stream Manager** — Manages stream lifecycle (start, stop, health); writes stream metadata to Stream DB; publishes events to Kafka

**CDN** — Delivers HLS/DASH segments to viewers globally; live segments have short TTL (2–4s); edge nodes pull from origin on miss

**Playback Service** — Returns manifest URL and CDN endpoint to viewers; validates auth

**Chat Service** — Real-time chat via WebSocket; same architecture as Chat Application design

**Discovery Service** — Browse live streams; search by category, viewer count; backed by Elasticsearch

**Recording Service** — Kafka consumer; assembles stream segments into VOD after stream ends; stores to S3

**Notification Service** — Kafka consumer; sends push/email when followed streamer goes live

**Stream DB (Cassandra)** — Stream metadata, viewer counts, stream health

**Chat DB (Cassandra)** — Chat messages per stream

**S3** — Live segment origin, VOD recordings

**Kafka** — Stream events, chat fan-out, recording triggers

## 4. Database Design

### Cassandra — streams

| Field | Type |
|---|---|
| stream_id | UUID (PK) |
| streamer_id | UUID |
| title | VARCHAR |
| category | VARCHAR |
| status | ENUM (live / ended) |
| viewer_count | COUNTER |
| started_at | TIMESTAMP |
| ended_at | TIMESTAMP, nullable |
| manifest_url | TEXT |

### Cassandra — stream_segments

| Field | Type |
|---|---|
| stream_id | UUID (partition key) |
| segment_seq | BIGINT (clustering) |
| s3_url | TEXT |
| duration_ms | INT |
| created_at | TIMESTAMP |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `stream:live:{streamerId}` | String | stream metadata JSON | while live |
| `stream:viewers:{streamId}` | Counter | viewer count | while live |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Stream Start

1. Streamer starts broadcasting software (OBS) → RTMP Ingest Server with stream key
2. RTMP server validates stream key → authenticates streamer
3. Stream Manager creates stream record in Cassandra (`status = live`)
4. Publishes `STREAM_STARTED` to Kafka → Notification Service alerts followers
5. Transcoder begins processing incoming RTMP:
   - Splits into 2s HLS segments
   - Encodes to 360p / 720p / 1080p in parallel
   - Pushes segments to S3 and CDN origin
   - Updates HLS manifest file every 2s

### 5.2 Viewer Playback

1. Viewer opens stream → Playback Service → returns CDN manifest URL
2. Client fetches `.m3u8` manifest from CDN (TTL 2s — refreshed frequently)
3. Manifest lists last N segments at all quality levels
4. Client fetches segments from CDN edge (nearest node)
5. ABR algorithm adjusts quality based on download speed
6. CDN edge pulls new segments from origin every 2s (live segments have short TTL)

**Latency breakdown:**
```
Streamer encodes → RTMP ingest → Transcoder (2s segment) → CDN origin → CDN edge → Viewer
≈ 0.5s + 2s + 0.5s + 1s + 0.5s ≈ 4.5s minimum
With CDN propagation: 6–10s typical
```

### 5.3 Stream Recording → VOD

1. Stream ends → `STREAM_ENDED` event to Kafka
2. Recording Service consumes event
3. Fetches all segment URLs from Cassandra `stream_segments`
4. Assembles into MP4 / HLS VOD file
5. Stores to S3 with permanent URL
6. Updates stream record with VOD URL

### 5.4 Chat

Same architecture as Chat Application design — WebSocket connections, Redis for active sessions, Cassandra for message persistence. At 10M msg/sec, fan-out to all viewers in a stream is the key challenge — use Redis Pub/Sub per stream channel.

## 6. Key Interview Concepts

### RTMP vs WebRTC for Ingest
- RTMP: mature, widely supported by broadcasting software (OBS), higher latency (2–5s), reliable for one-to-many
- WebRTC: sub-second latency, browser-native, complex for server-side processing
- Most platforms use RTMP for ingest (streamer → server) and HLS/DASH for delivery (server → viewers)

### HLS Segment TTL
Live segments have 2–4s TTL on CDN. This means CDN constantly pulls fresh segments from origin. For 1M streams × 3 quality variants = 3M segment requests/2s = 1.5M/sec to origin. CDN must be sized for this pull rate.

### Viewer Count at Scale
100M concurrent viewers updating viewer count = massive write load. Solution: approximate counting — each CDN edge reports viewer count to a central aggregator every 30s. Aggregator sums and updates Redis counter. Slight staleness (30s) is acceptable for viewer count display.

### Low-Latency Streaming (LL-HLS)
Standard HLS: 6–30s latency. Low-Latency HLS (LL-HLS): 2–5s latency by using partial segments (200ms chunks) and HTTP/2 push. Trade-off: more complex, higher origin load, not all CDNs support it.

## 7. Failure Scenarios

### RTMP Ingest Server Failure
- Detection: stream drops, health check fails
- Recovery: streamer's software reconnects to backup ingest server; brief stream interruption (<5s)
- Prevention: multiple ingest servers per region; DNS failover

### Transcoder Failure
- Detection: no new segments published; manifest not updated
- Recovery: stream manager detects stale manifest, restarts transcoder, reconnects to RTMP stream
- Prevention: multiple transcoder instances; health monitoring

### CDN Overload (Viral Stream)
- Scenario: stream suddenly gets 10M viewers
- Recovery: CDN auto-scales edge capacity; origin pull rate increases but is bounded by segment size
- Prevention: CDN with auto-scaling; pre-warm for known large events
