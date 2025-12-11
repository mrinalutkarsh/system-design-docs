# 🔀 Sharding & Partitioning

**Purpose, Need, Shard Keys, and Handling Hot Keys**

---

## 1. What Are Partitioning & Sharding?

### 🍰 Partitioning

**Partitioning** means splitting a large dataset into smaller pieces (partitions) stored within the **same database cluster**. Each partition contains a subset of rows and improves **query performance, manageability, and maintenance**.

**Examples:**
- 📅 Partitioning a table by date (Jan → Partition 1, Feb → Partition 2)
- 🗺️ Partitioning by region, user ID ranges, etc.

> **Note:** Partitioning is usually **internal to a single DB system**.

```
       ┌────────────────────────────┐
       │   Single DB (One Server)   │
       └─────────────┬──────────────┘
                     │
                     │
    ┌────────────────┼────────────────────┐
    │                │                    │
┌───────────┐     ┌───────────┐      ┌───────────┐
│ Partition │     │ Partition │      │ Partition │
│  P1       │     │   P2      │      │     P3    │
└───────────┘     └───────────┘      └───────────┘
(date range)        (region)         (userId)
```

### ⚙️ Sharding

**Sharding** means splitting the dataset across multiple physical database servers (shards). Each shard holds a fraction of the total data.

> **Note:** Sharding is **horizontal scaling**.

```
            ┌──────────────────┐
            │   Application    │
            └────────┬─────────┘
                     │
          Shard Key Routing
                     │
    ┌────────────────┼────────────────┐
    │                │                │
┌────────┐      ┌────────┐      ┌────────┐
│ Shard 1│      │ Shard 2│      │ Shard 3│
│Server A│      │Server B│      │Server C│
└────────┘      └────────┘      └────────┘
(1–1M)        (1M–2M)         (2M–3M)
```

**Benefits:**
- 💾 Handle huge datasets that cannot fit on one machine
- ⚡ Reduce read/write load per DB
- 🛡️ Improve availability and performance

**Example:**
- 👥 Users with IDs 1–1M → Shard A
- 👥 Users with IDs 1M–2M → Shard B

---

## 🚀 2. Why Do We Need Sharding?

**Sharding becomes necessary when:**
- 📦 The DB grows too large for a single node (storage or performance limits)
- 📊 High throughput is required (millions of reads/writes)
- 🌍 Latency is critical (regional sharding for low latency)
- 🚫 Avoiding a single point of failure

**Without sharding, you eventually hit:**
- 🐌 Slow queries
- 🔒 Locking issues
- 💰 Expensive vertical scaling
- ⚠️ Single-node performance ceilings

**Solution:** Sharding allows **scaling horizontally**, which is cheaper and limitless (theoretically).

---

## 🗝️ 3. How to Select a Good Shard Key

Choosing the shard key is the **most important decision** in sharding architecture.

### ✅ Characteristics of a Good Shard Key

A **good shard key** should:

1. **Distribute load evenly across shards**
   - Avoid having one shard overloaded while others idle

2. **Support your most common queries**
   - Example: If 90% of queries use `user_id`, shard by `user_id`

3. **Be stable and immutable**
   - Changing the shard key requires migrating data → painful & risky

### 📊 Common Shard Key Strategies

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **Range-based** | user_id 1–1M → Shard 1 | Good for predictable queries | Risk of hotspots if data is sequential |
| **Hash-based** | hash(user_id) % N | Balances data well | Harder for range queries |
| **Geo-based** | region → shard | Helps with latency | Uneven population by region |
| **Category-based** | product category → shard | Useful for e-commerce catalogs | Hot categories = hotspots |

---

## 🔥 4. Handling Hot Keys & Hot Shards

### ⚡ What is a Hot Key?

A **hot key** is a key that receives **disproportionately high traffic**.

**Examples:**
- 🔥 A viral product with millions of reads
- ⭐ A celebrity's user profile frequently accessed

**Consequences:**
- 🔴 CPU spikes
- ⏱️ Latency increases
- 📉 Uneven load distribution

### 🧯 Strategies to Handle Hot Keys

1. **🔀 Add Replicas for Hot Shards**
   - Reads can be spread across multiple replicas

2. **💾 Use Caching (Redis/Memcached)**
   - Cache hot objects so DB is not hit repeatedly

3. **🔑 Use a Compound Shard Key**
   - Example: `(product_id + timestamp bucket)` to spread writes

4. **♻️ Resharding (Rebalancing)**
   - Split the overloaded shard into two or more shards
   - Example: Shard A (IDs 1–1M) becomes A1 (1–500k) and A2 (500k–1M)

5. **🔀 Use Randomized Writes for Sequential Keys**
   - Instead of sequential order IDs, use UUID or hashed ID

---

## ⚖️ 5. Balancing Shards

Balancing ensures all shards have **similar load and storage**.

### 🔄 Balancing Techniques

- **Auto-rebalancing by DB** (MongoDB, YugabyteDB)
- **Moving partitions between shards**
- **Consistent Hashing** (used by DynamoDB, Cassandra)
- **Monitoring hot shards** and redistributing traffic

**Goal:** No single shard should become a bottleneck.

---

## 📋 Summary

- 🍰 **Partitioning:** Splitting data *within* a DB instance
- 🔠 **Sharding:** Splitting data *across multiple DB servers*
- 🔑 **Shard key selection** is crucial for **performance & scalability**
- 🔥 **Hot keys** cause uneven load → handled via **caching, replicas, rebalancing, compound keys**
- ⚖️ **Balanced shards** ensure smooth distributed database performance

---

## 💡 Key Concepts Quick Reference

| Concept | Definition |
|---------|-----------|
| **Partitioning** | Splits a large table into smaller logical parts inside the same DB server |
| **Sharding** | Splits data across multiple physical DB servers |
| **Range-based Shard Key** | Distributes data by ranges (e.g., user_id 1–1M → Shard 1) |
| **Hash-based Shard Key** | Distributes data using hash function (e.g., hash(user_id) % N) |
| **Geo-based Sharding** | Routes data by geographic location for low latency |
| **Hot Key** | A key receiving disproportionately high traffic |
| **Consistent Hashing** | Algorithm for evenly distributing data across shards |

### ❓ Common Interview Questions

**Q: ❓ What is the difference between partitioning and sharding?**

Partitioning splits data inside one database instance. Sharding distributes data across multiple database servers. Sharding provides horizontal scaling; partitioning is more about organization and performance inside a single DB.

**Q: 🤔 Why do we need sharding?**

When a single DB node can't handle storage, CPU, or read/write load, sharding lets us scale horizontally by adding more servers. It reduces bottlenecks and improves availability.

**Q: 🗝️ How do you select a good shard key?**

A good shard key should:
- Evenly distribute data and traffic
- Support common query patterns
- Be stable and immutable

Examples: hashed user_id, geo-based region ID.

**Q: 🔥 What are hot keys and how do you handle them?**

A hot key is a key that receives disproportionate traffic (e.g., a viral product).

Fixes include:
- Add read replicas
- Cache hot items (Redis/Memcached)
- Use compound shard keys
- Perform rebalancing/resharding
- Use consistent hashing

**Q: ⚖️ How do you balance shards?**

Through resharding, auto-balancers (MongoDB), moving partitions, or consistent hashing so no single shard becomes overloaded.
