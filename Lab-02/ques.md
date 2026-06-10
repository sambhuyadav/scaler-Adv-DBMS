Lab Tasks
1. SQLite3 Exploration
Install SQLite3 and perform the following:
Use any sample database.
Run:

 ls -lh


Observe file sizes.
Use PRAGMA commands to find:
Page size
Page count
Experiment with:
mmap_size
Try changing it and observe behavior.
Time queries:

 time SELECT * FROM users;


Compare execution time:
With mmap
Without mmap
Use commands like:

 ps aux | grep sqlite



2. PostgreSQL (PSQL) Setup
Install PostgreSQL.
Perform similar experiments as SQLite3:
Page size
Page count
Query execution time

3. Comparison Report
Create a .md (Markdown) file.
Include comparison between:
SQLite3 vs PostgreSQL:
Page Size
Page Count
Query Performance
mmap impact
 Do NOT submit screenshots
 Only submit the Markdown file

Submission Format
File: README.md (or any .md file)
Content:
Observations
Commands used
Comparison analysis