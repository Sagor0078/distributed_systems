# GFS — The Google File System

> **Paper:** Ghemawat, Gobioff, Leung — *SOSP 2003*
> **Course:** MIT 6.824 (2020), Lecture 3

---

## TL;DR

GFS is a **distributed file system** for huge files, sequential reads, and concurrent appends — the workload of MapReduce, crawlers, and log pipelines. It chooses **weak consistency + a single master** to win on **scale and simplicity**, accepting some application-visible anomalies in return.

| Property        | Choice                              |
| --------------- | ----------------------------------- |
| Workload        | Huge files, sequential, append-heavy |
| Sharding unit   | 64 MB chunks                        |
| Replication     | 3 copies per chunk                  |
| Coordinator     | Single master (state in RAM)        |
| Consistency     | Weak — record append is *at-least-once* |
| Scope           | One datacenter, internal apps only  |

---

## Why this paper matters

Distributed storage is **the** key abstraction in 6.824. GFS is a real-world, deployed system that touches every theme of the course at once:

- **Parallel performance** — sharding across many disks.
- **Fault tolerance** — replication.
- **Consistency** — the cost of replication.
- **Trade-off design** — none of these are free.

What was *new* in 2003 wasn't the ideas (sharding, replication) — it was the **scale**, the **industrial deployment**, and two bold choices: **weak consistency** and a **single master**.

---

## The Core Tension

```
  high performance  ──►  shard data over many servers
        │
        ▼
  many servers      ──►  constant failures
        │
        ▼
  fault tolerance   ──►  replication
        │
        ▼
  replication       ──►  inconsistency risk
        │
        ▼
  strong consistency ──► slower (more coordination)
```

Every distributed storage system is a point chosen on this chain. GFS picks **far performance / weak consistency**.

---

## Consistency: the ideal vs. the cheap

### Ideal: behave like one server

```
Time ────────────────────────────────────►
C1:   Wx=1
C2:        Wx=2
C3:                Rx=?
C4:                       Rx=?

Allowed answers: BOTH read 1, or BOTH read 2.
NOT allowed:     C3 sees 1, C4 sees 2.
```

A single server serializes everything → reads always reflect prior writes. **Strong consistency, terrible fault tolerance.**

### Naive replication breaks this

> **Analogy.** Two cashiers (S1, S2) each keep their own copy of the ledger. Customers shout updates to both. If two customers shout at once, S1 may write them in one order and S2 in the other — now the ledgers disagree. Worse, a customer might shout to S1, then have a heart attack before reaching S2 — S2 never hears the update at all.

That's why naive replication fails:
- Writes can arrive at replicas in different orders.
- A client crash mid-broadcast leaves replicas diverged.
- Fixing this requires **coordination**, which costs latency.

---

## GFS Context & Design Goals

- Used by **MapReduce, crawler/indexer, log analysis, YouTube (probably)**.
- One **global namespace per datacenter** — any client reads any file.
- **Automatic sharding** of each file across many disks.
- **Automatic recovery** from failures.
- Tuned for **sequential access to huge files** — *not* a low-latency small-record DB.
- Internal Google use only, one datacenter per deployment.

---

## Architecture at a Glance

```
                          ┌──────────────────────┐
                          │       MASTER         │
                          │  (single, in-RAM)    │
                          │                      │
                          │  file → chunk list   │
                          │  chunk → servers     │
                          │  chunk → version #   │
                          │  chunk → primary +   │
                          │           lease      │
                          └──────────┬───────────┘
                                     │ metadata only
                                     │ (no bulk data)
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
        ┌──────────┐           ┌──────────┐           ┌──────────┐
        │ Chunk    │           │ Chunk    │           │ Chunk    │
        │ Server 1 │           │ Server 2 │           │ Server 3 │
        │ ┌──────┐ │           │ ┌──────┐ │           │ ┌──────┐ │
        │ │chunk │ │           │ │chunk │ │           │ │chunk │ │
        │ │ A    │ │           │ │ A    │ │           │ │ A    │ │
        │ │chunk │ │           │ │ B    │ │           │ │ C    │ │
        │ │ B    │ │           │ │chunk │ │           │ │chunk │ │
        │ └──────┘ │           │ │ C    │ │           │ │ B    │ │
        │ 64 MB    │           │ └──────┘ │           │ └──────┘ │
        │ chunks   │           │          │           │          │
        └────▲─────┘           └────▲─────┘           └────▲─────┘
             │                      │                      │
             │      bulk data       │                      │
             └──────────────────────┼──────────────────────┘
                                    │
                              ┌─────┴─────┐
                              │  CLIENT   │
                              │ (library) │
                              └───────────┘
```

> **Analogy: a library with a single librarian.**
> The **librarian (master)** knows which **shelves (chunkservers)** hold which **books (chunks)**, but never carries the books herself. Each book is photocopied **3 times** and kept on 3 different shelves so no fire (disk failure) destroys it. Readers ask the librarian *where* a book is, then walk to a shelf themselves to read it. The librarian is the bottleneck for *finding* things — but never for *reading* them.

### Why a single master?

- **Pro:** simple, has a global view, easy to coordinate placement.
- **Con:** RAM and CPU eventually saturate. (And they did — see Retrospective below.)

### Why 64 MB chunks?

- Fewer metadata entries → master state fits in RAM.
- Amortizes TCP setup / seek overhead.
- **Cost:** terrible for many small files.

---

## Master State

| Where          | What                                          | Volatile? |
| -------------- | --------------------------------------------- | --------- |
| **RAM**        | filename → chunk-handle list                  | **No** (logged) |
| **RAM**        | chunk-handle → version #                      | **No** (logged) |
| **RAM**        | chunk-handle → chunkserver list               | Yes (re-learned from heartbeats) |
| **RAM**        | chunk-handle → primary, lease expiry          | Yes (re-issued after lease window) |
| **Disk**       | operation log + periodic checkpoint           | persistent |

> **Why a log + checkpoint?** The log makes every metadata mutation durable before acknowledging. The checkpoint is a snapshot so replay on restart doesn't take forever.

> **Why is "list of chunkservers" volatile?** Because chunkservers *tell the master* what they have at startup, via heartbeats. The master doesn't need to remember — the truth lives on the chunkservers.

---

## Read Path

```
   CLIENT                       MASTER                     CHUNKSERVER
     │                            │                             │
     │  filename + offset         │                             │
     │ ──────────────────────────►│                             │
     │                            │ look up chunk handle        │
     │                            │ find servers w/ latest ver  │
     │  chunk handle + servers    │                             │
     │ ◄──────────────────────────│                             │
     │                            │                             │
     │  (cache handle + servers)  │                             │
     │                            │                             │
     │  chunk handle + offset                                   │
     │ ────────────────────────────────────────────────────────►│
     │                                                          │ read
     │                                                          │ chunk
     │                                                          │ file
     │  data                                                    │
     │ ◄────────────────────────────────────────────────────────│
```

**Key point:** the master is **off the data path**. It hands out a "treasure map" and steps aside.

---

## Record Append Path

The interesting case — this is where consistency gets thorny.

```
                                  MASTER
                                    │
   1. C asks: "last chunk of file?"  │
   ──────────────────────────────────►
                                    │  if no primary, or lease expired:
                                    │   • pick primary P from up-to-date replicas
                                    │   • bump version #, write to log+disk
                                    │   • tell P + secondaries: "you're it, new ver"
                                    │   • replicas persist new version #
   2. M replies: P + secondaries     │
   ◄──────────────────────────────────
                                    │
   3. C pushes data to ALL replicas  │     (just buffered, not yet appended)
       ┌──────►  P                    │
       ├──────►  S1                   │
       └──────►  S2                   │
                                    │
   4. C says to P: "commit append"   │
   ────────────────────────────►  P  │
                                       │ checks lease + chunk space
                                       │ picks an offset (end of chunk)
                                       │ appends to local chunk file
                                       │ tells S1, S2: "append at offset N"
                                       │ waits for all to ack (or timeout)
                                       │
   5. P replies: "ok" or "error"     │
   ◄───────────────────────────────   │
                                    │
   6. on error, C RETRIES from step 1
```

> **Analogy: a shared whiteboard with a captain.**
> Three people (P, S1, S2) each have an identical whiteboard. The captain (P) decides what spot the next note goes in and tells everyone else "write 'foo' at row 7." If one of them was looking away and missed it, the writer is told something went wrong — and **just tries again** (which is why the record may show up *twice*).

---

## The Actual Consistency Guarantee

> If the primary tells a client that a **record append succeeded**, then any reader who later scans the file **will see the appended record somewhere**.

### What this does **not** guarantee:

- That failed appends won't *also* appear (they often do — the client retried).
- That all readers see the same content.
- That records appear in the same order across readers.
- That bytes between records are meaningful (there are padding gaps).

> **Mental model.** GFS gives applications "at-least-once append." Apps must tolerate **duplicates** and **gaps**. MapReduce was designed around this — records have checksums and unique IDs, and the framework deduplicates downstream.

---

## Failure Scenarios — How GFS Holds the Line

| Failure                                                | What happens                                                                                   |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| Appending client crashes mid-append                    | Some replicas may have it, some not → readers may see the record twice (later retry) or once.  |
| Client cached a stale primary                          | Old primary's lease expired → rejects the request → client refetches from master.              |
| Client cached a stale secondary                        | Reader may miss recent appends — allowed under the guarantee.                                  |
| Master crash + reboot                                  | Recovers from log + checkpoint. Chunkserver list re-learned from heartbeats.                   |
| Two concurrent appends                                 | Primary serializes them at different offsets — neither overwrites.                             |
| Secondary missed an append, then serves a read         | Reader sees stale data — **anomaly allowed**.                                                  |
| Primary dies before notifying all secondaries          | Version # gating: master only promotes replicas with the latest version → no silent rollback.  |
| Stale chunkserver (S4) comes back after primary dies   | Version # is older → master rejects it. Better "no answer" than "wrong answer."                |
| Network partition isolates primary (split brain)       | **Lease expiry** prevents two simultaneous primaries — old primary stops accepting writes when its lease runs out, *before* master grants a new one. |
| Whole-building power loss                              | Log + checkpoint on disk → master state recoverable. Chunkservers re-register on boot.         |
| All master replicas permanently lose disk              | **Guarantee broken** — but failure mode is "no answer" (fail-stop), not "wrong answer."        |
| Clock skew breaks lease math                           | **Guarantee broken** — possible split brain.                                                   |

### The lease trick (split-brain prevention)

```
Master grants lease  ──────────────────────────────► Primary
                     │   "you are primary for 60s"   │
                     │                                │
                     │  ── partition ──               │
                     │                                │
                     │   t < 60s: old P serves writes (correct, still has lease)
                     │   t = 60s: old P STOPS  ──────►│  (refuses appends)
                     │                                │
                     │   master waits ≥ lease window  │
                     │   then safely picks new P      │
```

**Invariant:** the master *never* grants a new primary lease until the old one has demonstrably expired. So there is never a window with two active primaries — assuming clocks are reasonable.

---

## What Strict Consistency Would Cost

GFS *chose* anomalies. Closing them would require:

- **Dedup on the primary** (or never retry on the client).
- **All-or-nothing writes** across replicas — i.e. a two-phase trick where writes are tentative until everyone promises to apply them.
- **New-primary sync**: when promoting a replica, *talk to all replicas* to recover the last few ops before serving traffic.
- **Read leases on secondaries** — or force all reads through the primary — to keep stale ex-secondaries from serving reads.

> You'll build exactly this in Labs 2 and 3 (Raft + replicated KV).

---

## Performance (Paper, Figure 3)

| Scenario                            | Numbers                                  | Note                                                       |
| ----------------------------------- | ---------------------------------------- | ---------------------------------------------------------- |
| Aggregate read, 16 chunkservers     | 94 MB/s total (~6 MB/s per server)       | Per-server is meh, but **scales** linearly with cluster.   |
| Production aggregate read           | ~500 MB/s                                | Real win is *aggregate*, not per-node.                     |
| Writes to different files           | Below ceiling — authors blame net stack. |                                                            |
| Concurrent appends to *one* file    | Bottlenecked by the chunkserver holding the file's last chunk. | Hot-spot risk for popular files. |

**Lesson:** for batch/parallel workloads, *aggregate throughput* matters more than single-stream latency. GFS optimized for the former.

---

## Open Questions / Future Reading

- **Small files?** Master RAM cost per file is prohibitive — BigTable answers this.
- **Billions of files?** Same problem — Colossus shards master metadata across many masters.
- **Wide-area GFS?** All replicas in one DC = no disaster tolerance. Cross-DC replication needs different consistency.
- **Recovery time?** Master fail-over was initially manual, **tens of minutes**.
- **Slow chunkservers?** Not really addressed — straggler handling came later.

---

## Retrospective ([ACM Queue interview](http://queue.acm.org/detail.cfm?id=1594206))

- **File count blew up 1000× past Table 2** — master RAM became the bottleneck.
- Thousands of clients → master CPU bottleneck.
- Applications had to be *designed around* GFS's loose semantics — significant burden.
- **Manual master failover** took tens of minutes — too slow.
- → **BigTable** handles small structured records.
- → **Colossus** shards the master itself.

---

## Summary

### Good ideas

- Global cluster file system as **shared infrastructure**.
- Clean **separation of metadata (master) and data (chunkservers)**.
- **Sharding** for aggregate throughput.
- **Huge chunks** to amortize per-op overhead.
- **Primary-orchestrated writes** for ordering.
- **Leases** to prevent split-brain primaries.

### Not so good

- Single master ran out of **RAM and CPU**.
- **Small-file** workloads are punished.
- No automatic master failover (initially).
- Consistency may have been **too relaxed** — app teams paid the cost.

---

> **One-line takeaway:** GFS works because Google's *applications* were willing to be designed around its weak guarantees — making the *system* simple was a deliberate trade against making the *application code* more complex.
