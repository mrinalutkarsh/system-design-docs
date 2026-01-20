# 📊 Apache Cassandra - Wide Column NoSQL Database

## 🎯 Overview

Apache Cassandra is a **distributed, highly available, eventually consistent, wide-column NoSQL database** designed to handle huge amounts of structured data across many commodity servers.

### ✨ Key Characteristics
- 🔄 **Peer-to-peer architecture** - No single point of failure
- 📈 **Linear scalability** - Performance increases linearly with added nodes
- ✍️ **Write-optimized** - Exceptional write throughput
- 🌍 **Multi-datacenter replication** - Geographic distribution support
- 🚫 **No master node** - Every node can serve client requests

---

## 🏗️ Data Model

### Basic Building Blocks

```
📦 Column (Key-Value Pair)
   ├── Column Key: Unique identifier
   └── Column Value: Stores one or collection of values

📋 Row: Collection of columns
📁 Table: Container of rows
🗄️ Keyspace: Container for tables (spans one or more nodes)
🌐 Cluster: Container of keyspaces
💻 Node: Computer system running Cassandra instance
```

### Hierarchy
```
Cluster
  └── Keyspace(s)
       └── Table(s)
            └── Row(s)
                 └── Column(s)
```

### 🎯 Use Cases
Perfect for:
- ⏱️ Time-series data (IoT sensor logs)
- 📊 Streaming services
- 💬 Messaging applications
- 📝 Activity logging
- 🌡️ Weather data
- 📈 Financial transactions

---

## 🔑 Partitioning & Data Distribution

### Partitioner

The **Partitioner** determines how data is distributed across the cluster using a **consistent hash ring**.

```
Request → Partition Key → Murmur3 Hash → Token Ring → Node Assignment
```

#### 🔍 How Coordinator Finds Nodes

When a request arrives:
1. **Takes the partition key**
2. **Hashes it using Murmur3** (default hashing function)
3. **Maps hash to token ring**
4. **Builds preference list** (replica locations)

**Key Benefit**: Decouples client access from data ownership → enables linear scalability without masters

---

## 🔁 Replication

### Replication Factor (RF)
- **RF = 3** means each row stored on **3 different nodes**
- Each keyspace can have **different RF**

### 📋 Replication Strategies

#### 1️⃣ Simple Strategy
- For **single datacenter** deployments
- First replica placed by partitioner
- Subsequent replicas on **next nodes clockwise**

#### 2️⃣ Network Topology Strategy (Production Default)
- For **multi-datacenter** deployments
- Different RF per datacenter
- Rack-aware placement

### 🎯 Consistency Levels

```
Quorum = floor(RF/2 + 1)

Examples (RF = 3):
├── ONE: 1 node must respond
├── QUORUM: 2 nodes must respond  
├── ALL: 3 nodes must respond
├── LOCAL_QUORUM: Quorum in same DC
└── EACH_QUORUM: Quorum in each DC
```

**Trade-off**: Higher consistency = Higher latency

---

## 📝 Write Path (High Performance Secret)

### Write Flow
```
┌─────────────┐
│Write Request│
└──────┬──────┘
       │
       ├──────────────────┐
       ↓                  ↓
┌─────────────┐    ┌──────────┐
│ CommitLog   │    │MemTable  │
│  (Disk)     │    │ (Memory) │
│  WAL/Append │    │  Sorted  │
└─────────────┘    └────┬─────┘
                        │ (when full)
                        ↓
                  ┌──────────┐
                  │ SSTable  │
                  │  (Disk)  │
                  │Immutable │
                  └──────────┘
```
![Cassandra's write path](../images/cassandraWritePath.png)

### 🔥 Why Writes Are Fast
1. **Sequential append** to CommitLog (no seek time)
2. **In-memory writes** to MemTable (fast)
3. **Batch flushes** to SSTables (efficient)
4. **No read-before-write** (unlike B-trees)

### ⚙️ Components

#### CommitLog (Write-Ahead Log)
- 💾 Durability guarantee
- 📊 Sequential writes (append-only)
- 🔄 Replayed on node restart
- 📦 **Segmented**: Multiple files, archived when data flushed

**Default**: 3 segments, each grows until threshold

#### MemTable
- 🧠 One per table per node (in-memory)
- 🔀 Stores data **sorted by partition + clustering keys**
- ⚡ Serves reads for unflushed data
- 💧 Flushed when size threshold reached
![Storing data to commitlog and memtable](../images/storingDataToCommitLogAndMemTable.png)

**CommitLog vs MemTable**:
```
CommitLog: Sequential Order → Fast writes
MemTable:  Sorted Order → Fast reads
```

#### SSTable (Sorted String Table)
- 💽 Immutable on-disk files
- 🔒 **Cannot be modified** after creation
- 🗑️ Updates/Deletes = New write operations
- 📑 Multiple SSTables per table
![Cassandra read operation workflow](../images/cassandraReadOperationWorkflow.png)

**Q: How to update/delete if immutable?**
**A**: Cassandra writes a new version with timestamp/tombstone

---

## 📖 Read Path

### Read Flow
```
Read Request
    ↓
┌─────────────┐
│  Row Cache  │ (Hot rows)
└──────┬──────┘
       │ miss
       ↓
┌─────────────┐
│  Key Cache  │ (Partition key → SSTable offset)
└──────┬──────┘
       │ miss
       ↓
┌─────────────────┐
│  Bloom Filter   │ (Key exists in SSTable?)
└──────┬──────────┘
       │ probably yes
       ↓
┌──────────────────┐
│ Partition Index  │
│  Summary (RAM)   │
└──────┬───────────┘
       ↓
┌──────────────────┐
│ Partition Index  │
│   File (Disk)    │
└──────┬───────────┘
       ↓
┌─────────────┐
│  Data File  │
└─────────────┘
```
![Anatomy of Cassandra write path](../images/anatomyOfCassandraWritePath.png)

### 🚀 Caching Layers

1. **Row Cache** 🔥
   - Caches entire frequently-read rows
   - Stored off-heap
   - Best for read-heavy workloads

2. **Key Cache** 🗝️
   - Maps partition keys → SSTable offsets
   - Stored on-heap
   - Updated on every write (can slow writes)

3. **Chunk Cache** 📦
   - Uncompressed data chunks from SSTables
   - Frequently accessed data

![Anatomy of Cassandra read path](../images/anatomyOfCassandraReadPath.png)

### 🌸 Bloom Filters
- **Probabilistic data structure**
- Tells if key **might exist** in SSTable
- **No false negatives** (if says "no", definitely not there)
- **Possible false positives** (if says "yes", might not be there)
- One per SSTable (stored in RAM)

**Benefit**: Avoids unnecessary disk reads

---

## 🔧 Compaction

### 💡 Why Compaction?

```
Problem: Multiple SSTables accumulate over time
         ↓
Solution: Merge them into fewer, larger SSTables
         ↓
Benefits: 
  - Faster reads (fewer files to check)
  - Remove deleted data (tombstones)
  - Reclaim disk space
```

### 📊 Compaction Strategies

#### 1️⃣ Size-Tiered Compaction Strategy (STCS)
- Merges SSTables of **similar size**
- **Best for**: Write-heavy, time-series data
- **Drawback**: Can use 2x disk space temporarily

#### 2️⃣ Leveled Compaction Strategy (LCS)
- Organizes SSTables into **levels** (L0, L1, L2...)
- Each level 10x larger than previous
- **Best for**: Read-heavy workloads
- **Benefit**: Predictable read performance

#### 3️⃣ Time-Window Compaction Strategy (TWCS)
- Compacts within **time windows**
- **Best for**: Time-series with TTL
- **Example**: Compact hourly/daily buckets

---

## 🪦 Tombstones (The Delete Challenge)

### 🤔 The Problem
```
Node A: DELETE key=123
Node B: Offline (missed the delete)
         ↓
Node B comes back online
         ↓
Repair runs → Node B resurrects deleted data! 😱
```

### ✅ The Solution: Tombstones
- Delete = **Soft delete** (mark with tombstone)
- Tombstone = marker with **timestamp**
- Default TTL: **10 days** (gc_grace_seconds)
- Removed during compaction

### ⚠️ Tombstone Problems
```java
// Anti-pattern: Many deletes
for (int i = 0; i < 1000000; i++) {
    session.execute("DELETE FROM users WHERE id = ?", i);
}
// Creates 1M tombstones → slow reads!
```

**Impact**: Accumulated tombstones → slow reads → timeouts

**Best Practice**: Use TTL for auto-expiring data instead of explicit deletes

---

## 🛡️ High Availability Features

### 🤝 Hinted Handoff

When a node is **down**:
```
1. Coordinator receives write
2. Target node is unavailable
3. Coordinator stores "hint" on local disk
   (hint = data + target node info)
4. Every 10 minutes, checks if target recovered
5. Replays hints when target is back
```

**Limitations**:
- ⚠️ Hints stored for **3 hours** (default)
- ⚠️ Data lost if coordinator dies
- ⚠️ Not a replacement for repair

### 👥 Gossip Protocol

**Purpose**: Node discovery and failure detection

Every node, **every second**:
```
1. Picks 1-3 random nodes
2. Exchanges state information
   - Endpoint state
   - Generation number
   - Heartbeat
   - Application state
3. Updates local view of cluster
```

#### Generation Number
- Incremented on **node restart**
- Helps detect node restarts vs. network issues

### 🩺 Failure Detection: Phi Accrual

**Traditional Heartbeat**: Boolean (alive/dead)
```
Problem: Hard to pick timeout
  - Too short → false positives
  - Too long → slow detection
```

**Phi Accrual Solution**: Suspicion level (0 to ∞)
```
Φ (phi) value:
├── 0-1: Healthy
├── 1-5: Slightly suspicious
├── 5-8: Moderately suspicious  
└── 8+: Likely dead (default threshold = 8)
```

**Adaptive**: Uses **historical heartbeat data** to adjust threshold

---

## 🕵️ Snitch (Topology Awareness)

**Purpose**: Understands cluster topology

**Responsibilities**:
1. 🗺️ Determines datacenter/rack of nodes
2. ⚡ Monitors read latencies
3. 📍 Guides replica placement
4. 🚫 Avoids slow nodes

**Common Snitches**:
- **SimpleSnitch**: Single datacenter
- **GossipingPropertyFileSnitch**: Multi-DC (production)
- **Ec2Snitch**: AWS deployments
- **GoogleCloudSnitch**: GCP deployments

---

## 💾 SSTable Storage on Disk

Each SSTable consists of:

```
📂 SSTable Components
├── 📄 Data File (actual data)
├── 🔑 Partition Index (partition key → offset)
├── 📋 Partition Summary (in RAM, subset of index)
├── 🌸 Bloom Filter (in RAM)
├── 📊 Statistics (metadata)
└── 🗜️ Compression Info
```

---

## 🎓 Interview Key Points

### 🔥 Why Cassandra for Writes?
```
1. Append-only CommitLog (sequential I/O)
2. In-memory MemTable (RAM speed)
3. Batch writes to SSTables
4. No locks/latches needed
5. Linear scalability
```

### 📖 Why Reads Can Be Slower?
```
1. Check MemTable
2. Check Row Cache
3. Scan multiple SSTables
4. Merge results by timestamp
5. Return latest version
```

**Solution**: Compaction + proper cache tuning

### ⚖️ CAP Theorem Position
```
Cassandra = AP System
├── A: Availability (always responds)
├── P: Partition Tolerance (works during network splits)
└── C: Eventually Consistent (tunable)
```

**Tunable Consistency**: Can achieve CP by using QUORUM/ALL

### 🎯 When to Use Cassandra?
✅ **Good for**:
- High write throughput needs
- Linear scalability required
- Multi-datacenter deployment
- Time-series data
- No complex joins needed

❌ **Bad for**:
- Complex queries/joins
- Strong ACID transactions
- Frequent updates/deletes
- Small datasets (<100GB)

---

## 🛠️ Common Interview Questions

### Q1: How does Cassandra achieve high write performance?
**Answer**: 
- Sequential writes to CommitLog (append-only)
- In-memory MemTable for immediate acknowledgment
- No read-before-write
- Batch flushes to immutable SSTables
- No locking required

### Q2: Explain the read path in Cassandra
**Answer**:
1. Check Row Cache → return if hit
2. Check Bloom filters of SSTables
3. Use Key Cache or Partition Index to find location
4. Read from MemTable + relevant SSTables
5. Merge results using timestamp (last write wins)
6. Return to client

### Q3: What are tombstones and why are they problematic?
**Answer**:
- Tombstones mark deleted data (soft delete)
- Needed for eventual consistency
- Problem: Accumulation slows reads (must scan all tombstones)
- Solution: Use TTL, tune gc_grace_seconds, monitor tombstone warnings

### Q4: Difference between Cassandra and MongoDB?
**Answer**:
```
Cassandra:
- Wide-column store
- AP (Available, Partition-tolerant)
- Peer-to-peer (no master)
- Better for writes
- Multi-DC built-in

MongoDB:
- Document store
- CP (Consistent, Partition-tolerant)
- Primary-Secondary (has master)
- Flexible schema
- Better for complex queries
```

### Q5: How to choose consistency level?
**Answer**:
```
Use Case → Consistency Level
├── Strong consistency → QUORUM/ALL
├── Fast reads → ONE
├── Fast writes → ONE
├── Multi-DC consistency → LOCAL_QUORUM
└── Critical data → EACH_QUORUM
```

---

## 📚 Quick Reference

### Consistency Formulas
```
Quorum = floor(RF/2) + 1

For RF=3:
- Quorum = 2
- Strong consistency: Read(QUORUM) + Write(QUORUM)
```

### Key Terminology Cheat Sheet
```
🔑 Partition Key: Determines node placement
🔀 Clustering Key: Determines sort order within partition
📊 Primary Key: Partition Key + Clustering Key(s)
💍 Token: Hash of partition key
🔄 Replica: Copy of data on different node
⏱️ Timestamp: Conflict resolution (last write wins)
🪦 Tombstone: Soft delete marker
🌸 Bloom Filter: Probabilistic existence check
```

---

## 🚀 Performance Tuning Tips

1. **Choose appropriate Compaction Strategy** for workload
2. **Tune cache sizes** based on RAM availability
3. **Use TTL** instead of deletes when possible
4. **Monitor tombstone warnings** in logs
5. **Partition data evenly** (avoid hot partitions)
6. **Use appropriate consistency levels** (don't always use QUORUM)
7. **Enable compression** for large datasets
8. **Run regular repairs** in multi-DC setups

---

## ✅ Best Practices

### Data Modeling
```java
// ✅ Good: Time-series partition
CREATE TABLE sensor_data (
    sensor_id UUID,
    day DATE,
    hour INT,
    reading DOUBLE,
    PRIMARY KEY ((sensor_id, day), hour)
);

// ❌ Bad: Unbounded partition
CREATE TABLE user_events (
    user_id UUID,
    event_time TIMESTAMP,
    event_data TEXT,
    PRIMARY KEY (user_id, event_time)
);
// Problem: One user's partition grows forever
```

### Write Patterns
```java
// ✅ Good: Batch same partition
BatchStatement batch = new BatchStatement();
batch.add(insert1); // same partition key
batch.add(insert2); // same partition key
session.execute(batch);

// ❌ Bad: Batch different partitions
// Coordinator becomes bottleneck
```

---

**Last Updated**: January 2026
**Version**: Cassandra 4.x+

---

*💡 Pro Tip: Always model your queries first, then design your tables. "Model your queries, not your data!"*