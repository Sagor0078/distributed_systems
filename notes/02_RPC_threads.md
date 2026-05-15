# Lecture 2 — Infrastructure: RPC & Threads

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Lecturer: Robert Morris · Course site: [pdos.csail.mit.edu/6.824](http://pdos.csail.mit.edu/6.824)

---

## Table of Contents

1. [Why Go for Distributed Systems?](#1-why-go-for-distributed-systems)
2. [Threads & Goroutines](#2-threads--goroutines)
3. [The Three Hazards of Threading](#3-the-three-hazards-of-threading)
4. [Threads vs. Event-Driven Programming](#4-threads-vs-event-driven-programming)
5. [Case Study: A Web Crawler](#5-case-study-a-web-crawler)
6. [Remote Procedure Call (RPC)](#6-remote-procedure-call-rpc)
7. [The Problem of Failure](#7-the-problem-of-failure)
8. [RPC Failure Semantics](#8-rpc-failure-semantics)
9. [Takeaway](#9-takeaway)

---

## 1. Why Go for Distributed Systems?

The labs use Go. It's not arbitrary — Go's design hits the sweet spot for building distributed systems.

| Feature | Why it matters |
|---|---|
| **Goroutines & channels** | Concurrency is a *language feature*, not a library afterthought |
| **`net/rpc` in the standard library** | RPC works out of the box — no extra dependencies |
| **Garbage collected, type safe** | No use-after-free, fewer footguns |
| **Small language** | Read a single 100-page book and you know the whole thing |
| **Static binaries** | `go build` produces one file you can drop anywhere |

> Reference: [Effective Go](https://golang.org/doc/effective_go.html).

> **Why not Python or C++?**
> - Python: GIL hurts true CPU parallelism; weak typing hides bugs that bite at scale.
> - C++: every memory bug is yours forever; concurrency tools are bolted on, not built in.

---

## 2. Threads & Goroutines

Go calls them **goroutines**. Mentally substitute "thread" — they're the same abstraction with cheaper bookkeeping.

> **Definition** — A thread is an independent flow of execution within one program. Multiple threads share the program's memory, but each carries its own program counter, registers, and call stack.

```
   One process, multiple threads:

   ┌─────────────────────────────────────────────────────┐
   │  Process memory (heap, globals)                     │
   ├──────────────┬──────────────┬──────────────────────┤
   │  Thread 1    │  Thread 2    │  Thread 3            │
   │  • PC        │  • PC        │  • PC                │
   │  • registers │  • registers │  • registers         │
   │  • stack     │  • stack     │  • stack             │
   └──────────────┴──────────────┴──────────────────────┘
       all three see the SAME heap and globals
```

### 2.1 Three reasons to use threads

| Reason | What it buys you |
|---|---|
| **I/O concurrency** | While thread A waits on the network, thread B does work |
| **Multicore parallelism** | Independent threads on different cores → real wall-clock speedup |
| **Convenience** | Periodic housekeeping (heartbeats, GC) doesn't tangle main logic |

### 2.2 Goroutines are cheap

| | OS thread | Goroutine |
|---|---|---|
| **Initial stack** | ~1 MB | ~2 KB (grows dynamically) |
| **Creation cost** | µs | ns |
| **Practical count** | thousands | millions |
| **Scheduled by** | OS kernel | Go runtime (m:n on top of a few OS threads) |

You can spin up 100,000 goroutines without thinking. You cannot do that with `pthread_create`.

### 2.3 Analogy — A team of chefs in one kitchen

One chef preparing a multi-course meal does everything sequentially: chop, then boil, then sauté. Slow.

A **team of chefs** in the same kitchen:

- One chops while another boils — **I/O concurrency** (waiting for water to boil = waiting for I/O).
- Multiple stoves (cores) → dishes cook in parallel — **multicore performance**.
- Everyone shares the spice rack and pantry — **shared memory**.

That's threading. The shared kitchen is also where the trouble starts.

---

## 3. The Three Hazards of Threading

### 3.1 Race conditions

Two chefs grab for the last pinch of salt at the same instant. Or one reads a recipe while another is rewriting it. The result is undefined: maybe both think they got the salt; maybe the half-rewritten recipe is unreadable.

```
   Time:    t1               t2
   T1:      read x  ─▶ 5
   T2:                       read x  ─▶ 5
   T1:      write x = 6
   T2:                       write x = 6     ← lost increment! should be 7
```

**Fix**: a **lock** (`sync.Mutex`). Only the thread holding the lock can touch the shared data.

```go
mu.Lock()
balance += amount
mu.Unlock()
```

> **Rule of thumb**: if more than one thread can read AND at least one can write, you need a lock (or another synchronization primitive).

### 3.2 Coordination

A producer chef preps ingredients; a consumer chef cooks them. How does the consumer know when prep is done?

**Fixes:**
- **`sync.Cond`** — explicit wait/signal.
- **`sync.WaitGroup`** — "wait for N goroutines to finish".
- **Channels** — pass the prepped ingredients down a conveyor belt.

### 3.3 Deadlock

Chef A holds the pan, waits for the knife. Chef B holds the knife, waits for the pan. Both wait forever.

```
   A → wants Pan       held by B
   A → has   Knife     wanted by B
   B → wants Knife
   B → has   Pan

   Cycle in the "wait-for" graph → deadlock.
```

> **Standard prevention**: always acquire locks in the same global order. If A and B both grab `(knife, pan)` in that order, neither can hold one while waiting for the other.

---

## 4. Threads vs. Event-Driven Programming

There's an alternative to "many threads sharing memory" — **a single thread cycling through tasks**.

```
   Event loop:
   ────────────
   while True:
       event = next_ready_event()
       handle(event)         ← never blocks
       continue

   "ready" might mean: socket has bytes, timer fired, file descriptor writable, etc.
```

### 4.1 Trade-offs

| | Threads | Event loop |
|---|---|---|
| Race conditions | Many sources | Almost none (single-threaded) |
| Multicore use | Native | Hard — need one event loop per core |
| Programming style | Linear, blocking-looking | Callbacks, state machines |
| Debugging | Hard (concurrent bugs) | Hard (callback spaghetti) |
| Memory overhead | Stack per thread | One stack |
| Best for | CPU + I/O mix | I/O-bound servers (Node.js, nginx) |

### 4.2 Analogy

- **Threads** = team of chefs, parallel work, occasional collisions.
- **Event loop** = one hyper-organized chef juggling a checklist, never confused but can't use multiple stoves.

This course leans on threads because the labs need true parallelism + simple sequential-looking code.

---

## 5. Case Study: A Web Crawler

Goal: visit every reachable page from a root URL.

**Constraints:**
- **High latency** per fetch → must do many in parallel (I/O concurrency).
- **Cycles & duplicates** → fetch each URL exactly once.
- **Termination** → stop when there's no more work.

### 5.1 Crawler 1 — Serial

```go
func Serial(url string, fetcher Fetcher, fetched map[string]bool) {
    if fetched[url] { return }
    fetched[url] = true
    urls, _ := fetcher.Fetch(url)
    for _, u := range urls {
        Serial(u, fetcher, fetched)
    }
}
```

Simple recursive DFS. The `fetched` map prevents revisits.

> **Problem**: one fetch at a time. A crawl with 10,000 pages × 100 ms each = 17 minutes.

### 5.2 Crawler 2 — Goroutines + Mutex

Launch each fetch in its own goroutine:

```go
go ConcurrentMutex(u, fetcher, fetched)
```

But all goroutines now share `fetched` — **race condition**. Two goroutines might both see "not fetched", both decide to fetch.

```
   Goroutine A:  if !fetched[u]      ← reads false
   Goroutine B:  if !fetched[u]      ← reads false (same instant)
   Goroutine A:  fetched[u] = true
   Goroutine B:  fetched[u] = true
   → both fetch the URL.
```

**Fix**: a mutex around the check-and-set:

```go
fetcher.mu.Lock()
already := fetched[url]
fetched[url] = true
fetcher.mu.Unlock()

if already { return }
// proceed to fetch
```

**How do we know everyone's done?** `sync.WaitGroup`:

```go
var wg sync.WaitGroup
for _, u := range urls {
    wg.Add(1)
    go func(u string) {
        defer wg.Done()
        ConcurrentMutex(u, fetcher, fetched)
    }(u)
}
wg.Wait()    // block until all children done
```

### 5.3 Crawler 3 — Channels (no shared map)

Different mindset: don't share data, **pass data through channels**.

```
   ┌─────────┐                  ┌─────────┐
   │ Master  │ ──URL to fetch──▶│ Worker  │
   │         │                  │         │
   │         │ ◀─URLs found ────│         │
   └─────────┘                  └─────────┘
        │
        │ keeps fetched-set, only the master touches it
        │ → no mutex needed
        ▼
     ... spawns more workers as new URLs come in ...
```

```go
func master(ch chan []string, fetcher Fetcher) {
    n := 1
    fetched := make(map[string]bool)
    for urls := range ch {
        for _, u := range urls {
            if !fetched[u] {
                fetched[u] = true
                n++
                go worker(u, ch, fetcher)
            }
        }
        n--
        if n == 0 { break }
    }
}

func worker(url string, ch chan []string, fetcher Fetcher) {
    urls, _ := fetcher.Fetch(url)
    ch <- urls
}
```

> **Slogan**: *"Don't communicate by sharing memory; share memory by communicating."*

The `fetched` map is only touched by the master — **no mutex needed**. The channel is the synchronization.

### 5.4 Channels in one picture

> A channel is a typed conveyor belt with optional buffering.

```
   Sender:    ch <- x       (puts on the belt, blocks if buffer full)
   Receiver:  x := <- ch    (takes off the belt, blocks if empty)

       ┌──────────────────┐
   ──▶ │ x   y   z        │ ──▶
       └──────────────────┘
       buffered channel (capacity 4)
```

Channels also synchronize: a receiver on an empty channel blocks until something arrives. That blocking *is* coordination.

---

## 6. Remote Procedure Call (RPC)

RPC is **the** building block for distributed systems. Without it, you'd write socket code in every layer.

> **Goal**: make a function call on a remote machine look and feel like a local function call.

### 6.1 Architecture

```
   CLIENT MACHINE                          SERVER MACHINE
   ┌─────────────────────┐                 ┌─────────────────────┐
   │ Application code    │                 │ Handler functions   │
   │   server.Add(2, 3)  │                 │   func Add(...)     │
   └──────────┬──────────┘                 └──────────▲──────────┘
              │                                       │
   ┌──────────▼──────────┐                 ┌──────────┴──────────┐
   │ Client Stub          │                │  Dispatcher          │
   │ - marshal args       │                │  - unmarshal args    │
   │ - prepare request    │                │  - lookup handler    │
   └──────────┬──────────┘                 └──────────▲──────────┘
              │                                       │
   ┌──────────▼──────────┐                 ┌──────────┴──────────┐
   │ RPC library          │                │  RPC library         │
   │ (net/rpc)            │                │  (net/rpc)           │
   └──────────┬──────────┘                 └──────────▲──────────┘
              │                                       │
              └──────────  NETWORK  ──────────────────┘
                   (TCP, encoded bytes)
```

### 6.2 Step by step

1. App calls a **normal-looking method** on a client object (the **stub**).
2. Stub **marshals** the arguments — converts Go values into a wire format (`gob`, JSON, protobuf, etc.).
3. RPC library sends the bytes over TCP.
4. Server's RPC library receives the bytes and hands them to the **dispatcher**.
5. Dispatcher looks up the function name (`"Add"`) and unmarshals the args.
6. Server runs the **handler**.
7. Result is marshalled, sent back, unmarshalled on the client, returned to the app.

> **The illusion**: from the app's perspective, it just called `server.Add(2, 3)`. The wire, the marshalling, the dispatch — invisible.

### 6.3 Analogy — Mail-order forms

You want a custom cake. You don't drive to the bakery, you fill out an **order form** (marshal), mail it (network), and a baker on the other end reads the form (dispatch), bakes (handler), and mails the cake back. The form-and-mail system pretends you "called" the bakery directly. RPC is the same protocol with bits instead of cakes.

---

## 7. The Problem of Failure

> **The defining headache of distributed systems**: you can't tell why a request didn't come back.

```
   CLIENT                               SERVER
     │                                    │
     │── Request ─────X (lost)            │ Case A: never arrived
     │                                    │
     │── Request ──────────────────▶ ─────│ Case B: arrived
     │                                    │  → processed
     │                                    │  → crashed
     │                                    X  → reply lost
     │ (timeout)                          │
     │                                    │
     │── Request ──────────────────▶ ─────│ Case C: arrived
     │                                    │  → being processed slowly
     │                                    │  → reply eventually comes
     │ (already timed out)                │
```

From the client's side, **all three look identical**: no reply. But the right next move is different for each.

### 7.1 The retry dilemma

| Case | Should client retry? | If client does retry... |
|---|---|---|
| **A** (request lost) | Yes | Server processes once. Good. |
| **B** (reply lost after success) | Maybe | Server processes again. For `Put`, fine. For `Withdraw($100)`, **bad**. |
| **C** (slow, eventually succeeds) | Maybe | Server processes twice. Same issue. |

The client cannot tell which case it's in. So:

> If the operation is **idempotent** (Put, Set, Read), just retry.
> If it's **non-idempotent** (Increment, Withdraw, AddToCart), you need a smarter protocol — duplicate detection, request IDs, etc.

---

## 8. RPC Failure Semantics

Three classic guarantees, weakest to strongest:

### 8.1 At-least-once

```
   Client retries until it gets a reply.
   → operation runs 1 or more times.
```

- **Simple**: just retry on timeout.
- **Safe only if**: handlers are **idempotent**. Re-applying gives the same result. Examples: `Set(k, v)`, `Lock(L)` if already-locked-by-me succeeds.
- **Unsafe for**: counters, withdrawals, list appends.

### 8.2 At-most-once

```
   Client sends each request once with a unique ID (XID).
   Server remembers recently-seen XIDs.
   On duplicate XID: return the cached reply, do NOT re-execute.
   → operation runs 0 or 1 times.
```

- **Safer**: non-idempotent ops don't double-execute.
- **Cost**: server keeps a duplicate-detection table; must decide when to garbage-collect.
- **Catch**: if server crashes and forgets the table → duplicate executions possible. Solving this requires persistence — Lab 3's territory.

### 8.3 Exactly-once

```
   At-most-once + fault-tolerant server (state survives crash).
   → operation runs exactly 1 time, even with failures.
```

The holy grail. Requires every layer to cooperate:

- Reliable retries on the client.
- Duplicate detection on the server.
- Persistent state so the server's table survives crashes.
- Replication so the server itself doesn't disappear.

Lab 3 builds exactly-once on top of Raft.

### 8.4 What Go's `net/rpc` gives you

> **At-most-once**, but the weak version.

- Uses a reliable TCP connection.
- On timeout or broken connection, returns an **error**.
- **Never automatically retries** — that's your call.

```go
err := client.Call("Foo.Add", args, &reply)
if err != nil {
    // YOU decide: retry? give up? compensate?
}
```

The framework hands you the failure, leaving the retry policy where it belongs: in code that understands whether the operation is safe to repeat.

### 8.5 Analogy — Online shopping confirmation

You click "Place Order" and get a network error.

- **At-least-once world**: click again — you might end up with two pairs of shoes.
- **At-most-once world**: the site uses your order ID (XID) to ensure only one purchase, but you may need to manually check your account to know if it went through.
- **Exactly-once world**: the site guarantees one pair regardless — but achieves this with extensive state management behind the scenes.

Real e-commerce sites lean on at-most-once + idempotency keys (the "X-Idempotency-Key" header is just an XID).

---

## 9. Takeaway

> **The two building blocks of every distributed system are threads (to overlap work) and RPCs (to coordinate across machines).** Threads need synchronization (locks, channels, WaitGroups); RPCs need a failure story (idempotency, retries, duplicate detection). The labs in this course exercise both — Lab 1's worker uses goroutines, Lab 2+ adds RPC failure handling on top.

### Mental checklist

| Concept | One-liner |
|---|---|
| **Goroutine** | Cheap user-space thread — spin up thousands |
| **Mutex** | Make a critical section atomic |
| **Channel** | Typed conveyor belt that also synchronizes |
| **WaitGroup** | "Wait for N goroutines to finish" |
| **Race condition** | Two threads, one shared mutable, no sync → undefined behavior |
| **Deadlock** | Cycle in the wait-for graph → fix by ordering lock acquisition |
| **RPC** | Make a remote call look local — marshal, send, dispatch, reply |
| **At-least-once** | Retry forever; only safe for idempotent ops |
| **At-most-once** | XIDs + dedup table; runs 0–1 times |
| **Exactly-once** | At-most-once + replicated/persisted server state |
| **Go's `net/rpc`** | At-most-once-ish; returns error on failure, never auto-retries |

### The recurring lesson

> **You cannot tell a slow server from a dead server from a lost packet.** Every higher-level protocol in this course — Raft, ZooKeeper, Aurora, Spanner — is some answer to that fundamental ambiguity.
