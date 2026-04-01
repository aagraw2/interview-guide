## 1. The Problem: Proximity Queries

How do you efficiently find nearby locations?

```
Problem:
  Find all restaurants within 5 km of user's location
  User: (37.7749, -122.4194) [San Francisco]

Naive approach:
  Calculate distance to every restaurant
  → O(N) for N restaurants
  → Slow for millions of locations

Better approach:
  Encode lat/lng into a string (geohash or S2 cell)
  → Nearby locations have similar strings
  → Use string prefix matching for proximity queries
  → O(log N) with spatial index
```

---

## 2. Geohash

### What is Geohash?

**Geohash** encodes latitude and longitude into a short string. Nearby locations have similar geohashes.

```
San Francisco:
  Lat: 37.7749, Lng: -122.4194
  Geohash: 9q8yy (5 characters)

Nearby location (1 km away):
  Lat: 37.7850, Lng: -122.4100
  Geohash: 9q8yz (shares prefix "9q8y")

Far location (New York):
  Lat: 40.7128, Lng: -74.0060
  Geohash: dr5ru (different prefix)
```

**Key property:** Longer shared prefix = closer locations.

---

### How Geohash Works

Geohash uses **interleaved binary encoding** of latitude and longitude.

#### Step 1: Binary Encoding

Divide the world into a grid using binary search.

```
Latitude range: [-90, 90]
Longitude range: [-180, 180]

Example: Lat 37.7749, Lng -122.4194

Longitude -122.4194:
  [-180, 180] → midpoint 0
  -122.4194 < 0 → left half → bit 0
  [-180, 0] → midpoint -90
  -122.4194 < -90 → left half → bit 0
  [-180, -90] → midpoint -135
  -122.4194 > -135 → right half → bit 1
  ...
  Binary: 01001...

Latitude 37.7749:
  [-90, 90] → midpoint 0
  37.7749 > 0 → right half → bit 1
  [0, 90] → midpoint 45
  37.7749 < 45 → left half → bit 0
  [0, 45] → midpoint 22.5
  37.7749 > 22.5 → right half → bit 1
  ...
  Binary: 10110...
```

---

#### Step 2: Interleave Bits

Alternate bits from longitude and latitude.

```
Longitude: 0 1 0 0 1 ...
Latitude:  1 0 1 1 0 ...

Interleaved: 01 10 01 01 10 ...
             └─ lng, lat, lng, lat, ...
```

---

#### Step 3: Encode to Base32

Convert binary to base32 (0-9, a-z excluding a, i, l, o).

```
Binary: 01100 11101 10100 10110 10010
Base32:    9     q     8     y     y

Geohash: 9q8yy
```

---

### Geohash Precision

Longer geohash = smaller area.

|Length|Cell Width|Cell Height|Example|
|---|---|---|---|
|1|≤ 5,000 km|≤ 5,000 km|9 (covers western US)|
|2|≤ 1,250 km|≤ 625 km|9q (covers California)|
|3|≤ 156 km|≤ 156 km|9q8 (covers Bay Area)|
|4|≤ 39 km|≤ 19.5 km|9q8y (covers San Francisco)|
|5|≤ 4.9 km|≤ 4.9 km|9q8yy (covers neighborhood)|
|6|≤ 1.2 km|≤ 0.6 km|9q8yyk (covers few blocks)|
|7|≤ 153 m|≤ 153 m|9q8yykq (covers single block)|
|8|≤ 38 m|≤ 19 m|9q8yykqz (covers building)|

---

### Proximity Queries with Geohash

#### Find nearby locations

```
User location: 9q8yy

Query:
  SELECT * FROM locations
  WHERE geohash LIKE '9q8yy%'

Result: All locations with geohash starting with "9q8yy"
        (within ~5 km)
```

---

#### Expand search radius

If not enough results, search neighboring cells.

```
User geohash: 9q8yy

Neighbors:
  North: 9q8yz
  South: 9q8yw
  East:  9q8yn
  West:  9q8yq
  NE:    9q8yp
  NW:    9q8yr
  SE:    9q8yj
  SW:    9q8ym

Query:
  SELECT * FROM locations
  WHERE geohash IN ('9q8yy', '9q8yz', '9q8yw', '9q8yn', '9q8yq', '9q8yp', '9q8yr', '9q8yj', '9q8ym')
```

---

### Geohash Limitations

#### Edge cases

Locations near cell boundaries may have different prefixes even if they're close.

```
Location A: 9q8yy (edge of cell)
Location B: 9q8yz (neighboring cell, 100m away)

Shared prefix: "9q8y" (4 characters)
But they're very close!
```

**Solution:** Always check neighboring cells.

---

#### Poles and dateline

Geohash has issues near poles and the international dateline. Cells become distorted.

---

## 3. S2 Geometry

### What is S2?

**S2** is a hierarchical spatial indexing system developed by Google. It maps the Earth to a sphere, then projects the sphere onto a cube, then subdivides the cube into cells.

```
Earth (sphere) → Cube (6 faces) → Cells (hierarchical subdivision)
```

**Key advantage over Geohash:** Cells are more uniform in size and shape.

---

### How S2 Works

#### Step 1: Map Earth to Sphere

Treat Earth as a unit sphere.

---

#### Step 2: Project Sphere to Cube

Project the sphere onto a cube (6 faces: front, back, left, right, top, bottom).

```
        ┌─────┐
        │ Top │
   ┌────┼─────┼────┬─────┐
   │Left│Front│Right│Back│
   └────┼─────┼────┴─────┘
        │Bottom│
        └─────┘
```

---

#### Step 3: Subdivide Faces into Cells

Each face is recursively subdivided into 4 cells (quadtree).

```
Level 0: 6 cells (one per face)
Level 1: 24 cells (4 per face)
Level 2: 96 cells (16 per face)
...
Level 30: 6 × 4^30 cells (very small)
```

---

### S2 Cell ID

Each cell has a unique 64-bit ID.

```
San Francisco:
  Lat: 37.7749, Lng: -122.4194
  S2 Cell ID (level 30): 9291041754864271189
  S2 Cell Token: 808f9b (hex encoding)
```

---

### S2 Cell Levels

|Level|Cell Size|Example|
|---|---|---|
|0|85,000 km|Entire face of cube|
|4|600 km|Large country|
|8|90 km|City|
|12|14 km|Neighborhood|
|16|2 km|Few blocks|
|20|300 m|Single block|
|24|50 m|Building|
|30|1 cm|Very precise|

---

### Proximity Queries with S2

#### Find nearby locations

```
User location: S2 Cell ID 9291041754864271189 (level 30)

Get parent cell at level 16 (2 km radius):
  Parent Cell ID: 9291041754864000000

Query:
  SELECT * FROM locations
  WHERE s2_cell_id >= 9291041754864000000
    AND s2_cell_id < 9291041754865000000
```

---

#### Covering a region

S2 can cover a region (circle, polygon) with a set of cells.

```
Find all locations within 5 km of user:
  1. Create circle with 5 km radius
  2. Cover circle with S2 cells (mix of levels)
  3. Query locations in those cells

Covering:
  [Cell A (level 14), Cell B (level 15), Cell C (level 16), ...]
```

---

### S2 Advantages over Geohash

```
✅ Uniform cell sizes (Geohash cells vary near poles)
✅ Better coverage (S2 covers regions with fewer cells)
✅ Hierarchical (easy to zoom in/out)
✅ No edge case issues (handles poles and dateline)
```

---

### S2 Disadvantages

```
❌ More complex (harder to understand and implement)
❌ 64-bit IDs (vs Geohash's short strings)
❌ Less human-readable (Geohash: "9q8yy", S2: "9291041754864271189")
```

---

## 4. Geohash vs S2

|Feature|Geohash|S2|
|---|---|---|
|Encoding|Base32 string|64-bit integer|
|Cell shape|Rectangle|Quadrilateral (more uniform)|
|Cell size|Varies (worse near poles)|Uniform|
|Precision|8 levels (1-8 chars)|30 levels|
|Human-readable|Yes ("9q8yy")|No (9291041754864271189)|
|Edge cases|Issues near poles/dateline|Handles well|
|Complexity|Simple|Complex|
|Use case|Simple proximity queries|Advanced spatial queries|

---

## 5. Use Cases

### Geohash

```
- Ride-sharing (Uber, Lyft): Find nearby drivers
- Food delivery (DoorDash): Find nearby restaurants
- Social apps (Tinder): Find nearby users
- Real estate (Zillow): Find nearby properties
```

---

### S2

```
- Google Maps: Spatial indexing, routing
- Pokémon Go: Geofencing, spawn locations
- Rideshare (Uber): Advanced routing, ETA calculation
- Geospatial databases (BigQuery GIS): Spatial queries
```

---

## 6. Implementation

### Geohash (Python)

```python
import geohash

# Encode
lat, lng = 37.7749, -122.4194
gh = geohash.encode(lat, lng, precision=5)
print(gh)  # "9q8yy"

# Decode
lat, lng = geohash.decode(gh)
print(lat, lng)  # (37.7749, -122.4194)

# Neighbors
neighbors = geohash.neighbors(gh)
print(neighbors)
# {'n': '9q8yz', 's': '9q8yw', 'e': '9q8yn', 'w': '9q8yq', ...}

# Proximity query
def find_nearby(user_geohash, locations):
    prefix = user_geohash[:4]  # 4-char prefix (~40 km)
    return [loc for loc in locations if loc['geohash'].startswith(prefix)]
```

---

### S2 (Python)

```python
import s2sphere

# Encode
lat, lng = 37.7749, -122.4194
latlng = s2sphere.LatLng.from_degrees(lat, lng)
cell_id = s2sphere.CellId.from_lat_lng(latlng)
print(cell_id.id())  # 9291041754864271189

# Get parent cell (level 16)
parent = cell_id.parent(16)
print(parent.id())

# Cover a region (circle with 5 km radius)
region = s2sphere.Cap.from_axis_angle(
    latlng.to_point(),
    s2sphere.Angle.from_degrees(5 / 111)  # 5 km ≈ 0.045 degrees
)
coverer = s2sphere.RegionCoverer()
coverer.min_level = 14
coverer.max_level = 18
covering = coverer.get_covering(region)
print([cell.id() for cell in covering])
```

---

## 7. Common Interview Questions + Answers

### Q: How do you find nearby locations efficiently?

> "Encode latitude and longitude into a spatial index like Geohash or S2. Nearby locations have similar encodings. For Geohash, locations with the same prefix are nearby. For example, if a user is at geohash '9q8yy', I query all locations with geohashes starting with '9q8yy'. This is much faster than calculating distance to every location. I also check neighboring cells to handle edge cases where nearby locations have different prefixes. This reduces the problem from O(N) to O(log N) with a spatial index."

---

### Q: What's the difference between Geohash and S2?

> "Geohash encodes lat/lng into a base32 string. It's simple and human-readable, but cells vary in size near poles and it has edge case issues. S2 maps the Earth to a cube and subdivides it hierarchically. It has uniform cell sizes, handles poles and the dateline well, and supports advanced spatial queries. Geohash is simpler and good for basic proximity queries. S2 is more complex but better for advanced use cases like routing and geofencing. Google uses S2 for Maps and Pokémon Go."

---

### Q: How do you handle edge cases where nearby locations have different geohashes?

> "Locations near cell boundaries may have different geohash prefixes even if they're close. To handle this, I always check neighboring cells. For a given geohash, I compute the 8 neighbors (north, south, east, west, and diagonals) and query all 9 cells. This ensures I don't miss nearby locations. For example, if the user is at '9q8yy', I query '9q8yy' and its 8 neighbors. This increases the query cost slightly but ensures complete results."

---

### Q: How does Uber use geohashing?

> "Uber uses geohashing to find nearby drivers. When a rider requests a ride, Uber encodes the rider's location into a geohash and queries drivers with matching geohash prefixes. This quickly narrows down the search to drivers in the same area. Uber also checks neighboring cells to ensure drivers near cell boundaries aren't missed. Geohashing allows Uber to scale to millions of drivers and riders without calculating distance to every driver."

---

## 8. Interview Tricks & Pitfalls

### ✅ Trick 1: Mention neighboring cells

Always check neighboring cells to handle edge cases. This shows you understand the limitations of geohashing.

---

### ✅ Trick 2: Know precision levels

Be able to explain how geohash length affects cell size (5 chars ≈ 5 km, 6 chars ≈ 1 km).

---

### ✅ Trick 3: Compare Geohash and S2

Know when to use each. Geohash for simplicity, S2 for advanced use cases.

---

### ❌ Pitfall 1: Assuming geohash prefix is always accurate

Nearby locations may have different prefixes if they're near cell boundaries. Always check neighbors.

---

### ❌ Pitfall 2: Not mentioning spatial index

Geohashing alone isn't enough. You need a spatial index (B-tree, quadtree) for efficient queries.

---

### ❌ Pitfall 3: Ignoring S2

Many candidates only know Geohash. Mentioning S2 shows deeper knowledge.

---

## 9. Quick Reference

```
Problem: Find nearby locations efficiently

Geohash:
  - Encode lat/lng into base32 string
  - Nearby locations have similar prefixes
  - Precision: 1-8 characters (5 chars ≈ 5 km)
  - Simple, human-readable
  - Issues near poles and dateline

S2:
  - Map Earth to cube, subdivide hierarchically
  - 64-bit cell IDs
  - 30 levels (level 16 ≈ 2 km)
  - Uniform cell sizes
  - Handles poles and dateline
  - More complex

Proximity Query:
  Geohash: SELECT * WHERE geohash LIKE '9q8yy%'
  S2:      SELECT * WHERE s2_cell_id BETWEEN start AND end

Edge Cases:
  - Check neighboring cells (8 neighbors)
  - Locations near boundaries may have different prefixes

Geohash vs S2:
  Geohash: Simple, human-readable, good for basic queries
  S2:      Complex, uniform cells, good for advanced queries

Use Cases:
  Geohash: Uber (find drivers), DoorDash (find restaurants)
  S2:      Google Maps (routing), Pokémon Go (geofencing)

Precision:
  Geohash 5 chars: ~5 km
  Geohash 6 chars: ~1 km
  Geohash 7 chars: ~150 m
  S2 level 16:     ~2 km
  S2 level 20:     ~300 m

Implementation:
  Geohash: geohash.encode(lat, lng, precision=5)
  S2:      s2sphere.CellId.from_lat_lng(latlng)
```
