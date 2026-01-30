🐘 Staff-Level SQL & Database Interview Guide
For: Java Backend Engineers (11+ Years Experience)
Focus: Internals, Optimization, Concurrency, and System Design.
1. 🚀 Query Performance & Optimization
At this level, you aren't just writing queries; you are fixing the ones that crash production.
Q1: How do you identify and optimize a slow-running query in a production environment?
Answer:
This requires a systematic approach, not just guessing indexes.
 * Identification: Use the database's "Slow Query Log" or APM tools (Datadog/NewRelic) to pinpoint queries exceeding a specific threshold (e.g., >200ms).
 * Analysis (EXPLAIN): Run EXPLAIN ANALYZE (Postgres) or EXPLAIN (MySQL) to view the execution plan.
   * Red Flags: Look for SEQ SCAN (Full Table Scan) on large tables, high "Cost" numbers, or temporary file sorts (sorting on disk instead of memory).
 * Optimization Strategy:
   * Indexing: Add missing indexes on columns used in WHERE, JOIN, or ORDER BY.
   * Covering Index: Create an index that includes all selected columns to allow an "Index Only Scan" (avoiding the heap lookup entirely).
   * Rewrite: Check for non-sargable arguments (e.g., WHERE YEAR(date_col) = 2023 prevents indexing; change to date_col BETWEEN '2023-01-01' AND '2023-12-31').
Q2: Explain the "Leftmost Prefix" rule in Composite Indexes.
Answer:
If you create a composite index on (A, B, C), the database builds a B-Tree sorted first by A, then B, then C.
 * Valid Usage: The index will be used for queries filtering on:
   * A
   * A and B
   * A, B, and C
 * Invalid Usage: The index will NOT be used (or used inefficiently) for queries filtering on:
   * B only (The tree isn't sorted by B globally).
   * C only.
   * B and C.
 * Key Takeaway: Order matters significantly when defining composite indexes in your schema migrations.
Q3: What is the difference between a Clustered and Non-Clustered Index?
Answer:
 * Clustered Index:
   * Determines the physical order of data on the disk.
   * There can be only one per table (usually the Primary Key).
   * Retrieving data via the Clustered Index is the fastest method because the index node contains the actual data row.
 * Non-Clustered (Secondary) Index:
   * Stored separately from the actual data table.
   * Contains the indexed column value + a pointer (usually the Primary Key) to the actual data row.
   * Requires a "Key Lookup" (extra hop) to fetch non-indexed columns, making it slightly slower than a clustered index for full row retrieval.
2. 🔒 Transactions & Concurrency (ACID)
Crucial for ensuring data integrity in distributed Java applications.
Q4: Explain the phenomena of "Dirty Read", "Non-Repeatable Read", and "Phantom Read".
Answer:
 * Dirty Read: Transaction A reads data that Transaction B has written but not yet committed. If B rolls back, A has "dirty" data. (Prevention: READ COMMITTED).
 * Non-Repeatable Read: Transaction A reads a row. Transaction B updates that row and commits. Transaction A reads the row again and gets a different value. (Prevention: REPEATABLE READ).
 * Phantom Read: Transaction A reads a set of rows matching a condition (e.g., "all users > age 25"). Transaction B inserts a new row that matches that condition. Transaction A runs the query again and sees a "phantom" row that wasn't there before. (Prevention: SERIALIZABLE or Snapshot Isolation).
Q5: How does MVCC (Multi-Version Concurrency Control) work?
Answer:
MVCC allows databases (like Postgres, MySQL/InnoDB) to handle concurrent reads and writes without locking the entire table.
 * Concept: When you update a row, the DB doesn't overwrite the original data immediately. Instead, it creates a new version of the row.
 * Mechanism: Readers see the version of the row that was "committed" before their transaction started. Writers work on a new version.
 * Benefit: "Readers don't block Writers, and Writers don't block Readers." This significantly increases concurrency compared to strict locking.
 * Cleanup: Requires a background process (like Postgres VACUUM) to remove old, invisible row versions ("dead tuples") to prevent bloat.
Q6: How do you handle Deadlocks in a Java + SQL application?
Answer:
 * Detection: The database engine detects a cycle in the dependency graph and kills one transaction (usually the one that did the least work) to free the lock.
 * Prevention Strategies:
   * Ordering: Ensure all transactions access tables/rows in the exact same order (e.g., always lock User A before User B).
   * Keep Transactions Short: Do not perform network calls (like HTTP requests) inside an open database transaction.
   * Retry Logic: In Java, catch the DeadlockLoserDataAccessException (Spring) and implement a retry mechanism with exponential backoff.
3. ☕ The Java Connection (ORM & Pooling)
Questions specifically about how your code interacts with the DB.
Q7: What is the "N+1 Select Problem" in Hibernate/JPA and how do you fix it?
Answer:
 * The Problem: You fetch a list of Parent entities (1 query). Then, for each parent, you access a lazy-loaded Child collection, triggering a new query for every parent (N queries). Total = N+1 queries.
 * Example: Fetching 100 Users, then accessing user.getOrders() for each loop iteration results in 101 queries.
 * The Fix:
   * Join Fetch: Use JPQL JOIN FETCH (e.g., SELECT u FROM User u JOIN FETCH u.orders). This forces a single SQL JOIN.
   * Entity Graph: Define @NamedEntityGraph to specify paths to load eagerly.
   * Batch Fetching: Set @BatchSize or hibernate.default_batch_fetch_size. This changes the N queries into N/BatchSize queries (e.g., fetching 50 IDs in one IN clause).
Q8: How do you size a Connection Pool (like HikariCP)? Is "bigger" always better?
Answer:
 * Misconception: Bigger is not better. A pool that is too large causes high context switching overhead on the CPU and disk thrashing.
 * Formula: A common starting point is: connections = ((core_count * 2) + effective_spindle_count).
 * Reality: For most workloads, a pool size of 10-20 is sufficient to handle thousands of concurrent frontend users. The database CPU can only process one query per core at a time; queuing connections in the pool is faster than context switching the OS threads.
Q9: Optimistic vs. Pessimistic Locking in JPA?
Answer:
 * Optimistic Locking (@Version):
   * Mechanism: Adds a version column. Upon update, checks WHERE id=X AND version=1. If 0 rows update (version changed meanwhile), throws OptimisticLockException.
   * Use Case: High read/low write conflict scenarios. Better performance (no DB locks).
 * Pessimistic Locking (SELECT ... FOR UPDATE):
   * Mechanism: Explicitly locks the row in the database when reading it. Other transactions wait until the lock is released.
   * Use Case: Critical financial transactions where conflicts are frequent and must be strictly serialized.
4. 📐 Schema Design & Architecture
Moving beyond code to system structure.
Q10: Normalization vs. Denormalization: When do you break the rules?
Answer:
 * Normalization (3NF): Removes redundancy, ensures data consistency. Great for write-heavy OLTP systems to prevent anomalies.
 * Denormalization: Intentionally introduces redundancy (duplicating data).
   * When to use: When JOIN performance becomes the bottleneck in a read-heavy system.
   * Example: Storing user_name inside the comments table. Normally, you'd JOIN users. But if you list 1 million comments, the JOIN is expensive. Storing the name avoids the JOIN but requires complex logic to update all comments if a user changes their name.
Q11: How do you store Hierarchical Data (Trees) in SQL?
Answer:
Traditional Adjacency List (Parent_ID) makes querying deep levels difficult (requires recursion).
 * Path Enumeration: Store a string column like 1/4/15/. Easy to find subtrees via LIKE '1/4/%'.
 * Nested Sets: Store left and right integer boundaries. Very fast reads for subtrees, but expensive updates (re-balancing the tree).
 * Closure Table: A separate table storing all ancestor-descendant relationships. Best for graph-like flexibility.
 * Modern Solution: Use Recursive Common Table Expressions (Recursive CTEs) WITH RECURSIVE which are supported by most modern DBs.
Q12: When should you use JSON columns (JSONB) in a Relational Database?
Answer:
 * Use Case: For "attributes" that vary wildly between rows (e.g., Product Specifications: a T-Shirt has size/color; a Laptop has RAM/CPU). Creating columns for every possible attribute creates a sparse, messy table.
 * Trade-off:
   * Pros: Flexibility of NoSQL with the ACID guarantees of SQL.
   * Cons: You cannot easily create foreign keys to values inside the JSON blob. Updates to a single field in the JSON usually require rewriting the whole blob (write amplification).
5. ⚡ Advanced SQL Syntax
Show you know modern SQL capabilities.
Q13: Explain Window Functions and give a use case.
Answer:
Window functions perform calculations across a set of table rows that are related to the current row, without collapsing them into a single output row (unlike GROUP BY).
 * Syntax: FUNCTION() OVER (PARTITION BY col ORDER BY col)
 * Use Case: "Find the top 3 highest salaries per department."
 * Example:
   SELECT *,
       RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) as rank
FROM employees
WHERE rank <= 3; -- Note: This requires a subquery/CTE wrapper

Q14: What is a Common Table Expression (CTE) and how does it differ from a Temporary Table?
Answer:
 * CTE (WITH clause): A temporary result set defined within the execution scope of a single SELECT, INSERT, UPDATE, or DELETE statement. It is not stored on disk; it is syntactic sugar (though Postgres can materialize them for performance). It improves readability over nested subqueries.
 * Temp Table: Stored in tempdb, exists for the duration of the session/transaction, can have indexes created on it, and can be accessed by multiple queries within that session. Better for multi-step processing of large datasets.
