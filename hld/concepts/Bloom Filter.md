## 1. What is a Bloom Filter?

A Bloom filter is a space-efficient probabilistic data structure that tests whether an element is a member of a set.

```
Key properties:
  ✅ Can definitively say "NOT in set" (no false negatives)
  ❌ Can only say "MAYBE in set" (possible false positives)
  ✅ Extremely space-efficient (bits, not bytes per element)
  ✅ O(1) insert and lookup
```

**Use case:** Avoid expensive operations (disk reads, network calls, database queries) for elements that definitely don't exist.

---

## 2. Why Bloom Filters Exist

### Problem: Set membership is expensive

```
Check if user_id exists in database:
  → Query database (10ms)
  → Result: user doesn't exist
  → Wasted 10ms and DB resources

With 10,000 requests/sec for non-existent users:
  → 100,000 wasted DB queries/sec
  → Database overwhelmed
```

### Solution: Bloom filter as a pre-filter

```
Check bloom filter (in-memory, <1μs):
  → "Definitely NOT in database" → return 404 immediately
  → "Maybe in database" → query database

Result:
  99% of invalid requests filtered out before hitting DB
  1% false positives still query DB (acceptable)
```

---

## 3. How Bloom Filters Work

### Data structure

A bloom filter is a bit array of size `m` and `k` independent hash functions.

```
Bit array (m = 16 bits, initially all 0):
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
```

### Insert operation

To insert element `x`:

1. Compute `k` hash values: `h1(x), h2(x), ..., hk(x)`
2. Set bits at those positions to 1

```
Insert "alice" (k = 3 hash functions):
  h1("alice") = 3  → set bit 3 to 1
  h2("alice") = 7  → set bit 7 to 1
  h3("alice") = 11 → set bit 11 to 1

┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 0 │ 0 │ 1 │ 0 │ 0 │ 0 │ 1 │ 0 │ 0 │ 0 │ 1 │ 0 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
```

### Lookup operation

To check if element `x` is in the set:

1. Compute `k` hash values: `h1(x), h2(x), ..., hk(x)`
2. Check if ALL corresponding bits are 1
    - If ANY bit is 0 → definitely NOT in set
    - If ALL bits are 1 → MAYBE in set (could be false positive)

```
Check "alice":
  h1("alice") = 3  → bit 3 is 1 ✓
  h2("alice") = 7  → bit 7 is 1 ✓
  h3("alice") = 11 → bit 11 is 1 ✓
  → All bits are 1 → "MAYBE in set"

Check "bob":
  h1("bob") = 2  → bit 2 is 0 ✗
  → At least one bit is 0 → "DEFINITELY NOT in set"
```

---

## 4. False Positive Rate

As more elements are inserted, more bits are set to 1, increasing the chance of false positives.

### Formula

```
False positive probability:
  p ≈ (1 - e^(-kn/m))^k

Where:
  k = number of hash functions
  n = number of elements inserted
  m = size of bit array
```

### Example

```
m = 1000 bits
k = 3 hash functions
n = 100 elements inserted

p ≈ (1 - e^(-3×100/1000))^3
  ≈ (1 - e^(-0.3))^3
  ≈ (1 - 0.74)^3
  ≈ 0.26^3
  ≈ 0.017 = 1.7% false positive rate
```

### Optimal number of hash functions

```
k_optimal = (m/n) × ln(2)

For m = 1000, n = 100:
  k_optimal = (1000/100) × 0.693 ≈ 7 hash functions
```

More hash functions reduce false positives but increase computation cost. Typical: k = 3-7.

---

## 5. Space Efficiency

Bloom filters are incredibly space-efficient compared to storing the actual set.

### Comparison

```
Store 1 million user IDs:

Hash set (64-bit IDs):
  1M × 8 bytes = 8 MB

Bloom filter (1% false positive rate):
  m = -n × ln(p) / (ln(2))^2
  m = -1M × ln(0.01) / 0.48
  m ≈ 9.6 million bits = 1.2 MB

Space savings: 8 MB → 1.2 MB (85% reduction)
```

For 0.1% false positive rate: 1.8 MB (still 77% smaller).

---

## 6. Real-World Use Cases

### 1. Database query optimization (RocksDB, Cassandra)

Avoid reading SSTables (on-disk files) that definitely don't contain a key.

```
Query: Get key "user:12345"

Without bloom filter:
  → Read SSTable 1 (disk I/O, 10ms) → not found
  → Read SSTable 2 (disk I/O, 10ms) → not found
  → Read SSTable 3 (disk I/O, 10ms) → found
  Total: 30ms

With bloom filter:
  → Check bloom filter for SSTable 1 → "NOT in set" → skip
  → Check bloom filter for SSTable 2 → "NOT in set" → skip
  → Check bloom filter for SSTable 3 → "MAYBE in set" → read (10ms) → found
  Total: 10ms (3x faster)
```

### 2. Cache penetration prevention

Prevent requests for non-existent keys from hitting the database.

```
Request: Get user_id = -1 (doesn't exist)

Without bloom filter:
  → Check cache → miss
  → Query database → not found
  → Return 404
  (Repeat 10,000 times → 10,000 DB queries)

With bloom filter:
  → Check bloom filter → "NOT in set" → return 404 immediately
  (No cache or DB query needed)
```

### 3. Web crawlers (Google, Bing)

Track which URLs have already been crawled without storing billions of URLs.

```
Crawl 1 billion URLs:

Hash set: 1B × 100 bytes/URL = 100 GB
Bloom filter (1% FP): 1.2 GB (98% space savings)

False positives: 1% of URLs incorrectly marked as "already crawled"
  → Acceptable trade-off for massive space savings
```

### 4. Distributed systems (Bitcoin, Ethereum)

Check if a transaction has been seen before without broadcasting the full transaction.

```
Node receives transaction hash:
  → Check bloom filter → "NOT seen" → broadcast to peers
  → Check bloom filter → "MAYBE seen" → check full transaction log
```

### 5. Spam filtering

Check if an email address is in a known spam list.

```
Email from: spam@example.com

Bloom filter: "MAYBE in spam list" → check full spam database
Bloom filter: "NOT in spam list" → deliver immediately
```

---

## 7. Limitations

### 1. No deletions

You cannot remove an element from a standard bloom filter.

```
Problem:
  Insert "alice" → set bits 3, 7, 11
  Insert "bob"   → set bits 3, 9, 14 (bit 3 overlaps)
  Delete "alice" → clear bits 3, 7, 11
  Check "bob"    → bit 3 is now 0 → false negative!
```

**Solution:** Counting Bloom Filter (see below).

### 2. No way to list elements

A bloom filter only answers "is X in the set?" You cannot iterate over all elements.

### 3. False positives increase over time

As more elements are added, the bit array fills up, increasing false positive rate.

```
After inserting n elements:
  50% of bits set → 12.5% false positive rate
  75% of bits set → 42% false positive rate
  90% of bits set → 65% false positive rate
```

**Solution:** Rebuild the bloom filter with a larger bit array when FP rate exceeds threshold.

---

## 8. Variants

### Counting Bloom Filter

Replace each bit with a counter (e.g., 4-bit counter).

```
Standard bloom filter:
┌───┬───┬───┬───┐
│ 0 │ 1 │ 1 │ 0 │  (bits)
└───┴───┴───┴───┘

Counting bloom filter:
┌───┬───┬───┬───┐
│ 0 │ 2 │ 3 │ 0 │  (counters)
└───┴───┴───┴───┘
```

- **Insert:** Increment counters at hash positions
- **Delete:** Decrement counters at hash positions
- **Lookup:** Check if all counters > 0

**Trade-off:** 4x more space (4 bits per counter vs 1 bit), but supports deletions.

### Cuckoo Filter

Alternative to bloom filter with similar space efficiency but supports deletions.

```
Pros:
  ✅ Supports deletions
  ✅ Better lookup performance
  ✅ Lower false positive rate for same space

Cons:
  ❌ More complex implementation
  ❌ Insertions can fail (need to rebuild)
```

Used in: Redis (for set membership checks).

### Scalable Bloom Filter

Dynamically grows by adding new bloom filters when the current one fills up.

```
Layer 1: Bloom filter (1 MB, 1% FP)
  → Fills up after 1M elements

Layer 2: Bloom filter (2 MB, 0.5% FP)
  → Fills up after 2M elements

Layer 3: Bloom filter (4 MB, 0.25% FP)
  → ...

Lookup: Check all layers (OR operation)
```

Maintains low false positive rate as the set grows.

---

## 9. Implementation

### Python example

```python
import hashlib

class BloomFilter:
    def __init__(self, size, num_hashes):
        self.size = size
        self.num_hashes = num_hashes
        self.bit_array = [0] * size
    
    def _hashes(self, item):
        """Generate k hash values for an item."""
        hashes = []
        for i in range(self.num_hashes):
            hash_val = int(hashlib.md5(f"{item}{i}".encode()).hexdigest(), 16)
            hashes.append(hash_val % self.size)
        return hashes
    
    def add(self, item):
        """Add an item to the bloom filter."""
        for hash_val in self._hashes(item):
            self.bit_array[hash_val] = 1
    
    def contains(self, item):
        """Check if an item might be in the set."""
        return all(self.bit_array[h] == 1 for h in self._hashes(item))

# Usage
bf = BloomFilter(size=1000, num_hashes=3)
bf.add("alice")
bf.add("bob")

print(bf.contains("alice"))  # True (definitely added)
print(bf.contains("bob"))    # True (definitely added)
print(bf.contains("charlie")) # False (definitely not added)
print(bf.contains("dave"))   # Maybe False, maybe True (false positive)
```

---

## 10. Common Interview Questions + Answers

### Q: What's the difference between a bloom filter and a hash set?

> "A hash set stores the actual elements and can definitively answer 'is X in the set?' with no false positives. A bloom filter only stores bits and can have false positives — it can say 'definitely NOT in set' or 'MAYBE in set'. The trade-off is space: a bloom filter uses 10-20x less memory than a hash set. Use a bloom filter when you can tolerate false positives and need to save space."

### Q: Why can't you delete from a standard bloom filter?

> "Because multiple elements can set the same bit to 1. If you delete an element by clearing its bits, you might accidentally clear bits that are also used by other elements, causing false negatives. Counting bloom filters solve this by using counters instead of bits, allowing safe decrements on deletion."

### Q: How do you choose the bloom filter size and number of hash functions?

> "It depends on the expected number of elements (n) and acceptable false positive rate (p). The optimal size is m = -n × ln(p) / (ln(2))^2, and the optimal number of hash functions is k = (m/n) × ln(2). For example, for 1 million elements with 1% false positive rate, you need about 1.2 MB and 7 hash functions."

### Q: Where are bloom filters used in real systems?

> "Cassandra and RocksDB use bloom filters to avoid reading SSTables that don't contain a key. Google Chrome uses bloom filters to check if a URL is in the malicious site list. Bitcoin uses bloom filters for lightweight SPV clients to filter transactions. Web crawlers use them to track visited URLs without storing billions of URLs in memory."

---

## 11. Interview Tricks & Pitfalls

### ✅ Trick 1: Emphasize "no false negatives"

The key property is that bloom filters never say "not in set" when the element is actually in the set. This makes them safe for filtering — you might do extra work (false positive), but you'll never miss a real element.

### ✅ Trick 2: Mention space savings

Always quantify the space savings: "10-20x smaller than a hash set" or "1.2 MB for 1 million elements at 1% FP rate vs 8 MB for a hash set."

### ✅ Trick 3: Know the real-world use cases

Mentioning Cassandra, RocksDB, or web crawlers shows you understand where bloom filters are actually used, not just the theory.

### ❌ Pitfall 1: Confusing false positives and false negatives

Bloom filters have false positives (say "maybe in set" when it's not), never false negatives (never say "not in set" when it is). Don't mix these up.

### ❌ Pitfall 2: Thinking bloom filters replace databases

Bloom filters are a pre-filter to avoid expensive operations. You still need the actual database or hash set for definitive answers on "maybe" results.

### ❌ Pitfall 3: Forgetting that FP rate increases over time

As the bloom filter fills up, false positive rate increases. In production, you need to monitor the FP rate and rebuild the filter with a larger size when it exceeds your threshold.

---

## 12. Quick Reference

```
What is a bloom filter?
  Probabilistic set membership test
  No false negatives, possible false positives
  Extremely space-efficient (bits, not bytes)

Operations:
  Insert: O(k) — set k bits to 1
  Lookup: O(k) — check if all k bits are 1
  Delete: Not supported (use counting bloom filter)

Space efficiency:
  Hash set: 8 bytes per element
  Bloom filter (1% FP): ~1.2 bytes per element
  Space savings: 85%

False positive rate:
  p ≈ (1 - e^(-kn/m))^k
  Increases as more elements are added
  Rebuild with larger size when FP rate too high

Optimal parameters:
  m = -n × ln(p) / (ln(2))^2  (bit array size)
  k = (m/n) × ln(2)           (number of hash functions)

Use cases:
  Database: Skip SSTables that don't contain key (Cassandra, RocksDB)
  Cache: Prevent cache penetration attacks
  Web crawlers: Track visited URLs
  Blockchain: Filter transactions
  Spam: Check email against spam list

Variants:
  Counting bloom filter: Supports deletions (4x space)
  Cuckoo filter: Better performance, supports deletions
  Scalable bloom filter: Grows dynamically

Limitations:
  ❌ No deletions (standard version)
  ❌ Cannot list elements
  ❌ False positive rate increases over time
  ✅ Never false negatives
```
