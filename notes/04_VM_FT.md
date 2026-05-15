# Lecture 4 — Primary/Backup Replication (VMware FT)

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Lecturer: Robert Morris · Course site: [pdos.csail.mit.edu/6.824](http://pdos.csail.mit.edu/6.824)

---

## Table of Contents

1. [Replication Basics](#1-replication-basics)
2. [Two Replication Styles](#2-two-replication-styles)
3. [VMware FT: Machine-Level Replication](#3-vmware-ft-machine-level-replication)
4. [Where Divergence Comes From](#4-where-divergence-comes-from)
5. [Log, Replay, and Timing](#5-log-replay-and-timing)
6. [Inputs: Network and DMA](#6-inputs-network-and-dma)
7. [Outputs and the Output Rule](#7-outputs-and-the-output-rule)
8. [Failure, Cutover, and Duplicates](#8-failure-cutover-and-duplicates)
9. [Split Brain and Tie-Breaking](#9-split-brain-and-tie-breaking)
10. [Performance and When FT Makes Sense](#10-performance-and-when-ft-makes-sense)
11. [Summary](#11-summary)

---

## 1. Replication Basics

Replication is the main tool for **availability** in the presence of server or network failure.

### What failures can replication handle?

- **Fail-stop** failures: a machine halts (power loss, disk full, cable unplugged).
- **Not covered well**: correlated failures (same bug on every replica), operator mistakes, or hardware defects that produce wrong results.
- **Large-scale events**: earthquakes or city-wide power loss need geographic separation.

### Analogy: Two pilots in the same cockpit

One pilot can take over if the other becomes incapacitated. This helps with sudden failures, but not if the aircraft itself is damaged or the flight plan is wrong.

---

## 2. Two Replication Styles

### State Transfer

- Primary executes.
- Primary ships the **state** to backups.
- Simple, but expensive when state is large.

### Replicated State Machine (RSM)

- Primary ships **operations** in a total order.
- All replicas execute operations deterministically.
- More complex, but often far less network traffic.

VMware FT uses an RSM approach. Labs 2–4 use it too.

---

## 3. VMware FT: Machine-Level Replication

VMware FT replicates the *entire machine state* (registers, memory, interrupts), not just application state.

**Goal:** run unmodified OS + server software and still get fault tolerance.

### High-level architecture

```
Clients
  |
  v
Primary VM  -- log channel -->  Backup VM
  |                               |
  | shared disk                   | shared disk
  v                               v
Network Disk Server (tie-breaker + storage)
```

Key ideas:
- **Primary** handles all real outputs.
- **Backup** replays and stays slightly behind.
- **Logging channel** carries nondeterministic events.

---

## 4. Where Divergence Comes From

If primary and backup diverge, a failover exposes inconsistent state.

### Sources of nondeterminism

- Device inputs (network packets, disk DMA)
- Interrupt timing
- Instructions that depend on external state (time, cycle counter)

### Why this matters: a lease example

The GFS master renews leases based on a timer interrupt. If the interrupt arrives just before a request on the backup but just after on the primary, they make opposite decisions. After failover, clients could see two primaries for the same lease.

**Conclusion:** primary and backup must see the **same events**, in the **same order**, at the **same instruction boundary**.

---

## 5. Log, Replay, and Timing

Each log entry includes:

- **Instruction number**
- **Event type**
- **Event data**

### Timer interrupts

Primary:
1. Hypervisor intercepts interrupt.
2. Records instruction number.
3. Sends log entry to backup.
4. Delivers interrupt to primary.

Backup:
1. Waits for log entry.
2. Programs CPU to interrupt after the recorded instruction count.
3. Injects the interrupt at the exact same point.

---

## 6. Inputs: Network and DMA

Network packets arrive via DMA, which is dangerous because DMA can modify guest memory while the guest is executing.

### Bounce buffer technique

1. NIC copies packet into a private buffer (not guest memory).
2. Hypervisor pauses the primary.
3. Copies the buffer into guest memory.
4. Sends packet + instruction number to backup.

Backup injects the same data at the same instruction boundary.

```
NIC --> [BOUNCE BUFFER] --(pause guest)--> GUEST MEMORY
  |                                         |
  | log entry                               | same instruction
  v                                         v
Backup receives and replays                 Backup memory updated
```

**Reason:** data must appear at the exact same point in execution on both replicas.

---

## 7. Outputs and the Output Rule

Primary and backup both execute output instructions, but only the primary is allowed to send output externally.

### The Output Rule

The primary must wait until the backup **acknowledges** the log entries that caused the output.

This prevents the primary from sending a reply that the backup never learns about.

### Example: increment RPC

```
Client -> "increment"
Primary executes: value 10 -> 11
Primary waits for backup ack
Primary sends "11" to client
Backup executes same increment, output suppressed
```

---

## 8. Failure, Cutover, and Duplicates

### What if the primary crashes after output?

The backup may emit the same output again after failover. This is usually OK:
- **TCP** drops duplicate sequence numbers.
- **Disk writes** are idempotent if the same data is written to the same block.

### What if the primary crashes before output?

If the primary never gets the ack, it never sends output. The backup will eventually replay and send it after becoming primary.

**Takeaway:** duplicates are common in replication. Clients must handle them or make them harmless.

---

## 9. Split Brain and Tie-Breaking

If primary and backup cannot reach each other, both might try to go live.

### Tie-breaker: atomic test-and-set on shared disk

Only the VM that successfully flips a shared flag may become primary. The other halts.

This makes the shared disk a potential single point of failure, so it should be replicated too.

---

## 10. Performance and When FT Makes Sense

### Observed performance

- Overhead is surprisingly low.
- Logging bandwidth ~18 Mbit/s for MySQL in the paper.

### When FT is attractive

- Critical but low-intensity services (DNS, small config services).
- Legacy services you cannot modify.

### When to prefer application-level replication

- High-throughput databases.
- When you can replicate only application state rather than full VM memory.

---

## 11. Summary

- **Primary/backup replication** gives availability under fail-stop failures.
- **VMware FT** replicates at machine level for transparency.
- **Output Rule** is central but limits performance.
- **Split brain** avoided via tie-breaking on shared storage.
- For performance, app-level replicated state machines are often better.

---

## References

- VMware KB 1013428 (multi-CPU support)
- http://www.wooditwork.com/2014/08/26/whats-new-vsphere-6-0-fault-tolerance/
- http://www-mount.ece.umn.edu/~jjyi/MoBS/2007/program/01C-Xu.pdf
{"mode":"full","isActive":false}