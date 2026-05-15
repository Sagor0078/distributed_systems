# Lecture 10 — Amazon Aurora: FAQ

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Companion Q&A to [10_Aurora.md](10_Aurora.md)
> Paper: *"Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases"* — Verbitski et al., SIGMOD 2017

---

## Table of Contents

1. [Why Aurora Beats Mirrored MySQL](#1-why-aurora-beats-mirrored-mysql)
2. [Crash Recovery: VCL and VDL](#2-crash-recovery-vcl-and-vdl)
3. [Aurora vs. Spanner](#3-aurora-vs-spanner)
4. [Why Traditional Databases Do So Much I/O](#4-why-traditional-databases-do-so-much-io)
5. [The "Redo Offload" Idea](#5-the-redo-offload-idea)
6. [Split-Brain & The Single Writer](#6-split-brain--the-single-writer)
7. [Cheat Sheet](#7-cheat-sheet)

---

## 1. Why Aurora Beats Mirrored MySQL

### Q: Why is Aurora faster than Mirrored MySQL (Table 1)?

**A.** **Network bytes.** Mirrored MySQL ships full 8 KB pages across the network; Aurora ships tiny log records.

```
   Mirrored MySQL — write 4 bytes:
   ─────────────────────────────────
   DB Server  ──▶  EBS  ──▶  EBS  ──▶  EBS  ──▶  EBS
                  8 KB     8 KB     8 KB     8 KB    = 32 KB on the wire
   (each modified page is a full block, replicated to 4 EBS volumes)

   Aurora — write 4 bytes:
   ─────────────────────────────────
   DB Server  ──▶  Storage  ──▶ ... 6 storage nodes
                  ~tens of bytes (log record) × 6  = a few hundred bytes
```

Mirrored MySQL is **page-shipping**. Aurora is **log-shipping**. Same logical work, ~100× less network traffic.

### 1.1 The trade-off

```
   Aurora's price for the speedup:
       Storage servers must UNDERSTAND log records
       Storage servers must APPLY them to data pages
       → The storage layer is now database-specific.
       → Aurora storage isn't reusable for anything else.
```

A general-purpose block store (EBS) is replaced by a **database-aware** replicated store.

> **Analogy** — Two ways to keep a friend's recipe binder in sync with yours.
> - **Mirrored MySQL**: photocopy and mail entire pages every time you tweak a measurement.
> - **Aurora**: mail a tiny sticky note — *"on page 47, change 1 tsp to 1 tbsp"* — and your friend applies it.
>
> The sticky-note system is faster and cheaper, **but** only works if your friend speaks the same recipe-binder language. A photocopier works on anything; the sticky-note system is bespoke.

---

## 2. Crash Recovery: VCL and VDL

When the DB server crashes, the log may have **holes** near the tail — entries that the old server hadn't yet replicated to enough storage nodes.

```
   Log entries on storage nodes:

      LSN:   100  101  102  103  104  105  106  107
      SN1:   ✓    ✓    ✓    ✓    ✓    ✓    -    -
      SN2:   ✓    ✓    ✓    ✓    ✓    -    -    -
      SN3:   ✓    ✓    ✓    ✓    -    -    -    -
      SN4:   ✓    ✓    ✓    ✓    ✓    ✓    ✓    -
      SN5:   ✓    ✓    ✓    ✓    ✓    -    -    -
      SN6:   ✓    ✓    ✓    ✓    ✓    ✓    -    -
                              ▲
                              VCL = 104
                              (last LSN with NO holes before it)
```

### 2.1 VCL — Volume Complete LSN

> **The highest LSN such that every log entry up to that point is present somewhere on a quorum.**

A recovering DB server scans the log (with quorum reads), finds VCL, and tells storage servers: *"erase anything after VCL."*

Why is this safe? Because **no transaction commits until all its log entries are in a write quorum.** So any committed transaction is entirely before VCL. The discarded suffix only contained in-flight (uncommitted) work.

### 2.2 What about uncommitted transactions before VCL?

Some transactions started but never committed before the crash. Their log entries are below VCL. Recovery sends **undo log records** to roll those back.

### 2.3 VDL — Volume Durable LSN

> The highest **CPL** (Consistency Point LSN) that is ≤ VCL.

Why does this matter? Database B-trees take **multiple log entries** to update consistently (rebalance, split, etc.). Reading the data structure mid-update is garbage.

```
   Log:   ... [split node]  [split node]  [insert]  [CPL]  [delete]
                                                       ▲
                                              safe to read here
                                              (B-tree is consistent)
```

- **CPL** = a log point where the B-tree is internally consistent.
- **VDL** = the latest such consistent point that's also fully present (≤ VCL).

### 2.4 Who uses VDL?

**Read-only replica instances** (Figure 3, §4.2.4). They serve read traffic by applying log entries to local cached pages, but only up to a CPL — otherwise they'd see half-updated B-trees.

> **Analogy** — A construction site.
> - **VCL** is "the latest stage of the building plan we've fully received".
> - **VDL** is "the latest stage where the building is structurally sound" — beams installed, joints bolted, no half-poured concrete. You can tour the site only up to VDL.

---

## 3. Aurora vs. Spanner

### Q: How do they compare?

**A.** They share goals; they diverge on scalability and writer model.

| Aspect | Aurora | Spanner |
|---|---|---|
| **Cross-datacenter replication** | ✓ (6 replicas across 3 AZs) | ✓ (Paxos groups across geo) |
| **Quorum writes** | ✓ (4 of 6) | ✓ (Paxos majority) |
| **High read throughput** | ✓ (read-only replicas via VDL) | ✓ (time-based snapshot reads) |
| **Number of writers** | **One** R/W DB server | **Many** (one per shard) |
| **Scaling writes** | Limited to one machine | Sharded — scales horizontally |
| **Cross-row/cross-shard txns** | Single server handles trivially | 2PC + Paxos |
| **Split-brain on writer failure** | Paper doesn't say | Paxos handles it cleanly |

> **The big architectural difference**: Aurora has a single writer. That's a huge simplification — only one source of LSNs, no distributed transactions, no two-phase commit. But it's a hard ceiling on write throughput.

### 3.1 Where each wins

```
   Workload that fits on one big machine:        Aurora wins  (simpler, fast)
   Workload that exceeds one machine's writes:   Spanner wins (scales horizontally)
   Multi-region writes with strong consistency:  Spanner wins (TrueTime + Paxos)
   Migration from a MySQL/Postgres app:          Aurora wins  (drop-in compatible)
```

> **Analogy** — A single-pilot jumbo jet vs. a fleet of regional jets. The jumbo (Aurora) carries massive payload to one destination cheaply, with a simple chain of command. The fleet (Spanner) carries the same total payload across many destinations, but needs air traffic control and coordinated schedules.

---

## 4. Why Traditional Databases Do So Much I/O

### Q: Why does MySQL generate so many disk writes per transaction?

**A.** Three layers piling on:

```
   ┌────────────────────────────────────┐
   │  Database (MySQL, Postgres, ...)   │
   │   • Append to redo log              │
   │   • Update B-tree data pages        │
   │   • Update index pages              │
   ├────────────────────────────────────┤
   │  Filesystem                        │
   │   • Update inodes                   │
   │   • Update block free bitmap        │
   │   • Append to filesystem journal    │
   ├────────────────────────────────────┤
   │  Disk                              │
   └────────────────────────────────────┘
```

Even a one-row UPDATE may touch a dozen disk blocks across these layers.

### Q: Are writes flushed immediately or buffered?

**A.** Split decision:

| Write type | When it hits disk |
|---|---|
| **Log entries** (WAL) | **Before** the client gets a commit reply (must be durable). Group commit batches concurrent transactions to amortize fsync. |
| **Data pages** (B-tree updates) | Lazily — kept in the buffer cache. Multiple transactions touching the same page combine into one disk write. |

The log is the source of truth; data pages can be reconstructed from it during recovery.

---

## 5. The "Redo Offload" Idea

### Q: What does §3.2 mean by "offloading redo"?

**A.** In a traditional DB:

```
   1. Read 8 KB page from storage.
   2. Modify a few bytes in RAM.
   3. Write 8 KB page back to storage.
   4. Append log entry to log on storage.
```

In Aurora:

```
   1. Send a tiny log record to storage.
   2. Storage server applies the log record to its local copy of the page.
   3. ... that's it.
```

The **redo application work** — taking a log entry and updating the page — has moved from the DB server to the storage server.

```
   Traditional:                Aurora:
   ────────────                ──────
   Storage = "dumb bytes"      Storage = "applies redo to data pages"

   DB:  reads page             DB:  ships log record only
        modifies in RAM        Storage:  applies log → page
        writes back full page             reads when DB asks
```

> **Why this is a win**: log records are ~tens of bytes; pages are 8 KB. Network is the bottleneck. Less data on the wire = faster commits.

### 5.1 The architectural shift

```
   Layer                     Traditional        Aurora
   ─────────────────────────────────────────────────────────
   Where is the "log"?       Local + EBS        On storage nodes
   Where is "redo applied"?  DB server          Storage nodes
   What does storage know?   Bytes              The database format
   Reusable storage?         Yes (e.g. EBS)     No (Aurora-only)
```

> **Big idea**: Move computation **to** the data instead of moving data to the computation. The same idea drives MapReduce locality and GFS chunkserver-local reads.

---

## 6. Split-Brain & The Single Writer

### Q: How does Aurora handle a 50/50 partition?

**A.** Because Aurora has **only one writer**, only one side of the partition matters:

```
   Network partition cuts the cluster in half:

       Side A: [DB server]  +  3 storage nodes
       Side B:                 3 storage nodes

   Side A needs 4-of-6 write quorum → can't write    (only has 3 storage)
   Side A still has 3-of-6 → can read                (read quorum = 3)
   Side B has no DB server → can do nothing
```

Result: **no two competing writers**, so no split brain. Aurora just becomes read-only until the partition heals or the configuration changes.

### Q: What if the DB server is wrongly believed dead and replaced?

**A.** *That's* the real split-brain risk. If the failure detector promotes a new DB server while the old one is still alive, both might issue conflicting writes.

> The paper **does not explain** how Aurora prevents this. Likely Amazon uses some external fencing mechanism (leases, generation numbers in storage requests) but it's not in the published paper.

### 6.1 Why Spanner is cleaner here

Spanner uses Paxos for **every** decision, including which server is the leader. So the leader-replacement protocol is the same fault-tolerant protocol as everything else, and split brain is structurally impossible.

> **Analogy — single-driver delivery van**:
> - Aurora is one driver in a van, with six warehouses holding the goods.
> - If the road splits the country in half, only the side with the driver can dispatch deliveries — no risk of two drivers contradicting each other.
> - But if HQ wrongly declares the driver dead and hires a replacement while the original is still driving, **now** you have two drivers handing out conflicting deliveries.
>
> The single-driver invariant makes most failures easy; the failure-of-the-driver case is where Aurora's design hides its hardest problem.

---

## 7. Cheat Sheet

| Concept | One-liner |
|---|---|
| **Page-shipping vs. log-shipping** | Aurora sends log records, not pages — ~100× less network |
| **Smart storage** | Storage servers apply redo log records; not reusable outside Aurora |
| **VCL** | Highest LSN with no holes — recovery truncates anything after |
| **CPL / VDL** | Consistency points where B-trees are internally consistent; what readers see |
| **Quorum** | 6 storage nodes, write quorum 4, read quorum 3 (across 3 AZs) |
| **Single writer** | One R/W DB server — simple, but a write-throughput ceiling |
| **Read-only replicas** | Apply redo locally up to VDL; scale read throughput |
| **Spanner contrast** | Spanner shards writes; Aurora can't |
| **Split-brain risk** | Comes from DB-server-replacement, not the storage layer; paper is silent |
| **Architectural lesson** | Push computation into the storage tier when network is the bottleneck |
