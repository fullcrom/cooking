# Concurrency Control Test Cases - MCO2

**Test ID:** Concurrency Control Testing  
**Module:** Distributed Database Concurrency Control and Replication

## Designed by:
- Dy, Danika L.
- Tandingan, Hyun Jei O.
- Felipe, Gerard Christian A.

## Test Data Source:
- `trans` table records with `trans_id` in range 1-1000
- Multiple concurrent transactions across 3 nodes (node0, node1, node2)

## Objectives:
To verify that the distributed database system correctly handles concurrent transactions across multiple nodes while maintaining data consistency through proper isolation level implementation and replication.

---

## Test Case 1: Concurrent Reads on Same Data Item

**Test ID:** CC-01  
**Isolation Levels Tested:** READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE

| Test Case # | Description | Test Steps | Expected Results | Actual Results | Pass/Fail | Performed by/Date |
|-------------|-------------|------------|------------------|----------------|-----------|-------------------|
| **1-1** | READ_UNCOMMITTED: Concurrent reads | 1. Start Transaction A on node0: `SELECT * FROM trans WHERE trans_id = 100`<br>2. Start Transaction B on node1: `SELECT * FROM trans WHERE trans_id = 100`<br>3. Start Transaction C on node2: `SELECT * FROM trans WHERE trans_id = 100`<br>4. All transactions execute simultaneously | - All 3 transactions complete successfully<br>- No locks acquired (READ_UNCOMMITTED behavior)<br>- All return same data<br>- Response time: < 100ms per transaction<br>- Database remains consistent | | | |
| **1-2** | READ_COMMITTED: Concurrent reads | Same steps as 1-1 with isolation level = READ_COMMITTED | - All 3 transactions complete successfully<br>- No read locks needed<br>- All return same committed data<br>- Response time: < 100ms per transaction<br>- Database remains consistent | | | |
| **1-3** | REPEATABLE_READ: Concurrent reads | Same steps as 1-1 with isolation level = REPEATABLE_READ | - All 3 transactions complete successfully<br>- Read locks acquired and held until commit<br>- All return same data<br>- Response time: < 150ms per transaction<br>- Database remains consistent | | | |
| **1-4** | SERIALIZABLE: Concurrent reads | Same steps as 1-1 with isolation level = SERIALIZABLE | - All 3 transactions complete successfully<br>- Read locks acquired and held until commit<br>- Multiple readers can coexist<br>- Response time: < 200ms per transaction<br>- Database remains consistent | | | |

---

## Test Case 2: Concurrent Writes and Reads on Same Data Item

**Test ID:** CC-02  
**Isolation Levels Tested:** READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE

| Test Case # | Description | Test Steps | Expected Results | Actual Results | Pass/Fail | Performed by/Date |
|-------------|-------------|------------|------------------|----------------|-----------|-------------------|
| **2-1** | READ_UNCOMMITTED: Write + Reads | 1. Start Transaction A on node0: `UPDATE trans SET balance = balance + 100 WHERE trans_id = 100`<br>2. Before A commits, start Transaction B on node1: `SELECT balance FROM trans WHERE trans_id = 100`<br>3. Before A commits, start Transaction C on node2: `SELECT balance FROM trans WHERE trans_id = 100`<br>4. Commit Transaction A | - Transaction A completes write<br>- Transactions B & C may read uncommitted data (dirty read allowed)<br>- After A commits, replication occurs<br>- Response time: < 150ms<br>- Final state consistent after replication | | | |
| **2-2** | READ_COMMITTED: Write + Reads | Same steps as 2-1 with isolation level = READ_COMMITTED | - Transaction A acquires write lock<br>- Transactions B & C wait for A to commit before reading<br>- No dirty reads occur<br>- Replication completes successfully<br>- Response time: < 200ms<br>- Database remains consistent | | | |
| **2-3** | REPEATABLE_READ: Write + Reads | Same steps as 2-1 with isolation level = REPEATABLE_READ | - Transaction A acquires write lock (held until commit)<br>- Transactions B & C acquire read locks after A commits<br>- No dirty reads or non-repeatable reads<br>- Replication completes successfully<br>- Response time: < 250ms<br>- Database remains consistent | | | |
| **2-4** | SERIALIZABLE: Write + Reads | Same steps as 2-1 with isolation level = SERIALIZABLE | - Transaction A acquires write lock (held until commit)<br>- Transactions B & C wait for write lock to release<br>- Strict serialization of operations<br>- Replication completes successfully<br>- Response time: < 400ms<br>- Database remains consistent | | | |

---

## Test Case 3: Concurrent Writes on Same Data Item

**Test ID:** CC-03  
**Isolation Levels Tested:** READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE

| Test Case # | Description | Test Steps | Expected Results | Actual Results | Pass/Fail | Performed by/Date |
|-------------|-------------|------------|------------------|----------------|-----------|-------------------|
| **3-1** | READ_UNCOMMITTED: Concurrent writes | 1. Start Transaction A on node0: `UPDATE trans SET balance = balance + 100 WHERE trans_id = 100`<br>2. Start Transaction B on node1: `UPDATE trans SET balance = balance + 200 WHERE trans_id = 100`<br>3. Start Transaction C on node2: `UPDATE trans SET balance = balance + 300 WHERE trans_id = 100`<br>4. All commit | - No locks acquired (READ_UNCOMMITTED)<br>- Lost updates may occur<br>- Final balance may be inconsistent<br>- Demonstrates need for locking<br>- Response time: < 150ms per transaction | | | |
| **3-2** | READ_COMMITTED: Concurrent writes | Same steps as 3-1 with isolation level = READ_COMMITTED | - Transactions serialize on write lock<br>- Each transaction waits for previous to complete<br>- No lost updates<br>- Replication maintains consistency<br>- Final balance = original + 600<br>- Response time: < 300ms total<br>- Database remains consistent | | | |
| **3-3** | REPEATABLE_READ: Concurrent writes | Same steps as 3-1 with isolation level = REPEATABLE_READ | - Transactions serialize on write lock<br>- Write locks held until commit<br>- No lost updates<br>- Replication maintains consistency<br>- Final balance = original + 600<br>- Response time: < 400ms total<br>- Database remains consistent | | | |
| **3-4** | SERIALIZABLE: Concurrent writes | Same steps as 3-1 with isolation level = SERIALIZABLE | - Strict serialization of all writes<br>- Each transaction waits for complete lock release<br>- No lost updates, no phantoms<br>- Replication maintains consistency<br>- Final balance = original + 600<br>- Response time: < 600ms total<br>- Database remains consistent | | | |

---

## Verification Checklist

For each test case, verify the following:

### Data Consistency
- [ ] All nodes have identical data after replication completes
- [ ] No data corruption or anomalies detected
- [ ] Transaction log shows correct sequence of operations
- [ ] Replication queue processed successfully

### Isolation Level Behavior
- [ ] READ_UNCOMMITTED: No locks, dirty reads possible
- [ ] READ_COMMITTED: Write locks only, no dirty reads
- [ ] REPEATABLE_READ: Read + write locks, no non-repeatable reads
- [ ] SERIALIZABLE: Strict locking, complete serialization

### Performance Metrics
- [ ] Response times recorded for each isolation level
- [ ] Lock wait times documented
- [ ] Throughput measured (transactions per second)
- [ ] Resource utilization monitored

### Error Handling
- [ ] Deadlock detection and resolution (if applicable)
- [ ] Timeout handling verified
- [ ] Transaction rollback on failure
- [ ] Error logging functional

---

## Test Execution Instructions

### Prerequisites
1. All 3 database nodes (node0, node1, node2) must be online
2. Backend server running on port 5000
3. Transaction log cleared before each test run
4. Test data initialized in all nodes

### Execution Steps
1. Clear logs: `POST /api/logs/clear`
2. Verify node status: `GET /api/nodes/status`
3. For each test case:
   - Execute transactions as specified
   - Record response times
   - Capture transaction logs
   - Verify data consistency across nodes
   - Check replication queue status
4. Document results in "Actual Results" column
5. Mark Pass/Fail based on expected vs actual

### Data Collection
- Transaction logs: `GET /api/logs/transactions`
- Replication queue: `GET /api/replication/queue`
- Lock status: `GET /api/locks/status`
- Node data: `GET /api/data/{node}?filter=by_trans_id&trans_id={id}`

---

## Expected Performance Summary

| Isolation Level | Case 1 (Reads) | Case 2 (Write+Reads) | Case 3 (Writes) |
|-----------------|----------------|----------------------|-----------------|
| READ_UNCOMMITTED | ~50-80ms | ~100-150ms | ~150-200ms |
| READ_COMMITTED | ~60-100ms | ~150-200ms | ~250-350ms |
| REPEATABLE_READ | ~80-150ms | ~200-300ms | ~350-500ms |
| SERIALIZABLE | ~150-250ms | ~300-500ms | ~500-800ms |

*Note: Times increase with SERIALIZABLE due to strict locking overhead*

---

## Reporting Template

### Test Summary Report

**Date:** ___________  
**Tester:** ___________  
**Environment:** 3-node distributed database system

#### Overall Results
- Total Test Cases: 12 (3 cases × 4 isolation levels)
- Passed: _____ / 12
- Failed: _____ / 12
- Pass Rate: _____%

#### Key Findings
1. Isolation Level Performance:
   - Fastest: ___________
   - Slowest: ___________
   - Recommended for production: ___________

2. Consistency Verification:
   - All nodes consistent: [ ] Yes [ ] No
   - Replication success rate: _____%
   - Data anomalies detected: ___________

3. Issues Encountered:
   - ___________
   - ___________

#### Conclusion
_Document whether the system meets the requirements for proper concurrency control and data consistency across all isolation levels._

---

## Notes
- Test on each isolation level: READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE
- Ensure all nodes remain in consistent state after each test
- Document any anomalies or unexpected behavior
- Performance may vary based on network latency and system load
