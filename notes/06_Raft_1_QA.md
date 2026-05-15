# Lecture 6 — Raft FAQ (Part 1)

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Topic: Raft consensus · Course site: [pdos.csail.mit.edu/6.824](http://pdos.csail.mit.edu/6.824)

---

## Table of Contents

1. [Simplicity vs. Performance](#1-simplicity-vs-performance)
2. [Raft in the Real World](#2-raft-in-the-real-world)
3. [Raft vs. Paxos](#3-raft-vs-paxos)
4. [Quorums and Partitions](#4-quorums-and-partitions)
5. [Leader Elections and Timeouts](#5-leader-elections-and-timeouts)
6. [Log Commitment and Client Requests](#6-log-commitment-and-client-requests)
7. [Security and Failure Model](#7-security-and-failure-model)
8. [Application Constraints](#8-application-constraints)
9. [References](#9-references)

---

## 1. Simplicity vs. Performance

**Q: Does Raft sacrifice anything for simplicity?**

**A:** Yes. Raft favors clarity over maximum performance. Examples:

- Every operation is persisted immediately. High performance usually batches writes.
- Only one `AppendEntries` is effectively in flight per follower (no pipelining).
- Snapshotting writes the full state, which is expensive for large databases.
- Catching up replicas with full snapshots is slow if they already have old state.
- Log order limits parallel execution on multicore systems.

These are deliberate trade-offs to keep the protocol teachable.

---

## 2. Raft in the Real World

**Q: Is Raft used in production systems?**

**A:** Yes. Examples include:

- Docker Swarm: https://docs.docker.com/engine/swarm/raft/
- etcd: https://etcd.io
- MongoDB
- CockroachDB, RethinkDB, TiKV (reported)

Many other systems are based on Multi-Paxos or Viewstamped Replication.

---

## 3. Raft vs. Paxos

**Q: What is Paxos, and how is Raft simpler?**

**A:** Paxos solves agreement on a **single value**. Raft solves agreement on an **indefinite sequence** of values *plus* recovery mechanisms for real systems.

Paxos is actually smaller in scope, but the real-world systems built on it are complex and often poorly explained. Raft’s strength is a clear, end-to-end description of a complete replicated service.

**Timeline:**
- Paxos invented in the late 1980s.
- Viewstamped Replication (very similar to Raft) published in 1988.
- Raft developed around 2012.

**Performance note:** Paxos variants can be faster in some settings, and leaderless variants (e.g., ePaxos) avoid a leader bottleneck when data centers are far apart.

---

## 4. Quorums and Partitions

**Q: Can Raft survive with only a minority of servers alive?**

**A:** Not with Raft’s safety properties. A majority is required to commit, or split brain becomes possible.

### Diagram: majority rule

```
5-node cluster
Majority = 3

Partition A: 3 nodes  -> can elect leader and commit
Partition B: 2 nodes  -> cannot elect leader
```

If no partition has a majority, the system pauses until the network heals.

---

## 5. Leader Elections and Timeouts

**Q: How long do elections pause client traffic?**

**A:** Usually around a tenth of a second. Failures are expected to be rare.

**Q: Why randomize election timeouts?**

**A:** To reduce split votes when two candidates start elections at the same time.

**Q: Is a too-short timeout dangerous?**

**A:** It hurts liveness, not safety. Too short causes constant elections; too long causes slow recovery.

---

## 6. Log Commitment and Client Requests

**Q: What if a leader crashes before replication finishes?**

**A:** The request may be lost. If it wasn’t committed, no client reply should have been sent. Clients must retry and handle duplicates.

**Q: When do followers apply log entries?**

**A:** Only after the leader marks them committed via `leaderCommit`.

**Q: Should a leader wait for all AppendEntries replies?**

**A:** No. Send them concurrently and commit after receiving a majority.

### Sketch: concurrent replication

```
for each follower:
  go send AppendEntries
  if reply ok: count++
if count >= majority: commit
```

---

## 7. Security and Failure Model

**Q: Why can’t Raft defend against malicious servers?**

**A:** Raft assumes fail-stop behavior (non-Byzantine). Malicious behavior breaks safety.

In practice, deployments rely on:
- firewalls
- authenticated messages
- trusted infrastructure

If Byzantine faults are expected, a different protocol is required.

---

## 8. Application Constraints

**Q: Are there limitations on applications built on Raft?**

**A:** Yes. The application must be **self-contained** and deterministic. If it talks to external systems, those systems must handle duplicate requests and inconsistent timing.

### Analogy: A synchronized choir

Every singer must sing the **exact same notes** at the **exact same time**. If one singer starts calling a separate orchestra, synchronization breaks down.

Example: a replicated ordering system calling a credit-card processor. If each replica makes its own external request, the card might be charged multiple times.

---

## 9. References

- Paxos Made Simple: http://css.csail.mit.edu/6.824/2014/papers/paxos-simple.pdf
- Raft page: http://raft.github.io/
- Ongaro thesis: http://ramcloud.stanford.edu/~ongaro/thesis.pdf
- Verified Raft proof: http://verdi.uwplse.org/raft-proof.pdf
{"mode":"full","isActive":false}