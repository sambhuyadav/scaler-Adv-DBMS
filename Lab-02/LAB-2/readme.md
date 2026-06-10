# Lab 2 — SQLite3 Internals + PostgreSQL Comparison

**Name:** Shambhu Yadav
**Roll No:** 10356
**Environment:** macOS arm64 — SQLite 3.51.0, PostgreSQL 14.20 (Homebrew)

---

## Aim
Look at how SQLite3 stores data on disk (pages, `mmap`, PRAGMA settings), measure
query timing with and without memory mapping, and compare SQLite3 with PostgreSQL.

## Tools used
Installed with Homebrew on macOS:

```bash
brew install sqlite3
brew install postgresql@15
```

- SQLite sample database: **Chinook** (a small music-store dataset).
- PostgreSQL test table: a `users` table with 50,000 rows.

> All numbers below are the actual output measured on my Mac (Apple Silicon).

---

## Part 1 — SQLite3

**Check the file and its pages**

```bash
ls -lh chinook.db        # 984K
sqlite3 chinook.db
```

```sql
PRAGMA page_size;    -- 4096
PRAGMA page_count;   -- 246
PRAGMA mmap_size;    -- 0   (memory mapping is OFF by default)
PRAGMA journal_mode; -- delete
PRAGMA cache_size;   -- 2000
SELECT COUNT(*) FROM Track;  -- 3503
```

| Metric | Value |
|---|---|
| File size | 984 KB |
| Page size | 4096 bytes (4 KB) |
| Page count | 246 |
| Total | 4096 × 246 ≈ 984 KB |

**In simple words:** the page size is 4 KB, the same as the operating system's
memory page. SQLite keeps them equal so one disk page fits exactly into one OS
page with no wasted space.

**Timing a full scan — without mmap** (using SQLite's built-in `.timer on`,
3503 rows):

```bash
sqlite3 chinook.db ".timer on" "SELECT * FROM Track;"
```

| Run | real time |
|---|---|
| Run 1 (cold) | 7 ms |
| Run 2 (warm) | 5 ms |
| Run 3 (warm) | 4 ms |

The first run is a little slower because the file is read from disk. After that
the OS keeps the file in its **page cache**, so the next runs are faster. This
speed-up comes from the operating system, not SQLite.

**Timing with mmap on**

```bash
sqlite3 chinook.db ".timer on" "PRAGMA mmap_size=30000000; SELECT * FROM Track;"
```

| Run | real time |
|---|---|
| Run 1 | 3 ms |
| Run 2 | 3 ms |
| Run 3 | 4 ms |

With `mmap`, SQLite reads the file like normal memory instead of using `read()`
calls, so it was slightly faster (3–4 ms vs 4–7 ms). But the gap is tiny because
a ~1 MB database already fits in cache. `mmap` mainly helps on big databases.
**This is an honest result, not a failed experiment.**

**A heavier query — JOIN + GROUP BY**

```sql
SELECT a.Title, COUNT(t.TrackId) c
FROM Album a JOIN Track t ON a.AlbumId = t.AlbumId
GROUP BY a.AlbumId ORDER BY c DESC LIMIT 20;
-- real time: < 1 ms
```

Both tables are tiny and already cached, so this finished in under a millisecond.

**Process check**

```bash
ps aux | grep sqlite
```

**0 processes** when no query is running — there is **no server, no background
process**. SQLite is a library that runs inside the program that opens it.
Simple, but only one writer can use the file at a time.

---

## Part 2 — PostgreSQL

**Setup (50,000 rows):**

```sql
INSERT INTO users (name, email, city, score)
SELECT 'User_'||i, 'user'||i||'@example.com',
  (ARRAY['Mumbai','Delhi','Bangalore','Chennai','Hyderabad'])[1+(i%5)],
  random()*100
FROM generate_series(1,50000) AS s(i);
-- Time: 172.620 ms
```

**Storage internals:**

```sql
SHOW block_size;                                       -- 8192
SELECT relpages FROM pg_class WHERE relname='users';   -- 568
SELECT pg_size_pretty(pg_relation_size('users'));      -- 4544 kB
SELECT pg_size_pretty(pg_total_relation_size('users'));-- 5688 kB
```

| Metric | Value |
|---|---|
| Block size | 8192 bytes (8 KB) |
| Block count | 568 |
| Relation size | 4544 kB |
| Total (with index) | 5688 kB |

PostgreSQL uses an **8 KB block**, double SQLite's 4 KB. Bigger blocks mean fewer
disk reads when scanning large tables, which suits a server.

**Query timings (`\timing on`):**

| Query | Time |
|---|---|
| `SELECT * FROM users LIMIT 100` | 1.398 ms |
| `GROUP BY city` (count + avg) | 17.509 ms |
| `WHERE score > 95` | 7.457 ms |
| `ORDER BY score DESC LIMIT 50` | 6.850 ms |

**Index experiment (most interesting part):**

```sql
-- BEFORE index: full table scan
EXPLAIN ANALYZE SELECT * FROM users WHERE score > 95;
--  Seq Scan on users ... Rows Removed by Filter: 47490
--  Execution Time: 5.386 ms

CREATE INDEX idx_users_score ON users(score);    -- Time: 31.816 ms

-- AFTER index, selective filter (few rows match): index IS used
EXPLAIN ANALYZE SELECT * FROM users WHERE score > 95;
--  Bitmap Heap Scan ... Heap Blocks: exact=562
--  ->  Bitmap Index Scan on idx_users_score
--  Execution Time: 1.537 ms        (about 3.5× faster)

-- AFTER index, broad filter (~90% match): index is IGNORED
EXPLAIN ANALYZE SELECT * FROM users WHERE score > 10;
--  Seq Scan on users ... Rows Removed by Filter: 4878   (45122 matched)
--  Execution Time: 4.895 ms
```

When few rows match (`score > 95`), PostgreSQL uses the index and is ~3.5× faster
(5.386 ms → 1.537 ms). When ~90% of rows match (`score > 10`), it **chooses to
skip the index** and just scans the whole table, because that is cheaper than
jumping around the index. The planner decides this automatically by comparing
costs — the most interesting observation of the lab.

**Memory config:**

```sql
SHOW shared_buffers;        -- 128MB
SHOW work_mem;              -- 4MB
SHOW effective_cache_size;  -- 4GB
```

**MVCC — dead rows in practice:**

```sql
UPDATE users SET score = score + 1 WHERE city = 'Mumbai';   -- 10000 rows
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname='users';
-- 50000 | 10000     <- 10,000 old row versions left behind

VACUUM users;
-- after vacuum: 50000 | 0      <- dead rows reclaimed
```

PostgreSQL never overwrites a row in place — an UPDATE writes a **new version**
and marks the old one dead. This is why readers don't block writers. `VACUUM`
later cleans up the dead versions.

**Process check:**

```bash
ps aux | grep postgres
```

PostgreSQL runs a main process plus background helpers — **checkpointer,
background writer, walwriter, autovacuum launcher, logical replication launcher,
stats collector** (about **6 background processes vs SQLite's 0**).

---

## Part 3 — Comparison

| Feature | SQLite3 | PostgreSQL |
|---|---|---|
| Page / block size | 4 KB | 8 KB |
| Dataset measured | 3,503 rows (984 KB) | 50,000 rows (5688 kB) |
| Full scan / filter | 4–7 ms | 5.386 ms (seq) → 1.537 ms (indexed) |
| Architecture | In-process library | Client–server |
| Background processes | 0 | ~6 |
| Concurrent writers | 1 (file lock) | Many (MVCC) |
| Memory mapping | `PRAGMA mmap_size` (manual) | `shared_buffers` (automatic, 128MB) |
| Query planner | Simple (rule-based) | Smart (cost-based) |
| Best for | Embedded / single-user apps | Multi-user server apps |

## Conclusion
SQLite is a small, fast, single-file library — great for one program. PostgreSQL
is a full server with concurrency, MVCC, crash recovery, and a smart query
planner — needed when many users use the database at once. Almost every
difference comes down to one fact: **SQLite is a library, PostgreSQL is a server.**
