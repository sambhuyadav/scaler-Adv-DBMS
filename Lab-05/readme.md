# Lab 5 — Shunting-Yard Evaluator + SQL SELECT Parser

**Name:** Shambhu Yadav
**Roll No:** 10356
**Environment:** macOS arm64

---

## Aim
Take a SQL-style `WHERE` condition like
`marks >= 80 AND (age < 20 OR id = 5)` and evaluate it against rows. I did this
two different ways to compare the approaches.

---

## Part A — Shunting-Yard (`shunting_yard/`)

Dijkstra's **Shunting-Yard** algorithm turns an *infix* expression (the way we
write it) into *postfix / RPN* (Reverse Polish Notation), which is easy to
evaluate with one stack.

**Steps in my code:**
1. **`tokenize`** — split the string into tokens (columns, numbers, operators,
   parentheses). `AND`/`OR` are recognised in any case.
2. **`precedence`** — a table saying comparisons (`>`, `<`, `=`, …) bind tighter
   than `AND`, which binds tighter than `OR`.
3. **`toPostfix`** — the shunting-yard step using an operator stack and the
   precedence table; parentheses handled explicitly.
4. **`evalPostfix`** — for each row, walk the postfix list with one integer
   stack: push operands, and for each operator pop two values and push the
   result. Comparison results (0/1) feed into `AND`/`OR`.

The program prints the postfix form and lists the matching rows. **No parse tree
and no recursion needed** — the postfix order already says what to do first.

## Part B — SQL SELECT Parser (`queryparsing/`)

This version parses a full mini SELECT:
`SELECT name FROM students WHERE marks >= 80 AND (age < 20 OR id = 5)`.

**Steps in my code:**
1. **`lex`** — turn the SQL text into tokens (`SELECT`, `FROM`, `WHERE`, names,
   numbers, comparisons, parentheses).
2. **Recursive-descent `Parser`** — build an **AST** (Abstract Syntax Tree). The
   grammar's nesting encodes the precedence:
   ```
   expr   := term (OR term)*       <- OR binds loosest
   term   := factor (AND factor)*  <- AND binds tighter
   factor := '(' expr ')' | Name Cmp Int   <- comparison binds tightest
   ```
3. **`eval`** — walk the AST once per row to get true/false.
4. **`printAst`** — prints the tree so the precedence is visible.

Because precedence lives in the tree shape, this evaluator needs **no precedence
table at all**.

## Build & run
```bash
cd shunting_yard && make run
cd ../queryparsing && make run
```

## Comparison of the two approaches
| | Shunting-Yard (Part A) | Recursive Descent (Part B) |
|---|---|---|
| Output | Postfix (RPN) | Parse tree (AST) |
| Precedence | In a table | In the grammar |
| Evaluation | One stack, no recursion | Walk the tree recursively |
| Good for | Simple expression evaluation | Real parsers / query engines |

## Conclusion
Both methods correctly filter the same rows. Shunting-yard is a neat, compact way
to evaluate one expression; the recursive-descent parser is closer to how a real
SQL engine turns a query into a tree it can plan and run.
