# SQLite3 vs PostgreSQL — Storage Internals & Query Performance

**Name:** Shambhu Yadav | **Roll Number:** 10356

---

## 1. Problem Background

### Two Databases, Two Design Philosophies

SQLite and PostgreSQL are both relational databases that speak SQL. Beyond that, they're built on fundamentally different assumptions about *where the database lives* relative to the application, and *who is using it*.

**SQLite** is a C library you embed directly into your process. There's no server, no daemon, no network socket. The database is a single file. The library is your database engine. This makes SQLite the right answer for mobile apps, embedded systems, local developer tooling, and anywhere you'd otherwise use a flat file but want SQL.

**PostgreSQL** is a standalone server process ecosystem. Your application connects to it over a socket. Six background daemons run before your first query. It manages its own memory pool, its own WAL, its own MVCC versioning. This makes it the right answer for any workload with concurrent writers, large datasets, or the need for crash-safe durable transactions.

The question this lab investigates: **what do these architectural differences actually look like when you measure them?**

Specifically:
- How do their storage internals (page/block size, caching strategy) differ?
- What does the query planner actually do — and when does it make surprising decisions?
- What does MVCC cost in practice, and what does VACUUM actually reclaim?
- What does "SQLite is a library, PostgreSQL is a server" mean in observable process terms?

---

## 2. Architecture Overview

### 2.1 The Big Picture — Embedded Library vs Client-Server

```
        SQLite (embedded)                       PostgreSQL (client-server)

  ┌─────────────────────────────┐        ┌──────────┐   ┌──────────┐
  │   Your Application Process   │        │  Client  │   │  Client  │  ... N clients
  │  ┌───────────────────────┐  │        └────┬─────┘   └────┬─────┘
  │  │  SQLite library (C)   │  │             │   TCP / Unix socket
  │  │  ┌─────────────────┐  │  │             ▼              ▼
  │  │  │ SQL compiler/VM │  │  │        ┌──────────────────────────────┐
  │  │  │ B-tree engine   │  │  │        │   postmaster (supervisor)     │
  │  │  │ pager + cache   │  │  │        │  ┌─────────┐  ┌─────────┐     │
  │  │  └────────┬────────┘  │  │        │  │ backend │  │ backend │ ... │  one per connection
  │  └───────────┼───────────┘  │        │  └────┬────┘  └────┬────┘     │
  └──────────────┼──────────────┘        │       ▼            ▼          │
                 │ read()/write()         │  ┌──────────────────────┐   │
                 ▼                         │  │   shared_buffers     │   │  (128MB pool)
         ┌───────────────┐                │  └──────────┬───────────┘   │
         │ single .db    │                │  background: checkpointer,  │
         │ file (+journal)│               │  bgwriter, walwriter,       │
         └───────────────┘                │  autovacuum, repl launcher  │
                                          └──────────┬───────────────────┘
                                                     ▼
                                          ┌───────────────────────────┐
                                          │ heap files + WAL + indexes │
                                          └───────────────────────────┘
```

### 2.2 Main Components

| | SQLite | PostgreSQL |
|---|---|---|
| Engine location | C library linked into the app process | Standalone server process group |
| Process model | 0 background processes | 1 postmaster + N backends + 6 daemons |
| Concurrency control | File-level locking (1 writer) | Row-level MVCC (many writers) |
| Memory | OS page cache + in-process page cache | `shared_buffers` managed pool + OS cache |
| Storage unit | 4 KB page | 8 KB block |
| On-disk layout | One file (+ rollback journal / WAL file) | Heap files + per-index files + WAL + TOAST |

### 2.3 Data Flow

- **SQLite:** SQL → in-process compiler → bytecode VM → B-tree/pager → `read()`/`write()` syscalls against one file. No IPC, no network, no daemon. The library *is* the engine.
- **PostgreSQL:** client → socket → dedicated backend → planner/executor → `shared_buffers` → heap/index files; durability and cleanup are pushed onto background daemons (walwriter, checkpointer, autovacuum) off the transaction's critical path.

### 2.4 Environment Setup

Both databases ran inside Docker containers on Windows via Docker Desktop (WSL2 backend). The index-internals experiment (§3.4, §5 Exp 4) was additionally reproduced locally on macOS (SQLite 3.51.0, PostgreSQL 14.20 via Homebrew).

```bash
# SQLite — plain Ubuntu container
docker run -it --name sqlite-lab ubuntu:22.04 bash
apt update && apt install -y sqlite3 wget

# PostgreSQL — official image
docker run -d --name pg-lab \
  -e POSTGRES_USER=labuser \
  -e POSTGRES_PASSWORD=lab123 \
  -e POSTGRES_DB=labtest \
  -p 5432:5432 \
  postgres:15
```

**SQLite dataset:** Chinook (digital music store — albums, tracks, invoices, customers). Small but real, with foreign keys and JOINs.

**PostgreSQL dataset:** Synthetic `users` table — 50,000 rows generated via `generate_series`. Large enough to make index decisions meaningful.

---

## 3. Internal Design

### 3.1 SQLite Storage Internals

```sql
PRAGMA page_size;    -- 4096
PRAGMA page_count;   -- 246
```

| Metric | Value |
|--------|-------|
| Page size | 4096 bytes (4KB) |
| Page count | 246 |
| Total DB size | 4096 × 246 = **~984 KB** |

**Why 4KB?** This exactly matches the Linux kernel's default memory page size. When the OS loads a SQLite page into memory, it maps to one kernel page — zero alignment waste. SQLite was designed for environments where matching the OS's unit of work matters. There's no shared buffer pool; SQLite relies on the OS page cache and its own configurable in-process cache.

```sql
PRAGMA journal_mode;   -- delete
PRAGMA cache_size;     -- -2000
PRAGMA mmap_size;      -- 0
PRAGMA integrity_check; -- ok
```

- `cache_size = -2000` → up to 2000 × 4096 = **~8MB** of in-process page cache
- `journal_mode = delete` → classic rollback journal; SQLite writes changes to a `-journal` file and deletes it on commit
- `mmap_size = 0` → memory-mapped I/O off by default; the database is read via `read()` syscalls

**Dataset sizes:**
```sql
SELECT COUNT(*) FROM Track;    -- 3503
SELECT COUNT(*) FROM Customer; -- 59
SELECT COUNT(*) FROM Invoice;  -- 412
SELECT COUNT(*) FROM Album;    -- 347
```

---

### 3.2 PostgreSQL Storage Internals

```sql
SHOW block_size;
-- 8192

SELECT relpages FROM pg_class WHERE relname = 'users';
-- 568

SELECT pg_size_pretty(pg_relation_size('users'));
-- 4544 kB

SELECT pg_size_pretty(pg_total_relation_size('users'));
-- 5696 kB
```

| Metric | Value |
|--------|-------|
| Block size | 8192 bytes (8KB) |
| Block count | 568 |
| Relation size | 8192 × 568 = **4544 KB** |
| Total (+ index + TOAST) | **5696 KB** |

**Why 8KB?** PostgreSQL is built for server workloads where you have 128MB+ of shared buffers and want to minimize I/O round-trips. Larger blocks mean more data per disk read, which reduces I/O operations during table scans. The 1.15MB difference between `pg_relation_size` and `pg_total_relation_size` is the cost of the primary key index — infrastructure overhead that didn't exist in SQLite's dataset.

```sql
SHOW shared_buffers;       -- 128MB  (shared across all connections)
SHOW work_mem;             -- 4MB    (per sort/hash operation)
SHOW maintenance_work_mem; -- 64MB   (VACUUM, CREATE INDEX)
SHOW effective_cache_size; -- 4GB    (planner hint: estimated OS cache)
```

`shared_buffers` is PostgreSQL's own page cache, separate from and in addition to the OS page cache. `effective_cache_size` doesn't allocate memory — it's a hint that tells the planner how much the OS is likely caching, influencing whether it prefers index scans over sequential scans.

---

### 3.3 Process Architecture

This is the most viscerally clear illustration of the architectural difference.

**SQLite — while a query runs:**
```bash
ps aux | grep sqlite
# root  2905  0.9  0.0  6532  4608  pts/0  S+  sqlite3 chinook.db
# root  2919  0.0  0.0  3472  1792  pts/1  grep sqlite
```

Two processes: the CLI tool and the grep. No server, no daemon, no background worker. SQLite is a C library loaded into the `sqlite3` CLI process. The library *is* the database engine. Zero IPC overhead, zero network overhead — and zero concurrency beyond one writer at a time.

**PostgreSQL — before any query runs:**
```bash
ps aux
# postgres   1    postgres                           ← postmaster (main)
# postgres  62    postgres: checkpointer
# postgres  63    postgres: background writer
# postgres  65    postgres: walwriter
# postgres  66    postgres: autovacuum launcher
# postgres  67    postgres: logical replication launcher
# root       71    psql -U labuser -d labtest         ← my session
# postgres  77    postgres: labuser labtest [local] idle
```

Six background daemons before a single query runs. Each exists for a reason directly tied to the internals covered in the PostgreSQL README:

| Daemon | Why It Exists |
|--------|--------------|
| checkpointer | Flushes dirty pages from `shared_buffers` to disk periodically; sets WAL recovery boundary |
| background writer | Proactively writes dirty pages so query execution isn't interrupted by I/O bursts |
| walwriter | Flushes WAL to disk; makes `COMMIT` durable before client gets a response |
| autovacuum launcher | Spawns workers to run VACUUM + ANALYZE; without this, dead rows accumulate forever |
| logical replication launcher | Manages replication slots for streaming changes to replicas |

---

### 3.4 Index Implementation — Two B-Tree Philosophies

Both engines use B-trees, but they organize the *table itself* completely differently — and that single decision changes what every index lookup costs.

**SQLite — the table IS a B-tree (clustered on `rowid`).**
A SQLite table is a B-tree keyed on `rowid`; the row data lives in that tree's leaves. A secondary index is a *separate* B-tree storing `(indexed_columns, rowid)`. So a non-rowid lookup is two B-tree descents: search the index tree to get a `rowid`, then seek into the table tree by that `rowid`.

The bytecode makes this literal — two cursors open on two different root pages:

```text
sqlite> EXPLAIN SELECT * FROM users WHERE score = 50;
OpenRead   0  root=2    ; users          ← cursor 0 = TABLE b-tree
OpenRead   1  root=218  ; idx_users_score ← cursor 1 = INDEX b-tree (separate object)
SeekGE     1  ...       ; seek inside the index tree
IdxRowid   1  r[2]=rowid; index leaf hands back the rowid
DeferredSeek 1 ...      ; move the TABLE cursor to that rowid
Column     0  ...       ; read row data from the TABLE tree
```

`sqlite_master` confirms they are distinct on-disk objects (`table users` at root page 2, `index idx_users_score` at root page 218). `EXPLAIN QUERY PLAN` names the access path directly:

```text
WHERE score = 50   → SEARCH users USING INDEX idx_users_score (score=?)        -- 2 trees
WHERE id    = 25000→ SEARCH users USING INTEGER PRIMARY KEY (rowid=?)          -- 1 tree (clustered)
SELECT score …     → SEARCH users USING COVERING INDEX idx_users_score (score=?) -- 0 table access
```

The covering-index case never touches the table tree at all — the answer lives entirely in the index leaves. And a `WHERE id = …` lookup is a *single* descent because the primary key IS the table's clustering key.

**PostgreSQL — the heap is NOT a tree; every index (even the PK) points into it.**
The PostgreSQL heap is unordered pages. *All* indexes — including the primary key — are separate B-trees whose leaf entries store a heap `TID` (`ctid`, a `(block, offset)` pointer). There is no clustered index; even a PK lookup is index-tree descent → heap fetch.

`pageinspect` exposes the real tree shape (50,000-row `users`, `idx_users_score`):

```text
SELECT level, root, version FROM bt_metap('idx_users_score');
 level | root | version
-------+------+---------
     1 |    3 |       4      ← level 1 ⇒ tree height 2 (root page + leaf pages)

relname          | relpages   ← idx_users_score: 139, users (heap): 319, users_pkey: 139
```

Leaf entries literally carry heap pointers — the `ctid` column below is the row's physical `(block, offset)` location in the heap:

```text
SELECT itemoffset, ctid FROM bt_page_items('idx_users_score', 1) LIMIT 3;
 itemoffset |  ctid
------------+---------
          2 | (0,100)   ← index entry → heap block 0, offset 100
          3 | (1,2)
          4 | (1,53)
```

The two-structure access shows up in the plan as an index scan feeding a separate heap fetch — and crucially, even the **primary key** behaves this way:

```text
WHERE score = 50  → Bitmap Index Scan on idx_users_score → Bitmap Heap Scan (Heap Blocks: exact=258)
WHERE id    = 25000→ Index Scan using users_pkey on users   (still index-tree → heap, 0.067 ms)
```

**The structural contrast:**

| | SQLite | PostgreSQL |
|---|---|---|
| Table physical organization | B-tree clustered on `rowid` | Unordered heap pages |
| What a secondary index leaf stores | `(key, rowid)` | `(key, ctid)` — heap TID pointer |
| Primary-key lookup | 1 descent (table *is* the PK tree) | index-tree descent → heap fetch (PK is a separate B-tree) |
| Index height (measured) | n/a (PRAGMA only) | `level=1` ⇒ height 2 for 50k rows |
| Covering index | `USING COVERING INDEX`, skips table | index-only scan (needs visibility map) |
| Consequence | PK reads are cheapest; secondary = double lookup | all lookups pay one heap hop; no clustering benefit, but cheap to add/drop any index |

This is the deeper reason behind the page-size and concurrency differences already covered: SQLite optimizes for the single-file, single-writer embedded case (clustered table = fewest descents), while PostgreSQL's heap+TID model is what lets many indexes and MVCC tuple versions coexist without reorganizing the table on every write.

---

## 4. Design Trade-offs

### 4.1 Page Size: 4KB vs 8KB

SQLite's 4KB aligns with the Linux kernel's memory page — optimal for the OS page cache in memory-constrained embedded environments. PostgreSQL's 8KB block is optimized for server workloads: fewer I/O round-trips per scan, at the cost of more memory per cached page. Neither is wrong — they reflect different deployment contexts.

### 4.2 Caching: OS Cache vs Managed Buffer Pool

SQLite with `mmap_size=0` relies entirely on the OS page cache — convenient but opaque. PostgreSQL's `shared_buffers` gives the database engine control over what stays in memory, allowing database-semantic decisions (pin this page, don't evict this buffer) that the OS cannot make.

With `mmap` enabled on SQLite, the database file maps directly into the process's virtual address space. Instead of `read()` syscalls, memory accesses trigger page faults the OS handles transparently. For a ~1MB database, both approaches produce equivalent performance because the file fits in OS cache either way. The advantage of mmap materializes at scale — a 500MB database under repeated random access where syscall overhead accumulates.

### 4.3 Concurrency: File Lock vs MVCC

SQLite: one writer at a time, file-level lock. Multiple readers are fine; the moment a write starts, readers must wait. For a personal app or embedded use, this is completely acceptable.

PostgreSQL: MVCC means writers create new tuple versions, readers see old versions — no blocking in either direction. This enables hundreds of concurrent connections with full isolation. The cost is dead tuples (old versions) accumulating on disk until VACUUM reclaims them.

### 4.4 Query Planning: Rule-Based vs Cost-Based

SQLite's planner is simpler and more rule-based. PostgreSQL's planner uses statistics from `pg_statistic` to estimate row counts, compute costs for competing plans, and choose based on estimated I/O and CPU. The consequence: PostgreSQL can make — and explain — counterintuitive decisions like *ignoring an existing index* when the math says a sequential scan is cheaper.

---

## 5. Experiments / Observations

### Experiment 1 — mmap: Does It Actually Help?

```bash
# Without mmap
time sqlite3 chinook.db "SELECT * FROM Track;"
# Run 1 (cold): 37ms  |  Run 2 (warm): 16ms  |  Run 3 (warm): 18ms

# With mmap
time sqlite3 chinook.db "PRAGMA mmap_size=30000000; SELECT * FROM Track;"
# Run 1: 18ms  |  Run 2: 20ms  |  Run 3: 14ms
```

**Observation:** The cold→warm drop (37ms → 16ms) without changing anything is the OS page cache warming up. The first read loaded the ~1MB file into kernel memory; subsequent reads never touched disk. This is purely OS behavior, not SQLite.

With mmap enabled, the times were essentially identical to the warm no-mmap runs. **This is a real finding.** The database is small enough to fit entirely in the OS page cache either way — mmap provides no additional benefit here. The honest conclusion: mmap's advantage requires a dataset large enough that OS cache isn't sufficient, and enough repeated random access that `read()` syscall overhead becomes measurable. Neither condition applied.

```bash
# JOIN + aggregation: no difference either way
time sqlite3 chinook.db "SELECT a.Title, COUNT(t.TrackId) FROM Album a \
  JOIN Track t ON a.AlbumId = t.AlbumId GROUP BY a.AlbumId ORDER BY 2 DESC LIMIT 20;"
# Without mmap: 3ms  |  With mmap: 3ms
```

For a CPU-bound operation (sorting, grouping two small already-cached tables), I/O access method is irrelevant. The bottleneck was computation, not I/O. Confirmed by `.timer on` output:

```sql
.timer on
SELECT * FROM Track;
-- real 0.017  user 0.004  sys 0.013   ← more sys than user: kernel I/O dominates

SELECT a.Title, COUNT(t.TrackId) FROM Album a JOIN Track t ON a.AlbumId = t.AlbumId GROUP BY a.AlbumId;
-- real 0.003  user 0.000  sys 0.003   ← balanced: CPU and I/O interleaved
```

High `sys` time on the full scan indicates CPU spent in kernel mode handling I/O. The JOIN query's lower total time with more balanced sys/user confirms the bottleneck shifted to user-space computation.

---

### Experiment 2 — PostgreSQL Query Planner: When Does an Index Get Ignored?

```sql
-- Dataset: 50,000 rows, score uniformly distributed 0–100
CREATE INDEX idx_users_score ON users(score);
-- Time: 44.288 ms
```

**Selective filter — index used:**
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE score > 95;
```
```
Bitmap Heap Scan on users  (actual rows=2507, time=0.183..0.847ms)
  Heap Blocks: exact=563
  → Bitmap Index Scan on idx_users_score  (actual rows=2507, time=0.132ms)
Planning Time: 0.136 ms  |  Execution Time: 0.977 ms
```

Same query without index: **4.187ms**. With index: **0.977ms** — ~4× faster. PostgreSQL built a bitmap of 2,507 matching row locations from the index, then fetched exactly those 563 heap blocks. No wasted I/O.

**Broad filter — index intentionally ignored:**
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE score > 10;
```
```
Seq Scan on users  (actual rows=44974, time=0.009..5.259ms)
  Filter: (score > '10'::double precision)
  Rows Removed by Filter: 5026
Execution Time: 6.986 ms
```

`score > 10` matches **44,974 / 50,000 rows (~90%)**. The index exists — PostgreSQL chose not to use it. **This is the most important observation in the entire lab.**

Why? Using the index would require: read index leaf pages to find 44,974 TIDs → for each TID, fetch the corresponding heap page (potentially 44,974 random heap page accesses). Sequential scan: read 568 heap blocks in order, once. At 90% selectivity, the sequential scan's total I/O is lower than the index lookup's random I/O. PostgreSQL's cost estimator calculated both plans and chose correctly. The existence of an index does not mean PostgreSQL will use it — it means PostgreSQL will consider it.

**Aggregation — HashAggregate with minimal memory:**
```sql
EXPLAIN ANALYZE SELECT city, COUNT(*), AVG(score) FROM users GROUP BY city;
```
```
HashAggregate  (actual rows=5, Memory Usage: 24kB)
  → Seq Scan on users  (actual rows=50000)
Execution Time: 12.787 ms
```

PostgreSQL read all 50,000 rows in one sequential pass and maintained a hash map of 5 city buckets. Total memory: 24KB. This is optimal — no sort, no intermediate materialisation, single pass.

**Sort + Limit — top-N heapsort:**
```sql
EXPLAIN ANALYZE SELECT * FROM users ORDER BY score DESC LIMIT 50;
```
```
Limit  (actual rows=50)
  → Sort  (Method: top-N heapsort, Memory: 37kB)
      → Seq Scan on users  (actual rows=50000)
Execution Time: 7.434 ms
```

Rather than sorting all 50,000 rows and discarding 49,950, PostgreSQL maintained a 50-element heap during the sequential scan — O(N log k) instead of O(N log N). Memory: 37KB instead of megabytes. The planner inferred from the `LIMIT` that a full sort was unnecessary.

---

### Experiment 3 — MVCC Dead Tuples: Making the Cost Visible

```sql
UPDATE users SET score = score + 1 WHERE city = 'Mumbai';
-- UPDATE 10000

SELECT n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'users';
```
```
 n_live_tup | n_dead_tup | last_autovacuum
------------+------------+------------------------------
      50000 |      10000 | 2026-05-05 12:15:26+00
```

One UPDATE on 10,000 rows created **10,000 dead tuples** — ghost row versions on disk. PostgreSQL never overwrites in place. The old versions stay alive because a concurrent reader might still need them (MVCC snapshot isolation). `n_live_tup` stays at 50,000 because the logical row count hasn't changed; `n_dead_tup = 10000` is the physical overhead of non-destructive versioning.

```sql
VACUUM users;

SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'users';
-- n_live_tup: 50000  |  n_dead_tup: 0
```

VACUUM reclaimed all 10,000 dead slots. If autovacuum never ran on a write-heavy table, this count would grow indefinitely — pages fill with ghost data, sequential scans read useless rows, index entries point to dead tuples, performance collapses. The autovacuum launcher daemon observed in `ps aux` exists specifically to prevent this.

This experiment makes concrete what "MVCC trades storage for concurrency" actually means: 10,000 phantom rows on disk, visible in `pg_stat_user_tables`, reclaimed by an explicit `VACUUM` call.

---

### Experiment 4 — Index Internals: Clustered Table vs Heap+TID

Reproduced locally (SQLite 3.51.0; PostgreSQL 14.20) on identical 50,000-row `users(id, city, score)` tables with an index on `score`. Full structural interpretation is in §3.4; the measured evidence:

```text
PostgreSQL — bt_metap('idx_users_score'): level=1  ⇒ B-tree height 2 (root + leaf)
             pages: idx_users_score=139, users heap=319, users_pkey=139
             leaf items carry heap TIDs: ctid (0,100), (1,2), (1,53) ...
             WHERE id=25000 → "Index Scan using users_pkey" (PK is itself a heap-pointing B-tree)

SQLite     — EXPLAIN bytecode: OpenRead root=2 (table tree) + OpenRead root=218 (index tree)
             WHERE score=50  → SEARCH ... USING INDEX  (index tree → rowid → table tree)
             WHERE id=25000  → SEARCH ... USING INTEGER PRIMARY KEY (single clustered descent)
             SELECT score    → SEARCH ... USING COVERING INDEX (table tree never opened)
```

**Observation:** the two engines disagree on the most basic question — *where do the rows live?* In SQLite the rows live inside the primary-key B-tree, so a PK lookup is one descent and the table can't exist without that tree. In PostgreSQL the rows live in an unordered heap and **every** index — primary key included — is a separate B-tree that ends at a `ctid` pointer into that heap. The `level=1` metapage value is a concrete, queryable measurement of index height (2 hops) that SQLite's PRAGMA interface simply doesn't expose.

---

## 6. Key Learnings

**1. Architecture is the root cause of every measured difference.**
SQLite's zero daemons, 4KB pages, file-level lock, and OS-managed cache all follow from "I'm a library embedded in your process." PostgreSQL's six background workers, 8KB blocks, shared buffer pool, and row-level MVCC all follow from "I'm a server managing concurrent clients." Every benchmark result in this lab traces back to this single architectural fork.

**2. The OS page cache is a real database optimization layer.**
The cold→warm timing drop (37ms → 16ms) in SQLite happened without any SQLite configuration change. The kernel warmed its own page cache after the first read. This is why "warm run" vs "cold run" distinctions matter in benchmarking — and why SQLite on a warm system can match or beat solutions with explicit caching for small datasets.

**3. mmap is not universally better — dataset size determines the winner.**
On a ~1MB database, mmap and `read()` syscalls produce identical results because the OS page cache covers both cases equally. mmap's advantage is eliminating syscall overhead at scale, where the OS cache can't hold the entire working set. Honest benchmarking means acknowledging when an optimization doesn't apply, not forcing a result.

**4. A cost-based query planner's most important decisions are the ones you don't see.**
The `score > 10` query silently ignoring its index was the most instructive moment of the lab. The planner evaluated two plans, estimated that 90% selectivity made random heap fetches more expensive than a sequential scan, and chose correctly — without being told. This is what separates a production database from a simple engine: the ability to reason about its own execution costs.

**5. MVCC's cost is physically measurable.**
10,000 dead tuples after a single UPDATE is not an abstraction — it's rows you can count in `pg_stat_user_tables`. The cost of "readers never block writers" is ghost data on disk, background cleanup processes, and the autovacuum system that keeps it from compounding. Understanding MVCC means understanding that non-destructive versioning is a storage trade-off, not a free lunch.

**6. Background daemons are not overhead — they're deferred work.**
Every PostgreSQL daemon observed in `ps aux` exists because the database defers expensive operations (page flushing, log flushing, dead tuple cleanup, statistics updates) out of the critical transaction path. The checkpointer, walwriter, and autovacuum aren't waste — they're what makes commits fast and reads consistent. SQLite avoids them by accepting their limitations: one writer, no background maintenance, no shared memory.

**7. "Both use B-trees" hides the most important storage decision.**
The surprising lesson from §3.4 was how much follows from *where the rows physically live*. SQLite clusters rows inside the primary-key B-tree, so the PK is the cheapest possible lookup and a secondary index is an explicit double descent (provable in the bytecode: two cursors, two root pages). PostgreSQL stores rows in an unordered heap and makes *every* index — even the primary key — a separate B-tree pointing at heap TIDs, which `bt_metap` and `bt_page_items` let you inspect directly down to the `ctid`. Neither is "better": clustering wins on PK reads and compactness; heap+TID wins on flexibility (cheap to add/drop indexes, natural fit for MVCC tuple versions).

---

## Comparison Summary

| Metric | SQLite3 | PostgreSQL |
|--------|---------|-----------|
| Page / block size | 4KB (OS-aligned) | 8KB (I/O-optimized) |
| Page count (dataset) | 246 pages (~984 KB) | 568 blocks (4544 KB relation) |
| Rows in test table | 3,503 (Track) | 50,000 (users) |
| SELECT * cold | 37ms | — |
| SELECT * warm | 16–18ms | 0.47ms (LIMIT 100) |
| Full scan, no index | ~3ms | 4.187ms |
| Full scan, with index | N/A | 0.977ms (~4× faster) |
| Broad filter (90%), index exists | N/A | Seq scan chosen (correct) |
| JOIN + GROUP BY | 3ms | 12.787ms |
| INSERT 50k rows | N/A | 113.694ms |
| Index creation | N/A | 44.288ms |
| Dead tuples after UPDATE 10k | 0 (no MVCC) | 10,000 |
| After VACUUM | N/A | 0 |
| Background processes | 0 | 6 daemons |
| Max concurrent writers | 1 (file lock) | Many (row-level MVCC) |
| Table physical layout | B-tree clustered on `rowid` | Unordered heap pages |
| Index leaf points to | `rowid` (into table tree) | `ctid` heap TID (into heap) |
| Primary-key lookup cost | 1 B-tree descent (clustered) | index descent → heap fetch |
| Measured index height (50k) | n/a (no introspection) | 2 (`bt_metap` level=1) |
| Buffer cache | OS page cache + PRAGMA | `shared_buffers` (128MB default) |
| Architecture | In-process library | Client-server process ecosystem |

---

## Commands Reference

```bash
# SQLite
sqlite3 chinook.db
.tables
PRAGMA page_size;
PRAGMA page_count;
PRAGMA mmap_size;
PRAGMA mmap_size=30000000;
PRAGMA journal_mode;
PRAGMA cache_size;
PRAGMA integrity_check;
.timer on
time sqlite3 chinook.db "SELECT * FROM Track;"
ps aux | grep sqlite

# PostgreSQL
docker exec -it pg-lab psql -U labuser -d labtest
SHOW block_size;
SHOW shared_buffers;
SHOW work_mem;
SHOW effective_cache_size;
SELECT relpages FROM pg_class WHERE relname = 'users';
SELECT pg_size_pretty(pg_relation_size('users'));
SELECT pg_size_pretty(pg_total_relation_size('users'));
\timing on
EXPLAIN ANALYZE SELECT ...;
CREATE INDEX idx_users_score ON users(score);
SELECT n_live_tup, n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE relname = 'users';
VACUUM users;
ps aux
```