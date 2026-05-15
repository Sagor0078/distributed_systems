# Lecture 7 — Raft (Part 2): Logs, Persistence, Snapshots & Linearizability

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Today: log reconciliation, persistence, log compaction (snapshots), linearizability, duplicate RPC detection, read-only optimizations
> Companion FAQ: [07_Raft_2_QA.md](07_Raft_2_QA.md)

---

## Table of Contents

1. [The Log Reconciliation Problem (Lab 2B)](#1-the-log-reconciliation-problem-lab-2b)
2. [The Election Restriction — Picking the Right Leader](#2-the-election-restriction--picking-the-right-leader)
3. [Fast Log Backup](#3-fast-log-backup)
4. [Persistence (Lab 2C)](#4-persistence-lab-2c)
5. [Snapshots & Log Compaction (Lab 3B)](#5-snapshots--log-compaction-lab-3b)
6. [Linearizability — What "Correct" Means](#6-linearizability--what-correct-means)
7. [Duplicate RPC Detection (Lab 3)](#7-duplicate-rpc-detection-lab-3)
8. [Read-Only Optimizations (Leases)](#8-read-only-optimizations-leases)
9. [Takeaway](#9-takeaway)

---

## 1. The Log Reconciliation Problem (Lab 2B)

### 1.1 The setup

While a leader is healthy, life is simple:

```
   Client ──▶ Leader ──▶ Followers       Clients never see follower logs.
                                          They only ever talk to the leader.
```

The drama begins **when the leader fails** and a new one takes over. Followers may have:

- **Missing entries** — the old leader crashed mid-replication.
- **Extra entries** — the old leader appended locally but never replicated.
- **Conflicting entries** — different commands occupy the same log slot after a series of crashes.

### 1.2 The safety property we must preserve

> **State Machine Safety** (Figure 3): If *any* server has applied a command at log index `i`, then **no** server applies a different command at index `i`.

Why? Because clients see the result of applied commands. If two servers disagree about what's at index `i`, the same client could see two different histories — the system stops behaving like a single coherent service.

```
   ✗ BAD:
      S1 applies index 12 = put(k1, v2)
      S2 applies index 12 = put(k2, x)
      → client behavior depends on which replica it talks to
```

### 1.3 How can logs disagree?

**Case A — Leader crashes mid-replication:**

```
   Term 3 leader was about to AppendEntries to all, then crashed.

   S1:  [3]                    ← got 0 of the appends
   S2:  [3] [3]                ← got 1 of the appends
   S3:  [3] [3]                ← got 1 of the appends
```

**Case B — Multiple successive crashes:**

```
   Index:    10  11  12  13
   S1:       3                  ← was leader briefly in term 3
   S2:       3   3   4          ← was leader in term 4
   S3:       3   3   5          ← was leader in term 5
```

Same slot (index 12), different terms, different commands. Now what?

### 1.4 The cleanup mechanism

> **Rule**: When a new leader is elected, **followers adopt the leader's log**, overwriting any conflicts.

The leader walks `nextIndex[follower]` **backwards** until it finds a prefix both share, then ships the rest. Step-by-step example (S3 is the new term-6 leader):

```
   S2:  ... 11  12  13       (12 has term=4)
   S3:  ... 11  12  13       (12 has term=5)   ← leader

   Step 1: S3 sends AppendEntries with prevLogIndex=12, prevLogTerm=5
           S2 checks: my entry at 12 has term=4, not 5 → REJECT
   Step 2: S3 decrements nextIndex[S2] to 12
           S3 sends AppendEntries with prevLogIndex=11, prevLogTerm=3 + entries [12, 13]
           S2 checks: my entry at 11 has term=3 → MATCH
                     S2 deletes its old index-12, accepts S3's entries 12 and 13
   Step 3: S1 has only index 10 — back up further, same process
```

After cleanup, **every follower's log = leader's log**.

### 1.5 Analogy — Editing a shared movie script

Imagine the director (leader) and several screenwriters (followers) keeping copies of a script. The director crashes mid-edit; some writers got page 47's revision, others didn't. A new director takes over. She walks the room: *"Compare page 47 with mine. Same? Good, move on. Different? Tear yours out and copy mine."* She works backwards from the latest page until she finds the common ancestor, then everyone re-copies forward from there.

---

## 2. The Election Restriction — Picking the Right Leader

### 2.1 The danger

When the new leader forces followers to adopt its log, **any entry not in the leader's log is lost**. That's catastrophic if a *committed* entry — one already acknowledged to a client — is among the lost.

> **Required invariant**: The newly elected leader **must have every committed entry** in its log.

### 2.2 Why "longest log" doesn't work

Tempting rule: *"elect the server with the longest log"*. Counter-example:

```
   S1:  [5][6][7]              ← longest, but 6 and 7 are NOT committed
   S2:  [5][8]                 ← shorter, but 8 may be committed
   S3:  [5][8]                 ← shorter, but 8 may be committed
```

How did this state arise?
1. S1 was leader in term 6 → appended 6 to itself → crashed.
2. S1 reboots, becomes leader in term 7 → appended 7 to itself → crashed and stays down.
3. S2 becomes leader in term 8 → appended 8 → replicated to S3 → may have committed.

If we elected S1 by "longest log", entry 8 — possibly committed — would vanish. **Disaster.**

### 2.3 The Election Restriction (§5.4.1)

A voter grants its vote only if the candidate is **"at least as up-to-date"** as itself, where up-to-date is defined as:

```
   Candidate is "more up-to-date" than voter if:
       candidate.lastLogTerm > voter.lastLogTerm
     OR
       candidate.lastLogTerm == voter.lastLogTerm
       AND candidate.log.length >= voter.log.length
```

> **Term first, length second.** A higher last-term beats a longer log.

### 2.4 Why it works

Apply to the example:

| Voter | Candidate S1 (last term=7) | Candidate S2 (last term=8) |
|---|---|---|
| **S2** (last term=8) | ✗ S1's last term 7 < 8 | ✓ same term, equal length |
| **S3** (last term=8) | ✗ S1's last term 7 < 8 | ✓ same term, equal length |

S1 cannot get a majority. Only S2 or S3 wins → committed entry 8 is preserved → S1 is forced to discard its uncommitted 6 and 7.

> *Note*: discarding 6 and 7 is fine — they were never committed → never replied to clients → clients will retransmit those commands.

### 2.5 Analogy — Promoting an heir

A kingdom must crown a new monarch from the dukes. The crown must go to someone who has seen the **most recent royal decree** — because they'll inherit the obligation to honor it. We don't pick by who has the **thickest scroll** of old decrees; we pick by who knows the **latest** ones. The Election Restriction is exactly this: the latest term wins, ties broken by completeness.

---

## 3. Fast Log Backup

### 3.1 The problem with Figure 2

The vanilla algorithm decrements `nextIndex` by **one** per failed `AppendEntries`. If a follower is 1000 entries behind, that's 1000 RPCs. The Lab 2B tester won't be patient.

### 3.2 The optimization (paper §5.3, fleshed out)

On rejection, the follower returns extra info:

```
   AppendEntries reply (on conflict):
       XTerm  = term of the conflicting entry in follower's log
       XIndex = index of follower's FIRST entry with that term
       XLen   = follower's log length
```

The leader uses three cases:

```
   Case 1 — leader has NO entries for XTerm:
       nextIndex = XIndex
       (jump past the follower's entire span of that term)

   Case 2 — leader HAS entries for XTerm:
       nextIndex = (leader's last entry for XTerm) + 1
       (rendezvous at the leader's authoritative tail of that term)

   Case 3 — follower's log is too short (no entry at prevLogIndex):
       nextIndex = XLen
       (skip to the end of follower's log)
```

### 3.3 Worked example

```
   Leader:    [4][6][6][6]                  XTerm=4, leader has term 4
   Follower:  [4][5][5]                     conflict at index 1

   Case 2 applies:
       leader's last term-4 entry is at index 0
       nextIndex = 1
       leader retries with prevLogIndex=0 → matches → ship 6,6,6 ✓
```

One round-trip instead of three.

---

## 4. Persistence (Lab 2C)

### 4.1 What happens after a crash?

Two ways to bring a crashed server back into the cluster:

| Strategy | What it costs | When it's used |
|---|---|---|
| **Fresh replacement** | full log/snapshot transfer (slow) | permanent hardware loss |
| **Reboot with state intact** | catch-up via tail of log (fast) | transient power outage, simultaneous failures |

We need **both**. The second requires writing critical state to disk.

### 4.2 What must persist? (Figure 2)

```
   ┌─────────────────────────────────────────────────────────┐
   │ PERSISTENT (must survive crash):                        │
   │   log[]         the commands themselves                 │
   │   currentTerm   monotonically increasing term           │
   │   votedFor      candidate I voted for this term         │
   ├─────────────────────────────────────────────────────────┤
   │ VOLATILE (safe to lose; rebuildable):                   │
   │   commitIndex   recomputed from majority on restart     │
   │   lastApplied   reset to 0; re-apply log to state mach. │
   │   nextIndex[]   reset to log length on leader election  │
   │   matchIndex[]  reset to 0 on leader election           │
   └─────────────────────────────────────────────────────────┘
```

### 4.3 Why each one?

- **`log[]`** — A server that participated in a commit *must* remember it after reboot, so any future leader (with majority overlap) is guaranteed to discover the committed entry.
- **`votedFor`** — Prevents a server from voting twice in the same term (vote, crash, reboot, vote again for someone else → two leaders).
- **`currentTerm`** — Guarantees terms only increase. Lets stale RPCs be detected.

### 4.4 Write timing

Save to non-volatile storage **before** sending any RPC or reply that depends on the new state. Otherwise a crash mid-RPC could leave the disk inconsistent with what peers observed.

### 4.5 Persistence is the bottleneck

```
   Hard disk write:    ~10 ms     → ~100 ops/sec
   SSD write:          ~0.1 ms    → ~10,000 ops/sec
   LAN RPC:            <1 ms      → not the bottleneck
```

Tricks to mitigate:

- **Batching** — group many log entries into one disk write.
- **Battery-backed RAM** — looks like RAM, survives power loss.
- **Group commit** — delay slightly to accumulate work.

### 4.6 Analogy — Black-box flight recorder

An airplane's flight data recorder doesn't store the *plane*; it stores enough about the plane's actions to reconstruct what happened. After a crash, you don't fly the same plane — you read the recorder and know what was true at the moment of impact. Raft's persisted state is the recorder: minimal data, but enough to restart with no committed entry lost.

---

## 5. Snapshots & Log Compaction (Lab 3B)

### 5.1 Why?

Logs grow forever. After a year, replaying the log to rebuild a 1 GB k/v table might take hours. Meanwhile the k/v table itself is only 1 GB.

> **Insight**: the executed prefix of the log is **already reflected** in the state machine's state. Once executed, we don't need the entries themselves — just the resulting state.

### 5.2 What can/can't be discarded?

```
   Log entries:    [ E1  E2  E3  E4  E5  E6  E7  E8 ]
                    ▲                  ▲       ▲
                    │                  │       │
                    └── all executed   │       └── not yet committed
                        and committed  │
                                       └── committed but not yet executed
                                           by the state machine

   Safely discardable: E1 .. E5    (executed)
   Must keep:         E6, E7, E8   (un-executed or un-committed)
```

### 5.3 The snapshot mechanism

```
   ┌───────────────────────────────────────────────────────┐
   │  SERVICE (k/v server)                                 │
   │   state = { k1: v9, k2: v3, k3: v7 }                  │
   │   │                                                   │
   │   │ "snapshot through log index N"                    │
   │   ▼                                                   │
   │   ┌──────────────┐                                    │
   │   │  snapshot    │  ── written to disk                │
   │   │  file        │     includes: state + N            │
   │   └──────────────┘                                    │
   ├───────────────────────────────────────────────────────┤
   │  RAFT                                                 │
   │   log: [ ... discarded ... | N+1 | N+2 | ... ]        │
   │                            ▲                          │
   │                            └── log now starts here    │
   └───────────────────────────────────────────────────────┘
```

Flow:
1. Service decides to snapshot (e.g., log grows past a threshold).
2. Service writes snapshot to disk, including `lastIncludedIndex = N`.
3. Service tells Raft: *"I'm snapshotted through index N"*.
4. Raft drops log entries `1..N`.

### 5.4 Restart procedure

```
   1. Service reads snapshot file from disk          → state restored
   2. Raft reads persisted log (entries N+1 onward)  → log restored
   3. Service tells Raft: lastApplied = N            → avoid re-applying
   4. Raft hands entries N+1, N+2, ... to service via applyCh
```

### 5.5 The follower-too-far-behind problem

What if a follower's log ends at index 50, but the leader has discarded everything before 200?

```
   Leader log:        [ ... gone ... | 200 | 201 | ... ]
   Follower log:      [ 1 .. 50 ]
                                       ▲ leader can't AppendEntries
                                         starting from anywhere ≤ 199
```

`AppendEntries` is useless here — the leader doesn't have the entries the follower needs. Solution: a new RPC, **`InstallSnapshot`**, which ships the snapshot file directly.

### 5.6 Analogy — Switching from a journal to a bank balance

A bank could record "Alice deposited $50, withdrew $20, deposited $30 …" forever. After enough transactions, it's faster to write down *"current balance: $1,247"* and discard the old entries. New auditors can read the balance file and continue from there. Snapshot = balance file. Log = transaction journal. The journal can be pruned because the balance file already captures its effect.

### 5.7 State vs. operation history — a duality

> Often you can choose: **store the state**, **store the history**, or **mix both**. Each option trades off space, restart time, and the ability to replay.

This duality shows up again in Spanner, Aurora, and most write-ahead-logged systems.

---

## 6. Linearizability — What "Correct" Means

### 6.1 The need for a definition

We've been hand-waving "correct". For Lab 3 we need a precise contract.

> **Linearizability**: an execution history is linearizable if you can find a **total order** of all operations such that:
> 1. The order **respects real time** (if op A finished before op B started, A comes before B).
> 2. Each **read returns the value of the most recent write** in that order.

It formalizes "the system behaves as if there were a single server."

### 6.2 How to read a history diagram

```
   |-Wx1-|                     ← write "1" to x; bar = start to end time
        |---Rx2---|            ← a read of x that returned 2
```

### 6.3 Example 1 — Linearizable

```
   |-Wx1-|       |-Wx2-|
       |---Rx2---|
         |-Rx1-|
```

Try the order: **Wx1 → Rx1 → Wx2 → Rx2**

- Time: Wx1 finishes before Wx2 starts ✓
- Value: Rx1 reads after Wx1 ✓, Rx2 reads after Wx2 ✓
- The reads sit inside their respective writes — overlapping ops can be ordered either way

**Linearizable** ✓

### 6.4 Example 2 — Not linearizable

```
   |-Wx1-|     |-Wx2-|
       |--Rx2--|
                       |-Rx1-|
```

Constraints:
- Wx1 → Wx2 (real time)
- Wx2 → Rx2 (value)
- Rx2 → Rx1 (real time)
- Rx1 → Wx2 (value, since Rx1 saw 1 not 2)

This forms a **cycle** → no valid total order → **not linearizable** ✗

### 6.5 Example 3 — Concurrent writes, single client read

```
   |-Wx0-|   |-Wx1-|
              |-Wx2-|
          |-Rx2-| |-Rx1-|
```

Order **Wx0 → Wx2 → Rx2 → Wx1 → Rx1** works. **Linearizable** ✓

> The service is **free to pick either order** for concurrent writes. Raft does this implicitly by whichever order it places them in the log.

### 6.6 Example 4 — Two clients must agree

```
   |-Wx0-|   |-Wx1-|
              |-Wx2-|
   C1:   |-Rx2-| |-Rx1-|
   C2:   |-Rx1-| |-Rx2-|
```

C1 sees Wx2 before Wx1. C2 sees Wx1 before Wx2. Constraints form a cycle → **not linearizable** ✗

> **Lesson**: all clients must observe the same ordering. This matters when there are replicas or caches.

### 6.7 Example 5 — Stale reads forbidden

```
   |-Wx1-|
           |-Wx2-|
                   |-Rx1-|
```

The read happens **after** Wx2 finished — by real time it must see ≥ value 2. Returning 1 is not linearizable.

**Linearizability forbids:**

- Split brain (two active leaders, divergent histories)
- Forgetting committed writes after reboot
- Reading from a lagging replica

### 6.8 Example 6 — Stale reply *is* OK in some cases

A client times out, retransmits. The server detected the duplicate and returned the cached old reply:

```
   C1:  |-Wx3-|              |-Wx4-|
   C2:           |-Rx3----------------|
```

C2's read **started** before Wx4 began. So returning 3 is consistent with order **Wx3 → Rx3 → Wx4**.

**Linearizable** ✓ — duplicate detection is safe.

> Useful reading: <https://www.anishathalye.com/2017/06/04/testing-distributed-systems-for-linearizability/>

### 6.9 Analogy — A perfect single-threaded clerk

Imagine **one** clerk handling every request, one at a time, in some order. Linearizability says: *"there exists some interleaving of the actual requests that this clerk could have produced."* The clerk has freedom to choose how to order **overlapping** requests, but cannot violate real time or invent stale data.

---

## 7. Duplicate RPC Detection (Lab 3)

### 7.1 The retry dilemma

When `Call()` returns false, the client can't tell:

```
   Possibility A:  request never reached server         → safe to resend
   Possibility B:  server executed, reply was lost      → resend = duplicate execution
                                                          (catastrophic for Put)
```

### 7.2 The mechanism

Tag each RPC with a unique identifier. The server keeps a table.

```
   Each client:    chooses 64-bit random clientID
                   numbers RPCs:  seq=1, 2, 3, ...
                   (only ONE outstanding RPC at a time)

   Each server:    duplicateTable[clientID] = { seq, lastReply }
```

On receiving an RPC:

```
   if seq <= duplicateTable[clientID].seq:
       return duplicateTable[clientID].lastReply    ← it's a duplicate
   else:
       Start() it through Raft → wait for applyCh →
       update duplicateTable → reply to client
```

### 7.3 Compact representation

Because each client has only **one outstanding RPC**:

- A new seq=N implies all previous seqs from that client are done.
- The server can store **one entry per client**, not one per RPC.

### 7.4 The hard questions

| Question | Answer |
|---|---|
| When can entries be garbage-collected? | When the client is known to be gone (heartbeat times out), or via explicit ACK of latest seq. |
| How does a new leader get the table? | All replicas update their tables as they execute. Information is already there. |
| How does a crashed server restore the table? | Replay log to rebuild it, or include the table in snapshots. |
| What if a duplicate arrives before the original executes? | Safe to `Start()` it again — when it appears on `applyCh`, check the table; skip execution if seen. |

### 7.5 Stale replies — and why they're OK

A scary scenario:

```
   C1                 C2
   put(x, 10)
                      get(x), reply 10 dropped
   put(x, 20)
                      retries get(x) → server returns cached 10 (not 20!)
```

Is returning stale 10 wrong? **No** — linearizable order is `Wx10 → Rx10 → Wx20`. C2's read *started* before `Wx20`, so 10 is a legitimate value to see.

---

## 8. Read-Only Optimizations (Leases)

### 8.1 Why even bother?

Most workloads are **read-heavy**. If every `Get()` goes through full log replication, throughput tanks.

### 8.2 The naive answer — commit the Get

> *"Just have the leader respond from its local table."*

**Unsafe.** Consider:

```
   S1 thinks it's leader, gets Get(k).
   But S1 actually lost leadership (network blip), didn't realize.
   S2 is the real leader, has processed Put(k, v_new).
   S1's table still has v_old.

   If S1 returns v_old → stale read → not linearizable → split brain.
```

The Figure 2 / Lab 3 fix: **commit Get into the log**. If S1 can't get majority acks, it can't reply → no stale data.

### 8.3 The lease optimization

```
   Lease period: 5 seconds (example)

   Every time leader gets majority AppendEntries replies →
       leader extends its "I can serve reads" lease by 5s

   New leader must WAIT for the previous lease to expire
       before processing any Put.
       (followers report timestamp of last AppendEntries in vote replies)

   During the lease: read locally from k/v table — no RPCs needed
```

### 8.4 Why it's safe

Two leaders never have overlapping leases — the new one waits out the old one. So during your lease you really are the only one serving requests → your table is fresh.

### 8.5 Why it requires clocks

All servers must agree on "what 5 seconds means". If the leader's clock runs slow, it might think the lease is alive when followers have moved on.

> **Lab note**: for 6.824 labs, commit `Get()`s — **don't** implement leases.

### 8.6 In practice

Real systems often accept **bounded staleness** (reads up to N ms old) for huge throughput gains. Cassandra, DynamoDB, and read replicas in SQL databases all live in this trade-off space.

### 8.7 Analogy — The doctor's "on duty" badge

A doctor holds a badge that says *"I am the on-duty surgeon until 3 PM."* While the badge is valid, the doctor signs prescriptions without phoning the hospital board. At 3 PM the badge expires; before issuing a new badge, the board confirms the previous holder is no longer acting. No two doctors are ever on duty at once — that's the lease.

---

## 9. Takeaway

> **Raft Part 2 builds out everything around the consensus core**: log reconciliation when leaders change, persistence so reboots don't lose history, snapshots to keep logs bounded, linearizability as the precise correctness contract, duplicate detection to handle retries safely, and lease-based reads as a real-world performance escape hatch.

### Mental checklist

| Concept | One-line summary |
|---|---|
| **State Machine Safety** | At most one command per log index, ever. |
| **Adopt-the-leader's-log** | Followers truncate and re-copy on conflict. |
| **Election Restriction** | Vote only for candidates "at least as up-to-date" (term first, length second). |
| **Fast log backup** | Skip by term, not by single index. |
| **Persistence** | log, currentTerm, votedFor must hit disk before any visible action. |
| **Snapshots** | Store state, drop the prefix of the log it reflects. |
| **InstallSnapshot** | The RPC for followers that fell behind a snapshot. |
| **Linearizability** | A total order honoring real-time, with each read seeing the most recent write. |
| **Duplicate detection** | One table entry per client (clientID + last seq + reply). |
| **Leases** | Bounded-time reads without log commits — fast, but needs clocks. |

The next set of papers (ZooKeeper, CRAQ, Spanner) build *on top of* Raft-like consensus — they assume this layer exists and explore what you can do with it.
