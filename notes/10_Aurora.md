# Lecture 10 — Database Logging, Quorums & Amazon Aurora

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Paper: *"Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases"* — Verbitski et al., SIGMOD 2017
> Companion FAQ: [10_Aurora_QA.md](10_Aurora_QA.md)

---

## Table of Contents

1. [Why This Paper?](#1-why-this-paper)
2. [The Road to Aurora](#2-the-road-to-aurora)
3. [Database Crash-Course (How a Transactional DB Uses Storage)](#3-database-crash-course-how-a-transactional-db-uses-storage)
4. [Mirrored MySQL — Why It Hurts](#4-mirrored-mysql--why-it-hurts)
5. [Aurora's Two Big Ideas](#5-auroras-two-big-ideas)
6. [Quorum Reads & Writes — A Primer](#6-quorum-reads--writes--a-primer)
7. [Aurora's Storage Protocol](#7-auroras-storage-protocol)
8. [Crash Recovery — VCL, VDL & Friends](#8-crash-recovery--vcl-vdl--friends)
9. [Scaling Beyond One Storage Server](#9-scaling-beyond-one-storage-server)
10. [Read-Only Replicas](#10-read-only-replicas)
11. [Takeaway](#11-takeaway)

---

## 1. Why This Paper?

Aurora is a **commercial cloud database** that delivers ~35× the throughput of mirrored MySQL on similar hardware (Table 1). It's worth studying because:

- It's a recent, successful, paying-customer cloud service.
- It shows the **limits of a general-purpose storage abstraction** (EBS).
- It illustrates the cross-layer optimization mindset: when you control both the database and the storage, you can break layering for huge wins.
- It's full of tidbits about what really matters in cloud infrastructure.

---

## 2. The Road to Aurora

Aurora didn't appear in a vacuum. Amazon's storage offerings evolved through three generations.

### 2.1 Generation 1 — EC2 with local disks

```
   ┌──────────────────────────────────────┐
   │ Physical machine (Amazon datacenter) │
   │  ┌──────────────────────────────┐    │
   │  │ Virtual machine (your code)  │    │
   │  │  Web server  |  MySQL        │    │
   │  └────────┬─────────────────────┘    │
   │           ▼                          │
   │       Local disk (attached)          │
   └──────────────────────────────────────┘
```

- Cheap, fast.
- **Problem**: if the physical machine dies, your disk dies with it. Backup to S3 helps but is coarse.

### 2.2 Generation 2 — EBS

EBS (Elastic Block Store) gives you a disk that **survives EC2 instance failure**.

```
   ┌──────────────┐         ┌──────────────┐
   │ EC2 instance │ ──RPC─▶ │  EBS server  │
   │   MySQL      │         │  + replica   │   chain replication
   │              │         │  (same AZ)   │   Paxos-based config
   └──────────────┘         └──────────────┘
```

- Looks like a disk; you can mount ext4 on it.
- Two replicas via chain replication, both **in the same availability zone**.
- If EC2 dies, restart on a new EC2 and re-attach the EBS volume.

> **Limitations:**
> - Same-AZ replicas → both die if the AZ has a fire, flood, or power cut.
> - The DB ships **full data pages** *and* log over the network, every write.

### 2.3 Generation 3 — Multi-AZ RDS (Figure 2)

Mirror your EBS volume to a second EC2/EBS pair **in a different AZ**.

```
                 Primary AZ                        Standby AZ
   ┌────────────────────────────────┐    ┌─────────────────────────┐
   │ EC2 (MySQL)                    │    │ EC2 (idle)              │
   │      │                         │    │                         │
   │      ▼                         │    │                         │
   │  ┌────┐    ┌────┐              │    │  ┌────┐    ┌────┐       │
   │  │EBS1│ ──▶│EBS2│              │    │  │EBS3│ ──▶│EBS4│       │
   │  └────┘    └────┘              │    │  └────┘    └────┘       │
   │      │                          │    │      ▲                  │
   │      └─── mirror write ─────────┼────┼──────┘                  │
   └────────────────────────────────┘    └─────────────────────────┘

   Every write must reach all FOUR EBS replicas before commit.
```

- Fault-tolerant across AZs ✓
- But **slow**: every commit waits on four EBS replicas, and ships **pages**, not log records.

This is the world Aurora was born to fix.

### 2.4 Analogy — From local file cabinet to interstate vault

- **Local disk** = paper files in your office desk. Fast, but lost if your office burns.
- **EBS** = a fireproof safe in the building. Survives your desk being destroyed; doesn't survive the building.
- **Multi-AZ RDS** = couriers carrying photocopies to a second building. Survives the building, but couriers are slow and copies are bulky.
- **Aurora** = couriers carrying small handwritten **change notices** instead of photocopies, to a network of vaults that know how to apply the notices themselves.

---

## 3. Database Crash-Course (How a Transactional DB Uses Storage)

### 3.1 Example transaction

> Transfer $10 from `x` to `y`.
> ```sql
> x = x + 10
> y = y - 10
> ```

The DB acquires locks on `x` and `y`, modifies cached data pages **in RAM**, and appends to the **Write-Ahead Log (WAL)** — also called the "redo log".

### 3.2 WAL records look like

```
   LSN   TID   key   old   new
   ─────────────────────────────
   101    7    x    500   510      ← x = x + 10
   102    7    y    750   740      ← y = y - 10
   103    7    commit               ← transaction done
```

LSN = Log Sequence Number, monotonically increasing.

### 3.3 Commit protocol

```
   1. Append log records to WAL in RAM.
   2. fsync WAL to disk.
   3. Release locks on x and y.
   4. Reply to client: "committed".

   (Modified data pages stay in RAM — flushed later, lazily.)
```

### 3.4 Why delay flushing data pages?

```
   Reasons to delay:
     • Pages are big (8 KB). Multiple transactions may touch the same page.
     • "Write absorption" — combine many updates into one disk write.
     • The log is enough to recover; pages can be reconstructed.
```

### 3.5 Crash recovery

```
   Replay log:
     For each committed transaction in the log:
         apply "new" values         (REDO)
     For each uncommitted transaction in the log:
         apply "old" values         (UNDO)
```

> **Why the WAL matters so much**: it's the source of truth. Data pages are a cache. If the log is durable, the database can survive any crash.

---

## 4. Mirrored MySQL — Why It Hurts

Mirrored MySQL writes EVERY modified 8 KB page to four EBS replicas. Even if a transaction only changes 4 bytes:

```
   Update 4 bytes:
       MySQL flushes 8 KB page  ▶  8 KB × 4 EBS replicas = 32 KB on the wire

   And the WAL also gets mirrored — same expansion.
```

**Two bottlenecks:**

1. **Network bytes** — pages are huge relative to actual change.
2. **Synchronous quad write** — must wait for *all four* EBS replicas.

> **The Aurora insight**: the WAL already contains a tiny description of every change. **Why ship pages at all?** Send the log to storage and have storage apply it.

---

## 5. Aurora's Two Big Ideas

```
   ┌──────────────────────────────────────────────────────────┐
   │ ① QUORUM WRITES                                          │
   │   N=6, W=4, R=3 — across 3 availability zones            │
   │   Tolerate a slow/dead AZ without waiting on it          │
   ├──────────────────────────────────────────────────────────┤
   │ ② SMART STORAGE                                          │
   │   Storage servers UNDERSTAND the DB's log format         │
   │   DB ships only log records (small)                      │
   │   Storage applies the log to data pages locally          │
   └──────────────────────────────────────────────────────────┘
```

### 5.1 The big picture (Figure 3)

```
   Clients (web servers)
         │
         ▼
   ┌──────────────────┐
   │   PRIMARY DB     │  (one R/W EC2 instance per customer)
   └────┬──────────┬──┘
        │          │  log records (small)
        │          ▼
        │     ┌──────────────────────────────────────────────┐
        │     │  6 storage servers across 3 AZs              │
        │     │  ┌────┐ ┌────┐  ┌────┐ ┌────┐  ┌────┐ ┌────┐ │
        │     │  │ S1 │ │ S2 │  │ S3 │ │ S4 │  │ S5 │ │ S6 │ │
        │     │  └────┘ └────┘  └────┘ └────┘  └────┘ └────┘ │
        │     │   AZ-1           AZ-2           AZ-3          │
        │     └──────────────────────────────────────────────┘
        │
        ▼ (log fan-out to read replicas)
   ┌──────────────────┐
   │  READ REPLICAS   │  serve read-only client requests
   └──────────────────┘
```

### 5.2 Why this gets 35× throughput

| Mirrored MySQL | Aurora |
|---|---|
| Ships full 8 KB pages | Ships log records (tens of bytes) |
| Waits for 4 same-AZ EBS replicas | Waits for 4 of 6 cross-AZ replicas (quorum) |
| Both replicas in one AZ | 2 replicas in each of 3 AZs |
| Network is the bottleneck | Network shrunk ~100×; bottleneck shifts to CPU/storage |

---

## 6. Quorum Reads & Writes — A Primer

Idea from Gifford (SOSP 1979), used widely.

### 6.1 The rule

```
   N = total replicas
   W = nodes a write must reach
   R = nodes a read must reach
   Requirement:  R + W > N    (the read and write sets must overlap)
```

Simplest case: N=3, W=2, R=2.

```
   Write quorum {A, B}          Read quorum {B, C}
                  ▲                  ▲
                  └── overlap at B ──┘
   → the read sees at least one node that has the write
```

### 6.2 How does the reader know which copy is newest?

> You **can't vote on content** — only one server in the read quorum might be up to date.

The writer attaches a **version number** to each write. Storage servers remember versions. The reader takes the **max version** among R replies.

### 6.3 What if you can't assemble a quorum?

```
   No quorum reachable → keep retrying.
   Eventually the partition heals or the config manager intervenes.
```

### 6.4 Why this is great vs. chain replication

| Property | Chain Replication | Quorum |
|---|---|---|
| Tolerates slow node | ✗ Whole chain slows | ✓ Skip the slow one |
| Tolerates failed node | ✗ Chain stops | ✓ Pick a different quorum |
| Need failure detector? | Yes | No (just go ahead with whoever responds) |
| Split brain among replicas | Possible | Impossible (overlap rule) |

### 6.5 Aurora's choice: N=6, W=4, R=3

Why not N=3, W=2, R=2 across 3 AZs?

> If one AZ goes down → only 2 replicas → "AZ+1" is violated. You couldn't afford **one more failure** without losing data.

With N=6 (two per AZ), W=4, R=3:

```
   Goal: "AZ + 1"
     • Tolerate ONE AZ outage → 4 replicas left → write quorum 4 still met ✓
     • Tolerate AZ outage + 1 more failure → 3 replicas left → read quorum 3 met ✓
       (can read & rebuild, but can't write)
```

> **Analogy — voting with overlap**
> Imagine 6 witnesses to a contract, spread across 3 cities. Any 4 of them must co-sign for the contract to take effect. Any 3 must agree on which version of the contract is current. Because 4 + 3 > 6, **the reading committee always shares at least one signer with the writing committee** — so the truth can never get lost, even if a whole city is offline.

---

## 7. Aurora's Storage Protocol

### 7.1 Writes are log records, not page updates

```
   DB writes never modify existing data items at storage servers.
   A "write" is a new log entry:
       • for an in-progress transaction, OR
       • for a commit record.

   DB sends each log entry to ALL SIX storage servers.
   Commit waits for a 4-of-6 write quorum on the transaction's last record.
```

When the quorum acks, the DB:

```
   1. Releases locks.
   2. Replies "committed" to the client.
```

### 7.2 What storage servers do

```
   On receiving a log entry:
       Append it to a per-page list of pending log entries.

   Background work (or on-demand during a read):
       Apply pending log entries to the local data page.
```

So each storage server stores:

```
   ┌────────────────────────────────────────┐
   │ Data pages (B-tree blocks)             │  ← possibly stale
   │ + per-page list of pending log records │  ← brings page up to date
   └────────────────────────────────────────┘
```

> **Analogy — restaurant kitchens with handwritten amendments**
> The recipe binders (data pages) are at every storage server. The head chef (DB) doesn't mail new binders — she sends little **amendment slips** ("on page 47, change salt → soy"). Cooks (storage servers) staple slips to the binder and apply them when the page is next read or in idle time.

### 7.3 Reads are NOT quorum reads

```
   DB reads happen on cache miss for a data page.
   DB tracks each storage server's SCL (Segment Complete LSN).
       → "this server has all log entries ≤ SCL"
   DB picks ONE storage server whose SCL ≥ latest committed LSN.
   Asks it for the page (which it brings up to date by applying pending logs).
```

Why not quorum reads? They're **expensive** (multiple round trips, lots of bytes). Because the DB knows which servers are caught up via SCL, it can pick one.

### 7.4 When *are* quorum reads used?

> **Only during crash recovery** — to find the VCL. See next section.

---

## 8. Crash Recovery — VCL, VDL & Friends

### 8.1 The recovery problem

The DB server crashed. A fresh DB server starts on a new EC2 instance and attaches to the **same six storage servers**. What state is the log in?

```
   Storage nodes' log entries:
       SN1:  101 102 103 104 105 106 -   -
       SN2:  101 102 103 104 105 -   -   -
       SN3:  101 102 103 104 -   -   -   -
       SN4:  101 102 103 104 105 106 107 -
       SN5:  101 102 103 104 105 -   -   -
       SN6:  101 102 103 104 105 106 -   -
                                ▲
                                VCL = 104
```

### 8.2 VCL — Volume Complete LSN

> The highest LSN such that **every** log entry up through it is present on a reachable storage server.

Found by **quorum-reading** the log to identify gaps.

### 8.3 What happens after finding VCL

```
   1. Tell all storage servers: delete every log entry above VCL.
   2. For transactions that committed before VCL: nothing to do — already durable.
   3. For transactions that STARTED but didn't COMMIT before the crash:
        issue undo log records to roll them back.
```

**Why this is safe**: no transaction commits unless its log records are on a **write quorum (4 of 6)**. By the overlap rule, the recovery's read quorum (3) will see them. So no committed transaction is above VCL.

### 8.4 VDL — Volume Durable LSN

B-tree updates may span multiple log records (page splits, rebalancing). Mid-sequence, the B-tree is **inconsistent**.

- **CPL** (Consistency Point LSN) = a log point at which the data structure is internally consistent.
- **VDL** = the highest CPL that's ≤ VCL.

> **Readers may only look at the database state as of VDL.** Anything above VDL might catch the B-tree mid-update.

```
   Log:  [op] [op] [op] [CPL] [op] [op] [CPL] [op]
                          ▲                     ▲
                          old                   newest CPL ≤ VCL
                          consistent              = VDL
                          point
```

### 8.5 Crash recovery — who decides the DB server is dead?

> Paper is silent. There must be **some external authority** (probably Amazon's control plane) that detects DB failure, decides to start a replacement, and prevents split brain. Aurora doesn't explain how.

---

## 9. Scaling Beyond One Storage Server

### 9.1 Protection Groups (PGs)

A volume bigger than one storage server is **sharded** into 10 GB chunks called **Protection Groups**. Each PG is independently replicated to 6 storage servers ("segments").

```
   Database volume   = lots of PGs
   PG 1              = stored on  storage servers {S1, S5, S9, S13, S17, S21}
   PG 2              = stored on  storage servers {S2, S6, S10, S14, S18, S22}
   PG 3              = stored on  storage servers {S3, S7, S11, S15, S19, S23}
   ...
```

The DB server tracks which storage servers hold each PG and only ships a log record to the PGs it touches.

### 9.2 Fast re-replication after a death

Suppose one storage server permanently dies, holding 1000 PGs (~10 TB).

```
   Naive: copy 10 TB from one source to one destination at 1 GB/s
          → 10,000 seconds (~3 hours)

   Aurora's trick:
       Each PG's OTHER replicas are on different storage servers.
       Each PG's NEW replica goes on yet another storage server.
       → 1000 PGs can re-replicate IN PARALLEL.
       Per-PG copy takes ~10 seconds.
       → Whole job: ~10 seconds.
```

> **Lesson**: many small parallel jobs beat one big sequential job. The shorter the rebuild window, the less likely a second failure catches you exposed.

> **Analogy — distributed book reprinting**
> A library that lost a branch. Each book was already in 5 other branches scattered across the city. Instead of one truck driving 10 hours to ferry a million books to a new building, every existing branch reprints a handful in parallel, drops them at 1000 different new locations — done in minutes.

---

## 10. Read-Only Replicas

### 10.1 Why have them?

The single primary DB is the write-throughput ceiling. **Read-only replicas** offload read traffic.

```
   Clients (read-heavy):
       SELECTs ─▶ read replicas (eventual consistency)
       UPDATEs ─▶ primary (strongly consistent)
```

### 10.2 How they stay current

```
   Primary streams log records to read replicas (same log it sends to storage).
   Replicas apply log records to their cached pages.
   Storage servers serve as fallback for cache misses.
```

### 10.3 Two pitfalls and their fixes

**Pitfall 1**: Replicas don't know what's locked or uncommitted.

> **Fix**: replicas **ignore or undo** writes from transactions that the log doesn't show as committed.

**Pitfall 2**: Replicas might see a B-tree mid-update — garbage.

> **Fix**: only apply log up to a **CPL** boundary. Replicas always read a consistent snapshot.

### 10.4 Staleness is the trade

```
   Read replicas may not show writes that just committed at the primary.
   Clients tolerate this for big throughput gains.
   (For strong reads, go to the primary.)
```

---

## 11. Takeaway

> **Aurora's wins come from breaking the storage abstraction.** By teaching the storage layer the database's log format, Aurora ships ~100× less data over the network than mirrored MySQL. Quorums (N=6, W=4, R=3 across 3 AZs) absorb slow or dead replicas without waiting and without split brain. The result is a 35× throughput improvement, plus AZ-level fault tolerance, at the cost of a database-specific (non-reusable) storage system.

### Mental checklist

| Concept | One-liner |
|---|---|
| **Mirrored MySQL** | Ships full pages × 4 EBS replicas — slow, page-heavy |
| **Log-only writes** | Aurora sends tiny log records, not 8 KB pages |
| **Smart storage** | Storage applies log records to its own data pages |
| **N=6, W=4, R=3** | Quorum across 3 AZs; AZ+1 fault tolerance |
| **No quorum reads (normal)** | DB tracks SCL, picks one caught-up storage server |
| **Quorum reads (recovery)** | Used only to find VCL after a crash |
| **VCL** | Highest LSN with no log holes — recovery truncates above it |
| **VDL** | Highest CPL ≤ VCL — last point where DB structures are consistent |
| **Protection Group** | 10 GB shard; volumes are made of many PGs |
| **Parallel re-replication** | Each PG independently → 10 TB in ~10 seconds |
| **Read replicas** | Apply log to cached pages, see consistent snapshots up to VDL |
| **Single writer** | Simplifies everything; also the scaling ceiling |
| **What's not explained** | How the DB server itself is fenced/replaced to avoid split brain |

### The recurring lesson

> **When the network is the bottleneck, move the computation to the data.** MapReduce did it for batch jobs. GFS did it for chunk reads. Aurora does it by teaching storage servers about the database log. The same idea will appear again in Spark and in disaggregated storage architectures.
