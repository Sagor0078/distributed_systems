# Lecture 3 — GFS FAQ (Human-Friendly)

> **MIT 6.824 · Distributed Systems Engineering (2020)**
> Topic: Google File System (GFS) · Course site: [pdos.csail.mit.edu/6.824](http://pdos.csail.mit.edu/6.824)

---

## Table of Contents

1. [Atomic Record Append](#1-atomic-record-append)
2. [Finding Data in Append-Only Files](#2-finding-data-in-append-only-files)
3. [Checksums](#3-checksums)
4. [Snapshots and Reference Counts](#4-snapshots-and-reference-counts)
5. [Compatibility with POSIX Apps](#5-compatibility-with-posix-apps)
6. [Replica Placement](#6-replica-placement)
7. [Leases and Split-Brain Prevention](#7-leases-and-split-brain-prevention)
8. [Chunk Size: Why 64 MB?](#8-chunk-size-why-64-mb)
9. [Correctness vs. Performance](#9-correctness-vs-performance)
10. [Master Failures and Evolution](#10-master-failures-and-evolution)

---

## 1. Atomic Record Append

**Q: Why is atomic record append *at-least-once* instead of exactly-once?**

**A:** When a write fails at some replicas, the client retries. This can cause the same record to appear more than once at replicas that did not fail. Achieving exactly-once would require the server to detect duplicates across retries, even if the primary fails between the original request and the retry. That adds significant complexity and performance overhead (you will implement this in Lab 3).

### Analogy: A clerk stamping receipts

You ask a clerk to stamp your receipt. The stamp sometimes jams, so you ask again. The clerk might have already stamped it, but you did not see it. Now you may end up with two stamps. Avoiding duplicates would require the clerk to keep a permanent log of which receipts were already stamped.

---

## 2. Finding Data in Append-Only Files

**Q: How do applications find records if the append offset is unpredictable?**

**A:** GFS is optimized for sequential scans. Applications read the file from start to end and validate each record. They do not need to know the exact offsets in advance.

**Q: How do applications detect padding and duplicates?**

**A:** Applications embed markers inside each record:
- A **magic number** or **checksum** to detect valid records vs padding.
- A **unique ID** per record to detect duplicates.

GFS provides a library that helps applications scan and filter records.

---

## 3. Checksums

**Q: What is a checksum?**

**A:** A checksum is a small value computed from a block of bytes. If the data changes, the checksum likely changes too.

GFS stores checksums alongside each chunk:
1. On write, the chunkserver computes and stores a checksum.
2. On read, it re-computes and compares.
3. If they differ, the chunk is corrupted and the read fails.

**Example:** CRC32 is a common checksum algorithm.

---

## 4. Snapshots and Reference Counts

**Q: What are reference counts in GFS?**

**A:** Snapshots use **copy-on-write**. When a snapshot is created, chunks are not copied immediately. Instead, the master increments a reference counter for each chunk.

If a client later modifies a chunk with a reference count greater than one, the master creates a copy so the snapshot remains unchanged.

### Analogy: Photocopying a binder

You make a photocopy of a binder but keep the same pages in both binders until someone wants to write on a page. At that moment, you copy the page and let them edit the copy. This avoids copying everything up front.

---

## 5. Compatibility with POSIX Apps

**Q: Can POSIX apps use GFS without changes?**

**A:** No. GFS was designed for new, large-scale applications (like MapReduce), not for retrofitting existing POSIX applications.

---

## 6. Replica Placement

**Q: How do clients find the nearest replica?**

**A:** The paper suggests Google used IP address layout as a proxy for physical proximity. In 2003, IP ranges often mapped cleanly to specific racks or machine rooms.

---

## 7. Leases and Split-Brain Prevention

**Q: Could two primaries exist after a network split?**

**A:** The lease mechanism prevents this. The master grants a time-limited lease (e.g., 60 seconds) to a primary. It will not grant a new lease to a different server until the old one expires. Even if the old primary is still alive, it must stop acting as primary once its lease ends.

### Diagram: Lease-based primary handoff

```
	Time --->

	Master grants lease to S1
	S1: [PRIMARY ACTIVE]-----------------[LEASE EXPIRES]
												  |
												  | Master can now lease to S2
												  v
											  S2: [PRIMARY ACTIVE]------>
```

---

## 8. Chunk Size: Why 64 MB?

**Q: Why is the chunk size so large?**

**A:** It reduces metadata overhead at the master and improves throughput for large sequential reads/writes. Clients can still read and write smaller parts of a chunk.

**Trade-off:** Small files get less parallelism because they live in fewer chunks.

---

## 9. Correctness vs. Performance

**Q: Is it acceptable that GFS relaxes consistency?**

**A:** This is a common distributed systems trade-off. Strong consistency adds coordination and latency. GFS targets workloads (like MapReduce) that tolerate:
- Duplicate records
- Inconsistent reads
- Holes in files

It is **not** suitable for correctness-critical data like bank balances.

---

## 10. Master Failures and Evolution

**Q: What happens if the master fails?**

**A:** GFS keeps replica masters, but early designs required **human intervention** to fail over. Later systems (like Raft-based designs) automate this.

**Q: Was a single master a good idea?**

**A:** It simplified the original system but became a bottleneck as the system scaled. Google replaced GFS with **Colossus**, which spreads metadata across multiple servers and improves automated recovery.

---

## Quick Visual Summary

```
	CLIENTS
		|
		v
	MASTER  <--- metadata, leases, placement
		|
		v
	CHUNK SERVERS  <--- actual data + checksums
```

> **Takeaway**
> GFS chooses simplicity and throughput over strict consistency. Its design works well for large, sequential workloads, and its ideas influenced many later systems, even as its single-master architecture became a scaling bottleneck.