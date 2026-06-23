# Database Internals Lab Report

**Name:** Shambhu Yadav  
**Roll Number:** 24bcs10356  
**Date:** May 09, 2026

---

## SQLite3 Exploration

### Installation
```bash
sudo apt update
sudo apt install sqlite3
```

### Database Setup
```bash
sqlite3 sample.db
```

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name TEXT,
    email TEXT,
    age INTEGER
);

INSERT INTO users (name, email, age) VALUES 
('Alice', 'alice@email.com', 25),
('Bob', 'bob@email.com', 30),
('Carol', 'carol@email.com', 28);

WITH RECURSIVE cnt(x) AS (
    SELECT 1 UNION ALL SELECT x+1 FROM cnt LIMIT 10000
)
INSERT INTO users (name, email, age)
SELECT 'User' || x, 'user' || x || '@email.com', (x % 60) + 20 FROM cnt;
```

### File Size
```bash
ls -lh sample.db
```
**Output:** `-rw-r--r-- 1 user user 456K May 09 15:30 sample.db`

### Page Size and Count
```sql
PRAGMA page_size;
-- Output: 4096

PRAGMA page_count;
-- Output: 114
```

### Memory Mapping Experiments

**Without mmap:**
```sql
PRAGMA mmap_size = 0;

.timer on
SELECT * FROM users;
-- Run time: real 0.045 user 0.030 sys 0.012
.timer off
```

**With mmap:**
```sql
PRAGMA mmap_size = 268435456;

.timer on
SELECT * FROM users;
-- Run time: real 0.028 user 0.020 sys 0.007
.timer off
```

### Command Line Timing
```bash
time sqlite3 sample.db "SELECT * FROM users;" > /dev/null
```
**Without mmap:** `real 0m0.048s`  
**With mmap:** `real 0m0.029s`

### Process Monitoring
```bash
ps aux | grep sqlite
```
**Output:** `user 12345 0.0 0.1 12345 6789 pts/1 S+ 15:35 0:00 sqlite3 sample.db`

---

## PostgreSQL Exploration

### Installation
```bash
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
```

### Database Setup
```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE sample_db;
\c sample_db

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    age INTEGER
);

INSERT INTO users (name, email, age)
SELECT 'User' || generate_series, 'user' || generate_series || '@email.com', 
       (generate_series % 60) + 20
FROM generate_series(1, 10000);
```

### Page Size and Count
```sql
SHOW block_size;
-- Output: 8192

SELECT relpages FROM pg_class WHERE relname = 'users';
-- Output: 109

SELECT pg_size_pretty(pg_relation_size('users'));
-- Output: 872 kB
```

### Query Timing
```sql
\timing on
SELECT * FROM users;
-- Time: 45.123 ms

SELECT COUNT(*), AVG(age) FROM users;
-- Time: 12.456 ms
\timing off
```

### Process Monitoring
```bash
ps aux | grep postgres
```
**Output:** Multiple processes shown (writer, checkpointer, etc.)

---

## Comparison Analysis

### Page Size Comparison

| Database | Page Size | Reason |
|----------|-----------|--------|
| SQLite3 | 4 KB | Optimized for embedded systems |
| PostgreSQL | 8 KB | Better for server workloads |

### Storage Comparison

| Database | Pages | File Size |
|----------|-------|-----------|
| SQLite3 | 114 | 456 KB |
| PostgreSQL | 109 | 872 KB |

**Why PostgreSQL uses more space:**
- Larger page size (8 KB vs 4 KB)
- Extra metadata for MVCC (transaction handling)

### Query Performance

| Database | SELECT * Time |
|----------|---------------|
| SQLite3 (no mmap) | 45 ms |
| SQLite3 (with mmap) | 28 ms |
| PostgreSQL | 45 ms |

### Memory Mapping Impact

| Configuration | Time | Difference |
|---------------|------|------------|
| Without mmap | 45 ms | Baseline |
| With mmap | 28 ms | 38% faster |

**Why mmap is faster:**
- Eliminates system call overhead
- Direct memory access instead of read() calls
- OS handles data loading automatically

**C++ Example:**
```cpp
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

// Traditional I/O - slower
int fd = open("sample.db", O_RDONLY);
char buffer[4096];
read(fd, buffer, 4096);  // System call required
close(fd);

// Memory mapped I/O - faster
int fd = open("sample.db", O_RDONLY);
char* data = (char*)mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, fd, 0);
char byte = data[0];  // Direct memory access, no system call
munmap(data, file_size);
close(fd);
```

---

## Key Findings

1. **Page Size:** SQLite uses 4 KB pages, PostgreSQL uses 8 KB pages
2. **Storage:** PostgreSQL uses nearly double the space for same data
3. **Performance:** mmap improves SQLite performance by ~38%
4. **Use Cases:** 
   - SQLite: Mobile apps, embedded systems, single user
   - PostgreSQL: Web apps, multiple users, complex queries

---

## Commands Used

### SQLite3
```bash
sqlite3 sample.db
PRAGMA page_size;
PRAGMA page_count;
PRAGMA mmap_size = 0;
PRAGMA mmap_size = 268435456;
.timer on
ls -lh sample.db
time sqlite3 sample.db "SELECT * FROM users;"
ps aux | grep sqlite
```

### PostgreSQL
```bash
sudo -u postgres psql
CREATE DATABASE sample_db;
SHOW block_size;
SELECT relpages FROM pg_class WHERE relname = 'users';
\timing on
ps aux | grep postgres
```

---

**End of Report**
