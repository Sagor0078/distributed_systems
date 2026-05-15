# Lecture 8 — ZooKeeper: FAQ

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Companion Q&A to [08_ZooKeeper.md](08_ZooKeeper.md) — covers consistency model, pipelining, wait-freedom, snapshots, watches, and operational concerns.

---

## Table of Contents

1. [Consistency Model](#1-consistency-model)
2. [Pipelining & FIFO Client Order](#2-pipelining--fifo-client-order)
3. [Wait-Freedom](#3-wait-freedom)
4. [Client Sessions, Retries & Duplicate Detection](#4-client-sessions-retries--duplicate-detection)
5. [Fuzzy Snapshots](#5-fuzzy-snapshots)
6. [Leader Election & Performance](#6-leader-election--performance)
7. [Coordination Patterns (Locks, Barriers)](#7-coordination-patterns-locks-barriers)
8. [Operational Questions](#8-operational-questions)
9. [Watches — Implementation](#9-watches--implementation)
10. [Cheat Sheet](#10-cheat-sheet)

---

## 1. Consistency Model

### Q: Why are only **writes** linearizable, not reads?

**A.** Performance — total read throughput.

```
   ┌─────────┐     writes ─────▶ ┌─────────┐
   │ Clients │                   │ LEADER  │  ← all writes funnel here (slow)
   └────┬────┘                   └────┬────┘
        │                              │ Zab broadcast
        │  reads (fast)                ▼
        └─────────────▶  Any of N replicas   ← reads served locally
```

A replica may **lag** the leader: it might not yet know about a committed write, or know about a write but not yet that it's committed. So a read served by a replica can return **stale** data.

The trade-off is intentional: **stale reads in exchange for scalable read throughput**. With N replicas, total read capacity scales ~Nx.

> **Analogy** — A library has many branches sharing a master catalog. A new book is added at headquarters; some branches receive the update minutes later. If you ask "is this book available?" at a slow branch, you might be told no even though the master catalog says yes. The library trades freshness for the ability to serve many patrons in parallel.

### Q: How does linearizability differ from serializability?

**A.** Both define a single total order. The difference:

| Property | Total order? | Respects real time? |
|---|---|---|
| **Linearizability** | ✓ | ✓ (if op A finished before B started, A < B) |
| **Serializability** | ✓ | ✗ (the order can ignore wall-clock time) |

> Reference: <http://www.bailis.org/blog/linearizability-versus-serializability/>

Section 2.3 of the ZooKeeper paper uses **"serializable"** to mean: writes appear to execute in some single order. Reads get a weaker guarantee: each read happens "at some point" in that write order, and a single client's reads never go backwards.

> ZooKeeper's two-tier guarantee:
> - **Writes** = linearizable (strong)
> - **Reads** = FIFO-client-order + observed-write-order monotonic (weaker but cheaper)

---

## 2. Pipelining & FIFO Client Order

### Q: What is pipelining?

**A.** Two distinct ideas tangled together:

**(a) Server-side batching.** The leader's Zab layer groups many client operations into one large network send + one large disk write.

```
   Without batching:                 With batching:
   write1 ─▶ disk ─▶ ack             write1 ┐
   write2 ─▶ disk ─▶ ack             write2 ├─▶ ONE disk write ─▶ ack all
   write3 ─▶ disk ─▶ ack             write3 ┘
   3 fsyncs (slow)                   1 fsync (much faster)
```

**(b) Client-side asynchrony.** Clients fire off many writes without waiting for replies. Replies arrive later as notifications.

```
   Synchronous (slow):                Asynchronous pipelined (fast):
   send w1, wait reply                send w1
   send w2, wait reply                send w2
   send w3, wait reply                send w3
   ...                                ... later: receive all 3 acks
```

Asynchrony **enables** batching — without many in-flight requests, the leader has nothing to batch.

### Q: Doesn't pipelining risk reordering writes?

**A.** Yes — and that's what the paper's §2.3 example is about. Imagine a client doing:

```
   client: write(/config/key1, ...)
           write(/config/key2, ...)
           write(/config/ready, true)
```

If those got reordered, another client might observe `ready=true` before the keys are populated — disaster.

ZooKeeper's **FIFO client order** prevents this: all operations from a single client are applied **in the order the client issued them**, end-to-end.

### Q: How does FIFO order solve the §2.3 race?

**A.** Same example: a client populates config znodes, then sets `Ready`. Because of FIFO:

```
   In log order:    write key1 → write key2 → write Ready
   When observer sees Ready=true → ALL preceding writes have already
                                   been applied → safe to read keys
```

> **Analogy** — A pneumatic tube system at a bank. You drop in deposit slips numbered 1, 2, 3. The tube system guarantees they arrive at the teller **in order**, even though the tubes themselves might overtake each other physically. ZooKeeper is the tube system; FIFO client order is the numbering.

### Q: How does the leader know what order a client wanted async writes in?

**A.** Paper is silent. Likely mechanism: the client numbers requests sequentially per session; the leader tracks the next expected number per session. State already needed for session timeouts, so adding sequence numbers is cheap. These sequence numbers must be replicated (so a failover leader knows them).

---

## 3. Wait-Freedom

### Q: What does "wait-free" mean precisely?

**A.** Herlihy's definition (1991):

> A wait-free implementation guarantees that **any process can complete any operation in a finite number of steps**, regardless of the execution speed of other processes.

In plain words: **no client ever has to wait on another client** to make progress.

### Q: How is ZooKeeper wait-free if clients use it to wait for each other?

**A.** The **API itself** is wait-free. No single call blocks on another client. Contrast with a typical lock service:

```
   ✗ NOT wait-free:                 ✓ ZooKeeper (wait-free):
   acquire_lock(L)                   create_ephemeral("/locks/L")
   ── blocks until holder            ── returns immediately
      releases it ──                  (success OR collision)
```

But clients **do** often need to wait. ZooKeeper provides this through a separate primitive — **watches** — which the client opts into.

> **The pattern**: factor blocking out of the basic API. Compose atomic operations + watches into higher-level blocking abstractions (locks, barriers, queues).

### Q: How can you build a lock from these primitives?

**A.** Sketch:

```
   acquire(L):
       my_node = create_ephemeral_sequential("/locks/L/req-")
       while True:
           children = list("/locks/L")
           if my_node is the lowest:
               return  // I have the lock
           prev = the child just before mine
           set_watch_on(prev)
           wait_for_watch
```

The **operations** (create, list, set_watch) are wait-free. The **composition** of them with the application's wait_for_watch loop produces a lock — but ZooKeeper itself never blocks.

> **Analogy** — A deli counter. You take a ticket (wait-free: the dispenser never blocks). You then choose whether to wait for your number to be called. The dispenser is wait-free; your behavior may not be.

---

## 4. Client Sessions, Retries & Duplicate Detection

### Q: What does a client do if a request times out?

**A.** Paper doesn't spell it out. The likely scheme (similar to Lab 3):

```
   Client tags each request with (sessionID, requestSeq).
   Leader tracks per-session: "highest committed requestSeq".
   On duplicate (seq already committed):
       return cached reply, do NOT re-execute.
```

This is the same duplicate-detection pattern as Raft labs.

### Q: After an async write, if the client immediately issues a read, will the read see the write?

**A.** Yes — that's exactly what **FIFO client order** guarantees.

Implementation hint: a client likely sends, with each read, the **zxid of its most recent submitted operation**. The serving replica blocks the read until it has applied up through that zxid.

```
   Client                  Replica
   ──────                  ───────
   async write(x=5)        (zxid 42)
   read(x), bring=42       Replica: "have I applied up to zxid 42?"
                                    if not, wait
                                    then serve read → returns 5
```

> **Analogy** — You drop a letter in the mailbox at 3 PM and then ask the postmaster "did my letter go out?" at 3:05. The postmaster checks the outgoing tray; if your letter is still sitting there, they wait until it ships before answering. From your perspective, you never see "no letter" after sending it.

---

## 5. Fuzzy Snapshots

### Q: Why "fuzzy" snapshots — what's wrong with a precise one?

**A.** A **precise snapshot** corresponds to exactly one log point: includes every op before it, none after. But making one requires **blocking writes** while you serialize the in-memory database. For a busy service, that's unacceptable downtime.

### Q: How does fuzzy snapshotting work?

**A.** Snapshot the live in-memory database **without** blocking writes. The result is **inconsistent** — some pages reflect writes A, B, C; others were captured before A.

```
   In-memory DB:    ┌────┐ ┌────┐ ┌────┐ ┌────┐
                    │ k1 │ │ k2 │ │ k3 │ │ k4 │
                    └─┬──┘ └─┬──┘ └─┬──┘ └─┬──┘
   Snapshot reads:    │      │      │      │
   (slow scan)     ───●──────●──────●──────●───▶
                     |W1     |W2,W3 |W4     |
                     ↑       ↑      ↑       ↑
                  pages captured at different times → fuzzy
```

### Q: How is it usable, then?

**A.** Crucial trick: **idempotent transactions + log replay**.

```
   Recovery procedure:
   1. Load fuzzy snapshot (taken starting at log index S).
   2. Replay ALL log entries from index S onward.
   3. Result: consistent state.

   Replaying a write that's already reflected in the snapshot is harmless
   because the transaction is idempotent.
```

ZooKeeper translates client API calls into idempotent transactions. Example:

```
   Client API:           setData(/foo, "v", version=7)
                                          ↑
                                   "only if currently v7"

   Leader generates:     setDataTXN(/foo, value="v", newVersion=8,
                                    timestamp=...)

   Applied twice? Same result. → idempotent.
```

> **Analogy** — Photographing a hotel where guests come and go. A long-exposure photo captures different floors at different moments — some guests appear twice (in lobby and hall), some don't appear at all. **Useless by itself.** But if you also have the security log of every guest movement during the photo, you can "play forward" the log and reconcile the photo into a coherent picture of who was where.

---

## 6. Leader Election & Performance

### Q: How does ZooKeeper elect leaders?

**A.** Via **Zab** (ZooKeeper Atomic Broadcast), conceptually similar to Raft — has leader election baked in.

> Paper: <http://dl.acm.org/citation.cfm?id=2056409>

### Q: How does ZooKeeper's performance compare to Raft?

**A.** Substantially better:

```
   ZooKeeper (3 servers):    ~21,000 writes/sec
   Your Raft (3 servers):
       - magnetic disk:        tens of writes/sec
       - SSD:                  hundreds of writes/sec
```

The gap comes from **batching, pipelining, and engineering effort**, not algorithmic differences. ZooKeeper is a heavily optimized production system.

---

## 7. Coordination Patterns (Locks, Barriers)

### Q: How does a client know when to leave a barrier?

**A.** Each barrier-participant creates a znode. Each client watches the **other participants' znodes**:

```
   Barrier "/barrier-1":
       /barrier-1/client-A    ← ephemeral
       /barrier-1/client-B    ← ephemeral
       /barrier-1/client-C    ← ephemeral

   Exit phase:
       Each client deletes its own znode and watches the others.
       Each client waits until ALL other znodes are gone.
       When they're all gone → everyone exits at once.
```

> **Analogy** — A diving team agrees to leave the boat together. Each diver puts a tag on a board to say "I'm still here". Each diver watches the board. When the board is empty, everyone surfaces.

### Q: What's a "universal object"?

**A.** A theoretical concept from Herlihy 1991. A primitive is universal if you can build **any** wait-free concurrent object using it. ZooKeeper's API (atomic create/setData with version checks + watches) is universal in this sense — proof that the API is expressive enough to coordinate anything.

> Wikipedia gentle intro: <https://en.wikipedia.org/wiki/Non-blocking_algorithm>
> Practical impact for this course: **none**. Just useful to know "ZooKeeper's API is expressive enough by formal proof."

---

## 8. Operational Questions

### Q: How big can the ZooKeeper database get? It's all in memory.

**A.** ZooKeeper is for **configuration and coordination**, not bulk data storage. A few GB typically — easily fits in modern server RAM.

Reference point: GFS's master held all metadata in memory and that worked fine.

### Q: Can you add servers without taking the cluster down?

**A.** Yes — **dynamic reconfiguration** was added after the original paper.

- Docs: <https://zookeeper.apache.org/doc/r3.5.3-beta/zookeeperReconfig.html>
- Paper: <https://www.usenix.org/system/files/conference/atc12/atc12-final74.pdf>

Compares interestingly with Raft's joint-consensus scheme (which arrived two years later).

---

## 9. Watches — Implementation

### Q: How are watches implemented client-side?

**A.** Implementation-defined. Common patterns:

| Style | Client API | Notification mechanism |
|---|---|---|
| **Callback** | `getW(path, callback)` | Library invokes `callback(event)` on a worker thread |
| **Channel** (Go) | `GetW(path)` returns `<-chan Event` | Caller does `case ev := <-ch:` in `select` |

Example Go client: <https://godoc.org/github.com/samuel/go-zookeeper/zk#Conn.GetW>

```go
// Sketch (Go)
data, _, watchCh, err := conn.GetW("/foo")
// ... do other work ...
select {
case ev := <-watchCh:
    // watch fired — /foo changed; re-read it
case <-time.After(5 * time.Second):
    // unrelated timeout
}
```

The watch is **one-shot** — fires once, then you must re-register.

---

## 10. Cheat Sheet

| Concept | One-liner |
|---|---|
| **Linearizable writes / FIFO reads** | Strong writes, scalable but possibly stale reads |
| **Pipelining** | Async clients + server batching → high throughput |
| **FIFO client order** | All ops from one client applied in submission order |
| **Wait-free API** | No call ever blocks on another client; blocking built via watches |
| **Sessions + seq numbers** | Duplicate detection; survives leader failover |
| **Fuzzy snapshot** | Non-blocking capture + idempotent log replay = consistent recovery |
| **Zab** | ZooKeeper's consensus protocol; Raft-like, with engineering polish |
| **Watch** | One-shot notification trigger; the only blocking primitive |
| **Universal object** | Formal proof ZooKeeper's API is expressive enough for any coordination task |
| **Dynamic reconfig** | Added post-paper; comparable to Raft's joint consensus |
