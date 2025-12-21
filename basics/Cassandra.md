# 🟣 Apache Cassandra

> If **availability**, **horizontal scale**, and **write throughput** matter more than **strict consistency**, Cassandra is usually on the table.

---

## 📌 What is **Apache Cassandra**?

**Apache Cassandra** is a **distributed, wide-column NoSQL database** designed for:

* ⚡ **High write throughput**
* 🌍 **Multi-datacenter replication**
* 💥 **No single point of failure**
* 📈 **Linear horizontal scalability**

It is heavily used by companies that need to ingest **massive volumes of time-series or event data** with **predictable low latency**.

---

## 🧠 One-Line Interview Definition

> *Cassandra is a peer-to-peer, partitioned, replicated, eventually consistent database optimized for high availability and fast writes.*

---

## 🧩 What Problems Was Cassandra Built to Solve?

Traditional RDBMS pain points at scale:

❌ Single master bottleneck
❌ Vertical scaling limits
❌ Expensive joins
❌ Downtime during failover

Cassandra solves:

✅ Always writable
✅ Multi-region active-active
✅ Cheap horizontal scaling
✅ Predictable performance at scale

---

## 🏆 Why Is Cassandra So Popular?

### 1️⃣ **Masterless Architecture**

* Every node is equal
* Any node can accept reads & writes
* No leader election delays

### 2️⃣ **Linear Scalability**

* Add nodes → get proportional throughput
* No rebalancing nightmares

### 3️⃣ **High Write Performance**

* Writes are sequential (append-only)
* No random disk I/O during writes

### 4️⃣ **Multi-DC Replication**

* Built-in support
* Used for geo-distributed systems

### 5️⃣ **Battle-Tested**

Used by:

* Netflix
* Apple
* Instagram
* Uber
* Discord

---

## ❌ When **NOT** to Use Cassandra (Very Important)

| Requirement                  | Cassandra Fit |
| ---------------------------- | ------------- |
| Complex joins                | ❌             |
| Ad-hoc queries               | ❌             |
| Strong consistency           | ❌             |
| Small datasets               | ❌             |
| Frequent updates to same row | ❌             |
| Financial transactions       | ❌             |

👉 If you need **ACID**, choose **Postgres / MySQL / Spanner**
👉 If you need **flexible querying**, choose **MongoDB**
👉 If you need **strong consistency**, choose **HBase / Spanner**

---

## 🟢 When Should You Use Cassandra?

Perfect fit for:

✅ Time-series data
✅ Event logs
✅ Messaging metadata
✅ User activity tracking
✅ IoT telemetry
✅ Recommendation feeds
✅ Metrics & monitoring data

---

## 🧱 Data Model (Wide Column Store)

```
Keyspace
 └── Table
      └── Partition Key → determines node
           └── Clustering Columns → sort within partition
                └── Columns
```

### Example

```sql
PRIMARY KEY ((user_id), timestamp)
```

* `user_id` → partition key
* `timestamp` → clustering column

📌 **Rule:**

> Model queries first, data second.

---

## 🧠 Partition Key – The MOST Important Concept

### Good Partition Key

* High cardinality
* Even distribution
* Prevents hotspots

### Bad Partition Key

❌ country
❌ status
❌ boolean

🚨 Hot partitions = performance death

---

## 🗺️ Cassandra Architecture (High Level)

```
Client
  |
  v
Coordinator Node
  |
  +--> Replica Node 1
  +--> Replica Node 2
  +--> Replica Node 3
```

* Client connects to **any node**
* That node becomes **Coordinator**
* Coordinator talks to replicas

---

## 🔁 Peer-to-Peer, Not Master-Slave

```
Node A  <--> Node B <--> Node C <--> Node D
   ^          ^          ^          ^
   +----------+----------+----------+
            Ring Topology
```

* Nodes arranged in a **consistent hashing ring**
* No leader
* No single point of failure

---

## 🧮 Consistent Hashing & Tokens

* Data is assigned via **tokens**
* Each node owns multiple token ranges
* Adding nodes = minimal reshuffling

---

## 📦 Replication & Replication Factor (RF)

```
RF = 3
```

Means:

* Each partition stored on **3 nodes**

### Strategies

* `SimpleStrategy` → single DC
* `NetworkTopologyStrategy` → multi-DC (REAL WORLD)

---

## ⚖️ Consistency Levels (Interview Favorite ⭐)

### Writes

* `ONE`
* `QUORUM`
* `ALL`

### Reads

* Same options

### Rule

```
R + W > RF  → Strong consistency
```

### Example

```
RF = 3
R = 2
W = 2
2 + 2 > 3  ✅
```

📌 Cassandra gives **tunable consistency**

---

## 🧪 What If Nodes Are Out of Sync?

### Mechanisms:

* **Read Repair**
* **Hinted Handoff**
* **Anti-Entropy Repair**

Eventually… data converges.

---

## ✍️ Write Path (VERY IMPORTANT)

```
Client
  |
  v
Commit Log (Disk append)
  |
  v
Memtable (Memory)
  |
  v
ACK to client
```

Later:

```
Memtable → SSTable (Disk)
```

### Why writes are fast:

* Sequential disk writes
* No random I/O
* No locking

---

## 📖 Read Path

```
Client
  |
  v
Memtable
  |
  v
Bloom Filter
  |
  v
SSTables
```

### Bloom Filter

* Probabilistic
* Avoids unnecessary disk reads

---

## 📂 SSTables & LSM Tree

Cassandra uses **LSM Tree**:

```
Writes → Memtable → SSTable
Multiple SSTables → Compaction
```

### Compaction Types

* SizeTiered
* Leveled
* TimeWindow (best for time-series)

---

## 🧹 Tombstones (Interview Trap ⚠️)

* Deletes don’t remove data immediately
* They create **tombstones**
* Cleaned during compaction

🚨 Too many tombstones = slow reads

---

## 🧠 CAP Theorem Position

```
Cassandra chooses: AP
```

* Availability ✅
* Partition tolerance ✅
* Consistency ❌ (but tunable)

---

## 🔄 Failure Handling

### Node Failure

* Writes still accepted
* Hints stored
* Replayed later

### Network Partition

* Both sides continue writing
* Conflict resolution later

---

## 🧑‍💻 Query Model (CQL)

* SQL-like syntax
* NOT relational semantics
* No joins
* No subqueries

---

## 🛑 Cassandra Anti-Patterns

❌ Secondary indexes at scale
❌ Large partitions (>100MB)
❌ High cardinality clustering keys
❌ Frequent updates to same row
❌ Ad-hoc querying

---

## 🧪 Real Interview Use-Cases

### 1️⃣ Metrics System

* Partition: metric_id
* Clustering: timestamp

### 2️⃣ Messaging Metadata

* Partition: conversation_id
* Clustering: message_time

### 3️⃣ User Activity Feed

* Partition: user_id
* Clustering: activity_time

---

## ⚙️ Operational Concerns (Senior-Level)

### Monitoring

* Compaction backlog
* Tombstone count
* Disk usage
* Latency percentiles

### Repair

* Run regularly (weekly/monthly)

### Scaling

* Add nodes → rebalance tokens

---

## 🆚 Cassandra vs Others

| Feature            | Cassandra | MongoDB | DynamoDB |
| ------------------ | --------- | ------- | -------- |
| Masterless         | ✅         | ❌       | ✅        |
| Strong consistency | ❌         | ❌       | Optional |
| Write throughput   | ⭐⭐⭐⭐⭐     | ⭐⭐⭐     | ⭐⭐⭐⭐⭐    |
| Multi-DC           | ⭐⭐⭐⭐⭐     | ⭐⭐      | ⭐⭐⭐⭐⭐    |

---

## 🧠 Final Interview Summary (Say This)

> *Cassandra is ideal when you need always-on writes, massive scale, and geo-replication. It trades strong consistency and query flexibility for availability, scalability, and predictable performance.*

---

## 🧾 One-Page Memory Hook 🧠

```
No Master
High Writes
Wide Rows
Partition Key Critical
Eventually Consistent
LSM + SSTables
AP System
```

---