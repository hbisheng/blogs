---
title: "Transaction Isolation Levels"
date: "2023-03-26"
weight: 20
---

**TL;DR** Transaction isolation levels define the permitted behaviors of a system when concurrent transactions are executed.

## Background

What's a transaction? A **transaction** is a group of operations (possibly operating on multiple data items) that either succeeds as a whole (commit) or fails as a whole (abort). There's no state of partial success or partial failure. Transactions allow application developers to focus on the correctness of operations within a transaction and worry less about the existence of concurrent operations. The **serializability** isolation level maintains the illusion that there's no concurrent transaction execution at all.

### ACID

Traditional relational databases (RDBMS) can achieve the ACID properties which include serializability

- *Atomicity:* The group of operations must succeed or fail together. Transactions can be aborted altogether.
- *Consistency:* Invariants/constraints within the database must not be violated. But the problem is that it's usually the application's job to decide whether the data is valid; the database simply stores it.
- *Isolation:* Transactions are executed concurrently but the final effects are as if they are executed serially.
- *Durability:* Data must be persisted even in case of failures.

### transaction isolation vs consistency level

How is transaction isolation related to the consistency level?

They are orthogonal concerns. Consistency levels matter when there are multiple copies of the same data. Transaction isolation matters even when there's only a single copy of the data.

Traditionally, the discussion of transaction isolation levels involves a single-machine database (e.g. a relational database management system). From the point of view of the consistency level, it trivially implements linearizability because there's only one copy of the data. For those, achieving serializability is the only concern. The first part of the post discuss the anomalies that may arise before serializability is achieved.

In a distributed database, things get more complex and linearizability is not guaranteed. For those, achieving serializability is not the only goal; there are anomalies that elude serializability.

## Anomalies due to no serializability

What happens if there's no/weak isolation? What issues may arise?

### **1. Dirty Writes**

**Your transaction modifies an uncommitted value.**

|  |  |
| --- | --- |
| **Txn1** | **Txn2** |
| Write(A = 2) |  |
|  | Write(A = 3) ← dirty write |
| abort |  |
|  | abort |
| Anomaly: A = 2 ← this is incorrect | |

This is a problem when you only maintain two versions (old vs new) per key.

Dirty writes must be prevented to maintain the correct semantics of transaction rollbacks.

### **2. Read uncommitted / dirty reads**

**Your transaction reads a value that has not been committed.**

- Initially, A = 1

|  |  |
| --- | --- |
| **Txn1** | **Txn2 (Increment A by 1)** |
| Write(A = 2) |  |
|  | Read(A) = 2 ← dirty read |
|  | Write(A = 3) |
| abort |  |
|  | commit |
| Anomaly: A = 3 ← this is incorrect | |

### **3. Non-repeatable Reads**

**Your reads do not come from the same point in time.** When you do multiple reads, you usually assume that they come from a certain consistent snapshot of the database.

**Read skew**

- Initially, SUM(A, B) = 100

|  |  |
| --- | --- |
| **Txn1** | **Txn2** |
| Read(A)=50 |  |
|  | Write(A=40) and Write(B=60) |
|  | Commit |
| Read(B)=60 |  |
| Anomaly: SUM(A,B) = 110, which should never happen  The cause is that the reads on A and B do not happen atomically. | |

### **4. Write Skew**

**Your read comes from a consistent snapshot, but they have changed when you write.**

Two transactions read the same data but update disjoint subsets of the data that was read.

- Constraint: A + B > 8
- Initially A = B = 5

|  |  |
| --- | --- |
| **Txn1** | **Txn2** |
| Read(A) = 5  it looks safe to decrement A |  |
|  | Read (B) = 5  it looks safe to decrement B |
| Write(A = 4) |  |
|  | Write(B = 4) |
| Anomaly: Constraint A + B > 8 is violated | |

This is similar to a **lost update**

- Initially, A = 0

|  |  |
| --- | --- |
| **Txn1**  (Read A, decrement A by 1) | **Txn2**  (Read A, decrement A by 1) |
| Read(A)=0 |  |
|  | Read(A)=0 |
| Write(A=1) |  |
|  | Write(A=1) |
| commit ← Read(A) in Txn2 is not repeatable |  |
|  | commit |
| Anomaly: After the two transactions are completed, A is only decremented by 1.  The cause is that the read on A is not repeatable. | |

Another anomaly that I don't have a name for

|  |  |
| --- | --- |
| **Txn1**  If k1 is empty, set k1 to v1 and k2 to v2. | **Txn2**  If k1 is empty, set k1 to v1 and k3 to v3. |
| Read(k1)=∅ |  |
|  | Read(k1)=∅ |
| Write(k1=v1) and Write(k2=v2) |  |
|  | Write(k1=v1) and Write(k3=v3) |
| commit ← Read(k1) in Txn2 is not repeatable |  |
|  | commit |
| Anomaly: k1=v1, k2=v2, k3=v3. But you never expect k2 and k3 to coexist. | |

### **5. Phantom Read (involving range queries)**

The keys you read are still repeatable, but what if the same query now has more records?

|  |  |
| --- | --- |
| **Txn1**  Select all records, calculate a max  Select all records again, calculate an average. | **Txn2**  Insert a large value |
| Read all records, max → 10 |  |
|  | Insert a record with value 1000 |
| Read all records, average → 100 |  |
| Anomaly: The average is larger than the max | |

New records are **inserted** into the range you read. Your second read of the range has additional records. The values of your first read are still valid, but the number of keys is not.

## Anomalies that elude serializability

### **1. Causal Reversal**

One write causes another write, your read observes the effect before the cause.

- Person 1 commits a transaction that writes A.
- Person 1 shouts at Person 2 to start the second transaction (an out-of-band approach)
- Person 2 commits a transaction that writes B.

Person 2 may not be able to read A (imagine this is a multi-master setup where writes can happen on different nodes)

Another way to look at it:

- In a MVCC system, B may have a lower timestamp than A, making it appear that B is written first.
- This is because A and B can hit different storage partitions.

### **2. Stale Read**

Reads are re-ordered arbitrarily.

Stale reads do not violate serializability. Serializability just indicates there's some order. It doesn't say anything about whether that order conforms to the real-time order.

- Txn1: Write A with an old value.
- Txn2: Write A with a new value.
- A client may see the old value (because it may talk to a replica that hasn't fully caught up yet). Unlike Immortal Write, the write transactions are not re-ordered. It's the read that gets reordered.

### **3. Immortal Write (I call it "evil reordering")**

Writes are reversed in time.

- Txn1: Write A with an old value.
- Txn2: Write A with a new value.
- A client may see the old value forever. Since serializability says nothing about real-time order, Txn2 can be re-ordered to before Txn1.

## **Transaction Isolation Levels**

### Description

|  |  |
| --- | --- |
| **Isolation Level** | **Description** |
| Read Uncommitted | It's ok to read uncommitted writes. |
| Read Committed | Only committed writes can be read. |
| Repeatable Read  (Snapshot Isolation) | Multiple reads within a transaction come from a consistent snapshot. |
| Serializable | The outcomes of the transactions are as if they are executed in some serial order. |
| Strict Serializable | Each **transaction** is executed **instantaneously** with a total order that respects the **real-time** constraint. In addition to being serializable, the serial order respects the real-time constraint (happen-before relationships). |

### Anomaly prevention

✔️ = prevented

|  |  |  |  |  |  |  |
| --- | --- | --- | --- | --- | --- | --- |
| **Isolation Level** | Dirty Write | Dirty Read | Non-repeatable Read | Phantom | Write Skew | 1. Causal Reversal 2. Stale Read 3. Immortal Write |
| Read Uncommitted | ✔️ |  |  |  |  |  |
| Read Committed | ✔️ | ✔️ |  |  |  |  |
| Repeatable Read  (Snapshot Isolation) | ✔️ | ✔️ | ✔️ | ✔️ |  |  |
| - Serializable - Serializable Snapshot Isolation | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ |  |
| Strict Serializable | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ | ✔️ |

### Implementation

|  |  |
| --- | --- |
| **Isolation Level** | **Implementation** |
| Read Uncommitted | - You keep **two versions per key**, the pre-value (for rollback) and the modified value. - No lock. Any operation from any transaction can operate on the data. There's really no isolation at all. |
| Read Committed | - You keep **two versions per key**, the modified but uncommitted value and the pre-value. Uncommitted values are not visible to other transactions. - Use **row-level lock**. Writes need to acquire exclusive locks (so that only one transaction is allowed to have an uncommitted write). Reads need no lock. |
| Repeatable read  (Snapshot Isolation) | Multi-version concurrency control (MVCC). You keep **multiple versions per key**. Your transaction may read the same value many times and they should read from the same snapshot, even when there are newer versions that are committed. Keep the versions as long as there are transactions operating on them. |
| Serializable  (*One-copy serializability* in a distributed system, otherwise useless) | Refer to strict serializable first.  A leader-follower system where the leader implements strict serializability but **requests can be routed to the followers**. This could result in **stale reads**, which are serializable but not strict serializable. In this system, writes have to be performed on the leader, so they are strict serializable. This is called **strong writes serializability***.*  In a system that achieves strict serializability per partition, writes that go to different partitions do not necessarily respect the real-time constraint. It's possible to get a causal reversal across partitions. This is called **strong partition serializability**. |
| Serializable Snapshot Isolation | Snapshot isolation + Optimistic concurrency control. Why is linearizability not maintained? Because of stale reads again. A read transaction that follows a write transaction may not read the latest value (because the read hits the follower). Optimistic concurrency control puts no restriction on reads; it only puts restrictions on writes. |
| Strict Serializable (serializable + linearizability) | Single-partition   - **2-phase locking (2PL) with deadlock prevention**   - Acquire locks on data objects and only release all locks when commit. Writes need exclusive locks; reads need shared locks.   - You only need **one version per key** because there are no (real) concurrent transactions operating on the same keys. - **Optimistic concurrency control**   - Proceed as if there's no conflict. When about to commit, check if the data objects (of both preconditions and writes) have been modified by another transaction. If yes, abort the current transaction and ask the client to retry.   Multi-partition   - 2-phase commit (2PC) and clock skew coordination (e.g. Spanner waits for the maximum time skew to pass before completing a write) |

## References & More resources

- https://dbmsmusings.blogspot.com/2019/05/introduction-to-transaction-isolation.html
- https://dbmsmusings.blogspot.com/2019/06/correctness-anomalies-under.html
- The transaction chapter from DDIA (Design Data Intensive Application).
- https://jepsen.io/consistency/models/serializable
