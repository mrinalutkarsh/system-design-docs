# 📦 LSM Trees, SSTables, Memtable Flush & Background Compaction

## 🌲 What is an LSM Tree?
**LSM = Log-Structured Merge Tree**

An **LSM Tree** is a **write-optimized storage architecture** used by databases like:

- Apache Cassandra
- RocksDB
- LevelDB
- HBase
- ScyllaDB

👉 Core idea:
> **Writes are sequential and fast. Reads are optimized later using compaction.**

---

## 🚨 Why LSM Trees Exist
Traditional B-Trees:
- ❌ Random disk writes
- ❌ Slow at high write throughput

LSM Trees:
- ✅ Sequential disk writes
- ✅ High write throughput
- ❌ Reads need extra work (handled by compaction & indexes)

---

## 🧠 High-Level Architecture


```
    Write Path
       |
       v
+----------------+
|   Memtable     |  (In-memory, sorted)
+----------------+
|
|  (flush)
v
+----------------+
|   SSTable      |  (Immutable, on disk)
+----------------+
|
|  (background)
v
+----------------+
| Compaction     |
+----------------+

```

---

## 🧾 What is a Memtable?
A **Memtable** is:

- In-memory
- Sorted (TreeMap / SkipList)
- Stores recent writes
- Very fast to write

### 🔹 Write Flow
```

Client Write
|
v
Write-Ahead Log (WAL)  ➜  crash safety
|
v
Memtable

```

✔ WAL ensures durability  
✔ Memtable ensures speed  

---

## 🚿 What is a Memtable Flush?
A **flush** happens when the memtable becomes **too large**.

### 🔹 What happens during flush?

1. Memtable is frozen (made immutable)
2. Data is written **sequentially** to disk
3. Disk file created → **SSTable**
4. New memtable starts accepting writes

### 🔹 Flush Diagram

```

Memtable (full)
|
v
+-------------------+
| Immutable Memtable|
+-------------------+
|
v
Write to Disk
|
v
+-------------------+
|   SSTable (L0)    |
+-------------------+

```

🚀 Flush = **fast sequential disk write**

---

## 📄 What is an SSTable?
**SSTable = Sorted String Table**

An **SSTable** is:

- Immutable
- Sorted by key
- Stored on disk
- Created from memtable flush

### 🔹 Properties
- ❌ No updates in place
- ❌ No deletes in place
- ✔ Extremely efficient sequential reads

### 🔹 SSTable Structure

```

+--------------------+

| Data Blocks            |
| ---------------------- |
| Index                  |
| --------------------   |
| Bloom Filter           |
| --------------------   |
| Metadata               |
| +--------------------+ |

```

👉 Bloom Filter helps avoid unnecessary disk reads

---

## 🗂️ Why Multiple SSTables Are a Problem
Over time:

- Many memtable flushes
- Many SSTables
- Same key may exist in multiple SSTables

### 🔥 Issues
- Reads must check **multiple files**
- Disk amplification
- Higher latency

➡️ **Solution: Compaction**

---

## 🔄 What is Background Compaction?
**Compaction** is a **background process** that:

- Merges multiple SSTables
- Removes obsolete data
- Deletes old versions of keys

### 🔹 Compaction Goals
- Reduce number of SSTables
- Improve read performance
- Clean up deleted/overwritten data

---

## 🧹 How Compaction Works

```

Before Compaction:

SSTable 1: key1 → v1
SSTable 2: key1 → v2
SSTable 3: key2 → v1

After Compaction:

SSTable New:
key1 → v2
key2 → v1

```

✔ Keeps **latest version only**  
✔ Removes tombstones (deletes)

---

## 📊 Levels in LSM (Simplified)
Most LSM implementations use **levels**:

```

Level 0 (L0): Many small SSTables
Level 1 (L1): Fewer, larger SSTables
Level 2 (L2): Even fewer, bigger SSTables
...

```

### 🔹 Data Movement
```

Memtable
|
v
L0  ➜  L1  ➜  L2  ➜  L3

```

➡️ Each level is **larger but fewer files**

---

## ⚖️ Write vs Read Trade-off

| Operation | Cost |
|---------|------|
| Write | 🟢 Very Fast |
| Read | 🟡 Medium |
| Storage | 🔴 Extra space during compaction |

This is called:
> **Write Amplification vs Read Optimization**

---

## 🎯 Interview One-Liner
> “LSM Trees optimize writes by batching them in memory, flushing them as immutable SSTables, and later merging them using background compaction to keep reads efficient.”

---

## 🧠 Mental Model (Easy to Remember)

```

Write Fast → Sort Later → Merge in Background

```

or

```

RAM → Disk → Cleanup

```

---

## 🧩 Where You’ll See This in Real Systems
- Cassandra → LSM + SSTables
- RocksDB → LSM
- ScyllaDB → LSM
- HBase → LSM
- DynamoDB → LSM-style engine

---

## ✅ Summary
- **LSM Tree** → write-optimized storage design
- **Memtable** → in-memory sorted buffer
- **Memtable Flush** → converts memtable → SSTable
- **SSTable** → immutable sorted disk file
- **Compaction** → merges SSTables in background

---
