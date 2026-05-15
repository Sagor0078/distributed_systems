# Lecture 6 — Raft (Part 1): Elections & Log Handling

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Today: Raft elections and log handling (Lab 2A, 2B)
> Next lecture: Raft persistence, client behavior, snapshots (Lab 2C, Lab 3)

---

## Table of Contents

1. [The Single Point of Failure Problem](#1-the-single-point-of-failure-problem)
2. [Split Brain — Why Replication Alone Isn't Enough](#2-split-brain--why-replication-alone-isnt-enough)
3. [The Big Insight: Majority Vote](#3-the-big-insight-majority-vote)
4. [Raft Overview](#4-raft-overview)
5. [Leader Election (Lab 2A)](#5-leader-election-lab-2a)
6. [Takeaway](#6-takeaway)

---

## 1. The Single Point of Failure Problem

Every fault-tolerant system we've studied so far quietly cheats:

| System | Replicates | …but trusts ONE thing |
|---|---|---|
| **MapReduce** | the computation | a single **master** to schedule and detect failures |
| **GFS** | the data | a single **master** to pick primaries |
| **VMware FT** | the entire VM | a single **test-and-set server** to pick the primary |

> **Pattern** — Replicate the *workers*, but route critical decisions through *one* trusted entity.

### Why this is appealing

A single decider can never disagree with itself. **No split brain.** Decisions are simple, consistent, and unambiguous.

### Why this is a problem

That one entity is now a **single point of failure**. The whole system inherits the availability of its weakest link — exactly what replication was supposed to fix.

> **The question driving Raft**: Can we make the *decision-maker itself* fault-tolerant?

---

## 2. Split Brain — Why Replication Alone Isn't Enough

### 2.1 A concrete scenario

Imagine a replicated test-and-set service (a lock). Two replicas `S1`, `S2`. Two clients `C1`, `C2`. Exactly one client should "win" the lock — get the reply `0` (previous state was unlocked, now locked).

```
   ┌────┐         ┌────┐
   │ C1 │ ──────▶ │ S1 │
   └────┘         └────┘
                    ╳        ← network partition between S1 and S2
   ┌────┐         ┌────┐
   │ C2 │ ──────▶ │ S2 │
   └────┘         └────┘
```

`C1` can talk to `S1` but not `S2`. From `C1`'s point of view, `S2` is silent. What should the system do?

### 2.2 The trap

| If we tell `C1` "go ahead with just `S1`"… | If we tell `C1` "wait for `S2`"… |
|---|---|
| Great when `S2` really crashed. | Great when `S2` really crashed but the network is fine. |
| **Disaster** when `S2` is alive serving `C2`: both clients get the lock. Split brain. | **Disaster** when `S2` crashed: system halts even though `S1` is healthy. No fault tolerance. |

### 2.3 The root cause

> A computer cannot tell **"server crashed"** apart from **"network broken"**.
> Both look identical: no response.

This was considered unsolvable for decades. Proposed workarounds all reintroduce a single point of failure:

- A **human** to manually cut over
- A **perfectly reliable** server (FT's test-and-set box)
- A **perfectly reliable** network (so silence = death)

We want better.

### 2.4 Analogy — The two-doctor problem

Two doctors share responsibility for one patient. The phones go down. Each doctor must decide: *"Should I prescribe medication, assuming my colleague is unreachable?"*

- If both prescribe → **overdose** (split brain).
- If neither prescribes → patient gets no treatment.

Neither doctor alone can safely choose. The solution we'll see is to add a **third doctor** — and require any two of them to agree before acting.

---

## 3. The Big Insight: Majority Vote

### 3.1 The rule

> Use an **odd number** of servers (3, 5, 7…). **Nothing happens** unless a **majority** agrees.

With 3 servers, any action needs 2 yes-votes. With 5, you need 3. With `2f+1` servers, you tolerate `f` failures.

### 3.2 Why it works

**Any two majorities must overlap.** It is geometrically impossible for two disjoint groups in a 3-node cluster to both have 2 members — the pigeonhole won't fit.

```
   3 servers, partitioned:    ┌────┐  ╳  ┌────┬────┐
                              │ S1 │  ╳  │ S2 │ S3 │
                              └────┘  ╳  └────┴────┘
                              minority   MAJORITY (can act)
```

- At most **one** side of any partition has a majority → at most one side can act → no split brain.
- **Majority is computed against ALL servers, not just live ones.** With 3 servers and 2 dead, the 1 survivor cannot proceed — that's the price of safety.

### 3.3 Why overlap is the real magic

Successive majorities are guaranteed to share at least one server.

```
   First majority (term 1):   {A, B}    ─┐
                                          │── overlap at B
   Second majority (term 2):  {B, C}    ─┘

   B remembers what term 1 decided → can convey that decision into term 2.
```

This is the secret weapon for Raft: **a new leader can always learn what the previous regime committed**, because some server in the new majority was also in the old one.

### 3.4 Analogy — The jury

A 12-person jury must reach a majority verdict. If 5 jurors get stuck in traffic, the remaining 7 can still deliberate and decide. If only 5 show up, no verdict — by design. The system trades **availability under heavy loss** for **never two contradicting verdicts**.

### 3.5 History

Two majority-based replication schemes were invented around 1990:

- **Paxos** (Leslie Lamport)
- **Viewstamped Replication** (Brian Oki, Barbara Liskov)

In the last 15 years they've gone from theory to running the internet (Chubby, ZooKeeper, etcd, Spanner). **Raft** (2014) is a cleaner re-presentation designed to be teachable.

---

## 4. Raft Overview

### 4.1 The big picture

Raft is a **library** linked into each replica. It does one thing: **agree on the same sequence of commands**, in the same order, across all replicas.

```
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Client  │ │ Client  │ │ Client  │
   └────┬────┘ └────┬────┘ └────┬────┘
        └───────────┼───────────┘
                    ▼
   ┌────────────────────────────────────┐
   │  REPLICA  (one of several)         │
   │  ┌──────────────────────────────┐  │
   │  │ k/v service                  │  │  ← application layer
   │  │ (state machine: a map)       │  │     (Lab 3)
   │  └──────────┬───────────────────┘  │
   │             │ Start(cmd)           │
   │             ▼                      │
   │  ┌──────────────────────────────┐  │
   │  │ Raft library                 │  │  ← consensus layer
   │  │ (log of commands)            │  │     (Lab 2)
   │  └──────────────────────────────┘  │
   └────────────────────────────────────┘
              │ AppendEntries RPC
              ▼ (to other replicas' Raft layers)
```

### 4.2 Lifecycle of one client command

```
   Client          Leader            Follower 1        Follower 2
     │               │                    │                  │
     │── Put(x,1) ──▶│                    │                  │
     │               │ 1. append to log   │                  │
     │               │── AppendEntries ──▶│                  │
     │               │── AppendEntries ────────────────────▶ │
     │               │                    │ 2. append to log │
     │               │ ◀──── ok ──────────│                  │
     │               │ ◀──── ok ────────────────────────────│
     │               │ 3. majority? YES → COMMIT             │
     │               │ 4. apply to state machine             │
     │ ◀── reply ────│                                       │
     │               │── next AppendEntries (commitIdx++) ──▶│
     │               │                    │ 5. apply         │
```

> **"Committed"** = a majority has the entry in their log.
> Once committed, the entry survives any future leader change — that's the guarantee.

### 4.3 Why a log at all?

The k/v state machine already holds the values. Why bother with a log?

- **Ordering** — replicas execute in identical order if they apply identical logs.
- **Tentative storage** — commands sit in the log *uncommitted* until majority confirms.
- **Re-sending** — leader needs to retry to a slow follower → must remember the command.
- **Persistence** — after reboot, replay the log to rebuild state.
- **Leader sync** — new leader uses log overlap to bring followers in line.

### 4.4 Are all logs identical?

**No** — at least, not at every instant.

- Some followers lag.
- During leadership churn, logs can briefly **diverge**.
- But: **committed entries are never lost**, and logs **converge** over time.

### 4.5 Lab 2 — The Raft API

```go
// Called by k/v server's Put/Get handler:
rf.Start(command) → (index, term, isLeader)
   // adds command to leader's log, kicks off AppendEntries
   // returns immediately — does NOT wait for commit
   // isLeader=false → caller should redirect to another server

// Raft pushes committed entries to the service via applyCh:
type ApplyMsg struct {
   Index   int
   Command interface{}
}
```

The k/v layer's `Put`/`Get` handler **waits on `applyCh`** for its index to appear → then replies to the client.

### 4.6 The two halves of Raft

> All of Raft boils down to two intertwined problems:
> 1. **Electing a leader** (this lecture)
> 2. **Keeping logs identical** despite failures (next lecture)

---

## 5. Leader Election (Lab 2A)

### 5.1 Why have a leader?

To force a single ordering of commands. Without a leader, two clients could submit `Put(x,1)` and `Put(x,2)` and different replicas could pick different orderings.

> *Note*: Paxos has no permanent leader and runs a mini-agreement per command — it works, but it's harder to think about. Raft optimizes for clarity.

### 5.2 Terms — numbered reigns

Every leader is tagged with a monotonically increasing integer called the **term**.

```
   Term: 1            2          3           4
        ┌──────┐    ┌────┐    ┌──────┐    ┌──────┐
        │ S1   │    │ S2 │    │ (no  │    │ S3   │
        │ leads│    │leads│   │leader│    │ leads│
        └──────┘    └────┘    └──────┘    └──────┘
                              ↑ split vote, election failed
```

Rules:

- **At most one leader per term** (might be zero).
- Higher term **trumps** lower term — old leader's messages are rejected.
- Term numbers stamp every RPC and let stale servers detect their own staleness.

### 5.3 Analogy — The king is dead, long live the king

Imagine medieval kingdoms with no clear succession rule. Each new monarch declares a **new regnal year** (term). If a king goes silent (hunting accident? plague?), the dukes (peers) call a council and pick a new king. The new king issues decrees stamped with the new year. Anyone receiving a decree from the old year ignores it — even if the "old" king is somehow still alive in a remote castle, his orders carry no weight.

### 5.4 When does an election start?

A peer becomes a candidate when it hasn't heard from the leader for an **election timeout** (~150–300 ms).

```
   leader sends heartbeats every ~50 ms
   ────●────●────●────●─── X ──────────────────────
                              ↑ peer's timer expires
                              ↑ peer increments currentTerm
                              ↑ peer votes for itself
                              ↑ peer sends RequestVote RPCs
```

> Sometimes the leader is fine and just slow — an unnecessary election starts. That's **wasteful but safe**.

### 5.5 How do we ensure at most one leader per term?

Two ironclad rules (Figure 2 in the paper):

1. **A candidate needs a majority of votes** to become leader.
2. **Each server casts at most one vote per term.**
   - A candidate votes for itself.
   - A non-candidate votes for the first qualifying candidate that asks (with extra rules about log freshness — covered next lecture).

Since two majorities must intersect, and the intersecting server can only vote once per term → **only one candidate can win term N**.

### 5.6 How do peers learn there's a new leader?

```
   Candidate     S2          S3          S4          S5
      │           │           │           │           │
      │── RequestVote(term=7) ──────────────────────▶ │
      │           │           │           │           │
      │ ◀── yes ──┤ ◀── yes ──┤           │           │
      │ (got majority)                                │
      │── AppendEntries (heartbeat, term=7) ────────▶ │
      │           │           │           │           │
      │       "ah, term 7 exists, I follow him now"   │
```

The heartbeats themselves are the announcement. Any peer seeing a heartbeat with a higher term updates its `currentTerm` and reverts to follower. New elections are suppressed as long as heartbeats keep arriving.

### 5.7 Why elections can fail

```
   Failure mode                  Cause                              Outcome
   ─────────────────────────────────────────────────────────────────────────
   ① Not enough servers up    less than a majority reachable    no quorum
   ② Split vote               two candidates with same timeout  no majority
                              both got their own vote + a few
                              others, neither hit the threshold
```

Either way: **timer expires again → new term → new election.**

### 5.8 Avoiding split votes — randomization

If every peer had the same timeout, partitioned-then-recovered clusters would re-trigger split votes forever. Raft breaks the symmetry with **randomized election timeouts**:

```
   Each peer picks: timeout = base + random(0, jitter)

   S1: ──────●─────────                  ← fires first
   S2: ─────────────●──                  ← still waiting
   S3: ────────●───────                  ← still waiting

   S1 becomes candidate, wins, sends heartbeats.
   S2 and S3 see heartbeat → reset their timers → never become candidates.
```

> **Pattern**: randomized backoff is a classic tool — Ethernet does it, BitTorrent does it, retry libraries do it. Whenever many independent agents might collide, give them random delays.

### 5.9 How to pick the election timeout

A balancing act:

- **Lower bound** — must be ≥ a few heartbeat intervals. If it's too tight, a dropped heartbeat triggers a needless election. Wastes time and may cause churn.
- **Random spread** — wide enough that the winner has time to send a heartbeat before runner-up's timer fires. Too narrow → repeated split votes.
- **Upper bound** — small enough that detecting a dead leader doesn't take forever. The MIT tester demands election within 5 seconds.

Typical numbers: heartbeat 50 ms, election timeout 150–300 ms.

### 5.10 The "zombie leader" worry

> What if the old leader doesn't notice it's been deposed?

Maybe it was in a network partition. Maybe it just missed the election messages. It still thinks it's leader and tries to commit new entries.

**It can't do damage.** Here's why:

```
   Old leader (term 5)          Majority of cluster (now term 6)
        │
        │── AppendEntries(term=5, ...) ──▶ rejected: "your term is stale"
        │                                  (each healthy peer is at term ≥ 6)
        │
        │ ◀── ok from 1 lonely follower    (one isolated peer in old partition)
        │
        │ 1 < majority → CANNOT COMMIT.
        │ Old leader can never tell the client "done."
```

- A new leader exists only if a **majority** moved to a higher term.
- The old leader can no longer get a majority of `AppendEntries` acks.
- Therefore the old leader can't commit anything → no split brain on **committed** state.
- A minority of stragglers may still accept the old leader's entries (logs diverge briefly), but the next lecture shows how the new leader cleans this up.

---

## 6. Takeaway

> **Raft solves the impossibility of distinguishing "crashed" from "partitioned" by requiring a *majority* for every decision.** Majorities intersect, so successive leaders cannot disagree about committed history. A single leader per term simplifies the protocol; randomized election timeouts make leader election converge in practice.

### Mental checklist

| Concept | One-line summary |
|---|---|
| **Majority quorum** | `2f+1` servers tolerate `f` failures; any two majorities share a server |
| **Term** | Monotonic integer; at most one leader per term |
| **Election timeout** | Triggers a new election; randomized to avoid split votes |
| **Heartbeat** | Leader's `AppendEntries` even when no new commands — suppresses elections |
| **Commit** | Entry replicated on a majority's logs → durable, will survive leader change |
| **Log** | Tentative storage, ordering, and replay buffer — the heart of consensus |
| **Zombie leader** | Stale leader can't get a majority → can't commit → harmless |

Next lecture: how the new leader reconciles divergent logs, persistence, and snapshots.
