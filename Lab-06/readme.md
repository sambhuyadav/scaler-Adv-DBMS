# Lab 6 — Transaction Manager (MVCC + Strict 2PL + Deadlock Detection)

**Name:** Shambhu Yadav
**Roll No:** 10356
**Environment:** macOS arm64

---

## Aim
Build a small in-memory transaction manager in C++ that shows three core database
ideas working together:
1. **MVCC** (Multi-Version Concurrency Control) — keep old versions of a row.
2. **Strict 2PL** (Strict Two-Phase Locking) — hold locks until commit.
3. **Deadlock detection** — find and break circular waits.

## What I implemented (`txn_manager.cpp`)

**1. MVCC version chains**
Each row points to a linked list of `Version` objects (`txn_id`, `data`, `next`).
A write **adds a new version to the front** instead of overwriting the old one:

```cpp
Version* new_version = new Version{txn_id, data, rows[row_id]};
rows[row_id] = new_version;
```

Old versions still exist — the same idea PostgreSQL uses so that readers don't
block writers.

**2. Strict 2PL via a `LockManager`**
- `acquire_lock` takes **SHARED** (read) or **EXCLUSIVE** (write) locks.
- Shared locks can be shared; exclusive locks conflict with everything. A shared
  lock can be **upgraded** to exclusive.
- Locks are released **only at commit/abort** (`release_locks`) — this is the
  "strict" part and prevents dirty reads.

**3. Deadlock detection**
- A **waits-for graph** records "transaction X is waiting for transaction Y".
- Before a transaction blocks, `has_deadlock()` runs a **DFS cycle check** on the
  graph.
- If a cycle is found, the transaction **aborts itself** to break the deadlock
  instead of waiting forever. `abort` then undoes its versions and releases its
  locks.

A `condition_variable` lets waiting transactions sleep instead of busy-looping,
and wakes them when locks are released.

## Build & run
```bash
cd Lab-6
g++ -std=c++17 -pthread txn_manager.cpp -o txn_manager
./txn_manager
```

## The demo (deadlock on purpose)
Two threads are started so they deadlock:
- **Txn 1:** lock row 100, then try row 200.
- **Txn 2:** lock row 200, then try row 100.

This is the classic deadlock. The lock manager detects the cycle in the waits-for
graph and aborts one transaction, so the other can finish. The program prints
which transaction committed/aborted and the final row values.

## Conclusion
This lab ties together the three mechanisms a real database uses for safe
concurrent transactions: **MVCC** keeps multiple row versions so reads and writes
don't block each other, **Strict 2PL** keeps things correct by holding locks to
commit, and **deadlock detection** stops two transactions from freezing each
other forever.
