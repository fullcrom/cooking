# 3. Concurrency Control and Consistency

This section explains how the system handles concurrent transactions while maintaining data consistency across all three database nodes in a horizontally fragmented distributed environment.

## 3.1 Implementation

Our system implements an Isolation Level-Aware Locking Strategy using an application-level lock manager. The backend coordinates all distributed transactions to ensure atomicity, meaning either all operations succeed across nodes or all fail together. This approach provides fine-grained control over transaction behavior based on the chosen SQL isolation level, allowing the system to balance consistency requirements with performance needs.

To implement this strategy, we use these facilities in our backend: a Transaction Registry that maintains a global, in-memory data structure tracking all active transactions with their state, including the transaction identifier, origin node, isolation level (READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, or SERIALIZABLE), the set of records this transaction has read and written, record-level locks held by the transaction, and status indicators showing whether the transaction is active, committed, or aborted. A Lock Manager tracks locks for individual trans_id values, maintaining separate lists of readers and writers for each record, with the structure allowing multiple transactions to hold shared read locks while ensuring only one transaction can hold an exclusive write lock at any time. The Replication Component forwards every write operation to appropriate nodes based on fragmentation rules, where records with dates before 1997 are sent to node1 and records with dates on or after 1997 are sent to node2. If a target node is offline during replication, writes are queued for later replay when the node recovers.

Our implementation supports four SQL isolation levels, each with distinct locking behaviors that directly impact both consistency guarantees and system performance. READ_UNCOMMITTED acquires no locks whatsoever, not even for write operations, which allows dirty reads where transactions can see uncommitted data from other transactions. This provides the highest concurrency and fastest performance but is unsafe for financial transactions where data accuracy is critical. READ_COMMITTED uses short-duration write locks that are released immediately after query execution and requires no read locks at all, preventing dirty reads by forcing readers to wait for active writers to commit but still allowing non-repeatable reads where the same query executed twice might return different results. This level provides an optimal balance for many workloads, particularly in scenarios requiring protection against dirty reads without the overhead of long-duration locks. REPEATABLE_READ employs long-duration write locks held until transaction commit along with read locks for snapshot consistency, preventing both dirty reads and non-repeatable reads by ensuring that once a transaction reads a value, subsequent reads within that transaction will see the same value. SERIALIZABLE uses long-duration read and write locks, with both types held until commit, representing the strictest isolation level that simulates completely serial execution, providing the highest consistency guarantees but the lowest concurrency due to extensive blocking.

The following algorithm illustrates the logic employed when a transaction needs to acquire a lock. For READ_UNCOMMITTED isolation, the system immediately returns success without acquiring any locks, as this level bypasses concurrency control entirely. For write operations at other isolation levels, the system first checks if any conflicting locks exist—either another transaction holding a write lock or other transactions holding read locks—and enters a waiting loop if conflicts are detected, periodically checking if the conflicting transactions have released their locks while enforcing a timeout to prevent indefinite waiting. Once conflicts clear or if no conflicts existed, the system acquires the write lock and records it in both the lock table and the transaction's lock registry. For read operations, READ_COMMITTED handles them specially by waiting only for any active writer to commit without actually acquiring a read lock, thus preventing dirty reads while minimizing lock overhead. REPEATABLE_READ and SERIALIZABLE must acquire actual read locks, so the system waits for any existing writer to release its lock, then acquires a shared read lock that will be held according to the isolation level's duration policy.

Algorithm: AcquireLock
Input: TransactionID, RecordID, LockType (read/write), IsolationLevel
Output: LockStatus (success/timeout)

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

In the three-node setup, all client requests pass through the backend API endpoint. The system first determines the origin node based on the client request, then initializes transaction tracking with the specified isolation level. Next, it acquires appropriate locks based on the isolation level and operation type using the algorithm described above. After acquiring necessary locks, the system executes the query on the origin node and immediately forwards write operations to target nodes—specifically, from node0 (central) to either node1 or node2 (fragments) based on the record's date field, or from node1 or node2 (fragments) back to node0 (central). The system then releases locks according to the isolation level policy, where READ_COMMITTED releases locks immediately after query execution while REPEATABLE_READ and SERIALIZABLE hold locks until transaction commit. Finally, the transaction is committed, which finalizes all changes and releases any remaining locks held by the transaction. If a target node is down during replication, the write is stored in a replicationQueue and replayed when the node recovers, ensuring eventual consistency across all nodes.

## 3.2 Experiment Methods

The experiment was conducted on the CCS Cloud infrastructure using three distinct virtual machines connected via a local network. Each node runs a full stack of the application: Node 0 (Central Node) at 10.2.14.10 contains the full dataset, Node 1 (Partition A) at 10.2.14.11 contains records where date is less than 1997, and Node 2 (Partition B) at 10.2.14.12 contains records where date is greater than or equal to 1997. The web application provides a Test Suite Dashboard that interacts with the backend API endpoint /api/query/execute, where each test specifies an isolationLevel parameter (READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, or SERIALIZABLE) to control transaction behavior. The comprehensive test suite concurrency.test.js executes 36 total tests, organized as 3 test cases multiplied by 4 isolation levels multiplied by 3 iterations per combination, providing thorough coverage of concurrency scenarios under different consistency guarantees.

## 3.3 Validation Strategy

Correctness was validated through three core concurrency scenarios:

**Pre-Test State**: Initialize clean database state on all nodes

**Case 1 - Concurrent Reads**: Insert a test record, then execute 3 simultaneous read operations from different nodes

## 3.3 Validation Strategy

Correctness was validated through three core concurrency scenarios, each designed to test different aspects of the locking mechanism and isolation level behavior. Before each test, the system initializes a clean database state on all nodes to ensure no residual data affects the results. Case 1 tests concurrent reads by inserting a test record and then executing 3 simultaneous read operations from different nodes to verify that multiple readers can access the same data without blocking each other. Case 2 tests write plus reads by inserting a test record and then executing 1 write operation concurrently with 3 read operations, specifically checking for dirty reads where readers might see uncommitted data and verifying final consistency after all operations complete. Case 3 tests concurrent writes by inserting a test record and then executing 2 simultaneous write operations from different nodes, verifying that the lock manager properly serializes the updates and ensures final consistency. Post-test verification queries all nodes to confirm read consistency where all nodes return the same values, the absence of dirty reads for READ_COMMITTED and higher isolation levels, no lost updates with the last writer wins policy properly enforced, and cross-node consistency after replication completes.
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
## 3.4 Test Cases and Results

We executed three distinct test cases across four isolation levels, with 3 iterations each, for a total of 36 tests. Each test case was executed 3 times per isolation level, resulting in 12 tests per case and 36 total tests. The automated test suite tracked consistency to determine whether all nodes returned or stored the same value, monitored for dirty reads to detect if any read operation saw uncommitted data, and measured duration to record average execution time in milliseconds for performance analysis.

Table 1. Concurrency Control Test Cases
Case ID
Scenario
Description
Case #1
Concurrent Reads
Multiple users (3 nodes) read the same record simultaneously. All reads should succeed and return identical values, demonstrating that the system allows multiple concurrent readers without blocking.
Case #2
Write + Reads
One user updates a record while others (3 nodes) read it concurrently. Reads should either see the old value (before commit) or new value (after commit), never intermediate states, with behavior varying by isolation level.
Case #3
Concurrent Writes
Two users attempt to update the same record simultaneously from different nodes. The lock manager should serialize the updates, with the last committed update becoming the final value across all nodes after replication.d=20000
Write: Updated to amount=2500.00, balance=7500.00
Read 1: amount=2500.00  (dirty read - saw uncommitted data)
## 3.5 Analysis of Results

Concurrent Reads (Case #1) In all 12 iterations across all isolation levels, concurrent read operations succeeded without blocking, with all three read operations returning identical values and demonstrating read consistency. Log Output (Case #1): [READ_COMMITTED] Read proceeds (no lock needed) on trans_id=10001 [READ_COMMITTED] Read proceeds (no lock needed) on trans_id=10001 [READ_COMMITTED] Read proceeds (no lock needed) on trans_id=10001 ✓ PASSED: All 3 concurrent reads consistent (245ms). This confirms that the system prioritizes read availability, allowing multiple simultaneous readers to access shared data without mutual interference.

Write + Concurrent Reads (Case #2) This case revealed critical differences between isolation levels in how they handle concurrent access to data being modified. READ_UNCOMMITTED exhibited dirty reads in multiple iterations, where concurrent read operations sometimes returned different values with some seeing the old amount of 2000.00 while others saw the new amount of 2500.00 before the write transaction committed, confirming that READ_UNCOMMITTED allows reading uncommitted data. Log Output (Case #2): [READ_UNCOMMITTED] No lock needed for write on trans_id=20000 Write: Updated to amount=2500.00, balance=7500.00 Read 1: amount=2500.00 (dirty read - saw uncommitted data) Read 2: amount=2000.00 (read old value) ⚠️ DIRTY READS DETECTED: Read values varied: [2000, 2500] ℹ️ This is EXPECTED behavior for READ_UNCOMMITTED ✓ PASSED: Final state consistent across nodes (198ms). In contrast, READ_COMMITTED successfully prevented all dirty reads, as concurrent read operations blocked until the write transaction committed, after which all reads saw only the new committed value of 2500.00, with zero dirty reads across all 3 iterations. Log Output (Case #2): [READ_COMMITTED] Write lock ACQUIRED on trans_id=20001 [READ_COMMITTED] Read WAITING for writer to commit on trans_id=20001... [READ_COMMITTED] Write lock RELEASED on trans_id=20001 Read 1: amount=2500.00 Read 2: amount=2500.00 ✓ All concurrent reads returned consistent values (2500) ✓ PASSED: Final state consistent across nodes (312ms). REPEATABLE_READ and SERIALIZABLE also prevented dirty reads but with higher blocking overhead, where average execution time increased by 40-60% compared to READ_COMMITTED due to long-duration locks held for snapshot consistency.

Concurrent Writes (Case #3) All isolation levels successfully serialized concurrent write operations through the lock manager, which forced updates to execute sequentially. When two nodes attempted to update the same trans_id, one acquired the write lock while the other waited or timed out, and after replication completed, all nodes converged to the same final value. Log Output (Case #3): [READ_COMMITTED] Write lock ACQUIRED on trans_id=30001 by tx-abc123 [READ_COMMITTED] Write waiting for writer tx-abc123 on trans_id=30001... [READ_COMMITTED] Write lock ACQUIRED on trans_id=30001 by tx-def456 Writes: 2 succeeded, 0 locked Final: amount=2300.00 (winner: node1) ✓ PASSED: Consistency maintained despite concurrent writes (387ms). This demonstrates that the locking mechanism effectively prevents lost updates and ensures deterministic outcomes even under high contention.

As noted by Cockroach Labs (2024), while SERIALIZABLE isolation provides the highest consistency guarantees, it introduces significant performance overhead due to increased contention and retry rates in distributed environments. Our experiments confirmed this trade-off: READ_UNCOMMITTED was fastest with an average of 198ms but unsafe due to dirty reads making it unacceptable for financial transactions, READ_COMMITTED provided an optimal balance averaging 312ms by preventing dirty reads while maintaining reasonable throughput, REPEATABLE_READ showed higher latency averaging 445ms due to long-duration read locks, and SERIALIZABLE demonstrated the highest latency averaging 520ms due to extensive blocking on both reads and writes. READ_COMMITTED effectively mitigated the risk of dirty reads, a critical requirement for financial transactions, while maintaining lower latency and higher throughput, making it the optimal choice for our workload.

Across all 36 test runs with READ_COMMITTED and higher isolation levels, the system demonstrated no dirty reads, no lost updates with all writes persisted correctly, no cross-node mismatches after replication completed, and 100% eventual consistency where all nodes converged to identical state post-recovery. The lock manager successfully enforced serialization of conflicting operations, ensuring that the ACID properties were maintained across the distributed system.