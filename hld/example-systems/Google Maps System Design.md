# Google Maps System Design

## System Overview
A mapping and navigation platform that provides map rendering, location search, turn-by-turn navigation with real-time traffic, ETA calculation, and route optimization.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Display interactive map tiles at any zoom level
- Search for places (restaurants, addresses, POIs)
- Get directions between two points (driving, walking, transit)
- Real-time traffic overlay and ETA
- Turn-by-turn navigation with rerouting
- User location tracking during navigation
- Contribute map data (reviews, photos, corrections)

### Non-Functional Requirements
- Availability: 99.99%
- Latency: <100ms for map tile load; <500ms for route calculation
- Scalability: 1B+ users, 25M+ map tile requests/sec
- Freshness: Traffic data updated every 30–60s; map data updated periodically

## 2. Back-of-the-Envelope Estimation

### Assumptions
- 1B users, 100M DAU
- Each user loads 20 map tiles per session, 5 sessions/day
- 10M route requests/day
- 50M active navigation sessions at peak
- Location ping every 5s during navigation

### Traffic
```
Map tile requests/sec   = 100M × 20 × 5 / 86400 ≈ 115K/sec → CDN
Route requests/sec      = 10M / 86400 ≈ 116/sec
Location pings/sec      = 50M × (1/5s) = 10M/sec
```

### Storage
```
Map tiles (all zoom levels) = ~100TB (pre-rendered, static)
POI database                = 200M POIs × 1KB = 200GB
Road graph                  = ~500GB (nodes + edges with attributes)
Traffic data                = rolling 1hr window, ~10GB
```

## 3. Core Components

**API Gateway** — Auth, rate limiting, routing

**Map Tile Service** — Serves pre-rendered map tiles by zoom level and coordinates; tiles served from CDN

**Tile Generation Service** — Offline pipeline; renders map tiles from raw map data (OpenStreetMap / proprietary); stores to S3; triggered on map data updates

**Search Service** — Place and address search; backed by Elasticsearch with geo-indexing; autocomplete via prefix search

**Routing Service** — Computes optimal route between two points; uses road graph with Dijkstra / A* / Contraction Hierarchies algorithm; incorporates real-time traffic weights

**Traffic Service** — Aggregates real-time location data from users in navigation mode; computes traffic speed per road segment; updates road graph weights

**Navigation Service** — Manages active navigation sessions; provides turn-by-turn instructions; detects deviation and triggers rerouting

**Location Ingestion Service** — Receives GPS pings from users in navigation mode; feeds into Traffic Service

**ETA Service** — Calculates estimated arrival time based on route + current traffic

**POI Service** — Points of interest data (restaurants, hospitals, etc.); search, details, reviews

**Map DB (PostgreSQL + PostGIS)** — Road graph, POI data, geo-indexed

**Traffic DB (Redis + Time-Series)** — Real-time traffic speed per road segment; short retention

**Search Index (Elasticsearch)** — POI and address search with geo-distance

**S3 + CDN** — Pre-rendered map tiles; static, highly cacheable

**Kafka** — Location ping stream, traffic aggregation pipeline

## 4. Database Design

### Selection Reasoning

| Store | Why |
|---|---|
| PostgreSQL + PostGIS | Road graph and POI data; geo-indexed; complex spatial queries |
| Redis | Real-time traffic speeds per road segment; sub-ms reads for routing |
| Elasticsearch | Full-text + geo search for places and addresses |
| S3 + CDN | Pre-rendered map tiles; static, globally cached |
| Kafka | Location ping stream for traffic aggregation |

### PostgreSQL — road_graph nodes

| Field | Type |
|---|---|
| node_id | BIGINT (PK) |
| lat | DOUBLE |
| lng | DOUBLE |
| location | POINT (PostGIS) |

### PostgreSQL — road_graph edges

| Field | Type |
|---|---|
| edge_id | BIGINT (PK) |
| from_node | BIGINT (FK → nodes) |
| to_node | BIGINT (FK → nodes) |
| distance_m | INT |
| speed_limit_kmh | INT |
| road_type | VARCHAR (highway / arterial / local) |
| is_one_way | BOOLEAN |

### PostgreSQL — pois

| Field | Type |
|---|---|
| poi_id | UUID (PK) |
| name | VARCHAR |
| category | VARCHAR |
| address | TEXT |
| location | POINT (PostGIS) |
| rating | DECIMAL |
| metadata | JSONB |

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `traffic:edge:{edgeId}` | String | current speed (km/h) | 120s |
| `traffic:segment:{segmentId}` | String | congestion level | 60s |
| `route:cache:{hash}` | String | route JSON | 300s |
| `session:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Map Tile Rendering

Map tiles are pre-rendered offline, not on-demand:

1. Map data updated (OSM import or proprietary update)
2. Tile Generation Service renders tiles at all zoom levels (0–20)
3. Tiles stored to S3: `tiles/{zoom}/{x}/{y}.png`
4. CDN caches tiles globally
5. Client requests tile: `GET /tiles/15/12345/67890.png`
6. CDN serves from cache (TTL: days/weeks — tiles rarely change)
7. On map data update: invalidate affected tiles, re-render

**Tile coordinate system (XYZ / Slippy Map):**
```
Zoom 0: 1 tile covers entire world
Zoom 10: 1M tiles cover the world
Zoom 15: 1B tiles — city-level detail
Each tile: 256×256 pixels PNG
```

### 5.2 Place Search

1. User types "coffee near me" → Search Service
2. Elasticsearch query: full-text on `name`, `category` + geo-distance filter (within 5km of user)
3. Sort by relevance × rating × distance
4. Return top 20 results with name, address, rating, distance
5. Autocomplete: prefix search on `name` field with geo-bias

### 5.3 Route Calculation

1. User requests route from A to B → Routing Service
2. Check route cache: `route:cache:{hash(A,B,mode,timestamp_bucket)}` in Redis
3. Cache hit: return cached route (valid for 5 min)
4. Cache miss: run routing algorithm on road graph
5. Load road graph from PostgreSQL (or in-memory graph for hot regions)
6. Apply real-time traffic weights from Redis (`traffic:edge:{edgeId}`)
7. Run Contraction Hierarchies (pre-processed) or A* for fast shortest path
8. Calculate ETA: sum of (edge_distance / current_speed) for all edges on route
9. Cache result in Redis (TTL 5 min)
10. Return route as polyline + turn-by-turn instructions

### 5.4 Real-Time Traffic

1. Users in navigation mode send GPS ping every 5s → Location Ingestion Service
2. Location Ingestion Service publishes to Kafka
3. Traffic Service consumes pings:
   - Map GPS coordinates to nearest road segment (map matching)
   - Calculate speed: distance between consecutive pings / time
   - Aggregate speeds per road segment (rolling 5-min window)
4. Update Redis: `SET traffic:edge:{edgeId} {speed_kmh} EX 120`
5. Routing Service reads traffic weights from Redis for route calculation

### 5.5 Navigation & Rerouting

1. User starts navigation → Navigation Service creates session
2. Client sends GPS ping every 5s
3. Navigation Service checks if user is on route:
   - If on route: send next turn instruction
   - If deviated (>50m off route): trigger reroute
4. Reroute: call Routing Service with current position as new origin
5. Return updated route to client

## 6. Key Interview Concepts

### Map Tiles — Pre-render vs On-demand
Pre-rendering all tiles offline is the right approach. On-demand rendering at 115K req/sec would require massive compute. Pre-rendered tiles are static, highly cacheable, and served entirely from CDN. Only re-render when map data changes (infrequent).

### Road Graph Representation
Graph: nodes = intersections, edges = road segments with attributes (distance, speed limit, road type). Routing algorithms (Dijkstra, A*) find shortest path by minimizing cost (time = distance / speed). Real-time traffic modifies edge weights dynamically.

### Contraction Hierarchies
Standard Dijkstra on a city-scale graph (millions of nodes) is too slow. Contraction Hierarchies pre-process the graph by "contracting" less important nodes, creating shortcut edges. Query time: milliseconds instead of seconds. Trade-off: preprocessing takes hours but queries are fast.

### Map Matching
GPS coordinates have noise (±10m). Map matching snaps GPS points to the nearest road segment using Hidden Markov Model (HMM) — considers road geometry, heading, and speed to determine which road the user is actually on.

### Geohash / H3 for Traffic Aggregation
Divide the world into hexagonal cells (H3) or grid cells (geohash). Aggregate traffic data per cell rather than per road segment for coarse-grained traffic overlay. Fine-grained per-segment data for routing.

### ETA Accuracy
ETA = sum of (segment_length / current_speed) for all route segments. Accuracy depends on traffic data freshness (30–60s lag) and historical patterns (time of day, day of week). ML models improve ETA by learning from historical trip data.

## 7. Failure Scenarios

### Traffic Service Failure
- Impact: routing uses historical/default speeds; ETA less accurate
- Recovery: Routing Service falls back to speed limit data; Traffic Service is stateless, restarts quickly
- Prevention: Redis retains last known traffic data (TTL 120s); brief outage has minimal impact

### Routing Service Overload
- Detection: latency spikes, queue depth grows
- Recovery: route cache in Redis absorbs repeated queries for same routes; auto-scale Routing Service instances
- Prevention: aggressive caching (5 min TTL); pre-compute popular routes (airport to city center, etc.)

### CDN Tile Cache Miss Storm
- Scenario: map data update invalidates millions of tiles simultaneously
- Recovery: staggered tile invalidation; CDN pulls from S3 on miss; S3 handles the load
- Prevention: invalidate tiles in batches; pre-warm CDN for high-traffic areas after update
