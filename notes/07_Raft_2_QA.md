# Lecture 7 — Raft (Part 2): FAQ

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Companion Q&A to [07_Raft_2.md](07_Raft_2.md) — covers log reconciliation, read-only optimizations, cluster membership changes, snapshots, and election subtleties.

---

## Table of Contents

1. [Where Raft Gets Used](#1-where-raft-gets-used)
2. [Read-Only Optimizations](#2-read-only-optimizations)
3. [Cluster Membership Changes](#3-cluster-membership-changes)
4. [Snapshots & Log Compaction](#4-snapshots--log-compaction)
5. [Election Rules & Edge Cases](#5-election-rules--edge-cases)
6. [Fast Log Backup](#6-fast-log-backup)
7. [Meta: Raft in Practice & Academia](#7-meta-raft-in-practice--academia)

---

## 1. Where Raft Gets Used

### Q: What can Raft replicate, besides a GFS-style master?

**A.** Raft replicates **any deterministic state machine**. Real-world uses:

| Application | What gets replicated |
|---|---|
| **Fault-tolerant k/v database** (Lab 3) | the key/value map |
| **Fault-tolerant MapReduce master** | task assignments, worker liveness |
| **Distributed lock service** (Chubby, ZooKeeper) | the lock table |
| **Service discovery / config** (etcd, Consul) | the registry |

> **Analogy** — Raft is the **stenographer** in a courtroom. Whatever the judge says, the stenographer writes down word-for-word in a numbered ledger. Multiple stenographers in different rooms keep identical ledgers. If one stenographer faints, another picks up the transcript. The *content* doesn't matter — Raft just guarantees that every copy of the ledger is identical.

---

## 2. Read-Only Optimizations

A read-only operation (`Get("k1")`) doesn't change state. It would be wasteful to run it through full log replication. But cutting corners is dangerous. This section covers the safe shortcuts.

### 2.1 The hidden danger of fast reads

> If you just answer `Get("k1")` from the leader's local table, you might be a **stale leader** who got partitioned and doesn't know about a new leader's writes. You'd return outdated data.

```
   Old leader (term 5, partitioned):       Real leader (term 6):
        k1 = "old"                              k1 = "new"
             │                                       │
             ▼                                       ▼
        Client asks ──── stale read! ────  Client asks ──── correct read
```

### 2.2 The no-op trick

### Q: When Raft receives a read request, does it still commit a no-op?

**A.** Section 8 of the paper mentions **two approaches**:

| Approach | Cost per read | Notes |
|---|---|---|
| **Heartbeat-then-read** | one round of AppendEntries | confirms leadership *right now* |
| **Lease** | none | requires bounded clock skew |

Real systems usually prefer **leases** — less communication.

The no-op trick is different: the leader commits a single no-op **at the very beginning of its term**, not per read.

### Q: Section 8 says no log writes are needed for reads, then introduces no-op commits. Contradiction?

**A.** No. The no-op is **once per term** (right after election), not per read. After it commits, the leader knows everything earlier in its log is committed, and can serve reads from its local table safely.

### Q: Why does the new leader need to commit a no-op at all?

**A.** Look at Figure 8 of the paper:

```
   Imagine S1 just won the election for term 4.
   Its log ends with:  [ ... | term3 | term2 ]
                                       ↑
                              Is this committed?
```

S1 doesn't know! If it crashes immediately, S5 might win the next election and **erase that entry** (per the Election Restriction).

**The fix**: S1 commits a fresh entry **in its own term (4)**. Two outcomes:

| Scenario | What happens |
|---|---|
| S1 succeeds in committing a term-4 entry | Then by the Log Matching Property, **all preceding entries** in S1's log are also committed. S1 can safely answer reads. |
| S1 crashes first | Doesn't matter — S1 never answered a read with possibly-stale data. |

> **Analogy** — A new CEO walks into the office. The desk has unsigned contracts from her predecessor. Are they binding? She doesn't know. So she signs one new contract of her own — once it's countersigned by the board (majority), she knows the *entire desk* is now legally locked in.

### Q: How does the heartbeat-lease scheme work, and why does it need bounded clock skew?

**A.** Sketch: every `AppendEntries` implicitly promises *"no other leader will be elected for the next 100 ms"*. If the leader gets a majority of acks, it can serve reads locally for 100 ms.

```
   t=0:   leader sends AppendEntries to all → majority acks
          leader knows: "I'm safe to read until t=100"
   t=10:  Get("k1") arrives → answer from local table, no RPC needed
   t=50:  Get("k2") arrives → answer locally, no RPC needed
   t=100: lease expires → must re-confirm with another AppendEntries
```

**Why clocks matter**: all servers must agree on what *"100 ms"* means. If the leader's clock runs slow, it might think the lease is still valid when followers have already started a new election.

---

## 3. Cluster Membership Changes

Adding or removing servers from a Raft cluster (e.g. retiring an old machine) is tricky because the **definition of "majority" changes mid-operation**.

### 3.1 The hazard

Suppose we naively switch from `C_old = {S1, S2, S3}` to `C_new = {S3, S4, S5}`:

```
   At the moment of switch:
       S1, S2  still think  majority = 2 of {S1,S2,S3}    → they could elect a leader
       S4, S5  already think majority = 2 of {S3,S4,S5}   → they could elect a different leader

   ✗  TWO LEADERS — split brain.
```

### 3.2 Joint consensus — the safe overlap

Raft's fix is a transitional configuration called **C_{old,new}** that requires a majority from **both** old and new:

```
   Phase 1:  C_old           majority = 2 of {S1,S2,S3}
                              │
                              ▼ leader appends C_{old,new} entry
   Phase 2:  C_{old,new}     majority = 2 of OLD  AND  2 of NEW
                              │ (overlapping requirement → impossible to have
                              │  two competing leaders)
                              ▼ leader appends C_new entry
   Phase 3:  C_new           majority = 2 of {S3,S4,S5}
```

### Q: What are C_old and C_new exactly? The leader sets?

**A.** They are the **set of servers** in each configuration (network addresses / identities), not just the leader.

### Q: What does it mean to be in C_{old,new}? Doesn't a server use either the old or new network?

**A.** During joint consensus, the leader needs a majority **from C_old AND a majority from C_new**, separately. There's no "merged" network — both quorums are independently required.

There's no risk of disagreement: once C_{old,new} is committed, *any future leader* in either config will see that log entry (by majority overlap).

### Q: I'm confused about Figure 11. Why does the leader step down after committing C_new?

**A.** If the leader isn't in C_new, it shouldn't keep leading after the change completes.

```
   C_old   = {S1, S2, S3}   (S1 is leader)
   C_new   = {S4, S5, S6}   (S1 is NOT in C_new)

   After S1 commits C_new:
        S1 is no longer part of the cluster.
        S1 steps down. One of S4/S5/S6 becomes the new leader.
```

> **Analogy** — A retiring CEO finalizes a merger transferring the company to a new board. The moment the merger paperwork is signed (`C_new` committed), she packs her office. She doesn't hang around the boardroom giving orders.

### Q: Why not just stop accepting requests, manually change configs, restart, and resume?

**A.** Two reasons:

1. **Partial failures** during the change must be safe. Not all servers may receive the "stop" command at the same time.
2. **Availability** — joint consensus lets the cluster keep serving clients during the change (with a brief slowdown).

### Q: Why not require removed servers to shut down immediately?

**A.** The paper's protocol commits `C_new` only to servers in `C_new`. Servers being removed never learn the change happened. So they keep timing out and starting elections, disrupting the new cluster.

The paper notes: *"after C_new is committed, servers not in C_new can be shut down"* — but it leaves that to operators rather than automating it.

### Q: Is requiring both majorities slow?

**A.** In the common case (no failures), the leader gets both majorities in **one RPC round-trip** — a few ms. Config changes happen rarely (months apart), so the overhead is negligible.

The dual-majority requirement is **not for speed** — it's for **correctness during leader failure mid-transition**.

### Q: What are non-voting servers for? To speed up replication?

**A.** No — replication speed is unchanged. The purpose is **catch-up without blocking**:

```
   Without non-voting phase:
      leader proposes C_{old,new} → can't commit until new servers
      catch up on the entire log → cluster stalls.

   With non-voting phase:
      new servers receive log entries but don't count for majority.
      They catch up quietly in the background.
      Once nearly caught up → propose C_{old,new} → commits fast.
```

### Q: When does joint consensus actually begin and end?

**A.**

| Event | State |
|---|---|
| Leader appends C_{old,new} to its log | Joint consensus **in progress** for this leader |
| Leader commits C_{old,new} | Joint consensus is **guaranteed to complete** |
| Leader crashes before committing C_{old,new} | New leader may not see it → joint consensus **ends early** |
| Some leader commits C_new | Joint consensus **done**, old config retired |

### Q: Can an uncommitted C_{old,new} entry be overwritten?

**A.** Yes. Any uncommitted log entry can be overwritten by a future leader. That's why the protocol carefully relies on **committed** C_{old,new} for its safety guarantees.

### Q: Why deny RequestVotes within a heartbeat interval, instead of just checking if the candidate is in the current config?

**A.** Cleaner separation. During joint consensus, a server in C_new might legitimately be leader while some C_old servers haven't yet learned about C_{old,new}. Those C_old servers don't yet know the C_new server is "in their config", so a config-check would reject a valid leader. The heartbeat-recency check sidesteps this entirely.

---

## 4. Snapshots & Log Compaction

### 4.1 Why snapshot?

Logs grow forever. A k/v server that's processed 1 billion `Put`s shouldn't keep all 1 billion log entries. Periodically, the server **snapshots** its current state and discards the prefix of the log it covers.

```
   Before snapshot:
      Log: [ entry 1 | entry 2 | ... | entry 9,999 | entry 10,000 ]
      State machine: <reflects all 10,000 entries>

   Snapshot at index 9,000:
      Snapshot: <state after applying entries 1..9000>
      Log:      [ entry 9,001 | ... | entry 10,000 ]   ← keep only the tail
```

### Q: What state is in the snapshot — Raft's, or the service's?

**A.** **The service's.** If you're building a k/v server, the snapshot contains the **k/v table**, not Raft's internal log structures.

### Q: If a snapshot covers a prefix of my log, deleting that prefix loses operations, right?

**A.** No — the snapshot **already reflects** those operations. The state machine has applied them. The log entries themselves are just history at that point and can be safely discarded.

> **Analogy** — A bank statement summarizes every transaction up to date X. Once you have the statement, you can throw away every individual receipt up to date X. The statement *encodes* them.

### Q: When the state is as big as the log (each op is a unique insert), is snapshotting still worth it?

**A.** Maybe — for two reasons:

- **Faster restart**: replaying a million log entries is slower than loading one big snapshot file.
- **Different access pattern**: a snapshot can be organized (e.g. as a sorted B-tree) for efficient reads, whereas the log is append-only.

But typically the log is *much* bigger than the state.

### Q: Doesn't `InstallSnapshot` cost a lot of bandwidth?

**A.** Yes, if the state is large (a multi-GB database). Mitigations:

- Keep more log entries on the leader, so most lagging followers catch up via `AppendEntries`.
- Transfer only **diffs** rather than the full snapshot (advanced; not in the paper).

### Q: Can writing a snapshot take longer than the election timeout?

**A.** Yes, this is a real problem. A 1 GB snapshot at 100 MB/s = 10 seconds to write, way longer than a 200 ms election timeout.

**Solution: copy-on-write fork.**

### Q: How does copy-on-write help?

**A.** The server `fork()`s. The OS marks all pages copy-on-write rather than physically copying them — fast. The child process writes the snapshot to disk slowly, while the parent keeps serving requests.

```
   Parent process                      Child process (after fork)
        │                                       │
        ├── keeps applying ops                  ├── reads memory snapshot
        ├── pages diverge as needed             ├── writes to disk slowly
        ▼                                       ▼
   handles clients                       saves snapshot
```

Only pages that the parent **modifies** during the snapshot get physically copied. Usually that's a small fraction.

### Q: Best compression for snapshots?

**A.** Depends on the data. Images → JPEG. Text → gzip. If consecutive snapshots share a lot, a **versioned tree** (sharing nodes across versions) saves space.

### 4.2 InstallSnapshot RPC

### Q: Why does `InstallSnapshot` need an `offset` field?

**A.** Snapshots can be huge → sent in **multiple chunks**, one chunk per RPC. The offset says where this chunk goes in the assembled file.

```
   InstallSnapshot RPC #1: offset=0,        data=[bytes 0..1023]
   InstallSnapshot RPC #2: offset=1024,     data=[bytes 1024..2047]
   InstallSnapshot RPC #3: offset=2048,     data=[bytes 2048..2999]  (last)
```

### Q: When does a follower receive a snapshot that's a **prefix** of its own log?

**A.** When the network reorders messages. Leader sends snapshot for index 100, then for index 110, but 110 arrives first. The 100-snapshot is now stale.

### Q: Does Figure 13 handle stale snapshots correctly?

**A.** **Not explicitly.** For Lab 3B, you must add:

```
   if lastIncludedIndex < followers's commitIndex:
       drop this RPC, return immediately
```

The paper omits this; we generalize.

### Q: If my snapshot is ahead of an incoming `InstallSnapshot`, is it safe to ignore it?

**A.** Yes — the recipient is already past that point. The newer snapshot subsumes the older one.

### Q: Must `InstallSnapshot` be atomic?

**A.** Yes — the installation must complete fully or not at all. Re-sending is harmless (idempotent).

### Q: How does the leader decide to send a snapshot vs. log entries?

**A.** Follower rejects an `AppendEntries` at index `i`. Leader looks up its log:

```
   if log contains index i:
       back up nextIndex[follower], retry AppendEntries
   else (entry was discarded by snapshotting):
       send InstallSnapshot instead
```

### Q: In real deployments, how often are snapshots sent over the wire?

**A.** Rarely — operators tune Raft to keep enough log on the leader to catch up most slow followers. `InstallSnapshot` is the last resort.

---

## 5. Election Rules & Edge Cases

### Q: Does adding an entry to the log count as "executing" it?

**A.** **No.** Three distinct steps:

```
   1. APPEND   — leader puts entry in its log         (not yet visible to service)
   2. COMMIT   — majority has the entry in their log  (durable, but state machine hasn't applied it)
   3. EXECUTE  — Raft hands committed entry to the    (service applies; client sees effect)
                 service via applyCh
```

In Lab 3, "execute" means your k/v code does `table[key] = value`.

### Q: A server disregards RequestVote if it thinks a leader exists. But if it thinks no leader exists, it starts its own election. When would it ever vote for someone else?

**A.** Subtle timing question. Walk through:

```
   Heartbeat interval: 10 ms.
   Leader sends heartbeats at t=10, 20, 30.

   Server S1: misses heartbeat at t=30
              its random election timer expires at t=35
              S1 → candidate, sends RequestVote

   Server S2: heard the heartbeat at t=30
              S2 knows the leader was alive at t=30
              S2's election timer won't expire until at least t=40
              (next expected heartbeat is at t=40)

   At t=35: S2 receives S1's RequestVote.
            S2 ignores it — only 5 ms since last heartbeat, leader probably still fine.
```

The case the paper has in mind: **S1 missed a heartbeat, S2 didn't.** S1 wants to start an election, but S2 (correctly) refuses to vote because *from S2's perspective*, the leader is still alive.

> **Analogy** — Two security guards. Guard A watches the back door and just saw the boss walk by 2 seconds ago. Guard B watches the front door and hasn't seen the boss in 30 seconds. Guard B shouts *"the boss is missing, let's elect a new one!"* Guard A says *"no way, I just saw her, calm down."*

---

## 6. Fast Log Backup

When a new leader's log doesn't match a follower's, the leader walks back `nextIndex` to find the divergence point. Naively this is one entry per RPC — too slow for big mismatches.

### Q: I'm confused by the "if leader knows about the conflicting term" branch.

**A.** Paper's scheme (end of §5.3):

```
   Follower rejects AppendEntries.
   Follower replies with:
      conflictTerm = term of the conflicting entry
      conflictIndex = index of FIRST entry with that term

   Leader checks its own log:
      ┌─ if leader HAS entries for conflictTerm
      │     → set nextIndex = index of leader's LAST entry for conflictTerm
      │     (leader has authority over this term, fast-forwards to the right spot)
      │
      └─ if leader has NO entries for conflictTerm
            → set nextIndex = conflictIndex
            (the follower's entire span of this term is wrong, skip past it)
```

The paper says "*move nextIndex to the first entry for the conflicting term*". But that doesn't make sense if the **leader has no entries for that term at all** — there's nothing to point to. The split into two cases handles this.

### Q: What if I just halve `nextIndex` each retry (exponential backoff)?

**A.** Tradeoff:

```
   Linear (-1 each retry):  many round-trips, exact landing point
   Exponential (×½):        few round-trips, may overshoot by 2×
   Per-term jump (paper):   minimal RPCs, precise — preferred
```

Exponential is a fine "good-enough" middle ground; the per-term approach is best.

---

## 7. Meta: Raft in Practice & Academia

### Q: How does your teaching experience compare to §9.1's claims (Raft easier than Paxos)?

**A.** Course staff are happy with both labs over the years. The Raft labs are **more ambitious** (leader, persistence, snapshots — Paxos labs had none of these), so a direct side-by-side wasn't run.

### Q: How big is Raft's academic impact?

**A.** Significant. The Raft paper is **the best modern explainer** of replicated state machines and has inspired many real implementations (etcd, Consul, TiKV, CockroachDB, …).

### Q: Suggested improvements over the original Raft?

**A.** See: <https://www.cl.cam.ac.uk/~ms705/pub/papers/2015-osr-raft.pdf>

---

## Cheat Sheet

| Theme | Key takeaway |
|---|---|
| **Reads** | Either round-trip for a heartbeat, or use a lease (clock-bounded). New leader commits a no-op first to fix Figure-8 anomalies. |
| **Config changes** | Two-phase `C_old → C_{old,new} → C_new`. Dual majority required during joint phase. |
| **Non-voting members** | Catch up logs before counting for majority — preserves availability during expansion. |
| **Snapshots** | Service-level state, not Raft state. Use copy-on-write fork to avoid blocking. Send via chunked `InstallSnapshot`. |
| **Stale RPCs** | Network reordering means you must drop snapshots/AppendEntries that arrive late. |
| **Fast log backup** | Skip by term, not by single index — handle both "leader knows this term" and "leader doesn't" cases. |
| **Election sanity** | Don't vote for a candidate if you've heard a heartbeat recently — your view of leader health is better than theirs. |
