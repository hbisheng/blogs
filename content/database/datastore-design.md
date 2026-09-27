---
title: "Datastore design: B+ tree and LSM tree"
date: "2023-04-03"
weight: 30
---

**TL;DR** B+ tree was the dominating data structure for database indexing. The LSM tree has become increasingly popular in the distributed database world. Let's consider building an **OLTP (online transactional processing)** data store from scratch.

Many of the notes here came from my reading of *Designing data-intensive application.*

### Bash

The simplest storage: **a single file and some bash commands**

- *Write*: append a new record (performance is hard to beat)
  - This is somewhat like a WAL. Whenever there's a write, OS updates the page cache with the new data and flushes it due to the `fsync` system call. If the current page is full, it allocates a new page to put the new data; the metadata of the file needs to be updated as well to include this new page.
- *Read*: find the latest record

Write performance is hard to beat; read performance is abysmal.

Suppose that the file is super large and spans a lot of pages, it will be difficult to traverse all the relevant pages to find a particular key. It'll be great if there's an index that tells you which page contains the last write of the target key.

### Hash index

To speed up reads, **use an in-memory hash index** to point to the positions of keys on disks.

- Keys on disk are append-only and can be compacted periodically.
- The index can be rebuilt if the node crashes.

However, this assumes that all keys fit into memory. Range query performances are still bad.

### B-Tree

Let's maintain an index on disk then? A B+ tree is exactly that.

- First of all, all keys are **sorted** and stored on multiple pages. A B+ tree index is built on top of those pages.
- The B+ tree itself is stored on separate pages. Each node is a page. The tree has a branching factor and each node has several child nodes.
- Do all leaves have the same depth? Yes. Nodes are added to the leaf nodes. Expansion is propagated bottom-up.

There's usually a **primary index** that's built on the primary key. The primary index is usually also the **clustered index** that contains the actual data on the leaf nodes. But nothing stops us from building a **secondary index** that sorts the same data pages by a different key and thus in a different order.

Range queries on the primary index are more efficient because they are sequential. Range queries on a secondary index are typically accessing the pages in random order. But this difference may be less relevant on SSDs.

Note: some differences between a B tree and a B+ tree:

1. B+ tree stores data at the leaf nodes only.
2. B+ tree links the leaf nodes with pointers.

Such an index is not very efficient for answering 2D queries; it can only fetch all data that matches the first dimension and then filter on the second dimension. One can map the 2D space into a 1D space and then do a linear scan (that probably involve many fragments). But more commonly, spatial indexes such as R-tree are used to **narrow down multiple dimensions simultaneously**.

### LSM tree

Coming back to the in-memory hash index, what if we still keep an in-memory map of keys and values to serve reads and writes? When there are too many keys, we must **flush the in-memory mapping of keys onto disks**. Before writing to disk, we'd better reformat the content so that lookups can be performed efficiently later. We sort the keys and write them to disk as **SSTables** (sorted string table). The sorted property allows us to optimize in different ways.

- The keys with their values are first sorted and then **compressed** (note that these tables can be pretty large so compression is definitely relevant here). Contiguous keys are stored within the same disk block.
- A **hash index** can be stored at the end of the disk block. It can be used to point to *the first key* of each disk block. A **binary search** can be performed to look up all disk blocks.
- SSTables are usually **immutable** but they can be **compacted** in the background. New segments are written to disk while old segments still serve read traffic. After the new segments are written, reads will be switched and old segments can safely be deleted.
- As for failure recovery for the in-memory content, use a WAL log (B+ tree needs to use this as well).

This is an **LSM (log-structured) tree.**

- Write: append to WAL; update in-memory map; flush the map to disk if necessary.
- Read: Look up the in-memory map. If not found, look up the SSTable in reverse order.

**Checksums** can be used to detect corrupted data (partially written records).
