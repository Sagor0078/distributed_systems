# Lecture 5 — Go Threads, Concurrency, and Debugging

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Lecturer: Robert Morris · Course site: [pdos.csail.mit.edu/6.824](http://pdos.csail.mit.edu/6.824)

---

## Table of Contents

1. [Why Concurrency?](#1-why-concurrency)
2. [Go Memory Model](#2-go-memory-model)
3. [Goroutines and Closures](#3-goroutines-and-closures)
4. [Time and Cancellation](#4-time-and-cancellation)
5. [Mutexes and Invariants](#5-mutexes-and-invariants)
6. [Condition Variables](#6-condition-variables)
7. [Channels](#7-channels)
8. [Debugging Concurrency](#8-debugging-concurrency)
9. [General CLI Tools](#9-general-cli-tools)

---

## 1. Why Concurrency?

Concurrency is the tool we use to express **independent activities** that overlap in time.

- **Expressivity**: a natural way to model many tasks happening at once.
- **Performance**: parallelism on multiple cores.
- **In labs**: correctness matters more than raw performance.

### Analogy: A busy coffee shop

One barista can take orders, grind beans, and steam milk, but a shop runs better with multiple baristas handling different tasks in parallel. The challenge is making sure they do not collide on the same espresso machine (shared resource).

---

## 2. Go Memory Model

The memory model explains **when one goroutine can reliably see another goroutine's writes**. It is a guide to correct concurrent code.

> If you feel forced to read the memory model frequently, your code might be too clever.

---

## 3. Goroutines and Closures

### Goroutines

- Lightweight threads.
- Run concurrently.
- Program ends when `main` exits.

### Closures and common gotchas

Closures capture variables from the surrounding scope. In loops, this often leads to bugs if the loop variable is reused.

Examples in code:
- `closure.go`: closures + goroutines
- `loop.go`: correct loop capture pattern
- `bad.go`: incorrect capture of loop variable

### Diagram: goroutines and shared variables

```
main goroutine
  |
  |-- go worker(i=0)
  |-- go worker(i=1)
  |-- go worker(i=2)
```

If each goroutine captures the same `i`, they may all print the **final** value.

---

## 4. Time and Cancellation

- `sleep.go`: `time.Now`, `time.Sleep`
- `sleep-cancel.go`: avoid infinite loops in labs; use `rf.killed()` or a cancel signal.

---

## 5. Mutexes and Invariants

Mutexes protect **invariants**, not just data.

### Key idea: critical sections

Inside a critical section, you may temporarily break an invariant while updating multiple fields. The invariant must hold **before unlock**.

Examples:
- `basic.go`: `Lock`, `Unlock`, and `defer` usage
- `per-field.go`: locking to protect invariants
- `bank.go`: classic race condition example

### Analogy: a shared spreadsheet

Two people editing different cells can still corrupt a formula that depends on both. Lock the sheet while updating related fields.

---

## 6. Condition Variables

Condition variables coordinate goroutines that must **wait** for something to become true.

- `cond.txt`: usage and pitfalls
- `vote-count-1.go` to `vote-count-4.go`: progressively correct patterns

**Rules of thumb:**
- Always hold the lock when calling `Wait`.
- Always check the condition in a loop.

### Diagram: wait/signal

```
goroutine A: lock -> while !ready { wait } -> proceed
goroutine B: lock -> set ready=true -> signal -> unlock
```

---

## 7. Channels

Channels combine communication and synchronization.

- `producer-consumer.go`
- `unbuffered.go`: unbuffered channels are synchronous
- `deadlock.go`: common pitfalls

### When channels shine

- Producer/consumer pipelines
- Fan-in / fan-out patterns
- Waiting for a group of results

### When channels are awkward

- "Kick" a goroutine that may not be waiting

> Instructor preference: use shared memory (`Mutex`, `Cond`) for most lab code.

---

## 8. Debugging Concurrency

### Tracing

Use `DPrintf` in `util.go` to trace phases in tests:

```go
// if the leader disconnects, a new one should be elected.
cfg.disconnect(leader1)
DPrintf("TESTACTION: leader disconnects")
cfg.checkOneLeader()
```

### Race detection

```bash
go test -race -run 2A
```

Helpful but not a proof; it only detects certain races.

### SIGQUIT stack dumps

- `Ctrl+\` prints all goroutine stacks, then exits.

### Leaking goroutines

- Use `ps` to inspect processes.
- Send `kill -QUIT pid` or `kill -KILL pid`.

### Run tests in parallel

Parallel runs reveal concurrency bugs more quickly:

https://gist.github.com/jonhoo/f686cacb4b9fe716d5aa

---

## 9. General CLI Tools

### Manuals and references

- `man toolname` (quit with `q`, search with `/`)
- `tldr toolname` (needs install: https://tldr.sh/#installation)

### Redirecting output

- `>` redirects stdout only
- `&>` redirects stdout + stderr
- `|` and `|&` pipe outputs
- `tee` writes to file and shows output

```bash
echo "example" | tee FILE
```

### Reading files quickly

- `head`, `tail`, `less`
- `grep -in "pattern" file`
- `rg "pattern"` (ripgrep)

---

> **Takeaway**
> Use goroutines for structure, mutexes or channels for correctness, and the memory model as the safety net. Debugging concurrency is a first-class skill in 6.824, so use tracing, the race detector, and parallel tests early and often.
{"mode":"full","isActive":false}