# Lecture 9 — CRAQ & Chain Replication: FAQ

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Companion Q&A to [09_CRAQ.md](09_CRAQ.md)
> Paper: *"Object Storage on CRAQ: High-throughput chain replication for read-mostly workloads"* — Terrace & Freedman, USENIX 2009

---

## Table of Contents

1. [Partitions & Split Brain](#1-partitions--split-brain)
2. [CR/CRAQ vs. Raft/Paxos](#2-crcraq-vs-raftpaxos)
3. [CR vs. Classic Primary/Backup](#3-cr-vs-classic-primarybackup)
4. [Spreading Load: Multiple Chains](#4-spreading-load-multiple-chains)
5. [Why "Sticky Reads" Don't Save Vanilla CR](#5-why-sticky-reads-dont-save-vanilla-cr)
6. [Failure Model & Byzantine Faults](#6-failure-model--byzantine-faults)
7. [Real-World Usage & Alternatives](#7-real-world-usage--alternatives)
8. [CRAQ Internals: Why Keep the Old Clean Copy?](#8-craq-internals-why-keep-the-old-clean-copy)
9. [Cheat Sheet](#9-cheat-sheet)

---

## 1. Partitions & Split Brain

### Q: How does CRAQ cope with network partitions and prevent split brain?

**A.** **It doesn't — by itself.** Chain Replication and CRAQ delegate the problem to an external **configuration service**.

```
   ┌───────────────────────────────────────┐
   │  Configuration Service (ZooKeeper)    │   ← decides chain membership
   │  fault-tolerant via Paxos/Raft/Zab    │     monitors liveness
   └────────────┬──────────────────────────┘
                │ "the chain is now [B → C → D]"
                ▼
        ┌────┐    ┌────┐    ┌────┐    ┌────┐
        │ A  │ ╳  │ B  │ ──▶│ C  │ ──▶│ D  │
        └────┘    └────┘    └────┘    └────┘
        dropped     new       middle    new
                    head                 tail
```

**The protocol:**
1. A chain node stops responding → other chain members stall (wait, do nothing dangerous).
2. The config service notices via its own health checks.
3. The config service computes a new chain and broadcasts it to nodes + clients.
4. Operations resume on the new chain.

The config service avoids its **own** split brain by running Paxos/Raft/ZooKeeper internally.

> This is exactly the same architecture as **GFS** (master picks chunk primaries) and **VMware FT** (test-and-set server picks the sole survivor).

> **Analogy** — A pipeline of factory workers passing parts down the line. If one worker collapses, the line halts. A foreman (config service) walks the floor, decides who's recovered, who needs replacing, and reorders the line. The workers don't have to vote among themselves — a separate authority resolves the question.

---

## 2. CR/CRAQ vs. Raft/Paxos

### Q: What are the tradeoffs?

**A.** All three are **replicated state machines** — same expressive power. They differ in where the work lands and how they react to failures.

| Aspect | Raft / Paxos | CR / CRAQ |
|---|---|---|
| **Write path** | Leader sends to **all** followers in parallel | Head sends to **one** neighbor; propagation walks the chain |
| **Read path** | Leader serves all reads | Tail (CR) or any node (CRAQ) serves reads |
| **Failure tolerance** | Survives loss of a **minority** with **no pause** | Any node failure **pauses the chain** until config service reconfigures |
| **Failure complexity** | Hairy — see Raft Figures 7, 8 | Simple — chain order makes recovery clean |
| **Throughput** | Bottlenecked at the leader | Spread across nodes |

### 2.1 Why CR is often faster

```
   Raft leader's work for one write:
       send to F1, F2, F3, F4   (4 outbound copies)

   CR head's work for one write:
       send to neighbor          (1 outbound copy)
```

The head sends **one** copy. Bandwidth is spread across the chain instead of bottlenecking at the leader.

### 2.2 Why Raft is more available

CR pauses on any node loss until reconfiguration completes. Raft keeps serving as long as a majority is up.

```
   3-node cluster, 1 node dies:
       Raft:    keeps going, no pause                   ← preferred when uptime matters
       CR:      stalls until config service reacts      ← preferred when throughput matters
```

> **Rule of thumb**: CR/CRAQ wins for **read-heavy, throughput-sensitive** workloads (object storage). Raft/Paxos wins for **availability-sensitive** workloads (coordination, locks).

---

## 3. CR vs. Classic Primary/Backup

### Q: Faster or slower than GFS-style primary/backup?

**A.** Depends on the chain length.

**Two replicas — roughly equal.** Chain Replication may win slightly because the tail replies **directly** to the client. Classic primary/backup makes the primary wait for backup ack, then reply itself.

```
   Classic 2-replica primary/backup:                CR with 2 replicas:
   Client ─▶ Primary ─▶ Backup                      Client ─▶ Head ─▶ Tail
            ◀──────── ack                                              │
   Client ◀──── reply                                Client ◀──────────┘
   (primary in critical path twice)                 (tail replies directly)
```

**Three+ replicas — CR wins on head bandwidth, but loses on latency.**

```
   Classic primary, 3 backups:
       Primary must send big writes to 3 backups → primary's NIC is a bottleneck.

   CR, 3-node chain:
       Each node sends to ONE neighbor → bandwidth spread evenly.
       But: write traverses 3 hops sequentially → latency higher.
```

> **Bandwidth vs. latency**: CR trades end-to-end write latency for evenly-distributed network load.

---

## 4. Spreading Load: Multiple Chains

### Q: The introduction mentions "multiple chains" to fix idle middle nodes. What does that mean?

**A.** In plain CR, **only the head and tail** are doing client-facing work. Middle nodes just pass data through and burn CPU on nothing useful.

**Trick**: run several chains on the same physical servers, rotating which node plays head/middle/tail.

```
   Three servers S1, S2, S3. Three shards C1, C2, C3.

       Chain C1:   S1(head)  →  S2(middle)  →  S3(tail)
       Chain C2:   S2(head)  →  S3(middle)  →  S1(tail)
       Chain C3:   S3(head)  →  S1(middle)  →  S2(tail)

   Total per-server work:
       S1:  head(C1) + middle(C3) + tail(C2)    ← balanced
       S2:  middle(C1) + head(C2) + tail(C3)    ← balanced
       S3:  tail(C1) + middle(C2) + head(C3)    ← balanced
```

Each server is the head of one chain, tail of another, middle of a third — total load is even.

**When does CRAQ still beat this?** When chain loads are **unequal** — a hot chain leaves the other chains' resources idle. CRAQ lets reads escape into the middle nodes regardless of which chain is hot.

> **Analogy** — A restaurant kitchen with three stations. Plain CR is like having each station do only one job (chopping, cooking, plating). Multiple chains is like rotating each station's role across three different dishes — the workload averages out. CRAQ is like saying "any station can plate any dish if the official plater is overwhelmed."

---

## 5. Why "Sticky Reads" Don't Save Vanilla CR

### Q: What if we just pin each client to one node for an entire session — is CR strongly consistent then?

**A.** **No.** Two distinct failures:

**Failure 1 — un-committed reads.** A non-tail node might have a write that **hasn't propagated** through the chain yet:

```
   Head ─▶ Middle ─▶ Tail
            ↑
       client reads here, sees uncommitted write W

   Then chain nodes after Middle fail → W is lost forever.
   But the client already saw W → "the write that never happened" → broken.
```

**Failure 2 — cross-client time travel.** Linearizability covers **all clients**, not one at a time:

```
   Time:    t=10           t=20
            Client A reads ──▶ sees W
                            Client B reads ──▶ doesn't see W
```

Even if A and B never switch nodes, if A sees a write and B (reading later in real time) doesn't, that's a linearizability violation.

> **Why CR's "read from tail" rule is special**: by the time the tail has it, every node has it → no committed write can vanish, and all nodes agree.

---

## 6. Failure Model & Byzantine Faults

### Q: Does CRAQ handle Byzantine failures?

**A.** **No.** CRAQ assumes **fail-stop**: nodes either work correctly or stop. Nodes don't lie, corrupt data, or send malicious messages.

### Q: What systems do handle Byzantine faults?

**A.** Two main approaches:

| Approach | Examples | Cost |
|---|---|---|
| **Byzantine consensus** (PBFT-style) | PBFT, Tendermint | Extra rounds + cryptography; requires 3f+1 nodes for f bad ones |
| **Client-side verification** | SUNDR, Bitcoin | Clients verify hashes/signatures; tricky to defend against **stale-but-signed** data |

> **Why it's hard**: a Byzantine adversary can return data whose signature is correct but value is outdated. The defense is freshness proofs (signed timestamps, chained hashes, blockchains), and these complicate everything.

---

## 7. Real-World Usage & Alternatives

### Q: Who uses Chain Replication?

**A.** Plain CR has many real-world users (though CRAQ specifically is rarer):

- **Amazon EBS** (block storage)
- **Ceph's RADOS** (object storage)
- **Google's Parameter Server** (ML training state)
- **COPS** (causally consistent geo-distributed storage)
- **FAWN** (energy-efficient cluster storage)

### Q: What alternatives exist?

```
   ┌──────────────────────────────────────────────────────────────┐
   │ Strong consistency, replicated state machines                │
   │   • Quorum (Paxos/Raft) — ZooKeeper, Chubby, Spanner          │
   │   • Chain Replication — Ceph, EBS                             │
   │   • Primary/Backup — GFS                                      │
   ├──────────────────────────────────────────────────────────────┤
   │ Trick to scale reads while keeping strong consistency        │
   │   • Leases (most common — see lecture 7)                      │
   │   • CRAQ's version-query approach (this paper)                │
   └──────────────────────────────────────────────────────────────┘
```

---

## 8. CRAQ Internals: Why Keep the Old Clean Copy?

### Q: When CRAQ sees a write and marks a new version "dirty", why hang on to the previous clean version?

**A.** Because the protocol may need to **return the old version** if asked.

### 8.1 The scenario

```
   On a non-tail node:
       clean:  version 1, data = "foo"
       dirty:  version 2, data = "bar"   ← propagated from head, not committed yet
```

Now a client reads. Node procedure:

```
   1. Ask the tail: "what's your latest committed version of this object?"
   2. Tail replies with its highest committed version number.
```

Two cases for the tail's reply:

```
   Tail says version 1:               Tail says version 2:
       Node returns "foo" (v1)            Node returns "bar" (v2)
       → must have kept v1!               → its dirty copy works
```

If the node discarded v1 the moment v2 arrived, **it couldn't satisfy the tail's "give me v1" answer**. So it keeps the old clean copy until the tail confirms the new version is committed (chain reaches the tail), at which point the dirty becomes clean and the old version is garbage-collected.

```
   Lifecycle of an object version on a non-tail node:

      (arrival)        (tail commits)
   ─── DIRTY ───────────▶ CLEAN ───────────▶ (overwritten by next write)
                              ▲
   old clean version kept ────┘
   alongside dirty, dropped only after the new version is clean
```

> **Analogy** — A magazine publisher with a printed issue on shelves (version 1) and the next issue at the printer (version 2). Until the new issue ships nationally, the publisher keeps reprinting the old issue for any store that requests it. Once the new issue is officially in circulation, the old plates are recycled.

---

## 9. Cheat Sheet

| Concept | One-liner |
|---|---|
| **Split brain in CRAQ** | Not solved internally — external config service (ZooKeeper) does it |
| **Head load** | One outbound copy per write — much lighter than Raft leader |
| **Tail role** | Authority for committed versions; serves reads in plain CR |
| **CR pause on failure** | Any node loss stalls the chain until reconfiguration |
| **Raft no-pause** | Survives minority loss with zero downtime |
| **Multiple chains** | Rotate head/tail roles across nodes for even load |
| **CRAQ's twist** | Dirty/clean per object + tail version query → reads from any node |
| **Old clean copy** | Kept so the node can answer tail-says-old-version |
| **Failure model** | Fail-stop only; not Byzantine |
| **Real users** | EBS, RADOS, COPS, FAWN, Google Parameter Server |
