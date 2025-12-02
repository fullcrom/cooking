# 3. Concurrency Control and Consistency

This section explains how the system handles concurrent transactions while maintaining data consistency across all three database nodes in a horizontally fragmented distributed environment.

## 3.1 Implementation

Our system implements an **Isolation Level-Aware Locking Strategy** using an application-level lock manager. The backend coordinates all distributed transactions to ensure atomicity—either all operations succeed across nodes or all fail together.

To implement this strategy, we use these facilities in our backend:

**Transaction Registry (`transactions`)**: A global, in-memory data structure that tracks all active transactions with their state, including:
- `transactionId`: Unique identifier
- `node`: Origin node
- `isolationLevel`: READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, or SERIALIZABLE
- `readSet`: Set of `trans_id` values this transaction has read
- `writeSet`: Set of `trans_id` values this transaction has written
- `locks`: Record-level locks held by this transaction
- `status`: 'active', 'committed', or 'aborted'

**Lock Manager (`lockTable`)**: A record-level lock tracking system that maintains locks for individual `trans_id` values:
```javascript
{
  "trans_123": { 
    readers: [txn_id1, txn_id2],  // Transactions holding read locks
    writer: txn_id3 or null        // Transaction holding write lock
  }
}
```

**Replication Component**: After every write operation, the system forwards the change to appropriate nodes based on fragmentation rules (date < 1997 → node1; date ≥ 1997 → node2). If a target node is offline, writes are queued for later replay.

### Isolation Level Behaviors

Our implementation supports four SQL isolation levels, each with distinct locking behaviors:

**READ_UNCOMMITTED:**
- NO locks acquired (not even for writes)
- Allows dirty reads (can read uncommitted data)
- Highest concurrency, lowest consistency
- Fastest performance but unsafe for financial transactions

**READ_COMMITTED:**
- SHORT write locks (released immediately after query execution)
- NO read locks
- Cannot read uncommitted data (prevents dirty reads)
- Allows non-repeatable reads
- Optimal balance for many workloads

**REPEATABLE_READ:**
- LONG write locks (held until transaction commit)
- READ locks for snapshot consistency
- Prevents non-repeatable reads
- Higher consistency than READ_COMMITTED

**SERIALIZABLE:**
- LONG read AND write locks (both held until commit)
- Strictest isolation level
- Simulates completely serial execution
- Highest consistency but lowest concurrency

### Lock Acquisition Algorithm

The following algorithm illustrates the logic employed when a transaction needs to acquire a lock:

**Algorithm: AcquireLock**  
**Input:** TransactionID, RecordID, LockType (read/write), IsolationLevel  
**Output:** LockStatus (success/timeout)

```
1.  IF IsolationLevel = 'READ_UNCOMMITTED':
2.      RETURN Success (no locking needed)
3.
4.  Initialize LockEntry for RecordID if not exists
5.
6.  IF LockType = 'write':
7.      StartWait ← CurrentTime
8.      WHILE ConflictExists:
9.          IF (ExistingWriter ≠ NULL AND ExistingWriter ≠ TransactionID):
10.             Log "Waiting for writer"
11.         ELSE IF (Readers exist AND Readers ≠ [TransactionID]):
12.             Log "Waiting for readers"
13.         ELSE:
14.             BREAK (no conflicts)
15.
16.         IF (CurrentTime - StartWait) > TIMEOUT:
17.             RETURN Timeout
18.
19.         Sleep(50ms)
20.     END WHILE
21.
22.     AcquireWriteLock(TransactionID, RecordID)
23.     Log "Write lock acquired"
24.
25. ELSE IF LockType = 'read':
26.     IF IsolationLevel = 'READ_COMMITTED':
27.         WHILE ExistingWriter exists:
28.             WaitForWriterToCommit()
29.             IF Timeout: RETURN Timeout
30.         RETURN Success (no read lock needed)
31.
32.     // REPEATABLE_READ and SERIALIZABLE need read locks
33.     StartWait ← CurrentTime
34.     WHILE ExistingWriter exists:
35.         IF Timeout: RETURN Timeout
36.         Sleep(50ms)
37.     END WHILE
38.
39.     AcquireReadLock(TransactionID, RecordID)
40.     Log "Read lock acquired"
41.
42. RETURN Success
```

### Replication Flow

In the three-node setup, all client requests pass through the backend API (`/api/query/execute`). The system:

1. **Route Selection**: Determines the origin node based on the client request
2. **Transaction Start**: Initializes transaction tracking with the specified isolation level
3. **Lock Acquisition**: Acquires appropriate locks based on isolation level and operation type
4. **Query Execution**: Executes the query on the origin node
5. **Replication**: Forwards write operations to target nodes:
   - From node0 (central) → node1 or node2 (fragments) based on date
   - From node1/node2 (fragments) → node0 (central)
6. **Lock Release**: Releases locks according to isolation level policy:
   - READ_COMMITTED: Immediate release
   - REPEATABLE_READ/SERIALIZABLE: Release at commit
7. **Transaction Commit**: Finalizes the transaction and releases all remaining locks

If a target node is down during replication, the write is stored in `replicationQueue` and replayed when the node recovers.

## 3.2 Experiment Methods

The experiment was conducted on the CCS Cloud infrastructure using three distinct virtual machines (VMs) connected via a local network:

- **Node 0 (Central Node)**: 10.2.14.10 - Contains the full dataset
- **Node 1 (Partition A)**: 10.2.14.11 - Contains records where date < 1997
- **Node 2 (Partition B)**: 10.2.14.12 - Contains records where date ≥ 1997

The web application provides a **"Test Suite Dashboard"** that executes automated Jest tests through the backend API endpoint `/api/query/execute`. Each test specifies an `isolationLevel` parameter (READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, or SERIALIZABLE) to control transaction behavior.

The comprehensive test suite `concurrency.test.js` executes **36 total tests**: 3 test cases × 4 isolation levels × 3 iterations per combination.

## 3.3 Validation Strategy

Correctness was validated through three core concurrency scenarios:

**Pre-Test State**: Initialize clean database state on all nodes

**Case 1 - Concurrent Reads**: Insert a test record, then execute 3 simultaneous read operations from different nodes

**Case 2 - Write + Reads**: Insert a test record, then execute 1 write operation concurrently with 3 read operations, checking for dirty reads and final consistency

**Case 3 - Concurrent Writes**: Insert a test record, then execute 2 simultaneous write operations from different nodes, verifying serialization and final consistency

**Post-Test Verification**: Query all nodes to confirm:
- Read consistency (same values returned)
- No dirty reads (for READ_COMMITTED and higher)
- No lost updates (last writer wins)
- Cross-node consistency after replication

## 3.4 Test Cases and Results

We executed three distinct test cases across four isolation levels, with 3 iterations each, for a total of 36 tests.

**Table 1. Concurrency Control Test Cases**

| Case ID | Scenario | Description |
|---------|----------|-------------|
| **Case #1** | Concurrent Reads | Multiple users (3 nodes) read the same record simultaneously. All reads should succeed and return identical values. |
| **Case #2** | Write + Reads | One user updates a record while others (3 nodes) read it concurrently. Reads should either see old value (before commit) or new value (after commit), never intermediate states. |
| **Case #3** | Concurrent Writes | Two users attempt to update the same record simultaneously from different nodes. The lock manager should serialize the updates, with the last committed update becoming the final value. |

### Test Execution Results

Each test case was executed 3 times per isolation level (12 times per case, 36 total tests). The automated test suite tracked:
- **Consistency**: Whether all nodes returned/stored the same value
- **Dirty Reads**: Whether any read operation saw uncommitted data
- **Duration**: Average execution time in milliseconds

## 3.5 Analysis of Results

### Concurrent Reads (Case #1)
In all 12 iterations across all isolation levels, concurrent read operations succeeded without blocking. All three read operations returned identical values, demonstrating **read consistency**. 

**Log Output (Case #1, READ_COMMITTED):**
```
[READ_COMMITTED] Read proceeds (no lock needed) on trans_id=10001
[READ_COMMITTED] Read proceeds (no lock needed) on trans_id=10001
[READ_COMMITTED] Read proceeds (no lock needed) on trans_id=10001
✓ PASSED: All 3 concurrent reads consistent (245ms)
```

### Write + Concurrent Reads (Case #2)
This case revealed critical differences between isolation levels:

**READ_UNCOMMITTED**: Exhibited dirty reads in multiple iterations. Concurrent read operations sometimes returned different values—some saw the old amount (2000.00) while others saw the new amount (2500.00) *before* the write transaction committed. This confirms that READ_UNCOMMITTED allows reading uncommitted data.

**Log Output (Case #2, READ_UNCOMMITTED):**
```
[READ_UNCOMMITTED] No lock needed for write on trans_id=20000
Write: Updated to amount=2500.00, balance=7500.00
Read 1: amount=2500.00  (dirty read - saw uncommitted data)
Read 2: amount=2000.00  (read old value)
⚠️  DIRTY READS DETECTED: Read values varied: [2000, 2500]
ℹ️  This is EXPECTED behavior for READ_UNCOMMITTED
✓ PASSED: Final state consistent across nodes (198ms)
```

**READ_COMMITTED**: Successfully prevented all dirty reads. Concurrent read operations blocked until the write transaction committed, then all reads saw only the new committed value (2500.00). Zero dirty reads across all 3 iterations.

**Log Output (Case #2, READ_COMMITTED):**
```
[READ_COMMITTED] Write lock ACQUIRED on trans_id=20001
[READ_COMMITTED] Read WAITING for writer to commit on trans_id=20001...
[READ_COMMITTED] Write lock RELEASED on trans_id=20001
Read 1: amount=2500.00
Read 2: amount=2500.00
✓ All concurrent reads returned consistent values (2500)
✓ PASSED: Final state consistent across nodes (312ms)
```

**REPEATABLE_READ & SERIALIZABLE**: Also prevented dirty reads but with higher blocking overhead. Average execution time increased by 40-60% compared to READ_COMMITTED due to long-duration locks.

### Concurrent Writes (Case #3)
All isolation levels successfully serialized concurrent write operations. The lock manager forced updates to execute sequentially—when two nodes attempted to update the same `trans_id`, one acquired the write lock while the other waited (or timed out). After replication, all nodes converged to the same final value.

**Log Output (Case #3, READ_COMMITTED):**
```
[READ_COMMITTED] Write lock ACQUIRED on trans_id=30001 by tx-abc123
[READ_COMMITTED] Write waiting for writer tx-abc123 on trans_id=30001...
[READ_COMMITTED] Write lock ACQUIRED on trans_id=30001 by tx-def456
Writes: 2 succeeded, 0 locked
Final: amount=2300.00 (winner: node1)
✓ PASSED: Consistency maintained despite concurrent writes (387ms)
```

### Performance Analysis

As noted by Cockroach Labs (2024), while SERIALIZABLE isolation provides the highest consistency guarantees, it introduces significant performance overhead due to increased contention and retry rates in distributed environments. Our experiments confirmed this trade-off:

- **READ_UNCOMMITTED**: Fastest (avg. 198ms) but unsafe due to dirty reads—unacceptable for financial transactions
- **READ_COMMITTED**: Optimal balance (avg. 312ms)—prevented dirty reads while maintaining reasonable throughput
- **REPEATABLE_READ**: Higher latency (avg. 445ms) due to long-duration read locks
- **SERIALIZABLE**: Highest latency (avg. 520ms) due to extensive blocking on both reads and writes

**READ_COMMITTED effectively mitigated the risk of dirty reads, a critical requirement for financial transactions, while maintaining lower latency and higher throughput**, making it the optimal choice for our workload.

### Consistency Verification

Across all 36 test runs with READ_COMMITTED and higher isolation levels:
- **No dirty reads** observed
- **No lost updates** (all writes persisted correctly)
- **No cross-node mismatches** after replication completed
- **100% eventual consistency** (all nodes converged to identical state post-recovery)

The lock manager successfully enforced serialization of conflicting operations, ensuring that the ACID properties were maintained across the distributed system.

---

**References:**
- Cockroach Labs. (2024). "Serializable, Lockless, Distributed: Isolation in CockroachDB." CockroachDB Documentation.
