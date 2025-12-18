# 🧠 Distributed Consistency, Quorums & Conflict Resolution  
*A practical, interview-ready summary*

---

## 📌 1. Replication Basics — `N`, `R`, `W`

- **N** = total number of replicas
- **W** = number of replicas that must ACK a write
- **R** = number of replicas queried during a read

### ✅ Valid Values

- 1 ≤ R ≤ N
- 1 ≤ W ≤ N

### ❌ Invalid Values

- `0` is **invalid** — a read/write must touch at least one replica.

---

## 🔁 2. The Quorum Rule (Golden Rule)

- **R + W > N** → No stale results (quorum consistency)
- **R + W ≤ N** → Stale reads possible

### 💡 Why This Works

Any **read quorum** and **write quorum** must **overlap**.

That overlap guarantees:

> **At least one replica read has the latest write**

---

## 🧪 3. What `R=1, W=1` or `R=1, W=2` Mean

### ⚡ Case 1: `R=1, W=1` (Fastest, weakest)

- Write → wait for 1 replica
- Read → read from 1 replica

```
A (new)   B (old)   C (old)
↑
write acked here
```

Read may hit `B` or `C` → ❌ **stale read possible**

**Pros:**
- ✅ Lowest latency

**Cons:**
- ❌ Eventual consistency only

---

### ⚖️ Case 2: `R=1, W=2`

- Write → wait for 2 replicas
- Read → read from 1 replica

```
A (new)   B (new)   C (old)
↑
read hits here
```

❗ Even though write succeeded, read can be stale.

**Key Insight:**

> `R=1` means *stale reads are possible*, not guaranteed.

---

## 🛡️ 4. Why `R=2, W=2` Prevents Stale Reads (N=3)

### Setup

- Replicas: A, B, C
- Write → A, B (new)
- C remains old

```
A (new)   B (new)   C (old)
```

### Possible Reads (R = 2)

- {A, B} → both new
- {A, C} → one new
- {B, C} → one new

🚫 Impossible to read two stale replicas.

### 🎯 Guarantee

- At most **1 replica** can be stale
- Read must touch **2 replicas**
- So **at least one replica is always fresh**

**Very Important:**

> Quorum guarantees **no stale result**,
> NOT that all replicas are fresh.

---

## 🧠 Mental Model (Lock This In)

**Quorum ≠ fresh replicas**

**Quorum = correct answer**

---

## 🗂️ 5. LSM Tree & SSTable Relationship

### 🌲 LSM Tree (Log-Structured Merge Tree)

- Write-optimized storage design
- Writes go to memory first
- Disk writes are sequential
- Background compaction merges data

### 📄 SSTable (Sorted String Table)

- Immutable
- Sorted on disk
- Created when MemTable flushes
- Never updated in place

### 🔗 Relationship

```
Client Write
    ↓
MemTable (in-memory, sorted)
    ↓ flush
SSTable (disk, immutable)
    ↓
Compaction (merge SSTables)
```

**One-liner:**

> **LSM Tree is the strategy; SSTables are the building blocks**

---

## 🗳️ 6. Paxos — Distributed Consensus

### ❓ What is Paxos?

A **consensus algorithm** that allows nodes to agree on a value even with failures, delays, and no global clock.

### 👥 Roles

| Role | Responsibility |
|---|---|
| Proposer | Proposes a value |
| Acceptor | Votes |
| Learner | Learns chosen value |

Note: A node can play multiple roles

### 🔄 Two Phases

#### Phase 1 — Prepare 🤝

- Proposer → Prepare(n)
- Acceptors → Promise (no smaller n)

#### Phase 2 — Accept ✅

- Proposer → Accept(n, value)
- Acceptors → Accept if promise holds

✔️ Once **majority accepts**, value is chosen.

### 🧠 Why Paxos Works

Any two majorities must overlap → Two different values cannot be chosen

### 🏭 Used In

- Google Spanner
- ZooKeeper (ZAB – Paxos-like)
- etcd (Raft – Paxos alternative)

---

## 🏁 7. Last Write Wins (LWW)

### ❓ What is LWW?

A **conflict resolution strategy**:

> The write with the **latest timestamp wins**

---

### 🔧 How it Works

```
Replica A → (value=10, ts=100)
Replica B → (value=20, ts=105)

Result → (value=20)
```

---

### 📍 Where LWW Is Used

- Cassandra / Dynamo-style DBs
- Quorum reads & read repair
- CRDT LWW Registers
- Object stores (S3-like)
- Caches

---

### ✅ Pros

- Simple
- Fast
- No coordination

### ❌ Cons

- Data loss on concurrent writes
- Clock skew issues
- Not merge-friendly

---

### 👍 Good Use Cases

- User presence
- Last login time
- Cache entries

### 🚫 Bad Use Cases

- Bank balances
- Counters
- Shopping carts

---

## 🧠 LWW Mental Model

**LWW answers:** "What should I keep?"

**NOT:** "What actually happened?"

---

## 🎯 8. Interview One-Liners

- **Quorum**
  > If read and write quorums overlap, stale results are impossible.

- **R=2, W=2**
  > At least one replica read must have the latest write.

- **LSM & SSTable**
  > LSM Tree is the write-optimized design; SSTables are immutable sorted files.

- **Paxos**
  > A consensus algorithm that uses majority voting to ensure agreement.

- **LWW**
  > A conflict-resolution strategy where the latest timestamp wins.

---

## 🧩 9. Final Takeaway

> **Distributed systems are about trade-offs.**
>
> **Quorums, Paxos, and LWW are tools** — each correct only within its assumptions.