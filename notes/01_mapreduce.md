# Lecture 1 — Introduction & MapReduce

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Lecturer: Robert Morris · Course site: [pdos.csail.mit.edu/6.824](http://pdos.csail.mit.edu/6.824)

---

## Table of Contents

1. [What is a Distributed System?](#1-what-is-a-distributed-system)
2. [Core Topics](#2-core-topics)
3. [Case Study: MapReduce](#3-case-study-mapreduce)
4. [Course Logistics](#4-course-logistics)

---

## 1. What is a Distributed System?

> **Definition** — Multiple cooperating computers presenting themselves as a single service.

Examples: storage for big websites, MapReduce, peer-to-peer sharing. Most critical internet infrastructure today is distributed.

### Why build one?

| Reason | What it buys you |
|---|---|
| **Parallelism** | More capacity by adding machines |
| **Replication** | Survive failures of individual nodes |
| **Proximity** | Place compute near users/sensors/devices |
| **Isolation** | Security boundaries between components |

### The cost

- Many concurrent parts → complex interactions
- Must cope with **partial failure** (some nodes alive, some dead, some lying)
- The theoretical speedup is hard to actually realize

### Analogy: a restaurant chain vs. one big restaurant

A single huge restaurant has one kitchen. If the chef is sick, you close. To serve more diners, you hire a faster chef.

A **restaurant chain** has many kitchens cooperating: one branch can close and customers go elsewhere; you scale by opening branches; branches sit physically close to neighborhoods.

That's the appeal — and the headache — of distributed systems. Branches must agree on the menu, prices, and gift-card balances. Coordination is where the hard problems live.

---

## 2. Core Topics

The course is about **infrastructure** that hides distribution from applications, organized around three primitives:

```
   ┌───────────┐   ┌────────────────┐   ┌─────────────┐
   │  STORAGE  │   │ COMMUNICATION  │   │ COMPUTATION │
   └───────────┘   └────────────────┘   └─────────────┘
```

Four recurring themes:

### 2.1 Implementation
RPC, threads, concurrency control. This is what the labs exercise.

### 2.2 Performance — *scalable throughput*

The dream:

```
   1 server   →   1x throughput
   N servers  →   Nx throughput   (linear scaling)
```

Reality bites as **N** grows:

- **Load imbalance** — some workers get heavy chunks
- **Stragglers** — slowest-of-N dictates job time
- **Non-parallelizable code** — startup, coordination, final aggregation
- **Shared bottlenecks** — network root, lock contention

Some problems *can't* be scaled away (latency for one user, hotspot writes). Those need redesign, not more hardware.

### 2.3 Fault Tolerance

With 1000s of servers, *something* is always broken.

| Property | Meaning |
|---|---|
| **Availability** | App keeps making progress despite failures |
| **Recoverability** | App returns to correct state after failures are fixed |

**Big idea:** replicated servers — if one crashes, others carry on.

### 2.4 Consistency

> *"Get(k) should return the value from the most recent Put(k, v)."*

Sounds simple. It isn't. Replicas drift; clients crash mid-update; servers crash after executing but before replying; network partitions cause split-brain.

**Consistency and performance are enemies:**

```
   STRONG CONSISTENCY  ◀─────────────────────▶  WEAK CONSISTENCY
   (more coordination)                          (less coordination)
   slower, simpler API                          faster, harder API
```

Every system picks a point on this spectrum.

---

## 3. Case Study: MapReduce

### 3.1 The Problem

- Multi-**hour** computations on multi-**terabyte** datasets
- Examples: build a search index, sort the web, analyze link structure
- Requires 1000s of machines
- **App authors are not distributed-systems experts**

### 3.2 The Idea

The programmer writes just two functions:

```
   Map(key, value)            →  list of (k2, v2)
   Reduce(k2, list of v2)     →  list of v3
```

That's it. The framework handles **everything else**: scheduling, data movement, failures, retries, load balancing.

### 3.3 Analogy: a national postal sorting facility

> You need to count how many letters were sent **to each city**, given billions of letters scattered across thousands of regional post offices.

**Step 1 — Map (sorting at each post office)**
At every regional office, workers grab letters off the conveyor belt. For each letter they see, they scribble a slip: *"Boston: 1"*, *"NYC: 1"*, *"Boston: 1"*, … and drop each slip into a bin labeled with the destination city.

> 🠮 *Map workers never talk to each other. Each just processes its own pile.*

**Step 2 — Shuffle (the truck network)**
A fleet of trucks collects bins from every regional office and delivers them by destination: all "Boston" slips converge at the Boston counter, all "NYC" slips at the NYC counter.

**Step 3 — Reduce (the counters)**
At each city's counter, a single worker counts the slips: *"Boston: 4,239,201"*. Writes the total to a master ledger.

> 🠮 *Reduce workers also never talk to each other. Each just counts its own pile.*

**Failure handling**
- If a regional office burns down → redo its sorting at a nearby office (the original letters are still in the national archive — that's GFS).
- If a city counter quits → reassign the slip pile to someone else.
- The work is **pure** — re-doing it yields the same answer.

That is MapReduce in three paragraphs.

### 3.4 Word-count example

```python
Map(filename, contents):
    for word w in contents.split():
        emit(w, "1")

Reduce(word, counts):
    emit(len(counts))
```

### 3.5 Data flow

```
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ Input 1 │    │ Input 2 │    │ Input 3 │      INPUT
   └────┬────┘    └────┬────┘    └────┬────┘      (split into M files on GFS)
        │              │              │
        ▼  Map         ▼  Map         ▼  Map      MAP PHASE
     (a,1)(b,1)     (b,1)        (a,1)(c,1)       (parallel, no interaction)
        │              │              │
        └──────┬───────┴───────┬──────┘           SHUFFLE
               │  partition    │                  (network, all-to-all)
               │   by key      │
        ┌──────┴────┬──────────┴──────┐
        ▼           ▼                 ▼
     (a,[1,1])   (b,[1,1])         (c,[1])
        │           │                 │
        ▼ Reduce    ▼ Reduce          ▼ Reduce    REDUCE PHASE
       a → 2       b → 2             c → 1       (parallel, no interaction)
        │           │                 │
        └───────────┴─────────────────┘           OUTPUT
                       │                          (one file per Reduce, on GFS)
                       ▼
                 final results
```

### 3.6 Architecture (Figure 1 of the paper)

```
                     ┌──────────────┐
                     │    MASTER    │   one process; tracks task state,
                     │              │   hands out work, detects failures
                     └──┬────────┬──┘
            ┌───────────┘        └───────────┐
            │ assign Map                     │ assign Reduce
            ▼                                ▼
    ┌───────────────┐    intermediate    ┌───────────────┐
    │  MAP WORKERS  │ ─── files via ───▶ │ REDUCE WORKERS│
    │               │      network       │               │
    │ read from GFS │   (shuffle phase)  │ write to GFS  │
    │ write local   │                    │               │
    └───────────────┘                    └───────────────┘
            ▲                                    │
            │                                    │
            └─────────── GFS cluster ◀───────────┘
                  (replicated, 64 MB chunks)
```

Flow:
1. Master assigns Map tasks until all complete. Maps write output to **local disk**, partitioned into one file per Reduce task (by hash of key).
2. After all Maps finish, master assigns Reduce tasks. Each Reduce pulls its partition from every Map worker, sorts by key, calls `Reduce()`, writes one output file to GFS.

### 3.7 What does it hide from the programmer?

- Sending app code to servers
- Tracking which tasks are done
- Moving data from Maps to Reduces
- Balancing load across workers
- Recovering from failures

### 3.8 What it *cannot* do

- No state or interaction between tasks (except via intermediate output)
- No iteration, no multi-stage pipelines (would need to chain jobs manually)
- No real-time / streaming processing

### 3.9 Fault tolerance — in detail

The framework's superpower is that **`Map` and `Reduce` must be pure deterministic functions**. No side effects, no file I/O outside the framework, no network calls. This means re-running them always produces the same output, so the framework can re-run freely.

| Failure | Recovery |
|---|---|
| **Map worker crashes** | Master re-runs its tasks on other workers. Intermediate data was on local disk → gone → must regenerate. (Skip if Reduces already fetched it.) |
| **Reduce worker crashes** | Finished output is safe on GFS (replicated). Unfinished tasks reassigned. |
| **Master gives same Map to two workers** | Reduces are told about only one. Other's output ignored. |
| **Master gives same Reduce to two workers** | Both try to write the same GFS file. **Atomic GFS rename** ensures exactly one complete file appears. |
| **Slow worker (straggler)** | Master speculatively launches a duplicate copy of the last few tasks; first to finish wins. |
| **Incorrect output (bad CPU/RAM)** | Out of scope. MR assumes **fail-stop** hardware. |
| **Master crashes** | Bad day. The paper's original design treats this as fatal — the whole job restarts. |

### 3.10 Performance — what limits MapReduce?

In 2004, the bottleneck was **network bandwidth**:

```
   ┌──────────────────────────────────────────────┐
   │              ROOT SWITCH                     │   100–200 Gbps total
   │              (bottleneck)                    │
   └──────┬──────────────────────────────┬────────┘
          │                              │
     ┌────┴────┐                    ┌────┴────┐
     │ rack    │     ... ...        │ rack    │
     │ switch  │                    │ switch  │
     └─┬─┬─┬─┬─┘                    └─┬─┬─┬─┬─┘
       │ │ │ │   1800 machines        │ │ │ │
                 → ~55 Mbps per machine through root
                 → much slower than local disk or RAM
```

Half of all shuffle traffic crosses the root switch. So the optimizations are network-centric:

1. **Locality** — schedule each Map on the GFS server holding its input. Read input from **local disk**, not over the network.
2. **Direct shuffle** — Map writes intermediate data to its local disk. Reduces fetch it directly from Map workers, **not** through GFS (which would double the traffic from replication).
3. **Chunky transfers** — `R` (number of Reduce tasks) is much smaller than the number of keys, so each network transfer is a big batch — efficient.

Today, network and root switches are much faster relative to CPU/disk, so the bottleneck has shifted.

### 3.11 Load balancing

> **Problem:** If task sizes vary, N-1 workers wait idle for 1 slow task to finish.

**Trick:** create **many more tasks than workers**. Master hands out new tasks as workers finish. Fast workers naturally pull more tasks; slow workers pull fewer. Everyone finishes near the same time.

### 3.12 Verdict

| Wins | Loses |
|---|---|
| Scales nearly linearly | Not the most efficient |
| Easy to program (sequential code) | Not flexible (no iteration/streaming) |
| Hides failures & data movement | Awkward for multi-stage pipelines |

### 3.13 Legacy

- **Hugely influential** — birthed Hadoop, Spark, and the modern big-data stack
- Probably **no longer used at Google** — replaced by Flume / FlumeJava
- **GFS** replaced by Colossus + Bigtable

The *ideas* — parallelism via partitioning, fault-tolerance via re-execution of pure functions, separation of user code from infrastructure — are everywhere now.

---

## 4. Course Logistics

### Format
- **Lectures** — big ideas + paper discussion (video-taped)
- **Papers** — read before class; submit a question + answer by midnight the prior night
- **Exams** — midterm in class + final exam week (mostly papers and labs)
- **Labs** — graded by test cases (tests provided)

### Labs

| # | Topic |
|---|---|
| 1 | MapReduce |
| 2 | Replication via Raft |
| 3 | Fault-tolerant key/value store |
| 4 | Sharded key/value store *(or a final project, in groups of 2–3)* |

### Survival tips
- Start labs **early** — debugging distributed code takes time
- Office hours + Piazza are your friends

---

> **Takeaway**
> MapReduce single-handedly made big-cluster computation mainstream. It's not the most efficient or flexible model, but it scales well and is easy to program — failures and data movement are hidden. Those were the right trade-offs in 2004, and the descendants of MapReduce dominate data engineering today.
