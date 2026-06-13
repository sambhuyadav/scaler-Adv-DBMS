# Lab 4 — Red-Black Tree and B-Tree

**Name:** Shambhu Yadav
**Roll No:** 10356
**Environment:** macOS arm64

---

## Aim
Implement two self-balancing search trees that databases use for indexes:
a **Red-Black Tree** (used for in-memory indexes) and a **B-Tree** (used for
on-disk indexes).

---

## Part A — Red-Black Tree (`red_black_tree/`)

A Red-Black Tree is a balanced binary search tree with these rules:
- every node is **red** or **black**,
- the root is black,
- a red node cannot have a red child,
- every path from a node down to its leaves has the same number of black nodes.

These rules keep the height around `O(log n)`, so search/insert/delete stay fast.

**What I implemented:**
- A `nil` sentinel node (one shared black leaf) to make the edge cases simpler.
- `insert(key)` — normal BST insert as a red node, then `fixInsert` repairs any
  red-red problem using the three standard cases (recolor / rotate), handled
  symmetrically for left and right.
- `remove(key)` — CLRS-style delete using `transplant` and the in-order
  successor, then `fixDelete` repairs the "double-black" case with its four
  sibling cases.
- `leftRotate` / `rightRotate` — the rotations that rebalance the tree.
- `print()` — a level-order (BFS) print showing each node's key and colour
  (e.g. `10:B`, `15:R`) so the balance can be checked by eye.

**Build & run:**
```bash
cd red_black_tree
make
./<binary>
```

## Part B — B-Tree (`b_tree/`)

A B-Tree is a balanced tree where each node holds **many keys** and has **many
children**. This is ideal for disk storage because one node = one disk page, so
the tree stays shallow and needs very few disk reads.

**What I implemented (minimum degree `t`):**
- Each node holds up to `2t − 1` keys and up to `2t` children.
- `insert(k)` — if the root is full it splits first (the tree grows taller),
  otherwise it descends with `insertNonFull`.
- `splitChild` — when a child is full, its middle key moves up to the parent and
  the child splits into two. This is what keeps a B-Tree balanced.
- `search(k)` — walks down the correct child at each level.
- `traverse()` — in-order traversal that prints all keys in sorted order.

**Build & run (menu-driven: insert / search / traverse):**
```bash
cd b_tree
make
./<binary>
```

## Extra — `hex_dump/`
A hex dump of a real SQLite database file (`my_database.db`) showing how SQLite
physically lays out its B-Tree pages on disk. This links the theory above to a
real database file.

## Conclusion
- **Red-Black Trees** are used for **in-memory** structures (few keys per node,
  balanced by colour rules).
- **B-Trees** are used for **on-disk** indexes (many keys per node to match disk
  pages).

Both keep operations at `O(log n)`, which is why they power database indexes.
