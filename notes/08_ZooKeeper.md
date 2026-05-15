# Lecture 8 — ZooKeeper: A General-Purpose Coordination Service

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Reading: *"ZooKeeper: wait-free coordination for internet-scale systems"* — Hunt, Konar, Junqueira, Reed (USENIX ATC 2010)
> Companion FAQ: [08_ZooKeeper_QA.md](08_ZooKeeper_QA.md)

---

## Table of Contents

1. [The Two Questions ZooKeeper Asks](#1-the-two-questions-zookeeper-asks)
2. [Question 1: Can N Replicas Give Nx Performance?](#2-question-1-can-n-replicas-give-nx-performance)
3. [ZooKeeper's Consistency Trade-off](#3-zookeepers-consistency-trade-off)
4. [Why Loose Reads Are Still Useful](#4-why-loose-reads-are-still-useful)
5. [Performance Tricks](#5-performance-tricks)
6. [Question 2: Coordination as a Service](#6-question-2-coordination-as-a-service)
7. [The ZooKeeper API](#7-the-zookeeper-api)
8. [Building Blocks: Mini-Transactions, Locks, Election](#8-building-blocks-mini-transactions-locks-election)
9. [Verdict](#9-verdict)
10. [Takeaway](#10-takeaway)

---

## 1. The Two Questions ZooKeeper Asks

The paper attacks two distinct problems with one design:

| # | Question | What the paper proposes |
|---|---|---|
| **1** | We paid for N replica servers. **Can we get Nx performance** from them? | A relaxed consistency model that lets reads be served by any replica |
| **2** | Can coordination be a **standalone, general-purpose service**? | A small filesystem-shaped API expressive enough to build locks, queues, leader election, etc. |

---

## 2. Question 1: Can N Replicas Give Nx Performance?

### 2.1 The setup — think Raft for now

```
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │ Client  │   │ Client  │   │ Client  │
   └────┬────┘   └────┬────┘   └────┬────┘
        └────────────┬┴───────────────┘
                     ▼
              ┌──────────────┐
              │    LEADER    │  state + log
              └──────┬───────┘
            ┌───────┴────────┐
            ▼                ▼
       ┌─────────┐      ┌─────────┐
       │FOLLOWER │ ...  │FOLLOWER │   state + log
       └─────────┘      └─────────┘
```

### 2.2 Writes get *slower* as N grows

The leader must wait for **majority acks** from a growing pool. More followers ≠ more write throughput.

### 2.3 The tempting idea: serve reads from replicas

If clients could read from **any** replica, read capacity would scale **O(N)** with cluster size.

### 2.4 But replica reads break linearizability

Three ways a replica's view can be stale:

```
   ① Replica wasn't in the leader's majority for write W  → doesn't have W
   ② Replica has W but hasn't heard "W is committed" yet
   ③ Replica is cut off from leader entirely
```

And there's a second hazard — **time travel**:

```
   Client reads from up-to-date Replica A    → sees x = 5
   Client switches to lagging Replica B      → sees x = 3   ← went backwards!
```

> Linearizability forbids both stale and time-traveling reads.

### 2.5 Raft's answer: route reads through the leader

Lab 3 does this. Result: **linearizable** ✓ but **no read parallelism** ✗.

### 2.6 ZooKeeper's answer: weaken the contract

```
   Lab 3 / strict Raft:           ZooKeeper:
   ─────────────────────          ──────────────────────
   Writes:   linearizable          Writes:   linearizable
   Reads:    linearizable          Reads:    FIFO client order
                                             (may be stale, but never
                                              go backwards for one client)

   Read throughput: O(1)           Read throughput: O(N)
```

> **Analogy** — A live sports broadcast. The TV feed (leader) has the freshest score. Radio relays (replicas) may be a few seconds behind. Different radio stations may show different scores at the same instant, but **for any one listener** the score never goes backwards in time. Many more listeners can be served than if everyone had to call the broadcast booth directly.

---

## 3. ZooKeeper's Consistency Trade-off

### 3.1 The two ordering guarantees (Section 2.3)

**(A) Linearizable writes.**

```
   Clients send writes ─▶ leader assigns each write a global ID (zxid)
                          ─▶ replicas execute in zxid order
                          ─▶ identical to Raft / Lab 3
```

**(B) FIFO client order.** For **each client**:

- The client's **writes** appear in the global write order **in the order the client issued them**.
- The client's **reads** each happen at some point in the global write order, and **successive reads from the same client never go backwards** in that order.
- A read happens **after** all of the same client's prior writes (the server may wait).

### 3.2 Implementation consequences

```
   Leader must:    preserve client write order across leader failover.
   Replicas must:  refuse to serve a read older than this client's latest read.
   Client must:    remember the highest zxid it has seen.
                   (Sends that zxid with every read so any replica can check.)
```

### 3.3 `sync()` — the freshness escape hatch

When a client truly needs the latest data:

```
   client.sync()      ← puts a marker in the leader's queue
   client.read(...)   ← waits until this replica has applied through the marker
                        → guaranteed to see all writes that completed before sync()
```

`sync()` makes that single read linearizable, at the cost of one round-trip.

---

## 4. Why Loose Reads Are Still Useful

> *"Why is it OK for reads to be stale?"*

The paper's answer is **read-order rules give you "read your own writes" and write-write ordering for free**, which covers most coordination patterns.

### 4.1 The "ready" znode pattern

A classic config-update protocol — a writer publishes many znodes, then flips a `ready` flag:

```
   Writer (one client):              Reader (any client, any time):
   ─────────────────────             ────────────────────────────────
   delete("ready")                   if exists("ready"):
   write f1                              read f1
   write f2                              read f2
   create("ready")
```

Because of FIFO order:

```
   Globally:    delete(ready)  ▶  write f1  ▶  write f2  ▶  create(ready)

   If a reader sees create(ready) at point X,
   then ALL writes f1, f2 are also visible at point X.
   The reader is guaranteed to read the new f1, f2 — never the old version.
```

This holds **even if the reader's previous reads were from a different replica.**

### 4.2 Watches preserve ordering too

```
   Reader:   exists("ready", watch=true)    ← arms a watch
             read f1                         ← reads old f1

   Writer:   delete("ready")
             write f1                        ← new f1
             write f2

   Reader:   ← gets watch notification (ready deleted)
             read f2                         ← will see new f2, because watch
                                               fired before this read
```

Watches deliver before subsequent reads see *later* writes — useful for cache invalidation.

### 4.3 Where loose reads do hurt

You can't do `x = read("counter"); write("counter", x+1)` safely — between the read and the write, anyone could change the counter. **Mini-transactions** (Section 8.1) fix this.

> **Analogy** — A whiteboard at a coffee shop showing "today's beans". The barista (writer) erases the board, lists new beans, then writes "READY". Customers (readers) only act on the board after seeing "READY". The chef's word *order* is what makes the system work, not strict moment-to-moment consistency.

---

## 5. Performance Tricks

ZooKeeper combines four techniques:

```
   ┌──────────────────────────────────────────────────────┐
   │ ① Replica reads        →  scales read throughput     │
   │ ② Async writes         →  many in-flight per client  │
   │ ③ Server-side batching →  amortize disk + network    │
   │ ④ Fuzzy snapshots      →  no write-blocking checkpts │
   └──────────────────────────────────────────────────────┘
```

### 5.1 Why batching needs async clients

Batching only works if the leader has many requests **at the same time**. If every client waits for a reply before sending the next request, the leader sees one at a time → nothing to batch. **Async writes** are what feed the batcher.

```
   Without async:    [w1] ─▶ ack ─▶ [w2] ─▶ ack ─▶ [w3]  ...
                     (one fsync per write)

   With async:       [w1][w2][w3][w4][w5] ─▶ ack all
                     (one fsync for the batch)
```

### 5.2 Result (Table 1 in paper)

| Servers | Read throughput | Write throughput |
|---|---|---|
| 3 | high | **21,000 writes/sec** |
| 5 | higher | slightly lower |
| 9 | highest | even lower |

Reads scale **up** with N, writes scale **down** — exactly the trade ZooKeeper is selling.

> 21,000 writes/sec is *much* faster than a naive per-disk-write throughput (10 ms = 100/sec). Batching is doing the heavy lifting.

---

## 6. Question 2: Coordination as a Service

### 6.1 What is "coordination"?

Bits of shared state that distributed systems use to agree on **who does what**:

| Use case | What's coordinated |
|---|---|
| **VMware FT** | Test-and-set lock to elect the sole primary |
| **GFS** | Which metadata replica is master? Who holds each chunk? |
| **MapReduce** | Master identity; worker registration; task assignment |
| **Web crawler** | Which URL is assigned to which worker |

Each of these built bespoke coordination logic. A **shared service** would save all that effort.

### 6.2 Could a Lab-3 k/v store do it?

Tempting:

```
   if Get("master") == "":
       Put("master", my_address)
       if Get("master") == my_address: act as master
```

But **Put/Get isn't atomic enough**:

```
   Client A: Get("master") → ""
   Client B: Get("master") → ""
   Client A: Put("master", "A")
   Client B: Put("master", "B")   ← overwrites!
   Both think they are master. SPLIT BRAIN.
```

Other gaps:

- No way to detect master death without polling.
- No way to be notified of master changes without polling.

ZooKeeper closes these with **atomic creates, version-checked writes, ephemeral nodes, and watches**.

---

## 7. The ZooKeeper API

### 7.1 The data model — a tree of "znodes"

```
   /
   ├── config/
   │   ├── shards     znode (data + version)
   │   └── ready      znode
   ├── locks/
   │   └── L1/
   │       ├── req-0001   (ephemeral, sequential)
   │       └── req-0002   (ephemeral, sequential)
   └── masters/
       └── primary    (ephemeral)
```

Looks like a filesystem, but **each znode holds bytes AND has a version counter**.

### 7.2 Three flavors of znode

| Type | Behavior |
|---|---|
| **Regular** | Persists until explicitly deleted |
| **Ephemeral** | Auto-deleted when the creating client's session ends (death, timeout) |
| **Sequential** | Name gets a monotonically increasing suffix `name-0001`, `name-0002`, ... |

### 7.3 Core operations (Section 2.2)

```
   create(path, data, flags)     exclusive — only the FIRST creator succeeds
   delete(path, version)         deletes only if znode.version == version
   exists(path, watch)           watch fires on later create/delete
   getData(path, watch)          watch fires on later setData
   setData(path, data, version)  CAS: writes only if znode.version == version
   getChildren(path, watch)      watch fires on child create/delete
   sync()                        flush this client's view through latest writes
```

### 7.4 Why this API is well-tuned to coordination

- **Exclusive `create`** → only one of N racing clients can succeed. Built-in mutex.
- **Versioned `setData` / `delete`** → atomic read-modify-write (mini-transactions).
- **Ephemeral nodes** → auto-cleanup when a client dies → locks self-release on crash.
- **Sequential nodes** → builds FIFO order across clients automatically.
- **Watches** → no polling. The server notifies the client when something changes.

> **Analogy** — A bulletin board at a community center, with rules:
> - One thumbtack per spot — only the **first** person to pin a note succeeds. (exclusive create)
> - When you leave the building, the staff automatically removes **your** notes. (ephemeral)
> - Notices are pinned in numbered order: #1, #2, #3 … (sequential)
> - You can ask the staff "tell me when this notice changes" instead of staring at the board. (watches)
> - To replace an existing note, you must hand in the staff a slip with the current "version number"; if someone else updated it first, your replacement is rejected. (version-checked setData)

---

## 8. Building Blocks: Mini-Transactions, Locks, Election

### 8.1 Mini-transaction: atomic increment

```python
while True:
    (x, v) = getData("/counter")
    if setData("/counter", x+1, version=v):
        break    # success
    # else: someone else updated; retry
```

The version check turns naive read-modify-write into an **atomic** operation. This is the same shape as a CPU's compare-and-swap, but on a distributed znode.

### 8.2 Simple lock (with herd effect)

```python
acquire(L):
    while True:
        if create("/locks/" + L, ephemeral=True):    # someone owns it now
            return                                   #   = me
        if exists("/locks/" + L, watch=True):
            wait_for_watch                           # blocks until released

release(L):
    delete("/locks/" + L)
    # OR: just close the session → ephemeral auto-deletes
```

**Problem**: when the lock is released, **all waiters wake up** and stampede to recreate it. With 100 contenders, 99 wake-ups are wasted. That's the **herd effect**.

### 8.3 Lock without herd effect (Section 2.4)

Each contender creates a **sequential, ephemeral** znode and only watches the **predecessor**:

```
   /locks/L/req-0001    ← lock holder (lowest number wins)
   /locks/L/req-0002    ← watches req-0001
   /locks/L/req-0003    ← watches req-0002
   /locks/L/req-0004    ← watches req-0003
```

```python
acquire(L):
    me = create("/locks/L/req-", ephemeral=True, sequential=True)
    while True:
        kids = sorted(children("/locks/L"))
        if kids[0] == me:
            return                                  # I have the lock
        prev = kid just before me
        if exists("/locks/L/" + prev, watch=True):
            wait_for_watch
        # loop: re-check, since prev might have died without acquiring
```

```
   When req-0001 releases:
       ─▶ only req-0002 wakes up (it's the only watcher)
       ─▶ req-0003, req-0004 keep waiting on their own predecessors
       ─▶ no herd
```

> **Analogy** — Single-file deli queue. Each customer takes a ticket and only watches the person directly in front of them. When that person is served and leaves, only you notice — everyone else is busy watching their own neighbor. Compare with the simple-lock version: a deli where everyone watches the *counter*, and whenever it's free, fifty people rush forward.

### 8.4 What these locks actually mean

> **Important**: ZooKeeper locks are **not** the same as in-process mutexes.

Differences:

```
   If the lock holder CRASHES:
       Ephemeral znode auto-deletes → lock is automatically released
       → other clients can proceed
       BUT: the crashed client's half-finished work is NOT undone.
```

So ZooKeeper locks **do not** make sequences of operations atomic. They give you:

- **Master election** — only one client holds the lock at a time, others wait. The winner is the master.
- **Soft locks** — useful for performance ("one worker per task"), not for correctness if duplicate work is harmless (idempotent tasks).

For atomicity, combine locks with the **"ready" pattern** or **mini-transactions**.

### 8.5 Leader election sketch

```python
# Each candidate runs:
me = create("/masters/primary", my_address, ephemeral=True)
if me succeeded:
    # I am the master.
    # Recover state from /config, clean up, start serving.
else:
    # Watch the existing master.
    exists("/masters/primary", watch=True)
    # When it disappears, retry the election.
```

> **The killer feature**: when the master dies, the ephemeral znode vanishes **automatically**, and watchers learn immediately. No timeout-driven polling.

---

## 9. Verdict

### 9.1 Where ZooKeeper shines

- **Master election** — the bread and butter use case.
- **Configuration distribution** — small slowly-changing data with watches.
- **Service discovery** — "who's currently the primary?"
- **Worker registration & work queues** — ephemeral nodes + sequential ordering.
- **Persistent small state** — replicated, fault-tolerant, in-memory.

### 9.2 What ZooKeeper does *not* replace

- **The GFS master's own replicated metadata** — too much data for ZooKeeper.
- **GFS's chunk-replication scheme** — application-specific.
- **General-purpose data storage** — it's coordination, not a database.

> **Idiom**: ZooKeeper rarely removes *all* the complexity of a distributed system. It removes a recurring **chunk** of it — the part where you'd otherwise reinvent locks, leader election, and watchers from scratch.

### 9.3 Topics this lecture skipped

- Persistence implementation details
- Specifics of batching and pipelining
- Fuzzy snapshot mechanics
- Idempotent operations
- Duplicate request detection

(Covered in the FAQ and Lab 3.)

---

## 10. Takeaway

> **ZooKeeper trades read linearizability for read throughput, then sells the savings as a general-purpose coordination service.** Linearizable writes plus FIFO client-order reads cover most coordination patterns. A tiny filesystem-shaped API (atomic create, versioned setData, ephemeral nodes, watches) is expressive enough to build locks, queues, leader election, and configuration distribution.

### Mental checklist

| Concept | One-liner |
|---|---|
| **Linearizable writes** | Same as Raft — leader assigns zxid, replicas apply in order |
| **FIFO client order** | Per-client reads + writes appear in submission order, reads never time-travel |
| **`sync()`** | Force a single read to see all prior committed writes |
| **`zxid` tracking** | Client carries highest-seen zxid; replicas refuse older reads |
| **Async writes + batching** | High write throughput from many in-flight clients |
| **Znode types** | Regular, ephemeral (auto-cleanup), sequential (auto-numbered) |
| **Watches** | One-shot notifications — replaces polling |
| **Mini-transaction** | `getData` + version-checked `setData` retry loop |
| **Herd-free lock** | Sequential ephemeral nodes, watch only your predecessor |
| **Master election** | `create("/masters/primary", ephemeral=true)` race |
