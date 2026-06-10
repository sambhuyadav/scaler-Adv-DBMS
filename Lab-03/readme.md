# Lab 3 — ClockSweep Buffer Pool Replacement

**Name:** Shambhu Yadav
**Roll No:** 10356
**Environment:** macOS arm64

---

## Aim
Implement PostgreSQL's **Clock Sweep** (also called *second-chance*)
page-replacement algorithm in C++. This is how a database decides which page to
remove from its buffer pool (in-memory page cache) when the pool is full.

## The idea (in simple words)
- The cache is a fixed array of slots arranged in a **circle**.
- A "hand" (pointer) moves around the circle like a clock.
- Each slot has a **reference bit** (`refBit`).
- On a **hit**, set that slot's `refBit = 1` (a "second chance").
- When we must evict and the hand lands on a slot:
  - if `refBit == 1` → set it to `0` and move on (give it one more chance),
  - if `refBit == 0` → this slot is the **victim**, evict it.

This is cheaper than true LRU because it never reorders a list — it just flips a
bit.

## What I implemented (`main.cpp`)
- A templated class `ClockSweep<T>` (works for `int`, `string`, or a future
  `Page` type).
- `putKey(key)` — insert a key; if the cache is full, run the clock sweep to pick
  a victim. If the key already exists, just refresh its `refBit`.
- `getKey(key)` — return the key on a hit and set its `refBit`; return a default
  value on a miss.
- `findVictimLocked()` — the clock-hand loop described above.
- A **background "aging" thread** that periodically clears all reference bits, so
  old pages don't keep their second chance forever. A `mutex` protects the shared
  data and a `condition_variable` stops the thread cleanly on shutdown.

## Build & run
```bash
cd Lab-3
cmake . && make          # or: c++ -std=c++17 -pthread main.cpp -o db_engine
./db_engine
```

## What the demos show
1. **Demo 1 (`int`, capacity 4):** fill the cache, let the background sweep age
   all bits to 0, then access keys 2 and 4 (their bits go back to 1). New inserts
   then evict the slots whose `refBit` is still 0 — the "second chance" rule
   working. Recently used keys (2, 4) survive; cold keys (1, 3) get evicted.
2. **Demo 2 (`string`, capacity 3):** the same eviction logic on string keys.
3. **Demo 3:** calling `putKey` on an existing key only refreshes its reference
   bit and does **not** grow the cache.

## Conclusion
Clock Sweep gives an LRU-like result (recently used pages stay) but with far less
work per access. This is why real databases like PostgreSQL use it for their
buffer pool instead of strict LRU.
