---
title: "MySQL and InnoDB"
date: "2023-05-22"
weight: 40
---

## Intro

**Relational database:** MySQL is a relational database. In a relational database, all data is organized in the shape of a **table**,with **rows** and **columns**. The first row can be thought of as the header. Similar to a table in its common sense, the header describes what information will be present for each row in the table; the header forms the **schema** of the table. Each row is a data point; each column is a property of that data point.

**SQL language**: With the SQL language, you can extract the rows/columns you want. It's like applying filters to the data, but it's more powerful than that. More advanced SQL usage includes **GROUP.** You can group the rows by a certain property. Rows with the same property value will be put into the same group. Then some arithmetics (MAX, MIN, AVG) can be performed within each group. In MySQL, you can combine multiple **SELECT** statements into a single query with a UNION.

**Data modeling**: You may need **multiple tables** that are related to describe your data. One motivation is to reduce duplication (think of TNF, Third Norm Form). A row in one table can reference to one or more rows of another table by a **foreign key**. Different tables can be **joined** to form a larger table that contains all the properties of the original tables.

## MySQL InnoDB

InnoDB maintains a B+ tree on disks where each tree node is a page on the disk. At the filesystem level, these are represented as InnoDB data files (.ibd files).

**What's a B+ tree?**

- It's a tree structure where data is sorted.
- Data is only stored on the leaf nodes. All leaf nodes are at the same depth.
- Internal nodes stored pairs of values and page numbers. The page number points to the child node; the associated value is the smallest value in the subtree. The internal nodes usually fit into memory so there's no disk IO for looking up the leaf node.

**Redo log:** Redo log is for tracking dirty pages for the data files that have not been flushed to disk. It's critical for data recovery.

One thing to remember is that the state of the database is fully represented by what's on the disk. Things in memory do not count; they can be lost at any time. Memory is for speed optimization and disk is for persistence. There's no way around a WAL (redo log).

**MVCC**: InnoDB achieves MVCC. MVCC allows data to be accessed concurrently without locking the entire table. Each row has a hidden column about the version. Old versions that are no longer referenced are cleaned up in a background thread.

(Note: While InnoDB doesn't take this approach, some optimization/implementation of MVCC for the B+ tree involves using copy-on-write for data pages. When a page is modified, it's not modified in place; a new page is created, updating all parenting pages up until the root. This forms a new B+ tree where most of the pages are reused. Each write creates a new root, representing a consistent snapshot of the database.)

**Buffer pool:** InnoDB engine uses a buffer pool to cache pages in memory. To avoid double caching (caching the same page at both the user space and kernel space), `O_DIRECT` can be used.

- Read flow: Follow the B+ tree to get the page number and read it. Read from the buffer pool if possible.
- Write flow: Follow the B+ tree to get the page number, get the page into the buffer pool if needed, modify the page in memory but flush it async, the redo log must be flushed for the write to be committed. If the page overflows, it will be split. Changes will be propagated up the tree.

**B+ tree size**

- Typical numbers
  - Page size is usually 16KB. The usable space is usually 14KB.
  - The size of a pointer is usually 12 bytes (4 bytes int key and 8 bytes page ID)
- How many pointers can be stored per page?
  - 14KB / 12 = 1194 pointers.
- How many 1KB rows can be stored per page?
  - 14KB / 1KB = 14 rows.
- How many rows are there for a two-level B+ tree?
  - 1194 \* 14 = 16K rows.
  - Memory usage of the internal nodes: (1 + 1194) \* 16KB = 19MB. That's tiny.
- How many rows are there for a three-level B+ tree?
  - 1194 \* 1194 \* 14 = 20M rows.
  - Rumor says you want to split your DB above this, but having a four-level B+ tree isn't performing much worse.
  - Memory usage of the internal nodes: (1 + 1194 + 1194 \* 1194) \* 16KB = 23GB. Still not the end of the world for large machines these days?

## MySQL Replication

### **GTID (**global transaction identifier**)**

It's a globally unique identifier for a transaction. It's in the format of `source_server_id: transaction_id` where the transaction id is a sequence number in commit order.

Each server records the set of GTIDs that have been executed. If a GTID transaction has been applied, any further transaction with the same GTID is ignored. GTID is useful for checking the replication progress of a follower.

Traditionally (Before MySQL 5.6), MySQL replication was file-based, which tracks the binlog file and positions. GTID makes the process easier. With auto-positioning, a follower doesn't have to know the binlog file and position, it just needs to know the GTID  of the transaction last executed.

### End-to-end replication flow

**Binlog** records the data changes to the database (e.g. write transactions). That sounds like the redo log? The difference is, that binlog keeps track of changes on the **transaction level** while the redo log keeps track of changes at the **page level**.

Binlog is useful for point-in-time recovery. You don't have to enable binlog if there's no follower attached.

#### Dump thread, IO thread, SQL thread

The leader runs a **dump thread** that reads from the binlog and sends it to the follower. The follower has an **IO thread** for receiving the binlog and stores it as the **relay log**. The **SQL thread** on the follower reads the relay log and applies the transactions.

The bottleneck is usually not the CPU but I/O (how fast the transactions can be applied). MySQL is multi-threaded. A MySQL leader can handle requests from concurrent connections. But the follower may not be able to keep up.

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| **Steps** | **Description** | **redo log** | **bin log** | **relay log** |
| 1 | The client sends a transaction to a MySQL leader. |  |  |  |
| 2 | The MySQL leader modifies the in-memory state. |  |  |  |
| 3 | The MySQL leader writes the transaction to the InnoDB redo log buffer.  The redo log buffer is periodically synced to disk.  If the server/OS crashes now, the transaction may or may not be persisted to disk. If persisted, it will just be an uncommitted transaction that will be rolled back. | txn? |  |  |
| 4 | The MySQL leader writes the transaction to the binlog buffer.  If the server/OS crashes now, the transaction may or may not be persisted to disk. | txn? | txn? |  |
| 5 | The MySQL leader's dump thread sends it to the MySQL follower. The IO thread on the follower puts it onto the relay log. | txn? | txn? | txn |
| 6 | Once the follower acknowledges that the transaction has been persisted on the follower, the leader commits the transaction by writing a COMMIT record to the redo log and binlog.  With `innodb_flush_log_at_trx_commit=1`, the fsync to the redo log happens now. With `sync_binlog=1`, the fsync to the binlog happens now. | txn, commit | txn, commit | txn |
| 7 | The leader acknowledges to the client that the transaction has been committed. |  |  |  |

### Replication types

1. Per database
   - This assumes each database is updated independently and concurrently.
2. Logical clock
   - Use a logical timestamp of each transaction in the binlog for dependency tracking.
     - On a follower, a transaction can only start when its dependency and all transactions before the dependency are committed.
   - By default, the logical timestamp is the commit timestamp.
     - Each transaction has a monotonically-increasing sequence number, plus the sequence number of the last committed transaction. As a result, each transaction depends on the last transaction that was committed. Note that there will be no dependency between concurrent transactions.
   - Write-set is a more advanced option.
     - It uses a big hash map to tell if two transactions touch the same key.
     - Only supports RBR (row-based replication).

### **Group commit**

A leader may be able to group multiple commits and flush them to disk in one operation. In this case, they become one operation in the binlog and the follower can apply those transactions in parallel.

`replica_preserve_commit_order` ensures that transactions are committed on the follower in the same order as the leader. Note that the *scheduling* (i.e. the start time) of the transaction can be out-of-order.

`binlog_group_commit_sync_delay` introduces delays for commits on the leader so that more transactions can be applied in parallel on the followers. This is meant to improve the replication lags.

## MySQL Backup

Backup is not only useful for backing up data but also for duplicating the data to create a new follower.

How `Xtrabackup` works is that it copies the InnoDB data files (.ibd files) along with the redo log and then performs crash recovery. The data files are large and they could be copied at different timestamps and the redo log keeps track of all the data file changes since those timestamps.

We need to track the redo log since the first data file is copied until the last data file. This can take hours. After the crash recovery, the state of the database reaches the end of the copy, not the start.

## PTTC (Percona Table Checksum)

PTTC checks that the data on the leader is the same as on the followers. It's useful for detecting data corruption.

It works on one table at a time. It divides the table into chunks of rows and calculates a checksum for each chunk. The checksum is calculated and inserted into a checksum table by a `REPLACE..SELECT` query. The key here is to perform a statement-based replication so the same operation is done on the follower. If the data is inconsistent, the checksum will be different.

## References

- [MySQL Parallel Replication: All the 5.7 and 8.0 Details (LOGICAL\_CLOCK)](https://www.youtube.com/watch?v=BDZ4CNQdU_0) - Jean-François Gagné
