# Lecture 11 — Frangipani: FAQ

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Companion Q&A to [11_Frangipani.md](11_Frangipani.md)
> Paper: *"Frangipani: A Scalable Distributed File System"* — Thekkath, Mann, Lee, SOSP 1997

---

## Table of Contents

1. [Why This Paper?](#1-why-this-paper)
2. [Frangipani vs. GFS](#2-frangipani-vs-gfs)
3. [Architecture Choices](#3-architecture-choices)
4. [Trust & Security](#4-trust--security)
5. [Crash Recovery & The Two Logs](#5-crash-recovery--the-two-logs)
6. [Performance Quirks](#6-performance-quirks)
7. [Striping, False Sharing & Lock Granularity](#7-striping-false-sharing--lock-granularity)
8. [Version Numbers & Cross-Log Replay](#8-version-numbers--cross-log-replay)
9. [Frangipani vs. Two-Phase Commit](#9-frangipani-vs-two-phase-commit)
10. [Historical & Misc](#10-historical--misc)
11. [Cheat Sheet](#11-cheat-sheet)

---

## 1. Why This Paper?

### Q: Why are we reading Frangipani?

**A.** Primarily as a **cache-coherence case study**. But also interesting:

- Each client has its **own log**, stored in the shared store so any other client can recover from it.
- Logs are **intertwined**: updates to a single object may be spread across many clients' logs. Replaying one log alone is tricky — hence the version-number trick.
- It's an early example of building a complex system out of **simple shared storage + smart decentralized participants**.

---

## 2. Frangipani vs. GFS

### Q: How does Frangipani differ from GFS?

**A.** Same word ("file system"), very different shape.

```
   GFS:                                    Frangipani:

   ┌─────────┐    ┌─────────┐              ┌─────────┐    ┌─────────┐
   │ Client  │    │ Client  │              │  WS-1   │    │  WS-2   │
   │ (dumb)  │    │ (dumb)  │              │  (FS    │    │  (FS    │
   └────┬────┘    └────┬────┘              │  logic) │    │  logic) │
        │              │                   └────┬────┘    └────┬────┘
        ▼              ▼                        │              │
   ┌──────────────────────────┐                 ▼              ▼
   │ GFS master + chunkservers│         ┌───────────────────────────┐
   │  (FS logic lives here)   │         │   Petal (dumb blocks)     │
   │  (no caches)             │         │   shared by all clients   │
   └──────────────────────────┘         └───────────────────────────┘
```

| Aspect | GFS | Frangipani |
|---|---|---|
| Where FS logic lives | **In the servers** | **In the client workstations** |
| Caching? | None | Heavy client-side cache |
| Cache coherence? | N/A | Required (locks, callbacks) |
| Typical workload | Huge sequential reads | Many small interactive operations |
| Locking | Centralized at server | Distributed lock service |
| Looks like a real FS? | No — library API | Yes — drop-in for POSIX programs |

### 2.1 Why two designs differ so much

> GFS's customers are **batch jobs** streaming through terabytes — caches wouldn't help. Frangipani's customers are **engineers at workstations** editing source files, where 90% of activity is on one user's recently-touched files — caches help enormously.

### 2.2 Why Frangipani is hard

There's **no single file server**. Two workstations might both create a file in the same directory at the same moment. With no central serializer, Frangipani must:

- Use a distributed lock service to serialize structural changes.
- Use logs so a crashed workstation's in-flight changes don't corrupt shared metadata.

> **Analogy** — Two ways to run a library.
> - **GFS**: a single warehouse with a giant conveyor belt — patrons request books and the warehouse delivers them. The warehouse decides what's where.
> - **Frangipani**: a big shared book vault that anyone can walk into, plus a sign-out clipboard (lock service) and a notebook (log) at the entrance so different librarians don't trip over each other when reshelving.

---

## 3. Architecture Choices

### Q: Why does Petal export a block interface? Why not file-level (like AFS)?

**A.** Three reasons:

1. **Reuse** — Petal already existed, with its fault tolerance and scaling solved.
2. **Decentralization** — putting FS logic in clients spreads load. Adding a workstation adds CPU; a central file server doesn't.
3. **Simplicity for Petal** — Petal doesn't have to know about directories, inodes, ACLs, file modes. It just stores 512-byte blocks.

> **Cost**: invariants on file-system structures are now harder to enforce because no one entity owns them. Frangipani had to build its own transaction system (locks + logs) on top.

---

## 4. Trust & Security

### Q: Can a workstation running Frangipani break security?

**A.** **Yes.** Since FS logic runs in clients, every client is trusted with raw access to Petal blocks. A malicious user could:

- Modify their local Frangipani code.
- Read or write any user's data in Petal.

> **When this is acceptable**: small trusted organizations (a research lab in 1997), or where Frangipani runs on **dedicated trusted servers** that re-export NFS to untrusted clients.

> **Why this design choice in the first place**: 1997-era research at DEC on workstations within a trusted network. Modern systems sit on the "untrusted client" side and put file logic in servers.

---

## 5. Crash Recovery & The Two Logs

### 5.1 Two logs?

| Log | Lives at | Records | Purpose |
|---|---|---|---|
| **Frangipani's log** (per workstation) | Petal (shared) | High-level FS ops (create, delete, rename) | Recover after a workstation crash |
| **Petal's log** | Inside Petal | Low-level block ops (block map, free list, replica busy bits) | Recover Petal's own internal state |

Both live on disk inside Petal — but Petal sees Frangipani's logs as opaque bytes.

```
   ┌──────────────────────────────────────────────┐
   │  Petal (shared block store)                  │
   │   ┌─────────────────────────────────────┐    │
   │   │ Data blocks (inodes, files, dirs)   │    │
   │   ├─────────────────────────────────────┤    │
   │   │ WS-1's Frangipani log               │    │ ← stored as opaque
   │   │ WS-2's Frangipani log               │    │   data from Petal's
   │   │ WS-3's Frangipani log               │    │   point of view
   │   ├─────────────────────────────────────┤    │
   │   │ Petal's own internal log            │    │ ← Petal manages this
   │   └─────────────────────────────────────┘    │
   └──────────────────────────────────────────────┘
```

### 5.2 Why per-workstation logs?

If a workstation crashes mid-operation (e.g., halfway through `create + update directory`), some other workstation needs to **finish or undo** the partial work. Since the log lives on shared Petal, any peer can read it and replay.

> **Classic 2PC fails here**: the local-disk log of a crashed server is unreachable, so progress halts. Frangipani's logs are public → recovery is always possible.

### 5.3 Q: Crash recovery only covers metadata, not file contents — what does that mean?

**A.** If you write data to a file and the workstation immediately crashes, **the data may be lost**. The file system metadata is preserved, but the bytes you wrote could vanish.

This matches normal Unix semantics:

```
   write(fd, data, n)
   // bytes are in OS cache
   // crash → bytes lost
   // app survives by using fsync(fd) when it matters
```

The file system defends **its own invariants** (otherwise it's unusable after a crash) but lets applications choose when content is worth the cost of `fsync()`.

### 5.4 Q: What if a workstation crashes after a syscall returns but before its log entry reaches Petal?

**A.** That update is lost completely. The application might print "I created file x" and then crash, and x will not exist.

> Same as normal Linux. The fix is **fsync()**.

> **Analogy — autosave vs. save**
> Word's autosave runs every few minutes. If you type a sentence and Word crashes a second later, the sentence is gone. The document's *file structure* is still consistent — you can open Word again and get the last autosave. But the sentence itself was application-level state that you chose not to save.

---

## 6. Performance Quirks

### Q: What does "File creation takes longer..." (§9.2) mean?

**A.** **Double-writing** for write-ahead logging.

```
   Every metadata change must hit Petal TWICE:
       1. Write to the log.
       2. Apply to the on-disk FS structure.
       Only then can the log entry be freed.
```

When the log fills, Frangipani must **stop and drain**: flush dirty metadata blocks to Petal so the corresponding log entries become reclaimable.

### Q: Why does a bigger log boost performance?

**A.** Write absorption. Suppose the benchmark creates 1000 files in the same directory:

```
   Small log: must drain every 100 ops → directory block written ~10 times.
   Large log: drain once at the end    → directory block written ~1 time.
```

The directory block is updated by every create, but a larger log lets many updates pile up before the block has to be flushed.

> **Analogy — taking out the trash**
> Small log = small kitchen bin: you trek to the dumpster after every 5 items. Large log = big bin: you trek once a week. Same total amount of trash, far fewer trips.

---

## 7. Striping, False Sharing & Lock Granularity

### Q: What does it mean to **stripe** a file?

**A.** Spread its blocks across multiple Petal servers:

```
   File F (blocks 0..7):

       Petal S0 :  [0]  [2]  [4]  [6]
       Petal S1 :  [1]  [3]  [5]  [7]
```

Benefits:

- Reading a big file uses many disks in parallel.
- Load is distributed evenly across servers.

Similar idea to sharding, but at the **block level within one file**, not at the file level.

### Q: What's the "false sharing" problem?

**A.** Two independent items, X and Y, sit in the same 512-byte block. Different workstations want to update X and Y simultaneously, but Petal's smallest unit is the whole block.

```
   Block:  [...... X ...... Y ......]    one 512-byte chunk

   WS-A wants to write X.
   WS-B wants to write Y.
   → they fight over the SAME block → it bounces back and forth.
   → they "share" the block even though they don't share any actual data.
```

It's **sharing** (same block) that is **false** (no logical overlap). Hurts performance for no real reason.

### Q: Would per-block locking help?

**A.** **Probably worse.** Reading one file would require acquiring N locks; the lock service would burn. And in practice, sharing happens at file granularity (editor writes file, compiler reads it), not block granularity.

> **The right granularity is the one that matches your workload.** Frangipani picked **per-file locks** because that's where real contention lives.

---

## 8. Version Numbers & Cross-Log Replay

### Q: I don't understand the rule "never replay a completed update."

**A.** This is **the** tricky part of Frangipani. Here's the failure scenario:

```
   WS-1's log:  [ create(xxx) ]  [ delete(xxx) ]
   WS-2's log:                                  [ create(xxx) ]

   Real history:
       WS-1 creates xxx
       WS-1 deletes xxx
       WS-2 creates xxx  ← so xxx EXISTS now

   WS-1 crashes. Recovery starts. Replays WS-1's log:
       create(xxx)     → harmless if not already done
       delete(xxx)     ← if we re-execute this, we delete WS-2's file!
                         CORRUPTION.
```

### The version-number trick

Every metadata block has a **version number**, bumped on every modification. Every log entry stores the version it expects:

```
   Recovery rule:
       For each log entry:
           if entry.version > block.version:
               apply it, bump block.version
           else:
               skip it — already applied (or superseded)
```

In the example:

- `delete(xxx)` log entry was version, say, 7 of the directory block.
- WS-2's `create(xxx)` already bumped the directory block to version 8.
- Recovery sees `7 ≤ 8` → skips the delete. ✓

> **Analogy — checks dated by sequence number**
> You wrote check #42 ("withdraw $100") but never deposited it. Meanwhile, the bank's ledger advanced past your transaction. When the check finally surfaces, the teller checks: *"the ledger is on entry 50; this check is for entry 42; the corresponding withdrawal already settled differently"* — rejects the check. Without sequence numbers, replaying old paperwork would corrupt the ledger.

---

## 9. Frangipani vs. Two-Phase Commit

### Q: Is Frangipani basically two-phase commit?

**A.** Similar shape, different geography.

| Step | Frangipani | Two-Phase Commit |
|---|---|---|
| Acquire locks | At the workstation executing the op | At each shard holding the data |
| Gather data | Pull blocks from Petal into local cache | Stays sharded across servers |
| Log | One log per workstation (in Petal) | One log per shard (local disk) |
| Apply | Workstation writes back to Petal | Each shard applies locally |
| Release locks | After log write | After all shards confirm |

### 9.1 Key differences

```
   Frangipani:        data + locks + log are MOVABLE (cached at WS)
   2PC:               data + locks + log are PINNED to specific servers
```

### 9.2 Recovery from a crash mid-operation

- **Frangipani**: any other workstation can read the dead WS's log from Petal and finish recovery.
- **Classic 2PC**: log is on the crashed server's local disk → unreachable → locks held → **progress halted** until the server reboots.
- **Modern 2PC** (Spanner): replicate each shard with Paxos → no single point of failure.

> **Frangipani's elegance**: by putting the per-WS log in shared storage, it gets distributed transactions **without** the classic 2PC blocking-on-coordinator-death pathology.

---

## 10. Historical & Misc

### Q: What is DEC?

**A.** Digital Equipment Corporation — the authors' employer. DEC was a major minicomputer maker; Unix was originally developed (at Bell Labs) on DEC PDP-11 hardware. DEC was acquired by Compaq in 1998 (a year after this paper), which was acquired by HP in 2002.

### Q: How does Petal take efficient snapshots?

**A.** Petal indexes its mapping by `(virtual block, epoch)`. To snapshot, just **increment the current epoch**:

```
   Before snapshot (epoch=5):     After snapshot (epoch=6):

   block 100 → (5, phys-A)        block 100 → (5, phys-A)   ← snapshot reads this
                                            → (6, phys-A)   ← live writes go here

   write block 100:
       since current epoch (6) > mapping's epoch (5),
       allocate phys-B, map (6, phys-B), leave (5, phys-A) intact.
```

- **Snapshot reads** specify the snapshot epoch → see the old mapping.
- **Live reads** see the highest-epoch mapping.

This is copy-on-write at the block-mapping level. No bulk copy; snapshots are instant.

### Q: What's the state of distributed FS today?

**A.** In 2020:

- Workstation FS like Frangipani has **waned** — laptops are self-contained, cloud services replace shared file servers.
- Protocols still in use: SMB, NFS, AFS.
- Modern distributed FS: Ceph, Lustre, xtreemfs, Dropbox.
- Most "storage" sales (NetApp, EMC) go toward **block** (iSCSI) or object stores.
- For web / big data: key-value stores and HDFS-style DFS dominate, not POSIX file servers.

---

## 11. Cheat Sheet

| Concept | One-liner |
|---|---|
| **FS logic in clients** | Cache locally, coordinate via lock service |
| **Two logs** | Frangipani's per-WS log (FS ops) + Petal's internal log (block ops) |
| **Shared logs in Petal** | Any peer can recover a crashed workstation's log |
| **Metadata-only recovery** | File content survival is the app's problem (use `fsync`) |
| **Write absorption** | Bigger log → fewer flushes of hot directory blocks |
| **Striping** | Spread blocks of one file across Petal servers |
| **False sharing** | Two items in one block → unrelated workstations fight over it |
| **Per-file locks** | Coarser than per-block, but matches real workloads |
| **Version numbers** | Skip already-applied log entries during crash recovery |
| **Frangipani vs. 2PC** | Movable data/locks/log vs. pinned-to-shards |
| **Petal snapshots** | Copy-on-write via `(block, epoch)` map indexing |

### The recurring lesson

> **Decentralization needs ceremony.** When you remove the central server (GFS master, 2PC coordinator), you get scalability and load balancing — but you have to add a lock service, distributed logs, and version numbers to recover the invariants that the central server used to enforce for free.
