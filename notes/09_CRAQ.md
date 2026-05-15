# Lecture 9 — Chain Replication & CRAQ

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Paper: *"Object Storage on CRAQ: High-throughput chain replication for read-mostly workloads"* — Terrace & Freedman, USENIX 2009
> Original Chain Replication paper: van Renesse & Schneider, OSDI 2004
> Companion FAQ: [09_CRAQ_QA.md](09_CRAQ_QA.md)

---

## Table of Contents

1. [The Big Ideas](#1-the-big-ideas)
2. [Chain Replication (CR)](#2-chain-replication-cr)
3. [Why CR Is Attractive vs. Raft](#3-why-cr-is-attractive-vs-raft)
4. [The Problem CRAQ Solves](#4-the-problem-craq-solves)
5. [CRAQ — Reading from Any Replica](#5-craq--reading-from-any-replica)
6. [Why CRAQ Can Do This but Raft Can't](#6-why-craq-can-do-this-but-raft-cant)
7. [The Catch: Partitions & the Configuration Manager](#7-the-catch-partitions--the-configuration-manager)
8. [Takeaway](#8-takeaway)

---

## 1. The Big Ideas

| Idea | What it gives you |
|---|---|
| **Chain Replication** | A primary/backup scheme arranged in a line — head → ... → tail |
| **CRAQ's twist** | Linearizable reads from **any** replica, not just the tail |
| **Configuration manager** | An external authority (ZooKeeper) handles failures the chain can't |

This lecture is partly a CR refresher and partly an answer to one question: **how do you scale read throughput when even the tail is saturated?**

---

## 2. Chain Replication (CR)

### 2.1 The shape

```
   Client writes ─▶ ┌────┐    ┌────┐    ┌────┐    ┌────┐
                    │ S1 │ ──▶│ S2 │ ──▶│ S3 │ ──▶│ S4 │
                    └────┘    └────┘    └────┘    └────┘
                    HEAD       middle    middle    TAIL ──▶ Client reads
                                                            (and write acks)
```

### 2.2 Write path

```
   1. Client sends write to HEAD.
   2. HEAD overwrites locally, forwards to S2.
   3. S2 overwrites, forwards to S3.
   4. S3 overwrites, forwards to TAIL.
   5. TAIL overwrites, sends ACK back to client.
   6. ACKs walk back up the chain (so each node learns "committed").
```

Once the tail has the write, **every node has it** — that's the chain's invariant.

### 2.3 Read path (vanilla CR)

```
   Client ─▶ TAIL ─▶ reply
   (No other nodes touched. No coordination needed.)
```

### 2.4 Why reads come from the tail

Only the tail has a definitive answer to "what's committed". Reading from earlier in the chain might return a write that **hasn't reached the tail yet** — that write could vanish if a node fails before it propagates further.

### 2.5 Goals achieved

- **Durability**: if the client gets an ACK, the data survives any single node failure (the tail had it → every node had it).
- **Linearizability**: the tail is effectively a single server defining the global order.

### 2.6 Failure recovery — at a glance

| Failure | Recovery |
|---|---|
| **Head dies** | S2 becomes the new head. No committed writes lost. May need to re-replay in-flight writes. |
| **Tail dies** | The predecessor (S3) becomes the new tail. Some writes that were "almost committed" complete. |
| **Middle dies** | Drop it from the chain. The predecessor may need to re-send recent writes to the new successor. |

The "good news": **every replica knows every committed write**. Recovery just adjusts the chain topology — no log reconciliation à la Raft.

### 2.7 Analogy — A bucket brigade

A line of firefighters passing buckets from a hydrant (head) to the fire (tail). The chief (client) hands buckets to the first firefighter and waits to hear the last firefighter shout *"poured!"*. Once that shout comes back, every firefighter has touched the bucket — losing any one of them doesn't lose water. New buckets only count as "fought the fire" once they reach the end of the line.

---

## 3. Why CR Is Attractive vs. Raft

CR and Raft are both **replicated state machines** — same expressive power. CR's advantages are *engineering* advantages:

### 3.1 Spread the work

```
   Raft leader's outbound traffic per write:    Σ followers
   CR head's outbound traffic per write:        1 neighbor

   Raft leader's read load:                     all client reads
   CR tail's read load:                         all client reads
                                                (but separate from the head!)
```

CR splits client load between **head (writes)** and **tail (reads)**. Raft funnels both through the leader.

### 3.2 Send each write just once

The CR head sends the write to one neighbor, who sends it onward. The bandwidth cost of replication is **spread across the chain**, not piled onto the head's NIC.

### 3.3 Cleaner failure story

> Recall Raft's Figure 7 — diverging logs, election restriction, no-op commits, fast-backup edge cases. Painful.

In CR, after a node dies and the configuration manager picks a new chain, the only cleanup is *"forward any in-flight writes that didn't quite reach the tail"*. No log-divergence puzzles.

---

## 4. The Problem CRAQ Solves

### 4.1 Bottleneck: only head and tail do client work

In a 4-node chain:

```
   Client work distribution:
       S1 (head):    handles all incoming writes
       S2 (middle):  passes data through, otherwise idle
       S3 (middle):  passes data through, otherwise idle
       S4 (tail):    handles all client reads
```

If reads dominate (typical for object storage), **the tail saturates** while S2 and S3 sit there with idle CPU.

### 4.2 The obvious workaround: multiple chains

Run several chains on the same servers, rotating who plays head/middle/tail:

```
   Three servers, three chains:
       Chain C1:  S1(head) → S2(middle) → S3(tail)
       Chain C2:  S2(head) → S3(middle) → S1(tail)
       Chain C3:  S3(head) → S1(middle) → S2(tail)

   Net load per server: 1 head + 1 middle + 1 tail = balanced.
```

This works **if chain loads are roughly equal**. They often aren't — one shard becomes a hotspot and you're back to a bottleneck.

> **Multiple chains help, but they're not enough when a single hot chain dominates.**

### 4.3 CRAQ's pitch

> *"Let reads come from any node in the chain, while preserving linearizability."*

Reads can then escape into idle middle nodes — total throughput scales with chain length.

### 4.4 Why naive "read from any replica" fails

Two failure modes (also covered in the FAQ):

```
   ① Stale-then-vanishing read:
      Middle node has write W (not yet at tail).
      Client reads W from middle.
      Tail fails before W propagates → W never committed → client saw a write
      that essentially never happened.

   ② Time-travel between clients:
      Client A reads from a fast node, sees W.
      Client B reads from a slow node later, doesn't see W.
      Linearizability requires that no client see history go backwards.
```

Plain "read from any replica" doesn't work. CRAQ adds machinery.

---

## 5. CRAQ — Reading from Any Replica

### 5.1 Per-object versioning

Every replica stores **multiple versions** of each object:

```
   Object "foo" on replica S2:

       version 7   data = "old"     CLEAN     ← last committed
       version 8   data = "new"     DIRTY     ← propagated, not yet committed
```

- **Clean** = the tail has acknowledged this version → definitively committed.
- **Dirty** = received from upstream but not yet ACKed by the tail.

### 5.2 Write protocol (same as CR + dirty/clean marking)

```
   1. Client → HEAD
   2. HEAD creates a DIRTY version, forwards to S2
   3. S2 creates a DIRTY version, forwards to S3
   4. S3 creates a DIRTY version, forwards to TAIL
   5. TAIL creates a CLEAN version, ACKs upstream
   6. Each predecessor receives ACK, flips its DIRTY → CLEAN, drops old clean
   7. HEAD finally gets ACK, replies to client
```

### 5.3 Read protocol

```
   Read on a non-tail replica:

       latest version is CLEAN ──▶ reply with that data        (fast path)

       latest version is DIRTY ──▶ "VERSION QUERY" to TAIL
                                   tail replies: "my latest committed = vN"
                                   replica returns data for vN
                                   (either its clean copy, or its dirty if vN == dirty)
```

### 5.4 Why both clean *and* dirty are kept

```
   Replica state:  clean=v7,  dirty=v8

   Client reads → replica asks tail.
       Tail says "v8":  return dirty data
       Tail says "v7":  return clean data    ← need it!

   If the replica had discarded v7 when v8 arrived,
   it couldn't answer "tail says v7" without a re-fetch.
```

### 5.5 Why this is linearizable

```
   Case A — replica's latest is CLEAN:
       That means no later write has reached this replica yet.
       Either the chain is quiescent, or upstream writes haven't propagated.
       Either way, the replica's clean version is consistent with the tail.
       (If the tail had a newer version, it would have propagated through here.)

   Case B — replica's latest is DIRTY:
       The replica explicitly defers to the tail's view.
       The tail's answer is by definition committed.
       So the client sees exactly what the tail would have served.
```

In both cases, the client gets a value the tail itself would have returned — same as plain CR.

### 5.6 The cost

A read on a replica with a dirty version requires **one extra RPC to the tail** (just a version number, no data). Cheap compared to shipping the object.

```
   Reads against quiescent objects (no recent writes):  0 extra RPCs
   Reads against actively written objects:              1 extra RPC each
```

### 5.7 Analogy — Library card catalog

A library has many branch reading rooms (replicas), but the main reference desk (tail) is the official source of truth.

- A book that's been on the shelf for a while → branches can hand it out directly. (clean)
- A new book is being processed across branches → branches don't yet know if it's officially catalogued. Before lending, they phone the reference desk: *"is book X officially available yet?"* The desk replies with the latest official version number, and the branch hands out **that** version — either the new one (if officially catalogued) or the older edition (if not).

Branches that touch only stable books answer instantly. Branches dealing with new arrivals incur one phone call.

---

## 6. Why CRAQ Can Do This but Raft Can't

> Key question: why doesn't a Raft follower do the same trick — answer reads from local state, ask the leader for the current commit point?

### 6.1 The chain invariant

In CR/CRAQ, a write **passes through every node** before committing. So every node:

- Knows about every write that **might be committed** (it has the dirty copy).
- Therefore knows when it *might* be stale (it has dirty objects).
- Can correctly decide when to consult the tail.

### 6.2 Raft's "majority" invariant

In Raft, the leader can commit a write **without all followers seeing it**:

```
   Leader writes W.
   Sends to F1, F2, F3, F4.
   F1, F2 ack → majority → W is COMMITTED.
   F3 and F4 may never receive W (or receive it much later).
```

A follower that didn't receive W has **no idea** W exists. It can't decide whether to ask the leader for confirmation — from its perspective, there's nothing to ask about. So it can't safely serve reads locally.

### 6.3 The trade

```
   CR/CRAQ:  every node sees every write → can do local reads with version queries
             BUT: every node must be alive for writes to progress

   Raft:     only majority needs to see each write → tolerates minority failures
             BUT: followers don't know what they don't know → can't serve local reads
```

> **Trade-off in one line**: CRAQ traded **fault tolerance during writes** for **read scalability**.

---

## 7. The Catch: Partitions & the Configuration Manager

### 7.1 CRAQ can't tolerate a partition by itself

If a chain node becomes unreachable, **writes stall**. CRAQ doesn't have a Raft-style majority quorum to keep going.

```
   ┌────┐    ┌────┐    ┌────┐    ┌────┐
   │ S1 │ ──▶│ S2 │ ╳  │ S3 │ ──▶│ S4 │
   └────┘    └────┘    └────┘    └────┘
                       ▲
                       S2-S3 link broken — chain is stuck
```

S2 doesn't know whether S3 crashed or is just unreachable. If S2 "promoted" S3's successor unilaterally, two parts of the network might form competing chains. **Split brain.**

### 7.2 The fix: defer to a configuration manager

```
   ┌──────────────────────────────────────────┐
   │  CONFIGURATION MANAGER  (ZooKeeper)      │
   │  - monitors all chain nodes              │
   │  - decides who's alive                   │
   │  - dictates the chain                    │
   │  - replicated via Paxos/Raft/Zab         │
   └──────────────┬───────────────────────────┘
                  │ "new chain = [S1 → S2 → S4]"
                  ▼
   ┌────┐    ┌────┐         ┌────┐
   │ S1 │ ──▶│ S2 │ ──────▶ │ S4 │     ← all servers & clients
   └────┘    └────┘         └────┘       follow the manager's decree
```

**Rules**:

- Every server and client obeys the manager's chain, **regardless** of what each locally thinks is alive.
- The manager itself is fault-tolerant (Paxos/Raft).
- Only one chain configuration is in effect at a time → no split brain.

### 7.3 This is a common architecture

```
   ┌─────────────────────────────────────────────────────────────┐
   │ Pattern: split metadata from data                           │
   ├─────────────────────────────────────────────────────────────┤
   │ • Metadata (small, critical):                               │
   │     replicated via Paxos/Raft/Zab → slow but safe            │
   │ • Data (huge, performance-critical):                        │
   │     sharded into groups, each replicated via CR / GFS / etc. │
   ├─────────────────────────────────────────────────────────────┤
   │ Examples:                                                   │
   │   GFS:         master (Paxos-like)   +  chunkservers         │
   │   VMware FT:   test-and-set service   +  primary/backup      │
   │   CRAQ:        ZooKeeper              +  chains              │
   │   Lab 4:       Raft (config)          +  Raft per shard      │
   └─────────────────────────────────────────────────────────────┘
```

> **Why this works**: the metadata service handles the "who's alive?" decisions that need consensus; the data layer can use a faster, simpler protocol (like CR) because it never has to make those decisions itself.

### 7.4 Analogy — Air traffic control

Pilots (chain nodes) don't argue about who's allowed to land if one of them goes silent. They defer to the control tower (configuration manager). The tower has its own redundancy and chain-of-command (Paxos/Raft) so it never gives conflicting orders. Pilots just follow.

---

## 8. Takeaway

> **Chain Replication is a primary/backup scheme reorganized into a line, so write traffic spreads across nodes instead of bottlenecking at one leader. CRAQ extends CR with per-object version tracking, letting any replica serve linearizable reads with at most one extra version-query RPC. The trade is fault tolerance: every node must be alive for writes to progress, so CRAQ relies on an external Paxos/Raft-based configuration manager to handle partitions and failures.**

### Mental checklist

| Concept | One-liner |
|---|---|
| **Head / Middle / Tail** | Writes enter at head, propagate down, tail is the commit point |
| **Read from tail (CR)** | Only the tail's view is definitive |
| **Per-object dirty/clean** | CRAQ marks recent writes dirty until the tail ACKs |
| **Version query** | A replica with a dirty value asks the tail for the latest committed version number |
| **Linearizability** | Preserved because either local clean = tail, or the read defers to the tail |
| **Why not Raft?** | Raft followers don't see every write, so they can't know when they're stale |
| **Multiple chains** | Rotate roles across servers to balance load — partial alternative to CRAQ |
| **Configuration manager** | External fault-tolerant authority that picks the chain (avoids split brain) |
| **Common pattern** | Small Paxos/Raft service controls many fast CR-like data groups |

The recurring theme: **separate the consensus problem from the data-movement problem**. Use a heavyweight protocol (Paxos/Raft) for the small, critical decisions; use a lightweight protocol (CR/CRAQ) for the bulk data flow.
