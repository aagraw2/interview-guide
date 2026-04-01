## 1. What is a Write-Ahead Log?

A Write-Ahead Log (WAL) is an append-only log where every change to the database is written **before** the actual data pages are modified. It's the foundation of crash recovery, replication, and change data capture in modern databases.

The core principle: **log the change first, apply it later**.

```
Write operation:
  1. Append change to WAL (durable, sequential write)
  2. Acknowledge to client
  3. Apply change to data pages in memory (async)
  4. Eventually flush dirty pages to disk
```

If the database crashes between steps 2 and 4, the WAL contains all committed changes. On restart, the database replays the WAL to reconstruct the correct state.

---

## 2. Why WAL Exists

### Problem: Random writes are slow

Updating data pages directly requires random disk I/O — seeking to the right page, reading it, modifying it, writing it back. This is slow.

```
Direct update to data page:
  Seek to page 4,582 → read → modify → write back
  Latency: ~10ms per write (disk seek time)
```

### Solution: Sequential append-only log

WAL converts random writes into sequential appends. Sequential writes are 100x faster than random writes on spinning disks and still significantly faster on SSDs.

```
WAL append:
  Append to end of log file
  Latency: ~0.1ms (no seek, just append)
```

The database can acknowledge the write immediately after the WAL entry is durable. The actual data page update happens asynchronously in the background.

---

## 3. How WAL Works

### Write path

```
Client: UPDATE users SET balance = 500 WHERE id = 123

Database:
  1. Generate log entry:
       LSN: 1001
       Transaction ID: 42
       Operation: UPDATE users SET balance = 500 WHERE id = 123
       Old value: 400
       New value: 500

  2. Append to WAL (fsync to disk)

  3. Return success to client

  4. (Background) Apply change to in-memory buffer pool

  5. (Background) Eventually flush dirty page to disk
```

### Log Sequence Number (LSN)

Every WAL entry has a monotonically increasing LSN. The LSN tracks:

- Which changes have been written to the log
- Which changes have been applied to data pages
- Which changes have been flushed to disk

```
WAL:
  LSN 1000: INSERT INTO orders ...
  LSN 1001: UPDATE users ...
  LSN 1002: DELETE FROM cart ...

Current state:
  WAL written up to: LSN 1002
  Applied to memory: LSN 1001
  Flushed to disk:   LSN 1000
```

---

## 4. Crash Recovery

When the database crashes and restarts, it uses the WAL to recover.

### Recovery process

```
1. Read the last checkpoint LSN from disk
   (checkpoint = last point where all changes were flushed)

2. Replay WAL from checkpoint LSN to end
   - Redo committed transactions
   - Undo uncommitted transactions

3. Database is now consistent
```

### Example

```
WAL:
  LSN 1000: BEGIN txn 10
  LSN 1001: UPDATE users SET balance = 500 WHERE id = 1 (txn 10)
  LSN 1002: COMMIT txn 10
  LSN 1003: BEGIN txn 11
  LSN 1004: INSERT INTO orders ... (txn 11)
  [CRASH — txn 11 never committed]

Recovery:
  - Replay LSN 1000-1002 → redo txn 10 (committed)
  - Rollback LSN 1003-1004 → undo txn 11 (uncommitted)
```

This is called **ARIES recovery** (Algorithm for Recovery and Isolation Exploiting Semantics) — used by PostgreSQL, MySQL InnoDB, SQL Server.

---

## 5. Checkpointing

Writing every change to the WAL forever would make the log infinitely large. Checkpointing solves this.

### What is a checkpoint?

A checkpoint is a point in time where:

1. All dirty pages in memory are flushed to disk
2. The current LSN is recorded as the checkpoint LSN
3. WAL entries before the checkpoint can be safely deleted

```
Timeline:
  LSN 1000 ──────────────────────────────────────────────────────────
  LSN 1500 [CHECKPOINT] ← all changes up to 1500 are on disk
  LSN 2000 ──────────────────────────────────────────────────────────

After checkpoint:
  - WAL entries 1000-1500 can be deleted (already on disk)
  - Recovery only needs to replay from LSN 1500 onward
```

### Checkpoint frequency

Too frequent: Wastes I/O flushing pages. Too infrequent: WAL grows large, recovery takes longer.

Typical: Every few minutes or when WAL reaches a size threshold (e.g., 1 GB).

---

## 6. WAL in Replication

WAL is the foundation of database replication. The primary sends WAL entries to replicas, which replay them to stay in sync.

### Streaming replication (PostgreSQL)

```
Primary:
  1. Write to WAL
  2. Stream WAL entries to replicas
  3. Acknowledge to client (after local WAL write)

Replica:
  1. Receive WAL entries
  2. Replay them to apply changes
  3. Replica is now consistent with primary (with some lag)
```

### Synchronous vs asynchronous replication

**Asynchronous:** Primary acknowledges immediately after writing to its own WAL. Replicas catch up later. Fast, but replicas may lag.

**Synchronous:** Primary waits for at least one replica to acknowledge WAL receipt before returning success. Slower, but guarantees replicas are up-to-date.

```
Async:
  Client → Primary → WAL → ACK to client (fast)
                  → Stream to replicas (async)

Sync:
  Client → Primary → WAL → Stream to replicas → Wait for ACK → ACK to client
```

---

## 7. Change Data Capture (CDC)

WAL enables Change Data Capture — streaming database changes to downstream systems without polling.

### How CDC works

```
Database WAL:
  LSN 1001: INSERT INTO orders (id=1, user_id=5, total=100)
  LSN 1002: UPDATE users SET balance = 400 WHERE id = 5

CDC tool (Debezium):
  1. Tail the WAL
  2. Parse log entries
  3. Publish to Kafka:
       Topic: orders
       Event: {"op": "INSERT", "id": 1, "user_id": 5, "total": 100}
```

### Use cases

- **Event-driven architecture:** Trigger downstream services when data changes
- **Data pipelines:** Sync database changes to data warehouse, search index, cache
- **Audit logging:** Track all changes for compliance

### CDC tools

- **Debezium:** Open-source CDC for MySQL, PostgreSQL, MongoDB, SQL Server
- **AWS DMS:** Managed CDC for AWS databases
- **Maxwell:** MySQL CDC to Kafka
- **Datastream (GCP):** Managed CDC for Cloud SQL

---

## 8. WAL Internals: Log Structure

### Log entry format

```
┌─────────────────────────────────────────────────────────┐
│ LSN | Txn ID | Type | Table | Old Value | New Value | CRC │
├─────────────────────────────────────────────────────────┤
│ 1001│   42   │UPDATE│ users │ bal=400   │ bal=500   │ ... │
└─────────────────────────────────────────────────────────┘
```

- **LSN:** Log sequence number (monotonic)
- **Txn ID:** Transaction identifier
- **Type:** INSERT, UPDATE, DELETE, COMMIT, ABORT
- **Table:** Which table was modified
- **Old/New Value:** Before and after state (for undo/redo)
- **CRC:** Checksum to detect corruption

### Log segments

WAL is split into fixed-size segments (e.g., 16 MB in PostgreSQL). Old segments are deleted after checkpointing.

```
WAL directory:
  000000010000000000000001  (segment 1, LSN 0-16MB)
  000000010000000000000002  (segment 2, LSN 16MB-32MB)
  000000010000000000000003  (segment 3, LSN 32MB-48MB)
```

---

## 9. WAL Configuration Trade-offs

### fsync frequency

After writing to WAL, the database calls `fsync()` to ensure the log is on disk (not just in OS buffer cache).

```
fsync = on  (default)
  Every commit waits for fsync → durable, but slower

fsync = off
  OS flushes WAL asynchronously → faster, but risk of data loss on crash
```

**Interview answer:** Always use `fsync = on` in production. The performance cost is acceptable for durability.

### WAL buffer size

WAL is first written to an in-memory buffer, then flushed to disk.

```
Small buffer → frequent flushes → more I/O
Large buffer → fewer flushes → better throughput, but more data loss on crash
```

Typical: 16 MB buffer.

### WAL compression

PostgreSQL and MySQL support WAL compression to reduce disk I/O and replication bandwidth.

```
Uncompressed: 1 GB WAL/hour
Compressed:   200 MB WAL/hour (5x reduction)

Trade-off: CPU cost for compression vs I/O savings
```

---

## 10. WAL in Different Databases

### PostgreSQL

- WAL is mandatory (no way to disable)
- Stored in `pg_wal/` directory
- Supports streaming replication via WAL
- Logical replication decodes WAL into row-level changes

### MySQL (InnoDB)

- Called "redo log" (same concept as WAL)
- Stored in `ib_logfile0`, `ib_logfile1` (circular buffer)
- Binary log (binlog) is a separate log for replication
- Binlog is statement-based or row-based; redo log is physical

### MongoDB

- Called "oplog" (operations log)
- Capped collection in `local.oplog.rs`
- Replicas tail the oplog to stay in sync
- Oplog is idempotent (can replay multiple times safely)

### Kafka

- Kafka is essentially a distributed WAL
- Each partition is an append-only log
- Consumers read from the log at their own pace
- Log compaction removes old entries for the same key

---

## 11. Common Interview Questions + Answers

### Q: Why does a database use a WAL instead of writing directly to data pages?

> "Sequential writes to a WAL are much faster than random writes to data pages — especially on spinning disks, but even on SSDs. The WAL lets the database acknowledge writes quickly while deferring the expensive random I/O to background processes. It also provides a durable record for crash recovery and replication."

### Q: What happens if the database crashes before a WAL entry is flushed to disk?

> "If the WAL entry wasn't flushed (fsync'd) before the crash, the transaction is lost — the client would have received an error or timeout, not a success acknowledgment. Databases only return success after the WAL is durable. If the crash happens after the WAL is flushed but before data pages are updated, recovery replays the WAL to reconstruct the correct state."

### Q: How does WAL enable replication?

> "The primary database streams WAL entries to replicas. Replicas replay the log to apply the same changes, keeping them in sync. This is more efficient than replicas independently executing the same queries — they just apply the already-computed changes. Synchronous replication waits for replica acknowledgment before committing; asynchronous replication doesn't wait, allowing replicas to lag."

### Q: What's the difference between WAL and binlog in MySQL?

> "The redo log (WAL) is InnoDB's physical log of page-level changes, used for crash recovery. The binlog is MySQL's logical log of statement or row-level changes, used for replication and point-in-time recovery. Both are append-only logs, but they serve different purposes. In a replicated setup, the primary writes to both; replicas read from the binlog."

---

## 12. Interview Tricks & Pitfalls

### ✅ Trick 1: WAL is the foundation of durability

When discussing ACID properties, mention that the 'D' (durability) is implemented via WAL. This shows you understand the connection between theory and implementation.

### ✅ Trick 2: WAL enables multiple features

Don't just say "WAL is for crash recovery." Mention it also enables replication, CDC, point-in-time recovery, and audit logging. This demonstrates breadth.

### ✅ Trick 3: Checkpointing prevents infinite log growth

Always mention checkpointing when discussing WAL. Without it, the log would grow forever. This shows you think about operational concerns.

### ❌ Pitfall 1: Confusing WAL with transaction log

They're the same thing. Different databases use different names: WAL (PostgreSQL), redo log (MySQL), oplog (MongoDB), commit log (Cassandra). Don't get confused by terminology.

### ❌ Pitfall 2: Thinking WAL is only for crash recovery

WAL is also critical for replication and CDC. If you only mention crash recovery, you're missing half the picture.

### ❌ Pitfall 3: Forgetting fsync

Writing to the WAL buffer isn't enough — it must be flushed to disk (fsync) for durability. Mentioning fsync shows you understand the OS-level details.

---

## 13. Quick Reference

```
What is WAL?
  Append-only log of all database changes
  Written BEFORE data pages are modified
  Enables crash recovery, replication, CDC

Why WAL?
  Sequential writes >> random writes (100x faster)
  Durability without waiting for data page flush
  Foundation for replication and change streaming

Write path:
  1. Append to WAL (fsync)
  2. ACK to client
  3. Apply to memory (async)
  4. Flush to disk (async)

Crash recovery:
  1. Read last checkpoint LSN
  2. Replay WAL from checkpoint to end
  3. Redo committed txns, undo uncommitted txns

Checkpointing:
  Flush all dirty pages to disk
  Record checkpoint LSN
  Delete old WAL segments

Replication:
  Primary streams WAL to replicas
  Replicas replay WAL to stay in sync
  Sync replication waits for replica ACK

CDC (Change Data Capture):
  Tail WAL to stream changes to Kafka/downstream
  Enables event-driven architecture
  Tools: Debezium, AWS DMS, Maxwell

Configuration:
  fsync = on (always in production)
  WAL buffer size (typically 16 MB)
  Checkpoint frequency (balance recovery time vs I/O)

Database names:
  PostgreSQL → WAL
  MySQL      → redo log + binlog
  MongoDB    → oplog
  Cassandra  → commit log
  Kafka      → partition log
```
