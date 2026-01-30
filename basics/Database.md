# 🐘 Staff-Level Backend Engineer: SQL & Database Deep Dive

**Target Audience:** Java Backend Engineers (11+ Years Experience)
**Focus:** Internals, Optimization, Concurrency, Distributed Data, and System Design.

---

## 📋 Table of Contents

1. [Query Execution & Optimization](#1-query-execution--optimization)
    - [Anatomy of a Slow Query](#q1-anatomy-of-a-slow-query-how-do-you-debug-it)
    - [Index Internals (Clustered vs. Secondary)](#q2-clustered-vs-non-clustered-indexes-internals)
    - [Composite Indexes & The Leftmost Prefix](#q3-composite-indexes--the-leftmost-prefix-rule)
    - [Sargable Queries](#q4-what-makes-a-query-non-sargable)
2. [Concurrency, Locking & ACID](#2-concurrency-locking--acid)
    - [Isolation Levels & Anomalies](#q5-explain-isolation-levels-and-read-phenomena)
    - [MVCC (Multi-Version Concurrency Control)](#q6-how-does-mvcc-work-internally)
    - [Deadlocks & Prevention](#q7-handling-deadlocks-in-high-concurrency-systems)
    - [Pessimistic vs. Optimistic Locking](#q8-optimistic-vs-pessimistic-locking-strategies)
3. [Schema Design & Distributed Systems](#3-schema-design--distributed-systems)
    - [Normalization vs. Performance Trade-offs](#q9-normalization-vs-denormalization-at-scale)
    - [Sharding & Partitioning](#q10-horizontal-sharding-vs-partitioning)
    - [Handling JSON/Unstructured Data](#q11-jsonb-vs-relational-columns)
    - [Generators & ID Strategies](#q12-primary-key-strategies-uuid-vs-sequence-vs-snowflake)
4. [Java & Database Interaction (ORM)](#4-java--database-interaction-orm)
    - [The N+1 Select Problem](#q13-the-n1-select-problem-in-hibernate)
    - [Connection Pooling (HikariCP)](#q14-connection-pool-sizing-and-configuration)
    - [Transaction Management](#q15-spring-transactional-pitfalls)
5. [Advanced SQL Syntax](#5-advanced-sql-syntax)
    - [Window Functions](#q16-window-functions-for-analytics)
    - [CTEs vs. Temp Tables](#q17-ctes-common-table-expressions-vs-temporary-tables)

---

## 1. Query Execution & Optimization

### Q1: Anatomy of a Slow Query: How do you debug it?
**The Scenario:** A critical API endpoint has degraded from 50ms to 2s.
**The Answer:**
1.  **Identify:** Check APM tools (Datadog/NewRelic) or the DB "Slow Query Log" to find the exact SQL.
2.  **Explain Plan:** Run `EXPLAIN (ANALYZE, BUFFERS)` (Postgres) or `EXPLAIN FORMAT=JSON` (MySQL).
3.  **Analyze the Plan:**
    * **Scan Type:** Look for `Seq Scan` (Postgres) or `ALL` (MySQL) on large tables. This implies a missing index.
    * **Cardinality Estimation:** Does the DB think it returns 5 rows, but actually returns 500,000? If so, `ANALYZE` the table to update statistics.
    * **Disk vs. Memory:** Check if the query is doing an "External Sort" (spilling to disk) because `work_mem` is too low.
4.  **Remediation:** Add indexes, rewrite the query to be sargable, or reduce the selected columns to enable an "Index Only Scan".

### Q2: Clustered vs. Non-Clustered Indexes (Internals)
**Q:** *Explain the physical difference and the performance implication of a "Key Lookup".*
**Answer:**
* **Clustered Index:** The leaf nodes of the B-Tree contain the **actual data pages**. The data is physically sorted on disk by this key. (Primary Key is usually clustered).
* **Non-Clustered (Secondary) Index:** The leaf nodes contain the **Index Key + Pointer** (usually the Primary Key) to the actual row.
* **Key Lookup (or Heap Fetch):** If you query a Secondary Index but select a column *not* in that index, the DB must:
    1.  Traverse the Secondary Index B-Tree.
    2.  Get the PK.
    3.  Traverse the Clustered Index B-Tree to find the row.
    * *Optimization:* Create a **Covering Index** (include all `SELECT` columns in the index) to avoid step 3.

### Q3: Composite Indexes & The Leftmost Prefix Rule
**Q:** *I have an index on `(user_id, status, created_at)`. Will `SELECT * FROM orders WHERE status = 'ACTIVE'` use the index?*
**Answer:**
**No.** (or mostly no, depending on the optimizer's "Skip Scan" capabilities).
* **The Rule:** A B-Tree is strictly ordered. An index on `(A, B, C)` is sorted by A, then B, then C.
* **Analogy:** It's like a phone book sorted by `Last Name`, then `First Name`. You cannot efficiently find everyone named "David" (First Name) without knowing their Last Name.
* **Working Queries:**
    * `WHERE user_id = ?`
    * `WHERE user_id = ? AND status = ?`
* **Broken Queries:**
    * `WHERE status = ?` (Skips the leading column).
    * `WHERE created_at = ?`

### Q4: What makes a query "Non-Sargable"?
**Q:** *Why is `SELECT * FROM users WHERE YEAR(created_at) = 2023` slow?*
**Answer:**
**SARG** = **S**earch **ARG**ument **ABLE**.
* **The Issue:** Wrapping a column in a function (`YEAR(col)`) hides the raw data from the B-Tree. The database must calculate the function for *every single row* in the table (Full Table Scan) to see if it matches.
* **The Fix:** Rewrite the query to compare raw column values against a range.
    ```sql
    -- Bad (Non-Sargable)
    SELECT * FROM users WHERE YEAR(created_at) = 2023;

    -- Good (Sargable)
    SELECT * FROM users 
    WHERE created_at >= '2023-01-01 00:00:00' 
      AND created_at <  '2024-01-01 00:00:00';
    ```

---

## 2. Concurrency, Locking & ACID

### Q5: Explain Isolation Levels and Read Phenomena
**Q:** *What risks do we take by using `READ COMMITTED` (default in Postgres) vs `REPEATABLE READ` (default in MySQL)?*
**Answer:**

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
| :--- | :---: | :---: | :---: |
| **Read Uncommitted** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Read Committed** | ❌ No | ✅ Yes | ✅ Yes |
| **Repeatable Read** | ❌ No | ❌ No | ⚠️ Maybe* |
| **Serializable** | ❌ No | ❌ No | ❌ No |

* **Dirty Read:** Reading uncommitted data (e.g., catching a transaction mid-air that eventually rolls back).
* **Non-Repeatable Read:** You query Row X (value=10). Another transaction commits X=20. You query Row X again (value=20). The row changed *during* your transaction.
* **Phantom Read:** You query `WHERE age > 20` (returns 5 rows). Another transaction inserts a new person (age 25). You query again, now you see 6 rows.

### Q6: How does MVCC work internally?
**Q:** *How does Postgres/MySQL allow reading without blocking writers?*
**Answer:**
**Multi-Version Concurrency Control (MVCC)** treats data as immutable versions.
1.  **Writes:** When you `UPDATE` a row, the DB doesn't overwrite it. It marks the old row as "dead" (for future transactions) and inserts a **new version** of the row.
2.  **Reads:** Each transaction is assigned a snapshot ID. It only sees row versions that were "committed" before the snapshot began.
3.  **Implication:** Readers and Writers do not block each other.
4.  **Maintenance:** In Postgres, `VACUUM` is required to clean up the "dead tuples." In MySQL (InnoDB), the Undo Log handles this.

### Q7: Handling Deadlocks in High-Concurrency Systems
**Q:** *Your logs show `Deadlock found when trying to get lock`. How do you fix this?*
**Answer:**
A deadlock occurs when Transaction A holds Lock 1 and waits for Lock 2, while Transaction B holds Lock 2 and waits for Lock 1.
1.  **Consistent Ordering:** Ensure your code acquires locks in the same order.
    * *Bad:* Class A locks User, then Wallet. Class B locks Wallet, then User.
    * *Fix:* Always lock User, then Wallet.
2.  **Atomic Updates:** Instead of `Read -> Calculate in Java -> Write`, use DB-side calculation: `UPDATE accounts SET balance = balance - 10 WHERE id = ?`.
3.  **Retry Logic:** Deadlocks are sometimes unavoidable in high load. Catch the specific SQL exception (e.g., `ORA-00060`, MySQL `1213`) and retry the transaction.

### Q8: Optimistic vs. Pessimistic Locking Strategies
**Q:** *When designing a ticket booking system, which locking strategy do you use?*
**Answer:**
* **Pessimistic (`SELECT ... FOR UPDATE`):**
    * Locks the row immediately. No one else can read/write it until you commit.
    * *Use when:* Contention is high (e.g., only 1 seat left, 1000 users trying to buy). Prevents users from wasting time filling forms only to fail at the end.
* **Optimistic (`@Version` column):**
    * No DB locks. Read the row (version 1). When updating, check: `UPDATE table SET val=new, ver=2 WHERE id=1 AND ver=1`.
    * If 0 rows updated, someone else modified it. Throw exception.
    * *Use when:* Contention is low. Better for system throughput/scalability.

---

## 3. Schema Design & Distributed Systems

### Q9: Normalization vs. Denormalization at Scale
**Q:** *Why would you intentionally violate 3rd Normal Form?*
**Answer:**
You normalize to reduce data redundancy and anomalies (Writes). You denormalize to optimize Read performance (Reads).
* **Scenario:** A `Comments` table in a social app.
* **Normalized:** `Comments` table has `user_id`. To show the username, you must `JOIN Users` on every read.
* **Denormalized:** Store `username` and `user_avatar_url` *inside* the `Comments` table.
* **Trade-off:** Reads are instant (no Joins). However, if the user changes their avatar, you must run an expensive background job to update millions of old comments (Write Amplification).

### Q10: Horizontal Sharding vs. Partitioning
**Q:** *What is the difference and how do you choose a Shard Key?*
**Answer:**
* **Partitioning:** Splitting a table into smaller chunks within the **same** database instance (e.g., `Orders_2022`, `Orders_2023`). Good for manageability and deleting old data.
* **Sharding:** Distributing data across **multiple** database servers. Good for infinite scaling of writes/storage.
* **Shard Key Selection:**
    * *Cardinality:* Must be high (e.g., `User_ID`, not `State`).
    * *Distribution:* Must be even. Using `Date` creates "Hot Spots" (all writes go to the current day's shard). `User_ID` or `Entity_ID` is usually best.

### Q11: JSONB vs. Relational Columns
**Q:** *Postgres supports JSONB. Should we just use that and abandon columns?*
**Answer:**
**No.**
* **Use Columns:** For core attributes that you query frequently, filter on, or use in Foreign Keys (e.g., `email`, `status`, `account_id`). Relational columns are smaller and faster to index.
* **Use JSONB:** For "attributes" that vary per row (e.g., Product Metadata, Feature Flags, 3rd party API responses).
* **Indexing JSON:** You can use GIN indexes in Postgres to query inside the JSON structure efficiently (`WHERE data @> '{"color": "red"}'`).

### Q12: Primary Key Strategies: UUID vs. Sequence vs. Snowflake
**Q:** *Why not just use Auto-Increment Integer for everything?*
**Answer:**
* **Auto-Increment (Sequence):**
    * *Pros:* Small (4/8 bytes), fast indexing, naturally sorted.
    * *Cons:* Leaks business info (User ID 500 means you have 500 users), hard to merge databases later, hard to shard (collision risk).
* **UUID (v4):**
    * *Pros:* Globally unique, can generate in Java (no DB roundtrip needed).
    * *Cons:* Large (16 bytes), random string causes **Index Fragmentation** (inserts happen randomly in the B-Tree pages, causing page splits and poor cache locality).
* **Snowflake / TSID (Time-Sorted ID):**
    * *Best of both:* Uses 64 bits. Starts with a Timestamp (sortable), includes Machine ID (distributed safe). Twitter Snowflake or Instagram ID pattern.

---

## 4. Java & Database Interaction (ORM)

### Q13: The N+1 Select Problem in Hibernate
**Q:** *Explain it and provide the solution.*
**Answer:**
* **The Bug:** You fetch a list of `Authors`. You iterate over them and call `author.getBooks()`.
    * Hibernate runs 1 query for Authors.
    * Then N queries (one per author) to fetch Books.
* **The Fix:**
    1.  **JPQL `JOIN FETCH`:**
        ```java
        @Query("SELECT a FROM Author a JOIN FETCH a.books")
        List<Author> findAllWithBooks();
        ```
    2.  **Entity Graphs:** Use `@NamedEntityGraph` to specify fetch plans dynamically.
    3.  **Batch Fetching:** Set `hibernate.default_batch_fetch_size = 50`. This changes N queries into N/50 queries using `WHERE id IN (...)`.

### Q14: Connection Pool Sizing and Configuration
**Q:** *We have a 64-core DB server. Should we set the connection pool size to 1000?*
**Answer:**
**No.**
* **Context Switching:** The CPU can only execute one thread per core. If 1000 connections are active, the OS spends more time swapping threads than executing SQL.
* **HikariCP Formula:** `connections = ((core_count * 2) + effective_spindle_count)`.
* For a 64-core server, a pool of ~130 is likely optimal. It is better to have requests queue in the application layer (cheap) than in the database layer (expensive).

### Q15: Spring Transactional Pitfalls
**Q:** *Why does this code fail to rollback on error?*
```java
@Transactional
public void doWork() {
    try {
        repo.save(entity);
    } catch (Exception e) {
        // log error
    }
}
```

Answer:
1. Swallowing the Exception: The catch block eats the exception. The Transaction Manager proxy never sees the error, so it commits. You must throw e or use TransactionAspectSupport.currentTransactionStatus().setRollbackOnly().
2. Checked Exceptions: By default, Spring only rolls back on RuntimeException (Unchecked). It does not rollback on CheckedException unless you specify @Transactional(rollbackFor = Exception.class).
5. Advanced SQL Syntax
### Q16: Window Functions for Analytics
**Q:** *Write a query to find the 3 highest-paid employees in each department.*
Answer:
```sql
Use DENSE_RANK() or ROW_NUMBER().
WITH RankedEmployees AS (
    SELECT 
        emp_name, 
        dept_id, 
        salary,
        DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) as rnk
    FROM employees
)
SELECT * FROM RankedEmployees WHERE rnk <= 3;
```
Why DENSE_RANK? If two people tie for 1st place, the next person is 2nd. If you use RANK, the next person is 3rd.
Q17: CTEs (Common Table Expressions) vs. Temporary Tables
Q: When do you use a CTE vs. a Temp Table?
Answer:
• CTE (WITH clause):
• Best for readability and breaking down complex logic in a single query.
• Scope is limited to that one statement.
• (Postgres specific) CTEs act as an "optimization fence" in older versions (pre-12), meaning the optimizer couldn't push predicates inside them.
• Temporary Table (CREATE TEMP TABLE):
• Best for multi-step processing where you need to verify data in between steps.
• You can create Indexes on Temp Tables to speed up subsequent joins.
• Scope persists for the whole session/transaction.


